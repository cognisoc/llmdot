# API Surface

Public types in `Llmdot.Core`, grouped by namespace.

## `Llmdot.Inference`

### `LoadedModel`

A GGUF model loaded into memory, ready for inference.

```csharp
public sealed class LoadedModel : IDisposable
{
    public TransformerConfig Config { get; }
    public BpeTokenizer Tokenizer { get; }
    public ChatTemplate? ChatTemplate { get; }
    public ModelCapabilities Capabilities { get; }

    public static LoadedModel Load(Stream stream);
    public ModelValidationResult Validate();
    public void Dispose();
}
```

The stream is owned by the returned instance and disposed with it.

### `InferenceEngine`

Runs autoregressive inference on a `LoadedModel` using an `IComputeBackend` for tensor operations.

```csharp
public sealed class InferenceEngine
{
    public InferenceEngine(LoadedModel model);                          // CPU backend
    public InferenceEngine(LoadedModel model, IComputeBackend backend); // custom backend

    public IAsyncEnumerable<int> Generate(
        int[] promptTokens,
        GenerationOptions options,
        CancellationToken cancellationToken = default);
}
```

`Generate` yields token IDs. Decode with `model.Tokenizer.Decode(...)`.

### `ChatSession`

Multi-turn chat over a `LoadedModel`. Handles prompt formatting via the model's `ChatTemplate` (or a `<role>content` fallback), encoding, decoding, and stop-sequence detection.

```csharp
public sealed class ChatSession
{
    public ChatSession(LoadedModel model);

    public IAsyncEnumerable<string> GenerateAsync(
        string prompt,
        GenerationOptions? options = null,
        CancellationToken cancellationToken = default);

    public void Reset();
}
```

### `GenerationOptions`

```csharp
public sealed class GenerationOptions
{
    public int MaxTokens { get; init; } = 256;
    public SamplingOptions Sampling { get; init; } = new();
    public string? StopSequence { get; init; }
    public string[] StopSequences { get; init; } = [];
}
```

A `GenerationOptionsBuilder` is also provided for fluent construction.

### `IComputeBackend`

Tensor compute abstraction so CPU and GPU backends are interchangeable. All operations work on contiguous `Span<float>` (or quantized `ReadOnlySpan<byte>` for weights).

```csharp
public interface IComputeBackend : IDisposable
{
    void MatMul(ReadOnlySpan<float> a, ReadOnlySpan<byte> b, Span<float> result,
                GgmlType bType, int aCols, int bCols, string? weightKey = null);
    void MatMulF32(ReadOnlySpan<float> a, ReadOnlySpan<float> b, Span<float> result,
                   int aCols, int bCols, string? weightKey = null);
    void RmsNorm(ReadOnlySpan<float> input, ReadOnlySpan<float> weights, Span<float> output, float epsilon);
    void LayerNorm(ReadOnlySpan<float> input, ReadOnlySpan<float> weights, ReadOnlySpan<float> bias,
                   Span<float> output, float epsilon);
    void ApplyRoPE(Span<float> query, Span<float> key, int headDim, int position, float freqBase, int rotaryDim);
    void ApplyRoPE(Span<float> query, Span<float> key, int headDim, int position, ReadOnlySpan<float> freqTable);
    void Softmax(Span<float> input, float? softcap = null);
    void Silu(ReadOnlySpan<float> input, Span<float> result);
    void SiluInPlace(Span<float> input);
    void Gelu(ReadOnlySpan<float> input, Span<float> result);
    float GeluScalar(float x);
    void Add(Span<float> a, ReadOnlySpan<float> b);
    void Add(ReadOnlySpan<float> a, ReadOnlySpan<float> b, Span<float> result);
    void Scale(ReadOnlySpan<float> input, float scale, Span<float> result);
    void Scale(Span<float> input, float scale);
    void Mul(ReadOnlySpan<float> a, ReadOnlySpan<float> b, Span<float> result);
    void Softcap(Span<float> input, float cap);
    void Conv1D(ReadOnlySpan<float> input, ReadOnlySpan<float> weights, Span<float> output,
                int kernelSize, int inputDim);
    void DequantizeToFloat(ReadOnlySpan<byte> src, Span<float> dst, GgmlType type, int numRows, int rowElements);
    int ArgMax(ReadOnlySpan<float> input);
}
```

`CpuBackend` is the default managed implementation. `MetalBackend` and `VulkanBackend` (in their own assemblies) implement the same interface.

### `ModelCapabilities`, `ModelInfo`

Derived descriptors of a loaded model (architecture name, execution template, etc.). See `ModelCapabilities.FromConfig(config)`.

## `Llmdot.Sampling`

### `SamplingOptions`

```csharp
public sealed class SamplingOptions
{
    public int TopK { get; init; } = 40;
    public float TopP { get; init; } = 0.95f;
    public float Temperature { get; init; } = 0.8f;
    public float RepeatPenalty { get; init; } = 1.1f;
    public int RepeatPenaltyWindowSize { get; init; } = 64;
    public int Seed { get; init; } = -1;       // -1 = random, >=0 = deterministic
}
```

`Sampler` consumes these options and runs over logits using the chosen `IComputeBackend`.

## `Llmdot.Loading`

GGUF parsing primitives: `GgufReader`, `GgufModel`, `GgufMetadata`, `GgufTensorInfo`, `GgufConstants`, `GgufValueType`, `GgmlType`, `ArchitectureResolver`, `TensorNameResolver`, `ModelValidator`, `ModelValidationResult`.

`ArchitectureResolver.Resolve(ggufModel)` is what produces the `TransformerConfig` that every execution template reads from.

## `Llmdot.Tokenization`

`BpeTokenizer` with `Encode(string) -> int[]` and `Decode(IEnumerable<int>) -> string`. Constructed via `BpeTokenizer.FromGguf(metadata)`.

`ChatTemplate` parses the GGUF chat template (Jinja-style) via `ChatTemplate.FromGguf(metadata, tokenizer)` and renders a list of `ChatMessageEntry(role, content)` to a prompt string.

## `Llmdot.Tensors`

`Tensor`, `TensorOps`, and `TensorSize` describe and manipulate tensor data. `Dequantize` and `Numeric` subnamespaces hold the dequantization and numeric helpers.
