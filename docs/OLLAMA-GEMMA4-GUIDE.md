# 🦙 Ollama + Gemma 4 26B Setup Guide

A step-by-step recipe for running the LLM Wiki plugin fully locally on
**gemma4:26b** via Ollama, using the tuned Modelfile in
[`docs/gemma4-26b-wiki.Modelfile`](./gemma4-26b-wiki.Modelfile).

Gemma 4 26B A4B is one of the plugin's recommended local models (see the
[Model Guide](./MODEL-GUIDE.md#-local-model-recommendations-ollama--lm-studio)):
strong instruction following, clean Markdown output, 262K native context,
and a MoE design that keeps memory below the 31B dense variant.

---

## Why a custom Modelfile at all?

The plugin talks to Ollama through the OpenAI-compatible endpoint
(`http://localhost:11434/v1`). That protocol has **no context-size field**,
so Ollama runs the model at its default context (4096 tokens unless you
raised `OLLAMA_CONTEXT_LENGTH`). The plugin's real traffic is much bigger:

| Call path | Input | Output budget |
|-----------|-------|---------------|
| Lint duplicate detection | up to ~15K tokens | 8K tokens |
| Query Wiki (10-page context) | up to ~32K tokens | 8K tokens |
| Source extraction batch | 1–3K prompt + source chunk | up to 16K tokens |

At 4K context Ollama **silently truncates the prompt** — no error, just
wrong or empty answers, dead-link mismatches, and lint batches that return
nothing. The Modelfile bakes in `num_ctx 65536` so every call path fits,
and pins sampling values suited to schema-constrained JSON extraction.

## 1. Build the model

```bash
ollama pull gemma4:26b
cd <vault-or-download-location>   # wherever you saved the Modelfile
ollama create gemma4-wiki -f gemma4-26b-wiki.Modelfile
ollama show gemma4-wiki           # confirm: context length 65536
```

## 2. Point the plugin at it

In Obsidian → Settings → LLM Wiki → LLM Configuration:

| Setting | Value |
|---------|-------|
| Provider | **Ollama (Local)** |
| Base URL | `http://localhost:11434/v1` (default) |
| API Key | `ollama` (any non-empty string) |
| Model | `gemma4-wiki` |

Click **Test Connection**, then ingest a small source note as a smoke test.

## 3. Memory tuning

64K context needs KV-cache room on top of the Q4 weights. If you hit
out-of-memory or heavy CPU offload, set these **server-side** environment
variables before starting Ollama:

```bash
export OLLAMA_FLASH_ATTENTION=1     # required for KV cache quantization
export OLLAMA_KV_CACHE_TYPE=q8_0    # ~halves KV cache memory, negligible quality loss
```

| Hardware | Suggested `num_ctx` |
|----------|---------------------|
| 24 GB (Apple Silicon / single consumer GPU) | 32768–65536 with `q8_0` KV cache |
| 48 GB+ | 65536 (default) or higher for very large vaults |

To change `num_ctx`, edit the Modelfile and re-run `ollama create` — it
rebuilds instantly (the weights are shared, only parameters change).

## 4. Troubleshooting

- **Empty responses or JSON cut off mid-object** — Gemma 4 is
  reasoning-capable, and its deliberation is billed against the same
  output budget as the answer. The plugin's token budgets already carry
  reasoning headroom, but if you still see empty results, enable
  **Disable thinking** in the plugin's LLM settings (the plugin walks its
  thinking-control fallback chain automatically).
- **Answers reference pages that don't exist / dead links after ingest** —
  classic context-truncation symptom. Verify `ollama show gemma4-wiki`
  reports 65536, and that you selected `gemma4-wiki`, not the raw
  `gemma4:26b`.
- **Repetitive or garbled extraction output** — do **not** raise
  `repeat_penalty` above 1.0; the plugin's settings warn it breaks
  grammar-constrained JSON on local models. Prefer lowering temperature
  toward 0.1 or raising it slightly toward 0.4 in the plugin's Custom
  advanced parameters.
- **Slow first call after idle** — Ollama unloads models after 5 minutes.
  Run `ollama run gemma4-wiki ""` to pre-warm, or raise `OLLAMA_KEEP_ALIVE`.

## Notes on what the Modelfile deliberately leaves alone

- `num_predict` — the plugin sends an explicit `max_tokens` per call
  (2K–16K by task); Ollama maps it onto `num_predict` per request.
- `SYSTEM` / `TEMPLATE` — the plugin ships its own system prompts, and the
  chat template comes from the base model. Overriding either degrades or
  breaks the pipeline.
- Temperature/top_p in the plugin's **Custom** mode override the Modelfile
  values per call; in **Default** mode (recommended) the Modelfile governs.
