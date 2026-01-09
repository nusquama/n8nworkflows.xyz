Monitor customer risk and AI feedback using PostgreSQL, Gmail and Discord

https://n8nworkflows.xyz/workflows/monitor-customer-risk-and-ai-feedback-using-postgresql--gmail-and-discord-12254


# Monitor customer risk and AI feedback using PostgreSQL, Gmail and Discord

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Customer Health Monitoring & Product Feedback Intelligence  
**Stated title:** Monitor customer risk and AI feedback using PostgreSQL, Gmail and Discord

### Purpose
This workflow monitors customer “health” (churn/escalation risk) by combining **PostgreSQL customer/payment/complaint data** with **scheduled reporting**. It produces:
- **Daily risk summaries** (Gmail + Discord) and logs notification status to **Google Sheets**
- **Weekly AI product feedback summaries** by turning raw customer issues into executive-ready feedback via a Groq/OpenAI-compatible LLM endpoint, then emailing the summary and logging AI results to Google Sheets

### Main logical blocks
**1.1 Triggers & mode selection (Daily vs Weekly)**  
Two schedule triggers set a flag (`isDaily` or `isWeekly`) and route execution via a Switch node.

**1.2 Daily risk evaluation & escalation output**  
Fetches customer/payment/complaint aggregates from PostgreSQL, classifies each customer into Low vs Very_High risk, merges results, sends daily summaries (email + Discord), and logs “sent” status to Google Sheets.

**1.3 Weekly AI feedback generation & weekly email report**  
Reads previously logged risk rows from Google Sheets, loops rows through an LLM prompt + HTTP call, merges AI feedback back into the original customer row, writes AI feedback into Google Sheets, waits (rate limiting), and finally emails an HTML weekly report.

---

## 2. Block-by-Block Analysis

### 2.1 Block: Triggers & Mode Selection

**Overview:**  
Starts the workflow on a daily and weekly schedule and routes the run to the appropriate branch using flags stored in the execution data.

**Nodes involved:**
- Daily Risk Check Trigger
- Weekly schedule1
- Edit Fields3
- Edit Fields2
- Switch1

#### Node: Daily Risk Check Trigger
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — daily entry point.
- **Configuration (interpreted):** Uses default interval (`rule.interval: [{}]`). In n8n this usually means “every day” (but you should confirm in UI; empty interval objects can be ambiguous depending on version).
- **Outputs:** Connects to **Edit Fields3**.
- **Edge cases / failures:**
  - Misconfigured schedule rule may not fire as expected due to the empty `{}` interval definition.

#### Node: Weekly schedule1
- **Type / role:** Schedule Trigger — weekly entry point.
- **Configuration:** Runs every week, **Monday at 09:00** (`triggerAtDay: [1]`, `triggerAtHour: 9`).
- **Outputs:** Connects to **Edit Fields2**.
- **Edge cases:**
  - Server timezone affects “09:00”; ensure n8n instance timezone matches business expectation.

#### Node: Edit Fields3
- **Type / role:** Set node (`n8n-nodes-base.set`) — sets run mode for daily.
- **Configuration:** Adds field `isDaily = "true"` (string).
- **Outputs:** To **Switch1**.
- **Notes:** Using `"true"` as a string requires strict string comparison downstream (which is done).
- **Edge cases:** If later changed to boolean `true`, Switch rules must be updated.

#### Node: Edit Fields2
- **Type / role:** Set node — sets run mode for weekly.
- **Configuration:** Adds field `isWeekly = "true"` (string).
- **Outputs:** To **Switch1**.

#### Node: Switch1
- **Type / role:** Switch node (`n8n-nodes-base.switch`) — routes execution.
- **Configuration:**
  - Rule 1: if `{{$json.isDaily}} == "true"` → output 0
  - Rule 2: if `{{$json.isWeekly}} == "true"` → output 1
- **Connections:**
  - Output 0 → **Fetch Customer Risk Data** (daily branch)
  - Output 1 → **Get row(s) in sheet1** (weekly branch)
- **Edge cases:**
  - If neither flag is set (manual execution), nothing matches and no branch runs.
  - If both flags exist and both equal `"true"`, only the first matching rule behavior depends on Switch settings (verify Switch configuration in UI).

---

### 2.2 Block: Daily Risk Evaluation & Escalation

**Overview:**  
Queries PostgreSQL to aggregate complaints and payment status per customer/product, evaluates whether the customer is “high risk” per rule logic, formats escalation records, merges them into one stream, sends notifications, and logs notification status to Google Sheets.

**Nodes involved:**
- Fetch Customer Risk Data
- Is High Risk Customer?
- Prepare Escalation Summary For Low Risk User
- Prepare Escalation Summary For High Risk User
- Merge Risk Result
- Send a message4 (Gmail)
- Send a message5 (Discord)
- Code in JavaScript3
- Append or update row in sheet3

#### Node: Fetch Customer Risk Data
- **Type / role:** Postgres node (`n8n-nodes-base.postgres`) — database query.
- **Configuration:**
  - Operation: Execute Query
  - Query returns:
    - `customer_id, customer_name, email`
    - `product_name`
    - `payment_status` from `payment_history.status`
    - `complaint_count` = `COUNT(customer_complaint.complaint_id)`
    - `complaints` = `STRING_AGG(cc.complaint, ' | ')`
  - Groups by customer + product + payment status.
- **Outputs:** To **Is High Risk Customer?**
- **Edge cases / failures:**
  - DB credentials/network issues; query timeouts.
  - If `payment_history` has multiple rows per customer/product, grouping may produce multiple records (and potentially duplicated customers).
  - `COUNT()` with LEFT JOIN counts 0 correctly only if `complaint_id` is NULL; OK.
  - `complaints` can be NULL when no complaints; downstream nodes currently use `reason`, not `complaints` (see mismatch note below).

#### Node: Is High Risk Customer?
- **Type / role:** IF node (`n8n-nodes-base.if`) — risk classification gate.
- **Configuration (as implemented):** “True” only if:
  1) `payment_status == "success"` AND  
  2) `complaint_count == "1"` (string comparison)
- **Connections:**
  - **True branch (index 0)** → Prepare Escalation Summary For Low Risk User
  - **False branch (index 1)** → Prepare Escalation Summary For High Risk User
- **Important logic note:** Despite the name, the condition currently identifies a **low-risk** scenario (payment success + only 1 complaint). Everything else is treated as **Very_High** risk.
- **Edge cases:**
  - `complaint_count` returned from SQL is numeric; comparing to string `"1"` may fail depending on n8n casting/typeValidation. You used strict validation; ensure the value is indeed a string. Otherwise, change right side to `1` or cast in SQL.
  - Payment status “success” must exactly match DB values; if DB uses “paid”, “succeeded”, etc., rule won’t match.

#### Node: Prepare Escalation Summary For Low Risk User
- **Type / role:** Set node — normalize output fields for low risk.
- **Configuration:**
  - Copies customer/product/payment/complaint_count
  - Sets:
    - `risk_level = "Low"`
    - `reason = "Payment is successfull and the compliants are low"`
    - `action_required = "No"`
- **Outputs:** To **Merge Risk Result** (input 0).
- **Edge cases:**
  - Typos in reason text (“successfull”, “compliants”) are user-facing in emails/logs.
  - This node does not use the aggregated `complaints` text from SQL.

#### Node: Prepare Escalation Summary For High Risk User
- **Type / role:** Set node — normalize output fields for high risk.
- **Configuration:**
  - Copies customer/product/payment/complaint_count
  - Sets:
    - `risk_level = "Very_High"`
    - `reason = "Payment failure or multiple complaints detected"`
    - `action_required = "Immediate follow-up needed"`
- **Outputs:** To **Merge Risk Result** (input 1).
- **Edge cases:** Same “complaints” mismatch—raw complaint details are not included; it uses a generic reason string.

#### Node: Merge Risk Result
- **Type / role:** Merge node (`n8n-nodes-base.merge`) — recombines low/high branches into one stream.
- **Configuration:** Default merge behavior (no explicit mode shown). In practice, this node is used as a join point for the IF’s two outputs.
- **Outputs:** Fan-out to **Send a message4** and **Send a message5**.
- **Edge cases:**
  - Depending on merge mode, ordering and completeness may vary. For a simple recombine, “Append” mode is typically desired; verify in UI.

#### Node: Send a message4
- **Type / role:** Gmail node (`n8n-nodes-base.gmail`) — sends daily HTML email summary.
- **Configuration:**
  - To: `user@example.com`
  - Subject: “🚨 Daily Customer Risk Summary – Action Required”
  - Body: HTML tables built with expressions:
    - High-risk table: `risk_level !== 'Low'`
    - Low-risk table: `risk_level === 'Low'`
    - Uses `$items()` to include all items from the current execution.
  - Columns include `reason` (generic text from Set nodes).
- **Outputs:** To **Code in JavaScript3** (for logging).
- **Edge cases / failures:**
  - Gmail OAuth credential expiration / insufficient scopes.
  - Large `$items()` may exceed email size limits.
  - HTML rendering differences in email clients.

#### Node: Send a message5
- **Type / role:** Discord node (`n8n-nodes-base.discord`) — posts daily risk report to a channel via webhook or bot integration.
- **Configuration:**
  - Resource: message
  - Content built from `$items()` with two sections:
    - High risk: `risk_level !== 'Low'`
    - Low risk: `risk_level === 'Low'`
  - `guildId` and `channelId` are blank in the JSON; likely required unless using webhook-only mode (but node shows `webhookId` set).
- **Inputs:** From **Merge Risk Result**
- **Edge cases / failures:**
  - If using bot credentials: missing guild/channel IDs will fail.
  - If using webhook-only: ensure node is configured accordingly in UI.
  - Message length limit on Discord (~2000 chars); many customers can exceed this.

#### Node: Code in JavaScript3
- **Type / role:** Code node — adds audit fields and sets notification status.
- **Configuration:**
  - Pulls all items from **Merge Risk Result** using `$items('Merge Risk Result')`.
  - For each item adds:
    - `notification_status: 'sent'`
    - `logged_at: new Date().toISOString()`
- **Outputs:** To **Append or update row in sheet3**
- **Edge cases:**
  - If node name changes, `$items('Merge Risk Result')` breaks.
  - If Merge Risk Result produces no items, sheet logging writes nothing.

#### Node: Append or update row in sheet3
- **Type / role:** Google Sheets node — log daily risk evaluation + notification status.
- **Configuration:**
  - Operation: Append or Update
  - Match column: `customer_id`
  - Writes: `customer_id, customer_name, email, product_name, payment_status, complaint_count, risk_level, reason, action_required, notification_status, logged_at`
  - DocumentId is a placeholder (`"your list value"`); sheet is `gid=0`.
- **Edge cases / failures:**
  - Incorrect Document ID / permissions.
  - Matching by `customer_id` requires consistent type formatting (string vs number).
  - Concurrency: multiple runs could race and overwrite updated rows.

---

### 2.3 Block: Weekly AI Feedback Generation & Weekly Reporting

**Overview:**  
Loads rows from Google Sheets, sends each row through an LLM prompt (Groq chat completions endpoint), merges the AI feedback back with original data, writes the enriched row back to Google Sheets, waits between batches, merges results, and emails a weekly HTML report including `ai_feedback`.

**Nodes involved:**
- Get row(s) in sheet1
- Loop Over Items1
- Prompt For Model1
- HTTP Request1
- Code in JavaScript
- Append or update row in sheet
- Wait1
- Merge1
- Send a message1

#### Node: Get row(s) in sheet1
- **Type / role:** Google Sheets node — reads rows to analyze.
- **Configuration:**
  - Operation implied: “Get row(s)”
  - Sheet: `gid=0`
  - Document: `your-document-id-here` (placeholder)
- **Outputs:** To **Loop Over Items1**
- **Edge cases:**
  - If sheet contains many rows, subsequent AI calls may be expensive/slow.
  - Missing/renamed columns can break prompt construction (expects fields like `product_name`, `reason`, etc.).

#### Node: Loop Over Items1
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates rows.
- **Configuration:** Default options; batch size not specified (n8n default is typically 1 unless set).
- **Connections:**
  - Output 0 → **Merge1** (collect original items path)
  - Output 1 → **Prompt For Model1** (AI processing path)
- **Behavioral note:** This is a common “loop + merge” pattern in n8n; correctness depends on Merge node mode and split node configuration.
- **Edge cases:**
  - Misconfigured batch size or merge mode can cause missing items or deadlocks.
  - If batch size > 1, `Prompt For Model1` code uses `$input.item` (single item) which is fine per item execution but consider how results merge.

#### Node: Prompt For Model1
- **Type / role:** Code node — builds LLM request body and preserves original row data.
- **Configuration highlights:**
  - Model: `llama-3.3-70b-versatile`
  - System instruction: senior product analyst; concise; “Do not sound like an AI”
  - User content embeds:
    - `product_name, customer_name, complaint_count, source (default support), reason`
  - Output includes:
    - `requestBody` (for HTTP Request)
    - `originalCustomerData` object preserving many fields (including `row_number`, `notification_status`, `logged_at`)
- **Outputs:** To **HTTP Request1**
- **Key variables/expressions:**
  - Uses `$input.item.json.<field>`
  - Sets `source` fallback: `$input.item.json.source || "support"`
- **Edge cases:**
  - If `reason` is empty/undefined, the model prompt will be weak; consider fallbacks.
  - Fields like `row_number` may not exist from Google Sheets unless included.

#### Node: HTTP Request1
- **Type / role:** HTTP Request — calls Groq OpenAI-compatible chat completions endpoint.
- **Configuration:**
  - POST `https://api.groq.com/openai/v1/chat/completions`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE` (should be replaced by credential/env var)
    - `Content-Type: application/json`
  - JSON body: `={{ $json.requestBody }}`
- **Outputs:** To **Code in JavaScript**
- **Edge cases / failures:**
  - Hardcoded token is insecure; use n8n credentials or environment variables.
  - Rate limits / 429 responses; timeouts; transient network failures.
  - Response shape assumed to be OpenAI-like: `choices[0].message.content`.

#### Node: Code in JavaScript
- **Type / role:** Code node — merges AI response back with the preserved customer data.
- **Configuration:**
  - `aiResponse = $input.item.json.choices[0].message.content`
  - `originalData = $('Prompt For Model1').first().json.originalCustomerData`
  - Outputs combined JSON with:
    - all `originalData` fields
    - `ai_feedback`
    - `feedback_generated_at` ISO timestamp
- **Outputs:** To **Append or update row in sheet**
- **Edge cases:**
  - If Groq returns an error payload, `choices[0]` will be undefined → runtime error.
  - `$('Prompt For Model1').first()` assumes correct pairing; in batch/parallel contexts it can mismatch. Safer: pass `originalCustomerData` through the HTTP request item itself (keep it in the same item) rather than re-referencing by node name.

#### Node: Append or update row in sheet
- **Type / role:** Google Sheets — writes AI-enriched data back to sheet.
- **Configuration:**
  - Operation: Append or Update
  - Matching column: `customer_id`
  - Writes: `customer_id, customer_name, email, product_name, payment_status, complaint_count, risk_level, reason, action_required, ai_feedback, feedback_generated_at`
  - Sheet: `value: 2092498842` (a gid-like numeric identifier)
  - Document: `your-document-id-here` placeholder
- **Outputs:** To **Wait1**
- **Edge cases:**
  - Matching on `customer_id` requires stable unique IDs; duplicates will overwrite.
  - Sheet ID mismatch vs read sheet (`gid=0`) suggests weekly branch may read one tab and write another tab—verify intended tabs.

#### Node: Wait1
- **Type / role:** Wait node — throttling / pacing between iterations.
- **Configuration:** No explicit duration shown; default Wait behavior requires configuration in UI (or uses webhook resume).
- **Outputs:** To **Loop Over Items1** (continues loop)
- **Edge cases:**
  - If Wait is not configured with a duration, executions may pause indefinitely.
  - If using “resume via webhook”, ensure webhook is triggered; otherwise the loop never completes.

#### Node: Merge1
- **Type / role:** Merge node — collects items for the weekly report email.
- **Configuration:** Not specified; used as join point from Loop Over Items1.
- **Outputs:** To **Send a message1**
- **Edge cases:** As with other Merge nodes, verify mode (“Append” usually).

#### Node: Send a message1
- **Type / role:** Gmail node — sends weekly HTML email including AI feedback.
- **Configuration:**
  - To: `user@example.com`
  - Subject: `=🤖 AI Product Feedback Analysis – {{ new Date().toLocaleDateString() }}`
  - HTML body:
    - Two tables:
      - High/Very High risk: `risk_level !== 'Low'`
      - Low risk: `risk_level === 'Low'`
    - Includes columns `ai_feedback` and `feedback_generated_at`
- **Inputs:** From **Merge1**
- **Edge cases:**
  - If AI feedback wasn’t generated (missing `ai_feedback`), the column will render empty.
  - Same size/HTML issues as daily email.

---

### 2.4 Sticky Notes (Documentation in-canvas)

These are non-executing nodes but convey intended structure.

- **Sticky Note4:** High-level explanation and setup steps.
- **Sticky Note5:** “Step 1: Triggers & Mode Selection”
- **Sticky Note6:** “Step2: Risk Evaluation & Escalation”
- **Sticky Note3:** “Step3: AI Feedback & Reporting”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Risk Check Trigger | Schedule Trigger | Starts daily run | — | Edit Fields3 | ## Step 1: Triggers & Mode Selection<br>Runs on daily and weekly schedules.<br>Determines whether the execution is a risk scan or a summary run. |
| Weekly schedule1 | Schedule Trigger | Starts weekly run | — | Edit Fields2 | ## Step 1: Triggers & Mode Selection<br>Runs on daily and weekly schedules.<br>Determines whether the execution is a risk scan or a summary run. |
| Edit Fields3 | Set | Sets `isDaily` flag | Daily Risk Check Trigger | Switch1 | ## Step 1: Triggers & Mode Selection<br>Runs on daily and weekly schedules.<br>Determines whether the execution is a risk scan or a summary run. |
| Edit Fields2 | Set | Sets `isWeekly` flag | Weekly schedule1 | Switch1 | ## Step 1: Triggers & Mode Selection<br>Runs on daily and weekly schedules.<br>Determines whether the execution is a risk scan or a summary run. |
| Switch1 | Switch | Routes to Daily vs Weekly branch | Edit Fields2, Edit Fields3 | Fetch Customer Risk Data; Get row(s) in sheet1 | ## Step 1: Triggers & Mode Selection<br>Runs on daily and weekly schedules.<br>Determines whether the execution is a risk scan or a summary run. |
| Fetch Customer Risk Data | Postgres | Fetches customer/payment/complaint aggregates | Switch1 (daily output) | Is High Risk Customer? | ## Step2: Risk Evaluation & Escalation<br>Fetches customer and payment data.<br>Classifies customers into low or high risk and prepares escalation records. |
| Is High Risk Customer? | IF | Classifies items (true=low risk as configured) | Fetch Customer Risk Data | Prepare Escalation Summary For Low Risk User; Prepare Escalation Summary For High Risk User | ## Step2: Risk Evaluation & Escalation<br>Fetches customer and payment data.<br>Classifies customers into low or high risk and prepares escalation records. |
| Prepare Escalation Summary For Low Risk User | Set | Formats Low risk record | Is High Risk Customer? (true) | Merge Risk Result | ## Step2: Risk Evaluation & Escalation<br>Fetches customer and payment data.<br>Classifies customers into low or high risk and prepares escalation records. |
| Prepare Escalation Summary For High Risk User | Set | Formats Very_High risk record | Is High Risk Customer? (false) | Merge Risk Result | ## Step2: Risk Evaluation & Escalation<br>Fetches customer and payment data.<br>Classifies customers into low or high risk and prepares escalation records. |
| Merge Risk Result | Merge | Recombines risk streams | Prepare Escalation Summary For Low Risk User; Prepare Escalation Summary For High Risk User | Send a message4; Send a message5 | ## Step2: Risk Evaluation & Escalation<br>Fetches customer and payment data.<br>Classifies customers into low or high risk and prepares escalation records. |
| Send a message4 | Gmail | Sends daily email report | Merge Risk Result | Code in JavaScript3 | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Send a message5 | Discord | Sends daily Discord report | Merge Risk Result | — | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Code in JavaScript3 | Code | Adds `notification_status` and `logged_at` | Send a message4 | Append or update row in sheet3 | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Append or update row in sheet3 | Google Sheets | Logs daily risk + notification status | Code in JavaScript3 | — | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Get row(s) in sheet1 | Google Sheets | Loads rows for weekly AI analysis | Switch1 (weekly output) | Loop Over Items1 | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Loop Over Items1 | Split In Batches | Iterates sheet rows | Get row(s) in sheet1; Wait1 | Merge1; Prompt For Model1 | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Prompt For Model1 | Code | Builds LLM prompt + preserves original row data | Loop Over Items1 | HTTP Request1 | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| HTTP Request1 | HTTP Request | Calls Groq chat completions API | Prompt For Model1 | Code in JavaScript | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Code in JavaScript | Code | Extracts AI response and merges into item | HTTP Request1 | Append or update row in sheet | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Append or update row in sheet | Google Sheets | Writes AI feedback back to sheet | Code in JavaScript | Wait1 | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Wait1 | Wait | Throttles/resumes loop | Append or update row in sheet | Loop Over Items1 | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Merge1 | Merge | Collects items for weekly email | Loop Over Items1 | Send a message1 | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Send a message1 | Gmail | Sends weekly email with AI feedback | Merge1 | — | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Sticky Note3 | Sticky Note | Canvas documentation | — | — | ## Step3: AI Feedback & Reporting<br>Generates product insights from customer issues.<br>Sends summaries and logs results for audits and analysis. |
| Sticky Note4 | Sticky Note | Canvas documentation | — | — | ## Customer Health Monitoring & Product Feedback Intelligence<br><br>### How it works<br>This workflow monitors customer health by combining payment behavior and customer feedback with AI-driven product intelligence. It runs automatically on a daily and weekly schedule to detect churn risk, classify customers by risk level, and convert raw customer issues into clear, executive-ready product insights.<br><br>Customer, payment, product, and feedback data are fetched from the database and evaluated using predefined rules. Customers are classified into Low, High, or Very High risk categories based on payment failures and issue frequency. High-risk customers are escalated immediately, while low-risk customers are logged for visibility without noise.<br><br>For customers with recurring issues, the workflow sends feedback data to an AI model that analyzes sentiment, identifies key problem themes, explains root causes, and generates actionable product improvement recommendations. These insights are stored alongside customer records for tracking and analysis.<br><br>Daily and weekly summaries are automatically sent to stakeholders via email and collaboration tools. All risk evaluations, feedback insights, and notification statuses are logged to Google Sheets, creating a reliable audit trail for leadership, support, and product teams.<br><br>### Setup steps<br>1. Configure daily and weekly Schedule Trigger nodes<br>2. Connect PostgreSQL to fetch customer, payment, and feedback data<br>3. Adjust risk rules in IF and Set nodes if needed<br>4. Add your AI API key to the HTTP Request node<br>5. Connect Gmail, Discord, and Google Sheets<br>6. Activate the workflow |
| Sticky Note5 | Sticky Note | Canvas documentation | — | — | ## Step 1: Triggers & Mode Selection<br>Runs on daily and weekly schedules.<br>Determines whether the execution is a risk scan or a summary run. |
| Sticky Note6 | Sticky Note | Canvas documentation | — | — | ## Step2: Risk Evaluation & Escalation<br>Fetches customer and payment data.<br>Classifies customers into low or high risk and prepares escalation records. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: *Customer Health Monitoring & Product Feedback Intelligence*
- Ensure workflow setting **Execution Order** is `v1` (as in JSON), unless you intentionally change it.

2) **Add triggers**
- Add **Schedule Trigger** named **Daily Risk Check Trigger**
  - Configure to run daily (set a clear rule in UI; do not leave an empty interval).
- Add **Schedule Trigger** named **Weekly schedule1**
  - Configure weekly: Monday, 09:00 (timezone as desired).

3) **Add mode flags**
- After Daily trigger, add **Set** node **Edit Fields3**
  - Add field `isDaily` (String) = `"true"`
- After Weekly trigger, add **Set** node **Edit Fields2**
  - Add field `isWeekly` (String) = `"true"`

4) **Add router**
- Add **Switch** node **Switch1**
  - Rule 1: `{{$json.isDaily}}` equals `"true"` → Output 0
  - Rule 2: `{{$json.isWeekly}}` equals `"true"` → Output 1
- Connect:
  - Daily trigger → Edit Fields3 → Switch1
  - Weekly trigger → Edit Fields2 → Switch1

### Daily branch (risk evaluation)

5) **PostgreSQL query**
- Add **Postgres** node **Fetch Customer Risk Data**
  - Credentials: configure Postgres host/db/user/password (or connection string).
  - Operation: *Execute Query*
  - Paste the SQL query from the analysis (customer/payment/products/complaints aggregation).
- Connect **Switch1 output 0** → **Fetch Customer Risk Data**

6) **Risk classification**
- Add **IF** node **Is High Risk Customer?**
  - Conditions (AND):
    - `payment_status` equals `success`
    - `complaint_count` equals `1` (make sure type matches; prefer numeric compare if possible)
- Connect **Fetch Customer Risk Data** → **Is High Risk Customer?**

7) **Format low risk**
- Add **Set** node **Prepare Escalation Summary For Low Risk User**
  - Map fields from input: `customer_id, customer_name, email, product_name, payment_status, complaint_count`
  - Set:
    - `risk_level = "Low"`
    - `reason = "Payment is successfull and the compliants are low"`
    - `action_required = "No"`
- Connect IF **true** → this Set node

8) **Format high risk**
- Add **Set** node **Prepare Escalation Summary For High Risk User**
  - Same base mappings
  - Set:
    - `risk_level = "Very_High"`
    - `reason = "Payment failure or multiple complaints detected"`
    - `action_required = "Immediate follow-up needed"`
- Connect IF **false** → this Set node

9) **Merge results**
- Add **Merge** node **Merge Risk Result**
  - Configure merge mode to *Append* (recommended for recombining branches).
- Connect:
  - Low risk Set → Merge input 0
  - High risk Set → Merge input 1

10) **Daily notifications**
- Add **Gmail** node **Send a message4**
  - Credentials: Gmail OAuth2 in n8n (grant `gmail.send`).
  - To: your recipient
  - Subject/body: use the provided HTML with `$items()` filters.
- Add **Discord** node **Send a message5**
  - Configure either:
    - Discord webhook (recommended), or
    - Bot token + guild/channel IDs
  - Paste the provided content expression.
- Connect **Merge Risk Result** → **Send a message4**
- Also connect **Merge Risk Result** → **Send a message5**

11) **Log notification status to Google Sheets**
- Add **Code** node **Code in JavaScript3**
  - Use code that pulls `$items('Merge Risk Result')` and adds `notification_status` + `logged_at`.
- Connect **Send a message4** → **Code in JavaScript3**
- Add **Google Sheets** node **Append or update row in sheet3**
  - Credentials: Google Sheets OAuth2/service account with edit access.
  - Operation: Append or Update
  - Document + Sheet: select your log sheet (tab)
  - Matching column: `customer_id`
  - Map columns as in JSON.
- Connect **Code in JavaScript3** → **Append or update row in sheet3**

### Weekly branch (AI feedback + weekly email)

12) **Read rows from Google Sheets**
- Add **Google Sheets** node **Get row(s) in sheet1**
  - Operation: Get Many / Read Rows (depending on UI wording)
  - Select Document and Sheet (gid=0 in the JSON)
- Connect **Switch1 output 1** → **Get row(s) in sheet1**

13) **Loop over rows**
- Add **Split In Batches** node **Loop Over Items1**
  - Set batch size explicitly (e.g., 1) to simplify pairing.
- Connect **Get row(s) in sheet1** → **Loop Over Items1**

14) **Create prompt payload**
- Add **Code** node **Prompt For Model1**
  - Build `requestBody` for chat completions
  - Preserve original row under `originalCustomerData`
- Connect **Loop Over Items1** → **Prompt For Model1**

15) **Call LLM endpoint**
- Add **HTTP Request** node **HTTP Request1**
  - Method: POST
  - URL: `https://api.groq.com/openai/v1/chat/completions`
  - Authentication: use a secure header:
    - Prefer `Authorization: Bearer {{$env.GROQ_API_KEY}}` or n8n credentials
  - Body: JSON = `{{$json.requestBody}}`
- Connect **Prompt For Model1** → **HTTP Request1**

16) **Merge AI response with original**
- Add **Code** node **Code in JavaScript**
  - Extract `choices[0].message.content`
  - Merge into preserved original data and add `feedback_generated_at`
- Connect **HTTP Request1** → **Code in JavaScript**

17) **Write AI feedback back to Google Sheets**
- Add **Google Sheets** node **Append or update row in sheet**
  - Operation: Append or Update
  - Matching column: `customer_id`
  - Map `ai_feedback` and `feedback_generated_at` plus other fields
  - Ensure the target tab is correct (JSON writes to a different gid-like value than it reads).
- Connect **Code in JavaScript** → **Append or update row in sheet**

18) **Throttle loop**
- Add **Wait** node **Wait1**
  - Configure a fixed wait time (e.g., 1–2 seconds) to respect rate limits, or remove if unnecessary.
- Connect **Append or update row in sheet** → **Wait1**
- Connect **Wait1** → **Loop Over Items1** (to continue the batch loop)

19) **Collect items for weekly email**
- Add **Merge** node **Merge1** (Append mode recommended)
- Connect **Loop Over Items1** (the other output used for collecting items) → **Merge1**

20) **Send weekly email**
- Add **Gmail** node **Send a message1**
  - Subject: `🤖 AI Product Feedback Analysis – {{ new Date().toLocaleDateString() }}`
  - Body: the provided HTML that includes `ai_feedback`
- Connect **Merge1** → **Send a message1**

21) **Activate workflow**
- Verify:
  - Postgres credentials and query output types
  - Google Sheet IDs/tabs for read vs write
  - Gmail + Discord credentials
  - Groq API key handling
  - Wait node duration to avoid stuck executions
- Activate.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow description, behavior explanation, and setup steps are embedded in the canvas. | Sticky Note: “Customer Health Monitoring & Product Feedback Intelligence” |
| The “Is High Risk Customer?” node currently flags “payment success + 1 complaint” as the TRUE branch, which is then labeled Low risk. Everything else becomes Very_High. | Consider renaming node or adjusting logic to match intended “High risk” semantics. |
| Daily notifications include both Gmail and Discord, but Discord config has empty `guildId`/`channelId` fields in JSON. | Verify whether you’re using webhook mode or bot mode and set required fields accordingly. |
| Weekly AI branch uses Groq’s OpenAI-compatible endpoint with a hardcoded Bearer token placeholder. | Replace with secure credential management (`$env` or n8n credential). |
| Weekly loop depends on Wait node configuration; an unconfigured Wait can pause runs indefinitely. | Configure duration-based wait for throttling. |