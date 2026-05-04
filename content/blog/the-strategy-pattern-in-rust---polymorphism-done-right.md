+++
title = "The strategy pattern in Rust - polymorphism done right"
date = 2026-03-23
description = "Enum dispatch vs dyn Trait vs generics - when to use each, what the compiler actually generates, and a practical output formatter example."

[taxonomies]
tags = ["rust", "design-patterns", "architecture", "performance"]
+++

You have a CLI tool that produces scan results. Some users want JSON. Others want a human-readable table. Security teams want [SARIF](https://sarifweb.azurewebsites.net/). The data is the same - the formatting changes. You could shove three `if` branches into one function and call it a day. But then someone asks for CSV, and the function grows. Then YAML. Then a custom format for their internal dashboard. Each format has its own edge cases, its own tests, its own dependencies. The function becomes a 400-line monster with interleaved concerns.

The strategy pattern says: extract the varying behavior into a separate type. Define the interface, implement it for each variant, and let the caller pick which one to use. In languages with class hierarchies, this means abstract classes and virtual methods. In Rust, it means traits.

<!-- more -->

## The pattern in one sentence

A strategy is a trait with one or more methods that define a behavior, and multiple types that implement it differently. The calling code depends on the trait, not on any specific implementation. You swap the implementation at construction time, and the rest of the code doesn't care.

If you've read [the adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) or [the repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/), this will feel familiar. The adapter wraps an external API behind a trait to isolate your code from dependencies. The repository wraps data access behind a trait to decouple storage from business logic. The strategy is the general case: wrap *any* varying behavior behind a trait so the caller doesn't need to know which variant is running.

Same mechanism - traits and implementations. Different intent.

## Three ways to do polymorphism in Rust

Rust gives you three distinct mechanisms for "call different code depending on the type." Each has different performance characteristics, different flexibility, and different compile-time vs runtime tradeoffs. Understanding when to use each is the core of this post.

### 1. Generics (static dispatch / monomorphization)

The compiler sees every concrete type used with the generic function and generates a specialized copy for each one. No indirection at runtime.

```rust
trait Formatter {
    fn format(&self, data: &ScanResult) -> String;
}

fn produce_output<F: Formatter>(formatter: &F, results: &[ScanResult]) -> String {
    results
        .iter()
        .map(|r| formatter.format(r))
        .collect::<Vec<_>>()
        .join("\n")
}
```

When you call `produce_output(&JsonFormatter, &results)` and `produce_output(&TableFormatter, &results)`, the compiler generates two separate functions: `produce_output::<JsonFormatter>` and `produce_output::<TableFormatter>`. Each one has the formatter's `format` method inlined directly - no vtable, no pointer chase, no indirect call.

The cost: binary size. If `produce_output` is 200 bytes of machine code and you use it with 5 formatter types, you get 1000 bytes of nearly identical code in the binary. For most applications this is irrelevant. For embedded systems or WASM targets where binary size matters, it's worth knowing.

You can see this with [`cargo-bloat`](https://github.com/nicosiatwe/cargo-bloat):

```bash
cargo bloat --release --filter produce_output
```

### 2. Trait objects (dynamic dispatch / dyn Trait)

Instead of generating a specialized function per type, you erase the concrete type behind a pointer and a vtable. The compiler generates one copy of the function, and each call goes through an indirect jump.

```rust
fn produce_output(formatter: &dyn Formatter, results: &[ScanResult]) -> String {
    results
        .iter()
        .map(|r| formatter.format(r))
        .collect::<Vec<_>>()
        .join("\n")
}
```

The `&dyn Formatter` is a fat pointer - 16 bytes on 64-bit:

```
&dyn Formatter (16 bytes on stack):
  [data_ptr: 8 bytes]   -> points to the concrete formatter value
  [vtable_ptr: 8 bytes]  -> points to a static vtable
```

The vtable is a compiler-generated static struct, one per concrete type per trait:

```
vtable for JsonFormatter as Formatter:
  [drop_in_place: 8 bytes]  -> destructor function pointer
  [size: 8 bytes]            -> size of JsonFormatter
  [align: 8 bytes]           -> alignment of JsonFormatter
  [format: 8 bytes]          -> pointer to JsonFormatter::format
```

When you call `formatter.format(data)`, the CPU loads the vtable pointer from the fat pointer, indexes into it to find the `format` function pointer, then does an indirect call. That's two memory accesses before executing any actual logic.

Key difference from C++: the vtable pointer lives in the fat pointer, not inside the object. A single `JsonFormatter` value can have different vtables depending on which trait you're using it through. This means no per-object overhead - the object itself is the same size whether you use it through `dyn Formatter` or `dyn Debug`.

When you need to own the trait object, use `Box<dyn Formatter>`:

```rust
struct OutputPipeline {
    formatter: Box<dyn Formatter>,
}

impl OutputPipeline {
    fn new(formatter: Box<dyn Formatter>) -> Self {
        Self { formatter }
    }

    fn run(&self, results: &[ScanResult]) -> String {
        produce_output(self.formatter.as_ref(), results)
    }
}
```

`Box<dyn Formatter>` is also 16 bytes (two pointers), but it owns the heap allocation. When the box is dropped, it calls the destructor through the vtable's `drop_in_place` and frees the memory.

### 3. Enum dispatch

No traits involved. You define an enum with one variant per strategy, and implement the behavior with a `match`:

```rust
enum OutputFormat {
    Json,
    Table { max_width: usize },
    Sarif { tool_name: String, tool_version: String },
}

impl OutputFormat {
    fn format(&self, data: &ScanResult) -> String {
        match self {
            OutputFormat::Json => serde_json::to_string_pretty(data).unwrap_or_default(),
            OutputFormat::Table { max_width } => format_table(data, *max_width),
            OutputFormat::Sarif { tool_name, tool_version } => {
                format_sarif(data, tool_name, tool_version)
            }
        }
    }
}
```

No vtable, no indirection, no heap allocation. The enum lives on the stack (or wherever you put it). The `match` compiles to a jump table or a series of conditional branches - either way, the CPU knows all possible targets at compile time, which means branch prediction works well.

The size of the enum equals the size of its largest variant plus a discriminant (usually 1 byte for up to 256 variants, but the compiler picks the smallest integer that fits). You can check with:

```rust
use std::mem;
println!("OutputFormat: {} bytes", mem::size_of::<OutputFormat>()); 
```

If one variant holds a `String` (24 bytes) and another holds just a `bool` (1 byte), every instance of the enum uses at least 25 bytes. The small variant wastes space. This is the tradeoff for avoiding heap allocation.

## Performance: how much does dispatch actually cost?

The [`enum_dispatch`](https://crates.io/crates/enum_dispatch) crate benchmarks are widely cited. Their numbers show enum dispatch running roughly 4-10x faster than `dyn Trait` in tight loops with millions of iterations. That sounds dramatic, but context matters.

The overhead of dynamic dispatch is two things: the indirect call itself (a few nanoseconds for the pointer chase) and the lost inlining opportunity. If the trait method does trivial work - incrementing a counter, returning a constant - the call overhead dominates. If the method does real work - formatting a struct to JSON, querying a database, writing to a file - the dispatch cost disappears into noise.

For output formatters that serialize data to strings? The `serde_json::to_string` call inside `JsonFormatter::format` takes microseconds. The 2-5 nanosecond vtable lookup is invisible.

My rule: if the function body takes less than ~50 nanoseconds, dispatch overhead *might* matter. Profile to confirm. If the function does I/O, serialization, or allocation - any form of real work - use whichever dispatch mechanism makes your code clearest.

For the rare case where dispatch is actually on the hot path (inner loops of a game engine, audio processing callbacks, tight numeric loops), the [`enum_dispatch`](https://crates.io/crates/enum_dispatch) crate generates the enum-based dispatch from trait definitions automatically:

```rust
use enum_dispatch::enum_dispatch;

#[enum_dispatch]
trait Formatter {
    fn format(&self, data: &ScanResult) -> String;
}

#[enum_dispatch(Formatter)]
enum OutputFormat {
    Json(JsonFormatter),
    Table(TableFormatter),
    Sarif(SarifFormatter),
}
```

This generates the `match`-based dispatch you'd write by hand, but you keep the trait-based API. Best of both worlds when you need it.

## The decision matrix

| Criterion | Generics | dyn Trait | Enum |
|---|---|---|---|
| Known at compile time? | Yes | No (runtime choice) | Yes |
| Extensible by downstream? | Yes | Yes | No |
| Heterogeneous collection? | No | Yes (`Vec<Box<dyn T>>`) | Yes (`Vec<Enum>`) |
| Call overhead | Zero | Vtable lookup | Branch/jump table |
| Binary size | Grows with types | One copy | One copy |
| Object safe required? | No | Yes | N/A |

The extensibility row is the one people miss. Enum dispatch is a closed set - adding a new variant means modifying the enum and every `match` that touches it. Trait-based dispatch (both generic and `dyn`) is an open set - anyone can implement the trait for a new type without touching existing code. If you're writing a library and users should be able to add their own formatters, enums won't work.

## When to use which

**Use generics** when the type is known at compile time and performance matters. The compiler monomorphizes everything, inlines aggressively, and generates optimal code. This is the default choice for library code. If you're already familiar with trait bounds from [the trait bounds post](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg/), you know how to express complex constraints cleanly with `where` clauses.

**Use `dyn Trait`** when you need runtime flexibility. Config-driven behavior, plugin systems, heterogeneous collections, dependency injection. The repository pattern from [my earlier post](/blog/the-repository-pattern-abstracting-data-access-in-rust/) is a perfect example: `Arc<dyn Repository<User>>` lets you swap InMemory for SQLite at runtime. The vtable cost is negligible for anything that does real work.

**Use enum dispatch** when the set of variants is small, closed, and known upfront. Parsing tokens (`Token::Ident`, `Token::Number`, `Token::String`), state machines (`State::Idle`, `State::Running`, `State::Done`), message types in a protocol. The compiler can optimize matches into jump tables and the data lives on the stack.

## Practical example: output formatters

Back to the opening scenario. You have scan results and need multiple output formats. Here's how I'd actually build it with `dyn Trait`:

```rust
use std::io::Write;

/// A single finding from a scan
#[derive(Debug, Clone, serde::Serialize)]
pub struct Finding {
    pub rule_id: String,
    pub severity: Severity,
    pub message: String,
    pub file: String,
    pub line: u32,
}

#[derive(Debug, Clone, serde::Serialize)]
pub enum Severity {
    Error,
    Warning,
    Info,
}

/// The strategy trait
pub trait OutputFormatter: Send + Sync {
    /// Format findings and write to the given writer.
    fn write_findings(&self, findings: &[Finding], writer: &mut dyn Write) -> std::io::Result<()>;

    /// File extension hint (used for --output flag)
    fn extension(&self) -> &str;
}
```

The trait takes a `&mut dyn Write` instead of returning a `String`. This avoids allocating the entire output in memory - you can stream directly to stdout or a file. Both the formatter and the writer use dynamic dispatch, which is fine: writing to a file involves syscalls that take microseconds. Two vtable lookups are nothing.

### JSON formatter

```rust
pub struct JsonFormatter {
    pub pretty: bool,
}

impl OutputFormatter for JsonFormatter {
    fn write_findings(&self, findings: &[Finding], writer: &mut dyn Write) -> std::io::Result<()> {
        let output = if self.pretty {
            serde_json::to_string_pretty(findings)
        } else {
            serde_json::to_string(findings)
        };

        let json = output.map_err(|e| std::io::Error::new(std::io::ErrorKind::Other, e))?;
        writer.write_all(json.as_bytes())?;
        writer.write_all(b"\n")
    }

    fn extension(&self) -> &str {
        "json"
    }
}
```

### Table formatter

```rust
pub struct TableFormatter {
    pub max_width: usize,
    pub show_line_numbers: bool,
}

impl OutputFormatter for TableFormatter {
    fn write_findings(&self, findings: &[Finding], writer: &mut dyn Write) -> std::io::Result<()> {
        // Header
        if self.show_line_numbers {
            writeln!(writer, "{:<12} {:<8} {:<30} {}:{}", 
                "RULE", "SEV", "MESSAGE", "FILE", "LINE")?;
        } else {
            writeln!(writer, "{:<12} {:<8} {:<30} {}", 
                "RULE", "SEV", "MESSAGE", "FILE")?;
        }

        writeln!(writer, "{}", "-".repeat(self.max_width.min(120)))?;

        for f in findings {
            let severity = match f.severity {
                Severity::Error => "ERROR",
                Severity::Warning => "WARN",
                Severity::Info => "INFO",
            };

            let msg = if f.message.len() > 30 {
                format!("{}...", &f.message[..27])
            } else {
                f.message.clone()
            };

            if self.show_line_numbers {
                writeln!(writer, "{:<12} {:<8} {:<30} {}:{}", 
                    f.rule_id, severity, msg, f.file, f.line)?;
            } else {
                writeln!(writer, "{:<12} {:<8} {:<30} {}", 
                    f.rule_id, severity, msg, f.file)?;
            }
        }

        writeln!(writer, "\n{} finding(s)", findings.len())
    }

    fn extension(&self) -> &str {
        "txt"
    }
}
```

### SARIF formatter

[SARIF](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html) (Static Analysis Results Interchange Format) is a JSON-based standard for static analysis tools. GitHub, Azure DevOps, and most CI platforms understand it natively. The format is verbose but well-specified:

```rust
pub struct SarifFormatter {
    pub tool_name: String,
    pub tool_version: String,
}

impl OutputFormatter for SarifFormatter {
    fn write_findings(&self, findings: &[Finding], writer: &mut dyn Write) -> std::io::Result<()> {
        let results: Vec<serde_json::Value> = findings
            .iter()
            .map(|f| {
                serde_json::json!({
                    "ruleId": f.rule_id,
                    "level": match f.severity {
                        Severity::Error => "error",
                        Severity::Warning => "warning",
                        Severity::Info => "note",
                    },
                    "message": { "text": f.message },
                    "locations": [{
                        "physicalLocation": {
                            "artifactLocation": { "uri": f.file },
                            "region": { "startLine": f.line }
                        }
                    }]
                })
            })
            .collect();

        let sarif = serde_json::json!({
            "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json",
            "version": "2.1.0",
            "runs": [{
                "tool": {
                    "driver": {
                        "name": self.tool_name,
                        "version": self.tool_version,
                    }
                },
                "results": results,
            }]
        });

        let output = serde_json::to_string_pretty(&sarif)
            .map_err(|e| std::io::Error::new(std::io::ErrorKind::Other, e))?;
        writer.write_all(output.as_bytes())?;
        writer.write_all(b"\n")
    }

    fn extension(&self) -> &str {
        "sarif"
    }
}
```

### Wiring it together

The strategy selection happens once, at startup. After that, everything works through the trait:

```rust
fn build_formatter(format: &str) -> Box<dyn OutputFormatter> {
    match format {
        "json" => Box::new(JsonFormatter { pretty: true }),
        "json-compact" => Box::new(JsonFormatter { pretty: false }),
        "table" => Box::new(TableFormatter {
            max_width: 120,
            show_line_numbers: true,
        }),
        "sarif" => Box::new(SarifFormatter {
            tool_name: "my-scanner".into(),
            tool_version: env!("CARGO_PKG_VERSION").into(),
        }),
        other => {
            eprintln!("Unknown format '{}', falling back to table", other);
            Box::new(TableFormatter {
                max_width: 120,
                show_line_numbers: true,
            })
        }
    }
}

fn main() -> std::io::Result<()> {
    let format = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "table".into());

    let formatter = build_formatter(&format);
    let findings = run_scan(); // your scan logic

    // Output to stdout
    let mut stdout = std::io::stdout().lock();
    formatter.write_findings(&findings, &mut stdout)?;

    // Or to a file
    let filename = format!("results.{}", formatter.extension());
    let mut file = std::fs::File::create(&filename)?;
    formatter.write_findings(&findings, &mut file)?;

    Ok(())
}
```

Notice what `main` doesn't know: which formatter is running, how findings are serialized, what the output looks like. It calls `write_findings` and moves on. Adding a CSV formatter means writing a new struct that implements `OutputFormatter` and adding one arm to `build_formatter`. Zero changes to the rest of the codebase.

## Composition over inheritance

OOP languages model strategy through class hierarchies. You'd have an abstract `Formatter` class, concrete subclasses, maybe a factory pattern to pick the right one. The hierarchy depth can get silly - `AbstractBaseFormatter -> StreamFormatter -> TextStreamFormatter -> PaddedTextStreamFormatter`.

Rust doesn't have inheritance. And that's a feature, not a limitation. Instead of building tall class trees, you compose small traits.

Say your formatters need colorized output for terminal display:

```rust
pub trait Colorizer: Send + Sync {
    fn colorize(&self, text: &str, severity: &Severity) -> String;
}

pub struct AnsiColorizer;

impl Colorizer for AnsiColorizer {
    fn colorize(&self, text: &str, severity: &Severity) -> String {
        match severity {
            Severity::Error => format!("\x1b[31m{}\x1b[0m", text),   // red
            Severity::Warning => format!("\x1b[33m{}\x1b[0m", text), // yellow
            Severity::Info => format!("\x1b[36m{}\x1b[0m", text),    // cyan
        }
    }
}

pub struct NoColor;

impl Colorizer for NoColor {
    fn colorize(&self, text: &str, _severity: &Severity) -> String {
        text.to_string()
    }
}
```

Now compose it into the table formatter:

```rust
pub struct TableFormatter {
    pub max_width: usize,
    pub show_line_numbers: bool,
    pub colorizer: Box<dyn Colorizer>,
}

impl TableFormatter {
    pub fn with_color() -> Self {
        Self {
            max_width: 120,
            show_line_numbers: true,
            colorizer: Box::new(AnsiColorizer),
        }
    }

    pub fn without_color() -> Self {
        Self {
            max_width: 120,
            show_line_numbers: true,
            colorizer: Box::new(NoColor),
        }
    }
}
```

The `TableFormatter` doesn't inherit from anything. It holds a `Colorizer` strategy and delegates to it. You can mix and match: `TableFormatter` with `AnsiColorizer`, `TableFormatter` with `NoColor`, or `TableFormatter` with a hypothetical `Html256Colorizer`. Each combination is flat - no hierarchy, no diamond inheritance problem, no fragile base class.

This is exactly how the Rust ecosystem works. Tower's [`Service`](https://docs.rs/tower/latest/tower/trait.Service.html) trait composes middleware by wrapping services in other services. `tracing`'s [`Layer`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/trait.Layer.html) trait composes logging strategies by stacking layers. `std::io::BufWriter<W: Write>` composes buffering with any writer. No inheritance anywhere.

## The standard library is full of strategies

You use the strategy pattern every time you call a generic function in `std`. A few you probably don't think of as "strategies":

**`std::io::Write`** - any output destination is a writing strategy. `File`, `TcpStream`, `Vec<u8>`, `Stdout` - all implement `Write`. `BufWriter<W: Write>` accepts any strategy via generics (static dispatch).

**`HashMap<K, V, S: BuildHasher>`** - the third type parameter is a hashing strategy. Default is `RandomState` (SipHash). Swap to [`ahash`](https://crates.io/crates/ahash) for faster hashing, [`fxhash`](https://crates.io/crates/rustc-hash) for even faster (but non-cryptographic) hashing. The map doesn't care - it calls `S::build_hasher()` and uses whatever hasher comes back.

**`Serializer` / `Deserializer` in serde** - the entire serde architecture is the strategy pattern. `Serialize` defines what data a type has. `Serializer` defines how to write it. `serde_json::Serializer`, `serde_yaml::Serializer`, `toml::Serializer` - each is a serialization strategy. You swap the strategy to change the output format. Same data, different behavior.

**`GlobalAlloc`** - the memory allocator itself is a strategy. `#[global_allocator]` lets you swap the default allocator for [`jemalloc`](https://crates.io/crates/tikv-jemallocator), [`mimalloc`](https://crates.io/crates/mimalloc), or a custom allocator. Every `Box::new`, every `Vec::push`, every allocation in the program goes through your chosen strategy.

## Trait upcasting and the strategy pattern

Since Rust 1.86, you can coerce `&dyn SubTrait` to `&dyn SuperTrait` directly. This matters for strategy patterns with trait hierarchies:

```rust
trait Formatter: std::fmt::Debug {
    fn format(&self, data: &ScanResult) -> String;
}

fn log_formatter(f: &dyn Formatter) {
    // Before Rust 1.86, you couldn't cast &dyn Formatter to &dyn Debug.
    // Now it just works:
    let debuggable: &dyn std::fmt::Debug = f;
    println!("Using formatter: {:?}", debuggable);
}
```

This simplifies strategy patterns where you want to log, compare, or inspect strategies through their supertrait bounds. Before 1.86, you needed workarounds like adding an `as_debug` method to the trait.

## When NOT to use strategy

The strategy pattern has a cost: indirection. Not runtime indirection - cognitive indirection. Every trait boundary means the reader has to find the implementation to understand what's actually happening. If you have a `Formatter` trait with exactly one implementation and no plans for a second one, the trait is ceremony.

**YAGNI (You Aren't Gonna Need It) applies.** Don't add a trait boundary "in case we need to swap it later." If you have one formatter, write a function:

```rust
fn format_as_json(findings: &[Finding]) -> String {
    serde_json::to_string_pretty(findings).unwrap_or_default()
}
```

When - *if* - you need a second format, extract the trait. Rust makes refactoring cheap: add a trait, move the function body into an `impl`, update callers from `format_as_json(data)` to `formatter.format(data)`. The compiler catches every call site you miss. This refactoring takes minutes, not days.

Signs you actually need the strategy pattern:

- **Multiple implementations exist today.** Not "might exist someday." Today.
- **The implementation is chosen at runtime.** Config flags, CLI arguments, feature toggles.
- **Tests need a different implementation.** Mock formatter that captures output for assertions.
- **Users of your library need to provide their own behavior.** You publish a trait, they implement it.

Signs you don't:

- **One implementation, no tests swapping it.** Just use a concrete type.
- **The "strategies" differ by one boolean.** Use a parameter, not a trait.
- **The behavior is static and known at compile time.** A function or a const might be enough.

## Closures as lightweight strategies

For simple, single-method strategies, closures avoid the ceremony of defining a trait and implementation types entirely:

```rust
fn process_findings<F>(findings: &[Finding], format: F) -> String
where
    F: Fn(&Finding) -> String,
{
    findings.iter().map(|f| format(f)).collect::<Vec<_>>().join("\n")
}

// Usage - no struct, no impl, no trait
let output = process_findings(&findings, |f| {
    format!("{}: {} ({}:{})", f.rule_id, f.message, f.file, f.line)
});
```

The [Rust Design Patterns](https://rust-unofficial.github.io/patterns/patterns/behavioural/strategy.html) book makes this point explicitly: in many cases, a closure *is* the strategy. You don't need the full trait machinery when a `Fn` bound does the job.

The line between "use a closure" and "use a trait" is roughly: does the strategy have state? Does it have multiple methods? Does it need a name for documentation or error messages? If yes to any, use a trait. If it's a single stateless transformation, a closure is cleaner.

## Putting it all together

The strategy pattern is just polymorphism with intent. You're not doing polymorphism because OOP theory says so - you're doing it because you have concrete, varying behavior that needs to be swappable. Rust gives you three dispatch mechanisms, and the right choice depends on your constraints:

Generics for library code and zero-cost abstraction. `dyn Trait` for runtime flexibility and dependency injection. Enum dispatch for closed, small, performance-critical variant sets. Closures for trivial single-method cases.

Start with the simplest option that works. A function. A closure. An enum. If you find yourself needing extensibility or runtime selection, extract a trait. The compiler will guide you through the refactoring - that's Rust's real advantage over languages where the wrong abstraction choice is expensive to undo.
