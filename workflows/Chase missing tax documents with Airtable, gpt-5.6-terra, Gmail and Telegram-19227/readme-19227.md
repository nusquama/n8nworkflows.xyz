Chase missing tax documents with Airtable, gpt-5.6-terra, Gmail and Telegram

https://n8nworkflows.xyz/workflows/chase-missing-tax-documents-with-airtable--gpt-5-6-terra--gmail-and-telegram-19227


# Chase missing tax documents with Airtable, gpt-5.6-terra, Gmail and Telegram

### 1. Workflow Overview

This workflow automates the process of tracking tax documents for an accounting firm, chasing missing documents from clients, escalating silent cases to the firm owner, and handling incoming document submissions via webhook. 

The logic is grouped into the following functional blocks:
- **1.1 Daily Chase Initialization:** Triggers every weekday morning, establishes firm-level operational parameters, and queries the Airtable client checklist.
- **1.2 Status Assessment & Routing:** Evaluates each client’s required versus received documents, calculates the duration since the last contact, and directs the flow to reminders, escalations, all-clear notices, or suppression.
- **1.3 AI Reminder Generation & Dispatch:** Drafts a personalized reminder via OpenAI (gpt-4o-mini), normalizes the output, dispatches the email via Gmail, updates Airtable records, and logs the transaction.
- **1.4 Owner Escalation:** Alerts the firm owner via Telegram and Gmail when a client has been unresponsive for a week, updates Airtable, and logs the escalation.
- **1.5 All-Clear Notification:** Sends a summary notification via Telegram and records an all-clear status in the log table when no clients are missing documents.
- **1.6 Webhook Document Intake & Validation:** Receives incoming document submissions, normalizes data, and validates required fields.
- **1.7 Client Matching & Processing:** Looks up client records in Airtable, validates if the submitted document is required, updates checklist statuses, logs the intake, and confirms receipt via Telegram.
- **1.8 Error Handling & Exception Management:** Catches invalid payloads or unmatched client/document submissions, triggering Telegram alerts and internal firm error notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Chase Initialization
- **Overview:** Initiates the daily tax document review process on a scheduled cron basis, configures firm identification parameters, and fetches all client records from Airtable.
- **Nodes Involved:** 
  - `When Workday Begins`
  - `Set Firm Chase Settings`
  - `Fetch Client Checklist`
- **Node Details:**
  - **When Workday Begins**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Cron expression set to run at 9:00 AM every Monday through Friday (`0 9 * * 1-5`).
    - *Outputs:* Triggers `Set Firm Chase Settings`.
    - *Edge Cases / Failure Types:* Execution skipped if n8n instance is offline.
  - **Set Firm Chase Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Assigns static variables: `firm_name` ("Lumen Tax and Accounting"), `owner_email`, and `firm_email`. Passes through existing input fields.
    - *Inputs:* `When Workday Begins` -> *Outputs:* `Fetch Client Checklist`.
  - **Fetch Client Checklist**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Read)
    - *Configuration:* Searches the `Clients` table in Airtable base `appTGcnO2nvOQC86I`.
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Set Firm Chase Settings` -> *Outputs:* `Evaluate Client Status`.
    - *Edge Cases / Failure Types:* Airtable API rate limits or authentication revocation.

---

#### 2.2 Status Assessment & Routing
- **Overview:** Executes custom JavaScript logic to evaluate individual client document statuses, determines elapsed time since the last reminder, and switches the execution path based on required actions.
- **Nodes Involved:**
  - `Evaluate Client Status`
  - `Route by Client Status`
- **Node Details:**
  - **Evaluate Client Status**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation / Logic)
    - *Configuration:* JavaScript execution comparing `required_docs` and `received_docs`. Determines if action is required based on missing documents and days elapsed since `last_reminded` (`remind`, `escalate`, `none`, or `all-clear`).
    - *Inputs:* `Fetch Client Checklist` -> *Outputs:* `Route by Client Status`.
  - **Route by Client Status**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Flow Control)
    - *Configuration:* Routes execution based on `{{ $json.action }}` matching strings: `remind`, `escalate`, or `all-clear`.
    - *Inputs:* `Evaluate Client Status` -> *Outputs:* `Generate Reminder Email`, `Notify Owner via Telegram`, or `Notify All Clear on Telegram`.

---

#### 2.3 AI Reminder Generation & Dispatch
- **Overview:** Utilizes an OpenAI agent and chat model to draft a warm, concise client reminder email, parses the output, sends the email, and logs the activity in Airtable.
- **Nodes Involved:**
  - `OpenAI Reminder Model`
  - `Generate Reminder Email`
  - `Parse Draft Email`
  - `Send Client Reminder Email`
  - `Update Reminder Date`
  - `Log Reminder Activity`
- **Node Details:**
  - **OpenAI Reminder Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Sub-node)
    - *Configuration:* Uses model `gpt-4o-mini` with temperature set to `0.4`.
    - *Credentials:* OpenAI API (`jonathan`).
    - *Outputs:* Connected to `Generate Reminder Email` via `ai_languageModel`.
  - **Generate Reminder Email**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* System prompt instructing the model to act as a client care assistant, keeping reminder emails under 140 words without guilt-tripping or em dashes, returning strictly valid JSON with `subject` and `body`.
    - *Inputs:* `Route by Client Status`, `OpenAI Reminder Model` -> *Outputs:* `Parse Draft Email`.
  - **Parse Draft Email**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Cleans markdown code fences, parses AI JSON output, and implements a deterministic fallback subject and body if parsing fails.
    - *Inputs:* `Generate Reminder Email` -> *Outputs:* `Send Client Reminder Email`.
  - **Send Client Reminder Email**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration:* Sends plain text email to client email address using parsed subject and body.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
    - *Inputs:* `Parse Draft Email` -> *Outputs:* `Update Reminder Date`.
    - *Edge Cases / Failure Types:* Gmail API token expiration or invalid recipient format.
  - **Update Reminder Date**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration:* Updates the `Clients` table, setting `last_reminded` to the current date (`{{ $now.format('yyyy-MM-dd') }}`) matched by `client_id`. Typecasting enabled.
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Send Client Reminder Email` -> *Outputs:* `Log Reminder Activity`.
  - **Log Reminder Activity**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration:* Creates a record in the `DocLog` table with timestamp, action (`reminder`), channel (`email`), client identifiers, and email subject details.
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Update Reminder Date` -> *Outputs:* None (Terminal node for this branch).

---

#### 2.4 Owner Escalation
- **Overview:** Escalates unresponsive client accounts by notifying the firm owner via Telegram and sending an internal escalation email, followed by Airtable status updates and activity logging.
- **Nodes Involved:**
  - `Notify Owner via Telegram`
  - `Send Escalation Email`
  - `Update Escalation Date`
  - `Log Escalation Activity`
- **Node Details:**
  - **Notify Owner via Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Messaging)
    - *Configuration:* Sends text alert detailing client name, ID, days silent, and missing documents to chat ID `123456789`.
    - *Credentials:* Telegram API (`CF Openai Bot`).
    - *Inputs:* `Route by Client Status` -> *Outputs:* `Send Escalation Email`.
  - **Send Escalation Email**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration:* Sends email to `owner_email` prompting a phone call for the silent client.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
    - *Inputs:* `Notify Owner via Telegram` -> *Outputs:* `Update Escalation Date`.
  - **Update Escalation Date**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration:* Updates `last_reminded` in the `Clients` table for the matching `client_id`.
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Send Escalation Email` -> *Outputs:* `Log Escalation Activity`.
  - **Log Escalation Activity**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration:* Creates a record in the `DocLog` table logging action type `escalation` over `email+telegram`.
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Update Escalation Date` -> *Outputs:* None (Terminal node for this branch).

---

#### 2.5 All-Clear Notification
- **Overview:** Executes when no clients require document chaser actions, posting an all-clear confirmation to Telegram and logging the event.
- **Nodes Involved:**
  - `Notify All Clear on Telegram`
  - `Log All Clear Activity`
- **Node Details:**
  - **Notify All Clear on Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Messaging)
    - *Configuration:* Posts an all-clear message indicating no clients are missing documents.
    - *Credentials:* Telegram API (`CF Openai Bot`).
    - *Inputs:* `Route by Client Status` -> *Outputs:* `Log All Clear Activity`.
  - **Log All Clear Activity**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration:* Creates an entry in the `DocLog` table with action `all_clear` and placeholder client identifiers (`ALL`).
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Notify All Clear on Telegram` -> *Outputs:* None (Terminal node for this branch).

---

#### 2.6 Webhook Document Intake & Validation
- **Overview:** Receives incoming document submissions via HTTP POST webhook, normalizes payload parameters, injects firm settings, and validates required data fields.
- **Nodes Involved:**
  - `Webhook: Document Received`
  - `Set Intake Data`
  - `Set Firm Intake Settings`
  - `Check Intake Validity`
- **Node Details:**
  - **Webhook: Document Received**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger)
    - *Configuration:* Listens on path `doc/received` for HTTP `POST` requests.
    - *Outputs:* `Set Intake Data`.
  - **Set Intake Data**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Extracts and sanitizes `client_id` (uppercase), `doc`, and `source` from request body.
    - *Inputs:* `Webhook: Document Received` -> *Outputs:* `Set Firm Intake Settings`.
  - **Set Firm Intake Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Assigns intake operational defaults (`firm_name`, `firm_email`).
    - *Inputs:* `Set Intake Data` -> *Outputs:* `Check Intake Validity`.
  - **Check Intake Validity**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Evaluates whether `client_id` and `doc` are not empty.
    - *Inputs:* `Set Firm Intake Settings` -> *Outputs:* `Fetch Client Info` (True branch) or `Prepare Invalid Notice` (False branch).

---

#### 2.7 Client Matching & Processing
- **Overview:** Searches Airtable for the submitted client, verifies if the document is expected on the client's requirement checklist, updates checklist completion status, and logs receipt confirmation.
- **Nodes Involved:**
  - `Fetch Client Info`
  - `Match Client ID and Prep Receipt`
  - `Check Client Exists`
  - `Log Document Reception`
  - `Document Receiving Activity`
  - `Confirm Receipt on Telegram`
- **Node Details:**
  - **Fetch Client Info**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Read)
    - *Configuration:* Searches the `Clients` table in Airtable base `appTGcnO2nvOQC86I`.
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Check Intake Validity` -> *Outputs:* `Match Client ID and Prep Receipt`.
  - **Match Client ID and Prep Receipt**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation / Logic)
    - *Configuration:* Validates if client exists and if the submitted document is listed in `required_docs`. Computes updated `received_docs` list and remaining missing documents.
    - *Inputs:* `Fetch Client Info` -> *Outputs:* `Check Client Exists`.
  - **Check Client Exists**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Evaluates boolean `found` property from previous node.
    - *Inputs:* `Match Client ID and Prep Receipt` -> *Outputs:* `Log Document Reception` (True branch) or `Notify Match Failure on Telegram` (False branch).
  - **Log Document Reception**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration:* Updates `received_docs` column in the `Clients` table for the matching client.
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Check Client Exists` -> *Outputs:* `Document Receiving Activity`.
  - **Document Receiving Activity**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration:* Creates an entry in the `DocLog` table recording action `received` via `webhook`.
    - *Credentials:* Airtable Personal Access Token (`contactmuhtadin`).
    - *Inputs:* `Log Document Reception` -> *Outputs:* `Confirm Receipt on Telegram`.
  - **Confirm Receipt on Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Messaging)
    - *Configuration:* Posts successful document intake confirmation and remaining checklist status to Telegram chat ID `123456789`.
    - *Credentials:* Telegram API (`CF Openai Bot`).
    - *Inputs:* `Document Receiving Activity` -> *Outputs:* None (Terminal node for this branch).

---

#### 2.8 Error Handling & Exception Management
- **Overview:** Catches malformed webhook payloads or unrecognized client/document combinations, notifying the firm via Telegram and email without modifying records.
- **Nodes Involved:**
  - `Prepare Invalid Notice`
  - `Alert Missing Data on Telegram`
  - `Send Data Missing Notice`
  - `Notify Match Failure on Telegram`
  - `Send Match Failure Notice`
- **Node Details:**
  - **Prepare Invalid Notice**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Sets rejection reason to "missing required fields".
    - *Inputs:* `Check Intake Validity` (False branch) -> *Outputs:* `Alert Missing Data on Telegram`.
  - **Alert Missing Data on Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Messaging)
    - *Configuration:* Sends rejection alert to Telegram chat ID `123456789`.
    - *Credentials:* Telegram API (`CF Openai Bot`).
    - *Inputs:* `Prepare Invalid Notice` -> *Outputs:* `Send Data Missing Notice`.
  - **Send Data Missing Notice**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration:* Emails firm inbox (`firm_email`) regarding missing intake fields.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
    - *Inputs:* `Alert Missing Data on Telegram` -> *Outputs:* None.
  - **Notify Match Failure on Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Messaging)
    - *Configuration:* Alerts Telegram chat ID `123456789` of unmatched client or document.
    - *Credentials:* Telegram API (`CF Openai Bot`).
    - *Inputs:* `Check Client Exists` (False branch) -> *Outputs:* `Send Match Failure Notice`.
  - **Send Match Failure Notice**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration:* Emails firm inbox (`firm_email`) regarding client or document mismatch.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
    - *Inputs:* `Notify Match Failure on Telegram` -> *Outputs:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Workflow documentation & setup overview | None | None | ## chase clients for missing tax documents with AI... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Load chase checklist... |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Assess chase routing... |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Draft reminder email... |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Send reminder record... |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Notify owner escalation... |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Log escalation status... |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Post all clear... |
| Sticky Note8 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Receive document intake... |
| Sticky Note9 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Validate known client... |
| Sticky Note10 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Record received document... |
| Sticky Note11 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Handle missing fields... |
| Sticky Note12 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Handle match failure... |
| When Workday Begins | `n8n-nodes-base.scheduleTrigger` | Triggers daily chase run | None | Set Firm Chase Settings | |
| Set Firm Chase Settings | `n8n-nodes-base.set` | Assigns firm metadata | When Workday Begins | Fetch Client Checklist | |
| Fetch Client Checklist | `n8n-nodes-base.airtable` | Queries Airtable clients | Set Firm Chase Settings | Evaluate Client Status | |
| Evaluate Client Status | `n8n-nodes-base.code` | Assesses missing documents & timing | Fetch Client Checklist | Route by Client Status | |
| Route by Client Status | `n8n-nodes-base.switch` | Routes action path | Evaluate Client Status | Generate Reminder Email, Notify Owner via Telegram, Notify All Clear on Telegram | |
| OpenAI Reminder Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM configuration for AI agent | None | Generate Reminder Email (ai_languageModel) | |
| Generate Reminder Email | `@n8n/n8n-nodes-langchain.agent` | AI agent drafts reminder email | Route by Client Status, OpenAI Reminder Model | Parse Draft Email | |
| Parse Draft Email | `n8n-nodes-base.code` | Parses and validates AI JSON output | Generate Reminder Email | Send Client Reminder Email | |
| Send Client Reminder Email | `n8n-nodes-base.gmail` | Sends reminder email to client | Parse Draft Email | Update Reminder Date | |
| Update Reminder Date | `n8n-nodes-base.airtable` | Updates last reminded date in Airtable | Send Client Reminder Email | Log Reminder Activity | |
| Log Reminder Activity | `n8n-nodes-base.airtable` | Logs reminder action in DocLog | Update Reminder Date | None | |
| Notify Owner via Telegram | `n8n-nodes-base.telegram` | Alerts owner of unresponsive client | Route by Client Status | Send Escalation Email | |
| Send Escalation Email | `n8n-nodes-base.gmail` | Emails owner escalation notice | Notify Owner via Telegram | Update Escalation Date | |
| Update Escalation Date | `n8n-nodes-base.airtable` | Updates escalation timestamp in Airtable | Send Escalation Email | Log Escalation Activity | |
| Log Escalation Activity | `n8n-nodes-base.airtable` | Logs escalation action in DocLog | Update Escalation Date | None | |
| Notify All Clear on Telegram | `n8n-nodes-base.telegram` | Posts all-clear status to Telegram | Route by Client Status | Log All Clear Activity | |
| Log All Clear Activity | `n8n-nodes-base.airtable` | Logs all-clear event in DocLog | Notify All Clear on Telegram | None | |
| Webhook: Document Received | `n8n-nodes-base.webhook` | Receives document intake requests | None | Set Intake Data | |
| Set Intake Data | `n8n-nodes-base.set` | Normalizes webhook payload parameters | Webhook: Document Received | Set Firm Intake Settings | |
| Set Firm Intake Settings | `n8n-nodes-base.set` | Sets firm intake defaults | Set Intake Data | Check Intake Validity | |
| Check Intake Validity | `n8n-nodes-base.if` | Validates required intake fields | Set Firm Intake Settings | Fetch Client Info, Prepare Invalid Notice | |
| Fetch Client Info | `n8n-nodes-base.airtable` | Looks up client record in Airtable | Check Intake Validity | Match Client ID and Prep Receipt | |
| Match Client ID and Prep Receipt | `n8n-nodes-base.code` | Matches client and checks document validity | Fetch Client Info | Check Client Exists | |
| Check Client Exists | `n8n-nodes-base.if` | Verifies client match success | Match Client ID and Prep Receipt | Log Document Reception, Notify Match Failure on Telegram | |
| Log Document Reception | `n8n-nodes-base.airtable` | Updates received docs in Airtable | Check Client Exists | Document Receiving Activity | |
| Document Receiving Activity | `n8n-nodes-base.airtable` | Logs document receipt in DocLog | Log Document Reception | Confirm Receipt on Telegram | |
| Confirm Receipt on Telegram | `n8n-nodes-base.telegram` | Confirms receipt on Telegram | Document Receiving Activity | None | |
| Prepare Invalid Notice | `n8n-nodes-base.set` | Formats invalid intake metadata | Check Intake Validity | Alert Missing Data on Telegram | |
| Alert Missing Data on Telegram | `n8n-nodes-base.telegram` | Alerts Telegram of missing intake data | Prepare Invalid Notice | Send Data Missing Notice | |
| Send Data Missing Notice | `n8n-nodes-base.gmail` | Emails firm inbox regarding missing fields | Alert Missing Data on Telegram | None | |
| Notify Match Failure on Telegram | `n8n-nodes-base.telegram` | Alerts Telegram of client/doc mismatch | Check Client Exists | Send Match Failure Notice | |
| Send Match Failure Notice | `n8n-nodes-base.gmail` | Emails firm inbox regarding mismatch error | Notify Match Failure on Telegram | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow in n8n:

1. **Create Schedule Trigger Node:**
   - Type: `Schedule Trigger` (`When Workday Begins`)
   - Parameter: Cron expression set to `0 9 * * 1-5`.
2. **Create Firm Chase Settings Node:**
   - Type: `Set` (`Set Firm Chase Settings`)
   - Assignments: Add string fields `firm_name` ("Lumen Tax and Accounting"), `owner_email`, and `firm_email`. Include other fields.
   - Connection: Connect `When Workday Begins` output to this node.
3. **Create Airtable Client Checklist Fetcher:**
   - Type: `Airtable` (`Fetch Client Checklist`)
   - Parameters: Base ID `appTGcnO2nvOQC86I`, Table Name `Clients`, Operation `Search`.
   - Credentials: Configure Airtable Personal Access Token.
   - Connection: Connect `Set Firm Chase Settings` output to this node.
4. **Create Evaluate Client Status Node:**
   - Type: `Code` (`Evaluate Client Status`)
   - Parameters: Insert JavaScript code to iterate over rows, compute missing documents against `required_docs` and `received_docs`, evaluate days since `last_reminded`, and assign actions (`remind`, `escalate`, `none`, or `all-clear`).
   - Connection: Connect `Fetch Client Checklist` output to this node.
5. **Create Route by Client Status Node:**
   - Type: `Switch` (`Route by Client Status`)
   - Parameters: Configure 3 routing rules based on `{{ $json.action }}` matching: `remind`, `escalate`, and `all-clear`.
   - Connection: Connect `Evaluate Client Status` output to this node.
6. **Set up AI Reminder Branch:**
   - Create an **OpenAI Chat Model** sub-node (`OpenAI Reminder Model`) using model `gpt-4o-mini`, temperature `0.4`, and configure OpenAI credentials.
   - Create an **AI Agent** node (`Generate Reminder Email`) connected via `ai_languageModel` to the OpenAI model. Configure the system prompt to return strictly valid JSON with `subject` and `body`.
   - Connect rule 1 (`remind`) of `Route by Client Status` to `Generate Reminder Email`.
   - Create a **Code** node (`Parse Draft Email`) to parse JSON output with a fallback mechanism. Connect to `Generate Reminder Email`.
   - Create a **Gmail** node (`Send Client Reminder Email`) configured with Gmail OAuth2 credentials, mapping recipient email, subject, and body from the parse node.
   - Create an **Airtable** node (`Update Reminder Date`) to update the `Clients` table (`client_id` matching, updating `last_reminded` to `{{ $now.format('yyyy-MM-dd') }}`).
   - Create an **Airtable** node (`Log Reminder Activity`) to create a record in `DocLog` (action: `reminder`, channel: `email`).
   - Connect sequentially: `Parse Draft Email` -> `Send Client Reminder Email` -> `Update Reminder Date` -> `Log Reminder Activity`.
7. **Set up Owner Escalation Branch:**
   - Connect rule 2 (`escalate`) of `Route by Client Status` to a **Telegram** node (`Notify Owner via Telegram`) using Telegram credentials and target chat ID `123456789`.
   - Connect to a **Gmail** node (`Send Escalation Email`) sending an escalation notice to `owner_email`.
   - Connect to an **Airtable** node (`Update Escalation Date`) updating `last_reminded` in `Clients`.
   - Connect to an **Airtable** node (`Log Escalation Activity`) creating a log entry in `DocLog` (action: `escalation`).
   - Connect sequentially: `Notify Owner via Telegram` -> `Send Escalation Email` -> `Update Escalation Date` -> `Log Escalation Activity`.
8. **Set up All-Clear Branch:**
   - Connect rule 3 (`all-clear`) of `Route by Client Status` to a **Telegram** node (`Notify All Clear on Telegram`) to post an all-clear notification.
   - Connect to an **Airtable** node (`Log All Clear Activity`) to log the event in `DocLog` (action: `all_clear`).
   - Connect sequentially: `Notify All Clear on Telegram` -> `Log All Clear Activity`.
9. **Set up Webhook Intake Branch:**
   - Create a **Webhook** trigger node (`Webhook: Document Received`) with method `POST` and path `doc/received`.
   - Create a **Set** node (`Set Intake Data`) extracting `client_id`, `doc`, and `source` from the body.
   - Create a **Set** node (`Set Firm Intake Settings`) assigning firm defaults.
   - Create an **If** node (`Check Intake Validity`) checking that `client_id` and `doc` are not empty.
   - Connect sequentially: `Webhook: Document Received` -> `Set Intake Data` -> `Set Firm Intake Settings` -> `Check Intake Validity`.
10. **Set up Client Matching & Receipt Branch:**
    - Connect the `True` output of `Check Intake Validity` to an **Airtable** node (`Fetch Client Info`) searching the `Clients` table.
    - Connect to a **Code** node (`Match Client ID and Prep Receipt`) verifying client existence and document requirement status.
    - Connect to an **If** node (`Check Client Exists`) evaluating the `found` boolean property.
    - From the `True` output of `Check Client Exists`, connect sequentially to:
      - **Airtable** (`Log Document Reception`): Updates `received_docs` in `Clients`.
      - **Airtable** (`Document Receiving Activity`): Creates a log entry in `DocLog` (action: `received`).
      - **Telegram** (`Confirm Receipt on Telegram`): Posts intake success confirmation to chat ID `123456789`.
11. **Set up Error Handling Branches:**
    - From the `False` output of `Check Intake Validity`, connect to a **Set** node (`Prepare Invalid Notice`), then a **Telegram** node (`Alert Missing Data on Telegram`), and finally a **Gmail** node (`Send Data Missing Notice`) targeting `firm_email`.
    - From the `False` output of `Check Client Exists`, connect to a **Telegram** node (`Notify Match Failure on Telegram`), and then a **Gmail** node (`Send Match Failure Notice`) targeting `firm_email`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n | [n8n Platform](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Consultation & Business Assessment | [Kh Muhtadin Consultation](https://khmuhtadin.com/consultation/) |