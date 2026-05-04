+++
title = "Building a Git implementation in Rust: objects and refs"
date = 2025-06-27
description = "Implement enough of Git in ~300 lines of Rust to init a repo, hash objects, write trees, and create commits that real `git log` can read. Demystifies what `.git/` actually contains."

[taxonomies]
tags = ["rust", "git", "low-level", "tools"]
+++

Git is a content-addressable filesystem with a UI bolted on top. Once you accept that, the source code stops being scary and `.git/` stops being magic. This post walks through implementing enough of Git in Rust to run `init`, `hash-object`, `cat-file`, `write-tree`, and `commit-tree` against a directory and have plain `git log` happily read the result. About 300 lines, no shortcuts on the on-disk format.

If you have not already, the [content-addressable storage post](/blog/content-addressable-storage---how-git-and-docker-store-data/) covers the blob framing, the SHA-1 prefix, and why `.git/objects/` fans out by the first two hex characters of a hash. I am going to assume that part and pick up at the byte layout of trees, commits, and refs.

<!-- more -->

## What `.git/` actually contains

Run `git init` in an empty directory and look at what gets created:

```
.git/
  HEAD                  # text: "ref: refs/heads/main"
  config                # ini-style config
  description           # human description, only used by gitweb
  hooks/                # shell scripts, ignored by us
  info/exclude          # extra .gitignore patterns
  objects/              # the content-addressed store
    info/
    pack/
  refs/
    heads/              # branch tips: refs/heads/main -> hash
    tags/               # tag names -> hash
```

Three things do real work: `objects/`, `refs/`, and `HEAD`. Everything else is auxiliary. To rebuild Git's read/write loop we need to write into `objects/`, write a hash into `refs/heads/<branch>`, and read `HEAD` to know which branch to update.

## Project layout

```toml
[package]
name = "rgit"
version = "0.1.0"
edition = "2024"

[dependencies]
sha1 = "0.10"
flate2 = "1.0"
hex = "0.4"
anyhow = "1"
```

`sha1` because Git still hashes objects with SHA-1. The [SHA-256 transition](https://git-scm.com/docs/hash-function-transition) has been in flight since 2020 but the on-disk format your installed Git writes today is still SHA-1. `flate2` for zlib (Git compresses every loose object). `anyhow` to keep the error story short.

## Object framing

Every loose object on disk is `<type> <size>\0<content>`, then zlib-deflated, written to `.git/objects/<first 2 hex>/<remaining 38>`. The SHA-1 is computed over the *uncompressed* framed bytes. That is what makes the hash deterministic across machines: compression settings can vary, content cannot.

```rust
use anyhow::{bail, Context, Result};
use flate2::{read::ZlibDecoder, write::ZlibEncoder, Compression};
use sha1::{Digest, Sha1};
use std::fs;
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Kind { Blob, Tree, Commit, Tag }

impl Kind {
    fn as_str(self) -> &'static str {
        match self {
            Kind::Blob => "blob", Kind::Tree => "tree",
            Kind::Commit => "commit", Kind::Tag => "tag",
        }
    }
    fn parse(s: &[u8]) -> Result<Kind> {
        Ok(match s {
            b"blob" => Kind::Blob, b"tree" => Kind::Tree,
            b"commit" => Kind::Commit, b"tag" => Kind::Tag,
            other => bail!("unknown kind: {}", String::from_utf8_lossy(other)),
        })
    }
}

fn objects_dir(repo: &Path) -> PathBuf { repo.join(".git/objects") }

pub fn write_object(repo: &Path, kind: Kind, body: &[u8]) -> Result<String> {
    let header = format!("{} {}\0", kind.as_str(), body.len());
    let mut framed = Vec::with_capacity(header.len() + body.len());
    framed.extend_from_slice(header.as_bytes());
    framed.extend_from_slice(body);

    let mut hasher = Sha1::new();
    hasher.update(&framed);
    let oid = hex::encode(hasher.finalize());

    let path = objects_dir(repo).join(&oid[..2]).join(&oid[2..]);
    if !path.exists() {
        fs::create_dir_all(path.parent().unwrap())?;
        let mut z = ZlibEncoder::new(Vec::new(), Compression::default());
        z.write_all(&framed)?;
        let compressed = z.finish()?;
        let tmp = path.with_extension("tmp");
        fs::write(&tmp, &compressed)?;
        fs::rename(&tmp, &path)?;
    }
    Ok(oid)
}

pub fn read_object(repo: &Path, oid: &str) -> Result<(Kind, Vec<u8>)> {
    let path = objects_dir(repo).join(&oid[..2]).join(&oid[2..]);
    let bytes = fs::read(&path).with_context(|| format!("object {oid} not found"))?;
    let mut buf = Vec::new();
    ZlibDecoder::new(&bytes[..]).read_to_end(&mut buf)?;

    let nul = buf.iter().position(|&b| b == 0).context("malformed header")?;
    let space = buf[..nul].iter().position(|&b| b == b' ').context("missing size")?;
    let kind = Kind::parse(&buf[..space])?;
    let size: usize = std::str::from_utf8(&buf[space + 1..nul])?.parse()?;
    let body = buf[nul + 1..].to_vec();
    if body.len() != size {
        bail!("size mismatch: header says {size}, got {}", body.len());
    }
    Ok((kind, body))
}
```

Two non-obvious bits. The atomic rename (write `*.tmp`, then `rename(2)`) means a crash mid-write does not leave a half-baked file at the canonical hash path - other readers will never see a corrupt object at a "valid" address. And the `if !path.exists()` short-circuit gives you deduplication for free: the same content always hashes to the same path, so the second write is a no-op.

The position-of-first-null trick for parsing works because the header `<type> <size>\0` has no embedded nulls. The body of a tree object absolutely contains nulls (they separate names from the raw SHA-1), but we only need the first null, which is always the header terminator.

## hash-object and cat-file

Both are one-liners on top of the framing primitive.

```rust
pub fn cmd_hash_object(repo: &Path, file: &Path, write: bool) -> Result<String> {
    let body = fs::read(file)?;
    if write {
        write_object(repo, Kind::Blob, &body)
    } else {
        let header = format!("blob {}\0", body.len());
        let mut h = Sha1::new();
        h.update(header.as_bytes());
        h.update(&body);
        Ok(hex::encode(h.finalize()))
    }
}

pub fn cmd_cat_file(repo: &Path, oid: &str, mode: char) -> Result<()> {
    let (kind, body) = read_object(repo, oid)?;
    match mode {
        't' => println!("{}", kind.as_str()),
        's' => println!("{}", body.len()),
        'p' => std::io::stdout().write_all(&body)?,
        _ => bail!("unknown mode -{mode}"),
    }
    Ok(())
}
```

Run `rgit hash-object README.md` and `git hash-object README.md` against the same file and you get the same 40-character hash. The framing is byte-identical.

## Tree objects

Trees are where the format stops being friendly. A tree object is a list of entries packed back-to-back with no length prefix between them:

```
<mode> <name>\0<20 raw bytes of SHA-1>
```

The mode is ASCII octal *without a leading zero*: `100644` for a regular file, `100755` for executable, `40000` for a subtree, `120000` for a symlink, `160000` for a submodule (gitlink). The name is just the filename, never a path. The hash is the *raw* 20 bytes, not hex - that catches a lot of people on the first attempt.

Entries must be sorted, but with one subtlety: trees sort as if their name had a trailing `/`. So `foo` (a file) comes before `foo.txt`, but `foo/` (a tree named `foo`) sorts after `foo.txt`. Get this wrong and your tree hashes will not match Git's, and `git fsck` will reject the tree as malformed. The reason is buried in the [tree-walk code](https://github.com/git/git/blob/master/tree-walk.c): Git wants the comparison order to be stable when you walk a tree as if every directory were expanded inline.

```rust
pub struct TreeEntry {
    pub mode: &'static str,   // "100644", "100755", "40000", ...
    pub name: String,
    pub oid: [u8; 20],        // raw, not hex
}

pub fn write_tree(repo: &Path, mut entries: Vec<TreeEntry>) -> Result<String> {
    entries.sort_by(|a, b| sort_key(a).cmp(&sort_key(b)));
    let mut body = Vec::new();
    for e in &entries {
        body.extend_from_slice(e.mode.as_bytes());
        body.push(b' ');
        body.extend_from_slice(e.name.as_bytes());
        body.push(0);
        body.extend_from_slice(&e.oid);
    }
    write_object(repo, Kind::Tree, &body)
}

fn sort_key(e: &TreeEntry) -> Vec<u8> {
    let mut k = e.name.as_bytes().to_vec();
    if e.mode == "40000" { k.push(b'/'); }
    k
}
```

`write-tree` walks a directory and recurses into subdirectories, hashing files as blobs along the way:

```rust
pub fn cmd_write_tree(repo: &Path, dir: &Path) -> Result<String> {
    let mut entries = Vec::new();
    for de in fs::read_dir(dir)? {
        let de = de?;
        let name = de.file_name().to_string_lossy().into_owned();
        if name == ".git" { continue; }
        let ft = de.file_type()?;
        let path = de.path();

        let (mode, oid_hex) = if ft.is_dir() {
            ("40000", cmd_write_tree(repo, &path)?)
        } else if ft.is_file() {
            let mode = if is_executable(&de)? { "100755" } else { "100644" };
            let oid = write_object(repo, Kind::Blob, &fs::read(&path)?)?;
            (mode, oid)
        } else {
            continue; // skip symlinks/sockets/fifos for the toy
        };

        let mut raw = [0u8; 20];
        hex::decode_to_slice(&oid_hex, &mut raw)?;
        entries.push(TreeEntry { mode, name, oid: raw });
    }
    write_tree(repo, entries)
}

#[cfg(unix)]
fn is_executable(de: &fs::DirEntry) -> Result<bool> {
    use std::os::unix::fs::PermissionsExt;
    Ok(de.metadata()?.permissions().mode() & 0o111 != 0)
}
#[cfg(not(unix))]
fn is_executable(_: &fs::DirEntry) -> Result<bool> { Ok(false) }
```

Two real Git details show up here. Modes are octal-as-ASCII, variable length (`40000` is 5 bytes, `100644` is 6). And Git only stores `100644` vs `100755` for files - it does not record the rest of your Unix permission bits. Owner, group, mtime, ctime, the full mode word: none of that is in a tree. That stuff lives in the index (`.git/index`), which is the staging area's representation, not anything that becomes part of history.

## Commit objects

A commit is text, UTF-8. The format is:

```
tree <hex sha1>
parent <hex sha1>     (zero or more, in order)
author Name <email> <unix-secs> <±HHMM>
committer Name <email> <unix-secs> <±HHMM>

<message>
```

Header order matters: `tree` first, then any `parent` lines, then `author`, then `committer`, then a blank line, then the message. Extra headers can appear between `committer` and the blank line - that is where `gpgsig`, `mergetag`, and friends live. The commit object format itself has not changed in 18 years; everything fancier is just more headers.

```rust
pub struct Sig {
    pub name: String,
    pub email: String,
    pub when: i64,
    pub tz: String, // "+0000", "-0530"
}

pub fn cmd_commit_tree(
    repo: &Path,
    tree: &str,
    parents: &[String],
    author: &Sig,
    committer: &Sig,
    message: &str,
) -> Result<String> {
    let mut body = String::new();
    body.push_str(&format!("tree {tree}\n"));
    for p in parents { body.push_str(&format!("parent {p}\n")); }
    body.push_str(&format!(
        "author {} <{}> {} {}\n",
        author.name, author.email, author.when, author.tz
    ));
    body.push_str(&format!(
        "committer {} <{}> {} {}\n",
        committer.name, committer.email, committer.when, committer.tz
    ));
    body.push('\n');
    body.push_str(message);
    if !message.ends_with('\n') { body.push('\n'); }
    write_object(repo, Kind::Commit, body.as_bytes())
}
```

That is the entire commit format. The reason `git log` feels rich (signed commits, merge metadata, GPG signatures) is that everything between `committer` and the blank line is "more headers, parsed loosely," and the body after the blank line is whatever you put there.

## refs and HEAD

A ref is a file containing a 40-character hex hash plus a newline. That is the entire spec.

```
$ cat .git/refs/heads/main
8b1378917efb70878...
```

`HEAD` is one of two things. Either a 40-character hash (you are in detached HEAD state) or `ref: refs/heads/<name>` (a symbolic ref pointing at a branch). When you `git checkout` a branch, Git rewrites `HEAD` to the symbolic form. When you `git checkout <commit-hash>`, it writes the hash directly.

```rust
pub fn write_ref(repo: &Path, ref_path: &str, oid: &str) -> Result<()> {
    let p = repo.join(".git").join(ref_path);
    fs::create_dir_all(p.parent().unwrap())?;
    fs::write(&p, format!("{oid}\n"))?;
    Ok(())
}

pub fn read_head(repo: &Path) -> Result<Option<String>> {
    let head = fs::read_to_string(repo.join(".git/HEAD"))?;
    let head = head.trim();
    if let Some(target) = head.strip_prefix("ref: ") {
        let p = repo.join(".git").join(target);
        if p.exists() {
            Ok(Some(fs::read_to_string(p)?.trim().to_string()))
        } else {
            Ok(None) // unborn branch (just-initialized repo)
        }
    } else {
        Ok(Some(head.to_string()))
    }
}

pub fn update_branch(repo: &Path, oid: &str) -> Result<()> {
    let head = fs::read_to_string(repo.join(".git/HEAD"))?;
    let target = head.trim().strip_prefix("ref: ")
        .context("detached HEAD; this toy does not handle that")?
        .to_string();
    write_ref(repo, &target, oid)
}
```

When refs accumulate, Git eventually packs them into `.git/packed-refs`, a single file with one `<hash> <refname>` line per ref. A complete implementation reads loose refs first and falls back to scanning packed-refs. Our toy never writes packed-refs, but you can spot one in the wild on any long-lived repo.

There is also `.git/logs/` (the reflog), which records every ref update for ~90 days. That is what makes `git reflog` and the recovery-from-`reset --hard` story work. We are skipping it - it is a separate text format, one append per update.

## init

```rust
pub fn cmd_init(repo: &Path) -> Result<()> {
    let g = repo.join(".git");
    fs::create_dir_all(g.join("objects/pack"))?;
    fs::create_dir_all(g.join("objects/info"))?;
    fs::create_dir_all(g.join("refs/heads"))?;
    fs::create_dir_all(g.join("refs/tags"))?;
    fs::write(g.join("HEAD"), "ref: refs/heads/main\n")?;
    fs::write(
        g.join("config"),
        "[core]\n\trepositoryformatversion = 0\n\tfilemode = true\n\tbare = false\n",
    )?;
    Ok(())
}
```

That is a valid Git repo. `git status` will run on it. `git log` will say there is nothing yet. There is no `description` file, no `hooks/`, no `info/exclude` - none of those matter to the protocol; they are convenience defaults.

## End-to-end

Wire the commands into `main` and you can drive the whole pipeline:

```bash
$ rgit init demo && cd demo
$ echo hello > a.txt
$ TREE=$(rgit write-tree .)
$ NOW=$(date +%s)
$ COMMIT=$(rgit commit-tree $TREE \
    --author "Me <me@x> $NOW +0000" \
    -m "first")
$ rgit update-ref $COMMIT
$ git log --oneline
<hash> first
$ git cat-file -p $COMMIT
tree <tree-hash>
author Me <me@x> 1746057600 +0000
committer Me <me@x> 1746057600 +0000

first
```

The last two commands are *real* `git`, reading objects we wrote. Since we are writing the exact byte format Git expects, every Git tool on the planet will accept these objects. That is the payoff of doing the format work properly: interoperability is automatic.

## Pack files: the part we did not build

Loose objects are great for understanding and terrible for storage. A repo with 100k commits is 100k tiny zlib files, which destroys filesystem performance and ignores the fact that adjacent commits share almost all their bytes. Git fixes this with packfiles.

A packfile is a single file at `.git/objects/pack/pack-<hash>.pack` containing many objects concatenated together with a small variable-length header per object, plus an index file `pack-<hash>.idx` that maps OID to byte offset in the pack. Two extra object types appear only inside packs: `OBJ_OFS_DELTA` and `OBJ_REF_DELTA`. Both encode an object as a delta (insert/copy ops, not line diffs) against another object. A new commit's tree is usually 99% identical to its parent's tree, and packing turns "two trees" into "one tree + a few hundred bytes of delta." On large repos this is the difference between a 50 GB and a 500 MB `.git/`.

Building a packer well is its own project: a similarity heuristic to pick base objects, a window size for delta search, the `.idx` v2 fan-out table, and a CRC for each object. The wire protocol used by `git fetch` literally streams packfiles, which is why both ends have to agree on the format byte-for-byte. The Git documentation has [a precise spec](https://git-scm.com/docs/gitformat-pack) and [gitoxide](https://github.com/Byron/gitoxide) is a full Rust reimplementation with production-quality packfile code worth reading if you want to take this further.

## Why this exercise is worth doing

Once you have written `write-tree` and `commit-tree` by hand, Git's command surface stops feeling magical. `git add` is "hash blobs into objects/, update the index." `git commit` is "build a tree from the index, write a commit object pointing at it, update the branch ref." `git checkout` is "read a tree, walk it, write the files out." Branches are 41-byte text files. Tags are either the same or a tiny annotated `tag` object pointing at the commit. Every "interesting" Git feature - rebase, cherry-pick, bisect, merge - is composition over this primitive set.

Content addressing is what holds it together. Because the identity of every object is a hash of its bytes, you cannot have two different things at the same address, so deduplication is automatic and verification is free. Refs are mutable but cheap to update because all you are doing is overwriting a 41-byte file. The hard parts of Git - merge resolution, rebase, the index, the reflog - all compose on top of this layer without changing the layer itself.

The full source for this post fits in about 300 lines and round-trips cleanly with real Git. If you want a next step, write a packfile *reader* (the index format is the simpler half - it is just a sorted array of OIDs with a 256-entry fan-out table). Once you can read packs, you can clone over HTTP without `git` installed and read the result with the code above. From there, "implementing Git in Rust" is mostly a matter of patience.
