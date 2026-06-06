# Architecture

`llmdot` uses a layered architecture that separates model loading, execution orchestration, and hardware acceleration. The core runtime is pure managed code. Backend adapters accelerate a narrow set of expensive primitives when available.

The runtime supports major decoder-only transformer and hybrid architectures in the 1–8B parameter range through **four execution templates** and a unified `TransformerConfig` resolved from GGUF metadata at load time. No per-model code paths are needed — all variation is expressed through configuration.

## High-level flow

1. Load a GGUF model and parse metadata, tensor layout, tokenizer assets, and quantization details.
2. Resolve a `TransformerConfig` from GGUF metadata keys and tensor names.
3. Select the appropriate execution template based on config flags.
4. Execute token generation through a managed inference loop.
5. Dispatch expensive tensor operations to the selected compute backend.
6. Stream decoded tokens through idiomatic .NET APIs.

## Layer diagram

```
 ┌────────────────────────────────────────────────────────────────┐
 │  Application  (ASP.NET Core, desktop, CLI, worker service)     │
 └──────────────┬─────────────────────────────────────────────────┘
                │  IChatClient / IAsyncEnumerable<string>
 ┌──────────────▼─────────────────────────────────────────────────┐
 │  Llmdot.Extensions.AI   (Microsoft.Extensions.AI integration)  │
 ├────────────────────────────────────────────────────────────────┤
 │  Llmdot.Core                                                   │
 │  ┌───────────┐  ┌─────────────────┐  ┌──────────────────────┐  │
 │  │ GGUF      │  │ Architecture    │  │ Sampling & Tokenizer │  │
 │  │ Loader    │─▶│ Resolver        │─▶│                      │  │
 │  └───────────┘  └────────┬────────┘  └──────────────────────┘  │
 │                          ▼                                     │
 │                 ┌─────────────────┐                            │
 │                 │ Model Graph     │  4 execution templates     │
 │                 │ + KV / Conv     │  resolved from config      │
 │                 │   State         │                            │
 │                 └────────┬────────┘                            │
 │                          ▼                                     │
 │                 ┌─────────────────┐                            │
 │                 │ Tensor Runtime  │  managed kernels,          │
 │                 │ (IComputeBackend)│  Span<T>, intrinsics      │
 │                 └────────┬────────┘                            │
 └──────────────────────────┼─────────────────────────────────────┘
                            ▼
               ┌────────────┴────────────┐
               │                         │
         ┌─────▼─────┐            ┌──────▼──────┐
         │ CPU       │            │ Optional    │
         │ (default, │            │ GPU: Vulkan │
         │  managed) │            │ / Metal /   │
         │           │            │ CUDA        │
         └───────────┘            └─────────────┘
```

The model graph reads only from a resolved `TransformerConfig` — never from raw GGUF keys directly. This is the central abstraction that eliminates per-architecture code paths.

## Four execution templates

| Template | Architectures | Example Models |
|---|---|---|
| **LLaMA-like** (sequential pre-norm) | `llama`, `phi3`, `qwen2`, `stablelm`, `mistral` | LLaMA-3.2, Qwen-2, Phi-3, Mistral-7B, StableLM-2 |
| **GPT-NeoX-like** (parallel residual) | `gptneox`, `phi2` | Pythia, Phi-2 |
| **Gemma-like** (embedding scaling + post-norm) | `gemma`, `gemma2` | Gemma 2B, Gemma-2 2B/9B |
| **LFM2-like** (hybrid convolution-attention) | `lfm2`, `lfm2_moe` | LFM2 350M–2.6B, LFM2-VL, LFM2-8B-A1B |

Within each template, all variation is expressed through `TransformerConfig` values.

See [Supported Models](supported-models.md) for the full architecture-to-model mapping. The repository's `doc/architecture.md` and `doc/model-architectures.md` contain the long-form design rationale.

## Goals and non-goals

**Goals**

- Load and execute supported GGUF models directly from .NET
- Cover all major 1–8B decoder architectures via the four execution templates
- Support small multimodal models (vision-language, audio) through pluggable modality encoders
- Provide a clean chat and text-generation API with async streaming and cancellation
- Integrate naturally with `Microsoft.Extensions.AI` abstractions
- Deliver strong CPU performance for quantized small-to-mid-sized models
- Offer optional GPU compute backends (Vulkan, Metal) without coupling model support to a vendor format

**Non-goals**

- Be the fastest inference engine on every hardware target
- Replace vendor-optimized GPU runtimes for large-scale serving
- Require ONNX conversion or proprietary model packaging
- Target frontier-scale (70B+) models as an early milestone
- Accelerate via NPU — NPUs are graph compilers, not programmable compute
