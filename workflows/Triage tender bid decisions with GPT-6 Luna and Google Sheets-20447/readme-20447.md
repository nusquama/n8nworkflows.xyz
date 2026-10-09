Triage tender bid decisions with GPT-6 Luna and Google Sheets

https://n8nworkflows.xyz/workflows/triage-tender-bid-decisions-with-gpt-6-luna-and-google-sheets-20447


# Triage tender bid decisions with GPT-6 Luna and Google Sheets

### 1. Workflow Overview

This workflow automates the intake, evaluation, tracking, and notification process for incoming construction and engineering tenders. It processes submissions to determine their strategic fit, logs the results into Google Sheets, routes communications to the bid manager depending on the decision, maintains a weekly deadline digest, and includes error-handling mechanisms.

The workflow logic is grouped into the following functional blocks:
- **1.1 Intake & Normalization:** Receives incoming tender payloads via webhook, standardizes data fields, and generates unique tracking IDs and timestamps.
- **1.2 Input Validation:** Verifies that all mandatory tender fields are present before proceeding. Incomplete submissions trigger a notification email and halt processing.
- **1.3 AI Assessment & Parsing:** Evaluates valid tenders using an LLM agent configured as a procurement analyst to score fit, assess risks, and return standardized JSON.
- **1.4 Decision Routing & Logging:** Branches execution based on the AI recommendation (`no_bid` vs. `bid`/`needs_review`), logs the record to Google Sheets accordingly, and dispatches detailed assessment emails to the bid manager.
- **1.5 Weekly Digest Automation:** Executes on a Monday cron schedule to query tracked tenders, filter items due within a 14-day window, and email a consolidated digest.
- **1.6 Error Handling:** Captures global workflow failures and emails error payload details for troubleshooting.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Intake & Normalization
- **Overview:** Receives raw HTTP POST payloads containing tender details from an intake form or CRM and normalizes data formats while generating unique identifiers and timestamps.
- **Nodes Involved:** 
  - `When Tender Received`
  - `Normalize Tender`

##### Node Details:
- **When Tender Received**
  - *Type & Role:* `n8n-nodes-base.webhook` (Webhook trigger)
  - *Configuration:* Listens for HTTP POST requests on path `tender-intake-0925`.
  - *Inputs / Outputs:* Inputs: None (Entry point) | Outputs: Passes HTTP request body to `Normalize Tender`.
  - *Edge Cases / Failures:* Invalid JSON or dropped connections from external form submissions.
- **Normalize Tender**
  - *Type & Role:* `n8n-nodes-base.set` (Data transformation / Assignment)
  - *Configuration:* Assigns a unique identifier string (`tender_id`), ISO timestamp (`received_at`), and safely trims whitespace from string parameters (`tender_title`, `client_name`, `due_date`, `est_value`, `tender_summary`, `source_url`, and `bid_manager_email`). Falls back to `user@example.com` if no email is provided.
  - *Key Expressions:* 
    - `tender_id`: `={{ 'TD-' + $now.toFormat('yyyyLLdd-HHmmss') + '-' + String(Math.floor(Math.random() * 9000) + 1000) }}`
    - `received_at`: `={{ $now.toISO() }}`
  - *Inputs / Outputs:* Inputs: `When Tender Received` | Outputs: Passes normalized object to `Validate Tender Input`.
  - *Edge Cases / Failures:* Missing nested body properties handled via safe chaining (`($json.body || {})`).

---

#### Block 1.2: Input Validation
- **Overview:** Checks that mandatory parameters (`tender_title`, `due_date`, and `tender_summary`) exist before executing costly AI operations. Invalid submissions bypass the assessment and notify the bid manager.
- **Nodes Involved:**
  - `Validate Tender Input`
  - `Email Invalid Tender Notice`

##### Node Details:
- **Validate Tender Input**
  - *Type & Role:* `n8n-nodes-base.if` (Conditional router)
  - *Configuration:* Evaluates if `tender_title`, `due_date`, and `tender_summary` are not empty simultaneously.
  - *Inputs / Outputs:* Inputs: `Normalize Tender` | Outputs: True branch goes to `AI Assess Tender`; False branch goes to `Email Invalid Tender Notice`.
  - *Edge Cases / Failures:* Blank spaces incorrectly passing validation are mitigated by prior trimming in the normalization step.
- **Email Invalid Tender Notice**
  - *Type & Role:* `n8n-nodes-base.gmail` (Email notification)
  - *Configuration:* Sends a plain text notification email to the bid manager detailing missing required fields.
  - *Key Expressions:* Recipient evaluated via `={{ $json.bid_manager_email || 'user@example.com' }}`.
  - *Inputs / Outputs:* Inputs: `Validate Tender Input` (False branch) | Outputs: Terminates execution branch.
  - *Credentials:* Uses Gmail OAuth2 credentials (`tzaBsDIL2rKaZdlq`).
  - *Edge Cases / Failures:* Authentication revocation or invalid email formats.

---

#### Block 1.3: AI Assessment & Parsing
- **Overview:** Employs an AI agent running on a specified chat model to analyze the tender text, evaluate risks, and output structured evaluation parameters.
- **Nodes Involved:**
  - `AI Assess Tender`
  - `Tender Model`
  - `Parse Tender Assessment`

##### Node Details:
- **AI Assess Tender**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent)
  - *Configuration:* Configured with a procurement analyst system message instructing conservative evaluation, strict JSON output constraints without markdown, and specific data keys (`fit_score`, `recommendation`, `strengths`, `risks`, `key_requirements`, `deadline_risk`, `summary`).
  - *Inputs / Outputs:* Inputs: `Validate Tender Input` (True branch), `Tender Model` (Language model) | Outputs: Passes agent output to `Parse Tender Assessment`.
  - *Edge Cases / Failures:* LLM returning malformed responses or conversational text outside of strict JSON formatting.
- **Tender Model**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Chat Model sub-node)
  - *Configuration:* Uses model `gpt-6-luna`.
  - *Credentials:* OpenAI API credential (`JpMsLwC6dr7iGBJZ`).
- **Parse Tender Assessment**
  - *Type & Role:* `n8n-nodes-base.code` (JavaScript execution)
  - *Configuration:* Extracts JSON payload blocks using custom parsing logic (`grab` helper function), joins array fields with semicolons, enforces default bounds on scores and categorical risks, and sets status to `Tracking`.
  - *Inputs / Outputs:* Inputs: `AI Assess Tender` | Outputs: Passes structured output to `Route Bid Decision`.

---

#### Block 1.4: Decision Routing & Logging
- **Overview:** Evaluates the AI recommendation to separate standard/reviewable tenders from rejected ones, logs the entries into the Google Sheets tracker, and emails the corresponding verdict to the bid manager.
- **Nodes Involved:**
  - `Route Bid Decision`
  - `Log Tracked Tender`
  - `Email Tender Assessment`
  - `Log No Bid Tender`
  - `Email No Bid Note`

##### Node Details:
- **Route Bid Decision**
  - *Type & Role:* `n8n-nodes-base.if` (Conditional router)
  - *Configuration:* Checks whether `recommendation` is not equal to `no_bid`.
  - *Inputs / Outputs:* Inputs: `Parse Tender Assessment` | Outputs: True branch (bids / needs review) goes to `Log Tracked Tender`; False branch (`no_bid`) goes to `Log No Bid Tender`.
- **Log Tracked Tender**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Spreadsheet row appender)
  - *Configuration:* Appends row data to the `Tenders` sheet with status `Tracking`.
  - *Credentials:* Google Sheets OAuth2 API (`CUYeeMWogKG05aan`). Spreadsheet ID: `1wrxzTGAThEC8Y91N2gmuICyzIPTPyzd6TzPczUkDmRc`.
- **Email Tender Assessment**
  - *Type & Role:* `n8n-nodes-base.gmail` (Email notification)
  - *Configuration:* Sends a plain text summary containing score, recommendation, deadline risk, strengths, risks, and requirements to check.
  - *Inputs / Outputs:* Inputs: `Log Tracked Tender` | Outputs: Terminates branch.
- **Log No Bid Tender**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Spreadsheet row appender)
  - *Configuration:* Appends row data to the `Tenders` sheet with status `No Bid`.
  - *Credentials:* Google Sheets OAuth2 API (`CUYeeMWogKG05aan`). Spreadsheet ID: `1wrxzTGAThEC8Y91N2gmuICyzIPTPyzd6TzPczUkDmRc`.
- **Email No Bid Note**
  - *Type & Role:* `n8n-nodes-base.gmail` (Email notification)
  - *Configuration:* Emails the bid manager with the rationale for rejecting the tender.
  - *Inputs / Outputs:* Inputs: `Log No Bid Tender` | Outputs: Terminates branch.

---

#### Block 1.5: Weekly Digest Automation
- **Overview:** Periodically queries the tender tracker sheet every Monday morning, filters for items due within the next 14 days, and compiles a summary digest.
- **Nodes Involved:**
  - `Weekly Digest Trigger`
  - `Read Tracked Tenders`
  - `Keep Tracked Tenders`
  - `Build Digest Text`
  - `Email Weekly Digest`

##### Node Details:
- **Weekly Digest Trigger**
  - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Cron trigger)
  - *Configuration:* Scheduled via cron expression `0 8 * * 1` (Mondays at 08:00).
- **Read Tracked Tenders**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Spreadsheet reader)
  - *Configuration:* Reads all rows from the `Tenders` sheet.
  - *Credentials:* Google Sheets OAuth2 API (`CUYeeMWogKG05aan`). Spreadsheet ID: `1wrxzTGAThEC8Y91N2gmuICyzIPTPyzd6TzPczUkDmRc`.
- **Keep Tracked Tenders**
  - *Type & Role:* `n8n-nodes-base.if` (Conditional filter)
  - *Configuration:* Filters rows where `status` equals `Tracking`.
- **Build Digest Text**
  - *Type & Role:* `n8n-nodes-base.code` (JavaScript execution)
  - *Configuration:* Calculates date differences against current time, filters items due between 0 and 14 days, and compiles formatted text or a default notice if no tenders match.
- **Email Weekly Digest**
  - *Type & Role:* `n8n-nodes-base.gmail` (Email notification)
  - *Configuration:* Sends the compiled weekly summary to the bid manager.

---

#### Block 1.6: Error Handling
- **Overview:** Intercepts uncaught workflow execution errors and notifies administrators via email.
- **Nodes Involved:**
  - `Tender Error Trigger`
  - `Email Error Alert`

##### Node Details:
- **Tender Error Trigger**
  - *Type & Role:* `n8n-nodes-base.errorTrigger` (Error handling trigger)
  - *Configuration:* Listens for execution errors across the workflow.
- **Email Error Alert**
  - *Type & Role:* `n8n-nodes-base.gmail` (Email notification)
  - *Configuration:* Converts error payload to string, truncates to 2000 characters, and emails the designated administrator (`user@example.com`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Tender Received | n8n-nodes-base.webhook | Webhook trigger | None | Normalize Tender | triage tender documents and decide whether to bid with AI<br><br>### How it works<br><br>Construction and engineering firms lose days to tenders that are a bad fit. This workflow takes each incoming tender, checks the details, and gives the bid manager a clear read on whether it is worth pursuing.<br><br>1. A tender arrives by webhook (form, CRM, or pasted document summary). Required fields are checked, and incomplete submissions are flagged by email and dropped.<br>2. An AI agent reads the tender scope, scores fit from 0 to 100, and recommends bid, needs_review, or no_bid. It also lists risks, strengths, key requirements, and a deadline risk.<br>3. Tracked tenders are logged to Google Sheets. The bid manager gets an email with the score, the reasoning, and the mandatory requirements to check.<br>4. No-bid tenders are logged with a one line reason and a short email, so the team stays consistent about why the company passes.<br>5. Every Monday a digest lists the tenders being tracked that are due within 14 days, so deadlines never slip.<br><br>### Setup steps<br><br>- Connect Google Sheets for the Tenders log and Gmail for the assessment, no-bid, digest and error emails.<br>- Create a spreadsheet with a Tenders tab using the column headers from the append nodes, and set its id on the three Sheets nodes.<br>- Point the webhook at your intake form or paste tool. Set the bid manager inbox on the email nodes.<br>- Pick a chat model that returns JSON: the workflow works with OpenAI, Anthropic, or any OpenAI-compatible endpoint.<br><br>### Customization<br><br>- Change the fit threshold or the no-bid rule in Route Bid Decision.<br>- Change the digest window (default 14 days) in Build Digest Text.<br>- Swap email for WhatsApp or Slack if your bid team lives there. |
| Normalize Tender | n8n-nodes-base.set | Data transformation | When Tender Received | Validate Tender Input | ## 1. Intake and validation<br><br>A tender arrives by webhook. Required fields are checked. Invalid submissions get an email notice and are not processed. |
| Validate Tender Input | n8n-nodes-base.if | Conditional validation | Normalize Tender | AI Assess Tender, Email Invalid Tender Notice | ## 1. Intake and validation<br><br>A tender arrives by webhook. Required fields are checked. Invalid submissions get an email notice and are not processed. |
| Email Invalid Tender Notice | n8n-nodes-base.gmail | Email notification | Validate Tender Input | None | ## 1. Intake and validation<br><br>A tender arrives by webhook. Required fields are checked. Invalid submissions get an email notice and are not processed. |
| AI Assess Tender | @n8n/n8n-nodes-langchain.agent | AI Agent assessment | Validate Tender Input, Tender Model | Parse Tender Assessment | ## 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Tender Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM chat model provider | None | AI Assess Tender | ## 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Parse Tender Assessment | n8n-nodes-base.code | Code parsing and normalization | AI Assess Tender | Route Bid Decision | ## 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Route Bid Decision | n8n-nodes-base.if | Conditional routing | Parse Tender Assessment | Log Tracked Tender, Log No Bid Tender | ## 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Log No Bid Tender | n8n-nodes-base.googleSheets | Spreadsheet row appender | Route Bid Decision | Email No Bid Note | ## 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Log Tracked Tender | n8n-nodes-base.googleSheets | Spreadsheet row appender | Route Bid Decision | Email Tender Assessment | ## 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Email No Bid Note | n8n-nodes-base.gmail | Email notification | Log No Bid Tender | None | ## 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Email Tender Assessment | n8n-nodes-base.gmail | Email notification | Log Tracked Tender | None | ## 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Weekly Digest Trigger | n8n-nodes-base.scheduleTrigger | Cron schedule trigger | None | Read Tracked Tenders | ## 3. Weekly digest<br><br>Every Monday the workflow reads tracked tenders, keeps the ones due within 14 days, and emails one digest to the bid manager. |
| Read Tracked Tenders | n8n-nodes-base.googleSheets | Spreadsheet reader | Weekly Digest Trigger | Keep Tracked Tenders | ## 3. Weekly digest<br><br>Every Monday the workflow reads tracked tenders, keeps the ones due within 14 days, and emails one digest to the bid manager. |
| Keep Tracked Tenders | n8n-nodes-base.if | Conditional filter | Read Tracked Tenders | Build Digest Text | ## 3. Weekly digest<br><br>Every Monday the workflow reads tracked tenders, keeps the ones due within 14 days, and emails one digest to the bid manager. |
| Build Digest Text | n8n-nodes-base.code | Digest text compilation | Keep Tracked Tenders | Email Weekly Digest | ## 3. Weekly digest<br><br>Every Monday the workflow reads tracked tenders, keeps the ones due within 14 days, and emails one digest to the bid manager. |
| Email Weekly Digest | n8n-nodes-base.gmail | Email notification | Build Digest Text | None | ## 3. Weekly digest<br><br>Every Monday the workflow reads tracked tenders, keeps the ones due within 14 days, and emails one digest to the bid manager. |
| Tender Error Trigger | n8n-nodes-base.errorTrigger | Error handling trigger | None | Email Error Alert | |
| Email Error Alert | n8n-nodes-base.gmail | Error email notification | Tender Error Trigger | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheets Document:**
   - Set up a spreadsheet titled `Tender Intake Tracker 0925` with a worksheet tab named `Tenders`.
   - Add the following column headers in row 1: `tender_id`, `tender_title`, `client_name`, `est_value`, `due_date`, `fit_score`, `recommendation`, `status`, `strengths`, `risks`, `key_requirements`, `deadline_risk`, `received_at`.
   - Note the Spreadsheet ID from the URL.

2. **Configure Credentials:**
   - Connect and authorize **Google Sheets OAuth2** API credentials.
   - Connect and authorize **Gmail OAuth2** API credentials.
   - Connect an **OpenAI API** credential and ensure access to the chat model (`gpt-6-luna`).

3. **Build the Main Intake & Validation Branch:**
   - **Node 1 (`When Tender Received`):** Create a Webhook node. Set method to `POST` and path to `tender-intake-0925`.
   - **Node 2 (`Normalize Tender`):** Create a Set (Edit Fields) node. Connect input from `When Tender Received`. Map fields: `tender_id`, `received_at`, `tender_title`, `client_name`, `due_date`, `est_value`, `tender_summary`, `source_url`, and `bid_manager_email` using the string-trimming expressions detailed in Block 1.1.
   - **Node 3 (`Validate Tender Input`):** Create an IF node connected after `Normalize Tender`. Add conditions checking that `tender_title`, `due_date`, and `tender_summary` are not empty.
   - **Node 4 (`Email Invalid Tender Notice`):** Connect the false output of `Validate Tender Input` to a Gmail node configured to email the bid manager regarding missing required fields.

4. **Build the AI Assessment & Parsing Branch:**
   - **Node 5 (`AI Assess Tender`):** Create an AI Agent node connected to the true output of `Validate Tender Input`.
   - **Node 6 (`Tender Model`):** Create an OpenAI Chat Model sub-node, select the `gpt-6-luna` model, link your OpenAI credential, and connect it to the AI Agent.
   - **Node 7 (`Parse Tender Assessment`):** Create a Code node connected after the AI Agent. Paste the JavaScript snippet that extracts JSON, parses arrays, and structures output parameters.

5. **Build the Routing & Logging Branch:**
   - **Node 8 (`Route Bid Decision`):** Create an IF node connected after `Parse Tender Assessment` with a condition checking that `recommendation` does not equal `no_bid`.
   - **Node 9 (`Log Tracked Tender`):** Connect the true branch of `Route Bid Decision` to a Google Sheets append node targeting the `Tenders` tab using your document ID, mapping all incoming fields with `status` set to `Tracking`.
   - **Node 10 (`Email Tender Assessment`):** Connect a Gmail node after `Log Tracked Tender` to email the assessment summary.
   - **Node 11 (`Log No Bid Tender`):** Connect the false branch of `Route Bid Decision` to a Google Sheets append node mapping all fields with `status` set to `No Bid`.
   - **Node 12 (`Email No Bid Note`):** Connect a Gmail node after `Log No Bid Tender` to email the rejection rationale.

6. **Build the Weekly Digest Branch:**
   - **Node 13 (`Weekly Digest Trigger`):** Create a Schedule Trigger node set via cron expression `0 8 * * 1`.
   - **Node 14 (`Read Tracked Tenders`):** Create a Google Sheets read node targeting the `Tenders` tab.
   - **Node 15 (`Keep Tracked Tenders`):** Create an IF node checking if `status` equals `Tracking`.
   - **Node 16 (`Build Digest Text`):** Create a Code node to filter tenders due within 14 days and generate summary text.
   - **Node 17 (`Email Weekly Digest`):** Connect a Gmail node to send the compiled text to the bid manager.

7. **Build the Error Handling Branch:**
   - **Node 18 (`Tender Error Trigger`):** Create an Error Trigger node.
   - **Node 19 (`Email Error Alert`):** Connect a Gmail node to stringify and email the error payload (truncated to 2000 characters) to an administrative address.

8. **Activate Workflow:** Verify all credential bindings and toggle the workflow active.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n | [https://n8n.partnerlinks.io/creator-khmuhtadin](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Professional Business Assessment & Consulting | [https://khmuhtadin.com/consultation/](https://khmuhtadin.com/consultation/) |