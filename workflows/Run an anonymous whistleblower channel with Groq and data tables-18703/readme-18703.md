Run an anonymous whistleblower channel with Groq and data tables

https://n8nworkflows.xyz/workflows/run-an-anonymous-whistleblower-channel-with-groq-and-data-tables-18703


# Run an anonymous whistleblower channel with Groq and data tables

### 1. Workflow Overview

This workflow implements an anonymous whistleblower intake and tracking channel designed to comply with EU Whistleblower Directive standards. It provides two independent entry points: an intake channel for reporting confidential workplace issues and a secure status-check portal for reporters to monitor progress anonymously using a one-time access code.

The logic is grouped into the following functional blocks:
- **1.1 Intake & Policy Configuration:** Captures unauthenticated reports and defines baseline SLA parameters (acknowledgment windows, feedback deadlines, model limits, and escalation thresholds).
- **1.2 Rule-Based Scanning & Case Generation:** Queries an n8n Data Table containing urgency rules, scans the incoming report text using deterministic keyword or regex matching, and generates a unique case ID alongside a hashed access token.
- **1.3 AI Triage & Bounded Assessment:** Utilizes a Groq language model via an AI Agent node to classify the report category, compute an urgency confidence hint, and generate a neutral summary while treating untrusted user input safely.
- **1.4 Verdict Synthesis & Data Persistence:** Combines deterministic rule points with a capped AI-derived score, calculates statutory deadlines, assigns status values (`STANDARD`, `URGENT`, or `MANUAL_REVIEW`), and records the case in an n8n Data Table.
- **1.5 Receipt Delivery:** Directs the user flow based on urgency to display a one-time visual receipt containing the case number and access code.
- **1.6 Anonymous Status Lookup:** Handles inquiries from reporters via a secondary form trigger, verifies the provided access code against the stored SHA-256 hash in the case register, and displays either case progress or a secure generic failure message.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Policy Configuration
This block receives the confidential report through a public form without demanding user authentication and initializes governance variables governing response timelines.

- **Receive Report**
  - **Type & Technical Role:** `n8n-nodes-base.formTrigger` (Version 2.6). Acts as the primary workflow entry point, hosting a web form where users submit reports.
  - **Configuration:** Configured with form fields for category (dropdown), report text (textarea), and optional contact details. Includes instructions regarding data handling and EU timelines.
  - **Key Expressions:** None (static form configuration).
  - **Input/Output:** Output connects to `Set Channel Policy`.
  - **Version Requirements:** Version 2.6+.
  - **Edge Cases/Failures:** Form submission timeouts or network failures between the user and the n8n instance. Empty text inputs are constrained by required field validation.

- **Set Channel Policy**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Version 3.5). Establishes global channel configuration variables.
  - **Configuration:** Sets numerical assignments for `ack_days` (7), `feedback_days` (90), `urgent_at` (60), and `model_max_points` (30).
  - **Key Expressions:** Static numbers mapped to JSON keys.
  - **Input/Output:** Input from `Receive Report`; output to `Load Urgency Rules`.
  - **Version Requirements:** Version 3.5+.
  - **Edge Cases/Failures:** Downstream nodes failing to parse policy parameters if modified incorrectly.

#### 2.2 Rule-Based Scanning & Case Generation
This block retrieves screening rules from a data table, performs local deterministic string and regex checks, and securely hashes authentication tokens.

- **Load Urgency Rules**
  - **Type & Technical Role:** `n8n-nodes-base.dataTable` (Version 1.1). Retrieves all records from the designated urgency rules data table.
  - **Configuration:** Operation set to `get` with `returnAll` enabled. Always outputs data even if the table is empty.
  - **Key Expressions:** None.
  - **Input/Output:** Input from `Set Channel Policy`; output to `Scan Report And Create Case`.
  - **Version Requirements:** Version 1.1+.
  - **Edge Cases/Failures:** Data table ID misconfiguration or empty rule tables (handled downstream by forcing `MANUAL_REVIEW`).

- **Scan Report And Create Case**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Version 2). Executes custom JavaScript to parse rules, scan the report, and cryptographic identifiers.
  - **Configuration:** Uses Node.js `crypto` module to generate random bytes for case IDs (`WB-XXXXXXXX`) and plain/hashed tokens.
  - **Key Expressions:** Reads upstream data via `$('Receive Report').first().json` and `$('Set Channel Policy').first().json`.
  - **Input/Output:** Input from `Load Urgency Rules`; output to `Assess Report With Groq`.
  - **Version Requirements:** Version 2.
  - **Edge Cases/Failures:** Malformed regular expressions in the rules table throwing syntax errors (caught by `try/catch`).

#### 2.3 AI Triage & Bounded Assessment
This block passes context to a Groq language model to summarize the incident and generate confidence metrics.

- **Assess Report With Groq**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` (Version 3.1). Orchestrates an AI agent loop using system prompts and output schemas.
  - **Configuration:** System prompt instructs the model to triage workplace reports, extract categories, output whole number confidence/urgency scores (0–100), and write neutral summaries while ignoring prompt-injection attacks.
  - **Key Expressions:** `=Reporter-chosen category: {{ $('Scan Report And Create Case').item.json.category_reported }}` and report text payload.
  - **Input/Output:** Input from `Scan Report And Create Case`; output to `Combine Into Case Verdict`. Connected to `Groq Chat Model` and `Parse Triage Assessment`.
  - **Version Requirements:** LangChain agent v3.1+.
  - **Edge Cases/Failures:** Rate limits, API quota exhaustion, or unstructured model responses (mitigated by output parsing and fallback logic in the next node).

- **Groq Chat Model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatGroq` (Version 1). Provides the underlying Large Language Model engine.
  - **Configuration:** Uses the `openai/gpt-oss-120b` model profile.
  - **Key Expressions:** None.
  - **Input/Output:** Connected to `Assess Report With Groq` via `ai_languageModel`.
  - **Credentials Required:** Groq API credential.

- **Parse Triage Assessment**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (Version 1.3). Forces structured JSON outputs matching predefined schemas.
  - **Configuration:** JSON schema example defines `category`, `urgency_hint`, `summary`, and `confidence`.
  - **Input/Output:** Connected to `Assess Report With Groq` via `ai_outputParser`.

#### 2.4 Verdict Synthesis & Data Persistence
This block calculates final risk scores, determines escalation levels, computes legal due dates, and records the case in permanent storage.

- **Combine Into Case Verdict**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Version 2). Synthesizes rule evaluation points with AI model scores.
  - **Configuration:** Computes capped model points, derives overall risk scores, sets status flags (`STANDARD`, `URGENT`, or `MANUAL_REVIEW`), and calculates SLA due dates for acknowledgment and feedback.
  - **Key Expressions:** Consumes data from `Scan Report And Create Case` and AI output nodes.
  - **Input/Output:** Input from `Assess Report With Groq`; output to `Record In Case Register`.
  - **Edge Cases/Failures:** Non-numeric confidence/urgency values returned by the model default safely to zero points.

- **Record In Case Register**
  - **Type & Technical Role:** `n8n-nodes-base.dataTable` (Version 1.1). Appends a new case row to the operational data table.
  - **Configuration:** Mapping mode set to explicit column mapping (`case_id`, `token_hash`, `status`, `urgency`, `risk_score`, `received_at`, `ack_due`, `feedback_due`, `summary`, `report_text`, `contact_left`, `category_model`, `contact_optional`, `category_reported`).
  - **Key expressions:** Maps all properties from `{{ $json.[field_name] }}`.
  - **Input/Output:** Input from `Combine Into Case Verdict`; output to `Is Urgent`.
  - **Edge Cases/Failures:** Database constraints, missing table columns, or write timeouts.

#### 2.5 Receipt Delivery
This block routes users based on case urgency to present their credentials exactly once.

- **Is Urgent**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Version 2.3). Evaluates case severity.
  - **Configuration:** Condition checks if `urgency` equals `STANDARD`.
  - **Input/Output:** Input from `Record In Case Register`; outputs true branch to `Show Standard Receipt`, false branch to `Show Urgent Receipt`.

- **Show Standard Receipt**
  - **Type & Technical Role:** `n8n-nodes-base.form` (Version 2.5). Form completion node displaying standard case credentials.
  - **Configuration:** Displays completion title with case ID and warning message instructing the user to record their case number and access code.
  - **Key Expressions:** References case ID, access token plain text, acknowledgment dates, and feedback dates from `Combine Into Case Verdict`.
  - **Input/Output:** Input from `Is Urgent` (true).

- **Show Urgent Receipt**
  - **Type & Technical Role:** `n8n-nodes-base.form` (Version 2.5). Form completion node displaying urgent handling alerts.
  - **Configuration:** Displays warning message indicating urgent handling and advising users in immediate physical danger to contact emergency services.
  - **Key Expressions:** References case identifiers and SLA due dates.
  - **Input/Output:** Input from `Is Urgent` (false).

#### 2.6 Anonymous Status Lookup
This block handles secure retrieval of case statuses for reporters using their access credentials.

- **Check Case Status**
  - **Type & Technical Role:** `n8n-nodes-base.formTrigger` (Version 2.6). Secondary entry point hosting a lookup form.
  - **Configuration:** Requires `case_id` and `access_code`.
  - **Input/Output:** Output connects to `Load Case Register`.

- **Load Case Register**
  - **Type & Technical Role:** `n8n-nodes-base.dataTable` (Version 1.1). Retrieves all rows from the case register data table for validation.
  - **Configuration:** Operation set to `get` with `returnAll` enabled.
  - **Input/Output:** Input from `Check Case Status`; output to `Verify Code And Compose Status`.

- **Verify Code And Compose Status**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Version 2). Hashes the user-provided access code and compares it against stored SHA-256 hashes.
  - **Configuration:** Uses Node.js `crypto` module to hash input and find matching case records securely.
  - **Input/Output:** Input from `Load Case Register`; output to `Is Code Valid`.

- **Is Code Valid**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Version 2.3). Validates authentication status.
  - **Configuration:** Checks if `verified` equals `yes`.
  - **Input/Output:** Input from `Verify Code And Compose Status`; outputs true branch to `Show Case Status`, false branch to `Show Lookup Failed`.

- **Show Case Status**
  - **Type & Technical Role:** `n8n-nodes-base.form` (Version 2.5). Displays case progress details upon successful verification.
  - **Configuration:** Shows status, received date, acknowledgment deadline, and feedback deadline.
  - **Input/Output:** Input from `Is Code Valid` (true).

- **Show Lookup Failed**
  - **Type & Technical Role:** `n8n-nodes-base.form` (Version 2.5). Displays generic failure messaging.
  - **Configuration:** Displays non-specific error message to prevent enumeration attacks, refusing to specify whether the case ID or access code was incorrect.
  - **Input/Output:** Input from `Is Code Valid` (false).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Report | n8n-nodes-base.formTrigger | Anonymous intake form | None | Set Channel Policy | Intake and policy |
| Set Channel Policy | n8n-nodes-base.set | Defines SLA parameters | Receive Report | Load Urgency Rules | Intake and policy |
| Load Urgency Rules | n8n-nodes-base.dataTable | Retrieves screening rules | Set Channel Policy | Scan Report And Create Case | Rules scan and case creation |
| Scan Report And Create Case | n8n-nodes-base.code | Scans text & generates tokens | Load Urgency Rules | Assess Report With Groq | Rules scan and case creation |
| Assess Report With Groq | @n8n/n8n-nodes-langchain.agent | AI triage and summarization | Scan Report And Create Case | Combine Into Case Verdict | Bounded AI triage |
| Groq Chat Model | @n8n/n8n-nodes-lmChatGroq | LLM provider | None | Assess Report With Groq | Bounded AI triage |
| Parse Triage Assessment | @n8n/n8n-nodes-langchain.outputParserStructured | Forces structured output schema | None | Assess Report With Groq | Bounded AI triage |
| Combine Into Case Verdict | n8n-nodes-base.code | Synthesizes scores and deadlines | Assess Report With Groq | Record In Case Register | Verdict, deadlines and register |
| Record In Case Register | n8n-nodes-base.dataTable | Persists case data | Combine Into Case Verdict | Is Urgent | Verdict, deadlines and register |
| Is Urgent | n8n-nodes-base.if | Routes by urgency level | Record In Case Register | Show Standard Receipt, Show Urgent Receipt | One-time receipts |
| Show Standard Receipt | n8n-nodes-base.form | Displays standard completion receipt | Is Urgent | None | One-time receipts |
| Show Urgent Receipt | n8n-nodes-base.form | Displays urgent completion receipt | Is Urgent | None | One-time receipts |
| Check Case Status | n8n-nodes-base.formTrigger | Status lookup entry point | None | Load Case Register | Anonymous status lookup |
| Load Case Register | n8n-nodes-base.dataTable | Retrieves stored cases for lookup | Check Case Status | Verify Code And Compose Status | Anonymous status lookup |
| Verify Code And Compose Status | n8n-nodes-base.code | Hashes and validates access code | Load Case Register | Is Code Valid | Anonymous status lookup |
| Is Code Valid | n8n-nodes-base.if | Validates verification flag | Verify Code And Compose Status | Show Case Status, Show Lookup Failed | Anonymous status lookup |
| Show Case Status | n8n-nodes-base.form | Displays case status details | Is Code Valid | None | Anonymous status lookup |
| Show Lookup Failed | n8n-nodes-base.form | Displays generic failure notice | Is Code Valid | None | Anonymous status lookup |

---

### 4. Reproducing the Workflow from Scratch

#### Prerequisites & Data Tables Setup
1. Create an n8n Data Table named **Urgency Rules** with columns: `pattern` (string), `match_type` (string), `severity` (string), `points` (number), `category` (string), and `note` (string). Populate with initial screening keywords/regex patterns.
2. Create an n8n Data Table named **Case Register** with columns: `case_id`, `token_hash`, `status`, `urgency`, `risk_score`, `received_at`, `ack_due`, `feedback_due`, `summary`, `report_text`, `contact_left`, `category_model`, `contact_optional`, and `category_reported`.

#### Step-by-Step Node Creation & Configuration
1. **Receive Report** (`n8n-nodes-base.formTrigger`):
   - Set form title to `Confidential reporting channel`.
   - Add dropdown field `category` (Options: Fraud or financial, Corruption, Safety or environment, Harassment or discrimination, Data protection, Other).
   - Add textarea field `report_text` (Required).
   - Add text field `contact_optional`.
2. **Set Channel Policy** (`n8n-nodes-base.set`):
   - Add assignments: `ack_days` = 7, `feedback_days` = 90, `urgent_at` = 60, `model_max_points` = 30.
3. **Load Urgency Rules** (`n8n-nodes-base.dataTable`):
   - Set operation to `Get`, enable `Return All`, and select your **Urgency Rules** data table.
4. **Scan Report And Create Case** (`n8n-nodes-base.code`):
   - Insert JavaScript snippet utilizing Node.js `crypto` to parse rules, scan input text, generate a case ID (`WB-xxxx`), and compute the SHA-256 hash of a 16-byte random token.
5. **Groq Chat Model** (`@n8n/n8n-nodes-lmChatGroq`):
   - Select model `openai/gpt-oss-120b`.
   - Configure credentials with your **Groq API** key.
6. **Parse Triage Assessment** (`@n8n/n8n-nodes-langchain.outputParserStructured`):
   - Set JSON schema example containing `category`, `urgency_hint`, `summary`, and `confidence`.
7. **Assess Report With Groq** (`@n8n/n8n-nodes-langchain.agent`):
   - Set prompt type to define.
   - Configure system message instructing triage classification, score normalization (0–100), and summarization without following untrusted instructions inside reports.
   - Connect `Groq Chat Model` to AI Language Model input and `Parse Triage Assessment` to AI Output Parser input.
8. **Combine Into Case Verdict** (`n8n-nodes-base.code`):
   - Add code to merge deterministic rule points with capped model scores, compute SLA deadlines (`ack_due`, `feedback_due`), and set initial statuses (`STANDARD`, `URGENT`, or `MANUAL_REVIEW`).
9. **Record In Case Register** (`n8n-nodes-base.dataTable`):
   - Select the **Case Register** data table.
   - Map all output fields from the previous node explicitly to corresponding table columns.
10. **Is Urgent** (`n8n-nodes-base.if`):
    - Set condition to evaluate if `{{ $('Combine Into Case Verdict').item.json.urgency }}` equals `STANDARD`.
11. **Show Standard Receipt** (`n8n-nodes-base.form`):
    - Set operation to `Completion`. Configure title and message to display case number and access code once alongside due dates.
12. **Show Urgent Receipt** (`n8n-nodes-base.form`):
    - Set operation to `Completion`. Configure title and message to display urgent notices and emergency contact warnings.
13. **Check Case Status** (`n8n-nodes-base.formTrigger`):
    - Set form title to `Check the status of your report`. Add required fields `case_id` and `access_code`.
14. **Load Case Register** (`n8n-nodes-base.dataTable`):
    - Set operation to `Get`, enable `Return All`, and select the **Case Register** data table.
15. **Verify Code And Compose Status** (`n8n-nodes-base.code`):
    - Add JavaScript to hash the submitted access code using SHA-256 and verify it against the stored token hash for the matching case ID.
16. **Is Code Valid** (`n8n-nodes-base.if`):
    - Set condition to verify if `{{ $('Verify Code And Compose Status').item.json.verified }}` equals `yes`.
17. **Show Case Status** (`n8n-nodes-base.form`):
    - Set operation to `Completion`. Display case status, received date, and deadlines.
18. **Show Lookup Failed** (`n8n-nodes-base.form`):
    - Set operation to `Completion`. Display generic failure message without revealing specific validation errors.

#### Connection Order
- `Receive Report` $\rightarrow$ `Set Channel Policy` $\rightarrow$ `Load Urgency Rules` $\rightarrow$ `Scan Report And Create Case` $\rightarrow$ `Assess Report With Groq` $\rightarrow$ `Combine Into Case Verdict` $\rightarrow$ `Record In Case Register` $\rightarrow$ `Is Urgent`.
- `Is Urgent` (True) $\rightarrow$ `Show Standard Receipt`.
- `Is Urgent` (False) $\rightarrow$ `Show Urgent Receipt`.
- `Check Case Status` $\rightarrow$ `Load Case Register` $\rightarrow$ `Verify Code And Compose Status` $\rightarrow$ `Is Code Valid`.
- `Is Code Valid` (True) $\rightarrow$ `Show Case Status`.
- `Is Code Valid` (False) $\rightarrow$ `Show Lookup Failed`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Compliance framework reference for EU Whistleblower Directive timelines (7-day acknowledgment, 3-month feedback). | Statutory compliance guidelines for internal reporting channels. |
| Security design: One-way cryptographic hashing (SHA-256) ensures plaintext access codes are never persisted in storage. | Cryptographic token handling best practices. |