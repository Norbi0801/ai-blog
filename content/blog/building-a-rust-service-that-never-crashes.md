+++
title = "Building a Rust service that never crashes"
date = 2025-07-02
description = "Strategies for maximum uptime in production Rust services - from panic handling and circuit breakers to fuzzing and chaos testing."

[taxonomies]
tags = ["rust", "reliability", "production", "architecture"]
+++

Rust eliminates use-after-free, data races, and null pointer dereferences at compile time. That's a massive head start on reliability. But "memory safe" doesn't mean "never crashes." Your service can still panic on an `unwrap()`, deadlock on a poisoned mutex, or fall over because a downstream database went away for 30 seconds.

Building a service that *actually* stays up in production requires deliberate work across multiple layers: panic containment, structured error handling, graceful degradation, dependency isolation, and proactive fault discovery. This post covers every layer, with code you can drop into a real service.

<!-- more -->

## The anatomy of a production crash

Before hardening anything, it helps to know what actually kills Rust services in production. In my experience, it breaks down roughly like this:

1. **Panics from `.unwrap()` / `.expect()` on `None` or `Err`** - the #1 killer by far
2. **External dependency failures** - database goes down, upstream API times out, DNS hiccups
3. **Resource exhaustion** - connection pool drained, file descriptors maxed, OOM
4. **Deadlocks and poisoned mutexes** - especially with `std::sync::Mutex` across async boundaries
5. **Misconfigurations** - wrong env var, bad connection string, expired TLS cert

Notice what's *not* on this list: segfaults, buffer overflows, use-after-free. Rust already handled those. Everything above is about logic errors and operational failures. Let's fix them.

## Panic handling: catch_unwind and the abort tradeoff

When a Rust program panics, the default behavior is *unwinding* - walking back up the call stack, running destructors, cleaning up resources. You can catch this with [`std::panic::catch_unwind`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html):

```rust
use std::panic;

fn handle_request(req: Request) -> Response {
    let result = panic::catch_unwind(panic::AssertUnwindSafe(|| {
        // your actual handler logic
        process_request(req)
    }));

    match result {
        Ok(response) => response,
        Err(panic_info) => {
            tracing::error!(?panic_info, "handler panicked");
            Response::internal_server_error()
        }
    }
}
```

This keeps your server alive even when a single request handler panics. Axum and actix-web both do something similar internally - they catch panics at the request boundary so one bad request doesn't bring down the whole process. But relying on this as your primary defense is a mistake. `catch_unwind` is a safety net, not a strategy.

**Important limitations:**

- It only catches *unwinding* panics. If you compile with `panic = "abort"`, panics terminate the process immediately - there's nothing to catch.
- It doesn't catch panics in spawned threads or tasks unless you wrap those too.
- A panic during `Drop` while already unwinding triggers a double panic, which aborts regardless.
- The caught value is `Box<dyn Any + Send>`, which is awkward to extract useful information from.

### The panic hook

Before `catch_unwind` even runs, the global [panic hook](https://doc.rust-lang.org/std/panic/fn.set_hook.html) fires. This is where you log the panic with a proper backtrace:

```rust
use std::panic;
use std::backtrace::Backtrace;

fn install_panic_hook() {
    panic::set_hook(Box::new(|info| {
        let backtrace = Backtrace::force_capture();
        let location = info.location().map(|l| format!("{}:{}:{}", l.file(), l.line(), l.column()));
        let payload = if let Some(s) = info.payload().downcast_ref::<&str>() {
            s.to_string()
        } else if let Some(s) = info.payload().downcast_ref::<String>() {
            s.clone()
        } else {
            "unknown payload".to_string()
        };

        tracing::error!(
            panic.payload = %payload,
            panic.location = ?location,
            panic.backtrace = %backtrace,
            "PANIC captured"
        );
    }));
}
```

Call this once in `main()`, before anything else. Now every panic - caught or not - gets proper structured logging. If you covered tracing setup from my [debugging post](/blog/debugging-rust-beyond-println), this hooks right into your existing telemetry pipeline.

### panic=abort vs panic=unwind

In `Cargo.toml`, you can choose the panic strategy:

```toml
[profile.release]
panic = "abort"    # terminate immediately on panic
# panic = "unwind" # default - walk the stack, run destructors
```

The tradeoffs:

| | `unwind` (default) | `abort` |
|---|---|---|
| Binary size | Larger (unwinding tables) | ~5-10% smaller |
| Runtime overhead | Minimal when not panicking | Zero |
| `catch_unwind` works? | Yes | No |
| Destructors run on panic? | Yes | No |
| Recovery possible? | Yes | No |

For long-running services where you want per-request panic isolation, keep `panic = "unwind"`. The binary size cost is worth it. Use `panic = "abort"` for CLIs, embedded systems, or WASM where panics are truly unrecoverable and you want the smallest binary.

The real insight: `panic = "abort"` *forces* you to never panic, because any panic kills the process. Some teams use this as a forcing function during development - it makes every `unwrap()` existentially dangerous, which tends to clean up error handling fast.

## Structured error handling: banning unwrap from production

The single most impactful thing you can do for reliability is eliminate `unwrap()` and `expect()` from production code paths. Every `unwrap()` is a potential panic. Every panic is a potential crash.

### The thiserror + anyhow split

Use [`thiserror`](https://crates.io/crates/thiserror) for your domain error types and [`anyhow`](https://crates.io/crates/anyhow) at the application boundary:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ServiceError {
    #[error("user {user_id} not found")]
    UserNotFound { user_id: String },

    #[error("database query failed")]
    Database(#[from] sqlx::Error),

    #[error("upstream API returned {status}")]
    Upstream { status: u16, body: String },

    #[error("request timed out after {elapsed:?}")]
    Timeout { elapsed: std::time::Duration },
}
```

This gives callers matchable variants. They can decide: retry on `Timeout`, return 404 on `UserNotFound`, alert on `Database`. Compare that to `anyhow::Error` which is great for propagating with `.context()` but opaque to match against:

```rust
use anyhow::Context;

async fn fetch_user_profile(id: &str) -> anyhow::Result<Profile> {
    let user = db.fetch_user(id)
        .await
        .context("failed to fetch user from database")?;

    let preferences = cache.get_preferences(id)
        .await
        .context("failed to load preferences from cache")?;

    Ok(Profile { user, preferences })
}
```

The `.context()` call wraps the underlying error with a human-readable message, building a chain: `failed to load preferences from cache: connection refused`. When this hits your error handler, you get the full story.

### Enforcing it with Clippy

Add these to your `clippy.toml` or CI config:

```toml
# clippy.toml
disallowed-methods = [
    { path = "core::option::Option::unwrap", reason = "use .context()? or handle the None case" },
    { path = "core::result::Result::unwrap", reason = "use .context()? or match the error" },
]
```

Or in your `Cargo.toml` / `lib.rs`:

```rust
#![deny(clippy::unwrap_used)]
#![deny(clippy::expect_used)]
```

This turns every `unwrap()` into a compile error. Aggressive? Yes. But it catches the #1 source of production panics at build time. You can use `#[allow(clippy::unwrap_used)]` on specific lines where you've genuinely proven the value is always `Some` or `Ok` - but you have to be explicit about it.

## Graceful shutdown

When your service receives SIGTERM (Kubernetes pod termination, systemd stop, `docker stop`), it needs to:

1. Stop accepting new connections
2. Finish in-flight requests
3. Flush buffers, close database connections
4. Exit cleanly

Here's the pattern with Axum and tokio:

```rust
use axum::Router;
use tokio::net::TcpListener;
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => tracing::info!("received Ctrl+C"),
        _ = terminate => tracing::info!("received SIGTERM"),
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new(); // your routes here
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    tracing::info!("listening on 0.0.0.0:3000");

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    tracing::info!("server shut down cleanly");
}
```

Axum's `with_graceful_shutdown` stops accepting new connections when the signal fires, then waits for in-flight requests to complete. But you need to set a deadline - Kubernetes gives you 30 seconds by default (`terminationGracePeriodSeconds`), so your handlers need to finish within that window or get killed with SIGKILL.

For background tasks that need their own cleanup:

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();
let worker_token = token.clone();

let worker = tokio::spawn(async move {
    loop {
        tokio::select! {
            _ = worker_token.cancelled() => {
                tracing::info!("worker shutting down, flushing buffer...");
                flush_buffer().await;
                break;
            }
            msg = queue.recv() => {
                if let Some(msg) = msg {
                    process(msg).await;
                }
            }
        }
    }
});

// When shutdown signal arrives:
token.cancel();
let _ = tokio::time::timeout(Duration::from_secs(10), worker).await;
```

The `CancellationToken` from [`tokio-util`](https://docs.rs/tokio-util) is the cleanest way to propagate shutdown intent across multiple tasks.

## Circuit breakers for external dependencies

Your Rust code can be perfect and your service still goes down because a database, cache, or upstream API failed. The circuit breaker pattern prevents cascading failures by fast-failing when a dependency is unhealthy.

The state machine is simple: **Closed** (normal, requests pass through) -> **Open** (too many failures, requests rejected immediately) -> **Half-Open** (test one request to see if the dependency recovered).

Here's a minimal implementation:

```rust
use std::sync::atomic::{AtomicU32, AtomicU64, Ordering};
use std::time::{Duration, SystemTime, UNIX_EPOCH};

pub struct CircuitBreaker {
    failure_count: AtomicU32,
    failure_threshold: u32,
    last_failure_time: AtomicU64,
    recovery_timeout: Duration,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum CircuitState {
    Closed,
    Open,
    HalfOpen,
}

impl CircuitBreaker {
    pub fn new(failure_threshold: u32, recovery_timeout: Duration) -> Self {
        Self {
            failure_count: AtomicU32::new(0),
            failure_threshold,
            last_failure_time: AtomicU64::new(0),
            recovery_timeout,
        }
    }

    pub fn state(&self) -> CircuitState {
        let failures = self.failure_count.load(Ordering::Relaxed);
        if failures < self.failure_threshold {
            return CircuitState::Closed;
        }

        let last_failure = self.last_failure_time.load(Ordering::Relaxed);
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();

        if now - last_failure >= self.recovery_timeout.as_secs() {
            CircuitState::HalfOpen
        } else {
            CircuitState::Open
        }
    }

    pub fn record_success(&self) {
        self.failure_count.store(0, Ordering::Relaxed);
    }

    pub fn record_failure(&self) {
        self.failure_count.fetch_add(1, Ordering::Relaxed);
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();
        self.last_failure_time.store(now, Ordering::Relaxed);
    }
}
```

Use it to wrap dependency calls:

```rust
async fn fetch_from_upstream(breaker: &CircuitBreaker, client: &reqwest::Client) -> Result<Data, ServiceError> {
    match breaker.state() {
        CircuitState::Open => {
            return Err(ServiceError::CircuitOpen { service: "upstream-api" });
        }
        CircuitState::Closed | CircuitState::HalfOpen => {}
    }

    match client.get("https://api.example.com/data").send().await {
        Ok(resp) if resp.status().is_success() => {
            breaker.record_success();
            let data = resp.json().await.map_err(|e| ServiceError::Deserialization(e.to_string()))?;
            Ok(data)
        }
        Ok(resp) => {
            breaker.record_failure();
            Err(ServiceError::Upstream {
                status: resp.status().as_u16(),
                body: resp.text().await.unwrap_or_default(),
            })
        }
        Err(e) => {
            breaker.record_failure();
            Err(ServiceError::Network(e.to_string()))
        }
    }
}
```

For production, consider [`failsafe-rs`](https://github.com/dmexe/failsafe-rs) or [`tower-circuitbreaker`](https://lib.rs/crates/tower-circuitbreaker) which integrates with the Tower middleware stack. If you're already using Axum (which is Tower-native), `tower-circuitbreaker` slots in naturally as a layer.

## Connection pool recovery

Connection pools are another failure point. The database restarts, connections go stale, and suddenly every request gets a "connection reset" error. Both [`deadpool`](https://crates.io/crates/deadpool) and [`bb8`](https://crates.io/crates/bb8) handle this, but you need to configure them correctly.

With `deadpool-postgres`:

```rust
use deadpool_postgres::{Config, ManagerConfig, RecyclingMethod, Runtime};

let mut cfg = Config::new();
cfg.host = Some("localhost".to_string());
cfg.dbname = Some("myapp".to_string());
cfg.manager = Some(ManagerConfig {
    recycling_method: RecyclingMethod::Verified,
});

let pool = cfg.create_pool(Some(Runtime::Tokio1), tokio_postgres::NoTls)?;
```

`RecyclingMethod::Verified` runs a test query (`SELECT 1`) before handing a connection to your code. This catches stale connections before they cause request failures. The cost is one extra round trip per checkout, but for most services that's negligible compared to the query itself.

With `bb8`:

```rust
use bb8::Pool;
use bb8_postgres::PostgresConnectionManager;

let manager = PostgresConnectionManager::new_from_stringlike(
    "host=localhost dbname=myapp",
    tokio_postgres::NoTls,
)?;

let pool = Pool::builder()
    .max_size(20)
    .min_idle(Some(5))           // keep 5 warm connections ready
    .connection_timeout(Duration::from_secs(5))
    .idle_timeout(Some(Duration::from_secs(600)))
    .max_lifetime(Some(Duration::from_secs(1800)))
    .test_on_check_out(true)     // validate before use
    .build(manager)
    .await?;
```

Key settings that prevent pool-related crashes:

- **`connection_timeout`**: Don't wait forever for a connection. 5 seconds is generous.
- **`max_lifetime`**: Rotate connections before the database server decides to kill them (most databases have an `idle_in_transaction_session_timeout` or similar).
- **`min_idle`**: Keep warm connections so cold-start latency doesn't cascade into timeouts.
- **`test_on_check_out`**: Validate connections. This is the cheapest insurance against stale connections.

## Watchdog patterns and health-based restarts

Sometimes the best recovery strategy is to let the process die and be restarted. But you want *controlled* death, not silent hangs. A hung service that holds open connections but processes nothing is worse than a crash - at least a crash gets restarted.

### Kubernetes health checks

Separate your health endpoints:

```rust
use axum::{routing::get, Json, Router};
use serde::Serialize;
use std::sync::Arc;

#[derive(Serialize)]
struct HealthResponse {
    status: String,
    db_connected: bool,
    uptime_seconds: u64,
}

async fn liveness() -> &'static str {
    // Am I alive? Can I respond to requests at all?
    "ok"
}

async fn readiness(state: Arc<AppState>) -> Json<HealthResponse> {
    let db_ok = state.db_pool.get().await.is_ok();
    let uptime = state.started_at.elapsed().as_secs();

    Json(HealthResponse {
        status: if db_ok { "ready".into() } else { "degraded".into() },
        db_connected: db_ok,
        uptime_seconds: uptime,
    })
}

fn health_routes() -> Router<Arc<AppState>> {
    Router::new()
        .route("/health/live", get(liveness))
        .route("/health/ready", get(readiness))
}
```

- **Liveness** (`/health/live`): Returns 200 if the process is running. If this fails, Kubernetes kills and restarts the pod. Keep this trivial - no dependency checks.
- **Readiness** (`/health/ready`): Returns 200 if the service can handle traffic. If the database is down, return 503 - Kubernetes removes the pod from the load balancer but doesn't kill it, giving the database time to recover.

### Internal watchdog

For detecting deadlocks and silent hangs, run a watchdog task that monitors your worker tasks:

```rust
use tokio::sync::watch;
use std::time::{Duration, Instant};

async fn watchdog(mut heartbeat_rx: watch::Receiver<Instant>, timeout: Duration) {
    loop {
        tokio::time::sleep(timeout / 2).await;

        let last_beat = *heartbeat_rx.borrow();
        if last_beat.elapsed() > timeout {
            tracing::error!(
                elapsed = ?last_beat.elapsed(),
                timeout = ?timeout,
                "watchdog: worker appears hung, initiating shutdown"
            );
            std::process::exit(1); // let the orchestrator restart us
        }
    }
}

// In your worker task:
async fn worker_loop(heartbeat_tx: watch::Sender<Instant>) {
    loop {
        let _ = heartbeat_tx.send(Instant::now());
        // do actual work
        process_next_job().await;
    }
}
```

The worker sends heartbeats via a `watch` channel. If the watchdog doesn't see a fresh heartbeat within the timeout, the worker is probably deadlocked or stuck on a blocking operation. Exiting with code 1 lets systemd or Kubernetes restart the process cleanly.

## Fuzzing: finding panics before production does

[`cargo-fuzz`](https://github.com/rust-fuzz/cargo-fuzz) (currently at v0.13.1) throws random inputs at your code to find panics, overflows, and logic errors. It uses [libFuzzer](https://llvm.org/docs/LibFuzzer.html) under the hood with coverage-guided mutation - it doesn't just generate random bytes, it tracks which code paths each input exercises and mutates toward uncovered branches.

Set it up:

```bash
cargo install cargo-fuzz
cargo fuzz init
```

Write a fuzz target in `fuzz/fuzz_targets/parse_input.rs`:

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;
use my_service::parser;

fuzz_target!(|data: &[u8]| {
    if let Ok(input) = std::str::from_utf8(data) {
        // This should never panic, regardless of input
        let _ = parser::parse_request(input);
    }
});
```

Run it:

```bash
cargo fuzz run parse_input -- -max_total_time=300  # 5 minutes
```

If it finds a crashing input, it saves it in `fuzz/artifacts/`. You can reproduce with:

```bash
cargo fuzz run parse_input fuzz/artifacts/parse_input/crash-abc123
```

For structured input fuzzing, use [`arbitrary`](https://crates.io/crates/arbitrary) to generate valid Rust types instead of raw bytes:

```rust
use arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;

#[derive(Debug, Arbitrary)]
struct FuzzInput {
    name: String,
    count: u32,
    enabled: bool,
}

fuzz_target!(|input: FuzzInput| {
    let _ = my_service::handle_config(input.name, input.count, input.enabled);
});
```

This generates structurally valid inputs that exercise your business logic, not just your parser.

### Integrating fuzzing into CI

Run fuzzing as a time-boxed CI step. You won't find everything in 60 seconds, but you'll catch regressions:

```yaml
# .github/workflows/fuzz.yml
name: Fuzz
on:
  schedule:
    - cron: '0 3 * * *'  # nightly at 3 AM
jobs:
  fuzz:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
      - run: cargo install cargo-fuzz
      - run: cargo fuzz run parse_input -- -max_total_time=120
```

## Chaos testing

Fuzzing finds code-level bugs. Chaos testing finds operational bugs - what happens when the network flakes, the database is slow, or a disk fills up.

### Simulating failures

Use [`toxiproxy`](https://github.com/Shopify/toxiproxy) to inject latency, connection resets, and bandwidth limits between your service and its dependencies:

```bash
# Add a proxy in front of your database
toxiproxy-cli create postgres_proxy -l 0.0.0.0:15432 -u localhost:5432

# Add 500ms latency
toxiproxy-cli toxic add postgres_proxy -t latency -a latency=500

# Drop 30% of connections
toxiproxy-cli toxic add postgres_proxy -t reset_peer -a reset=0.3
```

Point your service at `localhost:15432` instead of `5432` during testing. Then verify:

- Does it time out gracefully or hang forever?
- Does the circuit breaker trip?
- Does it recover when the toxic is removed?
- Do your health checks reflect the degradation?

### Fault injection in tests

For unit/integration tests, you can inject failures at the Rust level:

```rust
#[cfg(test)]
mod tests {
    use std::sync::atomic::{AtomicU32, Ordering};

    struct FlakyClient {
        call_count: AtomicU32,
        fail_every_n: u32,
    }

    impl FlakyClient {
        fn new(fail_every_n: u32) -> Self {
            Self {
                call_count: AtomicU32::new(0),
                fail_every_n,
            }
        }

        async fn request(&self) -> Result<String, std::io::Error> {
            let count = self.call_count.fetch_add(1, Ordering::Relaxed);
            if count % self.fail_every_n == 0 {
                return Err(std::io::Error::new(
                    std::io::ErrorKind::ConnectionReset,
                    "simulated failure",
                ));
            }
            Ok("success".to_string())
        }
    }

    #[tokio::test]
    async fn service_survives_flaky_dependency() {
        let client = FlakyClient::new(3); // fails every 3rd call
        let breaker = CircuitBreaker::new(5, Duration::from_secs(10));

        for _ in 0..20 {
            let _ = fetch_with_breaker(&breaker, &client).await;
        }

        // Service should still be functional
        assert_ne!(breaker.state(), CircuitState::Open);
    }
}
```

This pattern is simpler than full chaos engineering but catches the most common integration failures. Pair it with the [load testing](/blog/load-testing-your-rust-api---tools-and-methodology/) techniques for stress scenarios.

## Production hardening checklist

Here's what I check before any Rust service goes to production. Not all items apply to every service, but skipping any of them should be a conscious decision, not an oversight.

**Error handling:**
- [ ] No `unwrap()` or `expect()` on fallible paths (enforced by `clippy::unwrap_used`)
- [ ] Domain errors use `thiserror` with matchable variants
- [ ] All errors include context (`.context()` or structured fields)
- [ ] Panic hook installed with backtrace logging

**Resilience:**
- [ ] Circuit breakers on all external HTTP/gRPC calls
- [ ] Connection pool health checks enabled (`test_on_check_out` / `RecyclingMethod::Verified`)
- [ ] Timeouts on every external call (no unbounded `.await`)
- [ ] Graceful shutdown handles SIGTERM with connection draining

**Observability:**
- [ ] Structured logging with `tracing` (not `println!` or `log`)
- [ ] Request tracing with correlation IDs
- [ ] Health endpoints: `/health/live` (trivial) and `/health/ready` (dependency-aware)
- [ ] Metrics on error rates, latency percentiles, pool usage

**Testing:**
- [ ] Fuzz targets for all parsing and deserialization code
- [ ] Integration tests with simulated dependency failures
- [ ] Load tests that run longer than your connection pool's `max_lifetime`

**Deployment:**
- [ ] `panic = "unwind"` in release profile (or `"abort"` with full confidence in error handling)
- [ ] `RUST_BACKTRACE=1` in production environment
- [ ] Resource limits set (memory, file descriptors, connections)
- [ ] Liveness and readiness probes configured in orchestrator

None of these are exotic. They're the boring operational stuff that separates a service that runs fine on your laptop from one that survives three years in production without someone getting paged at 3 AM.

## Closing thought

Rust gives you a foundation that other languages can't match - no null pointers, no data races, no use-after-free. But that foundation is just the floor. The ceiling is set by how well you handle the failures that *are* possible: bad inputs, flaky networks, exhausted resources, configuration mistakes. The techniques in this post aren't about writing clever code. They're about being paranoid in exactly the right places, so the service can be boring in production. And boring production services are the best kind.
