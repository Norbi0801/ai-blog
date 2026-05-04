+++
title = "Rust and Redis - caching, pub/sub, and data structures"
date = 2025-12-31
description = "Using the redis crate for connection pooling, caching with TTL, pub/sub messaging, sorted sets, and streams - plus when Redis beats an in-memory HashMap."

[taxonomies]
tags = ["rust", "redis", "caching", "async"]
+++

In [the key-value store post](/blog/writing-a-key-value-store-in-rust/) we built a persistent KV store from scratch - append-only log, compaction, crash recovery, the works. About 300 lines of Rust to understand the fundamentals. But when you're building a real application, you don't want to maintain your own storage engine. You want something that already handles replication, TTL expiration, pub/sub, and a dozen specialized data structures out of the box.

That something is usually Redis. It's a single-threaded, in-memory data store that processes commands in microseconds. Discord uses it to track [presence for millions of users](https://discord.com/blog/how-discord-stores-trillions-of-messages). GitHub uses it for caching and background job queues. It's one of those tools where the getting-started story is simple (`SET key value`, `GET key`) but the depth keeps going.

The Rust ecosystem has a solid Redis client: the [`redis`](https://crates.io/crates/redis) crate (v1.0 as of writing), maintained under the [redis-rs](https://github.com/redis-rs/redis-rs) organization. It supports async operations via tokio, pipelining, pub/sub, streams, and all the data structure commands you'd expect. Combined with [`deadpool-redis`](https://crates.io/crates/deadpool-redis) for connection pooling, you get a production-ready setup in about 20 lines of configuration.

This post covers the practical side: connecting, caching patterns, pub/sub, sorted sets, streams, and the decision of when Redis actually makes sense over a simple `HashMap`.

<!-- more -->

## Dependencies

Here's what goes in your `Cargo.toml`:

```toml
[dependencies]
redis = { version = "1.0", features = ["tokio-comp", "streams"] }
deadpool-redis = "0.23"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

The `tokio-comp` feature enables async support through tokio. The `streams` feature adds Redis Streams commands (`XADD`, `XREAD`, `XREADGROUP`). If you don't need streams, you can drop that feature and save a bit of compile time.

`deadpool-redis` re-exports the `redis` crate, so you could skip the explicit `redis` dependency and use `deadpool_redis::redis` everywhere. I prefer listing both explicitly - it makes the dependency tree obvious when someone reads the manifest.

## Connecting to Redis

The simplest connection uses `Client::open` with a Redis URL, then calls `get_multiplexed_async_connection()`:

```rust
use redis::AsyncCommands;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = redis::Client::open("redis://127.0.0.1:6379/")?;
    let mut con = client.get_multiplexed_async_connection().await?;

    con.set("hello", "world").await?;
    let value: String = con.get("hello").await?;
    println!("{value}"); // "world"

    Ok(())
}
```

The `MultiplexedConnection` is the key type here. Unlike a plain TCP connection where one command blocks the socket until the response arrives, a multiplexed connection interleaves multiple in-flight commands over the same socket. It's cheaply cloneable - you can hand clones to different tokio tasks and they'll all share the same underlying TCP connection without stepping on each other.

This works because Redis is single-threaded. Commands arrive on the socket, get queued, execute sequentially, and responses come back in order. The multiplexed connection on the client side matches this model: it tags each outgoing command with an ID, dispatches the response to the right caller when it arrives.

For a single application instance handling moderate traffic, a single `MultiplexedConnection` is often enough. Redis can process [100,000+ commands per second](https://redis.io/docs/latest/operate/oss_and_on-prem/management/optimization/benchmarks/) on a single core, and the multiplexed connection won't bottleneck before Redis itself does.

## Connection pooling with deadpool-redis

When one connection isn't enough - high throughput, large payloads, or blocking commands like `BLPOP` that tie up a connection - you need a pool. If you've read [the connection pooling post](/blog/connection-pooling-why-and-how-to-reuse-database-connections/), you know the pattern: maintain a set of pre-established connections, borrow one per operation, return it when done.

[`deadpool-redis`](https://docs.rs/deadpool-redis/latest/deadpool_redis/) wraps this pattern for Redis:

```rust
use deadpool_redis::{Config, Runtime};
use redis::AsyncCommands;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cfg = Config::from_url("redis://127.0.0.1:6379/");
    let pool = cfg.create_pool(Some(Runtime::Tokio1))?;

    // Borrow a connection from the pool
    let mut con = pool.get().await?;

    con.set("counter", 0i64).await?;
    let val: i64 = con.incr("counter", 1).await?;
    println!("counter: {val}"); // 1

    // `con` is returned to the pool when dropped
    Ok(())
}
```

The pool manages connection lifecycle: it creates new connections on demand (up to a configurable maximum), health-checks them on checkout, and recycles them when returned. The default pool size is 16 connections, which is reasonable for most workloads. If you're running heavy pub/sub or blocking operations alongside normal commands, bump it up.

For production, you'll want to configure the pool from environment variables:

```rust
use deadpool_redis::{Config, Runtime};

fn create_pool() -> deadpool_redis::Pool {
    let redis_url = std::env::var("REDIS_URL")
        .unwrap_or_else(|_| "redis://127.0.0.1:6379/".to_string());

    let mut cfg = Config::from_url(redis_url);
    cfg.pool = Some(deadpool_redis::PoolConfig {
        max_size: 32,
        ..Default::default()
    });

    cfg.create_pool(Some(Runtime::Tokio1))
        .expect("failed to create redis pool")
}
```

Pass this pool into your application state (axum `State`, actix-web `Data`, or whatever your framework uses) and grab connections per request.

## Caching - SET, GET, and TTL

The most common Redis use case: cache expensive computations or database queries so you don't repeat them on every request. The pattern is called cache-aside (or lazy-loading): check the cache first, fall back to the source on miss, populate the cache for next time.

```rust
use redis::AsyncCommands;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct UserProfile {
    id: String,
    name: String,
    email: String,
    avatar_url: Option<String>,
}

async fn get_user_profile(
    con: &mut impl AsyncCommands,
    db: &Database,
    user_id: &str,
) -> Result<UserProfile, AppError> {
    let cache_key = format!("user:profile:{user_id}");

    // Try cache first
    let cached: Option<String> = con.get(&cache_key).await?;
    if let Some(json) = cached {
        return Ok(serde_json::from_str(&json)?);
    }

    // Cache miss - hit the database
    let profile = db.fetch_user_profile(user_id).await?;

    // Store in cache with 5-minute TTL
    let json = serde_json::to_string(&profile)?;
    con.set_ex(&cache_key, &json, 300).await?;

    Ok(profile)
}
```

A few things to note:

**Serialization format matters.** JSON is readable and debuggable (`redis-cli GET user:profile:abc123` shows human-readable output), but it's not the most compact. For high-throughput caches, consider [`rmp-serde`](https://crates.io/crates/rmp-serde) (MessagePack) or [`bincode`](https://crates.io/crates/bincode). MessagePack is typically 30-50% smaller than JSON for struct-heavy data, and serialization is faster because there are no field names in the output.

**TTL prevents stale data from living forever.** The `set_ex` command sets the value and expiration atomically - there's no window where the key exists without a TTL. If you forget the TTL, a cache entry lives until Redis restarts or you run out of memory. For most caches, 5-15 minutes is a reasonable default.

**Key naming convention.** Use colons as separators: `service:entity:id`. This makes `SCAN` patterns useful (`user:profile:*` matches all cached profiles) and keeps keys organized in tools like RedisInsight. Prefix with your service name if multiple services share a Redis instance.

For cache invalidation on writes, delete the key explicitly:

```rust
async fn update_user_profile(
    con: &mut impl AsyncCommands,
    db: &Database,
    user_id: &str,
    update: UpdateProfile,
) -> Result<UserProfile, AppError> {
    let profile = db.update_user_profile(user_id, update).await?;

    // Invalidate cache - next read will repopulate
    let cache_key = format!("user:profile:{user_id}");
    con.del(&cache_key).await?;

    Ok(profile)
}
```

Delete-on-write is simpler and safer than update-on-write. If the write succeeds but the cache update fails, you end up with stale data that won't expire until the TTL. With delete-on-write, a failed delete just means the old cached value lives until its TTL, and the next read will fetch fresh data.

## Pipelining - batching commands

Every Redis command is a network round trip: send command, wait for response, parse response. If you need to run 10 commands, that's 10 round trips. On a local network with 0.5ms latency per round trip, that's 5ms of waiting.

Pipelining batches multiple commands into a single send and reads all responses at once:

```rust
use redis::{pipe, AsyncCommands};

async fn cache_multiple_users(
    con: &mut impl AsyncCommands,
    users: &[UserProfile],
) -> Result<(), AppError> {
    let mut pipeline = pipe();

    for user in users {
        let key = format!("user:profile:{}", user.id);
        let json = serde_json::to_string(user)?;
        pipeline.cmd("SET").arg(&key).arg(&json).arg("EX").arg(300).ignore();
    }

    pipeline.query_async(con).await?;

    Ok(())
}
```

The `.ignore()` call tells the pipeline to skip the response for that command. We don't care about the `OK` response from each `SET` - we just want them all to execute. This keeps the return type clean.

For cases where you need atomicity - all commands succeed or none do - wrap the pipeline in `MULTI`/`EXEC`:

```rust
async fn transfer_score(
    con: &mut impl AsyncCommands,
    from: &str,
    to: &str,
    amount: i64,
) -> Result<(), AppError> {
    let (new_from, new_to): (i64, i64) = pipe()
        .atomic()
        .cmd("DECRBY").arg(from).arg(amount)
        .cmd("INCRBY").arg(to).arg(amount)
        .query_async(con)
        .await?;

    println!("{from}: {new_from}, {to}: {new_to}");
    Ok(())
}
```

The `.atomic()` call wraps everything in `MULTI`/`EXEC`. Redis executes the block without interleaving commands from other clients. This isn't a full transaction (no rollback on failure), but it guarantees that no other client sees a half-applied state.

## Pub/Sub - real-time messaging

Redis pub/sub gives you a lightweight message bus. Publishers send messages to channels, subscribers receive them in real time. No persistence, no acknowledgment, no replay - if a subscriber isn't connected when a message is published, it's gone. This makes pub/sub a poor fit for reliable job queues but a great fit for real-time notifications, cache invalidation broadcasts, and live updates.

If you've read [the event-driven architecture post](/blog/event-driven-architecture-beyond-request-response/), pub/sub is the simplest implementation of the "emit event, listeners react" pattern. The trade-off compared to in-process channels: pub/sub works across multiple application instances since Redis is the shared broker.

The subscriber needs a dedicated connection - it can't share one with regular commands because the connection enters a special subscription mode:

```rust
use redis::AsyncCommands;
use futures_util::StreamExt;

async fn subscribe_to_events(
    client: &redis::Client,
) -> Result<(), Box<dyn std::error::Error>> {
    let mut pubsub = client.get_async_pubsub().await?;
    pubsub.subscribe("cache-invalidation").await?;
    pubsub.subscribe("user-events").await?;

    let mut stream = pubsub.on_message();

    while let Some(msg) = stream.next().await {
        let channel: String = msg.get_channel_name().to_string();
        let payload: String = msg.get_payload()?;

        match channel.as_str() {
            "cache-invalidation" => {
                println!("invalidate: {payload}");
                // delete the cached key
            }
            "user-events" => {
                println!("user event: {payload}");
                // broadcast to websockets
            }
            _ => {}
        }
    }

    Ok(())
}
```

Publishing is just a regular command on any connection:

```rust
async fn publish_invalidation(
    con: &mut impl AsyncCommands,
    cache_key: &str,
) -> Result<(), AppError> {
    con.publish("cache-invalidation", cache_key).await?;
    Ok(())
}
```

A common production pattern: when instance A updates a user profile, it publishes to `cache-invalidation` with the key `user:profile:xyz`. All instances subscribe to that channel and delete the key from their local caches (if they have one) or from their Redis-backed cache. This keeps caches consistent across a fleet of application servers without each instance polling for changes.

The limitation to remember: pub/sub is fire-and-forget. If your subscriber crashes and reconnects, it misses every message published during the downtime. For durable messaging, use Redis Streams instead.

## Sorted sets - leaderboards and ranking

A sorted set maps members to floating-point scores and keeps them ordered by score. This makes it the natural data structure for leaderboards, rate limiting windows, priority queues, and anything where you need "top N" or "rank of X" queries.

Here's a leaderboard:

```rust
use redis::AsyncCommands;

async fn update_leaderboard(
    con: &mut impl AsyncCommands,
) -> Result<(), Box<dyn std::error::Error>> {
    let key = "leaderboard:weekly";

    // Add or update scores - ZADD is idempotent on the member
    con.zadd(key, "alice", 2500.0).await?;
    con.zadd(key, "bob", 3100.0).await?;
    con.zadd(key, "carol", 1800.0).await?;
    con.zadd(key, "dave", 4200.0).await?;
    con.zadd(key, "eve", 2900.0).await?;

    // Increment a score atomically
    let new_score: f64 = con.zincr(key, "alice", 500.0).await?;
    println!("alice's new score: {new_score}"); // 3000.0

    // Top 3 players (highest scores first)
    let top3: Vec<(String, f64)> = con
        .zrevrange_withscores(key, 0, 2)
        .await?;

    for (rank, (player, score)) in top3.iter().enumerate() {
        println!("#{}: {} - {:.0} pts", rank + 1, player, score);
    }
    // #1: dave - 4200 pts
    // #2: bob - 3100 pts
    // #3: alice - 3000 pts

    // Get a specific player's rank (0-indexed, highest first)
    let rank: Option<u64> = con.zrevrank(key, "bob").await?;
    println!("bob is ranked #{}", rank.unwrap() + 1); // #2

    // Set a 7-day TTL on the weekly leaderboard
    con.expire(key, 604800).await?;

    Ok(())
}
```

All of these operations are O(log N) where N is the number of members in the set. Redis implements sorted sets as a combination of a hash table (for O(1) member lookups) and a [skip list](https://en.wikipedia.org/wiki/Skip_list) (for ordered range queries). With a million members, `ZREVRANGE` to get the top 10 touches about 20 skip list nodes. That's sub-millisecond.

The `ZINCRBY` command (exposed as `zincr` in the Rust crate) is particularly useful - it atomically increments a member's score. No read-modify-write race condition, no locking. If two requests simultaneously increment Alice's score, both increments apply correctly because Redis is single-threaded.

## Streams - append-only event logs

Redis Streams are the durable counterpart to pub/sub. Messages are persisted, each gets a unique ID (typically a timestamp), and consumers can read from any point in the stream's history. If a consumer crashes and restarts, it picks up where it left off. This is closer to Kafka's model than to traditional pub/sub.

```rust
use redis::{AsyncCommands, streams::StreamReadReply};

async fn produce_events(
    con: &mut impl AsyncCommands,
) -> Result<(), Box<dyn std::error::Error>> {
    // Append events to the stream
    // "*" tells Redis to auto-generate the ID (timestamp-based)
    let id: String = con.xadd(
        "orders",
        "*",
        &[
            ("action", "created"),
            ("order_id", "ord-001"),
            ("total", "4999"),
        ],
    ).await?;
    println!("event id: {id}"); // e.g., "1720000000000-0"

    con.xadd(
        "orders",
        "*",
        &[
            ("action", "paid"),
            ("order_id", "ord-001"),
            ("payment_method", "card"),
        ],
    ).await?;

    Ok(())
}

async fn consume_events(
    con: &mut impl AsyncCommands,
) -> Result<(), Box<dyn std::error::Error>> {
    // Read all events from the beginning
    let opts = redis::streams::StreamReadOptions::default()
        .count(10);

    let reply: StreamReadReply = con
        .xread_options(&["orders"], &["0"], &opts)
        .await?;

    for stream_key in &reply.keys {
        println!("stream: {}", stream_key.key);
        for entry in &stream_key.ids {
            println!("  id: {}", entry.id);
            for (field, value) in &entry.map {
                let v: String = redis::from_redis_value(value)?;
                println!("    {field}: {v}");
            }
        }
    }

    Ok(())
}
```

For production workloads, you'll want consumer groups. They let multiple consumers split the work - each message gets delivered to exactly one consumer in the group, and Redis tracks which messages each consumer has acknowledged:

```rust
use redis::AsyncCommands;

async fn setup_consumer_group(
    con: &mut impl AsyncCommands,
) -> Result<(), Box<dyn std::error::Error>> {
    // Create a consumer group starting from the beginning of the stream
    // Ignore the error if the group already exists
    let result: Result<(), redis::RedisError> = con
        .xgroup_create_mkstream("orders", "order-processors", "0")
        .await;

    if let Err(e) = &result {
        if !e.to_string().contains("BUSYGROUP") {
            return Err(e.clone().into());
        }
    }

    Ok(())
}

async fn consume_as_group_member(
    con: &mut impl AsyncCommands,
    consumer_name: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    let opts = redis::streams::StreamReadOptions::default()
        .group("order-processors", consumer_name)
        .count(5)
        .block(5000); // block for 5 seconds if no new messages

    let reply: redis::streams::StreamReadReply = con
        .xread_options(&["orders"], &[">"], &opts)
        .await?;

    for stream_key in &reply.keys {
        for entry in &stream_key.ids {
            // Process the event...
            println!("[{consumer_name}] processing {}", entry.id);

            // Acknowledge the message
            con.xack("orders", "order-processors", &[&entry.id]).await?;
        }
    }

    Ok(())
}
```

The `">"` ID in `xread_options` means "give me messages that haven't been delivered to any consumer in this group yet." Each consumer gets different messages. If consumer A crashes before acknowledging, the message stays in its pending list and can be claimed by another consumer after a timeout using `XCLAIM` or `XAUTOCLAIM`.

Streams vs pub/sub - the trade-off is clear. Pub/sub is simpler, lower latency, and works when you want broadcast (every subscriber gets every message). Streams are for work distribution (each message processed once), durability (messages survive restarts), and replay (new consumers can read history).

## When Redis over an in-memory HashMap

If your application is a single process, a `HashMap` behind a `Mutex` or `RwLock` (or `dashmap` for concurrent access) handles caching just fine. Zero network overhead, zero serialization cost. So when does Redis earn its place?

**Multiple application instances.** The moment you scale to two or more instances behind a load balancer, each instance has its own `HashMap`. User A hits instance 1, populates the cache. User B hits instance 2, cache miss. Redis gives you a shared cache that all instances read from.

**Persistence across restarts.** A `HashMap` dies with the process. Redis supports RDB snapshots and AOF (append-only file) persistence. Configure `appendonly yes` and Redis writes every command to disk. Restart Redis, and your cache is still warm. For caches this might not matter, but for sorted sets tracking leaderboards or streams tracking event history, persistence matters a lot.

**TTL and eviction policies.** Redis handles TTL natively per key. With a `HashMap`, you'd need to implement your own expiration sweep (a background task that periodically checks timestamps) or use a crate like [`moka`](https://crates.io/crates/moka). Redis also supports eviction policies (`allkeys-lru`, `volatile-ttl`, etc.) when memory is full - configurable, well-tested behavior you'd have to build from scratch.

**Cross-service communication.** Pub/sub and streams let different services communicate through Redis without direct network connections between them. Your API server publishes events, your background worker consumes them, your websocket server broadcasts them - all through the same Redis instance.

**Specialized data structures.** A `HashMap` gives you O(1) key-value lookup. Redis gives you sorted sets (ranked access in O(log N)), HyperLogLog (cardinality estimation in 12KB), bitmaps (compact boolean arrays), and geospatial indexes. If your use case maps to one of these, Redis saves you from building it yourself.

The rule of thumb: if you're running one instance and only need key-value caching, start with `moka` or `dashmap`. Add Redis when you need shared state, pub/sub, or a data structure that a `HashMap` can't provide.

## Performance tips

**Reuse connections.** This should be obvious after [the connection pooling post](/blog/connection-pooling-why-and-how-to-reuse-database-connections/), but it bears repeating: never open a new connection per request. Use a `MultiplexedConnection` or a `deadpool-redis` pool. A Redis connection costs about 10KB of memory on the server side and a TCP socket - cheap to maintain, expensive to establish.

**Pipeline everything you can.** If your handler needs to SET three cache keys, don't send three separate commands. Batch them in a pipeline. The latency goes from 3 round trips to 1. For 10 commands at 0.5ms network latency each, that's 5ms saved - noticeable at the p99.

**Use the right serialization.** JSON is great for debugging but adds overhead. For hot-path caches where you're serializing and deserializing thousands of times per second, benchmark MessagePack or bincode against JSON. In a typical struct with 8-10 fields, MessagePack serializes 2-3x faster and produces 30-50% smaller payloads.

**Keep keys short.** Redis stores keys in memory. A key like `user-service:user-profile-cache:user-id:550e8400-e29b-41d4-a716-446655440000` is 72 bytes. `u:p:550e8400` is 10 bytes. Multiply by millions of keys and the difference is real. Find a balance between readability and brevity - `u:prof:{id}` is usually short enough.

**Set `maxmemory` and a policy.** Without a memory limit, Redis grows until the OS kills it. Set `maxmemory 2gb` (or whatever fits your instance) and `maxmemory-policy allkeys-lru`. This tells Redis to evict the least recently used keys when memory is full. For caches, this is exactly what you want - the hot data stays, the cold data gets evicted.

**Monitor with `INFO` and `SLOWLOG`.** Redis exposes metrics through the `INFO` command: memory usage, connected clients, keyspace hits/misses, commands per second. The `SLOWLOG` tracks commands that exceeded a configurable threshold (default 10ms). Check both periodically. A hit rate below 90% means your TTLs might be too short or your cache keys are wrong. Slow commands usually mean you're running O(N) operations (`KEYS *`, `SMEMBERS` on large sets) that you should replace with `SCAN` or sorted set range queries.
