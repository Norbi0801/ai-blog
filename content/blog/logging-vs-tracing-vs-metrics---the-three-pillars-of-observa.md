+++
title = "Logging vs tracing vs metrics - the three pillars of observability"
date = 2026-04-04
description = "What logs, traces, and metrics actually are, when to use each, and how to set up a practical observability stack in Rust with tracing, metrics, and OpenTelemetry."

[taxonomies]
tags = ["rust", "observability", "devops", "architecture"]
+++

Your service is running. Users are making requests. Then someone reports that checkout is slow. Or worse, Slack lights up with "is the API down?" and you're staring at a terminal with no idea what's happening inside your own system.

You check the logs. There are thousands of lines, most useless. You look at your monitoring dashboard - oh wait, you don't have one. You try to figure out which service is causing the slowdown, but you have no way to follow a request across service boundaries.

This is the gap observability fills. Not just "can I see logs" but "can I understand what my system is doing right now, and why it's doing it badly."

There are three fundamental signal types that give you this understanding: logs, metrics, and traces. Each answers a different question. None replaces the others. Getting all three right - and knowing when to use which - is the difference between debugging in minutes and debugging in hours.

<!-- more -->

## Logs - what happened

Logs are the simplest signal. An event happened, you write it down. A request came in, a query ran, an error occurred, a user logged in. Each log entry is a discrete record of something that took place at a specific moment.

```
2026-04-02T14:23:01Z INFO  request completed method=GET path=/api/orders status=200 duration_ms=42
2026-04-02T14:23:01Z ERROR failed to connect to payment gateway err="connection refused" retry=3
2026-04-02T14:23:02Z WARN  rate limit approaching user_id=usr_8f3a remaining=12
```

Logs are great for debugging specific incidents. When something breaks, you want to know exactly what happened, in what order, with as much context as possible. The error message, the stack trace, the request parameters, the user ID - all of it.

But logs have real problems at scale:

**Volume.** A busy service can produce millions of log lines per hour. Storing, indexing, and searching them costs real money. At one point in my career I watched a team's logging costs exceed their compute costs because nobody put limits on what got logged.

**No aggregation.** Logs tell you about individual events. They don't tell you "what percentage of requests are failing" or "what's the p99 latency." You can derive those numbers from logs, but it's expensive - you're scanning millions of records to compute a single number.

**No causality.** A plain log line doesn't know it belongs to the same request as another log line in a different service. You can add correlation IDs manually, but that's already moving toward tracing.

### Structured vs unstructured

Unstructured logs are strings. Structured logs are key-value pairs. If you're starting a new project in 2026, there's no reason to use unstructured logs. Ever.

```
// Unstructured - good luck parsing this programmatically
"User john@example.com placed order #4521 for $129.99 at 2026-04-02T14:23:01Z"

// Structured - every field is queryable
{"timestamp": "2026-04-02T14:23:01Z", "level": "info", "event": "order_placed",
 "user_email": "john@example.com", "order_id": 4521, "amount_cents": 12999}
```

Structured logs let you filter by any field without regex gymnastics. "Show me all errors from the payment service in the last hour where the user was on the enterprise plan." With structured logs that's a query. With unstructured logs that's a prayer.

## Metrics - what's the number

Metrics are numeric measurements collected over time. CPU usage is 73%. The request rate is 1,200 req/s. The p99 latency is 340ms. The error rate jumped from 0.1% to 4.2% in the last five minutes.

Three types cover almost everything:

**Counters** go up. Total requests served. Total errors. Total bytes transferred. They only ever increment (or reset to zero on restart). You derive rates from counters - "requests per second" is just the counter's rate of change.

**Gauges** go up and down. Current memory usage. Number of active connections. Queue depth. A gauge is a snapshot of a value that fluctuates.

**Histograms** record distributions. You don't just want the average latency - you want the p50, p90, p95, p99. A histogram buckets individual observations so you can answer "what percentage of requests complete in under 100ms?" Averages hide the pain; histograms expose it.

```
# Prometheus exposition format
http_requests_total{method="GET", path="/api/orders", status="200"} 48291
http_requests_total{method="GET", path="/api/orders", status="500"} 17
http_request_duration_seconds_bucket{le="0.01"} 39201
http_request_duration_seconds_bucket{le="0.05"} 45892
http_request_duration_seconds_bucket{le="0.1"} 47103
http_request_duration_seconds_bucket{le="0.5"} 48201
http_request_duration_seconds_bucket{le="1.0"} 48291
http_request_duration_seconds_bucket{le="+Inf"} 48291
active_connections 342
```

Metrics excel at answering "how is the system doing right now" and "how has it changed over time." Dashboards, alerts, SLOs - they're all built on metrics. When your alert fires at 3am saying error rate exceeded 5%, that's a metric crossing a threshold.

But metrics can't tell you *why*. Error rate spiked - which endpoint? Which users? What error? For that you need logs or traces. Metrics are the smoke detector. Logs and traces are the investigation.

### The cardinality trap

Every unique combination of label values creates a new time series. `method=GET, path=/api/orders, status=200` is one series. `method=POST, path=/api/orders, status=201` is another. This is fine with a handful of labels.

It stops being fine when you add high-cardinality labels like `user_id` or `request_id`. If you have a million users and three label dimensions, you've just created millions of time series. Your Prometheus instance will OOM, your storage will explode, and your monitoring team will hate you.

Rule of thumb: metric labels should have bounded, low cardinality. HTTP method (a few values), status code bucket (5 groups), service name (a fixed set), endpoint (dozens, not thousands). Anything unbounded belongs in logs or trace attributes, not metric labels.

## Traces - where did the time go

A trace follows a single request as it moves through your system. Not just "the request took 340ms" but "it spent 2ms in the API gateway, 15ms in the auth service, 280ms in the database query, 40ms serializing the response, and 3ms in network overhead."

Traces are built from **spans**. A span represents a unit of work - a function call, an HTTP request, a database query. Spans have a start time, an end time, and metadata (attributes). Spans nest inside other spans to form a tree, and the root span represents the entire request.

```
[Trace ID: abc123]
├── [Span: HTTP GET /api/orders] 340ms
│   ├── [Span: authenticate] 15ms
│   │   └── [Span: jwt_verify] 2ms
│   ├── [Span: fetch_orders] 285ms
│   │   ├── [Span: db_query SELECT * FROM orders] 270ms
│   │   └── [Span: serialize_response] 12ms
│   └── [Span: apply_rate_limit] 3ms
```

One look at that trace and you know the database query is the bottleneck. No log grepping, no guessing. The structure gives you the answer.

Traces really shine in distributed systems. When a request crosses from service A to service B to service C, each service creates its own spans but they all share a trace ID. This **context propagation** - passing the trace ID across service boundaries via HTTP headers or message metadata - is what makes distributed tracing possible.

Without traces, debugging a slow request in a microservices architecture means correlating logs across five different services, guessing at timing relationships, and hoping the clocks are synchronized. With traces, you get a single waterfall view of the entire request lifecycle.

### The sampling question

Tracing every request in a high-traffic system is expensive. At 10,000 requests per second, that's 10,000 trace trees per second, each with potentially dozens of spans. The storage and processing costs add up fast.

The solution is sampling. You don't need to trace every request - you need to trace *enough* requests to understand system behavior. Common strategies:

- **Head-based sampling**: Decide at the start whether to trace this request. Simple but you might miss interesting failures.
- **Tail-based sampling**: Collect everything, but only persist traces that are interesting (slow, errored, flagged). More expensive to run but catches the traces you actually care about.
- **Rate-limited sampling**: Keep N traces per second, regardless of traffic. Predictable cost.

A 1-5% sample rate is common in production. For error traces, always keep 100% - those are the ones you'll need.

## How they complement each other

Here's the mental model that clicks:

| Signal  | Question it answers                 | Good for                          | Bad for                        |
|---------|-------------------------------------|-----------------------------------|--------------------------------|
| Logs    | What exactly happened?              | Debugging specific incidents      | Aggregation, trends            |
| Metrics | How much / how fast / how often?    | Dashboards, alerts, SLOs          | Root cause analysis            |
| Traces  | Where did the time go?              | Latency analysis, dependencies    | High-volume event recording    |

A typical incident investigation uses all three:

1. **Metrics** tell you something is wrong. Error rate alert fires, latency p99 spikes on the dashboard.
2. **Traces** tell you where. You find a slow trace, see that the `inventory-service` span takes 2 seconds instead of the usual 50ms.
3. **Logs** tell you why. You look at the inventory service logs for that trace ID and find `"connection pool exhausted, waited 1.8s for available connection"`.

Metrics detect, traces locate, logs explain. If you only have one, you're blind to the other dimensions. Teams that rely only on logs spend hours grepping. Teams that rely only on metrics know something is broken but can't figure out what. Teams that rely only on traces can diagnose individual requests but miss systemic trends.

## The Rust observability stack

Rust's ecosystem has converged on a few crates that cover all three pillars. The good news: they're mature, performant, and they work together.

### tracing - your logging and tracing foundation

If you've read [Debugging Rust Beyond println!](/blog/debugging-rust-beyond-println), you already know the basics of the [`tracing`](https://crates.io/crates/tracing) crate. It handles both structured logging and span-based tracing through a single API. At 387 million downloads on crates.io and maintained by the Tokio project, it's the de facto standard.

The key insight: `tracing` doesn't force you to choose between logging and tracing. An `info!()` call inside a span automatically inherits that span's context. Your logs know which request they belong to, which function they're in, and what the relevant parameters are - without you manually threading context through every function.

```rust
use tracing::{info, warn, instrument};

#[instrument(skip(db_pool))]
async fn process_order(
    order_id: &str,
    user_id: &str,
    db_pool: &Pool,
) -> Result<Order, AppError> {
    info!("processing order");

    let user = fetch_user(user_id, db_pool).await?;
    let inventory = check_inventory(order_id, db_pool).await?;

    if inventory.available < 1 {
        warn!(available = inventory.available, "insufficient inventory");
        return Err(AppError::OutOfStock);
    }

    let order = create_order(order_id, &user, db_pool).await?;
    info!(total_cents = order.total_cents, "order completed");
    Ok(order)
}

#[instrument(skip(db_pool))]
async fn fetch_user(user_id: &str, db_pool: &Pool) -> Result<User, AppError> {
    info!("fetching user from database");
    // ...
}
```

The `#[instrument]` macro creates a span for each function call. The `order_id` and `user_id` parameters are automatically recorded as span fields. Every `info!`, `warn!`, or `error!` call inside that function appears within the span's context. The output looks like this:

```
2026-04-02T14:23:01Z INFO process_order{order_id="ord_123" user_id="usr_456"}: processing order
2026-04-02T14:23:01Z INFO process_order{order_id="ord_123" user_id="usr_456"}:fetch_user{user_id="usr_456"}: fetching user from database
2026-04-02T14:23:01Z INFO process_order{order_id="ord_123" user_id="usr_456"}: order completed total_cents=4999
```

Every log line carries its full span context. No manual correlation IDs. No boilerplate. The span nesting is automatic.

#### Setting up tracing-subscriber

The `tracing` crate defines the API. `tracing-subscriber` provides the actual implementations that decide what to do with the data - format it for stdout, filter by level, export to a collector.

```rust
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt, util::SubscriberInitExt};

fn init_logging() {
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info")))
        .with(fmt::layer()
            .json()                    // structured JSON output
            .with_target(true)         // include module path
            .with_thread_ids(true)     // useful for debugging concurrency
            .with_span_events(fmt::format::FmtSpan::CLOSE)) // log span durations
        .init();
}
```

The `EnvFilter` lets you control verbosity at runtime via the `RUST_LOG` environment variable. In development you might run `RUST_LOG=debug`. In production, `RUST_LOG=info,hyper=warn,tower=warn` keeps things quiet while still giving you visibility into your own code.

The `.json()` formatter outputs each log event as a JSON object - perfect for shipping to Elasticsearch, Loki, or any log aggregation system that understands structured data.

### metrics - lightweight, fast metric collection

The [`metrics`](https://crates.io/crates/metrics) crate follows a similar architecture to `tracing`: a facade crate defines macros (`counter!`, `gauge!`, `histogram!`), and an exporter crate handles where the numbers go.

```rust
use metrics::{counter, gauge, histogram};
use std::time::Instant;

async fn handle_request(req: Request) -> Response {
    let start = Instant::now();

    counter!("http_requests_total",
        "method" => req.method().to_string(),
        "path" => req.path().to_string()
    ).increment(1);

    gauge!("http_connections_active").increment(1.0);

    let response = process(req).await;

    gauge!("http_connections_active").decrement(1.0);

    histogram!("http_request_duration_seconds").record(start.elapsed().as_secs_f64());

    counter!("http_requests_total",
        "method" => req.method().to_string(),
        "path" => req.path().to_string(),
        "status" => response.status().as_u16().to_string()
    ).increment(1);

    response
}
```

The macros are essentially zero-cost when no exporter is installed - they compile down to no-ops. This means libraries can instrument themselves with metrics without imposing any cost on users who don't collect them.

For the exporter, `metrics-exporter-prometheus` is the most common choice:

```rust
use metrics_exporter_prometheus::PrometheusBuilder;

fn init_metrics() {
    // Starts an HTTP listener on :9000/metrics for Prometheus to scrape
    PrometheusBuilder::new()
        .with_http_listener(([0, 0, 0, 0], 9000))
        .install()
        .expect("failed to install Prometheus recorder");
}
```

That's it. Every `counter!`, `gauge!`, and `histogram!` call in your application now shows up at `http://localhost:9000/metrics` in Prometheus exposition format. Point Prometheus at it, build a Grafana dashboard, set up alerts. Standard stuff.

#### What to actually measure

The temptation is to metric everything. Resist it. Start with the [RED method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/) for request-driven services:

- **R**ate: requests per second
- **E**rrors: failed requests per second
- **D**uration: latency distribution (histogram)

And the [USE method](https://www.brendangregg.com/usemethod.html) for resources:

- **U**tilization: how full is the resource (CPU, memory, disk, connection pool)
- **S**aturation: how much work is waiting (queue depth, thread pool backlog)
- **E**rrors: resource-level errors (connection failures, OOM kills)

That gives you a solid baseline. Add custom business metrics only when you have a specific question: "how many orders per minute are we processing," "what's the cache hit rate," "how many retries are we doing on the payment gateway."

### OpenTelemetry - the unifying standard

[OpenTelemetry](https://opentelemetry.io/) (OTel) is a vendor-neutral standard for all three signals. Instead of sending traces to Jaeger, metrics to Prometheus, and logs to Loki through separate pipelines, OTel gives you a single SDK that exports everything in a standard format (OTLP) to any compatible backend.

The Rust implementation at version 0.30.0 supports traces, metrics (now stable), and logs. The key crate for Rust developers is [`tracing-opentelemetry`](https://crates.io/crates/tracing-opentelemetry), which bridges the `tracing` crate's spans into OpenTelemetry traces. You keep writing `#[instrument]` and `info!()` - the bridge handles converting them to OTel format.

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-opentelemetry = "0.30"
opentelemetry = { version = "0.30", features = ["trace"] }
opentelemetry_sdk = { version = "0.30", features = ["rt-tokio", "trace"] }
opentelemetry-otlp = { version = "0.30", features = ["tokio"] }
```

```rust
use opentelemetry::trace::TracerProvider;
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_otlp::SpanExporter;
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter, fmt};

fn init_telemetry() -> SdkTracerProvider {
    // OTLP exporter sends traces to a collector (Jaeger, Tempo, etc.)
    let exporter = SpanExporter::builder()
        .with_tonic()       // gRPC transport
        .build()
        .expect("failed to create OTLP exporter");

    let provider = SdkTracerProvider::builder()
        .with_batch_exporter(exporter)
        .build();

    let tracer = provider.tracer("my-service");

    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info")))
        .with(fmt::layer().json())                    // logs to stdout
        .with(OpenTelemetryLayer::new(tracer))        // traces to OTLP
        .init();

    provider
}

#[tokio::main]
async fn main() {
    let provider = init_telemetry();

    // ... run your app ...

    // Flush remaining spans on shutdown
    provider.shutdown().expect("failed to shutdown tracer");
}
```

Now every `#[instrument]` span in your application is both a structured log (via `fmt::layer`) and an OpenTelemetry trace span (via `OpenTelemetryLayer`). Same code, two outputs. The logs go to stdout (for Loki, CloudWatch, whatever your log pipeline is). The traces go to your OTLP collector (Jaeger, Grafana Tempo, Honeycomb, Datadog).

### Putting it all together

Here's a realistic setup that covers all three pillars. This is what I'd put in a production Rust service:

```rust
use metrics::counter;
use metrics_exporter_prometheus::PrometheusBuilder;
use opentelemetry::trace::TracerProvider;
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_otlp::SpanExporter;
use tracing::{info, instrument};
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter, fmt};
use std::time::Instant;

fn init_observability() -> SdkTracerProvider {
    // Metrics: Prometheus exporter on :9000
    PrometheusBuilder::new()
        .with_http_listener(([0, 0, 0, 0], 9000))
        .install()
        .expect("failed to install metrics recorder");

    // Traces: OTLP exporter to collector
    let exporter = SpanExporter::builder()
        .with_tonic()
        .build()
        .expect("failed to create OTLP exporter");

    let provider = SdkTracerProvider::builder()
        .with_batch_exporter(exporter)
        .build();

    let tracer = provider.tracer("order-service");

    // Logs + Traces: combined subscriber
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info")))
        .with(fmt::layer().json())
        .with(OpenTelemetryLayer::new(tracer))
        .init();

    provider
}

#[instrument(skip(db))]
async fn handle_create_order(
    user_id: &str,
    items: &[OrderItem],
    db: &DbPool,
) -> Result<Order, AppError> {
    let start = Instant::now();

    counter!("orders_received_total").increment(1);
    info!(item_count = items.len(), "received order request");

    // Validate inventory - creates a child span
    let availability = check_inventory(items, db).await?;
    if !availability.all_available {
        counter!("orders_rejected_total", "reason" => "out_of_stock").increment(1);
        info!("order rejected - insufficient inventory");
        return Err(AppError::OutOfStock);
    }

    // Charge payment - creates a child span
    let payment = charge_payment(user_id, availability.total_cents, db).await?;

    // Create order record - creates a child span
    let order = persist_order(user_id, items, &payment, db).await?;

    counter!("orders_completed_total").increment(1);
    metrics::histogram!("order_processing_duration_seconds")
        .record(start.elapsed().as_secs_f64());

    info!(order_id = %order.id, total_cents = order.total_cents, "order completed");
    Ok(order)
}

#[instrument(skip(db))]
async fn check_inventory(items: &[OrderItem], db: &DbPool) -> Result<Availability, AppError> {
    info!(item_count = items.len(), "checking inventory");
    // ... actual implementation
}

#[instrument(skip(db))]
async fn charge_payment(
    user_id: &str,
    amount_cents: i64,
    db: &DbPool,
) -> Result<Payment, AppError> {
    info!(amount_cents, "charging payment");
    // ... actual implementation
}
```

What you get from this single instrumented codebase:

1. **Logs** (JSON to stdout): every `info!` call, with full span context, filterable by any field.
2. **Traces** (OTLP to Jaeger/Tempo): the complete span tree for every request - `handle_create_order` -> `check_inventory` -> `charge_payment` -> `persist_order`, with timing for each.
3. **Metrics** (Prometheus on :9000): `orders_received_total`, `orders_completed_total`, `orders_rejected_total`, `order_processing_duration_seconds` - ready for dashboards and alerts.

Three pillars, one codebase, minimal boilerplate.

## Common mistakes

I've seen (and made) enough observability mistakes to fill a book. Here are the ones that hurt the most:

### Logging everything

The "log every function entry and exit" approach. Your production service writes 50GB of logs per day and nobody reads 99.9% of them. Worse, the signal gets buried in noise - when you actually need to find something, you're searching a haystack.

Be intentional. Log at boundaries: incoming requests, outgoing calls, errors, and business-significant events. Skip the routine internal steps unless you're debugging something specific (that's what `debug!` and `trace!` levels are for - leave them in the code but filter them out in production).

### Measuring nothing

The opposite extreme. No metrics, no dashboards, no alerts. You find out about problems when users complain. This usually happens because setting up metrics feels like a yak-shave - you need Prometheus, Grafana, storage, configuration.

Start small. The RED metrics (rate, errors, duration) for your main endpoints take 20 lines of code with the `metrics` crate. Add a Prometheus exporter, point a free Grafana Cloud instance at it, and set up one alert: "error rate > 5% for 5 minutes." You can build from there.

### Using trace IDs but not traces

Adding a `request_id` to every log line is good. But it's not tracing. You still have flat log lines with a shared ID - you see that they belong together, but you don't see the causal relationships or the timing breakdown. That's like having puzzle pieces but no picture on the box.

If you're already adding request IDs to logs, the jump to actual distributed tracing is small. The `tracing` crate gives you spans with real parent-child relationships. Add the OpenTelemetry layer and you get waterfall views in Jaeger for free.

### High-cardinality metric labels

I mentioned this earlier but it's worth repeating because it's the most common way to crash a Prometheus instance. User IDs, request IDs, email addresses, file paths - none of these belong in metric labels. Use bounded values: HTTP methods, status code classes (2xx, 4xx, 5xx), service names, endpoint groups.

If you need per-user breakdown, put user_id in trace attributes or log fields. That's what they're for.

### Not correlating signals

Your metrics alert fires. You open Grafana, see the spike. Then you open a completely separate tool to search logs. Then another tool for traces. None of them link to each other. You're manually copying timestamps and searching.

Modern observability stacks (Grafana + Tempo + Loki, Datadog, Honeycomb) let you jump from a metric to exemplar traces to correlated logs. The key is using the same trace ID everywhere. The `tracing` + `tracing-opentelemetry` setup does this automatically - your log events include the trace ID as a field, so you can pivot from a log line to the full trace and back.

## The practical setup checklist

If you're starting from zero, here's the order I'd recommend:

**Week 1: Structured logging.** Replace `println!` and `log` with `tracing` + `tracing-subscriber`. Use `#[instrument]` on your handler functions. Output JSON. Ship to whatever log aggregation you have (even just `docker logs` piped to Loki).

**Week 2: Basic metrics.** Add the `metrics` crate + `metrics-exporter-prometheus`. Instrument your HTTP layer with RED metrics. Set up Prometheus + Grafana (or use a hosted service). Create one dashboard with request rate, error rate, and latency percentiles. Set up one alert.

**Week 3: Distributed tracing.** Add `tracing-opentelemetry` + `opentelemetry-otlp`. Point it at Jaeger or Grafana Tempo. Verify you can see request traces end-to-end. If you have multiple services, add context propagation headers.

**Week 4: Connect them.** Ensure your log output includes trace IDs. Set up exemplars in Prometheus so you can jump from a metric to a trace. Build a runbook: "when this alert fires, here's which dashboard to check, here's how to find the relevant traces."

You don't need to boil the ocean. Each step gives you immediate value. And each step makes the next one more powerful - logs with trace IDs are more useful than logs without them, and metrics with exemplar traces are more useful than metrics alone.

## When to use what

Quick decision guide:

- "Is the system healthy right now?" - **Metrics**. Glance at the dashboard.
- "Why is this specific request slow?" - **Traces**. Find the trace, read the waterfall.
- "What error did this specific user hit?" - **Logs**. Filter by user ID and time range.
- "Has latency gotten worse since last deploy?" - **Metrics**. Compare p99 before and after.
- "Which downstream service is the bottleneck?" - **Traces**. Look at span durations.
- "What was the exact payload that caused the crash?" - **Logs**. Find the error log with context.
- "Should we alert on this?" - **Metrics**. Alerts need numeric thresholds.
- "Is this a systemic issue or a single bad request?" - **Metrics** first (is the error rate up?), then **traces** (are all slow traces hitting the same span?).

Observability isn't a feature you ship once. It's an ongoing practice. You add instrumentation as you encounter blind spots, remove noisy signals that nobody looks at, and refine your dashboards as you learn what questions you actually ask during incidents. The three pillars give you the vocabulary. The Rust ecosystem gives you the tools. The rest is discipline.
