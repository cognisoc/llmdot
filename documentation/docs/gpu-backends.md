# GPU Backends

`llmdot` ships two optional GPU compute backends in addition to the default managed CPU backend:

- **`Llmdot.Metal`** — Metal compute on Apple Silicon
- **`Llmdot.Vulkan`** — Vulkan compute on Windows and Linux

Both implement the same `IComputeBackend` interface as `CpuBackend`, so model compatibility is fully decoupled from hardware backend — same model, same code.

## Selecting a backend

You can pass a backend explicitly to `InferenceEngine`:

```csharp
using Llmdot.Inference;

var engine = new InferenceEngine(model, new CpuBackend());
```

For automatic detection, `Llmdot.Extensions.AI` ships a `BackendFactory`:

```csharp
using Llmdot.Extensions.AI;

IComputeBackend backend = BackendFactory.CreateBestAvailable();

// Or with the chosen name surfaced for logging:
var (backend, name) = BackendFactory.CreateBestAvailableWithInfo();
Console.WriteLine($"Using backend: {name}");

// Or force CPU:
var cpu = BackendFactory.CreateCpu();
```

`CreateBestAvailable` tries Metal on macOS (by probing `/System/Library/Frameworks/Metal.framework/Metal`), then Vulkan on Linux/Windows (`libvulkan.so.1` / `vulkan-1`), and falls back to `CpuBackend`.

## Metal backend

`Llmdot.Metal.MetalBackend` implements `IComputeBackend` over Metal compute pipelines. The backend probes for Metal at construction time and falls back gracefully if Metal is not available.

Internals (in the repo):

- `MetalRuntime` and `MetalInterop` manage the Metal device, queue, and buffer pools.
- Shaders live under `src/Llmdot.Metal/Shaders/`.
- Persistent and scratch buffers are managed inside the backend.

## Vulkan backend

`Llmdot.Vulkan.VulkanBackend` implements `IComputeBackend` over Vulkan compute pipelines, with `VulkanRuntime` providing device and queue management. Shaders live under `src/Llmdot.Vulkan/Shaders/`.

`VulkanBackendOptions` exposes:

```csharp
public sealed class VulkanBackendOptions
{
    public bool EnableValidationLayers { get; set; }
    public int DeviceIndex { get; set; }
}
```

## Acceleration model

Backends offload **individual operations** (matmul, RMS/LayerNorm, RoPE, softmax, SiLU/GELU, add/scale/mul, softcap, 1D causal convolution, dequantize, argmax) — not entire graphs. There are no all-or-nothing rewrites. The CPU backend remains the fallback for any operation a GPU backend chooses not to accelerate.

This is the project's stated **incremental acceleration** principle: GPU acceleration is additive and never gates which models you can run.
