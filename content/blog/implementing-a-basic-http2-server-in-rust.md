+++
title = "Implementing a basic HTTP/2 server in Rust"
date = 2025-10-17
description = "Binary framing, multiplexing, HPACK, flow control - building an HTTP/2 server with the h2 crate, then with hyper, and comparing latency against HTTP/1.1."

[taxonomies]
tags = ["rust", "http2", "networking", "hyper"]
+++

If you have not built a plain HTTP/1.1 parser yet, I covered that in [Writing a simple HTTP server from scratch in Rust](/blog/writing-a-simple-http-server-from-scratch-in-rust/). HTTP/2 is a different beast. The same semantics - methods, paths, headers, status codes - but the wire format has nothing to do with text. No `\r\n`. No `GET / HTTP/1.1`. Just framed binary, multiplexed streams, and a compressed header table that both peers maintain in lockstep.

This post implements a working HTTP/2 server using the [h2 crate](https://docs.rs/h2) directly so you can see the frames, then the same thing in [hyper](https://hyper.rs) where the protocol is hidden. Along the way: why HTTP/1.1 ran out of road, how binary framing solves head-of-line blocking, what HPACK actually compresses, and where HTTP/3 takes over.

<!-- more -->

## Why HTTP/1.1 needed replacing

HTTP/1.1 has one rule that defines its performance: one request at a time per connection. The server reads a request, writes a response, then reads the next one. You can pipeline requests (send several before reading any response), but pipelining never worked in practice - intermediaries break it, and one slow response head-of-line-blocks every request behind it.

Browsers worked around this by opening 6-8 parallel TCP connections per origin. Each handshake is a round-trip. Each TLS handshake is another two. On a 50ms RTT link that is 200ms of overhead before the first byte of payload moves. Worse, those connections have separate TCP congestion windows that all need to grow from scratch.

The header story is just as bad. Every request carries the same `User-Agent`, `Accept-Encoding`, `Cookie`, and a dozen others. A typical browser sends 800-1500 bytes of headers per request, uncompressed, on every single one. For a page with 100 sub-resources that is 80-150KB of redundant text on the wire.

HTTP/2 ([RFC 9113](https://www.rfc-editor.org/rfc/rfc9113)) fixes both: one TCP connection, many concurrent streams, headers compressed against a shared dictionary.

## The binary framing layer

Every HTTP/2 message is a sequence of frames. A frame has a fixed 9-byte header followed by a payload:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-+-----------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+
```

24-bit length, 8-bit type, 8-bit flags, 1 reserved bit, 31-bit stream id. That is the entire framing layer. Everything HTTP/2 does is stuffed into typed frames on numbered streams.

The [ten frame types](https://www.rfc-editor.org/rfc/rfc9113#name-frame-definitions) you actually see in practice:

- `DATA` (0x0) - request or response body bytes.
- `HEADERS` (0x1) - HPACK-encoded headers, opens a stream.
- `PRIORITY` (0x2) - stream priority hint. Deprecated by RFC 9113, but the frame still exists for compatibility.
- `RST_STREAM` (0x3) - cancel a single stream.
- `SETTINGS` (0x4) - connection-level parameters (max concurrent streams, initial window size, max frame size).
- `PUSH_PROMISE` (0x5) - server push. Deprecated in practice; Chrome dropped support in 2022.
- `PING` (0x6) - keepalive and RTT measurement.
- `GOAWAY` (0x7) - graceful shutdown of a connection.
- `WINDOW_UPDATE` (0x8) - flow control credit.
- `CONTINUATION` (0x9) - more HEADERS that did not fit in one frame.

Streams are odd-numbered for client-initiated, even-numbered for server-initiated. Stream 0 is the connection itself - SETTINGS, PING, GOAWAY, and connection-level WINDOW_UPDATE all live there.

## Multiplexing and stream states

A stream is the unit of a request/response exchange. Each stream has its own state machine:

```
                        +--------+
                send PP |        | recv PP
               ,--------|  idle  |--------.
              /         |        |         \
             v          +--------+          v
      +----------+          |           +----------+
      |          |          | send H /  |          |
,-----| reserved |          | recv H    | reserved |-----.
|     | (local)  |          |           | (remote) |     |
|     +----------+          v           +----------+     |
|         |             +--------+             |         |
|         |    recv ES  |        |  send ES    |         |
|         | ,-----------|  open  |-----------. |         |
|         | /           |        |            \|         |
|         v v           +--------+             v v       |
|     +----------+          |           +----------+     |
|     |   half   |          |           |   half   |     |
|     |  closed  |          | send R /  |  closed  |     |
|     | (remote) |          | recv R    | (local)  |     |
|     +----------+          |           +----------+     |
|          |                v                 |          |
|          |  send ES /     |     recv ES /   |          |
|          |  send R /  +--------+  send R /  |          |
|          |  recv R    | closed |  recv R    |          |
|          `----------->|        |<-----------'          |
|             send R /  +--------+ send R /              |
|             recv R                recv R               |
|                                                        |
|  send: frame sent     ES: END_STREAM flag              |
|  recv: frame received PP: PUSH_PROMISE                 |
|  H:    HEADERS         R: RST_STREAM                   |
'--------------------------------------------------------'
```

(That diagram is paraphrased from RFC 9113 section 5.1.) The takeaway: a stream lives from `idle` to `closed`. The client moves it to `open` by sending HEADERS. END_STREAM moves it to `half-closed`. Both sides setting END_STREAM closes it.

Concurrent streams on one TCP connection means HTTP/2 has its own scheduling problem. The server must interleave DATA frames from different streams so a slow response cannot starve fast ones. This is multiplexing. The hidden cost: HTTP/2 still runs over TCP, so a single packet loss stalls every stream until TCP recovers. That is HTTP/2's head-of-line blocking, and it is exactly what HTTP/3 fixes by moving to QUIC.

## HPACK in two paragraphs

[HPACK](https://www.rfc-editor.org/rfc/rfc7541) compresses headers using two tables both peers maintain. The static table is fixed - 61 entries with common headers like `:method GET`, `:status 200`, `accept-encoding gzip, deflate`. The dynamic table is built up as the connection proceeds; both peers add entries in the same order so an index `64` means the same header on both sides.

When you send `cookie: session=abc123` the first time, the encoder emits the literal name and value, with an instruction to insert it into the dynamic table. The second time, the encoder emits a single index. So the second request on a connection is dramatically smaller - you pay full cost once, near-zero cost after. This is also why HPACK has a [known security boundary](https://www.rfc-editor.org/rfc/rfc7541#section-7): if attacker-controlled values share a connection with secrets, the compressed length leaks information. CRIME-style attacks. The h2 crate implements [size-limit defenses](https://github.com/hyperium/h2) but you should still keep auth tokens out of headers an attacker can influence.

## Flow control

Each stream has a receive window measured in bytes. The default is 65535. The receiver advertises additional window with `WINDOW_UPDATE`. Senders cannot send DATA bytes beyond the current window. There are two windows: per-stream and connection-level. Both must have credit for a DATA frame to move.

This is the part nobody implements correctly the first time. A naive server that holds DATA frames in a buffer and never sends WINDOW_UPDATE will deadlock the moment a client tries to upload a big body. The h2 crate gives you `flow_control()` handles you call as you consume body bytes, which is the correct model.

## Building it with the h2 crate

The [`h2` crate](https://docs.rs/h2/latest/h2/) is the same one [hyper](https://github.com/hyperium/hyper) uses internally. It exposes the protocol directly: streams, frames, flow control. Cargo.toml:

```toml
[dependencies]
h2 = "0.4"
http = "1"
bytes = "1"
tokio = { version = "1", features = ["full"] }
```

Then a minimal server that handles every request with a 200 OK and a small body:

```rust
use bytes::Bytes;
use h2::server;
use http::{Response, StatusCode};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let listener = TcpListener::bind("127.0.0.1:8443").await?;
    println!("h2 server listening on {}", listener.local_addr()?);

    loop {
        let (socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            if let Err(e) = serve(socket).await {
                eprintln!("connection error: {e}");
            }
        });
    }
}

async fn serve(socket: tokio::net::TcpStream) -> Result<(), h2::Error> {
    // Perform the HTTP/2 connection preface and SETTINGS exchange.
    let mut connection = server::handshake(socket).await?;

    while let Some(request_result) = connection.accept().await {
        let (request, mut respond) = request_result?;
        tokio::spawn(async move {
            let (parts, mut body) = request.into_parts();
            println!("> {} {}", parts.method, parts.uri.path());

            // Drain the request body (and grant flow-control credit as we go).
            while let Some(chunk) = body.data().await {
                let chunk = chunk?;
                let _ = body.flow_control().release_capacity(chunk.len());
            }

            let response = Response::builder()
                .status(StatusCode::OK)
                .header("content-type", "text/plain")
                .body(())
                .unwrap();

            let mut send = respond.send_response(response, false)?;
            send.send_data(Bytes::from_static(b"hello over h2\n"), true)?;
            Ok::<_, h2::Error>(())
        });
    }
    Ok(())
}
```

Things worth noticing. The handshake call writes the server's SETTINGS frame and reads the client's preface (`PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`) and SETTINGS. The `accept` loop yields one fully-arrived HEADERS frame at a time, but those streams are concurrent - each spawned task processes its own. Calling `release_capacity` is what sends `WINDOW_UPDATE` back to the client; if you forget it, large uploads stall.

The `false` and `true` in `send_response` and `send_data` are the END_STREAM flag. Setting it on the HEADERS would mean "no body, response is complete." Setting it on the last DATA frame is what closes the stream cleanly.

## Plain TCP vs ALPN

The example above runs cleartext HTTP/2, sometimes called h2c. Browsers do not speak h2c - they only negotiate HTTP/2 over TLS via ALPN. ALPN is a TLS extension where the client offers `["h2", "http/1.1"]` and the server picks one. With [tokio-rustls](https://docs.rs/tokio-rustls) it looks like:

```rust
use tokio_rustls::rustls::ServerConfig;

let mut config = ServerConfig::builder()
    .with_no_client_auth()
    .with_single_cert(certs, key)?;
config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
```

After the TLS handshake, `connection.alpn_protocol()` tells you which one was picked. Pass the `TlsStream` into `h2::server::handshake` only when ALPN selected `h2`; otherwise hand it to your HTTP/1.1 handler.

## The same thing with hyper

In production you almost never use h2 directly. [hyper 1.x](https://hyper.rs) handles ALPN, version detection, keep-alive, and everything else. The same handler in hyper:

```rust
use hyper::{Request, Response, body::Bytes};
use hyper::server::conn::http2;
use hyper_util::rt::{TokioExecutor, TokioIo};
use http_body_util::Full;
use tokio::net::TcpListener;

async fn hello(_req: Request<hyper::body::Incoming>)
    -> Result<Response<Full<Bytes>>, std::convert::Infallible>
{
    Ok(Response::new(Full::new(Bytes::from_static(b"hello over h2\n"))))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let listener = TcpListener::bind("127.0.0.1:8443").await?;
    loop {
        let (stream, _) = listener.accept().await?;
        let io = TokioIo::new(stream);
        tokio::spawn(async move {
            if let Err(e) = http2::Builder::new(TokioExecutor::new())
                .serve_connection(io, hyper::service::service_fn(hello))
                .await
            {
                eprintln!("error: {e}");
            }
        });
    }
}
```

Or use `hyper_util::server::conn::auto::Builder` to accept both HTTP/1.1 and HTTP/2 on the same listener and pick based on ALPN or the connection preface. That is what every real Rust HTTP server does - axum, actix-web, warp all sit on top of this.

## Latency: HTTP/1.1 vs HTTP/2

A simple microbenchmark: load 50 small JSON resources from a local server. With [h2load](https://nghttp2.org/documentation/h2load.1.html) on a tokio + hyper backend on the same machine:

```
$ h2load -n 5000 -c 50 --h1 https://localhost:8443/api/items
finished in 1.42s, 3521 req/s, mean latency 14.1ms, p99 41ms

$ h2load -n 5000 -c 50 https://localhost:8443/api/items
finished in 0.78s, 6410 req/s, mean latency 7.6ms, p99 19ms
```

The numbers vary heavily with workload, but the shape repeats: HTTP/2 wins on small parallel requests because the connection cost amortizes over many streams and headers compress aggressively. On a single sequential request HTTP/2 is slightly slower - one extra round-trip for the SETTINGS exchange, plus the HPACK encoder doing more work than a single static `Host: example.com\r\n` line.

Where HTTP/2 does not help much:

- Single large download. One stream, one TCP connection, same as HTTP/1.1. Sometimes worse because of HTTP/2's per-stream window default of 65535 - you need to bump `initial_window_size` to multiple megabytes for high-bandwidth-delay-product links.
- Very small APIs over already-established HTTP/1.1 keep-alive connections. The wire-format savings are real but small.
- Lossy networks. HTTP/2's TCP head-of-line blocking gets worse as packet loss rises.

## When HTTP/3

[HTTP/3](https://www.rfc-editor.org/rfc/rfc9114) keeps the same semantics as HTTP/2 but runs over [QUIC](https://www.rfc-editor.org/rfc/rfc9000) instead of TCP. QUIC is UDP plus its own ordering, encryption (TLS 1.3 baked in), and multi-stream flow control. The win: a packet loss only stalls one stream, not the whole connection. The 0-RTT and 1-RTT handshake also folds TLS into the transport, saving a round-trip on a fresh connection.

You probably want HTTP/3 if any of these apply:

- Mobile clients on lossy links (cellular, Wi-Fi handoff).
- Connections with high RTT where saving one round-trip is meaningful.
- You serve a lot of media or many small assets where head-of-line blocking inside TCP is measurable.

In Rust the active crates are [quinn](https://github.com/quinn-rs/quinn) for QUIC and [h3](https://github.com/hyperium/h3) on top for HTTP/3. The API mirrors `h2`'s shape - handshake, accept streams, send headers and data - but the streams are independent at the transport level.

The reason HTTP/3 is not the default everywhere yet is operational. UDP gets weird treatment from middleboxes, NAT timeouts are shorter, and a lot of corporate networks still drop or rate-limit UDP. Most production deployments today still terminate TLS and HTTP/2 at the edge and speak HTTP/1.1 or HTTP/2 internally. HTTP/3 is great when it works, but the long tail of "it does not work" is real.

## What to take from this

Most days you write `axum::Router::new().route(...)` and never think about frames. That is correct. But when you see `STREAM_ID 5 RST_STREAM CANCEL` in a log, or your gRPC client mysteriously hangs on a 4MB upload, or a load test shows your server falling over at 1000 RPS while a competitor handles 10000, you need to know what is underneath. HTTP/2 is binary, multiplexed, flow-controlled, and stateful at the connection level. Every one of those words is a way it can go wrong.

The h2 crate is small enough to read end-to-end ([source on GitHub](https://github.com/hyperium/h2/tree/master/src)). If you have an afternoon, walk through `proto/streams/state.rs` and the frame codec. Once you have seen the actual byte manipulation it stops being magic, and you start writing better servers - even when you are using a framework that hides all of it.
