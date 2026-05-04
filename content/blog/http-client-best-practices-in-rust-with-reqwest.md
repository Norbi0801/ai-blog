+++
title = "HTTP client best practices in Rust with reqwest"
date = 2025-04-03
description = "Production-ready HTTP clients in Rust - client reuse, connection pooling, timeouts, error handling, retries, rate limiting, TLS configuration, middleware, and testing with wiremock."

[taxonomies]
tags = ["rust", "reqwest", "http", "testing"]
+++

You pull in `reqwest`, fire off a GET request, get your JSON back, ship it. Two weeks later, your service is leaking connections under load, the upstream API starts returning 429s, and your error logs are a wall of "connection reset by peer". The reqwest crate is genuinely excellent - 250 million downloads, battle-tested - but it gives you enough rope to build some impressive footguns if you don't configure it properly.

This post covers the things that matter when you're building HTTP clients that need to survive production: client reuse, connection pools, timeouts, error handling, retries, rate limiting, TLS hardening, middleware, and testing.

<!-- more -->

## Stop creating clients per request

This is the single most common mistake. Here's what it looks like:

```rust
async fn fetch_user(id: u64) -> Result<User, reqwest::Error> {
    let client = reqwest::Client::new(); // don't do this
    client
        .get(format!("https://api.example.com/users/{id}"))
        .send()
        .await?
        .json::<User>()
        .await
}
```

Every call to `Client::new()` allocates a new connection pool, a new TLS session cache, a new resolver state, new internal `Arc`s. Under load, you're spinning up and tearing down TCP connections for every single request instead of reusing them. HTTP keep-alive? Gone. TLS session resumption? Gone. Connection pooling? Nonexistent.

`reqwest::Client` already wraps its internals in an `Arc`, so cloning is cheap - it just bumps a reference count. Create one client at startup, share it everywhere:

```rust
use reqwest::Client;
use std::time::Duration;

fn build_http_client() -> Client {
    Client::builder()
        .timeout(Duration::from_secs(30))
        .connect_timeout(Duration::from_secs(5))
        .pool_idle_timeout(Duration::from_secs(90))
        .pool_max_idle_per_host(10)
        .user_agent("my-service/1.0")
        .build()
        .expect("Failed to build HTTP client")
}
```

Then pass this client around. In an Axum app, stick it in state. In a CLI tool, construct it once in `main`. Whatever your architecture, one `Client`, many requests.

Why does this matter so much? Let's look at what happens under the hood.

## Connection pooling internals

When you send a request through a `Client`, reqwest checks its internal connection pool for an existing idle connection to that host. If one exists, it reuses it - no TCP handshake, no TLS negotiation. If not, it opens a new connection and adds it to the pool after the response completes.

Two knobs control pool behavior:

**`pool_idle_timeout`** - how long an unused connection stays in the pool before being closed. The default is 90 seconds. For services that make bursty requests, you might increase this. For memory-constrained environments hitting many different hosts, lower it:

```rust
// Long-lived connections to a small number of backends
Client::builder()
    .pool_idle_timeout(Duration::from_secs(300))
    .build()?

// Short-lived connections to many different hosts (web crawler)
Client::builder()
    .pool_idle_timeout(Duration::from_secs(15))
    .pool_max_idle_per_host(2)
    .build()?
```

**`pool_max_idle_per_host`** - caps idle connections per host. By default this is unlimited, which is fine for most services. If you're hitting thousands of different hosts (crawler, link checker), set this low to avoid holding open thousands of idle sockets.

The connection pool is keyed by scheme + host + port. A request to `https://api.example.com:443/users` reuses connections from `https://api.example.com:443/products` because the pool key is the same. This is why client reuse matters so much - it's not just about avoiding allocation, it's about amortizing the TCP and TLS handshake cost across many requests to the same backend.

## Timeouts: three layers

reqwest gives you three distinct timeout controls, and you should configure all of them:

```rust
Client::builder()
    // Total time from request start to full response body received.
    // This is your backstop - nothing runs longer than this.
    .timeout(Duration::from_secs(30))

    // Time to establish the TCP connection.
    // Catches DNS failures, unreachable hosts, firewalls that drop SYN packets.
    .connect_timeout(Duration::from_secs(5))

    // Time between individual read operations on the response body.
    // Catches servers that accept the connection then stop sending data.
    .read_timeout(Duration::from_secs(10))

    .build()?
```

Without explicit timeouts, a hanging server can block your task indefinitely. The default `Client::new()` has **no timeout at all** - it will wait forever. This is the second most common production issue after creating clients per request.

A rough guideline for internal service-to-service calls:

| Setting | Value | Why |
|---------|-------|-----|
| `connect_timeout` | 2-5s | If your backend isn't reachable in 5 seconds, something is very wrong |
| `read_timeout` | 10-15s | Protects against slow-drip responses |
| `timeout` | 30s | Hard upper bound on total request time |

For external APIs (Stripe, GitHub, AWS), you might want longer timeouts since you don't control their infrastructure. But always set _something_.

You can also override the timeout per request:

```rust
let response = client
    .get("https://slow-api.example.com/report")
    .timeout(Duration::from_secs(120)) // override for this specific call
    .send()
    .await?;
```

## User-Agent: set it or regret it

Many APIs rate-limit or block requests with no User-Agent. Some return different responses based on it. GitHub's API, for instance, requires a User-Agent header and will reject requests without one.

```rust
Client::builder()
    .user_agent(concat!(env!("CARGO_PKG_NAME"), "/", env!("CARGO_PKG_VERSION")))
    .build()?
```

This sets the User-Agent from your Cargo.toml's package name and version at compile time. For a crate named `my-service` at version `0.3.1`, the header becomes `my-service/0.3.1`.

For default headers you need on every request (API keys, content types, correlation IDs):

```rust
use reqwest::header::{HeaderMap, HeaderValue, ACCEPT, AUTHORIZATION};

let mut headers = HeaderMap::new();
headers.insert(ACCEPT, HeaderValue::from_static("application/json"));
headers.insert(
    AUTHORIZATION,
    HeaderValue::from_str(&format!("Bearer {}", api_key))?,
);

let client = Client::builder()
    .default_headers(headers)
    .user_agent("my-service/1.0")
    .build()?;
```

Every request through this client will carry those headers. Individual requests can override or add more.

## JSON deserialization with serde

reqwest integrates directly with serde. The `json` feature (enabled by default) gives you `.json::<T>()` on responses:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct User {
    id: u64,
    login: String,
    email: Option<String>,
}

let user: User = client
    .get("https://api.github.com/users/octocat")
    .send()
    .await?
    .error_for_status()?  // we'll get to this
    .json::<User>()
    .await?;
```

A few things worth knowing about `.json()`:

**It consumes the response body.** You can't call it twice. If you need the raw bytes _and_ the parsed struct, read the bytes first:

```rust
let bytes = response.bytes().await?;
let user: User = serde_json::from_slice(&bytes)?;
// you still have `bytes` if you need it for logging/debugging
```

**It reads the entire body into memory.** For huge responses, consider streaming with `.json()` replaced by manual deserialization from `.bytes_stream()`. But for typical API responses (kilobytes to low megabytes), this is fine.

**Use `#[serde(rename_all = "camelCase")]` when the API uses camelCase.** Most APIs do. Serde's rename attributes save you from littering your code with `#[serde(rename = "firstName")]` on every field:

```rust
#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
struct ApiResponse {
    request_id: String,     // maps from "requestId"
    total_count: u64,       // maps from "totalCount"
    next_page_token: Option<String>,
}
```

**Default values prevent breakage when the API adds fields:**

```rust
#[derive(Debug, Deserialize)]
struct Config {
    name: String,
    #[serde(default)]
    enabled: bool,          // defaults to false if missing
    #[serde(default)]
    tags: Vec<String>,      // defaults to empty vec if missing
}
```

And `#[serde(deny_unknown_fields)]` does the opposite - fails if the response has fields you didn't expect. Useful during development to catch API drift, annoying in production when the provider adds a new field and your service falls over.

## Error handling: status codes vs network errors

reqwest's error handling has a subtlety that trips up a lot of people. **A successful `.send()` does not mean the request succeeded.** It means the HTTP round-trip completed - you got a response back. A 404, a 500, a 503 - these all return `Ok(Response)`, not `Err`.

There are two categories of errors:

**Network errors** - DNS resolution failure, connection refused, timeout, TLS handshake failure. These are returned as `Err(reqwest::Error)` from `.send()`.

**HTTP errors** - The server responded, but with a non-2xx status code. These are _not_ errors by default. You need to check explicitly.

The most common pattern uses `error_for_status()`:

```rust
let response = client
    .get("https://api.example.com/users/123")
    .send()
    .await?                  // network error?
    .error_for_status()?;   // HTTP 4xx/5xx?

let user: User = response.json().await?;
```

`error_for_status()` converts the response into `Err` if the status is 4xx or 5xx. The error includes the status code, which you can inspect:

```rust
match client.get(url).send().await {
    Ok(resp) => {
        match resp.error_for_status() {
            Ok(resp) => {
                let data: MyData = resp.json().await?;
                Ok(data)
            }
            Err(e) => {
                // e.status() gives you the StatusCode
                let status = e.status().unwrap();
                if status == StatusCode::NOT_FOUND {
                    Ok(MyData::default())
                } else if status == StatusCode::TOO_MANY_REQUESTS {
                    // handle rate limiting
                    Err(AppError::RateLimited)
                } else {
                    Err(AppError::Upstream(e))
                }
            }
        }
    }
    Err(e) if e.is_timeout() => Err(AppError::Timeout),
    Err(e) if e.is_connect() => Err(AppError::ConnectionFailed),
    Err(e) => Err(AppError::Network(e)),
}
```

For a cleaner approach, wrap the common pattern in a function or method on your own client type. If you read my [adapter pattern post](/blog/the-adapter-pattern-in-rust---wrapping-external-apis/), this fits naturally - your adapter's methods handle the HTTP-level error mapping, and the rest of your code works with your domain error type.

`reqwest::Error` has some useful inspection methods:

```rust
let err: reqwest::Error = /* ... */;

err.is_timeout()    // true if any timeout (connect, read, total) expired
err.is_connect()    // true if connection failed (refused, DNS, etc.)
err.is_request()    // true if error occurred while sending the request
err.is_body()       // true if error occurred while reading response body
err.is_decode()     // true if JSON/body deserialization failed
err.is_redirect()   // true if too many redirects
err.status()        // Some(StatusCode) if from error_for_status()
err.url()           // the URL that triggered the error
```

This is enough to build solid error classification without string-matching error messages.

## Retries and backoff

I covered retry strategies in depth in a [previous post](/blog/retry-strategies-exponential-backoff-jitter-and-circuit-breakers) - exponential backoff, jitter, circuit breakers, the whole spectrum. Here I'll focus specifically on how to wire retries into a reqwest client using `reqwest-middleware` and `reqwest-retry`.

First, the dependencies:

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
reqwest-middleware = "0.4"
reqwest-retry = "0.7"
```

Then build a client with retry middleware:

```rust
use reqwest::Client;
use reqwest_middleware::ClientBuilder;
use reqwest_retry::{
    RetryTransientMiddleware,
    policies::ExponentialBackoff,
};

let retry_policy = ExponentialBackoff::builder()
    .build_with_max_retries(3);

let client = ClientBuilder::new(Client::builder()
        .timeout(Duration::from_secs(30))
        .connect_timeout(Duration::from_secs(5))
        .build()?)
    .with(RetryTransientMiddleware::new_with_policy(retry_policy))
    .build();

// Use exactly like a normal reqwest client
let response = client
    .get("https://api.example.com/data")
    .send()
    .await?;
```

`reqwest-retry` classifies responses automatically:

- **Transient (retryable)**: 408 (Request Timeout), 429 (Too Many Requests), 500, 502, 503, 504, and network/timeout errors.
- **Fatal (not retried)**: 400, 401, 403, 404, and other client errors.

You can customize this with a `RetryableStrategy`:

```rust
use reqwest_retry::RetryableStrategy;
use reqwest_middleware::reqwest::Response;

struct CustomRetryStrategy;

impl RetryableStrategy for CustomRetryStrategy {
    fn handle(&self, res: &Result<Response, reqwest_middleware::Error>) -> Option<reqwest_retry::Retryable> {
        match res {
            Ok(resp) if resp.status() == 429 => {
                Some(reqwest_retry::Retryable::Transient)
            }
            Ok(resp) if resp.status().is_server_error() => {
                Some(reqwest_retry::Retryable::Transient)
            }
            Ok(_) => None, // success, don't retry
            Err(_) => Some(reqwest_retry::Retryable::Transient),
        }
    }
}
```

One thing `reqwest-retry` handles that you'd have to build yourself: it respects `Retry-After` headers when present. If the server says "come back in 30 seconds", the middleware waits 30 seconds instead of using its calculated backoff delay.

## Client-side rate limiting

Rate limiting from the server side was covered in my [REST API design post](/blog/designing-restful-apis-practical-guidelines-beyond-the-theory). But what about being a _good client_? If you know an API allows 100 requests per minute, you should throttle yourself rather than hammering the endpoint and handling 429 responses.

The simplest approach uses a `tokio::sync::Semaphore` to cap concurrency:

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

struct RateLimitedClient {
    client: reqwest::Client,
    semaphore: Arc<Semaphore>,
}

impl RateLimitedClient {
    fn new(client: reqwest::Client, max_concurrent: usize) -> Self {
        Self {
            client,
            semaphore: Arc::new(Semaphore::new(max_concurrent)),
        }
    }

    async fn get(&self, url: &str) -> Result<reqwest::Response, reqwest::Error> {
        let _permit = self.semaphore.acquire().await.unwrap();
        self.client.get(url).send().await
    }
}
```

This limits concurrency but not request rate. For actual rate limiting (N requests per time window), you need a token bucket or leaky bucket. The `governor` crate implements the Generic Cell Rate Algorithm (GCRA), which is a sophisticated leaky bucket variant:

```rust
use governor::{Quota, RateLimiter};
use std::num::NonZeroU32;

struct ThrottledClient {
    client: reqwest::Client,
    limiter: Arc<RateLimiter<
        governor::state::NotKeyed,
        governor::state::InMemoryState,
        governor::clock::DefaultClock,
    >>,
}

impl ThrottledClient {
    fn new(client: reqwest::Client, requests_per_second: u32) -> Self {
        let quota = Quota::per_second(NonZeroU32::new(requests_per_second).unwrap());
        Self {
            client,
            limiter: Arc::new(RateLimiter::direct(quota)),
        }
    }

    async fn get(&self, url: &str) -> Result<reqwest::Response, reqwest::Error> {
        self.limiter.until_ready().await;
        self.client.get(url).send().await
    }
}
```

`until_ready()` is async - it doesn't spin-wait. It yields the task until the rate limiter has a token available. For bursty workloads, you can configure a burst size with `Quota::per_second(n).allow_burst(NonZeroU32::new(burst).unwrap())`.

There's also [`reqwest-ratelimit`](https://crates.io/crates/reqwest-ratelimit) which integrates with `reqwest-middleware` as a middleware layer, so you can compose it with retries and logging without building a wrapper struct.

## TLS configuration and certificate management

By default, reqwest 0.12+ uses `rustls` as its TLS backend, which means no dependency on OpenSSL. This is a sensible default for most cases.

### Enforcing HTTPS-only

If your client should never make plaintext HTTP requests:

```rust
Client::builder()
    .https_only(true) // rejects http:// URLs at request time
    .build()?
```

### Minimum TLS version

Don't negotiate down to TLS 1.0 or 1.1:

```rust
use reqwest::tls::Version;

Client::builder()
    .min_tls_version(Version::TLS_1_2)
    .build()?
```

### Custom root certificates

For internal services with self-signed certs or private CAs:

```rust
use reqwest::tls::Certificate;

let cert_pem = std::fs::read("internal-ca.pem")?;
let cert = Certificate::from_pem(&cert_pem)?;

Client::builder()
    .tls_certs_merge(vec![cert]) // adds to system roots
    .build()?
```

Use `tls_certs_merge` to add your custom CA alongside the system's default trust store. Use `tls_certs_only` if you want to trust _only_ your certificates and nothing else - useful for services that should only talk to your internal infrastructure.

### Certificate pinning

reqwest doesn't have a built-in pin-sha256 mechanism. For certificate pinning, you have two options:

**Option 1: Use `rustls` directly with a custom `ServerCertVerifier`.**

This gives you full control. You extract the server's public key during the TLS handshake, hash it with SHA-256, and compare against your pinned values:

```rust
use rustls::client::danger::{ServerCertVerifier, ServerCertVerified, HandshakeSignatureValid};
use rustls::pki_types::{ServerName, CertificateDer, UnixTime};
use sha2::{Sha256, Digest};

#[derive(Debug)]
struct PinnedCertVerifier {
    pinned_hashes: Vec<[u8; 32]>,
    default_verifier: Arc<dyn ServerCertVerifier>,
}

impl ServerCertVerifier for PinnedCertVerifier {
    fn verify_server_cert(
        &self,
        end_entity: &CertificateDer,
        _intermediates: &[CertificateDer],
        _server_name: &ServerName,
        _ocsp_response: &[u8],
        _now: UnixTime,
    ) -> Result<ServerCertVerified, rustls::Error> {
        let mut hasher = Sha256::new();
        hasher.update(end_entity.as_ref());
        let hash: [u8; 32] = hasher.finalize().into();

        if self.pinned_hashes.contains(&hash) {
            Ok(ServerCertVerified::assertion())
        } else {
            Err(rustls::Error::General(
                "Certificate does not match pinned hash".into(),
            ))
        }
    }

    fn verify_tls12_signature(
        &self,
        message: &[u8],
        cert: &CertificateDer,
        dss: &rustls::DigitallySignedStruct,
    ) -> Result<HandshakeSignatureValid, rustls::Error> {
        self.default_verifier.verify_tls12_signature(message, cert, dss)
    }

    fn verify_tls13_signature(
        &self,
        message: &[u8],
        cert: &CertificateDer,
        dss: &rustls::DigitallySignedStruct,
    ) -> Result<HandshakeSignatureValid, rustls::Error> {
        self.default_verifier.verify_tls13_signature(message, cert, dss)
    }

    fn supported_verify_schemes(&self) -> Vec<rustls::SignatureScheme> {
        self.default_verifier.supported_verify_schemes()
    }
}
```

Then configure a rustls `ClientConfig` with this verifier and pass it to reqwest via `tls_backend_preconfigured`. This is production-grade but requires more ceremony.

**Option 2: Use the [`rustls-pin`](https://crates.io/crates/rustls-pin) crate** which wraps this pattern into a simpler API.

For most services, standard CA verification plus `https_only(true)` is sufficient. Certificate pinning is worth the effort for mobile apps, financial APIs, or any context where you're defending against compromised certificate authorities.

### Danger zone

Two methods exist for development purposes:

```rust
// DO NOT use in production
Client::builder()
    .danger_accept_invalid_certs(true)
    .danger_accept_invalid_hostnames(true)
    .build()?
```

These disable certificate and hostname verification entirely. If you see this in production code, it's a security vulnerability. Use `#[cfg(debug_assertions)]` or feature flags to make sure it can't reach a release build.

## Middleware with reqwest-middleware

The `reqwest-middleware` crate (by [TrueLayer](https://github.com/TrueLayer/reqwest-middleware)) wraps reqwest's `Client` and lets you stack middleware layers. We already saw `reqwest-retry`. Let's look at building custom middleware.

A middleware implements the `Middleware` trait:

```rust
use reqwest_middleware::{Middleware, Next};
use reqwest::Request;
use http::Extensions;

struct LoggingMiddleware;

#[async_trait::async_trait]
impl Middleware for LoggingMiddleware {
    async fn handle(
        &self,
        req: Request,
        extensions: &mut Extensions,
        next: Next<'_>,
    ) -> reqwest_middleware::Result<reqwest::Response> {
        let method = req.method().clone();
        let url = req.url().clone();
        let start = std::time::Instant::now();

        tracing::info!(%method, %url, "sending request");

        let result = next.run(req, extensions).await;

        let elapsed = start.elapsed();
        match &result {
            Ok(resp) => {
                tracing::info!(
                    %method,
                    %url,
                    status = %resp.status(),
                    elapsed_ms = elapsed.as_millis(),
                    "response received"
                );
            }
            Err(e) => {
                tracing::error!(
                    %method,
                    %url,
                    error = %e,
                    elapsed_ms = elapsed.as_millis(),
                    "request failed"
                );
            }
        }

        result
    }
}
```

Stack middleware in order - they execute top-to-bottom on the request path, bottom-to-top on the response:

```rust
use reqwest_middleware::ClientBuilder;
use reqwest_retry::{RetryTransientMiddleware, policies::ExponentialBackoff};

let retry_policy = ExponentialBackoff::builder()
    .build_with_max_retries(3);

let client = ClientBuilder::new(reqwest::Client::builder()
        .timeout(Duration::from_secs(30))
        .build()?)
    .with(LoggingMiddleware)
    .with(RetryTransientMiddleware::new_with_policy(retry_policy))
    .build();
```

Here, `LoggingMiddleware` runs first (outer), so it logs every attempt including retries. If you want to log only the final result, swap the order so logging is inner (added after retry).

Common middleware patterns worth building:

- **Request ID injection**: Add a `X-Request-Id` header for distributed tracing correlation. If you've set up [tracing and OpenTelemetry](/blog/monitoring-rust-applications-in-production), you can propagate the current span's trace ID.
- **Response caching**: Cache GET responses by URL with a TTL. The [`reqwest-cache`](https://crates.io/crates/reqwest-cache) crate provides this.
- **Metrics collection**: Count requests by method/status/host, measure latency histograms.

## Testing with wiremock

Testing HTTP clients against real APIs is slow, flaky, and costs money if the API is metered. [wiremock](https://crates.io/crates/wiremock) (by Luca Palmieri, author of "Zero to Production in Rust") gives you a local HTTP server that you can program with expected requests and canned responses.

```toml
[dev-dependencies]
wiremock = "0.6"
```

Basic test structure:

```rust
use wiremock::{MockServer, Mock, ResponseTemplate};
use wiremock::matchers::{method, path, header};

#[tokio::test]
async fn fetches_user_by_id() {
    // Start a mock server on a random port
    let server = MockServer::start().await;

    // Program the mock
    Mock::given(method("GET"))
        .and(path("/users/42"))
        .and(header("accept", "application/json"))
        .respond_with(
            ResponseTemplate::new(200)
                .set_body_json(serde_json::json!({
                    "id": 42,
                    "login": "octocat",
                    "email": "octo@example.com"
                })),
        )
        .expect(1) // assert this mock is called exactly once
        .mount(&server)
        .await;

    // Build a client pointing at the mock server
    let client = reqwest::Client::new();
    let url = format!("{}/users/42", server.uri());

    let user: User = client
        .get(&url)
        .header("accept", "application/json")
        .send()
        .await
        .unwrap()
        .json()
        .await
        .unwrap();

    assert_eq!(user.id, 42);
    assert_eq!(user.login, "octocat");
    // When `server` drops, it verifies all expectations (expect(1)) were met
}
```

Each `MockServer::start()` binds to a random available port, so tests run in parallel without conflicts. When the server drops, it checks that all mounted mocks had their expectations satisfied - if you said `expect(1)` and the mock was never called, the test panics.

### Testing error scenarios

This is where wiremock really shines. Testing how your client handles failures:

```rust
#[tokio::test]
async fn handles_server_error_gracefully() {
    let server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/users/42"))
        .respond_with(ResponseTemplate::new(500))
        .mount(&server)
        .await;

    let result = fetch_user(&client, &server.uri(), 42).await;
    assert!(matches!(result, Err(AppError::Upstream(_))));
}

#[tokio::test]
async fn handles_timeout() {
    let server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/users/42"))
        .respond_with(
            ResponseTemplate::new(200)
                .set_body_json(serde_json::json!({"id": 42, "login": "octocat"}))
                .set_delay(Duration::from_secs(10)), // delay longer than client timeout
        )
        .mount(&server)
        .await;

    let client = reqwest::Client::builder()
        .timeout(Duration::from_millis(500))
        .build()
        .unwrap();

    let result = fetch_user(&client, &server.uri(), 42).await;
    assert!(matches!(result, Err(AppError::Timeout)));
}

#[tokio::test]
async fn handles_rate_limiting() {
    let server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/users/42"))
        .respond_with(
            ResponseTemplate::new(429)
                .insert_header("retry-after", "5"),
        )
        .mount(&server)
        .await;

    let result = fetch_user(&client, &server.uri(), 42).await;
    assert!(matches!(result, Err(AppError::RateLimited)));
}
```

The `.set_delay()` method is what makes timeout testing possible without actually waiting for real timeouts. wiremock delays the response, your client's timeout fires first, and you can assert that your code handles it correctly.

### Testing retry behavior

Combine wiremock with `reqwest-middleware` to verify retries:

```rust
use wiremock::MockGuard;

#[tokio::test]
async fn retries_on_transient_failure() {
    let server = MockServer::start().await;

    // First two calls return 503, third succeeds
    let fail_mock = Mock::given(method("GET"))
        .and(path("/data"))
        .respond_with(ResponseTemplate::new(503))
        .expect(2)       // called twice (initial + 1 retry)
        .up_to_n_times(2) // only respond to first 2 matching requests
        .mount(&server)
        .await;

    Mock::given(method("GET"))
        .and(path("/data"))
        .respond_with(
            ResponseTemplate::new(200)
                .set_body_json(serde_json::json!({"value": "ok"})),
        )
        .expect(1) // third call succeeds
        .mount(&server)
        .await;

    let retry_policy = ExponentialBackoff::builder()
        .build_with_max_retries(3);

    let client = ClientBuilder::new(reqwest::Client::new())
        .with(RetryTransientMiddleware::new_with_policy(retry_policy))
        .build();

    let resp = client
        .get(format!("{}/data", server.uri()))
        .send()
        .await
        .unwrap();

    assert_eq!(resp.status(), 200);
    // Mock expectations are verified on drop
}
```

`up_to_n_times(2)` makes the mock respond only to the first two matching requests, after which wiremock falls through to the next matching mock (the 200 response). Combined with `expect(2)`, this gives you precise control over the retry sequence.

## Putting it all together

Here's a production-ready client setup that combines everything:

```rust
use reqwest::Client;
use reqwest_middleware::ClientBuilder;
use reqwest_retry::{RetryTransientMiddleware, policies::ExponentialBackoff};
use std::time::Duration;

pub struct ApiClient {
    inner: reqwest_middleware::ClientWithMiddleware,
    base_url: String,
}

impl ApiClient {
    pub fn new(base_url: &str, api_key: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let mut default_headers = reqwest::header::HeaderMap::new();
        default_headers.insert(
            reqwest::header::AUTHORIZATION,
            reqwest::header::HeaderValue::from_str(&format!("Bearer {}", api_key))?,
        );
        default_headers.insert(
            reqwest::header::ACCEPT,
            reqwest::header::HeaderValue::from_static("application/json"),
        );

        let raw_client = Client::builder()
            .user_agent(concat!(env!("CARGO_PKG_NAME"), "/", env!("CARGO_PKG_VERSION")))
            .default_headers(default_headers)
            .timeout(Duration::from_secs(30))
            .connect_timeout(Duration::from_secs(5))
            .read_timeout(Duration::from_secs(10))
            .pool_idle_timeout(Duration::from_secs(90))
            .pool_max_idle_per_host(10)
            .https_only(true)
            .build()?;

        let retry_policy = ExponentialBackoff::builder()
            .build_with_max_retries(3);

        let client = ClientBuilder::new(raw_client)
            .with(RetryTransientMiddleware::new_with_policy(retry_policy))
            .build();

        Ok(Self {
            inner: client,
            base_url: base_url.to_string(),
        })
    }

    pub async fn get_user(&self, id: u64) -> Result<User, ApiError> {
        let resp = self
            .inner
            .get(format!("{}/users/{}", self.base_url, id))
            .send()
            .await
            .map_err(|e| ApiError::Request(e.to_string()))?;

        match resp.status() {
            s if s.is_success() => {
                resp.json::<User>().await.map_err(|e| ApiError::Decode(e.to_string()))
            }
            reqwest::StatusCode::NOT_FOUND => Err(ApiError::NotFound(id)),
            reqwest::StatusCode::TOO_MANY_REQUESTS => Err(ApiError::RateLimited),
            s => Err(ApiError::Upstream(s.as_u16())),
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("request failed: {0}")]
    Request(String),
    #[error("failed to decode response: {0}")]
    Decode(String),
    #[error("user {0} not found")]
    NotFound(u64),
    #[error("rate limited")]
    RateLimited,
    #[error("upstream error: HTTP {0}")]
    Upstream(u16),
}
```

This gives you: connection reuse, timeouts at every level, automatic retries with exponential backoff, a proper User-Agent, HTTPS enforcement, JSON deserialization into typed structs, and error handling that distinguishes between "not found", "rate limited", "upstream broke", and "network failed".

## Quick reference

| Concern | Solution |
|---------|----------|
| Client per request | Create one `Client`, share via clone |
| No timeouts | Set `timeout`, `connect_timeout`, `read_timeout` |
| Connection exhaustion | Tune `pool_idle_timeout`, `pool_max_idle_per_host` |
| Missing User-Agent | `.user_agent()` on builder |
| Ignoring HTTP errors | `.error_for_status()` or match on `status()` |
| No retries | `reqwest-retry` middleware with exponential backoff |
| Hammering upstream | Client-side rate limiting with `governor` or semaphore |
| Plaintext leaking | `.https_only(true)` |
| Untestable | `wiremock` for mock HTTP servers |
| Cross-cutting concerns | `reqwest-middleware` for logging, tracing, metrics |

Most HTTP client bugs in production come from missing configuration, not wrong logic. Set your timeouts, reuse your client, handle your errors, test your failure paths. The defaults aren't enough.
