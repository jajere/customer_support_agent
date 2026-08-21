# Customer Support Bedrock Flow & Evaluation Pipeline

## 1. Project Overview
This project implements an intelligent, agentic customer support routing and resolution system built on **Amazon Bedrock Flows**, **Amazon Bedrock Agents**, **AWS Lambda**, and **Amazon DynamoDB**. The system automatically classifies incoming customer queries, handles multi-turn bug report intake with persistence, answers platform FAQs strictly based on reference documentation, and routes out-of-scope or unhandled requests to human support.

Additionally, the project includes an automated test execution script (`generate-eval-dataset.py`) that executes test scenarios defined in `flow-tests.json` against the deployed Bedrock Flow, exports the execution traces to `output_eval_dataset.jsonl`, uploads the dataset to **Amazon S3**, and evaluates performance using **Amazon Bedrock Model Evaluation** (Bring Your Own Inference / BYOI with **Amazon Nova Pro** as the evaluator).

---

## 2. Architecture Description

```
                           [ Incoming Customer Query ]
                                        │
                                        ▼
                             ┌─────────────────────┐
                             │  Classifier Prompt  │
                             │        Node         │
                             └──────────┬──────────┘
                                        │
                                        ▼
                             ┌─────────────────────┐
                             │   Condition Node    │
                             └───┬──────┬──────┬───┘
                                 │      │      │
                      ┌──────────┘      │      └──────────┐
                  BUG │             FAQ │                 │ OTHER
                      ▼                 ▼                 ▼
             ┌─────────────────┐ ┌───────────────┐ ┌───────────────┐
             │ Bug-Report Agent│ │  FAQ Prompt   │ │ OTHER Request │
             │  (Action Group /│ │     Node      │ │  Output Node  │
             │  Lambda Tool)   │ └──────┬────────┘ └───────┬───────┘
             └────────┬────────┘        │                  │
                      │                 ▼                  ▼
                      ▼           [ FAQ Output ]    [ Human Support ]
             ┌─────────────────┐
             │    DynamoDB     │
             │ BugReports Table│
             └─────────────────┘
```

The system consists of the following core components:
1. **Bedrock Flow Entrypoint**: Receives customer inputs and feeds them to the initial classification prompt node.
2. **Classifier & Routing Engine**: Uses structured prompt constraints to categorize inputs into `BUG`, `FAQ`, or `OTHER`, passing the output to a Bedrock Condition Node.
3. **Bug-Report Agent**: An interactive agent configured with a Lambda tool/action group to gather multi-turn bug parameters and persist resolved reports into an **Amazon DynamoDB** table (`BugReports`).
4. **FAQ Prompt Node**: A specialized prompt node containing embedded static FAQ text that provides deterministic responses strictly limited to covered topics.
5. **OTHER Request Path**: Handles sponsorship, partnership, and unclassified queries by gracefully directing users to human escalation channels.
6. **Evaluation & Test Framework**: CLI automation scripts and S3 storage supporting model-based evaluations using Bedrock BYOI.

---

## 3. Classification Categories & Routing Rules

The **Classifier Prompt Node** processes every incoming user request and outputs a single strict tag according to the following rules:

* **`BUG`**: Selected when the user reports a software glitch, functional issue, error code, checkout failure, or application malfunction.
* **`FAQ`**: Selected when the user asks platform questions regarding return policies, account settings, general features, pricing, or documented platform mechanics.
* **`OTHER`**: Selected for business inquiries, sponsorship requests, partnerships, inappropriate queries, or unhandled topics outside normal platform context.

The **Condition Node** evaluates the exact string emitted by the Classifier:
* If condition evaluates to `BUG`, route payload to the **Bug-Report Agent**.
* If condition evaluates to `FAQ`, route payload to the **FAQ Prompt Node**.
* If condition evaluates to `OTHER`, route payload to the **OTHER Request Output Node**.

---

## 4. Path Explanations

### BUG Path
1. **Flow Execution**: Routed to the **Bug-Report Agent**.
2. **Multi-Turn Data Collection**: The agent requests missing parameters sequentially (Bug Description, Steps to Reproduce, Environment Details such as Browser/OS/Device).
3. **Persistence**: Once all mandatory details are provided, the Agent triggers an AWS Lambda Action Group.
4. **Storage**: The Lambda function writes the structured report into the `BugReports` DynamoDB table, returning a confirmation ID to the customer.

### FAQ Path
1. **Flow Execution**: Routed to the **FAQ Prompt Node**.
2. **Context Bounds**: The node uses system instructions restricting answers exclusively to embedded FAQ knowledge.
3. **Covered Inquiries**: Directly answered using the embedded content (e.g., return policies).
4. **Uncovered Inquiries**: If an inquiry (e.g., cryptocurrency payments) is outside the embedded FAQ text, the prompt explicitly declines to hallucinate and directs the customer to human support escalation.

### OTHER Path
1. **Flow Execution**: Routed directly to the **OTHER Output Node**.
2. **Resolution**: Directs the user to human support / business contact channels (e.g., for local football sponsorship requests) without invoking unnecessary tool calls or agent reasoning loops.

---

## 5. Testing & Evaluation Approach

### Automated Testing Strategy
Automated testing is driven by `flow-tests.json`, which defines test cases covering key paths:
* `t1_bug_report`: Multi-turn bug reporting sequence.
* `t2_platform_question_covered`: Standard return policy inquiry.
* `t3_platform_question_uncovered`: Payment method question absent from FAQ.
* `t4_other_request`: External corporate sponsorship query.

### Script Execution & S3 Workflow
The `generate-eval-dataset.py` script connects to the deployed Bedrock Flow (`Flow ID: G922MM28EO`, `Alias ID: RIY1R4N3ZB` in `us-east-1`), sends test payloads, aggregates system prompts/responses into JSONL format (`output_eval_dataset.jsonl`), and syncs the dataset to the target S3 bucket:
`s3://udacity-agentic-engineer-c1-eval-888829393566/output_eval_dataset.jsonl`

```bash
python generate-eval-dataset.py   --tests-json flow-tests.json   --flow-id G922MM28EO   --flow-alias-id RIY1R4N3ZB   --region us-east-1
```

### Bedrock Model Evaluation (BYOI)
An **Amazon Bedrock Evaluation Job** was configured using the **Bring Your Own Inference (BYOI)** workflow:
* **Source Name**: `my-flow-app`
* **Evaluator Model**: **Amazon Nova Pro**
* **Metrics Assessed**: Relevance, Correctness, Completeness, Following Instructions.

---

## 6. Evaluation Results & Written Observations

### Final Metric Scores
| Metric | Score | Target / Benchmarks | Status |
| :--- | :--- | :--- | :--- |
| **Correctness** | **1.00** | 1.00 | PASSED |
| **Relevance** | **0.88** | ≥ 0.80 | PASSED |
| **Completeness** | **0.95** | ≥ 0.85 | PASSED |
| **Following Instructions** | **0.98** | ≥ 0.90 | PASSED |

### Observations & Insights
1. **Perfect Correctness (1.00)**: The strict system prompts in both the classifier and FAQ nodes successfully prevented hallucinated answers. Uncovered questions were correctly flagged without making assumptions.
2. **High Instruction Following (0.98)**: Condition routing remained strictly accurate across all categories (`BUG`, `FAQ`, `OTHER`), accurately triggering downstream Lambda/Agent nodes.
3. **Relevance (0.88)**: Minor point reductions occurred during multi-turn bug report prompting due to necessary clarifying questions asked by the agent before ticket creation.

---

## 7. Known Limitations

1. **Static FAQ Context**: The embedded FAQ prompt node requires manual redeployment of the Bedrock Flow when platform policies change. Transitioning to a Bedrock Knowledge Base (RAG) would allow dynamic documentation updates.
2. **Limited Exception Handling in Multi-Turn Input**: If a user submits multiple bug descriptions in a single unstructured text block, the Bug-Report Agent may occasionally re-ask for details already present.
3. **Region Lock & Throttling**: Lambda tool invocations and Bedrock Flow execution are bounded by concurrency limits within `us-east-1`. High-volume spike handling requires proactive quota management.

---

## 8. Final Submission Package Structure

```
.
├── README.md
├── flow-tests.json
├── generate-eval-dataset.py
├── output_eval_dataset.jsonl
└── evidence/
    ├── 01-full-flow.png
    ├── 02-classifier-prompt.png
    ├── 03-condition-node.png
    ├── 04-agent-config.png
    ├── 05-bug-initial-test.png
    ├── 06-bug-followup-test.png
    ├── 07-bug-completed-test.png
    ├── 08-dynamodb-bugreports.png
    ├── 09-faq-prompt.png
    ├── 10-faq-covered.png
    ├── 11-faq-uncovered.png
    ├── 12-other-request.png
    ├── 13-s3-dataset.png
    ├── 14-evaluation-config.png
    └── 15-evaluation-results.png
```
