+++
title = "Building a document Q&A system with RAG in Rust"
date = 2025-02-20
description = "End-to-end RAG in Rust: parse PDFs and markdown, chunk, embed, store in Qdrant, retrieve, answer with citations. About 300 lines of honest code plus how to tell if it works."

[taxonomies]
tags = ["rust", "rag", "ai", "qdrant"]
+++

The textbook RAG pipeline is seven boxes on a slide. Parse. Chunk. Embed. Store. Query. Retrieve. Generate. The distance between that slide and a system that answers a user's question with a correct citation is where most "RAG MVPs" die. Not because the idea is wrong, but because every box has three ways to be subtly broken, and the bugs compound multiplicatively.

This post is the walk-through I wish I had when I built my first production RAG. I am going to wire up the full pipeline in Rust, pick boring defaults at every step, and end with something you can point at a folder of PDFs and markdown files and get cited answers from. And critically, I am going to spend the last section on evaluation, because the difference between "it answered my test question" and "it works" is a measurement you have to design on purpose.

<!-- more -->

If you have not read [Fine-tuning vs RAG vs prompt engineering](/blog/fine-tuning-vs-rag-vs-prompt-engineering), start there for why you would build this at all. For the chunking step I am going to lean on [Text chunking strategies for RAG](/blog/text-chunking-strategies-for-rag) rather than re-derive it here. On the retrieval side, [HNSW: how vector search actually works](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood) and [Cosine similarity](/blog/cosine-similarity-the-math-behind-semantic-search) cover what is happening inside the vector DB.

## Architecture in one diagram

```
                ingest                             query
  PDF ─┐                                   user question
       ├─▶ parse ─▶ chunk ─▶ embed ─▶ ┐        │
  MD  ─┘                               │        ▼
                                       ▼      embed
                                    Qdrant ◀──┤
                                       │      search top-K
                                       ▼        │
                                    top-K chunks│
                                       │        ▼
                                       └─▶ prompt + context ─▶ LLM ─▶ answer + citations
```

The ingest side runs once per document version. The query side runs every request. Keep them separate in your code; they have different failure modes and different latency budgets.

The crates I will use:

- `pdf-extract` for PDF text extraction. Not perfect on complex layouts but boringly reliable for text-heavy documents.
- `pulldown-cmark` for walking markdown structure (so headings become metadata).
- `text-splitter` for recursive chunking with real token counts (via `tiktoken-rs`).
- `qdrant-client` for the vector store. Qdrant runs locally in a single Docker container, scales to production, and the Rust client is first-class.
- `reqwest` for calls to the embedding and LLM APIs. I will use OpenAI-compatible endpoints because that keeps this swappable with Azure, local llama.cpp, or Anthropic via a proxy.
- `serde` and `anyhow` because of course.

`Cargo.toml`:

```toml
[dependencies]
anyhow = "1"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", features = ["json"] }
pdf-extract = "0.7"
pulldown-cmark = "0.10"
text-splitter = { version = "0.13", features = ["tiktoken-rs", "markdown"] }
tiktoken-rs = "0.5"
qdrant-client = "1.11"
sha2 = "0.10"
uuid = { version = "1", features = ["v5"] }
```

## Step 1: parse

Extract the raw text of the document along with minimal structural metadata. PDFs give us only a stream of text; markdown gives us headings we can use as `section_path`.

```rust
use anyhow::Result;
use pulldown_cmark::{Event, HeadingLevel, Parser, Tag};
use std::path::Path;

#[derive(Debug, Clone)]
pub struct ParsedDoc {
    pub source_id: String,
    pub source_uri: String,
    pub segments: Vec<Segment>,
}

#[derive(Debug, Clone)]
pub struct Segment {
    pub text: String,
    pub section_path: Vec<String>,
}

pub fn parse_pdf(path: &Path) -> Result<ParsedDoc> {
    let bytes = std::fs::read(path)?;
    let text = pdf_extract::extract_text_from_mem(&bytes)?;
    Ok(ParsedDoc {
        source_id: sha256_hex(path.to_string_lossy().as_bytes()),
        source_uri: format!("file://{}", path.display()),
        segments: vec![Segment { text, section_path: vec![] }],
    })
}

pub fn parse_markdown(path: &Path) -> Result<ParsedDoc> {
    let md = std::fs::read_to_string(path)?;
    let parser = Parser::new(&md);
    let mut segments: Vec<Segment> = Vec::new();
    let mut heading_stack: Vec<(HeadingLevel, String)> = Vec::new();
    let mut buf = String::new();
    let mut in_heading: Option<HeadingLevel> = None;
    let mut heading_text = String::new();

    let flush = |segments: &mut Vec<Segment>,
                 buf: &mut String,
                 stack: &[(HeadingLevel, String)]| {
        let text = buf.trim().to_string();
        if !text.is_empty() {
            segments.push(Segment {
                text,
                section_path: stack.iter().map(|(_, s)| s.clone()).collect(),
            });
        }
        buf.clear();
    };

    for event in parser {
        match event {
            Event::Start(Tag::Heading(level, _, _)) => {
                flush(&mut segments, &mut buf, &heading_stack);
                in_heading = Some(level);
                heading_text.clear();
            }
            Event::End(Tag::Heading(level, _, _)) => {
                heading_stack.retain(|(l, _)| l < &level);
                heading_stack.push((level, heading_text.clone()));
                in_heading = None;
            }
            Event::Text(t) | Event::Code(t) => {
                if in_heading.is_some() {
                    heading_text.push_str(&t);
                } else {
                    buf.push_str(&t);
                    buf.push('\n');
                }
            }
            Event::SoftBreak | Event::HardBreak => buf.push('\n'),
            _ => {}
        }
    }
    flush(&mut segments, &mut buf, &heading_stack);

    Ok(ParsedDoc {
        source_id: sha256_hex(path.to_string_lossy().as_bytes()),
        source_uri: format!("file://{}", path.display()),
        segments,
    })
}

fn sha256_hex(bytes: &[u8]) -> String {
    use sha2::{Digest, Sha256};
    format!("{:x}", Sha256::digest(bytes))
}
```

For PDFs I keep the section path empty. If you want structured PDF parsing, look at [`lopdf`](https://github.com/J-F-Liu/lopdf) to walk the content streams and track font-size changes as heading signals. That is a post of its own. For now, a flat document still retrieves fine.

## Step 2: chunk

Use `text-splitter`'s `MarkdownSplitter` for markdown and `TextSplitter` for everything else. Both can be backed by a real tokenizer so the token budget is honest.

```rust
use text_splitter::{ChunkConfig, MarkdownSplitter, TextSplitter};
use tiktoken_rs::cl100k_base;

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct Chunk {
    pub id: String,
    pub text: String,
    pub embed_text: String,
    pub source_id: String,
    pub source_uri: String,
    pub section_path: Vec<String>,
    pub position: usize,
}

pub fn chunk_doc(doc: &ParsedDoc, max_tokens: usize) -> anyhow::Result<Vec<Chunk>> {
    let tokenizer = cl100k_base()?;
    let cfg = ChunkConfig::new(max_tokens)
        .with_sizer(tokenizer)
        .with_overlap(max_tokens / 8)?; // ~12% sentence-aware overlap
    let md_splitter = MarkdownSplitter::new(cfg.clone());
    let txt_splitter = TextSplitter::new(cfg);

    let mut out = Vec::new();
    let mut position = 0usize;
    for seg in &doc.segments {
        let pieces: Vec<&str> = if seg.section_path.is_empty() {
            txt_splitter.chunks(&seg.text).collect()
        } else {
            md_splitter.chunks(&seg.text).collect()
        };
        for piece in pieces {
            let heading = seg.section_path.join(" / ");
            let embed_text = if heading.is_empty() {
                piece.to_string()
            } else {
                format!("{}\n\n{}", heading, piece)
            };
            let id = sha256_hex(
                format!("{}:{}:{}", doc.source_id, position, piece).as_bytes(),
            );
            out.push(Chunk {
                id,
                text: piece.to_string(),
                embed_text,
                source_id: doc.source_id.clone(),
                source_uri: doc.source_uri.clone(),
                section_path: seg.section_path.clone(),
                position,
            });
            position += 1;
        }
    }
    Ok(out)
}
```

Two moves worth naming. First, `embed_text` prepends the section path so the embedder sees the context ("Billing / Plans / Enterprise / Volume discounts start at 100 seats"), while `text` is the clean version the LLM will see at answer time. Second, deterministic IDs via SHA-256 of `(source_id, position, text)` mean re-ingesting an unchanged document produces the same IDs, so Qdrant treats it as an upsert rather than a duplicate.

Why 12% overlap? Anything above 25% is paying to store near-duplicates, and below 10% concepts that straddle a chunk boundary get chopped. I covered this in more detail in the chunking post.

## Step 3: embed

One HTTP call per batch of texts. The OpenAI embeddings API takes up to 2048 inputs per call, which makes batching nearly free.

```rust
const EMBED_MODEL: &str = "text-embedding-3-small"; // 1536 dims
const EMBED_DIMS: usize = 1536;

pub struct Embedder {
    client: reqwest::Client,
    api_key: String,
    base_url: String,
}

impl Embedder {
    pub fn new(api_key: String, base_url: Option<String>) -> Self {
        Self {
            client: reqwest::Client::new(),
            api_key,
            base_url: base_url.unwrap_or_else(|| "https://api.openai.com/v1".into()),
        }
    }

    pub async fn embed(&self, texts: &[String]) -> anyhow::Result<Vec<Vec<f32>>> {
        #[derive(serde::Serialize)]
        struct Req<'a> { input: &'a [String], model: &'a str }
        #[derive(serde::Deserialize)]
        struct Resp { data: Vec<Item> }
        #[derive(serde::Deserialize)]
        struct Item { embedding: Vec<f32>, index: usize }

        let mut out = vec![Vec::new(); texts.len()];
        for batch in texts.chunks(128) {
            let resp: Resp = self
                .client
                .post(format!("{}/embeddings", self.base_url))
                .bearer_auth(&self.api_key)
                .json(&Req { input: batch, model: EMBED_MODEL })
                .send().await?.error_for_status()?
                .json().await?;
            // indices are local to the batch; compute offset
            let offset = /* track per-batch offset */ out.iter()
                .position(|v| v.is_empty()).unwrap_or(0);
            for item in resp.data {
                out[offset + item.index] = item.embedding;
            }
        }
        Ok(out)
    }
}
```

The `offset` bookkeeping in the real code should track how many items you have already filled; I simplified here for readability. In production, wrap this with a retry (exponential backoff on 429 and 5xx) and a rate limiter. OpenAI's tier-1 embedding rate limit is around 3000 RPM and 1M TPM.

Cost check: `text-embedding-3-small` is $0.02 per million tokens as of mid-2026. A 10,000 page PDF corpus is roughly 5M tokens, so one-time ingest is $0.10. Embedding the user query per request is 30 tokens, fractions of a cent. This is the cheap part of the pipeline.

## Step 4: store

Qdrant's Rust client wraps the gRPC API. Spin up a local instance with `docker run -p 6334:6334 qdrant/qdrant`.

```rust
use qdrant_client::Qdrant;
use qdrant_client::qdrant::{
    CreateCollectionBuilder, Distance, PointStruct, UpsertPointsBuilder,
    VectorParamsBuilder, Value as QValue,
};
use std::collections::HashMap;

pub struct Store {
    client: Qdrant,
    collection: String,
}

impl Store {
    pub async fn new(url: &str, collection: &str) -> anyhow::Result<Self> {
        let client = Qdrant::from_url(url).build()?;
        if !client.collection_exists(collection).await? {
            client.create_collection(
                CreateCollectionBuilder::new(collection)
                    .vectors_config(
                        VectorParamsBuilder::new(EMBED_DIMS as u64, Distance::Cosine),
                    ),
            ).await?;
        }
        Ok(Self { client, collection: collection.to_string() })
    }

    pub async fn upsert(&self, chunks: &[Chunk], vectors: Vec<Vec<f32>>)
        -> anyhow::Result<()>
    {
        let points: Vec<PointStruct> = chunks.iter().zip(vectors)
            .map(|(c, v)| {
                let mut payload: HashMap<String, QValue> = HashMap::new();
                payload.insert("text".into(), c.text.clone().into());
                payload.insert("source_id".into(), c.source_id.clone().into());
                payload.insert("source_uri".into(), c.source_uri.clone().into());
                payload.insert("section_path".into(),
                               c.section_path.join(" / ").into());
                payload.insert("position".into(), (c.position as i64).into());
                PointStruct::new(
                    uuid::Uuid::new_v5(&uuid::Uuid::NAMESPACE_OID, c.id.as_bytes())
                        .to_string(),
                    v, payload,
                )
            })
            .collect();
        self.client.upsert_points(
            UpsertPointsBuilder::new(&self.collection, points).wait(true),
        ).await?;
        Ok(())
    }

    pub async fn search(&self, vector: Vec<f32>, k: u64)
        -> anyhow::Result<Vec<Retrieved>>
    {
        use qdrant_client::qdrant::SearchPointsBuilder;
        let resp = self.client.search_points(
            SearchPointsBuilder::new(&self.collection, vector, k)
                .with_payload(true),
        ).await?;
        Ok(resp.result.into_iter().map(|p| Retrieved {
            score: p.score,
            text: p.payload.get("text").and_then(|v| v.as_str()).unwrap_or("").into(),
            source_uri: p.payload.get("source_uri").and_then(|v| v.as_str())
                .unwrap_or("").into(),
            section_path: p.payload.get("section_path").and_then(|v| v.as_str())
                .unwrap_or("").into(),
        }).collect())
    }
}

#[derive(Debug, Clone)]
pub struct Retrieved {
    pub score: f32,
    pub text: String,
    pub source_uri: String,
    pub section_path: String,
}
```

I use UUIDv5 over the chunk's deterministic hash because Qdrant point IDs must be UUIDs or u64s. The v5 namespace trick keeps them deterministic: same chunk content gives the same UUID every time.

Cosine distance, not L2. OpenAI embeddings are already L2-normalized, so cosine and dot product give the same ranking, but Qdrant is happy with cosine either way.

## Step 5: answer with citations

Retrieve top-K, format a prompt that forces the model to cite, parse citations out of the response.

```rust
pub struct Answerer {
    client: reqwest::Client,
    api_key: String,
    base_url: String,
    model: String,
}

#[derive(Debug)]
pub struct Answer {
    pub text: String,
    pub citations: Vec<Retrieved>,
}

impl Answerer {
    pub async fn answer(&self, question: &str, chunks: &[Retrieved])
        -> anyhow::Result<Answer>
    {
        let context = chunks.iter().enumerate().map(|(i, c)| {
            format!("[{}] source: {} ({})\n{}",
                    i + 1, c.source_uri, c.section_path, c.text)
        }).collect::<Vec<_>>().join("\n\n");

        let system = "You are a precise assistant. Answer only using the \
            CONTEXT below. Cite every factual claim with [1], [2] etc. \
            matching the numbered sources. If the context does not contain \
            the answer, say so plainly. Do not invent sources.";

        let user = format!("CONTEXT:\n{}\n\nQUESTION: {}", context, question);

        #[derive(serde::Serialize)]
        struct Msg<'a> { role: &'a str, content: &'a str }
        #[derive(serde::Serialize)]
        struct Req<'a> { model: &'a str, messages: Vec<Msg<'a>>, temperature: f32 }
        #[derive(serde::Deserialize)]
        struct Resp { choices: Vec<Choice> }
        #[derive(serde::Deserialize)]
        struct Choice { message: RespMsg }
        #[derive(serde::Deserialize)]
        struct RespMsg { content: String }

        let resp: Resp = self.client
            .post(format!("{}/chat/completions", self.base_url))
            .bearer_auth(&self.api_key)
            .json(&Req {
                model: &self.model,
                messages: vec![
                    Msg { role: "system", content: system },
                    Msg { role: "user", content: &user },
                ],
                temperature: 0.0,
            })
            .send().await?.error_for_status()?
            .json().await?;

        let text = resp.choices.into_iter().next()
            .map(|c| c.message.content).unwrap_or_default();

        // Extract which [n] tokens the model actually used
        let used: Vec<usize> = (1..=chunks.len())
            .filter(|n| text.contains(&format!("[{}]", n)))
            .collect();
        let citations: Vec<Retrieved> = used.into_iter()
            .map(|n| chunks[n - 1].clone()).collect();

        Ok(Answer { text, citations })
    }
}
```

Three things to notice. `temperature: 0.0` because we want the model to prefer what is in the context over its training prior. The system prompt explicitly says "only using the context" and "if the context does not contain the answer, say so" - this single sentence is the difference between a RAG that refuses to hallucinate and one that happily makes things up when retrieval fails. The citation extraction is a naive `[n]` pattern match; in production I would ask the model to emit structured output (JSON with `answer` and `cited_chunk_indices`) and parse that instead.

## The main loop

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let api_key = std::env::var("OPENAI_API_KEY")?;
    let embedder = Embedder::new(api_key.clone(), None);
    let store = Store::new("http://localhost:6334", "docs").await?;
    let answerer = Answerer {
        client: reqwest::Client::new(), api_key, model: "gpt-4o-mini".into(),
        base_url: "https://api.openai.com/v1".into(),
    };

    // Ingest
    for entry in walkdir::WalkDir::new("./corpus").into_iter().flatten() {
        if !entry.file_type().is_file() { continue; }
        let path = entry.path();
        let doc = match path.extension().and_then(|e| e.to_str()) {
            Some("pdf") => parse_pdf(path)?,
            Some("md") => parse_markdown(path)?,
            _ => continue,
        };
        let chunks = chunk_doc(&doc, 500)?;
        let texts: Vec<String> = chunks.iter().map(|c| c.embed_text.clone()).collect();
        let vectors = embedder.embed(&texts).await?;
        store.upsert(&chunks, vectors).await?;
    }

    // Query
    let question = "What is the refund window for enterprise customers?";
    let qv = embedder.embed(&[question.to_string()]).await?.pop().unwrap();
    let hits = store.search(qv, 8).await?;
    let answer = answerer.answer(question, &hits).await?;

    println!("{}\n\nCitations:", answer.text);
    for c in &answer.citations {
        println!("- {} ({})", c.source_uri, c.section_path);
    }
    Ok(())
}
```

That is the whole system. Roughly 300 lines including imports. It ingests a directory, answers questions, cites sources.

## Now the part everyone skips: does it work?

"It answered my test question" is not evaluation. It is a demo. A RAG system has two components that fail in different ways, and you need a separate measurement for each.

**Retrieval evaluation.** Build a small set of (question, expected_source_id) pairs. Twenty is enough to start. For each question, embed, search top-K, check if the expected source is in the returned set. Compute:

- **Recall@K**: the fraction of questions where the correct chunk is among the top K. This is your ceiling. If recall@10 is 0.6, nothing downstream can fix it. The LLM does not get to read what was not retrieved.
- **MRR** (mean reciprocal rank): average of `1/rank` where `rank` is the position of the first correct chunk. Rewards putting the right answer at the top, not just in the list.

```rust
pub struct EvalCase { pub question: String, pub expected_source_id: String }

pub async fn recall_at_k(cases: &[EvalCase], embedder: &Embedder, store: &Store, k: u64)
    -> anyhow::Result<f32>
{
    let mut hits = 0;
    for case in cases {
        let v = embedder.embed(&[case.question.clone()]).await?.pop().unwrap();
        let results = store.search(v, k).await?;
        if results.iter().any(|r| r.source_uri.contains(&case.expected_source_id)) {
            hits += 1;
        }
    }
    Ok(hits as f32 / cases.len() as f32)
}
```

**Generation evaluation.** Two signals that matter.

- **Faithfulness**: does the answer actually follow from the retrieved chunks, or did the model paper over a gap with training-prior fabrications? The cheap way to measure this is LLM-as-judge: feed `(question, retrieved_chunks, generated_answer)` to a different model (or the same one with a different prompt) and ask "is every claim in the answer supported by the chunks, yes/no". You will not get 100% agreement with human labels, but the delta between a good system and a broken one is obvious.
- **Citation accuracy**: for each `[n]` in the answer, does chunk `n` actually contain the claim the sentence is making? Again, LLM-as-judge, or manual review on a sample.

Ragas, TruLens, and DeepEval all wrap these metrics. They are Python libraries, but they talk to HTTP endpoints and do not care that your pipeline is in Rust. Run them against an HTTP shim around your `answer` function.

The numbers to aim for, very roughly: recall@10 above 0.85, MRR above 0.5, faithfulness above 0.9 on the eval set. Anything below and you have a concrete thing to fix rather than a vague "it feels off" complaint from users.

## What fails in practice

In the order I have seen these bite:

1. **Bad chunks.** Almost always. Fenced code blocks split in half, tables flattened to line noise, PDFs with two-column layouts that interleave column text. Inspect your chunks. Print ten random ones.
2. **Retrieval misses.** The question uses different vocabulary than the document. "How do I refund a customer?" vs a doc titled "Reversal Policy". Hybrid search (dense + BM25) and query rewriting are the fixes. Qdrant supports both.
3. **LLM ignoring context.** Usually a prompt problem. Add explicit "only from CONTEXT" language, lower temperature, and if you are on a small model consider structured output so the answer must reference chunk IDs.
4. **Stale index.** Documents changed, embeddings did not. Use the deterministic chunk IDs and re-ingest on a schedule, or listen for change events.

## What to take away

Three hundred lines of Rust gets you a production-shaped RAG pipeline: parse, chunk with real tokens, embed in batches, store in Qdrant with deterministic IDs, retrieve with cosine top-K, answer with an explicit "cite or refuse" system prompt. The parts that look trivial - token counting, char boundary safety, UUID v5 for stable upserts, temperature 0.0, prepending section headings into `embed_text` - are where every bug I have debugged in RAG lives.

The part that does not fit in 300 lines is evaluation, and it is the part that determines whether the system ships. Build the eval set first. Measure recall@K before you touch the prompt. Measure faithfulness before you celebrate the demo. Everything else is adjustable knobs on a pipeline whose baseline you do not yet know.

If you want to swap OpenAI for a local model, [Local LLMs with llama.cpp and Rust](/blog/local-llms-with-llama-cpp-and-rust) covers how to run both the embedder and the generator on your own hardware. The API shape stays the same; the cost model does not.

## References

- [`pdf-extract`](https://github.com/jrmuizel/pdf-extract) for PDF text extraction
- [`text-splitter`](https://github.com/benbrandt/text-splitter) recursive chunker by Ben Brandt
- [`qdrant-client`](https://github.com/qdrant/rust-client) Rust client for Qdrant
- [`tiktoken-rs`](https://github.com/zurawiki/tiktoken-rs) OpenAI tokenizer in Rust
- [Qdrant quickstart](https://qdrant.tech/documentation/quickstart/) for running the server
- [Ragas](https://github.com/explodinggradients/ragas) RAG evaluation metrics
- [OpenAI embeddings](https://platform.openai.com/docs/guides/embeddings) API reference
