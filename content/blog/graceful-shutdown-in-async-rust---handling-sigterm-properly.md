+++
title = "Graceful Shutdown in Async Rust - Handling SIGTERM Properly"
date = 2025-09-21
description = "How to shut down a tokio application without corrupting data or dropping in-flight requests - signals, CancellationToken, drain patterns, and shutdown timeouts."

[taxonomies]
tags = ["rust", "async", "tokio", "devops"]
+++

Your app is humming along, processing requests, writing to the database, flushing metrics. Kubernetes decides it's time to roll out a new version. It sends SIGTERM. Your process dies mid-write. The WAL is half-flushed. A user's payment got charged but the order record never landed. That background job was 95% done and now it'll run again from scratch when the new pod comes up - hopefully idempotently, but you never tested that path.

This is what happens when you ignore graceful shutdown. And in async Rust, where dozens of tasks run concurrently across multiple threads, the blast radius of a hard kill is bigger than you'd expect.

<!-- more -->

## What actually happens when your process gets killed

Three signals matter here, and they behave very differently:

**SIGINT** (signal 2) - sent when you press Ctrl+C in a terminal. The default behavior is to terminate the process. This is the signal you encounter during development.

**SIGTERM** (signal 15) - the polite "please stop" signal. This is what Kubernetes sends when it wants your pod to shut down. It's also what `docker stop`, `systemd stop`, and most process managers send first. The default behavior is termination, but the process can catch it and do cleanup.

**SIGKILL** (signal 9) - the uncatchable kill. The kernel removes your process immediately. No cleanup, no handlers, no last words. You cannot intercept this one.

The important contract: SIGTERM is a request, SIGKILL is an order. The gap between them is your shutdown window.

### The Kubernetes termination sequence

When Kubernetes terminates a pod, the sequence is:

1. Pod enters `Terminating` state
2. preStop hook runs (if configured)
3. SIGTERM sent to PID 1 in the container
4. Kubernetes waits up to `terminationGracePeriodSeconds` (default: **30 seconds**)
5. SIGKILL sent to any remaining processes

That 30-second window is all you get. If your app hasn't exited by then, the kernel rips it out. Same story with Docker (`docker stop` sends SIGTERM, waits 10 seconds, then SIGKILL).

This means your shutdown logic needs to be fast and bounded. An unbounded drain that waits for all connections to close naturally is a recipe for hitting the SIGKILL deadline.

## Listening for signals in tokio

If you read the [tokio internals post](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/), you know the runtime has a signal driver layer that wraps the I/O driver. That's what makes async signal handling possible.

### Ctrl+C - the simple case

Tokio provides `tokio::signal::ctrl_c()` that resolves when the process receives SIGINT:

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    // Spawn your application work
    let app_handle = tokio::spawn(run_server());

    // Wait for Ctrl+C
    signal::ctrl_c().await.expect("failed to listen for ctrl+c");
    println!("Received Ctrl+C, shutting down...");

    // Trigger shutdown...
    app_handle.abort();
}
```

This works on all platforms. But it only handles SIGINT. In production, you need SIGTERM too.

### SIGTERM - the production signal

SIGTERM handling requires the Unix-specific signal API:

```rust
use tokio::signal::unix::{signal, SignalKind};

async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    let terminate = async {
        signal(SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    tokio::select! {
        _ = ctrl_c => println!("Received SIGINT"),
        _ = terminate => println!("Received SIGTERM"),
    }
}
```

Two things to note. First, `signal(SignalKind::terminate())` installs a process-wide signal handler that replaces the default behavior. Even if you drop the `Signal` instance later, tokio keeps the handler installed for the lifetime of the process. This is documented in [`tokio::signal::unix::Signal`](https://docs.rs/tokio/latest/tokio/signal/unix/struct.Signal.html).

Second, `recv()` is [cancel-safe](https://docs.rs/tokio/latest/tokio/signal/unix/struct.Signal.html#cancel-safety). You can use it inside `select!` loops without losing signals.

For cross-platform builds, guard the Unix-specific code:

```rust
async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal(SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

On Windows, `std::future::pending()` effectively disables the SIGTERM branch since Windows doesn't have POSIX signals.

## Broadcasting shutdown to tasks

Detecting the signal is the easy part. The real challenge: telling every spawned task to stop what it's doing, finish cleanup, and exit. There are three main approaches, and they have meaningful tradeoffs.

### Approach 1: `tokio::sync::watch` channel

A watch channel holds a single value that receivers can observe. Flip the value to `true`, and every receiver sees it:

```rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    // Pass a receiver clone to each task
    for i in 0..4 {
        let mut rx = shutdown_rx.clone();
        tokio::spawn(async move {
            loop {
                tokio::select! {
                    _ = rx.changed() => {
                        if *rx.borrow() {
                            println!("Worker {i} shutting down");
                            // cleanup...
                            return;
                        }
                    }
                    _ = do_work() => {}
                }
            }
        });
    }

    shutdown_signal().await;
    let _ = shutdown_tx.send(true);
    // wait for tasks...
}
```

The watch channel is efficient for this - one sender, many receivers, and `changed()` is cancel-safe. But you're carrying around a `watch::Receiver<bool>`, which is a bit clunky. You also need to check the value after `changed()` resolves because watch delivers the *latest* value, not a diff.

### Approach 2: `tokio::sync::broadcast` channel

Similar idea but with message semantics:

```rust
use tokio::sync::broadcast;

let (shutdown_tx, _) = broadcast::channel::<()>(1);

// In each task:
let mut rx = shutdown_tx.subscribe();
tokio::select! {
    _ = rx.recv() => { /* shutdown */ }
    _ = do_work() => {}
}

// Trigger shutdown:
let _ = shutdown_tx.send(());
```

Broadcast channels are heavier than watch (they maintain a queue per receiver) and have a capacity you need to think about. For a simple shutdown signal, this is overkill.

### Approach 3: `CancellationToken` (recommended)

[`CancellationToken`](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html) from `tokio-util` is purpose-built for this exact problem. It's a lightweight, cloneable handle that supports hierarchical cancellation:

```rust
use tokio_util::sync::CancellationToken;

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();

    for i in 0..4 {
        let token = token.clone();
        tokio::spawn(async move {
            tokio::select! {
                _ = token.cancelled() => {
                    println!("Worker {i}: received shutdown signal");
                    cleanup(i).await;
                    println!("Worker {i}: cleanup done");
                }
                _ = run_worker(i) => {}
            }
        });
    }

    shutdown_signal().await;
    token.cancel();
    // wait for tasks to finish...
}
```

Why is this better than channels?

**Hierarchical cancellation.** You can create child tokens that get cancelled when the parent is cancelled, but cancelling a child doesn't affect the parent. This lets you model subsystem shutdown:

```rust
let root = CancellationToken::new();

// HTTP server subsystem
let http_token = root.child_token();

// Background job subsystem
let jobs_token = root.child_token();

// Cancel just the job subsystem
jobs_token.cancel();
// http_token is still active

// Cancel everything
root.cancel();
// Both http_token and jobs_token are now cancelled
```

**`run_until_cancelled` convenience.** Instead of `select!` boilerplate:

```rust
let result = token.run_until_cancelled(async {
    // your work here
    process_batch().await
}).await;

match result {
    Some(output) => println!("completed: {output:?}"),
    None => println!("cancelled before completion"),
}
```

**`DropGuard` for safety.** If the code that calls `cancel()` might panic or bail early, a drop guard ensures cancellation still happens:

```rust
let token = CancellationToken::new();
let guard = token.clone().drop_guard();

// If this function panics, the guard is dropped,
// which cancels the token automatically
risky_operation().await;

// Disarm the guard if everything went well
guard.disarm();
```

**Zero-cost when not cancelled.** Calling `cancelled()` returns a future that just checks an `AtomicBool` internally. No channels, no allocations on the hot path.

Internally, `CancellationToken` is backed by a tree of nodes ([source](https://github.com/tokio-rs/tokio/blob/master/tokio-util/src/sync/cancellation_token/tree_node.rs)). Each node has a parent link and a list of children. When you call `cancel()`, it walks the tree and wakes all registered wakers. Cloning a token just increments a reference count on the same node; creating a child token allocates a new node and links it to the parent.

## Waiting for tasks to finish

Signaling shutdown is half the battle. You also need to wait for in-flight work to actually complete. There are two patterns here.

### Pattern 1: `JoinSet` for a known set of tasks

If you spawn a fixed set of tasks at startup, collect their handles:

```rust
use tokio::task::JoinSet;

let mut tasks = JoinSet::new();
let token = CancellationToken::new();

for i in 0..4 {
    let token = token.clone();
    tasks.spawn(async move {
        tokio::select! {
            _ = token.cancelled() => {
                flush_buffers(i).await;
            }
            _ = run_worker(i) => {}
        }
    });
}

// Wait for shutdown signal
shutdown_signal().await;
token.cancel();

// Wait for all tasks to finish cleanup
while let Some(result) = tasks.join_next().await {
    if let Err(e) = result {
        eprintln!("Task failed during shutdown: {e}");
    }
}
println!("All tasks exited cleanly");
```

I covered `JoinSet` in the [tokio post](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/) - remember that dropping a `JoinSet` aborts all tasks in it, so don't let it go out of scope before draining.

### Pattern 2: `TaskTracker` for dynamic task spawning

When tasks are spawned dynamically (e.g., one per incoming HTTP request), you can't pre-build a `JoinSet`. [`TaskTracker`](https://docs.rs/tokio-util/latest/tokio_util/task/struct.TaskTracker.html) from tokio-util handles this:

```rust
use tokio_util::task::TaskTracker;

let tracker = TaskTracker::new();
let token = CancellationToken::new();

// In your accept loop:
loop {
    tokio::select! {
        Ok((stream, addr)) = listener.accept() => {
            let token = token.clone();
            tracker.spawn(async move {
                handle_connection(stream, addr, token).await;
            });
        }
        _ = token.cancelled() => break,
    }
}

// Stop accepting, close the tracker, wait for in-flight requests
tracker.close();
tracker.wait().await;
```

`tracker.close()` signals that no more tasks will be spawned. `tracker.wait()` returns a future that resolves when all tracked tasks complete *and* the tracker is closed. Without `close()`, `wait()` would hang forever.

## Putting it all together: a production shutdown pattern

Here's the full pattern with signal handling, CancellationToken propagation, connection draining, and a shutdown timeout:

```rust
use std::time::Duration;
use tokio::net::TcpListener;
use tokio::signal;
use tokio_util::sync::CancellationToken;
use tokio_util::task::TaskTracker;
use tracing::{info, warn, error};

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let token = CancellationToken::new();
    let tracker = TaskTracker::new();

    // Start the server
    let server_token = token.clone();
    let server_tracker = tracker.clone();
    let server = tokio::spawn(async move {
        run_server(server_token, server_tracker).await;
    });

    // Start background workers
    let worker_token = token.child_token();
    let worker_handle = tokio::spawn(async move {
        run_background_worker(worker_token).await;
    });

    // Wait for shutdown signal
    shutdown_signal().await;
    info!("Shutdown signal received, starting graceful shutdown");

    // Phase 1: Signal all tasks to stop
    token.cancel();

    // Phase 2: Wait for in-flight work with a timeout
    let shutdown = async {
        // Wait for the server to stop accepting
        let _ = server.await;

        // Close the tracker - no new tasks after this
        tracker.close();

        // Wait for all in-flight requests to finish
        tracker.wait().await;

        // Wait for background workers
        let _ = worker_handle.await;
    };

    let timeout = Duration::from_secs(25);
    match tokio::time::timeout(timeout, shutdown).await {
        Ok(()) => info!("Graceful shutdown completed"),
        Err(_) => warn!(
            "Shutdown timed out after {}s, forcing exit",
            timeout.as_secs()
        ),
    }
}

async fn run_server(token: CancellationToken, tracker: TaskTracker) {
    let listener = TcpListener::bind("0.0.0.0:8080")
        .await
        .expect("failed to bind");

    info!("Server listening on :8080");

    loop {
        tokio::select! {
            biased;

            _ = token.cancelled() => {
                info!("Server: stop accepting new connections");
                break;
            }

            result = listener.accept() => {
                match result {
                    Ok((stream, addr)) => {
                        let token = token.child_token();
                        tracker.spawn(async move {
                            if let Err(e) = handle_connection(stream, addr, token).await {
                                error!("Connection error from {addr}: {e}");
                            }
                        });
                    }
                    Err(e) => {
                        error!("Accept error: {e}");
                    }
                }
            }
        }
    }
}

async fn handle_connection(
    stream: tokio::net::TcpStream,
    addr: std::net::SocketAddr,
    token: CancellationToken,
) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    // Process requests on this connection
    // When token is cancelled, finish the current request but don't
    // accept new ones on this connection
    loop {
        tokio::select! {
            biased;

            _ = token.cancelled() => {
                info!("Connection {addr}: draining");
                // Finish any in-progress response, then close
                break;
            }

            request = read_request(&stream) => {
                match request {
                    Some(req) => {
                        // Process the request fully - don't abandon mid-response
                        process_and_respond(&stream, req).await?;
                    }
                    None => break, // Client disconnected
                }
            }
        }
    }
    Ok(())
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
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
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

A few design decisions worth explaining:

**`biased;` in the select.** I covered select's random ordering in the [tokio post](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/). For shutdown, you want the cancellation check to always take priority. `biased;` makes branches evaluate top-to-bottom, so `token.cancelled()` is checked before `listener.accept()` on every iteration. If a shutdown signal arrives, you stop accepting immediately rather than potentially taking one more connection.

**25-second timeout, not 30.** Kubernetes gives you 30 seconds by default. But your app needs a few seconds of buffer for the signal to propagate, for the preStop hook, and for the kernel to clean up. Setting your internal timeout to 25 seconds gives you a 5-second safety margin before the SIGKILL arrives.

**Child tokens for connections.** Each connection gets `token.child_token()` instead of `token.clone()`. Functionally, for shutdown purposes, they behave the same (parent cancellation propagates to children). But child tokens give you the option to cancel individual connections independently if you ever need that - a clone doesn't.

**TaskTracker closed after cancel.** The `tracker.close()` call must happen *after* `token.cancel()` triggers the server's accept loop to break. If you close the tracker while the server is still accepting, new spawn calls would panic.

## Draining connections properly

The trickiest part of shutdown is deciding what "finish in-flight work" actually means. There are levels:

**Level 1: Stop accepting, kill everything.** The nuclear option. Call `token.cancel()` and let all tasks get dropped. Fast, but you'll cut off active requests mid-response. Users see connection resets.

**Level 2: Stop accepting, finish current requests.** This is the standard. New TCP connections get refused (or get a 503 behind a load balancer), but active requests run to completion. The timeout acts as a safety net.

**Level 3: Stop accepting, drain existing connections.** HTTP/2 and gRPC connections are long-lived and multiplexed. "Finish current requests" might mean waiting for a stream that's been idle for minutes. For these, send a GOAWAY frame and give clients a short window to wrap up.

Axum supports level 2 out of the box with [`with_graceful_shutdown`](https://docs.rs/axum/latest/axum/serve/struct.WithGracefulShutdown.html):

```rust
use axum::{routing::get, Router};
use tokio::net::TcpListener;

let app = Router::new().route("/", get(|| async { "ok" }));
let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await
    .unwrap();
```

When the signal fires, axum stops accepting new connections and waits for in-flight requests to complete. But there's a catch: if a client holds a connection open without sending a request, axum waits forever. Add a `TimeoutLayer` from `tower-http` as a safety net:

```rust
use tower_http::timeout::TimeoutLayer;
use std::time::Duration;

let app = Router::new()
    .route("/", get(handler))
    .layer(TimeoutLayer::new(Duration::from_secs(10)));
```

## Flushing buffers and background work

HTTP connections are the visible part, but shutdown also means:

**Flushing log/metrics buffers.** If you're using a buffered tracing subscriber or a metrics exporter with a batch interval, those buffers need to flush before exit. Most tracing layers flush on drop, but verify this. For OpenTelemetry, call `shutdown()` on the tracer provider:

```rust
// In your shutdown sequence, after all tasks are done:
opentelemetry::global::shutdown_tracer_provider();
```

**Completing database transactions.** A transaction in progress when SIGTERM arrives should either commit or rollback. If you're using connection pools (sqlx, deadpool), dropping the pool will close connections, but in-progress transactions get rolled back by the database server - which is usually the right thing.

**Draining message queues.** If your app consumes from a queue (Kafka, RabbitMQ, Redis streams), stop polling for new messages but finish processing the current batch before acknowledging.

**Stopping scheduled tasks.** Background jobs running on an interval should check the cancellation token at the top of each iteration:

```rust
async fn run_background_worker(token: CancellationToken) {
    let mut interval = tokio::time::interval(Duration::from_secs(60));

    loop {
        tokio::select! {
            biased;

            _ = token.cancelled() => {
                info!("Background worker: shutting down");
                break;
            }

            _ = interval.tick() => {
                // Run the job. If it's long, check the token periodically
                // inside the job too.
                if let Err(e) = run_cleanup_job(&token).await {
                    error!("Cleanup job failed: {e}");
                }
            }
        }
    }
}

async fn run_cleanup_job(token: &CancellationToken) -> anyhow::Result<()> {
    let items = fetch_stale_items().await?;

    for chunk in items.chunks(100) {
        // Check between chunks - don't hold up shutdown for a huge batch
        if token.is_cancelled() {
            info!("Cleanup job interrupted, {} items remaining", items.len());
            break;
        }
        delete_batch(chunk).await?;
    }

    Ok(())
}
```

The key insight: `token.is_cancelled()` is a synchronous, non-blocking check (it reads an `AtomicBool`). Use it for quick polls inside CPU-bound or batch loops where you can't yield to `select!`.

## Why a shutdown timeout is non-negotiable

Without a timeout, your shutdown sequence has an implicit assumption: all tasks will terminate in finite time. That assumption is wrong in practice.

A database query might be stuck waiting on a lock. A client might hold a TCP connection without closing it. A background job might be processing a million-row batch. An external API call might hang because the remote server is down.

Any of these turns your "graceful" shutdown into a hang, and then Kubernetes sends SIGKILL anyway - but now you've wasted 30 seconds doing nothing.

Always wrap your shutdown in `tokio::time::timeout`:

```rust
let timeout = Duration::from_secs(25);

match tokio::time::timeout(timeout, shutdown_sequence()).await {
    Ok(()) => {
        info!("Clean shutdown completed");
        std::process::exit(0);
    }
    Err(_) => {
        warn!("Shutdown timed out, exiting forcefully");
        std::process::exit(1);
    }
}
```

Exit code matters. `exit(0)` tells the orchestrator "I shut down cleanly." `exit(1)` says "something went wrong" - which might trigger alerts or affect rollout decisions.

## Common mistakes

**Forgetting SIGTERM.** If you only handle `ctrl_c()`, your app works fine in development but ignores `docker stop` and `kubectl delete pod` in production. It sits there for 30 seconds until SIGKILL arrives. I've seen this in production more times than I'd like.

**Shutdown handler that panics.** If your shutdown code panics (maybe a channel receiver was already dropped), the process crashes without completing the drain. Use `.ok()` or explicit error handling in shutdown paths:

```rust
// Don't do this:
shutdown_tx.send(true).unwrap(); // panics if receiver dropped

// Do this:
if shutdown_tx.send(true).is_err() {
    // Receiver already dropped, tasks are gone. That's fine.
}
```

**Not passing the token deep enough.** The cancellation token needs to reach every long-running loop in your application. If a task spawns sub-tasks that don't receive the token, those sub-tasks won't know about shutdown and will keep running until the timeout (or SIGKILL).

**Double signal handling.** Pressing Ctrl+C twice typically means "I really want to quit now." You can handle this by tracking the first signal and doing an immediate exit on the second:

```rust
async fn shutdown_signal() {
    // First signal: graceful shutdown
    signal::ctrl_c().await.expect("failed to install handler");
    info!("Received shutdown signal. Press Ctrl+C again to force exit.");

    // Second signal: hard exit
    tokio::spawn(async {
        signal::ctrl_c().await.expect("failed to install handler");
        warn!("Forced exit");
        std::process::exit(1);
    });
}
```

## Dependencies and versions

Everything in this post uses:

- [`tokio`](https://crates.io/crates/tokio) 1.50.0 - `features = ["full"]` for signal handling, networking, and timers
- [`tokio-util`](https://crates.io/crates/tokio-util) 0.7.18 - for `CancellationToken` and `TaskTracker`

```toml
[dependencies]
tokio = { version = "1.50", features = ["full"] }
tokio-util = { version = "0.7", features = ["full"] }
```

If you want a higher-level abstraction, [`tokio-graceful-shutdown`](https://crates.io/crates/tokio-graceful-shutdown) provides a subsystem-based approach where you define a tree of subsystems that get shut down in order. It's a good fit if you have a complex service with many independent subsystems that have specific shutdown ordering requirements. For most applications, the CancellationToken + TaskTracker combo is enough.

## The checklist

Before you ship to production, verify:

- [ ] Your app handles both SIGINT and SIGTERM
- [ ] Every spawned task receives a cancellation token or shutdown receiver
- [ ] Long-running loops check for cancellation between iterations
- [ ] In-flight HTTP requests complete before the server exits
- [ ] Database connections are returned to the pool (or the pool is dropped cleanly)
- [ ] Log and metrics buffers are flushed
- [ ] A timeout caps the total shutdown duration below your orchestrator's grace period
- [ ] A second Ctrl+C forces an immediate exit during development
- [ ] Exit codes are set correctly (0 for clean, 1 for timeout/error)

Graceful shutdown isn't glamorous. It doesn't show up in benchmarks or feature lists. But it's the difference between "deploy went fine" and "deploy corrupted 47 user sessions." Get it right once, build it into your server skeleton, and never think about it again.
