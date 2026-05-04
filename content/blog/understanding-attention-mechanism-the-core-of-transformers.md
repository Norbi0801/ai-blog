+++
title = "Understanding attention mechanism: the core of transformers"
date = 2025-03-04
description = "What self-attention actually computes, why Q/K/V are three projections of the same thing, and how scaled dot-product and multi-head attention fit together."

[taxonomies]
tags = ["ai", "machine-learning", "transformers", "nlp"]
+++

Every time you use an LLM, type a message into a chatbot, or ask a model to summarize a PDF, the core operation running under the hood is attention. Not the convolutions from CNN papers, not the recurrence from LSTMs. A matrix multiply, a softmax, another matrix multiply. That is attention, and it is what made transformers work.

The 2017 paper "Attention Is All You Need" is eight years old and has been cited over 150,000 times. It is worth understanding the mechanism at the level of "what values are flowing through which matrices" rather than "black box that reads text." This post does that. No prior transformer knowledge assumed, but some linear algebra familiarity will help.

<!-- more -->

## The problem attention solves

Before attention, the dominant way to process sequences was RNNs and LSTMs: read token 1, update hidden state, read token 2, update hidden state, and so on. The hidden state is the model's only memory of the past. To incorporate information from token 5 into the processing of token 50, that information has to survive 45 update steps without being overwritten. In practice, long-range dependencies degrade fast.

Convolutions do better: each layer has a fixed receptive field, and you can stack layers to grow it. But the number of layers you need to connect two positions scales with the distance between them, so very long dependencies still require deep networks.

Attention lets every position in a sequence look at every other position in a single step. No recurrence, no convolution. The cost is quadratic in sequence length (we will see why), but the gain is that the number of operations between any two tokens is constant.

If you are not familiar with embedding vectors, go read about them somewhere else first. Short version: each token in the input (roughly, each subword) is mapped to a fixed-size vector, say R^512 or R^4096. These vectors are what flows through the attention layers.

## Self-attention in one paragraph

You have N input vectors. For each position i, you want to compute a new vector that is a weighted sum of all N input vectors, where the weights depend on how "relevant" each other position is to position i. The weights should sum to 1 (so the output stays on a sensible scale) and should be learned, not hardcoded.

That is the whole idea. Everything else is plumbing to make it learnable with gradient descent.

## Q, K, V: three views of the same input

The way transformers parameterize this is with three linear projections of the input. Given input matrix X of shape `(N, d_model)`, we compute:

```
Q = X @ W_Q    # queries,  shape (N, d_k)
K = X @ W_K    # keys,     shape (N, d_k)
V = X @ W_V    # values,   shape (N, d_v)
```

W_Q, W_K, W_V are learned weight matrices. In the original paper, d_model = 512, d_k = d_v = 64 (per head, more on that later).

Intuition, which is worth spending a moment on:

- **Query (Q)** for position i asks: "given what I currently represent, what kind of information am I looking for?"
- **Key (K)** for position j answers: "here is a summary of what I contain, for lookup purposes."
- **Value (V)** for position j provides: "here is the actual content I will contribute if you decide to attend to me."

The reason to split K and V is subtle. The key is what you match against; the value is what you aggregate. Separating them lets the model address by one representation and retrieve a different one. Think of K as a hash and V as a payload.

It is the same input X going into all three projections. The model learns three different "views" of each token, one for asking, one for advertising, one for delivering.

## Scaled dot-product attention

Given Q, K, V, attention is:

```
Attention(Q, K, V) = softmax(Q @ K^T / sqrt(d_k)) @ V
```

Let me unpack this piece by piece. `Q @ K^T` has shape `(N, N)`. Entry `(i, j)` is the dot product of query i with key j. That is your unnormalized similarity score. If query and key point in similar directions, the dot product is high. If they are orthogonal, zero. If they point opposite ways, negative.

Divide by `sqrt(d_k)`. Why? Dot products of two random vectors in R^d have variance that scales with d. Without the scaling, the variance of the pre-softmax scores grows with dimension, pushing softmax into regions where gradients vanish (softmax saturates when one logit dominates). Dividing by `sqrt(d_k)` keeps the variance roughly constant regardless of head size. This is not a cosmetic detail; training stability depends on it.

Apply softmax row-wise. Each row now sums to 1. Row i is "how much does position i attend to each of positions 1 through N." These are the attention weights.

Multiply by V (shape `(N, d_v)`). The output has shape `(N, d_v)`. Row i is a weighted sum of all value vectors, weighted by row i of the attention matrix.

## A tiny concrete example

Let N=3, d_k=2 for readability. Suppose after projection:

```
Q = [[1, 0],     K = [[1, 0],     V = [[1, 0],
     [0, 1],          [1, 1],          [0, 1],
     [1, 1]]          [0, 1]]          [1, 1]]
```

Compute Q @ K^T:

```
         K1    K2    K3
Q1 [[1,0] · [1,0]=1, [1,0] · [1,1]=1, [1,0] · [0,1]=0]
Q2 [[0,1] · [1,0]=0, [0,1] · [1,1]=1, [0,1] · [0,1]=1]
Q3 [[1,1] · [1,0]=1, [1,1] · [1,1]=2, [1,1] · [0,1]=1]
```

Divide by `sqrt(2) ~ 1.414`:

```
[[0.71, 0.71, 0.00],
 [0.00, 0.71, 0.71],
 [0.71, 1.41, 0.71]]
```

Softmax row-wise (exponentiate, normalize):

```
row 1: [0.42, 0.42, 0.16]
row 2: [0.16, 0.42, 0.42]
row 3: [0.21, 0.58, 0.21]
```

Each row sums to 1. Row 3, which had the highest-magnitude query, also has the sharpest distribution (0.58 concentrated on position 2). The output for position 3 will be mostly V[2] with small contributions from V[1] and V[3].

Multiply by V to get the three output vectors. Try it on paper, it takes a minute and is worth more than reading another paragraph about it.

## Why it captures long-range dependencies

Look at the shape of `Q @ K^T`: N by N. Every token has a score against every other token in a single operation. The attention from position 1 to position 500 is not mediated by positions 2 through 499. The gradient signal from position 1's output to position 500's value flows through one matmul and one softmax, not through 499 hidden state updates.

This is why transformers handle long contexts where LSTMs struggled. It is also why attention is expensive: N by N means memory and compute scale with N squared. For N=32k tokens, that is a billion entries just in the attention matrix, per head, per layer. This quadratic wall is the entire reason "efficient attention" is an active research area (FlashAttention, linear attention, sliding window attention, state space models like Mamba).

## Multi-head attention: multiple perspectives

One attention operation gives you one set of relationships. But what if you want to model both syntactic and semantic links, or short-range and long-range ones, at the same time?

Multi-head attention runs h copies of scaled dot-product attention in parallel, each with its own learned W_Q, W_K, W_V. The outputs are concatenated and projected:

```
head_i = Attention(X @ W_Q_i, X @ W_K_i, X @ W_V_i)
MHA(X) = concat(head_1, ..., head_h) @ W_O
```

In the original paper, d_model=512 and h=8, with d_k = d_v = 64. Note 8 * 64 = 512, so after concatenation you are back to d_model dimensions. The total parameter count of multi-head attention is roughly the same as a single attention with d_k=512: you paid nothing extra in compute, but you got h different attention patterns.

In practice, different heads specialize. People have published analyses showing some heads act like coreference resolvers (pronouns attend to their antecedents), some attend to the previous token, some attend to punctuation. You do not program this; it emerges from training.

Here is what a minimal multi-head attention looks like in PyTorch, stripped of bells and whistles:

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model=512, n_heads=8):
        super().__init__()
        assert d_model % n_heads == 0
        self.d_model = d_model
        self.n_heads = n_heads
        self.d_k = d_model // n_heads

        self.W_q = nn.Linear(d_model, d_model, bias=False)
        self.W_k = nn.Linear(d_model, d_model, bias=False)
        self.W_v = nn.Linear(d_model, d_model, bias=False)
        self.W_o = nn.Linear(d_model, d_model, bias=False)

    def forward(self, x, mask=None):
        B, N, _ = x.shape
        # Project and reshape: (B, N, d_model) -> (B, n_heads, N, d_k)
        q = self.W_q(x).view(B, N, self.n_heads, self.d_k).transpose(1, 2)
        k = self.W_k(x).view(B, N, self.n_heads, self.d_k).transpose(1, 2)
        v = self.W_v(x).view(B, N, self.n_heads, self.d_k).transpose(1, 2)

        scores = (q @ k.transpose(-2, -1)) / (self.d_k ** 0.5)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        weights = F.softmax(scores, dim=-1)
        out = weights @ v  # (B, n_heads, N, d_k)

        out = out.transpose(1, 2).contiguous().view(B, N, self.d_model)
        return self.W_o(out)
```

The `view` and `transpose` calls are how you run h independent attention operations without writing a loop. You reshape `(B, N, d_model)` into `(B, N, n_heads, d_k)`, transpose to `(B, n_heads, N, d_k)`, and then the final matmul broadcasts across the heads dimension.

In modern codebases you would replace the body of forward with a call to `torch.nn.functional.scaled_dot_product_attention` which dispatches to FlashAttention-2 or FlashAttention-3 depending on hardware. The mathematical result is identical; the memory access pattern is dramatically better.

## Positional encoding: attention is order-blind

There is a problem with the formulation above that most people miss the first time. Look at `softmax(Q @ K^T / sqrt(d_k)) @ V`. If you permute the rows of X, the output gets permuted the same way. The operation is equivariant to permutations of the input. Attention does not know anything about order.

For text, that is a disaster. "The dog chased the cat" and "The cat chased the dog" would produce the same token representations modulo permutation.

The fix is to inject positional information into the input embeddings before they ever reach attention. The original paper used sinusoidal positional encodings:

```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

Each position gets a unique d_model-dimensional vector, which is added elementwise to the token embedding. The choice of sinusoids (rather than learned embeddings) gives two nice properties: the encoding extends naturally to sequences longer than those seen at training time, and relative offsets can be expressed as linear functions of the encodings (because of sum-of-angles identities).

Modern models have mostly moved past absolute sinusoidal encoding:

- **Learned absolute**: just a lookup table of position vectors. Used in GPT-2, BERT.
- **Relative position bias**: add a learned bias to the attention scores based on `j - i`. T5 uses this.
- **RoPE (Rotary Position Embedding)**: rotate Q and K by an angle that depends on position. LLaMA, Mistral, Qwen, and most 2024 and later models use RoPE. It is beautifully simple and encodes relative positions naturally.
- **ALiBi**: add a fixed linear bias that decays with distance. Used in some long-context models because it extrapolates well.

If you want to get into why RoPE dominates, the short version is: it bakes relative position into the dot product itself without needing a separate bias, and it extrapolates reasonably well when you scale context length beyond training. Read the RoFormer paper and you will see that the "rotation" trick is just complex-number multiplication disguised as 2x2 block matrices.

## Causal masking

For decoder models (anything that generates text, which includes all autoregressive LLMs), attention at position i must not see positions i+1, i+2, and so on, or training would trivially leak the answer.

This is implemented by a mask. Before softmax, set the upper triangle of the score matrix to `-inf`. After exponentiation those become zero, and the row renormalizes over only the allowed positions.

```python
mask = torch.tril(torch.ones(N, N))  # 1 below diagonal, 0 above
scores = scores.masked_fill(mask == 0, float('-inf'))
```

Encoder-only models (BERT, embedders) use bidirectional attention with no mask. Encoder-decoder models (T5, original transformer) use causal masking only on the decoder side, and cross-attention from decoder to encoder is unmasked.

## Visualizing attention weights

The `(N, N)` attention matrix after softmax is a probability distribution that you can render as a heatmap. Row i shows what position i is attending to. It is the single most informative visualization you can produce for a transformer, and it is almost free: you just need to grab the tensor from the forward pass.

If you run BERT on "the bank of the river" and look at heads in middle layers, you will typically see "bank" attending strongly to "river", which is the model doing word-sense disambiguation. In GPT models with causal masking, the diagonal is always heaviest (a token can usually predict itself reasonably well), and off-diagonal structure reveals things like induction heads, which attend from a token to the position right after the last occurrence of that same token, enabling in-context learning.

The `bertviz` library is probably the easiest way to look at attention in HuggingFace models. For your own models, saving the softmax output and rendering with matplotlib is 10 lines of code.

## What the math is really doing

Strip away the matrices and attention is a soft, differentiable lookup table.

A hard lookup table takes a query, finds the one best-matching key, and returns its value. Attention takes a query, scores all keys by dot-product similarity, normalizes the scores with softmax, and returns a weighted combination of all values. As the scores get sharper (if you scale them up), attention approaches a hard lookup. As the scores get flatter, it approaches a uniform average.

This is why it works for such different domains. The same mechanism that lets a language model route "bank" to "river" also lets a vision transformer route a patch in the upper-left to a patch in the lower-right, and lets a multimodal model route an image patch to a text token. The substrate is always the same: N vectors projected three ways, N by N score matrix, softmax, weighted sum.

Once you see attention as differentiable content-based retrieval, the rest of the transformer makes sense. The feedforward layers between attention blocks are just per-token MLPs. The residual connections and layer norms are there to keep gradients well-behaved. The stack of layers gives you iterated refinement, each layer looking at a slightly more abstract representation than the last.

It is a lot of machinery built around one very good idea.
