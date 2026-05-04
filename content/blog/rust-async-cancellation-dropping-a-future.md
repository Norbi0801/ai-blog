+++
title = "Rust async cancellation - what happens when you drop a future"
date = 2026-01-14
description = "Dropping a future is cancellation. What that actually means for cleanup, cancellation safety, tokio::select!, and the bugs that hide in async code."

[taxonomies]
tags = ["rust", "async", "tokio", "concurrency"]
+++

Cancellation in Rust async code is one of those topics that looks simple at first. You drop a future, it stops running. Done. But once you have real production code with database transactions, temp files, and retry loops layered on top, you start seeing weird half-states: half-written files, transactions that never committed or rolled back, metrics that count requests that never finished, locks that look held but aren't. Most of these come from one place: the assumption that "stopping" an async task does something meaningful, when in reality it just means nobody calls `.poll()` on it anymore.

This post walks through what actually happens when you drop a future, why `tokio::select!` requires you to think about "cancellation safety", how `JoinHandle::abort` is more aggressive than dropping, why async `Drop` still doesn't exist, and what patterns work when you really need cleanup.

<!-- more -->

## A future is just a state machine

To understand cancellation, you have to remember what an `async fn` actually compiles to. The compiler turns this:

```rust
async fn fetch_and_save(url: &str, path: &str) -> std::io::Result<()> {
    let body = reqwest::get(url).await.unwrap().text().await.unwrap();
    tokio::fs::write(path, body).await?;
    Ok(())
}
```

into something roughly equivalent to a hand-rolled state machine:

```rust
enum FetchAndSave<'a> {
    Start { url: &'a str, path: &'a str },
    AwaitingGet  { /* path, future from reqwest::get */ },
    AwaitingText { /* path, future from .text()      */ },
    AwaitingWrite{ /* future from fs::write          */ },
    Done,
}
```

Each `.await` is a possible suspension point. When the runtime calls `poll()` on the outer future, the state machine advances as far as it can, then either returns `Poll::Ready(value)` or `Poll::Pending`. Pending means "I'm waiting on something, wake me up later." If you want a deeper look at how the compiler builds these, the [async book chapter on under-the-hood](https://rust-lang.github.io/async-book/02_execution/01_chapter.html) goes step by step.

Cancellation, in this model, is trivial: the runtime simply stops calling `.poll()`. The future is dropped wherever it's stored. That's it. There's no signal sent to the task. There's no callback. The state machine just gets destroyed mid-execution.

This has two important consequences:

1. **Cancellation only happens at `.await` points.** A function executing synchronous code between awaits cannot be cancelled. The runtime can only swap out a task when it returns `Poll::Pending`.
2. **`Drop` is the only cleanup mechanism.** When the state machine struct is destroyed, every field's `Drop` impl runs. If you held a `tokio::sync::MutexGuard` across an await, it gets unlocked. If you had an open file handle, it gets closed. But all of this happens *synchronously*, in `Drop`.

That second point is where most of the pain comes from.

## Why async Drop still doesn't exist

`Drop::drop(&mut self)` is a synchronous function. It cannot `.await`. This means anything that requires async cleanup - sending a final message, flushing a network buffer, committing or rolling back a transaction - cannot happen automatically when a future is cancelled.

There's been ongoing work on `AsyncDrop` for years. The [tracking issue](https://github.com/rust-lang/rust/issues/126482) was opened in 2024, and as of mid-2026 it's still experimental behind a nightly feature flag. The hard part isn't writing the trait, it's making it sound and composable: what runtime do you call into, what happens if the cleanup itself panics, how does the borrow checker reason about a value that needs an async destructor.

Until that lands, the practical answer is: if you have async cleanup, you cannot rely on `Drop`. You have to either:

- Make cleanup synchronous (close a file handle, drop a guard, abort a syscall).
- Spawn the cleanup as a detached task in `Drop`, accepting that it runs after the cancelled future is already gone.
- Avoid cancellation in the section of code that needs cleanup (see `CancellationToken` below).

The "spawn in Drop" pattern shows up a lot in libraries:

```rust
impl Drop for MyConnection {
    fn drop(&mut self) {
        if let Some(socket) = self.socket.take() {
            tokio::spawn(async move {
                let _ = socket.shutdown().await;
            });
        }
    }
}
```

This works as long as the runtime is still alive. If the runtime itself is shutting down, the spawned task may never run. `sqlx` does something similar for connections - the [pool drop logic](https://github.com/launchbadge/sqlx/blob/main/sqlx-core/src/pool/inner.rs) spawns a background task to drain connections, but warns in docs that you should call `pool.close().await` for guaranteed cleanup.

## tokio::select! and the meaning of "cancellation safety"

The most common place developers run into cancellation bugs is `tokio::select!`. The macro polls multiple futures concurrently and resolves with the first one that completes. The other branches are *dropped*.

```rust
tokio::select! {
    _ = tokio::time::sleep(Duration::from_secs(5)) => {
        println!("timeout");
    }
    msg = stream.next() => {
        println!("got message: {:?}", msg);
    }
}
```

If `sleep` wins, the `stream.next()` future is dropped. Whatever state it accumulated mid-poll is destroyed. This is the source of the term "cancellation safety": a future is cancel-safe if dropping it mid-execution does not lose data or leave shared state in a corrupted condition.

`stream.next()` happens to be cancel-safe for most stream implementations - the next call to `.next()` will resume from the same position. But consider this:

```rust
async fn read_chunk(socket: &mut TcpStream, buf: &mut [u8]) -> io::Result<usize> {
    socket.read(buf).await
}

tokio::select! {
    _ = tokio::time::sleep(Duration::from_secs(5)) => { /* timeout */ }
    n = read_chunk(&mut socket, &mut buf) => { /* use n bytes */ }
}
```

`AsyncReadExt::read` is cancel-safe by tokio's contract: if the future is dropped before completion, no bytes were consumed from the socket. The kernel buffer is untouched, so the next call to `read` will see the same bytes. The [tokio docs on cancellation safety](https://docs.rs/tokio/latest/tokio/macro.select.html#cancellation-safety) list which standard library and tokio APIs are safe.

But what about this:

```rust
async fn read_two(socket: &mut TcpStream) -> io::Result<(u8, u8)> {
    let a = socket.read_u8().await?;
    let b = socket.read_u8().await?;
    Ok((a, b))
}
```

Not cancel-safe. If the future is dropped between the two reads, byte `a` was consumed from the socket but byte `b` wasn't. The next call to `read_two` will return `(b, c)` instead of `(a, b)`. You've corrupted the protocol.

The fix is usually to keep the partial state in a struct and resume from it, or to wrap it in a `tokio::spawn` so the task itself is not subject to the select's cancellation:

```rust
let handle = tokio::spawn(read_two(socket));
tokio::select! {
    _ = tokio::time::sleep(Duration::from_secs(5)) => {
        handle.abort();
    }
    res = &mut handle => {
        // got a result before timeout
    }
}
```

This shifts the question from "is `read_two` cancel-safe" to "what does `JoinHandle::abort` do".

## JoinHandle::abort vs dropping

`tokio::spawn(future)` returns a `JoinHandle<T>`. The handle is a way to wait for the result and, separately, a way to cancel the task.

Dropping the `JoinHandle` does **not** cancel the task. The task keeps running. It just becomes detached - no one is going to await its result. This is the opposite of `std::thread::JoinHandle`, which doesn't cancel either, but at least you can't "lose" the OS thread.

To actually cancel a spawned task, you call `.abort()`:

```rust
let handle = tokio::spawn(async {
    loop {
        do_work().await;
    }
});
handle.abort();
let result = handle.await;
assert!(result.unwrap_err().is_cancelled());
```

Internally, `abort` flips a flag on the task header. The next time the runtime polls the task, it sees the flag and drops the future instead of polling it. So abort is still cooperative in the sense that the task only "stops" at the next await point - if the task is busy in a sync hot loop, abort doesn't preempt it. The [tokio task source](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/task/mod.rs) shows the abort handle is just an `AtomicWaker`-style mechanism setting a CANCELLED bit in the task state.

A subtle point: if the task is *currently* being polled on another thread when `abort` is called, abort returns immediately, but the actual cancellation happens after the current poll finishes. This is why `handle.await` is the only way to know the task has actually stopped.

## CancellationToken: cooperative cancellation done right

For anything more complex than "cancel one task", the standard pattern is `tokio_util::sync::CancellationToken`. It's a tree of tokens that can be cancelled together, with futures you can `.await` to wait for cancellation.

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();
let child = token.child_token();

let task = tokio::spawn(async move {
    loop {
        tokio::select! {
            _ = child.cancelled() => {
                println!("shutting down cleanly");
                cleanup().await;
                break;
            }
            _ = do_work() => { /* keep going */ }
        }
    }
});

// Later, from anywhere that holds the parent token:
token.cancel();
task.await.unwrap();
```

The key difference from `JoinHandle::abort` is that the task gets a chance to run async cleanup before exiting. The `child.cancelled()` future just becomes ready - the task is the one that decides what to do next. This is the only way to get reliable async cleanup today.

Child tokens cancel when the parent does, but you can also cancel a child without affecting the parent. This makes it natural to model graceful shutdown: a parent token at the application level, child tokens per subsystem, grandchild tokens per request.

## Real bugs from cancellation

Three patterns I keep seeing in production code:

**Database transactions left dangling.** This one usually shows up in select-with-timeout code:

```rust
let mut tx = pool.begin().await?;
sqlx::query!("INSERT ...").execute(&mut *tx).await?;
sqlx::query!("UPDATE ...").execute(&mut *tx).await?;
tx.commit().await?;
```

If the future running this is cancelled between the INSERT and the COMMIT, what happens? The `Transaction` value is dropped. Its `Drop` impl spawns a rollback task, which the runtime probably runs eventually. But there's a window where the database thinks the transaction is still open. If your code is wrapped in `tokio::time::timeout`, you might be racing the rollback against the next request that wants the same row. The [sqlx Transaction docs](https://docs.rs/sqlx/latest/sqlx/struct.Transaction.html) explicitly warn about this and recommend calling `tx.rollback().await` explicitly when you want guaranteed semantics.

**Temp files leaked.** `tempfile::NamedTempFile` deletes itself in `Drop` via `unlink(2)`. That's synchronous, so it works. But if you wrote bytes to it and the task got cancelled before you renamed it to its final destination, the partial data is gone (good), but if you did `tempfile.persist("final.txt")` and the cancellation happened after the rename but before logging, you'll have a `final.txt` that no one knows about. Cancellation often turns idempotent operations into ones that need careful resumption logic.

**Half-released locks.** `tokio::sync::Mutex::lock().await` returns a `MutexGuard` that releases on drop. Normally fine. But consider:

```rust
let guard = mutex.lock().await;
// (1) some sync work
do_async_thing().await;  // (2) await point
// (3) more work using guard
```

Cancellation between (1) and (2) drops the guard. Cancellation at (2) drops the guard mid-await, releasing the mutex. Whoever was waiting now gets the lock and sees state that may be in an intermediate condition the original task assumed it would finish updating. The fix is usually to do the entire critical section synchronously, or to make the data structure itself robust to partial updates.

## Structured concurrency

The general lesson is that unstructured `tokio::spawn` + dropping + abort is fragile. The Rust async ecosystem has been moving toward "structured concurrency" patterns where child tasks are tied to a parent scope and cleanup is enforced by the type system.

Three building blocks:

- **`tokio_util::task::TaskTracker`** - a counter that knows how many tasks are alive. You spawn through the tracker; it gives you a future that resolves when all tasks finish. Combined with a `CancellationToken`, this is the standard graceful-shutdown pattern.
- **`tokio::task::JoinSet`** - a collection of `JoinHandle`s with combinators. Dropping the set aborts all tasks. This is the closest thing to an OS-level "process group" semantically.
- **scoped tasks** (still nightly via various crates like [async-scoped](https://docs.rs/async-scoped/)) - allow spawned tasks to borrow from the parent. The scope cannot return until all children have joined, which makes lifetimes work out. Tokio's own `task::scope` is still not stable as of mid-2026, but `JoinSet` covers most use cases.

For most applications, the recipe is: one root `CancellationToken`, one root `TaskTracker`, every `tokio::spawn` goes through the tracker and gets a child token. Shutdown is `token.cancel(); tracker.wait().await;`. If a task takes too long to react to cancellation, you abort it after a grace period:

```rust
token.cancel();
tokio::select! {
    _ = tracker.wait() => println!("clean shutdown"),
    _ = tokio::time::sleep(Duration::from_secs(30)) => {
        println!("forcing shutdown");
        // tracker doesn't have abort; you'd track JoinHandles separately
    }
}
```

## Takeaways

Cancellation in Rust async is fast and lightweight - drop a future and it's gone. But the price is that the language gives you almost no help with cleanup. `Drop` is synchronous, so anything that needs async cleanup has to either be wrapped in a cooperative pattern (`CancellationToken`), restructured to be synchronous, or accept that cleanup happens out of band on a spawned task.

Three rules that have served me well:

1. Assume every `.await` is a possible cancellation point. If state would be inconsistent at that point, restructure the code.
2. Don't cross an `.await` boundary holding non-trivial resources unless you've thought about what cancellation does to them.
3. Use `JoinHandle::abort` and `CancellationToken` for explicit lifecycles. Don't rely on dropping futures as a cleanup mechanism unless you've verified that the resource's `Drop` impl is actually sufficient.

Async cancellation isn't broken in Rust. It's just unforgiving in the way most low-level Rust is unforgiving: the language hands you the primitives and expects you to know what you're doing with them. The good news is that once you internalize "drop = cancel = no async cleanup", most of the surprises stop being surprises.
