# Submission Guidelines for AutoJudge at TREC 2026

<details>
<summary>Prerequisite: Create an Account at TIRA.io and register a team to TREC AutoJudge in TIRA</summary>

### Step 1: Create an Account at TIRA

Please go to [https://www.tira.io/](https://www.tira.io/) and click on "Sign Up" to create a new account or "Log In" if you already have an account. You can either create an new account or Log in via GitHub.

<img width="1042" height="965" alt="Screenshot_20251210_074005" src="https://github.com/user-attachments/assets/6f05d18d-3b03-4314-94b4-b1136613b362" />

### Step 2: Register Your Team to TREC AutoJudge

After you have logged in to TIRA, please navigate to the TREC AutoJudge task at [https://www.tira.io/task-overview/trec-auto-judge](https://www.tira.io/task-overview/trec-auto-judge). There, please click on "Register".

<img width="1801" height="750" alt="Screenshot_20260624_211840" src="https://github.com/user-attachments/assets/b058b8a7-88d4-450c-a63b-303b1c5a27e0" />


### Step 3: Private Team Chat to Discuss Technical Aspects

After registration, [Maik](https://www.tira.io/u/maik_froebe) will start a private chat with you in TIRA (by default, messages will be forwarded to your e-mail) with [Laura](https://www.tira.io/u/laura-dietz) on cc. We will use this chat to help you with the submission process (for instance, if you do not have docker installed or do not want to use Github actions to submit, we can discuss how we can access the code to make the submission to TIRA).


### Step 4 (Optional): Manage your team

If you want to add others to your team, please navigate to your groups (under the hamburger menu) at [https://www.tira.io/g?type=my](https://www.tira.io/g?type=my)

</details>

## Auto Judge Submission to TIRA


<details>
<summary>Step 0: Requirements</summary>

Our requirements to a code submission (we can help to meet them) are:

- All code must be organized in a git repository (the repository can be private, it can be only on your local machine, we do not check that all changes are pushed)
- The repository must be clean (i.e., git status indicates no uncommitted chages, please use `.gitignore` to ensure that frequently changing files do not make problems). You can check this with `git status --porcelain`: if it prints nothing, the repository is clean; any output indicates uncommitted or untracked changes that you need to commit or add to `.gitignore`.
- When calling `tira-cli` the **currently checked-out git branch** is packaged and submitted to TIRA. (Check the current branch with `git branch --show-current`). You must *commit* all code first.
- We expect your test suite to pass: running `pytest` should complete without failures before you submit.
- The repository must contain a Dockerfile that specifies how the software is dockerized (the produced docker image is uploaded to TIRA, you can ensure that your Dockerfile is compatible with [dev-containers](https://containers.dev/), so that you can directly develop in the container)
- Your judge runs in a **sandbox without internet  access**. The LLM endpoint and model are injected at run time via environment variables (e.g. `OPENAI_BASE_URL`, `OPENAI_MODEL`, `OPENAI_API_KEY`) that you forward into the submission with `--forward-environment-variable`. Do not rely on any other internet access from within your judge.
- **Never include secrets into the Docker image.** Do not `COPY`/`ADD` API keys or tokens into the image; pass them only via `--forward-environment-variable` so they are injected at run time.

When those requirements are met, you can submit your auto-judge to TIRA. For this, you call `tira-cli code-submission ...` which will perform the following steps:

- The code is compiled into the Docker image as specified by the Dockerfile
- This image is tested on the local machine on the kiddie dataset to ensure that the software produces valid outputs
- If the outputs are valid, the docker image is uploaded to TIRA
- Within TIRA, we/you run the docker image on all datasets, potentially with multiple LLMs
</details>



<details>
<summary>Step 1: Install the tira-cli on your machine and ensure that our examples work</summary>

Please install Docker (or podman) on your machine, as well as the TIRA cli:

```
pip3 install --upgrade tira
```

Note that the Docker (or podman) daemon must be **running** when you submit — `tira-cli code-submission` builds and tests the image locally before uploading, so an installed-but-stopped daemon will cause the submission to fail.

**Podman users:** podman requires a container-signature policy file that some installations are missing. If the build fails at the first `FROM` step with `no policy.json file found`, create a permissive default (no root needed):

```
mkdir -p ~/.config/containers
printf '{\n  "default": [{"type": "insecureAcceptAnything"}]\n}\n' > ~/.config/containers/policy.json
```

(This file is host/user configuration for podman only; Docker does not use it.)

We have prepared a set of hello world examples in the [AutoJudge Starter kit](https://github.com/trec-auto-judge/auto-judge-starter-kit) that we recommend you to run on your machine.

- A naive auto judge system that shows the complete process without dependencies to an LLM: [https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/naive](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/naive#submit-to-tira)
- A tinyjudge system that uses an LLM and employs caching: [https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/tinyjudge](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/tinyjudge#submit-to-tira)

If you have familarized yourself with those examples and the approaches work on your machine with the described `tira-cli` command, everything should be fine. (if there is a problem, please do not hesitate to contact us, for instance, in the private chat that we initiated after your registration.)

</details>




<details>
<summary>Step 2: Ensure that your approach works on your machine</summary>

Please get your approach running in the `auto-judge run` framework. For starter-kit-based judges we recommend installing the full toolchain with `uv pip install -e '.[all]'` — always use `.[all]` to be sure you have everything needed to test, submit, and evaluate. If you are still setting up your environment, you can start coding right away with the lightweight `uv pip install -e .` and switch to `.[all]` once you are ready.

When this works, you can embedd your approach in a similar `tira-cli` command as the [naive](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/naive) or [tiny](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/tinyjudge) judges.

</details>




<details>
<summary>Step 3: Authentication and Login</summary>

We assume you have created an account at TIRA.io and have registered a team to TREC AutoJudge task following the prerequisite above.

The preferred way to submit to TREC AutoJudge is via Code Submissions, as this allows us to execute all judge system on new datasets with more LLMs.


Next, you need an authentication token:

- Navigate to the TREC AutoJudge task in TIRA [https://www.tira.io/task-overview/trec-auto-judge](https://www.tira.io/task-overview/trec-auto-judge)
- Click on "submit" => "Code Submission" => New Submission => "I want to submit from my local machine" => Next => Next. The UI shows your authentication token:

<img width="1808" height="985" alt="Screenshot_20260624_212656" src="https://github.com/user-attachments/assets/995cbd0e-1eae-4a70-a13b-acbf1d2229dc" />


Assuming your authentication token is AUTH-TOKEN, please authenticate via:

```
tira-cli login --token AUTH-TOKEN
```

Lastly, to verify that everything is correct, please run `tira-cli verify-installation`. Outputs might look like:

<img width="821" height="180" alt="Screenshot_20251210_095410" src="https://github.com/user-attachments/assets/51160132-eb19-4da3-8892-8a53adb41c71" />
</details>


<details>
<summary>Step 4: Submit your AutoJudges</summary>

This is basically as Step 1, but you need to remove the `--dry-run` flag of the `tira-cli` command. For instance, to submit the [tinyjudge](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/tinyjudge#submit-to-tira), the command would be:


```bash
tira-cli code-submission \
    --path . \
    --cache-behaviour deterministic \
    --mount-cache '$CACHE_DIR=EMPTY_DIR' \
    --task trec-auto-judge \
    --dataset kiddie-20260605-training \
    --forward-environment-variable OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL \
    --command 'auto-judge run --workflow /auto-judge/judges/tinyjudge/workflow.yml --rag-responses $inputDataset/runs/*/ --rag-topics $inputDataset/topics/*.jsonl --out-dir $outputDir'
```

</details>
