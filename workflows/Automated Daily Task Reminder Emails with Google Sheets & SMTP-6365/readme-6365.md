Automated Daily Task Reminder Emails with Google Sheets & SMTP

https://n8nworkflows.xyz/workflows/automated-daily-task-reminder-emails-with-google-sheets---smtp-6365


# Automated Daily Task Reminder Emails with Google Sheets & SMTP

### 1. Workflow Overview

This workflow automates the process of sending daily reminder emails for tasks listed in a Google Sheets content calendar. It is designed to run once per day, read the tasks scheduled for that day, filter them, generate personalized email content, send the reminder emails via SMTP, and update the Google Sheet to reflect the sent status.

**Logical Blocks:**

- **1.1 Daily Trigger:** Initiates the workflow execution daily.
- **1.2 Data Retrieval:** Reads the entire content calendar from Google Sheets.
- **1.3 Task Filtering:** Filters tasks scheduled for the current day.
- **1.4 Email Preparation and Sending:** Prepares and sends reminder emails.
- **1.5 Post-Send Update:** Updates the Google Sheet row to mark the task as reminded.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger

- **Overview:**  
  This block triggers the entire workflow automatically on a daily schedule, ensuring reminders are sent out every day without manual intervention.

- **Nodes Involved:**  
  - Daily Reminder Trigger

- **Node Details:**  
  - **Daily Reminder Trigger**  
    - **Type:** Cron Trigger  
    - **Role:** Initiates the workflow daily at a configured time (default or customized in node parameters).  
    - **Configuration:** No specific parameters shown, but typically configured to run once per day at a set hour.  
    - **Input/Output:** No input; output triggers the next node (Read Content Calendar).  
    - **Edge Cases:** Cron misconfiguration could cause no runs or multiple runs; time zone mismatch may lead to unexpected trigger times.

#### 2.2 Data Retrieval

- **Overview:**  
  Reads the entire content calendar from a Google Sheet to obtain task data for processing.

- **Nodes Involved:**  
  - Read Content Calendar

- **Node Details:**  
  - **Read Content Calendar**  
    - **Type:** Google Sheets (Read)  
    - **Role:** Fetches all task rows from a designated Google Sheet.  
    - **Configuration:** Uses Google Sheets API credentials; reads from a specified spreadsheet and sheet (parameters not detailed).  
    - **Input:** Triggered by Daily Reminder Trigger.  
    - **Output:** Passes the spreadsheet data downstream for filtering.  
    - **Edge Cases:** Possible errors include authentication failures, API quota limits, empty sheets, or incorrect sheet names.

#### 2.3 Task Filtering

- **Overview:**  
  Filters the list of tasks to isolate only those scheduled for the current day.

- **Nodes Involved:**  
  - Filter Today's Tasks

- **Node Details:**  
  - **Filter Today's Tasks**  
    - **Type:** Filter Node  
    - **Role:** Applies a condition to only pass tasks scheduled for the current date.  
    - **Configuration:** Likely uses an expression comparing task date fields with the current date (e.g., via `{{$now}}` or similar).  
    - **Input:** Receives all tasks from Read Content Calendar.  
    - **Output:** Passes filtered tasks to Send Reminder Email node.  
    - **Edge Cases:** Date format mismatches, timezone issues, tasks without dates, or empty filtered results.

#### 2.4 Email Preparation and Sending

- **Overview:**  
  Prepares personalized email content for each filtered task and sends reminder emails using SMTP or a configured email service.

- **Nodes Involved:**  
  - Send Reminder Email  
  - Code

- **Node Details:**  
  - **Send Reminder Email**  
    - **Type:** Email Send (SMTP or other email provider)  
    - **Role:** Sends out the reminder email to the task owner or designated recipient.  
    - **Configuration:**  
        - SMTP or email credentials configured externally (e.g., OAuth2 or SMTP username/password).  
        - Email fields (To, Subject, Body) dynamically built from task data, likely using expressions or parameters.  
    - **Input:** Receives filtered task(s) from Filter Today's Tasks.  
    - **Output:** Passes email sending result to the Code node.  
    - **Edge Cases:** Email send failures, invalid email addresses, SMTP authentication errors, rate limits.

  - **Code**  
    - **Type:** Code Node (JavaScript)  
    - **Role:** Processes email send results, possibly formats data or prepares update payload for Google Sheets.  
    - **Configuration:** Contains custom JavaScript code (details not shown) to handle post-email logic.  
    - **Input:** Receives output from Send Reminder Email.  
    - **Output:** Passes data to Update row in sheet node.  
    - **Edge Cases:** Coding errors, runtime exceptions, data format issues.

#### 2.5 Post-Send Update

- **Overview:**  
  Updates the corresponding row in Google Sheets to mark that the reminder email was sent, enabling tracking and preventing duplicate reminders.

- **Nodes Involved:**  
  - Update row in sheet

- **Node Details:**  
  - **Update row in sheet**  
    - **Type:** Google Sheets (Update)  
    - **Role:** Updates a specific row in the sheet, typically setting a "Reminder Sent" flag or timestamp.  
    - **Configuration:** Requires spreadsheet ID, sheet name, and row number (likely passed from previous Code node).  
    - **Input:** Receives data from Code node containing row identification and update values.  
    - **Output:** Final step; no further nodes connected.  
    - **Edge Cases:** API authentication failures, row not found errors, concurrent modification conflicts.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                   | Input Node(s)         | Output Node(s)         | Sticky Note |
|-----------------------|---------------------|---------------------------------|-----------------------|------------------------|-------------|
| Daily Reminder Trigger | Cron Trigger        | Initiates workflow daily         | —                     | Read Content Calendar  |             |
| Read Content Calendar  | Google Sheets Read  | Reads task data from Google Sheets | Daily Reminder Trigger | Filter Today's Tasks   |             |
| Filter Today's Tasks   | Filter              | Filters tasks for current day    | Read Content Calendar  | Send Reminder Email    |             |
| Send Reminder Email    | Email Send          | Sends reminder emails            | Filter Today's Tasks   | Code                   |             |
| Code                  | Code (JavaScript)   | Processes email results & prepares update data | Send Reminder Email | Update row in sheet    |             |
| Update row in sheet    | Google Sheets Update | Updates sheet to mark sent reminders | Code               | —                      |             |
| Sticky Note            | Sticky Note         | —                               | —                     | —                      |             |
| Sticky Note1           | Sticky Note         | —                               | —                     | —                      |             |
| Sticky Note2           | Sticky Note         | —                               | —                     | —                      |             |
| Sticky Note3           | Sticky Note         | —                               | —                     | —                      |             |
| Sticky Note4           | Sticky Note         | —                               | —                     | —                      |             |
| Sticky Note5           | Sticky Note         | —                               | —                     | —                      |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Name: Daily Reminder Trigger  
   - Type: Cron Trigger  
   - Configure to run once daily at your preferred time (e.g., 8:00 AM).  
   - No input connections.

2. **Create Google Sheets Read Node**  
   - Name: Read Content Calendar  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Configure credentials for Google Sheets access.  
   - Set Spreadsheet ID and Sheet Name where task data is stored.  
   - Connect input from Daily Reminder Trigger node.

3. **Create Filter Node**  
   - Name: Filter Today's Tasks  
   - Type: Filter  
   - Configuration: Set condition to filter rows where the task date equals the current date. Example expression:  
     `{{ $json["TaskDate"] === $now.format("YYYY-MM-DD") }}`  
   - Connect input from Read Content Calendar.

4. **Create Email Send Node**  
   - Name: Send Reminder Email  
   - Type: Email Send  
   - Configure SMTP or email credentials (SMTP server, port, username, password, or OAuth2).  
   - Set up dynamic email fields:  
     - To: use task-assigned email from filtered data.  
     - Subject: e.g., "Daily Task Reminder: {{ $json["TaskName"] }}"  
     - Body: compose message using task details.  
   - Connect input from Filter Today's Tasks.

5. **Create Code Node**  
   - Name: Code  
   - Type: Code  
   - Write JavaScript code to process the email sending result and build data for updating the Google Sheet row.  
   - For example, extract row number and prepare fields to mark the reminder as sent.  
   - Connect input from Send Reminder Email.

6. **Create Google Sheets Update Node**  
   - Name: Update row in sheet  
   - Type: Google Sheets  
   - Operation: Update Row  
   - Use the same credentials and spreadsheet as Read Content Calendar.  
   - Configure to update the row identified by the Code node with a "Reminder Sent" flag or timestamp.  
   - Connect input from Code node.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                             |
|----------------------------------------------------------------------------------------------|---------------------------------------------|
| Ensure Google Sheets API is enabled and OAuth2 credentials are properly configured in n8n.  | Google Cloud Console & n8n credential setup |
| SMTP credentials must allow sending emails from your chosen address; verify settings ahead. | SMTP provider documentation                  |
| Date/time comparisons require consistent timezone handling to avoid missing tasks.          | Use n8n expressions with timezone awareness |
| For advanced email formatting, HTML body can be used within Send Reminder Email node.       | n8n Email Send node docs                      |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies fully with all content policies and contains no illegal or protected elements. All data processed is legal and publicly accessible.