+++
title = "Write-Ahead Logging - How Databases Survive Crashes"
date = 2026-04-28
description = "WAL is the mechanism that makes database durability possible - here's how it works at the page level, how SQLite and Postgres implement it, and how to build a minimal one in Rust."

[taxonomies]
tags = ["databases", "rust", "systems-programming", "durability"]
+++

Your process crashes mid-transaction. Power cuts out while a write is halfway to disk. The kernel panics during a page flush. In all three cases, your database opens back up with every committed transaction intact and no half-written garbage in the data file. That guarantee comes from a single technique that every serious database engine uses: Write-Ahead Logging.

The idea is deceptively simple. Before you modify any data page in the database file, write a record of what you're about to do into a separate log file. If the process dies before the data file is updated, the log tells you exactly what still needs to happen. If the process dies during the log write itself, the incomplete entry fails a checksum and gets ignored. Either way, the database is recoverable.

I covered the practical side of SQLite's WAL mode - PRAGMAs, benchmarks, production configuration - in [Why SQLite with WAL Mode Is Good Enough for Most Web Apps](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/). This post goes underneath that. We're looking at the mechanism itself: what happens at the page and syscall level, why the ordering of writes matters so much, and what "durable" actually means when your data has to travel through multiple layers of caching before it hits persistent storage.

<!-- more -->

## The fundamental problem: torn writes

Databases store data in fixed-size pages - typically 4096 bytes, matching the OS page size and the physical sector size on most modern storage. A single SQL statement might modify multiple pages: updating a row changes the data page, updating an index changes a B-tree page, updating a counter changes another page.

Here's the problem. Writing those three pages to disk is not atomic. The OS writes them one at a time. If power fails after the first page is written but before the third, you have a database where some pages reflect the new state and others reflect the old state. The data file is internally inconsistent. Indexes point to rows that don't exist, or rows exist that no index references. Your database is corrupted.

This isn't theoretical. It happens. Storage devices might reorder writes internally. The OS page cache batches writes and flushes them on its own schedule. Even a single 4096-byte page write isn't guaranteed to be atomic on all hardware - some older drives have 512-byte physical sectors and a 4K "page write" is actually eight sector writes that can be interrupted.

The question every database engine has to answer: how do you make multi-page updates survive arbitrary crashes?

## Two strategies: undo vs. redo

There are two classical approaches, and most databases use one or both.

**Undo logging (rollback journal):** Before modifying a page in the data file, copy the original page to a separate journal file. Then modify the data file in place. If you crash mid-write, read the journal and restore the original pages. The data file gets the changes directly; the journal exists only to undo them if something goes wrong.

This is what SQLite uses in its default journal mode. The rollback journal contains the original content of every page that was modified, so recovery means copying those pages back.

**Redo logging (write-ahead log):** Don't modify the data file at all during the transaction. Instead, write the new page content to a log file. Readers reconstruct the current state by checking the log first, then falling back to the data file. Periodically, a checkpoint process applies the logged changes to the data file.

This is what SQLite uses in WAL mode, and it's what PostgreSQL, MySQL/InnoDB, SQL Server, and essentially every modern database uses.

The redo approach wins on concurrency. Since the data file isn't modified during writes, readers can keep reading it without locks. The only contention point is the WAL file itself - and since it's append-only, concurrent readers just need to know where to stop reading (their "end mark"). Writers never block readers. Readers never block writers.

## What the WAL file actually looks like

The WAL file is a sequence of frames. Each frame records one modified page. Here's the layout SQLite uses (from [walformat.html](https://sqlite.org/walformat.html)):

```
WAL Header (32 bytes):
  [magic_number:  u32]   0x377f0682 (little-endian) or 0x377f0683 (big-endian)
  [format_version: u32]  currently 3007000
  [page_size:     u32]   database page size in bytes
  [checkpoint_seq: u32]  checkpoint sequence number
  [salt_1:        u32]   random salt for checksums
  [salt_2:        u32]   random salt for checksums
  [checksum_1:    u32]   checksum of first 24 bytes
  [checksum_2:    u32]   checksum of first 24 bytes

Frame Header (24 bytes per frame):
  [page_number:   u32]   which page this frame contains
  [commit_size:   u32]   DB size in pages after commit (0 if not a commit frame)
  [salt_1:        u32]   must match WAL header salt
  [salt_2:        u32]   must match WAL header salt
  [checksum_1:    u32]   cumulative checksum
  [checksum_2:    u32]   cumulative checksum

Frame Data:
  [page_data: page_size bytes]
```

Two details matter here.

First, the checksums are *cumulative*. Frame N's checksum depends on frame N-1's checksum, which depends on N-2, and so on back to the header. This creates a chain - if any frame in the middle is corrupted or partially written, every frame after it will also fail checksum validation. Recovery walks the chain from the beginning and stops at the first broken link. Everything before that point is valid; everything after is discarded.

Second, the `commit_size` field. Most frames have this set to zero, meaning they're part of an uncommitted transaction. Only the last frame of a committed transaction has a non-zero value. This is what makes transactions atomic in the WAL: either all frames up to and including the commit frame are valid (checksums pass), or the commit frame is missing/corrupt and the entire transaction is rolled back by ignoring those frames.

You can examine a live WAL file's structure with SQLite's built-in pragmas:

```sql
PRAGMA wal_checkpoint(PASSIVE);  -- returns: busy, log frames, checkpointed frames
```

Or look at the raw bytes. On a database with 4096-byte pages, each frame is 24 + 4096 = 4120 bytes. The WAL header is 32 bytes. So a WAL file with 100 frames is exactly 32 + (100 * 4120) = 412,032 bytes.

## The wal-index: how readers find pages fast

When a reader needs page 47, it can't scan the entire WAL file looking for the most recent frame containing page 47. That would make reads O(n) in the number of WAL frames, which defeats the purpose.

SQLite solves this with the wal-index - the `-shm` file next to your database. This file is memory-mapped (not read via normal file I/O) and contains a hash table that maps page numbers to WAL frame positions. The hash function is `(page_number * 383) % 4096`, spread across 32KB blocks that each hold 4096 page numbers.

The critical property: the wal-index is *not* needed for crash recovery. It's a performance optimization. If the shm file is missing or corrupt, SQLite rebuilds it by scanning the WAL from the beginning. The WAL file alone is the source of truth.

Each reader "remembers" the WAL frame count at the moment it started its transaction. This is the end mark. The reader sees all frames up to that point and ignores anything appended afterward. This is how readers get a consistent snapshot without any locking - they just read a single integer (the frame count) at transaction start, and that number defines their view of the database.

Here's the full read path:

```
Reader needs page N:
  1. Check wal-index hash table for page N
  2. If found in WAL (frame number <= reader's end mark):
     -> read page data from WAL file at frame offset
  3. If not in WAL:
     -> read page data from main database file
```

This lookup is O(1) per page through the hash table, regardless of WAL size.

## fsync: the gap between "written" and "durable"

When your program calls `write()`, the data goes into the kernel's page cache. It is NOT on disk. The kernel will eventually flush it, but "eventually" could be 30 seconds later. If power fails before the flush, your write is gone.

`fsync()` forces the kernel to flush all modified pages of a file to the storage device and waits until the device confirms the data is on persistent media. In Rust, this is `file.sync_data()` (which maps to `fdatasync()` on Linux - slightly faster than full `fsync()` because it skips metadata updates that aren't needed for data recovery).

Here's the write ordering that makes WAL work:

```
1. Append frame(s) to WAL file
2. fsync(WAL file)         <- data is now durable on the log
3. Return "committed" to the application
...later, during checkpoint...
4. Write pages to database file
5. fsync(database file)    <- data is now durable in the main file
6. Truncate or reset WAL
7. fsync(WAL file)
```

The order of steps 1-2-3 is what makes this safe. The WAL is synced to disk before the application sees "commit successful." If the process crashes after step 2, the WAL contains the committed data and recovery replays it. If the process crashes during step 1 (before the fsync), the partial frame fails its checksum and gets discarded - the transaction was never committed, so losing it is correct.

Steps 4-7 are the checkpoint. The interesting thing: if you crash between steps 4 and 5, some pages in the data file might be updated and others might not. But that's fine - the WAL still has all the frames. On the next startup, recovery will apply them again. The checkpoint is idempotent.

SQLite gives you control over how aggressively it syncs:

```
PRAGMA synchronous = FULL;    -- fsync WAL on every commit (safest, slowest)
PRAGMA synchronous = NORMAL;  -- fsync only during checkpoint (default for WAL mode)
PRAGMA synchronous = OFF;     -- never fsync (fastest, data loss on OS crash)
```

With `NORMAL`, you can lose transactions committed between the last checkpoint and an OS crash or power failure. Application crashes (segfault, panic, OOM kill) are still safe because the OS page cache survives - only a kernel panic or power loss can cause data loss. For most web applications, that's an acceptable tradeoff. For financial systems, use `FULL`.

But even `fsync()` has caveats. Some storage controllers have volatile write caches that report "synced" before data actually reaches persistent NAND or magnetic media. Enterprise SSDs and server-grade drives typically have power-loss protection (capacitors that drain the cache to flash on power failure). Consumer SSDs vary. The [SQLite documentation on atomic commit](https://sqlite.org/atomiccommit.html) has a thorough section on these failure modes and calls them "broken fsync implementations" - there isn't much a database can do if the hardware lies about durability.

## Checkpointing: moving data from WAL to database

The WAL file grows with every write. If unchecked, a WAL that started at 0 bytes might grow to hundreds of megabytes during a burst of writes. Checkpointing transfers committed frames from the WAL back into the main database file, then resets the WAL so it can be reused.

SQLite auto-checkpoints when the WAL reaches 1000 frames (about 4MB with 4096-byte pages). You can adjust this with `PRAGMA wal_autocheckpoint = N` where N is the frame threshold. Setting it to 0 disables auto-checkpoint entirely.

SQLite offers four checkpoint modes through [sqlite3_wal_checkpoint_v2()](https://sqlite.org/c3ref/wal_checkpoint_v2.html):

**PASSIVE** - checkpoint whatever frames it can without waiting. If a reader is still using old frames, those frames stay. If a writer is active, just skip. This never blocks anything and never invokes the busy handler. The tradeoff: the checkpoint might be incomplete.

**FULL** - wait for all readers to finish reading from the WAL, then checkpoint all frames. Writers are blocked during this process via the busy handler. This guarantees the entire WAL gets transferred to the database file.

**RESTART** - same as FULL, but also ensures any new reader that starts after the checkpoint will read from the beginning of the WAL. Used when you want to guarantee the WAL file can be reused from offset zero.

**TRUNCATE** - same as RESTART, but physically truncates the WAL file to zero bytes afterward. The most aggressive option. The WAL file shrinks back to nothing.

The gotcha with checkpointing is long-running read transactions. A reader that started a transaction holds its end mark - the frame count at the time the transaction began. The checkpoint process cannot reclaim any frames past any active reader's end mark, because that reader might still need to reference those frames to construct its consistent view.

This is why the WAL grows under sustained write load with long-running readers. A reporting query that takes 30 seconds to execute prevents the checkpoint from progressing past the frame count that was current when the query started. Meanwhile, writers keep appending frames.

The fix is straightforward: keep read transactions short. Don't start a transaction, do computation outside the database, and then come back to read more. Open a transaction, read what you need, close it.

## WAL vs. rollback journal: a concrete comparison

Here's the same "update two pages" operation under both modes:

**Rollback journal mode:**
```
1. Read original page A from database file
2. Write original page A to journal file
3. Read original page B from database file
4. Write original page B to journal file
5. fsync(journal file)
6. Write modified page A to database file    <- random seek
7. Write modified page B to database file    <- random seek
8. fsync(database file)
9. Delete or truncate journal file
10. fsync(directory entry)
```

That's two fsyncs and two random writes to the database file.

**WAL mode:**
```
1. Append frame for page A to WAL           <- sequential write
2. Append frame for page B to WAL           <- sequential write
3. fsync(WAL file)
```

One fsync, two sequential writes. The data file isn't touched at all during the transaction. That's the performance difference - WAL turns random I/O into sequential I/O.

The rollback journal has one advantage: very large transactions. If a transaction modifies thousands of pages, the WAL file grows by thousands of frames. Readers must check the WAL for every page lookup, and checkpointing that many frames takes time. The rollback journal doesn't have this problem because pages go directly into the data file - there's nothing to checkpoint. SQLite's documentation [notes this tradeoff](https://sqlite.org/wal.html): transactions larger than about 100MB are faster with the rollback journal.

## Building a minimal WAL in Rust

The KV store we built in [Writing a Key-Value Store in Rust](/blog/writing-a-key-value-store-in-rust/) used an append-only log as the primary data store. A WAL is different - it protects writes to a separate data file. The log is temporary; the data file is permanent. Let's build one.

Our store manages a file of fixed-size pages. Every write goes through the WAL first. On startup, any committed WAL entries get replayed into the data file. A `checkpoint` call flushes the WAL to the data file on demand.

```rust
use std::collections::HashMap;
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Seek, SeekFrom, Write};
use std::path::{Path, PathBuf};

const PAGE_SIZE: usize = 4096;
const FRAME_HEADER_SIZE: usize = 12; // page_num(4) + data_len(4) + crc(4)

pub struct PageStore {
    data_file: File,
    wal_file: File,
    wal_path: PathBuf,
    wal_cache: HashMap<u32, Vec<u8>>,
}
```

The `wal_cache` is our in-memory overlay - pages that live in the WAL but haven't been checkpointed to the data file yet. Reads check the cache first.

Opening the store runs recovery automatically:

```rust
impl PageStore {
    pub fn open(path: impl AsRef<Path>) -> io::Result<Self> {
        let data_path = path.as_ref().to_path_buf();
        let wal_path = data_path.with_extension("wal");

        let data_file = OpenOptions::new()
            .read(true).write(true).create(true)
            .open(&data_path)?;

        let wal_file = OpenOptions::new()
            .read(true).write(true).create(true)
            .open(&wal_path)?;

        let mut store = PageStore {
            data_file,
            wal_file,
            wal_path,
            wal_cache: HashMap::new(),
        };

        store.recover()?;
        Ok(store)
    }
}
```

Writing a page appends a frame to the WAL, fsyncs it, then updates the in-memory cache. The data file is NOT touched:

```rust
impl PageStore {
    pub fn write_page(&mut self, page_num: u32, data: &[u8; PAGE_SIZE]) -> io::Result<()> {
        // Build the frame header
        let data_len = PAGE_SIZE as u32;
        let mut crc_input = Vec::with_capacity(4 + PAGE_SIZE);
        crc_input.extend_from_slice(&page_num.to_le_bytes());
        crc_input.extend_from_slice(data);
        let crc = crc32fast::hash(&crc_input);

        // Append frame to WAL: header + page data
        self.wal_file.seek(SeekFrom::End(0))?;
        self.wal_file.write_all(&page_num.to_le_bytes())?;
        self.wal_file.write_all(&data_len.to_le_bytes())?;
        self.wal_file.write_all(&crc.to_le_bytes())?;
        self.wal_file.write_all(data)?;

        // fsync the WAL - this is the durability guarantee
        self.wal_file.sync_data()?;

        // Update in-memory overlay
        self.wal_cache.insert(page_num, data.to_vec());
        Ok(())
    }
}
```

The `sync_data()` call after appending is the critical line. Once it returns, the frame is durable on disk. If the process crashes after this point, recovery will find and replay this frame. If the process crashes during the write (before sync_data completes), the partial frame will have a bad CRC and recovery ignores it.

Reading checks the WAL cache first, then falls back to the data file:

```rust
impl PageStore {
    pub fn read_page(&mut self, page_num: u32) -> io::Result<[u8; PAGE_SIZE]> {
        // WAL has the most recent version
        if let Some(data) = self.wal_cache.get(&page_num) {
            let mut page = [0u8; PAGE_SIZE];
            page.copy_from_slice(data);
            return Ok(page);
        }

        // Fall back to the data file
        let offset = page_num as u64 * PAGE_SIZE as u64;
        let file_len = self.data_file.metadata()?.len();
        if offset + PAGE_SIZE as u64 > file_len {
            return Ok([0u8; PAGE_SIZE]); // page doesn't exist yet
        }
        self.data_file.seek(SeekFrom::Start(offset))?;
        let mut page = [0u8; PAGE_SIZE];
        self.data_file.read_exact(&mut page)?;
        Ok(page)
    }
}
```

Recovery scans the WAL for valid frames and applies them to the data file. It stops at the first frame that fails its checksum - that's the boundary between committed and incomplete data:

```rust
impl PageStore {
    fn recover(&mut self) -> io::Result<()> {
        let frames = self.read_valid_frames()?;
        if frames.is_empty() {
            return Ok(());
        }

        eprintln!("WAL recovery: replaying {} frames", frames.len());

        for (page_num, data) in &frames {
            let offset = *page_num as u64 * PAGE_SIZE as u64;
            self.data_file.seek(SeekFrom::Start(offset))?;
            self.data_file.write_all(data)?;
        }
        self.data_file.sync_data()?;

        // WAL has been applied - truncate it
        self.wal_file.set_len(0)?;
        self.wal_file.sync_data()?;

        Ok(())
    }

    fn read_valid_frames(&mut self) -> io::Result<Vec<(u32, Vec<u8>)>> {
        let wal_len = self.wal_file.metadata()?.len();
        if wal_len == 0 {
            return Ok(vec![]);
        }

        self.wal_file.seek(SeekFrom::Start(0))?;
        let mut frames = Vec::new();
        let frame_size = FRAME_HEADER_SIZE as u64 + PAGE_SIZE as u64;
        let mut pos = 0u64;

        while pos + frame_size <= wal_len {
            let mut header = [0u8; FRAME_HEADER_SIZE];
            if self.wal_file.read_exact(&mut header).is_err() {
                break;
            }

            let page_num = u32::from_le_bytes(header[0..4].try_into().unwrap());
            let data_len = u32::from_le_bytes(header[4..8].try_into().unwrap());
            let stored_crc = u32::from_le_bytes(header[8..12].try_into().unwrap());

            if data_len as usize != PAGE_SIZE {
                break; // corrupted or partial header
            }

            let mut data = vec![0u8; PAGE_SIZE];
            if self.wal_file.read_exact(&mut data).is_err() {
                break; // partial page data
            }

            // Verify checksum
            let mut crc_input = Vec::with_capacity(4 + PAGE_SIZE);
            crc_input.extend_from_slice(&page_num.to_le_bytes());
            crc_input.extend_from_slice(&data);
            if crc32fast::hash(&crc_input) != stored_crc {
                eprintln!("WAL frame at offset {} failed CRC - stopping replay", pos);
                break;
            }

            frames.push((page_num, data));
            pos += frame_size;
        }

        Ok(frames)
    }
}
```

Checkpointing applies all WAL frames to the data file, syncs, and truncates the WAL:

```rust
impl PageStore {
    pub fn checkpoint(&mut self) -> io::Result<usize> {
        let frames = self.read_valid_frames()?;
        let count = frames.len();

        for (page_num, data) in &frames {
            let offset = *page_num as u64 * PAGE_SIZE as u64;
            self.data_file.seek(SeekFrom::Start(offset))?;
            self.data_file.write_all(data)?;
        }
        self.data_file.sync_data()?;

        self.wal_file.set_len(0)?;
        self.wal_file.sync_data()?;
        self.wal_cache.clear();

        Ok(count)
    }
}
```

The Cargo.toml is minimal:

```toml
[package]
name = "page-wal"
version = "0.1.0"
edition = "2021"

[dependencies]
crc32fast = "1.4"
```

Let's test it:

```rust
fn main() -> io::Result<()> {
    // Clean slate
    let _ = std::fs::remove_file("test.db");
    let _ = std::fs::remove_file("test.wal");

    let mut store = PageStore::open("test.db")?;

    // Write page 0
    let mut page = [0u8; PAGE_SIZE];
    page[0..5].copy_from_slice(b"hello");
    store.write_page(0, &page)?;

    // Write page 1
    let mut page2 = [0u8; PAGE_SIZE];
    page2[0..5].copy_from_slice(b"world");
    store.write_page(1, &page2)?;

    // Read back from WAL cache
    let p0 = store.read_page(0)?;
    assert_eq!(&p0[0..5], b"hello");

    // Checkpoint: flush WAL to data file
    let flushed = store.checkpoint()?;
    println!("checkpointed {} frames", flushed);

    // Simulate crash + recovery by reopening
    drop(store);
    let mut store = PageStore::open("test.db")?;
    let p1 = store.read_page(1)?;
    assert_eq!(&p1[0..5], b"world");
    println!("recovery successful - page 1 intact");

    Ok(())
}
```

This is about 150 lines of actual logic. Compare that to SQLite's [wal.c](https://github.com/sqlite/sqlite/blob/master/src/wal.c), which is over 6,000 lines - but the core idea is identical. Ours is missing shared-memory coordination (the shm file), concurrent reader support, cumulative checksums, and transaction boundaries. But the fundamental pattern - write to WAL, fsync, update cache, periodically checkpoint - is exactly what SQLite does.

The key difference from the append-only log in the [KV store post](/blog/writing-a-key-value-store-in-rust/): that log was the data store. Every read replayed the log on startup to rebuild the HashMap. Our WAL is ephemeral - it exists only to protect the data file during writes. After a checkpoint, the WAL is empty and the data file has everything. The log is a safety net, not the source of truth.

## How PostgreSQL does it differently

PostgreSQL's WAL serves the same fundamental purpose but is designed for a much larger problem space. Where SQLite's WAL is one file that gets truncated on checkpoint, PostgreSQL's WAL is a stream of 16MB segments (configurable via `--with-wal-segsize`) stored in the `pg_wal/` directory.

Each position in the stream has a Log Sequence Number (LSN) - a 64-bit byte offset into the total WAL output since the database was initialized. An LSN like `0/15D6A80` means "byte 22,874,752 from the beginning of all WAL data ever written." LSNs only go forward. They're used for replication (a replica says "I've applied everything up to LSN X, send me what's after"), point-in-time recovery, and crash recovery.

PostgreSQL WAL records are more granular than SQLite's page-level frames. A single WAL record might describe "insert tuple at offset 32 on page 47 of table 'orders'" rather than "here's the entire new content of page 47." This makes the WAL smaller for small changes but requires more complex recovery logic - the replay engine needs to understand the semantics of each record type.

The WAL record format (from [xlogrecord.h](https://github.com/postgres/postgres/blob/master/src/include/access/xlogrecord.h)):

```c
typedef struct XLogRecord {
    uint32    xl_tot_len;   // total length of record
    TransactionId xl_xid;  // transaction ID
    XLogRecPtr xl_prev;    // pointer to previous record in log
    uint8     xl_info;     // flag bits
    RmgrId    xl_rmid;     // resource manager ID
    pg_crc32c xl_crc;      // CRC of this record
} XLogRecord;
```

The `xl_rmid` field is what makes PostgreSQL's WAL extensible. Each subsystem (heap storage, B-tree indexes, GiST indexes, etc.) registers as a "resource manager" with its own WAL record types and replay functions. When recovery processes a record, it dispatches to the right resource manager's redo function. This is why extensions like PostGIS can participate in WAL-based replication - they register their own resource managers.

Both databases use the same core principle: write the intention before the action, fsync the log before reporting success, replay on crash. The implementation complexity scales with the feature set - SQLite needs page-level durability for a single-file embedded database, while PostgreSQL needs logical record types for replication, PITR, and a pluggable storage system.

## Why every serious database uses WAL

RocksDB writes to its WAL before inserting into the memtable. MySQL's InnoDB has a redo log that records page modifications before they're applied to the tablespace. SQL Server's transaction log is the same concept. CockroachDB and etcd use Raft's replicated log, which is fundamentally WAL with consensus - if you've read [Consensus Algorithms - Raft Explained Simply](/blog/consensus-algorithms-raft-explained-simply/), the Raft log is a WAL that multiple nodes agree on before committing.

The pattern repeats because the problem is universal: making non-atomic operations (multi-page disk writes) behave atomically. Every storage system that promises durability across crashes has to solve this, and append-only logging with checksummed entries is the cleanest solution anyone has found.

The alternatives are worse. Shadow paging (LMDB's approach) makes a copy of every modified page and atomically swaps the root pointer - but it fragments the data file and makes sequential scans slow. Append-only data structures (like our KV store's log) work but require the entire dataset to be replayed or indexed on startup. WAL hits the sweet spot: the data file can be read directly for fast lookups, the log handles durability, and checkpointing keeps the log bounded.

If you're building anything that persists state to disk - a database, a message queue, a configuration store, even a game save system - understanding WAL isn't optional. It's the difference between "your data might survive a crash" and "your data will survive a crash."
