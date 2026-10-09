Chase client documents with WhatsApp, Gmail, Google Sheets and GPT-4o-mini

https://n8nworkflows.xyz/workflows/chase-client-documents-with-whatsapp--gmail--google-sheets-and-gpt-4o-mini-20432


# Chase client documents with WhatsApp, Gmail, Google Sheets and GPT-4o-mini

### 1. Workflow Overview

This workflow automates document collection, tracking, and follow-ups for Chartered Accountant (CA) firms. It manages the lifecycle of client document requests—from initial intake and Google Drive organization to automated multi-channel messaging (WhatsApp and Gmail), client file uploads via a secure portal, intelligent AI-driven reminders, and internal status dashboards.

The logic is grouped into the following functional blocks:
- **1.1 Request Intake:** Receives new client document requests via an internal staff form or an external webhook, then configures firm-wide settings.
- **1.2 Folder, Tracker & First Message:** Creates a dedicated Google Drive folder for the client, logs the request in Google Sheets, dispatches the initial request via WhatsApp and Gmail, and logs the creation event.
- **1.3 Client Upload Portal:** Captures client uploads through an interactive n8n form, verifies the request ID against the tracker, and processes the incoming files.
- **1.4 File & Update Checklist:** Saves uploaded files directly to the client’s Google Drive folder and recalculates the document checklist status in Google Sheets.
- **1.5 Receipts & Ready to File:** Sends an upload receipt to the client detailing remaining pending items, logs the upload activity, and notifies the assigned CA if all required documents are received.
- **1.6 Daily Run & Team Digest:** Runs a daily cron trigger (Mon–Sat at 10:00 AM) to pull all active requests from Google Sheets and email a summary digest to the firm's team.
- **1.7 AI Reminder Drafting:** Evaluates pending requests, determines follow-up urgency, and utilizes GPT-4o-mini to draft context-aware reminder messages.
- **1.8 Send, Track & Escalate:** Dispatches WhatsApp and Gmail reminders, updates tracker counters, logs reminder activity, and escalates overdue or unresponsive files to the assigned CA.
- **1.9 Live Status Board & Security:** Serves a password-protected HTML status board via a webhook, displaying real-time tracking metrics and file completion progress.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Request Intake
- **Overview:** Ingests document request payloads from either an internal staff form submission or an API webhook, normalizes the client parameters, and establishes global operational variables.
- **Nodes Involved:** 
  - `New Request Form (Staff)1` (Form Trigger)
  - `Receive Request via Webhook` (Webhook)
  - `Set Intake Config` (Set)
  - `Normalize Request Data` (Code)

- **Node Details:**
  - **New Request Form (Staff)1**
    - *Type and Technical Role:* Form Trigger (`n8n-nodes-base.formTrigger`). Generates an internal web form for staff to create document requests.
    - *Configuration:* Path set to `ca-new-document-request`. Collects client name, email, WhatsApp, compliance type, period, documents required, due date, assigned CA email, and notes.
    - *Input/Output:* Trigger node. Outputs user-submitted form data.
  - **Receive Request via Webhook**
    - *Type and Technical Role:* Webhook (`n8n-nodes-base.webhook`). Acts as an alternative ingestion point for external systems via HTTP POST.
    - *Configuration:* Path set to `ca-document-request`, method set to `POST`.
    - *Input/Output:* Trigger node. Outputs incoming JSON payload.
  - **Set Intake Config**
    - *Type and Technical Role:* Set (`n8n-nodes-base.set`). Defines configuration parameters used across the intake pipeline.
    - *Configuration:* Assigns firm metadata, Google Sheet ID, tab names (`Document Requests`, `Activity Log`), Google Drive root folder ID, n8n base URL, WhatsApp API credentials/templates, default country code (`91`), reminder intervals, and board security keys.
    - *Input/Output:* Connected from intake triggers. Outputs configuration variables merged with item data.
  - **Normalize Request Data**
    - *Type and Technical Role:* Code (`n8n-nodes-base.code`). Standardizes raw input fields into a clean, uniform data schema.
    - *Configuration:* Parses comma/newline-separated document lists, formats Indian date strings, generates unique request IDs (`DOC-YYYYMMDD-XXXX`), calculates WhatsApp payload formats, and constructs unique secure upload links.
    - *Input/Output:* Receives intake payload and configuration. Outputs standardized request objects.
  - **Edge Cases & Failure Types:** Missing client names drop execution. Phone number normalization handles 10-digit inputs by prepending the default country code. Invalid dates default to 7 days from creation.

---

#### Block 1.2: Folder, Tracker & First Message
- **Overview:** Provisions a dedicated client storage folder in Google Drive, appends a new tracking record to Google Sheets, and broadcasts the initial document request via WhatsApp Cloud API and Gmail.
- **Nodes Involved:**
  - `Create Client Drive Folder` (Google Drive)
  - `Build Request Record & Messages` (Code)
  - `Add Request to Tracker Sheet` (Google Sheets)
  - `Send WhatsApp Document Request` (HTTP Request)
  - `Email Document Request` (Gmail)
  - `Log Request Created1` (Google Sheets)

- **Node Details:**
  - **Create Client Drive Folder**
    - *Type and Technical Role:* Google Drive (`n8n-nodes-base.googleDrive`). Creates a client-specific subfolder under the root directory.
    - *Configuration:* Resource: `folder`, Operation: `create`. Folder name mapped to `{{ $json.folder_name }}` inside the root folder ID defined in configuration.
    - *Input/Output:* Input: Normalized request data. Output: Google Drive folder metadata (including folder ID).
  - **Build Request Record & Messages**
    - *Type and Technical Role:* Code (`n8n-nodes-base.code`). Constructs database row schemas, HTML email bodies with styling/checklists, WhatsApp message templates, and audit log entries.
    - *Input/Output:* Input: Normalized request data and Drive folder creation output. Output: Structured row objects, email parameters, WhatsApp JSON payloads, and log records.
  - **Add Request to Tracker Sheet**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Appends the new request record to the tracking spreadsheet.
    - *Configuration:* Operation: `append`. Sheet name mapped via configuration (`requests_sheet`).
    - *Input/Output:* Input: Structured row data. Output: Appended sheet row confirmation.
  - **Send WhatsApp Document Request**
    - *Type and Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`). Sends an official template message via the Meta WhatsApp Cloud API.
    - *Configuration:* Method: `POST`, URL constructed using `wa_api_version` and `wa_phone_number_id`. Authentication via Generic Header Auth (Bearer token). Error handling set to continue regular output.
    - *Input/Output:* Input: Built message payload. Output: API response.
  - **Email Document Request**
    - *Type and Technical Role:* Gmail (`n8n-nodes-base.gmail`). Sends the initial document checklist and secure upload link to the client.
    - *Configuration:* Recipient mapped to client email, subject and HTML body mapped from code node. Attribution appending disabled.
    - *Input/Output:* Input: Email data. Output: Sent email confirmation.
  - **Log Request Created1**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Appends an event record to the activity audit log.
    - *Configuration:* Operation: `append`. Sheet name mapped to `activity_sheet`.
    - *Input/Output:* Input: Log parameters. Output: Appended row confirmation.

---

#### Block 1.3: Client Upload Portal
- **Overview:** Receives secure client document submissions through an n8n form interface, validates the embedded request ID, and splits multi-file uploads into individual processing streams.
- **Nodes Involved:**
  - `Client Upload Portal Form` (Form Trigger)
  - `Set Upload Config` (Set)
  - `Find Request in Tracker` (Google Sheets)
  - `Request Found?1` (If)
  - `Split Uploaded Files1` (Code)

- **Node Details:**
  - **Client Upload Portal Form**
    - *Type and Technical Role:* Form Trigger (`n8n-nodes-base.formTrigger`). Serves the secure document upload web portal.
    - *Configuration:* Path set to `ca-document-upload`. Collects hidden `request_id`, document checklist checkboxes, file attachments (`.pdf`, images, spreadsheets, etc.), and optional text notes for the CA.
    - *Input/Output:* Trigger node. Outputs submitted form fields and binary file attachments.
  - **Set Upload Config**
    - *Type and Technical Role:* Set (`n8n-nodes-base.set`). Merges operational configuration constants for the upload processing environment.
    - *Input/Output:* Input: Form trigger execution. Output: Configuration variables.
  - **Find Request in Tracker**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Queries the tracking sheet to locate the matching request record.
    - *Configuration:* Operation: `get` (or lookup filter). Filters rows where `request_id` matches form submission `request_id`. Always output data enabled.
    - *Input/Output:* Input: Form submission and configuration. Output: Matching request row data.
  - **Request Found?1**
    - *Type and Technical Role:* If (`n8n-nodes-base.if`). Validates whether a valid request ID exists in the tracking sheet.
    - *Configuration:* Condition: `request_id` is not empty.
    - *Input/Output:* Input: Tracker search results. Output: Routes valid requests forward; halts unknown request IDs.
  - **Split Uploaded Files1**
    - *Type and Technical Role:* Code (`n8n-nodes-base.code`). Iterates through binary file attachments, separating them into distinct individual items for downstream Google Drive storage.
    - *Input/Output:* Input: Form binary data and tracker row. Output: Array of individual file items containing binary data and target folder parameters.

---

#### Block 1.4: File & Update Checklist
- **Overview:** Uploads incoming client files into their designated Google Drive folder and evaluates checklist completion metrics.
- **Nodes Involved:**
  - `Save Files to Client Folder` (Google Drive)
  - `Compute Checklist Status1` (Code)
  - `Update Tracker Checklist` (Google Sheets)

- **Node Details:**
  - **Save Files to Client Folder**
    - *Type and Technical Role:* Google Drive (`n8n-nodes-base.googleDrive`). Uploads binary files into Google Drive.
    - *Configuration:* Resource: `file`, Operation: `upload`. File name prefixed with date stamp; folder ID mapped dynamically to client folder.
    - *Input/Output:* Input: Split file items. Output: Uploaded file metadata (including file ID and web link).
  - **Compute Checklist Status1**
    - *Type and Technical Role:* Code (`n8n-nodes-base.code`). Merges newly uploaded documents with historical submissions, calculates remaining pending items, sets completion flags, and builds client/CA notification payloads.
    - *Input/Output:* Input: Uploaded file metadata and tracker row. Output: Computed checklist status, update objects, notification payloads, and audit logs.
  - **Update Tracker Checklist**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Updates the request row in Google Sheets with current progress.
    - *Configuration:* Operation: `update`. Matching column set to `request_id`. Updates status (`Complete` or `Partial`), received/pending lists, upload timestamps, and next reminder schedules.
    - *Input/Output:* Input: Computed update metrics. Output: Sheet update confirmation.

---

#### Block 1.5: Receipts & Ready to File
- **Overview:** Dispatches upload confirmations to clients, logs the upload activity, and escalates completed files to the assigned CA.
- **Nodes Involved:**
  - `Send WhatsApp Upload Receipt` (HTTP Request)
  - `Email Upload Receipt1` (Gmail)
  - `Log Upload Activity` (Google Sheets)
  - `All Documents In?1` (If)
  - `Email CA: Ready to File` (Gmail)

- **Node Details:**
  - **Send WhatsApp Upload Receipt**
    - *Type and Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`). Sends a WhatsApp receipt template via Meta Cloud API.
    - *Configuration:* Method: `POST`, authentication via Header Auth. Continues on regular output upon failure.
    - *Input/Output:* Input: Computed checklist payload. Output: API response.
  - **Email Upload Receipt1**
    - *Type and Technical Role:* Gmail (`n8n-nodes-base.gmail`). Emails an upload receipt and current checklist status to the client.
    - *Configuration:* Recipient mapped to client email.
    - *Input/Output:* Input: Computed client HTML content. Output: Sent email confirmation.
  - **Log Upload Activity**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Appends an event entry to the activity log sheet.
    - *Configuration:* Operation: `append`. Sheet name mapped to `activity_sheet`.
    - *Input/Output:* Input: Log parameters. Output: Appended row confirmation.
  - **All Documents In?1**
    - *Type and Technical Role:* If (`n8n-nodes-base.if`). Evaluates whether all required checklist items have been received.
    - *Configuration:* Condition: `complete` equals boolean `true`.
    - *Input/Output:* Input: Computed checklist status. Output: Triggers CA notification if complete.
  - **Email CA: Ready to File**
    - *Type and Technical Role:* Gmail (`n8n-nodes-base.gmail`). Notifies the assigned CA that all required documents have been uploaded and files are ready for review.
    - *Configuration:* Recipient mapped to assigned CA email.
    - *Input/Output:* Input: CA notification HTML. Output: Sent email confirmation.

---

#### Block 1.6: Daily Run & Team Digest
- **Overview:** Executes a daily schedule trigger to fetch all active tracking requests and email an executive summary digest to the accounting team.
- **Nodes Involved:**
  - `Daily 10 AM Mon-Sat` (Schedule Trigger)
  - `Set Chaser Config` (Set)
  - `Get All Requests1` (Google Sheets)
  - `Build Team Digest1` (Code)
  - `Email Team Digest1` (Gmail)

- **Node Details:**
  - **Daily 10 AM Mon-Sat**
    - *Type and Technical Role:* Schedule Trigger (`n8n-nodes-base.scheduleTrigger`). Triggers the chaser workflow daily.
    - *Configuration:* Cron expression: `0 10 * * 1-6` (Monday through Saturday at 10:00 AM).
    - *Input/Output:* Trigger node. Outputs execution timestamp.
  - **Set Chaser Config**
    - *Type and Technical Role:* Set (`n8n-nodes-base.set`). Defines configuration variables for the chaser and digest pipeline.
    - *Input/Output:* Input: Schedule trigger. Output: Configuration variables.
  - **Get All Requests1**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Retrieves all rows from the document requests tracking sheet.
    - *Configuration:* Operation: `get` (all rows). Always output data enabled.
    - *Input/Output:* Input: Configuration variables. Output: Array of all tracking rows.
  - **Build Team Digest1**
    - *Type and Technical Role:* Code (`n8n-nodes-base.code`). Aggregates metrics for open, pending, overdue, and recently completed requests, generating an HTML team digest email.
    - *Input/Output:* Input: All request rows. Output: Email object containing team recipient, subject, and HTML body.
  - **Email Team Digest1**
    - *Type and Technical Role:* Gmail (`n8n-nodes-base.gmail`). Sends the daily management digest to the team email address.
    - *Configuration:* Recipient mapped to team email.
    - *Input/Output:* Input: Team digest object. Output: Sent email confirmation.

---

#### Block 1.7: AI Reminder Drafting
- **Overview:** Filters requests due for a follow-up, calculates urgency metrics, and leverages OpenAI GPT-4o-mini to draft context-appropriate reminder email copy.
- **Nodes Involved:**
  - `Plan Due Reminders1` (Code)
  - `AI Write Reminder Email` (OpenAI)
  - `Compose Reminder Messages` (Code)

- **Node Details:**
  - **Plan Due Reminders1**
    - *Type and Technical Role:* Code (`n8n-nodes-base.code`). Identifies requests whose next reminder date has passed, checks reminder counts against maximum limits, and calculates tone (`friendly`, `firm`, `urgent`) based on due dates.
    - *Input/Output:* Input: All request rows. Output: Filtered array of planning objects for items requiring follow-up.
  - **AI Write Reminder Email**
    - *Type and Technical Role:* OpenAI (`@n8n/n8n-nodes-langchain.openAi`). Generates personalized reminder email copy using GPT-4o-mini.
    - *Configuration:* Model: `gpt-4o-mini`, Temperature: `0.5`, JSON output enforced. System prompt enforces professional Indian business English tone rules based on proximity to due dates.
    - *Input/Output:* Input: Reminder planning items. Output: AI-generated JSON containing email subject and paragraphs.
  - **Compose Reminder Messages**
    - *Type and Technical Role:* Code (`n8n-nodes-base.code`). Merges AI-drafted text with static UI components, checklist tables, secure upload buttons, WhatsApp payloads, tracker updates, and escalation triggers.
    - *Input/Output:* Input: AI generation output and planning items. Output: Formatted email bodies, WhatsApp payloads, tracker update objects, audit logs, and CA escalation payloads.

---

#### Block 1.8: Send, Track & Escalate
- **Overview:** Executes multi-channel reminder dispatch via WhatsApp and Gmail, updates tracker states, logs activity, and escalates unresponsive requests to assigned CAs.
- **Nodes Involved:**
  - `Send WhatsApp Reminder` (HTTP Request)
  - `Send Email Reminder` (Gmail)
  - `Update Reminder Tracker1` (Google Sheets)
  - `log Reminder Activity` (Google Sheets)
  - `Needs Escalation?1` (If)
  - `Escalate To Assigned CA1` (Gmail)

- **Node Details:**
  - **Send WhatsApp Reminder**
    - *Type and Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`). Sends the WhatsApp reminder template.
    - *Configuration:* Method: `POST`, authentication via Header Auth. Continues on regular output.
    - *Input/Output:* Input: WhatsApp payload. Output: API response.
  - **Send Email Reminder**
    - *Type and Technical Role:* Gmail (`n8n-nodes-base.gmail`). Sends the AI-drafted email reminder to the client.
    - *Configuration:* Recipient mapped to client email.
    - *Input/Output:* Input: Reminder email content. Output: Sent email confirmation.
  - **Update Reminder Tracker1**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Updates reminder counters, last reminder timestamps, next reminder schedules, status flags, and pending document lists in Google Sheets.
    - *Configuration:* Operation: `update`. Matching column: `request_id`.
    - *Input/Output:* Input: Tracker update object. Output: Sheet update confirmation.
  - **Log Reminder Activity**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Logs the reminder event in the activity sheet.
    - *Configuration:* Operation: `append`. Sheet name mapped to `activity_sheet`.
    - *Input/Output:* Input: Log object. Output: Appended row confirmation.
  - **Needs Escalation?1**
    - *Type and Technical Role:* If (`n8n-nodes-base.if`). Evaluates whether the request requires escalation to the assigned CA.
    - *Configuration:* Condition: `escalate` equals boolean `true`.
    - *Input/Output:* Input: Reminder composition output. Output: Routes execution to CA escalation email if true.
  - **Escalate To Assigned CA1**
    - *Type and Technical Role:* Gmail (`n8n-nodes-base.gmail`). Emails the assigned CA with client contact details, folder links, and pending document lists for direct phone follow-up.
    - *Configuration:* Recipient mapped to `ca_email`.
    - *Input/Output:* Input: Escalation HTML content. Output: Sent email confirmation.

---

#### Block 1.9: Live Status Board & Security
- **Overview:** Serves a secure, password-protected real-time status board web page via webhook, rendering request progress, overdue metrics, and search filters.
- **Nodes Involved:**
  - `Status Board Page Request` (Webhook)
  - `Set Board Config` (Set)
  - `Get Board Data1` (Google Sheets)
  - `Render Status Board HTML` (Code)
  - `Return Board Page` (Respond to Webhook)

- **Node Details:**
  - **Status Board Page Request**
    - *Type and Technical Role:* Webhook (`n8n-nodes-base.webhook`). Receives browser requests for the status board.
    - *Configuration:* Path set to `ca-status-board`, response mode set to `Response Node`.
    - *Input/Output:* Trigger node. Outputs incoming HTTP query parameters.
  - **Set Board Config**
    - *Type and Technical Role:* Set (`n8n-nodes-base.set`). Defines security keys and firm metadata for status board validation.
    - *Input/Output:* Input: Webhook trigger. Output: Configuration variables.
  - **Get Board Data1**
    - *Type and Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Retrieves all document request rows to populate the status board.
    - *Configuration:* Operation: `get` (all rows). Execute once enabled. Always output data enabled.
    - *Input/Output:* Input: Configuration variables. Output: Array of all tracking rows.
  - **Render Status Board HTML**
    - *Type and Technical Role:* Code (`n8n-nodes-base.code`). Validates security query parameters (`?key=`), returns a 401 Unauthorized page upon key mismatch, or constructs a responsive HTML/CSS/JS dashboard displaying interactive summary metric tiles, search filtering, progress bars, and status badges.
    - *Input/Output:* Input: Webhook query parameters and tracking rows. Output: HTML string and HTTP response code (`200` or `401`).
  - **Return Board Page**
    - *Type and Technical Role:* Respond to Webhook (`n8n-nodes-base.respondToWebhook`). Returns the rendered HTML page to the browser.
    - *Configuration:* Response code mapped from input, response headers set to `Content-Type: text/html; charset=utf-8`, response body mapped to HTML output.
    - *Input/Output:* Input: Rendered HTML object. Output: HTTP response sent to browser.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## 📁 CA Firm Document Chaser – WhatsApp & Email Follow-ups... |
| Sticky Note – Request Intake | `n8n-nodes-base.stickyNote` | Intake documentation grouping | None | None | ## 📝 Request Intake<br>Staff raise a request through the form... |
| Sticky Note – Folder, Tracker & First Message | `n8n-nodes-base.stickyNote` | Intake processing group | None | None | ## 📂 Folder, Tracker & First Message<br>Each request gets its own Drive folder... |
| Sticky Note – Client Upload Portal | `n8n-nodes-base.stickyNote` | Upload portal group | None | None | ## 📤 Client Upload Portal<br>The client opens their link... |
| Sticky Note – File & Update Checklist | `n8n-nodes-base.stickyNote` | File processing group | None | None | ## 🗃️ File & Update Checklist<br>Every attachment is saved... |
| Sticky Note – Receipts & Ready to File | `n8n-nodes-base.stickyNote` | Receipt and completion group | None | None | ## ✅ Receipts & Ready to File<br>The client gets a receipt... |
| Sticky Note – Daily Run & Team Digest | `n8n-nodes-base.stickyNote` | Scheduler and digest group | None | None | ## ⏰ Daily Run & Team Digest<br>At 10 AM Monday to Saturday... |
| Sticky Note – AI Reminder Drafting | `n8n-nodes-base.stickyNote` | AI processing group | None | None | ## 🧠 AI Reminder Drafting<br>Picks requests due a nudge... |
| Sticky Note – Send, Track & Escalate | `n8n-nodes-base.stickyNote` | Follow-up and escalation group | None | None | ## 📣 Send, Track & Escalate<br>Reminders go out on WhatsApp... |
| Sticky Note – Live Status Board | `n8n-nodes-base.stickyNote` | Dashboard documentation | None | None | ## 📊 Live Status Board<br>Open the webhook URL with `?key=`... |
| Sticky Note – Credentials & Security | `n8n-nodes-base.stickyNote` | Security guidance | None | None | ## 🔐 Credentials & Security<br>Use OAuth2 for Google and Gmail... |
| Send WhatsApp Document Request | `n8n-nodes-base.httpRequest` | Sends initial WhatsApp template message | Add Request to Tracker Sheet | None | 📂 Folder, Tracker & First Message |
| Receive Request via Webhook | `n8n-nodes-base.webhook` | Ingests document requests via API POST | None | Set Intake Config | 📝 Request Intake |
| Set Intake Config | `n8n-nodes-base.set` | Sets configuration variables for intake | Receive Request via Webhook, New Request Form (Staff)1 | Normalize Request Data | 📝 Request Intake |
| Normalize Request Data | `n8n-nodes-base.code` | Normalizes request payloads and metadata | Set Intake Config | Create Client Drive Folder | 📝 Request Intake |
| Create Client Drive Folder | `n8n-nodes-base.googleDrive` | Creates client folder in Google Drive | Normalize Request Data | Build Request Record & Messages | 📂 Folder, Tracker & First Message |
| Build Request Record & Messages | `n8n-nodes-base.code` | Builds tracker rows and email/WhatsApp payloads | Create Client Drive Folder | Add Request to Tracker Sheet | 📂 Folder, Tracker & First Message |
| Add Request to Tracker Sheet | `n8n-nodes-base.googleSheets` | Appends request record to Google Sheet | Build Request Record & Messages | Send WhatsApp Document Request, Email Document Request, Log Request Created1 | 📂 Folder, Tracker & First Message |
| Email Document Request | `n8n-nodes-base.gmail` | Sends initial email with checklist and upload link | Add Request to Tracker Sheet | None | 📂 Folder, Tracker & First Message |
| Client Upload Portal Form | `n8n-nodes-base.formTrigger` | Serves client document upload form | None | Set Upload Config | 📤 Client Upload Portal |
| Set Upload Config | `n8n-nodes-base.set` | Sets configuration variables for uploads | Client Upload Portal Form | Find Request in Tracker | 📤 Client Upload Portal |
| Find Request in Tracker | `n8n-nodes-base.googleSheets` | Looks up request record in Google Sheet | Set Upload Config | Request Found?1 | 📤 Client Upload Portal |
| Save Files to Client Folder | `n8n-nodes-base.googleDrive` | Uploads client files to Google Drive folder | Split Uploaded Files1 | Compute Checklist Status1 | 🗃️ File & Update Checklist |
| Update Tracker Checklist | `n8n-nodes-base.googleSheets` | Updates checklist status in Google Sheet | Compute Checklist Status1 | Email Upload Receipt1, Send WhatsApp Upload Receipt, All Documents In?1, Log Upload Activity | 🗃️ File & Update Checklist |
| Send WhatsApp Upload Receipt | `n8n-nodes-base.httpRequest` | Sends upload receipt via WhatsApp | Update Tracker Checklist | None | ✅ Receipts & Ready to File |
| Log Upload Activity | `n8n-nodes-base.googleSheets` | Logs upload activity in Google Sheet | Update Tracker Checklist | None | ✅ Receipts & Ready to File |
| Email CA: Ready to File | `n8n-nodes-base.gmail` | Emails assigned CA when all docs are received | All Documents In?1 | None | ✅ Receipts & Ready to File |
| Daily 10 AM Mon-Sat | `n8n-nodes-base.scheduleTrigger` | Triggers daily chaser routine | None | Set Chaser Config | ⏰ Daily Run & Team Digest |
| Set Chaser Config | `n8n-nodes-base.set` | Sets configuration variables for chaser routine | Daily 10 AM Mon-Sat | Get All Requests1 | ⏰ Daily Run & Team Digest |
| AI Write Reminder Email | `@n8n/n8n-nodes-langchain.openAi` | Drafts reminder emails using GPT-4o-mini | Plan Due Reminders1 | Compose Reminder Messages | 🧠 AI Reminder Drafting |
| Compose Reminder Messages | `n8n-nodes-base.code` | Formats reminder emails, WhatsApp payloads, and logs | AI Write Reminder Email | Send WhatsApp Reminder, Send Email Reminder, Update Reminder Tracker1, Log Reminder Activity, Needs Escalation?1 | 📣 Send, Track & Escalate |
| Send WhatsApp Reminder | `n8n-nodes-base.httpRequest` | Sends WhatsApp reminder message | Compose Reminder Messages | None | 📣 Send, Track & Escalate |
| Send Email Reminder | `n8n-nodes-base.gmail` | Sends email reminder to client | Compose Reminder Messages | None | 📣 Send, Track & Escalate |
| Log Reminder Activity | `n8n-nodes-base.googleSheets` | Logs reminder activity in Google Sheet | Compose Reminder Messages | None | 📣 Send, Track & Escalate |
| Status Board Page Request | `n8n-nodes-base.webhook` | Receives status board browser requests | None | Set Board Config | 📊 Live Status Board |
| Set Board Config | `n8n-nodes-base.set` | Sets configuration variables for status board | Status Board Page Request | Get Board Data1 | 📊 Live Status Board |
| Render Status Board HTML | `n8n-nodes-base.code` | Generates HTML dashboard or 401 error page | Get Board Data1 | Return Board Page | 📊 Live Status Board |
| Return Board Page | `n8n-nodes-base.respondToWebhook` | Returns status board HTML to browser | Render Status Board HTML | None | 📊 Live Status Board |
| New Request Form (Staff)1 | `n8n-nodes-base.formTrigger` | Serves internal staff request form | None | Set Intake Config | 📝 Request Intake |
| Log Request Created1 | `n8n-nodes-base.googleSheets` | Logs request creation event in Google Sheet | Add Request to Tracker Sheet | None | 📂 Folder, Tracker & First Message |
| Email Upload Receipt1 | `n8n-nodes-base.gmail` | Emails upload receipt to client | Update Tracker Checklist | None | ✅ Receipts & Ready to File |
| Request Found?1 | `n8n-nodes-base.if` | Validates if request ID exists in tracker | Find Request in Tracker | Split Uploaded Files1 | 📤 Client Upload Portal |
| Split Uploaded Files1 | `n8n-nodes-base.code` | Splits form attachments into individual items | Request Found?1 | Save Files to Client Folder | 🗃️ File & Update Checklist |
| Compute Checklist Status1 | `n8n-nodes-base.code` | Computes received/pending checklist status | Save Files to Client Folder | Update Tracker Checklist | 🗃️ File & Update Checklist |
| All Documents In?1 | `n8n-nodes-base.if` | Checks if all checklist documents are received | Update Tracker Checklist | Email CA: Ready to File | ✅ Receipts & Ready to File |
| Get All Requests1 | `n8n-nodes-base.googleSheets` | Retrieves all requests from Google Sheet | Set Chaser Config | Build Team Digest1, Plan Due Reminders1 | ⏰ Daily Run & Team Digest |
| Build Team Digest1 | `n8n-nodes-base.code` | Builds team digest email summary | Get All Requests1 | Email Team Digest1 | ⏰ Daily Run & Team Digest |
| Email Team Digest1 | `n8n-nodes-base.gmail` | Emails daily digest summary to team | Build Team Digest1 | None | ⏰ Daily Run & Team Digest |
| Plan Due Reminders1 | `n8n-nodes-base.code` | Plans due reminders and calculates urgency tone | Get All Requests1 | AI Write Reminder Email | 🧠 AI Reminder Drafting |
| Update Reminder Tracker1 | `n8n-nodes-base.googleSheets` | Updates reminder tracking metrics in Google Sheet | Compose Reminder Messages | None | 📣 Send, Track & Escalate |
| Get Board Data1 | `n8n-nodes-base.googleSheets` | Retrieves all request data for status board | Set Board Config | Render Status Board HTML | 📊 Live Status Board |
| Needs Escalation?1 | `n8n-nodes-base.if` | Evaluates if request requires CA escalation | Compose Reminder Messages | Escalate To Assigned CA1 | 📣 Send, Track & Escalate |
| Escalate To Assigned CA1 | `n8n-nodes-base.gmail` | Emails assigned CA for client follow-up | Needs Escalation?1 | None | 📣 Send, Track & Escalate |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to rebuild the workflow manually in n8n:

1. **Create Supporting Infrastructure:**
   - Set up a Google Sheet named with two tabs: `Document Requests` and `Activity Log`. Define columns: `request_id`, `created_at`, `client_name`, `client_email`, `client_whatsapp`, `compliance_type`, `period`, `due_date`, `documents_required`, `documents_received`, `pending_documents`, `status`, `reminders_sent`, `last_reminder_at`, `next_reminder_at`, `last_upload_at`, `completed_at`, `upload_link`, `drive_folder_id`, `drive_folder_link`, `assigned_ca_email`, and `notes`.
   - Create a root folder in Google Drive for client document storage.
   - Configure Meta WhatsApp Cloud API templates (`document_request`, `document_reminder`, `documents_received`).

2. **Build Intake Triggers & Configuration:**
   - Create a **Form Trigger** (`New Request Form (Staff)1`) at path `ca-new-document-request` collecting client details, compliance type, period, checklist, due date, assigned CA email, and notes.
   - Create a **Webhook** (`Receive Request via Webhook`) at path `ca-document-request` (POST).
   - Create a **Set** node (`Set Intake Config`) connected to both intake triggers, assigning firm details, sheet IDs, Drive folder ID, n8n base URL (`https://your-n8n-instance.com`), WhatsApp phone number ID, API version (`v21.0`), template names, and a secure `board_key`.
   - Create a **Code** node (`Normalize Request Data`) to normalize incoming payloads, format phone numbers with country codes (`91`), and generate unique request IDs.

3. **Build Folder, Tracker & Initial Outreach:**
   - Add a **Google Drive** node (`Create Client Drive Folder`) configured to create folders dynamically inside the root folder using `{{ $json.folder_name }}`.
   - Add a **Code** node (`Build Request Record & Messages`) to structure database rows, HTML email bodies, and WhatsApp payloads.
   - Add a **Google Sheets** node (`Add Request to Tracker Sheet`) to append rows to the `Document Requests` tab.
   - Connect three parallel output branches from the tracker sheet:
     - An **HTTP Request** node (`Send WhatsApp Document Request`) configured with Generic Header Auth (Bearer token) pointing to the Meta Graph API endpoint.
     - A **Gmail** node (`Email Document Request`) to send the initial checklist email.
     - A **Google Sheets** node (`Log Request Created1`) to append an event entry to the `Activity Log` tab.

4. **Build Client Upload Portal:**
   - Create a **Form Trigger** (`Client Upload Portal Form`) at path `ca-document-upload` with a hidden `request_id` field, document checkboxes, file upload field, and message textarea.
   - Add a **Set** node (`Set Upload Config`) with matching configuration constants.
   - Add a **Google Sheets** node (`Find Request in Tracker`) to lookup rows where `request_id` matches the form submission.
   - Add an **If** node (`Request Found?1`) to verify that the request ID exists.
   - Add a **Code** node (`Split Uploaded Files1`) to split binary file attachments into individual items.
   - Add a **Google Drive** node (`Save Files to Client Folder`) to upload files into the client's folder.
   - Add a **Code** node (`Compute Checklist Status1`) to calculate received/pending documents and build receipt payloads.
   - Add a **Google Sheets** node (`Update Tracker Checklist`) to update request status in the sheet (`Complete` or `Partial`).
   - Connect parallel outputs from the tracker update:
     - **HTTP Request** (`Send WhatsApp Upload Receipt`)
     - **Gmail** (`Email Upload Receipt1`)
     - **Google Sheets** (`Log Upload Activity`)
     - An **If** node (`All Documents In?1`) checking if `complete` is true, connected to a **Gmail** node (`Email CA: Ready to File`).

5. **Build Daily Chaser & AI Follow-ups:**
   - Create a **Schedule Trigger** (`Daily 10 AM Mon-Sat`) set to cron expression `0 10 * * 1-6`.
   - Add a **Set** node (`Set Chaser Config`) and a **Google Sheets** node (`Get All Requests1`) to retrieve all rows.
   - Branch 1 (Team Digest): Connect to a **Code** node (`Build Team Digest1`) and a **Gmail** node (`Email Team Digest1`).
   - Branch 2 (AI Reminders): Connect to a **Code** node (`Plan Due Reminders1`), then an **OpenAI** node (`AI Write Reminder Email`) using model `gpt-4o-mini` with JSON output enabled.
   - Connect OpenAI output to a **Code** node (`Compose Reminder Messages`) to format reminder emails, WhatsApp messages, tracker payloads, and logs.
   - Branch outputs from the compose node:
     - **HTTP Request** (`Send WhatsApp Reminder`)
     - **Gmail** (`Send Email Reminder`)
     - **Google Sheets** (`Update Reminder Tracker1`)
     - **Google Sheets** (`Log Reminder Activity`)
     - An **If** node (`Needs Escalation?1`) connected to a **Gmail** node (`Escalate To Assigned CA1`).

6. **Build Live Status Board:**
   - Create a **Webhook** (`Status Board Page Request`) at path `ca-status-board` with response mode set to `Response Node`.
   - Add a **Set** node (`Set Board Config`) and a **Google Sheets** node (`Get Board Data1`).
   - Add a **Code** node (`Render Status Board HTML`) to validate query parameters (`?key=board_key`), return a 401 Unauthorized page upon mismatch, or render an interactive HTML dashboard.
   - Add a **Respond to Webhook** node (`Return Board Page`) returning `text/html; charset=utf-8` with the dynamic HTTP response code.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| CA Firm Document Chaser Workflow | Designed specifically for Chartered Accountants and accounting firms to automate client document collection during filing seasons. |
| Meta WhatsApp Cloud API Setup | Requires a verified Meta Business Account, configured WhatsApp phone number ID, and pre-approved message templates (`document_request`, `document_reminder`, `documents_received`). |
| OpenAI Model Requirement | Uses OpenAI GPT-4o-mini configured with JSON response formatting to draft natural-language reminder emails. |
| Security Warning (`board_key`) | Change the default `board_key` configuration value before activating the workflow, as the live status board displays sensitive client names, filing types, and pending document checklists. |