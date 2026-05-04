+++
title = "Monitoring Rust applications in production"
date = 2025-02-09
description = "A practical guide to tracing, metrics, structured logs, and health checks for Rust services using OpenTelemetry, Prometheus, and the tracing ecosystem."

[taxonomies]
tags = ["rust", "observability", "devops", "production"]
+++

Your Rust service compiles, passes tests, survives load testing, and deploys cleanly. Then at 3 AM your on-call phone rings because response times tripled and nobody knows why. The binary is fast, yes. But fast does not mean observable.

Monitoring is what bridges "it works on my machine" and "it works in production, and I can prove it." This post covers the full stack: structured logs, distributed tracing, Prometheus metrics, health endpoints, graceful degradation, and how to choose a monitoring backend. All with Rust code that compiles.

<!-- more -->

## The three pillars (and why you need all of them)

Observability rests on three signals:

- **Logs** - discrete events with context. "User 4821 hit a 404 on /api/orders/xyz."
- **Metrics** - numeric aggregates over time. "p99 latency is 240ms. Error rate is 0.3%."
- **Traces** - request-scoped timelines across services. "This request spent 12ms in auth, 180ms in the database, 4ms serializing."

Logs tell you *what happened*. Metrics tell you *how things are trending*. Traces tell you *where the time went*. Skip any one of them and you're flying partially blind.

## Structured JSON logs with `tracing`

If you've read my [debugging post](/blog/debugging-rust-beyond-println), you already know `tracing` replaces `println!` debugging with structured spans and events. In production, you want those events as machine-parseable JSON, not pretty terminal output.

```toml
[dependencies]
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["json", "env-filter"] }
```

```rust
use tracing_subscriber::{fmt, EnvFilter};

fn init_logging() {
    tracing_subscriber::fmt()
        .json()
        .with_env_filter(EnvFilter::from_default_env())
        .with_target(true)
        .with_file(true)
        .with_line_number(true)
        .with_thread_ids(true)
        .flatten_event(true)
        .init();
}
```

This gives you newline-delimited JSON. Every log line becomes a structured object:

```json
{
  "timestamp": "2026-04-14T08:12:33.441Z",
  "level": "INFO",
  "message": "request completed",
  "target": "api::handlers",
  "file": "src/handlers/orders.rs",
  "line": 47,
  "http.method": "GET",
  "http.path": "/api/orders/123",
  "http.status": 200,
  "latency_ms": 14
}
```

The `flatten_event(true)` call is important. Without it, your fields get nested under a `fields` key, which makes querying in Loki or Datadog annoying. With flattening, every structured field lands at the root level where log query languages expect them.

### What to log (and what not to)

Log at boundaries: incoming requests, outgoing calls (DB, HTTP, gRPC), errors, and business events that matter (order created, payment failed). Do not log inside hot loops or on every iteration of a tight computation. Logging has cost - even with `tracing`'s compile-time filtering, the formatting and I/O still happen for enabled levels.

Use `RUST_LOG=warn,my_service=info` in production. Global warn level keeps noisy dependencies quiet while your own code logs at info.

Never log secrets, tokens, or PII. It sounds obvious, but structured logging makes it easy to accidentally attach a full request body that contains an API key. Use a wrapper type that redacts on Display:

```rust
pub struct Redacted<T>(pub T);

impl<T> std::fmt::Display for Redacted<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "[REDACTED]")
    }
}
```

## Distributed tracing with OpenTelemetry

Structured logs work great for a single service. Once you have two services talking to each other, you need traces - a shared timeline that follows a request across network boundaries.

The `tracing` crate handles in-process span trees. OpenTelemetry exports those spans to a backend (Jaeger, Tempo, Datadog) where you can visualize the full request waterfall.

```toml
[dependencies]
opentelemetry = "0.31.0"
opentelemetry_sdk = "0.31.0"
opentelemetry-otlp = { version = "0.31.1", features = ["grpc-tonic"] }
tracing-opentelemetry = "0.32.1"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
```

Note the version alignment: `tracing-opentelemetry` 0.32.x pairs with `opentelemetry` 0.31.x. They've been offset by one since version 0.26. Get this wrong and you'll hit confusing trait bound errors.

```rust
use opentelemetry::global;
use opentelemetry::trace::TracerProvider;
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_sdk::Resource;
use opentelemetry_otlp::SpanExporter;
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

fn init_telemetry() -> SdkTracerProvider {
    let exporter = SpanExporter::builder()
        .with_tonic()
        .build()
        .expect("failed to create OTLP exporter");

    let provider = SdkTracerProvider::builder()
        .with_resource(
            Resource::builder()
                .with_service_name("my-api")
                .build(),
        )
        .with_batch_exporter(exporter)
        .build();

    let tracer = provider.tracer("my-api");

    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer().json().flatten_event(true))
        .with(OpenTelemetryLayer::new(tracer))
        .init();

    global::set_tracer_provider(provider.clone());
    provider
}
```

A key detail: since OpenTelemetry 0.28, `BatchSpanProcessor` creates its own background thread. No more passing in a tokio runtime. This is cleaner but means you should call `provider.shutdown()` explicitly before your process exits, or you'll lose buffered spans.

```rust
#[tokio::main]
async fn main() {
    let provider = init_telemetry();

    // ... run your app ...

    // Flush remaining spans before exit
    if let Err(e) = provider.shutdown() {
        eprintln!("failed to shutdown tracer provider: {e}");
    }
}
```

### How tracing-opentelemetry bridges the two worlds

Every `tracing` span becomes an OpenTelemetry span. The `#[instrument]` macro on your functions generates spans automatically. Fields you attach via `tracing::info!(user_id = %id, "processing request")` become span attributes in your trace backend.

Under the hood, `tracing`'s subscriber registry uses a [lock-free sharded slab](https://docs.rs/sharded-slab) for span storage - the same data structure gets reused for pooling. When a span closes, its slab slot is cleared in-place and recycled. This matters because in a high-throughput service, you're creating and destroying thousands of spans per second. Allocation pressure from span storage would show up in your latency percentiles if the implementation weren't careful about it.

## Prometheus metrics with the `metrics` crate

Traces are great for debugging individual requests. Metrics are what you stare at on dashboards. The [`metrics`](https://crates.io/crates/metrics) crate (v0.24.3) provides a lightweight facade - similar to how `log` works for logging - with a Prometheus exporter.

```toml
[dependencies]
metrics = "0.24.3"
metrics-exporter-prometheus = "0.18.1"
metrics-process = "2.3.1"
```

```rust
use metrics::{counter, gauge, histogram};
use metrics_exporter_prometheus::PrometheusBuilder;
use std::time::Instant;

fn init_metrics() -> metrics_exporter_prometheus::PrometheusHandle {
    let handle = PrometheusBuilder::new()
        .install_recorder()
        .expect("failed to install metrics recorder");

    // Expose process-level metrics (CPU, memory, open FDs)
    metrics_process::Collector::default()
        .describe()
        .collect();

    handle
}
```

The `PrometheusHandle` gives you a `render()` method that returns the Prometheus text format. Wire it to an endpoint:

```rust
use axum::{routing::get, Router, response::IntoResponse};

async fn metrics_endpoint(
    State(handle): State<metrics_exporter_prometheus::PrometheusHandle>,
) -> impl IntoResponse {
    handle.render()
}

// In your router:
// .route("/metrics", get(metrics_endpoint))
```

Now instrument your code at the boundaries:

```rust
use metrics::{counter, histogram};
use std::time::Instant;

pub async fn handle_request(method: &str, path: &str) -> Response {
    let start = Instant::now();

    let response = process_request().await;

    let status = response.status().as_u16().to_string();
    let duration = start.elapsed().as_secs_f64();

    counter!(
        "http_requests_total",
        "method" => method.to_string(),
        "path" => path.to_string(),
        "status" => status
    )
    .increment(1);

    histogram!(
        "http_request_duration_seconds",
        "method" => method.to_string(),
        "path" => path.to_string(),
    )
    .record(duration);

    response
}
```

In practice, you'd put this in middleware rather than every handler. But the pattern is the same: count requests, record durations, label with method/path/status.

### What metrics to track

This is where RED and USE come in.

**RED** (Rate, Errors, Duration) - coined by Tom Wilkie - measures services from the caller's perspective:

- **Rate**: requests per second (`http_requests_total`)
- **Errors**: failed requests per second (`http_requests_total` where status is 5xx)
- **Duration**: response time distribution (`http_request_duration_seconds` histogram)

**USE** (Utilization, Saturation, Errors) - coined by Brendan Gregg - measures infrastructure resources:

- **Utilization**: how busy is the resource (CPU percentage, connection pool usage ratio)
- **Saturation**: queued work that can't be processed yet (thread pool queue depth, backpressure signals)
- **Errors**: resource-level failures (disk I/O errors, OOM kills)

RED tells you *something is wrong*. USE tells you *why*. Your HTTP error rate spikes (RED) because your database connection pool is fully saturated (USE). You need both.

If you went through my [load testing post](/blog/load-testing-your-rust-api---tools-and-methodology/), you already measured latency percentiles and throughput under synthetic load. Production metrics are the same measurements, running continuously, on real traffic.

Concrete metrics to expose from a typical Rust API:

```rust
// RED metrics
counter!("http_requests_total", "method" => m, "path" => p, "status" => s);
histogram!("http_request_duration_seconds", "method" => m, "path" => p);

// USE metrics for your connection pool
gauge!("db_pool_connections_active").set(pool.size() as f64);
gauge!("db_pool_connections_idle").set(pool.num_idle() as f64);
gauge!("db_pool_connections_max").set(pool.max_size() as f64);

// USE metrics for your async runtime
gauge!("tokio_threads_alive").set(runtime_metrics.num_alive_tasks() as f64);
gauge!("tokio_blocking_queue_depth").set(runtime_metrics.blocking_queue_depth() as f64);

// Business metrics
counter!("orders_created_total");
counter!("payments_failed_total", "reason" => reason);
histogram!("order_processing_duration_seconds");
```

Business metrics are often more useful than infrastructure metrics for understanding impact. "Orders per minute dropped 40%" is more actionable than "CPU is at 80%."

## Health endpoints

A health endpoint isn't just "return 200." In a Kubernetes environment, you need three distinct probes. I covered axum routing patterns in the [webhook receiver post](/blog/building-a-webhook-receiver-in-rust), and health endpoints follow the same structure.

```rust
use axum::{extract::State, http::StatusCode, routing::get, Json, Router};
use serde::Serialize;

#[derive(Serialize)]
struct HealthResponse {
    status: &'static str,
    version: &'static str,
    uptime_seconds: u64,
}

/// Liveness: is the process alive and not deadlocked?
/// Never check external dependencies here.
async fn liveness() -> Json<HealthResponse> {
    Json(HealthResponse {
        status: "ok",
        version: env!("CARGO_PKG_VERSION"),
        uptime_seconds: get_uptime_seconds(),
    })
}

/// Readiness: can this instance serve traffic right now?
/// Check database, cache, essential downstream services.
async fn readiness(State(state): State<AppState>) -> StatusCode {
    let db_ok = sqlx::query("SELECT 1")
        .execute(&state.db_pool)
        .await
        .is_ok();

    let cache_ok = state.redis.ping().await.is_ok();

    if db_ok && cache_ok {
        StatusCode::OK
    } else {
        StatusCode::SERVICE_UNAVAILABLE
    }
}

/// Startup: has the service finished initializing?
/// Migrations done, caches warmed, etc.
async fn startup(State(state): State<AppState>) -> StatusCode {
    if state.initialized.load(std::sync::atomic::Ordering::Relaxed) {
        StatusCode::OK
    } else {
        StatusCode::SERVICE_UNAVAILABLE
    }
}

fn health_routes() -> Router<AppState> {
    Router::new()
        .route("/health/live", get(liveness))
        .route("/health/ready", get(readiness))
        .route("/health/startup", get(startup))
}
```

The critical rule: **never check dependencies in the liveness probe**. If your database goes down and your liveness check fails, Kubernetes restarts your pod. The pod comes back up, database is still down, liveness fails again. You've turned a database outage into a cascading restart loop. Liveness should only fail if the process itself is broken - deadlocked, panicked, stuck in an infinite loop.

Readiness is where dependency checks belong. When readiness fails, Kubernetes removes the pod from the service load balancer but keeps it running. The pod sits there, retrying its connections, and rejoins automatically when the dependency recovers.

## Graceful degradation

Not every failure needs to become a 500. Some failures can be handled by falling back to cached data, skipping optional features, or circuit-breaking away from a failing dependency.

The [`failsafe`](https://crates.io/crates/failsafe) crate (v1.3.0) implements the circuit breaker pattern:

```rust
use failsafe::{Config, CircuitBreaker, Error};
use std::time::Duration;

let circuit_breaker = Config::new()
    .failure_policy(
        failsafe::failure_policy::consecutive_failures(5,
            failsafe::backoff::exponential(
                Duration::from_secs(1),
                Duration::from_secs(30),
            )
        )
    )
    .build();

async fn get_recommendations(
    cb: &failsafe::StateMachine<impl failsafe::Instrument>,
    cache: &Cache,
    user_id: &str,
) -> Vec<Recommendation> {
    match cb.call(|| fetch_recommendations_from_ml_service(user_id)).await {
        Ok(recs) => recs,
        Err(Error::Rejected) => {
            // Circuit is open - ML service has been failing.
            // Serve cached recommendations instead.
            tracing::warn!(user_id, "circuit open, serving cached recommendations");
            counter!("recommendations_fallback_total").increment(1);
            cache.get_cached_recommendations(user_id)
                .unwrap_or_default()
        }
        Err(Error::Inner(e)) => {
            tracing::error!(user_id, error = %e, "recommendation fetch failed");
            counter!("recommendations_error_total").increment(1);
            Vec::new()
        }
    }
}
```

The circuit breaker has three states: **closed** (requests flow through), **open** (requests are rejected immediately - fast fail), and **half-open** (a single probe request is allowed through to test recovery). After 5 consecutive failures, the circuit opens for 1 second, then doubles the wait each time up to 30 seconds.

Notice the counter increment on fallback. This is essential. If your circuit breaker silently swallows failures and serves stale data forever, you'll never know something is broken. Metric + alert on fallback rate.

If you're using [Tower](https://crates.io/crates/tower) middleware (and you should be if you're on axum or tonic), the [`tower-resilience`](https://crates.io/crates/tower-resilience) crate gives you circuit breakers, bulkheads, rate limiters, and timeouts as composable layers:

```rust
use tower::ServiceBuilder;
use std::time::Duration;

let service = ServiceBuilder::new()
    .timeout(Duration::from_secs(5))       // Don't wait forever
    .rate_limit(1000, Duration::from_secs(1))  // Protect downstream
    .service(inner_service);
```

## Alerting: what to page on

Having metrics is pointless without alerts. But alerting on the wrong things means alert fatigue, and alert fatigue means you ignore the real pages.

**Page-worthy (wake someone up):**
- Error rate exceeds 1% for 5 minutes
- p99 latency exceeds SLO (e.g., 500ms) for 5 minutes
- Health readiness probe failing for 2+ minutes
- Zero requests received for 3+ minutes (service might be unreachable)
- Disk usage above 90%

**Warning (Slack notification, investigate during business hours):**
- Circuit breaker opened for any downstream dependency
- Connection pool utilization above 80%
- Memory usage trending upward (possible leak)
- Error rate above 0.5% but below 1%
- Response time p50 drifting up over 24 hours

**Not worth alerting on:**
- CPU spikes (transient spikes are normal; sustained high CPU with no latency impact is fine)
- Individual 5xx responses (this is what error *rate* is for)
- Deployment events (track them, but they're not alerts)

The general principle: alert on *symptoms* (users are affected) not *causes* (CPU is high). High CPU with normal latency and error rates means your service is busy but healthy.

## Self-hosted vs managed: picking a backend

You have all these signals - logs, traces, metrics. Where do they go?

### Grafana + Prometheus + Loki + Tempo (self-hosted)

The LGTM stack. All open source (Apache 2.0).

- **Prometheus** scrapes your `/metrics` endpoint and stores time-series data
- **Loki** ingests your JSON logs (designed to index labels, not full-text)
- **Tempo** stores distributed traces (works with OTLP)
- **Grafana** dashboards tie everything together

**Cost**: The software is free. Infrastructure runs $50-500/month depending on scale. The real cost is operational - someone has to manage retention, scaling, upgrades, and backups. For a team under 10 engineers, this might mean 2-4 hours per month of maintenance. For a large deployment, it could justify a dedicated SRE.

**Grafana Cloud** offers a managed version with a generous free tier (10K metrics series, 50GB logs, 3 users). Pro tier starts at $19/user/month plus usage-based pricing.

### Datadog

All-in-one SaaS. Ingests logs, metrics, traces, and provides dashboards, alerting, APM, and profiling in one UI.

**Cost**: Infrastructure monitoring starts at $15/host/month. APM is $31-40/host/month bundled. A mid-size setup (100 engineers, 125 APM hosts, 200 infra hosts) can easily hit $300K+/year. Pricing is based on the 99th percentile of hourly host counts - so auto-scaling can surprise you.

**Rust support**: No auto-instrumentation (unlike Java or Python). You instrument with OpenTelemetry and send OTLP to the Datadog Agent. The [`datadog-opentelemetry`](https://crates.io/crates/datadog-opentelemetry) crate handles the integration.

### Sentry

Not a full observability platform - it's focused on error tracking and performance monitoring. Captures panics, errors with stack traces, and performance transactions.

**Cost**: Free tier covers 5K errors and 10K performance events/month. Team tier is $26/month. Scales to $6K-24K/year for mid-size (500K-2M events).

**Rust SDK** (`sentry` crate) integrates with `tracing` via `sentry-tracing`. It captures panic backtraces, attaches breadcrumbs, and groups similar errors automatically.

### Which one?

| Factor | Self-hosted LGTM | Datadog | Sentry |
|---|---|---|---|
| **Best for** | Full control, budget-conscious | Teams wanting zero ops | Error tracking focus |
| **Rust support** | Native (Prometheus, OTLP) | OTLP via agent | Native SDK |
| **Cost at scale** | Low (infra only) | High | Moderate |
| **Ops burden** | You manage everything | Zero | Zero |
| **Trace + metrics + logs** | Yes (3 separate tools) | Yes (unified) | Partial (errors + perf) |

My recommendation for most Rust teams: start with **Grafana Cloud free tier + Sentry free tier**. Grafana Cloud handles metrics and logs, Sentry handles errors and panics. You get the critical signals without managing infrastructure or paying Datadog prices. If you outgrow the free tiers or need unified APM, evaluate Datadog or self-host.

## Putting it all together

Here's a minimal but production-ready setup that wires everything together:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.8"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["json", "env-filter"] }
opentelemetry = "0.31.0"
opentelemetry_sdk = "0.31.0"
opentelemetry-otlp = { version = "0.31.1", features = ["grpc-tonic"] }
tracing-opentelemetry = "0.32.1"
metrics = "0.24.3"
metrics-exporter-prometheus = "0.18.1"
sentry = "0.38"
sentry-tracing = "0.38"
```

```rust
use axum::{routing::get, Router};
use std::net::SocketAddr;

#[tokio::main]
async fn main() {
    // 1. Initialize Sentry (captures panics + errors)
    let _sentry_guard = sentry::init((
        std::env::var("SENTRY_DSN").ok(),
        sentry::ClientOptions {
            release: sentry::release_name!(),
            traces_sample_rate: 0.1, // Sample 10% of transactions
            ..Default::default()
        },
    ));

    // 2. Initialize OpenTelemetry tracing
    let tracer_provider = init_telemetry();

    // 3. Initialize Prometheus metrics
    let metrics_handle = init_metrics();

    // 4. Build the app
    let app = Router::new()
        .route("/health/live", get(liveness))
        .route("/health/ready", get(readiness))
        .route("/metrics", get(move || async move {
            metrics_handle.render()
        }))
        // ... your actual routes ...
        ;

    // 5. Run
    let addr = SocketAddr::from(([0, 0, 0, 0], 3000));
    tracing::info!(%addr, "starting server");

    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    // 6. Flush on shutdown
    if let Err(e) = tracer_provider.shutdown() {
        eprintln!("tracer shutdown error: {e}");
    }
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c().await.ok();
    tracing::info!("shutdown signal received");
}
```

The initialization order matters. Sentry goes first because it installs a panic handler - you want that active before anything else can panic. OpenTelemetry and metrics come next. On shutdown, the reverse: flush spans before the process exits.

## What this looks like in practice

With this setup running in production:

1. **Normal operation**: Grafana dashboards show steady request rates, flat latency percentiles, near-zero error rates. Connection pool utilization hovers at 30%.

2. **Slow dependency**: p99 latency ticks up. Traces show the database query step taking 500ms instead of 10ms. USE metrics show the connection pool at 95% utilization. The circuit breaker hasn't tripped yet but you see it coming. You scale the database before users notice.

3. **Downstream outage**: Circuit breaker opens after 5 failures. Fallback counter starts incrementing. Error rate stays flat because you're serving cached data. You get a Slack warning about the open circuit. You investigate without anyone getting paged.

4. **Real incident**: Error rate crosses 1% for 5 minutes. PagerDuty fires. You open Grafana, see the spike, click through to Sentry for the error details, pull up a trace to see exactly which service and which call is failing. Time to diagnosis: minutes, not hours.

That's the difference monitoring makes. Your Rust binary is already fast and safe. Monitoring makes it *understandable*.
