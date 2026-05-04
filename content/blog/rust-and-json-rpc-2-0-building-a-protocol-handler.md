+++
title = "Rust and JSON-RPC 2.0 - building a protocol handler"
date = 2025-12-29
description = "JSON-RPC is the boring, twenty-year-old protocol that quietly powers LSP, MCP, Ethereum, and Bitcoin. Here is how the spec actually works on the wire, and how to build a typed handler in Rust with serde - including batches, notifications, and proper error codes."

[taxonomies]
tags = ["rust", "json-rpc", "protocols", "mcp"]
+++

JSON-RPC is the protocol nobody talks about until you realize you have been using it all day. Your editor talks to rust-analyzer over JSON-RPC. Claude Desktop talks to MCP servers over JSON-RPC. Every Ethereum node, every Bitcoin node, every Cosmos chain exposes a JSON-RPC endpoint. Solana too. It is everywhere, and it has been since 2010 when version 2.0 of the spec was finalized.

The reason it survived two decades of REST hype, GraphQL hype, and gRPC hype is that the spec fits on a napkin. There is no IDL, no code generator, no schema registry, no TLS dance, no HTTP/2 requirement. There is a JSON envelope with four reserved field names, a list of seven error codes, and a rule about a magic string `"jsonrpc": "2.0"`. That is the entire protocol.

This post walks through the spec on the wire, builds a typed handler in Rust with serde, and looks at how LSP, MCP, and the Ethereum node operators all use the same protocol but with very different conventions. If you are not familiar with how serde models data shapes, [Serde deep dive - beyond derive](/blog/serde-deep-dive-beyond-derive/) is useful background - we will lean on enum representations heavily.

<!-- more -->

## The spec, on the wire

The full [JSON-RPC 2.0 spec](https://www.jsonrpc.org/specification) is roughly 1500 words. There are exactly four shapes you need to know.

A **request**:

```json
{"jsonrpc": "2.0", "method": "subtract", "params": [42, 23], "id": 1}
```

A **response** (success):

```json
{"jsonrpc": "2.0", "result": 19, "id": 1}
```

A **response** (error):

```json
{"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": 1}
```

A **notification** (a request with no `id`, expects no response):

```json
{"jsonrpc": "2.0", "method": "tick", "params": null}
```

That is it. The transport is unspecified - JSON-RPC does not care whether you send these bytes over HTTP POST, a TCP socket, stdin/stdout pipes, or carrier pigeon. Most implementations frame messages with `Content-Length` headers (LSP, MCP stdio) or one-message-per-HTTP-POST (Ethereum, Bitcoin).

Two things in the spec trip people up the first time they implement it.

**`id` matters more than it looks.** It can be a string, a number, or `null`. The server must echo it back unchanged. If the request was a notification (no `id` at all), there is no response. If you cannot even parse the request well enough to know what `id` it had, the response `id` is `null`. This is the only time `null` is a valid response `id`.

**`params` is optional and polymorphic.** It can be an array (positional arguments) or an object (named arguments) or absent entirely. `null` is technically allowed but the spec says SHOULD NOT. A server that supports both has to handle both. This is the part that makes typed handlers in Rust interesting.

## Modeling the envelope in Rust

We want types that round-trip the wire format with `#[derive(Serialize, Deserialize)]` and reject malformed input at the type system level. Three things to handle:

1. The `jsonrpc` field is always the literal string `"2.0"`.
2. `id` is `String | Number | Null` and we need to preserve which one we got.
3. `params` is `Array | Object | Absent`.
4. A response has either `result` or `error`, never both.

```rust
use serde::{Deserialize, Serialize};
use serde_json::Value;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum Version {
    #[serde(rename = "2.0")]
    V2,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(untagged)]
pub enum Id {
    Num(i64),
    Str(String),
    Null,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Request {
    pub jsonrpc: Version,
    pub method: String,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub params: Option<Value>,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub id: Option<Id>,
}
```

The `Version` enum with a single `#[serde(rename = "2.0")]` variant is a small but important trick. Deserialization fails with a useful error message if the field is missing or has any other value. You cannot construct a `Version` with the wrong string. The protocol invariant is encoded in the type.

`Id` uses `#[serde(untagged)]` so serde tries each variant in order until one matches. The order matters - `Num` is tried before `Str` because every JSON number is also valid input for `Str` if you let it be (it would not, but with custom impls people make this mistake). Putting `Null` last means `null` deserializes to the explicit `Null` variant rather than `None` from the outer `Option`.

The `Option<Id>` distinguishes a missing field (notification) from `id: null` (request with explicit null id). Without the `default` attribute, serde would error on a missing field. With `skip_serializing_if = "Option::is_none"`, we omit `id` entirely when it is a notification, which is what the spec wants.

For responses, the `result` xor `error` constraint is harder to enforce at the type level if you also want plain derives. The pragmatic encoding:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Response {
    pub jsonrpc: Version,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub result: Option<Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error: Option<RpcError>,
    pub id: Id,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RpcError {
    pub code: i32,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub data: Option<Value>,
}
```

If you want to make the invariant unforgeable, model the body as an enum with `#[serde(untagged)]`:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum ResponseBody {
    Success { result: Value },
    Failure { error: RpcError },
}
```

Then `Response { jsonrpc, body, id }` and you literally cannot construct a response with both fields. The downside is slightly noisier deserialization errors when neither shape matches.

## The error codes

The spec reserves seven codes in the `-32000` range:

| Code | Meaning |
|------|---------|
| -32700 | Parse error - invalid JSON |
| -32600 | Invalid Request - JSON is not a valid Request object |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |
| -32000 to -32099 | Server error (implementation-defined) |

Codes outside `-32768` to `-32000` are open for application use. LSP defines its own codes like `RequestCancelled = -32800`. MCP keeps to the standard codes plus its own data payloads. Ethereum nodes have completely invented their own ad-hoc range (`3` for execution reverted, with the revert reason in `data`), which is a continuing source of pain for client libraries.

A useful Rust pattern is to make the error codes a const enum:

```rust
impl RpcError {
    pub const PARSE_ERROR: i32 = -32700;
    pub const INVALID_REQUEST: i32 = -32600;
    pub const METHOD_NOT_FOUND: i32 = -32601;
    pub const INVALID_PARAMS: i32 = -32602;
    pub const INTERNAL_ERROR: i32 = -32603;

    pub fn method_not_found(method: &str) -> Self {
        Self {
            code: Self::METHOD_NOT_FOUND,
            message: format!("Method not found: {method}"),
            data: None,
        }
    }
}
```

## Building a typed dispatcher

The interesting Rust question is: how do you turn `params: Option<Value>` into typed argument structs without writing a giant `match` on method names that all look the same?

One clean approach is a trait-object registry where each method registers a typed handler and the dispatcher does the deserialization:

```rust
use async_trait::async_trait;
use std::collections::HashMap;
use std::sync::Arc;

#[async_trait]
pub trait Handler: Send + Sync {
    async fn call(&self, params: Option<Value>) -> Result<Value, RpcError>;
}

pub struct Router {
    methods: HashMap<String, Arc<dyn Handler>>,
}

impl Router {
    pub fn new() -> Self {
        Self { methods: HashMap::new() }
    }

    pub fn register<H: Handler + 'static>(&mut self, name: &str, h: H) {
        self.methods.insert(name.into(), Arc::new(h));
    }

    pub async fn dispatch(&self, req: Request) -> Option<Response> {
        let id = match req.id {
            Some(id) => id,
            None => {
                // Notification - no response, even on error.
                if let Some(h) = self.methods.get(&req.method) {
                    let _ = h.call(req.params).await;
                }
                return None;
            }
        };

        let body = match self.methods.get(&req.method) {
            Some(h) => match h.call(req.params).await {
                Ok(v) => ResponseBody::Success { result: v },
                Err(e) => ResponseBody::Failure { error: e },
            },
            None => ResponseBody::Failure {
                error: RpcError::method_not_found(&req.method),
            },
        };

        Some(Response { jsonrpc: Version::V2, body, id })
    }
}
```

To register a typed handler, wrap a closure that takes a concrete `Params` type:

```rust
pub struct TypedHandler<P, R, F> {
    func: F,
    _phantom: std::marker::PhantomData<(P, R)>,
}

#[async_trait]
impl<P, R, F, Fut> Handler for TypedHandler<P, R, F>
where
    P: for<'de> Deserialize<'de> + Send,
    R: Serialize + Send,
    F: Fn(P) -> Fut + Send + Sync,
    Fut: std::future::Future<Output = Result<R, RpcError>> + Send,
{
    async fn call(&self, params: Option<Value>) -> Result<Value, RpcError> {
        let p: P = match params {
            Some(v) => serde_json::from_value(v)
                .map_err(|e| RpcError {
                    code: RpcError::INVALID_PARAMS,
                    message: e.to_string(),
                    data: None,
                })?,
            None => serde_json::from_value(Value::Null)
                .map_err(|e| RpcError {
                    code: RpcError::INVALID_PARAMS,
                    message: e.to_string(),
                    data: None,
                })?,
        };
        let r = (self.func)(p).await?;
        serde_json::to_value(r).map_err(|e| RpcError {
            code: RpcError::INTERNAL_ERROR,
            message: e.to_string(),
            data: None,
        })
    }
}
```

That is the entire handler infrastructure - about 80 lines once the boilerplate is gone. Compare with what a gRPC equivalent needs: a `.proto` file, a build script that runs `protoc`, generated code, a tonic server, and a tower service stack. JSON-RPC stays small because the protocol stays small.

## Batches: one feature, a lot of edge cases

The 2.0 spec added one thing over 1.0: batched requests. You can send a JSON array of requests and the server returns a JSON array of responses. The order of responses does not have to match the order of requests - clients correlate by `id`.

```json
[
  {"jsonrpc": "2.0", "method": "sum", "params": [1,2,4], "id": "1"},
  {"jsonrpc": "2.0", "method": "notify", "params": [7]},
  {"jsonrpc": "2.0", "method": "subtract", "params": [42,23], "id": "2"}
]
```

Edge cases that a real implementation has to handle:

- An empty batch `[]` returns a single Invalid Request error response.
- A batch of only notifications returns no response at all (not `[]`, the entire HTTP response body is empty or the connection just sees no reply on stdio transports).
- A batch with one malformed entry returns a batch where that one entry is replaced by an Invalid Request response and the rest are processed normally.
- The MCP spec actually [removed support for batching in protocol revision 2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18/changelog) because it complicated the streaming HTTP transport. Most LSP servers also do not bother. Batching is a feature that survives in long-tail protocols (Bitcoin Core still uses it for block syncs) more than in newer ones.

A naive batch dispatcher in Rust:

```rust
pub async fn handle_message(router: &Router, raw: &str) -> Option<String> {
    let value: Value = match serde_json::from_str(raw) {
        Ok(v) => v,
        Err(_) => return Some(error_response_str(RpcError::PARSE_ERROR, "Parse error")),
    };

    if value.is_array() {
        let arr = value.as_array().unwrap();
        if arr.is_empty() {
            return Some(error_response_str(RpcError::INVALID_REQUEST, "Empty batch"));
        }
        let mut out = Vec::new();
        for item in arr {
            match serde_json::from_value::<Request>(item.clone()) {
                Ok(req) => {
                    if let Some(resp) = router.dispatch(req).await {
                        out.push(serde_json::to_value(resp).unwrap());
                    }
                }
                Err(_) => out.push(invalid_request_value()),
            }
        }
        if out.is_empty() {
            None  // batch of only notifications
        } else {
            Some(serde_json::to_string(&out).unwrap())
        }
    } else {
        match serde_json::from_value::<Request>(value) {
            Ok(req) => router.dispatch(req).await
                .map(|r| serde_json::to_string(&r).unwrap()),
            Err(_) => Some(error_response_str(RpcError::INVALID_REQUEST, "Invalid Request")),
        }
    }
}
```

## REST, gRPC, JSON-RPC: how they actually differ

These three protocols dominate inter-service communication, and they pick different defaults.

**REST** is resource-oriented. URLs name nouns, HTTP verbs are the actions. The protocol gives you status codes, caching headers, content negotiation, and a route shape that maps cleanly to CRUD. The downside is that anything that is not CRUD - "calculate this", "subscribe to that", "trigger this batch job" - has to be shoehorned into POST-as-RPC, at which point you have JSON-RPC with extra ceremony.

**gRPC** is binary, schema-first, HTTP/2-only, and has a built-in code generator. You write a `.proto`, you get strongly typed stubs in any of a dozen languages, and you get streaming RPCs for free. The cost is operational: every client needs the schema, you need protoc, you need HTTP/2 between every hop (and HTTP/2 over the public internet is still finicky in 2026), and debugging on the wire requires a tool like grpcurl because the bytes are unreadable. If you are building service-to-service in a polyglot company with a schema registry, gRPC is the right answer. If you are not, the operational cost is high. We covered the protobuf side of this in [Rust and protobuf - schema-first API design with prost](/blog/rust-and-protobuf-schema-first-api-design-with-prost/).

**JSON-RPC** is method-oriented, transport-agnostic, schema-optional, and human-readable on the wire. There is no IDL, no code generator, and no opinion on transport. The same protocol works over stdio (LSP, MCP), TCP (Bitcoin), HTTP (Ethereum), WebSocket (Solana subscriptions), and Server-Sent Events (MCP Streamable HTTP). You give up the strong typing of gRPC and the resource semantics of REST, but you gain the ability to debug with `cat` and `tee`.

In benchmarks, JSON-RPC over HTTP is roughly 2-4x slower than gRPC and produces 3-6x larger payloads, depending on data shape. For most real workloads (1-10ms of database time per request) this is invisible. For Ethereum nodes serving 50,000 RPS of `eth_getLogs`, it absolutely matters, which is why most node operators put a caching layer in front and why teams like [Erigon](https://github.com/erigontech/erigon) have been experimenting with binary alternatives.

## Why JSON-RPC is having a moment again

Two things happened in 2024-2025 that brought JSON-RPC back into the conversation.

First, [LSP](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/) finally became universal. Every editor speaks it. Every language has at least one server. The protocol is JSON-RPC 2.0 with a `Content-Length` framing and a method namespace like `textDocument/didOpen`. It works because the client and server are usually on the same machine and the framing is dead simple.

Second, [MCP](https://modelcontextprotocol.io/) picked JSON-RPC 2.0 as its base. When Anthropic released MCP in November 2024, the choice between gRPC, REST, and JSON-RPC came down to: an LLM agent needs to call a method on a server, get a structured result, sometimes stream progress. JSON-RPC fits exactly. The stdio transport works for local tools. The Streamable HTTP transport works for remote ones. The schema lives in the `tools/list` response, not in a separate IDL. The tooling ecosystem of every editor and every blockchain node already knew how to debug it. We covered the broader picture in [MCP - what it actually is and why it matters](/blog/mcp-what-it-actually-is-and-why-it-matters/).

The pattern across LSP and MCP is the same: a protocol that is simple enough to implement from scratch in an afternoon, transport-agnostic enough to embed in any process, and human-readable enough that you can paste a transcript into a bug report. The AI tooling space has a lot of "from-scratch implementations in a new language" because the surface is small enough to make that practical.

## What the implementation does not give you

A JSON-RPC handler is not a complete application. The protocol does not define:

- **Authentication.** The 2.0 spec says nothing about auth. LSP does not need it (local processes). MCP layers OAuth 2.0 on top for HTTP transports. Ethereum nodes typically rely on network-level auth (private RPC endpoints, API keys in URLs).
- **Versioning.** No method namespace, no version negotiation, no capability discovery. MCP added an `initialize` handshake that returns a server's protocol version and capability list, but that is convention layered on top.
- **Schemas.** Each method's params and result shape is whatever the server documents. There is no equivalent of OpenAPI or .proto for JSON-RPC. [OpenRPC](https://open-rpc.org/) tried to fill this gap but adoption is patchy. MCP solves it pragmatically: a `tools/list` method returns JSON Schemas for each tool's input.
- **Streaming.** The base protocol is request/response. Long-running operations either chunk via notifications (LSP `$/progress`), use a separate subscription protocol (Ethereum `eth_subscribe` over WebSocket), or move to Server-Sent Events (MCP Streamable HTTP).

In other words, JSON-RPC gives you the envelope. Everything else is on you - which is also why so many projects pick it. You only pay for what you need.

## Try it

The full handler from this post is around 200 lines of Rust. The crates worth knowing if you want to skip writing it yourself:

- [`jsonrpsee`](https://crates.io/crates/jsonrpsee) - the most-used JSON-RPC library in the Rust ecosystem, originally built for Substrate. Server, client, WebSocket, HTTP, proc-macro for typed methods.
- [`tower-lsp`](https://crates.io/crates/tower-lsp) - LSP server framework on top of tower. Uses JSON-RPC under the hood.
- [`rmcp`](https://github.com/modelcontextprotocol/rust-sdk) - the official MCP Rust SDK, JSON-RPC 2.0 with stdio and Streamable HTTP transports.

If you have not implemented a JSON-RPC handler before, do it once. The whole spec is small enough that you will keep the model in your head for years afterwards, and every time someone says "it talks JSON-RPC over X" you will know exactly what that means and what it does not.
