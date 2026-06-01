# Running OpenCode with Gemma 4 26B on macOS (via llama.cpp)

As of April 2026, **Gemma 4 tool calling is broken in Ollama v0.20.0** – the tool call parser fails and streaming drops tool calls entirely. OpenCode also has issues with local OpenAI‑compatible providers.

This guide documents a **working setup** using:

- **llama.cpp** (built from source with PR #21326 template fix + PR #21343 tokenizer fix) instead of Ollama
- **OpenCode built from source** with PR #16531 tool‑call compatibility layer

Tested on **macOS Apple Silicon** (M1 Max, 32GB) on April 2, 2026. --

---

## Why not Ollama?

Ollama has three problems with Gemma 4 right now:

1. **Tool call parser crashes** – `gemma4 tool call parsing failed: invalid character` ([#15241](https://github.com/ollama/ollama/issues/15241))
2. **Streaming drops tool calls** – tool call data goes into the reasoning field with empty content, so the actual tool call never reaches the client
3. **`<unused25>` token spam** – tokenizer bug causes garbage output

llama.cpp has fixes for all three (PRs [#21326](https://github.com/ggml-org/llama.cpp/pull/21326) and [#21343](https://github.com/ggml-org/llama.cpp/pull/21343)).

---

## Why not stock OpenCode?

Stock OpenCode can't recover when a local model:

- Returns tool calls as plain JSON text instead of proper function calls
- Sends `finish_reason: "stop"` instead of `"tool_calls"`
- Uses legacy `function_call` format instead of modern `tool_calls`

PR [#16531](https://github.com/anomalyco/opencode/pull/16531) adds a `toolParser` compatibility layer that handles all of these.
...
---

## Prerequisites

- macOS Apple Silicon with **24GB+ RAM** (32GB recommended for 26B model)
- Homebrew
- Close heavy apps (Chrome tabs, etc.) – the 26B model uses ~17GB

```bash
brew install git gh cmake bun

