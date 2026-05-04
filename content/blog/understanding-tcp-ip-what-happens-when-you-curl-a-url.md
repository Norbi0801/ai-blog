+++
title = "Understanding TCP/IP - what happens when you curl a URL"
date = 2025-01-01
description = "A packet-level walkthrough of a single curl command - DNS, TCP handshake, TLS, HTTP, and connection close - with real Wireshark captures and the layers underneath."

[taxonomies]
tags = ["networking", "tcp-ip", "wireshark", "debugging"]
+++

You type `curl https://example.com` and ~200 milliseconds later, HTML appears. Between those two events your laptop sent roughly 15 packets, opened a connection, negotiated cryptographic keys, and spoke at least three different protocols. If you have never watched that in Wireshark it feels like one operation. It is not.

Understanding what actually happens on the wire is the difference between "DNS is broken, I will restart the router" and "the resolver is returning stale records because the authoritative server dropped the NOTIFY." This post walks through every packet of a single HTTPS request, maps each one to its layer in the stack, and shows the byte-level structure you would see if you opened the capture yourself.

<!-- more -->

## The layers, briefly

Networking is stacked because no single component can handle every concern. The classic OSI model has seven layers, but in practice TCP/IP collapses to four:

- **Link layer** - how two machines on the same physical segment talk. Ethernet frames, Wi-Fi, MAC addresses, ARP. Framing, collision detection, MTU.
- **Network layer** - how packets reach a machine you cannot physically see. IP addresses, routing, fragmentation. ICMP lives here.
- **Transport layer** - how two processes (not just two machines) reliably exchange bytes. TCP, UDP, QUIC. Ports, sequence numbers, acknowledgements, flow control.
- **Application layer** - what those bytes mean. HTTP, DNS, TLS (debatable), SSH, SMTP.

Every packet on the wire is a nested set of headers. A TCP segment carrying an HTTP request is wrapped in an IP packet, which is wrapped in an Ethernet frame. Each layer adds its own header and only cares about its own fields:

```
+----------------------------------------------------------+
| Ethernet header | IP header | TCP header | HTTP payload  |
|    14 bytes     |  20 bytes |  20 bytes  |   n bytes     |
+----------------------------------------------------------+
```

Keep this picture in mind. Every Wireshark row you are about to look at is this sandwich with different fillings.

## Step 1 - DNS lookup

`example.com` is not an IP. Before curl can open a socket it needs an address. The resolver library (glibc on Linux, via `getaddrinfo`) follows `/etc/nsswitch.conf` and usually ends up querying the configured DNS server in `/etc/resolv.conf`.

DNS uses UDP port 53 by default. The query is a single datagram:

```
; <<>> DiG 9.18.28 <<>> example.com
;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:
example.com.		300	IN	A	93.184.216.34
```

On the wire this is 12 bytes of DNS header plus the QNAME encoded with length-prefixed labels (`7example3com0`), a 2-byte QTYPE (A = 1) and 2-byte QCLASS (IN = 1). Total question: around 29 bytes. Response: around 45 bytes including the answer RR. Two UDP packets, round-trip time dominated by whatever your resolver takes to answer.

What can go wrong here:

- Resolver returns SERVFAIL. Check the authoritative zone, not your laptop.
- Resolver returns a stale record. Check the TTL on the record and whether a CNAME is pointing somewhere dead.
- `getaddrinfo` hangs for 5 seconds. Almost always an IPv6 query (AAAA) timing out because the resolver never replies. `strace -e trace=sendto,recvfrom curl ...` shows this clearly.

Modern systems often use DoH or DoT which wraps DNS in TLS over TCP/443 or TCP/853. That changes the packet shape but not the logical query.

## Step 2 - ARP (if the gateway is on the same LAN)

Before curl can send a single byte of TCP it needs a MAC address to put in the Ethernet header. The destination IP (`93.184.216.34`) is not on your LAN, so you send to your default gateway. If your ARP cache does not already have the gateway's MAC, the kernel broadcasts:

```
Who has 192.168.1.1? Tell 192.168.1.42
```

Somebody answers:

```
192.168.1.1 is at aa:bb:cc:dd:ee:ff
```

Now the kernel can fill in the Ethernet destination field. ARP is pure link-layer. It does not leave the broadcast domain. This is why misconfigured VLANs or a broken switch can make a machine unreachable even when routing looks fine: `arp -n` shows `(incomplete)`.

## Step 3 - TCP three-way handshake

Now TCP opens a connection to `93.184.216.34:443`. Three packets:

**1. SYN (client -> server)**

```
Flags: [S], Seq=0, Win=64240, Options: [MSS=1460, SACK_PERM, TS, WS=7]
```

The client picks a random Initial Sequence Number (shown as 0 because Wireshark displays relative sequence numbers by default - use `tcp.options.timestamp.tsval` or disable "Relative Sequence Numbers" to see the real 32-bit ISN). It advertises its receive window, its max segment size (MSS, usually 1460 on Ethernet = 1500 MTU - 20 IP - 20 TCP), and optional features.

**2. SYN-ACK (server -> client)**

```
Flags: [S, A], Seq=0, Ack=1, Win=65535, Options: [MSS=1460, ...]
```

Server acknowledges your SYN (Ack = your ISN + 1) and sends its own ISN.

**3. ACK (client -> server)**

```
Flags: [A], Seq=1, Ack=1, Win=64240
```

Connection is ESTABLISHED. One round trip from your first SYN to the final ACK. This is the first RTT you pay for any TCP connection.

The TCP header itself is 20 bytes minimum:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgement Number                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset|  Rsv  |C|E|U|A|P|R|S|F|         Window Size           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |        Urgent Pointer         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Options                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Those flag bits (CWR, ECE, URG, ACK, PSH, RST, SYN, FIN) are what Wireshark summarises as `[S]`, `[S,A]`, `[FIN,ACK]` and so on. When you see `[R]` appear mid-conversation, something slammed the door.

The full RFC is [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293), which is the consolidated modern spec replacing RFC 793.

## Step 4 - TLS handshake

Port 443 means HTTPS. Before curl sends a single HTTP byte, it has to negotiate TLS. With TLS 1.3 (standard since 2018, [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)) this is one round trip:

**ClientHello (client -> server)**

- Random 32 bytes
- Supported cipher suites (e.g. `TLS_AES_128_GCM_SHA256`)
- Supported signature algorithms
- Key share (client's ephemeral X25519 public key)
- `server_name` extension (SNI) containing `example.com` - this is plaintext, which is why firewalls can still see which domain you are visiting even over HTTPS. ECH is trying to fix this but rollout is partial.

**ServerHello + EncryptedExtensions + Certificate + CertificateVerify + Finished (server -> client)**

All in one TCP segment if it fits. The server picks a cipher, sends its key share, and from here on everything after EncryptedExtensions is encrypted with the handshake traffic secret derived from the shared DH output.

**Finished (client -> server)**

Client verifies the certificate chain, checks the CertificateVerify signature, derives the same keys, sends its own Finished. Done.

Total: 1 RTT before first application data. In TLS 1.2 it was 2 RTTs. TLS 1.3 also supports 0-RTT for resumption, but with replay-attack caveats.

If you want to see the raw bytes without running Wireshark, `curl -v --trace-ascii trace.txt` gives you a readable dump. For wire-level, `SSLKEYLOGFILE=keys.log curl ...` plus loading `keys.log` in Wireshark's TLS preferences lets you decrypt the capture.

## Step 5 - HTTP request and response

Now we finally speak HTTP. A minimal HTTP/1.1 request is:

```
GET / HTTP/1.1
Host: example.com
User-Agent: curl/8.5.0
Accept: */*

```

That's roughly 75 bytes, encrypted in TLS records and riding inside TCP segments. The server responds with status, headers, and body:

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 1256
Date: Wed, 22 Apr 2026 10:00:00 GMT

<!doctype html>...
```

If the body is larger than the current TCP window or MSS, it is split across multiple segments. Each segment gets acknowledged. If an ACK does not arrive within the retransmission timeout, the sender retransmits. This is where network quality becomes visible: high latency, packet loss, and small windows multiply into perceived slowness.

Modern HTTP (`curl --http2` or `--http3`) adds multiplexing and header compression, but HTTP/1.1 is still the clearest for walking through a capture.

## Step 6 - Connection close

TCP closes with a four-way handshake, or sometimes three if one side piggybacks:

```
client -> server  [FIN, ACK]
server -> client  [ACK]
server -> client  [FIN, ACK]
client -> server  [ACK]
```

TLS also has a `close_notify` alert, which well-behaved clients send before the TCP FIN. curl does. Some clients just send RST and move on, which is sloppy but common.

After FIN, the socket lingers in `TIME_WAIT` for 2 * MSL (usually 60 seconds) to catch stray retransmissions. This is why `netstat -ant | grep TIME_WAIT | wc -l` spikes on a busy web server. It is not a leak. It is correctness.

## Packet structure, MTU, and fragmentation

Every Ethernet frame is capped at 1500 bytes payload by default. Subtract 20 for the IP header and 20 for TCP and you get 1460 bytes of actual application data per segment. That is the MSS I mentioned in the SYN.

If an IP packet is larger than the MTU of any link along the path, it must be fragmented. The IP header has three fields for this:

- **Identification** (16 bits) - shared ID across all fragments of the same original datagram
- **Flags** - includes DF (Don't Fragment) and MF (More Fragments)
- **Fragment Offset** (13 bits, in 8-byte units) - where this fragment sits in the original datagram

TCP almost never wants to fragment. Path MTU Discovery sets DF=1 and relies on intermediate routers sending `ICMP Fragmentation Needed` back when the packet is too big. The sender then lowers its effective MSS. When firewalls block those ICMP messages (the infamous ICMP black hole) you get connections that establish fine, transfer small payloads fine, and then hang forever on the first large segment. This is a classic debugging landmine.

You can reproduce it locally:

```
ip link set dev eth0 mtu 1400
curl -v https://example.com
```

If the middlebox drops ICMP, large responses stall. `tracepath` is the simplest tool to see actual path MTU.

## Real Wireshark capture

Here is what the entire curl looks like, filtered to one connection with `tcp.stream eq 0`:

```
No. Time       Source          Destination     Proto  Length Info
1   0.000000   192.168.1.42    192.168.1.1     DNS    75     Standard query A example.com
2   0.012431   192.168.1.1     192.168.1.42    DNS    91     Standard query response A 93.184.216.34
3   0.013102   192.168.1.42    93.184.216.34   TCP    74     44222 -> 443 [SYN] Seq=0 Win=64240
4   0.035612   93.184.216.34   192.168.1.42    TCP    74     443 -> 44222 [SYN, ACK] Seq=0 Ack=1
5   0.035701   192.168.1.42    93.184.216.34   TCP    66     44222 -> 443 [ACK] Seq=1 Ack=1
6   0.036204   192.168.1.42    93.184.216.34   TLSv1.3 583   Client Hello
7   0.059108   93.184.216.34   192.168.1.42    TLSv1.3 1434  Server Hello, Change Cipher Spec, ...
8   0.059200   192.168.1.42    93.184.216.34   TCP    66     44222 -> 443 [ACK] Seq=518 Ack=1369
9   0.060012   192.168.1.42    93.184.216.34   TLSv1.3 146   Change Cipher Spec, Finished
10  0.060314   192.168.1.42    93.184.216.34   TLSv1.3 141   Application Data (HTTP GET)
11  0.082901   93.184.216.34   192.168.1.42    TLSv1.3 1434  Application Data (HTTP response part 1)
12  0.083112   93.184.216.34   192.168.1.42    TLSv1.3 312   Application Data (HTTP response part 2)
13  0.083201   192.168.1.42    93.184.216.34   TCP    66     [ACK]
14  0.084502   192.168.1.42    93.184.216.34   TLSv1.3 97    Alert (Level: Warning, close_notify)
15  0.084601   192.168.1.42    93.184.216.34   TCP    66     [FIN, ACK]
16  0.107203   93.184.216.34   192.168.1.42    TCP    66     [FIN, ACK]
17  0.107301   192.168.1.42    93.184.216.34   TCP    66     [ACK]
```

Roughly 100 ms wall-clock, 17 packets, four protocols. Each line maps to something we just walked through. Try `tcpdump -w capture.pcap -i any 'host example.com'` on your own machine, open the pcap in Wireshark, and follow along. The `Follow TCP Stream` menu (Ctrl+Shift+U on Linux) is your best friend.

## Why this matters for debugging

Most production networking bugs are not code bugs. They are state or timing problems at one of these layers, and you cannot see them from the application.

- "Requests hang at exactly 5 seconds" - classic SYN retransmit. Initial RTO is 1s, doubles: 1 + 2 = 3s before second retry, total 5-6s before failure reports. Something is dropping SYNs. Firewall? Cloud security group?
- "Requests fail after exactly 60 seconds of idle time" - a NAT device is dropping state. Either send keepalives or lower `tcp_keepalive_time`.
- "Small responses work, large ones hang" - ICMP black hole. PMTUD is broken. Lower MSS manually or fix the firewall.
- "Connection resets at random" - RST packets. Use `tcpdump 'tcp[tcpflags] & tcp-rst != 0'` to catch them. Usually a load balancer timing out or an app calling `close` on a socket with unread data (which triggers RST instead of FIN).
- "TLS handshake fails on some clients" - SNI mismatch, certificate chain incomplete, or a cipher mismatch. `openssl s_client -connect host:443 -servername host` shows the full chain and chosen cipher.
- "DNS is fine but curl is slow" - AAAA timeouts again. `curl -4` to force IPv4 and compare.

None of this is visible in application logs. You need to see the packets.

## Tools worth knowing

- **tcpdump** - capture anywhere, any host. Filters use BPF syntax: `tcpdump -n -i any 'tcp and port 443 and host example.com'`.
- **Wireshark** - GUI for captures. Best protocol dissectors in existence. Can decrypt TLS if you have the session keys.
- **ss** - replacement for netstat. `ss -tnp` shows TCP sockets with processes. `ss -i` shows congestion window, RTT, retransmits per socket.
- **mtr** - traceroute + ping in one, continuously. Shows per-hop loss and latency.
- **bpftrace / bcc** - for when you need to see what the kernel is doing with your packets. `tcpretrans.bt` shows every retransmit with stack traces.

Read [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) for the socket API, then [The TCP/IP Guide](http://www.tcpipguide.com/) for protocol detail. Bookmark [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293) (TCP) and [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446) (TLS 1.3). Skim them once, reference them forever.

## Takeaway

A single `curl https://example.com` is a surprisingly rich event. DNS over UDP, ARP on the LAN, a TCP three-way handshake, a TLS 1.3 handshake with certificate verification, actual HTTP bytes, then graceful close. Each of those can fail in its own unique way, and every one of those failure modes has a distinctive signature in a packet capture.

Next time something is "slow" or "flaky," resist the urge to sprinkle retries. Open Wireshark. The wire does not lie.
