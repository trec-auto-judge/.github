# 7. Submit to TIRA

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Meta-evaluation](meta-evaluation.md).*

Submitting your auto-judge means making a **code submission**: `tira-cli` builds your repository's Dockerfile into an image, tests that image locally on the kiddie dataset, and — only if the outputs validate — uploads it to TIRA, where we run it on all datasets, potentially with multiple LLMs. Complete the [prerequisites](README.md#prerequisites) (TIRA account, team registration) before starting here.

If you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit ships an interactive walkthrough of this page: type `/autojudge-submit`.

## Step 1 — Meet the submission requirements

- **All code lives in a git repository** (private or even local-only is fine — we never check that changes are pushed).
- **The repository is clean.** `git status --porcelain` must print nothing; any output means uncommitted or untracked changes that you need to commit or `.gitignore`.
- **The currently checked-out branch is what gets submitted**, and only its *committed* state — uncommitted edits are silently excluded. Check with `git branch --show-current`.
- **`pytest` completes without failures.**
- **Example judges you did not write are deleted** so they do not ship with your submission.
- **A Dockerfile at the repo root** specifies how your software is dockerized. Making it [dev-container](https://containers.dev/) compatible lets you develop directly inside the container.
- **The sandbox has no internet access.** Your judge receives its LLM endpoint through forwarded environment variables or model preferences — see [Configure your LLM endpoint](configure-llm-endpoint.md). Nothing else on the network will be reachable.
- **No secrets in the image.** Never `COPY`/`ADD` API keys into the Dockerfile; pass them only via `--forward-environment-variable` so they are injected at run time.

## Step 2 — Install tira-cli and start Docker

```bash
pip3 install --upgrade tira
```

The Docker (or podman) daemon must be **running** when you submit — the build-and-test happens on your machine before anything is uploaded.

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
tira-cli verify-installation
```

A healthy verification looks like:

<img width="821" height="180" alt="tira-cli verify-installation output" src="https://github.com/user-attachments/assets/51160132-eb19-4da3-8892-8a53adb41c71" />

## Step 4 — Dry-run, then submit

The dry run builds the image and tests it locally on kiddie without uploading anything:

```bash
export OPENAI_API_KEY=...  OPENAI_BASE_URL=...  OPENAI_MODEL=...

tira-cli code-submission \
    --dry-run \
    --path . \
    --cache-behaviour deterministic \
    --mount-cache '$CACHE_DIR=EMPTY_DIR' \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --task trec-auto-judge \
    --dataset kiddie-20260605-training \
    --command 'auto-judge run --workflow /auto-judge/judges/myjudge/workflow.yml --variant best --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'
```

Three details that trip people up:

- **Everything your judge needs goes inside the quoted `--command`.** There is no `tira-cli --variant` flag — `--variant`, and any other `auto-judge run` option, belongs inside the command string. `$inputDataset` and `$outputDir` are substituted by TIRA.
- **The cache flags** (`--cache-behaviour deterministic`, `--mount-cache '$CACHE_DIR=EMPTY_DIR'`) apply to LLM judges that cache — [Prompt cache](prompt-cache.md) explains what they do. Judges without an LLM can omit them.
- **One submission covers one judge/variant.** Submit multiple variants by repeating the command with a different `--command` string.

When the dry run passes, remove `--dry-run` and run the same command to upload.

If anything fails — or you cannot run Docker locally at all — reach out in the private TIRA chat that we opened with your team at registration ([prerequisites](README.md#prerequisites)), and we will find a way to get your submission in.

## References

- [Configure your LLM endpoint](configure-llm-endpoint.md) — how the endpoint reaches your judge inside the sandbox, including `model_preferences` resolution
- [Prompt cache](prompt-cache.md) — what the TIRA cache flags do
- [TIRA participant documentation](https://docs.tira.io/participants/participate.html) — general TIRA submission background
