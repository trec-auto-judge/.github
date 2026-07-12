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
| `raw` | extra config dict — empty under environment-only configuration; retained so the `MinimaLlmConfig.from_dict(llm_config.raw)` pattern keeps working |

## Choose your LLM client — any OpenAI-compatible client works

The framework hands you endpoint coordinates, not a client, so you pick the library — use whatever you are comfortable with; the three below are worked examples, not a ranked list or an endorsement. Whichever you choose, the TIRA-compatible pattern is the same: **construct the client explicitly from the injected `llm_config`** rather than relying on the library's own environment lookup — the libraries disagree on variable names (the openai SDK reads `OPENAI_BASE_URL`, litellm's convention is `OPENAI_API_BASE`), and explicit construction makes the routing independent of those conventions. The starter kit's `tests/test_endpoint_contract.py` verifies the routing either way, and a `--dry-run` submission with `--cache-behaviour deterministic` additionally proves your client's cache survives an endpoint swap.

Whichever client you pick, judges make many similar LLM calls — one per response, per pair, or per nugget×response — so build the full list of requests first and hand it to a batched runner rather than looping over blocking calls; each library below has its own way to do this.

### LangChain

Construct `ChatOpenAI` explicitly from the injected config — LangChain has no endpoint management of its own (an argument-less client would silently fall back to the *openai SDK's* environment lookup, which works for `OPENAI_BASE_URL` but keeps the precedence implicit):

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model=llm_config.model, base_url=llm_config.base_url,
                 api_key=llm_config.api_key or "-", temperature=0.0)
```

Environment notes: two TIRA-specific pitfalls come with LangChain. Its cache key includes `openai_api_base`, so a plain `SQLiteCache` breaks deterministic re-execution (the endpoint gets swapped, every lookup misses) — normalize the endpoint out of the key, as the [LangChain example judge](https://github.com/laura-dietz/langchain-starterkit) does with its endpoint-agnostic cache wrapper. And extra request parameters need their own environment route (the example judge reads `OPENAI_EXTRA_BODY` as JSON), since `llm_config.raw` is empty under environment-only configuration.

### litellm

Pass the endpoint per call (or via a router), again explicitly from the injected config; for an OpenAI-compatible endpoint the model name takes litellm's `openai/` provider prefix:

```python
import litellm

response = litellm.completion(model=f"openai/{llm_config.model}",
                              api_base=llm_config.base_url,
                              api_key=llm_config.api_key,
                              messages=[...], temperature=0.0)
```

### minima-llm

A batteries-included option that folds prompt caching, rate limiting, retry, and batching into one runner. Build the full list of requests and hand it to `run_batched`:

```python
from minima_llm import MinimaLlmConfig, MinimaLlmRequest, OpenAIMinimaLlm

full_config = MinimaLlmConfig.from_dict(llm_config.raw)          # falls back to env when raw is empty
backend = OpenAIMinimaLlm(full_config)
responses = backend.run_batched([MinimaLlmRequest(...), ...])    # parallelizes, rate-limits, retries, caches
```

**DSPy** wires in through minima-llm's adapter: declare a `dspy.Signature` per judgment type — with explicit `reasoning` and `confidence` output fields — and run `run_dspy_batch(signature, items, converter, backend=...)`, which binds the backend as a `MinimaLlmDSPyLM` via `dspy.context` and yields structured outputs with the same batching and caching underneath (the [prefnugget-starterkit](https://github.com/laura-dietz/prefnugget-starterkit) judges work this way).

Environment notes: minima-llm natively reads the exact task-provided names (`OPENAI_BASE_URL`, `OPENAI_MODEL`, `OPENAI_API_KEY`, `CACHE_DIR`), so no rerouting is needed, and its cache key deliberately excludes the endpoint URL — compatible with TIRA's deterministic re-execution out of the box. See [minima-llm — Quick Start & DSPy](https://github.com/trec-auto-judge/minima-llm#quick-start).

Environment notes: litellm's own environment convention is `OPENAI_API_BASE`, *not* the task-provided `OPENAI_BASE_URL` — relying on litellm's env pickup would miss the injected endpoint, which is exactly why explicit `api_base=` is the safe pattern (the framework's `from_env` tolerates both names, but your client code should not depend on that). For caching, use litellm's **disk** backend pointed at the framework's directory (`litellm.cache = Cache(type="disk", disk_cache_dir=os.environ["CACHE_DIR"])`, per the [prompt cache page](05-prompt-cache.md)) — hosted backends (Redis/S3) cannot connect inside the sandbox — and verify with a deterministic dry run that its cache keys survive the endpoint swap.

The raw `openai` SDK and plain HTTP follow the same explicit-construction rule; minima-llm's [proxy mode](https://github.com/trec-auto-judge/minima-llm#proxy-mode) can add contract-compatible caching to any such client without code changes. Whatever you choose, never bake an endpoint or key into your code — a hardcoded value would break under TIRA, where the organizer decides which model your judge gets.

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
