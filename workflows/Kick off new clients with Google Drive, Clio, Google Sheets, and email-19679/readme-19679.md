Kick off new clients with Google Drive, Clio, Google Sheets, and email

https://n8nworkflows.xyz/workflows/kick-off-new-clients-with-google-drive--clio--google-sheets--and-email-19679


# Kick off new clients with Google Drive, Clio, Google Sheets, and email

### 1. Workflow Overview

The **New Client Kickoff Kit** workflow automates the administrative and communication processes required when a new legal client is marked as signed. It initiates directory creation, compliance-checked client outreach, practice management task scheduling, and master log recording. Additionally, it features a scheduled weekly rollup to summarize onboarding metrics and report administrative exceptions.

The logical flow is partitioned into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming CRM payloads via webhook and standardizes client and matter attributes.
- **1.2 Google Drive Provisioning:** Creates the primary client matter directory and evaluates API responses before generating configured subfolders.
- **1.3 Email Preparation & Compliance Evaluation:** Constructs the onboarding kickoff email containing scheduling links and routes it through an external guardrail sub-workflow.
- **1.4 Conditional Email Dispatch:** Routes the communication based on compliance and opt-out statuses, executing SMTP delivery or benign skipping.
- **1.5 Practice Management Task Generation & Master Logging:** Dynamically builds follow-up tasks for Clio and logs all structural onboarding events to a master Google Sheets ledger.
- **1.6 Weekly Rollup Reporting:** Scheduled mechanism that analyzes historical logs on a weekly cadence, aggregates metrics, and emails an internal performance and exception digest.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Normalization
- **Overview:** Receives raw webhook notifications from practice management systems or CRMs when a client signs, sanitizing and mapping incoming parameters for downstream tasks.
- **Nodes Involved:**
  - `When Client Signs`
  - `Parse Client Signed Data`

- **Node Details:**
  - **When Client Signs**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Acts as the primary entry point for the workflow.
    - *Configuration Choices:* Configured to listen for HTTP `POST` requests on the path `client-signed`.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: External HTTP request. Output: `Parse Client Signed Data`.
    - *Version-specific Requirements:* Version 2.1.
    - *Edge Cases / Failure Types:* Invalid JSON payloads or unauthorized external network calls.
    - *Sub-workflow Reference:* None.

  - **Parse Client Signed Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation). Standardizes raw body fields into a uniform schema.
    - *Configuration Choices:* Custom JavaScript execution mapping optional fields (`client_name`, `client_email`, `matter_id`, `practice_area`, `attorney_name`, `attorney_email`) and generating an ISO timestamp for `signed_at`.
    - *Key Expressions or Variables:* Uses `$json.body || $json`.
    - *Input and Output Connections:* Input: `When Client Signs`. Output: `Create Client Matter Folder`.
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* Missing required string attributes resulting in empty strings.
    - *Sub-workflow Reference:* None.

---

#### 2.2 Google Drive Provisioning
- **Overview:** Provisions the root Google Drive folder for the new matter within a designated parent directory and generates standard subfolders conditionally based on success.
- **Nodes Involved:**
  - `Create Client Matter Folder`
  - `Parse Matter Folder Response`
  - `Build Matter Subfolder List`
  - `Create Matter Subfolder`

- **Node Details:**
  - **Create Client Matter Folder**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Sends a POST request to the Google Drive API to create a directory.
    - *Configuration Choices:* Uses generic Google API credentials with Drive scopes. Employs `neverError: true` to prevent workflow interruption if folder generation fails.
    - *Key Expressions or Variables:* `={{ JSON.stringify({ name: $json.client_name + ' — ' + $json.matter_id, mimeType: 'application/vnd.google-apps.folder', parents: [$vars.ONBOARDING_DRIVE_PARENT_FOLDER_ID] }) }}`
    - *Input and Output Connections:* Input: `Parse Client Signed Data`. Output: `Parse Matter Folder Response`.
    - *Version-specific Requirements:* Version 4.4.
    - *Edge Cases / Failure Types:* Authentication revocation, rate limiting, or invalid parent folder ID.
    - *Sub-workflow Reference:* None.

  - **Parse Matter Folder Response**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation). Evaluates the HTTP response from Google Drive to confirm success or failure.
    - *Configuration Choices:* Extracts folder creation status, generating explicit boolean flags and URL construction parameters.
    - *Key Expressions or Variables:* References upstream node data using `$('Parse Client Signed Data').item.json`.
    - *Input and Output Connections:* Input: `Create Client Matter Folder`. Output: `Build Matter Subfolder List`, `Build Kickoff Email Content`, `Build Onboarding Task List`, `Append Onboarding Log To Sheets`.
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* Malformed JSON response structures.
    - *Sub-workflow Reference:* None.

  - **Build Matter Subfolder List**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation). Splits a comma-separated environment variable string into discrete items for subfolder creation.
    - *Configuration Choices:* Bypasses execution if the root folder was not created successfully.
    - *Key Expressions or Variables:* Uses `$vars.ONBOARDING_DRIVE_SUBFOLDERS`.
    - *Input and Output Connections:* Input: `Parse Matter Folder Response`. Output: `Create Matter Subfolder`.
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* Empty or malformed configuration variables.
    - *Sub-workflow Reference:* None.

  - **Create Matter Subfolder**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Loops through subfolder names to create them inside the parent matter folder.
    - *Configuration Choices:* HTTP POST to Google Drive API with `neverError: true`.
    - *Key Expressions or Variables:* `={{ JSON.stringify({ name: $json.subfolder_name, mimeType: 'application/vnd.google-apps.folder', parents: [$json.drive_folder_id] }) }}`
    - *Input and Output Connections:* Input: `Build Matter Subfolder List`. Output: None (Terminal node for this branch).
    - *Version-specific Requirements:* Version 4.4.
    - *Edge Cases / Failure Types:* API timeouts or permission errors on parent items.
    - *Sub-workflow Reference:* None.

---

#### 2.3 Email Preparation & Compliance Evaluation
- **Overview:** Assembles the HTML and plain-text client onboarding email complete with scheduling links, then invokes an external compliance sub-workflow.
- **Nodes Involved:**
  - `Build Kickoff Email Content`
  - `Check Compliance Before Email`

- **Node Details:**
  - **Build Kickoff Email Content**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation). Generates fully responsive HTML and text templates with escaped values.
    - *Configuration Choices:* Pulls firm configuration variables and dynamic client parameters.
    - *Key Expressions or Variables:* Uses `$vars.FIRM_NAME`, `$vars.CALENDLY_KICKOFF_EVENT_URL`, and `$('Parse Matter Folder Response').item.json`.
    - *Input and Output Connections:* Input: `Parse Matter Folder Response`. Output: `Check Compliance Before Email`.
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* Unescaped special characters in client names or practice areas.
    - *Sub-workflow Reference:* None.

  - **Check Compliance Before Email**
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflow` (Sub-Workflow Execution). Invokes a compliance and guardrail validation process.
    - *Configuration Choices:* Maps channel, message body, matter ID, recipient, and template identifier to input schema parameters.
    - *Key Expressions or Variables:* `={{ $vars.GUARDRAIL_WORKFLOW_ID }}`
    - *Input and Output Connections:* Input: `Build Kickoff Email Content`. Output: `If Kickoff Email Approved`.
    - *Version-specific Requirements:* Version 1.3.
    - *Edge Cases / Failure Types:* Missing target sub-workflow ID or execution failures within the guardrail.
    - *Sub-workflow Reference:* Invokes the workflow specified by `GUARDRAIL_WORKFLOW_ID`.

---

#### 2.4 Conditional Email Dispatch
- **Overview:** Evaluates the compliance output to determine whether to transmit the message via SMTP or terminate the communication branch safely due to opt-out rules.
- **Nodes Involved:**
  - `If Kickoff Email Approved`
  - `Send Kickoff Email`
  - `Skip Email If Client Opted Out`

- **Node Details:**
  - **If Kickoff Email Approved**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Splits execution based on compliance boolean output.
    - *Configuration Choices:* Evaluates `{{ $json.approved }}` using loose type validation.
    - *Key Expressions or Variables:* `={{ $json.approved }}`
    - *Input and Output Connections:* Input: `Check Compliance Before Email`. Output: True Branch (`Send Kickoff Email`), False Branch (`Skip Email If Client Opted Out`).
    - *Version-specific Requirements:* Version 2.3.
    - *Edge Cases / Failure Types:* Unexpected boolean payload types.
    - *Sub-workflow Reference:* None.

  - **Send Kickoff Email**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (Communication). Transmits the formatted email using SMTP credentials.
    - *Configuration Choices:* Uses explicit HTML and text body references from the build node, bypassing intermediate payload modifications.
    - *Key Expressions or Variables:* `={{ $('Build Kickoff Email Content').item.json.email_html }}`, `={{ $vars.FIRM_FROM_EMAIL }}`.
    - *Input and Output Connections:* Input: `If Kickoff Email Approved` (True). Output: None (Terminal node).
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* SMTP authentication errors, invalid recipient addresses, or connection timeouts.
    - *Sub-workflow Reference:* None.

  - **Skip Email If Client Opted Out**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Flow Control / Placeholder). Serves as a termination target for opted-out clients.
    - *Configuration Choices:* Default pass-through parameters.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: `If Kickoff Email Approved` (False). Output: None (Terminal node).
    - *Version-specific Requirements:* Version 1.0.
    - *Edge Cases / Failure Types:* None.
    - *Sub-workflow Reference:* None.

---

#### 2.5 Practice Management Task Generation & Master Logging
- **Overview:** Parses configuration variables to generate deferred follow-up tasks in Clio and logs all foundational onboarding metrics into a master Google Sheet.
- **Nodes Involved:**
  - `Build Onboarding Task List`
  - `Create Clio Onboarding Task`
  - `Append Onboarding Log To Sheets`

- **Node Details:**
  - **Build Onboarding Task List**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation). Splits day offset and task description definitions into individual execution items.
    - *Configuration Choices:* Computes dynamic future due dates based on arithmetic offsets.
    - *Key Expressions or Variables:* Uses `$vars.ONBOARDING_FOLLOWUP_TASKS`.
    - *Input and Output Connections:* Input: `Parse Matter Folder Response`. Output: `Create Clio Onboarding Task`.
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* Malformed delimiters or invalid number conversions.
    - *Sub-workflow Reference:* None.

  - **Create Clio Onboarding Task**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Submits task creation payloads to the Clio REST API.
    - *Configuration Choices:* Uses generic OAuth2 API credentials for Clio, setting task priority and matter association.
    - *Key Expressions or Variables:* `=https://{{ $vars.CLIO_BASE_URL.replace('https://', '') }}/api/v4/tasks.json`
    - *Input and Output Connections:* Input: `Build Onboarding Task List`. Output: None (Terminal node).
    - *Version-specific Requirements:* Version 4.4.
    - *Edge Cases / Failure Types:* Expired OAuth tokens, invalid matter IDs, or missing permissions.
    - *Sub-workflow Reference:* None.

  - **Append Onboarding Log To Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration). Appends a new row of onboarding metadata to the master tracking document.
    - *Configuration Choices:* Appends data to sheet name `Master Onboarding Log` using document ID configuration variables.
    - *Key Expressions or Variables:* `={{ $vars.ONBOARDING_LOG_SHEET_ID }}`
    - *Input and Output Connections:* Input: `Parse Matter Folder Response`. Output: None (Terminal node).
    - *Version-specific Requirements:* Version 4.0.
    - *Edge Cases / Failure Types:* Sheet permission errors, missing column definitions, or invalid spreadsheet IDs.
    - *Sub-workflow Reference:* None.

---

#### 2.6 Weekly Rollup Reporting
- **Overview:** Periodically queries the historical onboarding ledger, calculates performance metrics and exceptions, and distributes an internal summary report via SMTP.
- **Nodes Involved:**
  - `Weekly Onboarding Rollup Trigger`
  - `Read Onboarding Log Sheet`
  - `Summarize Weekly Onboarding Data`
  - `Build Weekly Onboarding Email`
  - `Send Weekly Onboarding Email`

- **Node Details:**
  - **Weekly Onboarding Rollup Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger). Initiates the weekly reporting workflow sequence.
    - *Configuration Choices:* Configured with a Cron expression set for every Monday at 8:00 AM (`0 8 * * 1`).
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: None (Trigger). Output: `Read Onboarding Log Sheet`.
    - *Version-specific Requirements:* Version 1.3.
    - *Edge Cases / Failure Types:* Server time zone discrepancies.
    - *Sub-workflow Reference:* None.

  - **Read Onboarding Log Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration). Reads all rows from the master log spreadsheet.
    - *Configuration Choices:* Operation set to `read` on sheet `Master Onboarding Log`.
    - *Key Expressions or Variables:* `={{ $vars.ONBOARDING_LOG_SHEET_ID }}`
    - *Input and Output Connections:* Input: `Weekly Onboarding Rollup Trigger`. Output: `Summarize Weekly Onboarding Data`.
    - *Version-specific Requirements:* Version 4.0.
    - *Edge Cases / Failure Types:* Large datasets causing memory overhead or missing sheet names.
    - *Sub-workflow Reference:* None.

  - **Summarize Weekly Onboarding Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation). Filters historical entries by lookback window and isolates folder creation failures.
    - *Configuration Choices:* Uses environment variable settings for lookback duration defaulting to 7 days.
    - *Key Expressions or Variables:* Uses `$vars.ONBOARDING_SUMMARY_LOOKBACK_DAYS`.
    - *Input and Output Connections:* Input: `Read Onboarding Log Sheet`. Output: `Build Weekly Onboarding Email`.
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* Invalid date string formatting in logged rows.
    - *Sub-workflow Reference:* None.

  - **Build Weekly Onboarding Email**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation). Constructs conditional email markup reporting zero-state or exception conditions.
    - *Configuration Choices:* Formats HTML tables for failed items or displays a clean empty state.
    - *Key Expressions or Variables:* Uses `$vars.FIRM_EMAIL`.
    - *Input and Output Connections:* Input: `Summarize Weekly Onboarding Data`. Output: `Send Weekly Onboarding Email`.
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* Missing firm email destination variables.
    - *Sub-workflow Reference:* None.

  - **Send Weekly Onboarding Email**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (Communication). Delivers the internal weekly management rollup via SMTP.
    - *Configuration Choices:* Sends HTML and plain-text alternatives to internal stakeholders.
    - *Key Expressions or Variables:* `={{ $json.email_html }}`, `={{ $vars.FIRM_FROM_EMAIL }}`.
    - *Input and Output Connections:* Input: `Build Weekly Onboarding Email`. Output: None (Terminal node).
    - *Version-specific Requirements:* Version 2.0.
    - *Edge Cases / Failure Types:* SMTP relay authentication limits.
    - *Sub-workflow Reference:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation block outlining workflow design, setup steps, and customization options. | None | None | ## New Client Kickoff Kit<br><br>### How it works<br><br>1. The workflow triggers when a client is marked as signed and processes the client's data.<br>2. It creates a main matter folder on Google Drive along with configured subfolders.<br>3. It builds and conditionally sends a kickoff email after compliance approval.<br>4. Onboarding follow-up tasks are created in Clio, and onboarding information is logged to a master Google Sheet.<br>5. A separate scheduled task reads the onboarding log, computes weekly summaries, builds reports, and sends a weekly onboarding email.<br><br>### Setup steps<br><br>- [ ] Configure the webhook URL to receive client signed notifications.<br>- [ ] Set up Google Drive API credentials for folder and subfolder creation.<br>- [ ] Provide Clio API credentials for follow-up task creation.<br>- [ ] Configure Google Sheets credentials and specify the master sheet for logging onboardings.<br>- [ ] Set up email credentials for sending kickoff and weekly summary emails.<br>- [ ] Configure the sub-workflow for compliance checking used before sending kickoff emails.<br><br>### Customization<br><br>Customize the names and number of subfolders in the ONBOARDING_DRIVE_SUBFOLDERS environment variable or in the Build Subfolder List node's code.<br>Adjust the ONBOARDING_FOLLOWUP_TASKS format or tasks in the Build Onboarding Follow-Up Task List node to match desired onboarding reminders.<br>Modify email templates in both Build Kickoff Email and Build Weekly Onboarding Email code nodes as needed. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Visual grouping note for client webhook trigger and initial data parsing. | None | None | ## Client signed trigger and parsing<br><br>Starts with the webhook trigger when a client is marked signed and parses the incoming client data. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Visual grouping note for main Google Drive folder creation. | None | None | ## Create main drive folder<br><br>Creates the main matter folder on Google Drive and parses the response for further processing. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Visual grouping note for subfolder structure generation. | None | None | ## Subfolder creation<br><br>Builds a list of subfolders based on configured names and creates these subfolders inside the main matter folder. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Visual grouping note for email preparation and compliance checks. | None | None | ## Kickoff email preparation and compliance<br><br>Builds the kickoff email content and runs a compliance sub-workflow before possible sending. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Visual grouping note for conditional email delivery and opt-out handling. | None | None | ## Kickoff email sending and opt-out<br><br>Sends the kickoff email if approved or passes through if client opted out. |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Visual grouping note for Clio task generation. | None | None | ## Follow-up task creation<br><br>Builds onboarding follow-up tasks and creates them in Clio. |
| `Sticky Note7` | `n8n-nodes-base.stickyNote` | Visual grouping note for master sheet event recording. | None | None | ## Log onboarding to master sheet<br><br>Logs the onboarding event to a master Google Sheet for record keeping. |
| `Sticky Note8` | `n8n-nodes-base.stickyNote` | Visual grouping note for scheduled weekly management rollups. | None | None | ## Weekly onboarding rollup<br><br>Scheduled trigger reads the onboarding log, summarizes weekly onboarding metrics, builds a report, and sends a summary email. |
| `When Client Signs` | `n8n-nodes-base.webhook` | Receives client signed POST payloads from external CRM or PM tool. | None | `Parse Client Signed Data` | Wire your CRM/PM tool's 'mark as signed' action (or a simple button/Zapier step) to POST here with the client's name, email, matter ID, practice area, and attorney contact. |
| `Parse Client Signed Data` | `n8n-nodes-base.code` | Normalizes incoming webhook fields into structured client and matter data properties. | `When Client Signs` | `Create Client Matter Folder` | Field names here match a generic 'mark as signed' payload — adjust to whatever your CRM/PM tool actually sends. |
| `Create Client Matter Folder` | `n8n-nodes-base.httpRequest` | Sends an API request to Google Drive to create the primary matter folder. | `Parse Client Signed Data` | `Parse Matter Folder Response` | Creates the top-level matter folder inside your configured parent folder (ONBOARDING_DRIVE_PARENT_FOLDER_ID). Needs a Google API (OAuth2) credential with Drive scope -- wire it manually after import. neverError is on so a failure doesn't halt the rest of kickoff -- checked in the next node. |
| `Parse Matter Folder Response` | `n8n-nodes-base.code` | Evaluates HTTP response flags to verify root Google Drive folder creation success. | `Create Client Matter Folder` | `Build Matter Subfolder List`, `Build Kickoff Email Content`, `Build Onboarding Task List`, `Append Onboarding Log To Sheets` | Reads the previous node's raw HTTP response, not $json passthrough -- checks for a Drive API error and continues either way. |
| `Build Matter Subfolder List` | `n8n-nodes-base.code` | Splits environment variables into discrete items to generate subfolders. | `Parse Matter Folder Response` | `Create Matter Subfolder` | ONBOARDING_DRIVE_SUBFOLDERS controls the whole structure -- edit that n8n Variable, not the code. |
| `Create Matter Subfolder` | `n8n-nodes-base.httpRequest` | Iterates over subfolder names and creates them inside the main folder. | `Build Matter Subfolder List` | None | Runs once per subfolder from the list above, nested inside the matter folder just created. |
| `Build Kickoff Email Content` | `n8n-nodes-base.code` | Constructs the HTML and text templates for the client onboarding email. | `Parse Matter Folder Response` | `Check Compliance Before Email` | CALENDLY_KICKOFF_EVENT_URL is your firm's scheduling link for the kickoff-meeting event type — reads from 'Parse Folder Creation Response' by name so this runs independent of how many subfolders were created. |
| `Check Compliance Before Email` | `n8n-nodes-base.executeWorkflow` | Invokes the compliance guardrail sub-workflow to evaluate message content. | `Build Kickoff Email Content` | `If Kickoff Email Approved` | Real communication to a real client -- opt-out and disclaimer handling here are genuinely load-bearing. |
| `If Kickoff Email Approved` | `n8n-nodes-base.if` | Routes execution based on whether the compliance check approved the email. | `Check Compliance Before Email` | `Send Kickoff Email`, `Skip Email If Client Opted Out` | Mirrors the guardrail's approved/opted-out result. |
| `Send Kickoff Email` | `n8n-nodes-base.emailSend` | Sends the kickoff email via SMTP if approved by the guardrail. | `If Kickoff Email Approved` | None | Reaches back to 'Build Kickoff Email' by name, not $json, since the guardrail call in between replaces $json with its own response. |
| `Skip Email If Client Opted Out` | `n8n-nodes-base.noOp` | Terminates email dispatch benignly if the client opted out. | `If Kickoff Email Approved` | None | Rare for a brand-new signed client, but respected the same as any other opt-out check in this catalog. |
| `Build Onboarding Task List` | `n8n-nodes-base.code` | Parses environment variables to build offset-based follow-up tasks. | `Parse Matter Folder Response` | `Create Clio Onboarding Task` | ONBOARDING_FOLLOWUP_TASKS controls which tasks get created and when — edit that n8n Variable, not the code. |
| `Create Clio Onboarding Task` | `n8n-nodes-base.httpRequest` | Creates follow-up onboarding tasks in Clio via API requests. | `Build Onboarding Task List` | None | Runs once per configured task from the previous step, landing in your firm's normal Clio task queue. |
| `Append Onboarding Log To Sheets` | `n8n-nodes-base.googleSheets` | Appends onboarding transaction details to the master Google Sheet log. | `Parse Matter Folder Response` | None | One row per new client, regardless of whether the Drive folder or kickoff email succeeded -- runs in parallel with those branches, not gated behind them. folder_created lets you spot any manual follow-up needed. |
| `Weekly Onboarding Rollup Trigger` | `n8n-nodes-base.scheduleTrigger` | Triggers the weekly onboarding summary execution every Monday morning. | None | `Read Onboarding Log Sheet` | Runs weekly, Monday 8am by default. |
| `Read Onboarding Log Sheet` | `n8n-nodes-base.googleSheets` | Reads historical onboarding log entries from Google Sheets. | `Weekly Onboarding Rollup Trigger` | `Summarize Weekly Onboarding Data` | Reads every logged onboarding, regardless of week. |
| `Summarize Weekly Onboarding Data` | `n8n-nodes-base.code` | Filters records by lookback window and computes failure exceptions. | `Read Onboarding Log Sheet` | `Build Weekly Onboarding Email` | ONBOARDING_SUMMARY_LOOKBACK_DAYS controls the window — defaults to 7. |
| `Build Weekly Onboarding Email` | `n8n-nodes-base.code` | Generates summary reports and empty-state notifications for management. | `Summarize Weekly Onboarding Data` | `Send Weekly Onboarding Email` | Explicitly sends a 'no new clients' message rather than a blank digest when nothing came in this week. |
| `Send Weekly Onboarding Email` | `n8n-nodes-base.emailSend` | Sends the weekly internal summary report via SMTP. | `Build Weekly Onboarding Email` | None | Internal, firm-wide summary — no guardrail wrap needed, matching this catalog's other weekly rollups. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild the workflow manually without importing the JSON, follow these sequential steps:

1. **Initialize the Webhook Trigger:**
   - Create a node of type `n8n-nodes-base.webhook`.
   - Set parameters: HTTP Method to `POST`, Path to `client-signed`.

2. **Add Data Parsing:**
   - Create a Code node (`n8n-nodes-base.code`), named `Parse Client Signed Data`.
   - Connect `When Client Signs` to this node.
   - Insert JavaScript to trim and extract `client_name`, `client_email`, `matter_id`, `practice_area`, `attorney_name`, `attorney_email`, and generate `signed_at`.

3. **Provision the Google Drive Root Folder:**
   - Create an HTTP Request node (`n8n-nodes-base.httpRequest`), named `Create Client Matter Folder`.
   - Connect `Parse Client Signed Data` to it.
   - Configure: Method `POST`, URL `https://www.googleapis.com/drive/v3/files`. Set Authentication to generic Google API (OAuth2). Enable `neverError` in response options.
   - Set JSON body expression to send the folder name, `application/vnd.google-apps.folder` mimeType, and parent ID referencing `$vars.ONBOARDING_DRIVE_PARENT_FOLDER_ID`.

4. **Parse Root Folder Response:**
   - Create a Code node (`n8n-nodes-base.code`), named `Parse Matter Folder Response`.
   - Connect `Create Client Matter Folder` to it.
   - Insert JavaScript checking for a valid folder ID and constructing `drive_folder_url` and `folder_created` flags.

5. **Branch 1 — Subfolder Generation:**
   - Create a Code node (`n8n-nodes-base.code`), named `Build Matter Subfolder List`, connected from `Parse Matter Folder Response`. Split `$vars.ONBOARDING_DRIVE_SUBFOLDERS` into items.
   - Create an HTTP Request node (`n8n-nodes-base.httpRequest`), named `Create Matter Subfolder`, connected from the subfolder list node. Configure `POST` to `https://www.googleapis.com/drive/v3/files` with Google API credentials, setting parents dynamically to `drive_folder_id`.

6. **Branch 2 — Kickoff Email & Compliance:**
   - Create a Code node (`n8n-nodes-base.code`), named `Build Kickoff Email Content`, connected from `Parse Matter Folder Response`. Construct HTML and text templates with scheduling URLs.
   - Create an Execute Workflow node (`n8n-nodes-base.executeWorkflow`), named `Check Compliance Before Email`, connected from the email builder. Configure expression workflow ID pointing to `{{ $vars.GUARDRAIL_WORKFLOW_ID }}` and map inputs (`channel`, `message`, `matter_id`, `recipient`, `template_id`).
   - Create an If node (`n8n-nodes-base.if`), named `If Kickoff Email Approved`, connected from the compliance node. Set condition to test `{{ $json.approved }}` equals true.
   - Create an Email Send node (`n8n-nodes-base.emailSend`), named `Send Kickoff Email`, connected to the True output of the If node. Configure SMTP credentials, pulling parameters via item references (`$('Build Kickoff Email Content').item.json`).
   - Create a No-Op node (`n8n-nodes-base.noOp`), named `Skip Email If Client Opted Out`, connected to the False output of the If node.

7. **Branch 3 — Clio Tasks & Master Sheet Logging:**
   - Create a Code node (`n8n-nodes-base.code`), named `Build Onboarding Task List`, connected from `Parse Matter Folder Response`. Parse `$vars.ONBOARDING_FOLLOWUP_TASKS` using offset definitions.
   - Create an HTTP Request node (`n8n-nodes-base.httpRequest`), named `Create Clio Onboarding Task`, connected from the task list node. Configure `POST` to `https://{CLIO_HOST}/api/v4/tasks.json` with generic OAuth2 credentials.
   - Create a Google Sheets node (`n8n-nodes-base.googleSheets`), named `Append Onboarding Log To Sheets`, connected from `Parse Matter Folder Response`. Set operation to `append`, document ID to `{{ $vars.ONBOARDING_LOG_SHEET_ID }}`, sheet name to `Master Onboarding Log`, and map the required metadata columns.

8. **Branch 4 — Scheduled Weekly Rollup:**
   - Create a Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`), named `Weekly Onboarding Rollup Trigger`, configured with cron expression `0 8 * * 1`.
   - Create a Google Sheets node (`n8n-nodes-base.googleSheets`), named `Read Onboarding Log Sheet`, connected from the trigger. Set operation to `read`, document ID to `{{ $vars.ONBOARDING_LOG_SHEET_ID }}`, and sheet name to `Master Onboarding Log`.
   - Create a Code node (`n8n-nodes-base.code`), named `Summarize Weekly Onboarding Data`, connected from the sheet read node. Implement lookback date filtering and exception collection.
   - Create a Code node (`n8n-nodes-base.code`), named `Build Weekly Onboarding Email`, connected from the summarization node to generate reporting HTML markup.
   - Create an Email Send node (`n8n-nodes-base.emailSend`), named `Send Weekly Onboarding Email`, connected from the email builder node. Configure SMTP credentials and recipient destinations.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Configured parent folder identifier variable | Referenced via environment variable `ONBOARDING_DRIVE_PARENT_FOLDER_ID`. |
| Subfolder configuration string variable | Referenced via environment variable `ONBOARDING_DRIVE_SUBFOLDERS`. |
| Follow-up task list configuration variable | Referenced via environment variable `ONBOARDING_FOLLOWUP_TASKS`. |
| Master spreadsheet ledger identifier variable | Referenced via environment variable `ONBOARDING_LOG_SHEET_ID`. |
| Calendly scheduling link variable | Referenced via environment variable `CALENDLY_KICKOFF_EVENT_URL`. |
| Guardrail sub-workflow pointer variable | Referenced via environment variable `GUARDRAIL_WORKFLOW_ID`. |