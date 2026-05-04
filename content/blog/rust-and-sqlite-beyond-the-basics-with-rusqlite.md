+++
title = "Rust and SQLite - beyond the basics with rusqlite"
date = 2026-01-03
description = "Custom functions, hooks, blob I/O, FTS5, virtual tables, the backup API, and how rusqlite stacks up against sqlx. The parts of the SQLite C API that are surprisingly easy to reach from Rust."

[taxonomies]
tags = ["rust", "sqlite", "rusqlite", "databases"]
+++

If your introduction to [rusqlite](https://github.com/rusqlite/rusqlite) was `Connection::open`, `execute`, and `query_map`, you've seen maybe a tenth of what the crate exposes. SQLite has roughly thirty years of accumulated features that most application code never touches: custom SQL functions, virtual tables, an online backup API, hooks that fire on every write, an incremental BLOB I/O layer, full-text search, JSON path operators, and a session extension that records changes as portable changesets.

rusqlite wraps all of this. Some features are behind cargo feature flags, some are gated by a SQLite build option, but the surface area is wide and the ergonomics are good. If you're already running SQLite in production (and after [SQLite is the most underrated database for startups](/blog/sqlite-is-the-most-underrated-database-for-startups/) you should be), the next step is learning what's hiding in that 250 KB shared library you're already linking against.

This post walks through the parts of the SQLite C API that are useful from Rust, with code that compiles against rusqlite 0.31. I'll skip the basics (open a connection, run a query) and focus on the features that come up in production but rarely get covered.

<!-- more -->

## The crate, the features, and the bundled question

rusqlite is split into a core crate plus a long list of feature flags. The ones that matter most:

```toml
[dependencies]
rusqlite = { version = "0.31", features = [
    "bundled",          # compile sqlite3.c, don't link system libsqlite3
    "functions",        # create_scalar_function, create_aggregate_function
    "hooks",            # update_hook, commit_hook, rollback_hook, preupdate_hook
    "blob",             # incremental BLOB I/O
    "backup",           # online backup API
    "vtab",             # custom virtual tables
    "session",          # changeset/patchset (requires SQLITE_ENABLE_SESSION at build time)
    "serde_json",       # serde_json::Value <-> SQLite values
] }
r2d2 = "0.8"
r2d2_sqlite = "0.24"
```

`bundled` is the one I always turn on. Linking against the system libsqlite3 means your binary depends on whatever ancient version ships with the host distro. With `bundled`, rusqlite compiles the SQLite amalgamation into your binary - reproducible, current, and includes the build-time flags rusqlite needs (FTS5, JSON1, RTREE, and so on). The cost is about 1 MB of binary size and a few extra seconds of build time.

If you're shipping a CLI tool or a containerized service, `bundled` is the right answer. If you're a Linux distro packager, you want the system library.

## Connection pooling with r2d2

rusqlite is synchronous. There is no `async fn execute`. SQLite itself is synchronous - it's a library that does file I/O on the calling thread. Wrapping every call in `tokio::task::spawn_blocking` works, but it's noisy. The cleaner pattern is r2d2 with [r2d2_sqlite](https://github.com/ivanceras/r2d2-sqlite):

```rust
use r2d2::Pool;
use r2d2_sqlite::SqliteConnectionManager;
use rusqlite::OpenFlags;

fn build_pool(path: &str) -> Result<Pool<SqliteConnectionManager>, r2d2::Error> {
    let manager = SqliteConnectionManager::file(path)
        .with_flags(OpenFlags::SQLITE_OPEN_READ_WRITE | OpenFlags::SQLITE_OPEN_CREATE)
        .with_init(|c| {
            c.execute_batch("
                PRAGMA journal_mode = WAL;
                PRAGMA synchronous = NORMAL;
                PRAGMA busy_timeout = 5000;
                PRAGMA foreign_keys = ON;
                PRAGMA cache_size = -64000;
                PRAGMA temp_store = MEMORY;
                PRAGMA mmap_size = 268435456;
            ")
        });
    Pool::builder().max_size(8).build(manager)
}
```

The `with_init` closure runs on every connection the pool creates. PRAGMAs in SQLite are scoped to a connection, not the database file, so you have to set them per-connection. The settings above are the same ones from the parent post - WAL mode for concurrent reads, `busy_timeout` so writers retry instead of failing immediately, a 64 MB cache, 256 MB of mmap.

How big should the pool be? For a SQLite-backed web app, the practical answer is: one writer connection, plus N reader connections. WAL mode allows unlimited readers, but only one writer at a time. If your pool has 16 connections all trying to write, fifteen of them are sitting in `busy_timeout` retry loops. Splitting reads and writes into two pools (a write pool of size 1, a read pool of size 8 to 16) is a common pattern.

```rust
struct Db {
    writer: Pool<SqliteConnectionManager>,
    reader: Pool<SqliteConnectionManager>,
}
```

Same database file, different pools. The writer pool serializes writes inside your application before they hit SQLite, which means clean errors instead of `SQLITE_BUSY`.

## Custom scalar functions

The `create_scalar_function` API lets you register a Rust function as if it were a built-in SQL function. Useful for anything SQLite doesn't ship: regex, hashing, custom string transforms, geospatial calculations, anything.

```rust
use rusqlite::functions::FunctionFlags;
use rusqlite::{Connection, Error};

fn register_regex(conn: &Connection) -> rusqlite::Result<()> {
    conn.create_scalar_function(
        "regex_match",
        2,
        FunctionFlags::SQLITE_UTF8
            | FunctionFlags::SQLITE_DETERMINISTIC
            | FunctionFlags::SQLITE_INNOCUOUS,
        |ctx| {
            let pattern: String = ctx.get(0)?;
            let text: String = ctx.get(1)?;
            let re = regex::Regex::new(&pattern)
                .map_err(|e| Error::UserFunctionError(Box::new(e)))?;
            Ok(re.is_match(&text))
        },
    )
}
```

After registering, you can use it in any query on that connection:

```sql
SELECT * FROM logs WHERE regex_match('ERROR \[\d+\]', message);
```

Two flags worth knowing. `SQLITE_DETERMINISTIC` tells the query planner that for the same arguments you always return the same result. This unlocks an optimization where SQLite can hoist the call out of an inner loop or use it in indexes. `SQLITE_INNOCUOUS` declares that the function has no side effects (no file I/O, no global state mutation), which lets it run inside triggers and views even with `PRAGMA trusted_schema = OFF`.

A common trap: you have to register the function on every connection that will use it. With r2d2, that means inside `with_init`. Functions registered on one connection are not visible on another.

## Custom aggregate functions

Aggregates are scalar functions plus state. You implement the `Aggregate` trait with three methods: `init` returns the starting state, `step` is called once per row, and `finalize` returns the result.

Here's a population standard deviation aggregate using Welford's online algorithm. SQLite doesn't ship one (only `STDDEV` in some forks), and the naive sum-of-squares approach is numerically unstable on large datasets:

```rust
use rusqlite::functions::{Aggregate, Context, FunctionFlags};
use rusqlite::Result;

#[derive(Default)]
struct StdDevState {
    n: u64,
    mean: f64,
    m2: f64,
}

struct StdDev;

impl Aggregate<StdDevState, Option<f64>> for StdDev {
    fn init(&self, _: &mut Context<'_>) -> Result<StdDevState> {
        Ok(StdDevState::default())
    }

    fn step(&self, ctx: &mut Context<'_>, state: &mut StdDevState) -> Result<()> {
        let value: f64 = ctx.get(0)?;
        state.n += 1;
        let delta = value - state.mean;
        state.mean += delta / state.n as f64;
        let delta2 = value - state.mean;
        state.m2 += delta * delta2;
        Ok(())
    }

    fn finalize(&self, _: &mut Context<'_>, state: Option<StdDevState>) -> Result<Option<f64>> {
        Ok(state.and_then(|s| {
            if s.n < 2 { None } else { Some((s.m2 / s.n as f64).sqrt()) }
        }))
    }
}

conn.create_aggregate_function(
    "stddev",
    1,
    FunctionFlags::SQLITE_UTF8 | FunctionFlags::SQLITE_DETERMINISTIC,
    StdDev,
)?;
```

```sql
SELECT region, stddev(latency_ms) FROM requests GROUP BY region;
```

There's also a `WindowAggregate` trait for window functions (`OVER(PARTITION BY ...)`), with an additional `inverse` method that lets SQLite remove rows from the window without recomputing from scratch. Worth using if you're computing rolling stats over large tables.

## Hooks - update, commit, rollback

Hooks are callbacks that SQLite invokes around every row change or transaction boundary. They're the right tool when you need to know that data changed without polling.

```rust
use rusqlite::hooks::Action;

conn.update_hook(Some(|action: Action, db: &str, table: &str, rowid: i64| {
    match action {
        Action::SQLITE_INSERT => log::info!("insert {}.{} rowid={}", db, table, rowid),
        Action::SQLITE_UPDATE => log::info!("update {}.{} rowid={}", db, table, rowid),
        Action::SQLITE_DELETE => log::info!("delete {}.{} rowid={}", db, table, rowid),
        _ => {}
    }
}));

conn.commit_hook(Some(|| -> bool {
    log::info!("about to commit");
    false  // return true to ABORT the commit, false to allow
}));

conn.rollback_hook(Some(|| {
    log::warn!("transaction rolled back");
}));
```

The update hook fires after each row change but before the transaction commits. It runs on the same thread that did the write, so don't do anything slow inside it. The classic use is invalidating an in-memory cache, fanning out a notification on a channel, or feeding a search index with `(table, rowid)` pairs to reindex later.

A subtle point: the hook gets the `rowid`, not the row contents. If you want the new values, you have to read them back inside the hook (which means starting a query in the middle of a write, which works but is slower than you'd think). For audit logging, the cleaner approach is the `preupdate_hook` (a separate feature in rusqlite, requires `SQLITE_ENABLE_PREUPDATE_HOOK` at SQLite build time) which gives you both the old and new column values.

For change capture across processes, the session extension (later in this post) is a better fit.

## Blob I/O - reading large blobs without loading them

If you store large binary data in SQLite (images, PDFs, audio chunks), the obvious approach is `SELECT data FROM files WHERE id = ?` which loads the entire blob into a `Vec<u8>`. For a 50 MB file, that's 50 MB of allocation per read.

The incremental BLOB I/O API lets you open a blob like a file and `read_at`/`write_at` ranges:

```rust
use rusqlite::blob::Blob;
use rusqlite::DatabaseName;
use std::io::{Read, Seek, SeekFrom};

let rowid: i64 = conn.query_row(
    "SELECT rowid FROM files WHERE name = ?1",
    ["report.pdf"],
    |r| r.get(0),
)?;

let mut blob = conn.blob_open(
    DatabaseName::Main,
    "files",
    "data",
    rowid,
    true,  // read_only
)?;

let size = blob.len();
let mut buf = vec![0u8; 4096];
blob.seek(SeekFrom::Start(0))?;
loop {
    let n = blob.read(&mut buf)?;
    if n == 0 { break; }
    sink.write_all(&buf[..n])?;
}
```

`Blob` implements `Read`, `Write`, and `Seek`, so it slots into anything that takes `impl Read`. You can stream a 1 GB blob through a hashing function with constant memory. Note the blob has to be allocated at the right size first (insert with `zeroblob(N)` to allocate N zero bytes), and you can't grow a blob with this API - only overwrite existing bytes.

Combine this with `mmap_size` (set in PRAGMAs above) and SQLite will memory-map the blob region rather than copying through the page cache. For read-heavy blob workloads this is dramatically faster than a server database that has to ship bytes over a socket.

## FTS5 - full-text search built in

FTS5 is SQLite's full-text search module. It's compiled in by default in the `bundled` build. You create a virtual table backed by an inverted index, and queries use a simple match syntax with relevance ranking:

```sql
CREATE VIRTUAL TABLE docs_fts USING fts5(
    title,
    body,
    content='docs',          -- back the FTS table with the docs table
    content_rowid='id',
    tokenize='porter unicode61 remove_diacritics 2'
);

-- Keep the FTS index in sync with the source table
CREATE TRIGGER docs_ai AFTER INSERT ON docs BEGIN
    INSERT INTO docs_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;
CREATE TRIGGER docs_ad AFTER DELETE ON docs BEGIN
    INSERT INTO docs_fts(docs_fts, rowid, title, body) VALUES('delete', old.id, old.title, old.body);
END;
CREATE TRIGGER docs_au AFTER UPDATE ON docs BEGIN
    INSERT INTO docs_fts(docs_fts, rowid, title, body) VALUES('delete', old.id, old.title, old.body);
    INSERT INTO docs_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;
```

From rusqlite, the query is just a normal `prepare`/`query_map`:

```rust
let mut stmt = conn.prepare(
    "SELECT d.id, d.title, snippet(docs_fts, 1, '<b>', '</b>', '...', 16) AS preview
     FROM docs_fts
     JOIN docs d ON d.id = docs_fts.rowid
     WHERE docs_fts MATCH ?1
     ORDER BY rank
     LIMIT 20"
)?;

let rows = stmt.query_map([query], |r| {
    Ok((r.get::<_, i64>(0)?, r.get::<_, String>(1)?, r.get::<_, String>(2)?))
})?;
```

Things FTS5 gives you for free: BM25 ranking (`ORDER BY rank`), result highlighting (`highlight()`), context snippets (`snippet()`), boolean operators (`AND`, `OR`, `NOT`), prefix matching (`hello*`), phrase matching (`"hello world"`), and column filters (`title:rust`).

The `tokenize` option above uses Porter stemming so "running" and "runs" match the same stem, plus Unicode normalization with diacritic removal so "café" matches "cafe". For non-Latin languages you can plug in a custom tokenizer in C, or use the [trigram](https://sqlite.org/fts5.html#the_trigram_tokenizer) tokenizer for substring search.

Performance? An FTS5 index on a 1 GB document corpus typically returns top-20 ranked results in single-digit milliseconds. The full-text search post on the [SQLite wiki](https://sqlite.org/fts5.html) has the gory details.

## JSON1 - schema-flexible columns

The JSON1 extension is also bundled in by default. SQLite stores JSON as text, but provides functions and operators to query into it without parsing the whole document.

```sql
CREATE TABLE events (
    id INTEGER PRIMARY KEY,
    received_at TEXT NOT NULL,
    payload TEXT NOT NULL CHECK (json_valid(payload))
);

-- Generated column for an indexed JSON field
ALTER TABLE events ADD COLUMN user_id TEXT
    GENERATED ALWAYS AS (json_extract(payload, '$.user.id')) STORED;

CREATE INDEX events_user_id ON events(user_id);
```

The `->` operator returns a JSON value, `->>` returns a SQL value. From rusqlite:

```rust
let payload = serde_json::json!({
    "user": { "id": "u_42", "email": "alice@example.com" },
    "action": "login",
    "ip": "10.0.0.1"
});

conn.execute(
    "INSERT INTO events (received_at, payload) VALUES (?1, ?2)",
    rusqlite::params![chrono::Utc::now().to_rfc3339(), payload.to_string()],
)?;

let mut stmt = conn.prepare(
    "SELECT id, payload->>'$.action', payload->'$.user'
     FROM events
     WHERE user_id = ?1"
)?;

let rows = stmt.query_map(["u_42"], |r| {
    let id: i64 = r.get(0)?;
    let action: String = r.get(1)?;
    let user_json: String = r.get(2)?;
    let user: serde_json::Value = serde_json::from_str(&user_json).unwrap();
    Ok((id, action, user))
})?;
```

With the `serde_json` feature on rusqlite, you can `r.get::<_, serde_json::Value>(2)?` directly. The conversion happens automatically.

For document-heavy workloads this gives you something close to Postgres `JSONB` ergonomics, with the caveat that SQLite stores JSON as text by default. The 2024 release added a binary format called `JSONB` (no relation to Postgres) accessed via `jsonb_extract` and friends, which is faster for repeated access. If you're on a recent SQLite (3.45+) and parsing the same payload many times, switch to the `jsonb_*` functions.

## Virtual tables - SQL over arbitrary data

A virtual table is a Rust type that pretends to be a table. You implement a few traits and your data appears in `sqlite_master` and works with `SELECT`, `JOIN`, indexes, and the query planner.

```rust
use rusqlite::vtab::{
    eponymous_only_module, Context, IndexInfo, VTab, VTabConnection, VTabCursor, Values,
};

#[repr(C)]
struct EnvVarsTab { base: rusqlite::vtab::sqlite3_vtab }

unsafe impl<'vtab> VTab<'vtab> for EnvVarsTab {
    type Aux = ();
    type Cursor = EnvVarsCursor;

    fn connect(
        _: &mut VTabConnection,
        _: Option<&Self::Aux>,
        _: &[&[u8]],
    ) -> rusqlite::Result<(String, Self)> {
        let schema = "CREATE TABLE x(name TEXT, value TEXT)".to_string();
        Ok((schema, EnvVarsTab { base: Default::default() }))
    }

    fn best_index(&self, info: &mut IndexInfo) -> rusqlite::Result<()> {
        info.set_estimated_cost(1.0);
        Ok(())
    }

    fn open(&'vtab mut self) -> rusqlite::Result<Self::Cursor> {
        Ok(EnvVarsCursor {
            base: Default::default(),
            iter: std::env::vars().collect::<Vec<_>>().into_iter(),
            current: None,
            rowid: 0,
        })
    }
}
```

(The full implementation needs `VTabCursor` with `filter`/`next`/`column`/`rowid` methods - rusqlite's [virtual table examples](https://github.com/rusqlite/rusqlite/tree/master/src/vtab) have a complete CSV implementation.)

Once registered, this works:

```sql
SELECT name, value FROM env WHERE name LIKE 'PATH%';
```

Real-world uses: exposing a directory tree as a table, querying CSV/Parquet files, joining live API data with stored data, building a graph traversal where nodes come from one table and edges come from a virtual table over an external store. SQLite's [series](https://sqlite.org/series.html), [csv](https://sqlite.org/csv.html), and [zipfile](https://sqlite.org/zipfile.html) are all built as virtual tables.

## The backup API

The parent post mentioned `sqlite3 .backup` and Litestream. The backup API is what those tools call under the hood, and it's exposed directly in rusqlite:

```rust
use rusqlite::backup::Backup;
use std::time::Duration;

fn snapshot(src: &Connection, dst_path: &str) -> rusqlite::Result<()> {
    let mut dst = Connection::open(dst_path)?;
    let backup = Backup::new(src, &mut dst)?;
    backup.run_to_completion(
        100,                              // pages per step
        Duration::from_millis(50),        // sleep between steps
        Some(|p| log::info!("backup progress: {}/{}", p.pagecount - p.remaining, p.pagecount)),
    )?;
    Ok(())
}
```

This is online and incremental. The source database stays open for reads and writes during the backup. When a writer modifies a page that hasn't been copied yet, SQLite copies the original version first to keep the snapshot consistent. If a writer modifies a page that has been copied, the backup restarts that page. With short, fast write transactions (which you should be doing anyway), the backup makes steady progress.

For local snapshots this is the right API. For continuous replication to object storage, Litestream wraps the WAL streaming API instead, which is more efficient because it ships only the WAL deltas.

## Session extension - portable changesets

The session extension records a transaction's changes as a binary changeset that you can apply to another database. It's the building block for sync between two SQLite instances.

```rust
use rusqlite::session::{Session, Changeset};

let mut session = Session::new(&conn)?;
session.attach(None)?;  // attach to all tables in the main database

// ... do some inserts/updates/deletes ...
conn.execute("INSERT INTO orders (user_id, total) VALUES (?, ?)", [42, 1999])?;
conn.execute("UPDATE inventory SET reserved = 1 WHERE id = ?", [7])?;

let changeset = session.changeset()?;

// Send `changeset` over the network, write to a file, etc.
// On the other side:
let mut other = Connection::open("replica.db")?;
Changeset::from_slice(&changeset).apply(&mut other, |_| true, |_, _| true)?;
```

The two callbacks are conflict resolution policies: the first decides which tables to apply changes to, the second decides what to do when a row already exists or has been deleted out from under the change.

This is one of the building blocks for tools like [cr-sqlite](https://github.com/vlcn-io/cr-sqlite), which adds CRDT semantics on top. For a custom local-first app where you control both ends, raw changesets are simpler and faster than rolling your own diff protocol.

The session extension is gated by `SQLITE_ENABLE_SESSION` at SQLite build time. With rusqlite's `bundled` plus the `session` feature flag, it's enabled automatically.

## When to use sqlx instead

rusqlite is great. So is [sqlx](https://github.com/launchbadge/sqlx). The differences come down to async, compile-time checking, and which databases you target.

**Pick rusqlite when:**
- You want full access to SQLite-specific features (custom functions, hooks, virtual tables, blob I/O, session extension). sqlx doesn't expose any of these for SQLite.
- Your application is naturally synchronous: a CLI tool, a desktop app, a worker that pulls jobs from a queue. The async overhead buys nothing.
- You want fine-grained control over connections (named statement caches, per-connection extensions, custom collations).
- You're embedding SQLite into a non-server context where pulling in tokio is overkill.

**Pick sqlx when:**
- You're writing an async web service (axum, actix, poem) and most of your stack is `async`. Wrapping every rusqlite call in `spawn_blocking` works but gets verbose.
- You want compile-time SQL verification with the `query!` macro, where queries are checked against a real database at build time.
- You might switch databases later (sqlx supports SQLite, Postgres, MySQL behind one API). The downside is the API is the lowest common denominator.
- You don't need any SQLite-specific features beyond CRUD.

The pragmatic take: in a typical async Rust web service, sqlx is the better default. In a CLI, an embedded engine, or anywhere you need the full SQLite C API, rusqlite wins.

You can also mix them. There's nothing stopping you from using sqlx for the hot path in your handlers and rusqlite for an admin script that needs to install a custom function or run a backup.

## Wrap up

The "rusqlite is just a thin wrapper" framing undersells how much surface area is here. Custom scalar and aggregate functions let you push logic into the database without writing C. Hooks turn writes into events. Blob I/O makes SQLite a serious option for binary content. FTS5 and JSON1 cover the two most common reasons people reach for a search engine or a document store. Virtual tables let you expose anything as SQL. The backup and session APIs are what real replication tools are built on.

If you've been treating SQLite as a simple key-value store with SQL syntax, the next step is reading the [SQLite docs index](https://sqlite.org/docs.html) cover to cover. There's a lot in there. The C API is exhaustive and rusqlite exposes most of it with reasonable Rust ergonomics. None of these features are exotic. They're just rarely covered in tutorials because the tutorials stop at "open a connection."
