# 1. Set Up Your Dev Environment

*Part of the [TREC AutoJudge HowTo](README.md). Next: [Configure your LLM endpoint](02-configure-llm-endpoint.md).*

Building an auto-judge starts from the [AutoJudge Starter Kit](https://github.com/trec-auto-judge/auto-judge-starter-kit), a forkable template that already contains working example judges, a synthetic test dataset, and the packaging needed for TIRA submission. This page takes you from an empty machine to a verified working environment.

If you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit ships an interactive walkthrough of this page: type `/autojudge-setup`.

## Step 1 — Clone the starter kit into your own repository

Rather than a GitHub fork (which permanently labels your judge as "forked from the starter kit" and gets entangled in fork-based PR flows), clone the [auto-judge-starter-kit](https://github.com/trec-auto-judge/auto-judge-starter-kit) repository into a fresh repository of your own and keep the original around as a `starterkit` remote:

```bash
# Create an empty repository for your judge on GitHub (no README), then:
git clone git@github.com:trec-auto-judge/auto-judge-starter-kit.git YOUR-JUDGE
cd YOUR-JUDGE
git remote rename origin starterkit
git remote add origin git@github.com:YOUR-USER/YOUR-JUDGE.git
git push -u origin main
```

Your repository is now a first-class project: `origin` is yours, and the `starterkit` remote lets you pull template improvements later (`git fetch starterkit && git merge starterkit/main` — expect a few conflicts once you have diverged; resolve keeping your versions). Library updates (`autojudge-base`, `minima-llm`, ...) arrive separately through `pip`/`uv pip install --upgrade`, not through the template.

## Step 2 — Create a virtual environment and install

```bash
uv venv
source .venv/bin/activate
uv pip install -e '.[all]'
```

The `.[all]` extra installs everything needed to develop, test, evaluate, and submit — use it unless you have a reason not to. Two things to watch:

- `uv venv` creates the environment but does **not** activate it. If you skip `source .venv/bin/activate`, the install may land in a different environment. Always activate first, then install.
- If you are still sorting out heavier tooling and just want to start coding, the lightweight `uv pip install -e .` installs only the judge runtime; switch to `.[all]` once you are ready to test or submit.

## Step 3 — Make the repository yours (mandatory)

The starter kit is a bare-bones template, and building on it **requires** making it your own — a submission that still introduces itself as `auto-judge-starterkit` counts as unconfigured. The test suite enforces this: once your own `origin` remote is set, `pytest` fails until the project is renamed and the README replaced (inside the unmodified template these checks stay skipped). Edit `pyproject.toml`:

| Field | Change to |
|-------|-----------|
| `name` | your judge's package name (anything but `auto-judge-starterkit`) |
| `description`, `authors`, `project.urls` | your details |
| `dependencies` | add what your judge needs (e.g. `dspy`, `litellm`) |

Keep `[tool.setuptools.packages.find]` with `include = ["judges*"]` unchanged — that is how your judge package gets discovered. After editing, reinstall:

```bash
uv pip install -e '.[all]' --refresh
```

Also replace the README's starter-kit overview with a description of your own approach, and put your implementation in its own `judges/<yourjudge>/` directory (the [develop page](03-develop-an-autojudge.md) shows how) rather than editing the examples in place.

## Step 4 — Verify the environment

Run the included smoke test, which executes an example judge on the synthetic `kiddie` dataset end to end:

```bash
bash run_kiddie.sh
pytest
```

Any failure at this point signals an environment problem — fix it now, before writing judge code. A `No module named 'judges…'` or `Failed to load judge classes` error almost always means the virtual environment isn't active: run `source .venv/bin/activate` and retry. Once both pass, continue with [configuring your LLM endpoint](02-configure-llm-endpoint.md) and [developing practices](03-develop-an-autojudge.md).

## Step 5 — Fetch the evaluation datasets

The synthetic `kiddie` dataset ships with the kit (the smoke test above uses it), but the real evaluation runs come from a password-protected release. `fetch_pilot_dataset.sh` downloads them into `./local-data/` (gitignored):

```bash
export TREC_AUTOJUDGE_USER=...                  # basic-auth login from the organizers (e.g. trec2025)
export TREC_AUTOJUDGE_PASSWORD=...              # basic-auth password
./fetch_pilot_dataset.sh                        # all tracks, or one at a time:
./fetch_pilot_dataset.sh --dataset dragun-repgen
```

The script reads `TREC_AUTOJUDGE_USER` and `TREC_AUTOJUDGE_PASSWORD` from the environment (both required) — **never commit them**. It fetches the released run tarballs, extracts each track into `./local-data/<track>/`, and prints the resulting layout so you can confirm it matches the paths in `datasets.yml` (which lists every dataset with its `responses`/`topics`, its `tira_id`, and its meta-evaluation `bucket`). Pass `--keep-archive` to retain the downloaded `.tar.gz`.

This fetches the **pilot/training** data (v0.2); the TREC 2026 AutoJudge test data releases in August (a sibling `fetch_test_dataset.sh` will handle it). Corpora and topics for some tracks come from the host tracks — see the data release page. With the data in place, [run your judge over it](04-run-workflows.md).

## References

- [auto-judge-base — Installation](https://github.com/trec-auto-judge/auto-judge-base#installation) — the core library your judge builds on
- [minima-llm — Installation](https://github.com/trec-auto-judge/minima-llm#installation) — the optional batteries-included LLM client
