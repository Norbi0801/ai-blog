+++
title = "MCP tools vs resources vs prompts - understanding the protocol"
date = 2025-04-12
description = "The three MCP primitives serve different roles: tools let models act, resources let apps provide context, prompts let users structure interactions. Here is how they differ under the hood."

[taxonomies]
tags = ["mcp", "rust", "architecture", "protocols"]
+++

MCP has three primitives. Not two, not five - three. Tools, resources, and prompts. They sound similar on the surface (all three expose "stuff" from a server to a client), but they have fundamentally different control semantics, different JSON-RPC methods, and different use cases. Picking the wrong one means your MCP server either confuses the model, fights the host application, or buries useful functionality where nobody finds it.

This post breaks down each primitive at the protocol level. What JSON-RPC messages actually get sent, who controls what, and when to reach for which one.

<!-- more -->

## The control hierarchy

Before looking at any JSON, the single most important concept in MCP is **who controls** each primitive:

| Primitive | Controlled by  | Think of it as                        |
|-----------|---------------|---------------------------------------|
| Tools     | The model      | "Functions the LLM can call"         |
| Resources | The application| "Data the host app attaches to context" |
| Prompts   | The user       | "Templates the user explicitly picks" |

This is not a soft guideline. The [MCP specification](https://modelcontextprotocol.io/specification/2025-11-25/server) defines this hierarchy explicitly. A tool is something the model decides to invoke. A resource is something the host application decides to include. A prompt is something the user explicitly selects.

Get this wrong and you end up with a tool that should be a resource (the model keeps calling it in a loop to "read" data it should already have in context), or a resource that should be a prompt (the app silently attaches a complex instruction template that the user never asked for).

## JSON-RPC 2.0 - the transport underneath

Everything in MCP is a [JSON-RPC 2.0](https://www.jsonrpc.org/) message. Three message types:

- **Requests** - have an `id`, expect a response
- **Responses** - match a request `id`, carry either `result` or `error`
- **Notifications** - no `id`, fire-and-forget

Every primitive follows the same pattern: a `*/list` method for discovery, then a specific method for usage (`tools/call`, `resources/read`, `prompts/get`). Servers can also push `notifications/*/list_changed` when their available primitives change.

The connection starts with a capability negotiation handshake before any primitives are used. More on that below.

## Tools - model-controlled actions

Tools are functions. The model sees their names, descriptions, and input schemas, then decides whether and when to call them. They can have side effects - creating records, sending emails, writing files, calling external APIs.

### Discovery: `tools/list`

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}
```

The server responds with an array of tool definitions. Each tool has a `name`, `description`, and an `inputSchema` (JSON Schema defining parameters):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "create_issue",
        "description": "Create a new issue in the project tracker",
        "inputSchema": {
          "type": "object",
          "properties": {
            "title": { "type": "string" },
            "body": { "type": "string" },
            "labels": {
              "type": "array",
              "items": { "type": "string" }
            }
          },
          "required": ["title"]
        }
      }
    ]
  }
}
```

### Execution: `tools/call`

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "create_issue",
    "arguments": {
      "title": "Fix flaky CI test",
      "labels": ["bug", "ci"]
    }
  }
}
```

The response carries a `content` array - the tool can return text, images, audio, or even embedded resources:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Created issue #42: Fix flaky CI test"
      }
    ],
    "isError": false
  }
}
```

That `isError` field matters. When `true`, it signals an execution-level failure (bad input, API down) rather than a protocol error. Clients should feed execution errors back to the model so it can self-correct. Protocol errors (malformed request, unknown tool) use standard JSON-RPC error responses instead.

### Tool annotations

Tools support behavioral hints through annotations. These do not change what the tool does - they tell the **client** how to present and gate it:

```json
{
  "name": "delete_file",
  "description": "Permanently delete a file from the filesystem",
  "inputSchema": { "type": "object", "properties": { "path": { "type": "string" } }, "required": ["path"] },
  "annotations": {
    "readOnlyHint": false,
    "destructiveHint": true,
    "idempotentHint": true,
    "openWorldHint": false
  }
}
```

The defaults tell you something about the spec's safety-first design: `readOnlyHint` defaults to `false` (assume writes), `destructiveHint` defaults to `true` (assume dangerous), `idempotentHint` defaults to `false` (assume not safe to retry), `openWorldHint` defaults to `true` (assume external interaction). A client seeing `destructiveHint: true` from a trusted server might show a confirmation dialog. One seeing `readOnlyHint: true` might auto-approve.

These are hints, not guarantees. The spec is [explicit about this](https://modelcontextprotocol.io/specification/2025-11-25/server/tools) - clients must consider annotations untrusted unless the server itself is trusted.

### Structured output

Tools can also define an `outputSchema` for typed, machine-parseable results. When present, the server returns structured data in a `structuredContent` field alongside a text fallback in `content`:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"temperature\": 22.5, \"humidity\": 65}"
      }
    ],
    "structuredContent": {
      "temperature": 22.5,
      "humidity": 65
    }
  }
}
```

The `content` field maintains backward compatibility. The `structuredContent` field gives typed clients something they can validate against the schema.

## Resources - application-controlled context

Resources are read-only data identified by URIs. Think file contents, database schemas, API docs, git diffs. The host application decides which resources to include in the model's context - not the model itself.

### Discovery: `resources/list`

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "resources/list"
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resources": [
      {
        "uri": "file:///project/schema.sql",
        "name": "schema.sql",
        "description": "Database schema definition",
        "mimeType": "application/sql"
      },
      {
        "uri": "git:///project/HEAD/diff",
        "name": "Current changes",
        "description": "Uncommitted git diff",
        "mimeType": "text/x-diff"
      }
    ]
  }
}
```

Each resource has a `uri` (unique identifier), a `name`, optional `description` and `mimeType`. The URI scheme is flexible - `file://`, `git://`, `https://`, or any custom scheme your server defines.

### Reading: `resources/read`

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "resources/read",
  "params": {
    "uri": "file:///project/schema.sql"
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "contents": [
      {
        "uri": "file:///project/schema.sql",
        "mimeType": "application/sql",
        "text": "CREATE TABLE users (\n  id TEXT PRIMARY KEY,\n  email TEXT NOT NULL UNIQUE,\n  created_at TEXT NOT NULL DEFAULT (datetime('now'))\n);"
      }
    ]
  }
}
```

Resources return either `text` (for text content) or `blob` (base64-encoded binary). A single `resources/read` can return multiple content items - useful when a logical resource maps to several pieces of data.

### Resource templates

Static resource lists are limiting. Resource templates let servers expose parameterized URIs using [RFC 6570 URI templates](https://datatracker.ietf.org/doc/html/rfc6570):

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/templates/list"
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resourceTemplates": [
      {
        "uriTemplate": "postgres://db/tables/{table_name}/schema",
        "name": "Table schema",
        "description": "Schema for a specific database table",
        "mimeType": "application/json"
      }
    ]
  }
}
```

The client (or user) fills in `{table_name}`, and the resulting URI is passed to `resources/read`. This is how you expose "all database tables" without listing each one statically.

### Subscriptions

Resources can change. The spec supports a subscription mechanism for real-time updates:

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/subscribe",
  "params": { "uri": "file:///project/schema.sql" }
}
```

When the resource changes, the server pushes a notification:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": { "uri": "file:///project/schema.sql" }
}
```

The client then re-reads the resource if needed. This is opt-in - the server must declare `"subscribe": true` in its resource capabilities during initialization.

### Annotations

Resources support metadata annotations that help clients decide how to use them:

```json
{
  "uri": "file:///project/README.md",
  "name": "README.md",
  "mimeType": "text/markdown",
  "annotations": {
    "audience": ["user", "assistant"],
    "priority": 0.8,
    "lastModified": "2026-03-15T10:30:00Z"
  }
}
```

The `audience` field is interesting - a resource marked `["user"]` is meant for the human to see, not the model. One marked `["assistant"]` is context for the model. Clients can use `priority` (0.0 to 1.0) to decide what fits in a limited context window.

## Prompts - user-controlled templates

Prompts are pre-built message templates that users explicitly select. Think slash commands, menu items, or workflow starters. The server defines the template structure and arguments; the user picks which prompt to use; the client resolves it into messages for the model.

### Discovery: `prompts/list`

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "prompts/list"
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "prompts": [
      {
        "name": "code_review",
        "description": "Review code for bugs, style issues, and improvements",
        "arguments": [
          {
            "name": "code",
            "description": "The code to review",
            "required": true
          },
          {
            "name": "language",
            "description": "Programming language",
            "required": false
          }
        ]
      },
      {
        "name": "explain_error",
        "description": "Explain an error message and suggest fixes",
        "arguments": [
          {
            "name": "error",
            "description": "The error message or stack trace",
            "required": true
          }
        ]
      }
    ]
  }
}
```

### Retrieval: `prompts/get`

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": {
      "code": "fn main() { let x = vec![1,2,3]; println!(\"{}\", x[5]); }"
    }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "description": "Code review prompt",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Review this code for bugs, performance issues, and style problems. Be specific about line numbers and suggest fixes.\n\n```rust\nfn main() { let x = vec![1,2,3]; println!(\"{}\", x[5]); }\n```"
        }
      }
    ]
  }
}
```

The key difference from tools: `prompts/get` returns a `messages` array with `role` and `content` fields. These are conversation messages that get injected directly into the chat. The server constructs the exact prompt text - the user just picks the template and provides arguments.

Prompt messages can contain embedded resources too:

```json
{
  "role": "user",
  "content": {
    "type": "resource",
    "resource": {
      "uri": "file:///project/src/main.rs",
      "mimeType": "text/x-rust",
      "text": "fn main() {\n    println!(\"Hello\");\n}"
    }
  }
}
```

This lets prompts pull in server-side data as part of the conversation template - combining the data access of resources with the structured interaction of prompts.

## Capabilities negotiation

Before any primitives can be used, the client and server exchange capabilities during initialization. This is the very first message in any MCP connection:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "elicitation": {}
    },
    "clientInfo": {
      "name": "my-editor",
      "version": "2.1.0"
    }
  }
}
```

The server responds with its own capabilities:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "prompts": { "listChanged": true }
    },
    "serverInfo": {
      "name": "my-mcp-server",
      "version": "1.0.0"
    }
  }
}
```

After this, the client sends a `notifications/initialized` notification, and the connection enters the operational phase. The capabilities object tells both sides what is available - a client should not call `tools/list` if the server did not declare `tools` in its capabilities.

Each capability key has sub-options. For resources: `subscribe` (can clients subscribe to changes?) and `listChanged` (will the server notify about list changes?). For tools and prompts: just `listChanged`.

The lifecycle is: **Initialize** (negotiate) -> **Operate** (exchange messages) -> **Shutdown** (graceful close).

## Building it in Rust with rmcp

The [official Rust MCP SDK](https://github.com/modelcontextprotocol/rust-sdk) (`rmcp` on [crates.io](https://crates.io/crates/rmcp)) provides macros that generate the JSON-RPC plumbing. Here is a server that exposes all three primitives:

```toml
# Cargo.toml
[dependencies]
rmcp = { version = "0.11", features = ["transport-io", "server"] }
serde = { version = "1", features = ["derive"] }
schemars = "0.8"
tokio = { version = "1", features = ["full"] }
```

The server struct and handler:

```rust
use rmcp::prelude::*;
use rmcp::model::*;

struct DevServer;

#[tool_router]
impl DevServer {
    #[tool(description = "Run a shell command in the project directory")]
    async fn run_command(
        &self,
        Parameters(params): Parameters<RunCommandParams>,
    ) -> Result<CallToolResult, McpError> {
        let output = tokio::process::Command::new("sh")
            .arg("-c")
            .arg(&params.command)
            .output()
            .await
            .map_err(|e| McpError::internal_error(e.to_string(), None))?;

        let stdout = String::from_utf8_lossy(&output.stdout);
        let stderr = String::from_utf8_lossy(&output.stderr);

        let text = if stderr.is_empty() {
            stdout.to_string()
        } else {
            format!("stdout:\n{stdout}\nstderr:\n{stderr}")
        };

        Ok(CallToolResult::success(vec![Content::text(text)]))
    }
}

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
struct RunCommandParams {
    #[schemars(description = "Shell command to execute")]
    command: String,
}
```

The `#[tool_router]` macro scans the `impl` block for `#[tool]` methods and generates a `ToolRouter<DevServer>` that dispatches `tools/call` requests. The `Parameters<T>` extractor deserializes and validates input against the JSON Schema derived from `schemars::JsonSchema`.

For the server handler, you declare capabilities and wire everything up:

```rust
#[tool_handler]
impl ServerHandler for DevServer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo {
            protocol_version: ProtocolVersion::V_2025_06_18,
            capabilities: ServerCapabilities::builder()
                .enable_tools()
                .build(),
            server_info: Implementation {
                name: "dev-server".into(),
                version: "0.1.0".into(),
            },
            instructions: Some(
                "Server for running project commands".into()
            ),
        }
    }
}
```

To run over stdio (the simplest transport, perfect for local use):

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let server = DevServer;
    let service = server.serve(rmcp::transport::stdio()).await?;
    service.waiting().await?;
    Ok(())
}
```

To add resources and prompts, enable them in capabilities:

```rust
ServerCapabilities::builder()
    .enable_tools()
    .enable_resources()
    .enable_prompts()
    .build()
```

Then implement the corresponding router macros (`#[resource_router]`, `#[prompt_router]`) on your server struct. The pattern is the same - the macro generates the JSON-RPC dispatch logic, you write the business logic.

## Decision framework

When you are designing an MCP server, ask three questions for each piece of functionality:

**1. Does it have side effects?**

If yes, it is a tool. Creating a GitHub issue, sending an email, writing a file, executing a query that modifies data - these are all tools. The model decides when to invoke them, and the client should show confirmation UI.

**2. Is it read-only data the model or app needs for context?**

If yes, it is a resource. Database schemas, file contents, configuration, API documentation, git history. The host application decides what to include in the model's context. Resources do not change state.

**3. Is it a structured interaction pattern the user should explicitly choose?**

If yes, it is a prompt. Code review templates, debugging workflows, migration guides, onboarding checklists. The user picks the prompt (like a slash command), provides arguments, and the server constructs the messages.

Some real-world examples:

| What you are building                     | Primitive | Why                                                 |
|------------------------------------------|-----------|-----------------------------------------------------|
| Execute SQL query                         | Tool      | Side effect (even SELECT can be expensive/slow)     |
| Expose database schema                    | Resource  | Read-only context the app can attach                |
| "Analyze this table" workflow             | Prompt    | User picks it, server builds the instruction set    |
| Create Jira ticket                        | Tool      | Side effect (creates external state)                |
| List open Jira tickets                    | Resource  | Read-only data for context                          |
| "Triage these bugs" template              | Prompt    | Structured interaction the user initiates           |
| Read file contents                        | Resource  | Read-only, URI-addressable data                     |
| Write to a file                           | Tool      | Side effect (modifies filesystem)                   |
| "Review this PR" workflow                 | Prompt    | User-initiated, returns structured messages         |

The boundary between tool and resource can be blurry for read operations. A `SELECT * FROM users` could be either. The deciding factor is control: if the model should be able to query arbitrary data on demand, make it a tool. If the application should pre-load specific data into context, make it a resource.

## What the spec does not cover

MCP intentionally stays out of a few areas:

- **How the model uses context.** MCP delivers tools, resources, and prompts to the client. What the host application does with them (prompt construction, context window management, tool call routing) is outside the spec.
- **Authentication between client and server.** The transport layer recommends OAuth for HTTP-based servers, but the data layer protocol has no auth concepts. This is a transport concern.
- **Tool execution sandboxing.** The spec says "tools represent arbitrary code execution" and recommends human-in-the-loop confirmation. But it does not define a sandbox model. That is the host's responsibility.

These are deliberate scope boundaries, not gaps. MCP is a context protocol, not an agent framework. It defines how to move context between systems - what happens with that context is up to the implementation.

## Further reading

- [MCP Specification (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25) - the authoritative protocol reference
- [Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture) - client-host-server model, lifecycle, layers
- [Official Rust SDK (rmcp)](https://github.com/modelcontextprotocol/rust-sdk) - the `rmcp` crate with macro-based server development
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector) - dev tool for testing MCP servers interactively
- [Tool annotations blog post](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/) - deep dive into behavioral hints and their UX implications
