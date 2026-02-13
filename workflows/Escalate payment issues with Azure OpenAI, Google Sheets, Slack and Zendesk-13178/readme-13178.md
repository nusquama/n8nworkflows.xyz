Escalate payment issues with Azure OpenAI, Google Sheets, Slack and Zendesk

https://n8nworkflows.xyz/workflows/escalate-payment-issues-with-azure-openai--google-sheets--slack-and-zendesk-13178


# Escalate payment issues with Azure OpenAI, Google Sheets, Slack and Zendesk

## 1. Workflow Overview

**Purpose:**  
This workflow receives payment-issue reports via a webhook, validates the claim against a **Google Sheets “source of truth”**, uses **Azure OpenAI (GPT-4o via LangChain nodes)** to generate a structured escalation summary, then (for confirmed escalation paths) notifies human support via **Gmail + Slack**, creates a **Zendesk ticket**, sends a **customer-safe email**, and finally **updates the Google Sheet** to mark that a confirmation email was sent. A separate **Error Trigger** path alerts Slack on failures.

**Target use cases:**
- Payment succeeded but order not created / failed
- Confirmation email not sent
- Reduce false escalations by enforcing sheet-based validation
- Consistent internal + customer communication

### 1.1 Webhook Intake & Normalization
Receives incoming structured issue payload and normalizes it into a predictable `body` object.

### 1.2 Transaction Lookup (Source of Truth)
Provides a Google Sheets tool to the AI agent so it can fetch authoritative transaction details.

### 1.3 AI Escalation Decision & Structured Output
An AI agent cross-checks issue data with sheet data, classifies the case, and produces a **structured** `{ Subject, Body }` output for internal escalation.

### 1.4 Human Support Notification + Ticketing
Uses the AI escalation summary to:
- email human support (Gmail),
- log the escalation to Slack,
- create a Zendesk ticket.

### 1.5 Customer Email Generation + Delivery
Generates a customer-facing email using AI (separate model + parser) and sends it via Gmail.

### 1.6 Record Update & Confirmation Tracking
Marks the sheet record as confirmation email sent (`yes`), matching by `transaction_id`.

### 1.7 Error Monitoring
On any workflow failure, posts an alert into a Slack channel.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Webhook Intake & Normalization
**Overview:** Accepts POST requests from an external system (e.g., Chatbase or another support intake layer) and reshapes the incoming payload into a stable structure for downstream expressions.

**Nodes involved:**
- Webhook Listener for Incoming Data
- Set Lead Data for Processing

#### Node: Webhook Listener for Incoming Data
- **Type / role:** `Webhook` — entry point to start the workflow on HTTP POST.
- **Configuration (interpreted):**
  - Method: `POST`
  - Path: `dcb80d6b-8159-427d-ad06-979648422abf` (this becomes the endpoint path)
- **Inputs/Outputs:**
  - No input (trigger node)
  - Output → `Set Lead Data for Processing`
- **Edge cases / failures:**
  - Invalid/missing JSON body (downstream expressions like `$json.body.intent` will fail or become `undefined`)
  - Authentication not enforced (no webhook auth options configured); risk of unsolicited calls unless handled at n8n level (e.g., webhook auth, firewall, or secret token)
- **Version notes:** Webhook node v2.1 supports response modes/options; none are configured here.

#### Node: Set Lead Data for Processing
- **Type / role:** `Set` — normalizes the incoming request into a single `body` object.
- **Configuration (interpreted):**
  - Creates field `body` (object) = `{{$json.body}}`
- **Key expressions / variables:**
  - `{{ $json.body }}` copied verbatim
- **Connections:**
  - Input ← Webhook
  - Output → Generate Escalation Analysis Email (AI)
- **Edge cases / failures:**
  - If webhook payload doesn’t contain `body`, `body` becomes `null/undefined`, breaking later usage (`$json.body.intent`, etc.)

---

### Block 1.2 — Transaction Lookup (Source of Truth)
**Overview:** Exposes a Google Sheets “tool” that the AI agent can invoke to retrieve transaction/order records, ensuring decisions are based on authoritative data.

**Nodes involved:**
- Retrieve Transaction Data from Google Sheets

#### Node: Retrieve Transaction Data from Google Sheets
- **Type / role:** `googleSheetsTool` — an AI Tool node (used by LangChain Agent) to query Sheets.
- **Configuration (interpreted):**
  - Spreadsheet: `sample_leads_50` (document ID `17rcNd_...`)
  - Sheet/tab: `Support_Transactions` (gid `2108445943`)
  - Operation specifics are not visible in the JSON excerpt (typical tool behavior: agent chooses how to query).
- **Credentials:** Google Sheets OAuth2 (`automations@techdome.ai`)
- **Connections:**
  - Tool output is connected to the agent via `ai_tool` → `Generate Escalation Analysis Email (AI)`
- **Edge cases / failures:**
  - OAuth token expiration/consent revoked
  - Sheet renamed / gid changed
  - Agent may fail to locate a row if there is no consistent key (e.g., transaction_id mismatch)
  - Google API quota limits or transient 429/5xx errors
- **Version notes:** Node typeVersion 4.7.

---

### Block 1.3 — AI Escalation Decision & Structured Output (Internal)
**Overview:** A LangChain Agent uses structured issue data plus the Sheets tool to decide whether escalation is warranted. It must output structured JSON for downstream email/Slack/Zendesk usage.

**Nodes involved:**
- Generate Escalation Analysis Email (AI)
- Execute Trade Decision Reasoning with Azure OpenAI
- Parse Structured AI Output for Email

#### Node: Execute Trade Decision Reasoning with Azure OpenAI
- **Type / role:** `lmChatAzureOpenAi` — Azure OpenAI chat model provider for the agent.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Options: default
- **Credentials:** Azure OpenAI account
- **Connections:**
  - Connected as `ai_languageModel` → `Generate Escalation Analysis Email (AI)`
- **Edge cases / failures:**
  - Azure deployment/model mismatch (in Azure, “model” often maps to a deployment name depending on setup)
  - 401/403 (key/endpoint), 429 throttling, timeouts
- **Version notes:** typeVersion 1.

#### Node: Parse Structured AI Output for Email
- **Type / role:** `outputParserStructured` — enforces JSON structure from the agent.
- **Configuration (interpreted):**
  - Expected schema example:
    - `Subject` (string)
    - `Body` (string)
- **Connections:**
  - Parser connected into agent via `ai_outputParser` → `Generate Escalation Analysis Email (AI)`
- **Edge cases / failures:**
  - If the agent returns non-JSON or missing keys, parsing fails and triggers error flow.

#### Node: Generate Escalation Analysis Email (AI)
- **Type / role:** `LangChain Agent` — orchestrates reasoning, tool use (Sheets), and produces a structured escalation summary.
- **Configuration (interpreted):**
  - Prompt includes:
    - Incoming issue fields from `{{$json.body.*}}`
    - “Google Sheet lookup result” referenced as `{{$json.sheet.*}}`
  - System rules (key logic):
    - Google Sheet is source of truth
    - Classify: `CONFIRMED`, `NEEDS CLARIFICATION`, `NOT VALID`
    - Only recommend escalation when:
      - `payment_status` is `SUCCESS` AND
      - (`order_status` is `NOT_CREATED` OR `FAILED`) OR `confirmation_email_sent` is `NO`
    - Must not invent missing data
    - Must not mention internal systems/AI/automation
  - Output is expected to be structured (via parser) as `output.Subject` and `output.Body`.
- **Important implementation note (potential data-shape gap):**
  - The prompt references `{{$json.sheet.payment_status}}` etc. However, no explicit non-AI node maps sheet results into `$json.sheet`. This implies the agent is expected to fetch and internalize sheet data via the Sheets tool, but **the prompt also expects it already present in the input JSON**. If the agent does not populate a `sheet` object in the final context, those expressions may resolve to `undefined`.
- **Connections:**
  - Input ← Set Lead Data for Processing
  - AI tool ← Retrieve Transaction Data from Google Sheets
  - AI model ← Execute Trade Decision Reasoning with Azure OpenAI
  - AI output parser ← Parse Structured AI Output for Email
  - Main outputs → (fan-out)
    - Send Escalation Notification to Human Support
    - Generate Customer-Facing Support Email (AI)
    - Log Human Escalation to Slack
- **Edge cases / failures:**
  - Missing `body.intent` / `issue_summary` / `transaction_id` causing poor decisions or parser failure
  - Sheets lookup ambiguity (multiple matches, no match)
  - Parser failure if output does not match schema
  - Logical risk: the workflow **always fans out** to escalation actions; there is **no IF/Switch** node enforcing “only for confirmed cases”. The “only recommend escalation when …” rule exists in text, but the automation still proceeds unless additional gating exists (it doesn’t in this JSON).

---

### Block 1.4 — Human Support Notification + Ticketing
**Overview:** Sends internal notifications (Gmail + Slack) and creates a Zendesk ticket using the AI-generated escalation summary.

**Nodes involved:**
- Send Escalation Notification to Human Support
- Log Human Escalation to Slack
- Create a ticket

#### Node: Send Escalation Notification to Human Support
- **Type / role:** `Gmail` — sends the internal escalation email.
- **Configuration (interpreted):**
  - Subject: `{{$json.output.Subject}}`
  - Message body: `{{$json.output.Body}}`
  - `sendTo` is set to an expression `"="` but no actual address is provided in the JSON. This likely must be configured (e.g., `support@company.com`).
- **Credentials:** Gmail OAuth2
- **Connections:**
  - Input ← Generate Escalation Analysis Email (AI)
- **Edge cases / failures:**
  - Missing recipient address (most likely currently misconfigured)
  - Gmail API permissions, token expiration
  - If `output.Subject/Body` missing due to parser/agent issues

#### Node: Log Human Escalation to Slack
- **Type / role:** `Slack` — logs escalation details for visibility and triage.
- **Configuration (interpreted):**
  - Posts a formatted message including:
    - Source / Intent from `Set Lead Data for Processing` → `body.source`, `body.intent`
    - AI escalation summary from `{{$json.output.Body}}`
    - Customer details (email, transaction_id, payment_method)
  - Target: Slack **user** selected (`newscctv22`, id `U09HMPVD466`) rather than a channel.
- **Credentials:** Slack API
- **Connections:**
  - Input ← Generate Escalation Analysis Email (AI)
  - Output → Set Lead data  AND Create a ticket (two downstream nodes)
- **Edge cases / failures:**
  - Posting to a user requires permissions and correct Slack scopes
  - Formatting issues if fields are missing
  - If the workflow should only log confirmed escalations, there is no gating here.

#### Node: Create a ticket
- **Type / role:** `Zendesk` — creates a support ticket for human follow-up.
- **Configuration (interpreted):**
  - Ticket description is composed from:
    - `body.source`, `body.intent`
    - Escalation summary from `Generate Escalation Analysis Email (AI).output.Body`
    - Customer/transaction context
  - Additional fields: none configured
  - (Subject/priority/assignee not shown; likely defaults)
- **Credentials:** Zendesk API
- **Connections:**
  - Input ← Log Human Escalation to Slack
- **Edge cases / failures:**
  - Zendesk auth errors, missing required fields (some Zendesk setups require subject, requester, etc.)
  - Rate limiting
  - If the referenced node output path changes or is empty

---

### Block 1.5 — Customer Email Generation + Delivery
**Overview:** Converts internal escalation context into a customer-safe email, parses it into `{Subject, Body}`, then sends it to the customer via Gmail.

**Nodes involved:**
- Generate Customer-Facing Support Email (AI)
- Execute Trade Decision Reasoning for Customer Email (Azure)
- parse Customer Email Output for Sending
- Send  Email to Customer

#### Node: Execute Trade Decision Reasoning for Customer Email (Azure)
- **Type / role:** `lmChatAzureOpenAi` — model for customer email generation.
- **Configuration:** `gpt-4o`
- **Credentials:** Azure OpenAI account
- **Connections:** `ai_languageModel` → Generate Customer-Facing Support Email (AI)
- **Edge cases:** same as other Azure node (deployment mismatch, throttling, etc.)

#### Node: parse Customer Email Output for Sending
- **Type / role:** `outputParserStructured` — enforces `{Subject, Body}` output.
- **Connections:** `ai_outputParser` → Generate Customer-Facing Support Email (AI)
- **Edge cases:** parsing failures if the model doesn’t respect the format.

#### Node: Generate Customer-Facing Support Email (AI)
- **Type / role:** `LangChain Agent` — generates a customer-friendly email, explicitly hiding internal language.
- **Configuration (interpreted):**
  - Uses customer details from `Set Lead Data for Processing` (email, transaction_id, payment_method)
  - Uses “Issue summary” from `{{$json.output.Body}}` (this is the internal AI escalation body coming from the previous AI agent branch)
  - System message emphasizes:
    - polite/empathetic tone
    - don’t expose internal statuses/escalation language
    - don’t promise refunds or timelines
  - Output format: Subject/Body structured
- **Connections:**
  - Input ← Generate Escalation Analysis Email (AI)
  - Output → Send  Email to Customer
- **Edge cases / failures:**
  - If internal `output.Body` is not suitable as “issue summary”, customer email may reflect internal phrasing
  - No gating: customer email will be sent even if escalation is NOT VALID unless the agent self-suppresses (not enforced by logic)

#### Node: Send  Email to Customer
- **Type / role:** `Gmail` — sends the generated customer email.
- **Configuration (interpreted):**
  - Subject: `{{$json.output.Subject}}`
  - Message: `{{$json.output.Body}}`
  - Recipient (`sendTo`) is again an empty expression `"="` in JSON; this must be set (typically `={{ $('Set Lead Data for Processing').item.json.body.email }}`).
- **Credentials:** Gmail OAuth2
- **Connections:**
  - Input ← Generate Customer-Facing Support Email (AI)
- **Edge cases / failures:**
  - Missing recipient (currently likely misconfigured)
  - Gmail auth/scopes
  - If parser failed and `output.*` missing

---

### Block 1.6 — Record Update & Confirmation Tracking
**Overview:** After logging/escalation steps, marks the transaction record in Google Sheets as “confirmation email sent”.

**Nodes involved:**
- Set Lead data 
- Update Lead Record in Google Sheets

#### Node: Set Lead data 
- **Type / role:** `Set` — sets `confirmation_email_sent` to `"yes"`.
- **Connections:**
  - Input ← Log Human Escalation to Slack
  - Output → Update Lead Record in Google Sheets
- **Edge cases:**
  - This runs after Slack logging, not after successfully sending the customer email; could mark “sent” even if Gmail failed.

#### Node: Update Lead Record in Google Sheets
- **Type / role:** `Google Sheets` — updates an existing row by matching `transaction_id`.
- **Configuration (interpreted):**
  - Operation: `update`
  - Matching column: `transaction_id`
  - Writes:
    - `transaction_id` = `{{ $('Set Lead Data for Processing').item.json.body.transaction_id }}`
    - `confirmation_email_sent` = `{{ $json.confirmation_email_sent }}`
  - Spreadsheet/tab: same as lookup (`sample_leads_50` → `Support_Transactions`)
- **Credentials:** Google Sheets OAuth2
- **Connections:**
  - Input ← Set Lead data
- **Edge cases / failures:**
  - No matching row for `transaction_id` → update fails or updates nothing (depends on node behavior)
  - Data type mismatch (string/number)
  - Concurrency: multiple updates for same transaction_id
- **Version notes:** Google Sheets node typeVersion 4.

---

### Block 1.7 — Error Monitoring
**Overview:** Captures any workflow failure (node error) and posts an alert to a Slack channel for rapid triage.

**Nodes involved:**
- Error Trigger
- Alert on Workflow Failure

#### Node: Error Trigger
- **Type / role:** `Error Trigger` — special trigger that runs when the workflow errors.
- **Connections:** Output → Alert on Workflow Failure
- **Edge cases:** Only runs for executions that fail; does not catch logical “soft failures” (e.g., AI output says NOT VALID).

#### Node: Alert on Workflow Failure
- **Type / role:** `Slack` — posts a fixed error message to a channel.
- **Configuration (interpreted):**
  - Channel: `workflow-errors` (id `C09GNB90TED`)
  - Text: `"error occured in the workflow , please check ."`
- **Credentials:** Slack API
- **Edge cases:** Slack permission/scopes, channel ID changes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Listener for Incoming Data | webhook | Entry point (HTTP POST intake) | — | Set Lead Data for Processing | ## 📥 Webhook Intake & Normalization; Receives structured issue data from external systems and prepares it for downstream processing. |
| Set Lead Data for Processing | set | Normalize payload to `body` | Webhook Listener for Incoming Data | Generate Escalation Analysis Email (AI) | ## 📥 Webhook Intake & Normalization; Normalizes request body; ensures consistent input structure. |
| Retrieve Transaction Data from Google Sheets | googleSheetsTool | AI tool for authoritative transaction lookup | — (tool) | Generate Escalation Analysis Email (AI) (ai_tool) | ## 📊 Transaction Lookup (Source of Truth); Sheet treated as source of truth; prevents false escalations. |
| Execute Trade Decision Reasoning with Azure OpenAI | lmChatAzureOpenAi | LLM provider for escalation agent | — (model) | Generate Escalation Analysis Email (AI) (ai_languageModel) | ## 🧠 AI Escalation Decision & Analysis; Applies strict escalation rules; produces human-readable summary. |
| Parse Structured AI Output for Email | outputParserStructured | Enforce `{Subject, Body}` structure | — (parser) | Generate Escalation Analysis Email (AI) (ai_outputParser) | ## 🧠 AI Escalation Decision & Analysis; Produces human-readable escalation summary. |
| Generate Escalation Analysis Email (AI) | langchain.agent | Decide escalation + generate structured internal summary | Set Lead Data for Processing | Send Escalation Notification to Human Support; Generate Customer-Facing Support Email (AI); Log Human Escalation to Slack | ## 🧠 AI Escalation Decision & Analysis; Confirms/rejects reported issues using sheet as truth. |
| Send Escalation Notification to Human Support | gmail | Email internal support team | Generate Escalation Analysis Email (AI) | — | ## 🚨 Human Support Notification; Sends escalation email to support. |
| Log Human Escalation to Slack | slack | Log escalation details to Slack | Generate Escalation Analysis Email (AI) | Set Lead data ; Create a ticket | ## 🚨 Human Support Notification; Logs detailed context in Slack. |
| Create a ticket | zendesk | Create Zendesk ticket for tracking | Log Human Escalation to Slack | — | ## 🎫 Zendesk Ticket Creation (Human Escalation); Uses AI summary + customer context for traceable follow-up. |
| Execute Trade Decision Reasoning for Customer Email (Azure) | lmChatAzureOpenAi | LLM provider for customer email agent | — (model) | Generate Customer-Facing Support Email (AI) (ai_languageModel) | ## ✉️ Customer-Facing Support Email; Generates calm, professional customer email. |
| parse Customer Email Output for Sending | outputParserStructured | Enforce `{Subject, Body}` for customer email | — (parser) | Generate Customer-Facing Support Email (AI) (ai_outputParser) | ## ✉️ Customer-Facing Support Email; Ensures output is sendable. |
| Generate Customer-Facing Support Email (AI) | langchain.agent | Convert internal context into customer-safe email | Generate Escalation Analysis Email (AI) | Send  Email to Customer | ## ✉️ Customer-Facing Support Email; Do not expose internal logic; reassure customer. |
| Send  Email to Customer | gmail | Send customer email | Generate Customer-Facing Support Email (AI) | — | ## ✉️ Customer-Facing Support Email; Sends final message to customer. |
| Set Lead data  | set | Set confirmation flag to “yes” | Log Human Escalation to Slack | Update Lead Record in Google Sheets | ## 📝 Record Update & Confirmation Tracking; Marks confirmation email as sent. |
| Update Lead Record in Google Sheets | googleSheets | Update sheet row by `transaction_id` | Set Lead data  | — | ## 📝 Record Update & Confirmation Tracking; Keeps transaction data in sync for auditability. |
| Error Trigger | errorTrigger | Start error-handling execution | — | Alert on Workflow Failure | ## ⚠️ Error Monitoring; Catches workflow failures and sends alerts to Slack. |
| Alert on Workflow Failure | slack | Post failure alert to Slack channel | Error Trigger | — | ## ⚠️ Error Monitoring; Enables quick troubleshooting via Slack alerts. |
| Sticky Note / Sticky Note1…9 | stickyNote | Documentation only | — | — | (Sticky notes are informational and do not execute) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger: Webhook**
   1. Add node: **Webhook**
   2. Set **HTTP Method** = `POST`
   3. Set **Path** to a unique path (e.g., `support-escalation`)
   4. Save to generate the production/test URLs.

2) **Normalize incoming payload**
   1. Add node: **Set**
   2. Add field:
      - Name: `body`
      - Type: **Object**
      - Value: `={{ $json.body }}`
   3. Connect: **Webhook → Set**

3) **Add AI escalation agent (internal)**
   1. Add node: **AI Agent** (LangChain Agent)
   2. Set **Prompt** to include your issue fields (intent, issue_summary, email, transaction_id, payment_method, conversation_id).
   3. Add a **System Message** with rules:
      - Google Sheets is source of truth
      - Output classification (CONFIRMED/NEEDS CLARIFICATION/NOT VALID)
      - Escalate only under the specified sheet conditions
      - No invented data; no mention of internal systems/AI
   4. Enable **Structured Output** (via output parser connection in steps below).
   5. Connect: **Set → AI Agent**

4) **Attach Azure OpenAI model to internal agent**
   1. Add node: **Azure OpenAI Chat Model** (lmChatAzureOpenAi)
   2. Select model/deployment: `gpt-4o` (ensure your Azure deployment matches)
   3. Configure **Azure OpenAI credentials**:
      - Endpoint, API key, API version, deployment name (as required by your n8n credential)
   4. Connect as **AI Language Model** into the agent.

5) **Attach Google Sheets Tool to internal agent**
   1. Add node: **Google Sheets Tool**
   2. Choose **OAuth2 credentials** for Google Sheets
   3. Select the **Spreadsheet** and **Sheet tab** (e.g., `Support_Transactions`)
   4. Connect as **AI Tool** into the agent.

6) **Attach Structured Output Parser to internal agent**
   1. Add node: **Structured Output Parser**
   2. Provide schema example:
      - `Subject` (string)
      - `Body` (string)
   3. Connect as **AI Output Parser** into the agent.

7) **Internal notifications (Gmail + Slack)**
   1. Add node: **Gmail** (internal support email)
      - To: set an explicit address (e.g., `support@yourdomain.com`)
      - Subject: `={{ $json.output.Subject }}`
      - Message: `={{ $json.output.Body }}`
      - Configure **Gmail OAuth2 credentials**
   2. Add node: **Slack**
      - Post to a **channel** or **user** (recommended: channel like `#support-escalations`)
      - Message text referencing:
        - `={{ $('Set Lead Data for Processing').item.json.body.email }}`
        - `={{ $json.output.Body }}`
      - Configure **Slack API credentials**
   3. Connect: **AI Agent → Gmail (internal)** and **AI Agent → Slack**

8) **Zendesk ticket creation**
   1. Add node: **Zendesk**
   2. Operation: **Create Ticket**
   3. Set **Description** using the escalation summary and customer context (as in the provided workflow).
   4. Configure **Zendesk API credentials** (subdomain, token/password as required).
   5. Connect: **Slack → Zendesk** (or connect directly from the agent; either is fine).

9) **Customer email generation (separate AI agent)**
   1. Add node: **AI Agent** (second agent for customer email)
   2. Prompt: instruct it to produce a short customer-safe email; do not mention escalation/internal statuses.
   3. Add **Azure OpenAI Chat Model** (gpt-4o) and connect as AI model.
   4. Add **Structured Output Parser** with `{Subject, Body}` and connect as parser.
   5. Connect: **Internal AI Agent → Customer AI Agent** (so it can reuse the internal summary).

10) **Send customer email**
   1. Add node: **Gmail**
      - To: `={{ $('Set Lead Data for Processing').item.json.body.email }}`
      - Subject: `={{ $json.output.Subject }}`
      - Message: `={{ $json.output.Body }}`
   2. Connect: **Customer AI Agent → Gmail (customer)**

11) **Update confirmation flag in Google Sheets**
   1. Add node: **Set**
      - Field: `confirmation_email_sent` = `yes`
   2. Add node: **Google Sheets** (not the tool)
      - Operation: **Update**
      - Spreadsheet/Sheet: same transaction sheet
      - Match column(s): `transaction_id`
      - Values:
        - `transaction_id` = `={{ $('Set Lead Data for Processing').item.json.body.transaction_id }}`
        - `confirmation_email_sent` = `={{ $json.confirmation_email_sent }}`
      - Credentials: Google Sheets OAuth2
   3. Connect: **Slack (or post-email step) → Set → Google Sheets Update**
   - Recommended improvement: connect this update **after** the customer email successfully sends.

12) **Add error monitoring**
   1. Add node: **Error Trigger**
   2. Add node: **Slack**
      - Channel: `#workflow-errors`
      - Message: include execution info (recommended), or a fixed alert as in current workflow
   3. Connect: **Error Trigger → Slack**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI-Powered Chatbase Customer Support Escalation Workflow” description, setup checklist, and operational intent | From the main sticky note content embedded in the canvas (internal documentation block). |
| Credential requirements: Webhook access, Google Sheets OAuth2, Azure OpenAI, Gmail OAuth2, Slack API, Zendesk API; best practices: treat Sheets as authoritative; avoid exposing internal escalation logic; rotate keys | From “🔐 Required Credentials & Access” sticky note content. |
| Reliability note: error monitoring posts alerts to Slack channel `workflow-errors` | From “⚠️ Error Monitoring” sticky note content. |

Disclamer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.