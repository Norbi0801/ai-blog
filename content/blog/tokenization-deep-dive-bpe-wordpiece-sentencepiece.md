+++
title = "Tokenization deep dive: BPE, WordPiece, SentencePiece"
date = 2025-03-01
description = "How LLMs actually chop text into tokens, why it matters for your bill and context window, and a 100-line BPE implementation in Rust."

[taxonomies]
tags = ["ai", "nlp", "rust", "llm"]
+++

If you have ever wondered why the string `"strawberry"` is three tokens to GPT-4 but `"🍓"` is two, why GPT charges you roughly 4x more to process Japanese than English, or why `count_r_in_strawberry()` is weirdly hard for models, the answer is tokenization. It is the layer between raw text and the tensors that attention operates on, and it determines a surprising amount of model behavior.

This post goes through the algorithms that power modern tokenizers: byte pair encoding (BPE), WordPiece, SentencePiece, and the Unigram LM. We will look at concrete token counts across languages, why 100,000-entry vocabularies are the current sweet spot, and finish with a working BPE trainer and encoder in roughly 100 lines of Rust.

<!-- more -->

## Why tokenize at all

Neural networks operate on fixed-size vectors of numbers. Text is a variable-length stream of Unicode code points. You need a deterministic function that turns one into the other.

The three historical options:

- **Character level**: one token per character. Vocabulary is tiny (maybe a few hundred entries for ASCII plus the most common Unicode ranges). Sequences get very long. The model has to learn what a "word" is from scratch. Memory cost of attention is O(N^2) in sequence length, which is brutal when your sequences are 4x longer than they need to be.
- **Word level**: split on whitespace, one token per word. Vocabulary explodes (English has maybe 200,000 words in common use, and that ignores inflections, typos, proper nouns, and every other language). Out-of-vocabulary words become `<UNK>`, which is terrible.
- **Subword level**: split into pieces smaller than words but bigger than characters. This is what everyone actually does.

Subword tokenization hits a nice middle ground: vocabulary stays manageable (30k to 200k entries), sequences are reasonably short, and there is no out-of-vocabulary problem because you can always fall back to characters or bytes.

## Byte pair encoding, step by step

BPE was originally a data compression algorithm from 1994. Rico Sennrich et al. repurposed it for NMT in 2015, and it has been the default for language models ever since (GPT-2, GPT-3, GPT-4, LLaMA, Mistral, Qwen: all BPE variants).

The idea is embarrassingly simple:

1. Start with every character (or every byte) as its own token.
2. Find the most frequent adjacent pair of tokens in your corpus.
3. Merge that pair into a new token. Add it to the vocabulary.
4. Repeat until you have the vocabulary size you want.

That is the entire algorithm. Let me walk through the canonical example from the Sennrich paper with a tiny corpus:

```
low     (5 times)
lower   (2 times)
newest  (6 times)
widest  (3 times)
```

We start with the word ends marked somehow. In the original paper they append `</w>`; modern byte-level BPE just treats spaces as regular bytes and does not do this. I will skip the end marker for clarity.

Initial state: each word is a sequence of single-character tokens.

```
l o w           x5
l o w e r       x2
n e w e s t     x6
w i d e s t     x3
```

Count adjacent pairs across the corpus, weighted by word frequency:

```
(l, o): 5 + 2 = 7
(o, w): 5 + 2 = 7
(w, e): 2 + 6 = 8
(e, r): 2
(n, e): 6
(e, w): 6
(e, s): 6 + 3 = 9
(s, t): 6 + 3 = 9
(w, i): 3
(i, d): 3
(d, e): 3
```

Tie between `(e, s)` and `(s, t)` at 9. Break ties arbitrarily. Pick `(e, s)`, merge into new token `es`:

```
l o w           x5
l o w e r       x2
n e w es t      x6
w i d es t      x3
```

Recount. Now `(es, t)` has frequency 9. Merge to `est`:

```
l o w           x5
l o w e r       x2
n e w est       x6
w i d est       x3
```

Next most frequent pair: `(l, o)` with 7. Merge to `lo`:

```
lo w            x5
lo w e r        x2
n e w est       x6
w i d est       x3
```

And so on. After enough merges, common sequences like `low`, `est`, `newest`, `widest` all end up as single tokens. Rare words stay as short sequences of small tokens. The algorithm naturally allocates vocabulary slots to frequent patterns and leaves rare patterns as combinations of smaller pieces.

The result is a list of merge rules (an ordered list of pairs to combine) plus a vocabulary. To encode new text, you split into starting units and apply the merges in order. To decode, you just concatenate the byte strings the tokens map to.

## Byte-level BPE

Sennrich's original BPE worked on Unicode characters. GPT-2 introduced a twist: run BPE on raw UTF-8 bytes instead. Your initial alphabet is exactly 256 entries (every possible byte). Every string is representable. No `<UNK>`, no preprocessing worries about normalization, no issues with rare CJK characters blowing up the vocabulary.

The downside is that a single emoji like `🍓` is encoded as 4 UTF-8 bytes, and if `🍓` is not frequent enough in training to earn a merge, you pay 4 tokens for one character. This is why `"🍓"` is 2 or 3 tokens in most modern tokenizers but `"strawberry"` is 1 or 2.

GPT-2 made one further tweak: they applied a bijective byte-to-unicode map so that the byte 0x00 does not render as a null character in debug output. It is purely cosmetic, and it is why if you dump GPT-2's vocabulary you see weird characters like `Ġ` (which represents the byte 0x20, i.e. a space).

The practical implication: a leading space is part of the token. `" world"` and `"world"` are different tokens in GPT. This is why prompt engineering sometimes involves being careful about whether you put a space before a continuation.

## WordPiece and Unigram LM

WordPiece is what BERT uses. It is BPE's close cousin, with one key difference in how it picks merges. BPE picks the pair with highest raw frequency. WordPiece picks the pair that maximizes the likelihood of the corpus under a language model where each token appears independently:

```
score(x, y) = freq(xy) / (freq(x) * freq(y))
```

This favors pairs where both components are relatively rare on their own but frequent together (like `##ing` in `walking`). The `##` prefix in BERT's vocabulary indicates "this piece is attached to the previous token with no space." You see `walking` tokenized as `walk` + `##ing`.

**Unigram LM** (the core algorithm behind SentencePiece's default mode) goes the other direction entirely. Instead of starting small and merging, you start with a huge vocabulary (every substring up to some length), compute each piece's probability, and iteratively drop the pieces whose removal costs the corpus the least likelihood. You keep pruning until you hit your target vocab size.

The advantage of Unigram is that for any new input, there are many possible segmentations, and each has a probability. You can sample (this is called "subword regularization" and helps generalization) or pick the most likely one. BPE and WordPiece are deterministic once trained.

## SentencePiece, the library

SentencePiece is Google's implementation. It is worth separating the library from the algorithms because SentencePiece supports both BPE and Unigram modes (and a few others), and people sometimes say "SentencePiece tokenizer" when they mean "Unigram" and vice versa.

The big contribution of SentencePiece is treating input as a raw Unicode string with zero preprocessing. Traditional BPE assumed you had already split on whitespace. SentencePiece replaces spaces with the special character `▁` (U+2581, "lower one-eighth block") and tokenizes the resulting sequence. This makes it language-agnostic: Japanese and Chinese do not use spaces to separate words, and SentencePiece happily chews them the same way it chews English.

LLaMA, Mistral, Gemma, and most non-OpenAI modern models use SentencePiece BPE. OpenAI's `tiktoken` uses byte-level BPE, which they rolled themselves in Rust (you can browse the code [here](https://github.com/openai/tiktoken)).

## Why you should care: cost, context, language

Every API you call with a Claude or GPT model charges per token. Every context window is measured in tokens. How your text tokenizes is how much you pay and how much fits.

Here are some rough numbers for current OpenAI tokenizers (`cl100k_base`, used by GPT-4):

| Input | Characters | Tokens | Chars/token |
|-------|-----------|--------|-------------|
| "Hello, world!" | 13 | 4 | 3.25 |
| "The quick brown fox jumps over the lazy dog." | 44 | 10 | 4.4 |
| 1 page of English Wikipedia | ~3000 | ~750 | ~4.0 |
| Same page in Spanish | ~3000 | ~900 | ~3.3 |
| Same page in Polish | ~3000 | ~1300 | ~2.3 |
| Same page in Japanese | ~1500 | ~1100 | ~1.4 |
| Same page in Korean | ~1500 | ~1300 | ~1.15 |

English gets roughly 4 characters per token. Slavic languages with rich morphology (Polish, Russian, Czech) get 2 to 3. Languages written without spaces and with non-Latin scripts can drop below 1.5, and for CJK the ratio depends heavily on which characters appear. Something like "🏳️‍🌈" (the rainbow flag emoji, a ZWJ sequence) is a full 14 UTF-8 bytes and can come out to 6 or 7 tokens.

This is not just a pricing issue. A 128k token context window holds roughly 350 pages of English text, maybe 200 pages of Polish, and only 100-150 pages of Japanese. If you are building an app that runs on non-English text, your effective context budget is smaller than the marketing number suggests.

The cause is simple: BPE merges are learned from training data, and the vast majority of OpenAI's training data is English. Frequent English words earn single-token representations. Rarer patterns in other languages don't.

There is an easy way to check: install `tiktoken` and run it.

```python
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")
print(len(enc.encode("Cześć, jak się masz?")))  # ~10 tokens
print(len(enc.encode("Hello, how are you?")))    # 6 tokens
```

Same semantic content, 66% more tokens in Polish.

## Why 100k is the magic vocabulary size

Vocabulary size is a hyperparameter. BERT used 30k. GPT-2 used 50k. GPT-4 (`cl100k_base`) uses about 100k. LLaMA 3 bumped to 128k. Qwen 2 went to 150k.

Bigger vocab, shorter sequences (good: attention is quadratic in length, so shorter is much cheaper at inference time). But bigger vocab also means a bigger embedding matrix (the input embedding layer is `vocab_size * d_model`, and so is the output projection before softmax). For a 70B parameter model with d_model=8192, going from 32k to 128k vocab adds (128000 - 32000) * 8192 * 2 ~= 1.5B parameters. That is not nothing.

The sweet spot is where the savings from shorter sequences at inference time outweigh the embedding cost. For very large models deployed heavily, pushing vocab up makes sense. For small models trained on a budget, smaller vocab is fine.

## Implementing BPE from scratch in Rust

Here is a minimal byte-level BPE trainer and encoder. Compile with Rust 1.74 or later, no external crates.

```rust
use std::collections::HashMap;

type Token = u32;

pub struct Bpe {
    // Merges in the order they were learned. Encoding applies them in order.
    merges: Vec<((Token, Token), Token)>,
    // Token id -> the raw bytes it expands to. First 256 entries are the bytes.
    vocab: Vec<Vec<u8>>,
}

fn get_stats(words: &[Vec<Token>], freqs: &[u64]) -> HashMap<(Token, Token), u64> {
    let mut counts = HashMap::new();
    for (word, &freq) in words.iter().zip(freqs) {
        for pair in word.windows(2) {
            *counts.entry((pair[0], pair[1])).or_insert(0) += freq;
        }
    }
    counts
}

fn merge_pair(word: &[Token], pair: (Token, Token), new_id: Token) -> Vec<Token> {
    let mut out = Vec::with_capacity(word.len());
    let mut i = 0;
    while i < word.len() {
        if i + 1 < word.len() && word[i] == pair.0 && word[i + 1] == pair.1 {
            out.push(new_id);
            i += 2;
        } else {
            out.push(word[i]);
            i += 1;
        }
    }
    out
}

pub fn train(corpus: &str, num_merges: usize) -> Bpe {
    // 1. Pre-tokenize on whitespace, count word frequencies.
    let mut freq_map: HashMap<&str, u64> = HashMap::new();
    for w in corpus.split_whitespace() {
        *freq_map.entry(w).or_insert(0) += 1;
    }

    // 2. Initialize vocab with every possible byte.
    let mut vocab: Vec<Vec<u8>> = (0..=255u8).map(|b| vec![b]).collect();

    // 3. Each word becomes a sequence of byte-token ids.
    let mut words: Vec<Vec<Token>> = freq_map
        .keys()
        .map(|w| w.bytes().map(|b| b as Token).collect())
        .collect();
    let freqs: Vec<u64> = freq_map.values().copied().collect();

    let mut merges: Vec<((Token, Token), Token)> = Vec::new();

    // 4. Greedy merges.
    for _ in 0..num_merges {
        let stats = get_stats(&words, &freqs);
        let Some((&pair, _)) = stats.iter().max_by_key(|&(_, c)| *c) else {
            break;
        };

        let new_id = vocab.len() as Token;
        let mut new_bytes = vocab[pair.0 as usize].clone();
        new_bytes.extend_from_slice(&vocab[pair.1 as usize]);
        vocab.push(new_bytes);

        merges.push((pair, new_id));
        for w in &mut words {
            *w = merge_pair(w, pair, new_id);
        }
    }

    Bpe { merges, vocab }
}

impl Bpe {
    pub fn encode(&self, text: &str) -> Vec<Token> {
        let mut out = Vec::new();
        for word in text.split_whitespace() {
            let mut tokens: Vec<Token> = word.bytes().map(|b| b as Token).collect();
            for (pair, new_id) in &self.merges {
                tokens = merge_pair(&tokens, *pair, *new_id);
            }
            out.extend(tokens);
        }
        out
    }

    pub fn decode(&self, tokens: &[Token]) -> String {
        let mut bytes = Vec::new();
        for &t in tokens {
            bytes.extend_from_slice(&self.vocab[t as usize]);
        }
        String::from_utf8_lossy(&bytes).into_owned()
    }
}

fn main() {
    let corpus = "low low low low low lower lower \
                  newest newest newest newest newest newest \
                  widest widest widest";
    let bpe = train(corpus, 10);

    for (i, tok) in bpe.vocab.iter().enumerate().skip(256) {
        println!("{}: {:?}", i, String::from_utf8_lossy(tok));
    }

    let encoded = bpe.encode("lowest newest");
    println!("encoded: {:?}", encoded);
    println!("decoded: {:?}", bpe.decode(&encoded));
}
```

Run it and you will see the first new tokens are things like `es`, `est`, `low`, `newest`, `widest`, which is exactly what the pen-and-paper trace predicted.

A few things this implementation is missing that a real production tokenizer would have:

- **Pre-tokenization with a regex**: `tiktoken` uses a regex like `'s|'t|'re|'ve|'m|'ll|'d| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+` to split text into "word-like" chunks before BPE runs. This keeps numbers, punctuation, and contractions separate from letters, which improves tokenization quality enormously.
- **Space handling**: byte-level BPE treats leading spaces as part of the token. My simplified version loses spaces during `split_whitespace`. A real version keeps them.
- **Speed**: `get_stats` rebuilds the full pair count every iteration, which is O(N) in corpus size per merge. Real implementations maintain an incremental data structure (priority queue of pair counts that gets updated as merges happen). `tiktoken` pushes the hot encoding loop into Rust and gets roughly 3-6x the throughput of HuggingFace's `tokenizers` library on typical workloads.
- **Tie-breaking**: picking the max element of a HashMap is non-deterministic in Rust (because iteration order depends on the hasher). Real implementations sort by frequency then by token id to make training reproducible.

## Quick guide to libraries

- **`tiktoken`**: OpenAI's byte-level BPE, Rust with Python bindings. Use it if you are talking to OpenAI APIs or want a fast byte-level BPE.
- **`sentencepiece`**: Google's C++ library with Python bindings. Supports BPE and Unigram. What you want for training new tokenizers from scratch on multilingual corpora.
- **`tokenizers`** (HuggingFace): Rust, with Python bindings. Supports basically every variant. Good default for most work inside the HF ecosystem.
- **`minbpe`** (Andrej Karpathy): Pedagogical Python BPE. Worth reading if you want to see a clean reference implementation that corresponds closely to what I wrote above. Find it on [GitHub](https://github.com/karpathy/minbpe).

## Takeaways

BPE is a greedy, frequency-driven compression scheme that turns out to be a fantastic way to tokenize text for neural networks. Every modern LLM uses it or a close relative (WordPiece, Unigram LM). The details matter: byte-level vs character-level, merge order, pre-tokenization regex, vocabulary size. All of them affect how many tokens your input takes, which feeds directly into cost and context.

The next time you hit a weird model behavior around counting letters, case-sensitivity, or non-English languages, check what the tokenizer did first. Half the time the model never saw the characters you thought you sent it.
