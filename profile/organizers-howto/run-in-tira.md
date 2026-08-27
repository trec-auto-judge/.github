# Run Submitted Code in TIRA

*Part of the TREC AutoJudge organizers HowTo. The participant-facing steps end with [Submit to TIRA](../howto/07-submit-to-tira.md); participants who want to run their own submission against a custom LLM should follow [Run on TIRA with custom endpoint](../howto/08-run-on-tira-custom-endpoint.md).*

After we have collected code submissions of AutoJudge systems in TIRA (i.e., the submissions that [participants submitted to TIRA](../howto/07-submit-to-tira.md)), we can run the collected AutoJudge systems on all datasets against some LLMs, potentially with repetitions. This page is aimed at organizers; as a participant, you can [stop after you have submitted your software](../howto/07-submit-to-tira.md) — this page is here to make transparent how we then run all collected AutoJudge systems. We as organizers will mostly run the collected AutoJudge systems against two smaller LLMs (at the moment we expect `openai/gpt-oss-20b` and `Qwen/Qwen2.5-7B-Instruct`), potentially with repetitions if our compute budget allows.

Before running AutoJudge systems through TIRA, please first get to know how [TIRA handles credentials for LLMs via REST API](https://docs.tira.io/participants/llms-via-rest-api.html).

## Run Submitted Code via the UI

After an AutoJudge system has been submitted, it appears in the UI, where one can start it by passing the environment variables that it needs (the required environment variables were collected during submission). This looks like this:

<img width="1705" height="1140" alt="forward-llm-environment-variables" src="https://github.com/user-attachments/assets/239801c2-b8f8-4885-8c5e-3617130e3efa" />


Please note that it is rather inconvenient to start many software executions via the UI, so we only do this for spot-checks; once a software seems to work, we include its execution in CLI scripts.

## Run Submitted Code via the CLI

To run many collected software submissions against multiple LLMs, with potentially multiple repetitions on multiple datasets, we use the CLI.

On a high level, the command for this would be:

```
export OPENAI_API_KEY=...
export OPENAI_BASE_URL=...
export OPENAI_MODEL=...

tira-cli run remote \
	--approach APPROACH-1 APPROACH-2 ... APPROACH-N \
	--dataset DATASET-1 DATASET-2 DATASET-3 \
	--parallelism 4 \
	--forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL
```

This specifies the approaches that should be executed (i.e., APPROACH-1 to APPROACH-N are approaches that were submitted to TIRA), the datasets, and the parallelism (e.g., above, 4 submissions can run in parallel). The CLI call terminates after everything has been executed, while ensuring that no more than 4 softwares run at any moment (to not put too much load on a single LLM).

We collect all scripts and such commands in the private repository [https://github.com/trec-auto-judge/evaluation-in-progress](https://github.com/trec-auto-judge/evaluation-in-progress).
