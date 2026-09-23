Track tenant reference requests with forms, Gmail, and Google Sheets

https://n8nworkflows.xyz/workflows/track-tenant-reference-requests-with-forms--gmail--and-google-sheets-19674


# Track tenant reference requests with forms, Gmail, and Google Sheets

### 1. Workflow Overview

This workflow automates the tenant reference verification process from the initial applicant intake through landlord follow-up, reminder dispatch, feedback collection, and automated escalations. It minimizes manual administrative overhead for real estate agents by tracking requests, communicating with landlords, and alerting agents when reference checks stall.

The workflow logic is categorized into the following functional blocks:
- **1.1 Applicant Intake & Tracking:** Receives tenant reference data via an n8n Form, logs a pending tracking entry in Google Sheets, and handles failure flows if writing to the spreadsheet fails.
- **1.2 Landlord Request Dispatch:** Emails the previous landlord with a link to the feedback form containing the submission identifier, handling delivery failure notifications to the agent.
- **1.3 Delayed Recheck & Reminder:** Waits for 48 hours, retrieves the tracker record, checks if the status remains pending, sends a reminder email to the landlord, updates the sheet status, and alerts the agent.
- **1.4 Landlord Feedback Processing:** Receives submitted feedback via an n8n Form, updates the tracking sheet with structured questionnaire answers, and logs raw payloads to a backup sheet upon failure.
- **1.5 Scheduled Overdue Scan & Escalation:** Executes an hourly check to identify pending requests exceeding a 49-hour threshold, notifies the agent with an escalation notice, and updates the tracking status to escalated.

---

### 2. Block-by-Block Analysis

#### 2.1 Applicant Intake & Tracking
- **Overview:** This block captures the initial tenant reference request submitted by the agent, records a pending row in Google Sheets, and triggers a fallback alert to the agent via Gmail if the spreadsheet append operation fails.
- **Nodes Involved:** `When Form Submitted`, `Append to Tenant Tracker`, `Notify Sheet Write Failure`.
- **Node Details:**
  - **When Form Submitted**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` — Acts as the workflow's primary entry point, exposing an interactive form for agents.
    - *Configuration Choices:* Configured with form fields: Applicant Name (Required), Previous Landlord / Letting Agent Email (Required, Email type), Agent Email (Required, Email type), and Property Address / Unit Number (Required).
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connects to `Append to Tenant Tracker`.
    - *Version-Specific:* TypeVersion 2.6.
    - *Edge Cases/Failures:* Form validation failures handled natively by the UI; empty or malformed email inputs will prevent downstream execution.
  - **Append to Tenant Tracker**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Appends a new row to the tracking spreadsheet.
    - *Configuration Choices:* Operation set to `append`, target document ID set via selector, worksheet set to default (`Sheet1`). Configured with `onError: continueErrorOutput`.
    - *Key Expressions/Variables:* Maps incoming form fields automatically.
    - *Input/Output:* Input from `When Form Submitted`. Main success output connects to `Email Reference Request`. Error output connects to `Notify Sheet Write Failure`.
    - *Version-Specific:* TypeVersion 4.7.
    - *Edge Cases/Failures:* Spreadsheet quota limits, incorrect column mapping, or connection drops trigger the error branch.
  - **Notify Sheet Write Failure**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` — Sends an alert email to the agent.
    - *Configuration Choices:* Subject set dynamically; message body informs the agent of the logging failure.
    - *Key Expressions/Variables:* `{{ $('When Form Submitted').item.json['Agent Email'] }}` and `{{ $('When Form Submitted').item.json['Applicant Name'] }}`.
    - *Input/Output:* Input from the error branch of `Append to Tenant Tracker`. No outgoing connections.
    - *Version-Specific:* TypeVersion 2.2.
    - *Edge Cases/Failures:* SMTP authentication errors or invalid agent email format.

#### 2.2 Landlord Request Dispatch
- **Overview:** This block sends the reference request email to the landlord with a unique feedback form URL containing the submission ID, and routes outbound email failures to an agent alert node.
- **Nodes Involved:** `Email Reference Request`, `Notify Email Failure`.
- **Node Details:**
  - **Email Reference Request**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` — Outbound email node for contacting the landlord.
    - *Configuration Choices:* Uses SMTP/Email credentials. Configured with `onError: continueErrorOutput`.
    - *Key Expressions/Variables:* `{{ $json['Landlord Email'] }}`, `{{ $json['Applicant Name'] }}`, `{{ $json['Property Address'] }}`, and `{{ encodeURIComponent($json['Submission ID']) }}`.
    - *Input/Output:* Input from `Append to Tenant Tracker`. Success output connects to `Wait 48 Hours`. Error output connects to `Notify Email Failure`.
    - *Version-Specific:* TypeVersion 2.1.
    - *Edge Cases/Failures:* Invalid recipient email addresses or mail server timeouts.
  - **Notify Email Failure**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` — Sends an alert to the agent regarding landlord email delivery failure.
    - *Configuration Choices:* Targets the agent's email with a customized warning message.
    - *Key Expressions/Variables:* `{{ $('When Form Submitted').item.json['Agent Email'] }}` and `{{ $('When Form Submitted').item.json['Applicant Name'] }}`.
    - *Input/Output:* Input from the error output of `Email Reference Request`. No outgoing connections.
    - *Version-Specific:* TypeVersion 2.2.
    - *Edge Cases/Failures:* Gmail authentication failure.

#### 2.3 Delayed Recheck & Reminder
- **Overview:** Pauses execution for 48 hours to allow the landlord time to respond, retrieves the tracking row from Google Sheets, verifies if the status remains pending, and triggers reminder workflows.
- **Nodes Involved:** `Wait 48 Hours`, `Read For Reminder Check`, `Check Pending Status`.
- **Node Details:**
  - **Wait 48 Hours**
    - *Type and Technical Role:* `n8n-nodes-base.wait` — Delays workflow execution.
    - *Configuration Choices:* Amount set to `48`, unit set to `hours`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Input from `Email Reference Request`; output connects to `Read For Reminder Check`.
    - *Version-Specific:* TypeVersion 1.1.
    - *Edge Cases/Failures:* Workflow instance restarts or long-term server downtime could impact precise timer execution.
  - **Read For Reminder Check**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Retrieves rows from Google Sheets.
    - *Configuration Choices:* Operation set to read/lookup (`filtersUI`), matching `Submission ID` against `{{ $execution.id }}`.
    - *Key Expressions/Variables:* `{{ $execution.id }}`.
    - *Input/Output:* Input from `Wait 48 Hours`; output connects to `Check Pending Status`.
    - *Version-Specific:* TypeVersion 4.7.
    - *Edge Cases/Failures:* Missing execution ID match due to record deletion or modification in the sheet.
  - **Check Pending Status**
    - *Type and Technical Role:* `n8n-nodes-base.if` — Conditional branching node.
    - *Configuration Choices:* Evaluates whether `{{ $json.Status }}` equals `Pending`.
    - *Key Expressions/Variables:* `{{ $json.Status }}`.
    - *Input/Output:* Input from `Read For Reminder Check`; true branch connects to `Email Reminder to Landlord`.
    - *Version-Specific:* TypeVersion 2.3.
    - *Edge Cases/Failures:* Status value mismatch due to manual sheet edits.

#### 2.4 Reminder and Agent Notice
- **Overview:** Sends a reminder email to the landlord, updates the tracking spreadsheet status to "Reminder Sent", and notifies the agent that a reminder has been issued.
- **Nodes Involved:** `Email Reminder to Landlord`, `Update Reminder Sent Status`, `Email Alert to Agent`.
- **Node Details:**
  - **Email Reminder to Landlord**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` — Sends a follow-up email to the landlord.
    - *Configuration Choices:* Uses SMTP credentials.
    - *Key Expressions/Variables:* `{{ $json['Landlord Email'] }}`, `{{ $json['Applicant Name'] }}`, `{{ $json['Property Address'] }}`, and `{{ encodeURIComponent($json['Submission ID']) }}`.
    - *Input/Output:* Input from `Check Pending Status`; output connects to `Update Reminder Sent Status`.
    - *Version-Specific:* TypeVersion 2.1.
    - *Edge Cases/Failures:* Mail delivery failures.
  - **Update Reminder Sent Status**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Updates existing row data.
    - *Configuration Choices:* Operation set to `update`, matching by `Submission ID` and updating `Status` to `Reminder Sent`.
    - *Key Expressions/Variables:* `{{ $execution.id }}` for `Submission ID`.
    - *Input/Output:* Input from `Email Reminder to Landlord`; output connects to `Email Alert to Agent`.
    - *Version-Specific:* TypeVersion 4.7.
    - *Edge Cases/Failures:* Matching column failure or sheet locking.
  - **Email Alert to Agent**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` — Notifies the agent about the reminder.
    - *Configuration Choices:* Targets agent's email address using node reference data.
    - *Key Expressions/Variables:* `{{ $('Read For Reminder Check').item.json['Agent Email'] }}`, `{{ $('Read For Reminder Check').item.json['Applicant Name'] }}`, and property details.
    - *Input/Output:* Input from `Update Reminder Sent Status`. No outgoing connections.
    - *Version-Specific:* TypeVersion 2.1.
    - *Edge Cases/Failures:* Invalid agent email format.

#### 2.5 Landlord Response Logging
- **Overview:** Receives landlord feedback via a dedicated form, updates the matching Google Sheets row with structured survey answers and a "Responded" status, and writes raw backups if the primary update fails.
- **Nodes Involved:** `When Feedback Received`, `Append Landlord Response`, `Log Raw Landlord Feedback`.
- **Node Details:**
  - **When Feedback Received**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` — Entry point form for landlord feedback collection.
    - *Configuration Choices:* Configured with radio buttons for rent payment history, property damage, future rental eligibility, proper notice given, overall recommendation, and a text field for `Submission ID`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connects to `Append Landlord Response`.
    - *Version-Specific:* TypeVersion 2.6.
    - *Edge Cases/Failures:* Missing or incorrect `Submission ID` input by the landlord.
  - **Append Landlord Response**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Updates or appends row data based on submission ID.
    - *Configuration Choices:* Operation set to `appendOrUpdate`, matching columns set to `Submission ID`. Configured with `onError: continueErrorOutput`.
    - *Key Expressions/Variables:* Maps feedback questionnaire fields, `{{ $json['Submission ID'] }}`, status set to `Responded`, and timestamp variables.
    - *Input/Output:* Input from `When Feedback Received`. Success output is empty; error output connects to `Log Raw Landlord Feedback`.
    - *Version-Specific:* TypeVersion 4.7.
    - *Edge Cases/Failures:* API limits or mismatched sheet schema.
  - **Log Raw Landlord Feedback**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Backup logging mechanism.
    - *Configuration Choices:* Operation set to `append` targeting the `Raw Response Backup` worksheet.
    - *Key Expressions/Variables:* `{{ new Date().toISOString() }}`, `{{ $json['Submission ID'] }}`, and `{{ JSON.stringify($json) }}`.
    - *Input/Output:* Input from error output of `Append Landlord Response`. No outgoing connections.
    - *Version-Specific:* TypeVersion 4.7.
    - *Edge Cases/Failures:* Backup sheet misconfiguration.

#### 2.6 Scheduled Overdue Scan & Escalation
- **Overview:** Periodically scans the tracking sheet for pending records, evaluates whether they have been pending for more than 49 hours, and handles escalations by emailing the agent and updating the sheet status.
- **Nodes Involved:** `Hourly Schedule Trigger`, `Fetch Overdue Entries`, `Check If Overdue 49 Hours`, `Email Escalation to Agent`, `Update to Escalated Status`.
- **Node Details:**
  - **Hourly Schedule Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` — Time-based trigger node.
    - *Configuration Choices:* Interval configured to trigger every hour.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connects to `Fetch Overdue Entries`.
    - *Version-Specific:* TypeVersion 1.4.
    - *Edge Cases/Failures:* Server downtime during scheduled intervals.
  - **Fetch Overdue Entries**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Reads records matching filter criteria.
    - *Configuration Choices:* Operation set to read, filtering rows where `Status` equals `Pending`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Input from `Hourly Schedule Trigger`; output connects to `Check If Overdue 49 Hours`.
    - *Version-Specific:* TypeVersion 4.7.
    - *Edge Cases/Failures:* Sheet connectivity issues.
  - **Check If Overdue 49 Hours**
    - *Type and Technical Role:* `n8n-nodes-base.if` — Date/time evaluation node.
    - *Configuration Choices:* Condition checks if `{{ $json.Timestamp }}` is before `={{ $now.minus({ hours: 49 }).toISO() }}`.
    - *Key Expressions/Variables:* `{{ $json.Timestamp }}` and `{{ $now.minus({ hours: 49 }).toISO() }}`.
    - *Input/Output:* Input from `Fetch Overdue Entries`; true branch connects to `Email Escalation to Agent`.
    - *Version-Specific:* TypeVersion 2.3.
    - *Edge Cases/Failures:* Timestamp string formatting mismatches.
  - **Email Escalation to Agent**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` — Sends urgent escalation notices to the agent.
    - *Configuration Choices:* Uses SMTP credentials.
    - *Key Expressions/Variables:* `{{ $json['Applicant Name'] }}`, `{{ $json['Property Address'] }}`, `{{ $json['Agent Email'] }}`, and `{{ $json['Submission ID'] }}`.
    - *Input/Output:* Input from `Check If Overdue 49 Hours`; output connects to `Update to Escalated Status`.
    - *Version-Specific:* TypeVersion 2.1.
    - *Edge Cases/Failures:* Outbound mail transmission errors.
  - **Update to Escalated Status**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Updates record status.
    - *Configuration Choices:* Operation set to `update`, matching by `Submission ID` and updating `Status` to `Escalated`.
    - *Key Expressions/Variables:* `{{ $('Fetch Overdue Entries').item.json['Submission ID'] }}`.
    - *Input/Output:* Input from `Email Escalation to Agent`. No outgoing connections.
    - *Version-Specific:* TypeVersion 4.7.
    - *Edge Cases/Failures:* Sheet sync conflicts or missing record IDs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Overview | None | None | Tenant Reference Verification & Screening Automation<br><br>### How it works<br><br>This workflow automates tenant reference verification from applicant intake through landlord follow-up and escalation. It creates a pending tracking record, emails the landlord for a reference, waits and checks whether the request is still pending, then sends reminders and notifies the agent as needed. A separate landlord feedback form updates the record and backs up the raw response, while a scheduled scan escalates overdue pending references after 49 hours.<br><br>### Setup steps<br><br>- Connect Google Sheets credentials and configure each Sheets node with the correct spreadsheet, worksheet, lookup keys, and status columns.<br>- Configure the applicant intake and landlord feedback forms so their fields match the sheet columns and email template variables.<br>- Connect email/Gmail credentials for landlord requests, reminder emails, agent alerts, and failure notifications.<br>- Set the Wait node duration and Schedule Trigger frequency to match the desired reminder and escalation timing.<br>- Review recipient addresses, subject lines, and message bodies for all email nodes before activating the workflow.<br><br>### Customization<br><br>You can adjust the pending, reminder-sent, responded, and escalated statuses, change the reminder delay or 49-hour escalation threshold, and tailor the email templates for different rental processes. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Applicant intake tracking<br><br>Captures a new applicant submission, creates the initial pending record in Google Sheets, and sends an alert if the sheet write fails. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Landlord request email<br><br>Sends the initial reference request to the landlord and triggers a failure alert if that outbound email cannot be sent. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Delayed pending recheck<br><br>Waits after the initial request, retrieves the applicant row again, and checks whether the reference request is still pending. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Reminder and agent notice<br><br>If the request remains pending, sends a reminder to the landlord, updates the sheet status, and notifies the agent. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Landlord response logging<br><br>Receives landlord feedback, marks the applicant record as responded, and stores a raw backup of the submitted response. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Scheduled overdue scan<br><br>Runs on a schedule, retrieves pending rows, and checks which requests have exceeded the 49-hour overdue threshold. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Escalation handling<br><br>Emails the agent about overdue pending references and updates the sheet status to escalated. |
| When Form Submitted | n8n-nodes-base.formTrigger | Intake Form Trigger | None | Append to Tenant Tracker | Applicant intake tracking |
| Append to Tenant Tracker | n8n-nodes-base.googleSheets | Log Initial Record | When Form Submitted | Email Reference Request, Notify Sheet Write Failure | Applicant intake tracking |
| Email Reference Request | n8n-nodes-base.emailSend | Send Landlord Request | Append to Tenant Tracker | Wait 48 Hours, Notify Email Failure | Landlord request email |
| Wait 48 Hours | n8n-nodes-base.wait | Delay Execution | Email Reference Request | Read For Reminder Check | Delayed pending recheck |
| Read For Reminder Check | n8n-nodes-base.googleSheets | Query Record for Reminder | Wait 48 Hours | Check Pending Status | Delayed pending recheck |
| Check Pending Status | n8n-nodes-base.if | Verify Pending Status | Read For Reminder Check | Email Reminder to Landlord | Delayed pending recheck |
| Email Reminder to Landlord | n8n-nodes-base.emailSend | Send Reminder Email | Check Pending Status | Update Reminder Sent Status | Reminder and agent notice |
| Update Reminder Sent Status | n8n-nodes-base.googleSheets | Update Sheet Status | Email Reminder to Landlord | Email Alert to Agent | Reminder and agent notice |
| Email Alert to Agent | n8n-nodes-base.emailSend | Notify Agent of Reminder | Update Reminder Sent Status | None | Reminder and agent notice |
| When Feedback Received | n8n-nodes-base.formTrigger | Landlord Feedback Trigger | None | Append Landlord Response | Landlord response logging |
| Append Landlord Response | n8n-nodes-base.googleSheets | Update Feedback in Sheet | When Feedback Received | Log Raw Landlord Feedback | Landlord response logging |
| Hourly Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled Run Trigger | None | Fetch Overdue Entries | Scheduled overdue scan |
| Fetch Overdue Entries | n8n-nodes-base.googleSheets | Fetch Pending Rows | Hourly Schedule Trigger | Check If Overdue 49 Hours | Scheduled overdue scan |
| Check If Overdue 49 Hours | n8n-nodes-base.if | Check Exceeded Threshold | Fetch Overdue Entries | Email Escalation to Agent | Scheduled overdue scan |
| Email Escalation to Agent | n8n-nodes-base.emailSend | Send Escalation Notice | Check If Overdue 49 Hours | Update to Escalated Status | Escalation handling |
| Update to Escalated Status | n8n-nodes-base.googleSheets | Update to Escalated | Email Escalation to Agent | None | Escalation handling |
| Notify Sheet Write Failure | n8n-nodes-base.gmail | Alert Agent on Sheet Error | Append to Tenant Tracker | None | Applicant intake tracking |
| Notify Email Failure | n8n-nodes-base.gmail | Alert Agent on Email Error | Email Reference Request | None | Landlord request email |
| Log Raw Landlord Feedback | n8n-nodes-base.googleSheets | Backup Landlord Feedback | Append Landlord Response | None | Landlord response logging |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to reconstruct the workflow manually in n8n:

1. **Create Intake Form Trigger (`When Form Submitted`)**
   - Add a **Form Trigger** node. Set title to `Tenant Reference Request`.
   - Add fields: `Applicant Name` (Text, Required), `Previous Landlord / Letting Agent Email` (Email, Required), `Agent Email` (Email, Required), and `Property Address / Unit Number` (Text, Required).
2. **Create Initial Tracking Node (`Append to Tenant Tracker`)**
   - Add a **Google Sheets** node connected to the success output of `When Form Submitted`.
   - Set operation to `append`. Select the "Tenant Reference Tracker" document and `Sheet1` worksheet.
   - Configure error handling (`Settings > On Error`) to **Continue (Using Error Output)**.
3. **Create Sheet Failure Notification (`Notify Sheet Write Failure`)**
   - Add a **Gmail** node connected to the error output of `Append to Tenant Tracker`.
   - Set recipient to `{{ $('When Form Submitted').item.json['Agent Email'] }}` and configure subject/body to alert the agent of write failures.
4. **Create Landlord Request Email (`Email Reference Request`)**
   - Add an **Email Send** (SMTP) node connected to the main success output of `Append to Tenant Tracker`.
   - Set `To Email` to `{{ $json['Landlord Email'] }}`.
   - Set Subject to `=Tenant Reference Request for {{ $json['Applicant Name'] }}`.
   - Set HTML body with template links pointing to your feedback form URL, appending `?Submission%20ID={{ encodeURIComponent($json['Submission ID']) }}`.
   - Configure error handling to **Continue (Using Error Output)**.
5. **Create Email Failure Notification (`Notify Email Failure`)**
   - Add a **Gmail** node connected to the error output of `Email Reference Request`.
   - Set recipient to agent and configure message content for delivery failures.
6. **Create Delay and Recheck Nodes**
   - Add a **Wait** node (`Wait 48 Hours`) connected to the success output of `Email Reference Request`. Set unit to `hours` and amount to `48`.
   - Add a **Google Sheets** node (`Read For Reminder Check`) connected to `Wait 48 Hours`. Set operation to read with filters matching `Submission ID` against `{{ $execution.id }}`.
   - Add an **If** node (`Check Pending Status`) connected to `Read For Reminder Check`. Set condition to check if `{{ $json.Status }}` equals `Pending`.
7. **Create Reminder & Agent Alert Nodes**
   - Add an **Email Send** node (`Email Reminder to Landlord`) connected to the true branch of `Check Pending Status`. Set recipient to landlord email and include form URL with submission ID.
   - Add a **Google Sheets** node (`Update Reminder Sent Status`) connected to `Email Reminder to Landlord`. Set operation to `update`, match by `Submission ID` (`{{ $execution.id }}`), and set `Status` to `Reminder Sent`.
   - Add an **Email Send** node (`Email Alert to Agent`) connected to `Update Reminder Sent Status`. Set `To Email` to `{{ $('Read For Reminder Check').item.json['Agent Email'] }}` and configure the alert message.
8. **Create Feedback Form and Processing Branch**
   - Add a **Form Trigger** node (`When Feedback Received`). Set form title to `Tenant Reference Feedback`.
   - Add radio button fields: `Rent on time?`, `Any property damage?`, `Would rent again?`, `Notice given properly?`, `Overall recommend?` (options: Yes/No), and a text field for `Submission ID`.
   - Add a **Google Sheets** node (`Append Landlord Response`) connected to `When Feedback Received`. Set operation to `appendOrUpdate`, matching on `Submission ID`. Map feedback fields and set `Status` to `Responded`. Enable error handling to **Continue (Using Error Output)**.
   - Add a **Google Sheets** node (`Log Raw Landlord Feedback`) connected to the error output of `Append Landlord Response`. Target the `Raw Response Backup` sheet, mapping `Submission ID`, timestamp, and `{{ JSON.stringify($json) }}`.
9. **Create Scheduled Overdue Scan Branch**
   - Add a **Schedule Trigger** node (`Hourly Schedule Trigger`). Set rule interval to hours (`1`).
   - Add a **Google Sheets** node (`Fetch Overdue Entries`) connected to the trigger. Set operation to read, filtering rows where `Status` equals `Pending`.
   - Add an **If** node (`Check If Overdue 49 Hours`) connected to `Fetch Overdue Entries`. Set condition to evaluate if `{{ $json.Timestamp }}` is before `={{ $now.minus({ hours: 49 }).toISO() }}`.
   - Add an **Email Send** node (`Email Escalation to Agent`) connected to the true branch. Set `To Email` to `{{ $json['Agent Email'] }}` and configure the escalation message.
   - Add a **Google Sheets** node (`Update to Escalated Status`) connected to `Email Escalation to Agent`. Set operation to `update`, matching by `Submission ID` (`{{ $('Fetch Overdue Entries').item.json['Submission ID'] }}`), and set `Status` to `Escalated`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Tenant Reference Verification & Screening Automation Template | Workflow architecture designed for automated residential leasing and tenant screening. |
| Google Sheets Database Requirements | Requires a spreadsheet named `Tenant Reference Tracker` with a primary worksheet (`Sheet1`) containing columns for `Submission ID`, `Applicant Name`, `Property Address`, `Landlord Email`, `Agent Email`, `Status`, `Timestamp`, questionnaire fields, and a secondary worksheet named `Raw Response Backup`. |