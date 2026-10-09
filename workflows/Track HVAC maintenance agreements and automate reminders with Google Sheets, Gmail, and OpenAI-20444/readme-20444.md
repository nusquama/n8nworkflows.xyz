Track HVAC maintenance agreements and automate reminders with Google Sheets, Gmail, and OpenAI

https://n8nworkflows.xyz/workflows/track-hvac-maintenance-agreements-and-automate-reminders-with-google-sheets--gmail--and-openai-20444


# Track HVAC maintenance agreements and automate reminders with Google Sheets, Gmail, and OpenAI

### 1. Workflow Overview

This workflow automates the administration, tracking, and customer communication lifecycle for HVAC maintenance agreements. It fulfills three primary operational functions: onboarding new agreements via webhook with automated AI welcome emails, performing daily checks to send service reminders and escalate overdue jobs to the office, and generating weekly AI-driven portfolio reviews for office management.

The architecture is divided into three functional blocks:
- **1.1 Intake & Welcome Processing:** Captures, normalizes, and validates inbound agreement webhook payloads. Valid records are passed to an AI model to draft a personalized welcome, logged into Google Sheets, and emailed to the customer. Invalid entries trigger an immediate office alert.
- **1.2 Daily Service Reminders:** Executes daily at 07:00 to query the tracker spreadsheet, evaluates service schedules against a 3-day threshold, generates customized AI reminders for due/overdue accounts, dispatches them, and aggregates unresolved overdues into an escalation notice for the office dispatcher.
- **1.3 Weekly Agreement Portfolio Review:** Executes every Monday at 09:00 to ingest the agreement book, compute active portfolio metrics (active counts, weekly due services, overdue counts, and 60-day renewal pipelines), and uses an AI assistant to summarize the book into an executive review email sent to the office manager.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Agreement Intake & Welcome

##### Overview
This block ingests raw JSON payloads representing newly signed HVAC maintenance agreements via an HTTP webhook. It normalizes variable fields, assigns default values (including a structured ID format if missing), validates required data points, drafts an AI-generated welcome email, commits the record to Google Sheets, and delivers the welcome message to the customer. If validation fails, an alert is emailed to the office instead.

##### Nodes Involved
- `When Agreement Added` (`n8n-nodes-base.webhook`)
- `Normalize New Agreement` (`n8n-nodes-base.set`)
- `Validate Agreement` (`n8n-nodes-base.if`)
- `Alert Office on Invalid Entry` (`n8n-nodes-base.gmail`)
- `Draft Welcome Email` (`@n8n/n8n-nodes-langchain.agent`)
- `Welcome Draft Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
- `Prepare Welcome Email` (`n8n-nodes-base.code`)
- `Add Agreement to Tracker` (`n8n-nodes-base.googleSheets`)
- `Email Welcome to Customer` (`n8n-nodes-base.gmail`)

##### Node Details

###### When Agreement Added
- **Type & Role:** Webhook trigger node. Listens for incoming HTTP POST requests containing new agreement data.
- **Configuration:** HTTP Method set to `POST`, path configured as `hvac-agreement-intake-0930`.
- **Key Expressions:** None.
- **Connections:** Input: None (Trigger); Output: `Normalize New Agreement`.
- **Edge Cases & Failures:** Incorrect URL endpoints, network timeouts, or missing payload bodies resulting in empty input data.

###### Normalize New Agreement
- **Type & Role:** Set node. Standardizes incoming payload properties, falling back to safe defaults and calculating missing auto-generated properties.
- **Configuration:** Maps JSON properties from `$json.body` or root properties into standardized schema properties (`agreement_id`, `customer_name`, `customer_email`, `customer_phone`, `address`, `system_type`, `filter_size`, `service_frequency_months`, `last_service_date`, `next_due_date`, `agreement_end_date`, `status`, `office_email`, `welcomed_at`).
- **Key Expressions:** 
  - `agreement_id`: `={{ $json.body.agreement_id || $json.agreement_id || ('AG-' + $now.toFormat('yyyyMMdd') + '-' + Math.floor(Math.random() * 900 + 100)) }}`
  - `service_frequency_months`: `={{ $json.body.service_frequency_months || $json.service_frequency_months || 3 }}`
  - `next_due_date`: `={{ $json.body.next_due_date || $json.next_due_date || $now.plus({months: 3}).toFormat('yyyy-MM-dd') }}`
- **Connections:** Input: `When Agreement Added`; Output: `Validate Agreement`.
- **Edge Cases & Failures:** Malformed date formats or invalid string types passed in the raw JSON payload.

###### Validate Agreement
- **Type & Role:** If node. Evaluates whether required business criteria exist before committing records to storage.
- **Configuration:** Combinator set to `and`. Checks four criteria for emptiness: `customer_name`, `customer_email`, `next_due_date`, and `office_email`.
- **Key Expressions:** Evaluates JSON fields for non-empty string state.
- **Connections:** Input: `Normalize New Agreement`; Output 1 (True): `Draft Welcome Email`; Output 2 (False): `Alert Office on Invalid Entry`.
- **Edge Cases & Failures:** Whitespace-only strings passing basic existence checks (though handled partially by loose validation settings).

###### Alert Office on Invalid Entry
- **Type & Role:** Gmail node. Sends a transactional error notification to internal office staff when agreement intake validation fails.
- **Configuration:** Sends plain text email via integrated Gmail OAuth2 credentials.
- **Key Expressions:** 
  - Send To: `={{ $json.office_email || 'user@example.com' }}`
  - Subject: `={{ 'New maintenance agreement rejected: missing details' }}`
  - Message: Joins array lines detailing missing parameters and raw posted values.
- **Connections:** Input: `Validate Agreement` (False branch); Output: None (Terminal node).
- **Credentials:** Gmail OAuth2 (`tzaBsDIL2rKaZdlq`).
- **Edge Cases & Failures:** Invalid office email property, expired or revoked OAuth2 credentials.

###### Draft Welcome Email
- **Type & Role:** Advanced AI Agent node. Utilizes an integrated language model to generate a custom customer welcome message based on agreement metadata.
- **Configuration:** Prompt type defined inline. System message enforces restrictions: under 130 words, plain paragraphs, strict JSON return structure with `subject` (max 10 words) and `body`.
- **Key Expressions:** 
  - Text prompt aggregates customer name, email, system type, filter size, frequency, next service date, and address.
- **Connections:** Input: `Validate Agreement` (True branch); Output: `Prepare Welcome Email`; AI Model Connection: `Welcome Draft Model`.
- **Edge Cases & Failures:** AI model returning markdown blocks or invalid JSON structure, handled downstream by parsing scripts.

###### Welcome Draft Model
- **Type & Role:** Openai Chat Model node. Provides the underlying LLM engine for the AI Agent.
- **Configuration:** Model selection set to `gpt-6-luna`.
- **Key Expressions:** None.
- **Connections:** Output (ai_languageModel): `Draft Welcome Email`.
- **Credentials:** OpenAI API (`JpMsLwC6dr7iGBJZ`).
- **Edge Cases & Failures:** API rate limits, upstream OpenAI service outages, or invalid API keys.

###### Prepare Welcome Email
- **Type & Role:** Code node (JavaScript). Parses raw AI text output, extracts JSON payloads safely, and merges them with the base agreement dataset.
- **Configuration:** Executes custom JavaScript iteration over input items.
- **Key Expressions:** `JSON.parse(raw)` wrapped in a try/catch fallback block; merges results with data from `Normalize New Agreement`.
- **Connections:** Input: `Draft Welcome Email`; Output: `Add Agreement to Tracker`.
- **Edge Cases & Failures:** Unrecoverable non-JSON text output from the LLM defaulting to empty parameters.

###### Add Agreement to Tracker
- **Type & Role:** Google Sheets node. Appends the validated and normalized agreement record into the specified spreadsheet tab.
- **Configuration:** Operation set to `append`. Mapping mode set to explicit column definitions (`agreement_id`, `customer_name`, `customer_email`, `customer_phone`, `address`, `system_type`, `filter_size`, `service_frequency_months`, `last_service_date`, `next_due_date`, `agreement_end_date`, `status`, `office_email`, `welcomed_at`, `last_reminded_at`, `reminder_count`). `reminder_count` explicitly initialized to `0`.
- **Key Expressions:** Maps fields from preceding node `$json` properties.
- **Connections:** Input: `Prepare Welcome Email`; Output: `Email Welcome to Customer`.
- **Credentials:** Google Sheets OAuth2 API (`CUYeeMWogKG05aan`).
- **Spreadsheet ID:** `1tnb6YJqodbsP1BIZ8GhlZl-6aIdvKxTYe7DuSmf19m4` (Tab: `Agreements`).
- **Edge Cases & Failures:** Sheet permission issues, quota limitations, or schema mismatches (renamed columns).

###### Email Welcome to Customer
- **Type & Role:** Gmail node. Sends the finalized welcome email to the newly onboarded customer.
- **Configuration:** Sends plain text email using parsed email subject and body attributes.
- **Key Expressions:** 
  - Send To: `={{ $('Prepare Welcome Email').first().json.customer_email }}`
  - Subject: `={{ $('Prepare Welcome Email').first().json.email_subject }}`
  - Message: `={{ $('Prepare Welcome Email').first().json.email_body }}`
- **Connections:** Input: `Add Agreement to Tracker`; Output: None (Terminal node for intake branch).
- **Credentials:** Gmail OAuth2 (`tzaBsDIL2rKaZdlq`).
- **Edge Cases & Failures:** Invalid customer email string, quota limits on sending domain.

---

#### 2.2 Block 2: Daily Service Reminders

##### Overview
This block runs automatically every morning at 07:00. It reads all rows from the Google Sheets agreement tracker, filters for active agreements where the next service date is due within 3 days or is already overdue, drafts a customized AI reminder for each matching customer, sends the customer email, and subsequently compiles a consolidated office alert listing any accounts that are past due.

##### Nodes Involved
- `Daily Service Check` (`n8n-nodes-base.scheduleTrigger`)
- `Read Agreements` (`n8n-nodes-base.googleSheets`)
- `Flag Due Agreements` (`n8n-nodes-base.code`)
- `Draft Reminder Email` (`@n8n/n8n-nodes-langchain.agent`)
- `Reminder Draft Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
- `Prepare Reminder Email` (`n8n-nodes-base.code`)
- `Email Reminder to Customer` (`n8n-nodes-base.gmail`)
- `Collate Overdue for Office` (`n8n-nodes-base.code`)
- `Email Overdue Alert to Office` (`n8n-nodes-base.gmail`)

##### Node Details

###### Daily Service Check
- **Type & Role:** Schedule Trigger node. Executes the workflow at a fixed cron schedule every morning.
- **Configuration:** Cron expression set to `0 7 * * *` (Daily at 07:00).
- **Key Expressions:** None.
- **Connections:** Input: None (Trigger); Output: `Read Agreements`.
- **Edge Cases & Failures:** Server timezone misconfigurations leading to premature or delayed executions.

###### Read Agreements
- **Type & Role:** Google Sheets node. Retrieves all records from the centralized maintenance tracking sheet.
- **Configuration:** Operation set to `read` (default list). Document ID and Sheet Name (`Agreements`) explicitly defined.
- **Key Expressions:** None.
- **Connections:** Input: `Daily Service Check`; Output: `Flag Due Agreements`.
- **Credentials:** Google Sheets OAuth2 API (`CUYeeMWogKG05aan`).
- **Spreadsheet ID:** `1tnb6YJqodbsP1BIZ8GhlZl-6aIdvKxTYe7DuSmf19m4` (Tab: `Agreements`).
- **Edge Cases & Failures:** Empty sheets returning zero items, API read rate limits.

###### Flag Due Agreements
- **Type & Role:** Code node (JavaScript). Filters agreement rows to isolate active accounts requiring service action.
- **Configuration:** JavaScript filtering logic calculating day differentials between current date (normalized to midnight) and `next_due_date`.
- **Key Expressions:** Checks if `status === 'active'` and `days_left <= 3`. Injects `days_left`, `is_overdue` ('yes' if `< 0`, otherwise 'no'), and default `office_email`.
- **Connections:** Input: `Read Agreements`; Output: `Draft Reminder Email`.
- **Edge Cases & Failures:** Unparseable date strings causing `NaN` calculations and silent record drops.

###### Draft Reminder Email
- **Type & Role:** Advanced AI Agent node. Generates a tailored reminder email for each due or overdue customer record passed from the filtering node.
- **Configuration:** System prompt instructs the agent to write a service reminder under 120 words, incorporating specific instructions if the account is overdue (negative days left). Returns JSON with `subject` and `body`.
- **Key Expressions:** Evaluates customer details, filter size, system type, and days until due.
- **Connections:** Input: `Flag Due Agreements`; Output: `Prepare Reminder Email`; AI Model Connection: `Reminder Draft Model`.
- **Edge Cases & Failures:** LLM response generation failures or syntax errors in multi-item loops.

###### Reminder Draft Model
- **Type & Role:** Openai Chat Model node. Powers the reminder generation agent.
- **Configuration:** Model selection set to `gpt-6-luna`.
- **Key Expressions:** None.
- **Connections:** Output (ai_languageModel): `Draft Reminder Email`.
- **Credentials:** OpenAI API (`JpMsLwC6dr7iGBJZ`).
- **Edge Cases & Failures:** API timeouts during high-volume batch processing of multiple due agreements.

###### Prepare Reminder Email
- **Type & Role:** Code node (JavaScript). Maps AI-generated response text back to corresponding customer metadata rows from the upstream flag node.
- **Configuration:** Index-based alignment loop matching item arrays between `Flag Due Agreements` and AI outputs.
- **Key Expressions:** Parses raw JSON from LLM output, falls back to raw text string if parsing fails, and binds properties to `email_subject` and `email_body`.
- **Connections:** Input: `Draft Reminder Email`; Output: `Email Reminder to Customer`.
- **Edge Cases & Failures:** Array length mismatches if upstream AI items drop or duplicate.

###### Email Reminder to Customer
- **Type & Role:** Gmail node. Iterates over processed items to send individual reminder emails to customers.
- **Configuration:** Sends plain text email using dynamic recipient and message attributes.
- **Key Expressions:** 
  - Send To: `={{ $json.customer_email }}`
  - Subject: `={{ $json.email_subject }}`
  - Message: `={{ $json.email_body }}`
- **Connections:** Input: `Prepare Reminder Email`; Output: `Collate Overdue for Office`.
- **Credentials:** Gmail OAuth2 (`tzaBsDIL2rKaZdlq`).
- **Edge Cases & Failures:** Invalid email addresses causing single-item execution failures within the batch.

###### Collate Overdue for Office
- **Type & Role:** Code node (JavaScript). Aggregates all overdue records into a single consolidated office escalation payload.
- **Configuration:** Filters flagged items where `is_overdue === 'yes'`. If no items are overdue, returns an empty array to halt execution paths.
- **Key Expressions:** Builds a formatted text body listing customer names, emails, due dates, calculated overdue days, and telephone numbers.
- **Connections:** Input: `Email Reminder to Customer`; Output: `Email Overdue Alert to Office` (conditional execution path).
- **Edge Cases & Failures:** Missing telephone numbers falling back safely to empty strings.

###### Email Overdue Alert to Office
- **Type & Role:** Gmail node. Delivers the aggregated overdue escalation summary to the office manager.
- **Configuration:** Sends plain text email containing overdue metrics.
- **Key Expressions:** 
  - Send To: `={{ $json.office_email || 'user@example.com' }}`
  - Subject: `={{ $json.overdue_subject }}`
  - Message: `={{ $json.overdue_body }}`
- **Connections:** Input: `Collate Overdue for Office`; Output: None (Terminal node).
- **Credentials:** Gmail OAuth2 (`tzaBsDIL2rKaZdlq`).
- **Edge Cases & Failures:** Execution skipped entirely when zero overdue agreements are detected.

---

#### 2.3 Block 3: Weekly Agreement Portfolio Review

##### Overview
This block triggers every Monday at 09:00 to perform a macro-level analysis of the HVAC maintenance agreement database. It reads all records, calculates portfolio statistics (active counts, services due within 7 days, overdue accounts, and upcoming renewals within 60 days), utilizes an AI agent to compose a structured operational review, and emails the briefing to the office manager.

##### Nodes Involved
- `Weekly Agreement Review` (`n8n-nodes-base.scheduleTrigger`)
- `Read Agreements for Review` (`n8n-nodes-base.googleSheets`)
- `Collate Agreement Stats` (`n8n-nodes-base.code`)
- `Draft Weekly Review` (`@n8n/n8n-nodes-langchain.agent`)
- `Weekly Review Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
- `Prepare Weekly Review` (`n8n-nodes-base.code`)
- `Email Weekly Review to Office` (`n8n-nodes-base.gmail`)

##### Node Details

###### Weekly Agreement Review
- **Type & Role:** Schedule Trigger node. Initiates the weekly workflow process on Mondays.
- **Configuration:** Cron expression set to `0 9 * * 1` (Every Monday at 09:00).
- **Key Expressions:** None.
- **Connections:** Input: None (Trigger); Output: `Read Agreements for Review`.
- **Edge Cases & Failures:** Server timezone offsets shifting execution day.

###### Read Agreements for Review
- **Type & Role:** Google Sheets node. Reads complete tracking data from the spreadsheet.
- **Configuration:** Operation set to read rows. Document ID and Sheet Name (`Agreements`) explicitly specified.
- **Key Expressions:** None.
- **Connections:** Input: `Weekly Agreement Review`; Output: `Collate Agreement Stats`.
- **Credentials:** Google Sheets OAuth2 API (`CUYeeMWogKG05aan`).
- **Spreadsheet ID:** `1tnb6YJqodbsP1BIZ8GhlZl-6aIdvKxTYe7DuSmf19m4` (Tab: `Agreements`).
- **Edge Cases & Failures:** Read limits or network timeouts.

###### Collate Agreement Stats
- **Type & Role:** Code node (JavaScript). Computes analytical metrics across active agreements.
- **Configuration:** JavaScript computations evaluating weekly service due windows (`<= 7 days`), overdue statuses (`< 0 days`), and upcoming agreement expirations (`0 to 60 days`).
- **Key Expressions:** Generates serialized JSON metadata strings (`due_week_json`, `overdue_json`, `renew_json`, `rows_json`) and localized date labels using `Asia/Jakarta` time zone standards.
- **Connections:** Input: `Read Agreements for Review`; Output: `Draft Weekly Review`.
- **Edge Cases & Failures:** Date parsing anomalies with non-standard date input strings.

###### Draft Weekly Review
- **Type & Role:** Advanced AI Agent node. Composes a concise weekly management narrative summarizing portfolio health based on aggregated metrics.
- **Configuration:** System prompt instructs the agent to write 4–5 sentences (active counts, services due this week, overdue accounts, and renewal metrics) without markdown, bullets, or em dashes, under 150 words. Returns JSON with `subject` and `body`.
- **Key Expressions:** Text prompt embeds week ending label, active counts, due lists, overdue arrays, and renewal arrays.
- **Connections:** Input: `Collate Agreement Stats`; Output: `Prepare Weekly Review`; AI Model Connection: `Weekly Review Model`.
- **Edge Cases & Failures:** Non-compliant response formatting from LLM.

###### Weekly Review Model
- **Type & Role:** Openai Chat Model node. Provides the underlying language model for the weekly review agent.
- **Configuration:** Model selection set to `gpt-6-luna`.
- **Key Expressions:** None.
- **Connections:** Output (ai_languageModel): `Draft Weekly Review`.
- **Credentials:** OpenAI API (`JpMsLwC6dr7iGBJZ`).
- **Edge Cases & Failures:** API timeouts or quota blockades.

###### Prepare Weekly Review
- **Type & Role:** Code node (JavaScript). Parses the AI-generated weekly review JSON response and merges it with portfolio metrics.
- **Configuration:** JavaScript error-handling block for JSON parsing with fallback strings.
- **Key Expressions:** Pulls base dataset from `Collate Agreement Stats` and assigns `email_subject` and `email_body`.
- **Connections:** Input: `Draft Weekly Review`; Output: `Email Weekly Review to Office`.
- **Edge Cases & Failures:** Malformed JSON output handling defaults to raw text blocks.

###### Email Weekly Review to Office
- **Type & Role:** Gmail node. Delivers the final weekly management report email to the office.
- **Configuration:** Sends plain text email using dynamic recipient and message attributes.
- **Key Expressions:** 
  - Send To: `={{ $('Collate Agreement Stats').first().json.office_email || 'user@example.com' }}`
  - Subject: `={{ $json.email_subject }}`
  - Message: `={{ $json.email_body }}`
- **Connections:** Input: `Prepare Weekly Review`; Output: None (Terminal node).
- **Credentials:** Gmail OAuth2 (`tzaBsDIL2rKaZdlq`).
- **Edge Cases & Failures:** Invalid office email destination or revoked OAuth credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation & Overview | None | None | track hvac maintenance agreements and remind customers when service is due with AI... |
| `1. Agreement intake and welcome` | `n8n-nodes-base.stickyNote` | Section Visual Container | None | None | 1. Agreement intake and welcome: A webhook collects a new maintenance agreement. Missing details are rejected by email. |
| `2. Daily service reminders` | `n8n-nodes-base.stickyNote` | Section Visual Container | None | None | 2. Daily service reminders: Every morning, agreements due for service are emailed a reminder. Overdue customers are then listed for the office. |
| `3. Weekly agreement review` | `n8n-nodes-base.stickyNote` | Section Visual Container | None | None | 3. Weekly agreement review: Each Monday, an AI review of active agreements and renewals is emailed to the office. |
| `When Agreement Added` | `n8n-nodes-base.webhook` | Webhook HTTP Endpoint | None | `Normalize New Agreement` | 1. Agreement intake and welcome |
| `Normalize New Agreement` | `n8n-nodes-base.set` | Data Normalization | `When Agreement Added` | `Validate Agreement` | 1. Agreement intake and welcome |
| `Validate Agreement` | `n8n-nodes-base.if` | Entry Verification | `Normalize New Agreement` | `Draft Welcome Email`, `Alert Office on Invalid Entry` | 1. Agreement intake and welcome |
| `Alert Office on Invalid Entry` | `n8n-nodes-base.gmail` | Rejection Notification | `Validate Agreement` | None | 1. Agreement intake and welcome |
| `Draft Welcome Email` | `@n8n/n8n-nodes-langchain.agent` | AI Content Generation | `Validate Agreement` | `Prepare Welcome Email` | 1. Agreement intake and welcome |
| `Welcome Draft Model` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider Engine | None | `Draft Welcome Email` | 1. Agreement intake and welcome |
| `Prepare Welcome Email` | `n8n-nodes-base.code` | Response Parser | `Draft Welcome Email` | `Add Agreement to Tracker` | 1. Agreement intake and welcome |
| `Add Agreement to Tracker` | `n8n-nodes-base.googleSheets` | Database Appender | `Prepare Welcome Email` | `Email Welcome to Customer` | 1. Agreement intake and welcome |
| `Email Welcome to Customer` | `n8n-nodes-base.gmail` | Customer Notification | `Add Agreement to Tracker` | None | 1. Agreement intake and welcome |
| `Daily Service Check` | `n8n-nodes-base.scheduleTrigger` | Cron Timer (07:00 Daily) | None | `Read Agreements` | 2. Daily service reminders |
| `Read Agreements` | `n8n-nodes-base.googleSheets` | Database Reader | `Daily Service Check` | `Flag Due Agreements` | 2. Daily service reminders |
| `Flag Due Agreements` | `n8n-nodes-base.code` | Schedule Evaluator | `Read Agreements` | `Draft Reminder Email` | 2. Daily service reminders |
| `Draft Reminder Email` | `@n8n/n8n-nodes-langchain.agent` | AI Reminder Drafter | `Flag Due Agreements` | `Prepare Reminder Email` | 2. Daily service reminders |
| `Reminder Draft Model` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider Engine | None | `Draft Reminder Email` | 2. Daily service reminders |
| `Prepare Reminder Email` | `n8n-nodes-base.code` | Response Parser | `Draft Reminder Email` | `Email Reminder to Customer` | 2. Daily service reminders |
| `Email Reminder to Customer` | `n8n-nodes-base.gmail` | Customer Notification | `Prepare Reminder Email` | `Collate Overdue for Office` | 2. Daily service reminders |
| `Collate Overdue for Office` | `n8n-nodes-base.code` | Escalation Aggregator | `Email Reminder to Customer` | `Email Overdue Alert to Office` | 2. Daily service reminders |
| `Email Overdue Alert to Office` | `n8n-nodes-base.gmail` | Office Escalation Notice | `Collate Overdue for Office` | None | 2. Daily service reminders |
| `Weekly Agreement Review` | `n8n-nodes-base.scheduleTrigger` | Cron Timer (09:00 Mon) | None | `Read Agreements for Review` | 3. Weekly agreement review |
| `Read Agreements for Review` | `n8n-nodes-base.googleSheets` | Database Reader | `Weekly Agreement Review` | `Collate Agreement Stats` | 3. Weekly agreement review |
| `Collate Agreement Stats` | `n8n-nodes-base.code` | Analytical Aggregator | `Read Agreements for Review` | `Draft Weekly Review` | 3. Weekly agreement review |
| `Draft Weekly Review` | `@n8n/n8n-nodes-langchain.agent` | AI Summary Drafter | `Collate Agreement Stats` | `Prepare Weekly Review` | 3. Weekly agreement review |
| `Weekly Review Model` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider Engine | None | `Draft Weekly Review` | 3. Weekly agreement review |
| `Prepare Weekly Review` | `n8n-nodes-base.code` | Response Parser | `Draft Weekly Review` | `Email Weekly Review to Office` | 3. Weekly agreement review |
| `Email Weekly Review to Office` | `n8n-nodes-base.gmail` | Management Reporting | `Prepare Weekly Review` | None | 3. Weekly agreement review |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in an n8n instance, execute the following configuration steps in sequence:

1. **Setup Google Sheets Tracker:** Create a spreadsheet named `HVAC Maintenance Agreements 0930` containing a single tab named `Agreements`. Set up the header row with the exact columns: `agreement_id`, `customer_name`, `customer_email`, `customer_phone`, `address`, `system_type`, `filter_size`, `service_frequency_months`, `last_service_date`, `next_due_date`, `agreement_end_date`, `status`, `office_email`, `welcomed_at`, `last_reminded_at`, and `reminder_count`.
2. **Configure Credentials:** Set up credentials for **Google Sheets OAuth2**, **Gmail OAuth2**, and an **OpenAI Chat Model** (compatible with models such as `gpt-6-luna` or equivalent).
3. **Build Block 1 (Intake & Welcome):**
   - Create a **Webhook** node (`When Agreement Added`). Set method to `POST` and path to `hvac-agreement-intake-0930`.
   - Create a **Set** node (`Normalize New Agreement`). Map incoming webhook fields (`agreement_id`, `customer_name`, `customer_email`, `customer_phone`, `address`, `system_type`, `filter_size`, `service_frequency_months`, `last_service_date`, `next_due_date`, `agreement_end_date`, `office_email`) with JavaScript fallback expressions (e.g., auto-generating ID strings with format `AG-yyyyMMdd-###` if missing). Set `status` to `active` and `welcomed_at` to `$now.toISO()`. Connect `When Agreement Added` -> `Normalize New Agreement`.
   - Create an **If** node (`Validate Agreement`). Configure conditions to ensure `customer_name`, `customer_email`, `next_due_date`, and `office_email` are not empty. Connect `Normalize New Agreement` -> `Validate Agreement`.
   - Create a **Gmail** node (`Alert Office on Invalid Entry`). Connect the `false` output of `Validate Agreement` here. Set recipient to `$json.office_email` and configure a plain-text rejection message.
   - Create an **Advanced AI Agent** node (`Draft Welcome Email`). Connect the `true` output of `Validate Agreement` here. Configure the prompt to build a welcoming email under 130 words returning JSON with keys `subject` and `body`.
   - Create an **OpenAI Chat Model** node (`Welcome Draft Model`). Set model to `gpt-6-luna` and link it to the AI agent's model input.
   - Create a **Code** node (`Prepare Welcome Email`). Add JavaScript to parse the LLM output safely using `JSON.parse` and merge it with normalized data. Connect `Draft Welcome Email` -> `Prepare Welcome Email`.
   - Create a **Google Sheets** node (`Add Agreement to Tracker`). Set operation to `append`, specify Spreadsheet ID `1tnb6YJqodbsP1BIZ8GhlZl-6aIdvKxTYe7DuSmf19m4`, tab `Agreements`, and map row values. Initialize `reminder_count` to `0`. Connect `Prepare Welcome Email` -> `Add Agreement to Tracker`.
   - Create a **Gmail** node (`Email Welcome to Customer`). Map recipient, subject, and body expressions referencing data from `Prepare Welcome Email`. Connect `Add Agreement to Tracker` -> `Email Welcome to Customer`.
4. **Build Block 2 (Daily Service Reminders):**
   - Create a **Schedule Trigger** node (`Daily Service Check`). Set cron expression to `0 7 * * *`.
   - Create a **Google Sheets** node (`Read Agreements`). Set operation to read all rows from tab `Agreements`. Connect `Daily Service Check` -> `Read Agreements`.
   - Create a **Code** node (`Flag Due Agreements`). Add JavaScript to filter active agreements where `days_left <= 3`. Attach `days_left` and `is_overdue` flags. Connect `Read Agreements` -> `Flag Due Agreements`.
   - Create an **Advanced AI Agent** node (`Draft Reminder Email`). Configure prompt restrictions for reminder emails under 120 words returning JSON keys `subject` and `body`. Connect `Flag Due Agreements` -> `Draft Reminder Email`.
   - Create an **OpenAI Chat Model** node (`Reminder Draft Model`). Set model to `gpt-6-luna` and link to the reminder agent.
   - Create a **Code** node (`Prepare Reminder Email`). Align AI outputs with flagged records index-by-index. Connect `Draft Reminder Email` -> `Prepare Reminder Email`.
   - Create a **Gmail** node (`Email Reminder to Customer`). Send the drafted reminder to `$json.customer_email`. Connect `Prepare Reminder Email` -> `Email Reminder to Customer`.
   - Create a **Code** node (`Collate Overdue for Office`). Filter overdue records and compile a text summary list. Connect `Email Reminder to Customer` -> `Collate Overdue for Office`.
   - Create a **Gmail** node (`Email Overdue Alert to Office`). Send the escalation notice to the office email if overdue items exist. Connect `Collate Overdue for Office` -> `Email Overdue Alert to Office`.
5. **Build Block 3 (Weekly Portfolio Review):**
   - Create a **Schedule Trigger** node (`Weekly Agreement Review`). Set cron expression to `0 9 * * 1` (Mondays at 09:00).
   - Create a **Google Sheets** node (`Read Agreements for Review`). Read all rows from tab `Agreements`. Connect `Weekly Agreement Review` -> `Read Agreements for Review`.
   - Create a **Code** node (`Collate Agreement Stats`). Calculate active counts, weekly due services, overdue accounts, and 60-day renewal pipelines. Connect `Read Agreements for Review` -> `Collate Agreement Stats`.
   - Create an **Advanced AI Agent** node (`Draft Weekly Review`). Set instructions for an executive summary under 150 words returning JSON keys `subject` and `body`. Connect `Collate Agreement Stats` -> `Draft Weekly Review`.
   - Create an **OpenAI Chat Model** node (`Weekly Review Model`). Set model to `gpt-6-luna` and link to the review agent.
   - Create a **Code** node (`Prepare Weekly Review`). Parse LLM JSON output and bind fields. Connect `Draft Weekly Review` -> `Prepare Weekly Review`.
   - Create a **Gmail** node (`Email Weekly Review to Office`). Deliver the management review report. Connect `Prepare Weekly Review` -> `Email Weekly Review to Office`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n Creator Platform | Built with [n8n](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Professional Consulting & Services | Project creator consultation link: [https://khmuhtadin.com/consultation/](https://khmuhtadin.com/consultation/) |