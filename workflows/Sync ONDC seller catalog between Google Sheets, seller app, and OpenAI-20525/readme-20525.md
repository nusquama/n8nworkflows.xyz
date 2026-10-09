Sync ONDC seller catalog between Google Sheets, seller app, and OpenAI

https://n8nworkflows.xyz/workflows/sync-ondc-seller-catalog-between-google-sheets--seller-app--and-openai-20525


# Sync ONDC seller catalog between Google Sheets, seller app, and OpenAI

### 1. Workflow Overview

This workflow synchronizes an ONDC seller catalog managed via Google Sheets with a seller app API. It operates on a recurring 30-minute schedule or an on-demand webhook trigger. The automation detects changes between the spreadsheet and the live seller app, processes product content through OpenAI (`gpt-4o-mini`) for normalization, validates images and ONDC retail compliance rules, pushes updates in batches, and updates the tracking columns in the spreadsheet alongside sending comprehensive email summaries and AI-generated error fixes.

The operational logic is organized into the following functional blocks:

- **1.1 Triggers & Load:** Initializes the execution via schedule or webhook, sets up environment variables, retrieves the Google Sheet catalog, and queries the live seller app API.
- **1.2 Change Detection & AI Clean-up:** Identifies modified, new, duplicated, or orphaned SKUs, handles price-jump thresholds, and sends eligible rows to OpenAI for attribute standardization.
- **1.3 Image Check:** Filters and tests product image URLs via HTTP HEAD requests to verify availability, MIME types, and file sizes.
- **1.4 Validate & Build:** Applies ONDC retail rules, constructs batched upsert and inventory payloads, and handles validation failures.
- **1.5 Push & Result Compilation:** Sends API payloads to the seller app, handles HTTP responses, consolidates sync statuses, and branches out data routing.
- **1.6 Write-back, Error Log & Reporting:** Writes execution updates back to the Google Sheets catalog tab, logs sync errors, emails summary reports, and prompts OpenAI to generate actionable fix suggestions for failing rows.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggers & Load
This block initializes the automation run, configures global operational parameters, and fetches the current states of both the local spreadsheet catalog and the live remote API catalog.

- **Sync Every 30 Minutes**
  - **Type & Role:** `Schedule Trigger` node. Automatically initiates a workflow run every 30 minutes.
  - **Configuration:** Interval set to 30 minutes.
  - **Connections:** Output connects to `Set Catalog Config`.
  - **Failure Types:** None.

- **Sync Now via Webhook**
  - **Type & Role:** `Webhook` node. Allows on-demand manual or external system triggering via HTTP POST.
  - **Configuration:** HTTP Method `POST`, path `ondc-catalog-sync`.
  - **Connections:** Output connects to `Set Catalog Config`.
  - **Failure Types:** Unauthorized access if unauthenticated; connection timeout.

- **Set Catalog Config**
  - **Type & Role:** `Set (Edit Fields)` node. Centralizes configuration variables (Sheet IDs, API endpoints, ONDC identifiers, batch limits, AI modes).
  - **Configuration:** Assigns environment variables including `sheet_id`, `catalog_sheet`, `errors_sheet`, `seller_api_base`, `provider_id`, `location_id`, and operational thresholds (`price_guard_pct`, `max_image_mb`, `batch_size`).
  - **Input/Output:** Input from triggers; output connects to `Get Catalog Sheet`.
  - **Failure Types:** Missing configuration values referenced downstream.

- **Get Catalog Sheet**
  - **Type & Role:** `Google Sheets` node. Retrieves all rows from the target catalog sheet.
  - **Configuration:** Uses document ID and sheet name derived from `Set Catalog Config`. Set to execute once (`executeOnce: true`).
  - **Input/Output:** Input from `Set Catalog Config`; output connects to `Fetch Live Catalog from Seller App`.
  - **Failure Types:** Google Sheets API authentication failure, missing sheet names, or quota limits.

- **Fetch Live Catalog from Seller App**
  - **Type & Role:** `HTTP Request` node. Fetches the active catalog from the seller app API for drift and reconciliation detection.
  - **Configuration:** Generic Header Auth, GET request to `seller_api_base` combined with `catalog_list_path`. Configured to continue on regular output (`onError: "continueRegularOutput"`) with a 60-second timeout.
  - **Input/Output:** Input from `Get Catalog Sheet`; output connects to `Detect Changed Rows`.
  - **Failure Types:** API downtime, network timeouts, invalid authentication headers.

---

#### 2.2 Change Detection & AI Clean-up
This block evaluates row-level changes, checks for price drops or spikes against safety guards, flags duplicates, and routes altered content to OpenAI for standardization.

- **Detect Changed Rows**
  - **Type & Role:** `Code` (JavaScript) node. Compares spreadsheet hashes (content and stock) with live API states to categorize SKUs (`skip`, `full`, `inventory`, `disable`, `duplicate`, `price_hold`, `orphan_disable`, `orphan_report`).
  - **Configuration:** Computes custom string hashes and checks against price guard percentages.
  - **Input/Output:** Input from `Fetch Live Catalog from Seller App`; output connects to `Select Rows for AI Clean-up`.
  - **Failure Types:** JavaScript parsing errors due to malformed cell data.

- **Select Rows for AI Clean-up**
  - **Type & Role:** `Code` (JavaScript) node. Filters rows marked for full updates requiring AI processing and constructs structured prompts.
  - **Configuration:** Returns a single marker item (`skip_ai: true`) if no rows need processing.
  - **Input/Output:** Input from `Detect Changed Rows`; output connects to `Skip AI?`.
  - **Failure Types:** Expression evaluation errors.

- **Skip AI?**
  - **Type & Role:** `If` node. Evaluates whether AI processing should be bypassed.
  - **Configuration:** Checks condition `{{ $json.skip_ai }}` equals `true`.
  - **Input/Output:** Input from `Select Rows for AI Clean-up`; True output connects to `Collect Image URLs`, False output connects to `AI Clean Product Attributes`.
  - **Failure Types:** Boolean evaluation mismatches.

- **AI Clean Product Attributes**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.OpenAI` node. Standardizes product titles, descriptions, categories, units, and attributes according to ONDC guidelines.
  - **Configuration:** Uses model `gpt-4o-mini` with a temperature of `0.2` and forced JSON output.
  - **Input/Output:** Input from `Skip AI?` (False branch); output connects to `Merge AI Output into Rows`.
  - **Failure Types:** OpenAI API rate limits, authentication errors, malformed JSON responses.

- **Merge AI Output into Rows**
  - **Type & Role:** `Code` (JavaScript) node. Pairs AI responses back to their respective SKUs and normalizes fields.
  - **Configuration:** Safely parses JSON output strings and truncates fields to character limits.
  - **Input/Output:** Input from `AI Clean Product Attributes`; output connects to `Collect Image URLs`.
  - **Failure Types:** JSON parsing failures from unescaped model outputs.

---

#### 2.3 Image Check
This block verifies product image URLs to ensure they are publicly accessible, return valid status codes, and adhere to size and format restrictions.

- **Collect Image URLs**
  - **Type & Role:** `Code` (JavaScript) node. Extracts unique image URLs from items undergoing full updates.
  - **Configuration:** Enforces limits via `max_image_checks`. Emits a `skip_img: true` marker if no images are found.
  - **Input/Output:** Inputs from `Skip AI?` (True branch) and `Merge AI Output into Rows`; output connects to `Skip Image Check?`.
  - **Failure Types:** Empty URL strings or malformed URI syntax.

- **Skip Image Check?**
  - **Type & Role:** `If` node. Evaluates whether image link verification is bypassed.
  - **Configuration:** Evaluates `{{ $json.skip_img }}`.
  - **Input/Output:** Input from `Collect Image URLs`; True output connects to `Validate & Build ONDC Payloads`, False output connects to `Check Image URLs`.
  - **Failure Types:** Boolean validation errors.

- **Check Image URLs**
  - **Type & Role:** `HTTP Request` node. Sends an HTTP HEAD request to validate image URLs.
  - **Configuration:** Method `HEAD`, timeout `15000ms`, batch size of 10 with 300ms intervals. Never errors on failure.
  - **Input/Output:** Input from `Skip Image Check?` (False branch); output connects to `Summarize Image Check`.
  - **Failure Types:** DNS resolution errors, connection timeouts.

- **Summarize Image Check**
  - **Type & Role:** `Code` (JavaScript) node. Analyzes HEAD response status codes, content-types, and content-lengths to categorize image links as valid, hard errors, or warnings.
  - **Configuration:** Enforces `max_image_mb` byte limits.
  - **Input/Output:** Input from `Check Image URLs`; output connects to `Validate & Build ONDC Payloads`.
  - **Failure Types:** Missing headers in response objects.

---

#### 2.4 Validate & Build
This block verifies rows against ONDC retail compliance rules (such as statutory requirements, FSSAI licenses, pricing logic, and inventory constraints) and constructs batched API payloads.

- **Validate & Build ONDC Payloads**
  - **Type & Role:** `Code` (JavaScript) node. Evaluates domain-specific validation rules (Grocery, F&B, Fashion, etc.), applies AI clean-ups, checks inventory parameters, and generates batched upsert and inventory payloads.
  - **Configuration:** Groups payloads into chunks based on `batch_size`.
  - **Input/Output:** Inputs from `Skip Image Check?` (True branch) and `Summarize Image Check`; output connects to `Has API Batches?`.
  - **Failure Types:** Unhandled edge cases in data typing or missing mandatory fields.

- **Has API Batches?**
  - **Type & Role:** `If` node. Determines if valid API batches exist for submission.
  - **Configuration:** Checks if `{{ $json.type === 'api' }}`.
  - **Input/Output:** Input from `Validate & Build ONDC Payloads`; True output connects to `Push Items to Seller App`, False output connects to `Compile Sync Results`.
  - **Failure Types:** Expression evaluation errors.

---

#### 2.5 Push & Result Compilation
This block sends the formatted payloads to the seller app API, collects execution responses, and consolidates the outcome for post-processing.

- **Push Items to Seller App**
  - **Type & Role:** `HTTP Request` node. Sends batched catalog and inventory payloads to the seller app API.
  - **Configuration:** Method `POST`, Generic Header Auth, batch size of 1, max 2 retries with 5000ms wait intervals.
  - **Input/Output:** Input from `Has API Batches?` (True branch); output connects to `Compile Sync Results`.
  - **Failure Types:** API endpoint rejection, authentication failures, gateway timeouts.

- **Compile Sync Results**
  - **Type & Role:** `Code` (JavaScript) node. Consolidates validation failures, API responses, and orphaned items into structured write-back updates, error logs, AI fix triggers, and reporting HTML.
  - **Configuration:** Prepares HTML email tables and tracks metrics (full updates, inventory updates, failures, warnings).
  - **Input/Output:** Inputs from `Has API Batches?` (False branch) and `Push Items to Seller App`; output connects to `Route by Result Type`.
  - **Failure Types:** Array out-of-bounds errors on empty API responses.

- **Route by Result Type**
  - **Type & Role:** `Switch` node. Routes compiled operational outputs according to their action type.
  - **Configuration:** Rules match `{{ $json.type }}` against:
    - `writeback` $\rightarrow$ `Prepare Catalog Row Update`
    - `error_log` $\rightarrow$ `Log Sync Errors to Sheet`
    - `report` $\rightarrow$ `Email Sync Report`
    - `ai_fix` $\rightarrow$ `AI Suggest Fixes for Errors`
  - **Input/Output:** Input from `Compile Sync Results`; outputs connect to their respective destination nodes.
  - **Failure Types:** Unmatched type strings resulting in dropped items.

---

#### 2.6 Write-back, Error Log & Reporting
This block writes execution statuses back to the Google Sheets catalog tab, appends error records, sends email summaries, and generates AI fix suggestions for failed items.

- **Prepare Catalog Row Update**
  - **Type & Role:** `Code` (JavaScript) node. Formats individual item updates for spreadsheet cell mapping.
  - **Configuration:** Operates per item (`runOnceForEachItem`).
  - **Input/Output:** Input from `Route by Result Type` (Output 1); output connects to `Update Catalog Rows in Sheet`.
  - **Failure Types:** Property access errors on missing update objects.

- **Update Catalog Rows in Sheet**
  - **Type & Role:** `Google Sheets` node. Updates catalog rows with sync statuses, hashes, timestamps, and AI attributes.
  - **Configuration:** Operation `update`, matching column `sku`, automatic mapping.
  - **Input/Output:** Input from `Prepare Catalog Row Update`; output has no subsequent connections.
  - **Failure Types:** Google Sheets API rate limits or mismatched SKU identifiers.

- **Log Sync Errors to Sheet**
  - **Type & Role:** `Google Sheets` node. Appends rejected SKU errors into the designated error log tab.
  - **Configuration:** Operation `append`, sheet name mapped via `errors_sheet`.
  - **Input/Output:** Input from `Route by Result Type` (Output 2); output has no subsequent connections.
  - **Failure Types:** Sheet schema mismatches or missing destination tabs.

- **Email Sync Report**
  - **Type & Role:** `Gmail` node. Emails the consolidated run summary and error report.
  - **Configuration:** Uses Gmail OAuth2 credentials, recipient from `report_email`, HTML body format. Continues on regular output on error.
  - **Input/Output:** Input from `Route by Result Type` (Output 3); output has no subsequent connections.
  - **Failure Types:** Gmail API authentication expiration or invalid recipient addresses.

- **AI Suggest Fixes for Errors**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.OpenAI` node. Analyzes failing validation errors and generates corrective suggestions.
  - **Configuration:** Uses model `gpt-4o-mini`, temperature `0.2`, strict JSON output. Continues on regular output on error.
  - **Input/Output:** Input from `Route by Result Type` (Output 4); output connects to `Format Fix Suggestions`.
  - **Failure Types:** OpenAI API rate limits or token limits exceeded.

- **Format Fix Suggestions**
  - **Type & Role:** `Code` (JavaScript) node. Pairs generated AI fix suggestions with their respective SKUs.
  - **Configuration:** Truncates combined error steps to a maximum of 600 characters.
  - **Input/Output:** Input from `AI Suggest Fixes for Errors`; output connects to `Save Fix Suggestions to Sheet`.
  - **Failure Types:** JSON parsing errors from model output.

- **Save Fix Suggestions to Sheet**
  - **Type & Role:** `Google Sheets` node. Writes AI fix suggestions back to the `ai_fix_suggestion` column in the catalog sheet.
  - **Configuration:** Operation `update`, matching column `sku`, automatic mapping.
  - **Input/Output:** Input from `Format Fix Suggestions`; output has no subsequent connections.
  - **Failure Types:** Google Sheets API rate limits or missing columns.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note – Overview** | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | 🛒 ONDC Seller Catalog Sync – Google Sheets to Seller App<br><br>For small Indian sellers on ONDC who keep their catalog in a spreadsheet and lose hours to rejected listings, messy titles and prices that drift between the sheet and what buyers actually see.<br><br>### How it works<br>Every 30 minutes (or on demand) the workflow compares your catalog sheet with the live catalog in your seller app and picks out only the SKUs that changed. GPT-4o-mini tidies titles, descriptions and attributes for new or edited rows, and image links are checked before upload. Each row is validated against ONDC retail rules, then valid items are pushed to the seller app in batches. Large price jumps are held until someone ticks `price_confirmed` in the sheet.<br><br>Results are written back to the sheet, rejected SKUs are logged with the reason and an AI-suggested fix, and you get an email summary when something needs attention.<br><br>### Setup steps<br>1. Create a Google Sheet with **Catalog** and **Sync Errors** tabs.<br>2. Connect Google Sheets, Gmail and OpenAI.<br>3. Add your seller app API key as Header Auth on both seller app nodes.<br>4. Fill in **Set Catalog Config**: sheet ID, API base URL and paths, provider and location IDs, consumer care details and report email.<br>5. Set `ai_apply_mode` to `suggest` if you'd rather review AI clean-ups in the sheet before they go live. Run once manually, then activate. |
| **Sticky Note – Change Detection & AI Clean-up** | `n8n-nodes-base.stickyNote` | Documentation block for change filtering and AI processing | None | None | 🧹 Change Detection & AI Clean-up<br>Only SKUs that changed since the last sync move forward. Big price jumps are held until someone confirms them. New or edited content gets AI-cleaned titles, descriptions and attributes, so you only pay for rows that need it. |
| **Sticky Note – Image Check** | `n8n-nodes-base.stickyNote` | Documentation block for image URL verification | None | None | 🖼️ Image Check<br>Sends a quick HEAD request to each new image URL to catch broken links, wrong file types and oversized files before the seller app rejects them. |
| **Sticky Note – Validate & Push** | `n8n-nodes-base.stickyNote` | Documentation block for rule validation and API upserts | None | None | 🚀 Validate & Push<br>Checks every row against ONDC retail rules such as unit, MRP, returns and veg/non-veg, then pushes valid items to the seller app in batches. Failures are kept for the report. |
| **Sticky Note – Write-back & Error Log** | `n8n-nodes-base.stickyNote` | Documentation block for spreadsheet write-backs and logging | None | None | 🗂️ Write-back & Error Log<br>Sync status and timestamps go back to the catalog sheet, and every rejected SKU is logged with the reason. |
| **Sticky Note – Report & AI Fixes** | `n8n-nodes-base.stickyNote` | Documentation block for reporting and AI fix suggestions | None | None | 📬 Report & AI Fixes<br>Emails a run summary when something needs attention. For failed rows, the AI suggests the exact fix so your catalog team isn't left guessing. |
| **Sticky Note – Credentials & Security** | `n8n-nodes-base.stickyNote` | Security best practices documentation | None | None | 🔐 Credentials & Security<br>Keep the seller app API key in Header Auth, never in the URL or config. Connect Google Sheets and Gmail with OAuth2 and OpenAI with an API key. Keep the sync webhook URL private. |
| **Sync Every 30 Minutes** | `n8n-nodes-base.scheduleTrigger` | Triggers workflow execution on a 30-minute interval | None | `Set Catalog Config` | |
| **Sync Now via Webhook** | `n8n-nodes-base.webhook` | Triggers workflow execution on-demand via HTTP POST | None | `Set Catalog Config` | |
| **Set Catalog Config** | `n8n-nodes-base.set` | Initializes global variables and configuration parameters | `Sync Every 30 Minutes`, `Sync Now via Webhook` | `Get Catalog Sheet` | |
| **Get Catalog Sheet** | `n8n-nodes-base.googleSheets` | Retrieves all rows from the catalog spreadsheet | `Set Catalog Config` | `Fetch Live Catalog from Seller App` | |
| **Fetch Live Catalog from Seller App** | `n8n-nodes-base.httpRequest` | Fetches the active catalog from the seller app API | `Get Catalog Sheet` | `Detect Changed Rows` | |
| **Detect Changed Rows** | `n8n-nodes-base.code` | Identifies changed, new, duplicate, or drifting SKUs | `Fetch Live Catalog from Seller App` | `Select Rows for AI Clean-up` | |
| **Select Rows for AI Clean-up** | `n8n-nodes-base.code` | Prepares prompt payloads for AI content normalization | `Detect Changed Rows` | `Skip AI?` | |
| **Skip AI?** | `n8n-nodes-base.if` | Evaluates whether AI attribute clean-up should be skipped | `Select Rows for AI Clean-up` | `Collect Image URLs`, `AI Clean Product Attributes` | |
| **AI Clean Product Attributes** | `@n8n/n8n-nodes-langchain.openAi` | Normalizes product content using OpenAI (`gpt-4o-mini`) | `Skip AI?` | `Merge AI Output into Rows` | |
| **Merge AI Output into Rows** | `n8n-nodes-base.code` | Normalizes and attaches AI output back to SKU items | `AI Clean Product Attributes` | `Collect Image URLs` | |
| **Collect Image URLs** | `n8n-nodes-base.code` | Gathers unique image URLs for live verification | `Skip AI?`, `Merge AI Output into Rows` | `Skip Image Check?` | |
| **Skip Image Check?** | `n8n-nodes-base.if` | Evaluates whether image link checking is bypassed | `Collect Image URLs` | `Validate & Build ONDC Payloads`, `Check Image URLs` | |
| **Check Image URLs** | `n8n-nodes-base.httpRequest` | Sends HTTP HEAD requests to validate image links | `Skip Image Check?` | `Summarize Image Check` | |
| **Summarize Image Check** | `n8n-nodes-base.code` | Categorizes image link check results and flags errors | `Check Image URLs` | `Validate & Build ONDC Payloads` | |
| **Validate & Build ONDC Payloads** | `n8n-nodes-base.code` | Validates ONDC compliance rules and creates batched payloads | `Skip Image Check?`, `Summarize Image Check` | `Has API Batches?` | |
| **Has API Batches?** | `n8n-nodes-base.if` | Determines whether valid API payload batches exist | `Validate & Build ONDC Payloads` | `Push Items to Seller App`, `Compile Sync Results` | |
| **Push Items to Seller App** | `n8n-nodes-base.httpRequest` | Pushes batched catalog and inventory payloads to the seller app | `Has API Batches?` | `Compile Sync Results` | |
| **Compile Sync Results** | `n8n-nodes-base.code` | Consolidates API responses, validation failures, and report data | `Has API Batches?`, `Push Items to Seller App` | `Route by Result Type` | |
| **Route by Result Type** | `n8n-nodes-base.switch` | Routes execution flow based on operational result types | `Compile Sync Results` | `Prepare Catalog Row Update`, `Log Sync Errors to Sheet`, `Email Sync Report`, `AI Suggest Fixes for Errors` | |
| **Prepare Catalog Row Update** | `n8n-nodes-base.code` | Formats individual updates for spreadsheet cell mapping | `Route by Result Type` | `Update Catalog Rows in Sheet` | |
| **Log Sync Errors to Sheet** | `n8n-nodes-base.googleSheets` | Appends rejected SKU error logs into the errors tab | `Route by Result Type` | None | |
| **Email Sync Report** | `n8n-nodes-base.gmail` | Sends the HTML run summary report via Gmail | `Route by Result Type` | None | |
| **AI Suggest Fixes for Errors** | `@n8n/n8n-nodes-langchain.openAi` | Generates AI fix instructions for validation errors | `Route by Result Type` | `Format Fix Suggestions` | |
| **Update Catalog Rows in Sheet** | `n8n-nodes-base.googleSheets` | Updates catalog rows in the spreadsheet with sync statuses | `Prepare Catalog Row Update` | None | |
| **Format Fix Suggestions** | `n8n-nodes-base.code` | Pairs AI fix suggestions with their respective SKUs | `AI Suggest Fixes for Errors` | `Save Fix Suggestions to Sheet` | |
| **Save Fix Suggestions to Sheet** | `n8n-nodes-base.googleSheets` | Saves AI fix suggestions back to the catalog sheet | `Format Fix Suggestions` | None | |
| **Sticky Note – Triggers & Load** | `n8n-nodes-base.stickyNote` | Documentation block for triggers and data loading | None | None | 📥 Triggers & Load<br>Runs every 30 minutes, or on demand from the webhook. Loads your catalog sheet and the live catalog from the seller app so the two can be compared. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow from scratch in n8n, perform the following steps:

1. **Create Triggers & Configuration:**
   - Add a **Schedule Trigger** node (`Sync Every 30 Minutes`) configured with an interval of 30 minutes.
   - Add a **Webhook** node (`Sync Now via Webhook`) with method `POST` and path `ondc-catalog-sync`.
   - Add a **Set** node (`Set Catalog Config`) connected to both triggers. Populate assignment fields for `sheet_id`, `catalog_sheet` ("Catalog"), `errors_sheet` ("Sync Errors"), `seller_api_base`, `provider_id`, `location_id`, `consumer_care`, and operational thresholds (`batch_size: 50`, `price_guard_pct: 30`, `ai_enabled: true`, `ai_apply_mode: auto`).

2. **Load Data:**
   - Add a **Google Sheets** node (`Get Catalog Sheet`) connected after `Set Catalog Config`. Set operation to retrieve all rows using the sheet ID and name from the config node. Enable *Execute Once*.
   - Add an **HTTP Request** node (`Fetch Live Catalog from Seller App`) connected after `Get Catalog Sheet`. Configure Generic Header Auth, GET method, and URL combining `seller_api_base` and `catalog_list_path`. Set error handling to continue on regular output.

3. **Change Detection & AI Processing:**
   - Add a **Code** node (`Detect Changed Rows`) containing the change detection, hashing, and price-guard logic.
   - Add a **Code** node (`Select Rows for AI Clean-up`) to isolate rows requiring full updates.
   - Add an **If** node (`Skip AI?`) checking `{{ $json.skip_ai }}`.
   - If false, add an **OpenAI** node (`AI Clean Product Attributes`) using model `gpt-4o-mini` with temperature `0.2` and JSON output enabled.
   - Add a **Code** node (`Merge AI Output into Rows`) to process model outputs.

4. **Image Verification:**
   - Add a **Code** node (`Collect Image URLs`) to gather unique image links.
   - Add an **If** node (`Skip Image Check?`) evaluating `{{ $json.skip_img }}`.
   - If false, add an **HTTP Request** node (`Check Image URLs`) with method `HEAD` and batching enabled (batch size 10, interval 300ms).
   - Add a **Code** node (`Summarize Image Check`) to parse HTTP status codes and headers.

5. **Validation & Payloads:**
   - Add a **Code** node (`Validate & Build ONDC Payloads`) to validate ONDC retail compliance rules and generate batched upsert and inventory payloads.
   - Add an **If** node (`Has API Batches?`) checking `{{ $json.type === 'api' }}`.

6. **API Submission & Results Compilation:**
   - Add an **HTTP Request** node (`Push Items to Seller App`) connected to the True branch of `Has API Batches?`. Set method `POST`, Generic Header Auth, batch size 1, and retries.
   - Add a **Code** node (`Compile Sync Results`) connected to both the False branch of `Has API Batches?` and `Push Items to Seller App`. Consolidates outputs into write-backs, error logs, reports, and AI fixes.
   - Add a **Switch** node (`Route by Result Type`) to route items based on `{{ $json.type }}` (`writeback`, `error_log`, `report`, `ai_fix`).

7. **Write-backs, Logging & Reporting:**
   - Route `writeback` to a **Code** node (`Prepare Catalog Row Update`) followed by a **Google Sheets** node (`Update Catalog Rows in Sheet`) updating rows where matching column is `sku`.
   - Route `error_log` to a **Google Sheets** node (`Log Sync Errors to Sheet`) appending rows to the errors sheet.
   - Route `report` to a **Gmail** node (`Email Sync Report`) sending HTML reports to `report_email`.
   - Route `ai_fix` to an **OpenAI** node (`AI Suggest Fixes for Errors`) using model `gpt-4o-mini`, followed by a **Code** node (`Format Fix Suggestions`) and a **Google Sheets** node (`Save Fix Suggestions to Sheet`) updating the catalog sheet.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Seller app API keys must be stored in n8n Header Auth credentials, never hardcoded in URLs or configuration variables. | Security Best Practices |
| Keep the webhook URL private to prevent unauthorized on-demand synchronization triggers. | Webhook Security |
| Ensure Google Sheets and Gmail OAuth2 credentials are correctly authenticated and granted permissions to access target spreadsheet and mailboxes. | Credential Setup |