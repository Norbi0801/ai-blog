+++
title = "Multimodal AI: processing images, audio and text together"
date = 2025-01-15
description = "How shared embedding spaces let one model reason across images, audio and text, and when you should reach for a specialized model instead."

[taxonomies]
tags = ["ai", "multimodal", "rust", "machine-learning"]
+++

A few years ago, if you wanted to search a photo library by description, you needed a whole pipeline: an object detector, a caption generator, a text index, some glue code to hope the captions matched the queries. Today you can do it with a single embedding model. The reason is boring and profound at the same time: images, audio, and text have all been coerced to live in the same vector space.

This post is about that trick, and what it looks like from Rust.

<!-- more -->

## What "multimodal" actually means

The word gets used in two quite different ways and it helps to separate them early.

1. **Multimodal input to a generative model.** You hand GPT-4o, Claude, or Gemini a prompt that contains both text and an image (or audio, or video), and it produces text back. The model was trained on interleaved modalities and learned a joint representation internally. You do not see the embeddings directly. You see token outputs.
2. **Multimodal embedding models.** You feed an image into an encoder, a piece of text into another encoder, and both produce vectors in the same high-dimensional space. Cosine similarity between the vectors approximates semantic similarity between the inputs. CLIP is the poster child.

These two worlds are connected but they solve different problems. Generative multimodal models are for reasoning and content creation. Embedding models are for search, clustering, deduplication, and retrieval. You usually want both in a real system, and you want to know when to reach for which.

## Shared embedding space, concretely

CLIP, published by OpenAI in 2021, trained a text encoder and an image encoder together with a contrastive objective. For every batch of N image/caption pairs, the model maximizes cosine similarity between matching pairs and minimizes it for the NxN-N non-matching pairs. Training ran on 400 million image-text pairs scraped from the public web.

The output of each encoder is a fixed-size vector. In the original CLIP ViT-B/32 model, both image and text embeddings live in R^512. You can compute:

```python
cos_sim = (img_vec @ text_vec) / (||img_vec|| * ||text_vec||)
```

and use it as a zero-shot classifier. No fine-tuning. No labels. Just "here is a photo" and "a photo of a cat" and a dot product.

The key property is that the text encoder and image encoder are separate stacks, but they project into the same space. That is what lets you:

- embed a million images at ingest time
- at query time, embed only the user's text query
- run a single ANN lookup (HNSW, FAISS, whatever) and get results

You are not running the image encoder at query time. That is the entire business case for CLIP-style retrieval: you move a slow computation (image encoding) from the hot path to the ingest path.

Newer models extend this idea. JinaCLIP, SigLIP, ImageBind, and a growing family of "omni" embedders include audio, depth maps, IMU data, and video frames into the same shared space. The architecture is the same. The training data and the contrastive loss change.

## Vision models as chat completions

GPT-4o, Claude 3.5 Sonnet, and Gemini all accept images inline in a chat request. The pattern is identical across providers at the wire level: a message's `content` is an array of typed parts, one of which is an image.

Anthropic's wire format looks like this:

```json
{
  "role": "user",
  "content": [
    { "type": "image", "source": {
        "type": "base64",
        "media_type": "image/png",
        "data": "<base64 bytes>"
    }},
    { "type": "text", "text": "What is unusual about this diagram?" }
  ]
}
```

OpenAI's looks like this:

```json
{
  "role": "user",
  "content": [
    { "type": "image_url",
      "image_url": { "url": "data:image/png;base64,<base64 bytes>" }},
    { "type": "text", "text": "What is unusual about this diagram?" }
  ]
}
```

Same idea, different JSON shape. The models will accept URLs too, but for anything private or time-sensitive you want base64. Claude supports up to 8000x8000 px per image (reduced to 2000x2000 if you send more than 20), so oversized images get downsampled by the API anyway, which costs you detail for nothing. Resize before you send.

## Calling a vision model from Rust

[`async-openai`](https://github.com/64bit/async-openai) (version 0.30+) mirrors the OpenAI API surface as typed Rust structs. Here is a minimal example that sends an image and a question:

```rust
use async_openai::{
    Client,
    types::{
        ChatCompletionRequestUserMessageArgs,
        ChatCompletionRequestMessageContentPartImageArgs,
        ChatCompletionRequestMessageContentPartTextArgs,
        CreateChatCompletionRequestArgs,
        ImageUrlArgs,
    },
};
use base64::{engine::general_purpose, Engine as _};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let client = Client::new();

    let bytes = std::fs::read("diagram.png")?;
    let b64 = general_purpose::STANDARD.encode(&bytes);
    let data_url = format!("data:image/png;base64,{}", b64);

    let image_part = ChatCompletionRequestMessageContentPartImageArgs::default()
        .image_url(ImageUrlArgs::default().url(data_url).build()?)
        .build()?;

    let text_part = ChatCompletionRequestMessageContentPartTextArgs::default()
        .text("What architecture does this diagram show?")
        .build()?;

    let message = ChatCompletionRequestUserMessageArgs::default()
        .content(vec![image_part.into(), text_part.into()])
        .build()?;

    let request = CreateChatCompletionRequestArgs::default()
        .model("gpt-4o")
        .messages(vec![message.into()])
        .max_tokens(512u32)
        .build()?;

    let response = client.chat().create(request).await?;
    println!("{}", response.choices[0].message.content.as_deref().unwrap_or(""));
    Ok(())
}
```

The Anthropic equivalent is essentially the same shape with `misanthropic` or a hand-rolled `reqwest` call against `api.anthropic.com/v1/messages`. The only real difference is that Anthropic wants the raw base64 string and media type as separate fields, not a data URL.

Two practical notes from shipping this kind of code:

- **Token costs are measured in pixels, not files.** A 1024x1024 image is roughly 765 input tokens for GPT-4o, 1600 for Claude. A 4K screenshot can burn 3-4K tokens before you say a single word.
- **Do not trust the model to OCR.** For clean text extraction, Tesseract, PaddleOCR, or a dedicated document model (Textract, Azure DI) is still more reliable and orders of magnitude cheaper. Use vision models for reasoning about images, not for converting them to strings.

## Audio: Whisper and its Rust bindings

Whisper is an encoder-decoder transformer trained on 680,000 hours of multilingual audio. It is open weights, and small enough to run on a laptop. The large-v3 model is 1.55B parameters and hits near-human WER on clean English.

You have two realistic options in Rust:

1. **Call the OpenAI transcription endpoint**, same API as GPT: send a WAV/MP3, get text back. Fine for prototypes, costs about $0.006 per minute, and you do not run the model yourself.
2. **Run Whisper locally via `whisper-rs`**, which binds to `whisper.cpp`. CPU inference is surprisingly usable for the `base` and `small` models; CUDA/Metal acceleration is opt-in via feature flags.

`whisper-rs` (0.16.0 as of March 2026) looks roughly like this:

```rust
use whisper_rs::{FullParams, SamplingStrategy, WhisperContext, WhisperContextParameters};

fn transcribe(wav_path: &str, model_path: &str) -> anyhow::Result<String> {
    let ctx = WhisperContext::new_with_params(
        model_path,
        WhisperContextParameters::default(),
    )?;
    let mut state = ctx.create_state()?;

    let mut params = FullParams::new(SamplingStrategy::Greedy { best_of: 1 });
    params.set_language(Some("en"));
    params.set_print_progress(false);

    // whisper expects 16kHz mono f32 samples in [-1.0, 1.0]
    let samples = load_wav_as_mono_f32_16k(wav_path)?;

    state.full(params, &samples)?;

    let n = state.full_n_segments()?;
    let mut out = String::new();
    for i in 0..n {
        out.push_str(&state.full_get_segment_text(i)?);
    }
    Ok(out)
}
```

The audio-preprocessing step (`load_wav_as_mono_f32_16k`) is the part people underestimate. Whisper does not resample for you. Feed it 44.1kHz and you will get garbage. `hound` plus `rubato` handles this in a few dozen lines.

Newer bindings like `whisper-cpp-plus` add VAD (voice activity detection) and streaming. The VAD piece matters more than it sounds: transcribing only the chunks that contain speech can be 2-3x faster on audio with long silences (podcasts, meetings, voicemail).

## Putting it together: image search by text

This is the canonical multimodal application and it is genuinely useful. The pipeline is:

1. At ingest: for every image, compute a CLIP embedding, store `(id, vector)` in a vector DB.
2. At query: compute a CLIP embedding of the user's text query.
3. Run approximate nearest neighbor search on the vector DB.
4. Return the top-K image IDs.

In Rust you can do all of this without a Python sidecar. `candle` (Hugging Face's Rust ML framework) has a working CLIP implementation. `qdrant-client` or `lancedb` will store vectors and run HNSW for you. A 100K-image index fits in a few hundred MB of RAM and returns top-10 results in low single-digit milliseconds on CPU.

The subtle part is that CLIP embeddings are not interchangeable with text-embedding-3-small or any other text-only embedding model. You cannot mix embedding spaces. If you are going to use CLIP, commit to it for both sides.

## Document understanding: why it is harder than it looks

"Document understanding" is the tempting umbrella term for everything from "summarize this PDF" to "extract line items from 200-page contracts." It is where multimodal models shine and also where they disappoint, depending on what the document actually is.

What the vision models are good at:

- Charts, diagrams, screenshots, UI mockups. Things where layout conveys meaning.
- Mixed layouts where text and figures need to be reasoned about together.
- Handwriting, especially messy handwriting where OCR struggles.

What they are not good at:

- Long documents. Every page is tokens. A 50-page PDF is a real cost problem.
- Forms with thousands of identical fields. A specialized document-AI service (Azure Document Intelligence, Google Document AI, Amazon Textract) with schema-aware extraction is more reliable and cheaper.
- Anything where you need structured output with guaranteed field positions (invoice line-item coordinates, for example).

A reasonable hybrid: use a cheap OCR pipeline to extract text, use a vision model only for pages that are visually complex, and use structured-output features (JSON mode or tool use) to force a predictable response shape.

## When to reach for multimodal vs specialized models

A rough rule I keep ending up with:

- **Use a specialized model when the task is narrow and you do it at scale.** Transcription at 1M minutes/month? Self-hosted Whisper. OCR on known form layouts? Tesseract with a trained template. Face recognition? A dedicated library, not Claude.
- **Use a multimodal generative model when the task is "read this and reason."** A vision LLM is expensive per call but cheap per engineering hour. Good for low-volume, high-complexity work.
- **Use embedding models when you need retrieval or similarity.** CLIP for image-text search, text-embedding-3 for text-only, audio embedders for "find similar sounds." Never mix embedding spaces.
- **Use a fine-tuned small model when you have labeled data.** A fine-tuned ViT for product categorization will beat GPT-4o on accuracy and cost if you have a few thousand examples and a clear taxonomy.

The mistake I see most often is people reaching for a big multimodal model for tasks that have had specialized solutions for a decade. The second most common mistake is the opposite: trying to build a specialized pipeline for a task that only shows up ten times a day, where a single API call would have been fine.

## What to take away

Shared embedding spaces are the trick that makes cross-modal search work. Generative multimodal models are the trick that makes cross-modal reasoning work. They are separate tools with separate cost profiles.

From Rust, `async-openai` and `whisper-rs` cover most of the real-world API surface. `candle` covers the cases where you want to run a model in-process. Vector databases have Rust clients. There is no longer a Python moat here for this class of work.

The engineering judgment has moved up a level. The question is rarely "can this be done" and more often "is this the right tool, at the right cost, for the right volume." That is a much nicer problem to have.

## References

- [CLIP: Connecting text and images (OpenAI)](https://openai.com/index/clip/)
- [`async-openai` on GitHub](https://github.com/64bit/async-openai)
- [`whisper-rs` on GitHub](https://github.com/tazz4843/whisper-rs)
- [`whisper.cpp`](https://github.com/ggerganov/whisper.cpp)
- [Claude Vision API docs](https://platform.claude.com/docs/en/build-with-claude/vision)
- [`candle` Rust ML framework](https://github.com/huggingface/candle)
