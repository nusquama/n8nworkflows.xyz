Moderate and govern user content with OpenAI GPT-4o, Slack and Gmail

https://n8nworkflows.xyz/workflows/moderate-and-govern-user-content-with-openai-gpt-4o--slack-and-gmail-13447


# Moderate and govern user content with OpenAI GPT-4o, Slack and Gmail

## 1. Workflow Overview

**Workflow name:** AI-driven content moderation and governance workflow  
**Purpose:** Automates end-to-end moderation of user-generated content using OpenAI (GPT‑4o) to validate content signals, orchestrate governance decisions (review/appeal/enforce), trigger escalation via Slack and email for critical cases, and write audit-ready logs to n8n Data Tables.

**Typical use cases**
- Marketplace listing moderation
- Community post/comment moderation
- Enterprise HR/content screening pipelines
- Any Trust & Safety / Governance process needing consistent, structured decisions and traceability

### 1.1 Ingest & Configuration
Receives moderation requests via an HTTP webhook and sets runtime configuration variables (threshold, escalation email, Slack channel, severity list).

### 1.2 Content Signal Validation (AI)
An AI agent checks the submitted content plus “moderation flags”, produces a structured assessment (status/severity/confidence), then routes processing by validation status.

### 1.3 Governance Orchestration (AI + Tools)
For flagged/suspicious content, a second AI agent decides the governance action type (REVIEW / APPEAL / ENFORCE / ESCALATE) and can invoke specialized agent tools (review, appeals, enforcement), each backed by GPT‑4o and structured outputs.

### 1.4 Enforcement, Storage, and Escalation
Routes by action type, stores records to Data Tables, and escalates critical enforcement actions by notifying Slack and sending an email.

### 1.5 Audit & Logging
Merges outputs from multiple paths, normalizes them in a Code node, and writes a comprehensive record to a moderation audit log Data Table.

---

## 2. Block-by-Block Analysis

### Block 1 — Ingest & Validate (Webhook + Config + Initial AI validation)
**Overview:** Accepts incoming moderation requests, injects configuration values, and runs an AI “Content Signal” validation to decide if content is valid, flagged, or suspicious.

**Nodes involved**
- Content Moderation Webhook
- Workflow Configuration
- Content Signal Agent
- OpenAI Model - Content Signal
- Structured Output - Content Signal
- Route by Validation Status

#### Node: Content Moderation Webhook
- **Type / role:** `Webhook` — entry point for POST requests.
- **Configuration choices:**
  - **Path:** `content-moderation`
  - **Method:** `POST`
  - **Response mode:** “Respond via Response Node” (`responseMode: responseNode`) — implies you should add a Response node if you want an HTTP response body; this workflow currently does not include one.
- **Expected input shape (implied by expressions):**
  - `body.content` (string)
  - `body.userId` (string/number)
  - `body.contentType` (string)
  - `body.moderationFlags` (object/array)
- **Outputs to:** Workflow Configuration
- **Failure/edge cases:**
  - Requests missing `body.content` etc. will cause downstream expressions to evaluate to `undefined` and degrade AI output or switch routing.
  - With `responseMode: responseNode` and no Response node, callers may not receive a proper response (depends on n8n version/behavior).

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes runtime parameters.
- **Configuration choices:**
  - Adds:
    - `moderationThreshold` = `0.75` (currently not used elsewhere in the workflow)
    - `escalationEmail` = placeholder string
    - `slackChannelId` = placeholder string
    - `severityLevels` = `["LOW","MEDIUM","HIGH","CRITICAL"]`
  - Keeps other incoming fields (`includeOtherFields: true`) so the webhook payload remains available downstream.
- **Outputs to:** Content Signal Agent
- **Failure/edge cases:**
  - Placeholders must be replaced or Slack/Email steps will fail (invalid destination).

#### Node: Content Signal Agent
- **Type / role:** `LangChain Agent` — produces a structured validation decision based on content + flags.
- **Key configuration:**
  - **User message (constructed text):** includes content, userId, contentType, and JSON stringified moderationFlags.
  - **System message:** defines responsibilities and output requirements.
  - **Has output parser:** yes (paired with Structured Output - Content Signal).
- **AI dependencies:**
  - Uses **OpenAI Model - Content Signal** as language model.
  - Uses **Structured Output - Content Signal** to constrain output.
- **Outputs to:** Route by Validation Status
- **Failure/edge cases:**
  - If `moderationFlags` contains circular structures, `JSON.stringify()` would fail (rare unless upstream sends invalid JSON-like objects; typically safe with webhook JSON).
  - Model may still return malformed JSON; structured parser reduces risk but can still fail on extreme outputs.

#### Node: OpenAI Model - Content Signal
- **Type / role:** `lmChatOpenAi` — GPT‑4o chat model for Content Signal Agent.
- **Configuration choices:**
  - **Model:** `gpt-4o`
  - **Temperature:** `0.2` (more deterministic)
- **Credentials:** OpenAI API credential (“OpenAi account”)
- **Connected as:** `ai_languageModel` input to Content Signal Agent
- **Failure/edge cases:**
  - Invalid API key, quota exhaustion, rate limiting (429), model not available, network timeouts.

#### Node: Structured Output - Content Signal
- **Type / role:** `Structured Output Parser` — enforces a JSON structure for validation results.
- **Schema example (conceptual):**
  - `validationStatus`: VALID / FLAGGED / SUSPICIOUS
  - `severityLevel`: LOW / MEDIUM / HIGH / CRITICAL
  - `confidenceScore`: 0..1
  - `violationTypes`: array
  - `reasoning`: string
  - `requiresGovernanceReview`: boolean
- **Connected as:** `ai_outputParser` input to Content Signal Agent
- **Failure/edge cases:**
  - Parser errors if agent output cannot be coerced into expected JSON.

#### Node: Route by Validation Status
- **Type / role:** `Switch` — routes based on `output.validationStatus`.
- **Rules:**
  - **Valid** if `{{$json.output.validationStatus}} == "VALID"`
  - **Flagged** if `... == "FLAGGED"`
  - **Fallback renamed:** “Suspicious”
- **Outputs to:**
  - **Valid →** Store Valid Content
  - **Flagged →** Governance Agent
  - **Suspicious (fallback) →** not connected in this workflow (important gap)
- **Failure/edge cases:**
  - If `output.validationStatus` missing/misspelled, item goes to fallback “Suspicious” and then stops (no connections).

**Sticky note covering this block (applied contextually):**
- **“## Ingest & Validate …”**: “Capture content via webhook and normalize signals using AI validation…”

---

### Block 2 — Governance Orchestration (Agent + specialized tools)
**Overview:** For flagged content, determines governance action type and can invoke specialized tools (review, appeals, enforcement) to produce a structured governance decision.

**Nodes involved**
- Governance Agent
- OpenAI Model - Governance
- Structured Output - Governance
- Review Agent Tool
- OpenAI Model - Review Tool
- Structured Output - Review Tool
- Appeals Agent Tool
- OpenAI Model - Appeals Tool
- Structured Output - Appeals Tool
- Enforcement Agent Tool
- OpenAI Model - Enforcement Tool
- Structured Output - Enforcement Tool
- Route by Action Type

#### Node: Governance Agent
- **Type / role:** `LangChain Agent` — orchestrator deciding REVIEW/APPEAL/ENFORCE/ESCALATE.
- **Key configuration:**
  - Text includes original content, userId, full validation output, severity, violation types.
  - System message instructs to call specialized agent tools.
  - Has output parser (Structured Output - Governance).
- **Tools available (connected via `ai_tool`):**
  - Review Agent Tool
  - Appeals Agent Tool
  - Enforcement Agent Tool
- **Outputs to:** Route by Action Type
- **Failure/edge cases:**
  - Tool invocation depends on the agent actually choosing to call tools; if it doesn’t, governance output may be shallow.
  - “ESCALATE” action type is defined in the prompt but not handled explicitly downstream (falls into “Other Actions” output, which is not connected).

#### Node: OpenAI Model - Governance
- **Type / role:** `lmChatOpenAi` — GPT‑4o model for Governance Agent.
- **Configuration choices:**
  - Temperature `0.3` (slightly more flexible)
- **Credentials:** OpenAI API credential
- **Connected as:** `ai_languageModel` to Governance Agent
- **Failure/edge cases:** Same as other OpenAI nodes (auth/quota/rate limits).

#### Node: Structured Output - Governance
- **Type / role:** Structured parser for governance decision.
- **Expected fields (conceptual):**
  - `actionType` (ENFORCE/REVIEW/…)
  - `enforcementAction`
  - `severityLevel`
  - `requiresEscalation` (note: downstream IF checks `output.escalationRequired`, which does **not** match this name)
  - `reasoning`, `nextSteps[]`, `complianceNotes`
- **Connected as:** `ai_outputParser` to Governance Agent
- **Important mismatch / edge case:**
  - The schema uses `requiresEscalation`, but the IF node later checks `output.escalationRequired`. Unless the model outputs both, escalation may never trigger.

#### Node: Review Agent Tool
- **Type / role:** `agentTool` — specialized review tool callable by Governance Agent.
- **Input mapping:** `{{$fromAI("contentData", "...", "json")}}` (expects Governance Agent to provide `contentData` to the tool)
- **System message:** detailed policy assessment and outcomes.
- **Backed by:**
  - OpenAI Model - Review Tool (GPT‑4o)
  - Structured Output - Review Tool
- **Connected as:** `ai_tool` into Governance Agent
- **Failure/edge cases:**
  - If Governance Agent doesn’t supply `contentData` in the expected format, tool execution may fail or be empty.

#### Node: OpenAI Model - Review Tool / Structured Output - Review Tool
- **Role:** LLM + parser to produce structured review results:
  - `reviewOutcome` (APPROVE/RESTRICT/REMOVE/ESCALATE)
  - `policyViolations[]`, `contextualFactors`, `reasoning`, `recommendedAction`
- **Failure/edge cases:** LLM availability; parser coercion failures.

#### Node: Appeals Agent Tool (+ model + parser)
- **Role:** Processes appeals if invoked.
- **Tool input:** `{{$fromAI("appealData", "...", "json")}}`
- **Structured output:** `appealOutcome`, `originalDecisionValid`, `newEvidenceConsidered`, `reasoning`, `userNotification`
- **Edge cases:** Tool likely never used unless Governance Agent is prompted with appeal context; webhook payload currently has no explicit “appeal” field.

#### Node: Enforcement Agent Tool (+ model + parser)
- **Role:** Determines enforcement actions.
- **Tool input:** `{{$fromAI("enforcementData", "...", "json")}}`
- **Structured output:** includes `enforcementAction`, `escalationRequired` (note naming differs again from governance schema), `complianceNotes`, etc.
- **Edge cases:** Output fields may not be carried into the final item unless Governance Agent includes them in its own structured output.

#### Node: Route by Action Type
- **Type / role:** `Switch` — routes based on `{{$json.output.actionType}}`.
- **Rules:**
  - **Review** if actionType == REVIEW
  - **Enforce** if actionType == ENFORCE
  - **Fallback renamed:** Other Actions (not connected)
- **Outputs to:**
  - Review → Store Flagged Content
  - Enforce → Store Enforcement Actions
- **Edge cases:**
  - actionType == APPEAL or ESCALATE will go to “Other Actions” and then stop (no connection).

**Sticky note covering this block (applied contextually):**
- **“## Governance Orchestration …”**: “Apply rule-based routing and agent-driven decision logic…”

---

### Block 3 — Enforcement & Escalation
**Overview:** Stores enforcement actions, checks whether escalation is required, and if so notifies Slack and emails an escalation address.

**Nodes involved**
- Store Enforcement Actions
- Check Escalation Required
- Notify Moderation Team
- Escalation Email

#### Node: Store Enforcement Actions
- **Type / role:** `Data Table` — persists enforcement outcomes.
- **Target table:** `EnforcementActions`
- **Input from:** Route by Action Type (Enforce output)
- **Output to:** Check Escalation Required
- **Edge cases:**
  - Data Table not created or access denied.
  - Stored record will be whatever fields exist on the item; if you need a stable schema, add a mapping Set/Code node.

#### Node: Check Escalation Required
- **Type / role:** `If` — decides whether to escalate.
- **Condition:** checks `{{$json.output.escalationRequired}} == true`
- **Inputs from:** Store Enforcement Actions
- **Outputs to:** Notify Moderation Team and Escalation Email (both connected on the “true” output)
- **Critical mismatch risk:**
  - Governance schema uses `requiresEscalation`, while IF checks `output.escalationRequired`. Unless governance output includes `escalationRequired`, escalation won’t happen.
  - Also, after storing in Data Table, the structure may change depending on Data Table node behavior/version (verify that `output.*` remains present).

#### Node: Notify Moderation Team
- **Type / role:** `Slack` — sends critical escalation message to a channel.
- **Authentication:** OAuth2 Slack
- **Channel:** uses `{{ $('Workflow Configuration').first().json.slackChannelId }}`
- **Message content:** includes action type, enforcement, severity, userId, contentType, reasoning, compliance notes, and `nextSteps`.
- **Output to:** Merge All Paths
- **Edge cases:**
  - Slack credential scopes missing (e.g., chat:write).
  - `nextSteps` missing or not an array → `.join("\n")` will error.

#### Node: Escalation Email
- **Type / role:** `Email Send` — emails escalation details (HTML).
- **To:** `{{ $('Workflow Configuration').first().json.escalationEmail }}`
- **From:** placeholder sender email
- **Subject:** includes severity
- **Output to:** Merge All Paths
- **Edge cases:**
  - SMTP/credential misconfiguration.
  - `nextSteps` missing/not array → `.map(...).join("")` can error.
  - `$now.toFormat(...)` requires Luxon DateTime in n8n; in most modern n8n, `$now` is available, but formatting support may vary by version.

**Sticky note covering this block (applied contextually):**
- **“## Enforcement & Escalation …”**: “Approve, flag, store, or escalate with notifications…”

---

### Block 4 — Storage for Valid/Flagged + Merge
**Overview:** Writes valid and flagged content to separate tables and merges all possible paths for unified audit logging.

**Nodes involved**
- Store Valid Content
- Store Flagged Content
- Merge All Paths

#### Node: Store Valid Content
- **Type / role:** `Data Table` — stores content assessed as VALID.
- **Target table:** `ValidContent`
- **Input from:** Route by Validation Status (Valid)
- **Output to:** Merge All Paths
- **Edge cases:** Data Table existence/permissions; schema consistency.

#### Node: Store Flagged Content
- **Type / role:** `Data Table` — stores items routed to REVIEW path.
- **Target table:** `FlaggedContent`
- **Input from:** Route by Action Type (Review)
- **Output to:** Merge All Paths
- **Edge cases:** actionType REVIEW is the only way content is stored as flagged here; FLAGGED validation that becomes ENFORCE bypasses this table.

#### Node: Merge All Paths
- **Type / role:** `Merge` — combines up to four streams for audit processing.
- **Mode:** “combine”
- **Combine by:** position
- **Number of inputs:** 4
- **Inputs connected:**
  1. From Store Valid Content
  2. From Store Flagged Content
  3. From Notify Moderation Team
  4. From Escalation Email
- **Output to:** Prepare Audit Data
- **Edge cases:**
  - If some inputs never emit items, combine-by-position can lead to missing/uneven pairing. For audit logging, an “append”/“pass-through” approach is often safer than positional combine unless you guarantee item alignment.

---

### Block 5 — Audit & Logging
**Overview:** Normalizes merged outputs into a consistent audit entry format and writes them to a `ModerationAuditLog` table.

**Nodes involved**
- Prepare Audit Data
- Audit Log

#### Node: Prepare Audit Data
- **Type / role:** `Code` — transforms all incoming items into normalized audit entries.
- **Key logic (interpreted):**
  - Reads all items from previous merge.
  - Builds an `auditEntry` with:
    - `timestamp`, `workflow_execution_id`
    - `workflow_path` (from `item.json.workflow_path` or `unknown`)
    - Many moderation/governance/enforcement fields using fallbacks:
      - `validation_status` from `validation_status` or `status`
      - content fields (`content_text`, `content`, etc.)
      - governance and enforcement fields with multiple fallback names
      - escalation fields, reviewer metadata
  - Returns an array of `{ json: auditEntry }` items.
- **Input from:** Merge All Paths
- **Output to:** Audit Log
- **Edge cases:**
  - Most upstream nodes do not set `workflow_path`, `content_id`, `platform`, etc., so many audit fields will be null/undefined unless you add mapping earlier.
  - If incoming items contain nested `output` objects (as the AI nodes do), the code currently expects flattened fields like `item.json.validation_status`, not `item.json.output.validationStatus`. Result: audit entries may miss key AI outcomes unless you adapt the code.

#### Node: Audit Log
- **Type / role:** `Data Table` — persists audit entries.
- **Target table:** `ModerationAuditLog`
- **Edge cases:** Data Table existence/permissions; volume growth.

**Sticky note covering this block (applied contextually):**
- **“## Audit & Logging …”**: “Merge results and generate structured audit logs…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Content Moderation Webhook | Webhook | Receives moderation requests via HTTP POST | — | Workflow Configuration | ## Ingest & Validate\nWhat – Capture content via webhook and normalize signals using AI validation.\nWhy – Ensure structured, policy-aligned inputs before enforcement decisions. |
| Workflow Configuration | Set | Defines thresholds and destinations (email/channel) | Content Moderation Webhook | Content Signal Agent | ## Ingest & Validate\nWhat – Capture content via webhook and normalize signals using AI validation.\nWhy – Ensure structured, policy-aligned inputs before enforcement decisions. |
| Content Signal Agent | LangChain Agent | Validates content + flags; outputs structured status/severity/confidence | Workflow Configuration | Route by Validation Status | ## Ingest & Validate\nWhat – Capture content via webhook and normalize signals using AI validation.\nWhy – Ensure structured, policy-aligned inputs before enforcement decisions. |
| OpenAI Model - Content Signal | OpenAI Chat Model | LLM for Content Signal Agent (GPT‑4o) | — | Content Signal Agent | ## Ingest & Validate\nWhat – Capture content via webhook and normalize signals using AI validation.\nWhy – Ensure structured, policy-aligned inputs before enforcement decisions. |
| Structured Output - Content Signal | Structured Output Parser | Enforces JSON structure for signal validation | — | Content Signal Agent | ## Ingest & Validate\nWhat – Capture content via webhook and normalize signals using AI validation.\nWhy – Ensure structured, policy-aligned inputs before enforcement decisions. |
| Route by Validation Status | Switch | Routes VALID vs FLAGGED vs fallback (Suspicious) | Content Signal Agent | Store Valid Content; Governance Agent | ## Ingest & Validate\nWhat – Capture content via webhook and normalize signals using AI validation.\nWhy – Ensure structured, policy-aligned inputs before enforcement decisions. |
| Governance Agent | LangChain Agent | Chooses governance action; may call tools | Route by Validation Status (Flagged) | Route by Action Type | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| OpenAI Model - Governance | OpenAI Chat Model | LLM for Governance Agent (GPT‑4o) | — | Governance Agent | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Structured Output - Governance | Structured Output Parser | Enforces JSON structure for governance decision | — | Governance Agent | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Review Agent Tool | Agent Tool | Tool callable by Governance Agent for deep review | — | Governance Agent (as tool) | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| OpenAI Model - Review Tool | OpenAI Chat Model | LLM for Review Agent Tool | — | Review Agent Tool | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Structured Output - Review Tool | Structured Output Parser | Enforces JSON output for review tool | — | Review Agent Tool | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Appeals Agent Tool | Agent Tool | Tool callable for appeals processing | — | Governance Agent (as tool) | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| OpenAI Model - Appeals Tool | OpenAI Chat Model | LLM for Appeals Agent Tool | — | Appeals Agent Tool | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Structured Output - Appeals Tool | Structured Output Parser | Enforces JSON output for appeals tool | — | Appeals Agent Tool | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Enforcement Agent Tool | Agent Tool | Tool callable for enforcement decisioning | — | Governance Agent (as tool) | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| OpenAI Model - Enforcement Tool | OpenAI Chat Model | LLM for Enforcement Agent Tool | — | Enforcement Agent Tool | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Structured Output - Enforcement Tool | Structured Output Parser | Enforces JSON output for enforcement tool | — | Enforcement Agent Tool | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Route by Action Type | Switch | Routes REVIEW vs ENFORCE vs fallback | Governance Agent | Store Flagged Content; Store Enforcement Actions | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Store Valid Content | Data Table | Persists VALID content | Route by Validation Status (Valid) | Merge All Paths | ## Enforcement & Escalation\nWhat – Approve, flag, store, or escalate with notifications.\nWhy – Ensure consistent action handling and compliance tracking. |
| Store Flagged Content | Data Table | Persists REVIEW-routed flagged content | Route by Action Type (Review) | Merge All Paths | ## Enforcement & Escalation\nWhat – Approve, flag, store, or escalate with notifications.\nWhy – Ensure consistent action handling and compliance tracking. |
| Store Enforcement Actions | Data Table | Persists ENFORCE actions | Route by Action Type (Enforce) | Check Escalation Required | ## Enforcement & Escalation\nWhat – Approve, flag, store, or escalate with notifications.\nWhy – Ensure consistent action handling and compliance tracking. |
| Check Escalation Required | IF | Escalation gate based on enforcement output flag | Store Enforcement Actions | Notify Moderation Team; Escalation Email | ## Enforcement & Escalation\nWhat – Approve, flag, store, or escalate with notifications.\nWhy – Ensure consistent action handling and compliance tracking. |
| Notify Moderation Team | Slack | Sends escalation alert to Slack channel | Check Escalation Required (true) | Merge All Paths | ## Enforcement & Escalation\nWhat – Approve, flag, store, or escalate with notifications.\nWhy – Ensure consistent action handling and compliance tracking. |
| Escalation Email | Email Send | Sends escalation email with HTML details | Check Escalation Required (true) | Merge All Paths | ## Enforcement & Escalation\nWhat – Approve, flag, store, or escalate with notifications.\nWhy – Ensure consistent action handling and compliance tracking. |
| Merge All Paths | Merge | Combines outputs for centralized audit transformation | Store Valid Content; Store Flagged Content; Notify Moderation Team; Escalation Email | Prepare Audit Data | ## Audit & Logging\nWhat – Merge results and generate structured audit logs.\nWhy – Maintain transparency, traceability, and review readiness. |
| Prepare Audit Data | Code | Normalizes items into audit entries | Merge All Paths | Audit Log | ## Audit & Logging\nWhat – Merge results and generate structured audit logs.\nWhy – Maintain transparency, traceability, and review readiness. |
| Audit Log | Data Table | Writes audit entries | Prepare Audit Data | — | ## Audit & Logging\nWhat – Merge results and generate structured audit logs.\nWhy – Maintain transparency, traceability, and review readiness. |
| Sticky Note | Sticky Note | Comment block | — | — | ## Prerequisites\nn8n account, OpenAI API key, Google Sheets or DB access, Gmail credentials, defined moderation policies.\n## Use Cases\nMarketplace listing moderation, enterprise HR review screening\n## Customization\nAdjust policy rules, add risk scoring, integrate Slack instead of Gmail\n## Benefits\nImproves moderation accuracy, reduces manual review, enforces governance consistency |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Setup Steps\n1. Connect OpenAI API credentials for AI agents.\n2. Configure Google Sheets or database for logging.\n3. Connect Gmail for escalation emails.\n4. Define moderation policies and routing rules.\n5. Activate webhook and test sample content. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## How It Works\nThis workflow automates end-to-end AI-driven content moderation for platforms managing user-generated content, including marketplaces, communities, and enterprise systems. It is designed for product, trust & safety, and governance teams seeking scalable, policy-aligned enforcement without subjective scoring. The workflow validates structured review, goal, and feedback data using a Performance Signal Agent that standardizes moderation signals and removes ambiguity. A Governance Agent then orchestrates policy enforcement, eligibility checks, escalation logic, and audit preparation. Content enters via webhook, is classified, validated, and routed by action type (approve, flag, escalate). Enforcement logic determines whether to store clean content, flag violations, or trigger escalation emails and team notifications. All actions are logged for traceability and compliance. This template solves inconsistent moderation decisions, lack of structured governance controls, and manual escalation overhead by embedding deterministic checkpoints, structured outputs, and audit-ready logging into a single automated pipeline. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Enforcement & Escalation\nWhat – Approve, flag, store, or escalate with notifications.\nWhy – Ensure consistent action handling and compliance tracking. |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## Governance Orchestration\nWhat – Apply rule-based routing and agent-driven decision logic.\nWhy – Remove subjective scoring and standardize eligibility checks. |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## Ingest & Validate\nWhat – Capture content via webhook and normalize signals using AI validation.\nWhy – Ensure structured, policy-aligned inputs before enforcement decisions. |
| Sticky Note6 | Sticky Note | Comment block | — | — | ## Audit & Logging\nWhat – Merge results and generate structured audit logs.\nWhy – Maintain transparency, traceability, and review readiness. |

---

## 4. Reproducing the Workflow from Scratch (Manual Build)

1. **Create a new workflow** in n8n named: *AI-driven content moderation and governance workflow*.

2. **Add “Content Moderation Webhook” (Webhook node)**
   - Method: **POST**
   - Path: **content-moderation**
   - Response mode: **Using “Respond to Webhook” node** (or change to “On Received” if you want immediate default responses).
   - Ensure you will send JSON shaped like:
     - `content`, `userId`, `contentType`, `moderationFlags` under the request body.

3. **Add “Workflow Configuration” (Set node)**
   - Add fields:
     - `moderationThreshold` (number) = `0.75`
     - `escalationEmail` (string) = your escalation mailbox
     - `slackChannelId` (string) = your Slack channel ID
     - `severityLevels` (array) = `["LOW","MEDIUM","HIGH","CRITICAL"]`
   - Enable **Include Other Fields** to preserve the webhook payload.
   - Connect: **Webhook → Set**

4. **Add “Content Signal Agent” (LangChain Agent node)**
   - Prompt type: **Define**
   - User text (expression-based) should include:
     - `{{$json.body.content}}`, `{{$json.body.userId}}`, `{{$json.body.contentType}}`,
     - `{{ JSON.stringify($json.body.moderationFlags) }}`
   - System message: define validation responsibilities and statuses (VALID/FLAGGED/SUSPICIOUS).
   - Enable **Structured Output** (has output parser).
   - Connect: **Workflow Configuration → Content Signal Agent**

5. **Add “OpenAI Model - Content Signal” (OpenAI Chat Model node)**
   - Model: **gpt-4o**
   - Temperature: **0.2**
   - Configure **OpenAI credentials** (API key).
   - Connect as **AI Language Model** to Content Signal Agent.

6. **Add “Structured Output - Content Signal” (Structured Output Parser node)**
   - Provide a JSON schema example containing:
     - validationStatus, severityLevel, confidenceScore, violationTypes, reasoning, requiresGovernanceReview
   - Connect as **AI Output Parser** to Content Signal Agent.

7. **Add “Route by Validation Status” (Switch node)**
   - Switch value: `{{$json.output.validationStatus}}`
   - Rule 1 output name: **Valid**, equals `VALID`
   - Rule 2 output name: **Flagged**, equals `FLAGGED`
   - Fallback output name: **Suspicious**
   - Connect: **Content Signal Agent → Switch**

8. **Add “Store Valid Content” (Data Table node)**
   - Data table: create/select **ValidContent**
   - Connect: **Switch (Valid) → Store Valid Content**

9. **Add “Governance Agent” (LangChain Agent node)**
   - User text: include content + userId + `{{ JSON.stringify($json.output) }}` etc.
   - System message: instruct to choose actionType and call tools (REVIEW/APPEAL/ENFORCE/ESCALATE).
   - Enable structured output.
   - Connect: **Switch (Flagged) → Governance Agent**
   - (Optionally also connect Suspicious → Governance Agent if you want human/AI handling for fallback.)

10. **Add “OpenAI Model - Governance” (OpenAI Chat Model)**
    - Model: **gpt-4o**
    - Temperature: **0.3**
    - Connect as **AI Language Model** to Governance Agent.

11. **Add “Structured Output - Governance” (Structured Output Parser)**
    - Include example fields:
      - actionType, enforcementAction, severityLevel, requiresEscalation (or escalationRequired), reasoning, nextSteps, complianceNotes
    - Connect as **AI Output Parser** to Governance Agent.
    - Recommendation: **standardize the escalation flag name** here to match your IF node (see step 18).

12. **Add “Review Agent Tool” (Agent Tool node)**
    - Tool description: detailed content review
    - Tool input expression: `{{$fromAI("contentData", "...", "json")}}`
    - Configure its own OpenAI model + structured output:
      - Add “OpenAI Model - Review Tool” (gpt-4o, temp 0.2) connected as tool language model
      - Add “Structured Output - Review Tool” with reviewOutcome/policyViolations/etc.
    - Connect **Review Agent Tool** to **Governance Agent** via **AI Tool** connection.

13. **Add “Appeals Agent Tool” (Agent Tool node)**
    - Input: `{{$fromAI("appealData", "...", "json")}}`
    - Add “OpenAI Model - Appeals Tool” (gpt-4o, temp 0.2)
    - Add “Structured Output - Appeals Tool”
    - Connect as **AI Tool** to Governance Agent.

14. **Add “Enforcement Agent Tool” (Agent Tool node)**
    - Input: `{{$fromAI("enforcementData", "...", "json")}}`
    - Add “OpenAI Model - Enforcement Tool” (gpt-4o, temp 0.2)
    - Add “Structured Output - Enforcement Tool”
    - Connect as **AI Tool** to Governance Agent.

15. **Add “Route by Action Type” (Switch node)**
    - Switch value: `{{$json.output.actionType}}`
    - Rule outputs:
      - **Review** when equals `REVIEW`
      - **Enforce** when equals `ENFORCE`
    - Fallback: **Other Actions**
    - Connect: **Governance Agent → Route by Action Type**

16. **Add “Store Flagged Content” (Data Table node)**
    - Data table: **FlaggedContent**
    - Connect: **Route by Action Type (Review) → Store Flagged Content**

17. **Add “Store Enforcement Actions” (Data Table node)**
    - Data table: **EnforcementActions**
    - Connect: **Route by Action Type (Enforce) → Store Enforcement Actions**

18. **Add “Check Escalation Required” (IF node)**
    - Condition: boolean equals true on `{{$json.output.escalationRequired}}`
    - Connect: **Store Enforcement Actions → IF**
    - Important: ensure your governance/enforcement structured outputs actually provide `output.escalationRequired` (or change the IF to match `requiresEscalation`).

19. **Add “Notify Moderation Team” (Slack node)**
    - Authentication: **OAuth2**
    - Operation: send message to channel
    - ChannelId expression: `{{ $('Workflow Configuration').first().json.slackChannelId }}`
    - Message text: include action, severity, user and reasoning; ensure `nextSteps` is an array before using `.join()`.
    - Configure **Slack OAuth2 credentials** with required scopes.
    - Connect: **IF (true) → Slack**

20. **Add “Escalation Email” (Email Send node)**
    - To: `{{ $('Workflow Configuration').first().json.escalationEmail }}`
    - From: your sender address
    - Subject: include severity
    - HTML body: include enforcement details and next steps.
    - Configure **email credentials** (SMTP or provider integration).
    - Connect: **IF (true) → Email**

21. **Add “Merge All Paths” (Merge node)**
    - Mode: **Combine**
    - Combine by: **Position**
    - Number of inputs: **4**
    - Connect inputs:
      1. **Store Valid Content → Merge**
      2. **Store Flagged Content → Merge**
      3. **Notify Moderation Team → Merge**
      4. **Escalation Email → Merge**

22. **Add “Prepare Audit Data” (Code node)**
    - Paste/implement logic to normalize merged items into a consistent audit schema.
    - Recommendation: map from actual AI fields (e.g., `item.json.output.validationStatus`) if you want accurate audit capture.
    - Connect: **Merge All Paths → Prepare Audit Data**

23. **Add “Audit Log” (Data Table node)**
    - Data table: **ModerationAuditLog**
    - Connect: **Prepare Audit Data → Audit Log**

24. **(Optional but recommended) Add a “Respond to Webhook” node**
    - Return a JSON summary (validationStatus, actionType, etc.) to the caller.
    - Connect it appropriately depending on which path you want to answer from.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: n8n account, OpenAI API key, Google Sheets or DB access, Gmail credentials, defined moderation policies. | Sticky note “Prerequisites” |
| Use Cases: Marketplace listing moderation, enterprise HR review screening | Sticky note “Prerequisites” |
| Customization: Adjust policy rules, add risk scoring, integrate Slack instead of Gmail | Sticky note “Prerequisites” |
| Benefits: Improves moderation accuracy, reduces manual review, enforces governance consistency | Sticky note “Prerequisites” |
| Setup Steps: connect OpenAI credentials; configure Sheets/DB logging; connect Gmail; define policies/routing rules; activate webhook and test sample content. | Sticky note “Setup Steps” |
| How it works: end-to-end moderation with validation, governance orchestration, enforcement, escalation, and audit logging for traceability. | Sticky note “How It Works” |

**Notable integration/logic issues to address when implementing**
- **No handling for “Suspicious” validation fallback** (Switch fallback not connected).
- **No handling for actionType “APPEAL” or “ESCALATE”** (falls to “Other Actions” with no connection).
- **Escalation flag naming mismatch** (`requiresEscalation` vs `escalationRequired`).
- **Audit Code node expects flattened fields**, but AI nodes output nested `output.*` structures—consider mapping before audit logging.

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.