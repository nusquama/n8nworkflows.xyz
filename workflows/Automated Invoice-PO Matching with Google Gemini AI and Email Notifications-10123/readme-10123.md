Automated Invoice-PO Matching with Google Gemini AI and Email Notifications

https://n8nworkflows.xyz/workflows/automated-invoice-po-matching-with-google-gemini-ai-and-email-notifications-10123


# Automated Invoice-PO Matching with Google Gemini AI and Email Notifications

### 1. Workflow Overview

This workflow automates the process of matching invoices with purchase orders (PO) using Google Gemini AI and Google Workspace tools, followed by payment approval or notification. It targets finance teams who want to streamline invoice verification, reduce manual errors, and automate notifications for mismatches.

**Logical blocks included:**

- **1.1 Input Reception & File Handling:** Detect new invoice files in Google Drive and download them.
- **1.2 Data Extraction:** Extract text from invoice PDFs and parse key invoice fields using AI.
- **1.3 Decision Making:** Categorize invoices based on total amount (above or below a threshold).
- **1.4 PO Matching via AI Agent:** Verify extracted invoice data against PO database using an AI agent.
- **1.5 Update & Approval:** Update Google Sheets with matching results and approve payments for matches.
- **1.6 Notification:** Send email alerts for invoices that do not match.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Handling

- **Overview:** Watches a specific Google Drive folder for new invoice files, downloads them for processing.
- **Nodes Involved:** Google Drive Trigger, Download file

**Node Details:**

- **Google Drive Trigger**
  - Type: Trigger node for Google Drive
  - Configuration: Watches folder with ID `1cxnSBv6t7SAgJDJZxAOtPXO0RZH7rfKf`, triggers on file creation every minute
  - Input: Triggered by new file creation in folder
  - Output: File metadata including file ID
  - Edge Cases: OAuth token expiry, folder permission issues, network latency

- **Download file**
  - Type: Google Drive node
  - Configuration: Downloads file by ID from previous node output
  - Input: File ID from Google Drive Trigger
  - Output: Binary file data (PDF invoice)
  - Edge Cases: File access denied, API rate limits, file not found

---

#### 2.2 Data Extraction

- **Overview:** Extracts text content from the PDF invoice and uses AI to extract structured invoice fields (vendor name, invoice number, dates, amounts, PO details).
- **Nodes Involved:** Extract from File, Information Extractor, Google Gemini Chat Model

**Node Details:**

- **Extract from File**
  - Type: PDF text extraction node
  - Configuration: Extracts text from the downloaded PDF
  - Input: Binary PDF data
  - Output: Extracted text content
  - Edge Cases: Unsupported PDF format, extraction failure, encrypted PDFs

- **Information Extractor**
  - Type: LangChain Information Extractor node
  - Configuration: Uses AI to parse text into structured fields including Vendor Name, Invoice Number, Invoice Date, Total Amount, PO Number, PO Date, Qty, Rate, Net Amount
  - Key Expressions: Inputs text from previous node, outputs JSON with extracted fields
  - Input: Text extracted from PDF
  - Output: JSON object of invoice fields
  - Edge Cases: AI model misunderstanding, incomplete extraction, missing required fields

- **Google Gemini Chat Model**
  - Type: Google Gemini AI language model
  - Configuration: Supports Information Extractor node with AI capabilities
  - Input: Text data
  - Output: Enhanced language understanding for extraction step
  - Credential: Google Palm API
  - Edge Cases: API quota exceeded, latency, transient errors

---

#### 2.3 Decision Making

- **Overview:** Determines invoice processing path by checking if total amount is above or below $5000.
- **Nodes Involved:** Switch

**Node Details:**

- **Switch**
  - Type: Conditional routing node
  - Configuration: Compares 'Total Amount' field extracted from invoice JSON
    - Output 'High Dollar' if amount >= 5000
    - Output 'Direct Payment' if amount < 5000
  - Input: Invoice fields JSON
  - Output: Routes to AI Agent or payment processing accordingly
  - Edge Cases: Missing or invalid 'Total Amount', type conversion errors

---

#### 2.4 PO Matching via AI Agent

- **Overview:** Validates invoice details against PO database using AI agent logic, outputs match status and remarks.
- **Nodes Involved:** AI Agent, PO_DB, Update_Row, Structured Output Parser, Google Gemini Chat Model1

**Node Details:**

- **PO_DB**
  - Type: Google Sheets node
  - Configuration: Reads PO data from a Google Sheets document (documentId `1T-H7zfvdx6NgBPrlveD8hXeW2q38VhtO_aO_2Sik1M0`, sheet "Sheet1")
  - Input: Query from AI Agent
  - Output: PO data rows
  - Edge Cases: Permission issues, sheet missing, incorrect range

- **AI Agent**
  - Type: LangChain AI Agent node
  - Configuration:
    - System message instructs AI to verify PO details and update matching status and remarks.
    - Inputs: PO Number, PO Date, Qty, Rate, Net Amount from invoice extraction.
    - Outputs structured JSON with "match" (true/false) and remarks.
  - Input: Data from Switch node and PO_DB
  - Output: Match result and remarks
  - Credential: Google Palm API (via Google Gemini Chat Model1)
  - Edge Cases: AI misinterpretation, API limits, invalid inputs

- **Structured Output Parser**
  - Type: LangChain output parser node
  - Configuration: Parses AI Agent's structured JSON response for match and remarks
  - Input: Raw AI output
  - Output: Clean JSON object with match status and remarks
  - Edge Cases: Parsing errors if AI output deviates from schema

- **Update_Row**
  - Type: Google Sheets node
  - Configuration: Updates PO sheet row matching the PO Number with invoice number, match status, and remarks
  - Input: AI Agent output data
  - Output: Updated spreadsheet row confirmation
  - Edge Cases: Row not found, API write failure, concurrency conflicts

- **Google Gemini Chat Model1**
  - Type: Language Model node as AI Agent LLM backend
  - Credential: Google Palm API
  - Input/Output: Linked to AI Agent for natural language processing

---

#### 2.5 Update & Approval

- **Overview:** If invoice matches PO, appends a new row to "Payment Process" Google Sheet marking status as "Process payment."
- **Nodes Involved:** If, Append row in sheet

**Node Details:**

- **If**
  - Type: Conditional node
  - Configuration: Checks if AI Agent output ‘match’ field equals "true"
  - Input: AI Agent JSON output
  - Output: Routes to append row for matches or send email for mismatches
  - Edge Cases: Missing match field, misformatted input

- **Append row in sheet**
  - Type: Google Sheets node
  - Configuration: Appends invoice and PO details with status "Process payment" to a separate Google Sheet (documentId `1km2dOTGoExdJjaQvXYMlz80IvnCHSAPY6kL1AavomUo`, sheet "Sheet1")
  - Input: Invoice and PO fields from Switch node
  - Output: Confirmation of row appended
  - Edge Cases: API limits, sheet permissions, data mapping errors

---

#### 2.6 Notification

- **Overview:** For invoices that do not match, sends an email notification to the finance team with invoice and PO info and remarks.
- **Nodes Involved:** Send a message

**Node Details:**

- **Send a message**
  - Type: Microsoft Outlook email node
  - Configuration:
    - Subject includes invoice number and note about mismatch
    - Body details invoice number, PO number, and AI-generated remarks
    - Recipient: abdul.matheen@teamhgs.com
  - Input: AI Agent output from If node mismatch path
  - Output: Email sent confirmation
  - Credential: Microsoft Outlook OAuth2 API
  - Edge Cases: Email delivery failure, OAuth token expiry, invalid email addresses

---

### 3. Summary Table

| Node Name              | Node Type                                   | Functional Role                       | Input Node(s)              | Output Node(s)                  | Sticky Note                                               |
|------------------------|---------------------------------------------|------------------------------------|----------------------------|-------------------------------|-----------------------------------------------------------|
| Google Drive Trigger    | Google Drive Trigger                         | Detect new invoice files            | -                          | Download file                  | Invoice PO matching process overview                       |
| Download file          | Google Drive                                | Download invoice file               | Google Drive Trigger        | Extract from File              | Extract the invoice data and extract required info         |
| Extract from File      | Extract from File                           | Extract text from PDF               | Download file              | Information Extractor          | Extract the invoice data and extract required info         |
| Information Extractor  | LangChain Information Extractor             | Parse structured invoice fields    | Extract from File          | Switch                        | Extract the invoice data and extract required info         |
| Google Gemini Chat Model| LangChain Language Model (Google Gemini)   | AI support for extraction           | -                          | Information Extractor          | Extract the invoice data and extract required info         |
| Switch                 | Switch                                     | Route based on invoice amount       | Information Extractor      | AI Agent, Append row in sheet  | Check the amount is greater than 5000 and validate PO DB   |
| AI Agent               | LangChain AI Agent                          | Match invoice with PO DB            | Switch, PO_DB             | If                           |                                                            |
| PO_DB                  | Google Sheets                              | Read PO database                   | AI Agent                   | AI Agent                      |                                                            |
| Update_Row             | Google Sheets                              | Update PO sheet with match status  | AI Agent                   | AI Agent                      |                                                            |
| Structured Output Parser| LangChain Output Parser                     | Parse AI agent output               | AI Agent                   | AI Agent                      |                                                            |
| Google Gemini Chat Model1| LangChain Language Model (Google Gemini)  | AI backend for AI Agent             | -                          | AI Agent                      |                                                            |
| If                     | If                                         | Branch on match result              | AI Agent                   | Append row in sheet, Send a message | Approval for payment process                               |
| Append row in sheet    | Google Sheets                              | Append payment approval entry      | If                         | -                             | Approval for payment process                               |
| Send a message         | Microsoft Outlook                          | Send mismatch notification email   | If                         | -                             | Send email if not match                                   |
| Sticky Note            | Sticky Note                               | Documentation comments              | -                          | -                             | Invoice PO matching process overview                       |
| Sticky Note1           | Sticky Note                               | Documentation comments              | -                          | -                             | Extract the invoice data and extract required info         |
| Sticky Note2           | Sticky Note                               | Documentation comments              | -                          | -                             | Check the amount is greater than 5000 and validate PO DB   |
| Sticky Note3           | Sticky Note                               | Documentation comments              | -                          | -                             | Approval for payment process                               |
| Sticky Note4           | Sticky Note                               | Documentation comments              | -                          | -                             | Send email if not match                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**
   - Type: Google Drive Trigger
   - Trigger on: File created in a specific folder
   - Folder ID: `1cxnSBv6t7SAgJDJZxAOtPXO0RZH7rfKf`
   - Poll interval: Every minute
   - Credentials: Google Drive OAuth2

2. **Add Google Drive node (Download file):**
   - Operation: Download
   - File ID: `={{ $json.id }}`
   - Connect from Google Drive Trigger
   - Credentials: Same Google Drive OAuth2

3. **Add Extract from File node:**
   - Operation: PDF text extraction
   - Input: Binary data from Download file node
   - Connect from Download file

4. **Add Information Extractor node:**
   - Use LangChain Information Extractor
   - Input Text: `={{ $json.text }}`
   - Define attributes:
     - Vendor Name (required)
     - Invoice Number (required)
     - Invoice Date (required)
     - Total Amount (number, required)
     - PO Number (required)
     - PO Date (required)
     - Qty (number, required)
     - Rate (number, required)
     - Net Amount (required)
   - Connect from Extract from File

5. **Add Google Gemini Chat Model node:**
   - Credential: Google Palm API
   - Connect as AI language model for Information Extractor node

6. **Add Switch node:**
   - Condition 1 (output key: "High Dollar"): Total Amount >= 5000
   - Condition 2 (output key: "Direct Payment"): Total Amount < 5000
   - Connect from Information Extractor

7. **Add Google Sheets node (PO_DB):**
   - Operation: Read PO data
   - Document ID: `1T-H7zfvdx6NgBPrlveD8hXeW2q38VhtO_aO_2Sik1M0`
   - Sheet Name: `Sheet1` (gid=0)
   - Credentials: Google Sheets OAuth2
   - Connect from AI Agent (see next step)

8. **Add AI Agent node:**
   - Prompt includes PO Number, PO Date, Qty, Rate, Net Amount from invoice
   - System message instructs verifying PO data, updating matching status and remarks
   - Output: JSON with `match` (true/false) and `remarks`
   - Connect main input from Switch node (High Dollar path)
   - Connect ai_tool input from PO_DB node
   - Credentials: Google Palm API (via Google Gemini Chat Model1)

9. **Add Google Gemini Chat Model1 node:**
   - Credential: Google Palm API
   - Connect as AI language model for AI Agent node

10. **Add Structured Output Parser node:**
    - JSON schema example:
      ```
      {
        "match": "true",
        "remarks": "Invoice matched"
      }
      ```
    - Connect from AI Agent's ai_outputParser output

11. **Add Google Sheets node (Update_Row):**
    - Operation: Update row in PO sheet
    - Match by PO Number column
    - Update columns:
      - Remarks: from AI output
      - Invoice Number: from AI output
      - Matching or Not Matching: from AI output
    - Connect ai_tool input from AI Agent node

12. **Add If node:**
    - Condition: `{{ $json.output.match }} == "true"`
    - Connect main input from AI Agent node

13. **Add Google Sheets node (Append row in sheet):**
    - Operation: Append row
    - Document ID: `1km2dOTGoExdJjaQvXYMlz80IvnCHSAPY6kL1AavomUo`
    - Sheet Name: `Sheet1`
    - Columns:
      - PO Number, Vendor Name, PO Date, Qty, Rate, Amount, Tax (0), Total Amount, Invoice Number, Status ("Process payment")
    - Connect from If node's True output

14. **Add Microsoft Outlook node (Send a message):**
    - Send email with subject about invoice mismatch
    - Body includes invoice number, PO number, remarks
    - Recipient: abdul.matheen@teamhgs.com
    - Connect from If node's False output
    - Credentials: Microsoft Outlook OAuth2

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                                                       |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Invoice PO matching process: Extract data, AI parse, amount check, review, match update, notify | Workflow purpose overview                                                                                                           |
| Extract the invoice data and extract required information from invoice                           | Sticky note emphasizing data extraction importance                                                                                  |
| Check the amount is greater than 5000 and validated with PO database                            | Sticky note clarifying amount-based routing logic                                                                                   |
| Approval for payment process                                                                    | Sticky note near approval-related nodes                                                                                            |
| Send email if not match                                                                         | Sticky note near email notification node                                                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.