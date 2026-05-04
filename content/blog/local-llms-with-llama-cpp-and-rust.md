+++
title = "Local LLMs with llama.cpp and Rust"
date = 2025-01-26
description = "Running 7B-70B language models on your own machine from Rust: GGUF format, quantization internals, hardware sizing, and when local actually beats a cloud API."

[taxonomies]
tags = ["rust", "llm", "llama-cpp", "ai"]
+++

The most underrated shift in practical AI over the last two years is that useful LLMs now fit on a single workstation. A 7B Mistral at Q4 runs at readable speeds on a five-year-old laptop. A 70B Llama at Q4 fits on a Mac Studio with 64 GB of unified memory. The implication is not that you should throw away your API keys. It is that "local inference" is no longer a lab curiosity, and from Rust the tooling is genuinely pleasant.

If you already read [Multimodal AI: processing images, audio and text together](/blog/multimodal-ai-images-audio-text-together), this is the other half of the story. That post was about calling a hosted model from Rust. This one is about running one in-process with zero network hops.

<!-- more -->

## The llama.cpp ecosystem in one paragraph

[`llama.cpp`](https://github.com/ggml-org/llama.cpp) is Georgi Gerganov's C/C++ inference engine for transformer language models. It started as a weekend project to run the original LLaMA on a MacBook and turned into the de facto runtime for local LLMs. It has no Python dependencies, compiles to a single binary, supports Metal on Apple Silicon, CUDA and ROCm for discrete GPUs, Vulkan for everything else, and plain SIMD (AVX2/AVX512/NEON) on CPUs. It loads models in its own file format called GGUF and supports most open-weight architectures published since 2023: Llama 1/2/3/3.1/3.2, Mistral, Mixtral, Qwen, Phi, Gemma, DeepSeek, SmolLM, and a growing list.

From Rust you do not talk to the CLI. You use [`llama-cpp-2`](https://github.com/utilityai/llama-cpp-rs) (part of the `llama-cpp-rs` workspace maintained by utilityai). It is a thin, safe-ish wrapper over `llama.cpp`'s C API. Features gate the backends: `cuda`, `metal`, `vulkan`, `hipblas`. Link the right one for your machine and you get the same Rust API regardless.

## GGUF: what the file actually is

GGUF replaced the older GGML format in August 2023 and has been stable since. A `.gguf` file is a single binary blob organized as:

1. **Magic** (`GGUF`, 4 bytes) + **version** (u32, currently 3).
2. **Tensor count** (u64) and **metadata KV count** (u64).
3. **Metadata key-value pairs.** Strings for keys, a tagged union for values (u8, i8, u16, i16, u32, i32, u64, f32, f64, bool, string, array). This is where you find `general.architecture = "llama"`, `llama.context_length = 131072`, `tokenizer.ggml.tokens = [...]`, the chat template, the BOS/EOS IDs.
4. **Tensor info records.** For each tensor: name, number of dimensions, shape, ggml dtype (one of ~30 quantization formats), byte offset into the data region.
5. **Alignment padding** to the value in `general.alignment` (default 32).
6. **Tensor data.** One contiguous region.

The format was designed to be `mmap`-friendly. `llama.cpp` does not allocate RAM for the weights. It memory-maps the file and lets the OS page in what it needs. This is why loading a 40 GB model is instantaneous and why you can cold-start inference without blowing your RAM budget. It also means the kernel can share the mapped pages across multiple processes: two `llama.cpp` servers on the same box pointing at the same file share physical memory.

You can inspect any GGUF file with `gguf-dump.py` from the llama.cpp tree or with the Rust [`gguf`](https://crates.io/crates/gguf) crate. It is worth doing once just to see that there are no surprises hiding inside.

## Quantization, not hand-wavingly

The reason a 7B model fits in 4 GB of RAM is that the weights are not stored as f32 or even f16. They are stored in blocks of quantized integers with a shared scale factor per block.

Take the commonly recommended `Q4_K_M` format. The `_K` family (introduced in 2023) uses *superblocks* of 256 weights, subdivided into 8 blocks of 32. Each superblock stores:

- One f16 scale for the whole superblock.
- One f16 min value for the whole superblock.
- 6-bit scales and 6-bit mins per 32-element sub-block (so 8 pairs).
- The actual 4-bit quantized weights, 32 per sub-block.

Work it out: 256 * 4 bits for the weights = 128 bytes. Plus 12 bytes of scales. Total 140 bytes for 256 weights. That is 4.375 bits per weight. An f32 model of the same size would use 1024 bytes for the same 256 weights. You saved 7.3x.

The `_M` suffix means "medium mix": attention and feed-forward tensors that are more sensitive get upgraded to `Q6_K` automatically, while the rest stay at 4 bits. This is why Q4_K_M is the default recommendation for almost every model on Hugging Face: it is within ~0.5-1% perplexity of the f16 reference on most benchmarks but fits in a third of the memory.

The broader table people actually pick from:

| Format | Bits/weight | 7B model size | Quality |
|--------|-------------|---------------|---------|
| F16 | 16 | ~14 GB | Reference |
| Q8_0 | 8.5 | ~7.2 GB | Indistinguishable from F16 |
| Q6_K | 6.6 | ~5.5 GB | Nearly indistinguishable |
| Q5_K_M | 5.7 | ~4.8 GB | Slight degradation on hard tasks |
| Q4_K_M | 4.9 | ~4.1 GB | Sweet spot for most uses |
| Q3_K_M | 3.9 | ~3.3 GB | Noticeable degradation |
| Q2_K | 2.7 | ~2.4 GB | Only useful when you are desperate |

Below Q3 the model starts dropping facts and producing structural errors. Between Q4_K_M and Q8_0 most users cannot tell the difference on everyday chat, summarization, or coding tasks. If you have the memory, Q5_K_M is an easy upgrade.

## Hardware: what you actually need

The rough rule: **VRAM or RAM in GB should be about the model size in GB plus 2 GB for KV cache and overhead.**

- **7B at Q4_K_M** runs on a 2018 laptop with 8 GB RAM. Expect 10-25 tokens/sec on CPU, 60-120 tokens/sec on Apple Silicon, 100+ tokens/sec on a mid-range GPU.
- **13B at Q4_K_M** needs about 10 GB. Apple Silicon with 16 GB works, but you are tight.
- **34B at Q4_K_M** needs ~22 GB. Think RTX 3090/4090 (24 GB VRAM), or Mac Studio.
- **70B at Q4_K_M** needs ~43 GB. This is where Apple Silicon with unified memory shines: an M2/M3 Ultra with 128 GB is dramatically cheaper than stacking two 24 GB GPUs, and you do not pay a PCIe transfer tax.

A detail that matters more than most people realize: the KV cache scales with context length and model size. A 7B model at 8K context is about 1 GB of KV. At 128K context, the same model needs 16 GB for KV alone. If you want long contexts, budget for the KV cache, not just the weights. `llama.cpp` supports KV cache quantization (`-ctk q8_0 -ctv q8_0`) which halves it at a tiny quality cost.

On the instruction-set side: `llama.cpp` will use AVX2, AVX-512, AMX (on recent Xeons), NEON+FP16 on ARM, and the GPU backends you compile in. Building with `cargo` and `-C target-cpu=native` on top of the right feature flags buys you real throughput. On a Ryzen 7950X, plain SSE2 vs AVX-512 BF16 can be a 3-4x gap on prompt-processing speed.

## Calling llama-cpp-2 from Rust

The minimum viable chatbot. Add to `Cargo.toml`:

```toml
[dependencies]
llama-cpp-2 = { version = "0.1", features = ["metal"] }  # or "cuda", "vulkan"
anyhow = "1"
```

And the code:

```rust
use llama_cpp_2::{
    context::params::LlamaContextParams,
    llama_backend::LlamaBackend,
    llama_batch::LlamaBatch,
    model::{params::LlamaModelParams, AddBos, LlamaModel, Special},
    sampling::LlamaSampler,
    token::LlamaToken,
};
use std::num::NonZeroU32;
use std::path::Path;

fn main() -> anyhow::Result<()> {
    let backend = LlamaBackend::init()?;

    let model_params = LlamaModelParams::default()
        .with_n_gpu_layers(999); // offload all layers when a GPU backend is built

    let model = LlamaModel::load_from_file(
        &backend,
        Path::new("models/mistral-7b-instruct-v0.3.Q4_K_M.gguf"),
        &model_params,
    )?;

    let ctx_params = LlamaContextParams::default()
        .with_n_ctx(Some(NonZeroU32::new(4096).unwrap()))
        .with_n_batch(512);

    let mut ctx = model.new_context(&backend, ctx_params)?;

    let prompt = "[INST] Explain, in two sentences, \
        what a memory-mapped file is. [/INST]";
    let tokens = model.str_to_token(prompt, AddBos::Always)?;

    let mut batch = LlamaBatch::new(512, 1);
    let last = tokens.len() as i32 - 1;
    for (i, token) in tokens.iter().enumerate() {
        batch.add(*token, i as i32, &[0], i as i32 == last)?;
    }
    ctx.decode(&mut batch)?;

    let mut sampler = LlamaSampler::chain_simple([
        LlamaSampler::top_k(40),
        LlamaSampler::top_p(0.9, 1),
        LlamaSampler::temp(0.7),
        LlamaSampler::dist(1234),
    ]);

    let mut cursor = tokens.len() as i32;
    for _ in 0..256 {
        let token: LlamaToken = sampler.sample(&ctx, -1);
        sampler.accept(token);

        if model.is_eog_token(token) {
            break;
        }

        let piece = model.token_to_str(token, Special::Tokenize)?;
        print!("{piece}");

        batch.clear();
        batch.add(token, cursor, &[0], true)?;
        ctx.decode(&mut batch)?;
        cursor += 1;
    }
    println!();
    Ok(())
}
```

A few things worth pointing out:

- `LlamaBackend::init()` is a process-global call. Do it once. The `ggml` library has some non-thread-local state.
- `n_gpu_layers(999)` means "offload everything." If the model does not fit, start lower and watch VRAM.
- The token loop is manual on purpose. `llama-cpp-2` does not hide the KV-cache cursor from you, and once you understand it you can do fancy things like speculative decoding, prompt caching across requests, or classifier-free guidance.

## Prompt templates are not optional

This is the single biggest source of "my local model gives garbage answers" bug reports. Every model family has a specific chat template, and feeding a prompt in the wrong format causes the model to ramble, repeat, or produce tokens it should not.

The three big families you will meet:

**Llama 3 / 3.1 / 3.2** (special tokens, no visible braces):

```
<|begin_of_text|><|start_header_id|>system<|end_header_id|>

You are a helpful assistant.<|eot_id|><|start_header_id|>user<|end_header_id|>

What is 2+2?<|eot_id|><|start_header_id|>assistant<|end_header_id|>

```

**Mistral / Mixtral Instruct**:

```
<s>[INST] What is 2+2? [/INST]
```

Multi-turn: `<s>[INST] Q1 [/INST] A1</s>[INST] Q2 [/INST]`. Note that Mistral has no system role at the wire level; system instructions go in the first `[INST]`.

**ChatML (Qwen, many fine-tunes)**:

```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
What is 2+2?<|im_end|>
<|im_start|>assistant
```

The GGUF file stores the official template under the `tokenizer.chat_template` metadata key as a Jinja2 string. `llama-cpp-2` exposes `LlamaModel::apply_chat_template` which will render it correctly if you give it a list of `{role, content}` pairs. Use it. Hand-writing template strings for every model is how you spend an afternoon debugging a missing `<|eot_id|>`.

## Local vs API: the honest comparison

**Latency.** On a Mac M3 Max with a 7B Q4_K_M, first token arrives in 20-40 ms after the prompt is tokenized, and throughput is 80-100 tokens/sec. A GPT-4o-class API call with a short prompt is typically 300-800 ms to first token including network, and 60-120 tokens/sec after. Local is faster for first-token latency; APIs are competitive on sustained throughput *for equivalent quality tiers*, but no 7B model is equivalent to GPT-4o.

**Cost.** At scale the math flips hard. Say you do 10M requests/month, each averaging 500 input + 200 output tokens. At frontier API pricing (~$3/M input, ~$15/M output as of mid-2026), that is $15K + $30K = $45K/month. The same workload on a 7B or 13B local model costs you one or two machines and electricity. A Mac Studio Ultra amortized over 36 months plus power is under $500/month. The question is whether the local model is good enough, not whether it is cheaper.

**Privacy.** Local is the only option for actually-sensitive data: legal, medical, source code under NDA, anything covered by data residency rules you cannot negotiate. Even with enterprise API tiers and zero-retention agreements, the traffic still leaves your network. With local, it does not.

**Quality.** This is where honesty is required. A 7B Q4_K_M Mistral is great for classification, summarization of short text, structured extraction, routing decisions, tool-calling glue, chat that follows a tight system prompt. It is not a frontier model. It will hallucinate more, reason less reliably, and fall apart on long multi-step problems. A 70B Q4_K_M Llama 3.1 is in the same neighborhood as GPT-4o-mini on most evaluations but noticeably behind Claude Sonnet or GPT-4o on anything that needs careful reasoning.

## When local makes sense

Concrete situations where I keep seeing local win:

- **High volume of cheap-per-call tasks.** Log classification, intent routing, PII redaction, document tagging. You do not need GPT-4o to decide whether a support ticket is about billing or shipping.
- **Offline or edge.** A CLI tool that ships with a bundled model, a desktop app with embedded AI, a factory-floor device with no internet. APIs are simply not an option.
- **Hard privacy constraints.** Code review for a closed-source monorepo, analysis of internal HR data, anything the legal team flagged.
- **Latency-sensitive loops.** An IDE autocomplete, a real-time UI assistant, a gaming NPC. Network round trips add up. A local 3B model at 150 tokens/sec feels instant in ways a cloud call never will.
- **Cost floor exists.** You run enough inference that the API bill is a line item. Break-even for a 70B local setup vs Sonnet-class pricing is surprisingly low, often under 5M requests/month.

And situations where it does not:

- **Low volume, high variance work.** If you answer a few thousand complex questions per day and each matters, pay the API.
- **You need the smartest model available.** Local cannot match frontier models yet.
- **You do not want to think about quantization, VRAM, driver updates, or GPU thermal throttling.** This is a real engineering commitment.

## Takeaway

Local inference in Rust used to require Python wrappers and shell scripts. It no longer does. `llama-cpp-2` gives you a clean Rust API, GGUF gives you a single self-describing file per model, and quantization gives you the freedom to pick the quality/memory tradeoff that fits your hardware. The tech is mature enough that "do we run this in-process" is a legitimate architectural question for most AI-flavored features, not a research project.

The question to keep asking is not "can I run this locally" but "does local buy me something I actually need." When it does (cost at scale, privacy, latency, offline), the Rust story is now good enough that the answer is almost always yes.

## References

- [`llama.cpp` on GitHub](https://github.com/ggml-org/llama.cpp)
- [`llama-cpp-rs` Rust bindings](https://github.com/utilityai/llama-cpp-rs)
- [GGUF format spec](https://github.com/ggml-org/ggml/blob/master/docs/gguf.md)
- [k-quants PR (quantization internals)](https://github.com/ggml-org/llama.cpp/pull/1684)
- [Hugging Face GGUF models hub](https://huggingface.co/models?library=gguf)
- [Llama 3 prompt template reference](https://llama.meta.com/docs/model-cards-and-prompt-formats/meta-llama-3/)
