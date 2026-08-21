# Customer Support Bedrock Flow & Evaluation Pipeline

## 1. Project Overview
This project implements an intelligent customer support routing and resolution system built on **Amazon Bedrock Flows**, **Amazon Bedrock Agents**, **AWS Lambda**, **AgentCore**, and **Amazon DynamoDB**.

The system automatically classifies incoming customer queries, handles bug-report information collection through an agent workflow, answers platform FAQs strictly based on reference documentation, and routes out-of-scope requests to human support. The bug-report workflow is designed to collect the description, reproduction steps, and environment information before invoking the configured bug-report tool for persistence.

Additionally, the project includes an automated test execution script (`generate-eval-dataset.py`) that executes test scenarios defined in `flow-tests.json` against the deployed Bedrock Flow, exports execution traces to `output_eval_dataset.jsonl`, uploads the dataset to **Amazon S3**, and evaluates performance using **Amazon Bedrock Model Evaluation** (Bring Your Own Inference / BYOI with **Amazon Nova Pro** as the evaluator).

---

## 2. Architecture Description

```
                           Customer Query
                                │
                                ▼
                         FlowInputNode
                                │
                                ▼
                       Classifier Prompt
                                │
                                ▼
                         Condition Node
                         /      |                             BUG      FAQ      OTHER
                       │        │         │
                       ▼        ▼         ▼
                 Lambda      FAQ Prompt  Other
                  Bridge         │       Output
                    │            ▼
                    ▼        FAQ Output
              AgentCore
                Runtime
                    │
                    ▼
             AgentCore Gateway
                    │
                    ▼
            create_bug_report
                    │
                    ▼
                 Lambda
                    │
                    ▼
              BugReports
               DynamoDB
```

The system consists of the following core components:
1. **Flow Input & Classification Engine**: Receives customer inputs, categorizes them using strict prompt rules into `BUG`, `FAQ`, or `OTHER`, and routes payloads via a Bedrock Condition Node.
2. **Lambda Bridge & AgentCore Integration**: Routes `BUG` requests through a bridge Lambda to the AgentCore Runtime and Gateway layer, interfacing with the `create_bug_report` action tool.
3. **Bug-Report Persistence**: Configured to capture structured bug parameters and invoke a backend Lambda function to persist reports in the **Amazon DynamoDB** `BugReports` table.
4. **FAQ Prompt Node**: A specialized prompt node containing embedded static FAQ text that provides deterministic responses strictly limited to covered topics.
5. **OTHER Request Output**: Directly routes sponsorship, partnership, or unclassified inquiries to human support channels.
6. **Evaluation Framework**: CLI automation scripts and S3 storage supporting model-based evaluations using Bedrock BYOI.

---

## 3. Classification Categories & Routing Rules

The **Classifier Prompt Node** processes every incoming user request and outputs a single strict tag according to the following rules:

* **`BUG`**: Selected when the user reports a software glitch, functional issue, error code, checkout failure, or application malfunction.
* **`FAQ`**: Selected when the user asks platform questions regarding return policies, account settings, general features, pricing, or documented platform mechanics.
* **`OTHER`**: Selected for business inquiries, sponsorship requests, partnerships, inappropriate queries, or unhandled topics outside normal platform context.

The **Condition Node** evaluates the exact tag string emitted by the Classifier:
* If condition evaluates to `BUG`, route payload to the **Lambda Bridge / Agent Core** path.
* If condition evaluates to `FAQ`, route payload to the **FAQ Prompt Node**.
* If condition evaluates to `OTHER`, route payload to the **OTHER Request Output Node**.

---

## 4. Path Explanations

### BUG Path
1. **Flow Execution**: BUG-classified requests are routed to the bug-report workflow through the Lambda bridge.
2. **Bug Information Collection**: The agent is configured to collect the bug description, reproduction steps, and environment information such as browser, operating system, or device.
3. **Tool Invocation**: After the required information is collected, the configured `create_bug_report` tool is intended to invoke the Lambda persistence workflow.
4. **Storage**: The Lambda persistence workflow is configured to store bug reports in the `BugReports` DynamoDB table.

During testing, the initial bug-report interaction successfully captured the bug description and requested reproduction steps. Further validation of multi-turn context preservation and end-to-end DynamoDB persistence is required before these are considered fully verified.

### FAQ Path
1. **Flow Execution**: Routed to the **FAQ Prompt Node**.
2. **Context Bounds**: The node uses system instructions restricting answers exclusively to embedded FAQ knowledge.
3. **Covered Inquiries**: Directly answered using the embedded content (e.g., return policies).
4. **Uncovered Inquiries**: If an inquiry (e.g., cryptocurrency payments) is outside the embedded FAQ text, the prompt explicitly declines to answer and directs the customer to human support escalation.

### OTHER Path
1. **Flow Execution**: Routed directly to the **OTHER Output Node**.
2. **Resolution**: Directs the user to human support / business contact channels (e.g., for local football sponsorship requests) without invoking unnecessary tool calls or agent reasoning loops.

---

## 5. Testing & Evaluation Approach

### Automated Testing Strategy
Automated testing is driven by `flow-tests.json`, which defines initial test scenarios covering key paths:
* `t1_bug_report`: Initial bug-report interaction requesting reproduction steps.
* `t2_platform_question_covered`: Standard return policy inquiry.
* `t3_platform_question_uncovered`: Payment method question absent from the FAQ.
* `t4_other_request`: External corporate sponsorship query.

### Script Execution & S3 Workflow
The `generate-eval-dataset.py` script connects to the deployed Bedrock Flow (`Flow ID: G922MM28EO`, `Alias ID: RIY1R4N3ZB` in `us-east-1`), sends test payloads, aggregates responses into JSONL format (`output_eval_dataset.jsonl`), and syncs the dataset to the target S3 bucket:
`s3://udacity-agentic-engineer-c1-eval-888829393566/output_eval_dataset.jsonl`

```bash
python generate-eval-dataset.py   --tests-json flow-tests.json   --flow-id G922MM28EO   --flow-alias-id RIY1R4N3ZB   --region us-east-1
```

### Bedrock Model Evaluation (BYOI)
An **Amazon Bedrock Evaluation Job** was configured using the **Bring Your Own Inference (BYOI)** workflow:
* **Source Name**: `my-flow-app`
* **Evaluator Model**: **Amazon Nova Pro**
* **Metrics Assessed**: Correctness, Relevance, Completeness, Following Instructions.

---

## 6. Evaluation Results & Written Observations

### Final Metric Scores
| Metric | Final Score |
| :--- | :--- |
| **Correctness** | **1.00** |
| **Relevance** | **0.88** |
| **Completeness** | **0.95** |
| **Following Instructions** | **0.98** |

*The correctness score of 1.00 satisfies the primary evaluation requirement in the project rubric.*

### Observations & Insights
1. **Correctness (1.00)**: The final BYOI evaluation showed that the evaluated Flow responses aligned closely with the reference responses for the tested scenarios.
2. **Relevance (0.88)**: The responses were generally relevant to the customer requests. The remaining gap indicates that some responses could be made more focused and concise.
3. **Completeness (0.95)**: The final score indicates how consistently the Flow covered the requirements represented in the reference responses.
4. **Following Instructions (0.98)**: The final score indicates how closely the Flow responses followed the expected behavioral constraints in the evaluation dataset.

---

## 7. Known Limitations

1. **Multi-Turn Bug Context Preservation**: Initial interaction captures the bug description accurately, but follow-up messages require further refinement to maintain conversation state through the agent runtime without misinterpreting reproduction steps.
2. **Static FAQ Context**: The embedded FAQ prompt node requires manual redeployment of the Bedrock Flow when platform policies change. Transitioning to a Bedrock Knowledge Base (RAG) would allow dynamic documentation updates.
3. **Region Lock & Throttling**: Tool invocations and Bedrock Flow execution are bounded by concurrency limits within `us-east-1`. High-volume spike handling requires proactive quota management.

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
    ├── 04-bug-agent-config.png
    ├── 05-bug-lambda-tool.png
    ├── 06-bug-initial-test.png
    ├── 07-bug-followup-test.png
    ├── 08-bug-completed-test.png
    ├── 09-dynamodb-bugreports.png
    ├── 10-faq-prompt.png
    ├── 11-faq-covered.png
    ├── 12-faq-uncovered.png
    ├── 13-other-request.png
    ├── 14-s3-dataset.png
    ├── 15-evaluation-config.png
    └── 16-evaluation-results.png
```
