+++
title = "epoll, kqueue, io_uring - how async I/O actually works"
date = 2025-08-30
description = "A syscall-level tour of the readiness and completion APIs that let one thread juggle 10,000 sockets, and how tokio hides all of it behind .await."
[taxonomies]
tags = ["linux", "async", "rust", "networking"]
+++

When you `.await` a socket in tokio, no thread sits there spinning. The OS kernel is doing the waiting for you, and exactly one of three families of syscalls is carrying the weight: `epoll` on Linux, `kqueue` on the BSDs and macOS, and `io_uring` on newer Linux kernels. If you don't know which mechanism the runtime picked and how it differs from `select(2)`, it is hard to reason about why your server falls over at 20K connections or why a single `read` ballooned from 2 microseconds to 2 milliseconds.

This post walks from `select` through `io_uring`, shows the kernel data structures each one uses, and ends with numbers for handling 10K idle TCP connections on a laptop. I assume you already know what a socket and a file descriptor are. If you don't, I covered the wire side of a TCP connection in [Understanding TCP/IP - what happens when you curl a URL](/blog/understanding-tcp-ip-what-happens-when-you-curl-a-url/).

<!-- more -->

## The problem: one thread, many sockets

A naive TCP server looks like this:

```rust
let listener = TcpListener::bind("0.0.0.0:8080")?;
loop {
    let (mut stream, _) = listener.accept()?;
    std::thread::spawn(move || {
        let mut buf = [0u8; 1024];
        loop {
            let n = stream.read(&mut buf).unwrap();
            if n == 0 { break; }
            stream.write_all(&buf[..n]).unwrap();
        }
    });
}
```

One OS thread per connection. The default stack on glibc is 8 MiB of virtual memory per thread. At 10,000 connections that is 80 GiB of VM, plus the kernel task struct (around 8 KiB each), plus scheduler overhead every time one of them wakes up. The machine does not fall over from CPU, it falls over from context switches and memory. This is the [C10K problem](http://www.kegel.com/c10k.html) Dan Kegel wrote up in 1999, and the last twenty-five years of OS development have been about fixing it.

The fix is always the same shape: one thread manages many sockets by asking the kernel "which of these file descriptors are ready?" and only doing real work on the ones that are. Everything below is a variation on that theme.

## select(2) - the 1983 answer

`select` was in 4.2BSD. Its signature sets the tone for everything that comes after:

```c
int select(int nfds,
           fd_set *readfds,
           fd_set *writefds,
           fd_set *exceptfds,
           struct timeval *timeout);
```

An `fd_set` is a bitmap. On glibc it is 1024 bits. You set a bit for every fd you care about, hand the bitmap to the kernel, block, and when something is ready the kernel clears the bits that are not ready and returns. You then loop through every fd you registered and test whether its bit is still set.

Three problems, each fatal at scale:

1. **The 1024 cap.** `FD_SETSIZE` is a compile time constant. You can raise it, but every process in the ecosystem assumes 1024, so raising it is a minefield.
2. **O(n) on every call.** The bitmap is copied into the kernel, the kernel walks every fd in it to check readiness, then copies it back. Every iteration of your event loop. 10K fds means 10K checks, every single tick.
3. **Stateless.** The kernel does not remember which fds you care about. You hand them over again every time.

You can still find `select` in the wild in polling loops where the fd count is tiny (two or three) because it is the one poll primitive that exists on literally every Unix. Don't use it for servers.

## poll(2) - the same idea, without the cap

`poll` swaps the bitmap for an array of structs:

```c
struct pollfd {
    int   fd;
    short events;    // what you want to know about
    short revents;   // what actually happened
};

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

This removed the 1024 cap and it let you ask about specific events (read, write, hangup) instead of three fixed categories. But it is still O(n) per call and still stateless. You copy the whole `pollfd` array in on every call and the kernel walks all of it.

At a few hundred fds the constant factors beat `epoll`. At a few thousand it falls off a cliff.

## epoll (Linux, 2002) - keep state in the kernel

`epoll` was Linus Torvalds' response to C10K. The trick is simple: the kernel keeps a persistent data structure (an "interest list") of fds you want to watch. You register once, and subsequent syscalls only return the fds that became ready since you last asked.

Three syscalls:

```c
int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

- `epoll_create1` makes a new epoll fd. It is itself a file descriptor, which is important because you can put epoll fds inside epoll fds (hierarchical event loops).
- `epoll_ctl` adds, modifies, or removes fds from the interest list. The kernel keeps this in a red-black tree keyed by fd number. Insertion, lookup, deletion are all O(log n).
- `epoll_wait` blocks until something is ready and returns a flat array of ready events. This is O(k) where k is the number of ready fds, not the number registered.

The last part is what makes it scale. In a typical server with 10,000 connections, maybe 50 are ready at any tick. `select` looks at all 10,000. `epoll_wait` looks at 50.

Under the hood the kernel uses a callback. When you register fd X with `epoll_ctl`, the kernel walks the "wait queues" on the underlying file object and installs a callback. When data arrives on the socket, the network stack wakes those wait queues, the callback fires, and the fd gets appended to a per-epoll "ready list". `epoll_wait` just drains that ready list. No scanning, no copying lists of thousands of fds.

### Level vs edge triggered

`epoll` has two modes and choosing wrong will cost you correctness, not just performance.

- **Level triggered** (default) - `epoll_wait` keeps reporting an fd as ready as long as the condition holds. A socket with 100 bytes in its receive buffer shows up on every call until you read it empty.
- **Edge triggered** (`EPOLLET`) - `epoll_wait` reports the fd exactly once, when the condition transitions from not-ready to ready. You are obligated to drain the socket (read until `EAGAIN`) or you will never hear about it again.

Edge triggered is faster at scale because it produces fewer wakeups, but it requires nonblocking fds and careful drain loops. Every real runtime uses it.

## kqueue (macOS and BSD, 2000) - the prettier cousin

FreeBSD shipped `kqueue` two years before Linux shipped `epoll`, and it is a more general design. Instead of a specialized file-descriptor watcher, `kqueue` is a generic kernel event queue. The same API watches fds, signals, timers, process exits, file system changes, and custom user events.

Two syscalls:

```c
int kqueue(void);
int kevent(int kq,
           const struct kevent *changelist, int nchanges,
           struct kevent *eventlist, int nevents,
           const struct timespec *timeout);
```

`kevent` is both the "register interest" call and the "wait for events" call, merged. Each `struct kevent` has a filter (`EVFILT_READ`, `EVFILT_WRITE`, `EVFILT_TIMER`, `EVFILT_VNODE`, ...) and flags (`EV_ADD`, `EV_DELETE`, `EV_ONESHOT`, `EV_CLEAR`, ...). `EV_CLEAR` gives you edge-triggered behavior, equivalent to `EPOLLET`.

The performance characteristics are the same as `epoll`: persistent interest list in the kernel, O(1) readiness signaling via callbacks. The API is just nicer because timers and signals fall out of the same mechanism instead of needing `timerfd` and `signalfd` bolt-ons like Linux.

## io_uring (Linux 5.1, 2019) - readiness is not enough

Everything above is a **readiness** API. The kernel tells you "this fd is ready, go read it." You still have to syscall into the kernel to actually move the bytes. Under microbenchmarks that second hop dominates. With [Meltdown](https://meltdownattack.com) mitigations on, a syscall costs around 1 microsecond on a typical x86 box. If you are doing one small read per connection, that is most of your time budget.

`io_uring` is a **completion** API. You submit an I/O operation (read, write, accept, send, recv, even `openat` and `fsync`) and later you pick up the result. The kernel does the work asynchronously. No copy to a buffer you own until the data has actually arrived.

The design is two lock-free ring buffers shared between userspace and the kernel via `mmap`:

```
userspace ----[Submission Queue]----> kernel
userspace <---[Completion Queue]----- kernel
```

To read from a socket you write a submission queue entry (SQE) describing the read into the ring, optionally call `io_uring_enter(2)` to nudge the kernel, and later read a completion queue entry (CQE) with the result. When the SQ is non-empty and a kernel poll thread is configured (`IORING_SETUP_SQPOLL`), even the `io_uring_enter` syscall goes away. You can do I/O without syscalls.

Three things make it materially different from `epoll`:

1. **Batched submission.** You can queue dozens of reads with one syscall.
2. **True async for operations that were always blocking.** `read` on a regular file, `fsync`, `openat`. None of these worked with `epoll` because epoll only cared about "ready to read" and regular files are "always ready."
3. **Linked chains.** `IOSQE_IO_LINK` lets you say "run this accept, and when it finishes run this recv on the new fd." The second operation never round-trips through userspace.

The liburing library hides the ring management. Jens Axboe (the author) maintains it at [axboe/liburing](https://github.com/axboe/liburing).

There is a catch. `io_uring` has had a rough security history. Google disabled it for unprivileged users inside Chrome and ChromeOS in 2023 after a string of LPE bugs. It is off by default in several container runtimes. When it works it is the fastest thing on Linux. When it is blocked you fall back to `epoll`.

## mio - one Rust API over all three

Writing `cfg(target_os = "linux")` everywhere is miserable. The Rust ecosystem settled on [mio](https://github.com/tokio-rs/mio) as the thin portable layer. Its API is a deliberate trace of `epoll`:

```rust
use mio::{Events, Interest, Poll, Token};
use mio::net::TcpListener;

let mut poll = Poll::new()?;
let mut events = Events::with_capacity(1024);
let mut listener = TcpListener::bind("127.0.0.1:8080".parse()?)?;

poll.registry().register(&mut listener, Token(0), Interest::READABLE)?;

loop {
    poll.poll(&mut events, None)?;
    for event in events.iter() {
        match event.token() {
            Token(0) => { /* accept loop */ }
            Token(n) => { /* handle connection n */ }
        }
    }
}
```

Under the hood `Poll` is an `epoll` fd on Linux, a `kqueue` fd on macOS/BSD, and IOCP on Windows. mio does not currently wrap `io_uring` because `io_uring` is a completion model and mio's API is readiness. Bridging the two efficiently is an open problem; the community is converging on separate runtimes (glommio, monoio, tokio-uring) for the io_uring path.

## tokio - where the threads go

`tokio` is built on mio. Each worker thread owns a `Poll`. When you call `.await` on a `TcpStream::read`, the future's `poll` method tries the read in nonblocking mode. If the kernel returns `EAGAIN`, the future registers the task's waker against that fd in the reactor's `Poll` and returns `Pending`. The worker goes back to running other ready tasks. When `epoll_wait` returns with that fd ready, the reactor looks up the waker, calls `wake()`, and the task gets scheduled to run again.

The whole async/await machine is a user-space scheduler that funnels readiness events from `epoll` into wakers. Nothing fancier than that. If you want to see it, the reactor lives at [tokio-rs/tokio/tokio/src/runtime/io/driver.rs](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/io/driver.rs).

This is why blocking inside an `async fn` is lethal. The worker thread you are sitting on is the one that was supposed to run `epoll_wait`. If you `std::thread::sleep(Duration::from_secs(1))` in a handler, every socket parked on that worker waits a second. `tokio::task::spawn_blocking` moves the call to a separate pool precisely so the reactor thread stays free.

## The benchmark: 10,000 idle connections

I ran a simple experiment on a Ryzen 7 7840U laptop with kernel 6.8 to put numbers on the theory. The server accepts connections, reads a heartbeat byte once per second from each, and writes nothing. Client is 10K TCP connections from another process on loopback, all idle except one byte per second each.

| Model | Memory | CPU (%) | p99 wakeup latency |
|---|---:|---:|---:|
| Thread per connection | 1.8 GiB | 18 | 420 us |
| `poll()` loop | 92 MiB | 31 | 2.1 ms |
| `epoll` level triggered | 94 MiB | 4.8 | 180 us |
| `epoll` edge triggered | 94 MiB | 3.1 | 120 us |
| `io_uring` (SQPOLL off) | 97 MiB | 2.4 | 90 us |
| `io_uring` (SQPOLL on) | 97 MiB | 5.5* | 45 us |

*SQPOLL spins a kernel thread, so CPU goes up even though user code is doing less.

Two things to notice. First, `poll` is worse than everything including threads for CPU, because it walks 10K `pollfd` structs on every tick. Second, the gap between `epoll` and `io_uring` for this workload is small because almost nothing is happening. Under heavier traffic, where `io_uring` can batch dozens of reads per syscall, the gap widens. Jens Axboe's numbers on NVMe IOPS show 2-3x over `epoll`-plus-`read`.

## What to take away

Pick the mechanism that fits your OS and your workload, and let the runtime hide it:

- On Linux for network code, `epoll` is the baseline and what tokio uses today. It is mature, stable, available everywhere.
- On macOS/BSD, `kqueue` is equivalent in performance and broader in scope (timers, fs events, signals).
- `io_uring` is the future for Linux I/O-heavy workloads, especially disk-heavy ones. Check that your deployment target allows it before building on it.
- `select` and `poll` are legacy. Use them only when the fd count is small and portability to ancient Unix matters.
- If you are writing Rust, you almost never touch any of this directly. You write `async fn`, tokio picks mio, mio picks the best poll primitive the kernel exposes, and your code scales.

The next time a server of yours stops scaling around a few thousand connections, open `strace -c` on the process and look at which syscalls dominate. If it is `poll` or `select`, your runtime or library is the problem. If it is `epoll_wait`, the bottleneck is somewhere else and this post bought you nothing. That is still progress.
