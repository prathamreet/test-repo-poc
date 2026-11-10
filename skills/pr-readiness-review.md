# Skill: PR Failure Triage & Cloud Diagnosis
# Target Agent: AWS DevOps Agent

name: pr-failure-triage
description: Standard operating procedure for triaging cloud, Terraform, CDK, and deployment failures on Pull Requests.

## Instructions
When a PR failure incident is received:

1. **Review Context & Metadata:**
   - Extract the PR number, commit SHA, and failed workflow run URL.
   - Inspect the git diff between the PR branch and `main`.

2. **Diagnose Cloud Infrastructure Issues:**
   - If CloudFormation / CDK failed: check CloudFormation stack events for rollback reasons, resource conflict exceptions, or invalid template references.
   - If Terraform plan/apply failed: identify the provider error (e.g. AWS API error, missing argument, state lock).
   - If container/ECS failed: check container exit codes, memory/CPU limits, or image pull secrets.
   - If IAM error: identify the missing action (e.g., `s3:GetObject`, `kms:Decrypt`) and affected resource ARN.

3. **Output Format (PR Comment):**
   Structure the response cleanly for the developer:
   - **Summary:** 1-line root-cause description.
   - **Root Cause & Evidence:** The exact AWS error or log snippet.
   - **Actionable Suggestion:** 1-2 concrete steps to fix the configuration in the PR.

