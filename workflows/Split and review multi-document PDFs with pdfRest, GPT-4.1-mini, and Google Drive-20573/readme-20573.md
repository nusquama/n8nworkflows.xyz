Split and review multi-document PDFs with pdfRest, GPT-4.1-mini, and Google Drive

https://n8nworkflows.xyz/workflows/split-and-review-multi-document-pdfs-with-pdfrest--gpt-4-1-mini--and-google-drive-20573


# Split and review multi-document PDFs with pdfRest, GPT-4.1-mini, and Google Drive

### 1. Workflow Overview

This workflow automates the ingestion, AI-powered segmentation, human validation, and reconstruction of multi-document PDF batches. Its primary purpose is to take a single large PDF file containing multiple distinct business documents (such as invoices, contracts, and receipts), intelligently suggest document boundaries, allow a human reviewer to inspect and correct those boundaries via interactive forms, split the source PDF into individual clean files, archive them optionally to Google Drive, and bundle them into a downloadable ZIP archive complete with a structured page manifest.

The logical execution flows through seven distinct functional blocks:

- **1.1 Input Reception & Configuration:** Ingests the raw PDF via an interactive form, loads operational parameters, and validates the file signature and page budget constraints.
- **1.2 PDF Preprocessing & Text Extraction:** Performs Optical Character Recognition (OCR) via pdfRest to create a searchable master copy, splits it into individual page files, converts each page into structured Markdown, and compiles a comprehensive page inventory.
- **1.3 AI Boundary Proposal:** Uses an OpenAI language model via the LangChain Information Extractor node to analyze the compiled text and propose contiguous document groupings with metadata and reasoning.
- **1.4 Human Review & Interactive Validation (Loop 1):** Renders an interactive n8n form displaying the AI-suggested page ranges, titles, and categories, then validates user adjustments for logical integrity (no overlapping or unassigned pages).
- **1.5 Iterative Correction Sub-Workflow (Loops 2 & 3):** Handles sequential error-correction cycles if the reviewer introduces invalid page ranges, enforcing a strict limit of three review attempts before terminating.
- **1.6 Document Splitting & Archival:** Splits the verified master PDF into individual document files based on approved page ranges, normalizes filenames, and optionally uploads copies to a designated Google Drive folder.
- **1.7 Bundling & Final Delivery:** Generates a machine-readable JSON page manifest, compresses all split PDFs and the manifest into a single ZIP archive, and delivers it via a final completion form download.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration

- **Overview:** Initializes the process by capturing a batch PDF upload from a user via an n8n form trigger, establishing global processing settings, and performing initial security and structural validations.
- **Nodes Involved:**
  - `Upload mixed batch`
  - `Settings`
  - `Validate batch`
  - `Inspect batch`
  - `Check page budget`

- **Node Details:**
  - **Upload mixed batch**
    - *Type & Technical Role:* `n8n-nodes-base.formTrigger` (v2.3) — Acts as the entry point, rendering an interactive upload form for a single PDF file.
    - *Configuration:* Configured to accept a single file field (`Batch PDF`) restricted to `.pdf` extensions.
    - *Expressions:* Uses standard webhook configuration.
    - *Connections:* Input: None (Trigger); Output: `Settings`.
    - *Edge Cases/Failures:* Submitting non-PDF files or multiple files is blocked upstream, but missing uploads throw form-level validation errors.
  - **Settings**
    - *Type & Technical Role:* `n8n-nodes-base.set` (v3.4) — Defines global execution parameters.
    - *Configuration:* Sets `maxPages` (60), `maxTextCharacters` (100000), `ocrLanguages` ("English"), `saveToDrive` (false), and `outputFolderId` ("PASTE_DRIVE_FOLDER_ID").
    - *Expressions:* None (static assignments).
    - *Connections:* Input: `Upload mixed batch`; Output: `Validate batch`.
  - **Validate batch**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript execution node ensuring data integrity.
    - *Configuration:* Validates that exactly one binary PDF file is present, checks the file header for the `%PDF-` magic string, and verifies that `outputFolderId` has been updated if `saveToDrive` is enabled.
    - *Expressions:* References `$('Settings')` and `$execution.id`.
    - *Connections:* Input: `Settings`; Output: `Inspect batch`.
    - *Edge Cases/Failures:* Throws explicit errors if zero/multiple files are uploaded, if the file is not a valid PDF, or if Drive sync is enabled without a valid folder ID.
  - **Inspect batch**
    - *Type & Technical Role:* `@pdfrest/n8n-nodes-pdfrest.pdfRest` (v1) — Calls the pdfRest API to inspect properties.
    - *Configuration:* Operation set to `pdfInfo`, querying `page_count` and `image_only`. Configured with `maxTries: 3` and `retryOnFail: true`.
    - *Connections:* Input: `Validate batch`; Output: `Check page budget`.
    - *Edge Cases/Failures:* API timeouts or invalid API keys result in retry failures after 3 attempts.
  - **Check page budget**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Validates numerical page limits.
    - *Configuration:* Confirms `page_count` is a positive integer and does not exceed `maxPages` defined in the Settings node.
    - *Expressions:* References `$('Settings')` and `$('Validate batch')`.
    - *Connections:* Input: `Inspect batch`; Output: `OCR mixed batch`.
    - *Edge Cases/Failures:* Throws an error if the page count exceeds the configured budget.

---

#### 2.2 PDF Preprocessing & Text Extraction

- **Overview:** Converts the uploaded raw PDF into a searchable master document via OCR, splits it into individual single-page PDFs, extracts Markdown text from each page, and compiles a unified inventory.
- **Nodes Involved:**
  - `OCR mixed batch`
  - `Keep searchable master`
  - `Split into individual pages`
  - `Number every source page`
  - `Read each page`
  - `Build complete page inventory`

- **Node Details:**
  - **OCR mixed batch**
    - *Type & Technical Role:* `@pdfrest/n8n-nodes-pdfrest.pdfRest` (v1) — Applies OCR to the master PDF.
    - *Configuration:* Operation set to `ocr`, downloading output files into a `processed` binary field.
    - *Expressions:* Uses dynamic languages expression: `={{ $('Settings').first().json.ocrLanguages.split(',').map(s => s.trim()) }}`.
    - *Connections:* Input: `Check page budget`; Output: `Keep searchable master`.
  - **Keep searchable master**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Isolates the OCR-processed binary data.
    - *Configuration:* Verifies the presence of `processed` binary data and passes it forward as the searchable master copy.
    - *Connections:* Input: `OCR mixed batch`; Output: `Split into individual pages`.
  - **Split into individual pages**
    - *Type & Technical Role:* `@pdfrest/n8n-nodes-pdfrest.pdfRest` (v1) — Splits the master PDF into individual page files.
    - *Configuration:* Operation set to `split`.
    - *Connections:* Input: `Keep searchable master`; Output: `Number every source page`.
  - **Number every source page**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Re-indexes split page binary keys.
    - *Configuration:* Sorts binary keys (`processed`, `processed_1`, etc.) matching the total page count, mapping each to an explicit sequential item.
    - *Connections:* Input: `Split into individual pages`; Output: `Read each page`.
  - **Read each page**
    - *Type & Technical Role:* `@pdfrest/n8n-nodes-pdfrest.pdfRest` (v1) — Converts individual pages to text format.
    - *Configuration:* Operation set to `convertMarkdown` with output type set to `json` and `pageBreakComments` enabled.
    - *Connections:* Input: `Number every source page`; Output: `Build complete page inventory`.
  - **Build complete page inventory**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Aggregates individual page texts.
    - *Configuration:* Combines extracted Markdown pages into a single consolidated string, flags blank pages (text length < 20 characters), and validates against `maxTextCharacters`.
    - *Connections:* Input: `Read each page`; Output: `Propose document boundaries`.

---

#### 2.3 AI Boundary Proposal

- **Overview:** Sends the consolidated page text to an OpenAI large language model structured via a LangChain Information Extractor node to suggest logical document boundaries, categories, titles, and counterparties.
- **Nodes Involved:**
  - `Propose document boundaries`
  - `AI model - connect credential`

- **Node Details:**
  - **Propose document boundaries**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.informationExtractor` (v1) — Extracts structured JSON data from unstructured text using an AI model.
    - *Configuration:* Uses a manual schema defining an array of `documents` with properties: `startPage`, `endPage`, `documentType` (enum: invoice, contract, correspondence, receipt, other), `title`, `counterparty`, and `reason`. System prompt instructs the model never to drop blank pages and to ensure ascending contiguous ranges.
    - *Expressions:* Input text mapped via `={{ $json.text }}`.
    - *Connections:* Input: `Build complete page inventory` (main), `AI model - connect credential` (AI model); Output: `Prepare boundary review`.
  - **AI model - connect credential**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) — Provides OpenAI LLM integration.
    - *Configuration:* Model set to `gpt-4.1-mini`, with temperature set to `0`. Requires OpenAI API credentials.
    - *Connections:* Output: Connected to `Propose document boundaries` via `ai_languageModel`.

---

#### 2.4 Human Review & Interactive Validation (Loop 1)

- **Overview:** Prepares HTML descriptions and form field structures based on AI proposals, renders an interactive review form to the user, and validates the submitted document groupings.
- **Nodes Involved:**
  - `Prepare boundary review`
  - `Review and correct boundaries`
  - `Check page groups 1`
  - `Cancelled 1?`
  - `Page groups valid 1?`

- **Node Details:**
  - **Prepare boundary review**
    - *Type & Technical Role:* `n8n-nodes-base.set` / `n8n-nodes-base.code` (v2) — Formats AI output into dynamic n8n form fields.
    - *Configuration:* Generates dynamic HTML review descriptions and form fields for each proposed document slot, plus two extra optional slots for manual splitting.
    - *Connections:* Input: `Propose document boundaries`; Output: `Review and correct boundaries`.
  - **Review and correct boundaries**
    - *Type & Technical Role:* `n8n-nodes-base.form` (v2.3) — Presents the interactive review form to the user.
    - *Configuration:* Operation set to `page`, rendering JSON-defined fields (`Include/Exclude`, `Title`, `Category`, `Start Page`, `End Page`, and global `Decision` dropdown: Approve/Cancel).
    - *Connections:* Input: `Prepare boundary review`; Output: `Check page groups 1`.
  - **Check page groups 1**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Validates user review submissions.
    - *Configuration:* Checks if `Decision === 'Cancel'`, validates that start/end page numbers are integers within range, ensures no pages overlap or remain unassigned, and verifies at least one document is included.
    - *Connections:* Input: `Review and correct boundaries`; Output: `Cancelled 1?`.
  - **Cancelled 1?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Branching logic for user cancellation.
    - *Configuration:* Checks if `$json.cancelled` is true.
    - *Connections:* Input: `Check page groups 1`; Output: True -> `Cancelled`, False -> `Page groups valid 1?`.
  - **Page groups valid 1?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Branching logic for validation status.
    - *Configuration:* Checks if `$json.valid` is true.
    - *Connections:* Input: `Cancelled 1?` (false branch); Output: True -> `Use approved page groups`, False -> `Correct page groups 2`.

---

#### 2.5 Iterative Correction Sub-Workflow (Loops 2 & 3)

- **Overview:** Provides up to two additional review and correction cycles if the initial user submission contains validation errors (overlapping, out-of-range, or missing pages).
- **Nodes Involved:**
  - `Correct page groups 2`
  - `Check page groups 2`
  - `Cancelled 2?`
  - `Page groups valid 2?`
  - `Correct page groups 3`
  - `Check page groups 3`
  - `Cancelled 3?`
  - `Page groups valid 3?`
  - `Explain unresolved page groups`
  - `Cancelled`

- **Node Details:**
  - **Correct page groups 2 / 3**
    - *Type & Technical Role:* `n8n-nodes-base.form` (v2.3) — Secondary/tertiary correction forms.
    - *Configuration:* Renders form fields pre-populated with previous user edits alongside explicit error lists explaining validation failures.
    - *Connections:* Input: Previous validation failure; Output: Corresponding `Check page groups` code node.
  - **Check page groups 2 / 3**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Re-validates page numbers and assignments for attempt 2 and 3.
    - *Connections:* Input: Correction form; Output: Corresponding `Cancelled?` conditional check.
  - **Cancelled 2? / 3?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Evaluates cancellation requests during correction loops.
  - **Page groups valid 2? / 3?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Evaluates whether re-submitted ranges pass validation.
    - *Connections:* Output (True) -> `Use approved page groups`; Output (False) -> Next correction form (or `Explain unresolved page groups` on attempt 3).
  - **Explain unresolved page groups**
    - *Type & Technical Role:* `n8n-nodes-base.form` (v2.3) — Terminal completion form.
    - *Configuration:* Operation set to `completion` (respond with text), displaying final error summaries when all 3 review attempts are exhausted.
  - **Cancelled**
    - *Type & Technical Role:* `n8n-nodes-base.form` (v2.3) — Terminal cancellation form.
    - *Configuration:* Operation set to `completion`, informing the user that review was cancelled and no documents were created.

---

#### 2.6 Document Splitting & Archival

- **Overview:** Takes validated page groups, splits the searchable master PDF into individual files using pdfRest, sanitizes filenames, and conditionally uploads them to Google Drive.
- **Nodes Involved:**
  - `Use approved page groups`
  - `Split approved documents`
  - `Name each reviewed document`
  - `Save copies to Drive?`
  - `File reviewed documents`
  - `Restore filed document binaries`

- **Node Details:**
  - **Use approved page groups**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Prepares approved payload for splitting.
    - *Configuration:* Formats page ranges (`startPage-endPage`) for the pdfRest split operation.
    - *Connections:* Input: Validated boundary output; Output: `Split approved documents`.
  - **Split approved documents**
    - *Type & Technical Role:* `@pdfrest/n8n-nodes-pdfrest.pdfRest` (v1) — Splits the master PDF into individual document files based on approved ranges.
    - *Configuration:* Operation set to `split`, with `pageRanges` mapped dynamically from `$json.ranges`.
    - *Connections:* Input: `Use approved page groups`; Output: `Name each reviewed document`.
  - **Name each reviewed document**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Filename normalization and metadata binding.
    - *Configuration:* Sanitizes document titles, appending batch ID, index, category, and sanitized title into a structured `.pdf` filename.
    - *Connections:* Input: `Split approved documents`; Output: `Save copies to Drive?`.
  - **Save copies to Drive?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Conditional router for Google Drive archival.
    - *Configuration:* Evaluates `saveToDrive` setting (`true`/`false`).
    - *Connections:* Input: `Name each reviewed document`; Output: True -> `File reviewed documents`, False -> `Build page manifest and download`.
  - **File reviewed documents**
    - *Type & Technical Role:* `n8n-nodes-base.googleDrive` (v3) — Uploads split PDFs to Google Drive.
    - *Configuration:* Operation set to `upload`, mapping file names and targeting `outputFolderId`. Requires Google Drive OAuth2 credentials.
    - *Connections:* Input: `Save copies to Drive?`; Output: `Restore filed document binaries`.
  - **Restore filed document binaries**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Restores binary data streams after Drive upload.
    - *Configuration:* Reattaches binary data and embeds Google Drive file IDs and web view links into the item JSON.
    - *Connections:* Input: `File reviewed documents`; Output: `Build page manifest and download`.

---

#### 2.7 Bundling & Final Delivery

- **Overview:** Compiles a page-to-document manifest JSON, compresses all split PDFs and the manifest into a ZIP bundle, and presents a download completion form to the user.
- **Nodes Involved:**
  - `Build page manifest and download`
  - `Create download ZIP`
  - `Download reviewed documents`

- **Node Details:**
  - **Build page manifest and download**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Manifest generation.
    - *Configuration:* Builds a JSON manifest (`page-manifest.json`) mapping every source page to its corresponding output document and page number, consolidating binary attachments.
    - *Connections:* Input: `Restore filed document binaries` (or `Save copies to Drive?` false branch); Output: `Create download ZIP`.
  - **Create download ZIP**
    - *Type & Technical Role:* `n8n-nodes-base.compression` (v1.1) — ZIP archive creation.
    - *Configuration:* Operation set to `compress`, bundling all binary inputs into a single `.zip` file named after the batch ID.
    - *Connections:* Input: `Build page manifest and download`; Output: `Download reviewed documents`.
  - **Download reviewed documents**
    - *Type & Technical Role:* `n8n-nodes-base.form` (v2.3) — Completion trigger delivering the download.
    - *Configuration:* Operation set to `completion`, configured with `respondWith: "returnBinary"` and `inputDataFieldName: "bundle"` to serve the ZIP file directly in the browser.
    - *Connections:* Input: `Create download ZIP`; Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Upload mixed batch | n8n-nodes-base.formTrigger | Ingests raw batch PDF via upload form | None | Settings | Setup instructions |
| Settings | n8n-nodes-base.set | Sets global workflow variables and limits | Upload mixed batch | Validate batch | Setup instructions |
| Validate batch | n8n-nodes-base.code | Validates file count, PDF magic bytes, and Drive setup | Settings | Inspect batch | Setup instructions |
| Inspect batch | @pdfrest/n8n-nodes-pdfrest.pdfRest | Retrieves page count and metadata via pdfRest | Validate batch | Check page budget | Setup instructions |
| Check page budget | n8n-nodes-base.code | Enforces maxPages ceiling | Inspect batch | OCR mixed batch | Setup instructions |
| OCR mixed batch | @pdfrest/n8n-nodes-pdfrest.pdfRest | Applies OCR to create searchable master PDF | Check page budget | Keep searchable master | Setup instructions |
| Keep searchable master | n8n-nodes-base.code | Isolates OCR output binary | OCR mixed batch | Split into individual pages | Setup instructions |
| Split into individual pages | @pdfrest/n8n-nodes-pdfrest.pdfRest | Splits master PDF into single-page files | Keep searchable master | Number every source page | How it works - preserve every page |
| Number every source page | n8n-nodes-base.code | Re-indexes split page binary keys sequentially | Split into individual pages | Read each page | How it works - preserve every page |
| Read each page | @pdfrest/n8n-nodes-pdfrest.pdfRest | Converts individual pages to Markdown JSON | Number every source page | Build complete page inventory | How it works - preserve every page |
| Build complete page inventory | n8n-nodes-base.code | Aggregates page texts and flags blank pages | Read each page | Propose document boundaries | How it works - preserve every page |
| Propose document boundaries | @n8n/n8n-nodes-langchain.informationExtractor | AI extraction of document segments | Build complete page inventory, AI model - connect credential | Prepare boundary review | How it works - preserve every page |
| AI model - connect credential | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides OpenAI LLM (gpt-4.1-mini) for extraction | None | Propose document boundaries | How it works - preserve every page |
| Prepare boundary review | n8n-nodes-base.code | Formats AI output into HTML/form fields | Propose document boundaries | Review and correct boundaries | How it works - preserve every page |
| Review and correct boundaries | n8n-nodes-base.form | Renders interactive boundary review form | Prepare boundary review | Check page groups 1 | How it works - preserve every page |
| Check page groups 1 | n8n-nodes-base.code | Validates initial user review submission | Review and correct boundaries | Cancelled 1? | How it works - preserve every page |
| Cancelled 1? | n8n-nodes-base.if | Checks if user cancelled review 1 | Check page groups 1 | Cancelled, Page groups valid 1? | Important notes |
| Page groups valid 1? | n8n-nodes-base.if | Routes valid vs invalid review 1 submissions | Cancelled 1? | Use approved page groups, Correct page groups 2 | Important notes |
| Correct page groups 2 | n8n-nodes-base.form | Renders attempt 2 correction form | Page groups valid 1? | Check page groups 2 | Important notes |
| Check page groups 2 | n8n-nodes-base.code | Validates review attempt 2 submission | Correct page groups 2 | Cancelled 2? | Important notes |
| Cancelled 2? | n8n-nodes-base.if | Checks if user cancelled review 2 | Check page groups 2 | Cancelled, Page groups valid 2? | Important notes |
| Page groups valid 2? | n8n-nodes-base.if | Routes valid vs invalid review 2 submissions | Cancelled 2? | Use approved page groups, Correct page groups 3 | Important notes |
| Correct page groups 3 | n8n-nodes-base.form | Renders attempt 3 correction form | Page groups valid 2? | Check page groups 3 | Important notes |
| Check page groups 3 | n8n-nodes-base.code | Validates review attempt 3 submission | Correct page groups 3 | Cancelled 3? | Important notes |
| Cancelled 3? | n8n-nodes-base.if | Checks if user cancelled review 3 | Check page groups 3 | Cancelled, Page groups valid 3? | Important notes |
| Page groups valid 3? | n8n-nodes-base.if | Routes final review attempt validation | Cancelled 3? | Use approved page groups, Explain unresolved page groups | Important notes |
| Explain unresolved page groups | n8n-nodes-base.form | Terminal completion form showing unresolvable errors | Page groups valid 3? | None | Important notes |
| Cancelled | n8n-nodes-base.form | Terminal completion form for cancelled reviews | Cancelled 1?, Cancelled 2?, Cancelled 3? | None | Important notes |
| Use approved page groups | n8n-nodes-base.code | Formats approved ranges for final splitting | Page groups valid 1?, Page groups valid 2?, Page groups valid 3? | Split approved documents | Important notes |
| Split approved documents | @pdfrest/n8n-nodes-pdfrest.pdfRest | Splits master PDF into approved documents | Use approved page groups | Name each reviewed document | Important notes |
| Name each reviewed document | n8n-nodes-base.code | Sanitizes filenames and binds document metadata | Split approved documents | Save copies to Drive? | Important notes |
| Save copies to Drive? | n8n-nodes-base.if | Determines whether to upload copies to Google Drive | Name each reviewed document | File reviewed documents, Build page manifest and download | Important notes |
| File reviewed documents | n8n-nodes-base.googleDrive | Uploads split documents to Google Drive folder | Save copies to Drive? | Restore filed document binaries | Important notes |
| Restore filed document binaries | n8n-nodes-base.code | Restores binary data and embeds Drive IDs/links | File reviewed documents | Build page manifest and download | Important notes |
| Build page manifest and download | n8n-nodes-base.code | Compiles JSON manifest and bundles binaries | Restore filed document binaries, Save copies to Drive? | Create download ZIP | Important notes |
| Create download ZIP | n8n-nodes-base.compression | Compresses split PDFs and manifest into ZIP | Build page manifest and download | Download reviewed documents | Important notes |
| Download reviewed documents | n8n-nodes-base.form | Terminal form serving the downloadable ZIP | Create download ZIP | None | Important notes |
| Setup instructions | n8n-nodes-base.stickyNote | Setup documentation and configuration guide | None | None | Setup instructions |
| How it works - preserve every page | n8n-nodes-base.stickyNote | Architectural overview of OCR, AI, and validation | None | None | How it works - preserve every page |
| Important notes | n8n-nodes-base.stickyNote | Operational guidelines, limits, and edge cases | None | None | Important notes |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n without importing the JSON, follow these sequential steps:

1. **Create the Entry Point:**
   - Add a **Form Trigger** node named `Upload mixed batch`. Configure a file upload field named `Batch PDF`, set type to file, required, accepting `.pdf`.
2. **Add Configuration & Validation:**
   - Add a **Set (Edit Fields)** node named `Settings`. Add number fields `maxPages` (60) and `maxTextCharacters` (100000), string field `ocrLanguages` ("English"), boolean field `saveToDrive` (false), and string field `outputFolderId` ("PASTE_DRIVE_FOLDER_ID").
   - Add a **Code** node named `Validate batch` to check file counts, magic bytes (`%PDF-`), and Drive configuration. Connect `Settings` -> `Validate batch`.
3. **Incorporate pdfRest Inspection & OCR:**
   - Add a **pdfRest** node named `Inspect batch`. Set operation to `pdfInfo` (queries: `page_count`, `image_only`). Connect `Validate batch` -> `Inspect batch`.
   - Add a **Code** node named `Check page budget` to verify page counts against `maxPages`. Connect `Inspect batch` -> `Check page budget`.
   - Add a **pdfRest** node named `OCR mixed batch`. Set operation to `ocr`, enabling output file downloads. Connect `Check page budget` -> `OCR mixed batch`.
   - Add a **Code** node named `Keep searchable master` to extract the OCR binary. Connect `OCR mixed batch` -> `Keep searchable master`.
4. **Page Splitting and Text Conversion:**
   - Add a **pdfRest** node named `Split into individual pages`. Set operation to `split`. Connect `Keep searchable master` -> `Split into individual pages`.
   - Add a **Code** node named `Number every source page` to index split pages. Connect `Split into individual pages` -> `Number every source page`.
   - Add a **pdfRest** node named `Read each page`. Set operation to `convertMarkdown` (outputType: JSON, pageBreakComments: on). Connect `Number every source page` -> `Read each page`.
   - Add a **Code** node named `Build complete page inventory` to aggregate Markdown text and flag blank pages. Connect `Read each page` -> `Build complete page inventory`.
5. **AI Extraction Integration:**
   - Add an **Information Extractor** node named `Propose document boundaries`. Configure manual schema with properties `documents` (array containing `startPage`, `endPage`, `documentType`, `title`, `counterparty`, `reason`).
   - Add an **OpenAI Chat Model** node named `AI model - connect credential`. Set model to `gpt-4.1-mini`, temperature `0`, and configure your OpenAI API credentials. Connect `AI model - connect credential` (`ai_languageModel`) to `Propose document boundaries`. Connect `Build complete page inventory` (`main`) to `Propose document boundaries`.
6. **Review Form & Validation Loop 1:**
   - Add a **Code** node named `Prepare boundary review` to format AI output into HTML/form structures. Connect `Propose document boundaries` -> `Prepare boundary review`.
   - Add a **Form** node named `Review and correct boundaries` (operation: page, JSON output). Connect `Prepare boundary review` -> `Review and correct boundaries`.
   - Add a **Code** node named `Check page groups 1` to validate user submissions. Connect `Review and correct boundaries` -> `Check page groups 1`.
   - Add an **If** node named `Cancelled 1?` (`$json.cancelled === true`). Connect `Check page groups 1` -> `Cancelled 1?`.
   - Add an **If** node named `Page groups valid 1?` (`$json.valid === true`). Connect `Cancelled 1?` (false branch) -> `Page groups valid 1?`.
7. **Correction Loops (Attempts 2 & 3):**
   - Add **Form** nodes (`Correct page groups 2`, `Correct page groups 3`), **Code** validation nodes (`Check page groups 2`, `Check page groups 3`), and **If** branches (`Cancelled 2?`, `Page groups valid 2?`, `Cancelled 3?`, `Page groups valid 3?`) mirroring step 6.
   - Attach terminal completion forms (`Cancelled` and `Explain unresolved page groups`) for cancellations or exhausted attempts.
8. **Document Splitting & Google Drive Archival:**
   - Add a **Code** node named `Use approved page groups` connected from the `true` branches of `Page groups valid 1?`, `2?`, and `3?`.
   - Add a **pdfRest** node named `Split approved documents`. Set operation to `split` with `pageRanges` set to `={{ $json.ranges }}`. Connect `Use approved page groups` -> `Split approved documents`.
   - Add a **Code** node named `Name each reviewed document` to sanitize filenames. Connect `Split approved documents` -> `Name each reviewed document`.
   - Add an **If** node named `Save copies to Drive?` (`saveToDrive === true`). Connect `Name each reviewed document` -> `Save copies to Drive?`.
   - Add a **Google Drive** node named `File reviewed documents` (operation: upload). Connect `Save copies to Drive?` (true) -> `File reviewed documents`.
   - Add a **Code** node named `Restore filed document binaries` to reattach binaries and Drive IDs. Connect `File reviewed documents` -> `Restore filed document binaries`.
9. **Manifest Generation & ZIP Delivery:**
   - Add a **Code** node named `Build page manifest and download` to generate `page-manifest.json` and combine binaries. Connect both `Restore filed document binaries` and `Save copies to Drive?` (false branch) to this node.
   - Add a **Compression** node named `Create download ZIP` (operation: compress). Connect `Build page manifest and download` -> `Create download ZIP`.
   - Add a **Form** node named `Download reviewed documents` (operation: completion, respondWith: returnBinary, inputDataFieldName: bundle). Connect `Create download ZIP` -> `Download reviewed documents`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| pdfRest API Toolkit Integration | Requires active pdfRest API credentials connected to all five pdfRest nodes (`Inspect batch`, `OCR mixed batch`, `Split into individual pages`, `Read each page`, and `Split approved documents`). |
| OpenAI API Integration | Requires an active OpenAI API credential configured on the LangChain Chat Model node (`gpt-4.1-mini`). |
| Google Drive Optional Integration | Requires Google Drive OAuth2 credentials and a valid destination folder ID only if `saveToDrive` is enabled (`true`). |
| Execution Retention & Storage | Split PDFs and manifests stored in n8n depend on local instance execution-retention settings; users should promptly download the generated ZIP archive. |