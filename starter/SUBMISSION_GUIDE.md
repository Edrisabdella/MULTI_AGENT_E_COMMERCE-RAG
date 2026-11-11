# NovaMart Project Submission Guide

This guide follows the supplied 30-page project specification. The source implementation is in `src/agent_orchestrator.py`; the provided support modules, tests, infrastructure, and configuration are preserved.

## 1. Implementation scope

| Project requirement | Implementation location |
|---|---|
| Inventory worker: order lookup, customer tier, customer order list | `build_inventory_agent()` |
| Refund worker: read shared inventory findings, enforce 30-day Standard / 60-day Premium window, record eligible return | `build_refund_agent()` |
| Three policy retriever agents and parallel retrieval with `ThreadPoolExecutor(max_workers=3)` / `as_completed()` | `build_policy_agent()` |
| Communication worker reads complete WorkflowState and drafts final reply | `build_communication_agent()` |
| Five orchestrator tools and all six routing rules | `build_orchestrator_agent()` |
| Versioned Bedrock Guardrail: content, PII, topic, profanity policies | `create_guardrail()` |
| AgentCore CLI runtime deployment using PUBLIC / HTTP / foundation execution role | `deploy_to_agentcore_runtime()` |
| AgentCore session-summary memory, seven-day retention | `configure_memory()` |
| CloudWatch INFO logging and X-Ray 100% sampling | `configure_observability()` |
| Persist runtime ARN and Guardrail IDs after deployment | `deploy_all()` writes deployment outputs to `.env` |

The WorkflowState read uses a strongly consistent DynamoDB read. Writes use the supplied conditional version check and retry logic.

## 2. Local prerequisites

Use Python 3.12+, AWS CLI credentials, and the `us-east-1` region. The AWS identity needs the permissions described in the main README and project document. Install Python packages:

```bash
python -m pip install -r requirements.txt
```

Install the AgentCore CLI prerequisites (Node.js 20+ and `uv`), then:

```bash
npm install -g @aws/agentcore@0.30.0
agentcore --version
```

If the legacy `bedrock-agentcore-starter-toolkit` is installed, uninstall it as described in the README before using the CLI.

## 3. Provision foundation infrastructure

From the `starter/` project root, deploy `infrastructure/starter_stack.yaml` in CloudFormation:

1. AWS Console → CloudFormation → Create stack → With new resources (standard).
2. Upload `infrastructure/starter_stack.yaml`.
3. Set stack name to `udacity-agentcore`.
4. Leave defaults, acknowledge IAM resource creation, and submit.
5. Wait for `CREATE_COMPLETE`.

Alternatively, in AWS CloudShell:

```bash
aws cloudformation deploy \
  --template-file infrastructure/starter_stack.yaml \
  --stack-name udacity-agentcore \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Seed the DynamoDB records and policy documents:

```bash
python infrastructure/seed_data.py
cp .env.example .env
python config.py
```

Do not fabricate AWS resource IDs. CloudFormation-derived names should resolve after the stack is deployed. Keep `.env` private and do not commit it.

## 4. Create and sync the three Knowledge Bases

In AWS Console → Amazon Bedrock → Knowledge Bases → Create Knowledge Base, create three **self-managed** Knowledge Bases. Do not select Managed KB. Use the pre-created S3 Vectors bucket and matching vector index from `python config.py`.

| Setting | Returns | Shipping | Warranty |
|---|---|---|---|
| Name | `novamart-returns-policy-kb` | `novamart-shipping-policy-kb` | `novamart-warranty-policy-kb` |
| S3 prefix | `policies/returns/` | `policies/shipping/` | `policies/warranty/` |
| Embedding model | `amazon.titan-embed-text-v2:0` | same | same |
| Vector index | `returns-policy-index` | `shipping-policy-index` | `warranty-policy-index` |

For all three, choose the existing vector bucket created by the CloudFormation stack. After creating each KB, run **Sync** and wait for a completed ingestion job. Copy the actual IDs into `.env`:

```dotenv
RETURNS_KB_ID=actual_returns_kb_id
SHIPPING_KB_ID=actual_shipping_kb_id
WARRANTY_KB_ID=actual_warranty_kb_id
```

The example values above are placeholders, not usable AWS IDs.

## 5. Run the implementation tests

From the `starter/` root:

```bash
python tests/test_agent.py task2
python tests/test_agent.py task3
python tests/test_agent.py task4
python tests/test_agent.py task5
python tests/test_agent.py task6
```

Then deploy the complete system:

```bash
python src/agent_orchestrator.py deploy
```

The deploy flow creates/publishes the Guardrail, deploys the AgentCore Runtime, configures Memory, configures observability, and writes the runtime ARN and Guardrail identifiers into `.env`. The Knowledge Base IDs must already be valid and synced before deployment.

Run the complete rubric suite:

```bash
python tests/test_agent.py all
```

The final score and all test results must be captured from the real AWS-backed run. A local static syntax check is not a substitute for these integration tests.

## 6. Required live scenarios

Run:

```bash
python src/agent_orchestrator.py test
```

Verify the following paths in the trace output:

1. Customer `CUST-001`: `I want to return my order ORD-27176` → Orchestrator → Inventory → Refund → Communication.
2. Policy question: `What is the return policy for premium customers?` → Orchestrator → PolicyAgent → three parallel KB retrievers → Communication.
3. Calculation: `How much are 5 items at $29.99 with 10% off?` → Orchestrator → Communication; skip Inventory, Policy, and Refund.

Also test Guardrail behavior: arithmetic on a supplied price and discount should be allowed; pricing negotiation, competitor-product requests, and legal threats should be blocked. Test both input and output handling where applicable.

## 7. Required submission evidence

Capture and submit these screenshots from your own AWS account / test environment:

- Full automated test output showing all checks passed and the required **120-point score**.
- AWS Console → CloudWatch → X-Ray traces → Service map, after a live test request. Select a recent time range (e.g. Last 5 minutes) and capture the connected NovaMart Orchestrator → worker-agent graph, including PolicyAgent and KnowledgeBase nodes.
- Recommended additional evidence: Guardrail validation examples (allowed calculation and blocked negotiation / competitor / legal-threat inputs).

X-Ray traces may take 30–60 seconds to appear after an invocation.

## 8. Clean up after submission

Only after evidence is captured and the project is submitted:

```bash
python infrastructure/cleanup.py       # dry run
python infrastructure/cleanup.py --yes  # delete provisioned resources
```

Cleanup removes the AWS resources and may be needed again if a reviewer requests changes.

## 9. Important execution boundary

The Python implementation can be prepared and syntax-checked locally, but the following cannot be truthfully marked complete without your AWS account and its credentials: CloudFormation deployment, DynamoDB/S3 seeding, manual Knowledge Base creation and sync, Guardrail publication, AgentCore Runtime/Memory deployment, CloudWatch/X-Ray configuration, live scenario execution, and screenshots. No real resource IDs or test scores have been invented in this package.
