# llmdot

**Run local GGUF language models from .NET — one package, one format, one programming model.**

`llmdot` is a native .NET runtime for local language model inference built around the **GGUF** model format. It executes major decoder-only transformer and hybrid architectures in the **1–8B parameter range** — including multimodal variants — through architecture-agnostic execution templates resolved from GGUF metadata at load time.

The project is designed around a single opinionated goal: **make local LLM execution in .NET as simple as adding a NuGet package, loading a GGUF file, and streaming tokens**. The default path is pure managed code with zero native runtime dependencies, focused on CPU-first execution. Optional packages provide GPU acceleration through thin backend adapters.

```csharp
using Llmdot;

await using var model = await LlmModel.LoadAsync("phi-3-mini-q4_k_m.gguf");
await using var session = model.CreateChatSession();

await foreach (var token in session.StreamAsync("Explain GGUF in one paragraph."))
    Console.Write(token);
```

!!! note "Target API shape"
    The snippet above reflects the target API shape. The implementation is **pre-alpha**; the in-repo API surface today uses `LoadedModel.Load(stream)`, `ChatSession`, and `InferenceEngine`. See [Getting Started](getting-started.md) for the working API.

## Why llmdot

The .NET inference landscape today forces developers into one of two uncomfortable tradeoffs:

| Option | Strength | Tradeoff |
|---|---|---|
| `llama.cpp` bindings | Broad model support | Native binaries, per-platform packaging, upstream integration debt |
| ONNX-based stacks | Strong hardware acceleration | Model conversion, large native dependencies, toolchain friction |

`llmdot` takes a third position:

- **GGUF-native** execution with no conversion pipeline
- **Pure managed core** — trimming-friendly, NativeAOT-friendly, single-file publish-friendly
- **Idiomatic .NET APIs** built for `IAsyncEnumerable<T>` streaming, DI, and `Microsoft.Extensions.Hosting`
- **Config-driven architectures** — new model families plug into existing execution templates with zero engine code
- **Focus on the common case** — small-to-mid quantized models where developer experience beats peak throughput

## Who llmdot is for

- **.NET developers** who want to ship local, private, or offline AI features without fighting the inference stack.
- **Software architects** evaluating local LLM runtimes for desktop, edge, and server workloads where packaging simplicity, deployment predictability, and platform portability matter as much as raw throughput.
- **Teams building on `Microsoft.Extensions.AI`** who need an `IChatClient`-compatible backend that runs fully in-process, with no sidecar services and no native toolchain.

## Design principles

1. **Zero native dependencies in the core path.** The default install is pure managed .NET. Native acceleration is always additive.
2. **GGUF is the ingestion format.** No ONNX conversion. No proprietary packaging. Community models work out of the box.
3. **Architecture support is declarative, not hard-coded.** New model families are resolved through `TransformerConfig` from GGUF metadata.
4. **Model compatibility is decoupled from hardware backend.** CPU, Vulkan, or Metal — same model, same code.
5. **Optimize for the common case.** 1–8B quantized models on consumer hardware. Small enough to fit, big enough to matter.
6. **Incremental acceleration.** Backends offload individual operations, not entire graphs. No all-or-nothing rewrites.

## Status

**Pre-alpha.** The specification, architecture, and execution template design are stable. Implementation is in active development. Do not use in production yet. See [Roadmap & Status](status.md).

## License

MIT.
