+++
title = "Implementing a skip list in Rust"
date = 2025-10-28
description = "Building a skip list from scratch in ~200 lines of Rust - probabilistic O(log n) search with random promotion, raw pointers, and memory layout that beats balanced trees on cache."

[taxonomies]
tags = ["rust", "algorithms", "data-structures", "memory"]
+++

A skip list is a sorted data structure that gives you O(log n) search, insert, and delete without any of the rotation gymnastics of red-black or AVL trees. It does this with two ingredients: linked lists stacked in layers, and a coin flip. That is the entire trick. Bill Pugh introduced it in 1990 in [a paper titled "Skip Lists: A Probabilistic Alternative to Balanced Trees"](https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplist-cacm1990.pdf), and the abstract says it plainly: skip lists are "a data structure that can be used in place of balanced trees... algorithms for insertion and deletion in skip lists are much simpler and significantly faster than equivalent algorithms for balanced trees."

You see them in production every day. Redis uses a skip list to back sorted sets (`ZADD`, `ZRANGE`); the implementation lives in [`t_zset.c`](https://github.com/redis/redis/blob/unstable/src/t_zset.c). LevelDB uses one for its in-memory write buffer. Java's `ConcurrentSkipListMap` is a skip list. The [`crossbeam-skiplist`](https://crates.io/crates/crossbeam-skiplist) crate gives Rust a lock-free version. The structure is popular with concurrent map authors because, unlike a balanced tree, you can insert and delete without ever rebalancing - so locks stay local and lock-free implementations stay tractable.

Let's build a single-threaded one in about 200 lines and see what makes it tick, including the part where Rust's ownership rules get genuinely awkward.

<!-- more -->

## The structure

A skip list is several sorted linked lists stacked on top of each other. The bottom list (level 0) contains every element. Each level above contains a random subset of the level below. The top is sparse, the bottom is dense.

```
level 3:  HEAD -------------------> 25 -----------------> NIL
level 2:  HEAD -------> 10 -------> 25 -------> 47 -----> NIL
level 1:  HEAD -> 3 --> 10 -> 18 -> 25 -> 32 -> 47 -----> NIL
level 0:  HEAD -> 3 --> 10 -> 18 -> 25 -> 32 -> 47 -> 56 -> NIL
```

To search for `32`, you start at the top-left. At level 3, the next pointer is `25`, then `NIL` - 32 is between 25 and NIL, so drop down. At level 2, you're sitting on `25`, next is `47` which overshoots - drop down. At level 1, you're on `25`, next is `32`. Found it.

Each step either moves right (skipping over many elements) or drops down one level. With n elements and roughly `log2(n)` levels, you do O(log n) work in expectation. The "in expectation" part matters - this is a probabilistic data structure. Worst case is O(n), but the probability of hitting that worst case is astronomically small once you have any reasonable number of elements.

## Why probabilistic

The reason skip lists exist is that balanced trees are annoying. Red-black trees have five rotation cases. AVL trees rebalance on every insert. B-trees split nodes. All of these are correct, fast, and a pain to implement without bugs.

A skip list replaces "balance" with "randomize." When you insert a node, you flip a coin. If it lands heads, the node is promoted to level 1. Flip again - heads, promote to level 2. Keep going until you get tails. That is the entire balancing strategy. There is no rebalancing on delete. Insert is local. Delete is local. The structure is self-similar at every layer.

The expected height of a node is `1 / (1 - p)` where `p` is the promotion probability. With `p = 0.5`, expected height is 2. The expected number of pointers in the entire structure is `2n`. Search depth is `log_{1/p}(n)`, which for `p = 0.5` is `log2(n)`.

Pugh's paper shows that for n = 2^14, the probability of search exceeding `3 * log2(n)` comparisons is less than 10^-6. In practice, you set a hard ceiling on the height (usually 32 levels, enough for 2^32 elements with `p = 0.5`) and stop worrying.

## Setting up

We're going to build this with no dependencies beyond `std`. The interface is a sorted set with `insert`, `contains`, `remove`.

```rust
use std::cmp::Ordering;
use std::ptr::NonNull;

const MAX_HEIGHT: usize = 32;
const P: u32 = u32::MAX / 2; // promotion probability ~ 0.5

pub struct SkipList<T: Ord> {
    head: NonNull<Node<T>>,
    height: usize,
    len: usize,
    rng: u64, // tiny xorshift state
}
```

The head is a sentinel node with no value. Every level starts at the head. We use `NonNull` rather than `*mut` because skip lists are full of pointers that should never be null mid-list, and `NonNull` makes that intent visible to both the compiler and the optimizer (it enables niche optimization on `Option<NonNull<T>>`).

## The node and the ownership problem

Here is where a skip list gets interesting in Rust. A node has a value, and an array of `next` pointers - one per level it appears on. The natural way to write that in any other language:

```rust
struct Node<T> {
    value: Option<T>,             // None for the head sentinel
    next: Vec<*mut Node<T>>,      // one pointer per level
}
```

This compiles. It is also wrong in spirit. In Rust, the ownership question is: who owns the node? You can't put `Box<Node<T>>` in two `next` arrays at once, because `Box` is unique ownership. But every node above level 0 is pointed to by exactly one node per level - so multiple "owners" exist if we squint at them as such.

The honest answer is that skip lists are inherently a structure with shared mutable pointers between siblings, and Rust's ownership model does not have an idiom that fits. You have three options:

1. `Rc<RefCell<Node<T>>>` - reference counted, runtime borrow checks. Slow, and the borrow checker becomes a runtime panic.
2. `Arc<Mutex<Node<T>>>` - same idea, plus locks. Slower.
3. Raw pointers, `unsafe`, and a careful `Drop` impl.

Production crates pick option 3. So do we. The price is admitting that you are writing C with a Rust accent for one specific module, and being disciplined about it.

```rust
struct Node<T> {
    value: Option<T>,
    height: usize,
    // Flexible array - allocated once with the node, length = height.
    // We store this as a pointer + length we control manually.
    next: [Option<NonNull<Node<T>>>; MAX_HEIGHT],
}

impl<T> Node<T> {
    fn new(value: Option<T>, height: usize) -> NonNull<Self> {
        let node = Box::new(Node {
            value,
            height,
            next: [None; MAX_HEIGHT],
        });
        // Box::into_raw transfers ownership to us. We are now responsible
        // for calling Box::from_raw to free it.
        NonNull::new(Box::into_raw(node)).unwrap()
    }
}
```

I'm using a fixed-size array of length `MAX_HEIGHT` rather than a true flexible array member. A real implementation would allocate exactly `height` pointer slots to save memory - `crossbeam-skiplist` does this with manual layout calls. For 32-level max height that's 256 wasted bytes per node in the worst case, which matters at scale but not in a learning post. The trade-off: a fixed array means simpler `Drop`, no manual `Layout::from_size_align`, and zero pointer arithmetic.

`Box::into_raw` is the key call. It hands ownership of the heap allocation from `Box` to the raw pointer. Until we call `Box::from_raw`, the allocation leaks. That is the bargain.

## Search

Search is the operation that does no mutation, so it's the easiest place to start. It also drives the structure of insert and delete - both walk the list the same way and stash bookkeeping along the way.

```rust
impl<T: Ord> SkipList<T> {
    pub fn contains(&self, value: &T) -> bool {
        unsafe {
            let mut current = self.head;
            // Walk from top level down.
            for level in (0..self.height).rev() {
                while let Some(next) = current.as_ref().next[level] {
                    let next_ref = next.as_ref();
                    match next_ref.value.as_ref().unwrap().cmp(value) {
                        Ordering::Less => current = next,
                        Ordering::Equal => return true,
                        Ordering::Greater => break,
                    }
                }
            }
            false
        }
    }
}
```

Read this carefully. At every level, we walk forward as long as `next.value < target`. When `next.value >= target`, we stop and drop down a level. If we land exactly on the target, return true. If we fall off level 0 without finding it, return false.

The whole function is wrapped in one `unsafe` block. Every `as_ref()` call on a `NonNull` is a contract: you are promising that the pointer is valid, points to a live object, and that no one else is mutating it during this borrow. In a single-threaded skip list with proper Drop, those invariants hold. In a concurrent one, you'd need atomics and epochs - that's what `crossbeam-skiplist` does and why it's 4000+ lines instead of 200.

## The random level

Before we write insert, we need the coin flip. The classic implementation:

```rust
fn random_level(&mut self) -> usize {
    // Tiny xorshift64* PRNG. Good enough for promotion decisions.
    self.rng ^= self.rng << 13;
    self.rng ^= self.rng >> 7;
    self.rng ^= self.rng << 17;

    let mut level = 1;
    let mut bits = self.rng;
    while level < MAX_HEIGHT && (bits as u32) < P {
        level += 1;
        bits >>= 1;
    }
    level
}
```

We don't call `rand::random()` because that's a dependency, and because we get to demonstrate something nice: each promotion decision is one bit of randomness, so we can extract `MAX_HEIGHT` decisions from a single 64-bit value. One PRNG step amortizes across the whole insert.

Compare this to a balanced tree, which needs zero randomness but has 5+ rotation cases. Pugh's whole point was that randomness is cheaper than careful invariants.

## Insert

Insert has two phases. Phase one: walk the list like search, but record the rightmost node at each level whose `next` we'll need to update. Phase two: allocate the new node and splice it in.

```rust
pub fn insert(&mut self, value: T) -> bool {
    let mut update: [NonNull<Node<T>>; MAX_HEIGHT] = [self.head; MAX_HEIGHT];

    unsafe {
        let mut current = self.head;
        for level in (0..self.height).rev() {
            while let Some(next) = current.as_ref().next[level] {
                match next.as_ref().value.as_ref().unwrap().cmp(&value) {
                    Ordering::Less => current = next,
                    Ordering::Equal => return false, // already present
                    Ordering::Greater => break,
                }
            }
            update[level] = current;
        }

        let new_height = self.random_level();
        if new_height > self.height {
            for level in self.height..new_height {
                update[level] = self.head;
            }
            self.height = new_height;
        }

        let new_node = Node::new(Some(value), new_height);
        for level in 0..new_height {
            let pred = update[level].as_mut();
            (*new_node.as_ptr()).next[level] = pred.next[level];
            pred.next[level] = Some(new_node);
        }
        self.len += 1;
        true
    }
}
```

The `update` array is the bookkeeping. `update[i]` is the node at level `i` whose `next[i]` pointer needs to become the new node. The splice is the same two-line dance you'd do in any singly-linked list, repeated for each level the new node lives on.

If the new node is taller than the current list, we extend the head to cover it. The head's `next` array goes up to `MAX_HEIGHT` already, so this is just bumping `self.height`.

Returning `bool` to signal "actually inserted" matches `HashSet::insert`. Duplicates are rejected at the `Ordering::Equal` arm.

## Delete

Delete is the mirror of insert. Walk the list, record predecessors at each level, then splice the target out. Then free the node.

```rust
pub fn remove(&mut self, value: &T) -> bool {
    let mut update: [NonNull<Node<T>>; MAX_HEIGHT] = [self.head; MAX_HEIGHT];

    unsafe {
        let mut current = self.head;
        let mut target: Option<NonNull<Node<T>>> = None;

        for level in (0..self.height).rev() {
            while let Some(next) = current.as_ref().next[level] {
                match next.as_ref().value.as_ref().unwrap().cmp(value) {
                    Ordering::Less => current = next,
                    Ordering::Equal => {
                        target = Some(next);
                        break;
                    }
                    Ordering::Greater => break,
                }
            }
            update[level] = current;
        }

        let target = match target {
            Some(t) => t,
            None => return false,
        };

        let target_ref = target.as_ref();
        for level in 0..target_ref.height {
            let pred = update[level].as_mut();
            if pred.next[level] == Some(target) {
                pred.next[level] = target_ref.next[level];
            }
        }

        // Shrink height if top levels became empty.
        while self.height > 1 && self.head.as_ref().next[self.height - 1].is_none() {
            self.height -= 1;
        }

        // Reclaim. This is the matching half of Box::into_raw in Node::new.
        drop(Box::from_raw(target.as_ptr()));
        self.len -= 1;
        true
    }
}
```

`Box::from_raw` is what makes the memory safe to free. It re-wraps the raw pointer in a `Box`, which then gets dropped at the end of the statement. The value `T` inside the `Option` runs its own `Drop` as part of dropping the `Box<Node<T>>`. Forget this call and you leak a node every time you delete one, which is a class of bug Rust normally prevents but cannot help you with once you've crossed into raw pointers.

## Drop

The big one. When the skip list itself is dropped, every node it owns must be freed. With a tree this is recursive. With a skip list, every node is on the level-0 list, which is just a singly-linked list. So we walk level 0 and free each node:

```rust
impl<T: Ord> Drop for SkipList<T> {
    fn drop(&mut self) {
        unsafe {
            let mut current = self.head.as_ref().next[0];
            while let Some(node) = current {
                current = node.as_ref().next[0];
                drop(Box::from_raw(node.as_ptr()));
            }
            // Don't forget the head sentinel.
            drop(Box::from_raw(self.head.as_ptr()));
        }
    }
}
```

The order matters. Read `current = node.next[0]` *before* freeing `node`, otherwise you have a use-after-free on the next iteration. This is the kind of bug that makes `unsafe` Rust feel like C - because at this layer, it is C.

## Memory layout

Let's stop and look at what we just built. Each node looks like this in memory (assuming `T = u64` on a 64-bit machine):

```
offset  field        size
  0     value tag    8         (Option<u64> discriminant + padding)
  8     value        8
 16     height       8         (usize)
 24     next[0]      8         (NonNull, niche-optimized so 8 bytes not 16)
 32     next[1]      8
  ...
280     next[31]     8
       ──────────
       288 bytes per node
```

288 bytes per node is a lot. For nodes that only live on level 0 (about half of them), 31 of those 32 pointer slots are wasted. The `crossbeam-skiplist` source allocates exactly `height` slots using [`std::alloc::Layout`](https://doc.rust-lang.org/std/alloc/struct.Layout.html) and the global allocator directly, dropping average node size to roughly `48 + 8 * expected_height` = ~64 bytes for `p = 0.5`.

The cache story is interesting. A red-black tree node is small (~40 bytes) but you chase O(log n) random pointers, each one a cache miss. A skip list at level 0 is just a singly-linked list - linearly scannable, prefetcher-friendly. The upper levels do random-looking jumps, but each jump is to a node that, once warm, sits in L1 for the rest of the operation. Cache behavior is one of the reasons skip lists hold up well in practice despite the apparent overhead.

## Why concurrent maps love this

Insert into a balanced tree can rotate a node arbitrarily far from where you started. That means a concurrent BST has to either lock the whole tree, lock paths from the root down, or use complex reader/writer schemes. Java's `ConcurrentSkipListMap` and Rust's `crossbeam-skiplist` both choose skip lists because each insert touches at most `O(log n)` adjacent nodes - and those touches are independent at each level.

Lock-free implementations use atomic CAS on the `next` pointers. The classic algorithm is by [Maged Michael (2002)](https://www.cs.tau.ac.il/~shanir/concurrent-data-structures.pdf): each pointer carries a "marked" bit indicating logical deletion, and inserts do CAS to splice the new node into level 0 first, then bottom-up to higher levels. Because nodes are never moved, only inserted and unlinked, the structure stays correct under concurrent readers without any locks at all.

This is also what makes Redis sorted sets fast under concurrent load (though Redis itself is single-threaded - it benefits from skip list's other property: simple, allocation-light operations).

## When to actually use one

In single-threaded Rust, prefer `BTreeMap` or `BTreeSet`. They beat skip lists on cache and constant factors. The standard library is well-tuned. There is no everyday reason to write a skip list in Rust unless you specifically want concurrency or you're building something like an LSM tree's memtable.

If you do want a skip list, [`crossbeam-skiplist`](https://docs.rs/crossbeam-skiplist) is the production answer. It's lock-free, epoch-based reclamation, and battle-tested. The point of writing one yourself is the same as writing a Merkle tree or a diff algorithm yourself - you stop seeing it as a black box. You see the coin flip, you see the splice, you see the `Box::into_raw` / `Box::from_raw` pair that makes the lifetime work, and you understand why the structure is shaped the way it is.

That understanding is what lets you read the [`t_zset.c` source](https://github.com/redis/redis/blob/unstable/src/t_zset.c) without flinching. 200 lines of unsafe Rust is a cheap price for that.
