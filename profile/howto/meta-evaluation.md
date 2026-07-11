# 6. Meta-Evaluation

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Prompt cache](prompt-cache.md) · Next: [Submit to TIRA](submit-to-tira.md).*

Knowing whether your judge *agrees with human assessors* — not just whether it runs — is what meta-evaluation answers: it correlates the leaderboard your judge produces against a ground-truth leaderboard, so you can iterate on judge quality before submitting. The tooling lives in the [auto-judge-evaluate](https://github.com/trec-auto-judge/auto-judge-evaluate) package, installed with the starter kit's `.[evaluate]` extra (included in `.[all]`).

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

The output reports one correlation row per (judge, truth measure, eval measure). Because the kiddie truth is **synthetic**, treat these numbers as a pipeline check, not a quality signal — meaningful correlations require real assessments.

Correlation methods are selectable (`--correlation kendall`, `pearson`, `spearman`, `tauap_b`, or top-heavy variants like `kendall@15`), and `--on-missing` controls how runs missing from either side are handled.

## Meta-evaluation service

For real TREC datasets, upload your `<variant>.eval.txt` files to the [TREC AutoJudge meta-evaluation service](https://trec-auto-judge.cs.unh.edu/), which scores them against held assessments.

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
