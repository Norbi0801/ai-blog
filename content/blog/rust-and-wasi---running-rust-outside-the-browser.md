+++
title = "Rust and WASI - running Rust outside the browser"
date = 2026-01-12
description = "WASI lets you run Rust-compiled WebAssembly with filesystem, network, and clock access outside the browser. A practical look at wasmtime, the capability model, and why WASI is the new container."

[taxonomies]
tags = ["rust", "wasi", "webassembly", "security"]
+++

If you've only seen WebAssembly running in browsers, you're seeing maybe a third of the story. The other two thirds are happening on edge networks, in serverless platforms, inside plugin hosts, and increasingly in places where you used to reach for Docker. The thing that makes it possible is WASI - the WebAssembly System Interface.

WASI is what gives a `.wasm` module the ability to open files, read environment variables, talk to a clock, or make HTTP requests. In the browser, WebAssembly modules are pure compute - if they want to touch the outside world, they call back into JavaScript and ask politely. Outside the browser, that browser-as-host model doesn't exist. WASI fills the gap with a standardized set of system-call-shaped interfaces that runtimes like wasmtime and wasmer implement.

If you're not familiar with the basics of compiling Rust to WebAssembly, I covered the toolchain, optimization, and JS interop in [WebAssembly with Rust - when and why](/blog/webassembly-with-rust---when-and-why/). This post picks up where that left off and walks through the server-side, host-side, container-replacement story.

<!-- more -->

## What WASI actually is

WASI is not a runtime. It's a specification - a set of WIT (WebAssembly Interface Types) interface definitions that describe what a host environment can expose to a WASM guest. The runtime (wasmtime, wasmer, WasmEdge, Spin, Fastly's Lucet derivative) is what implements those interfaces.

The current stable spec is **WASI 0.2** (also called Preview 2), which became the official release in January 2024. It defines a set of "worlds" - bundles of interfaces. The most common world is `wasi:cli/command`, which gives you what a typical command-line program needs: stdin, stdout, stderr, environment variables, command-line arguments, a filesystem, a clock, and random numbers. There's also `wasi:http/proxy` for HTTP servers and `wasi:keyvalue` for key-value stores.

Each interface is defined in WIT, like this snippet from the filesystem interface:

```wit
interface types {
    resource descriptor {
        read-via-stream: func(offset: filesize) -> result<input-stream, error-code>;
        write-via-stream: func(offset: filesize) -> result<output-stream, error-code>;
        sync: func() -> result<_, error-code>;
        // ... more methods
    }
}
```

This is the contract. The runtime implements it (wasmtime maps `descriptor.read-via-stream` to a real Linux file descriptor), the guest calls it through the WASM component model's typed function imports, and the runtime sandbox mediates the access.

The earlier spec, **WASI Preview 1** (`wasm32-wasip1` target, sometimes still written `wasm32-wasi`), exposed a flatter, POSIX-like ABI - functions like `fd_read`, `path_open`, `clock_time_get`. It's still widely supported because most existing Rust crates and runtimes target it, but new code should target Preview 2 (`wasm32-wasip2`) where possible. WASI Preview 3 is in active development and adds first-class async I/O and built-in HTTP - but it isn't shipping yet as of mid-2026.

## A first WASI program

Here is the smallest interesting WASI program in Rust. It reads a file, counts the lines, and prints the result to stdout. The same code you'd write for a normal Linux binary.

```rust
// src/main.rs
use std::fs;
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() < 2 {
        eprintln!("usage: line-count <path>");
        std::process::exit(1);
    }

    let contents = match fs::read_to_string(&args[1]) {
        Ok(s) => s,
        Err(e) => {
            eprintln!("could not read {}: {}", args[1], e);
            std::process::exit(1);
        }
    };

    println!("{}: {} lines", args[1], contents.lines().count());
}
```

Build it for WASI:

```bash
rustup target add wasm32-wasip2
cargo build --release --target wasm32-wasip2
```

The output is `target/wasm32-wasip2/release/line-count.wasm`. Run it with wasmtime:

```bash
wasmtime --dir=. line-count.wasm Cargo.toml
# Cargo.toml: 12 lines
```

That `--dir=.` is the important bit. It's a capability grant. Without it, the WASM module cannot see any filesystem at all - calls to `fs::read_to_string` would fail with a permission error. The host had to explicitly hand the guest a directory before any filesystem access becomes possible.

This is the heart of WASI's security model and the part most worth understanding.

## The capability model - why this is not a sandbox in the Linux sense

A Linux process inherits the world. By default, when you run a binary, it can read any file your user can read, open any TCP port, dial any IP address, look up environment variables, fork, exec, signal other processes, and so on. To restrict it, you have to actively wrap it in seccomp filters, AppArmor profiles, namespaces, or chroots. The default is "everything" and you opt out.

WASI inverts this. The default is "nothing" and you opt in. A `.wasm` module loaded by wasmtime starts with no filesystem access, no network access, no environment variables, no clock readable, no command-line arguments visible. The host must explicitly hand each capability over.

```bash
wasmtime \
    --dir=./data \
    --dir=./logs::/var/logs \
    --env=API_KEY=secret123 \
    --env=LOG_LEVEL=debug \
    -S http \
    my-app.wasm
```

What's happening here:

- `--dir=./data` - mount the host's `./data` directory at the same path inside the guest. The guest can read and write files inside this directory and nothing else. Path traversal (`../../etc/passwd`) cannot escape the mount because the host's `wasi-common` crate canonicalizes every path before it touches the real filesystem.
- `--dir=./logs::/var/logs` - mount with a different guest path. The guest sees `/var/logs`, but the host serves the files from `./logs`.
- `--env=API_KEY=secret123` - the only environment variables the guest sees are the ones explicitly forwarded. `env::vars()` inside the guest returns only `API_KEY` and `LOG_LEVEL`. Your host process's `HOME`, `PATH`, `AWS_SECRET_ACCESS_KEY`, and so on are invisible.
- `-S http` - enable the `wasi:http` interface so the guest can make HTTP requests. Without this flag, `reqwest::get(...)` (compiled with WASI HTTP support) would simply fail with a "no such interface" error at instantiation time.

This is what people mean when they say WASM modules are "deny by default." It's not just a marketing slogan - it's how the imports are linked. If the guest's WIT world references an interface the host hasn't provided, instantiation fails immediately. There's no way to "discover" capabilities at runtime; the linker enforces it before a single guest instruction executes.

Compare this to Linux containers. A container has a default-allow filesystem (rootfs) and default-allow networking (a virtual NIC bridged to the host), and the operator has to actively restrict things with read-only mounts, network policies, and capability drops (`--cap-drop=ALL`). It's the same conceptual goal - isolate untrusted code - achieved through opt-out rather than opt-in.

## How wasmtime actually runs your binary

A `.wasm` file is bytecode for a stack machine. wasmtime cannot execute it directly on x86_64 or ARM - it has to translate. The pipeline looks like this.

When you call `wasmtime ./line-count.wasm`, wasmtime:

1. **Parses** the binary using the `wasmparser` crate. It validates the module structure - section ordering, function signatures, type indices, the works. The validation is comprehensive enough that an invalid module is rejected before any code runs. [Source](https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wasmparser).
2. **Compiles** the WASM bytecode to native machine code using **Cranelift**, wasmtime's in-house code generator. Cranelift is a compiler backend roughly comparable to LLVM but designed for fast compilation rather than aggressive optimization. A typical 500 KB WASM module compiles in tens of milliseconds, versus seconds for the equivalent through LLVM. [Cranelift source](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift).
3. **Links** the imports. This is where the host wires up the WASI interfaces. The guest's import for `wasi:filesystem/types/[method]descriptor.read-via-stream` resolves to a Rust function in the host process. When the guest calls it, control transfers to native host code, which in turn issues a real `read()` syscall.
4. **Instantiates** the module. This allocates the linear memory, initializes globals, runs the module's start function (if any), and resolves the entry point.
5. **Executes** the entry point. For a CLI command, that's `_start` from the `wasi-libc` shim, which calls into your Rust `main`. The compiled native code runs in the host process's address space but with all memory accesses bounds-checked against the linear memory limits.

The bounds checks are the other half of the security story. Every load and store from WASM linear memory becomes, in compiled form, a check that the address is within the allocated range. On 64-bit hosts, wasmtime uses a clever trick: it allocates 8 GB of virtual address space per instance with most of it unmapped, so out-of-bounds accesses page-fault and the host catches the signal. The result is that a WASM module physically cannot read or write outside its sandbox - not in the "policy says no" sense but in the "the CPU's MMU traps it" sense.

You can poke at the compiled output:

```bash
wasmtime compile line-count.wasm  # produces line-count.cwasm (precompiled)
```

The `.cwasm` file is native machine code. wasmtime can `mmap` it directly and skip the Cranelift compilation step on subsequent runs. This is what edge platforms do - they precompile WASM modules at deploy time and keep the native code cached, so cold starts on incoming requests are dominated by instantiation (microseconds) rather than compilation.

## Cold starts: WASI vs containers

This is where the comparison with Docker gets interesting.

A Linux container cold start does roughly:

- `clone()` with the right namespaces (PID, mount, network, UTS, user)
- Set up the cgroup
- Mount the rootfs (overlayfs of layers)
- Set up the network namespace, veth pair, route configuration
- `execve()` the entry point binary
- The binary's dynamic linker loads shared libraries, runs initializers
- Application starts

Even a heavily optimized container - alpine base, statically linked binary - takes 50-200 ms to reach the first line of application code. Most production stacks measure 500-2000 ms cold starts.

A WASI cold start does:

- `mmap` the precompiled `.cwasm` file
- Allocate linear memory (one `mmap` of the right size)
- Resolve imports against the host's WASI implementation (a hash table lookup per import, dozens of imports total)
- Jump to `_start`

Bytecode Alliance benchmarks show wasmtime instantiation around 5-50 microseconds for typical CLI-shaped modules. Fastly reported running [over 100,000 isolates per CPU core](https://www.fastly.com/blog/announcing-lucet-fastly-native-webassembly-compiler-runtime) on their edge platform. The numbers are not in the same ballpark as containers - they're roughly 1000x faster for cold starts.

This is why edge platforms reach for WASI. When your platform receives a request and needs to run user code in response, container cold starts make per-request isolation infeasible. You end up keeping warm pools, accepting noisy-neighbor problems, or paying the cold start tax. With WASI, you can spin up a fresh isolated instance per request and the user does not notice.

## Plugins: the use case where WASI shines hardest

If you've ever shipped software with a plugin system, you know the pain. Native plugins (`.so`/`.dll`) can crash your process, leak memory, and run arbitrary code. Embedded scripting languages (Lua, Python) are slow and tie you to a specific language ecosystem. WebAssembly plugins solve both problems.

This is exactly why [Zed](https://zed.dev/), [Zellij](https://zellij.dev/), and [Lapce](https://lapce.dev/) all use WASM (with WASI for system access) for their extensions. A plugin author can write Rust, Go (TinyGo), AssemblyScript, or any language with a WASM backend. The host loads the `.wasm` file, grants it a narrow capability set (maybe access to a single config directory and a single HTTP endpoint), and runs it. If the plugin panics, only the plugin instance dies. If the plugin tries to read `~/.ssh/id_rsa`, the host says no.

A minimal plugin host in Rust using wasmtime looks like this:

```rust
use wasmtime::{Engine, Linker, Module, Store};
use wasmtime_wasi::preview1::{self, WasiP1Ctx};
use wasmtime_wasi::WasiCtxBuilder;

fn run_plugin(wasm_path: &str) -> anyhow::Result<()> {
    let engine = Engine::default();

    // Build a strict capability set: only stdout, no filesystem, no env vars
    let wasi = WasiCtxBuilder::new()
        .inherit_stdout()
        .build_p1();

    let mut store = Store::new(&engine, wasi);

    let module = Module::from_file(&engine, wasm_path)?;

    let mut linker: Linker<WasiP1Ctx> = Linker::new(&engine);
    preview1::add_to_linker_sync(&mut linker, |s| s)?;

    let instance = linker.instantiate(&mut store, &module)?;
    let start = instance.get_typed_func::<(), ()>(&mut store, "_start")?;
    start.call(&mut store, ())?;

    Ok(())
}
```

The `WasiCtxBuilder` is the capability list. The plugin gets `stdout` and nothing else. No filesystem, no network, no env vars, no clock, no random source. If you wanted to give it filesystem access to a single directory, you'd add `.preopened_dir(my_dir, "/work", DirPerms::READ, FilePerms::READ)?`. The default is locked down.

When you ship this to users, you can let them install community plugins from a registry the same way they install npm packages, and the security boundary is enforced by the runtime, not by trust. That's a big deal for any application with an extension ecosystem.

## Containers vs WASI: when to use which

The honest answer is that WASI is not a container replacement for most workloads today. It is, very specifically, a better fit for these:

| Workload shape | Container | WASI |
|---|---|---|
| Stateless function, called per-request | Possible but slow cold start | Microsecond cold start |
| Untrusted code from many tenants | Hardened, but each isolation has cost | Cheap isolation, deny-by-default |
| Multi-language plugin host | Awkward (subprocess + IPC) | Native fit |
| Full Linux app with arbitrary syscalls | Works as-is | Many syscalls unsupported |
| Daemon with persistent state, threads, sockets | Works fine | Possible but rough edges |
| Large existing binary, distroless image | Works | Won't compile |

The key gaps in WASI today:

- **Threading**: Limited. WASI 0.2 has no standard threading interface. You can use `wasi-threads` (a proposal) but support is uneven across runtimes. Most production WASI programs are single-threaded today.
- **Networking**: WASI 0.2 includes `wasi:sockets` for TCP and UDP, but library support in Rust's ecosystem is still catching up. Crates like `tokio` and `hyper` have WASI shims, but not every feature works.
- **Filesystem semantics**: WASI's filesystem interface is intentionally not a full POSIX implementation. No `mmap` of files, no `fcntl` advisory locking, no inotify. If your app uses these, it won't port cleanly.
- **Process model**: No `fork`, no `exec`, no signals between processes. A WASI program is a single instance. If you want concurrency, you do it inside one instance.

Containers handle these because they wrap a real Linux kernel. WASI by design exposes a smaller, portable surface. The trade-off is intentional - WASI binaries are smaller, start faster, run anywhere wasmtime runs (Linux, macOS, Windows, embedded), and have a tighter security model. If your workload fits, the trade is worth it.

The pattern that's emerging in production is **layered**: containers at the bottom for the operating system and the wasmtime runtime, WASI on top for tenant code. Fermyon Spin, Fastly Compute, and Cloudflare Workers all run wasmtime (or a derivative) inside their orchestration layer. Your `.wasm` deploys, but the platform is still Linux containers underneath.

## Getting your existing crates to work

The ecosystem is uneven. Some crates work out of the box on `wasm32-wasip2`, others need a `cfg` flag, others won't compile at all. Here's the rough state in mid-2026:

**Works well:** `serde`, `serde_json`, `tokio` (with the `rt` feature only - no full multithreaded runtime), `regex`, `chrono`, `uuid`, `sha2`/`blake3`, most pure-compute crates.

**Works with effort:** `reqwest` (use the `wasi` feature), `sqlx` (limited backends), `hyper` (server-side via `wasi:http`).

**Doesn't work:** Anything that opens raw sockets in non-standard ways (some database drivers), anything that uses `mio` directly, `rusqlite` with bundled SQLite (no `mmap`), anything with `unsafe` code that calls libc.

If a crate fails to compile, the usual culprits are missing system functions. The `wasi-libc` shim provides most of POSIX, but not all. When you see linker errors about undefined symbols like `pthread_create` or `getaddrinfo`, the crate is using something WASI doesn't expose. You can sometimes work around this by disabling default features:

```toml
[dependencies]
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }
```

The `default-features = false` is critical. Many crates pull in `tokio`'s full runtime, native TLS, or platform-specific code by default.

For a real-world example, here's a minimal HTTP server using `wasi:http` directly through the [`wasi`](https://crates.io/crates/wasi) crate's bindings (versions 0.13 and up speak Preview 2):

```rust
use wasi::http::types::{Fields, OutgoingResponse, ResponseOutparam};
use wasi::exports::http::incoming_handler::Guest;

struct Handler;

impl Guest for Handler {
    fn handle(
        request: wasi::http::types::IncomingRequest,
        response_out: ResponseOutparam,
    ) {
        let headers = Fields::new();
        let response = OutgoingResponse::new(headers);
        response.set_status_code(200).unwrap();

        let body = response.body().unwrap();
        let stream = body.write().unwrap();
        stream.blocking_write_and_flush(b"hello from wasi").unwrap();
        drop(stream);

        wasi::http::types::OutgoingBody::finish(body, None).unwrap();
        ResponseOutparam::set(response_out, Ok(response));
    }
}

wasi::http::proxy::export!(Handler);
```

Compile to `wasm32-wasip2`, then run it with wasmtime's `serve` subcommand:

```bash
cargo build --release --target wasm32-wasip2
wasmtime serve target/wasm32-wasip2/release/my_server.wasm
```

That's a HTTP server in a sandboxed binary that cold-starts in microseconds.

## Where this is going

WASI Preview 3 (in development as of writing) lands first-class async I/O via the component model's `future` and `stream` types, plus a deeper HTTP integration. Once it stabilizes, the rough edges around `tokio`-style concurrency mostly go away. The component model itself - which is what makes typed cross-language calls possible between WASM modules - is also getting more polished, and tools like [`cargo-component`](https://crates.io/crates/cargo-component) are making it usable without hand-writing WIT.

For now, WASI is a real, deployable thing. It's not going to replace your monolithic web app's container any time soon. But for plugin hosts, edge functions, serverless workloads, and anywhere you need to run untrusted code with a tight security boundary and a fast cold start, it is already the best tool available. Compile your Rust code with `--target wasm32-wasip2`, hand the host the capabilities it needs, and ship a binary that runs anywhere wasmtime runs - Linux server, Mac laptop, Windows VM, Raspberry Pi, FreeBSD jail - with the same bytes, the same security guarantees, and a fraction of the operational footprint of the container equivalent.
