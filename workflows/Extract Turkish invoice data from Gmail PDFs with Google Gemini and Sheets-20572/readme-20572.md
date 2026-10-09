Extract Turkish invoice data from Gmail PDFs with Google Gemini and Sheets

https://n8nworkflows.xyz/workflows/extract-turkish-invoice-data-from-gmail-pdfs-with-google-gemini-and-sheets-20572


# Extract Turkish invoice data from Gmail PDFs with Google Gemini and Sheets

### 1. Workflow Overview

This workflow automates the collection, processing, and recording of Turkish invoices received via email. It periodically checks a Gmail inbox for messages containing PDF attachments, handles multi-attachment emails, stores files temporarily in Google Drive, extracts their text, utilizes Google Gemini to classify and parse the document data, and finally writes structured summary and line-item details into two separate Google Sheets tabs.

The workflow logic is divided into the following functional blocks:
- **1.1 Input Reception & Preparation:** Triggers hourly on incoming emails with PDF attachments, captures message metadata, splits multi-file emails into individual items, and merges the context.
- **1.2 Attachment Extraction Loop:** Iterates through each attachment sequentially, uploading it to a temporary Google Drive folder and downloading it back to extract the raw text payload.
- **1.3 Invoice Classification Agent:** Evaluates whether the extracted text represents a valid invoice using an AI agent powered by Google Gemini.
- **1.4 Conditional Routing & Cleanup:** Branches execution based on the classification result; non-invoice files trigger a cleanup step to delete the temporary Google Drive file, while valid invoices proceed to deep data extraction.
- **1.5 Detailed AI Extraction:** Uses an LLM chain with a strict JSON schema and Google Gemini to parse comprehensive Turkish invoice fields and item arrays.
- **1.6 Data Persistence & Loopback:** Appends header metrics to the “Faturalar” sheet, splits and normalizes line items for the “Kalem Detay” sheet, merges both write branches, and loops back to process the next attachment.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preparation
- **Overview:** Captures incoming invoice emails from Gmail, isolates attachment keys and message identifiers, splits multiple attachments into individual items, and synchronizes them with the core message body.
- **Nodes Involved:** 
  - When Email Received
  - Set Email Fields
  - Split Attachments
  - Merge Email Data

- **Node Details:**
  - **When Email Received**
    - *Type and Technical Role:* `n8n-nodes-base.gmailTrigger` (Trigger)
    - *Configuration:* Polls every hour at minute 1 using Gmail OAuth2. Uses query filter `filename:pdf has:attachment` and downloads attachments automatically.
    - *Expressions/Variables:* None.
    - *Connections:* Outputs to `Merge Email Data` and `Set Email Fields`.
    - *Edge Cases/Failures:* OAuth2 token revocation or missing mailbox permissions will halt polling.

  - **Set Email Fields**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Extracts binary attachment keys into an array and captures the email message ID.
    - *Expressions/Variables:* `={{ $('When Email Received').item.binary.keys() }}` and `={{ $json.id }}`.
    - *Connections:* Input from `When Email Received`; output to `Split Attachments`.
    - *Edge Cases/Failures:* Fails if the trigger object lacks binary keys or message IDs.

  - **Split Attachments**
    - *Type and Technical Role:* `n8n-nodes-base.splitOut` (Flow Control)
    - *Configuration:* Splits the `attachments` array field into individual items, keeping other fields.
    - *Expressions/Variables:* Field to split: `attachments`.
    - *Connections:* Input from `Set Email Fields`; output to `Merge Email Data`.
    - *Edge Cases/Failures:* Emails with zero attachments yield no items.

  - **Merge Email Data**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Flow Control)
    - *Configuration:* Combines the split attachment items with the original email payload using advanced matching (`message_id` equals `id`).
    - *Expressions/Variables:* Match fields `message_id` and `id`.
    - *Connections:* Inputs from `Split Attachments` and `When Email Received`; output to `Loop Over Attachments`.
    - *Edge Cases/Failures:* Mismatched IDs cause items to drop.

---

#### 2.2 Attachment Extraction Loop
- **Overview:** Sequentially processes each extracted attachment by uploading it to Google Drive as a temporary working file, downloading it back, and parsing its text content.
- **Nodes Involved:**
  - Loop Over Attachments
  - Upload to Google Drive
  - Download from Google Drive
  - Extract Invoice Data

- **Node Details:**
  - **Loop Over Attachments**
    - *Type and Technical Role:* `n8n-nodes-base.splitInBatches` (Flow Control)
    - *Configuration:* Iterates through incoming items in batches to manage file streams.
    - *Connections:* Input from `Merge Email Data` and `Merge Sheet Data`; outputs loop continuation to `Upload to Google Drive`.
    - *Edge Cases/Failures:* Infinite loops if batch indexing misbehaves; ensure proper loopback closure.

  - **Upload to Google Drive**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Action)
    - *Configuration:* Uploads files to Google Drive folder ID `1DBYblztvxSefIGNvdXtDVcEz2HYFOC8J` (named "FATURALAR") using Google Drive OAuth2. Dynamically names files using email subject and attachment name.
    - *Expressions/Variables:* `={{ $('When Email Received').item.json.headers.subject }}_{{ $json.attachments }}`.
    - *Connections:* Input from `Loop Over Attachments`; output to `Download from Google Drive`.
    - *Edge Cases/Failures:* Storage quota limits or incorrect folder IDs throw API errors.

  - **Download from Google Drive**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Action)
    - *Configuration:* Downloads the uploaded file back using its generated file ID (`operation: download`).
    - *Expressions/Variables:* `={{ $json.id }}`.
    - *Connections:* Input from `Upload to Google Drive`; output to `Extract Invoice Data`.
    - *Edge Cases/Failures:* File propagation delays or missing file IDs.

  - **Extract Invoice Data**
    - *Type and Technical Role:* `n8n-nodes-base.extractFromFile` (Data Extraction)
    - *Configuration:* Extracts text content from PDF binary property (`data`).
    - *Expressions/Variables:* Binary property: `data`.
    - *Connections:* Input from `Download from Google Drive`; output to `AI Invoice Agent`.
    - *Edge Cases/Failures:* Scanned image-based PDFs without embedded text layers yield empty strings.

---

#### 2.3 Invoice Classification Agent
- **Overview:** Evaluates the extracted text to determine if the document qualifies as an invoice.
- **Nodes Involved:**
  - AI Invoice Agent
  - Google Gemini AI Chat
  - Parse Structured Details

- **Node Details:**
  - **AI Invoice Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI / LLM)
    - *Configuration:* Evaluates text content and returns either `"yes"` or `"no"` via a structured output parser.
    - *Expressions/Variables:* Prompt: `=Based on the text decide if the text is about an invoice or not: {{ $json.text }}`.
    - *Connections:* Input from `Extract Invoice Data`; output model connection from `Google Gemini AI Chat`, parser connection from `Parse Structured Details`, main execution output to `If Status Is Valid`.
    - *Edge Cases/Failures:* Ambiguous text formats may result in incorrect classification.

  - **Google Gemini AI Chat**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Model)
    - *Configuration:* Uses model `models/gemini-3.5-flash-lite` with Google Palm credentials.
    - *Connections:* Connects to `AI Invoice Agent` via `ai_languageModel`.

  - **Parse Structured Details**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Output Parser)
    - *Configuration:* Enforces a simple JSON schema expecting a `status` field.
    - *Connections:* Connects to `AI Invoice Agent` via `ai_outputParser`.

---

#### 2.4 Conditional Routing & Cleanup
- **Overview:** Routes documents based on classification; deletes temporary files for non-invoices and forwards valid invoices for deep data parsing.
- **Nodes Involved:**
  - If Status Is Valid
  - Delete from Google Drive

- **Node Details:**
  - **If Status Is Valid**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Checks if `{{ $json.output.status }}` equals `"yes"`.
    - *Expressions/Variables:* `={{ $json.output.status }}` == `yes`.
    - *Connections:* Input from `AI Invoice Agent`; True branch outputs to `Read Invoice Data`, False branch outputs to `Delete from Google Drive`.
    - *Edge Cases/Failures:* Schema output deviations cause routing failures.

  - **Delete from Google Drive**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Action)
    - *Configuration:* Deletes the temporary file from Google Drive if it is rejected as a non-invoice.
    - *Expressions/Variables:* `={{ $('Download from Google Drive').item.json.id }}`.
    - *Connections:* Input from `If Status Is Valid` (False branch).
    - *Edge Cases/Failures:* Missing file ID references.

---

#### 2.5 Detailed AI Extraction
- **Overview:** Parses valid invoice text using an LLM chain and a strict JSON schema tailored for Turkish e-Invoices and e-Archives.
- **Nodes Involved:**
  - Read Invoice Data
  - Google Gemini Chat Assistant
  - Parse Structured Output

- **Node Details:**
  - **Read Invoice Data**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI / LLM Chain)
    - *Configuration:* Executes a detailed prompt instructing the model to extract Turkish invoice headers, vendor/buyer tax numbers, financial totals, and line item arrays into a precise JSON schema.
    - *Expressions/Variables:* Prompt references `={{ $('Extract Invoice Data').item.json.text }}`.
    - *Connections:* Input from `If Status Is Valid` (True branch); model input from `Google Gemini Chat Assistant`; parser input from `Parse Structured Output`; main outputs to `Append to Invoice Sheet` and `Split Invoice Items`.
    - *Edge Cases/Failures:* Complex layouts or missing tax identifiers result in fallback values like `"OKUNAMADI"` or `null`.

  - **Google Gemini Chat Assistant**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Model)
    - *Configuration:* Uses model `models/gemini-3.5-flash-lite` with Google Palm credentials.
    - *Connections:* Connects to `Read Invoice Data` via `ai_languageModel`.

  - **Parse Structured Output**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Output Parser)
    - *Configuration:* Enforces a comprehensive JSON schema covering invoice metadata, category enums, and item rows.
    - *Connections:* Connects to `Read Invoice Data` via `ai_outputParser`.

---

#### 2.6 Data Persistence & Loopback
- **Overview:** Writes header data to the main invoice sheet, flattens and writes line items to a secondary sheet tab, merges the write paths, and returns to the attachment loop.
- **Nodes Involved:**
  - Append to Invoice Sheet
  - Split Invoice Items
  - Set Invoice Number
  - Append to Item Details
  - Merge Sheet Data

- **Node Details:**
  - **Append to Invoice Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action)
    - *Configuration:* Appends invoice summary attributes to the spreadsheet document `108fjJDV2G0VHMFeIUGj_lgbsQQBUatfJlvitTyEVr_k`, sheet tab `Faturalar`.
    - *Expressions/Variables:* Maps fields like `={{ $json.output.fatura_no }}` and processing date `={{ $now.format('dd.MM.yyyy') }}`.
    - *Connections:* Input from `Read Invoice Data`; output to `Merge Sheet Data`.
    - *Edge Cases/Failures:* Spreadsheet permission errors or column mismatch.

  - **Split Invoice Items**
    - *Type and Technical Role:* `n8n-nodes-base.splitOut` (Data Transformation)
    - *Configuration:* Splits the `output.kalemler` array into individual line-item objects.
    - *Expressions/Variables:* Field: `output.kalemler`.
    - *Connections:* Input from `Read Invoice Data`; output to `Set Invoice Number`.
    - *Edge Cases/Failures:* Empty item arrays yield no rows.

  - **Set Invoice Number**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Propagates the parent invoice number (`fatura_no`) down to each split line item while preserving other fields.
    - *Expressions/Variables:* `={{ $('Read Invoice Data').item.json.output.fatura_no }}`.
    - *Connections:* Input from `Split Invoice Items`; output to `Append to Item Details`.
    - *Edge Cases/Failures:* Missing parent invoice reference.

  - **Append to Item Details**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action)
    - *Configuration:* Appends item-level details to spreadsheet document `108fjJDV2G0VHMFeIUGj_lgbsQQBUatfJlvitTyEVr_k`, sheet tab `Kalem Detay` (gid: `2098142631`).
    - *Expressions/Variables:* Maps `Tutar`, `Miktar`, `Fatura No`, `Açıklama`, `KDV Oranı`, and `Birim Fiyat`.
    - *Connections:* Input from `Set Invoice Number`; output to `Merge Sheet Data`.
    - *Edge Cases/Failures:* Sheet mapping issues.

  - **Merge Sheet Data**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Flow Control)
    - *Configuration:* Combines the parallel Google Sheets write branches (header write and line item writes) back into a single execution stream.
    - *Connections:* Inputs from `Append to Invoice Sheet` and `Append to Item Details`; output loops back to `Loop Over Attachments`.
    - *Edge Cases/Failures:* Unbalanced branch completions can cause execution stalls.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Email Received | n8n-nodes-base.gmailTrigger | Polls incoming emails with PDF attachments | None | Merge Email Data, Set Email Fields | ## AI Invoice Processing<br><br>### How it works<br><br>This workflow monitors Gmail for incoming messages with invoice attachments, separates each attachment, and processes them one at a time. It temporarily uploads and downloads each file through Google Drive, extracts the file text, uses Gemini-powered AI steps to classify and read invoice data, then writes invoice and line-item details into Google Sheets. Non-invoice or rejected files are routed to a cleanup path that deletes the temporary Drive file, while completed sheet writes merge back into the loop for the next attachment.<br><br>### Setup steps<br><br>- Connect and authorize Gmail credentials for the Gmail Trigger, and configure the trigger filters to match the mailbox or labels where invoice emails arrive.<br>- Connect Google Drive credentials and set the upload/download/delete nodes to use the intended temporary folder or file handling settings.<br>- Connect Google Gemini credentials for both AI nodes, and verify the structured output schemas match the fields expected downstream.<br>- Connect Google Sheets credentials and configure both append-row nodes with the target spreadsheet, sheets, and column mappings.<br>- Test with sample invoice and non-invoice attachments to confirm classification, extraction, deletion, and looping behavior.<br><br>### Customization<br><br>You can customize the Gmail search criteria, the Drive temporary storage location, the Gemini prompts and structured schemas, and the Google Sheets column mappings for different invoice formats or additional extracted fields. |
| Set Email Fields | n8n-nodes-base.set | Extracts attachment keys and message ID | When Email Received | Split Attachments | ## Email attachment intake<br><br>Starts the workflow from Gmail, captures attachment metadata and the message ID, splits multiple attachments into separate items, and merges them with the original email context. |
| Split Attachments | n8n-nodes-base.splitOut | Splits attachments array into individual items | Set Email Fields | Merge Email Data | ## Email attachment intake<br><br>Starts the workflow from Gmail, captures attachment metadata and the message ID, splits multiple attachments into separate items, and merges them with the original email context. |
| Merge Email Data | n8n-nodes-base.merge | Merges split attachments with email context | Split Attachments, When Email Received | Loop Over Attachments | ## Email attachment intake<br><br>Starts the workflow from Gmail, captures attachment metadata and the message ID, splits multiple attachments into separate items, and merges them with the original email context. |
| Loop Over Attachments | n8n-nodes-base.splitInBatches | Iterates through attachments sequentially | Merge Email Data, Merge Sheet Data | Upload to Google Drive | ## Attachment extraction loop<br><br>Iterates through each attachment, uploads it to Google Drive, downloads it back for processing, and extracts text or structured content from the file. |
| Upload to Google Drive | n8n-nodes-base.googleDrive | Uploads attachment to temporary Google Drive folder | Loop Over Attachments | Download from Google Drive | ## Attachment extraction loop<br><br>Iterates through each attachment, uploads it to Google Drive, downloads it back for processing, and extracts text or structured content from the file. |
| Download from Google Drive | n8n-nodes-base.googleDrive | Downloads file back from Google Drive for processing | Upload to Google Drive | Extract Invoice Data | ## Attachment extraction loop<br><br>Iterates through each attachment, uploads it to Google Drive, downloads it back for processing, and extracts text or structured content from the file. |
| Extract Invoice Data | n8n-nodes-base.extractFromFile | Extracts text content from PDF binary | Download from Google Drive | AI Invoice Agent | ## Attachment extraction loop<br><br>Iterates through each attachment, uploads it to Google Drive, downloads it back for processing, and extracts text or structured content from the file. |
| AI Invoice Agent | @n8n/n8n-nodes-langchain.agent | Classifies whether extracted text is an invoice | Extract Invoice Data | If Status Is Valid | ## Invoice classification agent<br><br>Uses an AI agent with a Gemini chat model and structured output parser to decide whether the extracted file content should continue through invoice processing. |
| Google Gemini AI Chat | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides LLM backend for classification agent | None | AI Invoice Agent | ## Invoice classification agent<br><br>Uses an AI agent with a Gemini chat model and structured output parser to decide whether the extracted file content should continue through invoice processing. |
| Parse Structured Details | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces classification output schema | None | AI Invoice Agent | ## Invoice classification agent<br><br>Uses an AI agent with a Gemini chat model and structured output parser to decide whether the extracted file content should continue through invoice processing. |
| If Status Is Valid | n8n-nodes-base.if | Branches based on invoice classification status | AI Invoice Agent | Read Invoice Data, Delete from Google Drive | ## Route or delete file<br><br>Branches based on the invoice controller result, sending valid invoices onward and deleting temporary Drive files for the rejected path. |
| Delete from Google Drive | n8n-nodes-base.googleDrive | Deletes rejected non-invoice files from Drive | If Status Is Valid | None | ## Route or delete file<br><br>Branches based on the invoice controller result, sending valid invoices onward and deleting temporary Drive files for the rejected path. |
| Read Invoice Data | @n8n/n8n-nodes-langchain.chainLlm | Extracts structured invoice fields and line items via LLM | If Status Is Valid | Append to Invoice Sheet, Split Invoice Items | ## Extract invoice details<br><br>Reads accepted invoice content with a Gemini-backed LLM chain and structured output parser to produce the invoice fields used by the sheet-writing steps. |
| Google Gemini Chat Assistant | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides LLM backend for invoice extraction chain | None | Read Invoice Data | ## Extract invoice details<br><br>Reads accepted invoice content with a Gemini-backed LLM chain and structured output parser to produce the invoice fields used by the sheet-writing steps. |
| Parse Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces comprehensive invoice JSON schema | None | Read Invoice Data | ## Extract invoice details<br><br>Reads accepted invoice content with a Gemini-backed LLM chain and structured output parser to produce the invoice fields used by the sheet-writing steps. |
| Append to Invoice Sheet | n8n-nodes-base.googleSheets | Appends invoice summary header to Google Sheets | Read Invoice Data | Merge Sheet Data | ## Append invoice summary<br><br>Writes the main invoice-level data extracted by the AI chain into a Google Sheets row. |
| Split Invoice Items | n8n-nodes-base.splitOut | Splits extracted line-item array into individual items | Read Invoice Data | Set Invoice Number | ## Append line items<br><br>Splits extracted line-item data, prepares the invoice number field, appends item rows to a second Google Sheets target, then merges the sheet-write branches and returns to the attachment loop. |
| Set Invoice Number | n8n-nodes-base.set | Attaches parent invoice number to each line item | Split Invoice Items | Append to Item Details | ## Append line items<br><br>Splits extracted line-item data, prepares the invoice number field, appends item rows to a second Google Sheets target, then merges the sheet-write branches and returns to the attachment loop. |
| Append to Item Details | n8n-nodes-base.googleSheets | Appends line items to Google Sheets “Kalem Detay” | Set Invoice Number | Merge Sheet Data | ## Append line items<br><br>Splits extracted line-item data, prepares the invoice number field, appends item rows to a second Google Sheets target, then merges the sheet-write branches and returns to the attachment loop. |
| Merge Sheet Data | n8n-nodes-base.merge | Merges sheet-writing branches back to loop | Append to Invoice Sheet, Append to Item Details | Loop Over Attachments | ## Append line items<br><br>Splits extracted line-item data, prepares the invoice number field, appends item rows to a second Google Sheets target, then merges the sheet-write branches and returns to the attachment loop. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add a **Gmail Trigger** (`When Email Received`). Configure polling mode to every hour at minute 1 using Gmail OAuth2 credentials. Set filters to query `filename:pdf has:attachment` with attachment downloading enabled.
2. **Setup Email Intake & Splitting:**
   - Add a **Set** node (`Set Email Fields`). Assign `attachments` = `={{ $('When Email Received').item.binary.keys() }}` and `message_id` = `={{ $json.id }}`.
   - Add a **Split Out** node (`Split Attachments`). Set field to split out: `attachments`.
   - Add a **Merge** node (`Merge Email Data`). Set mode to `combine` with advanced merge matching `message_id` and `id`. Connect Gmail Trigger to input index 1 and Split Attachments to input index 0.
3. **Configure Attachment Loop & Drive Operations:**
   - Add a **Split In Batches** node (`Loop Over Attachments`). Connect `Merge Email Data` to its input.
   - Add a **Google Drive** node (`Upload to Google Drive`). Set operation to upload, specify Google Drive OAuth2 credentials, target folder ID (`1DBYblztvxSefIGNvdXtDVcEz2HYFOC8J`), and filename expression: `={{ $('When Email Received').item.json.headers.subject }}_{{ $json.attachments }}`.
   - Add a **Google Drive** node (`Download from Google Drive`). Set operation to download using file ID `={{ $json.id }}`.
   - Add an **Extract from File** node (`Extract Invoice Data`). Set operation to `pdf` and binary property name to `data`.
4. **Setup Classification AI Agent:**
   - Add an **AI Agent** (`AI Invoice Agent`). Set prompt: `=Based on the text decide if the text is about an invoice or not: {{ $json.text }}\n\nIf yes return \"yes\", if no return \"no\". Don't provide any additional information. Use only small letters`.
   - Attach a **Google Gemini Chat Model** (`Google Gemini AI Chat`) with model `models/gemini-3.5-flash-lite` and Google Palm API credentials to the agent's language model input.
   - Attach a **Structured Output Parser** (`Parse Structured Details`) defining a JSON schema with a single string property `status` to the agent's output parser input.
5. **Configure Conditional Branching & Cleanup:**
   - Add an **If** node (`If Status Is Valid`). Set condition testing if `={{ $json.output.status }}` equals `yes`.
   - Add a **Google Drive** node (`Delete from Google Drive`) connected to the False branch. Set operation to delete file using file ID `={{ $('Download from Google Drive').item.json.id }}`.
6. **Setup Detailed AI Invoice Extraction:**
   - Add a **Basic LLM Chain** (`Read Invoice Data`) connected to the True branch of the If node. Set prompt with Turkish invoice instructions and reference `={{ $('Extract Invoice Data').item.json.text }}`.
   - Attach a **Google Gemini Chat Model** (`Google Gemini Chat Assistant`) with model `models/gemini-3.5-flash-lite` and Google Palm API credentials.
   - Attach a **Structured Output Parser** (`Parse Structured Output`) configured with the comprehensive manual JSON schema provided in the workflow definition (covering headers, optional fields, and `kalemler` line-item array).
7. **Configure Data Persistence (Google Sheets):**
   - Add a **Google Sheets** node (`Append to Invoice Sheet`). Configure Google Sheets OAuth2 credentials, document ID `108fjJDV2G0VHMFeIUGj_lgbsQQBUatfJlvitTyEVr_k`, sheet name `Faturalar`, mapping mode to define below, and map all header fields (`Fatura No`, `Matrah`, `Genel Toplam`, `Alıcı VKN`, etc.) using `={{ $json.output.<fieldname> }}` alongside `={{ $now.format('dd.MM.yyyy') }}` for the processing date.
   - Add a **Split Out** node (`Split Invoice Items`) to separate `output.kalemler`.
   - Add a **Set** node (`Set Invoice Number`) to assign `fatura_no` = `={{ $('Read Invoice Data').item.json.output.fatura_no }}` while including other fields.
   - Add a **Google Sheets** node (`Append to Item Details`). Configure document ID `108fjJDV2G0VHMFeIUGj_lgbsQQBUatfJlvitTyEVr_k`, sheet tab `Kalem Detay`, and map item properties (`Fatura No`, `Açıklama`, `Miktar`, `Birim Fiyat`, `KDV Oranı`, `Tutar`).
8. **Finalize Loopback Connections:**
   - Add a **Merge** node (`Merge Sheet Data`) to combine the outputs of `Append to Invoice Sheet` and `Append to Item Details`.
   - Connect the output of `Merge Sheet Data` back to the loop input (`Loop Over Attachments`) to complete the batch processing cycle.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Source spreadsheet used for testing and production writing of invoices and line items. | [Google Sheets Document](https://docs.google.com/spreadsheets/d/108fjJDV2G0VHMFeIUGj_lgbsQQBUatfJlvitTyEVr_k/edit?usp=drivesdk) |
| Temporary Google Drive repository folder designated for incoming PDF attachments (`FATURALAR`). | [Google Drive Folder](https://drive.google.com/drive/folders/1DBYblztvxSefIGNvdXtDVcEz2HYFOC8J) |