Collect and escalate security patrol reports with gpt-6-luna, Google Sheets, and Gmail

https://n8nworkflows.xyz/workflows/collect-and-escalate-security-patrol-reports-with-gpt-6-luna--google-sheets--and-gmail-20438


# Collect and escalate security patrol reports with gpt-6-luna, Google Sheets, and Gmail

### 1. Workflow Overview

This workflow is designed to automate the collection, validation, AI-powered classification, logging, and escalation of security patrol and incident reports. It targets private security companies looking to streamline communication between guards in the field, operational supervisors, and site clients. 

The workflow logic is categorized into three functional blocks:
- **1.1 Report Intake and Escalation:** Receives reports from a web form, normalizes and validates fields, evaluates report severity using an AI model, logs the entry into Google Sheets, and triggers immediate multi-channel email alerts if the incident is urgent.
- **1.2 Daily Activity Report (DAR):** Automatically runs every day at 06:00 to aggregate the past 24 hours of security logs into a chronological digest and emails it to the operations manager, applying urgent flags in the subject line if necessary.
- **1.3 Monthly Site Review:** Runs on the first of every month at 08:00, aggregates the past 30 days of security operations, utilizes an AI model to draft an executive performance and review email, and delivers it to management.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Report Intake and Escalation

**Overview:**  
This block handles the entry point for security guards submitting logs or incident reports. It standardizes data formats, validates required inputs (rejecting invalid submissions via email), classifies the event severity via AI, records the data in a central ledger, and triggers immediate supervisor and client escalations for high-risk events.

**Nodes Involved:**
- Patrol Report Form
- Normalize Report
- Validate Report
- Email Officer Rejection
- Classify Report
- Classify Model
- Merge AI Summary
- Log Report to Sheets
- If Urgent
- Email Supervisor Alert
- Email Client Alert

**Node Details:**

- **Patrol Report Form**
  - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Webhook-based form trigger). Acts as the primary entry point, exposing a hosted form for mobile or desktop submission.
  - *Configuration Choices:* Configured with custom text fields (`officer_name`, `officer_email`, `site`, `shift`, `report_type`, `event_time`) and a textarea field (`description`). Required fields include officer name, officer email, site, and description.
  - *Key Expressions/Variables:* None (captures raw form payload).
  - *Connections:* Input: None (Trigger). Output: `Normalize Report`.
  - *Edge Cases/Failure Types:* Network drops or invalid form submissions. Unsubmitted inputs will not trigger the webhook.

- **Normalize Report**
  - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation and assignment).
  - *Configuration Choices:* Maps fields from diverse form-naming conventions into a clean, uniform schema. Generates operational metadata including a submission timestamp (`reported_at`) and a unique tracking ID (`report_id` formatted as `SEC-yyyyMMdd-###`). Injects default fallback emails for supervisors and clients (`user@example.com`).
  - *Key Expressions/Variables:* 
    - `{{ $json.officer_name || $json['Officer name'] || '' }}`
    - `{{ $now.toISO() }}`
    - `{{ 'SEC-' + $now.toFormat('yyyyMMdd') + '-' + (100 + Math.floor(Math.random() * 900)) }}`
  - *Connections:* Input: `Patrol Report Form`. Output: `Validate Report`.
  - *Edge Cases/Failure Types:* JavaScript engine runtime issues if date formatting libraries fail.

- **Validate Report**
  - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional branching).
  - *Configuration Choices:* Validates that critical parameters (`officer_name`, `officer_email`, `site`, `description`) are non-empty using loose type validation.
  - *Key Expressions/Variables:* Evaluates string non-empty states for `officer_name`, `officer_email`, `site`, and `description`.
  - *Connections:* Input: `Normalize Report`. Output (True): `Classify Report`. Output (False): `Email Officer Rejection`.
  - *Edge Cases/Failure Types:* Whitespace-only strings might pass unless pre-trimmed.

- **Email Officer Rejection**
  - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email notification).
  - *Configuration Choices:* Sends a plaintext notification to the submitting guard detailing missing parameters and posted values so they can correct and resubmit. Uses Gmail OAuth2 credentials.
  - *Key Expressions/Variables:* 
    - `sendTo`: `{{ $json.officer_email || 'user@example.com' }}`
    - `subject`: `Patrol report not saved: missing details`
  - *Connections:* Input: `Validate Report` (False branch). Output: None (Terminal node).
  - *Credentials Required:* Gmail OAuth2 (`Gmail Fresh Sep05`).
  - *Edge Cases/Failure Types:* Invalid recipient email addresses will throw an SMTP/Gmail API transmission error.

- **Classify Report**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent node).
  - *Configuration Choices:* Evaluates report data using strict system prompts to categorize severity into `urgent`, `normal`, or `informational`, generates a brief plain-text summary (<60 words), and supplies a suggested action for urgent tickets. Demands strict JSON output.
  - *Key Expressions/Variables:* 
    - `text`: `{{ 'Report: ' + $json.report_id + ' | Officer: ' + $json.officer_name + ' | Site: ' + $json.site + ' | Shift: ' + $json.shift + ' | Type: ' + $json.report_type + ' | Event time: ' + $json.event_time + ' | Description: ' + $json.description }}`
  - *Connections:* Input: `Validate Report` (True branch). Output: `Merge AI Summary`. Linked AI Model: `Classify Model`.
  - *Edge Cases/Failure Types:* Rate limits, API timeouts, or output format drift from the LLM.

- **Classify Model**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Chat Model provider).
  - *Configuration Choices:* Configured to use the model `gpt-6-luna`.
  - *Connections:* Output: Linked to `Classify Report` via `ai_languageModel` input.
  - *Credentials Required:* OpenAI API (`jonathan`).

- **Merge AI Summary**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution node).
  - *Configuration Choices:* Parses the raw string output from the AI agent into structured JSON. Falls back safely if the LLM output is malformed. Combines the AI outputs (`severity`, `summary`, `suggested_action`, `is_urgent`, `ai_raw`) with the initial normalized record.
  - *Key Expressions/Variables:* Iterates over `$input.all()` and cross-references data with `$('Normalize Report').first().json`.
  - *Connections:* Input: `Classify Report`. Output: `Log Report to Sheets`.

- **Log Report to Sheets**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet append operation).
  - *Configuration Choices:* Appends a new row to the worksheet named `Security Log` within the specified Google Sheets document. Maps all normalized fields, event notes, and classification data directly to sheet columns.
  - *Key Expressions/Variables:* Mapped expression parameters point to respective payload attributes (e.g., `={{ $json.site }}`, `={{ $json.summary }}`).
  - *Connections:* Input: `Merge AI Summary`. Output: `If Urgent`.
  - *Credentials Required:* Google Sheets OAuth2 API (`Feedback Analyzer Sheets`).
  - *Edge Cases/Failure Types:* Missing column headers in the target sheet or revoked API permissions will cause append operations to fail.

- **If Urgent**
  - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional evaluation).
  - *Configuration Choices:* Evaluates whether the report's `is_urgent` parameter equals `yes`.
  - *Key Expressions/Variables:* `={{ $json.is_urgent }}` equals `yes`.
  - *Connections:* Input: `Log Report to Sheets`. Output (True): `Email Supervisor Alert`. Output (False): None (Terminal).

- **Email Supervisor Alert**
  - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email notification).
  - *Configuration Choices:* Sends an urgent text-based alert to the operations supervisor containing full report details, operational flags, AI summary, and suggested mitigation steps.
  - *Key Expressions/Variables:* Uses references to upstream nodes (e.g., `$('Merge AI Summary').first().json.supervisor_email`) to assemble subject and message payloads.
  - *Connections:* Input: `If Urgent` (True branch). Output: `Email Client Alert`.
  - *Credentials Required:* Gmail OAuth2 (`Gmail Fresh Sep05`).

- **Email Client Alert**
  - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email notification).
  - *Configuration Choices:* Sends an urgent confirmation notice to the site client stating that an urgent incident was logged and that security supervisors are actively coordinating a response.
  - *Key Expressions/Variables:* Pulls recipient email and incident contexts from upstream node payloads.
  - *Connections:* Input: `Email Supervisor Alert`. Output: None (Terminal node).
  - *Credentials Required:* Gmail OAuth2 (`Gmail Fresh Sep05`).

---

#### Block 1.2: Daily Activity Report

**Overview:**  
This block triggers automatically once every day at 06:00, filters the security ledger for all events logged within the preceding 24-hour window, compiles a chronological operational summary, and emails the digest to the operations manager.

**Nodes Involved:**
- Daily Report Check
- Read Yesterday Logs
- Build DAR Digest
- Email Daily DAR

**Node Details:**

- **Daily Report Check**
  - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron scheduler).
  - *Configuration Choices:* Executes daily at 06:00 using the cron expression `0 6 * * *`.
  - *Connections:* Output: `Read Yesterday Logs`.

- **Read Yesterday Logs**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data retrieval).
  - *Configuration Choices:* Reads all rows from the `Security Log` worksheet.
  - *Connections:* Input: `Daily Report Check`. Output: `Build DAR Digest`.
  - *Credentials Required:* Google Sheets OAuth2 API (`Feedback Analyzer Sheets`).

- **Build DAR Digest**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript transformation).
  - *Configuration Choices:* Filters log records to strictly match timestamps falling within the last 24 hours. Formats timestamps for the `Asia/Pontianak` timezone, constructs bullet lines for every logged event, checks for any urgent occurrences, and establishes appropriate email subject flags.
  - *Key Expressions/Variables:* Processes incoming JSON arrays and evaluates cutoff time calculations (`now.getTime() - 24 * 3600 * 1000`).
  - *Connections:* Input: `Read Yesterday Logs`. Output: `Email Daily DAR`.

- **Email Daily DAR**
  - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email dispatch).
  - *Configuration Choices:* Delivers the finalized daily activity report payload to the supervisor's address.
  - *Key Expressions/Variables:* Uses `$json.supervisor_email`, `$json.dar_subject`, and `$json.dar_body`.
  - *Connections:* Input: `Build DAR Digest`. Output: None (Terminal node).
  - *Credentials Required:* Gmail OAuth2 (`Gmail Fresh Sep05`).

---

#### Block 1.3: Monthly Site Review

**Overview:**  
This block executes on the first day of every month, reviews all records from the previous 30 days, calculates operational KPIs (total events, incident breakdowns, site distribution), utilizes an AI agent to draft an executive review, and emails it to the operations team.

**Nodes Involved:**
- Monthly Review Check
- Read Month Logs
- Collate Month Stats
- Draft Monthly Review
- Monthly Review Model
- Prepare Monthly Review
- Email Monthly Review

**Node Details:**

- **Monthly Review Check**
  - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron scheduler).
  - *Configuration Choices:* Triggers on the 1st of every month at 08:00 (`0 8 1 * *`).
  - *Connections:* Output: `Read Month Logs`.

- **Read Month Logs**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data retrieval).
  - *Configuration Choices:* Reads all records from the `Security Log` worksheet.
  - *Connections:* Input: `Monthly Review Check`. Output: `Collate Month Stats`.
  - *Credentials Required:* Google Sheets OAuth2 API (`Feedback Analyzer Sheets`).

- **Collate Month Stats**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript data aggregator).
  - *Configuration Choices:* Filters log entries for a 30-day lookback window, counts total metrics, categorizes incidents and urgent flags, tallies activity levels per site, and bundles data into JSON structures for AI ingestion.
  - *Connections:* Input: `Read Month Logs`. Output: `Draft Monthly Review`.

- **Draft Monthly Review**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent node).
  - *Configuration Choices:* Uses system instructions to draft a concise, 3 to 5 sentence executive review email covering totals, incident summaries, and actionable recommendations based strictly on monthly stats. Requires output in valid JSON with `subject` and `body` keys.
  - *Key Expressions/Variables:* `{{ 'Month: ' + $json.month_label + ' | Total reports: ' + $json.total_reports + ' | Incidents: ' + $json.incident_count + ' | Urgent: ' + $json.urgent_count + ' | Reports by site: ' + $json.sites_json + ' | Items: ' + $json.rows_json }}`
  - *Connections:* Input: `Collate Month Stats`. Output: `Prepare Monthly Review`. Linked AI Model: `Monthly Review Model`.

- **Monthly Review Model**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Chat Model provider).
  - *Configuration Choices:* Uses model `gpt-6-luna`.
  - *Connections:* Output: Linked to `Draft Monthly Review` via `ai_languageModel` input.
  - *Credentials Required:* OpenAI API (`jonathan`).

- **Prepare Monthly Review**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution node).
  - *Configuration Choices:* Parses the AI-generated JSON response, ensuring safe fallbacks for subjects and bodies, and pairs them with distribution metadata.
  - *Connections:* Input: `Draft Monthly Review`. Output: `Email Monthly Review`.

- **Email Monthly Review**
  - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email dispatch).
  - *Configuration Choices:* Dispatches the final monthly performance review email to the operations supervisor.
  - *Key Expressions/Variables:* Uses `$('Prepare Monthly Review').first().json.supervisor_email`, `monthly_subject`, and `monthly_body`.
  - *Connections:* Input: `Prepare Monthly Review`. Output: None (Terminal node).
  - *Credentials Required:* Gmail OAuth2 (`Gmail Fresh Sep05`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Overview documentation and setup guide. | None | None | ## Collect security patrol reports and escalate urgent incidents... |
| 1. Report intake and escalation | `n8n-nodes-base.stickyNote` | Visual grouping block for intake logic. | None | None | ## 1. Report intake and escalation... |
| 2. Daily activity report | `n8n-nodes-base.stickyNote` | Visual grouping block for daily digest logic. | None | None | ## 2. Daily activity report... |
| 3. Monthly site review | `n8n-nodes-base.stickyNote` | Visual grouping block for monthly review logic. | None | None | ## 3. Monthly site review... |
| Patrol Report Form | `n8n-nodes-base.formTrigger` | Exposes form interface for mobile guard reporting. | None | Normalize Report | ## Collect security patrol reports and escalate urgent incidents... |
| Normalize Report | `n8n-nodes-base.set` | Normalizes incoming fields, timestamps, and IDs. | Patrol Report Form | Validate Report | ## 1. Report intake and escalation... |
| Validate Report | `n8n-nodes-base.if` | Validates required fields are populated. | Normalize Report | Classify Report, Email Officer Rejection | ## 1. Report intake and escalation... |
| Email Officer Rejection | `n8n-nodes-base.gmail` | Emails guard when mandatory details are missing. | Validate Report | None | ## 1. Report intake and escalation... |
| Classify Report | `@n8n/n8n-nodes-langchain.agent` | AI Agent evaluates incident severity and summarizes. | Validate Report | Merge AI Summary | ## 1. Report intake and escalation... |
| Classify Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM model provider (gpt-6-luna) for classification. | None | Classify Report | ## 1. Report intake and escalation... |
| Merge AI Summary | `n8n-nodes-base.code` | Parses AI output and combines it with report data. | Classify Report | Log Report to Sheets | ## 1. Report intake and escalation... |
| Log Report to Sheets | `n8n-nodes-base.googleSheets` | Appends complete structured report to Google Sheets. | Merge AI Summary | If Urgent | ## 1. Report intake and escalation... |
| If Urgent | `n8n-nodes-base.if` | Checks if report severity is marked urgent. | Log Report to Sheets | Email Supervisor Alert | ## 1. Report intake and escalation... |
| Email Supervisor Alert | `n8n-nodes-base.gmail` | Sends urgent escalation alert to the supervisor. | If Urgent | Email Client Alert | ## 1. Report intake and escalation... |
| Email Client Alert | `n8n-nodes-base.gmail` | Sends urgent incident notice to site client. | Email Supervisor Alert | None | ## 1. Report intake and escalation... |
| Daily Report Check | `n8n-nodes-base.scheduleTrigger` | Triggers daily at 06:00 for the activity digest. | None | Read Yesterday Logs | ## 2. Daily activity report... |
| Read Yesterday Logs | `n8n-nodes-base.googleSheets` | Fetches historical log records from Google Sheets. | Daily Report Check | Build DAR Digest | ## 2. Daily activity report... |
| Build DAR Digest | `n8n-nodes-base.code` | Filters past 24 hours of logs and formats digest. | Read Yesterday Logs | Email Daily DAR | ## 2. Daily activity report... |
| Email Daily DAR | `n8n-nodes-base.gmail` | Emails daily activity digest to management. | Build DAR Digest | None | ## 2. Daily activity report... |
| Monthly Review Check | `n8n-nodes-base.scheduleTrigger` | Triggers monthly on the 1st at 08:00. | None | Read Month Logs | ## 3. Monthly site review... |
| Read Month Logs | `n8n-nodes-base.googleSheets` | Fetches logs for the past 30 days. | Monthly Review Check | Collate Month Stats | ## 3. Monthly site review... |
| Collate Month Stats | `n8n-nodes-base.code` | Aggregates 30-day operational KPIs and site stats. | Read Month Logs | Draft Monthly Review | ## 3. Monthly site review... |
| Draft Monthly Review | `@n8n/n8n-nodes-langchain.agent` | AI Agent drafts monthly performance review email. | Collate Month Stats | Prepare Monthly Review | ## 3. Monthly site review... |
| Monthly Review Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM model provider (gpt-6-luna) for monthly review. | None | Draft Monthly Review | ## 3. Monthly site review... |
| Prepare Monthly Review | `n8n-nodes-base.code` | Parses AI monthly review payload into final fields. | Draft Monthly Review | Email Monthly Review | ## 3. Monthly site review... |
| Email Monthly Review | `n8n-nodes-base.gmail` | Emails monthly performance review to management. | Prepare Monthly Review | None | ## 3. Monthly site review... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Base Spreadsheet:** Create a Google Sheets file, add a worksheet tab named `Security Log`, and set up column headers matching: `report_id`, `reported_at`, `officer_name`, `officer_email`, `site`, `shift`, `report_type`, `event_time`, `description`, `severity`, `summary`, `suggested_action`, `is_urgent`, `supervisor_email`, and `client_email`.
2. **Setup Credentials:** Configure credentials for Google Sheets OAuth2, Gmail OAuth2, and an OpenAI API connection. Set timezone to `Asia/Pontianak`.
3. **Build Form Intake (Block 1.1):**
   - Add a **Patrol Report Form** trigger node. Configure form fields: `officer_name` (required), `officer_email` (required), `site` (required), `shift`, `report_type`, `event_time`, and `description` (textarea, required).
   - Add a **Normalize Report** (`Set`) node connected to the Form. Map input variables and generate dynamic identifiers (`report_id` using format `SEC-yyyyMMdd-###` and `reported_at` using `$now.toISO()`).
   - Add an **If** node named **Validate Report** to verify that `officer_name`, `officer_email`, `site`, and `description` are non-empty.
   - Connect the `false` branch of **Validate Report** to a **Gmail** node named **Email Officer Rejection** targeting `officer_email` with a rejection notification explaining missing fields.
4. **Configure AI Classification and Logging:**
   - Connect the `true` branch of **Validate Report** to an **AI Agent** node named **Classify Report**. Provide a system message instructing the model to classify reports into `urgent`, `normal`, or `informational`, write a <60-word summary, and supply a suggested action.
   - Attach an **OpenAI Chat Model** node (**Classify Model**) to the AI Agent, selecting model `gpt-6-luna`.
   - Connect **Classify Report** to a **Code** node named **Merge AI Summary** to parse the LLM JSON output and merge it with normalized fields.
   - Connect **Merge AI Summary** to a **Google Sheets** node named **Log Report to Sheets**, selecting operation `append`, targeting your spreadsheet ID and `Security Log` sheet name, mapping all attributes.
   - Connect **Log Report to Sheets** to an **If** node named **If Urgent** set to evaluate if `is_urgent` equals `yes`.
   - Connect the `true` branch of **If Urgent** to a **Gmail** node named **Email Supervisor Alert**, then connect that node to another **Gmail** node named **Email Client Alert** to notify the site client.
5. **Build Daily Activity Report (Block 1.2):**
   - Add a **Schedule Trigger** node named **Daily Report Check** with cron expression `0 6 * * *`.
   - Connect it to a **Google Sheets** node named **Read Yesterday Logs** to read rows from `Security Log`.
   - Connect to a **Code** node named **Build DAR Digest** to filter rows for the past 24 hours, format event lines, and determine urgent subject flags.
   - Connect to a **Gmail** node named **Email Daily DAR** to send the digest to the operations supervisor.
6. **Build Monthly Site Review (Block 1.3):**
   - Add a **Schedule Trigger** node named **Monthly Review Check** with cron expression `0 8 1 * *`.
   - Connect it to a **Google Sheets** node named **Read Month Logs** to read all worksheet rows.
   - Connect to a **Code** node named **Collate Month Stats** to filter a 30-day window, compute metrics, and structure JSON payloads.
   - Connect to an **AI Agent** node named **Draft Monthly Review** with an attached OpenAI Chat Model (**Monthly Review Model**, using `gpt-6-luna`) configured to output a structured JSON email review.
   - Connect to a **Code** node named **Prepare Monthly Review** to normalize the AI draft, and finally connect to a **Gmail** node named **Email Monthly Review** to deliver the executive report.
7. **Activation:** Verify all credential bindings, test form submissions, and activate the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n platform | [n8n Website](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Professional consulting and workflow design | [Consultation Booking](https://khmuhtadin.com/consultation/) |