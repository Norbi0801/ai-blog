+++
title = "HNSW: how vector search actually works under the hood"
date = 2025-01-18
description = "A walk through the HNSW graph, from the probabilistic layer assignment to a working Rust implementation, with brute force and IVF as reference points."

[taxonomies]
tags = ["rust", "algorithms", "vector-search", "ai"]
+++

If you embed a million photos with CLIP, you get a million 512-dimensional vectors. A user types "golden retriever on a beach" and you need to find the closest vector in the index in under 20 ms. Brute force is O(n*d) per query, which on a million vectors and 512 dims is roughly half a billion floating point multiplies. You can do it. It will not be fast.

Almost every production vector database (Qdrant, Weaviate, Milvus, pgvector 0.5+, Vespa, Elasticsearch's knn search) defaults to the same index: HNSW. Hierarchical Navigable Small World graphs. The algorithm is from a 2016 paper by Yury Malkov and Dmitry Yashunin, and it has held up because the core idea is simple and the implementation details are unusually well tuned.

If you have not read [Multimodal AI: processing images, audio and text together](/blog/multimodal-ai-images-audio-text-together), that post explains why the vectors exist in the first place. This one is about what happens after you have a pile of vectors and need to search them.

<!-- more -->

## The problem, precisely

You have N vectors in R^d. A query vector arrives. You want the k nearest neighbors under some distance metric, usually L2 or cosine. "Nearest" is well defined. The catch is that you want it fast, in high dimensions, for large N.

High dimensionality is the hard part. The curse of dimensionality has a concrete effect on search: as d grows, every point in the dataset tends to be roughly the same distance from the query, so pruning (the trick that makes kd-trees and ball trees work in low d) stops helping. For d > ~20, classic space-partitioning structures degrade to near brute force.

Approximate nearest neighbor (ANN) search trades a tiny bit of recall for a large speedup. In practice you want 95-99% recall at 10-100x the speed of brute force, and HNSW delivers that on typical workloads.

## Brute force, as a baseline

Before the graph, here is what you are trying to beat:

```rust
pub fn brute_force_knn(
    query: &[f32],
    dataset: &[Vec<f32>],
    k: usize,
) -> Vec<(usize, f32)> {
    let mut heap: std::collections::BinaryHeap<(OrderedFloat, usize)> =
        std::collections::BinaryHeap::new();

    for (i, v) in dataset.iter().enumerate() {
        let d = l2_squared(query, v);
        if heap.len() < k {
            heap.push((OrderedFloat(d), i));
        } else if d < heap.peek().unwrap().0 .0 {
            heap.pop();
            heap.push((OrderedFloat(d), i));
        }
    }

    let mut result: Vec<_> = heap.into_iter().map(|(d, i)| (i, d.0)).collect();
    result.sort_by(|a, b| a.1.partial_cmp(&b.1).unwrap());
    result
}

fn l2_squared(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b).map(|(x, y)| (x - y).powi(2)).sum()
}
```

On a modern CPU with AVX2 you can do roughly 20-40 GFLOPS of dot products. A million 512-dim vectors is 1024 MB of memory traffic, and that is the real bottleneck, not the arithmetic. SIMD helps, but you are still linear in N. For N = 10M this is tens of milliseconds per query on a good server and hundreds on a laptop.

The entire point of HNSW is to get to log(N) time.

## The graph, layer by layer

HNSW is two ideas stapled together.

**Navigable Small World (NSW).** Build a graph where each vector is a node, connected to a small number of nearest neighbors. At search time, start anywhere, greedily hop to the neighbor closest to the query, stop when no neighbor is closer. This works because the graph has the small world property: short paths exist between any two nodes via "long-range" edges.

**Hierarchical.** NSW alone gets stuck in local minima and the greedy walk takes too many hops. HNSW stacks NSW graphs into layers, with exponentially fewer nodes at each higher layer. Top layer: ~log(N) nodes with long edges. Bottom layer: every node, with short edges. Search starts at the top, follows the greedy path to the closest node in that layer, then descends one level and repeats.

It looks like this:

```
Layer 2:     A ------------------- F        (2 nodes, long jumps)
Layer 1:     A ----- C ----- E --- F        (4 nodes, medium jumps)
Layer 0:   A-B-C-D-E-F-G-H-I-J-K-L-M-N     (all N nodes, short jumps)
```

A node's maximum layer is chosen once, at insert time, from a geometric distribution. The math:

```
level = floor(-ln(uniform_0_1) * mL)
```

where mL is a normalization constant, usually set to 1/ln(M) for a target branching factor of M. The result: each node appears in layer 0, and with probability 1/M in layer 1, 1/M^2 in layer 2, and so on. This is the single line that gives the whole structure its log(N) property. Neat.

## Why log(N)?

Think of it as a skip list generalized to higher dimensions.

- Layer L has roughly N / M^L nodes.
- At each layer the greedy walk visits O(1) nodes on average (if the graph is well-built, the next best neighbor is almost always reachable in a few steps).
- The number of layers is O(log N) because the geometric distribution caps out there.

So total work is O(log N) hops, each hop costs one distance computation per visited neighbor, each node has at most M neighbors. Search is O(M * log N) distance computations. For N = 10M and M = 16, that is around 400 distance checks instead of 10 million. The 25000x reduction in work is what makes sub-millisecond ANN feasible.

This is a bound on the expected work under standard assumptions (uniform data, well-formed graph). On adversarial data it can degrade, which is why production implementations also tune the `ef` parameters below.

## The parameters, and what they actually do

Three knobs define HNSW behavior, and they all have a clear physical meaning:

- **M**: max neighbors per node at layers > 0. Typical: 8 to 64. Layer 0 gets 2*M because most searches end there. Larger M: higher recall, more memory (`2*M * 4 bytes * N` for the edge list alone), slower build.
- **efConstruction**: size of the dynamic candidate list used during insertion. Larger: better graph quality, slower build. Typical: 100-400.
- **efSearch**: size of the dynamic candidate list used during query. Larger: higher recall, slower queries. Typical: 50-500, tuned per workload. This is the knob you actually tune after deploy.

Memory math for a real index: 10M vectors at 768 dims with M=32 is:
- Vectors: 10M * 768 * 4 = 30 GB
- Graph edges: 10M * 64 * 4 = 2.5 GB
- Plus per-node layer metadata, around 80-120 MB.

The graph is cheap compared to the vectors themselves. This is usually a surprise to people expecting the index to be the expensive part.

## The search algorithm, step by step

Here is the search procedure, faithful to the paper, annotated.

```rust
use std::cmp::Reverse;
use std::collections::{BinaryHeap, HashSet};

// Min-heap by distance: the closest candidate pops first.
// Max-heap by distance: the worst accepted result pops first.
// We use both, which is why BinaryHeap and Reverse<T> show up together.

struct Hnsw {
    vectors: Vec<Vec<f32>>,
    // layers[l][node_id] = neighbor ids at layer l
    layers: Vec<Vec<Vec<usize>>>,
    entry_point: usize,
    top_layer: usize,
    m: usize,
    ef_construction: usize,
}

impl Hnsw {
    fn search_layer(
        &self,
        query: &[f32],
        entry_points: &[usize],
        ef: usize,
        layer: usize,
    ) -> Vec<(f32, usize)> {
        let mut visited: HashSet<usize> = entry_points.iter().copied().collect();

        // candidates: min-heap, closest first
        let mut candidates: BinaryHeap<Reverse<(OrderedFloat, usize)>> = BinaryHeap::new();
        // results: max-heap, worst first, capped at ef
        let mut results: BinaryHeap<(OrderedFloat, usize)> = BinaryHeap::new();

        for &ep in entry_points {
            let d = l2_squared(query, &self.vectors[ep]);
            candidates.push(Reverse((OrderedFloat(d), ep)));
            results.push((OrderedFloat(d), ep));
        }

        while let Some(Reverse((OrderedFloat(cd), c))) = candidates.pop() {
            let worst = results.peek().map(|x| x.0 .0).unwrap_or(f32::INFINITY);
            if cd > worst {
                break; // closest remaining candidate is worse than our worst result
            }

            for &neighbor in &self.layers[layer][c] {
                if !visited.insert(neighbor) {
                    continue;
                }
                let d = l2_squared(query, &self.vectors[neighbor]);
                let worst = results.peek().map(|x| x.0 .0).unwrap_or(f32::INFINITY);
                if results.len() < ef || d < worst {
                    candidates.push(Reverse((OrderedFloat(d), neighbor)));
                    results.push((OrderedFloat(d), neighbor));
                    if results.len() > ef {
                        results.pop();
                    }
                }
            }
        }

        let mut out: Vec<_> = results.into_iter().map(|(d, i)| (d.0, i)).collect();
        out.sort_by(|a, b| a.0.partial_cmp(&b.0).unwrap());
        out
    }

    pub fn search(&self, query: &[f32], k: usize, ef_search: usize) -> Vec<(usize, f32)> {
        let mut ep = vec![self.entry_point];
        // descend from top layer to layer 1 with ef=1 (greedy)
        for layer in (1..=self.top_layer).rev() {
            let found = self.search_layer(query, &ep, 1, layer);
            ep = vec![found[0].1];
        }
        // layer 0 with full ef
        let found = self.search_layer(query, &ep, ef_search.max(k), 0);
        found.into_iter().take(k).map(|(d, i)| (i, d)).collect()
    }
}
```

Two details worth noting. The upper layers use `ef = 1`, which is pure greedy descent. The bottom layer uses the tuned `ef_search`, which is where recall actually gets paid for. And the early-exit `if cd > worst { break; }` is what keeps the wall-clock bounded: the moment the best candidate is already worse than the worst accepted result, the frontier cannot improve and we stop.

`OrderedFloat` is a thin wrapper around `f32` that implements `Ord`; the real crate is [`ordered-float`](https://crates.io/crates/ordered-float), or you can roll your own.

## Build: inserting a new vector

Insertion is where HNSW earns its memory footprint. For each new vector:

1. Sample a level from the geometric distribution.
2. From the entry point, greedily find the nearest node at every layer above the new node's level.
3. At every layer from the new node's level down to 0, run `search_layer` with `ef = ef_construction`, pick the M best candidates, and wire them as neighbors. Also update those neighbors' neighbor lists (bidirectional edges).
4. If any neighbor now has more than M edges (2M at layer 0), prune.

Pruning is not "drop the farthest edge." The paper's `select_neighbors_heuristic` picks edges that preserve graph diversity: an edge to a close neighbor is only kept if there is no other kept neighbor that is both closer to the candidate and to the node. This avoids the pathological case of all M edges pointing in the same direction, which kills the small-world property.

```rust
fn select_neighbors_heuristic(
    &self,
    query: &[f32],
    candidates: Vec<(f32, usize)>,
    m: usize,
) -> Vec<usize> {
    let mut sorted = candidates;
    sorted.sort_by(|a, b| a.0.partial_cmp(&b.0).unwrap());

    let mut result: Vec<(f32, usize)> = Vec::with_capacity(m);
    for (dist_to_query, cand) in sorted {
        if result.len() >= m {
            break;
        }
        let cand_vec = &self.vectors[cand];
        // keep only if cand is closer to query than to any already-kept neighbor
        let good = result.iter().all(|(_, kept)| {
            l2_squared(cand_vec, &self.vectors[*kept]) > dist_to_query
        });
        if good {
            result.push((dist_to_query, cand));
        }
    }
    result.into_iter().map(|(_, i)| i).collect()
}
```

That single function is the difference between a graph with great recall and one that looks great on paper but dies in production. Every mature HNSW implementation ([hnswlib](https://github.com/nmslib/hnswlib/blob/master/hnswlib/hnswalg.h), [instant-distance](https://github.com/instant-labs/instant-distance), Qdrant's [`segment::index::hnsw_index`](https://github.com/qdrant/qdrant/tree/master/lib/segment/src/index/hnsw_index)) uses this heuristic by default.

Build cost: inserting N vectors is O(N * log N * ef_construction * M). On a laptop, a million vectors of 128 dims with `M=16, efConstruction=200` takes 1-3 minutes with `hnswlib`. Production builds of 100M+ vectors run for hours, and most databases do them on background threads while the old index still serves queries.

## IVF, for comparison

Inverted File Index (IVF) is the other major ANN family and the one used by FAISS's `IndexIVFPQ`. It works differently:

1. Cluster the dataset into `nlist` cells with k-means (typical: sqrt(N) cells).
2. Each vector is assigned to its nearest centroid and stored in that cell's list.
3. At query time, find the `nprobe` nearest centroids and scan only those cells.

Tradeoffs vs HNSW:

- **Build time**: IVF needs k-means training, which is O(N * nlist * iterations). Faster than HNSW for huge N once you have centroids.
- **Memory**: IVF is lighter. No neighbor graph, just cell assignments plus centroids.
- **Recall/speed curve**: HNSW dominates at high recall (>95%). IVF holds up better under heavy compression (IVFPQ quantizes vectors to 8 bytes; HNSW usually keeps raw f32).
- **Updates**: HNSW supports streaming inserts cleanly. IVF prefers batch rebuilds because cell assignments drift as data changes.
- **Disk**: IVF is a good fit for SSD-resident indexes. Only the probed cells need to be read. HNSW wants to live in RAM because the graph walk is pointer-chasing.

Most production systems end up with HNSW for in-memory low-latency paths and IVF (or IVFPQ, or the newer ScaNN) for multi-billion scale where RAM is the binding constraint. You do not have to pick one forever.

## Recall, tuning, and what actually moves the needle

Three practical things that matter more than the algorithm choice:

**Measure recall, not just speed.** Compute brute-force top-k on a held-out query set, compare to your ANN top-k, report recall@k. Every vector database has this as a benchmark target. Without it, "HNSW is fast" is meaningless - it is fast in exchange for recall you did not measure.

**Tune efSearch online.** It is cheap to change: no rebuild required, it only affects query time. Start at 2*k, double it until recall saturates, then back off. Most workloads end up between 64 and 256.

**Normalize your vectors if you use cosine.** HNSW does not care about the metric internally, but cosine similarity on non-normalized vectors is just a dot product, and the distance monotonicity breaks. Unit-norm your embeddings at ingest. CLIP already does this; not every model does.

**Think about deletes.** HNSW has no native delete. All implementations use tombstones and lazy cleanup. If your data churns faster than you rebuild, you will see recall decay. pgvector 0.5+ handles this transparently, hnswlib does not.

## What I skipped

The paper has a few more details worth knowing about once you go deeper: the exact entry point promotion when the new vector has a higher layer than the current top, the `extend_candidates` flag in the pruning heuristic, and the product quantization variants (HNSWSQ, HNSWPQ) that trade a few points of recall for a 4-8x memory reduction. hnswlib's C++ source is readable and 1500 lines; it is the best reference implementation to study.

The original paper is [Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs (Malkov & Yashunin, 2018)](https://arxiv.org/abs/1603.09320). The Rust crate [`instant-distance`](https://crates.io/crates/instant-distance) by the same team behind Quinn is a good starting point if you want HNSW in a real project without linking to a C++ library.

HNSW is one of those algorithms that rewards reading the source. The loops look innocent, but the invariants (bidirectional edges, heuristic pruning, geometric layer assignment) are what keep the graph navigable as it grows. Once it clicks, the rest of the vector-database stack stops feeling like magic.
