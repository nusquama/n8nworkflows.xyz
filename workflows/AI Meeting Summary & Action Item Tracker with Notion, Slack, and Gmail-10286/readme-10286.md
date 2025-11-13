AI Meeting Summary & Action Item Tracker with Notion, Slack, and Gmail

https://n8nworkflows.xyz/workflows/ai-meeting-summary---action-item-tracker-with-notion--slack--and-gmail-10286


# AI Meeting Summary & Action Item Tracker with Notion, Slack, and Gmail

### 1. Workflow Overview

This workflow automates the end-to-end process of meeting data management by capturing meeting transcripts, analyzing them with AI, and distributing actionable insights. It targets teams and organizations looking to streamline meeting follow-ups, task assignments, and documentation using integrated tools like Notion, Slack, Gmail, Google Calendar, and Google Sheets.

**Logical blocks:**

- **1.1 Input Reception**: Captures incoming meeting data via webhook and normalizes input.
- **1.2 AI Processing**: Uses OpenAI to analyze transcripts, extract summaries, decisions, and action items.
- **1.3 Data Enrichment**: Combines AI output with meeting metadata; enriches action items with priority scoring.
- **1.4 Meeting Summary Distribution**: Posts a structured meeting summary to Slack.
- **1.5 Meeting Note Creation**: Creates a summarized meeting note in Notion.
- **1.6 Action Item Processing**: Splits action items for individual handling, creates Notion tasks, sends email notifications, and sets calendar reminders.
- **1.7 Metrics Logging and Response**: Logs meeting analytics to Google Sheets and responds to the webhook with a JSON confirmation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives meeting data via a secure webhook, then parses and standardizes the data format for downstream processing.

**Nodes Involved:**  
- Receive Meeting Data  
- Parse Meeting Input  
- Sticky Note (comment on input)

**Node Details:**

- **Receive Meeting Data**  
  - Type: Webhook  
  - Role: Entry point to receive meeting transcript and metadata via HTTP POST at `/meeting-transcript`.  
  - Config: Responds with the last node output.  
  - Input: External HTTP POST requests.  
  - Output: Raw meeting data JSON.  
  - Edge Cases: Invalid HTTP method, malformed JSON, missing fields, security risks if webhook exposed.

- **Parse Meeting Input**  
  - Type: Code  
  - Role: Extracts and normalizes meeting information from diverse input formats, supporting different field names and data types.  
  - Key Logic: Maps fields like title, date, attendees, transcript, duration, URLs, and organizer with fallbacks. Splits comma-separated attendees into arrays. Calculates word count, attendee count, and recording presence flags.  
  - Input: Raw webhook JSON.  
  - Output: Structured meeting data JSON with consistent keys.  
  - Edge Cases: Missing or corrupt fields, attendees passed as string or array, transcript could be empty.

- **Sticky Note**  
  - Content: "Captures incoming meeting info via webhook."

---

#### 1.2 AI Processing

**Overview:**  
Sends the normalized meeting transcript and metadata to OpenAI’s Chat Completion API to generate structured JSON summarizing the meeting contents.

**Nodes Involved:**  
- AI Meeting Analysis  
- Sticky Note (comment on AI usage)

**Node Details:**

- **AI Meeting Analysis**  
  - Type: OpenAI (Chat Completion)  
  - Role: Uses a system prompt defining expected JSON keys and formats; sends meeting details and transcript as user message.  
  - Configuration: Temperature set to 0.3 for controlled creativity; max tokens 1500; expects JSON output with keys like executive_summary, key_decisions, action_items, etc.  
  - Input: Parsed meeting data JSON.  
  - Output: AI-generated JSON string embedded in Chat Completion response.  
  - Edge Cases: API rate limits, incomplete or malformed AI response, timeout, JSON parse errors, missing keys in AI output.

- **Sticky Note**  
  - Content: "Uses AI to analyze transcript and summarize."

---

#### 1.3 Data Enrichment

**Overview:**  
Parses AI JSON, enriches action items by assigning IDs, calculating priority scores, and adding metadata. Calculates completeness score and flags for urgent attention or unassigned tasks.

**Nodes Involved:**  
- Synthesize Intelligence  
- Sticky Note (comment on data enrichment)

**Node Details:**

- **Synthesize Intelligence**  
  - Type: Code  
  - Role: Parses AI response JSON, enriches action items with computed priority scores, assigns status and timestamps, sorts action items by priority score.  
  - Key Expressions: Priority scoring function, completeness score calculation considering presence and assignment of tasks, open questions, and next steps.  
  - Input: AI Meeting Analysis output and parsed meeting data.  
  - Output: Fully structured meeting intelligence object with metadata, enriched action items, and analysis flags.  
  - Edge Cases: Malformed AI JSON, empty action items array, missing owner or due dates.

- **Sticky Note**  
  - Content: "Combines AI insights with meeting metadata."

---

#### 1.4 Meeting Summary Distribution

**Overview:**  
Posts a formatted summary of the meeting and key insights to a Slack channel using Slack's messaging API with markdown support.

**Nodes Involved:**  
- Post Meeting Summary  
- Sticky Note (comment on Slack summary)

**Node Details:**

- **Post Meeting Summary**  
  - Type: Slack (Messaging)  
  - Role: Sends a detailed summary including meeting title, date, attendees, completeness score, executive summary, key decisions, action items (up to 5), warnings for unassigned tasks, and a recording link if available.  
  - Configuration: Uses OAuth2 credentials for Slack.  
  - Input: Synthesize Intelligence output JSON.  
  - Output: Slack message confirmation.  
  - Edge Cases: Slack API rate limits, message formatting errors, Slack channel permissions.

- **Sticky Note**  
  - Content: "Sends structured summary to Slack channel."

---

#### 1.5 Meeting Note Creation

**Overview:**  
Creates a meeting note page in Notion, including key decisions and discussion topics as headings and bulleted lists.

**Nodes Involved:**  
- Create Meeting Note  
- Sticky Note (comment on Notion note creation)

**Node Details:**

- **Create Meeting Note**  
  - Type: Notion  
  - Role: Creates a new Notion page with the meeting title and structured blocks for key decisions and discussion topics.  
  - Configuration: Page ID is configured via URL (empty in snippet, requires user input).  
  - Input: Synthesize Intelligence output JSON.  
  - Output: Notion page creation response.  
  - Edge Cases: Notion API rate limits, invalid page ID, permission errors.

- **Sticky Note**  
  - Content: "Adds meeting tasks to Google Calendar." (Note: This sticky note is near calendar node but the one near meeting note node says "Adds meeting tasks to Google Calendar." — The correct sticky note near "Create Meeting Note" says: "Adds action items as Notion tasks." Actually, the sticky note near "Create Meeting Note" says: "Adds meeting tasks to Google Calendar." This is likely referencing Calendar node; the note near Create Meeting Note says: "Adds action items as Notion tasks." We clarify accordingly.)

- **Sticky Note near Create Meeting Note**: "Adds meeting tasks to Google Calendar." (This appears misplaced; correct sticky note for Create Meeting Note is "Adds meeting tasks to Notion." The actual sticky note near Create Task in Notion says "Adds action items as Notion tasks.")

---

#### 1.6 Action Item Processing

**Overview:**  
Separates each action item for individual processing, creates corresponding Notion tasks, sends email notifications to owners, and creates Google Calendar reminders.

**Nodes Involved:**  
- Split Action Items  
- Create Task in Notion  
- Email Task Owner  
- Create Calendar Reminder  
- Sticky Notes (comments on splitting, task creation, emails, calendar)

**Node Details:**

- **Split Action Items**  
  - Type: Code  
  - Role: Takes the array of action items and outputs each as an individual item with meeting context for parallel processing.  
  - Input: Synthesize Intelligence output JSON.  
  - Output: Array of action items individually wrapped.  
  - Edge Cases: Empty action item list.

- **Create Task in Notion**  
  - Type: Notion  
  - Role: Creates a Notion page for each action item with the task title (task description).  
  - Configuration: Page ID must be specified by the user.  
  - Input: Individual action item JSON.  
  - Output: Notion page creation response.  
  - Edge Cases: API limits, invalid page ID, permission issues.

- **Email Task Owner**  
  - Type: Gmail  
  - Role: Sends an email to the assigned task owner (email address constructed as `{owner}@company.com`) with task details, priority color-coded, due date, and meeting link.  
  - Configuration: Uses Gmail credentials; email subject and body use HTML with dynamic content.  
  - Input: Output from Create Task in Notion (individual action item).  
  - Output: Email sent confirmation.  
  - Edge Cases: Invalid owner email, sending limits, SMTP failures.

- **Create Calendar Reminder**  
  - Type: Google Calendar  
  - Role: Adds a calendar event for the task owner on the due date (defaulting to 7 days later if missing), with the owner as attendee.  
  - Configuration: Calendar ID must be selected; start and end times based on due_date or default.  
  - Input: Individual action item JSON.  
  - Output: Calendar event creation response.  
  - Edge Cases: Missing due date, invalid calendar, permission errors.

- **Sticky Notes:**  
  - Split Action Items: "Separates each action item for processing." (Applied to both Split Action Items and Create Task in Notion).  
  - Create Task in Notion: "Adds action items as Notion tasks."  
  - Email Task Owner: "Sends email with assigned task details."  
  - Create Calendar Reminder: "Adds meeting tasks to Google Calendar."

---

#### 1.7 Metrics Logging and Response

**Overview:**  
Logs key meeting analytics in Google Sheets for tracking and insights, then returns a JSON confirmation response to the original webhook sender.

**Nodes Involved:**  
- Log Meeting Metrics  
- Return Success Response  
- Sticky Notes (comments on logging and response)

**Node Details:**

- **Log Meeting Metrics**  
  - Type: Google Sheets  
  - Role: Appends a new row with meeting date, duration, attendee count, number of decisions, sentiment, meeting ID, action items count, completeness score, high priority count, meeting title, follow-up flag, and unassigned task presence.  
  - Configuration: Document ID and sheet ID are user-configured; mapping mode explicitly defines columns.  
  - Input: Output from Post Meeting Summary, Create Calendar Reminder, and Email Task Owner (all chained to logging).  
  - Output: Sheet append confirmation.  
  - Edge Cases: Sheet access permissions, API limits, incorrect sheet ID.

- **Return Success Response**  
  - Type: Respond to Webhook  
  - Role: Sends a JSON response confirming success and summarizing meeting ID, number of action items created, completeness score, and executive summary.  
  - Input: Log Meeting Metrics output.  
  - Output: HTTP 200 JSON response to webhook caller.  
  - Edge Cases: Webhook response timeout, data access errors.

- **Sticky Notes:**  
  - Log Meeting Metrics: "Records analytics in Google Sheets."  
  - Return Success Response: "Sends JSON confirmation to webhook."

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                        | Input Node(s)            | Output Node(s)               | Sticky Note                                                      |
|-----------------------|---------------------|-------------------------------------|--------------------------|-----------------------------|-----------------------------------------------------------------|
| Receive Meeting Data   | Webhook             | Receive meeting data via HTTP POST  | —                        | Parse Meeting Input          | Captures incoming meeting info via webhook.                     |
| Parse Meeting Input    | Code                | Normalize and structure input data  | Receive Meeting Data      | AI Meeting Analysis          | Extracts and structures raw meeting details.                    |
| AI Meeting Analysis    | OpenAI Chat Completion | Analyze transcript with AI            | Parse Meeting Input       | Synthesize Intelligence      | Uses AI to analyze transcript and summarize.                    |
| Synthesize Intelligence| Code                | Enrich AI output, compute scores    | AI Meeting Analysis       | Post Meeting Summary, Create Meeting Note, Split Action Items | Combines AI insights with meeting metadata.                     |
| Post Meeting Summary   | Slack               | Post meeting summary to Slack       | Synthesize Intelligence  | Log Meeting Metrics          | Sends structured summary to Slack channel.                      |
| Create Meeting Note    | Notion              | Create meeting note page in Notion  | Synthesize Intelligence  | Create Calendar Reminder     | Adds meeting tasks to Google Calendar. (Note: sticky note placement ambiguous) |
| Split Action Items     | Code                | Separate action items for processing| Synthesize Intelligence  | Create Task in Notion        | Separates each action item for processing.                      |
| Create Task in Notion  | Notion              | Create Notion task per action item  | Split Action Items        | Email Task Owner             | Adds action items as Notion tasks.                              |
| Email Task Owner       | Gmail               | Notify task owner via email          | Create Task in Notion     | Log Meeting Metrics          | Sends email with assigned task details.                         |
| Create Calendar Reminder| Google Calendar    | Add calendar event for action items | Create Meeting Note       | Log Meeting Metrics          | Adds meeting tasks to Google Calendar.                          |
| Log Meeting Metrics    | Google Sheets        | Append meeting metrics data          | Post Meeting Summary, Email Task Owner, Create Calendar Reminder | Return Success Response | Records analytics in Google Sheets.                             |
| Return Success Response| Respond to Webhook   | Confirm webhook processing success   | Log Meeting Metrics       | —                           | Sends JSON confirmation to webhook.                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Receive Meeting Data**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `meeting-transcript`  
   - Response Mode: Last Node  
   - No authentication configured here (add if needed).

2. **Add Code Node: Parse Meeting Input**  
   - Connect from: Receive Meeting Data  
   - Purpose: Extract and normalize fields such as title, date, attendees, transcript, duration, URLs, organizer.  
   - Implement JavaScript code to support multiple field name variants and parse attendees string into array.  
   - Calculate word count, attendee count, and recording presence flags.

3. **Add OpenAI Node: AI Meeting Analysis**  
   - Connect from: Parse Meeting Input  
   - Resource: Chat Completion  
   - Model: GPT-4 or suitable  
   - Temperature: 0.3  
   - Max Tokens: 1500  
   - Prompt: System message defines exact JSON response structure with keys like executive_summary, key_decisions, action_items, etc.  
   - User message: Pass meeting metadata and transcript formatted from Parse Meeting Input output.

4. **Add Code Node: Synthesize Intelligence**  
   - Connect from: AI Meeting Analysis  
   - Purpose:  
     - Parse AI JSON from response.  
     - Define a priority scoring function based on priority level, owner presence, due date presence.  
     - Enrich action items with unique IDs, status, timestamps, and meeting context.  
     - Calculate completeness score considering action item completeness and analysis flags.  
     - Output consolidated meeting data with enriched action items.

5. **Add Slack Node: Post Meeting Summary**  
   - Connect from: Synthesize Intelligence  
   - Authentication: OAuth2 credentials for Slack workspace.  
   - Channel: Select appropriate channel.  
   - Message: Use markdown to format meeting summary, key decisions, action items (limit 5), warnings, and recording link.

6. **Add Notion Node: Create Meeting Note**  
   - Connect from: Synthesize Intelligence  
   - Authentication: Notion API credentials with access to relevant workspace/page.  
   - Page ID: Set target parent page for meeting notes.  
   - Create blocks: Heading for Key Decisions and Discussion Topics, with bulleted lists.

7. **Add Code Node: Split Action Items**  
   - Connect from: Synthesize Intelligence  
   - Purpose: Split array of action items into individual items for parallel processing.  
   - Output each action item with meeting metadata.

8. **Add Notion Node: Create Task in Notion**  
   - Connect from: Split Action Items  
   - Authentication: Same as meeting note node or separate credentials.  
   - Page ID: Target page for action items/tasks.  
   - Title: Use action item task description.

9. **Add Gmail Node: Email Task Owner**  
   - Connect from: Create Task in Notion  
   - Authentication: Gmail OAuth2 credentials with send permission.  
   - Recipient: Construct email as `"{{ $json.owner }}@company.com"`.  
   - Subject: "Action Item Assigned: {{ $json.task }}"  
   - HTML Body: Include meeting title, date, task details, priority color-coded, due date, meeting link, and footer about automation.

10. **Add Google Calendar Node: Create Calendar Reminder**  
    - Connect from: Create Meeting Note (or alternatively from Split Action Items if preferred)  
    - Authentication: Google Calendar OAuth2 credentials.  
    - Calendar ID: Select user/calendar for reminders.  
    - Event Start/End: Use due_date or default to 7 days later at 9-10 AM.  
    - Attendees: Include owner email.

11. **Add Google Sheets Node: Log Meeting Metrics**  
    - Connect from: Post Meeting Summary, Email Task Owner, and Create Calendar Reminder (merge outputs)  
    - Authentication: Google Sheets API credentials.  
    - Document ID: Set spreadsheet ID.  
    - Sheet Name/ID: Select target sheet.  
    - Operation: Append  
    - Define columns: Date, Duration, Attendees, Decisions, Sentiment, Meeting_ID, Action_Items, Completeness, High_Priority, Meeting_Title, Follow_Up_Needed, Unassigned_Tasks.

12. **Add Respond to Webhook Node: Return Success Response**  
    - Connect from: Log Meeting Metrics  
    - Response Type: JSON  
    - Response Body: JSON including success flag, meeting_id, action_items_created, completeness_score, executive_summary.

13. **Add Sticky Notes** at relevant points for documentation and clarity, matching the content from the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| This workflow automates meeting management end-to-end, featuring AI-driven analysis and multi-platform integration.                                    | Workflow summary sticky note.                                            |
| Connect with creator Daniel Shashko on LinkedIn for more automation insights: https://www.linkedin.com/in/daniel-shashko                              | Workflow summary sticky note.                                            |
| Ensure API credentials for Slack, Gmail, Google Calendar, Google Sheets, and Notion are correctly set with required scopes before deployment.          | Credential setup best practice.                                          |
| OpenAI prompt requires strict JSON output format; malformed AI responses can break downstream nodes. Handle errors by validating AI response JSON.      | AI processing considerations.                                            |
| Email owner addresses assume a fixed domain (@company.com); modify if different email domains are used in your organization.                            | Email Task Owner node configuration note.                               |
| Google Calendar events default to 9–10 AM on due date or one week later if due date missing; adjust as needed for business hours/timezones.             | Calendar reminder configuration note.                                   |
| Slack message uses markdown formatting; ensure Slack channel permissions allow bot posting.                                                             | Slack node usage note.                                                   |
| Google Sheets append operation requires correct sheet ID and document privileges; verify before running workflow.                                       | Metrics logging note.                                                    |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.