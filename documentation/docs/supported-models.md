# Supported Models

All supported architectures collapse into four execution templates. Within each template, all variation is expressed through `TransformerConfig` — no per-model code paths.

## Architecture map

| `general.architecture` | Model families | Template |
|---|---|---|
| `llama` | LLaMA-1/2/3, Mistral-7B, TinyLlama-1.1B, Mixtral | LLaMA-like |
| `phi3` | Phi-3 Mini / Medium | LLaMA-like |
| `qwen2` | Qwen-2 1.5B / 7B | LLaMA-like |
| `stablelm` | StableLM-2 1.6B | LLaMA-like (with flags) |
| `phi2` | Phi-2 2.7B | GPT-NeoX-like |
| `gptneox` | Pythia 1.4B / 6.9B | GPT-NeoX-like |
| `gemma` | Gemma 2B | Gemma-like |
| `gemma2` | Gemma-2 2B / 9B | Gemma-like |
| `lfm2` | LFM2 350M / 700M / 1.2B / 2.6B, LFM2-VL | LFM2-like |
| `lfm2_moe` | LFM2-8B-A1B, LFM2-24B-A2B | LFM2-like (MoE) |

## Execution templates

### LLaMA-like (sequential pre-norm)

```
x → norm → attn → + → norm → ffn → + → output
                  ↑                  ↑
                  x                  x
```

- Pre-norm placement (norm before attention and FFN)
- Separate Q/K/V tensors (or fused, handled by `TensorNameResolver`)
- SwiGLU or GeGLU FFN (detected by presence of gate tensor)
- No post-norm by default

### GPT-NeoX-like (parallel residual)

```
x → norm → attn → + → output
                  ↑
       norm → ffn ┘
                  ↑
                  x
```

Parallel attention and FFN, fused QKV, partial rotary.

### Gemma-like

Embedding scaling, post-norm, softcapping.

### LFM2-like (hybrid convolution-attention)

LFM2 (Liquid Foundation Model 2) replaces ~62% of attention layers with double-gated LIV (Liquid Input-Varying) causal convolutions — significantly faster on-device than a pure transformer of comparable size.

Key properties:

- Hybrid layer layout: most layers are gated convolutions (kernel=3); only ~38% are GQA attention layers
- GQA: 16/32 Q heads, 8 KV heads (2:1 ratio), RoPE with θ = 1,000,000
- SwiGLU FFN with auto-adjusted intermediate dimension
- Vocab: 65,536 (custom BPE with ChatML-style special tokens)
- MoE variants: 32 experts, top-4 routing (8B-A1B, 24B-A2B)
- Context: 32K

Multimodal LFM2 variants:

- **LFM2-VL** — vision-language (SigLIP2 vision encoder + LFM2 backbone + 2-layer MLP connector)
- **LFM2-Audio** — speech-to-speech (FastConformer encoder + Mimi codec decoder)

The LFM2 template introduces a new structural primitive — the **gated convolution block** — that does not exist in the other three templates, which is why it gets its own execution template rather than being expressed through the LLaMA-like path.

## Multimodal

Multimodal variants (vision-language via SigLIP2, speech via FastConformer / Mimi) plug in as **modality encoders** on top of the base LLM backbone. The core runtime is unchanged. Sentinel tokens in the tokenizer trigger modality-specific processing.

## Quantization formats

The core covers all common GGUF types:

- Block quantization: `Q4_0`, `Q4_1`, `Q5_0`, `Q5_1`, `Q8_0`
- K-quants: `Q2_K` through `Q6_K`
- Float: `F16`, `F32`, `BF16`

Dequantization is dispatched on `Loading.GgmlType` through `IComputeBackend.DequantizeToFloat` (or `TensorOps.DequantizeToFloat` for the loader-side caching path in `LoadedModel`).

## Reference

The repository's `doc/model-architectures.md` is the long-form architecture reference and supersedes anything that drifts between docs.
