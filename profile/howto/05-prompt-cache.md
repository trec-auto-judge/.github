# 5. Prompt Cache

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Run workflows](04-run-workflows.md) · Next: [Meta-evaluation](06-meta-evaluation.md).*

LLM judges re-run constantly during development, and without a cache every re-run repeats every LLM call — slow, expensive, and noisy. A prompt cache stores each response keyed by the exact request, making repeated runs instant and deterministic. Caching works with **any LLM client**: the framework only defines where the cache lives, and this page covers that contract, three ways to satisfy it (minima-llm, the caching proxy, or your client's own disk cache), and how caching interacts with TIRA.

## The contract: one directory, any client

The framework hands your judge a cache location — the `CACHE_DIR` environment variable, arriving as `llm_config.cache_dir` — and expects cached responses to persist under that directory. Setting it enables caching; leaving it unset disables it:

```bash
export CACHE_DIR="./cache"          # env var, or
```

```yaml
cache_dir: "./cache"                # in your llm-config yaml
```

Any caching mechanism qualifies, as long as it meets three requirements:

1. **The store lives under `$CACHE_DIR`** — that is the directory TIRA mounts and the only path that survives into a submission run.
2. **The backend is disk-based.** Inside TIRA's no-internet sandbox, a Redis, S3, or other hosted cache backend cannot connect — a common trap for litellm configurations that work fine locally.
3. **Prompts are byte-identical across runs.** Every caching library keys on the exact request (model, messages, temperature, ...), so nondeterministic ordering, timestamps, or dict-iteration randomness silently defeat the cache. Sort responses by `run_id` before building comparison pairs (a core [developing practice](03-develop-an-autojudge.md)) — and expect that changing the model or a sampling parameter invalidates the affected entries, by design.

## Option A — minima-llm's built-in cache

Judges using minima-llm (the starter-kit default) get caching for free once `cache_dir` is set: an SQLite database at `{cache_dir}/minima_llm.db` (WAL mode, safe for concurrent processes), keyed by a SHA-256 hash over model, messages, temperature, max_tokens, and extras.

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
--cache-behaviour deterministic --mount-cache '$CACHE_DIR=EMPTY_DIR'
```

- `--cache-behaviour deterministic` declares that repeated runs with the same cache produce the same output — information TIRA uses for reproducibility.
- `--mount-cache '$CACHE_DIR=EMPTY_DIR'` mounts a fresh, empty, writable directory as your cache, forcing all-fresh LLM calls; after the run, the output contains the populated cache for potential reuse. Mounting a pre-populated directory instead (`--mount-cache "CACHE_DIR=my-cache-dir"`) replays a bundled cache — handy for reproducing published results without an LLM endpoint.

The [submission guide](07-submit-to-tira.md) shows these flags in a complete command.

## References

- [minima-llm — Prompt Caching](https://github.com/trec-auto-judge/minima-llm#prompt-caching) — enable/disable, force refresh, debug tracing
- [minima-llm — Proxy Mode](https://github.com/trec-auto-judge/minima-llm#proxy-mode) — the caching proxy for any OpenAI-compatible client
- [minima-llm — Configuration](https://github.com/trec-auto-judge/minima-llm#configuration) — the full environment-variable table (`CACHE_DIR`, `CACHE_FORCE_REFRESH`, `MINIMA_TRACE_FILE`, rate limits, ...)
- [litellm — Caching](https://docs.litellm.ai/docs/caching/all_caches) — litellm's cache backends, including the disk cache used in Option C
