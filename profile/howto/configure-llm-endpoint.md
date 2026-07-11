# 2. Configure Your LLM Endpoint

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Set up your dev environment](setup-environment.md) · Next: [Developing practices](developing-practices.md).*

Every LLM-based judge receives its endpoint, model, and API key through a single injected object — the `llm_config` parameter passed to your judge methods — and must read them from there rather than hardcode them. That indirection is what allows the same judge code to run against your local endpoint during development and against the organizer-provided endpoint inside TIRA's sandbox, unchanged.

## What your judge receives

All three protocol methods (`judge`, `create_nuggets`, `create_qrels`) get an `llm_config: LlmConfigBase` carrying:

| Field | Content |
|-------|---------|
| `base_url` | endpoint URL (OpenAI-compatible) |
| `model` | resolved model identifier |
| `api_key` | key/token (may be empty for local endpoints) |
| `cache_dir` | prompt-cache directory, if enabled (see [Prompt cache](prompt-cache.md)) |
| `raw` | the full config dict, for any extra backend parameters |

## Choose your LLM client — any OpenAI-compatible client works

The framework hands you endpoint coordinates, not a client, so you pick the library:

- **minima-llm** (used by the starter-kit examples) adds prompt caching, rate limiting, and retry on top of a zero-dependency client:
  ```python
  from minima_llm import MinimaLlmConfig, OpenAIMinimaLlm

  full_config = MinimaLlmConfig.from_dict(llm_config.raw)
  backend = OpenAIMinimaLlm(full_config)
  ```
- **DSPy** wires in through minima-llm's adapter, so DSPy programs inherit caching and retry while still following the injected endpoint: `run_dspy_batch(...)` wraps your backend as a `MinimaLlmDSPyLM` and binds it via `dspy.context`. See [minima-llm — With DSPy](https://github.com/trec-auto-judge/minima-llm#quick-start).
- **litellm, LangChain, the raw `openai` SDK, or plain HTTP** work too — read `llm_config.base_url` / `.model` / `.api_key` and construct your client from them. minima-llm's [proxy mode](https://github.com/trec-auto-judge/minima-llm#proxy-mode) can add caching to any such client without code changes.

Whatever you choose, never bake an endpoint or key into your code — a hardcoded value would break under TIRA, where the organizer decides which model your judge gets.

## Local development: three config layers

Configuration resolves as **env → yaml → cli**, each layer overriding the previous.

**Environment variables** form the base layer:

```bash
export OPENAI_BASE_URL="http://localhost:8000/v1"   # any OpenAI-compatible endpoint: vLLM, Ollama, hosted OpenAI, ...
export OPENAI_MODEL="llama-3.3-70b-instruct"
export OPENAI_API_KEY="sk-..."                       # optional for unsecured local endpoints
export CACHE_DIR="./cache"                           # optional, enables prompt caching
```

**A YAML file** overrides the environment when passed via `--llm-config`:

```yaml
# llm-config.dev.yml — direct endpoint for local development
base_url: "http://localhost:11434/v1"
model: "llama3.2"
api_key: ""
cache_dir: "./cache"
```

```bash
auto-judge run --llm-config llm-config.dev.yml --workflow ...
```

## On TIRA: how the endpoint reaches your judge

Inside TIRA your judge runs in a sandbox without internet access, so the endpoint must be injected. Two mechanisms exist:

1. **Forwarded environment variables.** When you submit, `--forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL` carries your shell's values into the sandbox, where they populate `llm_config` exactly as in local development. The [submission guide](submit-to-tira.md) shows this in context.

2. **Model preferences (submission mode).** Instead of naming an endpoint, your repo ships an `llm-config.yml` declaring an ordered wish list that the organizers resolve against their model pool:

   ```yaml
   model_preferences:
     - "llama-3.3-70b-instruct"
     - "gpt-4o"
   on_no_match: "use_default"   # or "error" to fail with the list of available models
   ```

   At startup the first available preference wins, and your judge receives a ready-to-use `llm_config` with the organizer-chosen endpoint. Test the resolution locally with `auto-judge list-models --resolve llm-config.yml`.

Keeping both files in your repo — `llm-config.dev.yml` for development, `llm-config.yml` with preferences for submission — covers both worlds cleanly.

## References

- [LLM model configuration (llm_resolver)](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/llm_resolver/README.md) — the two config formats, resolution rules, troubleshooting
- [minima-llm — Configuration](https://github.com/trec-auto-judge/minima-llm#configuration) — full environment-variable table and YAML schema
- [minima-llm — Proxy Mode](https://github.com/trec-auto-judge/minima-llm#proxy-mode) — caching/rate-limiting for any OpenAI-compatible client
