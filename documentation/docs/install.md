# Installation

`llmdot` is distributed as a small set of NuGet packages. The core runtime is the single required dependency; everything else is additive and opt-in.

!!! warning "Pre-alpha"
    The project is in active development. NuGet packages are not yet published as stable releases. Track progress in the [roadmap](status.md).

## Packages

| Package | Purpose | Dependencies |
|---|---|---|
| `Llmdot.Core` | GGUF loader, model graph, CPU backend, sampling, tokenizer | Pure managed .NET |
| `Llmdot.Extensions.AI` | `IChatClient` + `Microsoft.Extensions.AI` integration | `Llmdot.Core` |
| `Llmdot.Backends.Vulkan` *(planned)* | Vulkan compute acceleration | Native Vulkan loader |
| `Llmdot.Backends.Metal` *(planned)* | Metal compute acceleration (Apple Silicon) | Native Metal |
| `Llmdot.Multimodal.Vision` *(planned)* | SigLIP2 vision encoder + connector | `Llmdot.Core` |

The repository also ships two GPU backend projects under `src/Llmdot.Metal` and `src/Llmdot.Vulkan`, plus a CLI under `src/Llmdot.Cli`. See [GPU Backends](gpu-backends.md) and [CLI](cli.md).

## Target frameworks

`llmdot` targets:

- **net8.0** (LTS — the compatibility floor)
- **net9.0**
- **net10.0**

The core has `Nullable` enabled, treats warnings as errors, and uses `LangVersion=13.0`. It is designed to be **trimming-friendly**, **NativeAOT-friendly**, and **single-file publish-friendly**.

## Repository layout

```
llmdot/
├── src/
│   ├── Llmdot.Core/              Core runtime: GGUF loader, graph, CPU backend
│   ├── Llmdot.Extensions.AI/     Microsoft.Extensions.AI integration
│   ├── Llmdot.Cli/               llmdot command-line tool
│   ├── Llmdot.Metal/             Optional Metal compute backend
│   └── Llmdot.Vulkan/            Optional Vulkan compute backend
├── samples/
│   └── Llmdot.Sample/            Minimal end-to-end example
├── tests/
│   └── Llmdot.Core.Tests/        Unit and integration tests
├── benches/
│   └── Llmdot.Benchmarks/        BenchmarkDotNet performance suite
└── doc/                          Vision, architecture, roadmap, platform strategy
```

## Building from source

Until packages are published, clone the repository and reference the projects directly.

```bash
git clone https://github.com/cognisoc/llmdot.git
cd llmdot
dotnet build Llmdot.sln
```

Then run the sample:

```bash
dotnet run --project samples/Llmdot.Sample -- path/to/model.gguf "Hello"
```
