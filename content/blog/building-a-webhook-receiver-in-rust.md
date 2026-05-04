+++
title = "Building a Webhook Receiver in Rust"
date = 2025-02-06
description = "How to build a production-grade webhook receiver with Axum - signature verification, idempotency, async processing, and retry handling."

[taxonomies]
tags = ["rust", "axum", "security", "architecture"]
+++

Someone POSTs JSON to your endpoint. You have maybe 10 seconds before they time out and retry. If you mess up the signature check, you're processing forged payloads. If you do the work synchronously, you'll miss the timeout window. If you don't deduplicate, you'll process the same event three times.

Webhooks sound simple. The failure modes are not.

This post walks through building a webhook receiver in Rust using [Axum](https://docs.rs/axum/0.8.8/axum/). We'll use GitHub webhooks as the concrete example, but the patterns apply to Stripe, Shopify, Slack, or anything else that sends signed HTTP callbacks.

<!-- more -->

## The problem shape

A webhook receiver has to do four things right:

1. **Verify the signature** - prove the payload actually came from the sender
2. **Deduplicate** - handle the same event arriving multiple times
3. **Respond fast** - return 200 before doing real work
4. **Process reliably** - don't lose events even if something crashes

Miss any one of these and you'll have a bad time in production. Let's build them one by one.

## Project setup

```toml
# Cargo.toml
[package]
name = "webhook-receiver"
version = "0.1.0"
edition = "2024"

[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
hmac = "0.12"
sha2 = "0.10"
hex = "0.4"
dashmap = "6"
tracing = "0.1"
tracing-subscriber = "0.3"
```

## Step 1: Accept the raw body

The first thing to get right - and the thing most tutorials get wrong - is body handling. Webhook signature verification needs the **raw bytes** of the request body. If your framework parses the JSON first and then re-serializes it, whitespace and key ordering can change. The signature won't match, and you'll spend hours debugging.

Axum makes this straightforward with the `Bytes` extractor:

```rust
use axum::{
    Router,
    routing::post,
    body::Bytes,
    http::{HeaderMap, StatusCode},
};

async fn webhook_handler(
    headers: HeaderMap,
    body: Bytes,
) -> StatusCode {
    // body is the raw bytes - untouched, unparsed
    // we'll verify signature against these exact bytes
    tracing::info!("received {} bytes", body.len());
    StatusCode::OK
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let app = Router::new()
        .route("/webhooks/github", post(webhook_handler));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    tracing::info!("listening on 0.0.0.0:3000");
    axum::serve(listener, app).await.unwrap();
}
```

Key detail: `Bytes` must be the **last** extractor argument. Axum can only consume the request body once. `HeaderMap` doesn't touch the body, so it goes first. If you tried putting `Json<T>` before `Bytes`, the body would already be consumed.

## Step 2: Signature verification

This is the security-critical part. GitHub signs every webhook payload with HMAC-SHA256 using a shared secret. The signature arrives in the `X-Hub-Signature-256` header, formatted as `sha256=<hex digest>`.

The verification flow:
1. Compute HMAC-SHA256 of the raw body using your secret
2. Compare the result to the signature from the header
3. Use constant-time comparison to prevent [timing attacks](https://en.wikipedia.org/wiki/Timing_attack)

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

fn verify_github_signature(
    secret: &[u8],
    payload: &[u8],
    signature_header: &str,
) -> bool {
    // GitHub sends "sha256=<hex>", strip the prefix
    let hex_signature = match signature_header.strip_prefix("sha256=") {
        Some(sig) => sig,
        None => return false,
    };

    // Decode the hex string to bytes
    let signature_bytes = match hex::decode(hex_signature) {
        Ok(bytes) => bytes,
        Err(_) => return false,
    };

    // Compute the expected HMAC
    let mut mac = HmacSha256::new_from_slice(secret)
        .expect("HMAC accepts any key length");
    mac.update(payload);

    // verify_slice does constant-time comparison internally
    mac.verify_slice(&signature_bytes).is_ok()
}
```

That `verify_slice` call is doing the heavy lifting. Under the hood, the `hmac` crate uses the [`subtle`](https://crates.io/crates/subtle) crate for constant-time equality checks. If you used `==` on the byte slices directly, an attacker could measure response times to incrementally guess the correct signature byte by byte. With constant-time comparison, the check always takes the same amount of time regardless of how many bytes match. The `subtle` crate implements this through the `ConstantTimeEq` trait, which compiles down to bitwise OR accumulation - every byte gets compared even if the first byte already differs.

Never roll your own comparison here. I covered why in the [Build vs Buy](/blog/build-vs-buy---when-to-use-a-library-and-when-to-write-your-/) post - crypto primitives are exactly the kind of thing you should always delegate to a reviewed library.

Now integrate this into the handler:

```rust
use std::sync::Arc;

struct AppState {
    webhook_secret: Vec<u8>,
}

async fn webhook_handler(
    axum::extract::State(state): axum::extract::State<Arc<AppState>>,
    headers: HeaderMap,
    body: Bytes,
) -> StatusCode {
    // Extract the signature header
    let signature = match headers
        .get("x-hub-signature-256")
        .and_then(|v| v.to_str().ok())
    {
        Some(sig) => sig,
        None => {
            tracing::warn!("missing X-Hub-Signature-256 header");
            return StatusCode::UNAUTHORIZED;
        }
    };

    // Verify
    if !verify_github_signature(&state.webhook_secret, &body, signature) {
        tracing::warn!("invalid webhook signature");
        return StatusCode::UNAUTHORIZED;
    }

    tracing::info!("signature verified");
    StatusCode::OK
}
```

### What about other providers?

The pattern is the same everywhere, the header names differ. Stripe uses `Stripe-Signature` with a `t=timestamp,v1=signature` format. Shopify uses `X-Shopify-Hmac-Sha256` with base64 encoding instead of hex. Slack uses `X-Slack-Signature` with `v0=hash` format and includes a timestamp in the signed payload.

You could abstract this behind a trait:

```rust
trait WebhookVerifier: Send + Sync {
    fn verify(&self, headers: &HeaderMap, body: &[u8]) -> bool;
}

struct GitHubVerifier {
    secret: Vec<u8>,
}

impl WebhookVerifier for GitHubVerifier {
    fn verify(&self, headers: &HeaderMap, body: &[u8]) -> bool {
        let sig = headers
            .get("x-hub-signature-256")
            .and_then(|v| v.to_str().ok());

        match sig {
            Some(s) => verify_github_signature(&self.secret, body, s),
            None => false,
        }
    }
}
```

If you read the [Adapter Pattern](/blog/the-adapter-pattern-in-rust---wrapping-external-apis/) post, this should look familiar - same idea of isolating provider-specific details behind a clean interface.

## Step 3: Idempotency

Webhook providers retry on failure. Some retry on success too, if they didn't get your 200 fast enough. GitHub includes an `X-GitHub-Delivery` header with a unique UUID for each delivery. When they retry, the same UUID comes back.

If your handler creates an order, sends an email, or charges a card, processing the same event twice is a real problem. The fix is straightforward: track which delivery IDs you've already handled.

For an in-memory solution that works well for single-instance deployments:

```rust
use dashmap::DashMap;
use std::time::{Duration, Instant};

struct IdempotencyStore {
    seen: DashMap<String, Instant>,
    ttl: Duration,
}

impl IdempotencyStore {
    fn new(ttl: Duration) -> Self {
        Self {
            seen: DashMap::new(),
            ttl,
        }
    }

    /// Returns true if this is a new event, false if duplicate.
    fn check_and_insert(&self, delivery_id: &str) -> bool {
        // Clean up expired entries occasionally
        if self.seen.len() > 10_000 {
            self.seen.retain(|_, inserted| inserted.elapsed() < self.ttl);
        }

        // try_insert returns Err if key exists
        match self.seen.entry(delivery_id.to_string()) {
            dashmap::mapref::entry::Entry::Occupied(entry) => {
                if entry.get().elapsed() > self.ttl {
                    // Expired, treat as new
                    drop(entry);
                    self.seen.insert(delivery_id.to_string(), Instant::now());
                    true
                } else {
                    false
                }
            }
            dashmap::mapref::entry::Entry::Vacant(entry) => {
                entry.insert(Instant::now());
                true
            }
        }
    }
}
```

[`DashMap`](https://docs.rs/dashmap/latest/dashmap/) is a concurrent hash map - it uses sharded locking internally, so multiple threads can read and write without a global lock. Perfect for a hot webhook endpoint.

The TTL is important. GitHub retries for up to 3 days, so a 7-day TTL gives you comfortable margin. For Stripe, the retry window is about 3 days too. After that, you can safely forget the delivery ID.

For multi-instance deployments, swap `DashMap` for Redis or a database table:

```sql
CREATE TABLE processed_webhooks (
    delivery_id TEXT PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Run daily to clean up
DELETE FROM processed_webhooks
WHERE processed_at < datetime('now', '-7 days');
```

The important thing is the check-and-insert must be **atomic**. With a database, use `INSERT ... ON CONFLICT DO NOTHING` and check the affected rows. With Redis, use `SET NX` with an expiry. A race between two concurrent deliveries of the same event should result in exactly one of them being processed.

## Step 4: Async processing

Here's where most webhook receivers fail under load. If your handler parses the payload, hits a database, calls three APIs, and sends an email - all before returning 200 - you're going to miss timeout windows. GitHub gives you 10 seconds. Stripe gives you 20. When they don't get a timely response, they retry, and now you're processing the same event multiple times (hello, idempotency).

The pattern: acknowledge immediately, process in the background.

If you went through the [Understanding Tokio](/blog/understanding-tokio) post, you know `tokio::sync::mpsc` channels. We'll use a bounded channel as a work queue:

```rust
use tokio::sync::mpsc;

#[derive(Debug)]
struct WebhookEvent {
    delivery_id: String,
    event_type: String,
    payload: Bytes,
}

fn spawn_webhook_processor(
    mut rx: mpsc::Receiver<WebhookEvent>,
) {
    tokio::spawn(async move {
        while let Some(event) = rx.recv().await {
            tracing::info!(
                delivery_id = %event.delivery_id,
                event_type = %event.event_type,
                "processing webhook"
            );

            if let Err(e) = process_event(&event).await {
                tracing::error!(
                    delivery_id = %event.delivery_id,
                    error = %e,
                    "failed to process webhook"
                );
                // In production: push to a dead letter queue
                // or a retry table with exponential backoff
            }
        }
    });
}

async fn process_event(event: &WebhookEvent) -> Result<(), Box<dyn std::error::Error>> {
    let payload: serde_json::Value = serde_json::from_slice(&event.payload)?;

    match event.event_type.as_str() {
        "push" => {
            let repo = payload["repository"]["full_name"]
                .as_str()
                .unwrap_or("unknown");
            tracing::info!(repo, "push event received");
            // trigger a build, update a cache, etc.
        }
        "pull_request" => {
            let action = payload["action"].as_str().unwrap_or("unknown");
            let number = payload["number"].as_u64().unwrap_or(0);
            tracing::info!(action, number, "pull request event");
        }
        _ => {
            tracing::debug!(
                event_type = %event.event_type,
                "unhandled event type, skipping"
            );
        }
    }

    Ok(())
}
```

Why a bounded channel and not just `tokio::spawn` per event? Backpressure. If webhooks arrive faster than you can process them, an unbounded approach just eats memory until the OOM killer visits. A bounded channel with, say, capacity 1000 means the 1001st send will await - which slows down the handler response - which triggers retries - which is fine because your idempotency layer catches the duplicates.

With the `mpsc` approach you also get a natural serialization point. If event ordering matters (it sometimes does - like PR opened before PR merged), a single consumer processes them in order. If throughput matters more, spawn multiple consumers each pulling from the same receiver (wrap it in `Arc<Mutex<Receiver>>` or use a proper work-stealing setup).

## Putting it all together

Here's the complete application with all four pieces wired up:

```rust
use axum::{
    Router,
    body::Bytes,
    extract::State,
    http::{HeaderMap, StatusCode},
    routing::post,
};
use hmac::{Hmac, Mac};
use sha2::Sha256;
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::mpsc;

type HmacSha256 = Hmac<Sha256>;

// --- Signature Verification ---

fn verify_github_signature(
    secret: &[u8],
    payload: &[u8],
    signature_header: &str,
) -> bool {
    let hex_signature = match signature_header.strip_prefix("sha256=") {
        Some(sig) => sig,
        None => return false,
    };

    let signature_bytes = match hex::decode(hex_signature) {
        Ok(bytes) => bytes,
        Err(_) => return false,
    };

    let mut mac = HmacSha256::new_from_slice(secret)
        .expect("HMAC accepts any key length");
    mac.update(payload);
    mac.verify_slice(&signature_bytes).is_ok()
}

// --- Idempotency ---

use dashmap::DashMap;
use std::time::Instant;

struct IdempotencyStore {
    seen: DashMap<String, Instant>,
    ttl: Duration,
}

impl IdempotencyStore {
    fn new(ttl: Duration) -> Self {
        Self {
            seen: DashMap::new(),
            ttl,
        }
    }

    fn check_and_insert(&self, delivery_id: &str) -> bool {
        match self.seen.entry(delivery_id.to_string()) {
            dashmap::mapref::entry::Entry::Occupied(entry) => {
                if entry.get().elapsed() > self.ttl {
                    drop(entry);
                    self.seen.insert(delivery_id.to_string(), Instant::now());
                    true
                } else {
                    false
                }
            }
            dashmap::mapref::entry::Entry::Vacant(entry) => {
                entry.insert(Instant::now());
                true
            }
        }
    }
}

// --- Async Processing ---

#[derive(Debug)]
struct WebhookEvent {
    delivery_id: String,
    event_type: String,
    payload: Bytes,
}

fn spawn_webhook_processor(mut rx: mpsc::Receiver<WebhookEvent>) {
    tokio::spawn(async move {
        while let Some(event) = rx.recv().await {
            tracing::info!(
                delivery_id = %event.delivery_id,
                event_type = %event.event_type,
                "processing webhook"
            );

            if let Err(e) = process_event(&event).await {
                tracing::error!(
                    delivery_id = %event.delivery_id,
                    error = %e,
                    "failed to process webhook"
                );
            }
        }
    });
}

async fn process_event(
    event: &WebhookEvent,
) -> Result<(), Box<dyn std::error::Error>> {
    let payload: serde_json::Value = serde_json::from_slice(&event.payload)?;

    match event.event_type.as_str() {
        "push" => {
            let repo = payload["repository"]["full_name"]
                .as_str()
                .unwrap_or("unknown");
            tracing::info!(repo, "push event received");
        }
        "pull_request" => {
            let action = payload["action"].as_str().unwrap_or("unknown");
            let number = payload["number"].as_u64().unwrap_or(0);
            tracing::info!(action, number, "pull request event");
        }
        other => {
            tracing::debug!(event_type = other, "unhandled event type");
        }
    }

    Ok(())
}

// --- Application ---

struct AppState {
    webhook_secret: Vec<u8>,
    idempotency: IdempotencyStore,
    event_tx: mpsc::Sender<WebhookEvent>,
}

async fn webhook_handler(
    State(state): State<Arc<AppState>>,
    headers: HeaderMap,
    body: Bytes,
) -> StatusCode {
    // 1. Verify signature
    let signature = match headers
        .get("x-hub-signature-256")
        .and_then(|v| v.to_str().ok())
    {
        Some(sig) => sig,
        None => return StatusCode::UNAUTHORIZED,
    };

    if !verify_github_signature(&state.webhook_secret, &body, signature) {
        tracing::warn!("invalid webhook signature");
        return StatusCode::UNAUTHORIZED;
    }

    // 2. Check idempotency
    let delivery_id = headers
        .get("x-github-delivery")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();

    if !state.idempotency.check_and_insert(&delivery_id) {
        tracing::info!(delivery_id, "duplicate delivery, skipping");
        return StatusCode::OK;
    }

    // 3. Extract event type
    let event_type = headers
        .get("x-github-event")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();

    // 4. Queue for async processing, respond immediately
    let event = WebhookEvent {
        delivery_id: delivery_id.clone(),
        event_type,
        payload: body,
    };

    match state.event_tx.try_send(event) {
        Ok(()) => StatusCode::OK,
        Err(mpsc::error::TrySendError::Full(_)) => {
            tracing::error!("webhook processing queue full");
            // 503 tells the sender to retry later
            StatusCode::SERVICE_UNAVAILABLE
        }
        Err(mpsc::error::TrySendError::Closed(_)) => {
            tracing::error!("webhook processor shut down");
            StatusCode::INTERNAL_SERVER_ERROR
        }
    }
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let secret = std::env::var("WEBHOOK_SECRET")
        .expect("WEBHOOK_SECRET must be set");

    let (tx, rx) = mpsc::channel::<WebhookEvent>(1000);

    spawn_webhook_processor(rx);

    let state = Arc::new(AppState {
        webhook_secret: secret.into_bytes(),
        idempotency: IdempotencyStore::new(Duration::from_secs(7 * 24 * 3600)),
        event_tx: tx,
    });

    let app = Router::new()
        .route("/webhooks/github", post(webhook_handler))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    tracing::info!("listening on 0.0.0.0:3000");
    axum::serve(listener, app).await.unwrap();
}
```

Notice `try_send` instead of `send` in the handler. `send` is async and will wait if the channel is full. That defeats the purpose - we want to respond immediately. `try_send` returns an error if the buffer is full, and we translate that to a 503 Service Unavailable, which tells GitHub to back off and retry.

## Testing locally

GitHub has a nice feature for this. Go to your repo's Settings > Webhooks, pick a webhook, and hit "Redeliver" on any past delivery. But for rapid iteration, curl works:

```bash
# Generate a test signature
SECRET="your-secret"
PAYLOAD='{"action":"opened","number":1}'
SIGNATURE="sha256=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$SECRET" | cut -d' ' -f2)"

curl -X POST http://localhost:3000/webhooks/github \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: $SIGNATURE" \
  -H "X-GitHub-Delivery: $(uuidgen)" \
  -H "X-GitHub-Event: pull_request" \
  -d "$PAYLOAD"
```

Send the same delivery ID twice to verify the idempotency layer. Second request should get a 200 but no processing log.

If you want to test under realistic conditions, use [ngrok](https://ngrok.com/) to expose your local server and point a real GitHub webhook at it. Much faster feedback loop than deploying to a staging environment. And if you want to validate throughput, the tools from the [Load Testing Your Rust API](/blog/load-testing-your-rust-api---tools-and-methodology/) post work great here - `wrk` with a Lua script that generates valid signatures can simulate realistic webhook floods.

## Production considerations

### Dead letter queue

The `process_event` function above just logs errors. In production, you need a dead letter queue. When processing fails, write the event to a database table or a file:

```rust
async fn handle_failed_event(event: &WebhookEvent, error: &str) {
    // Insert into a dead_letter_queue table
    // with the full payload, error message, and a retry_count
    // A background job picks these up and retries with exponential backoff
    tracing::error!(
        delivery_id = %event.delivery_id,
        error,
        "moved to dead letter queue"
    );
}
```

### Graceful shutdown

When your process gets SIGTERM, you need to finish processing in-flight events. Dropping the `Sender` half of the channel signals the processor to drain and stop:

```rust
// In main, after axum::serve returns:
drop(state); // drops the Sender
// The processor loop exits when rx.recv() returns None
// Give it a few seconds to finish
tokio::time::sleep(Duration::from_secs(5)).await;
```

A cleaner approach uses `tokio::signal` and a `CancellationToken`, but the drop-the-sender pattern works for simple cases.

### Multiple event types, different processing speeds

If push events take 50ms but deployment events take 30 seconds (because they trigger a full CI pipeline), a single processing queue means fast events get stuck behind slow ones. Split into multiple channels by event type, or use a proper task queue like [SQS](https://aws.amazon.com/sqs/) or [Redis streams](https://redis.io/docs/latest/develop/data-types/streams/).

### IP allowlisting

GitHub [publishes their webhook IP ranges](https://api.github.com/meta) in the `hooks` field. You can add a middleware layer that rejects requests from outside those ranges. Defense in depth - signature verification is your primary security, IP filtering is the bonus layer.

### Content length limits

A rogue (or compromised) webhook sender could POST a 2GB body and exhaust your memory. Axum has `DefaultBodyLimit` middleware:

```rust
use axum::extract::DefaultBodyLimit;

let app = Router::new()
    .route("/webhooks/github", post(webhook_handler))
    .layer(DefaultBodyLimit::max(5 * 1024 * 1024)) // 5MB
    .with_state(state);
```

GitHub's largest payloads (push events on repos with many commits) are typically under 1MB. 5MB gives generous headroom.

## What happens at the syscall level

When a webhook POST arrives, here's the actual path through the system:

1. `epoll_wait` wakes up a tokio worker thread (the I/O driver I talked about in the [Understanding Tokio](/blog/understanding-tokio) post)
2. tokio's scheduler picks up the connection task
3. hyper (axum's HTTP layer) reads the request via `read(fd, buf, len)` syscalls
4. Our handler runs - signature verification is pure CPU work (the HMAC computation), no syscalls needed
5. `try_send` on the mpsc channel is just a mutex lock + memory write - no syscall
6. hyper writes the "200 OK" response via `write(fd, buf, len)`
7. The webhook processor task gets scheduled by tokio's work-stealing scheduler - the I/O driver doesn't need to wake anything, it's already in the run queue

Total syscall count for a verified, deduplicated, queued webhook: roughly 2 (one read, one write). The signature verification, idempotency check, and queue insertion all happen in userspace. This is why Rust webhook receivers handle absurd throughput on modest hardware.

You can verify this yourself with `strace`:

```bash
strace -e trace=read,write,epoll_wait -p $(pidof webhook-receiver)
```

## Security checklist

Before you ship this to production:

- [ ] Webhook secret is loaded from environment, not hardcoded
- [ ] Signature verification uses constant-time comparison (the `hmac` crate handles this)
- [ ] Raw body is used for verification, not re-serialized JSON
- [ ] Missing or invalid signature returns 401, not 500
- [ ] Body size is limited via `DefaultBodyLimit`
- [ ] HTTPS only (terminate TLS at your load balancer or reverse proxy)
- [ ] Delivery IDs are tracked to prevent replay after the TTL expires
- [ ] Failed events go to a dead letter queue, not into the void

The signature verification is non-negotiable. Without it, anyone who discovers your webhook URL can forge events. With GitHub specifically, the `X-Hub-Signature-256` header is only present if you configured a secret. If you didn't set one - fix that first.

## Wrapping up

The building blocks are simple: `Bytes` for raw body access, `hmac` + `sha2` for signature verification, `DashMap` for concurrent idempotency tracking, and `mpsc` for async processing. What makes a webhook receiver production-grade isn't any single piece - it's getting all four right together.

The full code from this post compiles and runs as-is. Clone it, set `WEBHOOK_SECRET`, point a GitHub webhook at it with ngrok, and watch events flow through. Then break things on purpose - send bad signatures, duplicate delivery IDs, flood it with requests - and verify each layer does its job.
