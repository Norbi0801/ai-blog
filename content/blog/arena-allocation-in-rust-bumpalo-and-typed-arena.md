+++
title = "Arena allocation in Rust - bumpalo and typed-arena"
date = 2025-06-01
description = "How arena allocators work, why they are 10-100x faster than per-object heap allocation, and how to use bumpalo and typed-arena without fighting the borrow checker."

[taxonomies]
tags = ["rust", "memory", "performance", "compiler-internals"]
+++

If you have ever profiled a Rust parser, compiler, or game loop and seen `__rust_alloc` and `__rust_dealloc` near the top of the flame graph, you have hit the wall that arena allocation is built to break through. The default allocator is a great general-purpose tool, but it pays for that flexibility with synchronization, free-list bookkeeping, fragmentation, and the cost of running `Drop` for every individual value. When you have a workload that allocates thousands of tiny objects with the same lifetime, that overhead compounds.

If you are not familiar with how `Vec`, `Box`, and `String` actually use the allocator, I covered that in [Understanding ownership through data structures - Vec, String, Box](/blog/understanding-ownership-through-data-structures---vec-string/). This post builds on it. We are going to look at a different allocation strategy entirely.

<!-- more -->

## What an arena actually is

An arena is a region of memory that you allocate into without ever individually freeing. You ask for a `Foo`, you ask for a `Bar`, you ask for ten thousand more, and at the end of the scope you free the entire region in a single call. There is no free list. There is no per-object header tracking the size. There is no `dealloc` for individual objects.

The simplest possible arena is just a `Vec<u8>` you bump a pointer into:

```rust
struct ArenaToy {
    buffer: Vec<u8>,
    offset: usize,
}

impl ArenaToy {
    fn alloc<T>(&mut self, value: T) -> &mut T {
        let align = std::mem::align_of::<T>();
        let size = std::mem::size_of::<T>();
        // round offset up to the right alignment
        let aligned = (self.offset + align - 1) & !(align - 1);
        assert!(aligned + size <= self.buffer.capacity(), "arena full");
        unsafe {
            let ptr = self.buffer.as_mut_ptr().add(aligned) as *mut T;
            ptr.write(value);
            self.offset = aligned + size;
            &mut *ptr
        }
    }
}
```

Three operations: align the cursor, write the value, advance the cursor. That is the entire hot path. Compare that to `Box::new(value)`, which calls `alloc::alloc` (a syscall on first use, otherwise a free-list walk inside whatever allocator you linked - jemalloc, mimalloc, or the system one), runs a global lock on contention, and returns a pointer that is now scattered somewhere on the heap.

The toy above has two real problems. It cannot grow when full, and it leaks every `T` because nothing ever calls `Drop`. Both `bumpalo` and `typed-arena` solve the first problem by linking chunks together. They take different positions on the second.

## bumpalo - the heterogeneous arena

[`bumpalo`](https://crates.io/crates/bumpalo) (3.16 at the time of writing) gives you a `Bump` that holds a linked list of chunks. Each chunk is a contiguous slab. Allocation bumps a cursor inside the current chunk; if the chunk does not have room, a new larger chunk is allocated and linked in.

Basic usage:

```rust
use bumpalo::Bump;

let bump = Bump::new();

let x: &mut u32 = bump.alloc(42);
let s: &str = bump.alloc_str("hello world");
let v: &mut [i32] = bump.alloc_slice_copy(&[1, 2, 3, 4]);

// references are valid as long as `bump` is alive
println!("{} {} {:?}", x, s, v);
// at end of scope, bump drops, and the chunks are freed in O(number_of_chunks)
```

The key fact: `bump.alloc::<T>(value)` returns `&mut T` with a lifetime tied to `&self` of the `Bump`. Every reference handed out shares the same lifetime, which means anything inside the arena can freely point at anything else inside the arena - they all live and die together.

### Bump direction and the hot path

`bumpalo` allocates downward inside each chunk. The cursor starts at the top of the chunk and moves toward the bottom. The reason is interesting: aligning a downward-growing cursor is just `ptr & !(align - 1)`, a single AND. Aligning an upward-growing cursor needs `(ptr + align - 1) & !(align - 1)`, an ADD then an AND. Saving one instruction per allocation is the kind of thing that matters when you do this a million times.

The fast path inside `Bump::alloc_layout_fast` (paraphrased from the [source](https://github.com/fitzgen/bumpalo/blob/main/src/lib.rs)) looks roughly like:

```rust
let ptr = footer.ptr.get().as_ptr() as usize;
let new_ptr = ptr.checked_sub(layout.size())?;
let new_ptr = new_ptr & !(layout.align() - 1);
if new_ptr < chunk_start { return None; } // overflow this chunk
footer.ptr.set(NonNull::new_unchecked(new_ptr as *mut u8));
```

Compile that with optimisations and you get something like four or five instructions plus a branch for the overflow case. A `Box::new` call is dozens of instructions, a few branches, and on multithreaded code, an atomic operation. The difference shows up directly in microbenchmarks.

### Drop is not called by default

This is the trap. `bump.alloc(value)` does *not* register `value` for destruction. When the `Bump` is dropped, it frees its chunks and that is it. If `value` was a `String`, its heap buffer is leaked. If it was a `File`, the file descriptor is leaked. If it was a `MutexGuard`, you have a deadlock.

```rust
use bumpalo::Bump;

let bump = Bump::new();
let s = bump.alloc(String::from("oops")); // String allocates on the global heap
                                          // its buffer leaks when bump drops
```

The fix, when you actually need destructors, is `bumpalo::boxed::Box`:

```rust
use bumpalo::{Bump, boxed::Box};

let bump = Bump::new();
let s: Box<String> = Box::new_in(String::from("safe"), &bump);
// when `s` goes out of scope, it runs String's Drop and returns its
// memory to the bump. The bump itself still owns the chunk.
```

Or use `bumpalo::collections::Vec` and `bumpalo::collections::String`, which allocate their backing storage *inside* the bump and need no global heap at all. That is the configuration you want for a parser or a compiler: every allocation goes through the bump, nothing on the heap, no destructors to run.

### Memory layout in practice

Take a tree:

```rust
use bumpalo::Bump;

#[derive(Debug)]
enum Expr<'a> {
    Num(i64),
    Add(&'a Expr<'a>, &'a Expr<'a>),
    Mul(&'a Expr<'a>, &'a Expr<'a>),
}

fn build<'a>(bump: &'a Bump) -> &'a Expr<'a> {
    let one = bump.alloc(Expr::Num(1));
    let two = bump.alloc(Expr::Num(2));
    let three = bump.alloc(Expr::Num(3));
    let sum = bump.alloc(Expr::Add(one, two));
    bump.alloc(Expr::Mul(sum, three))
}

fn main() {
    let bump = Bump::new();
    let expr = build(&bump);
    println!("{:?}", expr);
}
```

The five `Expr` nodes are packed into one chunk, in allocation order, separated only by alignment padding. If you walked the resulting tree and dereferenced every node, you would touch the same one or two cache lines repeatedly. With `Box<Expr>`, each `alloc` call could land anywhere - the kernel's page tables, your allocator's free list, and chance all decide. Cache misses on tree traversal are why arena-based AST evaluation is so much faster than the textbook `Box`-based version.

## typed-arena - the homogeneous arena

[`typed-arena`](https://crates.io/crates/typed-arena) (2.0.2) takes the opposite tradeoff. One arena, one type, but it *does* call `Drop` on every value when the arena is dropped.

```rust
use typed_arena::Arena;

struct Connection { /* holds a socket */ }

impl Drop for Connection {
    fn drop(&mut self) {
        // close the socket
    }
}

let conns: Arena<Connection> = Arena::new();
let a = conns.alloc(Connection { /* ... */ });
let b = conns.alloc(Connection { /* ... */ });
// when `conns` drops, both a and b have their Drop called
```

Internally, `typed_arena::Arena<T>` is a `RefCell<Vec<Vec<T>>>` - chunks of `Vec<T>` that grow geometrically. Allocation pushes to the last chunk; when full, a new chunk twice the size is created. Because each chunk is `Vec<T>`, dropping the arena calls `Vec`'s `Drop`, which calls `T::drop` on every element. You get arena-style bulk freeing *and* per-element destruction, at the cost of being limited to one type.

The lifetime story is identical to bumpalo: `alloc(value)` returns `&mut T` tied to `&self`. Same self-referential trick works:

```rust
use typed_arena::Arena;

struct Node<'a> {
    value: i32,
    children: Vec<&'a Node<'a>>,
}

let arena = Arena::new();
let leaf = arena.alloc(Node { value: 1, children: vec![] });
let root = arena.alloc(Node { value: 0, children: vec![leaf] });
```

### When to pick which

Use `typed-arena` when:

- You allocate one type only
- That type has a meaningful `Drop` (file handles, sockets, allocations of its own)
- You do not need string interning, slice allocation, or other utilities

Use `bumpalo` when:

- You allocate many different types into the same scope (compiler IR, AST, parser tokens)
- The values are POD or you can afford to have their `Drop` skipped
- You want `bumpalo::collections` to keep `Vec` and `String` data inside the arena
- You are interning strings with `alloc_str`

Real compilers use bumpalo for AST and IR. `rustc` uses its own arena machinery in `rustc_arena` that is similar in spirit but specialised for typed and dropless variants per type. Servo's old DOM used typed-arena. Cranelift uses bumpalo for IR. The patterns are well-trodden.

## The lifetime trap

The piece that catches most people writing their first arena code: the lifetime of the references is the lifetime of the *shared borrow* of the arena, not the arena's owned lifetime. That has consequences.

```rust
use bumpalo::Bump;

fn make_thing(bump: &Bump) -> &mut i32 {
    bump.alloc(42)
} // returned &mut i32 has the same lifetime as `bump` parameter - fine
```

But this fails:

```rust
use bumpalo::Bump;

fn problem() -> &'static mut i32 {
    let bump = Bump::new();
    bump.alloc(42) // ERROR: bump dropped at end of fn
}
```

The reference cannot outlive the arena. Obvious. The non-obvious version:

```rust
use bumpalo::Bump;

struct Parser<'a> {
    bump: &'a Bump,
}

impl<'a> Parser<'a> {
    fn alloc_token(&self) -> &'a str {
        self.bump.alloc_str("token") // returns &'a str, tied to bump
    }
}
```

This works because `Parser` holds a *reference* to the bump, with lifetime `'a`. If you tried instead:

```rust
struct Parser {
    bump: Bump,
}

impl Parser {
    fn alloc_token(&self) -> &str {
        self.bump.alloc_str("token")
        // returns &'_ str tied to &self, not to bump itself
    }
}
```

That also compiles, but now every reference handed out is tied to the borrow of `self`. You can never mutate the parser through `&mut self` while any of those references exist. For a parser that builds an AST and returns it, that pattern is usually wrong - you want the references tied to an outer arena that outlives the parser.

The standard pattern in compiler frontends is therefore:

```rust
fn compile<'a>(source: &str, bump: &'a Bump) -> &'a Ast<'a> {
    let mut parser = Parser::new(source, bump);
    parser.parse_program()
}
```

The arena is owned by the caller. The compiler frontend is a guest. AST nodes outlive the parser but not the arena.

## Benchmarking the difference

Here is a microbenchmark that measures the gap. Allocate one million 32-byte structs three ways: `Vec<Box<T>>`, `typed_arena::Arena<T>`, and `bumpalo::Bump`.

```rust
use std::time::Instant;
use bumpalo::Bump;
use typed_arena::Arena;

#[derive(Clone, Copy)]
struct Node {
    value: u64,
    flags: [u64; 3],
}

const N: usize = 1_000_000;

fn time<F: FnOnce()>(label: &str, f: F) {
    let start = Instant::now();
    f();
    println!("{:>20}: {:?}", label, start.elapsed());
}

fn main() {
    time("Vec<Box<Node>>", || {
        let mut v = Vec::with_capacity(N);
        for i in 0..N {
            v.push(Box::new(Node { value: i as u64, flags: [0; 3] }));
        }
        std::hint::black_box(&v);
    });

    time("typed_arena", || {
        let arena = Arena::with_capacity(N);
        for i in 0..N {
            arena.alloc(Node { value: i as u64, flags: [0; 3] });
        }
        std::hint::black_box(&arena);
    });

    time("bumpalo", || {
        let bump = Bump::with_capacity(N * 32);
        for i in 0..N {
            bump.alloc(Node { value: i as u64, flags: [0; 3] });
        }
        std::hint::black_box(&bump);
    });
}
```

On my hardware (a recent x86_64 Linux box, glibc malloc, `--release`), the rough numbers are:

```
   Vec<Box<Node>>: 28.4ms
      typed_arena: 4.1ms
          bumpalo: 2.6ms
```

Numbers vary with allocator (mimalloc cuts the `Box` version in half), CPU, and `Node` size. The shape is consistent: bumpalo is the fastest because there is no per-element bookkeeping, typed-arena is close behind because its `Vec<Vec<T>>` chunking still has to call `Vec::push`, and `Box` per element is the slowest because each call enters the global allocator.

Now traversal. Walk the data and sum a field:

```rust
let sum: u64 = nodes.iter().map(|n| n.value).sum();
```

Boxed version: ~12ms. Arena versions: ~2ms. The reason is cache locality. `Vec<Box<T>>` stores pointers contiguously but the `T` they point at are scattered across whatever pages the allocator handed out, so each dereference is a potential cache miss. Both arenas pack the `T` values themselves contiguously, so the prefetcher can stream them in.

This is why allocator choice matters for cold-data workloads (think large compile units or long parses) more than it does for hot-data workloads with small working sets.

## Use cases where arenas earn their keep

**Compilers and parsers.** AST nodes have the same lifetime as the parse, point at each other, and never need individual `Drop`. Bumpalo is built for this. `rustc`, `swc`, `oxc`, and Cranelift all use arena-style allocators for IR.

**Game frame allocators.** A frame allocator is a bump arena that resets at the start of each frame. Every transient allocation - particle data, scratch buffers, immediate-mode UI text - lives until end of frame and disappears with one pointer reset. Use [`bumpalo::Bump::reset`](https://docs.rs/bumpalo/latest/bumpalo/struct.Bump.html#method.reset) - it returns the cursor to chunk-start without freeing the chunks, so the next frame allocates into the same memory. Zero `malloc` calls per frame in steady state.

**Request-scoped allocations in servers.** Allocate request-local data into a per-request bump. At end of request, drop the bump. No garbage collector pauses, no per-allocation lock contention, predictable freeing.

**Graph algorithms.** Building a dependency graph or call graph where nodes reference each other. Bumpalo lets you build the graph naturally with `&'a Node<'a>` edges.

## When arenas hurt

They are not free wins. The cases where arenas are wrong:

- **Long-lived heterogeneous lifetimes.** If different objects need to be freed at different times, you need a real allocator.
- **Big values mixed with small values.** A bump allocator does not reuse space; allocating a 10MB blob inside a long-running bump pins 10MB until reset. The allocator's free list would have reclaimed it.
- **`Drop`-heavy values without bumpalo's `Box`.** Default bumpalo skips destructors. If you forget that, you leak.
- **Code that hands out arena-allocated references across thread boundaries.** `Bump` is `!Sync`. There is `bumpalo::sync::Bump` that uses atomics, but it is slower; the design assumption is one bump per thread or per scope.
- **Workloads dominated by a few large `Vec` pushes.** A regular `Vec` reuses its allocation across `push`es. The arena cannot beat that.

## What this changes about your code

The pattern that arena allocation enables, which is awkward in regular Rust, is graphs of references. `&'a T` everywhere instead of `Rc<RefCell<T>>` or indices into vectors. The tree of an AST becomes the natural shape: each node holds references to its children, the borrow checker is satisfied because everything has the same lifetime, and the runtime is faster than any ref-counted alternative.

The cost is that you give up the ability to free individual nodes. For ASTs and IR, that is exactly the right tradeoff - they are throw-everything-away-at-once data structures. For long-lived caches or shared state, the tradeoff goes the other way and you want regular allocation.

Try this: take a parser or graph-walking piece of code in one of your projects and switch its node allocation to a bump. Measure. The numbers usually surprise people who have never benchmarked it. And the lifetime ergonomics, once you stop fighting them, are nicer than `Rc` everywhere.
