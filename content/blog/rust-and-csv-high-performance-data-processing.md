+++
title = "Rust and CSV - high-performance data processing"
date = 2025-12-26
description = "Process gigabyte-sized CSV files in seconds with the csv crate, rayon, and a streaming mindset. With benchmarks against Python pandas and notes on what makes Rust so much faster here."

[taxonomies]
tags = ["rust", "csv", "performance", "etl"]
+++

CSV is the format everyone hates and everyone uses. Logs from your reverse proxy, exports from your billing system, data dumps from a Postgres `COPY` - they all end up as comma-separated text. And when the file grows past a few hundred megabytes, the usual workflow ("just open it in pandas") starts to hurt. RAM fills up, your laptop fan spins, the kernel starts swapping. You wait.

The Rust [`csv`](https://docs.rs/csv) crate solves this differently. It is built around streaming byte-level parsing with zero allocations on the hot path, and when combined with [`serde`](https://docs.rs/serde) for typed records and [`rayon`](https://docs.rs/rayon) for data parallelism, you can chew through a 1 GB CSV in a few seconds on a laptop. This post walks through how it works under the hood, where the speed actually comes from, and how to use it for real ETL work.

If you're not already comfortable with serde's data model and the difference between `Deserialize`, `Visitor`, and zero-copy borrows, I covered that in [Serde deep dive - beyond derive](/blog/serde-deep-dive-beyond-derive/). The CSV crate plugs into that machinery directly, so the patterns from that post carry over.

<!-- more -->

## What the csv crate actually is

At its core, [BurntSushi/rust-csv](https://github.com/BurntSushi/rust-csv) is a state machine over bytes. The `Reader` does not work with `String`s by default. It produces `ByteRecord`s - `&[u8]` slices into an internal buffer. UTF-8 validation, allocation, and conversion to typed fields are all opt-in. That choice is why it's fast.

```toml
[dependencies]
csv = "1.3"
serde = { version = "1", features = ["derive"] }
```

A first program that just counts rows:

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let mut rdr = csv::Reader::from_path("access.log.csv")?;
    let mut count = 0u64;
    let mut record = csv::ByteRecord::new();
    while rdr.read_byte_record(&mut record)? {
        count += 1;
    }
    println!("rows: {count}");
    Ok(())
}
```

Two things to notice. First, `record` is allocated once and reused on every iteration - the loop is allocation-free. Second, `read_byte_record` returns `bool` for "got one" vs "EOF". This is the streaming API. Memory usage is constant regardless of file size: a few KB for buffers, full stop.

For comparison, the convenient typed API is `rdr.deserialize::<MyRow>()`, which returns an iterator of `Result<MyRow>` and goes through serde. We'll get to that, but it's important to know the byte-level path exists, because that's where the headline numbers come from.

## The state machine under the hood

CSV looks trivial until you remember the rules: fields can be quoted, quoted fields can contain commas, quoted fields can contain doubled quotes (`""`), records can span multiple lines if quoted, and different dialects use different separators and quote characters. RFC 4180 is the closest thing to a spec, and tools disagree on edge cases.

The csv crate uses a hand-written DFA implemented in [`csv-core`](https://docs.rs/csv-core), the `no_std` underbelly of the high-level crate. The state machine has states like `StartRecord`, `StartField`, `InField`, `InQuotedField`, `InEscapedQuote`, and emits transitions byte by byte. You can use `csv-core` directly in embedded contexts. Here is the simplified shape:

```rust
use csv_core::{ReadFieldResult, Reader};

let mut rdr = Reader::new();
let input = b"a,\"b,c\",d\n";
let mut output = [0u8; 1024];
let mut pos = 0;

loop {
    let (result, n_in, n_out) = rdr.read_field(&input[pos..], &mut output[..]);
    pos += n_in;
    match result {
        ReadFieldResult::Field { record_end } => {
            println!("field = {:?}, end = {}",
                std::str::from_utf8(&output[..n_out]).unwrap(), record_end);
            if record_end { /* row done */ }
        }
        ReadFieldResult::End => break,
        ReadFieldResult::InputEmpty | ReadFieldResult::OutputFull => break,
    }
}
```

Two outputs (`n_in`, `n_out`) and the `OutputFull` result are how the parser stays allocation-free. The caller owns the buffers; the parser just reports how many bytes it consumed and produced. This is the same pattern [`httparse`](https://docs.rs/httparse) and [`memchr`](https://docs.rs/memchr) use, and it is what makes the high-level crate able to reuse a single `ByteRecord` for the whole file.

The DFA itself uses [`memchr`](https://docs.rs/memchr) to vectorize the search for delimiters and quote characters when the field is unquoted - that's `pcmpeqb` / `pcmpestri` on x86, or NEON on ARM. On a typical CSV without much quoting, you're scanning at memory bandwidth.

## Headers, delimiters, quoting

The defaults assume RFC 4180 with a header row. You override with `ReaderBuilder`:

```rust
use csv::ReaderBuilder;

let mut rdr = ReaderBuilder::new()
    .delimiter(b';')          // German Excel exports
    .quote(b'\'')             // single-quoted strings
    .has_headers(false)       // raw data, no header row
    .flexible(true)           // allow ragged rows
    .comment(Some(b'#'))      // skip lines starting with #
    .from_path("data.csv")?;
```

`flexible(true)` is worth knowing about. By default, the reader errors when a record has a different field count than the first one - this catches malformed files early. For real-world log data where some lines genuinely have fewer fields, you turn it off and check `record.len()` per row.

`comment` is unusual for CSV but very common for log dumps and `tcpdump`-style exports. It treats lines starting with the byte as comments and skips them at the byte level, before quoting logic kicks in. Cheaper than filtering after parse.

For headers, `rdr.headers()?` returns a `&StringRecord` (UTF-8 validated) and `rdr.byte_headers()?` returns `&ByteRecord` (raw bytes). If you only need to look up a column index once, do it with the headers and then index by position in the loop - lookup-by-name per row adds up fast.

```rust
let headers = rdr.headers()?.clone();
let status_idx = headers.iter().position(|h| h == "status").unwrap();

let mut record = csv::StringRecord::new();
while rdr.read_record(&mut record)? {
    let status: u16 = record[status_idx].parse()?;
    // ...
}
```

## Serde integration - the typed path

Once your columns map cleanly to a struct, derive `Deserialize` and let serde drive the parser:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct AccessLog {
    timestamp: i64,
    method: String,
    path: String,
    status: u16,
    bytes: u64,
    #[serde(rename = "user-agent")]
    user_agent: String,
}

let mut rdr = csv::Reader::from_path("access.csv")?;
for result in rdr.deserialize::<AccessLog>() {
    let row: AccessLog = result?;
    if row.status >= 500 {
        // ...
    }
}
```

What happens here: `deserialize()` returns an iterator that, for each record, hands a CSV-specific `Deserializer` to serde. That deserializer implements `deserialize_struct` by matching field names against headers and calling the appropriate `visit_*` on the field type's `Visitor`. For numeric fields, the visitor parses bytes directly with `lexical-core`-style fast paths - no intermediate `String` allocation, no double-pass.

If you want the absolute minimum allocation, use `Cow<'a, str>` or `&'a [u8]` borrows. The csv crate supports zero-copy deserialization for byte-record-based reads when the field has no escaping:

```rust
use std::borrow::Cow;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Row<'a> {
    #[serde(borrow)]
    name: Cow<'a, str>,
    age: u32,
}
```

The crate's docs spell out the exact rules. Practically, borrowing pays off when fields are large and unquoted (logs, free-form text). For numeric-heavy data, just use `String` or owned types - the difference is noise.

If a column might be missing or empty, `Option<T>` does the right thing. For nullable strings represented as the literal text `NULL` or `\N`, you write a small custom deserializer:

```rust
fn null_or_string<'de, D>(d: D) -> Result<Option<String>, D::Error>
where D: serde::Deserializer<'de>
{
    let s: &str = serde::Deserialize::deserialize(d)?;
    Ok(if s == "NULL" || s == "\\N" || s.is_empty() { None } else { Some(s.to_string()) })
}

#[derive(Deserialize)]
struct Row {
    #[serde(deserialize_with = "null_or_string")]
    email: Option<String>,
}
```

## Writing CSVs

The writer mirrors the reader. `Writer` buffers output and flushes on drop, which means you must `flush()` (or let it drop) before checking the file - a classic gotcha:

```rust
use csv::Writer;
use serde::Serialize;

#[derive(Serialize)]
struct Row<'a> {
    id: u64,
    name: &'a str,
    score: f64,
}

let mut wtr = Writer::from_path("out.csv")?;
wtr.serialize(Row { id: 1, name: "alice", score: 99.4 })?;
wtr.serialize(Row { id: 2, name: "bob,jr", score: 51.2 })?;
wtr.flush()?;
```

`bob,jr` gets quoted automatically - the writer scans each field for `delimiter`, `quote`, and newline bytes and decides per-field. If you want forced quoting, use `WriterBuilder::quote_style(QuoteStyle::Always)`. For numeric columns where you know quoting is unnecessary, `QuoteStyle::Never` skips the scan and shaves a couple of percent off write time.

## Memory layout and why it's fast

Here is the key invariant the csv crate maintains: parsing a record never requires allocating proportional to the record's size after the initial buffer is grown. `ByteRecord` is internally a single `Vec<u8>` for field bytes plus a `Vec<usize>` of field-end offsets. When you call `read_byte_record(&mut record)`, the crate clears those vecs (which is `O(1)` - just sets `len = 0`) and refills them. Capacity is preserved.

That means the steady-state allocation pattern after the first row is zero. `cachegrind` on a 1 GB CSV shows a couple of dozen allocations total: buffer growth events, the `Reader` itself, the headers. Compare with a naive `BufReader::lines().split(',').collect::<Vec<_>>()` which allocates two `Vec`s and a handful of `String`s per row. On a million-row file, that's tens of millions of allocations, and your CPU time goes mostly to the allocator.

The buffer sizes matter too. `ReaderBuilder::buffer_capacity(1 << 20)` (1 MiB) is a good default for files on fast storage. The default of 8 KiB is tuned for being kind to embedded-style use. For a sequential scan of a multi-GB file from NVMe, larger buffers reduce syscall overhead.

## Benchmarking against pandas

Concrete numbers, with the same machine reading a 1 GB synthetic CSV (10 million rows, 10 columns, mix of numeric and string fields) from local NVMe:

| Approach | Wall time | Peak RSS |
|----------|-----------|----------|
| `pandas.read_csv` (default) | ~38 s | ~3.2 GB |
| `pandas.read_csv(engine="pyarrow")` | ~6.5 s | ~1.4 GB |
| `polars.read_csv` (eager) | ~4.1 s | ~1.1 GB |
| Rust csv crate, byte records | ~3.2 s | ~12 MB |
| Rust csv + serde, typed | ~4.0 s | ~12 MB |
| Rust csv + rayon (8 cores) | ~0.9 s | ~25 MB |

These are rough numbers from runs on a Ryzen 7 7840U laptop, and your mileage will vary with CPU, disk, and field shape. The shape of the result is what matters: pandas in default mode loads the whole file into a DataFrame (hence the 3 GB RSS - it allocates per-column typed arrays plus parsing scratch), while the Rust streaming approach holds a single record in memory at any moment.

The pyarrow engine in pandas closes most of the gap on wall time but still materializes everything. For ETL where you scan once and write somewhere else (another CSV, Parquet, a database), holding the whole thing in memory is just unnecessary cost.

## Parallel processing with rayon

The csv crate is single-threaded. That's fine for a streaming scan because you're usually I/O bound on cold reads. Once the file is in the page cache, you can saturate multiple cores by splitting the work.

The naive way: read all records into a `Vec`, then `.par_iter()`. That defeats the streaming model. The right way is to split the file by byte range and have each worker parse its chunk independently. The csv crate exposes `Reader::position` and `Reader::seek_raw`, but you have to be careful: a byte offset might land in the middle of a quoted field. The standard trick is to seek to a chunk boundary, then scan forward for the next unquoted newline, and start parsing from there.

For most ETL where rows are independent (filtering, transforming, aggregating), a simpler pattern works well: chunk the records in batches and process batches in parallel.

```rust
use rayon::prelude::*;
use serde::Deserialize;

#[derive(Deserialize)]
struct Row { ts: i64, status: u16, bytes: u64 }

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut rdr = csv::Reader::from_path("access.csv")?;
    let batch_size = 50_000;
    let mut batch: Vec<Row> = Vec::with_capacity(batch_size);
    let mut total_5xx_bytes: u64 = 0;

    for result in rdr.deserialize() {
        batch.push(result?);
        if batch.len() == batch_size {
            total_5xx_bytes += batch.par_iter()
                .filter(|r| r.status >= 500)
                .map(|r| r.bytes)
                .sum::<u64>();
            batch.clear();
        }
    }
    // tail
    total_5xx_bytes += batch.par_iter()
        .filter(|r| r.status >= 500)
        .map(|r| r.bytes)
        .sum::<u64>();

    println!("5xx bytes: {total_5xx_bytes}");
    Ok(())
}
```

The reader runs on the main thread and feeds rayon. Because `par_iter` over a `Vec` uses [`split_at_mut`](https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut)-based work stealing, the per-batch overhead is small and you get linear scaling up to the point where parsing on the main thread becomes the bottleneck.

For true parallel parsing, [`csv-async`](https://docs.rs/csv-async) plus tokio works, or you split the file at newline boundaries and spawn rayon tasks per chunk. Crossbeam channels between a parser thread and a pool of worker threads is another common shape. The right choice depends on whether the bottleneck is parsing or downstream work.

## Real-world ETL: reshaping access logs

A pattern I use often: read a CSV access log, filter to errors, enrich with geo data from another file, write to a new CSV. The whole thing in about 40 lines:

```rust
use csv::{Reader, Writer};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Deserialize)]
struct Log { ts: i64, ip: String, status: u16, path: String }

#[derive(Serialize)]
struct Out<'a> { ts: i64, ip: &'a str, country: &'a str, status: u16, path: &'a str }

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Load geo lookup once
    let mut geo: HashMap<String, String> = HashMap::new();
    let mut grdr = Reader::from_path("geo.csv")?;
    #[derive(Deserialize)]
    struct G { ip: String, country: String }
    for r in grdr.deserialize::<G>() {
        let g = r?;
        geo.insert(g.ip, g.country);
    }

    let mut rdr = Reader::from_path("access.csv")?;
    let mut wtr = Writer::from_path("errors.csv")?;
    for r in rdr.deserialize::<Log>() {
        let log = r?;
        if log.status < 400 { continue; }
        let country = geo.get(&log.ip).map(String::as_str).unwrap_or("??");
        wtr.serialize(Out {
            ts: log.ts, ip: &log.ip, country,
            status: log.status, path: &log.path,
        })?;
    }
    wtr.flush()?;
    Ok(())
}
```

Memory is bounded by the geo table size, not the access log size. On a 5 GB log with a 10k-IP geo file, this runs at disk speed and uses ~2 MB of RAM. The equivalent in pandas wants you to load both into DataFrames and `merge` - which works, until the log doesn't fit.

## When to reach for it

- Logs at scale (web servers, message queues, CDN dumps) where the file is bigger than RAM.
- Data transformation pipelines where the CSV is one step (CSV in, Parquet out, or CSV in, CSV out filtered).
- Embedded reporting where binary size matters - the csv crate compiles to about 200 KB stripped.
- Any place a Python script is currently the bottleneck and rewriting it in Rust takes an afternoon.

For interactive analysis, pandas and polars are still nicer. For batch jobs that run on a schedule or on machines without much RAM, csv + serde + rayon is hard to beat. The combination is also a great showcase for how Rust's zero-cost abstractions actually work in practice: the typed `serde` path generates code essentially as tight as hand-written byte parsing, and rayon's data parallelism comes with no setup cost beyond changing `iter` to `par_iter`.

The crates are small, the docs are good, and the API has been stable for years. Fewer surprises than most ecosystems, which is the highest praise you can give a tool you reach for at 2 AM when a customer's CSV broke an import job.
