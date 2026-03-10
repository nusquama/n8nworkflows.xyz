Analyze legal contracts with GPT-4.1 and manage cases in Google Sheets and Slack

https://n8nworkflows.xyz/workflows/analyze-legal-contracts-with-gpt-4-1-and-manage-cases-in-google-sheets-and-slack-12740


# Analyze legal contracts with GPT-4.1 and manage cases in Google Sheets and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Analyze legal contracts with GPT-4.1 and manage cases in Google Sheets and Slack  
**Workflow name (JSON):** Legal Case Intake and Contract Analysis Automation

**Purpose:**  
Automates intake of a new legal case + contract document via webhook, extracts text from a PDF, analyzes the contract with GPT-4.1 (structured JSON output), routes the case by type, logs it to Google Sheets, notifies the responsible attorney in Slack, sends a summary email to the client, generates a PDF review, schedules a follow-up in Google Calendar, and optionally pushes the case into practice management software via HTTP API.

**Primary use cases:**
- Law firm intake automation for multiple practice areas (Litigation/Corporate/Family Law)
- Fast contract clause extraction and risk spotting with consistent structured output
- Lightweight “case CRM” via Google Sheets + Slack notifications + calendar follow-ups
- Optional system-of-record sync to a practice management platform

### 1.1 Webhook Intake & Configuration
Receives a POST request with case metadata and a PDF; sets internal configuration values (attorney Slack IDs, Google IDs).

### 1.2 Data Normalization & PDF Text Extraction
Normalizes key fields and extracts text from an uploaded PDF.

### 1.3 AI Contract Analysis (GPT + Structured Output)
Sends contract text to a LangChain Agent powered by OpenAI chat model; enforces a JSON schema via structured output parser.

### 1.4 Routing, Logging, Notification, and Client Communication
Routes by case type; appends/updates Google Sheets; posts Slack message to the attorney; emails summary to client; generates a PDF report.

### 1.5 Scheduling & External System Integration
Creates a follow-up event in Google Calendar and calls an external practice management API endpoint.

---

## 2. Block-by-Block Analysis

### Block A — Webhook Intake & Global Configuration
**Overview:** Accepts new case submissions via HTTP POST and injects environment-specific IDs used later (Slack users, Google Sheet doc, calendar).  
**Nodes involved:**  
- Case/Contract Intake Webhook  
- Workflow Configuration

#### Node: Case/Contract Intake Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook). Entry point; receives intake payload.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `case-intake` (full URL depends on n8n base URL and webhook mode)
  - **Response Mode:** `lastNode` (client receives the output of the final executed node)
- **Expected input (practical):**
  - `body.clientName`, `body.clientEmail`, `body.caseType`, `body.caseDescription`
  - A **PDF file** (must be provided in a way that `Extract Contract Text` can read; typically multipart/form-data binary)
- **Outputs:** Passes the received JSON/binary forward.
- **Connections:** Outputs to **Workflow Configuration**.
- **Failure/edge cases:**
  - Missing or malformed body fields → downstream expressions may become empty.
  - If no PDF binary is provided (or wrong form field name), PDF extraction will fail.
  - `responseMode: lastNode` means any downstream failure may prevent a response to the caller (caller may see timeout or error).

#### Node: Workflow Configuration
- **Type / role:** `Set` node; stores reusable configuration constants in the execution data.
- **Key configuration choices:**
  - Sets:
    - `litigationAttorney` = placeholder Slack user/channel ID
    - `corporateAttorney` = placeholder Slack user/channel ID
    - `familyLawAttorney` = placeholder Slack user/channel ID
    - `sheetsDocumentId` = placeholder Google Sheets document ID
    - `calendarId` = placeholder Google Calendar ID
  - **Include other fields:** enabled (keeps webhook payload).
- **Connections:** Receives from webhook; outputs to **Normalize Client and Case Info**.
- **Failure/edge cases:**
  - Placeholders not replaced → Slack/Sheets/Calendar nodes will fail (invalid IDs).
  - If IDs are intended as Slack *user IDs* but Slack node expects a *channel ID*, delivery will fail (see Slack node details).

---

### Block B — Normalize Fields & Extract Contract Text
**Overview:** Ensures consistent naming and defaults for case fields, then extracts text from a PDF so the AI can analyze it.  
**Nodes involved:**  
- Normalize Client and Case Info  
- Extract Contract Text

#### Node: Normalize Client and Case Info
- **Type / role:** `Set` node; maps incoming webhook body fields to normalized top-level fields.
- **Key expressions/variables:**
  - `clientName` = `{{ $json.body.clientName || "Unknown Client" }}`
  - `clientEmail` = `{{ $json.body.clientEmail }}`
  - `caseType` = `{{ $json.body.caseType }}`
  - `caseDescription` = `{{ $json.body.caseDescription }}`
  - `intakeDate` = `{{ $now.toISO() }}`
- **Include other fields:** enabled.
- **Connections:** Outputs to **Extract Contract Text**.
- **Failure/edge cases:**
  - If webhook payload is not under `body` (depends on webhook settings and content-type), fields will be empty.
  - `clientEmail` missing → Google Sheets matching on `clientEmail` and email sending will fail or create bad records.

#### Node: Extract Contract Text
- **Type / role:** `Extract From File` node; extracts text from PDF.
- **Key configuration:**
  - **Operation:** PDF extraction
- **Input expectations:**
  - A binary PDF must be available on the incoming item (commonly under a binary property like `data`; exact property depends on webhook/form config).
- **Outputs:**
  - Produces `text` (extracted PDF text) used by AI node via `{{ $json.text }}`.
- **Connections:** Outputs to **Analyze Contract and Extract Clauses**.
- **Failure/edge cases:**
  - Scanned/image PDFs without OCR → extraction may yield empty/garbled text.
  - Large PDFs → may hit memory/time limits or produce too much text for the model.
  - If binary property name mismatches, node errors “No binary data found”.

---

### Block C — AI Contract Analysis (OpenAI + Structured Output)
**Overview:** Uses a LangChain Agent with an OpenAI Chat Model and a structured output parser to extract contract details as schema-valid JSON.  
**Nodes involved:**  
- Analyze Contract and Extract Clauses  
- OpenAI Chat Model  
- Structured Output Parser

#### Node: Analyze Contract and Extract Clauses
- **Type / role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`); orchestrates LLM call and output parsing.
- **Key configuration choices:**
  - **Prompt input text:** `Contract text to analyze: {{ $json.text }}`
  - **System message:** legal analysis instructions; requests parties, contract type, obligations, dates, payment terms, termination, liability/indemnification, dispute resolution, governing law, risky clauses.
  - **Output parser:** enabled (`hasOutputParser: true`) and connected to Structured Output Parser node.
- **Inputs / outputs:**
  - Input: extracted PDF text + normalized case fields (passed through).
  - Output: typically adds `output` object (structured result) when parsing succeeds. Downstream nodes reference `$json.output.parties`, `$json.output.contractType`, etc.
- **Connections:**
  - Receives main input from **Extract Contract Text**
  - Receives AI language model input from **OpenAI Chat Model** (ai_languageModel connection)
  - Receives output parser from **Structured Output Parser** (ai_outputParser connection)
  - Main output to **Route by Case Type**
- **Failure/edge cases:**
  - If extracted text is empty/too long → low-quality output or token-limit errors.
  - If the model returns JSON that doesn’t match schema → parser fails and the workflow errors.
  - Legal content ambiguity: parties/dates may be missing; schema requires `parties` and `contractType`, so the agent must infer or return placeholders—otherwise it fails.

#### Node: OpenAI Chat Model
- **Type / role:** OpenAI chat LLM provider node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`).
- **Key configuration:**
  - **Model:** `gpt-4.1-mini`
  - Credentials: “n8n free OpenAI API credits” (must be valid in your instance)
- **Connections:** Supplies model to **Analyze Contract and Extract Clauses** via `ai_languageModel`.
- **Failure/edge cases:**
  - Invalid/expired OpenAI credentials or insufficient quota → authentication/billing errors.
  - Model availability changes → execution failures if model is deprecated or blocked.

#### Node: Structured Output Parser
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`); enforces a JSON schema.
- **Key configuration:**
  - **Schema type:** Manual
  - **Schema:** object with properties:
    - `parties` (array of strings) **required**
    - `contractType` (string) **required**
    - `keyObligations` (array of `{party, obligation}`)
    - `importantDates` (`effectiveDate`, `terminationDate`, `renewalDates[]`)
    - `paymentTerms`, `terminationClauses`, `liabilityIndemnification`, `disputeResolution`, `governingLaw`
    - `riskyClauses` (array of strings)
- **Connections:** Provides parser to **Analyze Contract and Extract Clauses** via `ai_outputParser`.
- **Failure/edge cases:**
  - Any schema mismatch (wrong types, missing required fields) → parser throws error.
  - If model returns extra prose outside JSON → parser may fail unless the agent reliably formats output.

---

### Block D — Routing, Logging, Notifications, Client Email, PDF Generation
**Overview:** Routes by case type, writes to Google Sheets, alerts the responsible attorney in Slack with key analysis, sends an email summary to the client, and generates a PDF from HTML.  
**Nodes involved:**  
- Route by Case Type  
- Log Case and Contract Info  
- Notify Responsible Attorney  
- Send Summary to Client  
- Generate Contract Review PDF

#### Node: Route by Case Type
- **Type / role:** `Switch` node; routes execution based on `caseType`.
- **Key configuration:**
  - Rules:
    - Output “Litigation” if `{{ $json.caseType }}` equals `Litigation`
    - Output “Corporate” if equals `Corporate`
    - Output “Family Law” if equals `Family Law`
  - Fallback output renamed to **Other**
- **Connections:** In this workflow, only one outgoing connection is defined:
  - Switch → **Log Case and Contract Info** (wired from the node’s main output; practically this means the connected output path is the one used in the canvas wiring).
- **Failure/edge cases:**
  - Case type mismatch (e.g., “FamilyLaw”, lowercase, trailing spaces) → falls into “Other”; may still proceed depending on wiring, but attorney selection later may default unexpectedly.
  - If you intended different downstream logic per branch, the current connections do not show separate branch handling (only one downstream node is connected).

#### Node: Log Case and Contract Info
- **Type / role:** `Google Sheets` node; appends or updates a record in a “Case Log” sheet.
- **Key configuration:**
  - Operation: **Append or Update**
  - Sheet: `Case Log`
  - Document ID: from config: `{{ $('Workflow Configuration').first().json.sheetsDocumentId }}`
  - Matching column: `clientEmail`
  - Mapping mode: auto-map input data (relies on column names matching fields)
- **Connections:** Outputs to **Notify Responsible Attorney**.
- **Failure/edge cases:**
  - Sheet/tab name missing (“Case Log” not present) → error.
  - `clientEmail` empty → update matching fails; may append duplicates or fail.
  - Auto-map requires columns that match incoming keys; otherwise columns remain blank.

#### Node: Notify Responsible Attorney
- **Type / role:** `Slack` node; posts a rich Block Kit message to a destination selected by case type.
- **Key configuration:**
  - **Message type:** block
  - **Blocks:** includes header, client fields, case description, and contract summary:
    - Parties: `{{ $json.output.parties.join(', ') }}`
    - Contract Type: `{{ $json.output.contractType }}`
    - Payment Terms: `{{ $json.output.paymentTerms }}`
  - **Channel/recipient selection (`channelId`):**
    - Litigation → `Workflow Configuration.litigationAttorney`
    - Corporate → `Workflow Configuration.corporateAttorney`
    - Else → `Workflow Configuration.familyLawAttorney`
- **Important integration note:** This field is named `channelId`, so it typically expects a **Slack channel ID** (or DM channel ID). If you place a **Slack user ID** here (as the placeholders suggest), posting may fail unless the node supports direct user messaging by user ID in this mode.
- **Connections:** Outputs to **Send Summary to Client**.
- **Failure/edge cases:**
  - Invalid Slack token/workspace permissions.
  - Wrong destination identifier type (user ID vs channel ID).
  - If AI output missing `output.parties` etc. due to parsing failure, expressions error (but usually the workflow would have already failed earlier at the parser).

#### Node: Send Summary to Client
- **Type / role:** `Email Send` node; emails an HTML summary to the client.
- **Key configuration:**
  - To: `{{ $json.clientEmail }}`
  - Subject: `Contract Review Summary - {{ $json.clientName }}`
  - From: placeholder “Law firm email address”
  - HTML body uses:
    - `output.parties.join(', ')`
    - `output.contractType`
    - `output.paymentTerms`
    - `output.governingLaw`
- **Connections:** Outputs to **Generate Contract Review PDF**.
- **Failure/edge cases:**
  - SMTP/Email credentials not configured in n8n (this node typically requires an email account/SMTP setup).
  - Invalid `fromEmail` domain or SPF/DKIM issues → deliverability problems.
  - Missing client email → send failure.

#### Node: Generate Contract Review PDF
- **Type / role:** `Convert to File` node; converts HTML to a PDF file.
- **Key configuration:**
  - Operation: `html` → renders HTML and outputs a PDF binary
  - Filename: `Contract_Review_<clientName>_<yyyy-MM-dd>.pdf`
- **Connections:** Outputs to **Schedule Case Follow-ups**.
- **Failure/edge cases:**
  - HTML rendering limitations (CSS support, fonts).
  - If upstream node doesn’t provide HTML content in the expected field for this node, the PDF may be blank or node may error (ensure the convert node is configured to use the right input property; here it relies on node defaults).

---

### Block E — Scheduling & Practice Management Integration
**Overview:** Creates a follow-up event 7 days after intake, then POSTs the case + AI analysis to an external practice management system.  
**Nodes involved:**  
- Schedule Case Follow-ups  
- Integrate with Practice Management Software

#### Node: Schedule Case Follow-ups
- **Type / role:** `Google Calendar` node; creates an event.
- **Key configuration:**
  - Calendar ID: `{{ $('Workflow Configuration').first().json.calendarId }}`
  - Start: `{{ $now.plus({ days: 7 }).toISO() }}`
  - End: `{{ $now.plus({ days: 7, hours: 1 }).toISO() }}`
  - Summary: `Case Follow-up: <clientName> - <caseType>`
  - Description includes intake date, client name/email, case type
- **Connections:** Outputs to **Integrate with Practice Management Software**.
- **Failure/edge cases:**
  - Wrong calendar ID / missing permissions.
  - Time zone assumptions: `$now` uses server timezone unless configured; could schedule at an unexpected local time.

#### Node: Integrate with Practice Management Software
- **Type / role:** `HTTP Request` node; posts data to an external API endpoint (generic credential auth).
- **Key configuration:**
  - Method: POST
  - URL: placeholder “Practice management software API endpoint”
  - Authentication: `genericCredentialType` (you must configure the credential in n8n)
  - Body: JSON (declared as JSON body)
- **Critical implementation detail (expression correctness):**
  - The configured JSON body is:
    ```js
    {
      "clientName": {{ $json.clientName }},
      "clientEmail": {{ $json.clientEmail }},
      "caseType": {{ $json.caseType }},
      "caseDescription": {{ $json.caseDescription }},
      "intakeDate": {{ $json.intakeDate }},
      "contractAnalysis": {{ $json.output }}
    }
    ```
  - In n8n expressions, **strings must be quoted** to form valid JSON unless the UI builds an object safely. As written, this can produce invalid JSON (e.g., `"clientName": John Doe` instead of `"clientName": "John Doe"`).
  - Safer patterns:
    - Build a real object in an expression (if supported by the node’s JSON mode), or
    - Wrap string values with `{{$json.clientName}}` inside quotes, or
    - Use `={{ { clientName: $json.clientName, ... } }}` if the field accepts an object expression.
- **Connections:** Last node (webhook response returns this node’s output due to `lastNode` response mode).
- **Failure/edge cases:**
  - Auth misconfiguration → 401/403.
  - Endpoint downtime/timeouts.
  - Invalid JSON due to quoting → 400 Bad Request.
  - Sending `contractAnalysis` as an object may require `JSON.stringify()` depending on API expectations.

---

### Block F — Documentation / Sticky Notes (Canvas Annotations)
**Overview:** Non-executing notes that describe sections and setup.  
**Nodes involved:** Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4

- These nodes do not affect execution; they annotate groups:
  - “Webhook & Config”
  - “AI logic & Routing”
  - “Log & Notify”
  - “Scheduling”
  - “Main / Setup / Author / Company / Contact”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Case/Contract Intake Webhook | Webhook | Entry point for case + contract intake via POST | — | Workflow Configuration | ## Webhook & Config |
| Workflow Configuration | Set | Stores attorney/channel IDs and Google IDs | Case/Contract Intake Webhook | Normalize Client and Case Info | ## Webhook & Config |
| Normalize Client and Case Info | Set | Normalizes incoming fields; sets intake timestamp | Workflow Configuration | Extract Contract Text | ## Webhook & Config |
| Extract Contract Text | Extract From File | Extracts text from uploaded PDF | Normalize Client and Case Info | Analyze Contract and Extract Clauses | ## AI logic & Routing |
| Analyze Contract and Extract Clauses | LangChain Agent | GPT-driven contract analysis with structured JSON output | Extract Contract Text (+ AI model/parser inputs) | Route by Case Type | ## AI logic & Routing |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | Provides `gpt-4.1-mini` model to agent | — | Analyze Contract and Extract Clauses (ai_languageModel) | ## AI logic & Routing |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforces contract analysis JSON schema | — | Analyze Contract and Extract Clauses (ai_outputParser) | ## AI logic & Routing |
| Route by Case Type | Switch | Branches by `caseType` (Litigation/Corporate/Family Law/Other) | Analyze Contract and Extract Clauses | Log Case and Contract Info | ## AI logic & Routing |
| Log Case and Contract Info | Google Sheets | Append/update case row in “Case Log” keyed by clientEmail | Route by Case Type | Notify Responsible Attorney | ## Log & Notify |
| Notify Responsible Attorney | Slack | Sends Block Kit message with case + AI summary to selected destination | Log Case and Contract Info | Send Summary to Client | ## Log & Notify |
| Send Summary to Client | Email Send | Emails HTML summary to client | Notify Responsible Attorney | Generate Contract Review PDF | ## Log & Notify |
| Generate Contract Review PDF | Convert to File | Converts HTML to PDF file | Send Summary to Client | Schedule Case Follow-ups | ## Log & Notify |
| Schedule Case Follow-ups | Google Calendar | Creates a follow-up event 7 days later | Generate Contract Review PDF | Integrate with Practice Management Software | ## Scheduling |
| Integrate with Practice Management Software | HTTP Request | Posts case + analysis to external practice management API | Schedule Case Follow-ups | — | ## Scheduling |
| Sticky Note | Sticky Note | Canvas annotation | — | — | ## Webhook & Config |
| Sticky Note1 | Sticky Note | Canvas annotation | — | — | ## AI logic & Routing |
| Sticky Note2 | Sticky Note | Canvas annotation | — | — | ## Log & Notify |
| Sticky Note3 | Sticky Note | Canvas annotation | — | — | ## Scheduling  |
| Sticky Note4 | Sticky Note | Canvas annotation (overview, setup, author/contact) | — | — | ## Main\nAutomates law firm case and contract management. AI extracts key contract clauses, routes cases to appropriate attorneys, logs details, sends client summaries, and schedules follow-ups.\n\n## Setup\n1. Connect Webhook for new contracts/cases\n2. Connect OpenAI for contract analysis\n3. Connect Google Sheets for case logging\n4. Connect Slack for attorney notifications\n5. Connect email for client summaries\n6. Connect PDF generation node\n7. Connect calendar for follow-ups\n8. Optional integration with practice management software\n\n**Author:** Hyrum Hurst, AI Automation Engineer\n**Company:** QuarterSmart\n**Contact:** hyrum@quartersmart.com |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: “Legal Case Intake and Contract Analysis Automation” (or your preferred name).
   - Set workflow setting **Execution Order** to `v1` (matches JSON), optional.

2) **Add Webhook node**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: `case-intake`
   - Response: **Last Node**
   - Decide how you will upload the PDF:
     - Prefer **multipart/form-data** with a file field that n8n receives as binary.

3) **Add Set node: “Workflow Configuration”**
   - Add string fields:
     - `litigationAttorney` (Slack destination ID)
     - `corporateAttorney`
     - `familyLawAttorney`
     - `sheetsDocumentId` (Google Sheets file ID)
     - `calendarId` (Google Calendar ID)
   - Enable **Include Other Fields**.

4) **Add Set node: “Normalize Client and Case Info”**
   - Enable **Include Other Fields**.
   - Add fields (expressions):
     - `clientName` = `{{$json.body.clientName || "Unknown Client"}}`
     - `clientEmail` = `{{$json.body.clientEmail}}`
     - `caseType` = `{{$json.body.caseType}}`
     - `caseDescription` = `{{$json.body.caseDescription}}`
     - `intakeDate` = `{{$now.toISO()}}`

5) **Add “Extract From File” node: “Extract Contract Text”**
   - Operation: **PDF**
   - Ensure it is configured to read the correct **binary property** (set it if your PDF arrives under a specific binary key).
   - Test with a real webhook call to confirm it outputs a `text` field.

6) **Add LangChain nodes for AI**
   1. Add **OpenAI Chat Model** node
      - Model: `gpt-4.1-mini`
      - Credentials: configure **OpenAI API** credential (or your n8n-provided credits).
   2. Add **Structured Output Parser**
      - Schema Type: **Manual**
      - Paste the schema described in Section 2 (properties + required: `parties`, `contractType`).
   3. Add **LangChain Agent** node: “Analyze Contract and Extract Clauses”
      - Text input: `Contract text to analyze: {{$json.text}}`
      - Prompt type: “Define”
      - System message: include the extraction instructions (parties, type, obligations, dates, payment, termination, liability, dispute resolution, governing law, risky clauses).
      - Enable output parser in the agent node (or connect parser input as required by your n8n version).
   4. Connect:
      - **OpenAI Chat Model** → Agent (AI Language Model connection)
      - **Structured Output Parser** → Agent (AI Output Parser connection)
      - **Extract Contract Text** → Agent (main)

7) **Add Switch node: “Route by Case Type”**
   - Create rules:
     - If `{{$json.caseType}}` equals `Litigation` → output “Litigation”
     - equals `Corporate` → output “Corporate”
     - equals `Family Law` → output “Family Law”
   - Configure fallback output name: “Other”
   - Connect **Agent → Switch**.

8) **Add Google Sheets node: “Log Case and Contract Info”**
   - Credentials: **Google Sheets OAuth2**
   - Operation: **Append or Update**
   - Document ID: `{{ $('Workflow Configuration').first().json.sheetsDocumentId }}`
   - Sheet name: `Case Log` (create this tab in the spreadsheet)
   - Matching column: `clientEmail`
   - Mapping: Auto-map (ensure your sheet has columns matching the workflow’s field names, including `clientEmail`, `clientName`, `caseType`, `caseDescription`, `intakeDate`, and optionally a column for analysis fields if you want them stored).
   - Connect **Switch → Google Sheets** (choose the outputs you want connected; if you want all branches to log, connect each output including fallback).

9) **Add Slack node: “Notify Responsible Attorney”**
   - Credentials: Slack OAuth/token with permission to post to the destination.
   - Operation: send message to a **channel** (as configured).
   - Channel ID expression:
     - `{{$json.caseType === "Litigation" ? $('Workflow Configuration').first().json.litigationAttorney : $json.caseType === "Corporate" ? $('Workflow Configuration').first().json.corporateAttorney : $('Workflow Configuration').first().json.familyLawAttorney}}`
   - Message type: **Block** and paste the Block Kit JSON (header + fields + summary using `$json.output...`).
   - Connect **Google Sheets → Slack**.
   - Validate whether you’re storing **channel IDs** (recommended) vs **user IDs**.

10) **Add Email node: “Send Summary to Client”**
   - Configure SMTP/email credentials (depends on your n8n setup).
   - From: set a real firm email address.
   - To: `{{$json.clientEmail}}`
   - Subject: `Contract Review Summary - {{$json.clientName}}`
   - HTML: use the provided template referencing `$json.output.*`
   - Connect **Slack → Email**.

11) **Add Convert to File node: “Generate Contract Review PDF”**
   - Operation: HTML → file (PDF)
   - Filename expression:
     - `{{ 'Contract_Review_' + $json.clientName + '_' + $now.toFormat('yyyy-MM-dd') + '.pdf' }}`
   - Connect **Email → Convert to File**.
   - If you want the PDF to contain the email HTML, ensure the Convert node is actually converting the HTML content you intend (you may need to pass HTML explicitly depending on node defaults/version).

12) **Add Google Calendar node: “Schedule Case Follow-ups”**
   - Credentials: Google Calendar OAuth2
   - Calendar ID: `{{ $('Workflow Configuration').first().json.calendarId }}`
   - Start: `{{$now.plus({ days: 7 }).toISO()}}`
   - End: `{{$now.plus({ days: 7, hours: 1 }).toISO()}}`
   - Summary/Description: as specified in JSON.
   - Connect **Convert to File → Google Calendar**.

13) **Add HTTP Request node: “Integrate with Practice Management Software”**
   - Method: POST
   - URL: your practice management API endpoint
   - Authentication: configure a **Generic Credential** supported by your API (Header auth, API key, OAuth2, etc.).
   - Body: JSON
   - Implement the body safely. Recommended pattern (object expression):
     - Set the body to an expression that returns an object, e.g.:
       - `={{ { clientName: $json.clientName, clientEmail: $json.clientEmail, caseType: $json.caseType, caseDescription: $json.caseDescription, intakeDate: $json.intakeDate, contractAnalysis: $json.output } }}`
   - Connect **Google Calendar → HTTP Request**.

14) **(Optional) Add sticky notes**
   - Add canvas notes for: “Webhook & Config”, “AI logic & Routing”, “Log & Notify”, “Scheduling”, and the “Main/Setup/Author” note.

15) **Test end-to-end**
   - Send a POST to the webhook with:
     - JSON fields (client/case) under `body` as expected, and
     - a PDF file attached as binary
   - Confirm:
     - Extracted text is non-empty
     - Agent returns schema-valid JSON
     - Slack message renders and posts to correct destination
     - Google Sheets row is updated
     - Email sends
     - PDF file generated
     - Calendar event created
     - External API receives valid JSON

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automates law firm case and contract management. AI extracts key contract clauses, routes cases to appropriate attorneys, logs details, sends client summaries, and schedules follow-ups. | Sticky note “Main” (workflow intent) |
| Setup checklist: 1) Webhook 2) OpenAI 3) Google Sheets 4) Slack 5) Email 6) PDF generation 7) Calendar 8) Optional practice management integration | Sticky note “Main” (setup) |
| **Author:** Hyrum Hurst, AI Automation Engineer | Sticky note “Main” (credit) |
| **Company:** QuarterSmart | Sticky note “Main” (credit) |
| **Contact:** hyrum@quartersmart.com | Sticky note “Main” (contact) |