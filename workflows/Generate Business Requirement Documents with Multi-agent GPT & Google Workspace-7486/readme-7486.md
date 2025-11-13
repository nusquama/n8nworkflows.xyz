Generate Business Requirement Documents with Multi-agent GPT & Google Workspace

https://n8nworkflows.xyz/workflows/generate-business-requirement-documents-with-multi-agent-gpt---google-workspace-7486


# Generate Business Requirement Documents with Multi-agent GPT & Google Workspace

### 1. Workflow Overview

This workflow automates the generation of Business Requirements Documents (BRDs) using a multi-agent GPT system integrated with Google Workspace services. It targets Business Analysts, Project Managers, and Operations Teams who need to streamline BRD creation, tracking, and delivery based on submitted project requests and supporting documents.

The workflow is logically divided into six functional blocks:

- **1.1 Capture BRD Request & Supporting Files**: Triggered by form submissions capturing project details and uploads, storing these in Google Sheets and Google Drive.
- **1.2 Process & Extract Key Information**: Extracts text from uploaded PDFs and stores semantic embeddings in an in-memory vector store for AI retrieval.
- **1.3 Multi-Agent BRD Draft Generation**: Two specialized AI agents generate complementary parts of the BRD using retrieval-augmented generation (RAG) from the stored document embeddings.
- **1.4 Merge & Enrich Content**: Merges AI-generated content, configures metadata, and creates a Google Docs document draft.
- **1.5 Convert & Archive Final BRD**: Converts the Google Docs file to PDF format and archives it in Google Drive.
- **1.6 Deliver & Update Status**: Sends the final BRD PDF via email to the requester and updates the project status in the tracking Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Capture BRD Request & Supporting Files

- **Overview:**  
  This block captures BRD requests from a form submission, processes multiple uploaded files, creates a BRD request record, uploads files to Google Drive, and logs all records in Google Sheets.

- **Nodes Involved:**  
  - On form submission  
  - Handle multiple files  
  - Create BRD request record  
  - Upload supporting  
  - Create supporting document record(s)  
  - Add supporting document record(s)  
  - Add BRD record to tracking

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry trigger when a user submits the "Business Requirements Document Request Form".  
    - *Configuration:* Form fields include Project ID, Project Title, Business Domain (dropdown), Supporting Documents (PDF files), Notes, and Email. All required.  
    - *Inputs:* User form submission data.  
    - *Outputs:* Passes form data to downstream nodes.  
    - *Edge cases:* Missing required fields, invalid email format, unsupported file types.  

  - **Handle multiple files**  
    - *Type:* Code node  
    - *Role:* Separates multiple uploaded files into individual items for processing.  
    - *Key Code:* Extracts binary data labeled with "Supporting_Documents" prefix and outputs them as separate items.  
    - *Input:* Form submission item with possible multiple files.  
    - *Output:* One item per uploaded file for parallel processing.  
    - *Edge cases:* No files uploaded, corrupted file data.  

  - **Create BRD request record**  
    - *Type:* Code node  
    - *Role:* Extracts and formats main request fields from the form submission into a flat JSON object to be stored.  
    - *Key Expressions:* Accesses form fields from "On form submission".  
    - *Output:* Object with ProjectID, ProjectTitle, BusinessDomain, Notes, Email, Status ('Submitted'), SubmittedAt.  
    - *Edge cases:* Missing timestamp, malformed data.  

  - **Upload supporting**  
    - *Type:* Google Drive node  
    - *Role:* Uploads each supporting document file to a designated Google Drive folder ("SmartSales").  
    - *Configuration:* Filename prefixed with timestamp and original file name; folder ID specified.  
    - *Input:* Individual binary file data from "Handle multiple files".  
    - *Output:* Metadata of uploaded file (including Drive ID and links).  
    - *Edge cases:* Google Drive API errors, insufficient permissions, file size limits.  

  - **Create supporting document record(s)**  
    - *Type:* Code node  
    - *Role:* Creates a JSON record for each uploaded supporting document with metadata (ProjectID, DocumentId, DocumentName, URL, CreatedTime).  
    - *Input:* Output from "Upload supporting".  
    - *Output:* Structured record for Google Sheets insertion.  
    - *Edge cases:* Missing file metadata.  

  - **Add supporting document record(s)**  
    - *Type:* Google Sheets node  
    - *Role:* Appends supporting document metadata records to a dedicated Google Sheet tab ("Documents").  
    - *Configuration:* Automatic mapping; sheet and document IDs set.  
    - *Edge cases:* Google Sheets API errors, rate limits.  

  - **Add BRD record to tracking**  
    - *Type:* Google Sheets node  
    - *Role:* Appends the main BRD request record to the tracking sheet ("Requests").  
    - *Configuration:* Auto-mapping of fields; no matching columns for uniqueness.  
    - *Edge cases:* API errors, data format issues.  

---

#### 1.2 Process & Extract Key Information

- **Overview:**  
  Extracts text content from uploaded PDF files and inserts the extracted text into an in-memory vector store for semantic search by AI agents.

- **Nodes Involved:**  
  - Extract from File  
  - Insert Data to Store (vector store)  
  - Default Data Loader  
  - Embeddings

- **Node Details:**

  - **Extract from File**  
    - *Type:* Extract from File node  
    - *Role:* Extracts text from PDF files uploaded by the user.  
    - *Configuration:* Operation set to "pdf".  
    - *Input:* Binary PDF files from "Handle multiple files".  
    - *Output:* Extracted text content for each file.  
    - *Edge cases:* Corrupted PDFs, extraction failures.  

  - **Insert Data to Store**  
    - *Type:* Langchain vectorStoreInMemory node  
    - *Role:* Inserts extracted text into an in-memory vector store keyed by "vector_store_key".  
    - *Configuration:* Mode set to "insert".  
    - *Input:* Extracted text from "Extract from File" and documents from "Default Data Loader".  
    - *Output:* Updated vector store for querying.  
    - *Edge cases:* Memory limitations, insertion errors.  

  - **Default Data Loader**  
    - *Type:* Langchain documentDefaultDataLoader node  
    - *Role:* Loads default documents if any (context not detailed in JSON).  
    - *Input:* N/A  
    - *Output:* Documents fed into vector store insert.  
    - *Edge cases:* Missing default documents.  

  - **Embeddings**  
    - *Type:* Langchain embeddingsOpenAi node  
    - *Role:* Generates OpenAI embeddings for documents before insertion.  
    - *Credentials:* OpenAI API key configured.  
    - *Edge cases:* API rate limits, embedding failures, auth errors.  

---

#### 1.3 Multi-Agent BRD Draft Generation

- **Overview:**  
  Two specialized AI agents generate sections of the BRD using retrieval-augmented generation by querying the vector store.

- **Nodes Involved:**  
  - Query Data Tool  
  - OpenAI Chat Model  
  - General BRD Writer Agent  
  - Query Data Tool1  
  - OpenAI Chat Model1  
  - Business Requirement Writer Agent  
  - Request completed? (Filter)  

- **Node Details:**

  - **Query Data Tool & Query Data Tool1**  
    - *Type:* Langchain vectorStoreInMemory (tool mode)  
    - *Role:* Provides retrieval tool interface for agents to query knowledge base.  
    - *Configuration:* Tool name "knowledge_base"; memory key "vector_store_key".  
    - *Output:* Supplies relevant context to AI agents.  

  - **OpenAI Chat Model & OpenAI Chat Model1**  
    - *Type:* Langchain lmChatOpenAi nodes  
    - *Role:* Underlying language models serving the AI agents.  
    - *Configuration:* Models "gpt-4" and "gpt-4.1" respectively.  
    - *Credentials:* OpenAI API key.  
    - *Edge cases:* API limits, timeout, invalid model names.  

  - **General BRD Writer Agent**  
    - *Type:* Langchain agent node  
    - *Role:* Generates general BRD sections (executive summary, project overview, etc.).  
    - *Prompt:* Detailed system message defining the agent as an expert Business Analyst; strict markdown output; standard BRD structure.  
    - *Input:* Project metadata and vector store for queries.  
    - *Output:* Partial BRD content.  
    - *Edge cases:* Missing data in knowledge base, generation errors.  

  - **Business Requirement Writer Agent**  
    - *Type:* Langchain agent node  
    - *Role:* Generates detailed business and functional requirements sections.  
    - *Prompt:* Similar to General BRD Writer but focused on detailed requirements with IDs, acceptance criteria, traceability, etc.  
    - *Input:* Same as above.  
    - *Output:* Partial BRD content.  
    - *Edge cases:* As above.  

  - **Request completed?**  
    - *Type:* Filter node  
    - *Role:* Ensures agents only run if request status is not "Completed".  
    - *Condition:* Status != "Completed".  
    - *Edge cases:* Missing or malformed status field.  

---

#### 1.4 Merge & Enrich Content

- **Overview:**  
  Merges output from both AI agents, configures document metadata, and creates the Google Docs draft file.

- **Nodes Involved:**  
  - Merge  
  - Merge content (Code node)  
  - Configure metadata  
  - Create document file  

- **Node Details:**

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines outputs from General BRD Writer Agent and Business Requirement Writer Agent.  
    - *Configuration:* Default merge mode (likely 'append').  

  - **Merge content**  
    - *Type:* Code node  
    - *Role:* Concatenates the "output" strings from both agents into a single markdown string for the document.  
    - *Code:* Maps over inputs to join outputs with space delimiter. Returns mergedOutput and request metadata.  
    - *Edge cases:* Missing or empty outputs.  

  - **Configure metadata**  
    - *Type:* Set node  
    - *Role:* Assigns static and dynamic metadata fields needed for document creation, including Drive Folder ID, sender info, company name, and document content.  
    - *Edge cases:* Incorrect folder ID, missing sender email or name.  

  - **Create document file**  
    - *Type:* HTTP Request node (Google Drive API)  
    - *Role:* Creates a new Google Docs document with the BRD content in markdown format inside the specified Google Drive folder.  
    - *Configuration:* Multipart upload with JSON metadata and markdown content.  
    - *Authentication:* Google Drive OAuth2.  
    - *Edge cases:* API errors, invalid folder ID, content encoding issues.  

---

#### 1.5 Convert & Archive Final BRD

- **Overview:**  
  Converts the Google Docs file to PDF, archives it in a designated Google Drive folder.

- **Nodes Involved:**  
  - Convert document to PDF  
  - Archiving PDF File  

- **Node Details:**

  - **Convert document to PDF**  
    - *Type:* Google Drive node  
    - *Role:* Converts the created Google Docs file to PDF format via Google Drive API.  
    - *Configuration:* Downloads file as PDF using "docsToFormat" setting.  
    - *Input:* File ID from "Create document file".  
    - *Output:* PDF binary data.  
    - *Edge cases:* Conversion failures, file not found.  

  - **Archiving PDF File**  
    - *Type:* Google Drive node  
    - *Role:* Uploads the generated PDF into a dedicated Google Drive folder for archived BRDs.  
    - *Configuration:* Uses the original filename with ".pdf" suffix; folder ID specified.  
    - *Edge cases:* Upload errors, folder permission issues.  

---

#### 1.6 Deliver & Update Status

- **Overview:**  
  Sends the final BRD PDF to the requester by email and updates the request status in the tracking Google Sheet as "Completed".

- **Nodes Involved:**  
  - Send BRD response email  
  - Create record to update Google Sheet row  
  - Mark request as completed  
  - New request added to tracking sheet (trigger)  

- **Node Details:**

  - **Send BRD response email**  
    - *Type:* SendGrid node  
    - *Role:* Sends an HTML email with the BRD PDF attached to the requester.  
    - *Configuration:* Subject and body dynamically use project metadata; sender email/name configured; attachment included.  
    - *Credentials:* SendGrid API key.  
    - *Edge cases:* Email delivery failures, attachment size limits.  

  - **Create record to update Google Sheet row**  
    - *Type:* Code node  
    - *Role:* Prepares an updated record with Status set to "Completed" to update the tracking sheet.  
    - *Input:* Merged content request metadata.  
    - *Output:* Record for update operation.  

  - **Mark request as completed**  
    - *Type:* Google Sheets node  
    - *Role:* Updates the existing BRD request row in the tracking Google Sheet to set Status = "Completed".  
    - *Configuration:* Uses ProjectID as matching column; updates status and other fields.  
    - *Edge cases:* Sheet row not found, API failures.  

  - **New request added to tracking sheet**  
    - *Type:* Google Sheets Trigger node  
    - *Role:* Polls the tracking sheet every 5 minutes to detect new requests and triggers the AI generation process if status is not "Completed".  
    - *Edge cases:* Polling delays, concurrent updates.  

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                          | Input Node(s)                  | Output Node(s)                           | Sticky Note                                                                                                                                |
|-------------------------------|----------------------------------|----------------------------------------|-------------------------------|-----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger                     | Entry point; capture BRD request       | -                             | Handle multiple files, Create BRD request record | ### 1. Capture BRD Request & Supporting Files: Logs request and uploads files to Google Drive.                                                |
| Handle multiple files          | Code                            | Splits multiple uploaded files         | On form submission            | Upload supporting, Extract from File    | ### 1. Capture BRD Request & Supporting Files                                                                                               |
| Create BRD request record      | Code                            | Formats main request record             | On form submission            | Add BRD record to tracking               | ### 1. Capture BRD Request & Supporting Files                                                                                               |
| Upload supporting             | Google Drive                    | Uploads supporting documents to Drive  | Handle multiple files         | Create supporting document record(s)    | ### 1. Capture BRD Request & Supporting Files                                                                                               |
| Create supporting document record(s) | Code                            | Creates metadata record per file        | Upload supporting             | Add supporting document record(s)       | ### 1. Capture BRD Request & Supporting Files                                                                                               |
| Add supporting document record(s) | Google Sheets                   | Logs supporting document records        | Create supporting document record(s) | -                                   | ### 1. Capture BRD Request & Supporting Files                                                                                               |
| Add BRD record to tracking     | Google Sheets                   | Logs main BRD request record            | Create BRD request record     | -                                       | ### 1. Capture BRD Request & Supporting Files                                                                                               |
| Extract from File              | Extract from File                | Extracts text from PDFs                  | Handle multiple files         | Insert Data to Store                     | ### 2. Process & Extract Key Information: Parses PDFs and stores content in vector store.                                                    |
| Insert Data to Store           | Langchain vectorStoreInMemory   | Inserts extracted text into vector store | Extract from File, Default Data Loader | Query Data Tool, Query Data Tool1        | ### 2. Process & Extract Key Information                                                                                                   |
| Default Data Loader            | Langchain documentDefaultDataLoader | Loads default docs (if any)             | -                             | Insert Data to Store                     | ### 2. Process & Extract Key Information                                                                                                   |
| Embeddings                    | Langchain embeddingsOpenAi      | Creates embeddings for documents        | -                             | Insert Data to Store                     | ### 2. Process & Extract Key Information                                                                                                   |
| Query Data Tool               | Langchain vectorStoreInMemory   | Retrieval tool for General BRD Agent    | Insert Data to Store          | General BRD Writer Agent                 | ### 3. Multi-Agent BRD Draft Generation: AI agents use retrieval tools to generate BRD sections.                                            |
| OpenAI Chat Model             | Langchain lmChatOpenAi          | Language model for General BRD Agent    | -                             | General BRD Writer Agent                 | ### 3. Multi-Agent BRD Draft Generation                                                                                                    |
| General BRD Writer Agent      | Langchain agent                 | Generates general BRD content            | Request completed?, Query Data Tool, OpenAI Chat Model | Merge                                    | ### 3. Multi-Agent BRD Draft Generation                                                                                                    |
| Query Data Tool1              | Langchain vectorStoreInMemory   | Retrieval tool for Business Req Agent   | Insert Data to Store          | Business Requirement Writer Agent        | ### 3. Multi-Agent BRD Draft Generation                                                                                                    |
| OpenAI Chat Model1            | Langchain lmChatOpenAi          | Language model for Business Req Agent   | -                             | Business Requirement Writer Agent        | ### 3. Multi-Agent BRD Draft Generation                                                                                                    |
| Business Requirement Writer Agent | Langchain agent                 | Generates detailed requirements sections | Request completed?, Query Data Tool1, OpenAI Chat Model1 | Merge                                    | ### 3. Multi-Agent BRD Draft Generation                                                                                                    |
| Request completed?            | Filter                         | Checks if request status != Completed   | New request added to tracking sheet | General BRD Writer Agent, Business Requirement Writer Agent | ### 3. Multi-Agent BRD Draft Generation                                                                                                    |
| Merge                        | Merge                          | Combines outputs from both AI agents    | General BRD Writer Agent, Business Requirement Writer Agent | Merge content                            | ### 4. Merge & Enrich Content: Combines AI outputs for final document.                                                                      |
| Merge content                | Code                           | Concatenates AI outputs and adds metadata | Merge                         | Configure metadata                       | ### 4. Merge & Enrich Content                                                                                                               |
| Configure metadata           | Set                            | Sets document metadata and sender info | Merge content                | Create document file                     | ### 4. Merge & Enrich Content                                                                                                               |
| Create document file          | HTTP Request (Google Drive API) | Creates Google Docs document with BRD  | Configure metadata            | Convert document to PDF                  | ### 4. Merge & Enrich Content                                                                                                               |
| Convert document to PDF       | Google Drive                   | Converts Google Docs to PDF              | Create document file          | Archiving PDF File, Send BRD response email | ### 5. Convert & Archive Final BRD                                                                                                         |
| Archiving PDF File            | Google Drive                   | Archives PDF to dedicated folder         | Convert document to PDF       | -                                       | ### 5. Convert & Archive Final BRD                                                                                                         |
| Send BRD response email       | SendGrid                      | Sends final BRD PDF to requester         | Convert document to PDF       | Create record to update Google Sheet row | ### 6. Deliver & Update Status: Sends email and updates request status.                                                                     |
| Create record to update Google Sheet row | Code                           | Prepares updated record with status Completed | Send BRD response email       | Mark request as completed                | ### 6. Deliver & Update Status                                                                                                              |
| Mark request as completed     | Google Sheets                 | Updates tracking sheet with status Completed | Create record to update Google Sheet row | -                                       | ### 6. Deliver & Update Status                                                                                                              |
| New request added to tracking sheet | Google Sheets Trigger         | Polls tracking sheet for new requests   | -                             | Request completed?                       | ### 6. Deliver & Update Status                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**:  
   - Type: `Form Trigger`  
   - Name: "On form submission"  
   - Configure a form titled "Business Requirements Document Request Form" with fields:  
     - Project ID (string, required)  
     - Project Title (string, required)  
     - Business Domain (dropdown with predefined values, required)  
     - Supporting Documents (file upload, accept `.pdf`, required)  
     - Notes (textarea, required)  
     - Email (email type, required)  
   - This node triggers the workflow when the form is submitted.

2. **Add a Code Node to Handle Multiple Files**:  
   - Name: "Handle multiple files"  
   - JavaScript code: separates multiple uploaded files from the form submission into individual items.  
   - Input: output from "On form submission".  
   - Output: one item per file.

3. **Create a Code Node to Format BRD Request Record**:  
   - Name: "Create BRD request record"  
   - Mode: run once for each item (actually only one item from form)  
   - Code: extracts ProjectID, ProjectTitle, BusinessDomain, Notes, Email, Status='Submitted', SubmittedAt (timestamp) from form data.  
   - Input: "On form submission".  
   - Output: structured JSON record.

4. **Add Google Drive Node to Upload Supporting Documents**:  
   - Name: "Upload supporting"  
   - Operation: Upload file  
   - Folder ID: set to your designated folder for supporting docs (e.g., "SmartSales" folder ID)  
   - File name: dynamic with timestamp and original file name  
   - Input: "Handle multiple files".  
   - Credentials: Google Drive OAuth2.

5. **Create a Code Node to Construct Supporting Document Records**:  
   - Name: "Create supporting document record(s)"  
   - Mode: run once per item  
   - Code: builds metadata JSON (ProjectID from form, DocumentId, DocumentName, URL, CreatedTime from upload output)  
   - Input: "Upload supporting".  
   - Output: records for spreadsheet.

6. **Add Google Sheets Node to Log Supporting Documents**:  
   - Name: "Add supporting document record(s)"  
   - Operation: Append rows  
   - Target: Supporting Documents sheet/tab in your tracking spreadsheet  
   - Input: "Create supporting document record(s)"  
   - Credentials: Google Sheets OAuth2.

7. **Add Google Sheets Node to Log BRD Request Record**:  
   - Name: "Add BRD record to tracking"  
   - Operation: Append row  
   - Target: Requests sheet/tab in your tracking spreadsheet  
   - Input: "Create BRD request record"  
   - Credentials: Google Sheets OAuth2.

8. **Add Extract from File Node to Parse PDFs**:  
   - Name: "Extract from File"  
   - Operation: PDF text extraction  
   - Input: "Handle multiple files".  
   - Output: extracted text content.

9. **Add Langchain Embeddings Node**:  
   - Name: "Embeddings"  
   - Credentials: OpenAI API  
   - Input: "Extract from File".

10. **Add Langchain Default Data Loader Node** (optional depending on your use case):  
    - Name: "Default Data Loader"  
    - Input: none or default documents.

11. **Add Langchain Vector Store In Memory Node to Insert Data**:  
    - Name: "Insert Data to Store"  
    - Mode: insert  
    - Memory key: "vector_store_key"  
    - Input: from "Embeddings" and "Default Data Loader".

12. **Add Langchain Vector Store In Memory Nodes as Query Tools**:  
    - Name: "Query Data Tool" and "Query Data Tool1"  
    - Mode: retrieve-as-tool  
    - Tool name: "knowledge_base"  
    - Memory key: "vector_store_key"  

13. **Add Langchain OpenAI Chat Model Nodes**:  
    - Name: "OpenAI Chat Model" (model: gpt-4)  
    - Name: "OpenAI Chat Model1" (model: gpt-4.1)  
    - Credentials: OpenAI API.

14. **Add Langchain Agent Nodes for BRD Generation**:  
    - Name: "General BRD Writer Agent"  
      - System prompt: expert Business Analyst; standard BRD structure with executive summary, project overview, etc.  
      - Input: Project data and "Query Data Tool"  
      - Uses "OpenAI Chat Model".  
    - Name: "Business Requirement Writer Agent"  
      - System prompt: detailed business & functional requirements; elaborate templates and traceability.  
      - Input: Project data and "Query Data Tool1"  
      - Uses "OpenAI Chat Model1".  

15. **Add Filter Node "Request completed?"**  
    - Condition: allow processing only if status != "Completed".

16. **Connect "General BRD Writer Agent" and "Business Requirement Writer Agent" outputs to a Merge node**:  
    - Name: "Merge"  
    - Combine outputs from both agents.

17. **Add Code Node to Merge Agent Outputs**:  
    - Name: "Merge content"  
    - Code: concatenate outputs into a single markdown string; include request metadata.

18. **Add Set Node to Configure Metadata**:  
    - Name: "Configure metadata "  
    - Assign Drive Folder ID, Sender Email, Sender Name, Company Name, Document Content (merged markdown), Project ID, Project Title.

19. **Add HTTP Request Node to Create Google Docs File**:  
    - Name: "Create document file"  
    - Method: POST to Google Drive API multipart upload endpoint  
    - Body: multipart with JSON metadata and markdown content  
    - Authentication: Google Drive OAuth2  
    - Folder: use Drive Folder ID from metadata.

20. **Add Google Drive Node to Convert Docs to PDF**:  
    - Name: "Convert document to PDF"  
    - Operation: download with conversion to PDF  
    - Input: file ID from "Create document file".

21. **Add Google Drive Node to Archive PDF**:  
    - Name: "Archiving PDF File"  
    - Upload PDF to archive folder in Google Drive  
    - Use file name and folder ID.

22. **Add SendGrid Node to Send Email with PDF Attachment**:  
    - Name: "Send BRD response email"  
    - Configure subject and body using project metadata  
    - Attach PDF file (binary)  
    - Credentials: SendGrid API.

23. **Add Code Node to Prepare Google Sheets Update Record**:  
    - Name: "Create record to update Google Sheet row"  
    - Set Status to "Completed".

24. **Add Google Sheets Node to Update Request Status**:  
    - Name: "Mark request as completed"  
    - Operation: update row matching ProjectID  
    - Credentials: Google Sheets OAuth2.

25. **Add Google Sheets Trigger Node**:  
    - Name: "New request added to tracking sheet"  
    - Poll every 5 minutes for new rows in Requests sheet  
    - Trigger AI generation after filtering by status.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow addresses automation of BRD creation by combining multi-agent GPT with Google Workspace integration, enabling scalable and consistent document generation for enterprises. It handles form intake, document upload, semantic processing, AI-assisted drafting, document creation, archival, and delivery.                                                             | Workflow description and use case                                                                          |
| Sample supporting document PDF (used for testing): [Customer Feedback Analysis & Automation Platform.pdf](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Customer+Feedback+Analysis+%26+Automation+Platform.pdf)                                                                                                                                                           | Example input file                                                                                         |
| Sample output PDF for reference: [BRD-2025-001 Customer Feedback Analysis & Automation Platform.pdf](https://wisestackai.s3.ap-southeast-1.amazonaws.com/BRD-2025-001+Customer+Feedback+Analysis+%26+Automation+Platform.pdf)                                                                                                                                                   | Example generated document                                                                                |
| For setup, ensure Google Sheets and Drive APIs are enabled and OAuth2 credentials are properly configured. OpenAI API keys must have access to GPT-4 or equivalent. SendGrid API keys are required for email delivery.                                                                                                                                                          | Setup requirements                                                                                         |
| The AI agents’ system prompts can be customized to align documents with organizational standards and terminologies. Additional notification channels (e.g., Slack) can be integrated after the email step if desired.                                                                                                                                                            | Customization suggestions                                                                                   |
| Embedded images in sticky notes provide visual references to node configurations and workflow layout for user convenience.                                                                                                                                                                                                                                                  | Visual aids linked in sticky notes                                                                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow made with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected material. All data processed are legal and public.