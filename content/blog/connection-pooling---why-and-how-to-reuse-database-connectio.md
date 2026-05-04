+++
title = "Connection pooling - why and how to reuse database connections"
date = 2025-07-30
description = "Opening a database connection costs 1-70ms of TCP, TLS, and auth overhead - connection pools amortize that cost across thousands of requests using a borrow/return model."

[taxonomies]
tags = ["rust", "databases", "performance", "architecture"]
+++

Every time your web application handles a request, it probably needs to talk to a database. The naive approach - open a connection, run the query, close the connection - works fine during development. Under load, it falls apart. Each connection involves a TCP handshake, TLS negotiation, and authentication. That overhead compounds across thousands of concurrent requests until your application spends more time establishing connections than running actual queries.

Connection pooling solves this by maintaining a set of pre-established connections that your application borrows and returns. It's the [flyweight pattern](/blog/the-flyweight-pattern-sharing-data-efficiently-in-rust/) applied to network resources: instead of each request owning a private connection, requests share from a common pool. The difference is measured in milliseconds per request and database server load that drops by an order of magnitude.

<!-- more -->

## What makes opening a connection expensive

To understand why pooling matters, you need to know what happens when your application calls `PgPool::connect()` or `TcpStream::connect()` against a database server. It's not one operation - it's a pipeline of at least four network round trips before your first query executes.

**Step 1: TCP handshake.** Your application sends a SYN packet, the server responds with SYN-ACK, your application sends ACK. That's 1.5 round trips. On localhost, this takes under a millisecond. Across a network - your app in one availability zone, your database in another - add 0.5-2ms per round trip. A typical cloud setup sees 1-3ms just for TCP.

**Step 2: TLS handshake.** If you're connecting over TLS (and you should be, even within a VPC), add another 1-2 round trips for the TLS 1.3 handshake, or 2-3 for TLS 1.2. The client and server exchange certificates, negotiate cipher suites, derive session keys. This is computationally expensive on both sides - RSA key exchange involves modular exponentiation on numbers with thousands of bits. On modern hardware with TLS 1.3, expect 2-5ms.

**Step 3: Authentication.** PostgreSQL's default `scram-sha-256` authentication requires two additional message exchanges between client and server. The client sends a `SASLInitialResponse`, the server responds with a challenge, the client sends a `SASLResponse`, the server confirms. Each exchange involves hashing with a high iteration count (4096 by default in PostgreSQL). Add 0.5-1ms.

**Step 4: Backend startup.** PostgreSQL forks a new backend process for each connection. The server allocates memory for the connection state, loads shared catalog caches, and sets up the execution environment. On a cold start, this can take 1-5ms depending on server load. If the server's `max_connections` limit is nearly reached, memory pressure makes this worse.

Total cost of a fresh connection:

| Scenario | Estimated time |
|---|---|
| Localhost, no TLS | 1-2ms |
| Same datacenter, TLS 1.3 | 5-10ms |
| Cross-AZ (same region), TLS 1.3 | 10-25ms |
| Cross-region, TLS 1.3 | 40-70ms |

These numbers come from [benchmarks on local and remote PostgreSQL connections](https://www.cybertec-postgresql.com/en/postgresql-network-latency-does-make-a-big-difference/) and [connection overhead measurements](https://dr-knz.net/local-overheads-in-postgresql-and-cockroachdb.html). Your actual numbers will vary, but the pattern is consistent: establishing a connection takes 10-100x longer than running a simple query.

A `SELECT 1` on an already-open connection takes 50-200 microseconds on localhost. Opening a fresh connection to run that same query takes 5,000-25,000 microseconds in a typical cloud setup. The query itself is rounding error compared to the connection overhead.

Now multiply by traffic. If your API handles 1,000 requests per second and each request opens a fresh connection, you're spending 5-25 seconds of cumulative connection time per second - across all requests. The database server is constantly forking and tearing down processes. The connection overhead dominates your latency budget.

## How a connection pool works

A connection pool maintains a set of already-established database connections. Instead of open-query-close, the flow becomes borrow-query-return:

```
Application                    Pool                    Database
    |                           |                         |
    |--- acquire() ----------->|                         |
    |                           |--- (idle conn) ------->|
    |<-- connection ------------|                         |
    |                           |                         |
    |--- SELECT ... ---------------------------------------->|
    |<-- rows -----------------------------------------------------|
    |                           |                         |
    |--- release() ----------->|                         |
    |                           |--- (back to idle) -----|
    |                           |                         |
```

The first time a connection is requested, the pool creates it (paying the TCP/TLS/auth cost once). Subsequent requests borrow that same connection, avoiding the establishment overhead entirely. When the borrower is done, the connection returns to the pool's idle queue, ready for the next request.

Under the hood, most pool implementations use the same core data structure:

```rust
// Simplified conceptual model - not actual library code
struct Pool<C> {
    idle: Mutex<VecDeque<PooledConnection<C>>>,  // connections waiting to be borrowed
    semaphore: Semaphore,                         // limits total connections
    config: PoolConfig,
    factory: Box<dyn ConnectionFactory<C>>,        // knows how to create new connections
}

struct PooledConnection<C> {
    inner: C,                    // the actual database connection
    created_at: Instant,         // for max_lifetime checks
    last_used: Instant,          // for idle_timeout checks
}
```

The semaphore is the critical piece. It has a fixed number of permits equal to `max_connections`. When you call `pool.acquire()`:

1. Try to acquire a semaphore permit. If all permits are taken, block (or timeout) until one is released.
2. Once you have a permit, check the idle queue for an available connection.
3. If an idle connection exists, pop it from the queue and return it.
4. If the idle queue is empty, create a new connection (you have a permit, so you're within the limit).
5. When the connection is returned to the pool, push it back onto the idle queue and release the semaphore permit.

If you've read the [concurrency primitives post](/blog/concurrency-primitives-in-rust-mutex-rwlock-channels-atomics/), you'll recognize this pattern - it's a bounded channel, essentially. The semaphore enforces the bound, the `VecDeque` is the buffer, and the connections are the messages.

## Pool sizing - the part everyone gets wrong

The most common question: how many connections should the pool hold? The instinct is "more is better" - if 10 connections handle 100 requests/second, surely 100 connections handle 1,000 requests/second. That instinct is wrong.

### The PostgreSQL wiki formula

The [PostgreSQL wiki](https://wiki.postgresql.org/wiki/Number_Of_Database_Connections) has a formula that surprises most people:

```
pool_size = (core_count * 2) + effective_spindle_count
```

Where `core_count` is physical cores on the database server (not hyperthreading logical cores), and `effective_spindle_count` is the number of disk spindles that can work concurrently. For SSDs, this is somewhere between 0 (if your dataset fits in RAM) and 1.

A database server with 4 physical cores and SSD storage:

```
pool_size = (4 * 2) + 1 = 9
```

Nine connections. For a web application that handles thousands of requests per second. That feels wrong, but it's backed by years of PostgreSQL operational experience.

The reason: database operations are CPU-bound or I/O-bound, and the server has a fixed number of cores. With 4 cores, at most 4 queries can execute in parallel on CPU. If some queries are waiting on disk I/O, a few extra connections let the CPU stay busy while others are blocked on I/O. Beyond that, additional connections just add context-switching overhead and contention on shared resources (buffer pool, WAL locks, lock manager).

### What happens when you overprovision

Say you set `max_connections = 200` because you have 20 application instances each with a pool of 10. The PostgreSQL server now manages 200 backend processes. Each one:

- Consumes ~5-10MB of memory for connection state, work memory, and catalog caches
- Competes for shared buffer pool access (protected by LWLocks)
- Competes for WAL insert locks during writes
- Gets scheduled by the OS across your 4-8 cores

With 200 processes on 8 cores, the OS spends significant time context-switching between them. Each context switch flushes CPU caches, evicts hot data from L1/L2, and adds scheduling latency. Your individual query latency goes up even though total throughput is flat or declining.

This is why [PgBouncer](https://www.pgbouncer.org/) exists - it sits between your application pools and PostgreSQL, multiplexing hundreds of application connections down to a handful of actual PostgreSQL connections. Transaction-level pooling means PgBouncer borrows a real connection only for the duration of a transaction, then returns it.

### Practical sizing guidelines

For a single application instance talking to PostgreSQL:

| Workload | Suggested pool size |
|---|---|
| Light (< 50 req/s) | 5 |
| Medium (50-500 req/s) | 10-15 |
| Heavy (500+ req/s) | 15-25 |
| Multiple app instances | Use PgBouncer, keep total connections under `(cores * 2) + spindles` |

Start small. Monitor. Increase only when you see acquisition wait times climbing. The `pool.acquire()` latency is your signal - if it's consistently above 1ms, your pool might be too small. If it's always near zero but your database CPU is pegged, your pool might be too large.

For SQLite, sizing is a different game entirely. If you've read [Why SQLite with WAL Mode Is Good Enough for Most Web Apps](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/), you know that SQLite allows one writer at a time regardless of pool size. A pool of 5-10 connections gives you concurrent readers while writes are serialized by the engine itself.

## The Rust connection pool landscape

Rust has four major approaches to connection pooling. Each makes different tradeoffs around sync vs async, generics vs specialization, and configuration complexity.

### r2d2 - the synchronous workhorse

[r2d2](https://crates.io/crates/r2d2) (v0.8.10) is the original generic connection pool for Rust. It predates async/await and works with blocking connections - `diesel::PgConnection`, `postgres::Client`, anything that implements the `ManageConnection` trait.

```rust
use diesel::pg::PgConnection;
use r2d2::Pool;
use r2d2_diesel::ConnectionManager;
use std::time::Duration;

fn build_pool(database_url: &str) -> Pool<ConnectionManager<PgConnection>> {
    let manager = ConnectionManager::<PgConnection>::new(database_url);

    Pool::builder()
        .max_size(10)
        .min_idle(Some(2))
        .connection_timeout(Duration::from_secs(5))
        .idle_timeout(Some(Duration::from_secs(300)))
        .max_lifetime(Some(Duration::from_secs(1800)))
        .test_on_check_out(true)
        .build(manager)
        .expect("failed to build pool")
}
```

The `ManageConnection` trait is how r2d2 stays generic. A connection manager knows how to create, validate, and detect broken connections:

```rust
pub trait ManageConnection: Send + Sync + 'static {
    type Connection: Send + 'static;
    type Error: std::error::Error + 'static;

    fn connect(&self) -> Result<Self::Connection, Self::Error>;
    fn is_valid(&self, conn: &mut Self::Connection) -> Result<(), Self::Error>;
    fn has_broken(&self, conn: &mut Self::Connection) -> bool;
}
```

`is_valid` runs when `test_on_check_out` is true (default). For PostgreSQL, it executes `SELECT 1` or equivalent. `has_broken` is a cheaper check that doesn't require a round trip - it checks local connection state, like whether the TCP socket is still open.

The main limitation: r2d2 is synchronous. Calling `pool.get()` blocks the calling thread until a connection is available. In an async application, this means you need `spawn_blocking` to avoid blocking the tokio runtime's worker threads:

```rust
let pool = pool.clone();
let conn = tokio::task::spawn_blocking(move || pool.get())
    .await
    .expect("blocking task panicked")
    .expect("failed to get connection");
```

This works, but it's clunky and wastes a slot in tokio's blocking thread pool. If you're writing a new async application, you probably want an async pool.

### deadpool - lightweight async pooling

[deadpool](https://crates.io/crates/deadpool) (v0.13) is the async counterpart to r2d2. It's designed for async runtimes (tokio and async-std) and has a minimal core with database-specific adapters.

```rust
use deadpool_postgres::{Config, ManagerConfig, RecyclingMethod, Runtime};
use tokio_postgres::NoTls;

fn build_pool() -> deadpool_postgres::Pool {
    let mut cfg = Config::new();
    cfg.host = Some("localhost".to_string());
    cfg.port = Some(5432);
    cfg.dbname = Some("myapp".to_string());
    cfg.user = Some("myapp".to_string());
    cfg.password = Some("secret".to_string());

    cfg.manager = Some(ManagerConfig {
        recycling_method: RecyclingMethod::Verified,
    });

    cfg.create_pool(Some(Runtime::Tokio1), NoTls)
        .expect("failed to create pool")
}
```

deadpool's architecture is intentionally simple. No background threads, no periodic health checks. It uses a `tokio::sync::Semaphore` for concurrency control and a `VecDeque` for the idle connection queue. Everything happens lazily:

- Connections are created on demand, not eagerly on pool startup
- Health checks run only when a connection is about to be handed out (via `RecyclingMethod`)
- Idle connections are reaped only when something triggers pool maintenance

The `RecyclingMethod` enum controls how aggressively connections are validated:

```rust
pub enum RecyclingMethod {
    Fast,       // just check if the connection is closed (tokio_postgres::Client::is_closed())
    Verified,   // run a test query (SELECT 1)
    Clean,      // reset session state (DISCARD ALL) + test query
    Custom(String), // run a custom query
}
```

`Fast` (the default) is nearly zero-cost - it checks local state without a network round trip. `Verified` adds a round trip but catches stale connections that the server has dropped. `Clean` also resets any session-level state (temporary tables, SET variables, prepared statements), which matters if your application modifies session state and you don't want those changes leaking to the next borrower.

deadpool integrates neatly with config crates. The `Config` struct derives `serde::Deserialize`, so you can load pool configuration directly from TOML, JSON, or environment variables:

```toml
# config.toml
[pg]
host = "localhost"
port = 5432
dbname = "myapp"
user = "myapp"
password = "secret"

[pg.pool]
max_size = 16
timeouts.wait.secs = 5
timeouts.create.secs = 5
timeouts.recycle.secs = 5
```

### bb8 - the battle-tested async alternative

[bb8](https://crates.io/crates/bb8) (v0.9) is another async connection pool with a design closer to r2d2's. It uses the same `ManageConnection`-style trait pattern:

```rust
use bb8::Pool;
use bb8_postgres::PostgresConnectionManager;
use tokio_postgres::NoTls;

async fn build_pool() -> Pool<PostgresConnectionManager<NoTls>> {
    let manager = PostgresConnectionManager::new_from_stringlike(
        "host=localhost port=5432 dbname=myapp user=myapp password=secret",
        NoTls,
    )
    .expect("failed to create manager");

    Pool::builder()
        .max_size(15)
        .min_idle(Some(2))
        .connection_timeout(std::time::Duration::from_secs(5))
        .idle_timeout(Some(std::time::Duration::from_secs(300)))
        .max_lifetime(Some(std::time::Duration::from_secs(1800)))
        .test_on_check_out(true)
        .build(manager)
        .await
        .expect("failed to build pool")
}
```

The API mirrors r2d2 closely, which makes migration straightforward if you're moving from sync to async. Internally, bb8 uses a `futures-channel` waiter queue and `parking_lot::Mutex` for the idle connection store.

The practical difference between bb8 and deadpool is mostly ergonomic. bb8 has more configuration knobs (`min_idle`, `max_lifetime`, `test_on_check_out`) out of the box. deadpool is lighter and relies on adapter crates for database-specific features. Both are production-proven - bb8 has over 19 million total downloads.

### sqlx's built-in pool - the integrated option

If you're using [sqlx](https://crates.io/crates/sqlx) (v0.8.6), you already have a connection pool. `PgPool`, `SqlitePool`, and `MySqlPool` are type aliases for `Pool<Postgres>`, `Pool<Sqlite>`, and `Pool<MySql>` respectively. No separate crate needed.

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

async fn build_pool() -> sqlx::PgPool {
    PgPoolOptions::new()
        .max_connections(10)
        .min_connections(2)
        .acquire_timeout(Duration::from_secs(5))
        .idle_timeout(Duration::from_secs(300))
        .max_lifetime(Duration::from_secs(1800))
        .test_before_acquire(true)
        .connect("postgres://myapp:secret@localhost/myapp")
        .await
        .expect("failed to create pool")
}
```

sqlx's pool has a feature the standalone pools don't: automatic minimum connection maintenance. If `min_connections` is set, sqlx spawns a background task that monitors the pool and creates new connections when the count drops below the minimum. This means your first few requests after a cold start don't pay the connection establishment cost.

The `test_before_acquire` option (default: `true`) runs `Connection::ping()` before returning a connection. This is a lightweight protocol-level check that catches dead connections without running an actual SQL query. It's cheaper than deadpool's `RecyclingMethod::Verified` but slightly less thorough - it won't catch a connection where the server is alive but the session is in a bad state.

The integration advantage is real. When you're already using `sqlx::query!` for compile-time checked SQL (which I covered in [the database migrations post](/blog/database-migrations-in-rust-sqlx-diesel-and-sea-orm-compared/)), using sqlx's pool means one less dependency and no impedance mismatch between your pool and your query layer.

### When to use which

| Scenario | Pool |
|---|---|
| Diesel (sync) application | r2d2 |
| Diesel-async application | deadpool (via `diesel-async` integration) |
| sqlx application | sqlx's built-in pool |
| tokio-postgres directly | deadpool-postgres or bb8-postgres |
| Need maximum config knobs | bb8 |
| Need minimal dependencies | deadpool |

If you're starting a new project, sqlx with its built-in pool is the path of least resistance. If you're using Diesel, the ecosystem has already decided for you - `diesel-async` uses deadpool.

## Health checks, idle timeout, and max lifetime

These three settings work together to keep your pool healthy. Get them wrong and you'll see mysterious connection failures at 3 AM.

### Health checks: catching dead connections

Connections die. The database server restarts. A network partition happens. An idle connection hits the server's `idle_in_transaction_session_timeout`. A firewall drops the TCP connection after it's been idle too long. Your pool doesn't know any of this happened - the connection still looks open from the client side because TCP doesn't detect broken connections until you try to send data.

Without health checks, your application borrows a dead connection, sends a query, and waits for a response that never comes (until the TCP timeout fires, typically 15-30 seconds later). Under load, this cascades - multiple requests grab dead connections simultaneously, all of them hang, new requests pile up waiting for pool permits, and your application appears frozen.

Health checks prevent this by testing the connection before handing it out:

```rust
// sqlx: ping check (default, fast, protocol-level)
PgPoolOptions::new()
    .test_before_acquire(true)

// deadpool: verified check (runs SELECT 1)
ManagerConfig {
    recycling_method: RecyclingMethod::Verified,
}

// r2d2 / bb8: test_on_check_out (runs is_valid(), typically SELECT 1)
Pool::builder()
    .test_on_check_out(true)
```

The cost of a health check is one network round trip - 0.1-1ms depending on network latency. For applications where that overhead is unacceptable (high-frequency trading, real-time games), you can disable health checks and handle connection errors at the application level with retry logic. For 99% of web applications, the health check cost is negligible compared to the query itself.

### Idle timeout: reaping unused connections

When traffic drops - maybe it's 3 AM and nobody's using your app - the pool might be holding 10 open connections that aren't doing anything. Each idle connection consumes memory on the database server (~5-10MB per connection for PostgreSQL) and counts against `max_connections`.

`idle_timeout` closes connections that haven't been used within a specified duration:

```rust
PgPoolOptions::new()
    .idle_timeout(Duration::from_secs(300))  // close after 5 minutes idle
```

When a connection sits idle longer than this, it's removed from the pool on the next maintenance pass (or when someone tries to acquire it). The pool shrinks during low-traffic periods and grows back organically as traffic increases.

A good default is 5-10 minutes. Too short and you're paying connection establishment costs during normal traffic fluctuations (a brief lull between requests). Too long and you're holding database resources during extended quiet periods.

### Max lifetime: rotating connections proactively

Even active connections should be recycled eventually. `max_lifetime` closes a connection after it's been open for a fixed duration, regardless of whether it's idle or in use:

```rust
PgPoolOptions::new()
    .max_lifetime(Duration::from_secs(1800))  // close after 30 minutes
```

Why rotate connections that are working fine? Several reasons:

**Memory leak mitigation.** Some database drivers or prepared statement caches grow memory over time. Rotating connections periodically resets that state.

**DNS changes.** If your database connection string points to a hostname (common with managed databases like RDS), and the underlying IP changes (during a failover, for instance), existing connections stay pinned to the old IP. New connections resolve to the new IP. Without `max_lifetime`, your pool holds stale connections to the old server indefinitely.

**Load balancer fairness.** If you're connecting through a load balancer (PgBouncer, HAProxy, or AWS RDS Proxy), long-lived connections can cause uneven distribution. Rotating connections gives the load balancer opportunities to redistribute.

**PostgreSQL memory.** Long-lived PostgreSQL backend processes can accumulate fragmented memory from query execution. Terminating the connection lets the OS reclaim that memory cleanly.

A typical `max_lifetime` of 30 minutes to 1 hour works for most applications. Set it shorter than the database server's own connection timeout (`idle_in_transaction_session_timeout` in PostgreSQL, `wait_timeout` in MySQL) to ensure your pool recycles before the server forces a disconnect.

### How the three settings interact

Here's the lifecycle of a pooled connection with all three settings active:

```
t=0s     Connection created (TCP + TLS + auth)
t=1s     Borrowed for query, returned to idle
t=10s    Borrowed again, returned
t=300s   idle_timeout check: last used at t=10s, 290s ago - still under 300s
t=610s   idle_timeout check: last used at t=10s, 600s ago - EXCEEDS 300s, close it
```

Or with max_lifetime:

```
t=0s     Connection created
t=60s    Borrowed, returned
t=1800s  max_lifetime reached - connection closed on next return, even if busy
```

The connection gets closed by whichever limit fires first. A connection that's actively used every few seconds will never hit `idle_timeout` but will eventually hit `max_lifetime`. A connection that goes unused will hit `idle_timeout` first.

## Practical setup: Axum + sqlx + PostgreSQL

Let's put everything together in a production-ready Axum application. This isn't a toy example - it's close to what you'd actually deploy.

```rust
use axum::{
    extract::State,
    http::StatusCode,
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use sqlx::postgres::PgPoolOptions;
use sqlx::PgPool;
use std::time::Duration;
use tokio::net::TcpListener;

#[derive(Clone)]
struct AppState {
    db: PgPool,
}

#[tokio::main]
async fn main() {
    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://myapp:secret@localhost/myapp".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(10)
        .min_connections(2)
        .acquire_timeout(Duration::from_secs(3))
        .idle_timeout(Duration::from_secs(300))
        .max_lifetime(Duration::from_secs(1800))
        .test_before_acquire(true)
        .connect(&database_url)
        .await
        .expect("failed to connect to database");

    // Run migrations at startup
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("failed to run migrations");

    let state = AppState { db: pool };

    let app = Router::new()
        .route("/api/users", get(list_users).post(create_user))
        .route("/api/users/{id}", get(get_user))
        .route("/health", get(health_check))
        .with_state(state);

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

The `AppState` struct holds the pool. Axum clones this state for each request, but `PgPool` is an `Arc` internally - cloning it is just an atomic reference count increment, as covered in [the flyweight pattern post](/blog/the-flyweight-pattern-sharing-data-efficiently-in-rust/). No new connections are created.

The handler functions acquire connections from the pool:

```rust
#[derive(Serialize, sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    email: String,
    created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

async fn list_users(
    State(state): State<AppState>,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let users = sqlx::query_as::<_, User>("SELECT id, name, email, created_at FROM users ORDER BY created_at DESC LIMIT 100")
        .fetch_all(&state.db)
        .await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    Ok(Json(users))
}

async fn get_user(
    State(state): State<AppState>,
    axum::extract::Path(id): axum::extract::Path<i64>,
) -> Result<Json<User>, (StatusCode, String)> {
    let user = sqlx::query_as::<_, User>("SELECT id, name, email, created_at FROM users WHERE id = $1")
        .bind(id)
        .fetch_optional(&state.db)
        .await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?
        .ok_or((StatusCode::NOT_FOUND, "user not found".to_string()))?;

    Ok(Json(user))
}

async fn create_user(
    State(state): State<AppState>,
    Json(input): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<User>), (StatusCode, String)> {
    let user = sqlx::query_as::<_, User>(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email, created_at",
    )
    .bind(&input.name)
    .bind(&input.email)
    .fetch_one(&state.db)
    .await
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    Ok((StatusCode::CREATED, Json(user)))
}
```

Notice that `fetch_all`, `fetch_optional`, and `fetch_one` accept `&PgPool` directly. sqlx handles the borrow/return transparently - it acquires a connection, executes the query, and returns the connection to the pool. You never call `pool.acquire()` explicitly unless you need a connection for multiple queries in sequence:

```rust
async fn transfer_credits(
    State(state): State<AppState>,
    Json(input): Json<TransferRequest>,
) -> Result<StatusCode, (StatusCode, String)> {
    // Explicit acquire: hold one connection for the entire transaction
    let mut conn = state.db.acquire().await
        .map_err(|e| (StatusCode::SERVICE_UNAVAILABLE, e.to_string()))?;

    let mut tx = conn.begin().await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    sqlx::query("UPDATE accounts SET balance = balance - $1 WHERE id = $2")
        .bind(input.amount)
        .bind(input.from_id)
        .execute(&mut *tx)
        .await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    sqlx::query("UPDATE accounts SET balance = balance + $1 WHERE id = $2")
        .bind(input.amount)
        .bind(input.to_id)
        .execute(&mut *tx)
        .await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    tx.commit().await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    Ok(StatusCode::OK)
}
```

The transaction holds the borrowed connection for both queries. It's returned to the pool when `tx` is committed or when the `conn` binding is dropped.

### The health check endpoint

A health check endpoint that verifies the pool is functioning:

```rust
async fn health_check(State(state): State<AppState>) -> impl IntoResponse {
    match sqlx::query("SELECT 1").execute(&state.db).await {
        Ok(_) => (StatusCode::OK, "ok"),
        Err(_) => (StatusCode::SERVICE_UNAVAILABLE, "database unavailable"),
    }
}
```

This isn't just checking if the pool struct exists - it actually acquires a connection and runs a query. If the pool is exhausted (all connections busy and `acquire_timeout` would fire), or the database is unreachable, the health check fails. Load balancers and orchestrators use this to route traffic away from unhealthy instances.

## What goes wrong in practice

Connection pool bugs are some of the hardest to diagnose because they only show up under load and the symptoms look like something else entirely.

### Connection leak: the silent pool drain

The most common pool bug: acquiring a connection and never returning it. In Rust, this usually means holding a `PoolConnection` across an `.await` point that never completes, or accidentally moving it into a spawned task that outlives the request.

```rust
// BUG: connection leaks if the channel never receives
async fn bad_handler(State(state): State<AppState>) -> impl IntoResponse {
    let conn = state.db.acquire().await.unwrap();

    let (tx, rx) = tokio::sync::oneshot::channel::<()>();

    // If the sender is dropped without sending, rx.await returns Err
    // but conn is still held. It's only returned when this future completes.
    rx.await.ok();

    // conn is still alive here, but we might never reach this point
    drop(conn);

    StatusCode::OK
}
```

The symptom: your pool gradually runs out of connections. Requests start timing out on `acquire()`. The database shows `max_connections` open, but most of them are idle in the pool's view - they're "borrowed" by tasks that aren't doing anything with them.

The fix: always use the implicit borrow pattern (`sqlx::query().fetch_all(&pool)`) instead of explicit `pool.acquire()` when possible. When you do need explicit acquisition, keep the scope tight and watch for `.await` points between acquire and drop.

### Thundering herd on cold start

If your pool starts with `min_connections = 0` and your application gets a burst of traffic immediately (common after deployment), every request tries to create a connection simultaneously. If you get 50 requests in the first 100ms, the pool tries to open 50 connections at once. The database server gets slammed with 50 simultaneous TCP/TLS/auth handshakes.

The fix: set `min_connections` to at least 2-5. The pool pre-creates these connections at startup, so the first few requests have warm connections ready:

```rust
PgPoolOptions::new()
    .min_connections(5)  // pre-warm on startup
    .max_connections(15)
```

### Transaction timeout starvation

A long-running transaction holds a connection for its entire duration. If a single handler opens a transaction, runs an expensive query, calls an external API, and then commits - that connection is unavailable to the pool for the entire time. Under load, a few slow transactions can starve the pool:

```rust
// BAD: holds a pool connection while waiting for an HTTP response
let mut tx = pool.begin().await?;
sqlx::query("INSERT INTO orders ...").execute(&mut *tx).await?;

// This HTTP call might take 5 seconds. The connection is blocked.
let payment_result = reqwest::get("https://payments.example.com/charge")
    .await?;

sqlx::query("UPDATE orders SET paid = true ...").execute(&mut *tx).await?;
tx.commit().await?;
```

The fix: do I/O outside the transaction. Query, commit, then call external services. If you need atomicity across the database and external service, use the outbox pattern or saga instead.

## Monitoring your pool

You can't manage what you don't measure. sqlx doesn't expose pool metrics directly (there's an [open issue](https://github.com/launchbadge/sqlx/issues/1523) for it), but you can build basic observability:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;

#[derive(Clone)]
struct PoolMetrics {
    acquisitions: Arc<AtomicU64>,
    acquisition_timeouts: Arc<AtomicU64>,
    query_errors: Arc<AtomicU64>,
}

impl PoolMetrics {
    fn new() -> Self {
        Self {
            acquisitions: Arc::new(AtomicU64::new(0)),
            acquisition_timeouts: Arc::new(AtomicU64::new(0)),
            query_errors: Arc::new(AtomicU64::new(0)),
        }
    }
}

async fn monitored_query<T>(
    pool: &PgPool,
    metrics: &PoolMetrics,
    query_fn: impl std::future::Future<Output = Result<T, sqlx::Error>>,
) -> Result<T, sqlx::Error> {
    metrics.acquisitions.fetch_add(1, Ordering::Relaxed);

    match query_fn.await {
        Ok(result) => Ok(result),
        Err(e) => {
            if matches!(e, sqlx::Error::PoolTimedOut) {
                metrics.acquisition_timeouts.fetch_add(1, Ordering::Relaxed);
            }
            metrics.query_errors.fetch_add(1, Ordering::Relaxed);
            Err(e)
        }
    }
}
```

For production monitoring, the deadpool-postgres crate exposes `Pool::status()` which returns the current pool state:

```rust
let status = pool.status();
println!(
    "pool: size={}, available={}, waiting={}",
    status.size,
    status.available,
    status.waiting,
);
```

Key signals to watch:

- **`waiting > 0` consistently**: Pool is undersized. Requests are queueing for connections.
- **`available` always near `max_size`**: Pool is oversized. You're holding database resources for no reason.
- **Acquisition timeout rate > 0.1%**: Something is wrong - either pool is too small, or connections are leaking.
- **`size` fluctuating rapidly**: Connections are being created and destroyed frequently. Check your `idle_timeout` and `max_lifetime` settings - they might be too aggressive.

If you're using Prometheus for monitoring (which I touched on in [Monitoring Rust Applications in Production](/blog/monitoring-rust-applications-in-production/)), export these as gauges and counters, and alert on `waiting > 0` sustained for more than 30 seconds.

## The full configuration checklist

Before deploying, walk through this checklist:

```rust
PgPoolOptions::new()
    // Sizing
    .max_connections(10)         // start here, increase based on monitoring
    .min_connections(2)          // pre-warm to avoid cold start stampede

    // Timeouts
    .acquire_timeout(Duration::from_secs(3))   // fail fast if pool exhausted
    .idle_timeout(Duration::from_secs(300))    // reap idle connections after 5 min
    .max_lifetime(Duration::from_secs(1800))   // rotate connections every 30 min

    // Health
    .test_before_acquire(true)   // catch dead connections before use

    .connect(&database_url)
    .await?;
```

For each setting, the reasoning:

- **`max_connections(10)`**: Start conservative. More than `(db_cores * 2) + 1` for a single application instance rarely helps. Monitor acquisition latency to decide if you need more.
- **`min_connections(2)`**: Enough to handle the first few requests without cold-start latency. Not so many that you waste database resources when idle.
- **`acquire_timeout(3s)`**: Three seconds is long enough to survive brief contention spikes but short enough to fail fast when something is genuinely wrong. Your HTTP client timeout should be larger than this.
- **`idle_timeout(300s)`**: Five minutes of idle means the connection is reusable during normal traffic patterns but releases during extended quiet periods.
- **`max_lifetime(1800s)`**: Thirty minutes ensures DNS changes propagate within half an hour and prevents unbounded memory growth in long-lived connections.
- **`test_before_acquire(true)`**: The default, and you should keep it. The cost (~0.1ms) is negligible compared to the cost of handing out a dead connection.

Connection pooling isn't glamorous. There's no clever algorithm, no novel data structure. It's a bounded buffer with some timers attached. But it's one of those infrastructure details that separates "works on my laptop" from "handles 1,000 requests per second without breaking a sweat." Get the pool configuration right once, and it quietly does its job for the lifetime of your application.
