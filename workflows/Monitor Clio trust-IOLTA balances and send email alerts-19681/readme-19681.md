Monitor Clio trust/IOLTA balances and send email alerts

https://n8nworkflows.xyz/workflows/monitor-clio-trust-iolta-balances-and-send-email-alerts-19681


# Monitor Clio trust/IOLTA balances and send email alerts

### 1. Workflow Overview

This workflow automates the monitoring of trust and IOLTA account balances by fetching financial metrics from Clio, evaluating risk levels against predefined thresholds, managing alert cooldowns, and distributing notifications or escalations via SMTP. The workflow combines a daily operational alert loop with a weekly accounting rollup summary.

The logic is grouped into the following functional blocks:

- **1.1 Daily Input Reception & Data Fetching:** Triggers daily at 7:00 AM and queries the Clio API for all open matters containing trust balance parameters, client information, and responsible attorney metadata.
- **1.2 Daily Balance Processing & Tiering:** Unwraps the API payload, calculates risk tiers (`healthy`, `low`, or `negative`), applies a cooldown period to suppress duplicate alerts for unresolved balances, and filters for items requiring attention.
- **1.3 Trust Alert Preparation & Compliance:** Generates HTML-formatted notifications for low or negative balances and passes payloads through an external guardrail/compliance sub-workflow before evaluating dispatch approval.
- **1.4 Trust Alert Dispatch:** Sends the approved trust balance email via SMTP or routes skipped items through a no-operation node.
- **1.5 Negative Balance Escalation Process:** Evaluates filtered items for negative balances, builds urgent partner escalation emails, logs the events through an audit sub-workflow, and sends notifications via SMTP.
- **1.6 Weekly Trust Health Summary:** Triggers every Monday at 8:00 AM, fetches all open matter balances from Clio, aggregates portfolio-level health metrics, compiles a weekly rollup email, and sends it to accounting.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Input Reception & Data Fetching
This block initiates the daily monitoring sequence and retrieves current trust data from Clio.

- **Nodes Involved:** `When Daily at 7am`, `Fetch Open Matters With Trust Balance`
- **Node Details:**
  - `When Daily at 7am`
    - **Type:** `n8n-nodes-base.scheduleTrigger`
    - **Technical Role:** Time-based workflow entry point.
    - **Configuration:** Configured with a cron expression (`0 7 * * *`) to execute daily at 7:00 AM.
    - **Connections:** Output connects to `Fetch Open Matters With Trust Balance`.
    - **Edge Cases:** Missed executions if n8n is offline; catch-up logic depends on instance settings.
  - `Fetch Open Matters With Trust Balance`
    - **Type:** `n8n-nodes-base.httpRequest`
    - **Technical Role:** REST API client fetching matter records.
    - **Configuration:** Sends a `GET` request to `={{ $vars.CLIO_BASE_URL }}/api/v4/matters.json` with query parameters filtering by `status=open` and requesting specific fields (`id,display_number,description,trust_balance,client{name,email},responsible_attorney{id,name,email}`).
    - **Authentication:** OAuth2 (`Clio API (OAuth2)`).
    - **Connections:** Input from `When Daily at 7am`; output connects to `Split Matters Into Items`.
    - **Edge Cases:** Authentication token expiration, API rate limits, schema changes on the Clio API endpoint.

#### 2.2 Daily Balance Processing & Tiering
This block structures raw API payloads, applies business logic for thresholds and cooldowns, and filters out stable or recently alerted items.

- **Nodes Involved:** `Split Matters Into Items`, `Calculate Trust Balance Alert Tier`, `Filter Matters For Alerts`
- **Node Details:**
  - `Split Matters Into Items`
    - **Type:** `n8n-nodes-base.code`
    - **Technical Role:** Array unwrapper.
    - **Configuration:** Extracts items from the Clio response envelope (`$json.data`) into discrete workflow items.
    - **Connections:** Input from `Fetch Open Matters With Trust Balance`; output connects to `Calculate Trust Balance Alert Tier`.
    - **Edge Cases:** Empty array responses returning zero items.
  - `Calculate Trust Balance Alert Tier`
    - **Type:** `n8n-nodes-base.code`
    - **Technical Role:** Business logic evaluator for risk categorization and alert throttling.
    - **Configuration:** Reads variables `LOW_BALANCE_THRESHOLD` (default: 500) and `ALERT_COOLDOWN_DAYS` (default: 7). Uses workflow static data (`$getWorkflowStaticData('global')`) to track prior alert timestamps and tiers per matter ID. Worsening tiers (low to negative) bypass cooldown restrictions.
    - **Connections:** Input from `Split Matters Into Items`; output connects to `Filter Matters For Alerts`.
    - **Edge Cases:** Missing or non-numeric trust balance fields skipped safely via `Number.isNaN()` checks.
  - `Filter Matters For Alerts`
    - **Type:** `n8n-nodes-base.code`
    - **Technical Role:** Item filter.
    - **Configuration:** Evaluates the boolean `should_alert` flag set by the previous node, passing only actionable items downstream.
    - **Connections:** Input from `Calculate Trust Balance Alert Tier`; outputs connect to both `Build Alert Email For Trust Balance` and `If Trust Balance Is Negative`.

#### 2.3 Trust Alert Preparation & Compliance
This block compiles communication payloads for low/negative balances and validates them against a compliance guardrail sub-workflow.

- **Nodes Involved:** `Build Alert Email For Trust Balance`, `Run Compliance Workflow For Alert`, `If Alert Is Approved`
- **Node Details:**
  - `Build Alert Email For Trust Balance`
    - **Type:** `n8n-nodes-base.code`
    - **Technical Role:** Email content builder.
    - **Configuration:** Generates an HTML-formatted alert body and plaintext fallback using matter properties, applying dynamic accent colors based on alert severity. Assigns recipient using `FIRM_TRUST_ACCOUNTING_EMAIL` with a fallback to `FIRM_EMAIL`.
    - **Connections:** Input from `Filter Matters For Alerts`; output connects to `Run Compliance Workflow For Alert`.
  - `Run Compliance Workflow For Alert`
    - **Type:** `n8n-nodes-base.executeWorkflow`
    - **Technical Role:** Sub-workflow invoker for policy checks.
    - **Configuration:** Executes workflow identified by `{{ $vars.GUARDRAIL_WORKFLOW_ID }}` passing input payload containing `channel`, `message`, `matter_id`, `recipient`, and `template_id`.
    - **Sub-Workflow Reference:** Invokes the compliance guardrail workflow.
    - **Connections:** Input from `Build Alert Email For Trust Balance`; output connects to `If Alert Is Approved`.
  - `If Alert Is Approved`
    - **Type:** `n8n-nodes-base.if`
    - **Technical Role:** Conditional branch router.
    - **Configuration:** Evaluates whether `{{ $json.approved }}` equals `true`.
    - **Connections:** Input from `Run Compliance Workflow For Alert`; `true` branch connects to `Send Trust Balance Alert Email`, `false` branch connects to `Skip Alert For Opted-Out Recipient`.

#### 2.4 Trust Alert Dispatch
This block executes final email delivery for approved alerts or gracefully bypasses opted-out recipients.

- **Nodes Involved:** `Send Trust Balance Alert Email`, `Skip Alert For Opted-Out Recipient`
- **Node Details:**
  - `Send Trust Balance Alert Email`
    - **Type:** `n8n-nodes-base.emailSend`
    - **Technical Role:** SMTP email dispatcher.
    - **Configuration:** Uses expression bindings reaching back to `Build Alert Email For Trust Balance` via node naming (`$('Build Alert Email For Trust Balance').item.json`) to retrieve email subject, HTML body, plaintext body, and recipient. Uses `FIRM_FROM_EMAIL` as the sender address.
    - **Credentials:** SMTP (`Email account (SMTP)`).
    - **Connections:** Input from `If Alert Is Approved` (true branch); terminal node.
    - **Edge Cases:** SMTP authentication failures or network timeouts.
  - `Skip Alert For Opted-Out Recipient`
    - **Type:** `n8n-nodes-base.noOp`
    - **Technical Role:** Null operation terminal handler.
    - **Configuration:** Default configuration.
    - **Connections:** Input from `If Alert Is Approved` (false branch); terminal node.

#### 2.5 Negative Balance Escalation Process
This block isolates negative balance matters, compiles urgent partner communications, logs audit records, and dispatches management escalations.

- **Nodes Involved:** `If Trust Balance Is Negative`, `Build Partner Escalation Email`, `Run Audit Workflow For Escalation`, `Send Partner Escalation Email`, `Skip Partner Escalation`
- **Node Details:**
  - `If Trust Balance Is Negative`
    - **Type:** `n8n-nodes-base.if`
    - **Technical Role:** Severity filter.
    - **Configuration:** Checks if `{{ $json.alert_tier }}` equals `negative`.
    - **Connections:** Input from `Filter Matters For Alerts`; `true` branch connects to `Build Partner Escalation Email`, `false` branch connects to `Skip Partner Escalation`.
  - `Build Partner Escalation Email`
    - **Type:** `n8n-nodes-base.code`
    - **Technical Role:** Urgent email payload generator.
    - **Configuration:** Constructs a high-priority HTML and plaintext message targeted at managing partners. Assigns recipient via `FIRM_PARTNER_EMAIL` with fallback to `FIRM_EMAIL`.
    - **Connections:** Input from `If Trust Balance Is Negative` (true branch); output connects to `Run Audit Workflow For Escalation`.
  - `Run Audit Workflow For Escalation`
    - **Type:** `n8n-nodes-base.executeWorkflow`
    - **Technical Role:** Audit logging invocation.
    - **Configuration:** Executes workflow identified by `{{ $vars.GUARDRAIL_WORKFLOW_ID }}` with a partner escalation payload.
    - **Sub-Workflow Reference:** Invokes the compliance/audit guardrail workflow.
    - **Connections:** Input from `Build Partner Escalation Email`; output connects to `Send Partner Escalation Email`.
  - `Send Partner Escalation Email`
    - **Type:** `n8n-nodes-base.emailSend`
    - **Technical Role:** SMTP dispatcher for management alerts.
    - **Configuration:** Pulls parameters from `Build Partner Escalation Email` using node-name referencing (`$('Build Partner Escalation Email').item.json`). Uses `FIRM_FROM_EMAIL`.
    - **Credentials:** SMTP (`Email account (SMTP)`).
    - **Connections:** Input from `Run Audit Workflow For Escalation`; terminal node.
  - `Skip Partner Escalation`
    - **Type:** `n8n-nodes-base.noOp`
    - **Technical Role:** Null operation terminal handler for non-negative matters.
    - **Configuration:** Default configuration.
    - **Connections:** Input from `If Trust Balance Is Negative` (false branch); terminal node.

#### 2.6 Weekly Trust Health Summary
This block compiles portfolio-wide trust metrics on a weekly schedule and delivers a summary report to accounting.

- **Nodes Involved:** `When Weekly Monday at 8am`, `Fetch All Matters For Weekly Rollup`, `Calculate Weekly Trust Health Summary`, `Build Weekly Trust Health Email`, `Send Weekly Trust Health Email`
- **Node Details:**
  - `When Weekly Monday at 8am`
    - **Type:** `n8n-nodes-base.scheduleTrigger`
    - **Technical Role:** Weekly time-based workflow entry point.
    - **Configuration:** Configured with a cron expression (`0 8 * * 1`) executing every Monday at 8:00 AM.
    - **Connections:** Output connects to `Fetch All Matters For Weekly Rollup`.
  - `Fetch All Matters For Weekly Rollup`
    - **Type:** `n8n-nodes-base.httpRequest`
    - **Technical Role:** REST API client querying all open matters.
    - **Configuration:** Sends a `GET` request to `={{ $vars.CLIO_BASE_URL }}/api/v4/matters.json` with parameters `status=open` and fields limited to `id,display_number,trust_balance`.
    - **Authentication:** OAuth2 (`Clio API (OAuth2)`).
    - **Connections:** Input from `When Weekly Monday at 8am`; output connects to `Calculate Weekly Trust Health Summary`.
  - `Calculate Weekly Trust Health Summary`
    - **Type:** `n8n-nodes-base.code`
    - **Technical Role:** Statistical aggregation engine.
    - **Configuration:** Iterates over all open matters, categorizes balances into healthy, low, and negative counts, computes total negative exposure, and compiles arrays of negative matters.
    - **Connections:** Input from `Fetch All Matters For Weekly Rollup`; output connects to `Build Weekly Trust Health Email`.
  - `Build Weekly Trust Health Email`
    - **Type:** `n8n-nodes-base.code`
    - **Technical Role:** Summary email builder.
    - **Configuration:** Constructs a tabular HTML layout and text summary containing portfolio metrics. Assigns recipient using `FIRM_TRUST_ACCOUNTING_EMAIL` with fallback to `FIRM_EMAIL`.
    - **Connections:** Input from `Calculate Weekly Trust Health Summary`; output connects to `Send Weekly Trust Health Email`.
  - `Send Weekly Trust Health Email`
    - **Type:** `n8n-nodes-base.emailSend`
    - **Technical Role:** SMTP dispatcher for weekly reports.
    - **Configuration:** Dispatches HTML email using direct payload variables (`$json.email_html`, `$json.subject`, `$json.recipient`) sent from `FIRM_FROM_EMAIL`.
    - **Credentials:** SMTP (`Email account (SMTP)`).
    - **Connections:** Input from `Build Weekly Trust Health Email`; terminal node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `stickyNote` | Documentation container | None | None | Trust/IOLTA Balance Alerts<br><br>### How it works<br><br>1. The workflow runs daily and weekly scheduled triggers to check trust and IOLTA balances for open matters.<br>2. It fetches matter data from an external system and processes trust balances to determine alert tiers.<br>3. Alert-worthy trust balances trigger email alerts after compliance checks and potential partner escalations are logged and notified.<br>4. Negative balances cause escalations with separate email notifications for partners.<br>5. Weekly trust health rollup gathers summary data across all matters and sends accounting review emails.<br><br>### Setup steps<br><br>- [ ] Configure credentials for Clio API in the HTTP request nodes for data fetching.<br>- [ ] Set up email sending credentials for the emailSend nodes to deliver alerts.<br>- [ ] Ensure the sub-workflows invoked for compliance checks and escalation logging are present and properly configured.<br><br>### Customization<br><br>Customize alert thresholds and email content in the respective code nodes for tier computation and email building steps to suit organizational policies. |
| `Sticky Note1` | `stickyNote` | Documentation container | None | None | ## Daily trigger and data fetch<br><br>Daily scheduled trigger initiates the workflow and fetches open matters with trust balances from Clio API. |
| `Sticky Note2` | `stickyNote` | Documentation container | None | None | ## Trust balance processing<br><br>Processes fetched matter data to split records, compute trust balance alert tiers, and filter alert-worthy matters. |
| `Sticky Note3` | `stickyNote` | Documentation container | None | None | ## Trust alert preparation and compliance<br><br>Builds trust alert emails and runs compliance checks before final approval for sending alerts. |
| `Sticky Note4` | `stickyNote` | Documentation container | None | None | ## Send or skip trust alert email<br><br>Sends trust balance alert emails if approved or skips if recipients opted out. |
| `Sticky Note5` | `stickyNote` | Documentation container | None | None | ## Negative balance escalation process<br><br>Handles matters with negative trust balances by building escalation emails, logging for audit, and sending partner escalation emails or skipping escalation. |
| `Sticky Note6` | `stickyNote` | Documentation container | None | None | ## Weekly trigger and data fetch<br><br>Weekly scheduled trigger starts the trust health rollup process and fetches all matters for summary. |
| `Sticky Note7` | `stickyNote` | Documentation container | None | None | ## Weekly trust health summary and email<br><br>Computes weekly trust health summary, builds the summary email, and sends it to accounting for review. |
| `When Daily at 7am` | `scheduleTrigger` | Time-based trigger (daily) | None | `Fetch Open Matters With Trust Balance` | |
| `Fetch Open Matters With Trust Balance` | `httpRequest` | API client (Clio) | `When Daily at 7am` | `Split Matters Into Items` | |
| `Split Matters Into Items` | `code` | Data transformation | `Fetch Open Matters With Trust Balance` | `Calculate Trust Balance Alert Tier` | |
| `Calculate Trust Balance Alert Tier` | `code` | Risk evaluation & cooldown | `Split Matters Into Items` | `Filter Matters For Alerts` | |
| `Filter Matters For Alerts` | `code` | Item filtering | `Calculate Trust Balance Alert Tier` | `Build Alert Email For Trust Balance`, `If Trust Balance Is Negative` | |
| `Build Alert Email For Trust Balance` | `code` | Email template builder | `Filter Matters For Alerts` | `Run Compliance Workflow For Alert` | |
| `Run Compliance Workflow For Alert` | `executeWorkflow` | Sub-workflow integration | `Build Alert Email For Trust Balance` | `If Alert Is Approved` | |
| `If Alert Is Approved` | `if` | Conditional branching | `Run Compliance Workflow For Alert` | `Send Trust Balance Alert Email`, `Skip Alert For Opted-Out Recipient` | |
| `Send Trust Balance Alert Email` | `emailSend` | SMTP notification dispatch | `If Alert Is Approved` | None | |
| `Skip Alert For Opted-Out Recipient` | `noOp` | Null operation handler | `If Alert Is Approved` | None | |
| `If Trust Balance Is Negative` | `if` | Conditional branching | `Filter Matters For Alerts` | `Build Partner Escalation Email`, `Skip Partner Escalation` | |
| `Build Partner Escalation Email` | `code` | Urgent email builder | `If Trust Balance Is Negative` | `Run Audit Workflow For Escalation` | |
| `Run Audit Workflow For Escalation` | `executeWorkflow` | Audit logging integration | `Build Partner Escalation Email` | `Send Partner Escalation Email` | |
| `Send Partner Escalation Email` | `emailSend` | SMTP escalation dispatch | `Run Audit Workflow For Escalation` | None | |
| `Skip Partner Escalation` | `noOp` | Null operation handler | `If Trust Balance Is Negative` | None | |
| `When Weekly Monday at 8am` | `scheduleTrigger` | Time-based trigger (weekly) | None | `Fetch All Matters For Weekly Rollup` | |
| `Fetch All Matters For Weekly Rollup` | `httpRequest` | API client (Clio) | `When Weekly Monday at 8am` | `Calculate Weekly Trust Health Summary` | |
| `Calculate Weekly Trust Health Summary` | `code` | Statistical aggregation | `Fetch All Matters For Weekly Rollup` | `Build Weekly Trust Health Email` | |
| `Build Weekly Trust Health Email` | `code` | Report template builder | `Calculate Weekly Trust Health Summary` | `Send Weekly Trust Health Email` | |
| `Send Weekly Trust Health Email` | `emailSend` | SMTP summary dispatch | `Build Weekly Trust Health Email` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually reconstruct the workflow in n8n:

1. **Configure Environment Variables:**
   - Set `CLIO_BASE_URL` (e.g., `app.clio.com` or regional equivalent).
   - Set `FIRM_FROM_EMAIL` (sender address).
   - Set `FIRM_EMAIL` (general fallback recipient).
   - Set `FIRM_TRUST_ACCOUNTING_EMAIL` (primary accounting recipient).
   - Set `FIRM_PARTNER_EMAIL` (managing partner recipient).
   - Set `GUARDRAIL_WORKFLOW_ID` (target workflow ID for compliance/audit checks).
   - (Optional) Set `LOW_BALANCE_THRESHOLD` (default: `500`).
   - (Optional) Set `ALERT_COOLDOWN_DAYS` (default: `7`).

2. **Create Credentials:**
   - Configure an OAuth2 API credential named `Clio API (OAuth2)`.
   - Configure an SMTP credential named `Email account (SMTP)`.

3. **Build Branch 1: Daily Trust Monitoring & Alerts**
   - **Step 1:** Create a **Schedule Trigger** node named `When Daily at 7am`. Set cron expression to `0 7 * * *`.
   - **Step 2:** Create an **HTTP Request** node named `Fetch Open Matters With Trust Balance`. Set method to `GET`, URL to `=https://{{ $vars.CLIO_BASE_URL.replace('https://', '') }}/api/v4/matters.json`, query parameter `status` to `open`, and `fields` to `id,display_number,description,trust_balance,client{name,email},responsible_attorney{id,name,email}`. Assign the Clio OAuth2 credential. Connect `When Daily at 7am` output here.
   - **Step 3:** Create a **Code** node named `Split Matters Into Items`. Add JavaScript to unwrap `$json.data`. Connect input from `Fetch Open Matters With Trust Balance`.
   - **Step 4:** Create a **Code** node named `Calculate Trust Balance Alert Tier`. Add JavaScript to evaluate risk tiers against `LOW_BALANCE_THRESHOLD` and manage static data cooldowns. Connect input from `Split Matters Into Items`.
   - **Step 5:** Create a **Code** node named `Filter Matters For Alerts`. Add JavaScript to filter items where `should_alert` is true. Connect input from `Calculate Trust Balance Alert Tier`.

4. **Build Sub-Branch 1A: Accounting Notifications**
   - **Step 6:** Create a **Code** node named `Build Alert Email For Trust Balance`. Add JavaScript to generate HTML alert payloads and recipient assignment. Connect input from `Filter Matters For Alerts`.
   - **Step 7:** Create an **Execute Workflow** node named `Run Compliance Workflow For Alert`. Set Workflow ID to expression `={{ $vars.GUARDRAIL_WORKFLOW_ID }}` and define mapping inputs (`channel`, `message`, `matter_id`, `recipient`, `template_id`). Connect input from `Build Alert Email For Trust Balance`.
   - **Step 8:** Create an **If** node named `If Alert Is Approved`. Set condition for `{{ $json.approved }}` equals true. Connect input from `Run Compliance Workflow For Alert`.
   - **Step 9:** Create an **Email Send** node named `Send Trust Balance Alert Email`. Set parameters to reference `$('Build Alert Email For Trust Balance').item.json` properties. Assign SMTP credential. Connect input from `If Alert Is Approved` (true branch).
   - **Step 10:** Create a **No Operation** node named `Skip Alert For Opted-Out Recipient`. Connect input from `If Alert Is Approved` (false branch).

5. **Build Sub-Branch 1B: Partner Escalations**
   - **Step 11:** Create an **If** node named `If Trust Balance Is Negative`. Set condition for `{{ $json.alert_tier }}` equals `negative`. Connect input from `Filter Matters For Alerts`.
   - **Step 12:** Create a **Code** node named `Build Partner Escalation Email`. Add JavaScript to generate urgent partner notification markup. Connect input from `If Trust Balance Is Negative` (true branch).
   - **Step 13:** Create an **Execute Workflow** node named `Run Audit Workflow For Escalation`. Set Workflow ID to `={{ $vars.GUARDRAIL_WORKFLOW_ID }}` with audit mapping parameters. Connect input from `Build Partner Escalation Email`.
   - **Step 14:** Create an **Email Send** node named `Send Partner Escalation Email`. Set parameters to reference `$('Build Partner Escalation Email').item.json`. Assign SMTP credential. Connect input from `Run Audit Workflow For Escalation`.
   - **Step 15:** Create a **No Operation** node named `Skip Partner Escalation`. Connect input from `If Trust Balance Is Negative` (false branch).

6. **Build Branch 2: Weekly Rollup**
   - **Step 16:** Create a **Schedule Trigger** node named `When Weekly Monday at 8am`. Set cron expression to `0 8 * * 1`.
   - **Step 17:** Create an **HTTP Request** node named `Fetch All Matters For Weekly Rollup`. Set method to `GET`, URL to `=https://{{ $vars.CLIO_BASE_URL.replace('https://', '') }}/api/v4/matters.json`, query parameters `status=open` and `fields=id,display_number,trust_balance`. Assign Clio OAuth2 credential. Connect `When Weekly Monday at 8am` output here.
   - **Step 18:** Create a **Code** node named `Calculate Weekly Trust Health Summary`. Add JavaScript to aggregate balance metrics. Connect input from `Fetch All Matters For Weekly Rollup`.
   - **Step 19:** Create a **Code** node named `Build Weekly Trust Health Email`. Add JavaScript to compile summary HTML tables and recipient routing. Connect input from `Calculate Weekly Trust Health Summary`.
   - **Step 20:** Create an **Email Send** node named `Send Weekly Trust Health Email`. Set parameters to consume `$json` properties directly. Assign SMTP credential. Connect input from `Build Weekly Trust Health Email`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Clio API Integration | Ensure the `trust_balance` field is supported by the active version of the Clio REST API v4 implementation. |
| Sub-Workflow Dependency | The compliance and audit steps rely on an external guardrail sub-workflow mapped via `GUARDRAIL_WORKFLOW_ID`. |