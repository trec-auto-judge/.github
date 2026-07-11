# 4. Run Workflows

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Developing practices](developing-practices.md) · Next: [Prompt cache](prompt-cache.md).*

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

## Development flags worth knowing

| Flag | Purpose |
|------|---------|
| `--limit-topics 2` | run on a subset of topics (fast iteration) |
| `--topic TOPIC_ID` | run on one specific topic |
| `--variant NAME` | run a named variant from `workflow.yml` (`--all-variants` runs them all) |
| `--sweep NAME` | run a parameter sweep |
| `-S KEY=VALUE` | override a shared setting |
| `-N KEY=VALUE` / `-J KEY=VALUE` | override a nugget / judge setting |
| `--llm-config FILE` | point at an LLM config ([details](configure-llm-endpoint.md)) |

Variants and sweeps let one `workflow.yml` express a family of configurations — see the workflow guide's [variants](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md#variants) and [parameter sweeps](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md#parameter-sweeps) sections.

## What lands in the output directory

Given `filebase: "myjudge"` and `--out-dir ./output/`:

| File | When produced | Purpose |
|------|--------------|---------|
| `myjudge.eval.txt` | `judge: true` | leaderboard in evaluation format — the primary input for [meta-evaluation](meta-evaluation.md) |
| `myjudge.judgment.json` | `judge: true` | leaderboard scores (JSON) |
| `myjudge.nuggets.jsonl` | `create_nuggets: true` | generated nugget banks |
| `myjudge.qrels` | `create_qrels: true` | relevance judgments |
| `myjudge.config.yml` | always | full config snapshot for reproducibility |

## Running against multiple datasets

`run_all_datasets.py` drives one `auto-judge run` per dataset listed in a `datasets.yml`:

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

Useful switches: `--dataset NAME` restricts to named datasets, `--runs prio1` / `--topics assessed` use the declared subsets, `--dry-run` prints the commands, and `--keep-going` continues past failures.

## References

- [Workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md) — canonical `workflow.yml` schema: execution phases, settings, variants, sweeps, lifecycle flags, CLI reference
- [auto-judge-base — CLI](https://github.com/trec-auto-judge/auto-judge-base#cli) — `run`, `export-corpus`, `list-models`
- [auto-judge-evaluate](https://github.com/trec-auto-judge/auto-judge-evaluate) — `leaderboard` (summary statistics) and `eval-result` (format conversion) for post-processing run outputs
