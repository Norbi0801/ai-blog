+++
title = "The visitor pattern in Rust - traversing complex data structures"
date = 2026-03-25
description = "Why the visitor pattern - usually a Java-OOP relic - is alive and well in Rust. How serde, syn, and rustc use it, when to reach for it instead of plain enum matching, and how to build one yourself."

[taxonomies]
tags = ["rust", "design-patterns", "ast", "compilers"]
+++

The visitor pattern has a bad reputation. In Java textbooks it shows up as a contortion to bolt double dispatch onto a language that does not have it. The classic GoF dance - `accept(Visitor v)` calling `v.visitConcreteType(this)` - feels like a workaround for missing language features, because that is exactly what it is. Rust has algebraic data types and exhaustive `match`, so the entire motivation seems to evaporate. Why would anyone reach for a visitor in a language where you can just write `match expr { ... }`?

And yet the most important crates in the ecosystem are built around visitors. `serde`'s entire deserialization protocol is a visitor protocol. `syn` exposes [`Visit`](https://docs.rs/syn/latest/syn/visit/trait.Visit.html) and [`VisitMut`](https://docs.rs/syn/latest/syn/visit_mut/trait.VisitMut.html) traits with hundreds of methods. `rustc_ast` ships its own `Visitor` trait that the compiler uses for almost every analysis pass. Clippy's lints walk the HIR through visitors. Every serious code-analysis or AST-transformation crate ends up rediscovering the same shape.

That is not an accident. Match works great when you write the consumer right next to the data type. It falls apart when you want **many different operations over the same recursive structure, written by different people, possibly in different crates**. That is the visitor's actual sweet spot, and Rust gets there with a slightly different mechanic than Java did.

If you have read [Serde deep dive - beyond derive](/blog/serde-deep-dive-beyond-derive/), you have already seen the pattern in action - this post zooms out to the general technique, when to use it, and what the alternatives cost.

<!-- more -->

## Two dispatch problems in one pattern

Strip the GoF ceremony and the visitor pattern is solving two related problems at once.

**Problem 1: separating algorithms from data.** You have a recursive type - an AST, a DOM tree, a JSON document - and you want to run several operations on it. Pretty-printing, evaluation, type checking, optimization, serialization. If every operation lives as a method on the data type, the data type becomes a god object, and adding a new operation in a downstream crate is impossible because Rust does not let you add inherent methods from outside. So you turn the algorithm into a separate object that "visits" the structure.

**Problem 2: dispatching on the dynamic shape of input you do not own.** Your code wants to handle different cases differently, but the input arrives through some opaque protocol - a `Deserializer`, a parser, a network frame. You cannot pattern-match on a trait object directly. So you give the input a callback bag (the visitor) with one method per case, and the input picks the right method. This is double dispatch: the *first* dispatch is dynamic (the deserializer's `deserialize_any`), the *second* is static (it calls a specific method on the visitor trait).

Java visitors mostly fix problem 1. Rust visitors lean harder into problem 2, which is why `serde::de::Visitor` looks the way it does - the deserializer is the unknown, and the visitor is the type-aware callback that the deserializer calls back into.

## The minimal Rust shape

Two traits. Burn this into memory:

```rust
pub trait Visitor {
    type Output;
    fn visit_num(&mut self, n: f64) -> Self::Output;
    fn visit_add(&mut self, lhs: &Expr, rhs: &Expr) -> Self::Output;
    fn visit_mul(&mut self, lhs: &Expr, rhs: &Expr) -> Self::Output;
    fn visit_var(&mut self, name: &str) -> Self::Output;
}

pub trait Visitable {
    fn accept<V: Visitor>(&self, visitor: &mut V) -> V::Output;
}
```

`Visitor` is the algorithm, parametrized by an associated `Output` type so the same trait works for "evaluate to f64", "render to String", "count nodes as usize", or "build a new AST". `Visitable` is the data structure's promise that it will route a visitor to the right method.

A concrete `Expr` and a `Visitable` impl:

```rust
pub enum Expr {
    Num(f64),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
    Var(String),
}

impl Visitable for Expr {
    fn accept<V: Visitor>(&self, visitor: &mut V) -> V::Output {
        match self {
            Expr::Num(n) => visitor.visit_num(*n),
            Expr::Add(l, r) => visitor.visit_add(l, r),
            Expr::Mul(l, r) => visitor.visit_mul(l, r),
            Expr::Var(name) => visitor.visit_var(name),
        }
    }
}
```

Notice the structure: `accept` does the `match` exactly once. Every visitor reuses that dispatch instead of writing its own. Now any number of visitors can ride on top of it.

```rust
pub struct Eval<'env> {
    pub env: &'env std::collections::HashMap<String, f64>,
}

impl<'env> Visitor for Eval<'env> {
    type Output = f64;

    fn visit_num(&mut self, n: f64) -> f64 { n }
    fn visit_add(&mut self, l: &Expr, r: &Expr) -> f64 {
        l.accept(self) + r.accept(self)
    }
    fn visit_mul(&mut self, l: &Expr, r: &Expr) -> f64 {
        l.accept(self) * r.accept(self)
    }
    fn visit_var(&mut self, name: &str) -> f64 {
        *self.env.get(name).unwrap_or(&0.0)
    }
}

pub struct Pretty;

impl Visitor for Pretty {
    type Output = String;

    fn visit_num(&mut self, n: f64) -> String { n.to_string() }
    fn visit_add(&mut self, l: &Expr, r: &Expr) -> String {
        format!("({} + {})", l.accept(self), r.accept(self))
    }
    fn visit_mul(&mut self, l: &Expr, r: &Expr) -> String {
        format!("({} * {})", l.accept(self), r.accept(self))
    }
    fn visit_var(&mut self, name: &str) -> String { name.to_string() }
}
```

`Eval` carries state (`env`) through `&mut self`. `Pretty` is stateless. Both implement the same trait and can be swapped at the call site:

```rust
let e = Expr::Add(
    Box::new(Expr::Var("x".into())),
    Box::new(Expr::Mul(Box::new(Expr::Num(2.0)), Box::new(Expr::Num(3.0)))),
);

let env = std::collections::HashMap::from([("x".into(), 10.0)]);
assert_eq!(e.accept(&mut Eval { env: &env }), 16.0);
assert_eq!(e.accept(&mut Pretty), "(x + (2 * 3))");
```

The recursion lives in the visitor methods, not in `accept`. That sounds like a small detail but it is the whole point - **the visitor decides how to walk the structure**. A `CountNodes` visitor walks every node. A `FindFirstVar` visitor short-circuits on the first hit. A `ConstFold` visitor recurses bottom-up and replaces. Same `accept`, completely different traversals.

## Why not just `match`?

For one-off code, you should use `match`. Visitors are syntactic overhead for no benefit when you have one algorithm and you control both the data type and the consumer. The classic functional style:

```rust
fn eval(e: &Expr, env: &HashMap<String, f64>) -> f64 {
    match e {
        Expr::Num(n) => *n,
        Expr::Add(l, r) => eval(l, env) + eval(r, env),
        Expr::Mul(l, r) => eval(l, env) * eval(r, env),
        Expr::Var(n) => *env.get(n).unwrap_or(&0.0),
    }
}
```

This is shorter, clearer, and equally efficient. The compiler will likely inline it identically. So when does the visitor earn its keep?

**1. The data type lives in another crate.** You cannot add `impl Expr { fn type_check(&self) }` to a type defined in `syn` or `swc_ecma_ast`. You either write a free function with a giant `match` (and copy that match every time you write a new pass), or you implement the crate's `Visitor` trait once per pass.

**2. The structure has dozens of variants.** `syn::Expr` has [over 40 variants](https://docs.rs/syn/latest/syn/enum.Expr.html). `rustc_ast::Expr` has roughly 50. Writing a recursive `match` that defaults most arms to "recurse on children, do nothing here" is hundreds of lines of boilerplate per pass. The visitor pattern lets you provide *default methods* that walk the children, so each visitor only overrides the variants it cares about. That changes a 200-line match into a 5-line override.

**3. The default traversal is reusable.** If half your visitors want pre-order DFS, half want post-order, and a few want BFS, you encode each as a default `walk_*` method or as a separate trait implementation. Match cannot share a traversal across functions without explicit higher-order plumbing.

**4. You need stateful traversal.** Visitors hold mutable state across the walk - a current scope, a depth counter, a list of diagnostics. A function with an accumulator parameter does the same thing but every recursive call has to thread the parameter manually. The visitor's `&mut self` keeps that state implicit.

**5. The walk is driven by something else.** Serde's `Visitor` is the canonical case: *the deserializer drives the walk*, not your code. You hand the visitor over and the deserializer calls the right method based on what it finds in the input. There is no `match` to write because the data does not exist yet - it is being parsed.

If none of those apply, use `match`. If two or more apply, you probably want a visitor.

## How serde uses visitors

The `serde::de::Visitor` trait is one of the cleanest applications of the pattern in any language. The data model has 29 abstract types ([covered in the parent post](/blog/serde-deep-dive-beyond-derive/)), and the [`Visitor`](https://docs.rs/serde/latest/serde/de/trait.Visitor.html) trait has one method per type:

```rust
pub trait Visitor<'de>: Sized {
    type Value;
    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result;

    fn visit_bool<E: Error>(self, v: bool) -> Result<Self::Value, E> { ... }
    fn visit_i64<E: Error>(self, v: i64) -> Result<Self::Value, E> { ... }
    fn visit_str<E: Error>(self, v: &str) -> Result<Self::Value, E> { ... }
    fn visit_borrowed_str<E: Error>(self, v: &'de str) -> Result<Self::Value, E> { ... }
    fn visit_seq<A: SeqAccess<'de>>(self, seq: A) -> Result<Self::Value, A::Error> { ... }
    fn visit_map<A: MapAccess<'de>>(self, map: A) -> Result<Self::Value, A::Error> { ... }
    // ... 23 more
}
```

Default implementations all return a type error ("expected X, found Y"). When you implement `Deserialize` for a type that accepts both an integer and a string (the [string-or-struct pattern](/blog/serde-deep-dive-beyond-derive/#string-or-struct-pattern)), you override exactly the visitor methods you want to accept and let the others default to errors.

The double dispatch goes:

1. You call `serde_json::from_str::<MyType>(input)`.
2. `<MyType as Deserialize>::deserialize(deserializer)` runs. This is **dispatch 1**: which `Deserialize` impl?
3. Inside that impl, you build a `MyTypeVisitor` and pass it to the deserializer (`deserializer.deserialize_any(visitor)`).
4. The deserializer reads from JSON, sees a string, and calls `visitor.visit_str("...")`. This is **dispatch 2**: which case did the input land in?
5. Your `visit_str` constructs the final `MyType`.

Step 4 is the magic: the deserializer can hand the same visitor to whichever `visit_*` method matches the input. JSON gives you a string, BSON gives you a number, postcard gives you raw bytes - the visitor's job is to handle any of them. Match cannot do this because there is no value to match on yet; the deserializer has to *call you back*.

The shape repeats inside the data. Once `visit_seq` is called, you get a `SeqAccess` and call `seq.next_element::<T>()` in a loop, which kicks off another visitor walk for `T`. The whole protocol is coroutine-shaped: deserializer yields, visitor consumes, recursion on demand.

## syn and AST transformation

The other huge user of visitors in the Rust ecosystem is [`syn`](https://docs.rs/syn), the parser library powering essentially every proc macro. Once you parse a token stream into a `syn::File`, you almost never want to write `match` against the resulting tree - it has too many variants, and you usually only care about three of them.

Enable the feature flags and you get two traits:

```toml
[dependencies]
syn = { version = "2", features = ["full", "visit", "visit-mut"] }
```

`syn::visit::Visit<'ast>` for read-only walks. `syn::visit_mut::VisitMut` for in-place rewriting. Each has a method per AST node, and each provides a *default* implementation that recurses into the children. Override the methods for the nodes you care about, then call `syn::visit::visit_file` (or whichever entry point) and you are done.

A real-world example: a proc macro that wants to find every `await` point in a function body. With pure `match` you would write a recursive function with 40+ arms covering every `Expr` variant. With the visitor:

```rust
use syn::visit::{self, Visit};

#[derive(Default)]
struct AwaitFinder {
    spans: Vec<proc_macro2::Span>,
}

impl<'ast> Visit<'ast> for AwaitFinder {
    fn visit_expr_await(&mut self, node: &'ast syn::ExprAwait) {
        self.spans.push(node.await_token.span);
        // Don't forget to recurse into the inner expression.
        visit::visit_expr_await(self, node);
    }
}

fn collect_awaits(file: &syn::File) -> Vec<proc_macro2::Span> {
    let mut finder = AwaitFinder::default();
    finder.visit_file(file);
    finder.spans
}
```

That is the entire pass. You overrode one method out of hundreds. The default impls handle the recursion into every other variant, into nested closures, into match arms, into struct field initializers, all the way down. This is something `match` simply cannot give you without a 500-line recursive helper, and even then you would have to keep it in sync with `syn` releases that add new variants.

`VisitMut` works the same way for transformations. The compiler-internal [`rustc_ast::mut_visit`](https://github.com/rust-lang/rust/blob/master/compiler/rustc_ast/src/mut_visit.rs) follows the identical shape, just hand-rolled rather than generated. Clippy lints over HIR, the [`async-trait`](https://github.com/dtolnay/async-trait) macro rewrites function bodies, [`tracing-attributes`](https://docs.rs/tracing-attributes/) inserts spans - all visitors.

## Mutating walks: the borrow checker problem

Read-only visitors are easy. Mutating visitors that *replace nodes during traversal* hit the borrow checker hard, and it is worth seeing why.

A naive `VisitorMut::visit_expr(&mut self, e: &mut Expr)` cannot replace `*e` with a new `Expr` if any field of `*e` is being borrowed during the call - you cannot move out of `*e` while there is an active borrow into it. The standard trick is `std::mem::replace` or `std::mem::take` to swap the node out, work on the owned value, and put a new one back:

```rust
pub trait VisitorMut {
    fn visit_expr(&mut self, e: &mut Expr) {
        // Default: walk children.
        match e {
            Expr::Add(l, r) | Expr::Mul(l, r) => {
                self.visit_expr(l);
                self.visit_expr(r);
            }
            _ => {}
        }
    }
}

pub struct ConstFold;

impl VisitorMut for ConstFold {
    fn visit_expr(&mut self, e: &mut Expr) {
        // Walk children first (post-order).
        match e {
            Expr::Add(l, r) | Expr::Mul(l, r) => {
                self.visit_expr(l);
                self.visit_expr(r);
            }
            _ => {}
        }
        // Now look at this node.
        let folded = match e {
            Expr::Add(l, r) => match (&**l, &**r) {
                (Expr::Num(a), Expr::Num(b)) => Some(Expr::Num(a + b)),
                _ => None,
            },
            Expr::Mul(l, r) => match (&**l, &**r) {
                (Expr::Num(a), Expr::Num(b)) => Some(Expr::Num(a * b)),
                _ => None,
            },
            _ => None,
        };
        if let Some(new_node) = folded {
            *e = new_node;
        }
    }
}
```

The pattern post-order recurses, then rewrites in place via `*e = ...`. `mem::replace` is required when you need to move out of `e` to compute the new node from the old fields:

```rust
let old = std::mem::replace(e, Expr::Num(0.0));
let new = transform(old);
*e = new;
```

This dance is why crates like [`syn::visit_mut`](https://docs.rs/syn/latest/syn/visit_mut/index.html) provide pre-built `visit_mut_expr` helpers. They have already worked out the borrow-checker gymnastics for every variant.

## Trait objects vs generics for visitors

Look closely at how `Visitable::accept` is parametrized:

```rust
fn accept<V: Visitor>(&self, visitor: &mut V) -> V::Output;
```

This is a generic, monomorphized at compile time. Each concrete visitor produces its own copy of `accept`, the compiler inlines the dispatch, and the visitor often compiles down to the same code as a hand-written `match`. Zero overhead.

The cost: you cannot store visitors in a `Vec<Box<dyn Visitor>>`. Trait objects need `dyn`-compatibility, which means no associated types referenced through the trait object, no generic methods, and `Self: Sized` clauses everywhere they are needed. That is fine for serde (you only need one visitor per `Deserialize` impl) but limiting if you want to plug in passes dynamically.

If you need dynamic visitors, drop the associated `Output` type and accept `Box<dyn FnMut(...)>` callbacks, or use an enum of known visitor types. [Trait objects vs enums vs generics](/blog/trait-objects-vs-enums-vs-generics-picking-the-right-polymorphism-in-rust) covers the trade-offs in detail.

For most AST work you want generics. The compiler turns the visitor into the same machine code as a hand-rolled function, plus you get the compositional benefits.

## When to reach for it

A short decision flowchart:

- **Single operation, small enum, code in this file?** Use `match`. Move on.
- **One operation, but the enum has 40+ variants and you only care about three?** If you own the type, add a default-walking `walk` helper. If you do not, write a visitor.
- **Multiple operations on the same recursive structure, possibly defined in different crates?** Visitor.
- **The traversal is driven by external code (parser, deserializer, async stream)?** Visitor - probably not optional, the upstream API will demand it.
- **You need to rewrite nodes in place during the walk?** Mutating visitor with `mem::replace` semantics.

The pattern is not a default. It is the right answer for a specific class of problems - large, frozen, recursive data structures with many distinct passes - and the wrong answer almost everywhere else. Reach for it when you find yourself writing the same big `match` for the third time, or when the input is something you do not own. Otherwise, exhaustive `match` is the better Rust idiom.

What makes Rust's flavor of the visitor pattern interesting is that the compiler does most of the heavy lifting that Java visitors needed runtime double-dispatch for. Generic `accept`, monomorphized callbacks, exhaustiveness checks on the underlying `match` - the costs that made the GoF version feel awkward are gone, and what is left is a clean separation of "what is the data" from "what do I want to do with it". That is why the pattern keeps showing up in the most performance-sensitive crates in the ecosystem, even though, on the surface, it should not need to.
