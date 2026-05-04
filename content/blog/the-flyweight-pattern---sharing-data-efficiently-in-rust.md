+++
title = "The flyweight pattern - sharing data efficiently in Rust"
date = 2026-03-06
description = "How Arc<str>, string interning, and Cow<str> let you eliminate redundant allocations when your program is full of duplicate data."

[taxonomies]
tags = ["rust", "design-patterns", "performance", "memory"]
+++

You're parsing a million-line log file. Each line has a log level - `"INFO"`, `"WARN"`, `"ERROR"`. You store them as `String` fields in a struct. That's three unique values, but a million heap allocations. Every single `"INFO".to_string()` allocates 4 bytes on the heap, writes a pointer, a length, and a capacity onto the stack - 24 bytes of metadata for 4 bytes of content. Multiply by 800,000 INFO lines and you've burned ~22 MB on identical strings that could have been one shared allocation.

The flyweight pattern fixes this. Instead of each object owning a private copy of its data, objects share references to a common instance. The Gang of Four described it in 1994 for Java GUIs with thousands of character glyphs. In Rust, the pattern shows up everywhere - but the tools are different. No garbage collector to lean on, no implicit sharing. You pick your sharing primitive explicitly: `Arc<str>`, `Rc<str>`, `Cow<str>`, or a dedicated intern pool. Each has different tradeoffs in memory, performance, and thread safety.

<!-- more -->

## The cost of cloning strings

Before optimizing, you need to know what you're paying. Here's a typical struct that owns all its data:

```rust
struct LogEntry {
    level: String,     // 24 bytes on stack, N bytes on heap
    source: String,    // 24 bytes on stack, N bytes on heap
    message: String,   // 24 bytes on stack, N bytes on heap
}
```

On a 64-bit system, `String` occupies 24 bytes on the stack: an 8-byte pointer to heap data, an 8-byte length, and an 8-byte capacity. The heap allocation holds the actual UTF-8 bytes plus allocator overhead (usually 8-16 bytes for the allocation header, depending on the allocator).

```rust
use std::mem;

fn main() {
    println!("String:  {} bytes", mem::size_of::<String>());   // 24
    println!("&str:    {} bytes", mem::size_of::<&str>());      // 16
    println!("Box<str>: {} bytes", mem::size_of::<Box<str>>()); // 16
}
```

When you clone a `String`, you get a brand new heap allocation with a full copy of the bytes. For our log parser, calling `"INFO".to_string()` 800,000 times means 800,000 separate 4-byte heap allocations. The allocator has to find space, write headers, and hand back pointers - each time for the exact same four bytes.

Here's a quick way to see the impact:

```rust
use std::collections::HashMap;

fn count_unique_strings(entries: &[LogEntry]) -> HashMap<&str, usize> {
    let mut counts = HashMap::new();
    for entry in entries {
        *counts.entry(entry.level.as_str()).or_insert(0) += 1;
    }
    counts
}
```

If this returns `{"INFO": 800000, "WARN": 150000, "ERROR": 50000}`, you have a million allocations for three unique values. That's the problem the flyweight pattern solves.

## Arc<str> - the simplest flyweight

The most direct solution in Rust: replace `String` with `Arc<str>`. Cloning an `Arc` increments a reference count instead of copying heap data. All clones point to the same allocation.

```rust
use std::sync::Arc;

struct LogEntry {
    level: Arc<str>,
    source: Arc<str>,
    message: String, // messages are unique, keep as String
}
```

`Arc<str>` is 16 bytes on the stack - a pointer and a length (it's a fat pointer because `str` is a DST). On the heap, it stores 8 bytes for the strong count, 8 bytes for the weak count, then the actual string bytes. So a single `Arc<str>` holding `"INFO"` costs 16 (stack) + 16 (counts) + 4 (data) = 36 bytes total, plus allocator overhead.

But here's the key: the second clone costs only 16 bytes on the stack and zero new heap allocation. Compare that to a second `String::clone()` which costs 24 bytes on the stack plus another heap allocation.

```rust
fn main() {
    let level: Arc<str> = Arc::from("INFO");

    // All of these point to the same heap allocation
    let a = Arc::clone(&level);
    let b = Arc::clone(&level);
    let c = Arc::clone(&level);

    // Pointer comparison confirms they share data
    assert!(Arc::ptr_eq(&a, &b));
    assert!(Arc::ptr_eq(&b, &c));

    println!("Strong count: {}", Arc::strong_count(&level)); // 4
}
```

For our log parser, pre-create the three level strings and hand out clones:

```rust
use std::sync::Arc;

struct LevelCache {
    info: Arc<str>,
    warn: Arc<str>,
    error: Arc<str>,
}

impl LevelCache {
    fn new() -> Self {
        Self {
            info: Arc::from("INFO"),
            warn: Arc::from("WARN"),
            error: Arc::from("ERROR"),
        }
    }

    fn get(&self, raw: &str) -> Arc<str> {
        match raw {
            "INFO" => Arc::clone(&self.info),
            "WARN" => Arc::clone(&self.warn),
            "ERROR" => Arc::clone(&self.error),
            other => Arc::from(other), // fallback: allocate for unknown levels
        }
    }
}
```

Memory savings: instead of 1,000,000 heap allocations, you have 3. The million `Arc::clone` calls each do one atomic increment - roughly 5-15 nanoseconds depending on contention - versus a `memcpy` plus allocator call for `String::clone`.

## Memory layout under the hood

Let's look at what the compiler actually produces. On x86-64:

```
String "INFO" (24 bytes stack + heap alloc):
  Stack: [ptr: 8 bytes][len: 8 bytes][cap: 8 bytes]
  Heap:  [I][N][F][O] (4 bytes + allocator header)

Arc<str> "INFO" (16 bytes stack + heap alloc):
  Stack: [ptr: 8 bytes][len: 8 bytes]
  Heap:  [strong: 8 bytes][weak: 8 bytes][I][N][F][O]
```

`Arc<str>` saves 8 bytes per instance on the stack (no capacity field - `str` is immutable so capacity is meaningless). The heap layout differs too: `Arc` puts the reference counts inline with the data. This is important for cache locality - when you access the string, the counts are in the same cache line.

You can verify this yourself:

```rust
use std::sync::Arc;
use std::mem;

fn main() {
    println!("Arc<str>:    {} bytes", mem::size_of::<Arc<str>>());    // 16
    println!("Arc<String>: {} bytes", mem::size_of::<Arc<String>>()); // 8
    println!("String:      {} bytes", mem::size_of::<String>());      // 24
}
```

Notice `Arc<String>` is only 8 bytes - a thin pointer, because `String` is `Sized`. But `Arc<String>` adds a layer of indirection: the `Arc` points to heap memory containing the ref counts and a `String`, and that `String` itself points to a second heap allocation with the actual bytes. Two indirections to read the data. `Arc<str>` avoids this - one pointer, one heap allocation, data is right after the counts.

## Building a HashMap-based intern pool

A hardcoded cache works when you know the values upfront. When you don't - think config keys, AST identifiers, HTTP header names - you need a general-purpose intern pool. The idea: a `HashMap` that maps string content to a shared reference. If the string already exists in the pool, return a clone of the existing `Arc`. If not, insert it and return the new `Arc`.

```rust
use std::collections::HashMap;
use std::sync::Arc;

struct InternPool {
    pool: HashMap<Arc<str>, ()>,
}

impl InternPool {
    fn new() -> Self {
        Self {
            pool: HashMap::new(),
        }
    }

    fn intern(&mut self, s: &str) -> Arc<str> {
        if let Some((key, _)) = self.pool.get_key_value(s) {
            Arc::clone(key)
        } else {
            let arc: Arc<str> = Arc::from(s);
            self.pool.insert(Arc::clone(&arc), ());
            arc
        }
    }

    fn len(&self) -> usize {
        self.pool.len()
    }
}
```

A trick here: the `HashMap<Arc<str>, ()>` uses `Arc<str>` as the key. We look up by `&str` (which works because `Arc<str>` implements `Borrow<str>`) and get back the existing `Arc` via `get_key_value`. The value is `()` because we only care about the key's identity.

Usage:

```rust
fn main() {
    let mut pool = InternPool::new();

    let a = pool.intern("content-type");
    let b = pool.intern("content-type"); // reuses existing Arc
    let c = pool.intern("authorization"); // new allocation

    assert!(Arc::ptr_eq(&a, &b)); // same pointer
    println!("Pool size: {}", pool.len()); // 2
}
```

For a parser processing 100,000 identifiers with 500 unique values, this pool turns 100,000 allocations into 500. The `HashMap` lookup cost is O(1) amortized, and hashing short strings is fast - typically under 20 nanoseconds for strings under 64 bytes.

### Thread-safe variant

The pool above is single-threaded. For concurrent access, wrap it in a `Mutex` or use `DashMap`:

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct SharedInternPool {
    inner: Arc<Mutex<HashMap<Arc<str>, ()>>>,
}

impl SharedInternPool {
    fn new() -> Self {
        Self {
            inner: Arc::new(Mutex::new(HashMap::new())),
        }
    }

    fn intern(&self, s: &str) -> Arc<str> {
        let mut pool = self.inner.lock().unwrap();
        if let Some((key, _)) = pool.get_key_value(s) {
            Arc::clone(key)
        } else {
            let arc: Arc<str> = Arc::from(s);
            pool.insert(Arc::clone(&arc), ());
            arc
        }
    }
}
```

This works but the `Mutex` serializes all intern calls. If interning is on the hot path with high contention, consider lock-free alternatives or per-thread pools that merge periodically.

## The string_interner crate

For production use, [`string_interner`](https://crates.io/crates/string-interner) (v0.17 at time of writing) provides a battle-tested intern pool with a different design. Instead of handing back `Arc<str>`, it returns a `Symbol` - a lightweight integer handle that you resolve back to a string when needed.

```rust
use string_interner::StringInterner;

fn main() {
    let mut interner = StringInterner::default();

    let sym_a = interner.get_or_intern("hello");
    let sym_b = interner.get_or_intern("hello");
    let sym_c = interner.get_or_intern("world");

    // Symbols are just integers - comparison is O(1)
    assert_eq!(sym_a, sym_b);
    assert_ne!(sym_a, sym_c);

    // Resolve back to string
    assert_eq!(interner.resolve(sym_a), Some("hello"));

    println!(
        "Symbol size: {} bytes",
        std::mem::size_of_val(&sym_a)
    ); // 4 bytes
}
```

The symbol is 4 bytes. Compare that to `Arc<str>` at 16 bytes or `String` at 24 bytes. If you're storing millions of interned references in structs, the per-field savings add up fast.

The `StringInterner` uses a `StringBackend` by default, which stores all strings contiguously in a single buffer. This means excellent cache locality when resolving symbols - the strings are packed together in memory, not scattered across separate heap allocations like individual `Arc<str>` values would be.

The tradeoff: you need access to the interner to resolve a symbol back to a string. The symbol alone is meaningless. This makes it awkward for APIs where you want to pass around self-contained string values. `Arc<str>` is self-contained - you can resolve it anywhere. A symbol requires the interner to be in scope.

### lasso - concurrent interning

If you need thread-safe interning, [`lasso`](https://crates.io/crates/lasso) (v0.7) offers `ThreadedRodeo`:

```rust
use lasso::{Rodeo, Spur};

fn main() {
    let mut rodeo = Rodeo::default();

    let key: Spur = rodeo.get_or_intern("hello");
    let same_key = rodeo.get_or_intern("hello");

    assert_eq!(key, same_key);
    assert_eq!(rodeo.resolve(&key), "hello");

    // Read-only resolver for sharing across threads
    let reader = rodeo.into_reader();
    assert_eq!(reader.resolve(&key), "hello");
}
```

`lasso`'s `Spur` type is also 4 bytes. The `Rodeo` is for single-threaded building, `ThreadedRodeo` for concurrent building, and `RodeoReader`/`RodeoResolver` for lock-free reads after building is done. This build-then-read pattern fits many real workloads: parse and intern during startup, then resolve during execution.

## Cow<str> - a different kind of flyweight

`Cow<str>` (Clone-on-Write) solves a related but distinct problem. Where `Arc<str>` shares data between multiple owners, `Cow<str>` delays allocation until mutation is needed. It's a flyweight for the common case where data passes through unchanged.

If you read the [Markdown parser post](/blog/writing-a-markdown-parser-in-rust/), you saw pulldown-cmark's `CowStr` - a similar idea specialized for parsing. Most text chunks are just slices of the original input, never modified. `Cow` captures this: borrow when you can, allocate only when you must.

```rust
use std::borrow::Cow;

fn normalize_header(name: &str) -> Cow<'_, str> {
    if name.chars().all(|c| c.is_ascii_lowercase() || c == '-') {
        // Already normalized - return a borrow, zero allocation
        Cow::Borrowed(name)
    } else {
        // Needs work - allocate a new String
        Cow::Owned(name.to_ascii_lowercase())
    }
}

fn main() {
    let a = normalize_header("content-type");  // Borrowed - no alloc
    let b = normalize_header("Content-Type");  // Owned - allocates

    println!("a: {}, b: {}", a, b);

    // Both work identically as &str
    assert_eq!(&*a, "content-type");
    assert_eq!(&*b, "content-type");
}
```

Memory layout of `Cow<str>`:

```rust
use std::borrow::Cow;
use std::mem;

fn main() {
    // Cow<str> is 32 bytes: discriminant + largest variant (String = 24 bytes)
    println!("Cow<str>: {} bytes", mem::size_of::<Cow<'_, str>>());  // 32

    // But Borrowed variant uses only pointer + length internally
    // The capacity field is just padding in the Borrowed case
}
```

`Cow<str>` is 32 bytes on the stack - larger than both `String` (24) and `Arc<str>` (16). The enum needs space for the largest variant (`Owned(String)`) plus the discriminant. So `Cow` is not about saving stack space. It's about saving heap allocations in the common case where the data doesn't need modification.

When does this matter? Config parsing is a good example. You read a TOML file, most values are used as-is. Some need environment variable expansion:

```rust
use std::borrow::Cow;

fn expand_env(value: &str) -> Cow<'_, str> {
    if !value.contains("${") {
        return Cow::Borrowed(value);
    }

    let mut result = String::with_capacity(value.len());
    let mut chars = value.chars().peekable();

    while let Some(c) = chars.next() {
        if c == '$' && chars.peek() == Some(&'{') {
            chars.next(); // skip '{'
            let var_name: String = chars.by_ref().take_while(|&c| c != '}').collect();
            if let Ok(val) = std::env::var(&var_name) {
                result.push_str(&val);
            } else {
                result.push_str("${");
                result.push_str(&var_name);
                result.push('}');
            }
        } else {
            result.push(c);
        }
    }

    Cow::Owned(result)
}
```

If 90% of config values have no `${}` tokens, 90% of calls return a zero-cost borrow. Only the 10% that need expansion allocate.

## Rc vs Arc vs Clone - choosing the right tool

Single-threaded code doesn't need atomic reference counting. `Rc<str>` is the same concept as `Arc<str>` but uses non-atomic operations, making clone/drop faster:

```rust
use std::rc::Rc;
use std::sync::Arc;
use std::mem;

fn main() {
    println!("Rc<str>:  {} bytes", mem::size_of::<Rc<str>>());   // 16
    println!("Arc<str>: {} bytes", mem::size_of::<Arc<str>>());  // 16
    println!("String:   {} bytes", mem::size_of::<String>());    // 24
}
```

Same size on the stack. The difference is in the cost of cloning:

- **`String::clone`**: allocates new heap memory, copies all bytes. Cost scales with string length.
- **`Rc::clone`**: increments a `usize` counter. Constant time. Not thread-safe.
- **`Arc::clone`**: increments an `AtomicUsize` counter. Constant time. Thread-safe but has the cost of an atomic operation.

The atomic increment in `Arc::clone` is not free. On x86-64, it compiles to a `lock xadd` instruction which forces a cache line synchronization across cores. In tight loops with no contention, `Rc::clone` can be 2-5x faster than `Arc::clone`. But "2-5x faster" on a 5-nanosecond operation is still negligible compared to the cost of a heap allocation (typically 20-80 nanoseconds plus the `memcpy`).

Rule of thumb:

| Scenario | Use |
|---|---|
| Data shared across threads | `Arc<str>` |
| Data shared within one thread | `Rc<str>` |
| Data always unique or rarely cloned | `String` |
| Data usually borrowed, sometimes owned | `Cow<str>` |
| Millions of references, need compact handles | `string_interner` / `lasso` |

## Real-world flyweight: AST nodes

Compilers and interpreters are the classic flyweight use case. An AST for a large source file might have thousands of identifier nodes, but the actual unique identifiers - variable names, function names, type names - number in the hundreds.

```rust
use std::collections::HashMap;
use std::sync::Arc;

/// Simple token types for demonstration
#[derive(Debug, Clone)]
enum Token {
    Ident(Arc<str>),
    StringLit(Arc<str>),
    Number(f64),
    LParen,
    RParen,
    Eq,
}

struct Lexer {
    pool: HashMap<Arc<str>, ()>,
}

impl Lexer {
    fn new() -> Self {
        Self {
            pool: HashMap::new(),
        }
    }

    fn intern(&mut self, s: &str) -> Arc<str> {
        if let Some((key, _)) = self.pool.get_key_value(s) {
            Arc::clone(key)
        } else {
            let arc: Arc<str> = Arc::from(s);
            self.pool.insert(Arc::clone(&arc), ());
            arc
        }
    }

    fn tokenize(&mut self, input: &str) -> Vec<Token> {
        let mut tokens = Vec::new();
        let mut chars = input.chars().peekable();

        while let Some(&c) = chars.peek() {
            match c {
                ' ' | '\n' | '\t' => { chars.next(); }
                '(' => { chars.next(); tokens.push(Token::LParen); }
                ')' => { chars.next(); tokens.push(Token::RParen); }
                '=' => { chars.next(); tokens.push(Token::Eq); }
                '"' => {
                    chars.next();
                    let s: String = chars.by_ref().take_while(|&c| c != '"').collect();
                    tokens.push(Token::StringLit(self.intern(&s)));
                }
                c if c.is_ascii_digit() => {
                    let num: String = std::iter::once(c)
                        .chain(
                            std::iter::from_fn(|| {
                                chars.peek()
                                    .filter(|c| c.is_ascii_digit() || **c == '.')
                                    .map(|_| chars.next().unwrap())
                            })
                        )
                        .collect();
                    chars.next(); // consumed first char
                    if let Ok(n) = num.parse::<f64>() {
                        tokens.push(Token::Number(n));
                    }
                }
                c if c.is_ascii_alphabetic() || c == '_' => {
                    let ident: String = std::iter::from_fn(|| {
                        chars.peek()
                            .filter(|c| c.is_ascii_alphanumeric() || **c == '_')
                            .map(|_| chars.next().unwrap())
                    })
                    .collect();
                    tokens.push(Token::Ident(self.intern(&ident)));
                }
                _ => { chars.next(); }
            }
        }

        tokens
    }
}
```

In a source file with 200 uses of `x` and 150 uses of `result`, the intern pool stores two heap allocations. Every `Token::Ident` for `x` points to the same `Arc<str>`. This also makes identifier comparison O(1) - you can compare `Arc` pointers instead of comparing string contents byte by byte:

```rust
fn same_ident(a: &Arc<str>, b: &Arc<str>) -> bool {
    // Pointer comparison: O(1), no string comparison needed
    Arc::ptr_eq(a, b)
}
```

## Beyond strings: flyweight for any shared data

The pattern isn't limited to strings. Any immutable, frequently-duplicated data benefits. Consider a game with thousands of entities sharing a handful of sprite configurations:

```rust
use std::sync::Arc;

#[derive(Debug)]
struct SpriteData {
    texture_id: u32,
    width: u32,
    height: u32,
    frames: Vec<(u32, u32)>, // frame offsets
}

struct Entity {
    sprite: Arc<SpriteData>,  // shared, not cloned
    x: f32,
    y: f32,
    health: i32,
}

fn spawn_enemies(sprite: &Arc<SpriteData>, count: usize) -> Vec<Entity> {
    (0..count)
        .map(|i| Entity {
            sprite: Arc::clone(sprite),
            x: (i * 32) as f32,
            y: 0.0,
            health: 100,
        })
        .collect()
}
```

A thousand enemies, one `SpriteData` allocation. The `Vec<(u32, u32)>` inside `SpriteData` might hold 60 frames of animation data - without sharing, that's 1000 separate `Vec` allocations containing identical data.

## When the flyweight pattern hurts

Not everything should be shared. There are real costs:

**Atomic overhead on hot paths.** If you're cloning and dropping `Arc` millions of times per second in a tight loop, the atomic operations add up. Profile before assuming it's free. In single-threaded code, use `Rc` to avoid the atomic tax.

**Lifetime complexity with Cow.** `Cow<'a, str>` carries a lifetime. That lifetime propagates through every struct that holds it. If your struct needs to be `'static` (common for async tasks, thread spawning, or storing in long-lived collections), `Cow` becomes awkward and you'll end up calling `.into_owned()` everywhere - defeating the purpose.

**Indirection for small data.** Sharing a 4-byte string through `Arc<str>` costs 16 bytes on the stack plus 20 bytes on the heap (counts + data). Owning it as a `String` costs 24 bytes on the stack plus 4 bytes on the heap. For a single instance, `Arc` is actually *more* expensive. The break-even point is around 2-3 clones, depending on string length.

**Memory leaks in unbounded pools.** An intern pool that grows forever is a memory leak. If you're interning user-provided strings (HTTP headers, query parameters), the pool grows with every unique input. Add eviction or use a bounded pool. The `Arc`-based pool at least lets you detect unused entries by checking `Arc::strong_count`, but implementing LRU eviction on an intern pool adds complexity.

**Harder to reason about mutations.** Shared data is immutable by design. If you later need to mutate a value, you have to either clone it out of the `Arc` (losing the sharing benefit) or redesign with interior mutability. Think about your mutation patterns before committing to flyweight.

## The decision tree

Start with `String`. Seriously. Rust's allocator is fast, and for most programs, the overhead of duplicate strings is noise. Only reach for flyweight when you've measured a problem:

1. **Profile first.** Use `dhat` or `heaptrack` to measure allocation counts and total heap usage. If you see millions of small allocations for the same values, you have a flyweight opportunity.

2. **Few unique values, many references?** Classic flyweight territory. Use `Arc<str>` for simple cases, `string_interner` or `lasso` when you need compact symbols.

3. **Data passes through mostly unchanged?** `Cow<str>` is your tool. Zero-cost borrow in the common case, allocation only when needed.

4. **Single-threaded?** Use `Rc<str>` instead of `Arc<str>`. Same API, no atomic overhead.

5. **Cross-thread sharing?** `Arc<str>` is the natural choice. For concurrent interning, use `lasso`'s `ThreadedRodeo`.

The flyweight pattern is one of those optimizations that's almost invisible when done right. Your structs get slightly different field types, your constructors go through a pool, and your memory usage drops by an order of magnitude. The code reads nearly the same. The allocator does a fraction of the work. And the pattern composes well with everything else - you can put `Arc<str>` behind a trait boundary (as covered in the [adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/)), store flyweight handles in ECS components, or use interned symbols as `HashMap` keys with faster lookups.

The best part about doing this in Rust: the type system tells you exactly what's shared. `Arc<str>` in a struct signature is a declaration: "this data is shared and immutable." No hidden reference counting, no surprise clones. You see the sharing, you control the sharing, and the compiler enforces the immutability.
