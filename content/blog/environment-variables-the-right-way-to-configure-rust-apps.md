+++
title = "Environment variables: the right way to configure Rust apps"
date = 2025-08-27
description = "How to read env vars in Rust without footguns: dotenvy, config crate, type-safe structs, secret redaction, and where 12-factor breaks down."

[taxonomies]
tags = ["rust", "config", "12-factor", "devops"]
+++

Every non-trivial app needs configuration: a database URL, a port, a log level, a few feature flags, an API key or two. In Rust the temptation is to start with `std::env::var("DATABASE_URL").unwrap()` scattered across `main.rs` and call it done. That works for a weekend project. It falls apart the moment you add a second binary, a CI pipeline, or a colleague.

This post walks through how to do configuration properly in a Rust app: what `std::env::var` actually does at the OS level, how `dotenvy` and the `config` crate fit together, how to build a type-safe config struct that fails at startup instead of three hours into production, and how to handle secrets so they don't leak into logs or `Debug` output. Most of it is unspectacular plumbing, but the difference between getting it right on day one and bolting it on later is several afternoons of pain.

<!-- more -->

## Why env vars in the first place

The [12-factor app](https://12factor.net/config) manifesto, written in 2011 by Heroku engineers, is the reason most modern services read config from the environment. Factor III says: store config in the environment, not in the code, and not in language-specific config files committed to the repo. The argument is operational, not aesthetic:

- Env vars are language-agnostic. A Rust binary, a Python sidecar, and a shell script all read them the same way.
- Container orchestrators (Kubernetes, Nomad, ECS) inject env vars natively. There's no mounting config files or templating them at deploy time.
- Secrets managers (AWS Secrets Manager, Vault, Doppler) all expose values as env vars when the process starts.
- Diff between dev, staging, and prod is one set of env vars vs another. No code branches on `if cfg!(prod)`.

The downside is that env vars are flat strings. Anything structured (a list of allowed origins, a nested struct, a map of feature flags) has to be encoded somehow. That's where the choice between pure env, env + dotfiles, and env + config files comes in.

## What `std::env::var` actually does

```rust
use std::env;

fn main() {
    match env::var("DATABASE_URL") {
        Ok(val) => println!("DB: {}", val),
        Err(env::VarError::NotPresent) => eprintln!("DB not set"),
        Err(env::VarError::NotUnicode(_)) => eprintln!("DB has invalid UTF-8"),
    }
}
```

Two things to know about the underlying mechanism:

**1. The environment is a snapshot taken at process start.** When the OS calls `execve(2)` (Linux) or `CreateProcess` (Windows), the parent passes an array of `KEY=VALUE` strings. The Rust standard library copies that into a global table. Calling `env::var` reads from that table, not from the live OS state. If a parent shell exports a new variable after your process starts, you won't see it. If you `setenv` from C code in another thread, the Rust table may or may not pick it up depending on which API you used - and on Linux, `setenv` is famously not thread-safe ([glibc bug](https://www.evanjones.ca/setenv-is-not-thread-safe.html)).

**2. `env::set_var` and `env::remove_var` are `unsafe` as of Rust 1.85.** This was [stabilized in early 2025](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/) after years of debate. The reason is the same thread-safety issue: mutating the environment from one thread while another thread is reading it can crash. If you use `set_var` in tests, you now need an `unsafe` block, and you should hold a mutex if any test runs in parallel. We'll come back to this in the testing section.

`env::var` returns `Result<String, VarError>`. Match the error variants explicitly. `unwrap()` on a missing env var produces a panic message that says `NotPresent`, which is fine in a small CLI but useless when it shows up in a Kubernetes pod log at 3am.

## Parsing values into types

Env vars are strings. Almost everything you want is not. The cleanest pattern uses `FromStr`:

```rust
use std::env;
use std::str::FromStr;

fn parse_env<T: FromStr>(key: &str) -> Result<T, String>
where
    T::Err: std::fmt::Display,
{
    let raw = env::var(key).map_err(|e| format!("{key}: {e}"))?;
    raw.parse::<T>().map_err(|e| format!("{key}: {e}"))
}

fn main() -> Result<(), String> {
    let port: u16 = parse_env("PORT")?;
    let workers: usize = parse_env("WORKERS")?;
    let debug: bool = parse_env("DEBUG")?;
    println!("port={port} workers={workers} debug={debug}");
    Ok(())
}
```

This works for any type that implements `FromStr`: integers, floats, `bool`, `IpAddr`, `SocketAddr`, `PathBuf`, `Duration` (via humantime), `Url` (via the `url` crate). For collections, you have to define a separator yourself. The convention is comma-separated:

```rust
fn parse_list(key: &str) -> Result<Vec<String>, String> {
    Ok(env::var(key)
        .map_err(|e| format!("{key}: {e}"))?
        .split(',')
        .map(|s| s.trim().to_string())
        .filter(|s| !s.is_empty())
        .collect())
}
```

The empty-string filter matters because `"a,,b".split(',')` gives `["a", "", "b"]` and you almost never want the empty entry.

For booleans, `bool::from_str` only accepts the literal strings `"true"` and `"false"`. If you want `1`, `yes`, `on`, `enabled` to all work (and you do, because users will type all of these), parse manually:

```rust
fn parse_bool(s: &str) -> Result<bool, String> {
    match s.trim().to_ascii_lowercase().as_str() {
        "1" | "true" | "yes" | "on" | "enabled" => Ok(true),
        "0" | "false" | "no" | "off" | "disabled" => Ok(false),
        other => Err(format!("invalid bool: {other:?}")),
    }
}
```

## `dotenvy` for local development

In production, env vars come from the orchestrator. In dev, you want them to come from a file you can edit without restarting your shell. That's what `.env` files solve, and the [`dotenvy`](https://crates.io/crates/dotenvy) crate is the maintained successor to the original `dotenv` (which has been unmaintained since 2020).

```toml
[dependencies]
dotenvy = "0.15"
```

```rust
fn main() {
    // Load .env into the process environment.
    // Only does anything in dev - in prod the file doesn't exist.
    dotenvy::dotenv().ok();

    let db = std::env::var("DATABASE_URL").expect("DATABASE_URL not set");
    println!("connecting to {db}");
}
```

Two important details. First, `dotenvy::dotenv()` returns a `Result` - the `.ok()` discards the error so it doesn't fail when there's no `.env` file in production. Second, by default it only sets variables that are *not already* in the environment. So a real env var beats a `.env` value, which is exactly what you want: prod overrides dev defaults.

Add `.env` to your `.gitignore` and ship a `.env.example` with the variable names but no values:

```
# .env.example
DATABASE_URL=postgres://localhost/myapp_dev
PORT=8080
LOG_LEVEL=debug
JWT_SECRET=replace-me
```

Under the hood `dotenvy` uses `env::set_var`, which since Rust 1.85 means the call itself is `unsafe` to call directly. The crate wraps it - but importantly, you should call `dotenvy::dotenv()` *before* you spawn any threads, including the tokio runtime. This is one of the few times the order of statements at the top of `main` actually matters.

```rust
fn main() {
    dotenvy::dotenv().ok();  // before tokio
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async_main());
}
```

If you're using `#[tokio::main]`, the runtime is built before your function body runs, but the threads inside the runtime aren't spawned until you `await` something - so calling `dotenvy::dotenv()` as the first line of an async main is still safe in practice. Safer is to load env vars in a synchronous `fn main()` and then enter the runtime manually.

## Type-safe config with `config` and `serde`

Once you have more than three or four variables, writing `parse_env::<T>` calls one by one gets tedious. The standard solution is a typed struct that serde fills in from the environment.

The [`config`](https://crates.io/crates/config) crate (around 0.14 at time of writing) supports layered config: defaults, then a file, then env vars, with later layers overriding earlier ones.

```toml
[dependencies]
config = "0.14"
serde = { version = "1", features = ["derive"] }
```

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Settings {
    database_url: String,
    port: u16,
    workers: usize,
    log_level: String,
}

impl Settings {
    fn load() -> Result<Self, config::ConfigError> {
        config::Config::builder()
            // baked-in defaults
            .set_default("port", 8080)?
            .set_default("workers", 4)?
            .set_default("log_level", "info")?
            // optional config file
            .add_source(config::File::with_name("config").required(false))
            // env vars override everything, prefixed with APP_
            .add_source(config::Environment::with_prefix("APP").separator("__"))
            .build()?
            .try_deserialize()
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    dotenvy::dotenv().ok();
    let settings = Settings::load()?;
    println!("{settings:?}");
    Ok(())
}
```

A few things this gives you for free:

- **Prefix isolation.** `APP_DATABASE_URL` becomes `database_url`. Other env vars on the system don't pollute your config.
- **Nested fields.** `APP_DATABASE__URL` (note the double underscore separator) maps to `database.url` if you have a nested `Database` struct. This is how you encode structure into flat env vars.
- **Type-driven parsing.** serde converts `"8080"` to `u16`, `"true"` to `bool`, and gives you a useful error message if it can't.
- **Defaults.** `set_default` runs before any source, so optional fields don't need `Option<T>`.
- **One error path.** Missing required fields, invalid types, malformed files - all come out of `try_deserialize()` as a `ConfigError` with a path showing exactly what failed.

For lighter use cases there's [`envy`](https://crates.io/crates/envy), which is just env-to-struct without the layering. And [`figment`](https://crates.io/crates/figment) (used by Rocket) is the most flexible of the three, with profile support and metadata-rich errors. Pick the simplest one that fits.

## Validate at startup, not in handlers

The single most important rule of configuration: read and validate everything at process start. If a value is wrong, panic *immediately* with a clear error. Never lazy-load.

```rust
fn main() {
    dotenvy::dotenv().ok();
    let settings = match Settings::load() {
        Ok(s) => s,
        Err(e) => {
            eprintln!("config error: {e}");
            std::process::exit(1);
        }
    };
    settings.validate().unwrap_or_else(|e| {
        eprintln!("invalid config: {e}");
        std::process::exit(1);
    });

    run(settings);
}
```

Custom validation goes in a method:

```rust
impl Settings {
    fn validate(&self) -> Result<(), String> {
        if !(1..=65535).contains(&self.port) {
            return Err(format!("port {} out of range", self.port));
        }
        if self.workers == 0 {
            return Err("workers must be > 0".into());
        }
        if !self.database_url.starts_with("postgres://") {
            return Err("database_url must be a postgres URL".into());
        }
        Ok(())
    }
}
```

The reason for fail-fast is operational. Kubernetes, systemd, and every process supervisor handle "process exited with non-zero status at startup" perfectly - they restart, they alert, they roll back deployments. They handle "process is running but every request returns 500 because the API key was wrong" terribly. You want config errors to look like crashes, not bugs.

The [`validator`](https://crates.io/crates/validator) crate adds derive macros if your validation rules get complex:

```rust
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
struct Settings {
    #[validate(url)]
    database_url: String,
    #[validate(range(min = 1, max = 65535))]
    port: u16,
}
```

## Required, optional, default

Three patterns, three serde attributes:

```rust
#[derive(Deserialize)]
struct Settings {
    // Required: serde fails to deserialize if missing.
    database_url: String,

    // Optional: None if missing.
    sentry_dsn: Option<String>,

    // Default: uses Default::default if missing.
    #[serde(default = "default_port")]
    port: u16,

    // Default with literal value via wrapper struct or constant function.
    #[serde(default)]
    debug: bool,  // false
}

fn default_port() -> u16 { 8080 }
```

Use `Option<T>` for "this feature is off if not configured" (Sentry, Datadog, an optional cache). Use `#[serde(default)]` for "there's a sensible fallback." Use no attribute for "the app cannot start without this."

## Secrets: don't log them

The default `Debug` derive prints every field. So `dbg!(&settings)` cheerfully writes your database password and JWT secret to stderr, where it ends up in logs, in incident tickets, in Slack screenshots. This has burned every team I've ever worked with at least once.

The fix is the [`secrecy`](https://crates.io/crates/secrecy) crate:

```toml
secrecy = { version = "0.10", features = ["serde"] }
```

```rust
use secrecy::{ExposeSecret, SecretString};

#[derive(Debug, Deserialize)]
struct Settings {
    database_url: String,
    jwt_secret: SecretString,
}

fn main() {
    let settings = Settings::load().unwrap();
    println!("{settings:?}");
    // jwt_secret prints as `Secret([REDACTED alloc::string::String])`

    // Real access requires explicit unwrap:
    let secret: &str = settings.jwt_secret.expose_secret();
    sign_token(secret);
}
```

`SecretString` wraps a `String` and gives it a `Debug` impl that prints `[REDACTED]` instead of the value. The original is only accessible via `expose_secret()`, which is grep-able when you audit who's reading what. It also zeroes the memory on drop, which guards against leaks via core dumps.

If you don't want a dependency, write the same impl by hand:

```rust
struct Secret(String);

impl std::fmt::Debug for Secret {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str("[REDACTED]")
    }
}
```

The general rule: any field whose name contains `secret`, `password`, `token`, `key`, `dsn`, or `credential` should not be `Debug`-printable in plaintext. It costs you a one-line newtype.

A sneaky related issue: error messages. If your config layer logs `Err(format!("invalid value for JWT_SECRET: {raw}"))`, you've leaked the secret into the error message. Either don't include the value, or hash/truncate it.

## Testing with temporary env vars

Tests need to set env vars sometimes. Tests run in parallel by default. Env vars are process-global. This is a mess.

The [`temp-env`](https://crates.io/crates/temp-env) crate handles this with a guard pattern:

```rust
#[test]
fn loads_with_custom_port() {
    temp_env::with_var("APP_PORT", Some("9000"), || {
        let settings = Settings::load().unwrap();
        assert_eq!(settings.port, 9000);
    });
}
```

The closure runs with the variable set, and `temp_env` restores the previous value (or removes it) afterwards. Internally it holds a mutex so two `with_var` calls don't race.

Because of the `unsafe` `set_var` change in Rust 1.85, modern versions of `temp-env` wrap the unsafe call internally. If you're on an older Rust toolchain you don't need to do anything special; on newer ones the crate handles it for you.

For tests that need many vars, `with_vars` takes a slice:

```rust
temp_env::with_vars(
    [
        ("APP_PORT", Some("9000")),
        ("APP_DATABASE_URL", Some("postgres://test")),
    ],
    || { /* ... */ },
);
```

Avoid `serial_test` for this. It serializes the whole test binary, which is much slower than scoping env changes per test.

## Platform differences

Three things bite you when you ship the same binary across Linux, macOS, and Windows:

**1. Variable names are case-sensitive on Unix, case-insensitive on Windows.** On Windows, `Path` and `PATH` and `path` all refer to the same variable. On Linux they're different. The Rust stdlib does *not* paper this over - `env::var("path")` on Windows succeeds, on Linux fails. If you care about portability, normalize to uppercase yourself or use the `config` crate which handles this for you.

**2. PATH separator.** Colon on Unix, semicolon on Windows. `std::env::split_paths` and `std::env::join_paths` exist for exactly this. Use them.

**3. Maximum env block size.** Linux gives you ~128 KB total for argv+envp combined (`ARG_MAX`). Windows has a 32,767-character limit per variable. If you find yourself stuffing a JSON blob into an env var, you've outgrown env vars - go to a file.

## Env vars vs config files

When does it make sense to use a config file instead of (or alongside) env vars?

| Concern | Env vars | Config file (TOML/YAML) |
|---|---|---|
| Secrets | Good (managed by orchestrator) | Bad (must be templated/encrypted) |
| Lists, maps, nested structs | Awkward | Native |
| Per-environment overrides | Trivial | Requires file selection logic |
| Hot-reload | Impossible without process restart | Possible with `notify` crate |
| Auditing what's set | `env | grep APP_` | `cat config.toml` |
| Length limits | ~128 KB total | None practical |

In practice the answer is "both": a config file for structure (allowed origins, feature flag tree, per-tenant overrides) plus env vars for secrets and per-environment values. The `config` crate's layering is built for this. Don't put secrets in a TOML file you check into the repo.

YAML adds nothing over TOML for Rust apps and brings significant attack surface ([YAML deserialization vulnerabilities](https://github.com/dtolnay/serde-yaml#alternatives) are a recurring theme). If you don't have a strong reason for YAML, use TOML.

## What to take away

The setup that scales from a 100-line CLI to a real service:

1. One `Settings` struct deriving `Deserialize` and `Debug`.
2. Loaded once at startup via the `config` crate, with defaults + optional file + env override.
3. `dotenvy::dotenv().ok()` as the first line of `main` for local dev.
4. Validation method called immediately after load; non-zero exit on error.
5. `SecretString` (or a hand-rolled equivalent) for any secret field.
6. `temp_env` in tests instead of mutating the global environment directly.
7. Prefixed env vars (`APP_*`) so you don't collide with the OS or other tools.

None of this is exciting. All of it pays off the first time someone deploys with a typo in `DATABBASE_URL` and the app crashes at startup with a one-line error instead of silently serving 500s for forty minutes.
