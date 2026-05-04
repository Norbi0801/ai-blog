+++
title = "Writing a rate limiter in Rust"
date = 2025-05-13
description = "Building a token bucket rate limiter from scratch, adding per-key limiting, plugging it into Axum middleware, and knowing when to reach for governor instead."

[taxonomies]
tags = ["rust", "systems-programming", "networking"]
+++

Rate limiting is one of those things that sounds simple until you sit down to implement it. "Allow 100 requests per second" - how hard can it be? Then you start thinking about burst handling, per-user fairness, clock precision, concurrent access, and suddenly you're knee-deep in a design problem that has kept network engineers busy since the ATM protocol days.

This post builds a rate limiter from scratch in Rust. We start with the token bucket algorithm - the workhorse behind most production rate limiters - implement it step by step, add per-key limiting, wire it into HTTP middleware, and compare it to the other common algorithms. Then we look at when to throw it all away and use [governor](https://crates.io/crates/governor) instead.

<!-- more -->

## The token bucket - how it works

Imagine a bucket that holds tokens. Tokens drip in at a fixed rate - say 10 per second. Every request that arrives needs to grab a token from the bucket. If there's a token available, the request goes through and the token is consumed. If the bucket is empty, the request is rejected (or waits).

The bucket has a maximum capacity. When it's full, new tokens just overflow - they don't accumulate beyond the cap. This cap controls burst behavior: a client that's been idle for a while has a full bucket and can fire a burst of requests up to the bucket size, then falls back to the steady refill rate.

Three parameters define a token bucket:

- **capacity** - maximum tokens the bucket can hold (burst size)
- **refill_rate** - tokens added per second
- **tokens** - current count (starts at capacity)

The key insight: you don't actually need a background thread dripping tokens in. You can calculate how many tokens should have been added since the last check. This is called "passive replenishment" - the same trick [Firecracker's rate limiter](https://codecatalog.org/articles/firecracker-rate-limiting/) uses.

## Step 1: a basic RateLimiter

```rust
use std::time::Instant;

pub struct RateLimiter {
    capacity: f64,
    tokens: f64,
    refill_rate: f64, // tokens per second
    last_refill: Instant,
}

impl RateLimiter {
    pub fn new(capacity: u32, refill_rate: f64) -> Self {
        RateLimiter {
            capacity: capacity as f64,
            tokens: capacity as f64,
            refill_rate,
            last_refill: Instant::now(),
        }
    }

    fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.capacity);
        self.last_refill = now;
    }

    pub fn try_acquire(&mut self) -> bool {
        self.refill();
        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            true
        } else {
            false
        }
    }
}
```

`try_acquire` does two things: first, it calculates how many tokens have accumulated since the last call using elapsed time, capping at capacity. Then it tries to consume one token. If there's enough, it subtracts and returns `true`. Otherwise, the caller knows to reject or wait.

Why `f64` instead of integers? Because time is continuous. If your refill rate is 10 tokens/sec and 50ms have passed, you've earned 0.5 tokens. Integer math would round that to zero and you'd lose precision, especially at low rates or high-frequency checking.

Let's verify it works:

```rust
fn main() {
    let mut limiter = RateLimiter::new(5, 2.0); // 5 burst, 2/sec refill

    // Burn through the burst
    for i in 0..7 {
        println!("request {}: {}", i, limiter.try_acquire());
    }
    // Output: true true true true true false false

    // Wait 1 second - should get 2 more tokens
    std::thread::sleep(std::time::Duration::from_secs(1));

    for i in 0..3 {
        println!("after wait {}: {}", i, limiter.try_acquire());
    }
    // Output: true true false
}
```

Five requests go through immediately (the burst), then the bucket is empty. After one second, two tokens refill (rate = 2/sec), so two more requests succeed.

### What happens under the hood

The `refill` method is doing multiplication and a `min` - O(1), no allocations. `Instant::now()` on Linux calls `clock_gettime(CLOCK_MONOTONIC)`, which is a vDSO call - it doesn't even enter the kernel. On x86_64 it reads the TSC register and takes about 20-25 nanoseconds. So the overhead of our rate limiter per check is essentially one clock read plus some floating-point arithmetic.

The `Instant` type is monotonic by design - it can't go backwards, even if the system clock gets adjusted. This matters for rate limiting because a clock jump backward could suddenly give you a negative elapsed time and drain tokens. With `Instant`, `duration_since` is always non-negative.

## Step 2: async wait

`try_acquire` is non-blocking - it either succeeds or fails immediately. But sometimes you want the caller to wait until a token becomes available rather than getting rejected. If you've read the [Tokio post](/blog/understanding-tokio-the-rust-async-runtime-under-the-hood/), you know that blocking the thread is not an option in async code. We need an async `wait()` that yields to the runtime.

```rust
use std::time::Duration;

impl RateLimiter {
    pub fn time_until_available(&self) -> Duration {
        if self.tokens >= 1.0 {
            return Duration::ZERO;
        }
        let deficit = 1.0 - self.tokens;
        Duration::from_secs_f64(deficit / self.refill_rate)
    }
}
```

This calculates exactly how long until the next token arrives. If we need 0.3 tokens and the refill rate is 10/sec, that's 30 milliseconds.

But there's a problem. Our `RateLimiter` uses `&mut self` - we can't share it across async tasks. For the async version, we need to wrap it:

```rust
use std::sync::Mutex;
use std::sync::Arc;

#[derive(Clone)]
pub struct AsyncRateLimiter {
    inner: Arc<Mutex<RateLimiter>>,
}

impl AsyncRateLimiter {
    pub fn new(capacity: u32, refill_rate: f64) -> Self {
        AsyncRateLimiter {
            inner: Arc::new(Mutex::new(RateLimiter::new(capacity, refill_rate))),
        }
    }

    pub fn try_acquire(&self) -> bool {
        self.inner.lock().unwrap().try_acquire()
    }

    pub async fn wait(&self) {
        loop {
            let wait_time = {
                let mut limiter = self.inner.lock().unwrap();
                if limiter.try_acquire() {
                    return;
                }
                limiter.time_until_available()
            };
            tokio::time::sleep(wait_time).await;
        }
    }
}
```

A few design decisions worth noting:

**`Mutex` not `RwLock`.** Every operation mutates state (either consuming tokens or updating the refill timestamp), so we'd never get a shared read lock anyway. `Mutex` is simpler and slightly faster for this use case.

**`std::sync::Mutex` not `tokio::sync::Mutex`.** The lock is held for a few nanoseconds - just arithmetic, no I/O. A standard `Mutex` is fine here and cheaper than tokio's async-aware mutex. The rule of thumb: use `std::sync::Mutex` when the critical section is fast and doesn't `.await`. Use `tokio::sync::Mutex` when you need to hold the lock across await points.

**The loop.** Why not just sleep once? Because between calculating `wait_time` and actually sleeping, another task might grab the token. The loop handles this race: if someone else got the token while we slept, we recalculate and sleep again. In practice this loop almost never runs more than twice.

**Dropping the `MutexGuard` before the await.** The lock guard must not live across the `.await` point. If it did, the mutex would be held while the task is suspended, blocking every other task that needs rate limiting. We explicitly scope it with a block so the guard drops before `tokio::time::sleep`.

## Step 3: per-key limiting

A global rate limiter protects your server's total throughput. But usually you want per-user or per-IP limits - user A shouldn't be blocked because user B is hammering the API.

The pattern: a `HashMap` where each key gets its own limiter.

```rust
use std::collections::HashMap;
use std::hash::Hash;

pub struct KeyedRateLimiter<K: Eq + Hash> {
    limiters: Mutex<HashMap<K, RateLimiter>>,
    capacity: u32,
    refill_rate: f64,
}

impl<K: Eq + Hash + Clone> KeyedRateLimiter<K> {
    pub fn new(capacity: u32, refill_rate: f64) -> Self {
        KeyedRateLimiter {
            limiters: Mutex::new(HashMap::new()),
            capacity,
            refill_rate,
        }
    }

    pub fn try_acquire(&self, key: &K) -> bool {
        let mut map = self.limiters.lock().unwrap();
        let limiter = map
            .entry(key.clone())
            .or_insert_with(|| RateLimiter::new(self.capacity, self.refill_rate));
        limiter.try_acquire()
    }

    pub fn cleanup(&self) {
        let mut map = self.limiters.lock().unwrap();
        map.retain(|_, limiter| {
            let elapsed = limiter.last_refill.elapsed().as_secs();
            elapsed < 300 // remove entries idle for 5 minutes
        });
    }
}
```

If you read the [KV store post](/blog/writing-a-key-value-store-in-rust/), this pattern should look familiar - a `Mutex<HashMap>` with entries that get created on demand. The same trade-offs apply: a single `Mutex` means all keys contend on one lock. Under heavy load with many distinct keys, this becomes a bottleneck.

The fix is the same one we discussed there: sharding. Split the key space across multiple locked maps so that operations on different keys hit different locks:

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::Hasher;

pub struct ShardedKeyedLimiter<K: Eq + Hash> {
    shards: Vec<Mutex<HashMap<K, RateLimiter>>>,
    shard_count: usize,
    capacity: u32,
    refill_rate: f64,
}

impl<K: Eq + Hash + Clone> ShardedKeyedLimiter<K> {
    pub fn new(shard_count: usize, capacity: u32, refill_rate: f64) -> Self {
        let shards = (0..shard_count)
            .map(|_| Mutex::new(HashMap::new()))
            .collect();
        ShardedKeyedLimiter {
            shards,
            shard_count,
            capacity,
            refill_rate,
        }
    }

    fn shard_for(&self, key: &K) -> &Mutex<HashMap<K, RateLimiter>> {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        let idx = hasher.finish() as usize % self.shard_count;
        &self.shards[idx]
    }

    pub fn try_acquire(&self, key: &K) -> bool {
        let mut map = self.shard_for(key).lock().unwrap();
        let limiter = map
            .entry(key.clone())
            .or_insert_with(|| RateLimiter::new(self.capacity, self.refill_rate));
        limiter.try_acquire()
    }
}
```

With 16 shards, two concurrent requests from different users most likely hit different shards and don't block each other at all. This is the same approach [dashmap](https://crates.io/crates/dashmap) uses internally, and it's what governor uses for its `DashMapStateStore`.

### Memory management

There's an important question with per-key limiters: what happens to keys that stop showing up? A bot that hammers your API with random IPs creates a new limiter per IP. Without cleanup, that HashMap grows forever.

The `cleanup` method above removes entries that haven't been touched in 5 minutes. In production, you'd run this on a timer:

```rust
let limiter = Arc::new(KeyedRateLimiter::<String>::new(100, 10.0));
let cleanup_limiter = limiter.clone();

tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(60));
    loop {
        interval.tick().await;
        cleanup_limiter.cleanup();
    }
});
```

Governor handles this with its [`retain_recent()`](https://docs.rs/governor/latest/governor/struct.RateLimiter.html) method, which does the same thing - prune state for keys that haven't been seen recently.

## Sliding window vs fixed window vs token bucket

Before we go further, let's compare the three common rate limiting algorithms. Each makes different trade-offs.

### Fixed window counter

Divide time into fixed intervals (e.g., 1-minute windows). Count requests per window. Reset the counter when the window rolls over.

```rust
pub struct FixedWindow {
    count: u32,
    limit: u32,
    window_start: Instant,
    window_size: Duration,
}

impl FixedWindow {
    pub fn try_acquire(&mut self) -> bool {
        let now = Instant::now();
        if now.duration_since(self.window_start) >= self.window_size {
            self.window_start = now;
            self.count = 0;
        }
        if self.count < self.limit {
            self.count += 1;
            true
        } else {
            false
        }
    }
}
```

Simple and memory-efficient - one counter and one timestamp per key. But it has a nasty edge case: a client can fire `limit` requests at the end of one window and `limit` more at the start of the next, getting 2x the intended rate in a short burst spanning the boundary. If your limit is 100/minute, a client could send 200 requests in 2 seconds by timing it right.

### Sliding window log

Track the timestamp of every request. When a new request arrives, drop timestamps older than the window, then count what's left.

```rust
use std::collections::VecDeque;

pub struct SlidingWindowLog {
    timestamps: VecDeque<Instant>,
    limit: u32,
    window_size: Duration,
}

impl SlidingWindowLog {
    pub fn try_acquire(&mut self) -> bool {
        let now = Instant::now();
        let cutoff = now - self.window_size;
        while self.timestamps.front().is_some_and(|&t| t < cutoff) {
            self.timestamps.pop_front();
        }
        if (self.timestamps.len() as u32) < self.limit {
            self.timestamps.push_back(now);
            true
        } else {
            false
        }
    }
}
```

No boundary problem - the window slides smoothly. But memory usage is O(limit) per key because you're storing every timestamp. With a limit of 10,000 requests/minute and 100,000 users, that's up to a billion timestamps in the worst case. The `VecDeque` helps - `pop_front` is O(1) - but the storage cost is real.

### Sliding window counter

A compromise: keep counters for the current and previous windows, then weight them by how far into the current window you are.

```rust
pub struct SlidingWindowCounter {
    prev_count: u32,
    curr_count: u32,
    limit: u32,
    window_start: Instant,
    window_size: Duration,
}

impl SlidingWindowCounter {
    pub fn try_acquire(&mut self) -> bool {
        let now = Instant::now();
        let elapsed = now.duration_since(self.window_start);

        if elapsed >= self.window_size {
            self.prev_count = self.curr_count;
            self.curr_count = 0;
            self.window_start = now;
        }

        let weight = 1.0 - (elapsed.as_secs_f64() / self.window_size.as_secs_f64());
        let estimated = (self.prev_count as f64 * weight) + self.curr_count as f64;

        if estimated < self.limit as f64 {
            self.curr_count += 1;
            true
        } else {
            false
        }
    }
}
```

Two counters per key - fixed memory. The approximation is surprisingly accurate. Cloudflare [uses this approach](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) for their global rate limiting because it's both memory-efficient and smooth.

### The comparison

| | Fixed window | Sliding window log | Sliding window counter | Token bucket |
|---|---|---|---|---|
| Memory per key | 8 bytes | O(limit) | 16 bytes | 24 bytes |
| Boundary spikes | Yes (2x burst) | No | Minimal | No |
| Burst control | None | None | None | Explicit (capacity) |
| Precision | Window-granular | Exact | Approximate | Exact |
| Implementation | Trivial | Simple | Moderate | Simple |

Token bucket wins for most API rate limiting because it handles bursts explicitly. The capacity parameter gives you direct control over how much burst you allow, separate from the sustained rate. The [retry strategies post](/blog/retry-strategies-exponential-backoff-jitter-and-circuit-breakers/) covered the client side of this - jitter and backoff prevent thundering herds when clients get rate-limited. The token bucket is the server side of that story.

## When to use governor

Everything above is educational. For production, you should seriously consider [governor](https://crates.io/crates/governor) (v0.10.4, actively maintained). It implements the [Generic Cell Rate Algorithm](https://en.wikipedia.org/wiki/Generic_cell_rate_algorithm) (GCRA) - mathematically equivalent to a token bucket but more elegant.

GCRA replaces the "bucket of tokens" metaphor with a single timestamp: the Theoretical Arrival Time (TAT). When a request arrives, you compare its actual time to the TAT. If it's after `TAT - tolerance`, the request conforms. Then you update TAT to `TAT + interval` (or `now + interval` if the request arrived late). No counters, no refill calculations - just one timestamp comparison and one addition per check.

The governor API:

```rust
use governor::{Quota, RateLimiter};
use std::num::NonZeroU32;

// Direct (un-keyed) rate limiter: 50 requests per second
let limiter = RateLimiter::direct(Quota::per_second(NonZeroU32::new(50).unwrap()));

// Synchronous check
match limiter.check() {
    Ok(()) => { /* request allowed */ }
    Err(not_until) => {
        let wait = not_until.wait_time_from(governor::clock::DefaultClock::default().now());
        println!("rate limited, retry after {:?}", wait);
    }
}

// Async wait - resolves when a cell becomes available
limiter.until_ready().await;
```

For per-key limiting:

```rust
use governor::{Quota, RateLimiter};
use std::num::NonZeroU32;

// Keyed limiter - one state per key, backed by dashmap
let limiter = RateLimiter::keyed(
    Quota::per_second(NonZeroU32::new(10).unwrap())
);

// Per-IP check
let client_ip = "192.168.1.42";
match limiter.check_key(&client_ip) {
    Ok(()) => { /* allowed */ }
    Err(_) => { /* rate limited */ }
}

// Housekeeping - prune stale keys
limiter.retain_recent();
```

Governor handles everything we built manually: passive replenishment, concurrent access (using `DashMap` internally for keyed limiters), jitter support for async waiting, and stale key cleanup. The GCRA implementation tracks a single `Instant` per cell state, which is more memory-efficient than our `f64` tokens + `f64` capacity + `Instant` triple.

**When to roll your own vs use governor:**

Use governor when you need a correct, tested, production-ready limiter and you're fine with its API. It's well-maintained (899 stars, 605 commits, [latest release December 2025](https://github.com/boinkor-net/governor)), has `no_std` support, and handles edge cases around clock precision that we skipped.

Roll your own when you need a specialized algorithm (like the sliding window counter), when you're learning how rate limiting works (which is what this post is for), or when you need to integrate with an existing state store (Redis, database) that governor doesn't support out of the box.

## Middleware: rate limiting HTTP endpoints

The most common use case is HTTP rate limiting. With [tower](https://crates.io/crates/tower) and [axum](https://crates.io/crates/axum), you can write a middleware layer that intercepts requests before they reach your handlers.

Here's a rate-limiting middleware using our `KeyedRateLimiter`, keyed by client IP:

```rust
use axum::{
    body::Body,
    extract::ConnectInfo,
    http::{Request, Response, StatusCode},
    middleware::Next,
};
use std::net::SocketAddr;
use std::sync::Arc;

pub async fn rate_limit_middleware(
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
    request: Request<Body>,
    next: Next,
) -> Result<Response<Body>, StatusCode> {
    // In practice, extract this from app state
    let limiter: Arc<KeyedRateLimiter<String>> = get_limiter_from_state();

    let key = addr.ip().to_string();

    if !limiter.try_acquire(&key) {
        let response = Response::builder()
            .status(StatusCode::TOO_MANY_REQUESTS)
            .header("Retry-After", "1")
            .body(Body::from("rate limit exceeded"))
            .unwrap();
        return Ok(response);
    }

    Ok(next.run(request).await)
}
```

Wire it into your Axum router:

```rust
use axum::{Router, middleware, routing::get};

let limiter = Arc::new(KeyedRateLimiter::<String>::new(100, 10.0));

let app = Router::new()
    .route("/api/data", get(handler))
    .layer(middleware::from_fn(rate_limit_middleware));
```

A few things to be careful about:

**The `Retry-After` header.** [RFC 6585](https://datatracker.ietf.org/doc/html/rfc6585) defines `429 Too Many Requests` and recommends including `Retry-After` with the number of seconds the client should wait. Good clients respect this. The [retry strategies post](/blog/retry-strategies-exponential-backoff-jitter-and-circuit-breakers/) covered how well-behaved clients should handle 429 responses.

**X-Forwarded-For.** If your service sits behind a reverse proxy or load balancer, `ConnectInfo` gives you the proxy's IP, not the client's. You need to read the `X-Forwarded-For` or `X-Real-IP` header instead. But be careful - those headers can be spoofed by clients. Only trust them if you control the proxy layer.

**Different limits for different endpoints.** A login endpoint should have much tighter limits than a read-only data endpoint. You can create separate limiters or use a closure that picks parameters based on the route:

```rust
pub async fn tiered_rate_limit(
    request: Request<Body>,
    next: Next,
) -> Result<Response<Body>, StatusCode> {
    let path = request.uri().path();

    let (capacity, rate) = match path {
        p if p.starts_with("/api/auth") => (5, 1.0),     // 5 burst, 1/sec
        p if p.starts_with("/api/write") => (20, 5.0),   // 20 burst, 5/sec
        _ => (100, 50.0),                                  // 100 burst, 50/sec
    };

    // Create or fetch limiter with these params...
    // ...
    Ok(next.run(request).await)
}
```

### tower-governor: the ready-made option

If you went with governor, [tower-governor](https://github.com/benwis/tower-governor) wraps it into a Tower layer for direct use with Axum, Tonic, and Hyper:

```rust
use axum::{Router, routing::get};
use governor::Quota;
use std::num::NonZeroU32;
use tower_governor::{GovernorLayer, GovernorConfigBuilder};

let governor_conf = GovernorConfigBuilder::default()
    .per_second(10)
    .burst_size(30)
    .finish()
    .unwrap();

let app = Router::new()
    .route("/api/data", get(handler))
    .layer(GovernorLayer {
        config: governor_conf.into(),
    });
```

It handles IP extraction, 429 responses, and `Retry-After` headers automatically. For most production use cases, this is the right answer - you get correct rate limiting with about 10 lines of config.

## Production considerations

A few things we glossed over that matter in real deployments:

**Distributed rate limiting.** Everything in this post is in-process. If you run multiple instances behind a load balancer, each instance has its own limiter state. A client hitting different instances gets N times the intended rate (where N is instance count). For distributed limiting, you need shared state - typically Redis with [Lua scripts for atomic check-and-decrement](https://redis.io/glossary/rate-limiting/), or a dedicated service like [Envoy's rate limit service](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting).

**Clock drift.** Our implementation uses `Instant`, which is monotonic within a single process. Across processes or machines, clocks drift. Distributed rate limiters typically use Redis `PEXPIRE` for time tracking rather than relying on client-side clocks.

**Graceful degradation.** When a rate limiter's state store fails (Redis down, disk full), you have two choices: fail open (allow all requests) or fail closed (reject all). For most APIs, fail open is safer - you'd rather handle a brief spike than reject every single user because your Redis went away. The [monitoring post](/blog/monitoring-rust-applications-in-production/) covered how to set up alerts so you know when this happens.

**Response headers.** Beyond `Retry-After`, many APIs include `X-RateLimit-Limit` (the max), `X-RateLimit-Remaining` (tokens left), and `X-RateLimit-Reset` (when the limit resets). These help clients self-regulate before they hit 429s. It's the difference between a brick wall and a speed limit sign.

## The full Cargo.toml

Everything in this post compiles with:

```toml
[package]
name = "rate-limiter"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.8"
governor = "0.10"
tower-governor = "0.6"
dashmap = "6"
```

## Where to go from here

If you want to push this further:

1. **Add metrics.** Count how many requests are allowed vs rejected per key, per endpoint. Feed this into Prometheus or your metrics system. Rate limiter metrics are one of the best early warning signals for abuse or unexpected traffic patterns.

2. **Implement sliding window counter.** Take the sliding window counter code above, wrap it in the same `KeyedRateLimiter` pattern, and benchmark it against the token bucket. For high-cardinality key spaces (millions of distinct IPs), the sliding window counter's smaller memory footprint might win.

3. **Build a Redis-backed limiter.** Replace the in-memory `HashMap` with Redis `INCR` + `PEXPIRE` calls. This gives you distributed rate limiting across multiple instances. The tricky part is making the check-and-decrement atomic - you'll need a Lua script or Redis `MULTI`/`EXEC`.

4. **Try governor's jitter.** Governor's `until_ready_with_jitter` adds randomized delay to prevent synchronized retry storms. If you've implemented the retry strategies from the [backoff post](/blog/retry-strategies-exponential-backoff-jitter-and-circuit-breakers/), you'll recognize the same principle from the client side now applied server-side.

Rate limiting sits at the intersection of algorithms, systems programming, and API design. The token bucket is the foundation, but the interesting problems are in the details - distributed state, fair queuing, graceful degradation, and giving clients enough information to behave well. Start with governor or tower-governor for production, but build one from scratch at least once. The intuition you get from watching tokens drain and refill stays with you.
