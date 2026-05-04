+++
title = "MCP - what it actually is and why it matters"
date = 2025-02-26
description = "The Model Context Protocol gives AI agents a standard way to call tools and read data. Here is how it works, from stdio to Streamable HTTP, with a Rust server from scratch."

[taxonomies]
tags = ["mcp", "rust", "protocols", "architecture"]
+++

Every AI coding assistant, every chatbot with tool access, every agent that can "do things" - they all face the same integration problem. The LLM needs to talk to external systems. Databases, APIs, file systems, git repos, issue trackers. Without a standard protocol, every combination of AI app and external system needs its own custom connector. Five AI apps times ten external systems equals fifty integrations. Add another app? Ten more. Add another system? Five more. This is the N×M problem that the Model Context Protocol solves.

MCP is an open protocol, created by Anthropic and released in November 2024, that standardizes how LLM applications connect to external data and tools. Think of it as the USB-C of AI integrations - a single interface that any AI app can use to talk to any compatible server. Instead of N×M custom integrations, you get N clients and M servers that all speak the same language.

That sounds simple. The reality has a few more moving parts.

<!-- more -->

## The architecture: hosts, clients, servers

MCP follows a client-server model with three distinct roles:

- **Host** - the AI application. Claude Desktop, VS Code with Copilot, Cursor, your custom agent framework. The host is the thing the user interacts with.
- **Client** - a connector inside the host. Each host creates one client per server connection. The client manages the lifecycle, sends requests, receives responses.
- **Server** - a program that exposes tools, data, and prompt templates to clients. A server can be a local process on your machine or a remote HTTP service.

The relationship is one host to many clients, one client to one server. When Claude Desktop connects to three MCP servers (say, filesystem, GitHub, and a database), it spawns three separate clients internally, each maintaining its own connection.

```
┌─────────────────────────────────────┐
│           Host (Claude Desktop)     │
│                                     │
│  ┌─────────┐ ┌─────────┐ ┌────────┐│
│  │ Client 1│ │ Client 2│ │Client 3││
│  └────┬────┘ └────┬────┘ └───┬────┘│
└───────┼───────────┼──────────┼─────┘
        │           │          │
   ┌────▼────┐ ┌────▼───┐ ┌───▼─────┐
   │Filesystem│ │ GitHub │ │Database │
   │ Server   │ │ Server │ │ Server  │
   └─────────┘ └────────┘ └─────────┘
```

This separation matters. The host controls what the user sees. The client handles protocol details. The server provides capabilities. A server author doesn't need to know which host will connect - they just implement the MCP interface. A host author doesn't need to know what servers exist - they just speak MCP.

## JSON-RPC 2.0 under the hood

All MCP communication uses [JSON-RPC 2.0](https://www.jsonrpc.org/). Every message between client and server is a JSON object with three possible shapes:

**Request** - has an `id`, expects a response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}
```

**Response** - matches a request by `id`, carries `result` or `error`:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [{ "name": "read_file", "description": "..." }]
  }
}
```

**Notification** - no `id`, fire-and-forget:
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

The protocol is stateful. Every connection starts with an initialization handshake where client and server exchange capabilities - what primitives the server offers (tools, resources, prompts), what features the client supports (sampling, elicitation). After that comes the operational phase where actual work happens. Then a graceful shutdown when done.

This design was directly inspired by the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) (LSP), which did the same thing for code editors and programming language tooling. Before LSP, every editor needed a custom plugin for every language. After LSP, you write one language server and it works in VS Code, Neovim, Emacs, everything. MCP applies the same idea to AI applications and their tools.

## The three primitives

MCP servers expose functionality through three primitives. Each one has a different control model:

| Primitive  | Controlled by      | Purpose                              |
|------------|-------------------|---------------------------------------|
| **Tools**  | The AI model       | Functions the LLM decides to call     |
| **Resources** | The application | Data the host attaches to context     |
| **Prompts** | The user          | Templates the user explicitly picks   |

**Tools** are the most common. They are functions with side effects - create a file, run a query, send a message, call an API. The LLM sees the tool's name, description, and input schema, then decides whether and when to invoke it. When you ask Claude to "create a GitHub issue," it's calling a tool.

**Resources** are read-only data identified by URIs. File contents, database schemas, configuration files. The host application decides which resources to pull into the context window - the model doesn't choose.

**Prompts** are reusable message templates. Think slash commands - the user picks a prompt, provides arguments, and the server constructs structured messages for the model.

The distinction between these three isn't cosmetic. Tools use `tools/call`, resources use `resources/read`, prompts use `prompts/get`. They have different discovery methods, different response formats, and different security implications. A tool that executes shell commands needs human-in-the-loop confirmation. A resource that serves a README does not.

I'll go deeper into each primitive in a future post. For now, the important thing is that these three cover the full spectrum of what an AI agent needs from external systems: the ability to act (tools), the ability to read (resources), and structured interaction patterns (prompts).

## Transport: stdio vs Streamable HTTP

The protocol defines two standard transports. This is where MCP goes from abstract spec to concrete wires.

### stdio

The simplest transport. The client launches the MCP server as a child process and communicates through standard input/output. The client writes JSON-RPC messages to the server's stdin. The server writes responses to stdout. Messages are delimited by newlines.

```
Client                          Server (child process)
  │                                │
  │──── JSON-RPC via stdin ───────>│
  │                                │
  │<──── JSON-RPC via stdout ──────│
  │                                │
  │     (stderr for logging) ──────│
```

That's it. No network, no ports, no TLS. The server's stderr can be used for logging but must never contain MCP messages. The server's stdout must never contain anything that isn't a valid JSON-RPC message.

stdio is the default for local MCP servers. When Claude Desktop launches a filesystem server on your laptop, it uses stdio. The server runs with the same permissions as the host app, on the same machine. There's no network attack surface.

The tradeoff: stdio is one client per server. You can't share a stdio server across multiple hosts. If both Claude Desktop and VS Code want to use the same server, they each spawn their own instance.

### Streamable HTTP

For remote servers and multi-client scenarios, MCP uses Streamable HTTP (introduced in the [March 2025 spec update](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports), replacing the earlier HTTP+SSE transport).

The server exposes a single HTTP endpoint - say, `https://api.example.com/mcp`. This endpoint handles both POST and GET:

- **POST** for sending JSON-RPC messages from client to server. The client sends a request; the server responds with either a regular JSON response or upgrades to a Server-Sent Events (SSE) stream for streaming results.
- **GET** for opening a long-lived SSE stream where the server can push notifications and requests to the client.

The client sends requests like:

```http
POST /mcp HTTP/1.1
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Session-Id: 1868a90c-bf1a-4c5e-9e2b-3a7f4d8c9012
MCP-Protocol-Version: 2025-11-25

{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": { "sql": "SELECT count(*) FROM users" }
  }
}
```

The server can respond in one of two ways. For simple requests, a regular JSON response. For longer operations, it upgrades to SSE:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

id: evt_001
data:

id: evt_002
data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"t1","progress":50,"total":100}}

id: evt_003
data: {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"42"}]}}
```

Streamable HTTP supports session management (via `MCP-Session-Id` header), resumable connections (via SSE event IDs and `Last-Event-ID`), and standard HTTP authentication. The spec mandates that servers validate the `Origin` header to prevent DNS rebinding attacks and respond with 403 on invalid origins.

The key difference from the old HTTP+SSE transport: the server no longer needs to maintain a permanent long-lived connection. It can close the SSE stream and let the client reconnect (poll) with `Last-Event-ID` to resume where it left off. This makes it much more practical for serverless deployments and load-balanced environments.

### When to use which

**stdio** when:
- The server runs locally on the user's machine
- You're building a personal tool (filesystem access, local database, local scripts)
- You want zero configuration - no ports, no auth, no TLS

**Streamable HTTP** when:
- The server is remote (cloud-hosted, shared service)
- Multiple clients need to connect to the same server
- You need authentication, CORS, session management
- You're deploying to a serverless platform

In practice, many production setups use both. A local stdio server handles filesystem access and local tools, while remote Streamable HTTP servers provide cloud API integrations. The protocol layer is identical - the same JSON-RPC messages, the same primitives - only the transport differs.

## Configuring MCP servers

How do you actually connect an MCP server to your AI app? Here's what it looks like in Claude Desktop. The configuration lives in `claude_desktop_config.json`:

**macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

A stdio server that gives Claude access to your filesystem:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/you/projects",
        "/Users/you/documents"
      ]
    }
  }
}
```

The host launches `npx @modelcontextprotocol/server-filesystem` as a child process and talks to it via stdio. The server gets access to the specified directories.

Multiple servers, with environment variables for secrets:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/you/projects"]
    },
    "github": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-e", "GITHUB_PERSONAL_ACCESS_TOKEN",
        "ghcr.io/github/github-mcp-server"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_token_here"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

Each server entry specifies a `command` to run, `args` to pass, and optionally `env` for environment variables. Claude Desktop spawns each as a subprocess, connects via stdio, runs the initialization handshake, and the tools become available in your conversation.

For remote Streamable HTTP servers, the config is simpler - just a URL:

```json
{
  "mcpServers": {
    "sentry": {
      "url": "https://mcp.sentry.io/sse",
      "headers": {
        "Authorization": "Bearer your-sentry-token"
      }
    }
  }
}
```

Restart Claude Desktop after editing the config. The hammer icon in the bottom-left corner confirms how many MCP tools are connected.

## The ecosystem right now

The numbers tell the story of how fast this moved. MCP was released November 2024. By March 2025, OpenAI adopted it across their products including the ChatGPT desktop app. Google DeepMind followed in April 2025. AWS Bedrock added MCP support in November 2025. By early 2026, essentially every major AI provider supports it.

In December 2025, Anthropic [donated MCP to the Agentic AI Foundation](https://en.wikipedia.org/wiki/Model_Context_Protocol), a Linux Foundation initiative, making it a truly vendor-neutral standard.

The server ecosystem exploded accordingly. [mcp.so](https://mcp.so/) indexes over 19,000 servers. The [official MCP registry](https://registry.modelcontextprotocol.io/) grew from 90 to 518 servers in a single month. Smithery hosts 2,000+. There are over 5,800 community-built servers covering developer tools (1,200+), business applications (950+), web and search (600+), and AI/automation (450+).

The SDK side is equally broad. Official SDKs exist for TypeScript, Python, Java, Kotlin, C#, Go, Swift, Ruby, PHP, Perl, and Rust. The [Rust SDK](https://github.com/modelcontextprotocol/rust-sdk) (`rmcp` on crates.io) hit version 1.3.0 in March 2026, with 2.1 million monthly downloads and over 1,000 dependent crates.

Monthly SDK downloads across all languages crossed 97 million as of March 2026. That number was 22 million when OpenAI adopted MCP, 45 million when Microsoft integrated it into Copilot Studio, and 68 million when AWS joined.

This adoption curve matters because network effects compound. More servers mean more value for each host that speaks MCP. More hosts mean more incentive to build servers. The N×M problem that MCP was designed to solve gets worse as N and M grow - which means the protocol becomes more valuable, not less.

## Writing an MCP server in Rust

Enough theory. Here's a complete MCP server in Rust using the `rmcp` crate. This server exposes two tools: one for reading files within a designated directory, and one for listing directory contents.

Start with the dependencies:

```toml
[package]
name = "file-explorer-mcp"
version = "0.1.0"
edition = "2024"

[dependencies]
rmcp = { version = "1.3", features = ["server", "transport-io", "macros"] }
serde = { version = "1", features = ["derive"] }
schemars = "0.8"
tokio = { version = "1", features = ["full"] }
```

The `server` feature enables the server-side API. `transport-io` gives us stdio transport. `macros` provides `#[tool_router]` and `#[tool_handler]` for ergonomic handler definitions. `schemars` is needed because `rmcp` generates JSON Schema from Rust types to describe tool parameters to the LLM.

Now the server struct and its tools:

```rust
use rmcp::prelude::*;
use rmcp::model::*;
use std::path::{Path, PathBuf};

struct FileExplorer {
    base_dir: PathBuf,
}

impl FileExplorer {
    fn new(base_dir: PathBuf) -> Self {
        Self { base_dir }
    }

    fn validate_path(&self, requested: &str) -> Result<PathBuf, McpError> {
        if requested.contains('\0') {
            return Err(McpError::invalid_params("null byte in path", None));
        }

        let candidate = self.base_dir.join(requested);
        let canonical = candidate.canonicalize().map_err(|_| {
            McpError::invalid_params(
                format!("path not found: {requested}"),
                None,
            )
        })?;

        if !canonical.starts_with(&self.base_dir) {
            return Err(McpError::invalid_params(
                "path outside allowed directory",
                None,
            ));
        }

        Ok(canonical)
    }
}
```

The `validate_path` method does canonical path resolution. It joins the requested path with the base directory, resolves symlinks and `..` components via `canonicalize()`, then checks the result still starts with the base directory. This prevents path traversal - `../../../etc/passwd` resolves to `/etc/passwd`, which doesn't start with the base directory, so it's rejected.

Now the tool definitions:

```rust
#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
struct ReadFileParams {
    /// Relative path to the file within the allowed directory
    #[schemars(description = "Relative file path to read")]
    path: String,
}

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
struct ListDirParams {
    /// Relative path to the directory (empty string for root)
    #[schemars(description = "Relative directory path to list")]
    path: String,
}

#[tool_router]
impl FileExplorer {
    #[tool(description = "Read the contents of a file")]
    async fn read_file(
        &self,
        Parameters(params): Parameters<ReadFileParams>,
    ) -> Result<CallToolResult, McpError> {
        let validated = self.validate_path(&params.path)?;

        if !validated.is_file() {
            return Err(McpError::invalid_params("not a file", None));
        }

        let content = tokio::fs::read_to_string(&validated)
            .await
            .map_err(|e| {
                McpError::internal_error(format!("read failed: {e}"), None)
            })?;

        // Guard against huge files blowing up the context window
        if content.len() > 500_000 {
            return Ok(CallToolResult::success(vec![Content::text(
                format!(
                    "File too large ({} bytes). First 500 bytes:\n{}",
                    content.len(),
                    &content[..500]
                ),
            )]));
        }

        Ok(CallToolResult::success(vec![Content::text(content)]))
    }

    #[tool(description = "List files and directories at a path")]
    async fn list_dir(
        &self,
        Parameters(params): Parameters<ListDirParams>,
    ) -> Result<CallToolResult, McpError> {
        let validated = if params.path.is_empty() {
            self.base_dir.clone()
        } else {
            self.validate_path(&params.path)?
        };

        if !validated.is_dir() {
            return Err(McpError::invalid_params("not a directory", None));
        }

        let mut entries = Vec::new();
        let mut reader = tokio::fs::read_dir(&validated).await.map_err(|e| {
            McpError::internal_error(format!("readdir failed: {e}"), None)
        })?;

        while let Some(entry) = reader.next_entry().await.map_err(|e| {
            McpError::internal_error(format!("entry error: {e}"), None)
        })? {
            let file_type = entry.file_type().await.unwrap_or_else(|_| {
                // Fallback - treat as file if we can't determine type
                std::fs::metadata(entry.path())
                    .map(|m| m.file_type())
                    .unwrap_or_else(|_| std::fs::metadata("/dev/null").unwrap().file_type())
            });

            let name = entry.file_name().to_string_lossy().to_string();
            let suffix = if file_type.is_dir() { "/" } else { "" };
            entries.push(format!("{name}{suffix}"));
        }

        entries.sort();
        Ok(CallToolResult::success(vec![Content::text(
            entries.join("\n"),
        )]))
    }
}
```

The `#[tool_router]` macro scans every `#[tool]` method in the impl block and generates the dispatching logic for `tools/call` and `tools/list`. When a client sends `tools/list`, the macro-generated code returns the tool names, descriptions, and JSON Schemas derived from the `schemars::JsonSchema` implementation on the parameter structs.

`Parameters<T>` is an extractor - it deserializes the incoming JSON arguments into your typed struct and validates them against the schema. If the arguments don't match, the client gets a JSON-RPC error before your handler ever runs.

The server handler wires up capabilities and metadata:

```rust
#[tool_handler]
impl ServerHandler for FileExplorer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo {
            protocol_version: ProtocolVersion::V_2025_11_25,
            capabilities: ServerCapabilities::builder()
                .enable_tools()
                .build(),
            server_info: Implementation {
                name: "file-explorer".into(),
                version: "0.1.0".into(),
            },
            instructions: Some(
                "Explore and read files within the configured directory.".into(),
            ),
        }
    }
}
```

`ServerCapabilities::builder().enable_tools().build()` tells clients "I have tools, come ask for them." If you also had resources, you'd chain `.enable_resources()`. The `instructions` field is free-form text that the host can feed to the LLM as system context about what this server does.

Finally, the main function:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let base_dir = std::env::args()
        .nth(1)
        .unwrap_or_else(|| ".".to_string());

    let base_dir = PathBuf::from(&base_dir)
        .canonicalize()
        .expect("base directory must exist");

    eprintln!("file-explorer MCP server starting");
    eprintln!("base directory: {}", base_dir.display());

    let server = FileExplorer::new(base_dir);
    let service = server.serve(rmcp::transport::stdio()).await?;
    service.waiting().await?;

    Ok(())
}
```

Notice `eprintln!` for startup messages - they go to stderr, which is fine. The server's stdout is reserved for MCP messages. `rmcp::transport::stdio()` creates the stdio transport. `.serve()` starts the server and returns a handle. `.waiting().await` blocks until the client disconnects.

To use this with Claude Desktop, you'd compile it and add it to your config:

```json
{
  "mcpServers": {
    "file-explorer": {
      "command": "/path/to/file-explorer-mcp",
      "args": ["/Users/you/projects"]
    }
  }
}
```

Claude Desktop launches the binary, passes the directory path as an argument, connects via stdio, and your tools appear in the conversation.

## Testing with MCP Inspector

Before connecting to a real host, the [MCP Inspector](https://github.com/modelcontextprotocol/inspector) lets you test your server interactively. It's a dev tool that connects to any MCP server, shows the capability negotiation, lets you browse tools/resources/prompts, and execute tool calls manually.

```bash
npx @modelcontextprotocol/inspector ./target/release/file-explorer-mcp /tmp/test-dir
```

This opens a browser UI where you can call `list_dir` and `read_file` with different arguments, see the raw JSON-RPC messages, and verify your server behaves correctly before any LLM touches it. Extremely useful for debugging schema issues and error handling.

## What MCP is not

A few things MCP deliberately stays out of:

**It's not an agent framework.** MCP moves context between systems. What the host does with that context - prompt construction, context window management, tool call planning, multi-step reasoning - is outside the spec. MCP doesn't know what a "chain of thought" is.

**It's not an auth standard.** The spec recommends OAuth 2.1 for HTTP-based servers and includes transport-level security requirements (Origin validation, session management). But authentication is a transport concern, not a protocol concern. A stdio server running on your laptop doesn't need OAuth.

**It's not a sandbox.** The spec says "tools represent arbitrary code execution" and recommends human-in-the-loop confirmation. But it doesn't define how to sandbox tool execution. That's on the host. When Claude Desktop shows you a confirmation dialog before running a tool, that's Claude Desktop's policy, not something MCP mandates.

These are deliberate design choices. By keeping the scope narrow - context exchange between AI apps and external systems - MCP stays implementable, composable, and transport-agnostic. An agent framework built on top of MCP can add planning, sandboxing, and auth however it wants, and the MCP layer underneath doesn't need to change.

## Where this is going

The [MCP specification](https://modelcontextprotocol.io/specification/2025-11-25) is currently at version 2025-11-25. The [2026 roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) lists deeper security and authorization work as a priority, along with improvements to the registry and discovery mechanisms. The protocol version `2025-06-18` is the latest negotiated version in client-server handshakes.

The ecosystem has real momentum. 97 million monthly SDK downloads. 19,000+ indexed servers. Every major AI provider on board. The Linux Foundation stewarding the spec. At this point, MCP isn't a bet - it's the standard. If you're building anything that connects AI to external systems, you're either speaking MCP or you're building a bridge to something that does.

The Rust SDK makes it straightforward to build servers that are fast, safe, and easy to distribute as single binaries. No runtime, no npm, no Python virtualenv - just a compiled binary that speaks stdio. For teams that care about deployment simplicity and security, that's a meaningful advantage.

Build something. The [rmcp crate](https://crates.io/crates/rmcp) and the [MCP Inspector](https://github.com/modelcontextprotocol/inspector) are all you need to get started.

## Further reading

- [MCP Specification (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25) - the authoritative protocol reference
- [Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture) - client-host-server model, lifecycle, layers
- [Transport specification](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports) - stdio and Streamable HTTP in detail
- [Official Rust SDK (rmcp)](https://github.com/modelcontextprotocol/rust-sdk) - v1.3.0, 2.1M monthly downloads
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector) - interactive dev tool for testing servers
- [Connecting local servers](https://modelcontextprotocol.io/docs/develop/connect-local-servers) - configuration guide for Claude Desktop
- [Model Context Protocol on Wikipedia](https://en.wikipedia.org/wiki/Model_Context_Protocol) - history and adoption timeline
