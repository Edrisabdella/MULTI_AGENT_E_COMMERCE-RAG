# NovaMart Multi-Agent Customer Support Project — Complete Submission Guide

This guide is based on the supplied 30-page project specification/rubric and the `SUBMISSION_GUIDE.md`, `README.md`, `.env.example`, and `VALIDATION_REPORT.md` included in the prepared project ZIP.

**Important status:** The prepared ZIP is not yet proof of a completed AWS deployment or a 120/120 result. You must run the commands in your own environment, create/synchronize the AWS Knowledge Bases, deploy the runtime, pass the tests, and capture the required screenshots before submitting.

---

## 0. What you need before starting

- An AWS account with permission to create/use Bedrock, Bedrock Knowledge Bases, S3, S3 Vectors, DynamoDB, CloudFormation, IAM roles, AgentCore Runtime, AgentCore Memory, CloudWatch, and X-Ray resources.
- AWS region: **`us-east-1`**. Keep the console, AWS CLI, and project configuration in this region.
- A Windows computer with VS Code/PowerShell, or the supplied Udacity workspace.
- Python **3.12+**.
- AWS CLI configured with credentials for your AWS account.
- Node.js **20+**, `uv`, and AgentCore CLI **0.30.0** for runtime deployment.
- Internet access to install Python/npm dependencies and to allow the deployment tooling to download runtime dependencies.

Do not share AWS access keys, secret keys, session tokens, `.env` contents, or account credentials in chat or screenshots.

## 1. Download and extract the prepared project

1. Download `NovaMart_Multi_Agent_Customer_Support_Submission.zip` from the ChatGPT message where it was provided.
2. In Windows File Explorer, extract the ZIP. The archive contains a top-level folder named `starter`.
3. Open the extracted **`starter`** folder in VS Code. It must be the folder containing `config.py`, `requirements.txt`, `src`, `tests`, and `infrastructure`.

### PowerShell extraction example

If the ZIP is in your Downloads folder:

```powershell
$zip = "$HOME\Downloads\NovaMart_Multi_Agent_Customer_Support_Submission.zip"
$dest = "$HOME\NovaMart_Submission"
Expand-Archive -LiteralPath $zip -DestinationPath $dest -Force
Set-Location "$dest\starter"
Get-ChildItem
```

The listing should include `config.py`, `requirements.txt`, `src`, `tests`, `infrastructure`, and `agentcore`.

**Do not run project commands from the parent folder.** Run them from the extracted `starter` project root.

## 2. Install and verify local tools (Windows PowerShell)

Open a new PowerShell terminal in VS Code.

### 2.1 Check Python

```powershell
py -3.12 --version
python --version
```

Use Python 3.12 or newer. If `py -3.12` is not found, install Python 3.12 and reopen the terminal.

### 2.2 Create and activate a virtual environment

Run from the `starter` folder:

```powershell
py -3.14.7 -m venv .venv
.\.venv\Scripts\Activate.ps1
python --version
python -m pip install --upgrade pip
```

If PowerShell blocks activation, run this for the current terminal only, then activate again:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

### 2.3 Install Python dependencies

```powershell
python -m pip install -r requirements.txt
```

Confirm the main packages import:

```powershell
python -c "import boto3, strands, dotenv; print('Python dependencies import successfully')"
```

If installation fails, read the first package/error message, confirm internet access and Python version, and retry. Do not continue to the grading tests until dependencies install successfully.

### 2.4 Install Node.js 20+, uv, and AgentCore CLI

Check whether Node and npm are installed:

```powershell
node --version
npm --version
```

The Node version must be 20 or newer. If not installed, install the current Node.js LTS release, reopen PowerShell, and check again.

Install `uv` using the official Windows installer command:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Close and reopen PowerShell, then verify:

```powershell
uv --version
```

Install the project-specified AgentCore CLI version:

```powershell
npm install -g @aws/agentcore@0.30.0
agentcore --version
```

The version should report `0.30.0`. The project README warns that the old Python `bedrock-agentcore-starter-toolkit` also installs an `agentcore` command. If that legacy package is installed, remove it from the active Python environment:

```powershell
python -m pip uninstall bedrock-agentcore-starter-toolkit
```

Then verify the npm CLI again:

```powershell
agentcore --version
agentcore validate
```

If `agentcore` is not recognized after npm installation, reopen PowerShell and check that the npm global executable directory is on your Windows PATH.

## 3. Configure AWS CLI credentials

Install AWS CLI v2 if it is not already installed. Then verify:

```powershell
aws --version
```

Configure credentials for the AWS account you will use:

```powershell
aws configure
```

Enter the values when prompted:

- AWS Access Key ID: your own IAM access key
- AWS Secret Access Key: its matching secret
- Default region name: `us-east-1`
- Default output format: `json`

Verify the active identity and region:

```powershell
aws sts get-caller-identity --region us-east-1
aws configure get region
```

The first command must return your AWS account identity. Confirm that it is the intended account. If your organization uses AWS IAM Identity Center/SSO instead of access keys, use its approved login process (for example `aws configure sso` followed by `aws sso login --profile YOUR_PROFILE`) and use that profile consistently for all AWS commands. Do not mix credentials from different accounts.

Set the region for the current PowerShell session:

```powershell
$env:AWS_DEFAULT_REGION = "us-east-1"
$env:AWS_REGION = "us-east-1"
```

If using a named AWS CLI profile, set it for the current session too:

```powershell
$env:AWS_PROFILE = "YOUR_PROFILE_NAME"
```

Omit the profile command if you use the default profile.

## 4. Create the local environment file

Make sure the terminal is at the project root (`starter`):

```powershell
Get-Location
Test-Path .\config.py
Test-Path .\requirements.txt
```

Both `Test-Path` commands must return `True`.

Create `.env` from the supplied template:

```powershell
Copy-Item .env.example .env
```

At this stage, leave the three Knowledge Base IDs empty. They will be added after you create the Knowledge Bases in the AWS Console. Do not invent or copy example IDs.

The project defaults are:

```dotenv
AWS_REGION=us-east-1
PROJECT_NAME=udacity-agentcore
```

Keep `.env` private. Do not upload it to GitHub or include it in public screenshots.

## 5. Deploy the foundation CloudFormation stack

**This is a required prerequisite.** The stack creates the DynamoDB tables, policy-document S3 bucket, S3 Vectors bucket and indexes, AgentCore execution role, and CloudWatch log group used by the project.

### Option A — AWS Console (as specified in the project document)

1. Sign in to the AWS Console using the intended account.
2. Set the region to **US East (N. Virginia) / `us-east-1`**.
3. Open **CloudFormation**.
4. Choose **Create stack → With new resources (standard)**.
5. Choose **Upload a template file**.
6. Upload `infrastructure/starter_stack.yaml` from the extracted `starter` folder.
7. Choose Next.
8. Stack name: `udacity-agentcore`.
9. Leave stack options at their defaults and choose Next.
10. On the review page, check **I acknowledge that AWS CloudFormation might create IAM resources with custom names**.
11. Choose Submit.
12. Wait until the stack status is **`CREATE_COMPLETE`**. Do not continue if it fails or is still creating.

### Option B — AWS CLI (PowerShell)

Run from the `starter` folder:

```powershell
aws cloudformation deploy --template-file infrastructure/starter_stack.yaml --stack-name udacity-agentcore --capabilities CAPABILITY_NAMED_IAM --region us-east-1
```

Check the stack status:

```powershell
aws cloudformation describe-stacks --stack-name udacity-agentcore --query "Stacks[0].StackStatus" --output text --region us-east-1
```

Expected result:

```text
CREATE_COMPLETE
```

If the stack fails, open CloudFormation → the stack → Events, identify the first failure reason, fix the underlying permission/resource issue, and redeploy. Do not proceed with a failed stack.

## 6. Seed DynamoDB and S3 with sample data

After CloudFormation reaches `CREATE_COMPLETE`, run from the `starter` folder with the virtual environment activated and AWS credentials available:

```powershell
python infrastructure/seed_data.py
```

This is expected to populate the DynamoDB customer/order tables and upload policy documents to the policy S3 bucket under:

- `policies/returns/`
- `policies/shipping/`
- `policies/warranty/`

Run the configuration check:

```powershell
python config.py
```

Check that the configuration output resolves the foundation resources, including the Orders table, Customers table, Workflow State table, Policy Bucket, Vector Bucket, vector index names, AgentCore role, and Agent log group. Knowledge Base IDs and runtime/Guardrail values may still show as not yet created at this stage; that is expected.

If configuration reports no CloudFormation exports, check the AWS account, AWS profile, region, and stack status. The stack name/project name must be `udacity-agentcore` in `us-east-1`.

## 7. Create the three Bedrock Knowledge Bases (AWS Console)

The three Knowledge Bases are created manually. **Choose self-managed Knowledge Bases, not Managed KB.** The project document says the pre-created S3 Vectors bucket and matching indexes must be reused; the backing store cannot be changed afterwards.

### 7.1 Get the exact vector bucket and index names

In the project terminal:

```powershell
python config.py
```

Record the displayed Vector Bucket and the three vector index names. They should correspond to:

- `returns-policy-index`
- `shipping-policy-index`
- `warranty-policy-index`

Use the actual bucket name shown by your own configuration output. Do not guess the bucket name or suffix.

### 7.2 Create Returns Knowledge Base

1. AWS Console → Amazon Bedrock → Knowledge Bases → Create Knowledge Base.
2. Choose the self-managed setup path.
3. Name: `novamart-returns-policy-kb`.
4. Add/configure an S3 data source using the policy bucket created by the stack and the prefix `policies/returns/`.
5. Embedding model: **Amazon Titan Embed Text v2** (`amazon.titan-embed-text-v2:0`).
6. Vector store: choose **S3 Vectors**, then use the existing vector bucket from `python config.py`.
7. Select the existing index `returns-policy-index`.
8. Complete the wizard and create the Knowledge Base.
9. Open its data source and choose **Sync**. Wait for the ingestion/sync job to complete successfully.
10. Copy the actual Knowledge Base ID from the console into a safe temporary note.

### 7.3 Create Shipping Knowledge Base

Repeat the same process with:

- Name: `novamart-shipping-policy-kb`
- S3 prefix: `policies/shipping/`
- Embedding model: `amazon.titan-embed-text-v2:0`
- Existing vector bucket: the exact same pre-created bucket
- Existing vector index: `shipping-policy-index`

Create it, run **Sync**, wait for successful completion, and record its actual Knowledge Base ID.

### 7.4 Create Warranty Knowledge Base

Repeat with:

- Name: `novamart-warranty-policy-kb`
- S3 prefix: `policies/warranty/`
- Embedding model: `amazon.titan-embed-text-v2:0`
- Existing vector bucket: the exact same pre-created bucket
- Existing vector index: `warranty-policy-index`

Create it, run **Sync**, wait for successful completion, and record its actual Knowledge Base ID.

### 7.5 Put the real Knowledge Base IDs into `.env`

Open `.env` in VS Code and replace the blank values with the actual IDs copied from your AWS Console:

```dotenv
RETURNS_KB_ID=PASTE_REAL_RETURNS_KB_ID_HERE
SHIPPING_KB_ID=PASTE_REAL_SHIPPING_KB_ID_HERE
WARRANTY_KB_ID=PASTE_REAL_WARRANTY_KB_ID_HERE
```

Do not leave the placeholder text. Do not include quotation marks unless the template explicitly requires them (it does not). Save `.env`.

Re-run:

```powershell
python config.py
```

Confirm that all three Knowledge Base IDs display as non-empty values. Confirm in the Bedrock Console that all three data sources have completed a successful Sync.

## 8. Run Task 2 tests (multi-agent graph)

From the `starter` root, with the virtual environment activated and AWS credentials configured:

```powershell
python tests/test_agent.py task2
```

Do not proceed if this task fails. Read the full traceback and fix the issue first. Check especially:

- The active Python interpreter is the `.venv` interpreter.
- `strands-agents`, `boto3`, and `python-dotenv` installed successfully.
- CloudFormation resources exist and the seeded DynamoDB data is available.
- The `.env` file is in the project root.
- The active AWS identity has permission to read/update the DynamoDB tables.

The expected agent behaviors from the project specification include:

- Order/return/refund requests: Inventory first, then Refund.
- Policy questions: PolicyAgent, with all three KB retrievers running in parallel.
- Account/tier questions: Inventory, not Policy.
- Math/calculation questions: Communication only; skip Inventory, Policy, and Refund.
- CommunicationAgent must always be the last routing tool called and the Orchestrator must not compose the final customer reply itself.

## 9. Deploy the complete project (Guardrail, Runtime, Memory, Observability)

Only proceed after all three Knowledge Bases exist and are synchronized and Task 2 passes.

Run:

```powershell
python src/agent_orchestrator.py deploy
```

The project deploy flow is intended to:

1. Create the Bedrock Guardrail with the required content, PII, topic, and profanity policies.
2. Publish a numbered Guardrail version (not `DRAFT`).
3. Deploy the AgentCore Runtime through the supplied AgentCore CLI wrapper.
4. Configure AgentCore Memory with session-summary strategy and seven-day event retention.
5. Configure CloudWatch logging at INFO and X-Ray tracing with 100% sampling.
6. Save the actual Guardrail identifiers/version and AgentCore Runtime ARN into `.env`.

The runtime environment must include the values required by the rubric: `AWS_REGION`, `PROJECT_NAME`, `RETURNS_KB_ID`, `SHIPPING_KB_ID`, `WARRANTY_KB_ID`, `AGENT_LOG_GROUP`, `GUARDRAIL_ID`, and numbered `GUARDRAIL_VERSION`. Runtime networking must be PUBLIC, protocol HTTP, and execution role must be the foundation stack role.

If deployment fails, stop and inspect the complete error. Check:

- `agentcore --version` is the expected CLI version.
- `agentcore validate` succeeds.
- Node.js 20+ and `uv` are installed and on PATH.
- The AWS account/region/profile is correct.
- The foundation stack is `CREATE_COMPLETE`.
- The three real Knowledge Base IDs are in `.env` and their data sources are synced.
- The IAM identity has the permissions required by the README, including CloudFormation/CDK deployment and `iam:PassRole` where applicable.
- The Bedrock models used by the project are available/enabled for the account and region.

Do not manually invent a Runtime ARN or Guardrail version. Use the actual deployment output.

After deployment, inspect `.env` (do not share it publicly) and confirm these are populated:

```dotenv
RETURNS_KB_ID=...
SHIPPING_KB_ID=...
WARRANTY_KB_ID=...
AGENTCORE_RUNTIME_ARN=...
GUARDRAIL_ID=...
GUARDRAIL_VERSION=...
```

The Guardrail version must be a numbered published version, not `DRAFT`.

## 10. Run Tasks 3, 4, 5, and 6 tests

Run each task separately so any failure is easy to identify:

```powershell
python tests/test_agent.py task3
python tests/test_agent.py task4
python tests/test_agent.py task5
python tests/test_agent.py task6
```

Expected rubric checks include:

- **Task 3:** Guardrail policies and numbered version; AgentCore CLI runtime deployment; PUBLIC networking; HTTP protocol; execution role; required environment variables; runtime ARN in `.env`.
- **Task 4:** AgentCore Memory created with `summaryMemoryStrategy`, `eventExpiryDuration=7`, descriptive name/description, and returned memory ARN.
- **Task 5:** Three Knowledge Bases use Titan Embed Text v2, the foundation stack's S3 Vectors bucket, matching indexes, synced data sources, and non-empty IDs; parallel retrieval returns non-empty results from Returns, Shipping, and Warranty.
- **Task 6:** CloudWatch log group enabled at INFO and X-Ray tracing enabled with `samplingRate=1.0`.

If any task fails, fix it and rerun that task until it passes. Then rerun earlier dependent tasks if you changed deployment/configuration code.

## 11. Run live end-to-end scenarios

Run the project's built-in test scenarios:

```powershell
python src/agent_orchestrator.py test
```

The project document requires checking these scenarios and routing paths:

### Scenario A — Return request

Customer: `CUST-001`

Request:

```text
I want to return my order ORD-27176
```

Expected routing:

```text
Orchestrator → Inventory → Refund → Communication
```

### Scenario B — Policy question

Request:

```text
What is the return policy for premium customers?
```

Expected routing:

```text
Orchestrator → PolicyAgent → 3 parallel Knowledge Base retrievers → Communication
```

The results must be grounded in the retrieved policy passages. Confirm that the three retrievers return non-empty results.

### Scenario C — Calculation

Request:

```text
How much are 5 items at $29.99 with 10% off?
```

Expected routing:

```text
Orchestrator → CommunicationAgent
```

Inventory, Policy, and Refund must be skipped. The arithmetic question and answer must be allowed by the Guardrail.

### Additional Guardrail validation

The project specification requires verifying that:

- A calculation using a supplied price and discount is allowed.
- Pricing negotiation / requests to change an advertised price are blocked.
- Competitor-product requests are blocked.
- Legal threats are blocked.

Use safe test prompts that exercise those categories. Capture evidence of allowed and blocked outcomes if possible. Do not treat a successful deploy alone as proof that the Guardrail behaves correctly.

## 12. Run the full grading suite and verify the score

Run:

```powershell
python tests/test_agent.py all
```

**Submission target: all tests pass and the displayed score is 120/120.**

If the score is below 120, or any test fails:

1. Do not submit yet.
2. Save/copy the complete terminal output.
3. Identify the first failing test and error.
4. Correct the issue.
5. Rerun the relevant task test.
6. Rerun the full command above.

A command that exits without showing the expected full suite, a `0/0` result, a skipped test suite, a syntax-only check, or an unverified claim is **not** evidence of 120/120.

## 13. Capture the required screenshots

### Screenshot 1 — Full test score

1. Maximize the VS Code terminal or PowerShell window.
2. Run `python tests/test_agent.py all` again.
3. Make sure the final output clearly shows every test passed and the required **120-point score**.
4. Capture a screenshot with the full result visible.
5. Save it with a clear filename such as `submission-test-results-120.png`.

Do not crop away failures, warnings, or the score. Do not edit or fabricate the screenshot.

### Screenshot 2 — AWS X-Ray Service Map

1. Run a live request again:

   ```powershell
   python src/agent_orchestrator.py test
   ```

2. Note the trace ID printed by the test output if one is shown.
3. Wait approximately **30–60 seconds** for trace data to appear.
4. Open AWS Console → CloudWatch → X-Ray traces → Service map (the project document also identifies X-Ray → Service map).
5. Confirm the console is in `us-east-1` and the same AWS account used for deployment.
6. Select a recent time range, such as **Last 5 minutes**.
7. Refresh if necessary after waiting for trace ingestion.
8. Capture the connected trace graph showing `NovaMart-Orchestrator` and the worker-agent call chain. The rubric specifically expects worker nodes including PolicyAgent and KnowledgeBase agents in the map.
9. Save it with a clear filename such as `xray-service-map.png`.

If the map is empty, confirm that a live request completed, X-Ray is enabled, sampling is 100%, the time range is correct, and the console is in the correct region. Wait and refresh. Do not submit an empty map as evidence.

### Recommended extra screenshots

- Bedrock console showing the three Knowledge Bases and successful data-source sync jobs.
- Guardrail test evidence showing an allowed calculation and blocked negotiation/competitor/legal-threat inputs.
- AgentCore Runtime status showing the deployed runtime.

The required submission evidence in the project document is the 120-point test screenshot and X-Ray Service Map screenshot.

## 14. Final pre-submission checklist

Check every item before uploading/submitting:

- [ ] The submitted project is the prepared `starter` project, not another classroom exercise folder.
- [ ] `src/agent_orchestrator.py` contains the implementation.
- [ ] The required support files/tests/infrastructure have not been unintentionally changed.
- [ ] CloudFormation stack `udacity-agentcore` reached `CREATE_COMPLETE` in `us-east-1`.
- [ ] `infrastructure/seed_data.py` completed successfully.
- [ ] Returns Knowledge Base created with correct S3 prefix, vector bucket/index, Titan Embed Text v2, and successful Sync.
- [ ] Shipping Knowledge Base created with correct S3 prefix, vector bucket/index, Titan Embed Text v2, and successful Sync.
- [ ] Warranty Knowledge Base created with correct S3 prefix, vector bucket/index, Titan Embed Text v2, and successful Sync.
- [ ] `.env` has all three real KB IDs.
- [ ] `python tests/test_agent.py task2` passes.
- [ ] `python src/agent_orchestrator.py deploy` completes successfully.
- [ ] `.env` has the real AgentCore Runtime ARN, Guardrail ID, and numbered Guardrail version.
- [ ] `python tests/test_agent.py task3` passes.
- [ ] `python tests/test_agent.py task4` passes.
- [ ] `python tests/test_agent.py task5` passes and confirms all three retrieval sources return results.
- [ ] `python tests/test_agent.py task6` passes.
- [ ] The three end-to-end scenarios route as specified.
- [ ] Guardrail arithmetic/blocked-topic behavior has been checked.
- [ ] `python tests/test_agent.py all` passes with **120/120**.
- [ ] Screenshot of the actual 120-point result is saved.
- [ ] Screenshot of the connected X-Ray Service Map is saved.
- [ ] `.env`, AWS keys, secret values, and credentials are not included in a public repository or screenshots.
- [ ] The final ZIP/submission contains the required source code and evidence files in the format requested by Udacity.

**Submit only after the required tests and evidence are complete.** If Udacity's submission page asks for a particular file layout or separate screenshot uploads, follow that page's upload instructions as well.

## 15. Clean up AWS resources — only after submission/evidence

The project document explicitly says to clean up after taking screenshots and submitting, to avoid ongoing AWS charges. Do not run cleanup before you have captured all evidence or if you may need the deployed resources for further testing/review.

First run the dry run:

```powershell
python infrastructure/cleanup.py
```

Review what it says would be deleted. Only when you are certain you no longer need the resources, run:

```powershell
python infrastructure/cleanup.py --yes
```

Cleanup deletes project-created resources. You may need to recreate them if a reviewer requests changes.

---

## Appendix A — Quick command sequence (Windows PowerShell)

Run these from the extracted `starter` folder, in order. This is a quick reference; the detailed steps above explain the AWS Console work and what each command verifies.

```powershell
# 1. Create and activate Python virtual environment
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip

# 2. Install Python requirements
python -m pip install -r requirements.txt

# 3. Check AWS credentials and region
aws sts get-caller-identity --region us-east-1
$env:AWS_DEFAULT_REGION = "us-east-1"
$env:AWS_REGION = "us-east-1"

# 4. Create .env (do this once; do not overwrite an existing .env with real IDs)
if (-not (Test-Path .env)) { Copy-Item .env.example .env }

# 5. Verify AgentCore CLI
agentcore --version
agentcore validate

# 6. Deploy foundation stack (alternative to AWS Console deployment)
aws cloudformation deploy --template-file infrastructure/starter_stack.yaml --stack-name udacity-agentcore --capabilities CAPABILITY_NAMED_IAM --region us-east-1

# 7. Seed data and inspect configuration
python infrastructure/seed_data.py
python config.py

# 8. After creating/syncing all 3 KBs in AWS Console and entering their real IDs in .env:
python config.py
python tests/test_agent.py task2

# 9. Deploy Guardrail, Runtime, Memory, and Observability
python src/agent_orchestrator.py deploy

# 10. Run task tests
python tests/test_agent.py task3
python tests/test_agent.py task4
python tests/test_agent.py task5
python tests/test_agent.py task6

# 11. Run live scenarios and full grading suite
python src/agent_orchestrator.py test
python tests/test_agent.py all

# 12. After successful submission only: dry-run cleanup, then actual cleanup if desired
python infrastructure/cleanup.py
# python infrastructure/cleanup.py --yes
```

## Appendix B — AWS CloudShell foundation-stack command (Bash)

If you use AWS CloudShell instead of local AWS CLI, upload `infrastructure/starter_stack.yaml` into CloudShell, then run:

```bash
aws cloudformation deploy \
  --template-file starter_stack.yaml \
  --stack-name udacity-agentcore \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

aws cloudformation describe-stacks \
  --stack-name udacity-agentcore \
  --query 'Stacks[0].StackStatus' \
  --region us-east-1
```

Wait for `CREATE_COMPLETE`. The project seeding command (`python infrastructure/seed_data.py`) should be run from the configured Python project environment with the project files and dependencies available.

---

## Important note about grading

This guide provides the steps required by the supplied project document and the prepared submission guide. It cannot guarantee a 100/100 or 120/120 result: the final score depends on the actual rubric tests, AWS account permissions, successful resource creation, model access, Knowledge Base ingestion, runtime deployment, and trace evidence in your environment. Use the actual test output as the source of truth.
