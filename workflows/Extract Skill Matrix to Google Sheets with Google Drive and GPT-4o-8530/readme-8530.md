Extract Skill Matrix to Google Sheets with Google Drive and GPT-4o

https://n8nworkflows.xyz/workflows/extract-skill-matrix-to-google-sheets-with-google-drive-and-gpt-4o-8530


# Extract Skill Matrix to Google Sheets with Google Drive and GPT-4o

### 1. Workflow Overview

This workflow automates the extraction of a structured skill matrix from PDF resumes stored in a Google Drive folder, leveraging Azure OpenAI GPT-4o for natural language processing, and finally storing the extracted data in a Google Sheets spreadsheet. It is designed for HR or recruitment teams who want to efficiently analyze resumes for specific technical skills and maintain a searchable, up-to-date skills database.

**Logical Blocks:**

- **1.1 Input Reception and File Retrieval:** Trigger the process manually and search for resumes in a specified Google Drive folder.
- **1.2 File Download and Content Extraction:** Download each resume PDF and extract its textual content.
- **1.3 AI Skill Analysis:** Pass extracted text through the Azure OpenAI GPT-4o-mini model to analyze and extract skills structured in JSON.
- **1.4 Post-Processing and Filtering:** Parse and filter the AI output to retain only skills with proficiency level > 2.
- **1.5 Data Storage:** Append or update the filtered skills matrix into a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Retrieval

**Overview:**  
Starts the workflow via manual trigger and searches a dedicated Google Drive folder ("Resume_store") for all resume files.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Search Resume (Google Drive Search)  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually when user clicks “Execute workflow”.  
  - Configuration: Default manual trigger with no parameters.  
  - Input: None  
  - Output: Trigger signal to start the search process.  
  - Edge cases: Workflow not starting if user does not trigger manually.  

- **Search Resume**  
  - Type: Google Drive (File/Folder Search)  
  - Role: Lists all files within a specific folder identified by folder ID (1MIvpHU_ZqG76Vov2-D5WlS5dD3UhOMSz).  
  - Configuration: Searches inside folder “Resume_store” for all files, returns all results.  
  - Credentials: Google Drive OAuth2 (secured access).  
  - Input: Trigger from manual node.  
  - Output: List of file metadata (including file IDs) for resumes found.  
  - Edge cases: Empty folder returns no files; API quota or permission errors may block access.

---

#### 2.2 File Download and Content Extraction

**Overview:**  
Downloads each resume file found and extracts text content from the PDF document.

**Nodes Involved:**  
- Download Resume (Google Drive Download)  
- Extract content from File (PDF Text Extraction)  

**Node Details:**

- **Download Resume**  
  - Type: Google Drive (File Download)  
  - Role: Downloads the actual PDF file using file IDs from the previous search.  
  - Configuration: Uses dynamic expression to pull file ID from Search Resume output.  
  - Credentials: Google Drive OAuth2.  
  - Input: File IDs from Search Resume.  
  - Output: Binary PDF file data for each resume.  
  - Edge cases: Download failure if file is deleted, inaccessible, or network issues.

- **Extract content from File**  
  - Type: Extract from File (PDF Operation)  
  - Role: Extracts plain text from the downloaded PDF files.  
  - Configuration: Operation set to “pdf” to extract all text content.  
  - Input: Binary PDF data.  
  - Output: Text content of each resume.  
  - Edge cases: Corrupted PDFs or scanned images without text layer may yield no content.

---

#### 2.3 AI Skill Analysis

**Overview:**  
Uses Azure OpenAI GPT-4o-mini to analyze extracted resume text and extract a structured JSON skills matrix focused on a predefined tech stack.

**Nodes Involved:**  
- Azure OpenAI Chat Model  
- Skill analyser (Langchain AI Agent)  

**Node Details:**

- **Azure OpenAI Chat Model**  
  - Type: Langchain Azure OpenAI Chat Model  
  - Role: Provides the GPT-4o-mini model as an AI backend for language understanding and generation.  
  - Configuration: Model set to “gpt-4o-mini” with default options.  
  - Credentials: Azure Open AI API credentials securely configured.  
  - Input: Triggered by extracted text feeding into AI agent.  
  - Output: AI-generated response containing JSON skills data.  
  - Edge cases: API rate limits, authentication failures, or model unavailability.

- **Skill analyser**  
  - Type: Langchain Agent  
  - Role: Sends prompt to the GPT-4o-mini model to analyze resume text and extract skills.  
  - Configuration: Prompt instructs the AI to identify skills only from a predefined tech stack (React, Node.js, Angular, Python, Java, SQL, Docker, Kubernetes, AWS, Azure, GCP, HTML, CSS, JavaScript), rating skill level (1–5), and years of experience (integer or null). AI must return valid JSON only.  
  - Input: Plain text resume content.  
  - Output: AI response string with JSON containing skills array.  
  - Edge cases: AI returning malformed JSON, irrelevant or partial data.

---

#### 2.4 Post-Processing and Filtering

**Overview:**  
Processes AI output JSON, parses it, filters out skills with level ≤ 2, and structures data for storage.

**Nodes Involved:**  
- Parse structure json (Code Node)  

**Node Details:**

- **Parse structure json**  
  - Type: Code (JavaScript)  
  - Role: Parses the AI JSON string output, extracts skills, filters by proficiency level (> 2), and formats as array of JSON objects.  
  - Configuration: JavaScript code parses AI text, handles JSON parse errors by throwing exceptions, filters, then maps output.  
  - Input: AI JSON string from Skill analyser.  
  - Output: Array of skill objects with name, level, and years properties.  
  - Edge cases: Invalid JSON triggers error, no skills or empty arrays handled gracefully.

---

#### 2.5 Data Storage

**Overview:**  
Appends or updates the filtered skills matrix into a designated Google Sheets spreadsheet under “Sheet2” tab, associating all skills with a static candidate name.

**Nodes Involved:**  
- Update sheet with skill matrix (Google Sheets)  

**Node Details:**

- **Update sheet with skill matrix**  
  - Type: Google Sheets (Append or Update)  
  - Role: Stores each skill as a row with columns: Name ("John Doe"), Skill name, Skill level, Skill years.  
  - Configuration:  
    - Document ID set to a specific Google Sheets file (“Resume store”).  
    - Sheet name set to “Sheet2”.  
    - Mapping defines explicit columns and uses “Name” as the matching column for updates.  
    - Operation set to “appendOrUpdate” for idempotent inserts.  
  - Credentials: Google Sheets OAuth2.  
  - Input: Filtered skill objects.  
  - Output: Confirmation of rows added/updated.  
  - Edge cases: API permission errors, schema mismatches, or connectivity issues.

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                                | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                         |
|------------------------------|-----------------------------------|------------------------------------------------|----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                    | Starts workflow execution                       | —                          | Search Resume                 | **Starts the workflow execution**. Click “Execute workflow” to begin. Run when new resumes are added.                |
| Search Resume                | Google Drive (File/Folder Search) | Finds all resume files in the “Resume_store” folder | When clicking ‘Execute workflow’ | Download Resume               | Locates resume files in designated Google Drive folder “Resume_store”. Outputs list of files.                        |
| Download Resume              | Google Drive (File Download)       | Downloads each resume PDF file                  | Search Resume               | Extract content from File     | Downloads resume files from Google Drive using secure credentials.                                                  |
| Extract content from File    | Extract from File (PDF)             | Extracts readable text from PDFs                | Download Resume             | Skill analyser                | Converts PDF resumes into plain text content for analysis.                                                          |
| Azure OpenAI Chat Model      | Langchain Azure OpenAI Chat Model  | Provides GPT-4o-mini AI model                    | — (used internally by Skill analyser) | Skill analyser         | Supplies AI language model with GPT-4o-mini via secure Azure OpenAI credentials.                                    |
| Skill analyser               | Langchain Agent                    | Analyzes resume text for skills and proficiency| Extract content from File   | Parse structure json          | AI analyzes resumes focusing on specific tech stack and outputs structured JSON skills matrix.                      |
| Parse structure json         | Code (JavaScript)                  | Parses AI JSON, filters skills level > 2       | Skill analyser              | Update sheet with skill matrix | Parses and filters AI output JSON, handles errors, and structures data for storage.                                 |
| Update sheet with skill matrix | Google Sheets (Append or Update)  | Stores skill matrix in Google Sheets            | Parse structure json        | —                             | Appends or updates skills to “Resume store” Google Sheet, “Sheet2” tab, mapping skill data and hardcoded candidate. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named “When clicking ‘Execute workflow’”.  
   - No parameters needed. This node will start the workflow when manually executed.

2. **Add Google Drive Search Node**  
   - Add a Google Drive node named “Search Resume”.  
   - Set Resource to “File/Folder”.  
   - Operation: Search.  
   - Configure Filter to search files inside the folder with ID: `1MIvpHU_ZqG76Vov2-D5WlS5dD3UhOMSz` (the “Resume_store” folder).  
   - Enable “Return All” to get all files.  
   - Set Google Drive OAuth2 credentials with appropriate access to the folder.  
   - Connect the Manual Trigger node output to this node input.

3. **Add Google Drive Download Node**  
   - Add another Google Drive node named “Download Resume”.  
   - Set Resource to “File”.  
   - Operation: Download.  
   - File ID: Use expression to get current item’s file ID: `={{ $json.id }}`.  
   - Use the same Google Drive OAuth2 credentials.  
   - Connect “Search Resume” node output to this node input.

4. **Add Extract from File Node**  
   - Add an “Extract from File” node named “Extract content from File”.  
   - Operation: Set to “pdf” to extract text.  
   - Connect “Download Resume” node output to this node input.  
   - No additional credentials needed.

5. **Add Azure OpenAI Chat Model Node**  
   - Add a Langchain Azure OpenAI Chat Model node named “Azure OpenAI Chat Model”.  
   - Select model: “gpt-4o-mini”.  
   - Use Azure OpenAI API credentials with correct access.  
   - No direct input connection needed here; it will be used internally by the agent node.

6. **Add Langchain Agent Node for Skill Analysis**  
   - Add a Langchain Agent node named “Skill analyser”.  
   - Set prompt type to “define”.  
   - Prompt text:  
     ```
     Analyze the following resume and extract the skills into the required JSON format.

     Resume:
     {{ $json.text }}
     ```  
   - System message:  
     ```
     You are an AI assistant that analyzes resumes and extracts a structured skills matrix.  

     Rules:
     - Only include skills relevant to this predefined tech stack: [React, Node.js, Angular, Python, Java, SQL, Docker, Kubernetes, AWS, Azure, GCP, HTML, CSS, JavaScript].  
     - For each skill, return:
       - name (string)
       - level (1–5, where 1 = beginner and 5 = expert)
       - years (integer, number of years of experience if mentioned, otherwise estimate or set to null).
     - Always return valid JSON in this exact format:
     {
       "skills": [
         {"name": "React", "level": 4, "years": 5},
         {"name": "Node.js", "level": 3, "years": 2}
       ]
     }
     - Do not include text outside of the JSON.
     ```  
   - Connect “Extract content from File” node output to this node input.  
   - Select the Azure OpenAI Chat Model node under AI language model credentials.

7. **Add Code Node to Parse and Filter JSON**  
   - Add Code node named “Parse structure json”.  
   - Paste the following JavaScript code:
     ```js
     const rawOutput = $json["output"];
     let parsed;
     try {
       parsed = JSON.parse(rawOutput);
     } catch (e) {
       throw new Error("Invalid JSON from AI: " + e.message);
     }
     let skills = parsed.skills || [];
     skills = skills.filter(skill => skill.level > 2);
     return skills.map(skill => ({ json: skill }));
     ```  
   - Connect “Skill analyser” node output to this node input.

8. **Add Google Sheets Node to Store Skills**  
   - Add Google Sheets node named “Update sheet with skill matrix”.  
   - Operation: Append or Update.  
   - Document ID: Set to the target Google Sheets document ID: `1JlXxy90s0we_IqErHyvomrJSijb8pd4H91hOUCH6xCA` (the “Resume store” spreadsheet).  
   - Sheet Name: Set to “Sheet2” (tab ID 1424038785).  
   - Define columns mapping as:  
     - Name: Hardcoded string “John Doe”  
     - Skill name: `={{ $json.name }}`  
     - Skill level: `={{ $json.level }}`  
     - Skill years: `={{ $json.years }}`  
   - Use “Name” as the matching column for update operations.  
   - Use Google Sheets OAuth2 credentials with write access.  
   - Connect “Parse structure json” node output to this node input.

9. **Connect Nodes in Order:**  
   Manual Trigger → Search Resume → Download Resume → Extract content from File → Skill analyser → Parse structure json → Update sheet with skill matrix.

10. **Test the Workflow:**  
    - Execute the manual trigger.  
    - Verify that resumes are found, downloaded, processed, analyzed, filtered, and stored in Google Sheets correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow is designed to analyze resumes only for a predefined tech stack to maintain consistent skill matrix data.                     | Internal project context                                                                            |
| Prompt instructions require AI to return strictly valid JSON to avoid parsing errors.                                                   | Critical for stable JSON parsing in code node                                                     |
| Google Drive folder and Google Sheets document are specific to this project and require appropriate OAuth2 credentials with access.    | Ensure credentials have permission to read/write in these services                                |
| “John Doe” is a hardcoded candidate name; modify if dynamic candidate names are required.                                                | Candidate name field in Google Sheets node                                                        |
| For more about integrating GPT models in n8n using Langchain nodes, see: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/ | Documentation linked in n8n official resources                                                    |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies without illegal, offensive, or protected elements. All data processed is legal and public.