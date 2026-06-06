# Roadmap & Status

**Pre-alpha.** The specification, architecture, and execution template design are stable. Implementation is in active development. **Do not use in production yet.**

## Initial release targets

- [x] Architecture and execution template design
- [ ] GGUF loader (header, metadata, tensors, tokenizer assets)
- [ ] `TransformerConfig` resolver across all four templates
- [ ] CPU reference backend with quantized kernels
- [ ] LLaMA-like template end-to-end
- [ ] Token streaming via `IAsyncEnumerable<T>`
- [ ] `IChatClient` integration
- [ ] Remaining three execution templates
- [ ] Optional GPU backends

## Roadmap principles

The roadmap favors a **usable vertical slice over broad early ambition**. `llmdot` becomes credible when it can load a real GGUF model, run a complete generation loop, and integrate cleanly into ordinary .NET applications.

## Phase 0 — Foundation (complete)

- Repository structure and package boundaries
- Target frameworks: net8.0, net9.0, net10.0 (net8.0 LTS is the compatibility floor)
- Architecture and template design frozen (four execution templates covering all 1–8B model families)
- Multimodal scope: small multimodal models via pluggable modality encoders
- Quantization scope: block quants (Q4_0 / Q4_1 / Q5_0 / Q5_1 / Q8_0), K-quants (Q2_K–Q6_K), F16 / F32 / BF16
- Acceleration strategy: GPU (Vulkan, Metal) is in scope; NPU is explicitly out of scope (graph compilers, shared RAM bandwidth, ONNX conversion overhead)
- Coding conventions: C# 13, nullable enabled, warnings as errors, .editorconfig enforced

## Phase 1 — Managed core runtime

- GGUF parsing for all supported architectures
- `TransformerConfig` resolution from GGUF metadata
- `TensorNameResolver` for architecture-tolerant tensor access
- Managed tensor and buffer abstractions
- Four execution templates (LLaMA-like, GPT-NeoX-like, Gemma-like, LFM2-like)
- Token generation on a CPU-only backend
- Basic sampling and streaming APIs
- BPE tokenizer decoding from GGUF metadata
- 1D causal convolution primitive for LFM2 conv blocks

Exit criteria: a supported GGUF model from each template can be loaded directly, the runtime can generate text end-to-end on CPU for at least one model per template, `TransformerConfig` is correctly resolved from real GGUF files, token streaming works through an idiomatic public API.

## Beyond Phase 1

Subsequent phases cover broader template coverage, the `Microsoft.Extensions.AI` integration surface hardening, optional GPU backends (Metal, Vulkan), and the multimodal connectors. See the repository's `doc/roadmap.md` for the authoritative long-form plan.

## Contributing

Design feedback is welcome. Please read `doc/vision.md` and `doc/architecture.md` before opening an issue — most "why not X?" questions have explicit answers there (especially around ONNX, NPU, and native wrapping).

Areas most valuable right now:

- GGUF quantization format coverage
- Managed kernel optimization (intrinsics, vectorization)
- Tokenizer correctness across BPE variants
- Test fixtures for additional model families
