Classify and route customer feedback with GPT-4o, Gemini, and guardrails

https://n8nworkflows.xyz/workflows/classify-and-route-customer-feedback-with-gpt-4o--gemini--and-guardrails-13855


# Classify and route customer feedback with GPT-4o, Gemini, and guardrails

# 1. Workflow Overview

This workflow receives customer feedback through a webhook, normalizes the payload, screens the input with AI guardrails, classifies the feedback with an LLM, validates the generated structure, screens the drafted response again, and finally routes the item to a deterministic destination based on category and confidence.

Its main use case is automated triage of inbound customer feedback from a web form or similar source. It combines deterministic validation and routing with AI-based classification and drafting, while adding safety layers before and after model generation.

## 1.1 Intake & Normalization

The workflow starts with a webhook that accepts POST requests, then transforms inconsistent incoming field names into a standardized internal schema. It also checks whether the minimum required information is present.

## 1.2 Input Guardrails

Before the feedback reaches the main AI classifier, the workflow sends the normalized feedback text through a guardrails node. This is intended to detect PII, jailbreak attempts, and secret-key-like content.

## 1.3 AI Classification & Drafting

If the input is allowed, an AI Agent classifies the feedback and drafts a suggested response. The prompt forces the model to return JSON only, with a fixed list of fields and allowed categories.

## 1.4 Output Validation & Guardrails

The workflow validates the AI output using a Code node that parses JSON, checks required fields, and enforces allowed enum values. The drafted response is then screened with a second guardrails node.

## 1.5 Confidence Gate & Routing

Only outputs that are both valid and above a confidence threshold are routed automatically. Routing is category-based:
- `bug_report` / product issue path -> engineering
- `feature_request` -> product backlog
- `complaint` -> customer success escalation
- `praise` -> marketing/testimonial flag

Items with low confidence or validation failures are sent to a human-review queue.

---

# 2. Block-by-Block Analysis

## 2.1 Intake & Normalize

### Overview

This block receives external feedback data, maps inconsistent input fields into a normalized structure, and checks whether the required fields are present. It acts as the entry gate for the workflow and ensures downstream nodes receive predictable data.

### Nodes Involved

- Webhook - Feedback Intake
- Normalize Feedback
- Has Required Fields?
- Respond - Missing Fields

### Node Details

#### Webhook - Feedback Intake

- **Type and technical role:** `n8n-nodes-base.webhook`; workflow entry point for HTTP POST requests.
- **Configuration choices:**
  - Path: `customer-feedback`
  - HTTP Method: `POST`
  - Response Mode: `responseNode`, meaning the workflow expects dedicated Respond to Webhook nodes later.
- **Key expressions or variables used:** None in the node itself.
- **Input and output connections:**
  - No input; it is the trigger.
  - Outputs to `Normalize Feedback`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Wrong HTTP method.
  - Incorrect payload format.
  - Missing `body` object in the inbound request, which would affect normalization logic.
  - If no response node is reached, the webhook request may hang or fail.
- **Sub-workflow reference:** None.

#### Normalize Feedback

- **Type and technical role:** `n8n-nodes-base.code`; standardizes raw input into a canonical feedback object.
- **Configuration choices:**
  - Reads from `$input.first().json.body`.
  - Builds:
    - `feedbackId` as `FB-` + current timestamp
    - `customerName` from `name` or `fullName`, default `Anonymous`
    - `customerEmail` from `email`
    - `feedbackText` from `feedback`, `message`, or `body`
    - `product` from `product` or `service`, default `general`
    - `source` default `web`
    - `receivedAt` as current ISO timestamp
  - Adds boolean `hasRequiredFields`.
- **Key expressions or variables used:**
  - `$input.first().json.body`
  - JavaScript normalization logic
- **Input and output connections:**
  - Input from `Webhook - Feedback Intake`
  - Output to `Has Required Fields?`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - If `json.body` is undefined, `raw.name` access will throw.
  - Non-string values for fields may break `.trim()` unless coerced.
  - Timestamp-based `feedbackId` is simple but not collision-proof in high-throughput parallel calls.
- **Sub-workflow reference:** None.

#### Has Required Fields?

- **Type and technical role:** `n8n-nodes-base.if`; checks that required normalized fields are present.
- **Configuration choices:**
  - Condition checks whether `$json.hasRequiredFields` is `true`.
  - This means only email and feedback text are mandatory.
- **Key expressions or variables used:**
  - `={{ $json.hasRequiredFields }}`
- **Input and output connections:**
  - Input from `Normalize Feedback`
  - True output to `Input Guardrails`
  - False output to `Respond - Missing Fields`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If upstream code fails before setting `hasRequiredFields`, this condition may evaluate unexpectedly.
  - Business rule mismatch: response says “Email and feedback text are required” while `customerName` and `product` are optional.
- **Sub-workflow reference:** None.

#### Respond - Missing Fields

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns a 400 error for incomplete input.
- **Configuration choices:**
  - Response code: `400`
  - Response format: JSON
  - Body returns:
    - `status: error`
    - `message: Email and feedback text are required`
- **Key expressions or variables used:**
  - JSON body built with expression syntax.
- **Input and output connections:**
  - Input from false branch of `Has Required Fields?`
  - No downstream output
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - If reached after partial data corruption, still returns generic missing-field message.
- **Sub-workflow reference:** None.

---

## 2.2 Input Guardrails

### Overview

This block evaluates inbound feedback text before it reaches the classifier. It is designed to reduce unsafe or manipulative inputs by detecting PII, prompt-injection/jailbreak patterns, and secret-like content.

### Nodes Involved

- Input Guardrails
- Input Guardrails LLM
- Respond - Input Blocked

### Node Details

#### Input Guardrails

- **Type and technical role:** `@n8n/n8n-nodes-langchain.guardrails`; policy screening layer for the user-provided feedback text.
- **Configuration choices:**
  - Text to inspect: `{{$json.feedbackText}}`
  - Enabled checks:
    - PII: all
    - Jailbreak threshold: `0.8`
    - Secret key detection: permissiveness `balanced`
- **Key expressions or variables used:**
  - `={{ $json.feedbackText }}`
- **Input and output connections:**
  - Input from `Has Required Fields?` true branch
  - Main output 0 to `AI - Classify + Draft`
  - Main output 1 to `Respond - Input Blocked`
  - AI language model input from `Input Guardrails LLM`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Credential/model failure on the attached LLM.
  - False positives on PII or jailbreak detection.
  - Empty or extremely short feedback text may produce unstable screening behavior.
- **Sub-workflow reference:** None.

#### Input Guardrails LLM

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model backend used by the input guardrails node.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Uses OpenAI credentials named `OpenAi account 2`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No main input
  - Connected via `ai_languageModel` to `Input Guardrails`
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Invalid or missing OpenAI API credentials.
  - Model availability, quota exhaustion, or rate limiting.
- **Sub-workflow reference:** None.

#### Respond - Input Blocked

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns a blocked-input response when guardrails reject the message.
- **Configuration choices:**
  - Response code: `400`
  - JSON body:
    - `status: input_blocked`
    - `message: Your message could not be processed. Please rephrase and try again.`
- **Key expressions or variables used:** Static JSON expression.
- **Input and output connections:**
  - Input from second output of `Input Guardrails`
  - No downstream output
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - The message is generic and does not indicate which guardrail triggered.
- **Sub-workflow reference:** None.

---

## 2.3 AI Classification & Drafting

### Overview

This block sends the normalized, guardrail-approved feedback to an AI Agent that classifies the feedback and drafts a suggested response. The prompt requests strict JSON and constrains category values.

### Nodes Involved

- AI - Classify + Draft
- OpenRouter Chat Model1

### Node Details

#### AI - Classify + Draft

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; LLM-driven classification and response-drafting node.
- **Configuration choices:**
  - Prompt type: `define`
  - Prompt includes:
    - Customer name
    - Product
    - Feedback text
  - Requires output fields:
    - `sentiment`
    - `category`
    - `priority`
    - `suggestedResponse`
    - `confidence`
  - Allowed categories in prompt:
    - `bug_report`
    - `feature_request`
    - `praise`
    - `complaint`
    - `question`
- **Key expressions or variables used:**
  - `{{ $('Has Required Fields?').item.json.customerName }}`
  - `{{ $('Has Required Fields?').item.json.product }}`
  - `{{ $('Has Required Fields?').item.json.feedbackText }}`
- **Input and output connections:**
  - Input from `Input Guardrails`
  - Main output to `Validate AI Output`
  - AI language model input from `OpenRouter Chat Model1`
- **Version-specific requirements:** Type version `1.7`.
- **Edge cases or potential failure types:**
  - Model may return non-JSON or fenced JSON despite instructions.
  - Model may emit unsupported enum values.
  - `confidence` is requested in the prompt but not required by validation logic; this can later affect routing.
  - Prompt/category mismatch with downstream switch logic.
- **Sub-workflow reference:** None.

#### OpenRouter Chat Model1

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`; model backend for the classifier agent.
- **Configuration choices:**
  - Model: `google/gemini-3-flash-preview`
  - Uses OpenRouter credentials named `OpenRouter account 2`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No main input
  - Connected via `ai_languageModel` to `AI - Classify + Draft`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Invalid OpenRouter credentials.
  - Model changes or preview-model instability.
  - Rate limits or provider-side timeout.
- **Sub-workflow reference:** None.

---

## 2.4 Output Validation

### Overview

This block parses and validates the AI response deterministically, then applies output guardrails to the drafted response text. It prevents malformed or policy-unsafe output from being routed automatically.

### Nodes Involved

- Validate AI Output
- Output Guardrails
- OpenRouter Chat Model
- Respond - Output Blocked

### Node Details

#### Validate AI Output

- **Type and technical role:** `n8n-nodes-base.code`; deterministic parser and schema validator for the AI result.
- **Configuration choices:**
  - Runs once for each item.
  - Reads `input.output`.
  - If output is a string:
    - strips Markdown code fences
    - parses JSON
  - Requires fields:
    - `sentiment`
    - `category`
    - `priority`
    - `suggestedResponse`
  - Valid enum values:
    - Sentiment: `positive`, `negative`, `neutral`, `mixed`
    - Category: `bug_report`, `feature_request`, `praise`, `complaint`, `question`
    - Priority: `critical`, `high`, `medium`, `low`
  - Returns `validationPassed: true/false`
  - Adds `validationError` on failure
- **Key expressions or variables used:**
  - `$json`
  - `input.output`
- **Input and output connections:**
  - Input from `AI - Classify + Draft`
  - Output to `Output Guardrails`
- **Version-specific requirements:** Code node type version `2`, mode `runOnceForEachItem`.
- **Edge cases or potential failure types:**
  - If the agent output format changes and `output` no longer exists, validation may fail or parse the wrong structure.
  - `confidence` is not validated as required even though downstream logic expects it.
  - No numeric range check for confidence.
  - Invalid JSON is converted to a structured validation failure rather than hard-stopping.
- **Sub-workflow reference:** None.

#### Output Guardrails

- **Type and technical role:** `@n8n/n8n-nodes-langchain.guardrails`; screens the drafted response text before it is returned or used for routing.
- **Configuration choices:**
  - Text to inspect: `{{$json.suggestedResponse}}`
  - Enabled checks:
    - NSFW threshold: `0.8`
    - Secret key detection: permissiveness `balanced`
- **Key expressions or variables used:**
  - `={{ $json.suggestedResponse }}`
- **Input and output connections:**
  - Input from `Validate AI Output`
  - Main output 0 to `High Confidence + Valid?`
  - Main output 1 to `Respond - Output Blocked`
  - AI language model input from `OpenRouter Chat Model`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If `suggestedResponse` is missing because validation already failed, guardrails behavior may depend on node implementation.
  - False positives can block harmless outputs.
- **Sub-workflow reference:** None.

#### OpenRouter Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`; model backend for output guardrails.
- **Configuration choices:**
  - Model: `google/gemini-3-flash-preview`
  - Uses OpenRouter credentials named `OpenRouter account 2`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No main input
  - Connected via `ai_languageModel` to `Output Guardrails`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Credential, quota, or provider errors.
- **Sub-workflow reference:** None.

#### Respond - Output Blocked

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns an error when output guardrails reject the generated response.
- **Configuration choices:**
  - Response code: `400`
  - JSON body:
    - `status: input_blocked`
    - `message: Your message could not be processed. Please rephrase and try again.`
- **Key expressions or variables used:** Static JSON expression.
- **Input and output connections:**
  - Input from second output of `Output Guardrails`
  - No downstream output
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Status label says `input_blocked` even though this is an output-stage rejection.
- **Sub-workflow reference:** None.

---

## 2.5 Confidence Gate & Routing

### Overview

This block decides whether the AI result is trusted enough to route automatically. Approved outputs are routed by category to placeholder actions, while low-confidence or invalid items are queued for human review.

### Nodes Involved

- High Confidence + Valid?
- Route by Type
- Create Jira Ticket
- Add to Backlog
- Escalate to CS
- Flag as Testimonial
- Queue for Human Review
- Respond - Result

### Node Details

#### High Confidence + Valid?

- **Type and technical role:** `n8n-nodes-base.if`; automation gate based on confidence and validation status.
- **Configuration choices:**
  - Checks:
    - `confidence >= 0.7`
    - `validationPassed = true`
- **Key expressions or variables used:**
  - `={{ $('Validate AI Output').item.json.confidence }}`
  - `={{ $('Validate AI Output').item.json.validationPassed }}`
- **Input and output connections:**
  - Input from `Output Guardrails`
  - True output to `Route by Type`
  - False output to `Queue for Human Review`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If `confidence` is missing or not numeric, the numeric comparison may fail or route unexpectedly.
  - Validation failure still proceeds to this node; false branch handles it.
- **Sub-workflow reference:** None.

#### Route by Type

- **Type and technical role:** `n8n-nodes-base.switch`; category-based deterministic routing.
- **Configuration choices:**
  - Uses category from `Validate AI Output`
  - Named outputs:
    - `Product Issue` when category equals `product_issue`
    - `Feature Request` when category equals `feature_request`
    - `Complaint` when category equals `complaint`
    - `Praise` when category equals `praise`
  - Fallback output enabled as output index 2
- **Key expressions or variables used:**
  - `={{ $('Validate AI Output').item.json.category }}`
- **Input and output connections:**
  - Input from `High Confidence + Valid?`
  - Output 0 -> `Create Jira Ticket`
  - Output 1 -> `Add to Backlog`
  - Output 2 -> `Escalate to CS`
  - Output 3 -> `Flag as Testimonial`
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:**
  - Major logic mismatch: validator allows `bug_report`, but this switch expects `product_issue`.
  - Because fallback output is set to index 2, unmatched categories such as `bug_report` and `question` will go to the `Complaint` branch (`Escalate to CS`), which is likely unintended.
  - `question` has no explicit route.
- **Sub-workflow reference:** None.

#### Create Jira Ticket

- **Type and technical role:** `n8n-nodes-base.code`; placeholder for engineering ticket creation.
- **Configuration choices:**
  - Returns original validated payload plus:
    - `action: jira_ticket_created`
    - `destination: engineering`
- **Key expressions or variables used:**
  - `$('Validate AI Output').first().json`
- **Input and output connections:**
  - Input from `Route by Type`
  - Output to `Respond - Result`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - Currently no real Jira integration.
  - Uses `.first()` from `Validate AI Output`, which is acceptable in single-item execution but can be problematic if batching changes later.
- **Sub-workflow reference:** None.

#### Add to Backlog

- **Type and technical role:** `n8n-nodes-base.code`; placeholder for product backlog routing.
- **Configuration choices:**
  - Returns original validated payload plus:
    - `action: added_to_backlog`
    - `destination: product`
- **Key expressions or variables used:**
  - `$('Validate AI Output').first().json`
- **Input and output connections:**
  - Input from `Route by Type`
  - Output to `Respond - Result`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - Placeholder only; no actual tracker integration.
- **Sub-workflow reference:** None.

#### Escalate to CS

- **Type and technical role:** `n8n-nodes-base.code`; placeholder for customer success escalation.
- **Configuration choices:**
  - Returns original validated payload plus:
    - `action: escalated_to_cs`
    - `destination: customer_success`
    - `priority: high`
- **Key expressions or variables used:**
  - `$('Validate AI Output').first().json`
- **Input and output connections:**
  - Input from `Route by Type`
  - Output to `Respond - Result`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - Overwrites the AI-generated priority with `high`.
  - Because of switch fallback behavior, unmatched categories may land here unintentionally.
- **Sub-workflow reference:** None.

#### Flag as Testimonial

- **Type and technical role:** `n8n-nodes-base.code`; placeholder for marketing routing of praise.
- **Configuration choices:**
  - Returns original validated payload plus:
    - `action: testimonial_candidate`
    - `destination: marketing`
- **Key expressions or variables used:**
  - `$('Validate AI Output').first().json`
- **Input and output connections:**
  - Input from `Route by Type`
  - Output to `Respond - Result`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - Placeholder only; no CRM/marketing integration.
- **Sub-workflow reference:** None.

#### Queue for Human Review

- **Type and technical role:** `n8n-nodes-base.code`; fallback path for low-confidence or invalid AI outputs.
- **Configuration choices:**
  - Returns original validated payload plus:
    - `action: needs_human_review`
    - `reason: low_confidence_or_validation_failure`
- **Key expressions or variables used:**
  - `$('Validate AI Output').first().json`
- **Input and output connections:**
  - Input from false branch of `High Confidence + Valid?`
  - Output to `Respond - Result`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - No actual queue system is connected yet.
  - Reason is generic; does not distinguish confidence failure from schema failure.
- **Sub-workflow reference:** None.

#### Respond - Result

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; final success/fallback response to the caller.
- **Configuration choices:**
  - Responds with JSON body containing:
    - `feedbackId`
    - `category`
    - `action`
    - `destination`
    - `confidence`
- **Key expressions or variables used:**
  - `{{ JSON.stringify({ feedbackId: $json.feedbackId, category: $json.category, action: $json.action, destination: $json.destination, confidence: $json.confidence }) }}`
- **Input and output connections:**
  - Inputs from:
    - `Create Jira Ticket`
    - `Add to Backlog`
    - `Escalate to CS`
    - `Flag as Testimonial`
    - `Queue for Human Review`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - If upstream action nodes omit `destination`, response may contain undefined values.
  - Human review path does not set `destination`, so it may be absent in the final JSON.
- **Sub-workflow reference:** None.

---

## 2.6 Documentation / Visual Annotation Nodes

### Overview

These nodes do not affect execution. They provide in-canvas explanations and setup guidance.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation.
- **Configuration choices:** Contains a full overview of the pipeline, setup instructions, and customization notes.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1

- **Type and technical role:** Sticky note; labels the intake section.
- **Configuration choices:** Content `## Intake & Normalize`
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2

- **Type and technical role:** Sticky note; labels input guardrails section.
- **Configuration choices:** Content `## Input Guardrails`
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3

- **Type and technical role:** Sticky note; labels AI classification section.
- **Configuration choices:** Content `## AI Classification & Drafting`
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4

- **Type and technical role:** Sticky note; labels output validation section.
- **Configuration choices:** Content `## Output Validation`
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5

- **Type and technical role:** Sticky note; labels routing section.
- **Configuration choices:** Content `## Route by Type`
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Feedback Intake | Webhook | Receives POSTed customer feedback |  | Normalize Feedback | ## Intake & Normalize |
| Normalize Feedback | Code | Normalizes inbound payload into a standard schema | Webhook - Feedback Intake | Has Required Fields? | ## Intake & Normalize |
| Has Required Fields? | IF | Checks whether email and feedback text exist | Normalize Feedback | Input Guardrails; Respond - Missing Fields | ## Intake & Normalize |
| Respond - Missing Fields | Respond to Webhook | Returns 400 for incomplete payloads | Has Required Fields? |  | ## Intake & Normalize |
| Input Guardrails | Guardrails | Screens feedback text for PII, jailbreak, and secret-like content | Has Required Fields? | AI - Classify + Draft; Respond - Input Blocked | ## Input Guardrails |
| Input Guardrails LLM | OpenAI Chat Model | Supplies LLM backend for input guardrails |  | Input Guardrails | ## Input Guardrails |
| Respond - Input Blocked | Respond to Webhook | Returns 400 when inbound text is blocked by guardrails | Input Guardrails |  | ## Input Guardrails |
| AI - Classify + Draft | LangChain Agent | Classifies feedback and drafts a response | Input Guardrails | Validate AI Output | ## AI Classification & Drafting |
| OpenRouter Chat Model1 | OpenRouter Chat Model | Supplies Gemini model backend for classification agent |  | AI - Classify + Draft | ## AI Classification & Drafting |
| Validate AI Output | Code | Parses and validates AI JSON output | AI - Classify + Draft | Output Guardrails | ## Output Validation |
| Output Guardrails | Guardrails | Screens suggested response text before routing | Validate AI Output | High Confidence + Valid?; Respond - Output Blocked | ## Output Validation |
| OpenRouter Chat Model | OpenRouter Chat Model | Supplies Gemini model backend for output guardrails |  | Output Guardrails | ## Output Validation |
| Respond - Output Blocked | Respond to Webhook | Returns 400 when generated output is blocked | Output Guardrails |  | ## Output Validation |
| High Confidence + Valid? | IF | Allows auto-routing only when validation passes and confidence is high enough | Output Guardrails | Route by Type; Queue for Human Review | ## Route by Type |
| Route by Type | Switch | Routes approved items by feedback category | High Confidence + Valid? | Create Jira Ticket; Add to Backlog; Escalate to CS; Flag as Testimonial | ## Route by Type |
| Create Jira Ticket | Code | Placeholder action for engineering routing | Route by Type | Respond - Result | ## Route by Type |
| Add to Backlog | Code | Placeholder action for product backlog routing | Route by Type | Respond - Result | ## Route by Type |
| Escalate to CS | Code | Placeholder action for customer success escalation | Route by Type | Respond - Result | ## Route by Type |
| Flag as Testimonial | Code | Placeholder action for marketing/testimonial routing | Route by Type | Respond - Result | ## Route by Type |
| Queue for Human Review | Code | Placeholder fallback for low-confidence or invalid results | High Confidence + Valid? | Respond - Result | ## Route by Type |
| Respond - Result | Respond to Webhook | Returns final routing outcome to caller | Create Jira Ticket; Add to Backlog; Escalate to CS; Flag as Testimonial; Queue for Human Review |  | ## Route by Type |
| Sticky Note | Sticky Note | General in-canvas documentation |  |  | ## Complete Customer Feedback Pipeline<br>### How it works<br>This workflow chains all five stages of the hybrid deterministic + AI pattern:<br>1. **Intake** (Deterministic): Webhook receives feedback, Code node normalizes it, IF node validates required fields.<br>2. **Input Guardrails** (Deterministic): Guardrails node checks for PII, injection attempts, and blocked keywords. Flagged inputs are blocked.<br>3. **AI Classification + Drafting** (AI): AI Agent classifies the feedback (product issue, feature request, complaint, praise) and drafts a personalized response.<br>4. **Output Validation** (Deterministic): Code node validates the classification. Guardrails node checks the draft for policy violations.<br>5. **Routing** (Deterministic): High-confidence, valid results route by type: product issues create Jira tickets, feature requests go to backlog, complaints escalate to CS, and praise is flagged for testimonials. Low-confidence results queue for human review.<br><br>### Setup<br>- Connect your **LLM credentials** to the Chat Model nodes (AI Agent + both Guardrails nodes)<br>- Copy the Webhook test URL and send sample feedback with varying tone and content<br>- Review the **Route by Type** Switch node to map categories to your actual team integrations<br><br>### Customization<br>- Replace placeholder Code nodes (Jira, Backlog, Escalate, Testimonial) with real integrations<br>- Tune confidence thresholds in the **High Confidence + Valid?** node as you validate accuracy<br>- Add more feedback categories by extending the AI prompt and the Route by Type switch |
| Sticky Note1 | Sticky Note | Section label |  |  | ## Intake & Normalize |
| Sticky Note2 | Sticky Note | Section label |  |  | ## Input Guardrails |
| Sticky Note3 | Sticky Note | Section label |  |  | ## AI Classification & Drafting |
| Sticky Note4 | Sticky Note | Section label |  |  | ## Output Validation |
| Sticky Note5 | Sticky Note | Section label |  |  | ## Route by Type |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it the title:  
   `Classify and route customer feedback with GPT-4o, Gemini, and guardrails`.

2. **Add a Webhook node** named `Webhook - Feedback Intake`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `customer-feedback`.
   - Set **Response Mode** to `Using Respond to Webhook Node` / `responseNode`.

3. **Add a Code node** named `Normalize Feedback` after the webhook.
   - Connect `Webhook - Feedback Intake -> Normalize Feedback`.
   - Paste logic that:
     - reads `body` from the incoming webhook payload
     - creates:
       - `feedbackId`
       - `customerName`
       - `customerEmail`
       - `feedbackText`
       - `product`
       - `source`
       - `receivedAt`
     - creates `hasRequiredFields`
   - Use this logic behavior:
     - `name` or `fullName` -> customer name
     - `email` -> customer email
     - `feedback`, `message`, or `body` -> feedback text
     - `product` or `service` -> product
     - defaults: `Anonymous`, `general`, `web`

4. **Add an IF node** named `Has Required Fields?`.
   - Connect `Normalize Feedback -> Has Required Fields?`.
   - Configure a boolean condition:
     - left value: `{{$json.hasRequiredFields}}`
     - operation: `is true`

5. **Add a Respond to Webhook node** named `Respond - Missing Fields`.
   - Connect the **false** output of `Has Required Fields?` to it.
   - Set **Response Code** to `400`.
   - Set response format to JSON.
   - Return:
     - `status: error`
     - `message: Email and feedback text are required`

6. **Add a Guardrails node** named `Input Guardrails`.
   - Connect the **true** output of `Has Required Fields?` to it.
   - Set the text field to `{{$json.feedbackText}}`.
   - Enable:
     - PII detection: all
     - Jailbreak detection threshold: `0.8`
     - Secret keys detection: `balanced`

7. **Add an OpenAI Chat Model node** named `Input Guardrails LLM`.
   - Connect it to the `Input Guardrails` node using the AI language model connection.
   - Select model `gpt-4o`.
   - Configure valid OpenAI credentials.

8. **Add a Respond to Webhook node** named `Respond - Input Blocked`.
   - Connect the blocked/rejected output of `Input Guardrails` to it.
   - Set **Response Code** to `400`.
   - Return JSON:
     - `status: input_blocked`
     - `message: Your message could not be processed. Please rephrase and try again.`

9. **Add a LangChain Agent node** named `AI - Classify + Draft`.
   - Connect the allowed output of `Input Guardrails` to it.
   - Set prompt mode to define text manually.
   - Use a prompt equivalent to:
     - include customer name, product, and feedback text
     - instruct the model to return only valid JSON
     - require these fields:
       - `sentiment`
       - `category`
       - `priority`
       - `suggestedResponse`
       - `confidence`
     - restrict categories to:
       - `bug_report`
       - `feature_request`
       - `praise`
       - `complaint`
       - `question`

10. **Add an OpenRouter Chat Model node** named `OpenRouter Chat Model1`.
    - Connect it as the AI language model for `AI - Classify + Draft`.
    - Set model to `google/gemini-3-flash-preview`.
    - Configure valid OpenRouter credentials.

11. **Add a Code node** named `Validate AI Output`.
    - Connect `AI - Classify + Draft -> Validate AI Output`.
    - Set execution mode to **Run Once for Each Item**.
    - Implement logic that:
      - reads the agent result from `output`
      - strips code fences if present
      - parses JSON
      - verifies required fields:
        - `sentiment`
        - `category`
        - `priority`
        - `suggestedResponse`
      - verifies enum values:
        - sentiment in `positive`, `negative`, `neutral`, `mixed`
        - category in `bug_report`, `feature_request`, `praise`, `complaint`, `question`
        - priority in `critical`, `high`, `medium`, `low`
      - returns `validationPassed: true` or `false`
      - returns `validationError` on failure

12. **Add a Guardrails node** named `Output Guardrails`.
    - Connect `Validate AI Output -> Output Guardrails`.
    - Set text to `{{$json.suggestedResponse}}`.
    - Enable:
      - NSFW threshold: `0.8`
      - Secret keys detection: `balanced`

13. **Add an OpenRouter Chat Model node** named `OpenRouter Chat Model`.
    - Connect it as the AI language model for `Output Guardrails`.
    - Use model `google/gemini-3-flash-preview`.
    - Reuse the same OpenRouter credentials if desired.

14. **Add a Respond to Webhook node** named `Respond - Output Blocked`.
    - Connect the blocked/rejected output of `Output Guardrails` to it.
    - Set **Response Code** to `400`.
    - Return the same generic blocked message as above.

15. **Add an IF node** named `High Confidence + Valid?`.
    - Connect the allowed output of `Output Guardrails` to it.
    - Add two AND conditions:
      - `{{$('Validate AI Output').item.json.confidence}} >= 0.7`
      - `{{$('Validate AI Output').item.json.validationPassed}} is true`

16. **Add a Switch node** named `Route by Type`.
    - Connect the true output of `High Confidence + Valid?` to it.
    - Create explicit outputs for category checks based on `{{$('Validate AI Output').item.json.category}}`.
    - In the original workflow, the rules are:
      - `product_issue`
      - `feature_request`
      - `complaint`
      - `praise`
    - Important: this is inconsistent with the AI prompt and validator, which use `bug_report`, not `product_issue`.
    - To reproduce exactly, keep the mismatch.
    - To improve it, replace `product_issue` with `bug_report`.
    - Also add an explicit route for `question` if you want correct handling.

17. **Set fallback behavior** on `Route by Type`.
    - The original workflow enables fallback output and points unmatched categories to output index 2, which effectively sends fallback items to the complaint/customer-success branch.
    - Reproduce this only if you want exact parity.
    - Recommended fix: route fallback to human review instead.

18. **Add a Code node** named `Create Jira Ticket`.
    - Connect the first switch output to it.
    - Return the validated payload plus:
      - `action: jira_ticket_created`
      - `destination: engineering`
    - In a production build, replace this with a real Jira node and ticket mapping.

19. **Add a Code node** named `Add to Backlog`.
    - Connect the feature-request output to it.
    - Return the validated payload plus:
      - `action: added_to_backlog`
      - `destination: product`

20. **Add a Code node** named `Escalate to CS`.
    - Connect the complaint output to it.
    - Return the validated payload plus:
      - `action: escalated_to_cs`
      - `destination: customer_success`
      - `priority: high`

21. **Add a Code node** named `Flag as Testimonial`.
    - Connect the praise output to it.
    - Return the validated payload plus:
      - `action: testimonial_candidate`
      - `destination: marketing`

22. **Add a Code node** named `Queue for Human Review`.
    - Connect the false output of `High Confidence + Valid?` to it.
    - Return the validated payload plus:
      - `action: needs_human_review`
      - `reason: low_confidence_or_validation_failure`

23. **Add a Respond to Webhook node** named `Respond - Result`.
    - Connect all final action nodes to it:
      - `Create Jira Ticket`
      - `Add to Backlog`
      - `Escalate to CS`
      - `Flag as Testimonial`
      - `Queue for Human Review`
    - Configure JSON response returning:
      - `feedbackId`
      - `category`
      - `action`
      - `destination`
      - `confidence`

24. **Add sticky notes** if you want the same visual organization.
    - Add one overall note describing the five-stage pipeline and setup guidance.
    - Add section labels:
      - `## Intake & Normalize`
      - `## Input Guardrails`
      - `## AI Classification & Drafting`
      - `## Output Validation`
      - `## Route by Type`

25. **Configure credentials**
    - **OpenAI credential**
      - Required by `Input Guardrails LLM`
      - Must support `gpt-4o`
    - **OpenRouter credential**
      - Required by `OpenRouter Chat Model1`
      - Required by `OpenRouter Chat Model`
      - Must allow `google/gemini-3-flash-preview`

26. **Test with sample POST payloads** such as:
    - valid bug report
    - feature request
    - praise/testimonial candidate
    - complaint
    - incomplete payload missing email
    - content likely to trigger guardrails

27. **Recommended fixes after reproduction**
    - Align `bug_report` vs `product_issue`
    - Validate `confidence` explicitly as required numeric output
    - Add a defined route for `question`
    - Send fallback switch results to human review instead of complaint escalation
    - Replace placeholder Code nodes with Jira, Linear, Slack, Zendesk, email, or CRM integrations

### Sub-workflow setup

This workflow does **not** invoke any sub-workflows and has only one entry point:
- `Webhook - Feedback Intake`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Complete Customer Feedback Pipeline: intake, input guardrails, AI classification and drafting, output validation, and routing | In-canvas project note |
| Connect your LLM credentials to the Chat Model nodes for the AI Agent and both Guardrails nodes | Setup guidance |
| Copy the Webhook test URL and send sample feedback with varying tone and content | Testing guidance |
| Review the Route by Type Switch node to map categories to your actual team integrations | Customization guidance |
| Replace placeholder Code nodes for Jira, backlog, escalation, and testimonial routing with real integrations | Production hardening |
| Tune confidence thresholds in the High Confidence + Valid? node as you validate accuracy | Model calibration |
| Add more feedback categories by extending the AI prompt and the Route by Type switch | Functional extension |

## Important implementation notes

- The workflow title and visual notes describe routing for “product issues,” but the classifier and validator use `bug_report`. This mismatch is the most important logic issue in the current design.
- The `Route by Type` fallback configuration sends unmatched categories to the complaint branch, which can misroute `bug_report` and `question`.
- The final blocked-output response currently uses `status: input_blocked`, which is semantically misleading for output-stage failures.
- `confidence` is used for routing but is not enforced by the validator; this should be tightened if the workflow is used in production.