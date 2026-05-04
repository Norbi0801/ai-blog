+++
title = "Implementing a trie (prefix tree) in Rust"
date = 2025-10-31
description = "Building a trie from scratch in ~100 lines of Rust - autocomplete, prefix search, deletion, and why a HashMap is the wrong tool when keys share prefixes."

[taxonomies]
tags = ["rust", "algorithms", "data-structures", "memory"]
+++

Every time you type into a search box and watch suggestions appear, something is doing prefix matching. The autocomplete on your phone, the IP routing table in your kernel, the address bar in your browser, the spell check in your editor - all of these need to answer "what keys start with this prefix?" fast. A `HashMap<String, V>` cannot answer that question without scanning every key, and a sorted `BTreeMap` makes you do range gymnastics. A trie answers it natively.

The structure has a great name. Edward Fredkin coined "trie" in 1960, pronouncing it like "tree" because the middle syllable of "retrieval" sounds like "tree." Most people now say "try" to disambiguate it from regular trees. Either way, the idea is the same: store strings as paths, where each node holds one character and the path from the root to a node spells a prefix.

We're going to build one in about 100 lines, walk through insert, lookup, prefix collection, and the slightly awkward delete, then compare HashMap-backed children against a fixed-size array and look at where the memory actually goes.

<!-- more -->

## The structure

A trie is a tree where each edge is labeled with a character, and each node represents the prefix you get by following the path from the root. Some nodes are marked as "end of word" because a string ended there.

```
              (root)
             /  |  \
            c   d   t
            |   |   |
            a   o   o
           / \  |   |
          r   t g   ★ (to)
          |   |  ★ (dog)
          ★   ★ 
        (car)(cat)
```

The string `to` ends at depth 2. The string `cat` ends at depth 3. They share no prefix, so they branch at the root. `car` and `cat` share `ca`, so they share two nodes and only branch at the third. That sharing is the entire reason this data structure exists. With n strings of average length k, a HashMap stores n * k characters of key data. A trie stores at most that many, and usually far fewer when keys share prefixes.

The lookup cost is O(k) where k is the length of the query string. That's the same as hashing a string for HashMap lookup, which is also O(k) because hashing has to read every byte. The difference shows up when you ask "does any key start with this prefix?" - the trie still does O(k), and the HashMap has to scan all n keys.

## Where you find them in production

Tries quietly run a lot of infrastructure.

The Linux kernel uses a [LC-trie](https://github.com/torvalds/linux/blob/master/net/ipv4/fib_trie.c) for the IPv4 routing table. When a packet arrives, the kernel does longest-prefix match against all known routes, and a trie does that in time proportional to the address length, not the routing table size. The classic alternative was a hash on prefix, which doesn't help for variable-length matches.

Redis uses a compressed trie called [`rax`](https://github.com/redis/redis/blob/unstable/src/rax.c) to index stream entry IDs, cluster slot maps, and a few other things. Antirez wrote it specifically because Redis needed sorted lookup with prefix iteration and the existing skip list was overkill for short keys.

Web routers like [`matchit`](https://github.com/ibraheemdev/matchit) (used by axum) are radix tries. Routing `/users/:id/posts/:post_id` against `/users/42/posts/7` is a prefix walk with parameter capture.

The Aho-Corasick algorithm for multi-pattern string matching is a trie with extra failure links. The [`aho-corasick`](https://crates.io/crates/aho-corasick) crate that powers ripgrep's literal matching builds one of these.

Hunspell, the spell checker behind Firefox and LibreOffice, stores its dictionary as a trie variant. So does the suggestion engine in most IDEs.

## Setting up

No dependencies. Just `std`.

```rust
use std::collections::HashMap;
```

The first decision is how to store children at each node. Two options dominate.

**HashMap children.** Each node holds `HashMap<char, Node>`. Works for any Unicode codepoint. Memory grows with the number of children at each node. Lookup is one hash.

**Array children.** Each node holds `[Option<Box<Node>>; N]` where `N` is the alphabet size. Works for fixed alphabets like ASCII letters or DNA bases. Lookup is one array index, no hashing. Wastes memory on sparse nodes.

We'll start with HashMap because it's general, then look at the array variant later.

## The node and the trie

```rust
#[derive(Default)]
pub struct Trie {
    root: Node,
}

#[derive(Default)]
struct Node {
    children: HashMap<char, Node>,
    is_end: bool,
}

impl Trie {
    pub fn new() -> Self {
        Self::default()
    }
}
```

`is_end` is the bit that marks "a word ended here." Without it, you can't tell `car` from `card` - both would be valid paths to internal nodes, but only `card` would be in a trie that contained just `card`. The flag is what makes a node into a key.

The root is just a `Node` with no character of its own. It exists so every real character has a parent.

## Insert

To insert `cat`, walk from the root, following the character on each step and creating the child if it doesn't exist. Mark the final node as end.

```rust
impl Trie {
    pub fn insert(&mut self, word: &str) {
        let mut node = &mut self.root;
        for c in word.chars() {
            node = node.children.entry(c).or_default();
        }
        node.is_end = true;
    }
}
```

The `entry().or_default()` pattern is the cleanest way to do "get-or-insert" on a HashMap. `or_default()` calls `Node::default()`, which gives us an empty children map and `is_end = false`.

There's a subtle borrow checker thing happening here. `node = node.children.entry(c).or_default();` reborrows `node` as a mutable reference into its own child. The compiler is fine with this because the previous borrow of `node` is no longer used after the assignment. This pattern is sometimes called "tree walking with mutable references" and it's one of the cases where Rust's borrow checker gets out of your way once you stop fighting it.

`word.chars()` iterates Unicode scalar values. If you're indexing a fixed alphabet like ASCII letters, switch to `word.bytes()` and store `u8` keys. We'll come back to this.

## Search and prefix search

The two operations are nearly identical. Both walk the trie one character at a time. Search returns true only if the final node is marked end-of-word. Prefix search returns true if the final node exists at all.

```rust
impl Trie {
    pub fn contains(&self, word: &str) -> bool {
        self.find(word).is_some_and(|n| n.is_end)
    }

    pub fn starts_with(&self, prefix: &str) -> bool {
        self.find(prefix).is_some()
    }

    fn find(&self, s: &str) -> Option<&Node> {
        let mut node = &self.root;
        for c in s.chars() {
            node = node.children.get(&c)?;
        }
        Some(node)
    }
}
```

The `?` operator does exactly what you want: if any character is missing along the way, return `None` immediately. No nested matches.

Cost: O(k) where k is the length of the query. A HashMap does the same for `contains`, but it cannot do `starts_with` at all without iterating every key.

## Collecting all words with a prefix

This is the operation that justifies the whole data structure. Find the prefix node, then walk every descendant collecting end-of-word markers.

```rust
impl Trie {
    pub fn words_with_prefix(&self, prefix: &str) -> Vec<String> {
        let mut out = Vec::new();
        if let Some(node) = self.find(prefix) {
            collect(node, prefix.to_string(), &mut out);
        }
        out
    }
}

fn collect(node: &Node, prefix: String, out: &mut Vec<String>) {
    if node.is_end {
        out.push(prefix.clone());
    }
    for (c, child) in &node.children {
        let mut next = prefix.clone();
        next.push(*c);
        collect(child, next, out);
    }
}
```

Cost: O(p + m) where p is the prefix length and m is the total number of characters in the matching strings. You cannot do better than that on any data structure, because you have to produce m characters of output.

Compare to HashMap. To answer the same question, you would iterate every key, and for each key check whether it starts with the prefix. That's O(n * k) where n is the total number of keys and k is the average key length, regardless of how many matches you actually find. For an autocomplete feature that runs on every keystroke, this is the difference between a working feature and one that hangs the UI.

The order of results in our implementation is non-deterministic because `HashMap` iteration is non-deterministic. If you want sorted results, swap to `BTreeMap<char, Node>`. The cost is O(log alphabet_size) per step instead of O(1) average, which is irrelevant for any realistic alphabet.

## Delete

Delete is where tries get fiddly. The naive version is "find the node and unset `is_end`," which is correct for behavior but leaks nodes. If you insert `cat`, then delete it, the path `c -> a -> t` is still in memory.

The right behavior is to also prune any nodes that are no longer used by anything. A node is dead if it has no children and is not the end of any word. Because parents reference children, you have to prune from the bottom up.

```rust
impl Trie {
    pub fn remove(&mut self, word: &str) -> bool {
        let chars: Vec<char> = word.chars().collect();
        remove_inner(&mut self.root, &chars, 0)
    }
}

fn remove_inner(node: &mut Node, chars: &[char], depth: usize) -> bool {
    if depth == chars.len() {
        if !node.is_end {
            return false;
        }
        node.is_end = false;
        return true;
    }
    let c = chars[depth];
    let Some(child) = node.children.get_mut(&c) else {
        return false;
    };
    let removed = remove_inner(child, chars, depth + 1);
    if removed && child.children.is_empty() && !child.is_end {
        node.children.remove(&c);
    }
    removed
}
```

The recursion does two things on the way back up. First, it confirms the word was actually present (returning `false` lets the caller know nothing happened). Second, after each recursive call, it checks whether the child has become a useless leaf and detaches it from the parent's HashMap if so.

One thing to be careful about: if you delete `car` from a trie that also contains `card`, the path `c -> a -> r` still has a child `d`, so `r` survives even though `is_end = false`. The logic above handles that correctly because `child.children.is_empty()` is false at the `r` node.

## HashMap children vs array children

For an ASCII-only trie (say, lowercase a-z for an English dictionary), an array of 26 slots is faster and often smaller.

```rust
struct AsciiNode {
    children: [Option<Box<AsciiNode>>; 26],
    is_end: bool,
}
```

Each `Option<Box<AsciiNode>>` is 8 bytes on a 64-bit system because `Box` is non-null and Rust uses null-pointer optimization to fit `None` into the same 8 bytes. So one node is 26 * 8 + 1 byte for `is_end` + 7 bytes of padding = 216 bytes.

A `HashMap<char, Node>` starts at 48 bytes empty (three `usize` fields for capacity, length, and the table pointer). It allocates an internal table only on first insert, and that table is a power of two with a 7/8 load factor, so a node with one child needs roughly 48 bytes for the HashMap header plus a small allocation for the table plus the entry itself plus any rehashing slack. You're looking at on the order of 100-200 bytes per node depending on how many children it has, with worse cache behavior because the children live in a separate heap allocation.

The array version wins on three axes: no hash computation, all children in one cache line worth of pointers, predictable size. It loses on memory when most slots are empty - imagine a trie with one word `"hello"`, which has 5 nodes each wasting 25 of 26 slots. For dense tries with full alphabets, the array version is the right call. For sparse Unicode tries, HashMap is correct.

A middle ground used by [LC-tries](https://www.cs.princeton.edu/~rs/AlgsDS07/13RadixSort.pdf) and most production systems is a small inline array for the common case (say, 4 or 8 children) plus an overflow HashMap. Rust's [`smallvec`](https://crates.io/crates/smallvec) crate gives you that pattern for free.

## Memory: the compressed trie

The biggest win for real-world tries is path compression. Look at the trie for `["abandon", "abandoning"]`. The nodes for `a -> b -> a -> n -> d -> o -> n` are all one-child chains until you hit the end. Storing each character as its own node is pure overhead.

A radix trie (also called a Patricia trie) merges chains of single-child nodes into one node that holds a string. Now `["abandon", "abandoning"]` is two nodes: one labeled `abandon` (end-of-word), with one child labeled `ing` (end-of-word).

The savings are dramatic for English. The [`patricia_tree`](https://crates.io/crates/patricia_tree) crate reports 5-10x memory reduction over a plain trie for typical dictionary workloads. The cost is more complex insert and delete logic, because inserting `abandoned` into the compressed trie above means splitting the `abandon` node into `aband` plus children `on` and `oned`.

If you want to go even further, look at [`fst`](https://crates.io/crates/fst) by Andrew Gallant, the same person who wrote ripgrep. It builds a finite state transducer where common suffixes are also shared. A 1.4 GB Wikipedia title list compresses to about 30 MB and supports prefix queries directly on the compressed bytes. The tradeoff is that it's a static structure - you build it once, query it forever, no inserts.

## When a HashMap actually wins

A trie is the wrong choice when you only do exact lookup and never need prefix queries. HashMap will be faster and use less memory for that workload. The trie's overhead per character pays off when you can amortize it across prefix scans, longest-prefix matches, sorted iteration, or shared-prefix compression.

The other case where a HashMap beats a trie is short keys with extreme value diversity - say, hashing 64-bit integer IDs. There's no shared prefix structure to exploit, every key is roughly the same length, and the HashMap's single hash beats walking 8 nodes.

So the rule is straightforward: if your keys share prefixes and you ever need to ask "what keys start with X," reach for a trie. If they don't, or if you don't, use a HashMap. The 100 lines above are enough to start prototyping. When you outgrow them, the [`patricia_tree`](https://crates.io/crates/patricia_tree), [`radix_trie`](https://crates.io/crates/radix_trie), and [`fst`](https://crates.io/crates/fst) crates are waiting.
