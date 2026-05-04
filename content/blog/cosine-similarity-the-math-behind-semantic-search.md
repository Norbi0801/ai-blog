+++
title = "Cosine similarity: the math behind semantic search"
date = 2025-01-21
description = "Why cosine is the default metric for comparing embeddings, the geometric intuition behind it, and a Rust implementation from naive to SIMD-optimized."

[taxonomies]
tags = ["rust", "ai", "vector-search", "simd"]
+++

Every vector database, every RAG pipeline, every "related articles" feature I have seen in the last three years ends at the same line of code: a cosine similarity between two embedding vectors. It is the final arbiter of "does this document match the query". The formula is three symbols and a square root, but what those symbols do to high-dimensional space is worth a post.

If you have not read [Multimodal AI: processing images, audio and text together](/blog/multimodal-ai-images-audio-text-together), that post covers why embeddings exist and how models project images and text into the same space. This one is about what you do with the vectors once you have them.

<!-- more -->

## The formula

Given two vectors `a` and `b` in R^d:

```
cos(a, b) = (a . b) / (||a|| * ||b||)
```

The numerator is the dot product, `sum(a_i * b_i)`. The denominator is the product of the Euclidean norms, `sqrt(sum(a_i^2)) * sqrt(sum(b_i^2))`. The result is the cosine of the angle between the two vectors, which, being a cosine, is bounded in `[-1, 1]`:

- `1` means the vectors point in exactly the same direction.
- `0` means they are orthogonal (no linear relationship).
- `-1` means they point in exactly opposite directions.

For most modern embedding models (CLIP, SBERT, `text-embedding-3-small`, `bge-m3`) the output is already constrained so that cosine similarity is the metric the model was trained against. The loss function told the model "make matching pairs have a high cosine and non-matching pairs have a low cosine". You are, literally, querying the loss.

## Geometric intuition

Imagine two arrows starting from the origin. Cosine similarity does not care how long the arrows are. It cares which way they point.

That is the key property. Consider two documents, one short and one long, that both talk about "golden retrievers on beaches". If you bag-of-words them, the long one will have a much larger norm because it repeats the words more. Their dot product will also be larger. But the angle between the two vectors, in the subspace where "dog words" and "beach words" live, will be small. Cosine nails it. Euclidean distance and raw dot product will both be misled by length.

Another way to see it: cosine similarity is the dot product you would get if you first normalized both vectors to unit length. The denominator strips magnitude out of the answer, leaving only direction.

## Why direction, not magnitude, for embeddings

Neural embedding models are trained with contrastive or triplet losses. A typical CLIP training step looks like:

```
similarity = (img_vec @ text_vec.T) / temperature
loss = cross_entropy(similarity, labels)
```

where `img_vec` and `text_vec` are often L2-normalized before the dot product. The model is optimized to put "matching" (image, caption) pairs at small angles and non-matching pairs at large angles. Magnitude is almost a free parameter that the model never bothers to use meaningfully.

This is why you should be suspicious of magnitudes coming out of an embedding model. Sometimes they correlate weakly with confidence, sometimes with token length, sometimes they are just noise. Treating them as meaningful leads to ranking bugs that are hard to explain. Strip them out with cosine.

## Cosine vs Euclidean vs dot product

These three are the standard metrics in every vector DB (Qdrant, Milvus, pgvector, Weaviate, FAISS). They are mathematically related but behave differently.

**Dot product**: `a . b`. Fast, no division, no square roots. Use when your vectors are already normalized, because in that case `a . b = cos(a, b)`. Many production pipelines pre-normalize once at ingest and then use raw dot product at query time. This is the fastest option and gives identical results.

**Euclidean (L2) distance**: `sqrt(sum((a_i - b_i)^2))`. Cares about both direction and magnitude. For normalized vectors there is an exact relationship:

```
||a - b||^2 = ||a||^2 + ||b||^2 - 2 * (a . b)
            = 2 - 2 * cos(a, b)   (when ||a|| = ||b|| = 1)
```

So on unit vectors, L2 and cosine produce the same ranking. The distance values differ but the order of nearest neighbors is the same. This is why HNSW implementations can offer a "cosine" mode that is really just L2 on normalized vectors under the hood.

**Cosine similarity**: `(a . b) / (||a|| * ||b||)`. The safe default when you do not control normalization, or when you mix data from different models. Slower than dot product because of the two norms, but robust.

When to reach for which:

- **Text or image embeddings from a modern model**: cosine, or pre-normalize and use dot product. The model was trained for this.
- **Raw feature vectors where magnitude means something** (e.g. TF-IDF sums, physical measurements, counts): Euclidean or maybe even Manhattan (L1). Cosine will throw away information you need.
- **Recommendation systems with explicit user-item signals**: sometimes dot product unnormalized, because popular items should win by magnitude. This is model-dependent.

The rule I follow: if the model was trained with cosine loss, use cosine at query time. Match the training objective or accept that you are approximating it.

## A naive Rust implementation

No dependencies. Just `std`.

```rust
pub fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len(), "vectors must have equal length");

    let mut dot = 0.0f32;
    let mut norm_a = 0.0f32;
    let mut norm_b = 0.0f32;

    for i in 0..a.len() {
        dot += a[i] * b[i];
        norm_a += a[i] * a[i];
        norm_b += b[i] * b[i];
    }

    let denom = norm_a.sqrt() * norm_b.sqrt();
    if denom == 0.0 {
        return 0.0; // zero vector: undefined, return 0 rather than NaN
    }

    dot / denom
}
```

Three things to notice.

1. **One loop, three accumulators.** Doing three separate passes (`iter().zip().map().sum()` for each) is cleaner but slower because of cache behavior. The compiler usually fuses them, but not always. Writing it as one loop makes the intent explicit.
2. **Zero-vector check.** If either input is all zeros, the denominator is zero and you get `NaN`. A `NaN` inside a binary heap during nearest-neighbor search will corrupt the ordering. Return `0.0` and move on.
3. **`f32`, not `f64`.** Embeddings are stored as `f32` almost universally. Doubling the precision doubles the memory bandwidth and the embedding model was trained in `f32` or `bf16` anyway. Fake precision.

This version runs at somewhere between 1 and 3 GB/s of memory bandwidth on a modern x86 core, which is bottlenecked by the scalar floating-point pipeline, not by memory. We can do much better.

## Pre-normalization: the trick that changes everything

If you are going to compute cosine similarity against the same vector many times (which is exactly what happens in a search index), you are recomputing its norm over and over. Normalize once, store the unit-length version, and then cosine becomes a plain dot product.

```rust
pub fn normalize(v: &mut [f32]) {
    let norm: f32 = v.iter().map(|x| x * x).sum::<f32>().sqrt();
    if norm > 0.0 {
        let inv = 1.0 / norm;
        for x in v.iter_mut() {
            *x *= inv;
        }
    }
}

pub fn dot(a: &[f32], b: &[f32]) -> f32 {
    debug_assert_eq!(a.len(), b.len());
    a.iter().zip(b).map(|(x, y)| x * y).sum()
}
```

Now your search loop is:

```rust
for (i, v) in dataset.iter().enumerate() {
    let sim = dot(query, v); // both are unit vectors
    // ...
}
```

Every production cosine-based index I have looked at does this. OpenAI's embeddings API returns pre-normalized vectors so you can skip the normalization step entirely (the docs say this, though it is easy to miss). Pre-normalization saves two `sqrt` calls per comparison, which on a batch of a million comparisons is about 50 ms of pure transcendental math removed from your query path.

## Batch similarity: one query, many candidates

The shape you almost always want at query time: one query vector, many candidate vectors, return all similarities.

```rust
pub fn batch_cosine(query: &[f32], candidates: &[Vec<f32>]) -> Vec<f32> {
    // Precompute query norm once
    let q_norm: f32 = query.iter().map(|x| x * x).sum::<f32>().sqrt();

    candidates
        .iter()
        .map(|c| {
            let mut dot = 0.0f32;
            let mut c_norm = 0.0f32;
            for i in 0..query.len() {
                dot += query[i] * c[i];
                c_norm += c[i] * c[i];
            }
            let denom = q_norm * c_norm.sqrt();
            if denom == 0.0 { 0.0 } else { dot / denom }
        })
        .collect()
}
```

If the candidates are pre-normalized, it collapses to:

```rust
pub fn batch_dot(query: &[f32], candidates: &[Vec<f32>]) -> Vec<f32> {
    candidates
        .iter()
        .map(|c| {
            let mut s = 0.0f32;
            for i in 0..query.len() {
                s += query[i] * c[i];
            }
            s
        })
        .collect()
}
```

Memory layout matters enormously here. `Vec<Vec<f32>>` scatters your vectors across the heap: each candidate is a separate allocation, and each iteration chases a pointer before it can touch any floats. Switching to a single contiguous buffer of `N * d` floats with manual indexing speeds this up by 2-3x in my benchmarks, purely because of prefetching and page locality.

```rust
pub struct FlatIndex {
    data: Vec<f32>, // row-major: data[i * d .. (i+1) * d] is the i-th vector
    dim: usize,
    count: usize,
}

impl FlatIndex {
    pub fn row(&self, i: usize) -> &[f32] {
        &self.data[i * self.dim..(i + 1) * self.dim]
    }

    pub fn query(&self, q: &[f32]) -> Vec<f32> {
        (0..self.count).map(|i| dot(q, self.row(i))).collect()
    }
}
```

Simple. Fast. One allocation at build time.

## SIMD: doing eight floats per instruction

The scalar dot product does one multiply-add per cycle. AVX2 does eight `f32` multiply-adds per cycle (256-bit SIMD with `vfmadd231ps`). AVX-512 does sixteen. That is where the actual speedup lives.

You have three options in Rust.

**Option 1: trust the compiler.** LLVM auto-vectorizes simple loops reliably if you help it:

```rust
#[inline]
pub fn dot_autovec(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len());
    let mut acc = 0.0f32;
    // Work on 8-wide chunks to encourage the vectorizer
    for (x, y) in a.iter().zip(b.iter()) {
        acc += x * y;
    }
    acc
}
```

Compile with `RUSTFLAGS="-C target-cpu=native -C opt-level=3"` and `cargo rustc --release -- --emit=asm` to see the generated code. On my x86_64 box with AVX2, this produces a tight loop of `vfmadd231ps` instructions. Check [godbolt.rs](https://godbolt.rs) with `-C target-feature=+avx2` to see it in action.

The catch: reduction order. IEEE float addition is not associative, so the compiler will only vectorize if you allow it to reorder. Use `-C opt-level=3` with `-C target-cpu=native`, or explicitly use `std::arch` intrinsics if you need guarantees.

**Option 2: `std::simd` (portable SIMD, nightly as of 2026).** This is the clean path when it stabilizes:

```rust
#![feature(portable_simd)]
use std::simd::{f32x8, num::SimdFloat};

pub fn dot_simd(a: &[f32], b: &[f32]) -> f32 {
    let chunks = a.chunks_exact(8).zip(b.chunks_exact(8));
    let mut acc = f32x8::splat(0.0);

    for (ac, bc) in chunks {
        let va = f32x8::from_slice(ac);
        let vb = f32x8::from_slice(bc);
        acc += va * vb;
    }

    // Handle the tail
    let tail = a.len() - (a.len() % 8);
    let mut scalar_acc = acc.reduce_sum();
    for i in tail..a.len() {
        scalar_acc += a[i] * b[i];
    }

    scalar_acc
}
```

`f32x8` compiles to AVX2 on x86_64, NEON (two passes of four lanes) on aarch64, and scalar fallback where SIMD is unavailable. `portable_simd` has been cooking in nightly since 2021; the RFC is [here](https://github.com/rust-lang/portable-simd).

**Option 3: `std::arch` intrinsics.** When you need the last 10%, you go direct:

```rust
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2,fma")]
pub unsafe fn dot_avx2(a: &[f32], b: &[f32]) -> f32 {
    use std::arch::x86_64::*;
    debug_assert_eq!(a.len(), b.len());
    debug_assert!(a.len() % 8 == 0);

    let mut acc = _mm256_setzero_ps();
    for i in (0..a.len()).step_by(8) {
        let va = _mm256_loadu_ps(a.as_ptr().add(i));
        let vb = _mm256_loadu_ps(b.as_ptr().add(i));
        acc = _mm256_fmadd_ps(va, vb, acc);
    }

    // Horizontal sum of 8 lanes
    let sum128 = _mm_add_ps(
        _mm256_castps256_ps128(acc),
        _mm256_extractf128_ps(acc, 1),
    );
    let sum64 = _mm_add_ps(sum128, _mm_movehl_ps(sum128, sum128));
    let sum32 = _mm_add_ss(sum64, _mm_shuffle_ps(sum64, sum64, 1));
    _mm_cvtss_f32(sum32)
}
```

This is the shape of code you will find inside [simsimd](https://github.com/ashvardanian/SimSIMD) and inside FAISS's inner kernels. It is unsafe, it is platform-specific, and it is about 6-8x faster than the naive version on 768-dim vectors.

If you want the real numbers: on my Ryzen 7 7700X the naive loop does about 1.2 GB/s, autovec does about 7 GB/s, and hand-written AVX2 tops out around 12 GB/s, limited by L2 bandwidth when the working set fits in cache. For a 10M x 768 index that does not fit in L2, everyone converges at roughly DRAM bandwidth divided by the size of one vector.

## Gotchas I have hit

**NaN propagation.** Zero vectors, and vectors with `inf` components (rare, but happens when upstream preprocessing goes wrong), will poison your heap. Always defend against `denom == 0.0`.

**Mixed normalization.** If half your index is normalized and half is not, your similarities are meaningless. Normalize eagerly, on insert, and assert it in debug builds: `debug_assert!((norm(v) - 1.0).abs() < 1e-5)`.

**f16/bf16 inputs.** Some newer models emit half-precision embeddings to save memory. You can compute dot products directly in half precision on modern hardware, but reduction should be in `f32` to avoid catastrophic cancellation. Do not accumulate in `f16`.

**Cosine is not a metric.** It is a similarity, not a distance. It does not satisfy the triangle inequality. If you need a proper metric (e.g. for some geometric data structures), convert with `angular_distance = acos(cos_sim) / pi` or use `1 - cos_sim` and accept that the triangle inequality is only approximate.

**Batch size vs cache.** When you process a large batch of queries against the same index, blocking the computation so that both the query block and the index block fit in L2 turns this into a BLAS-style matmul problem. That is what `ndarray`, `faer-rs`, and `matrixmultiply` optimize for. At some point you stop writing your own kernel and call `sgemm`.

## Where this takes you

Once cosine similarity is fast and boring, the interesting problem becomes: how do you avoid computing it N times per query? That is the job of an approximate nearest neighbor index. The distance kernel sits at the innermost loop of every one of them. HNSW, IVF, ScaNN, they all ultimately come down to "evaluate the kernel on the candidates we could not prune".

If you want to see what comes next, [HNSW: how vector search actually works under the hood](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood) picks up here and builds the graph index. Everything in that post uses the kernel we just wrote.
