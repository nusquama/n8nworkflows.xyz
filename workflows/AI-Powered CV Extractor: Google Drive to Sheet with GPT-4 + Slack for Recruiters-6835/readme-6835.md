AI-Powered CV Extractor: Google Drive to Sheet with GPT-4 + Slack for Recruiters

https://n8nworkflows.xyz/workflows/ai-powered-cv-extractor--google-drive-to-sheet-with-gpt-4---slack-for-recruiters-6835


# AI-Powered CV Extractor: Google Drive to Sheet with GPT-4 + Slack for Recruiters

### 1. Workflow Overview

This workflow, titled **"AI-Powered CV Extractor: Google Drive to Sheet with GPT-4 + Slack for Recruiters"**, automates the processing of candidate resumes uploaded or updated in a specific Google Drive folder. Its primary purpose is to extract detailed candidate information from resume PDFs using GPT-4 AI, structure this data, and maintain an up-to-date centralized talent database in Google Sheets. Additionally, it notifies the hiring team via Slack whenever a profile is processed.

The workflow is designed for HR and Talent Acquisition (TA) teams who seek to reduce manual administrative work by automating CV parsing, data extraction, and candidate tracking.

**Logical Blocks:**

- **1.1 Trigger Block:** Detects new or updated resume files in a Google Drive folder every 5 minutes.
- **1.2 Profile Retrieval and Extraction:** Downloads the resume file and extracts raw text from PDFs.
- **1.3 AI-based Profile Analysis:** Sends extracted text to GPT-4 to produce a structured candidate profile.
- **1.4 Data Validation:** Ensures the extracted profile contains essential data (e.g., email).
- **1.5 Data Transformation:** Converts the structured profile into a row format suitable for Google Sheets.
- **1.6 Data Upsert and Notification:** Inserts or updates the candidate record in Google Sheets and sends a Slack notification to the hiring team.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

**Overview:**  
Monitors a specific Google Drive folder for any new resume uploads or updates. Triggers the workflow every 5 minutes upon detection.

**Nodes Involved:**  
- New profile uploaded to drive folder  
- Existing profile updated  
- Sticky Note1

**Node Details:**

- **New profile uploaded to drive folder**  
  - *Type:* Google Drive Trigger  
  - *Role:* Triggers workflow on new file creation inside the designated folder.  
  - *Configuration:* Polls every 5 minutes; watches folder ID `1J9GnGlJ70ppf88eZGrhLIo3jxIaqT1u7` (the CV folder).  
  - *Inputs:* None (trigger node).  
  - *Outputs:* File metadata forwarded to "Get profile detail".  
  - *Failure types:* Google API auth errors, quota limits, folder permissions issues.

- **Existing profile updated**  
  - *Type:* Google Drive Trigger  
  - *Role:* Similar to above; triggers on file updates (edits) in the same folder.  
  - *Configuration:* Same polling and folder as upload trigger.  
  - *Inputs:* None.  
  - *Outputs:* File metadata forwarded to "Get profile detail".  
  - *Failure types:* Same as upload trigger.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Documentation for trigger behavior and schedule.

---

#### 2.2 Profile Retrieval and Extraction

**Overview:**  
Downloads the triggered resume file from Google Drive and extracts its textual content using PDF extraction.

**Nodes Involved:**  
- Get profile detail  
- Extract profile information  
- Sticky Note2

**Node Details:**

- **Get profile detail**  
  - *Type:* Google Drive (Download file)  
  - *Role:* Downloads the actual resume PDF using the file URL from trigger.  
  - *Configuration:* Uses fileId from the input JSON’s `webViewLink`.  
  - *Inputs:* File metadata from trigger nodes.  
  - *Outputs:* Binary PDF file forwarded to "Extract profile information".  
  - *Failure types:* File permissions, invalid file ID, network errors.

- **Extract profile information**  
  - *Type:* Extract From File  
  - *Role:* Extracts raw text content from the downloaded PDF.  
  - *Configuration:* Operation set to PDF extraction.  
  - *Inputs:* Binary PDF from "Get profile detail".  
  - *Outputs:* Extracted text forwarded to "Profile Analyzer Agent".  
  - *Failure types:* Corrupt PDFs, unsupported PDF formats, extraction errors.

- **Sticky Note2**  
  - *Type:* Sticky Note  
  - *Role:* Documents this extraction block.

---

#### 2.3 AI-based Profile Analysis

**Overview:**  
Processes raw extracted text with a GPT-4 based agent to generate a structured candidate profile JSON.

**Nodes Involved:**  
- Profile Analyzer Agent  
- gpt4-1 model  
- json parser  
- Sticky Note5

**Node Details:**

- **Profile Analyzer Agent**  
  - *Type:* Langchain Agent Node  
  - *Role:* Sends extracted CV text to GPT-4 with a prompt to extract relevant candidate info.  
  - *Configuration:* Prompt includes extracted text under "CV Content". Output parser enabled to structure response.  
  - *Inputs:* Raw text from "Extract profile information".  
  - *Outputs:* Structured JSON candidate profile sent to "Verify profile email".  
  - *Failure types:* GPT API errors, prompt misconfiguration, rate-limits.

- **gpt4-1 model**  
  - *Type:* OpenAI Chat Language Model (GPT-4.1-mini)  
  - *Role:* Language model used by the agent for text processing.  
  - *Configuration:* Model parameter set to "gpt-4.1-mini". OpenAI API credentials configured.  
  - *Inputs:* Text prompt from agent node.  
  - *Outputs:* AI response forwarded to "json parser".  
  - *Failure types:* API key expiration, network issues.

- **json parser**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Validates and parses GPT output into a structured JSON with predefined schema for candidate profile fields.  
  - *Configuration:* Uses detailed JSON schema including name, contact, education, experience, skills, projects, awards, and more.  
  - *Inputs:* GPT-4 raw output.  
  - *Outputs:* Clean JSON profile sent to "Verify profile email".  
  - *Failure types:* Parsing errors if GPT output doesn't match schema.

- **Sticky Note5**  
  - *Type:* Sticky Note  
  - *Role:* Documents the AI analysis block.

---

#### 2.4 Data Validation

**Overview:**  
Filters out any profiles lacking an essential contact email, ensuring only valid candidate data proceeds.

**Nodes Involved:**  
- Verify profile email

**Node Details:**

- **Verify profile email**  
  - *Type:* Filter node  
  - *Role:* Validates that the extracted profile contains a non-empty email address.  
  - *Configuration:* Condition checks if `output.contact.email` is not empty.  
  - *Inputs:* Structured candidate profile JSON from "Profile Analyzer Agent".  
  - *Outputs:* Valid profiles continue to "Transform output"; invalid profiles stop flow here.  
  - *Failure types:* Missing or malformed email in profile JSON.

---

#### 2.5 Data Transformation

**Overview:**  
Transforms the structured profile JSON into a flattened object tailored for Google Sheets columns, including formatting dates and summarizing experience.

**Nodes Involved:**  
- Transform output  
- Sticky Note3  
- Sticky Note6

**Node Details:**

- **Transform output**  
  - *Type:* Code (JavaScript) node  
  - *Role:* Maps profile JSON fields to sheet columns, formats dates, calculates total experience years, and compiles education details into strings.  
  - *Key logic:*  
    - Extracts latest job experience.  
    - Formats start and end dates into "YYYY-MM" format.  
    - Calculates total years of experience.  
    - Formats education entries into readable strings.  
    - Limits skills to top 3 technical skills.  
    - Includes profile URL from Google Drive for reference.  
  - *Inputs:* Validated profile JSON from "Verify profile email".  
  - *Outputs:* Flattened JSON object for Google Sheets.  
  - *Failure types:* JavaScript errors, missing profile fields.

- **Sticky Note3 & Sticky Note6**  
  - *Type:* Sticky Notes  
  - *Role:* Visual aids/screenshots related to the transformation step.

---

#### 2.6 Data Upsert and Notification

**Overview:**  
Inserts or updates the candidate profile row in a Google Sheet and sends a Slack notification summarizing the action.

**Nodes Involved:**  
- Append or update row in sheet  
- Let the hiring team know  
- Sticky Note4  
- Sticky Note7  
- Sticky Note (unnamed but positioned near this block)

**Node Details:**

- **Append or update row in sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Upserts candidate data into Google Sheet using email as unique key to insert new or update existing rows.  
  - *Configuration:*  
    - Document ID and sheet name configured for "Processed Candidates" sheet.  
    - Columns auto-mapped from input data.  
    - Matching column set to "Email".  
  - *Inputs:* Flattened profile JSON from "Transform output".  
  - *Outputs:* Confirmation sent to "Let the hiring team know".  
  - *Failure types:* Sheet permission errors, exceeded API quotas, malformed data.

- **Let the hiring team know**  
  - *Type:* Slack node  
  - *Role:* Posts a message to the designated Slack channel notifying that a profile has been processed and recorded.  
  - *Configuration:*  
    - Uses OAuth2 Slack credentials.  
    - Posts to channel ID "C098LDNAG1E" with message including candidate’s full name.  
  - *Inputs:* Confirmation from Google Sheets node.  
  - *Outputs:* Workflow ends.  
  - *Failure types:* Slack API permission errors, invalid token, network issues.

- **Sticky Note4 & Sticky Note7**  
  - *Type:* Sticky Notes  
  - *Role:* Provide overview and screenshots for the final saving and notification process.

- **Sticky Note (Unnamed near output block)**  
  - *Type:* Sticky Note  
  - *Role:* Explains the overall workflow purpose, setup, customization, and target users in a detailed markdown format.

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                        | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                          |
|-------------------------------|---------------------------------------|-------------------------------------|---------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| New profile uploaded to drive folder | Google Drive Trigger                  | Trigger on new file upload           | None                            | Get profile detail               | Triggers workflow every 5 mins on new file in Google Drive folder                                  |
| Existing profile updated       | Google Drive Trigger                  | Trigger on file update               | None                            | Get profile detail               | Triggers workflow every 5 mins on updated file in Google Drive folder                              |
| Get profile detail             | Google Drive (Download file)          | Download resume PDF                  | New profile uploaded, Existing profile updated | Extract profile information      | Downloads the resume file based on file URL                                                        |
| Extract profile information    | Extract From File                     | Extract raw text from PDF            | Get profile detail              | Profile Analyzer Agent           | Downloads and extracts PDF content                                                                 |
| Profile Analyzer Agent         | Langchain Agent Node                  | AI extraction of candidate info      | Extract profile information     | Verify profile email             | Uses GPT-4 to extract structured profile from raw text                                            |
| gpt4-1 model                  | OpenAI GPT-4 Model                    | Language model for AI extraction     | Profile Analyzer Agent (inner)  | Json parser                     | GPT-4.1-mini model used for text processing                                                        |
| json parser                   | Langchain Structured Output Parser   | Validate/parse structured AI output  | gpt4-1 model                   | Profile Analyzer Agent           | Parses AI response to strict JSON schema                                                          |
| Verify profile email          | Filter                              | Validate presence of email           | Profile Analyzer Agent           | Transform output                | Filters profiles missing email address                                                            |
| Transform output             | Code (JavaScript)                   | Flatten and format profile data      | Verify profile email            | Append or update row in sheet    | Maps fields, formats dates, calculates experience, prepares data for Google Sheets                 |
| Append or update row in sheet | Google Sheets                        | Insert/update profile in Google Sheet | Transform output               | Let the hiring team know         | Upserts candidate data using email as unique key                                                  |
| Let the hiring team know      | Slack                              | Notify hiring team via Slack         | Append or update row in sheet   | None                           | Sends notification message to Slack channel                                                       |
| Sticky Note1                 | Sticky Note                         | Documentation for trigger block      | None                           | None                           | Trigger workflow when new profile upload/edit (every 5 mins)                                      |
| Sticky Note2                 | Sticky Note                         | Documentation for extraction block   | None                           | None                           | Extract profile information from Google Drive PDF                                                 |
| Sticky Note3                 | Sticky Note                         | Visual aid for transformation block  | None                           | None                           | Screenshot related to transformation step                                                        |
| Sticky Note4                 | Sticky Note                         | Visual aid for upsert & notification | None                           | None                           | Screenshot related to final save and notification                                                |
| Sticky Note5                 | Sticky Note                         | Documentation for profile analyzer   | None                           | None                           | Smart agent extracts information from candidate profile                                          |
| Sticky Note6                 | Sticky Note                         | Visual aid for transformation block  | None                           | None                           | Screenshot related to transformation step                                                        |
| Sticky Note7                 | Sticky Note                         | Workflow overview and setup guide    | None                           | None                           | Detailed markdown covering workflow purpose, users, setup, and customization                      |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Google Drive Triggers**  
- Add two Google Drive Trigger nodes:  
  - One configured for `fileCreated` event, polling every 5 minutes, watching the target folder by its ID (e.g., `/SmartHR/cv/`).  
  - One configured for `fileUpdated` event, same polling and folder.  
- Ensure Google Drive OAuth2 credentials are connected.

**Step 2: Download Resume File**  
- Add a Google Drive node "Get profile detail" that downloads the file based on the `webViewLink` URL from the trigger.  
- Connect both trigger nodes to this node.  
- Use the same Google Drive OAuth2 credentials.

**Step 3: Extract Text from PDF**  
- Add an "Extract From File" node set to operate on PDFs.  
- Connect it to the output of "Get profile detail".

**Step 4: Setup GPT-4 AI Profile Extraction**  
- Add a Langchain Agent node "Profile Analyzer Agent".  
- Set its prompt to:  
  ```
  Please extract all relevant information from this candidate:
  CV Content:
  ===
  {{ $json["text"] }}
  ===
  ```  
- Enable output parser with a JSON schema covering candidate details (full name, contact info, education, work experience, skills, projects, awards, volunteer experience, additional info).  
- Add a Langchain OpenAI node "gpt4-1 model" configured with model `gpt-4.1-mini`.  
- Connect the Langchain model node as the LM inside the agent node. Provide OpenAI API credentials.

**Step 5: Parse and Validate AI Output**  
- Add a Langchain Structured Output Parser node "json parser" with the detailed schema matching AI output.  
- Connect the GPT-4 model node to this parser.  
- Connect parser output to the "Profile Analyzer Agent" node’s output port.  
- Add a Filter node "Verify profile email" to check that `output.contact.email` is non-empty.  
- Connect "Profile Analyzer Agent" output to the filter node.

**Step 6: Transform Profile Data**  
- Add a Code node "Transform output" with JavaScript that:  
  - Extracts the latest job experience.  
  - Formats dates to "YYYY-MM".  
  - Calculates total years of experience.  
  - Formats education as readable strings.  
  - Limits skills to top 3 technical skills.  
  - Adds profile URL from Drive.  
- Connect filter node’s valid output to this node.

**Step 7: Append/Update Google Sheet Row**  
- Add a Google Sheets node "Append or update row in sheet".  
- Configure document ID and sheet name for the candidate database sheet.  
- Enable auto-mapping of input JSON fields to columns.  
- Set matching column to "Email" to enable upsert behavior.  
- Connect "Transform output" node to this node.  
- Use Google Sheets OAuth2 credentials.

**Step 8: Notify Hiring Team via Slack**  
- Add a Slack node "Let the hiring team know".  
- Set message text to: `{{ $json.FullName }} profile has been processed and recorded in Google Drive!`  
- Choose the Slack channel for notifications.  
- Use Slack OAuth2 credentials.  
- Connect Google Sheets node output to Slack node.

**Step 9: Add Sticky Notes for Documentation**  
- Add Sticky Note nodes with content describing each block: trigger, extraction, AI processing, validation, transformation, and notification.  
- Optionally add screenshots or visual aids relevant to each block.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                                |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates CV processing to keep candidate data clean, current, and centralized, saving hours of manual data entry work for HR and TA teams. It leverages GPT-4 to extract candidate profiles from PDFs stored in Google Drive and syncs the data into a Google Sheet database, with Slack notifications keeping recruiters informed.                                                                                                                                                                                                                                                                                                                                                                                                                         | Overview sticky note (Sticky Note7)                                                                                                         |
| Setup reminders: create a Google Drive folder (e.g., `/SmartHR/cv/`), prepare a Google Sheet with columns like Email, FullName, JobTitle, Phone, Location, Skills, and set proper API credentials for Google Drive, Google Sheets, OpenAI, and Slack.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Setup instructions from Sticky Note7                                                                                                       |
| Customize the GPT prompt to extract data tailored to different job roles or seniority levels. The field mapping in the transformation can be extended to include LinkedIn, portfolio links, or other candidate info. Notifications can be switched to other platforms such as Microsoft Teams or email. The data store can be replaced with Airtable, Notion, or a SQL database for scalability.                                                                                                                                                                                                                                                                                                                                                                              | Customization options from Sticky Note7                                                                                                    |
| Slack notification requires OAuth2 credentials with permission to post in the specified channel. Make sure the Slack app is properly authorized.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Slack node requirement                                                                                                                      |
| Google API quotas and permission scopes must allow read access to Drive folders and write access to the Google Sheet. Credentials should be refreshed before expiration to avoid workflow failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Google Drive and Sheets node notes                                                                                                         |
| PDF extraction may fail on scanned images or password-protected documents. Ensure resumes are in readable text-based PDF format for best results.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Extract profile information node limitations                                                                                              |
| GPT-4 API usage can incur costs and rate limits; monitor usage and consider batching or caching results if needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | GPT-4 model node considerations                                                                                                           |

---

**Disclaimer:**  
The provided text is exclusively derived from an automation workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.