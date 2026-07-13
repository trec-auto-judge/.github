# 4. Run Workflows

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Develop an AutoJudge](03-develop-an-autojudge.md) · Next: [Prompt cache](05-prompt-cache.md).*

The `auto-judge run` command executes your judge as declared in its `workflow.yml` — reading RAG responses and topics, calling your protocol methods in phase order, and writing the leaderboard and companion files to an output directory. This page covers the everyday commands; the [workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md) in auto-judge-base remains the canonical reference for the full `workflow.yml` schema (variants, sweeps, lifecycle flags, settings).

## The basic run

The starter kit ships the synthetic `kiddie` dataset for smoke testing:

```bash
auto-judge run \
    --workflow judges/myjudge/workflow.yml \
    --rag-responses data/kiddie/runs/repgen/ \
    --rag-topics data/kiddie/topics/kiddie-topics.jsonl \
    --out-dir ./output-kiddie/

auto-judge run --help    # all options
```

`bash run_kiddie.sh` wraps the same run plus a local meta-evaluation into one smoke test.

## What goes in a `workflow.yml`

Your judge's configuration belongs in its `workflow.yml`, not in your Python. Keep every tunable value — prompt style, number of nuggets, grading scale, thresholds — here as a *setting* rather than a hardcoded constant, so you can change behavior, run ablations, and submit variations without editing (and re-caching) a line of code. The essential entries:

```yaml
judge_class:  "judges.myjudge.my_judge:MyJudge"    # module:Class of your implementation
nugget_class: "judges.myjudge.my_judge:MyNuggets"  # only if you create nuggets
qrels_class:  "judges.myjudge.my_judge:MyQrels"    # only if you create qrels

create_nuggets: false        # which lifecycle phases run
create_qrels:   false
judge:          true
judge_uses_nuggets: false    # wiring: pass nuggets into judge(), etc.

settings:                    # kwargs delivered to every method
  filebase: "myjudge"        # names the output files
nugget_settings:             # kwargs for create_nuggets() only
  questions_per_topic: 20
judge_settings:              # kwargs for judge() only
  grading: "response"

uses_llm: true               # set false if the judge makes no LLM calls
```

Everything under `settings` / `nugget_settings` / `qrels_settings` / `judge_settings` arrives in the matching method as `**kwargs` — that contract is what lets you treat the workflow file as your judge's control panel instead of hunting through code. The [workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md) documents every key.

Two ways to change a setting without forking your code: a **variant** names a reusable block of overrides (below), selected with `--variant NAME`; a **CLI override** patches one value for one run — `-S key=value` (shared), `-N key=value` (nugget), `-J key=value` (judge) — handy for a quick experiment before you commit it as a variant.

## Development flags worth knowing

| Flag | Purpose |
|------|---------|
| `--limit-topics 2` | run on a subset of topics (fast iteration) |
| `--topic TOPIC_ID` | run on one specific topic |
| `--variant NAME` | run a named variant from `workflow.yml` (`--all-variants` runs them all) |
| `--sweep NAME` | run a parameter sweep |
| `-S KEY=VALUE` | override a shared setting |
| `-N KEY=VALUE` / `-J KEY=VALUE` | override a nugget / judge setting |

Variants and sweeps let one `workflow.yml` express a whole family of configurations. A variant names a block of setting overrides — with `filebase: "{_name}"`, each variant's output files carry its name automatically:

```yaml
settings:
  filebase: "{_name}"

variants:
  best:
    judge_settings:
      grading: "response"
  best-docs:
    judge_settings:
      grading: "docs"
```

`auto-judge run --variant best-docs ...` then produces `best-docs.eval.txt` and friends. Sweeps generalize this to a grid over setting values — the workflow guide's [variants](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md#variants) and [parameter sweeps](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md#parameter-sweeps) sections give the full semantics.

**Prefer variants for your submission's ablations.** When you want to compare method choices on TIRA — grade on responses vs. documents, 10 nuggets vs. 20 — express each as a variant and submit it with `--variant NAME` rather than editing code between runs. Every variant runs the same code under identical conditions, so the comparison is clean, and each `--variant` is its own [TIRA submission](07-submit-to-tira.md) (one variant per run).

## What lands in the output directory

Given `filebase: "myjudge"` and `--out-dir ./output/`:

| File | When produced | Purpose |
|------|--------------|---------|
| `myjudge.eval.txt` | `judge: true` | leaderboard in evaluation format — the primary input for [meta-evaluation](06-meta-evaluation.md) |
| `myjudge.judgment.json` | `judge: true` | leaderboard scores (JSON) |
| `myjudge.nuggets.jsonl` | `create_nuggets: true` | generated nugget banks |
| `myjudge.qrels` | `create_qrels: true` | relevance judgments |
| `myjudge.config.yml` | always | full config snapshot for reproducibility |

When inspecting `*.nuggets.jsonl` by hand, expect one `NuggetBank` JSON object per line, with the questions stored under `nugget_bank` as a **mapping keyed by nugget id** (not a list) — iterate its `.values()`.

## Running against multiple datasets

Each dataset is one `auto-judge run` — the same command as [the basic run](#the-basic-run) above, pointed at that dataset's responses and topics, plus `--corpus` when your judge reads documents:

```bash
auto-judge run \
    --workflow judges/myjudge/workflow.yml \
    --rag-responses data/dragun/runs/ \
    --rag-topics data/dragun/topics.jsonl \
    --corpus data/dragun/corpus.jsonl \
    --variant best \
    --out-dir ./output/dragun/
```

Run that directly for a single dataset. For several datasets, `run_all_datasets.py` automates exactly this — one `auto-judge run` per dataset listed in a `datasets.yml`, appending `--corpus` for datasets that declare one:

```bash
python run_all_datasets.py --workflow judges/myjudge/workflow.yml
python run_all_datasets.py --workflow judges/myjudge/workflow.yml --variant best --meta-evaluate
```

with datasets declared as:

```yaml
datasets:
  - name: kiddie
    responses: data/kiddie/runs/repgen/
    topics: data/kiddie/topics/kiddie-topics.jsonl
    truth: data/kiddie/eval/kiddie_fake.eval.ir_measures.txt   # optional, enables --meta-evaluate
    corpus: ...                                                # optional (recommended for doc-consulting judges)
    prio1_runs: [...]        # subset selected by --runs prio1
    assessed_topics: [...]   # subset selected by --topics assessed
```

`--dry-run` prints the exact `auto-judge run` command it would issue for each dataset — copy one to run a single dataset by hand. Other switches: `--dataset NAME` restricts to named datasets, `--runs prio1` / `--topics assessed` use the declared subsets, and `--keep-going` continues past failures.

## References

- [Workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md) — canonical `workflow.yml` schema: execution phases, settings, variants, sweeps, lifecycle flags, CLI reference
- [auto-judge-base — CLI](https://github.com/trec-auto-judge/auto-judge-base#cli) — `run`, `export-corpus`
- [auto-judge-evaluate](https://github.com/trec-auto-judge/auto-judge-evaluate) — `leaderboard` (summary statistics) and `eval-result` (format conversion) for post-processing run outputs
