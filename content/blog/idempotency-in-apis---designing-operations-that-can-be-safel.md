+++
title = "Idempotency in APIs - designing operations that can be safely retried"
date = 2025-10-14
description = "How to make your API endpoints safe to retry - idempotency keys, database constraints, Stripe's approach, and a practical Axum implementation."

[taxonomies]
tags = ["api", "architecture", "rust", "axum"]
+++

You send a POST request to create a payment. The connection drops. Did the server process it? You don't know. You retry. Now there might be two payments. Congratulations, you've just charged someone twice.

This failure mode is so common in distributed systems that there's a formal property for handling it: idempotency. An operation is idempotent if calling it multiple times produces the same result as calling it once. Get this right, and clients can safely retry anything without fear of duplicate side effects. Get it wrong, and every network hiccup becomes a customer support ticket.

I touched on idempotency briefly in the [webhook receiver post](/blog/building-a-webhook-receiver-in-rust-signatures-idempotency-and-async-processing/) - using a `DashMap` to deduplicate delivery IDs. That was enough for incoming webhooks where the provider controls the unique ID. When you're the one designing the API, the problem is deeper. You need to decide how clients identify retries, where the deduplication state lives, and what happens during the gap between "request received" and "response sent."

<!-- more -->

## What idempotent actually means

The word comes from mathematics. A function *f* is idempotent if *f(f(x)) = f(x)*. Applying it twice is the same as applying it once. In API terms: making the same request twice has the same observable effect as making it once.

"Same observable effect" is doing a lot of work in that sentence. It doesn't mean the server does nothing on the second call. It might hit the database again, run the same query, check some state. What matters is the outcome - no duplicate records, no double charges, no repeated side effects.

There's a subtle distinction between idempotency and being a no-op on retry. A truly idempotent endpoint might re-execute some logic on the second call and still arrive at the same state. Or it might recognize the duplicate and return a cached result. Both approaches are valid - the only requirement is that the world looks the same whether the client called once or ten times.

## HTTP methods and idempotency

The HTTP spec (RFC 9110, [Section 9.2.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2)) defines which methods are idempotent:

| Method | Idempotent | Safe | Why |
|--------|-----------|------|-----|
| GET | Yes | Yes | Read-only, no state change |
| HEAD | Yes | Yes | Same as GET, no body |
| PUT | Yes | No | Replaces the entire resource |
| DELETE | Yes | No | Deleting what's already gone is fine |
| POST | **No** | No | Creates new resources, triggers side effects |
| PATCH | **No** | No | Depends on the patch format |

GET and HEAD are trivially idempotent - they don't change anything. PUT is idempotent because it's a full replacement: "set this resource to exactly this state." Calling `PUT /users/42 {"name": "Alice"}` ten times still leaves you with one user named Alice.

DELETE is idempotent because deleting something that doesn't exist is a no-op. The first call returns 200, the second returns 404, but the server state is identical after both. Some teams return 204 on both to make the idempotency more visible to clients - either approach works.

POST is the problem child. `POST /orders` means "create a new order." Call it twice, you get two orders. The HTTP spec explicitly marks POST as non-idempotent because the server can't distinguish a retry from a genuinely new request.

PATCH sits in a grey area. A JSON Merge Patch (`{"name": "Alice"}`) is idempotent - applying it twice yields the same state. A JSON Patch with `{"op": "add", "path": "/tags/-", "value": "urgent"}` is not - it appends to an array, so calling it twice adds two entries. The spec says PATCH is not idempotent because it depends on the patch format.

## The problem: making POST safe to retry

Here's the timeline that breaks things:

```
Client                          Server
  |                               |
  |--- POST /payments ----------->|
  |                               |--- Create payment row
  |                               |--- Charge credit card
  |         (connection drops)    |--- Return 201
  |                               |
  |--- POST /payments ----------->|  (retry)
  |                               |--- Create ANOTHER payment row
  |                               |--- Charge credit card AGAIN
  |<------------ 201 -------------|
```

The client never received the first response, so it retries. The server has no way to know this is a retry. Both requests look identical - same body, same headers, same authentication. The server dutifully creates a second payment.

This is where idempotency keys come in.

## Idempotency keys

The pattern: the client generates a unique identifier (typically a UUID v4) and sends it with the request in a header. The server uses this key to recognize retries.

```
POST /v1/payments HTTP/1.1
Content-Type: application/json
Idempotency-Key: 7c4b8e92-3a1f-4d2e-9c8b-5f6a7d8e9f0a

{"amount": 2000, "currency": "usd", "customer": "cus_abc123"}
```

First time the server sees this key: process the request normally, store the key alongside the response. Second time: look up the key, return the stored response. The client can retry as many times as it wants - the payment happens exactly once.

This is exactly what Stripe does. Every mutating request to their API accepts an `Idempotency-Key` header. If you're using their SDK, it generates one automatically on retries with exponential backoff. Their implementation stores the key for 24 hours in API v1 (extended to 30 days in API v2), compares the request parameters on replay to make sure you're not reusing a key with different data, and returns 409 Conflict if two concurrent requests arrive with the same key while the first is still processing.

The concept is being standardized. [IETF draft-ietf-httpapi-idempotency-key-header](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) (currently at revision 07) defines the `Idempotency-Key` header as a structured header field. It specifies that servers should return:
- **400** if the header is required but missing
- **422** if the key is reused with different request parameters
- **409** if a concurrent request with the same key is still being processed

Whether or not you follow the draft exactly, these are sensible semantics. If you covered the [Designing RESTful APIs](/blog/designing-restful-apis-practical-guidelines-beyond-the-theory/) post, these error responses pair naturally with the RFC 9457 error format discussed there.

## Database-level idempotency

The simplest form of idempotency doesn't need a separate key at all - it uses the natural constraints of your data model.

Consider a "follow user" endpoint:

```sql
CREATE TABLE follows (
    follower_id TEXT NOT NULL,
    followed_id TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (follower_id, followed_id)
);
```

With that composite primary key, `INSERT INTO follows (follower_id, followed_id) VALUES ('alice', 'bob')` succeeds the first time and fails with a unique violation on retry. If you use `INSERT ... ON CONFLICT DO NOTHING`, the operation becomes truly idempotent - the second call is a no-op that returns success.

```sql
INSERT INTO follows (follower_id, followed_id)
VALUES ('alice', 'bob')
ON CONFLICT (follower_id, followed_id) DO NOTHING;
```

This works for any operation where the request naturally contains a unique identifier. "Add product X to cart Y" - unique on (cart_id, product_id). "Assign role R to user U" - unique on (user_id, role_id). If your data model already prevents duplicates, you get idempotency for free.

Where this breaks down: creating genuinely new resources where the ID is server-generated. `POST /orders` with `{"items": [...]}` doesn't have a natural unique key - the client doesn't know the order ID yet. That's when you need explicit idempotency keys.

## Application-level idempotency: the full pattern

For operations without natural deduplication, you need an idempotency key store. Here's the database schema, inspired by [Brandur Leach's excellent writeup](https://brandur.org/idempotency-keys) on implementing Stripe-like idempotency in Postgres:

```sql
CREATE TABLE idempotency_keys (
    key            TEXT        NOT NULL,
    user_id        TEXT        NOT NULL,
    request_path   TEXT        NOT NULL,
    request_body   JSONB       NOT NULL,
    response_code  INT         NULL,
    response_body  JSONB       NULL,
    locked_at      TIMESTAMPTZ NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, key)
);
```

The composite primary key `(user_id, key)` is critical. Without `user_id`, one user's idempotency key could collide with another's. The `locked_at` column prevents concurrent processing of the same key. The `response_code` and `response_body` columns cache the result.

The request lifecycle looks like this:

```
1. Receive request with Idempotency-Key header
2. Try INSERT into idempotency_keys
   - If conflict (key exists):
     a. If response_code is NOT NULL -> return cached response
     b. If locked_at is recent -> return 409 (in progress)
     c. If locked_at is stale -> take over the lock (previous attempt crashed)
   - If inserted (new key):
     a. Set locked_at = NOW()
3. Process the actual request
4. UPDATE idempotency_keys SET response_code = ..., response_body = ..., locked_at = NULL
5. Return response
```

Step 2 must be atomic. In Postgres, the `INSERT ... ON CONFLICT` handles this in a single statement. In SQLite, `INSERT OR IGNORE` plus checking the affected row count works. The database's own concurrency control prevents two threads from both thinking they're the first.

The stale lock check (step 2c) handles the case where a previous attempt crashed mid-processing. If `locked_at` is older than your timeout (say, 30 seconds), it's safe to assume the previous attempt is dead. Take the lock and re-process.

## Building it in Axum

Let's build a complete idempotency middleware for [Axum](https://docs.rs/axum/0.8.8/axum/). We'll use SQLite through `rusqlite` to keep the example self-contained, but the pattern translates to any database.

```toml
# Cargo.toml
[package]
name = "idempotent-api"
version = "0.1.0"
edition = "2024"

[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rusqlite = { version = "0.34", features = ["bundled"] }
uuid = { version = "1", features = ["v4"] }
tracing = "0.1"
tracing-subscriber = "0.3"
tower = "0.5"
http = "1"
http-body-util = "0.1"
bytes = "1"
```

First, the idempotency store:

```rust
use rusqlite::Connection;
use std::sync::Mutex;
use std::time::{SystemTime, UNIX_EPOCH};

pub struct IdempotencyStore {
    conn: Mutex<Connection>,
    lock_timeout_secs: u64,
}

#[derive(Debug, Clone)]
pub struct CachedResponse {
    pub status_code: u16,
    pub body: String,
}

impl IdempotencyStore {
    pub fn new(path: &str) -> Self {
        let conn = Connection::open(path).expect("failed to open db");

        conn.execute_batch(
            "CREATE TABLE IF NOT EXISTS idempotency_keys (
                key           TEXT NOT NULL,
                user_id       TEXT NOT NULL,
                request_path  TEXT NOT NULL,
                request_body  TEXT NOT NULL,
                response_code INTEGER,
                response_body TEXT,
                locked_at     INTEGER,
                created_at    INTEGER NOT NULL,
                PRIMARY KEY (user_id, key)
            );

            -- Clean up keys older than 24 hours on each startup
            DELETE FROM idempotency_keys
            WHERE created_at < unixepoch() - 86400;",
        )
        .expect("failed to create table");

        Self {
            conn: Mutex::new(conn),
            lock_timeout_secs: 30,
        }
    }

    fn now_unix(&self) -> u64 {
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs()
    }

    /// Try to acquire the idempotency key.
    /// Returns Ok(None) if we acquired the lock (proceed with processing).
    /// Returns Ok(Some(response)) if there's a cached response.
    /// Returns Err for conflicts or mismatches.
    pub fn try_acquire(
        &self,
        user_id: &str,
        key: &str,
        request_path: &str,
        request_body: &str,
    ) -> Result<Option<CachedResponse>, IdempotencyError> {
        let conn = self.conn.lock().unwrap();
        let now = self.now_unix();

        // Check if key already exists
        let existing = conn
            .query_row(
                "SELECT request_path, request_body, response_code,
                        response_body, locked_at
                 FROM idempotency_keys
                 WHERE user_id = ?1 AND key = ?2",
                rusqlite::params![user_id, key],
                |row| {
                    Ok((
                        row.get::<_, String>(0)?,
                        row.get::<_, String>(1)?,
                        row.get::<_, Option<u16>>(2)?,
                        row.get::<_, Option<String>>(3)?,
                        row.get::<_, Option<u64>>(4)?,
                    ))
                },
            )
            .ok();

        match existing {
            Some((orig_path, orig_body, resp_code, resp_body, locked_at)) => {
                // Key exists - validate request matches
                if orig_path != request_path || orig_body != request_body {
                    return Err(IdempotencyError::RequestMismatch);
                }

                // Already completed - return cached response
                if let (Some(code), Some(body)) = (resp_code, resp_body) {
                    return Ok(Some(CachedResponse {
                        status_code: code,
                        body,
                    }));
                }

                // Still in progress - check if lock is stale
                if let Some(lock_time) = locked_at {
                    if now - lock_time < self.lock_timeout_secs {
                        return Err(IdempotencyError::InProgress);
                    }
                }

                // Stale lock - take over
                conn.execute(
                    "UPDATE idempotency_keys
                     SET locked_at = ?1
                     WHERE user_id = ?2 AND key = ?3",
                    rusqlite::params![now, user_id, key],
                )
                .unwrap();

                Ok(None)
            }
            None => {
                // New key - insert and acquire lock
                conn.execute(
                    "INSERT INTO idempotency_keys
                        (key, user_id, request_path, request_body,
                         locked_at, created_at)
                     VALUES (?1, ?2, ?3, ?4, ?5, ?5)",
                    rusqlite::params![key, user_id, request_path, request_body, now],
                )
                .unwrap();

                Ok(None)
            }
        }
    }

    /// Store the response after processing completes.
    pub fn complete(
        &self,
        user_id: &str,
        key: &str,
        status_code: u16,
        response_body: &str,
    ) {
        let conn = self.conn.lock().unwrap();
        conn.execute(
            "UPDATE idempotency_keys
             SET response_code = ?1, response_body = ?2, locked_at = NULL
             WHERE user_id = ?3 AND key = ?4",
            rusqlite::params![status_code, response_body, user_id, key],
        )
        .unwrap();
    }

    /// Release the lock without storing a response (on processing failure).
    pub fn release(&self, user_id: &str, key: &str) {
        let conn = self.conn.lock().unwrap();
        conn.execute(
            "DELETE FROM idempotency_keys
             WHERE user_id = ?1 AND key = ?2 AND response_code IS NULL",
            rusqlite::params![user_id, key],
        )
        .unwrap();
    }
}

#[derive(Debug)]
pub enum IdempotencyError {
    RequestMismatch,
    InProgress,
}
```

A few things to note about this store. The `Mutex<Connection>` works for SQLite (which is single-writer anyway) - if you're using Postgres with a connection pool, each operation would grab a connection from the pool and use `INSERT ... ON CONFLICT` with row-level locking instead. The `release` method deletes the key entirely on failure rather than leaving a locked row, which lets the client retry cleanly. And the stale lock timeout of 30 seconds assumes your handlers don't take longer than that - adjust based on your actual processing time.

Now, the Axum middleware:

```rust
use axum::{
    Router,
    body::Bytes,
    extract::State,
    http::{HeaderMap, Request, StatusCode},
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::post,
};
use std::sync::Arc;

struct AppState {
    idempotency: IdempotencyStore,
}

async fn idempotency_middleware(
    State(state): State<Arc<AppState>>,
    headers: HeaderMap,
    request: Request<axum::body::Body>,
    next: Next,
) -> Response {
    // Only apply to POST requests
    if request.method() != http::Method::POST {
        return next.run(request).await;
    }

    // Extract the idempotency key
    let idem_key = match headers.get("idempotency-key").and_then(|v| v.to_str().ok()) {
        Some(key) => key.to_string(),
        None => {
            // Key is required for POST - return 400
            return (
                StatusCode::BAD_REQUEST,
                serde_json::json!({
                    "type": "https://api.example.com/errors/missing-idempotency-key",
                    "title": "Missing Idempotency-Key header",
                    "status": 400,
                    "detail": "POST requests require an Idempotency-Key header."
                })
                .to_string(),
            )
                .into_response();
        }
    };

    // For a real app, extract user_id from auth token.
    // Hardcoded here for clarity.
    let user_id = headers
        .get("x-user-id")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("anonymous")
        .to_string();

    let request_path = request.uri().path().to_string();

    // Buffer the body so we can read it for deduplication
    // and still pass it to the handler
    let (parts, body) = request.into_parts();
    let body_bytes = match axum::body::to_bytes(body, 1_048_576).await {
        Ok(b) => b,
        Err(_) => return StatusCode::PAYLOAD_TOO_LARGE.into_response(),
    };
    let body_str = String::from_utf8_lossy(&body_bytes).to_string();

    // Try to acquire the idempotency key
    match state
        .idempotency
        .try_acquire(&user_id, &idem_key, &request_path, &body_str)
    {
        Ok(Some(cached)) => {
            // Return cached response
            let status = StatusCode::from_u16(cached.status_code)
                .unwrap_or(StatusCode::INTERNAL_SERVER_ERROR);
            let mut response = (status, cached.body).into_response();
            response
                .headers_mut()
                .insert("idempotency-replayed", "true".parse().unwrap());
            response
        }
        Ok(None) => {
            // Acquired lock - process the request
            let request = Request::from_parts(parts, axum::body::Body::from(body_bytes));
            let response = next.run(request).await;

            // Extract the response to cache it
            let status = response.status();
            let (resp_parts, resp_body) = response.into_parts();
            let resp_bytes = axum::body::to_bytes(resp_body, 10_485_760)
                .await
                .unwrap_or_default();
            let resp_str = String::from_utf8_lossy(&resp_bytes).to_string();

            // Only cache successful responses
            if status.is_success() || status.is_client_error() {
                state
                    .idempotency
                    .complete(&user_id, &idem_key, status.as_u16(), &resp_str);
            } else {
                // Server error - release the lock so the client can retry
                state.idempotency.release(&user_id, &idem_key);
            }

            Response::from_parts(resp_parts, axum::body::Body::from(resp_bytes))
        }
        Err(IdempotencyError::RequestMismatch) => (
            StatusCode::UNPROCESSABLE_ENTITY,
            serde_json::json!({
                "type": "https://api.example.com/errors/idempotency-key-reuse",
                "title": "Idempotency key reused with different request",
                "status": 422,
                "detail": "This idempotency key was already used with different parameters."
            })
            .to_string(),
        )
            .into_response(),
        Err(IdempotencyError::InProgress) => (
            StatusCode::CONFLICT,
            serde_json::json!({
                "type": "https://api.example.com/errors/request-in-progress",
                "title": "Request already in progress",
                "status": 409,
                "detail": "A request with this idempotency key is currently being processed."
            })
            .to_string(),
        )
            .into_response(),
    }
}
```

The middleware only triggers for POST requests. GET, PUT, and DELETE are already idempotent by design (assuming your PUT handlers are true replacements and your DELETE handlers tolerate missing resources). The `idempotency-replayed: true` header on cached responses is a nice touch from the `axum-idempotent` [crate](https://docs.rs/axum-idempotent/latest/axum_idempotent/) - it tells the client "this was a cached replay, not fresh processing."

One detail worth calling out: the middleware caches client errors (4xx) but releases the lock on server errors (5xx). Caching a 422 Validation Error makes sense - sending the same invalid data again should get the same rejection. Caching a 500 would be wrong - that's a transient failure, and the client should be able to retry.

Wire it all up:

```rust
#[derive(serde::Deserialize)]
struct CreatePayment {
    amount: i64,
    currency: String,
    customer: String,
}

#[derive(serde::Serialize)]
struct Payment {
    id: String,
    amount: i64,
    currency: String,
    customer: String,
    status: String,
}

async fn create_payment(
    axum::Json(input): axum::Json<CreatePayment>,
) -> impl IntoResponse {
    // Simulate processing
    let payment = Payment {
        id: uuid::Uuid::new_v4().to_string(),
        amount: input.amount,
        currency: input.currency,
        customer: input.customer,
        status: "succeeded".to_string(),
    };

    (StatusCode::CREATED, axum::Json(payment))
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let state = Arc::new(AppState {
        idempotency: IdempotencyStore::new("idempotency.db"),
    });

    let app = Router::new()
        .route("/v1/payments", post(create_payment))
        .layer(middleware::from_fn_with_state(
            state.clone(),
            idempotency_middleware,
        ))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    tracing::info!("listening on 0.0.0.0:3000");
    axum::serve(listener, app).await.unwrap();
}
```

Test it:

```bash
# First request - processes normally
curl -s -D- -X POST http://localhost:3000/v1/payments \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: test-key-001" \
  -H "X-User-Id: user_42" \
  -d '{"amount": 2000, "currency": "usd", "customer": "cus_abc"}'

# HTTP/1.1 201 Created
# {"id":"a1b2c3...","amount":2000,"currency":"usd",...}

# Second request - same key, returns cached response
curl -s -D- -X POST http://localhost:3000/v1/payments \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: test-key-001" \
  -H "X-User-Id: user_42" \
  -d '{"amount": 2000, "currency": "usd", "customer": "cus_abc"}'

# HTTP/1.1 201 Created
# idempotency-replayed: true
# {"id":"a1b2c3...","amount":2000,"currency":"usd",...}
```

Same response, same payment ID, `idempotency-replayed: true` header present. The second request never hit the handler.

## The gap problem

There's a subtle issue with this pattern that catches people off guard. Between acquiring the lock and completing the response, there's a window where the operation has side effects but no cached response:

```
1. Insert idempotency key, set locked_at     -- lock acquired
2. Charge credit card via Stripe API          -- side effect happens
3. Insert payment row into database           -- local state updated
   (crash here)
4. UPDATE idempotency_keys SET response_code  -- never reached
```

If the server crashes at step 3, the credit card is charged but there's no cached response. The client retries, the stale lock check kicks in, and the handler runs again - charging the card a second time.

Stripe solves this by making their own API idempotent (Stripe charge calls accept their own idempotency keys). So even if your handler retries the Stripe call, Stripe deduplicates it on their end. This is idempotency all the way down.

For operations where the downstream service doesn't support idempotency keys, Brandur's [atomic phases](https://brandur.org/idempotency-keys) pattern helps. Break the operation into stages, each wrapped in a database transaction with a recovery point:

```rust
// Pseudocode for the atomic phases approach
loop {
    match current_recovery_point {
        RecoveryPoint::Started => {
            // Transaction: create the order row, advance recovery point
            db.transaction(|tx| {
                tx.insert_order(&order)?;
                tx.update_recovery_point(key, RecoveryPoint::OrderCreated)?;
                Ok(())
            })?;
        }
        RecoveryPoint::OrderCreated => {
            // External call: charge the card (with its own idempotency key)
            let charge = stripe::charge(&params, &format!("order-{}", key))?;
            // Transaction: record the charge, advance recovery point
            db.transaction(|tx| {
                tx.update_order_charge(&order_id, &charge.id)?;
                tx.update_recovery_point(key, RecoveryPoint::Charged)?;
                Ok(())
            })?;
        }
        RecoveryPoint::Charged => {
            // Transaction: finalize, store response, clear lock
            db.transaction(|tx| {
                tx.store_response(key, 201, &response_json)?;
                tx.update_recovery_point(key, RecoveryPoint::Finished)?;
                Ok(())
            })?;
        }
        RecoveryPoint::Finished => break,
    }
}
```

Each phase is a transaction between external calls. If the process crashes, a retry picks up from the last completed recovery point instead of starting over. The external calls use their own idempotency keys, so repeating them is safe.

This is heavier machinery than most APIs need. If your endpoints just do local database work with no external side effects, the simple lock-process-cache pattern is enough.

## When you don't need idempotency keys

Not every POST endpoint needs this treatment. Some operations are naturally idempotent without any extra work:

**Upserts.** `POST /settings` that creates or updates a user's settings based on the user ID. The database `INSERT ... ON CONFLICT UPDATE` handles deduplication. No key needed.

**Status transitions with guards.** "Cancel order 123" is idempotent if your handler checks the current status first: already canceled? Return success. Already shipped? Return error. The handler's own logic prevents double-processing.

**Append-only with natural keys.** "Log this event with trace ID X" where trace_id has a unique constraint. Duplicates bounce off the constraint.

Save the idempotency key machinery for operations that are genuinely non-idempotent: creating resources with server-generated IDs, triggering external side effects (emails, payments, notifications), or any multi-step process where partial completion is possible.

## Production checklist

If you're adding idempotency keys to your API:

- **Key format.** Accept any string up to 255 characters. Recommend UUID v4 in your docs but don't enforce it - some clients use their own schemes.
- **Key scope.** Always scope keys to the authenticated user. Without scoping, one user could accidentally (or maliciously) collide with another's keys.
- **TTL.** 24 hours is Stripe's v1 default. 30 days is their v2 default. Pick based on your retry window. Run a cleanup job to delete expired keys.
- **Concurrency.** Return 409 when a request with the same key is already in progress. Don't queue it, don't block - just tell the client to wait and retry.
- **Parameter validation.** Compare the stored request parameters with the retry's parameters. Return 422 if they differ. This catches bugs where a client accidentally reuses a key for a different operation.
- **Cache scope.** Cache 2xx and 4xx responses. Don't cache 5xx - those are transient failures the client should be able to retry through.
- **Observability.** Log when you return a cached response vs a fresh one. Track the replay rate - a high ratio might indicate client-side retry storms.
- **Documentation.** Tell your clients about idempotency keys in your API docs. Stripe's docs on this are a good model - clear, with examples, and they explain the header alongside authentication in the getting-started section.

If you want a ready-made solution for Axum, the [`axum-idempotent`](https://docs.rs/axum-idempotent/latest/axum_idempotent/) crate (v0.2.6 as of writing) handles most of this out of the box - direct key mode, configurable expiration, automatic replay headers, and it skips caching for error status codes. Worth evaluating before building your own.

## Wrapping up

Idempotency isn't about preventing duplicate requests. Duplicates will happen - networks are unreliable, clients have retry logic, load balancers replay requests. Idempotency is about making those duplicates harmless.

The approach depends on what your endpoint does. Read operations are already safe. Full replacements (PUT) are safe by design. For everything else - creation, side effects, multi-step processes - you need either natural database constraints or explicit idempotency keys with a stored response cache.

The pattern is straightforward: client sends a unique key, server locks on first sight, processes, caches the response, and replays it on subsequent calls. The hard part is handling the edges - concurrent requests, stale locks, parameter mismatches, and the gap between side effects and cached responses. Getting those edges right is the difference between an API that mostly works and one that's actually safe to retry.
