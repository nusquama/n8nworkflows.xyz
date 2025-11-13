Invoice Creator with Google Sheets & Automated Email Payment Reminder System

https://n8nworkflows.xyz/workflows/invoice-creator-with-google-sheets---automated-email-payment-reminder-system-6744


# Invoice Creator with Google Sheets & Automated Email Payment Reminder System

### 1. Workflow Overview

This workflow automates the creation of monthly client invoices and manages payment reminders for overdue invoices using Google Sheets as the data source and SMTP for email communication. It consists of two main logical blocks:

- **1.1 Invoice Creation Flow:**  
  Triggered monthly, it fetches active clients from a Google Sheet, generates invoice data per client, saves invoices back to Google Sheets, sends invoices via email, and logs the activity.

- **1.2 Payment Reminder Flow:**  
  Triggered daily, it retrieves overdue invoices, filters unpaid ones, calculates the appropriate reminder type based on how overdue the invoice is, sends tailored reminder emails (gentle, follow-up, urgent, or final notice), and updates the reminder log in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Invoice Creation Flow

**Overview:**  
This block handles the generation and dispatch of monthly invoices for active clients. It ensures only clients with active status and appropriate billing dates are invoiced, creates invoice records with unique numbers, stores them, sends emails, and logs the process.

**Nodes Involved:**  
- Monthly Invoice Trigger  
- Get Clients for Invoicing  
- Filter Active Clients  
- Generate Invoice Data  
- Save Invoice to Google Sheets  
- Send Invoice Email  
- Log Invoice Creation  

**Node Details:**

- **Monthly Invoice Trigger**  
  - Type: Cron Trigger  
  - Role: Initiates the invoice creation workflow on a monthly schedule (configuration details unspecified; user must set schedule).  
  - Inputs: None  
  - Outputs: Triggers "Get Clients for Invoicing"  
  - Failure modes: Cron misconfiguration, workflow not activated.  

- **Get Clients for Invoicing**  
  - Type: Google Sheets  
  - Role: Reads client data from the "Clients" sheet in a specified Google Sheets document using a Service Account authentication.  
  - Key Config: Reads entire sheet; expects columns including client_id (A), client_name (B), email (C), service_description (D), billing_date (E), status (F), amount (G).  
  - Inputs: Trigger from Monthly Invoice Trigger  
  - Outputs: All rows to "Filter Active Clients"  
  - Failure modes: Authentication errors, document ID or sheet name misconfiguration, API quota limits.  

- **Filter Active Clients**  
  - Type: Code (JavaScript)  
  - Role: Filters clients to only those with status = 'active' and billing_date on or before today, ignoring header rows and empty IDs. Converts and formats data for downstream nodes.  
  - Key Expressions: Uses JavaScript date comparisons and array filtering.  
  - Inputs: All client rows from Google Sheets node  
  - Outputs: Filtered client list to "Generate Invoice Data"  
  - Failure modes: Date parsing errors, empty or malformed client data, empty output if no clients match.  

- **Generate Invoice Data**  
  - Type: Code (JavaScript)  
  - Role: Creates a unique invoice number (format INV-YYYY-MM-DD-RND), calculates due date 30 days ahead, and assembles invoice data object including client info and status 'pending'.  
  - Key Expressions: Date and string formatting; random number for invoice uniqueness.  
  - Inputs: Single client JSON from filtered clients  
  - Outputs: Invoice JSON to "Save Invoice to Google Sheets"  
  - Failure modes: Date/time inconsistencies, random number collisions (very low probability).  

- **Save Invoice to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends or updates the "Invoices" sheet with generated invoice data using service account credentials.  
  - Key Config: Uses "appendOrUpdate" operation; expects invoice data columns matching generated JSON.  
  - Inputs: Invoice JSON from previous node  
  - Outputs: Confirmation to "Send Invoice Email"  
  - Failure modes: API errors, column mapping issues, authentication failure.  

- **Send Invoice Email**  
  - Type: Email Send (SMTP)  
  - Role: Sends the generated invoice via email to the client. Email subject includes invoice number; from address is billing@yourcompany.com.  
  - Key Config: Uses SMTP credentials; email body text is bound to a variable $json.result (not defined in code—likely a placeholder to be replaced or adjusted).  
  - Inputs: Confirmation of saved invoice  
  - Outputs: Success to "Log Invoice Creation"  
  - Failure modes: SMTP authentication failure, invalid email addresses, email sending errors.  

- **Log Invoice Creation**  
  - Type: Google Sheets  
  - Role: Logs invoice creation activity in an "Activity_Log" sheet for auditing purposes.  
  - Key Config: AppendOrUpdate operation; expects relevant fields to track invoice generation.  
  - Inputs: Successful email send event  
  - Outputs: None (end of flow)  
  - Failure modes: API errors, logging sheet misconfiguration.  

---

#### 2.2 Payment Reminder Flow

**Overview:**  
This block automatically checks for overdue invoices daily, filters those still unpaid, determines the severity of the reminder based on how overdue the invoice is, sends the appropriate email reminder, and updates the invoice record with reminder details.

**Nodes Involved:**  
- Daily Payment Reminder Check  
- Get Overdue Invoices  
- Filter Overdue Invoices  
- Calculate Reminder Type  
- Switch Reminder Type  
- Send Gentle Reminder  
- Send Follow-up Reminder  
- Send Urgent Reminder  
- Send Final Notice  
- Update Reminder Log  

**Node Details:**

- **Daily Payment Reminder Check**  
  - Type: Cron Trigger  
  - Role: Runs the payment reminder workflow daily (user must configure schedule).  
  - Inputs: None  
  - Outputs: Triggers "Get Overdue Invoices"  
  - Failure modes: Cron misconfiguration, workflow inactivity.  

- **Get Overdue Invoices**  
  - Type: Google Sheets  
  - Role: Reads all invoice records from "Invoices" sheet via service account authentication.  
  - Inputs: Trigger from daily cron  
  - Outputs: All invoice rows to "Filter Overdue Invoices"  
  - Failure modes: Authentication, document or sheet misconfiguration, API limits.  

- **Filter Overdue Invoices**  
  - Type: Code (JavaScript)  
  - Role: Filters invoices with status 'pending' and due date on or before yesterday (i.e., overdue at least one day), ignoring header and empty rows. Transforms data for downstream processing.  
  - Key Expressions: Date comparisons, filtering logic.  
  - Inputs: All invoices from Google Sheets  
  - Outputs: Filtered overdue invoices to "Calculate Reminder Type"  
  - Failure modes: Date parsing errors, empty or malformed data, no overdue invoices found.  

- **Calculate Reminder Type**  
  - Type: Code (JavaScript)  
  - Role: Calculates days overdue and assigns a reminder type with corresponding email subject:  
    - 0-6 days: gentle  
    - 7-13 days: follow-up  
    - 14-29 days: urgent  
    - 30+ days: final notice  
  - Inputs: Filtered overdue invoices  
  - Outputs: Invoice data augmented with days_overdue, reminder_type, and email_subject to "Switch Reminder Type"  
  - Failure modes: Date math errors, empty inputs, logic errors (note: there is a minor typo with an extraneous '-' in code that may cause failure).  

- **Switch Reminder Type**  
  - Type: Switch  
  - Role: Routes invoices to one of four email sending nodes based on reminder_type field.  
  - Inputs: Calculated invoice data  
  - Outputs: To one of "Send Gentle Reminder", "Send Follow-up Reminder", "Send Urgent Reminder", or "Send Final Notice"  
  - Failure modes: Missing or invalid reminder_type causing no branch selection.  

- **Send Gentle Reminder / Send Follow-up Reminder / Send Urgent Reminder / Send Final Notice**  
  - Type: Email Send (SMTP)  
  - Role: Sends the respective payment reminder email using SMTP credentials. Email subject includes dynamic email_subject and invoice_id. From address is billing@yourcompany.com.  
  - Inputs: Routed invoice data from Switch node  
  - Outputs: Success to "Update Reminder Log"  
  - Failure modes: SMTP authentication failure, invalid email, sending errors.  

- **Update Reminder Log**  
  - Type: Google Sheets  
  - Role: Updates the invoice record in the "Invoices" sheet to reflect the latest reminder sent, likely updating fields such as last_reminder_sent and reminder_count.  
  - Inputs: Successful email send  
  - Outputs: None (end of flow)  
  - Failure modes: API errors, update conflicts, authentication issues.  

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                                       | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                                      |
|---------------------------|------------------------|------------------------------------------------------|-----------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Monthly Invoice Trigger    | Cron                   | Triggers monthly invoice creation                     | None                        | Get Clients for Invoicing         | Invoice Creation Flow: Triggers workflow on a set monthly schedule.                             |
| Get Clients for Invoicing | Google Sheets          | Reads client data from "Clients" sheet                | Monthly Invoice Trigger      | Filter Active Clients             | Reads client data from Google Sheets.                                                           |
| Filter Active Clients      | Code (JavaScript)      | Filters active clients with billing date <= today     | Get Clients for Invoicing    | Generate Invoice Data             | Filters out inactive clients.                                                                   |
| Generate Invoice Data      | Code (JavaScript)      | Creates invoice number, due date, and invoice object  | Filter Active Clients        | Save Invoice to Google Sheets    | Creates invoice details in required format.                                                    |
| Save Invoice to Google Sheets | Google Sheets          | Appends or updates invoice record in "Invoices" sheet | Generate Invoice Data        | Send Invoice Email               | Appends or updates invoice record in the sheet.                                               |
| Send Invoice Email         | Email Send (SMTP)      | Sends invoice email to client                          | Save Invoice to Google Sheets| Log Invoice Creation             | Sends the invoice to the client via email.                                                    |
| Log Invoice Creation       | Google Sheets          | Logs invoice creation for auditing                     | Send Invoice Email           | None                            | Logs invoice creation for records/auditing.                                                   |
| Daily Payment Reminder Check | Cron                   | Triggers daily payment reminder workflow               | None                        | Get Overdue Invoices             | Reminder Flow: Triggers workflow daily to check overdue invoices.                              |
| Get Overdue Invoices       | Google Sheets          | Reads invoices from "Invoices" sheet                   | Daily Payment Reminder Check | Filter Overdue Invoices          | Reads overdue invoices from Google Sheets.                                                    |
| Filter Overdue Invoices    | Code (JavaScript)      | Filters unpaid invoices overdue as of yesterday       | Get Overdue Invoices         | Calculate Reminder Type          | Filters invoices still unpaid.                                                                 |
| Calculate Reminder Type    | Code (JavaScript)      | Calculates days overdue and assigns reminder type     | Filter Overdue Invoices      | Switch Reminder Type             | Calculates how many days overdue.                                                             |
| Switch Reminder Type       | Switch                 | Routes invoices to appropriate reminder email nodes   | Calculate Reminder Type      | Send Gentle/FU/Urgent/Final      | Decides which type of reminder to send.                                                       |
| Send Gentle Reminder       | Email Send (SMTP)      | Sends gentle payment reminder email                    | Switch Reminder Type         | Update Reminder Log              | Sends respective reminder email.                                                              |
| Send Follow-up Reminder    | Email Send (SMTP)      | Sends follow-up payment reminder email                 | Switch Reminder Type         | Update Reminder Log              | Sends respective reminder email.                                                              |
| Send Urgent Reminder       | Email Send (SMTP)      | Sends urgent payment reminder email                    | Switch Reminder Type         | Update Reminder Log              | Sends respective reminder email.                                                              |
| Send Final Notice          | Email Send (SMTP)      | Sends final notice payment reminder email              | Switch Reminder Type         | Update Reminder Log              | Sends respective reminder email.                                                              |
| Update Reminder Log        | Google Sheets          | Updates invoice record with latest reminder info       | Send Gentle/Follow-up/Urgent/Final | None                      | Updates reminder status in the sheet.                                                         |
| Sticky Note               | Sticky Note            | Describes Invoice Creation Flow                         | None                        | None                            | ## Invoice Creation Flow ... (detailed explanation)                                           |
| Sticky Note1              | Sticky Note            | Describes Reminder Flow                                 | None                        | None                            | ## Reminder Flow ... (detailed explanation)                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node "Monthly Invoice Trigger"**  
   - Type: Cron  
   - Configure to trigger monthly on desired day/time.  

2. **Create Google Sheets Node "Get Clients for Invoicing"**  
   - Operation: Read rows  
   - Google Sheets Document ID: Set your Google Sheet ID  
   - Sheet Name: "Clients"  
   - Authentication: Service Account credentials (configure with Google API credentials)  
   - Connect output of "Monthly Invoice Trigger" to this node.  

3. **Create Code Node "Filter Active Clients"**  
   - Paste JavaScript code to filter clients with `status === 'active'` and `billing_date <= today`.  
   - Input: connects from "Get Clients for Invoicing".  

4. **Create Code Node "Generate Invoice Data"**  
   - Paste JavaScript code to generate invoice number, due date (30 days ahead), and create invoice JSON object.  
   - Input: connects from "Filter Active Clients".  

5. **Create Google Sheets Node "Save Invoice to Google Sheets"**  
   - Operation: Append or Update row  
   - Google Sheets Document ID: same as above  
   - Sheet Name: "Invoices"  
   - Authentication: Service Account credentials  
   - Map columns to invoice data fields (invoice_number, client_id, etc.)  
   - Input: connects from "Generate Invoice Data".  

6. **Create Email Send Node "Send Invoice Email"**  
   - Set SMTP credentials (username, password, server, port)  
   - To: `{{$json.email}}`  
   - From: billing@yourcompany.com  
   - Subject: `New Invoice - {{$json.invoice_number}}`  
   - Body: configure with invoice details or a template (adjust `"text":"={{ $json.result }}"` as needed)  
   - Input: connects from "Save Invoice to Google Sheets".  

7. **Create Google Sheets Node "Log Invoice Creation"**  
   - Operation: Append or Update row  
   - Sheet Name: "Activity_Log"  
   - Authentication: Service Account credentials  
   - Map relevant fields for logging (e.g., invoice_number, date, status)  
   - Input: connects from "Send Invoice Email".  

---

8. **Create Cron Trigger Node "Daily Payment Reminder Check"**  
   - Type: Cron  
   - Configure to trigger daily at preferred time.  

9. **Create Google Sheets Node "Get Overdue Invoices"**  
   - Operation: Read rows  
   - Google Sheets Document ID: same as above  
   - Sheet Name: "Invoices"  
   - Authentication: Service Account credentials  
   - Input: connects from "Daily Payment Reminder Check".  

10. **Create Code Node "Filter Overdue Invoices"**  
    - Paste JavaScript code to select invoices with `status === 'pending'` and `due_date <= yesterday`.  
    - Input: connects from "Get Overdue Invoices".  

11. **Create Code Node "Calculate Reminder Type"**  
    - Paste JavaScript code calculating days overdue and setting reminder_type and email_subject accordingly.  
    - Input: connects from "Filter Overdue Invoices".  
    - **Important:** Fix minor typo in code if present (remove extraneous '-') before deployment.  

12. **Create Switch Node "Switch Reminder Type"**  
    - Rules: Based on `{{$json.reminder_type}}` equal to "gentle", "follow-up", "urgent", or "final".  
    - Input: connects from "Calculate Reminder Type".  

13. **Create Email Send Nodes for each reminder type:**  
    - "Send Gentle Reminder"  
    - "Send Follow-up Reminder"  
    - "Send Urgent Reminder"  
    - "Send Final Notice"  
    - Each configured with SMTP credentials, from billing@yourcompany.com, subject: `{{$json.email_subject}} - Invoice {{$json.invoice_id}}`, and to `{{$json.email}}`.  
    - Connect each output of "Switch Reminder Type" to the appropriate email send node.  

14. **Create Google Sheets Node "Update Reminder Log"**  
    - Operation: Update row  
    - Sheet Name: "Invoices"  
    - Authentication: Service Account credentials  
    - Map fields to update reminder status, last reminder sent date, reminder count, etc.  
    - Connect output of all email send nodes to this node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Invoice Creation Flow: Monthly Invoice Trigger → Get Clients for Invoicing → Filter Active Clients → Generate Invoice Data → Save Invoice → Send Email → Log | Sticky Note attached to Invoice Creation nodes             |
| Reminder Flow: Daily Payment Reminder Check → Get Overdue Invoices → Filter Overdue Invoices → Calculate Reminder Type → Switch → Send Reminder Email → Update | Sticky Note attached to Reminder Flow nodes                |
| Use Service Account for Google Sheets authentication to enable automated read/write access without user interaction.                                         | Google Sheets API and OAuth2 best practices                |
| Ensure SMTP credentials are valid and tested before deploying to avoid email delivery failures.                                                               | SMTP setup instructions based on provider                   |
| Invoice number generation uses a date and random number to reduce chance of collision; consider extending logic for high-volume clients.                      | Invoice numbering best practices                            |
| The calculation code for reminder type contains a minor typo (`reminderType = 'gentle';-`), which must be corrected to avoid runtime errors.                   | Code node syntax validation                                 |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.