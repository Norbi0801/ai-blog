+++
title = "The circuit breaker pattern - protecting your system from cascading failures"
date = 2026-02-28
description = "How the circuit breaker pattern prevents one failing dependency from taking down your entire system, with a from-scratch Rust implementation and real reqwest examples."

[taxonomies]
tags = ["rust", "architecture", "design-patterns", "resilience"]
+++

One of your upstream APIs starts responding slowly. Not down - just slow. 30-second timeouts instead of the usual 200ms. Your service has 200 threads, each now stuck waiting on that one dependency. Thread pool exhausted. Requests to every other endpoint start queueing. Health checks fail. The load balancer pulls you out of rotation. Users see 502s across the board. A single sluggish third-party service just cascaded into a full outage of your entire platform.

This is the failure mode that circuit breakers exist to prevent. The pattern, originally described by Michael Nygard in [Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/) and later popularized by Martin Fowler's [widely-referenced article](https://martinfowler.com/bliki/CircuitBreaker.html), borrows its name from electrical engineering. When current exceeds safe limits, a physical circuit breaker trips and cuts the circuit. No fire. Software circuit breakers do the same thing - when a dependency starts failing, stop calling it. Fail fast, let it recover, and try again later.

<!-- more -->

## The three states

A circuit breaker is a state machine with three states:

```text
         success
   ┌──────────────────┐
   │                   │
   ▼      failure      │
┌──────┐ threshold ┌──────┐  timeout  ┌───────────┐
│Closed├──────────►│ Open ├──────────►│ Half-Open │
└──┬───┘           └──────┘           └─────┬─────┘
   │                   ▲     failure        │
   │                   └────────────────────┘
   │                         success
   └─────────────────────────────────────────┘
              (from Half-Open)
```

**Closed** is the normal operating state. Every call goes through. The breaker tracks failures - either counting consecutive errors or measuring failure rate over a time window. As long as failures stay below a threshold, nothing changes.

**Open** means the breaker has tripped. Every call is rejected immediately with an error - no network request, no timeout wait, no resource consumption. The breaker holds a timer. When that timer expires, it transitions to Half-Open.

**Half-Open** is the probe state. The breaker lets a small number of calls through to test whether the downstream service has recovered. If they succeed, the breaker resets to Closed. If any fail, it trips back to Open, usually with a longer timeout than before.

The key insight is the Open state. Instead of 200 threads each discovering independently that the API is down (spending 30 seconds each doing it), one state transition happens and all subsequent calls fail instantly. Your thread pool stays healthy. Your other endpoints keep serving traffic. The failing dependency gets breathing room to recover instead of being hammered by retries.

## When to use a circuit breaker

Not every call needs a circuit breaker. Here's where they earn their keep:

**External API calls.** Third-party services are outside your control. Payment providers, geolocation APIs, email delivery services - any of them can degrade without warning. If you wrapped your external calls behind an adapter trait (something I covered in [the adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/)), adding a circuit breaker is a natural extension of that layer.

**Database connections.** When your database is overloaded, retrying with 200 connections simultaneously makes things worse, not better. A circuit breaker can stop the flood and give the database time to clear its backlog. If you've ever load tested a Rust API and watched the connection pool collapse, as I described in [the load testing post](/blog/load-testing-your-rust-api-tools-and-methodology/), a circuit breaker is one of the tools that prevents that in production.

**Rate-limited services.** When you're hitting a 429 (Too Many Requests), continuing to send traffic burns your rate limit budget and often results in escalating penalties - longer cooldowns, IP blocks, account suspensions. A circuit breaker stops the bleed.

**Internal microservices.** In a service mesh, one degraded service can poison everything upstream. Netflix learned this the hard way, which is why they built [Hystrix](https://github.com/Netflix/Hystrix) (now in maintenance mode). At their scale, the Netflix API processes over 10 billion Hystrix Command executions per day.

You probably don't need a circuit breaker for local computation, in-memory caches, or operations that are already fast-failing. The pattern adds overhead - state tracking, synchronization, timer management. Use it for I/O boundaries where failures are slow and expensive.

## Building one from scratch in Rust

Let's build a circuit breaker step by step. Not because you should use a hand-rolled one in production (use a crate), but because understanding the internals makes you a better consumer of the libraries.

### State representation

```rust
use std::time::{Duration, Instant};

#[derive(Debug)]
enum State {
    Closed {
        failure_count: u32,
    },
    Open {
        until: Instant,
        backoff: Duration,
    },
    HalfOpen {
        backoff: Duration,
    },
}

#[derive(Debug, Clone)]
pub struct CircuitBreakerConfig {
    pub failure_threshold: u32,
    pub initial_backoff: Duration,
    pub max_backoff: Duration,
    pub backoff_multiplier: f64,
}

impl Default for CircuitBreakerConfig {
    fn default() -> Self {
        Self {
            failure_threshold: 5,
            initial_backoff: Duration::from_secs(10),
            max_backoff: Duration::from_secs(300),
            backoff_multiplier: 2.0,
        }
    }
}
```

A few deliberate choices here. The `Open` state stores both an `Instant` (when to transition to Half-Open) and the current `Duration` (so we can escalate on repeated failures). The `Closed` state holds a failure counter rather than a ring buffer - simpler to start with. The config separates the threshold from the backoff parameters.

The `State` enum is 40 bytes on x86-64: the discriminant (1 byte, padded to 8 for alignment), plus the largest variant. `Open` holds an `Instant` (8 or 16 bytes depending on platform) and a `Duration` (16 bytes: two u64 fields for seconds and nanoseconds). The compiler packs it efficiently.

### The core struct

```rust
use std::sync::Mutex;

pub struct CircuitBreaker {
    state: Mutex<State>,
    config: CircuitBreakerConfig,
}

#[derive(Debug, Clone, PartialEq)]
pub enum CircuitError<E> {
    /// The underlying operation failed.
    Inner(E),
    /// The circuit breaker rejected the call.
    Rejected,
}

impl CircuitBreaker {
    pub fn new(config: CircuitBreakerConfig) -> Self {
        Self {
            state: Mutex::new(State::Closed { failure_count: 0 }),
            config,
        }
    }

    pub fn is_call_permitted(&self) -> bool {
        let mut state = self.state.lock().unwrap();
        match *state {
            State::Closed { .. } => true,
            State::Open { until, backoff } => {
                if Instant::now() >= until {
                    *state = State::HalfOpen { backoff };
                    true
                } else {
                    false
                }
            }
            State::HalfOpen { .. } => true,
        }
    }

    pub fn record_success(&self) {
        let mut state = self.state.lock().unwrap();
        match *state {
            State::HalfOpen { .. } => {
                *state = State::Closed { failure_count: 0 };
            }
            State::Closed { .. } => {
                *state = State::Closed { failure_count: 0 };
            }
            _ => {}
        }
    }

    pub fn record_failure(&self) {
        let mut state = self.state.lock().unwrap();
        match *state {
            State::Closed { failure_count } => {
                let new_count = failure_count + 1;
                if new_count >= self.config.failure_threshold {
                    *state = State::Open {
                        until: Instant::now() + self.config.initial_backoff,
                        backoff: self.config.initial_backoff,
                    };
                } else {
                    *state = State::Closed {
                        failure_count: new_count,
                    };
                }
            }
            State::HalfOpen { backoff } => {
                let next_backoff = Duration::from_secs_f64(
                    (backoff.as_secs_f64() * self.config.backoff_multiplier)
                        .min(self.config.max_backoff.as_secs_f64()),
                );
                *state = State::Open {
                    until: Instant::now() + next_backoff,
                    backoff: next_backoff,
                };
            }
            _ => {}
        }
    }
}
```

Notice how `record_failure` in the `HalfOpen` state escalates the backoff. If the initial backoff was 10 seconds and the multiplier is 2.0, the sequence goes: 10s, 20s, 40s, 80s, 160s, 300s (capped at `max_backoff`). This is exponential backoff on the breaker level. The system backs off progressively further when the dependency keeps failing, instead of retrying at a fixed interval.

### Wrapping calls

Now add a `call` method that ties it all together:

```rust
impl CircuitBreaker {
    pub fn call<F, T, E>(&self, f: F) -> Result<T, CircuitError<E>>
    where
        F: FnOnce() -> Result<T, E>,
    {
        if !self.is_call_permitted() {
            return Err(CircuitError::Rejected);
        }

        match f() {
            Ok(value) => {
                self.record_success();
                Ok(value)
            }
            Err(err) => {
                self.record_failure();
                Err(CircuitError::Inner(err))
            }
        }
    }
}
```

And an async variant:

```rust
impl CircuitBreaker {
    pub async fn call_async<F, Fut, T, E>(&self, f: F) -> Result<T, CircuitError<E>>
    where
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = Result<T, E>>,
    {
        if !self.is_call_permitted() {
            return Err(CircuitError::Rejected);
        }

        match f().await {
            Ok(value) => {
                self.record_success();
                Ok(value)
            }
            Err(err) => {
                self.record_failure();
                Err(CircuitError::Inner(err))
            }
        }
    }
}
```

One subtlety: we check `is_call_permitted()` before awaiting the future, but we hold no lock during the `.await`. The `Mutex` is only locked briefly inside `is_call_permitted()`, `record_success()`, and `record_failure()`. This is important - holding a `std::sync::Mutex` across an `.await` would block the async runtime. If you need finer-grained control in high-contention scenarios, consider `tokio::sync::Mutex` or an atomic state representation, but for most use cases the brief lock is fine.

## Exponential backoff - why it matters

The escalating backoff is worth discussing separately because it's often missing from naive implementations. A fixed timeout (say, always 30 seconds) creates a predictable thundering herd problem: every 30 seconds, the breaker transitions to Half-Open, sends a probe, fails, reopens. If you have 50 service instances, they all probe at roughly the same interval.

Exponential backoff breaks this synchronization. After the first trip: 10 seconds. After the second: 20. Then 40, 80, 160. Different instances trip at different times and back off at different rates, spreading the probe load. Adding jitter improves this further:

```rust
use rand::Rng;

fn next_backoff_with_jitter(
    current: Duration,
    multiplier: f64,
    max: Duration,
) -> Duration {
    let base = current.as_secs_f64() * multiplier;
    let capped = base.min(max.as_secs_f64());
    // Full jitter: uniform random between 0 and the calculated backoff
    let jittered = rand::rng().random_range(0.0..=capped);
    Duration::from_secs_f64(jittered)
}
```

Full jitter (random between 0 and the backoff duration) is what AWS [recommends](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) for their services. It distributes probes more evenly than equal jitter (base/2 + random(0, base/2)) or no jitter at all.

## Circuit breaker vs retry

These two patterns get confused constantly. They solve different problems at different layers.

**Retry** is a per-request strategy. A single call fails, you try it again with some delay. It's optimistic - it assumes the failure was transient (a dropped packet, a momentary hiccup). Retry logic lives inside the individual request path.

**Circuit breaker** is a system-wide strategy. It tracks failures across all requests to a dependency and makes a global decision: "this thing is down, stop trying." It's protective - it assumes the failure is systemic and prevents your system from contributing to the problem.

Here's the difference in code:

```rust
// Retry: per-request, optimistic
async fn fetch_with_retry(url: &str) -> Result<String, reqwest::Error> {
    let client = reqwest::Client::new();
    let mut last_err = None;
    for attempt in 0..3 {
        match client.get(url).send().await {
            Ok(resp) => return resp.text().await,
            Err(e) => {
                last_err = Some(e);
                tokio::time::sleep(Duration::from_millis(100 * 2u64.pow(attempt))).await;
            }
        }
    }
    Err(last_err.unwrap())
}

// Circuit breaker: system-wide, protective
async fn fetch_with_breaker(
    breaker: &CircuitBreaker,
    client: &reqwest::Client,
    url: &str,
) -> Result<String, CircuitError<reqwest::Error>> {
    breaker.call_async(|| async {
        client
            .get(url)
            .timeout(Duration::from_secs(5))
            .send()
            .await?
            .text()
            .await
    }).await
}
```

The retry function doesn't know or care about other requests. If 500 requests all hit a failing endpoint, you get 1,500 total attempts (500 x 3 retries). The circuit breaker version would let the first few fail, trip the breaker, and reject the remaining 495 instantly.

In practice, you combine both. Retry handles transient blips within a closed circuit. The circuit breaker handles sustained outages across the whole system:

```rust
async fn fetch_resilient(
    breaker: &CircuitBreaker,
    client: &reqwest::Client,
    url: &str,
) -> Result<String, CircuitError<reqwest::Error>> {
    breaker.call_async(|| async {
        let mut last_err = None;
        for attempt in 0..3 {
            match client
                .get(url)
                .timeout(Duration::from_secs(5))
                .send()
                .await
            {
                Ok(resp) => return resp.text().await,
                Err(e) if e.is_timeout() || e.is_connect() => {
                    last_err = Some(e);
                    tokio::time::sleep(Duration::from_millis(100 * 2u64.pow(attempt))).await;
                }
                Err(e) => return Err(e),
            }
        }
        Err(last_err.unwrap())
    }).await
}
```

The retry runs inside the circuit breaker. If all retries exhaust and the call still fails, the breaker records one failure. After `failure_threshold` such exhausted-retry failures, the breaker trips. This way transient errors get retried transparently, but sustained outages trigger the breaker.

## Using existing crates

Don't ship a hand-rolled circuit breaker in production. Use one of these:

### failsafe (v1.3.0, ~2.9M recent downloads)

The most popular Rust circuit breaker crate. Built around a `Config` builder with pluggable backoff strategies and failure policies.

```rust
use failsafe::backoff;
use failsafe::failure_policy;
use failsafe::CircuitBreaker;
use failsafe::Config;
use std::time::Duration;

// Exponential backoff from 5s to 120s, trips after 3 consecutive failures
let backoff = backoff::exponential(
    Duration::from_secs(5),
    Duration::from_secs(120),
);
let policy = failure_policy::consecutive_failures(3, backoff);
let breaker = Config::new().failure_policy(policy).build();

// Sync usage
match breaker.call(|| external_api_call()) {
    Ok(result) => println!("got: {result}"),
    Err(failsafe::Error::Rejected) => println!("circuit open, fast-failing"),
    Err(failsafe::Error::Inner(e)) => println!("call failed: {e}"),
}
```

failsafe also supports `success_rate_over_time_window` as a failure policy (trip when success rate drops below a threshold over a sliding window) and jittered backoff variants (`equal_jittered`, `full_jittered`). It works with futures through the `failsafe::futures::CircuitBreaker` trait.

Source: [dmexe/failsafe-rs](https://github.com/dmexe/failsafe-rs) on GitHub.

### recloser (v1.3.1, ~755K recent downloads)

A newer alternative built on ring buffers instead of simple counters. The ring buffer approach means it makes decisions based on failure rate over a window of recent calls rather than consecutive failures.

```rust
use recloser::Recloser;
use std::time::Duration;

let breaker = Recloser::custom()
    .error_rate(0.5)       // trip at 50% failure rate
    .closed_len(100)       // track last 100 calls in Closed
    .half_open_len(10)     // track 10 probe calls in Half-Open
    .open_wait(Duration::from_secs(30))
    .build();

// Wrapping a call
match breaker.call(|| risky_operation()) {
    Ok(val) => println!("success: {val:?}"),
    Err(recloser::Error::Rejected) => println!("circuit open"),
    Err(recloser::Error::Inner(e)) => println!("operation failed: {e}"),
}
```

The ring buffer implementation means recloser doesn't trip on a single burst of failures if overall success rate is high. It also supports async through `AsyncRecloser`.

Source: [recloser on docs.rs](https://docs.rs/recloser/1.3.1/recloser/).

### Choosing between them

Use failsafe if you want battle-tested code with flexible backoff strategies. Use recloser if you prefer rate-based tripping over consecutive-failure tripping. Both support async, both are `Send + Sync`, both work behind an `Arc` for sharing across tasks.

## A real example: protecting reqwest calls

Here's a complete, runnable example that wraps an HTTP client with a circuit breaker. This is closer to what you'd actually deploy:

```rust
use failsafe::backoff;
use failsafe::failure_policy;
use failsafe::futures::CircuitBreaker;
use failsafe::Config;
use reqwest::Client;
use serde::Deserialize;
use std::sync::Arc;
use std::time::Duration;

#[derive(Debug, Deserialize)]
struct ApiResponse {
    ip: String,
}

#[derive(Clone)]
struct ProtectedClient {
    http: Client,
    breaker: Arc<Config<failsafe::failure_policy::ConsecutiveFailures<failsafe::backoff::Exponential>>>,
}

impl ProtectedClient {
    fn new() -> Self {
        let backoff = backoff::exponential(
            Duration::from_secs(5),
            Duration::from_secs(120),
        );
        let policy = failure_policy::consecutive_failures(5, backoff);

        Self {
            http: Client::builder()
                .timeout(Duration::from_secs(10))
                .connect_timeout(Duration::from_secs(3))
                .build()
                .expect("failed to build HTTP client"),
            breaker: Arc::new(Config::new().failure_policy(policy).build()),
        }
    }

    async fn get_ip(&self) -> Result<String, String> {
        match self.breaker.call(async {
            let resp = self.http
                .get("https://httpbin.org/ip")
                .send()
                .await
                .map_err(|e| format!("request failed: {e}"))?;

            let body: ApiResponse = resp
                .json()
                .await
                .map_err(|e| format!("parse failed: {e}"))?;

            Ok(body.ip)
        }).await {
            Ok(ip) => Ok(ip),
            Err(failsafe::Error::Rejected) => {
                Err("circuit breaker open - service unavailable".to_string())
            }
            Err(failsafe::Error::Inner(e)) => Err(e),
        }
    }
}

#[tokio::main]
async fn main() {
    let client = ProtectedClient::new();

    for i in 0..20 {
        match client.get_ip().await {
            Ok(ip) => println!("[{i}] IP: {ip}"),
            Err(e) => println!("[{i}] Error: {e}"),
        }
        tokio::time::sleep(Duration::from_millis(500)).await;
    }
}
```

The `ProtectedClient` is `Clone` and can be shared across request handlers in an Axum or Actix app. The `Arc` around the breaker means all tasks share the same state. If you're using dependency injection (covered in [the DI patterns post](/blog/dependency-injection-patterns-without-a-framework/)), you'd wire this as a constructor-injected dependency into your services.

## Monitoring and alerting

A circuit breaker that trips silently is barely useful. You need visibility into state transitions and rejection rates.

### What to track

**State transitions.** Every Closed-to-Open transition is an incident signal. Log it with the dependency name, timestamp, and failure count at trip time. failsafe supports this through the `Instrument` trait:

```rust
use failsafe::Instrument;

struct BreakerMetrics;

impl Instrument for BreakerMetrics {
    fn on_call_rejected(&self) {
        // Increment counter: circuit_breaker_rejections_total
        eprintln!("[circuit-breaker] call rejected (circuit open)");
    }

    fn on_open(&self) {
        // Set gauge: circuit_breaker_state = open
        eprintln!("[circuit-breaker] state -> OPEN");
    }

    fn on_half_open(&self) {
        eprintln!("[circuit-breaker] state -> HALF-OPEN");
    }

    fn on_closed(&self) {
        eprintln!("[circuit-breaker] state -> CLOSED");
    }
}
```

In production, replace the `eprintln!` calls with actual metrics emission - Prometheus counters via the `metrics` crate, or structured JSON logs via `tracing`.

**Rejection rate.** If your breaker is rejecting 80% of requests to a dependency, that's not a blip. Set an alert. A good threshold: alert if rejections exceed 10% of total calls to a given dependency over a 5-minute window.

**Open duration.** How long the breaker stays open tells you about recovery patterns. If it's oscillating rapidly (open, half-open, open, half-open), the dependency is flapping and you might need to increase the backoff or investigate the root cause.

**Fallback utilization.** When the breaker is open, what does your application do? Return cached data? A degraded response? Nothing? Track which fallback path executes and how often. A circuit breaker without a fallback strategy just converts slow failures into fast failures - better, but not great.

### Alerting rules

A reasonable starting point:

| Metric | Condition | Severity |
|---|---|---|
| State transition to Open | Any occurrence | Warning |
| Open duration > 5 minutes | Sustained | Critical |
| Rejection rate > 10% over 5 min | Sustained | Warning |
| Rejection rate > 50% over 5 min | Sustained | Critical |
| Half-Open probe failure rate > 80% | Over 3 cycles | Critical |

Wire these into your existing alerting pipeline - PagerDuty, OpsGenie, Slack, whatever your team uses.

## Common mistakes

**Setting the failure threshold too low.** A threshold of 1 or 2 means a single timeout trips the breaker. Network blips happen. Start with 5 consecutive failures or a 50% failure rate over 100 calls and tune from there.

**Not setting timeouts on the underlying calls.** A circuit breaker doesn't help if each failure takes 60 seconds to detect. Set aggressive `connect_timeout` (2-3 seconds) and `timeout` (5-10 seconds) on your HTTP client. The breaker only reacts after the call returns - if the call hangs forever, the breaker never records anything.

**Sharing one breaker across unrelated dependencies.** Each dependency needs its own circuit breaker. If your payment API and your email API share a breaker, email failures will block payment processing. One breaker per external dependency, named and tracked independently.

**Forgetting the fallback.** When the breaker is open, returning a raw error is the minimum. Better options: serve stale cached data, return a default value, queue the operation for later, or degrade gracefully (show a page without personalization instead of failing entirely).

## What we covered

The circuit breaker is a state machine with three states - Closed, Open, Half-Open - that protects your system from cascading failures by fast-failing requests to a degraded dependency. It's complementary to retries (per-request optimism vs system-wide protection), works best with exponential backoff and jitter, and requires monitoring to be truly useful. In Rust, `failsafe` and `recloser` are production-ready implementations that handle the concurrency details for you.

The pattern is one piece of a larger resilience toolkit. Combined with timeouts, retries, bulkheads (thread pool isolation), and fallbacks, it keeps a single point of failure from becoming a system-wide outage. Start simple - one breaker around your most critical external dependency - and expand from there.
