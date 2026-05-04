+++
title = "SSH protocol basics - what happens when you ssh into a server"
date = 2026-02-14
description = "A packet-level walkthrough of an ssh session - version exchange, key exchange, authentication, channels, port forwarding - with the config tricks and security gotchas that actually matter."

[taxonomies]
tags = ["ssh", "networking", "security", "crypto"]
+++

You type `ssh user@host`, a prompt appears, and you start issuing commands. It feels like a terminal cable. It is not. Between you and that prompt, two processes negotiated a cipher, authenticated each other with public key crypto, derived session keys from a Diffie-Hellman exchange, and opened a multiplexed channel inside a single TCP connection. Every keystroke after that is encrypted and framed.

If you are not familiar with how TCP actually connects, I covered it in [Understanding TCP/IP](/blog/understanding-tcp-ip-what-happens-when-you-curl-a-url/). SSH lives on top of that. Everything in this post is application-layer protocol running inside a plain TCP socket on port 22.

<!-- more -->

## What SSH actually is

SSH is three protocols stacked together, defined in RFCs 4251 through 4254:

- **Transport layer protocol (RFC 4253)** - negotiates keys, encrypts and authenticates the byte stream. Gives you confidentiality, integrity, and server authentication.
- **User authentication protocol (RFC 4252)** - proves who the client is. Password, public key, keyboard-interactive, GSSAPI.
- **Connection protocol (RFC 4254)** - multiplexes the encrypted stream into channels. Interactive sessions, port forwards, X11, agent forwarding all share one TCP connection.

The order matters. You cannot authenticate over an unencrypted channel (that would leak the password), and you cannot multiplex over an unauthenticated channel (an attacker could inject a channel). Each layer assumes the one below is already working.

## Step 1 - TCP connect and version exchange

OpenSSH opens a TCP socket to port 22. Once the three-way handshake completes, neither side has sent any SSH bytes yet. The server speaks first:

```
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.4
```

The client replies with its own banner:

```
SSH-2.0-OpenSSH_9.7
```

This is plain ASCII, terminated with CRLF. It serves three purposes. First, version negotiation - if one side is SSH-1 only, the other can decide to abort. SSH-1 has been broken for 20 years, and every modern implementation rejects it. Second, implementation fingerprinting - the banner tells you what you are talking to, which is why security scanners use it. Third, it is the first thing that hits your logs. When something fails to connect, check if the banner ever arrived:

```
$ nc -v host 22
Connection to host 22 port [tcp/ssh] succeeded!
SSH-2.0-OpenSSH_9.6p1
```

If you see the banner, TCP and the SSH daemon are both alive. If you do not, the problem is somewhere in between.

## Step 2 - Algorithm negotiation (KEXINIT)

Immediately after banners, both sides send a `SSH_MSG_KEXINIT` packet. This is a binary message listing every algorithm each side supports, in preference order:

- Key exchange algorithms (`curve25519-sha256`, `ecdh-sha2-nistp256`, `diffie-hellman-group14-sha256`, ...)
- Server host key algorithms (`ssh-ed25519`, `rsa-sha2-512`, ...)
- Encryption ciphers for each direction (`chacha20-poly1305@openssh.com`, `aes256-gcm@openssh.com`, ...)
- MAC algorithms (often None when using an AEAD cipher like ChaCha20-Poly1305)
- Compression algorithms (almost always None in 2026)

The negotiation rule is simple: both sides walk the client's preference list and pick the first entry the server also supports. You can see the negotiated set with `ssh -v`:

```
debug1: kex: algorithm: curve25519-sha256
debug1: kex: host key algorithm: ssh-ed25519
debug1: kex: server->client cipher: chacha20-poly1305@openssh.com MAC: <implicit>
debug1: kex: client->server cipher: chacha20-poly1305@openssh.com MAC: <implicit>
```

This is where weak-algorithm attacks live. If your sshd still accepts `diffie-hellman-group1-sha1` (1024-bit DH prime, SHA-1), a sufficiently funded attacker can break the session. Audit with `ssh-audit` or look at `Ciphers`, `KexAlgorithms`, `MACs` in `sshd_config`. OpenSSH 9.x defaults are sane. Older ones are not.

## Step 3 - Key exchange (Diffie-Hellman)

Now the two sides need a shared secret that nobody listening on the wire can derive. SSH uses Diffie-Hellman for this, most commonly over Curve25519 these days.

The math in one paragraph: pick a prime `p` and generator `g` (for finite-field DH) or an elliptic curve point (for ECDH). Client picks a random private `a`, computes public `A = g^a mod p` (or `A = aG` on the curve), sends `A`. Server picks random `b`, sends `B = g^b`. Each side computes the shared secret `K = B^a = A^b = g^(ab)`. A passive observer sees `A` and `B` but cannot compute `g^(ab)` without solving the discrete log problem, which is infeasible for 2048-bit primes or a 256-bit curve.

But plain DH is vulnerable to man-in-the-middle. The server signs the exchange hash with its long-term host key:

```
H = HASH(V_C || V_S || I_C || I_S || K_S || e || f || K)
```

Where `V_C`, `V_S` are version strings, `I_C`, `I_S` are the KEXINIT payloads, `K_S` is the server's public host key, `e` and `f` are the DH public values, and `K` is the shared secret. The server sends `K_S` and a signature of `H` made with its private host key. The client verifies the signature using `K_S` and checks `K_S` against known hosts (more on that in a moment).

If an attacker tries to MITM, they can complete a DH exchange with each side, but they cannot forge the server's signature. The client would reject the session.

From `K` and `H` both sides derive six keys using KDF output - two encryption keys, two IV seeds, two MAC keys (one per direction). These keys are fresh per session. SSH also rekeys automatically every ~1 GB or hour, which limits the damage of any individual key compromise.

## Step 4 - Known hosts and TOFU

When the client receives `K_S`, it hashes it and looks it up in `~/.ssh/known_hosts`:

```
|1|abc123...|def456...= ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG...
```

The `|1|` prefix means the hostname is hashed (`HashKnownHosts yes`). The base64 blob is the public host key. If the fingerprint matches, the session continues silently. If there is no entry, you get the famous prompt:

```
The authenticity of host 'host (1.2.3.4)' can't be established.
ED25519 key fingerprint is SHA256:xYz...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This is Trust On First Use. You have no way to verify the key unless the server operator published it somewhere. In practice most people type `yes` and pray. Better options:

- Use SSHFP DNS records (`ssh -o VerifyHostKeyDNS=yes`). The host publishes its key fingerprint as a DNSSEC-signed record, and ssh validates it. Requires DNSSEC, which is rare.
- Use SSH certificates (below). The client trusts a CA, so individual host keys do not need to be pre-shared.
- Compare the fingerprint out-of-band on first connect. Good discipline, rarely practiced.

If the stored key ever changes, you get a much louder warning:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

This is either a MITM attack or a reprovisioned server. Do not blindly `ssh-keygen -R host`. Verify first.

## Step 5 - User authentication

At this point the channel is encrypted and the server is authenticated. Now the server needs to prove the client is allowed in. The client sends `SSH_MSG_SERVICE_REQUEST` for `ssh-userauth`, and the server enumerates allowed methods:

```
debug1: Authentications that can continue: publickey,password
```

### Password auth

Simplest, worst. The client encrypts the password inside the already-encrypted SSH channel and sends it. The server hashes it against `/etc/shadow` (or whatever PAM is configured for) and accepts or rejects. This is safe from wire sniffing because the channel is encrypted, but vulnerable to brute force, phishing, and leaked passwords. Disable it on anything facing the internet:

```
# /etc/ssh/sshd_config
PasswordAuthentication no
```

### Public key auth

The good one. The server has a list of authorized public keys for each user in `~/.ssh/authorized_keys`. Each line looks like:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG... norbert@laptop
```

The auth flow:

1. Client sends a `SSH_MSG_USERAUTH_REQUEST` with the public key blob.
2. Server checks if the key is in `authorized_keys` for that user. If not, reject.
3. If yes, server replies with `SSH_MSG_USERAUTH_PK_OK` (the key is acceptable, now prove you own it).
4. Client signs the session ID (which includes `H` from key exchange) with the corresponding private key and sends the signature.
5. Server verifies the signature against the public key. Pass or fail.

The signature covers the session ID, so it cannot be replayed on a different session. A stolen signature from one ssh login is useless for another.

`authorized_keys` supports per-key restrictions:

```
command="/usr/bin/git-shell",no-port-forwarding,no-X11-forwarding,no-pty ssh-ed25519 AAAA...
from="10.0.0.0/8" ssh-ed25519 AAAA...
```

The `command=` option forces a specific program to run regardless of what the client requested. This is how GitHub's `git@github.com` SSH works - every key runs `git-shell` which only understands git protocol commands.

### SSH certificates

Certificates solve the TOFU-and-authorized_keys-sprawl problem. Instead of copying public keys to every server, you trust a CA:

```
# /etc/ssh/sshd_config
TrustedUserCAKeys /etc/ssh/ca.pub
```

The CA signs user keys with `ssh-keygen -s`:

```
ssh-keygen -s ca -I norbert@laptop -n norbert,root -V +1h user.pub
```

That produces `user-cert.pub`, which the client presents during auth. The cert embeds the valid principals (usernames), validity window, and optional force-command. Short-lived certs (1 hour, 8 hours) kill most of the "terminated employee still has their key" risk. Teleport, Step-CA, and Smallstep all build on this primitive.

The server can also have a host certificate signed by a host CA, which replaces TOFU. The client trusts the CA once in `known_hosts` and every host signed by it validates automatically:

```
@cert-authority *.example.com ssh-ed25519 AAAA...
```

## Step 6 - ssh-agent

Your private key lives in `~/.ssh/id_ed25519`. It is (hopefully) encrypted with a passphrase. Typing that passphrase for every ssh, git push, and rsync would be miserable.

`ssh-agent` is a long-running process that holds decrypted keys in memory. On connect, ssh asks the agent to sign the session ID. The private key never leaves the agent, and the agent communicates with the ssh client over a Unix socket pointed to by `$SSH_AUTH_SOCK`.

```
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
```

Agent forwarding (`ForwardAgent yes` or `ssh -A`) extends this socket to the remote machine. That remote ssh can then talk back to your local agent to sign for further hops. Useful, but dangerous: any root on the remote box can impersonate you for the life of your connection. Use `ProxyJump` instead when you can:

```
# ~/.ssh/config
Host prod
  HostName 10.0.0.5
  ProxyJump bastion
```

`ProxyJump` tunnels TCP through the bastion without exposing the agent socket there.

## Step 7 - Channels and multiplexing

Once authenticated, the connection protocol takes over. The TCP socket is now a pipe carrying framed encrypted packets, each tagged with a channel number.

```
SSH_MSG_CHANNEL_OPEN    ("session", client_channel=0, window=2097152, max_packet=32768)
SSH_MSG_CHANNEL_REQUEST (channel=0, "pty-req", ...)
SSH_MSG_CHANNEL_REQUEST (channel=0, "shell")
SSH_MSG_CHANNEL_DATA    (channel=0, "ls -la\n")
SSH_MSG_CHANNEL_DATA    (channel=0, "total 24\n...")
```

One interactive shell, X11 forward, sftp subsystem, and three local port forwards can all coexist on the same TCP connection with different channel numbers. Each channel has its own flow-control window (`SSH_MSG_CHANNEL_WINDOW_ADJUST`), so a slow sftp transfer cannot starve your shell.

This is also what ControlMaster gives you. With:

```
Host *
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
```

The first ssh to a host opens a real TCP connection. Subsequent ssh, scp, or rsync invocations to the same host reuse that connection and just open a new channel. Startup drops from "full KEX + auth" (hundreds of ms) to "open a channel" (single RTT). If you have ever wondered why `scp file host:` feels slow, this is the fix.

## Port forwarding and tunnels

Port forwards are just channels with special types.

Local forward (`-L`) opens a local listening socket. When something connects to it, ssh opens a `direct-tcpip` channel to the server, which dials the requested remote address:

```
ssh -L 5432:db.internal:5432 bastion
# now psql -h localhost -p 5432 works
```

The traffic goes: your psql → localhost:5432 → ssh client → channel → ssh server on bastion → TCP to db.internal:5432.

Remote forward (`-R`) is the reverse. The server opens a listening socket, and incoming connections get forwarded back through the channel to the client, which dials a target from your side. This is how `ngrok`-style tools (before HTTP-level proxies became common) exposed a laptop to the internet via a VPS.

Dynamic forward (`-D`) runs a SOCKS5 proxy locally. Each SOCKS request becomes a new channel with a target chosen per-connection. Effectively a poor-person's VPN.

All three run inside the same encrypted, authenticated SSH session. No extra TCP connections, no extra auth.

## ~/.ssh/config tricks worth knowing

Stop typing `-p 2222 -i ~/.ssh/github -o StrictHostKeyChecking=no`. Put it in config once:

```
Host bastion
  HostName bastion.example.com
  User deploy
  Port 2222
  IdentityFile ~/.ssh/deploy_ed25519
  IdentitiesOnly yes

Host prod-*
  User deploy
  ProxyJump bastion
  IdentityFile ~/.ssh/deploy_ed25519

Host github.com
  IdentityFile ~/.ssh/github_ed25519
  IdentitiesOnly yes
```

Things to know:

- `IdentitiesOnly yes` stops ssh from offering every key in your agent. Without it, if you have eight keys loaded, ssh tries them all, and the server may disconnect after too many failures.
- `ProxyJump` replaces the old `ProxyCommand ssh -W %h:%p bastion` dance.
- Patterns cascade top-to-bottom, first-match-wins per option. Put specific hosts before wildcards.
- `Match` is more powerful than `Host` - can match on user, final hostname, exec result.

## Security considerations

A few things that actually bite in practice:

- **Disable password auth on anything internet-facing.** Bots hit port 22 within seconds of a new IP being online. Public-key-only removes the entire bruteforce attack surface.
- **Move the port or put sshd behind a firewall.** Does not improve crypto, does reduce log noise. Combine with fail2ban or (better) only allow source IPs you control.
- **Use ed25519 keys.** Smaller, faster, no parameter choice to get wrong. RSA is still fine if it is 3072-bit or larger, but there is no reason to pick it today.
- **Rotate host keys if a server is compromised.** The attacker might have exfiltrated them, and they let any future MITM go undetected until clients see the warning.
- **Think twice about agent forwarding.** Prefer `ProxyJump`. If you must forward, use `ssh-add -c` which requires confirmation for each signature.
- **Audit `authorized_keys` drift.** Some organizations sync it with config management, some put it in immutable images, almost nobody audits it. An attacker who appends their key owns the account until someone notices.
- **Log and monitor.** sshd logs every authentication attempt. Ship those to a central log, alert on brute force, alert on successful logins from unusual sources.

## What to read next

The RFCs are actually readable. Start with [RFC 4253](https://www.rfc-editor.org/rfc/rfc4253) (transport), then [RFC 4252](https://www.rfc-editor.org/rfc/rfc4252) (auth), then [RFC 4254](https://www.rfc-editor.org/rfc/rfc4254) (connection). `ssh -vvv` spells out every message the client sends. Pair that with `tcpdump -i any -w ssh.pcap port 22` and you can watch the handshake happen in Wireshark. It is worth doing once - a lot of the abstractions collapse into "oh, that is just a packet."

The OpenSSH source is at [github.com/openssh/openssh-portable](https://github.com/openssh/openssh-portable). `kex.c`, `packet.c`, and `channels.c` map one-to-one onto the RFC sections above. If you ever need to know exactly what your ssh client is doing, the answer is in there.
