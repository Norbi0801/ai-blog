+++
title = "The inventory pattern in Rust - compile-time plugin registration"
date = 2026-03-09
description = "How the inventory and linkme crates use linker tricks to build decentralized plugin registries, and when this pattern beats manual wiring."

[taxonomies]
tags = ["rust", "design-patterns", "compiler-internals", "architecture"]
+++

You're building a system with handlers. Maybe it's a CLI with subcommands, a web framework with route handlers, or a test harness with discovery. Every time you add a new handler, you also have to go update some central `Vec` or `match` arm. Forget one, and it silently doesn't exist.

This is the registration problem. And Rust has an interesting solution hiding in the linker.

<!-- more -->

## The manual approach and its pain

The straightforward way to build a handler registry looks like this:

```rust
type HandlerFn = fn(&str) -> String;

struct Handler {
    name: &'static str,
    func: HandlerFn,
}

fn all_handlers() -> Vec<Handler> {
    vec![
        Handler { name: "greet", func: handle_greet },
        Handler { name: "status", func: handle_status },
        Handler { name: "version", func: handle_version },
        // every new handler: add a line here
    ]
}
```

This works. It's explicit. But it has a scaling problem: every new handler requires editing `all_handlers()`. In a project with 50 handlers across 20 modules, that central function becomes a merge conflict magnet and a source of "why isn't my handler showing up" bugs.

You could reach for trait objects. If you've read my [post on trait objects vs enums vs generics](/blog/trait-objects-vs-enums-vs-generics-when-to-use-which-in-rust), you know the trade-offs there. A `Vec<Box<dyn Handler>>` gives you extensibility, but you still need something to populate that Vec. The dispatch mechanism is separate from the discovery mechanism.

What if handlers could register themselves just by existing in the binary?

## inventory: self-registering plugins

The [`inventory`](https://crates.io/crates/inventory) crate by David Tolnay solves exactly this. Three macros, one pattern:

```rust
use inventory;

// 1. Define your plugin type
pub struct Command {
    pub name: &'static str,
    pub description: &'static str,
    pub run: fn(&[String]),
}

// 2. Tell inventory about it
inventory::collect!(Command);
```

Now any module - in your crate or any downstream dependency - can register a command:

```rust
// src/commands/greet.rs
fn run_greet(args: &[String]) {
    let name = args.first().map(|s| s.as_str()).unwrap_or("world");
    println!("Hello, {}!", name);
}

inventory::submit! {
    Command {
        name: "greet",
        description: "Say hello",
        run: run_greet,
    }
}
```

```rust
// src/commands/version.rs
inventory::submit! {
    Command {
        name: "version",
        description: "Print version",
        run: |_args| println!("v{}", env!("CARGO_PKG_VERSION")),
    }
}
```

And in main, you iterate over everything that was registered:

```rust
fn main() {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let cmd_name = args.first().map(|s| s.as_str()).unwrap_or("help");

    for command in inventory::iter::<Command> {
        if command.name == cmd_name {
            (command.run)(&args[1..]);
            return;
        }
    }

    println!("Available commands:");
    for command in inventory::iter::<Command> {
        println!("  {:12} {}", command.name, command.description);
    }
}
```

No central list. No manual wiring. Add a file with `inventory::submit!`, and it shows up. Delete the file, and it's gone.

## What actually happens: linker sections and life-before-main

This isn't magic. It's linker exploitation. Let me walk through what the compiler and linker actually produce.

When you write `inventory::submit!`, the macro expands to something like this (simplified):

```rust
const _: () = {
    // Your value, stored as a static
    static VALUE: Command = Command {
        name: "greet",
        description: "Say hello",
        run: run_greet,
    };

    // A node in a linked list
    static NODE: inventory::Node = inventory::Node {
        value: &VALUE,
        next: UnsafeCell::new(None),
    };

    // A constructor function
    #[link_section = ".init_array"]
    static CTOR: unsafe extern "C" fn() = {
        unsafe extern "C" fn __ctor() {
            // Atomically prepend NODE to the registry's linked list
            inventory::ErasedNode::submit(NODE.value, &NODE);
        }
        __ctor
    };
};
```

Three things happen here:

**1. The value is a static.** Your `Command` struct lives in the binary's `.rodata` or `.data` section. It exists for the entire program lifetime - that's why `inventory::iter` yields `&'static T`.

**2. The node wraps it in a linked list element.** The `Registry` is a lock-free singly-linked list. Each `Node` has a value pointer and a `next` pointer. The registry head is an `AtomicPtr<Node>`.

**3. The constructor goes into a linker section.** This is where it gets platform-specific:

| Platform | Section | Mechanism |
|----------|---------|-----------|
| Linux/Android | `.init_array` | ELF init array |
| macOS/iOS | `__DATA,__mod_init_func` | Mach-O module initializers |
| Windows | `.CRT$XCU` | CRT startup table |
| WebAssembly | `__wasm_call_ctors` | WASM constructors |

On Linux, the ELF binary format has a section called `.init_array`. The dynamic linker (`ld-linux.so`) iterates every function pointer in this section and calls them before `main()` runs. This is the same mechanism C/C++ uses for `__attribute__((constructor))` and global object constructors.

The `submit` function itself uses compare-and-swap to prepend to the linked list:

```rust
unsafe fn submit(&'static self, new: &'static Node) {
    let mut head = self.head.load(Ordering::Relaxed);
    loop {
        *new.next.get() = head.as_ref();
        match self.head.compare_exchange(
            head,
            ptr::addr_of!(*new).cast_mut(),
            Ordering::Release,
            Ordering::Relaxed,
        ) {
            Ok(_) => return,
            Err(prev) => head = prev,
        }
    }
}
```

This is a classic lock-free push. The `Ordering::Release` on success ensures the node's `next` pointer is visible to any thread that later loads the head with `Acquire`. The CAS loop handles the (unlikely, during init) case where two constructors race.

By the time `main()` starts, every `submit!` in every linked object file has run, and the registry is fully populated.

## You can see it in the binary

If you're curious, you can verify this with `objdump`. Compile a binary using inventory and inspect the init array:

```bash
$ cargo build --release
$ objdump -s -j .init_array target/release/my_app
```

You'll see function pointers in the `.init_array` section - one for each `inventory::submit!` call. Each points to a small shim that calls into the registry's atomic linked list.

You can also check it with `readelf`:

```bash
$ readelf -S target/release/my_app | grep init
  [17] .init_array       INIT_ARRAY      0000000000045a00  00045a00
```

The section exists, the runtime calls it, and your plugins appear. No runtime reflection, no proc macro code generation of a central list - just the linker doing what linkers do.

## linkme: the zero-runtime alternative

The [`linkme`](https://crates.io/crates/linkme) crate, also by David Tolnay, takes a different approach. Instead of life-before-main constructors, it uses linker sections directly to create a contiguous slice:

```rust
use linkme::distributed_slice;

#[distributed_slice]
pub static COMMANDS: [Command];

#[distributed_slice(COMMANDS)]
static GREET: Command = Command {
    name: "greet",
    description: "Say hello",
    run: run_greet,
};

#[distributed_slice(COMMANDS)]
static VERSION: Command = Command {
    name: "version",
    description: "Print version",
    run: |_args| println!("v{}", env!("CARGO_PKG_VERSION")),
};

fn main() {
    // COMMANDS is a &'static [Command]
    println!("{} commands registered", COMMANDS.len());
    for cmd in COMMANDS {
        println!("  {}", cmd.name);
    }
}
```

The difference is fundamental. linkme doesn't run any code before main. Instead, it places each element in a named linker section, and uses platform-specific tricks to make the linker concatenate them into a contiguous block of memory. Boundary symbols mark the start and end, and at runtime you just read that memory as a slice.

No atomic operations. No linked list. No constructor functions. The "registration" is purely an artifact of how the linker laid out the binary. This is about as zero-cost as it gets.

### inventory vs linkme

| | inventory | linkme |
|--|-----------|--------|
| **Mechanism** | ctor + atomic linked list | Linker sections only |
| **Runtime cost** | Constructor per submission | Zero |
| **Dynamic libraries** | Yes (dlopen triggers registration) | No |
| **Data structure** | Linked list (unordered) | Contiguous slice |
| **API** | `submit!{}` + `iter::<T>` | `#[distributed_slice]` |
| **Random access** | No (iterator only) | Yes (it's a slice) |
| **len()** | Need to count via iterator | O(1) |

Pick linkme when you want zero overhead and don't need dynamic library support. Pick inventory when you need dlopen compatibility or when the constructor model fits your mental model better.

## A more realistic example: handler registry with metadata

Let me build something closer to what you'd use in production - an HTTP-style handler registry where each handler declares its method, path pattern, and function:

```rust
use inventory;

#[derive(Debug)]
pub enum Method {
    Get,
    Post,
    Put,
    Delete,
}

pub struct Route {
    pub method: Method,
    pub path: &'static str,
    pub handler: fn(&Request) -> Response,
    pub description: &'static str,
}

// Placeholder types for the example
pub struct Request {
    pub path: String,
    pub body: String,
}

pub struct Response {
    pub status: u16,
    pub body: String,
}

inventory::collect!(Route);
```

Now handlers register themselves across your codebase:

```rust
// src/handlers/health.rs
use crate::{Method, Request, Response, Route};

fn health_check(_req: &Request) -> Response {
    Response {
        status: 200,
        body: r#"{"status":"ok"}"#.to_string(),
    }
}

inventory::submit! {
    Route {
        method: Method::Get,
        path: "/health",
        handler: health_check,
        description: "Health check endpoint",
    }
}
```

```rust
// src/handlers/users.rs
use crate::{Method, Request, Response, Route};

fn list_users(_req: &Request) -> Response {
    Response {
        status: 200,
        body: r#"[{"id":1,"name":"Alice"}]"#.to_string(),
    }
}

fn create_user(req: &Request) -> Response {
    Response {
        status: 201,
        body: format!(r#"{{"created":true,"body":"{}"}}"#, req.body),
    }
}

inventory::submit! {
    Route {
        method: Method::Get,
        path: "/users",
        handler: list_users,
        description: "List all users",
    }
}

inventory::submit! {
    Route {
        method: Method::Post,
        path: "/users",
        handler: create_user,
        description: "Create a user",
    }
}
```

A simple router that uses the registry:

```rust
fn dispatch(req: &Request) -> Response {
    for route in inventory::iter::<Route> {
        if req.path == route.path {
            return (route.handler)(req);
        }
    }
    Response {
        status: 404,
        body: r#"{"error":"not found"}"#.to_string(),
    }
}

fn print_route_table() {
    println!("Registered routes:");
    for route in inventory::iter::<Route> {
        println!("  {:6?} {:20} {}", route.method, route.path, route.description);
    }
}
```

Adding a new route means adding a new file with `inventory::submit!`. The router doesn't change. The route table doesn't change. The module declarations in `mod.rs` are the only wiring you need, and that's just so rustc compiles the file.

## Comparison with other dispatch patterns

How does this stack up against the alternatives I covered in [trait objects vs enums vs generics](/blog/trait-objects-vs-enums-vs-generics-when-to-use-which-in-rust)?

**Enum dispatch** gives you exhaustive matching and zero-cost dispatch, but every new variant requires editing the enum definition and every match arm. It's the opposite of the open/closed principle. Great for a fixed set of 5 things, terrible for a growing set of 50.

**Trait objects** (`Vec<Box<dyn Handler>>`) give you extensibility, but someone still has to build the Vec. You typically end up with a `register()` function that each module calls, and a startup function that calls all the register functions. It's the same central-list problem, just moved one level up.

**inventory/linkme** solve both: open for extension (any crate can submit), and no central list. The cost is that you give up exhaustive matching (you can't know at compile time what's registered) and ordering guarantees.

| Approach | Open for extension | Central list needed | Exhaustive | Runtime cost |
|----------|-------------------|--------------------|-----------:|-------------|
| Enum dispatch | No | Yes (enum def) | Yes | Zero |
| Trait objects | Yes | Yes (registration) | No | vtable indirection |
| inventory | Yes | No | No | Constructor + linked list walk |
| linkme | Yes | No | No | Slice iteration |

## Who uses this in practice?

The inventory crate has been around since 2019 and sees steady use. Some notable patterns:

- **[tracing](https://crates.io/crates/tracing)** uses a similar mechanism internally for subscriber registration
- **[Bevy](https://bevyengine.org/)** uses type registration patterns for its ECS reflection system
- **Test frameworks** like [rstest](https://crates.io/crates/rstest) and custom harnesses use distributed registration for test discovery
- **CLI tools** with plugin architectures where each plugin self-registers its commands

Rust's own `#[test]` attribute works on a similar principle. The compiler collects all `#[test]` functions and generates a test harness that calls them. It just does it in the compiler rather than through linker sections.

The [Global Registration pre-RFC](https://internals.rust-lang.org/t/global-registration-a-kind-of-pre-rfc/20813) on the Rust internals forum has been discussing making this a first-class language feature. The key tension: should registries be crate-local or truly global? How do you handle multiple versions of the same crate? What about const evaluation? These are open questions.

## Gotchas and limitations

**Ordering is not guaranteed.** Both inventory and linkme make no promises about the order elements appear. If you need ordering, add a priority field and sort after collection:

```rust
pub struct Route {
    pub priority: u32,
    pub method: Method,
    pub path: &'static str,
    pub handler: fn(&Request) -> Response,
    pub description: &'static str,
}

fn sorted_routes() -> Vec<&'static Route> {
    let mut routes: Vec<_> = inventory::iter::<Route>.into_iter().collect();
    routes.sort_by_key(|r| r.priority);
    routes
}
```

**Values must be `Sync + 'static`.** Everything in an inventory lives for the entire program. You can't register values that borrow local data. The `Collect` trait requires `Sync + Sized + 'static`.

**Life-before-main is inherently tricky.** When inventory constructors run, the standard library isn't fully initialized. You can't allocate, panic, or use stdout in the expression passed to `submit!`. Stick to const-constructible values.

**Link-time dead code elimination can bite.** If a library crate has `inventory::submit!` calls but the binary doesn't reference anything from that library, the linker might discard the whole object file - including the constructor. You may need `#[used]` or explicit references to prevent this.

**No compile-time access.** You can't use the registry in const contexts. The set of registered items is only known after linking, which is after compilation. If you need compile-time plugin lists, you need proc macros or build scripts.

## When to reach for this pattern

Use inventory or linkme when:

- You have a growing set of similar items spread across modules or crates
- Each item is self-contained and doesn't depend on registration order
- You want to add/remove items without touching a central file
- You're building a plugin system, test framework, or command registry

Don't use it when:

- You have a small, fixed set of variants (use an enum)
- You need compile-time exhaustiveness checking
- You need guaranteed ordering without a sort step
- You're uncomfortable with linker-level behavior that's hard to debug

The inventory pattern sits in a specific niche: large-scale registration where the convenience of decentralization outweighs the loss of compiler-verified completeness. It's not something you reach for every day. But when you need it - when you're staring at a 200-line function that just builds a Vec of handlers - it's a clean solution built on the same linker mechanics that make C++ global constructors work.

The linker has been solving this problem since the 1970s. inventory and linkme just give you a safe Rust API for it.
