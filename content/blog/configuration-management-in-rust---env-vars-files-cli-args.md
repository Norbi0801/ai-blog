+++
title = "Configuration management in Rust - env vars, files, CLI args"
date = 2025-07-27
description = "A layered config system in Rust using config, dotenvy, and clap - from typed structs and defaults through env vars and CLI overrides, with secret handling and startup validation."

[taxonomies]
tags = ["rust", "configuration", "devops", "twelve-factor"]
+++

Every Rust project starts the same way. You hardcode a port number. Then a database URL. Then an API key. Then someone opens a PR with "please don't hardcode this" and you reach for `std::env::var("PORT").unwrap()`. That works until you have 30 env vars, half of them are optional, three of them are secrets, and you discover the production deploy crashed because someone typo'd `DATABSE_URL`.

Configuration is one of those problems that seems trivial until it isn't. The difference between a service that's easy to deploy and one that's a nightmare often comes down to how it handles config. Good config management means: one typed struct, validated once at startup, assembled from multiple sources with clear precedence, and secrets that never touch version control.

This post builds that system from scratch.

<!-- more -->

## The layered config model

The [12-factor app methodology](https://12factor.net/config) says config should live in environment variables, separate from code. That's good advice, but it's incomplete. Real services need a layered approach where multiple sources merge with clear precedence:

```
defaults (in code) -> config file -> environment variables -> CLI arguments
```

Each layer overrides the previous one. Defaults give you sane values for development. A config file (`config.toml`) lets you set environment-specific values without touching env vars. Env vars override the file for deployment flexibility. And CLI arguments override everything - useful for one-off runs, debugging, or testing a specific value without changing any file.

This isn't a new idea. Kubernetes, systemd, Docker Compose - they all follow the same pattern. The question is how to implement it cleanly in Rust without ending up with string-typed spaghetti spread across your codebase.

## Step 1: Define the config struct

Before choosing any crate, define what your config looks like. A single, typed struct:

```rust
use std::net::SocketAddr;

#[derive(Debug, Clone)]
pub struct AppConfig {
    pub host: String,
    pub port: u16,
    pub database_url: String,
    pub database_max_connections: u32,
    pub redis_url: Option<String>,
    pub log_level: String,
    pub jwt_secret: String,
    pub cors_origins: Vec<String>,
    pub request_timeout_secs: u64,
}

impl AppConfig {
    pub fn socket_addr(&self) -> SocketAddr {
        format!("{}:{}", self.host, self.port)
            .parse()
            .expect("invalid host:port in config")
    }
}
```

This is your single source of truth. Every part of your application reads from this struct - never from `std::env::var` directly. If you've read the [dependency injection post](/blog/dependency-injection-patterns-without-a-framework), you saw `AppConfig` passed into services via constructors. This post is about how that struct gets built.

The key properties of a good config struct:

- **Typed fields** - `port` is a `u16`, not a `String`. If someone sets `PORT=banana`, you find out at startup, not when the server tries to bind.
- **Optional fields use `Option<T>`** - Redis is optional. If `redis_url` is `None`, features that need it gracefully degrade. No panics, no magic empty strings.
- **No secrets in Debug** - we'll fix this shortly with the `secrecy` crate.
- **Flat or shallow nesting** - deep config hierarchies are a sign you're configuring too much. If your config struct has 50 fields, you probably have 5 services that should each own their own config.

## Step 2: Defaults in code

Defaults are the base layer. They should make `cargo run` work without any config file or env vars - at least in development.

```rust
impl Default for AppConfig {
    fn default() -> Self {
        Self {
            host: "127.0.0.1".to_string(),
            port: 3000,
            database_url: "sqlite://dev.db".to_string(),
            database_max_connections: 5,
            redis_url: None,
            log_level: "info".to_string(),
            jwt_secret: "dev-secret-do-not-use-in-prod".to_string(),
            cors_origins: vec!["http://localhost:3000".to_string()],
            request_timeout_secs: 30,
        }
    }
}
```

Two rules for defaults:

1. **Dev-friendly** - a developer should be able to clone the repo and `cargo run` immediately. SQLite as default DB, localhost as default host, permissive CORS.
2. **Prod-unsafe** - the `jwt_secret` default is an obvious placeholder. Your validation step (later in this post) will reject it in production. This is intentional - you want defaults that work in dev but scream "configure me" in prod.

## Step 3: Config files with TOML

For projects that need a config file, TOML is the natural choice in the Rust ecosystem. Your `Cargo.toml` is already TOML. The `toml` crate handles parsing, and `serde` handles deserialization.

```toml
# config.toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
toml = "0.8"
```

Add serde derives to your config struct:

```rust
use serde::Deserialize;

#[derive(Debug, Clone, Deserialize)]
pub struct FileConfig {
    pub host: Option<String>,
    pub port: Option<u16>,
    pub database_url: Option<String>,
    pub database_max_connections: Option<u32>,
    pub redis_url: Option<String>,
    pub log_level: Option<String>,
    pub jwt_secret: Option<String>,
    pub cors_origins: Option<Vec<String>>,
    pub request_timeout_secs: Option<u64>,
}
```

Notice: every field is `Option<T>`. The config file doesn't have to specify everything. Missing fields fall back to defaults. This is important - a partial config file that only overrides what it needs is much more useful than one that has to repeat every default.

```rust
impl AppConfig {
    pub fn with_file(mut self, path: &str) -> Self {
        let contents = match std::fs::read_to_string(path) {
            Ok(c) => c,
            Err(_) => return self, // file not found = use defaults
        };

        let file_config: FileConfig = toml::from_str(&contents)
            .expect("invalid config file syntax");

        if let Some(v) = file_config.host { self.host = v; }
        if let Some(v) = file_config.port { self.port = v; }
        if let Some(v) = file_config.database_url { self.database_url = v; }
        if let Some(v) = file_config.database_max_connections {
            self.database_max_connections = v;
        }
        if let Some(v) = file_config.redis_url { self.redis_url = Some(v); }
        if let Some(v) = file_config.log_level { self.log_level = v; }
        if let Some(v) = file_config.jwt_secret { self.jwt_secret = v; }
        if let Some(v) = file_config.cors_origins { self.cors_origins = v; }
        if let Some(v) = file_config.request_timeout_secs {
            self.request_timeout_secs = v;
        }

        self
    }
}
```

An example `config.toml`:

```toml
host = "0.0.0.0"
port = 8080
database_url = "postgres://localhost/myapp"
database_max_connections = 20
log_level = "debug"

cors_origins = [
    "https://myapp.com",
    "https://staging.myapp.com",
]
```

The manual field-by-field merge is verbose. We'll replace it with the `config` crate later. But this explicit version is useful for understanding what's happening - each layer selectively overrides the previous one.

## Step 4: Environment variables with dotenvy

Environment variables are the primary config mechanism in containerized environments. Kubernetes ConfigMaps, Docker `--env-file`, systemd `Environment=` directives - they all inject config as env vars.

[`dotenvy`](https://crates.io/crates/dotenvy) (v0.15) is the maintained successor to the original `dotenv` crate, which had a [security advisory](https://rustsec.org/advisories/RUSTSEC-2021-0141.html) and stopped receiving updates. It loads a `.env` file into the process environment during development, while in production the real env vars take precedence.

```toml
[dependencies]
dotenvy = "0.15"
```

```rust
fn load_dotenv() {
    // In development, load .env file. In production, this is a no-op
    // if the file doesn't exist.
    match dotenvy::dotenv() {
        Ok(path) => {
            tracing::debug!("loaded env from {}", path.display());
        }
        Err(dotenvy::Error::Io(ref e))
            if e.kind() == std::io::ErrorKind::NotFound =>
        {
            // No .env file - that's fine in production
            tracing::debug!("no .env file found, using system env vars");
        }
        Err(e) => {
            // Syntax error in .env file - fail fast
            panic!("failed to load .env file: {e}");
        }
    }
}
```

This is more nuanced than a bare `dotenvy::dotenv().ok()` that you see in tutorials. A missing `.env` is fine. A malformed `.env` is a bug that should crash the process immediately - you don't want to silently run with a half-parsed config.

Your `.env` file:

```bash
# .env - DO NOT COMMIT THIS FILE
DATABASE_URL=postgres://user:password@localhost:5432/myapp
JWT_SECRET=supersecretkey123
REDIS_URL=redis://localhost:6379
```

And the merge step:

```rust
impl AppConfig {
    pub fn with_env(mut self) -> Self {
        if let Ok(v) = std::env::var("HOST") { self.host = v; }
        if let Ok(v) = std::env::var("PORT") {
            self.port = v.parse().expect("PORT must be a valid u16");
        }
        if let Ok(v) = std::env::var("DATABASE_URL") { self.database_url = v; }
        if let Ok(v) = std::env::var("DATABASE_MAX_CONNECTIONS") {
            self.database_max_connections = v.parse()
                .expect("DATABASE_MAX_CONNECTIONS must be a valid u32");
        }
        if let Ok(v) = std::env::var("REDIS_URL") { self.redis_url = Some(v); }
        if let Ok(v) = std::env::var("LOG_LEVEL") { self.log_level = v; }
        if let Ok(v) = std::env::var("JWT_SECRET") { self.jwt_secret = v; }
        if let Ok(v) = std::env::var("CORS_ORIGINS") {
            self.cors_origins = v.split(',').map(|s| s.trim().to_string()).collect();
        }
        if let Ok(v) = std::env::var("REQUEST_TIMEOUT_SECS") {
            self.request_timeout_secs = v.parse()
                .expect("REQUEST_TIMEOUT_SECS must be a valid u64");
        }

        self
    }
}
```

The `.expect()` calls are intentional. If `PORT` is set but not a number, that's a hard error. You want that to blow up at startup, not silently fall back to the default. The 12-factor principle here: if someone explicitly sets an env var, respect it literally. Don't silently ignore malformed values.

### The .env rule

Your `.gitignore` must contain:

```gitignore
.env
.env.local
.env.*.local
```

Commit a `.env.example` with placeholder values so new developers know what to set:

```bash
# .env.example - copy to .env and fill in real values
DATABASE_URL=postgres://user:password@localhost:5432/myapp
JWT_SECRET=change-me
REDIS_URL=redis://localhost:6379
```

The [twelve-factor litmus test](https://12factor.net/config): could you open-source this repo right now without exposing any credentials? If your `.env` is committed, the answer is no.

## Step 5: CLI arguments with clap

The final override layer. CLI args are useful for dev-time overrides (`cargo run -- --port 9090`), debugging specific config values, and scripts that need to vary one parameter per invocation.

[`clap`](https://crates.io/crates/clap) (v4.5) with the derive API makes this clean:

```toml
[dependencies]
clap = { version = "4.5", features = ["derive"] }
```

```rust
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "myapp", about = "My application")]
pub struct CliArgs {
    /// Host to bind to
    #[arg(long, env = "HOST")]
    pub host: Option<String>,

    /// Port to listen on
    #[arg(short, long, env = "PORT")]
    pub port: Option<u16>,

    /// Database connection URL
    #[arg(long, env = "DATABASE_URL")]
    pub database_url: Option<String>,

    /// Maximum database connections
    #[arg(long, env = "DATABASE_MAX_CONNECTIONS")]
    pub database_max_connections: Option<u32>,

    /// Redis connection URL
    #[arg(long, env = "REDIS_URL")]
    pub redis_url: Option<String>,

    /// Log level (trace, debug, info, warn, error)
    #[arg(long, env = "LOG_LEVEL")]
    pub log_level: Option<String>,

    /// JWT signing secret
    #[arg(long, env = "JWT_SECRET")]
    pub jwt_secret: Option<String>,

    /// Path to config file
    #[arg(short, long, default_value = "config.toml")]
    pub config: String,

    /// Request timeout in seconds
    #[arg(long, env = "REQUEST_TIMEOUT_SECS")]
    pub request_timeout_secs: Option<u64>,
}
```

Notice the `env = "PORT"` attributes on each field. Clap can read from env vars directly, which gives you a combined CLI + env layer. But here's the subtlety - if you're also using `dotenvy` and manual env var reading, you'll double-read. I'll show the clean way to combine these in the next section.

The merge:

```rust
impl AppConfig {
    pub fn with_cli(mut self, args: &CliArgs) -> Self {
        if let Some(ref v) = args.host { self.host = v.clone(); }
        if let Some(v) = args.port { self.port = v; }
        if let Some(ref v) = args.database_url { self.database_url = v.clone(); }
        if let Some(v) = args.database_max_connections {
            self.database_max_connections = v;
        }
        if let Some(ref v) = args.redis_url { self.redis_url = Some(v.clone()); }
        if let Some(ref v) = args.log_level { self.log_level = v.clone(); }
        if let Some(ref v) = args.jwt_secret { self.jwt_secret = v.clone(); }
        if let Some(v) = args.request_timeout_secs {
            self.request_timeout_secs = v;
        }

        self
    }
}
```

## Putting the manual layers together

The full assembly:

```rust
impl AppConfig {
    pub fn load() -> Self {
        // Load .env before parsing CLI (so env vars are available to clap)
        load_dotenv();

        let args = CliArgs::parse();

        let config = AppConfig::default()      // 1. defaults
            .with_file(&args.config)            // 2. config file
            .with_env()                         // 3. env vars
            .with_cli(&args);                   // 4. CLI args

        config.validate();
        config
    }
}
```

Four lines, clear precedence, and a validation step at the end. This is the pattern.

But the manual approach has problems. Every time you add a field, you have to update four places: the struct, `Default`, `FileConfig`, `with_env`, and `with_cli`. That's five touch points per field. For a config struct with 15 fields, that's real maintenance cost.

## The config crate: doing it properly

The [`config`](https://crates.io/crates/config) crate (v0.15, ~3.1k GitHub stars, 58M+ downloads) exists to solve exactly this problem. It handles layered sources, type conversion, and serde deserialization in one pipeline.

```toml
[dependencies]
config = { version = "0.15", features = ["toml"] }
serde = { version = "1.0", features = ["derive"] }
clap = { version = "4.5", features = ["derive"] }
dotenvy = "0.15"
secrecy = "0.10"
```

Redefine your config struct with serde support:

```rust
use serde::Deserialize;
use secrecy::{ExposeSecret, SecretString};

#[derive(Debug, Deserialize, Clone)]
pub struct AppConfig {
    #[serde(default = "default_host")]
    pub host: String,

    #[serde(default = "default_port")]
    pub port: u16,

    pub database_url: String,

    #[serde(default = "default_max_conn")]
    pub database_max_connections: u32,

    pub redis_url: Option<String>,

    #[serde(default = "default_log_level")]
    pub log_level: String,

    pub jwt_secret: SecretString,

    #[serde(default = "default_cors_origins")]
    pub cors_origins: Vec<String>,

    #[serde(default = "default_timeout")]
    pub request_timeout_secs: u64,
}

fn default_host() -> String { "127.0.0.1".to_string() }
fn default_port() -> u16 { 3000 }
fn default_max_conn() -> u32 { 5 }
fn default_log_level() -> String { "info".to_string() }
fn default_cors_origins() -> Vec<String> {
    vec!["http://localhost:3000".to_string()]
}
fn default_timeout() -> u64 { 30 }
```

Three things changed from the manual version:

1. **`SecretString` for jwt_secret** - the [`secrecy`](https://crates.io/crates/secrecy) crate (v0.10) wraps sensitive values. Its `Debug` implementation prints `[REDACTED]` instead of the actual value. It zeroizes the memory on drop. You access the value with `.expose_secret()`, which makes every use of the secret explicit and grep-able. If you're curious about the monitoring side of this, the [monitoring post](/blog/monitoring-rust-applications-in-production) showed how structured logging can accidentally leak sensitive data - `SecretString` prevents that by design.
2. **`#[serde(default = "...")]`** - defaults are defined once, in the struct. No separate `Default` impl, no `FileConfig` mirror struct.
3. **Required fields have no default** - `database_url` and `jwt_secret` must come from somewhere. If no source provides them, deserialization fails at startup.

Now the layered builder:

```rust
use config::{Config, Environment, File};

impl AppConfig {
    pub fn load() -> Result<Self, config::ConfigError> {
        load_dotenv();

        let args = CliArgs::parse();

        let mut builder = Config::builder()
            // Layer 1: config file (optional)
            .add_source(
                File::with_name(&args.config)
                    .required(false)
            )
            // Layer 2: environment variables
            // APP_DATABASE_URL -> database_url
            .add_source(
                Environment::with_prefix("APP")
                    .separator("__")
                    .try_parsing(true)
            );

        // Layer 3: CLI overrides
        if let Some(ref host) = args.host {
            builder = builder.set_override("host", host.clone())?;
        }
        if let Some(port) = args.port {
            builder = builder.set_override("port", port as i64)?;
        }
        if let Some(ref db) = args.database_url {
            builder = builder.set_override("database_url", db.clone())?;
        }
        if let Some(max_conn) = args.database_max_connections {
            builder = builder.set_override(
                "database_max_connections", max_conn as i64
            )?;
        }
        if let Some(ref redis) = args.redis_url {
            builder = builder.set_override("redis_url", redis.clone())?;
        }
        if let Some(ref level) = args.log_level {
            builder = builder.set_override("log_level", level.clone())?;
        }
        if let Some(ref secret) = args.jwt_secret {
            builder = builder.set_override("jwt_secret", secret.clone())?;
        }
        if let Some(timeout) = args.request_timeout_secs {
            builder = builder.set_override(
                "request_timeout_secs", timeout as i64
            )?;
        }

        builder.build()?.try_deserialize()
    }
}
```

The `Environment::with_prefix("APP")` line means env vars are prefixed: `APP_DATABASE_URL`, `APP_PORT`, `APP_JWT_SECRET`. The prefix prevents collisions with system env vars. `separator("__")` enables nested config - `APP_DATABASE__MAX_CONNECTIONS` maps to `database.max_connections` if you used nested structs. `try_parsing(true)` tells the config crate to attempt parsing string env vars into their target types (so `APP_PORT=3000` becomes a `u16`, not a `String`).

The `File::with_name("config")` call is smart about extensions - it will look for `config.toml`, `config.yaml`, `config.json`, etc., and pick the right parser. Setting `.required(false)` means the app still starts without a config file.

## Under the hood: how config merges sources

The `config` crate internally represents all configuration as a tree of `Value` nodes. When you call `add_source()`, it reads the entire source into a `Value` tree, then deep-merges it with the existing tree. Later sources overwrite earlier values at the leaf level. For maps (nested structs), it merges recursively - a later source can override one nested field without wiping the others.

The final `try_deserialize()` call takes the merged `Value` tree and runs it through serde, producing your typed struct. If any required field is missing or a type conversion fails, you get a `ConfigError` with a description of what went wrong and (since v0.14) which source the problematic value came from.

This is why the `config` crate is worth the dependency. The alternative - manually merging `Option<T>` fields across four layers - scales linearly with the number of config fields and sources. The `config` crate's approach scales better: add a new field to the struct, add a `#[serde(default)]` if needed, and every source automatically picks it up.

## Validation at startup, not at use

This is the single most important principle in configuration management. Every config value should be validated the moment the application starts. If something is wrong, crash immediately with a clear error message. Do not wait until the value is used.

```rust
impl AppConfig {
    pub fn validate(&self) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();

        // Port must be in valid range
        if self.port == 0 {
            errors.push("port must be > 0".to_string());
        }

        // Database URL must be a real URL, not the dev default in prod
        if self.database_url.is_empty() {
            errors.push("database_url cannot be empty".to_string());
        }

        // JWT secret must not be the dev placeholder in production
        if std::env::var("APP_ENV").unwrap_or_default() == "production" {
            let secret = self.jwt_secret.expose_secret();
            if secret.contains("dev-secret") || secret.len() < 32 {
                errors.push(
                    "jwt_secret must be at least 32 chars in production"
                        .to_string()
                );
            }

            if self.database_url.starts_with("sqlite://") {
                errors.push(
                    "sqlite is not allowed in production".to_string()
                );
            }
        }

        // Timeout must be reasonable
        if self.request_timeout_secs == 0 || self.request_timeout_secs > 300 {
            errors.push(
                "request_timeout_secs must be between 1 and 300".to_string()
            );
        }

        // Validate CORS origins are valid URLs
        for origin in &self.cors_origins {
            if !origin.starts_with("http://") && !origin.starts_with("https://") {
                errors.push(
                    format!("invalid cors origin: {origin}")
                );
            }
        }

        // Validate max connections
        if self.database_max_connections == 0
            || self.database_max_connections > 1000
        {
            errors.push(
                "database_max_connections must be between 1 and 1000"
                    .to_string()
            );
        }

        if errors.is_empty() {
            Ok(())
        } else {
            Err(errors)
        }
    }
}
```

Why is this so important? Because the alternative is discovering at 3 AM that `jwt_secret` is empty when the first user tries to log in. Or that `database_url` points to the wrong host, but only the background job that runs every 6 hours actually connects there. Fail-fast config validation turns a production incident into a deploy failure - and deploy failures are infinitely easier to debug.

The validation function collects all errors, not just the first one. If you have three misconfigurations, you want to see all three at once, not fix them one at a time with three deploy cycles.

Use it in `main`:

```rust
#[tokio::main]
async fn main() {
    let config = AppConfig::load().unwrap_or_else(|e| {
        eprintln!("failed to load configuration: {e}");
        std::process::exit(1);
    });

    if let Err(errors) = config.validate() {
        eprintln!("configuration validation failed:");
        for error in &errors {
            eprintln!("  - {error}");
        }
        std::process::exit(1);
    }

    tracing::info!(
        host = %config.host,
        port = config.port,
        db_max_conn = config.database_max_connections,
        "configuration loaded"
    );

    // pass config to your services...
}
```

Notice the `tracing::info!` line logs config values but not secrets. `jwt_secret` is a `SecretString`, so even if you accidentally included it in a debug log, it would show as `[REDACTED]`.

## Secret management beyond .env

The `.env` + `SecretString` combo works for most projects. But there's a spectrum of secret management, and knowing where you are on it matters.

**Level 1: Environment variables** - the baseline. Secrets are env vars, loaded from `.env` in dev and injected by the platform in production (Kubernetes secrets, ECS task definitions, Fly.io secrets). This is what 90% of services need.

**Level 2: SecretString in memory** - what we added above. Prevents accidental logging, zeroizes on drop. Cheap to add, prevents a real class of bugs. Use it for every secret field.

**Level 3: Cloud secret managers** - AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault. The application fetches secrets at startup from the secret store instead of reading env vars. Useful when you need: rotation without redeployment, audit trails of secret access, centralized management across many services.

```rust
// Pseudocode for cloud secret manager integration
async fn load_secrets(config: &mut AppConfig) -> Result<(), Box<dyn Error>> {
    let client = aws_sdk_secretsmanager::Client::new(&aws_config).await;

    let secret = client
        .get_secret_value()
        .secret_id("myapp/production/jwt-secret")
        .send()
        .await?;

    if let Some(value) = secret.secret_string() {
        config.jwt_secret = SecretString::from(value.to_string());
    }

    Ok(())
}
```

For most teams, Level 1 + Level 2 is sufficient. Jump to Level 3 when you have compliance requirements, many services sharing secrets, or need automated rotation.

### Things that should never be in your config struct

- **Derived values** - if `socket_addr` is always `host:port`, compute it with a method, don't store it as a separate field
- **Feature flags** - use a proper feature flag system, not config
- **Business logic constants** - prices, thresholds, rate limits belong in a database or dedicated system, not in config that requires a redeploy to change
- **Anything that changes per-request** - config is immutable after startup

## Nested config with the config crate

As your application grows, a flat struct becomes unwieldy. Group related settings:

```rust
#[derive(Debug, Deserialize, Clone)]
pub struct AppConfig {
    #[serde(default)]
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub auth: AuthConfig,
    #[serde(default)]
    pub cors: CorsConfig,
}

#[derive(Debug, Deserialize, Clone)]
pub struct ServerConfig {
    #[serde(default = "default_host")]
    pub host: String,
    #[serde(default = "default_port")]
    pub port: u16,
    #[serde(default = "default_timeout")]
    pub request_timeout_secs: u64,
}

#[derive(Debug, Deserialize, Clone)]
pub struct DatabaseConfig {
    pub url: String,
    #[serde(default = "default_max_conn")]
    pub max_connections: u32,
}

#[derive(Debug, Deserialize, Clone)]
pub struct AuthConfig {
    pub jwt_secret: SecretString,
}

#[derive(Debug, Deserialize, Clone)]
pub struct CorsConfig {
    #[serde(default = "default_cors_origins")]
    pub origins: Vec<String>,
}
```

The corresponding TOML file looks natural:

```toml
[server]
host = "0.0.0.0"
port = 8080

[database]
url = "postgres://localhost/myapp"
max_connections = 20

[cors]
origins = ["https://myapp.com"]
```

And env vars use the double-underscore separator to express nesting:

```bash
APP_SERVER__PORT=9090
APP_DATABASE__URL=postgres://prod-host/myapp
APP_DATABASE__MAX_CONNECTIONS=50
APP_AUTH__JWT_SECRET=your-production-secret
```

The `config` crate's `Environment::with_prefix("APP").separator("__")` maps these automatically. `APP_DATABASE__URL` becomes `database.url` in the config tree, which serde deserializes into `config.database.url`.

## Environment-specific config files

A common pattern: a base config, overridden by an environment-specific file.

```rust
let env = std::env::var("APP_ENV").unwrap_or_else(|_| "development".into());

let config = Config::builder()
    .add_source(File::with_name("config/default").required(false))
    .add_source(File::with_name(&format!("config/{env}")).required(false))
    .add_source(
        Environment::with_prefix("APP")
            .separator("__")
            .try_parsing(true)
    )
    .build()?
    .try_deserialize::<AppConfig>()?;
```

Your config directory:

```
config/
  default.toml      # shared defaults
  development.toml   # dev overrides (debug logging, localhost)
  production.toml    # prod overrides (stricter settings)
  test.toml          # test overrides (in-memory DB, fast timeouts)
```

This follows the 12-factor recommendation against "named environments" only loosely. The environment-specific files contain non-sensitive overrides (log levels, timeouts, feature toggles). Actual credentials still come from env vars. The file just sets the baseline for that environment.

## Alternative: figment

[Figment](https://crates.io/crates/figment) (v0.10, by Sergio Benitez of Rocket fame) takes a different approach. Where the `config` crate focuses on flexibility, Figment focuses on provenance - knowing exactly where each value came from.

```rust
use figment::{Figment, providers::{Format, Toml, Env, Serialized}};

let config: AppConfig = Figment::new()
    .merge(Serialized::defaults(AppConfig::default()))
    .merge(Toml::file("config.toml"))
    .merge(Env::prefixed("APP_").split("__"))
    .extract()?;
```

When something goes wrong, Figment tells you exactly where the bad value came from:

```
error: invalid type: found string "banana", expected u16
 --> `APP_SERVER__PORT` environment variable
```

Compare with the `config` crate, which gives you the field path but not always the source. Figment's error messages are notably better for debugging misconfiguration in CI/CD pipelines where you can't easily inspect the environment.

Figment also has `merge` vs `join` semantics. `merge` replaces duplicate keys (later wins - the standard layering behavior). `join` only fills in missing keys (earlier wins). This is useful for "defaults that should not be overridden" scenarios, though those are rare.

Which one should you use? If you're building a library or framework that others configure, Figment's provenance tracking helps your users debug their config. If you're building a service, either works. I tend to reach for `config` because it's more established and the file format auto-detection is convenient, but Figment is a strong choice.

## The complete example

Here's the final, production-grade setup combining everything:

```toml
# Cargo.toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2021"

[dependencies]
config = { version = "0.15", features = ["toml"] }
serde = { version = "1.0", features = ["derive"] }
secrecy = { version = "0.10", features = ["serde"] }
clap = { version = "4.5", features = ["derive"] }
dotenvy = "0.15"
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }
```

```rust
// src/config.rs
use clap::Parser;
use config::{Config, Environment, File};
use secrecy::{ExposeSecret, SecretString};
use serde::Deserialize;

#[derive(Parser, Debug)]
#[command(name = "myapp")]
pub struct CliArgs {
    /// Path to config file
    #[arg(short, long, default_value = "config.toml")]
    pub config: String,

    /// Override server port
    #[arg(short, long)]
    pub port: Option<u16>,

    /// Override log level
    #[arg(long)]
    pub log_level: Option<String>,
}

#[derive(Debug, Deserialize, Clone)]
pub struct AppConfig {
    #[serde(default)]
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub auth: AuthConfig,
}

#[derive(Debug, Deserialize, Clone)]
pub struct ServerConfig {
    #[serde(default = "default_host")]
    pub host: String,
    #[serde(default = "default_port")]
    pub port: u16,
    #[serde(default = "default_timeout")]
    pub request_timeout_secs: u64,
    #[serde(default = "default_log_level")]
    pub log_level: String,
}

#[derive(Debug, Deserialize, Clone)]
pub struct DatabaseConfig {
    pub url: String,
    #[serde(default = "default_max_conn")]
    pub max_connections: u32,
}

#[derive(Clone)]
pub struct AuthConfig {
    pub jwt_secret: SecretString,
}

// Custom Debug to prevent secret leaking
impl std::fmt::Debug for AuthConfig {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("AuthConfig")
            .field("jwt_secret", &"[REDACTED]")
            .finish()
    }
}

// SecretString needs a custom Deserialize because the serde feature
// on secrecy 0.10 expects the "serde" feature flag
impl<'de> Deserialize<'de> for AuthConfig {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        #[derive(Deserialize)]
        struct Inner {
            jwt_secret: String,
        }
        let inner = Inner::deserialize(deserializer)?;
        Ok(AuthConfig {
            jwt_secret: SecretString::from(inner.jwt_secret),
        })
    }
}

fn default_host() -> String { "127.0.0.1".into() }
fn default_port() -> u16 { 3000 }
fn default_timeout() -> u64 { 30 }
fn default_log_level() -> String { "info".into() }
fn default_max_conn() -> u32 { 5 }

impl AppConfig {
    pub fn load() -> Result<Self, Box<dyn std::error::Error>> {
        // Load .env if present (dev only)
        load_dotenv();

        let args = CliArgs::parse();
        let env_name = std::env::var("APP_ENV")
            .unwrap_or_else(|_| "development".into());

        let mut builder = Config::builder()
            // Layer 1: base config file
            .add_source(
                File::with_name("config/default").required(false)
            )
            // Layer 2: environment-specific config file
            .add_source(
                File::with_name(&format!("config/{env_name}"))
                    .required(false)
            )
            // Layer 3: user-specified config file
            .add_source(
                File::with_name(&args.config).required(false)
            )
            // Layer 4: environment variables
            .add_source(
                Environment::with_prefix("APP")
                    .separator("__")
                    .try_parsing(true)
            );

        // Layer 5: CLI overrides (highest priority)
        if let Some(port) = args.port {
            builder = builder
                .set_override("server.port", port as i64)?;
        }
        if let Some(ref level) = args.log_level {
            builder = builder
                .set_override("server.log_level", level.as_str())?;
        }

        let config: AppConfig = builder.build()?.try_deserialize()?;

        config.validate()?;

        Ok(config)
    }

    fn validate(&self) -> Result<(), Box<dyn std::error::Error>> {
        let mut errors = Vec::new();

        if self.server.port == 0 {
            errors.push("server.port must be > 0");
        }

        if self.database.url.is_empty() {
            errors.push("database.url cannot be empty");
        }

        if self.database.max_connections == 0
            || self.database.max_connections > 500
        {
            errors.push("database.max_connections must be 1-500");
        }

        if self.server.request_timeout_secs == 0
            || self.server.request_timeout_secs > 300
        {
            errors.push("server.request_timeout_secs must be 1-300");
        }

        let env = std::env::var("APP_ENV").unwrap_or_default();
        if env == "production" {
            let secret = self.auth.jwt_secret.expose_secret();
            if secret.len() < 32 || secret.contains("dev-secret") {
                errors.push(
                    "auth.jwt_secret must be >= 32 chars in production"
                );
            }
            if self.database.url.starts_with("sqlite://") {
                errors.push("sqlite not allowed in production");
            }
        }

        if errors.is_empty() {
            Ok(())
        } else {
            let msg = errors.join("; ");
            Err(format!("config validation failed: {msg}").into())
        }
    }
}

fn load_dotenv() {
    match dotenvy::dotenv() {
        Ok(path) => {
            eprintln!("loaded .env from {}", path.display());
        }
        Err(dotenvy::Error::Io(ref e))
            if e.kind() == std::io::ErrorKind::NotFound =>
        {
            // No .env file, fine in production
        }
        Err(e) => {
            panic!("malformed .env file: {e}");
        }
    }
}
```

```rust
// src/main.rs
mod config;

use crate::config::AppConfig;

#[tokio::main]
async fn main() {
    let config = AppConfig::load().unwrap_or_else(|e| {
        eprintln!("fatal: {e}");
        std::process::exit(1);
    });

    // Initialize logging with the configured level
    tracing_subscriber::fmt()
        .with_env_filter(&config.server.log_level)
        .json()
        .init();

    tracing::info!(
        host = %config.server.host,
        port = config.server.port,
        db_max_conn = config.database.max_connections,
        env = %std::env::var("APP_ENV").unwrap_or_else(|_| "development".into()),
        "configuration loaded"
    );

    // Build your services using the config...
    // let db = DatabasePool::connect(
    //     &config.database.url,
    //     config.database.max_connections,
    // ).await.unwrap();
}
```

Run it:

```bash
# Development - uses .env + defaults
cargo run

# Override port for testing
cargo run -- --port 9090

# Production - all config from env vars
APP_ENV=production \
APP_SERVER__PORT=8080 \
APP_DATABASE__URL=postgres://prod:secret@db.internal/myapp \
APP_AUTH__JWT_SECRET=a]3kf9$mNp2xR7vB8qW5eT1yU6hJ4gL0 \
./myapp

# With a custom config file
cargo run -- --config /etc/myapp/config.toml --log-level debug
```

## Testing config loading

Config loading is worth testing. You don't want to discover a deserialization bug in production.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_default_config_file_parsing() {
        let toml_str = r#"
            [server]
            host = "0.0.0.0"
            port = 8080

            [database]
            url = "postgres://localhost/test"
            max_connections = 10

            [auth]
            jwt_secret = "test-secret-that-is-long-enough-for-validation"
        "#;

        let config: AppConfig = Config::builder()
            .add_source(config::File::from_str(
                toml_str,
                config::FileFormat::Toml,
            ))
            .build()
            .unwrap()
            .try_deserialize()
            .unwrap();

        assert_eq!(config.server.port, 8080);
        assert_eq!(config.database.max_connections, 10);
    }

    #[test]
    fn test_env_override() {
        // Temporarily set env var
        std::env::set_var("APP_SERVER__PORT", "9999");

        let config: AppConfig = Config::builder()
            .add_source(config::File::from_str(
                r#"
                [server]
                port = 3000
                [database]
                url = "sqlite://test.db"
                [auth]
                jwt_secret = "test-secret"
                "#,
                config::FileFormat::Toml,
            ))
            .add_source(
                Environment::with_prefix("APP")
                    .separator("__")
                    .try_parsing(true)
            )
            .build()
            .unwrap()
            .try_deserialize()
            .unwrap();

        assert_eq!(config.server.port, 9999);

        // Clean up
        std::env::remove_var("APP_SERVER__PORT");
    }

    #[test]
    fn test_validation_catches_zero_port() {
        let config = AppConfig {
            server: ServerConfig {
                host: "localhost".into(),
                port: 0,
                request_timeout_secs: 30,
                log_level: "info".into(),
            },
            database: DatabaseConfig {
                url: "sqlite://test.db".into(),
                max_connections: 5,
            },
            auth: AuthConfig {
                jwt_secret: SecretString::from("test-secret".to_string()),
            },
        };

        assert!(config.validate().is_err());
    }
}
```

Note: tests that set environment variables should not run in parallel (env vars are process-global). Use `serial_test` crate or the `#[serial]` attribute if you have many such tests. Or better - structure your config loading to accept an `Environment` source as a parameter so tests can inject values without touching the process environment.

## Checklist

Before deploying, verify:

- [ ] `.env` is in `.gitignore`
- [ ] `.env.example` exists with placeholder values
- [ ] All secret fields use `SecretString`
- [ ] Config struct has no `pub` fields that expose secrets in Debug output
- [ ] Validation runs at startup, before any work begins
- [ ] Required fields (no default) fail clearly when missing
- [ ] Production rejects dev-only defaults (SQLite, weak secrets)
- [ ] Config is loaded once and passed as a dependency, never re-read from env
- [ ] Log output after startup shows config values but not secrets

Configuration isn't glamorous. Nobody writes blog posts about it because it worked. But the services that are easiest to deploy, easiest to debug, and easiest to hand off to another team - they all have one thing in common: a single config struct, validated at startup, assembled from layered sources, with secrets that never touch git. Build that foundation once, and every feature you add afterward is easier to ship.
