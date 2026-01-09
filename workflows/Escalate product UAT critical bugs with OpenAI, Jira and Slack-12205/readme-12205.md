Escalate product UAT critical bugs with OpenAI, Jira and Slack

https://n8nworkflows.xyz/workflows/escalate-product-uat-critical-bugs-with-openai--jira-and-slack-12205


# Escalate product UAT critical bugs with OpenAI, Jira and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Escalate product UAT critical bugs with OpenAI, Jira and Slack  
**Workflow name (in JSON):** Product UAT Critical Bug Escalation (AI → Jira + Slack)  
**Purpose:** Receive UAT feedback, normalize it, use an OpenAI model to triage/classify it into a strict JSON schema, validate the AI output, and—when the feedback is treated as a critical bug—automatically create a Jira issue, alert engineering in Slack, and close the loop by notifying the tester (Slack DM or email) and responding to the original webhook with a structured status payload.

### 1.1 Ingestion & Configuration Bootstrap
- Entry via **Webhook**.
- Static configuration values (project keys, channels, thresholds) are injected via a **Set** node and merged into the webhook payload.

### 1.2 Input Normalization & Cleaning
- Converts incoming payload variants into a consistent `uat.*` structure.
- Cleans the message body to remove HTML and cap length for LLM safety/performance.

### 1.3 AI Triage (LLM) & Quality Control
- Calls OpenAI with a strict JSON-only prompt.
- Parses the LLM response, validates enums and clamps confidence to `[0..1]`.
- Adds a fallback triage object if parsing fails.

### 1.4 Critical Bug Escalation (Jira + Engineering Slack)
- Creates a Jira “Bug” issue with “High” priority and a structured description including repro steps and reporter info.
- Posts an engineering alert in a Slack channel with context and the Jira issue key.

### 1.5 Closed Loop (Tester Notification + Webhook Response)
- Composes a tester-facing acknowledgement message.
- Chooses Slack DM vs email based on `uat.source`.
- Responds to the webhook request with a status payload including type/severity/confidence.

---

## 2. Block-by-Block Analysis

### Block 1 — Ingestion & Configuration Bootstrap

**Overview:** Receives UAT feedback via HTTP POST and merges it with workflow configuration constants to support consistent downstream behavior.

**Nodes involved:** `trigger`, `data mapping`, `data merge`

#### Node: `trigger`
- **Type / role:** Webhook (n8n-nodes-base.webhook) — workflow entry point.
- **Config (interpreted):**
  - Method: **POST**
  - Path: `3a6c966e-0409-4a8d-9ad9-bbf834689964` (also used as webhookId)
- **Key data:** Incoming request body becomes `$json`.
- **Connections:**
  - Output → `data merge` (index 0)
- **Version notes:** typeVersion **2.1**.
- **Edge cases / failures:**
  - Unexpected payload shapes (missing fields) are handled later by normalization.
  - If caller expects immediate response but workflow errors before `Respond to Webhook`, request may hang/return error depending on n8n webhook settings.

#### Node: `data mapping`
- **Type / role:** Set (n8n-nodes-base.set) — injects configuration values under `cfg.*`.
- **Config choices:**
  - Creates:
    - `cfg.jiraProjectKey = "UAT"` (note: not actually referenced by the Jira node, which uses numeric IDs)
    - `cfg.jiraIssueTypeBug = "Bug"` (also not used directly)
    - `cfg.slackChannelEng = "#eng-uat"` (not used directly; Slack node uses a channel ID)
    - `cfg.slackChannelPm = "#product-uat"`
    - `cfg.sheetIdDigest = "YourID"`
    - `cfg.manualReviewEmail = "user@example.com"`
    - `cfg.confidenceThreshold = 0.6`
    - `cfg.dedupeEnabled = false`
    - `cfg.llmProvider = "openai"`
- **Connections:**
  - Output → `data merge` (index 1)
- **Version notes:** typeVersion **3.4**.
- **Edge cases / failures:**
  - If later logic were to rely on `cfg.*` (it currently doesn’t), missing/incorrect values would matter. As implemented, these fields mainly serve as placeholders/documentation.

#### Node: `data merge`
- **Type / role:** Merge (n8n-nodes-base.merge) — combines webhook payload and `cfg.*` object.
- **Config choices:**
  - Mode: **Combine**
  - Combine by: **Position**
- **Connections:**
  - Input 0 ← `trigger`
  - Input 1 ← `data mapping`
  - Output → `normalize`
- **Version notes:** typeVersion **3.2**.
- **Edge cases / failures:**
  - Combine-by-position assumes both inputs emit items in matching order/count. Here, each side likely emits a single item; if either side emits multiple items, pairing may misalign.

---

### Block 2 — Input Normalization & Cleaning

**Overview:** Normalizes varied inbound shapes into a stable `uat` object, then cleans the message text for LLM consumption.

**Nodes involved:** `normalize`, `clean text`

#### Node: `normalize`
- **Type / role:** Code (n8n-nodes-base.code) — schema normalization.
- **Config choices (behavior):**
  - Reads possible fields from multiple locations (`input.source`, `input.uat.source`, etc.).
  - Produces a single object:
    - `uat.source` (default `"webhook"`)
    - `uat.tester_name` (default `""`)
    - `uat.tester_email` (default `""`)
    - `uat.message_raw` (default `""`)
    - `uat.build_version` (default `"unknown"`)
    - `uat.page_url`, `uat.screenshot_url` (default `""`)
    - `uat.received_at` (ISO timestamp)
- **Key expressions/variables:** Uses `$json` as input.
- **Connections:**
  - Input ← `data merge`
  - Output → `clean text`
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - If inbound payload is not an object (rare), `$json || {}` mitigates.
  - Empty `tester_email` later causes Gmail send failures (invalid recipient).

#### Node: `clean text`
- **Type / role:** Code — sanitizes and truncates message content.
- **Config choices (behavior):**
  - Removes HTML tags: `/<[^>]*>/g`
  - Collapses whitespace
  - Trims
  - Truncates to **3000 characters**
  - Writes to `uat.message_clean`
- **Connections:**
  - Input ← `normalize`
  - Output → `AI agent`
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - If `uat.message_raw` is empty, `message_clean` becomes `""`, which may reduce LLM quality and could increase likelihood of `type=Noise`.

---

### Block 3 — AI Triage (LLM) & Quality Control

**Overview:** Sends cleaned UAT feedback to an OpenAI model with a strict JSON-only schema, then parses and validates the result to ensure downstream nodes receive safe, expected values.

**Nodes involved:** `AI agent`, `parsing and validation`

#### Node: `AI agent`
- **Type / role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) — LLM inference.
- **Config choices:**
  - Model: **gpt-5.2**
  - Prompt instructs: “return ONLY JSON” with a strict schema:
    - `sentiment`: Positive|Negative
    - `type`: CriticalBug|UXImprovement|FeatureRequest|Noise
    - `severity`: Blocker|Critical|Major|Minor
    - `summary` (≤160 chars), `suggested_title` (≤80 chars)
    - `components` limited to a predefined set
    - `repro_steps` array
    - `confidence` float
  - Rules push broken/crash/data loss/payment/login into CriticalBug with severity ≥ Critical.
- **Credentials:** OpenAI API credential required.
- **Connections:**
  - Input ← `clean text`
  - Output → `parsing and validation`
- **Version notes:** typeVersion **2** (LangChain OpenAI node).
- **Edge cases / failures:**
  - Model may return non-JSON, extra commentary, or invalid schema ⇒ handled by parsing node.
  - API failures: auth, quota, rate limit, timeout.
  - Prompt assumes `uat.message_clean` exists; if missing, prompt still compiles but may be empty.

#### Node: `parsing and validation`
- **Type / role:** Code — JSON parse + normalization/validation of AI output.
- **Config choices (behavior):**
  - Attempts to find model text in multiple possible fields:
    - `$json.output`, `$json.text`, `$json.message`, `$json.response.text`,
    - `$json.data[0].content`,
    - `$json.response[0].message.content`
  - `JSON.parse(raw)`:
    - On failure, sets `triage.parse_ok=false` and inserts a safe default triage:
      - type Noise, severity Minor, confidence 0, etc.
  - Validates enums:
    - `type` ∈ [CriticalBug, UXImprovement, FeatureRequest, Noise] else Noise
    - `severity` ∈ [Blocker, Critical, Major, Minor] else Minor
    - `sentiment` ∈ [Positive, Negative] else Negative
  - Clamps `confidence` to [0..1]
  - Outputs merged object: `...$json` plus `triage`
- **Connections:**
  - Input ← `AI agent`
  - Output → `critical bug`
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - If the OpenAI node output structure changes, the `raw` selector may miss the content and parse `""`, triggering fallback.
  - Even with valid JSON, fields may be missing; downstream expressions should tolerate undefined (some do, some don’t).

---

### Block 4 — Critical Bug Escalation (Jira + Engineering Slack)

**Overview:** Creates a Jira bug and posts an engineering Slack alert with severity, summary, build, components, and the resulting Jira key.

**Nodes involved:** `critical bug`, `engeneering alert`

#### Node: `critical bug`
- **Type / role:** Jira Software Cloud (n8n-nodes-base.jira) — Create Issue.
- **Config choices:**
  - Project: selected by internal ID **10002** (display name cached as “test2”)
  - Issue type: ID **10013** (cached “Bug”)
  - Summary: `triage.suggested_title || triage.summary`
  - Additional fields:
    - Priority: ID **2** (cached “High”)
    - Description includes:
      - Build, page URL, screenshot URL
      - Repro steps joined by newline
      - Reporter name/email and source
- **Credentials:** Jira Software Cloud API credential required.
- **Connections:**
  - Input ← `parsing and validation`
  - Output → `engeneering alert`
- **Version notes:** typeVersion **1**.
- **Edge cases / failures:**
  - Jira auth/permission errors (project or issue create permission).
  - The workflow name implies “critical bugs only”, but **there is no IF gate** here:
    - As implemented, it will create a Jira issue for *every* item, including `type=Noise`.
  - If `triage.repro_steps` is not an array, `.join()` may throw; code sets default `[]` only in parse-fail case, not necessarily for partial JSON responses.

#### Node: `engeneering alert`
- **Type / role:** Slack (n8n-nodes-base.slack) — post message to a channel.
- **Config choices:**
  - Authentication: OAuth2
  - Target: channel ID `C09V1228324` (cached `tous-n8n`)
  - Text includes:
    - severity, summary, build, components list
    - Jira identifier: `{{ $json.key || $json.id }}`
- **Credentials:** Slack OAuth2 credential required.
- **Connections:**
  - Input ← `critical bug`
  - Output → `compose reply branch 1`
- **Version notes:** typeVersion **2.3**.
- **Edge cases / failures:**
  - Slack auth/token scopes missing (e.g., `chat:write`).
  - If Jira response doesn’t include `key`/`id` in expected place, link reference will be incomplete.

---

### Block 5 — Closed Loop (Tester Notification + Webhook Response)

**Overview:** Builds a tester-friendly acknowledgement, routes it to Slack DM or Gmail depending on `uat.source`, then returns a structured response to the original webhook caller.

**Nodes involved:** `compose reply branch 1`, `how to contact`, `slack tester`, `tester email`, `Webhook response`

#### Node: `compose reply branch 1`
- **Type / role:** Set — prepares `reply.subject` and `reply.body`.
- **Config choices:**
  - `reply.subject`: `UAT feedback received — Ticket {{ $json.key || $json.id }}`
  - `reply.body`: thanks + labels as “Critical Bug (severity)” + ticket + summary + build
- **Connections:**
  - Input ← `engeneering alert`
  - Output → `how to contact`
- **Version notes:** typeVersion **3.4**.
- **Edge cases / failures:**
  - Message always says “logged it as a Critical Bug” even if triage type isn’t CriticalBug (because there is no branching on triage.type).

#### Node: `how to contact`
- **Type / role:** IF (n8n-nodes-base.if) — chooses Slack vs Email based on source.
- **Config choices:**
  - Condition: `{{ $json.uat.source }}` equals `"slack"`
  - True branch (index 0) → Slack DM
  - False branch (index 1) → Gmail
- **Connections:**
  - Input ← `compose reply branch 1`
  - Outputs:
    - True → `slack tester`
    - False → `tester email`
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:**
  - If `uat.source` is not exactly `"slack"` (case/variant), it will fall back to email.

#### Node: `slack tester`
- **Type / role:** Slack — send a DM to a user.
- **Config choices:**
  - Select: user
  - User ID: `U09UKKK9R25` (cached `analyticsn8n`)
  - Text: `{{ $json.reply.body }}`
- **Credentials:** Slack OAuth2 credential required.
- **Connections:**
  - Input ← `how to contact` (true branch)
  - Output → `Webhook response`
- **Version notes:** typeVersion **2.3**.
- **Edge cases / failures:**
  - This does **not** dynamically DM the submitting tester; it always messages the configured Slack user.
  - If the Slack app cannot DM that user (org policies), sending fails.

#### Node: `tester email`
- **Type / role:** Gmail (n8n-nodes-base.gmail) — send email.
- **Config choices:**
  - To: `{{ $json.uat.tester_email }}`
  - Subject: `{{ $json.reply.subject }}`
  - Message: `{{ $json.reply.body }}`
- **Credentials:** Gmail OAuth2 credential required.
- **Connections:**
  - Input ← `how to contact` (false branch)
  - Output → `Webhook response`
- **Version notes:** typeVersion **2.1**.
- **Edge cases / failures:**
  - Empty/invalid `uat.tester_email` will fail.
  - Gmail OAuth token expiry / missing scopes.

#### Node: `Webhook response`
- **Type / role:** Respond to Webhook — returns an HTTP response to the original POST.
- **Config choices:**
  - Respond with: **all incoming items**
  - HTTP code: **200**
  - Response body is configured via `responseKey` expression building JSON-like content:
    - `status: "received"`
    - `type`, `severity`, `confidence` pulled from `triage.*`
- **Connections:**
  - Input ← `slack tester` OR `tester email`
  - Output: none (terminus)
- **Version notes:** typeVersion **1.4**.
- **Edge cases / failures:**
  - The `responseKey` contains embedded `{{ }}` inside a string-like structure; depending on n8n evaluation, this can be brittle. If mis-evaluated, response may contain literal braces rather than values.
  - If execution errors before reaching this node, the webhook caller may not receive a 200.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| trigger | Webhook | Receive UAT feedback via HTTP POST | — | data merge | ## Ingestion & Normalization\n\nCollects UAT feedback from a webhook and normalizes all inputs\n(tester, source, build, page, message) into a clean, AI-ready data structure. |
| data mapping | Set | Inject `cfg.*` constants | — | data merge | ## Ingestion & Normalization\n\nCollects UAT feedback from a webhook and normalizes all inputs\n(tester, source, build, page, message) into a clean, AI-ready data structure. |
| data merge | Merge | Combine webhook payload + config | trigger; data mapping | normalize | ## Ingestion & Normalization\n\nCollects UAT feedback from a webhook and normalizes all inputs\n(tester, source, build, page, message) into a clean, AI-ready data structure. |
| normalize | Code | Normalize inbound fields to `uat.*` | data merge | clean text | ## Ingestion & Normalization\n\nCollects UAT feedback from a webhook and normalizes all inputs\n(tester, source, build, page, message) into a clean, AI-ready data structure. |
| clean text | Code | Sanitize and truncate message for LLM | normalize | AI agent | ## Ingestion & Normalization\n\nCollects UAT feedback from a webhook and normalizes all inputs\n(tester, source, build, page, message) into a clean, AI-ready data structure. |
| AI agent | OpenAI (LangChain) | Classify feedback into strict JSON triage | clean text | parsing and validation | ## AI Triage & Quality Control\n\nAn AI model analyzes the feedback and returns a structured triage\n(type, severity, summary, confidence), which is parsed and validated before execution. |
| parsing and validation | Code | Parse/validate AI JSON; apply fallbacks | AI agent | critical bug | ## AI Triage & Quality Control\n\nAn AI model analyzes the feedback and returns a structured triage\n(type, severity, summary, confidence), which is parsed and validated before execution. |
| critical bug | Jira | Create bug issue in Jira | parsing and validation | engeneering alert | ## Critical Bug Escalation\n\nValidated critical bugs automatically create a Jira issue\nand trigger an engineering Slack alert with full context. |
| engeneering alert | Slack | Post engineering escalation alert | critical bug | compose reply branch 1 | ## Critical Bug Escalation\n\nValidated critical bugs automatically create a Jira issue\nand trigger an engineering Slack alert with full context. |
| compose reply branch 1 | Set | Create tester reply subject/body | engeneering alert | how to contact | ## Closed Loop\n\nNotifies the tester via Slack or email\nand responds to the original webhook\nwith a structured status payload. |
| how to contact | IF | Route to Slack DM vs email | compose reply branch 1 | slack tester; tester email | ## Closed Loop\n\nNotifies the tester via Slack or email\nand responds to the original webhook\nwith a structured status payload. |
| slack tester | Slack | DM a Slack user with reply text | how to contact (true) | Webhook response | ## Closed Loop\n\nNotifies the tester via Slack or email\nand responds to the original webhook\nwith a structured status payload. |
| tester email | Gmail | Email the tester with reply subject/body | how to contact (false) | Webhook response | ## Closed Loop\n\nNotifies the tester via Slack or email\nand responds to the original webhook\nwith a structured status payload. |
| Webhook response | Respond to Webhook | Return status to webhook caller | slack tester; tester email | — | ## Closed Loop\n\nNotifies the tester via Slack or email\nand responds to the original webhook\nwith a structured status payload. |
| Sticky Note4 | Sticky Note | Documentation (how it works) | — | — | ## How it works\n\nThis workflow automates Product UAT critical bug escalation using AI.\n\nWhen a tester submits feedback via a webhook, the workflow normalizes and cleans the input before sending it to an AI model. The AI classifies the feedback, evaluates severity, and generates a structured summary with a confidence score.\n\nValidated critical bugs automatically create a Jira issue, trigger an engineering Slack alert, and notify the tester via Slack or email.\nThe workflow then responds to the original webhook with a structured status payload, ensuring full traceability and fast feedback loops. |
| Sticky Note | Sticky Note | Documentation (ingestion section label) | — | — | ## Ingestion & Normalization\n\nCollects UAT feedback from a webhook and normalizes all inputs\n(tester, source, build, page, message) into a clean, AI-ready data structure. |
| Sticky Note1 | Sticky Note | Documentation (AI section label) | — | — | ## AI Triage & Quality Control\n\nAn AI model analyzes the feedback and returns a structured triage\n(type, severity, summary, confidence), which is parsed and validated before execution. |
| Sticky Note2 | Sticky Note | Documentation (escalation section label) | — | — | ## Critical Bug Escalation\n\nValidated critical bugs automatically create a Jira issue\nand trigger an engineering Slack alert with full context. |
| Sticky Note3 | Sticky Note | Documentation (closed loop label) | — | — | ## Closed Loop\n\nNotifies the tester via Slack or email\nand responds to the original webhook\nwith a structured status payload. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Product UAT Critical Bug Escalation (AI → Jira + Slack)** (or your preferred name).
   - Set execution order (optional): default is fine; JSON uses `v1`.

2. **Add `Webhook` node**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: generate or set to a fixed UUID-like string.
   - This is your entry point.

3. **Add `Set` node for configuration (`data mapping`)**
   - Node type: **Set**
   - Add fields (as needed), e.g.:
     - `cfg.confidenceThreshold` (Number) = `0.6`
     - `cfg.dedupeEnabled` (Boolean) = `false`
     - (Optional placeholders) `cfg.jiraProjectKey`, `cfg.slackChannelEng`, etc.
   - Note: In the provided workflow, these config fields are not consumed by downstream nodes, but keep them if you plan to parameterize later.

4. **Add `Merge` node (`data merge`)**
   - Node type: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - Webhook → Merge (Input 1 / index 0)
     - Set (`data mapping`) → Merge (Input 2 / index 1)

5. **Add `Code` node (`normalize`)**
   - Node type: **Code**
   - Paste logic that maps various possible input fields into:
     - `uat.source`, `uat.tester_name`, `uat.tester_email`, `uat.message_raw`, `uat.build_version`, `uat.page_url`, `uat.screenshot_url`, `uat.received_at`
   - Connect: Merge → Normalize

6. **Add `Code` node (`clean text`)**
   - Node type: **Code**
   - Implement:
     - Strip HTML tags
     - Normalize whitespace
     - Trim and truncate (e.g., 3000 chars)
     - Save to `uat.message_clean`
   - Connect: Normalize → Clean text

7. **Add OpenAI node (`AI agent`)**
   - Node type: **OpenAI (LangChain)** (the `@n8n/n8n-nodes-langchain.openAi` node)
   - Select model: **gpt-5.2** (or an equivalent available in your account)
   - Prompt: instruct “return ONLY JSON” and include your schema and rules.
   - **Credentials:** create/select **OpenAI API** credentials in n8n.
   - Connect: Clean text → AI agent

8. **Add `Code` node (`parsing and validation`)**
   - Node type: **Code**
   - Implement:
     - Extract text from likely response fields (be defensive)
     - `JSON.parse`
     - On parse failure, create fallback triage and set `triage.parse_ok=false`
     - Validate `type`, `severity`, `sentiment`; clamp `confidence`
   - Connect: AI agent → parsing and validation

9. **Add Jira node (`critical bug`)**
   - Node type: **Jira Software Cloud**
   - Operation: **Create Issue** (Bug)
   - Select:
     - Project (choose from list)
     - Issue type: Bug
     - Priority: High
   - Map fields using expressions:
     - Summary: `{{ $json.triage.suggested_title || $json.triage.summary }}`
     - Description: include build/page/screenshot, repro steps, reporter info, source
   - **Credentials:** create/select **Jira Software Cloud API** credentials.
   - Connect: parsing and validation → Jira

10. **Add Slack node for engineering alert (`engeneering alert`)**
    - Node type: **Slack**
    - Operation: **Post message to channel**
    - Channel: pick your engineering channel
    - Text: include severity/summary/build/components and Jira key (`{{ $json.key || $json.id }}`)
    - **Credentials:** create/select **Slack OAuth2** credentials with `chat:write` and channel access.
    - Connect: Jira → Slack alert

11. **Add `Set` node (`compose reply branch 1`)**
    - Node type: **Set**
    - Create:
      - `reply.subject`
      - `reply.body`
    - Use expressions referencing `triage.*`, `uat.*`, and Jira key/id.
    - Connect: Slack alert → compose reply

12. **Add `IF` node (`how to contact`)**
    - Node type: **IF**
    - Condition: String equals
      - Left: `{{ $json.uat.source }}`
      - Right: `slack`
    - Connect: compose reply → IF

13. **Add Slack node for tester notification (`slack tester`)**
    - Node type: **Slack**
    - Operation: **Send message to user** (DM)
    - User: select a user (note: original workflow hard-codes a specific user ID)
    - Text: `{{ $json.reply.body }}`
    - Connect: IF (true) → slack tester

14. **Add Gmail node (`tester email`)**
    - Node type: **Gmail**
    - Operation: **Send**
    - To: `{{ $json.uat.tester_email }}`
    - Subject: `{{ $json.reply.subject }}`
    - Message: `{{ $json.reply.body }}`
    - **Credentials:** create/select **Gmail OAuth2** credentials.
    - Connect: IF (false) → tester email

15. **Add `Respond to Webhook` node (`Webhook response`)**
    - Node type: **Respond to Webhook**
    - Response code: **200**
    - Response payload: include `status`, `triage.type`, `triage.severity`, `triage.confidence`
    - Connect:
      - slack tester → Webhook response
      - tester email → Webhook response

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates Product UAT critical bug escalation using AI. When a tester submits feedback via a webhook, the workflow normalizes and cleans the input before sending it to an AI model. The AI classifies the feedback, evaluates severity, and generates a structured summary with a confidence score. Validated critical bugs automatically create a Jira issue, trigger an engineering Slack alert, and notify the tester via Slack or email. The workflow then responds to the original webhook with a structured status payload, ensuring full traceability and fast feedback loops. | From sticky note “How it works”. |
| Important implementation gap: despite the labels “critical bug” and “validated critical bugs”, there is **no filtering node** checking `triage.type == "CriticalBug"` or confidence/severity thresholds before creating Jira issues and sending alerts. | Workflow logic observation (recommended to add an IF gate if you want true “critical-only” behavior). |
| Tester Slack notification is not dynamic: it always DMs a fixed Slack user ID, not the submitting tester. | Workflow logic observation (consider mapping a tester Slack ID in input). |
| `cfg.*` configuration values are currently not referenced by Jira/Slack nodes (which use internal IDs). | Workflow structure observation (can be improved by parameterizing nodes with `cfg.*`). |