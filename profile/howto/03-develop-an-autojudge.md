# 3. Develop an AutoJudge

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Configure your LLM endpoint](02-configure-llm-endpoint.md) · Next: [Run workflows](04-run-workflows.md).*

A judge plugs into the framework by implementing the `AutoJudge` protocol — up to three methods that the workflow runner calls in order — and by declaring which of them to run in a `workflow.yml`. This page first shows the two common judge shapes (minimal and full protocol), then walks through each part of the data model with guidance on how to approach it, and closes with the conventions that keep judges reproducible. In [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit's `/autojudge-develop` skill walks you through this page and the three that follow (running, caching, meta-evaluation).

## Where your code lives

Create a directory under `judges/` in your fork:

```
judges/myjudge/
  __init__.py
  my_judge.py       # your judge class(es)
  workflow.yml      # workflow configuration
```

Remember to `git add judges/myjudge/` — new directories start untracked. Delete the example judges you did not write before [submitting](07-submit-to-tira.md).

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

## Working with the data model

### Reading responses and topics

Your judge receives an iterable of `Report` objects (one per system response: `metadata.run_id`, `metadata.topic_id`, the response text and sentences) and a sequence of `Request` objects (the evaluation topics: `request_id`, title, background, problem statement). Because RAG tracks differ in how they attach citations, `Report.get_sentences_with_citations()` normalizes the three sentence formats into one — reach for it whenever your judge needs to check what a response *cites* rather than just what it says.

→ API: [auto-judge-base — Data Loading Utilities](https://github.com/trec-auto-judge/auto-judge-base#data-loading-utilities)

### Calling the LLM in batches

Judges make many similar LLM calls — one per response, per pair, or per nugget×response — so build the full request list first and hand it to a batched runner instead of looping over blocking calls. Two starter-kit-proven patterns:

- **Plain prompts** (see `judges/tinyjudge/`): construct one `MinimaLlmRequest` per item, then let `OpenAIMinimaLlm(config).run_batched(requests)` parallelize, rate-limit, retry, and cache.
- **DSPy signatures** (see the [prefnugget-starterkit](https://github.com/laura-dietz/prefnugget-starterkit)): declare a `dspy.Signature` per judgment type — with explicit `reasoning` and `confidence` output fields — then run `run_dspy_batch(signature, items, converter, backend=...)` from minima-llm's DSPy adapter, gaining structured outputs with the same batching and caching underneath.

Either way, take the endpoint from `llm_config` as described in [Configure your LLM endpoint](02-configure-llm-endpoint.md).

→ API: [minima-llm — Quick Start & DSPy](https://github.com/trec-auto-judge/minima-llm#quick-start)

### Building the leaderboard

Declare the schema once in a `LeaderboardSpec` — each `MeasureSpec` carries a name, an optional `cast` (normalize incoming values), an `aggregate` (how per-topic values combine into the per-run `all` row), and a `description` — then let `LeaderboardBuilder` assemble the rows. The builder fails fast on typo'd or missing measure keys, computes the `all` rows automatically, and `build(expected_topic_ids=..., on_missing=...)` checks that every expected topic actually got scored, so a partially-failed run cannot silently produce a plausible-looking leaderboard. If your scores already sit in a list of records, `builder.add_records(records, run_id=..., topic_id=..., get_values=...)` saves the loop.

→ API: [Leaderboard guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/leaderboard/README.md) — specs, builder, output formats (`tot`, `ir_measures`), fluent verification

### Creating nuggets

Nuggets capture what a good answer should contain — as questions (`NuggetQuestion` with gold `Answer`s, optionally sub-nuggets and document `Reference`s) or as fact-like claims (`NuggetClaim`). Collect them per topic in a `NuggetBank` (`bank.add_nuggets([...])`, plus a `Creator` record documenting whether a human or an LLM produced them), and return all banks keyed by topic as `NuggetBanks`. Two strategies to learn from:

- **Query-only generation** — ask the LLM for nugget questions from the topic alone (prefnugget's `queryonly` judge): simple, cheap, and independent of the responses being judged.
- **Response-grounded / contrastive extraction** — mine nuggets from the responses themselves; prefnugget's main judges first rank responses by pairwise LLM preference, then iteratively extract *differentiating* questions from winner/loser pairs, deduplicating each round and capping the bank (e.g. at 20 questions).

The judging phase then typically grades every (response, nugget) pair and aggregates — coverage, average grade, max grade — into the leaderboard measures.

→ API: [NuggetBank format](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/nugget_data/README.md) — the v3 data model with verification

### Creating qrels

Qrels record fine-grained relevance judgments as `(topic_id, doc_id, grade)` rows. Define a `QrelsSpec` with three extractor functions (`topic_id`, `doc_id`, `grade`, plus an `on_duplicate` policy), build with `build_qrels(records, spec)`, verify coverage with `qrels.verify(expected_topic_ids=...)`, and serialize with `write_qrel_file(...)` into standard TREC format, deterministically sorted for reproducibility. When judging generated text that has no corpus ID, derive stable ids with the `doc_id_md5` helper (`md5:<hash of text>`).

→ API: [Qrels guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/qrels/README.md) — building, verifying, and serializing TREC-format qrels

### Configuring hyperparameters

Anything you might want to vary — prompt style, number of nuggets, grading scale — belongs in `workflow.yml` `settings` (or the phase-specific `nugget_settings`/`judge_settings`), which arrive in your methods as `**kwargs`. Named `variants` then override settings per configuration, so one workflow file expresses your whole method family and [`--variant`](04-run-workflows.md) selects one member; prefnugget's workflow files, with a dozen variants each, show this at scale.

→ API: [Workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md) — settings, variants, sweeps, custom nugget formats

## Learn from existing judges

| Example | What to learn from it |
|---------|----------------------|
| `judges/complete_example/` (starter kit) | the full three-protocol structure, modular classes, an exhaustively-commented `workflow.yml` with variants and sweeps |
| `judges/tinyjudge/` (starter kit) | the smallest realistic LLM judge: batched requests, prompt caching, clean `llm_config` handling |
| [prefnugget-starterkit](https://github.com/laura-dietz/prefnugget-starterkit) | a research-grade judge family: pairwise preference ranking, contrastive nugget extraction, DSPy signatures, variants at scale |
| [ir_axioms AutoJudge](https://github.com/webis-de/ir_axioms/tree/main/trec-auto-judge) | an axiomatic (non-generative) judge, including how to package heavyweight dependencies in Docker |

## Conventions that keep judges well-behaved

- **Read the endpoint from `llm_config`, never hardcode keys or URLs** — on TIRA the organizer injects the endpoint through this parameter ([details](02-configure-llm-endpoint.md)).
- **Describe every measure.** Each `MeasureSpec` `description` (what it represents, its range, how to read it) exports to `measures.yml` alongside your leaderboard and documents your judge for organizers and downstream tooling.
- **Sort before you compare.** Order responses by `run_id` before building comparison pairs, so prompts — and therefore [prompt-cache](05-prompt-cache.md) keys — stay identical across runs.
- **Accept the injected output parameters.** All judge methods receive auto-filled `filebase: str = "default"` and `outdir: Path = Path(".")` for constructing output paths; declare them explicitly. Setting `filebase: "{_name}"` in `workflow.yml` names output files after the variant or sweep being run.
- **Verify before returning.** The `build(expected_topic_ids=...)`, `leaderboard.verify(...)`, and `qrels.verify(...)` checks exist to fail fast — silent topic drop-out is the classic way a judge produces a wrong-but-plausible leaderboard.

## References

- [auto-judge-base — Quick Start & data classes](https://github.com/trec-auto-judge/auto-judge-base) — `Report`, `Request`, `Leaderboard`, `NuggetBanks`, data-loading utilities
- [Leaderboard guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/leaderboard/README.md) · [Qrels guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/qrels/README.md) · [NuggetBank format](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/nugget_data/README.md) · [Workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md)
- [minima-llm](https://github.com/trec-auto-judge/minima-llm) — batching, caching, DSPy adapter
