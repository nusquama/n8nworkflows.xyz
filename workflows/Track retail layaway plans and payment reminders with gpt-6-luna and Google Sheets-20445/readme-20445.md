Track retail layaway plans and payment reminders with gpt-6-luna and Google Sheets

https://n8nworkflows.xyz/workflows/track-retail-layaway-plans-and-payment-reminders-with-gpt-6-luna-and-google-sheets-20445


# Track retail layaway plans and payment reminders with gpt-6-luna and Google Sheets

### 1. Workflow Overview

This workflow automates the tracking, reminder management, and reporting for retail layaway plans using n8n, Google Sheets, Gmail, and an AI language model (`gpt-6-luna`). Its primary purpose is to eliminate revenue leakage caused by forgotten payment schedules, automate proactive client communication, and provide store owners with structured monthly summaries.

The system is organized into three distinct logical blocks:

- **1.1 Input Reception & Plan Registration:** Ingests new layaway plan details via a webhook, normalizes the data, validates required parameters, logs the transaction into a Google Sheets tracker, and issues a confirmation email to the customer (or sends a rejection notice to the store office if data is missing).
- **1.2 Daily Payment Sweep & Escalation:** Executes every morning at 08:00 to evaluate all open layaway plans, calculate days remaining until payment due dates, categorize plans as overdue or due soon, log alerts, dispatch individual customer reminder emails, and compile a consolidated office escalation report.
- **1.3 Monthly AI Plan Review:** Runs on the 1st of every month at 09:00 to aggregate active plan metrics, evaluate financial exposure, and leverage an AI agent model (`gpt-6-luna`) to write a concise, human-readable review for store management.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Plan Registration
This block receives incoming layaway plan payloads, sanitizes and structures data fields (generating a unique plan reference and calculating financial balances and due dates), validates critical attributes, appends valid records to Google Sheets, and confirms the subscription schedule to the buyer.

- **When Layaway Plan Created**
  - *Type & Technical Role:* `n8n-nodes-base.webhook` (Trigger)
  - *Configuration:* Listens for HTTP POST requests on path `layaway-intake-0927`.
  - *Key Expressions:* None (receives raw JSON payload).
  - *Connections:* Input: None; Output: `Normalize Plan`.
  - *Edge Cases / Failure Types:* Webhook downtime, malformed JSON payloads, missing headers.

- **Normalize Plan**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
  - *Configuration:* Maps incoming properties, applies default fallbacks, and generates calculated fields.
  - *Key Expressions:* 
    - `plan_id`: `={{ 'LAY-' + $now.toFormat('yyyyLLdd') + '-' + String(Math.floor(Math.random() * 9000) + 1000) }}`
    - `balance_due`: `={{ (parseFloat(String(($json.body || {}).price_total || '0').trim()) || 0) - (parseFloat(String(($json.body || {}).deposit_paid || '0').trim()) || 0) }}`
    - `next_due_date`: `={{ $now.plus({ days: (parseInt(String(($json.body || {}).interval_days || '30').trim(), 10) || 30) }).toISODate() }}`
  - *Connections:* Input: `When Layaway Plan Created`; Output: `Validate Plan`.
  - *Edge Cases / Failure Types:* NaN conversion errors if price/deposit fields contain non-numeric characters.

- **Validate Plan**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Logic)
  - *Configuration:* Checks that `customer_name`, `item`, and `next_due_date` are not empty.
  - *Key Expressions:* Evaluates `={{ $json.customer_name }}`, `={{ $json.item }}`, and `={{ $json.next_due_date }}` against empty string checks.
  - *Connections:* Input: `Normalize Plan`; Outputs: True branch to `Log Plan`, False branch to `Email Reject Notice`.
  - *Edge Cases / Failure Types:* Evaluates whitespace-only strings as valid unless trimmed (handled in Normalize step).

- **Log Plan**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action)
  - *Configuration:* Appends validated row data into spreadsheet `Layaway Plan Tracker 0927`, sheet `Plans`.
  - *Key Expressions:* Maps schema fields (`plan_id`, `customer_name`, `price_total`, `balance_due`, etc.) to incoming JSON attributes.
  - *Connections:* Input: `Validate Plan`; Output: `Email Plan Confirmation`.
  - *Credentials:* Google Sheets OAuth2 API (`Feedback Analyzer Sheets`).
  - *Edge Cases / Failure Types:* API rate limits, schema mismatches, sheet permission revoked.

- **Email Plan Confirmation**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Action)
  - *Configuration:* Sends plain-text email confirmation to the customer.
  - *Key Expressions:* 
    - Recipient: `={{ $json.customer_email || $json.office_email }}`
    - Subject: `={{ 'Layaway plan confirmed for ' + $json.item }}`
  - *Connections:* Input: `Log Plan`; Output: None.
  - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
  - *Edge Cases / Failure Types:* Invalid recipient email formats, quota exhaustion.

- **Email Reject Notice**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Action)
  - *Configuration:* Notifies store staff of rejected plan submissions due to missing fields.
  - *Key Expressions:* Recipient: `={{ $json.office_email || "user@example.com" }}`
  - *Connections:* Input: `Validate Plan` (False branch); Output: None.
  - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
  - *Edge Cases / Failure Types:* Unreachable office email address.

---

#### 2.2 Daily Payment Sweep & Escalation
This block triggers every morning to evaluate active sheet records, compute relative due dates, isolate actionable reminders, notify customers, and compile escalation metrics for management.

- **Daily Payment Sweep**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
  - *Configuration:* Cron expression scheduled for daily execution at 08:00 (`0 8 * * *`).
  - *Key Expressions:* None.
  - *Connections:* Input: None; Output: `Read Plans`.
  - *Edge Cases / Failure Types:* Server time zone misalignment.

- **Read Plans**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action)
  - *Configuration:* Reads all rows from sheet `Plans` in document `Layaway Plan Tracker 0927`.
  - *Key Expressions:* None.
  - *Connections:* Input: `Daily Payment Sweep`; Output: `Compute Payment State`.
  - *Credentials:* Google Sheets OAuth2 API (`Feedback Analyzer Sheets`).

- **Compute Payment State**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Custom JavaScript snippet evaluating date differences (`days_until_due`), filtering completed/cancelled plans, and assigning staging properties (`overdue`, `due_soon`, `ok`).
  - *Key Expressions:* Processes item arrays programmatically via Javascript Date objects.
  - *Connections:* Input: `Read Plans`; Output: `If Reminder Needed`.

- **If Reminder Needed**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Logic)
  - *Configuration:* Checks if `needs_reminder` equals `"yes"`.
  - *Key Expressions:* `={{ $json.needs_reminder }}` == `"yes"`
  - *Connections:* Input: `Compute Payment State`; Outputs: True branch to `If Overdue`, False branch to None.

- **If Overdue**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Logic)
  - *Configuration:* Checks if `stage` equals `"overdue"`.
  - *Key Expressions:* `={{ $json.stage }}` == `"overdue"`
  - *Connections:* Input: `If Reminder Needed`; Outputs: True branch to `Prepare Overdue Pack`, False branch to `Prepare Reminder Pack`.

- **Prepare Overdue Pack**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Assembles messaging parameters, overdue counters, and alerting identifiers for past-due accounts.
  - *Connections:* Input: `If Overdue` (True branch); Output: `Log Overdue Alert`.

- **Log Overdue Alert**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action)
  - *Configuration:* Appends alert metadata into sheet `Alerts`.
  - *Connections:* Input: `Prepare Overdue Pack`; Output: `Email Overdue Reminder`.
  - *Credentials:* Google Sheets OAuth2 API (`Feedback Analyzer Sheets`).

- **Email Overdue Reminder**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Action)
  - *Configuration:* Emails individual customers about overdue accounts.
  - *Connections:* Input: `Log Overdue Alert`; Output: `Aggregate Office Escalations`.
  - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).

- **Aggregate Office Escalations**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Gathers all overdue records from the execution context and compiles a single summary body with timezone formatting (`Asia/Pontianak`).
  - *Connections:* Input: `Email Overdue Reminder`; Output: `Email Office Escalation`.

- **Email Office Escalation**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Action)
  - *Configuration:* Sends the aggregated overdue list to the store office.
  - *Key Expressions:* Subject: `={{ "Overdue layaway plans: " + $json.office_count }}`
  - *Connections:* Input: `Aggregate Office Escalations`; Output: None.
  - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).

- **Prepare Reminder Pack**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Prepares standard reminder templates for accounts falling within the 3-day window.
  - *Connections:* Input: `If Overdue` (False branch); Output: `Log Reminder Alert`.

- **Log Reminder Alert**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action)
  - *Configuration:* Appends reminder alert metadata to sheet `Alerts`.
  - *Connections:* Input: `Prepare Reminder Pack`; Output: `Email Payment Reminder`.
  - *Credentials:* Google Sheets OAuth2 API (`Feedback Analyzer Sheets`).

- **Email Payment Reminder**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Action)
  - *Configuration:* Dispatches the friendly reminder email to the customer.
  - *Connections:* Input: `Log Reminder Alert`; Output: None.
  - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).

---

#### 2.3 Monthly AI Plan Review
This block aggregates general portfolio statistics on the first of the month and utilizes an LLM agent to synthesize management recommendations.

- **Monthly Plan Review**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
  - *Configuration:* Cron expression scheduled for the 1st of every month at 09:00 (`0 9 1 * *`).
  - *Connections:* Input: None; Output: `Read Plans Monthly`.

- **Read Plans Monthly**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action)
  - *Configuration:* Reads all records from the `Plans` tab.
  - *Connections:* Input: `Monthly Plan Review`; Output: `Compute Month Stats`.
  - *Credentials:* Google Sheets OAuth2 API (`Feedback Analyzer Sheets`).

- **Compute Month Stats**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Reduces sheet rows into a JSON metrics summary including active counts, total balances, overdue totals, and top 8 overdue accounts.
  - *Connections:* Input: `Read Plans Monthly`; Output: `AI Write Review`.

- **AI Write Review**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI / LangChain Agent)
  - *Configuration:* Acts as assistant to the retail store owner. Uses a system prompt to draft a plain-text email summary under 180 words based on provided JSON metrics.
  - *Key Expressions:* `={{ "Monthly layaway review data: " + $json.stats_json }}`
  - *Connections:* Input: `Compute Month Stats`, Model linked via `Monthly Review Model`; Output: `Email Monthly Review`.

- **Monthly Review Model**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Sub-node / Language Model)
  - *Configuration:* Configured to use model `gpt-6-luna`.
  - *Credentials:* OpenAI API (`jonathan`).
  - *Connections:* Linked to `AI Write Review` via AI Language Model input connector.

- **Email Monthly Review**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Action)
  - *Configuration:* Sends the AI-generated monthly report to the store management.
  - *Key Expressions:* Message body: `={{ $json.output }}`
  - *Connections:* Input: `AI Write Review`; Output: None.
  - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Layaway Plan Created | n8n-nodes-base.webhook | Webhook intake trigger | None | Normalize Plan | track layaway plans and payment reminders for retail<br>New layaway plans |
| Normalize Plan | n8n-nodes-base.set | Builds IDs, dates, and balance variables | When Layaway Plan Created | Validate Plan | New layaway plans |
| Validate Plan | n8n-nodes-base.if | Validates mandatory fields | Normalize Plan | Log Plan, Email Reject Notice | New layaway plans |
| Log Plan | n8n-nodes-base.googleSheets | Appends plan to Plans sheet | Validate Plan | Email Plan Confirmation | New layaway plans |
| Email Plan Confirmation | n8n-nodes-base.gmail | Emails payment schedule to buyer | Log Plan | None | New layaway plans |
| Email Reject Notice | n8n-nodes-base.gmail | Notifies office of validation failures | Validate Plan | None | New layaway plans |
| Daily Payment Sweep | n8n-nodes-base.scheduleTrigger | Daily 08:00 cron trigger | None | Read Plans | Daily payment sweeps |
| Read Plans | n8n-nodes-base.googleSheets | Reads plan database | Daily Payment Sweep | Compute Payment State | Daily payment sweeps |
| Compute Payment State | n8n-nodes-base.code | Calculates days until due & status tags | Read Plans | If Reminder Needed | Daily payment sweeps |
| If Reminder Needed | n8n-nodes-base.if | Filters plans needing attention | Compute Payment State | If Overdue | Daily payment sweeps |
| If Overdue | n8n-nodes-base.if | Separates overdue from due-soon | If Reminder Needed | Prepare Overdue Pack, Prepare Reminder Pack | Daily payment sweeps |
| Prepare Overdue Pack | n8n-nodes-base.code | Formats overdue alert payloads | If Overdue | Log Overdue Alert | Daily payment sweeps |
| Log Overdue Alert | n8n-nodes-base.googleSheets | Logs overdue alert to Alerts sheet | Prepare Overdue Pack | Email Overdue Reminder | Daily payment sweeps |
| Email Overdue Reminder | n8n-nodes-base.gmail | Emails overdue notice to customer | Log Overdue Alert | Aggregate Office Escalations | Daily payment sweeps |
| Aggregate Office Escalations | n8n-nodes-base.code | Compiles run escalation list | Email Overdue Reminder | Email Office Escalation | Daily payment sweeps |
| Email Office Escalation | n8n-nodes-base.gmail | Emails overdue summary to store office | Aggregate Office Escalations | None | Daily payment sweeps |
| Prepare Reminder Pack | n8n-nodes-base.code | Formats upcoming reminder payloads | If Overdue | Log Reminder Alert | Daily payment sweeps |
| Log Reminder Alert | n8n-nodes-base.googleSheets | Logs reminder alert to Alerts sheet | Prepare Reminder Pack | Email Payment Reminder | Daily payment sweeps |
| Email Payment Reminder | n8n-nodes-base.gmail | Emails reminder to customer | Log Reminder Alert | None | Daily payment sweeps |
| Monthly Plan Review | n8n-nodes-base.scheduleTrigger | Monthly 1st at 09:00 cron trigger | None | Read Plans Monthly | Monthly plan review |
| Read Plans Monthly | n8n-nodes-base.googleSheets | Reads plan database for reporting | Monthly Plan Review | Compute Month Stats | Monthly plan review |
| Compute Month Stats | n8n-nodes-base.code | Reduces rows to analytics JSON object | Read Plans Monthly | AI Write Review | Monthly plan review |
| AI Write Review | @n8n/n8n-nodes-langchain.agent | AI agent drafting monthly owner summary | Compute Month Stats, Monthly Review Model | Email Monthly Review | Monthly plan review |
| Email Monthly Review | n8n-nodes-base.gmail | Emails monthly review to store | AI Write Review | None | Monthly plan review |
| Monthly Review Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-6-Luna LLM engine configuration | None | AI Write Review | Monthly plan review |

---

### 4. Reproducing the Workflow from Scratch

1. **Setup Google Sheets:** Create a Google Drive spreadsheet named `Layaway Plan Tracker 0927` with two tabs:
   - Tab 1 (`Plans`): Columns: `plan_id`, `customer_name`, `customer_email`, `customer_phone`, `item`, `price_total`, `deposit_paid`, `balance_due`, `installment_amount`, `installments_total`, `interval_days`, `next_due_date`, `payments_made`, `status`, `created_at`.
   - Tab 2 (`Alerts`): Columns: `alert_id`, `plan_id`, `customer_name`, `item`, `alert_type`, `sent_to`, `sent_at`, `customer_email`, `office_email`.
2. **Configure Credentials:** Set up OAuth2 credentials for **Google Sheets** and **Gmail**, and set up an **OpenAI API** key credential.
3. **Build Flow 1 (Intake):**
   - Create a **Webhook** node (`When Layaway Plan Created`), set HTTP Method to `POST`, and path to `layaway-intake-0927`.
   - Add a **Set** node (`Normalize Plan`), configuring JavaScript expressions to generate unique IDs, parse prices/deposits safely, and calculate balances and due dates.
   - Add an **If** node (`Validate Plan`) evaluating non-empty conditions for `customer_name`, `item`, and `next_due_date`.
   - Connect the True branch to a **Google Sheets** node (`Log Plan`), mapping values to the `Plans` tab and selecting append operation. Connect its output to a **Gmail** node (`Email Plan Confirmation`).
   - Connect the False branch of `Validate Plan` to a **Gmail** node (`Email Reject Notice`).
4. **Build Flow 2 (Daily Sweep):**
   - Create a **Schedule Trigger** node (`Daily Payment Sweep`) with cron expression `0 8 * * *`.
   - Connect to a **Google Sheets** node (`Read Plans`) reading the `Plans` tab.
   - Add a **Code** node (`Compute Payment State`) implementing date difference calculations and staging logic.
   - Add an **If** node (`If Reminder Needed`) checking `needs_reminder == 'yes'`.
   - Add an **If** node (`If Overdue`) checking `stage == 'overdue'`.
   - *Overdue Branch:* Connect to a **Code** node (`Prepare Overdue Pack`), then a **Google Sheets** node (`Log Overdue Alert`) appending to `Alerts`, then a **Gmail** node (`Email Overdue Reminder`), then a **Code** node (`Aggregate Office Escalations`), and finally a **Gmail** node (`Email Office Escalation`).
   - *Reminder Branch:* Connect to a **Code** node (`Prepare Reminder Pack`), then a **Google Sheets** node (`Log Reminder Alert`) appending to `Alerts`, and finally a **Gmail** node (`Email Payment Reminder`).
5. **Build Flow 3 (Monthly Review):**
   - Create a **Schedule Trigger** node (`Monthly Plan Review`) with cron expression `0 9 1 * *`.
   - Connect to a **Google Sheets** node (`Read Plans Monthly`) reading the `Plans` tab.
   - Add a **Code** node (`Compute Month Stats`) to aggregate portfolio metrics into JSON format.
   - Add an **AI Agent** node (`AI Write Review`), configuring its system prompt to restrict word count and formatting.
   - Attach an **OpenAI Chat Model** node (`Monthly Review Model`) configured with model `gpt-6-luna` to the AI Agent's model input.
   - Connect the AI Agent output to a **Gmail** node (`Email Monthly Review`).
6. **Activate:** Verify all timezone settings (configured to `Asia/Pontianak`), select the appropriate credential instances on integration nodes, and activate the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n | [n8n Creator Partner Link](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Business assessment and consulting | [Consultation Request Link](https://khmuhtadin.com/consultation/) |