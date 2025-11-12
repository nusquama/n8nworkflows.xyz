Automated Lead Response Template using Google Forms, Sheets, and Gmail

https://n8nworkflows.xyz/workflows/automated-lead-response-template-using-google-forms--sheets--and-gmail-6711


# Automated Lead Response Template using Google Forms, Sheets, and Gmail

### 1. Workflow Overview

This workflow automates lead response management by integrating Google Forms, Google Sheets, and Gmail. It listens for new form responses recorded in a Google Sheet and performs two main actions:

- Sends a personalized confirmation email to the lead who submitted the form.
- Sends an internal notification email to the sales or support team with the detailed lead information.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by new rows added to a specific Google Sheet linked to a Google Form.
- **1.2 Lead Confirmation Email:** Sends a personalized thank-you email to the lead using their submitted email and form data.
- **1.3 Team Notification Email:** Sends an internal email with the full lead details for follow-up by the team.

This automation is ideal for capturing and responding to leads, contact form submissions, quote requests, or any customer intake process.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block detects new form submissions by monitoring a specific Google Sheet tab for added rows.
- **Nodes Involved:** Google Sheets Trigger

##### Node: Google Sheets Trigger

- **Type:** Trigger node — Google Sheets Trigger
- **Role:** Initiates the workflow on new row addition.
- **Configuration:** 
  - Watches the sheet named "Answers what are you interested in?" (Sheet ID: 1938874403) within a Google Sheets document.
  - Poll interval set to every minute.
  - Document ID specified for the Google Sheet linked to the form responses.
- **Key Expressions/Variables:** None (trigger node).
- **Input/Output Connections:** 
  - No input (trigger node).
  - Output connected to "Send a message" node.
- **Version Requirements:** Compatible with n8n googleSheetsTriggerOAuth2Api credentials.
- **Potential Failures:** 
  - Authentication errors if OAuth2 credentials expire or are invalid.
  - API rate limiting from Google Sheets.
  - Sheet or document ID changes causing trigger failure.
- **Sub-Workflow:** None.

---

#### 2.2 Lead Confirmation Email

- **Overview:** Sends a personalized thank-you email to the lead who submitted the form, confirming receipt of their inquiry.
- **Nodes Involved:** Send a message

##### Node: Send a message

- **Type:** Action node — Gmail (send email)
- **Role:** Sends a confirmation email to the lead.
- **Configuration:** 
  - Sends email to the lead’s email address extracted from the form field `Email`.
  - Message body is HTML formatted, greeting the lead by their `Full Name`.
  - Includes the lead's submitted `Additional Message or Query` quoted in the message.
  - Subject line: "Thank you for contacting us!"
  - CCs a fixed internal email address "your@email.com".
- **Key Expressions/Variables:** 
  - Recipient: `{{$json["Email"]}}`
  - Message body uses `{{$json["Full Name"]}}` and `{{$json["Additional Message or Query"]}}`
- **Input/Output Connections:** 
  - Input from "Google Sheets Trigger"
  - Output to "Send a message1"
- **Version Requirements:** Uses Gmail OAuth2 credentials, version 2.1.
- **Potential Failures:** 
  - Gmail API authentication errors.
  - Invalid or missing lead email address.
  - HTML formatting issues causing email rendering problems.
- **Sub-Workflow:** None.

---

#### 2.3 Team Notification Email

- **Overview:** Sends an internal email notification to the team with all the lead details for processing or follow-up.
- **Nodes Involved:** Send a message1

##### Node: Send a message1

- **Type:** Action node — Gmail (send email)
- **Role:** Notifies internal team about a new lead.
- **Configuration:** 
  - Recipient: Fixed internal email "your@email.com"
  - Message body is HTML formatted and includes:
    - Lead’s full name, email, and phone number (optional)
    - What the lead is interested in
    - Additional message or query content
    - A link to the Google Sheet containing all form responses.
  - Subject line: "📥 New lead from the form"
- **Key Expressions/Variables:** 
  - Uses multiple fields from the trigger data, e.g. `{{$json["Full Name"]}}`, `{{$json["Phone (optional)"]}}`, `{{$json["What are you interested in?"]}}`
- **Input/Output Connections:** 
  - Input from "Send a message"
  - No further output.
- **Version Requirements:** Gmail OAuth2 credentials, version 2.1.
- **Potential Failures:** 
  - Authentication or permission errors with Gmail API.
  - Incorrect fixed email address.
  - Missing or malformed form data fields.
- **Sub-Workflow:** None.

---

#### 2.4 Documentation and Notes (Sticky Notes)

Four sticky notes provide contextual documentation within the workflow:

- **Sticky Note:** Workflow overview and use case explanation.
- **Sticky Note1:** Instructions for the lead confirmation email node.
- **Sticky Note2:** Instructions for the team notification email node.
- **Sticky Note3:** Google Sheet column header requirements and recommendations.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                   | Input Node(s)         | Output Node(s)      | Sticky Note                                                                                                        |
|---------------------|----------------------------|---------------------------------|-----------------------|---------------------|--------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger| Google Sheets Trigger       | Detect new form submissions      | —                     | Send a message       | 📊 Google Sheet Column Requirements<br>Make sure your Google Sheet has these exact column headers:<br>- Timestamp<br>- Full Name<br>- Email<br>- Phone Number (optional)<br>- What are you interested in?<br>- Additional message or query |
| Send a message       | Gmail (Send Email)          | Send confirmation email to lead  | Google Sheets Trigger  | Send a message1      | 📩 Email to Lead<br>This node sends a confirmation message to the person who filled out the form.<br>✅ Connect Gmail account<br>✅ Personalize subject and body<br>✅ Use correct recipient email field (e.g., “Email”)                |
| Send a message1      | Gmail (Send Email)          | Send internal notification email | Send a message         | —                   | 📣 Team Notification Email<br>Sends internal email with lead info.<br>✅ Replace destination email<br>✅ CC multiple addresses if needed<br>✅ Format message as desired                                           |
| Sticky Note          | Sticky Note                 | Workflow overview and use case   | —                     | —                   | 📌 Workflow Overview<br>This automation listens for new Google Form responses and sends confirmation and team notification emails.<br>Use for lead capture, contact forms, quote requests, or intake forms.                  |
| Sticky Note1         | Sticky Note                 | Instructions for Send a message  | —                     | —                   | See "Send a message" sticky note                                                                                     |
| Sticky Note2         | Sticky Note                 | Instructions for Send a message1 | —                     | —                   | See "Send a message1" sticky note                                                                                    |
| Sticky Note3         | Sticky Note                 | Sheet columns requirements       | —                     | —                   | See "Google Sheets Trigger" sticky note                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Google Sheets Trigger**
   - Add node: "Google Sheets Trigger"
   - Set event to "rowAdded" (trigger when a new row is added).
   - Configure polling interval to every 1 minute.
   - Select your Google Sheets OAuth2 credentials.
   - Set Document ID to your Google Sheet containing form responses.
   - Set Sheet Name to the specific tab capturing form answers (e.g., "Answers what are you interested in?").
   - Save the node.

2. **Create Node: Send a message (Lead Confirmation Email)**
   - Add node: "Gmail" (Send Email)
   - Connect input from "Google Sheets Trigger".
   - Select Gmail OAuth2 credentials.
   - Set "Send To" field to `={{$json["Email"]}}`.
   - Subject: "Thank you for contacting us!"
   - Message: Use HTML formatted text:
     ```
     Hello {{$json["Full Name"]}},<br><br>
     Thank you for contacting us. We've received your message:<br>
     <em>{{$json["Additional Message or Query"]}}</em><br><br>
     We'll be in touch soon.<br><br>
     Best regards,<br>
     Your team
     ```
   - Add CC to your internal email address (e.g., your@email.com).
   - Save the node.

3. **Create Node: Send a message1 (Team Notification Email)**
   - Add node: "Gmail" (Send Email)
   - Connect input from "Send a message".
   - Select Gmail OAuth2 credentials.
   - Set "Send To" to your internal team email (e.g., your@email.com).
   - Subject: "📥 New lead from the form"
   - Message body (HTML formatted), example:
     ```
     📬 A new lead was received: <br><br>
     <strong>Name:</strong> {{$json["Full Name"]}} <br>
     <strong>Email:</strong> {{$json["Email"]}} <br>
     <strong>Phone:</strong> {{$json["Phone (optional)"]}} <br>
     <strong>Interest:</strong> {{$json["What are you interested in?"]}} <br>
     <strong>Message:</strong> {{$json["Additional message or Query"]}} <br><br>
     📄 <a href="https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit">View spreadsheet answers</a>
     ```
   - Save the node.

4. **Connect the nodes as follows:**
   - Google Sheets Trigger → Send a message → Send a message1

5. **Verify Google Sheet Column Headers**
   - Ensure your Google Sheet columns exactly match:
     - Timestamp
     - Full Name
     - Email
     - Phone Number (optional)
     - What are you interested in?
     - Additional message or query
   - If column names differ, update all node expressions accordingly.

6. **Test the Workflow**
   - Submit a test response via your Google Form.
   - Check that the confirmation email is received by the test lead email.
   - Verify the internal notification email is sent to your team.
   - Monitor the workflow execution for errors or failed runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| 📌 This workflow listens to Google Form responses captured in Google Sheets and sends automated emails to leads and team. | Workflow Overview sticky note content                          |
| 📩 Personalize confirmation email content and verify Gmail credentials are connected properly before activation.          | Send a message node sticky note                                |
| 📣 Replace internal notification email address and optionally CC others to keep team informed.                            | Send a message1 node sticky note                               |
| 📊 Make sure Google Sheet columns exactly match expected headers or update node expressions accordingly.                  | Google Sheets Trigger sticky note                              |
| Documentation and workflow template inspired by n8n community automation patterns                                        | n8n community and official docs (https://docs.n8n.io/)         |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.