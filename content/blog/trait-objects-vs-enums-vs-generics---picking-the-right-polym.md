+++
title = "Trait objects vs enums vs generics - picking the right polymorphism in Rust"
date = 2026-03-28
description = "When to use dyn Trait, enums, or generics for polymorphism in Rust - with vtable layouts, assembly comparisons, object safety rules, and a practical decision tree."

[taxonomies]
tags = ["rust", "polymorphism", "performance", "compiler-internals"]
+++

Rust gives you three ways to write code that operates on "different types that share behavior." Every time you reach for polymorphism, you're choosing between generics (static dispatch), trait objects (`dyn Trait`, dynamic dispatch), and enums (closed-set variants with pattern matching). Each compiles to fundamentally different machine code, has different constraints, and fits different problems.

Most Rust tutorials cover the syntax. This post covers the trade-offs: memory layout, performance characteristics, compile-time impact, and the rules that determine which approach you can even use. By the end, you'll have a decision tree you can apply without thinking twice.

<!-- more -->

## The running example

We need a concrete scenario. Say you're building a shape renderer. Different shapes, each with an `area()` method. Simple enough to illustrate the mechanics, complex enough to show real differences.

Here's the trait:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}
```

And two implementations:

```rust
struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "circle"
    }
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn name(&self) -> &str {
        "rectangle"
    }
}
```

Now watch how each polymorphism strategy handles "process a collection of shapes."

## Approach 1: generics (static dispatch)

Generics erase at compile time through monomorphization. If you read my [zero-cost abstractions post](/blog/zero-cost-abstractions-in-rust-what-it-actually-means/), you've already seen the mechanics - the compiler stamps out a concrete function for each type. Here's what that looks like for shapes:

```rust
fn print_area<S: Shape>(shape: &S) {
    println!("{}: {:.2}", shape.name(), shape.area());
}

fn main() {
    let c = Circle { radius: 5.0 };
    let r = Rectangle { width: 3.0, height: 4.0 };

    print_area(&c); // compiler generates print_area::<Circle>
    print_area(&r); // compiler generates print_area::<Rectangle>
}
```

The compiler produces two separate functions: `print_area::<Circle>` and `print_area::<Rectangle>`. Each contains a direct call (or inline) to the concrete `area()` implementation. No indirection, no pointer chasing, no vtable lookup.

The assembly for `print_area::<Circle>` on x86-64 (release mode) calls `area()` with something like:

```asm
vmulsd  xmm0, xmm1, xmm1    ; radius * radius
vmulsd  xmm0, xmm0, xmm2    ; * PI
```

Direct multiplication instructions. The compiler knows at compile time that `S` is `Circle`, knows the layout, knows the method address. It can inline, constant-fold, and vectorize.

**The limitation:** you can't have a `Vec<S>` that contains both circles and rectangles. A generic `Vec<S>` is monomorphized for one concrete type. `Vec<Circle>` or `Vec<Rectangle>` - not both. For heterogeneous collections, you need one of the other two approaches.

```rust
// This does NOT work:
fn process_shapes<S: Shape>(shapes: &[S]) {
    for s in shapes {
        println!("{:.2}", s.area());
    }
}

// You can call it with &[Circle] or &[Rectangle]
// but not a mixed collection of both.
```

**When generics shine:** processing a single known type at a time, library APIs that work across types, hot paths where the compiler needs full visibility for optimization.

## Approach 2: trait objects (dyn Trait)

When you prefix a trait with `dyn`, you're opting into runtime dispatch. The compiler doesn't generate specialized copies - it generates one function that works with any type implementing the trait, using a vtable to find the right method at runtime.

```rust
fn print_area_dyn(shape: &dyn Shape) {
    println!("{}: {:.2}", shape.name(), shape.area());
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 5.0 }),
        Box::new(Rectangle { width: 3.0, height: 4.0 }),
    ];

    for shape in &shapes {
        print_area_dyn(shape.as_ref());
    }
}
```

Now `shapes` holds circles and rectangles in the same `Vec`. The price: every call to `area()` goes through a vtable.

### The fat pointer

A `&dyn Shape` is not a regular reference. It's a *fat pointer* - 16 bytes on 64-bit systems instead of the usual 8. It consists of two pointers:

```
&dyn Shape (16 bytes):
  ┌─────────────┐
  │  data ptr    │ ──> the actual Circle or Rectangle on the heap/stack
  │  (8 bytes)   │
  ├─────────────┤
  │  vtable ptr  │ ──> static vtable for that concrete type's Shape impl
  │  (8 bytes)   │
  └─────────────┘
```

The data pointer points to the concrete value. The vtable pointer points to a compiler-generated static table.

### The vtable

Each (type, trait) combination produces one vtable, generated at compile time, stored in the binary's read-only data section. For `Circle` implementing `Shape`, the vtable looks like:

```
Shape vtable for Circle:
  ┌──────────────────────┐
  │  drop_in_place ptr   │  destructor (how to clean up a Circle)
  │  size: 8             │  size of Circle in bytes (one f64)
  │  align: 8            │  alignment of Circle
  │  area ptr            │ ──> Circle::area
  │  name ptr            │ ──> Circle::name
  └──────────────────────┘
```

The vtable stores the concrete type's `size`, `align`, and `drop` function - this is how Rust can drop a `Box<dyn Shape>` without knowing the concrete type. Then it lists function pointers for each trait method, in declaration order.

When you call `shape.area()` on a `&dyn Shape`, the generated assembly does:

```asm
mov     rax, [rdi + 8]       ; load vtable pointer from fat pointer
call    [rax + 24]           ; call 4th entry in vtable (area fn)
```

Two memory loads plus an indirect `call`. Compare that to the generic version's direct `vmulsd` - the indirect call prevents inlining, which prevents further optimizations (constant propagation, loop unrolling, SIMD vectorization).

### How Box<dyn Shape> is laid out

`Box<dyn Shape>` adds heap allocation on top:

```
Stack:                  Heap:
┌──────────────┐       ┌──────────────┐
│ data ptr     │ ────> │ radius: 5.0  │  (the Circle)
│ vtable ptr   │ ──┐   └──────────────┘
└──────────────┘   │
                   │   Static data (.rodata):
                   └─> ┌─────────────────┐
                       │ drop ptr         │
                       │ size: 8          │
                       │ align: 8         │
                       │ Circle::area ptr │
                       │ Circle::name ptr │
                       └─────────────────┘
```

Each `Box<dyn Shape>` is 16 bytes on the stack (the fat pointer) plus the concrete type's size on the heap. The vtable itself exists once per type, in the binary's read-only segment.

**When trait objects shine:** heterogeneous collections, plugin architectures, when the concrete type is decided at runtime (config-driven backends), and when you want to reduce binary size by avoiding monomorphization. If you've read the [repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/), that's exactly why `Arc<dyn DynRepository<User>>` exists - the storage backend is chosen at startup based on environment config, not known at compile time.

## Approach 3: enums

Enums encode a closed set of variants directly. No trait, no vtable, no indirection:

```rust
enum ShapeEnum {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
}

impl ShapeEnum {
    fn area(&self) -> f64 {
        match self {
            ShapeEnum::Circle { radius } => {
                std::f64::consts::PI * radius * radius
            }
            ShapeEnum::Rectangle { width, height } => {
                width * height
            }
        }
    }

    fn name(&self) -> &str {
        match self {
            ShapeEnum::Circle { .. } => "circle",
            ShapeEnum::Rectangle { .. } => "rectangle",
        }
    }
}
```

Now the collection is straightforward:

```rust
fn main() {
    let shapes = vec![
        ShapeEnum::Circle { radius: 5.0 },
        ShapeEnum::Rectangle { width: 3.0, height: 4.0 },
    ];

    for shape in &shapes {
        println!("{}: {:.2}", shape.name(), shape.area());
    }
}
```

No `Box`, no heap allocation, no fat pointers. Each element lives inline in the `Vec`'s contiguous buffer.

### Memory layout of enums

An enum's size is determined by its largest variant plus a discriminant tag:

```rust
use std::mem;

println!("Circle variant data:  {} bytes", mem::size_of::<f64>());      // 8
println!("Rectangle variant data: {} bytes", 2 * mem::size_of::<f64>()); // 16
println!("ShapeEnum size:       {} bytes", mem::size_of::<ShapeEnum>()); // 24
```

`ShapeEnum` is 24 bytes: 16 bytes for the largest variant's data (`Rectangle` has two `f64`s), plus 8 bytes for the discriminant (the compiler might use less, but alignment padding brings it to 24). Every element in a `Vec<ShapeEnum>` is 24 bytes, regardless of which variant it holds. A `Circle` variant wastes 8 bytes of padding.

Compare the memory layout of `Vec<ShapeEnum>` vs `Vec<Box<dyn Shape>>`:

```
Vec<ShapeEnum> (contiguous, cache-friendly):
┌──────────────────────────────────────────────┐
│ [tag|radius|pad] [tag|width|height] [tag|...] │
│  24 bytes each, inline, no indirection        │
└──────────────────────────────────────────────┘

Vec<Box<dyn Shape>> (pointer chasing):
Stack buffer:                    Heap (scattered):
┌─────────────────────┐          ┌──────────┐
│ data_ptr | vtbl_ptr │ ────────>│ Circle   │
│ data_ptr | vtbl_ptr │ ──┐     └──────────┘
│ ...                 │   │     ┌───────────┐
└─────────────────────┘   └────>│ Rectangle │
  16 bytes each                 └───────────┘
```

The enum version has better cache locality. Elements sit next to each other in contiguous memory. The trait object version requires following a pointer to the heap for each element - a cache miss on every access if the allocator scattered them.

### Exhaustive matching

The compiler guarantees you handle every variant. Add a `Triangle` to the enum and every `match` in your codebase produces a compile error until you handle it. This is the enum's killer feature for correctness:

```rust
enum ShapeEnum {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },  // new variant
}

// Every match block now fails to compile until you add the Triangle arm.
// The compiler tells you exactly which matches are incomplete.
```

With `dyn Shape`, adding a new `Triangle` struct that implements `Shape` requires no changes anywhere - it just works. Whether that's an advantage or a risk depends on your situation. If you *want* to be forced to handle every case (state machines, protocol messages, AST nodes), enums win. If you want open extensibility (plugins, user-provided types), trait objects win.

**When enums shine:** closed variant sets, state machines, AST nodes, message types, error types, anything where you know all possible variants at compile time and want exhaustive handling.

## Object safety - why not every trait can be dyn

Try this:

```rust
trait Cloneable {
    fn clone_self(&self) -> Self;
}

// This fails:
// fn take_cloneable(c: &dyn Cloneable) { ... }
// error[E0038]: the trait `Cloneable` is not dyn compatible
```

Not every trait can be used as `dyn Trait`. The Rust compiler enforces a set of rules called *dyn compatibility* (formerly "object safety"). The core issue is that a vtable is a fixed-size lookup table generated at compile time. Certain trait features make it impossible to construct such a table.

Here are the rules, and *why* each exists:

### Rule 1: methods can't return Self

```rust
trait Bad {
    fn clone_it(&self) -> Self;  // what size is the return value?
}
```

The vtable entry for `clone_it` needs a single function signature. But `Self` is `Circle` (8 bytes) for one implementation and `Rectangle` (16 bytes) for another. The caller doesn't know how much stack space to reserve for the return value. The compiler can't generate a single calling convention that works for all concrete types.

**Workaround:** return a `Box<dyn Bad>` instead, or add `where Self: Sized` to exclude the method from dynamic dispatch:

```rust
trait Better {
    fn clone_it(&self) -> Box<dyn Better>;
}

// Or exclude from vtable:
trait AlsoBetter {
    fn clone_it(&self) -> Self where Self: Sized;
    fn area(&self) -> f64; // this method IS in the vtable
}
```

### Rule 2: methods can't have generic type parameters

```rust
trait Bad {
    fn process<T: Display>(&self, item: T);
}
```

A vtable has a fixed number of entries. But `process::<String>`, `process::<i32>`, `process::<Vec<u8>>` - each is a different monomorphized function. The vtable would need infinite entries, one per possible `T`. That's not how vtables work.

**Workaround:** accept a trait object instead of a generic:

```rust
trait Better {
    fn process(&self, item: &dyn Display);
}
```

Now `process` has one signature, one vtable entry. The inner `dyn Display` handles the polymorphism.

### Rule 3: the trait can't require Self: Sized

```rust
trait Bad: Sized {
    fn area(&self) -> f64;
}
```

`dyn Bad` is unsized - its size isn't known at compile time. A supertrait bound of `Sized` means every implementor must be sized, but `dyn Bad` itself isn't, creating a contradiction.

### Rule 4: no associated constants

```rust
trait Bad {
    const MAX: usize;
    fn area(&self) -> f64;
}
```

Associated constants aren't function pointers - they can't live in a vtable. Each implementor could define a different value for `MAX`, but there's no vtable mechanism to look up a constant.

**Workaround:** use a method instead:

```rust
trait Better {
    fn max(&self) -> usize;
    fn area(&self) -> f64;
}
```

### Rule 5: no static methods (methods without a receiver)

```rust
trait Bad {
    fn new(radius: f64) -> Self;  // no &self, no self
    fn area(&self) -> f64;
}
```

A vtable is accessed through the fat pointer's vtable pointer, which comes from the object. A static method has no object, so there's no vtable to look up. You'd need to know the concrete type to call `new()`, which defeats the purpose of dynamic dispatch.

**Workaround:** add `where Self: Sized`:

```rust
trait Better {
    fn new(radius: f64) -> Self where Self: Sized;
    fn area(&self) -> f64; // still dyn-compatible
}
```

The `where Self: Sized` bound means "this method is only available when the concrete type is known." It's excluded from the vtable, and `dyn Better` simply doesn't have a `new()` method. The rest of the trait remains dyn-compatible.

### Rule 6: Self can't appear as a supertrait type parameter

```rust
trait Graph: PartialEq<Self> {
    // PartialEq<Self> means eq(&self, other: &Self)
    // But with dyn, what's Self? It's dyn Graph.
    // Can you compare a Circle to a Rectangle? The types don't match.
}
```

This creates the same sized-return problem. The supertrait's methods reference `Self`, which has unknown size behind `dyn`.

You'll hit these rules most often with `Clone` (returns `Self`), `Iterator` with generic `map`/`filter` chains (type parameters on methods), and constructor patterns (`fn new() -> Self`). The [E0038 error page](https://doc.rust-lang.org/error_codes/E0038.html) in rustc is the canonical reference for all violations.

## Performance: the numbers

Talk is cheap. Let me show you benchmark results. The [`enum_dispatch`](https://crates.io/crates/enum_dispatch) crate includes benchmarks that compare all three approaches on a `Vec` of 1024 elements with randomized concrete types:

| Approach | Time (ns/iter) | Relative speed |
|----------|---------------|----------------|
| `Vec<Box<dyn Trait>>` | 5,900,191 | 1.0x (baseline) |
| `Vec<&dyn Trait>` | 5,658,461 | 1.04x |
| `Vec<Enum>` (enum_dispatch) | 479,630 | **12.3x faster** |

That's not a typo. Enum dispatch is over 12x faster than boxed trait objects in this benchmark. Two factors drive the gap:

**1. No indirection.** A `Vec<Enum>` stores values inline. Iterating it is a sequential memory scan - the CPU prefetcher loves this. A `Vec<Box<dyn Trait>>` stores fat pointers, each pointing to a different heap allocation. Every element access is a pointer chase that may miss the L1 cache.

**2. The compiler can see through the match.** When the compiler encounters a `match` on an enum, it can inline each arm's code, apply constant folding, and potentially auto-vectorize the loop. With `dyn Trait`, the indirect `call` through the vtable is opaque - the compiler can't inline across it, can't see what the function does, can't optimize across the dispatch boundary.

This matches the pattern from the [zero-cost abstractions post](/blog/zero-cost-abstractions-in-rust-what-it-actually-means/) - when the compiler can see through the abstraction, it optimizes aggressively. When it can't (vtable indirection), you pay for it.

### Where the gap shrinks

Those benchmarks measure tight loops calling cheap methods. In real applications, the method body usually does real work - database queries, file I/O, network calls, complex computations. When the method body takes microseconds or milliseconds, the nanoseconds of vtable overhead are invisible.

```rust
// The vtable lookup here costs ~2ns.
// The HTTP request takes ~50ms.
// You're optimizing 0.000004% of the total time.
async fn fetch_data(source: &dyn DataSource) -> Result<Data, Error> {
    source.fetch("https://api.example.com/data").await
}
```

Profile before optimizing dispatch. If your method bodies are trivial and you're calling them millions of times in a tight loop, enum dispatch matters. If your method bodies do I/O, the dispatch mechanism is noise.

### Generics: the fastest path (when applicable)

Generics don't appear in the benchmark table because they're not directly comparable - a generic function processes one type at a time, not a heterogeneous collection. But for the single-type case, generics produce the fastest possible code:

```rust
fn total_area<S: Shape>(shapes: &[S]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}
```

This compiles to a tight loop with direct calls (or full inlining). No vtable, no match, no branch prediction misses. For `total_area::<Circle>`, the loop body might be just `vmulsd` + `vaddsd` - multiply and accumulate. The compiler can even auto-vectorize this with AVX instructions, processing 4 elements per cycle.

You get this performance because the compiler knows the exact type at every call site. It generates `total_area::<Circle>` and `total_area::<Rectangle>` as separate, fully-optimized functions. This is monomorphization doing its job - trading compile time and binary size for runtime speed.

## Compile time trade-offs

Each approach affects compile times differently:

**Generics** increase compile time proportionally to the number of concrete instantiations. If 20 different types implement `Shape` and you call `print_area::<T>` with each, the compiler generates 20 copies of `print_area`. Each copy goes through type checking, MIR generation, LLVM optimization, and code generation. For small functions this is fine. For large generic functions with deep call trees (think `serde`'s deserialize pipeline), monomorphization can explode compile times.

The standard library mitigates this with the inner-function pattern I showed in the [zero-cost abstractions post](/blog/zero-cost-abstractions-in-rust-what-it-actually-means/) - make the generic part a thin wrapper, put the real logic in a non-generic inner function.

**Trait objects** compile once. There's one `print_area_dyn` in the binary, regardless of how many types implement `Shape`. This is why library authors sometimes expose `dyn Trait` APIs for public interfaces - it keeps downstream compile times predictable. The [`std::io::Read`](https://doc.rust-lang.org/std/io/trait.Read.html) and [`std::io::Write`](https://doc.rust-lang.org/std/io/trait.Write.html) traits are used as `dyn Read` and `dyn Write` throughout the standard library for exactly this reason.

**Enums** fall in between. The `match` body is compiled once, but each arm may be inlined. The compiler doesn't duplicate the containing function per variant - it generates one function with a branch per arm. Compile time impact is minimal.

| Approach | Code duplication | Binary size | Compile time |
|----------|-----------------|-------------|--------------|
| Generics | N copies per N types | Largest | Slowest |
| Trait objects | 1 copy + vtables | Smallest | Fastest |
| Enums | 1 copy with match | Middle | Middle |

## The enum_dispatch crate

If you want enum performance with trait syntax, [`enum_dispatch`](https://crates.io/crates/enum_dispatch) bridges the gap. It generates enum boilerplate from a trait definition:

```rust
use enum_dispatch::enum_dispatch;

#[enum_dispatch]
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

#[enum_dispatch(Shape)]
enum ShapeEnum {
    Circle,
    Rectangle,
}

struct Circle { radius: f64 }
struct Rectangle { width: f64, height: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius }
    fn name(&self) -> &str { "circle" }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 { self.width * self.height }
    fn name(&self) -> &str { "rectangle" }
}
```

The macro generates the `match`-based `Shape` impl for `ShapeEnum` automatically. You get trait-based code organization with enum-level performance. The trade-off: all variants must be known at compile time (it's still a closed set), and both the trait and enum must live in the same crate.

## Real-world patterns

The abstract comparison only goes so far. Here's when each approach actually shows up in production code:

### Generics: library APIs and hot paths

The standard library is full of generics. `HashMap<K, V>`, `Vec<T>`, `Iterator::map()`, `Result<T, E>` - these are all monomorphized for each concrete use. [Serde](https://serde.rs/) uses generics for its `Serialize`/`Deserialize` traits so that serialization code compiles down to direct field access, no dynamic dispatch overhead.

Use generics when you're writing a library that downstream users will call with their own types, or when you're on a hot path and need the compiler to optimize aggressively.

### Trait objects: plugin systems and dependency injection

When the concrete type is chosen at runtime, you need `dyn`. This shows up in:

- **Repository pattern** - `Arc<dyn Repository<User>>` where the backend (SQLite, Postgres, in-memory) is picked from config. I covered this in the [repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/).
- **Middleware stacks** - `Vec<Box<dyn Middleware>>` in web frameworks where users register middleware at startup.
- **Plugin architectures** - loading shared libraries (`.so`/`.dll`) at runtime. The loaded code returns a `Box<dyn Plugin>`. The host application can't know the type at compile time.
- **Reducing binary size** - embedded systems or WASM targets where every kilobyte counts. A single `dyn` dispatch function is smaller than 20 monomorphized copies.

### Enums: closed domains

Enums dominate when you control all variants:

- **AST nodes** - a parser produces `Expr::Binary`, `Expr::Literal`, `Expr::Unary`, etc. The set of node types is fixed by the language grammar. I used this in my [JSON parser](/blog/writing-a-json-parser-from-scratch-in-rust/) and [Markdown parser](/blog/writing-a-markdown-parser-in-rust/) posts.
- **Protocol messages** - `Message::Ping`, `Message::Data`, `Message::Close`. The protocol spec defines all variants.
- **Error types** - `thiserror` enums with one variant per error category. You've seen `RepoError` in the [repository post](/blog/the-repository-pattern-abstracting-data-access-in-rust/) - three variants, exhaustive matching.
- **State machines** - `State::Idle`, `State::Processing`, `State::Done`. Exhaustive `match` ensures you handle every state transition.

## The decision tree

When you need polymorphism, walk this tree:

```
Is the set of types known at compile time?
├── NO ──> dyn Trait (trait objects)
│          You need runtime dispatch.
│          Example: plugin systems, config-driven backends
│
└── YES
    │
    Do you need a heterogeneous collection (mixed types in one Vec)?
    ├── NO ──> Generics
    │          The compiler monomorphizes; zero overhead.
    │          Example: library APIs, hot paths processing one type
    │
    └── YES
        │
        Is the set of variants closed (you control all of them)?
        ├── YES ──> Enum
        │           Best performance, exhaustive matching, no heap allocation.
        │           Example: AST nodes, state machines, message types
        │
        └── NO ──> dyn Trait (trait objects)
                   Third-party code might add new implementations.
                   Example: trait in a public library, extensible systems
```

And two override rules:

1. **If binary size or compile time is a constraint**, consider `dyn Trait` even when generics would work. The cost is a few nanoseconds per call; the savings are kilobytes of binary and seconds of compile time.

2. **If you're in a tight loop over millions of elements**, use enums or generics. The vtable overhead compounds. Profile first - but if dispatch is the bottleneck, switch away from `dyn`.

## Combining approaches

These aren't mutually exclusive. Real systems mix them:

```rust
// Generic function that accepts any Shape...
fn biggest_shape<S: Shape>(shapes: &[S]) -> Option<&S> {
    shapes.iter().max_by(|a, b| {
        a.area().partial_cmp(&b.area()).unwrap()
    })
}

// ...but the application stores shapes as an enum for cache locality
enum ShapeEnum {
    Circle(Circle),
    Rectangle(Rectangle),
}

// ...and accepts plugins via dyn Trait
struct Renderer {
    shapes: Vec<ShapeEnum>,            // owned shapes, enum for speed
    plugins: Vec<Box<dyn RenderPlugin>>, // extensible, runtime dispatch
}
```

The observer pattern post [I wrote earlier](/blog/the-observer-pattern-in-rust-events-without-callbacks-hell/) shows this in action - the `TypedEventBus` uses `HashMap<TypeId, Vec<Box<dyn Fn(&dyn Any)>>>` internally (trait objects for type erasure) while exposing a generic API externally (`subscribe<E>`, `emit<E>`). The public interface is statically typed; the internal storage uses dynamic dispatch. Each approach plays its role.

## Summary

| | Generics | Trait objects (`dyn`) | Enums |
|---|---|---|---|
| **Dispatch** | Static (compile time) | Dynamic (runtime vtable) | Static (match) |
| **Heterogeneous collections** | No | Yes | Yes |
| **Extensible** (new types later) | Yes | Yes | No (closed set) |
| **Heap allocation** | No | Usually (`Box<dyn>`) | No |
| **Cache locality** | Best (inline) | Worst (scattered heap) | Good (inline, with padding) |
| **Compiler can inline** | Yes | No (opaque vtable call) | Yes (each match arm) |
| **Binary size** | Grows with types | Constant | Constant |
| **Compile time** | Grows with types | Constant | Constant |
| **Exhaustive checking** | N/A | No | Yes |
| **Object safety required** | No | Yes | No |

Generics give you maximum performance and zero abstraction cost. Trait objects give you runtime flexibility. Enums give you exhaustive safety and cache-friendly layouts. Know which problem you're solving, and the right choice is obvious.
