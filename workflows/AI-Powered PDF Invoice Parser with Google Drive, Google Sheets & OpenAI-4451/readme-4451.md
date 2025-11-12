AI-Powered PDF Invoice Parser with Google Drive, Google Sheets & OpenAI

https://n8nworkflows.xyz/workflows/ai-powered-pdf-invoice-parser-with-google-drive--google-sheets---openai-4451


# AI-Powered PDF Invoice Parser with Google Drive, Google Sheets & OpenAI

---

### 1. Workflow Overview

This workflow automates the processing of PDF invoices stored in a specific Google Drive folder. It is designed for businesses seeking to automate bookkeeping tasks by extracting structured invoice data and logging it into a Google Sheets document. The workflow integrates Google Drive and Google Sheets with OpenAI’s GPT-4 language model for intelligent text analysis and categorization.

**Logical Blocks:**

- **1.1 Input Reception:** Monitoring Google Drive folder and downloading new PDF invoices.
- **1.2 PDF Processing:** Extracting text content from downloaded PDF files.
- **1.3 AI Processing:** Parsing extracted text with GPT-4 to extract structured invoice information and categorize invoices.
- **1.4 Data Storage:** Appending the parsed invoice data into a structured Google Sheets spreadsheet.
- **1.5 Setup & Documentation:** Sticky notes providing setup instructions, workflow overview, and operational context.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block monitors a designated Google Drive folder for newly created PDF invoice files and downloads them for further processing.

**Nodes Involved:**  
- Invoice Folder Monitor  
- Download Invoice PDF  

**Node Details:**

- **Invoice Folder Monitor**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches a specific Google Drive folder for newly added files, triggering the workflow.  
  - *Configuration:*  
    - Event: `fileCreated`  
    - Folder to watch: ID `"1KJ4fvXcKVMGJunsKvPYf8PkX5K9SVwFk"` (specific Google Drive folder)  
    - Polling every minute to detect new files.  
  - *Input:* None (trigger node).  
  - *Output:* Passes new file metadata with file ID downstream.  
  - *Potential Failures:* Authentication errors with Google Drive OAuth2, missing folder permissions, API rate limits.  
  - *Credentials:* Google Drive OAuth2 API required.  

- **Download Invoice PDF**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the invoice PDF file as binary data using the file ID from the trigger.  
  - *Configuration:*  
    - Operation: `download`  
    - File ID: Expression referencing the incoming file metadata ID (`{{$json.id}}`).  
  - *Input:* File ID from Invoice Folder Monitor.  
  - *Output:* Binary data of the PDF file.  
  - *Potential Failures:* Download failure if file is deleted/moved before download, authentication errors, API quota exceeded.  
  - *Credentials:* Google Drive OAuth2 API required.  

---

#### 1.2 PDF Processing

**Overview:**  
Extracts readable text from the binary PDF data to prepare it for AI parsing.

**Nodes Involved:**  
- PDF Text Extractor  

**Node Details:**

- **PDF Text Extractor**  
  - *Type:* Extract from File node  
  - *Role:* Converts PDF binary data into raw text using built-in PDF extraction/OCR capabilities.  
  - *Configuration:*  
    - Operation: `pdf`  
    - Default extraction settings, no custom options specified.  
  - *Input:* Binary PDF data from Download Invoice PDF node.  
  - *Output:* Extracted plain text of the invoice.  
  - *Potential Failures:* Poor quality PDFs may yield incomplete text, extraction timeouts, unsupported PDF formats.  
  - *Version:* Compatible with n8n version supporting extractFromFile node (v1+).  

---

#### 1.3 AI Processing

**Overview:**  
Uses GPT-4 via Langchain integration to parse the extracted invoice text, extract key invoice fields, detect invoice category, and produce a structured JSON output.

**Nodes Involved:**  
- Invoice Parser AI Agent  
- OpenAI Chat Model  
- Structured Output Parser  

**Node Details:**

- **Invoice Parser AI Agent**  
  - *Type:* Langchain Agent node  
  - *Role:* Central AI agent that sends the extracted text prompt to GPT-4 to perform invoice parsing and categorization.  
  - *Configuration:*  
    - Prompt: A detailed instruction to extract invoice fields and categorize, with exact JSON output format specified (invoice number, dates, vendor, amounts, items, tax, category).  
    - Uses the extracted raw text as input (`{{ $json.text }}`) embedded into the prompt.  
    - Output parser enabled to enforce structured JSON response.  
  - *Input:* Extracted text from PDF Text Extractor.  
  - *Output:* JSON with parsed invoice data.  
  - *Potential Failures:* OpenAI API errors (rate limits, auth), malformed text causing parsing errors, incomplete extraction leading to missing fields.  
  - *Version:* Node version 1.9 supports output parser integration.  

- **OpenAI Chat Model**  
  - *Type:* Langchain Language Model node  
  - *Role:* Provides GPT-4 model backend for AI agent.  
  - *Configuration:*  
    - Model selected: `gpt-4o-mini` (GPT-4 variant optimized for smaller footprint).  
  - *Input:* Receives prompts from AI Agent.  
  - *Output:* Replies with parsed text data.  
  - *Credentials:* OpenAI API key required.  
  - *Potential Failures:* API key invalid, model unavailable, network issues.  

- **Structured Output Parser**  
  - *Type:* Langchain Output Parser node  
  - *Role:* Validates and structurally parses the AI response into a JSON object according to the schema example provided.  
  - *Configuration:*  
    - JSON schema example defines all expected invoice fields and nested items array.  
  - *Input:* AI Agent output.  
  - *Output:* Clean JSON object with invoice data fields.  
  - *Potential Failures:* Parsing failures if AI response deviates from expected format.  

---

#### 1.4 Data Storage

**Overview:**  
Appends parsed invoice data into a Google Sheets document maintaining an organized invoice log.

**Nodes Involved:**  
- Insert Invoice Data  

**Node Details:**

- **Insert Invoice Data**  
  - *Type:* Google Sheets node  
  - *Role:* Adds a new row to the "Invoices" sheet with the parsed invoice details.  
  - *Configuration:*  
    - Operation: `append`  
    - Document ID: `"1u5dHeytao9y3L0Mgv8cSomPVLS3CMrn_eOwXW3oQ3c8"` (specific Google Sheets document)  
    - Sheet: `"Invoices"` (gid=0)  
    - Columns mapped exactly to parsed JSON fields: Invoice Number, Invoice Date, Due Date, Vendor Name, Total Amount, Currency, Items, Tax, Category.  
    - Mapping mode: Defined below (explicit column mapping).  
  - *Input:* JSON data from Invoice Parser AI Agent (after output parsing).  
  - *Output:* Confirmation of row appended.  
  - *Potential Failures:* Google Sheets API authentication, quota limits, sheet structure mismatch, data type conversion issues.  
  - *Credentials:* Google Sheets OAuth2 API required.  

---

#### 1.5 Setup & Documentation

**Overview:**  
Sticky Note nodes provide essential setup instructions, workflow description, and process overview to help users understand and maintain the workflow.

**Nodes Involved:**  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  

**Node Details:**

- **Sticky Note2**  
  - *Content:*  
    - Describes required Google Sheets column structure for the "Invoices" sheet.  
    - Lists required credentials: Google Drive OAuth2, Google Sheets OAuth2, OpenAI API key.  
  - *Role:* Setup instructions for users.  

- **Sticky Note3**  
  - *Content:*  
    - Summarizes workflow capabilities: monitoring Drive folder, downloading PDFs, extracting text, AI parsing & categorization, saving to Sheets, full automation.  
  - *Role:* High-level workflow description.  

- **Sticky Note4**  
  - *Content:*  
    - Stepwise workflow process overview detailing each node’s role from folder monitoring to data insertion.  
  - *Role:* Operational overview for quick reference.  

---

### 3. Summary Table

| Node Name             | Node Type                                   | Functional Role                          | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                  |
|-----------------------|---------------------------------------------|----------------------------------------|-------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| Invoice Folder Monitor | Google Drive Trigger                         | Trigger workflow on new PDF file       | None                    | Download Invoice PDF       | Setup instructions and overview notes apply to the entire workflow                         |
| Download Invoice PDF   | Google Drive node                           | Download PDF invoice file binary       | Invoice Folder Monitor   | PDF Text Extractor         | Setup instructions and overview notes apply to the entire workflow                         |
| PDF Text Extractor     | Extract from File node                      | Extract text from PDF binary            | Download Invoice PDF     | Invoice Parser AI Agent    | Setup instructions and overview notes apply to the entire workflow                         |
| Invoice Parser AI Agent| Langchain Agent node                        | Parse invoice text and categorize       | PDF Text Extractor, OpenAI Chat Model, Structured Output Parser (via connections) | Insert Invoice Data         | Setup instructions and overview notes apply to the entire workflow                         |
| OpenAI Chat Model      | Langchain Language Model                    | Provides GPT-4 model for parsing       | Invoice Parser AI Agent  | Invoice Parser AI Agent    | Setup instructions and overview notes apply to the entire workflow                         |
| Structured Output Parser| Langchain Output Parser                     | Validates and structures AI output JSON| Invoice Parser AI Agent  | Invoice Parser AI Agent    | Setup instructions and overview notes apply to the entire workflow                         |
| Insert Invoice Data    | Google Sheets node                         | Append parsed invoice data to sheet    | Invoice Parser AI Agent  | None                      | Setup instructions and overview notes apply to the entire workflow                         |
| Sticky Note2           | Sticky Note node                           | Setup instructions                     | None                    | None                      | Google Sheets Structure and required credentials                                           |
| Sticky Note3           | Sticky Note node                           | Workflow description                   | None                    | None                      | AI-powered PDF Invoice Parser overview                                                     |
| Sticky Note4           | Sticky Note node                           | Workflow process overview              | None                    | None                      | Stepwise workflow process explanation                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node (Invoice Folder Monitor):**  
   - Type: Google Drive Trigger  
   - Event: `fileCreated`  
   - Folder to watch: Enter the folder ID where invoices will be uploaded.  
   - Polling: Set to check every minute.  
   - Credential: Configure Google Drive OAuth2 credentials.

2. **Create Google Drive Node (Download Invoice PDF):**  
   - Type: Google Drive  
   - Operation: `Download`  
   - File ID: Set to expression `{{$json["id"]}}` to get file ID from trigger.  
   - Credential: Use same Google Drive OAuth2 credentials.  
   - Connect the output of Invoice Folder Monitor to this node.

3. **Create Extract from File Node (PDF Text Extractor):**  
   - Type: Extract from File  
   - Operation: `pdf`  
   - No additional options needed.  
   - Connect output of Download Invoice PDF (binary data) to this node.

4. **Create Langchain Agent Node (Invoice Parser AI Agent):**  
   - Type: Langchain Agent  
   - Prompt Type: Define  
   - Text prompt: Insert detailed instructions to extract invoice fields and category from raw text, including expected JSON format.  
   - Enable output parser.  
   - Connect output of PDF Text Extractor to this node.

5. **Create Langchain Language Model Node (OpenAI Chat Model):**  
   - Type: Langchain Language Model  
   - Model: Select `gpt-4o-mini` or GPT-4 equivalent.  
   - Credential: Configure with OpenAI API key.  
   - Connect this node as `ai_languageModel` input to Invoice Parser AI Agent.

6. **Create Langchain Output Parser Node (Structured Output Parser):**  
   - Type: Langchain Output Parser  
   - Provide JSON schema example matching the expected invoice data structure.  
   - Connect as `ai_outputParser` input to Invoice Parser AI Agent.

7. **Create Google Sheets Node (Insert Invoice Data):**  
   - Type: Google Sheets  
   - Operation: `append`  
   - Document ID: Use your target Google Sheets document ID.  
   - Sheet Name: `Invoices` or equivalent sheet name.  
   - Columns mapping: Map JSON fields from AI output to corresponding columns (`Invoice Number`, `Invoice Date`, `Due Date`, `Vendor Name`, `Total Amount`, `Currency`, `Items`, `Tax`, `Category`).  
   - Credential: Configure Google Sheets OAuth2 credentials.  
   - Connect output of Invoice Parser AI Agent to this node.

8. **Add Sticky Notes for Documentation (Optional but Recommended):**  
   - Create Sticky Notes nodes to document:  
     - Google Sheets column structure and required credentials.  
     - Workflow overview and capabilities.  
     - Stepwise process overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Google Sheets must have columns: Invoice Number, Invoice Date, Due Date, Vendor Name, Total Amount, Currency, Items, Tax, Category in that order. | Setup instructions from Sticky Note2                                                                                   |
| Required credentials include Google Drive OAuth2, Google Sheets OAuth2, and OpenAI API key with GPT-4 access enabled.                          | Sticky Note2                                                                                                           |
| This workflow automates invoice processing from PDF receipt to structured data logging, reducing manual bookkeeping effort.                    | Sticky Note3                                                                                                           |
| Workflow polls Drive folder every minute for new invoices; ensure folder permissions and API quotas are configured properly.                      | Sticky Note4                                                                                                           |
| For best AI parsing results, ensure PDFs are text-based or have good OCR quality; corrupted or scanned PDFs may reduce accuracy.                | General best practice, implied from node functions                                                                     |
| OpenAI API usage may incur costs; monitor usage to avoid unexpected charges.                                                                      | General best practice related to OpenAI integration                                                                     |
| Links to Google Drive folder and Google Sheets document must be updated to those of your own environment.                                         | Replace placeholder IDs with your own Google Drive folder and Google Sheets document                                    |

---

**Disclaimer:**  
The provided text is exclusively from an n8n automated workflow. It complies with all content policies and contains no illegal or offensive material. All processed data is legal and publicly accessible.

---