Multi-source tax & cash flow forecasting with GPT-4, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/multi-source-tax---cash-flow-forecasting-with-gpt-4--gmail-and-google-sheets-11903


# Multi-source tax & cash flow forecasting with GPT-4, Gmail and Google Sheets

## 1. Workflow Overview

**Purpose:** This workflow runs on a monthly schedule to collect revenue data from up to three external sources, aggregate it into a single dataset, use GPT‑4 to produce structured tax and cash-flow/tax-liability forecasts (3/6/12 months), then distribute results via **Gmail** and store a record in **Google Sheets**.

**Target use cases:** finance teams, accounting firms, tax professionals needing recurring tax liability projections, cash flow planning, and audit-friendly storage without manual consolidation.

### Logical blocks
1. **1.1 Monthly Trigger & Configuration**
   - Starts monthly and sets configurable URLs, rates, email, and sheet ID.
2. **1.2 Multi-source Revenue Retrieval**
   - Calls three HTTP endpoints in parallel to fetch revenue inputs.
3. **1.3 Revenue Consolidation**
   - Aggregates the three source payloads into a single combined structure.
4. **1.4 AI Tax Forecasting (GPT‑4 + Structured Output)**
   - Sends aggregated data to a LangChain Agent using GPT‑4o and enforces a JSON schema output.
5. **1.5 Report Formatting, Emailing, and Storage**
   - Builds a human-readable email body and writes a row (append/update) in Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Monthly Trigger & Configuration

**Overview:** Initiates the workflow monthly and centralizes operational parameters (API URLs, tax rates, email recipient, sheet ID) so downstream nodes reference one source of truth.

**Nodes involved:**
- Monthly Schedule
- Workflow Configuration

#### Node: **Monthly Schedule**
- **Type / role:** `Schedule Trigger` — entry point; executes on a schedule.
- **Configuration (interpreted):** Runs every month at **09:00** (server/project timezone).
- **Inputs / outputs:** No inputs. Outputs to **Workflow Configuration**.
- **Version notes:** typeVersion `1.3` (standard schedule trigger behavior).
- **Failure/edge cases:**
  - Timezone mismatch (n8n instance timezone vs expected business timezone).
  - Workflow is marked **inactive** (`active:false`), so it will not run until activated.

#### Node: **Workflow Configuration**
- **Type / role:** `Set` — defines reusable constants and parameters.
- **Configuration (interpreted):**
  - Sets:
    - `revenueSource1Url`, `revenueSource2Url`, `revenueSource3Url` (placeholders)
    - `taxAgentEmail` (placeholder)
    - `googleSheetId` (placeholder)
    - tax rates: `incomeTaxRate=0.25`, `vatRate=0.2`, `gstRate=0.1`, `withholdingTaxRate=0.15`
  - **Include other fields:** enabled (passes through any incoming JSON too).
- **Key expressions/variables:** None; static assignment.
- **Inputs / outputs:** Input from **Monthly Schedule**; outputs to all three **Fetch Revenue** HTTP nodes.
- **Failure/edge cases:**
  - Placeholder values not replaced → HTTP nodes fail (invalid URL), Gmail/Sheets fail (missing IDs/email).
  - Rates are defined but not explicitly injected into the AI prompt as variables; the agent only receives `$json.data` unless the upstream aggregation includes rates (see Block 2.4 notes).

---

### 2.2 Multi-source Revenue Retrieval

**Overview:** Fetches revenue data from three independent sources via HTTP requests, in parallel.

**Nodes involved:**
- Fetch Revenue Data - Source 1
- Fetch Revenue Data - Source 2
- Fetch Revenue Data - Source 3

#### Node: **Fetch Revenue Data - Source 1**
- **Type / role:** `HTTP Request` — retrieves revenue payload from source 1.
- **Configuration (interpreted):**
  - URL: `{{$('Workflow Configuration').first().json.revenueSource1Url}}`
  - Sends header: `Content-Type: application/json`
  - Uses default HTTP method (GET unless changed; not specified otherwise).
- **Inputs / outputs:** From **Workflow Configuration**; outputs to **Aggregate Revenue Data**.
- **Version notes:** typeVersion `4.3`.
- **Failure/edge cases:**
  - Invalid/empty URL placeholder.
  - Auth not configured (node shows no auth settings); if the endpoint needs auth, it will 401/403.
  - Non-JSON response may break assumptions downstream (AI expects `$json.data` later).
  - Timeouts / rate limits / transient failures; no retry logic configured.

#### Node: **Fetch Revenue Data - Source 2**
- Same structure as Source 1, URL uses `revenueSource2Url`.
- **Failure/edge cases:** same as above.

#### Node: **Fetch Revenue Data - Source 3**
- Same structure as Source 1, URL uses `revenueSource3Url`.
- **Failure/edge cases:** same as above.

---

### 2.3 Revenue Consolidation

**Overview:** Combines outputs of the three HTTP calls into a single aggregated item to feed the AI agent.

**Nodes involved:**
- Aggregate Revenue Data

#### Node: **Aggregate Revenue Data**
- **Type / role:** `Aggregate` — aggregates multiple incoming items into one.
- **Configuration (interpreted):**
  - Mode: `aggregateAllItemData` (collect all item data into a unified structure).
- **Inputs / outputs:**
  - Inputs: three separate connections from the three **Fetch Revenue Data** nodes.
  - Output: single aggregated stream to **Tax Forecasting AI Agent**.
- **Version notes:** typeVersion `1`.
- **Failure/edge cases:**
  - Shape of aggregated data depends on each HTTP node output structure. If sources return different JSON shapes, the aggregate result may be inconsistent.
  - If one source fails and stops execution, the aggregate node may never run (no error handling/continue-on-fail configured).

---

### 2.4 AI Tax Forecasting (GPT‑4 + Structured Output)

**Overview:** Uses a LangChain Agent backed by GPT‑4o to analyze aggregated revenue, compute forecasted tax liabilities, and produce a structured JSON result enforced by a schema parser.

**Nodes involved:**
- Tax Forecasting AI Agent
- OpenAI GPT-4
- Structured Forecast Output

#### Node: **Tax Forecasting AI Agent**
- **Type / role:** `LangChain Agent` — orchestrates prompt + tools (LM + output parser).
- **Configuration (interpreted):**
  - **Prompt type:** “define”
  - **Input text:** `={{ $json.data }}`
    - This assumes the incoming aggregated item has a `data` property. That may not be true depending on the HTTP response and aggregate node output shape.
  - **System message:** instructs the model to:
    1) analyze aggregated revenue,
    2) calculate liabilities for Income Tax/VAT/GST/Withholding using rates from config,
    3) produce 3/6/12 month rolling forecasts,
    4) identify trends and opportunities,
    5) return JSON matching the output parser schema.
  - **Output parser enabled:** yes.
- **Inputs / outputs:**
  - Main input: from **Aggregate Revenue Data**
  - AI connections:
    - Language model: **OpenAI GPT-4**
    - Output parser: **Structured Forecast Output**
  - Main output: to **Format Report Data**
- **Version notes:** typeVersion `3.1` (LangChain integration).
- **Failure/edge cases:**
  - **Critical data mismatch risk:** `$json.data` may be `undefined`. Many HTTP APIs return data at the root (e.g., `$json` itself), not nested under `data`.
  - **Rates not actually provided:** the system message says “using rate from workflow config,” but the agent is not explicitly given those rates unless they are present in `$json.data` (they currently live in **Workflow Configuration** item, not necessarily merged into aggregated data).
  - Output parser failures if the model returns non-conforming JSON (common without strong constraints).
  - Token limits if revenue payloads are large; consider summarizing or truncating.

#### Node: **OpenAI GPT-4**
- **Type / role:** `LM Chat OpenAI` — the LLM used by the agent.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Credentials: `openAiApi`
- **Inputs / outputs:** Connected to the agent via `ai_languageModel`.
- **Version notes:** typeVersion `1.3`.
- **Failure/edge cases:**
  - Invalid/expired OpenAI credential.
  - Model access not enabled for the API key / policy restrictions.
  - Rate limits/timeouts.

#### Node: **Structured Forecast Output**
- **Type / role:** `Structured Output Parser` — enforces a JSON schema.
- **Configuration (interpreted):**
  - Manual JSON schema requiring an object with:
    - `forecastPeriod` (string)
    - numeric totals: `totalRevenue`, each tax liability, `totalTaxLiability`
    - `forecast3Months`, `forecast6Months`, `forecast12Months` as objects
    - `insights` and `trends` arrays of strings
- **Inputs / outputs:** Connected to the agent via `ai_outputParser`.
- **Version notes:** typeVersion `1.3`.
- **Failure/edge cases:**
  - Parser rejects output if missing fields / wrong types.
  - Forecast objects have no defined sub-schema; inconsistent internal structure may still pass (object is allowed) but be hard to use later.

---

### 2.5 Report Formatting, Emailing, and Storage

**Overview:** Converts the structured forecast JSON into an email-friendly text report and persists key fields in Google Sheets for tracking/audit.

**Nodes involved:**
- Format Report Data
- Send Report to Tax Agent
- Store in Google Sheets

#### Node: **Format Report Data**
- **Type / role:** `Set` — prepares report metadata and email content.
- **Configuration (interpreted):**
  - Sets:
    - `reportDate = {{$now.format('yyyy-MM-dd')}}`
    - `reportTitle = "Monthly Tax Forecast Report"`
    - `emailSubject = "Tax Forecast Report - " + current month/year`
    - `emailBody` plain text including totals from `$json.totalRevenue`, `$json.totalTaxLiability`, etc.
  - Include other fields: enabled (keeps AI output fields).
- **Inputs / outputs:** Input from **Tax Forecasting AI Agent**; outputs to **Send Report to Tax Agent** and **Store in Google Sheets**.
- **Failure/edge cases:**
  - If AI output is missing numeric fields (parser failure or partial output), email body expressions may render `undefined`.
  - Formatting assumes USD with `$` prefix; may be incorrect for other currencies/locales.

#### Node: **Send Report to Tax Agent**
- **Type / role:** `Gmail` — sends the report via email.
- **Configuration (interpreted):**
  - To: `{{$('Workflow Configuration').first().json.taxAgentEmail}}`
  - Subject: `{{$json.emailSubject}}`
  - Body: `{{$json.emailBody}}`
  - Email type: text
  - Credentials: `gmailOAuth2`
- **Inputs / outputs:** Input from **Format Report Data**; no downstream nodes.
- **Version notes:** typeVersion `2.2`.
- **Failure/edge cases:**
  - Gmail OAuth expired / missing scopes.
  - Recipient email placeholder not set.
  - Gmail API quotas.
  - No attachment generation is implemented despite email text saying “attached” (“Please find attached…”); this is a content/logic mismatch.

#### Node: **Store in Google Sheets**
- **Type / role:** `Google Sheets` — persists forecast metrics.
- **Configuration (interpreted):**
  - Operation: **appendOrUpdate**
  - Document ID: `{{$('Workflow Configuration').first().json.googleSheetId}}`
  - Sheet name: **Tax Forecasts**
  - Mapping: auto-map with schema fields:
    - `reportDate`, `forecastPeriod`, `totalRevenue`, `incomeTaxLiability`, `vatLiability`, `gstLiability`, `withholdingTaxLiability`, `totalTaxLiability`
  - Matching column(s): `reportDate` (used as key for update vs append)
- **Inputs / outputs:** Input from **Format Report Data**; no downstream nodes.
- **Version notes:** typeVersion `4.7`.
- **Failure/edge cases:**
  - Sheet/tab “Tax Forecasts” does not exist.
  - `reportDate` as unique key: running multiple times on the same day overwrites prior row (may be desired or not).
  - OAuth expired / insufficient permissions.
  - Data type issues if AI outputs strings instead of numbers.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Schedule | scheduleTrigger | Monthly workflow entry point | — | Workflow Configuration | ## Monthly Schedule Trigger: Initiates workflow on consistent monthly cycle to ensure regular forecasting aligned with financial planning periods. |
| Workflow Configuration | set | Centralized config (URLs, rates, email, sheet ID) | Monthly Schedule | Fetch Revenue Data - Source 1; Fetch Revenue Data - Source 2; Fetch Revenue Data - Source 3 | ## Multi-Source Data Aggregation: Fetches revenue data from three distinct sources and consolidates into unified dataset, capturing complete revenue picture across all operational channels. |
| Fetch Revenue Data - Source 1 | httpRequest | Pull revenue from source 1 | Workflow Configuration | Aggregate Revenue Data | ## Multi-Source Data Aggregation: Fetches revenue data from three distinct sources and consolidates into unified dataset, capturing complete revenue picture across all operational channels. |
| Fetch Revenue Data - Source 2 | httpRequest | Pull revenue from source 2 | Workflow Configuration | Aggregate Revenue Data | ## Multi-Source Data Aggregation: Fetches revenue data from three distinct sources and consolidates into unified dataset, capturing complete revenue picture across all operational channels. |
| Fetch Revenue Data - Source 3 | httpRequest | Pull revenue from source 3 | Workflow Configuration | Aggregate Revenue Data | ## Multi-Source Data Aggregation: Fetches revenue data from three distinct sources and consolidates into unified dataset, capturing complete revenue picture across all operational channels. |
| Aggregate Revenue Data | aggregate | Combine multi-source responses into one dataset | Fetch Revenue Data - Source 1; Source 2; Source 3 | Tax Forecasting AI Agent | ## Multi-Source Data Aggregation: Fetches revenue data from three distinct sources and consolidates into unified dataset, capturing complete revenue picture across all operational channels. |
| Tax Forecasting AI Agent | @n8n/n8n-nodes-langchain.agent | LLM-driven tax forecasting with enforced JSON output | Aggregate Revenue Data | Format Report Data | ## Tax Forecasting AI Agent: GPT-4 model predicts tax obligations with intelligent, context-aware tax projections based on patterns and historical data. |
| OpenAI GPT-4 | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o language model for the agent | — (AI connection) | Tax Forecasting AI Agent (AI model input) | ## Tax Forecasting AI Agent: GPT-4 model predicts tax obligations with intelligent, context-aware tax projections based on patterns and historical data. |
| Structured Forecast Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce schema for forecast JSON | — (AI connection) | Tax Forecasting AI Agent (AI parser input) | ## Tax Forecasting AI Agent: GPT-4 model predicts tax obligations with intelligent, context-aware tax projections based on patterns and historical data. |
| Format Report Data | set | Build report metadata + email content | Tax Forecasting AI Agent | Send Report to Tax Agent; Store in Google Sheets | ## Report Generation & Distribution: Formats presentation-ready forecast documentation and sends via Gmail to tax agents, maintaining communication workflow with tax professionals. |
| Send Report to Tax Agent | gmail | Email report to tax agent | Format Report Data | — | ## Report Generation & Distribution: Formats presentation-ready forecast documentation and sends via Gmail to tax agents, maintaining communication workflow with tax professionals. |
| Store in Google Sheets | googleSheets | Persist forecast snapshot to sheet | Format Report Data | — | ## Report Generation & Distribution: Formats presentation-ready forecast documentation and sends via Gmail to tax agents, maintaining communication workflow with tax professionals. |
| Sticky Note | stickyNote | Documentation annotation | — | — | ## How It Works: Automates monthly revenue aggregation from multiple sources with intelligent tax forecasting using GPT-4 structured analysis… stores results in Google Sheets… |
| Sticky Note1 | stickyNote | Documentation annotation | — | — | ## Setup Steps: 1. Configure OpenAI API key… 6. Configure Google Sheets destination |
| Sticky Note2 | stickyNote | Documentation annotation | — | — | ## Prerequisites: OpenAI API key with GPT-4 access… ## Use Cases: Monthly tax liability projections… |
| Sticky Note3 | stickyNote | Documentation annotation | — | — | ## Customization: Adjust forecast model parameters… ## Benefits: Eliminates manual tax calculations… |
| Sticky Note4 | stickyNote | Documentation annotation | — | — | ## Multi-Source Data Aggregation: Fetches revenue data from three distinct sources and consolidates into unified dataset… |
| Sticky Note5 | stickyNote | Documentation annotation | — | — | ## Tax Forecasting AI Agent: GPT-4 model predicts tax obligations… |
| Sticky Note6 | stickyNote | Documentation annotation | — | — | ## Monthly Schedule Trigger: Initiates workflow on consistent monthly cycle… |
| Sticky Note7 | stickyNote | Documentation annotation | — | — | ## Report Generation & Distribution: Formats… sends via Gmail… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Automated Cash Flow and Tax Forecasting Engine**
   - (Optional) Add a description matching your business context.

2. **Add trigger: “Monthly Schedule”**
   - Node type: **Schedule Trigger**
   - Configure: **Every 1 month**, at **09:00** (adjust timezone in n8n settings as needed).

3. **Add configuration node: “Workflow Configuration”**
   - Node type: **Set**
   - Add fields:
     - `revenueSource1Url` (string) – your API endpoint
     - `revenueSource2Url` (string)
     - `revenueSource3Url` (string)
     - `taxAgentEmail` (string)
     - `googleSheetId` (string)
     - `incomeTaxRate` (number, e.g. 0.25)
     - `vatRate` (number, e.g. 0.2)
     - `gstRate` (number, e.g. 0.1)
     - `withholdingTaxRate` (number, e.g. 0.15)
   - Enable **Include Other Fields**.
   - Connect: **Monthly Schedule → Workflow Configuration**.

4. **Add three HTTP Request nodes**
   - Names:
     - **Fetch Revenue Data - Source 1**
     - **Fetch Revenue Data - Source 2**
     - **Fetch Revenue Data - Source 3**
   - Node type: **HTTP Request**
   - For each node:
     - URL expression:
       - Source 1: `{{$('Workflow Configuration').first().json.revenueSource1Url}}`
       - Source 2: `{{$('Workflow Configuration').first().json.revenueSource2Url}}`
       - Source 3: `{{$('Workflow Configuration').first().json.revenueSource3Url}}`
     - Add header: `Content-Type: application/json`
     - Configure authentication if your sources require it (API key/OAuth2/etc.).
   - Connect: **Workflow Configuration → each Fetch Revenue…** (three parallel connections).

5. **Add “Aggregate Revenue Data”**
   - Node type: **Aggregate**
   - Operation/mode: **Aggregate All Item Data** (aggregate all incoming item JSON)
   - Connect each **Fetch Revenue… → Aggregate Revenue Data**.

6. **Add LLM components**
   - **OpenAI model node**
     - Node type: **OpenAI Chat Model** (`lmChatOpenAi`)
     - Model: **gpt-4o**
     - Credentials: configure **OpenAI API** credential in n8n and select it here.
   - **Structured Output Parser**
     - Node type: **Structured Output Parser**
     - Schema: paste/define the object schema with fields:
       - `forecastPeriod`, `totalRevenue`, `incomeTaxLiability`, `vatLiability`, `gstLiability`, `withholdingTaxLiability`, `totalTaxLiability`, `forecast3Months`, `forecast6Months`, `forecast12Months`, `insights[]`, `trends[]`.

7. **Add “Tax Forecasting AI Agent”**
   - Node type: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Prompt type: **Define**
   - Text input: `{{$json.data}}` (as in the workflow)
   - System message: include instructions to compute liabilities/forecasts and return JSON conforming to the parser.
   - Connect AI ports:
     - **OpenAI model → Agent** via *AI Language Model* connection
     - **Structured Output Parser → Agent** via *AI Output Parser* connection
   - Connect main flow: **Aggregate Revenue Data → Tax Forecasting AI Agent**.

8. **Add “Format Report Data”**
   - Node type: **Set**
   - Enable **Include Other Fields**
   - Add fields (expressions):
     - `reportDate = {{$now.format('yyyy-MM-dd')}}`
     - `reportTitle = "Monthly Tax Forecast Report"`
     - `emailSubject = {{ 'Tax Forecast Report - ' + $now.format('MMMM yyyy') }}`
     - `emailBody` = build a text block referencing:
       - `$json.totalRevenue`, `$json.totalTaxLiability`, `$json.incomeTaxLiability`, `$json.vatLiability`, `$json.gstLiability`, `$json.withholdingTaxLiability`
   - Connect: **Tax Forecasting AI Agent → Format Report Data**.

9. **Add “Send Report to Tax Agent” (Gmail)**
   - Node type: **Gmail**
   - Operation: **Send**
   - To: `{{$('Workflow Configuration').first().json.taxAgentEmail}}`
   - Subject: `{{$json.emailSubject}}`
   - Message: `{{$json.emailBody}}`
   - Email type: **text**
   - Credentials: configure/select **Gmail OAuth2** credential.
   - Connect: **Format Report Data → Send Report to Tax Agent**.

10. **Add “Store in Google Sheets”**
   - Node type: **Google Sheets**
   - Operation: **Append or Update**
   - Document ID: `{{$('Workflow Configuration').first().json.googleSheetId}}`
   - Sheet name: **Tax Forecasts** (create this tab in the sheet)
   - Matching column: `reportDate`
   - Map columns:
     - `reportDate`, `forecastPeriod`, `totalRevenue`, `incomeTaxLiability`, `vatLiability`, `gstLiability`, `withholdingTaxLiability`, `totalTaxLiability`
   - Credentials: configure/select **Google Sheets OAuth2** credential.
   - Connect: **Format Report Data → Store in Google Sheets**.

11. **Activate the workflow**
   - Ensure all placeholders are replaced and credentials are valid.
   - Turn the workflow to **Active** so the monthly trigger runs.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **How It Works**: Automates monthly revenue aggregation from multiple sources with intelligent tax forecasting using GPT‑4 structured analysis… sends projections via Gmail and stores results in Google Sheets for audit trails. | Sticky note (workflow overview) |
| **Setup Steps**: Configure OpenAI API key; connect three revenue sources; map aggregation; define structured output; set up Gmail; configure Google Sheets destination. | Sticky note (setup) |
| **Prerequisites**: OpenAI API key with GPT‑4 access, Gmail account, Google Sheets, three revenue data source credentials. **Use Cases**: Monthly tax liability projections, quarterly estimated tax planning. | Sticky note (prereqs/use cases) |
| **Customization**: Adjust forecast model parameters, add revenue sources, modify email templates. **Benefits**: Eliminates manual tax calculations, enables proactive planning, improves cash flow forecasting accuracy. | Sticky note (customization/benefits) |

**Notable implementation gaps to consider when reproducing/modifying:**
- The agent input is `{{$json.data}}`; ensure your aggregated structure actually has a `data` field (or change to `{{$json}}` / a composed object).
- The prompt claims it uses rates from config, but rates are not explicitly passed into the agent unless you merge them into the aggregated payload.
- Email body mentions an attachment, but no file/report attachment is generated in the workflow.