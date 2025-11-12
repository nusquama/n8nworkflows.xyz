AI Auto-Save Gmail Receipts to Google Sheets + Google Drive

https://n8nworkflows.xyz/workflows/ai-auto-save-gmail-receipts-to-google-sheets---google-drive-5451


# AI Auto-Save Gmail Receipts to Google Sheets + Google Drive

---
### 1. Workflow Overview

This workflow automates the extraction and archival of receipt data from Gmail emails into Google Sheets and Google Drive, targeting users who want to streamline tax tracking and expense management. It operates by periodically scanning Gmail for unread emails labeled as receipts with attachments, downloading those attachments, extracting structured receipt information using an AI agent, and then saving both the raw files and extracted data for later use.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Email Retrieval:** Periodically triggers the workflow and fetches relevant unread emails with receipt attachments.
- **1.2 Attachment Download and Data Extraction:** Downloads email attachments and extracts textual data from PDF receipts.
- **1.3 AI-Based Receipt Data Parsing:** Uses an AI agent to analyze receipt text and extract structured fields.
- **1.4 Data Merging and Output Preparation:** Combines different data streams (raw extracted text, AI results, metadata) for unified processing.
- **1.5 Data Persistence & Archival:** Saves extracted data to Google Sheets and uploads receipt PDF files to Google Drive.
- **1.6 Email State Management:** Marks processed emails as read to avoid duplication.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Email Retrieval

- **Overview:**  
  This block initiates the workflow daily at 8 AM and fetches unread Gmail messages labeled specifically for receipts and containing attachments.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Emails with Receipts

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution every day at 8:00 AM.  
    - Configuration: Set to trigger daily at hour 8 (8 AM).  
    - Inputs: None (start node).  
    - Outputs: Triggers "Get Emails with Receipts".  
    - Edge Cases: If n8n server is offline at trigger time, execution might be delayed or skipped.

  - **Get Emails with Receipts**  
    - Type: Gmail Node  
    - Role: Retrieves all unread emails with attachments under a specific Gmail label for receipts.  
    - Configuration:  
      - Gmail OAuth2 credentials required.  
      - Query filter: `has:attachment label:<specific-label> is:unread`  
      - Returns all matching emails (no limit).  
      - Includes full message content (not simple).  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Passes emails to "Download Attachment".  
    - Edge Cases:  
      - Gmail API quota limits.  
      - Label must exist and be correctly set in Gmail.  
      - Network or auth failures.

---

#### 2.2 Attachment Download and Data Extraction

- **Overview:**  
  Downloads attachments from each matching email and extracts textual content from PDF invoices to prepare for AI analysis.

- **Nodes Involved:**  
  - Download Attachment  
  - Extract Data from Invoice  
  - Get Required Fields

- **Node Details:**

  - **Download Attachment**  
    - Type: Gmail Node  
    - Role: Downloads the attachments for each email.  
    - Configuration:  
      - Uses Gmail OAuth2 credentials.  
      - Downloads all attachments of the email identified by `id`.  
      - Outputs binary data of attachment(s).  
    - Inputs: Emails from "Get Emails with Receipts".  
    - Outputs: Binary attachments to "Upload File", "Extract Data from Invoice", and metadata to "Get Required Fields".  
    - Edge Cases:  
      - Large attachments may cause timeouts.  
      - Emails without attachments will fail or return empty attachments.  
      - Attachment binary data must be handled carefully.

  - **Extract Data from Invoice**  
    - Type: Extract From File Node  
    - Role: Extracts text from the first downloaded PDF attachment.  
    - Configuration:  
      - Operation set to PDF extraction.  
      - Uses the binary property `attachment_0`.  
    - Inputs: Binary attachment from "Download Attachment".  
    - Outputs: Extracted textual content to "Merge Data".  
    - Edge Cases:  
      - Non-PDF attachments may produce no output or error.  
      - Poor quality PDFs may yield incomplete extraction.

  - **Get Required Fields**  
    - Type: Set Node  
    - Role: Extracts and sets email metadata and key fields for further processing.  
    - Configuration:  
      - Maps email `id`, `threadId`, `textAsHtml`, `subject`, and `date` into JSON fields.  
    - Inputs: Metadata from "Download Attachment".  
    - Outputs: Metadata to "Merge Data".  
    - Edge Cases:  
      - Missing fields in email JSON could cause undefined values.

---

#### 2.3 AI-Based Receipt Data Parsing

- **Overview:**  
  Uses an AI agent with OpenAI GPT-4 to parse extracted receipt text into a structured JSON object containing standardized receipt fields.

- **Nodes Involved:**  
  - Merge Data  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **Merge Data**  
    - Type: Merge Node  
    - Role: Combines three inputs: extracted invoice text, email metadata, and AI parsing inputs.  
    - Configuration:  
      - Mode: Combine by position (combines corresponding items by their index).  
      - Inputs:  
        1. Output from "Extract Data from Invoice" (text)  
        2. Output from "Get Required Fields" (email metadata)  
        3. Output from "Download Attachment" (raw email data)  
    - Outputs: Merged data to "AI Agent".  
    - Edge Cases:  
      - Inputs must align by index, otherwise data mismatch may occur.

  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Sends receipt text to GPT-4 with a customized prompt to extract receipt details as JSON.  
    - Configuration:  
      - Prompt instructs extraction of specific fields (date, merchant, category, description, subtotal, tax, total, id, threadId).  
      - Returns only JSON object with defined fields.  
      - Retries enabled on failure.  
    - Inputs: Merged data from "Merge Data".  
    - Outputs: AI-generated JSON to "Append to Google Sheet" and "Mark Email as Read".  
    - Edge Cases:  
      - API rate limits or network errors.  
      - Parsing errors if prompt output is malformed.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Node  
    - Role: Provides GPT-4 model backend for the AI Agent.  
    - Configuration:  
      - Model: GPT-4  
      - Credentials: OpenAI API key required.  
    - Inputs: AI Agent language model requests.  
    - Outputs: AI Agent receives responses.  
    - Edge Cases:  
      - API quota exceeded.  
      - Invalid API key.

  - **Structured Output Parser**  
    - Type: Langchain Output Parser Node  
    - Role: Validates and parses AI Agent output against a strict JSON schema.  
    - Configuration:  
      - Schema requires all receipt fields with types and date format.  
    - Inputs: AI Agent raw output.  
    - Outputs: Parsed, validated JSON to AI Agent node continuation.  
    - Edge Cases:  
      - AI output not matching schema causes parsing failure.

---

#### 2.4 Data Persistence & Archival

- **Overview:**  
  Saves the parsed receipt data to a Google Sheet for record-keeping and uploads the original PDF receipt to a designated Google Drive folder.

- **Nodes Involved:**  
  - Append to Google Sheet  
  - Upload File

- **Node Details:**

  - **Append to Google Sheet**  
    - Type: Google Sheets Node  
    - Role: Appends a new row with receipt data into a specified Google Sheet.  
    - Configuration:  
      - Uses OAuth2 credentials for Google Sheets.  
      - Appends defined columns: Date, Merchant, Category, Description, Subtotal, Tax, Total.  
      - Matches columns by name.  
      - Sheet name and Document ID must be specified by the user.  
      - Does not attempt to convert field types (all strings).  
    - Inputs: AI Agent parsed output.  
    - Outputs: None (terminal for data persistence).  
    - Edge Cases:  
      - Invalid Sheet ID or insufficient permissions.  
      - Schema mismatch if column names differ.

  - **Upload File**  
    - Type: Google Drive Node  
    - Role: Uploads the downloaded receipt PDF to a user-specified Google Drive folder.  
    - Configuration:  
      - Uses Google Drive OAuth2 credentials.  
      - File name is dynamically generated from email date and sender name (cleaned for filesystem safety).  
      - Folder ID and Drive ID ("My Drive") must be set by the user.  
      - Input data field is the first attachment binary (`attachment_0`).  
    - Inputs: Attachment binary from "Download Attachment".  
    - Outputs: Merges data for AI Agent processing.  
    - Edge Cases:  
      - Invalid folder ID or permissions.  
      - Filename collisions or invalid characters.

---

#### 2.5 Email State Management

- **Overview:**  
  Marks each processed email as read to prevent reprocessing on subsequent workflow runs.

- **Nodes Involved:**  
  - Mark Email as Read

- **Node Details:**

  - **Mark Email as Read**  
    - Type: Gmail Node  
    - Role: Changes the Gmail message status to "read" for the processed email.  
    - Configuration:  
      - Uses Gmail OAuth2 credentials.  
      - Marks message identified by the AI Agent extracted `id`.  
    - Inputs: AI Agent output with `id` field.  
    - Outputs: None.  
    - Edge Cases:  
      - Failure to mark email read may cause duplicate processing.

---

#### 2.6 Sticky Notes (Documentation Nodes)

- **Overview:**  
  Provide in-editor documentation and usage hints.

- **Nodes Involved:**  
  - Sticky Note (Upload instructions)  
  - Sticky Note1 (Trigger information)

- **Node Details:**

  - **Sticky Note**  
    - Content: Explains uploading receipts to Google Drive, folder selection, and credential setup.  
    - No inputs/outputs.

  - **Sticky Note1**  
    - Content: Describes the schedule trigger node's role, frequency, and how to modify it.

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                           | Input Node(s)             | Output Node(s)                 | Sticky Note                                                                                                                |
|-------------------------|-------------------------------|-----------------------------------------|---------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger               | Starts workflow daily at 8AM             | None                      | Get Emails with Receipts       | ### 🕒 Entry Point – Auto-Run This Flow This node controls when the flow is triggered (currently 8am daily).                 |
| Get Emails with Receipts | Gmail                         | Fetch unread receipt emails with attachments | Schedule Trigger           | Download Attachment            |                                                                                                                            |
| Download Attachment     | Gmail                         | Download attachments from emails         | Get Emails with Receipts   | Upload File, Get Required Fields, Extract Data from Invoice |                                                                                                                            |
| Upload File             | Google Drive                  | Upload receipt PDF to Google Drive       | Download Attachment        | Merge Data                    | ### 📤 Upload Receipt to Google Drive This node saves the PDF attachment from the email to your Google Drive.              |
| Get Required Fields     | Set                           | Extract email metadata for processing    | Download Attachment        | Merge Data                    |                                                                                                                            |
| Extract Data from Invoice | Extract From File             | Extract text from PDF attachment          | Download Attachment        | Merge Data                    |                                                                                                                            |
| Merge Data              | Merge                         | Combine extracted text, metadata, raw email | Extract Data from Invoice, Get Required Fields, Upload File | AI Agent                      |                                                                                                                            |
| AI Agent                | Langchain Agent (GPT-4)       | Parse receipt text to structured JSON    | Merge Data                 | Append to Google Sheet, Mark Email as Read |                                                                                                                            |
| OpenAI Chat Model       | Langchain OpenAI              | Provides GPT-4 language model backend    | AI Agent (ai_languageModel) | AI Agent (response)            |                                                                                                                            |
| Structured Output Parser | Langchain Output Parser       | Validate and parse AI output JSON         | AI Agent (ai_outputParser) | AI Agent (continuation)        |                                                                                                                            |
| Append to Google Sheet  | Google Sheets                 | Append structured receipt data to sheet  | AI Agent                   | None                         |                                                                                                                            |
| Mark Email as Read      | Gmail                         | Mark processed email as read              | AI Agent                   | None                         |                                                                                                                            |
| Sticky Note             | Sticky Note                   | Documentation for Google Drive upload    | None                      | None                         | ### 📤 Upload Receipt to Google Drive This node saves the PDF attachment from the email to your Google Drive.              |
| Sticky Note1            | Sticky Note                   | Documentation for schedule trigger       | None                      | None                         | ### 🕒 Entry Point – Auto-Run This Flow This node controls when the flow is triggered (currently 8am daily).                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 8:00 AM (hour: 8)  
   - No credentials required  
   - Connect output to "Get Emails with Receipts".

2. **Create Get Emails with Receipts Node**  
   - Type: Gmail  
   - Operation: Get All  
   - Filters:  
     - Query: `has:attachment label:<your_receipts_label> is:unread`  
     - Label IDs: Use your Gmail label ID for receipts  
     - Read Status: Unread  
   - Credentials: Gmail OAuth2 setup with appropriate scopes  
   - Connect output to "Download Attachment".

3. **Create Download Attachment Node**  
   - Type: Gmail  
   - Operation: Get  
   - Message ID: Set to `{{$json["id"]}}` from input email  
   - Enable download attachments option  
   - Credentials: Same Gmail OAuth2 as above  
   - Connect outputs:  
     - To "Upload File"  
     - To "Get Required Fields"  
     - To "Extract Data from Invoice".

4. **Create Upload File Node**  
   - Type: Google Drive  
   - Action: Upload File  
   - File Name: Use expression to generate filename from date and sender name, sanitized for invalid characters  
   - Folder ID: Set to your desired Google Drive folder ID  
   - Drive ID: "My Drive" or as appropriate  
   - Input Data Field Name: `attachment_0` (binary data of first attachment)  
   - Credentials: Google Drive OAuth2 with upload permissions  
   - Connect output to "Merge Data".

5. **Create Get Required Fields Node**  
   - Type: Set  
   - Assign values:  
     - id = {{$json["id"]}}  
     - threadId = {{$json["threadId"]}}  
     - textAsHtml = {{$json["textAsHtml"]}}  
     - subject = {{$json["subject"]}}  
     - date = {{$json["date"]}}  
   - Connect output to "Merge Data".

6. **Create Extract Data from Invoice Node**  
   - Type: Extract From File  
   - Operation: PDF  
   - Binary Property Name: `attachment_0`  
   - Connect output to "Merge Data".

7. **Create Merge Data Node**  
   - Type: Merge  
   - Mode: Combine by Position  
   - Number of Inputs: 3 (from "Extract Data from Invoice", "Get Required Fields", "Upload File")  
   - Connect output to "AI Agent".

8. **Create AI Agent Node**  
   - Type: Langchain Agent  
   - Text prompt:  
     ```
     You are an expert assistant that extracts receipt fields for tax tracking.

     Here is the full receipt text:
     {{ $json.text }}

     Extract and return a single JSON object with the following fields:
     - "date": Format as YYYY-MM-DD.
     - "merchant": Name of business or company issuing the receipt. If a value is missing, return `null`.
     - "category": Infer based on context (e.g. Meals, Software, Travel, Office Supplies).
     - "description": A short summary of the service or item purchased (e.g. “Domain name renewal for 2 years”). If a value is missing, return `null`.
     - "subtotal": Value before GST. Extract only if clearly labeled. If a value is missing, return 0.
     - "tax": The Goods and Services Tax amount (labelled as GST or Tax). Return 0 if not present. If a value is missing, return 0.
     - "total": Final amount paid (after tax). If a value is missing, return 0.
     - "id": {{ $json.id }}
     - "threadId": {{ $json.threadId }}

     Respond ONLY with a JSON object.
     ```
   - Enable output parsing with the Structured Output Parser (configured next)  
   - Set retry on failure to true  
   - Connect outputs to "Append to Google Sheet" and "Mark Email as Read".

9. **Create OpenAI Chat Model Node**  
   - Type: Langchain OpenAI  
   - Model: GPT-4  
   - Credentials: Set up OpenAI API key with GPT-4 access  
   - Connect to AI Agent node's language model input.

10. **Create Structured Output Parser Node**  
    - Type: Langchain Output Parser  
    - Schema: JSON schema requiring date (string, date format), merchant (string), category (string), description (string), subtotal (number), tax (number), total (number), id (string), threadId (string)  
    - Connect to AI Agent node's output parser input.

11. **Create Append to Google Sheet Node**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: Your Google Sheet document ID  
    - Sheet Name: Your target sheet name  
    - Columns and Mappings:  
      - Date → `{{$json.output.date}}`  
      - Merchant → `{{$json.output.merchant}}`  
      - Category → `{{$json.output.category}}`  
      - Description → `{{$json.output.description}}`  
      - Subtotal → `{{$json.output.subtotal}}`  
      - Tax → `{{$json.output.tax}}`  
      - Total → `{{$json.output.total}}`  
    - Credentials: Google Sheets OAuth2 with write access  
    - Connect output to end.

12. **Create Mark Email as Read Node**  
    - Type: Gmail  
    - Operation: Mark as Read  
    - Message ID: `{{$json.output.id}}`  
    - Credentials: Gmail OAuth2  
    - Connect output to end.

13. **Add Sticky Notes** (optional but recommended for clarity)  
    - One near "Schedule Trigger" explaining it’s the daily trigger node.  
    - One near "Upload File" explaining Google Drive upload details and credential setup.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Ensure Gmail OAuth2 credentials have correct scopes for reading emails, downloading attachments, and marking as read. | Gmail API permissions                                                                                             |
| Google Drive and Sheets OAuth2 credentials require appropriate scopes to upload files and append sheet rows. | Google Cloud Console and n8n credential setup                                                                    |
| The AI Agent prompt is tailored for receipts and relies on GPT-4; adjust the prompt if your receipts vary significantly. | AI prompt customization                                                                                           |
| The Google Sheet must have columns named exactly: Date, Merchant, Category, Description, Subtotal, Tax, Total for proper mapping. | Google Sheets formatting                                                                                          |
| Folder ID and Sheet ID placeholders must be replaced with your actual Google Drive folder and Sheet document IDs. | User configuration required                                                                                       |
| The workflow is designed to process only unread emails with a specific label to prevent duplicates. Marking emails as read is critical. | Gmail label and status management                                                                                 |
| For troubleshooting rate limits or API errors, consider adding error handling or delays between API calls. | n8n workflow resiliency                                                                                           |
| This workflow can be extended to support other file types or additional data fields by modifying the extraction and AI parsing logic. | Customization and scaling                                                                                          |
| Video walk-through available here: [https://www.youtube.com/watch?v=example](https://www.youtube.com/watch?v=example) | (Replace with actual video link if available)                                                                    |

---

**Disclaimer:**  
The provided description is derived exclusively from an n8n workflow automation and complies with prevailing content policies. It contains no illegal, offensive, or protected material. All processed data is legal and publicly accessible.