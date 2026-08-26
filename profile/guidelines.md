# Guidelines for submitting an AutoJudge

The [Participant HowTo](howto/README.md) covers the full process in seven pages (setup, LLM endpoint, developing, running, prompt cache, meta-evaluation, submission). If you use Claude Code, the starter kit ships the skills `/autojudge-setup`, `/autojudge-develop`, and `/autojudge-submit`, which walk through the same steps. The notes below summarize the points that commonly cause problems.

## Evaluation data

- **Before touching evaluation data, read the [data-handling policy](howto/data-policy.md).** The datasets are anonymized and mostly off-limits to your *coding agent*, while the *judge* you build may read everything — the policy says which is which, and how to debug a failure without looking. Each dataset also ships it as `AGENTS.md` and `CLAUDE.md`, but that copy is only found once the data is already open.
- Do not work out which participant produced which run, by any method — including comparison against your own submissions, hashing, or matching on writing style. Judges are built before the truth data exists and the whole dataset is the test set, so a single identified run or a threshold tuned on a restricted topic undermines the evaluation for everyone, irreversibly.

## Setup

- Please clone the [starter kit](https://github.com/trec-auto-judge/auto-judge-starter-kit) into a new repository of your own (rather than a GitHub fork) and keep the original as a `starterkit` remote; see the [setup page](howto/01-setup-environment.md) for the commands.
- You must make the template your own: change the project name in `pyproject.toml`, replace the README, and put your judge in its own directory under `judges/`. The test suite checks this once your `origin` remote is set.
- `pytest` must pass before you submit. The tests discover your judges automatically (workflow parses, classes import), compare installed framework versions against the template, and are run again at submission time.

## LLM access and caching

- Configure the endpoint through environment variables (`OPENAI_BASE_URL`, `OPENAI_MODEL`, `OPENAI_API_KEY`, `CACHE_DIR` — see the [endpoint page](howto/02-configure-llm-endpoint.md)). Read them either from the injected `llm_config` parameter, which is recommended, or directly from the environment — `llm_config` is just the parsed view of the same variables. Do not hardcode endpoints or keys, and do not leave the lookup to your LLM library, whose variable names differ. Any OpenAI-compatible client works (minima-llm, DSPy, LangChain, litellm, plain SDK). On TIRA, we provide the endpoint and may run your judge with several models.
- Use a prompt cache under `$CACHE_DIR`, with a disk-based backend, and make sure the cache key does not include the endpoint URL: the submission check re-executes your judge with the endpoint disabled and expects identical output from cache alone. A judge that makes an unconditional call at startup, or whose cache keys on `base_url` (the LangChain default), fails this check. See the [prompt cache page](howto/05-prompt-cache.md) for per-client instructions.

## TREC 2026 AutoJudge Test

The test data is released — see the [participation checklist](howto/trec-2026-autojudge-test.md); **submission deadline: September 30, 2026**. In short: fetch the two test datasets (`./fetch_pilot_dataset.sh --dataset rag26` / `ragtime26`), run your judge and upload the leaderboards (`python run_all_datasets.py --workflow judges/<your-judge>/workflow.yml --meta-evaluate --dataset rag26-generation --upload-tira`, likewise for `ragtime26-repgen`), and ship the matching code submission. Local meta-evaluation on the test data correlates against a placeholder truth — a pipeline check, not a quality signal.

## Submission

See the [submission page](howto/07-submit-to-tira.md) for the full procedure.

- The repository must be clean (`git status --porcelain` prints nothing), on `main`, with everything committed — only committed state is submitted. Delete the example judges you did not write. Submit from a shell without unrelated secrets exported.
- In the Dockerfile, install into the base image's venv: `RUN . /venv/bin/activate && uv pip install -e .[all]`. An install with `--system` is not visible at runtime.
- The local test runs without network. Add `--allow-network` when testing against an external endpoint, or mount your populated cache (`--mount-cache '$CACHE_DIR=cache'`), which requires no network. All `auto-judge run` options, including `--variant`, belong inside the quoted `--command`.

If you run into problems, please use the private TIRA chat that we opened with your team at registration.
