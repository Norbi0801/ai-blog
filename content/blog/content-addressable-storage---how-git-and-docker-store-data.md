+++
title = "Content-addressable storage: how Git and Docker store data"
date = 2025-04-09
description = "Store data by its hash, not its name. Deduplication, immutability, and why Git, Docker, and IPFS all converged on the same idea. Build a simple CAS in Rust."

[taxonomies]
tags = ["git", "docker", "distributed-systems", "rust"]
+++

When you run `git commit`, you are not saving a file. You are hashing a blob and writing it into a key-value store where the key is the hash. When Docker pulls an image, it does not really ask for "nginx:1.25". It resolves that tag to a SHA-256 digest and asks for that digest. When IPFS fetches data, there is no filename at all, only a CID - a Content Identifier. These three systems converged independently on the same storage model: you address data by what it is, not by where it lives. This idea has a name, content-addressable storage, and once you see it, you start noticing it everywhere.

<!-- more -->

## What changes when the address is the content

In a normal filesystem, `/var/log/app.log` is a name that points to some bytes. The bytes can change. The name stays the same. Two users editing the same file see the same address even though the content drifts apart at every save.

In content-addressable storage, there is no name. There is only a hash. `store(bytes)` returns a hash. `get(hash)` returns those exact bytes or nothing. If you want a different version, you get a different hash. The address and the content are welded together.

Three properties fall out of this for free.

**Deduplication.** If two users upload the same 4 MB PDF, they produce the same hash and write to the same key. The second upload is a no-op. You never store duplicates because you literally cannot: the address is already taken by the identical bytes.

**Immutability.** You cannot edit an object. Changing one byte gives a completely different hash, which is a different object at a different address. The old bytes are still there under the old hash. Mutation becomes "write a new object and update a pointer somewhere else."

**Verifiability.** Given the hash and the bytes, you can check in O(n) that the bytes are intact. You do not need to trust the source of the bytes, only the hash. This is why CAS is the backbone of pretty much every decentralized storage system.

The cost is that you lose locality and meaningful names. A CAS is a flat namespace of 64-character hex strings. You need a separate layer on top if you want anything human-readable.

## Git: the canonical example

Every file, every directory, every commit in your repo is a Git object stored by hash. Run this in any repo you have lying around:

```bash
$ echo "hello" | git hash-object --stdin
ce013625030ba8dba906f756967f9e9ca394464a
```

That hash is not random. Git computed it like this:

1. Build a header: `"blob 6\0"` - the object type, a space, the byte length in ASCII, a null byte.
2. Concatenate header + content: `"blob 6\0hello\n"`.
3. Take the SHA-1 of that buffer.

You can verify it manually:

```bash
$ printf 'blob 6\0hello\n' | sha1sum
ce013625030ba8dba906f756967f9e9ca394464a  -
```

Now commit something and peek inside `.git/objects/`:

```bash
$ ls .git/objects/ce/
013625030ba8dba906f756967f9e9ca394464a
```

The first two hex characters become the directory name, the remaining 38 become the filename. This is not a performance hack, it is a workaround for filesystems that degrade when directories contain hundreds of thousands of entries. Git spreads objects across 256 buckets by first byte of the hash.

Inside the file is the content, zlib-compressed. The hash is of the uncompressed header+content, not the compressed bytes. Compression is a storage implementation detail; the identity is the content.

Git has four object types:

- **blob** - file contents, no metadata, no filename
- **tree** - a directory listing: (mode, name, hash) tuples pointing at blobs and other trees
- **commit** - parent hashes, tree hash, author, message
- **tag** - annotated tag pointing at a commit

Notice what is missing. A blob does not know its filename. "README.md" is a string inside the parent tree, not inside the blob itself. If you rename the file, the blob hash does not change. Git detects renames by matching blob contents across commits, which is why `git log --follow` can track history across moves without needing any explicit rename metadata.

Two files with identical content share a blob. Two identical subdirectories share a tree. A commit that only changes one file reuses every other blob and tree in the repo. This is why `.git/` often stays smaller than your working copy even after years of history: deduplication is automatic because identity is content.

Git has been transitioning from SHA-1 to SHA-256 since [2020](https://git-scm.com/docs/hash-function-transition). SHA-1 is broken against collision attacks but not (yet) against second-preimage attacks, and Git's specific use of SHA-1 was further hardened after the SHAttered collision was published. The format of objects does not change, only the hash function.

## Docker: layers keyed by digest

Docker images are a stack of tarballs. Each tarball is a layer, and each layer is addressed by the SHA-256 of its compressed contents. Pull an image and look at what is actually downloaded:

```bash
$ docker pull nginx:1.25
1.25: Pulling from library/nginx
9b18e9b68314: Pull complete
3d47f874b6e9: Pull complete
...
Digest: sha256:6dc2dd9e5e6b4b6c2c7e7b42b5c4e9a...
```

Those short hex strings are the first 12 characters of SHA-256 digests. The manifest Docker fetched from the registry looks roughly like this:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "digest": "sha256:6dc2dd9e5e6b...",
    "size": 7680
  },
  "layers": [
    {"digest": "sha256:9b18e9b68314...", "size": 31417842},
    {"digest": "sha256:3d47f874b6e9...", "size": 2156},
    ...
  ]
}
```

The manifest itself has a digest. The config blob has a digest. Each layer has a digest. The tag `nginx:1.25` resolves to the manifest digest, which pins every downstream digest transitively. If a registry serves you bytes that do not hash to the digest you asked for, your Docker daemon rejects them.

When you pull `node:20` after already having `python:3.12`, any layer they share (usually the Debian base layer) does not get downloaded again. The digest is already in `/var/lib/docker/overlay2/`. Same bytes, same digest, one copy on disk. This is why pulling a dozen images on top of `ubuntu:22.04` costs far less than twelve full images.

The OCI spec formalized this as the [image-spec](https://github.com/opencontainers/image-spec). The core concept is unchanged from Git: a manifest pointing at content-addressed blobs, and a tag layer on top for human names. Docker tags are exactly as mutable as Git branches. The underlying digests are not.

## IPFS: content addressing across a public network

IPFS takes the idea further. There is no central registry, no `docker pull`, no `git clone`. You have a CID and you ask the network who has the bytes that hash to it.

A CID is not just a hash. It is a self-describing identifier:

```
bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
```

Base32 on the outside, but inside it is:

- A version byte (v0 or v1)
- A codec (dag-pb, raw, dag-cbor, etc.) describing how to interpret the bytes
- A multihash: hash function ID + hash length + hash digest

The [multihash](https://github.com/multiformats/multihash) prefix means a CID is not locked to SHA-256. You can use BLAKE3, SHA-512, or something that does not exist yet. The codec prefix means the system knows whether to parse the bytes as JSON, CBOR, Protobuf, or treat them as an opaque blob. Git and Docker have an implicit schema ("this is always SHA-256 of zlib-compressed content"); IPFS encodes the schema in the address.

Fetching a CID on IPFS goes: ask the DHT who has this CID, connect to one of those peers, stream the bytes, hash them while streaming, abort if the hash does not match. The network does not need to be trusted. The CID is the proof.

## Building a CAS in Rust

Let's build the smallest thing that is useful. A CAS with two operations: `store(data) -> hash` and `get(hash) -> data`. Backed by a filesystem, keyed by SHA-256.

```toml
# Cargo.toml
[dependencies]
sha2 = "0.11"
hex = "0.4"
thiserror = "2"
```

```rust
use sha2::{Sha256, Digest};
use std::fs;
use std::io;
use std::path::{Path, PathBuf};

pub struct Cas {
    root: PathBuf,
}

#[derive(Debug, thiserror::Error)]
pub enum CasError {
    #[error("io: {0}")]
    Io(#[from] io::Error),
    #[error("hex: {0}")]
    Hex(#[from] hex::FromHexError),
    #[error("object not found: {0}")]
    NotFound(String),
    #[error("integrity check failed for {expected}")]
    Integrity { expected: String },
}

impl Cas {
    pub fn open(root: impl AsRef<Path>) -> Result<Self, CasError> {
        let root = root.as_ref().to_path_buf();
        fs::create_dir_all(&root)?;
        Ok(Cas { root })
    }

    fn path_for(&self, hash_hex: &str) -> PathBuf {
        // Git-style fan-out: first 2 chars as dir, rest as filename.
        self.root.join(&hash_hex[..2]).join(&hash_hex[2..])
    }

    pub fn store(&self, data: &[u8]) -> Result<String, CasError> {
        let mut hasher = Sha256::new();
        hasher.update(data);
        let hash_hex = hex::encode(hasher.finalize());

        let path = self.path_for(&hash_hex);
        if path.exists() {
            // Already have it. Dedup is free.
            return Ok(hash_hex);
        }

        fs::create_dir_all(path.parent().unwrap())?;

        // Write to a temp file in the same dir, then rename.
        // rename(2) is atomic on POSIX within one filesystem,
        // so concurrent writers never see a half-written object.
        let tmp = path.with_extension("tmp");
        fs::write(&tmp, data)?;
        fs::rename(&tmp, &path)?;

        Ok(hash_hex)
    }

    pub fn get(&self, hash_hex: &str) -> Result<Vec<u8>, CasError> {
        let path = self.path_for(hash_hex);
        let bytes = fs::read(&path).map_err(|e| {
            if e.kind() == io::ErrorKind::NotFound {
                CasError::NotFound(hash_hex.to_string())
            } else {
                e.into()
            }
        })?;

        // Verify on read. Disk bit rot, tampering, bugs elsewhere:
        // the hash is the source of truth, not the filename.
        let mut hasher = Sha256::new();
        hasher.update(&bytes);
        let actual = hex::encode(hasher.finalize());
        if actual != hash_hex {
            return Err(CasError::Integrity {
                expected: hash_hex.to_string(),
            });
        }

        Ok(bytes)
    }

    pub fn has(&self, hash_hex: &str) -> bool {
        self.path_for(hash_hex).exists()
    }
}

fn main() -> Result<(), CasError> {
    let cas = Cas::open("/tmp/cas-demo")?;

    let h1 = cas.store(b"hello world")?;
    let h2 = cas.store(b"hello world")?;
    let h3 = cas.store(b"goodbye world")?;

    assert_eq!(h1, h2, "same content -> same hash");
    assert_ne!(h1, h3, "different content -> different hash");

    let bytes = cas.get(&h1)?;
    assert_eq!(bytes, b"hello world");

    println!("stored: {}", h1);
    Ok(())
}
```

A few things worth pointing out.

The rename trick in `store` is important. If you write directly to the final path and crash halfway, you leave a corrupt object at a valid hash path. Other readers will get wrong bytes, and `get` will fail the integrity check (good) but the store is now broken. Writing to `*.tmp` and renaming means either the rename succeeds and the object is fully there, or it fails and the temp file is garbage you can clean up later. `rename(2)` on the same filesystem is atomic on POSIX, which is exactly what you need.

The `if path.exists()` check before writing is the deduplication. Two concurrent calls to `store` with the same bytes both compute the same hash, both see the file exists (or race and both rename, the last rename winning is still correct because the bytes are identical), and both return the same hash. No locking needed for correctness, because the hash collision would require a SHA-256 break.

The integrity check in `get` is what makes CAS an honest store. If you skip it, you are trusting the filesystem. Bit rot on old disks is real, and silent corruption of a blob would propagate. Hashing on read costs you one SHA-256 per GB (a few hundred milliseconds on modern hardware) and guarantees the bytes you return match the name you asked for.

What is missing from this toy: no compression (Git uses zlib), no packing (Git packs thousands of objects into packfiles and uses delta compression between them), no garbage collection (you need a mark-and-sweep to find unreferenced blobs), no streaming API for large files (hashing an 8 GB video in memory is a bad idea). All of these are layered on top of the same `store(data) -> hash` primitive.

## Merkle trees fall out automatically

Once you have CAS, you get Merkle trees almost for free. A Git tree object points to blobs by hash and to subtrees by hash. A commit points to a tree by hash and to its parent commits by hash. Follow any hash chain upward and you are walking a Merkle DAG.

The root commit hash is a cryptographic summary of the entire repository. Every blob, every tree, every ancestor commit contributes to it. Change one byte in one file and every hash from that blob up to the HEAD commit is different. This is what makes `git fsck` possible and why Git's history is tamper-evident: you cannot rewrite a commit without all descendants noticing.

IPFS DAG-PB nodes do the same thing. Docker image manifests do a shallow version of the same thing (manifest digest covers config digest covers layer digests).

If you want to see this built from scratch with inclusion proofs and the RFC 6962 domain-separation scheme, I wrote a [separate post on implementing a Merkle tree in Rust](/blog/implementing-a-merkle-tree-in-rust/). The TL;DR: once your leaves are content-addressed, hashing pairs of hashes up to a root gives you O(log n) proofs that any given leaf belongs to a given root.

## Why this pattern owns distributed systems

Once you stop naming data and start hashing it, a lot of hard distributed-systems problems become easier.

**Caching is exact.** If your CDN already has the bytes at this digest, skip the fetch. No "is the cached copy stale?" question, because the digest is the freshness.

**Replication is verifiable.** Send the digests you have; peer sends back what they do not have. The digests themselves prove the bytes on the wire are what you asked for.

**Conflict resolution is tractable.** Two writers never overwrite each other because they never write to the same address unless their bytes match exactly. Merging is about pointer updates, not byte-level merging.

**Supply chain security.** `docker pull nginx@sha256:6dc2dd...` pins the image cryptographically. Even if the tag is hijacked, the digest is not. This is also why Sigstore and the SLSA framework lean heavily on CAS: "this artifact is attested by this signature" only works if the artifact has a stable identity.

The tradeoffs are real. CAS is write-heavy (you cannot edit in place). Listing or searching requires a secondary index because the primary keyspace is meaningless. Large objects hurt because hashing is O(n). And the hash function itself is a trust root - the day SHA-256 breaks, every CAS built on it is downgraded from "cryptographically verifiable" to "probably fine for now."

But the gains are hard to give up once you have them. Immutable storage eliminates an entire class of race conditions. Deduplication shrinks repos, registries, and backup systems by one or two orders of magnitude. Verification turns distributed fetch into an operation you can do without trusting the sender.

Next time you `git commit` or `docker pull`, you are not moving files around. You are writing into, and reading from, a hash-keyed database that happens to be everywhere.
