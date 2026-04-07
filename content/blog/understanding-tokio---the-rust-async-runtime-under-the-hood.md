+++
title = "Understanding Tokio - the Rust Async Runtime Under the Hood"
date = 2026-04-07
description = "A deep look at how tokio actually works: the I/O driver, work-stealing scheduler, spawn vs spawn_blocking, select!, and the gotchas that bite everyone."

[taxonomies]
tags = ["rust", "async", "tokio", "performance"]
+++

Most Rust devs interact with tokio through `#[tokio::main]` and `.await`. You add the attribute, sprinkle some `async`, and things work. But what actually happens between your `.await` and the kernel returning bytes from a socket? That's what this post is about.

We're going to tear apart the tokio runtime - the I/O driver, the scheduler, the thread pools - and look at the real code that makes it tick. Not the docs version, the source code version.

<!-- more -->

## What `#[tokio::main]` actually builds

When you write this:

```rust
#[tokio::main]
async fn main() {
    println!("hello");
}
```

The macro expands to roughly:

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            println!("hello");
        })
}
```

That `build()` call constructs a layered stack of drivers. Not one monolithic event loop - a decorator chain where each layer wraps the one below:

1. **I/O Driver** - wraps `mio::Poll`, does the actual `epoll_wait`/`kevent` syscalls
2. **Signal Driver** - wraps the I/O driver to handle Unix signals
3. **Process Driver** - wraps the signal driver for child process management
4. **Time Driver** - wraps everything above, manages timers and `tokio::time::sleep`

You can see this layering in [`tokio/src/runtime/driver.rs`](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/driver.rs). Each driver delegates to the inner one for I/O, then adds its own functionality on top.

On top of the driver stack sits the scheduler. For the default multi-thread runtime, that's a work-stealing scheduler with one worker per CPU core. For `current_thread`, it's a single-threaded run loop on whatever thread called `block_on()`.

## The I/O driver - from `epoll_wait` to task wakeup

The core question: how does a kernel event (socket becomes readable) turn into your async function resuming?

Tokio doesn't talk to the kernel directly. It goes through [mio](https://github.com/tokio-rs/mio) (version 1.0.1 as of tokio 1.50.0), which abstracts over platform-specific I/O multiplexing:

- **Linux**: `epoll` (edge-triggered)
- **macOS/BSD**: `kqueue`
- **Windows**: IOCP (completion-based, needs an adaptation layer since the paradigm is fundamentally different from readiness-based)

The I/O driver ([`tokio/src/runtime/io/driver.rs`](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/io/driver.rs)) wraps `mio::Poll` and `mio::Events`. Here's the flow:

```
Your code:     stream.read(&mut buf).await
                        |
                        v
tokio TcpStream:  checks readiness flags (atomic load)
                        |
            +-----------+-----------+
            |                       |
        Ready?                  Not ready?
            |                       |
            v                       v
    do the syscall           register Waker with ScheduledIo
    (non-blocking read)      return Poll::Pending
            |                       |
            v                       v
    return Poll::Ready       task gets parked
                                    |
                                    v
                             driver calls mio::Poll::poll()
                             (epoll_wait / kevent syscall)
                                    |
                                    v
                             event arrives, driver looks up
                             ScheduledIo via mio token (O(1) slab)
                                    |
                                    v
                             updates atomic readiness flags
                             calls Waker::wake()
                                    |
                                    v
                             task pushed back onto scheduler queue
                             (gets polled again, this time Ready)
```

The important detail: the driver doesn't poll continuously. It checks for I/O events every **61 task polls** (the `event_interval` default). This means if the scheduler is busy running tasks, I/O events wait up to 61 ticks before being noticed. You can tune this with `runtime::Builder::event_interval()`, but the default balances throughput vs latency well for most workloads.

On Linux, mio's `Events` struct is essentially an array of `struct epoll_event` - near-zero abstraction cost. The edge-triggered mode means the kernel only notifies once per state change, so tokio must drain the socket fully or re-register interest.

## The work-stealing scheduler

This is where tokio gets its performance. The multi-thread scheduler lives in [`tokio/src/runtime/scheduler/multi_thread/`](https://github.com/tokio-rs/tokio/tree/master/tokio/src/runtime/scheduler/multi_thread), and the key files are `worker.rs` (the main loop) and `queue.rs` (the lock-free queue).

### Three-tier queue hierarchy

Each worker thread has three places to find work:

**1. LIFO slot** - a single-task slot optimized for message-passing patterns. When task A sends a message to task B, B gets placed here so it runs immediately on the same worker. This keeps the data hot in cache. But to prevent starvation, a worker only polls the LIFO slot for `MAX_LIFO_POLLS_PER_TICK` (3) consecutive times before moving on.

**2. Per-worker local queue** - a fixed-size ring buffer holding **256 tasks**. The implementation in `queue.rs` uses `AtomicU32` for head and tail pointers. On x86, pushing to your own local queue is zero-CAS - just an Acquire load plus a Release store. Stealing from another worker's queue requires a compare-and-swap.

**3. Global injection queue** - a mutex-protected intrusive linked list. External spawns (from non-tokio threads) and overflow from full local queues land here.

### The worker loop

Simplified from `Context::run()` in `worker.rs`:

```
loop {
    tick += 1;
    run_maintenance_if_needed(tick);

    // 1. Check LIFO slot (hot path for message passing)
    if let Some(task) = lifo_slot.take() {
        run(task);
        continue;
    }

    // 2. Pop from local queue
    if let Some(task) = local_queue.pop() {
        run(task);
        continue;
    }

    // 3. Try stealing from a random sibling
    if let Some(task) = steal_from_sibling() {
        run(task);
        continue;
    }

    // 4. Check global queue
    if let Some(task) = global_queue.pop() {
        run(task);
        continue;
    }

    // 5. Nothing to do - park (sleep)
    park();
}
```

The work-stealing is where it gets clever. When a worker runs out of tasks, it picks a random starting index (using a fast PRNG) and tries to steal from siblings. It doesn't steal just one task - it takes **half** of the victim's local queue. This amortizes the cost of the atomic CAS operations across multiple tasks.

There's also a throttle: only half the workers can be "searching" at the same time. This prevents a thundering herd when work is scarce.

The global queue gets checked periodically even when the local queue has work. For the multi-thread runtime, the interval is dynamically computed to target **10ms between global queue checks**. This prevents tasks spawned from external threads from being starved.

### Performance numbers

When tokio rewrote the scheduler (documented in their [scheduler blog post](https://tokio.rs/blog/2019-10-scheduler)), the numbers were dramatic:

| Benchmark | Before | After | Improvement |
|-----------|--------|-------|-------------|
| chained_spawn | 2,019,796 ns | 168,854 ns | **12x** |
| ping_pong | 1,279,948 ns | 562,659 ns | **2.3x** |
| Hyper HTTP (1 thread, 50 conn) | 113,923 req/s | 152,259 req/s | **34%** |

Most of the gains came from the LIFO slot and better cache locality.

## Cooperative preemption - the 128-operation budget

Here's something most people don't know: tokio enforces fairness at runtime by giving each task a **budget of 128 operations per scheduling tick**.

Every time your task `.await`s a tokio resource - socket read, channel receive, mutex lock, timer - it decrements the budget counter. When the budget hits zero, all tokio resources start returning `Poll::Pending` even if they have data ready. The task yields back to the scheduler, and other tasks get a chance to run.

This is implemented in [`tokio/src/runtime/coop.rs`](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/coop.rs). The budget is stored in a thread-local and decremented via `poll_proceed()` calls inside every tokio I/O and sync primitive.

Why does this matter? Without it, a task doing `loop { socket.read(&mut buf).await; }` on a socket with constant incoming data would monopolize the worker thread forever. Other tasks would starve. The budget system breaks this by forcing a yield every 128 operations. According to [tokio's preemption blog post](https://tokio.rs/blog/2020-04-preemption), this reduced tail latencies by roughly 3x.

This is also why you shouldn't use non-tokio I/O primitives inside async tasks - they don't participate in the budget system, so they bypass cooperative scheduling entirely.

## Multi-threaded vs current_thread runtime

```rust
// Multi-threaded (default)
#[tokio::main]
async fn main() { }

// Equivalent to:
#[tokio::main(flavor = "multi_thread", worker_threads = 8)]
async fn main() { }

// Single-threaded
#[tokio::main(flavor = "current_thread")]
async fn main() { }
```

The differences go deeper than "one thread vs many":

| | `multi_thread` | `current_thread` |
|---|---|---|
| Worker threads | N (default = CPU cores) | 0 (runs on caller) |
| Work stealing | Yes | No |
| `Send` bound on spawned futures | Required | Not required (with `LocalSet`) |
| Global queue check interval | Dynamic (~10ms) | Every 31 ticks (fixed) |
| Cross-thread sync overhead | CAS on queues | None |
| Thread name | `tokio-rt-worker` | Caller's thread |

`current_thread` is not just "multi_thread with 1 worker." It's a fundamentally simpler scheduler with zero atomic operations on the hot path. No lock-free queues, no steal attempts, no worker coordination. The source lives in [`tokio/src/runtime/scheduler/current_thread/`](https://github.com/tokio-rs/tokio/tree/master/tokio/src/runtime/scheduler/current_thread).

When to use `current_thread`:
- CLI tools where you need a few async operations
- WASM targets (no threads available)
- Tests (predictable single-threaded execution)
- When your task count is small and you want less overhead

When to use `multi_thread`:
- Servers handling concurrent connections
- Anything CPU-bound-ish that benefits from parallelism
- When you have many independent tasks

The default `worker_threads` count equals the number of CPU cores. You can override it:

```rust
tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)
    .build()
    .unwrap()
```

Or via the `TOKIO_WORKER_THREADS` environment variable.

## `spawn` vs `spawn_blocking`

This distinction trips people up, and I touched on it briefly in the [load testing post](/blog/load-testing-your-rust-api---tools-and-methodology/) when talking about bcrypt hashing. Time to explain what's actually happening underneath.

### `tokio::spawn` - run a future on the async scheduler

```rust
let handle = tokio::spawn(async {
    let resp = reqwest::get("https://example.com").await?;
    Ok::<_, reqwest::Error>(resp.text().await?)
});

let body = handle.await??;
```

The future runs on one of the worker threads. It must be `Send + 'static` because it might move between threads via work stealing. The critical rule: **never block inside a spawned future**. Every nanosecond you block, you're holding a worker thread hostage - no other tasks can run on it.

### `tokio::task::spawn_blocking` - run a closure on a dedicated thread pool

```rust
let hash = tokio::task::spawn_blocking(move || {
    bcrypt::hash(&password, 12)
}).await?;
```

This runs on a completely separate thread pool, not the async workers. The blocking pool details from [`tokio/src/runtime/blocking/pool.rs`](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/blocking/pool.rs):

- **Max threads**: 512 (default, configurable via `max_blocking_threads()`)
- **Spawn behavior**: threads created on-demand, not at startup
- **Keep-alive**: idle threads exit after **10 seconds**
- **When full**: tasks queue in a FIFO until a thread becomes available
- **Cannot be cancelled**: calling `abort()` on a blocking task's `JoinHandle` is a no-op while the closure is running. It only prevents the result from being delivered.

```rust
// Configuring the blocking pool
tokio::runtime::Builder::new_multi_thread()
    .max_blocking_threads(64)
    .thread_keep_alive(Duration::from_secs(30))
    .build()
    .unwrap()
```

Important gotcha: `spawn_blocking` tasks don't have a timeout. If your blocking closure hangs forever, the thread is gone forever. The runtime's `shutdown_timeout()` is the only safety net:

```rust
let rt = tokio::runtime::Builder::new_multi_thread()
    .enable_all()
    .build()
    .unwrap();

rt.shutdown_timeout(Duration::from_secs(10));
// Any blocking tasks still running after 10s are abandoned
```

## JoinHandle and JoinSet

### JoinHandle

Every `spawn` and `spawn_blocking` returns a `JoinHandle<T>`:

```rust
let handle: JoinHandle<String> = tokio::spawn(async {
    "result".to_string()
});

// Await the result
match handle.await {
    Ok(val) => println!("got: {val}"),
    Err(e) if e.is_cancelled() => println!("task was cancelled"),
    Err(e) if e.is_panic() => println!("task panicked"),
    Err(e) => println!("task failed: {e}"),
}
```

Key methods:
- `handle.abort()` - cancel the task (sets a flag, task cancelled at next `.await` point)
- `handle.abort_handle()` - returns a cloneable `AbortHandle` for remote cancellation
- `handle.is_finished()` - non-blocking check

### JoinSet - managing groups of tasks

`JoinSet` is the right way to manage dynamic groups of spawned tasks:

```rust
use tokio::task::JoinSet;

async fn fetch_all(urls: Vec<String>) -> Vec<String> {
    let mut set = JoinSet::new();

    for url in urls {
        set.spawn(async move {
            reqwest::get(&url).await.unwrap().text().await.unwrap()
        });
    }

    let mut results = Vec::new();
    while let Some(res) = set.join_next().await {
        match res {
            Ok(body) => results.push(body),
            Err(e) => eprintln!("task failed: {e}"),
        }
    }
    results
}
```

As of tokio 1.49.0, `JoinSet` implements `Extend` and `FromIterator`, so you can do:

```rust
let mut set: JoinSet<String> = urls
    .into_iter()
    .map(|url| async move {
        reqwest::get(&url).await.unwrap().text().await.unwrap()
    })
    .collect();
```

Critical behavior: **when a `JoinSet` is dropped, all tasks in it are immediately aborted**. This is usually what you want (structured concurrency), but be aware of it.

Other useful methods:
- `join_all()` - wait for everything, returns `Vec<T>`, panics on first `JoinError`
- `try_join_next()` - non-blocking poll, returns `None` if no task is ready yet
- `abort_all()` - cancel everything
- `detach_all()` - detach all tasks (they keep running even after `JoinSet` drops)

## `tokio::select!` - racing futures

`select!` polls multiple futures concurrently and returns when the first one completes:

```rust
use tokio::time::{sleep, Duration};
use tokio::sync::mpsc;

async fn process(mut rx: mpsc::Receiver<String>) {
    loop {
        tokio::select! {
            Some(msg) = rx.recv() => {
                println!("got message: {msg}");
            }
            _ = sleep(Duration::from_secs(30)) => {
                println!("no message in 30s, shutting down");
                break;
            }
        }
    }
}
```

### How it works internally

The [`select!` macro](https://github.com/tokio-rs/tokio/blob/master/tokio/src/macros/select.rs) expands to a single `poll` function that:

1. Evaluates all preconditions (`if` guards)
2. Generates a random permutation of enabled branches (to prevent starvation)
3. Polls each branch in that random order
4. First branch returning `Poll::Ready` wins - its handler runs, all other futures are dropped

The random ordering is important. If you have two channels and always poll the first one first, the second channel can starve under load. The randomization makes this fair.

If you want deterministic ordering (poll top-to-bottom), use `biased;`:

```rust
tokio::select! {
    biased;

    // This gets checked first, always
    _ = shutdown_signal() => { return; }

    // This only checked if shutdown isn't ready
    msg = rx.recv() => { handle(msg); }
}
```

### Cancellation safety

This is the sharp edge. When a branch loses the race, its future is **dropped**. If that future was in the middle of doing something, that work is lost.

**Cancel-safe** (no work lost on drop):
- `mpsc::Receiver::recv()`
- `TcpListener::accept()`
- `tokio::io::AsyncReadExt::read()`
- `oneshot::Receiver::recv()`

**NOT cancel-safe** (partial progress lost):
- `tokio::io::AsyncReadExt::read_exact()` - may have read some bytes
- `tokio::io::AsyncWriteExt::write_all()` - may have written some bytes
- `tokio::sync::Mutex::lock()` - loses queue position

Example of the problem:

```rust
let mut buf = [0u8; 1024];

loop {
    tokio::select! {
        // BUG: if read_exact read 500 bytes then the other branch
        // completes, those 500 bytes are gone forever
        result = stream.read_exact(&mut buf) => {
            process(&buf);
        }
        _ = some_other_future() => {
            // read_exact's partial progress is dropped
        }
    }
}
```

The fix: use cancel-safe operations in `select!`, or restructure to avoid the problem:

```rust
loop {
    tokio::select! {
        // read() is cancel-safe - returns whatever bytes are available
        result = stream.read(&mut buf) => {
            let n = result?;
            process(&buf[..n]);
        }
        _ = some_other_future() => { }
    }
}
```

### Pin requirements in loops

If you use `select!` in a loop and want to reuse a future across iterations, you need to pin it:

```rust
let sleep = tokio::time::sleep(Duration::from_secs(60));
tokio::pin!(sleep);

loop {
    tokio::select! {
        _ = &mut sleep => {
            println!("60 seconds total elapsed");
            break;
        }
        msg = rx.recv() => {
            // process msg, sleep future persists to next iteration
        }
    }
}
```

Without `tokio::pin!`, the borrow checker rejects this because `select!` needs `&mut` references to the futures, and they must be `Unpin`.

## Common gotchas

### 1. `std::thread::sleep` in async code

This is the number one mistake:

```rust
// WRONG - blocks the entire worker thread
async fn bad() {
    std::thread::sleep(Duration::from_secs(1));
}

// RIGHT - yields to the scheduler
async fn good() {
    tokio::time::sleep(Duration::from_secs(1)).await;
}
```

`std::thread::sleep` puts the OS thread to sleep. In a multi-thread runtime with 8 workers, doing this in 8 concurrent tasks deadlocks the entire runtime. No other tasks can make progress.

If you mentioned in a previous post about [debugging async code](/blog/debugging-rust-beyond-println), breakpoints have the same problem - they freeze the executor thread.

The same applies to any blocking operation: synchronous file I/O, CPU-heavy computation, blocking mutex locks. Use `spawn_blocking` for these.

### 2. Holding a `MutexGuard` across `.await`

```rust
// WRONG - std::sync::Mutex guard held across await
async fn bad(data: Arc<std::sync::Mutex<Vec<String>>>) {
    let mut guard = data.lock().unwrap();
    // This .await might suspend the task while holding the lock
    do_something_async().await;
    guard.push("done".to_string());
}
```

With `std::sync::Mutex`, if the task suspends at the `.await` and another task on the same thread tries to lock the same mutex, you get a deadlock. The worker thread is blocked waiting for a lock that can only be released by a task that needs that same worker thread to run.

Two solutions:

```rust
// Option 1: scope the lock tightly
async fn good_v1(data: Arc<std::sync::Mutex<Vec<String>>>) {
    // Lock, do sync work, drop guard before any await
    {
        let mut guard = data.lock().unwrap();
        guard.push("start".to_string());
    } // guard dropped here

    do_something_async().await;

    {
        let mut guard = data.lock().unwrap();
        guard.push("done".to_string());
    }
}

// Option 2: use tokio::sync::Mutex (async-aware)
async fn good_v2(data: Arc<tokio::sync::Mutex<Vec<String>>>) {
    let mut guard = data.lock().await; // yields instead of blocking
    do_something_async().await;
    guard.push("done".to_string());
}
```

Note: `tokio::sync::Mutex` is slower than `std::sync::Mutex` for uncontended cases. If you can scope your locks to avoid holding them across `.await` points, `std::sync::Mutex` is fine and faster.

### 3. Forgetting that `JoinHandle` detaches on drop

```rust
async fn oops() {
    // Task is spawned but handle is dropped immediately
    // The task keeps running in the background!
    tokio::spawn(async {
        loop {
            do_work().await;
        }
    });
    // If this function returns, the spawned task is still alive
}
```

This is by design - `tokio::spawn` detaches on drop, like `std::thread::spawn`. If you need structured concurrency (parent waits for children), use `JoinSet`.

### 4. Accidentally creating a runtime inside a runtime

```rust
async fn handler() {
    // PANIC: Cannot start a runtime from within a runtime
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async { });
}
```

If you need to call async code from a sync context that's already inside tokio, use `tokio::task::spawn_blocking` + `tokio::runtime::Handle::current()`:

```rust
async fn handler() {
    let result = tokio::task::spawn_blocking(|| {
        let handle = tokio::runtime::Handle::current();
        handle.block_on(async {
            // async code here
            42
        })
    }).await.unwrap();
}
```

### 5. Not understanding `Send` bounds

`tokio::spawn` requires `Send + 'static`. This means you can't hold a non-Send type (like `Rc` or a raw pointer) across an `.await`:

```rust
// COMPILE ERROR: Rc is not Send
async fn fails() {
    let data = Rc::new(42);
    some_async_fn().await; // Rc held across await = future not Send
    println!("{data}");
}
```

Fix: use `Arc` instead of `Rc`, or if you genuinely don't need cross-thread sharing, use `current_thread` runtime with `tokio::task::spawn_local`.

## When tokio, when smol?

The async runtime landscape in Rust has consolidated significantly.

**async-std is deprecated.** As of March 2025, the async-std team officially recommends migrating away. The last release was 1.13.2. Don't start new projects with it.

That leaves two real options:

### Tokio (~589 million downloads, ~20,700 dependent crates)

Use tokio when:
- You need the ecosystem. Hyper, tonic, axum, reqwest, sqlx, sea-orm - they all depend on tokio
- You need a production-grade, battle-tested runtime
- You need the full toolkit: timers, fs, signals, process management, sync primitives

Tokio is a framework, not just an executor. It provides `tokio::fs`, `tokio::net`, `tokio::sync`, `tokio::io` - a complete async standard library replacement.

### Smol (~14.5 million downloads)

Use smol when:
- You want minimal dependencies and a small binary
- You're building a library that shouldn't force tokio on users
- You need a simple executor in about ~1,000 lines of code
- Educational purposes - reading smol's source is feasible in an afternoon

Smol takes a different philosophy: instead of reimplementing everything, it provides adapters. `smol::Async<TcpStream>` wraps a standard library type and makes it async. It composes smaller crates (`async-executor`, `async-io`, `polling`) rather than shipping a monolith.

The practical answer for most projects: **use tokio**. The ecosystem lock-in is real - if you use any popular async crate, you're already pulling in tokio. Fighting it gains you nothing.

If you're writing a small utility or a library that should stay runtime-agnostic, consider smol or just depending on the `futures` crate traits and letting users bring their own executor.

## Runtime internals cheat sheet

Quick reference for the numbers that matter:

| Parameter | Default | Config |
|-----------|---------|--------|
| Worker threads | CPU cores | `worker_threads()` or `TOKIO_WORKER_THREADS` |
| Local queue size | 256 tasks | Not configurable |
| Global queue check | ~10ms interval | `global_queue_interval()` |
| I/O poll interval | Every 61 ticks | `event_interval()` |
| Coop budget | 128 ops/tick | Not configurable |
| Blocking pool max | 512 threads | `max_blocking_threads()` |
| Blocking thread keep-alive | 10 seconds | `thread_keep_alive()` |
| Worker stack size | 2 MiB | `thread_stack_size()` |

Current stable version: **tokio 1.50.0** (March 2026). LTS track 1.47.x supported until September 2026.

All source links referenced in this post point to the [`tokio-rs/tokio`](https://github.com/tokio-rs/tokio) repository on GitHub. If you want to go deeper, start with the [scheduler blog post](https://tokio.rs/blog/2019-10-scheduler) and the [cooperative preemption post](https://tokio.rs/blog/2020-04-preemption) - they explain the design decisions behind the code.
