+++
title = "Running ML models locally in Rust with candle"
date = 2025-12-23
description = "Hugging Face's candle gives you a PyTorch-shaped tensor library in pure Rust with CUDA and Metal kernels. When it wins over ort and llama.cpp, and a BERT classifier you can ship as a single binary."
 
[taxonomies]
tags = ["rust", "candle", "ml", "inference"]
+++

There are now three serious ways to run a neural network from Rust, and I find people routinely reach for the wrong one. If you want a generative LLM, [llama.cpp plus `llama-cpp-2`](/blog/local-llms-with-llama-cpp-and-rust) is the correct default. If you want to ship a PyTorch-trained classifier or embedder without bringing Python along, [ONNX Runtime through `ort`](/blog/onnx-runtime-in-rust-deploying-ml-models-without-python) is hard to beat. The third option, [`candle`](https://github.com/huggingface/candle), is the one people skip past, and it is the right answer more often than you think. This post is about what candle actually is, how it differs from the other two, and what a real text classifier looks like when you build it on top.

<!-- more -->

## What candle is, precisely

candle is a tensor library written in Rust by Hugging Face. It started in mid-2023 and has been a serious project since around version 0.3 in late 2023. The core design decision is that the tensor abstraction, the autograd engine, and the operator set are all native Rust. There is no Python in the loop, not even at build time, and no C++ inference runtime to link against. What you link against is a set of Rust crates plus, on GPU, a small amount of CUDA C or Metal Shading Language that ships as source inside the crate and is compiled on first build.

The workspace breaks into a few crates you will touch directly:

- `candle-core` gives you `Tensor`, `Device`, `DType`, the op set, autograd tape, and the memory backends (CPU via `ndarray`-like storage, CUDA via `cudarc`, Metal via `objc2` and a Metal kernel library, and WebAssembly via a SIMD path).
- `candle-nn` is the equivalent of `torch.nn`: `Linear`, `LayerNorm`, `Embedding`, `RmsNorm`, rotary embeddings, activations, plus the `VarBuilder` abstraction that maps safetensors weights to module parameters.
- `candle-transformers` is a curated zoo of model implementations written against `candle-nn`: BERT, DistilBERT, T5, Llama 1/2/3, Mistral, Mixtral, Falcon, Gemma, Phi, Qwen, Whisper, CLIP, Stable Diffusion, MusicGen. Each model is a few hundred lines of straightforward Rust that mirrors the reference implementation.
- `candle-examples` is a binary crate with runnable examples for most of the above, and is the single best teaching resource in the repo.

Because all of this is Rust, cargo resolves it and cross-compiles it like any other crate. A CPU-only build has no system dependencies at all. A CUDA build needs `nvcc` and the CUDA toolkit at build time, but the output still links the CUDA runtime dynamically the same way any CUDA program does.

## Where candle fits compared to ort and llama.cpp

The quick mental model:

- **`ort`** wraps Microsoft's C++ ONNX Runtime. You give it a graph (a `.onnx` file) and it executes it. You do not author the model; you export it from PyTorch. Operator coverage is huge, execution providers are everywhere, and performance on common architectures is state of the art.
- **`llama-cpp-2`** wraps a C++ inference engine hand-tuned for one architectural family: decoder-only transformers with rotary embeddings and grouped-query attention. Quantization is first-class. You load a GGUF file and generate tokens.
- **`candle`** is neither a graph executor nor a specialized LLM runtime. It is a tensor library. Each model is explicit Rust code you can read, step through, and modify. Weights live in safetensors. Execution is line-by-line tensor ops dispatched to CPU, CUDA, or Metal.

The tradeoffs fall out of that. candle wins when:

1. You want to ship a real Rust binary with no C++ dynamic libraries and no opset-version matchmaking. Pure `cargo build`, pure cross-compile.
2. The model architecture is modern and candle-transformers already has it. You get a faithful implementation plus downloadable weights in safetensors.
3. You want to modify the model: add a LoRA, swap in a new attention kernel, fuse in a custom head, do partial inference, or train a small piece on top. All of that is awkward in ORT and essentially impossible in `llama.cpp`.
4. You care about portability more than peak throughput. A CPU candle build runs unchanged on x86_64, aarch64, and WebAssembly; a Metal build runs on every recent Mac without touching `libonnxruntime`.

candle loses when the model has an odd operator set that nobody has ported yet, when you need bleeding-edge inference throughput on NVIDIA hardware (TensorRT through ORT beats candle's CUDA kernels on big transformer serving by 1.5-2x), or when you need sub-3-bit quantization at scale (that ecosystem lives in llama.cpp and ExLlama).

## Safetensors: why candle does not touch pickle

candle reads weights from the [safetensors](https://github.com/huggingface/safetensors) format, never from PyTorch `.pt` or `.bin` files. This matters for more than aesthetics.

A safetensors file is:

1. A `u64` little-endian header length.
2. A UTF-8 JSON header describing every tensor: dtype, shape, and byte offsets into the data region.
3. A contiguous binary blob of raw tensor data.

The JSON is bounded and parsable without running any code. Compare that to pickle, which is a Turing-complete serialization format that literally executes arbitrary Python when loaded. A malicious PyTorch checkpoint can run `rm -rf` on your laptop; a malicious safetensors file cannot. The format also supports `mmap`: candle's `VarBuilder::from_mmaped_safetensors` memory-maps the file and hands out views, so a 40 GB Llama 3 70B set of shards costs roughly nothing to open and lets the OS page weights in on demand.

If you are converting PyTorch checkpoints yourself, the `safetensors` Python package will do it in two lines. For Hugging Face models, almost every modern repo ships safetensors alongside the legacy bin files. `hf-hub` in Rust will download them straight into a local cache.

## The tensor API, briefly

candle's `Tensor` looks familiar to anyone coming from PyTorch. A few operations to anchor the shape:

```rust
use candle_core::{Device, Tensor, DType};

let device = Device::cuda_if_available(0)?;          // falls back to CPU
let a = Tensor::randn(0f32, 1f32, (3, 4), &device)?; // normal N(0,1)
let b = Tensor::ones((4, 2), DType::F32, &device)?;
let c = a.matmul(&b)?;                                // (3, 2)
let d = c.relu()?.sum_keepdim(1)?;                    // (3, 1)
```

Every op returns a `Result<Tensor, candle_core::Error>` because dtype and shape mismatches are caught at runtime but early. Ops are lazy on GPU in the sense that they return as soon as they enqueue a kernel; synchronization happens when you read data back with `.to_vec1()` or `.to_vec2()`. Autograd works by threading a `GradStore` through `Tensor::backward()` and is good enough for fine-tuning small heads and LoRAs but is not the reason people reach for candle.

One thing to know early: candle's `DType` enum includes `F32`, `F16`, `BF16`, `U8`, `U32`, and `I64`. There is no native int8 quantization in the tensor type. GGML-style quantized tensors live in `candle-core::quantized` as a parallel hierarchy with their own `QTensor` type and `q4_0`, `q4_k`, `q5_k`, `q8_0`, and friends. That path is how candle runs quantized LLMs; it is deliberately separate from the main tensor type because the memory layout and dispatch rules are different.

## Devices and GPU kernels

`Device` is an enum: `Cpu`, `Cuda(CudaDevice)`, `Metal(MetalDevice)`. Picking one is one line. The interesting part is what happens under the hood.

On CUDA, candle ships a set of `.cu` source files bundled into the `candle-kernels` crate. They are compiled to PTX at build time via `bindgen_cuda` and loaded into the process at runtime. For the ops candle supports natively, that is the only code path. For matrix multiplications, candle calls cuBLAS directly through `cudarc`; it does not try to out-perform NVIDIA's tuned BLAS. Flash-attention is an optional feature gated behind `candle-flash-attn` which wraps a hand-written Rust+CUDA implementation of Tri Dao's algorithm.

On Metal, candle ships `.metal` shader source compiled on demand by the OS and dispatched through `objc2`. The Metal backend is the reason candle feels particularly good on Apple Silicon: there is no CUDA toolkit to install, and unified memory means tensors move between CPU and GPU without explicit copies. Throughput on an M3 Max for a BERT-base forward pass is within 10-20% of the same model running under PyTorch MPS.

WebAssembly is the surprise. `candle-wasm-examples` runs Whisper-tiny and BERT sentence embeddings in the browser using SIMD-128 intrinsics and `WebGPU` for compute. The full ML model, not a stub, executes inside the tab. If you have ever wanted to ship an AI feature to users who do not want to install anything, this is the only credible path in 2026.

## A text classifier with BERT, end to end

The classic "what does this look like to actually use" example. Goal: load a fine-tuned BERT sentiment classifier, run a string through it, print the label.

`Cargo.toml`:

```toml
[dependencies]
candle-core = "0.8"
candle-nn = "0.8"
candle-transformers = "0.8"
tokenizers = "0.20"
hf-hub = "0.3"
serde_json = "1"
anyhow = "1"
```

The whole program:

```rust
use anyhow::{Context, Result};
use candle_core::{DType, Device, Tensor};
use candle_nn::{ops::softmax, VarBuilder};
use candle_transformers::models::bert::{BertModel, Config};
use hf_hub::{api::sync::Api, Repo, RepoType};
use tokenizers::Tokenizer;

fn pick_device() -> Result<Device> {
    if let Ok(d) = Device::new_cuda(0) { return Ok(d); }
    if let Ok(d) = Device::new_metal(0) { return Ok(d); }
    Ok(Device::Cpu)
}

fn main() -> Result<()> {
    let device = pick_device()?;

    // Fetch weights, tokenizer, config from the Hub. Cached under ~/.cache/huggingface.
    let api = Api::new()?;
    let repo = api.repo(Repo::new(
        "sentence-transformers/all-MiniLM-L6-v2".into(),
        RepoType::Model,
    ));
    let config_path    = repo.get("config.json")?;
    let tokenizer_path = repo.get("tokenizer.json")?;
    let weights_path   = repo.get("model.safetensors")?;

    let config: Config = serde_json::from_slice(&std::fs::read(config_path)?)
        .context("parsing bert config")?;
    let tokenizer = Tokenizer::from_file(tokenizer_path)
        .map_err(|e| anyhow::anyhow!("tokenizer load: {e}"))?;

    // mmap the weights into a VarBuilder. Unsafe because the file could
    // change under us while mapped; for local cached files it is fine.
    let vb = unsafe {
        VarBuilder::from_mmaped_safetensors(&[weights_path], DType::F32, &device)?
    };
    let model = BertModel::load(vb, &config)?;

    let text = "candle makes this surprisingly pleasant";
    let encoding = tokenizer.encode(text, true)
        .map_err(|e| anyhow::anyhow!("encode: {e}"))?;

    let ids = Tensor::new(encoding.get_ids(), &device)?.unsqueeze(0)?;
    let type_ids = ids.zeros_like()?;
    let mask = Tensor::new(encoding.get_attention_mask(), &device)?.unsqueeze(0)?;

    // BertModel::forward -> (batch, seq, hidden)
    let hidden = model.forward(&ids, &type_ids, Some(&mask))?;

    // Mean-pool over the sequence dimension with attention mask weighting.
    let mask_f = mask.to_dtype(DType::F32)?.unsqueeze(2)?; // (1, seq, 1)
    let masked = hidden.broadcast_mul(&mask_f)?;
    let summed = masked.sum(1)?;                           // (1, hidden)
    let counts = mask_f.sum(1)?.clamp(1e-9, f32::INFINITY)?;
    let embedding = summed.broadcast_div(&counts)?;

    // For a real classifier you would matmul against a head. For a
    // sentence encoder like MiniLM you L2-normalize and you are done.
    let norm = embedding.sqr()?.sum_keepdim(1)?.sqrt()?;
    let embedding = embedding.broadcast_div(&norm)?;

    println!("embedding shape: {:?}", embedding.shape());
    println!("first 8 dims: {:?}", embedding.squeeze(0)?.narrow(0, 0, 8)?.to_vec1::<f32>()?);
    Ok(())
}
```

A few things worth highlighting:

- `hf-hub` handles the download and caching. Rerunning the binary hits the cache, so the second launch of a 90 MB MiniLM model is instant.
- `VarBuilder::from_mmaped_safetensors` is the fast path. There is also `from_buffered_safetensors` if you already have the bytes in RAM and `from_tensors` if you are loading from a `HashMap<String, Tensor>` (useful in tests).
- For a fine-tuned classification model (say a DistilBERT-SST2 variant), you would load the classification head by extending `BertModel` with a linear layer, or use `candle-transformers`'s dedicated classifier builder. The shape of the code does not change.
- `softmax(&logits, 1)` from `candle_nn::ops` gives you probabilities; `argmax` on CPU is just a `.to_vec1()` and a fold. That is the entire "inference API": tensor in, tensor out.

Build time is the thing you notice. A cold build of this binary on CPU is under a minute on a modern laptop. With CUDA, the first build compiles the bundled CUDA kernels and takes 2-4 minutes; subsequent builds are cached. The resulting release binary is around 8 MB on CPU and around 25 MB with CUDA linkage. Compare that to the 500 MB of Python, PyTorch, and transformers that the equivalent script would need.

## Performance, without the hype

The PyTorch comparison people ask for. Hugging Face's own microbenchmark, which you can reproduce from `candle-examples`, runs BERT-base at sequence length 128 on an M2 Pro and reports roughly 8 ms per forward pass on Metal versus roughly 10 ms for PyTorch MPS on the same hardware. On an RTX 4090, candle's CUDA backend does BERT-base at batch 1 in about 1.8 ms versus 1.5 ms for PyTorch. These are in the same neighborhood: the bottleneck is cuBLAS in both cases.

The more interesting numbers are process-level:

- **Cold start.** A candle binary launches in around 40 ms to "ready to run forward pass" on Apple Silicon. The equivalent Python process takes 1.5-3 seconds because PyTorch's import path touches thousands of files.
- **Memory.** BERT-base under candle on CPU sits around 180 MB RSS. Under PyTorch it is 1.1 GB.
- **Image size.** A distroless container with a candle binary plus a safetensors file fits under 250 MB. The PyTorch equivalent is closer to 2 GB and needs either CUDA base images or a careful CPU-only wheel.

What you give up: a handful of rarely-used operators, some of the more exotic fused kernels, and PyTorch's enormous plugin ecosystem. You also give up training at scale. candle can train, and people do, but nobody is training a 70B model in it. If training is the job, stay in PyTorch; use candle for serving the result.

## The models you will actually reach for

From the current `candle-transformers` tree:

- **BERT, DistilBERT, RoBERTa, DeBERTa** for classification and embeddings.
- **T5 and FLAN-T5** for seq2seq tasks like summarization.
- **Llama 2 / 3 / 3.1, Mistral, Mixtral, Qwen 2, Phi 3, Gemma 2** for generation. The quantized variants are competitive with llama.cpp for small and medium models.
- **Whisper** (tiny through large-v3) for ASR. `candle-wasm-examples/whisper` runs this in a browser tab.
- **CLIP and SigLIP** for text-image similarity. Pair with the embedding pattern in the [cosine similarity post](/blog/cosine-similarity-the-math-behind-semantic-search) and you have a photo search that ships in one binary.
- **Stable Diffusion 1.5, 2.1, XL, and 3** for image generation. The SD-XL implementation is about 3x slower than a tuned ComfyUI stack on the same hardware but runs with zero Python and fits in a 200 MB binary.
- **MusicGen** for audio generation, which is genuinely novel as a "no Python" build target.

Each of these is a few hundred lines of Rust you can read. If the behavior is wrong, you set a breakpoint and step through it. That is not a small thing.

## When candle is and is not the answer

Use candle when you want a pure-Rust build, an architecturally familiar tensor API, and a model you can read and modify. Use it when you target platforms the ORT binary is awkward on (WebAssembly, embedded Linux without glibc, some FreeBSD setups). Use it when your deployment story is "ship a binary, a safetensors file, and a tokenizer.json, done."

Do not use candle when the model only exists as a PyTorch checkpoint with custom ops and you do not want to port them: ORT through `ort` will handle the exported graph and you will spend days fewer on it. Do not use candle for long-context generative LLM serving at high QPS: llama.cpp and vLLM are further along on that axis. And do not use candle when you need training at scale: nobody has built that infrastructure yet, and they probably will not, because that is not the niche.

## Closing

Running ML in Rust stopped being a research exercise around 2024 and became a normal engineering choice. candle is the option people keep overlooking because "tensor library" sounds less finished than "graph runtime." In practice that framing is inverted: the fact that models in candle are just Rust code is exactly what makes the deployment story clean. You type `cargo build --release`, you get a binary, and inside that binary is a neural network. The fact that it is boring is the point.

## References

- [`candle` on GitHub](https://github.com/huggingface/candle)
- [`candle-transformers` model zoo](https://github.com/huggingface/candle/tree/main/candle-transformers/src/models)
- [`candle-examples`](https://github.com/huggingface/candle/tree/main/candle-examples)
- [safetensors format spec](https://github.com/huggingface/safetensors)
- [`hf-hub` for downloading weights](https://github.com/huggingface/hf-hub)
- [`tokenizers` Rust crate](https://github.com/huggingface/tokenizers)
- [WebAssembly demos (Whisper, BERT, etc.)](https://github.com/huggingface/candle/tree/main/candle-wasm-examples)
