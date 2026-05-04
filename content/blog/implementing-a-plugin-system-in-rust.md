+++
title = "Implementing a plugin system in Rust"
date = 2025-02-12
description = "Four approaches to extensible Rust applications - trait objects, compile-time registration, dynamic loading, and WASM sandboxing - with trade-offs, code, and real-world examples."

[taxonomies]
tags = ["rust", "architecture", "design-patterns", "plugins"]
+++

You ship a binary. Users want to extend it. Maybe they want custom output formats, new authentication backends, domain-specific data transformers, or integrations with systems you've never heard of. You could add every feature yourself, but that doesn't scale. At some point you need a plugin system - a way for external code to hook into your application without modifying or recompiling the host.

This is a solved problem in languages with dynamic class loading (Java) or eval (Python, JavaScript). In Rust, it's more nuanced. There's no classloader, no runtime reflection, no eval. But there are four distinct approaches, each with different isolation guarantees, performance profiles, and development ergonomics. This post builds a working plugin system with each one.

<!-- more -->

## The Plugin trait

Every approach starts from the same place: a trait that defines what a plugin can do. This is the contract between your host application and any plugin that wants to participate.

```rust
pub trait Plugin: Send + Sync {
    /// A unique identifier for this plugin.
    fn name(&self) -> &str;

    /// Called once when the plugin is loaded.
    fn init(&mut self) -> Result<(), PluginError>;

    /// Called on each event the host dispatches.
    fn on_event(&self, event: &Event) -> Result<(), PluginError>;

    /// Called before the host shuts down.
    fn shutdown(&self) -> Result<(), PluginError> {
        Ok(()) // default: no-op
    }
}
```

If you've read the [strategy pattern post](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/), this shape is familiar - a trait with multiple implementations, and the host works against the trait without caring which concrete type is behind it. The difference here is lifecycle. A strategy is typically stateless and swappable per-call. A plugin is loaded once, initialized, receives events over time, and eventually shuts down. It has identity (a name) and state.

The `Send + Sync` bounds matter. Most host applications are async and multi-threaded. Without these bounds, you can't store plugins in an `Arc<Vec<Box<dyn Plugin>>>` or dispatch events from a tokio task. If a plugin needs interior mutability, it uses `Mutex` or `RwLock` internally - the host doesn't care.

The shared types look like this:

```rust
#[derive(Debug, Clone)]
pub struct Event {
    pub kind: String,
    pub payload: serde_json::Value,
}

#[derive(Debug, thiserror::Error)]
pub enum PluginError {
    #[error("init failed: {0}")]
    InitFailed(String),
    #[error("event handling failed: {0}")]
    EventFailed(String),
    #[error("{0}")]
    Other(String),
}
```

That's the foundation. Now, four ways to register and load things that implement it.

## Approach 1: Static registration with `Vec<Box<dyn Plugin>>`

The simplest plugin system is a list of trait objects built in `main`:

```rust
struct PluginHost {
    plugins: Vec<Box<dyn Plugin>>,
}

impl PluginHost {
    fn new() -> Self {
        Self { plugins: Vec::new() }
    }

    fn register(&mut self, plugin: Box<dyn Plugin>) {
        self.plugins.push(plugin);
    }

    fn init_all(&mut self) -> Result<(), PluginError> {
        for plugin in &mut self.plugins {
            println!("Initializing plugin: {}", plugin.name());
            plugin.init()?;
        }
        Ok(())
    }

    fn dispatch(&self, event: &Event) -> Result<(), PluginError> {
        for plugin in &self.plugins {
            plugin.on_event(event)?;
        }
        Ok(())
    }

    fn shutdown_all(&self) -> Result<(), PluginError> {
        for plugin in self.plugins.iter().rev() {
            plugin.shutdown()?;
        }
        Ok(())
    }
}
```

Usage:

```rust
fn main() -> Result<(), PluginError> {
    let mut host = PluginHost::new();
    host.register(Box::new(LoggingPlugin::new()));
    host.register(Box::new(MetricsPlugin::new("statsd://localhost:8125")));
    host.register(Box::new(WebhookPlugin::new("https://hooks.example.com/events")));
    host.init_all()?;

    host.dispatch(&Event {
        kind: "user.created".into(),
        payload: serde_json::json!({"user_id": "abc123"}),
    })?;

    host.shutdown_all()
}
```

Each plugin is a struct that implements the `Plugin` trait:

```rust
struct LoggingPlugin;

impl LoggingPlugin {
    fn new() -> Self { Self }
}

impl Plugin for LoggingPlugin {
    fn name(&self) -> &str { "logging" }

    fn init(&mut self) -> Result<(), PluginError> {
        println!("[logging] initialized");
        Ok(())
    }

    fn on_event(&self, event: &Event) -> Result<(), PluginError> {
        println!("[logging] event: {} - {}", event.kind, event.payload);
        Ok(())
    }
}
```

This is not really a "plugin system" - it's just polymorphism with a lifecycle. But it's worth stating explicitly because it covers more use cases than people think. If the set of plugins is known at compile time, there's no reason to go further. You get type checking, dead code elimination, LTO across plugin boundaries, and zero indirection beyond the vtable.

I covered the cost of `dyn Trait` dispatch in the [strategy pattern post](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/) - it's a pointer to data plus a pointer to a vtable. One level of indirection per call. For plugin init/shutdown that runs once, this cost is irrelevant. For hot-path event dispatch, it's measurable but rarely the bottleneck compared to whatever the plugin actually does.

**When to use this:** Internal extensibility points. Framework hooks. Middleware chains. Anything where "plugins" ship as crates and the host compiles them in.

## Approach 2: Compile-time registration with `inventory`

The `Vec<Box<dyn Plugin>>` approach requires the host's `main` to know about every plugin. The [`inventory`](https://crates.io/crates/inventory) crate (v0.3.24) removes that coupling. Plugins register themselves at compile time using linker sections, and the host discovers them by iterating a typed registry. No central list.

Here's how it works under the hood. On Linux, `inventory::submit!` places a function pointer in the `.text.startup` ELF section. On Windows, it goes into `.CRT$XCU`. These are "life-before-main" initialization sections - the OS runs them before `main` is called, similar to `__attribute__((constructor))` in C. Each shim evaluates an expression and pushes the result into a global `Vec` behind a `Once` lock. By the time `main` starts, every submitted value is already in the registry.

The setup requires a small adapter because `inventory` works with concrete types, not trait objects. You wrap a `Box<dyn Plugin>` in a newtype:

```rust
use inventory;

pub struct PluginEntry {
    pub create: fn() -> Box<dyn Plugin>,
}

inventory::collect!(PluginEntry);
```

Each plugin registers a factory function:

```rust
inventory::submit!(PluginEntry {
    create: || Box::new(LoggingPlugin::new()),
});
```

The host discovers all plugins without knowing their types:

```rust
fn main() -> Result<(), PluginError> {
    let mut host = PluginHost::new();

    for entry in inventory::iter::<PluginEntry> {
        let plugin = (entry.create)();
        println!("Discovered plugin: {}", plugin.name());
        host.register(plugin);
    }

    host.init_all()?;
    // ...
    Ok(())
}
```

This is powerful for multi-crate projects. Imagine a workspace where each crate provides plugins. They each call `inventory::submit!` in their own source files. The host binary pulls them in as dependencies, and `inventory::iter` collects everything. No manual wiring. Add a new plugin crate to the workspace, add it as a dependency, and it appears automatically.

The catch: this is still compile-time. The user can't drop a file into a folder and have it picked up. All plugin code must be present when the host binary is compiled. But within that constraint, it's ergonomic and zero-cost at runtime - the iteration happens once at startup.

**When to use this:** Plugin architectures within a Rust workspace. Extensible libraries where downstream crates register implementations. Test fixture discovery.

## Approach 3: Dynamic loading with `libloading`

True runtime extensibility - where users provide `.so`/`.dylib`/`.dll` files that the host loads at startup without recompilation - requires dynamic loading. The [`libloading`](https://crates.io/crates/libloading) crate (v0.9.0, ~333 million downloads) wraps `dlopen`/`dlsym` on Unix and `LoadLibrary`/`GetProcAddress` on Windows.

This is where things get serious. Rust does not have a stable ABI. Two Rust crates compiled with different compiler versions (or even different optimization flags) can have incompatible memory layouts for the same struct. You can't just export a `fn create_plugin() -> Box<dyn Plugin>` from a shared library and load it from the host - the vtable layout, `Box` representation, and `String` internals might all differ.

The standard solution: drop down to `extern "C"`. The C ABI is stable across compilers, platforms, and languages. Your plugin interface becomes a set of C-compatible function pointers.

First, define the FFI interface in a shared crate that both the host and plugin depend on:

```rust
// plugin-api/src/lib.rs
use std::os::raw::c_char;
use std::ffi::CStr;

/// Opaque handle to plugin state.
pub type PluginHandle = *mut std::ffi::c_void;

/// FFI-safe plugin vtable.
#[repr(C)]
pub struct PluginVTable {
    pub name: unsafe extern "C" fn(PluginHandle) -> *const c_char,
    pub init: unsafe extern "C" fn(PluginHandle) -> i32,
    pub on_event: unsafe extern "C" fn(PluginHandle, *const c_char, *const c_char) -> i32,
    pub shutdown: unsafe extern "C" fn(PluginHandle) -> i32,
    pub drop: unsafe extern "C" fn(PluginHandle),
}

/// Every plugin .so must export this function.
pub type CreatePluginFn = unsafe extern "C" fn() -> PluginInstance;

#[repr(C)]
pub struct PluginInstance {
    pub handle: PluginHandle,
    pub vtable: PluginVTable,
}
```

Notice the `#[repr(C)]` on every struct that crosses the FFI boundary. Without it, the Rust compiler is free to reorder fields and add padding however it wants. If you're not familiar with FFI string types like `CStr`, I covered the full zoo of Rust's string types in [a previous post](/blog/why-rust-has-so-many-string-types/).

A plugin implemented as a shared library:

```rust
// my-plugin/src/lib.rs
use plugin_api::*;
use std::ffi::{c_char, c_void, CStr, CString};

struct MyPlugin {
    name: CString,
}

impl MyPlugin {
    fn new() -> Self {
        Self {
            name: CString::new("my-dynamic-plugin").unwrap(),
        }
    }
}

unsafe extern "C" fn plugin_name(handle: PluginHandle) -> *const c_char {
    let plugin = &*(handle as *const MyPlugin);
    plugin.name.as_ptr()
}

unsafe extern "C" fn plugin_init(handle: PluginHandle) -> i32 {
    let plugin = &mut *(handle as *mut MyPlugin);
    println!("[{}] initialized", plugin.name.to_str().unwrap_or("?"));
    0 // success
}

unsafe extern "C" fn plugin_on_event(
    handle: PluginHandle,
    kind: *const c_char,
    payload: *const c_char,
) -> i32 {
    let plugin = &*(handle as *const MyPlugin);
    let kind = CStr::from_ptr(kind).to_str().unwrap_or("?");
    let payload = CStr::from_ptr(payload).to_str().unwrap_or("?");
    println!("[{}] event: {} - {}", plugin.name.to_str().unwrap_or("?"), kind, payload);
    0
}

unsafe extern "C" fn plugin_shutdown(handle: PluginHandle) -> i32 {
    let plugin = &*(handle as *const MyPlugin);
    println!("[{}] shutting down", plugin.name.to_str().unwrap_or("?"));
    0
}

unsafe extern "C" fn plugin_drop(handle: PluginHandle) {
    let _ = Box::from_raw(handle as *mut MyPlugin);
}

#[no_mangle]
pub extern "C" fn create_plugin() -> PluginInstance {
    let plugin = Box::new(MyPlugin::new());
    PluginInstance {
        handle: Box::into_raw(plugin) as PluginHandle,
        vtable: PluginVTable {
            name: plugin_name,
            init: plugin_init,
            on_event: plugin_on_event,
            shutdown: plugin_shutdown,
            drop: plugin_drop,
        },
    }
}
```

Build it as a cdylib:

```toml
# my-plugin/Cargo.toml
[lib]
crate-type = ["cdylib"]
```

The host loads plugins from a directory:

```rust
use libloading::{Library, Symbol};
use plugin_api::{CreatePluginFn, PluginInstance};
use std::ffi::CString;
use std::path::Path;

struct LoadedPlugin {
    instance: PluginInstance,
    _library: Library, // must outlive the plugin
}

fn load_plugins_from_dir(dir: &Path) -> Vec<LoadedPlugin> {
    let mut plugins = Vec::new();

    let entries = match std::fs::read_dir(dir) {
        Ok(e) => e,
        Err(_) => return plugins,
    };

    for entry in entries.flatten() {
        let path = entry.path();
        let ext = path.extension().and_then(|e| e.to_str()).unwrap_or("");

        // Match platform-specific extensions
        let is_plugin = match std::env::consts::OS {
            "linux" => ext == "so",
            "macos" => ext == "dylib",
            "windows" => ext == "dll",
            _ => false,
        };

        if !is_plugin {
            continue;
        }

        let result = (|| unsafe {
            let lib = Library::new(&path).ok()?;
            let create: Symbol<CreatePluginFn> = lib.get(b"create_plugin").ok()?;
            let instance = create();
            Some(LoadedPlugin {
                instance,
                _library: lib,
            })
        })();

        if let Some(loaded) = result {
            let name = unsafe {
                std::ffi::CStr::from_ptr((loaded.instance.vtable.name)(loaded.instance.handle))
            };
            println!("Loaded plugin: {}", name.to_str().unwrap_or("?"));
            plugins.push(loaded);
        }
    }

    plugins
}
```

There's a critical detail here: `_library: Library` must live as long as the plugin uses function pointers from it. If the `Library` is dropped, the shared object is unloaded, and every function pointer becomes a dangling reference. The underscore prefix tells Rust "I'm keeping this for its Drop behavior, not because I use it."

### The ABI stability problem

The C ABI constraint is painful. Every value crossing the boundary must be `repr(C)`: no `String`, no `Vec`, no `Box<dyn Trait>`, no `Result`, no `Option`. You're working with raw pointers, integer error codes, and null-terminated strings. It feels like writing C with Rust syntax.

Two crates try to solve this. [`abi_stable`](https://crates.io/crates/abi_stable) (v0.11.3) provides `#[sabi_trait]` - a macro that generates FFI-safe trait objects with a stable vtable layout - plus ABI-safe replacements for standard types (`RString`, `RVec`, `RBox`). It also adds load-time type checking so a mismatched plugin fails immediately instead of corrupting memory.

The newer [`stabby`](https://crates.io/crates/stabby) (maintained by ZettaScaleLabs) takes a different approach with `#[stabby::stabby]`, generating `repr(C)` layouts that still retain some of rustc's enum optimizations. It's more actively maintained as of early 2026.

Both crates let you write Rust-to-Rust FFI that feels like normal trait usage while maintaining ABI stability across compiler versions. But they add complexity and binary size. For most projects, the raw `extern "C"` approach with a well-defined vtable struct is simpler to audit and debug.

**When to use this:** Desktop applications with user-installed extensions. Build tools that load language-specific handlers. Any case where the plugin author can't recompile the host.

## Approach 4: WASM plugins

Dynamic loading has a fundamental trust problem. A `.so` file runs in the host's process with the host's permissions. A malicious or buggy plugin can read environment variables, open network connections, write to arbitrary files, or segfault the entire process. There's no isolation.

WebAssembly solves this with sandboxing. A WASM module runs in a virtual machine with no access to the host's memory, filesystem, or network unless the host explicitly grants it. It's the only approach on this list where you can safely load and execute code from untrusted sources.

[`extism`](https://crates.io/crates/extism) (v1.21.0) provides a high-level plugin framework built on [`wasmtime`](https://crates.io/crates/wasmtime) (v43.0.0). The host side is straightforward:

```rust
use extism::{Manifest, Plugin, Wasm};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Load a WASM plugin from a file
    let wasm = Wasm::file("plugins/my_plugin.wasm");
    let manifest = Manifest::new([wasm]);

    // Third argument enables WASI support (filesystem, env, etc.)
    let mut plugin = Plugin::new(&manifest, [], false)?;

    // Call a function exported by the plugin
    let result = plugin.call::<&str, &str>("on_event", r#"{"kind":"user.created"}"#)?;
    println!("Plugin returned: {}", result);

    Ok(())
}
```

The plugin is compiled to `wasm32-wasip2` and exports functions:

```rust
use extism_pdk::*;

#[plugin_fn]
pub fn on_event(input: String) -> FnResult<String> {
    let event: serde_json::Value = serde_json::from_str(&input)?;
    let kind = event["kind"].as_str().unwrap_or("unknown");
    Ok(format!("Processed event: {}", kind))
}
```

Build it:

```toml
# plugin/Cargo.toml
[lib]
crate-type = ["cdylib"]

[dependencies]
extism-pdk = "1.4"
serde_json = "1"
```

```bash
cargo build --target wasm32-wasip2 --release
```

The output is a `.wasm` file that the host loads at runtime. The plugin can't access the filesystem, network, or host memory unless the host explicitly provides WASI capabilities or host functions. If the plugin panics, the WASM runtime catches it - the host process stays alive.

This is the approach that Zellij, Zed, and Lapce use for their extension systems. Zellij moved from wasmtime to the wasmi interpreter in v0.44.0, trading execution speed for smaller binary size. Zed uses WASI Preview 2 with the WebAssembly Component Model, defining plugin interfaces in WIT (WebAssembly Interface Types) files that serve as a versioned contract. [Arroyo](https://www.arroyo.dev/blog/rust-plugin-systems/) went the opposite direction - they chose C ABI dynamic loading over WASM because their UDF plugins often depend on C libraries that don't compile to WASM easily.

### The overhead

WASM isn't free. There's JIT or AOT compilation overhead when loading a module, memory copying when passing data across the boundary (the plugin and host don't share memory), and the runtime itself adds to binary size. Wasmtime's cranelift code generator produces quality machine code, but it won't match native Rust compiled with LTO. For plugin functions called once per user action (like formatting output or transforming data), you'll never notice. For tight loops processing millions of events per second, benchmark first.

**When to use this:** Any system that loads code from users or third parties. Marketplace-style extension systems. Anything where a buggy plugin must not crash the host.

## Lifecycle and ordering

Whichever approach you pick, plugin lifecycle follows the same pattern:

1. **Discovery** - find plugins (scan a directory, iterate `inventory`, read a config file)
2. **Loading** - instantiate the plugin (call constructor, dlopen the library, compile the WASM)
3. **Initialization** - call `init()` in dependency order
4. **Runtime** - dispatch events, call hooks, delegate work
5. **Shutdown** - call `shutdown()` in reverse initialization order

The reverse-order shutdown matters. If plugin B depends on a resource that plugin A provides, you want B to clean up before A. This is the same pattern as destructor ordering in C++ or bean destruction in Spring.

A priority field helps:

```rust
pub trait Plugin: Send + Sync {
    fn name(&self) -> &str;
    fn priority(&self) -> i32 { 0 } // higher = initialized first
    fn init(&mut self) -> Result<(), PluginError>;
    fn on_event(&self, event: &Event) -> Result<(), PluginError>;
    fn shutdown(&self) -> Result<(), PluginError> { Ok(()) }
}
```

Sort plugins by priority before init, reverse-sort before shutdown. Explicit ordering beats implicit ordering every time.

If you've read the [dependency injection post](/blog/dependency-injection-patterns-without-a-framework/), you'll recognize the parallel - plugins are essentially runtime-discovered dependencies that the host wires together at startup. The `PluginHost` struct is a simple service locator. In more complex systems, you might let plugins request other plugins by name, forming a dependency graph that the host resolves topologically.

## Trade-offs at a glance

| | Trait objects | inventory | Dynamic loading | WASM |
|---|---|---|---|---|
| **Adds plugins at runtime** | No | No | Yes | Yes |
| **Sandbox isolation** | No | No | No | Yes |
| **Multi-language plugins** | No | No | Yes (C ABI) | Yes |
| **Call overhead** | vtable (negligible) | vtable (negligible) | function pointer (negligible) | data copy + VM |
| **Binary size** | Single binary | Single binary | Host + .so per plugin | Host + .wasm per plugin + runtime |
| **Complexity** | Low | Low | High (unsafe, FFI) | Medium (with extism) |
| **Plugin author DX** | Write Rust, add dep | Write Rust, add dep | Write Rust/C, build cdylib | Write any lang, compile to WASM |
| **ABI stability needed** | No | No | Yes | No (WASM is the ABI) |

## Picking the right approach

Start with the simplest approach that meets your actual constraints:

**Compile-time plugins** (approaches 1-2) if all plugin code is Rust, maintained by your team or close collaborators, and ships in the same binary. Most internal tools, CLI applications, and web frameworks fall here. Axum's middleware chain, for instance, is basically approach 1 with generics instead of trait objects.

**Dynamic loading** (approach 3) if plugins must be distributed separately from the host binary, but you trust the plugin authors. Build systems, database engines, and language runtimes use this. The complexity is real - you're committing to a C ABI contract and `unsafe` code that you need to audit carefully.

**WASM** (approach 4) if you need isolation, multi-language support, or can't trust plugin code. Extension marketplaces, user-facing customization systems, and multi-tenant platforms. The ecosystem has matured significantly - wasmtime 43.0.0 supports WASIp3 with async I/O, HTTP, and database access through the component model.

There's no shame in starting with approach 1 and migrating later. The `Plugin` trait stays the same across all four approaches. What changes is how you discover, load, and wrap implementations. If your trait interface is clean, the migration path is straightforward.

The plugin system is not the hard part. The hard part is designing the right trait - figuring out what hooks to expose, what data to pass, and what capabilities to grant. Get that interface right, and the loading mechanism is just plumbing.
