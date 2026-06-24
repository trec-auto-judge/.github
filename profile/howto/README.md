# Submission Guidelines for AutoJudge at TREC 2026

**Attention: this is still in progress**

<details>
<summary>Prerequisite: Create an Account at TIRA.io and register a team to TREC AutoJudge in TIRA</summary>

### Step 1: Create an Account at TIRA

Please go to [https://www.tira.io/](https://www.tira.io/) and click on "Sign Up" to create a new account or "Log In" if you already have an account. You can either create an new account or Log in via GitHub.

<img width="1042" height="965" alt="Screenshot_20251210_074005" src="https://github.com/user-attachments/assets/6f05d18d-3b03-4314-94b4-b1136613b362" />

### Step 2: Register Your Team to TREC AutoJudge

After you have logged in to TIRA, please navigate to the RAG4Reports task at [https://www.tira.io/task-overview/trec-auto-judge](https://www.tira.io/task-overview/trec-auto-judge). There, please click on "Register".

<img width="1807" height="712" alt="Screenshot_20260205_075346" src="https://github.com/user-attachments/assets/00cf8fe5-02d1-4b48-aabf-bd5de51cc910" />


### Step 3 (Optional): Manage your team

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

<img width="1964" height="503" alt="Screenshot_20251210_095119" src="https://github.com/user-attachments/assets/12e55ed2-a670-473c-ac4d-748a169afefa" />

Assuming your authentication token is AUTH-TOKEN, please authenticate via:

```
tira-cli login --token AUTH-TOKEN
```

Lastly, to verify that everything is correct, please run `tira-cli verify-installation`. Outputs might look like:

<img width="821" height="180" alt="Screenshot_20251210_095410" src="https://github.com/user-attachments/assets/51160132-eb19-4da3-8892-8a53adb41c71" />

</details>
