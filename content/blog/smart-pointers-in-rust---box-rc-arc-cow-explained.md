+++
title = "Smart pointers in Rust - Box, Rc, Arc, Cow explained"
date = 2026-02-11
description = "When to use Box, Rc, Arc, Cow, and Weak - with memory layouts, interior mutability, and a decision tree for picking the right pointer."

[taxonomies]
tags = ["rust", "memory", "smart-pointers", "concurrency"]
+++

Every value in Rust lives on the stack by default. The compiler knows the size, inserts a stack adjustment, and the value dies when the frame unwinds. No allocator, no indirection, no cost. But the stack has limits. It can't hold dynamically-sized data. It can't share ownership between multiple parts of your program. It can't represent recursive data structures. Smart pointers fill every one of those gaps.

If you've written Rust for more than a week, you've used `Box<T>`. If you've shared state across threads, you've reached for `Arc<T>`. But most Rust developers pick these types through trial and error - the compiler yells, you try a different wrapper, it compiles, you move on. This post is about understanding what each smart pointer actually does at the memory level, so you can pick the right one before the compiler forces your hand.

<!-- more -->

## What makes a pointer "smart"

A raw pointer (`*const T`, `*mut T`) is just a memory address. A reference (`&T`, `&mut T`) is a pointer with compile-time borrowing rules. A smart pointer is a struct that behaves like a pointer - it implements `Deref` so you can use `.` on it, and it implements `Drop` so it cleans up when it goes out of scope. The "smart" part is the automatic resource management.

All four types we'll cover - `Box`, `Rc`, `Arc`, `Cow` - implement `Deref<Target = T>`, meaning you can call methods on the inner `T` directly. But they each solve a different problem.

| Pointer | What it solves | Thread-safe | Clone cost |
|---------|---------------|------------|------------|
| `Box<T>` | Heap allocation, fixed-size indirection | Yes (if T: Send) | Full deep copy |
| `Rc<T>` | Shared ownership, single thread | No | Increment a counter |
| `Arc<T>` | Shared ownership, multi-thread | Yes | Atomic increment |
| `Cow<'a, B>` | Clone-on-write, defer allocation | Depends on B | Free if borrowed, full copy if mutated |

## Box - putting things on the heap

`Box<T>` is the simplest smart pointer. It allocates `T` on the heap and stores a pointer on the stack. When the `Box` goes out of scope, it frees the heap memory. That's the entire contract.

```rust
fn main() {
    let x = Box::new(42_i32);
    println!("value: {}", x); // Deref kicks in, prints 42
    // x dropped here, heap memory freed
}
```

On a 64-bit system, `Box<i32>` is 8 bytes on the stack (one pointer) and 4 bytes on the heap (the `i32`). Compare that to a plain `i32` which is 4 bytes on the stack and zero heap. You're paying 8 bytes of indirection overhead for something you could've kept on the stack. Why would you do that?

Three reasons.

### Reason 1: Recursive types

The compiler needs to know the size of every type at compile time. A recursive struct has infinite size:

```rust
// This won't compile - infinite size
enum List {
    Cons(i32, List),
    Nil,
}
```

The compiler can't lay out `List` because `Cons` contains another `List`, which contains another `List`, forever. `Box` breaks the recursion by replacing the inline value with a fixed-size pointer:

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    let list = List::Cons(1,
        Box::new(List::Cons(2,
            Box::new(List::Cons(3,
                Box::new(List::Nil))))));
}
```

Now `Cons` is `size_of::<i32>() + size_of::<Box<List>>()` = 4 + 8 = 12 bytes (plus alignment padding). The heap allocation holds the next node. This is how you build linked lists, trees, and any self-referential structure in Rust.

The actual memory layout for this three-element list:

```
Stack:           Heap:
+----------+     +---+----------+     +---+----------+     +---+------+
| ptr ------+--->| 1 | ptr ------+--->| 2 | ptr ------+--->| 3 | Nil  |
+----------+     +---+----------+     +---+----------+     +---+------+
  8 bytes          12 bytes             12 bytes             12 bytes
```

Each `Box::new()` is a separate heap allocation. That's three allocations for three elements. Compare to `Vec<i32>`, which stores all three values in one contiguous allocation. This is why linked lists in Rust are mostly pedagogical - `Vec` beats them on cache locality, allocation count, and memory overhead.

### Reason 2: Trait objects

When you need dynamic dispatch - a value whose concrete type isn't known at compile time - `Box<dyn Trait>` is the standard tool. I covered this in the context of [plugin systems](/blog/implementing-a-plugin-system-in-rust/) and [closures](/blog/closures-in-rust-fn-fnmut-fnonce-demystified/), but here's the memory angle.

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle { radius: f64 }
struct Rect { width: f64, height: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

fn largest_area(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).fold(0.0_f64, f64::max)
}
```

`Box<dyn Shape>` is a fat pointer: 16 bytes on the stack. The first 8 bytes point to the data on the heap. The second 8 bytes point to a vtable - a static table of function pointers for the `Shape` trait methods. When you call `s.area()`, the runtime looks up the `area` function pointer in the vtable and calls it. This is the same mechanism as virtual method dispatch in C++.

```
Stack (Box<dyn Shape>):      Heap:              Static:
+----------+----------+      +--------+         vtable:
| data_ptr | vtbl_ptr-+----->| area() |         +--------+
+----+-----+----------+      | drop() |         | radius |
     |                       | size   |         +--------+
     +---------------------->| align  |
                             +--------+
```

### Reason 3: Large values

Moving a 10KB struct between functions copies 10KB of bytes on every move. Boxing it means you move 8 bytes (the pointer) and the data stays put:

```rust
struct BigConfig {
    data: [u8; 10240], // 10KB on the stack
}

// Moving config copies 10KB each time
fn process(config: BigConfig) { /* ... */ }

// Moving boxed config copies 8 bytes (the pointer)
fn process_boxed(config: Box<BigConfig>) { /* ... */ }
```

The compiler may optimize some of these moves away (copy elision), but Boxing guarantees the data doesn't move. This matters when you're passing large structs through multiple function calls or storing them in collections.

### What Box looks like inside

Looking at the [actual source code](https://github.com/rust-lang/rust/blob/main/library/alloc/src/boxed.rs):

```rust
pub struct Box<T: ?Sized, A: Allocator = Global>(Unique<T>, A);
```

`Unique<T>` is an internal wrapper around `NonNull<T>` that tells the compiler: this pointer is the sole owner of the allocation. The `A: Allocator` parameter lets you use custom allocators (still unstable as of Rust 1.94). For all practical purposes, `Box<T>` is a `NonNull<T>` that calls `dealloc` on drop.

## Rc - shared ownership on a single thread

`Box` has one owner. When you need multiple parts of your code to own the same value, you need reference counting. `Rc<T>` (Reference Counted) tracks how many `Rc` pointers exist for a value. When the last one drops, the value is freed.

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(vec![1, 2, 3]);
    let b = Rc::clone(&a); // does NOT clone the Vec
    let c = Rc::clone(&a);

    println!("count: {}", Rc::strong_count(&a)); // 3
    println!("same data: {}", std::ptr::eq(&*a, &*b)); // true
}
```

`Rc::clone(&a)` doesn't clone the inner data. It increments a counter. All three variables - `a`, `b`, `c` - point to the exact same `Vec<i32>` on the heap. When `c` drops, the count goes to 2. When `b` drops, 1. When `a` drops, 0 - and the Vec is freed.

### The heap layout

Here's what `Rc` actually allocates. From the [standard library source](https://github.com/rust-lang/rust/blob/main/library/alloc/src/rc.rs):

```rust
#[repr(C)]
struct RcInner<T: ?Sized> {
    strong: Cell<usize>,  // 8 bytes
    weak: Cell<usize>,    // 8 bytes
    value: T,             // size_of::<T>() bytes
}
```

And `Rc` itself:

```rust
pub struct Rc<T: ?Sized, A: Allocator = Global> {
    ptr: NonNull<RcInner<T>>,
    phantom: PhantomData<RcInner<T>>,
    alloc: A,
}
```

For `Rc<i32>` on a 64-bit system:

```
Stack (Rc<i32>):      Heap (RcInner<i32>):
+----------+          +--------+--------+-------+
| ptr ------+-------->| strong | weak   | value |
+----------+          | 8B     | 8B     | 4B    |
  8 bytes             +--------+--------+-------+
                        = 20 bytes + padding = 24 bytes
```

8 bytes on the stack, 24 bytes on the heap. Every `Rc::clone` adds another 8 bytes on the stack and increments `strong` from, say, 1 to 2. No new heap allocation.

The `strong` and `weak` counters use `Cell<usize>` - non-atomic interior mutability. This is critical: because there's no atomic synchronization, `Rc` is fast. Incrementing a `Cell<usize>` compiles to a plain `add [mem], 1` on x86. But it's also why `Rc` is `!Send` and `!Sync` - you can't safely share it across threads.

### When Rc makes sense

**Tree structures with shared nodes.** A DOM tree where child nodes need references back to parents. A graph where multiple edges point to the same node.

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct TreeNode {
    value: i32,
    children: RefCell<Vec<Rc<TreeNode>>>,
    parent: RefCell<Option<Rc<TreeNode>>>,
}
```

**Immutable shared data in single-threaded apps.** Multiple components referencing the same configuration, the same parsed AST, the same font data. If you read the [flyweight pattern post](/blog/the-flyweight-pattern-sharing-data-efficiently-in-rust/), this is exactly the pattern - but with `Rc` instead of `Arc` when you don't need thread safety.

**Caching.** Keep an `Rc` to an expensive computation result and hand out clones to anyone who needs it.

### What Rc cannot do

`Rc` gives you shared ownership, but only shared *immutable* access. You can't get an `&mut T` from an `Rc<T>` (unless `strong_count == 1`, via `Rc::get_mut`). If you need mutation, you need interior mutability - we'll get to `RefCell` shortly.

## Arc - Rc for multiple threads

`Arc<T>` (Atomically Reference Counted) is `Rc<T>` with atomic counters instead of `Cell<usize>`. That single difference - atomic vs non-atomic increment - is what makes it safe to share across threads.

From the [source](https://github.com/rust-lang/rust/blob/main/library/alloc/src/sync.rs):

```rust
#[repr(C)]
struct ArcInner<T: ?Sized> {
    strong: Atomic<usize>,  // 8 bytes, atomic
    weak: Atomic<usize>,    // 8 bytes, atomic
    data: T,
}
```

The heap layout is identical to `Rc`. The size is identical. The API is nearly identical. The only difference is the cost of `clone` and `drop`.

### The cost of atomics

When `Rc` increments its counter, it does a plain memory write. When `Arc` increments, it uses `fetch_add(1, Relaxed)` - which compiles to `lock xadd` on x86. The `lock` prefix is a bus lock: it forces the CPU to acquire exclusive ownership of that cache line before modifying it.

On an uncontended core (only one thread touching that `Arc`), the overhead is small - roughly 5-10 nanoseconds per clone vs 2 nanoseconds for `Rc`. On contended cores (multiple threads cloning and dropping the same `Arc` simultaneously), the cost balloons because each core has to invalidate the others' cache lines. Measurements from [Travis Downs' concurrency cost hierarchy](https://travisdowns.github.io/blog/2020/07/06/concurrency-costs.html) show contended atomics reaching 100+ nanoseconds.

For most applications, this doesn't matter. You're not cloning `Arc` in a hot loop millions of times. But it's why `Rc` exists as a separate type - in single-threaded code, you're paying for synchronization you don't need.

### Arc in practice

If you've read the [dependency injection post](/blog/dependency-injection-patterns-without-a-framework/), you've already seen the most common `Arc` pattern: shared infrastructure in web services.

```rust
use std::sync::Arc;

struct AppState {
    db: DatabasePool,
    cache: RedisPool,
    config: AppConfig,
}

let state = Arc::new(AppState {
    db: DatabasePool::connect(&config.database_url).await?,
    cache: RedisPool::connect(&config.redis_url).await?,
    config,
});

// Each request handler gets a clone
let handler_state = Arc::clone(&state);
tokio::spawn(async move {
    handle_request(handler_state).await;
});
```

Each `tokio::spawn` moves an `Arc<AppState>` into the task. All tasks share the same database pool, the same cache connection, the same config. When the last task completes and drops its `Arc`, the pools are cleaned up.

This is also the pattern used in the [key-value store post](/blog/writing-a-key-value-store-in-rust/) for wrapping the store behind `Arc<RwLock<KvStore>>` - multiple threads reading and writing through a shared handle.

### Arc + Mutex vs Arc + RwLock

`Arc` gives you shared ownership. It does not give you shared mutation. For that, you pair it with a lock:

```rust
use std::sync::{Arc, Mutex, RwLock};

// Mutex: one reader OR one writer at a time
let counter = Arc::new(Mutex::new(0_u64));

// RwLock: many concurrent readers OR one exclusive writer
let cache = Arc::new(RwLock::new(HashMap::new()));
```

`Mutex` is simpler and has lower overhead per lock/unlock. `RwLock` allows concurrent reads, which matters when your workload is read-heavy. The standard library's `RwLock` can starve writers on some platforms - if that's a concern, use `parking_lot::RwLock` which has a fair scheduling policy.

```rust
use std::sync::{Arc, RwLock};
use std::collections::HashMap;
use std::thread;

fn main() {
    let map: Arc<RwLock<HashMap<String, String>>> = Arc::new(RwLock::new(HashMap::new()));

    let writer = {
        let map = Arc::clone(&map);
        thread::spawn(move || {
            let mut w = map.write().unwrap();
            w.insert("key".into(), "value".into());
        })
    };

    let reader = {
        let map = Arc::clone(&map);
        thread::spawn(move || {
            let r = map.read().unwrap();
            println!("{:?}", r.get("key"));
        })
    };

    writer.join().unwrap();
    reader.join().unwrap();
}
```

## Cow - clone on write

`Cow<'a, B>` is fundamentally different from the other three. It's not about heap allocation or reference counting. It's about deferring allocation until you actually need to mutate.

From the [source](https://github.com/rust-lang/rust/blob/main/library/alloc/src/borrow.rs):

```rust
pub enum Cow<'a, B: ?Sized + ToOwned + 'a> {
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```

It's an enum with two variants. `Borrowed` holds a reference. `Owned` holds an owned value. The `ToOwned` trait connects the borrowed type to its owned counterpart: `str` to `String`, `[T]` to `Vec<T>`, `Path` to `PathBuf`.

The magic is in `to_mut()`: if the `Cow` is `Borrowed`, it clones the data into an `Owned` variant. If it's already `Owned`, it returns a mutable reference to the existing data. You pay for the clone exactly once, and only if you actually mutate.

```rust
use std::borrow::Cow;

fn ensure_lowercase(input: &str) -> Cow<str> {
    if input.chars().all(|c| c.is_lowercase()) {
        Cow::Borrowed(input) // already lowercase, no allocation
    } else {
        Cow::Owned(input.to_lowercase()) // needs work, allocate
    }
}

fn main() {
    let a = ensure_lowercase("hello");    // Cow::Borrowed - zero alloc
    let b = ensure_lowercase("Hello");    // Cow::Owned - one alloc

    // Both can be used as &str through Deref
    println!("{} {}", a, b);
}
```

In the happy path - input is already lowercase - zero allocations. In the unhappy path, one allocation. Compare to a function that always returns `String`: you'd allocate even when the input was already fine.

### Where Cow shines

**String processing pipelines.** When most inputs pass through unchanged but some need transformation:

```rust
fn normalize_whitespace(s: &str) -> Cow<str> {
    if s.contains("  ") {
        let normalized = s.split_whitespace().collect::<Vec<_>>().join(" ");
        Cow::Owned(normalized)
    } else {
        Cow::Borrowed(s)
    }
}
```

**Deserialization.** Serde's `#[serde(borrow)]` attribute uses `Cow<str>` to borrow directly from the input buffer when possible. If the JSON string `"hello"` has no escape sequences, serde can point directly into the input bytes - zero copy. If it contains `\"` or `\n`, serde must allocate an unescaped `String`:

```rust
use serde::Deserialize;
use std::borrow::Cow;

#[derive(Deserialize)]
struct Request<'a> {
    #[serde(borrow)]
    name: Cow<'a, str>,    // borrows from input when possible
    #[serde(borrow)]
    path: Cow<'a, str>,    // same
}
```

**API boundaries.** Functions that accept either borrowed or owned data:

```rust
fn log_message(msg: Cow<str>) {
    println!("[LOG] {}", msg);
}

// Both work without the caller choosing
log_message(Cow::Borrowed("static string"));
log_message(Cow::Owned(format!("dynamic: {}", 42)));
```

I covered `Cow<str>` from the allocation-avoidance angle in the [flyweight pattern post](/blog/the-flyweight-pattern-sharing-data-efficiently-in-rust/) and from the string-type-selection angle in the [string types post](/blog/why-rust-has-so-many-string-types/). The unique angle here is understanding it as a smart pointer - it implements `Deref`, so it quacks like a `&str`, but it may secretly own its data.

## Weak references - breaking cycles

Reference counting has a well-known failure mode: cycles. If A holds an `Rc` to B and B holds an `Rc` to A, neither will ever reach a count of zero. They leak.

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    value: i32,
    next: RefCell<Option<Rc<Node>>>,
}

fn main() {
    let a = Rc::new(Node { value: 1, next: RefCell::new(None) });
    let b = Rc::new(Node { value: 2, next: RefCell::new(None) });

    // Create a cycle: a -> b -> a
    *a.next.borrow_mut() = Some(Rc::clone(&b));
    *b.next.borrow_mut() = Some(Rc::clone(&a));

    // When main ends:
    // a's strong count is 2 (main + b.next)
    // b's strong count is 2 (main + a.next)
    // main drops a: count goes to 1. Not zero. Not freed.
    // main drops b: count goes to 1. Not zero. Not freed.
    // Both leak.
}
```

`Weak<T>` breaks cycles. A `Weak` reference doesn't increment the strong count. It increments the *weak* count, which keeps the allocation alive (so the weak pointer stays valid) but doesn't prevent the *value* from being dropped.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    children: RefCell<Vec<Rc<Node>>>,
    parent: RefCell<Option<Weak<Node>>>, // Weak, not Rc
}

fn main() {
    let parent = Rc::new(Node {
        value: 1,
        children: RefCell::new(vec![]),
        parent: RefCell::new(None),
    });

    let child = Rc::new(Node {
        value: 2,
        children: RefCell::new(vec![]),
        parent: RefCell::new(Some(Rc::downgrade(&parent))),
    });

    parent.children.borrow_mut().push(Rc::clone(&child));

    // Access parent from child:
    if let Some(p) = child.parent.borrow().as_ref().and_then(|w| w.upgrade()) {
        println!("parent value: {}", p.value);
    }
}
```

`Rc::downgrade(&parent)` creates a `Weak<Node>`. It doesn't increment `parent`'s strong count. When you need the actual value, you call `weak.upgrade()`, which returns `Option<Rc<Node>>` - `Some` if the value is still alive, `None` if all strong references are gone.

The rule of thumb: **parent owns children (strong), children reference parents (weak).** This applies to DOM trees, observer patterns, caches, and any bidirectional relationship.

`Arc` has the same mechanism: `Arc::downgrade()` creates an `Arc<T>`'s counterpart `Weak<T>` (from `std::sync`).

### How weak counts work

Remember the `RcInner` layout has both `strong` and `weak` counters. The lifecycle is:

1. `Rc::new(value)` - strong=1, weak=1 (the "implicit weak" representing all strong refs)
2. `Rc::clone()` - strong=2, weak=1
3. `Rc::downgrade()` - strong=2, weak=2
4. Last `Rc` drops - strong=0, weak=1. The *value* is dropped (destructor runs), but the allocation stays alive because weak > 0
5. Last `Weak` drops - weak=0. The allocation is freed

This two-phase destruction is why `upgrade()` can return `None` - the value is gone but the allocation (with its counters) still exists, so the `Weak` pointer is valid enough to check.

## Interior mutability - Cell and RefCell

You've seen `RefCell` in the tree examples above, but let's explain what it actually does and why it exists.

Rust's borrowing rules enforce at compile time: either one `&mut T` or any number of `&T`, never both. Interior mutability moves this check to runtime, allowing mutation through a shared reference.

### Cell - copy-based mutation

`Cell<T>` works for types that implement `Copy`. It lets you get and set the value without a borrow:

```rust
use std::cell::Cell;

fn main() {
    let counter = Cell::new(0_u32);

    counter.set(counter.get() + 1);
    counter.set(counter.get() + 1);

    println!("{}", counter.get()); // 2
}
```

`Cell` has zero overhead. It's the same size as `T`. No runtime checks, no flags, no panics. The trick is that `get()` returns a copy and `set()` replaces the whole value. You never get a reference into the Cell, so there's no way to violate the aliasing rules.

This is exactly what `Rc` uses internally - `Cell<usize>` for its reference counts. Incrementing the count through a shared `&Rc<T>` reference works because `Cell` allows mutation through `&self`.

```rust
// Simplified Rc::clone
fn clone(this: &Rc<T>) -> Rc<T> {
    // this.inner().strong is a Cell<usize>
    this.inner().strong.set(this.inner().strong.get() + 1);
    Rc { ptr: this.ptr }
}
```

### RefCell - borrow-checked at runtime

`RefCell<T>` works for any `T`. It tracks borrows at runtime instead of compile time:

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);

    // borrow() returns Ref<Vec<i32>> - like &Vec<i32>
    {
        let r = data.borrow();
        println!("len: {}", r.len());
    } // Ref dropped, borrow released

    // borrow_mut() returns RefMut<Vec<i32>> - like &mut Vec<i32>
    {
        let mut w = data.borrow_mut();
        w.push(4);
    } // RefMut dropped, borrow released

    println!("{:?}", data.borrow()); // [1, 2, 3, 4]
}
```

`RefCell` stores a borrow counter alongside the value. `borrow()` checks that no mutable borrow exists and increments the reader count. `borrow_mut()` checks that no borrows of any kind exist. If either check fails, it panics.

```rust
let data = RefCell::new(42);
let r = data.borrow();
let w = data.borrow_mut(); // PANIC: already borrowed
```

This is why `RefCell` is a last resort, not a first choice. It converts compile-time safety into runtime panics. Use it when the compiler can't prove your borrows are disjoint - typically in graph structures, observer patterns, or when you need mutation inside an `Rc`.

### The Rc + RefCell pattern

`Rc` gives shared ownership. `RefCell` gives mutation through shared references. Together, they give multiple owners mutable access to the same value:

```rust
use std::rc::Rc;
use std::cell::RefCell;

type SharedVec = Rc<RefCell<Vec<String>>>;

fn add_item(list: &SharedVec, item: String) {
    list.borrow_mut().push(item);
}

fn main() {
    let list: SharedVec = Rc::new(RefCell::new(vec![]));
    let list2 = Rc::clone(&list);

    add_item(&list, "first".into());
    add_item(&list2, "second".into());

    println!("{:?}", list.borrow()); // ["first", "second"]
}
```

For the multi-threaded equivalent, you'd use `Arc<Mutex<T>>` or `Arc<RwLock<T>>` - the `Mutex` and `RwLock` provide interior mutability with thread safety, while `Arc` provides shared ownership.

| Pattern | Thread safety | Mutation model |
|---------|--------------|----------------|
| `Rc<RefCell<T>>` | Single-thread only | Runtime borrow check, panics on violation |
| `Arc<Mutex<T>>` | Multi-thread | Lock-based, blocks on contention |
| `Arc<RwLock<T>>` | Multi-thread | Read-write lock, concurrent reads allowed |

## Decision tree: which pointer do I need?

Start with the simplest option and move down only when you hit a real limitation:

**1. Do you need heap allocation but only one owner?**
Use `Box<T>`. Common cases: recursive types, trait objects (`Box<dyn Trait>`), large structs you don't want to move.

**2. Do you need multiple owners on the same thread?**
Use `Rc<T>`. Add `RefCell<T>` inside if you need mutation.

**3. Do you need multiple owners across threads?**
Use `Arc<T>`. Add `Mutex<T>` or `RwLock<T>` inside if you need mutation.

**4. Do you want to avoid allocation in the common case?**
Use `Cow<'a, B>`. Works when most inputs pass through unchanged and only some need transformation.

**5. Do you have a parent-child cycle?**
Use `Weak<T>` for the back-reference (child to parent). Keep `Rc` or `Arc` for the forward reference (parent to child).

Here's the same logic as code:

```
Need to store on heap?
├── Yes, one owner
│   └── Box<T>
├── Yes, shared ownership
│   ├── Single thread?
│   │   ├── Immutable? → Rc<T>
│   │   └── Mutable? → Rc<RefCell<T>>
│   └── Multi-thread?
│       ├── Immutable? → Arc<T>
│       ├── Mutable (exclusive access)? → Arc<Mutex<T>>
│       └── Mutable (read-heavy)? → Arc<RwLock<T>>
├── Maybe, only if I mutate
│   └── Cow<'a, B>
└── No
    └── Plain T or &T
```

## Practical examples

Let's put this all together with some real scenarios.

### Example 1: A configuration store

Configuration is read once, shared everywhere, never mutated. Plain `Arc<T>`:

```rust
use std::sync::Arc;

#[derive(Debug)]
struct Config {
    database_url: String,
    max_connections: u32,
    log_level: String,
}

impl Config {
    fn from_env() -> Self {
        Config {
            database_url: std::env::var("DATABASE_URL")
                .unwrap_or_else(|_| "postgres://localhost/app".into()),
            max_connections: std::env::var("MAX_CONN")
                .ok()
                .and_then(|s| s.parse().ok())
                .unwrap_or(10),
            log_level: std::env::var("LOG_LEVEL")
                .unwrap_or_else(|_| "info".into()),
        }
    }
}

fn start_server(config: Arc<Config>) {
    for i in 0..config.max_connections {
        let cfg = Arc::clone(&config);
        std::thread::spawn(move || {
            println!("Worker {i} connecting to {}", cfg.database_url);
        });
    }
}
```

No `Mutex`, no `RefCell`. The config is immutable after creation. `Arc` handles the shared ownership, and each thread gets read-only access for free.

### Example 2: An AST with parent references

A parsed AST where nodes need to reference their parents. Classic `Rc` + `Weak`:

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
enum NodeKind {
    Document,
    Heading(u8),
    Paragraph,
    Text(String),
}

struct AstNode {
    kind: NodeKind,
    children: RefCell<Vec<Rc<AstNode>>>,
    parent: RefCell<Option<Weak<AstNode>>>,
}

impl AstNode {
    fn new(kind: NodeKind) -> Rc<Self> {
        Rc::new(AstNode {
            kind,
            children: RefCell::new(vec![]),
            parent: RefCell::new(None),
        })
    }

    fn add_child(parent: &Rc<AstNode>, child: &Rc<AstNode>) {
        parent.children.borrow_mut().push(Rc::clone(child));
        *child.parent.borrow_mut() = Some(Rc::downgrade(parent));
    }

    fn depth(&self) -> usize {
        match self.parent.borrow().as_ref().and_then(|w| w.upgrade()) {
            Some(p) => 1 + p.depth(),
            None => 0,
        }
    }
}

fn main() {
    let doc = AstNode::new(NodeKind::Document);
    let heading = AstNode::new(NodeKind::Heading(1));
    let text = AstNode::new(NodeKind::Text("Hello".into()));

    AstNode::add_child(&doc, &heading);
    AstNode::add_child(&heading, &text);

    println!("text depth: {}", text.depth()); // 2
}
```

Parent references are `Weak` so they don't create cycles. Children are `Rc` because the parent owns them. `RefCell` on both `children` and `parent` because we modify them after construction.

### Example 3: A string normalizer with Cow

Processing user input where most strings are fine but some need fixing:

```rust
use std::borrow::Cow;

fn normalize_input(input: &str) -> Cow<str> {
    // Fast path: check if normalization is needed
    let needs_work = input.contains('\t')
        || input.contains('\r')
        || input.starts_with(' ')
        || input.ends_with(' ');

    if !needs_work {
        return Cow::Borrowed(input);
    }

    // Slow path: allocate and normalize
    let result = input
        .replace('\t', "    ")
        .replace('\r', "")
        .trim()
        .to_string();

    Cow::Owned(result)
}

fn process_lines(lines: &[&str]) -> Vec<Cow<str>> {
    lines.iter().map(|line| normalize_input(line)).collect()
}

fn main() {
    let lines = vec![
        "normal line",          // Borrowed - zero alloc
        "  leading space",      // Owned - one alloc
        "has\ttab",             // Owned - one alloc
        "also normal",          // Borrowed - zero alloc
    ];

    let normalized = process_lines(&lines);
    for line in &normalized {
        match line {
            Cow::Borrowed(_) => print!("[borrowed] "),
            Cow::Owned(_) => print!("[owned]   "),
        }
        println!("{}", line);
    }
}
```

If 90% of your input lines are already clean, you save 90% of the allocations. This is the same pattern `Path::to_string_lossy()` uses in the standard library - returns `Cow::Borrowed` when the path is valid UTF-8 (the common case), `Cow::Owned` only when replacement characters are needed.

## Common mistakes

**Using `Arc` when you need `Rc`.** If your code is single-threaded, `Rc` is strictly better. The compiler won't stop you from using `Arc` everywhere, but you're paying for atomic operations you don't need.

**Using `Rc<RefCell<T>>` when a simple `&mut T` would work.** If you can restructure your code to pass a mutable reference instead of sharing ownership, do that. Interior mutability is a tool for when the borrow checker genuinely can't express your access pattern - not a way to avoid thinking about ownership.

**Cloning an `Arc<Mutex<T>>` in a loop instead of taking the lock once.**

```rust
// Bad: clone Arc and lock on every iteration
for item in &items {
    let store = Arc::clone(&store);
    let mut guard = store.lock().unwrap();
    guard.insert(item.key.clone(), item.value.clone());
}

// Good: lock once, iterate inside the lock
let mut guard = store.lock().unwrap();
for item in &items {
    guard.insert(item.key.clone(), item.value.clone());
}
```

**Forgetting that `Cow::to_mut()` clones on first mutation.**

```rust
let mut cow: Cow<str> = Cow::Borrowed("hello");
// This clones "hello" into a String
let s: &mut String = cow.to_mut();
s.push_str(" world");
// Now cow is Cow::Owned("hello world")
// A second to_mut() does NOT clone again
let s2: &mut String = cow.to_mut();
s2.push_str("!");
```

The first `to_mut()` allocates. Every subsequent `to_mut()` on the same `Cow` is free. If you're going to mutate, do it all at once.

## Performance reality check

Here's a quick comparison of what each operation actually costs, measured on x86-64:

| Operation | Approximate cost |
|-----------|-----------------|
| Dereference any smart pointer | ~0 ns (compiled away) |
| `Box::new(42_i32)` | ~5-15 ns (heap alloc) |
| `Rc::clone()` | ~2 ns (non-atomic increment) |
| `Arc::clone()` (uncontended) | ~5-10 ns (atomic increment) |
| `Arc::clone()` (contended) | ~50-150 ns (cache line bouncing) |
| `RefCell::borrow()` | ~1 ns (counter check + increment) |
| `Mutex::lock()` (uncontended) | ~15-25 ns |
| `RwLock::read()` (uncontended) | ~10-20 ns |

The takeaway: dereferencing is free, cloning `Rc`/`Arc` is cheap, locks are cheap when uncontended. The expensive case is contention - multiple threads fighting over the same `Arc` or `Mutex`. If that's your bottleneck, the answer is usually to restructure the data to reduce sharing, not to pick a faster smart pointer.

## The standard library's pointer zoo

Beyond the four main types, Rust has a few more pointer-like types worth knowing:

- `Pin<P>` - guarantees the pointed-to value won't move in memory. Critical for self-referential types and `async`/`await` (futures that hold references to their own state).
- `ManuallyDrop<T>` - wraps a value and disables automatic drop. You become responsible for calling `drop` yourself. Used in low-level code where drop order matters.
- `MaybeUninit<T>` - wraps a value that might not be initialized yet. Used in unsafe code to create values before fully initializing them.

These are specialized tools. You'll encounter `Pin` when writing custom futures or working with `tokio`'s internals (I touched on this in the [Tokio post](/blog/understanding-tokio-the-rust-async-runtime-under-the-hood/)). The other two are for unsafe territory.

## Wrapping up

Smart pointers in Rust aren't about "making the compiler happy." They encode ownership semantics in the type system. `Box` says "exactly one owner." `Rc` says "multiple owners, one thread." `Arc` says "multiple owners, any thread." `Cow` says "borrowed until proven otherwise." Each one gives the compiler and your fellow developers information about how data flows through the program.

The decision tree is simple: start with plain values and references. Reach for `Box` when you need indirection. Reach for `Rc` or `Arc` when you need shared ownership. Reach for `Cow` when you want to defer allocation. Add `Cell`, `RefCell`, `Mutex`, or `RwLock` when you need interior mutability. And use `Weak` when you have cycles.

When in doubt, start with the least powerful option. You can always upgrade from `Box` to `Rc` or from `Rc` to `Arc` later - the API is nearly identical. Going the other direction (removing shared ownership after it's threaded through your codebase) is much harder.
