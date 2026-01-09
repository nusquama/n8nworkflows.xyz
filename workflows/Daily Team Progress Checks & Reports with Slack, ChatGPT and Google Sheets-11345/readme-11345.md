Daily Team Progress Checks & Reports with Slack, ChatGPT and Google Sheets

https://n8nworkflows.xyz/workflows/daily-team-progress-checks---reports-with-slack--chatgpt-and-google-sheets-11345


# Daily Team Progress Checks & Reports with Slack, ChatGPT and Google Sheets

### 1. Workflow Overview

This workflow automates the daily progress checking and reporting process for a team using Slack, ChatGPT (OpenAI via LangChain), and Google Sheets. It is designed to run daily at a fixed time, retrieve the current project task list from Google Sheets, filter for tasks not yet completed, send personalized Slack messages to team members requesting updates, wait for their replies, use AI to compile a summarized report of the responses, and then send this summary report to a client or stakeholder via Slack.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Data Retrieval:** Starts the workflow daily and fetches the project task breakdown structure (WBS) from Google Sheets.
- **1.2 Task Filtering & Member Information:** Filters uncompleted tasks and gathers group member details.
- **1.3 Task Status Reminder Generation:** Uses AI (ChatGPT) to update task status reminders tailored to members.
- **1.4 Slack Messaging & Waiting for Replies:** Sends messages to members on Slack and waits 30 minutes for their responses.
- **1.5 AI Report Compilation & Client Reporting:** Uses AI to compile a report summary of member replies, generates the final report, and sends it to the client on Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Retrieval

- **Overview:**  
  This block triggers the workflow daily at 17:00 and retrieves the current project task data from Google Sheets to begin processing.

- **Nodes Involved:**  
  - Start Daily at 17:00  
  - Get Project WBS  
  - Filter Uncompleted Tasks

- **Node Details:**  

  - **Start Daily at 17:00**  
    - Type: Cron Trigger  
    - Configuration: Set to execute once every day at 17:00 (5 PM).  
    - Input: None (trigger node)  
    - Output: Triggers next node "Get Project WBS"  
    - Edge cases: Cron misconfiguration or timezone mismatch could cause incorrect firing times.

  - **Get Project WBS**  
    - Type: Google Sheets Node  
    - Configuration: Retrieves project WBS (Work Breakdown Structure) data from a specified Google Sheets document and sheet.  
    - Key parameters include spreadsheet ID, sheet name, and range (not explicitly shown, but expected).  
    - Input: Triggered by Cron node.  
    - Output: Data passed to "Filter Uncompleted Tasks"  
    - Edge cases: Authentication issues with Google Sheets API, empty or malformed sheet data.

  - **Filter Uncompleted Tasks**  
    - Type: If Node  
    - Configuration: Filters tasks based on completion status, passing forward only those that are incomplete.  
    - Input: Receives full task list from Google Sheets.  
    - Output: Passes filtered tasks to "Group Member Info" for further processing.  
    - Edge cases: Incorrect or missing status fields could result in filtering errors or empty output.

---

#### 2.2 Task Filtering & Member Information

- **Overview:**  
  This block enriches the filtered task data by associating tasks with group members and prepares data for AI processing.

- **Nodes Involved:**  
  - Group Member Info  
  - Agent: Update task status reminders

- **Node Details:**  

  - **Group Member Info**  
    - Type: Function Node  
    - Configuration: Executes JavaScript code to process and organize group member information related to filtered tasks. Likely maps tasks to team members and possibly formats data for AI input.  
    - Input: Receives filtered uncompleted tasks.  
    - Output: Passes structured data to AI node "Agent: Update task status reminders".  
    - Edge cases: Logic or script errors, missing member data, or inconsistent input structure.

  - **Agent: Update task status reminders**  
    - Type: OpenAI Node (via LangChain)  
    - Configuration: Utilizes ChatGPT to generate personalized task status reminders for each team member based on task data.  
    - Input: Structured task and member info from Function node.  
    - Output: Sends generated messages to "Send Messages to Members" Slack node.  
    - Edge cases: API limits, malformed prompts, or unexpected AI responses.

---

#### 2.3 Slack Messaging & Waiting for Replies

- **Overview:**  
  This block sends the AI-generated task reminders to each team member via Slack and waits for a fixed period to collect their replies.

- **Nodes Involved:**  
  - Send Messages to Members  
  - Wait 30 Minutes for Member Replies

- **Node Details:**  

  - **Send Messages to Members**  
    - Type: Slack Node  
    - Configuration: Sends direct or channel messages to team members. Message content is derived from AI-generated reminders.  
    - Input: Receives task-specific reminder messages from AI node.  
    - Output: Triggers "Wait 30 Minutes for Member Replies" node.  
    - Credentials: Requires valid Slack OAuth2 credentials with permissions to post messages.  
    - Edge cases: Slack API rate limits, invalid recipient IDs, or connectivity issues.

  - **Wait 30 Minutes for Member Replies**  
    - Type: Wait Node (Webhook)  
    - Configuration: Pauses workflow for 30 minutes to allow members time to reply to Slack messages. Uses a webhook to resume after waiting.  
    - Input: Triggered after messages sent.  
    - Output: Triggers AI report compilation node after wait period.  
    - Edge cases: Workflow timeout limits, interruption during wait, or missed webhook calls.

---

#### 2.4 AI Report Compilation & Client Reporting

- **Overview:**  
  This block uses AI to compile a summary report of the collected replies, generates a formatted report, and sends it to the client or stakeholder via Slack.

- **Nodes Involved:**  
  - Agent: Compile Report Summary  
  - Generate Daily Report  
  - Send Report to Client

- **Node Details:**  

  - **Agent: Compile Report Summary**  
    - Type: OpenAI Node (via LangChain)  
    - Configuration: Processes the collected member replies to produce a concise summary report.  
    - Input: Data collected after wait period (member responses).  
    - Output: Passes summarized data to "Generate Daily Report".  
    - Edge cases: AI API errors, incomplete replies, or ambiguous input data.

  - **Generate Daily Report**  
    - Type: Function Node  
    - Configuration: Formats the AI summary into a final report, potentially adding timestamps, formatting, or additional data.  
    - Input: Receives AI-generated summary.  
    - Output: Sends formatted report to "Send Report to Client".  
    - Edge cases: Script logic errors, unexpected data structure.

  - **Send Report to Client**  
    - Type: Slack Node  
    - Configuration: Sends the final report message to a designated Slack channel or user representing the client or stakeholder.  
    - Input: Formatted report from Function node.  
    - Output: End of workflow branch.  
    - Credentials: Requires Slack OAuth2 credentials.  
    - Edge cases: Slack API issues, wrong channel/user ID.

---

### 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                           | Input Node(s)                 | Output Node(s)                    | Sticky Note                      |
|----------------------------------|----------------------------|-----------------------------------------|------------------------------|---------------------------------|---------------------------------|
| Start Daily at 17:00              | Cron Trigger               | Triggers workflow daily at 17:00        | -                            | Get Project WBS                 |                                 |
| Get Project WBS                  | Google Sheets              | Retrieves project task data              | Start Daily at 17:00          | Filter Uncompleted Tasks        |                                 |
| Filter Uncompleted Tasks         | If                         | Filters only uncompleted tasks           | Get Project WBS               | Group Member Info               |                                 |
| Group Member Info                | Function                   | Processes task-member associations       | Filter Uncompleted Tasks      | Agent: Update task status reminders |                                 |
| Agent: Update task status reminders | OpenAI (LangChain)         | Generates personalized task reminders    | Group Member Info             | Send Messages to Members        |                                 |
| Send Messages to Members         | Slack                      | Sends reminders to team members on Slack| Agent: Update task status reminders | Wait 30 Minutes for Member Replies |                                 |
| Wait 30 Minutes for Member Replies | Wait                      | Pauses to allow replies from members     | Send Messages to Members      | Agent: Compile Report Summary   |                                 |
| Agent: Compile Report Summary    | OpenAI (LangChain)         | Summarizes member replies into report    | Wait 30 Minutes for Member Replies | Generate Daily Report           |                                 |
| Generate Daily Report            | Function                   | Formats the final daily report            | Agent: Compile Report Summary | Send Report to Client           |                                 |
| Send Report to Client            | Slack                      | Sends report to client/stakeholder       | Generate Daily Report         | -                              |                                 |
| Sticky Note                     | Sticky Note                | (empty content)                          | -                            | -                              |                                 |
| Sticky Note2                    | Sticky Note                | (empty content)                          | -                            | -                              |                                 |
| Sticky Note4                    | Sticky Note                | (empty content)                          | -                            | -                              |                                 |
| Sticky Note6                    | Sticky Note                | (empty content)                          | -                            | -                              |                                 |
| Sticky Note7                    | Sticky Note                | (empty content)                          | -                            | -                              |                                 |
| Sticky Note8                    | Sticky Note                | (empty content)                          | -                            | -                              |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron node named "Start Daily at 17:00"**  
   - Set to trigger daily at 17:00 local time.  
   - No inputs required.

2. **Add a Google Sheets node named "Get Project WBS"**  
   - Connect output of "Start Daily at 17:00" to this node.  
   - Configure credentials for Google Sheets OAuth2.  
   - Set spreadsheet ID and sheet name/range to retrieve the list of project tasks (WBS).  
   - Ensure the data includes task status fields.

3. **Add an If node named "Filter Uncompleted Tasks"**  
   - Connect output of "Get Project WBS" to this node.  
   - Configure condition to filter tasks where the status does not equal "Completed" (or equivalent).  
   - Pass uncompleted tasks to the next node.

4. **Add a Function node named "Group Member Info"**  
   - Connect the 'true' output of the If node to this node.  
   - Write JavaScript code to associate tasks with team members, format data for AI use.  
   - Prepare output with member-task mappings.

5. **Add an OpenAI node named "Agent: Update task status reminders"**  
   - Connect output of "Group Member Info" to this node.  
   - Configure with OpenAI credentials (API key).  
   - Set model parameters (e.g., GPT-4 or GPT-3.5) with prompt designed to generate personalized Slack reminder messages based on task data.  
   - Output formatted messages for Slack.

6. **Add a Slack node named "Send Messages to Members"**  
   - Connect output of "Agent: Update task status reminders" to this node.  
   - Configure Slack credentials (OAuth2) with chat:write scope.  
   - Map AI-generated messages to Slack message text, and set recipient user IDs.  
   - Send messages to each team member.

7. **Add a Wait node named "Wait 30 Minutes for Member Replies"**  
   - Connect output of "Send Messages to Members" to this node.  
   - Configure to wait for 30 minutes before proceeding.  
   - Use webhook mode if asynchronous continuation is needed.

8. **Add an OpenAI node named "Agent: Compile Report Summary"**  
   - Connect output of the Wait node to this node.  
   - Configure with OpenAI credentials.  
   - Set prompt to process collected replies and summarize key updates and progress.  
   - Output summary text.

9. **Add a Function node named "Generate Daily Report"**  
   - Connect output of "Agent: Compile Report Summary" to this node.  
   - Write JavaScript to format the summary into a final report, add any metadata (date, project name).  
   - Prepare the message for Slack delivery.

10. **Add a Slack node named "Send Report to Client"**  
    - Connect output of "Generate Daily Report" to this node.  
    - Configure Slack credentials with permissions to post in client or stakeholder channel.  
    - Map the formatted report text to the message body.  
    - Send report.

**Note:**  
- Ensure all API credentials are securely configured in n8n credential manager.  
- Validate Slack channel/user IDs and Google Sheets access permissions before running.  
- Test each block independently to ensure data flows as expected.  
- Adjust AI prompts for desired tone and detail level in reminders and summary.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                      |
|----------------------------------------------------------------------------------------------------|------------------------------------|
| This workflow demonstrates integration between Slack, Google Sheets, and OpenAI for team reporting.| Workflow concept overview           |
| For Slack API setup, ensure OAuth2 credentials include chat:write and users:read scopes.           | Slack API documentation            |
| Google Sheets node requires appropriate sharing and API access setup for the target spreadsheet.   | Google Sheets API documentation    |
| OpenAI LangChain nodes require API keys and careful prompt engineering for quality responses.      | OpenAI API and LangChain guides    |
| Consider timezone settings for the Cron node to match team working hours.                          | n8n Cron node documentation        |

---

*Disclaimer:* The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.