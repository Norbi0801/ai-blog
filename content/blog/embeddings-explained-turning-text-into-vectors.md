+++
title = "Embeddings explained: turning text into vectors"
date = 2025-03-29
description = "What an embedding actually is, why a 384-dimensional vector can capture meaning, and a 50-line semantic search demo in Rust using fastembed."

[taxonomies]
tags = ["ai", "embeddings", "rust", "nlp"]
+++

You hand a model the string `"red sneakers under fifty bucks"` and it gives you back 384 floating-point numbers. That is an embedding. Those 384 numbers, when compared against the embeddings of every product description in your catalog, will surface "crimson trainers $39.99" before they surface "red sports car for sale". No keyword overlap. No synonyms list. Just two vectors and a dot product.

This post is about how that works. What an embedding is, where the vector comes from, why the dimensionality is what it is, how you compare them, and how to do all of it from Rust without a Python runtime in your stack.

<!-- more -->

## The one-sentence definition

An embedding is a learned function that maps a piece of input (a token, a word, a sentence, a document, an image) to a fixed-size vector of real numbers, such that semantically similar inputs land at nearby points and dissimilar inputs land far apart.

That is the whole idea. Everything below is implementation details.

The "learned" part is doing a lot of work. Nobody hand-designed the dimensions. A neural network was trained on billions of examples with an objective that punished it for putting unrelated things close together, and the resulting weights produce vectors that just happen to behave this way. You should treat the individual axes as opaque. Dimension 217 is not "fruitiness" or "anger" or anything human-readable. Only the geometry as a whole matters.

## Why vectors at all

Computers are very good at three operations on dense float arrays: multiply, add, compare. They are very bad at the operation "is this sentence about the same thing as that sentence". Embeddings convert the second problem into the first.

Three concrete things this unlocks:

- **Semantic search**: rank documents by how similar their embedding is to the query embedding, instead of by how many words overlap. This is how every modern RAG pipeline works under the hood.
- **Clustering and deduplication**: group similar documents without labels. K-means on embeddings of customer support tickets will surface your top problem categories without anyone tagging anything.
- **Recommendation and similar-item lookup**: "users who looked at this product also looked at..." reduces to a nearest-neighbour query in embedding space.

The classic alternative was lexical retrieval (TF-IDF, BM25). It still works, it is still cheap, and for queries with rare keywords it often beats embeddings outright. The two are complementary, which is why "hybrid search" is a thing. But if you have ever watched a user type `how do I cancel my subscription` and your inverted index serve them an article titled `billing adjustments`, you know exactly what embeddings fix.

## From token IDs to a sentence vector

A modern text embedding model is, almost always, a transformer. The pipeline looks like this:

```
"red sneakers"  ─▶ tokenizer ─▶ [101, 2417, 17073, 102]
                                          │
                                          ▼
                                  embedding lookup table
                                          │
                                          ▼
                                  N transformer layers
                                          │
                                          ▼
                                  pooling (mean or [CLS])
                                          │
                                          ▼
                                  optional L2 normalize
                                          │
                                          ▼
                                  [0.0123, -0.0876, ..., 0.0451]   (length 384)
```

Step by step:

1. **Tokenize** the input string into integer IDs. If you have not seen this before, the [tokenization deep dive](/blog/tokenization-deep-dive-bpe-wordpiece-sentencepiece) covers BPE and friends. Each model ships with its own tokenizer.
2. **Look up** each token ID in a learned embedding table, typically of shape `[vocab_size, hidden_dim]`. This gives you one vector per token. These per-token vectors are also "embeddings", but they are an internal artifact, not what you usually mean by "the embedding of a sentence".
3. **Run the transformer** stack. Self-attention mixes information across positions so each token's vector ends up contextualized by its neighbours. The [attention mechanism post](/blog/understanding-attention-mechanism-the-core-of-transformers) covers exactly what happens in those layers.
4. **Pool** the per-token output vectors down to a single vector for the whole input. The two common strategies are mean pooling (average over all token positions, masked by the attention mask) and CLS pooling (take the vector at position 0, which corresponds to a special `[CLS]` token added during training). Sentence-BERT style models use mean pooling, BERT-style models use CLS, and BGE-family models use CLS.
5. **L2-normalize** so the vector has unit length. Most modern embedding models output already-normalized vectors. This makes cosine similarity reduce to a plain dot product.

The tokenizer and the pooling strategy are part of the model. If you mean-pool a model that was trained for CLS pooling, you will get vectors that look reasonable but score worse than they should on every benchmark. Always use the model's official inference code or read its config.

## Why 384, 768, 1536, 3072

Embedding dimensionality is a hyperparameter the model author picks. Common values you will see in 2026:

| Model | Dimensions | Notes |
|---|---|---|
| `all-MiniLM-L6-v2` | 384 | Sentence-transformers default. 22 MB. CPU-friendly. |
| `all-mpnet-base-v2` | 768 | Stronger quality, ~110 MB. |
| `BAAI/bge-base-en-v1.5` | 768 | Top of the MTEB leaderboard for its size. |
| `BAAI/bge-m3` | 1024 | Multilingual, multi-functional. |
| `nomic-embed-text-v1.5` | 768 | Matryoshka, can be truncated to 64-768. |
| OpenAI `text-embedding-3-small` | 1536 | Default, cheap. Supports `dimensions` param. |
| OpenAI `text-embedding-3-large` | 3072 | Top quality, 5x the cost. |
| Cohere `embed-v3` | 1024 | Multilingual. |

Three things determine the choice:

**Capacity**. More dimensions can encode more distinctions. The marginal gain flattens fast. Going from 128 to 384 helps a lot. Going from 1536 to 3072 helps barely-measurably on most tasks.

**Compute**. The output dimension shows up linearly in storage cost (one float per dim per document) and in similarity-computation cost. A million-doc corpus stored as 1536-dim float32 is 6 GB; the same corpus at 384-dim is 1.5 GB. That difference matters for index memory, network transfer, and per-query latency.

**Distillation pressure**. Smaller embedding spaces are forced to be more efficient per axis. The 384-dim MiniLM models are competitive with much larger ones because they are distilled from bigger teachers and the contrastive loss has nowhere to hide redundant information.

The Matryoshka trick from Nomic and OpenAI is genuinely useful: train so that the first K dimensions of the vector are themselves a valid embedding for any K from a small list. You can store full vectors but query with a 256-dim prefix, then re-rank candidates with the full vector. This gives you the storage cost of a small model with most of the quality of a big one.

## Cosine similarity in one paragraph

Once you have two vectors `a` and `b`, you want a number that says how related their inputs are. The defacto answer is cosine similarity:

```
cos(a, b) = (a . b) / (||a|| * ||b||)
```

Bounded in [-1, 1], 1 means same direction, 0 means orthogonal, -1 means opposite. For L2-normalized vectors the denominator is 1 and you can drop it: cosine becomes raw dot product, which is what every vector database does in the hot path. I wrote a [whole post on cosine similarity](/blog/cosine-similarity-the-math-behind-semantic-search) including SIMD implementations and how it relates to Euclidean distance, so I will not relitigate it here.

In practice: normalize once at ingest, store the unit vectors, dot product at query time. That is the entire similarity layer.

## Visualizing what embeddings learn

You have 384 numbers. You cannot plot 384 dimensions. Two reduction techniques dominate:

**PCA** finds the linear projections that preserve the most variance. Cheap, deterministic, and useful as a first look. It captures global structure but tends to smear local clusters into blobs because the variance directions are fixed for the whole dataset.

**t-SNE** and **UMAP** preserve local neighbourhood structure. Points that were near each other in the original space stay near each other in 2D, at the cost of distorting global distances. They are nonlinear and stochastic. You will see crisp clusters that lexical methods would never produce: news headlines in `umap-learn` will separate into "tech earnings", "weather events", "sports scores" with no labels and no supervision.

Practical advice: PCA first to confirm "do my embeddings have signal at all". UMAP for talks and dashboards. Never trust the absolute distances in a 2D projection of 384-dim data. The reduction always lies a little.

## The Rust side: candle and ort

Two real options for running embedding models in Rust without a Python interpreter.

**candle** is HuggingFace's pure-Rust ML framework, started in 2023. It implements its own tensor library, supports CPU/CUDA/Metal, and ships model definitions for common architectures (BERT, Llama, Mistral, Whisper). For embeddings, the `candle-transformers` crate has BERT and JinaBERT wired up. The advantage is no native dependencies beyond the Rust toolchain. The cost is that you implement the model in Rust, so support for new architectures lags.

**ort** is the Rust binding for Microsoft's ONNX Runtime. You convert your model to ONNX once (HuggingFace's `optimum` does this in three lines of Python) and then run it from Rust against a single C++ runtime that supports CUDA, DirectML, CoreML, and CPU. The advantage is that ONNX is the lingua franca of ML deployment, so almost every model has an ONNX export already on the Hub. The cost is a native dependency and slightly more setup per platform.

For most embedding workloads I reach for ort, because the models you care about (MiniLM, BGE, multilingual-e5) all have prebuilt ONNX versions and inference is consistently faster than candle by 20-40% per token on CPU. If you want zero native deps and you are running on Apple Silicon, candle is the better fit.

There is also `fastembed-rs`, which wraps ort, hardcodes a list of well-known embedding models, and handles tokenizer + ONNX download from HuggingFace. For "I just want sentence vectors", it is the shortest path.

## A semantic search demo in 50 lines

Here is the smallest useful thing: ingest a list of strings, accept a query, return the top three most similar. No vector database, no indexing tricks, just a flat scan with cosine. It scales to a few hundred thousand documents on a laptop before you need HNSW.

`Cargo.toml`:

```toml
[package]
name = "tiny-search"
edition = "2021"

[dependencies]
fastembed = "4"
anyhow = "1"
```

`src/main.rs`:

```rust
use anyhow::Result;
use fastembed::{EmbeddingModel, InitOptions, TextEmbedding};

fn cosine(a: &[f32], b: &[f32]) -> f32 {
    let dot: f32 = a.iter().zip(b).map(|(x, y)| x * y).sum();
    let na: f32 = a.iter().map(|x| x * x).sum::<f32>().sqrt();
    let nb: f32 = b.iter().map(|x| x * x).sum::<f32>().sqrt();
    dot / (na * nb + 1e-12)
}

fn main() -> Result<()> {
    let model = TextEmbedding::try_new(
        InitOptions::new(EmbeddingModel::AllMiniLML6V2),
    )?;

    let docs: Vec<String> = vec![
        "Crimson trainers, mesh upper, $39.99".into(),
        "Vintage red Mustang for sale, low miles".into(),
        "Lightweight running shoes in burgundy".into(),
        "Office chair with lumbar support".into(),
        "Womens scarlet sneakers, size 8, on sale".into(),
        "How to dye old shoes red at home".into(),
    ];

    let doc_vecs = model.embed(docs.clone(), None)?;
    let query = "red sneakers under fifty bucks";
    let q_vec = model.embed(vec![query.to_string()], None)?.remove(0);

    let mut scored: Vec<(f32, &str)> = doc_vecs
        .iter()
        .zip(docs.iter())
        .map(|(v, d)| (cosine(&q_vec, v), d.as_str()))
        .collect();
    scored.sort_by(|a, b| b.0.partial_cmp(&a.0).unwrap());

    println!("query: {query}\n");
    for (score, doc) in scored.iter().take(3) {
        println!("{:.3}  {doc}", score);
    }
    Ok(())
}
```

Build and run:

```
$ cargo run --release
query: red sneakers under fifty bucks

0.624  Womens scarlet sneakers, size 8, on sale
0.587  Crimson trainers, mesh upper, $39.99
0.491  Lightweight running shoes in burgundy
```

The "Mustang" listing, despite containing the word "red", scores lower than three results that share zero keywords with the query. That is what semantic similarity buys you.

The first call downloads the MiniLM ONNX file and the tokenizer to a cache directory (a few hundred MB). Subsequent runs reuse the cache. On an M2 laptop, embedding the six documents plus the query takes about 30 ms after warmup.

This program is doing four things you should mentally separate, because each becomes a different system component when you scale it up:

1. **Loading the model**: cold start cost, share the `TextEmbedding` instance across requests in a server.
2. **Embedding the documents**: batched, ingest-time, async. In production this is a job that runs once per document version.
3. **Embedding the query**: synchronous, on every request, latency-critical. Keep it under 30 ms.
4. **Comparing**: a flat O(N*d) scan. Fine for thousands of vectors, painful at millions. Replace with [HNSW](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood) or a real vector database when you outgrow it.

## What embeddings will not do

Three failure modes worth naming:

**Negation is fragile**. "I love sushi" and "I do not love sushi" often score high cosine similarity. Embedding models look at content words and pooling tends to wash out function words. If your task hinges on negation, sentiment analysis on top of embeddings is more reliable than retrieval against a fixed set of phrases.

**Numerical and identifier matching is bad**. "$39.99" and "$3999" are nearly identical to a tokenizer that treats both as digit subtokens, but completely different as values. Same for SKUs, IDs, and version numbers. Use lexical search for those fields and combine the scores.

**Cross-lingual works iff the model was trained for it**. English-only models will happily embed Polish text into something that looks like a vector but does not cluster sensibly. Multilingual models (`bge-m3`, `multilingual-e5`, `paraphrase-multilingual-MiniLM`) handle this.

The pattern across all three: embeddings encode "what is this text about, in general", and they are very good at that. They are bad at exact matching, careful logic, and anything where small textual differences should produce large semantic differences.

## What to take away

An embedding is a transformer's pooled output, normalized, treated as a point in high-dimensional space. The geometry of that space encodes meaning. You compare points with cosine similarity, you visualize them with UMAP, and you serve them in production with a vector index when you have enough of them to need one.

Pick a model from the MTEB leaderboard that fits your latency and dimensionality budget. Run it from Rust with `fastembed` if you want the shortest path, with `ort` if you want control, with `candle` if you do not want native deps. Wire cosine similarity to your search endpoint and you have semantic retrieval. Everything else (HNSW, hybrid search, reranking) is incremental gain on top.
