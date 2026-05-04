+++
title = "Docker networking explained - bridge, host, overlay"
date = 2025-08-22
description = "How Docker networks actually work under the hood - veth pairs, iptables NAT, VXLAN encapsulation, embedded DNS, and when to pick each mode."

[taxonomies]
tags = ["docker", "networking", "devops", "linux"]
+++

Every Docker tutorial tells you to slap `-p 8080:80` on your `docker run` command and move on. It works. Traffic reaches your container. But what actually happened? Where did that port mapping go? Why can one container talk to another by name on a Compose network but not on the default bridge? Why does your overlay network drop packets under load when you enable encryption?

Docker networking is Linux networking with some automation on top. Once you see the primitives - network namespaces, veth pairs, bridges, iptables rules - the magic disappears and you can actually debug problems instead of restarting the daemon and hoping.

<!-- more -->

## The Container Network Model

Docker's networking architecture is built on the Container Network Model (CNM), implemented in [libnetwork](https://github.com/moby/moby/tree/master/libnetwork). Three abstractions make up the whole thing:

- **Sandbox** - an isolated network stack. Each container gets one. Under the hood, this is a Linux [network namespace](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) - its own interfaces, routing table, iptables rules, and DNS config, completely separate from the host.
- **Endpoint** - a virtual network interface that connects a Sandbox to a Network. Implemented as one end of a [veth pair](https://man7.org/linux/man-pages/man4/veth.4.html). Think of it as a virtual Ethernet cable with one plug inside the container and the other plug on the network.
- **Network** - a group of Endpoints that can communicate. Depending on the driver, this could be a Linux bridge, a VXLAN tunnel, or the host's own network stack.

Different network drivers (`bridge`, `host`, `overlay`, `macvlan`, `ipvlan`, `none`) implement these abstractions using different Linux primitives. The driver you choose determines isolation, performance, and what's possible.

## Bridge - the default

When you run `docker run` without specifying a network, your container lands on the default bridge network, backed by a Linux bridge interface called `docker0`.

### What happens when a container starts

Here's the sequence, roughly equivalent to what Docker automates:

```bash
# 1. Docker creates a network namespace for the container
ip netns add container0

# 2. Creates a veth pair - a virtual ethernet cable
ip link add veth0 type veth peer name ceth0

# 3. Moves one end into the container's namespace
ip link set ceth0 netns container0

# 4. Attaches the host end to the docker0 bridge
ip link set veth0 master docker0
ip link set veth0 up

# 5. Configures the container end with an IP from the bridge subnet
ip netns exec container0 ip addr add 172.17.0.2/16 dev ceth0
ip netns exec container0 ip link set ceth0 up
ip netns exec container0 ip route add default via 172.17.0.1
```

The result: your container has an `eth0` interface (the `ceth0` end of the veth pair) with an IP on the `172.17.0.0/16` subnet. The other end of the veth pair is plugged into `docker0`, which acts as a Layer 2 switch between all containers on that network.

You can verify this on any Docker host:

```bash
$ brctl show docker0
bridge name   bridge id           STP enabled   interfaces
docker0       8000.0242ac110001   no            veth7a2b3c4
                                                veth9d8e7f6

$ ip link show type veth
5: veth7a2b3c4@if4: <BROADCAST,MULTICAST,UP> ...
7: veth9d8e7f6@if6: <BROADCAST,MULTICAST,UP> ...
```

Each `vethXXX` interface on the host corresponds to one container. The `@ifN` suffix is the interface index of the peer inside the container's namespace.

### NAT and outbound traffic

Containers need to reach the internet. Since `172.17.0.0/16` is a private range that no external router knows about, Docker sets up masquerading (SNAT) via iptables:

```bash
$ iptables -t nat -L POSTROUTING -v
Chain POSTROUTING (policy ACCEPT)
target     prot opt in   out     source         destination
MASQUERADE all  --  any  !docker0 172.17.0.0/16  anywhere
```

Translation: any packet from the bridge subnet (`172.17.0.0/16`) leaving through an interface that isn't `docker0` gets its source IP rewritten to the host's IP. Standard NAT. The kernel's conntrack module tracks the mapping so return traffic finds its way back.

Docker also enables IP forwarding on the host (`net.ipv4.ip_forward = 1`). Without it, the kernel would drop forwarded packets instead of routing them.

### Network isolation between bridges

Docker prevents containers on different bridge networks from talking to each other using a two-stage isolation chain:

```bash
$ iptables -L DOCKER-ISOLATION-STAGE-1 -v
Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target                    in          out
DOCKER-ISOLATION-STAGE-2  br-abc123   !br-abc123
DOCKER-ISOLATION-STAGE-2  docker0     !docker0

$ iptables -L DOCKER-ISOLATION-STAGE-2 -v
Chain DOCKER-ISOLATION-STAGE-2 (2 references)
target   in   out
DROP     any  br-abc123
DROP     any  docker0
```

If a packet enters through one bridge and tries to leave through a different bridge, it gets dropped. This is why containers on separate user-defined networks can't reach each other without explicitly connecting them to the same network.

### Default bridge vs user-defined bridge

This is a critical distinction that trips up a lot of people. The default `docker0` bridge and a user-defined bridge (`docker network create mynet`) behave differently:

| Feature | Default bridge (`docker0`) | User-defined bridge |
|---------|--------------------------|-------------------|
| DNS resolution by container name | No | Yes |
| Automatic DNS via embedded server | No | Yes |
| ICC (inter-container connectivity) | All containers can reach all others | Configurable, default on |
| Connect/disconnect without restart | No | Yes |
| Link-based environment variables | Yes (legacy) | No |
| Subnet/gateway customization at creation | Limited | Full control |

The biggest difference: **DNS**. On the default bridge, containers can only reach each other by IP address. On a user-defined bridge, Docker's embedded DNS server resolves container names to IPs automatically. This is why `docker-compose` services can reach each other by name - Compose creates a user-defined bridge, not the default one.

## Host - no isolation, no overhead

```bash
docker run --network host nginx
```

Host mode eliminates the network namespace entirely. The container shares the host's network stack - same interfaces, same IP addresses, same port space. No veth pairs, no bridge, no NAT, no docker-proxy.

When nginx binds to port 80 inside the container, it binds to port 80 on the host. The `-p` flag is ignored (Docker prints a warning if you try).

### When host mode makes sense

- **Performance-sensitive workloads** - bridge networking adds 15-50 microseconds of latency per packet from iptables NAT traversal, veth pair copying, and bridge forwarding. Host mode eliminates all of it. In throughput tests, host mode can push ~40 Gbps where bridge tops out around 20-35 Gbps.
- **Containers that need to see all host network interfaces** - monitoring agents, network tools.
- **Applications that bind dynamic port ranges** - like SIP servers or P2P protocols where mapping hundreds of ports is impractical.

### The tradeoffs

Host mode removes all network isolation. A compromised container has full access to every interface, every listening port, and every network connection on the host. It can also bind to any port, potentially conflicting with host services or other host-mode containers.

It's also **Linux only**. Docker Desktop on Mac and Windows runs containers inside a Linux VM, so "host" mode gives you the VM's network, not your Mac's.

## None - full isolation

```bash
docker run --network none alpine ip addr
1: lo: <LOOPBACK,UP> mtu 65536
    inet 127.0.0.1/8 scope host lo
```

The `none` driver gives the container a network namespace with only a loopback interface. No external connectivity at all. Useful for batch processing jobs that only read from mounted volumes, or security-sensitive workloads that should never touch the network. You can manually add interfaces to the namespace later if needed.

## Overlay - networking across hosts

Bridge networks are local to a single Docker host. When you need containers on different machines to talk to each other directly - as in a Swarm cluster - you need overlay networking.

### VXLAN under the hood

Overlay networks use [VXLAN (Virtual Extensible LAN)](https://datatracker.ietf.org/doc/html/rfc7348), a standard encapsulation protocol that's been in the Linux kernel since version 3.7. The idea is straightforward: take the container's Ethernet frame, wrap it in a UDP packet, and send it across the physical network to the other host, where it gets unwrapped and delivered to the destination container.

The packet structure looks like this:

```
[Outer Ethernet Header]
  [Outer IP Header (host-to-host)]
    [Outer UDP Header (dst port 4789)]
      [VXLAN Header (24-bit VNI)]
        [Original Container Ethernet Frame]
          [Container IP Header]
            [Container TCP/UDP Header]
              [Payload]
```

Each overlay network gets a unique 24-bit VXLAN Network Identifier (VNI). The outer IP header contains the physical host IPs - these are routable on the underlay network. The inner frame is the original container traffic, which only makes sense within the overlay.

This encapsulation adds roughly **50 bytes of overhead per packet**. With a standard 1500-byte MTU on the underlay, the effective MTU for container traffic drops to around 1450 bytes. Docker sets this automatically, but if you're troubleshooting mysterious connection stalls (especially with `DF` bit set), MTU mismatch is the first thing to check.

### The control plane

Docker Swarm's overlay networking has two planes:

- **Data plane** - VXLAN encapsulated traffic on UDP port 4789.
- **Control plane** - a gossip protocol on port 7946 (TCP and UDP) that distributes which containers live on which hosts.

When a container on Host A wants to talk to a container on Host B, the local VXLAN interface needs to know that the destination MAC address lives on Host B. In a traditional VXLAN setup, you'd configure this manually or use a multicast group for BUM (Broadcast, Unknown unicast, Multicast) traffic. Docker Swarm skips multicast entirely - the gossip protocol distributes ARP and FDB (Forwarding Database) entries to all participating nodes. The VXLAN interface does ARP proxying using this distributed state.

This means overlay networks work across any IP-routable infrastructure - public cloud, on-prem, even across the internet - as long as the three ports are reachable (2377/tcp for Swarm management, 7946/tcp+udp for gossip, 4789/udp for VXLAN data).

### Overlay encryption - tread carefully

You can encrypt overlay traffic with `--opt encrypted`:

```bash
docker network create --driver overlay --opt encrypted my-secure-net
```

This establishes IPsec tunnels (ESP in transport mode with AES-GCM) between every pair of nodes that have tasks on the network. Keys rotate every 12 hours automatically.

Sounds great in theory. In practice, encrypted overlay has a known and severe performance problem. [moby/moby#33133](https://github.com/moby/moby/issues/33133) documents throughput dropping to roughly 1% of unencrypted overlay speeds. There's also [moby/moby#52005](https://github.com/moby/moby/issues/52005), where the 32-bit IPsec ESP sequence number overflows within the 12-hour key rotation interval on high-throughput links, causing silent packet drops.

If you need encrypted container traffic across hosts, consider a service mesh with mTLS (Linkerd, Istio) or WireGuard at the host level instead of Docker's built-in overlay encryption.

### Performance characteristics

Overlay adds latency on top of bridge networking - you're paying for VXLAN encapsulation/decapsulation plus the gossip protocol's state synchronization. Typical numbers:

| Mode | Throughput | Added latency |
|------|-----------|---------------|
| Host | ~40 Gbps | ~0 us |
| Bridge | ~20-35 Gbps | ~15-50 us |
| Overlay | ~15-25 Gbps | ~25-75 us |
| Overlay (encrypted) | ~0.3 Gbps | significant |

NICs with hardware VXLAN offloading can reduce CPU usage by around 30% for overlay traffic. Check your NIC's capabilities with `ethtool -k eth0 | grep vxlan`.

One more thing: overlay networks become unstable when approaching ~1000 containers on a single network. This is a Linux kernel limitation on the bridge/FDB side, not Docker-specific.

## macvlan and ipvlan - brief mention

Two more drivers worth knowing about:

**macvlan** assigns each container a unique MAC address and connects it directly to the physical network. No bridge, no NAT. The container appears as a separate physical device on the LAN. Requires promiscuous mode on the parent interface. Best for legacy applications that expect to be directly on the network or need DHCP with unique MACs. Most cloud providers block it.

**ipvlan** is similar but all containers share the parent interface's MAC address, getting only unique IPs. Works where promiscuous mode is unavailable or switches enforce MAC limits per port. L2 mode behaves like macvlan; L3 mode turns the host into a router.

Both provide near-host-level performance since they bypass the bridge entirely.

## DNS resolution - the 127.0.0.11 trick

On user-defined networks, Docker runs an embedded DNS server that resolves container names to their IPs. This server lives at `127.0.0.11` inside every container on a user-defined network.

But here's the clever part: the DNS server process is actually part of `dockerd`, running in the host's PID namespace. It doesn't use port 53 - it binds to a random high port to avoid conflicts with user applications that might run their own DNS. Docker then installs iptables DNAT rules inside the container's namespace to redirect port 53 traffic to the actual port:

```bash
# Inside a container on a user-defined network:
$ iptables -t nat -L OUTPUT
Chain OUTPUT (policy ACCEPT)
target  prot opt source    destination
DNAT    tcp  --  anywhere  127.0.0.11  tcp dpt:domain to:127.0.0.11:43500
DNAT    udp  --  anywhere  127.0.0.11  udp dpt:domain to:127.0.0.11:55432
```

The resolution flow:
1. Container queries `127.0.0.11:53`
2. iptables DNAT redirects to `127.0.0.11:<random-port>`
3. dockerd's embedded DNS server receives the query
4. If the name matches a container/service on the same network, it returns the container's IP
5. Otherwise, it forwards to the upstream DNS servers from the host's `/etc/resolv.conf`

On the default bridge, none of this exists. Docker just copies the host's `/etc/resolv.conf` into the container. Container name resolution doesn't work - you're stuck with IP addresses or the legacy `--link` flag.

You can see the source for this in [moby/libnetwork/resolver.go](https://github.com/moby/libnetwork/blob/master/resolver.go).

## Port mapping - what -p actually does

When you run `docker run -p 8080:80 nginx`, Docker does two things:

**1. iptables DNAT rule**

```bash
$ iptables -t nat -L DOCKER -v
Chain DOCKER (2 references)
target  prot opt in         out  source    destination
DNAT    tcp  --  !docker0   any  anywhere  anywhere  tcp dpt:8080 to:172.17.0.2:80
```

Any TCP packet arriving on port 8080 from outside the bridge gets its destination rewritten to `172.17.0.2:80` (the container's IP and port). The kernel's conntrack handles return traffic.

**2. docker-proxy process**

```bash
$ ps aux | grep docker-proxy
root  12345  docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 \
      -container-ip 172.17.0.2 -container-port 80
```

Docker spawns a userspace proxy that binds to `0.0.0.0:8080` on the host. This handles an edge case: traffic originating from the host itself (via `localhost:8080`) can't be routed through iptables PREROUTING rules because localhost traffic never hits that chain. The docker-proxy catches it in userspace and forwards it.

Each published port spawns a separate docker-proxy process. If you're publishing dozens of ports, that's dozens of processes. You can disable the proxy with `--userland-proxy=false` in the daemon config, but then localhost access to published ports won't work.

Worth noting: docker-proxy binds to `0.0.0.0` by default. If you run `-p 8080:80` thinking it's only accessible locally, it's not - it's exposed on all interfaces. Use `-p 127.0.0.1:8080:80` to bind only to localhost.

## docker-compose networking

Compose creates a user-defined bridge network for each project automatically, named `<project>_default`. All services join this network unless you specify otherwise.

```yaml
# docker-compose.yml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
  api:
    image: myapi
  db:
    image: postgres
```

Running `docker compose up` in a directory called `myapp` creates a network called `myapp_default`. All three services join it. The `api` service can connect to Postgres at `db:5432` - no IP addresses, no links, just the service name as hostname. This works because Compose uses a user-defined bridge with Docker's embedded DNS.

You can define multiple networks to isolate services:

```yaml
services:
  web:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
```

Now `web` can reach `api` but not `db`. The `api` service bridges both networks. Docker's iptables isolation chains (DOCKER-ISOLATION-STAGE-1/2) enforce the boundary.

## When to use which

**Default bridge** - never, really. Use a user-defined bridge instead. You get DNS resolution, better isolation control, and the ability to connect/disconnect containers without restarting them.

**User-defined bridge** - single-host setups, local development, most production deployments where all containers run on one machine. This is the right choice 90% of the time.

**Host** - latency-sensitive applications where every microsecond counts, network monitoring tools, or containers that need raw access to host interfaces. Accept the security tradeoff.

**Overlay** - multi-host deployments with Docker Swarm. If you're using Kubernetes, you'll use a CNI plugin (Calico, Cilium, Flannel) instead of Docker's overlay driver, but the underlying VXLAN concepts are the same.

**macvlan/ipvlan** - when containers need to appear as first-class devices on the physical network. Common for legacy applications, IoT gateways, or DHCP-dependent setups.

**None** - batch jobs, security-sensitive processing, anything that shouldn't have network access.

## Security implications

Each network mode carries different security characteristics. Here's what to watch for:

**Bridge mode** exposes containers to ARP spoofing from other containers on the same bridge. By default, containers have `CAP_NET_RAW`, which allows crafting arbitrary packets. A compromised container can ARP-spoof the bridge gateway, intercepting or modifying traffic from neighboring containers. Drop the capability if you don't need it:

```bash
docker run --cap-drop=NET_RAW myimage
```

The `docker-proxy` binding to `0.0.0.0` is another common exposure. Published ports are reachable from anywhere unless you bind them to a specific interface. And the `DOCKER-USER` iptables chain - where you're supposed to add custom firewall rules - is empty by default. Docker's own rules run after `DOCKER-USER`, so if you want to restrict external access to published ports, that's where your rules go.

**Host mode** is the most dangerous from a network security perspective. No namespace isolation means the container can bind any port, sniff any interface, and interact with every network service on the host. Only use it when you understand and accept this.

**Overlay mode** transmits container traffic as cleartext VXLAN by default. Any machine on the underlay network can capture and read it. The built-in encryption (`--opt encrypted`) has the performance problems mentioned earlier. For production, consider mTLS at the application or service mesh layer rather than relying on overlay encryption.

A broader concern: Docker modifies iptables rules globally. If you're running a host firewall (ufw, firewalld), Docker's rules can bypass it. The `DOCKER-USER` chain exists specifically to address this, but it requires manual setup. Docker [documents this](https://docs.docker.com/engine/network/packet-filtering-firewalls/) - read it before deploying containers on a machine that faces the internet.

Recent Docker networking CVEs to be aware of: [CVE-2025-9074](https://docs.docker.com/security/security-announcements/) allowed locally running containers to access the Docker Engine API through the default bridge subnet, potentially enabling container escape. Fixed in Docker Desktop 4.44.3.

## Wrapping up

Docker networking is a thin layer of automation over standard Linux networking primitives. The bridge driver creates veth pairs and Linux bridges. NAT uses iptables. Overlay uses VXLAN. DNS is an embedded server with iptables DNAT tricks. Once you know which primitive maps to which Docker concept, debugging becomes straightforward - `ip link`, `brctl show`, `iptables -t nat -L`, and `nsenter` are your tools.

If you're also containerizing Rust services, I covered how to get your image sizes under control in [Docker multi-stage builds for Rust - from 2GB to 20MB](/blog/docker-multi-stage-builds-for-rust-from-2gb-to-20mb).

For the v29 release, keep an eye on the [nftables backend](https://docs.docker.com/engine/network/firewall-nftables/) - iptables is on its way out, and Docker is following. The concepts stay the same, but the tooling for inspecting rules will change.
