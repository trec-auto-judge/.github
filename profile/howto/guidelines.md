# Guidelines for submitting an AutoJudge

The [Participant HowTo](README.md) covers the full process in seven pages (setup, LLM endpoint, developing, running, prompt cache, meta-evaluation, submission). If you use Claude Code, the starter kit ships the skills `/autojudge-setup`, `/autojudge-develop`, and `/autojudge-submit`, which walk through the same steps. The notes below summarize the points that commonly cause problems.

## Setup

- Please clone the [starter kit](https://github.com/trec-auto-judge/auto-judge-starter-kit) into a new repository of your own (rather than a GitHub fork) and keep the original as a `starterkit` remote; see the [setup page](01-setup-environment.md) for the commands.
- You must make the template your own: change the project name in `pyproject.toml`, replace the README, and put your judge in its own directory under `judges/`. The test suite checks this once your `origin` remote is set.
- `pytest` must pass before you submit. The tests discover your judges automatically (workflow parses, classes import), compare installed framework versions against the template, and are run again at submission time.

## LLM access and caching

- Read the endpoint from the injected `llm_config` parameter and do not hardcode endpoints or keys. Any OpenAI-compatible client works (minima-llm, DSPy, LangChain, litellm, plain SDK). On TIRA, we provide the endpoint and may run your judge with several models.
- Use a prompt cache under `$CACHE_DIR`, with a disk-based backend, and make sure the cache key does not include the endpoint URL: the submission check re-executes your judge with the endpoint disabled and expects identical output from cache alone. A judge that makes an unconditional call at startup, or whose cache keys on `base_url` (the LangChain default), fails this check. See the [prompt cache page](05-prompt-cache.md) for per-client instructions.

## Submission

See the [submission page](07-submit-to-tira.md) for the full procedure.

- The repository must be clean (`git status --porcelain` prints nothing), on `main`, with everything committed — only committed state is submitted. Delete the example judges you did not write. Submit from a shell without unrelated secrets exported.
- In the Dockerfile, install into the base image's venv: `RUN . /venv/bin/activate && uv pip install -e .[all]`. An install with `--system` is not visible at runtime.
- The local test runs without network. Add `--allow-network` when testing against an external endpoint, or mount your populated cache (`--mount-cache '$CACHE_DIR=cache'`), which requires no network. All `auto-judge run` options, including `--variant`, belong inside the quoted `--command`.

If you run into problems, please use the private TIRA chat that we opened with your team at registration.
