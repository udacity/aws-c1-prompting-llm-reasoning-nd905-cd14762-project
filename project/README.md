# Customer Support Chatbot with Amazon Bedrock Flows

A customer-support chatbot for a fictional online shop, built on **Amazon Bedrock
Flows**. The bot classifies every incoming customer message into one of three
categories and routes it down a dedicated path — creating a support ticket,
answering from an embedded FAQ, or redirecting to human support.

---

## 1. Overview

| Category | What happens |
|---|---|
| **Bug Report** | Details are extracted (description, steps to reproduce, environment) and a ticket is created in **DynamoDB** via a **Lambda** tool. |
| **Platform Question** | Answered using an **embedded FAQ** covering orders, shipping, returns, payments, products, account, and privacy. |
| **Other Request** | Politely redirected to the **human support phone line** (`1-800-555-0199`). |

---

## 2. Architecture

```
                       ┌─────────────────────────────┐
                       │        FlowInputNode        │
                       │        (message in)         │
                       └──────────────┬──────────────┘
                                      │ document
                       ┌──────────────▼──────────────┐
                       │          Classifier         │   Prompt node (Nova Micro)
                       │   outputs exactly one of:   │
                       │  Bug Report / Platform      │
                       │  Question / Other Request   │
                       └──────────────┬──────────────┘
                                      │ category
                       ┌──────────────▼──────────────┐
                       │       RouteByCategory       │   Condition node
                       │   (exact string matching)   │
                       └───┬──────────────┬──────────┘
           category ==      │              │  category ==            (everything else
           "Bug Report"     │              │  "Platform Question"     → default)
                            │              │
              ┌─────────────▼──┐    ┌──────▼───────────┐     ┌─────────────────┐
              │ BugReportParser│    │   FAQResponder    │     │ SupportRedirect │
              │ (extract JSON) │    │ (FAQ embedded)    │     │ (phone redirect)│
              └───────┬────────┘    └──────┬───────────┘     └────────┬────────┘
                      │                    │                          │
              ┌───────▼────────┐    ┌──────▼───────────┐     ┌────────▼────────┐
              │ CreateBugReport│    │    FAQOutput     │     │  RedirectOutput │
              │ (Lambda → DDB) │    └──────────────────┘     └─────────────────┘
              └───────┬────────┘
              ┌───────▼─────────┐
              │ BugReportConfirm│
              └───────┬─────────┘
              ┌───────▼──────────┐
              │ BugReportOutput  │
              └──────────────────┘
```

**Three distinct paths, each terminating at its own Output node.**

---

## 3. Classification & Routing

The **Classifier** is a Prompt node that returns **only** a category label, with
no extra words, punctuation, or whitespace. This is essential because the
Condition node uses **exact string matching**.

**Classifier prompt:**
> You are a routing classifier for a customer support chatbot at an online shop.
> Classify the customer's message into exactly ONE of these three categories:
> 1. "Bug Report" — the customer is reporting a problem, error, crash, glitch, or malfunction with the website or app.
> 2. "Platform Question" — the customer is asking a question about orders, shipping, returns, payments, products, account, or privacy.
> 3. "Other Request" — anything that is neither a bug report nor a platform question.
> Rules: Reply with ONLY the category label, exactly as written above. Do not add any other words, punctuation, quotes, spaces, or newlines. Do not explain your choice or answer the customer.

**Condition node (`RouteByCategory`)** — two explicit conditions + default:

| Condition name | Expression (exact match) | Routes to |
|---|---|---|
| `BugReport` | `category == "Bug Report"` | BugReportParser |
| `PlatformQuestion` | `category == "Platform Question"` | FAQResponder |
| *(default — if all false)* | — | SupportRedirect |

> The `"Other Request"` label (and any unexpected output) falls through to the
> **default** branch → SupportRedirect. This keeps the routing robust: an
> unexpected classifier output can never crash the flow.

---

## 4. The Three Paths

### 4.1 Bug Report path

```
BugReportParser  →  CreateBugReport  →  BugReportConfirm  →  BugReportOutput
(Prompt)             (Lambda node)       (Prompt)             (Output)
```

1. **BugReportParser** (Nova Lite, temperature 0) — extracts structured JSON:
   ```json
   {"description": "...", "stepsToReproduce": "...", "environment": "..."}
   ```
   The prompt instructs strict-JSON-only output and empty strings for missing
   fields (so a minimal report like "my app keeps crashing" still works).
2. **CreateBugReport** — a **Lambda node** invoking the `create-bug-report`
   function, which writes the ticket to the **BugReports** DynamoDB table and
   returns `{"ticketId": "...", "status": "OPEN"}`.
3. **BugReportConfirm** (Nova Micro) — turns the Lambda result into a friendly
   confirmation that includes the ticket ID and OPEN status.
4. **BugReportOutput** — terminates the branch.

> ### ⚠️ Important note for the reviewer: Lambda node instead of an Agent node
> The original project brief called for a **Bedrock Agent node with an action
> group**. However, **Amazon Bedrock Agents (Classic) was closed to new
> customers on July 30, 2026** and placed in maintenance mode (confirmed by AWS
> documentation and by Udacity support, who directed students to AgentCore).
> Because a new account cannot create a Classic agent, and Bedrock Flows cannot
> currently invoke AgentCore agents, the bug-report capability was implemented
> with the equivalent **native Flow nodes**: a Prompt node (collects/extracts
> the details) + a **Lambda node** (persists the ticket). The Lambda function
> supports both the flow-style event and the legacy agent-style event, so it
> remains fully compatible with the original tool contract.
>
> **Equivalent evidence:** the Agent-node screenshot is replaced by the
> BugReportParser prompt config + the CreateBugReport Lambda-node config, and a
> DynamoDB item created through the flow.

### 4.2 Platform Question path

`FAQResponder` (Nova Lite) — answers questions **only** from the embedded FAQ
(`online_shop_faq.md`). The prompt includes the full FAQ text between
`===== FAQ =====` / `===== END FAQ =====` markers, plus the rule:

> If the FAQ does NOT cover the question, do not guess — politely say you can't
> answer and ask them to contact human support by phone at 1-800-555-0199.

This gives the *"redirect when the FAQ doesn't cover the question"* behavior.

### 4.3 Other Request path

`SupportRedirect` (Nova Micro) — politely acknowledges the customer, explains
the request can't be handled automatically, and always includes the phone line
`1-800-555-0199`.

---

## 5. Infrastructure (CloudFormation)

All resources live in **us-east-1**.

| Stack | Template | Creates |
|---|---|---|
| `bug-report-tool-stack` | `cloudformation-tool.yaml` | DynamoDB table (`BugReports-<suffix>`), Lambda `create-bug-report-<suffix>`, IAM role, Bedrock `lambda:InvokeFunction` permission |
| `bug-report-testing-stack` | `cloudformation-testing.yaml` | S3 bucket (`udacity-agentic-engineer-c1-eval-<acct>`), Bedrock evaluation IAM role (`bedrock-eval-role`) |

Deploy (CLI or console upload):

```bash
aws cloudformation deploy --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack --capabilities CAPABILITY_NAMED_IAM --region us-east-1

aws cloudformation deploy --template-file cloudformation-testing.yaml \
  --stack-name bug-report-testing-stack --capabilities CAPABILITY_NAMED_IAM --region us-east-1
```

> Note: resource names carry a per-lab random suffix (e.g. the Lambda deployed
> as `create-bug-report-741c3be0`) so the lab can be multi-tenant.

---

## 6. Testing & Evaluation

### 6.1 Automated test suite (`flow-tests.json`)

Five test prompts covering every path **plus** edge cases:

| id | Path | Prompt |
|---|---|---|
| `t1_bug_report_complete` | Bug | "The checkout page crashes every time I click the Pay button. I'm using Chrome 120 on Windows 11." |
| `t2_bug_report_minimal` | Bug (edge) | "My app keeps crashing when I try to upload a photo." |
| `t3_platform_question_covered` | Platform | "How do I track my order?" |
| `t4_platform_question_uncovered` | Platform | "Do you have a physical store I can visit?" |
| `t5_other_request` | Other | "Can you tell me a joke?" |

### 6.2 Run the flow programmatically

```bash
# 1. Create a flow alias in the console (e.g. "v1" → KNL0RR0T20)
# 2. Run the eval dataset generator
python generate-eval-dataset.py \
  --tests-json flow-tests.json \
  --flow-id 5JMUGWDI8U \
  --flow-alias-id KNL0RR0T20 \
  --out-jsonl output_eval_dataset.jsonl \
  --region us-east-1
```

Result: **5/5 flow calls succeeded**, producing `output_eval_dataset.jsonl`.

### 6.3 Bedrock Evaluations (LLM-as-a-judge, BYOI)

```bash
aws s3 cp output_eval_dataset.jsonl \
  s3://udacity-agentic-engineer-c1-eval-620831149724/output_eval_dataset.jsonl --region us-east-1

aws bedrock create-evaluation-job \
  --job-name flow-eval-run-1 \
  --role-arn arn:aws:iam::620831149724:role/bedrock-eval-role \
  --evaluation-config '{
    "automated": {
      "datasetMetricConfigs": [{
        "taskType": "General",
        "dataset": {
          "name": "flow-eval-dataset",
          "datasetLocation": {"s3Uri": "s3://udacity-agentic-engineer-c1-eval-620831149724/output_eval_dataset.jsonl"}
        },
        "metricNames": ["Builtin.Correctness"]
      }],
      "evaluatorModelConfig": {
        "bedrockEvaluatorModels": [{"modelIdentifier": "amazon.nova-pro-v1:0"}]
      }
    }
  }' \
  --inference-config '{"models": [{"precomputedInferenceSource": {"inferenceSourceIdentifier": "my-flow-app"}}]}' \
  --output-data-config '{"s3Uri": "s3://udacity-agentic-engineer-c1-eval-620831149724/results/"}' \
  --region us-east-1
```

### 6.4 Results & written observations

- **Overall `Builtin.Correctness` score: 1.0** (all 5 records judged correct by
  the evaluator model, Nova Pro).
- **Bug path** — both a complete report and a *minimal* report (missing steps/
  environment) produced a ticket confirmation with an ID and `OPEN` status.
  Trace confirmed `Classifier → RouteByCategory → BugReportParser →
  CreateBugReport → BugReportConfirm → BugReportOutput`, and the DynamoDB item
  was written with the correct fields.
- **Platform path (covered)** — the tracking question was answered accurately
  from the FAQ (tracking link emailed; also available under My Orders).
- **Platform path (uncovered)** — the store-location question could not be
  answered from the FAQ and was met with a polite phone redirect. (Observation:
  this particular message was classified as *Other Request* rather than
  *Platform Question*, but the resulting behavior — phone redirect — was still
  correct, which is why the judge scored it correct. A future tuning could add
  an uncovered-but-platform example to the classifier to push such messages
  through the FAQ path instead.)
- **Other path** — the joke request was politely declined and redirected to
  `1-800-555-0199`.

---

## 7. Evidence checklist (rubric mapping)

| Rubric criterion | Evidence |
|---|---|
| Classification & Routing | Full flow diagram · Classifier prompt config · Condition node expressions (exact match) · three Output nodes |
| Bug Report path | BugReportParser prompt · CreateBugReport Lambda-node config (tool attached) · flow test responses (ticket created; minimal report) · DynamoDB `BugReports` item created through the flow |
| Platform Question & Other paths | FAQResponder prompt with embedded FAQ · flow test responses for covered, uncovered, and other-request messages |
| Testing & Evaluation | `flow-tests.json` (all three paths) · `output_eval_dataset.jsonl` · Bedrock Evaluation job results (correctness = 1.0) · this README's written observations |

---

## 8. Stand-out items

Implemented as optional enhancements are noted where applicable; the core
submission intentionally keeps the reference-embedding approach specified by the
project:

- **Structured/constrained classifier output** — the classifier is prompt-
  constrained to a single label with no extraneous tokens, and the bug parser
  is constrained to strict JSON (with the Lambda defensively stripping any
  markdown fences the model may add).
- **Edge-case tests** — included: a minimal bug report with missing fields.
- **Future work:** Amazon Bedrock Guardrails for prompt-injection/harmful
  content, swapping the embedded FAQ for a Bedrock Knowledge Base (RAG), and a
  multi-turn "clarify before create" step for vague bug reports.

---

## 9. Cleanup (after submission)

```bash
aws s3 rm s3://udacity-agentic-engineer-c1-eval-620831149724 --recursive --region us-east-1
aws cloudformation delete-stack --stack-name bug-report-testing-stack --region us-east-1
aws cloudformation delete-stack --stack-name bug-report-tool-stack --region us-east-1
```

Then, in the Bedrock console: delete the flow and the evaluation job.

---

## 10. Project files

| File | Purpose |
|---|---|
| `create_bug_report.py` | Lambda that stores bug reports in DynamoDB |
| `online_shop_faq.md` | FAQ embedded in the Platform Question branch |
| `cloudformation-tool.yaml` | Deploys DynamoDB + Lambda + IAM |
| `cloudformation-testing.yaml` | Deploys S3 + evaluation IAM role |
| `generate-eval-dataset.py` | Runs the flow over test prompts → JSONL |
| `flow-tests-template.json` / `flow-tests.json` | Test suite |
| `output_eval_dataset.jsonl` | Generated evaluation dataset |
| `requirements.txt` | Python dependencies (boto3) |
