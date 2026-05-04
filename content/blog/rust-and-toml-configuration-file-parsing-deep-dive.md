+++
title = "Rust and TOML - configuration file parsing deep dive"
date = 2026-01-09
description = "TOML is the config format Rust picked, and the toml crate handles parsing, serializing, and round-tripping comments. A tour through tables, arrays of tables, datetime, and why TOML beats YAML for config."

[taxonomies]
tags = ["rust", "toml", "config", "serde"]
+++

Every Rust project starts the same way: `cargo new`, and the first file you open is `Cargo.toml`. TOML is so deeply baked into the Rust ecosystem that most developers stop noticing it. It is just "the format Cargo uses." But TOML is a real, well-specified config language with its own data model, and the [`toml`](https://docs.rs/toml) crate is one of the most polished serde format crates in the ecosystem. It handles tables, arrays of tables, native datetime values, inline tables, and round-tripping with comments preserved - the last of which is something neither JSON nor YAML can do without third-party hacks.

This post walks through how TOML actually works on the wire, how it maps onto the serde data model, where the inline-table-vs-regular-table distinction matters, how `toml` and `toml_edit` differ, and why TOML wins for config files even when YAML feels more flexible. If you're not already comfortable with how serde decouples your structs from a wire format, I covered that in [Serde deep dive - beyond derive](/blog/serde-deep-dive-beyond-derive/) - the patterns there carry over directly.

<!-- more -->

## What TOML actually is

TOML stands for **Tom's Obvious, Minimal Language**. It was created by Tom Preston-Werner (one of GitHub's founders) in 2013 specifically as a config file format that was easier to read than JSON and less ambiguous than YAML. The current spec is [TOML 1.0.0](https://toml.io/en/v1.0.0), released in January 2021, and it is stable - new features land in 1.x point releases, and parsers are expected to be strict.

The data model is intentionally narrow. TOML has exactly seven value types:

- **String** - basic `"text"`, literal `'text'`, multi-line `"""text"""`, multi-line literal `'''text'''`
- **Integer** - 64-bit signed, with `0x`, `0o`, `0b` prefixes and `_` separators
- **Float** - 64-bit IEEE 754, including `inf` and `nan`
- **Boolean** - `true` / `false`, lowercase only
- **Datetime** - offset, local, date, and time variants - all RFC 3339 compatible
- **Array** - heterogeneous in the spec, homogeneous in practice for most parsers
- **Table** - the only container type, equivalent to a map

Notably absent: null. TOML has no null. Optional fields are expressed by being absent. This is a deliberate design choice and lines up nicely with `Option<T>` in Rust - more on that below.

## The toml crate landscape

There are three crates worth knowing about, all maintained as part of the [toml-rs](https://github.com/toml-rs/toml) workspace:

- [`toml`](https://docs.rs/toml) - the high-level serde-integrated parser. This is what you reach for 95% of the time. Current stable is 0.8.x.
- [`toml_edit`](https://docs.rs/toml_edit) - a format-preserving editor. Parses TOML into a tree that retains whitespace, comments, and original number formatting, so you can modify a single value and write back without reformatting the whole file. Cargo itself uses this.
- [`toml_datetime`](https://docs.rs/toml_datetime) - the datetime types factored out into a tiny crate so other formats can interop with them.

The split exists because two different jobs - "parse config into a typed struct" and "edit a config file in place" - have very different design constraints. The `toml` crate optimizes for ergonomic deserialization and throws away formatting. `toml_edit` keeps everything.

For most apps:

```toml
# Cargo.toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
toml = "0.8"
```

That's it. The `toml` crate pulls in `serde` and reuses the data model - if your struct already has `#[derive(Deserialize)]` for JSON, it almost certainly works with TOML too.

## Parsing into a struct

The basic shape:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Config {
    server: ServerConfig,
    database: DatabaseConfig,
}

#[derive(Debug, Deserialize)]
struct ServerConfig {
    host: String,
    port: u16,
    workers: usize,
}

#[derive(Debug, Deserialize)]
struct DatabaseConfig {
    url: String,
    max_connections: u32,
    timeout_secs: u64,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let raw = std::fs::read_to_string("config.toml")?;
    let cfg: Config = toml::from_str(&raw)?;
    println!("{:#?}", cfg);
    Ok(())
}
```

And the matching `config.toml`:

```toml
[server]
host = "0.0.0.0"
port = 8080
workers = 4

[database]
url = "postgres://localhost/myapp"
max_connections = 32
timeout_secs = 30
```

Each `[section]` header in TOML is a table. A table maps to a struct field whose type is itself a struct. This is the most common pattern - you nest tables for grouping, and the struct hierarchy mirrors the file structure exactly.

## Inline tables vs regular tables

TOML has two ways to write a table. The header form:

```toml
[server]
host = "0.0.0.0"
port = 8080
```

And the inline form:

```toml
server = { host = "0.0.0.0", port = 8080 }
```

These produce **identical parsed output**. The choice is purely stylistic. But there are rules:

- Inline tables must be on a single line. Newlines inside `{ }` are forbidden.
- Inline tables are immutable once defined. You cannot add to them with `[server.timeout]` after declaring `server = { ... }`.
- Header form scales better - you can grow a section over many lines and add comments to individual fields.

Practically, use header form for any section with more than two or three fields, and inline form only for short, atomic groups (like coordinates `point = { x = 1, y = 2 }`). Cargo follows this convention - dependencies are usually inline (`serde = "1.0"` is sugar for `serde = { version = "1.0" }`), but `[package]` is always a header.

From the parser's perspective the distinction is a syntactic shortcut. Both produce the same `Value::Table` and feed the same `serialize_map` / `deserialize_map` calls into serde.

## Arrays of tables

This is where TOML gets interesting and YAML falls down. An array of tables uses the double-bracket header `[[name]]`:

```toml
[[user]]
name = "alice"
role = "admin"

[[user]]
name = "bob"
role = "guest"

[[user]]
name = "carol"
role = "guest"
```

Each `[[user]]` block starts a new element in the `user` array. This maps cleanly to `Vec<User>`:

```rust
#[derive(Debug, Deserialize)]
struct Config {
    user: Vec<User>,
}

#[derive(Debug, Deserialize)]
struct User {
    name: String,
    role: String,
}
```

The same data could be written as inline-table arrays:

```toml
user = [
    { name = "alice", role = "admin" },
    { name = "bob", role = "guest" },
    { name = "carol", role = "guest" },
]
```

Both parse the same. But the `[[user]]` form lets each user be its own block with comments, dotted sub-tables, and arbitrary length - which is why Cargo's `[[bin]]` and `[[bench]]` use this form. Try writing a Cargo manifest with multiple `[[bin]]` targets in YAML or JSON and you immediately notice how much syntactic noise you save.

## Datetime is a first-class type

This is one of TOML's quiet wins. JSON has no datetime - you encode it as a string and pray everyone agrees on the format. YAML has datetime in the spec but parsers disagree on which subset to support. TOML has four datetime types with strict RFC 3339 grammar:

```toml
offset_dt = 1979-05-27T07:32:00Z
offset_dt_2 = 1979-05-27T00:32:00-07:00
local_dt = 1979-05-27T07:32:00
local_date = 1979-05-27
local_time = 07:32:00
```

The `toml` crate maps these to the [`toml::value::Datetime`](https://docs.rs/toml/latest/toml/value/struct.Datetime.html) type by default, but the pragmatic move is to use [`chrono`](https://docs.rs/chrono) or [`time`](https://docs.rs/time) and let serde do the conversion:

```rust
use chrono::{DateTime, Utc, NaiveDate};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Release {
    version: String,
    released_at: DateTime<Utc>,    // matches offset datetime
    deprecated_on: Option<NaiveDate>,  // matches local date
}
```

Both `chrono` and `time` ship serde adapters that read TOML's RFC 3339 strings. Because the value is **typed at parse time**, you cannot accidentally feed a misformatted date into your config and have it silently treated as a string - the parser rejects it. This is what people mean when they say "TOML is unambiguous."

## serde attributes that matter for config

A few serde attributes come up over and over for config parsing:

```rust
use serde::Deserialize;
use std::time::Duration;
use std::path::PathBuf;

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
struct ServerConfig {
    #[serde(default = "default_host")]
    host: String,

    #[serde(default = "default_port")]
    port: u16,

    #[serde(default)]
    tls: bool,

    #[serde(rename = "log-level")]
    log_level: String,

    #[serde(with = "humantime_serde")]
    request_timeout: Duration,

    cert_path: Option<PathBuf>,
}

fn default_host() -> String { "127.0.0.1".into() }
fn default_port() -> u16 { 8080 }
```

A few notes:

- `#[serde(deny_unknown_fields)]` is **the single most important attribute for config parsing**. Without it, a typo like `por = 8080` silently uses the default and your service binds to the wrong port. With it, the parser rejects the file with a clear error pointing at the offending key.
- `#[serde(default = "fn")]` lets you provide computed defaults. `#[serde(default)]` alone calls `Default::default()`.
- `#[serde(rename = "log-level")]` handles kebab-case keys, which TOML allows naturally and Rust struct fields don't.
- `Option<T>` for absent fields. Don't use sentinel values like `""` or `-1` - TOML has no null, but `Option` lines up perfectly with "key is missing."
- [`humantime_serde`](https://docs.rs/humantime-serde) lets you write `request_timeout = "30s"` instead of `30000` in milliseconds. Strongly recommended for human-edited config.

## Validation - past serde

Serde will catch type mismatches, missing required fields, unknown fields (with `deny_unknown_fields`), and invalid enum variants. It will not catch semantic errors like "port 0 is invalid" or "max_connections must be greater than min_connections."

The pragmatic pattern is a `validate()` method called right after deserialization:

```rust
impl Config {
    fn validate(&self) -> Result<(), String> {
        if self.server.port == 0 {
            return Err("server.port cannot be 0".into());
        }
        if self.database.max_connections == 0 {
            return Err("database.max_connections must be > 0".into());
        }
        if self.server.workers > 1024 {
            return Err(format!(
                "server.workers={} exceeds the safety limit of 1024",
                self.server.workers
            ));
        }
        Ok(())
    }
}

let cfg: Config = toml::from_str(&raw)?;
cfg.validate().map_err(|e| format!("invalid config: {e}"))?;
```

For richer validation, the [`validator`](https://docs.rs/validator) crate gives you `#[validate(range(min = 1, max = 65535))]` style attributes that integrate cleanly. For very complex config (cross-field constraints, environment-dependent rules), parse into an "unchecked" struct and convert into a "validated" struct with `TryFrom`. That makes invalid states unrepresentable in the rest of your code, which is the same pattern that makes domain modeling in Rust pleasant.

## Round-tripping with toml_edit

The plain `toml` crate's `Value` type is lossy. Parse a file, serialize it back, and you'll lose:

- Comments (they are not in the data model at all)
- Whitespace and blank-line groupings
- Choice of inline-vs-header form
- Number formatting (was it `0xff` or `255`?)

For tools that **edit** config files - think `cargo add`, `cargo set-version`, an installer that flips a feature flag - this is unacceptable. That's where [`toml_edit`](https://docs.rs/toml_edit) comes in:

```rust
use toml_edit::{DocumentMut, value};

let raw = std::fs::read_to_string("Cargo.toml")?;
let mut doc = raw.parse::<DocumentMut>()?;

// Bump version, preserve everything else.
doc["package"]["version"] = value("0.4.2");

std::fs::write("Cargo.toml", doc.to_string())?;
```

`toml_edit` parses into a tree of `Item` nodes, each carrying its surrounding decor (comments before, whitespace after, choice of formatting). Modifying a value updates the node but keeps every byte you didn't touch. This is the same library Cargo uses internally for `cargo add` - [you can read the implementation here](https://github.com/rust-lang/cargo/tree/master/crates/cargo-util-schemas).

The trade-off: `toml_edit` is heavier and has a more awkward API than `toml`. Don't reach for it unless you're actually editing files. For loading config at startup, `toml::from_str` into a serde struct is the right tool.

## Why TOML beats YAML and JSON for config

YAML's ambiguity is genuinely dangerous. The infamous "Norway problem":

```yaml
countries:
  - NO
  - SE
  - FI
```

In YAML 1.1 (which many parsers still use), `NO` is parsed as a boolean `false`, not a string. The same trap applies to `yes`, `on`, `off`, country code `TR`, and any unquoted string that happens to look like a YAML reserved word. There are 22 different ways to write `true` in YAML 1.1. PyYAML, Kubernetes manifests, GitHub Actions - these have all shipped bugs over this exact issue.

TOML doesn't have this problem because the value type is determined by **syntax**, not content. `NO` is invalid (bare words aren't strings). `"NO"` is unambiguously a string. `true` is the only spelling of true. Numbers are numbers. Strings are quoted. There are no surprises.

JSON's problem is different - it's an excellent data interchange format and a mediocre config format. The two complaints that come up every time:

- **No comments.** Configuration files explain themselves with comments. `// dev only` next to a debug flag is invaluable. JSON can't do it. People use `"_comment": "..."` keys, JSON5, JSONC - all of which are not real JSON, none of which interop. TOML's `#` comments work everywhere.
- **Trailing comma rules.** JSON forbids them. Every config file edit risks a syntax error from a stray comma. TOML allows trailing commas in arrays.

YAML solves both of these but introduces ambiguity, significant whitespace bugs, anchors and aliases as a footgun, and a spec so large that no parser implements all of it. TOML occupies the sweet spot: comments, types, no significant whitespace, no exotic features.

That's why Rust picked it for Cargo. Pip, poetry, Hugo, and the Python `pyproject.toml` standard followed for the same reasons.

## Best practices for config file design

A few patterns that show up in well-designed Rust apps:

1. **One `Config` struct, deeply nested by section.** Mirror the TOML structure 1:1 in your types. Don't flatten everything into a top-level struct - the file becomes unreadable at scale.

2. **`#[serde(deny_unknown_fields)]` on every config struct.** A typo is always a bug. Catch it loud at startup, not silently in production.

3. **`Option<T>` for genuinely optional fields, defaults for everything else.** If a missing key has a sensible meaning, give it a default. If absence is meaningful, use `Option`.

4. **Layer config sources.** Read defaults from a baked-in TOML string, override with `/etc/myapp/config.toml`, override with `$HOME/.config/myapp/config.toml`, override with environment variables. The [`figment`](https://docs.rs/figment) crate composes this cleanly and supports TOML out of the box.

5. **Validate after parsing, not during.** Parsing checks shape. Validation checks meaning. Keep them separate so error messages are clear.

6. **Don't put secrets in TOML.** Use environment variables or a secret manager. TOML files end up in dotfiles, backups, and screenshots.

7. **Version your config schema.** Add a `version = 1` key at the top and bump it when the schema changes incompatibly. Old config files are an inevitable support problem.

8. **Use kebab-case keys with `#[serde(rename_all = "kebab-case")]`.** It looks more natural in TOML than `snake_case`, and most established TOML files (Cargo, pyproject) use kebab-case for multi-word keys.

The toml crate is one of those pieces of the Rust ecosystem that just works. It's strict where strictness helps, ergonomic where serde already gives you a head start, and the format itself avoids the worst pitfalls of YAML and JSON. If you're writing a config-driven Rust service in 2026, there's almost no reason to pick anything else.
