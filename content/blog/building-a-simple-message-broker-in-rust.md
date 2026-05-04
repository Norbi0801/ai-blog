+++
title = "Building a simple message broker in Rust"
date = 2025-07-08
description = "Pub/sub from scratch in ~400 lines of Rust. Topics, subscribers, consumer groups, append-only persistence, and acknowledgments over a plain TCP protocol."

[taxonomies]
tags = ["rust", "networking", "tokio", "messaging"]
+++

NATS, Redis pub/sub, Kafka, RabbitMQ. They look very different on the outside, but if you squint at the small ones they are mostly the same machine: a TCP server, a few channels per topic, an append-only log, and some bookkeeping for who has read what. Once you have written that machine yourself, the docs of the real brokers stop reading like magic and start reading like a list of decisions.

This post builds a pub/sub broker from scratch in about four hundred lines of Rust. It speaks a tiny text protocol over TCP, persists every message to an append-only file, supports plain fan-out subscribers and consumer groups, and re-delivers messages that nobody acked. The networking layer assumes you are comfortable with TCP - if not, I covered the wire-level mechanics in [Understanding TCP/IP](/blog/understanding-tcp-ip-what-happens-when-you-curl-a-url/).

<!-- more -->

## What pub/sub actually is

Pub/sub is a delivery pattern, not a piece of software. A *publisher* sends a message tagged with a *topic*. The broker takes that message and hands it to every *subscriber* that registered interest in that topic. The publisher does not know who the subscribers are, and the subscribers do not know who the publisher is. The broker is the only thing that knows both ends.

There are three broad flavours you will see in the wild:

- **Pure fan-out.** Every subscriber gets every message. If a subscriber is offline, it misses everything. This is what Redis pub/sub does. It is fast and simple and forgets immediately.
- **Persistent log.** Every message is appended to a durable log. Subscribers read at their own pace and remember a position (an offset). If they fall behind, they catch up. This is Kafka.
- **Consumer groups on a queue.** A topic is a queue. Multiple subscribers form a *group*, and each message is delivered to exactly one member of the group. Different groups each get their own copy. NATS JetStream and RabbitMQ both do versions of this.

We are going to support all three modes in the same broker, because the differences are smaller than they look.

## Architecture

The broker is a single process. It owns the state of every topic. Clients connect over TCP, send commands, and read messages off the same socket. Internally we use tokio's in-process channels for fan-out and per-group queues, so each connection is just a thin task that translates between TCP frames and channel sends.

```
                +----------------------------+
  PUB orders -> |        TCP listener        | -> connection task
                +-------------+--------------+
                              |
                              v
                +----------------------------+
                |          Broker            |
                |  topics: HashMap<name, T>  |
                +-------------+--------------+
                              |
            +-----------------+-----------------+
            v                 v                 v
       broadcast::Sender   append-only      consumer groups
       (fan-out subs)      log file         (mpsc per group)
```

Three things matter here:

1. **`broadcast::Sender`** gives us cheap fan-out. Every plain subscriber gets a `broadcast::Receiver`, and a single `send` reaches all of them. If a slow subscriber falls more than `N` messages behind, it gets a `Lagged` error and we just drop it. Same trade-off as Redis pub/sub.
2. **The append-only log** is a plain file we open with `O_APPEND`. Every message is written before fan-out happens. If the process crashes mid-publish, the log is still consistent up to the last `flush`. We do not implement replay in this version, but the data is there.
3. **Consumer groups** are an `mpsc::Sender` per group. The broker pushes into the queue once per group, and group members pull from it in round-robin order. A pending-ack table tracks which messages are in flight, with a re-delivery timer.

## Wire protocol

The protocol is line-based with binary payloads. Every command is one ASCII line ending in `\n`. Commands that carry a payload include a byte length and the payload follows the line:

```
PUB <topic> <len>\n<bytes>
SUB <topic>\n
JOIN <topic> <group>\n
ACK <topic> <group> <id>\n
```

The server answers with:

```
OK <id>\n              -> for PUB
MSG <id> <topic> <len>\n<bytes>   -> for delivered messages
ERR <reason>\n         -> on protocol or runtime errors
```

This is intentionally close to NATS, which uses `PUB`, `SUB`, and `MSG` verbs over plain TCP with a similar header line. NATS adds a few fields (subject wildcards, reply subjects, sids) but the spirit is the same: text headers, opaque payloads, no framing protocol.

## The Cargo file

```toml
[package]
name = "tinybroker"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1.40", features = ["full"] }
anyhow = "1"
```

That is it. No serde, no extra channels crate, no logging library. We do everything with the std and tokio types.

## The state

```rust
use std::collections::HashMap;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use tokio::fs::{File, OpenOptions};
use tokio::io::{AsyncBufReadExt, AsyncReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::{broadcast, mpsc, Mutex, RwLock};
use tokio::time::{sleep, Duration, Instant};

#[derive(Clone, Debug)]
struct Message {
    id: u64,
    topic: String,
    payload: Vec<u8>,
}

struct Topic {
    fanout: broadcast::Sender<Message>,
    log: Mutex<File>,
    groups: Mutex<HashMap<String, Arc<Group>>>,
}

struct Group {
    name: String,
    tx: mpsc::Sender<Message>,
    rx: Mutex<mpsc::Receiver<Message>>,
    pending: Mutex<HashMap<u64, (Message, Instant)>>,
}

struct Broker {
    topics: RwLock<HashMap<String, Arc<Topic>>>,
    next_id: AtomicU64,
    data_dir: String,
}
```

A few choices worth calling out:

- `Message` is `Clone` because we hand the same payload to every subscriber. The `Vec<u8>` allocation is cloned per delivery; for production you would wrap it in `Arc<[u8]>`. For a learning broker this is fine.
- `Group` holds both the `Sender` and the `Receiver` because the receiver is shared among consumers in the same group. We wrap it in a `Mutex` so multiple consumer tasks can pull from it. That is exactly how Kafka and RabbitMQ make a group "round-robin" - one queue, many takers.
- `pending` maps in-flight message IDs to the original message plus the time we last delivered it. The redelivery loop walks this map every second.

## Opening or creating a topic

```rust
impl Broker {
    async fn new(data_dir: &str) -> anyhow::Result<Arc<Self>> {
        tokio::fs::create_dir_all(data_dir).await?;
        Ok(Arc::new(Self {
            topics: RwLock::new(HashMap::new()),
            next_id: AtomicU64::new(1),
            data_dir: data_dir.into(),
        }))
    }

    async fn topic(&self, name: &str) -> anyhow::Result<Arc<Topic>> {
        if let Some(t) = self.topics.read().await.get(name) {
            return Ok(t.clone());
        }
        let mut topics = self.topics.write().await;
        if let Some(t) = topics.get(name) {
            return Ok(t.clone());
        }
        let path = format!("{}/{}.log", self.data_dir, name);
        let file = OpenOptions::new()
            .create(true).append(true).read(true).open(&path).await?;
        let (fanout, _) = broadcast::channel(1024);
        let topic = Arc::new(Topic {
            fanout,
            log: Mutex::new(file),
            groups: Mutex::new(HashMap::new()),
        });
        topics.insert(name.to_string(), topic.clone());
        Ok(topic)
    }
}
```

The double-check pattern (read lock, drop, write lock, read again) avoids a thundering-herd of `create_dir_all` calls when a hundred clients connect at once. It is the same shape as Java's double-checked locking, except `RwLock` makes it actually safe.

## Publishing

Publishing is the only place where we touch the disk synchronously. We hold the topic log mutex while we write the header line, the payload, and a trailing newline. Holding a mutex across an `await` is normally a smell, but `tokio::sync::Mutex` is built for exactly that case - it parks the task instead of the OS thread.

```rust
async fn publish(&self, topic_name: &str, payload: Vec<u8>) -> anyhow::Result<u64> {
    let topic = self.topic(topic_name).await?;
    let id = self.next_id.fetch_add(1, Ordering::Relaxed);
    let msg = Message { id, topic: topic_name.into(), payload };
    {
        let mut f = topic.log.lock().await;
        let header = format!("{} {}\n", msg.id, msg.payload.len());
        f.write_all(header.as_bytes()).await?;
        f.write_all(&msg.payload).await?;
        f.write_all(b"\n").await?;
        f.flush().await?;
    }
    let _ = topic.fanout.send(msg.clone()); // best effort
    let groups = topic.groups.lock().await;
    for g in groups.values() {
        let _ = g.tx.send(msg.clone()).await;
    }
    Ok(id)
}
```

The order matters: log first, fan-out second. If we crashed between the broadcast send and the disk write, a subscriber would have a message that never persisted. With log-first, the worst case is a message in the log that no live subscriber received, which is the safer failure mode.

`flush().await` is a `fdatasync`-equivalent in tokio's eyes but actually only flushes the userland buffer. For real durability you would call `f.sync_data().await?` and pay the price - on a consumer SSD that is one to ten milliseconds per message. NATS JetStream lets you opt into this per-stream; Kafka does it in batches with `log.flush.interval.messages`. We just write through and trust the page cache.

## A consumer group

A group is created lazily the first time a client joins it. The mpsc channel is bounded so a slow group eventually back-pressures the publisher (or, with `try_send`, drops messages):

```rust
async fn join_group(topic: &Arc<Topic>, name: &str) -> Arc<Group> {
    let mut groups = topic.groups.lock().await;
    if let Some(g) = groups.get(name) {
        return g.clone();
    }
    let (tx, rx) = mpsc::channel::<Message>(1024);
    let g = Arc::new(Group {
        name: name.into(),
        tx,
        rx: Mutex::new(rx),
        pending: Mutex::new(HashMap::new()),
    });
    groups.insert(name.into(), g.clone());
    g
}
```

When a consumer pulls a message, we move it into `pending` with the current time. When the consumer sends `ACK`, we drop it. A background task scans `pending` and re-queues messages whose deadline has passed:

```rust
async fn redelivery_loop(group: Arc<Group>) {
    loop {
        sleep(Duration::from_secs(1)).await;
        let mut pending = group.pending.lock().await;
        let now = Instant::now();
        let stale: Vec<u64> = pending.iter()
            .filter(|(_, (_, t))| now.duration_since(*t) > Duration::from_secs(10))
            .map(|(id, _)| *id)
            .collect();
        for id in stale {
            if let Some((msg, _)) = pending.remove(&id) {
                let _ = group.tx.send(msg).await;
            }
        }
    }
}
```

This is the classic "visibility timeout" pattern from SQS. Ten seconds is a placeholder; production brokers expose this per-subscription.

## The connection handler

Each TCP connection is one tokio task. It reads commands, dispatches them, and writes back responses. Because reads and writes happen on different code paths (the writer also gets pushed messages from the broadcast/group channels), we split the socket and own each half independently:

```rust
async fn handle_client(stream: TcpStream, broker: Arc<Broker>) -> anyhow::Result<()> {
    let (read_half, mut write_half) = stream.into_split();
    let mut reader = BufReader::new(read_half);
    let (out_tx, mut out_rx) = mpsc::channel::<Vec<u8>>(64);

    // writer task: drains out_rx into the socket
    let writer = tokio::spawn(async move {
        while let Some(buf) = out_rx.recv().await {
            if write_half.write_all(&buf).await.is_err() { break; }
        }
    });

    let mut line = String::new();
    loop {
        line.clear();
        let n = reader.read_line(&mut line).await?;
        if n == 0 { break; } // client disconnected
        let line = line.trim_end();
        let mut parts = line.splitn(4, ' ');
        match parts.next() {
            Some("PUB") => {
                let topic = parts.next().unwrap_or("");
                let len: usize = parts.next().unwrap_or("0").parse().unwrap_or(0);
                let mut buf = vec![0u8; len];
                reader.read_exact(&mut buf).await?;
                let mut nl = [0u8; 1];
                let _ = reader.read_exact(&mut nl).await;
                let id = broker.publish(topic, buf).await?;
                let _ = out_tx.send(format!("OK {}\n", id).into_bytes()).await;
            }
            Some("SUB") => {
                let topic_name = parts.next().unwrap_or("").to_string();
                let topic = broker.topic(&topic_name).await?;
                let mut rx = topic.fanout.subscribe();
                let out = out_tx.clone();
                tokio::spawn(async move {
                    while let Ok(msg) = rx.recv().await {
                        let header = format!("MSG {} {} {}\n",
                            msg.id, msg.topic, msg.payload.len());
                        let mut buf = header.into_bytes();
                        buf.extend_from_slice(&msg.payload);
                        buf.push(b'\n');
                        if out.send(buf).await.is_err() { break; }
                    }
                });
            }
            Some("JOIN") => {
                let topic_name = parts.next().unwrap_or("").to_string();
                let group_name = parts.next().unwrap_or("").to_string();
                let topic = broker.topic(&topic_name).await?;
                let group = join_group(&topic, &group_name).await;
                spawn_redeliverer_once(group.clone());
                let out = out_tx.clone();
                let g = group.clone();
                tokio::spawn(async move {
                    loop {
                        let msg = { g.rx.lock().await.recv().await };
                        let Some(msg) = msg else { break };
                        g.pending.lock().await.insert(msg.id, (msg.clone(), Instant::now()));
                        let header = format!("MSG {} {} {}\n",
                            msg.id, msg.topic, msg.payload.len());
                        let mut buf = header.into_bytes();
                        buf.extend_from_slice(&msg.payload);
                        buf.push(b'\n');
                        if out.send(buf).await.is_err() { break; }
                    }
                });
            }
            Some("ACK") => {
                let topic_name = parts.next().unwrap_or("");
                let group_name = parts.next().unwrap_or("");
                let id: u64 = parts.next().unwrap_or("0").parse().unwrap_or(0);
                broker.ack(topic_name, group_name, id).await?;
            }
            _ => {
                let _ = out_tx.send(b"ERR unknown command\n".to_vec()).await;
            }
        }
    }
    drop(out_tx);
    let _ = writer.await;
    Ok(())
}
```

The trick that makes this clean is the `out_tx`/`out_rx` mpsc channel. The reader task, the broadcast subscriber task, and the consumer group task all write into the same `out_tx`. A single writer task drains it. We never touch the raw socket from more than one place, so there are no `Mutex<TcpStream>` shenanigans and no interleaved frames.

`spawn_redeliverer_once` is just a helper that uses an atomic flag on the `Group` to make sure only the first joiner starts the timer task. Code omitted for space; it is five lines.

## The acks path

```rust
impl Broker {
    async fn ack(&self, topic_name: &str, group_name: &str, id: u64)
        -> anyhow::Result<()>
    {
        let topic = self.topic(topic_name).await?;
        if let Some(g) = topic.groups.lock().await.get(group_name) {
            g.pending.lock().await.remove(&id);
        }
        Ok(())
    }
}
```

That is the whole correctness model. A message is "delivered" when it leaves the broker socket. It is "processed" when the consumer sends `ACK`. Anything in between is the broker's responsibility. If the consumer crashes after receiving but before acking, the redelivery loop sends it again to whichever group member is alive. This is at-least-once semantics, the same guarantee Kafka and SQS give you out of the box.

## Wiring `main`

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let broker = Broker::new("./data").await?;
    let listener = TcpListener::bind("127.0.0.1:7878").await?;
    println!("tinybroker listening on 127.0.0.1:7878");
    loop {
        let (sock, _) = listener.accept().await?;
        let b = broker.clone();
        tokio::spawn(async move {
            if let Err(e) = handle_client(sock, b).await {
                eprintln!("client error: {e:#}");
            }
        });
    }
}
```

You can talk to it with `nc`:

```
$ nc 127.0.0.1 7878
SUB orders
(nothing happens until somebody publishes)

# in another terminal:
$ printf 'PUB orders 5\nhello\n' | nc 127.0.0.1 7878
OK 1

# back in the first terminal:
MSG 1 orders 5
hello
```

## Where this falls short of NATS or Redis

The line count is the giveaway. NATS core is fifty thousand lines of Go, and there is a reason for every one of them:

- **Subject wildcards.** NATS lets you subscribe to `orders.*.eu` and match `orders.created.eu`. We do exact match only. Adding wildcards means a trie keyed on dot-separated tokens.
- **Backpressure across the cluster.** Our `broadcast::channel(1024)` will drop messages for slow subscribers. Real brokers either flow-control the publisher (TCP backpressure all the way through) or write to disk and let the subscriber catch up later.
- **Persistence and replay.** Our log is write-only. Kafka's whole identity is "you can replay from any offset". To get there you need a proper segmented log, an index, and a way for subscribers to ask "give me everything from offset N".
- **Clustering.** Single-process brokers cap out at one machine's worth of network and disk. NATS clusters with gossip, Kafka with Raft (KRaft) or ZooKeeper. Clustering is a different post; the data path inside one node is what we just built.
- **Authentication and TLS.** Two `tokio-rustls` lines and a credentials check would do it. Skipped for clarity.

## What you actually learned

If you have used a hosted broker for years without ever opening the source, the surprising thing is how much of the behaviour drops out of three primitives:

- A broadcast channel for fan-out.
- An mpsc channel per group for queue semantics.
- An append-only file for "remember everything".

Acknowledgments are just a hashmap with a timer. Consumer groups are just a shared receiver. Topics are just keys in a map. Every other feature is a refinement on top of these.

The next time you read a NATS or Kafka design doc, you will recognise most of it. Wildcards, partitions, replication, exactly-once - these are all problems layered on top of the same bones. You will also notice that "scale" almost always shows up as "we sharded the topic map across nodes" plus "we made the log smarter". The single-node version, the part you and I just wrote in one file, has been stable since the eighties when message queues first showed up on mainframes.

Four hundred lines of Rust will not survive a real production load, but it will serve a side project, run your local tests, and - more importantly - give you the mental model to read the source of the broker you actually deploy.
