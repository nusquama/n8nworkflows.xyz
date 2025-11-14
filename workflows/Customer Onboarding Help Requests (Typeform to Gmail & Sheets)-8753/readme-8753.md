Customer Onboarding Help Requests (Typeform to Gmail & Sheets)

https://n8nworkflows.xyz/workflows/customer-onboarding-help-requests--typeform-to-gmail---sheets--8753


# Customer Onboarding Help Requests (Typeform to Gmail & Sheets)

### 1. Workflow Overview

This workflow automates the process of handling customer onboarding help requests submitted via Typeform. It captures form submissions, logs the data into Google Sheets for tracking, validates essential fields like email, generates a professional welcome email, sends it via Gmail, creates a task in ClickUp for follow-up, enriches task data, and uses AI to summarize submissions and notify a Slack channel for internal awareness.

The workflow is logically divided into the following blocks:

- **1.1 Form Capture and Data Logging:** Trigger on Typeform submission and log data into Google Sheets.
- **1.2 Email Validation and Processing:** Check if an email exists; generate and send a professional welcome email if valid; otherwise log an error.
- **1.3 Task Creation and Data Enrichment:** Create a ClickUp task from the submission, format task data, and append/update it in a secondary Google Sheet.
- **1.4 AI Summarization and Notification:** Use Azure OpenAI GPT-4o to analyze submissions, generate structured summaries, and post insights to a Slack channel.
- **1.5 Supporting Utilities and Notes:** Sticky notes explain each step and provide setup instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Form Capture and Data Logging

**Overview:**  
This block captures new onboarding help requests from a Typeform form and logs all submitted data into a Google Sheet for tracking and analytics.

**Nodes Involved:**  
- Typeform Trigger  
- Log to Google Sheets  
- Sticky Note1 (explanatory)

**Node Details:**  

- **Typeform Trigger**  
  - Type: Trigger node for Typeform submissions  
  - Configuration: Listens to submissions on form ID `cXaYXrBp` using provided Typeform API credentials  
  - Input: Webhook from Typeform on form completion  
  - Output: JSON containing all form responses keyed by question titles  
  - Edge Cases: Webhook connectivity issues, API credential invalidation, delays in form submission propagation  

- **Log to Google Sheets**  
  - Type: Google Sheets node for appending data  
  - Configuration: Appends form submission data as a new row to a Google Sheet identified by document ID `1ZOwioaHTLWKltq6UbxzZh02UEBKfypdv_GOhcPu4fHg` on sheet `"Sheet1"` (gid=0)  
  - Maps multiple form fields (e.g., Full Name, Email Address, Issue Details) to corresponding columns  
  - Input: Data from Typeform Trigger  
  - Output: Confirmation of row append operation  
  - Edge Cases: Google API quota limits, credential expiry, sheet permission errors, mismatched headers causing data mapping failures  

- **Sticky Note1**  
  - Content: Step 1 explanation about form capture and setup instructions  
  - Applies to: Typeform Trigger node  

---

#### 1.2 Email Validation and Processing

**Overview:**  
This block validates presence of email data from form submission, generates a branded professional HTML email, and sends a welcome email via Gmail. If no valid email exists, it triggers an error handling path.

**Nodes Involved:**  
- Check Email Exists (If node)  
- Generate Professional Email HTML (Code node)  
- Send Welcome Email (Gmail node)  
- Handle Missing Email and log error (Function node)  
- Sticky Note3, Sticky Note4, Sticky Note5, Sticky Note6  

**Node Details:**  

- **Check Email Exists**  
  - Type: If node (conditional branching)  
  - Configuration: Checks if either `$json.email` or `$json.Email` field is non-empty (case-sensitive, strict validation)  
  - Input: Output from Google Sheets logging node (form data)  
  - Output:  
    - True branch: Proceed to email generation, task creation, and AI processing  
    - False branch: Route to error handling node  
  - Edge Cases: Email field missing, empty, or malformed; potential false positives if email field uses unexpected key  

- **Generate Professional Email HTML**  
  - Type: Code node (JavaScript)  
  - Configuration: Generates a fully styled, branded HTML email embedding all form fields as a customer inquiry notification  
  - Key Expressions: Iterates over all key-value pairs in input JSON to build email content dynamically; includes company branding, next steps, and contact info  
  - Input: Validated form submission data with email  
  - Output: JSON with `html` (email body), `htmlContent`, `itemCount`, and `emailSubject` fields  
  - Edge Cases: Unexpected form field names, empty values replaced by empty string, HTML injection risk mitigated by template usage  

- **Send Welcome Email**  
  - Type: Gmail node  
  - Configuration: Sends email to `{{$json.email}}` with subject `"Welcome to [Your Company]! Here are your onboarding resources"` and message body from generated HTML  
  - Credentials: Gmail OAuth2 credentials configured  
  - Input: HTML email from previous node  
  - Output: Send confirmation or error  
  - Edge Cases: Gmail API quota limits, OAuth token expiry, invalid recipient email, spam filtering  

- **Handle Missing Email and log error**  
  - Type: Function node  
  - Purpose: Custom error handling for submissions missing valid email addresses  
  - Input: Data from Check Email Exists false branch  
  - Output: Could log or notify internal teams for manual follow-up  
  - Edge Cases: Missing or malformed data, manual intervention required  

- **Sticky Notes 3-6**  
  - Explain email validation logic, email template design/customization, sending process, and error handling rationale  

---

#### 1.3 Task Creation and Data Enrichment

**Overview:**  
Creates a ClickUp task for each valid submission to track onboarding actions, formats task data for clarity, and logs enriched task details into a secondary Google Sheet for reporting.

**Nodes Involved:**  
- Create ClickUp Onboarding Task (ClickUp node)  
- Format Customer Data for Sheets (Code node)  
- Append or update row in sheet (Google Sheets node)  
- Sticky Note8, Sticky Note9, Sticky Note10  

**Node Details:**  

- **Create ClickUp Onboarding Task**  
  - Type: ClickUp node  
  - Configuration: Creates a task in ClickUp list ID `901411336526` with task name from form field `"Which tutorial did you complete?"`  
  - Assigns task to user matching `"First name"` field from form data  
  - Credentials: ClickUp API credentials configured  
  - Input: Data from email validation pass branch  
  - Output: ClickUp task JSON response with full task details  
  - Edge Cases: API limits, invalid assignee, missing list/team/space IDs, network errors  

- **Format Customer Data for Sheets**  
  - Type: Code node (JavaScript)  
  - Configuration: Transforms ClickUp task JSON into a structured, human-readable format suitable for Google Sheets logging, including:  
    - Date formatting for created, due, closed, done, start dates  
    - Extraction and concatenation of assignees, watchers, tags, custom fields  
    - Priority normalization  
    - Time estimates and spent time conversion to hours  
    - Metadata like permissions, archived status, URLs  
  - Input: ClickUp task data  
  - Output: Array of formatted JSON objects for sheet insertion  
  - Edge Cases: Missing fields, empty arrays, timestamp parsing errors  

- **Append or update row in sheet**  
  - Type: Google Sheets node  
  - Configuration: Appends or updates task data rows in Google Sheet ID `1_aMeJ3avtD48dLkQxmgTtjDgbvhWp8a5v5wzgzIDd3o` in `"Sheet1"` (gid=0)  
  - Maps complex task data fields into corresponding columns  
  - Input: Formatted task data from previous node  
  - Output: Confirmation of append/update operation  
  - Edge Cases: Sheet access issues, data mapping mismatches, API quota limits  

- **Sticky Notes 8-10**  
  - Explain AI summarizer role, data formatting details, and language model setup  

---

#### 1.4 AI Summarization and Notification

**Overview:**  
Uses Azure OpenAI GPT-4o to process raw form submission data, generate a concise structured summary with insights and actionable recommendations, and posts these results to a designated Slack channel for internal team awareness.

**Nodes Involved:**  
- Azure OpenAI Chat Model1 (Azure OpenAI Language Model node)  
- Simple Memory1 (Memory buffer node)  
- AI Agent (Langchain Agent node)  
- Structured Output Parser (Output parser node)  
- Send a message (Slack node)  
- Sticky Note11, Sticky Note12, Sticky Note13  

**Node Details:**  

- **Azure OpenAI Chat Model1**  
  - Type: Langchain Azure OpenAI chat model node  
  - Configuration: Uses GPT-4o model for natural language processing  
  - Credentials: Azure OpenAI API credentials configured  
  - Input: Prompt text and session context from memory node  
  - Output: Raw AI response JSON  
  - Edge Cases: API limits, network latency, malformed prompts  

- **Simple Memory1**  
  - Type: Langchain memory buffer window  
  - Configuration: Maintains last 7 conversation turns under session key `"json_review"` for context continuity  
  - Input: Form submission data  
  - Output: Context-rich input for AI Agent  
  - Edge Cases: Memory overflow, session management errors  

- **AI Agent**  
  - Type: Langchain agent node  
  - Configuration: Receives concatenated form submission text, system message instructing JSON output format with summary, insights, and call to action  
  - Input: Text from form data, system prompt, memory from Simple Memory1, and model from Azure OpenAI node  
  - Output: AI-generated structured JSON with defined schema  
  - Edge Cases: Unexpected input format, empty or irrelevant data, output parsing errors  

- **Structured Output Parser**  
  - Type: Langchain output parser node  
  - Configuration: Validates and enforces output JSON schema (summary:string, insights:array, callToAction:string)  
  - Input: Raw AI response  
  - Output: Clean JSON for downstream use  
  - Edge Cases: Parsing failures, schema mismatches  

- **Send a message (Slack)**  
  - Type: Slack node  
  - Configuration: Posts formatted message containing summary, insights, and call to action to Slack channel ID `C09H21LK9BJ` (channel named `reply-needed`)  
  - Credentials: Slack API token configured  
  - Input: Parsed AI output JSON  
  - Output: Slack post confirmation  
  - Edge Cases: Slack API rate limits, invalid channel ID, message formatting errors  

- **Sticky Notes 11-13**  
  - Explain conversation memory, output parsing, and Slack notification setup  

---

#### 1.5 Supporting Utilities and Notes

**Nodes Involved:**  
- Sticky Note2 (Google Sheets logging explanation)  
- Sticky Note7 (Task data logging explanation)  
- Sticky Note (Create ClickUp Task explanation)  

These provide setup guidance, best practices, and contextual information for each major step.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                         | Input Node(s)                | Output Node(s)                          | Sticky Note                                                                                                 |
|--------------------------------|--------------------------------------|---------------------------------------|-----------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------|
| Typeform Trigger               | Typeform Trigger                     | Capture form submissions               | —                           | Log to Google Sheets                   | 🎯 **STEP 1: Form Capture**<br>Triggers on Typeform submission. Setup your Typeform ID and credentials.     |
| Log to Google Sheets           | Google Sheets                       | Append form data to tracking sheet    | Typeform Trigger            | Check Email Exists                     | 📊 **STEP 2: Data Logging**<br>Append form data to Google Sheets; map fields; ensure headers match.          |
| Check Email Exists             | If                                 | Validate presence of email             | Log to Google Sheets        | Generate Professional Email HTML, Create ClickUp Onboarding Task, AI Agent / Handle Missing Email and log error | ✅ **STEP 3: Email Validation**<br>Check if valid email exists before sending welcome email.                  |
| Generate Professional Email HTML | Code                              | Generate branded HTML email            | Check Email Exists (true)   | Send Welcome Email                     | 🎨 **STEP 4: Email Template**<br>Generates professional HTML with branding, details, contact info.           |
| Send Welcome Email             | Gmail                              | Send onboarding welcome email          | Generate Professional Email HTML | —                                   | 📧 **STEP 5: Send Email**<br>Send email via Gmail; update sender and test before production.                  |
| Handle Missing Email and log error | Function                       | Handle invalid/missing email submissions | Check Email Exists (false)  | —                                     | ⚠️ **Error Handler**<br>Log missing emails for manual follow-up.                                            |
| Create ClickUp Onboarding Task | ClickUp                            | Create onboarding task in ClickUp      | Check Email Exists (true)   | Format Customer Data for Sheets        | 📌 **Create ClickUp Task**<br>Auto-create task for each valid submission.                                   |
| Format Customer Data for Sheets | Code                              | Format ClickUp task data for Sheets    | Create ClickUp Onboarding Task | Append or update row in sheet          | 🛠️ **Format ClickUp Data**<br>Clean and structure task data for logging.                                   |
| Append or update row in sheet | Google Sheets                      | Append/update enriched task data       | Format Customer Data for Sheets | —                                   | 📊 **Save Task Data**<br>Maintain enriched task dataset for reporting.                                     |
| Azure OpenAI Chat Model1       | Langchain Azure OpenAI Chat Model  | Provide GPT-4o language model          | —                           | AI Agent                              | 🧠 **Language Model**<br>GPT-4o from Azure OpenAI powers AI Agent.                                         |
| Simple Memory1                | Langchain Memory Buffer Window      | Maintain conversation memory           | —                           | AI Agent                              | 🗂️ **Conversation Memory**<br>Keep last 7 turns for context.                                               |
| AI Agent                     | Langchain Agent                      | Process form text, generate JSON summary | Check Email Exists (true)   | Send a message                        | 🤖 **AI Summarizer**<br>Produce structured summary, insights, and call to action.                          |
| Structured Output Parser     | Langchain Output Parser Structured   | Validate and enforce AI output schema  | AI Agent                   | AI Agent                              | 📦 **Structured Parser**<br>Validate AI output JSON format.                                                |
| Send a message               | Slack                              | Post AI summary and insights to Slack | AI Agent                   | —                                     | 💬 **Slack Notification**<br>Post summary, insights, and actions to Slack channel `reply-needed`.          |
| Sticky Note1                 | Sticky Note                       | Step 1 explanation                     | —                           | —                                     | See above                                                                                                  |
| Sticky Note2                 | Sticky Note                       | Step 2 explanation                     | —                           | —                                     | See above                                                                                                  |
| Sticky Note3                 | Sticky Note                       | Step 3 explanation                     | —                           | —                                     | See above                                                                                                  |
| Sticky Note4                 | Sticky Note                       | Step 4 explanation                     | —                           | —                                     | See above                                                                                                  |
| Sticky Note5                 | Sticky Note                       | Step 5 explanation                     | —                           | —                                     | See above                                                                                                  |
| Sticky Note6                 | Sticky Note                       | Error handler explanation              | —                           | —                                     | See above                                                                                                  |
| Sticky Note7                 | Sticky Note                       | Task data logging explanation          | —                           | —                                     | See above                                                                                                  |
| Sticky Note8                 | Sticky Note                       | AI summarizer explanation              | —                           | —                                     | See above                                                                                                  |
| Sticky Note9                 | Sticky Note                       | ClickUp data formatting explanation    | —                           | —                                     | See above                                                                                                  |
| Sticky Note10                | Sticky Note                       | Language model explanation             | —                           | —                                     | See above                                                                                                  |
| Sticky Note11                | Sticky Note                       | Conversation memory explanation        | —                           | —                                     | See above                                                                                                  |
| Sticky Note12                | Sticky Note                       | Structured output parser explanation   | —                           | —                                     | See above                                                                                                  |
| Sticky Note13                | Sticky Note                       | Slack notification explanation         | —                           | —                                     | See above                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Trigger Node**  
   - Type: Typeform Trigger  
   - Configure with your Typeform form ID (e.g., `cXaYXrBp`)  
   - Connect appropriate Typeform API credentials  
   - Ensure webhook is active and listening  

2. **Add Google Sheets Node to Log Form Data**  
   - Type: Google Sheets (Append operation)  
   - Connect to your Google Sheets OAuth2 credentials  
   - Set Document ID to your target sheet (e.g., `1ZOwioaHTLWKltq6UbxzZh02UEBKfypdv_GOhcPu4fHg`)  
   - Sheet Name: `"Sheet1"` or appropriate tab  
   - Map fields from form response JSON keys to sheet columns (e.g., Full Name, Email Address, Issue Details, etc.)  
   - Connect input from Typeform Trigger  

3. **Add If Node to Check Email Presence**  
   - Type: If  
   - Condition: Check if `$json.email` OR `$json.Email` is not empty  
   - Connect input from Google Sheets logging node  
   - True branch for email exists, False branch otherwise  

4. **On True Branch: Generate Professional Email HTML**  
   - Type: Code node (JavaScript)  
   - Paste provided script generating branded HTML email using all form fields  
   - Connect input from If node True branch  

5. **Send Welcome Email via Gmail**  
   - Type: Gmail node  
   - Configure with Gmail OAuth2 credentials  
   - Set recipient email as `{{$json.email}}`  
   - Set subject to `"Welcome to [Your Company]! Here are your onboarding resources"`  
   - Set message to output HTML from previous node  
   - Connect input from Code node  

6. **Handle False Branch: Log Missing Email Error**  
   - Type: Function node  
   - Implement logic to log or notify about submissions missing valid emails  
   - Connect input from If node False branch  

7. **Create ClickUp Onboarding Task**  
   - Type: ClickUp node  
   - Configure with your ClickUp API credentials  
   - Set List ID, Team ID, Space ID accordingly (e.g., list: `901411336526`)  
   - Task Name: map from form field such as `"Which tutorial did you complete?"`  
   - Assign to user based on `"First name"` or other form field  
   - Connect input from If node True branch  

8. **Format ClickUp Task Data for Google Sheets**  
   - Type: Code node (JavaScript)  
   - Paste provided formatting script to transform ClickUp task JSON into sheet-friendly format  
   - Connect input from ClickUp node  

9. **Append or Update Task Data in Secondary Google Sheet**  
   - Type: Google Sheets node (Append or Update)  
   - Connect to Google Sheets credentials  
   - Set Document ID to secondary sheet (e.g., `1_aMeJ3avtD48dLkQxmgTtjDgbvhWp8a5v5wzgzIDd3o`)  
   - Map formatted task data fields to correct columns  
   - Connect input from Code node  

10. **Setup Azure OpenAI Chat Model Node**  
    - Type: Langchain Azure OpenAI Chat Model  
    - Configure with Azure OpenAI API credentials  
    - Model: `gpt-4o`  

11. **Add Simple Memory Node**  
    - Type: Langchain Memory Buffer Window  
    - Session Key: `"json_review"`  
    - Context Window Length: 7 turns  

12. **Create AI Agent Node**  
    - Type: Langchain Agent  
    - Input Text: Concatenate all relevant form fields (e.g., first name, last name, email, company, tutorial responses)  
    - System Message: Provide instructions to output clean JSON with summary, insights, call to action  
    - Connect AI Language Model and Memory nodes as inputs  

13. **Add Structured Output Parser Node**  
    - Type: Langchain Output Parser Structured  
    - Provide JSON schema example for validation (summary:string, insights:array, callToAction:string)  
    - Connect input from AI Agent node output  

14. **Add Slack Node to Send Summaries**  
    - Type: Slack node  
    - Configure with Slack API credentials  
    - Select channel (e.g., `reply-needed`)  
    - Message text: Format with AI output JSON fields: summary, insights, callToAction  
    - Connect input from Structured Output Parser node  

15. **Wire Connections Appropriately**  
    - From Typeform Trigger → Log to Google Sheets  
    - Log to Google Sheets → Check Email Exists  
    - Check Email Exists True → Generate Email HTML, Create ClickUp Task, AI Agent  
    - Generate Email HTML → Send Welcome Email  
    - Create ClickUp Task → Format Customer Data → Append or Update Sheet  
    - AI Agent → Slack message  
    - Check Email Exists False → Handle Missing Email  

16. **Add Sticky Notes at Relevant Positions**  
    - Add explanatory sticky notes to clarify each step, including setup instructions and best practices  

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 🎯 Step 1 captures submissions automatically via webhook integration with Typeform.                                     | Sticky Note1                                                                                        |
| 📊 Step 2 logs detailed form data for analytics and tracking in Google Sheets.                                          | Sticky Note2                                                                                        |
| ✅ Step 3 validates presence of email before proceeding to further processing.                                          | Sticky Note3                                                                                        |
| 🎨 Step 4 generates a clean, branded HTML email with customer details and company info.                                 | Sticky Note4                                                                                        |
| 📧 Step 5 sends the welcome email via Gmail with OAuth2 authentication.                                                 | Sticky Note5                                                                                        |
| ⚠️ Missing emails are handled by logging for manual review to ensure no inquiries are lost.                            | Sticky Note6                                                                                        |
| 📌 ClickUp tasks are created automatically to assign onboarding requests for action tracking.                          | Sticky Note                                                                                         |
| 🛠️ Task data is cleaned and formatted to standardize logging in a secondary Google Sheet.                              | Sticky Note9                                                                                        |
| 🧠 Azure OpenAI GPT-4o model is used to provide advanced natural language processing for summarization.                | Sticky Note10                                                                                       |
| 🗂️ Conversation memory with last 7 turns provides context to the AI agent for better understanding.                    | Sticky Note11                                                                                       |
| 📦 Output parser ensures AI output adheres to expected JSON schema for downstream use.                                  | Sticky Note12                                                                                       |
| 💬 Slack notifications post structured insights to a dedicated internal channel to facilitate rapid action.            | Sticky Note13                                                                                       |
| For Gmail, ensure sender email is correctly configured and test with internal accounts to avoid delivery issues.       | Sticky Note5                                                                                        |
| For Google Sheets, ensure column headers exactly match mapped form fields to prevent data misalignment.                 | Sticky Note2                                                                                        |
| For ClickUp, validate list, team, and space IDs and assignee user IDs prior to live deployment.                        | Sticky Note                                                                                         |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.