Automated tax filing with multi-platform revenue analysis using GPT-4

https://n8nworkflows.xyz/workflows/automated-tax-filing-with-multi-platform-revenue-analysis-using-gpt-4-12029


# Automated tax filing with multi-platform revenue analysis using GPT-4

## 1. Workflow Overview

**Workflow name (in JSON):** *Autonomous Revenue Tax Agent Filing System*  
**Provided title:** *Automated tax filing with multi-platform revenue analysis using GPT-4*  

**Purpose:**  
Automatically collects revenue transactions from multiple platforms (Stripe, PayPal, Shopify, bank feed) for a defined period, normalizes them into a unified transaction schema, uses GPT‑4 to categorize each transaction into tax-relevant categories, summarizes totals by category, formats submission payloads (CSV/XML/JSON), then either submits to a tax API (if enabled) or emails a tax agent, and finally archives the output to Google Drive.

**Primary use cases:**
- Accounting/finance teams automating daily (configured) revenue aggregation and tax-category rollups
- Preparing periodic tax submission packets and audit-ready archives

**Logical blocks (by dependency and function):**
1. **1.1 Scheduling & Configuration** – defines the reporting period and operational settings (API/email/archive).
2. **1.2 Revenue Collection (Multi-source)** – pulls data from Stripe, PayPal, Shopify, and a bank feed API.
3. **1.3 Merge & Normalization** – combines disparate datasets and maps them into a common schema.
4. **1.4 AI Tax Categorization (GPT‑4 + structured output)** – classifies transactions into tax categories with confidence.
5. **1.5 Aggregation & Submission Formatting** – summarizes totals and generates CSV/XML/JSON payloads.
6. **1.6 Submission Routing (API vs Email) & Archival** – chooses submission channel and archives outputs.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Configuration

**Overview:**  
Triggers the workflow on a schedule and sets core variables like date range and destination settings used by downstream nodes.

**Nodes involved:**
- Daily Revenue Collection Schedule
- Workflow Configuration

#### Node: Daily Revenue Collection Schedule
- **Type / role:** `Schedule Trigger` – entry point; runs workflow automatically.
- **Configuration (interpreted):** Triggers daily at **02:00** (node rule uses `triggerAtHour: 2`).
- **Connections:**
  - **Output →** Workflow Configuration
- **Potential failures / edge cases:**
  - Timezone differences: schedule time depends on n8n instance timezone.
  - If the workflow should be “monthly” as sticky notes imply, this schedule currently does *daily*.

#### Node: Workflow Configuration
- **Type / role:** `Set` – creates/overrides workflow parameters.
- **Key fields set:**
  - `startDate` = yesterday start-of-day (ISO)
  - `endDate` = yesterday end-of-day (ISO)
  - `taxApiUrl` = placeholder
  - `taxApiEnabled` = placeholder (string, expected `"true"` or `"false"`)
  - `taxAgentEmail` = placeholder
  - `archiveFolderId` = placeholder (Google Drive folder ID)
- **Expressions / variables:**
  - Uses `$now.minus({ days: 1 }).startOf("day").toISO()` and `.endOf("day")`
- **Connections:**
  - **Output →** Get Stripe Transactions, Get PayPal Transactions, Get Shopify Orders, Get Bank Feed Data (fan-out)
- **Potential failures / edge cases:**
  - `taxApiEnabled` is treated as a **string** later (`equals "true"`). If set to boolean `true`, the IF check may fail.
  - PayPal node references `payoutBatchId`, but this node does **not** define it (likely misconfiguration; see Block 2.2).
  - Placeholders must be replaced before production use.

---

### 2.2 Revenue Collection (Multi-source)

**Overview:**  
Fetches revenue/transactional data in parallel from Stripe, PayPal, Shopify, and a bank feed API.

**Nodes involved:**
- Get Stripe Transactions
- Get PayPal Transactions
- Get Shopify Orders
- Get Bank Feed Data

#### Node: Get Stripe Transactions
- **Type / role:** `Stripe` – retrieves Stripe charges.
- **Configuration:**
  - Resource: `charge`
  - Operation: `getAll`
  - Limit: `100`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Merge All Revenue Sources (input 0)
- **Credential requirements:** Stripe credentials must be configured in n8n (not included in JSON).
- **Potential failures / edge cases:**
  - Pagination: `limit: 100` may not retrieve all charges for the period (and no date filter is applied here).
  - Missing time-window filtering: unlike Shopify/Bank, Stripe node has no `startDate/endDate` filter configured, so it may return the most recent charges rather than “yesterday”.

#### Node: Get PayPal Transactions
- **Type / role:** `PayPal` – retrieves PayPal payout batch details (as configured).
- **Configuration:**
  - Operation: `get`
  - `payoutBatchId` = `{{ $('Workflow Configuration').first().json.payoutBatchId }}`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Merge All Revenue Sources (input 1)
- **Credential requirements:** PayPal credentials configured in n8n.
- **Potential failures / edge cases (important):**
  - **Likely broken by default:** `payoutBatchId` is referenced but never set in “Workflow Configuration”. This will evaluate to `undefined`, causing PayPal API errors.
  - Semantics mismatch: for “transactions”, PayPal often requires a date-range transaction search endpoint, not a payout batch “get”.

#### Node: Get Shopify Orders
- **Type / role:** `Shopify` – retrieves Shopify orders in a date window.
- **Configuration:**
  - Operation: `getAll`
  - Limit: `100`
  - Filters:
    - `createdAtMin` = `startDate`
    - `createdAtMax` = `endDate`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Merge All Revenue Sources (input 2)
- **Credential requirements:** Shopify credentials configured in n8n.
- **Potential failures / edge cases:**
  - Pagination/limit may omit orders if more than 100/day.
  - Timezone issues between Shopify store time and `$now` in n8n.

#### Node: Get Bank Feed Data
- **Type / role:** `HTTP Request` – calls an external bank feed API.
- **Configuration:**
  - URL: placeholder
  - Auth: `predefinedCredentialType` (must be created in n8n)
  - Query parameters:
    - `start_date` = `startDate`
    - `end_date` = `endDate`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Merge All Revenue Sources (input 3)
- **Potential failures / edge cases:**
  - Bank API auth errors, rate limits, schema drift.
  - If API returns nested data, downstream normalization may not find expected fields (`amount`, `date`, etc.).

---

### 2.3 Merge & Normalization

**Overview:**  
Combines outputs from all revenue sources and maps each record into a consistent transaction structure.

**Nodes involved:**
- Merge All Revenue Sources
- Normalize Revenue Data

#### Node: Merge All Revenue Sources
- **Type / role:** `Merge` – combines 4 inputs.
- **Configuration:**
  - Mode: `combine`
  - Combine by: `position`
  - Number of inputs: `4`
- **Connections:**
  - **Inputs ←** Stripe (0), PayPal (1), Shopify (2), Bank (3)
  - **Output →** Normalize Revenue Data
- **Potential failures / edge cases:**
  - **Combine-by-position is risky:** it pairs item #1 from Stripe with item #1 from PayPal, etc., producing “wide” merged items. If sources have different item counts, results become misaligned or incomplete.
  - If the intent is to concatenate all transactions into one list, a different merge mode (append) is usually needed.

#### Node: Normalize Revenue Data
- **Type / role:** `Set` – maps each incoming item to normalized fields.
- **Fields produced (expressions):**
  - `transactionId` = `$json.id || $json.transaction_id || $json.order_id`
  - `amount` = `$json.amount || $json.total || $json.value`
  - `currency` = `$json.currency || "USD"`
  - `date` = `$json.created || $json.created_at || $json.date`
  - `description` = `$json.description || $json.memo || $json.name || ""`
  - `source` = `{{ $json.source || $runIndex === 0 ? "Stripe" : $runIndex === 1 ? "PayPal" : $runIndex === 2 ? "Shopify" : "Bank" }}`
- **Connections:**
  - **Input ←** Merge All Revenue Sources
  - **Output →** AI Income Categorizer
- **Potential failures / edge cases:**
  - If the merged item is a combined object with multiple embedded source objects, these expressions may not find the correct fields.
  - `$runIndex` does **not** reliably represent the originating source after a merge; it represents execution index, not “which input”.
  - Amount types: Stripe amounts can be in cents; Shopify totals may be strings; normalization doesn’t handle unit conversion or parsing.

---

### 2.4 AI Tax Categorization (GPT‑4 + structured output)

**Overview:**  
Uses a LangChain agent node with a GPT‑4 model to categorize each normalized transaction into tax categories, returning structured JSON (category/subcategory/taxable/confidence).

**Nodes involved:**
- AI Income Categorizer
- OpenAI GPT-4
- Structured Category Output

#### Node: AI Income Categorizer
- **Type / role:** `LangChain Agent` – orchestrates prompt + model + output parsing.
- **Prompt/input text:**
  - `Transaction: {{description}}, Amount: {{amount}} {{currency}}, Source: {{source}}`
- **System message:** Tax categorization specialist; must output:
  - `category` (one of: product_sales, service_revenue, interest_income, rental_income, other_income)
  - `subcategory` (freeform)
  - `taxable` (boolean)
  - `confidence` (0–1)
- **Connections:**
  - **Input ←** Normalize Revenue Data
  - **AI Language Model input ←** OpenAI GPT-4 node (via `ai_languageModel`)
  - **Output parser input ←** Structured Category Output (via `ai_outputParser`)
  - **Main Output →** Calculate Category Totals
- **Potential failures / edge cases:**
  - LLM may output invalid JSON if prompt adherence fails; structured parser mitigates but can still fail if output is non-conformant.
  - Without explicit instruction to copy through `amount`, downstream summarization expects `amount` to still exist (depends on how the agent node emits fields in your n8n version).

#### Node: OpenAI GPT-4
- **Type / role:** `LangChain Chat Model (OpenAI)` – provides GPT model for the agent.
- **Configuration:**
  - Model: `gpt-4o`
- **Credentials:** OpenAI API credential required (`openAiApi`).
- **Connections:**
  - **Output →** AI Income Categorizer (language model)
- **Potential failures / edge cases:**
  - API quota/rate limits, model access restrictions, network timeouts.

#### Node: Structured Category Output
- **Type / role:** `Structured Output Parser` – enforces JSON schema.
- **Schema:** object with properties:
  - `category` (string)
  - `subcategory` (string)
  - `taxable` (boolean)
  - `confidence` (number)
- **Connections:**
  - **Output →** AI Income Categorizer (output parser)
- **Potential failures / edge cases:**
  - If LLM returns category outside allowed set, it still passes schema (schema doesn’t restrict enum); downstream grouping may create unexpected categories.

---

### 2.5 Aggregation & Submission Formatting

**Overview:**  
Aggregates transaction amounts by AI-assigned category, then generates CSV, XML, and JSON payloads plus totals and timestamps.

**Nodes involved:**
- Calculate Category Totals
- Format Tax Submission Data

#### Node: Calculate Category Totals
- **Type / role:** `Summarize` – group-and-sum.
- **Configuration:**
  - Split by: `category`
  - Summarize: sum of `amount`
- **Connections:**
  - **Input ←** AI Income Categorizer
  - **Output →** Format Tax Submission Data
- **Potential failures / edge cases:**
  - If `amount` is missing or non-numeric, sum may be incorrect or null.
  - If agent output doesn’t preserve `amount`, summarize can’t compute totals.

#### Node: Format Tax Submission Data
- **Type / role:** `Code` – builds multi-format submission package.
- **What it generates:**
  - `csv` string with headers and per-category totals
  - `xml` string `<TaxSubmission>…</TaxSubmission>`
  - `jsonPayload` object `{ period, revenue[], timestamp }`
  - `timestamp` ISO
  - `totalRevenue` sum of category totals
- **Dependencies:**
  - Reads `startDate` and `endDate` from **Workflow Configuration**.
- **Connections:**
  - **Input ←** Calculate Category Totals
  - **Output →** Check API Availability
- **Potential failures / edge cases (important):**
  - Currency is hardcoded to `USD` for CSV/XML/JSON output.
  - Attachments later expect binary properties `csv/xml/json`, but this node outputs them as JSON strings/objects (not binary); Gmail attachment config may fail unless converted to binary first.
  - XML does not escape special characters in category names (unlikely but possible).

---

### 2.6 Submission Routing (API vs Email) & Archival

**Overview:**  
Determines whether the Tax API is enabled; submits payload via HTTP POST or emails the tax agent. Either way, merges paths and archives to Google Drive.

**Nodes involved:**
- Check API Availability
- Submit to Tax API
- Email to Tax Agent
- Merge Submission Paths
- Archive to Google Drive

#### Node: Check API Availability
- **Type / role:** `IF` – route based on configuration.
- **Condition:**
  - `$('Workflow Configuration').first().json.taxApiEnabled` **equals** `"true"` (string comparison)
- **Connections:**
  - **Input ←** Format Tax Submission Data
  - **True output →** Submit to Tax API
  - **False output →** Email to Tax Agent
- **Potential failures / edge cases:**
  - If `taxApiEnabled` is boolean `true`, the comparison to string `"true"` fails and will always go to email path.

#### Node: Submit to Tax API
- **Type / role:** `HTTP Request` – sends JSON payload to tax API endpoint.
- **Configuration:**
  - Method: `POST`
  - URL: from `taxApiUrl`
  - Body: JSON = `$json.jsonPayload`
  - Header: `Content-Type: application/json`
  - Auth: `predefinedCredentialType`
- **Connections:**
  - **Input ←** Check API Availability (true)
  - **Output →** Merge Submission Paths
- **Potential failures / edge cases:**
  - Auth misconfiguration, 4xx/5xx responses, schema mismatch with API requirements.
  - No retry/backoff configuration shown.

#### Node: Email to Tax Agent
- **Type / role:** `Gmail` – sends formatted message with attachments.
- **Configuration:**
  - To: `taxAgentEmail`
  - Subject: `Tax Filing Submission - {{timestamp}}`
  - HTML message includes period, totalRevenue, timestamp
  - Attachments: expects binary properties `csv`, `xml`, `json`
- **Connections:**
  - **Input ←** Check API Availability (false)
  - **Output →** Merge Submission Paths
- **Credentials:** Gmail OAuth2 required.
- **Potential failures / edge cases (important):**
  - **Attachment mismatch:** no node creates binary fields `csv/xml/json`. You likely need “Move/Convert to Binary” steps (e.g., *Convert to File* or *Move Binary Data*) before emailing.
  - Gmail sending limits and OAuth scope issues.

#### Node: Merge Submission Paths
- **Type / role:** `Merge` – joins whichever submission path ran into a single flow.
- **Configuration:** Not specified (defaults). Practically used to converge branches.
- **Connections:**
  - **Inputs ←** Submit to Tax API, Email to Tax Agent
  - **Output →** Archive to Google Drive
- **Potential failures / edge cases:**
  - Depending on merge mode default, it may wait for both branches (which never happens). In n8n, to “join” alternative branches safely, you often use a node that continues with whichever input arrives or configure merge mode appropriately.

#### Node: Archive to Google Drive
- **Type / role:** `Google Drive` – stores an archive file.
- **Configuration:**
  - Drive: “My Drive”
  - Folder ID: from `archiveFolderId`
  - File name: `tax_archive_<timestamp>.json`
- **Connections:**
  - **Input ←** Merge Submission Paths
- **Credentials:** Google Drive OAuth2 required.
- **Potential failures / edge cases:**
  - The node doesn’t specify file content/media/binary property; naming alone isn’t enough to upload a JSON file. Typically you must provide binary data or “file content” depending on operation.
  - Folder ID permissions: service account/user must have access.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Revenue Collection Schedule | scheduleTrigger | Time-based entry point | — | Workflow Configuration | ## Fetch Revenue Data\nWhat: Collects transactions from Stripe, PayPal, Shopify, and bank feeds simultaneously.\nWhy: Provides comprehensive revenue visibility across all income. |
| Workflow Configuration | set | Defines period and runtime settings | Daily Revenue Collection Schedule | Get Stripe Transactions; Get PayPal Transactions; Get Shopify Orders; Get Bank Feed Data | ## Fetch Revenue Data\nWhat: Collects transactions from Stripe, PayPal, Shopify, and bank feeds simultaneously.\nWhy: Provides comprehensive revenue visibility across all income. |
| Get Stripe Transactions | stripe | Pull Stripe charges | Workflow Configuration | Merge All Revenue Sources | ## Fetch Revenue Data\nWhat: Collects transactions from Stripe, PayPal, Shopify, and bank feeds simultaneously.\nWhy: Provides comprehensive revenue visibility across all income. |
| Get PayPal Transactions | payPal | Pull PayPal payout batch info | Workflow Configuration | Merge All Revenue Sources | ## Fetch Revenue Data\nWhat: Collects transactions from Stripe, PayPal, Shopify, and bank feeds simultaneously.\nWhy: Provides comprehensive revenue visibility across all income. |
| Get Shopify Orders | shopify | Pull Shopify orders in date window | Workflow Configuration | Merge All Revenue Sources | ## Fetch Revenue Data\nWhat: Collects transactions from Stripe, PayPal, Shopify, and bank feeds simultaneously.\nWhy: Provides comprehensive revenue visibility across all income. |
| Get Bank Feed Data | httpRequest | Pull bank feed transactions by API | Workflow Configuration | Merge All Revenue Sources | ## Fetch Revenue Data\nWhat: Collects transactions from Stripe, PayPal, Shopify, and bank feeds simultaneously.\nWhy: Provides comprehensive revenue visibility across all income. |
| Merge All Revenue Sources | merge | Combine source datasets | Get Stripe Transactions; Get PayPal Transactions; Get Shopify Orders; Get Bank Feed Data | Normalize Revenue Data | ## Normalize & Merge\nWhat: Consolidates multi-source data into unified records.\nWhy: Enables accurate analysis by eliminating  inconsistencies. |
| Normalize Revenue Data | set | Map to unified transaction schema | Merge All Revenue Sources | AI Income Categorizer | ## Normalize & Merge\nWhat: Consolidates multi-source data into unified records.\nWhy: Enables accurate analysis by eliminating  inconsistencies. |
| AI Income Categorizer | langchain.agent | Categorize each transaction for tax | Normalize Revenue Data; OpenAI GPT-4; Structured Category Output | Calculate Category Totals | ## AI Tax Analysis\nWhat: Applies OpenAI GPT-4 structured analysis to categorize income and forecast tax obligations.\nWhy: Delivers intelligent, context-aware tax projections beyond basic calculations. |
| OpenAI GPT-4 | lmChatOpenAi | LLM used by agent | — | AI Income Categorizer | ## AI Tax Analysis\nWhat: Applies OpenAI GPT-4 structured analysis to categorize income and forecast tax obligations.\nWhy: Delivers intelligent, context-aware tax projections beyond basic calculations. |
| Structured Category Output | outputParserStructured | Enforce structured LLM output | — | AI Income Categorizer | ## AI Tax Analysis\nWhat: Applies OpenAI GPT-4 structured analysis to categorize income and forecast tax obligations.\nWhy: Delivers intelligent, context-aware tax projections beyond basic calculations. |
| Calculate Category Totals | summarize | Group and sum by category | AI Income Categorizer | Format Tax Submission Data | ## Generate & Distribute\nWhat: Creates formatted reports, sends via Gmail to agents.\nWhy: Ensures real-time access, compliance documentation, and audit readiness. |
| Format Tax Submission Data | code | Build CSV/XML/JSON package | Calculate Category Totals | Check API Availability | ## Generate & Distribute\nWhat: Creates formatted reports, sends via Gmail to agents.\nWhy: Ensures real-time access, compliance documentation, and audit readiness. |
| Check API Availability | if | Route: API submit vs email | Format Tax Submission Data | Submit to Tax API; Email to Tax Agent | ## Generate & Distribute\nWhat: Creates formatted reports, sends via Gmail to agents.\nWhy: Ensures real-time access, compliance documentation, and audit readiness. |
| Submit to Tax API | httpRequest | POST payload to tax endpoint | Check API Availability | Merge Submission Paths | ## Generate & Distribute\nWhat: Creates formatted reports, sends via Gmail to agents.\nWhy: Ensures real-time access, compliance documentation, and audit readiness. |
| Email to Tax Agent | gmail | Email report and attachments | Check API Availability | Merge Submission Paths | ## Generate & Distribute\nWhat: Creates formatted reports, sends via Gmail to agents.\nWhy: Ensures real-time access, compliance documentation, and audit readiness. |
| Merge Submission Paths | merge | Converge alternate branches | Submit to Tax API; Email to Tax Agent | Archive to Google Drive | ## Generate & Distribute\nWhat: Creates formatted reports, sends via Gmail to agents.\nWhy: Ensures real-time access, compliance documentation, and audit readiness. |
| Archive to Google Drive | googleDrive | Archive output file | Merge Submission Paths | — |  |
| Sticky Note1 | stickyNote | Documentation | — | — | ## Prerequisites\nStripe, PayPal, Shopify, or bank feed accounts; OpenAI API key; Gmail account .\n\n## Use Cases\nAccounting firms automating quarterly tax prep for multiple clients\n\n## Customization\nModify revenue sources, adjust GPT-4 prompts for specific tax  \n\n## Benefits\nEliminates manual tax calculations, reduces forecasting errors |
| Sticky Note2 | stickyNote | Documentation | — | — | ## Setup Steps\n1. Connect Stripe, PayPal, Shopify credentials via n8n authentication.\n2. Configure OpenAI GPT-4 API key for structured tax analysis.\n3. Connect Gmail account for report distribution and Google Sheets.\n4. Set monthly trigger schedule and customize tax category rules. |
| Sticky Note3 | stickyNote | Documentation | — | — | ## How It Works\nThis workflow automates monthly revenue aggregation from multiple financial sources, including Stripe, PayPal, Shopify, and bank feeds, while delivering intelligent tax forecasting through GPT-4–based structured analysis. It systematically retrieves revenue data, consolidates disparate datasets into a unified view, and applies GPT-4 to predict upcoming tax obligations with greater accuracy. The system then generates clearly formatted, audit-ready reports and automatically distributes tax projections to designated agents via Gmail, while securely storing all outputs in Google Sheets to maintain traceable audit trails. Designed for tax professionals, accounting firms, and finance teams, it enables accurate predictive tax planning and supports a proactive compliance strategy without the need for manual calculations or spreadsheet-driven analysis. |
| Sticky Note6 | stickyNote | Documentation | — | — | ## Generate & Distribute\nWhat: Creates formatted reports, sends via Gmail to agents.\nWhy: Ensures real-time access, compliance documentation, and audit readiness. |
| Sticky Note7 | stickyNote | Documentation | — | — | ## AI Tax Analysis\nWhat: Applies OpenAI GPT-4 structured analysis to categorize income and forecast tax obligations.\nWhy: Delivers intelligent, context-aware tax projections beyond basic calculations. |
| Sticky Note8 | stickyNote | Documentation | — | — | ## Normalize & Merge\nWhat: Consolidates multi-source data into unified records.\nWhy: Enables accurate analysis by eliminating  inconsistencies. |
| Sticky Note9 | stickyNote | Documentation | — | — | ## Fetch Revenue Data\nWhat: Collects transactions from Stripe, PayPal, Shopify, and bank feeds simultaneously.\nWhy: Provides comprehensive revenue visibility across all income. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1. Add node **Schedule Trigger** named **Daily Revenue Collection Schedule**.
   2. Set schedule to run daily at **02:00** (adjust timezone/interval as desired).

2. **Add configuration node**
   1. Add **Set** node named **Workflow Configuration** connected from the trigger.
   2. Enable “Include Other Fields”.
   3. Add fields:
      - `startDate` (String): yesterday start-of-day ISO.
      - `endDate` (String): yesterday end-of-day ISO.
      - `taxApiUrl` (String): your tax API endpoint.
      - `taxApiEnabled` (String): `"true"` or `"false"` (keep as string if you copy the IF logic).
      - `taxAgentEmail` (String): destination email.
      - `archiveFolderId` (String): Google Drive folder ID.
   4. (Optional but needed if you keep PayPal “payout batch get”): add `payoutBatchId`.

3. **Add revenue source nodes (parallel fan-out from Workflow Configuration)**
   1. **Stripe** node “Get Stripe Transactions”
      - Resource: Charges; Operation: Get Many/Get All; Limit 100.
      - Configure Stripe credentials in **Credentials**.
      - (Recommended) Add date filtering if supported; otherwise add additional filtering in a Code/IF node.
   2. **PayPal** node “Get PayPal Transactions”
      - Operation currently set to **Get** by `payoutBatchId`.
      - Configure PayPal credentials.
      - Ensure `payoutBatchId` exists, or replace node with a PayPal “search/list transactions by date” operation.
   3. **Shopify** node “Get Shopify Orders”
      - Operation: Get All; Limit 100.
      - Set `createdAtMin` = `startDate`, `createdAtMax` = `endDate`.
      - Configure Shopify credentials.
   4. **HTTP Request** node “Get Bank Feed Data”
      - Method: GET (default) with query params `start_date`, `end_date`.
      - URL: your bank feed endpoint.
      - Authentication: configure a **predefined credential type** (e.g., Header Auth, OAuth2, API Key), matching your API.

4. **Merge all sources**
   1. Add **Merge** node “Merge All Revenue Sources”.
   2. Configure for **4 inputs**.
   3. Connect Stripe→Input1, PayPal→Input2, Shopify→Input3, Bank→Input4.
   4. (Recommended) Use an append/concatenate strategy rather than combine-by-position if you want a single transaction list.

5. **Normalize data**
   1. Add **Set** node “Normalize Revenue Data”.
   2. Map fields:
      - `transactionId`, `amount`, `currency`, `date`, `description`, `source` (using expressions suited to each source’s response).
   3. Connect Merge → Normalize.

6. **AI categorization (LangChain)**
   1. Add **OpenAI Chat Model** node “OpenAI GPT-4”; select model **gpt-4o**; attach OpenAI credentials.
   2. Add **Structured Output Parser** node “Structured Category Output” with manual JSON schema for `category/subcategory/taxable/confidence`.
   3. Add **LangChain Agent** node “AI Income Categorizer”:
      - Provide the transaction text template.
      - Set system message with category rules.
      - Connect model node to agent via **AI Language Model** connection.
      - Connect output parser node to agent via **AI Output Parser** connection.
   4. Connect Normalize → Agent (main).

7. **Summarize totals**
   1. Add **Summarize** node “Calculate Category Totals”.
   2. Split/group by `category`.
   3. Sum `amount`.
   4. Connect Agent → Summarize.

8. **Format submission package**
   1. Add **Code** node “Format Tax Submission Data”.
   2. Implement logic to produce `csv`, `xml`, `jsonPayload`, `timestamp`, `totalRevenue`.
   3. Connect Summarize → Code.

9. **Routing: API vs Email**
   1. Add **IF** node “Check API Availability”.
   2. Condition: `taxApiEnabled` equals `"true"`.
   3. Connect Code → IF.

10. **Tax API submission path (true)**
   1. Add **HTTP Request** node “Submit to Tax API”.
   2. Method POST; URL = `taxApiUrl`; JSON body = `jsonPayload`.
   3. Add header `Content-Type: application/json`.
   4. Configure credentials for the API auth method.
   5. Connect IF(true) → Submit.

11. **Email path (false)**
   1. Add **Gmail** node “Email to Tax Agent”.
   2. To = `taxAgentEmail`, HTML body includes period and totals.
   3. Attachments: you must first create binary files for `csv`, `xml`, and `json` (e.g., add nodes to convert strings to file/binary, then reference those binary properties).
   4. Configure Gmail OAuth2 credentials.
   5. Connect IF(false) → Gmail.

12. **Merge paths and archive**
   1. Add **Merge** node “Merge Submission Paths” to converge the two branches (configure merge behavior so it doesn’t wait for both).
   2. Connect Submit → Merge; Gmail → Merge.
   3. Add **Google Drive** node “Archive to Google Drive”.
      - Set destination folder = `archiveFolderId`.
      - Set file name using timestamp.
      - Ensure you pass actual file content (binary) or use the correct Google Drive operation that supports text content.
   4. Connect Merge → Google Drive.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Stripe, PayPal, Shopify, or bank feed accounts; OpenAI API key; Gmail account. | Prerequisites (Sticky Note1) |
| Accounting firms automating quarterly tax prep for multiple clients | Use cases (Sticky Note1) |
| Modify revenue sources, adjust GPT-4 prompts for specific tax | Customization (Sticky Note1) |
| Eliminates manual tax calculations, reduces forecasting errors | Benefits (Sticky Note1) |
| Setup Steps: connect Stripe/PayPal/Shopify credentials; configure OpenAI; connect Gmail and Google Sheets; set monthly trigger schedule and customize tax category rules. | Setup guidance (Sticky Note2). Note: workflow uses Google Drive (not Google Sheets) and schedule is daily in JSON. |
| “How It Works” narrative describing monthly aggregation, GPT‑4 forecasting, emailing, and storing in Google Sheets | Product description (Sticky Note3). Note: current workflow archives to Google Drive, not Sheets. |
| Generate & Distribute: creates formatted reports, sends via Gmail to agents for compliance/audit readiness | Sticky Note6 |
| AI Tax Analysis: GPT‑4 structured categorization and forecasting | Sticky Note7 |
| Normalize & Merge: consolidate multi-source data | Sticky Note8 |
| Fetch Revenue Data: simultaneous collection across sources | Sticky Note9 |