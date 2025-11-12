Website Lead Management: Send Contact Form Submissions to WhatsApp & Google Sheets

https://n8nworkflows.xyz/workflows/website-lead-management--send-contact-form-submissions-to-whatsapp---google-sheets-7424


# Website Lead Management: Send Contact Form Submissions to WhatsApp & Google Sheets

### 1. Workflow Overview

This workflow is designed for **Website Lead Management**, automating the processing of contact form submissions by sending instant alerts to WhatsApp and logging the data into Google Sheets for record-keeping and analysis. It targets businesses that want to capture website inquiries efficiently and maintain organized lead data.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives incoming POST requests from a website contact form.
- **1.2 Data Formatting:** Cleans and structures the incoming lead data, preparing it for messaging and storage.
- **1.3 Notification Dispatch:** Sends a formatted lead notification to WhatsApp using the WhatsApp Business API.
- **1.4 Data Logging:** Logs the lead details into a Google Sheets spreadsheet for archival and tracking.
- **1.5 Documentation:** Provides an embedded sticky note with detailed instructions, setup notes, and workflow explanation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
This block receives HTTP POST requests from the website’s contact form containing lead information.

- **Nodes Involved:**  
`Contact Form Trigger`

- **Node Details:**

  - **Contact Form Trigger**  
    - **Type:** Webhook Trigger  
    - **Role:** Entry point of the workflow, listening for POST submissions at endpoint `/get_data`.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Path: `get_data`  
      - No additional options configured.  
    - **Expressions/Variables:** Receives JSON payload typically containing fields: name, email, phone, service, message.  
    - **Connections:** Outputs to `Format Lead Data`.  
    - **Edge Cases:**  
      - Missing or malformed payload can cause downstream errors if not handled.  
      - Unauthorized or unexpected POST requests are not explicitly filtered here.  
    - **Version:** n8n webhook node v2.1

#### 1.2 Data Formatting

- **Overview:**  
Processes the incoming raw form data to ensure it is clean, non-empty (defaults to "N/A"), and formats a structured message enriched with emojis and an Indian timezone timestamp for WhatsApp notification and Google Sheets logging.

- **Nodes Involved:**  
`Format Lead Data`

- **Node Details:**

  - **Format Lead Data**  
    - **Type:** Code (JavaScript)  
    - **Role:** Data cleaning and formatting.  
    - **Configuration:**  
      - Custom JavaScript code processes all incoming items.  
      - Extracts fields: name, email, phone, service, message.  
      - Trims whitespace, replaces empty fields with `"N/A"`.  
      - Generates a timestamp in `Asia/Kolkata` timezone.  
      - Constructs a formatted text message with emojis and line separators for WhatsApp.  
    - **Key Expressions:**  
      - Uses `$input.all()` to process all incoming items.  
      - Local timezone formatting for timestamp.  
      - Template literal for WhatsApp message with emojis (e.g., 📩, 👤).  
    - **Connections:** Outputs to both `WhatsApp Alert` and `Log to Database`.  
    - **Edge Cases:**  
      - If input data structure changes (e.g., missing `body`), may fallback to empty JSON object.  
      - Date formatting depends on server timezone and `toLocaleString` support.  
    - **Version:** Code node v2

#### 1.3 Notification Dispatch

- **Overview:**  
Sends the formatted lead data as an immediate WhatsApp message alert to a specified mobile number using WhatsApp Business API credentials.

- **Nodes Involved:**  
`WhatsApp Alert`

- **Node Details:**

  - **WhatsApp Alert**  
    - **Type:** WhatsApp node (n8n-nodes-base.whatsApp)  
    - **Role:** Sends a WhatsApp message containing lead details.  
    - **Configuration:**  
      - Operation: send  
      - Recipient Phone Number: placeholder `YOUR_MOBILE_NUMBER` (to be updated)  
      - Phone Number ID: placeholder `YOUR_PHONE_NUMBER_ID` (from WhatsApp Business API)  
      - Message body: expression binding to `{{$json.formattedText}}` from `Format Lead Data`.  
    - **Credentials:** Requires WhatsApp Business API credentials configured in n8n.  
    - **Connections:** No outputs (end of this branch).  
    - **Edge Cases:**  
      - Authentication failure or invalid credentials.  
      - API rate limits or connectivity issues.  
      - Incorrect phone number format or unregistered recipient number.  
    - **Version:** WhatsApp node v1

#### 1.4 Data Logging

- **Overview:**  
Appends or updates a Google Sheets document with the cleaned lead data for permanent record and follow-up.

- **Nodes Involved:**  
`Log to Database`

- **Node Details:**

  - **Log to Database**  
    - **Type:** Google Sheets node  
    - **Role:** Append or update lead data rows in Google Sheets.  
    - **Configuration:**  
      - Operation: appendOrUpdate  
      - Sheet Name: `Sheet1` (gid=0)  
      - Document ID: placeholder `YOUR_SPREADSHEET_ID`  
      - Defines columns: date, name, email, phone, service, message  
      - Mapping mode is explicit definition of fields from JSON.  
      - Matching column for update: date (though likely each submission is unique)  
    - **Credentials:** Requires Google Sheets OAuth2 credentials.  
    - **Connections:** No outputs (end of this branch).  
    - **Edge Cases:**  
      - Invalid or revoked Google credentials.  
      - Spreadsheet ID or sheet name mismatch.  
      - API rate limits or network issues.  
      - Data type mismatches or empty fields are handled by prior node.  
    - **Version:** Google Sheets node v4.7

#### 1.5 Documentation / Notes

- **Overview:**  
Provides embedded documentation for users to understand and configure the workflow.

- **Nodes Involved:**  
`Sticky Note`

- **Node Details:**

  - **Sticky Note**  
    - **Type:** Sticky Note  
    - **Role:** Contains detailed notes on the workflow purpose, setup instructions, node roles, and placeholders to be replaced.  
    - **Configuration:**  
      - Large note block explaining each node, setup requirements for Google Sheets and WhatsApp API, webhook URL usage, and required fields.  
    - **Connections:** None (informational).  
    - **Edge Cases:** None.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                           | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                                   |
|---------------------|---------------------|-----------------------------------------|-------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Contact Form Trigger | Webhook             | Receives POST requests from contact form |                         | Format Lead Data             | ## 🎯 Website Lead Management: Send Contact Form Submissions to WhatsApp & Google Sheets (full content in Sticky Note)         |
| Format Lead Data     | Code                | Cleans and formats lead data             | Contact Form Trigger     | WhatsApp Alert, Log to Database | See Sticky Note for detailed purpose and formatting instructions                                                                |
| WhatsApp Alert       | WhatsApp            | Sends WhatsApp message alert              | Format Lead Data         |                             | Phone and Phone Number ID placeholders must be updated per setup instructions                                                  |
| Log to Database      | Google Sheets       | Logs lead data into Google Sheets         | Format Lead Data         |                             | Requires Google Sheets with defined columns; update Spreadsheet ID placeholder                                                 |
| Sticky Note          | Sticky Note         | Embedded documentation and setup guidance |                         |                             | Contains full workflow explanation, setup requirements, and placeholders to replace with actual credentials and IDs           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Configure:  
       - HTTP Method: POST  
       - Path: `get_data`  
   - Purpose: To receive contact form submissions as JSON POST requests.

2. **Add Code Node to Format Lead Data**  
   - Type: Code  
   - Paste the following JavaScript logic:  
     ```javascript
     const items = $input.all().map(item => {
       const data = item.json.body || item.json || {};
       const clean = val => (val || "").toString().trim() || "N/A";
       const name = clean(data.name);
       const email = clean(data.email);
       const phone = clean(data.phone);
       const service = clean(data.service);
       const message = clean(data.message);
       const receivedAt = new Date().toLocaleString("en-IN", { timeZone: "Asia/Kolkata" });
       const formattedText =
         `📩 *New Contact Form Submission*\n` +
         `━━━━━━━━━━━━━━\n` +
         `👤 *Name:* ${name}\n` +
         `📧 *Email:* ${email}\n` +
         `📱 *Phone:* ${phone}\n` +
         `🛠 *Service:* ${service}\n` +
         `💬 *Message:* ${message}\n` +
         `🕒 *Received At:* ${receivedAt}`;
       return {
         json: {
           name,
           email,
           phone,
           service,
           message,
           receivedAt,
           formattedText
         }
       };
     });
     return items;
     ```
   - Connect input from the webhook node.

3. **Add WhatsApp Node to Send Alert**  
   - Type: WhatsApp  
   - Configure:  
       - Operation: send  
       - Recipient Phone Number: Replace with your mobile number including country code (e.g., +919876543210)  
       - Phone Number ID: Replace with your WhatsApp Business API phone number ID  
       - Text Body: Use expression `{{$json.formattedText}}`  
   - Set WhatsApp Business API credentials in n8n credentials manager.  
   - Connect input from the Code node.

4. **Add Google Sheets Node to Log Lead Data**  
   - Type: Google Sheets  
   - Configure:  
       - Operation: appendOrUpdate  
       - Sheet Name: `Sheet1` (or your actual sheet name, ensure sheet exists)  
       - Document ID: Replace with your Google Sheet ID  
       - Columns: Map fields as follows:  
         - date: `{{$json.receivedAt}}`  
         - name: `{{$json.name}}`  
         - email: `{{$json.email}}`  
         - phone: `{{$json.phone}}`  
         - service: `{{$json.service}}`  
         - message: `{{$json.message}}`  
       - Matching Columns: `date` (optional based on update strategy)  
   - Set Google Sheets OAuth2 credentials.  
   - Connect input from the Code node.

5. **Add Sticky Note for Documentation** (Optional but recommended)  
   - Type: Sticky Note  
   - Add detailed notes covering:  
     - Workflow purpose  
     - Node roles  
     - Setup instructions (Google Sheets columns, WhatsApp API credentials, webhook URL usage)  
     - Placeholder reminders for phone numbers and spreadsheet IDs

6. **Connect Nodes**  
   - Webhook → Code  
   - Code → WhatsApp  
   - Code → Google Sheets

7. **Test Workflow**  
   - Deploy webhook URL, update your website contact form to POST data to this URL.  
   - Confirm WhatsApp alert receipt.  
   - Verify Google Sheets logging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| WhatsApp Business API requires registration and phone number setup. Ensure you obtain `Phone Number ID` from your WhatsApp API dashboard.                              | WhatsApp Business API documentation                              |
| Google Sheets must have a sheet named `Sheet1` (or update node accordingly) with columns: date, name, email, phone, service, message.                                   | Google Sheets setup                                              |
| Webhook URL generated by n8n must be copied and added as the form action URL in your website's contact form HTML.                                                     | Webhook URL setup                                               |
| The timestamp uses Indian Standard Time (Asia/Kolkata). Adjust code if a different timezone is needed.                                                                 | Javascript Date localization                                     |
| For production use, consider adding validation, error handling, and security measures (e.g., webhook secrets or authentication).                                        | Security best practices                                         |

---

This detailed reference enables advanced users and AI agents to understand, recreate, and maintain the workflow with clarity, ensuring robust lead management automation integrating website forms, WhatsApp notifications, and Google Sheets logging.

---

**Disclaimer:**  
The provided text is derived exclusively from an automated n8n workflow. It complies strictly with applicable content policies, contains no illegal or offensive material, and processes only legal and public data.