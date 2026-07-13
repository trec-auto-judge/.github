# 5. Prompt Cache

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Run workflows](04-run-workflows.md) · Next: [Meta-evaluation](06-meta-evaluation.md).*

LLM judges re-run constantly during development, and without a cache every re-run repeats every LLM call — slow, expensive, and noisy. A prompt cache stores each response keyed by the exact request, so repeated runs are instant. That same cache then does a second, larger job at submission time: you ship it with your judge, TIRA reproduces your results from it without spending a single LLM call, and — because a judge that reproduces from its cache alone has *proven* it is reproducible — the cache doubles as your proof of correctness, which TIRA validates automatically. This page walks that lifecycle, the one contract your cache must meet, three ways to meet it (minima-llm, the caching proxy, or your client's own disk cache), and the TIRA flags that drive it.

## The lifecycle: seed locally, ship the cache, prove reproducibility

Before the mechanics, the whole idea in four steps — the cache travels with your judge from your laptop into TIRA:

1. **Seed it locally.** As you develop and run your judge against a real LLM endpoint, every call is written under `$CACHE_DIR`. Once you have run over the evaluation topics, that directory holds an answer for every prompt your judge sends.
2. **Ship it with your submission.** `tira-cli` uploads your cache directory alongside your code (through `--mount-cache`, [below](#caching-on-tira)), so the warm cache becomes part of the submission — not merely a local convenience.
3. **TIRA reproduces your results for free.** With the cache warm, TIRA re-runs your judge with the LLM endpoint switched off and serves every call from your cache. Reproducing your run costs the organizers no API calls — which is what makes evaluating many judges across many systems affordable.
4. **The cache is your proof of correctness.** `--cache-behaviour deterministic` makes TIRA re-execute your judge from the cache with the endpoint disabled and check that it reproduces the same output. Passing means your judge is *provably* reproducible from its cache alone — the cache is the evidence, and TIRA checks it for you.

All of this rests on one assumption: **your judge sends byte-identical prompts on every run.** If a prompt changes between runs — reordered inputs, an embedded timestamp, a different model — the replay misses, and with the endpoint disabled the judge has nowhere to turn and fails. Determinism is what turns a caching speed-up into a reproducibility guarantee; the contract below is how you preserve it.

## The contract: one directory, any client

The framework hands your judge a cache location — the `CACHE_DIR` environment variable, arriving as `llm_config.cache_dir` — and expects cached responses to persist under that directory. Setting the environment variable enables caching; leaving it unset disables it:

```bash
export CACHE_DIR="./cache"
```

Any caching mechanism qualifies, as long as it meets four requirements:

1. **The store lives under `$CACHE_DIR`** — that is the directory TIRA mounts and the only path that survives into a submission run.
2. **The backend is disk-based.** Inside TIRA's no-internet sandbox, a Redis, S3, or other hosted cache backend cannot connect — a common trap for litellm configurations that work fine locally.
3. **The key must not include the endpoint URL.** `--cache-behaviour deterministic` makes tira **re-execute your judge with the endpoint disabled** (`OPENAI_BASE_URL=EMPTY`) and the first run's cache — proving cache-only reproducibility. minima-llm's key excludes the endpoint by design; LangChain's default key *includes* `openai_api_base` and must be normalized (the LangChain example judge shows an endpoint-agnostic cache wrapper). Keying on the *model* stays correct — a different LLM is supposed to miss.
4. **Prompts are byte-identical across runs.** Every caching library keys on the exact request (model, messages, temperature, ...), so nondeterministic ordering, timestamps, or dict-iteration randomness silently defeat the cache. Sort responses by `run_id` before building comparison pairs (a core [developing practice](03-develop-an-autojudge.md)) — and expect that changing the model or a sampling parameter invalidates the affected entries, by design.

## Option A — minima-llm's built-in cache

Judges that use minima-llm get caching for free once `cache_dir` is set: an SQLite database at `{cache_dir}/minima_llm.db` (WAL mode, safe for concurrent processes), keyed by a SHA-256 hash over model, messages, temperature, max_tokens, and extras.

Two knobs come with it:

- **Force refresh** — re-ask the LLM while still recording the new answers:
  ```bash
  export CACHE_FORCE_REFRESH=1      # or force_refresh: true in yaml
  ```
- **Miss debugging** — when runs miss unexpectedly, trace the key computation:
  ```bash
  export MINIMA_TRACE_FILE=trace.jsonl
  ```
  Every lookup logs one JSONL line with the key and the canonical request JSON; diff the canonical strings between two runs to see exactly which field changed.

## Option B — any OpenAI-compatible client through the caching proxy

For judges built on litellm, LangChain, the raw `openai` SDK, or plain HTTP, minima-llm's proxy adds the same cache **without code changes**: run `minimallm-proxy` (which reads `CACHE_DIR` and your endpoint config) and point your client's `base_url` at `http://localhost:8990/v1`. Requests flow through the cache exactly as in Option A — including the force-refresh pragma `{"cache": {"no-cache": true}}` on individual requests. Only non-streaming `/v1/chat/completions` is supported.

## Option C — your client's own disk cache

Libraries with native caching work too, provided you point their **disk** backend at the framework's directory. For litellm:

```python
import os, litellm
from litellm.caching.caching import Cache

litellm.cache = Cache(type="disk", disk_cache_dir=os.environ["CACHE_DIR"])
response = litellm.completion(model=..., messages=..., caching=True)
```

(Requires the `litellm[caching]` extra; litellm's `cache={"no-cache": True}` per-request option mirrors force-refresh.) LangChain's cache works the same way — verified in practice by the LangChain example judge:

```python
from langchain_community.cache import SQLiteCache
from langchain_core.globals import set_llm_cache

set_llm_cache(SQLiteCache(database_path=f"{os.environ['CACHE_DIR']}/langchain_cache.db"))
```

DSPy's built-in cache follows the same pattern — configure its disk location under `$CACHE_DIR`. Whatever the library, resist its Redis/S3/semantic backends for submissions (requirement 2 above), and debug misses with that library's own tooling; the *method* from Option A — trace what goes into the key, diff two runs — carries over even where `MINIMA_TRACE_FILE` does not.

## Caching on TIRA

TIRA runs your judge in a sandbox and controls the cache mount through two `tira-cli code-submission` flags — client-agnostic, since they operate on the `$CACHE_DIR` directory itself:

```bash
--cache-behaviour deterministic --mount-cache '$CACHE_DIR=cache'
```

- `--cache-behaviour deterministic` declares that repeated runs with the same cache produce the same output — and tira-cli *verifies* it: after the first local run, your judge is **re-executed with the LLM endpoint disabled** (`OPENAI_BASE_URL=EMPTY`) and the first run's cache, and must reproduce valid output from cache alone. Judges that phone home unconditionally, or whose cache keys include the endpoint URL (requirement 3), fail this second run.
- `--mount-cache '$VARIABLE=DIRECTORY'` mounts a local directory as the environment variable `$VARIABLE` (the container writes to a *copy*, so your local cache is never mutated) — and, crucially for submissions, `tira-cli` **uploads that directory with your code**, so the cache you mount here is what TIRA has on hand when your judge runs. Two right-hand sides:
  - a pre-populated directory (your seeded cache, e.g. `'$CACHE_DIR=cache'`) — **the recommended path for a real submission**: your warm cache ships with the code, and TIRA reproduces your results from it with no LLM calls. Hits require the *same prompts and the same model* the cache was built with, so run and submit with the same `OPENAI_MODEL`.
  - `EMPTY_DIR` — a fresh, empty, writable directory that forces all-fresh LLM calls (a cold start), leaving the populated cache in the run output. Reach for it to prove your judge works from scratch or to regenerate a cache; the first run then spends real LLM calls rather than replaying.

  The variable name must match **what your judge actually reads** — `$CACHE_DIR` is the framework convention, but judges on other backends may use additional or different variables (e.g. a DSPy-based judge honoring `DSPY_CACHEDIR`); repeat `--mount-cache` per variable if needed.

The [submission guide](07-submit-to-tira.md) shows these flags in a complete command.

## References

- [minima-llm — Prompt Caching](https://github.com/trec-auto-judge/minima-llm#prompt-caching) — enable/disable, force refresh, debug tracing
- [minima-llm — Proxy Mode](https://github.com/trec-auto-judge/minima-llm#proxy-mode) — the caching proxy for any OpenAI-compatible client
- [minima-llm — Configuration](https://github.com/trec-auto-judge/minima-llm#configuration) — the full environment-variable table (`CACHE_DIR`, `CACHE_FORCE_REFRESH`, `MINIMA_TRACE_FILE`, rate limits, ...)
- [litellm — Caching](https://docs.litellm.ai/docs/caching/all_caches) — litellm's cache backends, including the disk cache used in Option C
