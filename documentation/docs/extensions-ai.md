# Microsoft.Extensions.AI Integration

`Llmdot.Extensions.AI` adapts `llmdot` to the `Microsoft.Extensions.AI` abstractions, exposing the runtime as an `IChatClient` that can be registered through standard .NET dependency injection.

## Quick start

```csharp
using Llmdot.Extensions.AI;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.DependencyInjection;

var services = new ServiceCollection();

services.AddLlmdot("phi-3-mini-q4_k_m.gguf");

var sp = services.BuildServiceProvider();
var chat = sp.GetRequiredService<IChatClient>();

await foreach (var update in chat.GetStreamingResponseAsync(
    [new ChatMessage(ChatRole.User, "Explain GGUF in one paragraph.")]))
{
    Console.Write(update.Text);
}
```

## Registration overloads

```csharp
// Just give a model path:
services.AddLlmdot("path/to/model.gguf");

// Or configure all options:
services.AddLlmdot(options =>
{
    options.ModelPath      = "path/to/model.gguf";
    options.ContextLength  = 4096;
    options.MaxTokens      = 256;
    options.Temperature    = 0.8f;
    options.TopK           = 40;
    options.TopP           = 0.95f;
    options.RepeatPenalty  = 1.1f;
});
```

Both overloads register `LlmdotChatClient` as a singleton and bind it to `IChatClient`.

## `LlmdotOptions`

```csharp
public sealed class LlmdotOptions
{
    public string ModelPath      { get; set; } = string.Empty;
    public int    ContextLength  { get; set; } = 4096;
    public int    MaxTokens      { get; set; } = 256;
    public float  Temperature    { get; set; } = 0.8f;
    public int    TopK           { get; set; } = 40;
    public float  TopP           { get; set; } = 0.95f;
    public float  RepeatPenalty  { get; set; } = 1.1f;
}
```

`ModelPath` is required. `LlmdotChatClient` throws on construction if it is unset.

## `LlmdotChatClient`

`LlmdotChatClient` implements `IChatClient`:

```csharp
public sealed class LlmdotChatClient : IChatClient
{
    public Task<ChatResponse> GetResponseAsync(
        IEnumerable<ChatMessage> messages,
        ChatOptions? options = null,
        CancellationToken cancellationToken = default);

    public IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IEnumerable<ChatMessage> messages,
        ChatOptions? options = null,
        CancellationToken cancellationToken = default);

    public ChatClientMetadata Metadata { get; }
    public void Dispose();
}
```

Behaviour:

- Messages are projected to `ChatMessageEntry(role, content)` tuples. If the model has a `ChatTemplate`, it is used to format the prompt; otherwise the `<role>content` fallback runs.
- Per-call `ChatOptions` overrides win over the registered `LlmdotOptions` for `MaxOutputTokens`, `Temperature`, and `StopSequences`.
- `Metadata.ProviderName` is `"llmdot"`. `Metadata.DefaultModelId` is the resolved architecture string from the GGUF metadata.

## Backend selection

`Llmdot.Extensions.AI` also exposes `BackendFactory` for selecting the best available compute backend (Metal / Vulkan / CPU). See [GPU Backends](gpu-backends.md).
