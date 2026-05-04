+++
title = "Axum vs Actix-web vs Warp - Rust web frameworks in 2026"
date = 2025-06-07
description = "A practical comparison of the three major Rust web frameworks: API design, middleware, extractors, error handling, ecosystem, and when to pick which."

[taxonomies]
tags = ["rust", "web", "axum", "architecture"]
+++

Three frameworks. All async, all fast, all production-ready. If you search "which Rust web framework should I use" you'll find a hundred threads with a hundred opinions, and most of them boil down to "it depends." Not very helpful.

So instead of vibes, let's look at the actual code. We'll build the same endpoint in all three, compare how they handle middleware, extractors, and errors, look at real numbers, and walk through the tradeoffs that actually matter when you're picking one for a project.

<!-- more -->

## The state of play in 2026

A quick snapshot of where each framework stands today:

| | **Axum** | **Actix-web** | **Warp** |
|---|---|---|---|
| Latest version | 0.8.8 (Dec 2025) | 4.13.0 (Feb 2026) | 0.4.2 (Aug 2025) |
| Total downloads | ~277M | ~63M | ~39M |
| Recent downloads | ~63M | ~9.4M | ~4M |
| GitHub stars | ~22K | ~23K | ~10K |
| Async runtime | tokio | tokio (default) | tokio |
| HTTP layer | hyper 1.x | custom (h1/h2) | hyper 1.x |
| Middleware | Tower | Own system + Tower compat | Filter composition |

The download numbers tell a clear story. Axum has exploded in adoption - 63 million recent downloads versus Actix-web's 9.4 million. Warp is a distant third. This wasn't the case two years ago. Axum's tight integration with the tokio ecosystem and its ergonomic API have made it the default choice for new projects.

But popularity isn't the whole picture. Let's look at what's actually different.

## The same endpoint, three ways

The best way to compare frameworks is to write the same thing in each. Here's a simple endpoint: `POST /api/users` that accepts a JSON body, validates it, and returns the created user.

### Axum

```rust
use axum::{
    extract::State,
    http::StatusCode,
    response::IntoResponse,
    routing::post,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Serialize, Clone)]
struct User {
    id: u64,
    name: String,
    email: String,
}

type AppState = Arc<RwLock<Vec<User>>>;

async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<CreateUser>,
) -> Result<(StatusCode, Json<User>), (StatusCode, String)> {
    if payload.email.is_empty() {
        return Err((StatusCode::BAD_REQUEST, "email is required".into()));
    }

    let mut users = state.write().await;
    let user = User {
        id: users.len() as u64 + 1,
        name: payload.name,
        email: payload.email,
    };
    users.push(user.clone());

    Ok((StatusCode::CREATED, Json(user)))
}

#[tokio::main]
async fn main() {
    let state: AppState = Arc::new(RwLock::new(Vec::new()));

    let app = Router::new()
        .route("/api/users", post(create_user))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

The handler is a plain async function. State and body are extracted via typed function parameters. If `Json` deserialization fails, axum returns a 422 automatically. The return type uses `Result` with `IntoResponse` on both sides.

### Actix-web

```rust
use actix_web::{post, web, App, HttpResponse, HttpServer};
use serde::{Deserialize, Serialize};
use std::sync::RwLock;

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Serialize, Clone)]
struct User {
    id: u64,
    name: String,
    email: String,
}

struct AppState {
    users: RwLock<Vec<User>>,
}

#[post("/api/users")]
async fn create_user(
    state: web::Data<AppState>,
    payload: web::Json<CreateUser>,
) -> HttpResponse {
    if payload.email.is_empty() {
        return HttpResponse::BadRequest().body("email is required");
    }

    let mut users = state.users.write().unwrap();
    let user = User {
        id: users.len() as u64 + 1,
        name: payload.name.clone(),
        email: payload.email.clone(),
    };
    users.push(user.clone());

    HttpResponse::Created().json(user)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let state = web::Data::new(AppState {
        users: RwLock::new(Vec::new()),
    });

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .service(create_user)
    })
    .bind("0.0.0.0:3000")?
    .run()
    .await
}
```

Actix uses attribute macros (`#[post("/api/users")]`) to annotate handlers directly. State is wrapped in `web::Data` (an `Arc` internally). The closure in `HttpServer::new` runs per worker thread - actix spawns one worker per CPU core by default, each with its own `App` instance.

Notice `payload.name.clone()` - actix's `web::Json` wraps the inner value, and you need to clone out of it (or call `into_inner()`). A small friction point.

### Warp

```rust
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::RwLock;
use warp::Filter;

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Serialize, Clone)]
struct User {
    id: u64,
    name: String,
    email: String,
}

type AppState = Arc<RwLock<Vec<User>>>;

fn with_state(
    state: AppState,
) -> impl Filter<Extract = (AppState,), Error = std::convert::Infallible> + Clone {
    warp::any().map(move || state.clone())
}

async fn create_user(
    state: AppState,
    payload: CreateUser,
) -> Result<impl warp::Reply, warp::Rejection> {
    if payload.email.is_empty() {
        return Err(warp::reject::custom(ValidationError));
    }

    let mut users = state.write().await;
    let user = User {
        id: users.len() as u64 + 1,
        name: payload.name,
        email: payload.email,
    };
    users.push(user.clone());

    Ok(warp::reply::with_status(
        warp::reply::json(&user),
        warp::http::StatusCode::CREATED,
    ))
}

#[derive(Debug)]
struct ValidationError;
impl warp::reject::Reject for ValidationError {}

#[tokio::main]
async fn main() {
    let state: AppState = Arc::new(RwLock::new(Vec::new()));

    let create = warp::path!("api" / "users")
        .and(warp::post())
        .and(with_state(state))
        .and(warp::body::json())
        .and_then(create_user);

    warp::serve(create).run(([0, 0, 0, 0], 3000)).await;
}
```

Warp is the odd one out. Routes are built by composing `Filter` values with `.and()`. Each filter extracts something - the path, the method, the state, the body - and the handler receives all extracted values as function arguments. It's conceptually elegant but practically verbose. That `with_state` helper function? You'll write one in every warp project. And custom rejections require implementing the `Reject` trait for each error type.

## What the code tells us

Even from this simple example, the design philosophies are visible:

**Axum** treats handlers as plain functions. Extraction happens through the type system - if your function takes `Json<CreateUser>`, axum figures out how to get it from the request. This is the same pattern used in the [webhook receiver post](/blog/building-a-webhook-receiver-in-rust/). It maps naturally to how you already think about function signatures.

**Actix-web** is the most "framework-y." It has its own attribute macros, its own extraction wrappers, and its own runtime configuration. If you've used Express.js or Django, the mental model transfers. The `#[post]` attribute is nice for discoverability - you can grep for routes.

**Warp** encodes the entire request pipeline in the type system. Every `.and()` call changes the type of the filter chain. The compiler knows exactly what data flows through each stage. This is powerful for correctness but makes error messages absolutely brutal when things don't line up.

## Extractors

Extractors are how data gets from the HTTP request into your handler. This is where the frameworks diverge most.

### Axum's approach

Axum extractors are types that implement `FromRequest` or `FromRequestParts`. Your handler can take any number of them as arguments:

```rust
async fn handler(
    State(db): State<DbPool>,
    Query(params): Query<ListParams>,
    headers: HeaderMap,
    Json(body): Json<CreateItem>,
) -> impl IntoResponse {
    // all four values extracted from the request
}
```

Since axum 0.8, `FromRequestParts` is a regular async trait - no more `#[async_trait]` macro needed. The `Option<T>` extractor also got smarter: it now distinguishes between "the value wasn't present" and "the value was present but invalid," which fixes a long-standing footgun where bad auth tokens silently became `None`.

Writing custom extractors is straightforward:

```rust
struct AuthenticatedUser {
    id: u64,
    role: String,
}

impl<S> FromRequestParts<S> for AuthenticatedUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        let token = parts
            .headers
            .get("Authorization")
            .and_then(|v| v.to_str().ok())
            .ok_or((StatusCode::UNAUTHORIZED, "missing token".into()))?;

        // validate token, look up user...
        Ok(AuthenticatedUser { id: 1, role: "admin".into() })
    }
}

// Now just add it to any handler:
async fn admin_endpoint(user: AuthenticatedUser) -> impl IntoResponse {
    format!("hello user {}", user.id)
}
```

### Actix-web's approach

Actix extractors implement the `FromRequest` trait. They look similar on the surface:

```rust
async fn handler(
    db: web::Data<DbPool>,
    params: web::Query<ListParams>,
    req: HttpRequest,
    body: web::Json<CreateItem>,
) -> HttpResponse {
    // ...
}
```

The key difference: actix has a limit of 12 extractor arguments per handler (it implements the handler trait for tuples up to size 12). In practice you rarely hit this, but it's a design-time limit that axum doesn't have.

Actix also provides `web::Path` for path parameters, `web::Form` for form data, and `web::Payload` for raw bytes. The `HttpRequest` type gives you access to everything else.

### Warp's approach

Warp doesn't have extractors in the same sense. Instead, everything is a `Filter`:

```rust
let route = warp::path!("api" / "items" / u64)
    .and(warp::header::<String>("authorization"))
    .and(warp::query::<ListParams>())
    .and(warp::body::json::<CreateItem>())
    .and_then(handler);
```

Each `.and()` adds another value to the extraction tuple. The handler receives them in order. This is maximally explicit - you can see exactly what's being extracted by reading the filter chain. But it also means the route definition and the handler signature must be kept in sync manually. Add an `.and()` in the filter chain but forget to add the corresponding parameter to the handler? The compiler error will be a novel-length type mismatch that takes real effort to decode.

## Middleware

If you've read my post on [understanding tokio](/blog/understanding-tokio-the-rust-async-runtime-under-the-hood/), you know the async ecosystem revolves around traits like `Future` and `Service`. Middleware is where that matters most.

### Axum: Tower all the way

Axum is built on [Tower](https://docs.rs/tower/latest/tower/), the service abstraction from the tokio team. A Tower middleware is a `Layer` that wraps a `Service`. This means every middleware written for any Tower-compatible framework works with axum out of the box.

```rust
use axum::{middleware, Router};
use tower_http::{
    compression::CompressionLayer,
    cors::CorsLayer,
    trace::TraceLayer,
};

let app = Router::new()
    .route("/api/users", post(create_user))
    .layer(TraceLayer::new_for_http())
    .layer(CompressionLayer::new())
    .layer(CorsLayer::permissive());
```

The [tower-http](https://docs.rs/tower-http/latest/tower_http/) crate provides a rich set of middleware: compression, CORS, request ID, rate limiting, timeout, request body limits, sensitive headers redaction. These all work with axum, tonic (gRPC), and any other Tower-based framework.

You can also write lightweight middleware using axum's `from_fn`:

```rust
async fn auth_middleware(
    req: axum::extract::Request,
    next: middleware::Next,
) -> Result<impl IntoResponse, StatusCode> {
    let token = req
        .headers()
        .get("Authorization")
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // validate token...

    Ok(next.run(req).await)
}

let app = Router::new()
    .route("/api/protected", get(protected_handler))
    .route_layer(middleware::from_fn(auth_middleware));
```

### Actix-web: two systems

Actix has its own middleware trait (`Transform` + `Service`) that predates Tower. The API is more verbose:

```rust
use actix_web::middleware;

let app = App::new()
    .wrap(middleware::Logger::default())
    .wrap(middleware::Compress::default())
    .wrap(
        actix_cors::Cors::default()
            .allow_any_origin()
            .allow_any_method(),
    );
```

Actix also supports Tower middleware through a compatibility layer, but the experience is mixed. Some Tower middleware works cleanly, some requires adapters. The answer to "should I use actix middleware or Tower middleware?" depends on the specific middleware, and that research overhead adds up.

Writing custom actix middleware involves implementing two traits (`Transform` and `Service`) and managing a decent amount of boilerplate:

```rust
use actix_web::dev::{Service, ServiceRequest, ServiceResponse, Transform};
use futures::future::{ok, LocalBoxFuture, Ready};

pub struct Auth;

impl<S, B> Transform<S, ServiceRequest> for Auth
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = actix_web::Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = actix_web::Error;
    type Transform = AuthMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(AuthMiddleware { service })
    }
}
```

That's just the factory. You still need the `AuthMiddleware<S>` struct with its `Service` impl. Compare that to axum's `from_fn` - one async function versus two trait implementations and a wrapper struct.

### Warp: filters as middleware

Warp doesn't have a separate middleware concept. Filters are middleware. Want logging? Add a filter. Want auth? Add a filter. Want CORS? There's a filter for that.

```rust
let cors = warp::cors()
    .allow_any_origin()
    .allow_methods(vec!["GET", "POST"]);

let log = warp::log("api");

let routes = create_route
    .with(cors)
    .with(log);
```

The `.with()` method adds a wrapper filter. This is conceptually clean - everything is just filter composition. But the practical limitation is that warp's filter ecosystem is smaller than Tower's. If tower-http has what you need, axum gives you access to it directly. With warp, you might end up writing the filter yourself.

## Error handling

Error handling is where framework design decisions have the most visible impact on your code.

### Axum

Axum errors work through the `IntoResponse` trait. Anything that implements it can be returned from a handler. The idiomatic pattern is a custom error enum:

```rust
use axum::response::{IntoResponse, Response};
use axum::http::StatusCode;

enum AppError {
    NotFound(String),
    Validation(String),
    Internal(anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg),
            AppError::Validation(msg) => (StatusCode::BAD_REQUEST, msg),
            AppError::Internal(err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                format!("internal error: {}", err),
            ),
        };
        (status, message).into_response()
    }
}

// Then handlers return Result<T, AppError>
async fn get_user(Path(id): Path<u64>) -> Result<Json<User>, AppError> {
    let user = find_user(id)
        .await
        .ok_or_else(|| AppError::NotFound(format!("user {} not found", id)))?;
    Ok(Json(user))
}
```

If you've read my [RESTful API design post](/blog/designing-restful-apis-practical-guidelines-beyond-the-theory/), you'll recognize this as a natural fit for RFC 9457 problem details - your `IntoResponse` impl can return structured JSON errors.

### Actix-web

Actix uses the `ResponseError` trait. Your error type defines its own HTTP status and body:

```rust
use actix_web::{HttpResponse, ResponseError};
use std::fmt;

#[derive(Debug)]
enum AppError {
    NotFound(String),
    Validation(String),
    Internal(String),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(msg) => write!(f, "not found: {}", msg),
            AppError::Validation(msg) => write!(f, "validation: {}", msg),
            AppError::Internal(msg) => write!(f, "internal: {}", msg),
        }
    }
}

impl ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        match self {
            AppError::NotFound(msg) => HttpResponse::NotFound().body(msg.clone()),
            AppError::Validation(msg) => HttpResponse::BadRequest().body(msg.clone()),
            AppError::Internal(msg) => {
                HttpResponse::InternalServerError().body(msg.clone())
            }
        }
    }
}
```

Same idea, different trait. The main difference: `ResponseError` requires `Display + Debug`, so your errors must be printable. Axum's `IntoResponse` has no such constraint - you just produce a response.

### Warp

Warp has the most unusual error model. Handlers return `Result<impl Reply, Rejection>`. A `Rejection` is not an HTTP error response - it's a signal that this filter didn't match and warp should try the next one. If no filter matches, the rejection becomes a 500.

To return proper HTTP errors, you need a rejection handler:

```rust
async fn handle_rejection(
    err: warp::Rejection,
) -> Result<impl warp::Reply, std::convert::Infallible> {
    if err.find::<ValidationError>().is_some() {
        Ok(warp::reply::with_status(
            "validation error",
            StatusCode::BAD_REQUEST,
        ))
    } else if err.is_not_found() {
        Ok(warp::reply::with_status(
            "not found",
            StatusCode::NOT_FOUND,
        ))
    } else {
        Ok(warp::reply::with_status(
            "internal error",
            StatusCode::INTERNAL_SERVER_ERROR,
        ))
    }
}

let routes = create_route.recover(handle_rejection);
```

This works, but it centralizes all error handling into one function that pattern-matches on rejection types. As your API grows, this function grows too. The rejection model is powerful for filter composition - a filter can reject a request and the framework tries the next route - but it's awkward for application-level errors where you know exactly what response you want.

## Performance

Everyone wants the benchmark numbers. Here they are, with caveats.

In synthetic benchmarks (TechEmpower-style plaintext and JSON responses), actix-web consistently handles 10-15% more requests per second than axum under heavy load. Actix's custom HTTP implementation, with its own buffer management and zero-copy parsing, gives it an edge in raw throughput. On the TechEmpower composite score, both rank in the top tier across all languages and frameworks.

Axum, running on hyper 1.x, isn't far behind. Where it compensates is memory efficiency - axum tends to use less memory per connection, which matters in containerized deployments with tight memory limits. If you're running 50 instances on Kubernetes and each saves 20MB of RAM, that's a gigabyte of headroom you get back.

Warp, also built on hyper 1.x, benchmarks close to axum. The filter composition layer adds negligible overhead.

**Why you shouldn't over-index on this:** the performance difference between these frameworks is smaller than the difference between a good database query and a bad one. If your endpoint does any real work - database calls, JSON serialization of non-trivial payloads, business logic - the framework overhead is noise. I covered load testing methodology in my [load testing post](/blog/load-testing-your-rust-api-tools-and-methodology/) - if you're comparing frameworks, benchmark your actual workload, not "hello world."

The scenario where raw framework throughput matters: reverse proxies, API gateways, static file servers - anything that sits in the hot path doing minimal per-request work. That's actix-web territory. For everything else, pick the framework that makes your team most productive.

## Ecosystem and community

### Axum

Axum's biggest advantage isn't its own API - it's the Tower ecosystem. Because axum is Tower-native, you get access to:

- [tower-http](https://docs.rs/tower-http/latest/tower_http/) - compression, CORS, tracing, request ID, timeouts, body limits
- [tonic](https://docs.rs/tonic/latest/tonic/) - gRPC, same Tower middleware works for both HTTP and gRPC
- [tower-sessions](https://docs.rs/tower-sessions/latest/tower_sessions/) - session management
- [axum-extra](https://docs.rs/axum-extra/latest/axum_extra/) - typed headers, cookie jar, form handling, caching

Axum is maintained by the tokio team (primarily David Pedersen). Being part of the tokio-rs org means it stays in sync with tokio, hyper, and tower releases. When hyper 1.0 shipped, axum was the first framework to integrate it.

The 0.8 release (January 2025) brought meaningful improvements: the `{param}` path syntax aligning with OpenAPI, native async trait support removing the `#[async_trait]` dependency, and the smarter `Option<T>` extractor I mentioned earlier.

### Actix-web

Actix-web has the longest history. It's been production-ready since 2018. The actix ecosystem includes:

- [actix-cors](https://docs.rs/actix-cors/latest/actix_cors/) - CORS
- [actix-session](https://docs.rs/actix-session/latest/actix_session/) - session management
- [actix-identity](https://docs.rs/actix-identity/latest/actix_identity/) - auth identity
- [actix-ws](https://docs.rs/actix-ws/latest/actix_ws/) - WebSockets
- [actix-multipart](https://docs.rs/actix-multipart/latest/actix_multipart/) - file uploads

The framework survived a maintainer crisis in 2020 (the original author stepped back) and has since rebuilt under new maintainers. It's stable, well-documented, and has the most Stack Overflow answers of any Rust web framework. If you're stuck, someone has probably asked about it before.

The 4.x series brought io-uring support as an experimental feature, which could give it another performance edge on Linux 5.1+.

### Warp

Warp is maintained by Sean McArthur, who also maintains hyper and reqwest - so it's in good hands technically. But the ecosystem is significantly smaller. The 0.4 release (mid 2025) focused on upgrading to hyper 1.x and trimming scope: TLS support was dropped entirely, and multipart/websocket features became opt-in.

Sean himself [said it directly](https://seanmonstar.com/blog/warp-v04/): if you want a "standard, super fast, featureful" framework, use axum. Warp's value proposition is specifically its `Filter` system for functional, type-driven routing.

This isn't a death sentence - warp works fine for what it does. But the signal is clear: new development energy in the Rust web ecosystem flows toward axum.

## When to pick which

**Pick Axum when:**
- You're starting a new project and want the largest ecosystem
- You use (or plan to use) tonic for gRPC alongside REST
- You want Tower middleware compatibility
- Your team values simple handler signatures and readable code
- You're building a typical web API or service

This is the default choice in 2026. Not because the others are bad, but because axum has the strongest combination of ergonomics, ecosystem, and momentum.

**Pick Actix-web when:**
- Raw throughput is your primary constraint (proxies, gateways, real-time data pipelines)
- You're maintaining an existing actix-web codebase (migration to axum has costs)
- You want the most battle-tested option with the longest production track record
- You need io-uring support
- Your team is already familiar with its patterns

If you read the [Rust in production post](/blog/rust-in-production-what-companies-actually-use-it-for/), Cloudflare built Pingora from scratch rather than using any framework - but actix-web's performance profile is closest to what that kind of infrastructure demands.

**Pick Warp when:**
- You genuinely prefer functional composition and want it enforced by the type system
- You're building small, focused services where filter composition is a natural fit
- You don't need a large middleware ecosystem
- You're comfortable writing custom filters for features other frameworks provide out of the box

Be honest with yourself on this one. "I like functional programming" is a valid reason. "My team of five will maintain this for years" might point toward axum's larger community.

## Migration reality check

If you're on actix-web and wondering whether to migrate: probably not, unless you're actively hitting pain points. A working actix-web service doesn't become worse because axum is popular. The framework does its job. Migration costs are real - different handler signatures, different middleware APIs, different error handling patterns. If you read my post on [the adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/), you know the value of abstracting over implementation details. If your business logic is behind trait boundaries, swapping the web framework touches only the HTTP layer. If it's tangled into handlers, a migration is a rewrite.

If you're on warp and feeling the ecosystem squeeze - fewer examples, fewer middleware options, occasional type error nightmares - migrating to axum is relatively smooth. Both sit on hyper 1.x and tokio. Your async code, your database layer, your business logic all carry over. It's mainly the routing and handler signatures that change.

For new projects, start with axum unless you have a specific, measurable reason to choose something else. "Measurable" is the key word. Not "I heard actix is faster" - but "we benchmarked our specific workload and the 12% throughput difference at p99 matters for our SLA." That's a real reason. Everything else is premature optimization.

## The framework matters less than you think

Nobody in framework comparison posts wants to say this: for most Rust web services, the framework is maybe 5% of your codebase. The rest is database access, business logic, serialization, error handling, configuration, and deployment. A well-structured service on any of these three frameworks will outperform a poorly-structured one on the "best" framework.

Pick one, learn it well, build something real. The Rust ecosystem has matured to the point where all three options are solid. The decision that actually matters is shipping.
