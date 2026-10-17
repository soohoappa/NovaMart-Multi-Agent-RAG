# NovaMart — Multi-Agent E-Commerce RAG

Udacity AWS Agent Engineer Final Project

A multi-agent customer support system built with the Strands Agents SDK, Amazon Bedrock, and Amazon Bedrock AgentCore Runtime.

The system retrieves customer and order information, searches company policies through parallel RAG, evaluates return requests, and delegates the final customer response to a Communication Agent.

**Final automated assessment: 120/120 points.**

## Architecture

```text
Customer Request
    |
OrchestratorAgent
    |
    +-- InventoryAgent ------ DynamoDB orders and customers
    |
    +-- RefundAgent --------- Inventory context and return initiation
    |
    +-- PolicyAgent
    |       |
    |       +-- ReturnsRetriever ---- Returns Knowledge Base
    |       +-- ShippingRetriever --- Shipping Knowledge Base
    |       +-- WarrantyRetriever --- Warranty Knowledge Base
    |                Parallel execution
    |
    +-- CommunicationAgent -- Final customer-facing response

Shared results: DynamoDB WorkflowState
Policy documents: Amazon S3
Vector storage: Amazon S3 Vectors
Deployment: AgentCore CLI → AgentCore Runtime
Safety: Amazon Bedrock Guardrails
Observability: CloudWatch Logs and AWS X-Ray
```

Connections represent delegation relationships. Execution order depends on the request type.

## Agent Configuration

| Agent | Model configuration | Temperature | Tools |
|---|---|---:|---|
| Orchestrator | `config.ORCHESTRATOR_MODEL_ID` — Claude Haiku 4.5 | 0.0 | `initialize_session`, four `route_to_*` tools |
| Inventory | `config.WORKER_MODEL_ID` — Claude Sonnet 4.5 | 0.1 | `check_order_status`, `get_customer_tier`, `list_customer_orders` |
| Refund | Worker model | 0.1 | `get_inventory_context`, `initiate_refund` |
| Policy coordinator | Worker model | 0.2 | `search_all_policies` |
| Three policy retrievers | Worker model | 0.0 | One domain-specific retrieval tool each |
| Communication | Worker model | 0.3 | `get_full_workflow_context` |

Model IDs are referenced through configuration constants rather than hardcoded in agent builders.

### Routing Rules

1. Call `initialize_session` first for every request, including follow-up messages.
2. Route order-status, return, and refund requests through Inventory, then Refund.
3. Route policy questions to Policy.
4. Route customer account and tier questions to Inventory, not Policy.
5. Route calculations directly to Communication after initialization.
6. Call Communication as the final tool call and use its response as the customer-facing answer.

The Orchestrator uses `SequentialToolExecutor` to execute its tools sequentially. This resolved an observed initialization race in which Inventory attempted to read WorkflowState before the session record was available.

Policy retrieval remains parallel through `ThreadPoolExecutor(max_workers=3)` and `as_completed()`.

## Shared WorkflowState

Each session has a DynamoDB record keyed by `session_id`.

Routing tools read the current state, invoke their worker, and save its output using `_update_workflow_state()` with the previously read `expected_version`.

Worker results are stored in:

- `inventory_agent`
- `policy_agent`
- `refund_agent`
- `communication_agent`

Version-based conditional updates provide optimistic locking. Existing session records are preserved for follow-up requests.

## Policy RAG

Three independently configured Knowledge Bases use:

| Domain | S3 prefix | Vector index |
|---|---|---|
| Returns | `policies/returns/` | `returns-policy-index` |
| Shipping | `policies/shipping/` | `shipping-policy-index` |
| Warranty | `policies/warranty/` | `warranty-policy-index` |

All use Amazon Titan Text Embeddings V2 (`amazon.titan-embed-text-v2:0`), 1024-dimensional embeddings, and the foundation stack's S3 Vectors backing store.

Each retriever is a Strands Agent with one tool calling `retrieve_from_knowledge_base()`. Results from all three agents are collected and combined for the Policy coordinator.

The final response preserves retrieved document filenames and S3 source references. All three data sources were synchronized and returned passages during testing.

## Return Handling

Inventory retrieves facts without making eligibility decisions.

Refund checks Inventory context and applies the configured return windows:

- Standard: 30 days.
- Premium: 60 days.
- The window starts from delivery, not purchase.

Missing delivery information must not be invented.

`initiate_refund()` records a return request by changing an existing delivered order to `return_requested`, with a reason and return reference. A conditional update prevents creating a nonexistent order or repeating a return request for an order no longer in delivered status.

This action does not execute a payment refund.

## Guardrails

`create_guardrail()` configures:

| Policy | Configuration |
|---|---|
| Content | SEXUAL, VIOLENCE, HATE: HIGH; INSULTS, MISCONDUCT: MEDIUM |
| PII | Credit card numbers and SSNs: BLOCK; emails and phones: ANONYMIZE |
| Topics | DENY competitor products, pricing negotiations, and legal threats |
| Words | Managed profanity list |
| Messages | Friendly blocked-input and blocked-output responses |

The topic policy uses the STANDARD tier and `us.guardrail.v1:0`. Its pricing-negotiation definition excludes arithmetic using an already specified price and discount.

A numbered guardrail version is published. The supplied `_apply_guardrail()` applies the configured guardrail to agent models.

## Deployment and Memory

Deployment uses the supplied `agentcore_cli.py` helper:

1. Stage application code in `build/runtime/`.
2. Configure PUBLIC networking, HTTP, and the foundation execution role.
3. Pass region, project, KB, logging, and guardrail environment variables.
4. Deploy with the AgentCore CLI.
5. Read the deployed Runtime ARN and wait for READY.

The runtime package targets Python 3.12. Local development used Windows CMD, Miniconda, and Python 3.11.15.

Validated deployment tooling included AgentCore CLI 0.30.0, Node.js, and `uv`.

`configure_memory()` provisions a Memory resource with:

- `summaryMemoryStrategy`
- `/summaries/{actorId}/{sessionId}` namespace
- Seven-day event retention
- ACTIVE-state waiting and ARN return

Memory resource configuration was verified by the automated tests. Persistent conversation recovery through Memory event writes and retrieval was not demonstrated; resource provisioning alone does not establish that behavior.

## Observability

`configure_observability()` passes the following settings to the supplied `apply_observability_config()` inside `try/except`:

- CloudWatch log group: `config.AGENT_LOG_GROUP`
- Logging level: INFO
- Logging enabled
- X-Ray tracing enabled
- Sampling rate: 1.0

The supplied tracing helper records worker and Knowledge Base calls from local execution and deployed Runtime requests.

## Setup and Execution

Run commands from the project root.

### Windows CMD

```cmd
conda activate aws
chcp 65001
set PYTHONUTF8=1
set PYTHONIOENCODING=utf-8
python -m pip install -r requirements.txt
```

Configure AWS credentials locally for `us-east-1`. Do not store credentials in source files.

Create a local environment file:

```cmd
copy .env.example .env
```

Populate KB IDs, the deployed Runtime ARN, and the numbered guardrail ID/version as the resources become available.

### Infrastructure

For a new environment, use `infrastructure/starter_stack.yaml`.

This project began with an older foundation template where `VectorStoreBucket` was a general-purpose S3 bucket. CloudFormation rejected changing that logical resource to an S3 Vectors type.

For the existing stack, `infrastructure/migration_stack.yaml` preserves the original bucket and adds the actual vector bucket as `PolicyVectorBucket`, together with the three indexes. Output exports used by configuration are preserved.

This is a documented logical-ID difference from the rubric's `VectorStoreBucket` terminology. The Knowledge Bases use the actual stack-created S3 Vectors bucket and matching indexes.

The migration template is intended for updating that older stack.

For a new environment, seed sample data once after foundation deployment:

```cmd
python infrastructure\seed_data.py
python config.py
```

The seeder generates random order IDs. Hardcoded demonstration order IDs may therefore be absent. Repeated seeding may add more sample orders.

Create and synchronize all three Knowledge Bases before full deployment.

### Local Tests

These commands execute locally but call actual AWS services:

```cmd
python tests\test_agent.py task2
python tests\test_agent.py task5
python src\agent_orchestrator.py chat
python src\agent_orchestrator.py test
```

### Deployment

Restore the account-independent CLI configuration templates after cloning:

```cmd
copy agentcore\agentcore.example.json agentcore\agentcore.json
copy agentcore\aws-targets.example.json agentcore\aws-targets.json
```

Run these copy commands only for a fresh checkout. Do not overwrite an existing deployment configuration. The deployment helper populates the runtime settings; account-specific files remain excluded from Git.

After all prerequisites and implementation tasks are complete:

```cmd
python src\agent_orchestrator.py deploy
```

Save the printed Runtime ARN and numbered guardrail configuration in the local `.env`.

### Deployed Runtime Invocation

```cmd
python src\agent_orchestrator.py invoke "I want to return my order ORD-27176."
python src\agent_orchestrator.py invoke "What is the return policy for Premium customers?"
python src\agent_orchestrator.py invoke "How much are 5 items at $29.99 with 10% off?"
```

### Full Assessment

Run after deployment:

```cmd
python tests\test_agent.py all
```

Final result: **120/120 points (100%)**.

## Rubric Evidence

| Rubric area | Implementation / evidence |
|---|---|
| Worker tools, models, temperatures | `src/agent_orchestrator.py` agent builders; Task 2 |
| Parallel multi-agent RAG | `build_policy_agent()`, `search_all_policies()`; Task 5 |
| Routing and shared state | `build_orchestrator_agent()` and WorkflowState helpers; local traces |
| Guardrail policies and numbered version | `create_guardrail()`; Task 3 and deployed adversarial tests |
| CLI-based HTTP Runtime deployment | `deploy_to_agentcore_runtime()`; successful Runtime invocations |
| Session-summary Memory resource | `configure_memory()`; Task 4 |
| Three synchronized S3 Vectors KBs | Task 5 and source-bearing policy responses |
| Logging and tracing configuration | `configure_observability()`; Task 6 |
| Connected end-to-end trace graph | Service Map and Runtime trace screenshots |

### Screenshots

- [Final assessment: 120/120](docs/screenshots/all-tests-120.png)
- [Assessment details, part 1](docs/screenshots/all-tests-details-01.png)
- [Assessment details, part 2](docs/screenshots/all-tests-details-02.png)
- [Full Service Map](docs/screenshots/xray-service-map-overview.png)
- [Service Map detail 1](docs/screenshots/xray-service-map-detail-01.png)
- [Service Map detail 2](docs/screenshots/xray-service-map-detail-02.png)
- [Deployed return-request trace map](docs/screenshots/runtime-return-trace-map.png)
- [Deployed return-request trace details](docs/screenshots/runtime-return-trace-details.png)

The full Service Map includes Policy and Knowledge Base nodes. The individual return-request trace shows Inventory, Refund, and Communication; that request does not require Policy.

## Manual Validation

| Scenario | Observed result |
|---|---|
| Existing order-status inquiry | Order facts retrieved; Inventory → Refund → Communication |
| Customer-tier inquiry | Premium tier retrieved; Policy and Refund skipped |
| Calculation | Communication path; final total `$134.96` |
| Policy question | Three KB retrievers executed; final response included actual S3 references |
| Nonexistent order return | Missing order reported; return initiation not performed |
| Pricing negotiation | Blocked |
| Competitor product recommendation | Blocked |
| Legal threat | Blocked |
| Ordinary calculation with discount | Allowed by Guardrails |
| PII in observed agent output | Email displayed as `{EMAIL}` |

## Limitations

- A successful eligible return and resulting order mutation have not been demonstrated end to end.
- Seeded orders do not include an actual delivery timestamp; eligibility cannot be safely inferred from order date or estimated delivery.
- Policy reasoning and routing depend on model behavior. Passing the assessment does not prove correctness for every input.
- AgentCore Memory provisioning was tested; durable conversation restoration was not demonstrated.
- Optional dashboards, authentication, frontend, and additional persistent session storage are outside this submission's scope.

## Submission and Repository Hygiene

The private Udacity submission includes the completed source, required environment configuration, and assessment/tracing evidence.

The public repository must exclude:

- `.env` and AWS credentials
- Real account-specific deployment configuration
- Runtime and Memory identifiers
- Generated deployment state and packages
- Unredacted screenshots or logs containing account-specific information

Use `.env.example` with blank resource identifiers for public documentation. Keep the real `.env` only in the private submission and local environment.

## Cleanup

Run cleanup only after submission and evidence preservation:

```cmd
python infrastructure\cleanup.py
```

Review the dry-run output before running:

```cmd
python infrastructure\cleanup.py --yes
```

Confirm that project resources are removed afterward. The retained legacy S3 bucket and migration-specific resources should also be checked during cleanup. Preserve source code, documentation, and redacted evidence.