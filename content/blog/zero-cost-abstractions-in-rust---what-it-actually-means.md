+++
title = "Zero-cost abstractions in Rust - what it actually means"
date = 2025-05-21
description = "What 'zero-cost' really means under the hood: iterator chains, monomorphization, newtypes, inline hints, and when abstractions are NOT free."

[taxonomies]
tags = ["rust", "performance", "compiler-internals"]
+++

"Zero-cost abstractions" is probably the most repeated phrase in any Rust pitch. It shows up in conference talks, README files, job postings, and roughly every other Reddit thread about the language. But ask someone what it actually means and you'll usually get a vague answer about "no runtime overhead" followed by hand-waving toward iterators.

The concept comes from Bjarne Stroustrup, who laid down two rules for C++: "What you don't use, you don't pay for. And further: what you do use, you couldn't hand code any better." Rust adopted this principle and, in many cases, delivers on it more consistently than C++ does. But "zero-cost" doesn't mean "zero tradeoffs," and understanding where the boundary lies is what separates knowing the slogan from knowing the language.

<!-- more -->

## The textbook example: iterator chains

Every explanation of zero-cost abstractions starts with iterators, and for good reason. They're the clearest demonstration of high-level code compiling down to the same machine code as a manual loop.

Consider summing the squares of even numbers in a vector:

```rust
fn sum_even_squares(v: &[i64]) -> i64 {
    v.iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * x)
        .sum()
}
```

The "hand-written" equivalent:

```rust
fn sum_even_squares_manual(v: &[i64]) -> i64 {
    let mut total = 0i64;
    for &x in v {
        if x % 2 == 0 {
            total += x * x;
        }
    }
    total
}
```

Throw both of these into [Godbolt](https://rust.godbolt.org/) with `-O` (or compile locally with `cargo rustc --release -- --emit asm`). The generated x86-64 assembly is functionally identical. No function calls for `filter` or `map`. No intermediate allocations. No closure structs in memory. The compiler fuses the entire chain into a single loop that reads each element, checks divisibility, multiplies, and accumulates.

The assembly for the hot loop looks roughly like this on x86-64:

```asm
.LBB0_4:
    mov     rax, qword ptr [rdi]       ; load element
    test    al, 1                       ; check if even (bit 0)
    jne     .LBB0_6                    ; skip if odd
    imul    rax, rax                    ; square it
    add     rcx, rax                    ; accumulate
.LBB0_6:
    add     rdi, 8                      ; next element
    cmp     rdi, rsi                    ; reached the end?
    jne     .LBB0_4                    ; loop
```

Five instructions in the hot path. No `call` instructions, no indirect jumps. The iterator chain, the closures, the `filter` and `map` adaptor structs - all gone. The compiler saw through every layer.

How does this work? Three things happen:

1. **Monomorphization** - the compiler generates a concrete version of `Iterator::sum()` for the specific `Map<Filter<Iter<i64>>>` type. No generics remain at runtime.
2. **Inlining** - each adaptor's `next()` method is tiny and gets inlined into the caller. The chain collapses into a single function.
3. **LLVM optimization** - after inlining, LLVM sees a simple loop pattern and applies standard optimizations: dead code elimination, register allocation, loop simplification.

The Rust Book's [performance chapter](https://doc.rust-lang.org/book/ch13-04-performance.html) uses an audio decoder as an example. The compiler knows the loop has a fixed number of iterations, unrolls it, and eliminates loop control overhead entirely. The iterator version benchmarks identically to the hand-unrolled version.

This is what "you couldn't hand code any better" looks like. The abstraction gave you readability and composability. The compiled output gave up nothing.

## Monomorphization: the compiler's copy machine

The engine behind most zero-cost abstractions in Rust is monomorphization. When you write a generic function:

```rust
fn double<T: std::ops::Mul<Output = T> + Copy>(x: T) -> T {
    x * x
}

fn main() {
    let a = double(3i32);
    let b = double(2.5f64);
}
```

The compiler doesn't generate one function that handles arbitrary types at runtime. It generates two functions: `double::<i32>` and `double::<f64>`. Each is specialized for its concrete type, using the right CPU instructions (`imul` for integers, `mulsd` for doubles). No type checks, no vtable lookups, no boxing.

This is fundamentally different from how Java generics work (type erasure, everything goes through `Object`) or how Go interfaces work (implicit vtables). Rust pays for this at compile time - more code to generate, more functions to optimize - but at runtime, the generic function call is as fast as if you'd written a non-generic version by hand.

You can see monomorphization's effect on binary size directly. Compile this:

```rust
fn print_it<T: std::fmt::Display>(val: T) {
    println!("{val}");
}

fn main() {
    print_it(42i32);
    print_it(3.14f64);
    print_it("hello");
    print_it(true);
}
```

Run `nm` or `objdump` on the release binary and you'll find four distinct `print_it` symbols. The compiler duplicated the function body four times. That's the tradeoff: monomorphization trades binary size and compile time for runtime speed. It's not free - it's just that the cost is paid before your program runs.

If you've read my post on [closures](/blog/closures-in-rust-fn-fnmut-fnonce-demystified/), you've already seen this in action. Each closure gets a unique anonymous struct type, and generic functions accepting `impl Fn(...)` get monomorphized for each closure. That's why `iter().map(|x| x * 2)` compiles to the same code as a `for` loop - the closure type is known, the call is inlined, the abstraction vanishes.

## The newtype pattern: type safety that vanishes

The newtype pattern wraps an existing type in a single-field struct to give it a distinct type identity:

```rust
struct UserId(u64);
struct ProductId(u64);

fn get_user(id: UserId) -> String {
    format!("user_{}", id.0)
}

fn main() {
    let user = UserId(42);
    let product = ProductId(99);

    get_user(user);      // compiles
    // get_user(product); // compile error: expected UserId, found ProductId
}
```

Both `UserId` and `ProductId` are `u64` underneath, but the type system won't let you mix them up. You can't accidentally pass a product ID where a user ID is expected.

The key insight: the wrapper has exactly the same memory layout as the inner type. Rust guarantees this for single-field structs (and you can be explicit about it with `#[repr(transparent)]`). At runtime, `UserId(42)` is just the number `42` in a register. No indirection, no wrapper overhead, no extra memory.

Check it yourself:

```rust
use std::mem;

struct Meters(f64);

fn main() {
    println!("f64 size: {}", mem::size_of::<f64>());       // 8
    println!("Meters size: {}", mem::size_of::<Meters>());  // 8

    println!("f64 align: {}", mem::align_of::<f64>());      // 8
    println!("Meters align: {}", mem::align_of::<Meters>()); // 8
}
```

Same size, same alignment. The compiler treats them identically in the generated code. The `Meters` type exists only for the type checker - it's erased completely by the time you reach machine instructions.

This extends to conversions too. If you implement `From<f64> for Meters`:

```rust
impl From<f64> for Meters {
    fn from(val: f64) -> Self {
        Meters(val)
    }
}
```

In release mode, `Meters::from(3.14)` compiles to... nothing. It's a no-op. The value is already in the right register with the right representation. The compiler proves the conversion is a type-level identity and eliminates it entirely.

The newtype pattern is zero-cost in the purest sense: the abstraction provides compile-time guarantees (type safety, trait implementations isolated to the wrapper) and produces literally identical machine code to using the raw type.

## Inline hints: helping the compiler see through boundaries

Inlining is the optimization that makes most zero-cost abstractions possible. When the compiler inlines a function call, it replaces the `call` instruction with the function's body at the call site. This enables further optimizations: constant propagation, dead code elimination, loop fusion.

The compiler does this automatically for small functions, but it can't always make the right call. That's where `#[inline]` attributes come in:

```rust
#[inline]
fn is_even(x: i64) -> bool {
    x % 2 == 0
}

#[inline(always)]
fn square(x: i64) -> i64 {
    x * x
}

#[inline(never)]
fn log_result(result: i64) {
    println!("Result: {result}");
}
```

Three levels:

- **`#[inline]`** - suggest inlining. The compiler takes it as a hint but can still decide not to. The main effect is making the function's body available for [cross-crate inlining](https://nnethercote.github.io/perf-book/inlining.html) - without it, a function in a library crate can't be inlined by code in a dependent crate, because the compiler doesn't have the function body available.
- **`#[inline(always)]`** - strongly suggest inlining. The compiler will inline in all but the most exceptional cases. Use this sparingly and only for trivial functions where a benchmark justifies it.
- **`#[inline(never)]`** - prevent inlining. Useful for cold paths (error handling, logging) where inlining would bloat the hot code and hurt instruction cache performance.

The critical detail people miss: **generic functions are already available for cross-crate inlining**, because monomorphization requires the function body to be present at the call site anyway. You don't need `#[inline]` on generic functions. It's non-generic, public functions in library crates where `#[inline]` makes a difference.

The [Rust standard library developer guide](https://std-dev-guide.rust-lang.org/policy/inline.html) has a clear policy: put `#[inline]` on small, non-generic functions. Don't sprinkle it everywhere. If you don't care about fine-grained control and just want maximum optimization across crate boundaries, setting `lto = true` in your `Cargo.toml` profile is often a better approach:

```toml
[profile.release]
lto = true
```

Link-Time Optimization gives the compiler visibility across all crates at once, enabling inlining decisions that `#[inline]` hints can't achieve on their own. The cost is longer compile times.

## When zero-cost is not zero-cost

Here's the part most "zero-cost abstractions" explanations skip. There are real, measurable cases where Rust abstractions carry overhead. Understanding them is more useful than memorizing the success stories.

### Dynamic dispatch: the vtable tax

When you use `dyn Trait`, you're explicitly opting out of monomorphization:

```rust
fn apply(f: &dyn Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

fn apply_generic(f: impl Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}
```

The generic version compiles to a direct call (or is inlined entirely). The `dyn` version generates an indirect call through a vtable pointer - load the vtable, dereference the function pointer, jump. That's two extra memory loads, and more importantly, the compiler can't inline across the indirection because it doesn't know at compile time which function is behind the pointer.

I covered this in more detail in the [closures post](/blog/closures-in-rust-fn-fnmut-fnonce-demystified/) - look at the assembly comparison between `impl Fn` and `Box<dyn Fn>`. The `impl Fn` path produces a direct `call` instruction (or disappears entirely via inlining), while `Box<dyn Fn>` produces `call qword ptr [rax + 24]` plus a heap allocation.

This is by design. Dynamic dispatch isn't a bug - it's an explicit choice you make when you need runtime polymorphism. The [repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/) uses `Arc<dyn DynRepository>` intentionally, because the concrete implementation is determined at startup based on configuration. That vtable call per database operation is invisible compared to the actual I/O. Static dispatch matters in hot loops over millions of iterations, not in code that does a network round-trip per call.

The rule: use generics (static dispatch) on hot paths. Use `dyn Trait` on cold paths where flexibility matters more than the nanoseconds saved by inlining.

### Async trait boxing

Until native `async fn` in `dyn Trait` is fully stabilized (it's been a long road - see Niko Matsakis's [box box box](https://smallcultfollowing.com/babysteps/blog/2025/03/24/box-box-box/) post for the complexities), the common workaround with the [`async-trait`](https://crates.io/crates/async-trait) crate transforms every async method into one that returns `Pin<Box<dyn Future + Send>>`:

```rust
#[async_trait::async_trait]
trait Store {
    async fn get(&self, key: &str) -> Option<String>;
}
```

This desugars to roughly:

```rust
trait Store {
    fn get<'a>(&'a self, key: &'a str) -> Pin<Box<dyn Future<Output = Option<String>> + Send + 'a>>;
}
```

Every call to `get()` allocates a `Box`. On the heap. Every time. That's not zero-cost by any definition. For most applications - a web server handling requests, a CLI tool - the allocation is negligible compared to the actual I/O. But for embedded systems with no allocator, or tight loops calling an async function millions of times, it's a real cost.

Native `async fn in trait` (stabilized in Rust 1.75) avoids this for static dispatch:

```rust
trait Store {
    async fn get(&self, key: &str) -> Option<String>;
}

fn use_store(s: impl Store) {
    // no Box, no heap allocation - monomorphized
}
```

But using the same trait with `dyn Store` still requires some form of boxing for the future, because the compiler doesn't know the future's size at compile time. The language team is working on solutions (inline storage, stack allocation), but as of now, `dyn` + `async` = heap allocation.

### The SIMD wall: when "zero-cost per call" hides a bigger cost

This is the most subtle case, and it was demonstrated brilliantly by [turbopuffer's engineering blog](https://turbopuffer.com/blog/zero-cost) in February 2026.

They had a merge iterator - a standard pattern for combining sorted streams. Their `Iterator::next()` implementation was correct and, in isolation, zero-cost: each call to `next()` compiled to roughly seven x86 instructions. You couldn't hand-code a single step of the merge any better.

But here's the problem. The `Iterator` trait's `next()` returns one element at a time. Each call mutates internal state (positions within the source iterators) and the next call's output depends on the state left by the previous call. This sequential dependency makes it impossible for the compiler to:

- **Unroll** the loop - it can't predict what `next()` returns without executing the previous call
- **Vectorize** with SIMD - processing 4 or 8 elements in parallel requires independent operations

Their merge iterator was zero-cost *per call*. But the abstraction boundary between calls hid the loop shape that SIMD needs. The compiler could see inside one `next()` call perfectly, but it couldn't see across calls.

The fix was batched iteration - instead of returning one element, fill a contiguous buffer of 512 key-value pairs, then process the batch with a tight loop over an array. The inner loop over the batch was simple enough for the compiler to auto-vectorize with SIMD instructions.

Result: the merge operation went from 6.5ms per 100K values to ~110 microseconds. A 60x speedup. Their customer's full-text search query dropped from 220ms to 47ms. Not because the original code was wrong, but because the "zero-cost" abstraction boundary prevented a higher-level optimization.

This is a nuanced point worth internalizing: **an abstraction can be zero-cost at its own level while preventing optimizations at a higher level**. The Iterator trait compiles each `next()` call perfectly. But the one-at-a-time contract means the compiler never sees the batch-level pattern where the real performance lives.

### Compile time and binary size: the costs nobody measures

Monomorphization is zero-cost *at runtime*. At compile time, it's the opposite. Every instantiation of a generic function creates a new copy in the binary. Call `HashMap<String, i32>`, `HashMap<String, String>`, and `HashMap<u64, Vec<u8>>` and you get three complete copies of HashMap's implementation.

This has two measurable effects:

1. **Compile time** - more code to generate, optimize, and link. Heavily generic codebases (think anything using `serde`, `tokio`, or `diesel`) can have noticeably longer compile times because of monomorphization explosion.
2. **Binary size** - each monomorphized instance takes space. For embedded systems or WebAssembly targets where binary size matters, this is a real constraint.

The standard library itself uses a technique to mitigate this: implement the core logic in a non-generic inner function and make the generic outer function a thin wrapper that calls it. You'll see this pattern in [`std::fs::read`](https://github.com/rust-lang/rust/blob/master/library/std/src/fs.rs):

```rust
pub fn read<P: AsRef<Path>>(path: P) -> io::Result<Vec<u8>> {
    fn inner(path: &Path) -> io::Result<Vec<u8>> {
        let mut file = File::open(path)?;
        let size = file.metadata().map(|m| m.len() as usize).ok();
        let mut buf = Vec::with_capacity(size.unwrap_or(0));
        file.read_to_end(&mut buf)?;
        Ok(buf)
    }
    inner(path.as_ref())
}
```

The generic part (accepting `AsRef<Path>`) is a single-line wrapper. The real logic lives in a non-generic `inner()` function that gets compiled once. No matter how many different types you call `fs::read` with, the heavy code exists only once in the binary.

## The mental model

Zero-cost abstractions in Rust are real, but they're not magic. They work because of a specific combination of language design choices:

- **Ownership and borrowing** eliminate the need for runtime garbage collection
- **Monomorphization** eliminates runtime type dispatch for generics
- **Move semantics** and **no implicit copies** mean the compiler knows exactly where data lives
- **A strong inlining pipeline** through LLVM collapses abstraction layers

When all of these align - and they do in the common cases of iterators, closures, newtypes, and generic functions - you genuinely get machine code that's as good as hand-written C. When they don't align - `dyn Trait`, async boxing, abstraction boundaries that block vectorization - you're paying a cost, and you should know about it.

The honest version of "zero-cost abstractions" isn't "abstractions are free." It's "the compiler is really good at removing abstraction layers, and the language is designed to make that removal possible." That's less catchy, but it's the truth that helps you write faster code.
