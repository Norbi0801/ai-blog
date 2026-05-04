+++
title = "Backpressure - when your system can't keep up"
date = 2025-06-10
description = "What backpressure is, why ignoring it kills production systems, and how to implement it in Rust with bounded channels, load shedding, and circuit breakers."

[taxonomies]
tags = ["rust", "architecture", "performance", "distributed-systems"]
+++

Your producer pushes 10,000 messages per second. Your consumer processes 2,000. The remaining 8,000 pile up in memory. In ten minutes, you're at 4.8 million queued messages. In an hour, the OOM killer ends your process.

This is what happens when you ignore backpressure.

<!-- more -->

## The core problem

Backpressure is a feedback mechanism. When a downstream component is slower than its upstream producer, the system needs a way to signal "slow down" back up the chain. Without it, you get unbounded queue growth, memory exhaustion, and cascading failures.

The term comes from fluid dynamics - literal pressure that opposes the flow in a pipe. In software, it's the same idea: the pipe between producer and consumer has limited capacity, and when the consumer can't drain it fast enough, pressure builds up.

The critical insight is that every system has a throughput limit. You can't make the consumer infinitely fast. The question is: what happens when you hit that limit?

There are exactly three options:

1. **Buffer** - store excess work somewhere (memory, disk, external queue)
2. **Drop** - reject excess work (return 503, drop messages)
3. **Block** - slow down the producer until the consumer catches up

Each has tradeoffs. Buffering delays the problem. Dropping loses work. Blocking reduces throughput. The right choice depends on your domain.

## TCP invented this decades ago

Before we write any Rust, it's worth understanding that TCP solved this problem in the 1980s. The mechanism is called flow control, and it's one of the most elegant backpressure implementations ever designed.

Every TCP segment carries a 16-bit field called the **receive window** (`rwnd`). The receiver advertises how many bytes of buffer space it has available. The sender is not allowed to have more unacknowledged bytes in flight than the receiver's advertised window.

Here's what happens step by step:

1. Receiver has a 64KB receive buffer
2. Receiver advertises `rwnd = 65535` in its ACK packets
3. Sender transmits data, receiver buffer fills up
4. Application reads slowly from the socket - buffer stays full
5. Receiver advertises `rwnd = 0` - "I'm full, stop sending"
6. Sender stops transmitting, starts a **persist timer**
7. Persist timer fires periodically, sender sends a 1-byte **window probe**
8. Eventually the application reads data, buffer space opens up
9. Receiver advertises `rwnd = 32768` - "I have room again"
10. Sender resumes transmission

This is blocking backpressure at the transport layer. The producer (sender) is forced to slow down because the consumer (receiver) explicitly communicates its capacity. No data is lost. No buffers explode.

The key design insight: **the consumer controls the flow, not the producer**. The producer doesn't guess how fast to send. The consumer tells it exactly how much room is available.

Every `TcpStream` in your Rust application benefits from this. If your HTTP handler is slow, TCP will automatically slow down the client. The problem is that application-level queues sit above TCP and break this natural backpressure chain.

## Where backpressure breaks down

Consider a typical async web service:

```rust
use tokio::sync::mpsc;

// This is the problem
let (tx, mut rx) = mpsc::unbounded_channel::<Job>();

// Producer: accepts every incoming request
tokio::spawn(async move {
    loop {
        let job = accept_request().await;
        // This never blocks. Never fails. Queue grows forever.
        tx.send(job).unwrap();
    }
});

// Consumer: processes jobs slowly
tokio::spawn(async move {
    while let Some(job) = rx.recv().await {
        process_job(job).await; // Takes 50ms per job
    }
});
```

The `unbounded_channel` is the root cause. It accepts messages without limit. If requests arrive faster than `process_job` can handle them, the channel grows until the process runs out of memory.

This pattern shows up everywhere: message queue consumers that buffer in memory, HTTP servers that accept connections into an unbounded queue, stream processors that read faster than they write.

## Bounded channels: the first line of defense

Tokio's bounded `mpsc::channel` (as of tokio 1.50.0) is the simplest backpressure mechanism in async Rust:

```rust
use tokio::sync::mpsc;

// Buffer up to 1000 messages. After that, send() will await.
let (tx, mut rx) = mpsc::channel::<Job>(1000);

tokio::spawn(async move {
    loop {
        let job = accept_request().await;
        // This awaits when the channel is full.
        // The producer is forced to slow down.
        if tx.send(job).await.is_err() {
            eprintln!("receiver dropped, shutting down");
            break;
        }
    }
});

tokio::spawn(async move {
    while let Some(job) = rx.recv().await {
        process_job(job).await;
    }
});
```

The function signature is straightforward:

```rust
pub fn channel<T>(buffer: usize) -> (Sender<T>, Receiver<T>)
```

The `buffer` parameter sets the capacity. When the channel holds `buffer` messages, `Sender::send()` becomes a pending future - it won't complete until the receiver pulls a message out. This is blocking backpressure: the producer physically cannot outrun the consumer.

Under the hood, tokio's bounded channel uses a semaphore. The sender acquires a permit before writing to the channel's internal buffer. When no permits are available (channel is full), the sender's future parks itself on the semaphore's wait list. When the receiver reads a message, it releases a permit, waking one parked sender.

This is efficient. No spin loops. No polling. The runtime schedules senders only when there's actual capacity.

### Choosing the buffer size

Buffer size is a tradeoff between latency and throughput:

- **Too small** (1-10): maximum backpressure sensitivity, but high contention between producer and consumer. Good for scenarios where you want immediate feedback.
- **Medium** (100-1000): absorbs short bursts without blocking the producer. Good default for most services.
- **Too large** (100,000+): you're basically an unbounded channel with extra steps. The backpressure kicks in too late to be useful.

A decent heuristic: set the buffer to roughly 2-5x what your consumer can process in one second. If your consumer handles 500 jobs/sec, a buffer of 1000-2500 lets it absorb ~2-5 seconds of burst traffic before the producer slows down.

### try_send: non-blocking alternative

Sometimes blocking the producer isn't acceptable. `try_send` returns immediately with an error if the channel is full:

```rust
use tokio::sync::mpsc;
use tokio::sync::mpsc::error::TrySendError;

let (tx, mut rx) = mpsc::channel::<Job>(100);

// Producer that drops work instead of blocking
tokio::spawn(async move {
    loop {
        let job = accept_request().await;
        match tx.try_send(job) {
            Ok(()) => { /* queued successfully */ }
            Err(TrySendError::Full(rejected_job)) => {
                // Channel full - shed this request
                eprintln!("backpressure: dropping job");
                increment_counter!("jobs.dropped");
            }
            Err(TrySendError::Closed(_)) => break,
        }
    }
});
```

This is **drop backpressure** - excess work is discarded. You get the rejected item back in `TrySendError::Full`, so you can log it, return a 503 to the caller, or route it to a dead-letter queue.

## Rate limiting

Bounded channels handle backpressure within a single process. Rate limiting handles it at the API boundary - controlling how many requests enter the system per time window.

The [tower](https://docs.rs/tower) ecosystem provides composable middleware for this. Here's a rate limiter with Axum:

```rust
use axum::{Router, routing::get};
use tower::ServiceBuilder;
use tower::limit::RateLimitLayer;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/api/process", get(handle_request))
        .layer(
            ServiceBuilder::new()
                // Allow 100 requests per second
                .layer(RateLimitLayer::new(100, Duration::from_secs(1)))
        );

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn handle_request() -> &'static str {
    "processed"
}
```

If you've read my [post on Rust web frameworks](/blog/axum-vs-actix-web-vs-warp-rust-web-frameworks-in-2026), you'll recognize tower's layer-based middleware composition. `RateLimitLayer` wraps your service and delays requests beyond the configured rate.

This is fundamentally different from bounded channels. Rate limiting enforces a fixed throughput ceiling regardless of actual capacity. If your service can handle 500 req/s but you limit to 100, you're leaving capacity on the table. If your service degrades at 80 req/s on a bad day, a 100 req/s limit still lets too much through.

Rate limiting is best for protecting external APIs where you want predictable behavior, not for dynamic backpressure.

## Load shedding: the 503 strategy

Load shedding is more adaptive than rate limiting. Instead of a fixed rate, the system monitors its own health and rejects requests when it's overloaded.

Tower provides `LoadShed` middleware that rejects requests when the inner service reports it's not ready:

```rust
use axum::{Router, routing::get};
use tower::ServiceBuilder;
use tower::load_shed::LoadShedLayer;

let app = Router::new()
    .route("/api/process", get(handle_request))
    .layer(
        ServiceBuilder::new()
            .layer(LoadShedLayer::new())
    );
```

But a more practical approach is building load shedding based on observable metrics - queue depth, latency, or concurrent request count:

```rust
use axum::{
    Router, routing::get,
    extract::State,
    http::StatusCode,
    response::IntoResponse,
};
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    in_flight: Arc<AtomicUsize>,
    max_concurrent: usize,
}

async fn handle_with_shedding(
    State(state): State<AppState>,
) -> impl IntoResponse {
    let current = state.in_flight.fetch_add(1, Ordering::Relaxed);

    // Shed load if we're over capacity
    if current >= state.max_concurrent {
        state.in_flight.fetch_sub(1, Ordering::Relaxed);
        return (
            StatusCode::SERVICE_UNAVAILABLE,
            "server overloaded, try again later"
        ).into_response();
    }

    // Do actual work
    let result = do_expensive_work().await;

    state.in_flight.fetch_sub(1, Ordering::Relaxed);
    result.into_response()
}
```

The `503 Service Unavailable` response is the HTTP equivalent of TCP's zero window. You're telling the upstream: "I have no capacity right now." A well-behaved client will back off and retry.

The beauty of this approach is that it's self-regulating. When the system recovers, it stops returning 503s automatically. No manual intervention, no configuration changes.

## Circuit breakers: protecting against cascading failure

A circuit breaker sits between your service and a downstream dependency. When the dependency starts failing, the circuit breaker "opens" and short-circuits requests without even attempting the call.

States:

- **Closed** - normal operation, requests pass through
- **Open** - dependency is failing, requests are immediately rejected
- **Half-open** - testing if the dependency has recovered

Here's a minimal implementation:

```rust
use std::sync::atomic::{AtomicU64, AtomicU8, Ordering};
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::Mutex;

const CLOSED: u8 = 0;
const OPEN: u8 = 1;
const HALF_OPEN: u8 = 2;

pub struct CircuitBreaker {
    state: AtomicU8,
    failure_count: AtomicU64,
    failure_threshold: u64,
    last_failure_time: Mutex<Option<Instant>>,
    recovery_timeout: Duration,
}

impl CircuitBreaker {
    pub fn new(failure_threshold: u64, recovery_timeout: Duration) -> Self {
        Self {
            state: AtomicU8::new(CLOSED),
            failure_count: AtomicU64::new(0),
            failure_threshold,
            last_failure_time: Mutex::new(None),
            recovery_timeout,
        }
    }

    pub async fn call<F, T, E>(&self, f: F) -> Result<T, CircuitError<E>>
    where
        F: std::future::Future<Output = Result<T, E>>,
    {
        match self.state.load(Ordering::Relaxed) {
            OPEN => {
                // Check if recovery timeout has elapsed
                let last = self.last_failure_time.lock().await;
                if let Some(t) = *last {
                    if t.elapsed() >= self.recovery_timeout {
                        drop(last);
                        self.state.store(HALF_OPEN, Ordering::Relaxed);
                        // Fall through to try the call
                    } else {
                        return Err(CircuitError::CircuitOpen);
                    }
                }
            }
            _ => {}
        }

        match f.await {
            Ok(result) => {
                // Success - reset to closed
                self.failure_count.store(0, Ordering::Relaxed);
                self.state.store(CLOSED, Ordering::Relaxed);
                Ok(result)
            }
            Err(e) => {
                let count = self.failure_count.fetch_add(1, Ordering::Relaxed) + 1;
                if count >= self.failure_threshold {
                    self.state.store(OPEN, Ordering::Relaxed);
                    *self.last_failure_time.lock().await = Some(Instant::now());
                }
                Err(CircuitError::Inner(e))
            }
        }
    }
}

pub enum CircuitError<E> {
    CircuitOpen,
    Inner(E),
}
```

The circuit breaker is a form of backpressure that propagates across service boundaries. When service B is overwhelmed and starts timing out, service A's circuit breaker opens and stops sending traffic. This prevents A from wasting resources on calls that will fail, and gives B breathing room to recover.

Without circuit breakers, a slow downstream service causes upstream services to accumulate blocked tasks, exhausting thread pools and connection pools. This is how a single slow database query takes down an entire microservice architecture.

## Detecting backpressure problems

You can't fix what you can't see. Here are the metrics that tell you backpressure is needed or failing:

### Queue depth monitoring

The most direct signal. If you're using bounded channels, track how full they are:

```rust
use tokio::sync::mpsc;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

struct MonitoredChannel<T> {
    tx: mpsc::Sender<T>,
    rx: mpsc::Receiver<T>,
    depth: Arc<AtomicUsize>,
    capacity: usize,
}

impl<T> MonitoredChannel<T> {
    fn new(capacity: usize) -> Self {
        let (tx, rx) = mpsc::channel(capacity);
        Self {
            tx,
            rx,
            depth: Arc::new(AtomicUsize::new(0)),
            capacity,
        }
    }

    async fn send(&self, item: T) -> Result<(), mpsc::error::SendError<T>> {
        self.tx.send(item).await?;
        let current = self.depth.fetch_add(1, Ordering::Relaxed);

        // Alert when queue is 80% full
        let utilization = (current + 1) as f64 / self.capacity as f64;
        if utilization > 0.8 {
            eprintln!(
                "WARNING: channel at {:.0}% capacity ({}/{})",
                utilization * 100.0,
                current + 1,
                self.capacity
            );
        }
        Ok(())
    }

    async fn recv(&mut self) -> Option<T> {
        let item = self.rx.recv().await?;
        self.depth.fetch_sub(1, Ordering::Relaxed);
        Some(item)
    }
}
```

In production, replace the `eprintln!` with actual metrics emission. If you're using [Prometheus](https://prometheus.io/), expose the queue depth as a gauge. Set alerts at 70-80% capacity - that gives you time to react before the channel fills completely.

Key metrics to track:

- **Queue depth** - how many items are waiting. Trending upward means your consumer is falling behind.
- **Queue wait time** - how long items sit in the queue before processing. This is the latency tax your users pay.
- **Producer block rate** - how often `send()` has to await. High block rates mean backpressure is actively engaging.
- **Drop rate** - if using `try_send`, how many items are being rejected. Spikes here mean you're shedding load.

If you've read my [post on load testing](/blog/load-testing-your-rust-api-tools-and-methodology), you'll know how to generate the traffic that reveals these limits. Run a ramp-up test, watch queue depth, and find the inflection point where your consumer starts falling behind.

### The RED method

For HTTP services, track **R**ate, **E**rror rate, and **D**uration:

- Rate dropping while request volume increases = backpressure is throttling
- Error rate spiking (503s) = load shedding is engaging
- Duration increasing = queues are filling up, requests are waiting longer

These three signals together tell you exactly what's happening.

## Real-world examples

### Discord's Read States service

Discord's [migration from Go to Rust](https://discord.com/blog/why-discord-is-switching-from-go-to-rust) for their Read States service is a canonical example of backpressure problems. The Go implementation suffered latency spikes every ~2 minutes from garbage collection forced to scan a large LRU cache. While the GC ran, the service couldn't process messages fast enough, creating implicit backpressure that manifested as user-visible lag.

The Rust rewrite eliminated GC pauses, but the architectural lesson is broader: any component that periodically stalls becomes a backpressure bottleneck. Garbage collectors, lock contention, disk I/O bursts - they all temporarily reduce consumer throughput, causing upstream queues to grow.

### Kafka consumers

Message queue consumers are backpressure's natural habitat. A Kafka consumer that processes messages slower than they're produced will fall behind, growing its consumer lag. Without proper handling:

1. Consumer lag grows continuously
2. Messages age in the topic, potentially hitting retention limits
3. The consumer's in-memory state grows (if it buffers batches)
4. Eventually, the consumer either OOMs or loses messages to retention

The fix is usually a combination of bounded internal buffers and horizontal scaling - add more consumer instances to increase aggregate throughput.

### The thundering herd

When a circuit breaker transitions from open to half-open, it allows a test request through. If that succeeds, the circuit closes and all queued requests flood the recovering service simultaneously. This is the thundering herd problem.

The solution: don't transition from open to fully closed instantly. Use a gradual ramp-up - send 10% of traffic, then 25%, then 50%, then 100%. Each step validates that the downstream service can actually handle the load.

## Putting it all together

A production system typically layers multiple backpressure mechanisms:

```
Client
  │
  ├─ Rate Limiter (tower::limit) ──── fixed ceiling
  │
  ├─ Load Shedder (503 responses) ─── adaptive rejection
  │
  ├─ Bounded Channel (mpsc::channel) ─ internal buffering
  │
  ├─ Circuit Breaker ──────────────── dependency protection
  │
  └─ Consumer
```

Each layer handles a different failure mode:

- **Rate limiter** prevents abuse and ensures fair resource allocation
- **Load shedder** protects against sudden traffic spikes that exceed rate limits
- **Bounded channels** prevent internal memory exhaustion
- **Circuit breakers** prevent cascading failures from downstream services

The order matters. Rate limiting happens first (cheapest to evaluate). Load shedding happens next (slightly more expensive, checks system state). Bounded channels handle the internal pipeline. Circuit breakers protect outbound calls.

## The golden rule

If you take one thing from this post: **never use an unbounded queue in production**. Every `unbounded_channel()`, every `Vec` that grows without limit, every connection pool without a max size - these are all ticking time bombs waiting for a traffic spike.

Bounded queues force you to answer the hard question upfront: what happens when the system is full? That's an uncomfortable question during development. It's a much worse question at 3 AM when your pager goes off.

TCP figured this out 40 years ago. Your application should too.
