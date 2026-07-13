# 6. Meta-Evaluation

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Prompt cache](05-prompt-cache.md) · Next: [Submit to TIRA](07-submit-to-tira.md).*

Knowing whether your judge *agrees with human assessors* — not just whether it runs — is what meta-evaluation answers: it correlates the leaderboard your judge produces against a ground-truth leaderboard, so you can iterate on judge quality before submitting. The tooling lives in the [auto-judge-evaluate](https://github.com/trec-auto-judge/auto-judge-evaluate) package, installed with the starter kit's `.[evaluate]` extra (included in `.[all]`).

## The authoritative meta-evaluation runs on TIRA

We share only a **limited** slice of ground truth. Full human assessments stay held back — publishing them would invite tuning a judge to the test set, and a judge fitted to leaked truth tells us nothing about how it generalizes. So the meta-evaluation that counts is the one the organizers run on your [TIRA submission](07-submit-to-tira.md), scoring your judge's output against assessments you never see.

The `auto-judge-evaluate` package covers the other half — **local method development.** Against the synthetic `kiddie` truth (or whatever limited truth we release), you run the same correlation yourself to debug the pipeline, compare variants, and catch regressions before submitting. Read those numbers as directional; the submission is the verdict.

## The three things meta-evaluation correlates

Every correlation answers one question — *does my judge rank systems the way the truth does?* — and three inputs define it:

- **Truth measures** (`--truth-measure`) — metrics on the official ground-truth leaderboard, built from human assessments: the ranking you want to reproduce.
- **Eval measures** (`--eval-measure`) — the measures *your auto-judge* produces (coverage, average grade, ...): the ranking under test.
- **Correlation methods** (`--correlation`) — how the two rankings are compared: Kendall and Spearman reward getting the *ordering* of systems right (usually what a leaderboard is for), Pearson also rewards matching score magnitudes, and top-heavy variants like `kendall@15` weight agreement among the best systems, mirroring how leaderboards are actually read.

Leave any of the three unset and meta-evaluate uses all available values, emitting one row per (truth measure × eval measure × correlation method) — so you can see which of your judge's measures best tracks which aspect of the truth, under which notion of agreement.

## Local meta-evaluation on kiddie

The starter kit ships a synthetic truth leaderboard for the `kiddie` dataset, so the full loop works offline:

```bash
auto-judge-evaluate meta-evaluate \
    --truth-leaderboard data/kiddie/eval/kiddie_fake.eval.ir_measures.txt \
    --truth-format ir_measures --truth-header \
    --eval-format ir_measures \
    --on-missing default \
    output-kiddie/*.eval.txt
```

The command produces one correlation row per (judge, truth measure, eval measure, correlation method). Because the kiddie truth is **synthetic**, treat these numbers as a pipeline check, not a quality signal — meaningful correlations require real assessments. Narrow the output with `--truth-measure`, `--eval-measure`, and `--correlation` (each repeatable; all values used if omitted), and control how runs missing from either side are handled with `--on-missing`. The full option set is listed by `auto-judge-evaluate meta-evaluate --help` and documented in the [auto-judge-evaluate](https://github.com/trec-auto-judge/auto-judge-evaluate) README.

## Reading the results: table or JSONL

By default meta-evaluate prints a plain table to stdout — quick to read for a single run. To keep results for plotting or analysis, write them to a file:

- **`--output results.jsonl`** — one JSON record per correlation row, the format the analysis module and any pandas/plotting workflow expect. Prefer this whenever you will chart trends or compare many runs.
- **`--output results.txt`** (or `--out-format table`) — the same plain table, saved to disk.

For debugging *why* a correlation came out as it did, `--diagnostics-dir DIR` additionally dumps the per-comparison system rankings as JSONL.

## Meta-evaluation service

The meta-evaluation that decides your result runs on your [TIRA submission](07-submit-to-tira.md) against held assessments. As a lighter alternative for scoring outputs you already have, the [TREC AutoJudge meta-evaluation service](https://trec-auto-judge.cs.unh.edu/) scores uploaded `<variant>.eval.txt` files against the same held truth without re-running your code.

## Beyond leaderboard correlation

The same package covers the neighboring questions:

| Command | Question it answers |
|---------|--------------------|
| `auto-judge-evaluate meta-evaluate` | does my judge rank systems like the truth does? |
| `auto-judge-evaluate qrel-evaluate` | do my qrels agree with truth qrels? (precision/recall/F1, Cohen's κ, Krippendorff's α, ...) |
| `auto-judge-evaluate leaderboard` | what are my leaderboard's per-run summary statistics? |
| `auto-judge-evaluate eval-result` | convert/verify eval files between formats (`trec_eval`, `tot`, `ir_measures`, ...) |

For post-hoc analysis across many judges and datasets, the analysis module (`python -m autojudge_evaluate.analysis.correlation_table`) produces correlation tables (LaTeX/GitHub/TSV/HTML) and plots.

## References

- [auto-judge-evaluate](https://github.com/trec-auto-judge/auto-judge-evaluate) — full documentation of all four subcommands and the analysis module
