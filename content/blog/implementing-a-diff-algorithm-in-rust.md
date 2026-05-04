+++
title = "Implementing a diff algorithm in Rust"
date = 2025-10-19
description = "Building Myers diff from scratch - edit graphs, shortest edit scripts, and generating unified diff output in ~200 lines of Rust with zero dependencies."

[taxonomies]
tags = ["rust", "algorithms", "git"]
+++

Every `git diff`, every code review, every merge conflict runs through the same fundamental algorithm: given two sequences, find the minimal set of edits that turns one into the other. Most developers treat this as a black box. Run `diff a.txt b.txt`, get some output with `+` and `-` lines, move on. But the algorithm behind it is elegant, and implementing it yourself teaches you something no amount of documentation reading will.

We're building a working diff tool in about 200 lines of Rust. No external crates. Just `std`. By the end you'll know what `git diff` computes, how it computes it, and why some diffs look weird while others don't.

<!-- more -->

## What diff actually computes

The diff problem: given sequence A (the old file) and sequence B (the new file), find the shortest edit script (SES) that transforms A into B using only two operations - delete an element from A, or insert an element from B. Elements present in both sequences stay for free.

For files, each "element" is a line. If `old = ["A", "B", "C"]` and `new = ["A", "C"]`, the shortest edit script is: keep A, delete B, keep C. One edit.

The length of the shortest edit script is the edit distance D. Eugene Myers published an algorithm in 1986 that finds this in O((M+N) * D) time, where M and N are the lengths of the two sequences. When the sequences are similar (small D), this is nearly linear. When they're completely different (D = M+N), it degrades to O((M+N)^2) - but at that point any algorithm struggles.

The paper is "An O(ND) Difference Algorithm and Its Variations" (Algorithmica, 1986). It's what GNU diff and Git use by default. The [PDF is freely available](http://www.xmailserver.org/diff2.pdf) and surprisingly readable for an academic paper.

## The edit graph

Myers' core insight is geometric. Map the diff problem onto a graph:

- The X-axis represents positions in the old sequence (length N)
- The Y-axis represents positions in the new sequence (length M)
- Moving **right** (x+1) means deleting an element from old. Costs 1.
- Moving **down** (y+1) means inserting an element from new. Costs 1.
- Moving **diagonally** (x+1, y+1) means the elements match. Free.

Here's what this looks like for `old = ["A", "B"]`, `new = ["B", "C"]`:

```
            A     B
        (0,0)-(1,0)-(2,0)
          |         \  |
    B   (0,1)-(1,1)-(2,1)
          |     |     |
    C   (0,2)-(1,2)-(2,2)
```

The diagonal from (1,0) to (2,1) exists because `old[1] == new[0]` - both are "B". Finding the shortest edit script means finding the path from (0,0) to (2,2) with the fewest horizontal and vertical moves. Diagonals are free.

The optimal path: right to (1,0) [delete A], diagonal to (2,1) [match B], down to (2,2) [insert C]. Two edits.

This is fundamentally a shortest-path problem. Classical edit distance (Levenshtein) solves it with a full N*M dynamic programming table. Myers' insight is that you don't need the full table - you can run a BFS-like search outward from (0,0), where each "level" corresponds to one more edit. When D is small relative to N and M, you skip most of the grid entirely.

## Diagonals, D-contours, and snakes

Every point (x, y) on the grid sits on a diagonal numbered `k = x - y`. The starting point (0,0) is on diagonal 0. A deletion (right move) increases k by 1. An insertion (down move) decreases k by 1. A match (diagonal move) keeps k the same.

This means after exactly D edits, you can only be on diagonals where k has the same parity as D and |k| <= D. For D=0 you're on diagonal 0 only, for D=1 on {-1, 1}, for D=2 on {-2, 0, 2}, and so on. The algorithm iterates D = 0, 1, 2, ... and for each D computes the furthest reachable point on each valid diagonal.

A "snake" is a sequence of free diagonal moves after an edit. The algorithm greedily extends them. Two files differing by only a few lines produce long snakes and a small D, keeping the algorithm fast.

The data structure is a single array V where `V[k]` stores the furthest x-coordinate reached on diagonal k. For each new D, each diagonal extends from the better of two neighbors: diagonal k-1 (via a deletion) or diagonal k+1 (via an insertion). Then it slides diagonally through any matching elements.

## Implementing the forward search

```rust
#[derive(Debug, Clone)]
enum Change<'a> {
    Equal(&'a str),
    Delete(&'a str),
    Insert(&'a str),
}
```

Lifetimes keep us from copying strings. Each `Change` borrows from the input slices.

The forward search finds the edit distance and stores a snapshot of V at each step so we can retrace our path later:

```rust
fn diff<'a>(old: &[&'a str], new: &[&'a str]) -> Vec<Change<'a>> {
    let (n, m) = (old.len(), new.len());
    let max = n + m;
    if max == 0 {
        return vec![];
    }

    // V[k] = furthest x on diagonal k. Indexed as v[k + offset].
    let off = max as isize;
    let at = |k: isize| (k + off) as usize;
    let mut v = vec![0usize; 2 * max + 1];
    let mut trace: Vec<Vec<usize>> = Vec::new();
    let mut shortest = 0;

    'search: for d in 0..=max {
        for i in 0..=d {
            let k = -(d as isize) + 2 * i as isize;

            // Choose: extend from k+1 (insertion) or k-1 (deletion)?
            let mut x = if k == -(d as isize)
                || (k != d as isize && v[at(k - 1)] < v[at(k + 1)])
            {
                v[at(k + 1)] // came from k+1 via down move
            } else {
                v[at(k - 1)] + 1 // came from k-1 via right move
            };
            let mut y = (x as isize - k) as usize;

            // extend the snake: free diagonal moves
            while x < n && y < m && old[x] == new[y] {
                x += 1;
                y += 1;
            }
            v[at(k)] = x;

            if x >= n && y >= m {
                trace.push(v.clone());
                shortest = d;
                break 'search;
            }
        }
        trace.push(v.clone());
    }

    backtrack(&trace, old, new, shortest)
}
```

The outer loop increments D. The inner loop visits each valid diagonal for that D. The `at` closure maps diagonal k (which can be negative) to an array index by adding an offset.

The neighbor selection deserves a closer look. On boundary diagonals (k == -D or k == D), only one neighbor exists. For interior diagonals, we pick whichever neighbor has progressed further along x. When `v[at(k-1)] < v[at(k+1)]`, the k+1 neighbor consumed more of the old sequence, so we extend from it via an insertion. Otherwise we extend from k-1 via a deletion.

When the search reaches (N, M), we've found the minimum edit distance. The `trace` vector holds V snapshots - one per D value - which the backtracking phase uses to reconstruct the actual edit sequence.

## Recovering the edit script

The forward search tells us the edit distance but not the actual edits. To recover the edit script, we walk backwards through the trace snapshots. At each D, we figure out which move was taken by comparing with the V state at D-1:

```rust
fn backtrack<'a>(
    trace: &[Vec<usize>],
    old: &[&'a str],
    new: &[&'a str],
    shortest: usize,
) -> Vec<Change<'a>> {
    let off = (old.len() + new.len()) as isize;
    let at = |k: isize| (k + off) as usize;

    let mut x = old.len() as isize;
    let mut y = new.len() as isize;
    let mut ops: Vec<Change<'a>> = Vec::new();

    for d in (0..=shortest).rev() {
        let k = x - y;

        if d == 0 {
            // remaining moves are all diagonal (matches)
            while x > 0 {
                x -= 1;
                y -= 1;
                ops.push(Change::Equal(old[x as usize]));
            }
            break;
        }

        let prev = &trace[d - 1];
        let di = d as isize;
        let went_down = k == -di
            || (k != di && prev[at(k - 1)] < prev[at(k + 1)]);

        let prev_k = if went_down { k + 1 } else { k - 1 };
        let prev_x = prev[at(prev_k)] as isize;
        let prev_y = prev_x - prev_k;

        // the snake: diagonal moves that happened after the edit
        let snake_start = if went_down { prev_x } else { prev_x + 1 };
        while x > snake_start {
            x -= 1;
            y -= 1;
            ops.push(Change::Equal(old[x as usize]));
        }

        // the edit itself
        if went_down {
            ops.push(Change::Insert(new[prev_y as usize]));
        } else {
            ops.push(Change::Delete(old[prev_x as usize]));
        }

        x = prev_x;
        y = prev_y;
    }

    ops.reverse();
    ops
}
```

We work backwards from (N, M). For each edit D, we determine whether the D-th edit was an insertion (went_down) or deletion by replaying the same neighbor-selection logic against the previous V snapshot. Then we record any diagonal matches that form the snake, followed by the edit itself. Since we collect operations back-to-front, `reverse()` at the end puts them in the right order.

The `d == 0` base case handles the common prefix. If the files start with identical lines, those are all diagonal moves at D=0, and we record them as `Equal` operations.

## Generating unified diff output

The raw edit script is a flat list of Equal/Delete/Insert operations. Unified diff format groups these into hunks - regions of changes surrounded by context lines:

```
--- old_file
+++ new_file
@@ -1,4 +1,3 @@
 context
-deleted
+inserted
 context
```

The `@@ -a,b +c,d @@` header says: this hunk starts at line `a` in the old file and spans `b` lines, starts at line `c` in the new file and spans `d` lines. Context lines count in both. The default context is 3 lines, matching `git diff` and `diff -U3`.

To produce hunks, we scan for non-Equal changes, merge groups within `2 * context` lines of each other (so adjacent hunks don't awkwardly split), and expand each group with context:

```rust
fn print_diff(
    w: &mut impl Write,
    changes: &[Change],
    old_name: &str,
    new_name: &str,
    ctx: usize,
    color: bool,
) -> io::Result<bool> {
    let n = changes.len();
    let mut hunks: Vec<(usize, usize)> = Vec::new();
    let mut i = 0;

    while i < n {
        while i < n && matches!(changes[i], Change::Equal(_)) {
            i += 1;
        }
        if i >= n { break; }
        let start = i;
        while i < n {
            match changes[i] {
                Change::Equal(_) => {
                    let run = changes[i..]
                        .iter()
                        .take_while(|c| matches!(c, Change::Equal(_)))
                        .count();
                    if run > 2 * ctx { break; }
                    i += run;
                }
                _ => i += 1,
            }
        }
        hunks.push((start, i));
    }

    if hunks.is_empty() { return Ok(false); }

    writeln!(w, "--- {old_name}")?;
    writeln!(w, "+++ {new_name}")?;

    for &(start, end) in &hunks {
        let lo = start.saturating_sub(ctx);
        let hi = (end + ctx).min(n);

        // count old/new lines up to hunk start for line numbers
        let (mut ol, mut nl) = (0usize, 0usize);
        for c in &changes[..lo] {
            if !matches!(c, Change::Insert(_)) { ol += 1; }
            if !matches!(c, Change::Delete(_)) { nl += 1; }
        }
        // count old/new lines within the hunk
        let (mut oc, mut nc) = (0usize, 0usize);
        for c in &changes[lo..hi] {
            if !matches!(c, Change::Insert(_)) { oc += 1; }
            if !matches!(c, Change::Delete(_)) { nc += 1; }
        }

        let hdr = format!(
            "@@ -{},{} +{},{} @@", ol + 1, oc, nl + 1, nc
        );
        if color {
            writeln!(w, "\x1b[36m{hdr}\x1b[0m")?;
        } else {
            writeln!(w, "{hdr}")?;
        }

        for c in &changes[lo..hi] {
            match c {
                Change::Equal(s) => writeln!(w, " {s}")?,
                Change::Delete(s) => {
                    if color { write!(w, "\x1b[31m")? }
                    write!(w, "-{s}")?;
                    if color { write!(w, "\x1b[0m")? }
                    writeln!(w)?;
                }
                Change::Insert(s) => {
                    if color { write!(w, "\x1b[32m")? }
                    write!(w, "+{s}")?;
                    if color { write!(w, "\x1b[0m")? }
                    writeln!(w)?;
                }
            }
        }
    }
    Ok(true)
}
```

The hunk header math counts old lines (Equal + Delete) and new lines (Equal + Insert) within the hunk range. Context lines contribute to both counts. That's why the numbers in `@@` don't simply equal the number of `+` and `-` lines - the common context lines pad both sides.

ANSI escape codes handle color: `\x1b[31m` for red (deletions), `\x1b[32m` for green (insertions), `\x1b[36m` for cyan (hunk headers), `\x1b[0m` to reset. These are the same codes that `git diff` emits when `color.diff` is enabled - no crate needed.

## Word-level highlighting

Line-level diff tells you which lines changed. Word-level diff tells you what changed within those lines. The trick: reuse the exact same Myers algorithm on word tokens instead of lines.

```rust
fn tokenize(s: &str) -> Vec<&str> {
    let b = s.as_bytes();
    let (mut tokens, mut i) = (Vec::new(), 0);
    while i < b.len() {
        let start = i;
        let word = b[i].is_ascii_alphanumeric() || b[i] == b'_';
        while i < b.len()
            && (b[i].is_ascii_alphanumeric() || b[i] == b'_') == word
        {
            i += 1;
        }
        tokens.push(&s[start..i]);
    }
    tokens
}
```

The tokenizer splits on word boundaries - alphanumeric sequences vs. everything else. `"fn main() {"` becomes `["fn", " ", "main", "()", " ", "{"]`. Each token is a `&str` borrowing from the original line. No allocations.

To highlight inline changes, run `diff` on the tokens and wrap changed tokens with ANSI reverse-video:

```rust
fn word_highlight(
    old_line: &str,
    new_line: &str,
) -> (String, String) {
    let changes = diff(&tokenize(old_line), &tokenize(new_line));

    let (mut del, mut ins) = (
        String::from("\x1b[31m-"),
        String::from("\x1b[32m+"),
    );

    for c in &changes {
        match c {
            Change::Equal(w) => {
                del.push_str(w);
                ins.push_str(w);
            }
            Change::Delete(w) => {
                del.push_str("\x1b[7m"); // reverse video on
                del.push_str(w);
                del.push_str("\x1b[27m"); // reverse off
            }
            Change::Insert(w) => {
                ins.push_str("\x1b[7m");
                ins.push_str(w);
                ins.push_str("\x1b[27m");
            }
        }
    }
    del.push_str("\x1b[0m");
    ins.push_str("\x1b[0m");
    (del, ins)
}
```

`\x1b[7m` activates reverse video, swapping foreground and background colors. Deleted words appear as highlighted text on the red deletion line, inserted words as highlighted text on the green insertion line. This is essentially how [delta](https://github.com/dandavison/delta) highlights inline changes - a second diff pass on paired lines.

The same algorithm at different granularities. Lines for file-level diff, words for inline diff. You could go deeper - character-level, grapheme-level, even AST-level (which is what [difftastic](https://github.com/Wilfred/difftastic) does using tree-sitter to parse both files into syntax trees and diff the tree structure).

Integrating word highlighting into `print_diff` is straightforward: when you encounter a `Delete` followed by an `Insert`, call `word_highlight` on the pair and print the highlighted versions instead of the plain +/- lines.

## Putting it together

```rust
use std::env;
use std::fs;
use std::io::{self, BufWriter, IsTerminal, Write};

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() < 3 {
        eprintln!("usage: tinydiff <old> <new>");
        std::process::exit(2);
    }

    let read = |path: &str| {
        fs::read_to_string(path).unwrap_or_else(|e| {
            eprintln!("tinydiff: {path}: {e}");
            std::process::exit(2);
        })
    };
    let old_text = read(&args[1]);
    let new_text = read(&args[2]);

    let old: Vec<&str> = old_text.lines().collect();
    let new: Vec<&str> = new_text.lines().collect();
    let changes = diff(&old, &new);

    let color = io::stdout().is_terminal()
        && env::var("NO_COLOR").is_err();
    let out = io::stdout();
    let mut w = BufWriter::new(out.lock());

    let found = print_diff(
        &mut w, &changes, &args[1], &args[2], 3, color,
    ).unwrap_or(false);

    // Exit codes match diff convention: 0=same, 1=different, 2=error
    std::process::exit(i32::from(found));
}
```

Exit codes follow the `diff` convention: 0 means identical, 1 means differences found, 2 means error. This three-way distinction matters for scripting - `if tinydiff old.rs new.rs; then echo "files match"; fi` works correctly.

Color is enabled only when writing to a terminal and `NO_COLOR` is unset, respecting the [no-color.org](https://no-color.org/) convention. When piped to another program, colors are disabled automatically.

```bash
$ cargo run -- old.rs new.rs
--- old.rs
+++ new.rs
@@ -1,5 +1,5 @@
 fn main() {
-    let x = 42;
-    println!("{}", x);
+    let name = "world";
+    println!("hello, {name}");
     process();
 }
```

About 200 lines total, zero dependencies. Not production-grade - a real diff tool needs the linear space optimization, binary file detection, and more edge cases around trailing newlines. But it computes the same edit script as `git diff` for any two text files.

## What Git actually does under the hood

Git's diff engine lives in [`xdiff/xdiffi.c`](https://github.com/git/git/blob/master/xdiff/xdiffi.c), a modified version of the libxdiff library. Before March 2006, Git shelled out to the system `diff` command via `popen()`. Linus merged the built-in xdiff implementation because the fork/exec overhead was killing performance on large repositories.

The xdiff code uses the linear-space variant of Myers' algorithm. Our implementation stores the entire V array for every D, consuming O((N+M) * D) memory. For a 10,000-line file with 500 changes, that's roughly 100MB of trace data. The linear-space version avoids this with a divide-and-conquer strategy:

1. Run the search **forward** from (0,0) and **backward** from (N,M) simultaneously.
2. When the two wavefronts overlap on the same diagonal, the overlapping segment is the "middle snake."
3. The middle snake divides the problem in half.
4. Recurse on each half.

Memory drops to O(N+M) because you only need two V arrays (forward and backward), not the entire trace history. Time stays O((N+M) * D). The [`similar`](https://github.com/mitsuhiko/similar/blob/main/src/algorithms/myers.rs) crate by Armin Ronacher implements this variant in Rust - the `find_middle_snake` function runs the bidirectional search, and `conquer` handles the recursion.

## Beyond Myers: patience and histogram diff

Git supports four diff algorithms via `git diff --diff-algorithm=<name>`:

**Myers** (the default) finds the mathematically shortest edit script. Fast and usually produces good output. But "shortest" doesn't always mean "readable." When you reorder two functions in a file, Myers might match closing braces from one function with opening braces from another, producing a confusing diff even though the edit count is minimal.

**Patience diff** (by Bram Cohen, the BitTorrent creator) addresses this. It first identifies lines that appear exactly once in both files - typically function signatures, struct declarations, section headers. These unique lines become anchors. The algorithm computes the longest common subsequence of just the anchors, splits the file at those points, and recursively diffs each section with Myers.

The result: diffs that align on structural boundaries. Moving a function shows as a clean block deletion and insertion instead of a mess of mismatched braces. The cost is speed (unique-line preprocessing adds overhead) and it struggles when files have few unique lines.

**Histogram diff** was ported from JGit (the Java Git implementation) and merged into Git in version 1.7.7 (2011). Instead of anchoring only on unique lines, it builds a frequency histogram and anchors on the lowest-occurrence matches. This makes it strictly more general than patience diff - when unique lines exist, it behaves identically. When they don't, it falls back gracefully rather than punting to Myers.

A [2020 study by Nugroho, Hata, and Matsumoto](https://doi.org/10.1007/s10664-019-09772-z) compared all four algorithms across 14 Java projects. The algorithms produced different output in 2-8% of commits. More telling: 6-13% of bug-introducing change identifications differed between algorithms. The researchers recommended histogram diff for code changes. You can set it globally:

```bash
git config --global diff.algorithm histogram
```

**Minimal** is Myers with all heuristics disabled. It spends extra time to guarantee the absolute smallest possible diff. Useful for generating patches in constrained environments, but too slow for interactive use on large files.

Each algorithm has its place. Myers is a solid default. Patience and histogram trade minimal edit distance for human-readable output. If you're reviewing diffs daily, try `histogram` - the output tends to align better with how developers think about code changes.

## The O(ND) intuition

The N*D factor in O(ND) is what makes this algorithm practical for version control. When two files share 95% of their content, D is about 5% of N. The algorithm runs in roughly O(N) time - effectively linear. This is why `git diff` feels instant even on large files.

The classical Levenshtein distance algorithm fills a full N*M table regardless of similarity. For two 10,000-line files differing by 50 lines, Myers processes roughly 500,000 operations. Levenshtein processes 100,000,000. Three orders of magnitude difference.

The flip side: for completely unrelated files, D approaches N+M and Myers degrades to O((N+M)^2). But if two files share nothing, you don't need a diff - you need a replacement.

If you want a production-ready implementation rather than rolling your own, the Rust ecosystem has several solid options. [`similar`](https://crates.io/crates/similar) (v3.0, 112M downloads) supports five algorithms and powers the [`insta`](https://crates.io/crates/insta) snapshot testing framework. [`dissimilar`](https://crates.io/crates/dissimilar) by dtolnay focuses on character-level diff with semantic cleanup. [`imara-diff`](https://crates.io/crates/imara-diff) is a high-performance implementation used by the [Helix editor](https://helix-editor.com/), where the histogram variant outperforms Myers by 10-100x in their benchmarks.

But building it once - even a naive version without the linear space optimization - changes how you read `git diff` output. You stop seeing magic and start seeing a shortest path through an edit graph. A grid, some diagonals, and a greedy search that terminates at the right corner.
