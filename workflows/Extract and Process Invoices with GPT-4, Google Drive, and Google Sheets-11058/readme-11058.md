Extract and Process Invoices with GPT-4, Google Drive, and Google Sheets

https://n8nworkflows.xyz/workflows/extract-and-process-invoices-with-gpt-4--google-drive--and-google-sheets-11058


# Extract and Process Invoices with GPT-4, Google Drive, and Google Sheets

### 1. Workflow Overview

This workflow, titled **"Extract and Process Invoices with GPT-4, Google Drive, and Google Sheets"**, automates the processing of incoming invoice PDFs stored in a Google Drive input folder. It extracts invoice data using AI (GPT-4), matches invoices to booking accounts from a Google Sheet, saves the processed invoice PDF into a designated Google Drive folder, and updates a Google Sheets booking list accordingly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Preparation**: Watches a specific Google Drive folder for new invoice PDFs, downloads them, and extracts text content.
- **1.2 AI Processing and Data Extraction**: Uses GPT-4-based AI agents to interpret extracted text, parse structured invoice data, and assign correct booking accounts.
- **1.3 Data Post-Processing and Output Handling**: Saves processed PDFs to designated folders, updates the booking list in Google Sheets, and moves the original invoice file to an output folder for processed invoices.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and File Preparation

**Overview:**  
This block triggers the workflow on new invoice PDFs uploaded to a monitored Google Drive folder, downloads the file, and extracts its textual content and binary data for further processing.

**Nodes Involved:**  
- Google Drive Trigger  
- Download file  
- Extract from File  

**Node Details:**

- **Google Drive Trigger**  
  - *Type:* Trigger node monitoring Google Drive for new files.  
  - *Configuration:* Watches the specific folder "Input AI Invoice Agent" by folder ID. Polls every minute for new files created.  
  - *Inputs:* None (trigger).  
  - *Outputs:* Emits new file metadata including file ID.  
  - *Failure Modes:* Possible auth errors, folder permission issues, API rate limits.  
  - *Credentials:* Google Drive OAuth2.  

- **Download file**  
  - *Type:* Google Drive node for file download.  
  - *Configuration:* Downloads the file using file ID from the trigger node. Delivers binary data property named "data".  
  - *Inputs:* File ID from Google Drive Trigger.  
  - *Outputs:* File binary data and metadata.  
  - *Failure Modes:* File not found if deleted before download, permissions.  
  - *Credentials:* Google Drive OAuth2.  

- **Extract from File**  
  - *Type:* File extraction node, configured for PDF.  
  - *Configuration:* Extracts text and keeps source binary data from the downloaded PDF binary property "data".  
  - *Inputs:* Binary data from downloaded file.  
  - *Outputs:* Extracted text content and original binary content.  
  - *Failure Modes:* PDF extraction failures on malformed or encrypted PDFs.  

---

#### 1.2 AI Processing and Data Extraction

**Overview:**  
This block uses a GPT-4 powered AI agent to parse the extracted invoice text, identify relevant invoice details, and determine the proper booking account using a Google Sheets-based chart of accounts. It then structures and validates output data.

**Nodes Involved:**  
- OpenAI Chat Model  
- Chart of Accounts (Google Sheets)  
- Simple Memory (Buffer Window)  
- Invoice Agent (Langchain Agent)  
- Structured Output Parser  

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* Language model node using OpenAI GPT-4 (gpt-4o-mini).  
  - *Configuration:* Model set to GPT-4 mini variant without additional options.  
  - *Inputs:* Triggered by the Invoice Agent node as the language model backend.  
  - *Outputs:* AI-generated chat completions.  
  - *Failure Modes:* API quota limits, auth errors, latency.  
  - *Credentials:* OpenAI API key.  

- **Chart of Accounts**  
  - *Type:* Google Sheets Tool node.  
  - *Configuration:* Reads booking accounts from a specific Google Sheet tab ("booking accounts" sheet). Used as a tool by the AI agent for account matching.  
  - *Inputs:* None directly; AI agent uses it as a reference tool.  
  - *Outputs:* Data passed to AI agent as tool reference.  
  - *Failure Modes:* Sheet access issues, permission errors.  
  - *Credentials:* Google Sheets OAuth2.  

- **Simple Memory**  
  - *Type:* Langchain memory buffer node.  
  - *Configuration:* Uses custom session key based on the Google Drive file ID to maintain context across AI interactions.  
  - *Inputs:* AI conversation context.  
  - *Outputs:* Maintains session memory for the Invoice Agent.  
  - *Failure Modes:* Session key misconfiguration, memory overflow.  

- **Invoice Agent**  
  - *Type:* Langchain agent node integrating AI language model with tool and memory.  
  - *Configuration:*  
    - Text prompt prepends "Invoice: " plus extracted text.  
    - System message instructs the assistant to extract invoice info and use the "booking accounts" tool for matching accounts.  
    - Uses structured output parser to enforce output schema.  
    - On error, continues with normal output (avoids hard stops).  
  - *Inputs:* Extracted text from "Extract from File" node, AI model and tools, memory context.  
  - *Outputs:* Parsed structured invoice data ready for next steps.  
  - *Failure Modes:* AI parsing errors, tool data inconsistencies, schema validation failures.  

- **Structured Output Parser**  
  - *Type:* Langchain output parser node.  
  - *Configuration:* Manual JSON schema defining expected invoice fields: invoice dates (two formats), booking account code, vendor, amounts, currency, booking text, invoice number, folder ID.  
  - *Inputs:* Raw AI output from Invoice Agent.  
  - *Outputs:* Validated structured data object with invoice details.  
  - *Failure Modes:* Parsing failures if AI output does not match schema or missing required fields ("konto" is required but appears as "bookingAccount" in other places - possible mismatch to be checked).  

---

#### 1.3 Data Post-Processing and Output Handling

**Overview:**  
This block saves the processed invoice PDF to a target Google Drive folder (based on booking account), updates the booking list in Google Sheets with extracted invoice details, and moves the original file to an output folder.

**Nodes Involved:**  
- Edit Fields  
- Save to Drive  
- Update Booking List  
- Move file  

**Node Details:**

- **Edit Fields**  
  - *Type:* Set node to adjust fields and prepare binary data for saving.  
  - *Configuration:* Assigns the binary "data" property from the downloaded file to the field expected by the Save to Drive node. Passes through other fields unchanged.  
  - *Inputs:* Structured output from AI agent and original binary data.  
  - *Outputs:* Prepared data with correct binary field naming for upload.  
  - *Failure Modes:* Misalignment of binary data property names causing upload failures.  

- **Save to Drive**  
  - *Type:* Google Drive node for file upload.  
  - *Configuration:*  
    - File name constructed as "{invoiceDate2} {vendor}.pdf" (e.g., "250912 VendorName.pdf").  
    - Uploads to folder ID specified by the AI agent output ("folderID" field).  
    - Uses binary data field "data" as file content.  
  - *Inputs:* Prepared binary data with metadata.  
  - *Outputs:* Confirmation metadata of saved file.  
  - *Failure Modes:* Folder access denied, invalid folder ID, file name conflicts, quota limits.  
  - *Credentials:* Google Drive OAuth2.  

- **Update Booking List**  
  - *Type:* Google Sheets node for appending data.  
  - *Configuration:*  
    - Appends a new row to the "booking list" sheet with columns: vendor, currency, invoice date, total amount, invoice number, booking account, invoice description.  
    - Values mapped directly from AI output fields.  
  - *Inputs:* Structured data from Invoice Agent output parser.  
  - *Outputs:* Confirmation of row append.  
  - *Failure Modes:* Sheet permission issues, schema mismatch, data conversion errors.  
  - *Credentials:* Google Sheets OAuth2.  

- **Move file**  
  - *Type:* Google Drive node to move files between folders.  
  - *Configuration:*  
    - Moves original invoice file from input folder to an "Output Invoice Agent" folder after processing.  
  - *Inputs:* File ID from Google Drive Trigger.  
  - *Outputs:* Confirmation of move operation.  
  - *Failure Modes:* Folder permission issues, file locking, network failures.  
  - *Credentials:* Google Drive OAuth2.  

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                            | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                       |
|---------------------|----------------------------------|--------------------------------------------|-------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger | Google Drive Trigger             | Watches for new invoice files in Drive input folder | None                    | Download file           | ## INPUT\n### Upload Invoice PDF from Google Drive Input Folder and Extract the data from the PDF binary file                   |
| Download file       | Google Drive                     | Downloads new invoice PDF from Drive       | Google Drive Trigger    | Extract from File       | ## INPUT\n### Upload Invoice PDF from Google Drive Input Folder and Extract the data from the PDF binary file                   |
| Extract from File   | Extract from File                | Extracts text and original binary from PDF | Download file           | Invoice Agent           | ## INPUT\n### Upload Invoice PDF from Google Drive Input Folder and Extract the data from the PDF binary file                   |
| Invoice Agent       | Langchain Agent                 | AI extracts invoice info and matches booking account | Extract from File, OpenAI Chat Model, Chart of Accounts, Simple Memory, Structured Output Parser | Update Booking List, Edit Fields | ## AI AGENT\n### reading relevant information (vendor, bookingtext, amount, currency etc.) from the invoice, and matching the correct booking account.\n### The output parser set the defined variables. |
| OpenAI Chat Model   | OpenAI GPT-4 Model              | Provides GPT-4 language model for AI agent | Invoice Agent (ai_languageModel) | Invoice Agent           | ## AI AGENT\n### reading relevant information (vendor, bookingtext, amount, currency etc.) from the invoice, and matching the correct booking account.\n### The output parser set the defined variables. |
| Chart of Accounts   | Google Sheets                   | Provides booking accounts reference for AI | None                    | Invoice Agent           | ## AI AGENT\n### reading relevant information (vendor, bookingtext, amount, currency etc.) from the invoice, and matching the correct booking account.\n### The output parser set the defined variables. |
| Simple Memory       | Langchain Memory Buffer         | Maintains AI conversation context          | None                    | Invoice Agent           | ## AI AGENT\n### reading relevant information (vendor, bookingtext, amount, currency etc.) from the invoice, and matching the correct booking account.\n### The output parser set the defined variables. |
| Structured Output Parser | Langchain Output Parser      | Validates and structures AI output data    | Invoice Agent           | Invoice Agent           | ## AI AGENT\n### reading relevant information (vendor, bookingtext, amount, currency etc.) from the invoice, and matching the correct booking account.\n### The output parser set the defined variables. |
| Update Booking List | Google Sheets                   | Appends extracted invoice data to booking list sheet | Invoice Agent           | Move file               | ## OUTPUT\n### Updating the Booking list with actual Invoice by upending a new row\n### Move the processed invoice to the output folder. |
| Edit Fields         | Set                            | Prepares binary data for saving to Drive   | Invoice Agent           | Save to Drive           | ## OUTPUT\n### Saving Invoice with new filename (YYMMDD vendor) as PDF-File in the according folder (matching to the booking account) |
| Save to Drive       | Google Drive                   | Saves renamed invoice PDF to assigned folder | Edit Fields             | Update Booking List     | ## OUTPUT\n### Saving Invoice with new filename (YYMMDD vendor) as PDF-File in the according folder (matching to the booking account) |
| Move file           | Google Drive                   | Moves processed invoice from input to output folder | Update Booking List     | None                    | ## OUTPUT\n### Updating the Booking list with actual Invoice by upending a new row\n### Move the processed invoice to the output folder. |
| Sticky Note3        | Sticky Note                    | Note for Input block                        | None                    | None                    | ## INPUT\n### Upload Invoice PDF from Google Drive Input Folder and Extract the data from the PDF binary file                   |
| Sticky Note1        | Sticky Note                    | Note for AI Agent block                     | None                    | None                    | ## AI AGENT\n### reading relevant information (vendor, bookingtext, amount, currency etc.) from the invoice, and matching the correct booking account.\n### The output parser set the defined variables. |
| Sticky Note         | Sticky Note                    | Note for Output block                        | None                    | None                    | ## OUTPUT\n### Saving Invoice with new filename (YYMMDD vendor) as PDF-File in the according folder (matching to the booking account) |
| Sticky Note4        | Sticky Note                    | Note for Output block                        | None                    | None                    | ## OUTPUT\n### Updating the Booking list with actual Invoice by upending a new row\n### Move the processed invoice to the output folder. |
| Sticky Note2        | Sticky Note                    | General workflow overview and credential info | None                    | None                    | # INVOICE AGENT BASIC\n\n## AI-Powered Invoice Extraction & Google Drive Automation\n\nThis workflow processes PDF incoming invoices from an input folder and extracts all relevant data. It checks the correct booking account and assigns the invoice to the corresponding storage directory. In doing so, it renames the file to the format (YYMMDD Vendor).pdf.\n\nThe chart of accounts can be customised in the corresponding Google Sheet so that everyone can find the chart of accounts they need.\n\n### Credentials:\n- Google Drive\n- Google Sheets\n- Open AI (GPT 4.0 mini) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Configure: Event = "File Created", Poll every minute  
   - Folder to watch: Set by folder ID "1YvZXJugBrB1AaJnSFWFfgQ4gJEpDWLks" ("Input AI Invoice Agent")  
   - Credential: Connect your Google Drive OAuth2 credentials  

2. **Add Download file node**  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: Expression `{{$json["id"]}}` from trigger node output  
   - Credential: Google Drive OAuth2 (same as above)  
   - Connect Google Drive Trigger → Download file  

3. **Add Extract from File node**  
   - Type: Extract from File  
   - Operation: PDF text extraction  
   - Binary Property Name: "data" (from downloaded file)  
   - Connect Download file → Extract from File  

4. **Add OpenAI Chat Model node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: Select GPT-4 mini variant (e.g., "gpt-4o-mini")  
   - Credential: Configure OpenAI API key credentials  

5. **Add Chart of Accounts node**  
   - Type: Google Sheets Tool  
   - Document ID: Use Google Sheet ID containing booking accounts "1eOAvuaURbQe-e5XZwGcafrOOdEZf8nl5Gnh_gDJ0p88"  
   - Sheet Name: Tab ID "2104487111" or sheet named "booking accounts"  
   - Credential: Google Sheets OAuth2 credentials  

6. **Add Simple Memory node**  
   - Type: Langchain Memory Buffer Window  
   - Session Key: Use expression `{{$node["Google Drive Trigger"].item.json.id}}` to uniquely identify session per file  
   - Session ID Type: Custom Key  

7. **Add Structured Output Parser node**  
   - Type: Langchain Output Parser Structured  
   - Schema Type: Manual  
   - Define JSON schema fields for invoice data (dates, vendor, booking account, amounts, currency, invoice number, folderID) as per workflow JSON  
   - Include required fields: invoiceDate, invoiceDate2, bookingAccount (ensure naming consistency), vendor, folderID  

8. **Add Invoice Agent node**  
   - Type: Langchain Agent  
   - Text input: Expression `"Invoice: " + {{$node["Extract from File"].json["text"]}}`  
   - System prompt: Describe role as helpful booking assistant extracting invoice info and using "booking accounts" tool  
   - Enable output parser and attach Structured Output Parser node  
   - Attach OpenAI Chat Model as language model  
   - Attach Chart of Accounts as AI tool  
   - Attach Simple Memory node for AI memory  
   - On error: Continue regular output  

9. **Add Edit Fields (Set) node**  
   - Type: Set  
   - Assign binary field "data" with value from `{{$node["Download file"].binary.data}}` to prepare for upload  
   - Include all other fields unchanged  
   - Connect Invoice Agent → Edit Fields  

10. **Add Save to Drive node**  
    - Type: Google Drive  
    - Operation: Upload file  
    - File Name: Expression `{{$json.output.invoiceDate2 + " " + $json.output.vendor + ".pdf"}}`  
    - Folder ID: Expression `{{$json.output.folderID}}`  
    - Input Data Field: "data" (binary)  
    - Credential: Google Drive OAuth2  
    - Connect Edit Fields → Save to Drive  

11. **Add Update Booking List node**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: Same Google Sheet as Chart of Accounts  
    - Sheet Name: "booking list" (gid=0)  
    - Columns: Map vendor, currency, invoice date, total amount, invoice number, booking account, invoice description from AI output fields  
    - Credential: Google Sheets OAuth2  
    - Connect Invoice Agent → Update Booking List  

12. **Add Move File node**  
    - Type: Google Drive  
    - Operation: Move  
    - File ID: Expression `{{$node["Google Drive Trigger"].item.json.id}}`  
    - Destination Folder ID: "1_M0f6Ms1wvTVmu3Set5ybiXa2fkfmmkc" ("Output Invoice Agent" folder)  
    - Credential: Google Drive OAuth2  
    - Connect Update Booking List → Move File  

13. **Connect Save to Drive → Update Booking List**  
    - To ensure booking list updates after saving file  

14. **Add sticky notes** for documentation at appropriate places as per original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates invoice processing from Google Drive input folder using AI for data extraction and matching, saving files renamed by date and vendor in proper folders, and updating booking records in Google Sheets.                                                                                                                                                | Workflow overview and use case                                                                                                       |
| The chart of accounts and folder mappings are customizable via Google Sheets, allowing flexible booking account assignment.                                                                                                                                                                                                                                                    | Booking accounts customization                                                                                                       |
| Credentials required: Google Drive OAuth2, Google Sheets OAuth2, OpenAI API key (for GPT-4 mini).                                                                                                                                                                                                                                                                               | Credential setup                                                                                                                     |
| Sticky note with workflow description and credentials included inside the workflow for user reference.                                                                                                                                                                                                                                                                          | Sticky Note2 content                                                                                                                 |
| Refer to Google Drive folder IDs and Google Sheet IDs carefully, ensure permissions for all used accounts to avoid access errors.                                                                                                                                                                                                                                              | Integration considerations                                                                                                          |
| Invoice date parsing expects two formats: MM/DD/YY and YYMMDD for filename and record keeping.                                                                                                                                                                                                                                                                                   | Data format requirements                                                                                                            |
| AI output schema validation is critical to ensure downstream nodes receive correctly structured data. Schema mismatch may cause failures.                                                                                                                                                                                                                                       | Data validation best practices                                                                                                      |
| For best performance, monitor quota limits and error logs on Google API and OpenAI usage; implement error handling or alerting as needed.                                                                                                                                                                                                                                       | Operational notes                                                                                                                   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, respecting current content policies and containing no illegal, offensive, or protected elements. All manipulated data are legal and public.