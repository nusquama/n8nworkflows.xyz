AI-Powered Interview Preparation System using Local LLM for Campus Placements

https://n8nworkflows.xyz/workflows/ai-powered-interview-preparation-system-using-local-llm-for-campus-placements-4761


# AI-Powered Interview Preparation System using Local LLM for Campus Placements

### 1. Workflow Overview

This workflow, titled **"AI-Powered Interview Preparation System using Local LLM for Campus Placements"**, is designed to automate and streamline the preparation process for campus placement interviews. It accepts candidate data via uploaded CSV files, processes the data using a local Large Language Model (LLM) and AI agents, generates personalized interview preparation content, produces PDF files, and subsequently emails these to the candidates. The workflow also updates candidate statuses in a Google Spreadsheet to track progress.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Data Extraction:** Handling CSV file uploads, parsing candidate data, and initializing Google Sheets to store candidate information.

- **1.2 Data Integration and Selection:** Adding parsed data to Google Sheets, selecting specific candidates or data rows for further processing.

- **1.3 AI Processing and Interview Preparation:** Utilizing a local LLM (Ollama Chat Model) and AI agents to generate interview preparation content based on candidate data.

- **1.4 Content Formatting and PDF Generation:** Converting AI-generated content into markdown format, generating PDFs from this content.

- **1.5 Email Preparation and Sending:** Preparing personalized email prompts via AI agents, sending the generated PDFs to candidates via email.

- **1.6 Status Update and Workflow Looping:** Updating candidate statuses in Google Sheets post emailing, allowing for iterative processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Extraction

**Overview:**  
This block manages incoming CSV uploads containing candidate details through a form trigger, extracts the CSV data, and creates a new Google Spreadsheet to store this data.

**Nodes Involved:**  
- Parse Uploaded CSV of Candidates (Form Trigger)  
- extract csv data (Extract from File)  
- create a sheet in google spreadsheet (Google Sheets)  
- Merge (Merge node)

**Node Details:**

- **Parse Uploaded CSV of Candidates**  
  - *Type:* Form Trigger  
  - *Role:* Entry point of the workflow; waits for user to upload a CSV file with candidates' information.  
  - *Configuration:* Webhook enabled to receive the CSV file.  
  - *Inputs:* External form upload.  
  - *Outputs:* Passes file to `extract csv data` and `create a sheet in google spreadsheet`.  
  - *Potential Failures:* Missing or malformed CSV file; webhook failures.

- **extract csv data**  
  - *Type:* Extract from File  
  - *Role:* Parses the uploaded CSV file content into structured data format.  
  - *Configuration:* Set to parse CSV format; no additional parameters specified.  
  - *Inputs:* Receives file from Form Trigger.  
  - *Outputs:* Passes extracted data to the `Merge` node.  
  - *Potential Failures:* Corrupted or non-CSV files; extraction errors.

- **create a sheet in google spreadsheet**  
  - *Type:* Google Sheets  
  - *Role:* Creates a new sheet in Google Sheets to store candidate data.  
  - *Configuration:* Authenticated with Google Sheets OAuth2 credentials; parameters to create a spreadsheet dynamically.  
  - *Inputs:* Receives original form data to initiate sheet creation.  
  - *Outputs:* Passes control to `Merge` node.  
  - *Potential Failures:* Google API authentication errors; quota limits; permission issues.

- **Merge**  
  - *Type:* Merge  
  - *Role:* Joins outputs from `extract csv data` and `create a sheet in google spreadsheet` nodes to synchronize data flow.  
  - *Inputs:* From `extract csv data` and `create a sheet in google spreadsheet`.  
  - *Outputs:* Forwards combined data to `Add csv data to google spreadsheet`.  
  - *Potential Failures:* Data alignment issues; merge conflicts.

---

#### 1.2 Data Integration and Selection

**Overview:**  
This block takes the extracted candidate data, populates the created Google Sheet with this data, and selects the first row based on a user-selected column for further AI processing.

**Nodes Involved:**  
- Add csv data to google spreadsheet (Google Sheets)  
- Select first row based on selected column (Google Sheets)  
- Merge1 (Merge node)  
- Merge2 (Merge node)

**Node Details:**

- **Add csv data to google spreadsheet**  
  - *Type:* Google Sheets  
  - *Role:* Inserts parsed CSV candidate data into the newly created sheet.  
  - *Configuration:* Uses Google Sheets credentials; configured to append or update rows based on extracted data.  
  - *Inputs:* Receives merged data from `Merge`.  
  - *Outputs:* Connects to `Select first row based on selected column`.  
  - *Potential Failures:* API limits; data schema mismatches; write permission errors.

- **Select first row based on selected column**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves the first candidate row based on criteria (likely a selected column value indicating readiness or priority).  
  - *Configuration:* Query/filter settings based on the selected column (details not explicit).  
  - *Inputs:* From `Add csv data to google spreadsheet`.  
  - *Outputs:* Feeds both AI processing agents (`Job Interview Preparation Agent` and `Email prompt Agent`).  
  - *Potential Failures:* No matching rows found; query errors.

- **Merge1 & Merge2**  
  - *Type:* Merge  
  - *Role:* Synchronize outputs from the AI agents and email prompt agents before proceeding to PDF creation and email sending.  
  - *Inputs:* From AI processing nodes and email prompt nodes respectively.  
  - *Outputs:* Forward to `Merge3` for final merge before email dispatch.  
  - *Potential Failures:* Data mismatches in merging; timing issues.

---

#### 1.3 AI Processing and Interview Preparation

**Overview:**  
This core block uses local LLM models and AI agents to analyze candidate data, generate personalized interview preparation content, and prepare email prompts.

**Nodes Involved:**  
- Ollama Chat Model  
- Gemini Search Tool  
- Job Interview Preparation Agent  
- Ollama Chat Model1  
- Email prompt Agent

**Node Details:**

- **Ollama Chat Model**  
  - *Type:* Local LLM Chat Model (Langchain node)  
  - *Role:* Provides language model capabilities for the `Job Interview Preparation Agent`.  
  - *Configuration:* Local Ollama LLM integration without external API calls (suitable for privacy and latency).  
  - *Inputs:* Feeds the agent with candidate data.  
  - *Outputs:* To `Job Interview Preparation Agent`.  
  - *Potential Failures:* Model loading failures; resource constraints.

- **Gemini Search Tool**  
  - *Type:* AI Tool integration  
  - *Role:* Supplies external or contextual search data to enrich the interview preparation content.  
  - *Inputs:* Feeds into `Job Interview Preparation Agent` as an auxiliary tool.  
  - *Outputs:* To `Job Interview Preparation Agent`.  
  - *Potential Failures:* Search service downtime; query errors.

- **Job Interview Preparation Agent**  
  - *Type:* Langchain AI Agent  
  - *Role:* Generates detailed, customized interview preparation material using candidate data and AI tools.  
  - *Configuration:* Executes once per candidate row; combines outputs from LLM and Search Tool.  
  - *Inputs:* From `Ollama Chat Model` and `Gemini Search Tool`.  
  - *Outputs:* Passes generated content to `change item name to markdown`.  
  - *Potential Failures:* Agent timeout; incomplete data input.

- **Ollama Chat Model1**  
  - *Type:* Local LLM Chat Model  
  - *Role:* Provides language model capabilities for the `Email prompt Agent`.  
  - *Inputs:* From `Select first row based on selected column`.  
  - *Outputs:* To `Email prompt Agent`.  
  - *Potential Failures:* Same as Ollama Chat Model.

- **Email prompt Agent**  
  - *Type:* Langchain AI Agent  
  - *Role:* Generates personalized email content to accompany the PDF interview preparation file.  
  - *Configuration:* Executes once per candidate; uses output from `Ollama Chat Model1`.  
  - *Outputs:* To the `Merge1` and `Merge2` nodes to synchronize with interview content.  
  - *Potential Failures:* Email content generation errors.

---

#### 1.4 Content Formatting and PDF Generation

**Overview:**  
This block formats the AI-generated content into markdown and uses a template API to convert it into PDF files.

**Nodes Involved:**  
- change item name to markdown (Code)  
- Create PDF files (API Template IO)  
- Merge3 (Merge)

**Node Details:**

- **change item name to markdown**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Converts or formats the AI-generated interview preparation content into markdown syntax suitable for PDF generation.  
  - *Configuration:* Custom script to rename or reformat fields, ensuring markdown compatibility.  
  - *Inputs:* From `Job Interview Preparation Agent`.  
  - *Outputs:* To `Create PDF files`.  
  - *Potential Failures:* Script errors; unexpected data formats.

- **Create PDF files**  
  - *Type:* API Template IO  
  - *Role:* Converts markdown content into PDF files using a template-based API.  
  - *Configuration:* Template ID and API credentials configured in parameters (not detailed).  
  - *Inputs:* Receives markdown-formatted content.  
  - *Outputs:* Passes generated PDF metadata to `Merge1` and `Merge2`.  
  - *Potential Failures:* API errors; template issues; file size limits.

- **Merge3**  
  - *Type:* Merge  
  - *Role:* Combines outputs of email and PDF generation steps before sending emails.  
  - *Inputs:* From `Merge1` and `Merge2`.  
  - *Outputs:* To `Send Email with PDF`.  
  - *Potential Failures:* Merge conflicts; data synchronization issues.

---

#### 1.5 Email Preparation and Sending

**Overview:**  
This block sends the generated PDF files to candidates by email and updates Google Sheets to reflect the updated status.

**Nodes Involved:**  
- Send Email with PDF (Email Send)  
- Update the selected column to spreadsheet (Google Sheets)

**Node Details:**

- **Send Email with PDF**  
  - *Type:* Email Send  
  - *Role:* Sends email with the PDF attachment to each candidate.  
  - *Configuration:* SMTP or OAuth2 credentials configured (e.g., Outlook OAuth2); dynamic email parameters such as recipient address, subject, body from AI-generated content.  
  - *Inputs:* From `Merge3`.  
  - *Outputs:* To `Update the selected column to spreadsheet`.  
  - *Potential Failures:* SMTP/authentication errors; attachment size limits; invalid email addresses.

- **Update the selected column to spreadsheet**  
  - *Type:* Google Sheets  
  - *Role:* Updates candidate’s status or relevant column in the Google Sheet to mark email sent or progress.  
  - *Configuration:* Uses Google Sheets credentials; writes to specific cell or row based on candidate.  
  - *Inputs:* From `Send Email with PDF`.  
  - *Outputs:* Loops back to `Select first row based on selected column` for iterative processing.  
  - *Potential Failures:* API write errors; concurrent edit conflicts.

---

#### 1.6 Miscellaneous Nodes

- **Sticky Note, Sticky Note1, Sticky Note2**  
  - *Type:* Sticky Note  
  - *Role:* Visual annotations in the workflow canvas; no effect on data flow or processing.  
  - *Content:* Empty in this workflow.  
  - *Potential Failures:* None.

---

### 3. Summary Table

| Node Name                          | Node Type                       | Functional Role                                 | Input Node(s)                            | Output Node(s)                                  | Sticky Note |
|-----------------------------------|--------------------------------|------------------------------------------------|-----------------------------------------|-------------------------------------------------|-------------|
| Parse Uploaded CSV of Candidates   | Form Trigger                   | Receives candidate CSV uploads                  | External webhook/form                    | extract csv data, create a sheet in google spreadsheet |             |
| extract csv data                  | Extract from File              | Parses uploaded CSV file                         | Parse Uploaded CSV of Candidates         | Merge                                           |             |
| create a sheet in google spreadsheet | Google Sheets                 | Creates new Google Sheet for candidates         | Parse Uploaded CSV of Candidates         | Merge                                           |             |
| Merge                            | Merge                         | Joins extracted data and sheet creation outputs | extract csv data, create a sheet in google spreadsheet | Add csv data to google spreadsheet               |             |
| Add csv data to google spreadsheet | Google Sheets                 | Adds candidate data to Google Sheet              | Merge                                    | Select first row based on selected column       |             |
| Select first row based on selected column | Google Sheets                 | Selects candidate row for processing             | Add csv data to google spreadsheet       | Job Interview Preparation Agent, Email prompt Agent |             |
| Ollama Chat Model                | LLM Chat Model (Langchain)    | Local LLM for interview preparation agent       | Select first row based on selected column | Job Interview Preparation Agent                  |             |
| Gemini Search Tool               | AI Tool                      | Provides search data for interview preparation   | Select first row based on selected column | Job Interview Preparation Agent                  |             |
| Job Interview Preparation Agent  | Langchain Agent               | Generates interview preparation content          | Ollama Chat Model, Gemini Search Tool    | change item name to markdown                      |             |
| change item name to markdown      | Code                         | Formats AI content into markdown                  | Job Interview Preparation Agent          | Create PDF files                                 |             |
| Create PDF files                 | API Template IO               | Converts markdown to PDF                           | change item name to markdown              | Merge1, Merge2                                   |             |
| Merge1                          | Merge                         | Merges email prompt with PDF generation           | Email prompt Agent, Create PDF files      | Merge3                                           |             |
| Merge2                          | Merge                         | Merges email prompt with PDF generation           | Email prompt Agent, Create PDF files      | Merge3                                           |             |
| Merge3                          | Merge                         | Final merge before sending email                   | Merge1, Merge2                            | Send Email with PDF                              |             |
| Ollama Chat Model1               | LLM Chat Model (Langchain)    | Local LLM for email prompt agent                   | Select first row based on selected column | Email prompt Agent                               |             |
| Email prompt Agent              | Langchain Agent               | Generates personalized email content               | Ollama Chat Model1                        | Merge1, Merge2                                   |             |
| Send Email with PDF             | Email Send                   | Sends email with PDF attachment                      | Merge3                                   | Update the selected column to spreadsheet       |             |
| Update the selected column to spreadsheet | Google Sheets                 | Updates candidate status in sheet                   | Send Email with PDF                       | Select first row based on selected column       |             |
| Sticky Note                     | Sticky Note                  | Visual annotation (empty)                            | None                                     | None                                            |             |
| Sticky Note1                    | Sticky Note                  | Visual annotation (empty)                            | None                                     | None                                            |             |
| Sticky Note2                    | Sticky Note                  | Visual annotation (empty)                            | None                                     | None                                            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**  
   - Add a **Form Trigger** node named `Parse Uploaded CSV of Candidates`.  
   - Configure it to accept CSV file uploads via webhook.

2. **Add Extract from File Node:**  
   - Add an **Extract from File** node named `extract csv data`.  
   - Connect it to the Form Trigger node.  
   - Configure to parse CSV format files.

3. **Create Google Sheets Node for Sheet Creation:**  
   - Add a **Google Sheets** node named `create a sheet in google spreadsheet`.  
   - Connect it to the Form Trigger node (parallel to extract csv data).  
   - Configure with Google OAuth2 credentials.  
   - Set it to create a new spreadsheet dynamically.

4. **Add Merge Node:**  
   - Add a **Merge** node named `Merge`.  
   - Connect outputs from `extract csv data` and `create a sheet in google spreadsheet` to this node.

5. **Add Google Sheets Node to Add Data:**  
   - Add a **Google Sheets** node named `Add csv data to google spreadsheet`.  
   - Connect the `Merge` node output here.  
   - Configure to insert rows into the newly created spreadsheet.

6. **Add Google Sheets Node to Select Candidate Row:**  
   - Add a **Google Sheets** node named `Select first row based on selected column`.  
   - Connect the previous node output here.  
   - Configure to select the first row based on a specified column (e.g., "Status" or "Ready").

7. **Add Ollama Chat Model Node for Interview Preparation:**  
   - Add a **Langchain Ollama Chat Model** node named `Ollama Chat Model`.  
   - Connect `Select first row based on selected column` to this node as input.  
   - Configure to use local Ollama LLM integration.

8. **Add Gemini Search Tool Node:**  
   - Add a **Gemini Search Tool** node named `Gemini Search Tool`.  
   - Connect the same input as Ollama Chat Model.  
   - Configure API or local setup as needed.

9. **Add Job Interview Preparation Agent Node:**  
   - Add a **Langchain Agent** node named `Job Interview Preparation Agent`.  
   - Connect `Ollama Chat Model` as language model input and `Gemini Search Tool` as AI tool input.  
   - Connect `Select first row based on selected column` as main input.  
   - Configure to execute once per candidate.

10. **Add Code Node to Format Content:**  
    - Add a **Code** node named `change item name to markdown`.  
    - Connect from `Job Interview Preparation Agent`.  
    - Write JavaScript to convert AI output into markdown format.

11. **Add API Template IO Node to Create PDFs:**  
    - Add an **API Template IO** node named `Create PDF files`.  
    - Connect from the Code node.  
    - Configure with template ID and API credentials to generate PDFs.

12. **Add Ollama Chat Model Node for Email Prompt:**  
    - Add another **Langchain Ollama Chat Model** node named `Ollama Chat Model1`.  
    - Connect from `Select first row based on selected column`.  
    - Configure similarly as the first Ollama node.

13. **Add Email Prompt Agent Node:**  
    - Add a **Langchain Agent** node named `Email prompt Agent`.  
    - Connect `Ollama Chat Model1` as language model input.  
    - Connect `Select first row based on selected column` as main input.  
    - Configure to generate personalized email text.

14. **Add Two Merge Nodes:**  
    - Add **Merge1** and **Merge2** nodes.  
    - Configure `Merge1` to merge outputs from `Email prompt Agent` and `Create PDF files` (first path).  
    - Configure `Merge2` similarly (second path).  
    - Connect these merges to a final `Merge3` node.

15. **Add Merge3 Node:**  
    - Add a **Merge** node named `Merge3`.  
    - Connect `Merge1` and `Merge2` to it.  
    - This node synchronizes email and PDF data.

16. **Add Email Send Node:**  
    - Add an **Email Send** node named `Send Email with PDF`.  
    - Connect from `Merge3`.  
    - Configure SMTP or OAuth2 credentials (e.g., Outlook OAuth2).  
    - Use dynamic fields for recipient email, subject, body, and attach the PDF.

17. **Add Google Sheets Node to Update Status:**  
    - Add a **Google Sheets** node named `Update the selected column to spreadsheet`.  
    - Connect from `Send Email with PDF`.  
    - Configure to update a candidate’s status column indicating email sent.

18. **Loop Back for Iterative Processing:**  
    - Connect `Update the selected column to spreadsheet` back to `Select first row based on selected column` to process next candidates iteratively.

19. **Add Empty Sticky Notes (Optional):**  
    - Add **Sticky Note**, **Sticky Note1**, and **Sticky Note2** nodes for visual annotations if desired.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow uses local LLM integration via Ollama, which enhances privacy and reduces latency compared to external APIs.        | See Ollama LLM official docs for setup: https://ollama.com/docs                                |
| The Gemini Search Tool integration supplements AI content with contextual search data, improving relevance and quality.           | Gemini Search API documentation (if applicable)                                               |
| Google Sheets OAuth2 credentials must be configured with edit permissions on target spreadsheets.                                  | Google Cloud Console: https://console.cloud.google.com/apis/credentials                         |
| Email credentials require proper SMTP or OAuth2 setup; Outlook OAuth2 is recommended for secure email sending.                     | Microsoft OAuth2 setup guide: https://docs.microsoft.com/en-us/azure/active-directory/develop/ |
| Template IO API is used for PDF generation; ensure API keys and template IDs are correctly configured.                             | Template IO: https://templat.io/api                                                             |

---

This completes the detailed reference documentation of the **AI-Powered Interview Preparation System using Local LLM for Campus Placements** workflow. It enables users and AI agents to fully understand, reproduce, and modify the workflow with clear insight into node configurations, data flow, and integration points.