+++
title = "Fine-tuning vs RAG vs prompt engineering: when to use what"
date = 2025-01-23
description = "Three levers for customizing LLM behavior, what they actually cost, how to combine them, and the signs you picked the wrong one."

[taxonomies]
tags = ["ai", "llm", "rag", "fine-tuning"]
+++

Most "we need AI in the product" conversations eventually narrow to the same question. The base model is close but not right. It hallucinates our API names. It ignores our internal jargon. It does not know about the pricing change from last Tuesday. So what do we do?

There are three levers. They have overlapping marketing copy and very non-overlapping cost profiles. Pick the wrong one and you spend six weeks and five figures solving a problem that a better system prompt would have fixed. Pick the wrong one in the other direction and you spend six months producing brittle prompt spaghetti that a fine-tuned 8B model could have replaced in a weekend.

<!-- more -->

If you have not read [Multimodal AI: processing images, audio and text together](/blog/multimodal-ai-images-audio-text-together), that post covers the general shape of how you call a modern LLM from production code. This one is about what to do when the default behavior is not quite good enough.

## The three levers

1. **Prompt engineering.** Change what you send in. No training, no retrieval, no new weights.
2. **Retrieval-augmented generation (RAG).** Fetch relevant context at request time and stuff it into the prompt.
3. **Fine-tuning.** Change the model's weights so it produces different output for the same input.

They are not mutually exclusive. In production systems they are usually stacked. But they solve different problems and the failure modes are different enough that treating them as "tools for the same job" leads to bad decisions.

## Prompt engineering: the default, and often the answer

Prompt engineering is everything you can do by changing the request. System prompts, few-shot examples, chain-of-thought scaffolding, output format coercion, tool definitions, role play. It is by far the cheapest lever because the only thing you pay for is tokens.

For reference, as of mid-2026, input token prices on frontier APIs sit around $2-3 per million (Claude Sonnet 4.6, GPT-4o), with smaller models like Claude Haiku and GPT-4o-mini an order of magnitude cheaper. Output tokens cost 4-5x what input tokens cost. A well-crafted 1500-token system prompt used across a million requests is $3-4.50 in input tokens. A prompt with three well-chosen few-shot examples at 300 tokens each adds another $3.

Prompt caching changes this math dramatically. Anthropic's cache reads are 90% cheaper than fresh input, and OpenAI's prompt caching gives roughly 50% off reused prefixes. If your system prompt is stable across requests, most of its cost vanishes after the first call. This means long, detailed prompts are economically fine in 2026 in a way they were not in 2023.

What prompt engineering is good at:

- Changing *tone, style, format*. Making the model answer in JSON, in Polish, with bullets, without apologies, as a terse senior engineer.
- Teaching the model about *short, stable facts*. "Our product is called Foo. It supports x, y, z."
- *One-shot behaviors*. Give three examples of the transformation you want and the model extrapolates.
- Compositional tasks via *tool use*. Let the model call your functions instead of trying to know your database.

Where it runs out of room:

- **Context window budget.** If you need 50,000 tokens of domain knowledge in every request, even with caching you are paying for that window and slowing down responses.
- **Reliability degrades with prompt size.** Models get worse at following instructions buried in long prompts. This is the "lost in the middle" problem and it is still a problem even with 200K windows.
- **No memory across requests.** The model does not learn. Every call is from scratch.

If you have not tried to solve a problem with a better prompt, you probably should before you move on. An embarrassing number of "we need fine-tuning" requirements dissolve once someone writes a proper system prompt with three good examples.

## RAG: when the model needs knowledge it does not have

RAG is the architecture for "the model is smart but does not know our stuff." You maintain a corpus (docs, tickets, a product catalog, a knowledge base) and at request time you fetch the relevant chunks and paste them into the prompt.

The pipeline is boring and well understood:

1. At ingest, chunk your documents (typically 200-800 tokens per chunk, with overlap).
2. Embed each chunk with a text embedding model. OpenAI's `text-embedding-3-small` at 512 or 1536 dims costs around $0.02 per million tokens. Cohere, Voyage, and open-source alternatives like `bge-m3` are in a similar ballpark or free if self-hosted.
3. Store `(chunk_id, vector, text, metadata)` in a vector DB indexed with [HNSW](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood) or IVF.
4. At query time, embed the query, pull the top-K chunks (usually K=5 to K=20), optionally re-rank with a cross-encoder, and put them into the prompt as context.
5. The LLM answers using the retrieved context.

The economics look like this. For a 10 million token corpus, embedding it once costs roughly $0.20. Storage in a managed vector DB runs $20-100 per million vectors per month depending on provider. At query time you pay for one tiny embedding call (the query is maybe 30 tokens, so fractions of a cent) and the context tokens added to the main call (5 chunks at 500 tokens = 2500 extra input tokens, roughly $0.008 at Sonnet 4.6 rates).

What RAG is good at:

- **Freshness.** New documents show up in answers as soon as they are indexed. No retraining.
- **Attribution.** You can return the source chunks alongside the answer. Users (and auditors) see what the model read.
- **Scale.** You can have terabytes of corpus and still pull the relevant slice per query.

Where it fails:

- **Retrieval quality is the ceiling.** If HNSW returns irrelevant chunks, the LLM will confidently quote irrelevant chunks. Garbage in, eloquent garbage out.
- **Chunking is fiddly.** Too small and you lose context. Too large and you dilute relevance. Semantic chunking, sliding windows, hierarchical summaries are all rabbit holes.
- **Multi-hop reasoning.** "Which of our customers signed in 2024 and have an open P0 ticket in the billing module?" is not a retrieval problem, it is a query problem. RAG on top of unstructured text will often miss it. You want SQL or a graph query and a tool-using agent to orchestrate.
- **Not for style or behavior.** RAG adds facts to the prompt. It does not teach the model to answer in a particular voice or format any more reliably than a system prompt would.

## Fine-tuning: when you want to change how the model behaves

Fine-tuning is the heavy lever. You collect a dataset of `(input, desired output)` pairs and run gradient updates on the model's weights. The result is a new model that is structurally better at your task.

In 2026 the practical options are:

- **API-hosted fine-tuning.** OpenAI and Anthropic both offer supervised fine-tuning on their smaller models (GPT-4o-mini, Claude Haiku). You upload a JSONL, they train a LoRA adapter, you call your custom model via an ID. Training cost for GPT-4o-mini is around $3 per million training tokens. Inference costs a modest premium over the base model (typically 1.5-2x).
- **Open-weight fine-tuning.** Take Llama 3.1 8B, Qwen 2.5, or Mistral, train a LoRA adapter on your data with `peft` or `axolotl`, deploy on vLLM. LoRA on an 8B model fits on a single H100 and finishes in a few hours for a dataset of tens of thousands of examples. Inference costs drop to whatever your GPU hours cost, which at scale can be 10-100x cheaper per token than calling frontier APIs.
- **Preference tuning (DPO, KTO, ORPO).** After SFT, align the model to human preferences with pairs of good/bad responses. This is what turns a "competent but weird" model into one that sounds right.

What fine-tuning is good at:

- **Style and format consistency.** If you need every output to be valid JSON in a specific schema, a fine-tuned small model will do this more reliably than a giant prompt-engineered frontier model.
- **Latency and cost at scale.** A fine-tuned 8B model self-hosted is the right answer for anything with millions of requests per day and tight latency budgets.
- **Narrow, specialized tasks.** Classification, structured extraction, domain translation. Tasks where the input and output shapes are stable.
- **Replacing long prompts.** If your system prompt is 5000 tokens of "here is how to behave," a fine-tune can bake that behavior into the weights and drop the prompt to 100 tokens.

Where it fails:

- **You need data.** A few hundred high-quality examples is the minimum. A few thousand is comfortable. The dataset is the project; the training script is an afternoon.
- **Facts change.** Fine-tuning teaches behavior, not knowledge. If you bake "our pricing is $29/mo" into the weights, you are in trouble when pricing changes.
- **Evaluation is hard.** A prompt change is easy to A/B. A fine-tune requires a held-out eval set, ideally with human labels, and you need to rerun it every time you retrain.
- **Rollback is slow.** If the fine-tuned model regresses, you redeploy the old one. If a prompt regresses, you edit a string.

## The decision tree

In practice, the ordering I keep landing on:

1. **Try a better prompt first.** Write a clear system prompt, add three good few-shot examples, force structured output. This is free and fast. A surprising number of problems stop here.
2. **If the model needs facts it does not have, add RAG.** Anything that is "look it up" by nature. Product docs, recent events, internal data.
3. **If the model needs to behave differently for the same input, fine-tune.** Anything that is "respond in this specific shape" or "handle this specific classification problem."
4. **If you need both: fine-tune for behavior, RAG for knowledge.** This is the end state for most serious production systems.

Cost per 1M requests, rough order of magnitude, assuming a 500-token input and 200-token output:

| Approach | Inference cost | Setup cost | Time to ship |
|---|---|---|---|
| Prompt engineering (Sonnet 4.6) | $3,500 | $0 | hours |
| RAG (Sonnet 4.6 + vector DB) | $4,000 + infra | $100-1000 | days |
| Fine-tuned Haiku | $1,200 | $100-500 | days |
| Fine-tuned 8B self-hosted | $50-200 | $500-5000 | weeks |
| Fine-tuned 8B + RAG | $100-300 | $1000-5000 | weeks |

Quality, loosely ranked for a well-scoped task:

- Open-ended reasoning: frontier model with good prompt > fine-tuned small model
- Structured extraction on a known schema: fine-tuned small model > frontier model
- Answering questions about a corpus: RAG on a frontier model > fine-tune alone
- Classifying into a fixed taxonomy: fine-tuned small model, by a mile

## Combining them: fine-tuned model + RAG

The stacked version looks like this. You fine-tune a small model (Haiku, Llama 3.1 8B) to speak in the right voice, respect the right output format, and follow your task-specific instructions without being told. Then at request time, you retrieve context from your RAG index and paste it into the prompt.

The fine-tune absorbs everything that is stable about the task. The RAG layer absorbs everything that changes. Your prompt stays short because the behavior is baked in and the facts come from retrieval.

This is the shape of most production systems that have been running for more than a year. The team started with prompt engineering, added RAG when the "model does not know our stuff" complaints piled up, and fine-tuned when inference costs got painful or quality plateaued on a specific task.

## How each approach fails, in one line each

- **Prompt engineering fails** when the behavior you need depends on knowledge the model does not have, or when prompt size grows beyond what attention can reliably handle.
- **RAG fails** when retrieval brings back irrelevant chunks, when the answer requires reasoning across many documents, or when the question is better answered by a database query than a semantic search.
- **Fine-tuning fails** when your dataset is too small, when the underlying facts change faster than you can retrain, or when you skip evaluation and ship a model that is subtly worse on the 10% of cases you did not think about.

The metaskill is noticing which one is failing. A fine-tune that hallucinates facts is not a fine-tune problem, it is a RAG problem. A RAG pipeline that returns the wrong tone is not a retrieval problem, it is a prompt or fine-tune problem. Misattribution of the failure mode is how teams end up spending a quarter training a model that did not need to be trained.

## What to take away

Three levers. Not three competing frameworks, three different tools.

Prompt engineering is where you start and often where you stay. RAG is the right answer when the gap between the model and your product is knowledge. Fine-tuning is the right answer when the gap is behavior and you have the data to close it.

If you have the time to build only one of these properly, build evaluation. The approach that wins is the one you can measure.

## References

- [OpenAI fine-tuning docs](https://platform.openai.com/docs/guides/fine-tuning)
- [Anthropic prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685)
- [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
- [axolotl](https://github.com/axolotl-ai-cloud/axolotl) for open-weight fine-tuning
- [vLLM](https://github.com/vllm-project/vllm) for serving fine-tuned models
