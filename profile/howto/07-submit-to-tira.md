# 7. Submit to TIRA

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Meta-evaluation](06-meta-evaluation.md).*

Submitting your auto-judge has two parts, and we ask you to do **both**: a **data submission** — you run your judge on the released datasets and upload the leaderboards — and a **code submission**, where `tira-cli` builds your repository's Dockerfile into an image, tests that image locally on the kiddie dataset, and — only if the outputs validate — uploads it to TIRA, where we run it on all datasets, potentially with multiple LLMs. Complete the [prerequisites](README.md#prerequisites) (TIRA account, team registration) before starting here.

If you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit ships an interactive walkthrough of this page: type `/autojudge-submit`.

> **Submitting for the TREC 2026 AutoJudge test?** This page explains the mechanics for any dataset; the [TREC 2026 AutoJudge Test page](trec-2026-autojudge-test.md) is the checklist for that campaign — which datasets, which commands.

## Three ways to submit

There are three submission paths with different mechanics — most participants use the first:

1. **Code submission** — `tira-cli code-submission` builds a Docker image for your judge and submits it via TIRA. Organizers will run it on all datasets for reproducibility. The preferred path.
2. **Data submission** — you run your judge locally and upload the *outputs* (leaderboards) with `tira-cli upload`. Useful when you cannot ship runnable code, or to lodge results quickly.
3. **Meta-evaluation service** (optional) — deposit your `*.eval.txt` into a meta-evaluation service (rsync, per track), which correlates it against held truth. Some host institutions run their own; TIRA data submission is usually preferred.

> **Please make both a data submission and a code submission** — the data submission lodges your results now, the code submission lets us reproduce and re-run them.

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

- **The evaluation datasets are fetched.** You will run your judge on them before submitting — see [setup step 5](01-setup-environment.md#step-5--fetch-the-evaluation-datasets).

## Step 2 — Install tira-cli and start Docker

Inside your activated venv (the starter kit's `.[all]` extra may already provide it):

```bash
uv pip install --upgrade tira
```

(`pip3 install --upgrade tira` works equally outside a venv.)

For a code submission, Docker or podman must be able to **build and run containers** when you submit — the build-and-test happens on your machine before anything is uploaded. The starter kit ships a read-only preflight that diagnoses the common container-runtime problems and prints the exact fix for each:

```bash
./check_container_setup.sh     # add --fix to also apply the one safe self-repair (podman system migrate)
tira-cli verify-installation
```

At this stage only the container-side ✓s matter — `verify-installation` reports "not valid" until it can also check authentication and image upload, which needs the login and `--task`/`--team` scoping from step 3.

(Not sure which engine you have? `docker version` — the first line says `Podman Engine` or `Docker Engine`; many distributions ship `docker` as a podman compatibility shim.)

**Podman users:** podman works fully rootless ("headless") — no root daemon required. Three things need to be in place, all covered by the preflight above:

- **A docker-compatible endpoint.** Start the user-level API socket and make sure tira-cli finds it — either through your distribution's docker-compat shim (`docker` resolving to podman, e.g. the `podman-docker` package) or via the `DOCKER_HOST` variable:

  ```bash
  systemctl --user enable --now podman.socket
  export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock   # only if `docker` is not podman
  ```

- **Subordinate user IDs.** If pulling the base image fails with *"potentially insufficient UIDs or GIDs available in user namespace"*, your user lacks subordinate ID ranges. Grant them and migrate:

  ```bash
  sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
  podman system migrate
  ```

  If the ranges exist but pulls still fail, run `podman system migrate` once more: rootless podman keeps its namespace alive in a per-session pause process, and a stale one freezes an old (collapsed) mapping even after the configuration is fixed. `./check_container_setup.sh` detects an unusable mapping either way — and `./check_container_setup.sh --fix` applies the migrate itself when (and only when) that check fails.

- **A signature policy.** If the build fails at the first `FROM` step with `no policy.json file found` (podman-only; Docker never needs it):

  ```bash
  mkdir -p ~/.config/containers
  printf '{\n  "default": [{"type": "insecureAcceptAnything"}]\n}\n' > ~/.config/containers/policy.json
  ```

## Step 3 — Authenticate

Fetch your authentication token from TIRA: navigate to the [TREC AutoJudge task](https://www.tira.io/task-overview/trec-auto-judge), then *submit → Code Submission → New Submission → "I want to submit from my local machine" → Next → Next*:

<img width="1808" height="985" alt="TIRA UI showing the authentication token" src="https://github.com/user-attachments/assets/995cbd0e-1eae-4a70-a13b-acbf1d2229dc" />

```bash
tira-cli login --token <auth-token>
tira-cli verify-installation --task trec-auto-judge --team <your-team>
```

Scoping the verification to your task and team confirms not just the installation but also that your login and registration line up. A healthy verification looks like:

<img width="821" height="180" alt="tira-cli verify-installation output" src="https://github.com/user-attachments/assets/51160132-eb19-4da3-8892-8a53adb41c71" />


## Step 4 — Data submission: run your judge, upload run outputs

**Before you start** double check that

1. [the datasets are fetched](01-setup-environment.md#step-5--fetch-the-evaluation-datasets) into `./local-data/`, and
2. each dataset's `tira_id` (upload target) and `bucket` (meta-eval track) is correctly listed in `datasets.yml`.

**Run and warm prompt cache**

Run your judge locally with an LLM model of your choice, following the [run-workflow instructions](04-run-workflows.md) or using [`run_all_datasets.py`](04-run-workflows.md#running-against-multiple-datasets). We highly recommend using a prompt cache, which lets you verify and reproduce your runs.

```bash
python run_all_datasets.py --workflow judges/<your-judge>/workflow.yml --variant <variant> --dataset dragun-repgen
```

Double-check that the run produced the expected `.eval.txt` leaderboards in `ir_measures` format.

**Submit data to TIRA**

To upload the locally computed autojudge results from the run's output directory to TIRA, call:

```bash
tira-cli upload --dataset <tira-id> --directory <out-dir> --system <run-name>

# example (tira-id from datasets.yml; out-dir is the per-run folder the run above printed):
tira-cli upload --dataset dragun-repgen-20260608-test \
    --directory ./output/dragun-repgen/tinyjudge/context-all-all --system <my_runid>
```

`run_all_datasets.py` writes each result into its own folder so no two runs collide and each holds exactly one leaderboard — which is what `tira-cli upload` expects. The path is:

```
./output/<dataset>/<judge>/<variant>-<runs>-<topics>/
```

- `<dataset>` — the dataset name, `<judge>` — your judge's directory name, `<variant>` — the `--variant` (or `default`).
- `<runs>` — the `--runs` filter, `all` or `prio1`; `<topics>` — the `--topics` filter, `all` or `assessed`.

Both filters default to `all`, so a plain run lands in `<variant>-all-all` (above, `context-all-all` for the `tinyjudge` example's `context` variant). Add `--dry-run` to validate the leaderboard format without uploading.

**Alternative: run judge and upload results in one step**

`run_all_datasets.py` can run your judge on a dataset (or all datasets) and upload the leaderboards to TIRA in one command:

```bash
python run_all_datasets.py --workflow judges/<your-judge>/workflow.yml --variant <variant> \
    --dataset dragun-repgen --upload-tira
```

Drop `--dataset` to sweep every fetched dataset.

**Custom meta-evaluation service**

To use this infrastructure after the TIRA submission has closed, you can deposit your `*.eval.txt` into a meta-evaluation service via `rsync`:

```bash
python run_all_datasets.py --workflow judges/<your-judge>/workflow.yml --variant <variant> \
    --dataset dragun-repgen --upload-metaeval --metaeval-dest evalserver:/path/autojudge-eval/in
```

The dataset's `bucket` (from `datasets.yml`) is appended to the destination.

**Local meta-evaluation**

For method development, you can obtain a manual ground truth (such as the one provided for `kiddie`). With the dataset's `truth` leaderboard set in `datasets.yml`, run your judge and meta-evaluate against it locally:

```bash
python run_all_datasets.py --workflow judges/<your-judge>/workflow.yml --dataset kiddie --meta-evaluate
```



## Step 5 — Code Submission: Dry-run, then submit code

Now submit the code to match the data submission.

The dry run builds the image and tests it locally on kiddie to bring up any failures without uploading anything:

```bash
export OPENAI_API_KEY=...  OPENAI_BASE_URL=...  OPENAI_MODEL=...  CACHE_DIR=./cache

pytest
git commit -a

# Optional: run the same workflow/variant/model locally to validate end to end
# (and to seed ./cache, if you later switch to the warm-cache mount — see Prompt cache):
auto-judge run --workflow judges/<your-judge>/workflow.yml --variant <variant> \
    --rag-responses data/kiddie/runs/repgen/ --rag-topics data/kiddie/topics/kiddie-topics.jsonl \
    --out-dir ./output-kiddie/

tira-cli code-submission \
    --dry-run \
    --path . \
    --cache-behaviour deterministic \
    --mount-cache '$CACHE_DIR=EMPTY_DIR' \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --task trec-auto-judge \
    --dataset kiddie-20260605-training \
    --command 'auto-judge run --workflow /auto-judge/judges/<your-judge>/workflow.yml --variant <variant> --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'
```

When the dry run passes, remove `--dry-run` and run the same command to upload.



Details to know:

- **All judge specific command line options go inside the quoted `--command`.** There is no `tira-cli --variant` flag — `--variant`, and any other `auto-judge run` option, belongs inside the command string. `$inputDataset` and `$outputDir` are substituted by TIRA.
- **The cache flags** (`--cache-behaviour deterministic`, `--mount-cache '$CACHE_DIR=EMPTY_DIR'`) apply to LLM judges that cache — [Prompt cache](05-prompt-cache.md) explains the full lifecycle. Judges without an LLM can omit them. With `EMPTY_DIR`, TIRA starts from an empty cache and re-executes your judge deterministically to seed then replay it. Mounting your locally-seeded cache instead (`'$CACHE_DIR=cache'`) would let TIRA replay from it with no LLM calls — **pending confirmation that `tira-cli` uploads the mounted cache** (see [Prompt cache](05-prompt-cache.md)); seed it by running the same workflow, variant, and `OPENAI_MODEL` you submit, since mismatched prompts miss. The mount variable must match what your judge reads — `CACHE_DIR` by convention, backends may differ.
- **One submission covers one judge/variant.** Submit multiple variants by repeating the tira-cli command with a different `--command` string.


### Troubleshooting errors

- **`No module named '...'` inside the container, although it installs fine locally** — the `trec-auto-judge-base` image runs Python from `/venv` (`PATH=/venv/bin`), so dependencies must be installed *into that venv*: the Dockerfile needs `RUN . /venv/bin/activate && uv pip install -e .[all]`, **not** `uv pip install --system ...` (a system install is invisible at runtime). Bites exactly when your judge adds dependencies beyond the template's.
- **`Connection error` from your LLM client inside the container** — tira's local test runs **without network by default**, mirroring the TIRA sandbox. Testing against an external endpoint (OpenRouter, hosted OpenAI, ...) needs `--allow-network` on the `code-submission` command; alternatively, a mounted warm cache lets the judge complete with no network at all. Inside real TIRA the organizer-provided endpoint is cluster-internal, so this only concerns your local test.
- **`The cache directory mounted via CACHE_DIR was not used during the execution`** — tira-cli verifies that the mounted cache actually gets touched. Most often this is a *symptom*: the judge crashed before its first LLM call (check the error above it in the log). If the judge genuinely ran, it is not honoring `$CACHE_DIR` — see [Prompt cache](05-prompt-cache.md).

If anything fails — or you cannot run Docker locally at all — reach out in the private TIRA chat that we opened with your team at registration ([prerequisites](README.md#prerequisites)), and we will find a way to get your submission in.



## A complete session walkthrough

This walks the whole pipeline end to end with the starter kit's `tinyjudge` example — fetch a real dataset, run the judge (which warms its prompt cache and produces the leaderboards), upload those leaderboards, then ship the code.

It performs both TIRA uploads, which are independent — do either or both:

- **TIRA code upload** ships your Docker image and the organizers run it on every dataset, with their choice of LLMs. Reproducible, and the preferred path.
- **TIRA data upload** ships the leaderboards *you* produced locally; nothing re-runs on TIRA's side. Faster, and the fallback when your judge cannot ship as runnable code.

```bash
git clone git@github.com:<your-user>/<your-judge>.git && cd <your-judge>
uv venv && source .venv/bin/activate
uv pip install -e '.[all]'
uv pip install --upgrade tira

# container-runtime preflight: is Docker/podman ready to build and run containers?
# (prints the exact fix for each failed check; --fix applies the one safe self-repair)
./check_container_setup.sh

tira-cli login --token <auth-token>

# set LLM environment and prompt cache
export OPENAI_API_KEY=... OPENAI_BASE_URL=... OPENAI_MODEL=... CACHE_DIR=./cache

pytest                                # code runs and meets minimum requirements
git status --porcelain                # must be empty: gitignore build artifacts, commit the rest
git branch --show-current             # recommended: main branch


# local end-to-end test on the synthetic dataset
bash run_kiddie.sh

# good practice: meta-evaluate YOUR judge against kiddie's manual ground truth before submitting
python run_all_datasets.py --workflow judges/tinyjudge/workflow.yml --variant context \
    --dataset kiddie --meta-evaluate


# --- 1. fetch a real dataset (skip if already fetched in setup step 5; needs those credentials) ---
export TREC_AUTOJUDGE_USER=...  TREC_AUTOJUDGE_PASSWORD=...
./fetch_datasets.py --dataset dragun-repgen             # -> ./local-data/dragun25/

# --- 2. run the judge, then DATA-upload its leaderboards ---

#   The first run issues its LLM calls concurrently (this takes a while) and caches every
#   prompt and answer into ./cache, warming it for reuse.

python run_all_datasets.py --workflow judges/tinyjudge/workflow.yml --variant context \
    --dataset dragun-repgen --upload-tira

#   --upload-tira uploads the leaderboards to TIRA (the data submission).
#   Drop --dataset to run on every fetched dataset; add --dry-run first to
#   print the commands without executing them.


# --- 3. CODE-upload the judge (dry-run builds + tests locally, uploads nothing) ---
tira-cli verify-installation --task trec-auto-judge --team <your-team>

tira-cli code-submission --dry-run --path . \
    --cache-behaviour deterministic --mount-cache '$CACHE_DIR=EMPTY_DIR' \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --task trec-auto-judge --dataset kiddie-20260605-training \
    --command 'auto-judge run --workflow /auto-judge/judges/tinyjudge/workflow.yml --variant context --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'
#   EMPTY_DIR cold-starts and lets TIRA re-seed the cache deterministically. To instead ship the
#   ./cache you warmed above (cost-free replay on TIRA), swap in --mount-cache '$CACHE_DIR=cache'
#   — see Prompt cache (pending confirmation that the mounted cache uploads).

# dry run green? re-run the same command without --dry-run to upload the code,
# testing it on a real dataset instead of kiddie:
tira-cli code-submission --path . \
    --cache-behaviour deterministic --mount-cache '$CACHE_DIR=EMPTY_DIR' \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --task trec-auto-judge --dataset dragun-repgen-20260608-test \
    --command 'auto-judge run --workflow /auto-judge/judges/tinyjudge/workflow.yml --variant context --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'
```

## References
- [Fetch the datasets](01-setup-environment.md#step-5--fetch-the-evaluation-datasets) — download the released runs into `./local-data/`
- [Configure your LLM endpoint](02-configure-llm-endpoint.md) — how the endpoint reaches your judge inside the sandbox
- [Run workflows](04-run-workflows.md) — `auto-judge run`, variants, and `run_all_datasets.py`
- [Prompt cache](05-prompt-cache.md) — what the TIRA cache flags do
- [Meta-evaluation](06-meta-evaluation.md) — correlating your leaderboards against truth
- [TIRA participant documentation](https://docs.tira.io/participants/participate.html) — general TIRA submission background

