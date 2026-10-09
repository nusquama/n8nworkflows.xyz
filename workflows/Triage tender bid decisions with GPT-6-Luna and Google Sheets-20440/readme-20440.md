Triage tender bid decisions with GPT-6-Luna and Google Sheets

https://n8nworkflows.xyz/workflows/triage-tender-bid-decisions-with-gpt-6-luna-and-google-sheets-20440


# Triage tender bid decisions with GPT-6-Luna and Google Sheets

### 1. Workflow Overview

This workflow automates the intake, evaluation, tracking, and notification process for incoming tender submissions in a construction and engineering services company. Its primary purpose is to save the bid management team time by assessing incoming opportunities against company criteria using an AI model, logging the outcomes to Google Sheets, distributing email alerts based on the bid decision, and sending a weekly operational digest for approaching deadlines.

The workflow logic is divided into four functional blocks:
- **1.1 Intake and Validation:** Receives HTTP POST webhooks containing tender details, normalizes data fields, and validates that critical information exists. Incomplete submissions trigger a direct notification email to the bid manager and halt processing.
- **1.2 AI Assessment and Routing:** Passes validated tenders to an AI agent powered by a custom chat model (`gpt-6-luna`) to calculate fit scores, flag risks, and provide structured recommendations. Based on the output, decisions are separated into tracked opportunities vs. rejected bids, subsequently logging data to Google Sheets and dispatching specific stakeholder emails.
- **1.3 Weekly Digest:** Operates on a scheduled cron trigger every Monday at 08:00 to review active tracked tenders from Google Sheets, filters items due within a 14-day window, and formats a compiled progress email.
- **1.4 Error Handling:** Intercepts execution failures globally using an error trigger node and emails diagnostic text payload summaries to a designated administrator.

---

### 2. Block-by-Block Analysis

#### 1.1 Intake and Validation
- **Overview:** Receives raw intake payloads from webhooks (such as forms or CRMs), cleans and standardizes property formats, generates unique tracking identifiers, and ensures that mandatory parameters are present before committing resources to AI analysis.
- **Nodes Involved:** 
  - `When Tender Received`
  - `Normalize Tender`
  - `Validate Tender Input`
  - `Email Invalid Tender Notice`
- **Node Details:**
  - **When Tender Received**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (v2) — Acts as the primary HTTP entry point accepting POST requests.
    - *Configuration choices:* Configured on webhook path `tender-intake-0925`.
    - *Key expressions or variables:* None (trigger node).
    - *Input/Output:* Inputs: None. Outputs: Sends incoming payload to `Normalize Tender`.
    - *Edge cases/Potential failures:* Invalid HTTP method (must be POST), missing payload body structure.
  - **Normalize Tender**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) — Standardizes data formats, generates unique IDs, and assigns fallback defaults.
    - *Configuration choices:* Uses explicit assignments to populate standardized keys.
    - *Key expressions or variables:* 
      - `tender_id`: `={{ 'TD-' + $now.toFormat('yyyyLLdd-HHmmss') + '-' + String(Math.floor(Math.random() * 9000) + 1000) }}`
      - `received_at`: `={{ $now.toISO() }}`
      - `tender_title`, `client_name`, `due_date`, `est_value`, `tender_summary`, `source_url`: String-trimmed values extracted from `$json.body`.
      - `bid_manager_email`: Defaults to `'user@example.com'` if omitted.
    - *Input/Output:* Inputs: `When Tender Received`. Outputs: Sends normalized dataset to `Validate Tender Input`.
    - *Edge cases/Potential failures:* Malformed JSON payloads missing the `.body` container.
  - **Validate Tender Input**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.3) — Evaluates conditional state requirements.
    - *Configuration choices:* Enforces strict evaluation where `tender_title`, `due_date`, and `tender_summary` must not be empty.
    - *Key expressions or variables:* Checks `$json.tender_title`, `$json.due_date`, and `$json.tender_summary`.
    - *Input/Output:* Inputs: `Normalize Tender`. Outputs: True branch goes to `AI Assess Tender`; False branch goes to `Email Invalid Tender Notice`.
    - *Edge cases/Potential failures:* Whitespace-only strings that slip past basic type checking.
  - **Email Invalid Tender Notice**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) — Dispatches alert emails via Gmail OAuth2.
    - *Configuration choices:* Sends plain text email alerting the bid manager of missing required fields.
    - *Key expressions or variables:* Recipient: `={{ $json.bid_manager_email || 'user@example.com' }}`.
    - *Credentials:* `gmailOAuth2` (“Gmail Fresh Sep05”).
    - *Input/Output:* Inputs: `Validate Tender Input` (False branch). Outputs: Terminates branch.
    - *Edge cases/Potential failures:* Invalid OAuth2 refresh tokens or revoked Gmail API access permissions.

#### 1.2 AI Assessment and Routing
- **Overview:** Leverages an AI language model agent with specialized procurement persona instructions to structurally evaluate tenders, clean output responses via a code parser, log information to a Google Sheet, and send tailored summary emails based on the recommendation outcome.
- **Nodes Involved:**
  - `AI Assess Tender`
  - `Tender Model`
  - `Parse Tender Assessment`
  - `Route Bid Decision`
  - `Log Tracked Tender`
  - `Email Tender Assessment`
  - `Log No Bid Tender`
  - `Email No Bid Note`
- **Node Details:**
  - **AI Assess Tender**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (v3.1) — Orchestrates structured model interaction.
    - *Configuration choices:* Defines a strict system prompt instructing the model to operate as a conservative procurement analyst and output strictly valid JSON without code fences.
    - *Key expressions or variables:* Constructs prompt text dynamically from normalized fields (title, client, due date, estimated value, source link, summary description).
    - *Input/Output:* Inputs: `Validate Tender Input` (True branch), `Tender Model`. Outputs: Passes raw text string output to `Parse Tender Assessment`.
    - *Edge cases/Potential failures:* Model response hallucination or returning markdown-wrapped JSON blocks causing parsing exceptions.
  - **Tender Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) — Language model provider configuration node.
    - *Configuration choices:* Uses model `gpt-6-luna`.
    - *Credentials:* `openAiApi` (“jonathan”).
    - *Input/Output:* Inputs: None. Outputs: Provides model context to `AI Assess Tender`.
  - **Parse Tender Assessment**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — Extracts valid JSON blocks from raw LLM output strings, applies fallbacks, and standardizes schema fields.
    - *Configuration choices:* Executes custom JavaScript to locate boundaries (`{` and `}`) of JSON response, parsing fields including `fit_score`, `recommendation`, `strengths`, `risks`, `key_requirements`, and `deadline_risk`.
    - *Key expressions or variables:* Merges original normalized payload variables with processed AI assessment metadata, setting initial workflow status to `'Tracking'`.
    - *Input/Output:* Inputs: `AI Assess Tender`. Outputs: Passes processed structured object to `Route Bid Decision`.
    - *Edge cases/Potential failures:* Complete extraction failure returns an empty object, triggering fallback default values (`recommendation: 'needs_review'`, `fit_score: 0`).
  - **Route Bid Decision**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.3) — Branches workflow execution based on the final AI recommendation.
    - *Configuration choices:* Evaluates whether `$json.recommendation` is not equal to `no_bid`.
    - *Key expressions or variables:* `={{ $json.recommendation }}` compared against `no_bid`.
    - *Input/Output:* Inputs: `Parse Tender Assessment`. Outputs: True branch (`bid` or `needs_review`) routes to `Log Tracked Tender`; False branch (`no_bid`) routes to `Log No Bid Tender`.
  - **Log Tracked Tender**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.6) — Appends spreadsheet rows.
    - *Configuration choices:* Appends data to spreadsheet ID `1wrxzTGAThEC8Y91N2gmuICyzIPTPyzd6TzPczUkDmRc`, sheet name `Tenders`.
    - *Credentials:* `googleSheetsOAuth2Api` (“Feedback Analyzer Sheets”).
    - *Input/Output:* Inputs: `Route Bid Decision` (True branch). Outputs: Sends row data to `Email Tender Assessment`.
    - *Edge cases/Potential failures:* Schema mismatch, Google API quota limits, or unauthorized document scopes.
  - **Email Tender Assessment**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) — Dispatches notification emails to bid managers.
    - *Configuration choices:* Sends detailed review metrics for active opportunities.
    - *Credentials:* `gmailOAuth2` (“Gmail Fresh Sep05”).
    - *Input/Output:* Inputs: `Log Tracked Tender`. Outputs: Terminates active path.
  - **Log No Bid Tender**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.6) — Appends spreadsheet rows.
    - *Configuration choices:* Appends data to the same spreadsheet and sheet name, explicitly forcing status to `No Bid`.
    - *Credentials:* `googleSheetsOAuth2Api` (“Feedback Analyzer Sheets”).
    - *Input/Output:* Inputs: `Route Bid Decision` (False branch). Outputs: Sends row data to `Email No Bid Note`.
  - **Email No Bid Note**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) — Dispatches rationale emails.
    - *Configuration choices:* Notifies stakeholders why a specific tender was passed on.
    - *Credentials:* `gmailOAuth2` (“Gmail Fresh Sep05”).
    - *Input/Output:* Inputs: `Log No Bid Tender`. Outputs: Terminates active path.

#### 1.3 Weekly Digest
- **Overview:** Periodically queries Google Sheets to inspect ongoing tenders, filters items due within a 14-day window, and formats a combined weekly status update email.
- **Nodes Involved:**
  - `Weekly Digest Trigger`
  - `Read Tracked Tenders`
  - `Keep Tracked Tenders`
  - `Build Digest Text`
  - `Email Weekly Digest`
- **Node Details:**
  - **Weekly Digest Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) — Cron schedule executor.
    - *Configuration choices:* Configured to trigger using cron expression `0 8 * * 1` (Every Monday at 08:00 AM).
    - *Input/Output:* Inputs: None. Outputs: Triggers `Read Tracked Tenders`.
  - **Read Tracked Tenders**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.6) — Retrieves worksheet contents.
    - *Configuration choices:* Reads rows from spreadsheet ID `1wrxzTGAThEC8Y91N2gmuICyzIPTPyzd6TzPczUkDmRc`, sheet name `Tenders`.
    - *Credentials:* `googleSheetsOAuth2Api` (“Feedback Analyzer Sheets”).
    - *Input/Output:* Inputs: `Weekly Digest Trigger`. Outputs: Sends row array to `Keep Tracked Tenders`.
  - **Keep Tracked Tenders**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.3) — Filters rows based on status.
    - *Configuration choices:* Keeps rows only where `$json.status` equals `Tracking`.
    - *Input/Output:* Inputs: `Read Tracked Tenders`. Outputs: Passes filtered items to `Build Digest Text`.
  - **Build Digest Text**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — Aggregates and filters row dates mathematically.
    - *Configuration choices:* Evaluates upcoming intervals (`0 <= days <= 14`) relative to `$now`, generating a consolidated text summary.
    - *Input/Output:* Inputs: `Keep Tracked Tenders`. Outputs: Passes digest text string payload to `Email Weekly Digest`.
  - **Email Weekly Digest**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) — Sends compiled operational summaries.
    - *Configuration choices:* Emails compiled digest payload.
    - *Credentials:* `gmailOAuth2` (“Gmail Fresh Sep05”).
    - *Input/Output:* Inputs: `Build Digest Text`. Outputs: Terminates active path.

#### 1.4 Error Handling
- **Overview:** Catches unhandled exceptions across the workflow layout and forwards truncated diagnostic error metrics via email.
- **Nodes Involved:**
  - `Tender Error Trigger`
  - `Email Error Alert`
- **Node Details:**
  - **Tender Error Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.errorTrigger` (v1) — Global execution error listener.
    - *Input/Output:* Inputs: Error state. Outputs: Sends error object to `Email Error Alert`.
  - **Email Error Alert**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) — System alert dispatcher.
    - *Configuration choices:* Converts error context into a string and truncates length to 2000 characters. Sends notification to `user@example.com`.
    - *Credentials:* `gmailOAuth2` (“Gmail Fresh Sep05”).
    - *Input/Output:* Inputs: `Tender Error Trigger`. Outputs: Terminates execution.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Tender Received | webhook | Accepts incoming webhook POST requests containing tender details | None | Normalize Tender | triage tender documents and decide whether to bid with AI<br><br>### How it works<br><br>Construction and engineering firms lose days to tenders that are a bad fit. This workflow takes each incoming tender, checks the details, and gives the bid manager a clear read on whether it is worth pursuing.<br><br>1. A tender arrives by webhook (form, CRM, or pasted document summary). Required fields are checked, and incomplete submissions are flagged by email and dropped.<br>2. An AI agent reads the tender scope, scores fit from 0 to 100, and recommends bid, needs_review, or no_bid. It also lists risks, strengths, key requirements, and a deadline risk.<br>3. Tracked tenders are logged to Google Sheets. The bid manager gets an email with the score, the reasoning, and the mandatory requirements to check.<br>4. No-bid tenders are logged with a one line reason and a short email, so the team stays consistent about why the company passes.<br>5. Every Monday a digest lists the tenders being tracked that are due within 14 days, so deadlines never slip.<br><br>### Setup steps<br><br>- Connect Google Sheets for the Tenders log and Gmail for the assessment, no-bid, digest and error emails.<br>- Create a spreadsheet with a Tenders tab using the column headers from the append nodes, and set its id on the three Sheets nodes.<br>- Point the webhook at your intake form or paste tool. Set the bid manager inbox on the email nodes.<br>- Pick a chat model that returns JSON: the workflow works with OpenAI, Anthropic, or any OpenAI-compatible endpoint.<br><br>### Customization<br><br>- Change the fit threshold or the no-bid rule in Route Bid Decision.<br>- Change the digest window (default 14 days) in Build Digest Text.<br>- Swap email for WhatsApp or Slack if your bid team lives there. |
| Normalize Tender | set | Normalizes fields, generates unique IDs, and assigns defaults | When Tender Received | Validate Tender Input | triage tender documents and decide whether to bid with AI... |
| Validate Tender Input | if | Validates required fields exist before processing | Normalize Tender | AI Assess Tender, Email Invalid Tender Notice | triage tender documents and decide whether to bid with AI... |
| Email Invalid Tender Notice | gmail | Emails bid manager about missing required fields | Validate Tender Input | None | triage tender documents and decide whether to bid with AI... |
| AI Assess Tender | agent | Evaluates tender scope via AI model and returns JSON assessment | Validate Tender Input, Tender Model | Parse Tender Assessment | 2. AI bid assessment<br><br>An AI agent scores fit, flags risks and requirements, and recommends bid, needs_review or no_bid. Each outcome is logged and emailed. |
| Tender Model | lmChatOpenAi | Provides language model context (gpt-6-luna) | None | AI Assess Tender | 2. AI bid assessment... |
| Parse Tender Assessment | code | Parses JSON from AI output, sanitizes fields, and standardizes data | AI Assess Tender | Route Bid Decision | 2. AI bid assessment... |
| Route Bid Decision | if | Routes workflow based on recommendation (bid/needs_review vs no_bid) | Parse Tender Assessment | Log Tracked Tender, Log No Bid Tender | 2. AI bid assessment... |
| Log No Bid Tender | googleSheets | Appends rejected tender row with status No Bid to Google Sheets | Route Bid Decision | Email No Bid Note | 2. AI bid assessment... |
| Log Tracked Tender | googleSheets | Appends active tender row with status Tracking to Google Sheets | Route Bid Decision | Email Tender Assessment | 2. AI bid assessment... |
| Email No Bid Note | gmail | Emails rejection reason and risk summary to bid manager | Log No Bid Tender | None | 2. AI bid assessment... |
| Email Tender Assessment | gmail | Emails complete assessment score and metadata to bid manager | Log Tracked Tender | None | 2. AI bid assessment... |
| Weekly Digest Trigger | scheduleTrigger | Triggers execution weekly on Mondays at 08:00 AM | None | Read Tracked Tenders | 3. Weekly digest<br><br>Every Monday the workflow reads tracked tenders, keeps the ones due within 14 days, and emails one digest to the bid manager. |
| Read Tracked Tenders | googleSheets | Reads rows from the Tenders sheet | Weekly Digest Trigger | Keep Tracked Tenders | 3. Weekly digest... |
| Keep Tracked Tenders | if | Filters rows to keep only those with Tracking status | Read Tracked Tenders | Build Digest Text | 3. Weekly digest... |
| Build Digest Text | code | Calculates deadline intervals and formats digest message text | Keep Tracked Tenders | Email Weekly Digest | 3. Weekly digest... |
| Email Weekly Digest | gmail | Emails weekly summary of upcoming tracked tenders | Build Digest Text | None | 3. Weekly digest... |
| Tender Error Trigger | errorTrigger | Catches execution errors across the entire workflow | None | Email Error Alert | None |
| Email Error Alert | gmail | Emails truncated error payloads to administration | Tender Error Trigger | None | None |
| Sticky Note | stickyNote | General workflow documentation and setup guide | None | None | triage tender documents and decide whether to bid with AI... |
| Intake note | stickyNote | Visual section note for intake and validation block | None | None | 1. Intake and validation<br><br>A tender arrives by webhook. Required fields are checked. Invalid submissions get an email notice and are not processed. |
| Decision note | stickyNote | Visual section note for AI assessment block | None | None | 2. AI bid assessment... |
| Digest note | stickyNote | Visual section note for weekly digest block | None | None | 3. Weekly digest... |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually inside n8n:

1. **Create Intake and Normalization Nodes:**
   - Create a **Webhook** node (`When Tender Received`). Set HTTP Method to `POST` and path to `tender-intake-0925`.
   - Create a **Set** node (`Normalize Tender`). Connect `When Tender Received` to it. Configure assignments to generate fields: `tender_id` (`={{ 'TD-' + $now.toFormat('yyyyLLdd-HHmmss') + '-' + String(Math.floor(Math.random() * 9000) + 1000) }}`), `received_at` (`={{ $now.toISO() }}`), and trim string inputs for `tender_title`, `client_name`, `due_date`, `est_value`, `tender_summary`, `source_url`, and `bid_manager_email` (with a fallback string `'user@example.com'`).

2. **Add Validation and Fallback Notice:**
   - Create an **If** node (`Validate Tender Input`). Connect `Normalize Tender` to it. Set conditions to verify that `tender_title`, `due_date`, and `tender_summary` are **notEmpty**.
   - Create a **Gmail** node (`Email Invalid Tender Notice`). Connect the **False** output branch of `Validate Tender Input` to it. Set recipient to `={{ $json.bid_manager_email || 'user@example.com' }}`, subject to `={{ 'Tender intake missing fields: ' + ($json.tender_title || '(no title)') }}`, and body text listing missing required fields. Configure your `gmailOAuth2` credentials.

3. **Configure the AI Agent and Language Model:**
   - Create an **OpenAI Chat Model** node (`Tender Model`). Select model `gpt-6-luna` and configure your OpenAI API credentials (`openAiApi`).
   - Create an **AI Agent** node (`AI Assess Tender`). Connect the **True** output branch of `Validate Tender Input` to its main input, and connect `Tender Model` to its AI language model input port. Set prompt text dynamically referencing tender parameters, and configure the system message to enforce strict JSON formatting containing keys: `fit_score`, `recommendation`, `strengths`, `risks`, `key_requirements`, `deadline_risk`, and `summary`.

4. **Parse and Route AI Output:**
   - Create a **Code** node (`Parse Tender Assessment`). Connect `AI Assess Tender` to it. Add JavaScript to extract JSON blocks safely from raw response strings, set defaults for missing values, and establish a base status of `'Tracking'`.
   - Create an **If** node (`Route Bid Decision`). Connect `Parse Tender Assessment` to it. Configure a condition evaluating whether `recommendation` **notEquals** `no_bid`.

5. **Configure Decision Branches (Google Sheets & Gmail):**
   - **Tracked Branch (True):**
     - Create a **Google Sheets** node (`Log Tracked Tender`). Connect the **True** branch of `Route Bid Decision` to it. Set operation to **append**, document ID to your target Google Sheet ID, and sheet name to `Tenders`. Map all row columns (`tender_id`, `tender_title`, `client_name`, `est_value`, `due_date`, `fit_score`, `recommendation`, `status`, `strengths`, `risks`, `key_requirements`, `deadline_risk`, `received_at`). Configure `googleSheetsOAuth2Api` credentials.
     - Create a **Gmail** node (`Email Tender Assessment`). Connect `Log Tracked Tender` to it. Configure recipient, subject, and summary message templates.
   - **No-Bid Branch (False):**
     - Create a **Google Sheets** node (`Log No Bid Tender`). Connect the **False** branch of `Route Bid Decision` to it. Mirror the append configuration above, but explicitly hardcode the `status` column mapping value to `"No Bid"`.
     - Create a **Gmail** node (`Email No Bid Note`). Connect `Log No Bid Tender` to it. Configure recipient and rejection rationale message contents.

6. **Set up the Weekly Digest Block:**
   - Create a **Schedule Trigger** node (`Weekly Digest Trigger`). Set interval rule to cron expression `0 8 * * 1`.
   - Create a **Google Sheets** node (`Read Tracked Tenders`). Connect the trigger to it. Set operation to **get/read** from the `Tenders` sheet.
   - Create an **If** node (`Keep Tracked Tenders`). Connect `Read Tracked Tenders` to it. Set condition to check if `status` equals `Tracking`.
   - Create a **Code** node (`Build Digest Text`). Connect `Keep Tracked Tenders` to it. Add script logic to calculate date differences (`0 <= days <= 14`) and format an aggregate status report text string.
   - Create a **Gmail** node (`Email Weekly Digest`). Connect `Build Digest Text` to it. Set recipient variable and bind message output to `{{ $json.digest_text }}`.

7. **Establish Global Error Handling:**
   - Create an **Error Trigger** node (`Tender Error Trigger`).
   - Create a **Gmail** node (`Email Error Alert`). Connect `Tender Error Trigger` to it. Set destination recipient and message body expression to `={{ JSON.stringify($json).slice(0, 2000) }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n platform integration | [n8n Website](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Professional business assessment consultations | [Consultation Request Link](https://khmuhtadin.com/consultation/) |