+++
title = "Building a chatbot with memory in Rust"
date = 2025-06-15
description = "How ChatGPT remembers you: sliding windows, rolling summaries, and embedding-based recall wired up in about 250 lines of Axum and SQLite."

[taxonomies]
tags = ["rust", "ai", "llm", "axum"]
+++

A base LLM is a pure function. You hand it a list of messages, you get tokens back, and the next call starts from zero. No file on disk. No session. Nothing carried over. Yet ChatGPT apparently remembers your dog's name three weeks later, picks up the tone of yesterday's conversation, and knows that "the project" means the one you were debugging on Tuesday.

The trick is not in the model. The trick is a small, boring server sitting in front of it that curates what goes into the next call. This post is about writing that server in Rust. We are going to build short-term memory (so the assistant sees the last few turns verbatim), rolling summaries (so older turns are compressed into a paragraph instead of dropped), and long-term semantic memory (so facts from months-old conversations can be recalled by meaning). Axum for the API, SQLite for storage, an embedding API for recall, roughly 250 lines of honest code.

<!-- more -->

If you have not read [Fine-tuning vs RAG vs prompt engineering](/blog/fine-tuning-vs-rag-vs-prompt-engineering), that post frames why you would layer retrieval on top of a base model at all. The long-term memory piece here is a specialised RAG, so the full RAG pipeline from [Building a document Q&A system with RAG in Rust](/blog/building-a-document-qa-system-with-rag-in-rust) is the sibling you want to read next. For the similarity math I rely on [Cosine similarity: the math behind semantic search](/blog/cosine-similarity-the-math-behind-semantic-search).

## Three kinds of memory, and why you need all three

The single biggest category error in chatbot design is treating "memory" as one feature. It is at least three. Each one is cheap to get wrong and they interact.

1. **Short-term buffer.** The last N turns, verbatim. This is what makes follow-up questions work. "Can you make that shorter?" is meaningless without the last assistant reply sitting right there in the prompt.
2. **Session summary.** A compressed view of everything in this conversation that is older than the buffer. Rolled forward so the prompt stays bounded even if the conversation runs for a thousand turns.
3. **Long-term semantic memory.** Facts, preferences, and references pulled from previous conversations by meaning, not by position. This is what lets the bot remember your dog's name in a session you started a month later.

A production system uses all three because they cover different failure modes. Drop the buffer and coherent multi-turn reasoning falls apart. Drop summaries and long sessions hit the context window and start dropping facts at random. Drop long-term memory and the assistant feels anonymous every time you come back.

## Architecture

```
       ┌─────────────────────────────────────────────────┐
       │              POST /chat                         │
       │  { session_id, user_id, message }               │
       └────────────────────────┬────────────────────────┘
                                │
                                ▼
           ┌────────────────────────────────────┐
           │ 1. load last N messages (buffer)   │
           │ 2. load session summary if any     │
           │ 3. embed new message               │
           │ 4. retrieve top-K old messages     │
           │    from other sessions by cosine   │
           └────────────────────────┬───────────┘
                                    ▼
           ┌────────────────────────────────────┐
           │  build prompt under token budget   │
           │    [system]                        │
           │    [long-term: K snippets]         │
           │    [summary of this session]       │
           │    [buffer: last N turns]          │
           │    [user: new message]             │
           └────────────────────────┬───────────┘
                                    ▼
                               LLM call
                                    │
                                    ▼
           ┌────────────────────────────────────┐
           │  store user + assistant messages   │
           │  (with embeddings) in SQLite       │
           │  if buffer > threshold: summarize  │
           │    oldest half into session summary│
           └────────────────────────────────────┘
```

The ingest and serve paths in [the RAG post](/blog/building-a-document-qa-system-with-rag-in-rust) were separate processes. Here they are the same request. Every chat turn is both a query against memory and a write to memory.

## Schema

Three tables. I am using SQLite with the [`sqlx`](https://docs.rs/sqlx) crate and storing embeddings as raw `BLOB`s of little-endian `f32`s. With embeddings at 1536 dimensions that is 6 KB per row, fine for tens of thousands of messages. Past that you want a real vector index (see the [HNSW post](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood)) but SQLite in WAL mode will happily do brute-force cosine on a few million rows if you are patient.

```sql
CREATE TABLE sessions (
    id          TEXT PRIMARY KEY,
    user_id     TEXT NOT NULL,
    summary     TEXT NOT NULL DEFAULT '',
    created_at  TEXT NOT NULL
);

CREATE TABLE messages (
    id          TEXT PRIMARY KEY,
    session_id  TEXT NOT NULL REFERENCES sessions(id),
    user_id     TEXT NOT NULL,
    role        TEXT NOT NULL,   -- 'user' | 'assistant'
    content     TEXT NOT NULL,
    embedding   BLOB NOT NULL,
    created_at  TEXT NOT NULL
);
CREATE INDEX idx_messages_user ON messages(user_id, created_at);
CREATE INDEX idx_messages_session ON messages(session_id, created_at);
```

One schema note worth internalising: the embedding goes on every message, not just user messages. The assistant's replies are exactly the things you want to recall later ("I told you last week that the deploy was broken because of the feature flag"). Embedding both doubles your storage but halves your bug reports.

## The server

I will paste the whole thing and then walk through it. `Cargo.toml`:

```toml
[dependencies]
anyhow = "1"
axum = "0.7"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite", "migrate"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", features = ["json"] }
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1", features = ["v4"] }
tiktoken-rs = "0.5"
```

`src/main.rs`:

```rust
use anyhow::{anyhow, Result};
use axum::{extract::State, routing::post, Json, Router};
use serde::{Deserialize, Serialize};
use sqlx::{sqlite::SqlitePoolOptions, SqlitePool};
use std::sync::Arc;

const BUFFER_TURNS: i64 = 6;          // last N user+assistant messages kept verbatim
const SUMMARY_TRIGGER: i64 = 20;      // compress once session has more than this
const LONG_TERM_K: usize = 4;         // top-K old snippets to pull in
const MAX_CONTEXT_TOKENS: usize = 6000;
const EMBED_MODEL: &str = "text-embedding-3-small";   // 1536 dims
const CHAT_MODEL: &str = "gpt-4o-mini";

#[derive(Clone)]
struct AppState {
    db: SqlitePool,
    http: reqwest::Client,
    api_key: String,
    tokenizer: Arc<tiktoken_rs::CoreBPE>,
}

#[derive(Deserialize)]
struct ChatReq { session_id: String, user_id: String, message: String }
#[derive(Serialize)]
struct ChatResp { reply: String }

#[derive(Serialize, Deserialize, Clone)]
struct Msg { role: String, content: String }

#[tokio::main]
async fn main() -> Result<()> {
    let db = SqlitePoolOptions::new().connect("sqlite://chat.db?mode=rwc").await?;
    sqlx::migrate!("./migrations").run(&db).await?;

    let state = AppState {
        db,
        http: reqwest::Client::new(),
        api_key: std::env::var("OPENAI_API_KEY")?,
        tokenizer: Arc::new(tiktoken_rs::cl100k_base()?),
    };
    let app = Router::new().route("/chat", post(chat)).with_state(state);
    let lst = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(lst, app).await?;
    Ok(())
}

async fn chat(State(s): State<AppState>, Json(r): Json<ChatReq>) -> Json<ChatResp> {
    let reply = do_chat(&s, &r).await.unwrap_or_else(|e| format!("error: {e}"));
    Json(ChatResp { reply })
}

async fn do_chat(s: &AppState, r: &ChatReq) -> Result<String> {
    ensure_session(&s.db, &r.session_id, &r.user_id).await?;

    // 1. embed the incoming message
    let q_emb = embed(s, &r.message).await?;

    // 2. pull the three kinds of memory
    let buffer   = load_buffer(&s.db, &r.session_id, BUFFER_TURNS).await?;
    let summary  = load_summary(&s.db, &r.session_id).await?;
    let recalled = recall(&s.db, &r.user_id, &r.session_id, &q_emb, LONG_TERM_K).await?;

    // 3. build a prompt that fits
    let messages = build_prompt(s, &summary, &recalled, &buffer, &r.message)?;

    // 4. call the LLM
    let reply = complete(s, &messages).await?;

    // 5. persist both sides with embeddings
    let a_emb = embed(s, &reply).await?;
    store_msg(&s.db, &r.session_id, &r.user_id, "user", &r.message, &q_emb).await?;
    store_msg(&s.db, &r.session_id, &r.user_id, "assistant", &reply, &a_emb).await?;

    // 6. compress if the session is getting long
    maybe_summarize(s, &r.session_id).await?;

    Ok(reply)
}
```

That is the orchestration. The helpers fill in the rest.

### Embedding

Every message becomes a vector once, at write time. The query message is embedded inline because we need it right away for retrieval.

```rust
async fn embed(s: &AppState, text: &str) -> Result<Vec<f32>> {
    #[derive(Serialize)] struct Req<'a> { input: &'a str, model: &'a str }
    #[derive(Deserialize)] struct Resp { data: Vec<Item> }
    #[derive(Deserialize)] struct Item { embedding: Vec<f32> }

    let r: Resp = s.http.post("https://api.openai.com/v1/embeddings")
        .bearer_auth(&s.api_key)
        .json(&Req { input: text, model: EMBED_MODEL })
        .send().await?.error_for_status()?.json().await?;
    r.data.into_iter().next().map(|i| i.embedding)
        .ok_or_else(|| anyhow!("no embedding"))
}
```

### Storing, and the BLOB trick

```rust
fn enc_vec(v: &[f32]) -> Vec<u8> {
    let mut b = Vec::with_capacity(v.len() * 4);
    for x in v { b.extend_from_slice(&x.to_le_bytes()); }
    b
}
fn dec_vec(b: &[u8]) -> Vec<f32> {
    b.chunks_exact(4).map(|c| f32::from_le_bytes(c.try_into().unwrap())).collect()
}

async fn store_msg(db: &SqlitePool, sid: &str, uid: &str,
                   role: &str, content: &str, emb: &[f32]) -> Result<()> {
    sqlx::query("INSERT INTO messages(id, session_id, user_id, role, content, embedding, created_at)
                 VALUES (?, ?, ?, ?, ?, ?, ?)")
        .bind(uuid::Uuid::new_v4().to_string())
        .bind(sid).bind(uid).bind(role).bind(content)
        .bind(enc_vec(emb))
        .bind(chrono::Utc::now().to_rfc3339())
        .execute(db).await?;
    Ok(())
}
```

I am deliberately avoiding a clever vector extension. SQLite blobs plus a brute-force cosine scan over a single user's history is embarrassingly fast up to tens of thousands of messages, which is more than almost anyone has. Use the right tool at the right scale.

### Recall

Cosine similarity over all messages belonging to this user, *excluding* the current session so we are pulling genuinely old context. Top-K by score.

```rust
fn cosine(a: &[f32], b: &[f32]) -> f32 {
    let (mut dot, mut na, mut nb) = (0.0, 0.0, 0.0);
    for (x, y) in a.iter().zip(b) { dot += x * y; na += x * x; nb += y * y; }
    if na == 0.0 || nb == 0.0 { 0.0 } else { dot / (na.sqrt() * nb.sqrt()) }
}

async fn recall(db: &SqlitePool, uid: &str, sid: &str,
                q: &[f32], k: usize) -> Result<Vec<String>> {
    let rows: Vec<(String, Vec<u8>)> = sqlx::query_as(
        "SELECT content, embedding FROM messages
         WHERE user_id = ? AND session_id != ?
         ORDER BY created_at DESC LIMIT 5000")
        .bind(uid).bind(sid).fetch_all(db).await?;

    let mut scored: Vec<(f32, String)> = rows.into_iter()
        .map(|(c, e)| (cosine(q, &dec_vec(&e)), c))
        .collect();
    scored.sort_by(|a, b| b.0.partial_cmp(&a.0).unwrap());
    Ok(scored.into_iter().filter(|(s, _)| *s > 0.35).take(k).map(|(_, c)| c).collect())
}
```

Two things worth flagging. The `LIMIT 5000` is a sanity cap; past some size you want a real index, not a full table scan per request. The `> 0.35` threshold filters out "we retrieved something but it was semantically unrelated", which is what you see when the current question has no genuine long-term context. If you always return K snippets, you end up stuffing the prompt with noise that degrades the reply.

### Buffer and summary

```rust
async fn load_buffer(db: &SqlitePool, sid: &str, n: i64) -> Result<Vec<Msg>> {
    let rows: Vec<(String, String)> = sqlx::query_as(
        "SELECT role, content FROM messages WHERE session_id = ?
         ORDER BY created_at DESC LIMIT ?")
        .bind(sid).bind(n * 2).fetch_all(db).await?;
    Ok(rows.into_iter().rev().map(|(role, content)| Msg { role, content }).collect())
}

async fn load_summary(db: &SqlitePool, sid: &str) -> Result<String> {
    let s: (String,) = sqlx::query_as("SELECT summary FROM sessions WHERE id = ?")
        .bind(sid).fetch_one(db).await?;
    Ok(s.0)
}
```

### Token budgeting

This is where people quietly get it wrong. Stuffing long-term snippets plus a buffer plus a new message can blow the context window, and when it does the LLM call fails or worse, silently truncates from one end. You cannot trust the model provider to do this sensibly. Measure and decide yourself.

```rust
fn tokens(s: &AppState, text: &str) -> usize {
    s.tokenizer.encode_with_special_tokens(text).len()
}

fn build_prompt(s: &AppState, summary: &str, recalled: &[String],
                buffer: &[Msg], new: &str) -> Result<Vec<Msg>> {
    let mut out = vec![Msg { role: "system".into(),
        content: "You are a helpful assistant with persistent memory.".into() }];

    if !recalled.is_empty() {
        let body = recalled.iter().enumerate()
            .map(|(i, c)| format!("[{}] {}", i + 1, c)).collect::<Vec<_>>().join("\n");
        out.push(Msg { role: "system".into(),
            content: format!("Potentially relevant past context:\n{body}") });
    }
    if !summary.is_empty() {
        out.push(Msg { role: "system".into(),
            content: format!("Summary so far: {summary}") });
    }
    out.extend(buffer.iter().cloned());
    out.push(Msg { role: "user".into(), content: new.into() });

    // enforce the budget: drop oldest buffer turns until we fit
    while out.iter().map(|m| tokens(s, &m.content)).sum::<usize>() > MAX_CONTEXT_TOKENS {
        let drop_idx = out.iter().position(|m| m.role == "user" || m.role == "assistant")
            .ok_or_else(|| anyhow!("cannot fit prompt"))?;
        out.remove(drop_idx);
    }
    Ok(out)
}
```

The eviction order matters. We drop the oldest *buffer* turn, not the summary and not the recalled snippets. Losing a recent turn degrades coherence by one exchange. Losing the summary loses everything older than the buffer. Losing a recalled snippet loses a fact the user explicitly asked us to remember. Rank by what hurts least to lose.

### Summarisation

Running summarisation on every turn is wasteful. Running it never is how you end up with a 400-turn prompt. Compromise: when the total messages exceed `SUMMARY_TRIGGER`, fold the *older half* into the existing summary and leave the recent half as buffer.

```rust
async fn maybe_summarize(s: &AppState, sid: &str) -> Result<()> {
    let (count,): (i64,) = sqlx::query_as(
        "SELECT COUNT(*) FROM messages WHERE session_id = ?")
        .bind(sid).fetch_one(&s.db).await?;
    if count < SUMMARY_TRIGGER { return Ok(()); }

    let cut = count / 2;
    let rows: Vec<(String, String)> = sqlx::query_as(
        "SELECT role, content FROM messages WHERE session_id = ?
         ORDER BY created_at ASC LIMIT ?")
        .bind(sid).bind(cut).fetch_all(&s.db).await?;
    let existing = load_summary(&s.db, sid).await?;

    let transcript = rows.iter()
        .map(|(r, c)| format!("{r}: {c}")).collect::<Vec<_>>().join("\n");
    let prompt = format!(
        "Existing summary:\n{existing}\n\nNew exchanges:\n{transcript}\n\n\
         Write a concise updated summary (under 300 words) that preserves facts, \
         decisions, and user preferences. Drop small-talk.");

    let new_summary = complete(s, &[Msg { role: "user".into(), content: prompt }]).await?;

    sqlx::query("UPDATE sessions SET summary = ? WHERE id = ?")
        .bind(&new_summary).bind(sid).execute(&s.db).await?;
    // optional: delete the summarised messages here if you want
    Ok(())
}
```

Notice the prompt explicitly says "preserves facts, decisions, and user preferences". Without that instruction the model will happily summarise away the one thing the user actually cared about ("we decided to use Postgres, not MySQL") in favour of aesthetic prose. Summaries are lossy by definition; you are telling the model *what* to lose.

### Chat completion

Bog-standard OpenAI-compatible call. Easy to swap for Claude, local llama.cpp via an OpenAI-shim, or Azure.

```rust
async fn complete(s: &AppState, messages: &[Msg]) -> Result<String> {
    #[derive(Serialize)] struct Req<'a> { model: &'a str, messages: &'a [Msg] }
    let v: serde_json::Value = s.http.post("https://api.openai.com/v1/chat/completions")
        .bearer_auth(&s.api_key)
        .json(&Req { model: CHAT_MODEL, messages })
        .send().await?.error_for_status()?.json().await?;
    Ok(v["choices"][0]["message"]["content"].as_str().unwrap_or("").to_string())
}

async fn ensure_session(db: &SqlitePool, sid: &str, uid: &str) -> Result<()> {
    sqlx::query("INSERT OR IGNORE INTO sessions(id, user_id, summary, created_at)
                 VALUES (?, ?, '', ?)")
        .bind(sid).bind(uid).bind(chrono::Utc::now().to_rfc3339())
        .execute(db).await?;
    Ok(())
}
```

That is the whole thing. Under 250 lines of Rust, a real vector recall, a real summariser, a real token budget. Point `curl` at it with `{"session_id":"s1","user_id":"u1","message":"hi"}` and watch the `messages` table fill up.

## Session management, briefly

I glossed over where `session_id` and `user_id` come from because the interesting bit is upstream. A few rules that will save you debugging time:

- A `session_id` is a conversation. A `user_id` is a person. Memory recall reads across *all* sessions of a user, not across users. Mix these up and you leak one user's "my API key is ..." into another's prompt.
- Rotate sessions aggressively. Long-lived session IDs get copied into pastebins and logs. Start a new session on inactivity (say, 4 hours) and let long-term recall do the cross-session bridging.
- Authenticate the `user_id`. The example above trusts the client, which is fine for local testing and catastrophic in production. Put this behind a JWT check or a cookie-authenticated session layer.

## Context window is a budget, not a ceiling

The temptation with a 200K-token model is to think the budget does not exist. It does, just in a different shape. Bigger prompts are slower, more expensive, and *worse*: the "lost in the middle" effect means instructions buried in long prompts get ignored at a measurable rate. Anthropic and OpenAI have both published evals showing accuracy dropping when relevant facts are placed in the middle of a 100K-token context.

So the `MAX_CONTEXT_TOKENS = 6000` in the example is not a hardware limit. It is a quality budget. Fit the conversation in 6K and you get faster, cheaper, more obedient replies. The memory machinery exists *so that* you can hold the prompt small while still feeling stateful.

## What ChatGPT actually does

ChatGPT's memory feature, launched in 2024 and expanded since, is roughly this same recipe with a lot more engineering. The model has access to two memory-like stores: a conversation-level buffer (what you can see in the current chat), and a user-level "Memories" list the model can explicitly write to and read from. The explicit writes are what lets you say "remember that I prefer Python" and have it stick. The implicit retrieval is what lets it bring up last week's conversation without you asking.

The interesting design choice there is the hybrid between extractive ("the user said their dog is named Biscuit, store that") and implicit ("pull last week's conversation about deploys if it seems relevant"). Our 250-line server does only the implicit side. Extending it with explicit memory is ten lines: a `remember(text)` tool the model can call, which writes into a `memories` table that gets retrieved into the prompt unconditionally per user.

## Failure modes worth knowing

A short list of bugs that will bite you.

- **Recall poisoning.** The assistant says something wrong, you do not correct it, it gets embedded, and next week it is the top retrieval result for a related query. The model then confidently repeats its own past hallucination. Fix: add a user-side "mark incorrect" signal that deletes or downweights retrievals.
- **Summary drift.** Each summarisation is lossy. Summarise the summary enough times and you lose the original facts entirely. Fix: keep the raw messages, summarise from the full transcript each time, not from the previous summary.
- **Cross-user leakage.** A mis-scoped SQL query that forgets `WHERE user_id = ?` pulls memory from strangers. Fix: never write that query without the clause; better, enforce it at a view or a typed layer.
- **Embedding drift.** You upgrade the embedding model. Old vectors are now incompatible with new queries and similarity collapses. Fix: store the embedding model name per row and re-embed in the background when you migrate.
- **Context pollution by buffer.** "Can you make that shorter?" retrieved five unrelated snippets because the query embedding was short and generic. Fix: skip long-term recall on messages shorter than, say, 15 characters, or when the top similarity is below your threshold.

Each of those has a fix that is shorter than the bug report. Build the server and you will hit at least two of them in the first week. That is the real reason the architecture is shaped this way: not because the diagram is elegant, but because each layer exists to fail independently from the others.
