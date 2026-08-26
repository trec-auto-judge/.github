# TREC 2026 AutoJudge Test — How to Participate

*Part of the [TREC AutoJudge HowTo](README.md). The activity pages ([1](01-setup-environment.md)–[7](07-submit-to-tira.md)) explain each step in depth; this page is the checklist for the 2026 test phase.*

The **TREC 2026 AutoJudge test data is released.** Participating means running your judge on the two test datasets and submitting the results — both as a data submission (your leaderboards) and as a code submission (your judge, for reproducible re-runs) — by the **submission deadline: September 30, 2026**. If you have worked through the HowTo, everything below is familiar; the only news is *which* datasets and *which* commands.

| Dataset | Host track | Systems to judge | Topics |
|---------|-----------|------------------|--------|
| `rag26-generation` | TREC 2026 RAG (generation) | 83 runs from 25 teams | 119 |
| `ragtime26-repgen` | TREC 2026 RAGTIME (report generation) | 49 runs from 10 teams | 103 |

> The runs are **anonymized**, and the [data-handling policy](data-policy.md) governs what you and your coding agent may look at. Your judge reads everything — that is the task; you don't. Each archive also ships the policy as `AGENTS.md` / `CLAUDE.md`.

## 1. Fetch the test data

With the release credentials (from the organizers) in your environment ([setup step 5](01-setup-environment.md#step-5--fetch-the-evaluation-datasets)):

```bash
export TREC_AUTOJUDGE_USER=...  TREC_AUTOJUDGE_PASSWORD=...
./fetch_pilot_dataset.sh --dataset rag26
./fetch_pilot_dataset.sh --dataset ragtime26
```

Each track extracts to `./local-data/<track>/`; the starter kit's `datasets.yml` already references both test datasets by name.

## 2. Run your judge and upload the leaderboards

One command per dataset runs your judge, meta-evaluates, and uploads the leaderboards to TIRA (the data submission):

```bash
python run_all_datasets.py --workflow judges/<your-judge>/workflow.yml \
    --meta-evaluate --dataset rag26-generation --upload-tira
python run_all_datasets.py --workflow judges/<your-judge>/workflow.yml \
    --meta-evaluate --dataset ragtime26-repgen --upload-tira
```

Add `--variant NAME` to submit a specific [variant](04-run-workflows.md), and `--dry-run` first to see the exact commands without executing. Use a [prompt cache](05-prompt-cache.md) (`CACHE_DIR`) so re-runs are free and your results are reproducible.

Two things to know when reading the output:

- **The local meta-evaluation is a pipeline check only.** The test releases ship a *placeholder* truth leaderboard — the correlations you see locally are meaningless by design. The meta-evaluation that counts runs on the organizers' side against held-out assessments ([why](06-meta-evaluation.md#the-authoritative-meta-evaluation-runs-on-tira)).
- **Judge all runs.** Submissions must cover every run in the dataset — partial submissions (e.g. `--runs prio1`) cannot be accepted.

## 3. Submit your code

Ship the same judge/variant as a code submission so we can re-run it — the full procedure, dry run included, is in [Submit to TIRA](07-submit-to-tira.md):

```bash
tira-cli code-submission --dry-run --path . \
    --cache-behaviour deterministic --mount-cache '$CACHE_DIR=EMPTY_DIR' \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --task trec-auto-judge --dataset kiddie-20260605-training \
    --command 'auto-judge run --workflow /auto-judge/judges/<your-judge>/workflow.yml --variant <variant> --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'
```

## Questions?

Use the private TIRA chat we opened with your team at [registration](README.md#prerequisites).
