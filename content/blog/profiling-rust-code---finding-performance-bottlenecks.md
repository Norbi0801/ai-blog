+++
title = "Profiling Rust code - finding performance bottlenecks"
date = 2025-12-09
description = "A practical guide to profiling Rust programs with perf, cargo-flamegraph, criterion, and dhat - from identifying bottlenecks to verifying fixes."

[taxonomies]
tags = ["rust", "performance", "profiling", "tooling"]
+++

Your Rust program compiles, passes tests, and gives the right answers. But it's slow. You have a hunch about where the problem is - maybe that loop, maybe that serialization call, maybe all those `.clone()`s you sprinkled around to make the borrow checker happy. But hunches are wrong more often than not. I've lost count of how many times I was *sure* about a bottleneck, only to have a profiler point at something completely different.

Profiling is about replacing guesses with data. This post walks through the tools I actually use to find and fix performance problems in Rust code: `perf`, `cargo-flamegraph`, `criterion`, and `dhat`. Not just how to run them - how to read their output, what patterns to look for, and how to close the loop from "this is slow" to "this is fixed."

<!-- more -->

## Before you profile: release mode and debug symbols

The single most common profiling mistake is profiling a debug build. Rust's debug builds run with `opt-level = 0`, which means no inlining, no loop unrolling, no dead code elimination, and bounds checks everywhere. A debug build can be 10-50x slower than release. If you profile it, you'll find bottlenecks that don't exist in your actual production binary.

Always profile release builds. But you also need debug symbols, otherwise your profiler output will be a wall of memory addresses instead of function names. The good news: Rust includes debug symbols in release builds by default since Cargo sets `debug = true` in the release profile. You can verify this in your `Cargo.toml`:

```toml
[profile.release]
debug = true    # full debug info, default in modern Cargo
# debug = 2    # equivalent, explicit level
```

If you want maximum optimization with symbols:

```toml
[profile.release]
opt-level = 3
debug = true
lto = "thin"      # link-time optimization
codegen-units = 1 # slower compile, better optimization
```

The `codegen-units = 1` setting forces the compiler to process the entire crate as one unit, enabling more aggressive cross-function optimization. `lto = "thin"` adds cross-crate inlining. Both slow down compilation significantly, but can yield another 10-20% runtime improvement over the default release profile.

There's also a useful middle ground for profiling - a custom profile:

```toml
[profile.profiling]
inherits = "release"
debug = true
strip = false
```

Build it with `cargo build --profile profiling`. This keeps your regular release builds clean while giving you a dedicated profiling target.

### Quick note on optimization levels

Here's what each level actually does:

| Level | What it does | When to use |
|-------|-------------|-------------|
| `opt-level = 0` | No optimizations. Fast compile. | Development, debugging |
| `opt-level = 1` | Basic optimizations. Moderate compile. | Rarely useful on its own |
| `opt-level = 2` | Most optimizations. Good compile/perf balance. | Default for release |
| `opt-level = 3` | All optimizations including vectorization. | Compute-heavy workloads |
| `opt-level = "s"` | Optimize for binary size | Embedded, WASM |
| `opt-level = "z"` | Aggressively optimize for size | Embedded, WASM |

The difference between `opt-level = 2` and `3` is usually small (single-digit percentages) unless your code does heavy numerical work that benefits from auto-vectorization. Don't cargo-cult `opt-level = 3` - measure it.

## Tool 1: perf - the Linux profiler

`perf` is a sampling profiler built into the Linux kernel. It periodically interrupts your program and records which function is executing. After enough samples, functions that take more CPU time show up more often. It's the foundation that tools like `cargo-flamegraph` build on.

### Installing and running perf

```bash
# Ubuntu/Debian
sudo apt install linux-tools-common linux-tools-$(uname -r)

# Allow perf for non-root users
echo 1 | sudo tee /proc/sys/kernel/perf_event_paranoid
# Or more permissive: echo -1

# Record samples for your binary
perf record -g --call-graph dwarf ./target/release/my_app

# View the report
perf report
```

The `-g --call-graph dwarf` flags tell perf to capture full call stacks using DWARF debug info. Without these, you only see leaf functions - the ones actually on the CPU - but not who called them. That's like knowing which room is on fire but not how the fire got there.

### Reading perf report

`perf report` opens an interactive TUI. You'll see something like:

```
  Overhead  Command   Shared Object      Symbol
+   32.10%  my_app    my_app             [.] my_app::parser::parse_document
+   18.45%  my_app    my_app             [.] alloc::raw_vec::RawVec<T,A>::grow_amortized
+   12.30%  my_app    my_app             [.] core::fmt::write
+    8.20%  my_app    libc.so.6          [.] __memmove_avx_unaligned_erms
+    6.15%  my_app    my_app             [.] my_app::model::Document::clone
```

This tells a story. 32% of CPU time is in `parse_document` - expected, that's the core logic. But 18% is in `grow_amortized` (Vec resizing), 12% in `fmt::write` (formatting), and 8% in `memmove` (memory copying). Those are your leads.

Press Enter on any function to expand its call chain. You'll see which callers are responsible for each function's overhead. That `memmove` might be coming from Vec growth, or from String operations, or from something else entirely. The call chain tells you.

### perf stat for quick overviews

Before doing a full recording, `perf stat` gives you hardware-level numbers:

```bash
perf stat -d ./target/release/my_app
```

```
 Performance counter stats for './target/release/my_app':

         2,431.15 msec  task-clock
              842       context-switches
           12,847       page-faults
    8,234,567,890       cycles
    6,123,456,789       instructions     #  0.74  insn per cycle
      312,456,789       cache-references
       45,678,901       cache-misses     # 14.62% of all cache refs
      789,012,345       branches
       23,456,789       branch-misses    #  2.97% of all branches
```

Key numbers to watch:
- **Instructions per cycle (IPC)**: Healthy is 1.0-4.0. Below 1.0 means the CPU is stalling - likely cache misses or branch mispredictions.
- **Cache miss rate**: Above 10% means your data access patterns are unfriendly. Consider struct layout, data-oriented design.
- **Branch miss rate**: Above 5% means lots of unpredictable branches. Consider branchless algorithms or sorting data to improve prediction.

## Tool 2: cargo-flamegraph - visualizing where time goes

`cargo-flamegraph` wraps `perf` (Linux) or DTrace (macOS) and produces an SVG flamegraph. It's the tool I reach for first when something is slow.

```bash
cargo install flamegraph

# Profile the default binary
cargo flamegraph

# Profile a specific binary or example
cargo flamegraph --bin my_app
cargo flamegraph --example benchmark_scenario

# Profile tests
cargo flamegraph --test integration_tests -- test_name

# Pass arguments to the program
cargo flamegraph -- --input large_file.json
```

`cargo flamegraph` builds in release mode by default and runs the binary under perf. When the program exits, it generates `flamegraph.svg` in your project root.

### How to read a flamegraph

Open the SVG in a browser - it's interactive (you can click to zoom, search with Ctrl+F). Here's what you're looking at:

**The y-axis is stack depth.** Each row is a stack frame. The bottom is `main()`, the top is the function that was actually executing when the sample was taken. Reading bottom-to-top shows you the call chain: main called A, A called B, B called C.

**The x-axis is NOT time.** This trips people up constantly. The x-axis represents the proportion of samples. A wide box means that function (or its children) appeared in many samples. Boxes are sorted alphabetically, not chronologically.

**Width = time spent.** A function that's 30% of the x-axis width was on-CPU (directly or through its callees) for roughly 30% of the program's runtime.

**Look at the top edge.** Functions at the very top of the flame are the ones actually executing (leaf functions). Wide boxes at the top are your direct hotspots. Wide boxes lower in the stack just mean they're common callers - that's expected for things like `main` or `tokio::runtime::Runtime::block_on`.

### Patterns to look for

**Flat tops (plateaus):** A wide box at the top of the flame means a single function is burning a lot of CPU. This is the clearest signal - go look at that function.

```
                 ┌─────────────────────────────────┐
                 │ alloc::raw_vec::RawVec::grow     │  <- plateau: lots of Vec resizing
        ┌────────┴─────────────────────────────────┴────────┐
        │ my_app::ingest::process_batch                      │
   ┌────┴────────────────────────────────────────────────────┴────┐
   │ main                                                         │
```

**Wide callers with many thin children:** A function that's wide but has many narrow children on top is probably a hot loop calling many small functions. The function itself isn't slow - it just gets called a lot.

**Unexpected standard library functions:** If `alloc::`, `core::fmt::`, or `__memmove` show up prominently, you're doing too many allocations, formatting operations, or memory copies. These are almost always fixable.

**Deep stacks:** Very deep flamegraphs (lots of frames) can indicate excessive recursion or deeply nested abstractions. Each frame has overhead.

### Searching the flamegraph

The interactive SVG supports search (Ctrl+F in browser). Search for:
- `alloc::` - to find allocation hotspots
- `clone` - to find expensive clones
- `drop` - to find expensive destructors
- `fmt::` - to find formatting overhead
- Your crate name - to isolate your code from library code

## Tool 3: criterion - statistically rigorous benchmarks

If you've been reading along from my [load testing post](/blog/load-testing-your-rust-api-tools-and-methodology), you know the value of measuring before and after changes. `criterion` does this at the function level. It runs your code thousands of times, applies statistical analysis, and tells you whether a change actually made a difference or is just noise.

### Setup

Add criterion as a dev dependency and configure a benchmark harness:

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.8", features = ["html_reports"] }

[[bench]]
name = "my_benchmarks"
harness = false
```

The `harness = false` tells Cargo not to use the built-in test harness - criterion provides its own.

### Writing benchmarks

Create `benches/my_benchmarks.rs`:

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId, Throughput};

fn parse_document(input: &str) -> Vec<String> {
    input.lines().map(|l| l.to_uppercase()).collect()
}

fn bench_parse(c: &mut Criterion) {
    let small_input = "hello\nworld\n".repeat(10);
    let large_input = "hello\nworld\n".repeat(10_000);

    let mut group = c.benchmark_group("parse_document");

    group.throughput(Throughput::Bytes(small_input.len() as u64));
    group.bench_with_input(
        BenchmarkId::new("small", small_input.len()),
        &small_input,
        |b, input| {
            b.iter(|| parse_document(black_box(input)))
        },
    );

    group.throughput(Throughput::Bytes(large_input.len() as u64));
    group.bench_with_input(
        BenchmarkId::new("large", large_input.len()),
        &large_input,
        |b, input| {
            b.iter(|| parse_document(black_box(input)))
        },
    );

    group.finish();
}

criterion_group!(benches, bench_parse);
criterion_main!(benches);
```

Run with `cargo bench`:

```
parse_document/small/120
                        time:   [1.2345 us 1.2456 us 1.2567 us]
                        thrpt:  [91.12 MiB/s 92.34 MiB/s 93.56 MiB/s]

parse_document/large/120000
                        time:   [1.1234 ms 1.1345 ms 1.1456 ms]
                        thrpt:  [99.87 MiB/s 101.23 MiB/s 102.34 MiB/s]
                        change: [-12.345% -10.234% -8.123%] (p = 0.00 < 0.05)
                        Performance has improved.
```

The three numbers in brackets are the lower bound, estimate, and upper bound of a 95% confidence interval. The `change` line appears when you've run the benchmark before - it compares against the previous run and tells you if the difference is statistically significant.

### Key criterion features

**`black_box()`** prevents the compiler from optimizing away your benchmark. Without it, the compiler might realize the result is unused and skip the computation entirely. Always wrap inputs and sometimes outputs in `black_box()`.

**`Throughput`** lets you measure bytes/second or elements/second instead of just time. Essential for I/O-heavy or parsing benchmarks where you want to compare across different input sizes.

**`measurement_time()`** controls how long criterion spends on each benchmark:

```rust
let mut group = c.benchmark_group("expensive_ops");
group.measurement_time(std::time::Duration::from_secs(10));
group.sample_size(50); // fewer samples if each takes a while
```

**`iter_batched()`** for benchmarks that need fresh state each iteration:

```rust
group.bench_function("drain_vec", |b| {
    b.iter_batched(
        || (0..1000).collect::<Vec<i32>>(), // setup: create fresh vec
        |mut vec| vec.drain(..).sum::<i32>(), // benchmark: consume it
        criterion::BatchSize::SmallInput,
    )
});
```

### #[bench] vs criterion

Rust's nightly-only `#[bench]` attribute is minimal - it gives you nanoseconds per iteration and that's about it. Criterion gives you:
- Statistical analysis with confidence intervals
- Automatic comparison against previous runs
- HTML reports with plots
- Throughput measurement
- Parameterized benchmarks

There's no reason to use `#[bench]` unless you're specifically benchmarking compiler internals on nightly. Use criterion.

## Tool 4: dhat - heap profiling

CPU profiling tells you where time goes. Heap profiling tells you where memory goes. The `dhat` crate wraps the system allocator, tracks every allocation, and generates a report you can view in a browser.

If `alloc::` functions showed up in your flamegraph, dhat is your next step.

### Setup

Add dhat with a feature flag so it doesn't affect normal builds:

```toml
[dependencies]
dhat = { version = "0.3", optional = true }

[features]
dhat-heap = ["dhat"]
```

In your `main.rs`:

```rust
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();

    // ... your normal code
}
```

Run with the feature enabled:

```bash
cargo run --release --features dhat-heap
```

When the program exits, dhat writes a `dhat-heap.json` file. Open it in the [DHAT viewer](https://nnethercote.github.io/dh_view/dh_view.html) (a web-based tool by the same author).

### Reading dhat output

The viewer shows a tree of allocation sites sorted by total bytes allocated. Each entry shows:

- **Total bytes**: How many bytes this call site allocated over the program's lifetime
- **Total blocks**: Number of individual allocations
- **At t-gmax**: Bytes still alive at the point of peak heap usage
- **At t-end**: Bytes still alive at program exit (potential leaks if non-zero)
- **Reads/writes per byte**: How much each allocated byte was actually used

The "reads/writes per byte" metric is gold. An allocation that's barely read after creation is wasted work. Common examples:

```rust
// Bad: allocates a String just to check a condition
if format!("{}", value).contains("error") {
    handle_error();
}

// Better: no allocation
if value.to_string().contains("error") { // still allocates
    handle_error();
}

// Best: no allocation at all
if is_error(value) {
    handle_error();
}
```

### dhat for testing allocation counts

dhat also supports assertions in tests - you can enforce exact allocation counts:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_allocation_count() {
        let _profiler = dhat::Profiler::builder().testing().build();

        let stats = dhat::HeapStats::get();
        let before = stats.total_blocks;

        // Run the code under test
        let result = process_batch(&input);

        let stats = dhat::HeapStats::get();
        let after = stats.total_blocks;

        // This operation should do at most 5 allocations
        assert!(after - before <= 5, "Too many allocations: {}", after - before);
    }
}
```

This is powerful for regressions - if someone accidentally adds allocations to a hot path, the test catches it.

## Common bottlenecks and fixes

After profiling dozens of Rust programs, the same patterns come up repeatedly. Here's what to look for and how to fix each one.

### 1. Unnecessary allocations in hot loops

The number one bottleneck. If your flamegraph shows `alloc::raw_vec::RawVec::grow_amortized` prominently, you're growing Vecs (or Strings, which are `Vec<u8>` underneath) inside a hot loop.

```rust
// Slow: allocates a new Vec every iteration
fn process_items(items: &[Item]) -> Vec<Result> {
    let mut results = Vec::new(); // grows repeatedly
    for item in items {
        let temp: Vec<u8> = item.serialize(); // alloc per iteration
        results.push(parse(&temp));
    }
    results
}

// Fast: pre-allocate, reuse buffers
fn process_items(items: &[Item]) -> Vec<Result> {
    let mut results = Vec::with_capacity(items.len()); // one allocation
    let mut buf = Vec::with_capacity(256); // reusable buffer
    for item in items {
        buf.clear(); // reuse without deallocating
        item.serialize_into(&mut buf);
        results.push(parse(&buf));
    }
    results
}
```

`Vec::with_capacity()` is your best friend. If you know (or can estimate) how many elements you'll need, pre-allocate. The difference can be dramatic - I've seen 3-5x speedups just from this change in allocation-heavy code.

### 2. Excessive cloning

`.clone()` on anything that contains heap data means a new allocation. In complex types with nested Vecs and Strings, a single clone can trigger dozens of allocations.

```rust
// Slow: clones the entire config on every request
fn handle_request(config: &AppConfig, req: Request) -> Response {
    let config = config.clone(); // deep copy of all fields
    process(config, req)
}

// Fast: pass by reference
fn handle_request(config: &AppConfig, req: Request) -> Response {
    process(config, req)
}

// If you need ownership, use Arc
fn handle_request(config: Arc<AppConfig>, req: Request) -> Response {
    let config = Arc::clone(&config); // just increments a counter
    process(config, req)
}
```

Look for `.clone()` in your code and ask: do I actually need a new copy, or can I borrow? If multiple owners are needed, `Arc` (or `Rc` for single-threaded code) gives you shared ownership with just an atomic increment instead of a full copy.

Also worth knowing: `clone_from` can be faster than `clone` when you already have a value of the right type:

```rust
// a = b.clone() drops a, then allocates a new copy of b
// a.clone_from(&b) can reuse a's existing buffer
let mut buffer = String::with_capacity(1024);
// In a loop:
buffer.clone_from(&new_value); // reuses buffer's allocation if it fits
```

### 3. String formatting in hot paths

`format!()` always allocates a new String. In a hot path, this adds up fast.

```rust
// Slow: format! allocates every time
fn log_metric(name: &str, value: f64) {
    let msg = format!("metric.{}: {}", name, value); // allocation
    logger.write(&msg);
}

// Faster: write directly to a buffer
fn log_metric(name: &str, value: f64, buf: &mut String) {
    use std::fmt::Write;
    buf.clear();
    write!(buf, "metric.{}: {}", name, value).unwrap(); // no allocation
    logger.write(buf);
}
```

The `write!` macro writes into an existing buffer. If the buffer has enough capacity, this is allocation-free.

### 4. Collecting iterators unnecessarily

```rust
// Slow: collects into a Vec just to iterate again
let filtered: Vec<&Item> = items.iter()
    .filter(|i| i.is_active())
    .collect();
let total: i64 = filtered.iter().map(|i| i.value).sum();

// Fast: chain the iterators
let total: i64 = items.iter()
    .filter(|i| i.is_active())
    .map(|i| i.value)
    .sum();
```

Every `.collect()` allocates. If you're just going to iterate over the result, skip the collection. Rust's lazy iterators compose without intermediate allocations.

### 5. HashMap with default hasher in hot paths

Rust's default `HashMap` uses SipHash - designed for HashDoS resistance, not speed. If your keys are not user-controlled (no DoS risk), switch to a faster hasher:

```toml
[dependencies]
rustc-hash = "2"
```

```rust
use rustc_hash::FxHashMap;

// Drop-in replacement - same API, ~2x faster for integer and small keys
let mut map: FxHashMap<u64, Value> = FxHashMap::default();
```

[FxHashMap](https://crates.io/crates/rustc-hash) uses the same hash algorithm as the Rust compiler internally. It's not cryptographically secure, but for internal data structures it's significantly faster.

### 6. Syscall overhead

If `perf report` shows significant time in kernel functions (symbols starting with `[k]`), you might be making too many syscalls. Common culprits:

- **Small writes**: Writing 1 byte at a time to a file means one syscall per byte. Use `BufWriter`.
- **Small reads**: Same problem, use `BufReader`.
- **Frequent time checks**: `SystemTime::now()` or `Instant::now()` in a tight loop. Cache the timestamp.

```rust
use std::io::BufWriter;
use std::fs::File;

// Slow: syscall per write
let mut file = File::create("output.txt")?;
for line in data {
    writeln!(file, "{}", line)?; // syscall every time
}

// Fast: buffered, flushes in batches
let mut file = BufWriter::new(File::create("output.txt")?);
for line in data {
    writeln!(file, "{}", line)?; // writes to buffer
}
// flushes remaining data on drop
```

## The workflow: profile, identify, fix, verify

Knowing the tools isn't enough. You need a process. Here's the workflow I follow every time:

### Step 1: Establish a baseline

Before touching any code, get a reproducible benchmark. This is your "before" measurement.

```bash
# Create a benchmark that exercises the slow code path
# benches/hot_path.rs
cargo bench -- hot_path > baseline.txt
```

If you don't have a criterion benchmark yet, at minimum use `hyperfine` for end-to-end timing:

```bash
cargo install hyperfine
hyperfine './target/release/my_app --input test_data.json'
```

`hyperfine` runs the command multiple times and gives you mean, stddev, and min/max. It also warns you about statistical outliers.

### Step 2: Profile

Generate a flamegraph:

```bash
cargo flamegraph -- --input test_data.json
```

Open `flamegraph.svg` in a browser. Look for the patterns described above - wide flat tops, unexpected stdlib functions, allocation frames.

If allocations look suspicious, run dhat:

```bash
cargo run --release --features dhat-heap -- --input test_data.json
# Open dhat-heap.json in the DHAT viewer
```

### Step 3: Form a hypothesis

Based on the profiling data, identify the specific bottleneck. Be precise: "the `parse_line` function is spending 40% of its time in `Vec::push` because it starts with an empty Vec and grows it one element at a time on lines averaging 50 tokens."

Don't skip this step. If you can't articulate what's slow and why, you'll make changes that don't help or make things worse.

### Step 4: Fix it

Make the smallest change that addresses the hypothesis. One change at a time. If you change five things at once, you won't know which one helped (or if they cancelled each other out).

```rust
// Hypothesis: Vec::push in parse_line is causing reallocations
// Fix: pre-allocate based on a reasonable estimate

fn parse_line(line: &str) -> Vec<Token> {
    // Average line has ~50 tokens, allocate for that
    let mut tokens = Vec::with_capacity(64);
    // ... parsing logic
    tokens
}
```

### Step 5: Verify

Run the same benchmark again and compare:

```bash
cargo bench -- hot_path
```

Criterion will automatically compare against the previous run:

```
hot_path/parse_document
                        time:   [1.0123 ms 1.0234 ms 1.0345 ms]
                        change: [-15.234% -13.567% -11.890%] (p = 0.00 < 0.05)
                        Performance has improved.
```

If the change line says "Performance has improved" with p < 0.05, your fix worked. If it says "No change in performance", your hypothesis was wrong. Go back to step 2 and look at the profile again.

Also re-run the flamegraph to confirm the bottleneck is gone and you haven't introduced a new one:

```bash
cargo flamegraph -- --input test_data.json
# Compare visually with the previous flamegraph
```

### Step 6: Repeat

Performance work is iterative. Fixing one bottleneck reveals the next one. Keep profiling until you hit your performance target or reach diminishing returns.

## A real example: the full cycle

Let me walk through a concrete example. Say we have a log processor that reads lines, parses them, and counts patterns:

```rust
use std::collections::HashMap;

fn count_patterns(input: &str) -> HashMap<String, usize> {
    let mut counts = HashMap::new();
    for line in input.lines() {
        let parts: Vec<&str> = line.split_whitespace().collect();
        if parts.len() >= 3 {
            let key = format!("{}:{}", parts[0], parts[2]);
            *counts.entry(key).or_insert(0) += 1;
        }
    }
    counts
}
```

This looks fine at first glance. Let's profile it with a large input.

**Flamegraph reveals:** 35% in `alloc::raw_vec::RawVec::grow_amortized`, 20% in `core::fmt::write`, 15% in `hashbrown::raw::RawTable::resize`.

Three bottlenecks:

1. **Vec allocation on every line** (`split_whitespace().collect()`): We don't need a Vec - we just need the first and third elements.

2. **format! on every line**: `format!("{}:{}", parts[0], parts[2])` allocates a new String for every line.

3. **HashMap resizing**: The map starts empty and resizes as it grows.

Here's the optimized version:

```rust
use std::collections::HashMap;
use std::fmt::Write;

fn count_patterns(input: &str) -> HashMap<String, usize> {
    // Estimate: typical log files have ~1000 unique patterns
    let mut counts = HashMap::with_capacity(1024);
    let mut key_buf = String::with_capacity(128);

    for line in input.lines() {
        let mut parts = line.split_whitespace();
        let first = match parts.next() {
            Some(p) => p,
            None => continue,
        };
        // Skip second element
        if parts.next().is_none() { continue; }
        let third = match parts.next() {
            Some(p) => p,
            None => continue,
        };

        key_buf.clear();
        write!(key_buf, "{}:{}", first, third).unwrap();

        *counts.entry(key_buf.clone()).or_insert(0) += 1;
    }
    counts
}
```

Wait - we still have `key_buf.clone()` on the entry path. We can eliminate even that with the entry API trick:

```rust
use std::collections::HashMap;
use std::fmt::Write;

fn count_patterns(input: &str) -> HashMap<String, usize> {
    let mut counts: HashMap<String, usize> = HashMap::with_capacity(1024);
    let mut key_buf = String::with_capacity(128);

    for line in input.lines() {
        let mut parts = line.split_whitespace();
        let first = match parts.next() {
            Some(p) => p,
            None => continue,
        };
        if parts.next().is_none() { continue; }
        let third = match parts.next() {
            Some(p) => p,
            None => continue,
        };

        key_buf.clear();
        write!(key_buf, "{}:{}", first, third).unwrap();

        // Only clone when we see a NEW key
        if let Some(count) = counts.get_mut(key_buf.as_str()) {
            *count += 1;
        } else {
            counts.insert(key_buf.clone(), 1);
        }
    }
    counts
}
```

Now we only allocate for the `HashMap` key when encountering a genuinely new pattern. For a log file where most lines match existing patterns, this eliminates the vast majority of allocations.

Benchmark result:

```
count_patterns/large_log
                        time:   [12.345 ms 12.567 ms 12.789 ms]
                        change: [-42.3% -40.1% -37.8%] (p = 0.00 < 0.05)
                        Performance has improved.
```

40% faster, just by eliminating unnecessary allocations. The flamegraph confirms: `grow_amortized` dropped from 35% to under 5%, `core::fmt::write` disappeared from the hot path entirely.

## Bonus: samply for interactive profiling

Worth a quick mention: [samply](https://github.com/mstange/samply) is a sampling profiler that feeds into the Firefox Profiler UI. It gives you an interactive web-based view with timeline, call tree, and flamegraph all in one.

```bash
cargo install samply
samply record ./target/release/my_app
```

It opens your browser with a rich profiling interface. If you find SVG flamegraphs limiting, samply's UI offers more ways to slice the data - filter by thread, zoom into time ranges, compare call trees. It works on Linux, macOS, and Windows.

## Bonus: Profile-Guided Optimization (PGO)

Once you've squeezed out algorithmic improvements, there's one more trick: let the compiler learn from your program's actual runtime behavior.

```bash
# Step 1: Build an instrumented binary
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" cargo build --release

# Step 2: Run it with representative workload
./target/release/my_app --input typical_data.json

# Step 3: Merge the profile data
llvm-profdata merge -o /tmp/pgo-data/merged.profdata /tmp/pgo-data

# Step 4: Rebuild with the profile data
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" cargo build --release
```

PGO teaches the compiler which branches are taken most often, which functions are hot, and what call patterns exist. This enables better inlining decisions, branch prediction hints, and code layout. Typical gains are 10-20% on top of an already-optimized release build.

The `cargo-pgo` tool wraps these steps in a friendlier CLI if you prefer:

```bash
cargo install cargo-pgo
cargo pgo build
cargo pgo run -- --input typical_data.json
cargo pgo optimize
```

## Summary

The profiling workflow is simple: measure, identify, fix, verify. The tools map to different questions:

| Question | Tool |
|----------|------|
| Where is CPU time going? | `perf`, `cargo-flamegraph`, `samply` |
| Is this specific change faster? | `criterion` |
| Where are allocations happening? | `dhat` |
| Is the overall binary faster? | `hyperfine` |
| Can the compiler optimize better? | PGO |

The key insight: performance problems are almost never where you think they are. The functions you spent the most time writing aren't necessarily the ones that take the most time running. Profile first, optimize second. Rinse and repeat until you're happy.

If you covered load testing from a systems perspective (covered in my earlier [load testing post](/blog/load-testing-your-rust-api-tools-and-methodology)), profiling is the microscope that complements that telescope. Load testing tells you the *application* is slow. Profiling tells you *why*. And if you need to understand what's happening once the fix hits production, check out my post on [monitoring Rust applications in production](/blog/monitoring-rust-applications-in-production) for the observability side.
