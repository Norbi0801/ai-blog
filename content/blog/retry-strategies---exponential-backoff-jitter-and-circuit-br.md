+++
title = "Retry strategies - exponential backoff, jitter, and circuit breakers"
date = 2025-12-17
description = "Why naive retries amplify failures, how exponential backoff and jitter prevent thundering herds, and when a circuit breaker should replace retries entirely."

[taxonomies]
tags = ["rust", "resilience", "async", "architecture"]
+++

A request fails. You retry it. It fails again. You retry it again, along with the 10,000 other clients that got the same failure. The server, which was struggling before, now receives 30,000 requests instead of 10,000. Congratulations - your retry logic just turned a partial outage into a total one.

Retries are one of those things that seem trivially simple until they're not. The gap between "try again" and "try again correctly" is where production incidents live.

<!-- more -->

## The retry spectrum

There are four common retry strategies, each progressively smarter:

| Strategy | Delay sequence | Use case |
|----------|---------------|----------|
| Immediate retry | 0, 0, 0, ... | Almost never. Maybe local in-memory operations |
| Fixed delay | 1s, 1s, 1s, ... | Simple cases with a single client |
| Exponential backoff | 1s, 2s, 4s, 8s, ... | Multiple clients hitting one endpoint |
| Backoff + jitter | ~0.7s, ~2.3s, ~3.1s, ~9.8s | Production systems with concurrent clients |

The first two are fine in isolation. The moment you have multiple callers retrying against the same backend, you need exponential backoff. And the moment those callers' retry timers are synchronized (they all failed at the same time, so they all retry at the same time), you need jitter.

## Fixed delay: the starting point

The simplest retry. Wait N seconds, try again:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn retry_fixed<F, Fut, T, E>(
    mut op: F,
    max_retries: u32,
    delay: Duration,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut attempts = 0;
    loop {
        match op().await {
            Ok(v) => return Ok(v),
            Err(e) => {
                attempts += 1;
                if attempts >= max_retries {
                    return Err(e);
                }
                sleep(delay).await;
            }
        }
    }
}
```

If you've read my [Tokio internals post](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/), you know that `tokio::time::sleep` doesn't actually block a thread - it registers a timer with the runtime's time driver and yields the task. The executor is free to run other futures while your retry waits.

Fixed delay works when you're the only client. One CLI tool retrying a flaky API? Fine. But picture 500 microservice instances all retrying a database that just came back up, all with a 1-second fixed delay. Every second, 500 connections hit the database simultaneously. The database staggers, fails again, and the cycle repeats. This is the thundering herd.

## Exponential backoff

The idea: each consecutive failure increases the wait time multiplicatively. The standard formula:

```
delay = min(base * 2^attempt, max_delay)
```

With a 1-second base and 60-second cap:

- Attempt 0: 1s
- Attempt 1: 2s
- Attempt 2: 4s
- Attempt 3: 8s
- Attempt 4: 16s
- Attempt 5: 32s
- Attempt 6: 60s (capped)

This gives the failing service progressively more breathing room. By the fifth retry, you're only hitting it every 32 seconds instead of every second.

Here's a from-scratch implementation:

```rust
use std::time::Duration;
use tokio::time::sleep;

struct ExponentialBackoff {
    base: Duration,
    max_delay: Duration,
    max_retries: u32,
}

impl ExponentialBackoff {
    fn delay_for(&self, attempt: u32) -> Duration {
        let delay = self.base.saturating_mul(2u32.saturating_pow(attempt));
        delay.min(self.max_delay)
    }
}

async fn retry_exponential<F, Fut, T, E>(
    config: &ExponentialBackoff,
    mut op: F,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut attempt = 0;
    loop {
        match op().await {
            Ok(v) => return Ok(v),
            Err(e) => {
                attempt += 1;
                if attempt >= config.max_retries {
                    return Err(e);
                }
                let delay = config.delay_for(attempt - 1);
                tracing::warn!(
                    attempt,
                    delay_ms = delay.as_millis() as u64,
                    "operation failed, backing off"
                );
                sleep(delay).await;
            }
        }
    }
}
```

Note `saturating_mul` and `saturating_pow` instead of regular arithmetic. Without saturation, `2^32 * 1_000_000_000` (nanoseconds in a second) overflows a `u64` at attempt 27. In production, you'd hit `max_delay` long before that, but defensive math costs nothing.

Exponential backoff solves the "hammering a dead server" problem. But it introduces a subtler one.

## The thundering herd and jitter

Consider 1,000 clients that all failed at time T=0. With exponential backoff and no jitter:

- T=1s: all 1,000 clients retry
- T=3s: all remaining failures retry
- T=7s: all remaining failures retry

The requests aren't spread out - they're just clustered at different points. You've replaced a constant barrage with periodic stampedes. The [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) demonstrated this clearly: pure exponential backoff without jitter can be worse than fixed delay when contention is high.

Jitter adds randomness to the delay so that clients naturally spread themselves out. There are three common jitter strategies:

### Full jitter

```rust
use rand::Rng;

fn full_jitter(base: Duration, attempt: u32, max_delay: Duration) -> Duration {
    let ceiling = base.saturating_mul(2u32.saturating_pow(attempt)).min(max_delay);
    let jittered = rand::rng().random_range(Duration::ZERO..=ceiling);
    jittered
}
```

The delay is uniformly random between 0 and the exponential ceiling. This produces the best spread across clients. The downside: you can get a delay of near-zero, which occasionally means a very aggressive retry.

### Equal jitter

```rust
fn equal_jitter(base: Duration, attempt: u32, max_delay: Duration) -> Duration {
    let ceiling = base.saturating_mul(2u32.saturating_pow(attempt)).min(max_delay);
    let half = ceiling / 2;
    let jittered = half + rand::rng().random_range(Duration::ZERO..=half);
    jittered
}
```

Half the exponential value is guaranteed, the other half is randomized. You always wait at least half the backoff period - no near-zero delays. A reasonable middle ground.

### Decorrelated jitter

```rust
fn decorrelated_jitter(
    base: Duration,
    previous_delay: Duration,
    max_delay: Duration,
) -> Duration {
    let ceiling = previous_delay.saturating_mul(3).max(base);
    let jittered = rand::rng().random_range(base..=ceiling);
    jittered.min(max_delay)
}
```

This one doesn't use the attempt number at all. Each delay is based on the previous delay, multiplied by a factor (typically 3), then randomized. It tends to produce longer backoff sequences than full jitter, which is useful when you'd rather err on the side of waiting longer.

AWS's analysis showed **full jitter** completing the fastest and generating the fewest total calls in high-contention scenarios. Equal jitter is a close second with slightly more predictable behavior. Unless you have a specific reason to choose otherwise, full jitter is the default choice.

Let's put it together into a complete retry function:

```rust
use rand::Rng;
use std::time::Duration;
use tokio::time::sleep;

pub struct RetryConfig {
    pub base_delay: Duration,
    pub max_delay: Duration,
    pub max_retries: u32,
}

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            base_delay: Duration::from_secs(1),
            max_delay: Duration::from_secs(60),
            max_retries: 5,
        }
    }
}

pub async fn retry_with_jitter<F, Fut, T, E>(
    config: &RetryConfig,
    mut op: F,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
    E: std::fmt::Display,
{
    let mut attempt = 0;
    loop {
        match op().await {
            Ok(v) => return Ok(v),
            Err(e) => {
                attempt += 1;
                if attempt >= config.max_retries {
                    return Err(e);
                }
                let ceiling = config
                    .base_delay
                    .saturating_mul(2u32.saturating_pow(attempt - 1))
                    .min(config.max_delay);
                let delay = rand::rng().random_range(Duration::ZERO..=ceiling);
                tracing::warn!(
                    attempt,
                    max_retries = config.max_retries,
                    delay_ms = delay.as_millis() as u64,
                    error = %e,
                    "retrying after failure"
                );
                sleep(delay).await;
            }
        }
    }
}
```

## The idempotency prerequisite

Before you add retries to anything, ask one question: **is this operation safe to repeat?**

A GET request is idempotent by definition. Creating a payment is not. If you retry a `POST /payments` and the first request actually succeeded (you just didn't get the response), you've charged the customer twice.

Idempotency requires either:

1. **Natural idempotency** - the operation is inherently safe to repeat (reads, upserts, deletes by ID)
2. **Idempotency keys** - the client sends a unique key, the server deduplicates

If you're building a webhook receiver, I covered idempotency with `DashMap` and TTL-based deduplication in the [webhook receiver post](/blog/building-a-webhook-receiver-in-rust). The same principle applies to any retried write operation: the server needs to recognize "I've already processed this request" and return the cached result.

Never retry non-idempotent operations without an idempotency mechanism. The retry might succeed twice.

## The `backoff` crate

Writing your own retry loop is fine for learning, but for production code the [`backoff`](https://crates.io/crates/backoff) crate (v0.4.0) handles edge cases you'd rather not think about - clock drift, elapsed time limits, and async runtime integration.

```toml
[dependencies]
backoff = { version = "0.4", features = ["tokio"] }
reqwest = { version = "0.12", features = ["json"] }
tokio = { version = "1", features = ["full"] }
```

```rust
use backoff::ExponentialBackoffBuilder;
use std::time::Duration;

async fn fetch_with_retry(url: &str) -> Result<String, backoff::Error<reqwest::Error>> {
    let backoff = ExponentialBackoffBuilder::new()
        .with_initial_interval(Duration::from_secs(1))
        .with_multiplier(2.0)
        .with_randomization_factor(0.5) // jitter: +/- 50%
        .with_max_interval(Duration::from_secs(30))
        .with_max_elapsed_time(Some(Duration::from_secs(120)))
        .build();

    backoff::future::retry(backoff, || async {
        let response = reqwest::get(url).await?.text().await?;
        Ok(response)
    })
    .await
}
```

The `with_randomization_factor(0.5)` means each delay is randomized within 50% of the calculated interval. A computed delay of 4 seconds becomes something between 2 and 6 seconds.

The `with_max_elapsed_time` is critical. Without it, an operation could retry for hours. Setting a total time budget means the retry loop will give up when `next_backoff()` returns `None` after the deadline passes.

### Permanent vs transient errors

Not all errors should be retried. A 404 is never going to succeed on retry. A 503 might. The `backoff` crate models this with its `Error` enum:

```rust
use backoff::Error;

async fn fetch_order(order_id: &str) -> Result<Order, backoff::Error<ApiError>> {
    let backoff = ExponentialBackoffBuilder::new()
        .with_max_elapsed_time(Some(Duration::from_secs(30)))
        .build();

    backoff::future::retry(backoff, || async {
        match api_client.get_order(order_id).await {
            Ok(order) => Ok(order),
            Err(e) if e.status() == Some(StatusCode::NOT_FOUND) => {
                // Don't retry 404s. The order doesn't exist.
                Err(Error::Permanent(e))
            }
            Err(e) if e.status() == Some(StatusCode::TOO_MANY_REQUESTS) => {
                // Rate limited - retry, but respect Retry-After if present
                Err(Error::retry_after(e, Duration::from_secs(5)))
            }
            Err(e) => {
                // Network error, 500, 502, 503 - transient, retry automatically
                Err(Error::transient(e))
            }
        }
    })
    .await
}
```

The `?` operator on errors inside the retry closure maps to `Error::Transient` by default, which is the right call for most I/O errors. You only need explicit `Error::Permanent` for cases where retrying is pointless. I talked about [REST API status codes and Retry-After headers](/blog/designing-restful-apis-practical-guidelines-beyond-the-theory) previously - a well-designed API tells you whether retrying makes sense through its response codes.

## `backon`: an alternative API

If you prefer a more fluent interface, [`backon`](https://crates.io/crates/backon) (v1.6.0) uses an extension-trait approach:

```rust
use backon::{ExponentialBuilder, Retryable};

async fn fetch_data() -> Result<String, reqwest::Error> {
    reqwest::get("https://api.example.com/data")
        .await?
        .text()
        .await
}

async fn fetch_with_retry() -> Result<String, reqwest::Error> {
    fetch_data
        .retry(ExponentialBuilder::default().with_jitter())
        .sleep(tokio::time::sleep)
        .when(|e| e.is_connect() || e.is_timeout())
        .notify(|err, dur| {
            tracing::warn!(error = %err, delay = ?dur, "retrying request");
        })
        .await
        .map_err(|e| e)
}
```

The `.when()` filter replaces the Permanent/Transient distinction - you tell it which errors are retryable. The `.notify()` callback is useful for metrics and logging. `backon` also supports `FibonacciBuilder` and `ConstantBuilder` if you need different backoff curves. The API is clean, but `backoff` has a larger ecosystem and more documentation. Pick whichever fits your codebase.

## Circuit breaker: when retries are the wrong tool

Retries operate at the **request level**. A circuit breaker operates at the **system level**. The difference matters.

When a downstream service is down, retrying every individual request means every caller adds load to a system that's already failing. Even with exponential backoff and jitter, you're still sending probes. Multiply that by thousands of callers and the recovery window shrinks because the failing service never gets a break.

A circuit breaker stops the bleeding. It tracks failure rates and, once a threshold is crossed, short-circuits all requests immediately without even attempting the call. After a cooldown period, it lets a single probe through to test if the service has recovered.

### Three states

```
    Success           Failure threshold
  +--------+         exceeded           +------+
  |        v                            |      v
  | CLOSED |  ------>  OPEN  -------->  | HALF |
  |  (ok)  |          (reject all)      | OPEN |
  +--------+                            +------+
      ^                                    |
      |              Probe succeeds        |
      +------------------------------------+
      |              Probe fails           |
      |         +---->  Back to OPEN       |
      |         |                          |
      |         +--------------------------+
```

- **Closed**: normal operation. Requests pass through. Failures are counted.
- **Open**: failures exceeded the threshold. All requests are immediately rejected with a fast failure. No network call is made. After a timeout, transition to half-open.
- **Half-open**: a single test request is allowed through. If it succeeds, back to closed. If it fails, back to open with a longer timeout (exponential backoff on the cooldown itself).

I introduced the `failsafe` crate briefly in the [monitoring post](/blog/monitoring-rust-applications-in-production). Let's look at a more complete implementation. Here's a circuit breaker built from scratch so you can see the mechanics:

```rust
use std::sync::atomic::{AtomicU32, AtomicU64, Ordering};
use std::sync::Mutex;
use std::time::{Duration, Instant};

#[derive(Debug, Clone, Copy, PartialEq)]
enum State {
    Closed,
    Open,
    HalfOpen,
}

pub struct CircuitBreaker {
    state: Mutex<State>,
    consecutive_failures: AtomicU32,
    failure_threshold: u32,
    last_failure_time: Mutex<Option<Instant>>,
    cooldown: Duration,
    half_open_permits: AtomicU32,
}

#[derive(Debug)]
pub enum CircuitError<E> {
    /// The underlying operation failed
    Inner(E),
    /// Circuit is open - call was rejected without attempting the operation
    Rejected,
}

impl CircuitBreaker {
    pub fn new(failure_threshold: u32, cooldown: Duration) -> Self {
        Self {
            state: Mutex::new(State::Closed),
            consecutive_failures: AtomicU32::new(0),
            failure_threshold,
            last_failure_time: Mutex::new(None),
            cooldown,
            half_open_permits: AtomicU32::new(0),
        }
    }

    pub async fn call<F, Fut, T, E>(&self, op: F) -> Result<T, CircuitError<E>>
    where
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = Result<T, E>>,
    {
        // Check if we should allow this call
        if !self.should_permit() {
            return Err(CircuitError::Rejected);
        }

        match op().await {
            Ok(v) => {
                self.record_success();
                Ok(v)
            }
            Err(e) => {
                self.record_failure();
                Err(CircuitError::Inner(e))
            }
        }
    }

    fn should_permit(&self) -> bool {
        let mut state = self.state.lock().unwrap();
        match *state {
            State::Closed => true,
            State::Open => {
                // Check if cooldown has elapsed
                let last = self.last_failure_time.lock().unwrap();
                if let Some(t) = *last {
                    if t.elapsed() >= self.cooldown {
                        *state = State::HalfOpen;
                        self.half_open_permits.store(1, Ordering::SeqCst);
                        // Allow exactly one probe
                        self.half_open_permits
                            .fetch_update(Ordering::SeqCst, Ordering::SeqCst, |v| {
                                if v > 0 { Some(v - 1) } else { None }
                            })
                            .is_ok()
                    } else {
                        false
                    }
                } else {
                    false
                }
            }
            State::HalfOpen => {
                // Only allow if we have permits left
                self.half_open_permits
                    .fetch_update(Ordering::SeqCst, Ordering::SeqCst, |v| {
                        if v > 0 { Some(v - 1) } else { None }
                    })
                    .is_ok()
            }
        }
    }

    fn record_success(&self) {
        self.consecutive_failures.store(0, Ordering::SeqCst);
        let mut state = self.state.lock().unwrap();
        *state = State::Closed;
    }

    fn record_failure(&self) {
        let failures = self.consecutive_failures.fetch_add(1, Ordering::SeqCst) + 1;
        *self.last_failure_time.lock().unwrap() = Some(Instant::now());
        if failures >= self.failure_threshold {
            let mut state = self.state.lock().unwrap();
            *state = State::Open;
        }
    }
}
```

Usage looks like this:

```rust
use std::time::Duration;

let cb = CircuitBreaker::new(5, Duration::from_secs(10));

// In your request handler:
match cb.call(|| fetch_from_payment_service(order_id)).await {
    Ok(payment) => {
        // Normal path
        HttpResponse::ok(payment)
    }
    Err(CircuitError::Rejected) => {
        // Circuit is open. Don't even try. Return a degraded response.
        tracing::warn!("payment service circuit open, returning cached data");
        HttpResponse::ok(cached_payment_status(order_id))
    }
    Err(CircuitError::Inner(e)) => {
        // The call was attempted and failed
        tracing::error!(error = %e, "payment service call failed");
        HttpResponse::internal_server_error()
    }
}
```

The key insight: `CircuitError::Rejected` is fast. No network call, no timeout, no wasted resources. When a dependency is down and you know it's down, failing immediately is better than waiting 30 seconds for a timeout on every request.

## Retry vs circuit breaker: when to use which

These aren't competing patterns - they compose.

**Retries** handle transient failures at the individual request level. A single dropped connection, a momentary 503, a network blip. The assumption is that the next attempt will probably work.

**Circuit breakers** handle sustained failures at the system level. The downstream service is genuinely down. Retrying won't help - it'll make things worse. The assumption is that the service needs time to recover.

In practice, you layer them:

```rust
// Pseudo-code showing composition
let cb = CircuitBreaker::new(5, Duration::from_secs(30));

// Each call gets retried up to 3 times with backoff.
// But if 5 consecutive calls fail (across any number of callers),
// the circuit opens and all callers get fast Rejected errors
// until the service recovers.

let result = cb.call(|| {
    retry_with_jitter(&RetryConfig { max_retries: 3, ..Default::default() }, || {
        fetch_from_service(url)
    })
}).await;
```

The retry handles the small stuff. The circuit breaker handles the big stuff. Without the circuit breaker, your retry logic generates 3x the load on a failing service (every caller retries 3 times). With the circuit breaker, the first few callers retry normally, the circuit opens after detecting a pattern, and everyone else gets a fast failure.

## Production checklist

After building retry and circuit breaker logic into a few services, some lessons become obvious:

**Always set a total timeout.** Exponential backoff with no maximum elapsed time can retry for hours. The `backoff` crate's `max_elapsed_time` handles this. For manual implementations, track elapsed time since the first attempt and bail when the budget is spent.

**Log every retry with structured fields.** Include the attempt number, delay, error message, and operation name. Without this, debugging retry storms is guesswork. If you're using the `tracing` crate (and you should be - see my [monitoring post](/blog/monitoring-rust-applications-in-production)), structured fields make retry events filterable and countable.

**Emit metrics.** At minimum: retry count per operation, circuit breaker state transitions, and rejected-by-circuit-breaker count. If your circuit breaker opens and you don't have a metric for it, you're flying blind.

**Respect `Retry-After` headers.** When an API responds with `429 Too Many Requests` and a `Retry-After` header, use that duration instead of your own backoff calculation. The server is telling you exactly how long to wait. The `backoff` crate supports this with `Error::retry_after(err, duration)`.

**Don't retry client errors.** A 400 Bad Request, 401 Unauthorized, or 404 Not Found will never succeed on retry. Only retry server errors (5xx) and network-level failures (timeouts, connection resets, DNS failures). Classify errors before deciding to retry.

**Consider retry budgets.** Instead of "each request retries up to 3 times," some systems use a shared budget: "across all requests, we allow 10% extra traffic for retries." Google's SRE book recommends this approach. When the retry budget is exhausted, new requests fail fast even before hitting the circuit breaker threshold. This prevents the pathological case where N clients each retrying M times turns your traffic into N*M requests.

**Test the failure path.** Unit-test your circuit breaker state transitions. Simulate a sequence of failures and verify the circuit opens. Simulate recovery and verify it closes. These state machines have subtle bugs (off-by-one on failure counts, race conditions on state transitions) that only show up under load.

## Wrapping up

The progression is: fixed delay -> exponential backoff -> backoff with jitter -> circuit breaker. Each layer addresses a failure mode that the previous one doesn't handle.

Fixed delay works for a single client. Exponential backoff prevents hammering. Jitter prevents thundering herds. Circuit breakers prevent cascading failures across services.

For most Rust services, adding `backoff = { version = "0.4", features = ["tokio"] }` to your `Cargo.toml` and wrapping external calls in `backoff::future::retry` with a properly configured `ExponentialBackoff` gets you 90% of the way there. Add a circuit breaker (via `failsafe` or your own implementation) for critical dependencies, and you have a resilient system that degrades gracefully instead of falling over.
