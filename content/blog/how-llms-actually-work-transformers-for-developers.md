+++
title = "How LLMs actually work: transformers explained for developers"
date = 2025-10-03
description = "A mechanical walkthrough of what happens when an LLM generates one token: tokenization, embeddings, attention layers, softmax, sampling, and why hallucinations are baked in."

[taxonomies]
tags = ["ai", "llm", "transformers", "machine-learning"]
+++

You type `The capital of France is` into ChatGPT. About 200 milliseconds later, ` Paris` appears. Then ` and`. Then ` it`. Then ` is`. Each word is the result of roughly half a trillion floating-point multiplications, run through 80-some layers of a neural network, ending in a probability distribution over 100,000-ish possible next tokens.

This post is about what actually happens in those 200 milliseconds. Not the math derivations - the mechanics. By the end you should be able to read a sentence like "the model has 70B parameters and a 128k context window" and have a concrete picture of what those numbers refer to.

I will not be deriving backpropagation or explaining how the weights got trained. That is a separate and much longer story. We are looking at inference: the model is frozen, you give it text, it gives you the next token. Repeat.

<!-- more -->

## The mental model in one paragraph

An LLM is a function. Input: a sequence of integers (token IDs). Output: a probability distribution over the vocabulary, telling you how likely each possible next token is. To generate text, you sample one token from that distribution, append it to the input, and call the function again. That is it. Everything else is what happens inside the function.

The function is a stack of identical layers. Each layer takes a matrix (one row per token, one column per "feature") and produces another matrix of the same shape, but with the rows mixed up so that information from earlier tokens has been blended into later ones. After 32 or 80 such layers, the last row of the matrix has absorbed enough context that you can project it down to the vocabulary and ask "what comes next?"

Now let us walk through the steps.

## Step 1: Tokenization (BPE)

You hand the model a string. The model does not see strings. It sees integers. The conversion is done by a tokenizer, and modern tokenizers use a variant of Byte Pair Encoding (BPE).

BPE starts from raw bytes and greedily merges the most frequent adjacent pair, over and over, until you have a vocabulary of fixed size (typically 32k to 128k). The result is that common English words become single tokens, less common words split into a handful of subword pieces, and unusual sequences (Chinese characters, emoji, source code) fall back to individual bytes.

Concretely, for the OpenAI cl100k tokenizer:

```
"The capital of France is Paris"
 ->  [791, 6864, 315, 9822, 374, 12366]
 ->  ["The", " capital", " of", " France", " is", " Paris"]
```

Six words, six tokens. Notice the leading space in `" capital"` - in BPE tokenizers the space is part of the token, not a separator. This is why model output streams in chunks like ` and`, ` then`, ` because`, with the space attached.

A messier example:

```
"antidisestablishmentarianism"
 ->  ["ant", "idis", "establish", "ment", "arian", "ism"]
```

One word, six tokens. This is why "tokens" and "words" are not the same thing, and why the rule of thumb is roughly 0.75 words per token for English.

If you want to go deeper on how BPE actually merges, why WordPiece and SentencePiece differ, and what happens when a model meets a token it has never seen during training, I covered that in [Tokenization deep dive: BPE, WordPiece, SentencePiece](/blog/tokenization-deep-dive-bpe-wordpiece-sentencepiece/). For this post, "string in, list of integer IDs out" is enough.

## Step 2: Embeddings (integers to vectors)

The model has a learned matrix called the embedding table. Its shape is `(vocab_size, d_model)`. For Llama 3 70B, that is roughly `(128256, 8192)`. Each row is the vector that token gets associated with.

Token ID 791 ("The") becomes row 791 of the embedding table: a vector of 8192 floats. Token 6864 (" capital") becomes row 6864. Stack those six row vectors and you have a matrix of shape `(6, 8192)`. That is the input to the rest of the network.

These vectors are the model's only handle on what each token "means." They were not handcrafted; they emerged from training. After training, semantically related tokens land near each other in vector space - " king" minus " man" plus " woman" really does come out close to " queen", as the famous word2vec paper showed. The same logic applies inside an LLM, just at higher dimensionality and with richer relationships.

There is one more thing added at this step: positional information. The embedding tells the model what the token is, but not where it is in the sequence. Modern LLMs use rotary positional embeddings (RoPE), which rotate pairs of dimensions in each vector by an angle that depends on the position. The math is mildly involved but the effect is simple: after the rotation, two tokens at the same distance from each other produce the same dot product regardless of their absolute position. Position becomes relative, which generalizes better to long contexts.

For a concrete walkthrough of how embeddings encode meaning and how to use them outside an LLM context, see [Embeddings explained: turning text into vectors](/blog/embeddings-explained-turning-text-into-vectors/).

## Step 3: The transformer layers

Now the real work. The `(N, d_model)` matrix flows through a stack of layers. Each layer has the same structure:

```
x -> LayerNorm -> MultiHeadAttention -> add residual ->
     LayerNorm -> FeedForwardNetwork -> add residual -> output
```

Two sub-blocks per layer: attention and feedforward. The residual connections (`x + sublayer(x)`) are why you can stack 80 layers without the gradients dying during training. They also mean each layer is making a *correction* to the running representation, not replacing it.

**Attention** is where information moves between token positions. For each token, the model computes three projections (Query, Key, Value) of its current representation. The query of token i is dotted against the keys of all preceding tokens. The resulting scores are softmaxed into weights, and those weights are used to take a weighted sum of the values. The result replaces the row for token i.

In plain English: at every layer, every token gets to look back at every previous token, decide which ones are relevant to it right now, and pull in their information. Multi-head means this is done in parallel across (typically) 32 to 64 different "channels," each of which learns to track a different kind of relationship - syntactic agreement in one head, coreference in another, code structure in a third.

This is the operation that costs `O(N^2)` in sequence length and is the reason your 128k-context request gets expensive. I went into detail on what self-attention is actually computing, why we scale by sqrt(d_k), and why the K/V split is non-obvious, in [Understanding the attention mechanism](/blog/understanding-attention-mechanism-the-core-of-transformers/).

**Feedforward** is a per-token MLP, run independently on every row. Two linear layers with a nonlinearity in the middle (SiLU or GeLU in modern LLMs), expanding the hidden dimension by ~4x and then projecting back. This is where most of the parameters live - in a 70B model the FFN holds roughly two thirds of the weights. It is the model's "compute budget" for processing each token after the attention layer has done the mixing.

So the loop, layer by layer, is: mix information across positions (attention), then crunch each position individually (feedforward). Eighty times.

## Step 4: From hidden states to logits

After the last layer you have a matrix of shape `(N, d_model)`. To predict the next token, you only care about the last row - the one corresponding to the most recent input token. (During training, you predict from every position in parallel. At inference, only the final position matters.)

That last vector goes through one more linear projection, called the *unembedding* or *output head*, with shape `(d_model, vocab_size)`. The result is a vector of `vocab_size` floats, one per possible next token. These are called *logits*. Higher logit = the model thinks this token is more likely to come next.

In many models, the unembedding matrix is tied to (literally shares weights with) the embedding matrix from step 2. This is called weight tying. It saves parameters, and intuitively it makes sense: the same notion of "what this token is" should govern both how it is read in and how it is predicted.

## Step 5: Sampling (softmax, temperature, top-p)

Logits are not probabilities. They are arbitrary real numbers. To turn them into a distribution, you apply softmax:

```
P(token_i) = exp(logit_i / T) / sum_j exp(logit_j / T)
```

`T` is the **temperature**. At T=1, you get the raw distribution. At T<1, the distribution becomes peakier - high-logit tokens get even more probability mass, low-logit tokens get squashed. At T=0 (in practice, T very small), softmax collapses to argmax: deterministic, always picks the highest logit. At T>1, the distribution flattens out, so unlikely tokens get more chance.

This is why "lower temperature = more deterministic, higher temperature = more creative." It is literally how peaked the distribution is.

Once you have a probability over the full 128k vocabulary, you still need to sample one token. Naively sampling from the full distribution can produce nonsense, because the long tail of unlikely tokens, summed together, is non-trivial in mass. So we truncate.

**Top-k sampling** keeps only the k highest-probability tokens, renormalizes, and samples from those. Simple and easy to reason about. Common values: k=40 to 100.

**Top-p (nucleus) sampling** sorts tokens by probability, takes the smallest set whose cumulative probability exceeds p (typically 0.9 to 0.95), and samples from that. Adapts to the shape of the distribution: when the model is confident, the nucleus is small; when it is uncertain, it widens. Generally preferred over top-k.

You can also stack them: top-k filter, then top-p, then temperature. Most modern inference servers expose all three.

So if you ever wondered why setting `temperature=0` produces the same output every time, that is why: there is no randomness left. Argmax is deterministic.

## The autoregressive loop, end to end

Putting it all together, here is what generating ` Paris` after `The capital of France is` looks like:

1. Tokenize: `[791, 6864, 315, 9822, 374]` (5 tokens).
2. Embed: 5 vectors of length d_model, plus positional rotation. Matrix is `(5, d_model)`.
3. Run through L layers. Each layer reshapes the matrix in place using attention and FFN, but keeps shape `(5, d_model)`.
4. Take the last row (representation of token 374, " is"). Project to logits via the unembedding: vector of length 128256.
5. Apply temperature, top-p, softmax.
6. Sample. Out comes 12366 (" Paris").
7. Append 12366 to the input. You now have 6 tokens. Go back to step 2.

That is one forward pass per token. For a 70B model on an H100, expect ~50 tokens per second uncached. With KV-caching (where the keys and values from previous positions are saved across calls so attention does not recompute them), throughput jumps several times.

## Why LLMs hallucinate

Now you can see where hallucinations come from. They are not a bug in the sense of "we forgot a check." They are a structural property of step 5.

The model produces a probability distribution. That distribution is the model's best guess given (a) the prompt and (b) the patterns it learned during training. There is no separate module that goes "wait, do I actually know this?" Every step of inference, the model commits to *some* token. If the prompt is `Cite a paper by John Smith on quantum chromodynamics in 2003`, and no such paper appears strongly in its training data, the model still has to produce *something*. It will produce the token sequence that looks most paper-like - plausible authors, plausible journals, plausible DOIs - because that is what its training distribution rewards.

There is no internal flag for "I am making this up." The model is statistical text completion. If the most probable continuation under its weights is wrong, the output is wrong, and the model has no mechanism to know.

You can mitigate this with retrieval (give the model the actual document so the right answer becomes the most probable next tokens), with fine-tuning to abstain ("If you are not sure, say 'I do not know'"), or with calibration techniques. But the core issue - that the model always samples something - is architectural. It will not be fixed by making the model bigger.

## Context window: what it actually means

The context window is the maximum number of tokens N that the model can take as input in one forward pass. Llama 3.1 ships with 128k. GPT-4 Turbo has 128k. Claude has 200k+. Gemini reaches into the millions.

What sets this limit? Two things.

First, the positional encoding. RoPE was trained at some maximum length. Beyond that, the rotations start landing in territory the model has not seen, and quality degrades unless you do tricks like RoPE scaling.

Second, the quadratic cost. Attention computes an `N x N` matrix per layer per head. Doubling the context quadruples the attention compute and memory. The KV cache scales linearly in N, but for large N it dominates GPU memory: a 70B model at 128k context with bf16 KV cache eats ~40 GB just for the cache. This is why long-context inference is expensive, and why people are working on alternatives (sliding window attention, Mamba-style state-space models, sparse attention) that break the quadratic scaling.

Practically: when you fill up a 128k context, the model sees all 128k tokens, every single one of which has to be attended to by every later token at every layer. There is no "forgetting old stuff" - just compute and memory pressure that grows with how much you stuff in.

## What I left out

Plenty. Mixture of experts (where a sparse subset of FFN weights activates per token, used in models like Mixtral and DeepSeek). Grouped-query attention (a memory-saving variant where multiple query heads share K and V). Speculative decoding (using a small draft model to propose tokens and a big model to verify in parallel). Quantization (storing weights in 4-bit instead of 16-bit and dequantizing on the fly).

But the skeleton above - tokenize, embed, stack of attention+FFN layers, project to logits, sample - is what nearly every LLM you interact with is doing under the hood. Once you have that picture, the optimizations all become "okay, but how do we do this step faster" rather than "what is even happening."

The next time someone tells you GPT-4 is "just predicting the next word," you can say yes, technically, but here is what "predicting" involves: a hundred billion parameters arranged in roughly 80 attention-FFN sandwiches, processing 8192-dimensional vectors, mixing information across thousands of token positions, and producing a 128k-way probability distribution that gets sampled exactly once per word.

It is just text completion. There is just a lot of math behind "just".
