Send AI-prioritized follow-up reminders with GPT-4.1-mini, Gmail and Slack

https://n8nworkflows.xyz/workflows/send-ai-prioritized-follow-up-reminders-with-gpt-4-1-mini--gmail-and-slack-12960


# Send AI-prioritized follow-up reminders with GPT-4.1-mini, Gmail and Slack

## 1. Workflow Overview

**Purpose:** This workflow monitors a Google Sheet for new follow-up tasks, uses an OpenAI (GPT-4.1-mini–style) agent to classify urgency, and then sends prioritized reminders via **Gmail**, **Slack**, and **Twilio SMS**. It also posts a **daily Slack summary** of pending follow-ups and notifies an admin channel when executions error.

**Target use cases:**
- Sales/customer success follow-ups that must be chased based on urgency
- Personal assistant–style “remind me to follow up” pipelines
- Team operations where urgent items require multi-channel escalation

### 1.1 Task Intake & Configuration
Receives a new row/task from Google Sheets and prepares configuration/context.

### 1.2 Data Normalization
Maps raw sheet fields into a consistent structure for AI and downstream routing.

### 1.3 AI Urgency Classification (LLM + Structured Parsing)
An AI Agent calls an OpenAI chat model and uses a structured parser to produce a machine-routable urgency result.

### 1.4 Urgency Routing & Notifications
Routes tasks by urgency and sends appropriate notifications (email, Slack, SMS). Logs outcomes to Sheets.

### 1.5 Daily Summary (Scheduled)
On a schedule, pulls follow-ups from Sheets and posts a digest to Slack.

### 1.6 Error Handling
Catches workflow execution errors and alerts an admin Slack channel.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Task Intake & Configuration
**Overview:** Triggers when a new task is added in Google Sheets, then sets workflow-level configuration values (placeholders in the provided JSON).  
**Nodes involved:** `New Task Entry Trigger`, `Workflow Configuration`

#### Node: New Task Entry Trigger
- **Type / role:** `googleSheetsTrigger` — entrypoint; listens for Google Sheets changes/new rows.
- **Configuration (interpreted):** Parameters are empty in JSON, so the trigger is not fully configured here. In practice it must specify:
  - Spreadsheet ID, Sheet name/tab
  - Trigger event (new row / updated row)
  - Polling interval (or watch mechanism depending on node capabilities)
- **Inputs / outputs:**
  - **Output:** to `Workflow Configuration`
- **Version notes:** TypeVersion `1` (older trigger version; UI differs from later releases).
- **Failure modes / edge cases:**
  - Missing/invalid Google credentials
  - Spreadsheet/tab not found
  - Duplicate triggers if sheet updates modify previously processed rows (unless deduplication is implemented)

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines/overrides fields used by later nodes (e.g., recipients, templates, default values).
- **Configuration (interpreted):** Empty parameters, so currently it does not set values. Intended usage typically includes constants like:
  - Default Slack channel, admin channel
  - Email “from”/signature or escalation recipients
  - Thresholds (e.g., VIP flag rules)
- **Inputs / outputs:**
  - **Input:** from `New Task Entry Trigger`
  - **Output:** to `Normalize Task Data`
- **Version notes:** TypeVersion `3.4`.
- **Failure modes / edge cases:**
  - If later nodes reference config fields that are never set, expressions can evaluate to `null` and break routing/templating.

---

### Block 1.2 — Data Normalization
**Overview:** Converts sheet row fields into a normalized “task object” for the AI agent and downstream notification templates.  
**Nodes involved:** `Normalize Task Data`

#### Node: Normalize Task Data
- **Type / role:** `Set` — field mapping/renaming and cleanup.
- **Configuration (interpreted):** Empty parameters in JSON; in a working workflow, this would typically:
  - Standardize fields like `taskTitle`, `contactEmail`, `contactName`, `dueDate`, `notes`, `vip`, `owner`
  - Ensure types (dates/booleans) are consistent
- **Inputs / outputs:**
  - **Input:** from `Workflow Configuration`
  - **Output:** to `Classify Follow-up Urgency`
- **Version notes:** TypeVersion `3.4`.
- **Failure modes / edge cases:**
  - Missing required columns from Sheets
  - Date parsing issues (locale/timezone)
  - Boolean normalization (e.g., “TRUE”, “Yes”, “1”)

---

### Block 1.3 — AI Urgency Classification (LLM + Structured Parsing)
**Overview:** Uses a LangChain Agent node that calls an OpenAI chat model and emits a structured result (e.g., `HIGH|MEDIUM|LOW`) parsed via a structured output parser.  
**Nodes involved:** `Classify Follow-up Urgency`, `OpenAI Chat Model`, `Structured Output Parser`

#### Node: Classify Follow-up Urgency
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM call(s) and returns an “urgency” decision.
- **Configuration (interpreted):** Empty parameters in JSON, but its wiring indicates:
  - It consumes a language model from `OpenAI Chat Model`
  - It uses `Structured Output Parser` to constrain output into predictable fields
- **Key variables/expressions:** Not visible (empty parameters). Typically would include a system prompt like:
  - “Classify urgency based on due date, VIP, sentiment, and task description…”
- **Inputs / outputs:**
  - **Main input:** from `Normalize Task Data`
  - **Main output:** to `Route by Urgency`
  - **AI language model input:** from `OpenAI Chat Model`
  - **AI output parser input:** from `Structured Output Parser`
- **Version notes:** TypeVersion `3` (LangChain agent nodes evolve quickly; ensure compatibility with your n8n version).
- **Failure modes / edge cases:**
  - Model/API auth errors, rate limits
  - Parser failures if model output doesn’t match schema
  - Nondeterministic classifications without temperature control or strong schema enforcement

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — provides an OpenAI chat completion model to the agent.
- **Configuration (interpreted):** Empty parameters; should define:
  - Model name (user referenced GPT-4.1-mini; in n8n this is set here)
  - Temperature, max tokens, timeout
- **Connections:**
  - **Output (ai_languageModel):** into `Classify Follow-up Urgency`
- **Version notes:** TypeVersion `1.3`.
- **Failure modes / edge cases:**
  - Missing OpenAI credentials
  - Model not available in the account/region
  - Token limits if you pass large task context

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces a schema so AI output can be routed reliably.
- **Configuration (interpreted):** Empty parameters; normally includes a JSON schema or field definitions, e.g.:
  - `urgency`: enum `HIGH|MEDIUM|LOW`
  - Optional: `reason`, `recommendedChannel`, `followUpBy`
- **Connections:**
  - **Output (ai_outputParser):** into `Classify Follow-up Urgency`
- **Version notes:** TypeVersion `1.3`.
- **Failure modes / edge cases:**
  - Schema mismatch (model returns extra text)
  - If schema is too strict, minor formatting deviations can fail the node

---

### Block 1.4 — Urgency Routing & Notifications
**Overview:** Switches execution path by urgency and sends notifications via email/Slack/SMS, then logs status back to Google Sheets.  
**Nodes involved:** `Route by Urgency`, `Send High Urgency Email`, `Send High Urgency Slack Alert`, `Send VIP SMS Reminder`, `Send Medium Urgency Email`, `Add to Weekly Summary`, `Log Follow-up Status`

#### Node: Route by Urgency
- **Type / role:** `Switch` — branches the workflow (High / Medium / Low).
- **Configuration (interpreted):** Empty parameters; but connections show **three outputs**:
  1. Output 0 → High urgency path
  2. Output 1 → Medium urgency path
  3. Output 2 → Low urgency path (“weekly summary” bucket)
- **Inputs / outputs:**
  - **Input:** from `Classify Follow-up Urgency`
  - **Outputs:** to `Send High Urgency Email`, `Send Medium Urgency Email`, `Add to Weekly Summary`
- **Version notes:** TypeVersion `3.4`.
- **Failure modes / edge cases:**
  - If the AI output field used for switching is missing/typoed, items may fall into “no match” (unhandled) unless a default route is configured.

#### Node: Send High Urgency Email
- **Type / role:** `gmail` — sends an immediate email for high urgency tasks.
- **Configuration (interpreted):** Empty parameters; must specify:
  - Operation: Send email
  - To/Subject/Body expressions (likely based on normalized task fields)
- **Inputs / outputs:**
  - **Input:** from `Route by Urgency` (High branch)
  - **Outputs:** fan-out to `Send High Urgency Slack Alert` and `Send VIP SMS Reminder`
- **Version notes:** TypeVersion `2.2`.
- **Failure modes / edge cases:**
  - Gmail OAuth not configured / token expired
  - “To” missing/invalid email from sheet
  - Gmail sending limits / throttling

#### Node: Send High Urgency Slack Alert
- **Type / role:** `slack` — posts an urgent alert to Slack.
- **Configuration (interpreted):** Empty parameters; must define:
  - Channel/user, message text, formatting/blocks
- **Inputs / outputs:**
  - **Input:** from `Send High Urgency Email`
  - **Output:** to `Log Follow-up Status`
- **Version notes:** TypeVersion `2.4`.
- **Failure modes / edge cases:**
  - Slack token missing/invalid
  - Channel not found, missing scopes (e.g., `chat:write`)
  - Message formatting errors if using blocks JSON

#### Node: Send VIP SMS Reminder
- **Type / role:** `twilio` — sends SMS (intended for VIP escalation).
- **Configuration (interpreted):** Empty parameters; typically includes:
  - From number, To number (often from task/contact or owner)
  - SMS body
- **Inputs / outputs:**
  - **Input:** from `Send High Urgency Email`
  - **Output:** to `Log Follow-up Status`
- **Version notes:** TypeVersion `1`.
- **Failure modes / edge cases:**
  - Twilio credentials missing
  - Invalid phone numbers (E.164 formatting)
  - Region/capability restrictions, SMS compliance, messaging service configuration

#### Node: Send Medium Urgency Email
- **Type / role:** `gmail` — sends email for medium urgency tasks (less escalated than high).
- **Configuration (interpreted):** Empty parameters; should define recipients and template.
- **Inputs / outputs:**
  - **Input:** from `Route by Urgency` (Medium branch)
  - **Output:** to `Log Follow-up Status`
- **Version notes:** TypeVersion `2.2`.
- **Failure modes / edge cases:** Same as the high urgency email node.

#### Node: Add to Weekly Summary
- **Type / role:** `Set` — prepares low urgency items for later batching (weekly summary).
- **Configuration (interpreted):** Empty parameters; likely intended to:
  - Mark task status as “Queued for weekly summary”
  - Add categorization fields
- **Inputs / outputs:**
  - **Input:** from `Route by Urgency` (Low branch)
  - **Output:** to `Log Follow-up Status`
- **Version notes:** TypeVersion `3.4`.
- **Failure modes / edge cases:**
  - Without an actual persistence mechanism (DB/Sheet append), a Set node alone won’t truly “queue” items.

#### Node: Log Follow-up Status
- **Type / role:** `googleSheets` — logs the outcome/status back to Google Sheets (or a log sheet).
- **Configuration (interpreted):** Empty parameters; should define:
  - Document & sheet/tab
  - Operation: append row / update row
  - Mapping fields: urgency, channels sent, timestamps, execution id, errors (if any)
- **Inputs / outputs:**
  - **Inputs:** from `Send High Urgency Slack Alert`, `Send VIP SMS Reminder`, `Send Medium Urgency Email`, `Add to Weekly Summary`
  - **Output:** none further
- **Version notes:** TypeVersion `4.7`.
- **Failure modes / edge cases:**
  - Write conflicts / API quota
  - Incorrect row matching if updating
  - Permissions: sheet shared with service account/user?

---

### Block 1.5 — Daily Summary (Scheduled)
**Overview:** Runs on a schedule, fetches follow-ups from Google Sheets, and posts a daily summary to Slack.  
**Nodes involved:** `Daily Summary Schedule`, `Fetch Daily Follow-ups`, `Send Daily Summary to Slack`

#### Node: Daily Summary Schedule
- **Type / role:** `scheduleTrigger` — starts daily summary execution.
- **Configuration (interpreted):** Empty parameters; must define:
  - Interval (daily) and time/timezone
- **Inputs / outputs:**
  - **Output:** to `Fetch Daily Follow-ups`
- **Version notes:** TypeVersion `1.3`.
- **Failure modes / edge cases:**
  - Timezone mismatch causing unexpected run times
  - Missed executions if n8n is down (depending on n8n mode)

#### Node: Fetch Daily Follow-ups
- **Type / role:** `googleSheets` — reads tasks to include in daily summary.
- **Configuration (interpreted):** Empty parameters; normally:
  - Read from a range or filter view
  - Possibly filter by “due today/overdue” or “status != done”
- **Inputs / outputs:**
  - **Input:** from `Daily Summary Schedule`
  - **Output:** to `Send Daily Summary to Slack`
- **Version notes:** TypeVersion `4.7`.
- **Failure modes / edge cases:**
  - Large sheets → pagination/limits
  - Filtering not applied → noisy summaries

#### Node: Send Daily Summary to Slack
- **Type / role:** `slack` — posts a compiled digest message.
- **Configuration (interpreted):** Empty parameters; should define:
  - Target channel and message formatting
  - Potentially loop/aggregate behavior (requires additional nodes in real builds; not present here)
- **Inputs / outputs:**
  - **Input:** from `Fetch Daily Follow-ups`
  - **Output:** none further
- **Version notes:** TypeVersion `2.4`.
- **Failure modes / edge cases:** Slack auth/scope issues; message too long if many tasks.

---

### Block 1.6 — Error Handling
**Overview:** When any workflow execution errors, it triggers a Slack alert to an admin channel.  
**Nodes involved:** `Error Handler Trigger`, `Alert Admin on Error`

#### Node: Error Handler Trigger
- **Type / role:** `errorTrigger` — special trigger that runs on workflow errors.
- **Configuration (interpreted):** Default; fires when another workflow errors (within same n8n instance configuration).
- **Inputs / outputs:**
  - **Output:** to `Alert Admin on Error`
- **Version notes:** TypeVersion `1`.
- **Failure modes / edge cases:**
  - If Slack node fails too, you lose visibility (consider also email/webhook fallback).

#### Node: Alert Admin on Error
- **Type / role:** `slack` — sends error details to admins.
- **Configuration (interpreted):** Empty parameters; should include:
  - Channel, message with error name, node name, execution URL
- **Inputs / outputs:**
  - **Input:** from `Error Handler Trigger`
- **Version notes:** TypeVersion `2.4`.
- **Failure modes / edge cases:** Missing Slack scopes; overly verbose payloads.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Task Entry Trigger | googleSheetsTrigger | Entry trigger for new tasks from Sheets | — | Workflow Configuration |  |
| Workflow Configuration | Set | Establish constants/default config for run | New Task Entry Trigger | Normalize Task Data |  |
| Normalize Task Data | Set | Normalize/rename task fields | Workflow Configuration | Classify Follow-up Urgency |  |
| Classify Follow-up Urgency | LangChain Agent | LLM-based urgency classification | Normalize Task Data; (AI) OpenAI Chat Model; (AI) Structured Output Parser | Route by Urgency |  |
| OpenAI Chat Model | OpenAI Chat Model | Provides LLM to agent | — | Classify Follow-up Urgency (ai_languageModel) |  |
| Structured Output Parser | Structured Output Parser | Constrains LLM output into schema | — | Classify Follow-up Urgency (ai_outputParser) |  |
| Route by Urgency | Switch | Branching by urgency | Classify Follow-up Urgency | Send High Urgency Email; Send Medium Urgency Email; Add to Weekly Summary |  |
| Send High Urgency Email | Gmail | Email escalation for high urgency | Route by Urgency | Send High Urgency Slack Alert; Send VIP SMS Reminder |  |
| Send High Urgency Slack Alert | Slack | Slack escalation for high urgency | Send High Urgency Email | Log Follow-up Status |  |
| Send VIP SMS Reminder | Twilio | SMS escalation (VIP) | Send High Urgency Email | Log Follow-up Status |  |
| Send Medium Urgency Email | Gmail | Email for medium urgency | Route by Urgency | Log Follow-up Status |  |
| Add to Weekly Summary | Set | Mark/prepare low urgency for weekly summary | Route by Urgency | Log Follow-up Status |  |
| Log Follow-up Status | Google Sheets | Writes back status/log | Send High Urgency Slack Alert; Send VIP SMS Reminder; Send Medium Urgency Email; Add to Weekly Summary | — |  |
| Daily Summary Schedule | Schedule Trigger | Daily trigger for digest | — | Fetch Daily Follow-ups |  |
| Fetch Daily Follow-ups | Google Sheets | Fetch tasks for daily summary | Daily Summary Schedule | Send Daily Summary to Slack |  |
| Send Daily Summary to Slack | Slack | Post daily digest | Fetch Daily Follow-ups | — |  |
| Error Handler Trigger | Error Trigger | Starts error flow when executions fail | — | Alert Admin on Error |  |
| Alert Admin on Error | Slack | Notifies admins about errors | Error Handler Trigger | — |  |
| Sticky Note | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note1 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note2 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note3 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note4 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note5 | Sticky Note | Canvas annotation | — | — |  |

> Note: All sticky notes have empty content in the provided workflow.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Intelligent Follow-up Reminders with Multi-Channel Notifications* (or your preferred name).

2. **Add trigger: “Google Sheets Trigger”** (`New Task Entry Trigger`)
   - Configure Google credentials.
   - Select Spreadsheet + Sheet.
   - Choose the event (typically “New Row”).
   - Ensure the trigger produces the row fields you expect (headers as keys).

3. **Add “Set” node** (`Workflow Configuration`)
   - Add static fields you will reuse, e.g.:
     - `slackChannelUrgent`, `slackChannelDaily`, `adminChannel`
     - `fromEmailName`, `replyTo`, etc.
   - Connect: **Trigger → Workflow Configuration**.

4. **Add “Set” node** (`Normalize Task Data`)
   - Map sheet columns into normalized fields, e.g.:
     - `taskTitle`, `contactName`, `contactEmail`, `phone`, `dueDate`, `notes`, `vip`, `owner`
   - Connect: **Workflow Configuration → Normalize Task Data**.

5. **Add LangChain “OpenAI Chat Model” node** (`OpenAI Chat Model`)
   - Configure OpenAI credentials (API key).
   - Select model (e.g., GPT-4.1-mini if available in your n8n version/provider list).
   - Set temperature low (e.g., 0–0.3) for consistent routing.

6. **Add LangChain “Structured Output Parser” node** (`Structured Output Parser`)
   - Define a strict schema, at minimum:
     - `urgency` as enum: `HIGH`, `MEDIUM`, `LOW`
     - (Optional) `reason` string
   - Keep the schema aligned with your Switch conditions.

7. **Add LangChain “AI Agent” node** (`Classify Follow-up Urgency`)
   - Provide instructions/prompt that references your normalized fields.
   - Attach:
     - **OpenAI Chat Model → Agent** via the **AI language model** connection.
     - **Structured Output Parser → Agent** via the **AI output parser** connection.
   - Connect: **Normalize Task Data → Classify Follow-up Urgency**.

8. **Add “Switch” node** (`Route by Urgency`)
   - Set the value to evaluate to your parsed field (e.g., `{{$json.urgency}}` depending on agent output structure).
   - Create 3 cases:
     - `HIGH` → Output 0
     - `MEDIUM` → Output 1
     - `LOW` → Output 2
   - Connect: **Agent → Switch**.

9. **High urgency path**
   - Add **Gmail node** (`Send High Urgency Email`)
     - Configure Gmail OAuth2 credentials.
     - Operation: Send.
     - Set `To`, `Subject`, `Body` using normalized fields.
   - Add **Slack node** (`Send High Urgency Slack Alert`)
     - Configure Slack credentials (bot token).
     - Post to urgent channel; include task details and urgency reason.
   - Add **Twilio node** (`Send VIP SMS Reminder`)
     - Configure Twilio credentials (Account SID/Auth Token).
     - Set From/To and body.
     - If SMS should only send for VIP, add an IF node before Twilio (not present in JSON, but recommended).
   - Wiring:
     - **Switch (HIGH) → Gmail**
     - **Gmail → Slack**
     - **Gmail → Twilio**
     - **Slack → Log Follow-up Status**
     - **Twilio → Log Follow-up Status**

10. **Medium urgency path**
    - Add **Gmail node** (`Send Medium Urgency Email`) with a calmer template.
    - Connect: **Switch (MEDIUM) → Send Medium Urgency Email → Log Follow-up Status**

11. **Low urgency path**
    - Add **Set node** (`Add to Weekly Summary`) to mark the task as “weekly”.
    - Connect: **Switch (LOW) → Add to Weekly Summary → Log Follow-up Status**
    - If you truly need weekly batching, add a persistence step (append to a “Weekly Summary” sheet/tab or datastore).

12. **Add Google Sheets node** (`Log Follow-up Status`)
    - Configure Google credentials.
    - Choose operation (append row or update original row).
    - Store fields like: `taskId/rowId`, `urgency`, `channelsSent`, `timestamp`, `status`.
    - Connect all relevant branches into this node.

13. **Daily summary flow**
    - Add **Schedule Trigger** (`Daily Summary Schedule`) set to run daily at your chosen time/timezone.
    - Add **Google Sheets** (`Fetch Daily Follow-ups`) to read tasks to summarize (filter by status/due date).
    - Add **Slack** (`Send Daily Summary to Slack`) to post the digest.
    - Connect: **Schedule → Fetch → Slack**
    - If you need aggregation/formatting, add an “Aggregate”/“Code” node (not present in JSON).

14. **Error handling flow**
    - Add **Error Trigger** (`Error Handler Trigger`).
    - Add **Slack** (`Alert Admin on Error`) to post error details (node name, message, execution link).
    - Connect: **Error Trigger → Slack**
    - Ensure Slack credentials and admin channel are correct.

15. **Activate the workflow** after testing each branch with sample rows.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow contains sticky note nodes but all have empty content. | Canvas annotations are present but not used. |
| Many nodes have empty parameters in the provided JSON. | To run this workflow, you must configure triggers, sheet IDs/ranges, message templates, switch rules, and credentials. |
| “Weekly summary” is not implemented as a true batch mechanism. | A `Set` node alone won’t persist items; add a storage step (Sheets/DB) plus a weekly schedule to send it. |