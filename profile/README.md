## Auto-Judge👋 Infrastructure

* [Info about TREC Auto-Judge](https://trec-auto-judge.cs.unh.edu/): What is this about?
* [Data Set](https://trec-auto-judge.cs.unh.edu/datareleases/): Grab the pilot data to participate!
* [Starter Kit](https://github.com/trec-auto-judge/auto-judge-starter-kit)  Start developing your auto-judge!
* [Participant HowTo](https://github.com/trec-auto-judge/.github/blob/main/profile/howto/README.md): The canonical step-by-step guide, one page per activity — [setup](https://github.com/trec-auto-judge/.github/blob/main/profile/howto/01-setup-environment.md), [LLM endpoint](https://github.com/trec-auto-judge/.github/blob/main/profile/howto/02-configure-llm-endpoint.md), [develop an autojudge](https://github.com/trec-auto-judge/.github/blob/main/profile/howto/03-develop-an-autojudge.md), [running](https://github.com/trec-auto-judge/.github/blob/main/profile/howto/04-run-workflows.md), [prompt cache](https://github.com/trec-auto-judge/.github/blob/main/profile/howto/05-prompt-cache.md), [meta-evaluation](https://github.com/trec-auto-judge/.github/blob/main/profile/howto/06-meta-evaluation.md), and [submission](https://github.com/trec-auto-judge/.github/blob/main/profile/howto/07-submit-to-tira.md).
* [Meta-evaluation](https://github.com/trec-auto-judge/auto-judge-evaluate) For local evaluation and development.

![Auto-Judge](examiner-logo-animated.gif)


# Example AutoJudges

Please add additional examples.

* Starter auto-judges include a [naive judge without dependencies](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/naive), a [minimal LLM-baed example](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/tinyjudge), and [a complete LLM example](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/complete_example).
* [PrefNugget-based auto-judges](https://github.com/laura-dietz/prefnugget-starterkit)
* [LangChain preference-nugget auto-judge](https://github.com/laura-dietz/langchain-starterkit) — brings its own LLM client (LangChain), with an endpoint-agnostic prompt cache and on-demand preference-pair growth
* [IR Axioms inspired auto-judges](https://github.com/webis-de/ir_axioms/tree/main/trec-auto-judge)
* [Pyterrier retrieval auto-judges](https://github.com/webis-de/trec26-auto-judge/tree/main/judges/pyterrier_retrieval)
* [AutoNuggetizer-based auto-judge](https://github.com/webis-de/trec26-auto-judge/tree/main/judges/auto_nuggetizer)


