# End-to-End Setup Guide: AWS DevOps Agent & GitHub Integration

This step-by-step guide walks you through setting up the complete POC: from provisioning the **AWS DevOps Agent Space**, connecting the **GitHub App**, configuring the **generic webhook**, to verifying the **PR failure triage** loop.

---

## Architecture Flow

```
1. Developer opens PR to 'main'
2. GitHub Action runs Job 1: Lint & Unit Tests
   ├─ If FAILED ──► Workflow stops. ($0 spent on Agent)
   └─ If PASSED ──► Runs Job 2: Cloud Validation (Terraform/CDK/Deploy)
                     ├─ If PASSED ──► PR is Green.
                     └─ If FAILED ──► Runs Job 3: DevOps Agent Triage
                                       │
                                       ▼ (Webhook)
                               AWS DevOps Agent Space
                                       │
                    ┌──────────────────┴──────────────────┐
                    ▼                                     ▼
        [Path A: Native Comment]              [Path B: Lambda Fallback]
        Agent comments directly               EventBridge -> Lambda ->
        onto the GitHub PR                    ListJournalRecords -> PR Comment
```

---

## Phase 1: Create the AWS DevOps Agent Space

1. Log in to the **AWS Management Console**.
2. Search for and open **AWS DevOps Agent** (or navigate via Systems Manager / CloudWatch).
3. Choose **Create Agent Space**.
4. Configure the details:
   * **Agent Space Name:** `pr-triage-agent-space` (or your company naming standard).
   * **Target AWS Region:** Choose your primary workload region (e.g., `us-east-1` or `eu-west-1`).
   * **Execution Timeout:** Set to **8 or 10 minutes** (Cost guardrail: prevents any single run from exceeding ~$4.00).
5. Under **Observability & IAM Access**:
   * Grant the Agent Space read-only permissions to Amazon CloudWatch, CloudTrail, and your target workloads (ECS, EKS, RDS, CloudFormation).
6. Click **Create Agent Space**.

---

## Phase 2: Connect GitHub App to the Agent Space

1. In your new Agent Space, navigate to the **Integrations** or **Capabilities** tab.
2. Under **Source Control**, click **Add Integration** $\rightarrow$ select **GitHub**.
3. Click **Install GitHub App** (you will be redirected to GitHub).
4. Select your **Organization** and choose **Only select repositories**:
   * Select your target repository (e.g., `devops-agent-poc`).
5. Verify permissions:
   * **Repository Contents:** Read-only (reads code structure & configs).
   * **Commit History & Pull Requests:** Read & Write (Read for git diff analysis; Write for native PR comments).
   * **Actions / Workflows:** Read-only (checks build logs).
6. Click **Install & Authorize**.
7. Return to the AWS Console and verify the status shows **Connected (Healthy)**.

---

## Phase 3: Create the Generic Webhook in AWS

1. Inside your Agent Space, open **Capabilities** $\rightarrow$ **Webhooks**.
2. Click **Create Webhook**.
3. Set the webhook properties:
   * **Webhook Name:** `github-pr-failure-webhook`
   * **Authentication Type:** Select **Bearer Token** (or Secret Key).
4. Click **Create**.
5. **IMPORTANT:** Copy and safely store the two values displayed:
   * **Webhook Endpoint URL:** e.g., `https://<region>.api.aws/webhook/generic/<WEBHOOK_ID>`
   * **Webhook Secret / Token:** e.g., `sec_xxxxxxxxxxxxxxxxxxxx`

---

## Phase 4: Configure GitHub Repository Secrets

1. In your GitHub repository, open **Settings** $\rightarrow$ **Secrets and variables** $\rightarrow$ **Actions**.
2. Click **New repository secret** and add:
   * **Name:** `DEVOPS_AGENT_WEBHOOK_URL`
   * **Value:** *(Paste your Webhook URL from Phase 3)*
3. Click **New repository secret** again and add:
   * **Name:** `DEVOPS_AGENT_WEBHOOK_SECRET`
   * **Value:** *(Paste your Webhook Secret token from Phase 3)*

---

## Phase 5: Upload Custom Skills (Runbooks)

1. In the AWS DevOps Agent console, open **Custom Skills** $\rightarrow$ **Create Skill**.
2. **Skill Name:** `pr-failure-triage`
3. Copy the instructions from [`skills/pr-readiness-review.md`](file:///e:/thrash/devops-agent-poc/skills/pr-readiness-review.md) in your workspace:
   * Instructs the agent to check CloudFormation rollbacks, container exit codes, IAM missing permissions, and git diffs.
   * Enforces that the agent formats the output into: **Summary**, **Root Cause & Evidence**, and **Actionable Suggestion**.
4. Click **Save Skill**.

---

## Phase 6: Enable the PR CI/CD Workflow in GitHub

1. The workflow is already created in your repo at:
   [`.github/workflows/pr-ci-cd.yml`](file:///e:/thrash/devops-agent-poc/.github/workflows/pr-ci-cd.yml)
2. Commit and push this file to your default branch (`main`):
   ```bash
   git add .github/workflows/pr-ci-cd.yml
   git commit -m "ci: add PR failure triage workflow with DevOps Agent"
   git push origin main
   ```

---

## Phase 7: Setup Lambda Fallback (Guaranteed PR Comments)

If your Agent Space blocks autonomous direct writes (due to AWS's `approval-ref` guardrail), this Lambda guarantees the comment is posted.

1. **Create Lambda Function:**
   * Go to AWS Lambda Console $\rightarrow$ **Create Function**.
   * Function Name: `devops-agent-pr-commenter`
   * Runtime: **Python 3.12**.
2. **Deploy Code:**
   * Copy the code from [`lambda/pr_comment_handler.py`](file:///e:/thrash/devops-agent-poc/lambda/pr_comment_handler.py) and paste it into the Lambda editor.
3. **Set Environment Variables:**
   * `GITHUB_TOKEN`: A GitHub Personal Access Token (PAT) or GitHub App token with `repo` / `pull_requests:write` permission.
   * `GITHUB_REPO_OWNER`: Your GitHub username or organization name.
   * `GITHUB_REPO_NAME`: Your repository name.
4. **Attach IAM Permissions:**
   * Under Configuration $\rightarrow$ Permissions, attach an inline policy to the execution role:
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": [
             "aidevops:ListJournalRecords",
             "aidevops:GetInvestigation"
           ],
           "Resource": "*"
         }
       ]
     }
     ```
5. **Create EventBridge Rule:**
   * Go to Amazon EventBridge $\rightarrow$ **Rules** $\rightarrow$ **Create Rule**.
   * Rule Name: `devops-agent-pr-investigation-completed`
   * Event Bus: `default`
   * Event Pattern:
     ```json
     {
       "source": ["aws.aidevops"],
       "detail-type": ["Investigation Completed", "Investigation Failed"]
     }
     ```
   * Target: **Lambda Function** $\rightarrow$ `devops-agent-pr-commenter`.

---

## Phase 8: Verification & Testing

### Test 1: Verify Lint/Test Failures Are Ignored ($0 Spend)
1. Create a branch: `git checkout -b test/lint-fail`.
2. Add a syntax error or failing unit test in your code.
3. Push and open a PR targeting `main`.
4. **Expected Result:**
   * Job 1 (`lint-and-unit-tests`) **FAILS**.
   * Job 2 (`cloud-validation`) is **SKIPPED**.
   * Job 3 (`devops-agent-triage`) is **SKIPPED**.
   * **AWS DevOps Agent is NOT called. $0 billed.**

### Test 2: Verify Cloud Failures Trigger the Agent
1. Create a branch: `git checkout -b test/cloud-fail`.
2. Keep lint and tests passing.
3. In [`.github/workflows/pr-ci-cd.yml`](file:///e:/thrash/devops-agent-poc/.github/workflows/pr-ci-cd.yml), uncomment `exit 1` in Job 2 to simulate a failed cloud deployment.
4. Push and open a PR targeting `main`.
5. **Expected Result:**
   * Job 1 (`lint-and-unit-tests`) **PASSES**.
   * Job 2 (`cloud-validation`) **FAILS**.
   * Job 3 (`devops-agent-triage`) **EXECUTES**.
   * The webhook fires to AWS.
   * Within 2–3 minutes, an automated comment from AWS DevOps Agent appears on the PR detailing the root cause and suggestions!

