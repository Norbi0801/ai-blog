+++
title = "JSON Schema - validating data at the boundary"
date = 2025-11-14
description = "How JSON Schema lets you define, generate, and enforce data contracts at every system boundary - from API inputs to config files to MCP tool calls."

[taxonomies]
tags = ["rust", "json-schema", "validation", "api"]
+++

Rust's type system catches a lot of bugs at compile time. But there is a gap. The moment data crosses a system boundary - an HTTP request body, a config file loaded from disk, a webhook payload, an MCP tool call from an AI model - you are dealing with bytes that could contain anything. `serde` will reject structurally invalid JSON, sure. But what about a `port` field set to `-5`? An `email` field containing `"lol not an email"`? A string that should match a specific regex but does not?

This is where JSON Schema fits in. It is a declarative language for describing the shape and constraints of JSON data, and it has become the standard way to define data contracts across system boundaries.

<!-- more -->

## The boundary problem

If you have built a webhook receiver (I wrote about [building one in Rust](/blog/building-a-webhook-receiver-in-rust/) previously), you know the feeling. External systems send you JSON, and you hope it matches your expectations. You deserialize into a Rust struct, and if it works, great. If it does not, you get a serde error that says something like `missing field 'user_id' at line 1 column 42`.

That is structural validation - does this blob of bytes parse into my type? But it does not cover semantic validation:

- Is this integer within an acceptable range?
- Does this string match a known pattern?
- Are these two fields mutually exclusive?
- Is this array non-empty?

You could write all of this logic by hand in every handler. Or you could describe your constraints once and let a validator enforce them everywhere.

## What JSON Schema actually is

JSON Schema is a specification - currently at [Draft 2020-12](https://json-schema.org/draft/2020-12) - that uses JSON to describe JSON. A schema is itself a JSON document that defines what valid data looks like.

Here is a basic example:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "age": {
      "type": "integer",
      "minimum": 0,
      "maximum": 150
    },
    "email": {
      "type": "string",
      "format": "email"
    },
    "role": {
      "type": "string",
      "enum": ["admin", "user", "moderator"]
    }
  },
  "required": ["name", "email", "role"]
}
```

This single document tells you: the data is an object, it must have `name`, `email`, and `role`, the name is between 1-100 characters, age (if present) is a non-negative integer up to 150, email should look like an email, and role must be one of three specific values.

No code. No language-specific logic. Just a declarative contract that any JSON Schema validator - in Rust, Python, JavaScript, Go, whatever - can enforce.

## The keyword vocabulary

JSON Schema's power comes from its keywords. Let me walk through the ones you will actually use.

### Type constraints

The `type` keyword restricts what kind of JSON value is accepted:

```json
{ "type": "string" }
{ "type": "integer" }
{ "type": "number" }
{ "type": "boolean" }
{ "type": "array" }
{ "type": "object" }
{ "type": ["string", "null"] }
```

That last one - an array of types - is how you express nullable fields. The value must be either a string or `null`.

### String validation

```json
{
  "type": "string",
  "minLength": 1,
  "maxLength": 255,
  "pattern": "^[a-z][a-z0-9_]*$"
}
```

`pattern` takes a regular expression (ECMA 262 flavor). This one enforces a lowercase identifier format - starts with a letter, followed by letters, digits, or underscores.

The `format` keyword provides semantic validation for common string types:

```json
{ "type": "string", "format": "email" }
{ "type": "string", "format": "uri" }
{ "type": "string", "format": "date-time" }
{ "type": "string", "format": "ipv4" }
{ "type": "string", "format": "uuid" }
```

One nuance: `format` is an annotation by default in Draft 2020-12. Validators are not required to enforce it unless you opt in. In the `jsonschema` Rust crate, you enable it with `.should_validate_formats(true)`.

### Numeric constraints

```json
{
  "type": "integer",
  "minimum": 1,
  "maximum": 65535,
  "multipleOf": 1
}
```

This describes a valid port number. `minimum` and `maximum` are inclusive. For exclusive bounds, use `exclusiveMinimum` and `exclusiveMaximum`:

```json
{
  "type": "number",
  "exclusiveMinimum": 0,
  "exclusiveMaximum": 1.0
}
```

### Enumerations and constants

```json
{ "enum": ["tcp", "udp", "quic"] }
{ "const": "v2" }
```

`enum` restricts to a set of allowed values (of any type - not just strings). `const` pins it to a single exact value.

### Array validation

```json
{
  "type": "array",
  "items": { "type": "string", "format": "ipv4" },
  "minItems": 1,
  "maxItems": 10,
  "uniqueItems": true
}
```

This describes a non-empty list of up to 10 unique IPv4 addresses. `items` defines the schema that every element must satisfy.

For tuple-like arrays where each position has a different type, Draft 2020-12 introduced `prefixItems`:

```json
{
  "type": "array",
  "prefixItems": [
    { "type": "string" },
    { "type": "integer" },
    { "type": "boolean" }
  ],
  "items": false
}
```

This says: first element is a string, second is an integer, third is a boolean, and no additional elements are allowed.

### Object validation

```json
{
  "type": "object",
  "properties": {
    "host": { "type": "string" },
    "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
  },
  "required": ["host"],
  "additionalProperties": false
}
```

`required` lists fields that must be present. `additionalProperties: false` rejects any keys not listed in `properties` - this is useful when you want strict schemas that catch typos in field names.

### Composition - allOf, anyOf, oneOf, not

This is where schemas get really expressive.

```json
{
  "oneOf": [
    {
      "type": "object",
      "properties": {
        "type": { "const": "webhook" },
        "url": { "type": "string", "format": "uri" }
      },
      "required": ["type", "url"]
    },
    {
      "type": "object",
      "properties": {
        "type": { "const": "email" },
        "address": { "type": "string", "format": "email" }
      },
      "required": ["type", "address"]
    }
  ]
}
```

`oneOf` means exactly one of these sub-schemas must match. `anyOf` means at least one. `allOf` means all of them. `not` inverts. You can model tagged unions, conditional fields, and complex domain rules without writing procedural validation code.

### Conditional schemas

```json
{
  "type": "object",
  "properties": {
    "payment_method": { "enum": ["card", "bank_transfer"] },
    "card_number": { "type": "string" },
    "iban": { "type": "string" }
  },
  "required": ["payment_method"],
  "if": {
    "properties": { "payment_method": { "const": "card" } }
  },
  "then": {
    "required": ["card_number"]
  },
  "else": {
    "required": ["iban"]
  }
}
```

The `if`/`then`/`else` keywords let you express "when field X has value Y, then field Z is required." This models the kind of conditional validation that usually ends up as a tangle of `if` statements in handler code.

## Generating schemas from Rust structs with schemars

Writing JSON Schema by hand is fine for small schemas, but tedious for complex types. The [`schemars`](https://crates.io/crates/schemars) crate (v1.2.1 as of writing) generates JSON Schema directly from your Rust types.

```toml
[dependencies]
schemars = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

Derive `JsonSchema` alongside your serde derives:

```rust
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, JsonSchema)]
pub struct CreateUserRequest {
    /// The user's display name.
    #[schemars(length(min = 1, max = 100))]
    pub name: String,

    /// Email address.
    #[schemars(email)]
    pub email: String,

    /// Must be between 1 and 65535.
    #[schemars(range(min = 1, max = 65535))]
    pub port: u16,

    /// User role.
    pub role: Role,

    /// Optional bio, max 500 characters.
    #[schemars(length(max = 500))]
    pub bio: Option<String>,
}

#[derive(Debug, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum Role {
    Admin,
    User,
    Moderator,
}
```

Generate the schema at build time or runtime:

```rust
use schemars::schema_for;

fn main() {
    let schema = schema_for!(CreateUserRequest);
    println!("{}", serde_json::to_string_pretty(&schema).unwrap());
}
```

This outputs a full Draft 2020-12 schema with all the constraints you specified via attributes. The `Option<String>` for `bio` becomes a nullable type (`["string", "null"]`). The `Role` enum becomes `{"enum": ["admin", "user", "moderator"]}` because schemars reads the `#[serde(rename_all = "snake_case")]` attribute.

That is the key insight: **schemars reads your serde attributes**. If you use `#[serde(rename = "...")]`, `#[serde(tag = "type")]`, `#[serde(flatten)]`, or any other serde annotation, the generated schema reflects it. Your schema and your deserialization logic stay in sync automatically.

### What the derive macro generates under the hood

When you write `#[derive(JsonSchema)]`, schemars generates an implementation of the `JsonSchema` trait:

```rust
pub trait JsonSchema {
    fn schema_name() -> Cow<'static, str>;
    fn schema_id() -> Cow<'static, str>;
    fn json_schema(generator: &mut SchemaGenerator) -> Schema;
}
```

`json_schema()` returns a `Schema` value that wraps a `serde_json::Value`. The generator handles `$ref` resolution - if your struct contains another `JsonSchema` type, it generates a `$defs` reference instead of inlining the schema, avoiding infinite recursion for recursive types.

### Validation attributes

schemars gives you a rich set of attributes that map directly to JSON Schema keywords:

```rust
#[schemars(length(min = 1, max = 255))]    // minLength / maxLength
#[schemars(range(min = 0, max = 100))]     // minimum / maximum
#[schemars(pattern = r"^[a-z]+$")]         // pattern (regex)
#[schemars(email)]                          // format: "email"
#[schemars(url)]                            // format: "uri"
#[schemars(contains(pattern = "needle"))]   // contains
#[schemars(required)]                       // forces Option<T> into required
```

Doc comments (`///`) become `description` fields in the schema. That is nice because it means your Rust documentation doubles as your API documentation.

## Runtime validation with jsonschema

Generating a schema is half the story. You also need to validate incoming data against it. The [`jsonschema`](https://crates.io/crates/jsonschema) crate (v0.45.0) handles this.

```toml
[dependencies]
jsonschema = "0.45"
serde_json = "1"
```

### Basic validation

```rust
use serde_json::json;

fn main() {
    let schema = json!({
        "type": "object",
        "properties": {
            "name": { "type": "string", "minLength": 1 },
            "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
        },
        "required": ["name", "port"]
    });

    let valid_input = json!({"name": "web", "port": 8080});
    let invalid_input = json!({"name": "", "port": -1});

    // Quick boolean check
    assert!(jsonschema::is_valid(&schema, &valid_input));
    assert!(!jsonschema::is_valid(&schema, &invalid_input));

    // Detailed error messages
    if let Err(errors) = jsonschema::validate(&schema, &invalid_input) {
        for error in errors {
            eprintln!("{}", error);
            // "'' is shorter than 1 character"
            // "-1 is less than the minimum of 1"
        }
    }
}
```

### Compiled validators for repeated use

If you are validating the same schema against many inputs (like in an API handler), compile the validator once:

```rust
use jsonschema::Validator;
use serde_json::json;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let schema = json!({
        "type": "object",
        "properties": {
            "name": { "type": "string", "minLength": 1 }
        },
        "required": ["name"]
    });

    // Compile once
    let validator = jsonschema::validator_for(&schema)?;

    // Validate many times - no re-compilation
    let inputs = vec![
        json!({"name": "alice"}),
        json!({"name": ""}),
        json!({}),
        json!({"name": 42}),
    ];

    for input in &inputs {
        if validator.is_valid(input) {
            println!("valid: {}", input);
        } else {
            let errors: Vec<String> = validator
                .iter_errors(input)
                .map(|e| e.to_string())
                .collect();
            println!("invalid: {} - errors: {:?}", input, errors);
        }
    }

    Ok(())
}
```

The compiled `Validator` builds an internal representation of the schema once, so subsequent validations skip the parsing step. For hot paths, this matters.

### Enabling format validation

Remember that `format` is annotation-only by default. To enforce it:

```rust
let validator = jsonschema::options()
    .should_validate_formats(true)
    .build(&schema)?;
```

Or for a specific draft:

```rust
let validator = jsonschema::draft202012::options()
    .should_validate_formats(true)
    .build(&schema)?;
```

### Performance

The `jsonschema` crate is fast. Benchmarks from the project show it is 75-645x faster than alternatives like `valico` and `jsonschema_valid` for complex schemas. For recursive schemas, the gap widens to over 5000x. It uses a compiled validation approach internally - the schema is analyzed once and converted to an optimized instruction set.

## Putting it together: generate and validate

Here is the full pipeline - define your types in Rust, generate the schema, validate incoming data:

```rust
use jsonschema::Validator;
use schemars::{schema_for, JsonSchema};
use serde::{Deserialize, Serialize};
use serde_json::json;

#[derive(Debug, Serialize, Deserialize, JsonSchema)]
pub struct DeployConfig {
    /// Target environment.
    pub environment: Environment,

    /// Docker image reference.
    #[schemars(pattern = r"^[\w.\-/]+:[\w.\-]+$")]
    pub image: String,

    /// Number of replicas (1-20).
    #[schemars(range(min = 1, max = 20))]
    pub replicas: u8,

    /// CPU limit in millicores.
    #[schemars(range(min = 100, max = 8000))]
    pub cpu_millis: u32,

    /// Memory limit in MiB.
    #[schemars(range(min = 64, max = 16384))]
    pub memory_mib: u32,

    /// Optional health check path.
    pub health_check: Option<String>,
}

#[derive(Debug, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "lowercase")]
pub enum Environment {
    Dev,
    Staging,
    Production,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Generate the schema from Rust types
    let schema = schema_for!(DeployConfig);
    let schema_value = serde_json::to_value(&schema)?;

    // Compile the validator
    let validator = jsonschema::options()
        .should_validate_formats(true)
        .build(&schema_value)?;

    // Simulate incoming JSON from an API request
    let input = json!({
        "environment": "staging",
        "image": "myapp/backend:v1.4.2",
        "replicas": 3,
        "cpu_millis": 500,
        "memory_mib": 256,
        "health_check": "/healthz"
    });

    // Validate before deserializing
    if let Err(errors) = validator.validate(&input) {
        let messages: Vec<String> = errors.map(|e| {
            format!("{} at {}", e, e.instance_path)
        }).collect();
        eprintln!("Validation failed:\n{}", messages.join("\n"));
        return Ok(());
    }

    // Safe to deserialize - we know it matches our schema
    let config: DeployConfig = serde_json::from_value(input)?;
    println!("Deploying {} with {} replicas", config.image, config.replicas);

    Ok(())
}
```

Validate first, then deserialize. This way you get rich, structured error messages about exactly which constraints failed, rather than a generic serde error. The user (or calling system) knows that `replicas` must be between 1 and 20, not just that "invalid type" occurred.

## JSON Schema in OpenAPI

If you have read my post on [designing RESTful APIs](/blog/designing-restful-apis-practical-guidelines-beyond-the-theory/), you know that API documentation matters. OpenAPI is the standard for describing REST APIs, and since version 3.1.0, OpenAPI uses JSON Schema Draft 2020-12 with full compatibility.

This means the schema you generate with schemars is directly usable in an OpenAPI spec:

```yaml
paths:
  /api/deploy:
    post:
      summary: Create a deployment
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/DeployConfig'
      responses:
        '200':
          description: Deployment created
components:
  schemas:
    DeployConfig:
      type: object
      properties:
        environment:
          type: string
          enum: [dev, staging, production]
        image:
          type: string
          pattern: "^[\\w.\\-/]+:[\\w.\\-]+$"
        replicas:
          type: integer
          minimum: 1
          maximum: 20
        cpu_millis:
          type: integer
          minimum: 100
          maximum: 8000
        memory_mib:
          type: integer
          minimum: 64
          maximum: 16384
        health_check:
          type: [string, "null"]
      required: [environment, image, replicas, cpu_millis, memory_mib]
```

Before OpenAPI 3.1, the spec used a modified subset of JSON Schema Draft 7 with some incompatibilities (like `nullable: true` instead of `type: ["string", "null"]`). If you are starting a new API, use OpenAPI 3.1+ and get full JSON Schema support out of the box.

Several Rust crates - like [`utoipa`](https://crates.io/crates/utoipa) and [`aide`](https://crates.io/crates/aide) - can generate OpenAPI specs from your Rust types, using the same derive macro approach. The schemas end up in the `components/schemas` section of the generated spec, and API gateways, client generators, and documentation tools all consume them.

## JSON Schema in MCP tool definitions

If you have worked with MCP (I covered the [protocol primitives here](/blog/mcp-tools-vs-resources-vs-prompts-understanding-the-protocol/)), you know that tool definitions use JSON Schema for their `inputSchema`. This is how an AI model knows what parameters a tool accepts and what constraints apply.

Here is a tool definition:

```json
{
  "name": "create_deployment",
  "description": "Deploy a service to the target environment",
  "inputSchema": {
    "type": "object",
    "properties": {
      "environment": {
        "type": "string",
        "enum": ["dev", "staging", "production"],
        "description": "Target deployment environment"
      },
      "image": {
        "type": "string",
        "pattern": "^[\\w.\\-/]+:[\\w.\\-]+$",
        "description": "Docker image reference (e.g., myapp/backend:v1.4.2)"
      },
      "replicas": {
        "type": "integer",
        "minimum": 1,
        "maximum": 20,
        "description": "Number of instances to run"
      }
    },
    "required": ["environment", "image", "replicas"]
  }
}
```

The MCP spec (2025-06-18) requires `inputSchema` to be a JSON Schema object with `"type": "object"`. Servers *must* validate all tool inputs against this schema before execution. Since the spec also added `outputSchema` support, you can validate both directions.

Here is what makes this powerful: you can generate the `inputSchema` directly from your Rust handler's input type using schemars. The schema that validates your API request body is the same schema that tells an AI model what arguments your tool accepts. One source of truth.

```rust
use schemars::{schema_for, JsonSchema};
use serde::Deserialize;

#[derive(Debug, Deserialize, JsonSchema)]
pub struct CreateDeploymentInput {
    /// Target deployment environment.
    pub environment: Environment,

    /// Docker image reference (e.g., myapp/backend:v1.4.2).
    #[schemars(pattern = r"^[\w.\-/]+:[\w.\-]+$")]
    pub image: String,

    /// Number of instances to run.
    #[schemars(range(min = 1, max = 20))]
    pub replicas: u8,
}

// Generate the schema for MCP tool registration
fn tool_input_schema() -> serde_json::Value {
    serde_json::to_value(schema_for!(CreateDeploymentInput)).unwrap()
}
```

The descriptions from doc comments end up in the schema's `description` fields, which means the AI model sees them when deciding how to call the tool. Good doc comments on your Rust structs directly improve how well AI models use your tools.

## Validating config files

API inputs are not the only boundary. Config files are another one. When your application reads a YAML or TOML config at startup, you are trusting that the file on disk is valid. JSON Schema can catch misconfigurations before they cause runtime panics.

```rust
use jsonschema::Validator;
use schemars::{schema_for, JsonSchema};
use serde::Deserialize;

#[derive(Debug, Deserialize, JsonSchema)]
pub struct AppConfig {
    /// Port to listen on.
    #[schemars(range(min = 1, max = 65535))]
    pub port: u16,

    /// Database connection URL.
    #[schemars(url)]
    pub database_url: String,

    /// Maximum concurrent connections.
    #[schemars(range(min = 1, max = 1000))]
    pub max_connections: u32,

    /// Log level.
    pub log_level: LogLevel,
}

#[derive(Debug, Deserialize, JsonSchema)]
#[serde(rename_all = "lowercase")]
pub enum LogLevel {
    Trace,
    Debug,
    Info,
    Warn,
    Error,
}

fn validate_config(raw_json: &serde_json::Value) -> Result<AppConfig, Vec<String>> {
    let schema = serde_json::to_value(schema_for!(AppConfig)).unwrap();
    let validator = jsonschema::options()
        .should_validate_formats(true)
        .build(&schema)
        .expect("valid schema");

    let errors: Vec<String> = validator
        .iter_errors(raw_json)
        .map(|e| format!("{} (at {})", e, e.instance_path))
        .collect();

    if !errors.is_empty() {
        return Err(errors);
    }

    serde_json::from_value(raw_json.clone()).map_err(|e| vec![e.to_string()])
}
```

When the config is wrong, the user sees something like `"10000 is greater than the maximum of 1000" (at /max_connections)` instead of a cryptic panic three minutes into startup when the connection pool overflows.

## Where to validate: the boundary principle

The general rule: validate at the boundary, trust internally.

The "boundary" is wherever data enters your system from an untrusted source:

- **HTTP request handlers** - validate the request body before passing it to business logic
- **Config file loading** - validate at startup, fail fast with clear messages
- **Message queue consumers** - validate the message payload before processing
- **MCP tool handlers** - validate `inputSchema` before executing the tool
- **CLI argument parsing** - validate complex structured inputs (JSON flags, config overrides)
- **Webhook receivers** - validate the payload after verifying the signature (as I covered in the [webhook post](/blog/building-a-webhook-receiver-in-rust/))

Once data passes validation at the boundary, your internal functions can work with typed Rust structs and trust that constraints hold. You do not need to re-validate `port > 0` in every function that touches the port value - you checked it once at the edge.

This maps directly to how Rust's type system works. If you have read my post on [trait bounds](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg/), you know that `where T: Display + Send` constrains what can enter a function. JSON Schema does the same thing for data - it constrains what can enter your system. The difference is that trait bounds are checked at compile time, while JSON Schema is checked at runtime. Both serve the same purpose: making invalid states unrepresentable.

## Schema reuse with $ref and $defs

Real systems have schemas that reference each other. A `User` schema appears in the `CreateOrder` schema which appears in the `BatchImport` schema. JSON Schema handles this with `$ref`:

```json
{
  "$defs": {
    "Address": {
      "type": "object",
      "properties": {
        "street": { "type": "string" },
        "city": { "type": "string" },
        "zip": { "type": "string", "pattern": "^[0-9]{5}$" }
      },
      "required": ["street", "city", "zip"]
    }
  },
  "type": "object",
  "properties": {
    "billing_address": { "$ref": "#/$defs/Address" },
    "shipping_address": { "$ref": "#/$defs/Address" }
  },
  "required": ["billing_address"]
}
```

schemars handles this automatically. When your struct contains another `JsonSchema` type, the generated schema uses `$defs` and `$ref` to avoid duplication. Recursive types (like tree nodes) work too - the generator detects cycles and emits references instead of infinitely nested schemas.

## Practical considerations

A few things I have learned from using JSON Schema in production:

**Keep schemas close to the types.** If your schema is generated from your Rust struct, they cannot drift apart. If you maintain schemas manually (for an external API you consume), put them in version-controlled `.json` files and load them at startup.

**Fail with all errors, not just the first one.** The `iter_errors` method on `jsonschema`'s validator gives you every violation, not just the first one. Return all of them to the caller. Nobody wants to fix one field, resubmit, fix another field, resubmit, repeat.

**Use `additionalProperties: false` carefully.** It rejects unknown fields, which is great for catching typos but breaks forward compatibility. If your API adds a new optional field, old schemas with `additionalProperties: false` will reject it. For public APIs, leave it open. For internal configs, lock it down.

**Schema validation is not a substitute for business logic validation.** JSON Schema can check that `quantity` is a positive integer, but it cannot check that you have enough inventory to fulfill the order. Schema validation handles structural and constraint validation. Business rules still live in your handler code.

**Watch for format validation performance.** Validating `format: "email"` or `format: "uri"` requires regex matching. If you are validating thousands of records per second, benchmark with format validation enabled vs disabled and decide whether the accuracy-performance tradeoff is worth it for your use case.

## Wrapping up

JSON Schema sits at the intersection of documentation, validation, and interoperability. You define your constraints once - in a language-agnostic format - and get:

- Runtime validation with detailed error messages
- OpenAPI documentation for your REST endpoints
- MCP tool definitions that AI models can understand
- Client-side validation in any language
- Config file validation at startup

With `schemars`, you generate schemas from Rust types you already have. With `jsonschema`, you validate incoming data against those schemas. The types enforce structure at compile time, the schema enforces constraints at runtime. Together they close the gap at the boundary.

Your Rust types say "this is a `u16`." Your JSON Schema says "this is an integer between 1 and 65535." The first catches type mismatches at compile time. The second catches constraint violations at runtime, at the boundary, where untrusted data meets your system. Both are necessary. Neither is sufficient alone.
