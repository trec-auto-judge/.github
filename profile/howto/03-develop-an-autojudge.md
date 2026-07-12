# 3. Develop an AutoJudge

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Configure your LLM endpoint](02-configure-llm-endpoint.md) · Next: [Run workflows](04-run-workflows.md).*

A judge plugs into the framework by implementing the `AutoJudge` protocol — up to three methods that the workflow runner calls in order — and by declaring which of them to run in a `workflow.yml`. This page first shows the two common judge shapes (minimal and full protocol), then walks through each part of the data model with guidance on how to approach it, and closes with the conventions that keep judges reproducible. In [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit's `/autojudge-develop` skill walks you through this page and the three that follow (running, caching, meta-evaluation).

## Where your code lives

Create a directory under `judges/` in your repository:

```
judges/myjudge/
  __init__.py
  my_judge.py       # your judge class(es)
  workflow.yml      # workflow configuration
```

Remember to `git add judges/myjudge/` — new directories start untracked. Delete the example judges you did not write before [submitting](07-submit-to-tira.md).

### Minimum-compatibility tests, for free

The starter kit's `pytest` suite exists to check **every judge implementation in your repo for minimum compatibility with the framework** — and it finds your judges by itself. Discovery works off `git ls-files judges/*/workflow.yml`: every *git-tracked* workflow becomes its own test case (deliberately ignoring untracked local leftovers), and for each one the suite verifies the two things the framework will do to your judge at load time:

1. the `workflow.yml` parses as valid YAML, and
2. every declared class reference (`judge_class`, `nugget_class`, `qrels_class` — the `module:ClassName` strings) imports and resolves,

which is exactly how `auto-judge run` — locally and inside TIRA — loads your judge. A judge that fails these checks cannot run at all, so keeping `pytest` green is a [submission requirement](07-submit-to-tira.md). The moment you `git add` a new judge directory, it is covered; no test edits needed. Alongside them, a customization check fails until the template is [made your own](01-setup-environment.md) (project renamed, README replaced), and an endpoint-contract test runs each judge against a local pretend LLM endpoint to verify the [injected environment variables](02-configure-llm-endpoint.md#the-endpoint-contract) actually reach your LLM client — a judge with a hardcoded endpoint or model will not run on TIRA, and that failure would otherwise only surface after submission. Judges that make no LLM calls declare `uses_llm: false` in their own `workflow.yml`, which marks their case as expected-to-fail (`xfail`). These checks are the *minimum* — add judge-specific tests (parsers, scoring logic, aggregation) on top, as the example judges do.

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

Your judge receives an iterable of `Report` objects (one system's answer to one topic) and a sequence of `Request` objects (the topics). The fields you actually reach for:

**`Request`** — the evaluation topic:

| Field | Type | Meaning |
|-------|------|---------|
| `request_id` | `str` | topic id — the key that ties requests, reports, nuggets, and qrels together |
| `title` | `str` | the query/topic title (the only always-present text field) |
| `background` | `str?` | narrative context for the information need |
| `problem_statement` | `str?` | what a good answer must resolve |
| `collection_ids` | `list[str]?` | corpora the answers draw from |
| `word_limit` / `limit` | `int?` | length budget the systems were held to |

**`Report`** — one system response:

| Field | Type | Meaning |
|-------|------|---------|
| `metadata` | `ReportMetaData` | identifiers: `.run_id`, `.team_id`, `.topic_id` (aligns with `Request.request_id`) |
| `responses` | `list[ReportSentence]` | the answer, sentence by sentence (each has `.text` and `.citations`) |
| `references` | `list[str]?` | doc ids the response draws on; for index-style citations, the lookup table they resolve against |
| `documents` | `dict[str, Document]?` | cited/retrieved documents by id, when the track ships them inline |
| `ranking` | `list[RetrievedDocuments]?` | a retrieval run attached to the report, for judges that score retrieval |

Rather than touch `responses` directly, prefer the accessors: `get_text()` (whole answer), `get_sentences()` (plain strings), `get_paragraphs()`, and — because the tracks disagree on how citations attach (RAGtime maps doc-id→confidence, NeuCLIR lists doc-ids, RAG'24 lists indices into `references`) — `get_sentences_with_citations()`, which normalizes all three into `NeuclirReportSentence` with `.citations` as a plain `list[str]` of doc ids. Reach for it whenever your judge cares what a response *cites*, not just what it says.

**`Document`** — an entry in `Report.documents` (or a corpus you export):

| Field | Type | Meaning |
|-------|------|---------|
| `id` | `str` | document id, matching what citations reference |
| `text` | `str` | body text |
| `title` / `url` | `str?` | when the source provides them |
| `metadata` | `dict?` | track-specific extras (the model allows unknown keys) |

Use `document.get_text()` to get title and body joined, rather than concatenating by hand.

→ API: [auto-judge-base — Data Loading Utilities](https://github.com/trec-auto-judge/auto-judge-base#data-loading-utilities)

### Calling the LLM

Building the judgments means many similar LLM calls — one per response, per pair, or per nugget×response. Take the endpoint from the injected `llm_config` and prefer a batched runner over a loop of blocking calls. [Configure your LLM endpoint](02-configure-llm-endpoint.md#choose-your-llm-client--any-openai-compatible-client-works) covers the client choices (LangChain, litellm, minima-llm), their batching APIs, and how to run DSPy signatures.

→ API: [minima-llm — Quick Start & DSPy](https://github.com/trec-auto-judge/minima-llm#quick-start)

### Building the leaderboard

Declare the schema once in a `LeaderboardSpec` — each `MeasureSpec` carries a name, an optional `cast` (normalize incoming values), an `aggregate` (how per-topic values combine into the per-run `all` row), and a `description` — then let `LeaderboardBuilder` assemble the rows. The builder fails fast on typo'd or missing measure keys, computes the `all` rows automatically, and `build(expected_topic_ids=..., on_missing=...)` checks that every expected topic actually got scored, so a partially-failed run cannot silently produce a plausible-looking leaderboard. If your scores already sit in a list of records, `builder.add_records(records, run_id=..., topic_id=..., get_values=...)` saves the loop.

→ API: [Leaderboard guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/leaderboard/README.md) — specs, builder, output formats (`tot`, `ir_measures`), fluent verification

### Creating nuggets

Nuggets capture what a good answer should contain — as questions (`NuggetQuestion` with gold `Answer`s, optionally sub-nuggets and document `Reference`s) or as fact-like claims (`NuggetClaim`). Collect them per topic in a `NuggetBank` (`bank.add_nuggets([...])`, plus a `Creator` record documenting whether a human or an LLM produced them), and return all banks keyed by topic as `NuggetBanks`. Two strategies to learn from:

- **Query-only generation** — ask the LLM for nugget questions from the topic alone (prefnugget's `queryonly` judge): simple, cheap, and independent of the responses being judged.
- **Response-grounded / contrastive extraction** — mine nuggets from the responses themselves; prefnugget's main judges first rank responses by pairwise LLM preference, then iteratively extract *differentiating* questions from winner/loser pairs, deduplicating each round and capping the bank (e.g. at 20 questions).

The judging phase then typically grades every (response, nugget) pair and aggregates — coverage, average grade, max grade — into the leaderboard measures.

Note that the workflow runner verifies nugget banks before judging: a topic with an *empty* bank fails the run (`NuggetBanksVerificationError`). An empty bank usually means the LLM calls failed silently — check the [endpoint configuration](02-configure-llm-endpoint.md#troubleshooting) before suspecting your extraction logic.

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
- **Preflight the endpoint — but warn, don't raise.** One probe LLM call at the start of each phase surfaces a stale model id or bad key as one clear provider error instead of hundreds of failed batch calls (see [endpoint troubleshooting](02-configure-llm-endpoint.md#troubleshooting)). Make it a *warning*: a judge with a populated [prompt cache](05-prompt-cache.md) must be able to complete with no working endpoint at all — TIRA's deterministic re-execution runs it exactly that way. Let the all-calls-failed condition be the hard error instead.

## References

- [auto-judge-base — Quick Start & data classes](https://github.com/trec-auto-judge/auto-judge-base) — `Report`, `Request`, `Leaderboard`, `NuggetBanks`, data-loading utilities
- [Leaderboard guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/leaderboard/README.md) · [Qrels guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/qrels/README.md) · [NuggetBank format](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/nugget_data/README.md) · [Workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md)
- [minima-llm](https://github.com/trec-auto-judge/minima-llm) — batching, caching, DSPy adapter
