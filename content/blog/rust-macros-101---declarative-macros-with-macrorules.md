+++
title = "Rust Macros 101 - Declarative Macros with macro_rules!"
date = 2026-04-03
description = "A deep dive into Rust's macro_rules! system: pattern matching, fragment specifiers, repetitions, hygiene, TT munchers, and practical macros you can steal."

[taxonomies]
tags = ["rust", "macros", "metaprogramming"]
+++

Before you reach for proc macros, `syn`, and `quote` - there's an entire code generation system built into the language that requires zero dependencies and compiles in milliseconds. Declarative macros with `macro_rules!` are Rust's pattern-matching code generator. They're not as flexible as proc macros, but they cover a surprising amount of ground, and understanding them is a prerequisite for understanding the rest of the macro ecosystem.

This post covers how `macro_rules!` actually works - the matching rules, repetitions, hygiene model, and the sharp edges you'll hit. We'll build three practical macros along the way.

<!-- more -->

## What macro_rules! actually does

A `macro_rules!` definition is a set of pattern-matching arms. When you invoke the macro, the compiler tries each arm top-to-bottom until one matches. The matched arm's right-hand side becomes the expanded code.

```rust
macro_rules! greet {
    () => {
        println!("hello, world");
    };
    ($name:expr) => {
        println!("hello, {}", $name);
    };
}

greet!();           // expands to: println!("hello, world");
greet!("Alice");    // expands to: println!("hello, {}", "Alice");
```

Each arm has the form `(matcher) => { transcriber }`. The matcher is a pattern against the token stream. The transcriber is the code that gets pasted in. Variables captured in the matcher (like `$name`) are substituted in the transcriber.

The semicolons between arms are required. The braces around the transcriber can be `{}`, `()`, or `[]` - doesn't matter. Convention is `{}`.

One thing that trips people up: this is not textual substitution like C preprocessor macros. The Rust compiler parses the input into a token stream, matches it against the pattern, and produces a new token stream. The output is then parsed as Rust code. This distinction matters for hygiene (more on that later).

## Fragment specifiers - what your patterns can capture

The `$name:spec` syntax in matchers captures a piece of the input. The fragment specifier (`spec`) tells the compiler what kind of syntax construct to expect. Rust has 14 of them:

| Specifier | Matches | Example |
|-----------|---------|---------|
| `expr` | Any expression | `1 + 2`, `foo()`, `if x { 1 } else { 2 }` |
| `ty` | A type | `i32`, `Vec<String>`, `&'a str` |
| `ident` | An identifier | `foo`, `MyStruct`, `x` |
| `pat` | A pattern | `Some(x)`, `1..=5`, `ref mut y` |
| `path` | A path | `std::collections::HashMap`, `crate::foo` |
| `stmt` | A statement | `let x = 1`, `x.push(2)` |
| `block` | A block expression | `{ let x = 1; x + 2 }` |
| `item` | An item definition | `fn foo() {}`, `struct Bar;`, `impl Baz {}` |
| `literal` | A literal | `42`, `"hello"`, `true`, `3.14` |
| `lifetime` | A lifetime | `'a`, `'static` |
| `meta` | Attribute contents | `derive(Debug)`, `cfg(test)` |
| `tt` | A single token tree | Anything: a token or a delimited group |
| `vis` | A visibility qualifier | `pub`, `pub(crate)`, or empty |
| `pat_param` | A pattern (no top-level `\|`) | Like `pat` but allows `\|` to follow |

The most important ones you'll use day-to-day are `expr`, `ty`, `ident`, `tt`, and `literal`.

### The 2024 edition change

If you're on the [2024 edition](https://doc.rust-lang.org/edition-guide/rust-2024/macro-fragment-specifiers.html), `expr` now matches `const { ... }` blocks and `_` expressions in addition to regular expressions. If you need the old behavior (because another arm starts with `const`), use `expr_2021`. Running `cargo fix --edition` will convert your macros automatically.

### Fragment opacity

This is a subtle but important rule. When you capture something with any specifier other than `ident`, `lifetime`, or `tt`, the captured fragment becomes **opaque**. You can't re-match it with another specifier in a subsequent macro invocation.

```rust
macro_rules! capture {
    ($e:expr) => {
        inspect!($e);
    };
}

macro_rules! inspect {
    (1 + 2) => { println!("got literal addition"); };
    ($e:expr) => { println!("got something else"); };
}

capture!(1 + 2);
// Prints "got something else" - NOT "got literal addition"
// Because $e was captured as an opaque expr, it can't be
// destructured again by inspect!
```

If you need to pass tokens through multiple macro layers and re-match them, use `$($t:tt)*` - token trees preserve the full structure.

## Follow-set restrictions

You can't put arbitrary tokens after a fragment specifier in a matcher. The compiler enforces [follow-set restrictions](https://doc.rust-lang.org/reference/macros-by-example.html) to prevent ambiguity:

- After `expr` or `stmt`: only `=>`, `,`, `;`
- After `pat`: only `=>`, `,`, `=`, `if`, `in`
- After `ty` or `path`: only `=>`, `,`, `=`, `|`, `;`, `:`, `>`, `>>`, `[`, `{`, `as`, `where`
- After `tt`, `ident`, `literal`, `lifetime`, `block`, `meta`: no restrictions

This means you can't write something like:

```rust
macro_rules! bad {
    ($e:expr $t:ty) => { };  // ERROR: ty cannot follow expr
}
```

You need a separator:

```rust
macro_rules! good {
    ($e:expr, $t:ty) => { };  // comma separates them - fine
}
```

These restrictions exist because the parser needs to unambiguously decide where one fragment ends and the next begins. If `expr` could be followed by anything, the compiler couldn't tell when the expression is done.

## Repetitions

Repetitions are what make `macro_rules!` actually useful for code generation. The syntax is `$( ... ) sep rep` where:

- `$( ... )` wraps the repeated pattern
- `sep` is an optional separator token (usually `,` or `;`)
- `rep` is `*` (zero or more), `+` (one or more), or `?` (zero or one)

```rust
macro_rules! make_vec {
    ( $( $elem:expr ),* ) => {
        {
            let mut v = Vec::new();
            $( v.push($elem); )*
            v
        }
    };
}

let v = make_vec![1, 2, 3];
// Expands to:
// {
//     let mut v = Vec::new();
//     v.push(1);
//     v.push(2);
//     v.push(3);
//     v
// }
```

In the transcriber, `$( ... )*` repeats its body once for each captured element. Every metavariable used inside the repetition must have been captured at the same repetition depth.

### Nested repetitions

You can nest them:

```rust
macro_rules! matrix {
    ( $( [ $( $elem:expr ),* ] ),* ) => {
        vec![ $( vec![ $( $elem ),* ] ),* ]
    };
}

let m = matrix![[1, 2], [3, 4], [5, 6]];
// m: Vec<Vec<i32>> = [[1, 2], [3, 4], [5, 6]]
```

### Trailing comma handling

A common pattern for accepting optional trailing commas:

```rust
macro_rules! my_macro {
    ( $( $item:expr ),* $(,)? ) => {
        // ...
    };
}

my_macro![1, 2, 3];   // works
my_macro![1, 2, 3,];  // also works
```

The `$(,)?` matches zero or one trailing comma. Get in the habit of adding this to every macro that takes comma-separated arguments - your users will thank you.

## Practical macro #1: vec_of_strings!

The standard `vec!` macro creates a `Vec<T>`, but you often want `Vec<String>` from string literals. Writing `.to_string()` or `.into()` on every element gets old fast. Let's fix that:

```rust
macro_rules! vec_of_strings {
    ( $( $s:expr ),* $(,)? ) => {
        vec![ $( String::from($s) ),* ]
    };
}

let names = vec_of_strings!["Alice", "Bob", "Charlie"];
// Type: Vec<String>

// Also works with non-literal expressions:
let dynamic = "Dave";
let names = vec_of_strings!["Alice", dynamic, &format!("Mr. {}", "E")];
```

Short, useful, zero-cost at runtime (same code you'd write by hand). This is the sweet spot for declarative macros - eliminating repetitive boilerplate that's too small to justify a function.

## Practical macro #2: assert_matches!

Standard `assert_eq!` checks for equality. But sometimes you want to check that a value matches a pattern. The standard library gained `assert_matches!` in nightly, but you can write your own:

```rust
macro_rules! assert_matches {
    ($value:expr, $pattern:pat $(,)?) => {
        match $value {
            $pattern => {}
            ref actual => panic!(
                "assertion failed: `{:?}` does not match `{}`",
                actual,
                stringify!($pattern)
            ),
        }
    };
    ($value:expr, $pattern:pat, $($msg:tt)+) => {
        match $value {
            $pattern => {}
            ref actual => panic!(
                "assertion failed: `{:?}` does not match `{}`: {}",
                actual,
                stringify!($pattern),
                format_args!($($msg)+)
            ),
        }
    };
}

#[derive(Debug)]
enum Response {
    Ok(u16),
    Error(String),
}

let resp = Response::Ok(200);
assert_matches!(resp, Response::Ok(200));
assert_matches!(resp, Response::Ok(_), "expected Ok variant");
// assert_matches!(resp, Response::Error(_)); // panics with useful message
```

A few things to notice:

- The second arm uses `$($msg:tt)+` to capture an arbitrary format string with arguments. The `tt` specifier is crucial here - it preserves everything as raw tokens, so `format_args!` can parse it later.
- `stringify!($pattern)` turns the pattern back into a string literal at compile time. This gives readable error messages without any runtime cost.
- `ref actual` in the catch-all arm avoids moving the value, so we can print it with `Debug`.

## The TT muncher pattern

When your macro needs to process a complex, recursive grammar, the TT muncher is the go-to pattern. It works by consuming tokens from the front of the input one piece at a time, generating output, and recursing on the tail.

The name comes from capturing the unprocessed input as `$($tail:tt)*` - you're "munching" token trees.

```rust
macro_rules! html {
    // Base case: nothing left to process
    () => { String::new() };

    // Self-closing tag: <br/>, then process remaining siblings
    (<$tag:ident /> $($rest:tt)*) => {{
        let mut out = format!("<{} />", stringify!($tag));
        out.push_str(&html!($($rest)*));
        out
    }};

    // Tag with text content: <p>"hello"</p>, then process remaining siblings
    (<$tag:ident> $content:literal </$close:ident> $($rest:tt)*) => {{
        let mut out = format!("<{}>{}</{}>",
            stringify!($tag), $content, stringify!($close));
        out.push_str(&html!($($rest)*));
        out
    }};
}

fn main() {
    let br = html!(<br />);
    assert_eq!(br, "<br />");

    let para = html!(<p> "hello world" </p>);
    assert_eq!(para, "<p>hello world</p>");

    // The real power: processing sibling elements
    let page = html!(
        <h1> "Title" </h1>
        <p> "First paragraph." </p>
        <br />
        <p> "Second paragraph." </p>
    );
    assert_eq!(
        page,
        "<h1>Title</h1><p>First paragraph.</p><br /><p>Second paragraph.</p>"
    );
}
```

This is a textbook TT muncher. Each arm matches a specific structure at the front of the token stream (a self-closing tag or a tag with content), processes it, then recurses on `$($rest:tt)*` - whatever's left. The base case `() => { String::new() }` terminates the recursion when all tokens are consumed.

Notice we can't handle deeply nested tags like `<div><p>"hi"</p></div>` - the `$($rest:tt)*` at the end is greedy and would swallow the closing `</div>`. This is genuinely where `macro_rules!` hits a wall. A proc macro with a proper token stream iterator handles nesting trivially. For a declarative macro, you'd need to use Rust's own delimiters (braces or brackets) to group children, which defeats the purpose of HTML-like syntax.

### Performance warning

TT munchers are inherently quadratic in compile time. Processing N token trees requires the compiler to attempt matching against N tokens, then N-1, then N-2, and so on. For small inputs this is fine. For large inputs (hundreds of tokens), compile times balloon. If you find yourself building a parser with many TT muncher arms, that's the signal to switch to a proc macro, where you get a proper `TokenStream` iterator and linear-time processing.

## Hygiene

C macros are infamous for variable name collisions. Rust's `macro_rules!` have a partial solution called **mixed-site hygiene**.

The rule is:
- **Local variables** and **labels** are resolved at the **definition site** (where the macro is written)
- **Everything else** (functions, types, modules) is resolved at the **invocation site** (where the macro is called)

Here's what that means in practice:

```rust
let x = 1;

macro_rules! check {
    () => {
        // x resolves at definition site - it sees x = 1
        assert_eq!(x, 1);
    };
}

{
    let x = 99;
    check!();  // passes! x is 1, not 99
}
```

The `x` inside the macro refers to the `x` that was in scope when the macro was defined, not the `x` at the call site. This prevents accidental variable capture.

But hygiene has limits. You **cannot** define a variable in one macro invocation and use it in another:

```rust
macro_rules! m {
    (define) => { let x = 42; };
    (refer) => { dbg!(x); };
}

fn main() {
    m!(define);
    m!(refer);  // ERROR: cannot find value `x` in this scope
}
```

Each macro expansion gets its own syntax context. The `x` from `m!(define)` lives in a different context than the `x` that `m!(refer)` tries to access. They're different "colors" of `x` as far as the compiler is concerned.

This is usually what you want. But when you need a macro to introduce a binding that the caller can use, the trick is to let the caller name it:

```rust
macro_rules! let_val {
    ($name:ident = $val:expr) => {
        let $name = $val;
    };
}

let_val!(answer = 42);
println!("{answer}");  // works - the caller chose the name
```

Since `$name` is captured from the call site, it lives in the call site's syntax context. No collision.

## Debugging with cargo expand

When your macro doesn't expand the way you expect, if you're not familiar with `cargo-expand`, I covered it in [Debugging Rust Beyond println!](/blog/debugging-rust-beyond-println/) - it's your best friend for macro development.

The short version:

```bash
cargo install cargo-expand
cargo expand           # expand everything
cargo expand my_module # expand a specific module
```

Here's a real debugging session. Say you wrote this:

```rust
macro_rules! make_fn {
    ($name:ident, $ret:ty, $body:expr) => {
        fn $name() -> $ret {
            $body
        }
    };
}

make_fn!(get_answer, i32, { 42 });
```

Running `cargo expand` shows:

```rust
fn get_answer() -> i32 {
    {
        42
    }
}
```

Notice the double braces - `$body` captured `{ 42 }` as an expression (which is a block), and the transcriber wrapped it in another `{ }`. It compiles and works, but if this were a more complex case, the extra nesting could cause issues. You'd catch this immediately in the expansion.

Another tool: `trace_macros!` on nightly. It prints each macro invocation as the compiler processes it:

```rust
#![feature(trace_macros)]

trace_macros!(true);
make_fn!(get_answer, i32, { 42 });
trace_macros!(false);
```

This outputs the expansion steps to stderr during compilation. Useful when you have macros calling macros and need to see the chain.

## Scoping and visibility

`macro_rules!` macros have **textual scope** by default - they're visible from the point of definition to the end of their enclosing scope, like a `let` binding:

```rust
// greet!(); // ERROR: not yet defined

macro_rules! greet {
    () => { println!("hi"); };
}

greet!(); // OK

mod inner {
    // greet!(); // OK from parent scope
}
```

To make a macro available across crates, use `#[macro_export]`:

```rust
#[macro_export]
macro_rules! my_macro {
    () => {};
}

// Now accessible as crate::my_macro! or by path
```

### The $crate metavariable

When your exported macro needs to reference items from its own crate, use `$crate`:

```rust
// In crate `my_utils`:
pub fn internal_helper() -> String {
    "helped".into()
}

#[macro_export]
macro_rules! do_thing {
    () => {
        $crate::internal_helper()
    };
}
```

Without `$crate`, the macro would try to find `internal_helper` in the caller's crate and fail. `$crate` always resolves to the crate where the macro is defined, regardless of where it's invoked. Use it in every `#[macro_export]` macro that references items from its own crate.

## When macro_rules! is enough (and when it isn't)

**macro_rules! handles:**
- Reducing repetitive code (boilerplate elimination)
- Simple DSLs with known grammar
- Variadic function-like syntax (`vec!`, `println!`, `format_args!`)
- Conditional compilation helpers
- Test fixtures and assertion helpers

**You need proc macros when you need:**
- **Derive macros** (`#[derive(MyTrait)]`) - inspecting struct fields and generating trait impls
- **Attribute macros** (`#[my_attr]`) - transforming items based on attributes
- **Arbitrary string parsing** - proc macros get a `TokenStream` they can iterate however they want
- **External data** - reading files, env vars, or schemas at compile time
- **Complex transformations** - anything that feels like writing a compiler pass

The practical dividing line: if you can describe your macro's input as a fixed set of patterns with repetitions, `macro_rules!` works. If you need to inspect the structure of a type definition (field names, field types, generics), reach for a proc macro.

There's a middle ground emerging. The Rust team is working on [declarative macro improvements](https://rust-lang.github.io/rust-project-goals/2025h1/macro-improvements.html) that would let `macro_rules!` handle derive and attribute positions, potentially eliminating the need for proc macros in many cases. Not stable yet, but worth watching.

## Metavariable expressions (nightly)

[RFC 3086](https://rust-lang.github.io/rfcs/3086-macro-metavar-expr.html) introduces metavariable expressions - metadata about repetitions. As of early 2026, these are still behind `#![feature(macro_metavar_expr)]` on nightly, but they solve real pain points:

```rust
#![feature(macro_metavar_expr)]

macro_rules! count_args {
    ( $( $x:expr ),* ) => {
        ${count(x)}  // number of repetitions, no recursion needed
    };
}

macro_rules! enumerate {
    ( $( $x:expr ),* ) => {
        [$( (${index()}, $x) ),*]
        // index() gives 0, 1, 2, ... for each repetition
    };
}

assert_eq!(count_args!(a, b, c), 3);
assert_eq!(enumerate!("a", "b", "c"), [(0, "a"), (1, "b"), (2, "c")]);
```

Without these, counting repetitions requires a [recursive workaround](https://lukaswirth.dev/tlborm/decl-macros/building-blocks/counting.html) with nested macro calls. `${count()}` is a one-liner. `${index()}` and `${length()}` give you positional info inside repetitions. `${ignore(x)}` expands to nothing but lets you repeat at the same depth as `x`.

The stabilization effort has [stalled](https://github.com/rust-lang/rust/pull/122808) due to design questions around depth parameters, but the core functionality works and is widely used in nightly projects.

## Putting it all together: patterns worth stealing

Here are a few more patterns that show up in real codebases.

### Enum dispatch

```rust
macro_rules! dispatch {
    ($value:expr, $enum:ident, [ $( $variant:ident ),* ], $method:ident ( $($arg:expr),* )) => {
        match $value {
            $( $enum::$variant(inner) => inner.$method($($arg),*), )*
        }
    };
}

enum Shape {
    Circle(Circle),
    Rect(Rect),
}

// Instead of writing the match manually every time:
let area = dispatch!(shape, Shape, [Circle, Rect], area());
```

### Newtype wrapper

```rust
macro_rules! newtype {
    ($name:ident, $inner:ty) => {
        #[derive(Debug, Clone, PartialEq, Eq, Hash)]
        pub struct $name($inner);

        impl $name {
            pub fn new(val: $inner) -> Self {
                Self(val)
            }

            pub fn into_inner(self) -> $inner {
                self.0
            }
        }

        impl std::fmt::Display for $name {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                write!(f, "{}", self.0)
            }
        }
    };
}

newtype!(UserId, String);
newtype!(Email, String);
newtype!(Port, u16);

// UserId and Email are now distinct types that won't accidentally mix
```

### Compile-time map

```rust
macro_rules! static_map {
    ( $( $key:expr => $val:expr ),* $(,)? ) => {
        |key| match key {
            $( $key => Some($val), )*
            _ => None,
        }
    };
}

let status_text = static_map! {
    200 => "OK",
    404 => "Not Found",
    500 => "Internal Server Error",
};

assert_eq!(status_text(200), Some("OK"));
assert_eq!(status_text(418), None);
```

## Common mistakes

**Forgetting that repetition variables must be used at the same depth:**

```rust
macro_rules! bad {
    ( $( $x:expr ),* ) => {
        // ERROR: $x is at depth 1, but used at depth 0
        println!("{}", $x);
    };
}
```

**Matching too greedily:**

```rust
macro_rules! too_greedy {
    ( $( $t:tt )* extra ) => { };
}

too_greedy!(a b c extra);
// The $($t:tt)* eats everything including "extra" - the arm never matches
```

`tt` is greedy. If you need a terminator, put distinctive syntax before the tail:

```rust
macro_rules! ok {
    ( $( $t:tt )* ; extra ) => { };
}
```

**Infinite recursion in TT munchers:**

If your base case doesn't match, the macro recurses forever until hitting the recursion limit (128 by default). Always test your base case first. You can raise the limit with `#![recursion_limit = "256"]`, but if you need to, your macro probably needs a redesign.

## Where to go from here

The [Little Book of Rust Macros](https://lukaswirth.dev/tlborm/) is the definitive reference. It covers patterns I didn't have space for here - push-down accumulation, callbacks, internal rules with `@` prefixes, and more.

For the standard library's own macros, read the source. `vec!` is [straightforward](https://github.com/rust-lang/rust/blob/master/library/alloc/src/macros.rs). `println!` chains through several layers. `assert_eq!` uses most of the techniques from this post. Reading working macros in real codebases teaches you patterns faster than any tutorial.

And when `macro_rules!` hits its limits - when you need to inspect struct fields, parse custom attributes, or generate code from external schemas - that's when proc macros enter the picture. But that's a different post.
