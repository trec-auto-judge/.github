# 7. Submit to TIRA

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Meta-evaluation](06-meta-evaluation.md).*

Submitting your auto-judge means making a **code submission**: `tira-cli` builds your repository's Dockerfile into an image, tests that image locally on the kiddie dataset, and — only if the outputs validate — uploads it to TIRA, where we run it on all datasets, potentially with multiple LLMs. Complete the [prerequisites](README.md#prerequisites) (TIRA account, team registration) before starting here.

If you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit ships an interactive walkthrough of this page: type `/autojudge-submit`.

## Three ways to submit

There are three submission paths with different mechanics — most participants use the first:

1. **Code submission** (Steps 1–4 below) — `tira-cli code-submission` ships your judge's Docker image and the organizers run it on all datasets. The reproducible, preferred path.
2. **Data submission** — you run your judge locally and upload the *outputs* (leaderboards) with `tira-cli upload`. Useful when you cannot ship runnable code, or to lodge results quickly. See [Uploading run outputs](#uploading-run-outputs).
3. **Meta-evaluation service** — deposit your `*.eval.txt` into a meta-evaluation service (rsync, per track), which correlates it against held truth. Some host institutions run their own; TIRA data submission is usually preferred.

## Step 1 — Meet the submission requirements

- **Your judge runs locally, end to end.** With your LLM environment loaded (`OPENAI_BASE_URL`/`OPENAI_MODEL`/`OPENAI_API_KEY`, plus `CACHE_DIR` for caching judges), `bash run_kiddie.sh` completes without errors — fixing problems locally beats debugging them through a Docker build.
- **All code lives in a git repository** (private or even local-only is fine — we never check that changes are pushed).
- **The repository is clean.** `git status --porcelain` must print nothing; any output means uncommitted or untracked changes. Commit your code; add build artifacts, caches, and output directories to `.gitignore`.
- **The currently checked-out branch is what gets submitted**, and only its *committed* state — uncommitted edits are silently excluded. We recommend submitting from `main`; whatever `git branch --show-current` prints is what runs.
- **`pytest` completes without failures** — `tira-cli` runs the test suite as part of the submission, so red tests block it. Beyond your own judge tests, the starter-kit suite checks that every judge's workflow parses and its classes import (minimum compatibility), that the installed framework versions (`autojudge-base`, `tira`) are up to date with the template's requirements, and that the template was customized (project name and README changed).
- **Example judges you did not write are deleted** so they do not ship with your submission.
- **The template is customized**: `pyproject.toml` no longer says `name = "auto-judge-starterkit"`, and the README describes *your* judge — an unrenamed template reads as an unconfigured submission.
- **A Dockerfile at the repo root** specifies how your software is dockerized. Making it [dev-container](https://containers.dev/) compatible lets you develop directly inside the container.
- **The sandbox has no internet access.** Your judge receives its LLM endpoint through forwarded environment variables — see [Configure your LLM endpoint](02-configure-llm-endpoint.md). Nothing else on the network will be reachable.
- **No secrets in the image.** Never `COPY`/`ADD` API keys into the Dockerfile; pass them only via `--forward-environment-variable` so they are injected at run time.
- **Submit from a clean shell.** Current `tira-cli` versions record the submitting shell's environment as build metadata, so unset (or never export) secrets unrelated to the submission before running `code-submission`.

## Step 2 — Install tira-cli and start Docker

Inside your activated venv (the starter kit's `.[all]` extra may already provide it):

```bash
uv pip install --upgrade tira
```

(`pip3 install --upgrade tira` works equally outside a venv.)

The Docker (or podman) daemon must be **running** when you submit — the build-and-test happens on your machine before anything is uploaded. Checking early saves a late surprise:

```bash
tira-cli verify-installation
```

**Podman users:** if the build fails at the first `FROM` step with `no policy.json file found`, create the missing signature-policy file (podman-only; Docker never needs it):

```bash
mkdir -p ~/.config/containers
printf '{\n  "default": [{"type": "insecureAcceptAnything"}]\n}\n' > ~/.config/containers/policy.json
```

## Step 3 — Authenticate

Fetch your authentication token from TIRA: navigate to the [TREC AutoJudge task](https://www.tira.io/task-overview/trec-auto-judge), then *submit → Code Submission → New Submission → "I want to submit from my local machine" → Next → Next*:

<img width="1808" height="985" alt="TIRA UI showing the authentication token" src="https://github.com/user-attachments/assets/995cbd0e-1eae-4a70-a13b-acbf1d2229dc" />

```bash
tira-cli login --token AUTH-TOKEN
tira-cli verify-installation --task trec-auto-judge --team YOUR-TEAM
```

Scoping the verification to your task and team confirms not just the installation but also that your login and registration line up. A healthy verification looks like:

<img width="821" height="180" alt="tira-cli verify-installation output" src="https://github.com/user-attachments/assets/51160132-eb19-4da3-8892-8a53adb41c71" />

## Step 4 — Dry-run, then submit

The dry run builds the image and tests it locally on kiddie without uploading anything:

```bash
export OPENAI_API_KEY=...  OPENAI_BASE_URL=...  OPENAI_MODEL=...  CACHE_DIR=./cache

# Seed ./cache by running the SAME workflow/variant/model you submit below,
# so the mounted cache replays instead of calling the LLM (see Prompt cache):
auto-judge run --workflow judges/myjudge/workflow.yml --variant best \
    --rag-responses data/kiddie/runs/repgen/ --rag-topics data/kiddie/topics/kiddie-topics.jsonl \
    --out-dir ./output-kiddie/

tira-cli code-submission \
    --dry-run \
    --path . \
    --cache-behaviour deterministic \
    --mount-cache '$CACHE_DIR=cache' \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --task trec-auto-judge \
    --dataset kiddie-20260605-training \
    --command 'auto-judge run --workflow /auto-judge/judges/myjudge/workflow.yml --variant best --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'
```

Three details that trip people up:

- **Everything your judge needs goes inside the quoted `--command`.** There is no `tira-cli --variant` flag — `--variant`, and any other `auto-judge run` option, belongs inside the command string. `$inputDataset` and `$outputDir` are substituted by TIRA.
- **The cache flags** (`--cache-behaviour deterministic`, `--mount-cache '$CACHE_DIR=cache'`) apply to LLM judges that cache — [Prompt cache](05-prompt-cache.md) explains the full lifecycle. Judges without an LLM can omit them. Mount your *seeded* cache (`'$CACHE_DIR=cache'`): `tira-cli` uploads it with your code, so TIRA reproduces your results from cache with no LLM calls, and the local dry run replays in seconds instead of LLM-minutes. Seed it first by running the same workflow, variant, and `OPENAI_MODEL` you submit — mismatched prompts miss. (`EMPTY_DIR` instead of `cache` forces a cold start with fresh LLM calls; use it only to regenerate a cache. The mount variable must match what your judge reads — `CACHE_DIR` by convention, backends may differ.)
- **One submission covers one judge/variant.** Submit multiple variants by repeating the command with a different `--command` string.

When the dry run passes, remove `--dry-run` and run the same command to upload.

### Troubleshooting the dry run

- **`No module named '...'` inside the container, although it installs fine locally** — the `trec-auto-judge-base` image runs Python from `/venv` (`PATH=/venv/bin`), so dependencies must be installed *into that venv*: the Dockerfile needs `RUN . /venv/bin/activate && uv pip install -e .[all]`, **not** `uv pip install --system ...` (a system install is invisible at runtime). Bites exactly when your judge adds dependencies beyond the template's.
- **`Connection error` from your LLM client inside the container** — tira's local test runs **without network by default**, mirroring the TIRA sandbox. Testing against an external endpoint (OpenRouter, hosted OpenAI, ...) needs `--allow-network` on the `code-submission` command; alternatively, a mounted warm cache lets the judge complete with no network at all. Inside real TIRA the organizer-provided endpoint is cluster-internal, so this only concerns your local test.
- **`The cache directory mounted via CACHE_DIR was not used during the execution`** — tira-cli verifies that the mounted cache actually gets touched. Most often this is a *symptom*: the judge crashed before its first LLM call (check the error above it in the log). If the judge genuinely ran, it is not honoring `$CACHE_DIR` — see [Prompt cache](05-prompt-cache.md).

If anything fails — or you cannot run Docker locally at all — reach out in the private TIRA chat that we opened with your team at registration ([prerequisites](README.md#prerequisites)), and we will find a way to get your submission in.

## A complete session

Condensed from a real, successful submission (the [prefnugget-starterkit](https://github.com/laura-dietz/prefnugget-starterkit), `queryonly` judge), with the cache mount shown as the recommended warm-cache flow from [Prompt cache](05-prompt-cache.md):

```bash
git clone git@github.com:YOUR-USER/YOUR-JUDGE.git && cd YOUR-JUDGE
uv venv && source .venv/bin/activate
uv pip install -e '.[all]'

export OPENAI_API_KEY=... OPENAI_BASE_URL=... OPENAI_MODEL=... CACHE_DIR=./cache
bash run_kiddie.sh                    # local end-to-end test — also seeds ./cache for the judge it runs
                                      # (seed with the SAME judge/variant/model you submit, or the mount misses)

git status --porcelain                # must be empty: gitignore build artifacts, commit the rest
git branch --show-current             # recommended: main

uv pip install --upgrade tira
tira-cli login --token XXXXX
tira-cli verify-installation --task trec-auto-judge --team YOUR-TEAM

tira-cli code-submission --dry-run --path . \
    --cache-behaviour deterministic --mount-cache '$CACHE_DIR=cache' \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --task trec-auto-judge --dataset kiddie-20260605-training \
    --command 'auto-judge run --workflow /auto-judge/judges/queryonly/workflow.yml --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'

# dry run green? same command without --dry-run:
tira-cli code-submission --path . \
    --cache-behaviour deterministic --mount-cache '$CACHE_DIR=cache' \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --task trec-auto-judge --dataset kiddie-20260605-training \
    --command 'auto-judge run --workflow /auto-judge/judges/queryonly/workflow.yml --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'
```

## Uploading run outputs

Instead of (or alongside) a code submission, run your judge locally and upload the `.eval.txt` leaderboards it produces. Both destinations consume the same `ir_measures` leaderboard, and [`run_all_datasets.py`](04-run-workflows.md#running-against-multiple-datasets) does the run and the upload in one step (`--dry-run` prints the exact commands first):

```bash
# 1. fetch the dataset (once) — credentials are in setup step 5
./fetch_pilot_dataset.sh --dataset dragun-repgen

# 2. run your judge and upload its leaderboard to TIRA:
python run_all_datasets.py --workflow judges/myjudge/workflow.yml --dataset dragun-repgen --upload-tira

#    add the meta-evaluation-service deposit alongside --upload-tira if you want it:
#    --upload-metaeval --metaeval-dest c02:/autojudge-eval/in
```

The [datasets are fetched](01-setup-environment.md#step-5--fetch-the-evaluation-datasets) into `./local-data/`, and each dataset's `tira_id` (upload target) and `bucket` (meta-eval track) live in `datasets.yml`.

By hand, the two interactions are:

- **TIRA data upload** — `tira-cli upload --dataset <tira_id> --directory <out-dir>` zips the output directory and validates the leaderboard against the dataset's format (`trec-eval-leaderboard`); add `--dry-run` to validate without uploading, and `--system NAME` to label the run.
- **Meta-evaluation service** — `rsync -Laur <out-dir>/*.eval.txt <dest>/<bucket>/`; the operator's watcher correlates each `*.eval.txt` against held truth.

## References

- [Configure your LLM endpoint](02-configure-llm-endpoint.md) — how the endpoint reaches your judge inside the sandbox
- [Prompt cache](05-prompt-cache.md) — what the TIRA cache flags do
- [TIRA participant documentation](https://docs.tira.io/participants/participate.html) — general TIRA submission background
