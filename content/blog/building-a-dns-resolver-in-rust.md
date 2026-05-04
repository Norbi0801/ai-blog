+++
title = "Building a DNS resolver in Rust"
date = 2025-06-21
description = "A byte-level walk through DNS packets, recursive resolution from root to authoritative, and a ~200-line Rust resolver you can actually run."

[taxonomies]
tags = ["rust", "networking", "dns", "protocol"]
+++

Every HTTP request you will make today starts with a DNS lookup. If you're not familiar with where that lookup sits in the larger stack, I covered the full picture in [Understanding TCP/IP - what happens when you curl a URL](/blog/understanding-tcp-ip-what-happens-when-you-curl-a-url/). This post goes one layer deeper into that first UDP datagram and builds the whole resolver from scratch in Rust.

DNS feels mysterious because most developers only interact with it through `dig`, `nslookup`, or a library call. The moment you parse your first UDP packet by hand the mystery evaporates. It is a 12-byte header, a couple of length-prefixed labels, and a tree of servers that cooperate to answer your question. You can fit the whole thing in about two hundred lines of Rust.

<!-- more -->

## What DNS actually is

DNS is a hierarchical, distributed key-value store where keys are domain names and values are records. The records you care about most of the time:

- **A** - an IPv4 address. `example.com A 93.184.216.34`.
- **AAAA** - an IPv6 address. `example.com AAAA 2606:2800:220:1:248:1893:25c8:1946`.
- **CNAME** - "this name is actually an alias for that name". Resolvers follow the chain.
- **MX** - mail servers and priorities for a domain.
- **TXT** - free-form text. Used for SPF, DKIM, domain verification, and ad hoc metadata.
- **NS** - which nameservers are authoritative for a zone.
- **SOA** - start of authority. Serial number, refresh intervals, the primary nameserver.

The protocol lives in [RFC 1035](https://datatracker.ietf.org/doc/html/rfc1035) from 1987 with a pile of extensions since. Transport is UDP port 53 by default, with TCP fallback when a response would not fit in 512 bytes (classic DNS) or any size over EDNS0. For this post we stick to UDP.

## The wire format, one field at a time

Every DNS message, query or response, has the same shape:

```
+---------------------+
|       Header        |  12 bytes
+---------------------+
|      Question       |  QDCOUNT entries
+---------------------+
|       Answer        |  ANCOUNT entries
+---------------------+
|     Authority       |  NSCOUNT entries
+---------------------+
|     Additional      |  ARCOUNT entries
+---------------------+
```

The header is exactly 12 bytes:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ID                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|QR|  Opcode |AA|TC|RD|RA| Z|AD|CD|   RCODE   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    QDCOUNT     |     ANCOUNT                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    NSCOUNT     |     ARCOUNT                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

A query for `example.com A` looks like this on the wire:

```
// Header (12 bytes)
13 37                  // ID = 0x1337 (we pick this)
01 00                  // Flags: QR=0 (query), Opcode=0, RD=1 (recursion desired)
00 01                  // QDCOUNT = 1
00 00                  // ANCOUNT = 0
00 00                  // NSCOUNT = 0
00 00                  // ARCOUNT = 0

// Question
07 65 78 61 6d 70 6c 65  // "example" (len 7, then ASCII)
03 63 6f 6d              // "com"
00                       // null terminator
00 01                    // QTYPE = A (1)
00 01                    // QCLASS = IN (1)
```

QNAMEs are not null-terminated strings. They're sequences of length-prefixed labels ending in a zero byte. `example.com` becomes `\x07example\x03com\x00`. This is a common place where hand-written parsers trip.

Responses add one trick: **pointer compression**. Instead of repeating the full domain name in every resource record, the server can write a two-byte pointer where the top two bits are `11` and the remaining 14 bits are an offset into the message. When you read a length byte and see the top two bits set, jump to that offset and keep reading labels. You must not update your read position while jumping, and you must bound the number of jumps or a malicious server will send you into an infinite loop.

## Recursive resolution

When your laptop asks `8.8.8.8` for `example.com`, Google's resolver does the walking for you. If you build a resolver that talks directly to authoritative servers, you do the walking yourself. The walk looks like this:

1. **Ask a root server.** There are 13 root server letters (A through M). `a.root-servers.net` is at `198.41.0.4`. Ask it for `example.com A`. It will not know the answer, but it will return NS records for `com.` in the authority section and their IPs as glue in the additional section.
2. **Ask a `.com` nameserver.** Pick one of the NS IPs from the root response. Ask the same question. It returns NS records for `example.com.` (the authoritative servers for that zone).
3. **Ask the authoritative server.** This one actually has the A record.

Each step is a separate UDP exchange. With no caching you pay three round trips. With caching (covered below) most queries never reach the root.

The query bits that matter for this walk:

- **RD (Recursion Desired)** - set when you want the resolver to do the walking. When you talk directly to root or TLD servers you leave it off; they will not recurse for you.
- **AA (Authoritative Answer)** - the responding server is authoritative for the zone. Root servers set AA=0; they are authoritative only for the root zone itself, not for `example.com`.
- **RCODE** - 0 NOERROR, 2 SERVFAIL, 3 NXDOMAIN. The rest you rarely see.

## A minimal resolver in Rust

Enough theory. Here is a working resolver that sends a query, parses the response, and follows a simple recursive walk. It is deliberately compact and has no dependencies beyond the standard library.

```rust
// Cargo.toml
// [package]
// name = "tiny-dns"
// version = "0.1.0"
// edition = "2021"

use std::net::{Ipv4Addr, UdpSocket};
use std::time::Duration;

const ROOT: &str = "198.41.0.4:53"; // a.root-servers.net

#[derive(Debug, Clone, Copy, PartialEq)]
#[repr(u16)]
enum QType {
    A = 1,
    NS = 2,
    CNAME = 5,
    MX = 15,
    TXT = 16,
    AAAA = 28,
}

struct Buf {
    data: Vec<u8>,
    pos: usize,
}

impl Buf {
    fn new(data: Vec<u8>) -> Self { Self { data, pos: 0 } }
    fn u8(&mut self) -> u8 { let v = self.data[self.pos]; self.pos += 1; v }
    fn u16(&mut self) -> u16 { ((self.u8() as u16) << 8) | self.u8() as u16 }
    fn u32(&mut self) -> u32 {
        ((self.u8() as u32) << 24) | ((self.u8() as u32) << 16)
            | ((self.u8() as u32) << 8) | self.u8() as u32
    }

    fn qname(&mut self) -> String {
        let mut out = String::new();
        let mut pos = self.pos;
        let mut jumped = false;
        let mut hops = 0;
        loop {
            if hops > 20 { break; }
            let len = self.data[pos];
            if len & 0xC0 == 0xC0 {
                if !jumped { self.pos = pos + 2; }
                let b2 = self.data[pos + 1] as u16;
                pos = ((len as u16 & 0x3F) << 8 | b2) as usize;
                jumped = true;
                hops += 1;
                continue;
            }
            pos += 1;
            if len == 0 { break; }
            if !out.is_empty() { out.push('.'); }
            let label = &self.data[pos..pos + len as usize];
            out.push_str(&String::from_utf8_lossy(label));
            pos += len as usize;
        }
        if !jumped { self.pos = pos; }
        out
    }
}

fn encode_qname(name: &str, out: &mut Vec<u8>) {
    for label in name.split('.').filter(|s| !s.is_empty()) {
        out.push(label.len() as u8);
        out.extend_from_slice(label.as_bytes());
    }
    out.push(0);
}

fn build_query(id: u16, name: &str, qtype: QType, rd: bool) -> Vec<u8> {
    let mut buf = Vec::with_capacity(64);
    buf.extend_from_slice(&id.to_be_bytes());
    let flags: u16 = if rd { 0x0100 } else { 0x0000 };
    buf.extend_from_slice(&flags.to_be_bytes());
    buf.extend_from_slice(&1u16.to_be_bytes()); // QDCOUNT
    buf.extend_from_slice(&0u16.to_be_bytes()); // ANCOUNT
    buf.extend_from_slice(&0u16.to_be_bytes()); // NSCOUNT
    buf.extend_from_slice(&0u16.to_be_bytes()); // ARCOUNT
    encode_qname(name, &mut buf);
    buf.extend_from_slice(&(qtype as u16).to_be_bytes());
    buf.extend_from_slice(&1u16.to_be_bytes()); // QCLASS IN
    buf
}

#[derive(Debug)]
struct Record {
    name: String,
    rtype: u16,
    ttl: u32,
    data: RData,
}

#[derive(Debug)]
enum RData {
    A(Ipv4Addr),
    Ns(String),
    Cname(String),
    Mx { pref: u16, exchange: String },
    Txt(String),
    Other(Vec<u8>),
}

#[derive(Debug)]
struct Response {
    rcode: u8,
    answers: Vec<Record>,
    authorities: Vec<Record>,
    additionals: Vec<Record>,
}

fn parse(bytes: Vec<u8>) -> Response {
    let mut b = Buf::new(bytes);
    let _id = b.u16();
    let flags = b.u16();
    let rcode = (flags & 0x000F) as u8;
    let qd = b.u16();
    let an = b.u16();
    let ns = b.u16();
    let ar = b.u16();

    for _ in 0..qd {
        let _ = b.qname();
        b.u16(); // qtype
        b.u16(); // qclass
    }

    let read_n = |b: &mut Buf, n: u16| -> Vec<Record> {
        let mut out = Vec::with_capacity(n as usize);
        for _ in 0..n {
            let name = b.qname();
            let rtype = b.u16();
            let _class = b.u16();
            let ttl = b.u32();
            let rdlen = b.u16() as usize;
            let end = b.pos + rdlen;
            let data = match rtype {
                1 => RData::A(Ipv4Addr::new(b.u8(), b.u8(), b.u8(), b.u8())),
                2 => RData::Ns(b.qname()),
                5 => RData::Cname(b.qname()),
                15 => {
                    let pref = b.u16();
                    RData::Mx { pref, exchange: b.qname() }
                }
                16 => {
                    let tlen = b.u8() as usize;
                    let s = String::from_utf8_lossy(
                        &b.data[b.pos..b.pos + tlen]
                    ).into_owned();
                    b.pos += tlen;
                    RData::Txt(s)
                }
                _ => {
                    let raw = b.data[b.pos..end].to_vec();
                    b.pos = end;
                    RData::Other(raw)
                }
            };
            b.pos = end;
            out.push(Record { name, rtype, ttl, data });
        }
        out
    };

    let answers = read_n(&mut b, an);
    let authorities = read_n(&mut b, ns);
    let additionals = read_n(&mut b, ar);

    Response { rcode, answers, authorities, additionals }
}

fn query(server: &str, name: &str, qtype: QType, rd: bool)
    -> std::io::Result<Response>
{
    let sock = UdpSocket::bind("0.0.0.0:0")?;
    sock.set_read_timeout(Some(Duration::from_secs(3)))?;
    let pkt = build_query(rand_id(), name, qtype, rd);
    sock.send_to(&pkt, server)?;
    let mut buf = [0u8; 4096];
    let (n, _) = sock.recv_from(&mut buf)?;
    Ok(parse(buf[..n].to_vec()))
}

fn rand_id() -> u16 {
    // Not cryptographic. Fine for a demo. Use rand::random() in real code.
    use std::time::{SystemTime, UNIX_EPOCH};
    let ns = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().subsec_nanos();
    (ns & 0xFFFF) as u16
}

fn resolve(name: &str, qtype: QType) -> std::io::Result<Option<Ipv4Addr>> {
    let mut ns_ip: String = ROOT.to_string();
    loop {
        let resp = query(&ns_ip, name, qtype, false)?;
        if let Some(ip) = resp.answers.iter().find_map(|r| match &r.data {
            RData::A(ip) => Some(*ip),
            _ => None,
        }) {
            return Ok(Some(ip));
        }
        // No answer yet. Use glue in additionals if available.
        let next = resp.additionals.iter().find_map(|r| match &r.data {
            RData::A(ip) => Some(*ip),
            _ => None,
        });
        if let Some(next) = next {
            ns_ip = format!("{}:53", next);
            continue;
        }
        // Fall back: resolve the first NS name ourselves.
        let ns_name = resp.authorities.iter().find_map(|r| match &r.data {
            RData::Ns(n) => Some(n.clone()),
            _ => None,
        });
        match ns_name {
            Some(n) => {
                let ip = resolve(&n, QType::A)?;
                match ip {
                    Some(ip) => ns_ip = format!("{}:53", ip),
                    None => return Ok(None),
                }
            }
            None => return Ok(None),
        }
    }
}

fn main() -> std::io::Result<()> {
    let name = std::env::args().nth(1).unwrap_or_else(|| "example.com".into());
    match resolve(&name, QType::A)? {
        Some(ip) => println!("{} -> {}", name, ip),
        None => println!("{} -> (no answer)", name),
    }
    Ok(())
}
```

Run it:

```
$ cargo run --release example.com
example.com -> 93.184.216.34
```

What happened:

1. `resolve` sent a UDP query to `198.41.0.4:53` with RD=0. The root returned no answers but included `.com` NS records plus their A-record glue in the additional section.
2. We picked the first glue IP and sent the same question to that server. It returned NS records for `example.com.` with glue pointing at the authoritative servers.
3. We queried the authoritative server. It returned an A record in the answer section.

The whole walk is three packets, three responses. You can watch them with `tcpdump -n -i any 'udp port 53'`.

## Caching with TTL

Every record carries a TTL in seconds. When a resolver gets an answer it can reuse that answer for TTL seconds before asking again. This is why production resolvers cost so much less than you'd expect: the vast majority of queries hit the cache.

A simple cache keyed by `(name, qtype)`:

```rust
use std::collections::HashMap;
use std::time::{Instant, Duration};

struct CacheEntry {
    record: Record,
    expires: Instant,
}

struct Cache {
    map: HashMap<(String, u16), CacheEntry>,
}

impl Cache {
    fn get(&mut self, name: &str, qtype: u16) -> Option<&Record> {
        let key = (name.to_lowercase(), qtype);
        let now = Instant::now();
        if let Some(e) = self.map.get(&key) {
            if e.expires > now {
                return Some(&e.record);
            }
        }
        self.map.remove(&key);
        None
    }

    fn put(&mut self, rec: Record) {
        let expires = Instant::now() + Duration::from_secs(rec.ttl as u64);
        self.map.insert(
            (rec.name.to_lowercase(), rec.rtype),
            CacheEntry { record: rec, expires },
        );
    }
}
```

Three things to watch out for:

1. **TTL floors.** Servers sometimes return TTL=0 meaning "do not cache". Respect that.
2. **Negative caching.** If the authoritative server returns NXDOMAIN you should cache that too, for the TTL on the SOA record in the authority section ([RFC 2308](https://datatracker.ietf.org/doc/html/rfc2308)). Otherwise a single typo becomes a storm of queries.
3. **Case insensitivity.** Names compare case-insensitively. `Example.COM` and `example.com` are the same key.

Real resolvers also implement bailiwick checks (do not cache a record for `google.com` from a `evil.com` server), EDNS0 for larger responses, DNSSEC validation, and rate limiting. None of those are hard individually; collectively they're why `unbound` exists.

## Query types worth knowing

- **A / AAAA** are 99% of what your application asks for.
- **CNAME** resolvers follow automatically. `www.example.com` is often a CNAME to `example.com`, and a client asking for `www.example.com A` gets both RRs back.
- **MX** records have a preference integer and an exchange name. Lower preference wins. The exchange is another name you then resolve to A/AAAA.
- **TXT** records get abused for everything. SPF (`v=spf1 include:_spf.google.com ~all`), DKIM public keys, domain ownership verification for SaaS products. Each TXT record is a series of length-prefixed strings, which is why you sometimes see long SPF entries split across multiple strings concatenated at read time.
- **NS** records tell you who is authoritative. Querying for NS on a zone you operate is the fastest way to confirm delegation is correct.

## Why 512 bytes matters

Classic DNS over UDP caps each message at 512 bytes of payload. Anything larger and the server sets the TC (truncated) flag, and the client is supposed to retry over TCP. EDNS0 ([RFC 6891](https://datatracker.ietf.org/doc/html/rfc6891)) lets clients advertise a larger buffer (typically 4096) in an OPT pseudo-record in the additional section. DNSSEC responses and big TXT records absolutely require this.

You also need to care about UDP fragmentation. IP fragments can be dropped silently by firewalls, which is why modern resolvers often set a smaller EDNS buffer like 1232 to stay within typical path MTU after IP and UDP headers. See [DNS Flag Day 2020](https://dnsflagday.net/2020/) for the history.

## Where to go from here

The resolver above is a teaching tool. If you want to push it further:

- Add EDNS0 so you can handle large responses.
- Follow CNAME chains in your resolver logic instead of relying on the authoritative server to include the final A record.
- Implement retries with a different root letter when one times out.
- Look at [`hickory-dns`](https://github.com/hickory-dns/hickory-dns) (formerly trust-dns) for a production-grade resolver library written in Rust. Its packet parser is a more rigorous version of what you wrote here; reading it is good time spent.

You now know what happens in the first couple of milliseconds of every HTTP request. The next time curl says `Could not resolve host`, you can open Wireshark, filter `udp.port == 53`, and read the bytes. They will make sense.
