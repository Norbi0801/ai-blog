+++
title = "How Shopify uses Rust for Ruby performance - the YJIT story"
date = 2025-10-05
description = "Inside YJIT: how Shopify built a JIT compiler in Rust that makes Ruby 92% faster, processed 80M req/s on Black Friday, and what lazy basic block versioning actually means."

[taxonomies]
tags = ["rust", "ruby", "compilers", "performance"]
+++

Most "rewrite it in Rust" stories follow a predictable pattern: take a service written in Go/Python/Java, rewrite it in Rust, show benchmark improvements, celebrate. Shopify did something more interesting. They used Rust not to replace Ruby, but to make Ruby itself faster. The result is YJIT - a JIT compiler that lives inside CRuby, written in Rust, that now handles 80 million requests per minute on Shopify's production infrastructure during Black Friday.

If you read my earlier post on [Rust in production](/blog/rust-in-production-what-companies-actually-use-it-for/), I briefly mentioned Shopify's YJIT as an example of "compiler/runtime tooling" - one of the patterns where Rust adoption actually makes sense. This post goes deep on the technical details: the compiler architecture, why they chose Rust, how lazy basic block versioning works, and the actual production numbers.

<!-- more -->

## The problem: Ruby is slow and everyone knows it

Ruby on Rails powers a massive portion of the web. GitHub, Shopify, Basecamp, Airbnb - these are not small services. But Ruby's execution speed has always been its weakest point. CRuby (the reference implementation) is an interpreter. It reads Ruby source code, compiles it to YARV (Yet Another Ruby VM) bytecode, and then interprets that bytecode one instruction at a time.

Interpretation is inherently slower than native execution. Every bytecode instruction goes through a dispatch loop - read the opcode, look up the handler, call it, repeat. Each dispatch has branch prediction costs. Each handler does type checking at runtime because Ruby is dynamically typed. A simple `a + b` might call integer addition, float addition, string concatenation, or a custom `+` method depending on the runtime types of `a` and `b`. The interpreter checks this every single time the instruction executes.

JIT compilation attacks this problem directly. Instead of interpreting bytecode repeatedly, you compile hot code paths to native machine code. If you've observed that `a + b` uses integers in a particular method 10,000 times in a row, you can generate x86/ARM64 instructions that do integer addition directly - skipping the type check, the dispatch, and the method lookup.

Ruby has had JIT compilers before YJIT. MJIT (Method-based JIT, later renamed RJIT) shipped with Ruby 2.6 in 2018. It worked by generating C code from Ruby bytecode, writing it to a file, and calling GCC/LLVM to compile it. The problem: compilation was slow. Generating C, writing to disk, invoking an external compiler, loading the shared library - this whole pipeline took so long that short-lived methods were cold by the time their compiled code was ready. MJIT improved micro-benchmarks but barely moved the needle on real Rails applications.

JRuby and TruffleRuby are alternative Ruby implementations that achieve better performance through the JVM and GraalVM respectively. But they can't run C extensions (a massive chunk of the Ruby ecosystem depends on C extensions), they use significantly more memory, and they're not drop-in replacements for CRuby.

Shopify needed something that compiled fast, ran inside CRuby, supported all existing C extensions, and didn't bloat memory usage. That's a tall order.

## What YJIT actually is

YJIT (Yet Another JIT) was created by Maxime Chevalier-Boisvert at Shopify. It's not an external tool or a separate Ruby implementation - it's a JIT compiler that lives inside CRuby itself, compiling YARV bytecode to native machine code at runtime.

The key innovation is the compilation strategy: lazy basic block versioning (LBBV). I'll explain the internals shortly, but the high-level idea is that YJIT doesn't try to compile entire methods at once. It compiles one basic block at a time, only when execution reaches that block, and it generates specialized versions of each block based on the types it actually observes at runtime.

This design gives YJIT three properties that matter for production Ruby:

1. **Near-instant warmup** - no waiting for profile data or tracing. Code is compiled the first time a block executes with a particular type context.
2. **Low memory overhead** - you only compile code paths that actually execute, not the entire method body.
3. **Simple architecture** - no intermediate representation (in early versions), no register allocator, no complex optimization passes. Just direct bytecode-to-machine-code translation with type specialization.

YJIT shipped in Ruby 3.1 (December 2021) as an experimental feature. It became production-ready in Ruby 3.2, and as of Ruby 3.4 it's ~92% faster than the interpreter across headline benchmarks.

## The architecture: lazy basic block versioning

To understand how YJIT works, you need to understand basic blocks and why versioning them is powerful.

A basic block is a sequence of instructions with one entry point and one exit point. No branches in the middle - execution flows straight through from top to bottom. When you hit a conditional (like an `if` statement), that's where one basic block ends and two or more begin.

In a traditional JIT, you'd compile an entire method into machine code, insert type guards (runtime checks) before operations that depend on types, and execute. If a type guard fails, you deoptimize - throw away the compiled code and fall back to the interpreter.

YJIT takes a different approach with LBBV. Here's how it works, step by step:

**Step 1: Execution reaches a new basic block.** Let's say the interpreter is executing method `calculate_total` and hits a block that starts with `a + b`. YJIT intercepts this.

**Step 2: YJIT inspects the current type context.** It looks at the actual runtime types of the values on the stack and in local variables. Say `a` is a `Fixnum` (63-bit integer) and `b` is a `Fixnum`.

**Step 3: YJIT generates a *version* of this block specialized for that context.** It emits machine code that assumes `a` and `b` are both `Fixnum`, does the integer addition directly, and propagates the resulting type information (the result is also a `Fixnum`) to the *successor* blocks.

**Step 4: At the exit of this block, YJIT generates a stub** - a small piece of code that, when hit, will trigger compilation of the next block with the propagated type context.

**Step 5: When execution reaches the stub, YJIT compiles the next block**, specialized for the type context that was propagated from the predecessor. The stub gets patched to jump directly to the newly compiled block.

This is the "lazy" part. Code is only compiled when execution reaches it. And it's the "versioning" part - if the same basic block is later reached with different types (say `a` is a `Float`), YJIT generates a *new version* of that block specialized for floats. Multiple versions of the same block can coexist, each optimized for different type contexts.

The beauty of this approach is that type propagation comes for free. Traditional JITs need complex type inference passes - dataflow analysis, abstract interpretation, or speculative profiling. YJIT just... observes what types are actually there and propagates that knowledge forward through the control flow graph as it compiles block by block.

Maxime Chevalier-Boisvert's [academic paper](https://arxiv.org/abs/1411.0352) on lazy basic block versioning showed that this technique eliminates an average of 71% of type checks without any costly program analysis. The algorithm's simplicity - no fixed-point iteration, no worklist algorithm, no phi nodes - is what makes it suitable as a baseline JIT inside an interpreter.

Here's a simplified example to make this concrete:

```ruby
def process(items)
  total = 0
  items.each do |item|
    total += item.price  # called 10,000 times
  end
  total
end
```

Without YJIT, every iteration of that block does: dispatch the `price` method call (check if `item` has a `price` method, look it up in the method table), dispatch the `+` call (check types of `total` and the return value of `price`), and update `total`.

With YJIT, after the first iteration where it observes that `item` is a `Product` and `price` returns a `Fixnum`, it generates machine code that:
- Checks that `item` is still a `Product` (a single type tag comparison)
- Inlines the `price` method call (if it's a simple attribute reader)
- Does a direct integer addition instead of a method dispatch for `+`
- If any guard fails, takes a "side exit" back to the interpreter

The side exit mechanism is critical. YJIT never crashes or produces wrong results. If the type assumptions are wrong (say someone passes a `SpecialProduct` subclass that overrides `price`), execution falls back to the interpreter. Correctness is always preserved.

## The Rust port: from C to Rust in three months

YJIT was originally written in C99, which made sense for integration with CRuby (which is itself written in C). But as the project grew beyond 11,000 lines of C code, the team hit the ceiling of what C could comfortably manage.

The decision to port to Rust came from Alan Wu, a senior developer on the YJIT team. Maxime was immediately interested. They briefly considered Zig (which would have been feasible since YJIT has minimal dependencies), but chose Rust for its maturity and larger community.

Four of the six-person team actively participated in the port. It took three months. The strategy was pragmatic: translate C to Rust more or less directly, function by function, struct by struct. No major architectural changes during the port.

### What Rust gave them

**Pattern matching and enums.** A JIT compiler is fundamentally a big match on instruction types. In C, this was a chain of `if/else` or `switch` statements with no exhaustiveness checking. Miss a case and you get silent wrong behavior at runtime. Rust's `match` on enums is exhaustive - add a new bytecode instruction handler and the compiler tells you every place you need to update.

```rust
// Simplified example of what YJIT codegen looks like
match insn {
    Insn::OptPlus { left, right } => {
        // Generate specialized addition code
        gen_opt_plus(ctx, left, right)
    }
    Insn::OptMinus { left, right } => {
        gen_opt_minus(ctx, left, right)
    }
    Insn::Send { method, argc } => {
        gen_send(ctx, method, argc)
    }
    // ... every instruction must be handled
}
```

**The macro system.** YJIT uses Rust macros extensively for code generation patterns that repeat across architectures (x86-64 and ARM64). Compared to C preprocessor macros, Rust macros are hygienic, support pattern matching syntax, and produce useful error messages. The team called this "a huge improvement over C preprocessor macros in both safety and ergonomics."

**Cargo and conditional compilation.** YJIT supports multiple platforms (x86-64, ARM64) and build configurations (debug, release, with/without stats). In C, this meant a maze of `#ifdef` blocks. Cargo features make this declarative and composable.

**No garbage collector.** A JIT compiler manages executable memory pages directly. It allocates code buffers, writes machine instructions into them, patches jump targets, and eventually frees code that's no longer reachable. Doing this inside a language with a GC would be a nightmare - the GC might move your code pages, or your code pages might confuse the GC's pointer scanning. Rust's manual memory management (with compile-time safety) is ideal here.

### What was painful

The port wasn't all smooth. The team documented the friction honestly in their [engineering blog post](https://shopify.engineering/porting-yjit-ruby-compiler-to-rust):

**Bindgen limitations.** YJIT needs to call into CRuby's C API - a lot. The `bindgen` tool auto-generates Rust FFI bindings from C headers, but it silently failed to export certain definitions with no error messages. The team had to manually supplement bindgen with bindings for about 140 functions, 30 structs/unions, and 500 constants.

**Integer casting.** Rust requires explicit casts between integer types. CRuby's API uses a mix of `VALUE` (pointer-sized unsigned int), `long`, `int`, `size_t`, and various other integer types. Every boundary crossing needed explicit casts, making the FFI code noisy and verbose.

**Cyclic data structures.** The control flow graph (CFG) is inherently cyclic - blocks point to their successors, and successors point back to predecessors. In C, you just use raw pointers. In Rust, cyclic references require either `unsafe` raw pointers, `Rc<RefCell<T>>`, or an arena allocator. The team used `Rc<RefCell<T>>` initially, which introduced subtle bugs with reference counting for their machine code memory management.

**Unsafe code volume.** Every call into CRuby's C API requires an `unsafe` block. With hundreds of FFI call sites, this created visual noise throughout the codebase. The `unsafe` keyword is supposed to mark code that needs extra scrutiny, but when every other function has one, the signal-to-noise ratio drops.

Despite these issues, the team's verdict was clear: the Rust codebase is "much more maintainable and pleasant to work with than the original C codebase." The port "brought fresh energy into the project."

## Source code walkthrough

YJIT's Rust code lives in `yjit/src/` within the [CRuby repository](https://github.com/ruby/ruby). Here are the key files:

- **`yjit/src/core.rs`** - the heart of YJIT. Basic block versioning logic, block management, the type context (`Ctx`) that tracks known types for stack values and locals, and the stub/patching mechanism.
- **`yjit/src/codegen.rs`** - bytecode-to-machine-code translation. One function per YARV instruction. This is the biggest file and the one that grows when new Ruby features need JIT support.
- **`yjit/src/asm/`** - the in-memory assembler. Two backends: x86-64 and ARM64. These are custom assemblers written from scratch in Rust - not wrappers around LLVM or any external library.
- **`yjit/src/cruby.rs`** - manually written C bindings that supplement bindgen's auto-generated output.
- **`yjit/src/stats.rs`** - runtime statistics collection for performance analysis (enabled with `--yjit-stats`).
- **`yjit/src/options.rs`** - command-line argument parsing for YJIT-specific flags.

The C integration layer consists of `yjit.c` and `yjit.h` on the CRuby side, plus `yjit.rb` which exposes a Ruby module for runtime inspection.

The compilation pipeline looks like this:

```
Ruby source code
       |
       v
YARV bytecode (CRuby compiler - written in C)
       |
       v
YJIT intercepts execution at call threshold (default: 30 calls)
       |
       v
codegen.rs: translate bytecode -> IR operations
       |
       v
asm/: emit native machine code (x86-64 or ARM64)
       |
       v
Executable memory region (inline + outlined code pages)
       |
       v
Direct execution - interpreter bypassed for compiled blocks
```

YJIT uses two distinct memory regions: **inline code** for the main execution path, and **outlined code** for infrequently-executed paths (like side exits back to the interpreter). Both fall under a configurable memory limit (`--yjit-mem-size`, default 128 MiB).

A method triggers compilation after it's been called 30 times (configurable via `--yjit-call-threshold`). This threshold automatically increases to 120 once 40,000+ instruction sequences accumulate in the process - an adaptive mechanism that prevents memory bloat in large applications.

## Production numbers

This is where YJIT earns its keep. Benchmarks are nice, but Shopify runs YJIT on their Storefront Renderer (SFR) - the service that renders every Shopify store's customer-facing pages. This is real production traffic.

### Ruby 3.2 (production-ready release)

Shopify deployed YJIT globally across their entire SFR infrastructure, which handles [over 75 million requests per minute](https://shopify.engineering/ruby-yjit-is-production-ready). The measured speedup: **5-10% reduction in end-to-end request completion time**, depending on time of day.

That might sound modest compared to the 38% benchmark improvements, and there's a good reason for the gap. Real Rails applications spend significant time in I/O - database queries, cache reads, external API calls. YJIT only speeds up CPU-bound Ruby execution. If a request spends 30% of its time executing Ruby and 70% waiting on I/O, a 38% improvement to the Ruby portion translates to roughly 11% end-to-end.

Memory overhead was cut to approximately one-third of what it was in Ruby 3.1, through three optimizations: metadata space compaction, garbage collection for unused compiled machine code, and lazy allocation of memory pages (don't allocate a page until code actually gets written to it).

### Ruby 3.3

Enabling YJIT on Ruby 3.3 gives a [15% speedup](https://railsatscale.com/2023-09-18-ruby-3-3-s-yjit-runs-shopify-s-production-code-15-faster/) over the interpreter on Shopify's production workload. Upgrading from Ruby 3.2+YJIT to 3.3+YJIT: 13% additional speedup on average.

Memory usage actually went *down* compared to Ruby 3.2's YJIT, despite compiling more code. The team achieved this through more aggressive code GC and better metadata encoding.

### Ruby 3.4

The latest numbers are impressive. YJIT 3.4 is [~92% faster than the interpreter](https://railsatscale.com/2025-01-10-yjit-3-4-even-faster-and-more-memory-efficient/) across headline benchmarks. That's nearly 2x the performance of the plain interpreter.

On Black Friday/Cyber Monday 2024, Shopify processed **$11.5 billion in global sales**, with app servers running a prerelease version of YJIT 3.4 handling **over 80 million requests per minute**. Latency was slightly better at both p50 and p99 compared to the previous year - and the previous year was already running YJIT.

A key improvement in 3.4 is method inlining. On the Lobsters benchmark (a real Rails app), YJIT inlines **56.3% of C method calls** and 4.8% of Ruby calls. On liquid-render (Shopify's templating engine), those numbers jump to **82.5% of C calls** and 7.6% of Ruby calls inlined. Inlining eliminates call overhead and opens up further optimization opportunities within the inlined code.

### YJIT vs other Ruby JITs

YJIT is the most memory-efficient Ruby JIT. JRuby and TruffleRuby both achieve higher peak throughput on certain benchmarks, but they use significantly more memory and can't run CRuby's C extensions. For a real Rails application with dozens of gems (many with C extensions), YJIT is the only practical JIT option.

| | YJIT 3.4 | TruffleRuby | JRuby |
|---|---|---|---|
| Peak speedup (benchmarks) | ~92% over interp | Higher on some | Higher on some |
| Memory overhead | ~21% over interp | 3-5x | 2-3x |
| C extension support | Full | Limited | None |
| Warmup time | Near-instant | Minutes | Seconds |
| Drop-in CRuby replacement | Yes | No | No |

## Why this matters beyond Ruby

YJIT is one of the clearest examples of a pattern I think will define the next decade of language tooling: **using Rust to build performance-critical infrastructure for other language ecosystems**.

Consider what happened here. Shopify didn't ask Ruby developers to learn Rust. They didn't rewrite their Rails application. They used Rust at the compiler level, inside CRuby, where it's invisible to application developers. Every Ruby developer running Ruby 3.1+ benefits from Rust without writing a single line of it. Since Ruby 3.3, YJIT is enabled by default when `--yjit` is passed, and it's heading toward being enabled by default with no flags at all.

This pattern is showing up everywhere:

- **[Ruff](https://github.com/astral-sh/ruff)** - a Python linter written in Rust, 10-100x faster than existing Python linters
- **[SWC](https://github.com/swc-project/swc)** - a JavaScript/TypeScript compiler written in Rust, replacing Babel
- **[Biome](https://github.com/biomejs/biome)** - a JavaScript toolchain (formatter, linter) in Rust
- **[Oxc](https://github.com/oxc-project/oxc)** - JavaScript parser, linter, and bundler in Rust
- **[uv](https://github.com/astral-sh/uv)** - a Python package manager written in Rust, replacing pip
- **[Deno](https://deno.com/)** - a JavaScript runtime with Rust internals using V8
- **[rspack](https://github.com/web-infra-dev/rspack)** - a Webpack-compatible bundler in Rust

The common thread: these tools don't require their users to know Rust. A Python developer using Ruff or a JavaScript developer using SWC experiences Rust's performance benefits transparently. Rust becomes the implementation language for developer tooling, not the language developers write their applications in.

This is arguably a more impactful form of Rust adoption than rewriting web services. When you rewrite a service, you improve performance for one team. When you rewrite a compiler or build tool, you improve performance for every developer in that ecosystem.

## ZJIT: what's next

The YJIT team isn't resting. They're already building the successor: [ZJIT](https://rubykaigi.org/2025/presentations/maximecb.html), a next-generation Ruby JIT that was announced at RubyKaigi 2025 and has been [merged into Ruby 4.0](https://railsatscale.com/2025-12-24-launch-zjit/).

YJIT's lazy basic block versioning is simple and effective, but it has limits. It doesn't optimize across basic block boundaries. It doesn't have a register allocator. It can't do loop-invariant code motion or common subexpression elimination - the kinds of optimizations you'd find in a mature optimizing compiler.

ZJIT addresses this with a fundamentally different architecture: a method-based compiler using Static Single Assignment (SSA) form. SSA is the standard intermediate representation used by LLVM, GCC, and HotSpot - it's the foundation of modern compiler optimization. ZJIT translates YARV bytecode into an SSA-based HIR (High-level Intermediate Representation), which enables all the classical optimizations:

- Constant propagation and folding
- Dead code elimination
- Type propagation across entire methods (not just block-by-block)
- Common subexpression elimination
- Inlining with full inter-procedural analysis

ZJIT is compiled by default in Ruby 4.0 but not enabled by default - it's still experimental. The team is honest about its current state: faster than the interpreter but not yet matching YJIT's performance. The goal is to surpass YJIT once the optimization passes mature.

And yes, ZJIT is also written in Rust. The Rust port of YJIT wasn't just a one-time investment - it established Rust as the implementation language for Ruby's JIT infrastructure going forward.

## What I take away from this

Three things stand out from the YJIT story:

**Incremental adoption works.** Shopify didn't ask the Ruby community to accept a massive change. YJIT started as an experimental flag in Ruby 3.1, became production-ready in 3.2, got faster in 3.3 and 3.4, and is now the default compilation strategy. Each step delivered measurable value. Each step was reversible (just don't pass `--yjit`). This is how you get language-level changes adopted.

**Rust's safety guarantees matter most in infrastructure code.** A bug in your web application might show a wrong product price. A bug in a JIT compiler might silently generate wrong machine code that corrupts memory in ways that are nearly impossible to debug. Rust doesn't just prevent segfaults in YJIT - it makes the YJIT developers more productive because they spend less time tracking down memory corruption bugs and more time implementing optimizations. The team explicitly said the Rust codebase "brought fresh energy into the project."

**The "rewrite in Rust" meme is evolving.** The early Rust adoption stories were about rewriting services (Discord, Figma). The current wave is about rewriting *tooling* - compilers, linters, package managers, bundlers, runtimes. Tools that sit underneath application code and amplify performance for entire ecosystems. YJIT is the prototype of this pattern: a small team (six people, four actively porting) spent three months on a Rust port, and the result speeds up every Ruby application on the planet.

That's a pretty good return on investment.
