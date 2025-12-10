Extract Meeting Tasks from Google Docs to GoHighLevel CRM with GPT-4

https://n8nworkflows.xyz/workflows/extract-meeting-tasks-from-google-docs-to-gohighlevel-crm-with-gpt-4-11535


# Extract Meeting Tasks from Google Docs to GoHighLevel CRM with GPT-4

---

### 1. Workflow Overview

This workflow automates the extraction of actionable tasks from Google Meet notes stored in Google Drive and creates corresponding tasks in the GoHighLevel CRM. It runs daily, organizing meeting files, extracting notes, leveraging AI (GPT-4) to identify tasks assigned to the user, and sending a detailed email summary.  

**Target Use Cases:**  
- Professionals who attend multiple meetings and want automated task extraction  
- Teams using Google Meet and Google Docs for meeting notes  
- CRM users leveraging GoHighLevel for task management  
- Users who want daily email summaries of meeting action items  

**Logical Blocks:**  
- **1.1 Scheduled Trigger & File Discovery:** Daily initiation and retrieval of meeting files from Google Drive  
- **1.2 File Categorization & Organization:** Sorting files into Recordings, Notes, and Chat folders  
- **1.3 Note Content Extraction:** Downloading and parsing Google Docs content  
- **1.4 AI Task Extraction & CRM Integration:** Using GPT-4 to identify tasks, find contacts, and create tasks in GoHighLevel  
- **1.5 Summary Generation & Email Notification:** Summarizing actions taken and emailing a formatted report to the user  

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & File Discovery

**Overview:**  
This block initiates the workflow daily and fetches all files from a specified Google Drive folder containing meeting records.

**Nodes Involved:**  
- DAILY ORGANIZATION TRIGGER  
- FIND MEETING RECORD FOLDER  

**Node Details:**  

- **DAILY ORGANIZATION TRIGGER**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow once per day automatically  
  - Configuration: Runs daily (interval with no specific time configured, defaults to daily)  
  - Inputs: None  
  - Outputs: Triggers "FIND MEETING RECORD FOLDER"  
  - Potential Failures: Trigger misconfiguration, n8n runtime downtime  

- **FIND MEETING RECORD FOLDER**  
  - Type: Google Drive node  
  - Role: Retrieves all files and folders inside a designated Google Drive folder that stores meeting records  
  - Configuration: Folder URL must be replaced with the user’s meeting records folder URL (parameter `folderId` from URL mode)  
  - Credentials: Google Drive OAuth2 required  
  - Input: Trigger from DAILY ORGANIZATION TRIGGER  
  - Output: All files/folders in meeting records folder, passed to SPLIT OUT NOTES, CHAT, RECORDINGS  
  - Edge Cases: Empty folder returns zero items; invalid folder URL or auth failure; API rate limits  

#### 1.2 File Categorization & Organization

**Overview:**  
This block categorizes retrieved files into Recordings, Notes, and Chat logs based on filename conventions and moves them into corresponding Google Drive folders.

**Nodes Involved:**  
- SPLIT OUT NOTES, CHAT, RECORDINGS  
- ROUTE FILE TYPES  
- MOVE RECORDINGS TO RECORDINGS FOLDER  
- MOVE NOTES TO NOTES FOLDER  
- MOVE CHAT TO CHAT FOLDER  
- COMBINE ALL  

**Node Details:**  

- **SPLIT OUT NOTES, CHAT, RECORDINGS**  
  - Type: Code node (JavaScript)  
  - Role: Parses files by name to separate recordings, notes, and chat logs; excludes the "Recordings" folder itself  
  - Logic:  
    - Files with names containing "- recording" → recordings  
    - Files with "- notes by gemini" → notes  
    - Files with "- chat" → chats  
  - Input: Files from FIND MEETING RECORD FOLDER  
  - Output: Multiple items with `type` property set accordingly  
  - Edge Cases: Files with unexpected naming; missing files; folder named "recordings" excluded  

- **ROUTE FILE TYPES**  
  - Type: Switch node  
  - Role: Routes items based on the `type` property (`recording`, `note`, `chat`) to different paths  
  - Input: Items from SPLIT OUT NOTES, CHAT, RECORDINGS  
  - Output: Three branches for recordings, notes, and chat  

- **MOVE RECORDINGS TO RECORDINGS FOLDER**  
  - Type: Google Drive node  
  - Role: Moves each recording file to a dedicated Recordings folder  
  - Configuration: Requires Recordings folder URL; uses file ID from input  
  - Credentials: Google Drive OAuth2  
  - Input: ROUTE FILE TYPES "RECORDINGS" output  
  - Output: Moved file info  

- **MOVE NOTES TO NOTES FOLDER**  
  - Type: Google Drive node  
  - Role: Moves note files to the Notes folder  
  - Configuration: Requires Notes folder URL  
  - Credentials: Google Drive OAuth2  
  - Input: ROUTE FILE TYPES "NOTES" output  

- **MOVE CHAT TO CHAT FOLDER**  
  - Type: Google Drive node  
  - Role: Moves chat files to the Chat folder  
  - Configuration: Requires Chat folder URL  
  - Credentials: Google Drive OAuth2  
  - Input: ROUTE FILE TYPES "CHAT" output  

- **COMBINE ALL**  
  - Type: Merge node  
  - Role: Combines outputs from the three move nodes back into a single stream for further processing  
  - Input: Outputs from the three MOVE nodes  

#### 1.3 Note Content Extraction

**Overview:**  
Downloads the content of note files from Google Docs and extracts clean text to prepare for AI processing.

**Nodes Involved:**  
- FILTER FOR NOTES  
- SUMMARIZE AND CREATE TASK LIST  
- GET NOTES  
- EXTRACT DATE AND CONTENT  
- PREP DATA  

**Node Details:**  

- **FILTER FOR NOTES**  
  - Type: Filter node  
  - Role: Passes only Google Docs files (`mimeType` = `application/vnd.google-apps.document`)  
  - Input: Combined files from COMBINE ALL  
  - Output: Only Google Docs meeting notes  

- **SUMMARIZE AND CREATE TASK LIST**  
  - Type: SplitInBatches node  
  - Role: Processes notes in batches to limit API or node workload  
  - Input: FILTER FOR NOTES output  
  - Output: Each batch triggers GET NOTES  

- **GET NOTES**  
  - Type: Google Docs node  
  - Role: Downloads full document content from Google Docs given the doc ID  
  - Credentials: Google Docs OAuth2  
  - Input: Document IDs from SUMMARIZE AND CREATE TASK LIST  
  - Output: Raw Google Docs JSON content  

- **EXTRACT DATE AND CONTENT**  
  - Type: Code node (JavaScript)  
  - Role: Parses Google Docs JSON to extract title and textual content, concatenates paragraphs, handles mentions and links, cleans whitespace  
  - Input: GET NOTES raw content  
  - Output: Cleaned text along with document title and formatted summary string  

- **PREP DATA**  
  - Type: Set node  
  - Role: Prepares data for the AI agent by renaming fields (e.g., summary)  
  - Input: EXTRACT DATE AND CONTENT output  
  - Output: Prepared data with a `summary` property for AI  

#### 1.4 AI Task Extraction & CRM Integration

**Overview:**  
Uses an AI agent powered by GPT-4 to identify tasks assigned to the user from meeting notes, find corresponding contacts in GoHighLevel CRM, and create actionable tasks with due dates.

**Nodes Involved:**  
- ORGANIZATION AGENT  
- OpenAI Chat Model  
- FIND CONTACT IN GHL  
- CREATE TASKS IN GHL  
- Simple Memory (for context retention)  

**Node Details:**  

- **ORGANIZATION AGENT**  
  - Type: Langchain AI Agent node  
  - Role: Core logic to extract action items from notes, identify contacts, and create tasks  
  - Configuration:  
    - System prompt instructs the agent to parse meeting notes, find "Leo" assigned tasks, identify contacts from meeting title, and use two tools: FIND CONTACT IN GHL and CREATE TASKS IN GHL  
    - Default due date is 3 business days if not specified  
    - Today's date injected dynamically  
  - Input: PREP DATA with summaries  
  - Output: Action items and contact-task mappings  
  - Edge Cases:  
    - No matching contact found → lists tasks without assignment  
    - Vague or missing action items are skipped  
    - API rate limits on AI or CRM calls  

- **OpenAI Chat Model**  
  - Type: Language model node (powered by GPT-5.1 in config, fallback GPT-4.1 in other instances)  
  - Role: Provides the AI processing for ORGANIZATION AGENT and SUMMARIZATION AGENT  
  - Credentials: OpenAI API key required  

- **FIND CONTACT IN GHL**  
  - Type: GoHighLevel CRM node  
  - Role: Searches contacts in GHL matching the extracted meeting participant(s)  
  - Configuration: Uses AI-generated query string for contact search  
  - Credentials: GoHighLevel OAuth2  
  - Input: From ORGANIZATION AGENT AI tool calls  

- **CREATE TASKS IN GHL**  
  - Type: GoHighLevel CRM node  
  - Role: Creates tasks under the identified contact with title, due date, and body details  
  - Configuration: Uses AI results for task title, due date, and description  
  - Credentials: GoHighLevel OAuth2  
  - Additional Field: `assignedTo` set to configured GoHighLevel user ID (must be replaced by user)  

- **Simple Memory**  
  - Type: Langchain memory node (Buffer Window)  
  - Role: Maintains session context during AI interaction for ORGANIZATION AGENT  
  - Session Key: Document ID from PREP DATA  
  - Purpose: Improves AI contextual understanding across batch processing  

#### 1.5 Summary Generation & Email Notification

**Overview:**  
Summarizes the outcome of task creation for all processed meetings and sends the user a well-formatted HTML email with details.

**Nodes Involved:**  
- COMBINE OUTPUTS  
- SUMMARIZATION AGENT  
- Simple Memory1  
- OpenAI Chat Model1  
- SEND EMAIL TO LEO  

**Node Details:**  

- **COMBINE OUTPUTS**  
  - Type: Aggregate node  
  - Role: Collects all task creation outputs for summary  
  - Input: Outputs from ORGANIZATION AGENT  

- **SUMMARIZATION AGENT**  
  - Type: Langchain AI Agent node  
  - Role: Analyzes the organization output and composes a detailed email summary  
  - Configuration:  
    - System message instructs to extract meeting count, tasks created per contact, due dates, and issues  
    - Generates subject line and contact-specific HTML blocks  
    - Uses SEND EMAIL TO LEO as tool to send email  
  - Input: COMBINE OUTPUTS aggregated data  
  - Output: Email content and metadata  

- **Simple Memory1**  
  - Type: Langchain memory node (Buffer Window) for summary context  
  - Session Key: "summary" (fixed key)  

- **OpenAI Chat Model1**  
  - Type: Language model node (GPT-4.1)  
  - Role: Generates the email summary text and HTML body  

- **SEND EMAIL TO LEO**  
  - Type: Gmail node  
  - Role: Sends the summary email to the user  
  - Configuration:  
    - Email address must be replaced by user’s email  
    - HTML message body dynamically filled by AI-generated content  
    - Subject set dynamically via AI output  
  - Credentials: Gmail OAuth2  

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                              | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                          |
|-------------------------------|--------------------------------|----------------------------------------------|----------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| DAILY ORGANIZATION TRIGGER     | Schedule Trigger               | Triggers workflow daily                       | None                             | FIND MEETING RECORD FOLDER             | 📁 File Organization                                                                              |
| FIND MEETING RECORD FOLDER     | Google Drive                  | Fetches all meeting files in Google Drive    | DAILY ORGANIZATION TRIGGER       | SPLIT OUT NOTES, CHAT, RECORDINGS     | 📁 File Organization                                                                              |
| SPLIT OUT NOTES, CHAT, RECORDINGS | Code                       | Categorizes files into recordings, notes, chat | FIND MEETING RECORD FOLDER       | ROUTE FILE TYPES                      | 📁 File Organization                                                                              |
| ROUTE FILE TYPES               | Switch                       | Routes files by type                          | SPLIT OUT NOTES, CHAT, RECORDINGS | MOVE RECORDINGS TO..., MOVE NOTES TO..., MOVE CHAT TO... | 📁 File Organization                                                                              |
| MOVE RECORDINGS TO RECORDINGS FOLDER | Google Drive            | Moves recording files to Recordings folder  | ROUTE FILE TYPES                 | COMBINE ALL                          | 📁 File Organization                                                                              |
| MOVE NOTES TO NOTES FOLDER     | Google Drive                  | Moves notes files to Notes folder             | ROUTE FILE TYPES                 | COMBINE ALL                          | 📁 File Organization                                                                              |
| MOVE CHAT TO CHAT FOLDER       | Google Drive                  | Moves chat files to Chat folder                | ROUTE FILE TYPES                 | COMBINE ALL                          | 📁 File Organization                                                                              |
| COMBINE ALL                   | Merge                        | Combines moved files into single stream       | MOVE RECORDINGS..., MOVE NOTES..., MOVE CHAT TO... | FILTER FOR NOTES                  |                                                                                                |
| FILTER FOR NOTES               | Filter                       | Filters only Google Docs meeting notes        | COMBINE ALL                     | SUMMARIZE AND CREATE TASK LIST        |                                                                                                |
| SUMMARIZE AND CREATE TASK LIST | SplitInBatches              | Processes notes in batches                      | FILTER FOR NOTES                | GET NOTES, COMBINE OUTPUTS             |                                                                                                |
| GET NOTES                     | Google Docs                  | Downloads Google Docs content                   | SUMMARIZE AND CREATE TASK LIST  | EXTRACT DATE AND CONTENT              |                                                                                                |
| EXTRACT DATE AND CONTENT      | Code                         | Extracts title and text from Google Docs JSON | GET NOTES                      | PREP DATA                           |                                                                                                |
| PREP DATA                    | Set                          | Prepares data with summary field                | EXTRACT DATE AND CONTENT        | ORGANIZATION AGENT                    |                                                                                                |
| ORGANIZATION AGENT            | Langchain AI Agent           | Extracts tasks, finds contacts, creates tasks | PREP DATA                     | SUMMARIZE AND CREATE TASK LIST         | 🤖 AI Task Extraction                                                                             |
| OpenAI Chat Model             | Language Model (OpenAI)      | Provides GPT-4 based AI processing             | ORGANIZATION AGENT (ai_languageModel) | ORGANIZATION AGENT                  | 🤖 AI Task Extraction                                                                             |
| FIND CONTACT IN GHL           | GoHighLevel Tool             | Searches CRM contacts                           | ORGANIZATION AGENT (ai_tool)    | ORGANIZATION AGENT (ai_tool)           | 🤖 AI Task Extraction                                                                             |
| CREATE TASKS IN GHL           | GoHighLevel Tool             | Creates tasks in CRM                            | ORGANIZATION AGENT (ai_tool)    | ORGANIZATION AGENT (ai_tool)           | 🤖 AI Task Extraction                                                                             |
| Simple Memory                 | Langchain Memory             | Maintains AI session context                    | PREP DATA                      | ORGANIZATION AGENT                     | 🤖 AI Task Extraction                                                                             |
| COMBINE OUTPUTS              | Aggregate                    | Aggregates task creation results                | ORGANIZATION AGENT             | SUMMARIZATION AGENT                    |                                                                                                |
| SUMMARIZATION AGENT           | Langchain AI Agent           | Summarizes task creation and composes email    | COMBINE OUTPUTS                | SEND EMAIL TO LEO                      | 📧 Email Summary                                                                                  |
| Simple Memory1                | Langchain Memory             | Maintains summary AI context                     | SUMMARIZATION AGENT            | SUMMARIZATION AGENT                    | 📧 Email Summary                                                                                  |
| OpenAI Chat Model1            | Language Model (OpenAI)      | GPT-4 based summary generation                   | SUMMARIZATION AGENT (ai_languageModel) | SUMMARIZATION AGENT                  | 📧 Email Summary                                                                                  |
| SEND EMAIL TO LEO             | Gmail Tool                   | Sends HTML email summary to user                 | SUMMARIZATION AGENT (ai_tool)  | None                                 | 📧 Email Summary                                                                                  |
| 📋 Main Documentation         | Sticky Note                  | Workflow overview and setup instructions         | None                          | None                                 |                                                                                                |
| 📁 File Organization Section  | Sticky Note                  | Explains file categorization and folder setup    | None                          | None                                 | Covers file organization nodes                                                                   |
| 🤖 AI Task Extraction Section | Sticky Note                  | Explains AI task extraction block and customization | None                          | None                                 | Covers AI extraction and CRM integration nodes                                                   |
| 📧 Email Summary Section      | Sticky Note                  | Explains email summary block and setup           | None                          | None                                 | Covers summary and email notification nodes                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `DAILY ORGANIZATION TRIGGER`  
   - Type: Schedule Trigger  
   - Set to run daily (default interval)  

2. **Create a Google Drive node to list files**  
   - Name: `FIND MEETING RECORD FOLDER`  
   - Type: Google Drive (File/Folder resource)  
   - Operation: List files in folder  
   - Parameters:  
     - Folder ID: Set using URL mode with your Meetings folder URL  
   - Credentials: Your Google Drive OAuth2 account  
   - Connect output of `DAILY ORGANIZATION TRIGGER` to this node  

3. **Create a Code node to split files by type**  
   - Name: `SPLIT OUT NOTES, CHAT, RECORDINGS`  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that sorts files by name into three arrays, tags each item with `type` property  
   - Connect output of `FIND MEETING RECORD FOLDER` to this node  

4. **Create a Switch node to route based on `type`**  
   - Name: `ROUTE FILE TYPES`  
   - Type: Switch  
   - Set 3 outputs: RECORDINGS, NOTES, CHAT  
   - Condition per output: `$json.type === 'recording'`, `'note'`, `'chat'` respectively  
   - Connect output of `SPLIT OUT NOTES, CHAT, RECORDINGS` to this node  

5. **Create three Google Drive nodes to move files to respective folders:**  
   - `MOVE RECORDINGS TO RECORDINGS FOLDER`  
     - Operation: Move file  
     - File ID: `{{$json.id}}`  
     - Folder ID: Your Recordings folder URL  
     - Credentials: Google Drive OAuth2  
     - Connect from ROUTE FILE TYPES RECORDINGS output  
   - `MOVE NOTES TO NOTES FOLDER`  
     - Same as above, with Notes folder URL  
     - Connect from ROUTE FILE TYPES NOTES output  
   - `MOVE CHAT TO CHAT FOLDER`  
     - Same as above, with Chat folder URL  
     - Connect from ROUTE FILE TYPES CHAT output  

6. **Create a Merge node to combine moved files**  
   - Name: `COMBINE ALL`  
   - Mode: Combine inputs (3 inputs)  
   - Connect outputs of the three MOVE nodes to this node  

7. **Create a Filter node to pass only Google Docs**  
   - Name: `FILTER FOR NOTES`  
   - Condition: `$json.mimeType === 'application/vnd.google-apps.document'`  
   - Connect output of `COMBINE ALL` to this node  

8. **Create a SplitInBatches node to batch process notes**  
   - Name: `SUMMARIZE AND CREATE TASK LIST`  
   - Batch size: Default (or as needed)  
   - Connect output of `FILTER FOR NOTES` to this node  

9. **Create a Google Docs node to fetch document content**  
   - Name: `GET NOTES`  
   - Operation: Get document content by document ID (`{{$json.id}}`)  
   - Credentials: Google Docs OAuth2  
   - Connect output of `SUMMARIZE AND CREATE TASK LIST` to this node  

10. **Create a Code node to extract and clean text content**  
    - Name: `EXTRACT DATE AND CONTENT`  
    - Paste provided JS code to parse Google Docs JSON and extract title and text  
    - Connect output of `GET NOTES` to this node  

11. **Create a Set node to prepare AI input**  
    - Name: `PREP DATA`  
    - Add field `summary` with value `{{$json.summary}}`  
    - Connect output of `EXTRACT DATE AND CONTENT` to this node  

12. **Create Langchain Agent node for AI task extraction**  
    - Name: `ORGANIZATION AGENT`  
    - Text prompt: Use the detailed prompt to instruct GPT-4 to extract tasks assigned to "Leo", find contacts in GoHighLevel, create tasks with due dates  
    - Set system message describing workflow steps and task extraction guidelines  
    - Configure to use OpenAI Chat Model (GPT-5.1 if available)  
    - Link AI tool calls to GoHighLevel `FIND CONTACT IN GHL` and `CREATE TASKS IN GHL` nodes (see below)  
    - Connect output of `PREP DATA` to this node  

13. **Create OpenAI Chat Model node**  
    - Name: `OpenAI Chat Model`  
    - Model: GPT-5.1 or GPT-4  
    - Credentials: OpenAI API key  
    - Connect AI language model input/output to `ORGANIZATION AGENT`  

14. **Create GoHighLevel nodes:**  
    - `FIND CONTACT IN GHL`  
      - Operation: Get all contacts with filter query from AI prompt  
      - Credentials: GoHighLevel OAuth2  
      - Connect AI tool call from `ORGANIZATION AGENT` to this node  
    - `CREATE TASKS IN GHL`  
      - Operation: Create task resource  
      - Parameters: Title, dueDate, contactId, body from AI prompt outputs  
      - Additional field: assignedTo → your GHL user ID  
      - Credentials: GoHighLevel OAuth2  
      - Connect AI tool call from `ORGANIZATION AGENT` to this node  

15. **Create Langchain Memory node for AI context**  
    - Name: `Simple Memory`  
    - Session Key: `{{$json.documentId}}` (custom key)  
    - Connect to `ORGANIZATION AGENT` AI memory input  

16. **Create Aggregate node to collect AI outputs**  
    - Name: `COMBINE OUTPUTS`  
    - Connect output of `SUMMARIZE AND CREATE TASK LIST` (AI results) to this node  

17. **Create Langchain Agent node for summary and email**  
    - Name: `SUMMARIZATION AGENT`  
    - Text prompt: Summarize tasks created, meeting count, issues, and format email content and subject  
    - Configure to use OpenAI Chat Model (GPT-4)  
    - Use SEND EMAIL TO LEO tool for sending email  
    - Connect output of `COMBINE OUTPUTS` to this node  

18. **Create Langchain Memory node for summary context**  
    - Name: `Simple Memory1`  
    - Session Key: `summary` (fixed)  
    - Connect to `SUMMARIZATION AGENT` AI memory input  

19. **Create OpenAI Chat Model node for summary**  
    - Name: `OpenAI Chat Model1`  
    - Model: GPT-4  
    - Credentials: OpenAI API key  
    - Connect AI language model input/output to `SUMMARIZATION AGENT`  

20. **Create Gmail node to send email**  
    - Name: `SEND EMAIL TO LEO`  
    - Send To: Your email address (replace placeholder)  
    - Subject and message body: dynamic, filled via AI outputs from `SUMMARIZATION AGENT`  
    - Credentials: Gmail OAuth2  
    - Connect AI tool call from `SUMMARIZATION AGENT` to this node  

21. **Connect nodes following the workflow logic:**  
    - DAILY ORGANIZATION TRIGGER → FIND MEETING RECORD FOLDER → SPLIT OUT NOTES, CHAT, RECORDINGS → ROUTE FILE TYPES → MOVE nodes → COMBINE ALL → FILTER FOR NOTES → SUMMARIZE AND CREATE TASK LIST → GET NOTES → EXTRACT DATE AND CONTENT → PREP DATA → ORGANIZATION AGENT → SUMMARIZE AND CREATE TASK LIST (loop) → COMBINE OUTPUTS → SUMMARIZATION AGENT → SEND EMAIL TO LEO  

22. **Replace all placeholder values:**  
    - Google Drive folder URLs for meeting records, recordings, notes, chat  
    - GoHighLevel user ID for task assignment  
    - Email address in Gmail node  
    - OAuth2 credentials for Google Drive, Google Docs, GoHighLevel, Gmail  
    - OpenAI API key  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                          | Context or Link                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| **Workflow Purpose:** Automatically extracts action items from Google Meet notes and creates tasks in GoHighLevel CRM, then emails a summary.                                                                                                                                                     | Main workflow description                                  |
| **File Organization Section:** Scans your Google Meet recordings folder and sorts files into Recordings, Notes, and Chat folders. Replace folder URLs accordingly.                                                                                                                                  | Sticky note near file organization nodes                   |
| **AI Task Extraction Section:** Uses GPT-4 to read meeting notes, find the contact in GoHighLevel, and create tasks with due dates. Customize the system prompt for task naming conventions.                                                                                                         | Sticky note near AI agent nodes                            |
| **Email Summary Section:** Sends a beautifully formatted HTML email displaying processed meetings, created tasks, due dates, and status. Update the recipient email in the Gmail tool.                                                                                                              | Sticky note near email nodes                               |
| **Task Naming Guidelines:** Extract precise, actionable tasks assigned to "Leo" only. Skip vague or unrelated items. Use clear titles and include context in task bodies.                                                                                                                           | System prompt in AI agent node                             |
| **Google Docs Content Extraction:** The code handles multiple content types including paragraphs, person mentions, and rich links, cleaning excessive newlines for clarity.                                                                                                                          | Code node EXTRACT DATE AND CONTENT                         |
| **OpenAI Model Versions:** GPT-5.1 is configured for main task extraction when available, GPT-4.1 used for summary generation. Ensure your OpenAI account supports these models or adjust accordingly.                                                                                                | OpenAI Chat Model nodes                                   |
| **OAuth2 Credentials:** Required for Google Drive, Google Docs, GoHighLevel, Gmail, and OpenAI API. Ensure scopes include file access, email sending, and CRM write permissions.                                                                                                                     | General setup requirement                                 |
| **Error Handling:** Watch for API rate limits, authorization failures, and missing or malformed documents. Logging or retry mechanisms are recommended in production deployments.                                                                                                                    | General operational notes                                 |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---