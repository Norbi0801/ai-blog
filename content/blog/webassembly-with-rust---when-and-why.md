+++
title = "WebAssembly with Rust - when and why"
date = 2025-05-07
description = "A practical look at compiling Rust to WebAssembly - where it genuinely outperforms JavaScript, where it doesn't, and how to ship small .wasm binaries."

[taxonomies]
tags = ["rust", "webassembly", "performance", "javascript"]
+++

WebAssembly has been "the future of the web" for about eight years now. In that time it went from an experimental compile target to something Cloudflare runs across 330+ edge locations, Figma uses for its rendering engine, and Google Earth uses for 3D terrain processing. But for most developers, the question isn't whether WASM is useful somewhere - it's whether it's useful for *their* specific problem.

This post is about that decision. When does compiling Rust to WASM actually pay off? What's the toolchain look like in practice? Where does WASM beat JavaScript, and where does it lose? And once you've committed to shipping a `.wasm` file, how do you keep it small?

<!-- more -->

## The three places where Rust-to-WASM makes sense

Not every workload benefits from WASM. The overhead of loading, compiling, and instantiating a module means you need enough CPU-bound work to amortize that startup cost. Three categories consistently clear that bar.

### CPU-intensive computation in the browser

Image manipulation, video encoding, physics simulations, cryptography, compression. Anything where you'd write a tight loop over a large buffer. JavaScript engines have gotten remarkably fast, but they still carry overhead that WASM doesn't: garbage collection pauses, JIT deoptimization when type assumptions break, and boxed numerics when values escape to the heap.

Figma's renderer is the canonical example. Their multiplayer design tool compiles a C++ rendering engine to WASM and runs it in the browser. The result is sub-frame latency for complex vector operations that would be impractical in pure JavaScript. [Squoosh](https://squoosh.app/), Google's image compression tool, runs codecs like MozJPEG and AVIF as WASM modules - codecs originally written in C/C++ that would take years to rewrite in JavaScript and still be slower.

For Rust specifically, the story is strong. Rust's ownership model means no GC in the compiled output, predictable memory layout, and the compiler's aggressive inlining and monomorphization produce tight machine code even through the WASM abstraction layer.

### Edge computing and serverless

Cloudflare Workers, Fastly Compute, and Fermyon Spin all run WASM as their execution model. The key property: WASM modules start in microseconds, not milliseconds. Fastly reports running over 100,000 WASM isolates per CPU core. Compare that to spinning up a Node.js process or a container.

For edge functions that need to be fast, cold-start quickly, and run in a sandbox, Rust-to-WASM is the natural choice. You write Rust, compile to `wasm32-wasip2`, deploy it to an edge platform, and get near-native speed with memory-safe sandboxing. The WASI (WebAssembly System Interface) standard gives your module access to files, environment variables, and HTTP - but only what the host explicitly grants.

```rust
// A Cloudflare Worker written in Rust, compiled to WASM
use worker::*;

#[event(fetch)]
async fn fetch(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    let url = req.url()?;
    let path = url.path();

    match path {
        "/api/compress" => {
            let body = req.bytes().await?;
            let compressed = miniz_oxide::deflate::compress_to_vec(&body, 6);
            Response::from_bytes(compressed)
        }
        _ => Response::error("not found", 404),
    }
}
```

### Plugin systems

I covered this in detail in [implementing a plugin system in Rust](/blog/implementing-a-plugin-system-in-rust/), but the short version: WASM provides memory-safe sandboxing for untrusted code. A `.wasm` plugin can't read your environment variables, can't access the filesystem, and can't segfault your host process. That's why Zed, Zellij, and Lapce all use WASM for their extension systems.

## The toolchain: wasm-pack and wasm-bindgen

Two tools form the backbone of Rust-to-WASM for browser targets.

[**wasm-bindgen**](https://crates.io/crates/wasm-bindgen) (v0.2.116) generates the glue code between Rust and JavaScript. When you annotate a Rust function with `#[wasm_bindgen]`, the macro generates JavaScript bindings that handle type conversion, memory management, and the WASM linear memory protocol. It handles strings, numbers, booleans, structs, and enums across the boundary.

[**wasm-pack**](https://crates.io/crates/wasm-pack) (v0.14.0) orchestrates the build. It calls `cargo build --target wasm32-unknown-unknown`, runs `wasm-bindgen` to generate JS bindings, optionally runs `wasm-opt` for optimization, and packages the result as an npm module you can import from JavaScript like any other dependency.

Here's the minimal setup:

```toml
# Cargo.toml
[package]
name = "image-blur"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"

[profile.release]
lto = true
opt-level = "z"
strip = true
codegen-units = 1
panic = "abort"
```

The `crate-type = ["cdylib"]` is essential. It tells Cargo to produce a C-compatible dynamic library, which is the format wasm-pack expects as input for generating the `.wasm` binary. The `rlib` is optional - it lets you also use the crate as a normal Rust library for testing.

The `[profile.release]` section deserves a closer look. Each setting targets binary size:

- `lto = true` - enables link-time optimization across all crates, letting LLVM inline and eliminate dead code across crate boundaries
- `opt-level = "z"` - optimizes aggressively for size over speed (though `"s"` sometimes produces smaller binaries - measure both)
- `strip = true` - removes debug symbols and name sections from the output
- `codegen-units = 1` - compiles the entire crate as a single unit, giving LLVM maximum visibility for optimization
- `panic = "abort"` - removes the unwinding machinery, saving roughly 10-20 KB depending on the crate graph

## A practical example: image blur from Rust, called from JavaScript

Theory is nice. Let's build something. A box blur over raw pixel data - the kind of operation where WASM should clearly win over JavaScript because it's a tight numeric loop over a large buffer.

The Rust side:

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

/// Apply a box blur to RGBA pixel data.
/// `width` and `height` are image dimensions.
/// `radius` is the blur kernel radius (1 = 3x3, 2 = 5x5, etc.)
#[wasm_bindgen]
pub fn box_blur(pixels: &mut [u8], width: u32, height: u32, radius: u32) {
    let w = width as usize;
    let h = height as usize;
    let r = radius as usize;

    // We need a scratch buffer for the intermediate horizontal pass
    let mut buf = vec![0u8; pixels.len()];

    // Horizontal pass
    for y in 0..h {
        for x in 0..w {
            let mut r_sum: u32 = 0;
            let mut g_sum: u32 = 0;
            let mut b_sum: u32 = 0;
            let mut a_sum: u32 = 0;
            let mut count: u32 = 0;

            let x_start = x.saturating_sub(r);
            let x_end = (x + r + 1).min(w);

            for kx in x_start..x_end {
                let idx = (y * w + kx) * 4;
                r_sum += pixels[idx] as u32;
                g_sum += pixels[idx + 1] as u32;
                b_sum += pixels[idx + 2] as u32;
                a_sum += pixels[idx + 3] as u32;
                count += 1;
            }

            let idx = (y * w + x) * 4;
            buf[idx] = (r_sum / count) as u8;
            buf[idx + 1] = (g_sum / count) as u8;
            buf[idx + 2] = (b_sum / count) as u8;
            buf[idx + 3] = (a_sum / count) as u8;
        }
    }

    // Vertical pass (reads from buf, writes back to pixels)
    for y in 0..h {
        for x in 0..w {
            let mut r_sum: u32 = 0;
            let mut g_sum: u32 = 0;
            let mut b_sum: u32 = 0;
            let mut a_sum: u32 = 0;
            let mut count: u32 = 0;

            let y_start = y.saturating_sub(r);
            let y_end = (y + r + 1).min(h);

            for ky in y_start..y_end {
                let idx = (ky * w + x) * 4;
                r_sum += buf[idx] as u32;
                g_sum += buf[idx + 1] as u32;
                b_sum += buf[idx + 2] as u32;
                a_sum += buf[idx + 3] as u32;
                count += 1;
            }

            let idx = (y * w + x) * 4;
            pixels[idx] = (r_sum / count) as u8;
            pixels[idx + 1] = (g_sum / count) as u8;
            pixels[idx + 2] = (b_sum / count) as u8;
            pixels[idx + 3] = (a_sum / count) as u8;
        }
    }
}
```

Build it:

```bash
wasm-pack build --target web --release
```

This produces a `pkg/` directory containing `image_blur_bg.wasm`, `image_blur.js` (the generated bindings), and `package.json`. The JS file handles instantiating the WASM module and managing the linear memory buffer.

The JavaScript side:

```javascript
import init, { box_blur } from './pkg/image_blur.js';

async function blurImage(canvas) {
    await init(); // loads and compiles the .wasm

    const ctx = canvas.getContext('2d');
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

    // imageData.data is a Uint8ClampedArray - wasm_bindgen handles
    // copying it into WASM linear memory and back
    box_blur(imageData.data, canvas.width, canvas.height, 5);

    ctx.putImageData(imageData, 0, 0);
}
```

That's it. The `#[wasm_bindgen]` attribute on the Rust function generates a JavaScript wrapper that handles the `&mut [u8]` parameter. Under the hood, it allocates space in the WASM module's linear memory, copies the pixel data in, calls the Rust function, and copies the modified data back out.

### What happens under the hood

The `&mut [u8]` parameter is where things get interesting. WASM modules can't directly access JavaScript's heap. They have their own linear memory - a contiguous `ArrayBuffer` that grows on demand. When you pass a `Uint8ClampedArray` to a wasm-bindgen function that takes `&mut [u8]`, the generated JS code:

1. Allocates `n` bytes in the WASM module's linear memory using the module's `__wbindgen_malloc` export
2. Copies the JavaScript array's data into that allocation
3. Passes the pointer and length to the Rust function (two `i32` arguments at the WASM level)
4. After the function returns, copies the data back from WASM memory to the JavaScript array
5. Calls `__wbindgen_free` to deallocate the WASM-side buffer

This copy-in/copy-out is the main overhead of WASM interop. For our blur example with a 1920x1080 RGBA image, that's ~8 MB copied in and ~8 MB copied back. On modern hardware this takes under a millisecond, but for functions called thousands of times per second on small data, the copy overhead can dominate the actual computation.

If you need zero-copy access, you can work with the WASM linear memory directly from JavaScript:

```javascript
import init, { box_blur, memory } from './pkg/image_blur.js';

// Access WASM linear memory directly
const wasmMemory = new Uint8Array(memory.buffer);
```

But this is fragile - the memory buffer can be invalidated when WASM allocates and the buffer grows. Use it only when profiling shows the copy is the bottleneck.

## WASM vs JavaScript: what's actually faster

The benchmarks paint a nuanced picture. Research from the [Benchmarking WebAssembly](https://benchmarkingwasm.github.io/BenchmarkingWebAssembly/) project and independent tests on [The New Stack](https://thenewstack.io/webassembly-vs-javascript-testing-side-by-side-performance/) show:

**WASM wins (2-6x faster, sometimes more):**
- Tight numeric loops (matrix multiplication, image processing, physics)
- Cryptographic operations (hashing, encryption)
- Compression/decompression (zlib, brotli, zstd)
- Parsing binary formats (protobuf, msgpack)
- Anything that benefits from predictable memory layout and no GC pauses

On mobile devices the gap widens further - Safari on iPhone showed up to 30x speedups for small compute-intensive workloads in some benchmarks, because mobile JS engines are more conservative with JIT optimization to save battery.

**JavaScript wins or ties:**
- DOM manipulation (WASM can't touch the DOM directly - every call goes through JavaScript)
- String-heavy operations (Rust strings are UTF-8, JS strings are UTF-16 - conversion overhead adds up)
- Small functions called frequently across the boundary (interop overhead dominates)
- Memory-heavy workloads with large allocations (WASM linear memory management is less sophisticated than V8's GC for certain allocation patterns)

**The surprising one:** for large input sizes, JavaScript can actually be faster. V8's GC handles large, short-lived allocations efficiently. WASM modules managing their own heap via `dlmalloc` or `wee_alloc` don't always match that for workloads with heavy allocation churn. The research shows WASM memory usage jumps by 24 MB for large inputs and 74 MB for extra-large inputs, while JavaScript stays relatively flat.

The practical takeaway: don't port your entire frontend to Rust. Port the hot loop. Keep everything else in JavaScript. The hybrid approach - JavaScript for UI, WASM for compute - consistently outperforms going all-in on either side.

## Limitations you'll hit

### No direct DOM access

WASM code runs in a sandbox. It can't call `document.getElementById()` or manipulate the DOM directly. Every DOM interaction must go through JavaScript. Frameworks like [Leptos](https://leptos.dev/) and [Yew](https://yew.rs/) abstract this away with virtual DOM implementations, but under the hood they're still calling JavaScript for every DOM mutation.

This is why full-WASM frontend frameworks remain niche. For a typical CRUD web app, you're paying WASM compilation, bundle size, and interop overhead to do something that JavaScript already does efficiently. WASM frontends make more sense for applications that are compute-heavy and DOM-light - think CAD tools, audio workstations, or data visualization.

### Threading is there, but constrained

WASM threads are supported in all modern browsers (96.78% global coverage according to [Can I Use](https://caniuse.com/wasm-threads)), but the programming model is unusual. Each thread is a Web Worker that creates its own WASM instance sharing the same `WebAssembly.Memory` backed by a `SharedArrayBuffer`. WASM globals are thread-local. WASM tables (used for indirect function calls) can't be shared across workers.

On the Rust side, you can't just use `std::thread::spawn`. The `wasm32-unknown-unknown` target has no thread support. For parallelism you need `wasm-bindgen-rayon` or manual Web Worker management. Rayon's work-stealing threadpool can be made to work in WASM, but it requires the right HTTP headers (`Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp`) because `SharedArrayBuffer` was restricted after the Spectre vulnerabilities.

```rust
// This compiles but panics at runtime on wasm32-unknown-unknown:
// std::thread::spawn(|| { /* ... */ });

// Instead, use wasm-bindgen-rayon for data-parallel workloads
use rayon::prelude::*;

#[wasm_bindgen]
pub fn parallel_sum(data: &[f64]) -> f64 {
    data.par_iter().sum()
}
```

### No filesystem, no network sockets

In the browser, WASM modules have no filesystem access and no raw socket access. You get what the browser gives you: `fetch()` through JavaScript interop, `IndexedDB` through JavaScript interop, and `WebSocket` through JavaScript interop. Everything goes through the JS bridge.

Outside the browser, WASI changes this picture. WASI Preview 2 (stable since late 2024) provides filesystem, environment variables, clocks, and random number generation. WASI Preview 3 is in development and will add async I/O and HTTP as first-class capabilities. But in the browser, you're sandboxed.

## Shrinking the .wasm binary

A naive `cargo build --release --target wasm32-unknown-unknown` on our image blur example produces a `.wasm` file of about 43 KB. Not terrible, but we can do much better.

### Step 1: Cargo.toml profile (already covered above)

The `[profile.release]` settings from earlier (`lto`, `opt-level = "z"`, `strip`, `codegen-units = 1`, `panic = "abort"`) get us down to roughly 28 KB. Most of that savings comes from `lto` and `panic = "abort"`.

### Step 2: wasm-opt

[`wasm-opt`](https://github.com/WebAssembly/binaryen) is a post-compilation optimizer from the Binaryen toolkit. It runs transformation passes on the `.wasm` binary that LLVM's WASM backend doesn't perform. Install it and run:

```bash
wasm-opt -Oz -o output.wasm input.wasm
```

The `-Oz` flag optimizes aggressively for size. This typically shaves another 15-20% off the binary. Our blur module drops to about 23 KB.

wasm-pack runs `wasm-opt` automatically if it's installed. You can verify by checking the build output for "Optimizing wasm binaries with `wasm-opt`..."

### Step 3: Audit what's in the binary with twiggy

[Twiggy](https://rustwasm.github.io/twiggy/) is a code size profiler for WASM. It parses the binary's sections and call graph to show what's actually consuming space.

```bash
twiggy top output.wasm
```

This produces a table like:

```
 Shallow Bytes │ Shallow % │ Item
───────────────┼───────────┼──────────────────────
          4812 │    20.93% │ box_blur
          2340 │    10.18% │ dlmalloc::dlmalloc::Dlmalloc::malloc
          1876 │     8.16% │ __rust_alloc
          1204 │     5.24% │ core::slice::sort::merge_sort
           ...
```

If you see functions you don't expect (like `core::fmt::write` or `core::panicking::panic_fmt`), those are pulled in by `format!` macros, `.unwrap()` calls, or `assert!` statements. Each one adds formatting machinery. Replace `.unwrap()` with `.unwrap_or()` or match statements in hot paths, and avoid `format!` in release builds.

### Step 4: Replace the allocator

Rust's default allocator in WASM is `dlmalloc`, which is general-purpose and relatively large. If your module makes few allocations, you can switch to a smaller allocator. The [`wee_alloc`](https://crates.io/crates/wee_alloc) allocator trades allocation speed for a ~10 KB reduction in code size. Note that `wee_alloc` is no longer actively maintained, so [`lol_alloc`](https://crates.io/crates/lol_alloc) and [`talc`](https://crates.io/crates/talc) are alternatives worth evaluating.

```rust
#[global_allocator]
static ALLOC: talc::Talck<talc::locking::AssumeUnlockable, talc::ClaimOnOom> =
    talc::Talc::new(talc::ClaimOnOom).lock();
```

### Step 5: wasm-snip for dead code

[`wasm-snip`](https://github.com/nickel-org/nickel.rs/tree/master/nickel_wasm_snip) replaces function bodies with `unreachable` traps. Run it on functions that twiggy shows as large but that you know aren't called at runtime (like panic formatting in a module that never panics):

```bash
wasm-snip --snip-rust-panicking-code input.wasm -o snipped.wasm
wasm-opt -Oz snipped.wasm -o final.wasm
```

The `wasm-opt` pass after snipping is important - it runs dead code elimination (`--dce`) that removes functions only reachable through the now-unreachable snipped code.

### Step 6: Compression

After all the above, gzip or brotli compression on the server side cuts the transfer size dramatically. WASM binaries compress well because they have repetitive instruction patterns. Our 23 KB blur module compresses to about 9 KB with gzip. Make sure your CDN or web server serves `.wasm` files with `Content-Encoding: gzip` or `br`.

### Size progression summary

| Stage | Size |
|---|---|
| Default release build | ~43 KB |
| + LTO, opt-level "z", panic abort, strip | ~28 KB |
| + wasm-opt -Oz | ~23 KB |
| + gzip | ~9 KB |

For comparison, React's minified+gzipped bundle is about 44 KB. A well-optimized WASM module that does real computation can be smaller than your UI framework.

## Passing complex types across the boundary

Primitive types (`u32`, `f64`, `bool`) cross the WASM boundary directly - they map to WASM's native `i32`, `f64`, etc. But what about strings, structs, and collections?

### Strings

Rust strings are UTF-8. JavaScript strings are UTF-16. wasm-bindgen handles the conversion automatically, but it's not free - every string crossing the boundary gets transcoded. For functions that process strings heavily, this overhead can matter.

```rust
#[wasm_bindgen]
pub fn count_words(text: &str) -> u32 {
    text.split_whitespace().count() as u32
}
```

The generated JavaScript converts the JS string to UTF-8 bytes, copies them into WASM linear memory, and passes a pointer + length. After the function returns, the WASM-side allocation is freed. For a 10 KB string, this adds maybe 5-10 microseconds. For a 10 MB document processed once, it's negligible. For a 100-byte string processed 100,000 times per second, consider passing a reference to a persistent buffer instead.

### Structs

Structs annotated with `#[wasm_bindgen]` become JavaScript classes:

```rust
#[wasm_bindgen]
pub struct Point {
    pub x: f64,
    pub y: f64,
}

#[wasm_bindgen]
impl Point {
    #[wasm_bindgen(constructor)]
    pub fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }

    pub fn distance_to(&self, other: &Point) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }
}
```

```javascript
import { Point } from './pkg/my_module.js';

const a = new Point(0, 0);
const b = new Point(3, 4);
console.log(a.distance_to(b)); // 5.0

// Important: call .free() when done, or use the explicit destructor
a.free();
b.free();
```

The JavaScript `Point` object is a handle (an integer index) into the WASM module's internal slab allocator. The actual struct data lives in WASM linear memory. This means JavaScript never sees the raw bytes - it calls methods through the generated bindings. The downside: if you forget to call `.free()`, the WASM-side memory leaks. There's no garbage collector watching the WASM heap.

### Collections and complex types: serde-wasm-bindgen

For `Vec<T>`, `HashMap`, nested structs, or any complex type, the cleanest approach is [`serde-wasm-bindgen`](https://crates.io/crates/serde-wasm-bindgen). It converts Rust types to native JavaScript values through serde, avoiding the JSON round-trip that older approaches required:

```rust
use serde::{Serialize, Deserialize};
use wasm_bindgen::prelude::*;

#[derive(Serialize, Deserialize)]
pub struct SearchResult {
    pub title: String,
    pub score: f64,
    pub highlights: Vec<(usize, usize)>,
}

#[wasm_bindgen]
pub fn search(query: &str, documents: JsValue) -> Result<JsValue, JsError> {
    let docs: Vec<String> = serde_wasm_bindgen::from_value(documents)?;

    let results: Vec<SearchResult> = docs
        .iter()
        .enumerate()
        .filter(|(_, doc)| doc.contains(query))
        .map(|(i, doc)| SearchResult {
            title: format!("Document {}", i),
            score: 1.0 / (i + 1) as f64,
            highlights: find_matches(doc, query),
        })
        .collect();

    Ok(serde_wasm_bindgen::to_value(&results)?)
}

fn find_matches(doc: &str, query: &str) -> Vec<(usize, usize)> {
    doc.match_indices(query)
        .map(|(start, matched)| (start, start + matched.len()))
        .collect()
}
```

This is now the officially recommended approach over `JsValue::from_serde`, which was deprecated. `serde-wasm-bindgen` produces smaller code than the JSON-based alternative and handles JavaScript-specific types (like `undefined`, `Map`, `Set`) correctly.

## When not to use WASM

If you've read this far and you're thinking "I should rewrite my Next.js app in Rust + WASM" - probably not. Here's when JavaScript is the better choice:

- **CRUD web apps.** If your app is forms, lists, modals, and API calls, JavaScript frameworks handle this well. The DOM interaction overhead of WASM would slow you down, not speed you up.
- **Small utility functions.** A function that validates an email or formats a date doesn't have enough compute to overcome WASM's instantiation and interop costs.
- **Text-heavy processing where V8 is already optimized.** Regular expressions, JSON parsing, string manipulation - V8 has years of optimization for these. You'd be surprised how often `JSON.parse()` in JavaScript beats a Rust JSON parser compiled to WASM because V8's native C++ JSON parser handles it, not the JIT-compiled JavaScript.
- **Prototyping.** The Rust-to-WASM feedback loop is slower than editing JavaScript and refreshing the browser. Save it for when you've validated the idea and identified a genuine performance bottleneck.

The sweet spot is surgical. Identify the function that's too slow in JavaScript, port that function to Rust, compile it to WASM, call it from JavaScript. Keep everything else in JavaScript. That's the pattern that Figma, Squoosh, and most successful WASM-in-production projects follow.

If you're coming from a TypeScript background, the mental model translation I covered in [Rust for TypeScript developers](/blog/rust-for-typescript-developers-a-mental-model-translation/) applies directly here - you're writing Rust, but the calling code is still JavaScript. And if you need to profile the Rust side to find what's actually slow, the techniques from [profiling Rust code](/blog/profiling-rust-code-finding-performance-bottlenecks/) work on WASM targets too (twiggy for size, `console.time()`/`console.timeEnd()` on the JS side, and Chrome DevTools' WASM profiler for flame graphs).

## The bottom line

WASM with Rust is a precision tool. It's not a replacement for JavaScript - it's an escape hatch for when JavaScript isn't fast enough. The toolchain (`wasm-pack` + `wasm-bindgen`) is mature. The optimization path (LTO, `wasm-opt`, twiggy, compression) is well-documented and effective. The limitations (no DOM, no filesystem in-browser, copy overhead at the boundary) are real but manageable.

Start with a profiler. Find the bottleneck. Port that one function. Measure again. That's the workflow. If the numbers justify it, WASM is one of the most impactful performance tools available to web developers today. If they don't, you saved yourself from a premature optimization that would have made your codebase harder to maintain for no measurable benefit.
