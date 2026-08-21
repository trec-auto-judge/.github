# 8. Run Collected AutoJudge Systems in TIRA

*Part of the [TREC AutoJudge HowTo](README.md). Previous: [Submit to TIRA](07-submit-to-tira.md).*

After we collected code submissions of AutoJudge systems in TIRA (e.g., a set of submissions that [participants submitted to TIRA](07-submit-to-tira.md)), we can run the collected AutoJudge systems on all datasets against some LLMs, potentially with some repetitions. This documentation here is mostly focused at Organizers or participants who want to run their code submissions against custom LLMs, because we expect that mostly organizers run the submitted AutoJudge systems in TIRA. Still, even when you not want to run collected AutoJudge systems in TIRA (e.g., as participant you [can stop after you submitted the software](07-submit-to-tira.md)) this documentation still is here to ensure that we make transparant how we then run all collected AutoJudge systems. We as organizers will mostly run the collected AutoJudge systems against two smaller LLMs (at the moment we think this will be `openai/gpt-oss-20b` and `Qwen/Qwen2.5-7B-Instruct`), potentially with some repetitions if allowed by our compute budget. If you want to run your code submission against your own custom LLM, then this documentation here describes how to do this.

Before running AutoJudge systems trough TIRA, please first get to know how [TIRA handles Credentials for LLMs via REST API](https://docs.tira.io/participants/llms-via-rest-api.html).

## Run an Collected AutoJudge System via the UI

After an AutoJudge system got submitted, it appears in the UI, and one can start it there, by passing the environment variables that this AutoJudge needs (the required environment variables got collected during the submission), this looks like this:

<img width="1705" height="1140" alt="forward-llm-environment-variables" src="https://github.com/user-attachments/assets/239801c2-b8f8-4885-8c5e-3617130e3efa" />


Please note that it is rather unconvenient to start many software executions via the UI, so we only do this for spot-checks, and after a software seems to work, we include its execution into CLI scripts.

## Run Collected AutoJudge Systems via the CLI


