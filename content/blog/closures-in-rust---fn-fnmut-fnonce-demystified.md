+++
title = "Closures in Rust - Fn, FnMut, FnOnce demystified"
date = 2025-07-19
description = "What the compiler actually generates when you write a closure, how it picks between Fn/FnMut/FnOnce, and why Rust closures cost nothing at runtime."

[taxonomies]
tags = ["rust", "closures", "compiler-internals", "performance"]
+++

Every language has closures. JavaScript, Python, Ruby, Go - they all let you capture variables from the surrounding scope and pass around little chunks of logic. Rust has closures too, but they work fundamentally differently under the hood. Where other languages heap-allocate a closure environment and lean on a garbage collector to clean it up, Rust turns each closure into a unique struct, decides at compile time how captured variables are accessed, and generates code that's as efficient as a hand-written function call.

The mechanism behind this is three traits: `Fn`, `FnMut`, and `FnOnce`. If you've used closures with iterators, `thread::spawn`, or callback-heavy APIs, you've run into them. But the compiler picks which one applies, and the rules aren't always obvious. Let's unpack what's actually happening.

<!-- more -->

## A closure is a struct

This is the single most important thing to understand. When you write:

```rust
let offset = 10;
let add = |x: i32| x + offset;
```

The compiler doesn't create a function pointer or allocate anything on the heap. It generates something roughly equivalent to:

```rust
struct __ClosureAdd {
    offset: i32,  // captured by reference in practice, value here for simplicity
}

impl Fn<(i32,)> for __ClosureAdd {
    extern "rust-call" fn call(&self, (x,): (i32,)) -> i32 {
        x + self.offset
    }
}
```

Every closure gets its own unique anonymous type. Even two closures with identical signatures and bodies have different types. The struct's fields are the captured variables, and the closure body becomes the `call` method implementation.

You can verify this with `std::mem::size_of_val`:

```rust
fn main() {
    let x: i32 = 42;
    let y: i64 = 100;

    let c1 = || x;           // captures one i32
    let c2 = || x + y as i32; // captures i32 + i64

    println!("c1 size: {}", std::mem::size_of_val(&c1)); // 4
    println!("c2 size: {}", std::mem::size_of_val(&c2)); // 12 (4 + 8, with alignment)

    let c3 = || 42;          // captures nothing
    println!("c3 size: {}", std::mem::size_of_val(&c3)); // 0
}
```

A closure that captures nothing is zero-sized. A closure that captures an `i32` is 4 bytes. No vtable pointer, no reference count, no allocation header. Just the data.

## The three traits

The closure traits live in [`std::ops`](https://doc.rust-lang.org/std/ops/index.html) and form a hierarchy. Here are the [actual definitions](https://doc.rust-lang.org/std/ops/trait.FnOnce.html) from the standard library (simplified, the real ones use unstable `extern "rust-call"` ABI):

```rust
pub trait FnOnce<Args> {
    type Output;
    fn call_once(self, args: Args) -> Self::Output;
}

pub trait FnMut<Args>: FnOnce<Args> {
    fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait Fn<Args>: FnMut<Args> {
    fn call(&self, args: Args) -> Self::Output;
}
```

The difference is entirely in the `self` parameter:

| Trait    | Receiver    | Can call multiple times? | Can mutate captures? |
|----------|-------------|--------------------------|----------------------|
| `FnOnce` | `self`      | No (consumed)            | Yes (owns them)      |
| `FnMut`  | `&mut self` | Yes                      | Yes                  |
| `Fn`     | `&self`     | Yes                      | No                   |

The hierarchy matters: `Fn: FnMut: FnOnce`. This means every `Fn` is also an `FnMut`, and every `FnMut` is also an `FnOnce`. If a function expects `FnOnce`, you can pass any closure. If it expects `Fn`, only closures that don't mutate or consume captures will work.

If you've read my post on [trait bounds](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg/), this supertrait chain should look familiar - `FnOnce` is the supertrait that everything builds on.

## How the compiler picks the trait

The compiler always picks the most permissive trait a closure can implement. The algorithm is straightforward:

1. If the closure body only reads captured variables (or captures nothing) - implement `Fn` (which also gives you `FnMut` and `FnOnce` for free)
2. If the closure body mutates a captured variable - implement `FnMut` (and `FnOnce`)
3. If the closure body moves a captured variable out (consumes it) - implement only `FnOnce`

```rust
fn main() {
    let name = String::from("Alice");

    // Fn: only reads `name`
    let greet = || println!("Hello, {name}");
    greet();
    greet(); // can call multiple times

    let mut counter = 0;

    // FnMut: mutates `counter`
    let mut increment = || {
        counter += 1;
        counter
    };
    increment();
    increment(); // still fine, just needs &mut

    let data = vec![1, 2, 3];

    // FnOnce: moves `data` into the return value
    let consume = || {
        let owned = data; // data moved into closure body
        owned.len()
    };
    consume();
    // consume(); // ERROR: closure already consumed `data`
}
```

Notice that the second closure (`increment`) needs to be declared `let mut` - because calling it requires `&mut self`, and you need a mutable binding to get a mutable reference.

## Capture semantics and RFC 2229

Before Rust 2021, closures captured entire variables. If you accessed `point.x` inside a closure, the compiler would capture all of `point`. This led to frustrating borrow checker errors where you couldn't use `point.y` outside the closure even though the closure only touched `point.x`.

[RFC 2229](https://rust-lang.github.io/rfcs/2229-capture-disjoint-fields.html) changed this. Since the 2021 edition, closures capture the minimal path needed:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let mut point = Point { x: 1, y: 2 };

    // Rust 2021: only captures `point.x`
    let mut cx = || point.x += 1;

    // This works in 2021 - `point.y` isn't captured
    println!("y = {}", point.y);

    cx();
    println!("x = {}", point.x); // 2
}
```

In Rust 2018, this wouldn't compile. The closure would capture the entire `point` mutably, blocking access to `point.y`. The 2021 edition captures only `point.x`, leaving `point.y` free.

This disjoint capture also affects which trait the closure implements. If a struct has a `String` field and an `i32` field, and the closure only moves the `i32`, it won't be limited to `FnOnce` just because the struct contains a non-Copy `String`.

The default capture modes, from least to most restrictive:

- **Shared reference** (`&T`) - used when the closure only reads the value
- **Mutable reference** (`&mut T`) - used when the closure modifies the value
- **By value** (`T`) - used when the closure needs ownership (e.g., moves the value somewhere else, or the type doesn't implement `Copy`)

## The move keyword

`move` forces all captured variables to be moved into the closure by value, regardless of how they're used in the body:

```rust
fn make_greeter(name: String) -> impl Fn() {
    // Without `move`, this borrows `name`
    // But `name` is dropped at the end of this function
    // So we MUST move it into the closure
    move || println!("Hello, {name}")
}

fn main() {
    let greeter = make_greeter("Alice".to_string());
    greeter();
    greeter();
}
```

Without `move`, the closure would try to borrow `name`, which doesn't live long enough. The `move` keyword transfers ownership of `name` into the closure struct, so it lives as long as the closure does.

A common point of confusion: `move` doesn't change which trait the closure implements. A `move` closure that only reads its captures still implements `Fn`:

```rust
fn main() {
    let x = 42;

    // `move` copies x into the closure (i32 is Copy)
    // but the closure still only reads it
    let f = move || println!("{x}");
    f();
    f(); // still Fn, not FnOnce

    // x is still usable because i32 is Copy
    println!("{x}");
}
```

`move` affects how variables are captured (by value instead of by reference). The trait is still determined by how the closure body uses those captures.

The most common use case for `move` is spawning threads or returning closures from functions - any situation where the closure must outlive the current scope.

## Closures as function arguments

You have three options for accepting closures as parameters, and the choice matters. I covered static vs dynamic dispatch in my [strategy pattern post](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/), and closures follow the same principles.

### Generics with trait bounds (static dispatch)

```rust
fn apply_twice<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(f(x))
}

fn main() {
    let result = apply_twice(|x| x * 2, 5);
    println!("{result}"); // 20
}
```

The compiler monomorphizes `apply_twice` for each concrete closure type. The closure call gets inlined - there's no function pointer, no vtable lookup. In release mode, the generated assembly for `apply_twice(|x| x * 2, 5)` is often just the constant `20`.

### Trait objects (dynamic dispatch)

```rust
fn apply_all(functions: &[&dyn Fn(i32) -> i32], x: i32) -> Vec<i32> {
    functions.iter().map(|f| f(x)).collect()
}
```

When you need a heterogeneous collection of closures (different types in the same container), you need `dyn Fn`. Each call goes through a vtable - an indirect function pointer. This prevents inlining but gives you runtime flexibility.

### Function pointers

```rust
fn apply_fp(f: fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}
```

The lowercase `fn` is a function pointer type. It can only accept actual functions or closures that capture nothing (since there's no environment to carry). Function pointers are 8 bytes (a single pointer), while `dyn Fn` is a fat pointer (16 bytes - pointer to data + pointer to vtable).

Pick generics by default. Reach for `dyn Fn` when you need to store different closures in one collection. Use `fn` pointers when you're interfacing with C or need a simple callback with no environment.

## Closures as return types

Returning closures is where it gets interesting. Since every closure has a unique anonymous type, you can't name it. Two options:

### impl Fn (stack-allocated, static dispatch)

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

fn main() {
    let add5 = make_adder(5);
    println!("{}", add5(10)); // 15
}
```

`impl Fn(i32) -> i32` means "I'm returning some type that implements `Fn(i32) -> i32`, but I won't tell you which." The closure lives on the stack (the caller's stack frame). No heap allocation. The compiler knows the exact type and can inline everything.

The catch: you can only return one concrete type. This won't work:

```rust
// ERROR: different return types
fn make_op(add: bool) -> impl Fn(i32) -> i32 {
    if add {
        |x| x + 1    // anonymous type A
    } else {
        |x| x * 2    // anonymous type B - different type!
    }
}
```

### Box\<dyn Fn\> (heap-allocated, dynamic dispatch)

```rust
fn make_op(add: bool) -> Box<dyn Fn(i32) -> i32> {
    if add {
        Box::new(|x| x + 1)
    } else {
        Box::new(|x| x * 2)
    }
}

fn main() {
    let op = make_op(true);
    println!("{}", op(10)); // 11
}
```

`Box<dyn Fn>` puts the closure on the heap and uses a fat pointer (data pointer + vtable pointer). Each call is an indirect call through the vtable. You pay for the heap allocation and the indirection, but you gain the ability to return different closure types from the same function.

Here's what the memory layout looks like:

```text
impl Fn:
  [closure data on stack]    <- direct call, inlineable

Box<dyn Fn>:
  Stack: [ptr to heap data | ptr to vtable]   (16 bytes)
  Heap:  [closure captured data]
  Vtable: [drop_fn | size | align | call_fn]
```

The performance difference is measurable. The [EventHelix analysis](https://www.eventhelix.com/rust/rust-to-assembly-return-impl-fn-vs-dyn-fn/) shows that `impl Fn` compiles down to a direct `call` instruction (or gets inlined entirely), while `Box<dyn Fn>` generates a heap allocation via `__rust_alloc` plus an indirect `call` through the vtable.

## Zero-cost proof

Let's look at what the compiler actually produces. Take this iterator chain:

```rust
pub fn sum_of_squares(v: &[i32]) -> i32 {
    v.iter()
        .filter(|&&x| x > 0)
        .map(|&x| x * x)
        .sum()
}
```

Two closures, a filter and a map. On [Godbolt](https://rust.godbolt.org/) with `-O` (release optimization), this compiles to a tight loop with no function calls, no closure structs in memory, no indirect jumps. The closures are fully inlined. The compiler sees through the iterator adaptors and the closures and generates the same code you'd get from a hand-written `for` loop with an accumulator.

Compare that to the `Box<dyn Fn>` approach:

```rust
pub fn apply_boxed(f: Box<dyn Fn(i32) -> i32>, x: i32) -> i32 {
    f(x)
}
```

This generates an indirect call through the vtable - `call qword ptr [rax + 24]` on x86-64. The compiler can't inline across the vtable indirection because it doesn't know at compile time which function is behind the pointer.

This is the core of "zero-cost": when you use generics or `impl Fn`, you get static dispatch. The closure type is known at compile time, so the compiler can inline, optimize, and eliminate all abstraction overhead. You only pay for dynamic dispatch when you explicitly ask for it with `dyn`.

## Closures you can implement yourself

The `Fn` traits aren't magic - they're traits. On nightly, you can implement them manually (though this requires the unstable `fn_traits` and `unboxed_closures` features):

```rust
#![feature(fn_traits, unboxed_closures)]

struct Multiplier {
    factor: i32,
}

impl FnOnce<(i32,)> for Multiplier {
    type Output = i32;
    extern "rust-call" fn call_once(self, (x,): (i32,)) -> i32 {
        self.factor * x
    }
}

impl FnMut<(i32,)> for Multiplier {
    extern "rust-call" fn call_mut(&mut self, (x,): (i32,)) -> i32 {
        self.factor * x
    }
}

impl Fn<(i32,)> for Multiplier {
    extern "rust-call" fn call(&self, (x,): (i32,)) -> i32 {
        self.factor * x
    }
}
```

On stable Rust, you can't implement `Fn` traits directly, but you can achieve the same effect by overloading the call operator through a newtype that wraps a closure, or simply using a struct with a method and passing it where `impl Fn` is expected via a closure wrapper.

This is worth knowing because it demystifies what closures are - they're just structs with a special syntax for construction and calling.

## How Rust closures differ from JS and Python

In JavaScript and Python, closures are heap-allocated objects managed by a garbage collector. Let's compare:

**JavaScript:**

```javascript
function makeCounter() {
    let count = 0;
    return () => {
        count += 1;
        return count;
    };
}
```

The `count` variable is allocated on the heap as part of a "closure scope" object. The returned function holds a reference to this scope. The garbage collector (V8's mark-and-sweep) keeps `count` alive as long as the closure exists. There's no way to know at the call site whether the closure mutates its captures - everything goes through a shared reference.

**Python:**

```python
def make_counter():
    count = [0]  # list because nonlocal is clunky
    def counter():
        count[0] += 1
        return count[0]
    return counter
```

Python uses reference counting plus a cycle collector. The closure's `__closure__` attribute holds `cell` objects that reference the captured variables. Every access to `count` goes through a cell indirection.

**Rust:**

```rust
fn make_counter() -> impl FnMut() -> i32 {
    let mut count = 0;
    move || {
        count += 1;
        count
    }
}
```

No heap allocation. `count` is a field of the closure struct, living on the stack of whoever owns the closure. No garbage collector, no reference counting, no cell indirection. The `FnMut` trait tells the caller "you need exclusive access to call this" - a compile-time guarantee that prevents data races.

The key differences summarized:

| Aspect | Rust | JavaScript | Python |
|--------|------|------------|--------|
| Allocation | Stack (or heap if boxed) | Heap (GC-managed) | Heap (refcounted) |
| Capture tracking | Compile-time (Fn/FnMut/FnOnce) | None (runtime free-for-all) | None |
| Mutation control | Type system enforced | No enforcement | No enforcement |
| Overhead per closure | 0 bytes if no captures | Object header + scope chain | Object header + cell objects |
| Dead capture cleanup | Deterministic (drop at scope end) | GC sweep (non-deterministic) | Refcount decrement |
| Can cause memory leaks | No (without Rc cycles) | Yes (accidental scope retention) | Yes (reference cycles) |

JavaScript has a particularly nasty footgun here: [closures can prevent GC of variables they don't even use](https://jakearchibald.com/2024/garbage-collection-and-closures/). If two closures share a scope, and one of them references a large buffer, the buffer stays alive even if only the other closure survives. V8 has optimizations for this, but it's not guaranteed across engines.

Rust doesn't have this problem. RFC 2229 ensures closures capture exactly what they need, and nothing more.

## Practical patterns and gotchas

**Closure in a struct** - a common pattern for callbacks:

```rust
struct Button<F: Fn()> {
    on_click: F,
}

impl<F: Fn()> Button<F> {
    fn click(&self) {
        (self.on_click)();
    }
}

fn main() {
    let label = String::from("Submit");
    let button = Button {
        on_click: move || println!("Clicked: {label}"),
    };
    button.click();
}
```

The generic approach means each `Button` with a different closure is a different concrete type. If you need to store multiple buttons with different callbacks in one `Vec`, switch to `Box<dyn Fn()>`:

```rust
struct DynButton {
    on_click: Box<dyn Fn()>,
}

fn main() {
    let buttons: Vec<DynButton> = vec![
        DynButton { on_click: Box::new(|| println!("Save")) },
        DynButton { on_click: Box::new(|| println!("Cancel")) },
    ];

    for b in &buttons {
        (b.on_click)();
    }
}
```

This is the same trade-off I discussed in the [strategy pattern post](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/) - static dispatch for performance, dynamic dispatch for flexibility.

**The clone-before-move pattern** for sharing data across multiple closures:

```rust
fn main() {
    let shared = String::from("shared data");

    let c1 = {
        let shared = shared.clone();
        move || println!("c1: {shared}")
    };

    let c2 = move || println!("c2: {shared}");

    c1();
    c2();
}
```

Each closure gets its own owned copy. If you want actual sharing without cloning, use `Arc`:

```rust
use std::sync::Arc;

fn main() {
    let shared = Arc::new(String::from("shared data"));

    let c1 = {
        let shared = Arc::clone(&shared);
        move || println!("c1: {shared}")
    };

    let c2 = {
        let shared = Arc::clone(&shared);
        move || println!("c2: {shared}")
    };

    c1();
    c2();
}
```

**Async closures** - as of Rust 1.85 (stabilized February 2025), you can use `async` closures directly:

```rust
async fn fetch_all<F, Fut>(urls: Vec<String>, fetcher: F)
where
    F: async Fn(&str) -> Result<String, Box<dyn std::error::Error>>,
{
    for url in &urls {
        match fetcher(url).await {
            Ok(body) => println!("Got {} bytes from {url}", body.len()),
            Err(e) => eprintln!("Failed {url}: {e}"),
        }
    }
}
```

Before 1.85, you had to use the `Fn(&str) -> impl Future<Output = ...>` workaround, which had issues with lifetime captures. The `async Fn` trait bound is cleaner and handles lifetimes correctly.

## When to use which

Quick reference for the three traits as function parameters:

- **`Fn`** - Use when you need to call the closure multiple times and don't need mutation. Iterators, read-only callbacks, pure transformations.
- **`FnMut`** - Use when the closure needs to update state between calls. Accumulators, stateful filters, builders.
- **`FnOnce`** - Use when you only call the closure once, or when it needs to transfer ownership out. `thread::spawn`, `unwrap_or_else`, one-shot callbacks.

When in doubt, start with `FnOnce` as your bound - it's the most permissive from the caller's perspective (any closure satisfies it). Only tighten to `FnMut` or `Fn` when you actually need to call it multiple times.

Rust's closure system is one of those things that seems complex at first but has a clean underlying model. Closures are structs. The three traits describe how the struct's fields are accessed. The compiler picks the right trait. Everything else follows from that.
