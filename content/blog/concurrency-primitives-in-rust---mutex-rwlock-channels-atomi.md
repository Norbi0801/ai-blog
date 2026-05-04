+++
title = "Concurrency primitives in Rust - Mutex, RwLock, channels, atomics"
date = 2025-07-25
description = "A practical guide to Rust's concurrency toolkit: when to reach for Mutex vs RwLock vs channels vs atomics, what happens under the hood, and how to avoid deadlocks."

[taxonomies]
tags = ["rust", "concurrency", "performance", "systems-programming"]
+++

Rust gives you memory safety without a garbage collector. It does not give you freedom from concurrency bugs. You can still deadlock. You can still starve a writer. You can still burn CPU in a spin loop because you picked the wrong primitive. The compiler prevents data races at compile time through the ownership system, `Send`, and `Sync` - but choosing the right synchronization primitive for your workload is still on you.

This post is a practical walkthrough of the four main concurrency primitives in std, plus their tokio counterparts. For each one: what it does internally, when to use it, when not to, and a real example. If you're not familiar with tokio's runtime model - the work-stealing scheduler, spawn vs spawn_blocking, and the cooperative preemption budget - I covered all of that in [Understanding Tokio](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/).

<!-- more -->

## Arc - the foundation for shared ownership

Before we get into synchronization, we need `Arc`. Every primitive in this post assumes you're sharing data across threads, and `Arc<T>` (atomic reference counting) is how you do that in Rust.

```rust
use std::sync::Arc;

let data = Arc::new(vec![1, 2, 3]);
let data_clone = Arc::clone(&data);

std::thread::spawn(move || {
    println!("from thread: {:?}", data_clone);
});
```

`Arc` uses an [`AtomicUsize`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html) internally for its reference count. Every clone increments the count with `fetch_add(1, Relaxed)`, every drop decrements with `fetch_sub(1, Release)`. The last drop uses an `Acquire` fence before deallocating to ensure all writes from other threads are visible. If you used `Arc<str>` for string interning in the [flyweight pattern post](/blog/the-flyweight-pattern-sharing-data-efficiently-in-rust), that's the same mechanism at work.

`Arc` gives you shared read access. It does not let you mutate the inner data. For that, you combine it with one of the primitives below. The pattern is almost always `Arc<Mutex<T>>`, `Arc<RwLock<T>>`, or `Arc<AtomicU64>`.

One thing worth noting: `Arc::clone()` is not free. The atomic increment costs roughly 10-20ns depending on contention and cache line bouncing between cores. For hot paths where you're cloning millions of times per second, this matters. Usually it doesn't.

## Mutex - one thread at a time

[`std::sync::Mutex<T>`](https://doc.rust-lang.org/std/sync/struct.Mutex.html) gives exclusive access. One thread locks it, does its work, and unlocks it. Everyone else waits.

### What's inside

A std `Mutex` is three things packed together:

1. **A system mutex** - on Linux this is a futex (fast userspace mutex). The fast path is a single atomic compare-and-swap in userspace. Only if the lock is contended does it fall through to the `futex` syscall and actually put the thread to sleep.
2. **An `UnsafeCell<T>`** - this is what makes interior mutability possible. The `Mutex` hands out `&mut T` through `MutexGuard` even though the `Mutex` itself is behind a shared reference.
3. **A poison flag** - an `AtomicBool` that gets set if a thread panics while holding the lock. Subsequent `lock()` calls return `Err(PoisonError)` instead of `Ok(MutexGuard)`.

You can see the actual implementation in [std's source](https://doc.rust-lang.org/src/std/sync/poison/mutex.rs.html).

### Basic usage

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0u64));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    handles.push(thread::spawn(move || {
        for _ in 0..1000 {
            let mut guard = counter.lock().unwrap();
            *guard += 1;
            // guard dropped here, lock released
        }
    }));
}

for handle in handles {
    handle.join().unwrap();
}

assert_eq!(*counter.lock().unwrap(), 10_000);
```

The `.unwrap()` on `lock()` is because of poisoning. In production, you might want to handle the poisoned case:

```rust
let guard = counter.lock().unwrap_or_else(|poisoned| {
    log::warn!("mutex was poisoned, recovering");
    poisoned.into_inner()
});
```

This recovers the data inside a poisoned mutex. Whether that's safe depends on your invariants.

### The scope of the lock matters

This is the number one Mutex mistake:

```rust
// BAD: holds lock during the expensive operation
let mut guard = state.lock().unwrap();
let result = expensive_computation(&guard.data);
guard.result = result;
```

```rust
// GOOD: clone what you need, drop the lock, compute, re-lock
let data = {
    let guard = state.lock().unwrap();
    guard.data.clone()
};
let result = expensive_computation(&data);
{
    let mut guard = state.lock().unwrap();
    guard.result = result;
}
```

Every nanosecond you hold the lock is a nanosecond other threads are blocked. Keep critical sections as short as possible.

### parking_lot as an alternative

The [`parking_lot`](https://crates.io/crates/parking_lot) crate provides a `Mutex` that's smaller (1 byte vs ~40 bytes for std on Linux), faster under contention (up to 5x in benchmarks), and doesn't use poisoning. It also supports eventual fairness - every ~0.5ms it forces a fair unlock to prevent starvation.

```rust
use parking_lot::Mutex;

let data = Mutex::new(vec![1, 2, 3]);
let guard = data.lock(); // no Result, no poisoning
```

The trade-off: no poisoning means a panicking thread silently unlocks the mutex, possibly leaving the data in an inconsistent state. For most applications, this is fine - if you're panicking, you're probably about to crash anyway. But if you're building a server that catches panics per-request, poisoning is a useful safety net.

## RwLock - many readers, one writer

[`std::sync::RwLock<T>`](https://doc.rust-lang.org/std/sync/struct.RwLock.html) is for workloads where reads vastly outnumber writes. Multiple threads can hold a read lock simultaneously, but a write lock requires exclusive access.

```rust
use std::sync::{Arc, RwLock};
use std::thread;

let config = Arc::new(RwLock::new(AppConfig::default()));

// Readers - these can all run concurrently
for _ in 0..8 {
    let config = Arc::clone(&config);
    thread::spawn(move || {
        let cfg = config.read().unwrap();
        println!("max_connections: {}", cfg.max_connections);
    });
}

// Writer - blocks until all readers release
{
    let mut cfg = config.write().unwrap();
    cfg.max_connections = 200;
}
```

### When RwLock beats Mutex

RwLock wins when:
- Read-to-write ratio is at least 10:1
- Read critical sections are non-trivial (not just reading a single integer)
- You have enough concurrent readers to benefit from parallelism

RwLock loses when:
- Writes are frequent - each write must wait for all readers to drain
- Critical sections are tiny - the overhead of tracking reader count outweighs the benefit
- You only have 2-3 threads - the contention isn't high enough to justify the complexity

A common mistake is using `RwLock` for a configuration object that gets read on every request and written once at startup. That's a valid use case - but if the config is small, consider just using `Arc<Config>` and swapping the entire `Arc` atomically with [`arc_swap`](https://crates.io/crates/arc_swap). No locking at all.

### Writer starvation

The std `RwLock` documentation says it plainly: the priority policy depends on the OS, and no particular policy is guaranteed. On some platforms, a continuous stream of readers can starve writers indefinitely - the write lock never gets acquired because there's always at least one active reader.

[`parking_lot::RwLock`](https://docs.rs/parking_lot/latest/parking_lot/type.RwLock.html) solves this with task-fair locking. Any critical section longer than 1ms always uses a fair unlock, and on average fairness is enforced every 0.5ms. [`tokio::sync::RwLock`](https://docs.rs/tokio/latest/tokio/sync/struct.RwLock.html) uses a write-preferring FIFO queue - once a writer is waiting, no new readers are admitted.

If you're using std's `RwLock` and writers matter, test under load. Or just use `parking_lot`.

## Channels - message passing instead of shared state

Channels flip the model. Instead of multiple threads accessing shared data protected by a lock, one thread owns the data and other threads send messages to it. This is the actor model in miniature.

### std::sync::mpsc

The standard library provides [`mpsc`](https://doc.rust-lang.org/std/sync/mpsc/) - multiple producer, single consumer.

```rust
use std::sync::mpsc;
use std::thread;

#[derive(Debug)]
enum Command {
    Increment(u64),
    GetTotal(mpsc::Sender<u64>),
    Shutdown,
}

let (tx, rx) = mpsc::channel(); // unbounded

// Spawn a worker that owns the state
let worker = thread::spawn(move || {
    let mut total: u64 = 0;
    loop {
        match rx.recv().unwrap() {
            Command::Increment(n) => total += n,
            Command::GetTotal(reply) => { reply.send(total).unwrap(); }
            Command::Shutdown => break,
        }
    }
});

// Producers send commands
let tx2 = tx.clone();
thread::spawn(move || {
    for i in 0..1000 {
        tx2.send(Command::Increment(i)).unwrap();
    }
});

tx.send(Command::Increment(42)).unwrap();

// Query the total
let (reply_tx, reply_rx) = mpsc::channel();
tx.send(Command::GetTotal(reply_tx)).unwrap();
let total = reply_rx.recv().unwrap();

tx.send(Command::Shutdown).unwrap();
worker.join().unwrap();
```

Notice the pattern: the `total` variable lives inside the worker thread. No `Mutex`, no `Arc`. The channel is the synchronization mechanism. If you've seen bounded channels used for backpressure in the [webhook receiver post](/blog/building-a-webhook-receiver-in-rust), this is the same idea scaled up.

`mpsc::channel()` creates an unbounded channel. It will happily allocate memory until your process OOMs. For production code, use bounded channels or tokio's bounded mpsc.

### crossbeam-channel

[`crossbeam-channel`](https://crates.io/crates/crossbeam-channel) is what you want when std's mpsc isn't enough:

- **MPMC** - multiple producers AND multiple consumers. You can fan out work to a pool of workers.
- **`select!` macro** - wait on multiple channels at once, like Go's `select`.
- **Bounded channels** - built in, with backpressure.
- **Performance** - lock-free implementation, 2-10x faster than std's mpsc under contention.

```rust
use crossbeam_channel::{bounded, select, Receiver, Sender};
use std::thread;
use std::time::Duration;

let (task_tx, task_rx): (Sender<String>, Receiver<String>) = bounded(100);
let (done_tx, done_rx) = bounded(0); // rendezvous channel

// Spin up a worker pool
for id in 0..4 {
    let task_rx = task_rx.clone();
    let done_tx = done_tx.clone();
    thread::spawn(move || {
        while let Ok(task) = task_rx.recv() {
            println!("worker {} processing: {}", id, task);
            // simulate work
            thread::sleep(Duration::from_millis(10));
        }
        drop(done_tx);
    });
}

// Send tasks
for i in 0..20 {
    task_tx.send(format!("task-{}", i)).unwrap();
}
drop(task_tx); // close channel, workers will drain and exit

// Wait for all workers to finish
drop(done_tx);
done_rx.recv().ok(); // blocks until all senders dropped
```

The key difference from std: `task_rx.clone()` works. Multiple workers pull from the same channel. Work gets distributed automatically.

### tokio::sync::mpsc

In async code, use [`tokio::sync::mpsc`](https://docs.rs/tokio/latest/tokio/sync/mpsc/). It's always bounded (you must specify a capacity), and `send()` is async - it yields the task instead of blocking the thread.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(100);

    tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            println!("got: {}", msg);
        }
    });

    tx.send("hello".into()).await.unwrap();
    tx.send("world".into()).await.unwrap();
}
```

Never use `std::sync::mpsc` inside async code. Its `recv()` blocks the thread, which means it blocks the entire tokio worker thread and starves other tasks on that worker. Always use `tokio::sync::mpsc` (or `crossbeam` with `spawn_blocking`).

### When to use channels vs shared state

Channels are better when:
- One thread/task "owns" the data and processes commands sequentially
- You want to decouple producers from consumers
- The work involves fan-out (one producer, many workers)
- You want natural backpressure via bounded channels

Shared state (Mutex/RwLock) is better when:
- Multiple threads need to read/write the same data with low latency
- The critical section is tiny (increment a counter, flip a flag)
- You need to query current state without round-trip message passing

Don't dogmatically pick one approach. A real system usually uses both - channels for coarse-grained coordination between subsystems, mutexes for fine-grained state within a subsystem.

## Atomics - lock-free operations

Atomic types are the lowest-level concurrency primitive. They use CPU instructions (like `LOCK CMPXCHG` on x86) that guarantee certain memory operations are indivisible. No OS calls, no sleeping, no locking. Just one instruction that the CPU guarantees completes atomically with respect to all other cores.

Rust provides atomic types in [`std::sync::atomic`](https://doc.rust-lang.org/std/sync/atomic/): `AtomicBool`, `AtomicI8` through `AtomicI64`, `AtomicU8` through `AtomicU64`, `AtomicUsize`, and `AtomicPtr`.

### Simple counter

The most common use case: a counter that multiple threads increment.

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

let counter = Arc::new(AtomicU64::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    handles.push(thread::spawn(move || {
        for _ in 0..1000 {
            counter.fetch_add(1, Ordering::Relaxed);
        }
    }));
}

for h in handles {
    h.join().unwrap();
}

assert_eq!(counter.load(Ordering::Relaxed), 10_000);
```

Compare this to the Mutex version earlier. No `lock()`, no `unwrap()`, no `MutexGuard`. Just `fetch_add`. For a simple counter, atomics are ~10-50x faster than a Mutex depending on contention.

### Memory ordering - the hard part

Every atomic operation takes an [`Ordering`](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html) parameter. This tells the CPU (and the compiler) how much it needs to synchronize with other threads. From weakest to strongest:

**`Relaxed`** - the operation is atomic, but there's no synchronization of other memory. Other threads might see surrounding writes in a different order. Use this for counters, statistics, anything where you just need the final value to be correct.

**`Acquire`** (loads) / **`Release`** (stores) - these form a pair. When thread B does an `Acquire` load and sees a value that thread A wrote with a `Release` store, thread B is guaranteed to see all writes that thread A did before its `Release` store. This is how you implement a lock: the lock acquisition is `Acquire`, the release is `Release`.

```rust
use std::sync::atomic::{AtomicBool, Ordering};

static READY: AtomicBool = AtomicBool::new(false);
static mut DATA: u64 = 0; // safe only because of the ordering guarantees

fn producer() {
    unsafe { DATA = 42; }              // write data first
    READY.store(true, Ordering::Release); // then signal
}

fn consumer() {
    while !READY.load(Ordering::Acquire) {} // spin until signaled
    // Acquire guarantees we see DATA = 42
    assert_eq!(unsafe { DATA }, 42);
}
```

Don't actually write code like this with `static mut` - it's here to show the ordering guarantee. In practice, use proper synchronization types that build on these orderings.

**`AcqRel`** - both acquire and release in one operation. Used for read-modify-write operations like `compare_exchange` that both read and write. The read part gets `Acquire` semantics, the write part gets `Release` semantics.

**`SeqCst`** - sequentially consistent. Everything above, plus a global total order that all threads agree on. This is the sledgehammer. It's the easiest to reason about because it behaves like you'd naively expect - all operations happen in one global sequence. But it comes with a cost. On x86 it typically adds an `MFENCE` instruction or a locked operation, which can be 5-10x slower than `Relaxed` in benchmarks.

Mara Bos's [Rust Atomics and Locks](https://marabos.nl/atomics/) covers this in far more depth than I can here. If you're writing lock-free data structures, read that book.

### When to reach for atomics

The rule is simple: **use atomics for single values, use Mutex for compound state**.

Good uses for atomics:
- Counters and statistics (`AtomicU64` with `fetch_add`)
- Flags and state machines (`AtomicBool`, `AtomicU8`)
- Sequence numbers (`AtomicU64` with `fetch_add`)
- Stop signals (`AtomicBool` checked in a loop)
- One-time initialization (`AtomicBool` or `std::sync::Once`)

Bad uses for atomics:
- Updating two related fields (you need both to be consistent - use a Mutex)
- Anything requiring a critical section longer than one operation
- Complex state transitions where you need to read multiple values, compute, and write back

For the CAS (compare-and-swap) pattern specifically:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

let value = AtomicU64::new(100);

// Atomically subtract 30, but only if value >= 30
let mut current = value.load(Ordering::Relaxed);
loop {
    if current < 30 {
        println!("insufficient balance");
        break;
    }
    match value.compare_exchange_weak(
        current,
        current - 30,
        Ordering::AcqRel,
        Ordering::Relaxed,
    ) {
        Ok(_) => {
            println!("subtracted 30, new value: {}", current - 30);
            break;
        }
        Err(actual) => current = actual, // someone else changed it, retry
    }
}
```

`compare_exchange_weak` can spuriously fail (return `Err` even when the value matches) but is faster on architectures like ARM that use LL/SC (load-linked/store-conditional) instead of native CAS. Always prefer `_weak` when you're already in a loop.

## tokio variants - sync vs async

tokio provides async versions of several primitives in [`tokio::sync`](https://docs.rs/tokio/latest/tokio/sync/). The critical question is: when do you need the async version?

### tokio::sync::Mutex vs std::sync::Mutex

The [tokio docs](https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html) say it directly: **prefer `std::sync::Mutex` in async code**. The async mutex is more expensive - roughly 25x slower in micro-benchmarks. Why? Because it has to manage a wait queue of tasks, integrate with the tokio waker system, and handle cancellation safety. The std mutex just does a futex call.

Use `tokio::sync::Mutex` only when you need to hold the lock across an `.await` point:

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

let db = Arc::new(Mutex::new(DatabaseConnection::new()));

tokio::spawn(async move {
    let mut conn = db.lock().await;  // async lock
    conn.execute("INSERT ...").await; // hold lock across .await
    conn.execute("UPDATE ...").await; // still holding
    // lock released when guard drops
});
```

With `std::sync::Mutex`, holding the guard across `.await` would block the tokio worker thread while the future is suspended. The async mutex avoids this by yielding the task. But if your critical section has no awaits in it, just use std:

```rust
use std::sync::Mutex;

// Fine in async code - no .await while lock is held
let value = {
    let guard = state.lock().unwrap();
    guard.current_count
};
```

### tokio::sync::RwLock

Same rule applies. Use tokio's `RwLock` when you need to hold a read or write lock across `.await` points. Otherwise, std or parking_lot.

One advantage of tokio's `RwLock`: it's write-preferring by default with a FIFO queue. No writer starvation.

### Other tokio::sync primitives

Worth knowing about:

- [`tokio::sync::Notify`](https://docs.rs/tokio/latest/tokio/sync/struct.Notify.html) - async version of a condition variable. One task waits, another notifies it.
- [`tokio::sync::Semaphore`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html) - limit concurrent access to N permits. Great for connection pooling or rate limiting.
- [`tokio::sync::watch`](https://docs.rs/tokio/latest/tokio/sync/watch/) - single-producer, multi-consumer. All receivers see the latest value. Perfect for config that updates at runtime.
- [`tokio::sync::broadcast`](https://docs.rs/tokio/latest/tokio/sync/broadcast/) - multi-producer, multi-consumer where every receiver gets every message. Event bus pattern.
- [`tokio::sync::oneshot`](https://docs.rs/tokio/latest/tokio/sync/oneshot/) - send exactly one value. The request-response pattern (send a query, get one answer back).

## Deadlock avoidance

Rust prevents data races at compile time. It does not prevent deadlocks. Here's how they happen and how to avoid them.

### Classic deadlock - lock ordering

```rust
// Thread 1:
let _a = mutex_a.lock().unwrap();
let _b = mutex_b.lock().unwrap(); // waits for thread 2

// Thread 2:
let _b = mutex_b.lock().unwrap();
let _a = mutex_a.lock().unwrap(); // waits for thread 1
// Both threads wait forever
```

The fix is trivial in principle: always acquire locks in the same order. If every thread locks A before B, no circular wait is possible.

```rust
// Both threads:
let _a = mutex_a.lock().unwrap();
let _b = mutex_b.lock().unwrap();
```

In practice, enforcing this across a codebase is hard. Some strategies:

1. **Number your locks** and always acquire in ascending order. This is what database systems do internally.
2. **Use `try_lock()`** with a fallback. If you can't get the second lock, release the first and retry:

```rust
loop {
    let a = mutex_a.lock().unwrap();
    match mutex_b.try_lock() {
        Ok(b) => {
            // got both locks
            do_work(&a, &b);
            break;
        }
        Err(_) => {
            drop(a); // release first lock
            std::thread::yield_now(); // let other threads progress
        }
    }
}
```

3. **Restructure to use fewer locks**. If you need two locks at once, maybe the data should be in one `Mutex<(A, B)>`.
4. **Switch to channels**. The actor model avoids shared state entirely, so deadlocks from lock ordering become impossible (though you can still deadlock if two actors wait for each other's responses).

### Single-thread deadlock

This one catches people off guard:

```rust
let guard = mutex.lock().unwrap();
// ... some code path eventually calls:
let guard2 = mutex.lock().unwrap(); // deadlock! same thread, same mutex
```

`std::sync::Mutex` is not reentrant. If the same thread tries to lock it twice, it blocks forever. [`parking_lot::ReentrantMutex`](https://docs.rs/parking_lot/latest/parking_lot/type.ReentrantMutex.html) allows this, but reentrant locking is usually a design smell. If you need it, your critical sections are probably too coarse.

### Detection tools

- [`parking_lot::deadlock`](https://amanieu.github.io/parking_lot/parking_lot/deadlock/) - call `check_deadlock()` periodically from a background thread. It inspects the lock dependency graph and reports cycles.
- [`no_deadlocks`](https://crates.io/crates/no_deadlocks) - drop-in replacement for std locks that detects deadlocks and conflicting lock orders at runtime.
- [`lockbud`](https://github.com/BurtonQin/lockbud) - static analysis tool that detects potential deadlocks and double-lock bugs without running the code.

## Decision matrix

Here's the practical summary. When you have a concurrency need, walk through this:

**"I need a shared counter or flag"** - Use `AtomicU64` / `AtomicBool`. No locking overhead. Use `Relaxed` ordering unless you're synchronizing other data.

**"I need to protect compound state, writes are frequent"** - Use `Mutex`. Keep the critical section short. Consider `parking_lot::Mutex` if you're on Linux and want better contention performance.

**"I need to protect state, reads vastly outnumber writes"** - Use `RwLock`. But benchmark against `Mutex` first - for small critical sections the difference is negligible, and `Mutex` is simpler. Consider `arc_swap` if the data is rarely updated.

**"I need to send work to a background processor"** - Use a bounded channel. `tokio::sync::mpsc` in async code, `crossbeam-channel` in sync code.

**"I need fan-out to a worker pool"** - Use `crossbeam-channel` (MPMC). Multiple workers pull from the same receiver.

**"I need to hold a lock across .await"** - Use `tokio::sync::Mutex` or `tokio::sync::RwLock`. This is the only reason to use the async variants.

**"I need to broadcast state changes to many consumers"** - `tokio::sync::watch` for latest-value, `tokio::sync::broadcast` for every-message.

The overwhelming majority of real-world Rust concurrency fits into one of these buckets. Pick the simplest primitive that solves your problem, benchmark if performance matters, and keep your critical sections short. The compiler handles data races. The architecture is on you.
