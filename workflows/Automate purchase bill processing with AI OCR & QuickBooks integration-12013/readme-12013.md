Automate purchase bill processing with AI OCR & QuickBooks integration

https://n8nworkflows.xyz/workflows/automate-purchase-bill-processing-with-ai-ocr---quickbooks-integration-12013


# Automate purchase bill processing with AI OCR & QuickBooks integration

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow ingests purchase invoice PDFs from an n8n web form, extracts structured invoice fields via AI-powered OCR/text understanding, reconciles invoice line items and vendors against QuickBooks, auto-creates any missing QuickBooks Items/Vendors, then creates a **QuickBooks Bill** containing the invoice’s line items and tax.

**Target use cases:**
- Automating accounts payable intake (supplier invoices) into QuickBooks.
- Reducing manual data entry by converting invoice PDFs into QuickBooks Bills.
- Standardizing vendor/item master data by auto-creating missing records.

### 1.1 Input Reception (Form Upload)
Receives one or more invoice PDFs through an n8n Form Trigger.

### 1.2 PDF Processing & Iteration
Splits uploaded binary files into separate items and loops through each invoice file.

### 1.3 Text Extraction & Cleanup
Extracts text from each PDF and normalizes it into a single cleaned string suitable for LLM extraction.

### 1.4 AI Data Extraction (LLM)
Uses an OpenRouter chat model + Information Extractor node to turn invoice text into structured JSON (vendor, dates, line items, totals).

### 1.5 Item Reconciliation & Auto-Creation (QuickBooks)
Compares extracted line item names with existing QuickBooks Items; creates missing items via QuickBooks API; produces a unified mapping of item name → QuickBooks Item Id.

### 1.6 Vendor Reconciliation & Auto-Creation (QuickBooks)
Searches vendor by name; creates vendor if missing using extracted contact info.

### 1.7 Bill Payload Construction & Bill Creation (QuickBooks)
Builds the QuickBooks Bill payload with correct Item references, quantities, unit prices, tax line, and vendor reference; posts it to QuickBooks.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception (Form Upload)
**Overview:** Provides a web form where a user uploads an invoice PDF. The node outputs binary data for the uploaded file(s).  
**Nodes involved:** `On bill submission`

#### Node: On bill submission
- **Type / role:** `Form Trigger` — entry point; hosts a public form endpoint.
- **Key configuration:**
  - **Form title:** “Invoice Parser + QuickBooks Bill Creator”
  - **Field:** File upload “Invoice File”, required, accepts `.pdf`.
- **Inputs:** None (trigger).
- **Outputs:** One item containing binary file(s) under `binary` (field names depend on form upload).
- **Potential failures / edge cases:**
  - Non-PDF upload blocked by accept types, but MIME mismatch can still occur in some environments.
  - Large PDFs may exceed n8n binary limits (depends on instance config).
- **Version notes:** Form Trigger `typeVersion 2.3`.

Sticky note context:
- **Workflow Overview** (applies generally): setup steps and how it works.

---

### Block 1.2 — PDF Processing & Iteration
**Overview:** Converts a single form submission that may contain multiple binary entries into separate workflow items, then iterates through them with batching.  
**Nodes involved:** `Convert to Separate Items`, `Loop Over Invoices`

#### Node: Convert to Separate Items
- **Type / role:** `Code` — splits multiple binaries into individual items.
- **Configuration choices (interpreted):**
  - Iterates over `item.binary` keys; for each key, emits a new item:
    - `json.file`, `json.filename` set to key
    - keeps `binary.data` for downstream PDF extraction
- **Key variables/expressions:** Uses `$input.first()` and `item.binary`.
- **Inputs:** Output from `On bill submission`.
- **Outputs:** Multiple items, each with one binary `data` property.
- **Edge cases / failures:**
  - If no binary exists, returns empty output → downstream nodes receive nothing.
  - If binary key names are unexpected, `filename` becomes that key rather than original filename.

#### Node: Loop Over Invoices
- **Type / role:** `Split In Batches` — iterates invoice items.
- **Configuration choices:** Default options (batch size default in n8n UI if not set explicitly).
- **Inputs:** From `Convert to Separate Items`.
- **Outputs / connections:**
  - **Main output (index 1)** goes to `Extract from PDF` (this is the loop “current item” path in this workflow).
  - After bill creation, `Create Bill` connects back to `Loop Over Invoices` to continue iteration.
- **Edge cases / failures:**
  - Mis-wiring batch outputs can cause no iteration; here the workflow relies on the second output.
  - If an iteration fails mid-way, remaining invoices won’t process unless error handling is added.

Sticky note context:
- **PDF Processing:** “Converts uploaded PDFs to separate items and loops through each invoice for extraction.”

---

### Block 1.3 — Text Extraction & Cleanup
**Overview:** Extracts text from a PDF, then cleans it (newline normalization, trimming) to improve LLM extraction reliability.  
**Nodes involved:** `Extract from PDF`, `Clean Text`

#### Node: Extract from PDF
- **Type / role:** `Extract from File` — PDF text extraction.
- **Configuration choices:**
  - **Operation:** `pdf`
  - Uses incoming binary `data`.
- **Inputs:** One invoice item from `Loop Over Invoices`.
- **Outputs:** JSON with extracted `text` (typical for this node).
- **Edge cases / failures:**
  - Scanned/image-only PDFs may yield little/no text (no OCR here unless the node supports it in your environment).
  - Password-protected or corrupted PDFs cause extraction errors.

#### Node: Clean Text
- **Type / role:** `Code` — normalizes extracted text.
- **Configuration choices (interpreted):**
  - Reads `$json["text"]` first.
  - If binary exists, attempts to read `$binary.data.data` as UTF-8 (fallback).
  - Replaces `\r\n` with `\n`, collapses multiple newlines, trims.
- **Outputs:** `{ cleaned_text: "<normalized text>" }`
- **Edge cases / failures:**
  - If extracted text is empty and binary decode is not meaningful, `cleaned_text` may be empty → AI extractor likely fails or returns blanks.
  - Very large text may exceed LLM context.

Sticky note context:
- Covered under **PDF Processing** area in the canvas.

---

### Block 1.4 — AI Data Extraction (LLM)
**Overview:** Uses OpenRouter LLM and a schema-driven extractor to convert invoice text into structured JSON fields.  
**Nodes involved:** `OpenRouter Chat Model`, `Extract Invoice Data`

#### Node: OpenRouter Chat Model
- **Type / role:** `LangChain Chat Model (OpenRouter)` — provides the LLM for downstream extraction.
- **Configuration choices:**
  - **Model:** `openai/gpt-oss-20b:free`
- **Connections:** Wired to `Extract Invoice Data` via the `ai_languageModel` connection.
- **Credential requirements:** OpenRouter API key configured in n8n credentials for this node type.
- **Edge cases / failures:**
  - Rate limits / quota for free model.
  - Model availability changes (OpenRouter can deprecate “free” variants).
  - Latency/timeouts for large PDFs.

#### Node: Extract Invoice Data
- **Type / role:** `Information Extractor` — schema-based structured extraction from text.
- **Configuration choices:**
  - **Input text:** `{{$json.cleaned_text}}`
  - **System prompt:** “Extract invoice data accurately… Return numbers as numbers, not strings.”
  - **Schema mode:** `fromJson` using the provided JSON schema example (invoice_number, dates, vendor object, line_items array, subtotal/tax/total).
- **Inputs:** From `Clean Text` plus LLM from `OpenRouter Chat Model`.
- **Outputs:** Under n8n’s extractor format; downstream code expects `output` in the JSON.
- **Edge cases / failures:**
  - If invoice date formats are ambiguous, output may be inconsistent.
  - Numeric parsing instruction helps, but models can still output strings → downstream `parseFloat()` partially mitigates.
  - Vendor name missing → later QuickBooks vendor lookup fails.

Sticky note context:
- **AI Extraction:** “Extracts structured data from invoice text using OpenRouter LLM…”

---

### Block 1.5 — Item Reconciliation & Auto-Creation (QuickBooks)
**Overview:** Converts extracted line items into candidate QuickBooks Items, compares to existing QuickBooks Items, creates missing ones, and builds a unified mapping used later for Bill line references.  
**Nodes involved:** `Prepare Items to Check`, `Get All QB Items`, `Check Which Items to Create`, `Need to Create Items?`, `Split Items to Create`, `Create Items`, `Merge Item Creation Paths`, `Collect All Item Mappings`

#### Node: Prepare Items to Check
- **Type / role:** `Code` — normalize extracted line items.
- **Configuration choices (interpreted):**
  - Reads `const extractedData = $input.first().json.output;`
  - Builds `itemsToCheck` from `extractedData.line_items` where `item_name` is non-empty.
  - For each item: `{ name, description, unitPrice, type: 'Service' }`
- **Outputs:** `{ extractedData, itemsToCheck }`
- **Edge cases:**
  - If `output` field name differs (extractor changes), this breaks.
  - If line_items missing, `itemsToCheck` empty → later logic may create a bill with no lines (or fail later).

#### Node: Get All QB Items
- **Type / role:** `QuickBooks` — fetches existing Items.
- **Configuration choices:**
  - Resource: `item`
  - Operation: `getAll`
  - Return all: `true`
- **Credential requirements:** QuickBooks OAuth2 credentials (company access).
- **Edge cases / failures:**
  - OAuth token expiration / refresh issues.
  - Large item catalogs may be slow; pagination handled by node, but could hit API limits.

#### Node: Check Which Items to Create
- **Type / role:** `Code` — determines missing items and builds mapping for existing ones.
- **Configuration choices (interpreted):**
  - Builds `existingItemMap` keyed by lowercase name → `{id, name}`
  - For each `itemsToCheck`:
    - If exists: `itemMapping[item.name] = existingId`
    - Else: push into `itemsToCreate` with default refs:
      - `incomeAccountRef: "79"`
      - `expenseAccountRef: "80"`
- **Outputs:**
  - `itemsToCreate` (array)
  - `existingItemMapping` (object)
  - `needsCreation` (boolean)
- **Edge cases:**
  - Name collisions by case-insensitive matching.
  - AccountRef IDs `79/80` must exist in the target QuickBooks company; otherwise item creation fails.

#### Node: Need to Create Items?
- **Type / role:** `IF` — branches based on whether missing items exist.
- **Condition:** `{{$json.needsCreation}} equals true`
- **Outputs:**
  - **True:** to `Split Items to Create`
  - **False:** directly to `Merge Item Creation Paths` (index 1)
- **Edge cases:** If `needsCreation` undefined, strict validation may treat it as not equal to true → skips creation.

#### Node: Split Items to Create
- **Type / role:** `Split Out` — emits one item per entry in `itemsToCreate`.
- **Field:** `itemsToCreate`
- **Output:** Each item becomes `{name, description, type, incomeAccountRef, expenseAccountRef}`.
- **Edge cases:** If `itemsToCreate` empty and branch still taken, output can be empty (but IF logic prevents that).

#### Node: Create Items
- **Type / role:** `HTTP Request` — creates QuickBooks Items via REST API.
- **Configuration choices (interpreted):**
  - POST `https://sandbox-quickbooks.api.intuit.com/v3/company/{companyId}/item?minorversion=75`
  - JSON body built from item fields:
    - `Name`, `Description`, `Type`
    - `IncomeAccountRef.value`
    - `ExpenseAccountRef.value`
  - Headers: `Content-Type: application/json`
  - Authentication: predefined credential type `quickBooksOAuth2Api`
- **Inputs:** One item per item to create.
- **Outputs:** QuickBooks API response (often wraps created object under `Item`).
- **Edge cases / failures:**
  - `{companyId}` must be replaced (or parameterized) in the URL; otherwise request fails.
  - Item name already exists → QuickBooks returns duplicate name error.
  - Wrong account refs → validation errors.
  - Sandbox endpoint requires sandbox realm/company.

#### Node: Merge Item Creation Paths
- **Type / role:** `Merge` — recombines the “items created” path with the “no creation needed” path.
- **Mode:** `combine` with `combineAll`
- **alwaysOutputData:** true (prevents empty output issues when one branch has no items)
- **Inputs:**
  - Input 1: from `Create Items`
  - Input 2: from `Need to Create Items?` (false branch)
- **Outputs:** Combined stream to proceed with mapping consolidation.
- **Edge cases:** Combine modes can produce unexpected structures; here the next code node pulls data directly from other nodes via selectors (`$('...')`), reducing dependency on merged payload shape.

#### Node: Collect All Item Mappings
- **Type / role:** `Code` — consolidates mappings from existing + created items and passes extractedData forward.
- **Configuration choices (interpreted):**
  - Pulls `existingItemMapping` from `Check Which Items to Create`.
  - Tries to read all outputs from `Create Items` and add `Name -> Id` to mapping.
  - Pulls `extractedData` from `Prepare Items to Check`.
- **Outputs:** `{ extractedData, allItemMapping }`
- **Edge cases / failures:**
  - If `Create Items` returns wrapper `Item`, code handles `item.Item || item`.
  - If QuickBooks response shape changes, mapping may be incomplete → later “Item not found” errors.

Sticky note context:
- **Item Management:** “Checks existing QuickBooks items… Creates missing items automatically with default accounts.”

---

### Block 1.6 — Vendor Reconciliation & Auto-Creation (QuickBooks)
**Overview:** Searches for the vendor by exact display name. If not found, creates a new vendor using extracted name/email.  
**Nodes involved:** `Find Vendor`, `Vendor Exists?`, `Create Vendor`

#### Node: Find Vendor
- **Type / role:** `QuickBooks` — query vendor list.
- **Configuration choices:**
  - Resource: `vendor`
  - Operation: `getAll`
  - Query filter: `WHERE DisplayName = '{{ $json.extractedData.vendor.name }}'`
- **Inputs:** From `Collect All Item Mappings`
- **Outputs:** Vendor records (if any).
- **Edge cases / failures:**
  - Exact match is strict; “ACME Inc.” vs “ACME, Inc” won’t match.
  - If multiple vendors share same DisplayName (shouldn’t, but possible), first item behavior matters.

#### Node: Vendor Exists?
- **Type / role:** `IF` — checks if a vendor record was returned.
- **Condition:** `{{$json.Id}} is empty`
  - Interpreted: if `Id` is empty ⇒ vendor not found ⇒ create vendor.
- **Outputs:**
  - **True (Id empty):** `Create Vendor`
  - **False:** `Build Bill Payload`
- **Edge cases:**
  - If `Find Vendor` returns no items, n8n IF may receive no input at all; behavior depends on node execution. Often it just won’t run. This workflow assumes a single item flow; consider enabling “Always Output Data” upstream or adding a fallback.
  - If `Id` exists but vendor data incomplete, bill creation still proceeds.

#### Node: Create Vendor
- **Type / role:** `QuickBooks` — creates a vendor.
- **Configuration choices:**
  - Resource: `vendor`, operation: `create`
  - `displayName` from extracted vendor name.
  - Additional fields:
    - `CompanyName` = vendor name
    - `PrimaryEmailAddr` = extracted vendor email
- **Inputs:** From `Vendor Exists?` true branch.
- **Outputs:** Created vendor with `Id`, `DisplayName`, etc.
- **Edge cases / failures:**
  - Missing vendor name → QuickBooks validation error.
  - Email format invalid → may be rejected depending on QuickBooks rules.

Sticky note context:
- **Vendor Management:** “Searches for vendor by name. Creates new vendor record if not found…”

---

### Block 1.7 — Bill Payload Construction & Bill Creation (QuickBooks)
**Overview:** Builds a QuickBooks Bill payload using extracted invoice details, resolved vendor id, and resolved item ids; then posts to the QuickBooks Bill endpoint.  
**Nodes involved:** `Build Bill Payload`, `Create Bill `

#### Node: Build Bill Payload
- **Type / role:** `Code` — constructs final Bill JSON.
- **Configuration choices (interpreted):**
  - Pulls `extractedData` and `allItemMapping` from `Collect All Item Mappings`.
  - Determines vendor:
    - Prefer `Create Vendor` output if available; otherwise uses `Find Vendor`.
    - Throws error if no vendorId.
  - Builds `Line` array from `extractedData.line_items`:
    - Looks up `itemId` by exact name; falls back to case-insensitive scan across mapping keys.
    - Throws error if item not found.
    - Sets:
      - `DetailType: "ItemBasedExpenseLineDetail"`
      - `Qty`, `UnitPrice`, `Amount`
      - `TaxCodeRef.value = "NON"`
  - Adds tax line if `extractedData.tax > 0` using `AccountBasedExpenseLineDetail` with `AccountRef.value = '1'`.
  - Normalizes dates to `YYYY-MM-DD` if needed using `new Date(...).toISOString().split('T')[0]`.
  - Sets:
    - `VendorRef`, `DocNumber`, `TxnDate`, `DueDate`
    - Optional `VendorAddr` parsed from vendor address lines.
- **Outputs:** `{ billPayload, summary }`
- **Edge cases / failures:**
  - Date parsing with `new Date(string)` is locale/format sensitive; “01/02/2026” ambiguity may produce wrong date.
  - If `amount` is missing or non-numeric, `parseFloat(item.amount)` yields `NaN` → QuickBooks API rejects payload.
  - Tax account ref `'1'` must exist and be appropriate; otherwise Bill creation fails.
  - If line item names in invoice don’t match created/known items (even after case-insensitive), workflow stops with “Item not found”.

#### Node: Create Bill 
- **Type / role:** `HTTP Request` — creates the Bill via QuickBooks REST API.
- **Configuration choices:**
  - POST `https://sandbox-quickbooks.api.intuit.com/v3/company/{companyId}/bill?minorversion=75`
  - Body: `JSON.stringify($json.billPayload)`
  - Header: `Content-Type: application/json`
  - Auth: `quickBooksOAuth2Api`
- **Connections:** After creation, connects back to `Loop Over Invoices` to continue batch iteration.
- **Edge cases / failures:**
  - `{companyId}` placeholder must be replaced.
  - Duplicate `DocNumber` may trigger QuickBooks duplicate document number restrictions.
  - If VendorRef/ItemRef invalid, QuickBooks returns 400 with validation details.

Sticky note context:
- **Bill Creation:** “Builds complete bill payload… Posts to QuickBooks API.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On bill submission | Form Trigger | Collect invoice PDF upload via web form | — | Convert to Separate Items | Workflow Overview: setup steps + flow summary |
| Convert to Separate Items | Code | Split multiple binary uploads into separate items | On bill submission | Loop Over Invoices | PDF Processing: Converts uploaded PDFs to separate items and loops through each invoice for extraction. |
| Loop Over Invoices | Split In Batches | Iterate through invoices one-by-one | Convert to Separate Items; Create Bill  | Extract from PDF | PDF Processing: Converts uploaded PDFs to separate items and loops through each invoice for extraction. |
| Extract from PDF | Extract From File | Extract text from PDF | Loop Over Invoices | Clean Text | PDF Processing: Converts uploaded PDFs to separate items and loops through each invoice for extraction. |
| Clean Text | Code | Normalize extracted text for LLM | Extract from PDF | Extract Invoice Data | PDF Processing: Converts uploaded PDFs to separate items and loops through each invoice for extraction. |
| OpenRouter Chat Model | LangChain LLM (OpenRouter) | Provides LLM for extraction | — | Extract Invoice Data (ai_languageModel) | AI Extraction: Extracts structured data from invoice text using OpenRouter LLM (vendor, line items, dates, amounts). |
| Extract Invoice Data | Information Extractor | Schema-based extraction to structured JSON | Clean Text; OpenRouter Chat Model | Prepare Items to Check | AI Extraction: Extracts structured data from invoice text using OpenRouter LLM (vendor, line items, dates, amounts). |
| Prepare Items to Check | Code | Build normalized list of line items to reconcile | Extract Invoice Data | Get All QB Items | Item Management: Checks existing QuickBooks items against invoice line items. Creates missing items automatically with default accounts. |
| Get All QB Items | QuickBooks | Fetch all existing QB items | Prepare Items to Check | Check Which Items to Create | Item Management: Checks existing QuickBooks items against invoice line items. Creates missing items automatically with default accounts. |
| Check Which Items to Create | Code | Decide which items exist vs must be created | Get All QB Items | Need to Create Items? | Item Management: Checks existing QuickBooks items against invoice line items. Creates missing items automatically with default accounts. |
| Need to Create Items? | IF | Branch: create missing items or skip | Check Which Items to Create | Split Items to Create; Merge Item Creation Paths | Item Management: Checks existing QuickBooks items against invoice line items. Creates missing items automatically with default accounts. |
| Split Items to Create | Split Out | Emit one item per missing item | Need to Create Items? (true) | Create Items | Item Management: Checks existing QuickBooks items against invoice line items. Creates missing items automatically with default accounts. |
| Create Items | HTTP Request | Create QuickBooks Items via REST | Split Items to Create | Merge Item Creation Paths | Item Management: Checks existing QuickBooks items against invoice line items. Creates missing items automatically with default accounts. |
| Merge Item Creation Paths | Merge | Rejoin “created items” and “no creation” paths | Create Items; Need to Create Items? (false) | Collect All Item Mappings | Item Management: Checks existing QuickBooks items against invoice line items. Creates missing items automatically with default accounts. |
| Collect All Item Mappings | Code | Consolidate item name→id mapping | Merge Item Creation Paths | Find Vendor | Item Management: Checks existing QuickBooks items against invoice line items. Creates missing items automatically with default accounts. |
| Find Vendor | QuickBooks | Search vendor by DisplayName | Collect All Item Mappings | Vendor Exists? | Vendor Management: Searches for vendor by name. Creates new vendor record if not found, using extracted contact info. |
| Vendor Exists? | IF | Branch: vendor missing → create | Find Vendor | Create Vendor; Build Bill Payload | Vendor Management: Searches for vendor by name. Creates new vendor record if not found, using extracted contact info. |
| Create Vendor | QuickBooks | Create vendor in QuickBooks | Vendor Exists? (true) | Build Bill Payload | Vendor Management: Searches for vendor by name. Creates new vendor record if not found, using extracted contact info. |
| Build Bill Payload | Code | Build QuickBooks Bill JSON payload | Vendor Exists? (false) OR Create Vendor | Create Bill  | Bill Creation: Builds complete bill payload with all line items, vendor ref, dates, and tax. Posts to QuickBooks API. |
| Create Bill  | HTTP Request | Create QuickBooks Bill via REST | Build Bill Payload | Loop Over Invoices | Bill Creation: Builds complete bill payload with all line items, vendor ref, dates, and tax. Posts to QuickBooks API. |
| Workflow Overview | Sticky Note | Documentation | — | — |  |
| PDF Processing | Sticky Note | Documentation | — | — |  |
| AI Extraction | Sticky Note | Documentation | — | — |  |
| Item Management | Sticky Note | Documentation | — | — |  |
| Vendor Management | Sticky Note | Documentation | — | — |  |
| Bill Creation | Sticky Note | Documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Form Trigger**
   - Add node: **Form Trigger**
   - Title: `Invoice Parser + QuickBooks Bill Creator`
   - Add field:
     - Type: **File**
     - Label: `Invoice File`
     - Required: true
     - Accept: `.pdf`

2) **Split binary uploads into separate items**
   - Add node: **Code** named `Convert to Separate Items`
   - Paste logic that iterates `item.binary` keys and outputs one item per binary, placing that binary under `binary.data`.

3) **Iterate invoices**
   - Add node: **Split In Batches** named `Loop Over Invoices`
   - Leave defaults (or set batch size as desired).
   - Connect: `On bill submission` → `Convert to Separate Items` → `Loop Over Invoices`.

4) **Extract text from PDF**
   - Add node: **Extract From File**
   - Operation: **PDF**
   - Connect from `Loop Over Invoices` (the output used for per-item processing) → `Extract from PDF`.

5) **Clean extracted text**
   - Add node: **Code** named `Clean Text`
   - Implement newline normalization and trimming; output `{ cleaned_text }`.
   - Connect: `Extract from PDF` → `Clean Text`.

6) **Add OpenRouter LLM**
   - Add node: **OpenRouter Chat Model** (LangChain)
   - Set model: `openai/gpt-oss-20b:free` (or your preferred model)
   - Configure **OpenRouter credentials** (API key).

7) **Extract structured invoice JSON**
   - Add node: **Information Extractor**
   - Text: `{{$json.cleaned_text}}`
   - Provide a **system prompt** that requests vendor + line items + totals and numeric outputs.
   - Configure schema mode “from JSON” and paste the schema example used (invoice_number, invoice_date, due_date, vendor object, line_items fields, subtotal/tax/total).
   - Connect:
     - `Clean Text` → `Extract Invoice Data`
     - `OpenRouter Chat Model` → `Extract Invoice Data` via the AI language model connection.

8) **Normalize items for reconciliation**
   - Add node: **Code** named `Prepare Items to Check`
   - Read extracted object from `json.output` and create `itemsToCheck` from `line_items` (filter empty names).
   - Connect: `Extract Invoice Data` → `Prepare Items to Check`.

9) **Fetch all QuickBooks Items**
   - Add node: **QuickBooks**
   - Resource: **Item**
   - Operation: **Get All**
   - Return all: true
   - Configure **QuickBooks OAuth2 credentials**.
   - Connect: `Prepare Items to Check` → `Get All QB Items`.

10) **Compute missing items + mapping**
   - Add node: **Code** named `Check Which Items to Create`
   - Build a case-insensitive map of existing items (by Name).
   - Produce:
     - `itemsToCreate` with default `incomeAccountRef` and `expenseAccountRef` (set these to valid account IDs in your QuickBooks company; the workflow uses `79` and `80`)
     - `existingItemMapping`
     - `needsCreation`
   - Connect: `Get All QB Items` → `Check Which Items to Create`.

11) **Branch on whether to create items**
   - Add node: **IF** named `Need to Create Items?`
   - Condition: boolean `{{$json.needsCreation}}` equals `true`
   - Connect: `Check Which Items to Create` → `Need to Create Items?`

12) **Split out items to create**
   - Add node: **Split Out** named `Split Items to Create`
   - Field to split out: `itemsToCreate`
   - Connect: `Need to Create Items?` (true) → `Split Items to Create`.

13) **Create missing items in QuickBooks (REST)**
   - Add node: **HTTP Request** named `Create Items`
   - Method: POST
   - URL: `https://sandbox-quickbooks.api.intuit.com/v3/company/{companyId}/item?minorversion=75`
     - Replace `{companyId}` with your realm/company id (or use an n8n expression/variable).
   - Auth: **QuickBooks OAuth2** credential type
   - JSON body (fields from split items): Name, Description, Type, IncomeAccountRef.value, ExpenseAccountRef.value
   - Header: `Content-Type: application/json`
   - Connect: `Split Items to Create` → `Create Items`.

14) **Merge “created” vs “none created” paths**
   - Add node: **Merge** named `Merge Item Creation Paths`
   - Mode: **Combine**, “combine all”
   - Enable **Always Output Data**
   - Connect:
     - `Create Items` → `Merge Item Creation Paths` input 1
     - `Need to Create Items?` (false) → `Merge Item Creation Paths` input 2

15) **Consolidate item mappings**
   - Add node: **Code** named `Collect All Item Mappings`
   - Combine:
     - `existingItemMapping` from `Check Which Items to Create`
     - Newly created items from `Create Items` outputs (handle wrapper `Item`)
   - Output: `{ extractedData, allItemMapping }`
   - Connect: `Merge Item Creation Paths` → `Collect All Item Mappings`.

16) **Find vendor in QuickBooks**
   - Add node: **QuickBooks** named `Find Vendor`
   - Resource: **Vendor**
   - Operation: **Get All**
   - Query: `WHERE DisplayName = '{{ $json.extractedData.vendor.name }}'`
   - Connect: `Collect All Item Mappings` → `Find Vendor`.

17) **Branch: vendor exists?**
   - Add node: **IF** named `Vendor Exists?`
   - Condition: `{{$json.Id}} is empty`
   - Connect: `Find Vendor` → `Vendor Exists?`

18) **Create vendor if missing**
   - Add node: **QuickBooks** named `Create Vendor`
   - Resource: Vendor; Operation: Create
   - DisplayName: extracted vendor name
   - Additional fields: CompanyName and PrimaryEmailAddr from extracted vendor info
   - Connect: `Vendor Exists?` (true) → `Create Vendor`

19) **Build QuickBooks Bill payload**
   - Add node: **Code** named `Build Bill Payload`
   - Logic:
     - Determine vendorId from `Create Vendor` if executed, else from `Find Vendor`
     - For each extracted `line_items`, find item id in `allItemMapping` (case-insensitive fallback)
     - Build `Line` entries with `ItemBasedExpenseLineDetail`
     - Optionally add tax line with `AccountBasedExpenseLineDetail` (ensure account id is correct)
     - Normalize invoice_date/due_date to `YYYY-MM-DD`
     - Output `{ billPayload, summary }`
   - Connect:
     - `Vendor Exists?` (false) → `Build Bill Payload`
     - `Create Vendor` → `Build Bill Payload`

20) **Create Bill in QuickBooks (REST)**
   - Add node: **HTTP Request** named `Create Bill`
   - Method: POST
   - URL: `https://sandbox-quickbooks.api.intuit.com/v3/company/{companyId}/bill?minorversion=75`
   - Auth: QuickBooks OAuth2 credential
   - Body: JSON = `{{$json.billPayload}}` (or stringify as in the workflow)
   - Header: `Content-Type: application/json`
   - Connect: `Build Bill Payload` → `Create Bill`

21) **Continue the invoice loop**
   - Connect: `Create Bill` → `Loop Over Invoices` (to trigger next batch iteration)

22) **Credentials & constants to set**
   - **QuickBooks OAuth2 credential**: ensure it’s authorized for the intended QuickBooks company (sandbox vs prod).
   - Replace `{companyId}` in both QuickBooks REST URLs.
   - Validate account refs:
     - Item creation uses IncomeAccountRef `79` and ExpenseAccountRef `80`.
     - Tax line uses AccountRef `1`.
   - **OpenRouter credential**: API key and allowed model.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Connect QuickBooks OAuth2 credentials” | From sticky note “Workflow Overview” |
| “Add OpenRouter API key for AI extraction” | From sticky note “Workflow Overview” |
| “Configure income/expense account refs (currently 79/80)” | From sticky note “Workflow Overview” |
| “Test with sample invoice PDF” | From sticky note “Workflow Overview” |
| “Access form URL to submit invoices” | From sticky note “Workflow Overview” |