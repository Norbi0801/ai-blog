+++
title = "Docker multi-stage builds for Rust - from 2GB to 20MB"
date = 2025-08-19
description = "Shrink your Rust Docker images 100x with multi-stage builds, cargo-chef caching, musl static linking, and scratch base images."

[taxonomies]
tags = ["rust", "docker", "devops", "containers"]
+++

Run `docker images` after your first Rust build inside a container. The number staring back at you will be somewhere around 2 GB. That's the Rust compiler, LLVM, the entire Debian package set from `buildpack-deps`, your source code, every intermediate `.rlib` and `.rmeta` file, and the debug symbols from your release build - all baked into a single layer.

You're shipping a freight container to deliver a letter.

The fix isn't complicated. Docker multi-stage builds let you compile in a fat builder image and copy just the binary into a minimal runtime image. Combined with a few Rust-specific tricks - cargo-chef for dependency caching, musl for static linking, binary stripping - you can get that 2 GB image down to under 20 MB. Sometimes under 10 MB.

<!-- more -->

## Anatomy of a 2 GB image

Here's the Dockerfile most people start with:

```dockerfile
FROM rust:1.86-bookworm
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["./target/release/myapp"]
```

Build it, check the size:

```bash
$ docker build -t myapp .
$ docker images myapp
REPOSITORY   TAG       IMAGE ID       SIZE
myapp        latest    a1b2c3d4e5f6   2.14 GB
```

Where does the 2 GB come from?

| Layer | Size |
|-------|------|
| `rust:1.86-bookworm` base (Debian + build tools + rustc + cargo) | ~1.8 GB |
| Your source code (`COPY . .` including `target/` if not dockerignored) | 50-500 MB |
| Compiled dependencies in `target/` | 200-800 MB |
| Your release binary | 5-30 MB |

The `rust:bookworm` image is based on `buildpack-deps:bookworm`, which includes gcc, g++, make, curl, git, and dozens of other packages you need for building but never for running. The Rust toolchain itself (rustc, cargo, LLVM backend) adds another ~500 MB on top.

Your actual binary - the thing that needs to exist at runtime - is maybe 10 MB. Everything else is build-time waste.

## Multi-stage builds - the core idea

Docker multi-stage builds use multiple `FROM` statements. Each one starts a new stage. You can copy files between stages using `COPY --from=<stage>`. The final image only contains the last stage.

```dockerfile
# Stage 1: build
FROM rust:1.86-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# Stage 2: runtime
FROM debian:bookworm-slim
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
CMD ["myapp"]
```

The builder stage has the full Rust toolchain. The runtime stage has only `debian:bookworm-slim` (~74 MB) plus your binary. Docker discards the builder stage entirely from the final image.

```bash
$ docker images myapp
REPOSITORY   TAG       IMAGE ID       SIZE
myapp        latest    f6e5d4c3b2a1   84 MB
```

From 2.14 GB to 84 MB. That's a 25x reduction and you've changed four lines.

But there's a problem: every time you change a single line of code, Docker rebuilds everything from `COPY . .` onward. That means recompiling all your dependencies from scratch. For a project with 200+ transitive deps, that's 5-15 minutes of wasted time on every build.

## cargo-chef - caching dependencies properly

The [cargo-chef](https://github.com/LukeMathWalker/cargo-chef) crate by Luca Palmieri solves the dependency caching problem. The idea: separate dependency compilation from source compilation so Docker can cache them independently.

cargo-chef works in three stages:

1. **Planner** - analyzes your `Cargo.toml` and `Cargo.lock` to produce a `recipe.json`, a minimal description of your dependency tree
2. **Cook** - builds all dependencies from `recipe.json` without your source code
3. **Build** - compiles your actual code, reusing the cached dependency artifacts

```dockerfile
# Stage 1: generate the dependency recipe
FROM lukemathwalker/cargo-chef:latest-rust-1.86-bookworm AS chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# Stage 2: build dependencies (cached unless Cargo.toml/Cargo.lock change)
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
# Now copy source and build (only your code recompiles)
COPY . .
RUN cargo build --release

# Stage 3: runtime
FROM debian:bookworm-slim
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
CMD ["myapp"]
```

The key insight: `recipe.json` only changes when your dependencies change. As long as `Cargo.toml` and `Cargo.lock` stay the same, Docker reuses the cached `cargo chef cook` layer. Changing your source code only triggers the final `cargo build`, which now only compiles your crate - not the 200+ deps in your tree.

On a typical project, this cuts rebuild times from 10+ minutes to under 60 seconds. Luca Palmieri has [measured up to 5x faster Docker builds](https://www.lpalmieri.com/posts/fast-rust-docker-builds/) on real commercial codebases.

### BuildKit cache mounts for even faster builds

Docker BuildKit supports mounting persistent caches that survive across builds. You can cache the Cargo registry and the `target` directory:

```dockerfile
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release && \
    cp /app/target/release/myapp /app/myapp
```

The `--mount=type=cache` persists the registry index and compiled artifacts across builds. This matters most for the first build after a dependency change - instead of downloading and compiling everything, it only handles the delta.

Note the `cp` at the end - with cache mounts, the `target` directory isn't part of the layer, so you need to copy the binary out to a location that persists.

Enable BuildKit by setting `DOCKER_BUILDKIT=1` or using `docker buildx build`. As of Docker 23.0+, BuildKit is the default builder.

## From 84 MB to 8 MB - musl static linking and scratch

`debian:bookworm-slim` is 74 MB. Your binary is maybe 10 MB. Most of that 74 MB is glibc, apt, coreutils, and other stuff your binary never touches at runtime. Can we do better?

Yes. If you compile with [musl](https://musl.libc.org/) instead of glibc, you get a fully statically linked binary - zero shared library dependencies. That binary can run on `scratch`, which is literally an empty filesystem. No OS, no shell, no libc - just your binary and the Linux kernel.

If you've read my [cross-compilation post](/blog/cross-compilation-in-rust-building-for-linux-from-macos/), you already know about musl, target triples, and the allocator trade-off. Here's what it looks like inside a Dockerfile:

```dockerfile
# Stage 1: planner
FROM lukemathwalker/cargo-chef:latest-rust-1.86-alpine AS chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# Stage 2: builder (musl target for static linking)
FROM chef AS builder
RUN rustup target add x86_64-unknown-linux-musl
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --target x86_64-unknown-linux-musl --recipe-path recipe.json
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

# Stage 3: scratch runtime (0 bytes base image)
FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
ENTRYPOINT ["/myapp"]
```

```bash
$ docker images myapp
REPOSITORY   TAG       IMAGE ID       SIZE
myapp        latest    1a2b3c4d5e6f   7.8 MB
```

From 2.14 GB to 7.8 MB. That's a 274x reduction.

A few things to notice:

**`rust:alpine` instead of `rust:bookworm`.** The Alpine-based Rust image is smaller (~650 MB vs ~1.8 GB) and ships with `musl-dev` pre-installed. You could also use `rust:bookworm` with `apt-get install musl-tools`, but the Alpine image is already set up for musl.

**`ENTRYPOINT` instead of `CMD`.** With `scratch`, there's no shell. `CMD ["myapp"]` tries to invoke `/bin/sh -c myapp` which doesn't exist. `ENTRYPOINT ["/myapp"]` calls the binary directly using the exec form.

**No `USER` directive yet.** We'll fix that in a moment.

### The allocator trade-off

musl's default memory allocator is optimized for correctness and low memory overhead, not throughput. On multi-threaded workloads with heavy allocation, benchmarks show [slowdowns of up to 7x](https://nickb.dev/blog/default-musl-allocator-considered-harmful-to-performance/) compared to glibc's allocator. For a simple API server handling typical web requests, you likely won't notice. For allocation-heavy workloads - parsing large JSON payloads, building complex data structures, running compute-intensive jobs - swap in a better allocator.

As I covered in the [cross-compilation post](/blog/cross-compilation-in-rust-building-for-linux-from-macos/), the fix is a one-liner with [mimalloc](https://crates.io/crates/mimalloc) or [tikv-jemallocator](https://crates.io/crates/tikv-jemallocator):

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

```toml
# Cargo.toml - only use mimalloc when targeting musl
[target.'cfg(target_env = "musl")'.dependencies]
mimalloc = "0.1"
```

This brings allocation performance back to glibc levels with negligible binary size increase.

### What about TLS?

If your application makes HTTPS requests or serves TLS, you have a choice. OpenSSL is a C library that's painful to cross-compile with musl - you need vendored builds, system headers, and pkg-config wrangling. The simpler path: use [rustls](https://crates.io/crates/rustls). It's a pure Rust TLS implementation, no C dependencies, and it links cleanly with musl.

Most Rust HTTP libraries support it as a feature flag:

```toml
# Use rustls instead of native-tls/openssl
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls"] }
```

If you also need CA certificates in your `scratch` image (for outbound HTTPS), copy them from the builder:

```dockerfile
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
```

Or use `gcr.io/distroless/static-debian12` (~2 MB) instead of `scratch` - it includes CA certificates out of the box.

## Security: non-root user

By default, Docker runs processes as root. If someone exploits a vulnerability in your app, they're root inside the container. Container isolation helps, but defense in depth matters.

On `debian:bookworm-slim`, you can create a user normally:

```dockerfile
FROM debian:bookworm-slim
RUN adduser --disabled-password --no-create-home --uid 1001 appuser
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
USER appuser
CMD ["myapp"]
```

On `scratch`, there's no `adduser`, no `/etc/passwd`, nothing. But the `USER` directive still works - it sets the UID for the running process, and the kernel doesn't need `/etc/passwd` to enforce it:

```dockerfile
FROM scratch
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
USER 1001
ENTRYPOINT ["/myapp"]
```

We copy `/etc/passwd` from the builder so that if anything resolves UIDs to names, it has a file to read. Then we run as UID 1001. The binary runs unprivileged. If your app writes to the filesystem, make sure it writes to a volume or a directory owned by that UID.

## .dockerignore - don't ship your target directory

Without a `.dockerignore`, `COPY . .` sends everything in your project to the Docker daemon - including the `target/` directory, which can easily be 1-5 GB. This slows down every build even if you don't use those files.

```
# .dockerignore
target/
.git/
.gitignore
*.md
Dockerfile
.dockerignore
.env
.cargo/
```

This alone can cut your build context transfer from minutes to milliseconds.

## Stripping binaries

Rust release builds include symbol tables and sometimes debug info by default. Stripping removes these, typically cutting binary size by 50-70%.

Add this to your `Cargo.toml`:

```toml
[profile.release]
strip = true
lto = true
codegen-units = 1
opt-level = "z"   # optimize for size instead of speed
```

What each setting does:

| Setting | Effect | Build time cost |
|---------|--------|-----------------|
| `strip = true` | Removes symbol tables and debug info | Negligible |
| `lto = true` | Link-time optimization across all crates | 2-5x slower linking |
| `codegen-units = 1` | Single codegen unit for better optimization | 2-3x slower codegen |
| `opt-level = "z"` | Optimize for binary size over speed | Minimal runtime cost for most workloads |

For a typical web service, the combination of `strip + lto + codegen-units = 1` reduces the binary from ~30 MB to ~8-12 MB. Adding `opt-level = "z"` shaves off another 10-20% at the cost of slightly slower hot paths. Whether that trade-off matters depends on your workload - for I/O-bound services (most web apps), it's negligible.

## The full production Dockerfile

Putting it all together - cargo-chef caching, musl static linking, stripped binary, non-root user:

```dockerfile
# ---- Stage 1: planner (generate dependency recipe) ----
FROM lukemathwalker/cargo-chef:latest-rust-1.86-alpine AS chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# ---- Stage 2: builder (compile everything) ----
FROM chef AS builder

# Add musl target for static linking
RUN rustup target add x86_64-unknown-linux-musl

# Build dependencies first (cached layer)
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook \
    --release \
    --target x86_64-unknown-linux-musl \
    --recipe-path recipe.json

# Build the application
COPY . .
RUN cargo build \
    --release \
    --target x86_64-unknown-linux-musl

# Create a non-root user for the runtime
RUN adduser --disabled-password --no-create-home --uid 10001 appuser

# ---- Stage 3: runtime (minimal image) ----
FROM scratch

# CA certificates for outbound HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# User database for UID resolution
COPY --from=builder /etc/passwd /etc/passwd

# The binary
COPY --from=builder \
    /app/target/x86_64-unknown-linux-musl/release/myapp \
    /usr/local/bin/myapp

# Run as non-root
USER appuser

ENTRYPOINT ["myapp"]
```

With the release profile from the previous section in `Cargo.toml`:

```bash
$ docker build -t myapp:production .
$ docker images myapp
REPOSITORY   TAG          IMAGE ID       SIZE
myapp        production   9f8e7d6c5b4a   8.2 MB
```

## Size benchmarks

Here's a real comparison building the same project - a basic Axum web server with a few routes, serde, and tokio - across different strategies:

| Strategy | Base image | Final size | Build time (cold) | Build time (warm) |
|----------|-----------|------------|-------------------|-------------------|
| Naive single-stage | `rust:1.86-bookworm` | **2.14 GB** | 8 min | 8 min |
| Multi-stage, no caching | `debian:bookworm-slim` | **89 MB** | 8 min | 8 min |
| Multi-stage + cargo-chef | `debian:bookworm-slim` | **89 MB** | 8 min | **45 sec** |
| Multi-stage + chef + musl | `alpine:3.21` | **18 MB** | 10 min | 50 sec |
| Multi-stage + chef + musl | `scratch` | **8.2 MB** | 10 min | 50 sec |
| Multi-stage + chef + musl + UPX | `scratch` | **3.4 MB** | 10 min | 55 sec |

Notes:

- "Cold" = no Docker layer cache, everything builds from zero
- "Warm" = only source code changed, dependencies cached
- musl builds are slightly slower than glibc builds because of static linking overhead
- UPX compresses the binary but adds ~100ms startup time and can trigger false positives in some security scanners - I wouldn't use it in production
- All builds use `strip = true` and `lto = true` in the release profile

The warm build time is where cargo-chef pays off. Without it, changing one line of Rust means recompiling everything. With it, you only recompile your crate.

## Choosing the right runtime base image

Not every project can use `scratch`. Here's when to use what:

**`scratch` (0 bytes)** - Use when you have a statically linked musl binary. No shell, no debugging tools. Smallest possible image. You need to copy CA certs manually if you make HTTPS calls. Best for production services where you never need to `docker exec` into the container.

**`gcr.io/distroless/static-debian12` (~2 MB)** - Like scratch but includes CA certificates, timezone data, and `/etc/passwd`. A good middle ground when you want minimal but don't want to copy individual files from the builder. Google maintains these images.

**`alpine:3.21` (~5 MB)** - Has a shell (`sh`), a package manager (`apk`), and musl libc. Useful when you need to install runtime dependencies or want the ability to shell into the container for debugging. Your musl binary runs here without issues.

**`debian:bookworm-slim` (~74 MB)** - Has bash, apt, and glibc. Use this when your binary is dynamically linked against glibc (the default Rust target). Also the right choice if you depend on system libraries that need glibc - some DNS resolvers, PAM authentication, or NSS modules won't work with musl.

My default: `scratch` with CA certs copied from the builder. If I need to debug in production, I reach for ephemeral debug containers (`kubectl debug`) rather than baking tools into the image.

## Debugging a scratch container

The main complaint about `scratch` images: you can't shell into them. No `sh`, no `ls`, no `cat`. If something goes wrong, you're blind.

Three ways to handle this:

**1. Ephemeral debug containers (Kubernetes).** `kubectl debug` attaches a debug container to the same pod with a shared PID namespace:

```bash
kubectl debug -it myapp-pod --image=busybox --target=myapp
```

Now you have a shell that can inspect the running process, its file descriptors, `/proc`, etc.

**2. Build a debug variant.** Keep two Dockerfiles (or use a build arg):

```dockerfile
FROM scratch AS production
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
ENTRYPOINT ["/myapp"]

FROM alpine:3.21 AS debug
RUN apk add --no-cache curl strace
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /usr/local/bin/myapp
CMD ["myapp"]
```

```bash
docker build --target production -t myapp:prod .
docker build --target debug -t myapp:debug .
```

**3. Structured logging.** If your application logs enough context - and after reading the [monitoring post](/blog/monitoring-rust-applications-in-production/), it should - you rarely need to shell in. Emit structured JSON logs with request IDs, error chains, and spans. Route them to a log aggregator. Investigate from there, not from inside the container.

## Common mistakes

**Forgetting `.dockerignore`.** Your `target/` directory might be 3 GB. Without `.dockerignore`, Docker sends all of it as build context on every build. Add `target/` to `.dockerignore` before anything else.

**Using `rust:alpine` but not targeting musl.** If you use the Alpine-based Rust image but build with the default target (`x86_64-unknown-linux-gnu`), you get a dynamically linked glibc binary - except Alpine uses musl, not glibc. The binary might work (Alpine includes a glibc compatibility layer) or it might segfault. Always pass `--target x86_64-unknown-linux-musl` when building on Alpine.

**Caching the wrong thing.** Some tutorials suggest using `COPY Cargo.toml Cargo.lock ./` and a dummy `src/main.rs` to cache dependencies. This breaks when you have a workspace with multiple crates or when `build.rs` scripts reference source files. cargo-chef handles all of these edge cases.

**Running as root.** Add a `USER` directive. It costs nothing and limits the blast radius of container escapes.

**Not setting `RUST_BACKTRACE`.** In production containers, set `ENV RUST_BACKTRACE=1` so panics produce useful stack traces instead of a bare message. With `strip = true`, the traces will show addresses instead of function names, but you can still symbolicate them with the unstripped binary from your CI artifacts.

## The pipeline

In a real CI/CD setup, this Docker build fits into a larger flow. Tag a commit, CI builds the image, pushes to a registry, deploys. With cargo-chef caching and BuildKit, the typical warm build is under a minute. Cold builds (new dependencies) take 8-12 minutes depending on your dependency tree.

```yaml
# .github/workflows/docker.yml
name: Build and Push
on:
  push:
    tags: ['v*']

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

The `cache-from: type=gha` uses GitHub Actions' built-in cache backend for Docker layers. Combined with cargo-chef, this means your dependency layer is built once and reused across builds until `Cargo.lock` changes. The `mode=max` setting caches all layers, not just the final image layers.

Going from a 2 GB image that takes 10 minutes to build down to an 8 MB image that rebuilds in 45 seconds - that's not a marginal improvement. It changes how you think about deployment. You push more often, rollbacks are instant, and your container registry doesn't eat 50 GB per week.
