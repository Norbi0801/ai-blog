+++
title = "Implementing a Merkle tree in Rust"
date = 2025-04-06
description = "Building a Merkle tree from scratch in ~100 lines of Rust - hash construction, inclusion proofs, verification, and why this data structure powers Git, Bitcoin, and certificate transparency."

[taxonomies]
tags = ["rust", "algorithms", "cryptography", "distributed-systems"]
+++

Every time you run `git commit`, you're building a Merkle tree. Every Bitcoin block contains one. Every TLS certificate you trust was logged in one. The Merkle tree is one of those data structures that shows up everywhere in distributed systems, yet most developers have never implemented one from scratch.

The core idea is simple: hash your data at the leaves, then recursively hash pairs of hashes until you reach a single root. That root is a fingerprint of the entire dataset. Change one byte anywhere, and the root changes. But the real power comes from proofs - you can prove that a specific piece of data exists in the tree by providing only O(log n) hashes instead of the entire dataset.

Let's build one.

<!-- more -->

## The structure

A Merkle tree is a binary tree where:

- **Leaf nodes** hold hashes of the actual data
- **Internal nodes** hold hashes of their two children concatenated together
- **The root** is a single hash that commits to the entire dataset

Here's what a tree with four items looks like:

```
            root = H(H01 || H23)
           /                    \
     H01 = H(H0 || H1)    H23 = H(H2 || H3)
       /        \            /        \
   H0 = H(A)  H1 = H(B)  H2 = H(C)  H3 = H(D)
      |           |           |           |
      A           B           C           D
```

If someone gives you the root hash, and you want to verify that `B` is in the tree, you don't need all the data. You just need `H0` (B's sibling) and `H23` (the sibling of the parent). Two hashes instead of four items. With a million items, that's 20 hashes instead of a million. That's the point.

## Setting up

We need a hash function. SHA-256 is the standard choice for Merkle trees - it's what Bitcoin, Git (as of its SHA-256 transition), and Certificate Transparency all use.

```toml
# Cargo.toml
[dependencies]
sha2 = "0.11"
```

The `sha2` crate is part of the [RustCrypto](https://github.com/RustCrypto/hashes) project. It gives us a `Sha256` struct that implements the `Digest` trait for incremental hashing.

## Hash functions with domain separation

Before we build the tree, we need two hash functions - one for leaves and one for internal nodes:

```rust
use sha2::{Sha256, Digest};

type Hash = [u8; 32];

fn hash_leaf(data: &[u8]) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update([0x00]);
    hasher.update(data);
    hasher.finalize().into()
}

fn hash_pair(left: &Hash, right: &Hash) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update([0x01]);
    hasher.update(left);
    hasher.update(right);
    hasher.finalize().into()
}
```

Notice the `0x00` and `0x01` prefix bytes. This is called **domain separation**, and it's not optional if you care about security.

Without domain separation, an attacker could construct a fake "leaf" whose content happens to be the concatenation of two child hashes. The tree would accept it as a valid internal node. This is a second preimage attack - you find a different input that produces the same root hash.

[RFC 6962](https://www.rfc-editor.org/rfc/rfc6962.html) (Certificate Transparency) defines exactly this scheme: `0x00` for leaf nodes, `0x01` for internal nodes. By prefixing the hash input, a leaf can never collide with an internal node because their hash domains are disjoint.

The `finalize()` call returns a `GenericArray<u8, U32>`, and `.into()` converts it to our `[u8; 32]` type alias. Clean.

## Building the tree

The tree is stored as layers - the bottom layer is the leaf hashes, each subsequent layer is half the size (rounding up), and the top layer is a single hash: the root.

```rust
pub struct MerkleTree {
    layers: Vec<Vec<Hash>>,
}

impl MerkleTree {
    pub fn build(data: &[&[u8]]) -> Self {
        assert!(!data.is_empty(), "cannot build a tree from empty data");

        let leaves: Vec<Hash> = data.iter().map(|d| hash_leaf(d)).collect();
        let mut layers = vec![leaves];

        while layers.last().unwrap().len() > 1 {
            let prev = layers.last().unwrap();
            let next = prev
                .chunks(2)
                .map(|pair| {
                    if pair.len() == 2 {
                        hash_pair(&pair[0], &pair[1])
                    } else {
                        // Odd number of nodes: duplicate the last one
                        hash_pair(&pair[0], &pair[0])
                    }
                })
                .collect();
            layers.push(next);
        }

        MerkleTree { layers }
    }

    pub fn root(&self) -> &Hash {
        &self.layers.last().unwrap()[0]
    }

    pub fn leaf_count(&self) -> usize {
        self.layers[0].len()
    }
}
```

The `chunks(2)` iterator gives us pairs of hashes. When there's an odd number, the last chunk has one element, and we duplicate it. This is the same approach Bitcoin uses - [the Bitcoin wiki](https://en.bitcoin.it/wiki/Block_hashing_algorithm) documents this explicitly.

One thing worth noting: we're building bottom-up with `Vec<Vec<Hash>>`. Each `Hash` is 32 bytes on the stack (it's a `[u8; 32]`, a fixed-size array, `Copy`). No heap allocation per hash, no `Box`, no `Rc`. The only heap allocations are the `Vec`s themselves. For a tree with n leaves, we store 2n - 1 hashes total (geometric series), so memory usage is roughly 64n bytes. That's it.

An alternative design would use a single flat `Vec<Hash>` with index arithmetic (like a binary heap), but layers make the proof generation code much clearer.

## Generating inclusion proofs

This is where Merkle trees earn their keep. Given a leaf index, we walk up the tree collecting the sibling hash at each level. The verifier can then reconstruct the path from leaf to root.

```rust
#[derive(Debug, Clone)]
pub enum Side {
    Left,
    Right,
}

#[derive(Debug, Clone)]
pub struct ProofEntry {
    pub hash: Hash,
    pub side: Side,
}

impl MerkleTree {
    pub fn proof(&self, index: usize) -> Vec<ProofEntry> {
        assert!(index < self.leaf_count(), "leaf index out of range");

        let mut entries = Vec::new();
        let mut idx = index;

        for layer in &self.layers[..self.layers.len() - 1] {
            let (sibling_idx, side) = if idx % 2 == 0 {
                // We're a left child, sibling is on the right
                (idx + 1, Side::Right)
            } else {
                // We're a right child, sibling is on the left
                (idx - 1, Side::Left)
            };

            let sibling_hash = if sibling_idx < layer.len() {
                layer[sibling_idx]
            } else {
                // Odd layer: no sibling exists, duplicate ourselves
                layer[idx]
            };

            entries.push(ProofEntry {
                hash: sibling_hash,
                side,
            });

            idx /= 2; // Move to parent index
        }

        entries
    }
}
```

The `Side` enum records where the sibling sits relative to us. This matters during verification - hash concatenation is not commutative. `H(A || B) != H(B || A)`.

The `idx /= 2` at the end of each iteration maps a node's position in one layer to its parent's position in the next layer. Left child at index 4 and right child at index 5 both map to parent at index 2. Integer division handles this naturally.

Proof size is `layers.len() - 1`, which equals `ceil(log2(n))`. For a tree with 1 million leaves, that's 20 entries - 20 * 32 = 640 bytes.

## Verification

The verifier has four things: the root hash, the raw data they want to check, the leaf index, and the proof. They don't need the full tree.

```rust
pub fn verify(root: &Hash, data: &[u8], proof: &[ProofEntry]) -> bool {
    let mut current = hash_leaf(data);

    for entry in proof {
        current = match entry.side {
            Side::Left => hash_pair(&entry.hash, &current),
            Side::Right => hash_pair(&current, &entry.hash),
        };
    }

    current == *root
}
```

That's the entire verifier. Walk the proof bottom-up, combining the current hash with each sibling in the correct order, and check if you arrive at the expected root.

If `Side::Left`, the sibling was to our left, so it goes first: `H(sibling || current)`. If `Side::Right`, we go first: `H(current || sibling)`. One mismatch anywhere in the chain and the final hash won't match the root.

## Putting it all together

```rust
fn main() {
    let data: Vec<&[u8]> = vec![
        b"transaction_001",
        b"transaction_002",
        b"transaction_003",
        b"transaction_004",
        b"transaction_005",
    ];

    let tree = MerkleTree::build(&data);

    println!("Root: {}", hex::encode(tree.root()));
    println!("Leaves: {}", tree.leaf_count());

    // Prove that transaction_002 (index 1) is in the tree
    let proof = tree.proof(1);
    println!("\nProof for index 1 ({} entries):", proof.len());
    for (i, entry) in proof.iter().enumerate() {
        println!(
            "  level {}: {} ({:?})",
            i,
            &hex::encode(entry.hash)[..16],
            entry.side
        );
    }

    // Verify
    let valid = verify(tree.root(), b"transaction_002", &proof);
    println!("\nVerification: {}", if valid { "PASS" } else { "FAIL" });

    // Tamper with the data and verify again
    let tampered = verify(tree.root(), b"transaction_099", &proof);
    println!("Tampered:     {}", if tampered { "PASS" } else { "FAIL" });
}
```

If you add `hex = "0.4"` to your `Cargo.toml` and run this, you'll get output like:

```
Root: 0a1b2c3d...  (64 hex chars)
Leaves: 5

Proof for index 1 (3 entries):
  level 0: 7f83b1657ff1fc53 (Left)
  level 1: 3e2d4f8a91bc07e5 (Right)
  level 2: a8c5d9e2f1034b67 (Right)

Verification: PASS
Tampered:     FAIL
```

Three proof entries for five leaves. `ceil(log2(5))` = 3. The math checks out.

## What's happening in memory

Let's look at what the compiler actually produces. Our `Hash` type is `[u8; 32]` - a fixed-size array that lives on the stack when used as a local variable, or inline inside a `Vec`'s heap buffer. No indirection. No pointer chasing.

A `ProofEntry` is 33 bytes in theory (32 for the hash + 1 for the enum discriminant), but the compiler aligns it to 34 bytes (1 byte padding after the `Side` enum to align the next entry). You can verify this:

```rust
println!("Size of ProofEntry: {}", std::mem::size_of::<ProofEntry>());
// Prints: 33
```

Actually, the compiler packs `Side` (1 byte) right before or after the `[u8; 32]`, and since `[u8; 32]` has alignment 1, there's no padding needed. The struct is exactly 33 bytes. No waste.

Compare this to a naive implementation using `Vec<u8>` for hashes - each hash would require a separate heap allocation (24 bytes for the Vec header + 32 bytes on the heap + allocator overhead). With `[u8; 32]`, everything is contiguous in memory, which means better cache locality when iterating through proof entries.

## Where Merkle trees live in the real world

### Git

Git's object model is a Merkle DAG (directed acyclic graph, not strictly a tree since commits can have multiple parents). Every blob, tree, and commit object is content-addressed by its SHA hash. A tree object lists the hashes of its child blobs and subtrees. Change one file, and the hash changes propagate up through every tree object to the commit.

This is why `git diff` between two commits is fast - Git walks both trees from the root and stops descending into subtrees where the hashes match. If two subtrees have the same hash, their contents are identical. No need to compare files.

The [Pro Git book](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) has an excellent walkthrough of this object model.

### Bitcoin

Every Bitcoin block header contains a Merkle root of all transactions in that block. This enables [Simplified Payment Verification (SPV)](https://developer.bitcoin.org/devguide/block_chain.html) - a lightweight client can verify that a transaction was included in a block by downloading only the 80-byte block header and a Merkle proof, instead of the full block (which can be several megabytes).

Bitcoin uses double-SHA-256 (`SHA256(SHA256(data))`) instead of single SHA-256. This is a defense against [length-extension attacks](https://en.wikipedia.org/wiki/Length_extension_attack) that affect Merkle-Damgard hash functions like SHA-256. Our implementation uses domain separation instead, which achieves the same goal more cleanly.

Bitcoin also duplicates the last hash when there's an odd number of nodes - exactly what our implementation does.

### Certificate Transparency

[RFC 6962](https://www.rfc-editor.org/rfc/rfc6962.html) defines Certificate Transparency (CT) logs as append-only Merkle trees of TLS certificates. When a certificate authority issues a cert, it gets logged in a public CT log. Anyone can verify that a specific certificate was logged by requesting a Merkle inclusion proof from the log server.

CT logs define two types of proofs:

- **Audit proofs** (inclusion): prove a certificate exists in the log - this is what we implemented
- **Consistency proofs**: prove that the log is append-only between two points in time, meaning no prior entries were modified or deleted

The domain separation scheme in our implementation (`0x00` for leaves, `0x01` for internal nodes) comes directly from RFC 6962. We didn't invent it - we borrowed it from the most battle-tested Merkle tree specification in production.

Russ Cox wrote an excellent deep dive on this topic: [Transparent Logs for Skeptical Clients](https://research.swtch.com/tlog).

## Why this works for distributed systems

Three properties make Merkle trees uniquely suited for distributed systems:

**Tamper evidence.** The root hash commits to the entire dataset. If a node modifies any piece of data, anyone with the root hash can detect it. This is why blockchains use Merkle trees - you don't need to trust the node serving you data, you just need to trust the root hash.

**Efficient verification.** A proof is O(log n) hashes. A Bitcoin SPV wallet can verify a transaction in a block of 4,000 transactions with just 12 hashes (384 bytes). This is what makes lightweight clients possible - you don't need to download the entire blockchain to verify your own transactions.

**Efficient synchronization.** When two nodes want to compare their datasets, they start by comparing root hashes. If the roots differ, they recurse into children, only descending into subtrees where hashes differ. This tree-based comparison skips over identical subtrees entirely. This is how anti-entropy protocols in distributed databases (like Dynamo or Cassandra) detect and repair inconsistencies. Instead of comparing every key-value pair, they compare Merkle tree nodes level by level.

## Extending the implementation

Our ~100-line implementation covers the core concepts, but production Merkle trees typically add a few more features:

**Append-only trees.** CT logs and blockchain state tries need to grow over time without recalculating everything. You can extend our `MerkleTree` with an `append` method that recalculates only the rightmost path - O(log n) work per append instead of O(n) for a full rebuild.

**Sparse Merkle trees.** Instead of indexing by position (0, 1, 2...), you can use the hash of a key as the index into a tree of depth 256 (for SHA-256). Most leaves are empty (a default hash). This gives you authenticated key-value lookups and is used in Ethereum's state trie. The [sparse-merkle-tree](https://crates.io/crates/sparse-merkle-tree) crate implements this variant.

**Multi-proofs.** If you need to prove multiple leaves at once, you can share common ancestors between proofs. The [rs_merkle](https://crates.io/crates/rs_merkle) crate supports this - its `MerkleProof` type can batch multiple leaf proofs together, reducing total proof size.

**Streaming construction.** Our implementation collects all leaves in memory before building. For very large datasets (billions of leaves), you'd want to build the tree incrementally, flushing completed subtrees to disk. This is how large-scale CT logs operate.

## The complete code

Here's everything in one place. Copy it into a fresh `cargo new merkle-tree` project, add `sha2 = "0.11"` to `Cargo.toml`, and it compiles and runs:

```rust
use sha2::{Digest, Sha256};

type Hash = [u8; 32];

fn hash_leaf(data: &[u8]) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update([0x00]);
    hasher.update(data);
    hasher.finalize().into()
}

fn hash_pair(left: &Hash, right: &Hash) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update([0x01]);
    hasher.update(left);
    hasher.update(right);
    hasher.finalize().into()
}

#[derive(Debug, Clone)]
pub enum Side { Left, Right }

#[derive(Debug, Clone)]
pub struct ProofEntry {
    pub hash: Hash,
    pub side: Side,
}

pub struct MerkleTree {
    layers: Vec<Vec<Hash>>,
}

impl MerkleTree {
    pub fn build(data: &[&[u8]]) -> Self {
        assert!(!data.is_empty());
        let leaves: Vec<Hash> = data.iter().map(|d| hash_leaf(d)).collect();
        let mut layers = vec![leaves];

        while layers.last().unwrap().len() > 1 {
            let prev = layers.last().unwrap();
            let next = prev
                .chunks(2)
                .map(|pair| {
                    if pair.len() == 2 {
                        hash_pair(&pair[0], &pair[1])
                    } else {
                        hash_pair(&pair[0], &pair[0])
                    }
                })
                .collect();
            layers.push(next);
        }
        MerkleTree { layers }
    }

    pub fn root(&self) -> &Hash { &self.layers.last().unwrap()[0] }

    pub fn proof(&self, index: usize) -> Vec<ProofEntry> {
        assert!(index < self.layers[0].len());
        let mut entries = Vec::new();
        let mut idx = index;

        for layer in &self.layers[..self.layers.len() - 1] {
            let (sibling_idx, side) = if idx % 2 == 0 {
                (idx + 1, Side::Right)
            } else {
                (idx - 1, Side::Left)
            };
            let sibling_hash = if sibling_idx < layer.len() {
                layer[sibling_idx]
            } else {
                layer[idx]
            };
            entries.push(ProofEntry { hash: sibling_hash, side });
            idx /= 2;
        }
        entries
    }
}

pub fn verify(root: &Hash, data: &[u8], proof: &[ProofEntry]) -> bool {
    let mut current = hash_leaf(data);
    for entry in proof {
        current = match entry.side {
            Side::Left => hash_pair(&entry.hash, &current),
            Side::Right => hash_pair(&current, &entry.hash),
        };
    }
    current == *root
}

fn main() {
    let data: Vec<&[u8]> = vec![
        b"tx_alice_sends_10",
        b"tx_bob_sends_5",
        b"tx_carol_sends_20",
        b"tx_dave_sends_1",
        b"tx_eve_sends_50",
    ];

    let tree = MerkleTree::build(&data);
    let root = tree.root();

    // Prove tx_bob_sends_5 (index 1)
    let proof = tree.proof(1);

    assert!(verify(root, b"tx_bob_sends_5", &proof));
    assert!(!verify(root, b"tx_bob_sends_999", &proof));

    println!("root: {:x?}", &root[..8]);
    println!("proof length: {} entries", proof.len());
    println!("verification: ok");
}
```

90 lines of actual logic. No unsafe. No allocator tricks. No external dependencies beyond a hash function. That's a complete, secure Merkle tree with proof generation and verification.

The same fundamental structure - hash leaves, combine pairs, prove with siblings - scales from a weekend project to systems that secure billions of dollars in Bitcoin transactions and billions of TLS certificates in CT logs. The difference is operational complexity, not algorithmic complexity. The algorithm stays the same.
