+++
title = "Structured logging in Rust with tracing"
date = 2025-04-29
description = "A deep dive into the tracing crate internals: spans, events, the Layer system, structured fields, per-layer filtering, custom layers, and a practical setup from scratch."

[taxonomies]
tags = ["rust", "tracing", "observability", "logging"]
+++

The `tracing` crate has quietly become the standard for diagnostics in Rust. Over 527 million downloads. Every major async framework depends on it. If you've read my [monitoring post](/blog/monitoring-rust-applications-in-production), you saw the production setup - JSON logs, OpenTelemetry export, the full stack. And the [debugging post](/blog/debugging-rust-beyond-println) introduced `#[instrument]` as a replacement for println-debugging async code.

This post goes deeper. How does tracing actually work under the hood? What are the data structures behind spans? Why does the Layer system exist, and how do you write your own? What does `#[instrument]` really generate for async functions? We're going to look at the source code.

<!-- more -->

## Why not `log`?

The [`log`](https://crates.io/crates/log) crate (v0.4.29, ~798 million downloads) is Rust's original logging facade. Five severity levels, a global logger, simple text messages. It works. But it was designed for a world of single-threaded synchronous programs.

The fundamental problem: `log` records are *events* - isolated points in time. A log line says "this happened" but not "this happened while processing request X inside function Y." There's no built-in concept of context propagation.

```rust
// With log: two concurrent requests produce interleaved noise
log::info!("processing order");
log::info!("fetching items");
log::info!("processing order");  // which order? no idea
```

The `log` crate added key-value structured fields in 0.4.x via the `kv` feature, but it's opt-in and bolted on. The `Record` struct carries level, target, module path, file, line, and a format message. That's it.

`tracing` was designed from scratch with two core concepts that `log` lacks:

**Spans** - ranges of time with attached context. A span has a beginning, an end, and can be entered and exited multiple times. Events emitted inside a span automatically inherit its context.

**Structured fields** - first-class typed key-value pairs on every span and event, not afterthoughts. When you write `info!(user_id = 42, "request received")`, `user_id` is a typed field that formatters and exporters can access programmatically - not just a string interpolated into a message.

The compatibility bridge exists both ways. `tracing-log` lets tracing consume `log` records, and `tracing` emits `log` records by default (so libraries using `log` still show up in your tracing subscriber). You don't have to choose one ecosystem and abandon the other.

## The architecture: Callsite, Metadata, Subscriber, Dispatch

Every `span!()` or `event!()` macro invocation creates a static [`Callsite`](https://docs.rs/tracing-core/latest/tracing_core/callsite/index.html). This is a compile-time registration point containing `Metadata` - the span/event name, target module, severity level, list of field names, and source file location. All static. Zero runtime allocation.

```rust
// This event macro:
tracing::info!(user_id = 42, "login succeeded");

// Creates a static Metadata containing:
// - name: "event src/auth.rs:15"
// - target: "my_app::auth"
// - level: INFO
// - fields: ["user_id", "message"]
// - file: "src/auth.rs", line: 15
```

On first use, the callsite registers with a global registry and asks the current subscriber: "are you interested in this?" The subscriber responds with one of three states:

- **Always** - unconditionally interested; skip the `enabled()` check on future calls
- **Sometimes** - need to re-check `enabled()` each time (maybe filtering depends on runtime state)
- **Never** - not interested; the macro short-circuits entirely, near-zero cost

This **interest caching** is what makes disabled tracing instrumentation almost free. If your subscriber says "Never" to a DEBUG-level callsite, every subsequent hit is a single atomic load checking the cached interest. No function calls, no formatting, no allocation.

The [`Dispatch`](https://docs.rs/tracing-core/latest/src/tracing_core/dispatcher.rs.html) struct wraps the subscriber and routes events to it. It has two modes:

```rust
enum Kind<T> {
    Global(&'static (dyn Subscriber + Send + Sync)),  // no allocation
    Scoped(T),                                          // Arc-wrapped
}
```

The global dispatcher (set via `set_global_default()` or `.init()`) uses a static reference. No Arc, no refcount, no allocation on dispatch. This is the common case in production.

## Span internals

The [`Span`](https://github.com/tokio-rs/tracing/blob/master/tracing/src/span.rs) struct is small:

```rust
pub struct Span {
    inner: Option<Inner>,
    meta: Option<&'static Metadata<'static>>,
}

struct Inner {
    id: Id,           // NonZeroU64
    subscriber: Dispatch,
}
```

Roughly three pointer-widths plus discriminants. When the subscriber says "Not interested" for a callsite, the span is created with `inner: None` - a **disabled span**. Entering, exiting, and dropping a disabled span does nothing. The `Drop` implementation is `#[inline(always)]` with a branch checking emptiness, and the compiler typically eliminates it entirely.

Where does span data actually live? That's the subscriber's job. The standard `Registry` from `tracing-subscriber` uses a [lock-free sharded slab](https://github.com/tokio-rs/tracing/blob/master/tracing-subscriber/src/registry/sharded.rs):

```rust
pub struct Registry {
    spans: Pool<DataInner>,                          // sharded-slab
    current_spans: ThreadLocal<RefCell<SpanStack>>,  // per-thread span stack
}

struct DataInner {
    metadata: &'static Metadata<'static>,
    parent: Option<Id>,
    ref_count: AtomicUsize,
    extensions: RwLock<ExtensionsInner>,  // typemap for Layer-specific data
}
```

The `Pool<DataInner>` is from the [`sharded-slab`](https://crates.io/crates/sharded-slab) crate. Span storage is sharded by the creating thread for fast access in the common case. Cross-thread access uses a lock-free steal mechanism. When a span closes, its slot is cleared in-place and recycled - no allocation churn even under thousands of spans per second.

The `extensions` field is a typemap. Each Layer can store its own per-span data (like an OpenTelemetry span context, or timing information) without coordinating with other layers. This is the key design that makes layer composition work.

## Structured fields

Fields in tracing are not strings. They are typed values recorded through the [`Value`](https://docs.rs/tracing-core/latest/tracing_core/field/trait.Value.html) trait:

```rust
pub trait Value {
    fn record(&self, key: &Field, visitor: &mut dyn Visit);
}
```

When you write:

```rust
tracing::info!(
    user_id = 42_u64,
    email = %user.email,         // Display formatting
    request = ?req,              // Debug formatting
    admin = true,
    "login succeeded"
);
```

Each field is recorded with its original type. The `%` sigil calls `Display`, `?` calls `Debug`. Without a sigil, the value uses its native `Value` implementation - integers, bools, and strings are recorded without any formatting overhead.

This matters for machine consumption. A JSON formatter can emit `"user_id": 42` as a number, not `"user_id": "42"` as a string. A metrics layer can extract numeric fields directly. An OpenTelemetry exporter maps them to typed span attributes.

### Span fields vs event fields

Span fields are recorded once and carried for the span's lifetime. Event fields belong to a single event. When an event fires inside a span, subscribers see both:

```rust
let span = tracing::info_span!("http_request", method = "GET", path = "/api/users");
let _guard = span.enter();

// This event inherits the span's method and path fields
tracing::info!(status = 200, latency_ms = 14, "request completed");
```

In the JSON output (using `flatten_event(true)` as shown in the [monitoring post](/blog/monitoring-rust-applications-in-production)):

```json
{
  "timestamp": "2026-05-22T10:30:00Z",
  "level": "INFO",
  "message": "request completed",
  "status": 200,
  "latency_ms": 14,
  "http_request.method": "GET",
  "http_request.path": "/api/users"
}
```

Span fields get prefixed with the span name. Every event inside that span automatically carries the context. No manually threading a `request_id` through ten function calls.

### Recording fields after span creation

Sometimes you don't know a field's value when the span starts:

```rust
let span = tracing::info_span!("db_query", rows = tracing::field::Empty);

async fn execute_query(span: &tracing::Span) {
    let rows = sqlx::query("SELECT ...").fetch_all(&pool).await?;
    span.record("rows", rows.len());
    // "rows" is now filled in for all subsequent events in this span
}
```

The `tracing::field::Empty` placeholder declares the field in the span's metadata (so subscribers know to expect it) without recording a value yet. This is useful for fields that represent outcomes - you know the span will produce a row count, but you don't know it yet.

## The Layer system

The `Subscriber` trait represents a *complete* collection strategy. It assigns span IDs, stores span data, filters events, and formats output. Implementing a full `Subscriber` from scratch is heavy - and you can only have one active subscriber.

The [`Layer`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/trait.Layer.html) trait decomposes this. A Layer is a modular behavior that observes span and event lifecycle hooks without owning the infrastructure:

```rust
pub trait Layer<S: Subscriber> {
    fn on_new_span(&self, attrs: &Attributes<'_>, id: &Id, ctx: Context<'_, S>) { }
    fn on_event(&self, event: &Event<'_>, ctx: Context<'_, S>) { }
    fn on_enter(&self, id: &Id, ctx: Context<'_, S>) { }
    fn on_exit(&self, id: &Id, ctx: Context<'_, S>) { }
    fn on_close(&self, id: Id, ctx: Context<'_, S>) { }
    fn on_record(&self, id: &Id, values: &Record<'_>, ctx: Context<'_, S>) { }
}
```

The `Context` parameter gives read access to the inner subscriber's span data (through the `LookupSpan` trait). Layers don't assign IDs or store data - the `Registry` handles that. Layers just react.

Layers compose via `.with()`:

```rust
use tracing_subscriber::{registry::Registry, layer::SubscriberExt, fmt, EnvFilter};

let subscriber = Registry::default()
    .with(EnvFilter::from_default_env())    // filtering
    .with(fmt::layer())                      // console output
    .with(some_otel_layer)                   // OpenTelemetry export
    .with(my_custom_layer);                  // your own logic
```

Each `.with()` produces a `Layered<Layer, Inner>` struct. When an event fires, it propagates through the stack. Every layer sees every event (unless filtered).

## Per-layer filtering

Before tracing-subscriber 0.3, filtering was global. `EnvFilter` at the top of the stack decides "is this event enabled?" and all layers below see the same answer. This means if your console layer wants DEBUG output but your JSON file layer only wants WARN, you're stuck.

Per-layer filtering fixes this. Each layer can have its own [`Filter`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/trait.Filter.html):

```rust
use tracing_subscriber::{
    filter::{EnvFilter, LevelFilter},
    layer::SubscriberExt,
    fmt,
    registry::Registry,
};

let subscriber = Registry::default()
    .with(
        fmt::layer()
            .pretty()
            .with_filter(EnvFilter::new("debug"))  // console: verbose
    )
    .with(
        fmt::layer()
            .json()
            .with_writer(std::io::sink)  // would be a file in production
            .with_filter(LevelFilter::WARN)  // file: only warnings+
    );
```

Now the pretty console layer gets DEBUG events while the JSON file layer only gets WARN and above. Different layers, different filters, same subscriber. The filtering is evaluated per-layer using a `FilterMap` bitfield stored on each span in the registry.

## Writing a custom Layer

This is where tracing's architecture pays off. Need to count errors per module? Send specific events to a webhook? Track span durations? Write a Layer:

```rust
use tracing::{Event, Id, Subscriber};
use tracing_subscriber::{layer::Context, registry::LookupSpan, Layer};
use std::sync::atomic::{AtomicU64, Ordering};

pub struct ErrorCounter {
    count: AtomicU64,
}

impl ErrorCounter {
    pub fn new() -> Self {
        Self { count: AtomicU64::new(0) }
    }

    pub fn count(&self) -> u64 {
        self.count.load(Ordering::Relaxed)
    }
}

impl<S> Layer<S> for ErrorCounter
where
    S: Subscriber + for<'a> LookupSpan<'a>,
{
    fn on_event(&self, event: &Event<'_>, _ctx: Context<'_, S>) {
        if event.metadata().level() == &tracing::Level::ERROR {
            self.count.fetch_add(1, Ordering::Relaxed);
        }
    }
}
```

A more useful example - timing spans:

```rust
use std::time::Instant;
use tracing_subscriber::registry::SpanRef;

pub struct TimingLayer;

struct Timings {
    started: Instant,
}

impl<S> Layer<S> for TimingLayer
where
    S: Subscriber + for<'a> LookupSpan<'a>,
{
    fn on_new_span(
        &self,
        _attrs: &tracing::span::Attributes<'_>,
        id: &Id,
        ctx: Context<'_, S>,
    ) {
        if let Some(span) = ctx.span(id) {
            span.extensions_mut().insert(Timings {
                started: Instant::now(),
            });
        }
    }

    fn on_close(&self, id: Id, ctx: Context<'_, S>) {
        if let Some(span) = ctx.span(&id) {
            let extensions = span.extensions();
            if let Some(timings) = extensions.get::<Timings>() {
                let elapsed = timings.started.elapsed();
                let metadata = span.metadata();
                // In production you'd emit this to metrics
                println!(
                    "span {:?} (target: {}) took {:?}",
                    metadata.name(),
                    metadata.target(),
                    elapsed
                );
            }
        }
    }
}
```

Notice `span.extensions_mut().insert()` - that's the typemap on each span in the Registry. Your Layer stores its own data (the `Timings` struct) per span without interfering with other layers. The `on_close` hook fires when the span's reference count reaches zero.

## `#[instrument]` under the hood

The `#[instrument]` attribute macro from `tracing-attributes` is the most used feature. I showed basic usage in the [debugging post](/blog/debugging-rust-beyond-println). Let's look at what it actually generates.

For a **sync function**:

```rust
#[tracing::instrument]
fn process(id: u64) -> String {
    format!("done: {id}")
}

// Expands roughly to:
fn process(id: u64) -> String {
    let __span = tracing::info_span!("process", id = id);
    let __guard = __span.enter();
    { format!("done: {id}") }
    // __guard dropped here, span exited
}
```

For an **async function**, the expansion is different and critical:

```rust
#[tracing::instrument]
async fn fetch(url: &str) -> Result<String, Error> {
    reqwest::get(url).await?.text().await
}

// Expands roughly to:
fn fetch(url: &str) -> impl Future<Output = Result<String, Error>> {
    let __span = tracing::info_span!("fetch", url = url);
    async move {
        reqwest::get(url).await?.text().await
    }
    .instrument(__span)  // wraps the future with Instrumented<F>
}
```

The `.instrument(__span)` call wraps the future in [`Instrumented<F>`](https://docs.rs/tracing/latest/tracing/instrument/struct.Instrumented.html). This is the key difference from the sync case. A sync function uses `span.enter()` which returns a guard that keeps the span active until dropped. That works because the function runs to completion on one thread.

An async function can suspend at any `.await`. If we used `span.enter()` and the task suspended, the span would stay "entered" while a completely different task runs on the same thread. That task's events would incorrectly appear inside our span.

`Instrumented<F>` solves this by entering the span on every `poll()` and exiting when `poll()` returns:

```rust
// Simplified from tracing's source
impl<F: Future> Future for Instrumented<F> {
    type Output = F::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        let _guard = this.span.enter();  // enter on poll
        this.inner.poll(cx)
        // _guard drops here, span exits
    }
}
```

If you've read the [tokio internals post](/blog/understanding-tokio), you know the executor polls futures when they're woken. `Instrumented<F>` ensures the span is only "active" when the future is actually executing, not when it's parked waiting for I/O.

The `#[instrument]` macro supports several parameters worth knowing:

```rust
#[instrument(
    name = "custom_name",           // override span name
    target = "my_crate::db",       // override target
    level = "debug",                // override level (default: INFO)
    skip(pool, password),           // don't record these fields
    skip_all,                       // skip all parameters
    fields(request_id = %uuid),     // add extra fields
    err,                            // record Err variant as error event
    err(Display),                   // use Display instead of Debug for errors
    err(level = "warn"),            // error event at custom level
    ret,                            // record return value
    ret(level = "debug"),           // return value at custom level
    parent = some_span,             // explicit parent span
)]
```

The `err` parameter is particularly useful. Instead of manually logging errors at every `?` return:

```rust
#[instrument(err)]
async fn create_user(name: &str) -> Result<User, AppError> {
    let user = db.insert(name).await?;  // if this fails, error is logged
    Ok(user)
}
```

The macro wraps the return in a check - if the result is `Err`, it emits an ERROR-level event with the error value before returning.

## RUST_LOG filtering in depth

The `EnvFilter` from `tracing-subscriber` parses `RUST_LOG` with a richer syntax than most people use. The full directive format is:

```
target[span{field=value}]=level
```

Every component is optional. Some examples beyond the basics:

```bash
# Per-module with different levels
RUST_LOG="warn,my_app=debug,my_app::db=trace"

# Filter by span name - only events inside spans named "http_request"
RUST_LOG="my_app[http_request]=debug"

# Filter by field presence - only spans that have a "user_id" field
RUST_LOG="[{user_id}]=trace"

# Filter by field value (regex match against Debug output)
RUST_LOG='[{http.method="POST"}]=debug'

# Suppress noisy dependencies
RUST_LOG="info,hyper=warn,h2=warn,tower=warn,sqlx::query=warn"
```

Important detail: crate names with dashes become underscores in targets. `my-cool-crate` becomes `my_cool_crate` in RUST_LOG.

A directive without a level enables everything: `RUST_LOG="my_app::db"` is equivalent to `RUST_LOG="my_app::db=trace"`.

For production, the pattern from the [monitoring post](/blog/monitoring-rust-applications-in-production) still holds: `RUST_LOG=warn,my_service=info`. Global warn keeps dependencies quiet. Your own code at info. Bump to debug when investigating.

## tracing-appender: file output and rotation

Console output works for development. Production usually needs file output with rotation. [`tracing-appender`](https://crates.io/crates/tracing-appender) (v0.2.4) handles this:

```rust
use tracing_appender::rolling::{RollingFileAppender, Rotation};
use tracing_subscriber::{fmt, layer::SubscriberExt, registry::Registry, EnvFilter};

fn init_logging() {
    let file_appender = RollingFileAppender::new(
        Rotation::DAILY,       // rotate daily
        "/var/log/myapp",      // directory
        "myapp.log",           // filename prefix
    );

    // Non-blocking writer - logging doesn't block your async tasks
    let (non_blocking, _guard) = tracing_appender::non_blocking(file_appender);

    let subscriber = Registry::default()
        .with(EnvFilter::from_default_env())
        .with(
            fmt::layer()
                .pretty()
                .with_ansi(true)                // colored console output
        )
        .with(
            fmt::layer()
                .json()
                .flatten_event(true)
                .with_writer(non_blocking)       // JSON to file
        );

    tracing::subscriber::set_global_default(subscriber)
        .expect("failed to set subscriber");
}
```

The `non_blocking` wrapper is critical. Without it, every log write is a blocking I/O operation. In an async application, that's a worker thread stalled on `write()`. The non-blocking writer spawns a dedicated thread that drains a channel of log entries and writes them. Your async task just pushes to the channel and continues.

The `_guard` must be held for the application's lifetime. When it drops, the writer thread flushes pending entries and shuts down. Drop it before exit or you lose buffered logs. Assign it in `main()`, not in a function that returns.

Rotation options: `Rotation::MINUTELY`, `Rotation::HOURLY`, `Rotation::DAILY`, `Rotation::NEVER`. Files get named with a timestamp suffix: `myapp.log.2026-05-22`.

## Practical setup from scratch

Here's a complete, production-ready logging setup that combines everything. Two outputs: pretty console for development, JSON file for production. Per-layer filtering. Non-blocking file writer. OpenTelemetry-ready.

```toml
[dependencies]
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["json", "env-filter"] }
tracing-appender = "0.2.4"
tokio = { version = "1", features = ["full"] }
```

```rust
use tracing_appender::rolling::{RollingFileAppender, Rotation};
use tracing_subscriber::{
    filter::{EnvFilter, LevelFilter},
    fmt,
    layer::SubscriberExt,
    registry::Registry,
};

struct LogGuards {
    _file_guard: tracing_appender::non_blocking::WorkerGuard,
}

fn init_tracing() -> LogGuards {
    // File appender with daily rotation
    let file_appender = RollingFileAppender::new(
        Rotation::DAILY,
        "logs",
        "app.log",
    );
    let (non_blocking_file, file_guard) = tracing_appender::non_blocking(file_appender);

    // Console layer: pretty output, verbose in dev
    let console_layer = fmt::layer()
        .pretty()
        .with_ansi(true)
        .with_target(true)
        .with_thread_ids(false)
        .with_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("debug"))
        );

    // File layer: JSON output, only info+ in production
    let file_layer = fmt::layer()
        .json()
        .flatten_event(true)
        .with_target(true)
        .with_file(true)
        .with_line_number(true)
        .with_thread_ids(true)
        .with_writer(non_blocking_file)
        .with_filter(LevelFilter::INFO);

    let subscriber = Registry::default()
        .with(console_layer)
        .with(file_layer);

    tracing::subscriber::set_global_default(subscriber)
        .expect("failed to set tracing subscriber");

    LogGuards {
        _file_guard: file_guard,
    }
}

#[tracing::instrument(skip(pool), err)]
async fn handle_request(
    method: &str,
    path: &str,
    pool: &sqlx::PgPool,
) -> Result<String, Box<dyn std::error::Error>> {
    tracing::info!("request received");

    let rows = sqlx::query("SELECT 1")
        .fetch_all(pool)
        .await?;

    tracing::debug!(row_count = rows.len(), "query completed");
    Ok(format!("OK: {} rows", rows.len()))
}

#[tokio::main]
async fn main() {
    // _guards must live for the duration of main
    let _guards = init_tracing();

    tracing::info!(
        version = env!("CARGO_PKG_VERSION"),
        "application starting"
    );

    // ... your app setup here ...

    tracing::info!("shutting down");
    // _guards dropped here - flushes pending log entries
}
```

The setup gives you:
- Colored, human-readable console output at DEBUG level (overridable via `RUST_LOG`)
- Machine-parseable JSON in `logs/app.log` at INFO level, rotated daily
- Non-blocking file writes - your async tasks never stall on I/O
- Automatic span context on all events via `#[instrument]`
- Structured fields that JSON consumers can query directly

To add OpenTelemetry export on top of this, add the `tracing-opentelemetry` layer as shown in the [monitoring post](/blog/monitoring-rust-applications-in-production). The layer system means you just `.with()` another layer - no changes to existing code.

## Performance characteristics

Some numbers from the [tracing benchmark suite](https://github.com/tokio-rs/tracing/tree/master/tracing/benches) and [PR #1974](https://github.com/tokio-rs/tracing/pull/1974):

**Disabled spans** (no subscriber, or subscriber not interested):
- `span` creation: ~696 ps
- `span` creation + enter: ~466 ps
- Empty span (no fields): ~226 ps

That's sub-nanosecond. The interest caching system means a disabled `info!()` call in a hot loop costs almost nothing. The compiler can often eliminate the entire call after inlining the interest check.

**Enabled spans** (with a subscriber):
- Creating a span with 3 fields: ~200-400 ns
- Entering/exiting a span: ~50-100 ns
- Emitting an event with formatting: ~500-1000 ns (depends on field count and formatter)

For comparison, `log` has lower per-event overhead because there's no span tracking or layer dispatch. But `log` gives you no contextual information. The tracing overhead is proportional to the number of active layers and the complexity of field recording.

The practical impact: in a typical web service handling thousands of requests per second, tracing overhead is negligible compared to network I/O and database queries. You'd need to be instrumenting inside a tight computational loop to notice it. And for those cases, `tracing::Level::TRACE` combined with `RUST_LOG` filtering means the instrumentation only activates when you need it.

One thing to watch: the `Registry`'s sharded slab allocates per-shard metadata lazily. Under high concurrency with many short-lived spans, the slab recycles slots efficiently. But if you're creating millions of concurrent spans (not typical), memory usage can spike. This was addressed after [issue #1005](https://github.com/tokio-rs/tracing/issues/1005). For normal server workloads with hundreds or even thousands of concurrent spans, it's not a concern.

## When to reach for tracing vs when it's overkill

Use `tracing` when:
- You're building a service or long-running application
- You have async code (spans track across `.await` points correctly)
- You need structured log output (JSON for log aggregation)
- You want to add OpenTelemetry later without rewriting logging
- Multiple concurrent request streams need context separation

Use `println!`/`eprintln!` when:
- Quick one-off scripts or CLI tools with no concurrent operations
- Prototyping where you'll add proper logging later
- Build scripts or proc macros

Use `log` when:
- Writing a library that should minimize dependency weight
- You genuinely only need severity-level text messages
- Your consumers might not use tracing (though the compatibility bridge largely eliminates this concern)

The tracing ecosystem - `tracing`, `tracing-subscriber`, `tracing-appender`, `tracing-opentelemetry` - forms a coherent stack from development debugging to production observability. The investment in setting it up from the start pays off when you're staring at a dashboard at 3 AM trying to figure out why latency spiked.
