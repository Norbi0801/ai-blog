+++
title = "Building a semantic search engine in Rust"
date = 2025-07-05
description = "End-to-end semantic search in Rust with Qdrant: embeddings, indexing, filtered queries, hybrid dense plus sparse retrieval, and a fair comparison with Elasticsearch."

[taxonomies]
tags = ["rust", "vector-search", "qdrant", "search"]
+++

The canonical "Elasticsearch works great until it doesn't" moment happens when a user types `how do I cancel my subscription` and your index returns the FAQ entry titled `billing adjustments`. The user meant the same thing. Your inverted index did not. Every token was different.

Semantic search fixes this by comparing meaning instead of words. You embed the documents, embed the query, and rank by vector similarity. The implementation is small. The interesting part is what you wire around it: filtering, hybrid scoring, and the pipeline that keeps the index in sync with your data.

This post is the honest version of building that pipeline in Rust, end to end, on top of Qdrant. If you have not read [HNSW: how vector search actually works](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood) and [Cosine similarity](/blog/cosine-similarity-the-math-behind-semantic-search), those cover the index and the metric that sit under this. I will use them as black boxes.

<!-- more -->

## What the engine has to do

Four things, in order of how often they run:

1. **Query**: take a string, return ranked documents in tens of milliseconds.
2. **Filter**: restrict results by metadata (tenant, language, date range, tag).
3. **Index**: upsert new and changed documents in the background.
4. **Re-embed**: when the embedding model changes, rebuild from scratch.

The architecture is always the same two boxes plus a vector database:

```
      index path                        query path
 docs ─▶ embed ─▶ upsert ─▶ Qdrant ◀─ search ◀─ embed ◀─ query string
                             │
                          filters
```

The index path runs on writes. The query path runs on every request. They share the embedding client and the collection, and nothing else. Keep them in separate modules so their latency budgets cannot infect each other.

## Dependencies

```toml
[dependencies]
qdrant-client = "1.13"
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
```

`qdrant-client` is Qdrant's official Rust SDK. It speaks gRPC under the hood, which matters because upserts of a few thousand points per batch are routine and HTTP would be the bottleneck. `reqwest` is for the embedding API client.

## The embedding client

Semantic search begins with turning a string into a vector. You can self-host a model (see [Local LLMs with llama.cpp and Rust](/blog/local-llms-with-llama-cpp-and-rust) for the general idea) or call an API. The API is 50 lines of Rust and has no GPU to own. I will use OpenAI's `text-embedding-3-small` at 1536 dims because it is cheap, fast, and good enough for most text workloads.

```rust
use serde::{Deserialize, Serialize};

pub struct EmbeddingClient {
    http: reqwest::Client,
    api_key: String,
    model: String,
}

#[derive(Serialize)]
struct EmbedRequest<'a> {
    model: &'a str,
    input: &'a [String],
}

#[derive(Deserialize)]
struct EmbedResponse {
    data: Vec<EmbedItem>,
}

#[derive(Deserialize)]
struct EmbedItem {
    embedding: Vec<f32>,
    index: usize,
}

impl EmbeddingClient {
    pub fn new(api_key: String) -> Self {
        Self {
            http: reqwest::Client::new(),
            api_key,
            model: "text-embedding-3-small".to_string(),
        }
    }

    pub async fn embed(&self, inputs: &[String]) -> anyhow::Result<Vec<Vec<f32>>> {
        let req = EmbedRequest { model: &self.model, input: inputs };
        let resp: EmbedResponse = self
            .http
            .post("https://api.openai.com/v1/embeddings")
            .bearer_auth(&self.api_key)
            .json(&req)
            .send()
            .await?
            .error_for_status()?
            .json()
            .await?;

        // the API preserves order via the `index` field; sort to be safe
        let mut items = resp.data;
        items.sort_by_key(|i| i.index);
        Ok(items.into_iter().map(|i| i.embedding).collect())
    }
}
```

Two details that burn you in production. The API returns items with an `index` field, and while the ordering has been stable in practice, sorting by it is the cheap insurance. And OpenAI has a per-request token cap around 300K tokens; batch inputs in groups of 64 to 256 strings and cap each string's length before you send.

## The Qdrant collection

A collection is Qdrant's equivalent of an Elasticsearch index. It holds points, where each point is a vector plus a payload (JSON metadata) plus an id.

```rust
use qdrant_client::qdrant::{
    CreateCollectionBuilder, Distance, VectorParamsBuilder,
};
use qdrant_client::Qdrant;

pub async fn ensure_collection(client: &Qdrant, name: &str, dim: u64) -> anyhow::Result<()> {
    if client.collection_exists(name).await? {
        return Ok(());
    }
    client
        .create_collection(
            CreateCollectionBuilder::new(name)
                .vectors_config(VectorParamsBuilder::new(dim, Distance::Cosine)),
        )
        .await?;
    Ok(())
}
```

`Distance::Cosine` tells Qdrant to normalize vectors at ingest and rank by dot product. It is the right default for almost every embedding model shipped in the last three years; they are all trained against a cosine loss. Do not use `Distance::Euclid` with a cosine-trained model unless you like debugging sadness.

On 10M documents at 1536 dims you are looking at ~60 GB of raw vectors plus another few GB of HNSW graph. Qdrant supports on-disk storage and scalar quantization (int8, 4x smaller) that are worth turning on once the index stops fitting in RAM; the defaults are fine until then.

## The index pipeline

A document in my example is a blog post: an id, a title, a body, a language, and some tags. The payload is what survives into query results and what filters can reference.

```rust
use qdrant_client::qdrant::{PointStruct, UpsertPointsBuilder};
use serde_json::json;

#[derive(Clone)]
pub struct Doc {
    pub id: u64,
    pub title: String,
    pub body: String,
    pub lang: String,
    pub tags: Vec<String>,
    pub published_at: i64, // unix seconds
}

pub async fn index_docs(
    qdrant: &Qdrant,
    embedder: &EmbeddingClient,
    collection: &str,
    docs: &[Doc],
) -> anyhow::Result<()> {
    let inputs: Vec<String> =
        docs.iter().map(|d| format!("{}\n\n{}", d.title, d.body)).collect();
    let vectors = embedder.embed(&inputs).await?;

    let points: Vec<PointStruct> = docs
        .iter()
        .zip(vectors)
        .map(|(d, v)| {
            let payload = json!({
                "title": d.title,
                "body": d.body,
                "lang": d.lang,
                "tags": d.tags,
                "published_at": d.published_at,
            });
            PointStruct::new(d.id, v, payload.try_into().unwrap())
        })
        .collect();

    qdrant
        .upsert_points(UpsertPointsBuilder::new(collection, points).wait(true))
        .await?;
    Ok(())
}
```

Three decisions in this function are worth defending.

First, I concatenate title and body before embedding. Splitting into two vectors per doc is tempting, but unless you separately score and combine them, a single representation is simpler and usually better. If the body is long, chunk it with [Text chunking strategies for RAG](/blog/text-chunking-strategies-for-rag) and store one point per chunk with a `doc_id` link.

Second, I store the full body in the payload. This trades storage for query latency: you can return snippets without a second hop to your primary database. For docs larger than a few kilobytes, store only the snippet or a foreign key and hydrate on the way out.

Third, `wait(true)` blocks until the upsert is durably indexed. Drop it for high-throughput bulk loads, keep it when you need read-your-writes.

## The query pipeline

A single query costs one embedding call plus one vector search. The code is short; the details in the filter are what earn the paycheck.

```rust
use qdrant_client::qdrant::{
    Condition, Filter, ScoredPoint, SearchPointsBuilder, with_payload_selector::SelectorOptions,
    WithPayloadSelector,
};

pub struct Query<'a> {
    pub text: &'a str,
    pub lang: Option<&'a str>,
    pub tags_any: &'a [String],
    pub min_published_at: Option<i64>,
    pub top_k: u64,
}

pub async fn search(
    qdrant: &Qdrant,
    embedder: &EmbeddingClient,
    collection: &str,
    q: Query<'_>,
) -> anyhow::Result<Vec<ScoredPoint>> {
    let query_vec = embedder.embed(&[q.text.to_string()]).await?.remove(0);

    let mut conditions: Vec<Condition> = Vec::new();
    if let Some(lang) = q.lang {
        conditions.push(Condition::matches("lang", lang.to_string()));
    }
    if !q.tags_any.is_empty() {
        conditions.push(Condition::matches("tags", q.tags_any.to_vec()));
    }
    if let Some(ts) = q.min_published_at {
        conditions.push(Condition::range("published_at", ts..));
    }

    let mut builder = SearchPointsBuilder::new(collection, query_vec, q.top_k)
        .with_payload(WithPayloadSelector {
            selector_options: Some(SelectorOptions::Enable(true)),
        })
        .score_threshold(0.30);
    if !conditions.is_empty() {
        builder = builder.filter(Filter::must(conditions));
    }

    let resp = qdrant.search_points(builder).await?;
    Ok(resp.result)
}
```

A few things to say about the score and the filter.

**Scores.** With `Distance::Cosine` and normalized vectors, Qdrant returns a similarity score in `[-1, 1]`. Scores above `0.8` are usually strong matches. Scores under `0.25` are almost always noise. The `score_threshold(0.30)` line is a cheap way to drop garbage results before you show them. The exact cutoff is model-specific; measure on your data.

**Filters apply before the vector search.** This is a Qdrant specialty. It uses payload indexes plus filterable HNSW so that the ANN walk only visits points that match the filter. Elasticsearch's kNN support post-filters, which means a very selective filter can leave you with zero results from a top-k that did not include them. If you filter on tenant or language, create a payload index:

```rust
use qdrant_client::qdrant::{CreateFieldIndexCollectionBuilder, FieldType};

qdrant
    .create_field_index(CreateFieldIndexCollectionBuilder::new(
        collection, "lang", FieldType::Keyword,
    ))
    .await?;
```

No index means Qdrant falls back to a linear payload scan per candidate, which erases the speed advantage.

## Hybrid search: dense plus sparse

Dense vectors are great at meaning. They are bad at exact strings: product SKUs, error codes, people's names. A user searching `E0507` should not get fuzzy matches on "error handling patterns". The fix is hybrid search: run a dense query and a sparse keyword query side by side, then fuse the results.

Qdrant supports sparse vectors natively. The sparse vector is a map `{token_id: weight}` that encodes either raw term frequencies or a BM25-like score. Most people use the `fastembed` crate to generate them client-side; server-side BM25 is on Qdrant's roadmap but not default as of 1.12.

The fusion is where the real decision is. Reciprocal Rank Fusion (RRF) is the boring, well-tested choice:

```rust
use std::collections::HashMap;

pub fn rrf(result_sets: &[&[ScoredPoint]], k: f32) -> Vec<(u64, f32)> {
    let mut fused: HashMap<u64, f32> = HashMap::new();
    for set in result_sets {
        for (rank, pt) in set.iter().enumerate() {
            let id = match pt.id.as_ref().and_then(|i| i.point_id_options.as_ref()) {
                Some(qdrant_client::qdrant::point_id::PointIdOptions::Num(n)) => *n,
                _ => continue,
            };
            *fused.entry(id).or_insert(0.0) += 1.0 / (k + rank as f32 + 1.0);
        }
    }
    let mut out: Vec<(u64, f32)> = fused.into_iter().collect();
    out.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    out
}
```

`k = 60` is the value from the original RRF paper and the one production systems keep landing on. The reason it works is that it uses ranks, not scores, which means you can fuse two completely incomparable systems (cosine similarity and BM25) without calibrating them. You pay for that with information loss: a document ranked #1 by a huge margin looks the same as one that barely edged out #2.

Alternative: weighted score fusion if both systems produce comparable, normalized scores. If you have two dense queries, that works. Across dense and sparse, RRF wins nine times out of ten.

Qdrant 1.10+ supports a native `query` API that runs both the dense and sparse searches server-side with a fusion strategy. It is the cleanest option if you are on a recent version; the client-side RRF above is the portable fallback.

## How this compares to Elasticsearch full-text

Elasticsearch has supported kNN since 8.0 and the story is surprisingly good, so the comparison is worth laying out honestly.

**Ranking quality.** For queries where the user uses synonyms, reformulations, or natural language, semantic beats BM25 by a large margin. For queries where the user knows the exact term (SKU, function name, brand), BM25 still wins. This is not opinion; MTEB and BEIR leaderboards separate these query types and the numbers are consistent.

**Recall under filters.** Qdrant's filterable HNSW is the single biggest practical win. Elasticsearch's kNN post-filtering means a 0.1%-selective filter (one tenant out of a thousand) can silently return nothing. You can mitigate with `num_candidates` tuning, but you are paying the cost of a brute-force scan for correctness.

**Operational shape.** Elasticsearch is one system to run that does both text and vectors. Qdrant is a second system next to a primary data store. If your team already runs Elasticsearch and your queries skew keyword-heavy, adding a dense field to an existing index is less moving parts than a new database. If you are starting fresh and search is the main feature, a purpose-built vector database will reward you with tighter tail latency and better filter behavior.

**Cost and memory.** Elasticsearch's HNSW implementation is Lucene's, which is competent but not as tunable as Qdrant's. Qdrant's quantization story (scalar, binary, product) lets you push 10x more vectors per GB of RAM at a recall cost you get to choose. Elasticsearch added int8 quantization in 8.13; the ergonomics are worse but the option exists.

**Hybrid search.** Elasticsearch's native `rrf` retriever was added in 8.14. It works. Qdrant's fusion API is a little newer and arguably cleaner because dense and sparse vectors are first-class objects rather than fields on a document.

My summary: use Elasticsearch if text search is already the majority of your use case and you can live with the filter caveat. Use Qdrant if semantic quality, filter correctness, or per-vector cost are the metrics your boss will ask about.

## Practical notes that will save you a weekend

**Embed in batches, not per-document.** Every major embedding API charges per token, not per request, and the fixed overhead per HTTP call dominates at 1 input. Batches of 64 to 256 are the sweet spot.

**Version your embeddings.** When you change models, every existing point has to be re-embedded. Store the model name in the payload and refuse to serve queries with a mismatch. Aliased collections let you swap them atomically.

**Do not re-embed on every read.** Cache the query vector for common queries. If you have a typeahead layer that fires on every keystroke, pre-compute the top-N popular queries once a day.

**Measure recall against BM25, not against yourself.** The user's bar is "better than what they had before". If your previous search was Postgres full-text or Algolia, build a golden query set of 100 to 500 queries with expected documents, and evaluate both systems with `recall@10` and `MRR`. The numbers almost always move the argument from "feels smarter" to "is smarter, here is the graph".

**Keep the index path async and idempotent.** An upsert with the same id is a safe no-op-ish operation. Retries are cheap. If your document source emits change events, a 30-line worker that reads a queue, embeds, and upserts is all you need.

The whole engine, including the embedding client, the index function, the query function, the filter builder, and the RRF fusion, lands around 200 lines of Rust. The hard work was done by the embedding model and by Qdrant's HNSW; what you are writing is the glue that turns those into a product users will actually search.
