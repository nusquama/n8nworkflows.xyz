Generate pool service visit reports with Google Sheets, Gemini, Gmail, Slack, and Telegram

https://n8nworkflows.xyz/workflows/generate-pool-service-visit-reports-with-google-sheets--gemini--gmail--slack--and-telegram-20429


# Generate pool service visit reports with Google Sheets, Gemini, Gmail, Slack, and Telegram

### 1. Workflow Overview

This workflow automates the collection, processing, and management of swimming pool service visit reports. It bridges field data gathered by technicians with back-office tracking systems, AI-driven assessment tools, and communication channels. 

Target use cases include pool maintenance companies seeking to eliminate manual paperwork, standardize chemical and equipment safety checks, automatically notify customers of service results, and maintain robust audit trails in Google Sheets.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Customer Validation:** Captures technician form submissions and validates customer records against Google Sheets.
- **1.2 Context Preparation & AI Structuring:** Gathers recent visit history, evaluates chemical and physical readings against predefined thresholds, and uses Google Gemini to parse notes into structured operational data.
- **1.3 AI Failure Handling (Fallback):** Catches parsing errors, logs fallback records to Google Sheets, and alerts staff via Slack.
- **1.4 Visit Logging & Work Order Management:** Persists structured visit records, evaluates follow-up criteria, updates customer files, and creates work orders when necessary.
- **1.5 Urgent Alerting & Manager Approvals:** Triggers immediate Slack alerts for critical issues and manages approval cycles via Gmail for customer-facing reports.
- **1.6 Scheduled Operations & Error Monitoring:** Runs a daily morning digest of open work orders and captures unexpected workflow execution errors via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Customer Validation
This block initiates the primary execution path when a technician submits a pool visit form, verifying that the target customer exists in the system.
- **Nodes Involved:** `When Technician Submits Form`, `Fetch Customer Details`, `If Customer Found`, `Send Unknown Customer Alert`
- **Node Details:**
  - `When Technician Submits Form`: 
    - Type: `n8n-nodes-base.formTrigger` (v2.2)
    - Role: Webhook-based form endpoint capturing customer ID, technician name, visit date, chemical levels, pressure readings, and freeform notes.
    - Configuration: Form fields defined with mandatory validations for Customer ID, Technician Name, Visit Date, and Technician Notes.
    - Input/Output: Inputs via web trigger; outputs form submission data.
    - Edge Cases: Unhandled empty inputs or invalid customer IDs.
  - `Fetch Customer Details`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Looks up customer metadata in the "Customers" sheet using the submitted Customer ID.
    - Configuration: Returns the first match based on the lookup column `Customer ID`. Authenticated via Google Service Account.
    - Expressions: `={{ $('When Technician Submits Form').first().json['Customer ID'].trim() }}`
    - Input/Output: Receives form data; outputs customer record row or empty data.
    - Edge Cases: Spreadsheet API timeout, authentication revocation, or missing columns.
  - `If Customer Found`: 
    - Type: `n8n-nodes-base.if` (v2.2)
    - Role: Branches execution depending on whether a valid customer record was retrieved.
    - Configuration: Loose type validation checking if `{{ $json['Customer ID'] }}` is not empty.
    - Input/Output: Receives customer details; routes to history retrieval (true) or unknown customer alert (false).
  - `Send Unknown Customer Alert`: 
    - Type: `n8n-nodes-base.telegram` (v1.2)
    - Role: Sends an instant warning to operations when a technician submits a visit for an unregistered customer ID.
    - Configuration: Uses Telegram Bot API to send formatted alert text.
    - Expressions: `={{ '⚠️ Unknown Customer ID "' + $('When Technician Submits Form').first().json['Customer ID'] + '" from ' + $('When Technician Submits Form').first().json['Technician Name'] + '. Visit notes were NOT processed:\n\n' + $('When Technician Submits Form').first().json['Technician Notes'] }}`
    - Input/Output: Receives false evaluation from conditional branch; outputs message dispatch confirmation.
    - Edge Cases: Invalid chat ID or Telegram API rate limit.

#### 2.2 Context Preparation & AI Structuring
This block compiles historical visit data, computes rule-based chemistry flags, and passes structured context into Google Gemini for AI analysis.
- **Nodes Involved:** `Fetch Visit History`, `Prepare Agent Context`, `Structure Notes Agent`, `Google Gemini Model`, `Parse Structured Output`
- **Node Details:**
  - `Fetch Visit History`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Pulls past visit records for the customer from the "Visits" sheet.
    - Configuration: Filters rows matching the fetched `Customer ID`. Authenticated via Google Service Account.
    - Expressions: `={{ $('Fetch Customer Details').first().json['Customer ID'] }}`
    - Input/Output: Receives customer ID; outputs historical visit rows.
  - `Prepare Agent Context`: 
    - Type: `n8n-nodes-base.code` (v2)
    - Role: JavaScript execution step that normalizes chemical thresholds, computes rule-based flags (e.g., severe chlorine drop or high filter pressure), formats history, and constructs the prompt payload (`agent_input`).
    - Configuration: Custom JS code block utilizing built-in configuration dictionaries for chemical acceptable ranges and PSI limits.
    - Input/Output: Receives visit history rows and form values; outputs a consolidated JSON object containing structured flags, readings tables, and the master agent prompt string.
  - `Structure Notes Agent`: 
    - Type: `@n8n/n8n-nodes-langchain.agent` (v1.7)
    - Role: LangChain AI Agent that parses unstructured technician notes into strict operational data.
    - Configuration: Configured with a comprehensive system prompt enforcing data extraction rules, with error handling set to continue on error output (`continueErrorOutput`).
    - Expressions: `={{ $json.agent_input }}`
    - Input/Output: Receives `agent_input`; outputs structured AI parameters or triggers error branch.
    - Edge Cases: API hallucinations, rate limits, or output schema mismatches (handled by fallback route).
  - `Google Gemini Model`: 
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (v1)
    - Role: Provides the underlying LLM engine for the agent.
    - Configuration: Uses model `models/gemini-3.1-flash-lite` with a low temperature of `0.1` for deterministic extraction.
    - Sub-workflow/AI Reference: Linked directly to `Structure Notes Agent`.
  - `Parse Structured Output`: 
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.2)
    - Role: Forces the LLM to return output conforming to a manual JSON schema specifying pool conditions, chemical issues, equipment problems, follow-up flags, and customer summaries.
    - Configuration: Manual JSON schema containing enumerations for pool conditions, clarity levels, and severities.

#### 2.3 AI Failure Handling (Fallback)
Captures failures from the AI structuring stage, formats a placeholder record, logs it for manual triage, and notifies the team.
- **Nodes Involved:** `Format Fallback Record`, `Append Failed Visit to Sheets`, `Post AI Failure to Slack`
- **Node Details:**
  - `Format Fallback Record`: 
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Generates a fallback visit record with default placeholder values when the AI agent fails.
    - Input/Output: Receives error state from agent; outputs sanitized fallback record JSON.
  - `Append Failed Visit to Sheets`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Appends the flagged failed visit row to the "Visits" sheet with an error status.
    - Input/Output: Receives fallback record; outputs appended row confirmation.
  - `Post AI Failure to Slack`: 
    - Type: `n8n-nodes-base.slack` (v2.7)
    - Role: Notifies the operations channel that an AI parsing failure occurred and requests manual review.
    - Expressions: `={{ '🛠️ AI structuring FAILED for ' + $json.customer_name + ' (' +$json.visit_id + '). Raw notes saved in Visits sheet, please review manually:\n\n' + $json.raw_notes }}`
    - Input/Output: Receives appended row data; outputs Slack message confirmation.

#### 2.4 Visit Logging & Work Order Management
Builds the final visit payload, records it in Google Sheets, evaluates follow-up necessities, updates customer profiles, and creates work orders.
- **Nodes Involved:** `Build Visit Record`, `Append Visit to Sheets`, `If Follow-Up Needed`, `Append Work Order to Sheets`, `Update Customer in Sheets`
- **Node Details:**
  - `Build Visit Record`: 
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Complex JavaScript utility that sanitizes AI output, merges rule-based flags, computes due dates, evaluates approval triggers, and generates HTML templates for customer reports and manager approval emails.
    - Input/Output: Receives agent output and context; outputs master visit record containing HTML strings, metadata flags, and operational parameters.
  - `Append Visit to Sheets`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Appends the successful structured visit record to the "Visits" spreadsheet.
    - Input/Output: Receives built visit payload; outputs row confirmation.
  - `If Follow-Up Needed`: 
    - Type: `n8n-nodes-base.if` (v2.2)
    - Role: Evaluates whether follow-up actions or repairs are required.
    - Expressions: `={{ $('Build Visit Record').first().json.follow_up_needed }}` equals `"Yes"`.
  - `Append Work Order to Sheets`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Creates a new work order entry in the "Work Orders" sheet if follow-ups are required.
    - Input/Output: Receives data from true branch of follow-up check; outputs created row.
  - `Update Customer in Sheets`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Updates the customer profile in the "Customers" sheet with the latest visit date, condition, and open issues.
    - Configuration: Uses `Customer ID` as the matching column for updates.

#### 2.5 Urgent Alerting & Manager Approvals
Evaluates urgency and manager approval requirements, dispatching alerts and handling email confirmation loops.
- **Nodes Involved:** `If Urgent Issue`, `Post Urgent Alert to Slack`, `If Approval Required`, `Send Manager Approval Email`, `Wait for Manager Response`, `If Manager Approved`, `Mark Report Held in Sheets`, `Email Customer Report`, `Mark Report Sent in Sheets`
- **Node Details:**
  - `If Urgent Issue`: 
    - Type: `n8n-nodes-base.if` (v2.2)
    - Role: Checks if visit urgency is rated as `Urgent`.
  - `Post Urgent Alert to Slack`: 
    - Type: `n8n-nodes-base.slack` (v2.7)
    - Role: Posts high-priority emergency alerts to the Slack operations channel.
    - Expressions: `={{ $('Build Visit Record').first().json.ops_alert }}`
  - `If Approval Required`: 
    - Type: `n8n-nodes-base.if` (v2.2)
    - Role: Determines whether the generated report requires manager review before customer delivery.
    - Expressions: `={{ $('Build Visit Record').first().json.needs_approval }}` equals `"Yes"`.
  - `Send Manager Approval Email`: 
    - Type: `n8n-nodes-base.gmail` (v2.1)
    - Role: Emails a formatted approval summary containing interactive approve/hold action links to the manager.
    - Expressions: Dynamically appends execution resume URLs to approve or reject actions.
  - `Wait for Manager Response`: 
    - Type: `n8n-nodes-base.wait` (v1.1)
    - Role: Pauses workflow execution awaiting webhook input from the manager's email click.
    - Configuration: Resumes via webhook with a 24-hour limit window.
  - `If Manager Approved`: 
    - Type: `n8n-nodes-base.if` (v2.2)
    - Role: Checks if the manager clicked the approval link (`action=approve`).
  - `Mark Report Held in Sheets`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Updates the visit record status to held if the manager rejected or timed out.
  - `Email Customer Report`: 
    - Type: `n8n-nodes-base.gmail` (v2.1)
    - Role: Sends the finalized HTML service report email to the pool owner.
    - Expressions: Uses `={{ $('Build Visit Record').first().json.customer_email }}` and `report_html`.
  - `Mark Report Sent in Sheets`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Updates the "Visits" sheet to log that the customer report has been successfully dispatched.

#### 2.6 Scheduled Operations & Error Monitoring
Runs scheduled routines and global error handlers to maintain oversight.
- **Nodes Involved:** `Every Morning at 7-30 AM`, `Fetch Open Work Orders`, `Compile Daily Digest`, `Send Digest to Telegram`, `On Workflow Error`, `Send Error Alert to Telegram`
- **Node Details:**
  - `Every Morning at 7-30 AM`: 
    - Type: `n8n-nodes-base.scheduleTrigger` (v1.2)
    - Role: Cron trigger executing Monday through Saturday at 07:30 AM (`30 7 * * 1-6`).
  - `Fetch Open Work Orders`: 
    - Type: `n8n-nodes-base.googleSheets` (v4.5)
    - Role: Queries the "Work Orders" sheet for rows where `Status` equals `Open`.
  - `Compile Daily Digest`: 
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Compiles open work orders into an organized Telegram text message categorized by overdue status, due today, and upcoming items.
  - `Send Digest to Telegram`: 
    - Type: `n8n-nodes-base.telegram` (v1.2)
    - Role: Dispatches the daily operations digest to the designated Telegram chat.
  - `On Workflow Error`: 
    - Type: `n8n-nodes-base.errorTrigger` (v1)
    - Role: Global error handling trigger activated upon any unexpected node execution failure.
  - `Send Error Alert to Telegram`: 
    - Type: `n8n-nodes-base.telegram` (v1.2)
    - Role: Sends detailed debugging diagnostics (workflow name, failed node, error message) to Telegram.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation and setup instructions | None | None | Pool Service Visit Generator |
| Sticky Note1 | n8n-nodes-base.stickyNote | Grouping comment for form verification | None | None | Verify customer from form |
| Sticky Note2 | n8n-nodes-base.stickyNote | Grouping comment for unknown customer alert | None | None | Notify missing customer record |
| Sticky Note3 | n8n-nodes-base.stickyNote | Grouping comment for visit history retrieval | None | None | Retrieve customer visit history |
| Sticky Note4 | n8n-nodes-base.stickyNote | Grouping comment for AI notes structuring | None | None | Structure notes with AI |
| Sticky Note5 | n8n-nodes-base.stickyNote | Grouping comment for AI failure handling | None | None | Handle AI processing failure |
| Sticky Note6 | n8n-nodes-base.stickyNote | Grouping comment for completed visit recording | None | None | Record completed visit |
| Sticky Note7 | n8n-nodes-base.stickyNote | Grouping comment for work orders & follow-ups | None | None | Manage follow-ups and work orders |
| Sticky Note8 | n8n-nodes-base.stickyNote | Grouping comment for urgent alerts | None | None | Alert urgent service issues |
| Sticky Note9 | n8n-nodes-base.stickyNote | Grouping comment for manager report approvals | None | None | Manage manager report approvals |
| Sticky Note10 | n8n-nodes-base.stickyNote | Grouping comment for customer report emails | None | None | Send visit report to customer |
| Sticky Note11 | n8n-nodes-base.stickyNote | Grouping comment for daily work order digest | None | None | Daily open work orders digest |
| Sticky Note12 | n8n-nodes-base.stickyNote | Grouping comment for error alerting | None | None | Workflow error alerting |
| When Technician Submits Form | n8n-nodes-base.formTrigger | Captures technician form submissions | Webhook Trigger | Fetch Customer Details | Verify customer from form |
| Fetch Customer Details | n8n-nodes-base.googleSheets | Retrieves customer profile from sheet | When Technician Submits Form | If Customer Found | Verify customer from form |
| If Customer Found | n8n-nodes-base.if | Branches based on customer existence | Fetch Customer Details | Fetch Visit History, Send Unknown Customer Alert | Verify customer from form |
| Send Unknown Customer Alert | n8n-nodes-base.telegram | Alerts Telegram on missing customer ID | If Customer Found | None | Notify missing customer record |
| Fetch Visit History | n8n-nodes-base.googleSheets | Pulls recent customer visit history | If Customer Found | Prepare Agent Context | Retrieve customer visit history |
| Prepare Agent Context | n8n-nodes-base.code | Normalizes readings and builds prompt context | Fetch Visit History | Structure Notes Agent | Retrieve customer visit history |
| Structure Notes Agent | @n8n/n8n-nodes-langchain.agent | AI agent parsing technician notes | Prepare Agent Context, Google Gemini Model, Parse Structured Output | Build Visit Record, Format Fallback Record | Structure notes with AI |
| Google Gemini Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for structuring agent | None | Structure Notes Agent | Structure notes with AI |
| Parse Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces strict JSON schema on AI output | None | Structure Notes Agent | Structure notes with AI |
| Build Visit Record | n8n-nodes-base.code | Formats structured data into final visit payload | Structure Notes Agent | Append Visit to Sheets | Record completed visit |
| Append Visit to Sheets | n8n-nodes-base.googleSheets | Logs successful visit to Google Sheets | Build Visit Record | If Follow-Up Needed | Record completed visit |
| If Follow-Up Needed | n8n-nodes-base.if | Checks if follow-up tasks are needed | Append Visit to Sheets | Append Work Order to Sheets, Update Customer in Sheets | Manage follow-ups and work orders |
| Append Work Order to Sheets | n8n-nodes-base.googleSheets | Creates work order entry | If Follow-Up Needed | Update Customer in Sheets | Manage follow-ups and work orders |
| Update Customer in Sheets | n8n-nodes-base.googleSheets | Updates customer profile status | If Follow-Up Needed, Append Work Order to Sheets | If Urgent Issue | Manage follow-ups and work orders |
| If Urgent Issue | n8n-nodes-base.if | Checks if visit urgency is urgent | Update Customer in Sheets | Post Urgent Alert to Slack, If Approval Required | Alert urgent service issues |
| Post Urgent Alert to Slack | n8n-nodes-base.slack | Posts urgent operational alert to Slack | If Urgent Issue | If Approval Required | Alert urgent service issues |
| If Approval Required | n8n-nodes-base.if | Evaluates if manager approval is needed | If Urgent Issue, Post Urgent Alert to Slack | Send Manager Approval Email, Email Customer Report | Manage manager report approvals |
| Send Manager Approval Email | n8n-nodes-base.gmail | Emails approval request to manager | If Approval Required | Wait for Manager Response | Manage manager report approvals |
| Wait for Manager Response | n8n-nodes-base.wait | Pauses workflow for manager approval webhook | Send Manager Approval Email | If Manager Approved | Manage manager report approvals |
| If Manager Approved | n8n-nodes-base.if | Checks manager decision | Wait for Manager Response | Email Customer Report, Mark Report Held in Sheets | Manage manager report approvals |
| Mark Report Held in Sheets | n8n-nodes-base.googleSheets | Updates visit status as held | If Manager Approved | None | Manage manager report approvals |
| Email Customer Report | n8n-nodes-base.gmail | Emails service report to customer | If Approval Required, If Manager Approved | Mark Report Sent in Sheets | Send visit report to customer |
| Mark Report Sent in Sheets | n8n-nodes-base.googleSheets | Marks visit report status as sent | Email Customer Report | None | Send visit report to customer |
| Format Fallback Record | n8n-nodes-base.code | Generates fallback payload on AI failure | Structure Notes Agent | Append Failed Visit to Sheets | Handle AI processing failure |
| Append Failed Visit to Sheets | n8n-nodes-base.googleSheets | Logs failed AI parse attempt to Sheets | Format Fallback Record | Post AI Failure to Slack | Handle AI processing failure |
| Post AI Failure to Slack | n8n-nodes-base.slack | Notifies Slack channel of AI failure | Append Failed Visit to Sheets | None | Handle AI processing failure |
| Every Morning at 7-30 AM | n8n-nodes-base.scheduleTrigger | Daily cron trigger for open work orders | Schedule Trigger | Fetch Open Work Orders | Daily open work orders digest |
| Fetch Open Work Orders | n8n-nodes-base.googleSheets | Retrieves open work orders from sheet | Every Morning at 7-30 AM | Compile Daily Digest | Daily open work orders digest |
| Compile Daily Digest | n8n-nodes-base.code | Formats open work orders into digest text | Fetch Open Work Orders | Send Digest to Telegram | Daily open work orders digest |
| Send Digest to Telegram | n8n-nodes-base.telegram | Sends daily digest to Telegram chat | Compile Daily Digest | None | Daily open work orders digest |
| On Workflow Error | n8n-nodes-base.errorTrigger | Captures unexpected workflow errors | Error Trigger | Send Error Alert to Telegram | Workflow error alerting |
| Send Error Alert to Telegram | n8n-nodes-base.telegram | Sends error diagnostics to Telegram | On Workflow Error | None | Workflow error alerting |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to rebuild the entire workflow manually in n8n:

1. **Create the Google Sheets Document:** Set up a spreadsheet with three sheets named `Customers`, `Visits`, and `Work Orders` containing the respective column headers referenced in the mapping definitions (e.g., `Customer ID`, `Customer Name`, `Visit ID`, `Visit Date`, `Work Order ID`, etc.).
2. **Set up Credentials:** Configure credentials for:
   - Google Service Account (with read/write access to your Google Sheets document).
   - Google Gemini / PaLM API (using `models/gemini-3.1-flash-lite`).
   - Gmail OAuth2.
   - Slack API.
   - Telegram Bot API.
3. **Build Input Reception Branch:**
   - Create a `When Technician Submits Form` node (`n8n-nodes-base.formTrigger`, v2.2) and configure the form fields (Customer ID, Technician Name, Visit Date, chemical levels, filter pressure, and notes).
   - Add a `Fetch Customer Details` node (`n8n-nodes-base.googleSheets`) connected to the `Customers` sheet, looking up `Customer ID`.
   - Add an `If Customer Found` node to verify that the customer ID exists.
   - Attach a `Send Unknown Customer Alert` Telegram node to the false output of the `If` node.
4. **Build Context & AI Structuring Branch:**
   - Connect the true branch of `If Customer Found` to a `Fetch Visit History` Google Sheets node querying the `Visits` sheet.
   - Connect it to a `Prepare Agent Context` Code node containing threshold evaluation logic and prompt assembly.
   - Add an AI Agent node (`Structure Notes Agent`) connected to a `Google Gemini Model` node and a `Parse Structured Output` output parser node configured with the manual JSON schema. Set error handling on the agent to continue on error output.
5. **Build AI Failure & Fallback Handling:**
   - Connect the error/fallback output of the AI Agent to a `Format Fallback Record` Code node.
   - Connect it to an `Append Failed Visit to Sheets` Google Sheets node and a `Post AI Failure to Slack` Slack node.
6. **Build Visit Logging & Work Order Management:**
   - Connect the successful AI output to a `Build Visit Record` Code node.
   - Connect it to an `Append Visit to Sheets` Google Sheets node.
   - Add an `If Follow-Up Needed` node checking if `follow_up_needed` equals `"Yes"`.
   - If true, connect to an `Append Work Order to Sheets` node. Both branches lead to an `Update Customer in Sheets` node.
7. **Build Urgent Alerting & Manager Approval Flow:**
   - Connect `Update Customer in Sheets` to an `If Urgent Issue` node checking if `urgency` equals `"Urgent"`.
   - If urgent, post an alert using a `Post Urgent Alert to Slack` node.
   - Add an `If Approval Required` node checking if `needs_approval` equals `"Yes"`.
   - If approval is required, connect to a `Send Manager Approval Email` Gmail node, followed by a `Wait for Manager Response` Wait node.
   - Add an `If Manager Approved` node evaluating `{{ $json.query.action }}` equal to `"approve"`.
   - If approved, send the report via an `Email Customer Report` Gmail node and mark it as sent using a `Mark Report Sent in Sheets` node. If rejected or timed out, update the status via `Mark Report Held in Sheets`. If no approval is needed, bypass directly to `Email Customer Report`.
8. **Build Scheduled Digest & Error Triggers:**
   - Create a `Every Morning at 7-30 AM` Schedule Trigger node (`30 7 * * 1-6`), connect it to a `Fetch Open Work Orders` Google Sheets node, a `Compile Daily Digest` Code node, and a `Send Digest to Telegram` node.
   - Create an `On Workflow Error` Error Trigger node and connect it to a `Send Error Alert to Telegram` node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video reference included in the original workflow documentation | [YouTube Tutorial](https://www.youtube.com/watch?v=uAWvXz7FrzI) |