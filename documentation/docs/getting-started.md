# Getting Started

This page walks through the working API as it exists in the repository today. The README also documents a future "target API shape" (`LlmModel.LoadAsync` / `model.CreateChatSession`); the implementation currently exposes `LoadedModel`, `InferenceEngine`, and `ChatSession`.

## Prerequisites

- .NET SDK targeting **net8.0**, **net9.0**, or **net10.0**
- A GGUF model file (e.g. a quantized Phi-3, LLaMA, Qwen2, Gemma, or LFM2 model)

## Loading a model and streaming chat tokens

The sample app in `samples/Llmdot.Sample/Program.cs` shows the end-to-end flow.

```csharp
using Llmdot.Inference;
using Llmdot.Models;

using var stream = File.OpenRead("phi-3-mini-q4_k_m.gguf");
var model = LoadedModel.Load(stream);

var session = new ChatSession(model);
var options = new GenerationOptions { MaxTokens = 256 };

await foreach (var text in session.GenerateAsync("Explain GGUF in one paragraph.", options))
    Console.Write(text);

model.Dispose();
```

`LoadedModel.Load(stream)` takes ownership of the stream and disposes it when the model is disposed.

`ChatSession.GenerateAsync` returns an `IAsyncEnumerable<string>` of decoded text pieces, formatted through the model's chat template if one is present in the GGUF metadata. If the model has no chat template, a simple `<role>content` fallback is used.

## Raw text completion

When you want the model to continue text without any chat templating, use `InferenceEngine` directly:

```csharp
var engine = new InferenceEngine(model);
var tokens = model.Tokenizer.Encode("The capital of France is");

if (model.Config.BosTokenId > 0)
    tokens = [model.Config.BosTokenId, .. tokens];

var options = new GenerationOptions { MaxTokens = 64 };

await foreach (var tokenId in engine.Generate(tokens, options))
    Console.Write(model.Tokenizer.Decode([tokenId]));
```

## Multi-turn chat

`ChatSession` keeps an internal `_history` of user/assistant turns and re-formats the full conversation each call. Call `Reset()` to clear it.

```csharp
var session = new ChatSession(model);

await foreach (var t in session.GenerateAsync("Hi, who are you?")) Console.Write(t);
await foreach (var t in session.GenerateAsync("Repeat that in French.")) Console.Write(t);

session.Reset(); // clear conversation
```

## Cancellation

All streaming APIs accept a `CancellationToken`. Hooking it up to Ctrl+C is the same pattern as any .NET console app:

```csharp
var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

await foreach (var text in session.GenerateAsync(prompt, options, cts.Token))
    Console.Write(text);
```

## Next steps

- [Installation](install.md) — package layout and target frameworks
- [Model Loading](model-loading.md) — what `LoadedModel.Load` actually does
- [API Surface](api.md) — public types in `Llmdot.Core`
- [CLI](cli.md) — the `llmdot` command-line tool
- [Microsoft.Extensions.AI](extensions-ai.md) — `IChatClient` integration
