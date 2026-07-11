# TREC AutoJudge — Participant HowTo

Welcome! Participating in TREC AutoJudge means building an automatic judge for RAG responses and submitting it to TIRA, and this guide walks you through the whole journey — from an empty machine to a completed submission — one activity at a time:

| # | Activity | You will... |
|---|----------|-------------|
| 1 | [Set up your dev environment](setup-environment.md) | fork the starter kit, install with `uv`, verify on the kiddie dataset |
| 2 | [Configure your LLM endpoint](configure-llm-endpoint.md) | pick an LLM client (any OpenAI-compatible one), configure it locally and for TIRA |
| 3 | [Developing practices](developing-practices.md) | implement the `AutoJudge` protocol and follow the conventions that keep judges reproducible |
| 4 | [Run workflows](run-workflows.md) | run your judge with `auto-judge run`, use variants, dev flags, and multi-dataset runs |
| 5 | [Prompt cache](prompt-cache.md) | make repeated LLM runs instant and deterministic; debug cache misses |
| 6 | [Meta-evaluation](meta-evaluation.md) | measure how well your judge agrees with ground truth |
| 7 | [Submit to TIRA](submit-to-tira.md) | dry-run and upload your judge as a code submission |

Newcomers should read 1 → 7 in order; returning participants can jump straight to the activity they need. In [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the starter kit provides two interactive walkthroughs: `/autojudge-setup` (activities 1–4) and `/autojudge-submit` (activity 7).

This guide is the **canonical documentation** for participating — the [starter kit](https://github.com/trec-auto-judge/auto-judge-starter-kit) README and skills intentionally defer to it. Deep reference material stays with the libraries it documents ([auto-judge-base](https://github.com/trec-auto-judge/auto-judge-base), [minima-llm](https://github.com/trec-auto-judge/minima-llm), [auto-judge-evaluate](https://github.com/trec-auto-judge/auto-judge-evaluate)); each activity page links to the relevant sections.

## Prerequisites

Before the activities above, register with TIRA once:

### 1. Create an account at TIRA

Go to [tira.io](https://www.tira.io/) and click *Sign Up* to create an account (or *Log In* — GitHub login works too).

<img width="1042" height="965" alt="TIRA sign-up page" src="https://github.com/user-attachments/assets/6f05d18d-3b03-4314-94b4-b1136613b362" />

### 2. Register your team to TREC AutoJudge

Navigate to the [TREC AutoJudge task](https://www.tira.io/task-overview/trec-auto-judge) and click *Register*.

<img width="1801" height="750" alt="TIRA task page with the Register button" src="https://github.com/user-attachments/assets/b058b8a7-88d4-450c-a63b-303b1c5a27e0" />

### 3. Private team chat

After registration, [Maik](https://www.tira.io/u/maik_froebe) starts a private chat with you in TIRA (messages are forwarded to your e-mail by default) with [Laura](https://www.tira.io/u/laura-dietz) on cc. Use this chat for any technical questions — for instance if you cannot run Docker locally or want help with the submission process.

### 4. Manage your team (optional)

Add teammates via your groups page at [tira.io/g?type=my](https://www.tira.io/g?type=my) (under the hamburger menu).
