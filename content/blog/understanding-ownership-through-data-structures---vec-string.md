+++
title = "Understanding ownership through data structures - Vec, String, Box"
date = 2025-05-01
description = "Ownership in Rust explained by looking at what Vec, String, and Box actually do with memory - allocations, moves, drops, and what the compiler generates."

[taxonomies]
tags = ["rust", "ownership", "memory", "compiler-internals"]
+++

Most ownership explanations start with the rules. One owner. Value dropped when owner goes out of scope. Move semantics by default. You nod, you write some code, the compiler yells at you, you add `.clone()` until it stops. You know the rules but you don't feel them.

The gap isn't knowledge - it's intuition. And the fastest way to build intuition for ownership is to look at what the standard library data structures actually do with memory. `Vec`, `String`, and `Box` aren't just types you use. They're the clearest demonstrations of how Rust's ownership model maps to real allocator calls, pointer manipulation, and compiler-inserted cleanup code.

If you've written C or C++, you already know this stuff. You've called `malloc` and `free`. You've tracked who owns a heap buffer. You've debugged double-frees and use-after-frees. Rust's ownership system is the compile-time enforcement of rules you were already following manually - or trying to.

<!-- more -->

## Ownership is a question about free()

Strip away the language and ownership comes down to one question: who calls `free()`?

In C, the answer is "whoever remembers to." In C++, it's "whoever holds the `unique_ptr`, or the last `shared_ptr`." In Rust, the compiler answers this question at compile time, for every value, with no runtime cost.

Here's the mental model. Every value in Rust has exactly one owner - a variable binding, a struct field, or a collection element. When that owner goes out of scope, the compiler inserts a call to drop the value, which (for heap-allocating types) means calling the allocator to free the memory. No garbage collector, no reference counting by default, no runtime bookkeeping.

The insight is that this isn't abstract. It's mechanical. And you can see the mechanics by looking at how `Vec<T>` works.

## Vec<T> - the ownership textbook

`Vec<T>` is the single best type for understanding ownership because it does everything: heap allocation, reallocation, element ownership, and cleanup. Let's look at what it actually is.

### The stack layout

From the [standard library source](https://github.com/rust-lang/rust/blob/main/library/alloc/src/vec/mod.rs):

```rust
pub struct Vec<T, A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}
```

`RawVec` holds a pointer and a capacity. So on a 64-bit system, `Vec<T>` is three machine words - 24 bytes - sitting on the stack:

```
Stack (24 bytes)                    Heap
+----------+----------+----------+
| pointer  |  length  | capacity |  ---> [ elem0 | elem1 | elem2 | ... | unused ]
| 0x7fa0.. |    3     |    4     |
+----------+----------+----------+
   8 bytes    8 bytes    8 bytes
```

The pointer points to a contiguous heap allocation. Length is how many elements are initialized. Capacity is how many elements the allocation can hold before it needs to grow. This is identical to what you'd build in C:

```c
typedef struct {
    int32_t* ptr;
    size_t len;
    size_t cap;
} Vec_i32;
```

### What push() does

When you call `vec.push(value)`, two things can happen. If `len < capacity`, Rust writes the value into the next slot and increments `len`. Fast path - no allocator involved.

If `len == capacity`, the buffer is full. Rust needs to grow it. Here's the actual growth logic from [`RawVec::grow_amortized`](https://github.com/rust-lang/rust/blob/main/library/alloc/src/raw_vec.rs):

```rust
// Simplified from the actual source
fn grow_amortized(&mut self, len: usize, additional: usize) {
    let required_cap = len + additional;
    let cap = cmp::max(self.capacity * 2, required_cap);
    let cap = cmp::max(Self::MIN_NON_ZERO_CAP, cap);
    // ... calls realloc or alloc + memcpy ...
}
```

The growth factor is 2x. Double the current capacity, or use the required capacity, whichever is larger. The minimum non-zero capacity depends on element size:

| Element size | First allocation capacity |
|---|---|
| 1 byte (`u8`, `bool`) | 8 |
| 2-1024 bytes (`i32`, most structs) | 4 |
| > 1024 bytes | 1 |

So for `Vec<i32>`, the capacity progression is: 0 -> 4 -> 8 -> 16 -> 32 -> 64 -> ...

There was a [proposal](https://github.com/rust-lang/rust/issues/111307) to switch to 1.5x growth (like Facebook's folly `FBVector`) to reduce memory waste. It was closed - the change caused performance regressions in the compiler itself. Doubling remains.

Under the hood, the growth calls [`realloc`](https://doc.rust-lang.org/std/alloc/fn.realloc.html), which maps to the system allocator's `realloc`. If the allocator can extend the existing block in place, it does. If not, it allocates a new block, copies everything over with `memcpy`, and frees the old block. You can see why `push()` is amortized O(1) - most pushes are a simple write, and the expensive reallocation happens less and less frequently as the buffer grows.

### What drop does

When a `Vec<T>` goes out of scope, the compiler inserts a call to its `Drop` implementation. Here's what happens, step by step:

1. Drop each element: iterate from index 0 to `len`, calling `drop()` on each `T` (if `T` implements `Drop`)
2. Deallocate the buffer: call `dealloc(ptr, layout)` to free the heap memory
3. The 24-byte stack value simply ceases to exist when the stack frame unwinds

In C terms, it's:

```c
// What Rust generates (conceptually)
void vec_drop(Vec_i32* v) {
    // Step 1: drop elements (no-op for i32, but for String it would free each one)
    for (size_t i = 0; i < v->len; i++) {
        element_drop(&v->ptr[i]);
    }
    // Step 2: free the buffer
    free(v->ptr);
    // Step 3: stack memory reclaimed automatically
}
```

This is why `Vec<String>` doesn't leak memory. When the `Vec` drops, it drops each `String`, which drops each `String`'s internal `Vec<u8>`, which frees each string's heap buffer. Ownership is recursive.

### What a move actually is

This is the part that surprises C++ developers. In Rust, moving a `Vec` is a `memcpy` of 24 bytes. That's it.

```rust
let v1 = vec![1, 2, 3];
let v2 = v1; // move: copies 24 bytes on the stack
// v1 is now invalid - compiler enforces this
```

After the move, `v2` has the same pointer, length, and capacity that `v1` had. The heap data didn't move at all. The compiler simply marks `v1` as uninitialized and refuses to let you use it.

```
Before move:
  v1: [ptr=0x7fa0 | len=3 | cap=4]  --->  heap: [1, 2, 3, _]

After move:
  v1: [invalidated by compiler]
  v2: [ptr=0x7fa0 | len=3 | cap=4]  --->  heap: [1, 2, 3, _]
```

Compare this to C++ move semantics, where `std::move` invokes the move constructor, which typically copies the pointer and then nulls out the source. The C++ version does the same pointer shuffle at runtime, but it relies on the move constructor being correctly implemented. Rust does it by flat copy and compiler enforcement. No move constructors, no risk of forgetting to null the source, no use-after-move bugs.

The C++ equivalent would be:

```cpp
// C++ - what std::vector's move constructor does
vector(vector&& other) noexcept
    : ptr_(other.ptr_), len_(other.len_), cap_(other.cap_) {
    other.ptr_ = nullptr;  // Rust doesn't need this
    other.len_ = 0;        // because the compiler prevents access
    other.cap_ = 0;
}
```

Rust doesn't null the source because the compiler statically prevents all access to `v1` after the move. There's nothing to null - the compiler simply won't generate code that reads from `v1`.

### What clone does

If you actually need two independent copies, you call `.clone()`:

```rust
let v1 = vec![1, 2, 3];
let v2 = v1.clone(); // allocates new heap buffer, copies all elements
```

This allocates a new heap buffer with the same capacity, copies all elements over, and returns a new `Vec` pointing to the new buffer. Two independent heap allocations, two independent owners. Modify one, the other doesn't change.

```
After clone:
  v1: [ptr=0x7fa0 | len=3 | cap=4]  --->  heap A: [1, 2, 3, _]
  v2: [ptr=0x8bc0 | len=3 | cap=4]  --->  heap B: [1, 2, 3, _]
```

For `Vec<i32>`, cloning is O(n) where n is the number of elements - one `memcpy`. For `Vec<String>`, it's worse: each `String` gets cloned individually, meaning n separate heap allocations. This is why experienced Rust developers avoid `.clone()` on large collections when a reference will do.

## String - Vec<u8> with a safety invariant

I covered [why Rust has so many string types](/blog/why-rust-has-so-many-string-types/) in a previous post, including the memory layout of `String` and `&str`. The ownership angle is simpler: `String` is literally a `Vec<u8>`.

From the [source](https://github.com/rust-lang/rust/blob/master/library/alloc/src/string.rs):

```rust
pub struct String {
    vec: Vec<u8>,
}
```

Same 24-byte stack layout. Same growth strategy on `push_str`. Same `memcpy` on move. Same deallocation on drop. The only difference is that every method on `String` enforces that the bytes are valid UTF-8.

This means everything you just learned about `Vec` ownership applies directly:

```rust
let s1 = String::from("hello");
let s2 = s1; // moves 24 bytes, heap stays put
// s1 is dead

let s3 = s2.clone(); // new heap allocation, copies 5 bytes
// s2 and s3 are independent
```

The question "why can't I use `s1` after assigning it to `s2`?" has a concrete answer: because `s1`'s pointer still points to the same heap buffer, and if both `s1` and `s2` tried to free it, you'd get a double-free. Rust prevents this at compile time by invalidating `s1` after the move.

Where `String` gets interesting from an ownership perspective is `&str`. A `&str` is a borrowed view into a `String`'s buffer - 16 bytes on the stack (pointer + length, no capacity). It doesn't own the data, so it doesn't free it on drop. The compiler uses lifetimes to ensure the `&str` doesn't outlive the `String` it borrows from. If you want the full lifetime story, I covered it in [Lifetimes in Rust - the mental model that finally clicked](/blog/lifetimes-in-rust-the-mental-model-that-finally-clicked/).

## Box<T> - ownership of exactly one heap value

While `Vec` manages a dynamically-sized array on the heap, `Box<T>` puts exactly one value there. From the [source](https://github.com/rust-lang/rust/blob/master/library/alloc/src/boxed.rs):

```rust
pub struct Box<T: ?Sized, A: Allocator = Global>(Unique<T>, A);
```

`Unique<T>` wraps a `NonNull<T>` pointer. The `Global` allocator is a zero-sized type (ZST), so `Box<T>` is 8 bytes on the stack - just a pointer.

```
Stack (8 bytes)           Heap (size_of::<T>() bytes)
+----------+
| pointer  |  --------->  [ value of type T ]
| 0x7fa0.. |
+----------+
```

### The assembly

For `Box::new(42_i32)`, the compiler generates something close to:

```
; allocate 4 bytes with 4-byte alignment
call __rust_alloc       ; returns pointer in rax
mov dword ptr [rax], 42 ; write 42 to heap
; ... use the box ...
; on drop:
call __rust_dealloc     ; free the 4 bytes
```

You can verify this on [Compiler Explorer](https://rust.godbolt.org/) - write a function that takes and returns a `Box<i32>`, compile with `-C opt-level=3`, and you'll see the `__rust_alloc` / `__rust_dealloc` calls. These map to the system allocator (`malloc`/`free` on most platforms, `jemalloc` if configured).

### Box and move semantics

`Box` follows the same move rules as everything else. Moving a `Box` copies 8 bytes (the pointer) and invalidates the source:

```rust
let b1 = Box::new(vec![1, 2, 3]);
let b2 = b1; // copies 8 bytes (the pointer to the heap Vec)
// b1 is invalid
```

This is Rust's version of C++'s `std::unique_ptr`. One owner, exclusive access, automatic cleanup. The difference: `unique_ptr` has a runtime null state (moved-from pointers become null), and you can accidentally dereference a moved-from `unique_ptr` at runtime. Rust prevents this at compile time.

### Box and recursive types

The classic reason for `Box` is breaking recursive type definitions. I covered this in detail in [Smart pointers in Rust](/blog/smart-pointers-in-rust-box-rc-arc-cow-explained/), but the ownership angle is worth repeating:

```rust
enum Expr {
    Literal(i64),
    Add(Box<Expr>, Box<Expr>),
}
```

Without `Box`, the compiler can't compute the size of `Expr` because it contains itself. `Box` replaces the inline value with a fixed-size pointer (8 bytes). Each `Box<Expr>` owns its heap-allocated `Expr`, and when the parent `Expr` is dropped, it drops both `Box`es, which drop their contents recursively. An entire AST cleaned up by a single drop of the root node.

## What the compiler actually generates

You've seen the user-facing model: one owner, drop at scope end, move invalidates the source. But what does the compiler actually do to enforce this?

### Drop insertion in MIR

Rust compiles through an intermediate representation called [MIR](https://rustc-dev-guide.rust-lang.org/mir/index.html) (Mid-level IR). During MIR construction, the compiler inserts `Drop` terminators at every point where a value goes out of scope:

```
// Simplified MIR for: let v = vec![1, 2, 3];
bb0: {
    _1 = Vec::new();        // create the vec
    Vec::push(_1, 1);
    Vec::push(_1, 2);
    Vec::push(_1, 3);
    // ... use v ...
}
bb1: {
    drop(_1) -> bb2;        // compiler-inserted drop
}
bb2: {
    return;
}
```

### Drop elaboration

The initial drop insertion is coarse - it doesn't account for moves. A later compiler pass called [drop elaboration](https://rustc-dev-guide.rust-lang.org/mir/drop-elaboration.html) refines each drop into one of four categories:

- **Static drop**: The value is definitely initialized. Keep the drop call.
- **Dead drop**: The value was definitely moved away. Remove the drop entirely.
- **Conditional drop**: The value might or might not have been moved (depends on runtime control flow). Insert a drop flag - a boolean that tracks whether the value is still initialized.
- **Open drop**: The value is partially initialized (some fields moved out). Decompose into per-field drops.

Here's what conditional drops look like in practice:

```rust
fn example(condition: bool) {
    let v = vec![1, 2, 3];
    if condition {
        drop(v); // explicit drop, moves v
    }
    // At this point, is v initialized? Depends on `condition`.
    // The compiler inserts a drop flag.
}
```

The compiler generates something like:

```rust
// Pseudo-code of what the compiler emits
fn example(condition: bool) {
    let v = vec![1, 2, 3];
    let mut _v_alive = true;       // drop flag

    if condition {
        drop(v);
        _v_alive = false;
    }

    if _v_alive {                  // conditional drop at scope end
        drop(v);
    }
}
```

The drop flag is a single byte on the stack. In optimized builds, LLVM often eliminates it entirely by proving the control flow makes it redundant.

### Drop glue

For composite types, the compiler generates "drop glue" - a function that calls `Drop::drop()` on the type (if it implements `Drop`), then recursively calls drop glue on each field. For `Vec<String>`:

1. `Vec::drop()` is called - iterates elements, calls drop on each `String`
2. Each `String::drop()` delegates to `Vec::<u8>::drop()` - frees the byte buffer
3. `Vec::drop()` frees the main element buffer

The entire chain is generated at compile time. No vtables, no runtime dispatch (unless you're using trait objects). You can inspect the MIR output yourself with `rustc --emit=mir your_file.rs`.

## When one owner isn't enough - Clone, Rc, Arc

Sometimes you genuinely need multiple parts of your program to access the same data. You have three options, and the choice depends on whether you need independent copies or shared access, and whether you're crossing thread boundaries.

### Clone - independent copies

`Clone` gives you a full deep copy. For `Vec<T>` where `T: Clone`, this means allocating a new buffer and cloning every element:

```rust
let original = vec![String::from("alpha"), String::from("beta")];
let cloned = original.clone();
// Two independent Vecs, each with their own Strings, each with their own heap buffers
// Modifying `cloned` doesn't affect `original`
```

Cost: O(n) where n is the total size of all owned data. For `Vec<String>`, that's one allocation for the new Vec buffer plus one allocation per String element. For a Vec with 10,000 strings, that's 10,001 allocations.

### Rc - shared ownership, single thread

When cloning is too expensive and you need multiple owners of the same data, [`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html) (Reference Counted) shares a single heap allocation:

```rust
use std::rc::Rc;

let data = Rc::new(vec![1, 2, 3, 4, 5]);
let data2 = Rc::clone(&data); // increments counter, does NOT copy the Vec
let data3 = Rc::clone(&data); // counter is now 3
// All three point to the same heap Vec
```

The memory layout:

```
Stack:
  data:  [ptr] ---\
  data2: [ptr] ----+--->  Heap: [ strong: 3 | weak: 0 | Vec{ptr, len, cap} ]
  data3: [ptr] ---/                                          |
                                                             v
                                                      [ 1, 2, 3, 4, 5 ]
```

`Rc::clone()` is O(1) - it just increments a `usize` counter using `Cell` (no atomics). When an `Rc` is dropped, it decrements the counter. When the counter hits zero, the value is dropped and the allocation is freed.

The source tells the story. From [`library/alloc/src/rc.rs`](https://github.com/rust-lang/rust/blob/master/library/alloc/src/rc.rs):

```rust
struct RcInner<T: ?Sized> {
    strong: Cell<usize>,  // non-atomic counter
    weak: Cell<usize>,
    value: T,
}
```

`Cell<usize>` means no atomic operations - just a plain integer increment. This is why `Rc` is `!Send` and `!Sync`. You cannot send it across threads. If two threads incremented the counter simultaneously without synchronization, you'd get a data race. Rust prevents this at compile time.

### Arc - shared ownership across threads

[`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) (Atomically Reference Counted) is the thread-safe version of `Rc`:

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(vec![1, 2, 3]);
let data_clone = Arc::clone(&data);

let handle = thread::spawn(move || {
    println!("from thread: {:?}", data_clone);
    // data_clone dropped here, counter decremented atomically
});

println!("from main: {:?}", data);
handle.join().unwrap();
// data dropped here, counter hits 0, Vec is freed
```

Same layout as `Rc`, but the counters use atomic operations:

```rust
// From library/alloc/src/sync.rs
struct ArcInner<T: ?Sized> {
    strong: atomic::AtomicUsize,  // atomic counter
    weak: atomic::AtomicUsize,
    data: T,
}
```

The clone uses `Relaxed` ordering for the increment (no data synchronization needed - the value was already accessible) and `Release` ordering for the decrement, with an `Acquire` fence before deallocation. This ensures all writes from other threads are visible before the final drop runs. If you want the deep dive on these ordering guarantees, the [Rustonomicon's Arc chapter](https://doc.rust-lang.org/nomicon/arc-mutex/arc-clone.html) walks through every line.

### When to pick which

The decision tree is simple:

**Do you need independent mutation?** Use `Clone`. Each owner gets their own copy and can modify it freely.

**Do you need shared read-only access on one thread?** Use `Rc`. O(1) cloning, no copying, minimal overhead.

**Do you need shared read-only access across threads?** Use `Arc`. Same as `Rc` but with atomic counter operations.

**Do you need shared mutable access?** Combine `Arc` with `Mutex` or `RwLock`: `Arc<Mutex<T>>`. The `Arc` handles ownership across threads, the `Mutex` handles synchronized mutation.

Performance-wise, for data over ~100 bytes, `Rc::clone()` / `Arc::clone()` is dramatically faster than `Clone` because it doesn't copy the data. For trivial types like `i32` or `[u8; 4]`, just clone or copy - the overhead of reference counting (heap allocation, pointer indirection, counter operations) isn't worth it for 4 bytes of data.

I covered the usage patterns and decision tree for these types in more detail in [Smart pointers in Rust](/blog/smart-pointers-in-rust-box-rc-arc-cow-explained/).

## The C++ translation table

If you're coming from C++, here's how the concepts map:

| Rust | C++ | Key difference |
|------|-----|---------------|
| `let x = val;` (move) | `auto x = std::move(val);` | Rust: compile-time enforcement, no null state. C++: runtime convention, moved-from state is valid but unspecified |
| `let x = val.clone();` | `auto x = val;` (copy ctor) | Rust: explicit. C++: implicit (can be surprising) |
| `Box<T>` | `std::unique_ptr<T>` | Rust: no null state, 8 bytes. C++: nullable, 8 bytes |
| `Rc<T>` | `std::shared_ptr<T>` (single-threaded) | Rust: compile-time thread safety. C++: no thread restriction on shared_ptr itself |
| `Arc<T>` | `std::shared_ptr<T>` | Both use atomic refcounting. Rust separates the thread-safe version |
| `&T` | `const T&` | Rust: borrow checker proves validity. C++: dangling references are UB |
| `&mut T` | `T&` | Rust: exclusive access guaranteed. C++: aliasing possible |
| `Drop` | Destructor | Rust: compiler inserts calls, no manual delete. C++: RAII with same guarantee, but use-after-free is possible |

The biggest conceptual shift: in C++, move semantics are a runtime optimization with a valid moved-from state. In Rust, moves are a compile-time ownership transfer. The moved-from variable doesn't enter some "empty but valid" state - it ceases to exist as far as the type system is concerned.

## The payoff

The reason all of this matters isn't theoretical. It's practical.

In C, a function that returns a `Vec`-equivalent (a struct with a pointer, length, and capacity) transfers ownership to the caller. If the caller forgets to free it - memory leak. If the caller frees it twice - undefined behavior. If the caller frees it and then reads from it - undefined behavior. Every one of these bugs has caused real security vulnerabilities.

In C++, RAII and smart pointers solve most of these cases, but the language still allows you to take a reference to a `unique_ptr`'s contents, move the `unique_ptr`, and dereference the now-dangling reference. Undefined behavior, no compiler warning.

In Rust, the ownership rules make all of these impossible. Not "unlikely" or "caught by sanitizers in testing." Impossible. The compiler sees every move, every borrow, every drop, and it refuses to compile code where any of them conflict.

And the cost? Zero. The moves are `memcpy`. The drops are function calls the compiler inserts at exactly the right places. The borrow checks happen at compile time and leave no trace in the binary. The resulting machine code is the same as what a careful C programmer would write by hand - except it's correct by construction.

That's the real point of ownership. It's not a feature you learn to work around. It's the compiler writing your `free()` calls, correctly, every time.
