# 2. Configure Your LLM Endpoint

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Set up your dev environment](01-setup-environment.md) · Next: [Develop an AutoJudge](03-develop-an-autojudge.md).*

Every LLM-based judge receives its endpoint, model, and API key through a single injected object — the `llm_config` parameter passed to your judge methods — and must read them from there rather than hardcode them. That indirection is what allows the same judge code to run against your local endpoint during development and against the organizer-provided endpoint inside TIRA's sandbox, unchanged. In [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit's `/autojudge-setup` skill walks you through this page together with the environment setup.

## What your judge receives

All three protocol methods (`judge`, `create_nuggets`, `create_qrels`) get an `llm_config: LlmConfigBase` carrying:

| Field | Content |
|-------|---------|
| `base_url` | endpoint URL (OpenAI-compatible) |
| `model` | resolved model identifier |
| `api_key` | key/token (may be empty for local endpoints) |
| `cache_dir` | prompt-cache directory, if enabled (see [Prompt cache](05-prompt-cache.md)) |
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

## Local development

The endpoint is configured through **environment variables** — the only injection path:

```bash
export OPENAI_BASE_URL="http://localhost:8000/v1"   # any OpenAI-compatible endpoint: vLLM, Ollama, hosted OpenAI, ...
export OPENAI_MODEL="llama-3.3-70b-instruct"
export OPENAI_API_KEY="sk-..."                       # optional for unsecured local endpoints
export CACHE_DIR="./cache"                           # optional, enables prompt caching
```


## On TIRA: how the endpoint reaches your judge

Inside TIRA your judge runs in a sandbox without internet access, so the endpoint must be injected.

### The endpoint contract

Your judge **must use the task-provided environment variables as-is** — `OPENAI_BASE_URL`, `OPENAI_MODEL`, `OPENAI_API_KEY` — and route their values into whichever LLM client it uses. Every client has a way to set these explicitly (litellm: `api_base`/`api_key` arguments; LangChain: `base_url`/`api_key`; the openai SDK likewise). A judge with a hardcoded endpoint or model **will not run on TIRA**: the sandbox has no route to your provider, and we choose which model your judge gets — the failure only surfaces after the code is submitted, in a remote run you cannot debug.

To catch this before submission, the starter kit's `tests/test_endpoint_contract.py` runs each judge against a local pretend endpoint and asserts that the judge contacts it, with the injected model. Judges that make no LLM calls at all declare `uses_llm: false` in their own `workflow.yml`, which marks their case as expected-to-fail (`xfail`) — do not delete the test.

The injection mechanism: **forwarded environment variables.** `--forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL` registers the variable **names** as your judge's runtime requirements. Values are supplied per run by whoever runs it: your shell for the local test, and the organizers — with their cluster-internal endpoint and their choice of model — for runs on TIRA. Your judge cannot pin a specific external endpoint into remote runs, by design. The [submission guide](07-submit-to-tira.md) shows this in context.

Model choice on TIRA rests with the organizers through the injected `OPENAI_MODEL`. To evaluate your judge with several models locally, loop over env injections (`OPENAI_MODEL=a auto-judge run ...; OPENAI_MODEL=b auto-judge run ...`).

> **Historical note:** earlier versions supported an `llm-config.yml` (`--llm-config`, `--submission`, `model_preferences`). This mechanism has been removed — if your repo still carries such a file or commands, delete the file and rely on the environment variables.

## Troubleshooting

- **`404 No endpoints found for <model>`** (or similar) usually means a stale model identifier — hosted providers rename and retire models. List what your endpoint actually serves and pick a current id:
  ```bash
  curl -s "$OPENAI_BASE_URL/models" -H "Authorization: Bearer $OPENAI_API_KEY" | head
  ```
- **Overriding a shared environment script.** A later `export OPENAI_MODEL=...` overrides whatever a sourced team script set — no need to edit the shared script.
- **Silent empty results.** When every LLM call fails (bad key, dead endpoint, stale model), judges tend to produce empty output, which the framework's verification then rejects (e.g. `Empty nugget banks for N topic(s)`). Treat that error as "check the endpoint first", not as a bug in your judge logic.

## References

- [minima-llm — Configuration](https://github.com/trec-auto-judge/minima-llm#configuration) — full environment-variable table and YAML schema
- [minima-llm — Proxy Mode](https://github.com/trec-auto-judge/minima-llm#proxy-mode) — caching/rate-limiting for any OpenAI-compatible client
