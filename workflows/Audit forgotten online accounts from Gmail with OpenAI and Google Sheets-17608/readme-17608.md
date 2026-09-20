Audit forgotten online accounts from Gmail with OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/audit-forgotten-online-accounts-from-gmail-with-openai-and-google-sheets-17608


# Audit forgotten online accounts from Gmail with OpenAI and Google Sheets

### 1. Workflow Overview

This workflow automates the discovery, extraction, risk assessment, and tracking of online accounts using emails from a connected Gmail inbox, an AI model via OpenRouter, and a Google Sheets register. It operates in two main modes: **Discovery Mode** (triggered manually or daily) and **Digest Mode** (triggered weekly).

The architecture is divided into the following functional blocks:
- **1.1 Initialization & Mode Routing:** Entry points (Manual, Daily Schedule, Weekly Schedule), execution mode setters, and global template parameter initialization.
- **1.2 Account Discovery & Ingestion:** Searches Gmail messages based on specific queries, fetches full message contents, and cleans/filters emails for candidate account signals.
- **1.3 AI Extraction & Normalization:** Constructs model-neutral AI payloads, queries the OpenRouter API, and normalizes unstructured AI responses into structured schema fields.
- **1.4 Inventory Upsert & Human Preservation:** Labels processed Gmail messages, filters validated account signals, reads existing Google Sheets data to preserve human-set statuses/actions, and upserts the records.
- **1.5 Review Notifications & Weekly Digest:** Evaluates records requiring review to send targeted Gmail alerts, and processes the weekly privacy digest for dormant or action-required accounts.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Initialization & Mode Routing

#### Overview
This block handles workflow triggering via manual or scheduled executions, sets the execution mode (discovery vs. digest), and standardizes global variables like spreadsheet IDs, email configurations, and AI model parameters.

#### Nodes Involved
- `Manual Historical Audit`
- `Daily Account Discovery`
- `Weekly Privacy Digest`
- `Set Discovery Mode`
- `Set Digest Mode`
- `Set Template Fields`
- `Is Weekly Digest Run`

#### Node Details

- **Manual Historical Audit**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Manual execution entry point.
  - **Configuration Choices:** Default setup.
  - **Expressions/Variables:** None.
  - **Connections:** Output connects to `Set Discovery Mode`.
  - **Version Requirements:** v1.
  - **Edge Cases / Failures:** None.

- **Daily Account Discovery**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` — Cron-based scheduled trigger.
  - **Configuration Choices:** Runs daily at `09:20 AM` (`20 9 * * *`).
  - **Expressions/Variables:** None.
  - **Connections:** Output connects to `Set Discovery Mode`.
  - **Version Requirements:** v1.2.
  - **Edge Cases / Failures:** Timezone dependencies match workflow settings.

- **Weekly Privacy Digest**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` — Cron-based scheduled trigger for weekly review.
  - **Configuration Choices:** Runs every Monday at `09:00 AM` (`0 9 * * 1`).
  - **Expressions/Variables:** None.
  - **Connections:** Output connects to `Set Digest Mode`.
  - **Version Requirements:** v1.2.
  - **Edge Cases / Failures:** Timezone dependencies match workflow settings.

- **Set Discovery Mode**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Sets execution mode variable.
  - **Configuration Choices:** Assigns string value `discovery` to `executionMode`.
  - **Expressions/Variables:** `{{ $json.executionMode }}`
  - **Connections:** Input from `Manual Historical Audit` or `Daily Account Discovery`; output to `Set Template Fields`.
  - **Version Requirements:** v3.4.
  - **Edge Cases / Failures:** None.

- **Set Digest Mode**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Sets execution mode variable.
  - **Configuration Choices:** Assigns string value `digest` to `executionMode`.
  - **Expressions/Variables:** `{{ $json.executionMode }}`
  - **Connections:** Input from `Weekly Privacy Digest`; output to `Set Template Fields`.
  - **Version Requirements:** v3.4.
  - **Edge Cases / Failures:** None.

- **Set Template Fields**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Global configuration repository.
  - **Configuration Choices:** Sets search queries, processed Gmail labels, Google Sheet IDs, tab names, review/digest emails, AI model parameters, confidence thresholds, target country, timezone, and sensitive categories.
  - **Expressions/Variables:** Hardcoded configuration parameters.
  - **Connections:** Input from `Set Discovery Mode` or `Set Digest Mode`; output to `Is Weekly Digest Run`.
  - **Version Requirements:** v3.4.
  - **Edge Cases / Failures:** Incorrect Sheet IDs or missing credential bindings will fail subsequent Google Sheets or Gmail nodes.

- **Is Weekly Digest Run**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Conditional branching based on execution mode.
  - **Configuration Choices:** Checks if `{{ $json.executionMode }}` equals `digest`.
  - **Expressions/Variables:** `{{ $json.executionMode }}`
  - **Connections:** Input from `Set Template Fields`; true output to `Read Account Register`, false output to `Search Gmail Account Signals`.
  - **Version Requirements:** v2.2.
  - **Edge Cases / Failures:** Evaluates string equality strictly.

---

### Block 1.2: Account Discovery & Ingestion

#### Overview
This block interacts with Gmail to search for account-related correspondence, retrieves full email payloads, and parses candidate bodies while filtering out irrelevant auto-responses or low-text emails.

#### Nodes Involved
- `Search Gmail Account Signals`
- `Fetch Full Account Emails`
- `Prepare Candidate Account Emails`

#### Node Details

- **Search Gmail Account Signals**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` — Searches messages using Gmail query operators.
  - **Configuration Choices:** Operation `getAll`, resource `message`, limit defined by `maxDiscoveryMessages`, filter query bound to `gmailDiscoveryQuery`.
  - **Expressions/Variables:** `{{ $json.maxDiscoveryMessages }}`, `{{ $json.gmailDiscoveryQuery }}`
  - **Connections:** Input from `Is Weekly Digest Run` (false branch); output to `Fetch Full Account Emails`.
  - **Version Requirements:** v2.1.
  - **Credential Requirements:** Gmail OAuth2 credentials.
  - **Edge Cases / Failures:** Authentication token expiration or invalid query syntax.

- **Fetch Full Account Emails**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` — Retrieves complete email contents.
  - **Configuration Choices:** Operation `get`, resource `message`.
  - **Expressions/Variables:** `{{ $json.id }}`
  - **Connections:** Input from `Search Gmail Account Signals`; output to `Prepare Candidate Account Emails`.
  - **Version Requirements:** v2.1.
  - **Credential Requirements:** Gmail OAuth2 credentials.
  - **Edge Cases / Failures:** Rate limiting or missing message IDs.

- **Prepare Candidate Account Emails**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript-based data cleaner and filter.
  - **Configuration Choices:** Extracts headers, strips HTML tags, filters out automated delivery daemons/bounces, and enforces a minimum body length of 24 characters.
  - **Expressions/Variables:** Accesses previous node payloads and pulls configuration from `Set Template Fields`.
  - **Connections:** Input from `Fetch Full Account Emails`; output to `Build Account Inventory AI Request`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failures:** Malformed email header arrays.

---

### Block 1.3: AI Extraction & Normalization

#### Overview
This block structures prompts for AI processing, sends requests to the OpenRouter chat completions endpoint, and normalizes model outputs into a verified account signal schema containing risk metrics.

#### Nodes Involved
- `Build Account Inventory AI Request`
- `Extract Account Signals with AI`
- `Normalize Account Extraction`

#### Node Details

- **Build Account Inventory AI Request**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Prepares HTTP payloads for AI processing.
  - **Configuration Choices:** Defines strict JSON schema rules, zero temperature, and encapsulates email metadata and message bodies.
  - **Expressions/Variables:** Uses `j.aiModel`, `j.messageId`, `j.sourceFrom`, `j.sourceSubject`, `j.receivedAt`, and `j.messageBody`.
  - **Connections:** Input from `Prepare Candidate Account Emails`; output to `Extract Account Signals with AI`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failures:** String truncation constraints on large email bodies.

- **Extract Account Signals with AI**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Communicates with the AI provider gateway.
  - **Configuration Choices:** POST method to `https://openrouter.ai/api/v1/chat/completions`, JSON body transmission, 60-second timeout, batch size of 1 with 800ms interval.
  - **Expressions/Variables:** `{{ JSON.stringify($json.aiRequestBody) }}`
  - **Connections:** Input from `Build Account Inventory AI Request`; output to `Normalize Account Extraction`.
  - **Version Requirements:** v4.2.
  - **Credential Requirements:** HTTP Header Auth (OpenRouter / AI Gateway).
  - **Edge Cases / Failures:** API timeouts, rate limits, or invalid JSON output from the model.

- **Normalize Account Extraction**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Parses and validates AI model responses.
  - **Configuration Choices:** Strips Markdown code blocks, validates categories against an allowed list, calculates risk scores based on sensitivity/billing signals, and generates unique account keys.
  - **Expressions/Variables:** Accesses upstream responses and configuration fields.
  - **Connections:** Input from `Extract Account Signals with AI`; outputs to `Label Assessed Gmail Messages` and `Is Account Signal`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failures:** Malformed JSON from LLM outputs causing parsing exceptions.

---

### Block 1.4: Inventory Upsert & Human Preservation

#### Overview
This block applies audit labels to assessed emails, filters for verified account signals, merges incoming data with existing Google Sheet records to protect human-set decisions, and upserts the records.

#### Nodes Involved
- `Label Assessed Gmail Messages`
- `Is Account Signal`
- `Prepare Account Register Row`
- `Read Existing Account Register`
- `Preserve Human Account Decisions`
- `Upsert Account Register`
- `Restore Account Context`

#### Node Details

- **Label Assessed Gmail Messages**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` — Applies audit tracking labels to messages.
  - **Configuration Choices:** Operation `addLabels`, resource `message`, error handling configured to continue on regular output.
  - **Expressions/Variables:** `{{ $json.processedGmailLabel }}`, `{{ $json.messageId }}`
  - **Connections:** Input from `Normalize Account Extraction`.
  - **Version Requirements:** v2.1.
  - **Credential Requirements:** Gmail OAuth2 credentials.
  - **Edge Cases / Failures:** Label not existing in Gmail or API permission errors.

- **Is Account Signal**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Evaluates if the extraction represents a valid account signal.
  - **Configuration Choices:** Checks if `{{ $json.isAccountSignal }}` equals `true`.
  - **Expressions/Variables:** `{{ $json.isAccountSignal }}`
  - **Connections:** Input from `Normalize Account Extraction`; true output to `Prepare Account Register Row`, false branch terminates.
  - **Version Requirements:** v2.2.
  - **Edge Cases / Failures:** Strict boolean matching.

- **Prepare Account Register Row**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Formats data for spreadsheet insertion.
  - **Configuration Choices:** Maps normalized parameters to exact column headers matching the Google Sheet structure.
  - **Expressions/Variables:** `{{ j.timezone }}`, dynamic timestamping.
  - **Connections:** Input from `Is Account Signal`; output to `Read Existing Account Register`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failures:** Timezone formatting errors.

- **Read Existing Account Register**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Fetches current sheet rows.
  - **Configuration Choices:** Reads data from configured spreadsheet ID and tab name.
  - **Expressions/Variables:** `{{ $(\"Set Template Fields\").first().json.accountRegisterSheetTab }}`, `{{ $(\"Set Template Fields\").first().json.accountRegisterSheetId }}`
  - **Connections:** Input from `Prepare Account Register Row`; output to `Preserve Human Account Decisions`.
  - **Version Requirements:** v4.6.
  - **Credential Requirements:** Google Sheets OAuth2 credentials.
  - **Edge Cases / Failures:** Missing tab names or permission errors.

- **Preserve Human Account Decisions**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript merger preventing overwrite of user edits.
  - **Configuration Choices:** Retains human-edited values for `Status` (e.g., Closed), `Desired Action`, `Data Rights Draft`, and `First Observed At`.
  - **Expressions/Variables:** Accesses upstream sheet data and proposed row metrics.
  - **Connections:** Input from `Read Existing Account Register`; output to `Upsert Account Register`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failures:** Key mismatches during map lookup.

- **Upsert Account Register**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Updates or appends rows in Google Sheets.
  - **Configuration Choices:** Operation `appendOrUpdate`, matching columns set to `Account Key`, error handling configured to continue on regular output.
  - **Expressions/Variables:** Dynamic sheet and tab references.
  - **Connections:** Input from `Preserve Human Account Decisions`; output to `Restore Account Context`.
  - **Version Requirements:** v4.6.
  - **Credential Requirements:** Google Sheets OAuth2 credentials.
  - **Edge Cases / Failures:** Schema column mismatches or rate limits.

- **Restore Account Context**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Restores original item context following Google Sheets output substitution.
  - **Configuration Choices:** Matches rows using `Account Key`.
  - **Expressions/Variables:** `{{ item.json?.['Account Key'] }}`
  - **Connections:** Input from `Upsert Account Register`; output to `Needs Account Review`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failures:** Missing key matches.

---

### Block 1.5: Review Notifications & Weekly Digest

#### Overview
This block filters records requiring human review to dispatch targeted Gmail alerts, and reads the account register during weekly runs to compile and send a prioritized privacy digest with data-rights drafts.

#### Nodes Involved
- `Needs Account Review`
- `Send Review Notice`
- `Read Account Register`
- `Prepare Privacy Digest`
- `Send Weekly Privacy Digest`

#### Node Details

- **Needs Account Review**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Conditional filter for review items.
  - **Configuration Choices:** Checks if `{{ $json["Status"] }}` equals `Needs review`.
  - **Expressions/Variables:** `{{ $json["Status"] }}`
  - **Connections:** Input from `Restore Account Context`; true output to `Send Review Notice`, false branch terminates.
  - **Version Requirements:** v2.2.
  - **Edge Cases / Failures:** String matching discrepancies.

- **Send Review Notice**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` — Sends review notification emails.
  - **Configuration Choices:** Plain text email, custom sender name (`Digital Closet Audit`), error handling configured to continue on regular output.
  - **Expressions/Variables:** `{{ $(\"Set Template Fields\").first().json.reviewEmail }}`, `{{ $json["Service Name"] }}`, `{{ $json["Category"] }}`, `{{ $json["Confidence"] }}`, `{{ $json["Review Reason"] }}`, `{{ $json["Source Link"] }}`
  - **Connections:** Input from `Needs Account Review`.
  - **Version Requirements:** v2.1.
  - **Credential Requirements:** Gmail OAuth2 credentials.
  - **Edge Cases / Failures:** SMTP limits or invalid recipient addresses.

- **Read Account Register**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Reads full register data for weekly digests.
  - **Configuration Choices:** Operation `get` / read range.
  - **Expressions/Variables:** `{{ $json.accountRegisterSheetTab }}`, `{{ $json.accountRegisterSheetId }}`
  - **Connections:** Input from `Is Weekly Digest Run` (true branch); output to `Prepare Privacy Digest`.
  - **Version Requirements:** v4.6.
  - **Credential Requirements:** Google Sheets OAuth2 credentials.
  - **Edge Cases / Failures:** Empty spreadsheet tabs or missing range definitions.

- **Prepare Privacy Digest**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Prioritizes records and generates data rights request drafts.
  - **Configuration Choices:** Filters out closed/ignored accounts, scores prioritization based on risk and requested actions, slices top 12 items, and formats digest text.
  - **Expressions/Variables:** Accesses spreadsheet row objects and template configurations.
  - **Connections:** Input from `Read Account Register`; output to `Send Weekly Privacy Digest`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failures:** Handling empty datasets when no active records require review.

- **Send Weekly Privacy Digest**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` — Dispatches the weekly summary email.
  - **Configuration Choices:** Plain text email, custom sender name (`Digital Closet Audit`), error handling configured to continue on regular output.
  - **Expressions/Variables:** `{{ $json.digestEmail }}`, `{{ $json.digestBody }}`, `{{ $json.reviewCount }}`
  - **Connections:** Input from `Prepare Privacy Digest`.
  - **Version Requirements:** v2.1.
  - **Credential Requirements:** Gmail OAuth2 credentials.
  - **Edge Cases / Failures:** Delivery failures due to malformed message bodies.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note — Overview | n8n-nodes-base.stickyNote | Documentation and setup guidelines | None | None | Digital Closet Audit<br><br>Your inbox is a map of every company that still knows you.<br><br>This privacy-first workflow turns that map into a **reviewable account inventory**. It searches only for account signals you choose, uses any compatible AI model to identify the service and evidence, and stores a single source-linked record per account in Google Sheets.<br><br>## What you get<br><br>- Forgotten services, newsletters, subscriptions, and accounts collected in one register.<br>- A practical risk score based on account age, recent activity, billing language, and sensitive categories.<br>- A weekly privacy digest with the few accounts that deserve your attention.<br>- Copy-ready data export and deletion request drafts. The workflow never sends them.<br><br>## Safety model<br><br>The AI is advisory. Low-confidence discoveries are marked **Needs review**. Gmail is never deleted or archived. The processed label prevents the same email from being assessed twice. No service is contacted, no account is closed, and no personal data is exported without an explicit human decision.<br><br>## Before activation<br><br>1. Create a Google Sheet with a tab named **Account Register** and the headers listed in the **Configure before activation** note.<br>2. In n8n, connect Gmail, Google Sheets, and an HTTP Header Auth credential for your AI provider.<br>3. In **Set Template Fields**, add your Sheet ID and email addresses, then run **Manual Historical Audit** with the restricted test query.<br>4. Review every row and confirm the Gmail label was applied before widening the discovery query or activating a schedule. |
| Sticky Note — Configure first | n8n-nodes-base.stickyNote | Pre-activation configuration instructions | None | None | ## Configure before activation<br><br>**Set Template Fields** is the only place to configure the workflow. Before activating, create a Google Sheet tab named **Account Register** with these exact headers:<br><br>**Account Key**, **Status**, **Service Name**, **Service Domain**, **Category**, **Sensitive Signal**, **Billing Signal**, **Risk Score**, **Confidence**, **Account Evidence**, **First Observed At**, **Last Observed At**, **Source From**, **Source Subject**, **Source Message ID**, **Source Thread ID**, **Source Link**, **Desired Action**, **Data Rights Draft**, **Review Reason**, **Last Digest At**.<br><br>Then connect Gmail, Google Sheets, and an AI HTTP Header Auth credential in n8n. Do not place a secret in any node field.<br><br>Configure:<br><br>- Gmail search queries for discovery and testing.<br>- Your processed Gmail label and Google Sheets register.<br>- Your preferred AI model and confidence threshold.<br>- Email addresses for the weekly digest and review notices.<br>- Your home country for data-rights wording.<br><br>Create the Gmail label `n8n-account-audited` before running a discovery. Start with the test query and review every row before using a wider historical query. Use **Open**, **Needs review**, **Closed**, or **Ignored** for Status; use **Review**, **Keep**, **Export data**, or **Delete account** for Desired Action.<br><br>Never place API keys in a node field. Use credentials. |
| Sticky Note — Discover account signals | n8n-nodes-base.stickyNote | Explains discovery logic and idempotency guards | None | None | ## 1. Find account evidence, not every email<br><br>The Gmail query looks for account creation, verification, password reset, billing, receipt, and policy-change signals. The preparation node rejects automated delivery notices and messages without enough usable text.<br><br>Each email has a unique Message ID. The processed label is the first idempotency guard; the Account Key is the second guard when records are written to Google Sheets. |
| Sticky Note — Extract with AI | n8n-nodes-base.stickyNote | Explains model extraction rules and gateway setup | None | None | ## 2. Extract structured facts with a model you choose<br><br>The model receives one candidate email and must return strict JSON: service name, website domain, category, account evidence, billing signal, and confidence. It is explicitly prohibited from inferring passwords, payment details, or personal secrets.<br><br>The template uses OpenRouter HTTP Header Auth only as a model gateway. Replace it with any compatible gateway if preferred. |
| Sticky Note — Build a durable privacy register | n8n-nodes-base.stickyNote | Explains Google Sheets upsert logic | None | None | ## 3. Keep one source-backed account record<br><br>Google Sheets receives a single upserted row per Account Key. It keeps the newest evidence, the latest observed date, an explainable risk score, and a direct Gmail source link.<br><br>Rows are **Needs review** when the model is uncertain, the service cannot be identified, or the message looks sensitive. The workflow never treats a row as proof that an account is currently active. |
| Sticky Note — Review actions stay human | n8n-nodes-base.stickyNote | Outlines human-in-the-loop decision processes | None | None | ## 4. Decide, then act yourself<br><br>Set `Desired Action` in the register to **Keep**, **Export data**, or **Delete account**. The weekly digest includes a copy-ready request draft and the exact source message that led to the record.<br><br>No deletion, export request, support ticket, or email is sent automatically. This avoids accidental account closure, loss of access, and unreviewed privacy requests. |
| Sticky Note — Weekly privacy pulse | n8n-nodes-base.stickyNote | Explains weekly review triggers and prioritization | None | None | ## 5. Receive one useful privacy digest<br><br>The weekly run reads the register and sends a concise digest only when action is needed. It prioritizes dormant accounts with billing language, sensitive categories, old records that were never reviewed, and rows you explicitly marked for export or deletion.<br><br>The digest is a decision aid, not legal advice. Check each service's current privacy process before submitting a request. |
| Sticky Note — Privacy boundaries | n8n-nodes-base.stickyNote | Security and privacy best practices | None | None | Privacy boundaries<br><br>- Grant Gmail access only to the mailbox you intend to audit.<br>- Use a restricted Gmail query during testing.<br>- Do not store passwords, full payment numbers, identity documents, health details, or message attachments in the register.<br>- Restrict access to the Google Sheet because it becomes an inventory of your online footprint.<br>- An AI extraction is a suggestion. Confirm every review row before acting. |
| Manual Historical Audit | n8n-nodes-base.manualTrigger | Manual execution trigger | None | Set Discovery Mode | |
| Daily Account Discovery | n8n-nodes-base.scheduleTrigger | Daily schedule execution trigger | None | Set Discovery Mode | |
| Set Discovery Mode | n8n-nodes-base.set | Sets execution mode to discovery | Manual Historical Audit, Daily Account Discovery | Set Template Fields | |
| Weekly Privacy Digest | n8n-nodes-base.scheduleTrigger | Weekly schedule execution trigger | None | Set Digest Mode | |
| Set Digest Mode | n8n-nodes-base.set | Sets execution mode to digest | Weekly Privacy Digest | Set Template Fields | |
| Set Template Fields | n8n-nodes-base.set | Configures global workflow parameters and schema values | Set Discovery Mode, Set Digest Mode | Is Weekly Digest Run | |
| Is Weekly Digest Run | n8n-nodes-base.if | Evaluates execution path (digest vs discovery) | Set Template Fields | Read Account Register, Search Gmail Account Signals | |
| Search Gmail Account Signals | n8n-nodes-base.gmail | Queries Gmail for account-related emails | Is Weekly Digest Run | Fetch Full Account Emails | |
| Fetch Full Account Emails | n8n-nodes-base.gmail | Retrieves full message payloads | Search Gmail Account Signals | Prepare Candidate Account Emails | |
| Prepare Candidate Account Emails | n8n-nodes-base.code | Cleans email bodies and filters out automated notices | Fetch Full Account Emails | Build Account Inventory AI Request | |
| Build Account Inventory AI Request | n8n-nodes-base.code | Constructs model-neutral AI prompt payload | Prepare Candidate Account Emails | Extract Account Signals with AI | |
| Extract Account Signals with AI | n8n-nodes-base.httpRequest | Sends extraction request to AI model gateway | Build Account Inventory AI Request | Normalize Account Extraction | |
| Normalize Account Extraction | n8n-nodes-base.code | Parses AI response, normalizes domains and computes risk score | Extract Account Signals with AI | Label Assessed Gmail Messages, Is Account Signal | |
| Label Assessed Gmail Messages | n8n-nodes-base.gmail | Applies processed audit label to assessed Gmail message | Normalize Account Extraction | None | |
| Is Account Signal | n8n-nodes-base.if | Filters verified account signals | Normalize Account Extraction | Prepare Account Register Row | |
| Prepare Account Register Row | n8n-nodes-base.code | Maps attributes to Google Sheets column structure | Is Account Signal | Read Existing Account Register | |
| Read Existing Account Register | n8n-nodes-base.googleSheets | Fetches existing rows to preserve human edits | Prepare Account Register Row | Preserve Human Account Decisions | |
| Preserve Human Account Decisions | n8n-nodes-base.code | Merges fresh data without overriding human-set statuses/actions | Read Existing Account Register | Upsert Account Register | |
| Upsert Account Register | n8n-nodes-base.googleSheets | Updates or appends rows in Google Sheets | Preserve Human Account Decisions | Restore Account Context | |
| Restore Account Context | n8n-nodes-base.code | Restores initial context using Account Key | Upsert Account Register | Needs Account Review | |
| Needs Account Review | n8n-nodes-base.if | Filters rows requiring manual review | Restore Account Context | Send Review Notice | |
| Send Review Notice | n8n-nodes-base.gmail | Sends review alert email | Needs Account Review | None | |
| Read Account Register | n8n-nodes-base.googleSheets | Reads register data for weekly digest | Is Weekly Digest Run | Prepare Privacy Digest | |
| Prepare Privacy Digest | n8n-nodes-base.code | Prioritizes items and builds data rights request drafts | Read Account Register | Send Weekly Privacy Digest | |
| Send Weekly Privacy Digest | n8n-nodes-base.gmail | Sends weekly summary email | Prepare Privacy Digest | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create Triggers & Mode Setters**
   - Add a **Manual Trigger** (`Manual Historical Audit`).
   - Add two **Schedule Trigger** nodes (`Daily Account Discovery` set to cron `20 9 * * *` and `Weekly Privacy Digest` set to cron `0 9 * * 1`).
   - Add two **Set** nodes (`Set Discovery Mode` assigning `executionMode = "discovery"` and `Set Digest Mode` assigning `executionMode = "digest"`).
   - Connect `Manual Historical Audit` and `Daily Account Discovery` to `Set Discovery Mode`. Connect `Weekly Privacy Digest` to `Set Digest Mode`.

2. **Configure Global Variables**
   - Add a **Set** node named `Set Template Fields`.
   - Populate assignments for: `gmailDiscoveryQuery`, `gmailTestQuery`, `processedGmailLabel` (set to `n8n-account-audited`), `accountRegisterSheetId`, `accountRegisterSheetTab` (set to `Account Register`), `digestEmail`, `reviewEmail`, `aiModel` (e.g., `openai/gpt-4.1-mini`), `minimumConfidence` (`0.82`), `countryOrRegion`, `timezone`, `maxDiscoveryMessages` (`100`), and `sensitiveCategories`.
   - Connect both `Set Discovery Mode` and `Set Digest Mode` to `Set Template Fields`.

3. **Configure Execution Branching**
   - Add an **If** node named `Is Weekly Digest Run` checking if `{{ $json.executionMode }}` equals `digest`.
   - Connect `Set Template Fields` output to `Is Weekly Digest Run`.

4. **Build Discovery & Ingestion Pipeline (False Branch)**
   - Add a **Gmail** node (`Search Gmail Account Signals`) configured with operation `getAll`, resource `message`, limit `{{ $json.maxDiscoveryMessages }}`, and query filter `{{ $json.gmailDiscoveryQuery }}`. Connect the `false` output of `Is Weekly Digest Run` here.
   - Add a **Gmail** node (`Fetch Full Account Emails`) configured with operation `get`, resource `message`, and messageId `{{ $json.id }}`. Connect `Search Gmail Account Signals` to it.
   - Add a **Code** node (`Prepare Candidate Account Emails`) using JavaScript to strip HTML, extract headers, and filter out system bounces. Connect `Fetch Full Account Emails` to it.

5. **Build AI Extraction & Normalization Pipeline**
   - Add a **Code** node (`Build Account Inventory AI Request`) to structure the LLM prompt payload and schema rules. Connect `Prepare Candidate Account Emails` to it.
   - Add an **HTTP Request** node (`Extract Account Signals with AI`) targeting `https://openrouter.ai/api/v1/chat/completions` via POST, configured with JSON body formatting and an **HTTP Header Auth** credential. Connect `Build Account Inventory AI Request` to it.
   - Add a **Code** node (`Normalize Account Extraction`) to parse LLM responses, normalize domains, and calculate risk scores. Connect `Extract Account Signals with AI` to it.

6. **Build Upsert & Preservation Pipeline**
   - Add a **Gmail** node (`Label Assessed Gmail Messages`) with operation `addLabels`, labelIds `{{ $json.processedGmailLabel }}`, and messageId `{{ $json.messageId }}`. Enable error handling ("Continue on Fail"). Connect `Normalize Account Extraction` to it.
   - Add an **If** node (`Is Account Signal`) checking `{{ $json.isAccountSignal === true }}`. Connect `Normalize Account Extraction` to it.
   - Add a **Code** node (`Prepare Account Register Row`) to map attributes to column headers. Connect the `true` output of `Is Account Signal` to it.
   - Add a **Google Sheets** node (`Read Existing Account Register`) reading from the configured Sheet ID and tab name using **Google Sheets OAuth2** credentials. Connect `Prepare Account Register Row` to it.
   - Add a **Code** node (`Preserve Human Account Decisions`) to merge incoming updates without overwriting human edits (`Status`, `Desired Action`, etc.). Connect `Read Existing Account Register` to it.
   - Add a **Google Sheets** node (`Upsert Account Register`) with operation `appendOrUpdate`, matching columns set to `Account Key`, and error handling set to continue on fail. Connect `Preserve Human Account Decisions` to it.
   - Add a **Code** node (`Restore Account Context`) matching records by `Account Key`. Connect `Upsert Account Register` to it.

7. **Build Review Notice & Weekly Digest Pipeline**
   - Add an **If** node (`Needs Account Review`) checking if `{{ $json["Status"] === "Needs review" }}`. Connect `Restore Account Context` to it.
   - Add a **Gmail** node (`Send Review Notice`) sending review alert emails using template variables. Enable error handling. Connect the `true` output of `Needs Account Review` to it.
   - Add a **Google Sheets** node (`Read Account Register`) connected to the `true` output of `Is Weekly Digest Run`, reading sheet data for weekly summaries.
   - Add a **Code** node (`Prepare Privacy Digest`) to filter, sort, prioritize records, and generate data rights request drafts. Connect `Read Account Register` to it.
   - Add a **Gmail** node (`Send Weekly Privacy Digest`) to dispatch the weekly summary email. Enable error handling. Connect `Prepare Privacy Digest` to it.

8. **Credentials Setup**
   - Configure **Gmail OAuth2** credentials with permissions to read, label, and send mail.
   - Configure **Google Sheets OAuth2** credentials with read/write access to the target workbook.
   - Configure **HTTP Header Auth** credentials for OpenRouter or your chosen AI gateway.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Digital Closet Audit overview and privacy-first design model | Detailed in workflow sticky notes (Nodes 01–08) |
| Pre-activation requirements and exact Google Sheets header schema | Required columns: Account Key, Status, Service Name, Service Domain, Category, Sensitive Signal, Billing Signal, Risk Score, Confidence, Account Evidence, First Observed At, Last Observed At, Source From, Source Subject, Source Message ID, Source Thread ID, Source Link, Desired Action, Data Rights Draft, Review Reason, Last Digest At |