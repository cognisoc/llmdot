# Model Loading

All loading goes through `LoadedModel.Load(Stream)` in `Llmdot.Core`.

```csharp
using var stream = File.OpenRead("model.gguf");
var model = LoadedModel.Load(stream);
```

The stream is owned by the returned `LoadedModel` and is disposed when the model is disposed.

## What loading does

`LoadedModel.Load` runs the following steps:

1. **`GgufReader.Read(stream)`** parses the GGUF header, metadata, and tensor descriptors.
2. **`ArchitectureResolver.Resolve(ggufModel)`** produces a `TransformerConfig` from GGUF metadata keys. This is the unified config every execution template reads from — never raw GGUF keys.
3. **`TensorNameResolver`** indexes the tensor descriptors so the graph can look up tensors by logical name across architectures.
4. **Tensor data** is read from the file at `TensorDataOffset + info.Offset` for each tensor and stored as a `Tensor` (name, dimensions, element type, raw bytes, element count).
5. **`BpeTokenizer.FromGguf(metadata)`** builds the BPE tokenizer from vocab/merges in the GGUF metadata.
6. **`ChatTemplate.FromGguf(metadata, tokenizer)`** parses an optional chat template if present.

## Inspecting a loaded model

`LoadedModel` exposes:

- `Config` — the resolved `TransformerConfig` (hidden size, layer count, context length, vocab, heads, rope params, BOS/EOS, FFN type, embedding scale, conv params for hybrid models, etc.)
- `Tokenizer` — the BPE tokenizer
- `ChatTemplate` — nullable; present only if the GGUF metadata included one
- `Capabilities` — derived from `Config`, including the architecture name and execution template
- `Validate()` — runs `ModelValidator` over the loaded GGUF and config

The CLI's `info` command and the sample app both print a summary of these properties.

## Validation

```csharp
var result = model.Validate();
```

`ModelValidator.Validate` runs through the GGUF model and resolved config to surface issues before inference.

## Disposal

`LoadedModel` owns the underlying stream. Always dispose it:

```csharp
model.Dispose();
```

Or with a `using` declaration:

```csharp
using var stream = File.OpenRead(path);
using var model = LoadedModel.Load(stream);
```

## Loaded layout

Internally, `LoadedModel`:

- Reads each tensor's bytes from the file into memory
- Caches per-tensor dequantized `float[]` buffers on first access via `GetDequantizedWeights(name)` so repeated reads of the same weight don't repeat the dequantization
- Uses `TensorOps.DequantizeToFloat` driven by `Loading.GgmlType` (Q4_0/Q4_1/Q5_0/Q5_1/Q8_0, Q2_K–Q6_K, F16/F32/BF16)

See [Architecture](architecture.md) for the broader execution-template story this loading step feeds into.
