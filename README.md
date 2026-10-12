# NovaMart Multi-Agent Customer Support System

A production-grade, multi-agent AI customer support system built on **Amazon Bedrock AgentCore** and the **Strands Agents SDK**. This is the capstone project ("Multi-Agent E-commerce RAG") for Udacity's *Future AWS Agent Engineer* course.

NovaMart is a fictional e-commerce company whose support team handles thousands of order-status, return, and policy requests daily. This system automates that workflow end-to-end: a central orchestrator understands each request, routes it to the right specialist agent, retrieves the relevant facts and policy, makes a decision where needed, and composes a professional customer-facing reply — all while staying inside enterprise guardrails and fully observable in CloudWatch and X-Ray.

## Architecture

Five Strands agents work together in an **Orchestrator → Workers** hierarchy, coordinating through a shared `WorkflowState` record in DynamoDB (optimistic locking, versioned writes).

```
                         Customer Request
                                │
                                ▼
                     ┌─────────────────────┐
                     │  OrchestratorAgent    │  Claude Haiku 4.5
                     │  routes requests,     │  temperature 0.0
                     │  manages WorkflowState│
                     └──────────┬───────────┘
            ┌───────────────────┼───────────────────┬─────────────────────┐
            ▼                   ▼                    ▼                    ▼
   ┌─────────────────┐ ┌─────────────────┐  ┌─────────────────┐ ┌──────────────────────┐
   │  InventoryAgent  │ │   PolicyAgent    │  │   RefundAgent    │ │ CommunicationAgent    │
   │  DynamoDB lookups│ │ multi-agent RAG  │  │ eligibility      │ │ composes final reply  │
   └─────────────────┘ │  coordinator     │  │ decisions        │ └──────────────────────┘
                        └────────┬─────────┘  └─────────────────┘
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                    ▼
     ┌─────────────────┐ ┌─────────────────┐  ┌─────────────────┐
     │ ReturnsRetriever │ │ShippingRetriever │  │WarrantyRetriever │   (run in parallel via
     │  Bedrock KB      │ │  Bedrock KB      │  │  Bedrock KB      │    ThreadPoolExecutor)
     └─────────────────┘ └─────────────────┘  └─────────────────┘
```

| Agent | Model | Temp | Responsibility | Tools |
|---|---|---|---|---|
| **OrchestratorAgent** | Claude Haiku 4.5 | 0.0 | Routes every request, enforces 6 routing rules, owns `WorkflowState` | `initialize_session`, `route_to_inventory_agent`, `route_to_policy_agent`, `route_to_refund_agent`, `route_to_communication_agent` |
| **InventoryAgent** | Claude Sonnet 4.5 | 0.1 | Pure fact-gathering from DynamoDB — order status, customer tier. Never makes decisions. | `check_order_status`, `get_customer_tier`, `list_customer_orders` |
| **PolicyAgent** | Claude Sonnet 4.5 | 0.2 | Coordinates parallel RAG across 3 Knowledge Bases and synthesizes an answer | `search_all_policies` (internally fans out to 3 retriever sub-agents) |
| **RefundAgent** | Claude Sonnet 4.5 | 0.1 | Applies return-eligibility rules (30-day Standard / 60-day Premium) and initiates refunds | `get_inventory_context`, `initiate_refund` |
| **CommunicationAgent** | Claude Sonnet 4.5 | 0.3 | Drafts the final warm, professional customer-facing reply from the full workflow context | `get_full_workflow_context` |

### Orchestrator routing rules
1. Every request → `initialize_session` first.
2. Order status / return / refund → Inventory, then Refund.
3. Policy-meaning questions (windows, rates, warranty terms) → Policy.
4. Account questions ("am I Premium?") → Inventory, **never** Policy.
5. Math/calculation questions → answered directly by the orchestrator (no Inventory/Policy/Refund), rounding currency only after the full calculation.
6. Every request → `route_to_communication_agent` last, with no exceptions. The orchestrator never writes the customer-facing reply itself.

### Multi-agent RAG (PolicyAgent)
`PolicyAgent` is itself a coordinator over three specialist retriever agents — one per policy domain (Returns, Shipping, Warranty). On a query, `search_all_policies` dispatches all three **simultaneously** via `ThreadPoolExecutor(max_workers=3)` / `as_completed`, each hitting its own Bedrock Knowledge Base, then merges and synthesizes the results. This cuts retrieval latency roughly 3x versus sequential calls.

## Infrastructure

| Component | Details |
|---|---|
| **Compute** | Amazon Bedrock AgentCore Runtime (public network, HTTP), deployed via the AgentCore CLI |
| **Models** | Claude Haiku 4.5 (orchestrator routing), Claude Sonnet 4.5 (all worker agents) |
| **Data store** | DynamoDB — `udacity-agentcore-orders`, `udacity-agentcore-customers`, `udacity-agentcore-workflow-state` (optimistic locking) |
| **Knowledge Bases** | 3x Amazon Bedrock Knowledge Bases (self-managed / vector store), Titan Text Embeddings v2, backed by **S3 Vectors** (serverless vector store, no OpenSearch) |
| **Guardrails** | Amazon Bedrock Guardrails — content filters (sexual/violence/hate at HIGH, insults/misconduct at MEDIUM), PII blocking (card numbers, SSNs) and anonymization (email, phone), topic denial (competitor products, pricing negotiations, legal threats), managed profanity list |
| **Memory** | AgentCore Memory, `SESSION_SUMMARY` strategy, 7-day event retention |
| **Observability** | CloudWatch Logs (INFO level) + AWS X-Ray distributed tracing (100% sampling via Transaction Search) |
| **IaC** | CloudFormation (`infrastructure/starter_stack.yaml`) provisions all DynamoDB tables, the policy-docs S3 bucket, the S3 Vectors bucket + 3 indexes, the IAM execution role, and the CloudWatch log group |

### Guardrail design note
The "Pricing Negotiations" topic is deliberately scoped to exclude ordinary arithmetic on an already-specified price and discount — it blocks *haggling* ("can you knock $20 off?"), not *math* ("what's 5 items at $29.99 with 10% off?"). A naively broad topic definition here breaks legitimate calculation requests; this was validated explicitly against the project's required math test case.

## Repository structure

```
starter/
├── src/
│   ├── agent_orchestrator.py    # All 5 agents, guardrail, deploy, memory, observability
│   ├── agent_utils.py           # Terminal trace UI, AgentTrace helper (provided)
│   ├── agent_observability.py   # X-Ray tracing decorator + CloudWatch logging (provided)
│   ├── bedrock_kb_retrieval.py  # Knowledge Base retrieval helper (provided)
│   ├── agentcore_cli.py         # AgentCore CLI wrapper: stage, configure, deploy (provided)
│   └── demo.py                  # Local end-to-end demo runner (provided)
├── infrastructure/
│   ├── starter_stack.yaml       # CloudFormation template
│   ├── seed_data.py             # Seeds DynamoDB + uploads policy docs to S3
│   └── cleanup.py               # Tears down all deployed resources
├── tests/
│   └── test_agent.py            # 120-point automated test suite (tasks 2–6)
├── config.py                    # Central config — reads CloudFormation exports + .env
├── .env                         # Runtime ARN, Guardrail ID/version, 3 KB IDs
└── requirements.txt
```

## Running it

### Prerequisites
- Python 3.12+, Node.js 20+, `uv`
- AWS CLI configured with credentials for `us-east-1`
- Bedrock model access enabled for Claude Haiku 4.5 and Claude Sonnet 4.5
- AgentCore CLI: `npm install -g @aws/agentcore@0.30.0`

### Setup
```bash
python -m venv venv && venv\Scripts\activate        # Windows
pip install -r requirements.txt

aws cloudformation deploy \
  --template-file infrastructure/starter_stack.yaml \
  --stack-name udacity-agentcore \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

python infrastructure/seed_data.py
cp .env.example .env
```

### Test locally (no AWS deploy required)
```bash
python tests/test_agent.py task2
python src/agent_orchestrator.py test     # 3 scripted scenarios
python src/agent_orchestrator.py chat     # interactive
```

### Deploy (guardrail + runtime + memory + observability, one command)
```bash
python src/agent_orchestrator.py deploy
```
Copy the printed `AGENTCORE_RUNTIME_ARN`, `GUARDRAIL_ID`, `GUARDRAIL_VERSION` into `.env`.

### Knowledge Bases (console-only — self-managed, existing S3 Vectors bucket/index)
Create 3 Knowledge Bases in the Bedrock console (Returns / Shipping / Warranty), each pointed at its S3 prefix and the existing `returns-policy-index` / `shipping-policy-index` / `warranty-policy-index` on the CloudFormation-provisioned S3 Vectors bucket. Sync each, then add the 3 KB IDs to `.env` and redeploy.

### Verify
```bash
python tests/test_agent.py all
```

## Test results

```
Task 2 — Multi-Agent Orchestration .......... 40/40  (100%)
Task 3 — AgentCore Deployment + Guardrails ... 20/20  (100%)
Task 4 — Memory .............................. 15/15  (100%)
Task 5 — Bedrock Knowledge Bases ............. 25/25  (100%)
Task 6 — Observability ....................... 20/20  (100%)
───────────────────────────────────────────────────────
Total ........................................ 120/120 (100%)
```

## Example interaction

**Request:** *"I want to return my wireless headphones from order ORD-27176"*

```
Orchestrator → initialize_session
Orchestrator → route_to_inventory_agent
  InventoryAgent: Order facts (Wireless Headphones Pro, $149.99, Premium tier,
                  60-day window) retrieved from DynamoDB
Orchestrator → route_to_refund_agent
  RefundAgent: Within window → APPROVED → return reference RET-DA485ABE
Orchestrator → route_to_communication_agent
  CommunicationAgent: Composes final warm, professional reply with next steps
```

**Request:** *"What is the return policy for premium customers?"*

```
Orchestrator → route_to_policy_agent
  PolicyAgent → search_all_policies (3 KBs queried in parallel, ~5–12s total)
    ReturnsRetriever, ShippingRetriever, WarrantyRetriever all respond
  Synthesized answer: 60-day Premium return window, tier requirements,
                       full return process, cited from source policy docs
Orchestrator → route_to_communication_agent
```

## Tech stack

Amazon Bedrock AgentCore · Strands Agents SDK · Claude Haiku 4.5 / Sonnet 4.5 · Amazon Bedrock Knowledge Bases · S3 Vectors · Amazon Bedrock Guardrails · AgentCore Memory · DynamoDB · AWS X-Ray · Amazon CloudWatch · AWS CloudFormation

## Acknowledgment

Built as the capstone project for Udacity's *Future AWS Agent Engineer* course. Starter scaffolding, tests, and infrastructure templates (`agent_utils.py`, `agent_observability.py`, `bedrock_kb_retrieval.py`, `agentcore_cli.py`, `tests/test_agent.py`, CloudFormation template) provided by Udacity/AWS; all agent logic, guardrail configuration, and orchestration in `agent_orchestrator.py` implemented as part of this submission.
