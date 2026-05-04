+++
title = "Why SQLite with WAL Mode Is Good Enough for Most Web Apps"
date = 2025-01-04
description = "SQLite in WAL mode handles thousands of requests per second on a single VPS - here's the configuration, benchmarks, and the exact point where you actually need Postgres."

[taxonomies]
tags = ["sqlite", "databases", "architecture", "performance"]
+++

Your web app probably doesn't need a database server. Not Postgres, not MySQL, not a managed database costing you $15-50/month. A single SQLite file on the same machine as your application, configured with Write-Ahead Logging, will handle more traffic than you think.

This isn't a contrarian take for the sake of it. SQLite serves over a trillion queries a day across all the devices running it. The question isn't whether SQLite is production-ready - it's whether your specific workload actually needs the complexity of a client-server database.

<!-- more -->

## How WAL Mode Works

By default, SQLite uses rollback journal mode. When you write to the database, SQLite copies the original pages into a separate journal file, then modifies the database file in place. If the process crashes mid-write, it reads the journal and rolls back. This works, but it has a brutal limitation: readers and writers block each other. A single write transaction locks the entire database for everyone.

WAL mode flips this around. Instead of modifying the database file directly, writes go into a separate write-ahead log file (the `-wal` file next to your database). Readers continue reading from the database file and the WAL, constructing their view of the data from both. A reader that starts a transaction "remembers" where the WAL ended at that moment - its end mark. Any writes appended to the WAL after that point are invisible to that reader.

The result: multiple readers can operate concurrently with a single writer. Readers never block writers. Writers never block readers. The only serialization point is that two write transactions can't run at the same time - one writer at a time, always.

Periodically, a checkpoint operation transfers pages from the WAL back into the main database file. SQLite does this automatically when the WAL reaches 1000 pages (about 4MB with the default page size), but you can trigger it manually or adjust the threshold.

Here's what this looks like at the file system level:

```
myapp.db       # main database file
myapp.db-wal   # write-ahead log (appended to by writers)
myapp.db-shm   # shared memory file (coordinates readers)
```

The `-shm` file is a shared memory index that tracks which WAL frames have been committed and which are still in progress. Readers use it to figure out their end mark without needing to scan the entire WAL.

## The Production PRAGMA Checklist

Enabling WAL mode is one `PRAGMA` statement, but there are several others you should set on every connection for production use. Here's the full set:

```sql
PRAGMA journal_mode = WAL;          -- enable write-ahead logging
PRAGMA synchronous = NORMAL;        -- fsync only on checkpoint, not every commit
PRAGMA busy_timeout = 5000;         -- wait 5s for locks instead of failing immediately
PRAGMA foreign_keys = ON;           -- enforce foreign key constraints
PRAGMA cache_size = -64000;         -- 64MB page cache (negative = KiB)
PRAGMA temp_store = MEMORY;         -- keep temp tables in RAM
PRAGMA mmap_size = 268435456;       -- memory-map up to 256MB of the database
```

Let me walk through the ones that matter most.

### journal_mode = WAL

This is the big one. Unlike most PRAGMAs, `journal_mode` is persistent - you set it once and it sticks across connections and restarts. Every other PRAGMA on this list needs to be set on each new connection.

```rust
// With sqlx in Rust, set PRAGMAs via the connection options
use sqlx::sqlite::{SqliteConnectOptions, SqliteJournalMode, SqliteSynchronous};

let options = SqliteConnectOptions::new()
    .filename("myapp.db")
    .journal_mode(SqliteJournalMode::Wal)
    .synchronous(SqliteSynchronous::Normal)
    .busy_timeout(std::time::Duration::from_secs(5))
    .pragma("foreign_keys", "ON")
    .pragma("cache_size", "-64000")
    .pragma("temp_store", "MEMORY")
    .pragma("mmap_size", "268435456");
```

### synchronous = NORMAL

The default `synchronous = FULL` forces an `fsync()` syscall on every single commit. That's the most durable option - your data survives a power outage mid-write. But in WAL mode, `NORMAL` only syncs during checkpoints. Committed transactions can still be lost if the operating system crashes or you lose power between checkpoints, but not on application crashes. For a web app that isn't handling financial transactions, this tradeoff is worth it.

### busy_timeout = 5000

Without this, any connection that tries to write while another write is in progress gets an immediate `SQLITE_BUSY` error. With `busy_timeout`, the connection will retry internally for the specified number of milliseconds before giving up. Five seconds is a reasonable default. If your writes regularly take longer than that, you have a different problem.

One important detail: `busy_timeout` is per-connection, not per-database. You need to set it every time you open a connection. If you're using a connection pool, make sure the pool configuration runs these PRAGMAs on connection creation.

### foreign_keys = ON

SQLite disables foreign key enforcement by default for backwards compatibility. This is a footgun. Always turn it on unless you have a very specific reason not to.

### cache_size and mmap_size

The page cache (`cache_size`) keeps frequently accessed database pages in memory. The negative value means KiB, so `-64000` is roughly 64MB. Memory-mapping (`mmap_size`) lets the OS manage page access through the virtual memory system, reducing `read()` syscall overhead. Both are tuning knobs - the right values depend on your database size and access patterns.

## The Benchmarks

Theory is nice. Numbers are better.

[Shivek Khurana's production benchmark](https://shivekkhurana.com/blog/sqlite-in-production/) tested SQLite under realistic conditions on an 8-core Intel i9 with SSD storage. With WAL mode enabled and proper PRAGMAs:

**Write throughput:**
- 2 concurrent workers: ~496 writes/sec
- 8 concurrent workers: ~4,000 writes/sec
- 64 concurrent workers: **16,461 writes/sec** (peak)

**Mixed workload (80% reads, 20% writes):**
- Peak throughput: ~9,400 ops/sec at 14 concurrent workers
- Read P99 latency: consistently under 6ms
- Write P99 latency: under 10ms until exceeding core count

[phiresky's benchmarks](https://phiresky.github.io/blog/2020/sqlite-performance-tuning/) hit **100,000 SELECTs per second** with proper tuning on a scaled database with concurrent readers.

Those numbers are from beefy developer machines. What about a $10/month VPS? A typical 2-vCPU, 2GB RAM cloud instance with NVMe storage can sustain roughly 2,000-5,000 mixed operations per second with WAL mode. That's enough to handle hundreds of concurrent users making API requests.

To put this in perspective: if your average API endpoint does one read and one write, 2,000 ops/sec translates to roughly 1,000 requests per second. If each user generates one request every 5 seconds, that's 5,000 concurrent users on a single $10 VPS. Most web apps never see that kind of traffic.

If you want to benchmark your own setup, I covered the methodology and tools in depth in [Load Testing Your Rust API](/blog/load-testing-your-rust-api---tools-and-methodology/) - `hey` for quick baselines, `wrk2` for accurate latency percentiles.

## The Connection Pool Question

With client-server databases like Postgres, you set up a connection pool because each connection is an OS-level process or thread with real overhead. SQLite connections are just in-process file handles - much cheaper. But you still need a pooling strategy because of the single-writer constraint.

The pattern that works well:

```rust
use sqlx::sqlite::SqlitePoolOptions;

// Multiple reader connections, but writes are serialized anyway
let pool = SqlitePoolOptions::new()
    .max_connections(10) // readers can be concurrent
    .after_connect(|conn, _meta| {
        Box::pin(async move {
            sqlx::query("PRAGMA foreign_keys = ON")
                .execute(&mut *conn).await?;
            sqlx::query("PRAGMA busy_timeout = 5000")
                .execute(&mut *conn).await?;
            sqlx::query("PRAGMA cache_size = -64000")
                .execute(&mut *conn).await?;
            sqlx::query("PRAGMA temp_store = MEMORY")
                .execute(&mut *conn).await?;
            sqlx::query("PRAGMA mmap_size = 268435456")
                .execute(&mut *conn).await?;
            Ok(())
        })
    })
    .connect("sqlite:myapp.db").await?;
```

Some frameworks take a different approach: one dedicated write connection and a pool of read connections. This prevents write contention entirely at the connection pool level, rather than relying on `busy_timeout` retries. Both approaches work - the dedicated writer is slightly more predictable under load.

If you're using the repository pattern (which I covered in [The Repository Pattern - Abstracting Data Access in Rust](/blog/the-repository-pattern-abstracting-data-access-in-rust)), your SQLite implementation slots in behind the same trait as Postgres or an in-memory backend. The pooling details stay hidden from your business logic.

## Where SQLite Breaks Down

SQLite with WAL mode isn't a universal solution. There are clear situations where you need something else.

### Multiple Application Servers

SQLite is an embedded database. It lives on the same filesystem as your application. The moment you scale horizontally - two app servers, three containers, a Kubernetes deployment with replicas - you can't share a single SQLite file across them. Network filesystems like NFS technically work but the performance is abysmal and you'll hit locking issues.

If you need horizontal scaling, you need a client-server database. That's the line.

### Write-Heavy Workloads

The single-writer limitation is real. If your application does more writes than reads - high-frequency event logging, real-time analytics ingestion, IoT sensor data at scale - you'll hit the write serialization bottleneck. One writer at a time means write throughput has a hard ceiling, no matter how much hardware you throw at it.

For context, the experimental `BEGIN CONCURRENT` feature (available in SQLite's `begin-concurrent-wal2` branch but not in mainline releases) allows optimistic concurrent writes that only conflict if they touch the same B-tree pages. Turso's libSQL fork already implements this. But vanilla SQLite today is single-writer.

### Large Databases on Slow Storage

SQLite works best when the database fits in the OS page cache (or at least the hot pages do). A 50GB database on a VPS with 2GB RAM will spend a lot of time hitting disk. Managed Postgres with dedicated memory and I/O optimization handles large datasets more gracefully.

### Complex Queries Across Large Datasets

SQLite doesn't have a query planner as sophisticated as Postgres. It doesn't support parallel query execution. It doesn't have hash joins (it uses nested loops). For analytical queries scanning millions of rows with multiple joins, Postgres will be significantly faster.

## Litestream: Solving the Backup Problem

The biggest operational concern with SQLite in production is backups. You can't just `cp myapp.db myapp.db.bak` while the application is running - you might copy a half-written state. SQLite has a built-in `.backup` command, but it requires stopping writes.

[Litestream](https://litestream.io/) solves this elegantly. It runs as a sidecar process that continuously replicates your WAL changes to S3, Google Cloud Storage, Azure Blob Storage, or any S3-compatible store like Cloudflare R2. It hooks into SQLite's WAL mechanism directly - as pages are written to the WAL, Litestream streams them to your backup target.

```yaml
# litestream.yml
dbs:
  - path: /data/myapp.db
    replicas:
      - type: s3
        bucket: my-backup-bucket
        path: myapp
        region: us-east-1
```

```bash
# Run your app through Litestream
litestream replicate -config litestream.yml
```

Recovery is just as simple:

```bash
# Restore from S3
litestream restore -config litestream.yml /data/myapp.db
```

Litestream 0.5 (released in late 2025) cleaned up the architecture significantly after creator Ben Johnson's detour into the FUSE-based LiteFS approach. The single-binary sidecar model won out in practice - it's simpler to deploy and reason about.

The cost of S3 storage for a typical web app database is negligible. A 1GB database with daily WAL changes might cost $0.02/month on S3. Compare that to managed database backups that are either limited (7-day retention on free tiers) or cost extra.

## Turso: When You Need Distribution

If your workload genuinely needs multiple servers reading the database - maybe you're deploying to edge locations for latency, or you need read replicas - [Turso](https://turso.tech/) bridges the gap. It's built on libSQL (a fork of SQLite) and replicates your database to edge locations globally.

The pricing tells the story of SQLite's cost advantage:

| | **SQLite + Litestream** | **Turso (Developer)** | **DigitalOcean Postgres** | **AWS RDS Postgres** | **Supabase Pro** |
|---|---|---|---|---|---|
| Monthly cost | ~$0.02 (S3 backup) | $4.99 | $15 | $15-20 | $25 |
| Storage included | Unlimited (disk) | 9GB | 10GB | 20GB | 8GB |
| Horizontal reads | No | Yes (edge) | No (need replicas) | Yes ($$$) | No |
| Setup complexity | Low | Medium | Medium | High | Low |
| Maintenance | None | Managed | Managed | Managed | Managed |

For a single-server deployment, you're looking at $0 for SQLite versus $15-50/month for the cheapest managed Postgres. Over a year, that's $180-600 saved per project. If you're running multiple side projects or microservices, the savings compound.

Turso's free tier (5GB storage, 500M row reads/month) is generous enough for most side projects and early-stage apps. The jump to $4.99/month unlocks unlimited databases and 2.5 billion row reads - still far cheaper than managed Postgres.

## A Real Production Setup

Here's what a production-ready SQLite configuration looks like for a Rust web application:

```rust
use sqlx::sqlite::{SqliteConnectOptions, SqliteJournalMode, SqliteSynchronous};
use sqlx::SqlitePool;
use std::str::FromStr;

pub async fn init_db() -> Result<SqlitePool, sqlx::Error> {
    let options = SqliteConnectOptions::from_str("sqlite:data/app.db")?
        .create_if_missing(true)
        .journal_mode(SqliteJournalMode::Wal)
        .synchronous(SqliteSynchronous::Normal)
        .busy_timeout(std::time::Duration::from_secs(5))
        .pragma("foreign_keys", "ON")
        .pragma("cache_size", "-64000")
        .pragma("temp_store", "MEMORY")
        .pragma("mmap_size", "268435456");

    let pool = SqlitePool::connect_with(options).await?;

    // Run migrations
    sqlx::migrate!("./migrations").run(&pool).await?;

    Ok(pool)
}
```

The directory structure:

```
myapp/
  data/
    app.db          # database file
    app.db-wal      # WAL file (managed by SQLite)
    app.db-shm      # shared memory (managed by SQLite)
  migrations/
    001_initial.sql
  litestream.yml    # backup config
  Dockerfile
```

A Dockerfile that bundles Litestream:

```dockerfile
FROM rust:1.85 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates wget && \
    wget https://github.com/benbjohnson/litestream/releases/download/v0.5.0/litestream-v0.5.0-linux-amd64.tar.gz && \
    tar -xzf litestream-v0.5.0-linux-amd64.tar.gz -C /usr/local/bin/

COPY --from=builder /app/target/release/myapp /usr/local/bin/
COPY litestream.yml /etc/litestream.yml

# Restore database from backup on startup, then run with replication
CMD ["sh", "-c", \
    "litestream restore -if-replica-exists -config /etc/litestream.yml /data/app.db && \
     litestream replicate -config /etc/litestream.yml -exec /usr/local/bin/myapp"]
```

The `litestream replicate -exec` flag starts Litestream as a wrapper around your application. It begins replication, launches your app as a child process, and handles graceful shutdown of both. On startup, `litestream restore` pulls the latest backup if one exists, so you can deploy to a fresh server and pick up right where you left off.

## WAL Mode Gotchas

A few things that bite people in practice:

**WAL file growth.** Under sustained write load, the WAL file can grow large if checkpointing can't keep up. SQLite auto-checkpoints at 1000 pages, but if a long-running read transaction is active, the checkpoint can't advance past that reader's end mark. The fix: keep read transactions short and avoid holding open cursors while doing other work.

**Shared storage.** WAL mode uses shared memory (`-shm` file) for coordination. This requires that all connections come from the same machine. Docker volumes and local filesystems are fine. NFS and most network filesystems are not.

**File permissions.** SQLite needs write access to the directory containing the database, not just the database file itself. It creates the `-wal` and `-shm` files as siblings. If the directory is read-only, WAL mode fails silently and falls back to journal mode.

**VACUUM.** Running `VACUUM` on a large database rewrites the entire file and temporarily requires double the disk space. For databases over a few hundred MB, use `PRAGMA auto_vacuum = INCREMENTAL` and run `PRAGMA incremental_vacuum` periodically instead.

## The Decision Framework

Use SQLite with WAL mode when:

- Single application server (one binary, one VPS, one container)
- Read-heavy or balanced read/write workload
- Database fits comfortably in available RAM (or hot pages do)
- You want zero operational overhead for the database layer
- Cost matters (hobby projects, bootstrapped products, microservices)

Switch to Postgres (or another client-server DB) when:

- You need multiple application servers writing to the same database
- Write throughput exceeds what a single writer can handle
- You need advanced query features (parallel scans, hash joins, CTEs with materialization hints)
- Your dataset is large enough that dedicated database memory management matters
- You need row-level security, logical replication, or pub/sub (`LISTEN`/`NOTIFY`)

The gap between these two categories is wider than most developers think. A single well-configured VPS running SQLite in WAL mode, backed by Litestream, can serve thousands of users and handle the traffic patterns of the vast majority of web applications.

Stop paying for database servers you don't need.
