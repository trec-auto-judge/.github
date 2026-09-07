# 3. Develop an AutoJudge

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Configure your LLM endpoint](02-configure-llm-endpoint.md) · Next: [Run workflows](04-run-workflows.md).*

> Working with evaluation data? The [data-handling policy](data-policy.md) governs what your coding agent may look at, and how to debug a failure it may not inspect.

A judge plugs into the framework by implementing the `AutoJudge` protocol — up to three methods that the workflow runner calls in order — and by declaring which of them to run in a `workflow.yml`. This page first shows the three common judge shapes (minimal, full protocol, and judge-only with external nugget banks), then walks through each part of the data model with guidance on how to approach it, and closes with the conventions that keep judges reproducible. In [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit's `/autojudge-develop` skill walks you through this page and the three that follow (running, caching, meta-evaluation).

## Where your code lives

Create a directory under `judges/` in your repository:

```
judges/myjudge/
  __init__.py
  my_judge.py       # your judge class(es)
  workflow.yml      # workflow configuration
```

Remember to `git add judges/myjudge/` — new directories start untracked. Delete the example judges you did not write before [submitting](07-submit-to-tira.md). For a quick end-to-end smoke test of your judge on the kiddie dataset, point the `WORKFLOW` line in `run_kiddie.sh` at your `workflow.yml`.

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

## Judge-only with externally built nugget banks

A third shape sits between the two: the judge implements only `judge()` (the `LeaderboardJudgeProtocol`), and the nugget banks come from elsewhere — a previous run, another judge, or a hand-curated file — supplied at run time with [`--nugget-banks`](04-run-workflows.md#development-flags-worth-knowing) (a JSON/JSONL file or a directory). The class **must** still declare `nugget_banks_type`, or the runner cannot deserialize the banks:

```python
from autojudge_base import (
    Leaderboard,
    LeaderboardBuilder,
    LeaderboardSpec,
    MeasureSpec,
    NuggetBanks,
)

MY_SPEC = LeaderboardSpec(measures=(
    MeasureSpec("NUGGET_SCORE", description="Average coverage of the external nuggets"),
))

class MyJudge:
    nugget_banks_type = NuggetBanks   # required — tells the runner how to load --nugget-banks

    def judge(self, rag_responses, rag_topics, llm_config, **kwargs) -> Leaderboard:
        nugget_banks = kwargs.get("nugget_banks")
        builder = LeaderboardBuilder(MY_SPEC)
        for response in rag_responses:
            score = evaluate_response(response, nugget_banks)  # your logic
            builder.add(
                run_id=response.metadata.run_id,
                topic_id=response.metadata.topic_id,
                values={"NUGGET_SCORE": score},
            )
        topic_ids = [topic.request_id for topic in rag_topics]
        return builder.build(expected_topic_ids=topic_ids, on_missing="fix_aggregate")
```

with `workflow.yml` wiring the banks into `judge()` while leaving nugget creation off:

```yaml
judge_class: "judges.myjudge.my_judge:MyJudge"

create_nuggets: false
judge: true
judge_uses_nuggets: true

settings:
  filebase: "{_name}"
```

```bash
auto-judge run --workflow judges/myjudge/workflow.yml \
    --nugget-banks path/to/banks.nuggets.jsonl ...
```

## Qrels judge: grade once, aggregate into the leaderboard

A judge that grades relevance directly records its judgments as qrels and derives the leaderboard from them. `create_qrels()` runs the grading once and returns the verified `Qrels`; with `judge_uses_qrels: true` the runner passes the same rows into `judge()`, which aggregates them into leaderboard measures without a second grading pass:

```python
from autojudge_base import Leaderboard, Qrels

class MyJudge:
    def create_qrels(self, rag_responses, rag_topics, llm_config, **kwargs) -> Qrels:
        # grade each report (doc_id = run_id) or each cited document (doc_id = corpus id),
        # build with build_qrels, verify, return — see "Creating qrels" below
        return qrels

    def judge(self, rag_responses, rag_topics, llm_config, qrels=None, **kwargs) -> Leaderboard:
        # aggregate the qrels rows into leaderboard measures
        return leaderboard
```

```yaml
qrels_class: "judges.myjudge.my_judge:MyJudge"
judge_class: "judges.myjudge.my_judge:MyJudge"

create_qrels: true
judge: true
judge_uses_qrels: true
```

`create_qrels` defaults on when `judge_uses_qrels` is set, mirroring how `create_nuggets` follows the nugget wiring flags.

## Working with the data model

### Reading responses and topics

Your judge receives an iterable of `Report` objects (one system's answer to one topic) and a sequence of `Request` objects (the topics). The tables below name the fields judges reach for most; because these models evolve, treat the generated [API reference](https://trec-auto-judge.github.io/auto-judge-base/) as the authoritative, always-current listing — [`Report`](https://trec-auto-judge.github.io/auto-judge-base/api/data-models/#autojudge_base.report.Report), [`Request`](https://trec-auto-judge.github.io/auto-judge-base/api/data-models/#autojudge_base.request.Request), and [`Document`](https://trec-auto-judge.github.io/auto-judge-base/api/data-models/#autojudge_base.document.document.Document) list every field with its type and default, straight from the pydantic sources.

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

If you reuse `Report` objects to build a track submission of your own, verify each before writing it: `Report.verify_rag()` checks the TREC RAG 2026 fields (required metadata, deduped references, ≤3 citations/sentence, every reference cited, 1024-word limit); `Report.verify_ragtime()` checks the RAGTIME format. Judges that only read reports skip this; a submission run should not.

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

→ API: [Leaderboard guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/leaderboard/README.md) — specs, builder, output formats (`tot`, `ir_measures`), fluent verification · reference: [Leaderboard](https://trec-auto-judge.github.io/auto-judge-base/api/leaderboard/)

### Creating nuggets

Nuggets capture what a good answer should contain — as questions (`NuggetQuestion` with gold `Answer`s, optionally sub-nuggets and document `Reference`s) or as fact-like claims (`NuggetClaim`). Collect them per topic in a `NuggetBank` (`bank.add_nuggets([...])`, plus a `Creator` record documenting whether a human or an LLM produced them), and return all banks keyed by topic as `NuggetBanks`. Two strategies to learn from:

- **Query-only generation** — ask the LLM for nugget questions from the topic alone: simple, cheap, and independent of the responses being judged.
- **Response-grounded / contrastive extraction** — mine nuggets from the responses themselves; for example, rank responses by pairwise LLM preference, then iteratively extract *differentiating* questions from winner/loser pairs, deduplicating each round and capping the nugget bank (e.g. at 20 questions).

The judging phase then typically grades every (response, nugget) pair and aggregates — coverage, average grade, max grade — into the leaderboard measures.

Note that the workflow runner verifies nugget banks before judging: a topic with an *empty* bank fails the run (`NuggetBanksVerificationError`). An empty bank usually means the LLM calls failed silently — check the [endpoint configuration](02-configure-llm-endpoint.md#troubleshooting) before suspecting your extraction logic.

→ API: [NuggetBank format](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/nugget_data/README.md) — the v3 data model with verification · reference: [Nuggets](https://trec-auto-judge.github.io/auto-judge-base/api/nuggets/)

### Creating qrels

Qrels record fine-grained relevance judgments as `(topic_id, doc_id, grade)` rows. Define a `QrelsSpec` with three extractor functions (`topic_id`, `doc_id`, `grade`, plus an `on_duplicate` policy), build with `build_qrels(records, spec)`, verify coverage with `qrels.verify(expected_topic_ids=...)`, and return the verified `Qrels` from `create_qrels()`. The workflow runner writes the returned qrels to `<filebase>.qrels` in standard TREC format, deterministically sorted ([What lands in the output directory](04-run-workflows.md#what-lands-in-the-output-directory)); a judge never serializes qrels itself, and `write_qrel_file(...)` remains for scripts outside the workflow.

The `doc_id` column names the judged unit, in one of two modes: a judge that grades **whole reports** uses the report's run tag (`Report.metadata.run_id`) as the doc id, so that qrels rows map one-to-one onto leaderboard cells (each topic has one report per run); a judge that grades **cited documents** keeps the corpus doc ids that the citations reference.

→ API: [Qrels guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/qrels/README.md) — building, verifying, and serializing TREC-format qrels · reference: [Qrels](https://trec-auto-judge.github.io/auto-judge-base/api/qrels/)

### Configuring hyperparameters

Anything you might want to vary — prompt style, number of nuggets, grading scale — belongs in `workflow.yml` `settings` (or the phase-specific `nugget_settings`/`judge_settings`), which arrive in your methods as `**kwargs`. Named `variants` then override settings per configuration, so one workflow file expresses your whole method family and [`--variant`](04-run-workflows.md) selects one member; the starter kit's `complete_example` workflow, with several variants and a couple of sweeps, shows the pattern.

→ API: [Workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md) — settings, variants, sweeps, custom nugget formats

## Learn from existing judges

| Example | What to learn from it |
|---------|----------------------|
| `judges/complete_example/` (starter kit) | the full three-protocol structure, modular classes, an exhaustively-commented `workflow.yml` with variants and sweeps |
| `judges/tinyjudge/` (starter kit) | the smallest realistic LLM judge: batched requests, prompt caching, clean `llm_config` handling |
| `judges/naive/` (starter kit) | a minimal non-LLM baseline: the least code needed to emit a valid leaderboard |

## Conventions that keep judges well-behaved

- **Read the endpoint from `llm_config`, never hardcode keys or URLs** — on TIRA the organizer injects the endpoint through this parameter ([details](02-configure-llm-endpoint.md)).
- **Expect empty reports.** A run may ship `responses: []` for a topic it skipped — score it (typically the bottom grade), never crash or drop the topic. The starter kit's test suite checks this.
- **Describe every measure.** Each `MeasureSpec` `description` (what it represents, its range, how to read it) exports to `measures.yml` alongside your leaderboard and documents your judge for organizers and downstream tooling.
- **Sort before you compare.** Order responses by `run_id` before building comparison pairs, so prompts — and therefore [prompt-cache](05-prompt-cache.md) keys — stay identical across runs.
- **Accept the injected output parameters.** All judge methods receive auto-filled `filebase: str = "default"` and `outdir: Path = Path(".")` for constructing output paths; declare them explicitly. Setting `filebase: "{_name}"` in `workflow.yml` names output files after the variant or sweep being run.
- **Verify before returning — always.** Call the verification the moment you have each artifact: `build(expected_topic_ids=...)` / `leaderboard.verify(...)` for the leaderboard, `qrels.verify(expected_topic_ids=...)` for qrels, and rely on the runner's nugget-bank verification (a topic with an empty bank raises `NuggetBanksVerificationError`). These checks turn the classic silent failure — a topic quietly dropped, an empty bank, a partially-completed run — into a loud error at the source, instead of a wrong-but-plausible leaderboard that costs everyone downstream far more grief to untangle. Pass `expected_topic_ids` explicitly so "missing" is actually detectable.
- **Preflight the endpoint — but warn, don't raise.** One probe LLM call at the start of each phase surfaces a stale model id or bad key as one clear provider error instead of hundreds of failed batch calls (see [endpoint troubleshooting](02-configure-llm-endpoint.md#troubleshooting)). Make it a *warning*: a judge with a populated [prompt cache](05-prompt-cache.md) must be able to complete with no working endpoint at all — TIRA's deterministic re-execution runs it exactly that way. Let the all-calls-failed condition be the hard error instead.

## References

- [auto-judge-base API](https://trec-auto-judge.github.io/auto-judge-base/) — generated reference for the data models, leaderboard, qrels, nuggets, and the `AutoJudge` protocol, rendered from the pydantic sources
- [auto-judge-base — Quick Start & data classes](https://github.com/trec-auto-judge/auto-judge-base) — `Report`, `Request`, `Leaderboard`, `NuggetBanks`, data-loading utilities
- [Leaderboard guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/leaderboard/README.md) · [Qrels guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/qrels/README.md) · [NuggetBank format](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/nugget_data/README.md) · [Workflow guide](https://github.com/trec-auto-judge/auto-judge-base/blob/main/src/autojudge_base/workflow/README.md)
- [minima-llm](https://github.com/trec-auto-judge/minima-llm) — batching, caching, DSPy adapter
