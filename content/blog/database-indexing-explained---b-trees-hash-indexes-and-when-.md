+++
title = "Database Indexing Explained - B-trees, Hash Indexes, and When to Index"
date = 2025-08-05
description = "How database indexes actually work at the page level, why column order in composite indexes matters more than you think, and how to use EXPLAIN QUERY PLAN to stop guessing."

[taxonomies]
tags = ["sqlite", "databases", "performance", "indexing"]
+++

You can configure WAL mode, tune your PRAGMAs, set up connection pooling - and still have a slow application because one query is doing a full table scan on a 2 million row table. I covered the SQLite configuration side in [Why SQLite with WAL Mode Is Good Enough for Most Web Apps](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/) and the point I made there stands: the library choice matters far less than your indexes and query plans.

This is the indexes and query plans part.

<!-- more -->

## What an Index Actually Is

A database index is a separate data structure that maps column values to the rows that contain them. Think of it like the index at the back of a textbook - instead of reading every page to find where "B-tree" is mentioned, you flip to the index, find the entry, and jump directly to page 247.

Without an index, the database has to read every single row in the table to answer your query. That's a full table scan. For 100 rows, nobody notices. For 10 million rows, your API endpoint takes seconds instead of milliseconds.

The critical thing to understand: an index is a completely separate structure from the table data. It takes up additional disk space, it needs to be updated on every INSERT, UPDATE, and DELETE, and it exists purely to make reads faster. Every index is a tradeoff between read speed and write overhead.

## B-tree Indexes - the Default and the Workhorse

SQLite uses B-trees for everything. Every table, every index, every internal structure. When you `CREATE INDEX`, you're creating a B-tree. When you create a table with a rowid (which is the default), the table itself is stored as a B-tree keyed by rowid. Understanding B-trees is understanding how SQLite thinks.

### The Page Layer

SQLite stores data in fixed-size pages. The default page size is 4096 bytes (4 KB), configurable between 512 and 65536 bytes. The entire database file is an array of these pages. Page 1 is the file header, and every page after that is either part of a B-tree (internal nodes and leaf nodes), an overflow page, a freelist page, or a pointer map page.

A B-tree node occupies exactly one page. Each page contains a small header and then a sorted array of cells. For an index B-tree, each cell holds an index key (the value of the indexed column) and a rowid pointing back to the table. For an internal (non-leaf) node, each cell also holds a pointer to a child page.

Here's the layout of an index B-tree internal page:

```
+------------------------------------------------------------------+
| Page Header (8-12 bytes)                                         |
|   - page type flag (interior vs leaf)                            |
|   - number of cells                                              |
|   - pointer to right-most child page                             |
+------------------------------------------------------------------+
| Cell Pointer Array (2 bytes per cell)                            |
+------------------------------------------------------------------+
| Free space                                                       |
+------------------------------------------------------------------+
| Cell N: [left-child-page-ptr | key-payload | rowid]             |
| Cell N-1: [left-child-page-ptr | key-payload | rowid]           |
| ...                                                              |
| Cell 1: [left-child-page-ptr | key-payload | rowid]             |
+------------------------------------------------------------------+
```

Cells are stored from the bottom of the page upward, while the cell pointer array grows downward from the header. This lets SQLite insert cells without rearranging the entire page - it just appends a new cell to the bottom and adds a pointer entry.

### Fanout and Tree Depth

The power of B-trees comes from fanout - how many children each internal node has. With a 4 KB page and integer keys, an internal node can hold roughly 500 child pointers. That means:

- **Depth 1** (root only): ~500 leaf pages = ~2 MB of index data
- **Depth 2**: ~250,000 leaf pages = ~1 GB
- **Depth 3**: ~125,000,000 leaf pages = ~476 GB

For a table with 10 million rows, the index B-tree is probably 3 levels deep. Finding any row requires reading at most 3 pages from disk. Compare that to a full table scan that might need to read tens of thousands of pages.

This is why indexed lookups are O(log n) - but it's a very wide log. With a fanout of 500, log base 500 of 10 million is about 2.8. Three page reads. If those pages are in the OS page cache (which they usually are for the upper levels of the tree), the actual disk I/O is often just one read for the leaf page.

### Table B-trees vs Index B-trees

SQLite has two flavors of B-tree, and the distinction matters:

**Table B-trees (B*-trees)** store the actual row data. They're keyed by the integer rowid. All content lives in leaf nodes - internal nodes contain only rowids and child page pointers. This maximizes fanout in internal nodes because they don't carry payload data.

**Index B-trees** store index keys and corresponding rowids. Both internal nodes and leaf nodes carry key data. This means internal nodes are heavier, reducing fanout slightly compared to table B-trees.

When you query with an index, two B-tree lookups happen:

1. Walk the **index B-tree** to find the matching key and its rowid
2. Walk the **table B-tree** using that rowid to fetch the full row

That second step is the rowid lookup, and it's important to keep in mind - it's an extra B-tree traversal for every matching row. For queries that match many rows, these lookups add up. This is where covering indexes become valuable, but more on that later.

### WITHOUT ROWID Tables

If you create a table with `WITHOUT ROWID`, SQLite uses a regular B-tree (not B*-tree) where the primary key is the B-tree key and row data is stored in both internal and leaf nodes. This eliminates the rowid lookup for primary key queries, but internal nodes get larger and fanout drops. It's a win for tables with small rows and text primary keys. For most tables, the default rowid B-tree is better.

```sql
-- Default: B*-tree keyed by integer rowid
CREATE TABLE users (
    id INTEGER PRIMARY KEY,  -- this IS the rowid
    email TEXT NOT NULL,
    name TEXT NOT NULL
);

-- WITHOUT ROWID: regular B-tree keyed by primary key
CREATE TABLE sessions (
    token TEXT PRIMARY KEY,
    user_id INTEGER,
    expires_at TEXT
) WITHOUT ROWID;
```

Use `WITHOUT ROWID` when your primary key is already a natural key (like a session token or UUID) and you frequently look up by that key. Don't use it when your rows are large - the reduced fanout from storing data in internal nodes hurts more than the saved rowid lookup helps.

## Hash Indexes

A hash index uses a hash function to map keys directly to bucket locations. Given a key, compute the hash, jump to the bucket, find the row. O(1) average case for exact-match lookups, compared to O(log n) for B-trees.

Sounds better, right? For exact equality checks, it is. But hash indexes have real limitations:

- **No range queries.** `WHERE price > 100` can't use a hash index. The hash function destroys ordering - consecutive values hash to unrelated buckets.
- **No prefix matching.** `WHERE name LIKE 'John%'` can't use a hash index.
- **No ordering.** `ORDER BY created_at` gets nothing from a hash index.
- **No partial matches on composite keys.** A hash index on `(city, name)` can only be used when both columns are specified with equality.

B-trees handle all of these cases because they maintain sorted order. A B-tree index on `created_at` supports `WHERE created_at > '2026-01-01'`, `ORDER BY created_at`, and range scans - all with the same index.

### SQLite Doesn't Have Hash Indexes

This is worth stating clearly: SQLite only supports B-tree indexes. There is no `CREATE INDEX ... USING HASH` syntax. This is a deliberate design choice.

The [SQLite documentation](https://sqlite.org/queryplanner.html) explains the reasoning: SQLite already has a robust, high-performance B-tree implementation at its core. Adding a separate hash table implementation would increase the library size (SQLite is designed for embedded devices where every kilobyte matters) for minimal practical gain. The B-tree's O(log n) with a fanout of 500 is fast enough that the difference from O(1) is negligible for the database sizes SQLite targets.

PostgreSQL, MySQL, and other client-server databases do support hash indexes. In PostgreSQL, hash indexes became crash-safe in version 10 and are useful for very large tables where you only ever do exact equality lookups on a single column. In practice, even PostgreSQL's own documentation notes that B-tree indexes can handle equality checks nearly as well, so hash indexes are a niche optimization.

If you're using SQLite and need O(1) lookups, the practical approach is to compute a hash in your application code and store it as an indexed column:

```sql
-- Store a hash for fast exact lookups
ALTER TABLE urls ADD COLUMN url_hash INTEGER;
CREATE INDEX idx_urls_hash ON urls(url_hash);

-- Query using the hash (with the original for collision safety)
SELECT * FROM urls WHERE url_hash = 1234567890 AND url = 'https://example.com/page';
```

This gives you hash-speed filtering (the B-tree narrows to a handful of rows by hash) with correctness (the `AND url = ...` handles collisions). For most workloads, the plain B-tree index on the original column is fast enough that this trick isn't worth the complexity.

## Composite Indexes

A composite index is an index on multiple columns. It's the single most impactful optimization you can make, and also the one most people get wrong.

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

This creates a B-tree sorted first by `user_id`, then by `status` within each `user_id`. The mental model: think of it as a phone book sorted by last name, then first name.

### The Left-to-Right Rule

The most important rule for composite indexes: **left to right, no skipping, stops at the first range condition.**

An index on `(a, b, c)` can be used for:
- `WHERE a = 1` - uses column `a`
- `WHERE a = 1 AND b = 2` - uses columns `a` and `b`
- `WHERE a = 1 AND b = 2 AND c = 3` - uses all three columns
- `WHERE a = 1 AND b > 5` - uses `a` (equality) and `b` (range), but stops here
- `WHERE a = 1 AND b > 5 AND c = 3` - uses `a` and `b` only. `c` is NOT used because `b` is a range condition

It CANNOT be used for:
- `WHERE b = 2` - skips `a`, can't use the index at all
- `WHERE a = 1 AND c = 3` - skips `b`, only uses `a`
- `WHERE c = 3` - skips `a` and `b`

This has massive practical implications. If your application frequently queries `WHERE status = 'active' AND user_id = 42`, the index `(user_id, status)` works perfectly. But `(status, user_id)` also works - and might be better or worse depending on selectivity. If there are 1000 distinct users and only 3 distinct statuses, `(user_id, status)` narrows the search much faster because `user_id` has higher selectivity.

### Column Order Strategy

Put the most selective columns first (the ones that narrow the result set the most), with one exception: if you have a mix of equality and range conditions, put the equality columns first regardless of selectivity.

```sql
-- Query pattern: WHERE user_id = ? AND created_at > ?
-- Good: equality column first, then range
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- Bad: range column first kills the user_id filter
CREATE INDEX idx_orders_created_user ON orders(created_at, user_id);
```

With the good index, SQLite jumps to the `user_id` section of the B-tree, then scans forward from the `created_at` cutoff. With the bad index, SQLite scans all `created_at` values from the cutoff onward and has to check `user_id` for each one.

## Covering Indexes

A covering index contains all the columns a query needs - both the filtered columns and the selected columns. When SQLite can satisfy a query entirely from the index, it skips the rowid lookup into the table B-tree. The query "covers" itself from the index alone.

```sql
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    category TEXT NOT NULL,
    price_cents INTEGER NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    created_at TEXT NOT NULL
);

-- Regular index: find products, then look up each row for the name
CREATE INDEX idx_products_category ON products(category);

-- Covering index for a common query pattern
CREATE INDEX idx_products_category_price_name ON products(category, price_cents, name);
```

Now compare:

```sql
-- With idx_products_category:
-- 1. Search index B-tree for category = 'electronics' -> get rowids
-- 2. For each rowid, look up the table B-tree to get price_cents and name
-- That's TWO B-tree traversals per matching row

EXPLAIN QUERY PLAN
SELECT name, price_cents FROM products WHERE category = 'electronics';
-- SEARCH products USING INDEX idx_products_category (category=?)

-- With idx_products_category_price_name:
-- 1. Search index B-tree for category = 'electronics'
-- 2. Read price_cents and name directly from the index leaf
-- ONE B-tree traversal total

EXPLAIN QUERY PLAN
SELECT name, price_cents FROM products WHERE category = 'electronics';
-- SEARCH products USING COVERING INDEX idx_products_category_price_name (category=?)
```

The word "COVERING" in the output tells you the table B-tree was never touched. For queries returning many rows, this can be a 2x or larger speedup.

The tradeoff: covering indexes are wider (more columns = more bytes per index entry = fewer entries per page = deeper tree). Don't make every index a covering index. Target the 2-3 hottest query patterns in your application.

## Reading EXPLAIN QUERY PLAN

`EXPLAIN QUERY PLAN` is the tool that turns index optimization from guessing into engineering. Run it before any query and SQLite tells you exactly what it plans to do.

```sql
EXPLAIN QUERY PLAN SELECT * FROM orders WHERE user_id = 42;
```

The output has a few key patterns to recognize:

### SCAN - Full Table Scan

```
SCAN orders
```

This means SQLite is reading every row in the table. For small tables (hundreds of rows), this is fine. For large tables, this is your performance problem.

### SEARCH - Index Lookup

```
SEARCH orders USING INDEX idx_orders_user (user_id=?)
```

SQLite is using the index to jump directly to matching rows. This is what you want.

### SEARCH USING COVERING INDEX

```
SEARCH orders USING COVERING INDEX idx_orders_user_status (user_id=? AND status=?)
```

Even better - the index contains everything the query needs. No table lookup at all.

### SCAN USING INDEX

```
SCAN orders USING INDEX idx_orders_created
```

This is a full scan, but in index order. SQLite reads every row but does it in the order of the index. This happens when you `ORDER BY` an indexed column without a WHERE clause. It's better than a full scan plus a sort, but it's still reading everything.

### USE TEMP B-TREE

```
SCAN orders
USE TEMP B-TREE FOR ORDER BY
```

SQLite is creating a temporary B-tree to sort the results. This means there's no index that satisfies the `ORDER BY`, so it has to sort after scanning. On large result sets, this is expensive.

### A Practical Debugging Session

Here's a real workflow. You have an API endpoint that's slow:

```sql
-- The slow query
SELECT id, status, total_cents
FROM orders
WHERE user_id = 42 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 20;
```

Check the plan:

```sql
EXPLAIN QUERY PLAN
SELECT id, status, total_cents
FROM orders
WHERE user_id = 42 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 20;
-- SCAN orders
-- USE TEMP B-TREE FOR ORDER BY
```

Full scan plus a sort. On 2 million orders, this is painful. Add a composite index:

```sql
CREATE INDEX idx_orders_user_status_created
ON orders(user_id, status, created_at);
```

Check again:

```sql
EXPLAIN QUERY PLAN
SELECT id, status, total_cents
FROM orders
WHERE user_id = 42 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 20;
-- SEARCH orders USING INDEX idx_orders_user_status_created (user_id=? AND status=?)
```

Now SQLite jumps to `user_id=42, status='pending'` in the index, and because `created_at` is the next column, the results are already sorted. No temp B-tree, no full scan. The `LIMIT 20` means SQLite reads exactly 20 entries from the index and stops.

Want to eliminate the table lookup too? Make it a covering index:

```sql
CREATE INDEX idx_orders_user_status_created_covering
ON orders(user_id, status, created_at, id, total_cents);

EXPLAIN QUERY PLAN
SELECT id, status, total_cents
FROM orders
WHERE user_id = 42 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 20;
-- SEARCH orders USING COVERING INDEX idx_orders_user_status_created_covering (user_id=? AND status=?)
```

From full table scan to covering index search - potentially thousands of times faster on a large table.

## When Indexes Hurt

Indexes aren't free. Every index you add has costs that compound silently.

### Write Overhead

Every INSERT must update every index on the table. Every UPDATE that touches an indexed column must update that index. Every DELETE must remove entries from every index.

The numbers are significant. [Benchmarks on PostgreSQL](https://www.percona.com/blog/benchmarking-postgresql-the-hidden-cost-of-over-indexing/) showed that a table with 5 indexes had roughly 40% lower insert throughput compared to a table with 1 index. Informal SQLite tests show similar patterns - inserts into a table with multiple indexes can be 2-4x slower than inserts into an unindexed table.

For a read-heavy web application (which most are), this is an acceptable tradeoff. For a write-heavy workload - event logging, analytics ingestion, IoT sensor data - every unnecessary index directly hurts your throughput ceiling. Remember from the [WAL mode post](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/): SQLite serializes writes, so anything that makes individual writes slower reduces your overall write capacity.

### Storage Overhead

Each index is a separate B-tree that takes up space in the database file. A table with 1 million rows and 3 columns might be 50 MB. Add 5 indexes and the file could be 200 MB. On a VPS with limited NVMe storage, this matters. More importantly, a larger database means more pages, which means more pressure on the OS page cache, which means more disk I/O.

### The Over-Indexing Trap

New developers tend to add an index for every column that appears in a WHERE clause. This creates a mess:

```sql
-- Don't do this
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_orders_total ON orders(total_cents);
```

SQLite can only use one index per table in a query (with some exceptions for OR clauses using its OR-optimization). Having 4 single-column indexes means SQLite picks the best one and ignores the rest. A single composite index that matches your actual query pattern will outperform all of them:

```sql
-- Do this instead
CREATE INDEX idx_orders_user_status_created ON orders(user_id, status, created_at);
```

This one index handles `WHERE user_id = ?`, `WHERE user_id = ? AND status = ?`, and `WHERE user_id = ? AND status = ? ORDER BY created_at` - three query patterns with one index instead of three.

### Redundant Indexes

An index on `(user_id, status)` makes a separate index on `(user_id)` redundant. The composite index already handles lookups by `user_id` alone (it's the leftmost column). SQLite won't complain about redundant indexes - it'll just maintain both, wasting write performance and storage for zero read benefit.

Check for redundant indexes with:

```sql
-- List all indexes on a table
SELECT name, sql FROM sqlite_master
WHERE type = 'index' AND tbl_name = 'orders';
```

Review the output and drop any index whose leftmost columns are a prefix of another index.

## The ANALYZE Command

SQLite's query planner makes decisions based on table and index statistics. Without statistics, it guesses - and sometimes it guesses wrong, choosing a full scan when an index would be faster, or picking the wrong index in a multi-index situation.

```sql
-- Gather statistics for all tables and indexes
ANALYZE;

-- Or for a specific table
ANALYZE orders;
```

`ANALYZE` scans your indexes and records the distribution of values in the `sqlite_stat1` table (and `sqlite_stat4` if compiled with `SQLITE_ENABLE_STAT4`). The query planner reads these statistics to estimate how many rows each index lookup will return, which lets it choose the most efficient plan.

Run `ANALYZE` after:
- Bulk data imports
- Significant data growth (10x or more rows)
- Adding new indexes
- Any time EXPLAIN QUERY PLAN shows a plan that doesn't make sense

In a production application, running `ANALYZE` periodically (daily or weekly, depending on data churn) keeps the query planner informed. It's a read-only scan of your indexes - it doesn't modify data and it's safe to run under load.

## A Practical Indexing Workflow

Here's the process I use for every table that grows beyond trivial size:

**1. Start with no indexes** (besides the primary key). Write your application code. Let your actual query patterns emerge.

**2. Identify slow queries.** Use your application's query logging or SQLite's `sqlite3_profile` callback to find queries that take more than a few milliseconds.

**3. Run EXPLAIN QUERY PLAN** on each slow query. Look for `SCAN` and `USE TEMP B-TREE`.

**4. Design composite indexes** that match your WHERE + ORDER BY patterns. Equality columns first, then range columns, then ORDER BY columns.

**5. Verify with EXPLAIN QUERY PLAN** again. You should see `SEARCH` instead of `SCAN`, and no temp B-tree if your index covers the ORDER BY.

**6. Run ANALYZE** so the query planner has accurate statistics.

**7. Benchmark.** Not just the read query - also check that your writes haven't slowed down unacceptably.

Don't over-think it upfront. Add indexes in response to measured problems, not anticipated ones. A few well-chosen composite indexes will do more for your performance than a dozen single-column indexes scattered across the schema.

Your database is a B-tree machine. Learn to work with the tree, and the tree works for you.
