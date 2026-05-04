+++
title = "Distributed tracing with OpenTelemetry in Rust"
date = 2025-08-16
description = "How trace context propagates across services in practice - W3C headers, the OpenTelemetry SDK, OTLP, and a working example tracing a request across two Axum services into Jaeger."

[taxonomies]
tags = ["rust", "observability", "opentelemetry", "axum"]
+++

You have one service. You add `#[instrument]` here and there, point an OTLP exporter at Jaeger, and you get a nice waterfall view. Easy.

Then you split that service into two. Now the trace stops at the network boundary. The HTTP client in service A logs its outbound request, service B logs the inbound, and Jaeger shows you two unrelated traces with no parent-child relationship between them. You can squint at timestamps and pretend, but you've lost the thing that made tracing useful.

The fix is simple in concept: serialize the current span context into HTTP headers on the way out, deserialize them on the way in, and link the new span to that remote parent. The W3C Trace Context spec defines exactly how. OpenTelemetry implements it. The Rust SDK exposes it. But wiring it up correctly involves a few details that the docs handwave past.

If you're not familiar with the basics of `tracing`, spans, and the OTel SDK, I covered those in [Logging vs tracing vs metrics](/blog/logging-vs-tracing-vs-metrics---the-three-pillars-of-observa). This post picks up where that one ended - everything past the single-service setup.

<!-- more -->

## What "context propagation" actually is

A span has a SpanContext: a TraceId (16 bytes), a SpanId (8 bytes), TraceFlags (1 byte for sampling decisions), and TraceState (vendor-specific key-value pairs). When you create a child span, the child inherits the parent's TraceId and gets a fresh SpanId, with the parent's SpanId stored as `parent_span_id`. That's how the tree is built.

Inside one process, this happens automatically. Tokio task-local storage holds the "current" span; spawning a child span reads from it. The `tracing` crate's `Span::current()` and `tracing-opentelemetry`'s context bridge handle the plumbing.

Across processes, there is no shared task-local storage. The HTTP request from service A to service B is just bytes on the wire. To preserve the trace, A has to write its current SpanContext into the request somewhere B can read it, and B has to read it before creating its first span. That somewhere is HTTP headers. The encoding is W3C Trace Context.

## W3C Trace Context, briefly

The [W3C Trace Context spec](https://www.w3.org/TR/trace-context/) defines two headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate:  rojo=00f067aa0ba902b7,congo=t61rcWkgMzE
```

The `traceparent` header has four fields separated by dashes:

- **version** (`00`) - currently always `00`, future-proofing for format changes.
- **trace-id** (`4bf92f3577b34da6a3ce929d0e0e4736`) - 16 bytes hex-encoded. Globally unique per trace.
- **parent-id** (`00f067aa0ba902b7`) - 8 bytes hex-encoded. The span ID of the *outbound* span on the sender side. Becomes the parent_span_id of the receiver's root span.
- **trace-flags** (`01`) - 8 bits. Today only the lowest bit (`sampled`) is used. `01` means "this trace is being recorded, you should record too." `00` means "we decided not to sample, you can probably skip too."

`tracestate` carries vendor-specific data and is mostly used by APM vendors (Datadog, New Relic, Lightstep) to round-trip their own correlation IDs alongside the W3C ID. You can ignore it for most applications - just propagate it untouched.

This is the format every modern tracing system speaks. Older systems used Zipkin's B3 (`X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled`) or Jaeger's `uber-trace-id`. OTel can speak any of those, but for new systems use W3C.

## The Rust SDK pieces

Three crates do the actual work:

- [`opentelemetry`](https://crates.io/crates/opentelemetry) - the API. Defines `Tracer`, `Span`, `SpanContext`, `TextMapPropagator`. No I/O.
- [`opentelemetry_sdk`](https://crates.io/crates/opentelemetry_sdk) - the SDK. Real implementations: `SdkTracerProvider`, batch processors, samplers, resource detection.
- [`opentelemetry-otlp`](https://crates.io/crates/opentelemetry-otlp) - the OTLP exporter (gRPC or HTTP). Sends spans to a collector.

For HTTP propagation you also need:

- [`opentelemetry-http`](https://crates.io/crates/opentelemetry-http) - the `HeaderInjector` and `HeaderExtractor` adapters that let propagators read/write `http::HeaderMap`.

And to bridge the `tracing` crate's spans to OTel:

- [`tracing-opentelemetry`](https://crates.io/crates/tracing-opentelemetry) - converts `tracing::Span` into OTel spans on the way out.

Versions in this post: `opentelemetry` 0.30, `opentelemetry_sdk` 0.30, `opentelemetry-otlp` 0.30, `tracing-opentelemetry` 0.30. The Rust OTel crates have moved fast historically - APIs shifted between 0.21 and 0.27 - but 0.28+ has been stable.

## A practical example: two Axum services

Let's build the smallest realistic distributed system. `service-a` is an HTTP API the user calls. It enriches the request by calling `service-b`, then returns a combined response. We want a single trace covering both services.

```
[Client] --GET /orders/42--> [service-a] --GET /customers/7--> [service-b]
```

### Cargo.toml (shared)

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-opentelemetry = "0.30"

opentelemetry = "0.30"
opentelemetry_sdk = { version = "0.30", features = ["rt-tokio"] }
opentelemetry-otlp = { version = "0.30", features = ["grpc-tonic"] }
opentelemetry-http = "0.30"
opentelemetry-semantic-conventions = "0.30"
```

### Telemetry init - shared across both services

This is the boilerplate that registers a global propagator, builds the OTLP exporter, and wires up `tracing-opentelemetry`. The only difference between services is the service name.

```rust
// telemetry.rs
use opentelemetry::trace::TracerProvider;
use opentelemetry::global;
use opentelemetry_sdk::propagation::TraceContextPropagator;
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_sdk::Resource;
use opentelemetry_otlp::SpanExporter;
use opentelemetry_semantic_conventions::resource::SERVICE_NAME;
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter, fmt};

pub fn init(service_name: &'static str) -> SdkTracerProvider {
    // Critical: this registers the W3C Trace Context propagator globally.
    // Without it, get_text_map_propagator() returns a no-op and headers
    // never get injected or extracted.
    global::set_text_map_propagator(TraceContextPropagator::new());

    let exporter = SpanExporter::builder()
        .with_tonic()
        .with_endpoint("http://localhost:4317")
        .build()
        .expect("failed to build OTLP exporter");

    let resource = Resource::builder()
        .with_attribute(opentelemetry::KeyValue::new(SERVICE_NAME, service_name))
        .build();

    let provider = SdkTracerProvider::builder()
        .with_resource(resource)
        .with_batch_exporter(exporter)
        .build();

    let tracer = provider.tracer(service_name);

    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info,h2=warn,hyper=warn,tonic=warn")))
        .with(fmt::layer().json())
        .with(OpenTelemetryLayer::new(tracer))
        .init();

    provider
}
```

The line that gets missed most often is `global::set_text_map_propagator(...)`. The `tracing-opentelemetry` layer happily creates spans without it, and your single-service traces look fine. But the moment you try to inject headers, `global::get_text_map_propagator()` returns a `NoopTextMapPropagator` that injects nothing. Your two services produce two unrelated traces and you waste an hour wondering why.

### service-b - the downstream service

`service-b` only needs to extract the incoming context. We do that with a tower middleware so every handler benefits without manual code.

```rust
// service-b/src/main.rs
use axum::{routing::get, Router, extract::Path, Json};
use opentelemetry::global;
use opentelemetry::trace::TraceContextExt;
use opentelemetry_http::HeaderExtractor;
use serde::Serialize;
use tracing::{info, instrument, Span};
use tracing_opentelemetry::OpenTelemetrySpanExt;

mod telemetry;

#[derive(Serialize)]
struct Customer {
    id: u64,
    name: String,
    tier: String,
}

#[instrument(skip_all, fields(customer_id = id))]
async fn get_customer(Path(id): Path<u64>) -> Json<Customer> {
    info!("looking up customer");
    // Pretend we hit a database.
    tokio::time::sleep(std::time::Duration::from_millis(35)).await;
    Json(Customer {
        id,
        name: "Ada Lovelace".into(),
        tier: "gold".into(),
    })
}

async fn extract_trace_context<B>(
    req: axum::http::Request<B>,
    next: axum::middleware::Next,
) -> axum::response::Response
where
    B: Send + 'static,
{
    let parent_cx = global::get_text_map_propagator(|propagator| {
        propagator.extract(&HeaderExtractor(req.headers()))
    });

    // Attach the remote context to the current tracing Span. From this
    // point on, any child #[instrument] spans become children of the
    // remote span instead of starting a new trace.
    Span::current().set_parent(parent_cx);

    next.run(req).await
}

#[tokio::main]
async fn main() {
    let provider = telemetry::init("service-b");

    let app = Router::new()
        .route("/customers/{id}", get(get_customer))
        .layer(axum::middleware::from_fn(extract_trace_context));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3001").await.unwrap();
    axum::serve(listener, app).await.unwrap();

    provider.shutdown().unwrap();
}
```

Two important details here:

1. The middleware reads headers via `HeaderExtractor`, which is just a thin wrapper that implements OTel's `Extractor` trait over `http::HeaderMap`. The propagator does the actual W3C parsing.
2. `Span::current().set_parent(parent_cx)` is the bridge. Without it, the span created by `#[instrument]` has no remote parent and starts its own trace. The `OpenTelemetrySpanExt` import is what brings `set_parent` into scope - it's easy to miss and the compiler error doesn't suggest it.

A subtle gotcha: the middleware needs to run *inside* a span that has the same scope as your handler, otherwise `Span::current()` is not the handler's span. Axum's default tower-http `TraceLayer` creates a request-scoped span, but if you're not using it you may want to wrap the middleware itself with `#[instrument]` or use `tracing::info_span!` explicitly. The simplest path is to layer `tower_http::trace::TraceLayer::new_for_http()` *under* this middleware so a span exists when `Span::current()` is called.

### service-a - the upstream service

`service-a` does the opposite: it injects the current context into outbound headers. The cleanest way is a small helper that wraps `reqwest::RequestBuilder`.

```rust
// service-a/src/main.rs
use axum::{routing::get, Router, extract::Path, Json};
use opentelemetry::global;
use opentelemetry_http::HeaderInjector;
use serde::{Deserialize, Serialize};
use tracing::{info, instrument, Span};
use tracing_opentelemetry::OpenTelemetrySpanExt;

mod telemetry;

#[derive(Deserialize)]
struct Customer {
    id: u64,
    name: String,
    tier: String,
}

#[derive(Serialize)]
struct OrderResponse {
    order_id: u64,
    customer_name: String,
    tier: String,
}

#[instrument(skip_all, fields(order_id = id))]
async fn get_order(Path(id): Path<u64>) -> Json<OrderResponse> {
    info!("fetching order");

    // In a real system you'd look this up. Hardcode for the demo.
    let customer_id: u64 = 7;

    let customer = fetch_customer(customer_id).await;

    Json(OrderResponse {
        order_id: id,
        customer_name: customer.name,
        tier: customer.tier,
    })
}

#[instrument]
async fn fetch_customer(id: u64) -> Customer {
    let client = reqwest::Client::new();
    let url = format!("http://localhost:3001/customers/{id}");

    // Build the request, then inject trace context headers.
    let mut req = client.get(&url).build().unwrap();

    let cx = Span::current().context();  // OTel context for current span
    global::get_text_map_propagator(|propagator| {
        propagator.inject_context(&cx, &mut HeaderInjector(req.headers_mut()));
    });

    let resp = client.execute(req).await.unwrap();
    resp.json::<Customer>().await.unwrap()
}

#[tokio::main]
async fn main() {
    let provider = telemetry::init("service-a");

    let app = Router::new()
        .route("/orders/{id}", get(get_order));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();

    provider.shutdown().unwrap();
}
```

The injection sequence is:

1. `Span::current()` returns the active `tracing::Span`.
2. `.context()` (from `OpenTelemetrySpanExt`) converts it into an OTel `Context` containing the current `SpanContext`.
3. `global::get_text_map_propagator(...)` grabs the propagator we registered at startup.
4. `inject_context` writes `traceparent` (and `tracestate` if non-empty) into the headers.

After this the outbound `reqwest` request carries headers like:

```
traceparent: 00-2c2a3a4b5c6d7e8f9a0b1c2d3e4f5061-1122334455667788-01
```

`service-b` extracts that, and Jaeger shows both spans under the same trace tree. That is the whole point of the exercise.

### Running it locally

You need a collector that speaks OTLP. The simplest setup is the all-in-one Jaeger image, which has accepted OTLP natively since v1.35:

```bash
docker run --rm -d \
  --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

Port 16686 is the UI. Port 4317 is OTLP/gRPC (what our exporter uses). 4318 is OTLP/HTTP if you prefer that.

Then:

```bash
RUST_LOG=info cargo run -p service-b &
RUST_LOG=info cargo run -p service-a &
curl http://localhost:3000/orders/42
```

Open `http://localhost:16686`, pick `service-a` from the dropdown, hit Find Traces. You should see one trace with three spans:

```
service-a  GET /orders/42       58ms
  service-a  fetch_customer       42ms
    service-b  GET /customers/{id}  37ms
      service-b  get_customer       35ms
```

The horizontal bars line up. Click into any span and you see the attributes - `http.method`, `http.url`, the `customer_id` field from `#[instrument]`. That is the payoff for the wiring above.

## Exporters: OTLP vs Jaeger native vs Zipkin

You'll see three exporter crates floating around the docs and old blog posts. Pick OTLP.

**OTLP** (`opentelemetry-otlp`) is the OpenTelemetry-native protocol. Two transports: gRPC (port 4317) and HTTP/protobuf (port 4318). Every modern backend speaks it: Jaeger 1.35+, Tempo, Honeycomb, Datadog, Lightstep, the OTel Collector. This is what you should use in 2026.

**Jaeger native** (the old `opentelemetry-jaeger` crate) used Jaeger's Thrift protocol over UDP. It was deprecated and the crate was archived in 2023. Don't use it. Even Jaeger itself recommends OTLP now.

**Zipkin** (`opentelemetry-zipkin`) sends spans in Zipkin's JSON format over HTTP. Useful only if your existing infrastructure is Zipkin-based and you can't change it. Otherwise OTLP into the OTel Collector and translate downstream.

For production you almost always want to run an OTel Collector between your services and the backend. The collector batches, retries, samples, redacts, and lets you switch backends without redeploying applications. Apps export OTLP to the collector, the collector exports OTLP (or whatever) to one or more destinations.

## Sampling decisions and the parent flag

The `traceparent` header carries a `sampled` bit. The receiver should respect it: if the parent says "sampled," sample; if "not sampled," don't waste storage. The default OTel sampler does exactly this via `ParentBased(TraceIdRatioBased(rate))`:

- If a parent context exists, follow its sampling decision.
- If no parent exists (this is a root span), apply the configured ratio.

This way the sampling decision is made *once* at the edge of the system and propagated consistently. Imagine the alternative: each service samples independently at 1%. A request that crosses 5 services has a 0.01^5 chance of all five spans being sampled together. You'd see basically nothing in your traces.

To configure:

```rust
use opentelemetry_sdk::trace::Sampler;

let provider = SdkTracerProvider::builder()
    .with_sampler(Sampler::ParentBased(Box::new(
        Sampler::TraceIdRatioBased(0.05) // 5% at the root
    )))
    .with_resource(resource)
    .with_batch_exporter(exporter)
    .build();
```

For higher-fidelity production setups, do head sampling at the edge for a baseline (1-5%), then tail sampling at the OTel Collector to keep error and slow traces at 100%. Tail sampling needs the collector because individual services don't know the final outcome of the trace at span-start time.

## Pitfalls I hit and you probably will too

**Forgetting to call `provider.shutdown()`.** The batch exporter buffers spans. If your process exits without flushing, you lose the last batch - which is exactly the batch that includes the span where things went wrong. Always shutdown on the way out, or use a tokio signal handler that calls it.

**Mismatched propagator on the two ends.** If service A injects W3C and service B is configured for B3, the extraction silently produces an empty context. No error, just two unrelated traces. Standardize on W3C across the fleet.

**Span attributes vs metric labels.** Putting `user_id` on a span attribute is fine and useful. Putting it on a metric label crashes Prometheus (covered in the [previous post](/blog/logging-vs-tracing-vs-metrics---the-three-pillars-of-observa)). The two have different cardinality budgets - traces are sampled, metrics are not.

**Reqwest middleware vs manual injection.** For a real codebase, use [`reqwest-middleware`](https://crates.io/crates/reqwest-middleware) with [`reqwest-tracing`](https://crates.io/crates/reqwest-tracing), which handles injection on every request automatically. The manual approach above is for clarity. The same applies inbound: use `tower-http`'s tracing layer plus a small extractor middleware rather than rolling your own.

**Span names with high cardinality.** Don't put the path parameter into the span name (`GET /customers/7`). Use the route template (`GET /customers/{id}`) and put the id in an attribute. Otherwise span names explode and aggregation in the UI becomes useless.

**Forgetting to import `OpenTelemetrySpanExt`.** `Span::current().set_parent(...)` and `Span::current().context()` only exist on this trait. The compiler error is not particularly helpful - it'll say "no method named `set_parent`" and not suggest the trait import.

## Where to go from here

The example above is intentionally minimal. Real-world setups add:

- A tower-http TraceLayer for incoming HTTP, plus a `MakeSpan` that uses route templates not paths.
- `reqwest-tracing` for outbound HTTP, so injection is automatic.
- Database instrumentation (`sqlx` has tracing built in, you just need to enable the feature).
- Async runtime instrumentation (`tokio-console` for tasks, separate from OTel).
- An OTel Collector deployment with tail sampling and PII scrubbing.

The principle stays the same: a span context flows through your system in headers, every service extracts on input and injects on output, and Jaeger (or Tempo, or whatever) reassembles the tree from the trace ID. Once you have it working for two services, scaling to twenty is just repetition.

The reward is real. The first time someone says "checkout is slow today" and you can pull up a trace showing 1.8s of that 2s in a single Postgres call in service C, you'll wonder how anyone debugged distributed systems before this existed. The answer is "badly, mostly by guessing." Don't go back.
