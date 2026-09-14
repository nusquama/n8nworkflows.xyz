Track customer email commitments with Gmail, OpenRouter and Google Sheets

https://n8nworkflows.xyz/workflows/track-customer-email-commitments-with-gmail--openrouter-and-google-sheets-17538


# Track customer email commitments with Gmail, OpenRouter and Google Sheets

### 1. Workflow Overview

The **Track customer commitments with Gmail, OpenRouter and Google Sheets** workflow automates the extraction and tracking of explicit business promises made to customers via outbound email. It scans sent messages, uses an LLM via OpenRouter to analyze and extract actionable commitments, records them in a Google Sheets register, and automates tracking and escalations through internal notifications.

The logic operates through two distinct entry points sharing a centralized configuration block:
- **1.1 Input Reception & Configuration:** Handles scheduled or manual triggers, setting the operational mode (`scan` vs. `reminder`) and injecting global environment variables.
- **1.2 Outbound Email Scanning & AI Processing:** Scans Gmail Sent Mail, filters out internal or empty recipients, formats payloads, requests structured JSON extractions from OpenRouter, applies processed labels, and logs valid commitments.
- **1.3 Commitment Register & Human Review:** Manages data persistence within Google Sheets and routes records based on confidence thresholds (notifying owners for standard commitments or reviewers for low-confidence data).
- **1.4 Daily Promise Control & Escalation:** Executes on a weekday schedule to query open commitments, aggregate upcoming/overdue tasks per owner, and escalate unresolved items to management.

---

### 2. Block-by-Block Analysis

#### Block 1: Initialization and Configuration
- **Overview:** Establishes triggers, sets the execution path (`scan` or `reminder`), and loads global operational variables used across downstream nodes.
- **Nodes Involved:** `Manual Scan Test`, `Hourly Sent Email Scan`, `Daily Promise Control`, `Set Scan Mode`, `Set Reminder Mode`, `Set Template Fields`, `Is Daily Reminder Run`.
- **Node Details:**
  - `Manual Scan Test` / `Hourly Sent Email Scan` / `Daily Promise Control`
    - *Type and Technical Role:* Trigger nodes (Manual Trigger, Schedule Trigger). Initiates manual tests, hourly scans, or weekday morning control loops.
    - *Configuration Choices:* Cron expressions (`5 * * * *` for hourly scans; `0 8 * * 1-5` for weekday mornings at 8:00 AM).
    - *Input/Output:* No inputs; outputs execution signals to Mode setting nodes.
    - *Edge Cases:* Timezone misalignments if n8n instance time differs from expected operational hours.
  - `Set Scan Mode` / `Set Reminder Mode`
    - *Type and Technical Role:* Set (Data transformation). Assigns string values (`scan` or `reminder`) to the `executionMode` property.
    - *Input/Output:* Input from triggers; outputs to `Set Template Fields`.
  - `Set Template Fields`
    - *Type and Technical Role:* Set (Configuration provider). Establishes centralized variables (internal domains, Google Sheets ID, recipient emails, AI model identifiers, thresholds, timezones).
    - *Key Expressions or Variables:* Defines parameters like `internalDomains`, `commitmentSheetId`, `minimumConfidence`, and `aiModel`.
    - *Input/Output:* Input from mode setters; output to `Is Daily Reminder Run`.
  - `Is Daily Reminder Run`
    - *Type and Technical Role:* If (Conditional router). Evaluates `executionMode` to split processing paths.
    - *Key Expressions:* `={{ $json.executionMode }}` equals `reminder`.
    - *Input/Output:* Input from template fields; routes to either Gmail scanning (`scan`) or Google Sheets reading (`reminder`).

#### Block 2: Email Ingestion and AI Extraction
- **Overview:** Retrieves sent emails, filters out internal correspondence, structures the message payload into a model-neutral prompt, executes an OpenRouter API request, and normalizes the JSON output.
- **Nodes Involved:** `Search Sent Customer Emails`, `Prepare Sent Customer Messages`, `Build Commitment AI Request`, `Extract Commitments with AI`, `Normalize Commitment Extraction`.
- **Node Details:**
  - `Search Sent Customer Emails`
    - *Type and Technical Role:* Gmail (API Integration). Queries sent messages.
    - *Configuration Choices:* Uses dynamic search queries (`q`) defined in template variables.
    - *Input/Output:* Input from `Is Daily Reminder Run`; outputs raw message objects.
    - *Edge Cases:* Authentication expiration or strict Gmail API rate limits.
  - `Prepare Sent Customer Messages`
    - *Type and Technical Role:* Code (JavaScript data manipulation). Parses email headers, extracts To/Cc addresses, filters out internal domains/no-reply accounts, and validates body length.
    - *Input/Output:* Input from Gmail search; outputs cleaned customer-facing message objects containing IDs, snippets, and source links.
  - `Build Commitment AI Request`
    - *Type and Technical Role:* Code (JavaScript data manipulation). Constructs the LLM instruction set, JSON response format enforcement rules, and context payload.
    - *Input/Output:* Input from preparation code; outputs structured AI request bodies.
  - `Extract Commitments with AI`
    - *Type and Technical Role:* HTTP Request (API Integration). Calls OpenRouter's chat completions endpoint.
    - *Configuration Choices:* POST request to `https://openrouter.ai/api/v1/chat/completions` using Generic HTTP Header Authentication. Enforces JSON object response formatting and a 60-second timeout.
    - *Input/Output:* Input from AI request builder; outputs raw model responses.
    - *Edge Cases:* API timeouts, model quota limits, or invalid JSON return structures.
  - `Normalize Commitment Extraction`
    - *Type and Technical Role:* Code (JavaScript data manipulation). Parses LLM JSON responses, handles markdown formatting artifacts, matches extractions back to source messages, and assigns status categories (`Ignored`, `Needs review`, `Open`).
    - *Input/Output:* Input from HTTP request; outputs normalized commitment objects.

#### Block 3: Persistence and Routing
- **Overview:** Applies tracking labels to scanned emails, logs verified commitments to Google Sheets, and routes items needing attention to the appropriate internal recipient.
- **Nodes Involved:** `Label Assessed Gmail Messages`, `Is Customer Commitment`, `Prepare Commitment Register Row`, `Append Commitment Register`, `Restore Commitment Context`, `Needs Human Review`, `Send Owner Commitment Notice`, `Send Reviewer Notice`.
- **Node Details:**
  - `Label Assessed Gmail Messages`
    - *Type and Technical Role:* Gmail (API Integration). Applies tracking labels to prevent duplicate processing.
    - *Configuration Choices:* Adds the label defined in `processedGmailLabel`. Uses error continuation (`onError: continueRegularOutput`).
  - `Is Customer Commitment` & `Needs Human Review`
    - *Type and Technical Role:* If (Conditional routers). Filters valid commitments and distinguishes standard entries from those requiring manual verification.
  - `Prepare Commitment Register Row` & `Restore Commitment Context`
    - *Type and Technical Role:* Code (JavaScript data manipulation). Maps object properties to match Google Sheets header rows and restores state context after append operations.
  - `Append Commitment Register`
    - *Type and Technical Role:* Google Sheets (API Integration). Appends rows to the designated spreadsheet tab.
    - *Configuration Choices:* Auto-maps input schema using dynamic spreadsheet ID and tab name variables.
  - `Send Owner Commitment Notice` & `Send Reviewer Notice`
    - *Type and Technical Role:* Gmail (API Integration). Dispatches internal notification emails.
    - *Configuration Choices:* Sends plaintext emails using custom sender names without automated tracking attributions.

#### Block 4: Daily Control and Escalation
- **Overview:** Reads open or rescheduled commitments from Google Sheets on weekdays, evaluates due dates against configured reminder windows, groups actions per owner, and escalates overdue items to management.
- **Nodes Involved:** `Read Commitment Register`, `Prepare Daily Promise Actions`, `Send Daily Promise Action List`.
- **Node Details:**
  - `Read Commitment Register`
    - *Type and Technical Role:* Google Sheets (API Integration). Reads all rows from the commitment register tab.
  - `Prepare Daily Promise Actions`
    - *Type and Technical Role:* Code (JavaScript data manipulation). Filters open/rescheduled entries, computes due timestamps, identifies overdue statuses against escalation delays, and groups action items by owner or manager email.
  - `Send Daily Promise Action List`
    - *Type and Technical Role:* Gmail (API Integration). Sends aggregated task lists and escalation notices to owners and managers.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note — Overview | n8n-nodes-base.stickyNote | Documentation | None | None | Customer Promise Control Tower<br><br>Turn promises made in customer emails into accountable work before they are missed.<br><br>This workflow scans **sent customer emails**, uses your chosen AI model to identify explicit commitments, and records the promise, owner, customer, due date, confidence, and source message in Google Sheets. A daily control run sends the owner a reminder for commitments due soon or overdue.<br><br>## What it protects<br><br>- Quotes, documents, callbacks, fixes, approvals, updates, samples, and delivery confirmations that someone said they would send or complete.<br>- Customer trust, renewals, revenue, and team accountability.<br><br>## Safety model<br><br>The workflow never sends a message to the customer. Low-confidence detections go to **Needs review**. Every record keeps its source-message ID, and Gmail receives a processed label so the same sent email is not assessed twice.<br><br>## Before activation<br><br>Configure the yellow Set Template Fields node, create the Google Sheets headers listed there, connect Gmail, Google Sheets, and an OpenRouter HTTP Header Auth credential, then test with one sent customer email. |
| Sticky Note — Configure first | n8n-nodes-base.stickyNote | Documentation | None | None | ## Configure before activation<br><br>Use **Set Template Fields** as the single place to set:<br><br>- Your company domain(s) and the Gmail search query.<br>- Google Sheets ID, Commitment Register tab, and recipient emails.<br>- Reminder window, escalation delay, and confidence threshold.<br>- The AI model available through your OpenRouter credential.<br><br>Create a Gmail label named `n8n-commitment-tracked` before running the scan. The workflow adds this label after it assesses each sent message, including messages that contain no commitment.<br><br>Use a dedicated inbox or a restricted Gmail query while testing. |
| Sticky Note — Scan sent customer email | n8n-nodes-base.stickyNote | Documentation | None | None | ## 1. Scan only external sent messages<br><br>An hourly schedule searches Gmail Sent Mail. Internal, automated, and empty messages are excluded before AI is used.<br><br>The Message ID is the idempotency key. Keep the processed-label filter in the Gmail query so already assessed emails are never scanned again. |
| Sticky Note — Extract commitments with AI | n8n-nodes-base.stickyNote | Documentation | None | None | ## 2. Extract only explicit commitments<br><br>The AI receives the sent message and returns structured JSON. It must identify an action, the exact promise, the customer, the owner, a due date when stated, and confidence.<br><br>Use any OpenRouter-supported model. The template is model-neutral and does not depend on a specific provider model. |
| Sticky Note — Create a durable register | n8n-nodes-base.stickyNote | Documentation | None | None | ## 3. Write a source-backed commitment record<br><br>Each qualifying message becomes one Google Sheets row. Records are marked **Open** when confident, or **Needs review** when a due date is unclear or confidence is below the threshold.<br><br>The source message ID and Gmail thread ID make every promise traceable. The owner is notified internally; the customer is never contacted by this workflow. |
| Sticky Note — Daily promise control | n8n-nodes-base.stickyNote | Documentation | None | None | ## 4. Remind before a promise breaks<br><br>Every weekday morning, the workflow reads open commitments. It sends the owner one concise action list for commitments due within the configured window and escalates overdue commitments to the manager.<br><br>To close or reschedule a promise, update its Status or Due Date directly in Google Sheets. |
| Sticky Note — Privacy and review | n8n-nodes-base.stickyNote | Documentation | None | None | ## Privacy and human control<br><br>Only grant access to the mailbox that should be monitored. Exclude personal, internal, and automated mail through the Gmail query and domain settings.<br><br>AI classification is advisory. Review low-confidence records. Do not use this workflow to automatically make, change, or send customer commitments. |
| Manual Scan Test | n8n-nodes-base.manualTrigger | Trigger workflow execution manually | None | Set Scan Mode | |
| Hourly Sent Email Scan | n8n-nodes-base.scheduleTrigger | Trigger hourly scans | None | Set Scan Mode | |
| Set Scan Mode | n8n-nodes-base.set | Sets execution mode to scan | Manual Scan Test, Hourly Sent Email Scan | Set Template Fields | |
| Daily Promise Control | n8n-nodes-base.scheduleTrigger | Trigger weekday morning control runs | None | Set Reminder Mode | |
| Set Reminder Mode | n8n-nodes-base.set | Sets execution mode to reminder | Daily Promise Control | Set Template Fields | |
| Set Template Fields | n8n-nodes-base.set | Defines global configuration variables | Set Scan Mode, Set Reminder Mode | Is Daily Reminder Run | |
| Is Daily Reminder Run | n8n-nodes-base.if | Routes execution based on mode | Set Template Fields | Read Commitment Register, Search Sent Customer Emails | |
| Search Sent Customer Emails | n8n-nodes-base.gmail | Retrieves messages from Gmail Sent Mail | Is Daily Reminder Run | Prepare Sent Customer Messages | |
| Prepare Sent Customer Messages | n8n-nodes-base.code | Filters out internal and empty messages | Search Sent Customer Emails | Build Commitment AI Request | |
| Build Commitment AI Request | n8n-nodes-base.code | Constructs OpenRouter API payload | Prepare Sent Customer Messages | Extract Commitments with AI | |
| Extract Commitments with AI | n8n-nodes-base.httpRequest | Calls OpenRouter chat completions API | Build Commitment AI Request | Normalize Commitment Extraction | |
| Normalize Commitment Extraction | n8n-nodes-base.code | Parses LLM response and standardizes fields | Extract Commitments with AI | Label Assessed Gmail Messages, Is Customer Commitment | |
| Label Assessed Gmail Messages | n8n-nodes-base.gmail | Applies tracking label to assessed emails | Normalize Commitment Extraction | None | |
| Is Customer Commitment | n8n-nodes-base.if | Filters out non-commitment messages | Normalize Commitment Extraction | Prepare Commitment Register Row | |
| Prepare Commitment Register Row | n8n-nodes-base.code | Formats data for Google Sheets row schema | Is Customer Commitment | Append Commitment Register | |
| Append Commitment Register | n8n-nodes-base.googleSheets | Appends commitment data to Google Sheets | Prepare Commitment Register Row | Restore Commitment Context | |
| Restore Commitment Context | n8n-nodes-base.code | Restores original context properties | Append Commitment Register | Needs Human Review | |
| Needs Human Review | n8n-nodes-base.if | Distinguishes normal records from review items | Restore Commitment Context | Send Reviewer Notice, Send Owner Commitment Notice | |
| Send Owner Commitment Notice | n8n-nodes-base.gmail | Emails notification to commitment owner | Needs Human Review | None | |
| Send Reviewer Notice | n8n-nodes-base.gmail | Emails notification to reviewer | Needs Human Review | None | |
| Read Commitment Register | n8n-nodes-base.googleSheets | Reads rows from Google Sheets | Is Daily Reminder Run | Prepare Daily Promise Actions | |
| Prepare Daily Promise Actions | n8n-nodes-base.code | Aggregates tasks per owner and evaluates escalations | Read Commitment Register | Send Daily Promise Action List | |
| Send Daily Promise Action List | n8n-nodes-base.gmail | Emails due-soon or escalated action lists | Prepare Daily Promise Actions | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create Triggers and Mode Setters:**
   - Create a **Manual Trigger** (`Manual Scan Test`), a **Schedule Trigger** (`Hourly Sent Email Scan` set to cron `5 * * * *`), and a **Schedule Trigger** (`Daily Promise Control` set to cron `0 8 * * 1-5`).
   - Create two **Set** nodes (`Set Scan Mode` and `Set Reminder Mode`) assigning `executionMode` to `"scan"` and `"reminder"` respectively. Connect the manual and hourly triggers to `Set Scan Mode`, and the weekday trigger to `Set Reminder Mode`.
2. **Configure Global Variables:**
   - Add a **Set** node (`Set Template Fields`) connected to both mode setters.
   - Define fields: `internalDomains` (string), `gmailSearchQuery` (string: `in:sent newer_than:2h -label:n8n-commitment-tracked`), `processedGmailLabel` (string: `n8n-commitment-tracked`), `commitmentSheetId` (string), `commitmentsSheetTab` (string: `Commitments`), `defaultOwnerEmail`, `reviewerEmail`, `managerEmail`, `aiModel` (string: e.g., `openai/gpt-4.1-mini`), `minimumConfidence` (number: `0.8`), `reminderWindowHours` (number: `24`), `escalateAfterDays` (number: `1`), and `timezone` (string: `America/Toronto`).
3. **Configure Branching (Scan vs. Reminder):**
   - Add an **If** node (`Is Daily Reminder Run`) connected to `Set Template Fields`. Check if `{{ $json.executionMode }}` equals `"reminder"`.
4. **Build the Scan and AI Pipeline (False Branch):**
   - **Gmail Node (`Search Sent Customer Emails`):** Resource `message`, operation `getAll`, filter query `={{ $json.gmailSearchQuery }}`. Requires Gmail OAuth2 credentials.
   - **Code Node (`Prepare Sent Customer Messages`):** Insert JavaScript to parse headers, filter internal domain recipients, and validate body length.
   - **Code Node (`Build Commitment AI Request`):** Insert JavaScript to format prompt rules and build the OpenRouter payload.
   - **HTTP Request Node (`Extract Commitments with AI`):** POST to `https://openrouter.ai/api/v1/chat/completions`. Set Authentication to `Generic Credential Type` -> `HTTP Header Auth` (Header Name: `Authorization`, Value: `Bearer <your-openrouter-key>`). Set body type to JSON.
   - **Code Node (`Normalize Commitment Extraction`):** Insert JavaScript to parse LLM output, handle markdown wrapping, and evaluate confidence thresholds.
   - **Gmail Node (`Label Assessed Gmail Messages`):** Resource `message`, operation `addLabels`, Message ID `={{ $json.messageId }}`, Label IDs `={{ $json.processedGmailLabel }}`. Enable `Continue On Fail`.
   - **If Node (`Is Customer Commitment`):** Check if `{{ $json.containsCommitment }}` equals `true`.
5. **Build the Persistence and Review Pipeline:**
   - **Code Node (`Prepare Commitment Register Row`):** Format output to match spreadsheet columns (`Commitment ID`, `Status`, `Due Date`, `Owner Email`, `Customer Email`, etc.).
   - **Google Sheets Node (`Append Commitment Register`):** Operation `append`. Select Document ID and Sheet Name using expressions referencing `Set Template Fields`. Requires Google Sheets OAuth2 credentials.
   - **Code Node (`Restore Commitment Context`):** Match returned rows back to original items by `Commitment ID`.
   - **If Node (`Needs Human Review`):** Check if `{{ $json["Status"] }}` equals `"Needs review"`.
   - **Gmail Nodes (`Send Reviewer Notice` & `Send Owner Commitment Notice`):** Configure recipient addresses, subject lines, and text bodies dynamically based on workflow status. Enable `Continue On Fail`.
6. **Build the Daily Control Pipeline (True Branch):**
   - **Google Sheets Node (`Read Commitment Register`):** Operation `get` (or getAll) using Document ID and Sheet Name from configuration variables.
   - **Code Node (`Prepare Daily Promise Actions`):** Insert JavaScript to group open/rescheduled items by owner, check due windows, and flag escalation requirements.
   - **Gmail Node (`Send Daily Promise Action List`):** Send aggregated reminders and escalation notices to recipients. Enable `Continue On Fail`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure the Gmail label `n8n-commitment-tracked` exists prior to running the workflow to prevent API errors during label application. | Prerequisites & Setup |
| Google Sheets headers must match the exact column mappings generated in the `Prepare Commitment Register Row` node (including `Commitment ID`, `Status`, `Due Date`, `Owner Email`, etc.). | Data Schema Configuration |