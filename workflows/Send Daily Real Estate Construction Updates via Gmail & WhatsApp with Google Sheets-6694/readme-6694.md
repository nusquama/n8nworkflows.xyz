Send Daily Real Estate Construction Updates via Gmail & WhatsApp with Google Sheets

https://n8nworkflows.xyz/workflows/send-daily-real-estate-construction-updates-via-gmail---whatsapp-with-google-sheets-6694


# Send Daily Real Estate Construction Updates via Gmail & WhatsApp with Google Sheets

### 1. Workflow Overview

This workflow automates the daily distribution of real estate construction updates to buyers and stakeholders via Gmail and WhatsApp. It targets project managers and clients who require timely, summarized progress reports extracted from a Google Sheet maintained by contractors.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow every day at 8:00 AM IST.
- **1.2 Google Sheet Data Preparation**: Configures Google Sheet parameters and reads data containing construction reports.
- **1.3 Data Filtering**: Filters reports to include only those dated for the current day.
- **1.4 Conditional Check**: Determines if there are any reports for today to process.
- **1.5 Notification Recipients Setup**: Defines email and WhatsApp recipients for notifications.
- **1.6 Content Creation**: Builds detailed notification messages for email and WhatsApp.
- **1.7 Notification Dispatch**: Sends reports via WhatsApp and email to recipients.
- **1.8 Logging**: Logs the notification dispatch activities for audit purposes.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
Triggers the workflow daily at 8:00 AM Indian Standard Time to start the process of checking and sending construction updates.

- **Nodes Involved:**  
  - Daily Trigger (8:00 AM IST)  
  - Sticky Note - Trigger

- **Node Details:**

  - **Daily Trigger (8:00 AM IST)**  
    - *Type & Role:* Schedule Trigger node; triggers workflow execution on a fixed daily schedule.  
    - *Configuration:* Set to trigger at hour 8 (8:00 AM IST).  
    - *Input/Output:* No input; output triggers next node.  
    - *Edge Cases:* Workflow won’t run if n8n server is down or if scheduler fails. Timezone issues may arise if server time differs from IST.  
    - *Sticky Note:* "Triggers every day at 8:00 AM IST to check Google Sheet for new construction reports from contractors."

---

#### 2.2 Google Sheet Data Preparation

- **Overview:**  
Prepares configuration parameters for accessing the Google Sheet and reads all data entries from the specified worksheet containing construction reports.

- **Nodes Involved:**  
  - Set Google Sheet Config  
  - Read Google Sheet Data  
  - Sticky Note - Config  
  - Sticky Note - Read Sheet

- **Node Details:**

  - **Set Google Sheet Config**  
    - *Type & Role:* Set node; defines constants for Google Sheet ID, sheet name, and today's date (formatted as DD/MM/YYYY).  
    - *Key Parameters:*  
      - Sheet_ID: Google Sheet document ID (hardcoded string).  
      - Sheet_Name: Worksheet/tab name ("Construction_Reports").  
      - Today_Date: Dynamic current date string using expression `{{$now.format('DD/MM/YYYY')}}`.  
    - *Input/Output:* Receives trigger from schedule, outputs JSON with config variables.  
    - *Potential Failures:* Incorrect Sheet ID or name will cause downstream nodes to fail. Date formatting must be consistent with Google Sheet data.  
    - *Sticky Note:* "Configure Google Sheet ID, sheet name, and today's date for filtering the latest reports."

  - **Read Google Sheet Data**  
    - *Type & Role:* Google Sheets node; reads entire data from the specified sheet.  
    - *Configuration:* Uses dynamic expressions to get Sheet Name and Document ID from previous node.  
    - *Credentials:* Uses OAuth2 credentials linked to Google Sheets account.  
    - *Input/Output:* Input from Set Google Sheet Config; outputs all rows as JSON items.  
    - *Potential Failures:* Authentication errors, API rate limits, or invalid sheet parameters can cause failure.  
    - *Sticky Note:* "Read all data from the Google Sheet containing contractor reports with project details, progress, and status."

---

#### 2.3 Data Filtering

- **Overview:**  
Filters the imported Google Sheet data to retain only reports matching today's date.

- **Nodes Involved:**  
  - Filter Today's Reports  
  - Sticky Note - Filter

- **Node Details:**

  - **Filter Today's Reports**  
    - *Type & Role:* Filter node; filters JSON items where the "Date" field equals today's date defined earlier.  
    - *Configuration:* Condition: `$json.Date === Today_Date` (using expression referencing Set Google Sheet Config node).  
    - *Input/Output:* Input from Read Google Sheet Data; outputs filtered data.  
    - *Edge Cases:* Date formats must match exactly; otherwise, filtering will exclude valid reports. Empty filtering results possible if no reports for today.  
    - *Sticky Note:* "Filter the sheet records to get only today's reports based on the Date column."

---

#### 2.4 Conditional Check

- **Overview:**  
Checks whether any reports were found after filtering. If yes, continues to prepare notifications; if no, workflow currently proceeds silently (no explicit notification to admin in this version).

- **Nodes Involved:**  
  - Check If Reports Found  
  - Sticky Note - Check Reports

- **Node Details:**

  - **Check If Reports Found**  
    - *Type & Role:* If node; conditionally routes workflow based on presence of data.  
    - *Configuration:* Checks if the count of incoming items is greater than 0 (`$runIndex > 0`).  
    - *Input/Output:* Input from Filter Today's Reports; outputs to Set Notification Recipients if true; no false branch connected (workflow ends silently on no data).  
    - *Edge Cases:* If no reports, no notifications are sent and no alerts generated. Could be enhanced with admin alert on no data.  
    - *Sticky Note:* "Check if any reports were found for today. If yes, send notifications. If no, send alert to admin."

---

#### 2.5 Notification Recipients Setup

- **Overview:**  
Defines the email addresses and WhatsApp numbers of users who should receive the daily updates.

- **Nodes Involved:**  
  - Set Notification Recipients  
  - Sticky Note - Contacts

- **Node Details:**

  - **Set Notification Recipients**  
    - *Type & Role:* Set node; stores comma-separated strings of email addresses and phone numbers.  
    - *Parameters:*  
      - Notification_Users: Emails (e.g., "user1@gmail.com,user2@gmail.com,projectmanager@company.com")  
      - WhatsApp_Numbers: Phone numbers with country code (e.g., "+919876543210,+919876543211,+919876543212")  
      - Admin_Email: Admin email address for alerts (currently unused).  
    - *Input/Output:* Input from Check If Reports Found; outputs to Create Notification Content.  
    - *Edge Cases:* Incorrectly formatted emails or phone numbers can cause notification failures.  
    - *Sticky Note:* "Email and WhatsApp contact lists for users who should receive daily construction report notifications."

---

#### 2.6 Content Creation

- **Overview:**  
Generates formatted notification messages summarizing today’s construction reports, tailored separately for email and WhatsApp.

- **Nodes Involved:**  
  - Create Notification Content  
  - Sticky Note - Send Notifications1

- **Node Details:**

  - **Create Notification Content**  
    - *Type & Role:* Set node; creates three key string fields:  
      - Report_Summary: A markdown-styled summary for email detailing each project’s status, progress, issues, budget, and next milestone.  
      - Email_Subject: Email subject line with date and number of projects updated.  
      - WhatsApp_Message: Concise WhatsApp message with bullet-point project updates and a note pointing to email for details.  
    - *Expressions:* Uses `.map()` on filtered reports to build multiline summaries dynamically.  
    - *Input/Output:* Input from Set Notification Recipients; outputs to Send WhatsApp Notification and Send Email Notification.  
    - *Edge Cases:* If any report fields are missing in the sheet, fallback or empty values are handled (e.g., Issues defaults to 'None reported'). Expression errors if property names mismatch.  
    - *Sticky Note:* "Create detailed construction reports."

---

#### 2.7 Notification Dispatch

- **Overview:**  
Sends the generated reports via WhatsApp and email to the specified recipients.

- **Nodes Involved:**  
  - Send WhatsApp Notification  
  - Send Email Notification  
  - Sticky Note - Send Notifications

- **Node Details:**

  - **Send WhatsApp Notification**  
    - *Type & Role:* WhatsApp node; sends text messages to specified numbers.  
    - *Configuration:*  
      - Text body taken from Create Notification Content node’s WhatsApp_Message field.  
      - Recipient phone number is currently set statically to "+919876543210" (hardcoded; does not iterate over multiple numbers).  
    - *Credentials:* Uses WhatsApp API credentials (named "WhatsApp-test").  
    - *Input/Output:* Input from Create Notification Content; outputs to Log Successful Notifications.  
    - *Edge Cases:* Hardcoded recipient limits notification to one number only; to notify all, iteration or splitting logic is needed. Authentication or sending errors may occur.  
    - *Sticky Note:* "Send detailed construction reports to all users via Email and WhatsApp with project summaries."

  - **Send Email Notification**  
    - *Type & Role:* Email Send node; sends email notifications with report summaries.  
    - *Configuration:*  
      - Email body uses Report_Summary field from Create Notification Content.  
      - Subject uses Email_Subject field.  
      - To addresses are comma-separated emails from Set Notification Recipients.  
      - From email is fixed as "reports@constructioncompany.com".  
      - Email format is plain text.  
    - *Credentials:* SMTP credentials (named "SMTP -test").  
    - *Input/Output:* Input from Create Notification Content; outputs to Log Successful Notifications.  
    - *Edge Cases:* Invalid SMTP credentials or blocked senders can cause failures. Large recipient lists may hit SMTP limits.  
    - *Sticky Note:* "Send detailed construction reports to all users via Email and WhatsApp with project summaries."

---

#### 2.8 Logging

- **Overview:**  
Records a summary log of the notification dispatch including timestamp, number of recipients, projects reported, and sheet name.

- **Nodes Involved:**  
  - Log Successful Notifications  
  - Sticky Note - Logging

- **Node Details:**

  - **Log Successful Notifications**  
    - *Type & Role:* Set node; formats and stores a success message log as strings.  
    - *Parameters:*  
      - Success_Log: Timestamped success message with recipient count and project count.  
      - Log_Status: Fixed string "COMPLETED".  
      - Projects_Reported: Comma-separated project names reported today.  
    - *Input/Output:* Input from both Send WhatsApp Notification and Send Email Notification converging here.  
    - *Edge Cases:* If notifications fail upstream, this log may be inaccurate; no error logging implemented.  
    - *Sticky Note:* "Log all notification activities with timestamps, recipient count, and project details for audit trail."

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                             | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                                     |
|---------------------------|-------------------------|---------------------------------------------|--------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Daily Trigger (8:00 AM IST) | Schedule Trigger        | Starts workflow daily at 8:00 AM IST        | -                        | Set Google Sheet Config        | Triggers every day at 8:00 AM IST to check Google Sheet for new construction reports from contractors. |
| Set Google Sheet Config    | Set                     | Defines Google Sheet ID, sheet name, date  | Daily Trigger             | Read Google Sheet Data         | Configure Google Sheet ID, sheet name, and today's date for filtering the latest reports.       |
| Read Google Sheet Data     | Google Sheets           | Reads all contractor reports from Google Sheet | Set Google Sheet Config   | Filter Today's Reports         | Read all data from the Google Sheet containing contractor reports with project details, progress, and status. |
| Filter Today's Reports     | Filter                  | Filters reports for today’s date            | Read Google Sheet Data    | Check If Reports Found         | Filter the sheet records to get only today's reports based on the Date column.                   |
| Check If Reports Found     | If                      | Checks if any reports exist for today       | Filter Today's Reports    | Set Notification Recipients    | Check if any reports were found for today. If yes, send notifications. If no, send alert to admin. |
| Set Notification Recipients | Set                     | Sets email & WhatsApp recipients            | Check If Reports Found    | Create Notification Content    | Email and WhatsApp contact lists for users who should receive daily construction report notifications. |
| Create Notification Content | Set                     | Creates email and WhatsApp message content | Set Notification Recipients | Send WhatsApp Notification, Send Email Notification | Create detailed construction reports.                                                          |
| Send WhatsApp Notification | WhatsApp                | Sends WhatsApp messages to recipients       | Create Notification Content | Log Successful Notifications   | Send detailed construction reports to all users via Email and WhatsApp with project summaries.  |
| Send Email Notification    | Email Send              | Sends email notifications                    | Create Notification Content | Log Successful Notifications   | Send detailed construction reports to all users via Email and WhatsApp with project summaries.  |
| Log Successful Notifications | Set                     | Logs notification success details            | Send WhatsApp Notification, Send Email Notification | -                             | Log all notification activities with timestamps, recipient count, and project details for audit trail. |
| Sticky Note - Trigger      | Sticky Note             | Explanatory note for trigger                  | -                        | -                             | Triggers every day at 8:00 AM IST to check Google Sheet for new construction reports from contractors. |
| Sticky Note - Config       | Sticky Note             | Explanatory note for Google Sheet config     | -                        | -                             | Configure Google Sheet ID, sheet name, and today's date for filtering the latest reports.       |
| Sticky Note - Read Sheet   | Sticky Note             | Explanatory note for reading sheet data      | -                        | -                             | Read all data from the Google Sheet containing contractor reports with project details, progress, and status. |
| Sticky Note - Filter       | Sticky Note             | Explanatory note for filtering today's reports| -                        | -                             | Filter the sheet records to get only today's reports based on the Date column.                   |
| Sticky Note - Check Reports| Sticky Note             | Explanatory note on checking reports presence| -                        | -                             | Check if any reports were found for today. If yes, send notifications. If no, send alert to admin. |
| Sticky Note - Contacts     | Sticky Note             | Explanatory note on notification recipients  | -                        | -                             | Email and WhatsApp contact lists for users who should receive daily construction report notifications. |
| Sticky Note - Send Notifications | Sticky Note        | Explanatory note on sending notifications    | -                        | -                             | Send detailed construction reports to all users via Email and WhatsApp with project summaries.  |
| Sticky Note - Send Notifications1| Sticky Note          | Explanatory note on content creation          | -                        | -                             | Create detailed construction reports.                                                          |
| Sticky Note - Logging      | Sticky Note             | Explanatory note on logging                   | -                        | -                             | Log all notification activities with timestamps, recipient count, and project details for audit trail. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**:  
   - Name: "Daily Trigger (8:00 AM IST)"  
   - Set to trigger daily at hour 8 (8:00 AM IST).  
   - No credentials needed.

2. **Create a Set node**:  
   - Name: "Set Google Sheet Config"  
   - Add three fields:  
     - `Sheet_ID` (string): Set to your Google Sheet ID (e.g., "1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms").  
     - `Sheet_Name` (string): Set to your worksheet name (e.g., "Construction_Reports").  
     - `Today_Date` (string): Set to expression `{{$now.format('DD/MM/YYYY')}}` to dynamically get today's date.

3. **Create a Google Sheets node**:  
   - Name: "Read Google Sheet Data"  
   - Operation: Read all rows  
   - Set Document ID: expression `={{ $json.Sheet_ID }}` from previous Set node  
   - Set Sheet Name: expression `={{ $json.Sheet_Name }}` from previous Set node  
   - Add Google Sheets OAuth2 credentials.

4. **Create a Filter node**:  
   - Name: "Filter Today's Reports"  
   - Condition: Filter where property `Date` equals `={{$json.Today_Date}}` (use value from Set Google Sheet Config node).  
   - Ensure date formats match exactly.

5. **Create an If node**:  
   - Name: "Check If Reports Found"  
   - Condition: `{{$runIndex}} > 0` to check if any items passed filter.  
   - Connect "true" output to next step; "false" can end workflow or be extended for admin alerting.

6. **Create a Set node**:  
   - Name: "Set Notification Recipients"  
   - Add three fields:  
     - `Notification_Users` (string): Comma-separated emails (e.g., "user1@gmail.com,user2@gmail.com,projectmanager@company.com").  
     - `WhatsApp_Numbers` (string): Comma-separated phone numbers with country code (e.g., "+919876543210,+919876543211,+919876543212").  
     - `Admin_Email` (string): Admin email for alerts (optional).

7. **Create a Set node**:  
   - Name: "Create Notification Content"  
   - Add fields with expressions:  
     - `Report_Summary`: Build a formatted multiline string summarizing projects from filtered reports using `.map()` expression iterating over filtered data.  
     - `Email_Subject`: e.g., `"🏗️ Daily Construction Reports - {{Today_Date}} ({{number_of_projects}} Projects Updated)"`.  
     - `WhatsApp_Message`: Concise multiline summary for WhatsApp, also using `.map()`.

8. **Create a WhatsApp node**:  
   - Name: "Send WhatsApp Notification"  
   - Operation: Send message  
   - Text Body: Use `WhatsApp_Message` from previous node.  
   - Recipient Phone Number: Initially set statically; to send to multiple, implement iteration or split logic.  
   - Set WhatsApp API credentials.

9. **Create an Email Send node**:  
   - Name: "Send Email Notification"  
   - To: Use `Notification_Users` from Set Notification Recipients node.  
   - From: Set a valid "from" email (e.g., "reports@constructioncompany.com").  
   - Subject: Use `Email_Subject` from Create Notification Content.  
   - Text: Use `Report_Summary`.  
   - Email format: Plain text.  
   - Configure SMTP credentials.

10. **Create a Set node**:  
    - Name: "Log Successful Notifications"  
    - Fields:  
      - `Success_Log`: Timestamped message with recipient count and projects reported.  
      - `Log_Status`: Fixed string "COMPLETED".  
      - `Projects_Reported`: List of project names from filtered reports.  
    - Connect outputs of both Send WhatsApp Notification and Send Email Notification into this node.

11. **Connect all nodes sequentially**:  
    - Daily Trigger → Set Google Sheet Config → Read Google Sheet Data → Filter Today's Reports → Check If Reports Found (true branch) → Set Notification Recipients → Create Notification Content → Send WhatsApp Notification and Send Email Notification → Log Successful Notifications.

12. **Add Sticky Notes** (optional but recommended) for clarity at each block with the content provided in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                        |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| This workflow uses the WhatsApp node with static recipient number; to support multiple recipients, iteration or splitting logic is required. | WhatsApp Sending Logic                                                |
| Date formatting consistency between Google Sheet and n8n is critical for filtering accuracy.                   | Use DD/MM/YYYY format in both                                        |
| SMTP credentials and Google Sheets OAuth2 must be preconfigured in n8n credentials settings.                   | n8n Credentials Setup                                                 |
| Admin alert on no reports found can be added for enhanced monitoring.                                          | Suggested enhancement                                                 |
| Workflow uses plain text emails; consider HTML format for better presentation if desired.                      | Email Formatting                                                     |
| Workflow respects n8n’s rate limits; sending many WhatsApp messages may require batching or delays.           | Rate Limiting Considerations                                          |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.