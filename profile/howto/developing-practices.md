# 3. Developing Practices

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Configure your LLM endpoint](configure-llm-endpoint.md) · Next: [Run workflows](run-workflows.md).*

A judge plugs into the framework by implementing the `AutoJudge` protocol — up to three methods that the workflow runner calls in order — and by declaring which of them to run in a `workflow.yml`. This page shows the two common shapes (minimal and full protocol) and the conventions that keep judges reproducible and cache-friendly.

## Where your code lives

Create a directory under `judges/` in your fork:

```
judges/myjudge/
  __init__.py
  my_judge.py       # your judge class(es)
  workflow.yml      # workflow configuration
```

Remember to `git add judges/myjudge/` — new directories start untracked. The starter kit's example judges (`judges/complete_example/`, and the minimal LLM example `judges/tinyjudge/`) serve as reference during development; delete the ones you did not write before [submitting](submit-to-tira.md).

## Minimal judge: leaderboard only

When your judge produces only a leaderboard, one method suffices:

```python
from autojudge_base import Leaderboard, LeaderboardBuilder, LeaderboardSpec, MeasureSpec

MY_SPEC = LeaderboardSpec(measures=(
    MeasureSpec("MY_SCORE", description="What this measure captures, its range, and how to interpret it"),
))

class MyJudge:
    def judge(self, rag_responses, rag_topics, llm_config, **kwargs) -> Leaderboard:
        builder = LeaderboardBuilder(MY_SPEC)
        for response in rag_responses:
            score = evaluate_response(response)  # your logic
            builder.add(
                run_id=response.metadata.run_id,
                topic_id=response.metadata.topic_id,
                values={"MY_SCORE": score},
            )
        topic_ids = [t.request_id for t in rag_topics]
        return builder.build(expected_topic_ids=topic_ids, on_missing="fix_aggregate")
```

paired with a minimal `workflow.yml`:

```yaml
judge_class: "judges.myjudge.my_judge:MyJudge"

create_nuggets: false
create_qrels: false
judge: true

settings:
  filebase: "{_name}"
```

## Full protocol: nuggets + qrels + leaderboard

Multi-phase judges first create nuggets (and optionally qrels), then judge with them:

```python
from autojudge_base import NuggetBanks

class MyJudge:
    nugget_banks_type = NuggetBanks

    def create_nuggets(self, rag_responses, rag_topics, llm_config, **kwargs):
        # generate nugget questions/claims per topic; return NuggetBanks or None
        return nugget_banks

    def create_qrels(self, rag_responses, rag_topics, llm_config, **kwargs):
        # generate relevance judgments; return Qrels or None
        return None

    def judge(self, rag_responses, rag_topics, llm_config, **kwargs) -> Leaderboard:
        nugget_banks = kwargs.get("nugget_banks")
        # score responses against the nuggets
        return leaderboard
```

with the phases enabled in `workflow.yml`:

```yaml
nugget_class: "judges.myjudge.my_judge:MyJudge"
judge_class: "judges.myjudge.my_judge:MyJudge"

create_nuggets: true
judge: true
nugget_depends_on_responses: true
judge_uses_nuggets: true

settings:
  filebase: "{_name}"
```

Separate classes per phase (`nugget_class`, `qrels_class`, `judge_class`) work as well — `judges/complete_example/` demonstrates the modular pattern, and the [workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md) documents every lifecycle flag.

## Conventions that keep judges well-behaved

- **Read the endpoint from `llm_config`, never hardcode keys or URLs** — on TIRA the organizer injects the endpoint through this parameter, as explained in [Configure your LLM endpoint](configure-llm-endpoint.md).
- **Describe every measure.** Give each `MeasureSpec` a `description` (what it represents, its range, how to read it); these export to `measures.yml` alongside your leaderboard and document your judge for organizers and downstream tooling.
- **Sort before you compare.** Order responses by `run_id` before building comparison pairs, so prompts — and therefore [prompt-cache](prompt-cache.md) keys — stay identical across runs.
- **Accept the injected output parameters.** All judge methods receive auto-filled `filebase: str = "default"` and `outdir: Path = Path(".")` for constructing output paths; declare them explicitly. Setting `filebase: "{_name}"` in `workflow.yml` names output files after the variant or sweep being run.
- **Layer configuration as env → yaml → cli**, each overriding the previous — the same rule the framework applies to LLM config.

## References

- [auto-judge-base — Quick Start & data classes](https://github.com/trec-auto-judge/auto-judge-base) — `Report`, `Request`, `Leaderboard`, `NuggetBanks`, and data-loading utilities (including `Report.get_sentences_with_citations()` for the unified sentence formats)
- [Leaderboard guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/leaderboard/README.md) — specs, builder, output formats, verification
- [Qrels guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/qrels/README.md) — building, verifying, and serializing TREC-format qrels
- [NuggetBank format](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/nugget_data/README.md) — the v3 nugget data model with verification
- [Workflow guide — settings, variants, custom formats](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md)
