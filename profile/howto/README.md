# Submission Guidelines for AutoJudge at TREC 2026

**Attention: this is still in progress**

<details>
<summary>Prerequisite: Create an Account at TIRA.io and register a team to TREC AutoJudge in TIRA</summary>

### Step 1: Create an Account at TIRA

Please go to [https://www.tira.io/](https://www.tira.io/) and click on "Sign Up" to create a new account or "Log In" if you already have an account. You can either create an new account or Log in via GitHub.

<img width="1042" height="965" alt="Screenshot_20251210_074005" src="https://github.com/user-attachments/assets/6f05d18d-3b03-4314-94b4-b1136613b362" />

### Step 2: Register Your Team to TREC AutoJudge

After you have logged in to TIRA, please navigate to the RAG4Reports task at [https://www.tira.io/task-overview/trec-auto-judge](https://www.tira.io/task-overview/trec-auto-judge). There, please click on "Register".

<img width="1801" height="750" alt="Screenshot_20260624_211840" src="https://github.com/user-attachments/assets/b058b8a7-88d4-450c-a63b-303b1c5a27e0" />


### Step 3: Private Team Chat to Discuss Technical Aspects

After registration, [Maik](https://www.tira.io/u/maik_froebe) will start a private chat with you in TIRA (by default, the xy to mail) with Laura on cc. We will use this chat to xy.


### Step 4 (Optional): Manage your team

If you want to add others to your team, please navigate to your groups (under the hamburger menu) at [https://www.tira.io/g?type=my](https://www.tira.io/g?type=my)

</details>

## Preferred: Code Submissions

To ensure that we can run XYZ...


<details>
<summary>Step 1: Ensure that your approach works on your machine</summary>

</details>



<details>
<summary>Step 3: Authentication and Login</summary>

We assume you have created an account at TIRA.io and have registered a team to TREC AutoJudge task following the prerequisite above.

The preferred way to submit to TREC AutoJudge is via Code Submissions, as this allows to xyz.

Please install the TIRA cli via:

```
pip3 install --upgrade tira
```

Next, you need an authentication token:

- Navigate to the Rag4Reports task in TIRA [https://www.tira.io/task-overview/trec-auto-judge](https://www.tira.io/task-overview/trec-auto-judge)
- Click on "submit" => "Code Submission" => "I want to submit from my local machine" => Next. The UI shows your authentication token:

<img width="1808" height="985" alt="Screenshot_20260624_212656" src="https://github.com/user-attachments/assets/995cbd0e-1eae-4a70-a13b-acbf1d2229dc" />


Assuming your authentication token is AUTH-TOKEN, please authenticate via:

```
tira-cli login --token AUTH-TOKEN
```

Lastly, to verify that everything is correct, please run `tira-cli verify-installation`. Outputs might look like:

<img width="821" height="180" alt="Screenshot_20251210_095410" src="https://github.com/user-attachments/assets/51160132-eb19-4da3-8892-8a53adb41c71" />

</details>
