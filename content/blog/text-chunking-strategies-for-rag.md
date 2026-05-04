+++
title = "Text chunking strategies for RAG: the part that actually matters"
date = 2025-02-18
description = "How you split documents matters more than which embedding model you pick. Fixed, sentence, paragraph, recursive, semantic chunking, plus a Rust implementation that does not lie to you."

[taxonomies]
tags = ["rag", "ai", "llm", "rust"]
+++

If you have built a RAG pipeline that retrieves the wrong chunks, you probably debugged it by swapping embedding models. `text-embedding-3-small` to `text-embedding-3-large`. `bge-m3` to `voyage-3`. Cohere to Jina. The numbers move a little, the bad retrievals stay bad.

The problem is almost never the embedding model. The problem is what you fed it. A 1536-dimensional vector for a 12-token fragment that got sliced out of the middle of a table cell is a precise representation of nothing. An embedding of a 2000-token wall of mixed topics is an average of averages. Either way, your nearest-neighbor search returns garbage, and no amount of dimension tuning fixes it.

<!-- more -->

This post is about the step that happens before embedding. It assumes you have read [Fine-tuning vs RAG vs prompt engineering](/blog/fine-tuning-vs-rag-vs-prompt-engineering), which covers where RAG fits in the larger picture, and [HNSW: how vector search actually works under the hood](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood), which covers the retrieval side. Here we zoom in on chunking.

## What a chunk actually is

A chunk is a self-contained unit of text that gets one embedding and one row in your vector store. At query time, it is the minimum thing you can retrieve. If the answer to a question is scattered across two chunks, you need both to be returned together or your top-K has to be big enough to include them both.

Chunks have three things attached to them:

1. The text that gets embedded.
2. The text that gets shown to the LLM at answer time (sometimes different from what you embedded).
3. Metadata: source document ID, page number, section heading, timestamps, whatever you want to filter on.

People forget (2) and (3) and then wonder why their RAG system cannot say "according to the 2024 pricing guide, section 3.2" even when the retrieval is correct.

## Tokens, not characters

The unit that matters is tokens, because that is what the embedding model consumes and what the LLM pays for. Characters are a crude approximation. English text is roughly 4 characters per token for modern BPE tokenizers, but code is closer to 3, Japanese can be 1, and structured data (JSON with long keys) is all over the place.

If you budget 500 characters per chunk, you might be feeding 180 tokens of Japanese or 90 tokens of JSON into the same bucket. Your embeddings will be inconsistent in density and your retrieval quality will suffer.

Count tokens. The [`tiktoken-rs`](https://github.com/zurawiki/tiktoken-rs) crate wraps OpenAI's tokenizer for BPE-compatible models. For Anthropic's tokenizer, `anthropic-tokenizer-rs` exists but is less maintained; the safe move is to overcount with `cl100k_base` since Claude and GPT tokenizers are close enough for chunking purposes.

## Strategy 1: fixed-size

The simplest possible chunker. Split on character or token boundaries every N units. Move on.

```rust
pub fn fixed_size_chunks(text: &str, size: usize, overlap: usize) -> Vec<&str> {
    assert!(overlap < size, "overlap must be smaller than size");
    let bytes = text.as_bytes();
    let mut chunks = Vec::new();
    let mut start = 0;
    while start < bytes.len() {
        let end = (start + size).min(bytes.len());
        // Walk back to a char boundary to avoid splitting UTF-8
        let mut safe_end = end;
        while safe_end > start && !text.is_char_boundary(safe_end) {
            safe_end -= 1;
        }
        chunks.push(&text[start..safe_end]);
        if safe_end == bytes.len() {
            break;
        }
        start = safe_end.saturating_sub(overlap);
    }
    chunks
}
```

That `is_char_boundary` check is not optional. Rust `&str` slicing panics on non-boundary indices because UTF-8 multi-byte sequences would be corrupted. The first time you feed this chunker a document with emojis or Cyrillic text, you will learn that the hard way.

Fixed-size chunking is fine for a baseline. It will retrieve something relevant most of the time. But it slices through sentences, breaks code blocks in half, splits enumerated lists at item 3, and generally produces chunks that read like someone tore pages out of a book with their eyes closed.

## Strategy 2: sentence-based

Split on sentence boundaries. Group sentences until you hit your token budget. This respects the unit humans use for complete thoughts.

The naive approach is splitting on `.`, `!`, `?`. This breaks on "Dr. Smith went to the U.S. Navy." and "The price is $1.50." and every URL ever. Use a real sentence segmenter.

The `unicode-segmentation` crate gives you Unicode-aware word and sentence boundaries. It is correct on most prose but underperforms on technical text (abbreviations, inline code) compared to heavier tools like [pragmatic-segmenter](https://github.com/diasks2/pragmatic_segmenter) (no Rust port, but the rules are documented). For most RAG use cases, `unicode-segmentation` is good enough:

```rust
use unicode_segmentation::UnicodeSegmentation;

pub fn sentence_chunks(text: &str, max_tokens: usize, est_tokens_per_char: f32)
    -> Vec<String>
{
    let sentences: Vec<&str> = text.unicode_sentences().collect();
    let mut chunks = Vec::new();
    let mut current = String::new();
    let mut current_tokens = 0.0_f32;

    for sent in sentences {
        let sent_tokens = sent.len() as f32 * est_tokens_per_char;
        if current_tokens + sent_tokens > max_tokens as f32 && !current.is_empty() {
            chunks.push(std::mem::take(&mut current));
            current_tokens = 0.0;
        }
        current.push_str(sent);
        current_tokens += sent_tokens;
    }
    if !current.is_empty() {
        chunks.push(current);
    }
    chunks
}
```

For English prose, `est_tokens_per_char` of about 0.25 is close. For serious work, replace that estimate with a real tokenizer call.

The benefit is that chunk boundaries land on something meaningful. The cost is variance: chunk sizes now range from "one terse sentence" to "almost the full budget." Your vector density is inconsistent.

## Strategy 3: paragraph-based

One step up. Split on `\n\n` (or whatever your document calls a paragraph break), pack paragraphs together until you hit the budget, spill to a new chunk.

Paragraphs are usually coherent. A well-written paragraph is already one idea. If your corpus is decently formatted prose (blog posts, documentation, essays) this outperforms sentence chunking because it preserves the author's intended unit of thought.

The failure mode is paragraphs that are too long (over your budget). You need a fallback: if a single paragraph exceeds the budget, drop back to sentence splitting inside it. This is the start of what people call "recursive" chunking.

## Strategy 4: recursive / structural

Recursive chunking is the strategy LangChain's `RecursiveCharacterTextSplitter` popularized, and it remains the best default for mixed-content corpora. The idea: try to split on the largest unit first, and only fall back to smaller units if the current piece exceeds the budget.

For Markdown: split on H1 headers, then H2, then H3, then paragraphs, then sentences, then characters. For code: split on top-level declarations (functions, classes), then blocks, then lines. For HTML: section, article, paragraph.

Pseudocode:

```rust
fn recursive_split(text: &str, budget: usize, separators: &[&str]) -> Vec<String> {
    if token_count(text) <= budget {
        return vec![text.to_string()];
    }
    if separators.is_empty() {
        // Give up: fall back to fixed-size byte split
        return fixed_size_chunks(text, budget * 4, budget)
            .into_iter().map(String::from).collect();
    }
    let sep = separators[0];
    let rest = &separators[1..];
    let pieces: Vec<&str> = text.split(sep).collect();

    let mut out = Vec::new();
    let mut buf = String::new();
    for piece in pieces {
        let candidate_len = token_count(&buf) + token_count(piece);
        if candidate_len > budget && !buf.is_empty() {
            out.push(std::mem::take(&mut buf));
        }
        if token_count(piece) > budget {
            // Piece alone is too big, recurse with smaller separators
            for sub in recursive_split(piece, budget, rest) {
                out.push(sub);
            }
        } else {
            if !buf.is_empty() { buf.push_str(sep); }
            buf.push_str(piece);
        }
    }
    if !buf.is_empty() { out.push(buf); }
    out
}

fn token_count(s: &str) -> usize { s.len() / 4 } // replace with real tokenizer
```

For Markdown specifically, the [`pulldown-cmark`](https://github.com/raphlinus/pulldown-cmark) crate emits events you can use to walk the document structure. You know when you are inside a heading, a list, a code block, a table. Recursive chunking that respects Markdown structure never splits a fenced code block in half, which is the single most annoying thing a naive chunker does to a technical docs corpus.

The [`text-splitter`](https://github.com/benbrandt/text-splitter) crate by Ben Brandt implements this well for Rust. It supports plain text, Markdown, and code (via tree-sitter), handles token counting via `tiktoken-rs` or `tokenizers`, and is what I reach for when I do not want to write my own.

## Strategy 5: semantic chunking

Semantic chunking uses embeddings to find natural topic boundaries. The idea: embed every sentence, compute cosine similarity between adjacent sentences, and cut when similarity drops below a threshold (or at local minima).

```
sent_1 -> embed -> v1
sent_2 -> embed -> v2
sent_3 -> embed -> v3
sim(v1,v2) = 0.91  # stay together
sim(v2,v3) = 0.42  # topic shift, cut here
```

This produces chunks that are topically coherent regardless of formatting. It works well for transcripts, legal documents, or anything where the author did not use paragraph breaks helpfully.

The downsides are real. You pay for an embedding call per sentence at ingest, which for a million-sentence corpus is not free (around $0.02 per million tokens with `text-embedding-3-small`, so embedding sentences and then re-embedding the final chunks roughly doubles cost). You introduce variance in chunk size. And the threshold is hyperparameter-sensitive: too tight and you get 1-sentence chunks, too loose and you get the whole document as one.

In my experience, semantic chunking shines on long-form narrative (podcasts transcribed, legal filings, academic papers) and adds little on already-structured content (API docs, Stack Overflow answers, READMEs). Use it when structure is absent.

## Overlap

Overlap is when consecutive chunks share some text. Chunk 1 covers tokens 0-500, chunk 2 covers 400-900, chunk 3 covers 800-1300. The shared 100 tokens between each pair exist so that a concept that straddles a boundary is visible in at least one full chunk.

Typical values: 10-20% overlap. For 500-token chunks, 50-100 tokens of overlap. More than 25% is usually waste; you are paying to embed and store near-duplicate content.

The gotcha: if you do sentence-aware or paragraph-aware chunking, overlap should also be sentence-aware. Overlapping in the middle of a sentence undoes the whole point of sentence chunking. For my chunker I keep a ring buffer of the last N sentences and prepend them to the next chunk.

## Chunk size tradeoffs

The two failure modes, concretely:

**Too small.** You embed a 40-token fragment that says "this function returns an error if the input is malformed." Great, but *which* function? Without the surrounding definition, the embedding is dominated by generic error-handling semantics. Your nearest-neighbor search will return this chunk for any question about error handling, regardless of whether the actual function is relevant. Retrieval precision collapses.

**Too big.** You embed a 2000-token chunk covering five different subsections. The single 1536-dim vector averages across all of them. When a user asks about subsection 3, chunk 4 of some unrelated document that happens to focus on that topic will score higher than yours, because your vector is diluted by the other four subsections. Retrieval recall collapses.

Where the sweet spot lives depends on content and queries. Rough ranges I have seen work across domains:

- API documentation, short Q&A: 200-400 tokens.
- Technical blog posts, prose documentation: 400-800 tokens.
- Long-form content (books, papers, legal): 600-1200 tokens, usually with hierarchical summaries on top.
- Code (function-level): whatever the function is, up to roughly 1000 tokens, else split at block boundaries.

Test with your own eval set. Numbers from blog posts, including this one, are priors, not answers.

## Metadata preservation: the thing people forget

A chunk without metadata is a string floating in space. With metadata, it is a citation. The difference shows up when your PM asks "where did the model get this number?" and you have to answer more than "vibes."

Minimum metadata for a serious RAG pipeline:

```rust
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct Chunk {
    pub id: String,              // deterministic hash of content + position
    pub text: String,            // what the LLM sees
    pub embed_text: String,      // what the embedder saw (may differ, e.g. with prepended heading)
    pub source_id: String,       // document ID
    pub source_uri: String,      // e.g. https://docs.example.com/pricing
    pub section_path: Vec<String>, // ["Billing", "Plans", "Enterprise"]
    pub position: usize,         // chunk index within document
    pub token_count: usize,
    pub created_at: String,      // ISO 8601
}
```

Two subtle moves:

**`embed_text` vs `text`.** Prepending the section heading (or document title) to the text you embed improves retrieval a lot. "Pricing / Enterprise / Volume discounts start at 100 seats." embeds differently from "Volume discounts start at 100 seats." But at answer time you may want to show the LLM only the second version plus the path as metadata, so it does not have to parse the prepended heading out. Store both.

**Deterministic IDs.** Hash content plus source plus position. Re-ingesting a document should produce the same IDs for unchanged chunks so your vector DB knows what to update versus insert. SHA-256 of `format!("{}:{}:{}", source_id, position, text)` is fine.

## Putting it together

Here is a Rust chunker that does recursive, sentence-aware, overlap-aware chunking with metadata preservation. It depends on `unicode-segmentation` and `serde`:

```rust
use unicode_segmentation::UnicodeSegmentation;

pub struct Chunker {
    pub max_tokens: usize,
    pub overlap_tokens: usize,
    pub tokens_per_char: f32,
}

impl Chunker {
    pub fn chunk(
        &self,
        text: &str,
        source_id: &str,
        source_uri: &str,
    ) -> Vec<Chunk> {
        let paragraphs: Vec<&str> = text.split("\n\n")
            .filter(|p| !p.trim().is_empty())
            .collect();

        let mut out = Vec::new();
        let mut buf = String::new();
        let mut overlap_buf: Vec<String> = Vec::new();
        let mut pos = 0usize;

        for para in paragraphs {
            let para_tokens = self.estimate_tokens(para);
            if self.estimate_tokens(&buf) + para_tokens > self.max_tokens
               && !buf.is_empty()
            {
                out.push(self.finalize(&buf, source_id, source_uri, pos));
                pos += 1;
                buf = self.seed_with_overlap(&overlap_buf);
            }

            if para_tokens > self.max_tokens {
                // Paragraph alone too big, drop to sentences
                for sent in para.unicode_sentences() {
                    let sent_tokens = self.estimate_tokens(sent);
                    if self.estimate_tokens(&buf) + sent_tokens > self.max_tokens
                       && !buf.is_empty()
                    {
                        out.push(self.finalize(&buf, source_id, source_uri, pos));
                        pos += 1;
                        buf = self.seed_with_overlap(&overlap_buf);
                    }
                    buf.push_str(sent);
                    Self::push_overlap(&mut overlap_buf, sent);
                }
            } else {
                if !buf.is_empty() { buf.push_str("\n\n"); }
                buf.push_str(para);
                Self::push_overlap(&mut overlap_buf, para);
            }
        }
        if !buf.is_empty() {
            out.push(self.finalize(&buf, source_id, source_uri, pos));
        }
        out
    }

    fn estimate_tokens(&self, s: &str) -> usize {
        (s.len() as f32 * self.tokens_per_char).ceil() as usize
    }

    fn seed_with_overlap(&self, overlap: &[String]) -> String {
        let mut seed = String::new();
        let mut tokens = 0usize;
        for piece in overlap.iter().rev() {
            let t = self.estimate_tokens(piece);
            if tokens + t > self.overlap_tokens { break; }
            seed = format!("{} {}", piece, seed);
            tokens += t;
        }
        seed.trim().to_string()
    }

    fn push_overlap(buf: &mut Vec<String>, piece: &str) {
        buf.push(piece.to_string());
        if buf.len() > 8 { buf.remove(0); }
    }

    fn finalize(
        &self,
        text: &str,
        source_id: &str,
        source_uri: &str,
        position: usize,
    ) -> Chunk {
        let text = text.trim().to_string();
        let id = deterministic_id(source_id, position, &text);
        let token_count = self.estimate_tokens(&text);
        Chunk {
            id,
            embed_text: text.clone(),
            text,
            source_id: source_id.to_string(),
            source_uri: source_uri.to_string(),
            section_path: Vec::new(),
            position,
            token_count,
            created_at: now_iso(),
        }
    }
}

fn deterministic_id(source_id: &str, position: usize, text: &str) -> String {
    use sha2::{Digest, Sha256};
    let mut hasher = Sha256::new();
    hasher.update(source_id.as_bytes());
    hasher.update(position.to_le_bytes());
    hasher.update(text.as_bytes());
    format!("{:x}", hasher.finalize())
}

fn now_iso() -> String {
    // use `time` or `chrono` crate here in real code
    "2026-04-23T00:00:00Z".to_string()
}
```

Things this does right: paragraph-first with sentence fallback, UTF-8 safe splits through `unicode-segmentation`, overlap is sentence or paragraph sized not byte-sized, deterministic IDs for stable upserts, and the struct is ready to serialize to your vector DB.

Things to wire up for production: swap `estimate_tokens` for a real tokenizer call (`tiktoken-rs` for OpenAI, `tokenizers` crate for Hugging Face), fill in `section_path` by walking Markdown headings with `pulldown-cmark`, and handle code blocks as atomic units by pre-processing with a Markdown event walker so fenced blocks never cross chunk boundaries.

## What to take away

The chunking choice has a bigger effect on retrieval quality than the embedding model choice, for any budget that matters. Start with recursive chunking that respects document structure (headings, paragraphs, sentences) with 400-800 token chunks and 10-20% sentence-aware overlap. Preserve metadata aggressively: source URI, section path, position, deterministic ID. Measure.

Semantic chunking is worth it for unstructured long-form content. Skip it for structured docs. Fixed-size is fine for a first draft only.

The reason "chunking is fiddly" keeps showing up in RAG post-mortems is that people treat it as preprocessing, adjacent to the real problem. It is the problem. The embedding model is a function that maps strings to vectors. Whether those vectors are useful is almost entirely determined by what strings you gave it.

## References

- [`text-splitter`](https://github.com/benbrandt/text-splitter) Rust crate (recursive, markdown, code via tree-sitter)
- [`tiktoken-rs`](https://github.com/zurawiki/tiktoken-rs) for OpenAI tokenizer counts
- [`unicode-segmentation`](https://github.com/unicode-rs/unicode-segmentation) for Unicode-aware sentence splits
- [`pulldown-cmark`](https://github.com/raphlinus/pulldown-cmark) for Markdown event parsing
- [LangChain RecursiveCharacterTextSplitter](https://python.langchain.com/api_reference/text_splitters/character/langchain_text_splitters.character.RecursiveCharacterTextSplitter.html) reference implementation
- [Chunking strategies for RAG (Pinecone)](https://www.pinecone.io/learn/chunking-strategies/) overview with evaluation data
- [Semantic chunking notebook (Greg Kamradt)](https://github.com/FullStackRetrieval-com/RetrievalTutorials/blob/main/tutorials/LevelsOfTextSplitting/5_Levels_Of_Text_Splitting.ipynb) origin of the five levels framing
