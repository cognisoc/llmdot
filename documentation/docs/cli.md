# Command-Line Tool

`llmdot` ships a CLI (`src/Llmdot.Cli`) for inspecting models and running one-shot or interactive generation without writing any code.

## Usage

```text
Usage: llmdot <command> [options]

Commands:
  info <model.gguf>                          Show model metadata
  chat <model.gguf> [prompt] [options]       Interactive or one-shot chat
  complete <model.gguf> <prompt> [options]   Raw text completion

Options:
  --max-tokens N        Maximum tokens to generate (default: 256)
  --temperature F       Sampling temperature (default: 0.8, 0=greedy)
  --top-k N             Top-K sampling (default: 40)
  --top-p F             Top-P/nucleus sampling (default: 0.95)
  --repeat-penalty F    Repeat penalty (default: 1.1)
  --seed N              Random seed (-1=random, default: -1)
```

## Examples

Show model metadata (architecture, layer count, vocab, heads, rope params, chat template presence, etc.):

```bash
llmdot info phi-3-mini-q4_k_m.gguf
```

One-shot chat — formats the prompt through the model's chat template if available, otherwise uses a `<role>content` fallback:

```bash
llmdot chat phi-3-mini-q4_k_m.gguf "Explain GGUF in one paragraph."
```

Interactive chat — omit the prompt argument:

```bash
llmdot chat phi-3-mini-q4_k_m.gguf
```

Raw completion — bypasses chat templating entirely:

```bash
llmdot complete tinyllama-1.1b-q8_0.gguf "The capital of France is" --max-tokens 32 --temperature 0
```

Greedy / deterministic sampling:

```bash
llmdot complete model.gguf "Hello" --temperature 0 --seed 42
```

## How the sample app differs

The `samples/Llmdot.Sample` project is a smaller, self-contained example that uses `LoadedModel`, `ChatSession`, and `InferenceEngine` directly. It accepts `--raw` and `--chat` flags to override the auto-detected mode (chat is used when the model has a chat template; raw otherwise).

```bash
dotnet run --project samples/Llmdot.Sample -- model.gguf "Hello"
dotnet run --project samples/Llmdot.Sample -- model.gguf --raw "The capital of France is"
```
