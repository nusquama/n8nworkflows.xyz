Process Resumes from Google Drive to ClickUp using GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/process-resumes-from-google-drive-to-clickup-using-gpt-4o-and-google-sheets-8529


# Process Resumes from Google Drive to ClickUp using GPT-4o and Google Sheets

### 1. Workflow Overview

This workflow automates the processing of newly uploaded resumes in a designated Google Drive folder, extracting structured candidate data using GPT-4o AI, storing that data in Google Sheets, and creating corresponding hiring tasks in ClickUp. It is designed to streamline recruitment pipelines by eliminating manual data entry and ensuring rapid, consistent candidate information handling.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Detect new resume files uploaded to Google Drive.
- **1.2 File Processing:** Download the resume file and convert from PDF to raw text.
- **1.3 AI Data Extraction:** Use GPT-4o to analyze the resume text and extract key candidate details as JSON.
- **1.4 Data Cleaning:** Sanitize and verify the AI output JSON for database compatibility.
- **1.5 Database Storage:** Append or update the candidate information in a Google Sheets database.
- **1.6 Task Creation:** Generate a new hiring task in ClickUp assigned to the recruitment team.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block continuously monitors a specific Google Drive folder for new resume files. Upon detecting a new file creation event, it triggers the workflow to start processing.

**Nodes Involved:**  
- Watch for New Resumes  
- Sticky Note - Trigger (documentation node)

**Node Details:**

- **Watch for New Resumes**  
  - Type: Google Drive Trigger  
  - Role: Monitors a folder on Google Drive for new files (event = fileCreated)  
  - Configuration: Watches folder with ID `1MIvpHU_ZqG76Vov2-D5WlS5dD3UhOMSz` named "Resume_store", polls every minute  
  - Inputs: None (trigger node)  
  - Outputs: Emits metadata about new files (including file ID)  
  - Edge cases: Potential delays due to polling frequency, Google Drive API rate limits, auth token expiration  
  - Sticky Note Reference: "🚀 WORKFLOW START" explains trigger settings and user tips

#### 1.2 File Processing

**Overview:**  
Downloads the detected resume file from Google Drive and converts it from PDF format into plain text suitable for AI processing.

**Nodes Involved:**  
- Download Resume File  
- Convert PDF to Text  
- Sticky Note - Download  
- Sticky Note - Extract

**Node Details:**

- **Download Resume File**  
  - Type: Google Drive  
  - Role: Downloads the actual resume file (PDF) given the file ID from trigger  
  - Configuration: Operation = download; File ID dynamically taken from trigger output  
  - Inputs: File metadata from "Watch for New Resumes"  
  - Outputs: Binary PDF file data  
  - Edge cases: File not found, permission errors, network timeouts  

- **Convert PDF to Text**  
  - Type: Extract From File  
  - Role: Extracts text content from PDF binary data  
  - Configuration: Operation = pdf extraction  
  - Inputs: Binary PDF data from download node  
  - Outputs: Text content of resume  
  - Edge cases: Scanned image PDFs will have poor extraction results; corrupted PDFs may fail  

- **Sticky Notes**  
  - Provide detailed explanations of these steps and tips for users  
  - Highlight reliance on text-based PDFs for best extraction quality

#### 1.3 AI Data Extraction

**Overview:**  
Uses an advanced GPT-4o AI model via Langchain agent to parse the resume text and extract structured candidate fields in JSON format.

**Nodes Involved:**  
- AI Resume Analyzer (Langchain Agent)  
- AI Language Model (Azure OpenAI GPT-4o-mini)  
- Sticky Note - AI Analysis

**Node Details:**

- **AI Resume Analyzer**  
  - Type: Langchain Agent  
  - Role: Defines the prompt and calls the AI language model to process resume text  
  - Configuration: Prompt instructs AI to extract fields — name, email, phone, experience years, skills array, current role, education — from resume text (passed dynamically)  
  - Inputs: Resume text string from PDF conversion  
  - Outputs: Raw AI JSON response string (wrapped in code fences)  
  - Edge cases: AI may produce malformed JSON, partial data, or hallucinated info; requires downstream cleaning  

- **AI Language Model**  
  - Type: Langchain LM Chat Azure OpenAI  
  - Role: Runs the GPT-4o-mini model to generate AI response  
  - Configuration: Model set to `gpt-4o-mini`, no additional options  
  - Credentials: Azure OpenAI API account  
  - Inputs: Prompt from AI Resume Analyzer  
  - Outputs: AI-generated text response  
  - Edge cases: API rate limits, authentication failures, timeouts  

- **Sticky Note**  
  - Details AI role and fields extracted  
  - Emphasizes AI’s ability to handle diverse resume formats

#### 1.4 Data Cleaning

**Overview:**  
Processes the AI raw JSON output to remove formatting artifacts and parse it into a clean JSON object suitable for database insertion.

**Nodes Involved:**  
- Clean AI Response (Code node)  
- Sticky Note - Clean Data

**Node Details:**

- **Clean AI Response**  
  - Type: Code (JavaScript)  
  - Role: Strips out markdown code fences (```json```), trims text, attempts safe JSON parse  
  - Configuration: Custom JS code that catches JSON parse errors and returns error info if parsing fails  
  - Inputs: Raw AI JSON string from AI Resume Analyzer  
  - Outputs: Parsed JSON object or error object  
  - Edge cases: Malformed AI output causing parse failure; returns error message for manual review  
  - Sub-workflow: None  

- **Sticky Note**  
  - Explains cleaning steps and rationale  
  - Recommends validation before saving data

#### 1.5 Database Storage

**Overview:**  
Appends or updates the cleaned candidate details in a centralized Google Sheets document acting as the candidate database.

**Nodes Involved:**  
- Save Candidate to Database (Google Sheets)  
- Sticky Note - Database

**Node Details:**

- **Save Candidate to Database**  
  - Type: Google Sheets  
  - Role: Append or update candidate record based on matching candidate name  
  - Configuration:  
    - Document ID: Google Sheet with ID `1JlXxy90s0we_IqErHyvomrJSijb8pd4H91hOUCH6xCA`  
    - Sheet: "Sheet1" (gid=0)  
    - Operation: appendOrUpdate (match on "Name" column with `$json.name`)  
    - Fields saved: Email, Phone, Years of experience, Skills, Current role, Education  
  - Inputs: Cleaned JSON with candidate fields  
  - Outputs: Confirmation of sheet update  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Duplicate names causing overwrite; Google API quota limits; auth expiration  

- **Sticky Note**  
  - Describes database role and data stored  
  - Highlights permanent record keeping and searchability

#### 1.6 Task Creation

**Overview:**  
Creates a new task in ClickUp corresponding to the candidate, assigning it to the hiring team to initiate the recruitment process.

**Nodes Involved:**  
- Create Hiring Task (ClickUp)  
- Sticky Note - Task Creation

**Node Details:**

- **Create Hiring Task**  
  - Type: ClickUp  
  - Role: Creates a new task titled “[Candidate Name] Hiring process” in a specific ClickUp list  
  - Configuration:  
    - Team: 9016683627  
    - Space: 90162844741  
    - Folder: 90164394824  
    - List: 901610812551  
    - Task name: dynamic from `$json.name` + " Hiring process"  
    - Assignees: user ID 95074494 (recruiting team member)  
  - Inputs: Candidate name from Google Sheets output  
  - Outputs: Created task metadata  
  - Credentials: ClickUp API OAuth2  
  - Edge cases: API failure, invalid team/list IDs, permission errors  

- **Sticky Note**  
  - Details task creation, assignee, and its role in recruitment pipeline

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                    | Input Node(s)             | Output Node(s)         | Sticky Note                                                                                              |
|-------------------------|---------------------------------|----------------------------------|---------------------------|------------------------|--------------------------------------------------------------------------------------------------------|
| Watch for New Resumes    | Google Drive Trigger             | Detect new resume uploads        | None                      | Download Resume File    | 🚀 WORKFLOW START: Monitors folder Resume_store every 1 minute for new files                            |
| Download Resume File     | Google Drive                    | Download resume PDF file          | Watch for New Resumes      | Convert PDF to Text     | 📥 STEP 1: Downloads resume PDF from Drive                                                             |
| Convert PDF to Text      | Extract From File                | Extract text from PDF             | Download Resume File       | AI Resume Analyzer      | 📄 STEP 2: Converts PDF to text, best with text-based PDFs                                             |
| AI Resume Analyzer       | Langchain Agent                 | Extract candidate info using AI  | Convert PDF to Text        | Clean AI Response       | 🤖 STEP 3: Extracts structured candidate fields from resume text using GPT-4o                          |
| AI Language Model        | Langchain LM Chat Azure OpenAI | Runs GPT-4o-mini model            | AI Resume Analyzer         | AI Resume Analyzer      | (No sticky note)                                                                                        |
| Clean AI Response        | Code (JavaScript)               | Clean and parse AI JSON output   | AI Resume Analyzer         | Save Candidate to Database | 🔧 STEP 4: Removes code fences, safely parses JSON, handles errors                                     |
| Save Candidate to Database | Google Sheets                  | Store/update candidate in sheet  | Clean AI Response          | Create Hiring Task      | 💾 STEP 5: Saves candidate data to Google Sheets, updates existing records                             |
| Create Hiring Task       | ClickUp                        | Create hiring task for candidate | Save Candidate to Database | None                   | ✅ STEP 6 - FINAL: Creates new ClickUp task assigned to hiring team                                   |
| Sticky Note - Trigger    | Sticky Note                    | Documentation                    | None                      | None                   | See details in node                                                                                     |
| Sticky Note - Download   | Sticky Note                    | Documentation                    | None                      | None                   | See details in node                                                                                     |
| Sticky Note - Extract    | Sticky Note                    | Documentation                    | None                      | None                   | See details in node                                                                                     |
| Sticky Note - AI Analysis | Sticky Note                   | Documentation                    | None                      | None                   | See details in node                                                                                     |
| Sticky Note - Clean Data | Sticky Note                    | Documentation                    | None                      | None                   | See details in node                                                                                     |
| Sticky Note - Database   | Sticky Note                    | Documentation                    | None                      | None                   | See details in node                                                                                     |
| Sticky Note - Task Creation | Sticky Note                 | Documentation                    | None                      | None                   | See details in node                                                                                     |
| Sticky Note - Overview   | Sticky Note                    | Documentation                    | None                      | None                   | Overview of entire workflow purpose and benefits                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger node:**
   - Type: Google Drive Trigger  
   - Configure to watch folder ID `1MIvpHU_ZqG76Vov2-D5WlS5dD3UhOMSz` ("Resume_store")  
   - Event: `fileCreated`  
   - Polling interval: every 1 minute  
   - Credential: Google Drive OAuth2 account  

2. **Create a Google Drive node to download file:**
   - Type: Google Drive  
   - Operation: `download`  
   - File ID: Reference from trigger output (`{{$json.id}}`)  
   - Credential: Google Drive OAuth2 account  
   - Connect output of trigger node here  

3. **Create an Extract From File node:**
   - Type: Extract From File  
   - Operation: `pdf`  
   - Input: Binary data from download node  
   - Connect output of download node here  

4. **Create a Langchain Agent node (AI Resume Analyzer):**
   - Type: Langchain Agent  
   - Prompt Type: `define`  
   - Text prompt to AI:
     ```
     Extract the following fields from the resume text below and return them as JSON only:

     {
       "name": "",
       "email": "",
       "phone": "",
       "experience_years": "",
       "skills": [],
       "current_role": "",
       "education": ""
     }

     Resume text:
     {{$json["text"]}}
     ```
   - Connect output of PDF text extraction node here  

5. **Create a Langchain LM Chat Azure OpenAI node:**
   - Type: Langchain LM Chat Azure OpenAI  
   - Model: `gpt-4o-mini`  
   - Credential: Azure OpenAI API account  
   - Connect output of Langchain Agent node here  

6. **Connect Langchain LM node output back to Langchain Agent node’s AI Language Model input:**
   - This connection enables Langchain Agent to use the LM node for processing  

7. **Create a Code node (Clean AI Response):**
   - Type: Code (JavaScript)  
   - Code:
     ```javascript
     return items.map(item => {
       let text = item.json.output;
       text = text.replace(/```json|```/g, "").trim();
       let parsed = {};
       try {
         parsed = JSON.parse(text);
       } catch (err) {
         parsed = { error: "Failed to parse JSON", raw: text };
       }
       return { json: parsed };
     });
     ```
   - Connect output of AI Resume Analyzer here  

8. **Create a Google Sheets node (Save Candidate to Database):**
   - Type: Google Sheets  
   - Operation: `appendOrUpdate`  
   - Document ID: `1JlXxy90s0we_IqErHyvomrJSijb8pd4H91hOUCH6xCA`  
   - Sheet Name: "Sheet1"  
   - Value To Match On: `{{$json.name}}`  
   - Column To Match On: `Name`  
   - Fields to save (mapped from input JSON):
     - Email → Email  
     - Phone → Phone  
     - Years of experience → experience_years  
     - Skills → skills  
     - Current role → current_role  
     - Education → education  
   - Credential: Google Sheets OAuth2 account  
   - Connect output of Clean AI Response node here  

9. **Create a ClickUp node (Create Hiring Task):**
   - Type: ClickUp  
   - Team ID: `9016683627`  
   - Space ID: `90162844741`  
   - Folder ID: `90164394824`  
   - List ID: `901610812551`  
   - Task Name: `{{$json.name}} Hiring process`  
   - Additional Fields: Assignees → `[95074494]`  
   - Credential: ClickUp API OAuth2 account  
   - Connect output of Google Sheets node here  

10. **Set up sticky notes for documentation at each key step (optional but recommended):**
    - Describe purpose and user tips for each block (trigger, download, extract, AI, clean, save, task creation).  
    - Use markdown for clarity.

11. **Validate credentials and API permissions:**
    - Google Drive: Folder access and file read  
    - Google Sheets: Edit permissions on the candidate database sheet  
    - Azure OpenAI: Access to GPT-4o-mini model  
    - ClickUp: Permissions to create tasks in specified workspace and assign members  

12. **Test workflow:**
    - Upload a sample PDF resume to the watched Google Drive folder  
    - Observe automatic processing, data extraction, database update, and task creation  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                        | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| To ensure accurate text extraction, resumes should be text-based PDFs rather than scanned images. OCR is not implemented in this workflow.                                        | Sticky Note - Extract                                                                                       |
| The AI model prompt is crafted to return JSON only, but occasional malformed outputs are handled gracefully by the cleaning code node.                                            | Sticky Note - AI Analysis, Clean AI Response                                                               |
| The Google Sheets database acts as a central, searchable HR repository updated automatically to prevent duplicate entries based on candidate name matching.                       | Sticky Note - Database                                                                                      |
| ClickUp task names follow the convention “[Candidate Name] Hiring process” and are automatically assigned to a recruiting team member with user ID 95074494.                      | Sticky Note - Task Creation                                                                                 |
| Workflow runtime is under 2 minutes per resume, significantly accelerating recruitment pipelines by automating repetitive tasks.                                                 | Sticky Note - Overview                                                                                      |
| Requires setup of OAuth2 credentials for Google Drive, Google Sheets, ClickUp, and Azure OpenAI in n8n prior to deployment.                                                        | Workflow prerequisites                                                                                      |
| For troubleshooting, monitor API quotas and error logs, especially for AI parsing errors and Google API rate limits.                                                              | Best practice note                                                                                          |
| More info on ClickUp API: https://clickup.com/api; Google Sheets API: https://developers.google.com/sheets/api; Azure OpenAI: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/ | Useful external resource links                                                                               |

---

This structured document fully explains the workflow’s design, nodes, configuration, and operational logic, enabling efficient reproduction, debugging, and extension by advanced users or automated agents.