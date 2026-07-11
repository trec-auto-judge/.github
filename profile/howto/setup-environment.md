# 1. Set Up Your Dev Environment

*Part of the [TREC AutoJudge HowTo](README.md). Next: [Configure your LLM endpoint](configure-llm-endpoint.md).*

Building an auto-judge starts from the [AutoJudge Starter Kit](https://github.com/trec-auto-judge/auto-judge-starter-kit), a forkable template that already contains working example judges, a synthetic test dataset, and the packaging needed for TIRA submission. This page takes you from an empty machine to a verified working environment.

If you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit ships an interactive walkthrough of this page: type `/autojudge-setup`.

## Step 1 — Fork and clone the starter kit

Fork [auto-judge-starter-kit](https://github.com/trec-auto-judge/auto-judge-starter-kit) on GitHub, then:

```bash
git clone git@github.com:YOUR-USER/YOUR-FORK.git
cd YOUR-FORK
git remote add upstream git@github.com:trec-auto-judge/auto-judge-starter-kit.git
```

The upstream remote lets you pull template improvements later (`git fetch upstream && git merge upstream/main`). Library updates (`autojudge-base`, `minima-llm`, ...) arrive separately through `pip`/`uv pip install --upgrade`, not through the template.

## Step 2 — Create a virtual environment and install

```bash
uv venv
source .venv/bin/activate
uv pip install -e '.[all]'
```

The `.[all]` extra installs everything needed to develop, test, evaluate, and submit — use it unless you have a reason not to. Two things to watch:

- `uv venv` creates the environment but does **not** activate it. If you skip `source .venv/bin/activate`, the install may land in a different environment. Always activate first, then install.
- If you are still sorting out heavier tooling and just want to start coding, the lightweight `uv pip install -e .` installs only the judge runtime; switch to `.[all]` once you are ready to test or submit.

## Step 3 — Make the fork yours

Edit `pyproject.toml`:

| Field | Change to |
|-------|-----------|
| `name` | your judge's package name (anything but `auto-judge-starterkit`) |
| `description`, `authors`, `project.urls` | your details |
| `dependencies` | add what your judge needs (e.g. `dspy`, `litellm`) |

Keep `[tool.setuptools.packages.find]` with `include = ["judges*"]` unchanged — that is how your judge package gets discovered. After editing, reinstall:

```bash
uv pip install -e '.[all]' --refresh
```

Also replace the README's starter-kit overview with a description of your own approach.

## Step 4 — Verify the environment

Run the included smoke test, which executes an example judge on the synthetic `kiddie` dataset end to end:

```bash
bash run_kiddie.sh
pytest
```

Any failure at this point signals an environment problem — fix it now, before writing judge code. Once both pass, continue with [configuring your LLM endpoint](configure-llm-endpoint.md) and [developing practices](developing-practices.md).

## References

- [auto-judge-base — Installation](https://github.com/trec-auto-judge/auto-judge-base#installation) — the core library your judge builds on
- [minima-llm — Installation](https://github.com/trec-auto-judge/minima-llm#installation) — the optional batteries-included LLM client
