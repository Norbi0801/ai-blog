+++
title = "Iterator adaptors you should know in Rust"
date = 2025-11-11
description = "Beyond map/filter/collect - the iterator adaptors that make Rust code concise, expressive, and still zero-cost."

[taxonomies]
tags = ["rust", "iterators", "performance", "standard-library"]
+++

Most Rust developers pick up `map`, `filter`, and `collect` in their first week. These three get you far. But the `Iterator` trait in std has [over 70 methods](https://doc.rust-lang.org/std/iter/trait.Iterator.html), and a lot of them solve problems that people keep reimplementing with manual loops and mutable accumulators. This post covers the iterator adaptors I reach for regularly - the ones that eliminate entire categories of boilerplate once you know they exist.

If you're unfamiliar with how closures work in Rust - capture modes, the `Fn`/`FnMut`/`FnOnce` hierarchy, why closures are zero-sized structs - I wrote about that in [Closures in Rust - Fn, FnMut, FnOnce demystified](/blog/closures-in-rust-fn-fnmut-fnonce-demystified). Iterator adaptors lean on closures heavily, so that background helps.

<!-- more -->

## Nothing happens until you consume

Before getting into specific adaptors, there's one fundamental thing to internalize: iterator adaptors are lazy. Calling `.map()` or `.filter()` doesn't iterate over anything. It constructs a new struct that wraps the original iterator and describes the transformation. No work is performed until something calls `.next()` on the chain - usually via `collect`, `for_each`, `sum`, `count`, or a `for` loop.

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    // This does absolutely nothing. No closures execute.
    let _lazy = v.iter().map(|x| {
        println!("processing {x}");
        x * 2
    });

    // The compiler even warns you:
    // "unused `Map` that must be used - iterators are lazy and do nothing unless consumed"
}
```

This isn't a quirk - it's a design decision. Laziness means the compiler can fuse your entire chain into a single pass. A chain like `.filter().map().take(5)` doesn't create intermediate collections. It processes one element at a time through the full pipeline, and it stops after 5 matches. You get the composability of functional style with the performance of a hand-written loop.

If you want to see what the compiler actually generates for an adaptor, look at the [source in `core::iter`](https://github.com/rust-lang/rust/blob/main/library/core/src/iter/traits/iterator.rs). Each adaptor is a struct that holds the original iterator plus any state (a closure, a counter, a flag). Its `next()` method calls the inner iterator's `next()` and applies the transformation. The compiler inlines all of it.

## The adaptors

### chain - glue two iterators together

`chain` takes two iterators and produces a single iterator that yields all elements from the first, then all from the second. Both must yield the same type.

```rust
fn main() {
    let defaults = vec!["--verbose", "--color=auto"];
    let user_args: Vec<&str> = std::env::args().skip(1).collect::<Vec<_>>()
        .iter().map(|s| s.as_str()).collect();

    // In practice, you'd work with owned Strings, but the idea stands:
    let all_flags: Vec<&str> = defaults.iter().copied()
        .chain(user_args.iter().copied())
        .collect();

    println!("{all_flags:?}");
}
```

A more common real-world use: prepending a header to a stream of records, or merging a config file's entries with environment overrides. `chain` also works nicely with `std::iter::once` to tack a single element onto a sequence:

```rust
let with_header = std::iter::once("name,email")
    .chain(rows.iter().map(|r| r.as_str()));
```

### zip - pair up two sequences element-by-element

`zip` walks two iterators in lockstep, yielding tuples. It stops when the shorter one runs out.

```rust
fn main() {
    let keys = ["host", "port", "db"];
    let values = ["localhost", "5432", "myapp"];

    let config: std::collections::HashMap<&str, &str> = keys.iter()
        .zip(values.iter())
        .map(|(&k, &v)| (k, v))
        .collect();

    assert_eq!(config["port"], "5432");
}
```

`zip` is also the tool for comparing two sequences pairwise. Need to check if two slices are "close enough" element-wise?

```rust
fn approx_equal(a: &[f64], b: &[f64], epsilon: f64) -> bool {
    a.len() == b.len()
        && a.iter().zip(b.iter()).all(|(x, y)| (x - y).abs() < epsilon)
}
```

### enumerate - index alongside value

You know this one, but it's worth noting that `enumerate` yields `(usize, T)` tuples, and it's almost always better than maintaining a manual counter.

```rust
fn find_first_negative(values: &[i32]) -> Option<usize> {
    values.iter()
        .enumerate()
        .find(|(_, &v)| v < 0)
        .map(|(i, _)| i)
}
```

One thing people miss: `enumerate` works on any iterator, not just slices. You can enumerate lines from a file, packets from a socket, or results from a database query. The counter is just a `usize` that increments in `next()` - [here's the implementation](https://github.com/rust-lang/rust/blob/main/library/core/src/iter/adapters/enumerate.rs), it's about 20 lines.

### take and skip - slicing without slicing

`take(n)` yields the first `n` elements, then stops. `skip(n)` discards the first `n` and yields the rest. Together, they're pagination for iterators.

```rust
fn paginate<T>(items: &[T], page: usize, per_page: usize) -> &[T] {
    // For slices, you'd just index. But for arbitrary iterators:
    // items.iter().skip(page * per_page).take(per_page)
    let start = page * per_page;
    let end = (start + per_page).min(items.len());
    &items[start..end]
}

// With iterators, it works on anything - files, network streams, generators:
fn first_10_primes() -> Vec<u64> {
    Primes::new().take(10).collect()
}
```

There's also `take_while` and `skip_while` which accept predicates instead of counts:

```rust
fn main() {
    let log_line = "2026-05-21 INFO server started on :8080";

    // Skip the timestamp, take until the next space
    let level: String = log_line.chars()
        .skip_while(|c| *c != ' ')
        .skip(1)
        .take_while(|c| *c != ' ')
        .collect();

    assert_eq!(level, "INFO");
}
```

### flat_map - flatten nested structures in one step

`flat_map` applies a function that returns an iterator, then flattens the results into a single sequence. It's `map` followed by `flatten`, but in one call.

```rust
fn main() {
    let sentences = vec![
        "the quick brown fox",
        "jumps over the lazy dog",
    ];

    let words: Vec<&str> = sentences.iter()
        .flat_map(|s| s.split_whitespace())
        .collect();

    assert_eq!(words.len(), 9);
}
```

This is incredibly useful when you have a list of things that each contain a list of other things. Users with multiple email addresses, directories with files, JSON arrays nested inside objects. Instead of a nested loop:

```rust
// Without flat_map
let mut all_addrs = Vec::new();
for user in &users {
    for addr in &user.addresses {
        all_addrs.push(addr);
    }
}

// With flat_map
let all_addrs: Vec<_> = users.iter()
    .flat_map(|u| u.addresses.iter())
    .collect();
```

`flat_map` also pairs well with `Option`. Since `Option` implements `IntoIterator` (yielding 0 or 1 elements), you can use `flat_map` as a filter-map in one shot:

```rust
let parsed: Vec<i32> = strings.iter()
    .flat_map(|s| s.parse::<i32>())
    .collect();
```

This silently drops unparseable strings. (There's also `.filter_map()` which does the same thing more explicitly with `Option` return types - use whichever reads better in context.)

### scan - fold with intermediate results

`scan` is like `fold`, but instead of producing a single final value, it yields a value at each step. It carries mutable state across iterations.

```rust
fn main() {
    let transactions = vec![100, -30, -20, 50, -80];

    let running_balance: Vec<i32> = transactions.iter()
        .scan(0i32, |balance, &tx| {
            *balance += tx;
            Some(*balance)
        })
        .collect();

    assert_eq!(running_balance, vec![100, 70, 50, 100, 20]);
}
```

The closure receives `&mut State` and the current element, and returns `Option<OutputType>`. Returning `None` stops the iteration early. This makes `scan` useful for "take while accumulating" patterns:

```rust
// Take items from a stream as long as we're under a byte budget
let selected: Vec<&str> = messages.iter()
    .scan(0usize, |total_bytes, msg| {
        *total_bytes += msg.len();
        if *total_bytes > 1024 {
            None // stop - budget exceeded
        } else {
            Some(*msg)
        }
    })
    .collect();
```

### peekable - look ahead without consuming

`peekable()` wraps an iterator so you can call `.peek()` to see the next element without advancing. This is essential for any kind of parsing or lookahead logic.

```rust
fn tokenize_numbers(input: &str) -> Vec<String> {
    let mut chars = input.chars().peekable();
    let mut tokens = Vec::new();

    while let Some(&c) = chars.peek() {
        if c.is_ascii_digit() {
            let mut num = String::new();
            while let Some(&d) = chars.peek() {
                if d.is_ascii_digit() {
                    num.push(d);
                    chars.next();
                } else {
                    break;
                }
            }
            tokens.push(num);
        } else {
            chars.next(); // skip non-digit
        }
    }
    tokens
}

fn main() {
    let result = tokenize_numbers("abc123def456");
    assert_eq!(result, vec!["123", "456"]);
}
```

Under the hood, `Peekable` stores an `Option<Item>` field. When you call `peek()`, it calls the inner iterator's `next()` once and caches the result. The next call to `next()` on the `Peekable` returns the cached value instead of advancing again. Simple, but it changes what's possible with a forward-only iterator.

There's also `peek_mut()` if you need to modify the peeked value before consuming it, and `next_if()` / `next_if_eq()` for conditional consumption - these were stabilized in Rust 1.51 and they're great for parsers.

### windows and chunks - sliding and grouping over slices

These are methods on slices rather than the `Iterator` trait, but they return iterators and they solve problems people keep solving with manual indexing.

`windows(n)` gives you overlapping sub-slices of length `n`:

```rust
fn main() {
    let temps = [18.5, 19.2, 21.0, 20.3, 22.1, 19.8];

    // Moving average over 3 readings
    let moving_avg: Vec<f64> = temps.windows(3)
        .map(|w| w.iter().sum::<f64>() / w.len() as f64)
        .collect();

    // [19.57, 20.17, 21.13, 20.73]
    println!("{moving_avg:.2?}");
}
```

`chunks(n)` gives you non-overlapping groups:

```rust
fn main() {
    let pixels: Vec<u8> = vec![255, 0, 0, 0, 255, 0, 0, 0, 255]; // RGB

    for (i, rgb) in pixels.chunks(3).enumerate() {
        println!("pixel {i}: R={} G={} B={}", rgb[0], rgb[1], rgb[2]);
    }
}
```

If you want compile-time guarantees on chunk size, use `chunks_exact` (panics if not evenly divisible) or `array_chunks` (returns `&[T; N]` instead of `&[T]`, nightly-only as of writing). For `windows`, there's `array_windows` which is also still unstable.

### intersperse - insert separators

`intersperse` places a value between every pair of elements. It's the iterator equivalent of `join`. As of Rust 1.79+, this is still behind the `iter_intersperse` feature gate - track [issue #79524](https://github.com/rust-lang/rust/issues/79524) for stabilization. But the [`itertools`](https://crates.io/crates/itertools) crate (v0.14) has had it forever:

```rust
use itertools::Itertools;

fn main() {
    let parts = vec!["usr", "local", "bin"];

    let path: String = parts.iter()
        .copied()
        .intersperse("/")
        .collect();

    assert_eq!(path, "usr/local/bin");
}
```

"But that's just `join`!" - yes, and `join` exists on slices. The difference is that `intersperse` works on any iterator, including lazy ones. You can intersperse newlines between lines streaming from a file without collecting into a `Vec` first.

There's also `intersperse_with` which takes a closure, useful when the separator needs to be freshly allocated each time (e.g., commas that are `String` rather than `&str`).

## Custom iterators

All the adaptors above come for free once you implement one method: `next`. The `Iterator` trait has a single required method - everything else is a default method built on top of it.

If you've read about [trait bounds and associated types](/blog/rust-trait-bounds), you know how this works. `Iterator` has an associated type `Item`, and `next` returns `Option<Self::Item>`. Here's a practical example - a Fibonacci iterator:

```rust
struct Fibonacci {
    a: u64,
    b: u64,
}

impl Fibonacci {
    fn new() -> Self {
        Fibonacci { a: 0, b: 1 }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        let value = self.a;
        self.a = self.b;
        self.b = value + self.b;
        Some(value) // infinite iterator - never returns None
    }
}

fn main() {
    // Every adaptor now works on Fibonacci
    let first_10: Vec<u64> = Fibonacci::new().take(10).collect();
    assert_eq!(first_10, vec![0, 1, 1, 2, 3, 5, 8, 13, 21, 34]);

    // First Fibonacci number above 1000
    let big = Fibonacci::new()
        .find(|&n| n > 1000)
        .unwrap();
    assert_eq!(big, 1597);

    // Sum of even Fibonacci numbers below 4 million (Project Euler #2)
    let sum: u64 = Fibonacci::new()
        .take_while(|&n| n < 4_000_000)
        .filter(|n| n % 2 == 0)
        .sum();
    assert_eq!(sum, 4613732);
}
```

You implement `next`, and you get `map`, `filter`, `take`, `zip`, `chain`, `enumerate`, `flat_map`, `scan`, `peekable`, `sum`, `count`, `find`, `any`, `all`, `min`, `max`, `nth`, `fold`, `for_each`, and dozens more. That's the payoff of Rust's trait system - one method implementation gives you an entire API.

For returning iterators from functions, use `impl Iterator<Item = T>` to avoid naming the concrete type (which is usually some nested `Map<Filter<Chain<...>>>` monster):

```rust
fn even_squares(limit: u64) -> impl Iterator<Item = u64> {
    (0..limit)
        .map(|n| n * n)
        .filter(|n| n % 2 == 0)
}
```

If you need dynamic dispatch (e.g., returning different iterator types from match arms), box it: `Box<dyn Iterator<Item = T>>`. I covered the static vs dynamic dispatch tradeoff in [The strategy pattern in Rust](/blog/the-strategy-pattern-in-rust) - the same reasoning applies here.

## Performance - iterators vs for loops

New Rust developers worry that iterator chains are slower than hand-written `for` loops. They're not. The Rust Book's [performance chapter](https://doc.rust-lang.org/book/ch13-04-performance.html) shows the benchmark: an audio decoder using iterators clocked 19,234,900 ns/iter vs 19,620,300 ns/iter for the loop version. The iterator version was actually faster by a hair, though the difference is noise.

The reason is straightforward. The compiler monomorphizes each adaptor (creates a specialized version for your concrete types), inlines the closure bodies, and the optimizer fuses the entire chain into a single loop. You can verify this yourself on [Compiler Explorer](https://rust.godbolt.org/). Take this code:

```rust
pub fn sum_of_squares(v: &[i32]) -> i32 {
    v.iter().map(|x| x * x).sum()
}
```

And compare it to:

```rust
pub fn sum_of_squares_loop(v: &[i32]) -> i32 {
    let mut total = 0;
    for x in v {
        total += x * x;
    }
    total
}
```

With `-O` (release mode), they produce identical assembly. The `Map` struct, the `Sum` implementation, the closure - all gone. What remains is a tight loop with a multiply and an add. This is what "zero-cost abstraction" means: you don't pay for the abstraction layer at runtime. The cost is entirely at compile time, where the compiler does the work of fusing everything together.

There's one caveat worth knowing, and [Nicole Tietz-Sokolskaya wrote about it well](https://ntietz.com/blog/rusts-iterators-optimize-footgun/): if your iterator chain crosses a function boundary without inlining, the optimizer can't fuse it. This usually doesn't matter because the compiler is aggressive about inlining small functions. But if you're writing a hot loop in a library, consider `#[inline]` on functions that return `impl Iterator`.

## When to reach for these

A rule of thumb: if you find yourself writing a `for` loop with a mutable accumulator, an index variable, or a boolean flag, check whether an iterator adaptor already does what you want. `enumerate` replaces manual indices. `take_while` replaces boolean flags. `scan` replaces mutable accumulators. `zip` replaces parallel indexing.

The goal isn't to chain as many adaptors as possible - I talked about readability tradeoffs in [Writing Rust that reads like pseudocode](/blog/writing-rust-that-reads-like-pseudocode). A 10-adaptor chain that nobody can parse is worse than a 5-line loop. But a well-chosen adaptor often communicates intent better than the loop it replaces. When you see `.take_while(|t| t.is_valid())`, you know immediately: "we're consuming elements until one fails validation." The equivalent loop buries that intent in control flow.

Explore the [full Iterator docs](https://doc.rust-lang.org/std/iter/trait.Iterator.html). Spend 20 minutes reading through the method list. You'll find things like `step_by`, `inspect` (for debugging mid-chain), `unzip` (the inverse of `zip`), `partition`, and `fold` that solve specific problems elegantly. And if you want even more, the [`itertools`](https://crates.io/crates/itertools) crate adds `group_by`, `tuple_windows`, `join`, `sorted`, `dedup`, `unique`, and many others that complement the standard library well.
