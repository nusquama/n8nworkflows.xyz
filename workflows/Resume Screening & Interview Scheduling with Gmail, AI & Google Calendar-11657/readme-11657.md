Resume Screening & Interview Scheduling with Gmail, AI & Google Calendar

https://n8nworkflows.xyz/workflows/resume-screening---interview-scheduling-with-gmail--ai---google-calendar-11657


# Resume Screening & Interview Scheduling with Gmail, AI & Google Calendar

### 1. Workflow Overview

This workflow automates the hiring process by screening incoming job applications via Gmail, analyzing resumes using AI, storing candidate data in Airtable, scheduling interviews on Google Calendar, and sending interview confirmation emails. It targets HR teams or recruiters who want to automate candidate intake, evaluation, and interview scheduling while ensuring accurate AI-driven candidate-role matching.

Logical blocks:

- **1.1 Input Reception and Job Application Detection:** Monitors Gmail inbox for new emails, classifies if an email is a job application via AI, then retrieves the full message with attachments.

- **1.2 Resume Processing and AI Evaluation:** Uploads resume attachments to Google Drive, extracts resume text from PDFs, analyzes resumes against predefined roles with AI, and stores candidate evaluation in Airtable.

- **1.3 Interview Scheduling Decision and Setup:** Evaluates candidate fit score, filters qualified candidates (score ≥ 8), calculates the next business day, and finds the earliest available 1-hour interview slot using Google Calendar and AI Agent.

- **1.4 Interview Event Creation and Candidate Notification:** Creates Google Calendar event for the interview, invites candidate and interviewer, and sends a personalized interview confirmation email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Job Application Detection

**Overview:**  
This block continuously monitors Gmail for incoming emails, uses an AI model to classify if the email is a job application, and retrieves detailed message data including attachments for valid applications.

**Nodes Involved:**  
- Gmail Trigger1  
- Message a model2 (OpenAI classification)  
- If2  
- Get a message1

**Node Details:**

- **Gmail Trigger1**  
  - Type: Gmail Trigger  
  - Role: Watches Gmail inbox every minute for new emails, downloads attachments automatically for subsequent nodes.  
  - Parameters: Poll interval set to every minute, attachments downloaded and stored in property prefix `data`.  
  - Inputs: None (trigger node)  
  - Outputs: Emits new emails with metadata and attachments.  
  - Failure Cases: Gmail API auth errors, rate limits, network issues.

- **Message a model2**  
  - Type: OpenAI (LangChain) Model  
  - Role: Classifies incoming emails as job applications or not using GPT-4.1-mini with a detailed prompt analyzing subject and body, returning "YES" or "NO".  
  - Parameters: Model set to GPT-4.1-mini; prompt instructs to classify emails strictly.  
  - Inputs: Output from Gmail Trigger1 email item.  
  - Outputs: Single text response "YES" or "NO" in message.content.  
  - Failure Cases: API key invalid, prompt errors, model unavailability, network timeout.

- **If2**  
  - Type: If Condition  
  - Role: Proceeds only if AI classification is not "NO" (i.e., email is a job application).  
  - Parameters: Condition checks if `message.content` ≠ "NO" exactly.  
  - Inputs: Output from Message a model2 node.  
  - Outputs: Continues workflow for "YES" responses only.  
  - Failure Cases: Expression evaluation errors if `message.content` missing.

- **Get a message1**  
  - Type: Gmail  
  - Role: Fetches full email details and attachments for the identified job application email.  
  - Parameters: Uses message ID from Gmail Trigger1, downloads attachments to `data0`.  
  - Inputs: From If2 node passing filtered job applications.  
  - Outputs: Full email JSON including attachments.  
  - Failure Cases: Message ID invalid, Gmail API errors, attachment download failures.

---

#### 2.2 Resume Processing and AI Evaluation

**Overview:**  
Uploads the resume attachment to Google Drive, extracts text from PDF, then uses AI to analyze the resume against available roles. Stores the evaluation and candidate details in Airtable.

**Nodes Involved:**  
- Upload file1  
- Extract from File1  
- Available Positions1  
- Message a model3 (OpenAI resume analysis)  
- Create a record2 (Airtable)

**Node Details:**

- **Upload file1**  
  - Type: Google Drive  
  - Role: Uploads the candidate's resume attachment to a specified folder in Google Drive.  
  - Parameters: Filename formatted as "Resume - [candidate email]", folder and drive IDs selected from list.  
  - Inputs: Attachment binary data from Get a message1.  
  - Outputs: Confirmation and metadata of uploaded file.  
  - Failure Cases: Drive auth errors, quota exceeded, file write errors.

- **Extract from File1**  
  - Type: Extract from File (PDF)  
  - Role: Extracts text content from the uploaded PDF resume for AI processing.  
  - Parameters: Operation set to PDF extraction on binary property `data0`.  
  - Inputs: Binary resume file from Upload file1.  
  - Outputs: Extracted plain text resume in JSON.  
  - Failure Cases: Corrupted PDFs, unsupported formats, extraction failures.

- **Available Positions1**  
  - Type: Set  
  - Role: Defines the list of available job positions for AI matching.  
  - Parameters: Sets three string values ["Automation Engineer", "Full Stack Developer", "Javascript Developer"].  
  - Inputs: Extracted resume text output.  
  - Outputs: Adds available positions as JSON properties for downstream AI node.  
  - Failure Cases: Misconfiguration if roles are missing or malformed.

- **Message a model3**  
  - Type: OpenAI (LangChain) Model  
  - Role: Analyzes candidate resume text against available roles, outputs a JSON object with recommended role, fit score (1–10), strengths, gaps, years of experience, top skills, and reasoning.  
  - Parameters: GPT-4.1-mini model, complex prompt defining scoring criteria and JSON-only response.  
  - Inputs: Resume text and available positions from previous nodes.  
  - Outputs: Structured JSON result with candidate evaluation.  
  - Failure Cases: Model errors, prompt misinterpretation, JSON parsing issues.

- **Create a record2**  
  - Type: Airtable  
  - Role: Saves candidate evaluation data into Airtable base and table.  
  - Parameters: Maps AI output fields and candidate email/name from Get a message1 into Airtable columns, with schema including Name, Email, Role, Score, Strengths, Gaps, Experience, Skills, Reasoning.  
  - Inputs: Candidate evaluation JSON + email details.  
  - Outputs: Airtable record creation confirmation.  
  - Failure Cases: Airtable auth errors, schema mismatch, API rate limits.

---

#### 2.3 Interview Scheduling Decision and Setup

**Overview:**  
Filters candidates with fit score ≥ 8, calculates the next business day excluding weekends, and uses an AI agent to find the earliest available 1-hour interview slot on Google Calendar.

**Nodes Involved:**  
- If3  
- Create a record3 (Airtable for high-score candidates)  
- Get Next Business Day1 (Code)  
- Get Events1 (Google Calendar Tool)  
- Check Availability1 (Google Calendar Tool)  
- AI Agent1 (LangChain AI agent)  
- OpenAI Chat Model2 (Language model used by AI Agent)  
- OpenAI Chat Model3 (Language model for structured parsing)  
- Structured Output Parser1

**Node Details:**

- **If3**  
  - Type: If Condition  
  - Role: Checks if candidate fit score from AI analysis is greater or equal to 8, passes qualified candidates.  
  - Parameters: Numeric comparison on `message.content.fit_score` ≥ 8.  
  - Inputs: AI candidate evaluation from Message a model3.  
  - Outputs: Branches qualified candidates to scheduling steps.  
  - Failure Cases: Missing or malformed score data.

- **Create a record3**  
  - Type: Airtable  
  - Role: Stores qualified candidate records in a separate Airtable base/table (possibly for further processing).  
  - Parameters: Similar field mapping as Create a record2, but uses specific Airtable base and table IDs.  
  - Inputs: AI candidate evaluation JSON and email data.  
  - Outputs: Record creation confirmation.  
  - Failure Cases: Airtable API or credential issues.

- **Get Next Business Day1**  
  - Type: Code  
  - Role: Programmatically calculates the next business day date and constructs timestamps for the interview window (9:00 AM to 6:00 PM) and default 1-hour slot (9–10 AM). Skips weekends.  
  - Parameters: Custom JavaScript code computing next business day and timestamps in "YYYY-MM-DD HH:mm:ss" format.  
  - Inputs: Triggered after candidate qualification.  
  - Outputs: JSON with nextBusinessDay, windowStart, windowEnd, defaultSlotStart, defaultSlotEnd.  
  - Failure Cases: Date computation errors, timezone issues.

- **Get Events1**  
  - Type: Google Calendar Tool  
  - Role: Retrieves all calendar events for the calculated next business day between windowStart and windowEnd.  
  - Parameters: Uses calendar "doshidev58@gmail.com" with timezone Asia/Kolkata, timeMin and timeMax set from Get Next Business Day1.  
  - Inputs: Next business day window timestamps.  
  - Outputs: List of booked events.  
  - Failure Cases: Calendar permission errors, network failures.

- **Check Availability1**  
  - Type: Google Calendar Tool  
  - Role: Checks availability given events to find free slots on the calendar. Called by AI agent.  
  - Parameters: Same calendar and timezone as Get Events1 with identical time window.  
  - Inputs: Receives events from Get Events1 via AI Agent.  
  - Outputs: Availability info for AI agent processing.  
  - Failure Cases: API or quota errors.

- **AI Agent1**  
  - Type: LangChain AI Agent  
  - Role: Orchestrates the interview slot search by querying calendar events, checking availability, and selecting the earliest free 1-hour slot between 9 AM and 6 PM on the next business day. Returns slot in strict format or fallback message if none found.  
  - Parameters: Detailed instructions embedding nextBusinessDay, tools to call Get Events1 and Check Availability1, output formatting rules.  
  - Inputs: Next business day info and calendar events.  
  - Outputs: JSON with `start_time` and `end_time` or an unavailability message.  
  - Failure Cases: AI reasoning errors, tool invocation errors, response parsing issues.

- **OpenAI Chat Model2**  
  - Type: OpenAI Language Model  
  - Role: Used internally by AI Agent1 for reasoning steps.  
  - Parameters: GPT-4.1-mini model.  
  - Inputs/Outputs: Internal to AI Agent1.  
  - Failure Cases: API errors.

- **OpenAI Chat Model3**  
  - Type: OpenAI Language Model  
  - Role: Processes AI Agent1's output for structured parsing.  
  - Parameters: GPT-4.1-mini model.  
  - Failure Cases: API or parsing errors.

- **Structured Output Parser1**  
  - Type: Structured Output Parser  
  - Role: Parses AI Agent1 output strictly as JSON with `start_time` and `end_time`. Enables downstream nodes to reliably consume interview slot data.  
  - Parameters: Auto-fix enabled, JSON schema example provided.  
  - Failure Cases: Parsing failures if AI output deviates.

---

#### 2.4 Interview Event Creation and Candidate Notification

**Overview:**  
Creates a Google Calendar event for the interview, invites both candidate and interviewer, then sends a personalized Gmail confirmation email with interview details.

**Nodes Involved:**  
- Create an event1 (Google Calendar)  
- Send a message1 (Gmail)  
- Create a record2 (Airtable) [triggered after email sent, duplicates record creation for tracking]

**Node Details:**

- **Create an event1**  
  - Type: Google Calendar  
  - Role: Books the interview event on Google Calendar between the times provided by the AI Agent, invites candidate and interviewer email addresses, and sets event details and description.  
  - Parameters: Uses start/end times from AI Agent output, calendar "doshidev58@gmail.com", summary "Interview Scheduled with Dev Doshi", location fixed, attendees include candidate and interviewer emails, description includes recommended role.  
  - Inputs: Interview slot from Structured Output Parser1 and candidate info from Get a message1 and If3.  
  - Outputs: Event creation confirmation.  
  - Failure Cases: Calendar permission errors, attendee email validation, event conflicts.

- **Send a message1**  
  - Type: Gmail  
  - Role: Sends a custom HTML email to the candidate confirming interview date, time, duration, location, and role.  
  - Parameters: Uses candidate's email from Gmail, personalized message with interview details formatted with date/time parsing and role from AI evaluation. Subject includes interview date.  
  - Inputs: Event details from Create an event1 and candidate info from Get a message1.  
  - Outputs: Confirmation of email sent.  
  - Failure Cases: Gmail sending limits, invalid recipient emails.

- **Create a record2** (already described in block 2.2)  
  - Role: In this context, it appears triggered after Send a message1 to confirm candidate data stored, possibly a duplicate or backup record.  
  - Failure Cases: See above.

---

### 3. Summary Table

| Node Name              | Node Type                      | Functional Role                              | Input Node(s)           | Output Node(s)           | Sticky Note                                                  |
|------------------------|--------------------------------|----------------------------------------------|-------------------------|--------------------------|--------------------------------------------------------------|
| Gmail Trigger1         | Gmail Trigger                  | Monitors Gmail inbox for new emails          | None                    | Message a model2          | See Sticky Note6: Step 1 - Detect and collect job apps       |
| Message a model2       | OpenAI (LangChain) Model       | Classifies emails as job applications        | Gmail Trigger1          | If2                      | See Sticky Note6                                              |
| If2                    | If Condition                   | Filters emails classified as job applications| Message a model2        | Get a message1           | See Sticky Note6                                              |
| Get a message1         | Gmail                          | Retrieves full email and attachments         | If2                     | Upload file1, Extract from File1 | See Sticky Note6                                          |
| Upload file1           | Google Drive                   | Uploads resume attachment                     | Get a message1          | Extract from File1        | See Sticky Note6                                              |
| Extract from File1     | Extract from File (PDF)        | Extracts text from PDF resume                  | Upload file1            | Available Positions1      | See Sticky Note7: Step 2 - Analyze resume and evaluate fit   |
| Available Positions1   | Set                            | Defines available job roles                    | Extract from File1      | Message a model3          | See Sticky Note7                                              |
| Message a model3       | OpenAI (LangChain) Model       | Analyzes resume vs available positions        | Available Positions1    | If3                      | See Sticky Note7                                              |
| If3                    | If Condition                   | Filters candidates with fit score ≥ 8         | Message a model3        | Get Next Business Day1, Create a record3 | See Sticky Note8: Step 3 - Check availability           |
| Create a record3       | Airtable                       | Stores qualified candidate details            | If3                     | None                     | See Sticky Note8                                              |
| Get Next Business Day1 | Code                           | Calculates next business day and time window  | If3                     | Get Events1, AI Agent1    | See Sticky Note8                                              |
| Get Events1            | Google Calendar Tool           | Fetches events on next business day            | Get Next Business Day1  | AI Agent1                | See Sticky Note8                                              |
| Check Availability1    | Google Calendar Tool           | Checks free slots in calendar                   | Get Next Business Day1  | AI Agent1                | See Sticky Note8                                              |
| AI Agent1              | LangChain AI Agent             | Finds earliest free 1-hour slot on next business day | Get Events1, Check Availability1, Get Next Business Day1 | Create an event1       | See Sticky Note8                                              |
| OpenAI Chat Model2     | OpenAI Language Model          | Supports AI Agent1 reasoning                    | AI Agent1 (internal)    | AI Agent1                | Internal to AI Agent1                                         |
| OpenAI Chat Model3     | OpenAI Language Model          | Parses AI Agent1 output                          | AI Agent1               | Structured Output Parser1 | Internal to AI Agent1                                         |
| Structured Output Parser1 | Structured Output Parser    | Parses AI Agent JSON output for slot times     | OpenAI Chat Model3      | Create an event1         | See Sticky Note8                                              |
| Create an event1       | Google Calendar                | Books interview event and invites attendees    | AI Agent1, Structured Output Parser1, Get a message1, If3 | Send a message1          | See Sticky Note9: Step 4 - Schedule interview and notify    |
| Send a message1        | Gmail                         | Sends interview confirmation email             | Create an event1        | Create a record2         | See Sticky Note9                                              |
| Create a record2       | Airtable                      | Stores candidate data after email sent          | Send a message1         | None                     | See Sticky Note9                                              |
| Sticky Note5           | Sticky Note                   | Workflow overview and setup instructions        | None                    | None                     | Workflow overview and setup instructions                      |
| Sticky Note6           | Sticky Note                   | Step 1: Detect and collect job-application data | None                    | None                     | Step 1 description                                           |
| Sticky Note7           | Sticky Note                   | Step 2: Analyze resume and evaluate candidate fit | None                    | None                     | Step 2 description                                           |
| Sticky Note8           | Sticky Note                   | Step 3: Check availability on next business day | None                    | None                     | Step 3 description                                           |
| Sticky Note9           | Sticky Note                   | Step 4: Schedule interview and notify candidate | None                    | None                     | Step 4 description                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: Gmail Trigger  
   - Poll every minute  
   - Enable attachment download with prefix `data`  
   - Connect your Gmail OAuth2 credentials

2. **Add OpenAI Node "Message a model2" for Email Classification:**  
   - Type: OpenAI (LangChain)  
   - Model: GPT-4.1-mini  
   - Prompt: Classify email as job application ("YES"/"NO") based on subject and body with detailed instructions  
   - Input: Use Gmail Trigger output fields `subject` and `textAsHtml`  
   - Connect Gmail Trigger output to this node

3. **Add If Node "If2":**  
   - Condition: `message.content` not equal to "NO"  
   - Connect Message a model2 output to If2

4. **Add Gmail Node "Get a message1":**  
   - Operation: Get message by ID  
   - Message ID: Use `id` from Gmail Trigger1  
   - Attachments downloaded to binary property `data0`  
   - Connect If2 "true" output to this node  
   - Use Gmail OAuth2 credentials for access

5. **Add Google Drive Node "Upload file1":**  
   - Upload attachment binary from `data0`  
   - File name: "Resume - [candidate email]" dynamically set  
   - Select Drive and Folder IDs for storage  
   - Connect Get a message1 output to this node  
   - Use Google Drive credentials

6. **Add Extract from File Node "Extract from File1":**  
   - Operation: PDF text extraction on binary `data0`  
   - Connect Upload file1 output to this node

7. **Add Set Node "Available Positions1":**  
   - Define string fields: Position 1 = "Automation Engineer", position 2 = "Full Stack Developer", Position 3 = "Javascript Developer"  
   - Connect Extract from File1 output to this node

8. **Add OpenAI Node "Message a model3" for Resume Analysis:**  
   - Model: GPT-4.1-mini  
   - Prompt: Detailed resume analysis comparing extracted text against available roles, scoring 1-10, returning JSON with recommended role, score, strengths, gaps, experience, skills, reasoning  
   - Connect Available Positions1 output to this node

9. **Add Airtable Node "Create a record2":**  
   - Base and Table: Set your Airtable base and table for candidate data  
   - Map fields from Message a model3 JSON and Get a message1 email data (Name, Email, Score, etc.)  
   - Connect Send a message1 output to this node (or directly from Message a model3 if desired)

10. **Add If Node "If3":**  
    - Condition: Candidate fit score ≥ 8 from Message a model3 output  
    - Connect Message a model3 output to this node

11. **Add Airtable Node "Create a record3":**  
    - Base and Table: Different Airtable base/table for qualified candidates  
    - Map same fields as Create a record2  
    - Connect If3 "true" output to this node

12. **Add Code Node "Get Next Business Day1":**  
    - Paste JavaScript code to calculate next business day and timestamps for 9:00-18:00 window, skipping weekends  
    - Connect If3 "true" output to this node

13. **Add Google Calendar Node "Get Events1":**  
    - Operation: Get all events for `windowStart` to `windowEnd` from Get Next Business Day1  
    - Select calendar and timezone (e.g., Asia/Kolkata)  
    - Connect Get Next Business Day1 output to this node

14. **Add Google Calendar Node "Check Availability1":**  
    - Set calendar and timezone same as Get Events1  
    - Time window same as Get Events1  
    - Connect Get Next Business Day1 output to this node (used by AI Agent)

15. **Add LangChain AI Agent Node "AI Agent1":**  
    - Prompt instructs to find earliest 1-hour free slot on nextBusinessDay using Get Events and Check Availability tools  
    - Connect Get Events1 and Check Availability1 as AI tools inputs  
    - Connect Get Next Business Day1 output as input for nextBusinessDay variable  
    - Output expected: JSON with start_time and end_time of interview slot

16. **Add OpenAI Chat Model Nodes "OpenAI Chat Model2" and "OpenAI Chat Model3":**  
    - Both use GPT-4.1-mini  
    - Connect as language model and output parser within AI Agent1 workflow

17. **Add Structured Output Parser Node "Structured Output Parser1":**  
    - Schema: JSON with `start_time` and `end_time`  
    - Auto-fix enabled to parse AI Agent1 output  
    - Connect OpenAI Chat Model3 output to this node

18. **Add Google Calendar Node "Create an event1":**  
    - Create event on calendar with start and end times from Structured Output Parser1  
    - Event summary: "Interview Scheduled with Dev Doshi"  
    - Location: Fixed office address  
    - Attendees: Candidate email from Get a message1 and interviewer email  
    - Description includes recommended role from AI evaluation  
    - Connect Structured Output Parser1 output to this node

19. **Add Gmail Node "Send a message1":**  
    - Send email to candidate from Get a message1 email address  
    - Compose personalized HTML message with interview details (date, time, duration, location, role) dynamically filled  
    - Subject includes interview date  
    - Connect Create an event1 output to this node

20. **Finalize Connections:**  
    - Ensure nodes are connected as per logical sequence described  
    - Configure credentials for Gmail, Google Drive, Google Calendar, Airtable, and OpenAI  
    - Validate all expressions and field mappings

21. **Test Workflow:**  
    - Send test job application email with resume attachment to your Gmail  
    - Monitor workflow execution and logs for errors  
    - Adjust prompts, credentials, or node parameters as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| AI-Powered Resume Screening and Interview Scheduling Agent automates candidate intake, evaluation, and scheduling end-to-end.         | Workflow Overview Sticky Note                                                                    |
| Step 1: Detect and collect job-application data via Gmail and AI classification.                                                       | Sticky Note6                                                                                     |
| Step 2: Analyze resume content with AI against predefined roles to evaluate candidate fit and score.                                  | Sticky Note7                                                                                     |
| Step 3: Calculate next business day and use AI Agent with calendar tools to find earliest free interview slot.                        | Sticky Note8                                                                                     |
| Step 4: Schedule interview on Google Calendar and notify candidate via personalized email.                                            | Sticky Note9                                                                                     |
| Use GPT-4.1-mini models for classification, resume analysis, and AI Agent orchestration for best results.                             | Model selection across AI nodes                                                                  |
| Ensure all credentials (Gmail OAuth2, Google Drive, Google Calendar, Airtable API, OpenAI) are set up and tested before activating.  | Critical for smooth workflow operation                                                          |
| Airtable base/table schema must include fields for candidate info, scores, strengths, gaps, reasoning, and recommended role.          | Schema defined in Airtable Create record nodes                                                   |
| Google Calendar time zone set to Asia/Kolkata in example; adjust according to your local time zone.                                   | Calendar nodes configuration                                                                     |
| The AI prompts contain strict instructions for JSON-only outputs to avoid parsing errors downstream.                                  | Important for stable automation                                                                  |
| Workflow uses sticky notes for detailed explanations and setup instructions; review them in n8n editor for quick reference.           | See sticky notes positioned at workflow start and each major step                                |

---

This documentation fully describes the "Resume Screening & Interview Scheduling with Gmail, AI & Google Calendar" workflow, enabling replication, modification, and troubleshooting by advanced users or AI automation agents.