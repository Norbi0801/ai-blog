+++
title = "Building an MCP server in Rust with rmcp"
date = 2025-07-11
description = "A from-scratch tutorial on building a working MCP server in Rust using rmcp: defining tools, stdio transport, JSON Schema inputs, error handling, testing, and Claude Desktop integration."

[taxonomies]
tags = ["rust", "mcp", "rmcp", "tutorial"]
+++

MCP servers are just JSON-RPC processes. A client sends `tools/list`, your server responds with a list of tools. The client sends `tools/call` with a tool name and arguments, your server runs the logic and returns a result. That's the core loop.

The Rust ecosystem has an official SDK for this: [rmcp](https://github.com/modelcontextprotocol/rust-sdk), maintained under the `modelcontextprotocol` GitHub org. It handles the protocol framing, transport, and schema generation. You define tools as methods, derive input schemas from types, and let the macros wire everything together.

This post walks through building a complete MCP server - a notes manager that creates, lists, searches, and deletes notes. By the end you'll have a binary that plugs into Claude Desktop over stdio and handles real tool calls.

<!-- more -->

## Why Rust for MCP servers

Most MCP servers out there are TypeScript or Python. They work fine. But Rust gives you a few things that matter specifically for MCP:

**Single binary deployment.** No `npx -y` downloading packages at startup. No Python virtualenv activation. You `cargo build --release`, copy the binary, done. If you read the [MCP security checklist](/blog/mcp-security-checklist-10-things-to-check-before-deploying-your-server), you know that `npx` without version pins is a supply chain risk. A static binary eliminates that entire class of problem.

**Low resource usage.** MCP servers run as background processes - one per client session. A Rust binary idles at 2-4MB RSS. A Node.js process idles at 30-50MB. When Claude Desktop spawns five MCP servers, that difference adds up.

**Type safety for tool schemas.** rmcp generates JSON Schema from your Rust types at compile time. If you add a field to your input struct but forget to handle it, the compiler catches it. In TypeScript MCP servers, the Zod schema and the handler can drift apart silently.

## Project setup

```bash
cargo new mcp-notes --bin
cd mcp-notes
```

Your `Cargo.toml`:

```toml
[package]
name = "mcp-notes"
version = "0.1.0"
edition = "2024"

[dependencies]
rmcp = { version = "1.3", features = ["server", "transport-io"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
schemars = "0.8"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
anyhow = "1"
```

The `server` feature pulls in the `ServerHandler` trait, the tool macros, and the routing infrastructure. `transport-io` gives you `stdio()` - the transport that reads JSON-RPC from stdin and writes to stdout.

The `schemars` crate is what generates JSON Schema from your Rust types. rmcp re-exports it, but I prefer listing it explicitly since you'll be deriving `JsonSchema` on your own structs.

## The ServerHandler trait

Every rmcp server implements `ServerHandler`. This trait tells the framework what your server can do and how to handle requests. The critical method is `get_info()`:

```rust
use rmcp::{ServerHandler, model::*};

impl ServerHandler for NoteServer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo::new(
            ServerCapabilities::builder()
                .enable_tools()
                .build(),
        )
        .with_server_info(Implementation::from_build_env())
        .with_protocol_version(ProtocolVersion::V_2024_11_05)
        .with_instructions(
            "A notes management server. Create, list, search, and delete notes."
                .to_string(),
        )
    }
}
```

`ServerCapabilities::builder().enable_tools().build()` tells the client "this server exposes tools." Without `.enable_tools()`, the client won't even try to call `tools/list`. The `instructions` field is the system-level description the model sees - keep it short and factual.

Under the hood, `ServerHandler` has default implementations for `list_tools()`, `call_tool()`, and other MCP methods that return "method not found." The macros override these defaults, which is where the real magic happens.

## Defining the server struct

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::Mutex;
use rmcp::handler::server::router::tool::ToolRouter;

#[derive(Clone)]
pub struct NoteServer {
    notes: Arc<Mutex<HashMap<String, Note>>>,
    tool_router: ToolRouter<NoteServer>,
}

#[derive(Debug, Clone, serde::Serialize)]
pub struct Note {
    pub id: String,
    pub title: String,
    pub content: String,
    pub created_at: String,
}
```

Two things to note here. First, `ToolRouter<NoteServer>` is a field on the struct itself. The `#[tool_router]` macro generates a `Self::tool_router()` static method that builds this router, and you store the result during construction. This is how rmcp knows which methods are tools at runtime.

Second, the state (`notes`) is wrapped in `Arc<Mutex<_>>` because the server must be `Clone`. rmcp clones your handler when setting up the service. If you've worked with [axum's State extractor](https://docs.rs/axum/latest/axum/extract/struct.State.html), this pattern is familiar - shared state behind an `Arc`. I'm using `tokio::sync::Mutex` here rather than `std::sync::Mutex` because the lock is held across `.await` points in async handlers. If you need a refresher on why that matters, I covered tokio's concurrency primitives in [Understanding Tokio](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/).

## Defining tools

This is the core of the server. Each tool is a method annotated with `#[tool]` inside a `#[tool_router]` impl block:

```rust
use rmcp::{
    ErrorData as McpError, RoleServer,
    handler::server::wrapper::Parameters,
    model::*,
    tool, tool_handler, tool_router,
    schemars,
};

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct CreateNoteParams {
    #[schemars(description = "Title of the note")]
    pub title: String,
    #[schemars(description = "Content body of the note")]
    pub content: String,
}

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct GetNoteParams {
    #[schemars(description = "ID of the note to retrieve")]
    pub id: String,
}

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct DeleteNoteParams {
    #[schemars(description = "ID of the note to delete")]
    pub id: String,
}

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct SearchNotesParams {
    #[schemars(description = "Search query to match against note titles and content")]
    pub query: String,
}

#[tool_router]
impl NoteServer {
    pub fn new() -> Self {
        Self {
            notes: Arc::new(Mutex::new(HashMap::new())),
            tool_router: Self::tool_router(),
        }
    }

    #[tool(description = "Create a new note with a title and content. Returns the created note with its generated ID.")]
    async fn create_note(
        &self,
        Parameters(params): Parameters<CreateNoteParams>,
    ) -> Result<CallToolResult, McpError> {
        let id = format!("{:08x}", rand_id());
        let now = chrono::Utc::now().to_rfc3339();

        let note = Note {
            id: id.clone(),
            title: params.title,
            content: params.content,
            created_at: now,
        };

        self.notes.lock().await.insert(id.clone(), note.clone());

        let json = serde_json::to_string_pretty(&note)
            .map_err(|e| McpError::internal_error(format!("serialization failed: {e}"), None))?;

        Ok(CallToolResult::success(vec![Content::text(json)]))
    }

    #[tool(description = "List all notes. Returns an array of all stored notes.")]
    async fn list_notes(&self) -> Result<CallToolResult, McpError> {
        let notes = self.notes.lock().await;
        let all: Vec<&Note> = notes.values().collect();

        let json = serde_json::to_string_pretty(&all)
            .map_err(|e| McpError::internal_error(format!("serialization failed: {e}"), None))?;

        Ok(CallToolResult::success(vec![Content::text(json)]))
    }

    #[tool(description = "Get a single note by its ID.")]
    async fn get_note(
        &self,
        Parameters(params): Parameters<GetNoteParams>,
    ) -> Result<CallToolResult, McpError> {
        let notes = self.notes.lock().await;

        match notes.get(&params.id) {
            Some(note) => {
                let json = serde_json::to_string_pretty(note)
                    .map_err(|e| {
                        McpError::internal_error(format!("serialization failed: {e}"), None)
                    })?;
                Ok(CallToolResult::success(vec![Content::text(json)]))
            }
            None => Ok(CallToolResult::error(vec![Content::text(format!(
                "Note with id '{}' not found",
                params.id
            ))])),
        }
    }

    #[tool(description = "Delete a note by its ID. Returns confirmation or an error if the note doesn't exist.")]
    async fn delete_note(
        &self,
        Parameters(params): Parameters<DeleteNoteParams>,
    ) -> Result<CallToolResult, McpError> {
        let mut notes = self.notes.lock().await;

        match notes.remove(&params.id) {
            Some(note) => Ok(CallToolResult::success(vec![Content::text(format!(
                "Deleted note '{}'",
                note.title
            ))])),
            None => Ok(CallToolResult::error(vec![Content::text(format!(
                "Note with id '{}' not found",
                params.id
            ))])),
        }
    }

    #[tool(description = "Search notes by matching a query against titles and content. Case-insensitive.")]
    async fn search_notes(
        &self,
        Parameters(params): Parameters<SearchNotesParams>,
    ) -> Result<CallToolResult, McpError> {
        let notes = self.notes.lock().await;
        let query = params.query.to_lowercase();

        let matches: Vec<&Note> = notes
            .values()
            .filter(|n| {
                n.title.to_lowercase().contains(&query)
                    || n.content.to_lowercase().contains(&query)
            })
            .collect();

        let json = serde_json::to_string_pretty(&matches)
            .map_err(|e| McpError::internal_error(format!("serialization failed: {e}"), None))?;

        Ok(CallToolResult::success(vec![Content::text(json)]))
    }
}

fn rand_id() -> u32 {
    use std::collections::hash_map::RandomState;
    use std::hash::{BuildHasher, Hasher};
    RandomState::new().build_hasher().finish() as u32
}
```

Let's unpack what the macros do.

`#[tool_router]` scans the impl block for all `#[tool]`-annotated methods and generates a `Self::tool_router()` method that builds a `ToolRouter<NoteServer>`. This router holds a dispatch table: tool name to handler function, plus the `Tool` descriptors (name, description, inputSchema) for each tool.

`#[tool(description = "...")]` marks a method as a tool. The method name becomes the tool name (`create_note`, `list_notes`, etc.). The description is what the LLM sees when deciding which tool to call - make it specific and actionable.

`Parameters<T>` is the extractor for tool arguments. When the client sends `tools/call` with `{"name": "create_note", "arguments": {"title": "foo", "content": "bar"}}`, rmcp deserializes the `arguments` object into `T` (here, `CreateNoteParams`) and wraps it in `Parameters`. If deserialization fails, the framework returns an invalid params error before your handler runs.

Tools that take no arguments - like `list_notes` - just omit the `Parameters` argument entirely.

## What inputSchema actually generates

This is worth looking at because it's the contract between your server and the LLM. When the client calls `tools/list`, rmcp returns a `Tool` object for each method. For `create_note`, the generated schema looks like:

```json
{
  "name": "create_note",
  "description": "Create a new note with a title and content. Returns the created note with its generated ID.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "title": {
        "type": "string",
        "description": "Title of the note"
      },
      "content": {
        "type": "string",
        "description": "Content body of the note"
      }
    },
    "required": ["title", "content"]
  }
}
```

This comes from `schemars::JsonSchema` derived on `CreateNoteParams`. The `#[schemars(description = "...")]` attribute on each field becomes the `description` in the JSON Schema properties. Both fields are `String` (not `Option<String>`), so schemars puts them in `required`.

If you want optional fields:

```rust
#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct UpdateNoteParams {
    #[schemars(description = "ID of the note to update")]
    pub id: String,
    #[schemars(description = "New title (optional)")]
    pub title: Option<String>,
    #[schemars(description = "New content (optional)")]
    pub content: Option<String>,
}
```

This generates a schema where `id` is required but `title` and `content` are optional. The LLM sees this and knows it can update just the title without providing content.

You can also use enums, nested objects, and constrained types. schemars handles `Vec<T>` (as JSON arrays), `HashMap<String, T>` (as JSON objects with additionalProperties), and custom enum representations. The [schemars docs](https://docs.rs/schemars/0.8/schemars/) cover the full mapping.

One gotcha: the `description` on the struct itself (via `#[schemars(description = "...")]` on the struct) doesn't show up in the tool definition - only the `#[tool(description = "...")]` attribute matters there. The struct-level description goes into the schema's root, which most clients ignore.

## Error handling - two levels

MCP has two distinct error mechanisms, and confusing them is a common mistake.

**Protocol errors** mean the tool couldn't execute at all. Database connection failed, required parameter missing, internal panic. Return `Err(McpError)`:

```rust
// The framework wraps this in a JSON-RPC error response
return Err(McpError::internal_error(
    format!("database unreachable: {e}"),
    None,
));
```

The client sees a JSON-RPC error with a code and message. The LLM typically won't retry protocol errors.

**Tool execution errors** mean the tool ran, but the result is an error from the tool's domain. Note not found, search returned nothing, validation failed. Return `Ok(CallToolResult::error(...))`:

```rust
// The framework wraps this in a successful JSON-RPC response 
// with isError: true in the result
Ok(CallToolResult::error(vec![Content::text(format!(
    "Note with id '{}' not found",
    params.id
))]))
```

The client sees a successful JSON-RPC response, but the `isError: true` flag tells the LLM "this tool call didn't get what you wanted." The model can then decide to try again with different arguments or tell the user.

The distinction maps to HTTP status codes: protocol errors are 5xx (server broken), tool errors are 4xx-ish (your request was understood but couldn't be fulfilled). Use `Err(McpError)` for things the user can't fix. Use `CallToolResult::error()` for things the LLM might fix by adjusting its approach.

`McpError` (which is just a type alias for `rmcp::ErrorData`) has convenience constructors:

```rust
McpError::internal_error("something broke", None)     // code -32603
McpError::invalid_params("missing field 'id'", None)  // code -32602
McpError::invalid_request("bad JSON", None)            // code -32600
```

The second argument is `Option<serde_json::Value>` for attaching structured error data. Useful for returning validation details or debug context.

## The main function

```rust
use anyhow::Result;
use rmcp::ServiceExt;
use tracing_subscriber::EnvFilter;

#[tool_handler]
impl ServerHandler for NoteServer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo::new(
            ServerCapabilities::builder()
                .enable_tools()
                .build(),
        )
        .with_server_info(Implementation::from_build_env())
        .with_protocol_version(ProtocolVersion::V_2024_11_05)
        .with_instructions(
            "A notes management server. Create, list, search, and delete plain-text notes."
                .to_string(),
        )
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::from_default_env()
                .add_directive(tracing::Level::DEBUG.into()),
        )
        .with_writer(std::io::stderr)
        .with_ansi(false)
        .init();

    tracing::info!("starting mcp-notes server");

    let server = NoteServer::new();
    let service = server.serve(rmcp::transport::stdio()).await?;

    service.waiting().await?;
    Ok(())
}
```

Three critical details here.

**Logging goes to stderr.** This is mandatory, not optional. The MCP protocol uses stdout as the transport channel - JSON-RPC messages flow through it. If your logger writes to stdout, it corrupts the protocol stream and the client disconnects. `.with_writer(std::io::stderr)` redirects all tracing output to stderr. `.with_ansi(false)` disables color codes since stderr might not be a terminal. If you want a deeper look at the tracing ecosystem, I covered it in [Structured logging in Rust with tracing](/blog/structured-logging-in-rust-with-tracing).

**`#[tool_handler]`** on the `ServerHandler` impl is what generates the `list_tools()` and `call_tool()` implementations. Without it, your server would report that it supports tools (via `enable_tools()`) but return "method not found" when the client actually tries to list or call them. The macro connects the `ToolRouter` from `#[tool_router]` to the `ServerHandler` trait methods.

**`service.waiting().await?`** blocks until the client disconnects or the transport closes. For stdio, this means the server runs until the parent process (Claude Desktop, claude cli, etc.) terminates it. The `.serve()` method performs the MCP initialization handshake - the client sends `initialize`, the server responds with capabilities, the client sends `initialized` notification - and then enters the request loop.

## Claude Desktop integration

Build a release binary:

```bash
cargo build --release
```

Then edit your Claude Desktop config. The file location depends on your OS:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux:** `~/.config/claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "notes": {
      "command": "/absolute/path/to/target/release/mcp-notes",
      "args": []
    }
  }
}
```

Use the absolute path to the binary. Relative paths are resolved from Claude Desktop's working directory, which is unpredictable.

During development, you can point directly at `cargo run`:

```json
{
  "mcpServers": {
    "notes": {
      "command": "cargo",
      "args": ["run", "--release", "--manifest-path", "/absolute/path/to/mcp-notes/Cargo.toml"]
    }
  }
}
```

After saving the config, **fully restart Claude Desktop** (not just close the window - quit the app). Any JSON syntax error in the config silently disables all servers. If your server doesn't appear, check the config for trailing commas or missing quotes first.

Once it's loaded, you'll see "notes" in the tools list. Ask Claude something like "create a note titled 'meeting notes' with the content 'discuss Q3 roadmap'" and it will call your `create_note` tool.

## Testing

You don't need Claude Desktop to test your server. rmcp supports in-process testing with `tokio::io::duplex`, which creates an in-memory bidirectional byte stream - one end for the server, one for a test client.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rmcp::{ClientHandler, ServiceExt};

    #[derive(Default, Clone)]
    struct TestClient;
    impl ClientHandler for TestClient {}

    #[tokio::test]
    async fn test_create_and_list_notes() -> anyhow::Result<()> {
        let server = NoteServer::new();
        let client = TestClient::default();

        let (server_io, client_io) = tokio::io::duplex(4096);

        let server_handle = tokio::spawn(async move {
            let svc = server.serve(server_io).await?;
            svc.waiting().await?;
            anyhow::Ok(())
        });

        let client_svc = client.serve(client_io).await?;

        // Create a note
        let result = client_svc
            .call_tool(CallToolRequestParams {
                name: "create_note".into(),
                arguments: Some(serde_json::json!({
                    "title": "test note",
                    "content": "hello world"
                }).as_object().unwrap().clone()),
                ..Default::default()
            })
            .await?;
        assert!(!result.is_error.unwrap_or(false));

        // List notes
        let result = client_svc
            .call_tool(CallToolRequestParams {
                name: "list_notes".into(),
                arguments: None,
                ..Default::default()
            })
            .await?;
        assert!(!result.is_error.unwrap_or(false));

        let text = match &result.content[0] {
            Content::Text(t) => &t.text,
            _ => panic!("expected text content"),
        };
        assert!(text.contains("test note"));

        // Clean up
        client_svc.cancel().await?;
        let _ = server_handle.await;
        Ok(())
    }

    #[tokio::test]
    async fn test_get_nonexistent_note() -> anyhow::Result<()> {
        let server = NoteServer::new();
        let client = TestClient::default();

        let (server_io, client_io) = tokio::io::duplex(4096);

        let server_handle = tokio::spawn(async move {
            let svc = server.serve(server_io).await?;
            svc.waiting().await?;
            anyhow::Ok(())
        });

        let client_svc = client.serve(client_io).await?;

        let result = client_svc
            .call_tool(CallToolRequestParams {
                name: "get_note".into(),
                arguments: Some(serde_json::json!({
                    "id": "nonexistent"
                }).as_object().unwrap().clone()),
                ..Default::default()
            })
            .await?;

        // Tool execution error, not protocol error
        assert!(result.is_error.unwrap_or(false));

        client_svc.cancel().await?;
        let _ = server_handle.await;
        Ok(())
    }
}
```

The test flow mirrors what Claude Desktop does: the client connects, the MCP handshake happens automatically, and then you send `call_tool` requests. The `duplex(4096)` buffer size is fine for testing - MCP messages are small.

The `TestClient` is a minimal `ClientHandler` with no capabilities. It just needs to exist so the handshake completes. The `client_svc.call_tool()` method is a convenience wrapper that constructs the JSON-RPC request, sends it, and deserializes the response.

Note the distinction in assertions: `is_error.unwrap_or(false)` checks the tool-level error flag, not whether the RPC call itself failed. If the RPC failed (protocol error), `.call_tool()` would return `Err`, not `Ok` with `is_error: true`.

## What happens on the wire

To see the actual JSON-RPC messages, you can use the [MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector) or just log the raw bytes. Here's the flow for a `create_note` call:

```
Client -> Server:
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}}}

Server -> Client:
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{}},"serverInfo":{"name":"mcp-notes","version":"0.1.0"}}}

Client -> Server:
{"jsonrpc":"2.0","method":"notifications/initialized"}

Client -> Server:
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}

Server -> Client:
{"jsonrpc":"2.0","id":2,"result":{"tools":[{"name":"create_note","description":"Create a new note...","inputSchema":{...}},...]}}

Client -> Server:
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"create_note","arguments":{"title":"meeting notes","content":"discuss Q3 roadmap"}}}

Server -> Client:
{"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"{\"id\":\"a1b2c3d4\",...}"}],"isError":false}}
```

The first three messages are the initialization handshake. Then the client discovers available tools via `tools/list`. Finally it calls a specific tool with arguments. Each message is newline-delimited JSON on the stdio stream.

## Deployment

For production, build a static binary:

```bash
cargo build --release --target x86_64-unknown-linux-musl
```

The musl target produces a fully static binary with no dynamic library dependencies. Check the size:

```bash
ls -lh target/x86_64-unknown-linux-musl/release/mcp-notes
# typically 4-8 MB depending on dependencies
```

You can strip it further:

```toml
# Cargo.toml
[profile.release]
strip = true
lto = true
codegen-units = 1
```

This gets you down to 2-4 MB typically. Small enough to embed in a Docker image, distribute as a GitHub release artifact, or ship via Homebrew.

For Docker:

```dockerfile
FROM rust:1.87-alpine AS builder
RUN apk add --no-cache musl-dev
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/mcp-notes /mcp-notes
ENTRYPOINT ["/mcp-notes"]
```

A `scratch` base image with just your binary. No shell, no package manager, no attack surface. This works for stdio transport since the container's stdin/stdout become the transport channel.

For HTTP transport (if you later add `transport-streamable-http-server`), you'd expose a port and run behind a reverse proxy with TLS. But for Claude Desktop integration, stdio is the standard - and the simpler option.

## Where to go from here

This server stores notes in memory - they vanish when the process stops. For persistence, swap the `HashMap` for SQLite (via `rusqlite` or `sqlx`) or write to a file. The tool handlers don't change, just the storage layer behind `self.notes`.

You can also add [resources](https://modelcontextprotocol.io/docs/concepts/resources) (read-only data the model can reference) and [prompts](https://modelcontextprotocol.io/docs/concepts/prompts) (reusable prompt templates). rmcp has `#[prompt]` and resource handler support following the same macro pattern as tools.

Before deploying anything to production, run through the [MCP security checklist](/blog/mcp-security-checklist-10-things-to-check-before-deploying-your-server) - input validation, response size limits, logging, and the rest. The security surface of MCP servers is different from normal web services, and the checklist covers the specific things that go wrong.

The full source for this post is straightforward enough to keep in a single `main.rs`. Once you're past five or six tools, split into modules - one file per tool group, a shared `state.rs` for the server struct. rmcp doesn't impose a file structure; you organize however makes sense for your project.

The [rmcp repository](https://github.com/modelcontextprotocol/rust-sdk) has more examples: a counter server, a memory/knowledge-graph server, and transport examples for TCP, WebSocket, and HTTP. The counter example in particular is a good minimal reference if you want to cross-check the patterns shown here.
