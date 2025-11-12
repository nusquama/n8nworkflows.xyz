Compare Vendor Quotations with Grok AI and Export to Google Sheets & Email

https://n8nworkflows.xyz/workflows/compare-vendor-quotations-with-grok-ai-and-export-to-google-sheets---email-6688


# Compare Vendor Quotations with Grok AI and Export to Google Sheets & Email

---

### 1. Workflow Overview

This workflow automates the process of comparing vendor quotations using AI-powered summarization, logging the results in Google Sheets, and emailing a summary to relevant stakeholders. It is designed for procurement teams or project managers who receive multiple vendor quotes in various file formats and want to efficiently generate a structured comparison with AI assistance.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Data Extraction:** Accept uploaded quote files via webhook and extract usable data from them.
- **1.2 AI Processing & Summarization:** Send extracted data to an AI service (Grok) to generate a summarized comparison.
- **1.3 Synchronization & Formatting:** Wait for AI response, format the summary into structured data.
- **1.4 Data Logging & Notification:** Append the formatted summary to a Google Sheet and send an email notification with the summary.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Extraction

**Overview:**  
This block receives vendor quotation files through an HTTP webhook, then extracts essential data from these files to prepare for AI processing.

**Nodes Involved:**  
- Upload Quotes (Webhook)  
- Extract File Data (Function)

**Node Details:**  

- **Upload Quotes**  
  - *Type:* Webhook Trigger  
  - *Role:* Entry point that triggers when vendors upload quote files.  
  - *Configuration:* Listens on path `/vendor-quote-upload`, accepting files such as PDFs or Excel documents through HTTP POST.  
  - *Input/Output:* Receives binary file data; outputs raw file data.  
  - *Edge Cases:* File size limits, unsupported file formats, missing file data, malformed HTTP requests.  
  - *Notes:* This node expects valid file uploads; no authentication configured here by default.

- **Extract File Data**  
  - *Type:* Function (Code Node)  
  - *Role:* Parses the uploaded file binary data to extract filename and raw binary content for AI processing.  
  - *Configuration:* Custom JavaScript function that maps over input items extracting `fileName` and `fileData` from binary `data` field.  
  - *Key Expressions:* Reads `item.binary['data']` for file details.  
  - *Input/Output:* Input from webhook; outputs JSON-formatted file metadata and content.  
  - *Edge Cases:* Missing `data` binary property, malformed or empty files, potential for unsupported file format needing further parsing outside this node.

---

#### 1.2 AI Processing & Summarization

**Overview:**  
This block sends extracted file data to the Grok AI summarization API, requesting a concise, structured summary of the vendor quotations.

**Nodes Involved:**  
- AI Summarization (HTTP Request)  
- Wait For Reply (Wait Node)

**Node Details:**  

- **AI Summarization**  
  - *Type:* HTTP Request  
  - *Role:* Calls the external Grok AI API endpoint to generate a summary of the quotation files.  
  - *Configuration:* POST request to `https://api.grok.xai.com/summarize` with extracted file data as payload (exact body not visible, presumed).  
  - *Input/Output:* Input JSON includes file data; output is AI-generated summary JSON.  
  - *Edge Cases:* API authentication failures (if any), network timeouts, invalid response formats, rate limiting from Grok API, malformed request body.  
  - *Version Notes:* Requires n8n HTTP Request node version supporting POST and JSON payloads.

- **Wait For Reply**  
  - *Type:* Wait Node  
  - *Role:* Pauses workflow execution until the AI response is ready, ensuring downstream nodes process valid AI output.  
  - *Configuration:* Uses webhook ID to resume workflow upon AI response reception or timeout.  
  - *Input/Output:* Input from HTTP Request; output triggers Format Summary.  
  - *Edge Cases:* Timeout if AI response delayed or not received, webhook misconfiguration, duplicate triggers.

---

#### 1.3 Synchronization & Formatting

**Overview:**  
Formats the raw AI-generated summary into a clean, structured object representing key vendor quote attributes for logging and emailing.

**Nodes Involved:**  
- Format Summary (Function)

**Node Details:**  

- **Format Summary**  
  - *Type:* Function (Code Node)  
  - *Role:* Parses AI summary JSON, normalizing fields like vendor name, price, delivery time, and features into consistent JSON structure.  
  - *Configuration:* JavaScript function mapping items, setting defaults if fields are missing (`vendor: 'Unknown'`, `price: 0`, etc.).  
  - *Key Expressions:* Accesses properties `summary.vendor`, `summary.price`, `summary.deliveryTime`, `summary.features`.  
  - *Input/Output:* Input AI JSON; outputs cleaned JSON summary objects.  
  - *Edge Cases:* Missing or malformed AI response fields, empty features array, inconsistent data types.

---

#### 1.4 Data Logging & Notification

**Overview:**  
Logs the formatted summary to a Google Sheet for record-keeping and sends an email notification with the summary details to stakeholders.

**Nodes Involved:**  
- Log to Google Sheets (Google Sheets)  
- Send email (Email Send)

**Node Details:**  

- **Log to Google Sheets**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends the summarized data as a new row in a specified Google Sheet tab `QuotationSummary!A:D`.  
  - *Configuration:* Uses Google API credentials; operation set to "append"; target sheet ID must be configured.  
  - *Input/Output:* Takes formatted JSON summary as input; no output required beyond success signal.  
  - *Edge Cases:* Credential expiration, insufficient permissions on sheet, invalid sheet ID, quota limits, data format mismatches.  
  - *Credentials:* Requires valid OAuth2 Google API credentials with Sheets access.

- **Send email**  
  - *Type:* Email Send Node  
  - *Role:* Sends an email with the vendor quotation summary to a fixed recipient list.  
  - *Configuration:* SMTP credentials configured; email subject fixed as "Vendor Quotation Comparison"; recipient email is `team@gmail.com`; sender is `admin@gmail.com`; email format is text.  
  - *Key Expressions:* The email text references the summary fields but appears to contain an expression snippet that may not render properly (`json: { vendor: ... }` inside text). Ideally, this should be a template or concatenation for readability.  
  - *Input/Output:* Input is formatted summary JSON; no output.  
  - *Edge Cases:* SMTP authentication failures, invalid recipient addresses, email quota limits, malformed email content expressions.  
  - *Credentials:* Requires functional SMTP credentials.

---

### 3. Summary Table

| Node Name        | Node Type           | Functional Role                                  | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                     |
|------------------|---------------------|-------------------------------------------------|------------------------|------------------------|-----------------------------------------------------------------------------------------------------------------|
| Upload Quotes    | Webhook Trigger     | Receives uploaded vendor quote files             | -                      | Extract File Data       | Triggers when vendor quote files are uploaded (via webhook). Accepts PDFs, Excel, or other docs.                |
| Extract File Data | Function            | Extracts filename and file data from upload      | Upload Quotes           | AI Summarization        | Parses and extracts key data from uploaded quote files. Converts file contents to structured text or JSON.     |
| AI Summarization | HTTP Request        | Sends extracted data to Grok AI for summarization | Extract File Data       | Wait For Reply          | Sends extracted data to an AI model to generate a comparison summary using Grok or another LLM.                 |
| Wait For Reply   | Wait Node           | Waits for AI response before proceeding          | AI Summarization        | Format Summary          | Waits for the AI response before proceeding to ensure the summary is ready.                                    |
| Format Summary   | Function            | Formats AI response into structured summary      | Wait For Reply          | Log to Google Sheets    | Cleans and formats the AI response into a readable comparison table. Converts raw AI output into structured data. |
| Log to Google Sheets | Google Sheets    | Appends summary data to Google Sheet              | Format Summary          | Send email              | Appends the formatted summary into a Google Sheet for tracking vendor comparisons.                             |
| Send email       | Email Send          | Emails the formatted summary to stakeholders      | Log to Google Sheets    | -                      | Emails decision-ready vendor quotation summary to the procurement or stakeholder team.                         |
| Sticky Note      | Sticky Note         | Provides minimal Google Sheet columns reference   | -                      | -                      | Minimal Google Sheet Columns (Top 5): Vendor Name, Total Amount, Delivery Timeline, AI Summary, Status          |
| Sticky Note1     | Sticky Note         | Explains notes for each node in the workflow      | -                      | -                      | Detailed notes describing each node’s role and configuration within the workflow.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Upload Quotes":**  
   - Type: Webhook Trigger  
   - Path: `/vendor-quote-upload`  
   - Accept file uploads (binary data) via HTTP POST.  
   - No authentication configured by default.

2. **Add Function Node "Extract File Data":**  
   - Connect input from `Upload Quotes`.  
   - Code snippet:  
     ```javascript
     return items.map(item => {
       const file = item.binary['data'];
       return {
         json: {
           fileName: file.fileName,
           fileData: file.data,
         },
       };
     });
     ```  
   - Purpose: Extract filename and binary content from uploaded files.

3. **Insert HTTP Request Node "AI Summarization":**  
   - Connect input from `Extract File Data`.  
   - Method: POST  
   - URL: `https://api.grok.xai.com/summarize`  
   - Body: Include extracted file data in JSON format (adapt as per Grok API docs).  
   - Headers: Set Content-Type to `application/json`.  
   - Handle authentication if required by Grok API.  
   - Ensure proper error handling and timeouts.

4. **Add Wait Node "Wait For Reply":**  
   - Connect input from `AI Summarization`.  
   - Use webhook ID to pause workflow until AI response is ready.  
   - No parameters; default wait behavior.

5. **Create Function Node "Format Summary":**  
   - Connect input from `Wait For Reply`.  
   - Code snippet:  
     ```javascript
     return items.map(item => {
       const summary = item.json;
       return {
         json: {
           vendor: summary.vendor || 'Unknown',
           price: summary.price || 0,
           deliveryTime: summary.deliveryTime || 'N/A',
           features: summary.features || [],
         },
       };
     });
     ```  
   - Purpose: Normalize AI output into consistent fields.

6. **Add Google Sheets Node "Log to Google Sheets":**  
   - Connect input from `Format Summary`.  
   - Operation: Append  
   - Sheet ID: Set your target Google Sheet ID.  
   - Range: `QuotationSummary!A:D` (or modify as needed).  
   - Credentials: Configure with Google API OAuth2 credentials having Sheets access.  
   - Map node input fields to corresponding columns.

7. **Add Email Send Node "Send email":**  
   - Connect input from `Log to Google Sheets`.  
   - SMTP credentials: Configure with valid SMTP credentials.  
   - From Email: `admin@gmail.com` (replace as needed).  
   - To Email: `team@gmail.com` (replace as needed).  
   - Subject: `Vendor Quotation Comparison`  
   - Email Text: Format the summary info into a readable string, e.g.:  
     ```
     Vendor: {{$json["vendor"]}}
     Price: {{$json["price"]}}
     Delivery Time: {{$json["deliveryTime"]}}
     Features: {{$json["features"].join(", ")}}
     ```  
   - Email Format: Plain text or HTML as preferred.

8. **Optional: Add Sticky Notes for Documentation:**  
   - Add sticky notes to document minimal Google Sheet columns and detailed node notes for team reference.

9. **Activate and Test Workflow:**  
   - Deploy the webhook URL.  
   - Upload test vendor quote files.  
   - Verify AI summarization, Google Sheets logging, and email notifications.  
   - Monitor for errors and adjust credentials and API parameters as necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Minimal Google Sheet Columns (Top 5): Vendor Name, Total Amount, Delivery Timeline, AI Summary, Status         | See Sticky Note node content for recommended columns                                           |
| Detailed node descriptions and workflow explanations provided in Sticky Note1 node                             | Useful for onboarding new users and maintaining the workflow                                   |
| Grok AI Summarization API endpoint: https://api.grok.xai.com/summarize                                        | Ensure correct API usage and credentials per Grok documentation                               |
| Google Sheets API OAuth2 credentials required for appending data                                              | Set up via Google Cloud Console with Sheets API enabled                                       |
| SMTP credentials needed for sending emails (supports common providers)                                        | Configure SMTP node with valid credentials for reliable email delivery                         |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---