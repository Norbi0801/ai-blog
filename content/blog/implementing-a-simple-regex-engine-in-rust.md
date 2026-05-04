+++
title = "Implementing a simple regex engine in Rust"
date = 2025-10-22
description = "Building a minimal regex engine in ~200 lines of Rust using Thompson's NFA construction - then comparing it to the regex crate, including why backtracking engines hit ReDoS and finite automata don't."

[taxonomies]
tags = ["rust", "algorithms", "regex", "automata"]
+++

Regex looks like magic punctuation that either does what you want or quietly hangs your service for an hour. Most developers reach for `regex::Regex::new("...")` and move on. But the difference between a regex that finishes in 30 microseconds on a megabyte of input and one that crashes a Node.js process on a 100-byte string comes down to which algorithm sits behind that constructor. There are two camps - backtracking and finite automata - and the choice shapes everything: performance, security, supported features, even how the pattern compiles.

The fastest way to internalize this is to build one of each path. We're going the finite-automata route, because that's what Rust's [regex crate](https://crates.io/crates/regex) does, and because it's the path that doesn't blow up on adversarial input. The result is around 200 lines of Rust that supports literal characters, `.`, `*`, `+`, and `?`. No groups, no alternation, no backreferences. Just enough to see how Thompson's construction turns a pattern into a state machine, and how simulating that machine matches input in linear time.

<!-- more -->

## Two algorithms, two universes

A regex engine has to answer one question: does this pattern match this string? There are two fundamentally different ways to do it.

**Backtracking engines** (PCRE, Python's `re` module, JavaScript's `RegExp`, Java's `java.util.regex`) walk the pattern recursively. When they hit a `*` or `?`, they greedily consume input. If the rest of the pattern fails to match, they back up and try a shorter consumption. This naturally handles features like backreferences (`\1`), lookaround, and capturing groups - those things require remembering specific positions in the input, which is exactly what a recursive walk gives you for free.

**Finite-automata engines** (RE2, Go's `regexp`, Rust's `regex`) compile the pattern into a state machine. A nondeterministic finite automaton (NFA) lets multiple states be "active" at once. Matching is just stepping the machine forward one input character at a time, advancing every active state in parallel. There's no backtracking because there's nothing to back up over - all possibilities are explored simultaneously.

The trade-off is real. Backtrackers support features automata can't (efficiently), at the cost of pathological worst cases. Automata guarantee O(n*m) matching time but give up backreferences. The Rust regex crate's [design doc](https://github.com/rust-lang/regex/blob/master/HACKING.md) is upfront about this: "The single most important consequence of this design is that the regex engine cannot perform backtracking, but it also cannot easily implement features like backreferences."

We're building the automata version. Let's start with the data model.

## Modeling the NFA

An NFA is a graph. Each state has one transition condition (consume a specific character, consume any character, or take a free epsilon move) and a list of states it transitions to:

```rust
use std::collections::HashSet;

#[derive(Debug, Clone)]
enum Trans {
    Char(char),  // consume this exact char
    Any,         // consume any single char (.)
    Epsilon,     // free move, no input consumed
    Match,       // accept state
}

#[derive(Debug)]
struct State {
    trans: Trans,
    outs: Vec<usize>,  // indices into NFA.states
}

#[derive(Debug)]
struct NFA {
    states: Vec<State>,
    start: usize,
}
```

States are stored in a flat `Vec` and referenced by index. This avoids the lifetime headaches you'd get with `Box<State>` or `Rc<State>` once you start building cycles - and `*` produces cycles, so you will. Index-based graphs in Rust are the path of least pain when the graph isn't tree-shaped.

A quick check on memory:

```rust
println!("State: {} bytes", std::mem::size_of::<State>());
// State: 32 bytes
```

That's 8 bytes for the `Trans` enum (largest variant is `Char(char)` - 4 bytes for `char`, plus discriminant and alignment) plus 24 bytes for `Vec<usize>` (pointer, len, cap). For the pattern `a+b*c?`, we end up with 7-8 states - roughly 256 bytes total. Compared to the input we're matching against, the NFA is tiny.

## Thompson's construction

Ken Thompson's [1968 paper "Regular Expression Search Algorithm"](https://dl.acm.org/doi/10.1145/363347.363387) gave us the recipe for turning a regex into an NFA in linear time and linear space. The trick is composability: each subpattern compiles into a "fragment" with one entry state and a set of "dangling" output edges that haven't been wired up yet. When fragments combine, you wire the dangling edges of one into the entry of the next.

```rust
#[derive(Debug)]
struct Frag {
    start: usize,
    dangling: Vec<usize>,  // states whose outs need to be patched
}

struct Compiler {
    states: Vec<State>,
}

impl Compiler {
    fn new() -> Self { Compiler { states: vec![] } }

    fn add(&mut self, trans: Trans) -> usize {
        let id = self.states.len();
        self.states.push(State { trans, outs: vec![] });
        id
    }

    fn patch(&mut self, dangling: &[usize], target: usize) {
        for &s in dangling {
            self.states[s].outs.push(target);
        }
    }
}
```

Each construction rule maps to a small method:

```rust
impl Compiler {
    fn atom(&mut self, t: Trans) -> Frag {
        let s = self.add(t);
        Frag { start: s, dangling: vec![s] }
    }

    fn concat(&mut self, a: Frag, b: Frag) -> Frag {
        self.patch(&a.dangling, b.start);
        Frag { start: a.start, dangling: b.dangling }
    }

    fn star(&mut self, a: Frag) -> Frag {
        // SPLIT state: epsilon to either "enter a" or "skip a"
        let split = self.add(Trans::Epsilon);
        self.states[split].outs.push(a.start);
        // a's exit loops back to the split
        self.patch(&a.dangling, split);
        Frag { start: split, dangling: vec![split] }
    }

    fn plus(&mut self, a: Frag) -> Frag {
        // Like star, but enter a at least once
        let split = self.add(Trans::Epsilon);
        self.states[split].outs.push(a.start);
        self.patch(&a.dangling, split);
        Frag { start: a.start, dangling: vec![split] }
    }

    fn question(&mut self, a: Frag) -> Frag {
        // Either run a, or skip it
        let split = self.add(Trans::Epsilon);
        self.states[split].outs.push(a.start);
        let mut dangling = vec![split];
        dangling.extend(a.dangling);
        Frag { start: split, dangling }
    }
}
```

The `star` case is the one to stare at. We make a SPLIT state - an epsilon-only state with two outgoing edges. The first edge is wired into the start of the inner fragment `a`. The second edge is left dangling, because we don't yet know what comes after the `*` in the larger pattern. We then loop the inner fragment's exit back to the SPLIT state. That's the cycle. When the matcher is "in" the SPLIT state, it can either dive into `a` again (matching another iteration) or take the dangling exit (stopping iteration). Both possibilities are explored simultaneously by the NFA simulator.

`plus` is `star` rearranged: the difference is that the entry point is `a.start` rather than the split, so we have to traverse `a` at least once before the split offers an exit.

`question` doesn't loop - the SPLIT state's two epsilons go to either `a.start` (run it once) or out (skip it).

## Parsing the pattern

The parser is dumb on purpose. Walk left to right, recognize an "atom" (character or `.`), then peek for a postfix operator (`*`, `+`, `?`) and apply it:

```rust
fn compile(pattern: &str) -> NFA {
    let mut c = Compiler::new();
    let chars: Vec<char> = pattern.chars().collect();
    let mut current: Option<Frag> = None;
    let mut i = 0;

    while i < chars.len() {
        let atom_trans = match chars[i] {
            '.' => Trans::Any,
            ch => Trans::Char(ch),
        };
        let mut frag = c.atom(atom_trans);
        i += 1;

        if i < chars.len() {
            match chars[i] {
                '*' => { frag = c.star(frag); i += 1; }
                '+' => { frag = c.plus(frag); i += 1; }
                '?' => { frag = c.question(frag); i += 1; }
                _ => {}
            }
        }

        current = Some(match current {
            None => frag,
            Some(prev) => c.concat(prev, frag),
        });
    }

    let frag = current.expect("empty pattern");
    let match_state = c.add(Trans::Match);
    c.patch(&frag.dangling, match_state);
    NFA { states: c.states, start: frag.start }
}
```

That's it. The whole compiler is under 80 lines. It's intentionally missing escape handling, alternation (`|`), grouping (`(...)`), and character classes (`[a-z]`) - each of those is a multi-day rabbit hole on its own, and the point here is the construction itself.

## Simulating the NFA

The matcher is the other half. It maintains a set of "currently active" states, advances all of them on each input character, and accepts if any of them is the `Match` state at the end:

```rust
fn matches(nfa: &NFA, input: &str) -> bool {
    let mut current: HashSet<usize> = HashSet::new();
    add_state(&nfa.states, nfa.start, &mut current);

    for ch in input.chars() {
        let mut next: HashSet<usize> = HashSet::new();
        for &s in &current {
            let consumes = match &nfa.states[s].trans {
                Trans::Char(c) => *c == ch,
                Trans::Any => true,
                _ => false,
            };
            if consumes {
                for &out in &nfa.states[s].outs {
                    add_state(&nfa.states, out, &mut next);
                }
            }
        }
        current = next;
    }

    current.iter().any(|&s| matches!(nfa.states[s].trans, Trans::Match))
}

fn add_state(states: &[State], s: usize, set: &mut HashSet<usize>) {
    if !set.insert(s) {
        return;  // already added, avoids infinite loops on epsilon cycles
    }
    if matches!(states[s].trans, Trans::Epsilon) {
        for &out in &states[s].outs {
            add_state(states, out, set);
        }
    }
}
```

`add_state` is the epsilon closure: when you add a state to the active set, you also add every state reachable from it via epsilon edges. The `set.insert(s)` guard is what prevents infinite recursion when the NFA has cycles - and `*` always produces cycles.

The size of `current` is bounded by the number of states in the NFA. That's the whole reason this runs in O(n*m) time: each input character does at most O(m) work, where m is the number of NFA states.

A quick smoke test:

```rust
fn main() {
    let nfa = compile("a+b*c?");
    assert!(matches(&nfa, "a"));
    assert!(matches(&nfa, "abc"));
    assert!(matches(&nfa, "aaaabbb"));
    assert!(matches(&nfa, "aaa"));
    assert!(!matches(&nfa, ""));
    assert!(!matches(&nfa, "b"));

    let dotted = compile("a.b");
    assert!(matches(&dotted, "axb"));
    assert!(matches(&dotted, "a4b"));
    assert!(!matches(&dotted, "ab"));
}
```

This is an *anchored* match - it requires the entire input to be consumed by the pattern. To get "find anywhere" semantics, you'd prepend `.*` to the compiled pattern (which is roughly what `regex::Regex::find` does internally for unanchored searches). Total file size: around 180 lines including blank lines and the test main. We're under budget.

## Why backtracking engines die on `(a+)+`

Now to the part that actually matters in production. Take the regex `(a+)+$` and run it against `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"` (30 a's followed by a `!`). In Python:

```python
import re, time
start = time.perf_counter()
re.match(r"(a+)+$", "a" * 30 + "!")
print(f"{time.perf_counter() - start:.2f}s")
# ~5 seconds and climbing per added 'a'
```

Add one more `a` and the time roughly doubles. By 35 a's you're waiting minutes. By 40 you've gone home. This is **catastrophic backtracking**, and it's the foundation of an entire class of denial-of-service attacks called [ReDoS](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS).

Why does it happen? The pattern `(a+)+` is ambiguous: a string of N a's can be split into groups in 2^(N-1) different ways. The outer `+` can run once and grab all of them, or twice with various splits, or three times, and so on. A backtracking engine tries each split sequentially. When the trailing `!` fails to match (because the input ends with `!` not the expected end-of-string after a's), the engine backtracks and tries the next split. With no early-abort heuristic for "I've already tried this state", it explores all 2^(N-1) possibilities.

The same input through Rust's regex crate finishes in microseconds:

```rust
use regex::Regex;
let re = Regex::new(r"(a+)+$").unwrap();
let input = "a".repeat(30) + "!";
let start = std::time::Instant::now();
let _ = re.is_match(&input);
println!("{:?}", start.elapsed());
// Roughly 5-50us regardless of input length
```

Because the NFA simulation tracks the *set* of active states rather than a single path through the pattern, it doesn't matter that there are exponentially many ways the input could split. There are only a fixed number of states, and we never visit more than O(states * input_length) (state, position) pairs total. ReDoS just isn't a category of bug that exists in this engine.

This is why CloudFlare's [July 2019 outage](https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/) was such a big deal - a single regex deployed to their WAF rules engine took 100% of CPU on every machine in their fleet because PCRE backtracked. Their post-mortem ends with "we're moving to RE2". RE2 is the C++ ancestor of the same NFA-simulation approach Rust's regex crate uses.

## What the regex crate actually does

Our toy engine is a single algorithm: NFA simulation with epsilon closure. The real Rust [regex crate](https://docs.rs/regex/) is way more interesting. As of v1.11, it's a [hybrid of multiple engines](https://github.com/rust-lang/regex/blob/master/HACKING.md) selected dynamically by a meta-matcher:

**Pike VM.** This is the closest thing to what we built - a generalized NFA simulator that also tracks capture group positions. It's the slowest engine but supports the full feature set. It's the fallback when nothing faster applies.

**Bounded backtracking.** For small patterns and small inputs, the crate uses a backtracking engine - but with a key safety net. It maintains a bitset of (state, input_position) pairs it has already visited and refuses to visit them twice. This bounds the total work at O(states * input_length), which is the same guarantee as NFA simulation. You get the speed of backtracking without the catastrophic worst case.

**Lazy DFA.** Compiles the NFA into a deterministic finite automaton on demand. A DFA has no nondeterminism - one transition per (state, character) pair - so matching is just a tight loop indexing into a transition table. Building the full DFA upfront would be exponential in the worst case, so it's built lazily as input is consumed, with an LRU cache that bounds memory. The [aho-corasick crate](https://crates.io/crates/aho-corasick) is used for multi-pattern literal optimization in the same spirit.

**One-pass NFA.** A specialized engine for patterns where the NFA happens to be deterministic without further work. Many real-world patterns fall into this category.

**Literal scanning.** Before any of the above runs, the meta-matcher checks if the pattern starts with a literal prefix. If `^Hello, ` is the start, it uses [memchr](https://crates.io/crates/memchr) (which uses SIMD) to skip through the input looking for the prefix, only running the regex engine on candidate matches. For many practical patterns this is the dominant cost.

The crate's [PERFORMANCE.md](https://github.com/rust-lang/regex/blob/master/PERFORMANCE.md) is worth reading end to end. It explains why simple changes like compiling a `Regex` once and reusing it (instead of re-compiling on every call) matter a lot - the meta-matcher does extensive analysis on the pattern at construction time.

## Performance comparison

Quick benchmark on a million-character input matching `a*b`:

```rust
use std::time::Instant;
use regex::Regex;

fn main() {
    let input = "a".repeat(1_000_000) + "b";

    let our_nfa = compile("a*b");
    let start = Instant::now();
    let _ = matches(&our_nfa, &input);
    println!("ours:    {:?}", start.elapsed());

    let real = Regex::new("^a*b$").unwrap();
    let start = Instant::now();
    let _ = real.is_match(&input);
    println!("regex:   {:?}", start.elapsed());
}
```

On my machine our toy engine takes around 35ms; the regex crate finishes in roughly 400 microseconds - about 80x faster on this input. The gap comes mostly from `HashSet<usize>` allocation per character (we could use a `Vec<bool>` instead) and the absence of literal scanning (the regex crate just `memchr`s for `b` and confirms everything before it is `a`). Build a real DFA and add SIMD literal scanning and you'd close most of that gap. But you would not be writing it in 200 lines anymore.

## What you take away

This whole exercise is around 200 lines that fit in a single file and a single afternoon. What you walk away with isn't a usable regex library - it's the mental model. When you read the regex crate's docs and they say "linear time worst case guaranteed", you now know what guarantee that is and why backtracking engines can't make it. When a teammate proposes filtering user input through `(.*)+` in PCRE, you know exactly what you're looking at. When the engine you reach for in your stack supports backreferences, you know what you're trading away.

The code lives in one `main.rs`. Compile it. Try `(.*?)b` (you'll discover lazy quantifiers are a whole separate construction). Try adding alternation - you'll need a new SPLIT case in the construction and a parser that handles operator precedence. Try character classes; you'll realize `Trans::Char(char)` should probably become `Trans::CharSet(BitSet)`. Each addition reveals more about what the [Russ Cox articles](https://swtch.com/~rsc/regexp/) on regex implementation are talking about - those are the canonical reference if you want to take this further, and they are where the design of RE2 (and by extension Rust's regex crate) was first laid out in public.

The pattern compiles to states. The states form a graph. The matcher walks the graph one character at a time. That's the whole secret. Every other regex engine in production is some elaboration on this idea or a deliberate departure from it.
