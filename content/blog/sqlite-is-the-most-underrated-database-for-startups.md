+++
title = "SQLite is the most underrated database for startups"
date = 2025-04-26
description = "Your startup probably doesn't need managed Postgres - SQLite handles thousands of req/s, costs $0/month, and the ecosystem around it has quietly become production-grade."

[taxonomies]
tags = ["sqlite", "databases", "startups", "architecture"]
+++

Every startup I've seen in the last two years starts the same way. Day one: pick a managed Postgres provider. Day two: configure connection strings, SSL certs, VPC peering, and a backup schedule. Day three: get the first $15 invoice for a database that handles 12 requests per minute from the founder's browser.

That $15/month is not the real cost. The real cost is the operational surface area you just adopted. Connection pool exhaustion at 3 AM. Major version upgrades that need downtime windows. A credentials rotation that takes down staging because someone forgot to update the secret. All of this for an application that could have been a single file on disk.

SQLite is the most deployed database engine in the world. It runs on every smartphone, every browser, every embedded device. But somehow the default advice for web applications is still "use Postgres" - even when the application will serve fewer requests than SQLite handles during its own test suite.

That's changing. The tooling around SQLite has matured to the point where running it in production isn't a compromise - it's a deliberate architectural advantage.

<!-- more -->

## The $0/month database

SQLite is not a server. It's a library linked into your application. There's no daemon to monitor, no port to firewall, no authentication to configure. Your database is a single file on disk. Backups are `cp app.db app.db.bak` (with the caveat that you should use the backup API or Litestream for consistency during writes - more on that later).

This means:

- No monthly bill for a database server
- No connection strings to manage across environments
- No network latency between your app and your data
- No "the database is on a different subnet" debugging sessions
- No credentials to rotate

For a startup running on a single $5-10 VPS, this is the entire database bill: $0. You're already paying for the server that runs your application. SQLite uses that same server's disk and memory.

Compare that to the cheapest managed Postgres options in 2026:

| Provider | Cheapest plan | Storage | What you get |
|---|---|---|---|
| **SQLite on your VPS** | $0 | Your disk | Everything |
| **Neon** (free tier) | $0 | 0.5 GB | 100 compute-hours/month, then it sleeps |
| **Supabase** (free tier) | $0 | 500 MB | Paused after 7 days inactive |
| **DigitalOcean** | $15/mo | 10 GB | Single node, no HA |
| **Supabase Pro** | $25/mo | 8 GB | Always-on, daily backups |
| **AWS RDS** (db.t4g.micro) | ~$15/mo | 20 GB | Plus storage/IO costs |
| **PlanetScale** | $39/mo | 10 GB | Branching, no foreign keys |

The free tiers have sharp edges. Neon's free plan gives you 100 compute-hours per month - roughly 4 days of continuous usage. Supabase pauses your database if you don't use it for a week. These are great for demos, not for a product you're trying to get off the ground.

The moment you need a database that stays on and handles real traffic, you're paying $15-25/month minimum. That's $180-300/year per project. If you're a solo founder running three side projects while figuring out which one has legs, that's $540-900/year in database hosting alone.

SQLite: $0. For all three.

## How much traffic can it actually handle?

This is where most people's intuition is wrong. They assume SQLite is for prototypes and toy apps. The numbers say otherwise.

I covered the technical details of WAL mode configuration and benchmarks in [Why SQLite with WAL Mode Is Good Enough for Most Web Apps](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/), so I won't repeat the PRAGMA setup here. The short version: enable WAL mode, set `busy_timeout`, tune your cache, and SQLite becomes a different beast.

Here are real benchmark numbers from multiple independent sources:

**Read throughput:**
- [phiresky's benchmarks](https://phiresky.github.io/blog/2020/sqlite-performance-tuning/): **100,000 SELECTs/second** with concurrent readers on a multi-GB database
- [Expensify](https://use.expensify.com/blog/scaling-sqlite-to-4m-qps-on-a-single-server): **4 million queries per second** on a single bare-metal server (production workload, not synthetic)
- Typical $10 VPS (2 vCPU, NVMe): 5,000-10,000 reads/second with WAL mode

**Write throughput:**
- [Stephen Margheim's Rails benchmarks](https://fractaledmind.com/2023/12/05/sqlite-myths-linear-writes-do-not-scale/): peak **2,730 write requests/second** on a MacBook Pro with 10 concurrent workers
- Production benchmarks on 8-core Intel with SSD: **16,461 writes/second** at 64 concurrent workers
- Typical $10 VPS: 1,000-3,000 writes/second

**Mixed workloads (the realistic scenario):**
- 80% reads / 20% writes on modest hardware: ~9,400 ops/second
- Typical web app on a $10 VPS: 2,000-5,000 mixed ops/second

Let me put those VPS numbers in context. If your average API endpoint executes two queries (one read, one write), 2,000 ops/second gives you about 1,000 HTTP requests per second. If your average user makes one request every 3 seconds, that's 3,000 concurrent users. On a ten dollar server.

For context on what "real traffic" looks like:

- Twitter in 2007 handled 600 requests/second
- Shopify in 2013 handled 833 requests/second
- Most SaaS startups with paying customers see 10-100 req/s at peak

Your startup is not Twitter. Your startup is not even 2007 Twitter. A single SQLite database on a single VPS will handle your traffic for months or years before you need to think about anything else.

## The single-writer "limitation" in practice

The most common objection to SQLite in production is the single-writer constraint. WAL mode allows unlimited concurrent readers, but only one write transaction can execute at a time. Other writers queue up, waiting for their turn (with `busy_timeout` controlling how long they wait before giving up).

This sounds scary. In practice, it's not.

A typical web application write - insert a row, update a counter, create a session - takes 1-5 milliseconds. At 2ms per write, a single writer can handle 500 writes per second. At 0.5ms (common for simple inserts on NVMe), that's 2,000 writes per second.

The key insight: write transactions in a web app should be fast. If your write transaction is slow, the problem isn't SQLite's single-writer model - the problem is your transaction. Long-running transactions that hold the write lock while doing network calls or heavy computation will tank your throughput on any database. SQLite just makes this mistake more visible.

The pattern that works:

```rust
// Good: fast, focused write transaction
let result = sqlx::query!(
    "INSERT INTO orders (user_id, total_cents, status) VALUES (?, ?, ?)",
    user_id, total, "pending"
)
.execute(&pool)
.await?;

// Bad: holding a write lock while doing external work
let tx = pool.begin().await?;
sqlx::query!("UPDATE inventory SET reserved = true WHERE id = ?", item_id)
    .execute(&mut *tx).await?;
let payment = stripe_client.charge(amount).await?; // network call inside transaction!
sqlx::query!("INSERT INTO payments (order_id, stripe_id) VALUES (?, ?)", order_id, payment.id)
    .execute(&mut *tx).await?;
tx.commit().await?;
```

Do the external call first, then write the result in a single fast transaction. This is good practice regardless of your database, but SQLite rewards it more directly.

## Backups: the `cp` that actually works

One of the most underappreciated things about SQLite is how trivially simple backups become.

With Postgres, backups mean `pg_dump` (which takes a lock and scales linearly with database size), WAL archiving (complex to set up), or managed backup features (which cost extra and have retention limits).

With SQLite, you have three options:

**Option 1: SQLite backup API** - Use `.backup` in the CLI or the `sqlite3_backup_*` API. This creates a consistent point-in-time copy even while the database is being written to. It's safe and built into SQLite itself.

```bash
sqlite3 app.db ".backup /backups/app-$(date +%Y%m%d).db"
```

**Option 2: Filesystem snapshot** - If your VPS provider supports volume snapshots (DigitalOcean, Hetzner, Vultr all do), a snapshot of the volume containing your database is an instant, consistent backup. This costs pennies.

**Option 3: Litestream** - This is the production-grade option. [Litestream](https://litestream.io/) runs as a sidecar that continuously streams WAL changes to S3, R2, GCS, or Azure Blob Storage. Sub-second replication lag. Automatic. Set it once and forget it.

I covered the Litestream setup with Docker in the [WAL mode post](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/), but the cost breakdown is worth repeating: a typical web app database under 1 GB generates maybe $0.02/month in S3 storage costs. Two cents. For continuous, point-in-time recovery.

Compare that to managed Postgres backups:
- Supabase free tier: 7-day retention, daily only
- AWS RDS: automated backups free up to allocated storage, but point-in-time recovery requires provisioned IOPS
- DigitalOcean: daily backups add 20% to your monthly cost

## The ecosystem that changed everything

Five years ago, recommending SQLite for a production web app was a hard sell because the tooling gaps were real. No replication, no distribution, no easy path to scale out. That's no longer the case. A constellation of tools has filled every meaningful gap.

### Litestream - disaster recovery solved

[Litestream](https://litestream.io/) v0.5 (released late 2025) settled into its final architecture: a standalone binary that replicates WAL changes to object storage. No FUSE mounts, no custom VFS - just a sidecar process that watches your WAL file and streams pages to S3.

The v0.5 release also added read replica support through an optional VFS extension. Read-only workloads can serve queries directly from replica storage without restoring a full database copy. The VFS builds a page index from LTX files and fetches pages on demand. This means you can have a secondary service reading from your replicated database in a different region, without running a full restore.

### Turso - distributed SQLite at the edge

[Turso](https://turso.tech/) is built on [libSQL](https://github.com/tursodatabase/libsql), a fork of SQLite that adds the features SQLite can't add because of its strict backwards compatibility policy. The big ones:

- **Concurrent writes** - libSQL implements the `BEGIN CONCURRENT` approach that vanilla SQLite only has in an experimental branch. Multiple writers can operate simultaneously as long as they don't modify the same B-tree pages.
- **Native vector search** - built-in similarity search for embeddings, no extension needed.
- **Network protocol** - SQLite over HTTP/WebSocket, so you can use it from serverless functions or browser-based apps.
- **Edge replication** - your database replicated to locations close to your users.

Turso's pricing is aggressively startup-friendly:

| Plan | Cost | Databases | Row reads/mo | Row writes/mo | Storage |
|---|---|---|---|---|---|
| Free | $0 | 100 | 500M | 10M | 5 GB |
| Developer | $4.99/mo | Unlimited | 2.5B | 25M | 9 GB |
| Scaler | $24.92/mo | Unlimited | 100B | 100M | 24 GB |

500 million row reads on the free tier. That's not a toy. A typical API request might read 5-10 rows - that's 50-100 million API requests per month before you pay anything. Most startups won't hit that for a long time.

### Cloudflare D1 - SQLite in the serverless model

If you're building on Cloudflare Workers, [D1](https://developers.cloudflare.com/d1/) gives you SQLite databases at the edge with automatic replication. The pricing follows Cloudflare's pattern: generous free tier (5M rows read/day, 100K rows written/day, 5 GB storage), then pay-as-you-go.

D1 is a good fit if you're already in the Cloudflare ecosystem. If you're not, Turso or plain SQLite + Litestream are simpler choices.

### cr-sqlite - CRDTs for conflict-free sync

[cr-sqlite](https://github.com/vlcn-io/cr-sqlite) adds Conflict-free Replicated Data Types to SQLite tables. This enables peer-to-peer sync between SQLite databases without a central server. Think local-first applications where each user has a local SQLite database and changes merge automatically.

This is more niche, but for apps where offline capability matters (field service tools, collaborative editors, mobile-first products), it's a compelling option that simply doesn't exist in the Postgres world.

## The total cost of ownership calculation

Let's do the math for a real scenario: a SaaS startup in its first year, running a web API with a database backend.

### Scenario: managed Postgres

```
DigitalOcean Managed Postgres (basic):     $15/mo  = $180/yr
   or Supabase Pro:                        $25/mo  = $300/yr
   or AWS RDS (db.t4g.micro):              $15/mo  = $180/yr
     + storage ($0.115/GB):                 ~$3/mo  =  $36/yr
     + backup storage:                      ~$2/mo  =  $24/yr

Application server (DigitalOcean droplet):  $12/mo  = $144/yr

Total (cheapest):                                    $324/yr
Total (Supabase):                                    $444/yr
Total (AWS):                                         $384/yr
```

### Scenario: SQLite on the same server

```
Application server (same droplet):          $12/mo  = $144/yr
Litestream to S3:                           ~$0.05/mo = ~$1/yr
SQLite:                                     $0      = $0/yr

Total:                                               $145/yr
```

That's $179-299 saved in the first year. For a bootstrapped founder, that's real money. But the cost savings aren't even the biggest win - it's the operational simplicity.

With managed Postgres, you're managing:
- Connection strings per environment (dev, staging, production)
- Database credentials and rotation
- Connection pool configuration
- SSL certificate management
- Backup verification
- Major version upgrades (Postgres 16 -> 17)
- Network connectivity between app server and database
- Monitoring database CPU, memory, connections, replication lag

With SQLite, you're managing:
- A file on disk
- Litestream config (10 lines of YAML)

That's not just fewer dollars. It's fewer things that can break at 2 AM. Fewer things to document for your co-founder. Fewer things to debug when the deploy fails.

## When you actually need to leave SQLite

SQLite is not the right choice for every situation. Here are the concrete thresholds where you should reach for something else. Not "might want to consider" - actually need to.

**You need multiple application servers writing to the same database.** This is the hard boundary. If you're scaling horizontally - multiple containers, Kubernetes pods, separate machines - they can't share a SQLite file. Network filesystems don't work reliably. This is where you need a client-server database, or you switch to Turso which solves this specific problem.

Note the emphasis on "writing." If you have multiple read-only replicas, Litestream's VFS can handle that. The constraint is concurrent writes from multiple machines.

**Your write throughput consistently exceeds 1,000-2,000 writes/second.** For most startups, this means you've already won. You have a product people use. You have revenue. You can afford the Postgres migration. But until you see sustained write pressure at this level, SQLite is fine.

**You need features that SQLite doesn't have.** Specifically:
- `LISTEN`/`NOTIFY` for real-time subscriptions
- Row-level security policies
- Parallel query execution across multiple cores
- Hash joins (SQLite only does nested loops)
- Full-text search with ranking more sophisticated than FTS5
- Logical replication to a data warehouse
- `JSONB` with GIN indexes for document queries at scale

If you need one of these on day one, pick Postgres. If you might need them someday, start with SQLite and migrate when "someday" arrives.

**Your database is large and your queries are analytical.** SQLite's query planner is good but not as sophisticated as Postgres. Once you're joining across tables with millions of rows and running aggregations with complex filtering, Postgres will execute those queries faster. The crossover point is roughly "when your database exceeds available RAM and your queries scan large ranges."

## The migration escape hatch

The fear of starting with SQLite is that migration will be painful when you outgrow it. If you write SQL directly in your handlers, yes, it will be painful - SQLite and Postgres have different SQL dialects, different type systems, different function names.

But if you're using the repository pattern - which I wrote about in [The Repository Pattern - Abstracting Data Access in Rust](/blog/the-repository-pattern-abstracting-data-access-in-rust/) - migration is swapping one implementation for another behind the same trait. Your business logic never touches SQL directly.

```rust
// Your handler doesn't care what's behind the trait
async fn create_order(repo: &dyn OrderRepository, input: CreateOrderInput) -> Result<Order> {
    let order = repo.create(NewOrder {
        user_id: input.user_id,
        total_cents: input.total_cents,
        status: OrderStatus::Pending,
    }).await?;

    Ok(order)
}
```

When the day comes to switch to Postgres, you write a new `PostgresOrderRepository` that implements the same trait, run a data migration script, and swap the implementation in your dependency injection setup. The rest of your codebase doesn't change.

If you set up your architecture with this in mind from day one (and it takes maybe 30 minutes of extra work), the migration fear evaporates. You're not locked in. You're making a reversible choice.

## Companies running SQLite in production

This isn't a theoretical argument. Real companies run SQLite at serious scale:

- **Expensify** - scaled SQLite to 4 million queries per second on a single server. Their production workload. Not a benchmark.
- **Fly.io** - built their entire platform around the idea that SQLite on a single machine, close to your users, beats a remote database server. They created LiteFS for distributed SQLite replication.
- **Tailscale** - uses SQLite for their coordination server's data storage.
- **Signal** - stores message databases in SQLite on device, with the server-side components also using SQLite where appropriate.
- **Pieter Levels (Levelsio)** - runs multiple profitable products (Nomad List, Remote OK, Photo AI) on single-server architectures with SQLite. Millions in annual revenue.
- **Rails 8** - DHH and the Rails team made SQLite a first-class production database in Rails 8, with built-in support for Solid Cache, Solid Queue, and Solid Cable all backed by SQLite. This wasn't an experiment - it was a statement about where production databases are heading.

The SQLite homepage has a page titled [Appropriate Uses For SQLite](https://sqlite.org/whentouse.html) that's worth reading. The SQLite authors themselves state that SQLite works well for sites with fewer than 100K hits/day - but that estimate is conservative and based on default configuration without WAL mode. With WAL mode and proper PRAGMAs, the ceiling is much higher.

## The five-minute production setup

If you're convinced and want to try this, here's the minimal production-ready setup. I'll use Rust as the example since that's what I write, but the SQLite configuration applies to any language.

**Step 1: Configure SQLite properly**

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 5000;
PRAGMA foreign_keys = ON;
PRAGMA cache_size = -64000;
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 268435456;
```

I explained what each of these does and why in the [WAL mode post](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/). The short version: WAL mode enables concurrent reads, `synchronous = NORMAL` trades a tiny durability edge case for major write speed, `busy_timeout` prevents immediate failures under contention, and the cache/mmap settings keep hot data in memory.

**Step 2: Set up Litestream**

```yaml
# litestream.yml
dbs:
  - path: /data/app.db
    replicas:
      - type: s3
        bucket: my-app-backups
        path: app
        region: us-east-1
        access-key-id: ${AWS_ACCESS_KEY_ID}
        secret-access-key: ${AWS_SECRET_ACCESS_KEY}
```

If you're using Cloudflare R2 instead of S3 (free egress), replace the replica config:

```yaml
    replicas:
      - type: s3
        bucket: my-app-backups
        path: app
        endpoint: https://<account-id>.r2.cloudflarestorage.com
        access-key-id: ${R2_ACCESS_KEY_ID}
        secret-access-key: ${R2_SECRET_ACCESS_KEY}
```

**Step 3: Wrap your app with Litestream**

```bash
litestream replicate -config /etc/litestream.yml -exec "./myapp"
```

Done. Your database is now continuously replicated to object storage. Recovery is:

```bash
litestream restore -config /etc/litestream.yml /data/app.db
```

Total setup time: 5 minutes. Total ongoing cost: ~$0.02/month for S3 storage. Total operational complexity: near zero.

## The mindset shift

The reason SQLite is underrated isn't technical - the benchmarks speak for themselves. It's psychological. Developers have been trained to reach for client-server databases by default. "Real" applications use Postgres. SQLite is for mobile apps and prototypes. This was reasonable advice in 2015 when Litestream didn't exist, Turso didn't exist, and the only way to back up SQLite was to stop your application.

In 2026, that advice is outdated. The ecosystem has caught up. The tooling is production-grade. The performance was always there - people just didn't configure it properly.

Start with SQLite. Build your product. Talk to users. Find product-market fit. When (if) you outgrow it, you'll have revenue to fund the migration and data to prove you need it.

Stop paying for infrastructure you don't need yet. The most successful startups I know optimize for speed of iteration, not for scale they haven't reached. SQLite is the fastest path from "idea" to "running in production" - and it'll stay running in production a lot longer than you think.
