+++
title = "The middleware pattern - wrapping behavior around handlers"
date = 2026-03-11
description = "Middleware is more than a web framework feature - it's a general pattern for composing cross-cutting concerns around any handler, from Express to Tower to a from-scratch Rust implementation."

[taxonomies]
tags = ["rust", "design-patterns", "architecture", "tower"]
+++

Every web framework has middleware. Express has `app.use()`. Django has `MIDDLEWARE`. Axum has Tower layers. But middleware isn't really a web concept. It's a general pattern: take a function, wrap it with behavior that runs before and/or after, return a new function with the same shape. Logging, auth, timing, rate limiting, input validation - all of these are cross-cutting concerns that you want to apply uniformly without polluting your core logic.

The idea is old. It shows up in Python decorators, in aspect-oriented programming, in the Unix pipe philosophy. What differs across languages and frameworks is the *mechanism* for composition and the tradeoffs that come with it.

This post breaks down the middleware pattern across different ecosystems, goes deep into Tower's `Service` and `Layer` traits in Rust, and then builds a middleware stack from scratch to show what's actually happening under the abstractions.

<!-- more -->

## The pattern at its core

Strip away every framework and the middleware pattern reduces to one thing: function wrapping.

```
middleware(handler) -> new_handler
```

The new handler has the same signature as the original. It does some work before calling the inner handler, does some work after, and returns the result (possibly modified). That's it. The power comes from *composition* - stacking multiple wrappers, each unaware of the others.

Here's the simplest possible version in pseudocode:

```
fn logging(inner):
    fn wrapped(request):
        log("before")
        response = inner(request)
        log("after")
        return response
    return wrapped

fn timed(inner):
    fn wrapped(request):
        start = now()
        response = inner(request)
        log(now() - start)
        return response
    return wrapped

handler = logging(timed(actual_handler))
```

Call `handler(request)` and you get: log "before", start timer, run actual handler, log elapsed time, log "after". The wrapping order determines execution order, and it matters more than most people expect.

## Express: middleware as a linked list

Express.js popularized middleware for a generation of developers. Its model is a linear chain where each middleware calls `next()` to pass control forward:

```javascript
const express = require('express');
const app = express();

app.use((req, res, next) => {
    console.log(`${req.method} ${req.url}`);
    const start = Date.now();
    next();
    console.log(`completed in ${Date.now() - start}ms`);
});

app.use((req, res, next) => {
    if (!req.headers.authorization) {
        return res.status(401).json({ error: 'unauthorized' });
    }
    next();
});

app.get('/api/data', (req, res) => {
    res.json({ message: 'hello' });
});
```

The key mechanic: `next()` is a callback that invokes the next middleware in the stack. If you don't call it, the chain stops. This gives you short-circuiting for free - the auth middleware above returns a 401 without ever reaching the handler.

But Express middleware is mutable-state-passing. Each middleware shares the same `req` and `res` objects, mutating them in place. You've probably seen `req.user = decoded` in an auth middleware. It works, but it's a stringly-typed convention. Nothing in the type system tells you that `req.user` exists by the time your handler runs. That depends entirely on middleware ordering.

This is the first hint that ordering isn't just an implementation detail - it's part of your application's correctness.

## Python decorators: middleware with syntax sugar

Python decorators are the middleware pattern with dedicated syntax. If you've used closures in Rust (covered in [Closures in Rust - Fn, FnMut, FnOnce demystified](/blog/closures-in-rust-fn-fnmut-fnonce-demystified/)), the mechanics will feel familiar - a decorator is a higher-order function that captures the inner function and returns a wrapper.

```python
import time
import functools

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

def require_auth(func):
    @functools.wraps(func)
    def wrapper(request, *args, **kwargs):
        if not request.get("token"):
            raise PermissionError("no token")
        return func(request, *args, **kwargs)
    return wrapper

@timed
@require_auth
def handle_request(request):
    return {"status": "ok"}
```

The `@` syntax is just sugar for `handle_request = timed(require_auth(handle_request))`. Decorators stack bottom-up: `@require_auth` wraps first (innermost), then `@timed` wraps that result (outermost). So when you call `handle_request`, the timer starts, auth is checked, and then the actual function runs.

This is elegant but has the same problem as Express: nothing enforces that the decorators are in the right order. Swap `@timed` and `@require_auth` and you're timing auth failures too - maybe that's fine, maybe it's a bug. The language doesn't know.

Parameterized decorators add another layer of nesting:

```python
def rate_limit(max_calls, period_seconds):
    def decorator(func):
        calls = []
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            calls[:] = [t for t in calls if now - t < period_seconds]
            if len(calls) >= max_calls:
                raise RuntimeError("rate limited")
            calls.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limit(max_calls=10, period_seconds=60)
def api_endpoint(request):
    return {"data": "value"}
```

Three levels of functions: the factory returns the decorator, the decorator returns the wrapper. This works but the cognitive overhead grows fast when you start stacking parameterized decorators. And in async Python, you need to also handle `async def wrapper` with `await func(...)`, which is yet another axis of complexity.

## Tower: middleware with types

Tower is where the middleware pattern gets serious. Rather than relying on conventions (Express's `next()`) or syntax sugar (Python's `@`), Tower encodes the entire pattern in Rust's type system. Every middleware is a `Service`, every wrapper is a `Layer`, and the compiler verifies that everything fits together.

### The Service trait

At Tower's core is [`Service`](https://docs.rs/tower-service/latest/tower_service/trait.Service.html), defined in the `tower-service` crate:

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(
        &mut self,
        cx: &mut Context<'_>,
    ) -> Poll<Result<(), Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

If you squint, this is just `async fn(Request) -> Result<Response, Error>` with two additions.

First: the associated `Future` type. Because Rust's async functions return opaque futures, and trait methods with `async fn` historically couldn't use `impl Future` in return position, Tower makes the future type explicit. This lets each `Service` implementation return its own concrete future type, enabling the compiler to monomorphize and inline aggressively.

Second: `poll_ready`. This is for backpressure. A service can signal that it's currently at capacity (a connection pool is full, a rate limiter has hit its window) by returning `Poll::Pending` from `poll_ready`. Callers are expected to check `poll_ready` before calling `call`. If you've worked with Tokio's internals (I wrote about this in [Understanding Tokio](/blog/understanding-tokio-the-rust-async-runtime-under-the-hood/)), this follows the same `Poll`-based pattern that drives the entire async ecosystem.

The Tokio blog post ["Inventing the Service trait"](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait) walks through the full design motivation. The short version: `poll_ready` exists because without it, a load balancer can't check which backend has capacity before dispatching a request.

### The Layer trait

A [`Layer`](https://docs.rs/tower/latest/tower/trait.Layer.html) wraps a `Service` to produce a new `Service`:

```rust
pub trait Layer<S> {
    type Service;

    fn layer(&self, inner: S) -> Self::Service;
}
```

That's it. `Layer` is the middleware factory. It takes an inner service and returns a wrapped service. The pattern is identical to Python's decorator - `Layer::layer(inner)` is `decorator(func)` - but the types constrain what's possible. The returned `Self::Service` must itself implement `Service`, which the compiler checks at build time.

### A concrete example: Timeout

Here's what a real Tower middleware looks like. This is a simplified version of [Tower's built-in Timeout](https://github.com/tower-rs/tower/blob/master/tower/src/timeout/mod.rs):

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;
use tower::{Layer, Service};

// The Layer - a factory that knows how to wrap services
#[derive(Clone)]
pub struct TimeoutLayer {
    duration: Duration,
}

impl TimeoutLayer {
    pub fn new(duration: Duration) -> Self {
        TimeoutLayer { duration }
    }
}

impl<S> Layer<S> for TimeoutLayer {
    type Service = Timeout<S>;

    fn layer(&self, inner: S) -> Self::Service {
        Timeout {
            inner,
            duration: self.duration,
        }
    }
}

// The Service - wraps another service with timeout behavior
#[derive(Clone)]
pub struct Timeout<S> {
    inner: S,
    duration: Duration,
}

impl<S, Request> Service<Request> for Timeout<S>
where
    S: Service<Request>,
    S::Error: From<tokio::time::error::Elapsed>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = Pin<Box<dyn Future<Output = Result<S::Response, S::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Request) -> Self::Future {
        let fut = self.inner.call(req);
        let duration = self.duration;
        Box::pin(async move {
            tokio::time::timeout(duration, fut)
                .await
                .map_err(Into::into)?
        })
    }
}
```

Two structs, two trait impls. `TimeoutLayer` is the factory (stores configuration), `Timeout<S>` is the wrapped service (stores the inner service plus config). The `call` method wraps the inner future with `tokio::time::timeout`. The `poll_ready` delegates to the inner service - this middleware doesn't add its own backpressure.

This is more boilerplate than the Python version, no question. But the payoff is real: the compiler guarantees that `Timeout<S>` is a valid `Service` whenever `S` is. You can't accidentally forget to call the inner service. You can't return the wrong type. And because the types flow through, you can compose layers without runtime checks.

### Composing with ServiceBuilder

Tower's [`ServiceBuilder`](https://docs.rs/tower/latest/tower/struct.ServiceBuilder.html) chains layers together:

```rust
use tower::ServiceBuilder;
use tower_http::trace::TraceLayer;
use std::time::Duration;

let service = ServiceBuilder::new()
    .layer(TraceLayer::new_for_http())
    .layer(TimeoutLayer::new(Duration::from_secs(30)))
    .service(my_handler);
```

Layers apply bottom-up, just like Python decorators. `my_handler` gets wrapped by `TimeoutLayer` first, then `TraceLayer` wraps that. So the execution order is: tracing starts, timeout starts, handler runs, timeout checks, tracing ends.

In Axum, you use this same mechanism:

```rust
use axum::{Router, routing::get, middleware};
use tower::ServiceBuilder;
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/api/data", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(TraceLayer::new_for_http())
            .layer(TimeoutLayer::new(Duration::from_secs(30)))
    );
```

Axum doesn't have its own middleware system. It uses Tower directly. Every Axum handler is a `Service`, every Axum `Router` is a `Service`, and layers compose naturally. This is the benefit of standardizing on a single abstraction - the `TraceLayer` from `tower-http` works with Axum, Hyper, Tonic, and anything else built on Tower.

If you read [the adapter pattern in Rust](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/), the principle is the same: define a common interface, implement it for concrete types, and let the abstraction handle composition. The difference is that adapters wrap *external* APIs while middleware wraps *your own* handlers.

## Building a middleware stack from scratch

Let's strip away Tower and build the pattern ourselves. This makes the mechanics explicit.

Our goal: a composable middleware stack for simple request/response handlers, with before and after hooks, short-circuiting, and ordering.

```rust
use std::collections::HashMap;

// A basic request/response model
#[derive(Debug, Clone)]
struct Request {
    path: String,
    headers: HashMap<String, String>,
    body: String,
}

#[derive(Debug, Clone)]
struct Response {
    status: u16,
    body: String,
}

// The handler type - a function from Request to Response
type Handler = Box<dyn Fn(Request) -> Response + Send + Sync>;

// A middleware takes a handler and returns a new handler
type Middleware = Box<dyn Fn(Handler) -> Handler + Send + Sync>;

fn logging_middleware() -> Middleware {
    Box::new(|next: Handler| -> Handler {
        Box::new(move |req: Request| {
            println!("[LOG] --> {} {}", req.path, req.body.len());
            let resp = next(req);
            println!("[LOG] <-- {}", resp.status);
            resp
        })
    })
}

fn timing_middleware() -> Middleware {
    Box::new(|next: Handler| -> Handler {
        Box::new(move |req: Request| {
            let start = std::time::Instant::now();
            let resp = next(req);
            let elapsed = start.elapsed();
            println!("[TIME] {}ms", elapsed.as_millis());
            resp
        })
    })
}

fn auth_middleware(required_token: String) -> Middleware {
    Box::new(move |next: Handler| -> Handler {
        let token = required_token.clone();
        Box::new(move |req: Request| {
            match req.headers.get("authorization") {
                Some(t) if t == &token => next(req),
                _ => Response {
                    status: 401,
                    body: "unauthorized".to_string(),
                },
            }
        })
    })
}
```

Each middleware is a function that takes a `Handler` and returns a new `Handler`. The returned handler captures the original via closure - if you're comfortable with `Fn` trait bounds from [the closures post](/blog/closures-in-rust-fn-fnmut-fnonce-demystified/), this is just `Fn` closures all the way down.

Now the stack:

```rust
struct MiddlewareStack {
    layers: Vec<Middleware>,
}

impl MiddlewareStack {
    fn new() -> Self {
        MiddlewareStack { layers: Vec::new() }
    }

    fn use_middleware(&mut self, m: Middleware) {
        self.layers.push(m);
    }

    fn wrap(self, handler: Handler) -> Handler {
        // Apply in reverse order so that the first middleware
        // added is the outermost wrapper
        self.layers
            .into_iter()
            .rev()
            .fold(handler, |h, middleware| middleware(h))
    }
}

fn main() {
    let mut stack = MiddlewareStack::new();
    stack.use_middleware(logging_middleware());
    stack.use_middleware(timing_middleware());
    stack.use_middleware(auth_middleware("secret-token".to_string()));

    let handler: Handler = Box::new(|req: Request| Response {
        status: 200,
        body: format!("hello from {}", req.path),
    });

    let wrapped = stack.wrap(handler);

    // Successful request
    let mut headers = HashMap::new();
    headers.insert("authorization".to_string(), "secret-token".to_string());
    let req = Request {
        path: "/api/data".to_string(),
        headers,
        body: "{}".to_string(),
    };
    let resp = wrapped(req);
    println!("Response: {} {}", resp.status, resp.body);

    // Unauthorized request
    let req = Request {
        path: "/api/data".to_string(),
        headers: HashMap::new(),
        body: "{}".to_string(),
    };
    let resp = wrapped(req);
    println!("Response: {} {}", resp.status, resp.body);
}
```

Output:

```
[LOG] --> /api/data 2
[TIME] 0ms
[LOG] <-- 200
Response: 200 hello from /api/data
[LOG] --> /api/data 2
[TIME] 0ms
[LOG] <-- 401
Response: 401 unauthorized
```

The `fold` in `wrap` is the key. We iterate the layers in reverse and fold them around the handler. First added = outermost wrapper. So logging wraps timing, which wraps auth, which wraps the handler. Logging sees *every* request and *every* response, including auth failures. Timing only measures from auth check through handler execution.

Notice the auth middleware short-circuits - it returns a 401 `Response` directly without calling `next`. The inner handler never runs. This is the same mechanic as Express's "don't call `next()`" pattern, but expressed through control flow rather than callbacks.

## Ordering matters

Let's prove that ordering changes behavior. Swap timing and auth in the stack:

```rust
let mut stack = MiddlewareStack::new();
stack.use_middleware(logging_middleware());
stack.use_middleware(auth_middleware("secret-token".to_string()));
stack.use_middleware(timing_middleware());
```

Now the execution order is: logging -> auth -> timing -> handler.

For an unauthorized request, auth short-circuits before timing ever starts. The timing middleware never runs at all. In the previous ordering, timing measured the auth check. Now it doesn't. Neither ordering is inherently wrong - it depends on what you're trying to measure. But you need to know which one you have.

This is why every framework that supports middleware has opinions about default ordering. Tower's `ServiceBuilder` applies layers top-to-bottom (outermost first). Express processes `app.use()` calls in registration order. Django processes `MIDDLEWARE` top-to-bottom on the way in and bottom-to-top on the way out. They all address the same fundamental issue: when you have N middleware, there are N! possible orderings, and most of them are wrong.

A common convention that works well in practice:

1. **Tracing/logging** - outermost, sees everything
2. **Metrics/timing** - next, captures total request duration
3. **Rate limiting** - reject over-limit requests before doing any real work
4. **Authentication** - verify identity
5. **Authorization** - check permissions
6. **Input validation** - verify request shape
7. **Handler** - your actual logic

Each layer depends on the ones before it. Auth can't check permissions without an identity. Rate limiting shouldn't count requests that fail auth. Timing should capture the full picture. Logging should see everything, including errors from any layer.

## The before/after split

Some frameworks split middleware into explicit before and after phases instead of wrapping. This is common in frameworks that want to avoid the nested closure model:

```rust
trait BeforeAfter: Send + Sync {
    /// Runs before the handler. Return Err to short-circuit.
    fn before(&self, req: &mut Request) -> Result<(), Response>;

    /// Runs after the handler. Can observe (not modify) the response.
    fn after(&self, req: &Request, resp: &Response);
}

struct LoggingMiddleware;

impl BeforeAfter for LoggingMiddleware {
    fn before(&self, req: &mut Request) -> Result<(), Response> {
        println!("[LOG] --> {}", req.path);
        Ok(())
    }

    fn after(&self, req: &Request, resp: &Response) {
        println!("[LOG] <-- {} for {}", resp.status, req.path);
    }
}

struct AuthMiddleware {
    token: String,
}

impl BeforeAfter for AuthMiddleware {
    fn before(&self, req: &mut Request) -> Result<(), Response> {
        match req.headers.get("authorization") {
            Some(t) if t == &self.token => Ok(()),
            _ => Err(Response {
                status: 401,
                body: "unauthorized".to_string(),
            }),
        }
    }

    fn after(&self, _req: &Request, _resp: &Response) {}
}

fn run_with_middleware(
    middlewares: &[Box<dyn BeforeAfter>],
    handler: &Handler,
    mut req: Request,
) -> Response {
    // Run before hooks in order
    for mw in middlewares {
        if let Err(early_response) = mw.before(&mut req) {
            return early_response;
        }
    }

    let resp = handler(req.clone());

    // Run after hooks in reverse order
    for mw in middlewares.iter().rev() {
        mw.after(&req, &resp);
    }

    resp
}
```

This approach is simpler to reason about - you don't need to trace nested closures to understand execution order. The `before` hooks run top-to-bottom, the handler runs, the `after` hooks run bottom-to-top. It mirrors how call stacks work: first in, last out.

The tradeoff: before/after middleware can't easily wrap async behavior around the handler. You can't start a timer in `before` and read it in `after` without some form of shared state. The wrapping model handles this naturally because the timer variable lives in the closure's scope. Each model has its place.

## Where the models converge

Despite looking different on the surface, all middleware implementations reduce to the same abstract operation:

```
outer(inner(request)) -> response
```

Express uses linked callbacks. Python uses nested closures. Tower uses generic type composition. The before/after model unwraps the nesting into a flat pipeline. But they all solve the same problem: separating cross-cutting concerns from core logic, applying them uniformly, and controlling their execution order.

Tower's approach wins in systems where you need *composability across library boundaries*. Because `Service` is a trait with concrete types, anyone can publish a middleware crate (like [`tower-http`](https://docs.rs/tower-http/latest/tower_http/) with its `TraceLayer`, `CorsLayer`, `CompressionLayer`) and it works with any framework built on Tower. This is the same insight behind the plugin pattern I covered in [implementing a plugin system in Rust](/blog/implementing-a-plugin-system-in-rust/) - a shared interface enables an ecosystem.

Python decorators win for local, application-specific middleware where the overhead of defining types isn't worth it. Need to rate limit one endpoint? A decorator is three lines.

Express's model wins for rapid prototyping where you want to slap middleware on and iterate fast, at the cost of type safety.

The before/after split wins for frameworks that prioritize clarity over flexibility - when you want developers to immediately see the execution model without tracing through wrapper closures.

Pick the model that matches your constraints. For a long-lived Rust service with a team of contributors, Tower's type-level guarantees pay for themselves quickly. For a script or a prototype, a simple closure wrapper does the job.
