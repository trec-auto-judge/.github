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
- The repository must be clean (i.e., git status indicates no uncommitted chages, please use `.gitignore` to ensure that frequently changing files do not make problems)
- Optional and recommended: The repository is compatible with [dev-containers](https://containers.dev/)

As soon as those requirements are met, a code submission to TIRA performs the following steps:

- The code is compiled into the Docker image as specified by the dev-container
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

We have prepared a set of hello world examples in the [AutoJudge Starter kit](https://github.com/trec-auto-judge/auto-judge-starter-kit) that we recommend you to run on your machine.

- A naive auto judge system that shows the complete process without dependencies to an LLM: [https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/naive](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/naive)
- A tinyjudge system that uses an LLM and employs caching: [https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/tinyjudge](https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/tinyjudge)

If you have familarized yourself with those examples and the approaches work on your machine with the described `tira-cli` command, everything should be fine. (if there is a problem, please do not hesitate to contact us, for instance, in the private chat that we initiated after your registration.)

</details>




<details>
<summary>Step 2: Ensure that your approach works on your machine</summary>

Please get your approach running in the `auto-judge run` framework. When this works, you can embedd your approach in a similar `tira-cli` command as the [https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/naive](naive) or [https://github.com/trec-auto-judge/auto-judge-starter-kit/tree/main/judges/tinyjudge](tiny) judges.

</details>




<details>
<summary>Step 3: Authentication and Login</summary>

We assume you have created an account at TIRA.io and have registered a team to TREC AutoJudge task following the prerequisite above.

The preferred way to submit to TREC AutoJudge is via Code Submissions, as this allows us to execute all judge system on new datasets with more LLMs.


Next, you need an authentication token:

- Navigate to the TREC AutoJudge task in TIRA [https://www.tira.io/task-overview/trec-auto-judge](https://www.tira.io/task-overview/trec-auto-judge)
- Click on "submit" => "Code Submission" => "I want to submit from my local machine" => Next. The UI shows your authentication token:

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

This is basically as Step 1, but you need to remove the `--dry-run` flag of the `tira-cli` command.

</details>
