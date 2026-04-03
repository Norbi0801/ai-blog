+++
title = "Debugging Rust Beyond println!"
date = 2026-04-02
description = "A practical guide to Rust debugging tools: dbg!, RUST_BACKTRACE, cargo-expand, rust-gdb, CodeLLDB, tracing, core dumps, and workflows for different bug types."

[taxonomies]
tags = ["rust", "debugging", "tools", "developer-experience"]
+++

Every Rust developer's first debugger is `println!`. It works. You sprinkle it around, recompile, squint at the output, add more prints, recompile again. Eventually you find the bug, delete the print statements, and move on. But this gets painful fast - especially with async code, macro-heavy codebases, or bugs that only show up under specific conditions.

Rust ships with a surprisingly good debugging story that most developers never fully explore. This post walks through the tools and techniques that replaced most of my `println!` usage, organized by the type of problem they solve best.

<!-- more -->

## dbg! - println's smarter sibling

The `dbg!` macro is the single easiest upgrade from `println!`. It prints the file, line number, the expression itself, and its value - all to stderr.

```rust
let width = 10;
let height = dbg!(width * 2);
// stderr: [src/main.rs:3:18] width * 2 = 20
```

What makes `dbg!` actually useful is that it returns the value it prints. You can drop it inline without restructuring your code:

```rust
fn process(items: Vec<Item>) -> Vec<Item> {
    items
        .into_iter()
        .filter(|item| dbg!(item.is_valid()))
        .map(|item| dbg!(transform(item)))
        .collect()
}
```

Every call prints the expression, its result, and the exact location. Compare that to writing `println!("item.is_valid() at line 4 = {}", item.is_valid())` by hand.

A few things to know about `dbg!`:

**It moves the value.** The macro takes ownership and returns it. For `Copy` types this is invisible. For non-`Copy` types, pass a reference:

```rust
let name = String::from("rust");
dbg!(&name);  // borrows, doesn't move
println!("{}", name);  // still valid
```

**Multiple arguments become a tuple:**

```rust
let a = 1;
let b = 2;
let (x, y) = dbg!(a, b);
// stderr: [src/main.rs:4:19] (a, b) = (1, 2)
```

**It works in release builds.** Unlike C's `assert`, `dbg!` isn't stripped in `--release`. This is intentional - sometimes the bug you're chasing only reproduces in release mode. Just remember to remove it before shipping.

**Empty invocation returns unit:**

```rust
dbg!();
// stderr: [src/main.rs:1:1]
```

This is occasionally useful as a "did execution reach here?" marker.

Under the hood, `dbg!` is straightforward. The [implementation in std](https://doc.rust-lang.org/src/std/macros.rs.html) calls `eprintln!` with the `file!()`, `line!()`, `stringify!()` of the expression, and the `Debug` format of the value. It requires `Debug` on the type - if your type doesn't implement it, you'll get a compile error pointing at the `dbg!` call, which is actually a nice nudge to add `#[derive(Debug)]`.

## RUST_BACKTRACE - what happened before the panic

When your program panics, Rust gives you a message but not the call chain that led there. Setting `RUST_BACKTRACE` changes that:

```bash
RUST_BACKTRACE=1 cargo run
```

Now panics include a stack trace. There are two levels:

- `RUST_BACKTRACE=1` - shows the stack frames relevant to your code, filtering out most of the runtime internals (thread setup, panic handler guts)
- `RUST_BACKTRACE=full` - shows every frame, including Rust runtime initialization and panic infrastructure

In practice, `1` is what you want 90% of the time. The `full` output is useful when you suspect the issue is in how your code interacts with the runtime - for example, a thread panic during startup or a signal handler issue.

```
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 5',
    src/main.rs:12:15
stack backtrace:
   0: std::panicking::begin_panic_handler
   1: core::panicking::panic_fmt
   2: core::panicking::panic_bounds_check
   3: myapp::process_data
             at ./src/main.rs:12:15
   4: myapp::main
             at ./src/main.rs:5:5
```

A couple of practical notes:

**Debug symbols matter.** The backtrace is only useful if your binary has debug info. In dev builds (`cargo build`) you get them by default. In release builds, add this to `Cargo.toml`:

```toml
[profile.release]
debug = true
```

This increases binary size but keeps full symbol names and line numbers. The performance impact is zero - debug info lives in a separate section and isn't loaded during normal execution.

**Set it in your shell profile.** I have `export RUST_BACKTRACE=1` in my `.bashrc`. The overhead on normal execution is negligible - the backtrace machinery only kicks in on panic, and if you're panicking, you want the trace.

**For tests too.** `RUST_BACKTRACE=1 cargo test` gives you stack traces for test failures, which makes tracking down assertion failures much faster.

## cargo-expand - seeing what macros actually generate

Macros are one of Rust's power tools, but when a derive macro produces a confusing error or your `macro_rules!` doesn't behave as expected, you're debugging code you can't see. [cargo-expand](https://github.com/dtolnay/cargo-expand) makes that invisible code visible.

```bash
cargo install cargo-expand
```

It requires a nightly toolchain installed (though it doesn't need to be your default):

```bash
rustup install nightly
cargo expand
```

This outputs the fully expanded source of your crate - every `#[derive]`, every `macro_rules!` invocation, every proc macro - all resolved into plain Rust.

Here's a real example. Say you have:

```rust
#[derive(Debug, Clone)]
struct Point {
    x: f64,
    y: f64,
}
```

Running `cargo expand` shows what `Debug` and `Clone` actually generate:

```rust
struct Point {
    x: f64,
    y: f64,
}

impl ::core::fmt::Debug for Point {
    fn fmt(&self, f: &mut ::core::fmt::Formatter) -> ::core::fmt::Result {
        ::core::fmt::Formatter::debug_struct_field2_finish(
            f, "Point", "x", &self.x, "y", &&self.y,
        )
    }
}

impl ::core::clone::Clone for Point {
    #[inline]
    fn clone(&self) -> Point {
        Point {
            x: ::core::clone::Clone::clone(&self.x),
            y: ::core::clone::Clone::clone(&self.y),
        }
    }
}
```

You can also expand a single module or item:

```bash
cargo expand module_name           # expand specific module
cargo expand --lib                 # expand only lib.rs
cargo expand --bin mybin           # expand specific binary
```

Where `cargo-expand` really shines is debugging proc macros. If you're using something like `serde`, `thiserror`, or `sqlx` and getting a confusing error in generated code, expanding shows you exactly what was generated. The error suddenly makes sense because you can see the actual code the compiler is complaining about.

One caveat from the [cargo-expand docs](https://github.com/dtolnay/cargo-expand): macro expansion to text is a lossy process. The expanded output is a debugging aid - don't expect it to compile or behave identically to the original. It's for reading, not for copy-pasting.

## rust-gdb and rust-lldb - real debuggers

`println!` and `dbg!` require recompilation every time you want to inspect something new. A proper debugger lets you stop execution at any point and inspect anything - locals, heap data, thread state - without modifying code.

Rust ships with `rust-gdb` and `rust-lldb`, which are wrappers around GDB and LLDB that load Rust-specific pretty-printers. Without these wrappers, a `Vec<String>` in GDB looks like a pile of raw pointers and length fields. With them, you see the actual strings.

### Getting started with rust-gdb

Build your project in debug mode (the default):

```bash
cargo build
rust-gdb target/debug/myapp
```

Essential commands:

```
(gdb) break myapp::main            # breakpoint at main
(gdb) break src/main.rs:42         # breakpoint at line 42
(gdb) run                           # start execution
(gdb) next                          # step over
(gdb) step                          # step into
(gdb) print variable_name           # inspect a variable
(gdb) print *some_ref               # dereference and print
(gdb) backtrace                     # show call stack
(gdb) continue                      # resume until next breakpoint
(gdb) info locals                   # show all local variables
```

### Conditional breakpoints

This is where debuggers become dramatically more useful than print statements. Instead of adding `if` guards to your prints, tell the debugger to only stop when a condition is true:

```
(gdb) break src/parser.rs:87 if index > 100
(gdb) break process_item if item.priority == 0
```

In a loop processing thousands of items, you can break only on the one that's causing trouble. Doing this with `println!` means either drowning in output or adding temporary filter code.

### Watchpoints

Watchpoints stop execution when a variable's value changes. If you know *what* changed but not *when* or *where*:

```
(gdb) watch my_counter
(gdb) watch -l (*some_ptr)
```

Every time `my_counter` is modified, execution stops and GDB shows you the old value, new value, and exact line that changed it. This is invaluable for tracking down logic bugs where a value ends up wrong and you can't figure out which code path modified it.

### rust-lldb on macOS

On macOS, LLDB is the native debugger. The syntax is slightly different:

```bash
rust-lldb target/debug/myapp
```

```
(lldb) breakpoint set --file main.rs --line 42
(lldb) breakpoint set --name main
(lldb) breakpoint modify --condition 'index > 100' 1
(lldb) run
(lldb) frame variable                # like gdb's 'info locals'
(lldb) expression some_var           # like gdb's 'print'
(lldb) watchpoint set variable my_counter
```

The pretty-printers work the same way - `Vec`, `String`, `HashMap` all display as readable Rust types instead of raw memory.

## IDE debugging with VS Code + CodeLLDB

If the terminal debugger workflow feels clunky, [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) brings full graphical debugging to VS Code with first-class Rust support.

### Setup

1. Install the CodeLLDB extension from the VS Code marketplace
2. Install rust-analyzer (you probably already have this)
3. Create `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug executable",
            "cargo": {
                "args": [
                    "build",
                    "--bin=myapp",
                    "--package=myapp"
                ],
                "filter": {
                    "name": "myapp",
                    "kind": "bin"
                }
            },
            "args": [],
            "cwd": "${workspaceFolder}"
        },
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug unit tests",
            "cargo": {
                "args": [
                    "test",
                    "--no-run",
                    "--lib",
                    "--package=myapp"
                ],
                "filter": {
                    "kind": "lib"
                }
            },
            "args": [],
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

CodeLLDB integrates with Cargo directly - it builds your project before launching the debugger, so you always debug the latest code.

### What you get

- **Click-to-set breakpoints** in the gutter
- **Conditional breakpoints** - right-click a breakpoint, "Edit Breakpoint", enter a condition like `i == 42`
- **Logpoints** - like `dbg!` but without modifying code. Right-click gutter, "Add Logpoint", enter `processing item {item:?}`. The debugger prints it without stopping execution.
- **Variable inspector** - hover over any variable to see its value. `Vec`, `HashMap`, `String` are shown as readable Rust types thanks to built-in visualizers.
- **Watch expressions** - add expressions to the Watch panel to track values across breakpoints
- **Call stack navigation** - click any frame in the call stack to jump to that context and inspect its locals

### Debugging tests

This is one of the biggest wins. With the "Debug unit tests" config above, you can set a breakpoint inside a test function and step through it. No more adding `println!` to tests, running them, reading output, and repeating. You see everything live.

Rust-analyzer also adds "Debug" code lenses above test functions and `main`, so you can click to start debugging without even touching `launch.json`.

### A note on optimizations

If you're debugging a release build and variables show as `<optimized out>`, the compiler has removed them. Either debug in dev mode or add this to `Cargo.toml`:

```toml
[profile.release]
opt-level = 2
debug = true
```

You can also set `opt-level = 1` for a less aggressive optimization that preserves more debug info at the cost of a slower binary.

## tracing - structured debugging for async and complex systems

When you're debugging a multi-threaded or async system, breakpoints become less practical. Setting a breakpoint in an async function might stop the executor, freezing all tasks. And `println!` output from concurrent tasks is an interleaved mess.

The [`tracing`](https://crates.io/crates/tracing) crate solves this with structured, context-aware logging. Think of it as `println!` that knows about causality - which request spawned which task, which function called which function, and what the relevant parameters were.

### Basic setup

```toml
# Cargo.toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["fmt", "env-filter"] }
```

```rust
use tracing::{info, debug, warn, error, trace};
use tracing_subscriber::EnvFilter;

fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info"))
        )
        .init();

    info!("application started");
}
```

Now you can control log levels at runtime without recompiling:

```bash
RUST_LOG=debug cargo run                          # everything at debug+
RUST_LOG=myapp=trace,hyper=warn cargo run         # per-crate control
RUST_LOG=myapp::db=debug cargo run                # per-module control
```

### #[instrument] - automatic function tracing

Here's where `tracing` becomes a debugging superpower. The `#[instrument]` attribute automatically creates a [span](https://docs.rs/tracing/latest/tracing/attr.instrument.html) for a function, recording its arguments and timing:

```rust
use tracing::instrument;

#[instrument]
fn process_order(order_id: u64, customer: &str) -> Result<Receipt, OrderError> {
    debug!("validating inventory");
    let items = fetch_items(order_id)?;

    debug!(item_count = items.len(), "fetched items");
    let total = calculate_total(&items)?;

    info!(total_cents = total, "order processed");
    Ok(Receipt { order_id, total })
}
```

The output includes the function name and arguments as structured context:

```
2026-04-01T10:30:00Z DEBUG process_order{order_id=42 customer="alice"}: myapp: validating inventory
2026-04-01T10:30:00Z DEBUG process_order{order_id=42 customer="alice"}: myapp: fetched items item_count=5
2026-04-01T10:30:01Z  INFO process_order{order_id=42 customer="alice"}: myapp: order processed total_cents=4999
```

Every log line is automatically tagged with the function and its arguments. When you're looking at logs from a production issue, you can filter by `order_id=42` and get the complete trace for that specific order across all functions.

### Controlling what gets recorded

You don't always want every argument in the span:

```rust
#[instrument(skip(password, db_pool))]
async fn login(username: &str, password: &str, db_pool: &Pool) -> Result<Token, AuthError> {
    // password and db_pool won't appear in traces
    // ...
}

#[instrument(fields(request_id = %uuid::Uuid::new_v4()))]
async fn handle_request(req: Request) -> Response {
    // adds a request_id field to the span
    // ...
}

#[instrument(level = "debug", name = "db_query")]
async fn execute_query(sql: &str) -> Result<Rows, DbError> {
    // span named "db_query" at debug level instead of default info
    // ...
}
```

### Why this beats println! for async

Consider two concurrent requests being processed. With `println!`:

```
processing order
fetching items
processing order
calculating total
fetching items
done
calculating total
done
```

Which line belongs to which request? No idea. With `tracing`:

```
INFO process_order{order_id=42}: fetching items
INFO process_order{order_id=99}: fetching items
INFO process_order{order_id=42}: calculating total
INFO process_order{order_id=42}: done
INFO process_order{order_id=99}: calculating total
INFO process_order{order_id=99}: done
```

Each line carries its context. You can grep for `order_id=42` and get a clean, sequential trace of just that request.

## Core dumps - post-mortem debugging

Sometimes a bug only happens in production, or in a CI environment you can't reproduce locally. Core dumps capture the full memory state of a process at the moment it crashes, letting you debug the crash after the fact.

### Enabling core dumps on Linux

```bash
# Allow core dumps
ulimit -c unlimited

# Set a meaningful core dump path (otherwise it might go to the current directory)
echo '/tmp/core.%e.%p' | sudo tee /proc/sys/kernel/core_pattern
```

Now run your program. When it crashes (SIGSEGV, SIGABRT, etc.), a core file appears at the configured path.

### Analyzing a core dump

```bash
rust-gdb target/debug/myapp /tmp/core.myapp.12345
```

You're dropped into a GDB session at the exact point of the crash:

```
(gdb) backtrace              # see what happened
(gdb) frame 3                # jump to an interesting frame
(gdb) info locals            # see local variables at that frame
(gdb) print some_struct      # inspect specific values
```

The combination of debug symbols and Rust's pretty-printers means you see actual `String` contents, `Vec` elements, and `Option` variants - not raw pointers.

### Making core dumps useful in practice

**Keep debug symbols in production binaries.** As mentioned earlier, set `debug = true` in your release profile. Without symbols, a core dump shows you hex addresses instead of function names.

**Save the exact binary.** A core dump is only useful with the exact binary that produced it. If you rebuild, even from the same source, the addresses shift. In CI, archive the binary alongside the core dump.

**Consider coredumpctl on systemd.** On systems running systemd, `coredumpctl` manages core dumps automatically:

```bash
coredumpctl list                    # recent crashes
coredumpctl debug myapp             # launch gdb on the latest crash
coredumpctl info                    # metadata about the crash
```

## Putting it together - workflows by bug type

Different bugs call for different tools. Here's how I approach each category.

### "It panics but I don't know where"

1. Set `RUST_BACKTRACE=1` and run again
2. The backtrace shows the exact file and line
3. If the panic is in a dependency, use `RUST_BACKTRACE=full` to see the complete chain
4. Set a breakpoint at the panic location to inspect state before the panic

This is the simplest case. The backtrace is almost always enough.

### "The value is wrong but I don't know when it goes wrong"

1. Open VS Code, set a breakpoint where the incorrect value is used
2. Run in debug mode, inspect the value
3. Set a **watchpoint** on the variable - `watch my_value` in GDB or use the Watch panel in VS Code
4. Re-run. The debugger stops every time the value changes, showing you old and new values

For loops or iterators, use **conditional breakpoints**: `break src/lib.rs:50 if counter > expected_max`.

### "The macro generates confusing errors"

1. `cargo expand` the module in question
2. Read the generated code - find the line the compiler is complaining about
3. If it's a derive macro, check whether your type satisfies all the trait bounds the macro needs
4. If it's a proc macro, check the macro's documentation for attribute options you might be missing

### "The async code behaves weirdly"

1. Add `#[instrument]` to the suspicious functions
2. Set `RUST_LOG=debug` and run
3. Look at the span context - does the function receive the arguments you expect?
4. Check ordering - are things happening in the order you assume?
5. Add `debug!()` calls with specific values inside the instrumented functions

Breakpoints in async code often freeze the entire executor, hiding timing-related bugs. `tracing` lets you observe behavior without altering it.

### "It crashes in production but works locally"

1. Get a core dump from the production environment
2. Load it with `rust-gdb target/release/myapp /path/to/core`
3. `backtrace` to see the crash point
4. `frame N` and `info locals` to inspect the state
5. If the binary was built with `debug = true` in the release profile, you get full symbol names and line numbers

If core dumps aren't available, add `tracing` with a file or network subscriber. The structured output gives you much more context than plain log lines when reconstructing what happened.

### "It works in debug but breaks in release"

This usually points to undefined behavior (in unsafe code) or optimization-sensitive logic:

1. First, reproduce with `RUST_BACKTRACE=1` in release mode
2. Build release with debug info: `debug = true` in `[profile.release]`
3. Try `opt-level = 1` instead of `opt-level = 3` to see if the crash disappears - this narrows it to optimization-related issues
4. If you have `unsafe` blocks, run under [Miri](https://github.com/rust-lang/miri): `cargo +nightly miri run` - it catches undefined behavior that only manifests under optimization
5. For non-UB cases, debug the release binary with CodeLLDB, keeping in mind that some variables will show as `<optimized out>`

## Tools I didn't cover but are worth knowing

A few more worth mentioning briefly:

- [**Miri**](https://github.com/rust-lang/miri) - an interpreter that detects undefined behavior in unsafe code. Catches things like use-after-free, out-of-bounds access, and data races. Run with `cargo +nightly miri test`.
- [**cargo-careful**](https://github.com/RalfJung/cargo-careful) - runs your code with extra standard library checks enabled. Catches some UB without the full Miri overhead.
- [**color-backtrace**](https://crates.io/crates/color-backtrace) - drop-in replacement for the default panic handler that produces syntax-highlighted backtraces with source context. Add it with two lines in `main()` and never squint at plain-text backtraces again.
- [**rr**](https://rr-project.org/) - record-and-replay debugger for Linux. Records a full execution trace that you can replay forwards and backwards. When a bug is non-deterministic, rr lets you record a failing run and replay it as many times as needed with full GDB support.

## The debugging mindset

The real upgrade isn't any single tool - it's breaking the habit of reaching for `println!` first. Before adding a print statement, ask: what am I actually trying to learn?

- "Where does it crash?" - `RUST_BACKTRACE`
- "What's this value right now?" - `dbg!` or a breakpoint
- "When does this value change?" - watchpoint
- "What code did the macro generate?" - `cargo-expand`
- "What's the execution flow across async tasks?" - `tracing` with `#[instrument]`
- "What was the state when it crashed in production?" - core dump

Each tool answers a different question. Picking the right one first saves you the recompile-run-read-repeat cycle that `println!` debugging demands.
