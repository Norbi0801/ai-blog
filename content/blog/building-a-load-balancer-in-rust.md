+++
title = "Building a load balancer in Rust"
date = 2025-06-29
description = "A walk through the internals of an L7 load balancer in Rust with hyper - round-robin, least-connections, weighted, random, TCP and HTTP health checks, connection draining, hot reload, and per-backend metrics."

[taxonomies]
tags = ["rust", "networking", "hyper", "infrastructure"]
+++

The fastest way to understand what HAProxy and nginx are actually doing is to build a small version yourself. Not a production replacement - a few hundred lines of Rust that proxies HTTP, picks backends with the four classic algorithms, runs health checks, drains connections on shutdown, and reloads the backend list without dropping requests. Once you've held those pieces in your head, the configuration knobs in HAProxy stop looking like magic.

If you're not familiar with what a "real" Rust proxy looks like at scale, I covered Cloudflare's framework in [Cloudflare Pingora - replacing nginx with Rust](/blog/cloudflare-pingora---replacing-nginx-with-rust/). This post zooms in the other direction. Forget the trillion-requests-per-second tier. The goal here is to map the abstract concepts (selection, health, draining, reload) to concrete bytes and `tokio` tasks.

<!-- more -->

## What an L7 load balancer actually does

Layer 7 means the proxy parses the application protocol - in our case HTTP - before deciding what to do. That is what separates it from an L4 balancer like LVS or HAProxy in `mode tcp`, which just shuffles bytes between sockets without looking inside.

For each incoming request the L7 balancer needs to:

1. Accept the inbound TCP connection and parse the HTTP request.
2. Pick a backend from a pool of healthy candidates.
3. Open (or reuse) a connection to that backend and forward the request.
4. Stream the response back to the client.
5. Record what happened - status code, latency, bytes - per backend.

Around that hot path there are background concerns: probing each backend periodically to know if it's healthy, swapping the backend list at runtime when configuration changes, and finishing in-flight requests cleanly when the operator sends `SIGTERM`.

The version we'll build keeps `Cargo.toml` short:

```toml
[dependencies]
hyper = { version = "1.5", features = ["http1", "server", "client"] }
hyper-util = { version = "0.1", features = ["tokio", "server-auto", "client-legacy"] }
http-body-util = "0.1"
tokio = { version = "1", features = ["full"] }
tokio-util = "0.7"
arc-swap = "1.7"
parking_lot = "0.12"
```

`arc-swap` is the trick that makes hot reload painless: it lets us replace an `Arc<BackendPool>` atomically without taking a lock on the read path. `parking_lot` for the mutex around metrics. Everything else is the standard hyper 1.x stack ([hyper docs](https://hyper.rs/), [hyper-util on crates.io](https://crates.io/crates/hyper-util)).

## The shared state

Three things every request needs to see: the backend pool, the algorithm, and the metrics sink.

```rust
use arc_swap::ArcSwap;
use parking_lot::Mutex;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, AtomicU64, Ordering};

#[derive(Debug)]
pub struct Backend {
    pub addr: String,           // "127.0.0.1:9001"
    pub weight: u32,            // 1 = baseline
    pub healthy: AtomicUsize,   // 0 = down, 1 = up
    pub in_flight: AtomicUsize, // for least-connections
    pub requests: AtomicU64,    // total served
    pub errors: AtomicU64,
}

#[derive(Clone, Copy, Debug)]
pub enum Algorithm {
    RoundRobin,
    Random,
    LeastConnections,
    Weighted,
}

pub struct Pool {
    pub backends: Vec<Arc<Backend>>,
    pub rr_cursor: AtomicUsize,
}

pub struct Shared {
    pub pool: ArcSwap<Pool>,
    pub algo: Algorithm,
    pub metrics: Mutex<Histogram>,
    pub draining: std::sync::atomic::AtomicBool,
}
```

A few things to call out. `healthy` and `in_flight` are atomics so the request path never touches a mutex when picking a backend - on a 32-core box, a contended mutex on every request would torch tail latency. The `Pool` itself sits inside `ArcSwap`, which is a [lock-free RCU-style pointer swap](https://docs.rs/arc-swap/latest/arc_swap/) - readers grab a snapshot in a few nanoseconds, and reload simply publishes a new `Arc<Pool>`.

## Selection: the four algorithms

Each algorithm gets one function. They all take a snapshot of the pool and return a backend (or `None` if everything is down).

```rust
use rand::seq::IteratorRandom;
use rand::Rng;

pub fn pick<'a>(pool: &'a Pool, algo: Algorithm) -> Option<Arc<Backend>> {
    let healthy: Vec<&Arc<Backend>> = pool.backends.iter()
        .filter(|b| b.healthy.load(Ordering::Acquire) == 1)
        .collect();
    if healthy.is_empty() { return None; }

    let chosen = match algo {
        Algorithm::RoundRobin => {
            // fetch_add wraps modulo len - this is the classic atomic counter trick
            let i = pool.rr_cursor.fetch_add(1, Ordering::Relaxed) % healthy.len();
            healthy[i].clone()
        }
        Algorithm::Random => {
            let mut rng = rand::thread_rng();
            healthy.iter().choose(&mut rng).unwrap().clone().clone()
        }
        Algorithm::LeastConnections => {
            healthy.iter()
                .min_by_key(|b| b.in_flight.load(Ordering::Acquire))
                .unwrap()
                .clone()
                .clone()
        }
        Algorithm::Weighted => {
            // Smooth weighted random: expand into a virtual list weighted by `weight`
            let total: u32 = healthy.iter().map(|b| b.weight.max(1)).sum();
            let mut roll = rand::thread_rng().gen_range(0..total);
            let mut picked = healthy[0].clone();
            for b in healthy.iter() {
                let w = b.weight.max(1);
                if roll < w { picked = (*b).clone().clone(); break; }
                roll -= w;
            }
            picked
        }
    };
    Some(chosen)
}
```

The interesting one is round-robin. nginx's classic round-robin uses a per-worker counter incremented under a mutex - fine when each worker has its own pool. Our balancer is multi-threaded with a shared pool, so a `fetch_add` on an `AtomicUsize` is the right primitive. `Relaxed` ordering is enough because the counter doesn't synchronize with anything else; we just want a number that monotonically increases.

Least-connections is a linear scan over healthy backends. That sounds expensive but it's perfectly fine up to a few hundred backends - the load is dominated by network I/O. If you ever have thousands of backends, switch to a min-heap that you update on `in_flight` change, or shard the pool.

Weighted picks a uniform random integer in `[0, sum_of_weights)` and walks the list. With three backends weighted `[1, 2, 5]`, a roll of 0 hits the first, 1-2 the second, 3-7 the third. This is `O(n)` per request; for huge pools the trick is the [smooth weighted round-robin algorithm nginx uses](https://github.com/phusion/nginx/blob/master/src/http/modules/ngx_http_upstream_round_robin.c), which keeps a "current weight" per backend and avoids RNG entirely.

## Proxying with hyper

The body of the proxy is small. Hyper 1.x splits "server" and "client" cleanly: we accept connections with `hyper-util`'s auto server, and we open outbound connections with the legacy client.

```rust
use hyper::body::Incoming;
use hyper::{Request, Response, StatusCode, Uri};
use hyper_util::client::legacy::{Client, connect::HttpConnector};
use hyper_util::rt::{TokioExecutor, TokioIo};
use http_body_util::{BodyExt, Full};
use bytes::Bytes;

type ProxyBody = http_body_util::combinators::BoxBody<Bytes, hyper::Error>;

async fn proxy(
    req: Request<Incoming>,
    shared: Arc<Shared>,
    client: Client<HttpConnector, Incoming>,
) -> Result<Response<ProxyBody>, hyper::Error> {
    if shared.draining.load(Ordering::Acquire) {
        // 503 immediately during graceful shutdown
        return Ok(error_response(StatusCode::SERVICE_UNAVAILABLE, "draining"));
    }

    let pool = shared.pool.load();
    let backend = match pick(&pool, shared.algo) {
        Some(b) => b,
        None => return Ok(error_response(StatusCode::BAD_GATEWAY, "no healthy backends")),
    };

    // Build the upstream URI by replacing host:port
    let path_and_query = req.uri().path_and_query()
        .map(|x| x.as_str())
        .unwrap_or("/");
    let uri: Uri = format!("http://{}{}", backend.addr, path_and_query)
        .parse()
        .unwrap();

    let (mut parts, body) = req.into_parts();
    parts.uri = uri;
    parts.headers.remove("host");          // upstream wants its own Host
    parts.headers.insert("x-forwarded-for", "127.0.0.1".parse().unwrap());
    let upstream_req = Request::from_parts(parts, body);

    backend.in_flight.fetch_add(1, Ordering::AcqRel);
    let start = std::time::Instant::now();

    let result = client.request(upstream_req).await;

    backend.in_flight.fetch_sub(1, Ordering::AcqRel);
    let elapsed = start.elapsed();
    shared.metrics.lock().observe(&backend.addr, elapsed);

    match result {
        Ok(resp) => {
            backend.requests.fetch_add(1, Ordering::Relaxed);
            let (parts, body) = resp.into_parts();
            Ok(Response::from_parts(parts, body.boxed()))
        }
        Err(e) => {
            backend.errors.fetch_add(1, Ordering::Relaxed);
            // A connection error here is a strong signal - mark the backend down
            // so health checks have to bring it back.
            backend.healthy.store(0, Ordering::Release);
            eprintln!("upstream error for {}: {e}", backend.addr);
            Ok(error_response(StatusCode::BAD_GATEWAY, "upstream error"))
        }
    }
}

fn error_response(status: StatusCode, msg: &'static str) -> Response<ProxyBody> {
    Response::builder()
        .status(status)
        .body(Full::new(Bytes::from(msg)).map_err(|e| match e {}).boxed())
        .unwrap()
}
```

A few things worth dwelling on.

The `client` is shared across all requests. Hyper's legacy client maintains an internal connection pool keyed on the upstream `(scheme, host, port)`. That means once we've hit `127.0.0.1:9001` for the first request, the keep-alive connection sits in the pool and the next request reuses it. This is the same property that drove Cloudflare's connection-reuse jump from 87.1% to 99.92% with Pingora - we get a small version of that "for free" because every request goes through the same client.

We rewrite the request URI rather than doing a new request because hyper's client treats the URI's host and port as the routing key. We also strip the `Host` header so the client can fill in the upstream's own host. In a real balancer you'd preserve the original `Host` in `X-Forwarded-Host` and add `X-Forwarded-Proto` - the [list of de facto headers RFC 7239 codifies](https://www.rfc-editor.org/rfc/rfc7239) is worth memorizing.

`in_flight` increments before and decrements after the upstream call. It has to be a paired increment/decrement under all error paths, otherwise least-connections stops working and one backend gets starved as its counter drifts upward. Using a guard (RAII) is cleaner in larger code:

```rust
struct InFlight<'a>(&'a Backend);
impl<'a> InFlight<'a> {
    fn new(b: &'a Backend) -> Self { b.in_flight.fetch_add(1, Ordering::AcqRel); Self(b) }
}
impl<'a> Drop for InFlight<'a> {
    fn drop(&mut self) { self.0.in_flight.fetch_sub(1, Ordering::AcqRel); }
}
```

Now even if the future is cancelled (client disconnect), the decrement runs.

## Health checks - TCP and HTTP

A backend is "healthy" if a probe succeeds N times in a row, and "unhealthy" if it fails M times in a row. That hysteresis matters: if you flip on a single failure, you'll mark backends down during transient hiccups and oscillate.

```rust
use tokio::net::TcpStream;
use tokio::time::{interval, Duration};

#[derive(Clone, Debug)]
pub enum Probe {
    Tcp,
    Http { path: String },
}

pub async fn health_loop(shared: Arc<Shared>, probe: Probe) {
    let mut tick = interval(Duration::from_secs(2));
    let mut consecutive_ok: std::collections::HashMap<String, u32> = Default::default();
    let mut consecutive_fail: std::collections::HashMap<String, u32> = Default::default();

    loop {
        tick.tick().await;
        let pool = shared.pool.load_full();
        for b in &pool.backends {
            let ok = check_one(&b.addr, &probe).await;
            let addr = b.addr.clone();
            if ok {
                let n = consecutive_ok.entry(addr.clone()).or_insert(0);
                *n += 1;
                consecutive_fail.remove(&addr);
                if *n >= 2 && b.healthy.load(Ordering::Acquire) == 0 {
                    b.healthy.store(1, Ordering::Release);
                    eprintln!("{} -> healthy", b.addr);
                }
            } else {
                let n = consecutive_fail.entry(addr.clone()).or_insert(0);
                *n += 1;
                consecutive_ok.remove(&addr);
                if *n >= 3 && b.healthy.load(Ordering::Acquire) == 1 {
                    b.healthy.store(0, Ordering::Release);
                    eprintln!("{} -> unhealthy", b.addr);
                }
            }
        }
    }
}

async fn check_one(addr: &str, probe: &Probe) -> bool {
    match probe {
        Probe::Tcp => {
            tokio::time::timeout(Duration::from_millis(500), TcpStream::connect(addr))
                .await
                .map(|r| r.is_ok())
                .unwrap_or(false)
        }
        Probe::Http { path } => {
            // Single-shot connection, raw write to keep the demo dependency-free
            let req = format!(
                "GET {path} HTTP/1.1\r\nHost: {addr}\r\nConnection: close\r\n\r\n"
            );
            let res: std::io::Result<bool> = tokio::time::timeout(
                Duration::from_secs(1),
                async {
                    use tokio::io::{AsyncReadExt, AsyncWriteExt};
                    let mut s = TcpStream::connect(addr).await?;
                    s.write_all(req.as_bytes()).await?;
                    let mut buf = [0u8; 12];
                    s.read_exact(&mut buf).await?;
                    // "HTTP/1.1 2xx" or "HTTP/1.1 3xx" -> healthy
                    Ok(buf.starts_with(b"HTTP/1.1 2") || buf.starts_with(b"HTTP/1.1 3"))
                }
            ).await.unwrap_or(Ok(false));
            res.unwrap_or(false)
        }
    }
}
```

The TCP probe is what HAProxy calls `option tcp-check`. It only proves that the kernel accepted a SYN and completed the handshake - it doesn't prove the application is alive. For anything but the simplest backend, an HTTP probe (`option httpchk GET /healthz`) is what you want.

The HTTP probe here is deliberately written with raw byte writes rather than another hyper client, because health checks run hundreds of times more often than user requests and you want them lean. A real implementation would parse the full status line and read the response body or close, but for this demo "starts with `HTTP/1.1 2`" is good enough.

A subtle but important detail: don't use the same client and keep-alive pool that user traffic uses for health checks. If you reuse a stale pooled connection that the upstream has half-closed, the health check fails for the wrong reason. Always open a fresh connection for probes.

## Hot reload of the backend list

The reason we wrapped `Pool` in `ArcSwap` is this. Reload looks like:

```rust
pub fn reload(shared: &Shared, new_backends: Vec<Arc<Backend>>) {
    let new_pool = Arc::new(Pool {
        backends: new_backends,
        rr_cursor: AtomicUsize::new(0),
    });
    shared.pool.store(new_pool);
    eprintln!("reloaded backend list, {} entries", shared.pool.load().backends.len());
}
```

`store` swaps the `Arc<Pool>` atomically. Any in-flight request that already grabbed the old `Arc` via `pool.load()` keeps using it until that request finishes - the old pool stays alive as long as any reference exists. Once the last reference drops, the old `Pool` is freed. No locks. No coordination. No "stop the world" while config reloads.

In practice you'd hook this up to `SIGHUP`, a config file watcher with [notify](https://crates.io/crates/notify), or an admin endpoint:

```rust
#[cfg(unix)]
async fn watch_sighup(shared: Arc<Shared>, config_path: String) {
    use tokio::signal::unix::{signal, SignalKind};
    let mut sig = signal(SignalKind::hangup()).unwrap();
    while sig.recv().await.is_some() {
        match load_backends_from_file(&config_path) {
            Ok(b) => reload(&shared, b),
            Err(e) => eprintln!("reload failed: {e}"),
        }
    }
}
```

This is essentially what nginx's `nginx -s reload` does at a higher level - it forks new workers with the updated config, then drains the old ones. Our version is simpler because everything lives in one process: the new backend set is just a different `Arc<Pool>` that future requests will see.

## Connection draining on shutdown

When the operator sends `SIGTERM`, two things have to happen in order:

1. Stop accepting new connections.
2. Let in-flight requests finish, up to a deadline.

Hyper's auto server gives us this through `tokio_util::sync::CancellationToken` and `with_graceful_shutdown`. The pattern is:

```rust
use tokio_util::sync::CancellationToken;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let shared = Arc::new(build_shared_state());
    let client = Client::builder(TokioExecutor::new()).build_http();
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    let shutdown = CancellationToken::new();

    // ctrl-c / sigterm
    let token = shutdown.clone();
    tokio::spawn(async move {
        tokio::signal::ctrl_c().await.ok();
        token.cancel();
    });

    tokio::spawn(health_loop(shared.clone(), Probe::Http { path: "/healthz".into() }));

    loop {
        tokio::select! {
            _ = shutdown.cancelled() => break,
            accept = listener.accept() => {
                let (stream, _) = accept?;
                let io = TokioIo::new(stream);
                let shared = shared.clone();
                let client = client.clone();
                let token = shutdown.clone();
                tokio::spawn(async move {
                    let svc = hyper::service::service_fn(move |req| {
                        proxy(req, shared.clone(), client.clone())
                    });
                    let conn = hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
                        .serve_connection(io, svc);
                    tokio::pin!(conn);
                    tokio::select! {
                        res = conn.as_mut() => { let _ = res; }
                        _ = token.cancelled() => {
                            conn.as_mut().graceful_shutdown();
                            let _ = conn.await;
                        }
                    }
                });
            }
        }
    }

    // Flip the global drain flag - new requests get 503 even if they sneak in.
    shared.draining.store(true, Ordering::Release);
    // Wait up to 30s for in-flight requests to finish naturally.
    tokio::time::sleep(Duration::from_secs(30)).await;
    Ok(())
}
```

Two things here that are easy to get wrong. First, draining requires telling hyper "no new requests on this connection" but allowing the current request to complete - that's exactly what `graceful_shutdown` does on a hyper connection. Second, you also want to refuse new requests at the application layer, because a client may have a keep-alive connection from before draining started and try to push another request through. The `draining` atomic check at the top of `proxy()` is what catches that.

If you want to learn how this looks at a much bigger scale, Pingora's "graceful upgrade" mechanism transfers the listening socket itself to a new process so zero connections are refused during the handover - I went into the details in the [Pingora post](/blog/cloudflare-pingora---replacing-nginx-with-rust/).

## Per-backend metrics

The two questions you always end up asking: "which backend is taking the traffic" and "how is the latency distributed". The first is a counter, the second needs a histogram.

```rust
use std::collections::HashMap;

pub struct Histogram {
    // Pre-defined buckets in microseconds: 100us, 1ms, 10ms, 100ms, 1s, 10s, +inf
    buckets_us: [u64; 7],
    counts: HashMap<String, [u64; 7]>,
    totals: HashMap<String, u64>,
}

impl Histogram {
    pub fn new() -> Self {
        Self {
            buckets_us: [100, 1_000, 10_000, 100_000, 1_000_000, 10_000_000, u64::MAX],
            counts: Default::default(),
            totals: Default::default(),
        }
    }
    pub fn observe(&mut self, addr: &str, d: Duration) {
        let us = d.as_micros() as u64;
        let entry = self.counts.entry(addr.to_string()).or_insert([0; 7]);
        for (i, bucket) in self.buckets_us.iter().enumerate() {
            if us <= *bucket { entry[i] += 1; break; }
        }
        *self.totals.entry(addr.to_string()).or_insert(0) += 1;
    }
    pub fn render_prometheus(&self) -> String {
        let mut out = String::new();
        out.push_str("# TYPE lb_request_duration_us histogram\n");
        for (addr, counts) in &self.counts {
            let mut cumulative = 0;
            for (i, bucket) in self.buckets_us.iter().enumerate() {
                cumulative += counts[i];
                let le = if *bucket == u64::MAX { "+Inf".into() } else { bucket.to_string() };
                out.push_str(&format!(
                    "lb_request_duration_us_bucket{{backend=\"{addr}\",le=\"{le}\"}} {cumulative}\n"
                ));
            }
        }
        for (addr, total) in &self.totals {
            out.push_str(&format!("lb_requests_total{{backend=\"{addr}\"}} {total}\n"));
        }
        out
    }
}
```

The histogram is intentionally fixed-bucket, in the [Prometheus exposition format](https://prometheus.io/docs/instrumenting/exposition_formats/). For real systems you'd reach for [`metrics`](https://crates.io/crates/metrics) plus an exporter, or [`hdrhistogram`](https://crates.io/crates/hdrhistogram) for high-resolution percentiles. The point is: per-backend `requests_total` plus a duration histogram is the floor of useful observability for a load balancer. Without those two metrics you can't answer "did the new node start serving traffic" or "is one backend slow" - which are the only two questions that ever come up in an outage.

Surface the metrics with a route check in `proxy()`:

```rust
if req.uri().path() == "/metrics" {
    let body = shared.metrics.lock().render_prometheus();
    return Ok(Response::new(Full::new(Bytes::from(body))
        .map_err(|e| match e {}).boxed()));
}
```

## What HAProxy and nginx are doing that we skipped

Once you have the skeleton above, the gaps to a real load balancer become legible.

**TLS termination.** A real edge balancer terminates TLS, which means certificate management, SNI, ALPN, OCSP stapling, session resumption. The standard Rust path is [`rustls`](https://crates.io/crates/rustls) with `tokio-rustls` in front of hyper. nginx ships with OpenSSL bindings; HAProxy uses OpenSSL or AWS-LC.

**Layer-4 mode.** HAProxy in `mode tcp` doesn't parse HTTP - it just splices bytes between sockets, often with [`SO_SPLICE`](https://man7.org/linux/man-pages/man2/splice.2.html) or `tee` so the data never crosses userspace. Our L7 implementation always parses the HTTP envelope, which costs CPU.

**Sticky sessions.** Picking a backend deterministically based on a cookie or source IP hash so a user's whole session lands on the same backend. This is the [`hash` directive](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#hash) in nginx and `balance source` in HAProxy. Easy to add: hash the cookie value modulo the number of healthy backends. The hard part is what to do when the backend list changes - you want [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing) to keep most users mapped to the same backend after a reload.

**Backend connection pools per upstream.** Hyper's client gives us this implicitly, but real proxies expose detailed knobs: max connections per upstream, idle timeout, max requests per connection. Pingora's [tiered hot-pool / shared-pool design](/blog/cloudflare-pingora---replacing-nginx-with-rust/) is what this looks like at the high end.

**Slow start.** When a backend transitions from unhealthy to healthy, you don't want it to immediately receive a full share of traffic - it might have a cold cache or still be warming up. nginx's `slow_start` parameter ramps a backend's effective weight up over a configured period.

**Active vs passive health checks.** What we built is active: independent of user traffic, the balancer probes backends. Passive health checks treat user-traffic errors as health signals - "this backend returned three 502s in a row, mark it down." Both are useful and they catch different failure modes.

**Circuit breakers, rate limiting, retries with backoff.** Each of these is its own tuning surface. HAProxy has `option redispatch` for retrying on a different backend; Envoy has [outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier) for ejecting flaky backends statistically.

**Observability beyond metrics.** Access logs, distributed tracing headers, request IDs threaded through. The minimum you need to debug "a request was slow" in production.

## What you have at the end

A few hundred lines of Rust, plus dependencies you'd already have around in a hyper-based service. It can:

- Accept HTTP/1.1 traffic and forward it to one of N backends.
- Pick that backend with round-robin, random, least-connections, or weighted random.
- Probe backends every 2 seconds with TCP or HTTP and apply hysteresis before flipping state.
- Reload the backend list at runtime via `SIGHUP` without dropping requests.
- Drain on `SIGTERM`, refusing new requests with 503 while finishing in-flight ones.
- Expose a `/metrics` endpoint with per-backend counts and a duration histogram.

That covers the spine of what a generic L7 balancer does. Everything HAProxy and nginx do beyond this - TLS, sticky sessions, advanced health checks, retries, slow start, layer-4 splicing - is layered onto the same core.

The exercise is worth doing once. After you've spent an afternoon writing your own least-connections selector and watching `in_flight` counters move in real time, the configuration block in HAProxy that says `balance leastconn` stops being magic. You've already written it.
