+++
title = "Unsafe Rust - what it enables and when you need it"
date = 2025-05-04
description = "Unsafe doesn't mean dangerous - it means the compiler can't verify your guarantees. Here's what it unlocks, when you actually need it, and how to wrap it safely."

[taxonomies]
tags = ["rust", "unsafe", "ffi", "systems-programming"]
+++

There's a persistent misconception that `unsafe` in Rust means "this code is dangerous, stay away." It doesn't. It means "the compiler can't verify this invariant, so the programmer is taking responsibility." That's a very different statement. Safe Rust code can still panic, deadlock, leak memory, and overwrite files. And unsafe Rust code can be perfectly correct - the entire standard library is proof of that.

Every `Vec<T>` you've ever used, every `String`, every `Arc` and `Mutex` - they're all built on top of `unsafe` internally. The point isn't to avoid `unsafe`. It's to understand exactly what it enables, what the rules are, and how to build safe abstractions on top of it.

<!-- more -->

## The five superpowers

Inside an `unsafe` block (or an `unsafe fn`), you get access to five operations that safe Rust doesn't allow. The [Rust Reference](https://doc.rust-lang.org/reference/unsafety.html) calls these "unsafe superpowers":

1. Dereference raw pointers
2. Call `unsafe` functions and methods
3. Access or modify mutable static variables
4. Implement `unsafe` traits
5. Access fields of `union`s

That's it. Five things. Everything else - the borrow checker, lifetime rules, type checking, bounds checking on slices - all of that still applies inside `unsafe`. You don't get to turn off the compiler. You get to do five specific things that require human verification.

Let's look at each one in actual code.

### 1. Raw pointers

Raw pointers (`*const T` and `*mut T`) are Rust's version of C pointers. You can create them in safe code, but dereferencing them requires `unsafe`:

```rust
fn main() {
    let mut value = 42;

    // Creating raw pointers is safe
    let r1: *const i32 = &raw const value;
    let r2: *mut i32 = &raw mut value;

    // Dereferencing is not
    unsafe {
        println!("r1 = {}", *r1);
        *r2 = 99;
        println!("r2 = {}", *r2);
    }
}
```

Why are raw pointers unsafe to dereference? Because the compiler can't guarantee they point to valid memory. Raw pointers can be null. They can dangle. They can alias mutable references. They ignore all borrowing rules.

But they're essential when you need to do things the borrow checker can't reason about - like building a doubly-linked list where each node points both forward and backward. Safe Rust literally can't express that ownership pattern, because who owns the node? The previous node? The next one? Both?

Here's the actual representation. A safe `&T` reference and a `*const T` raw pointer have the exact same memory layout - a single pointer-sized integer (8 bytes on 64-bit). The only difference is what the compiler checks at compile time:

```rust
use std::mem::{size_of, align_of};

fn main() {
    // Same size, same alignment
    assert_eq!(size_of::<&i32>(), size_of::<*const i32>());       // 8
    assert_eq!(align_of::<&i32>(), align_of::<*const i32>());     // 8

    // Fat pointers for slices are 16 bytes (pointer + length)
    assert_eq!(size_of::<&[i32]>(), 16);
    assert_eq!(size_of::<*const [i32]>(), 16);
}
```

At the machine code level, dereferencing a `&T` and a `*const T` generate the exact same `mov` instruction. The safety is purely a compile-time construct. Zero runtime cost.

### 2. Calling unsafe functions

An `unsafe fn` is a function that has preconditions the compiler can't check. The caller is responsible for upholding them:

```rust
/// Reinterprets a `&[u8]` as a `&str` without checking UTF-8 validity.
///
/// # Safety
///
/// The caller must ensure `bytes` contains valid UTF-8.
unsafe fn str_from_bytes_unchecked(bytes: &[u8]) -> &str {
    unsafe { std::str::from_utf8_unchecked(bytes) }
}

fn main() {
    let bytes = b"hello world";

    // Safe version - validates UTF-8, returns Result
    let s1 = std::str::from_utf8(bytes).unwrap();

    // Unsafe version - trusts the caller
    let s2 = unsafe { str_from_bytes_unchecked(bytes) };

    assert_eq!(s1, s2);
}
```

The `// # Safety` doc comment isn't decoration. It's the contract. Every `unsafe fn` in well-written Rust documents exactly what the caller must guarantee. If you violate that contract, you get undefined behavior - not a panic, not an error, but the compiler and runtime making no guarantees about what happens.

The standard library is full of these. [`slice::from_raw_parts`](https://doc.rust-lang.org/std/slice/fn.from_raw_parts.html) requires the pointer to be valid for `len * size_of::<T>()` bytes. [`Vec::set_len`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.set_len) requires the new length to be less than or equal to capacity and all elements up to the new length to be initialized. Break those contracts and you're in UB territory.

### 3. Mutable statics

Global mutable state is unsafe because the compiler can't prevent data races:

```rust
static mut REQUEST_COUNT: u64 = 0;

/// # Safety
///
/// Must only be called from a single thread.
unsafe fn increment_count() {
    unsafe {
        REQUEST_COUNT += 1;
    }
}

fn main() {
    // Single-threaded - this is fine
    unsafe {
        increment_count();
        increment_count();
        println!("count: {}", *(&raw const REQUEST_COUNT));
    }
}
```

In practice, you almost never need `static mut`. Use `std::sync::atomic` types or `std::sync::OnceLock` instead:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

static REQUEST_COUNT: AtomicU64 = AtomicU64::new(0);

fn increment_count() {
    REQUEST_COUNT.fetch_add(1, Ordering::Relaxed);
}

fn main() {
    increment_count();
    increment_count();
    println!("count: {}", REQUEST_COUNT.load(Ordering::Relaxed));
}
```

Same result, no `unsafe`, thread-safe. The atomic version compiles to a single `lock xadd` instruction on x86. The `static mut` version compiles to a plain `add` - faster, but wrong if two threads call it simultaneously.

### 4. Unsafe traits

An `unsafe trait` is a trait with invariants that the compiler can't verify. The canonical examples are [`Send`](https://doc.rust-lang.org/std/marker/trait.Send.html) and [`Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html):

```rust
/// A type that wraps a raw pointer but guarantees
/// the pointer is always valid and the data is only
/// accessed from one thread at a time.
struct JsonBuffer {
    ptr: *mut u8,
    len: usize,
}

// The compiler won't auto-implement Send because *mut u8 is !Send.
// We promise the invariants hold.
unsafe impl Send for JsonBuffer {}
```

If you've worked with [trait bounds](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg/), you know `Send` means "safe to transfer between threads" and `Sync` means "safe to share references between threads." The compiler auto-derives these based on the fields of your struct. But when your struct contains raw pointers (which are `!Send` and `!Sync` by default), you sometimes need to override the compiler's conservative choice.

Getting this wrong is how you get data races. `unsafe impl Send` is a promise, and the compiler trusts you completely.

### 5. Union field access

Unions store multiple types in the same memory location, like C unions. Reading a field is unsafe because Rust can't know which variant is currently stored:

```rust
#[repr(C)]
union IntOrFloat {
    i: i32,
    f: f32,
}

fn main() {
    let val = IntOrFloat { i: 42 };

    unsafe {
        println!("as int: {}", val.i);
        // Reinterprets the same 4 bytes as a float
        println!("as float: {}", val.f);
    }
}
```

Unions are mostly used in FFI when interfacing with C code that uses them. In pure Rust, you'd use an `enum` instead - Rust enums are tagged unions with the compiler tracking which variant is active.

## What unsafe does NOT disable

This is worth being explicit about. Inside an `unsafe` block, all of the following still apply:

- **Borrow checker**: you can't have `&mut T` and `&T` simultaneously (unless you use raw pointers)
- **Lifetime checking**: references still can't outlive their data
- **Type checking**: you can't assign a `String` to a `u32`
- **Bounds checking**: `vec[i]` still panics on out-of-bounds (use `.get_unchecked()` if you want to skip it)
- **Move semantics**: values still get moved when ownership transfers

`unsafe` is a scalpel, not a sledgehammer. It relaxes exactly five constraints and nothing else.

## When you actually need unsafe

After working with enough Rust codebases, the legitimate use cases for `unsafe` fall into a few clear categories.

### FFI - calling into C or being called from C

This is the most common and most justified use. If you're wrapping a C library, every call crosses the FFI boundary and requires `unsafe`:

```rust
// Linking to the C standard library's abs function
unsafe extern "C" {
    fn abs(input: i32) -> i32;
}

fn absolute_value(x: i32) -> i32 {
    unsafe { abs(x) }
}

fn main() {
    println!("{}", absolute_value(-42)); // 42
}
```

Real-world examples: the [`openssl`](https://crates.io/crates/openssl) crate wraps libssl, [`libz-sys`](https://crates.io/crates/libz-sys) wraps zlib, and the [`windows`](https://crates.io/crates/windows) crate wraps the entire Win32 API. The `windows` crate is actually the single crate with the most `unsafe` usage on crates.io - because every Win32 API call is an FFI call.

Going the other direction, exposing Rust functions to C is common for embedding Rust in existing systems:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn rust_process_buffer(ptr: *const u8, len: usize) -> i32 {
    if ptr.is_null() {
        return -1;
    }
    let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
    // Process the data safely from here on
    slice.iter().map(|&b| b as i32).sum()
}
```

### Performance-critical data structures

Some data structures can't be expressed efficiently (or at all) with safe Rust's ownership model.

Take `Vec::push`. Here's what the [actual implementation](https://github.com/rust-lang/rust/blob/main/library/alloc/src/vec/mod.rs) looks like (simplified):

```rust
impl<T> Vec<T> {
    pub fn push(&mut self, value: T) {
        if self.len == self.capacity() {
            self.grow(); // reallocate
        }
        unsafe {
            let end = self.as_mut_ptr().add(self.len);
            std::ptr::write(end, value);
            self.len += 1;
        }
    }
}
```

Why `unsafe`? Because `ptr::write` writes a value to a raw pointer without dropping whatever was there before. The code knows the slot at `self.len` is uninitialized memory - there's nothing to drop. But the compiler doesn't know that. The programmer maintains the invariant that `self.len` always equals the count of initialized elements.

Another classic: [`std::collections::LinkedList`](https://doc.rust-lang.org/src/alloc/collections/linked_list.rs.html) uses raw pointers internally because, as I mentioned earlier, doubly-linked lists have shared ownership that the borrow checker can't express.

And it's not just the standard library. Tokio stores channel wakers in an intrusive linked list using raw pointers - I covered how tokio's internals work in [Understanding Tokio](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/) if you want more context on that runtime architecture.

### SIMD and hardware intrinsics

When you need to use CPU-specific instructions:

```rust
#[cfg(target_arch = "x86_64")]
fn sum_simd(data: &[f32]) -> f32 {
    #[cfg(target_arch = "x86_64")]
    use std::arch::x86_64::*;

    if data.len() < 8 {
        return data.iter().sum();
    }

    unsafe {
        let mut acc = _mm256_setzero_ps();
        let chunks = data.chunks_exact(8);
        let remainder = chunks.remainder();

        for chunk in chunks {
            let v = _mm256_loadu_ps(chunk.as_ptr());
            acc = _mm256_add_ps(acc, v);
        }

        // Horizontal sum of the 8 lanes
        let hi = _mm256_extractf128_ps(acc, 1);
        let lo = _mm256_castps256_ps128(acc);
        let sum128 = _mm_add_ps(hi, lo);
        let shuf = _mm_movehdup_ps(sum128);
        let sum64 = _mm_add_ss(sum128, shuf);
        let shuf2 = _mm_movehl_ps(shuf, sum64);
        let result = _mm_add_ss(sum64, shuf2);

        let mut total = _mm_cvtss_f32(result);
        total += remainder.iter().sum::<f32>();
        total
    }
}
```

The compiler can't verify that your CPU actually supports AVX2 at compile time (that's a runtime property), so these intrinsics are `unsafe`.

### Implementing `Pin`-based contracts

If you've worked with async Rust, you know about `Pin<T>`. Implementing `Future` by hand for self-referential types requires `unsafe` - because the compiler can't prove the memory won't move. The [phantom types post](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost) covered how `PhantomData` encodes compile-time information; `Pin` is in the same family but enforces a runtime property (address stability) through the type system, backed by `unsafe` implementations of `Unpin`.

## When you do NOT need unsafe

This is the more important section. Most Rust code should never contain an `unsafe` block.

**"I need to mutate through a shared reference"** - Use `Cell<T>`, `RefCell<T>`, `Mutex<T>`, or `RwLock<T>`. Interior mutability is a first-class pattern in Rust.

**"I need a global variable"** - Use `std::sync::OnceLock`, `std::sync::LazyLock`, or `std::sync::atomic` types. All safe.

**"I need to convert between types"** - Use `From`/`Into` traits, `.as_bytes()`, `bytemuck` crate, or `zerocopy` crate. Almost never `std::mem::transmute`.

**"I need a linked list"** - Use `VecDeque`, `Vec`, or `BTreeMap`. Seriously. The number of programs that genuinely need a linked list in 2026 is very small. And if you do, use `std::collections::LinkedList` or the [`slab`](https://crates.io/crates/slab) crate's arena-based approach.

**"I need better performance"** - Profile first. The borrow checker doesn't make your code slow. Unnecessary allocations, poor cache locality, and algorithmic complexity make your code slow. Reaching for `unsafe` before profiling is cargo-culting C habits.

**"The borrow checker won't let me"** - This is often a design signal, not a compiler limitation. If you're fighting the borrow checker, restructure your data ownership. Split the struct, use indices instead of references, clone where the cost is trivial.

According to the [Rust Foundation's security analysis](https://developers.slashdot.org/story/24/05/25/2250236/rust-foundation-reports-20-of-rust-crates-use-unsafe-keyword), about 19% of crates on crates.io use the `unsafe` keyword. But a large portion of those are FFI bindings (the `-sys` crates). In application code - web servers, CLI tools, business logic - the number is far lower.

## Safe abstractions over unsafe: the pattern

This is the actual skill of writing `unsafe` Rust. You don't sprinkle `unsafe` throughout your codebase. You write a small, carefully audited `unsafe` core and wrap it in a safe API that makes misuse impossible.

The textbook example from the [Rust Book](https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html) is `split_at_mut`. Safe Rust won't let you borrow two mutable slices from the same source, even if they don't overlap:

```rust
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();
    let ptr = values.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut data = vec![1, 2, 3, 4, 5, 6];
    let (left, right) = split_at_mut(&mut data, 3);

    left[0] = 10;
    right[0] = 40;

    assert_eq!(left, &[10, 2, 3]);
    assert_eq!(right, &[40, 5, 6]);
}
```

The function itself is not `unsafe`. Callers don't need to worry about any invariants. But internally, it uses raw pointer arithmetic to create two non-overlapping mutable slices. The `assert!(mid <= len)` is the boundary enforcement - it makes it impossible for safe code to violate the preconditions.

This pattern has three steps:

1. **Identify the invariant** - "the two slices must not overlap"
2. **Enforce it at the API boundary** - the `assert!` makes out-of-bounds splits panic before reaching the `unsafe` block
3. **Document it** - a `// SAFETY:` comment on the `unsafe` block explaining why the invariants hold

Let's look at a more realistic example. Say you're building a bump allocator - a fast allocation strategy where you just increment a pointer:

```rust
use std::alloc::Layout;

pub struct BumpAllocator {
    storage: Vec<u8>,
    offset: usize,
}

impl BumpAllocator {
    pub fn new(capacity: usize) -> Self {
        BumpAllocator {
            storage: vec![0u8; capacity],
            offset: 0,
        }
    }

    /// Allocates space for a `T` and returns a mutable reference to it.
    /// Returns `None` if there isn't enough space.
    pub fn alloc<T>(&mut self, value: T) -> Option<&mut T> {
        let layout = Layout::new::<T>();
        let aligned_offset = align_up(self.offset, layout.align());
        let new_offset = aligned_offset + layout.size();

        if new_offset > self.storage.len() {
            return None; // Out of space
        }

        // SAFETY:
        // - aligned_offset is within bounds (checked above)
        // - the pointer is properly aligned (align_up ensures this)
        // - we have exclusive access (&mut self)
        // - no other references to this memory exist (offset only moves forward)
        unsafe {
            let ptr = self.storage.as_mut_ptr().add(aligned_offset) as *mut T;
            std::ptr::write(ptr, value);
            self.offset = new_offset;
            Some(&mut *ptr)
        }
    }

    pub fn bytes_remaining(&self) -> usize {
        self.storage.len() - self.offset
    }
}

fn align_up(offset: usize, align: usize) -> usize {
    (offset + align - 1) & !(align - 1)
}

fn main() {
    let mut bump = BumpAllocator::new(1024);

    let x = bump.alloc(42u32).unwrap();
    println!("x = {x}");

    let y = bump.alloc(3.14f64).unwrap();
    println!("y = {y}");

    let s = bump.alloc([1u8, 2, 3, 4]).unwrap();
    println!("s = {s:?}");

    println!("bytes remaining: {}", bump.bytes_remaining());
}
```

The public API is entirely safe. You can't misalign a pointer, write out of bounds, or create aliased mutable references through the public interface. The `unsafe` is internal, small, and documented. That's the pattern.

## Verifying unsafe code with Miri

Writing `unsafe` code correctly is hard. [Miri](https://github.com/rust-lang/miri) is an interpreter for Rust's Mid-level Intermediate Representation (MIR) that detects undefined behavior at runtime. It was recently featured in a [POPL 2026 paper](https://dl.acm.org/doi/10.1145/3776690) - one of the most prestigious programming languages conferences.

Miri catches:

- Out-of-bounds memory access
- Use-after-free
- Invalid use of uninitialized data
- Violations of aliasing rules (Stacked Borrows / Tree Borrows)
- Data races
- Memory leaks (optional)

Install and run it:

```bash
rustup +nightly component add miri
cargo +nightly miri test
cargo +nightly miri run
```

Here's Miri catching a real bug - use-after-free through a dangling pointer:

```rust
fn main() {
    let ptr: *const i32;
    {
        let val = 42;
        ptr = &val as *const i32;
    } // val is dropped here

    // Miri catches this:
    // error: Undefined Behavior: constructing invalid value:
    // encountered a dangling reference
    unsafe {
        println!("{}", *ptr);
    }
}
```

The compiler might not catch this. The program might even appear to work (the stack memory might still contain 42). But it's undefined behavior, and Miri will find it.

In their evaluation on over 100,000 crates, Miri successfully executed more than 70% of test suites and has found dozens of real bugs in production Rust code, including in the standard library itself.

## Auditing unsafe with cargo-geiger

[`cargo-geiger`](https://github.com/geiger-rs/cargo-geiger) scans your dependency tree and reports `unsafe` usage like a Geiger counter measuring radiation:

```bash
cargo install cargo-geiger
cargo geiger
```

It produces output showing which dependencies use `unsafe` and which declare `#![forbid(unsafe_code)]`. For application code, `#![forbid(unsafe_code)]` at the crate root is a strong statement:

```rust
// src/main.rs or src/lib.rs
#![forbid(unsafe_code)]
```

This makes any `unsafe` block a compile error in your crate. The [axum](https://github.com/tokio-rs/axum) web framework does this - the entire framework is `#![forbid(unsafe_code)]`, delegating all unsafe work to lower-level crates like tokio and hyper that specialize in getting it right.

That's a good architecture: application code is `forbid(unsafe_code)`, library code that needs `unsafe` keeps it minimal and well-tested with Miri.

## The rules to internalize

If you take away one thing from this post, make it this:

`unsafe` is not a quality judgment. It's a responsibility marker. It says "here, the compiler needs human help." Your job is to make that help as small, as documented, and as auditable as possible - then wrap it so nobody else has to think about it.

Concretely:

- **Keep `unsafe` blocks small**. Five lines, not fifty. The less code inside `unsafe`, the less code you need to audit.
- **Document every `unsafe` block** with a `// SAFETY:` comment explaining why the invariants hold.
- **Document every `unsafe fn`** with a `# Safety` section listing what callers must guarantee.
- **Test with Miri**. Run `cargo +nightly miri test` on any code that contains `unsafe`. Make it part of CI.
- **Use `#![forbid(unsafe_code)]`** in application crates. Push `unsafe` down into specialized, well-tested libraries.
- **Profile before reaching for `unsafe` for performance**. The borrow checker isn't your bottleneck.
- **Prefer existing safe abstractions**. Before writing your own `unsafe`, check if a well-maintained crate already does what you need. `crossbeam` for lock-free data structures, `rayon` for parallel iterators, `bytes` for zero-copy buffers - these have been hammered on by thousands of users and tested with Miri.

The ecosystem gives you the tools to write safe code that's also fast. Use them.
