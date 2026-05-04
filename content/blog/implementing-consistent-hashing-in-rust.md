+++
title = "Implementing consistent hashing in Rust"
date = 2025-11-05
description = "Building a hash ring from scratch in ~100 lines of Rust - the modulo trap, virtual nodes, binary search on a sorted Vec, and why every distributed cache and shardable database leans on this trick."

[taxonomies]
tags = ["rust", "distributed-systems", "algorithms", "data-structures"]
+++

You have a million keys and ten cache servers. Question: which server owns key `user:42:profile`? The textbook answer is `hash(key) % 10`, and it works perfectly until you add an eleventh server. Now `hash(key) % 11` sends almost every key to a different machine. Roughly 90% of your cache invalidates in a single deploy. The site goes down for ten minutes while everything reheats from the database.

This is the problem [David Karger and his co-authors](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf) solved in 1997. The paper is titled "Consistent Hashing and Random Trees" and the key result is a hash function where adding or removing one node out of N moves only `K/N` keys instead of all of them. Akamai, the company Karger co-founded, used it to build the first commercial CDN. Today every shardable distributed system you can name uses some variant: Amazon Dynamo, Cassandra, Riak, Discord's session sharding, Memcached's `ketama` client, Cloudflare's DNS routing.

Let's build one in around 100 lines of Rust and look at what makes it work.

<!-- more -->

## Why modulo hashing falls apart

Modulo hashing is the obvious approach. You have N nodes indexed `0..N`. For each key, compute `hash(key) % N` and that's the owner.

```rust
fn modulo_owner(key: &str, nodes: &[&str]) -> usize {
    let mut hasher = std::collections::hash_map::DefaultHasher::new();
    std::hash::Hash::hash(&key, &mut hasher);
    (std::hash::Hasher::finish(&hasher) as usize) % nodes.len()
}
```

Two seconds, done. The problem is what happens when `nodes.len()` changes. Suppose you had 10 nodes and key `k` lived at index `3`. The hash is `12345`, so `12345 % 10 = 5` (let's pretend, it's an example). Now you add an 11th node. `12345 % 11 = 8`. The key has to move. This happens for almost every key.

Concretely: if you go from N to N+1 nodes, roughly `1 - 1/(N+1)` of your keys move to a new owner. For 10 -> 11 that is 91%. For 100 -> 101 that is 99%. Adding capacity gets *worse* as your cluster grows, which is the opposite of what you want. Removing a node has the same problem.

This is fine for a hash table inside a single process - you'd resize anyway, and the table sees every entry in memory. It's a disaster for a distributed cache, where moving a key means a network round trip, a database read, and a write to a new node.

## The ring

Karger's idea is to hash both keys and nodes onto the same circular keyspace. Imagine a clock face that goes from 0 to `2^64 - 1` and wraps around. Each node hashes to some point on the ring. Each key hashes to some point. A key belongs to the first node you hit walking clockwise from the key's position.

```
            node A (h=100)
                /
          ----+----
         /         \
        |   key X   |   key X hashes to 250
        |  (h=250)  |   walking clockwise: hits node B at 300
         \         /    so X is owned by B
          ----+----
            \
            node C (h=900)         node B (h=300)
```

Now add a node D at position 200. Which keys move? Only the keys that hashed between 100 (A) and 200 (D). They used to walk clockwise to B, now they stop at D. Every other key still resolves to the same node it always did. If hashes are uniformly distributed and the cluster has N nodes, adding one node reassigns roughly `1/N` of the keys. With N = 100, that's 1% redistribution instead of 99%.

That is the entire trick. The rest is implementation.

## A first implementation

We need: (1) a way to put node identifiers on the ring, (2) a way to look up the next node clockwise from a hash, (3) reasonable performance for both. The natural data structure is a sorted map from `u64` to node name. In Rust, `BTreeMap<u64, String>` works, but for our purposes a sorted `Vec<(u64, String)>` with binary search is faster on lookup, simpler to reason about, and matches what the C `ketama` library does. Cache lines beat tree pointers every time.

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

pub struct HashRing {
    ring: Vec<(u64, String)>, // sorted by u64
    vnodes_per_node: u32,
}

impl HashRing {
    pub fn new(vnodes_per_node: u32) -> Self {
        Self { ring: Vec::new(), vnodes_per_node }
    }

    fn hash<T: Hash>(value: &T) -> u64 {
        let mut h = DefaultHasher::new();
        value.hash(&mut h);
        h.finish()
    }

    pub fn add_node(&mut self, node: &str) {
        for v in 0..self.vnodes_per_node {
            let label = format!("{node}#{v}");
            let pos = Self::hash(&label);
            self.ring.push((pos, node.to_string()));
        }
        self.ring.sort_by_key(|&(p, _)| p);
    }

    pub fn remove_node(&mut self, node: &str) {
        self.ring.retain(|(_, n)| n != node);
    }

    pub fn get(&self, key: &str) -> Option<&str> {
        if self.ring.is_empty() { return None; }
        let h = Self::hash(&key);
        let idx = match self.ring.binary_search_by_key(&h, |&(p, _)| p) {
            Ok(i) => i,
            Err(i) => if i == self.ring.len() { 0 } else { i },
        };
        Some(self.ring[idx].1.as_str())
    }
}
```

That is the entire data structure. Forty lines. A test fits in another twenty:

```rust
#[test]
fn distributes_and_survives_resize() {
    let mut ring = HashRing::new(150);
    for n in ["a", "b", "c", "d"] { ring.add_node(n); }

    let keys: Vec<String> = (0..100_000).map(|i| format!("k{i}")).collect();
    let owners_before: Vec<&str> = keys.iter()
        .map(|k| ring.get(k).unwrap()).collect();

    ring.add_node("e");
    let owners_after: Vec<&str> = keys.iter()
        .map(|k| ring.get(k).unwrap()).collect();

    let moved = owners_before.iter().zip(&owners_after)
        .filter(|(a, b)| a != b).count();
    // Expected ~ 100_000 / 5 = 20_000. Actual within a few percent.
    assert!(moved < 25_000);
}
```

For modulo hashing the same test would show ~80,000 keys moving. The ratio `moved / total_keys` is the lever every distributed cache cares about.

## Why `binary_search_by_key` and `Err(i)`

The lookup deserves a closer read because it is where consistent hashing actually happens.

`Vec::binary_search_by_key` returns `Result<usize, usize>`. `Ok(i)` means an exact match at index `i`. `Err(i)` means no match, but `i` is the position where the element *would* be inserted to keep the vec sorted. That is exactly the next-clockwise position we want.

The wraparound is the special case. If `Err(i)` returns `self.ring.len()`, the key hashed past the last node on the ring. Walking clockwise wraps to position 0 - the first node in the sorted order. That is the `if i == self.ring.len() { 0 }` line.

`binary_search_by_key` is `O(log N)`, branch-prediction friendly, and reads sequential cache lines on the way down. With 1000 nodes and 150 vnodes each (150,000 entries), each lookup is around 18 comparisons on roughly 18 cache lines. On a modern CPU that is sub-microsecond. The C `ketama` implementation is structurally identical.

## The balance problem and virtual nodes

If you actually run the 4-node test above with `vnodes_per_node = 1`, you will discover that the load is awful. One node might end up owning 50% of the keys, another 5%. The hash function distributes uniformly only "in expectation," and with four points on a ring of `2^64`, the gaps between them are wildly uneven by random chance.

The fix is virtual nodes (vnodes). Instead of putting each physical node on the ring once, put it on 100-200 times under different labels. For node `cache-a`, hash `cache-a#0`, `cache-a#1`, ..., `cache-a#149`. All 150 positions resolve back to `cache-a` for ownership, but the ring now has 600 positions for 4 nodes instead of 4. By the law of large numbers, the gaps even out. Standard deviation of node load drops from ~30% with 1 vnode to ~3% with 150.

This is why every production implementation does it. Cassandra exposes it as `num_tokens` (default 256). The `ketama` Memcached client uses 160 by default. Riak used 64 ring partitions for many years before switching to its own scheme.

The cost is memory. With 150 vnodes per physical node and 1000 physical nodes, you have 150,000 entries. At maybe 40 bytes each (u64 + small String + heap overhead), that is about 6 MB. Trivial. Lookup is still `O(log(N * vnodes))`, which adds maybe 7 comparisons over `O(log N)`. Also trivial.

## What "in expectation" actually means

The math behind the ring is worth a beat. If you place `M` random points uniformly on a circle of length `L`, the gaps between consecutive points follow a [Dirichlet distribution](https://en.wikipedia.org/wiki/Dirichlet_distribution). The expected gap length is `L/M`, and the standard deviation of any single gap is roughly `L/M`. So with 4 nodes (M=4), one gap could easily be 2x the average, meaning that one node owns 2x its fair share.

With `M = 4 * 150 = 600`, the relative deviation drops by a factor of `sqrt(150) ~ 12`. That is the whole reason virtual nodes work - it is the central limit theorem doing the balancing.

There is a cleaner approach that does not need vnodes called [rendezvous hashing](https://en.wikipedia.org/wiki/Rendezvous_hashing) (also called HRW), and a more recent one called [jump consistent hash](https://arxiv.org/abs/1406.2294) from Google that uses no memory and is O(log N) without any ring at all. Both have tradeoffs: rendezvous hashing is O(N) per lookup, jump hash does not handle arbitrary node names. The classic ring with vnodes wins in practice because it is `O(log N)`, supports node names, and survives churn.

## Where this shows up in production

**Memcached clients.** The original `ketama` algorithm by Last.fm is exactly the ring-with-vnodes design above, with MD5 as the hash. Every modern memcached client - `pylibmc`, `dalli`, the Go `gomemcache` library - implements it. Switching servers in a memcached pool moves only `1/N` of your keys, so cache hit rate barely dips during a deploy.

**Cassandra and ScyllaDB.** Both partition data using a token ring. Each node owns one or more tokens (Cassandra's `num_tokens`), and a row's partition key is hashed (Murmur3) to find its owner. Bootstrapping a new node streams in only the keys whose tokens fall into the new node's slice. The [Cassandra docs on token allocation](https://cassandra.apache.org/doc/latest/cassandra/operating/topo_changes.html) describe this in detail.

**DynamoDB.** Amazon's [original Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) (Section 4.3) literally walks through consistent hashing and the virtual node optimization as the foundation of the storage layer. It is the textbook case.

**Discord's session sharding.** Discord routes each user's gateway connection to a session server using consistent hashing on the user ID. They can add session servers without disconnecting most users. The eng blog has [a writeup](https://discord.com/blog/scaling-elixir-f9b8e1e7c29b) on the broader architecture.

**Cloudflare's load balancers.** Cloudflare's L4 load balancer uses consistent hashing (specifically [maglev hashing](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/), Google's variant) to keep TCP connections sticky across backend changes. When a backend goes away, only its connections drop. Without consistent hashing, every connection would rebalance.

## Where it doesn't help

Consistent hashing is a tool for distributing *independent* keys. It does not help if your workload is hot on one key. If everyone in your system is reading `user:bieber:followers`, that key still lives on one node, and that node is on fire. The fix there is replication (multiple owners), request coalescing, or a per-key cache layer in front. Consistent hashing only solves the question "which node should this key go to," not "what if every key is the same key."

It also does not solve replication. The ring tells you the *primary* owner. If you want replicas, the standard pattern is to also assign the key to the next R-1 nodes clockwise on the ring. Dynamo does exactly this.

## The full file

For reference, here is the whole implementation as a single file. It compiles and runs as `cargo test`.

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

pub struct HashRing {
    ring: Vec<(u64, String)>,
    vnodes_per_node: u32,
}

impl HashRing {
    pub fn new(vnodes_per_node: u32) -> Self {
        Self { ring: Vec::new(), vnodes_per_node }
    }

    fn hash<T: Hash>(value: &T) -> u64 {
        let mut h = DefaultHasher::new();
        value.hash(&mut h);
        h.finish()
    }

    pub fn add_node(&mut self, node: &str) {
        for v in 0..self.vnodes_per_node {
            let label = format!("{node}#{v}");
            let pos = Self::hash(&label);
            self.ring.push((pos, node.to_string()));
        }
        self.ring.sort_by_key(|&(p, _)| p);
    }

    pub fn remove_node(&mut self, node: &str) {
        self.ring.retain(|(_, n)| n != node);
    }

    pub fn get(&self, key: &str) -> Option<&str> {
        if self.ring.is_empty() { return None; }
        let h = Self::hash(&key);
        let idx = match self.ring.binary_search_by_key(&h, |&(p, _)| p) {
            Ok(i) => i,
            Err(i) => if i == self.ring.len() { 0 } else { i },
        };
        Some(self.ring[idx].1.as_str())
    }

    pub fn get_n(&self, key: &str, n: usize) -> Vec<&str> {
        if self.ring.is_empty() { return Vec::new(); }
        let h = Self::hash(&key);
        let start = match self.ring.binary_search_by_key(&h, |&(p, _)| p) {
            Ok(i) => i,
            Err(i) => if i == self.ring.len() { 0 } else { i },
        };
        let mut out: Vec<&str> = Vec::with_capacity(n);
        let mut i = start;
        while out.len() < n && out.len() < self.ring.len() {
            let candidate = self.ring[i].1.as_str();
            if !out.contains(&candidate) {
                out.push(candidate);
            }
            i = (i + 1) % self.ring.len();
            if i == start { break; }
        }
        out
    }
}
```

`get_n` is the replication helper - walk clockwise collecting distinct physical nodes until you have R of them. That is the full Dynamo-style replica set logic in 15 lines.

A few caveats before you ship this. `DefaultHasher` is `SipHash`, which is fine for correctness but slower than `xxhash` or `fxhash` for this use case. For production, swap in a fast non-cryptographic hash. Also, `format!("{node}#{v}")` allocates - if you're rebuilding the ring constantly you may want a custom labeling scheme that avoids allocation. Neither matters for understanding the algorithm.

## What changes at scale

The reason every serious distributed system reaches for consistent hashing eventually is not that it is clever. It is that the alternatives all break in different but predictable ways. Modulo hashing breaks on resize. Static range partitioning breaks when a range goes hot. Manual sharding breaks when you need to add capacity at 3am. Consistent hashing degrades gracefully: you add a node, a fraction of keys move, the rest of the system does not notice.

The whole thing is fewer than 100 lines of straightforward code. If you have ever wondered why Cassandra "just works" when you grow the cluster, or why your memcached deploys do not nuke the cache, this is the trick.
