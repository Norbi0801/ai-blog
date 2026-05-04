+++
title = "How Discord handles millions of concurrent users with Rust"
date = 2025-09-30
description = "Three separate Rust migrations at Discord - Read States, SortedSet NIF, and data services - each solving a different scaling problem."

[taxonomies]
tags = ["rust", "architecture", "performance", "concurrency"]
+++

Discord's relationship with Rust isn't a single rewrite story. It's three distinct migrations, each targeting a different bottleneck, using Rust in fundamentally different ways. One replaced a Go service. Another extended Elixir through native functions. The third built an entirely new infrastructure layer between their API and their database.

If you read my [overview of Rust in production](/blog/rust-in-production---what-companies-actually-use-it-for/), you saw the high-level summary: Go's garbage collector caused latency spikes, Rust eliminated them. That's true, but it's only a third of the story. The details of *how* they solved each problem - and *why* they picked Rust over alternatives in each case - are worth examining closely.

<!-- more -->

## Read States: when garbage collection becomes the bottleneck

The Read States service tracks which channels and messages every user has read. It gets hit on every connection, every message send, and every message read. It's about as hot-path as a service gets.

### The architecture

Each Read States server maintains an LRU (Least Recently Used) cache in memory. A single server holds tens of millions of Read States, and the cache sees hundreds of thousands of mutations per second. For persistence, the cache is backed by a Cassandra cluster - but the whole point of the cache is to avoid hitting Cassandra on the hot path.

Each Read State entry contains several atomic counters. The most visible one is the unread @mention count for a channel - that badge number you see in the sidebar. These counters need to be updated atomically and frequently reset to zero (when you read a channel, the mention count resets).

The original implementation was Go. It worked well initially, but as the cache grew, a pattern appeared in their monitoring dashboards.

### The two-minute spike

Every two minutes, latency would spike. Not subtly - the p99 would jump to 10-40 milliseconds, visible in graphs as a perfectly periodic sawtooth pattern.

The cause was Go's garbage collector. Go's runtime forces a GC cycle at minimum every two minutes, regardless of memory pressure. This is controlled by the `GOGC` environment variable and Go's internal GC triggers, but even with tuning, you can't disable the periodic collection entirely.

For most Go services, this is fine. A GC pause of a few milliseconds on a service processing short-lived HTTP requests is barely noticeable. But Read States was different. The LRU cache held millions of live objects. Go's GC is a concurrent, tri-color mark-and-sweep collector - it needs to traverse the entire object graph to determine what's alive and what's dead. With millions of cache entries, each containing pointers to sub-objects, that traversal takes real time.

Discord's engineers tried the obvious things. They reduced allocations. They tuned `GOGC`. They experimented with Go versions 1.8 through 1.10. Nothing worked because the problem was structural: a tracing garbage collector must walk live objects, and they had millions of them.

They also tried shrinking the LRU cache. Smaller cache means faster GC scans, and it did reduce spike magnitude. But a smaller cache means higher miss rates, which means more Cassandra queries, which pushed up the p99 latency from a different angle. They were trading one latency source for another.

### The Rust rewrite

When a user's Read State gets evicted from the LRU cache in Rust, that memory is freed immediately. There's no background process that will eventually get around to reclaiming it. Rust's ownership model means the data structure itself controls the lifecycle of every allocation.

The team completed the initial port in May 2019, using Rust nightly because `async/await` hadn't stabilized yet. They built on [tokio](https://tokio.rs/) for the async runtime.

After the initial port matched Go's non-spike performance (and eliminated the spikes entirely), they made several optimizations:

**BTreeMap instead of HashMap.** The LRU cache's internal map was switched from a `HashMap` to a `BTreeMap`. This seems counterintuitive - `HashMap` has O(1) lookups versus `BTreeMap`'s O(log n). But `BTreeMap` stores its data in contiguous nodes, improving cache locality for iteration-heavy workloads. It also uses less memory per entry because it doesn't need to maintain a hash table with empty buckets for acceptable load factors. When you have 8 million entries, the per-entry memory overhead adds up.

**Reduced copies.** The Rust compiler's borrow checker pushed them toward designs that minimized data copying. Where Go made it easy to pass copies of structs around (value semantics by default), Rust's ownership model naturally led to passing references and avoiding unnecessary clones.

**Tokio 0.2 upgrade.** When tokio 0.2 landed with its new scheduler (a work-stealing runtime), they got CPU utilization improvements without changing their own code. The runtime upgrade propagated performance gains to every service built on it.

After optimization, the results were decisive:

- Average response time: **microseconds** (was milliseconds in Go)
- Periodic latency spikes: **gone entirely**
- Cache capacity: increased to **8 million Read States** - up from what they could fit before, with lower total memory usage
- At the time, Discord had fewer than 50 engineers supporting over 250 million users

The Read States migration is a case study in workload-specific language selection. Go's garbage collector isn't bad - it's actually one of the best implementations of a concurrent GC in any language. But for a service whose entire purpose is to hold millions of long-lived objects in memory and mutate them at high frequency, any tracing GC will create periodic pauses proportional to heap size.

## SortedSet NIF: Rust as an extension language for Elixir

Discord's real-time messaging infrastructure runs on Elixir, built on the BEAM virtual machine (Erlang's runtime). BEAM is excellent at concurrency - its lightweight process model is designed for millions of simultaneous connections. But BEAM processes operate on immutable data structures, and that creates problems at scale for certain operations.

### The member list problem

Discord needed to change how they rendered guild member lists. Instead of sending the entire member list to every client (which doesn't scale for servers with hundreds of thousands of members), they switched to sending only the visible portion and streaming updates for adds, removes, and reorders.

This required a server-side data structure that could hold hundreds of thousands of entries, maintain sort order, and report the index of every insertion and removal - because the client needs to know which position changed, not just what changed.

In Elixir, the natural approach is a sorted list. But Elixir lists are linked lists - inserting into a sorted list of 250,000 elements means traversing the list to find the insertion point. The team benchmarked this and got 500-3000 microseconds per operation at 5,000 elements. At 250,000 elements, they were looking at 170,000 microseconds (170ms) per insert. That's per operation, on a data structure getting mutated constantly across thousands of guilds.

### Evolution through data structures

Before reaching for Rust, the team iterated through several pure-Elixir approaches:

**Erlang's `:ordsets`** - sorted lists backed by tuples. Better than raw lists, but still O(n) insertion. At 250,000 elements: ~27,000 microseconds per insert.

**Custom Skip List** - a wrapper around a list of "cells," each containing a small ordered set with first/last items and counts. This brought 250,000-element insertion down to ~5,000 microseconds. But worst-case behavior was bad: inserting at the start caused cascading evictions across cells, pushing it to 19,000 microseconds.

**OrderedSet** - the breakthrough iteration. Instead of fixed-size cells that cascade evictions, cells could swell and split, dynamically inserting new cells in the middle. This brought 250,000-element insertion down to ~640 microseconds average, with worst case at 4 microseconds. A massive improvement, but they wanted more.

### The Rust NIF

NIFs (Native Implemented Functions) let you write functions in C or Rust that compile into the BEAM VM and can be called directly from Elixir code. The team used [Rustler](https://github.com/rustler-beam/rustler), a library that provides safe Rust bindings to the NIF API and guarantees that a badly-behaved NIF won't crash the entire VM.

The Rust implementation uses a vector of vectors - structurally similar to their OrderedSet concept, but operating on mutable, contiguous memory instead of immutable Elixir terms. Operations scan linearly through buckets to find the right one, then binary search within the bucket. At 250,000 elements: **3.68 microseconds** per insert. At 1 million elements: **0.61-3.68 microseconds**.

That's a 160x improvement over the Skip List in the worst case, and a 6.5x improvement over the OrderedSet in the best case.

One important implementation detail: all operations completed well under 1 millisecond. This matters because of BEAM's scheduling model. BEAM uses preemptive scheduling based on "reductions" (roughly, function calls). A NIF that runs too long - more than a millisecond or so - starves other BEAM processes on that scheduler thread. If your NIF takes 10ms, you need to manually yield back to the scheduler using "dirty NIFs" or chunked processing. By staying under 1ms, Discord's SortedSet NIF avoided this complexity entirely.

Discord [open-sourced the SortedSet NIF](https://github.com/discord/sorted_set_nif) on GitHub. It powers every guild in Discord - from three-person friend groups to 200,000-member communities. The proof-of-concept took about a week.

### Why this works architecturally

This is a different usage pattern from the Read States rewrite. Here, Rust isn't replacing Elixir - it's solving a specific algorithmic bottleneck that Elixir's immutable data structures can't handle efficiently. The rest of the guild infrastructure stays in Elixir, where BEAM's process isolation, fault tolerance, and hot code reloading are genuinely valuable.

It's the same pattern as Shopify using Rust to build YJIT inside Ruby: use Rust as an accelerator for a specific hot spot rather than rewriting the entire service.

## Data services: the Rust intermediary layer

The third Rust migration at Discord is less discussed but arguably the most architecturally significant. Starting around 2022, Discord built an entirely new layer of Rust services sitting between their API monolith and their database clusters.

### The problem: hot partitions

Discord was migrating their message storage from Cassandra to [ScyllaDB](https://www.scylladb.com/) (a C++ rewrite of Cassandra). But switching databases didn't solve their core problem: hot partitions.

When a major event happens - a game launch announcement, a streamer going live, the 2022 FIFA World Cup Final - hundreds of thousands of users flood into the same channels simultaneously. All those read and write requests target the same database partitions. In Cassandra, this caused unbounded concurrency on those partitions, leading to cascading latency where every subsequent query piled up behind the slow ones.

### The architecture

The Rust data services sit between Discord's API monolith and ScyllaDB:

```
┌─────────────────────┐
│   API Monolith      │
│   (Python/Go)       │
└──────────┬──────────┘
           │ gRPC
┌──────────▼──────────┐
│   Rust Data Service │
│  - Request coalesce │
│  - Consistent route │
│  - No business logic│
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│     ScyllaDB        │
│   (72 nodes)        │
└─────────────────────┘
```

Each data service exposes roughly one gRPC endpoint per database query. They intentionally contain *no business logic* - they're pure data access coordinators. This is a deliberate design choice: keeping business logic out of the data layer means these services can focus entirely on database access patterns, concurrency control, and caching.

### Request coalescing

The key innovation in the Rust data services is request coalescing (sometimes called request deduplication or request collapsing).

When multiple users request the same data simultaneously - say, loading the same channel's message history - the data service executes only one database query instead of hundreds. The first request for a given key spawns an async task (a tokio task) that performs the actual database call. Subsequent requests for the same key don't spawn new tasks - they subscribe to the existing one and await the same result.

```
User A ─┐
User B ─┼─→ Single DB query ─→ Result broadcast to all
User C ─┘
```

In Rust with tokio, this pattern is natural to implement safely. You can use a `DashMap` (from the [`dashmap`](https://crates.io/crates/dashmap) crate) or a `tokio::sync::Mutex`-protected `HashMap` keyed by request parameters, where the value is a `tokio::sync::broadcast` or `tokio::sync::watch` channel. The first request inserts a channel and spawns the query task. Subsequent requests find the existing channel and subscribe. When the query completes, all subscribers get the result simultaneously.

Doing this in a language with a GC would work, but in a high-throughput data service, you'd face the same problem as Read States: thousands of in-flight request entries creating GC pressure. In Rust, completed entries are dropped immediately when the broadcast completes and all subscribers have received the result.

### Consistent hash routing

Request coalescing only works if requests for the same data land on the same service instance. Discord uses consistent hashing with the channel ID as the routing key. All requests for a given channel's data route to the same data service node, maximizing coalescing effectiveness.

This is a tradeoff: consistent routing means a hot channel creates a hot service node. But that's manageable because coalescing means the hot node isn't hammering the database - it's serving hundreds of clients from a single query result.

### The migration tool

For the actual Cassandra-to-ScyllaDB migration, Discord needed to move trillions of messages. They initially considered ScyllaDB's Apache Spark-based migrator, which estimated three months for the migration.

Instead, they wrote a custom migration tool in Rust. It reads token ranges from the source database, checkpoints progress locally in SQLite (so it can resume after crashes), and streams data into ScyllaDB at high throughput. Peak migration speed: **3.2 million messages per second**. Total migration time: **9 days** instead of three months.

### Results

After the migration to ScyllaDB with the Rust data services layer:

- **Node reduction**: 177 Cassandra nodes down to 72 ScyllaDB nodes
- **p99 read latency**: 15ms (was 40-125ms with Cassandra)
- **p99 write latency**: 5ms (was 5-70ms with Cassandra)
- **Operational burden**: the database went from being a frequent source of on-call pages to what the team described as "a quiet, well-behaved database"

The 2022 World Cup Final was the first major stress test. Traffic spiked massively and the system absorbed it without incident - a scenario that previously would have triggered cascading failures.

## What Discord kept in other languages

This is the part that often gets lost in "Company X switched to Rust" narratives. Discord didn't rewrite Discord in Rust. They rewrote three specific components:

**Still in Elixir:** The real-time gateway layer - the WebSocket connections, presence tracking, and message fanout. BEAM's process model handles millions of concurrent connections naturally, and Elixir's fault tolerance (supervisors, process isolation) is genuinely hard to replicate in Rust.

**Still in Python:** Parts of the API layer and internal tooling. Python's ecosystem for web APIs, its hiring pool, and its iteration speed make it the pragmatic choice for business logic that changes frequently.

**Still in Go:** Various backend services where GC pauses aren't a problem - services with small heaps, short-lived objects, or lower latency requirements.

**Rust for:** Stateful services with large in-memory caches (Read States), CPU-bound algorithmic bottlenecks in otherwise-Elixir services (SortedSet), and data access coordination layers with high concurrency requirements (data services).

## Lessons for picking Rust vs Go for a specific service

Discord's Read States migration is frequently cited in "Rust vs Go" debates, usually to argue that Rust is faster. That misses the point. The lesson is narrower and more useful:

**Evaluate based on your memory profile, not your language preference.** If your service processes requests statelessly (small heap, short-lived allocations), Go's GC overhead is effectively invisible and you'll ship faster. If your service holds millions of long-lived objects in memory and mutates them at high frequency, any tracing GC will create pauses proportional to your heap size. That's physics, not language quality.

**Consider Rust when your bottleneck is at a layer boundary.** Discord's data services don't contain business logic - they're infrastructure. Infrastructure code changes infrequently, runs for years, and needs to handle extreme load predictably. That's where Rust's upfront investment pays off most.

**Use Rust as an extension before using it as a replacement.** The SortedSet NIF approach - keeping 99% of the service in Elixir and dropping to Rust for one hot data structure - delivered massive gains with minimal disruption. If you have a BEAM, JVM, or Python service with one computational bottleneck, a Rust NIF/FFI/PyO3 extension might solve your problem without a full rewrite.

**Don't optimize for the wrong metric.** Discord's team was under 50 engineers supporting 250 million users when they made these decisions. They didn't have the headcount for speculative rewrites. Each migration targeted a specific, measured production problem with monitoring data proving the bottleneck existed before a single line of Rust was written.

The common thread across all three migrations: Discord didn't adopt Rust because they wanted to use Rust. They adopted it because they had specific problems - GC pauses, data structure performance in a functional runtime, database concurrency control - and Rust happened to be the best tool for each one. That's the only durable reason to rewrite anything.
