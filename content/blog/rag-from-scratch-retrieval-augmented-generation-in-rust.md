+++
title = "RAG from scratch: retrieval augmented generation in Rust"
date = 2025-12-14
description = "What RAG actually is under the hood, the five-step pipeline, and a minimal 150-line Rust program that wires it up with Qdrant and OpenAI embeddings."

[taxonomies]
tags = ["rust", "rag", "ai", "embeddings"]
+++

The shortest honest definition of retrieval augmented generation is "don't retrain the model, just give it a cheat sheet." You keep a pile of documents somewhere, at query time you pull the few that look relevant, paste them into the prompt, and let the language model answer using that as context. The model never learned your internal pricing page. It never had to. It read the part you handed it and answered from there.

That is it. Everything you read about RAG - hybrid search, re-ranking, parent-child chunking, HyDE, graph RAG - is ornament on that one idea. Before any of those matter, you should be able to sit down and build the naive version end to end in under two hundred lines. This post does that in Rust.

<!-- more -->

If you want the larger frame of when RAG is the right answer at all, I wrote about that in [Fine-tuning vs RAG vs prompt engineering](/blog/fine-tuning-vs-rag-vs-prompt-engineering). The production-shaped version of what follows, with PDF parsing, citations, and evaluation, is [Building a document Q&A system with RAG in Rust](/blog/building-a-document-qa-system-with-rag-in-rust). This post is the thing in between: a stripped-down tutorial that shows what each piece of the pipeline is actually doing.

## The pipeline, drawn honestly

```
INGEST (run once per corpus change)
    documents
        │
        ▼
    chunking ──────▶ smaller strings (few hundred tokens each)
        │
        ▼
    embedding API ─▶ one float vector per chunk (e.g. 1536 dims)
        │
        ▼
    vector DB ─────▶ (vector, original text, metadata) rows

QUERY (run once per user question)
    question
        │
        ▼
    embedding API ─▶ one float vector for the question
        │
        ▼
    vector DB ─────▶ top-K most similar chunks by cosine distance
        │
        ▼
    LLM ──▶ prompt: "using this context, answer the question"
```

Five moving parts: chunking, embedding, storage, similarity search, and a final prompt. None of them is conceptually hard. The reason production RAG is fiddly is that every one of these five steps has three ways to be quietly wrong, and the bugs compound. Knowing what each step is doing in isolation is the difference between debugging RAG in hours and debugging it in weeks.

## What an embedding actually is

An embedding model takes a string and returns a fixed-length vector of floats. For `text-embedding-3-small`, that vector has 1536 dimensions. For `text-embedding-3-large`, 3072. Each dimension is meaningless on its own; what matters is that semantically similar strings map to vectors that are close in that 1536-dimensional space.

Concretely, if you embed "how do I reset my password" and "I forgot my login credentials," the two vectors will have a cosine similarity around 0.7 to 0.9. Embed "how do I reset my password" and "banana bread recipe" and you will see cosine similarity around 0.1 to 0.3. The model was trained on enough web text that this clustering is fairly robust across paraphrases, languages (for multilingual models), and topic boundaries.

The vectors are already L2-normalized by OpenAI, meaning their length is 1.0. This has a useful consequence: cosine similarity between them equals their dot product, and both are in `[-1, 1]`. If you ever need to compute similarity manually:

```rust
fn cosine(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b).map(|(x, y)| x * y).sum()
}
```

That is the whole math. The [cosine similarity post](/blog/cosine-similarity-the-math-behind-semantic-search) goes deeper on why this measure and not L2 or Manhattan distance; for our purposes, it is one fused multiply-add per dimension per comparison. On modern CPUs with AVX, a vector DB can do tens of millions of these per second per core, which is why you can search a million-chunk corpus in a few milliseconds.

## Step 1: a corpus

For a tutorial we do not need to parse PDFs. We need a list of strings that contain facts the base LLM does not know. I will make some up:

```rust
const CORPUS: &[&str] = &[
    "Acme refund policy: customers on the Enterprise plan can request \
     a full refund within 45 days of purchase. Pro plan customers have a \
     14-day window, and Starter plan refunds are not offered.",
    "Acme supports SSO through Okta, Azure AD, and Google Workspace. SAML \
     2.0 configuration requires the Enterprise plan. SCIM user provisioning \
     is available on Enterprise only.",
    "API rate limits: 60 requests per minute on Starter, 600 on Pro, and \
     custom limits on Enterprise. Rate limits are enforced per organization, \
     not per user.",
    "The Acme CLI is installed via `cargo install acme-cli`. It requires \
     Rust 1.78 or newer. The binary is also published as a static musl build \
     on the GitHub releases page for users without a Rust toolchain.",
    "Data residency: Acme stores customer data in eu-central-1 by default. \
     US customers can opt into us-east-1. Enterprise customers can request \
     a dedicated VPC deployment.",
];
```

Five chunks, already sized reasonably. In a real system your corpus is a folder of markdown or a table of wiki pages, and you have to split it before embedding. More on that in a moment.

## Step 2: embed the corpus

One HTTP call to OpenAI. The API takes up to 2048 inputs per call so batching is free up to that limit.

```rust
use anyhow::Result;
use reqwest::Client;
use serde::{Deserialize, Serialize};

const EMBED_MODEL: &str = "text-embedding-3-small";
const EMBED_DIMS: u64 = 1536;

#[derive(Serialize)]
struct EmbedReq<'a> {
    input: &'a [&'a str],
    model: &'a str,
}

#[derive(Deserialize)]
struct EmbedResp {
    data: Vec<EmbedItem>,
}

#[derive(Deserialize)]
struct EmbedItem {
    embedding: Vec<f32>,
    index: usize,
}

async fn embed(client: &Client, api_key: &str, inputs: &[&str]) -> Result<Vec<Vec<f32>>> {
    let resp: EmbedResp = client
        .post("https://api.openai.com/v1/embeddings")
        .bearer_auth(api_key)
        .json(&EmbedReq { input: inputs, model: EMBED_MODEL })
        .send()
        .await?
        .error_for_status()?
        .json()
        .await?;

    let mut out = vec![Vec::new(); inputs.len()];
    for item in resp.data {
        out[item.index] = item.embedding;
    }
    Ok(out)
}
```

Two things to notice. The API returns items with an `index` field, not in input order - always read the index field and place each vector back in the right slot, otherwise you silently scramble your corpus. The `error_for_status()` call converts 4xx and 5xx into `reqwest::Error` before attempting JSON deserialization, which turns "the API rate-limited me" into a useful error instead of a serde "missing field `data`" confusion.

Cost note: `text-embedding-3-small` is $0.02 per million tokens as of mid-2026. Five short chunks is a few hundred tokens total, so ingesting this corpus costs a thousandth of a cent. Embedding millions of tokens of documentation is still under a dollar. The expensive part of RAG is the LLM call at answer time, not the embedding.

## Step 3: store in Qdrant

Qdrant is a vector database written in Rust, which means its Rust client is first-class. Spin it up locally with one command:

```bash
docker run -p 6334:6334 qdrant/qdrant
```

Then in code, create a collection and upsert points. A "point" in Qdrant terminology is one row: an ID, a vector, and an arbitrary JSON payload.

```rust
use qdrant_client::Qdrant;
use qdrant_client::qdrant::{
    CreateCollectionBuilder, Distance, PointStruct,
    SearchPointsBuilder, UpsertPointsBuilder, VectorParamsBuilder,
    Value as QValue,
};
use std::collections::HashMap;

async fn init_store(client: &Qdrant, collection: &str) -> Result<()> {
    if !client.collection_exists(collection).await? {
        client.create_collection(
            CreateCollectionBuilder::new(collection)
                .vectors_config(VectorParamsBuilder::new(EMBED_DIMS, Distance::Cosine)),
        ).await?;
    }
    Ok(())
}

async fn upsert_corpus(
    client: &Qdrant,
    collection: &str,
    texts: &[&str],
    vectors: Vec<Vec<f32>>,
) -> Result<()> {
    let points: Vec<PointStruct> = texts.iter().zip(vectors).enumerate()
        .map(|(i, (text, vec))| {
            let mut payload: HashMap<String, QValue> = HashMap::new();
            payload.insert("text".into(), text.to_string().into());
            PointStruct::new(i as u64, vec, payload)
        })
        .collect();

    client.upsert_points(
        UpsertPointsBuilder::new(collection, points).wait(true),
    ).await?;
    Ok(())
}
```

I am using `i as u64` as the point ID, which is fine for a demo. In a real system use UUIDv5 over a hash of the chunk content so re-ingesting an unchanged chunk is an idempotent upsert instead of a duplicate. The document Q&A post covers that pattern.

`Distance::Cosine` is the one you want for OpenAI embeddings. The vectors are already normalized so cosine and dot product give identical rankings, but "cosine" is the right name to write down because future-you will read the schema and know what metric the scores mean.

Under the hood Qdrant indexes these with HNSW, a hierarchical graph where each node points to its approximate nearest neighbors at multiple scales. When you search, it walks from a coarse top layer down to the fine-grained base layer in log time. The full story is in the [HNSW post](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood).

## Step 4: search

Embed the query, call `search_points`, get back the top K with their payloads attached.

```rust
#[derive(Debug)]
struct Hit {
    score: f32,
    text: String,
}

async fn search(
    client: &Qdrant,
    collection: &str,
    query_vec: Vec<f32>,
    k: u64,
) -> Result<Vec<Hit>> {
    let resp = client.search_points(
        SearchPointsBuilder::new(collection, query_vec, k)
            .with_payload(true),
    ).await?;
    Ok(resp.result.into_iter().map(|p| Hit {
        score: p.score,
        text: p.payload.get("text")
            .and_then(|v| v.as_str())
            .unwrap_or("")
            .to_string(),
    }).collect())
}
```

Score is in `[-1, 1]` for cosine. In practice for OpenAI embeddings on natural-language queries, relevant chunks score around 0.4 to 0.7, and anything below 0.3 is usually noise. Those thresholds are model-specific - never hard-code a cutoff without first printing scores from a known-good query.

## Step 5: generate

Pull the top-K hits, format a prompt that gives the LLM the context and the instruction to use it, parse the answer.

```rust
#[derive(Serialize)]
struct ChatReq<'a> {
    model: &'a str,
    messages: Vec<ChatMsg<'a>>,
    temperature: f32,
}

#[derive(Serialize)]
struct ChatMsg<'a> { role: &'a str, content: &'a str }

#[derive(Deserialize)]
struct ChatResp { choices: Vec<Choice> }
#[derive(Deserialize)]
struct Choice { message: RespMsg }
#[derive(Deserialize)]
struct RespMsg { content: String }

async fn answer(client: &Client, api_key: &str, question: &str, hits: &[Hit])
    -> Result<String>
{
    let context = hits.iter().enumerate()
        .map(|(i, h)| format!("[{}] {}", i + 1, h.text))
        .collect::<Vec<_>>()
        .join("\n\n");

    let system = "Answer the question using only the CONTEXT. \
                  If the context does not contain the answer, say so.";
    let user = format!("CONTEXT:\n{}\n\nQUESTION: {}", context, question);

    let resp: ChatResp = client
        .post("https://api.openai.com/v1/chat/completions")
        .bearer_auth(api_key)
        .json(&ChatReq {
            model: "gpt-4o-mini",
            messages: vec![
                ChatMsg { role: "system", content: system },
                ChatMsg { role: "user", content: &user },
            ],
            temperature: 0.0,
        })
        .send().await?.error_for_status()?
        .json().await?;

    Ok(resp.choices.into_iter().next()
        .map(|c| c.message.content).unwrap_or_default())
}
```

Three choices worth naming. `temperature: 0.0` because we want the model to prefer what is in the context over whatever it thinks it knows from training. The "only the CONTEXT" language plus "if the context does not contain the answer, say so" is the single most important sentence in any RAG system - it is what separates a pipeline that refuses to hallucinate from one that happily invents numbers when retrieval misses. And `gpt-4o-mini` because the interesting work happened before we got here; the final model just has to read a few sentences and summarize.

## Wiring it together

```rust
#[tokio::main]
async fn main() -> Result<()> {
    let api_key = std::env::var("OPENAI_API_KEY")?;
    let http = Client::new();
    let qdrant = Qdrant::from_url("http://localhost:6334").build()?;
    let collection = "demo";

    init_store(&qdrant, collection).await?;
    let vectors = embed(&http, &api_key, CORPUS).await?;
    upsert_corpus(&qdrant, collection, CORPUS, vectors).await?;

    let question = "Can Starter plan customers get a refund?";
    let qvec = embed(&http, &api_key, &[question]).await?.pop().unwrap();
    let hits = search(&qdrant, collection, qvec, 3).await?;

    println!("Top hits:");
    for h in &hits {
        println!("  {:.3}  {}", h.score, &h.text[..h.text.len().min(80)]);
    }

    let ans = answer(&http, &api_key, question, &hits).await?;
    println!("\nAnswer: {}", ans);
    Ok(())
}
```

Run this with `OPENAI_API_KEY=... cargo run` and you should see the refund policy chunk retrieved first, followed by an answer like "Starter plan refunds are not offered." The LLM did not need to be trained on Acme's pricing. It just needed to be handed the relevant paragraph.

## What about chunking

The corpus above was pre-split into paragraph-sized pieces because I wrote it that way. For a real corpus you start with documents and have to chop them up. This is the step that disproportionately determines whether your RAG works, and it has its own post: [Text chunking strategies for RAG](/blog/text-chunking-strategies-for-rag). The short version:

- **Tokens, not characters.** English is roughly 4 chars per token for BPE, code is closer to 3, structured data is wildly variable. Budget in tokens using `tiktoken-rs`.
- **500 to 800 tokens per chunk** is the boring right answer for most prose corpora. Small enough that one embedding represents one idea. Big enough that the LLM has context to work with.
- **10-15% overlap between adjacent chunks.** Any less and concepts that straddle a boundary get halved. Any more and you are paying to store near-duplicates.
- **Recursive splitting** that prefers paragraph, then sentence, then word boundaries. The `text-splitter` crate does this correctly with real token counts.
- **Prepend section headings to the embedded text.** An embedding of "Volume discounts start at 100 seats" is ambiguous. An embedding of "Enterprise plan / Billing / Volume discounts start at 100 seats" is specific. Keep the clean version for the LLM, embed the enriched version.

If your retrieval is returning the wrong chunks, swap your embedding model zero times and print twenty random chunks first. The answer is almost always visible in them.

## Memory, bytes, and the cost model

A quick tour of the numbers, because they matter more than the ceremony suggests.

One 1536-dim float32 vector is 6,144 bytes. A million chunks is 6 GB of vectors alone, not counting the HNSW graph (which typically doubles storage) or the payload text. Qdrant supports scalar quantization (int8 per dimension) to cut this to 1.5 GB per million at a small recall cost, and binary quantization for further savings.

The HNSW graph has two main knobs: `m` (neighbors per node, default 16) and `ef_construct` (candidates examined during insertion, default 128). Higher values give better recall at query time but slower insertions. Defaults are fine until you are past a few million vectors.

At query time the cost breakdown is roughly:

- Embedding the question: one call, ~30 tokens, fractions of a cent.
- Qdrant search: a few milliseconds of CPU, measured in microseconds of wall clock.
- LLM answer call: 500-2000 input tokens (the retrieved chunks) plus however long the answer is. This dominates everything else by two orders of magnitude.

If you are worried about RAG infrastructure cost, you are worrying about the wrong thing. The LLM bill is what you budget for.

## When to reach for RAG versus fine-tuning

If you have not read the [full comparison](/blog/fine-tuning-vs-rag-vs-prompt-engineering), the one-paragraph version: RAG solves "the model does not know our stuff," fine-tuning solves "the model does not behave the way we want." You use RAG when the gap is knowledge (docs, product data, recent events), because you can add and remove documents without retraining. You use fine-tuning when the gap is style or format (structured output, specific tone, narrow classification), because prompt engineering gets brittle past a certain prompt length. The serious production systems I have seen eventually stack both: a fine-tuned small model that knows how to behave plus a RAG index that knows the current facts.

If your problem is "the model gets the vocabulary right but cites pricing from 2021," that is a RAG problem. If it is "the model knows our data but refuses to answer in JSON without a 2000-token system prompt," that is a fine-tuning problem. Getting that diagnosis wrong is how teams spend a quarter retraining a model that just needed a vector index.

## What to take away

Five steps. Chunk, embed, store, search, prompt. About 150 lines of Rust to see them end to end. The ornaments - hybrid BM25+dense search, cross-encoder re-rankers, parent-child chunking, HyDE, query rewriting - are all improvements on specific failure modes that show up when you measure a baseline. Build the baseline first. Print the scores. Print the chunks. If the right chunk is in the top-K and the answer is still wrong, it is a prompt problem. If the right chunk is not in the top-K, it is a chunking or embedding problem. If everything looks right and users still complain, it is time to build an eval set, which is the subject of the [document Q&A post](/blog/building-a-document-qa-system-with-rag-in-rust).

The first RAG you build should fit in a single file. The second one should include evaluation. Everything after that is tuning.

## References

- [Qdrant Rust client](https://github.com/qdrant/rust-client) on GitHub
- [Qdrant quickstart](https://qdrant.tech/documentation/quickstart/) for the Docker one-liner
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings) and [pricing](https://openai.com/api/pricing/)
- [`text-splitter`](https://github.com/benbrandt/text-splitter) for recursive chunking with real token counts
- [`tiktoken-rs`](https://github.com/zurawiki/tiktoken-rs) for BPE tokenization in Rust
- [HNSW: Efficient and robust approximate nearest neighbor search](https://arxiv.org/abs/1603.09320), the original paper
