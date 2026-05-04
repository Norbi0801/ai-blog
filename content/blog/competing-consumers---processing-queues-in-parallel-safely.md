+++
title = "Competing consumers - processing queues in parallel safely"
date = 2025-07-22
description = "Multiple workers, one queue, zero double-processing - atomic claims, retry with backoff, dead letter queues, and why SQLite is probably enough."

[taxonomies]
tags = ["rust", "concurrency", "architecture", "queues"]
+++

You have a table full of jobs. You want to process them fast, so you spawn multiple workers. Two of them grab the same row. One overwrites the other's result. A third crashes mid-processing and the job vanishes into the void. A fourth keeps retrying a poison message forever, burning CPU until someone notices.

This is the competing consumers problem. The pattern is old - RabbitMQ's documentation has described it for over a decade - but the failure modes catch people off guard every time. The good news: the building blocks to solve it are straightforward once you see them.

<!-- more -->

## The double-processing problem

The naive approach looks reasonable at first glance:

```rust
// DON'T DO THIS
let job = sqlx::query_as::<_, Job>("SELECT * FROM jobs WHERE status = 'pending' LIMIT 1")
    .fetch_optional(&pool).await?;

if let Some(job) = job {
    sqlx::query("UPDATE jobs SET status = 'processing' WHERE id = ?")
        .bind(&job.id).execute(&pool).await?;

    process(job).await?;
}
```

Two workers execute the SELECT at the same time. Both see the same pending job. Both update it to `processing`. Both process it. Your customer gets charged twice, your email gets sent twice, your webhook fires twice. The SELECT and the UPDATE are two separate operations with a gap between them where other transactions can interleave.

The fix is an **atomic claim** - a single statement that finds a pending job AND marks it as claimed in one shot, so no other worker can see it in between.

### Atomic claim with SQLite

If you've read [Why SQLite with WAL Mode Is Good Enough for Most Web Apps](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/), you know SQLite supports concurrent readers with a single writer. That single-writer constraint actually works in our favor for queues - it serializes all claims through one write lock, making double-processing impossible:

```sql
BEGIN IMMEDIATE;
UPDATE jobs SET status = 'processing', claimed_by = ?, claimed_at = CURRENT_TIMESTAMP
  WHERE id = (
    SELECT id FROM jobs
    WHERE status = 'pending'
    ORDER BY created_at ASC
    LIMIT 1
  )
  RETURNING *;
COMMIT;
```

`BEGIN IMMEDIATE` is critical here. A plain `BEGIN` in SQLite starts as a read transaction and only tries to upgrade to a write lock when it hits the UPDATE. If the database is already locked by another writer at that moment, it fails with `SQLITE_BUSY` - and here's the nasty part - **regardless of your `busy_timeout` setting**. `BEGIN IMMEDIATE` acquires the write lock upfront, so `busy_timeout` actually works and concurrent workers queue up politely instead of failing.

The subselect + UPDATE + RETURNING runs as a single atomic statement. By the time another worker acquires the write lock, the job's status is already `processing` and the subselect skips it.

In Rust with sqlx:

```rust
use sqlx::SqlitePool;

#[derive(Debug, sqlx::FromRow)]
struct Job {
    id: String,
    payload: String,
    status: String,
    claimed_by: Option<String>,
    retry_count: i32,
    max_retries: i32,
    created_at: String,
    visible_at: String,
}

async fn claim_job(pool: &SqlitePool, worker_id: &str) -> Result<Option<Job>, sqlx::Error> {
    let job = sqlx::query_as::<_, Job>(
        "UPDATE jobs SET status = 'processing', claimed_by = ?, claimed_at = CURRENT_TIMESTAMP
         WHERE id = (
           SELECT id FROM jobs
           WHERE status = 'pending' AND visible_at <= CURRENT_TIMESTAMP
           ORDER BY created_at ASC
           LIMIT 1
         )
         RETURNING *"
    )
    .bind(worker_id)
    .fetch_optional(pool)
    .await?;

    Ok(job)
}
```

Notice the `visible_at <= CURRENT_TIMESTAMP` condition. That's for delayed retries - more on that in a minute.

### How Postgres does it: SKIP LOCKED

If you're on Postgres, the gold standard is `FOR UPDATE SKIP LOCKED` (available since Postgres 9.5):

```sql
UPDATE jobs SET status = 'processing', claimed_by = $1
WHERE id = (
  SELECT id FROM jobs
  WHERE status = 'pending' AND visible_at <= now()
  ORDER BY created_at
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING *;
```

`SKIP LOCKED` tells Postgres to skip rows that are already locked by other transactions instead of blocking or failing. Workers never wait for each other and never get the same row. This is true row-level locking - multiple workers can claim different jobs simultaneously, while SQLite serializes them through the single write lock.

For most workloads, the difference doesn't matter. SQLite's write lock acquisition + subselect + update takes microseconds. You'd need hundreds of workers claiming thousands of jobs per second before the serialization becomes a bottleneck. But if you're already on Postgres, `SKIP LOCKED` is the better tool.

Solid Queue (from 37signals, powering HEY and Basecamp), PG Boss, and River all use this pattern in production.

## The queue table

A production queue table needs more than `id`, `payload`, and `status`:

```sql
CREATE TABLE jobs (
    id TEXT PRIMARY KEY,
    queue TEXT NOT NULL DEFAULT 'default',
    payload TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    claimed_by TEXT,
    claimed_at TEXT,
    completed_at TEXT,
    failed_at TEXT,
    error_message TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    max_retries INTEGER NOT NULL DEFAULT 3,
    visible_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_jobs_claimable ON jobs(status, visible_at, created_at)
  WHERE status = 'pending';
```

The partial index on `status = 'pending'` is important. As your table grows to millions of completed jobs, you don't want the claim query scanning through all of them. The index only contains pending jobs, staying small and fast.

The `queue` column lets you run multiple logical queues in one table - `emails`, `webhooks`, `reports` - each with their own workers and concurrency limits. Add it to the claim query's WHERE clause:

```rust
async fn claim_job(pool: &SqlitePool, worker_id: &str, queue: &str) -> Result<Option<Job>, sqlx::Error> {
    sqlx::query_as::<_, Job>(
        "UPDATE jobs SET status = 'processing', claimed_by = ?, claimed_at = CURRENT_TIMESTAMP
         WHERE id = (
           SELECT id FROM jobs
           WHERE status = 'pending' AND queue = ? AND visible_at <= CURRENT_TIMESTAMP
           ORDER BY created_at ASC
           LIMIT 1
         )
         RETURNING *"
    )
    .bind(worker_id)
    .bind(queue)
    .fetch_optional(pool)
    .await
}
```

## Retry with exponential backoff

Jobs fail. Network timeouts, downstream service outages, temporary disk full conditions. The first instinct is to retry immediately. The second instinct is to retry on a fixed interval. Both are wrong for production.

Immediate retry hammers a service that's already struggling. Fixed interval retry (say, every 30 seconds) creates a steady stream of requests that can prevent recovery. Exponential backoff with jitter spreads retries over increasing time windows, giving the failing system breathing room:

```rust
use rand::Rng;
use std::time::Duration;

fn retry_delay(retry_count: i32, base: Duration) -> Duration {
    let exponential = base.as_millis() as u64 * 2u64.pow(retry_count as u32);
    let max_delay = 3600_000u64; // cap at 1 hour
    let capped = exponential.min(max_delay);

    // Add jitter: random value between 0 and capped delay
    let jitter = rand::rng().random_range(0..=capped);
    Duration::from_millis(jitter)
}
```

The jitter is the part people skip, and it matters more than the exponential part. Without jitter, if 50 jobs all fail at the same time (because a downstream service went down), they all retry at exactly the same time on each subsequent attempt. The retries form a thundering herd that hits the recovering service with a synchronized burst. Full jitter randomizes across the entire window, spreading load evenly.

AWS published a [good analysis](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) comparing no jitter, equal jitter (random between half and full delay), and full jitter (random between zero and full delay). Full jitter consistently produces the lowest total work across clients.

When a job fails, don't retry it inline. Instead, update the row to reschedule it:

```rust
async fn fail_job(pool: &SqlitePool, job: &Job, error: &str) -> Result<(), sqlx::Error> {
    let new_retry_count = job.retry_count + 1;

    if new_retry_count >= job.max_retries {
        // Move to dead letter state
        sqlx::query(
            "UPDATE jobs SET status = 'dead', error_message = ?, failed_at = CURRENT_TIMESTAMP
             WHERE id = ?"
        )
        .bind(error)
        .bind(&job.id)
        .execute(pool)
        .await?;
    } else {
        // Schedule retry with backoff
        let delay = retry_delay(new_retry_count, Duration::from_secs(5));
        let visible_at = chrono::Utc::now() + chrono::Duration::from_std(delay).unwrap();

        sqlx::query(
            "UPDATE jobs SET status = 'pending', claimed_by = NULL, retry_count = ?,
             error_message = ?, visible_at = ?
             WHERE id = ?"
        )
        .bind(new_retry_count)
        .bind(error)
        .bind(visible_at.to_rfc3339())
        .bind(&job.id)
        .execute(pool)
        .await?;
    }

    Ok(())
}
```

Setting `status = 'pending'` and clearing `claimed_by` puts the job back in the pool. The `visible_at` timestamp ensures no worker picks it up until the backoff period has elapsed. The claim query's `visible_at <= CURRENT_TIMESTAMP` condition handles the rest.

## Dead letter queue

After `max_retries` failures, a job is either permanently broken (malformed payload, deleted resource, logic bug) or stuck on a transient issue that's lasting longer than your retry budget. Either way, stop trying. Move it to a dead letter state and alert someone.

The simplest approach is a status column - `dead` - right in the same table. No separate table, no separate infrastructure. Query it periodically:

```rust
async fn count_dead_jobs(pool: &SqlitePool) -> Result<i64, sqlx::Error> {
    let row = sqlx::query_scalar::<_, i64>("SELECT COUNT(*) FROM jobs WHERE status = 'dead'")
        .fetch_one(pool)
        .await?;
    Ok(row)
}
```

Wire that count to your monitoring. If you set up Prometheus metrics as described in [Monitoring Rust Applications](/blog/monitoring-rust-applications/), expose it as a gauge:

```rust
use metrics::gauge;

// In your monitoring loop
let dead_count = count_dead_jobs(&pool).await?;
gauge!("jobs_dead_total").set(dead_count as f64);
```

A growing dead letter count is a signal. Maybe a downstream API changed its schema. Maybe a bug in your processing code rejects valid payloads. The dead jobs carry their `error_message` and full `payload` - you can inspect them, fix the bug, and replay:

```rust
async fn replay_dead_jobs(pool: &SqlitePool) -> Result<u64, sqlx::Error> {
    let result = sqlx::query(
        "UPDATE jobs SET status = 'pending', retry_count = 0, claimed_by = NULL,
         visible_at = CURRENT_TIMESTAMP, error_message = NULL
         WHERE status = 'dead'"
    )
    .execute(pool)
    .await?;

    Ok(result.rows_affected())
}
```

One query to resurrect everything. Fix the root cause first, obviously.

## Semaphore + tokio::spawn: the worker pool

Now the interesting part - running multiple workers safely with controlled concurrency. The core idea: a `tokio::Semaphore` limits how many jobs process simultaneously, and each worker is a `tokio::spawn`ed task that acquires a permit before claiming work.

If you've used `tokio::sync::mpsc` for the webhook processor in [Building a Webhook Receiver in Rust](/blog/building-a-webhook-receiver-in-rust/), this is the pull-based counterpart. Instead of pushing events into a channel, workers pull jobs from the database. The semaphore replaces the channel's bounded capacity as the backpressure mechanism.

Here's how tokio's `Semaphore` works under the hood. The [implementation](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/batch_semaphore.rs) uses an atomic counter for available permits (the fast path - no lock needed if permits are available) and a mutex-protected intrusive doubly-linked list of waiters for the slow path. When a task calls `acquire()` and permits are available, it's just an atomic decrement - no syscall, no context switch. Only when all permits are held does a task actually go to sleep on the wait list, and it's woken in FIFO order so no task starves.

```rust
use sqlx::SqlitePool;
use std::sync::Arc;
use tokio::sync::Semaphore;
use tokio_util::sync::CancellationToken;

struct WorkerPool {
    pool: SqlitePool,
    semaphore: Arc<Semaphore>,
    cancel: CancellationToken,
    worker_id: String,
    queue: String,
}

impl WorkerPool {
    fn new(
        pool: SqlitePool,
        concurrency: usize,
        worker_id: String,
        queue: String,
    ) -> Self {
        Self {
            pool,
            semaphore: Arc::new(Semaphore::new(concurrency)),
            cancel: CancellationToken::new(),
            worker_id,
            queue,
        }
    }

    async fn run(&self) {
        tracing::info!(
            worker_id = %self.worker_id,
            queue = %self.queue,
            concurrency = self.semaphore.available_permits(),
            "worker pool started"
        );

        loop {
            tokio::select! {
                _ = self.cancel.cancelled() => {
                    tracing::info!("shutdown signal received, draining...");
                    // Wait for all in-flight jobs to finish
                    let _ = self.semaphore
                        .acquire_many(self.semaphore.available_permits() as u32)
                        .await;
                    break;
                }
                permit = self.semaphore.clone().acquire_owned() => {
                    let permit = permit.expect("semaphore closed");

                    match claim_job(&self.pool, &self.worker_id, &self.queue).await {
                        Ok(Some(job)) => {
                            let pool = self.pool.clone();
                            tokio::spawn(async move {
                                let _permit = permit; // held until task completes

                                let job_id = job.id.clone();
                                tracing::info!(job_id = %job_id, "processing job");

                                match process_job(&job).await {
                                    Ok(()) => {
                                        complete_job(&pool, &job_id).await.ok();
                                        tracing::info!(job_id = %job_id, "job completed");
                                    }
                                    Err(e) => {
                                        let error = format!("{e:#}");
                                        fail_job(&pool, &job, &error).await.ok();
                                        tracing::error!(
                                            job_id = %job_id,
                                            error = %error,
                                            "job failed"
                                        );
                                    }
                                }
                            });
                        }
                        Ok(None) => {
                            // No pending jobs - release permit and back off
                            drop(permit);
                            tokio::time::sleep(Duration::from_secs(1)).await;
                        }
                        Err(e) => {
                            drop(permit);
                            tracing::error!(error = %e, "failed to claim job");
                            tokio::time::sleep(Duration::from_secs(5)).await;
                        }
                    }
                }
            }
        }
    }

    fn shutdown(&self) {
        self.cancel.cancel();
    }
}

async fn complete_job(pool: &SqlitePool, job_id: &str) -> Result<(), sqlx::Error> {
    sqlx::query(
        "UPDATE jobs SET status = 'completed', completed_at = CURRENT_TIMESTAMP WHERE id = ?"
    )
    .bind(job_id)
    .execute(pool)
    .await?;
    Ok(())
}

async fn process_job(job: &Job) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    // Your actual processing logic here
    let payload: serde_json::Value = serde_json::from_str(&job.payload)?;
    tracing::info!(payload = %payload, "processing payload");
    Ok(())
}
```

A few important details in this design:

**`acquire_owned` instead of `acquire`.** The owned permit can move into the spawned task. A regular `acquire` returns a permit tied to the semaphore's lifetime, which doesn't work across `tokio::spawn` boundaries since the spawned future needs to be `'static`.

**The permit is held until the task finishes.** `let _permit = permit;` inside the spawned task means the permit drops when the task completes (or panics). This is the concurrency limit - if you set the semaphore to 10 permits, at most 10 jobs process simultaneously. The 11th `acquire_owned` call waits until one finishes.

**Backoff on empty queue.** When `claim_job` returns `None`, the worker sleeps for a second instead of spinning. Without this, an idle worker hammers the database with thousands of pointless claim queries per second. You could also use `NOTIFY`/`LISTEN` on Postgres or a filesystem watcher on the SQLite WAL to wake workers on new inserts, but polling with a 1-second sleep is simple and usually sufficient.

**Graceful shutdown.** The `CancellationToken` signals the loop to stop claiming new jobs. The loop then waits for all permits to be returned - meaning all in-flight tasks have finished - before exiting. No jobs abandoned mid-processing.

Wire it up with signal handling:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::fmt::init();

    let pool = sqlx::SqlitePool::connect("sqlite:jobs.db").await?;

    // Run migrations (in production, use sqlx::migrate!)
    sqlx::query(
        "CREATE TABLE IF NOT EXISTS jobs (
            id TEXT PRIMARY KEY,
            queue TEXT NOT NULL DEFAULT 'default',
            payload TEXT NOT NULL,
            status TEXT NOT NULL DEFAULT 'pending',
            claimed_by TEXT,
            claimed_at TEXT,
            completed_at TEXT,
            failed_at TEXT,
            error_message TEXT,
            retry_count INTEGER NOT NULL DEFAULT 0,
            max_retries INTEGER NOT NULL DEFAULT 3,
            visible_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
            created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
        )"
    )
    .execute(&pool)
    .await?;

    let worker = WorkerPool::new(
        pool.clone(),
        4, // process up to 4 jobs concurrently
        "worker-1".to_string(),
        "default".to_string(),
    );

    // Spawn the worker pool
    let worker_handle = {
        let worker = &worker;
        tokio::spawn(async move { worker.run().await })
    };

    // Wait for Ctrl+C
    tokio::signal::ctrl_c().await?;
    tracing::info!("shutting down...");
    worker.shutdown();
    worker_handle.await?;

    Ok(())
}
```

## Visibility timeout: handling crashed workers

There's a gap in the design so far. If a worker claims a job, then crashes (OOM kill, power loss, segfault), the job stays in `processing` status forever. Nobody retries it.

The fix is a visibility timeout - a background task that reclaims jobs stuck in `processing` for too long:

```rust
async fn reap_stale_jobs(pool: &SqlitePool, timeout_seconds: i64) -> Result<u64, sqlx::Error> {
    let result = sqlx::query(
        "UPDATE jobs SET status = 'pending', claimed_by = NULL,
         retry_count = retry_count + 1,
         visible_at = CURRENT_TIMESTAMP
         WHERE status = 'processing'
         AND claimed_at < datetime('now', ? || ' seconds')"
    )
    .bind(format!("-{}", timeout_seconds))
    .execute(pool)
    .await?;

    if result.rows_affected() > 0 {
        tracing::warn!(
            count = result.rows_affected(),
            "reaped stale jobs"
        );
    }

    Ok(result.rows_affected())
}
```

Run this on a timer:

```rust
// In main, alongside the worker pool
let reaper_pool = pool.clone();
let reaper_cancel = cancel_token.clone();
tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(30));
    loop {
        tokio::select! {
            _ = reaper_cancel.cancelled() => break,
            _ = interval.tick() => {
                reap_stale_jobs(&reaper_pool, 300).await.ok(); // 5 min timeout
            }
        }
    }
});
```

This is the same concept as SQS's visibility timeout. Amazon got it right: don't track whether workers are alive - track whether they've been holding a message too long. Simpler and more reliable than heartbeat protocols.

## Backend comparison: RabbitMQ vs Redis vs SQLite

So far we've built everything on SQLite. That's intentional - it's the simplest option that works. But here's how dedicated message brokers compare when you need more.

### RabbitMQ

RabbitMQ was built for this exact pattern. Competing consumers get round-robin delivery, and `basic.qos` (prefetch count) controls how many unacknowledged messages each consumer holds. Set `prefetch_count = 1` for fair distribution, `100-300` for throughput.

The killer feature is [Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx) (DLX). Configure `x-dead-letter-exchange` on a queue and rejected messages (or messages that exceed TTL or queue length limits) automatically route to another exchange. No application code needed - it's infrastructure-level DLQ.

With [lapin](https://crates.io/crates/lapin) (v3.0) in Rust:

```rust
use lapin::{
    options::*, types::FieldTable, BasicProperties,
    Channel, Connection, ConnectionProperties,
};

async fn setup_consumer(channel: &Channel) -> Result<(), lapin::Error> {
    // Set prefetch - only deliver 10 unacked messages per consumer
    channel.basic_qos(10, BasicQosOptions::default()).await?;

    // Declare queue with dead letter exchange
    let mut args = FieldTable::default();
    args.insert(
        "x-dead-letter-exchange".into(),
        lapin::types::AMQPValue::LongString("dlx".into()),
    );

    channel.queue_declare("jobs", QueueDeclareOptions::default(), args).await?;

    // Start consuming
    let consumer = channel
        .basic_consume("jobs", "worker-1", BasicConsumeOptions::default(), FieldTable::default())
        .await?;

    // Process messages...
    Ok(())
}
```

**When to use RabbitMQ:** Multiple services need to consume from the same queue. You need routing (topic exchanges, headers-based routing). You want built-in DLX, TTL, and priority queues without writing them yourself. Your queue throughput exceeds what SQLite's single writer can handle (roughly 10,000+ claims/sec).

**The cost:** Another service to deploy, monitor, and keep running. RabbitMQ clusters need careful tuning. Erlang's memory management can surprise you. The [CloudAMQP](https://www.cloudamqp.com/) managed offering starts at $0/month for the free tier but gets expensive fast.

### Redis

Redis gives you two queue primitives. The classic approach uses lists with [BLMOVE](https://redis.io/docs/latest/commands/blmove/) (replacing the deprecated BRPOPLPUSH):

```
BLMOVE work_queue processing_list RIGHT LEFT 0
```

This atomically pops from the work queue and pushes to a processing list. If the consumer crashes, the item sits in `processing_list` until a monitor process moves it back.

The modern approach is [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/) with consumer groups:

```rust
use redis::AsyncCommands;

async fn consume_stream(
    conn: &mut redis::aio::MultiplexedConnection,
) -> Result<(), redis::RedisError> {
    // Read new messages for this consumer
    let result: redis::Value = redis::cmd("XREADGROUP")
        .arg("GROUP").arg("workers")
        .arg("CONSUMER").arg("worker-1")
        .arg("COUNT").arg(1)
        .arg("BLOCK").arg(5000) // block 5 seconds
        .arg("STREAMS").arg("jobs").arg(">")
        .query_async(conn)
        .await?;

    // After processing, acknowledge
    let _: () = redis::cmd("XACK")
        .arg("jobs")
        .arg("workers")
        .arg(&message_id)
        .query_async(conn)
        .await?;

    Ok(())
}
```

Streams track delivery count automatically via `XPENDING`. When `times_delivered` exceeds your threshold, move the message to a dead letter stream with `XADD` and `XACK` the original. Redis doesn't do this automatically - you build the DLQ logic yourself.

**When to use Redis:** You're already running Redis for caching or sessions. You need sub-millisecond latency on queue operations. Your throughput needs are high (Redis handles ~1M messages/sec in-memory). Consumer groups give you competing consumers without application-level locking.

**The cost:** Redis is in-memory. Your queue is bounded by RAM. Redis 8.0's persistence improvements help, but if you lose the Redis instance before an RDB/AOF flush, you lose messages. For queues where losing a few messages on crash is acceptable (telemetry, analytics), this is fine. For payment processing, it's not.

### SQLite

You've already seen the implementation. Single-writer serialization through `BEGIN IMMEDIATE`, no external dependencies, the queue lives right next to your application.

**Throughput numbers:** A well-configured SQLite instance on a VPS with NVMe storage sustains roughly 10,000-15,000 job claims per second. The [plainjob](https://github.com/justplainstuff/plainjob) benchmark (JavaScript, SQLite-backed queue) hit 15,000 jobs/sec. Rust with direct sqlx should match or exceed that.

**When SQLite is enough:** Single application server. Under 5,000 jobs/minute. No need for multiple services consuming the same queue. You want zero operational overhead.

**When it's not:** Multiple services on different machines need to consume jobs. Write throughput needs exceed the single-writer limit. You need pub/sub semantics (one message consumed by multiple subscribers).

### The comparison at a glance

| | SQLite | Redis | RabbitMQ |
|---|---|---|---|
| Deploy complexity | Zero (embedded) | Low-medium | Medium-high |
| Throughput ceiling | ~15K claims/sec | ~1M msg/sec | ~50K msg/sec |
| Persistence | Durable (WAL + fsync) | Configurable (AOF/RDB) | Durable (disk-backed) |
| Built-in DLQ | No (application logic) | No (application logic) | Yes (DLX) |
| Competing consumers | Via write lock serialization | Via consumer groups | Via round-robin + prefetch |
| Multi-service | No | Yes | Yes |
| Retry/backoff | Application logic | Application logic | Via DLX + TTL |
| Cost (managed) | $0 | $15-100/mo | $20-200/mo |

## When SQLite is enough (almost always at the start)

This is the same argument I made in the [SQLite WAL](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/) post, applied to queues. Most applications start as a single binary on a single server. You don't need RabbitMQ to send confirmation emails. You don't need Redis Streams to generate PDF reports. A jobs table in your existing SQLite database, claimed by a worker pool running in the same process, handles it.

The migration path is clean if you used the [repository pattern](/blog/the-repository-pattern-abstracting-data-access-in-rust/). Your `JobQueue` trait stays the same. Today the implementation is `SqliteJobQueue`. When you scale to multiple servers, you write `RedisJobQueue` or `RabbitMqJobQueue` behind the same interface. Your worker pool code doesn't change at all.

The honest progression looks like this:

1. **SQLite in-process** - one binary, one server, zero infrastructure. Handles most startups and side projects forever.
2. **Postgres SKIP LOCKED** - you're already on Postgres for other reasons, and `SKIP LOCKED` gives you a proper queue without adding infrastructure. Used in production at 37signals (Solid Queue).
3. **Redis Streams** - you need sub-millisecond latency or are already running Redis. Good for high-throughput, lower-durability workloads.
4. **RabbitMQ/NATS/Kafka** - multiple services, complex routing, fan-out patterns, event sourcing at scale. This is real infrastructure with real operational cost.

Most projects never need to leave step 1. Some reach step 2. Very few actually need steps 3 or 4, and by the time you do, you'll know it from concrete symptoms (write lock contention in metrics, request latency spikes from queue polling, multiple services needing to consume the same stream) rather than speculative architecture.

Start with the simplest thing. Measure. Move up when the numbers tell you to.
