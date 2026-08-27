# 8. Run on TIRA with custom endpoint

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Submit to TIRA](07-submit-to-tira.md).*

After the [code submission](07-submit-to-tira.md) uploads, your judge lives in TIRA as a *software* — and you can execute it **remotely on TIRA's workers**, on any dataset, against an LLM endpoint of your choice. This is optional: we organizers run all submitted code ourselves (documented in [Run Submitted Code in TIRA](../organizers-howto/run-in-tira.md)). Run it yourself when you want to verify that your submission behaves in TIRA exactly as it did locally, or to produce results with a custom LLM.

## Step 1 — Identify your submission's run-tag

Every code submission gets a **run-tag** — the name under which it is addressable in TIRA. The upload prints it on its final line:

```
✓ Your code submission is available in TIRA as $run-tag.
```

This page writes the name as `$run-tag`; yours is currently auto-generated (soon `tira-cli code-submission` gains an optional `--run-tag XXX` to choose it yourself). You can always look it up in the TIRA UI: task page → *submit → Code Submission* lists each submitted software by its run-tag.

The full **approach id** used below is `<task>/<team>/<run-tag>`, e.g.:

```
trec-auto-judge/<your-team>/$run-tag
```

## Step 2 — Set the environment variables

The remote run needs your judge's LLM environment. Export the variables locally; here `--forward-environment-variable` ships their *values* to TIRA, which injects them into the running container. (Note the contrast to the same flag on `tira-cli code-submission`, which transmits only the variable *names* — see [Submit to TIRA](07-submit-to-tira.md#details-to-know).)

```bash
export OPENAI_API_KEY=...
export OPENAI_BASE_URL=...
export OPENAI_MODEL=...
```

> **Warning — your API key is shared with TIRA.** Forwarded values are transmitted from your shell into the container that runs your code on TIRA. TIRA does not persist them in long-term storage, but they pass through its job queue on the way to the worker (short-term storage) — your key travels through and is used on infrastructure outside your control. Do not forward your main unlimited key: use a separate key with a **capped usage limit** (every major provider supports per-key spending limits), and revoke it when you are done.

Two more constraints on the endpoint:

- **`OPENAI_BASE_URL` is dereferenced on TIRA's workers, not on your machine.** A `localhost` or institution-internal endpoint will not be reachable. Use a publicly reachable endpoint, or one of the LLM endpoints TIRA itself provides — see [TIRA's LLMs-via-REST-API documentation](https://docs.tira.io/participants/llms-via-rest-api.html).
- Use the same variable names your judge reads (for the starter kit: `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_MODEL` — see [Configure your LLM endpoint](02-configure-llm-endpoint.md)).
- **Only variables declared at submission time can be forwarded.** The `--forward-environment-variable` list of the original `tira-cli code-submission` is recorded with the software; if the submission was made without it, the remote run silently ignores your endpoint variables — resubmit with the flag ([Submit to TIRA](07-submit-to-tira.md)) and use the new run-tag.

## Step 3 — Issue the `run remote` call

Spot-check on the synthetic kiddie dataset first:

```bash
tira-cli run remote \
    --approach trec-auto-judge/<your-team>/$run-tag \
    --dataset kiddie-20260605-training \
    --parallelism 1 \
    --mount-cache '$CACHE_DIR=EMPTY_DIR' \
    --require "evaluation.Model==$OPENAI_MODEL" \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL
```

The call starts the execution on TIRA and polls until it completes; results appear under your team on the task page. Notes:

- **`--mount-cache '$CACHE_DIR=EMPTY_DIR'`** — keep the single quotes (nothing must expand in your local shell). Remote workers cannot see your local `./cache`, so the cache starts empty; to replay from a warm cache, mount a *previous run's* `run_id` instead of `EMPTY_DIR` (see `tira-cli run remote --help`).
- **Dataset names** are TIRA dataset ids — the `tira_id` values in the starter kit's `datasets.yml`.
- Both `--approach` and `--dataset` accept multiple values and run the Cartesian product; `--parallelism` bounds how many executions run at once (keep it low to not overload the LLM endpoint).
- **`--runs-per-approach N`** repeats each combination until N runs exist — for measuring run-to-run variance.
- **`--require "evaluation.Model==$OPENAI_MODEL"`** skips combinations that already have a run with a matching model. Note the **double** quotes — here the local expansion is deliberate: your shell substitutes `$OPENAI_MODEL` from your environment, so re-invoking the command only executes what is missing for the model you currently have configured.

When the spot check passes, point the same command at the real datasets, e.g. `--dataset rag26-20260827_1-test ragtime26-20260827-test`.

## Alternative — start a run via the UI

A single run can also be started from the TIRA UI: after submission, your software appears on the task page, where you can start it by filling in the environment variables it needs (the variable *names* declared at submission — you type the values into the form, so the same [key warning](#step-2--set-the-environment-variables) applies):

<img width="1705" height="1140" alt="forward-llm-environment-variables" src="https://github.com/user-attachments/assets/239801c2-b8f8-4885-8c5e-3617130e3efa" />

Starting many executions via the UI is inconvenient — use it for a one-off spot-check, and the CLI call above for anything repeated.

## References

- [Submit to TIRA](07-submit-to-tira.md) — the code submission this page builds on
- [Run Submitted Code in TIRA](../organizers-howto/run-in-tira.md) — how we organizers batch-run all collected submissions
- [Configure your LLM endpoint](02-configure-llm-endpoint.md) — the environment variables your judge reads
- [Prompt cache](05-prompt-cache.md) — what the cache mount does
- [TIRA: LLMs via REST API](https://docs.tira.io/participants/llms-via-rest-api.html) — TIRA-provided LLM endpoints and credential handling
