AI-Powered Bank Statement Analysis & Transaction Categorization

https://n8nworkflows.xyz/workflows/ai-powered-bank-statement-analysis---transaction-categorization-6629


# AI-Powered Bank Statement Analysis & Transaction Categorization

### 1. Workflow Overview

This workflow automates the ingestion, extraction, categorization, and storage of bank statement data uploaded by users. It is designed to handle multiple file formats (PDF and Excel/CSV), extract transaction details using AI (OpenAI GPT-4), clean and validate the data, summarize expenses and income, and finally save the processed information to a PostgreSQL database. The workflow concludes by responding to the user with a concise summary of the processed statement.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Receives uploaded bank statement files via a webhook.
- **1.2 File Preparation & Routing:** Processes uploaded file metadata and routes files based on type (PDF or Excel/CSV).
- **1.3 Data Extraction:** Extracts raw text from PDFs or parses Excel/CSV files.
- **1.4 AI Processing:** Uses OpenAI GPT-4 to extract structured bank statement data and categorize transactions.
- **1.5 Data Cleanup & Summarization:** Cleans and validates AI output, computes financial summaries.
- **1.6 Persistence & Response:** Saves data to PostgreSQL database and sends a structured success response to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures incoming bank statement files uploaded by users through an HTTP POST request.

**Nodes Involved:**  
- Upload Statement

**Node Details:**  

- **Upload Statement**  
  - Type: Webhook  
  - Role: Entry point for bank statement file uploads. Receives raw binary data via POST on path `/upload-statement`.  
  - Configuration: Raw body reception enabled to handle binary uploads; responds via a response node downstream.  
  - Inputs: External HTTP POST request  
  - Outputs: Passes raw file data and metadata to next node  
  - Edge cases: Large file uploads might timeout; unsupported HTTP methods; missing file data  
  - Notes: Marked as "ENTRY POINT" for user uploads.

---

#### 1.2 File Preparation & Routing

**Overview:**  
Processes uploaded file metadata for downstream handling and automatically routes files based on MIME type (PDF vs Excel/CSV).

**Nodes Involved:**  
- File Handler  
- Check File Type  

**Node Details:**  

- **File Handler**  
  - Type: Code (JavaScript)  
  - Role: Extracts file metadata from raw binary input, prepares a JSON object indicating readiness for processing.  
  - Configuration: Parses input binary files, collects filename, MIME type, and upload timestamp. Outputs metadata plus original binary.  
  - Inputs: Upload Statement node output  
  - Outputs: Structured metadata and binary file data  
  - Edge cases: No files uploaded; multiple files; unexpected binary structure  
  - Notes: Covers multiple file uploads and prepares for routing.

- **Check File Type**  
  - Type: If Node  
  - Role: Determines file processing path based on MIME type. Checks if content type contains "application/pdf".  
  - Configuration: Condition checks if contentType includes "application/pdf" (case sensitive).  
  - Inputs: File Handler output  
  - Outputs:  
    - True branch: PDF files routed to PDF text extraction  
    - False branch: Other files routed to Excel/CSV parser  
  - Edge cases: Unknown or unsupported MIME types; missing contentType field  
  - Notes: Implements smart routing for format detection.

---

#### 1.3 Data Extraction

**Overview:**  
Extracts raw textual data from bank statements depending on file format.

**Nodes Involved:**  
- Extract PDF Text  
- Parse Excel/CSV  

**Node Details:**  

- **Extract PDF Text**  
  - Type: Extract From File  
  - Role: Uses OCR or PDF text extraction to convert PDF content into text.  
  - Configuration: Operation set to "extractText".  
  - Inputs: Check File Type (True branch)  
  - Outputs: Extracted text content passed forward  
  - Edge cases: Poor scan quality; encrypted PDFs; large PDFs causing timeouts  
  - Notes: Dedicated PDF handler.

- **Parse Excel/CSV**  
  - Type: Spreadsheet File  
  - Role: Parses Excel or CSV bank statements into structured data arrays.  
  - Configuration: Header row at index 0; operation set to "parseExcel".  
  - Inputs: Check File Type (False branch)  
  - Outputs: Parsed spreadsheet JSON data  
  - Edge cases: Malformed files; missing header row; large files  
  - Notes: Dedicated spreadsheet handler.

---

#### 1.4 AI Processing

**Overview:**  
Uses GPT-4 AI to transform extracted raw text or parsed data into a clean, categorized JSON bank statement representation.

**Nodes Involved:**  
- AI Data Extractor  

**Node Details:**  

- **AI Data Extractor**  
  - Type: OpenAI Node  
  - Role: Sends extracted text or parsed data to GPT-4 for data extraction and automatic categorization.  
  - Configuration:  
    - Model: "gpt-4o-mini"  
    - Prompts system with instructions to extract account details, transactions, and categorize them into predefined categories.  
    - User message includes extracted text or parsed data dynamically.  
  - Inputs: Extract PDF Text or Parse Excel/CSV output  
  - Outputs: AI-generated JSON containing account info, transactions, and categories  
  - Credentials: Requires OpenAI API credentials  
  - Edge cases: API rate limits; malformed prompts; incomplete AI output; network issues  
  - Notes: Core AI-powered extraction logic.

---

#### 1.5 Data Cleanup & Summarization

**Overview:**  
Validates and cleans AI output, generates unique transaction IDs, normalizes descriptions and amounts, and computes financial summaries by category.

**Nodes Involved:**  
- Process & Summarize  

**Node Details:**  

- **Process & Summarize**  
  - Type: Code (JavaScript)  
  - Role: Parses AI response, handles JSON extraction errors, normalizes transaction data, computes totals for income, expenses, and category breakdowns, adds processed timestamps.  
  - Key logic:  
    - Extracts JSON from AI response content using regex and JSON.parse  
    - Maps transactions with unique IDs and cleans data formatting  
    - Calculates total expenses (absolute sum of negative amounts), total income, and net change  
    - Summarizes expenses by category  
  - Inputs: AI Data Extractor output  
  - Outputs: Cleaned structured JSON with account info, transactions, summary, and status  
  - Edge cases: JSON parse errors; missing fields; empty transactions array  
  - Notes: Critical data validation and enrichment step.

---

#### 1.6 Persistence & Response

**Overview:**  
Stores processed data into a PostgreSQL database and sends a JSON response summarizing the transaction processing results.

**Nodes Involved:**  
- Save to Database  
- Send Response  

**Node Details:**  

- **Save to Database**  
  - Type: PostgreSQL  
  - Role: Inserts processed bank statement JSON and summary data into a configured PostgreSQL table named `bank_statements`.  
  - Configuration:  
    - Columns mapped: raw_data (full JSON string), bank_name, processed_at, total_income, account_number, total_expenses, statement_period, total_transactions  
  - Inputs: Process & Summarize output  
  - Credentials: PostgreSQL credentials required  
  - Edge cases: DB connection failures; unique constraints; insertion errors  
  - Notes: Ensures data persistence.

- **Send Response**  
  - Type: Respond to Webhook  
  - Role: Sends a success JSON response back to the client including account number, total transactions processed, total expenses, total income, and category breakdown.  
  - Configuration: JSON response with dynamic fields from processed data.  
  - Inputs: Process & Summarize output  
  - Outputs: HTTP response to webhook caller  
  - Edge cases: Response formatting errors; network issues  
  - Notes: Final user communication.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                 | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                         |
|---------------------|-----------------------|--------------------------------|-----------------------|---------------------------|---------------------------------------------------------------------------------------------------|
| Upload Statement    | Webhook               | Entry point for file upload     | -                     | File Handler              | 📥 **ENTRY POINT** Users upload bank statements here via POST request                             |
| File Handler        | Code                  | Extracts file metadata          | Upload Statement      | Check File Type           | 🔍 **FILE PREP** Extracts file info and prepares for processing. Handles multiple file formats.  |
| Check File Type     | If                    | Routes based on file type       | File Handler          | Extract PDF Text, Parse Excel/CSV | 🔀 **SMART ROUTING** PDFs go to text extraction; Excel/CSV to parser. Automatic format detection.  |
| Extract PDF Text    | Extract From File      | Extracts text from PDFs         | Check File Type       | AI Data Extractor         | 📄 **PDF HANDLER** Extracts text from PDF bank statements using OCR                               |
| Parse Excel/CSV     | Spreadsheet File      | Parses Excel/CSV files          | Check File Type       | AI Data Extractor         | 📊 **SPREADSHEET HANDLER** Parses Excel/CSV files and converts to structured data                 |
| AI Data Extractor   | OpenAI                | Extracts data and categorizes   | Extract PDF Text, Parse Excel/CSV | Process & Summarize        | 🤖 **AI MAGIC** GPT-4 extracts account details, transactions, auto-categorizes expenses          |
| Process & Summarize | Code                  | Cleans and summarizes data      | AI Data Extractor     | Save to Database, Send Response | 🧹 **DATA CLEANUP** Cleans & validates transaction formatting, amounts, and category summaries    |
| Save to Database    | PostgreSQL            | Stores processed data           | Process & Summarize   | -                         | 💾 **PERSISTENCE** Saves processed data to PostgreSQL database                                   |
| Send Response       | Respond to Webhook    | Returns processing summary      | Process & Summarize   | -                         | ✅ **SUCCESS RESPONSE** Returns summary: transaction count, expenses, category breakdown          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: Upload Statement  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/upload-statement`  
   - Options: Enable raw body reception (for binary files)  
   - Response Mode: Response Node (enable downstream responding)  

2. **Create Code Node for File Handling**  
   - Name: File Handler  
   - Connect from: Upload Statement  
   - Paste JavaScript code to extract filenames, content types, and timestamps from the binary input, outputting metadata alongside the original binary data.  

3. **Create If Node for File Type Checking**  
   - Name: Check File Type  
   - Connect from: File Handler  
   - Condition: Check if `contentType` includes string `application/pdf` (case sensitive)  
   - True branch: PDF processing  
   - False branch: Excel/CSV processing  

4. **Create Extract From File Node**  
   - Name: Extract PDF Text  
   - Connect from: Check File Type (True branch)  
   - Operation: Extract Text (default OCR/text extraction)  

5. **Create Spreadsheet File Node**  
   - Name: Parse Excel/CSV  
   - Connect from: Check File Type (False branch)  
   - Operation: Parse Excel  
   - Options: Header Row at index 0  

6. **Create OpenAI Node**  
   - Name: AI Data Extractor  
   - Connect from: Extract PDF Text and Parse Excel/CSV (both to same AI node input)  
   - Model: `gpt-4o-mini` (or equivalent GPT-4 variant)  
   - Messages:  
     - System prompt instructing extraction of account details, transactions, and categorization with example output structure.  
     - User prompt passing extracted text or parsed data dynamically.  
   - Credentials: Configure OpenAI API credentials  

7. **Create Code Node for Data Cleanup & Summarization**  
   - Name: Process & Summarize  
   - Connect from: AI Data Extractor  
   - JavaScript code to parse AI JSON output, handle errors, clean transaction data, generate transaction IDs, calculate totals, and build summary JSON.  

8. **Create PostgreSQL Node**  
   - Name: Save to Database  
   - Connect from: Process & Summarize  
   - Operation: Insert  
   - Table: `bank_statements` (create this table with columns for raw_data, bank_name, processed_at, total_income, account_number, total_expenses, statement_period, total_transactions)  
   - Map columns to JSON fields accordingly  
   - Credentials: Configure PostgreSQL connection credentials  

9. **Create Respond to Webhook Node**  
   - Name: Send Response  
   - Connect from: Process & Summarize  
   - Response Type: JSON  
   - Response Body: JSON object with success status, account number, transactions count, total expenses, total income, and categories breakdown dynamically filled from workflow data  

10. **Connect Save to Database and Send Response nodes in parallel** from Process & Summarize node outputs.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                              |
|------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Handles both PDF and Excel bank statement uploads in a single workflow with smart format detection.  | Workflow feature overview                                    |
| Utilizes GPT-4 for advanced extraction and auto-categorization of bank transaction data.             | AI Data Extractor node description                            |
| Saves complete raw JSON and summarized data to a PostgreSQL database for audit and analysis.         | Persistence block notes                                      |
| Returns structured JSON response summarizing results for API clients or UI consumption.              | Success response node notes                                  |
| Typical output example: includes success flag, transaction count, expense totals, and category sums. | See sticky note with JSON snippet                            |

---

**Disclaimer:** The provided workflow is generated exclusively by an automated n8n process, fully compliant with content policies and legal standards. All data processed is legal and public.