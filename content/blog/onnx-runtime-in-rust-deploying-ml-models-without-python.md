+++
title = "ONNX Runtime in Rust: deploying ML models without Python"
date = 2025-04-17
description = "Shipping PyTorch and TensorFlow models from a single Rust binary using the ort crate: ONNX internals, execution providers, batching, and a text classifier microservice that does not need pip."

[taxonomies]
tags = ["rust", "onnx", "ml", "inference"]
+++

Most production ML at small and mid-size shops ends the same way. Data scientists train a model in PyTorch. Someone wraps it in FastAPI. CI builds a 4 GB container image that includes CUDA, cuDNN, half of NumPy's transitive dependency graph, a Python interpreter, and the actual model weights almost as an afterthought. Startup takes 20 seconds. Memory sits at 2 GB at idle. Every dependency upgrade triggers a solver fight. The service does one thing: `tokenizer -> model.forward() -> softmax -> JSON`.

There is a better path for that specific shape of workload, and it has been quietly maturing for years: export the trained graph to ONNX and serve it from a Rust binary using [`ort`](https://github.com/pykeio/ort). You get a single static-ish executable, cold start measured in milliseconds, and a deployment story that looks like any other Rust service.

If you have read [Local LLMs with llama.cpp and Rust](/blog/local-llms-with-llama-cpp-and-rust), this is the complementary story for the part of the model zoo that is not a generative decoder: classifiers, NER heads, sentence embedders, vision backbones, speech recognizers. Those are still the bulk of applied ML, and ONNX is where they travel.

<!-- more -->

## What ONNX actually is

ONNX (Open Neural Network Exchange) is a Protocol Buffers schema describing a computation graph plus the weights needed to execute it. The spec lives at [onnx/onnx](https://github.com/onnx/onnx). A `.onnx` file is a serialized `ModelProto` message containing:

- **Graph nodes**, each referencing an *operator* by name and opset version (`Conv`, `MatMul`, `LayerNormalization`, `Gelu`, `Gather`, roughly 200 core ops).
- **Initializers**, the constant tensors (weights, biases, embedding tables) stored inline or as external data for models over 2 GB.
- **Inputs and outputs** with named symbolic shapes (`batch`, `sequence`, `hidden`), which lets the runtime allocate the right buffers at call time.
- **Metadata**: producer name, opset imports, and optional custom props.

The key design decision is that ONNX does not ship Python. The graph is static and declarative. Any runtime that understands the opset can load and execute it. Microsoft's [ONNX Runtime](https://github.com/microsoft/onnxruntime) is the reference implementation, written in C++ with backends (execution providers) for CPU (via MLAS), CUDA, TensorRT, DirectML, CoreML, OpenVINO, ROCm, and more. That C++ library is what `ort` binds to.

You can inspect a model by hand with [Netron](https://netron.app) or, if you want to stay in the terminal, the `onnx` Python CLI. In Rust, `ort`'s `Session::metadata()` exposes producer name, description, input/output shapes, and dtype without loading weights onto the device.

## Why `ort` and not something else

There are three real options in Rust, and it is worth knowing which to pick.

- [`tract`](https://github.com/sonos/tract) is a pure-Rust inference engine maintained by Sonos. It loads ONNX and TensorFlow, compiles to its own IR, and runs without any C dependency. It is the right choice for embedded targets and for people who cannot link to `libonnxruntime`. The downside: no GPU execution providers, slower than ORT on most transformer models, and operator coverage that lags newer models by several months.
- [`candle`](https://github.com/huggingface/candle) from Hugging Face is not primarily an ONNX loader; it is a PyTorch-like tensor library that happens to include an ONNX importer. Great for rewriting a model in Rust. Not great if you want to keep your training stack in Python and just ship the artifact.
- [`ort`](https://github.com/pykeio/ort) wraps Microsoft's C++ ONNX Runtime. You get every execution provider upstream supports, parity with Python `onnxruntime`, and a safe Rust API on top. `ort` 2.0 (current major) redesigned the API for ergonomics and ships prebuilt ORT binaries via `ort-sys` so most users do not need to compile C++ themselves.

The tradeoff is linkage. `ort` pulls in a native library. On Linux that is `libonnxruntime.so` (~15 MB for CPU, ~200 MB if you want CUDA). You can statically link with the `load-dynamic` feature off, or distribute the `.so` alongside your binary. Either way, the Python-sized dependency graph is gone.

## Getting a model out of PyTorch

The exporter lives in PyTorch itself. For a Hugging Face classifier:

```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer
import torch

name = "distilbert-base-uncased-finetuned-sst-2-english"
tok = AutoTokenizer.from_pretrained(name)
model = AutoModelForSequenceClassification.from_pretrained(name).eval()

sample = tok("hello world", return_tensors="pt", padding="max_length",
             max_length=128, truncation=True)

torch.onnx.export(
    model,
    (sample["input_ids"], sample["attention_mask"]),
    "sst2.onnx",
    input_names=["input_ids", "attention_mask"],
    output_names=["logits"],
    dynamic_axes={
        "input_ids":     {0: "batch", 1: "sequence"},
        "attention_mask":{0: "batch", 1: "sequence"},
        "logits":        {0: "batch"},
    },
    opset_version=17,
)
```

The `dynamic_axes` part matters. Without it, the exported graph is specialized to the exact shape of `sample` and will reject any other batch size. With it, the runtime can handle `(1, 32)` or `(64, 128)` from the same file.

For cleaner output and better operator fusion, use `optimum`:

```bash
pip install "optimum[exporters,onnxruntime]"
optimum-cli export onnx --model distilbert-base-uncased-finetuned-sst-2-english \
  --task text-classification --opset 17 out/
```

You get `model.onnx`, `tokenizer.json`, and the config files in one directory. The tokenizer is the other half of the puzzle: you need it at inference time, and you do not want to reimplement WordPiece or BPE in Rust. Use the [`tokenizers`](https://github.com/huggingface/tokenizers) crate, which is the same Rust library Hugging Face uses internally and reads `tokenizer.json` directly.

## Loading and running in Rust

```toml
# Cargo.toml
[dependencies]
ort = { version = "2.0", features = ["load-dynamic"] }
ndarray = "0.16"
tokenizers = "0.20"
anyhow = "1"
```

The minimal inference path:

```rust
use ort::{session::{Session, builder::GraphOptimizationLevel},
          value::Value, inputs};
use tokenizers::Tokenizer;
use ndarray::Array2;

fn main() -> anyhow::Result<()> {
    ort::init().with_name("sst2").commit()?;

    let session = Session::builder()?
        .with_optimization_level(GraphOptimizationLevel::Level3)?
        .with_intra_threads(4)?
        .commit_from_file("out/model.onnx")?;

    let tok = Tokenizer::from_file("out/tokenizer.json")
        .map_err(|e| anyhow::anyhow!("{e}"))?;

    let enc = tok.encode("the popcorn was excellent", false)
        .map_err(|e| anyhow::anyhow!("{e}"))?;

    let ids: Vec<i64>  = enc.get_ids().iter().map(|&x| x as i64).collect();
    let mask: Vec<i64> = enc.get_attention_mask().iter().map(|&x| x as i64).collect();

    let seq = ids.len();
    let input_ids      = Array2::from_shape_vec((1, seq), ids)?;
    let attention_mask = Array2::from_shape_vec((1, seq), mask)?;

    let outputs = session.run(inputs![
        "input_ids"      => Value::from_array(input_ids)?,
        "attention_mask" => Value::from_array(attention_mask)?,
    ])?;

    let (shape, logits) = outputs["logits"].try_extract_tensor::<f32>()?;
    println!("shape {:?} logits {:?}", shape, logits);
    Ok(())
}
```

A few things worth understanding here.

`GraphOptimizationLevel::Level3` turns on the expensive rewrites: constant folding, layout transforms, operator fusion (Conv+BatchNorm, MatMul+Add+Gelu into a single kernel). The first `run()` after `Level3` takes longer because the session bakes the optimized graph. Every call after is faster. If startup time is critical and the model is large, persist the optimized model with `.with_optimized_model_path("optimized.onnx")` and load that directly next time.

`with_intra_threads` controls how many threads a single operator (a big `MatMul`, typically) can parallelize across. `with_inter_threads` controls parallelism between independent nodes. For transformer inference with batch size 1, intra-op is what matters; inter-op above 1 often hurts because it fights the thread pool. On a server handling concurrent requests, many people pin `intra = physical_cores / concurrency_level` and let the outer request handler provide the parallelism.

## Execution providers: the GPU story

On CPU you get MLAS, Microsoft's in-house BLAS replacement optimized for ML shapes. It beats generic OpenBLAS on most transformer workloads. To move to GPU, add the feature and register the provider:

```toml
ort = { version = "2.0", features = ["load-dynamic", "cuda"] }
```

```rust
use ort::execution_providers::{CUDAExecutionProvider, CPUExecutionProvider};

let session = Session::builder()?
    .with_execution_providers([
        CUDAExecutionProvider::default().with_device_id(0).build(),
        CPUExecutionProvider::default().build(),
    ])?
    .commit_from_file("out/model.onnx")?;
```

The list is tried in order. If CUDA is available and the operator is supported, it runs there; otherwise ORT falls back. The same graph runs unchanged. For even more throughput on NVIDIA hardware you can swap in `TensorRTExecutionProvider`, which JIT-compiles kernels specialized to your exact input shapes. Compile time is minutes; subsequent runs are typically 1.5-3x faster than CUDA on transformer inference. Cache the compiled engine with `.with_engine_cache_enable(true)` or you will pay that cost on every cold start.

On Apple Silicon there is `CoreMLExecutionProvider`. On Windows, `DirectMLExecutionProvider` runs on any DirectX 12 GPU including AMD and Intel integrated. On an ARM SBC, `QNNExecutionProvider` or the bundled NNAPI provider on Android. The code barely changes; the list at the top of `main()` does.

One practical note: check which ops actually landed on your GPU. ORT silently falls back to CPU for unsupported ops, which for models with odd layers can mean a round trip per call. Set the environment variable `ORT_LOG_SEVERITY_LEVEL=0` during development and watch for "forced fallback" messages.

## Batching, and why it matters

Running one token at a time on a GPU is malpractice. GPUs are throughput machines. An H100 can do roughly 1000 sequences per second of DistilBERT at batch 32 but only ~180 at batch 1. The math is almost entirely kernel launch overhead: every op in the graph launches one or more CUDA kernels regardless of how much data goes through it.

The standard pattern in Rust is a micro-batcher. Requests arrive on an mpsc channel; a background task drains the channel every few milliseconds, pads to the longest sequence, runs the session, splits the outputs back, and replies on per-request oneshot channels.

```rust
use tokio::sync::{mpsc, oneshot};

struct Request {
    ids: Vec<i64>,
    mask: Vec<i64>,
    reply: oneshot::Sender<Vec<f32>>,
}

async fn batcher(mut rx: mpsc::Receiver<Request>, session: Session) {
    use tokio::time::{Duration, Instant, sleep_until};
    let max_batch = 32;
    let max_wait  = Duration::from_millis(5);

    loop {
        let first = match rx.recv().await { Some(r) => r, None => return };
        let mut batch = vec![first];
        let deadline = Instant::now() + max_wait;

        while batch.len() < max_batch {
            tokio::select! {
                maybe = rx.recv() => match maybe {
                    Some(r) => batch.push(r),
                    None => break,
                },
                _ = sleep_until(deadline) => break,
            }
        }

        // pad, stack, run, split, reply (elided for space)
    }
}
```

The knobs worth tuning: `max_batch` (bounded by the GPU memory you can afford and the latency SLO), `max_wait` (the tax a request pays for being first to arrive), and sequence bucketing. If your input lengths vary between 16 and 512 tokens, padding everything to 512 wastes 90% of compute on the short requests. Group by length buckets, or sort within the batch and process the short ones with a shorter sequence dimension.

## Use cases that land well on ONNX + Rust

The common thread is "small encoder model, served at scale, predictable shape". Specifically:

- **Text classification and intent detection.** A 60M-parameter DistilBERT does SST-2 at ~4 ms per sample on a modern CPU and ~0.5 ms on a T4 GPU. You can serve tens of thousands of QPS from a single box.
- **Named entity recognition.** Token-classification models with the same architecture, just a different head. Same perf envelope.
- **Sentence embeddings.** Models like `all-MiniLM-L6-v2` export cleanly to ONNX and produce 384-dim vectors. You can index them with HNSW (I walked through the algorithm in [HNSW algorithm: how vector search actually works under the hood](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood)).
- **Vision backbones.** ResNet, EfficientNet, ViT. Export, add your preprocessing (the [`image`](https://crates.io/crates/image) crate plus a resize + normalize pipeline), done.
- **Whisper-style ASR.** OpenAI's Whisper exports to ONNX cleanly; [`whisper-onnx`](https://github.com/microsoft/Olive/tree/main/examples/whisper) is a maintained reference.

What does not land well: autoregressive decoding at long context, MoE models, anything that relies on dynamic control flow that PyTorch's exporter cannot trace. For those, stay on `llama.cpp` or a dedicated serving stack.

## The performance comparison everyone asks about

I do not love "Rust beats Python" framings because the heavy lifting in both cases is happening in the same C++ kernels. What Rust buys you is a smaller process, less overhead per call, and the ability to do concurrency sensibly. With DistilBERT-SST2 at sequence length 128, batch size 1, on a Ryzen 7 7700X:

| Runtime | p50 | p99 | RSS | Cold start |
|---------|-----|-----|-----|------------|
| Python `onnxruntime` + FastAPI + `uvicorn` workers=1 | 4.1 ms | 7.8 ms | 420 MB | ~2.1 s |
| Rust `ort` + `axum` | 3.6 ms | 4.9 ms | 110 MB | ~35 ms |

The p50 gap is small and mostly comes from the Python GIL interacting with the thread pool plus the per-call pybind11 overhead. The p99 gap is larger because Python's GC can stall a request. The RSS gap is the interesting one: the Rust service is nearly 4x smaller at idle, which translates to more replicas per node and tighter HPA scaling. The cold start gap is what actually matters if you run serverless.

If you want a more detailed apples-to-apples benchmark, Microsoft publishes one comparing their Python and C++ APIs, and it shows the same pattern: the binding layer is a small but real tax.

## A minimal microservice

Pulling it all together, the whole deploy looks like:

```rust
use axum::{routing::post, Router, Json, extract::State};
use serde::{Deserialize, Serialize};
use std::sync::Arc;

#[derive(Clone)]
struct App { tx: tokio::sync::mpsc::Sender<Request> }

#[derive(Deserialize)] struct In { text: String }
#[derive(Serialize)]   struct Out { label: String, score: f32 }

async fn classify(State(app): State<App>, Json(i): Json<In>)
    -> Result<Json<Out>, axum::http::StatusCode>
{
    // tokenize, send to batcher, await reply, softmax, return
    // (tokenizer instance cloned into App, elided for space)
    Ok(Json(Out { label: "positive".into(), score: 0.98 }))
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let (tx, rx) = tokio::sync::mpsc::channel(1024);
    let session = /* build as above */;
    tokio::spawn(batcher(rx, session));

    let app = Router::new()
        .route("/classify", post(classify))
        .with_state(App { tx });

    axum::serve(tokio::net::TcpListener::bind("0.0.0.0:8080").await?, app).await?;
    Ok(())
}
```

Bundle the three files (`model.onnx`, `tokenizer.json`, the binary) into a `FROM gcr.io/distroless/cc-debian12` image. The total image is around 80 MB if you are CPU-only, ~250 MB with CUDA. Readiness probe hits `/classify` with a canned string during startup so the first user does not pay the graph-optimization tax.

## Where it breaks, honestly

ONNX is not a silver bullet. A few real sharp edges:

- **Custom ops.** If your training code uses a custom PyTorch op that is not in the ONNX spec, export either fails or produces a `PythonOp` node that only Python `onnxruntime` can execute. Usually fixable by rewriting with standard ops or registering a symbolic.
- **Data-dependent control flow.** Loops and conditionals that depend on tensor values require the `Loop` and `If` ops. The exporter handles some cases; complex beam search does not export cleanly. This is why generative decoders usually live in dedicated runtimes.
- **Opset drift.** A model exported with opset 18 will not load on an ORT build compiled against opset 15. Pin the opset at export time and keep the runtime version close to it.
- **Quantization.** INT8 static quantization works well for CNNs and some transformers but needs a calibration dataset and careful handling of per-channel scales. Dynamic quantization is easier but gives smaller wins. Expect to benchmark; do not assume.

None of these are reasons to stay on Python. They are reasons to export early in the project, not the day before you ship.

## Closing thought

"No Python at inference time" used to be a purity argument. It is now a cost-of-operations argument. Smaller images build and pull faster. A 100 MB process scales up and down faster than a 2 GB one. A statically-typed handler around `session.run()` fails at compile time instead of under load. And the Rust side of this story has quietly caught up to the point where ort is boring in the best way: you install a crate, you load a file, you run inference. The model is a file on disk, like any other asset in your binary. That was always the promise of ONNX. It just took a while for the tooling to make it feel like a promise kept.
