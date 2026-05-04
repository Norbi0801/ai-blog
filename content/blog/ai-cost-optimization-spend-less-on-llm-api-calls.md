+++
title = "AI cost optimization: spend less on LLM API calls"
date = 2025-05-27
description = "Practical techniques for cutting LLM spend without hurting quality: caching, prompt compression, model routing, batching, token accounting, and a cost-aware client in Rust."

[taxonomies]
tags = ["ai", "llm", "rust", "performance"]
+++

The first invoice is always educational. A side project that worked beautifully in staging on ten test cases starts costing $400 a day as soon as real users show up. The logs look fine. The model is good. The bill is terrifying.

This post is about what you do next. It is about the boring, effective techniques that actually drop LLM spend by 5-20x in production without degrading quality: caching, prompt compression, model routing, batch inference, token counting, and a small amount of Rust glue that makes the whole thing observable.

<!-- more -->

If you have not read [Fine-tuning vs RAG vs prompt engineering](/blog/fine-tuning-vs-rag-vs-prompt-engineering-when-to-use-what), that post is the architectural twin of this one. That one is about changing the model's behavior. This one is about paying less for whatever behavior you already have.

## Where the money actually goes

LLM cost is a pure function of tokens. For each request you pay:

```
cost = input_tokens * price_per_input + output_tokens * price_per_output
```

As of mid-2026, rough per-million-token prices on the major APIs:

| Model | Input | Output | Cached input |
|---|---|---|---|
| Claude Sonnet 4.6 | $3 | $15 | $0.30 |
| Claude Haiku 4 | $0.80 | $4 | $0.08 |
| GPT-4o | $2.50 | $10 | $1.25 |
| GPT-4o-mini | $0.15 | $0.60 | $0.075 |
| Gemini 2.5 Flash | $0.30 | $2.50 | $0.075 |

A few things jump out. Output tokens are 4-5x more expensive than input tokens. Cached input is 5-20x cheaper than fresh input. Small models are 10-20x cheaper than their frontier siblings. Every serious cost optimization technique below is an exploit of one of those three ratios.

Before you do anything else, know your per-feature cost. If you cannot answer "how much does it cost us when a user clicks this button" in dollars to two significant figures, stop reading this post and go instrument your code. All of the advice below is useless without that number because you will not know what worked.

## Count tokens before you send them

You cannot estimate or bound cost without knowing the token count of what you are about to ship. The API will tell you afterwards. That is too late if your budget is one cent per request and the prompt you just built is 400 cents.

For OpenAI and Anthropic, the tokenizer is not a secret. OpenAI publishes the `cl100k_base` and `o200k_base` BPE encodings used by GPT-4o and the o-series. Anthropic publishes their Claude tokenizer via the `@anthropic-ai/tokenizer` JS package and, via the official SDK, a `count_tokens` endpoint.

From Rust, the pragmatic choice is [`tiktoken-rs`](https://github.com/zurawiki/tiktoken-rs) for OpenAI and the official Anthropic count-tokens endpoint for Claude (since their tokenizer is not open-sourced in a reusable form):

```rust
use tiktoken_rs::{get_bpe_from_model, CoreBPE};

pub struct TokenCounter {
    bpe: CoreBPE,
}

impl TokenCounter {
    pub fn for_model(model: &str) -> anyhow::Result<Self> {
        Ok(Self {
            bpe: get_bpe_from_model(model)?,
        })
    }

    pub fn count(&self, text: &str) -> usize {
        self.bpe.encode_with_special_tokens(text).len()
    }

    pub fn count_messages(&self, messages: &[(&str, &str)]) -> usize {
        // OpenAI's ChatML overhead: 3 tokens per message, 3 tokens priming.
        let mut total = 3;
        for (role, content) in messages {
            total += 3 + self.count(role) + self.count(content);
        }
        total
    }
}
```

The per-message overhead (the magic 3) is documented in OpenAI's cookbook and has been stable across models. It matters: for 100 short messages that adds 300 tokens you did not see in the raw content.

Pair this with a budget check before the request goes out. The shape I keep coming back to:

```rust
pub struct RequestBudget {
    pub max_input_tokens: usize,
    pub max_output_tokens: usize,
    pub max_cost_cents: u32,
}

pub fn check_budget(
    counter: &TokenCounter,
    messages: &[(&str, &str)],
    max_output: usize,
    budget: &RequestBudget,
    price: &PricePerMillion,
) -> Result<(), CostError> {
    let input = counter.count_messages(messages);
    if input > budget.max_input_tokens {
        return Err(CostError::InputTooLong(input));
    }
    let cost_cents =
        (input as u64 * price.input_cents_per_million + max_output as u64 * price.output_cents_per_million)
            / 1_000_000;
    if cost_cents > budget.max_cost_cents as u64 {
        return Err(CostError::BudgetExceeded(cost_cents));
    }
    Ok(())
}
```

That five-line check has killed more expensive outages than any clever prompt I have written. A bad retry loop, a user pasting a 2 MB file, a prompt template that grew a field nobody tested: all of them show up here as a rejected request rather than a five-figure surprise.

## Exact cache: the easiest 30% you will ever save

Exact caching is a hash table keyed on the full request. Same model, same messages, same temperature, same tools: return the cached response. Do not call the API.

In a typical product, somewhere between 20% and 50% of user queries are textually identical to something that was asked in the last hour. "Summarize this article" on an RSS feed. "What do I do next" from a constrained UI. Dashboards that re-run the same prompt every minute. Any deterministic pipeline with a flaky retry.

The key derivation matters. You want to hash:

- Model name and version.
- The full list of messages after normalization (strip trailing whitespace, unify newlines).
- Temperature, top_p, max_tokens, stop sequences.
- The tool definitions, sorted by name.
- The system prompt verbatim.

And crucially you want a content hash (SHA-256 of a canonical JSON serialization) rather than any kind of semantic digest. If the inputs are bit-identical, the output is cacheable. If they are not, it is not an exact cache problem.

```rust
use sha2::{Digest, Sha256};

fn request_key(req: &CanonicalRequest) -> String {
    let bytes = serde_json::to_vec(req).expect("canonical serde");
    let mut h = Sha256::new();
    h.update(&bytes);
    format!("llm:{:x}", h.finalize())
}
```

Store the response with a TTL. Models and prompts evolve; you do not want six-month-old answers coming back. A TTL of a few hours to a few days is usually right. For anything deterministic (temperature 0, stable tools), longer is fine.

Temperature deserves its own note. At temperature 0 the model is approximately deterministic, which makes caching lossless. At temperature > 0 it is not, and caching changes the observable distribution of outputs. That is fine for most product surfaces (users do not care that the summary they got was one of ten possibilities) but it matters for anything where you want diverse samples, like brainstorming.

## Prompt caching: the provider does it for you

Both Anthropic and OpenAI now offer server-side prompt caching. The mechanic is the same: long stable prefixes (system prompt, tool definitions, long context documents) are cached on the provider's side and the next request that reuses that prefix pays a fraction of the input price.

Anthropic's caching is explicit. You mark a block with `cache_control: { "type": "ephemeral" }` and the provider remembers it for five minutes (or one hour with the longer-TTL option). Cache writes cost 1.25x fresh input. Cache reads cost 0.1x (90% off). The break-even is roughly after two requests that reuse the same prefix.

OpenAI's caching is automatic on any prefix over 1024 tokens. Cache reads are 50% off. The provider hashes prefixes and reuses them without you opting in.

The practical design implication is that you want stable prefixes. Put variable content (the user's query) at the end of the message list, not at the beginning. Put tool definitions and long system context up front. If you regenerate your system prompt from a template that includes a timestamp or a UUID, congratulations, you just cache-busted yourself.

For a chatbot with a 3000-token system prompt and 50,000 requests per day, moving from "no caching" to "prompt caching on the system prompt" is a 30-40% total cost cut. It is one of the highest-leverage optimizations available and it takes half an hour to implement.

## Semantic cache: for paraphrases

Exact cache catches identical requests. Semantic cache catches paraphrases. "How do I reset my password?" and "i forgot my password how do i change it" are different strings with the same answer.

The mechanic:

1. Embed each incoming query with a small text embedding model (`text-embedding-3-small` at 512 dims is fine, roughly $0.02 per million tokens).
2. Run a nearest-neighbor search against the cache of previous (query_embedding, response) pairs.
3. If the top hit has cosine similarity above a threshold (0.92-0.97 depending on domain), return that response. Otherwise call the LLM and store the new pair.

If you want the mechanics of the similarity and search layer, I wrote about them in [HNSW algorithm: how vector search actually works](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood) and [Cosine similarity: the math behind semantic search](/blog/cosine-similarity-the-math-behind-semantic-search).

The risk with semantic cache is false positives. "Reset my password" and "delete my password" are semantically close but require different answers. The threshold is everything. Start high (0.97), measure your cache hit rate, lower it only if you have a quality eval that catches regressions.

Do not semantically cache anything with side effects. Do not semantically cache personalized responses. Do not semantically cache anything that includes user-specific data in the answer. The cache key must not be contaminated by user identity unless it is partitioned per user.

## Prompt compression: smaller inputs, same task

Most production prompts are padded. They have instructions the model no longer needs because modern models got better. They have examples that were useful in 2023 and are noise in 2026. They have Markdown decorations that burn 5-10% of the tokens without changing the output.

Three techniques that consistently work:

**Strip Markdown and pretty-printing.** `json.dumps` with default indentation doubles token count for the same data. Use compact JSON. Remove trailing whitespace. Use tabs instead of spaces if you are shipping code in prompts (one token per tab versus one token per space run).

**Truncate long context with attention to position.** If you are feeding documents into a RAG prompt, the middle of a long context is read least reliably by the model (the "lost in the middle" problem). If you must truncate, truncate the middle, not the end. Keep the head (1-2 KB) and the tail (1-2 KB) and drop everything between with a `[...truncated 40KB...]` marker.

**Use structured summaries for history.** A chat transcript with 50 turns is a lot of tokens. A structured summary of that transcript, generated once by a cheap model, is 200 tokens and preserves most of what matters. The pattern is: whenever history exceeds N turns, summarize turns 1..N-10 into a system message, keep turns N-10..N verbatim.

There are also model-assisted compressors like LLMLingua (Microsoft, 2023) that use a small language model to score and drop low-information tokens. They can achieve 2-5x compression with minimal quality loss on RAG-style contexts. In Rust you would run them as a preprocessing step via ONNX or `candle`. Worth knowing exists; overkill for most systems.

## Model routing: cheap first, escalate on failure

A frontier model is worth using when it matters. Most of what users ask does not need one. The idea of model routing is to send the request to the cheapest model that can plausibly answer it, and only escalate if the cheap model declines or fails a quality check.

The simplest router is a classifier. Take a labeled set of past queries, label each with "Haiku-sufficient" or "Sonnet-required", train a small classifier (logistic regression on embeddings is a good baseline), and use it to pick the model. 70-90% of queries routed to Haiku is typical for assistants that handle FAQ and simple tasks.

A more subtle router is the two-stage confidence check. Always run Haiku first. Ask it to output a structured response that includes a confidence field. If confidence is above a threshold, return the answer. If not, redo the request with Sonnet. You pay Haiku cost on most requests and Haiku + Sonnet on the hard ones, but the hard ones are a small fraction.

Rough math on a workload of 1M requests at 500 in / 200 out:

- All Sonnet: 1M * (500 * $3 + 200 * $15) / 1M = $4,500
- All Haiku: 1M * (500 * $0.80 + 200 * $4) / 1M = $1,200
- Routed 80/20 Haiku/Sonnet: 0.8 * $1,200 + 0.2 * $4,500 = $1,860
- Two-stage with 20% escalation: $1,200 + 0.2 * $4,500 = $2,100

The two-stage pattern costs a bit more than a perfect router but needs no training data. It is the right starting point.

There is a failure mode worth flagging. If your cheap model is wrong confidently (Dunning-Kruger for LLMs), confidence-gated routing degrades quality. You need an eval set that measures accuracy at each stage, not just average quality, to catch this.

## Batch inference: 50% off if you can wait

OpenAI and Anthropic both offer a batch API. You submit a JSONL of requests, the provider processes them within 24 hours (usually much sooner), and you pay 50% of the interactive price. If your workload is not user-facing, this is free money.

Good candidates for batch:

- Nightly re-embedding of a document corpus.
- Offline evaluation runs.
- Bulk tagging or classification.
- Background enrichment of records created during the day.
- Any cron-triggered LLM work.

Bad candidates:

- Anything the user is waiting on.
- Anything whose output becomes stale quickly.

From Rust, the batch endpoints are JSONL in, JSONL out. You can write a batch submitter in a couple of hundred lines of `reqwest` and `serde_json`. The operational part (tracking batch status, handling partial failures, resuming) is the harder half, but it is the same pattern as any long-running async job queue.

## Streaming: zero cost savings, big UX win

Streaming does not reduce your bill. You pay for the same number of output tokens whether you stream them or receive them in a single blob. What it does is let users cancel early.

This matters more than it sounds. If a user asks for a long summary and your stream is 2000 tokens deep when they close the tab, you still pay for what was generated up to the point the connection closed. The savings come from what was not generated.

The discipline is to cancel the stream on the client and propagate that cancellation into your request handler. In Rust this maps nicely onto `tokio::select!` or a `CancellationToken`:

```rust
use tokio_util::sync::CancellationToken;

async fn stream_response(
    client: &LlmClient,
    req: Request,
    cancel: CancellationToken,
) -> anyhow::Result<String> {
    let mut stream = client.stream(req).await?;
    let mut buf = String::new();
    loop {
        tokio::select! {
            biased;
            _ = cancel.cancelled() => {
                // drop the stream; SDKs cancel the upstream HTTP
                return Ok(buf);
            }
            chunk = stream.next() => match chunk {
                Some(Ok(piece)) => buf.push_str(&piece),
                Some(Err(e)) => return Err(e.into()),
                None => return Ok(buf),
            }
        }
    }
}
```

Dropping the stream closes the HTTP connection, which signals the provider to stop billing on subsequent tokens. Verify this end-to-end with your provider; behavior has been consistent for both Anthropic and OpenAI for at least two years, but it is the kind of thing that silently changes.

## When local models start to win

A self-hosted 8B model on a single H100 (spot, ~$1.50/hr on-demand in 2026) does roughly 150-300 tokens/sec at typical batch sizes. That is 540K-1M tokens per hour, or somewhere in the range of $0.0015-$0.003 per thousand tokens served. Compare that to Haiku at $0.80 per million ($0.0008 per thousand input, $0.004 per thousand output): the self-hosted model is in the same ballpark if you keep the GPU busy.

The crossover rules, roughly:

- Below 10M tokens/day of traffic: do not host. The GPU sits idle and the engineering time outweighs the savings.
- 10M to 100M tokens/day: host if you have the people, otherwise stay on API.
- Over 100M tokens/day: host. The math is not close.

The other axis is latency and privacy. Local inference means no network hop, no rate limits, no data leaving your infrastructure. If any of those are hard requirements, the decision is not really about cost.

[Local LLMs with llama.cpp and Rust](/blog/local-llms-with-llama-cpp-and-rust) goes into detail on how to actually do this if you decide to cross that line.

## Per-feature cost attribution

None of the above matters if you cannot see where the money goes. Instrument every LLM call with labels for the feature, the tenant, the model, whether the cache was hit, and the final token counts from the response.

A minimal shape:

```rust
#[derive(Clone)]
pub struct CallMetrics {
    pub feature: &'static str,
    pub model: String,
    pub cache_hit: bool,
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub cached_tokens: u32,
    pub cost_millicents: u64,
    pub latency_ms: u32,
}

impl CallMetrics {
    pub fn emit(&self, registry: &MetricsRegistry) {
        registry
            .counter("llm_cost_millicents_total")
            .with_labels(&[("feature", self.feature), ("model", &self.model)])
            .inc_by(self.cost_millicents);
        registry
            .counter("llm_tokens_total")
            .with_labels(&[
                ("feature", self.feature),
                ("kind", "input"),
            ])
            .inc_by(self.input_tokens as u64);
        // ...
    }
}
```

Fan it out to Prometheus, ClickHouse, whatever you already have. The point is that every dashboard about LLM spend should let you pivot by feature. That is how you find the one feature that is 80% of the bill and aim the optimizations above at it instead of spreading them evenly.

## A sketch of a cost-aware Rust client

Putting the pieces together, the shape of a sensible cost-aware LLM client is:

```rust
pub struct CostAwareClient {
    providers: HashMap<String, Arc<dyn LlmProvider>>,
    cache: Arc<dyn ResponseCache>,
    router: Arc<dyn ModelRouter>,
    counter: TokenCounter,
    metrics: Arc<MetricsRegistry>,
}

impl CostAwareClient {
    pub async fn complete(
        &self,
        feature: &'static str,
        req: Request,
        budget: &RequestBudget,
    ) -> Result<Response, LlmError> {
        let start = Instant::now();

        // 1. Token count + budget check
        let tokens = self.counter.count_messages(&req.messages);
        check_budget(&self.counter, &req.messages, req.max_output_tokens, budget, &req.price)?;

        // 2. Exact cache
        let key = request_key(&req.canonical());
        if let Some(hit) = self.cache.get(&key).await? {
            self.emit_metrics(feature, &req, &hit, true, start.elapsed());
            return Ok(hit);
        }

        // 3. Route to a model
        let model = self.router.choose(&req, tokens).await?;
        let req = req.with_model(model);

        // 4. Call the provider
        let resp = self
            .providers
            .get(req.provider())
            .ok_or(LlmError::NoProvider)?
            .complete(&req)
            .await?;

        // 5. Cache and emit metrics
        self.cache.put(&key, &resp, req.cache_ttl).await?;
        self.emit_metrics(feature, &req, &resp, false, start.elapsed());

        Ok(resp)
    }
}
```

None of the individual pieces are novel. The point is that they live in one place: every LLM call in the codebase goes through one component that counts tokens, checks a budget, hits a cache, picks a model, and emits metrics. If you have that, every optimization in this post becomes a patch to one file. If you do not, every optimization becomes a migration.

## What to take away

LLM cost is tokens. Every serious optimization is either "send fewer tokens", "send cheaper tokens", or "skip the call entirely". Exact cache and prompt caching are the first two moves; they compound, they are easy, and they pay for themselves the first week. Model routing and compression come next and need measurement to tune. Batch inference is free money for any workload that is not user-facing. Local models are worth it at scale, not before.

The unglamorous part is the plumbing. Per-feature cost attribution, token counting before the request, a budget check, and a single client that every call goes through. That is the infrastructure that lets you say no to a 10x cost increase before it ships, instead of explaining it to finance after the invoice arrives.

## References

- [Anthropic prompt caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [OpenAI prompt caching guide](https://platform.openai.com/docs/guides/prompt-caching)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [`tiktoken-rs` on GitHub](https://github.com/zurawiki/tiktoken-rs)
- [LLMLingua: compressing prompts for LLMs](https://github.com/microsoft/LLMLingua)
- [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
