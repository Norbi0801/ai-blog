+++
title = "Cross-compilation in Rust - building for Linux from macOS"
date = 2025-03-23
description = "The full cross-compilation setup in Rust: rustup targets, linker config, musl static binaries, cargo-zigbuild, cross-rs, ARM builds, and CI matrices that ship one binary per platform."

[taxonomies]
tags = ["rust", "cross-compilation", "devops", "ci"]
+++

You wrote a CLI tool on your Mac. It works. Now your coworker on Ubuntu wants to use it. You `cargo build --release`, send them the binary, and they get:

```
bash: ./mytool: cannot execute binary file: Exec format error
```

Right. You compiled a Mach-O binary for `aarch64-apple-darwin`. Their machine expects an ELF binary for `x86_64-unknown-linux-gnu`. Same source code, same language, completely incompatible output. This is where cross-compilation comes in - building a binary on one platform that runs on another.

Rust makes cross-compilation dramatically easier than C or C++. The compiler already speaks every target's ABI. `rustup` can fetch pre-built standard libraries for over 250 targets. But "easier" does not mean "zero effort." You still need a linker that understands the target, you need to handle native C dependencies, and you need to decide between dynamic linking (glibc), static linking (musl), or sidestepping the problem entirely with Docker.

<!-- more -->

## Target triples - what you're actually telling the compiler

Every Rust target is identified by a target triple (which is often a quadruple). The format is `<arch>-<vendor>-<os>-<env>`:

```
x86_64-unknown-linux-gnu
  |       |      |    |
  |       |      |    +-- environment (C library / ABI)
  |       |      +------- operating system
  |       +-------------- vendor (often "unknown")
  +---------------------- CPU architecture
```

Some real targets you'll encounter:

| Target | What it is |
|--------|-----------|
| `x86_64-unknown-linux-gnu` | Linux on x86_64, dynamically linked to glibc |
| `x86_64-unknown-linux-musl` | Linux on x86_64, statically linked with musl |
| `aarch64-unknown-linux-gnu` | Linux on ARM64 (Raspberry Pi 4/5, AWS Graviton) |
| `aarch64-apple-darwin` | macOS on Apple Silicon |
| `x86_64-apple-darwin` | macOS on Intel |
| `x86_64-pc-windows-msvc` | Windows on x86_64 with MSVC ABI |
| `x86_64-pc-windows-gnu` | Windows on x86_64 with MinGW ABI |

Rust currently supports around [258 platform targets](https://doc.rust-lang.org/rustc/platform-support.html) across three tiers. Tier 1 (8 targets) is guaranteed to build and pass tests. Tier 2 (~73 targets) is guaranteed to build. Tier 3 (~177 targets) has platform support in the codebase but no automatic builds.

Run `rustup target list` to see what's available for your toolchain. The ones you have installed are marked:

```bash
$ rustup target list --installed
aarch64-apple-darwin
x86_64-unknown-linux-gnu
```

## The minimal setup - rustup target add and a linker

Cross-compilation in Rust has two ingredients:

1. **The Rust standard library compiled for the target.** `rustup` fetches this.
2. **A linker that can produce binaries for the target.** This is the part that trips people up.

Step one is always the same:

```bash
rustup target add x86_64-unknown-linux-gnu
```

This downloads pre-compiled `std`, `core`, and `alloc` for that target. Now `rustc` can compile your Rust code to x86_64 Linux object files. But it can't link them - because your macOS `ld` doesn't understand ELF binaries. You need a cross-linker.

### Installing a cross-linker on macOS

The traditional approach uses GNU cross-toolchains via Homebrew:

```bash
# For x86_64 Linux (glibc)
brew install x86_64-unknown-linux-gnu

# For aarch64 Linux (glibc)
brew install aarch64-unknown-linux-gnu

# For musl (static binaries)
brew install filosottile/musl-cross/musl-cross
```

Then tell Cargo which linker to use. Create or edit `.cargo/config.toml` in your project (or `~/.cargo/config.toml` globally):

```toml
[target.x86_64-unknown-linux-gnu]
linker = "x86_64-linux-gnu-gcc"

[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"

[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
```

Now build:

```bash
cargo build --target x86_64-unknown-linux-gnu --release
```

The resulting binary lands in `target/x86_64-unknown-linux-gnu/release/mytool`. Copy it to a Linux machine, `chmod +x`, run it. If your project is pure Rust with no C dependencies, this often just works.

### What happens when it doesn't just work

The moment your dependency tree includes a `-sys` crate - anything that wraps a C library - things get complicated. `openssl-sys` needs OpenSSL headers and libraries compiled for the target. `libsqlite3-sys` needs SQLite for the target. Your macOS Homebrew libraries are compiled for `aarch64-apple-darwin` and are useless for a Linux target.

You have three options:

1. **Avoid C dependencies.** Use `rustls` instead of `openssl`. Use `rusqlite` with the `bundled` feature (it compiles SQLite from source). This is the path of least resistance.
2. **Install cross-compiled libraries.** This gets painful fast. You need the right headers, the right `.a` or `.so` files, and you need to set `PKG_CONFIG_PATH` and various environment variables.
3. **Use a tool that handles it for you.** This is where `cross`, `cargo-zigbuild`, and Docker come in.

## cargo-zigbuild - the linker shortcut

[cargo-zigbuild](https://github.com/rust-cross/cargo-zigbuild) replaces the system linker with Zig's bundled C compiler and linker. Why does that help? Because Zig ships with pre-built libc for dozens of platforms. You don't need to install a GNU cross-toolchain at all.

Install it:

```bash
cargo install cargo-zigbuild
# Install zig itself
brew install zig
# Or: pip3 install ziglang
```

Then replace `cargo build` with `cargo zigbuild`:

```bash
rustup target add x86_64-unknown-linux-gnu
cargo zigbuild --target x86_64-unknown-linux-gnu --release
```

No `.cargo/config.toml` linker entry needed. No cross-toolchain to install. Zig handles it.

One particularly useful feature: you can pin the minimum glibc version by appending it to the target:

```bash
# Build for glibc 2.17 (CentOS 7 / RHEL 7 compatible)
cargo zigbuild --target x86_64-unknown-linux-gnu.2.17 --release
```

This matters because a binary linked against glibc 2.35 (Ubuntu 22.04) will crash with `GLIBC_2.35 not found` on a server running glibc 2.28 (Debian 10). Pinning glibc 2.17 gives you compatibility with almost any Linux machine from the last decade. Some teams have reported [5-10x faster build times](https://www.drmhse.com/posts/fast-rust-docker-builds-with-zigbuild/) after switching to cargo-zigbuild compared to Docker-based cross-compilation.

The limitation: cargo-zigbuild only supports Linux and macOS targets currently. Windows cross-compilation still needs other tools.

## musl - fully static binaries

Dynamic linking against glibc means your binary depends on the exact glibc version on the target machine. A binary built on Ubuntu 24.04 might not run on Debian 11 because the glibc versions differ. This is the "works on my machine" problem at the binary level.

musl libc solves this by producing fully static binaries. No shared library dependencies at all. The binary is self-contained - copy it anywhere, it runs.

```bash
rustup target add x86_64-unknown-linux-musl

# With cargo-zigbuild (easiest)
cargo zigbuild --target x86_64-unknown-linux-musl --release

# Or with the musl-cross toolchain
cargo build --target x86_64-unknown-linux-musl --release
```

Verify it's actually static:

```bash
$ file target/x86_64-unknown-linux-musl/release/mytool
mytool: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
statically linked, stripped, with debug_info

$ ldd target/x86_64-unknown-linux-musl/release/mytool
not a dynamic executable
```

Compare that to a glibc build:

```bash
$ ldd target/x86_64-unknown-linux-gnu/release/mytool
linux-vdso.so.1 (0x00007ffce93fe000)
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
/lib64/ld-linux-x86-64.so.2
```

### The trade-offs of musl

Static musl binaries are not universally better. Some things to know:

**Binary size increases.** The entire C standard library gets linked in. A hello-world Rust binary goes from ~300KB (glibc, dynamic) to ~400KB (musl, static). For larger projects the delta shrinks proportionally - your application code dwarfs libc.

**DNS resolution behaves differently.** glibc's DNS resolver supports `/etc/nsswitch.conf`, mDNS, and various OS-level DNS plugins. musl's resolver only reads `/etc/resolv.conf`. In most server environments this is fine. In environments with complex name resolution (some corporate networks, some container orchestrators), you might hit surprising lookup failures.

**Allocator performance.** musl's default allocator (mallocng since musl 1.2.1) is designed for correctness and low memory overhead, not raw throughput. For allocation-heavy workloads, you might see 10-30% slowdowns compared to glibc's ptmalloc2. The fix: use a custom allocator like [jemalloc](https://crates.io/crates/tikv-jemallocator) or [mimalloc](https://crates.io/crates/mimalloc):

```rust
use tikv_jemallocator::Jemalloc;

#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;

fn main() {
    // musl binary, but with jemalloc's allocation performance
}
```

Add to `Cargo.toml`:

```toml
[target.'cfg(target_env = "musl")'.dependencies]
tikv-jemallocator = "0.6"
```

This uses jemalloc only when building for musl, leaving glibc builds with their default allocator.

## cross-rs - Docker-based cross-compilation

[cross](https://github.com/cross-rs/cross) (8.1k GitHub stars) takes a different approach. Instead of installing cross-toolchains on your host, it runs the entire build inside a Docker container that has the right toolchain, headers, and libraries pre-installed.

```bash
cargo install cross --git https://github.com/cross-rs/cross

# Drop-in replacement for cargo build
cross build --target x86_64-unknown-linux-gnu --release
cross build --target aarch64-unknown-linux-gnu --release
cross build --target armv7-unknown-linux-gnueabihf --release
```

cross supports 50+ targets out of the box. It handles C dependencies that would be painful to set up manually - it ships Docker images with cross-compiled OpenSSL, zlib, and other common libraries.

The downsides: you need Docker running, builds are slower (Docker overhead, no shared compilation cache by default), and the images are large (some are 1-2 GB). For CI where you're building once and throwing away the environment, this is fine. For local iteration where you're rebuilding every few seconds, the Docker overhead adds up.

### Custom cross configuration

If your project has unusual C dependencies, you can extend cross's Docker images. Create a `Cross.toml` at the project root:

```toml
[target.x86_64-unknown-linux-gnu]
image = "ghcr.io/cross-rs/x86_64-unknown-linux-gnu:main"
pre-build = [
    "apt-get update && apt-get install -y libpq-dev"
]
```

This installs PostgreSQL development headers into the cross-compilation container before building.

## Building for ARM - Raspberry Pi and AWS Graviton

ARM targets are one of the most common cross-compilation needs. Raspberry Pi 4 and 5 run `aarch64-unknown-linux-gnu` (64-bit) or `armv7-unknown-linux-gnueabihf` (32-bit). AWS Graviton instances are `aarch64-unknown-linux-gnu`.

With cargo-zigbuild:

```bash
rustup target add aarch64-unknown-linux-gnu
cargo zigbuild --target aarch64-unknown-linux-gnu --release
```

With cross:

```bash
cross build --target aarch64-unknown-linux-gnu --release
```

For a Raspberry Pi Zero (ARMv6):

```bash
cross build --target arm-unknown-linux-gnueabihf --release
```

### Testing ARM binaries without hardware

You can run ARM binaries on your x86 machine using QEMU:

```bash
# Install QEMU user-mode emulation
brew install qemu

# Run an aarch64 Linux binary on x86 macOS
qemu-aarch64 ./target/aarch64-unknown-linux-gnu/release/mytool
```

cross does this automatically for `cross test` - it runs your test suite under QEMU for the target architecture. Performance is 5-10x slower than native, but it catches architecture-specific bugs (endianness, alignment, pointer sizes) without needing real hardware.

## Docker as a cross-compilation environment

Sometimes you don't want to install any cross-compilation tools. You just want to build inside a Linux container and copy the binary out. This is the simplest approach when you don't need to build for multiple architectures - just targeting Linux x86_64 from macOS.

```dockerfile
# Dockerfile.build
FROM rust:1.85-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/mytool /usr/local/bin/
CMD ["mytool"]
```

```bash
docker build -f Dockerfile.build -t mytool .
# Extract just the binary
docker create --name extract mytool
docker cp extract:/usr/local/bin/mytool ./mytool-linux
docker rm extract
```

For static binaries that run anywhere without a container:

```dockerfile
FROM rust:1.85-alpine AS builder
RUN apk add --no-cache musl-dev
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/mytool /
ENTRYPOINT ["/mytool"]
```

The `FROM scratch` image is literally empty. Since the musl binary has zero shared library dependencies, it runs directly on the kernel. The final image is just your binary - often under 10 MB.

This approach is simpler than setting up cross-toolchains but slower: every build runs inside Docker, and you lose your local compilation cache unless you set up Docker layer caching or cargo registry mounts.

## CI matrix builds - shipping for every platform

The real payoff of cross-compilation is automated builds. Tag a release, CI builds binaries for every target, attaches them to the GitHub release. Users download the one they need.

Here's a GitHub Actions workflow that builds for Linux (x86_64 and ARM64), macOS (Apple Silicon and Intel), and Windows:

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  build:
    strategy:
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
            name: mytool-linux-x86_64
          - target: aarch64-unknown-linux-gnu
            os: ubuntu-latest
            name: mytool-linux-aarch64
            use_cross: true
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
            name: mytool-linux-x86_64-static
          - target: aarch64-apple-darwin
            os: macos-latest
            name: mytool-macos-aarch64
          - target: x86_64-apple-darwin
            os: macos-13
            name: mytool-macos-x86_64
          - target: x86_64-pc-windows-msvc
            os: windows-latest
            name: mytool-windows-x86_64.exe
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install cross
        if: matrix.use_cross
        run: cargo install cross --git https://github.com/cross-rs/cross

      - name: Install musl tools
        if: contains(matrix.target, 'musl')
        run: sudo apt-get update && sudo apt-get install -y musl-tools

      - name: Build
        run: |
          if [ "${{ matrix.use_cross }}" = "true" ]; then
            cross build --release --target ${{ matrix.target }}
          else
            cargo build --release --target ${{ matrix.target }}
          fi
        shell: bash

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.name }}
          path: target/${{ matrix.target }}/release/mytool${{ contains(matrix.name, '.exe') && '.exe' || '' }}
```

Key decisions in this matrix:

**Native builds where possible.** macOS targets build on macOS runners, Windows on Windows runners. This avoids cross-compilation entirely for those platforms. You only cross-compile when the runner OS doesn't match the target - like building `aarch64-unknown-linux-gnu` on an `ubuntu-latest` (x86_64) runner.

**cross for ARM Linux.** GitHub doesn't offer ARM Linux runners on the free tier. You could use cargo-zigbuild instead - it's faster since there's no Docker overhead. But cross handles C dependencies more reliably.

**Separate static builds.** The musl target produces a binary that runs on any Linux distribution regardless of glibc version. Worth offering alongside the glibc build.

### Caching in CI

Cross-compilation builds can be slow. Cache the target directory per-target:

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/registry
      ~/.cargo/git
      target
    key: ${{ matrix.target }}-cargo-${{ hashFiles('**/Cargo.lock') }}
```

This saves ~2-5 minutes on subsequent builds because compiled dependencies are reused. Each target triple gets its own cache key since the compiled artifacts are architecture-specific.

## The practical checklist

When you need to ship Rust binaries for multiple platforms, here's the decision tree:

**Pure Rust project, no C dependencies?**
Use `cargo-zigbuild`. Install zig, `cargo zigbuild --target <triple> --release`. No Docker, no cross-toolchains, fast builds.

**Has C dependencies (OpenSSL, SQLite, libpq)?**
Try the `bundled` feature first (`rusqlite` supports it, `openssl` does not). If bundled isn't available, use `cross`. It ships Docker images with the right libraries pre-built.

**Need maximum portability on Linux?**
Build with `x86_64-unknown-linux-musl`. The binary runs on any Linux distribution. Consider adding `tikv-jemallocator` if you're allocation-heavy.

**Building for Raspberry Pi?**
Pi 4/5 (64-bit OS): `aarch64-unknown-linux-gnu`. Pi 3 or Pi 4 with 32-bit OS: `armv7-unknown-linux-gnueabihf`. Pi Zero/Zero W: `arm-unknown-linux-gnueabihf`.

**CI release pipeline?**
Use the matrix strategy. Build natively where you can (macOS on macOS runners, Windows on Windows runners). Cross-compile ARM Linux from x86_64. Attach all binaries to the GitHub release.

## What's actually in the binary

When you cross-compile, it helps to verify the output is what you expect. `file` tells you the format, architecture, and linking:

```bash
$ file target/x86_64-unknown-linux-gnu/release/mytool
ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
for GNU/Linux 3.2.0, BuildID[sha1]=abc123..., with debug_info, not stripped

$ file target/aarch64-unknown-linux-gnu/release/mytool
ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV),
dynamically linked, interpreter /lib/ld-linux-aarch64.so.1,
for GNU/Linux 3.7.0, with debug_info, not stripped

$ file target/x86_64-unknown-linux-musl/release/mytool
ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
statically linked, with debug_info, not stripped
```

Notice the musl binary says "statically linked" and has no interpreter line. It needs nothing from the system to run.

For production releases, strip the debug info to reduce binary size:

```bash
# Strip debug symbols (70-80% size reduction is common)
strip target/x86_64-unknown-linux-musl/release/mytool
```

Or configure Cargo to do it automatically in `Cargo.toml`:

```toml
[profile.release]
strip = true
lto = true
codegen-units = 1
```

`lto = true` enables link-time optimization across all crates. `codegen-units = 1` forces single-threaded code generation, which produces better-optimized code at the cost of slower compilation. Combined with `strip = true`, these settings typically reduce binary size by 50-70% compared to a default release build.

## When cross-compilation isn't worth it

Cross-compilation shines for distributing pre-built binaries. But sometimes the simpler answer is: don't.

If your target audience uses `cargo install`, they're building from source on their own machine. No cross-compilation needed. If you're deploying to servers you control, build on the server (or in a CI runner matching the server's architecture). If you're building container images, use a multi-stage Dockerfile - the build runs inside the container, natively.

Cross-compilation adds complexity. Linker configuration, C library compatibility, testing on foreign architectures. It's worth it when you're shipping a CLI tool to thousands of users who expect a download link. For internal services deployed to a known environment, a simple `cargo build --release` on the target platform (or in a matching container) is usually the right call.
