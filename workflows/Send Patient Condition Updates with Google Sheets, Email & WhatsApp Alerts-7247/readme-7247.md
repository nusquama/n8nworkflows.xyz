Send Patient Condition Updates with Google Sheets, Email & WhatsApp Alerts

https://n8nworkflows.xyz/workflows/send-patient-condition-updates-with-google-sheets--email---whatsapp-alerts-7247


# Send Patient Condition Updates with Google Sheets, Email & WhatsApp Alerts

---

### 1. Workflow Overview

This workflow automates the daily process of sending patient condition updates to assigned doctors using Google Sheets data as the source. It targets healthcare scenarios where patient status monitoring and timely communication to medical staff are critical. The workflow runs every morning at 8 AM and performs the following logical blocks:

- **1.1 Scheduled Trigger & Data Retrieval:** Initiates the workflow daily and reads patient data from a Google Sheets document.
- **1.2 Data Filtering & Processing:** Filters for active patients and processes their data to generate detailed condition reports, including vital signs and medication details, and assesses condition severity.
- **1.3 Notification Distribution:** Sends personalized email and WhatsApp messages to assigned doctors and triggers critical alerts for patients flagged as critical.
- **1.4 Logging:** Logs all generated reports into a separate Google Sheets sheet for record-keeping.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Data Retrieval

- **Overview:** This block sets the daily execution time and retrieves raw patient data from Google Sheets.
- **Nodes Involved:**  
  - Daily Trigger (8 AM)  
  - Read Patient Data

- **Node Details:**

  - **Daily Trigger (8 AM)**  
    - *Type & Role:* Cron trigger node; schedules the workflow to run at a fixed time daily.  
    - *Configuration:* Default cron set for 8 AM daily (no custom parameters detailed).  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Triggers "Read Patient Data".  
    - *Edge Cases:* Misconfiguration can cause missed runs. Timezone issues may cause unexpected trigger times.

  - **Read Patient Data**  
    - *Type & Role:* Google Sheets node; reads patient data from the "Patients" sheet.  
    - *Configuration:* Reads from the first sheet (`gid=0`) of a Google Sheets document identified by `YOUR_GOOGLE_SHEET_ID`. Uses service account authentication.  
    - *Inputs:* Trigger output from "Daily Trigger (8 AM)".  
    - *Outputs:* Patient data rows to "Filter Active Patients".  
    - *Edge Cases:* Authentication failures, incorrect document ID, empty or malformed sheets, API rate limits.

#### 1.2 Data Filtering & Processing

- **Overview:** Filters for patients marked as "Active" and processes each patient's data to create detailed reports, including condition status and message content for email and WhatsApp.
- **Nodes Involved:**  
  - Filter Active Patients  
  - Process Patient Data

- **Node Details:**

  - **Filter Active Patients**  
    - *Type & Role:* Filter node; selects only patients with `Status` field set to "Active".  
    - *Configuration:* Condition: `$json.Status == "Active"` (case sensitive, strict type).  
    - *Inputs:* Patient data rows from "Read Patient Data".  
    - *Outputs:* Only active patient records proceed to processing.  
    - *Edge Cases:* Misspelled or missing Status fields cause filtering errors or data omission.

  - **Process Patient Data**  
    - *Type & Role:* Code node (JavaScript); transforms patient data into structured report objects, assesses condition severity, and generates email and WhatsApp message content.  
    - *Configuration Highlights:*  
      - Iterates over all active patients.  
      - Extracts key data: Patient ID, Name, Age, Current Condition, Vitals (Temperature, BP, Heart Rate), Medication, Assigned Doctor contacts, Priority, Last Update date.  
      - Determines condition status: defaults to "Stable", changes to "Attention Required" if temperature > 100.4°F, or "Critical" if priority is "Critical".  
      - Generates email subject and body with detailed patient info and condition summary.  
      - Creates formatted WhatsApp message text with key patient details.  
    - *Key Expressions:* Uses JavaScript date formatting, conditional logic on vitals and priority, template strings for messages.  
    - *Inputs:* Filtered active patient data.  
    - *Outputs:* Array of detailed report objects to multiple downstream nodes.  
    - *Edge Cases:* Missing vital signs defaulted to "N/A", malformed numeric values, timezone differences affecting date, large datasets might cause performance issues.

#### 1.3 Notification Distribution

- **Overview:** Sends email and WhatsApp notifications to assigned doctors, filters for critical patients to send urgent alerts to hospital staff.
- **Nodes Involved:**  
  - Send Email Report  
  - Send WhatsApp Message  
  - Filter Critical Patients  
  - Send Critical Alert

- **Node Details:**

  - **Send Email Report**  
    - *Type & Role:* Email Send node; sends patient report emails to the assigned doctor.  
    - *Configuration:*  
      - Subject: derived from report data (`Daily Patient Report - [Name] ([ID]) - [ConditionStatus]`).  
      - To: doctor's email from patient data.  
      - From: static sender email `your-email@gmail.com`.  
      - Credentials: SMTP configured (named "SMTP -test").  
    - *Inputs:* Processed patient reports.  
    - *Outputs:* None downstream.  
    - *Edge Cases:* SMTP authentication errors, invalid recipient emails, email content formatting issues.

  - **Send WhatsApp Message**  
    - *Type & Role:* HTTP Request node; sends WhatsApp messages via an external API endpoint.  
    - *Configuration:*  
      - POST request to `https://api.whatsapp.com/send` with JSON body including phone number and message.  
      - Headers include `Content-Type: application/json` and bearer token authorization (`YOUR_WHATSAPP_TOKEN`).  
      - Phone and message values taken from processed report data.  
    - *Inputs:* Processed patient reports.  
    - *Outputs:* None downstream.  
    - *Edge Cases:* API token expiry, invalid phone numbers, message formatting issues, HTTP timeouts or failures.

  - **Filter Critical Patients**  
    - *Type & Role:* Filter node; selects reports with `conditionStatus` equal to "Critical".  
    - *Configuration:* Condition: `$json.conditionStatus == "Critical"`.  
    - *Inputs:* Processed patient reports.  
    - *Outputs:* Only critical patient reports forwarded to "Send Critical Alert".  
    - *Edge Cases:* Missing or malformed conditionStatus field.

  - **Send Critical Alert**  
    - *Type & Role:* Email Send node; sends urgent alert emails to hospital emergency staff for critical patients.  
    - *Configuration:*  
      - Subject includes "🚨 CRITICAL PATIENT ALERT - [PatientName]".  
      - To: static emails `head.doctor@hospital.com,emergency@hospital.com`.  
      - From: `your-email@gmail.com`.  
      - Credentials: same SMTP as above.  
    - *Inputs:* Critical patient reports from filter.  
    - *Outputs:* None downstream.  
    - *Edge Cases:* Same as general email node plus risk of missing critical alerts if filter fails.

#### 1.4 Logging

- **Overview:** Logs all generated patient reports into a separate Google Sheets sheet for audit and tracking purposes.
- **Nodes Involved:**  
  - Log Report to Sheet

- **Node Details:**

  - **Log Report to Sheet**  
    - *Type & Role:* Google Sheets node; appends patient report data to the "Reports_Log" sheet (`gid=1`) in the same Google Sheets document.  
    - *Configuration:* Uses service account authentication; writes to a dedicated sheet for report logs.  
    - *Inputs:* Processed patient reports.  
    - *Outputs:* None downstream.  
    - *Edge Cases:* API quota limits, incorrect sheet ID or permissions, data format mismatches.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                              | Input Node(s)             | Output Node(s)                              | Sticky Note                                                                                  |
|------------------------|----------------------|----------------------------------------------|---------------------------|---------------------------------------------|----------------------------------------------------------------------------------------------|
| Daily Trigger (8 AM)    | Cron Trigger         | Starts workflow daily at 8 AM                 | -                         | Read Patient Data                           | ## How It Works<br>- **Cron Trigger** - Schedules daily run at 8 AM.                         |
| Read Patient Data      | Google Sheets        | Reads patient data from "Patients" sheet     | Daily Trigger (8 AM)       | Filter Active Patients                       | - **Google Sheets (Read)** - Fetches patient data from "Patients" sheet.                     |
| Filter Active Patients  | Filter               | Filters patients with status "Active"         | Read Patient Data          | Process Patient Data                         | - **Filter Node** - Selects active patients.                                                |
| Process Patient Data    | Code (JavaScript)    | Generates reports and messages per patient    | Filter Active Patients     | Send Email Report, Send WhatsApp Message, Filter Critical Patients, Log Report to Sheet | - **Code Node** - Processes data, creates report content, checks for critical conditions.    |
| Send Email Report       | Email Send           | Sends report emails to assigned doctors       | Process Patient Data       | -                                           | - **Email Send Node** - Sends reports to doctors via Gmail.                                 |
| Send WhatsApp Message   | HTTP Request         | Sends WhatsApp messages to assigned doctors   | Process Patient Data       | -                                           | - **HTTP Request Node** - Sends WhatsApp messages.                                          |
| Filter Critical Patients| Filter               | Selects critical patient reports               | Process Patient Data       | Send Critical Alert                          | - **Filter Critical** - Identifies critical patients.                                       |
| Send Critical Alert     | Email Send           | Sends urgent alerts for critical patients     | Filter Critical Patients   | -                                           | - **Critical Alert Email** - Notifies hospital staff.                                       |
| Log Report to Sheet     | Google Sheets        | Logs all reports to "Reports_Log" sheet       | Process Patient Data       | -                                           | - **Google Sheets (Write)** - Logs reports to "Reports_Log" sheet.                          |
| Sticky Note            | Sticky Note          | Provides overview and explanation               | -                         | -                                           | See all above sticky notes combined for comprehensive workflow summary.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node**  
   - Name: `Daily Trigger (8 AM)`  
   - Type: Cron Trigger  
   - Configure to run once every day at 8:00 AM local time.  

2. **Create a Google Sheets node to read patient data**  
   - Name: `Read Patient Data`  
   - Type: Google Sheets (Read)  
   - Set Document ID to your Google Sheet containing patient data.  
   - Select the first sheet (`gid=0`) or sheet named "Patients".  
   - Use Service Account credentials with read permissions.  
   - Connect output of `Daily Trigger (8 AM)` to this node.  

3. **Add a Filter node to select active patients**  
   - Name: `Filter Active Patients`  
   - Condition: `$json.Status == "Active"` (string equals, case sensitive)  
   - Connect output of `Read Patient Data` to this node.  

4. **Add a Code node to process patient data**  
   - Name: `Process Patient Data`  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript code that:  
     - Iterates over all active patients.  
     - Extracts patient info, vitals, medication, assigned doctor contacts.  
     - Determines condition status based on temperature and priority.  
     - Builds email subject, email body, and WhatsApp message text.  
   - Connect output of `Filter Active Patients` to this node.  

5. **Create an Email Send node for report emails**  
   - Name: `Send Email Report`  
   - Use SMTP credentials configured for your email service.  
   - From email: your verified sender address (e.g., `your-email@gmail.com`).  
   - To email: use expression `{{$json.doctorEmail}}`.  
   - Subject: `{{$json.emailSubject}}`  
   - Body: use the generated email body from code node (`$json.emailBody`).  
   - Connect output of `Process Patient Data` to this node.  

6. **Create an HTTP Request node to send WhatsApp messages**  
   - Name: `Send WhatsApp Message`  
   - Method: POST  
   - URL: `https://api.whatsapp.com/send` (or your WhatsApp API endpoint)  
   - Headers: `Content-Type: application/json`, `Authorization: Bearer YOUR_WHATSAPP_TOKEN`  
   - Body (JSON): Include `phone` set to `{{$json.doctorWhatsApp}}` and `message` set to `{{$json.whatsappMessage}}`.  
   - Connect output of `Process Patient Data` to this node.  

7. **Add a Filter node to identify critical patients**  
   - Name: `Filter Critical Patients`  
   - Condition: `$json.conditionStatus == "Critical"` (string equals)  
   - Connect output of `Process Patient Data` to this node.  

8. **Create an Email Send node for critical alerts**  
   - Name: `Send Critical Alert`  
   - Use same SMTP credentials as above.  
   - From email: same as above.  
   - To email: static list `head.doctor@hospital.com,emergency@hospital.com`.  
   - Subject: `🚨 CRITICAL PATIENT ALERT - {{$json.patientName}}`  
   - Body: customize as needed or reuse code node data.  
   - Connect output of `Filter Critical Patients` to this node.  

9. **Create Google Sheets node to log reports**  
   - Name: `Log Report to Sheet`  
   - Document ID: same as patient data sheet.  
   - Sheet: second sheet (`gid=1`), named "Reports_Log".  
   - Use service account credentials with write permissions.  
   - Connect output of `Process Patient Data` to this node.  

10. **Validate all connections:**  
    - `Daily Trigger (8 AM)` → `Read Patient Data` → `Filter Active Patients` → `Process Patient Data`  
    - `Process Patient Data` → `Send Email Report`  
    - `Process Patient Data` → `Send WhatsApp Message`  
    - `Process Patient Data` → `Filter Critical Patients` → `Send Critical Alert`  
    - `Process Patient Data` → `Log Report to Sheet`  

11. **Test the workflow manually and verify:**  
    - Trigger the workflow and confirm data retrieval from Google Sheets.  
    - Validate output of code node for correct report formatting.  
    - Confirm emails and WhatsApp messages are sent correctly.  
    - Confirm critical alerts are sent for critical patients only.  
    - Ensure logs are appended properly to the "Reports_Log" sheet.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow sends daily patient reports at 8 AM using Google Sheets as data source and notifies doctors via email and WhatsApp. | Workflow purpose and overview.                                                                                |
| Ensure Google Sheets API credentials (Service Account) have proper read/write permissions on the target spreadsheet.           | Credential setup for Google Sheets integration.                                                              |
| SMTP credentials must allow sending emails from the configured sender address (`your-email@gmail.com`).                         | Email sending setup.                                                                                          |
| WhatsApp API requires a valid token and correct API endpoint; adjust URL and headers if using a different provider.            | WhatsApp messaging configuration.                                                                             |
| Patient data sheet should include columns: `Patient ID`, `Patient Name`, `Age`, `Current Condition`, `Temperature`, etc.       | Data schema requirements for proper processing.                                                              |
| Date and time in reports use US locale formatting; adjust in code node if needed.                                               | Localization details in code node.                                                                            |
| Critical patients trigger additional alert emails to hospital emergency contacts.                                              | Workflow escalation logic.                                                                                     |
| Sticky Note in workflow summarizes key steps and node roles for reference.                                                     | In-workflow documentation for users and maintainers.                                                        |

---

**Disclaimer:** The above documentation is based solely on an n8n workflow designed for lawful and ethical healthcare communication automation. It respects all content policies and handles only public and legal data.  

---