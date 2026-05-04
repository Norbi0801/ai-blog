+++
title = "Rust and TLS - understanding HTTPS from the socket level"
date = 2026-01-06
description = "What rustls actually does between accept() and the first HTTP byte - handshake, certificate chains, SNI, ALPN, mTLS, pinning, and Let's Encrypt automation."

[taxonomies]
tags = ["rust", "tls", "rustls", "security"]
+++

If you have ever stared at a `reqwest` call and wondered what happens between `TcpStream::connect` and the first byte of HTTP, this post is for you. TLS is the layer most Rust devs never look inside, partly because the high-level APIs are good enough to feel invisible, and partly because the specs are intimidating. But once you have written a TLS client by hand, the rest of the ecosystem (`hyper`, `reqwest`, `axum`, `tonic`) stops being magic.

I covered the handshake at a packet level in [Understanding TCP/IP - what happens when you curl a URL](/blog/understanding-tcp-ip-what-happens-when-you-curl-a-url/). This post zooms into the TLS portion specifically, with Rust code you can run.

<!-- more -->

## The two TLS stacks you will meet

In Rust there are basically two camps:

- **rustls** - pure Rust, no OpenSSL, no C linking surprises. Uses [`aws-lc-rs`](https://github.com/aws/aws-lc-rs) (default since rustls 0.23) or [`ring`](https://github.com/briansmith/ring) for the actual cryptography. Modern only - TLS 1.2 and 1.3, no SSLv3 or earlier ciphers ever shipped.
- **native-tls** - a thin wrapper around whatever the platform provides. SecureTransport on macOS, SChannel on Windows, OpenSSL on Linux. You inherit your distro's TLS quirks and CVEs.

The default for most modern crates is now rustls. `reqwest` ships rustls behind the `rustls-tls` feature, `hyper` uses `hyper-rustls`, `tokio-tungstenite` wires up rustls for WebSockets. OpenSSL still appears when you need an obscure cipher, FIPS compliance, or interop with a legacy server that demands TLS 1.0.

The reason the ecosystem has consolidated on rustls is operational. A pure-Rust dependency builds the same way everywhere, statically links into your binary, and does not require pkg-config or `OPENSSL_DIR` at compile time. If you have ever cross-compiled a Rust binary that links OpenSSL, you know the pain. With rustls, `cargo build --target x86_64-unknown-linux-musl` just works.

## A minimal TLS client by hand

Skip the abstractions. Here is a TLS client that fetches `https://example.com` using `rustls` and `tokio-rustls` directly:

```rust
use std::sync::Arc;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use tokio_rustls::rustls::{ClientConfig, RootCertStore};
use tokio_rustls::TlsConnector;
use rustls::pki_types::ServerName;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut roots = RootCertStore::empty();
    roots.extend(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());

    let config = ClientConfig::builder()
        .with_root_certificates(roots)
        .with_no_client_auth();

    let connector = TlsConnector::from(Arc::new(config));
    let dnsname = ServerName::try_from("example.com")?;

    let stream = TcpStream::connect("example.com:443").await?;
    let mut tls = connector.connect(dnsname, stream).await?;

    tls.write_all(b"GET / HTTP/1.0\r\nHost: example.com\r\n\r\n").await?;
    let mut buf = Vec::new();
    tls.read_to_end(&mut buf).await?;
    println!("{}", String::from_utf8_lossy(&buf));
    Ok(())
}
```

`Cargo.toml`:

```toml
tokio = { version = "1", features = ["full"] }
tokio-rustls = "0.26"
rustls = "0.23"
webpki-roots = "0.26"
```

Five things actually happen here:

1. We build a `RootCertStore` from Mozilla's CA list, embedded in the binary via `webpki-roots`. No system CA store touched.
2. `ClientConfig::builder()` is a typestate builder that walks you through required choices: cipher suites (defaults to safe set), root certs, client auth.
3. `TlsConnector::connect` runs the handshake. Under the hood it sends a `ClientHello`, reads `ServerHello + Certificate + ...`, verifies the chain, and derives traffic keys.
4. The `tls` value implements `AsyncRead + AsyncWrite`, so you treat it like any other socket. All bytes you write are encrypted; all bytes you read are decrypted.
5. There is no explicit "close_notify". Dropping the stream sends one if the handshake is complete.

## What certificate verification actually checks

Most "TLS errors" come down to chain validation. When `rustls` (specifically the `rustls-webpki` crate) validates a server cert, it walks through:

1. **Signature chain.** Each cert is signed by the next one up. Validate using the issuer's public key. Continue until you hit a self-signed cert that exists in your `RootCertStore`. If the chain stops short, you get `UnknownIssuer`.
2. **Validity window.** `notBefore <= now <= notAfter`. Skew on the client clock is the most common cause of `Expired` errors in containers.
3. **Key usage.** The leaf cert must have `digitalSignature` (and `keyAgreement` for some suites) in its `keyUsage` extension and `serverAuth` in `extKeyUsage`.
4. **Hostname match.** The leaf cert's Subject Alternative Name (SAN) entries must include the hostname you connected to. The CN field has been ignored for hostname matching since [RFC 6125](https://datatracker.ietf.org/doc/html/rfc6125) and rustls strictly requires SAN.
5. **Path constraints.** Intermediate CAs may have `nameConstraints` that limit which domains they can sign. rustls enforces these.

Note what is *not* on this list: **revocation**. By default rustls does not check OCSP or CRL. This is intentional - OCSP soft-fail is a known weakness, OCSP stapling is unreliable in practice, and CRLs are huge and slow. If you need CRL checking you can build it explicitly:

```rust
use rustls::client::WebPkiServerVerifier;

let verifier = WebPkiServerVerifier::builder(Arc::new(roots))
    .with_crls(my_crls)
    .build()?;
```

Browsers handle this with their own infrastructure (CRLite, CRLSets). Server-side TLS clients almost always skip it. If you actually need to revoke, rotate the cert.

## Self-signed certs for development

For local dev you do not want to muck around with a CA. Two practical approaches:

**Option 1: rcgen.** Generate a cert programmatically:

```rust
use rcgen::{generate_simple_self_signed, CertifiedKey};

let CertifiedKey { cert, key_pair } =
    generate_simple_self_signed(vec!["localhost".into()])?;
std::fs::write("cert.pem", cert.pem())?;
std::fs::write("key.pem", key_pair.serialize_pem())?;
```

This is great for tests where you spin up a server, configure a custom verifier, and tear it down. It is also what `axum-server`'s testing examples use.

**Option 2: mkcert.** A separate binary that creates a local CA in your system trust store, then issues certs from it. Browsers and curl trust them automatically. You install it once, run `mkcert localhost 127.0.0.1`, and forget about it.

If you want a Rust client to accept a self-signed cert without installing a CA, you implement `ServerCertVerifier`. Do not turn off verification - implement a verifier that accepts a *specific* cert:

```rust
use rustls::client::danger::{ServerCertVerifier, ServerCertVerified};
use rustls::pki_types::CertificateDer;

#[derive(Debug)]
struct PinnedCert(CertificateDer<'static>);

impl ServerCertVerifier for PinnedCert {
    fn verify_server_cert(
        &self,
        end_entity: &CertificateDer<'_>,
        _intermediates: &[CertificateDer<'_>],
        _server_name: &ServerName<'_>,
        _ocsp: &[u8],
        _now: rustls::pki_types::UnixTime,
    ) -> Result<ServerCertVerified, rustls::Error> {
        if end_entity.as_ref() == self.0.as_ref() {
            Ok(ServerCertVerified::assertion())
        } else {
            Err(rustls::Error::General("cert mismatch".into()))
        }
    }
    // ... verify_tls12_signature, verify_tls13_signature, supported_verify_schemes
}
```

This is also the foundation of certificate pinning - more on that below.

## SNI - the unencrypted hostname

`Server Name Indication` is a TLS extension where the client puts the hostname it wants to talk to in the *unencrypted* `ClientHello`. It exists because TLS originally assumed one IP = one cert, and that broke as soon as virtual hosts arrived.

In rustls, SNI is handled automatically when you pass a `ServerName` to `connector.connect`. On the server side, you typically use a `ResolvesServerCert` trait implementation:

```rust
use rustls::server::{ClientHello, ResolvesServerCert};

struct MyResolver { /* map hostname -> CertifiedKey */ }

impl ResolvesServerCert for MyResolver {
    fn resolve(&self, hello: ClientHello) -> Option<Arc<rustls::sign::CertifiedKey>> {
        let host = hello.server_name()?;
        self.lookup(host)
    }
}
```

Two practical points:

- SNI is **plaintext on the wire**. A network observer can see which domain you visited even over HTTPS. ECH (Encrypted Client Hello) fixes this but rollout is incomplete. Cloudflare and Mozilla support it; many CDNs do not.
- Some servers send a *default* cert if the SNI is missing or unrecognised. Your client sees a cert mismatch and bails out. If a TLS connection works in browser but fails in your Rust code, check that you are passing the right `ServerName`.

## ALPN - picking HTTP/2 vs HTTP/1.1

Application-Layer Protocol Negotiation is a TLS extension where the client lists protocols it speaks, and the server picks one. This is how a single 443 socket can carry HTTP/1.1 or HTTP/2 (or HTTP/3 over QUIC, but that is UDP).

In rustls:

```rust
let mut config = ClientConfig::builder()
    .with_root_certificates(roots)
    .with_no_client_auth();
config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
```

After the handshake, `tls.get_ref().1.alpn_protocol()` returns the negotiated string. `hyper` uses this to decide whether to speak HTTP/2 framing or HTTP/1.1 line-based parsing on the same TCP stream.

Without ALPN, an HTTP/2 server has no safe way to upgrade because the first request bytes look like HTTP/1.1. ALPN is what made HTTP/2 deployable.

## mTLS - both sides authenticate

Mutual TLS flips the usual model: the server requires the client to present a cert during the handshake. Common in service-to-service auth, zero-trust networks, and things like Kubernetes API server authentication.

Server side (with `tokio-rustls`):

```rust
use rustls::server::WebPkiClientVerifier;

let client_roots = load_client_ca_pem("client-ca.pem")?;
let client_verifier = WebPkiClientVerifier::builder(Arc::new(client_roots)).build()?;

let server_config = ServerConfig::builder()
    .with_client_cert_verifier(client_verifier)
    .with_single_cert(server_certs, server_key)?;
```

Client side - load the cert and key, call `with_client_auth_cert` instead of `with_no_client_auth`:

```rust
let config = ClientConfig::builder()
    .with_root_certificates(roots)
    .with_client_auth_cert(client_certs, client_key)?;
```

The handshake adds two extra messages: `CertificateRequest` from the server, then `Certificate` and `CertificateVerify` from the client. The server validates the client cert chain just like a client validates a server cert. Identity ends up in the certificate's Subject or SAN, and your application reads it via `connection.peer_certificates()` after the handshake completes.

mTLS is operationally heavy because every client needs a cert, every cert expires, and rotation is a real problem. SPIFFE/SPIRE and `cert-manager` exist to automate this in service meshes.

## How reqwest and hyper wire rustls in

`reqwest` is a façade. With the `rustls-tls` feature, the build pulls in `hyper-rustls`, which in turn pulls in `tokio-rustls`. The chain looks like:

```
reqwest::Client
    -> hyper::Client<HttpsConnector>
        -> hyper_rustls::HttpsConnector<HttpConnector>
            -> tokio_rustls::TlsConnector
                -> rustls::ClientConnection
                    -> aws_lc_rs / ring (actual crypto)
```

When you call `client.get("https://...").send().await`, here is what actually happens:

1. `reqwest` resolves the URL, picks the connector based on scheme.
2. `HttpsConnector` opens a TCP stream via `HttpConnector` (with DNS lookup, IPv6 happy eyeballs, etc).
3. It calls `tokio_rustls::TlsConnector::connect` on the TCP stream.
4. ALPN gets negotiated based on what `hyper` advertises (`h2` and `http/1.1`).
5. Once the handshake finishes, `hyper` checks `alpn_protocol()` and either speaks HTTP/2 framing or HTTP/1.1.
6. The encrypted stream is treated as a normal `AsyncRead + AsyncWrite` from then on.

You can drop down a level any time you want. If you need fine control over the TLS config (custom roots, a specific verifier, ALPN order), you build a `rustls::ClientConfig` yourself and hand it to `hyper_rustls::HttpsConnectorBuilder::with_tls_config`. `reqwest::ClientBuilder::use_preconfigured_tls` does the same at the reqwest level.

This is also how you fix things like "my CI pipeline cannot find any roots". The default `webpki-roots` static list works everywhere; system roots require platform-specific code.

## Certificate pinning

Pinning is the practice of refusing connections to a server unless its certificate matches a known fingerprint. The point is to defeat a compromised CA - even if an attacker convinces a CA to issue a cert for your domain, it will not match the pin.

There are two flavours:

- **Public key pinning.** You pin the SPKI hash. Survives cert rotation as long as the key stays the same.
- **Certificate pinning.** You pin the full cert hash. Has to be updated every renewal.

With rustls, pinning is just a custom `ServerCertVerifier`. You compute the SHA-256 of the SubjectPublicKeyInfo (the DER bytes of the cert's public-key block) and compare:

```rust
use sha2::{Sha256, Digest};
use x509_parser::prelude::*;

fn spki_pin(cert_der: &[u8]) -> [u8; 32] {
    let (_, cert) = X509Certificate::from_der(cert_der).unwrap();
    let spki = cert.public_key().raw;
    Sha256::digest(spki).into()
}
```

If `spki_pin(end_entity)` does not match your hardcoded value, reject the connection. Mobile apps did this aggressively in the 2010s; it has fallen out of fashion because rotating a pin during an outage is painful. HPKP (the HTTP header) was deprecated for the same reason. Today you mostly see pinning in IoT, banking apps, and internal client/server pairs where you control both ends.

Do not pin against intermediates - CAs reissue intermediates without notice and you will brick clients. Pin the leaf or the leaf's public key.

## Let's Encrypt automation with ACME

A TLS server is useless without a valid cert, and Let's Encrypt has made free certs the default. The ACME protocol ([RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)) automates the process: you ask Let's Encrypt for a cert, prove you control the domain (via HTTP-01 or DNS-01), and download the cert.

In Rust, the modern crate is [`instant-acme`](https://github.com/instant-labs/instant-acme) - async, no OpenSSL, written by the rustls maintainers. The older `acme-lib` is sync and depends on OpenSSL via `openssl-sys`. Use `instant-acme` for new projects.

A skeleton:

```rust
use instant_acme::{Account, NewAccount, NewOrder, Identifier, LetsEncrypt};

let (account, _) = Account::create(
    &NewAccount {
        contact: &["mailto:admin@example.com"],
        terms_of_service_agreed: true,
        only_return_existing: false,
    },
    LetsEncrypt::Production.url(),
    None,
).await?;

let mut order = account.new_order(&NewOrder {
    identifiers: &[Identifier::Dns("example.com".to_string())],
}).await?;

let authorizations = order.authorizations().await?;
// For each authorization, complete an HTTP-01 or DNS-01 challenge.
// Then call order.finalize(&csr_der).await? and order.certificate().await?.
```

Production setups usually combine `instant-acme` with a small task that renews certs every ~60 days, writes them to disk, and signals the TLS server to reload. `rustls` does not hot-reload by default, but you can wrap your `ResolvesServerCert` impl in something that reads the cert from a `RwLock` and swap it on disk change.

If you do not want to write this yourself, [`rustls-acme`](https://github.com/FlorianUekermann/rustls-acme) does it as a tower service. You wrap your acceptor and it handles ACME-TLS-ALPN-01 challenges in-band on port 443, no separate web server needed.

## What I take away from all this

The TLS layer is dense, but the parts you actually deal with as a Rust developer fit on one page:

- One library does the work: `rustls`. Treat OpenSSL as legacy unless you have a specific reason.
- Certificate verification is chain + dates + SAN + key usage. Revocation is mostly fiction in 2026.
- SNI tells the server which cert to use, ALPN tells it which protocol to speak. Both happen during the handshake.
- mTLS is just symmetric verification with extra ops cost.
- Pinning is powerful and dangerous - use it only when you control both ends.
- ACME and `instant-acme` make free, automated certs a 50-line problem.

If you really want to internalise the handshake, capture a connection with `SSLKEYLOGFILE` set, decrypt it in Wireshark, and step through every record. The protocol stops feeling magical the moment you watch your `ClientHello` go out as a few hundred bytes of TLV and come back with a key share you can compute the shared secret from.
