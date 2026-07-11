# 5. Prompt Cache

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Run workflows](run-workflows.md) · Next: [Meta-evaluation](meta-evaluation.md).*

LLM judges re-run constantly during development, and without a cache every re-run repeats every LLM call — slow, expensive, and noisy. Judges built on minima-llm get an SQLite-backed prompt cache that makes repeated runs instant and deterministic; this page explains how to switch it on, how it decides what counts as "the same prompt", how to debug misses, and how caching interacts with TIRA submissions.

## Turning it on

Setting a cache directory enables the cache; leaving it unset disables it:

```bash
export CACHE_DIR="./cache"          # env var, or
```

```yaml
cache_dir: "./cache"                # in your llm-config yaml
```

The database lands at `{cache_dir}/minima_llm.db` (SQLite in WAL mode, safe for concurrent processes).

## What makes a cache hit

Each response is stored under a SHA-256 hash of the request parameters — model, messages, temperature, max_tokens, and extras — so a hit requires the prompt to be *byte-identical*. Two practical consequences:

- **Keep prompt construction deterministic.** Sort responses by `run_id` before building comparison pairs (a core [developing practice](developing-practices.md)); any nondeterministic ordering, timestamps, or dict-iteration randomness silently changes the hash and defeats the cache.
- **Model or parameter changes invalidate everything** for those requests — expected behavior, not a bug.

## Refreshing and debugging

To re-ask the LLM while still recording the new answers, bypass cache reads with:

```bash
export CACHE_FORCE_REFRESH=1        # or force_refresh: true in yaml
```

When runs miss the cache unexpectedly, trace the key computation:

```bash
export MINIMA_TRACE_FILE=trace.jsonl
```

Every cache lookup then logs one JSONL line with the key and the canonical request JSON — diff the canonical strings between two runs to see exactly which field changed.

## Caching on TIRA

TIRA runs your judge in a sandbox and controls the cache mount through two `tira-cli code-submission` flags:

```bash
--cache-behaviour deterministic --mount-cache '$CACHE_DIR=EMPTY_DIR'
```

- `--cache-behaviour deterministic` declares that repeated runs with the same cache produce the same output — information TIRA uses for reproducibility.
- `--mount-cache '$CACHE_DIR=EMPTY_DIR'` mounts a fresh, empty, writable directory as your cache, forcing all-fresh LLM calls; after the run, the output contains the populated cache for potential reuse. Mounting a pre-populated directory instead (`--mount-cache "CACHE_DIR=my-cache-dir"`) replays a bundled cache — handy for reproducing published results without an LLM endpoint.

The [submission guide](submit-to-tira.md) shows these flags in a complete command.

## References

- [minima-llm — Prompt Caching](https://github.com/trec-auto-judge/minima-llm#prompt-caching) — enable/disable, force refresh, debug tracing
- [minima-llm — Configuration](https://github.com/trec-auto-judge/minima-llm#configuration) — the full environment-variable table (`CACHE_DIR`, `CACHE_FORCE_REFRESH`, `MINIMA_TRACE_FILE`, rate limits, ...)
