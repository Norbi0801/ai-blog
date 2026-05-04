+++
title = "Lifetimes in Rust - the mental model that finally clicked"
date = 2025-11-19
description = "Lifetimes aren't about how long things live - they're constraints the compiler checks at compile time, and they cost nothing at runtime."

[taxonomies]
tags = ["rust", "lifetimes", "compiler-internals", "type-system"]
+++

The 2025 State of Rust Survey showed that developers typically need 2-4 weeks to get comfortable with ownership and borrowing, but *months* to feel confident with lifetimes. I was in that camp. I'd read the chapter in The Book, nod along, then hit a lifetime error in real code and stare at it like it was written in Klingon.

The problem wasn't that lifetimes are inherently hard. The problem was that every explanation I found started with the wrong mental model.

<!-- more -->

## The wrong mental model

The most common way people explain lifetimes goes something like: "Lifetimes tell the compiler how long a value lives." This sounds reasonable. You see `'a`, you think "ah, this is the lifetime of the value - the span of time it exists." Then you try to *control* it - stretch `'a` longer, shrink it, manipulate scope boundaries. And none of it works, because that's not what lifetimes are.

You don't tell the compiler how long something lives. The compiler already knows that. It can see every `let` binding, every block boundary, every function call. It knows exactly when each value is created and exactly when it's dropped.

What lifetimes actually do is let you describe *relationships* between references. Not durations. Constraints.

## Borrowed books

Here's the analogy that made everything click for me.

You walk into a library and borrow a book. The librarian stamps your card with a return date. You don't get to decide when the library closes or when the book was printed. Those are facts. What the stamp says is: "you must return this book before the library closes for renovation."

In Rust:

- The **book** is the owned value (a `String`, a `Vec<T>`, whatever owns the data).
- **Borrowing** the book is taking a reference (`&book` or `&mut book`).
- The **library card stamp** is the lifetime annotation (`'a`). It doesn't control when the book was created or when the library closes. It says: "this borrow is valid for at least this long."
- **Returning the book** is when the reference goes out of scope.
- The **library closing for renovation** is when the owned value is dropped.

The compiler's job is simple: check that every borrowed book is returned before the library closes. If you try to keep reading a book after the library has been demolished, that's a dangling reference, and the compiler says no.

If you've been following along with the [closures post](/blog/closures-in-rust-fn-fnmut-fnonce-demystified/), you already saw how Rust tracks captured variables by reference versus by value. Lifetimes are the mechanism that makes those reference-captures safe.

```rust
fn main() {
    let library_open;                    // declare the reference
    {
        let book = String::from("Dune"); // the book exists
        library_open = &book;            // borrow it
    }                                    // book dropped here - library closed
    // println!("{}", library_open);     // ERROR: book is gone
}
```

The compiler knows `book` lives until the end of the inner block. It knows `library_open` tries to use the reference after that block. The lifetimes don't overlap correctly, so it rejects the code. No annotation needed - the compiler figured this out from the structure of the code.

## What `'a` actually means

When you write `'a` in a function signature, you're not naming a specific scope or a concrete timespan. You're declaring a *constraint variable*. The compiler fills in the concrete scope later, based on how the function is called.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

Read this as: "There exists some scope `'a` such that both `x` and `y` are valid for at least `'a`, and the return value is also valid for at least `'a`."

The compiler picks the smallest scope that satisfies all the constraints. If `x` lives for 10 lines and `y` lives for 5 lines, then `'a` is 5 lines - the intersection of both input lifetimes.

```rust
fn main() {
    let result;
    let string1 = String::from("long string");
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
        println!("longest: {}", result); // works - both strings alive here
    }
    // println!("{}", result); // ERROR: string2 is dead, 'a has ended
}
```

The key insight: `'a` is not the lifetime of `x` or `y` individually. It's the overlap region where both are valid. The return value can't outlive that overlap.

## When the compiler figures it out: elision rules

Most of the time, you don't write lifetime annotations. The compiler infers them through three rules defined in [RFC 141](https://rust-lang.github.io/rfcs/0141-lifetime-elision.html). When Aaron Turon proposed these rules in 2014, analysis of the standard library showed they'd eliminate about 87% of explicit lifetime annotations. That number has held up.

The rules apply to function signatures (not struct definitions):

**Rule 1: Each reference parameter gets its own lifetime.**

```rust
// You write:
fn first_word(s: &str) -> &str { ... }
// Compiler sees:
fn first_word<'a>(s: &'a str) -> &str { ... }

// Two parameters, two lifetimes:
fn foo(x: &i32, y: &str) -> ... { ... }
// Compiler sees:
fn foo<'a, 'b>(x: &'a i32, y: &'b str) -> ... { ... }
```

**Rule 2: If there's exactly one input lifetime, it's assigned to all output lifetimes.**

```rust
// You write:
fn first_word(s: &str) -> &str { ... }
// After Rule 1:
fn first_word<'a>(s: &'a str) -> &str { ... }
// After Rule 2:
fn first_word<'a>(s: &'a str) -> &'a str { ... }
// Done. All outputs resolved.
```

**Rule 3: If one of the parameters is `&self` or `&mut self`, the lifetime of `self` is assigned to all output lifetimes.**

```rust
impl MyStruct {
    // You write:
    fn name(&self) -> &str { &self.name }
    // Compiler sees:
    fn name<'a>(&'a self) -> &'a str { &self.name }
}
```

This is why most methods on structs don't need explicit lifetimes. The compiler assumes you're returning something borrowed from `self`, which is true 90%+ of the time.

**When elision fails:**

```rust
// This won't compile:
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

Rule 1 gives `x` lifetime `'a` and `y` lifetime `'b`. Rule 2 doesn't apply (two input lifetimes, not one). Rule 3 doesn't apply (no `self`). The output lifetime is ambiguous - the compiler doesn't know if the return value should live as long as `x` or as long as `y`. You get:

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----      ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value,
          but the signature does not say whether it is borrowed from `x` or `y`
```

This error message is actually excellent. It tells you exactly what the compiler can't figure out and why. The fix is the annotated version we saw earlier: `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str`.

The elision implementation lives in [`compiler/rustc_resolve/src/late.rs`](https://github.com/rust-lang/rust/blob/master/compiler/rustc_resolve/src/late.rs) in the Rust compiler. Notably, elision happens during name resolution - before type checking even starts. The key functions are `resolve_elided_lifetime` and `create_fresh_lifetime`, which generate synthetic lifetime parameters based on the three rules above. [PR #97720](https://github.com/rust-lang/rust/pull/97720) by cjgillot unified the handling of anonymous lifetimes across regular and async functions in Rust 1.64.

## Lifetime bounds on structs

Function signatures are the most common place you'll see lifetimes, but structs that hold references need them too.

```rust
struct Excerpt<'a> {
    part: &'a str,
}
```

Think back to the library analogy. This struct is like a bookmark placed inside a borrowed book. The bookmark (`Excerpt`) can't outlive the book (`str` data) it's pointing into. The `'a` says: "this struct is valid for at most as long as the data its `part` field references."

```rust
fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence;
    {
        let excerpt = Excerpt {
            part: &novel[..16], // borrows from novel
        };
        first_sentence = excerpt.part; // this is fine - novel still alive
    }
    println!("{}", first_sentence); // works: novel outlives first_sentence
}
```

But try to outlive the source:

```rust
fn main() {
    let excerpt;
    {
        let novel = String::from("Call me Ishmael.");
        excerpt = Excerpt { part: &novel };
    } // novel dropped here
    // println!("{}", excerpt.part); // ERROR: novel is gone
}
```

```
error[E0597]: `novel` does not live long enough
 --> src/main.rs:5:33
  |
4 |         let novel = String::from("Call me Ishmael.");
  |             ----- binding `novel` declared here
5 |         excerpt = Excerpt { part: &novel };
  |                                   ^^^^^^ borrowed value does not live long enough
6 |     }
  |     - `novel` dropped here while still borrowed
7 |     println!("{}", excerpt.part);
  |                    ------------ borrow later used here
```

When your struct has multiple reference fields, they can have different lifetimes:

```rust
struct Pair<'a, 'b> {
    left: &'a str,
    right: &'b str,
}
```

Or the same lifetime if they must point to data that lives equally long:

```rust
struct Window<'a> {
    title: &'a str,
    content: &'a str,
}
```

A common beginner mistake is adding `'a` to a struct that doesn't need it - one that owns all its data:

```rust
// Wrong - no references, no lifetimes needed:
struct User {
    name: String,    // owned
    age: u32,        // owned
}

// Right - has a reference, needs a lifetime:
struct UserView<'a> {
    name: &'a str,   // borrowed
    age: u32,        // owned
}
```

The rule is simple: if your struct contains a reference (`&T` or `&mut T`), it needs a lifetime parameter. If it only contains owned types (`String`, `Vec<T>`, `Box<T>`), it doesn't. If you came from the [TypeScript post](/blog/rust-for-typescript-developers-a-mental-model-translation/), this is the biggest gap - TypeScript objects never need to think about whether they own their strings or just reference someone else's.

## Multiple lifetimes: when and why

Sometimes you genuinely need two different lifetime parameters:

```rust
struct Parser<'input, 'config> {
    source: &'input str,
    delimiter: &'config str,
}
```

Here, the input text and the configuration might come from different scopes. The source might be a short-lived string read from a socket, while the delimiter is a `&'static str` compiled into the binary. Using two lifetimes lets callers provide references with different validity scopes:

```rust
fn parse<'input, 'config>(
    source: &'input str,
    config: &'config Config,
) -> Parser<'input, 'config> {
    Parser {
        source,
        delimiter: &config.delimiter,
    }
}
```

If you used a single `'a` for both, the compiler would constrain both references to the shorter of the two lifetimes, which might be unnecessarily restrictive.

That said, don't reach for multiple lifetimes by default. Start with one. The compiler will tell you if it's not enough.

## `'static` - the lifetime that outlives everything

`'static` is a lifetime that lasts for the entire program. String literals are `&'static str` because they're baked into the binary:

```rust
let s: &'static str = "hello"; // lives in the binary's read-only memory
```

But `'static` doesn't only mean "literal embedded in the binary." It means "this reference is valid for as long as the program runs." An intentionally leaked `Box` also has a `'static` lifetime:

```rust
let leaked: &'static str = Box::leak(Box::new(String::from("hello")));
```

A common misconception (covered well in [pretzelhammer's lifetime misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md)) is that `T: 'static` means "`T` is a `&'static` reference." It doesn't. It means "`T` can be held onto indefinitely - it doesn't contain any non-`'static` references." Owned types like `String`, `Vec<u8>`, and `i32` all satisfy `T: 'static` because they own their data and can live as long as needed.

This is why `thread::spawn` requires `F: 'static` - the closure might outlive the current stack frame, so it can't borrow local variables. It needs to either own its data or reference only `'static` data:

```rust
use std::thread;

fn main() {
    let name = String::from("world");
    // thread::spawn(|| println!("{}", name)); // ERROR: closure borrows `name`
    thread::spawn(move || println!("{}", name)); // OK: closure takes ownership
}
```

## Common errors and how to read them

After seeing enough lifetime errors, patterns emerge. Here are the ones you'll hit most often.

### E0597: value does not live long enough

This is the classic "dangling reference" error. Something is dropped while a reference to it still exists.

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
    }
    println!("{}", r);
}
```

```
error[E0597]: `x` does not live long enough
 --> src/main.rs:5:13
  |
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |     println!("{}", r);
  |                    - borrow later used here
```

**How to read it:** Follow the three lines. (1) Where the borrow happens. (2) Where the value is dropped. (3) Where the borrow is used after the drop. The fix is either moving the usage before the drop, or extending the value's scope.

### E0106: missing lifetime specifier

The elision rules couldn't determine the output lifetime.

```rust
struct Config {
    name: &str,  // missing lifetime
}
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:11
  |
2 |     name: &str,
  |           ^ expected named lifetime parameter
```

**Fix:** `struct Config<'a> { name: &'a str }` - or, more often, ask yourself if you really want a reference here. `name: String` is simpler if `Config` owns its data.

### E0621: explicit lifetime required

You annotated one parameter but the body uses another:

```rust
fn pick<'a>(x: &'a i32, y: &i32) -> &'a i32 {
    if *x > *y { x } else { y }
}
```

```
error[E0621]: explicit lifetime required in the type of `y`
```

**Fix:** Either `y: &'a i32` (both must share the lifetime) or restructure so you never return `y`.

### E0623: lifetime mismatch

Two parameters have different inferred lifetimes but data flows between them:

```rust
fn swap_slices(a: &mut [u8], b: &mut [u8]) {
    core::mem::swap(&mut a, &mut b);
}
```

The compiler infers `'a` for `a` and `'b` for `b`, but `swap` requires both to have the same lifetime. **Fix:** Give both the same lifetime: `fn swap_slices<'a>(a: &'a mut [u8], b: &'a mut [u8])`.

## Under the hood: zero runtime cost

Here's a fact that surprises many people: lifetimes don't exist at runtime. They're completely erased before code generation.

If you take two functions:

```rust
fn with_lifetime<'a>(x: &'a str) -> &'a str { x }
fn without_lifetime(x: &str) -> &str { x }
```

And compile them with `--release`, they produce identical assembly. You can verify this on [Compiler Explorer](https://rust.godbolt.org/). Both compile down to a simple register move - the lifetime annotation generates zero additional instructions.

This is by design. Lifetimes are a purely compile-time construct. As [discussed in the Rust forum](https://users.rust-lang.org/t/lifetime-parameters-and-code-generation/101680), they're erased before the monomorphization phase. This is why alternative Rust frontends like mrustc and gccrs can compile Rust code without implementing full lifetime checking - the backend never sees lifetimes.

Contrast this with garbage-collected languages where reference tracking has real runtime cost - reference counting bumps, GC pauses, weak reference tables. Rust's lifetime system gives you the same safety guarantees with zero overhead. The checking happens once, at compile time, and then it's gone.

## Rust 2024 edition: what changed

The Rust 2024 edition (stabilized in [Rust 1.85.0](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/), February 2025) changed how lifetimes interact with return-position `impl Trait`. This is defined in [RFC 3498](https://rust-lang.github.io/rfcs/3498-lifetime-capture-rules-2024.html).

In Rust 2021:

```rust
fn foo(s: &str) -> impl Sized {
    // 's lifetime is NOT captured - the return type
    // doesn't depend on 's lifetime
    42u32
}
```

In Rust 2024, all in-scope generic parameters - including lifetimes - are implicitly captured:

```rust
fn foo(s: &str) -> impl Sized {
    // 's lifetime IS captured now
    42u32
}
```

If you need to opt out, use the `use<>` syntax ([RFC 3617](https://rust-lang.github.io/rfcs/3617-precise-capturing.html), stabilized in Rust 1.82):

```rust
fn foo<'a>(s: &'a str) -> impl Sized + use<> {
    // explicitly captures nothing
    42u32
}
```

The other big change: temporaries in tail expressions are now dropped before local variables. This fixes subtle bugs with `RefCell` borrows and `RwLock` guards but can surprise you if you relied on the old behavior.

## When to avoid lifetimes entirely

Not every reference problem needs lifetime annotations. Sometimes the better answer is to not use references at all.

**Clone it.** If the data is small and you're fighting the borrow checker, just `.clone()`. A 50-byte `String::clone()` takes nanoseconds. Don't optimize for zero-copy when correctness is the bottleneck.

**Use `Rc<T>` or `Arc<T>`.** Shared ownership without lifetimes. As [corrode.dev's "Don't Worry About Lifetimes" post](https://corrode.dev/blog/lifetimes/) puts it: "lifetimes spread in your codebase like a virus. Once you add a lifetime annotation, you have to add it to all the functions that call it." Sometimes `Arc<str>` is the pragmatic choice, especially in concurrent code. If you read the [flyweight pattern post](/blog/the-flyweight-pattern-in-rust-when-strings-are-eating-your-memory/), you saw `Arc<str>` used exactly this way for shared string data.

**Own the data.** `struct Config { name: String }` is simpler than `struct Config<'a> { name: &'a str }`. The owned version doesn't propagate lifetime requirements to every struct and function that touches it.

The right time for explicit lifetimes is when performance actually matters (you've profiled it), when you're writing library APIs that need to be zero-copy, or when the compiler tells you it can't figure things out on its own.

## What's coming: Polonius

The current borrow checker (NLL - Non-Lexical Lifetimes, stabilized in Rust 2018) is good but conservative. It rejects some programs that are actually safe. The next-generation borrow checker, [Polonius](https://rust-lang.github.io/rust-project-goals/2026/polonius.html), models lifetimes as sets of loans rather than code regions. It accepts strictly more programs.

You can try it today on nightly with `-Zpolonius`. One known case it fixes: conditional borrowing where you return early in one branch.

```rust
fn get_or_insert(map: &mut HashMap<String, String>, key: &str) -> &String {
    // NLL rejects this - it thinks the mutable borrow of `map`
    // conflicts with the returned reference
    if let Some(value) = map.get(key) {
        return value;
    }
    map.insert(key.to_string(), String::new());
    &map[key]
}
```

NLL can't see that the `return` in the `if` branch means the mutable borrow at the bottom is unreachable when the immutable borrow escapes. Polonius can. The [2026 project goals](https://rust-lang.github.io/rust-project-goals/2026/polonius.html) include stabilizing Polonius alpha, so this should land in stable Rust before long.

## The mental model, summarized

Lifetimes are not durations. They're constraints on relationships between references. The compiler already knows when values are created and destroyed. What it needs from you - sometimes - is clarification about which references are connected to which data.

Think of it as filling out a library card, not as controlling the universe. You're just documenting the rules that already exist.

And when the compiler asks for a lifetime annotation, it's not being difficult. It's saying: "I see multiple possible interpretations here. Which one did you mean?" Answer the question, and it'll handle the rest.
