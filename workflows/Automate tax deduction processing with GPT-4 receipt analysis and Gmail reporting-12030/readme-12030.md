Automate tax deduction processing with GPT-4 receipt analysis and Gmail reporting

https://n8nworkflows.xyz/workflows/automate-tax-deduction-processing-with-gpt-4-receipt-analysis-and-gmail-reporting-12030


# Automate tax deduction processing with GPT-4 receipt analysis and Gmail reporting

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name (JSON):** *Intelligent Tax Deduction Processing and Filing*  
**User-provided title:** *Automate tax deduction processing with GPT-4 receipt analysis and Gmail reporting*

**Purpose:**  
Runs monthly, fetches expense receipt files and revenue data from external APIs, extracts text from receipts, uses an OpenAI chat model with structured output parsing to classify deductible expenses, merges deductions with revenue, matches expenses to revenue periods, calculates deduction totals and net amounts, formats a report, and emails it to a tax agent via Gmail.

**Target use cases:**  
- Accountants/tax professionals automating monthly receipt review and categorization  
- Finance teams producing consistent, audit-friendly deduction summaries  
- Multi-client processing (with API endpoints pointing to per-client datasets)

### 1.1 Scheduling & Runtime Configuration
Sets the monthly trigger and defines key variables (API endpoints, tax agent email, tax year).

### 1.2 Fetch & Validate Expense Receipts
Downloads receipt files and verifies that binary file data exists before extraction.

### 1.3 Receipt Text Extraction + AI Deduction Classification
Extracts text from receipt PDF(s), then uses GPT (OpenAI Chat Model) + a Structured Output Parser to return normalized JSON fields like vendor/date/amount/category/deductibility.

### 1.4 Fetch Revenue + Merge Deductions
Fetches revenue data, then merges revenue and AI-derived deductions into a single combined stream.

### 1.5 Period Matching + Deduction Computation
Attempts to match each expense to a revenue period, then computes totals and net values.

### 1.6 Report Formatting + Gmail Delivery
Shapes final report fields and emails the summary to the configured tax agent.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Runtime Configuration

**Overview:**  
Triggers the workflow monthly and sets configuration variables used by downstream nodes (endpoints, email, tax year).

**Nodes involved:**  
- Monthly Tax Processing Schedule  
- Workflow Configuration

#### Node: Monthly Tax Processing Schedule
- **Type / role:** `Schedule Trigger` — entry point; emits an execution on a schedule.
- **Configuration (interpreted):**
  - Runs **every month** at **09:00** (server / n8n instance timezone).
- **Inputs:** None (trigger).
- **Outputs:** To **Workflow Configuration**.
- **Version considerations:** typeVersion `1.3`.
- **Failure modes / edge cases:**
  - Timezone misunderstandings (instance timezone vs expected business timezone).
  - Missed runs if instance is down; depending on n8n settings, triggers do not “catch up”.

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes constants/variables for reuse.
- **Configuration (interpreted):**
  - Adds/overwrites these fields:
    - `expenseReceiptsApiUrl` (placeholder string)
    - `revenueDataApiUrl` (placeholder string)
    - `taxAgentEmail` (placeholder string)
    - `currentTaxYear` (number) = `new Date().getFullYear()`
  - `includeOtherFields: true` (keeps any incoming fields).
- **Key expressions / variables:**
  - `={{ new Date().getFullYear() }}`
- **Inputs:** From Schedule Trigger.
- **Outputs:** To **Fetch Expense Receipts** and **Fetch Revenue Data** (fan-out).
- **Failure modes / edge cases:**
  - Placeholder values not replaced → HTTP nodes will fail with invalid URL.
  - `taxAgentEmail` invalid → Gmail node may fail or bounce.
- **Version considerations:** typeVersion `3.4`.

---

### Block 2 — Fetch & Validate Expense Receipts

**Overview:**  
Downloads receipt content as a file and ensures binary data is present before attempting PDF extraction.

**Nodes involved:**  
- Fetch Expense Receipts  
- Check Receipt File Type

#### Node: Fetch Expense Receipts
- **Type / role:** `HTTP Request` — retrieves receipt file(s).
- **Configuration (interpreted):**
  - `URL` is taken from config: `$('Workflow Configuration').first().json.expenseReceiptsApiUrl`
  - Response format is set to **File** (binary).
- **Key expressions / variables:**
  - `={{ $('Workflow Configuration').first().json.expenseReceiptsApiUrl }}`
- **Inputs:** From Workflow Configuration.
- **Outputs:** To **Check Receipt File Type**.
- **Failure modes / edge cases:**
  - Auth not configured for the endpoint (no headers/credentials in node) → 401/403.
  - Endpoint returns JSON instead of file → binary may be missing; downstream IF fails.
  - Multiple receipts: this node as configured does not explicitly paginate or iterate; behavior depends on API response (single file vs collection).
- **Version considerations:** typeVersion `4.3`.

#### Node: Check Receipt File Type
- **Type / role:** `IF` — validates that binary exists.
- **Configuration (interpreted):**
  - Condition: checks whether `{{ $binary }}` **exists**.
  - Only the **true** branch is wired; false branch is not handled.
- **Inputs:** From Fetch Expense Receipts.
- **Outputs:** True path → **Extract Receipt Data**.
- **Failure modes / edge cases:**
  - If binary is absent, workflow effectively stops for that branch (no false-path handling).
  - “File type” is not actually validated (no MIME/extension check); only binary existence is tested.
- **Version considerations:** typeVersion `2.3`.

---

### Block 3 — Receipt Text Extraction + AI Deduction Classification

**Overview:**  
Extracts text from a PDF receipt and sends it to a LangChain Agent powered by OpenAI, enforcing a strict structured JSON output via a schema.

**Nodes involved:**  
- Extract Receipt Data  
- Extract Deductible Categories  
- OpenAI Chat Model  
- Structured Output Parser

#### Node: Extract Receipt Data
- **Type / role:** `Extract From File` — converts PDF binary to text.
- **Configuration (interpreted):**
  - Operation: **PDF** extraction.
- **Inputs:** From Check Receipt File Type (true branch).
- **Outputs:** To **Extract Deductible Categories** (text in `$json.text`).
- **Failure modes / edge cases:**
  - Non-PDF input despite “binary exists” check → extraction error.
  - Scanned/image-only PDFs may extract poorly without OCR support (depends on node capabilities/version).
- **Version considerations:** typeVersion `1.1`.

#### Node: OpenAI Chat Model
- **Type / role:** `LangChain Chat Model (OpenAI)` — provides the underlying model for the agent.
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
  - Credentials: `openAiApi` (must be configured in n8n).
- **Inputs / outputs (AI connections):**
  - Connected via `ai_languageModel` output to **Extract Deductible Categories**.
- **Failure modes / edge cases:**
  - Invalid/expired OpenAI API key → auth errors.
  - Rate limits / quota exceeded.
  - Model name not available in the account/region.
- **Version considerations:** typeVersion `1.3`.

#### Node: Structured Output Parser
- **Type / role:** `LangChain Structured Output Parser` — enforces schema-based JSON output.
- **Configuration (interpreted):**
  - Manual JSON schema with fields:
    - `vendor` (string)
    - `expenseDate` (string)
    - `amount` (number)
    - `category` (string)
    - `deductibilityStatus` (string)
    - `deductionPercentage` (number)
    - `notes` (string)
- **Inputs / outputs (AI connections):**
  - Connected via `ai_outputParser` to **Extract Deductible Categories**.
- **Failure modes / edge cases:**
  - If the model returns non-conforming JSON, parsing fails.
  - `expenseDate` is only typed as string; inconsistent formats may break later date logic.
- **Version considerations:** typeVersion `1.3`.

#### Node: Extract Deductible Categories
- **Type / role:** `LangChain Agent` — prompts model to analyze receipt text and produce deductible classification.
- **Configuration (interpreted):**
  - Prompt text: `Receipt data: {{ $json.text }}`
  - System message instructs extraction of vendor/date/amount/category/deductibility + percent + justification.
  - Output parser enabled (`hasOutputParser: true`) and wired to Structured Output Parser node.
  - Language model wired to OpenAI Chat Model node.
- **Key expressions / variables:**
  - `=Receipt data: {{ $json.text }}`
- **Inputs:** Extracted text from **Extract Receipt Data**.
- **Outputs:** To **Merge Revenue and Deductions** (structured JSON object per receipt).
- **Failure modes / edge cases:**
  - Hallucinated values if receipt text is incomplete/garbled.
  - Currency ambiguity (amount number without currency).
  - Partial deductibility rules vary by jurisdiction; system prompt is generic.
- **Version considerations:** typeVersion `3.1`.

---

### Block 4 — Fetch Revenue + Merge Deductions

**Overview:**  
Retrieves revenue data and combines it with the AI-extracted deduction stream.

**Nodes involved:**  
- Fetch Revenue Data  
- Merge Revenue and Deductions

#### Node: Fetch Revenue Data
- **Type / role:** `HTTP Request` — retrieves revenue data as JSON.
- **Configuration (interpreted):**
  - `URL` from config: `$('Workflow Configuration').first().json.revenueDataApiUrl`
  - Response format: **JSON**
- **Inputs:** From Workflow Configuration.
- **Outputs:** To **Merge Revenue and Deductions** (input index 1).
- **Failure modes / edge cases:**
  - Missing auth/headers if endpoint requires it → 401/403.
  - Schema mismatch: downstream code expects `revenue` array (see later).
- **Version considerations:** typeVersion `4.3`.

#### Node: Merge Revenue and Deductions
- **Type / role:** `Merge` — combines two inbound streams into one item set.
- **Configuration (interpreted):**
  - Mode: **Combine**
  - Combine By: **combineAll** (creates combined items from both inputs)
- **Inputs:**
  - Input 0: From **Extract Deductible Categories**
  - Input 1: From **Fetch Revenue Data**
- **Outputs:** To **Match Expenses to Revenue Periods**
- **Failure modes / edge cases:**
  - If one branch doesn’t execute (e.g., receipts missing), merge behavior may yield unexpected empty results.
  - `combineAll` can create data duplication depending on item counts (Cartesian-like behavior) if both sides have multiple items.
- **Version considerations:** typeVersion `3.2`.

---

### Block 5 — Period Matching + Deduction Computation

**Overview:**  
Attempts to align expenses to revenue periods and compute deduction totals. In the provided workflow, there are **field-name mismatches** that will likely prevent correct calculations without modification.

**Nodes involved:**  
- Match Expenses to Revenue Periods  
- Calculate Tax Deductions

#### Node: Match Expenses to Revenue Periods
- **Type / role:** `Code` — matches expense dates to revenue periods.
- **Configuration (interpreted):**
  - Reads:
    - `const expenses = $input.first().json.expenses || [];`
    - `const revenue = $input.last().json.revenue || [];`
  - For each `expense`, it expects `expense.date`.
  - For each revenue record, expects `rev.periodStart` and `rev.periodEnd`.
  - Outputs items like:
    ```js
    { expense: expense, revenue: matchingRevenue || null, matched: !!matchingRevenue }
    ```
- **Inputs:** From Merge Revenue and Deductions.
- **Outputs:** To Calculate Tax Deductions.
- **Major integration issue (schema mismatch):**
  - Upstream AI output schema produces fields like `expenseDate`, `amount`, `deductionPercentage`, etc., **not** an `expenses` array.
  - Revenue fetch node likely returns an array directly or under another key, not necessarily `{ revenue: [...] }`.
  - Therefore, `expenses` and/or `revenue` will often be `[]`, producing no matches.
- **Failure modes / edge cases:**
  - Date parsing errors if `expense.date` is not ISO-compatible.
  - Revenue periods overlapping (find returns first match only).
  - Unmatched revenue yields `revenue: null`.
- **Version considerations:** typeVersion `2`.

#### Node: Calculate Tax Deductions
- **Type / role:** `Code` — aggregates deductions by period and computes net amounts.
- **Configuration (interpreted):**
  - Expects each input item to have:
    - `revenuePeriod` (string)
    - `revenue` (number)
    - `expenseAmount` (number)
    - `deductionPercentage` (number)
    - `category` (string)
  - Computes:
    - `deductionAmount = expenseAmount * (deductionPercentage/100)`
  - Groups into `periodMap`, then outputs per period:
    - `period`, `revenue`, `totalDeductions`, `netAmount`, `expenses[]`, `expenseCount`
- **Inputs:** From Match Expenses to Revenue Periods.
- **Outputs:** To Format Tax Report.
- **Major integration issue (schema mismatch):**
  - The previous node outputs `{ expense: {...}, revenue: {...}, matched: true/false }`, not `revenuePeriod/expenseAmount`.
  - Without mapping/transform, this node will treat values as missing and default to 0, producing incorrect totals.
- **Failure modes / edge cases:**
  - If `revenue` is an object (from matching) rather than a number, `revenue - totalDeductions` may yield `NaN`.
- **Version considerations:** typeVersion `2`.

---

### Block 6 — Report Formatting + Gmail Delivery

**Overview:**  
Prepares report fields and emails a summary to the tax agent. As provided, the formatter expects fields that the current calculation node does not output (another schema mismatch).

**Nodes involved:**  
- Format Tax Report  
- Send to Tax Agent

#### Node: Format Tax Report
- **Type / role:** `Set` — formats the final report object.
- **Configuration (interpreted):**
  - Sets:
    - `reportTitle` = `Tax Deduction Report - {{ currentTaxYear }}`
    - `generatedDate` = ISO timestamp
    - `totalRevenue` = `{{ $json.totalRevenue }}`
    - `totalDeductions` = `{{ $json.totalDeductions }}`
    - `netIncome` = `{{ $json.netIncome }}`
    - `deductionsByCategory` = `{{ $json.deductionsByCategory }}`
    - `expenseDetails` = `{{ $json.expenseDetails }}`
- **Inputs:** From Calculate Tax Deductions.
- **Outputs:** To Send to Tax Agent.
- **Major integration issue (schema mismatch):**
  - Calculate Tax Deductions outputs `revenue`, `totalDeductions`, `netAmount`, etc.—not `totalRevenue`, `netIncome`, etc.
  - This node will likely produce `undefined` totals unless the calculation node is adjusted or an intermediate aggregation step is added.
- **Version considerations:** typeVersion `3.4`.

#### Node: Send to Tax Agent
- **Type / role:** `Gmail` — sends email to configured recipient.
- **Configuration (interpreted):**
  - To: `taxAgentEmail` from Workflow Configuration
  - Subject: `Tax Deduction Report - {{ currentTaxYear }}`
  - Body: text email containing `totalRevenue`, `totalDeductions`, `netIncome`
  - Note: No attachment is configured despite message text saying “attached”.
- **Credentials:** `gmailOAuth2` must be configured.
- **Inputs:** From Format Tax Report.
- **Outputs:** End of workflow.
- **Failure modes / edge cases:**
  - OAuth token expired/revoked.
  - Sending limits (Gmail API quotas).
  - Body references undefined fields (will render blank/“undefined”).
- **Version considerations:** typeVersion `2.2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Tax Processing Schedule | Schedule Trigger | Monthly entry point | — | Workflow Configuration | ## How It Works  \nThis workflow automates monthly tax processing by ingesting expense receipts alongside revenue data, extracting structured deduction details using GPT-4, and accurately matching expenses to their corresponding revenue periods. It retrieves receipts with built-in type validation, parses deduction information through OpenAI structured output extraction, and consolidates revenue records into a unified dataset. The system then intelligently aligns expenses with revenue timelines, calculates eligible deductions, and generates well-formatted tax reports that are automatically sent to designated agents via Gmail. Designed for accountants, tax professionals, and finance teams, it enables automated expense categorization and optimized deduction calculations. |
| Workflow Configuration | Set | Defines endpoints, recipient, tax year | Monthly Tax Processing Schedule | Fetch Expense Receipts; Fetch Revenue Data | ## Setup Steps  \n1. Configure receipt storage source and OpenAI Chat Model API key.\n2. Connect Gmail for report delivery and set up tax agent email.\n3. Define expense categories, revenue periods, and deduction rules.\n4. Schedule monthly trigger and test extraction |
| Fetch Expense Receipts | HTTP Request | Download receipt file(s) | Workflow Configuration | Check Receipt File Type | ## Fetch & Validate Expenses  \nWhat: Retrieves expense receipts and validates file types before processing.\nWhy: Ensures data quality and prevents processing errors from invalid formats. |
| Check Receipt File Type | IF | Ensures binary data exists | Fetch Expense Receipts | Extract Receipt Data | ## Fetch & Validate Expenses  \nWhat: Retrieves expense receipts and validates file types before processing.\nWhy: Ensures data quality and prevents processing errors from invalid formats. |
| Extract Receipt Data | Extract From File | Extracts text from PDF receipt | Check Receipt File Type | Extract Deductible Categories | ## Fetch & Validate Expenses  \nWhat: Retrieves expense receipts and validates file types before processing.\nWhy: Ensures data quality and prevents processing errors from invalid formats. |
| Extract Deductible Categories | LangChain Agent | Classifies deductible info into structured JSON | Extract Receipt Data; OpenAI Chat Model (AI); Structured Output Parser (AI) | Merge Revenue and Deductions | ## Prerequisites  \nExpense receipt repository; OpenAI API key; Gmail account; revenue data source.\n\n## Use Cases\nAccountants automating receipt processing for multiple clients; \n\n## Customization\nAdjust extraction prompts for industry-specific expenses, modify deduction rules \n\n## Benefits\nEliminates manual receipt review, reduces categorization errors |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | Provides GPT model to agent | — | Extract Deductible Categories (AI connection) | ## Prerequisites  \nExpense receipt repository; OpenAI API key; Gmail account; revenue data source.\n\n## Use Cases\nAccountants automating receipt processing for multiple clients; \n\n## Customization\nAdjust extraction prompts for industry-specific expenses, modify deduction rules \n\n## Benefits\nEliminates manual receipt review, reduces categorization errors |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforces JSON schema output | — | Extract Deductible Categories (AI connection) | ## Prerequisites  \nExpense receipt repository; OpenAI API key; Gmail account; revenue data source.\n\n## Use Cases\nAccountants automating receipt processing for multiple clients; \n\n## Customization\nAdjust extraction prompts for industry-specific expenses, modify deduction rules \n\n## Benefits\nEliminates manual receipt review, reduces categorization errors |
| Fetch Revenue Data | HTTP Request | Fetch revenue data (JSON) | Workflow Configuration | Merge Revenue and Deductions | ## Merge Revenue & Expenses\nWhat: Consolidates expense and revenue datasets into aligned records by period.\nWhy: Enables accurate deduction-to-revenue matching for tax calculations. |
| Merge Revenue and Deductions | Merge | Combines revenue and deductions streams | Extract Deductible Categories; Fetch Revenue Data | Match Expenses to Revenue Periods | ## Merge Revenue & Expenses\nWhat: Consolidates expense and revenue datasets into aligned records by period.\nWhy: Enables accurate deduction-to-revenue matching for tax calculations. |
| Match Expenses to Revenue Periods | Code | Aligns expenses to revenue periods | Merge Revenue and Deductions | Calculate Tax Deductions | ## Calculate & Format\nWhat: Matches expenses to revenue periods, calculates deductions, generates tax reports.\nWhy: Produces audit-ready documentation and maximizes legitimate tax advantages. |
| Calculate Tax Deductions | Code | Computes deductions and net amount per period | Match Expenses to Revenue Periods | Format Tax Report | ## Calculate & Format\nWhat: Matches expenses to revenue periods, calculates deductions, generates tax reports.\nWhy: Produces audit-ready documentation and maximizes legitimate tax advantages. |
| Format Tax Report | Set | Formats output report fields | Calculate Tax Deductions | Send to Tax Agent | ## Calculate & Format\nWhat: Matches expenses to revenue periods, calculates deductions, generates tax reports.\nWhy: Produces audit-ready documentation and maximizes legitimate tax advantages. |
| Send to Tax Agent | Gmail | Email report to agent | Format Tax Report | — | ## Setup Steps  \n1. Configure receipt storage source and OpenAI Chat Model API key.\n2. Connect Gmail for report delivery and set up tax agent email.\n3. Define expense categories, revenue periods, and deduction rules.\n4. Schedule monthly trigger and test extraction |
| Sticky Note1 | Sticky Note | Workspace annotation | — | — | ## Prerequisites  \nExpense receipt repository; OpenAI API key; Gmail account; revenue data source.\n\n## Use Cases\nAccountants automating receipt processing for multiple clients; \n\n## Customization\nAdjust extraction prompts for industry-specific expenses, modify deduction rules \n\n## Benefits\nEliminates manual receipt review, reduces categorization errors |
| Sticky Note2 | Sticky Note | Workspace annotation | — | — | ## Setup Steps  \n1. Configure receipt storage source and OpenAI Chat Model API key.\n2. Connect Gmail for report delivery and set up tax agent email.\n3. Define expense categories, revenue periods, and deduction rules.\n4. Schedule monthly trigger and test extraction |
| Sticky Note3 | Sticky Note | Workspace annotation | — | — | ## How It Works  \nThis workflow automates monthly tax processing by ingesting expense receipts alongside revenue data, extracting structured deduction details using GPT-4, and accurately matching expenses to their corresponding revenue periods. It retrieves receipts with built-in type validation, parses deduction information through OpenAI structured output extraction, and consolidates revenue records into a unified dataset. The system then intelligently aligns expenses with revenue timelines, calculates eligible deductions, and generates well-formatted tax reports that are automatically sent to designated agents via Gmail. Designed for accountants, tax professionals, and finance teams, it enables automated expense categorization and optimized deduction calculations. |
| Sticky Note5 | Sticky Note | Workspace annotation | — | — | ## Calculate & Format\nWhat: Matches expenses to revenue periods, calculates deductions, generates tax reports.\nWhy: Produces audit-ready documentation and maximizes legitimate tax advantages. |
| Sticky Note6 | Sticky Note | Workspace annotation | — | — | ## Merge Revenue & Expenses\nWhat: Consolidates expense and revenue datasets into aligned records by period.\nWhy: Enables accurate deduction-to-revenue matching for tax calculations. |
| Sticky Note7 | Sticky Note | Workspace annotation | — | — | ## Fetch & Validate Expenses  \nWhat: Retrieves expense receipts and validates file types before processing.\nWhy: Ensures data quality and prevents processing errors from invalid formats. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Monthly Tax Processing Schedule”**
   - Node type: **Schedule Trigger**
   - Set interval: **Every 1 month**, trigger at **09:00**.

2) **Create “Workflow Configuration” (Set)**
   - Add fields:
     - `expenseReceiptsApiUrl` (string): set to your receipts API endpoint
     - `revenueDataApiUrl` (string): set to your revenue API endpoint
     - `taxAgentEmail` (string): recipient email
     - `currentTaxYear` (number): expression `new Date().getFullYear()`
   - Enable “Include Other Fields”.

3) **Create “Fetch Expense Receipts” (HTTP Request)**
   - URL: expression `$('Workflow Configuration').first().json.expenseReceiptsApiUrl`
   - Response: **File** (binary).
   - Add authentication/headers as required by your receipts API (not present in JSON, but typically needed).

4) **Create “Check Receipt File Type” (IF)**
   - Condition: `$binary` **exists**.
   - Connect **true** output to the next node.
   - (Recommended when rebuilding) Add a false-path handler (e.g., send an alert / stop with message).

5) **Create “Extract Receipt Data” (Extract From File)**
   - Operation: **PDF** extraction.

6) **Create “OpenAI Chat Model” (LangChain OpenAI Chat Model)**
   - Choose model: `gpt-4.1-mini`
   - Configure **OpenAI API credentials** in n8n:
     - Create credential: **OpenAI API**
     - Paste API key, set base URL if needed.

7) **Create “Structured Output Parser” (LangChain Structured Output Parser)**
   - Schema type: **Manual**
   - Paste the schema with fields: vendor, expenseDate, amount, category, deductibilityStatus, deductionPercentage, notes.

8) **Create “Extract Deductible Categories” (LangChain Agent)**
   - Prompt type: “Define”
   - Text: `Receipt data: {{ $json.text }}`
   - System message: tax deduction specialist instructions (as in workflow)
   - Enable structured output parsing and connect:
     - AI language model input ← **OpenAI Chat Model**
     - AI output parser input ← **Structured Output Parser**
   - Connect main input from **Extract Receipt Data**.

9) **Create “Fetch Revenue Data” (HTTP Request)**
   - URL: expression `$('Workflow Configuration').first().json.revenueDataApiUrl`
   - Response format: **JSON**
   - Add auth/headers required by the revenue API.

10) **Create “Merge Revenue and Deductions” (Merge)**
   - Mode: **Combine**
   - Combine by: **Combine All**
   - Connect:
     - Input 0 ← **Extract Deductible Categories**
     - Input 1 ← **Fetch Revenue Data**

11) **Create “Match Expenses to Revenue Periods” (Code)**
   - Paste the provided JS code.
   - Connect input ← **Merge Revenue and Deductions**
   - Note: To make it functional, you will likely need to adapt it to your actual data shape (see mismatches described above).

12) **Create “Calculate Tax Deductions” (Code)**
   - Paste the provided JS code.
   - Connect input ← **Match Expenses to Revenue Periods**
   - Note: Also requires adapting field names/types to match the previous node output.

13) **Create “Format Tax Report” (Set)**
   - Set:
     - `reportTitle` = `Tax Deduction Report - {{ currentTaxYear }}`
     - `generatedDate` = `new Date().toISOString()`
     - And map totals (but ensure upstream provides `totalRevenue/totalDeductions/netIncome`, or update mappings to match upstream outputs).

14) **Create “Send to Tax Agent” (Gmail)**
   - Operation: **Send**
   - To: `$('Workflow Configuration').first().json.taxAgentEmail`
   - Subject: `Tax Deduction Report - {{ currentTaxYear }}`
   - Body: summary fields from the report
   - Configure **Gmail OAuth2 credentials** in n8n:
     - Create credential: **Gmail OAuth2**
     - Complete OAuth consent and connect the mailbox used to send emails.

15) **Connect nodes in order**
   - Schedule Trigger → Workflow Configuration  
   - Workflow Configuration → Fetch Expense Receipts → Check Receipt File Type → Extract Receipt Data → Extract Deductible Categories → Merge  
   - Workflow Configuration → Fetch Revenue Data → Merge  
   - Merge → Match Expenses to Revenue Periods → Calculate Tax Deductions → Format Tax Report → Send to Tax Agent

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Expense receipt repository; OpenAI API key; Gmail account; revenue data source. | Prerequisites (Sticky Note “Prerequisites”) |
| Configure receipt storage source and OpenAI Chat Model API key; connect Gmail; define categories/periods/rules; schedule monthly trigger and test extraction | Setup guidance (Sticky Note “Setup Steps”) |
| Workflow automates monthly tax processing: ingest receipts + revenue, GPT structured extraction, match periods, calculate deductions, generate and email report | How it works (Sticky Note “How It Works”) |
| Produces audit-ready documentation and maximizes legitimate tax advantages | Rationale (Sticky Note “Calculate & Format”) |

**Important implementation warning (current JSON):**  
Several downstream nodes expect fields (`expenses[]`, `revenue[]`, `expense.date`, `revenuePeriod`, `expenseAmount`, `totalRevenue`, `netIncome`, etc.) that are not produced by upstream nodes as currently configured. To make the workflow operational, you will need to add mapping steps (typically a **Set** or **Code** node) to normalize the AI output + revenue API output into the shapes expected by the matching and calculation code, or rewrite those code nodes to use the actual fields (`expenseDate`, `amount`, etc.).