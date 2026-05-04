+++
title = "Rust in production - what companies actually use it for"
date = 2025-02-15
description = "Real Rust production stories from Cloudflare, Discord, Figma, AWS, Shopify, 1Password, and Dropbox - the problems they solved and why they didn't rewrite everything."

[taxonomies]
tags = ["rust", "production", "architecture", "infrastructure"]
+++

Every few months someone publishes a "Rust is taking over the world" article full of hype and short on details. Then someone else responds with "Rust is overhyped" and we go in circles.

Neither take is useful. What's actually useful is looking at what specific companies use Rust for, what problems it solved, and - just as importantly - what they chose *not* to rewrite. Because the pattern is clear: nobody rewrites their entire stack in Rust. They pick the parts where Rust's guarantees translate directly into production outcomes.

Let's look at seven companies, the concrete problems they faced, and what Rust did for them.

<!-- more -->

## Cloudflare - replacing NGINX at a trillion requests per day

Cloudflare's proxy layer used to run on NGINX. It worked, but NGINX's architecture made it increasingly painful to add the custom logic Cloudflare needed. Connection reuse across workers was poor (NGINX uses a process-per-core model where each worker manages its own connection pool), and extending it with Lua had hit diminishing returns.

So they built [Pingora](https://github.com/cloudflare/pingora) - a proxy framework written in Rust from scratch. Not a fork of NGINX, not a wrapper. A new architecture built on async Rust with tokio.

The numbers are hard to argue with:

- **1 trillion+ requests per day**, 40 million requests per second
- **70% less CPU** and **67% less memory** compared to the old NGINX-based stack on identical traffic
- Connection reuse went from 87.1% to **99.92%** for a major customer - that's 160x fewer new connections per second
- TTFB (Time to First Byte) dropped by 5ms on median and 80ms at the tail

The connection reuse improvement alone is significant. Each new TCP connection means a handshake, potentially a TLS handshake on top of that, and extra latency for the end user. Pingora's multi-threaded architecture lets all workers share a single connection pool to each upstream, which NGINX's per-process model couldn't do.

Cloudflare [open-sourced Pingora](https://blog.cloudflare.com/pingora-open-source/) in early 2024. A recent blog post showed them [optimizing hot-path string lookups with a trie structure](https://blog.cloudflare.com/pingora-saving-compute-1-percent-at-a-time/), saving 1% CPU fleet-wide - which at their scale translates to real money.

What they *didn't* do: rewrite their Workers runtime, their dashboard, their DNS resolver, or their analytics pipeline in Rust. Pingora replaced one specific layer - the HTTP proxy - where Rust's performance, memory safety, and async model gave them measurable wins.

## Discord - killing garbage collection latency spikes

Discord's Read States service tracks which channels you've read and which have unread messages. Every time you open Discord, this service gets hit. It's one of their hottest paths.

The original Go implementation had a problem. Every two minutes - like clockwork - latency would spike to 10-40 milliseconds. The cause was Go's garbage collector. The service maintained a large LRU cache (millions of entries), and Go's GC had to scan it periodically. Even though Go's GC is concurrent, the stop-the-world pauses on a heap that size were enough to create user-visible delays.

They tried tuning GC parameters. They tried reducing allocations. Nothing eliminated the fundamental issue: a tracing garbage collector has to walk live objects, and when you have millions of them in a cache, that takes time.

The Rust rewrite removed the problem entirely. No GC means no GC pauses. The LRU cache is just a data structure in memory - no runtime is periodically scanning it.

Results after the migration:
- Average latency dropped to **microseconds** (from milliseconds in Go)
- The periodic 10-40ms spikes disappeared completely
- They increased cache capacity to 8 million Read States with lower memory usage

The key insight from Discord's story isn't "Rust is faster than Go" in general. It's that for *this specific workload* - a service holding millions of long-lived objects in memory - Rust's ownership model was fundamentally better suited than any garbage-collected language. A service that processes short-lived requests and doesn't hold much state might show no meaningful difference.

Discord documented this in detail in their blog post [Why Discord is switching from Go to Rust](https://discord.com/blog/why-discord-is-switching-from-go-to-rust).

## Figma - 10x multiplayer performance

Figma's multiplayer editing lets hundreds of designers collaborate on the same file simultaneously. Their original multiplayer server was written in TypeScript. As files got larger and more users piled into single documents, serialization became the bottleneck.

They rewrote the multiplayer server in Rust and saw [serialization time improve by over 10x](https://www.figma.com/blog/rust-in-production-at-figma/). Not 10% - ten times faster. The server went from being a scaling concern to something they could essentially stop worrying about.

Their architecture is interesting: each document gets its own Rust child process that communicates with the host via stdin/stdout. The Rust process uses so little memory that running thousands of them in parallel is feasible. It's a clean isolation model - one document crashes, others are unaffected.

More recently (mid 2025), Figma published another post about [memory optimizations in Rust](https://www.figma.com/blog/supporting-faster-file-load-times-with-memory-optimizations-in-rust/) where they switched from a HashMap to a Vec for representing file structures internally. The result: 20% faster deserialization at p99 and 20% memory savings across their entire multiplayer fleet. The kind of optimization that's straightforward in Rust because you control the memory layout.

What Figma *didn't* do: rewrite their rendering engine (which runs in WebAssembly/C++), their design tool UI (TypeScript/React), or their API layer. They specifically targeted the multiplayer sync server where serialization speed and memory efficiency directly impacted user experience.

## Shopify - a Rust JIT compiler for Ruby

This one surprises people. Shopify doesn't run Rust services. Instead, they used Rust to make *Ruby* faster.

[YJIT](https://github.com/Shopify/yjit) (Yet Another JIT) is a just-in-time compiler that lives inside CRuby itself. It was originally written in C99, and the team [ported it to Rust](https://shopify.engineering/porting-yjit-ruby-compiler-to-rust) in about three months.

Why Rust for a JIT compiler? A JIT generates machine code at runtime - it's manipulating raw memory, managing code pages, dealing with instruction encoding. This is exactly the kind of low-level work where C's lack of safety guarantees leads to subtle, hard-to-debug memory corruption bugs. Rust gives you the same level of control over memory layout and code generation while catching an entire class of bugs at compile time.

The performance improvements are significant:

- Ruby 3.2 with YJIT: **41% faster** than without YJIT on benchmarks
- Only **33% memory overhead** compared to the interpreter
- YJIT is the most memory-efficient Ruby JIT - beating JRuby and TruffleRuby despite them having far more sophisticated optimization passes

Shopify runs YJIT on their production Storefront Renderer, which handles Black Friday traffic. The Rust port didn't just improve safety - the team reported that the Rust codebase is genuinely easier to maintain than the C version was.

This is a pattern worth highlighting: Rust doesn't have to be "the thing you deploy." It can be the tool that makes your existing stack better. YJIT ships as part of CRuby - every Ruby developer using Ruby 3.2+ benefits from Rust without knowing it.

## AWS - microVMs in 50,000 lines of Rust

Every time you invoke an AWS Lambda function or spin up a Fargate container, you're running inside [Firecracker](https://github.com/firecracker-microvm/firecracker) - a Virtual Machine Monitor (VMM) that AWS built from scratch in Rust.

Firecracker creates lightweight microVMs using Linux KVM. It's designed with a minimal attack surface: only 5 emulated devices (virtio-net, virtio-block, virtio-vsock, serial console, and a keyboard controller used solely for stopping the VM). Compare that to QEMU, which emulates dozens of devices and weighs in at over a million lines of code.

Firecracker is about **50,000 lines of Rust** - a 96% reduction in code compared to QEMU. The results:

- Boot time: **~100ms** for a microVM
- Memory footprint: as low as **5 MB** per microVM
- Density: **150+ microVMs created per second** on a single host, thousands running concurrently
- Nearly a **quadrillion** requests handled across Cloudflare's- across AWS's global infrastructure

Why Rust specifically? AWS's [NSDI paper on Firecracker](https://www.usenix.org/system/files/nsdi20-paper-agache.pdf) explains it clearly: Firecracker sits at the security boundary between tenants. A vulnerability in the VMM could let one customer's code access another's. Memory safety isn't a nice-to-have here - it's a hard requirement. C/C++ would require extraordinary discipline and tooling to achieve the same safety level that Rust provides by default.

The minimal device model matters too. Fewer emulated devices means fewer potential bugs, fewer side channels, and a smaller surface for exploitation. Rust's type system helps enforce this minimalism - you can make invalid device states unrepresentable.

## 1Password - one Rust core, every platform

1Password took a different approach. Instead of rewriting a performance-critical component, they built a shared core in Rust that handles business logic, cryptography, database access, and server communication across all their platforms - macOS, Windows, Linux, iOS, Android, and browser extensions.

Before Rust, each platform had its own implementation. Roughly **0% code sharing**. A bug fix had to be reimplemented and tested on every platform independently. A new feature meant coordinating releases across multiple teams writing in different languages.

After moving to a Rust core with thin platform-native UI layers on top: **63% code sharing**. The cryptography code - the part that absolutely must be correct - is written once and compiled for every target. No more hoping that the Swift implementation and the Kotlin implementation handle edge cases the same way.

The architecture looks like this:

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  macOS UI    │  │  Windows UI  │  │  Linux UI    │
│  (Swift)     │  │  (C#/XAML)   │  │  (GTK)       │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └────────────┬────┘────────────┬────┘
                    │                 │
              ┌─────▼─────────────────▼─────┐
              │        Rust Core            │
              │  - Cryptography             │
              │  - Database (SQLite)        │
              │  - Sync engine              │
              │  - Vault logic              │
              │  - Server communication     │
              └─────────────────────────────┘
```

If you've read my earlier post on [the adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/), this is that pattern at an architectural scale. The Rust core defines the domain boundary, and each platform's UI is essentially an adapter.

1Password also open-sourced [Typeshare](https://github.com/1Password/typeshare), a tool that generates TypeScript, Swift, and Kotlin type definitions from Rust structs - keeping the FFI boundary type-safe across languages.

## Dropbox - sync engine on half a billion devices

Dropbox rewrote the core of their desktop sync engine in Rust. This is the component called Nucleus - it's what makes the Dropbox folder on your computer work. It runs on **over 500 million devices**, handles trillions of files, and deals with every bizarre filesystem edge case you can imagine (case sensitivity differences, Unicode normalization, symlink loops, file systems that truncate names).

Alongside the sync engine, Dropbox invested heavily in Rust-based compression:

- **[rust-brotli](https://github.com/dropbox/rust-brotli)** - a Brotli decompressor/compressor in Rust that streams decompression alongside the network download, so you overlap I/O with compute
- **Broccoli** - their modified Brotli that supports chunk-level parallelism. They can compress at **3x the rate of Google's reference Brotli implementation** by splitting files into chunks, compressing each chunk on a separate core, and concatenating the results
- **DivANS** - an experimental codec also in Rust, chosen specifically because Rust's safe subset guarantees deterministic behavior - critical for a compression algorithm where non-determinism means data corruption

The Broccoli optimization reduced median sync latency and data transfer by **over 30%**. When you're syncing files for half a billion users, 30% less data transfer is a massive infrastructure cost reduction.

## The pattern nobody talks about

Look at those seven case studies and a pattern emerges. No company started with "let's rewrite our API endpoints in Rust." Every single one started with infrastructure:

- **Proxies and networking** (Cloudflare)
- **High-throughput stateful services** (Discord)
- **Serialization-heavy backend processes** (Figma)
- **Compiler/runtime tooling** (Shopify)
- **Virtualization** (AWS)
- **Cross-platform shared core** (1Password)
- **Sync engines and compression** (Dropbox)

The adoption path is almost always: infrastructure and CLI tools first, performance-critical backend services second, business logic almost never.

This makes sense. Rust's compile times, steeper learning curve, and smaller ecosystem for web-oriented libraries mean it's a poor fit for rapidly iterating on business logic where correctness-at-the-type-level matters less than speed-of-development. But for infrastructure code that runs for years, handles extreme load, or sits at a security boundary - that's where the investment pays off.

None of these companies rewrote everything. Cloudflare's dashboard is still in React. Discord's API layer still runs plenty of Python and Go. Figma's frontend is TypeScript. Shopify's storefront is Rails. They rewrote the *specific components* where Rust's properties - no GC pauses, predictable performance, memory safety without runtime overhead, zero-cost abstractions - translated into measurable production improvements.

## The 2026 job market

The practical question for many developers: is Rust worth investing in for your career?

The [2025 Stack Overflow Developer Survey](https://survey.stackoverflow.co/2025/) showed Rust as the most admired programming language for the **tenth consecutive year** at 72%. Cargo is the most admired build tool at 71%. These aren't just vanity metrics - they indicate developer retention. People who learn Rust tend to stick with it.

The job market numbers tell a more nuanced story:

- **$110K-$210K globally**, with senior roles in the US reaching **$170K-$300K**
- Job postings have grown roughly **15% year-over-year**, more than doubling over the past two years
- Rust developers command a premium over C/C++ developers due to a smaller talent pool

The [2025 State of Rust Survey](https://blog.rust-lang.org/2026/03/02/2025-State-Of-Rust-Survey-results/) (7,156 respondents) showed **45.5% organizational adoption** - up from 38.7% a year earlier. That's a 17.6% year-over-year increase in companies using Rust for non-trivial work.

But here's the nuance: most Rust jobs aren't "Rust developer" positions. They're infrastructure engineer roles, systems programming roles, or platform engineering roles where Rust is one of the required tools. You'll find them at cloud providers, database companies, security firms, fintech infrastructure, and developer tooling startups. If you're looking for "Rust web developer" roles building CRUD APIs - those exist, but they're a small fraction of the market.

The highest-demand specializations pair Rust with domains:
- **WebAssembly** - for edge computing, browser-based tooling, plugin systems
- **Embedded systems** - automotive, IoT, firmware
- **Cloud infrastructure** - proxies, service meshes, observability pipelines
- **Cryptography and security** - password managers, key management, TLS implementations

## Why they didn't rewrite everything

This is the question that doesn't get asked enough. If Rust is so great, why does every company keep large parts of their stack in Go, Python, TypeScript, or Java?

Because rewriting working software has costs beyond engineering time:

1. **Institutional knowledge** - the existing codebase encodes years of bug fixes and edge case handling. A rewrite risks losing that knowledge.
2. **Hiring** - there are far more Python/Go/Java developers than Rust developers. Building a team that can maintain a Rust codebase is harder and more expensive.
3. **Iteration speed** - for business logic that changes weekly based on product experiments, Rust's compile times and strictness slow you down more than they help.
4. **Risk** - a rewrite is a project with uncertain timelines. The existing code works. Breaking it for theoretical gains is a hard sell to engineering leadership.

The smart companies treat Rust as a scalpel, not a sledgehammer. They identify the components where performance, safety, or reliability are the primary constraints - and those are the components that get rewritten.

## What this means for you

If you're considering Rust at your company, don't start with your API layer. Start with:

- **CLI tools** that your team uses daily. Small scope, fast feedback, and a single-binary distribution model that ops teams love.
- **A performance-critical library** used by multiple services. Write it in Rust, expose a C API or use FFI, and call it from your existing stack.
- **Infrastructure components** - a custom proxy, a log processor, a data pipeline stage.
- **Anything at a security boundary** - auth services, crypto operations, input validation for untrusted data.

Build a small team's expertise on these smaller projects. Let them develop patterns, build internal libraries, and establish best practices. Then, when a larger component genuinely needs Rust's properties, you have the team and the institutional knowledge to do it well.

The companies in this post didn't adopt Rust because it was trendy. They adopted it because they had specific, measurable problems - GC pauses, CPU costs at scale, memory safety at security boundaries, cross-platform code duplication - and Rust solved them. That's the only reason worth rewriting anything.
