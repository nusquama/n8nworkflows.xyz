Extract and Organize Receipt Data for Expense Tracking with VLM Run and Google

https://n8nworkflows.xyz/workflows/extract-and-organize-receipt-data-for-expense-tracking-with-vlm-run-and-google-5051


# Extract and Organize Receipt Data for Expense Tracking with VLM Run and Google

---

### 1. Workflow Overview

This workflow automates the extraction and organization of receipt data to streamline expense tracking. It continuously monitors a specific Google Drive folder for new receipt uploads (images or PDFs), applies AI-powered data extraction using the VLM Run service, formats the extracted information into a structured form, and appends the data into a Google Sheets document for centralized expense management.

**Logical Blocks:**

- **1.1 Input Reception & Monitoring:** Watches a designated Google Drive folder for new receipt files and downloads them.
- **1.2 AI Data Extraction:** Processes downloaded files with VLM Run to extract key receipt details such as merchant name, amount, currency, and date.
- **1.3 Data Formatting and Storage:** Structures the extracted data and appends it to a Google Sheets expense database.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Monitoring

**Overview:**  
This block detects newly uploaded receipt files in a monitored Google Drive folder and downloads them for processing.

**Nodes Involved:**  
- Monitor Receipt Uploads  
- Download Receipt File

---

##### Node: Monitor Receipt Uploads

- **Type & Role:** Google Drive Trigger node; initiates workflow on new file creation.
- **Configuration:**  
  - Trigger event: `fileCreated`  
  - Poll interval: every minute  
  - Watches a specific Google Drive folder (folder ID: `1reWORwI1tMa-eGB75NCXq9eRw4CiQIhX`)  
- **Expressions/Variables:** Uses the folder ID to watch specific folder.
- **Input/Output Connections:** No input nodes; outputs to `Download Receipt File`.
- **Version Requirements:** n8n Google Drive Trigger node, version 1 or higher.
- **Edge Cases / Potential Failures:**  
  - Authentication failure with Google Drive OAuth2.  
  - API rate limits if many files uploaded rapidly.  
  - Folder permission issues.
- **Sub-Workflow References:** None.

---

##### Node: Download Receipt File

- **Type & Role:** Google Drive node; downloads the actual receipt files identified by the trigger.
- **Configuration:**  
  - Operation: `download`  
  - File ID dynamically set from trigger output (`{{$json.id}}`)  
  - Stores file binary data in property `data`.  
- **Expressions/Variables:** Uses dynamic expression to target specific file from trigger.
- **Input/Output Connections:** Input from `Monitor Receipt Uploads`; outputs to `VLM Run Receipt Parser`.
- **Version Requirements:** Google Drive node version 3 or higher.
- **Edge Cases / Potential Failures:**  
  - File not found or deleted before download.  
  - Unsupported file format or corrupted file.  
  - Download timeout or connectivity issues.
- **Sub-Workflow References:** None.

---

#### 1.2 AI Data Extraction

**Overview:**  
This block sends the downloaded receipt file to the VLM Run AI service to extract structured receipt information, including merchant, amount, currency, and date.

**Nodes Involved:**  
- VLM Run Receipt Parser

---

##### Node: VLM Run Receipt Parser

- **Type & Role:** VLM Run node for AI-powered document parsing.
- **Configuration:**  
  - Domain: `document.receipt` (specifies receipt extraction model)  
  - Uses binary data from previous node.  
- **Expressions/Variables:** None besides domain selection.
- **Input/Output Connections:** Input from `Download Receipt File`; outputs to `Format Receipt Data`.
- **Version Requirements:** VLM Run node integration; requires valid VLM Run API credentials.
- **Edge Cases / Potential Failures:**  
  - API authentication failure.  
  - Poor image quality causing extraction errors.  
  - Unexpected receipt formats or missing fields.  
  - API rate limits or downtime.
- **Sub-Workflow References:** None.

---

#### 1.3 Data Formatting and Storage

**Overview:**  
This block formats the raw AI data into a consistent structure and appends the cleaned data as a new entry into a Google Sheets document for expense tracking.

**Nodes Involved:**  
- Format Receipt Data  
- Save to Expense Database

---

##### Node: Format Receipt Data

- **Type & Role:** Set node; re-maps and cleans extracted AI data into defined fields.
- **Configuration:**  
  - Keeps only fields: Customer, Merchant, Amount, Currency, Date.  
  - Extracted from JSON response paths:  
    - `Customer` = `{{$json.response.customer_name}}`  
    - `Merchant` = `{{$json.response.merchant_name}}`  
    - `Amount` = `{{$json.response.total}}`  
    - `Currency` = `{{$json.response.currency}}`  
    - `Date` = `{{$json.response.transaction_date}}`  
- **Expressions/Variables:** Uses JSONPath expressions to extract data.
- **Input/Output Connections:** Input from `VLM Run Receipt Parser`; outputs to `Save to Expense Database`.
- **Version Requirements:** n8n Set node version 1 or higher.
- **Edge Cases / Potential Failures:**  
  - Missing or null fields in AI response.  
  - Data type mismatches or formatting inconsistencies.
- **Sub-Workflow References:** None.

---

##### Node: Save to Expense Database

- **Type & Role:** Google Sheets node; appends formatted receipt data as a new row.
- **Configuration:**  
  - Operation: `append`  
  - Document ID: Google Sheets document ID (`11_VjMdhv_JN2eSRZiw_t0dIN-yShkn2jlCDwiG8eb14`)  
  - Sheet name: `Sheet1` (gid=0)  
  - Columns mapped from formatted data fields: Customer, Merchant, Amount, Currency, Date.  
  - Matching Columns: Customer (used for potential matching but append mode disables replacing).  
  - No automatic type conversion or field to string conversion enabled.  
- **Expressions/Variables:** Columns mapped directly from Set node output.
- **Input/Output Connections:** Input from `Format Receipt Data`; no output as this is an endpoint.
- **Version Requirements:** Google Sheets node version 4.6 or higher.
- **Edge Cases / Potential Failures:**  
  - Google Sheets API quota limits or permission errors.  
  - Schema mismatch if sheet columns are changed externally.  
  - Network or OAuth token expiration issues.
- **Sub-Workflow References:** None.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                           | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                   |
|-------------------------|----------------------------|-----------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| 🧾 Workflow Overview    | Sticky Note                | Workflow high-level explanation         | -                      | -                        | Overview of workflow purpose, features, and requirements                                                     |
| 📁 Input Processing Documentation | Sticky Note                | Explains input monitoring process       | -                      | -                        | Description of Google Drive folder monitoring and supported receipt formats                                  |
| 🤖 AI Extraction Documentation | Sticky Note                | Explains AI extraction details           | -                      | -                        | Details on VLM Run AI extraction capabilities                                                                |
| 📊 Storage Documentation | Sticky Note                | Explains data storage in Google Sheets  | -                      | -                        | Notes on data structure and benefits of Google Sheets storage                                                |
| Monitor Receipt Uploads  | Google Drive Trigger       | Triggers workflow on new receipt upload | -                      | Download Receipt File     | Monitors Google Drive folder for new receipt uploads and triggers processing automatically                    |
| Download Receipt File    | Google Drive               | Downloads receipt files                  | Monitor Receipt Uploads | VLM Run Receipt Parser   | Downloads receipt files from Google Drive for AI processing                                                  |
| VLM Run Receipt Parser  | VLM Run Node               | AI extraction of receipt data            | Download Receipt File   | Format Receipt Data       | Uses VLM AI to extract merchant name, amount, currency, and date from receipts                               |
| Format Receipt Data      | Set Node                  | Formats AI response into structured data| VLM Run Receipt Parser | Save to Expense Database  | Transforms AI-extracted receipt data into clean, structured format for spreadsheet storage                   |
| Save to Expense Database | Google Sheets              | Saves formatted data to Google Sheets   | Format Receipt Data     | -                        | Automatically saves extracted receipt data to Google Sheets for expense tracking                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation:**  
   - Add four Sticky Note nodes with the content describing workflow overview, input processing, AI extraction, and data storage.  
   - Position them for readability (optional).

2. **Set Up Google Drive Trigger Node:**  
   - Add a **Google Drive Trigger** node named `Monitor Receipt Uploads`.  
   - Set event to `fileCreated`.  
   - Configure polling to run every minute.  
   - Select "Trigger on specific folder" and specify the target folder by its folder ID (e.g., `1reWORwI1tMa-eGB75NCXq9eRw4CiQIhX`).  
   - Attach Google Drive OAuth2 credentials with appropriate permissions.

3. **Add Google Drive Node to Download Files:**  
   - Add a **Google Drive** node named `Download Receipt File`.  
   - Set operation to `download`.  
   - For file ID, use expression: `{{$json.id}}` from trigger output.  
   - Set binary property name to `data`.  
   - Connect `Monitor Receipt Uploads` output to this node’s input.  
   - Use the same Google Drive OAuth2 credentials.

4. **Add VLM Run Node for Receipt Parsing:**  
   - Add a **VLM Run** node named `VLM Run Receipt Parser`.  
   - Set domain to `document.receipt` to specify receipt extraction.  
   - Ensure it is configured to use binary data input (property `data`).  
   - Connect `Download Receipt File` output to this node’s input.  
   - Attach VLM Run API credentials.

5. **Add Set Node to Format Extracted Data:**  
   - Add a **Set** node named `Format Receipt Data`.  
   - Enable "Keep Only Set" option to ensure only specified fields pass forward.  
   - Define string fields with the following values:  
     - Customer: `{{$json.response.customer_name}}`  
     - Merchant: `{{$json.response.merchant_name}}`  
     - Amount: `{{$json.response.total}}`  
     - Currency: `{{$json.response.currency}}`  
     - Date: `{{$json.response.transaction_date}}`  
   - Connect output of `VLM Run Receipt Parser` to this node.

6. **Add Google Sheets Node to Append Data:**  
   - Add a **Google Sheets** node named `Save to Expense Database`.  
   - Set operation to `append`.  
   - Specify the Google Sheets document ID (e.g., `11_VjMdhv_JN2eSRZiw_t0dIN-yShkn2jlCDwiG8eb14`).  
   - Specify the target sheet name or gid (e.g., `Sheet1` or `gid=0`).  
   - Define columns mapping as follows:  
     - Customer: `{{$json.Customer}}`  
     - Merchant: `{{$json.Merchant}}`  
     - Amount: `{{$json.Amount}}`  
     - Currency: `{{$json.Currency}}`  
     - Date: `{{$json.Date}}`  
   - Use Google Sheets OAuth2 credentials with write access.  
   - Connect output of `Format Receipt Data` to this node.

7. **Save and Activate Workflow:**  
   - Review all credentials and permissions.  
   - Enable the workflow to auto-trigger on receipt uploads.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                                        |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates receipt processing using Google Drive trigger, VLM Run AI for data extraction, and Google Sheets for storage.| Project summary from workflow sticky notes.                                                                                           |
| Requires VLM Run API credentials with access to receipt extraction domain (`document.receipt`).                                 | VLM Run API Documentation: https://vlm.run/docs                                                                                      |
| Google Drive and Google Sheets credentials require OAuth2 setup with appropriate scopes for file monitoring, download, and edit.| Google API OAuth2 guide: https://developers.google.com/identity/protocols/oauth2                                                        |
| Supports receipt image formats: JPG, PNG, WEBP, and PDF documents.                                                             | Mentioned in input processing sticky note.                                                                                            |
| Designed for business expense reporting, personal finance tracking, and accounting automation.                                 | Use cases highlighted in workflow overview sticky note.                                                                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---