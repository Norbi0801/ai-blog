+++
title = "Implementing a Simple TCP Chat in Rust"
date = 2025-10-25
description = "Building a multi-client chat server and client with tokio in ~200 lines - rooms, private messages, colored output, and both sides in a single binary."

[taxonomies]
tags = ["rust", "tokio", "networking", "async"]
+++

A TCP chat server is the "hello world" of async networking. It sounds trivial - accept connections, relay messages - but getting it right forces you to confront every interesting problem in concurrent I/O: shared mutable state across tasks, fan-out message delivery, half-open connections, clean disconnects, and the question of how to split a bidirectional socket into separate read and write paths without fighting the borrow checker.

We're going to build one in about 200 lines of Rust. Not a toy echo server - a real multi-room chat with username registration, private messages, room switching, and colored terminal output. Server and client live in the same binary, selected by subcommand. Three dependencies: [tokio](https://crates.io/crates/tokio), [clap](https://crates.io/crates/clap), and [colored](https://crates.io/crates/colored).

If you haven't read the [tokio internals post](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/) yet, you'll want to - we'll use `spawn`, `RwLock`, `mpsc` channels, and TCP primitives without explaining what the runtime does with them under the hood. The [concurrency primitives post](/blog/concurrency-primitives-in-rust-mutex-rwlock-channels-atomics) covers `Arc`, `RwLock`, and channels in depth. And if you want to see what raw TCP socket handling looks like without any async runtime, the [HTTP server from scratch post](/blog/writing-a-simple-http-server-from-scratch-in-rust) builds one with just `std::net`.

<!-- more -->

## What we're building

Two modes in one binary:

```bash
# Start the server
$ chat server --bind 127.0.0.1:4000

# Connect as a client (in another terminal)
$ chat connect --addr 127.0.0.1:4000 --name alice
```

Features:

- **Username registration** - first thing after connecting, reject duplicates
- **Rooms** - everyone starts in `general`, switch with `/join`
- **Broadcast** - messages go to everyone in your room
- **Private messages** - `/msg alice hey` delivers only to alice
- **Room listing and who's online** - `/rooms` and `/who`
- **Colored output** - system messages in yellow, DMs in magenta, info in cyan
- **Clean disconnect handling** - notify the room when someone drops

## The protocol

We're not implementing a real protocol like IRC ([RFC 1459](https://www.rfc-editor.org/rfc/rfc1459)). Our "protocol" is newline-delimited text. That's it.

The handshake:

1. Client connects via TCP
2. Client sends username as the first line: `alice\n`
3. Server responds `OK\n` or `ERR: name taken\n`

After that, the client sends lines and the server routes them. Lines starting with `/` are commands, everything else is a chat message broadcast to the current room. The server sends messages to clients as plain text - the client colorizes based on prefix patterns (`***` for system events, `[DM from ...]` for private messages).

No binary framing, no length prefixes, no message IDs. For a production chat you'd want all of those. Here we get away with newline-delimited text because `BufReader::read_line` handles the framing for us, and we're not sending binary data.

## Project setup

```toml
[package]
name = "chat"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1.50", features = ["full"] }
clap = { version = "4.6", features = ["derive"] }
colored = "3"
```

Three crates. `tokio` with `full` enables the multi-threaded runtime, TCP, I/O utilities, sync primitives, and signal handling. `clap` with `derive` gives us the subcommand parsing. `colored` adds `.red()`, `.green()`, etc. as extension methods on strings via the `Colorize` trait.

Single file: `src/main.rs`. Everything below goes in it.

## Shared state: the hard part

The interesting architectural question in any chat server is: how do you deliver a message from one client's task to another?

You have N connected clients, each handled by a separate tokio task. When alice sends "hello", the server needs to fan that message out to every other task handling a client in the same room. Those tasks are running concurrently - possibly on different OS threads if you're on the multi-threaded runtime.

There are two common approaches:

**Broadcast channel** - tokio's `broadcast::channel` gives you multi-producer, multi-consumer semantics. Every sender can send, every receiver gets every message. Sounds perfect for chat, but it has a problem: it's a bounded channel, and if any receiver falls behind, messages get dropped with a `RecvError::Lagged` error. That could mean lost chat messages. Also, it doesn't give you per-user routing - you'd need to filter at each receiver, which wastes work for private messages and rooms.

**Per-client mpsc channel** - give each client their own `mpsc::UnboundedSender`. Store all senders in a shared `HashMap<String, Sender>`. When you want to send a message to alice, look up her sender and push the message. For room broadcast, iterate over all members and send to each. This gives you precise routing and no lagged-receiver problem (at the cost of unbounded queue growth if a client stalls - more on that later).

We'll use the per-client channel approach. It maps cleanly to rooms and private messages.

Here's the state:

```rust
use std::collections::{HashMap, HashSet};
use std::sync::Arc;

use tokio::sync::{mpsc, RwLock};

type Tx = mpsc::UnboundedSender<String>;

struct State {
    users: HashMap<String, Tx>,
    rooms: HashMap<String, HashSet<String>>,
}
```

`users` maps usernames to their outbound channel. `rooms` maps room names to the set of usernames in that room. Both are behind `Arc<RwLock<State>>` so every task can access them.

Why `RwLock` and not `Mutex`? Because the most common operation - broadcasting a message - only needs to read the state (look up senders, check room membership). Multiple broadcasts can happen concurrently without blocking each other. We only take a write lock when someone joins, leaves, or disconnects. The [concurrency primitives post](/blog/concurrency-primitives-in-rust-mutex-rwlock-channels-atomics) covers this tradeoff in detail - `RwLock` pays a slightly higher per-lock cost in exchange for much better throughput when reads dominate writes.

We're using `tokio::sync::RwLock`, not `std::sync::RwLock`. The tokio version is designed to be held across `.await` points - its guard is `Send`. The std version's guard is not `Send`, and if you try to hold it across an await in a multi-threaded runtime, you'll get a compile error. That said, tokio's `RwLock` is heavier than std's - it uses an internal semaphore instead of the OS futex. For short critical sections where you don't await while holding the lock, `std::sync::RwLock` wrapped in `tokio::task::block_in_place` is faster. Our locks are short enough that it doesn't matter.

The helper methods on `State`:

```rust
impl State {
    fn new() -> Self {
        let mut rooms = HashMap::new();
        rooms.insert("general".into(), HashSet::new());
        State { users: HashMap::new(), rooms }
    }

    fn broadcast(&self, room: &str, msg: &str, skip: Option<&str>) {
        if let Some(members) = self.rooms.get(room) {
            for name in members {
                if skip == Some(name.as_str()) {
                    continue;
                }
                if let Some(tx) = self.users.get(name) {
                    let _ = tx.send(msg.to_string());
                }
            }
        }
    }

    fn send_to(&self, target: &str, msg: &str) -> bool {
        self.users
            .get(target)
            .map(|tx| tx.send(msg.to_string()).is_ok())
            .unwrap_or(false)
    }

    fn join_room(&mut self, room: &str, user: &str) {
        self.rooms
            .entry(room.to_string())
            .or_default()
            .insert(user.to_string());
    }

    fn leave_all_rooms(&mut self, user: &str) {
        for members in self.rooms.values_mut() {
            members.remove(user);
        }
    }

    fn user_room(&self, user: &str) -> Option<String> {
        self.rooms
            .iter()
            .find(|(_, members)| members.contains(user))
            .map(|(room, _)| room.clone())
    }
}
```

`broadcast` iterates over room members and sends to each. The `skip` parameter excludes the sender - you don't want to see your own message echoed back. Note the `let _ = tx.send(...)` - we're intentionally ignoring send failures. If a client's receiver has been dropped (they disconnected but we haven't cleaned up yet), the send fails silently. In a more robust system you'd track these failures and remove dead clients proactively.

`user_room` finds which room a user is in by scanning all rooms. With a few hundred users this is fine. At scale, you'd add a reverse index (`HashMap<String, String>` mapping user to room) to make it O(1). But we're building a chat for learning, not for Discord's 200 million users.

## The server

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};

async fn run_server(bind: &str) {
    let state = Arc::new(RwLock::new(State::new()));
    let listener = TcpListener::bind(bind).await.unwrap();
    println!("listening on {}", bind);

    loop {
        let (stream, addr) = listener.accept().await.unwrap();
        println!("connection from {}", addr);
        let state = Arc::clone(&state);
        tokio::spawn(handle_client(state, stream));
    }
}
```

The accept loop is the same pattern you see in every async TCP server. `TcpListener::bind` creates the socket, binds, and listens (three syscalls: `socket`, `bind`, `listen`). Each `accept().await` parks the task until a new connection arrives. When it does, we clone the `Arc` (cheap atomic increment) and spawn a new task to handle the client.

Every `tokio::spawn` here creates a lightweight task on the runtime's scheduler - not an OS thread. If you have 10,000 connected clients, you have 10,000 tasks, all multiplexed over a handful of worker threads (one per CPU core by default). The [tokio internals post](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/) explains exactly how the work-stealing scheduler distributes these tasks.

### The client handler

This is where the real work happens:

```rust
async fn handle_client(state: Arc<RwLock<State>>, stream: TcpStream) {
    let (reader, mut writer) = stream.into_split();
    let mut reader = BufReader::new(reader);
    let mut line = String::new();

    // Step 1: read username
    if reader.read_line(&mut line).await.unwrap_or(0) == 0 {
        return;
    }
    let name = line.trim().to_string();
    line.clear();

    // Step 2: register
    let (tx, mut rx) = mpsc::unbounded_channel();
    {
        let mut s = state.write().await;
        if s.users.contains_key(&name) {
            let _ = writer.write_all(b"ERR: name taken\n").await;
            return;
        }
        s.users.insert(name.clone(), tx);
        s.join_room("general", &name);
        s.broadcast(
            "general",
            &format!("*** {} joined general ***\n", name),
            Some(&name),
        );
    }
    let _ = writer.write_all(b"OK\n").await;

    // Step 3: spawn writer task
    let write_task = tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            if writer.write_all(msg.as_bytes()).await.is_err() {
                break;
            }
        }
    });

    // Step 4: read loop
    while reader.read_line(&mut line).await.unwrap_or(0) > 0 {
        let msg = line.trim().to_string();
        line.clear();
        if msg.is_empty() {
            continue;
        }

        if let Some(room) = msg.strip_prefix("/join ") {
            let room = room.trim();
            let mut s = state.write().await;
            s.leave_all_rooms(&name);
            s.join_room(room, &name);
            s.broadcast(
                room,
                &format!("*** {} joined {} ***\n", name, room),
                Some(&name),
            );
        } else if let Some(rest) = msg.strip_prefix("/msg ") {
            if let Some((target, text)) = rest.split_once(' ') {
                let s = state.read().await;
                let dm = format!("[DM from {}] {}\n", name, text);
                if !s.send_to(target, &dm) {
                    let _ = s.send_to(&name, &format!("*** {} is not online ***\n", target));
                }
            }
        } else if msg == "/rooms" {
            let s = state.read().await;
            let list: Vec<_> = s
                .rooms
                .iter()
                .map(|(r, m)| format!("  {} ({})", r, m.len()))
                .collect();
            let _ = s.send_to(&name, &format!("Rooms:\n{}\n", list.join("\n")));
        } else if msg == "/who" {
            let s = state.read().await;
            if let Some(room) = s.user_room(&name) {
                let members: Vec<_> = s.rooms[&room].iter().cloned().collect();
                let _ = s.send_to(&name, &format!("In {}: {}\n", room, members.join(", ")));
            }
        } else {
            let s = state.read().await;
            if let Some(room) = s.user_room(&name) {
                s.broadcast(&room, &format!("{}: {}\n", name, msg), Some(&name));
            }
        }
    }

    // Step 5: cleanup on disconnect
    {
        let mut s = state.write().await;
        if let Some(room) = s.user_room(&name) {
            s.broadcast(&room, &format!("*** {} left ***\n", name), Some(&name));
        }
        s.leave_all_rooms(&name);
        s.users.remove(&name);
    }
    write_task.abort();
}
```

There are a few things worth unpacking here.

### `into_split()` and the ownership problem

A `TcpStream` is a single bidirectional resource. We need to read from it in one task and write to it in another. Rust's ownership model doesn't let two tasks hold `&mut stream` simultaneously - which is exactly the point.

`stream.into_split()` solves this by consuming the stream and returning two halves: `OwnedReadHalf` and `OwnedWriteHalf`. Internally, both halves hold an `Arc<TcpStream>` - the underlying file descriptor is shared, but the read and write halves are separate types that only expose read or write operations respectively. The OS kernel maintains separate read and write buffers for each socket, so concurrent reads and writes on the same fd are safe at the syscall level.

There's also `stream.split()` which returns borrowed halves using references instead of `Arc`. It's slightly cheaper (no atomic increment), but the halves can't be moved into separate tasks because they borrow the original stream. Since we need to `tokio::spawn` the write task, we need owned halves.

### Why a separate write task?

We could read from the socket and write to it in the same task, using `tokio::select!` to multiplex between "new data from the socket" and "new message from the channel". That works fine:

```rust
loop {
    tokio::select! {
        result = reader.read_line(&mut line) => { /* handle input */ }
        Some(msg) = rx.recv() => { /* write to socket */ }
    }
}
```

The `select!` approach has one subtlety: `read_line` is not cancel-safe. If the `select!` cancels a partially-completed `read_line` because the channel branch fired first, data already read into the internal buffer may be lost. You'd need to use `BufReader::lines()` with `next_line()` instead, which is cancel-safe because it manages its own buffer.

The separate task approach sidesteps this entirely. It also decouples the read and write paths - if a write blocks (the client's TCP receive buffer is full because they're not reading), it doesn't stall the read loop. In practice for a chat app it barely matters. But the pattern of splitting readers and writers into separate tasks is worth knowing - it's how most production TCP servers handle bidirectional communication.

### Unbounded channels: the tradeoff

We're using `mpsc::unbounded_channel()`. An unbounded channel never exerts backpressure - sends always succeed immediately (unless the receiver is dropped). This means if a client stops reading from their TCP connection but stays connected, messages pile up in their channel without limit. In theory, a slow client could eat all your server's memory.

For a chat app among friends, this is fine. For a production system, you'd use a bounded channel (`mpsc::channel(100)`) and either drop messages when the channel is full or disconnect slow clients. The [backpressure post](/blog/backpressure-when-your-system-cant-keep-up) covers this in depth.

### The cleanup dance

When the read loop ends (client disconnected or read error), we take a write lock, broadcast the leave message, remove the user from all rooms, and remove their sender from the users map. Then we `abort()` the write task.

`abort()` is a hard cancel - it drops the task's future at the next `.await` point. This is safe here because the write task's only job is forwarding channel messages to the socket. Once the client has disconnected, there's nothing useful to write. The socket's write half will be dropped, which triggers a TCP `FIN` sequence (or `RST` if there's buffered data that wasn't acknowledged).

## The client

```rust
use colored::Colorize;

async fn run_client(addr: &str, name: &str) {
    let stream = TcpStream::connect(addr).await.unwrap();
    let (reader, mut writer) = stream.into_split();
    let mut reader = BufReader::new(reader);

    // Send username
    writer
        .write_all(format!("{}\n", name).as_bytes())
        .await
        .unwrap();

    // Check server response
    let mut line = String::new();
    reader.read_line(&mut line).await.unwrap();
    if line.trim() != "OK" {
        eprintln!("{}", line.trim().red());
        return;
    }
    println!(
        "{}",
        format!("Connected as {}. Type /help for commands.", name).green()
    );

    // Spawn a task to read from server and print
    tokio::spawn(async move {
        let mut line = String::new();
        loop {
            line.clear();
            match reader.read_line(&mut line).await {
                Ok(0) | Err(_) => {
                    eprintln!("{}", "Disconnected from server.".red());
                    std::process::exit(0);
                }
                Ok(_) => {
                    let msg = line.trim();
                    if msg.starts_with("***") {
                        println!("{}", msg.yellow());
                    } else if msg.starts_with("[DM") {
                        println!("{}", msg.magenta());
                    } else if msg.starts_with("Rooms:") || msg.starts_with("In ") {
                        println!("{}", msg.cyan());
                    } else {
                        println!("{}", msg);
                    }
                }
            }
        }
    });

    // Read from stdin, send to server
    let stdin = BufReader::new(tokio::io::stdin());
    let mut lines = stdin.lines();
    while let Ok(Some(input)) = lines.next_line().await {
        if input.trim() == "/help" {
            println!("{}", "/join <room>   - switch to a room".cyan());
            println!("{}", "/msg <user> <text> - send a private message".cyan());
            println!("{}", "/rooms         - list all rooms".cyan());
            println!("{}", "/who           - who's in your room".cyan());
            continue;
        }
        if writer
            .write_all(format!("{}\n", input).as_bytes())
            .await
            .is_err()
        {
            break;
        }
    }
}
```

The client is simpler. Same `into_split()` pattern: one task reads from the server and prints colored output, the main task reads from stdin and sends to the server.

The colored output uses prefix-based detection. Messages starting with `***` are system events (joins, leaves) - yellow. Messages starting with `[DM` are private messages - magenta. Room listings and who output - cyan. Everything else is plain text. This is crude but effective. A real protocol would include message types in a structured format instead of relying on string prefixes.

`tokio::io::stdin()` deserves a note. Reading from stdin in async code is tricky. The actual `read(0, ...)` syscall on stdin is blocking - it waits for the user to type something. Tokio handles this by running stdin reads on a dedicated blocking thread pool (the same pool used by `spawn_blocking`). The `AsyncRead` implementation for `Stdin` delegates to that pool, so it doesn't block the async worker threads. You can verify this with `strace` - you'll see the `read(0, ...)` call happening on a different thread than the `epoll_wait` calls.

## One binary, two modes

```rust
use clap::Parser;

#[derive(Parser)]
#[command(name = "chat", about = "TCP chat - server and client in one binary")]
enum Cli {
    /// Start the chat server
    Server {
        /// Address to bind to
        #[arg(short, long, default_value = "127.0.0.1:4000")]
        bind: String,
    },
    /// Connect to a chat server
    Connect {
        /// Server address
        #[arg(short, long, default_value = "127.0.0.1:4000")]
        addr: String,
        /// Your username
        #[arg(short, long)]
        name: String,
    },
}

#[tokio::main]
async fn main() {
    match Cli::parse() {
        Cli::Server { bind } => run_server(&bind).await,
        Cli::Connect { addr, name } => run_client(&addr, &name).await,
    }
}
```

Clap's derive API on an enum gives us subcommands. `chat server` and `chat connect` each get their own arguments. The `#[tokio::main]` macro sets up the multi-threaded runtime - for the server this matters (multiple tasks across cores), for the client it's overkill but harmless.

## What happens under the hood

Trace what actually happens at the syscall level when alice sends "hello" and it reaches bob.

**Alice's terminal:**

1. Alice types "hello" and hits enter
2. The `read(0, ...)` syscall on the stdin blocking thread returns `"hello\n"`
3. The blocking thread wakes the async task via a waker
4. The async task calls `writer.write_all(b"hello\n")`, which does a `write(fd, "hello\n", 6)` syscall on alice's TCP socket

**Server - alice's handler task:**

5. The I/O driver's `epoll_wait` returns a readiness event for alice's socket fd
6. The scheduler wakes alice's handler task
7. `reader.read_line()` does a `read(fd, ...)` syscall, gets `"hello\n"`
8. The handler acquires a read lock on `State`
9. Finds alice is in `general`, iterates over members, calls `tx.send()` for bob (and any other members)
10. The `send()` pushes the string into bob's channel and wakes bob's write task

**Server - bob's write task:**

11. Bob's write task wakes up, calls `rx.recv()`, gets `"alice: hello\n"`
12. Calls `writer.write_all(b"alice: hello\n")`, which does a `write(fd, ...)` on bob's socket
13. The kernel puts the bytes into bob's TCP send buffer, eventually transmits them

**Bob's terminal:**

14. Bob's client's `epoll_wait` fires when bob's socket becomes readable
15. Bob's reader task wakes, calls `read_line()`, gets `"alice: hello\n"`
16. Prefix doesn't match any special pattern, so it's printed as plain text

Four `read` syscalls, two `write` syscalls, a couple of `epoll_wait` returns, an `RwLock` acquisition, and a channel send/recv. All without allocating a thread per connection, without copying the message more than necessary, and without any explicit callbacks or event loop wiring.

You can see this yourself with strace:

```bash
strace -f -e trace=read,write,epoll_wait target/debug/chat server --bind 127.0.0.1:4000
```

The `-f` flag follows child threads (tokio's worker threads are OS threads), and you'll see the `epoll_wait` calls on the worker threads interspersed with `read` and `write` calls on specific socket file descriptors.

## What we left out

This is a learning project, not a production chat server. Here's what a real system would need:

**Backpressure on slow clients.** Our unbounded channels mean a client that stops reading can cause unbounded memory growth on the server. Use `mpsc::channel(n)` with a bounded capacity and either drop messages or disconnect when the buffer fills.

**Authentication.** Anyone can claim any username. A real system needs auth tokens, not just "first line is your name."

**Message persistence.** When bob disconnects and reconnects, he's lost everything. A production chat stores messages in a database and replays missed ones on reconnect.

**TLS.** We're sending plaintext over TCP. Everything is readable by anyone who can sniff the network. Wrap the `TcpStream` in [tokio-rustls](https://crates.io/crates/tokio-rustls) for encrypted connections.

**Proper framing.** Newline-delimited text breaks if someone sends a message containing a newline (which our terminal client won't, but a programmatic client could). Length-prefixed binary frames are more robust. [tokio-util's `LengthDelimitedCodec`](https://docs.rs/tokio-util/latest/tokio_util/codec/length_delimited/index.html) handles this.

**Graceful shutdown.** If you Ctrl+C the server, all clients get a `RST` packet and an ugly disconnection. The [graceful shutdown post](/blog/graceful-shutdown-in-async-rust-handling-sigterm-properly) covers how to drain connections cleanly using `CancellationToken`.

**Rate limiting.** A misbehaving client can flood a room. Track message timestamps per client and disconnect or throttle when they exceed a threshold.

The full code is about 200 lines. Clone it, run it, open three terminals, and watch the messages flow. Then add one of the missing features above - TLS is a good starting point because `tokio-rustls` is a drop-in wrapper around `TcpStream`. That exercise will teach you more about async I/O composition than reading another ten blog posts.
