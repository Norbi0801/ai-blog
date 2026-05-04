+++
title = "Cloudflare Pingora - replacing nginx with Rust"
date = 2025-03-15
description = "How Cloudflare built Pingora from scratch in Rust to replace nginx, achieving 70% less CPU, 99.92% connection reuse, and a programmable proxy architecture that handles a trillion requests per day."

[taxonomies]
tags = ["rust", "infrastructure", "performance", "networking"]
+++

In September 2022, Cloudflare published a blog post titled ["How we built Pingora, the proxy that connects Cloudflare to the Internet."](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/) It wasn't a proof of concept. By the time they wrote about it, Pingora was already handling every HTTP request flowing through Cloudflare's edge - over a trillion requests per day, peaking at 40 million requests per second. They'd replaced nginx in production and didn't look back.

This isn't a story about language tribalism. Cloudflare ran nginx for years and it worked. The replacement happened because nginx's architecture couldn't solve specific problems Cloudflare was hitting at their scale - problems rooted in the process model, connection isolation, memory safety, and extensibility. Pingora solved them.

<!-- more -->

## What nginx does well (and where it falls apart)

nginx was built to solve the [C10K problem](https://en.wikipedia.org/wiki/C10k_problem) - handling 10,000 concurrent connections on a single machine. Igor Sysoev designed it around an event-driven, non-blocking I/O model using `epoll` (Linux) or `kqueue` (BSD). Each worker process runs a tight event loop: call `epoll_wait()`, collect ready events, advance state machines, repeat. No thread-per-connection overhead. No context switching storm.

This was revolutionary in the early 2000s when Apache's prefork model was allocating several megabytes per connection. nginx could handle hundreds of thousands of concurrent connections on modest hardware.

The architecture looks like this:

```
                    ┌──────────────────┐
                    │   Master Process │
                    │   (manages workers)
                    └────────┬─────────┘
              ┌──────────────┼──────────────┐
              v              v              v
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Worker 0   │ │  Worker 1   │ │  Worker 2   │
    │  (process)  │ │  (process)  │ │  (process)  │
    │             │ │             │ │             │ 
    │ event loop  │ │ event loop  │ │ event loop  │
    │ conn pool A │ │ conn pool B │ │ conn pool C │
    └─────────────┘ └─────────────┘ └─────────────┘
```

Each worker is a separate OS process, pinned to a CPU core. This model works great - until you start running a CDN that serves 20% of the internet. Three problems emerge:

**Connection pools can't be shared across workers.** Each worker process has its own pool of upstream connections. If Worker 0 has an idle connection to origin server X, Worker 1 can't use it - it has to open a new one. At Cloudflare's scale, this meant millions of unnecessary TCP and TLS handshakes per second. For one major customer, connection reuse was only 87.1%. Every connection that gets created instead of reused adds latency (TCP handshake + TLS handshake) and burns CPU cycles on both sides.

**Extending nginx means writing C or Lua.** nginx modules are written in C, directly against nginx's internal APIs. Cloudflare layered significant custom logic using OpenResty (nginx + LuaJIT), but Lua's type system is basically nonexistent, and the boundary between C and Lua is a constant source of subtle bugs. Adding features like custom load balancing, per-request authentication, or header manipulation meant writing fragile glue code.

**Memory safety is a real operational cost.** nginx is written in C. C doesn't have bounds checking, use-after-free protection, or data race prevention. When you run C code on the critical path of a trillion daily requests, every memory bug is a potential security incident or crash.

## nginx's CVE history tells the story

This isn't theoretical. nginx has accumulated a consistent stream of memory safety vulnerabilities over its lifetime:

- **CVE-2013-2028** - stack-based buffer overflow in the chunked transfer encoding parser
- **CVE-2014-0133** - SPDY heap buffer overflow
- **CVE-2016-0746** - use-after-free during CNAME response processing in the resolver
- **CVE-2021-23017** - 1-byte memory overwrite in the DNS resolver, exploitable for remote code execution
- **CVE-2022-41742** - memory disclosure in `ngx_http_mp4_module`
- **CVE-2024-24990** - use-after-free in the HTTP/3 implementation
- **CVE-2024-32760** - buffer overwrite in HTTP/3
- **CVE-2024-34161** - memory disclosure in HTTP/3
- **CVE-2025-53859** - memory over-read in the SMTP authentication handler exposing sensitive data

The [full list on nginx.org](https://nginx.org/en/security_advisories.html) goes deeper. The pattern is clear: buffer overflows, use-after-free, out-of-bounds reads. These are the exact vulnerability classes that Rust's ownership system eliminates at compile time. Not mitigates - eliminates. You literally cannot write a use-after-free in safe Rust because the compiler won't let you.

For Cloudflare, every one of these CVEs means emergency patching across thousands of edge servers worldwide. The operational cost of running memory-unsafe code at the edge of the internet adds up.

## Pingora's architecture: threads, not processes

Pingora's fundamental architectural decision is using threads instead of processes for its workers. This seems like a small change, but it cascades into everything.

```
                    ┌──────────────────┐
                    │   Pingora Server │
                    └────────┬─────────┘
              ┌──────────────┼──────────────┐
              v              v              v
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Worker 0   │ │  Worker 1   │ │  Worker 2   │
    │  (thread)   │ │  (thread)   │ │  (thread)   │
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │               │               │
           └───────────────┼───────────────┘
                           v
                 ┌───────────────────┐
                 │  Shared Connection│
                 │       Pool        │
                 │  (all workers)    │
                 └───────────────────┘
```

Because threads share a memory space, the connection pool is shared. When any worker finishes a request to an upstream server, that connection goes into a common pool. Any other worker can pick it up for the next request to the same upstream. The result for that major customer: connection reuse jumped from 87.1% to **99.92%** - that's 160x fewer new connections being created per second.

Shared memory also means shared caches, shared configuration, shared routing tables - all without IPC overhead or data serialization. In nginx, sharing state between workers requires shared memory zones with manual locking. In Pingora, it's just a Rust `Arc<T>` behind an atomic reference counter.

### The async runtime

Pingora is built on [tokio](https://tokio.rs/), Rust's async runtime. Each worker thread runs a tokio executor with work-stealing enabled. When one worker runs out of tasks, it steals from another worker's queue. This prevents the situation where one worker is saturated while others are idle - a problem that nginx's `accept_mutex` only partially solves.

The work-stealing scheduler is what makes the multi-threaded model practical at scale. Without it, you'd get uneven load distribution across cores. With it, Pingora naturally balances work across all available threads without manual tuning.

### Connection pooling internals

The connection pool uses a tiered design. Most traffic hits a per-thread "hot pool" backed by an `AtomicPtr` - a lock-free atomic pointer swap at the CPU instruction level. No mutex, no contention. When the hot pool overflows or runs empty, it falls back to a shared pool protected by a mutex.

This means 90%+ of connection reuse operations are completely lock-free. The mutex only gets touched on overflow or underflow. It's a practical design that avoids the false dichotomy of "everything lock-free" versus "everything behind a mutex."

When a request to an upstream finishes, the connection is automatically returned to the pool without any special configuration. Subsequent requests reuse it, skipping the TCP and TLS handshake entirely.

## The ProxyHttp trait - programmability as a first-class concept

Where nginx exposes configuration directives and Lua callbacks, Pingora exposes a Rust trait. The [`ProxyHttp`](https://docs.rs/pingora/latest/pingora/prelude/trait.ProxyHttp.html) trait defines the lifecycle of every request passing through the proxy:

```rust
use pingora::prelude::*;
use pingora_proxy::{ProxyHttp, Session};
use pingora_load_balancing::{LoadBalancer, selection::RoundRobin};
use std::sync::Arc;
use async_trait::async_trait;

pub struct MyProxy {
    lb: Arc<LoadBalancer<RoundRobin>>,
}

#[async_trait]
impl ProxyHttp for MyProxy {
    type CTX = ();

    fn new_ctx(&self) -> Self::CTX {}

    // The only required method: where should this request go?
    async fn upstream_peer(
        &self,
        session: &mut Session,
        _ctx: &mut Self::CTX,
    ) -> Result<Box<HttpPeer>> {
        let upstream = self.lb.select(b"", 256)
            .unwrap();

        let peer = Box::new(HttpPeer::new(
            upstream,
            true,  // TLS
            "one.one.one.one".to_string(),
        ));
        Ok(peer)
    }

    // Modify or reject the request before it reaches upstream
    async fn request_filter(
        &self,
        session: &mut Session,
        _ctx: &mut Self::CTX,
    ) -> Result<bool> {
        // Return true to short-circuit (send response directly)
        // Return false to continue to upstream
        Ok(false)
    }

    // Modify the request header before sending to upstream
    async fn upstream_request_filter(
        &self,
        session: &mut Session,
        upstream_request: &mut RequestHeader,
        _ctx: &mut Self::CTX,
    ) -> Result<()> {
        upstream_request.insert_header(
            "Host", "one.one.one.one"
        )?;
        Ok(())
    }
}
```

The full request lifecycle flows through these phases in order:

1. **`early_request_filter`** - runs before full request parsing, useful for rate limiting or IP blocking
2. **`request_filter`** - inspect or modify request headers, return early responses
3. **`upstream_peer`** - decide which upstream server to connect to (required)
4. **`connected_to_upstream`** - runs after TCP/TLS connection is established
5. **`upstream_request_filter`** - modify headers before sending to upstream
6. **`request_body_filter`** - transform request body chunks as they stream through
7. **`upstream_response_filter`** - inspect/modify response headers from upstream
8. **`response_body_filter`** - transform response body chunks
9. **`logging`** - runs after response is sent, for metrics and logging

Every phase is optional except `upstream_peer`. You implement only what you need. The framework handles connection pooling, TLS, HTTP parsing, keep-alive, and buffering. You write the routing and business logic.

Compare this to nginx, where adding custom request routing means either writing an nginx C module (dealing with nginx's internal request structure, memory pools, and callback chains) or writing Lua scripts that run inside OpenResty's coroutine model. Pingora gives you a compiled, type-checked, memory-safe Rust function with full access to the request at each phase.

## Graceful upgrades without dropping requests

Deploying new proxy code to thousands of edge servers without dropping requests is hard. Pingora has a built-in graceful upgrade mechanism:

1. Start a new Pingora instance with the `--upgrade` flag. It doesn't try to bind the listening socket - instead it requests the socket from the old instance.
2. Send `SIGQUIT` to the old instance. The old instance transfers its listening socket to the new one.
3. The new instance immediately starts accepting connections on the transferred socket. The old instance enters graceful shutdown, finishing in-flight requests within a configurable grace period.

The guarantee: every incoming request is handled by either the old or the new instance. No connection refused errors. No dropped requests. This uses a combination of Unix domain sockets for socket transfer between instances and `SO_REUSEPORT` for the handoff.

nginx has a similar mechanism (`nginx -s reload`), but Pingora's version was designed for binary upgrades - replacing the entire executable, not just reloading configuration.

## Performance at Cloudflare's scale

The production numbers, compared to the nginx-based stack on identical traffic:

| Metric | nginx | Pingora | Improvement |
|--------|-------|---------|-------------|
| CPU usage | baseline | -70% | 3.3x more efficient |
| Memory usage | baseline | -67% | 3x more efficient |
| Connection reuse (major customer) | 87.1% | 99.92% | 160x fewer new connections |
| TTFB (median) | baseline | -5ms | |
| TTFB (p99 tail) | baseline | -80ms | |

The CPU and memory savings come from three sources: shared connection pools (fewer TLS handshakes), Rust's lack of garbage collection overhead (compared to the Lua/LuaJIT layer in the old stack), and the work-stealing scheduler distributing load more evenly.

The TTFB improvements come primarily from connection reuse. Reusing a warm TCP+TLS connection to an upstream means the first byte of the response arrives faster - no handshake latency.

### Optimizing the hot path

Cloudflare published a post about [optimizing string lookups in Pingora](https://blog.cloudflare.com/pingora-saving-compute-1-percent-at-a-time/) with a trie data structure. The `clear_internal_headers` function - which strips internal headers before forwarding requests - was consuming 1.7% of Pingora's total CPU time. That's 680 out of 40,000 saturated CPU cores globally.

They replaced linear string matching with a trie, saving 1.28% of total CPU utilization on `pingora-origin`. At Cloudflare's scale, 1% of CPU is thousands of servers worth of compute. This is the kind of optimization that only makes sense when you're running at the scale where small percentages translate into real infrastructure costs.

## Beyond Pingora: Oxy and FL2

Pingora is the open-source foundation. Internally, Cloudflare has built additional layers on top of it.

[Oxy](https://blog.cloudflare.com/introducing-oxy/) is their internal framework for building proxy services in Rust, built on Pingora's primitives. FL2 is the next-generation proxy layer built on Oxy, replacing FL1 (the nginx/LuaJIT-based system).

FL2 started serving customer traffic in early 2025, with progressive rollout throughout the year. Cloudflare [reported the results](https://blog.cloudflare.com/20-percent-internet-upgrade/): 10ms reduction in response time, 25% performance boost, and when paired with their Gen 13 AMD Turin hardware, 2x throughput and 50% better power efficiency. The migration from FL1 to FL2 was expected to complete in early 2026.

This is worth highlighting: Pingora wasn't a one-off. It became the foundation for Cloudflare's entire proxy infrastructure. The investment in building a Rust proxy framework paid compound returns - every new service built on Oxy/Pingora inherits the performance and safety characteristics without re-engineering them.

## Open source and the ecosystem

Cloudflare [open-sourced Pingora](https://blog.cloudflare.com/pingora-open-source/) in February 2024 under the Apache 2.0 license. As of early 2026, the [GitHub repository](https://github.com/cloudflare/pingora) has over 26,000 stars and the latest release is [v0.8.0](https://github.com/cloudflare/pingora/releases/tag/0.8.0) (March 2026).

The framework is split into multiple crates:

- **`pingora`** - the main crate, re-exports everything you need
- **`pingora-core`** - protocol implementations, runtime, base traits
- **`pingora-proxy`** - the `ProxyHttp` trait and proxy logic
- **`pingora-load-balancing`** - round-robin, hashing, weighted selection
- **`pingora-cache`** - HTTP caching layer
- **`pingora-http`** - HTTP header parsing and manipulation
- **`pingora-openssl` / `pingora-boringssl`** - TLS backends

Protocol support includes HTTP/1.1 and HTTP/2 end-to-end, gRPC proxying, WebSocket proxying, and TLS termination. HTTP/3 support is on the roadmap.

A minimal load balancer is about 80 lines of Rust:

```rust
use pingora::prelude::*;
use pingora_load_balancing::{LoadBalancer, selection::RoundRobin};
use std::sync::Arc;

fn main() {
    let mut server = Server::new(None).unwrap();
    server.bootstrap();

    let upstreams: LoadBalancer<RoundRobin> = LoadBalancer::try_from_iter(
        ["1.1.1.1:443", "1.0.0.1:443"]
    ).unwrap();

    let mut proxy = pingora_proxy::http_proxy_service(
        &server.configuration,
        MyProxy {
            lb: Arc::new(upstreams),
        },
    );
    proxy.add_tcp("0.0.0.0:6188");

    server.add_service(proxy);
    server.run_forever();
}
```

That's a production-grade load balancer with connection pooling, keep-alive, work-stealing, and graceful restarts - all handled by the framework. You write the routing logic. The framework handles the plumbing.

## What this means for the proxy ecosystem

Pingora isn't an nginx replacement in the "drop-in alternative" sense. You can't take an `nginx.conf` and feed it to Pingora. It's a framework for building proxies, not a pre-configured proxy server. [As Navendu Pottekkat pointed out](https://navendu.me/posts/pingora/), this distinction matters.

If you need a reverse proxy you can configure with a YAML/TOML file and deploy without writing code, nginx, Caddy, HAProxy, or Envoy are still the right tools. Pingora competes at a different level: it's for teams building custom proxy infrastructure where the proxy logic is part of the product.

That said, Pingora is shifting the conversation about what proxy infrastructure should look like:

**Memory safety as table stakes.** When a major CDN provider replaces its C-based proxy specifically because of memory safety concerns, it sends a signal. The HTTP/3 CVEs in nginx from 2024 (use-after-free, buffer overwrite, memory disclosure - four vulnerabilities in a single protocol implementation) demonstrate that adding new protocol support in C continues to be risky. Pingora proves you can have C-level performance with compile-time memory safety guarantees.

**Programmable proxies over configurable ones.** nginx's config language is powerful but limited. When your proxy needs to make routing decisions based on JWT claims, call an external service for authentication, or implement custom rate limiting with shared state across workers - you end up fighting the configuration model. Pingora's trait-based approach means your proxy logic is just Rust code. You get the full language: pattern matching, error handling, async/await, type safety, the entire crates.io ecosystem.

**Connection sharing as a default.** The multi-threaded shared-pool model should be the default for proxy architectures in 2026. The per-process isolation model was designed for an era when shared-nothing was the safest concurrency model. Rust's ownership system makes shared-state concurrency safe by default - the compiler catches data races before your code even runs.

**Projects building on Pingora.** The open-source release has spawned projects like [River](https://github.com/memorysafety/river), a reverse proxy built on Pingora by the Internet Security Research Group (ISRG) - the same organization behind Let's Encrypt. River aims to be a configurable, memory-safe reverse proxy that can replace nginx for common use cases without requiring users to write Rust code. The [Prossimo](https://www.memorysafety.org/) initiative from ISRG is specifically funding memory-safe implementations of critical internet infrastructure, and Pingora fits right into that mission.

## The bigger picture

Cloudflare's move is part of a broader industry shift. The White House's [ONCD report on memory safety](https://www.whitehouse.gov/oncd/briefing-room/2024/02/26/memory-safety-statements-of-support/) explicitly called out the need for memory-safe languages in critical infrastructure. CISA has published guidance recommending memory-safe alternatives for new development. The Linux kernel now accepts Rust modules. Android's memory safety vulnerabilities dropped as the percentage of new code written in memory-safe languages increased.

nginx will continue running billions of deployments for years. It's battle-tested, well-documented, and has an enormous ecosystem. But for new proxy infrastructure at scale - especially infrastructure that sits at a security boundary, handles untrusted input, and needs custom programmable logic - the argument for building on a memory-safe foundation is hard to ignore.

Cloudflare didn't rewrite their proxy because Rust is trendy. They did it because nginx's architecture couldn't efficiently share connections across workers, C's memory model kept producing CVEs in critical code paths, and Lua scripting couldn't keep up with their need for type-safe programmable logic. Pingora solved all three problems, and the 70% CPU reduction was a bonus on top.

The source code is at [github.com/cloudflare/pingora](https://github.com/cloudflare/pingora). The docs are solid. If you're building anything that proxies HTTP traffic and you've been struggling with nginx module development or Envoy filter chains, it's worth spending an afternoon with Pingora's examples. The `ProxyHttp` trait might be the cleanest proxy abstraction I've seen.
