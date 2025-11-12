Email new leads from Google Sheets via Outlook on a schedule

https://n8nworkflows.xyz/workflows/email-new-leads-from-google-sheets-via-outlook-on-a-schedule-8095


# Email new leads from Google Sheets via Outlook on a schedule

### 1. Workflow Overview

This workflow automates sending outreach emails to new leads listed in a Google Sheet on a daily schedule using Microsoft Outlook. It ensures that each lead is contacted only once by marking them as “Contacted” in a secondary Google Sheet. The workflow is structured into four logical blocks:

- **1.1 Schedule Trigger:** Initiates the workflow once daily at a specified hour.
- **1.2 Lead Retrieval from Google Sheets:** Reads lead entries from a Google Sheet containing new leads.
- **1.3 Filtering Uncontacted Leads:** Filters out leads who have already been contacted.
- **1.4 Email Sending and Lead Status Update:** Sends a templated email via Outlook to uncontacted leads and updates their status in a Google Sheet to avoid duplicate outreach.

This design allows for scalable, repeatable outreach automation with clear tracking of contacted leads.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block triggers the workflow automatically every day at a specified hour to initiate the lead outreach process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* `scheduleTrigger` - Starts the workflow based on a time schedule.  
    - *Configuration:* Set to trigger daily at 9:00 AM.  
    - *Key Expressions/Variables:* None.  
    - *Input/Output Connections:* No input nodes; outputs to the Google Sheets node ("Get row(s) in sheet3").  
    - *Version:* 1.2  
    - *Edge Cases/Potential Failures:*  
      - System time zone differences might affect trigger timing if not configured correctly.  
      - Workflow may fail to trigger if n8n instance is down.  
    - *Sub-workflow:* None.

---

#### 2.2 Lead Retrieval from Google Sheets

- **Overview:**  
  Fetches all rows from a Google Sheet representing new leads to process them for outreach.

- **Nodes Involved:**  
  - Get row(s) in sheet3

- **Node Details:**

  - **Get row(s) in sheet3**  
    - *Type & Role:* `googleSheets` - Reads rows from a specified Google Sheet.  
    - *Configuration:*  
      - Document ID: Google Sheet “New Leads” (spreadsheet ID `1sXXVbl2kKdYTzCmZDe7QyeMp1N9wZg9K63oGK2UIaeU`).  
      - Sheet Name: First sheet (gid=0).  
      - No filters or ranges specified, so it retrieves all rows by default.  
    - *Key Expressions/Variables:* None used here.  
    - *Input/Output Connections:* Input from Schedule Trigger; outputs data to Filter1 node.  
    - *Version:* 4.7  
    - *Edge Cases/Potential Failures:*  
      - Authentication errors if OAuth token expires or is revoked.  
      - Empty or malformed sheets may cause empty outputs or errors.  
      - Network timeouts or Google API rate limiting.  
    - *Sub-workflow:* None.

---

#### 2.3 Filtering Uncontacted Leads

- **Overview:**  
  Filters the fetched leads to only those who have not yet been contacted (i.e., where the “Contacted” field is empty).

- **Nodes Involved:**  
  - Filter1

- **Node Details:**

  - **Filter1**  
    - *Type & Role:* `filter` - Filters data based on conditions.  
    - *Configuration:*  
      - Condition: The field `Contacted` is empty (string empty check, case sensitive).  
    - *Key Expressions/Variables:*  
      - Expression testing `$json.Contacted` for emptiness (`{{$json.Contacted}} == ""`).  
    - *Input/Output Connections:* Input from Google Sheets node; outputs to both Append or Update row and Send a message nodes.  
    - *Version:* 2.2  
    - *Edge Cases/Potential Failures:*  
      - If the `Contacted` field is missing or named differently, filter will fail or behave unexpectedly.  
      - Case sensitivity might cause false negatives if values vary in case.  
    - *Sub-workflow:* None.

---

#### 2.4 Email Sending and Lead Status Update

- **Overview:**  
  Sends a templated outreach email to each uncontacted lead using Outlook and then marks the lead as contacted in another Google Sheet to prevent duplicate emails.

- **Nodes Involved:**  
  - Append or update row in sheet1  
  - Send a message

- **Node Details:**

  - **Append or update row in sheet1**  
    - *Type & Role:* `googleSheets` - Updates or appends rows in a Google Sheet to track contacted leads.  
    - *Configuration:*  
      - Operation: `appendOrUpdate` on the “leads” sheet (gid=0) of spreadsheet ID `14T6rilaOl1LBTwNnu7ILE3T1equWOz0noI-OIUaI3zU`.  
      - Matching Column: “Email” used to find existing row to update; otherwise appends new.  
      - Columns updated: “Email” with lead email, “Contacted” set to "Yes".  
      - No automatic type conversion or string conversion.  
    - *Key Expressions/Variables:*  
      - Uses lead email and sets `Contacted` to “Yes”.  
    - *Input/Output Connections:* Input from Filter1; outputs to Send a message.  
    - *Version:* 4.7  
    - *Edge Cases/Potential Failures:*  
      - Authentication issues with Google Sheets OAuth2.  
      - Race conditions if multiple workflow runs update the same row simultaneously.  
      - Data schema mismatch if columns are renamed or missing.  
    - *Sub-workflow:* None.

  - **Send a message**  
    - *Type & Role:* `microsoftOutlook` - Sends an email via Outlook using OAuth2 credentials.  
    - *Configuration:*  
      - Subject: “Build AI Agents & Automations with n8n”.  
      - Body: Personalized outreach message including contact info and links.  
      - To: Dynamically mapped to `{{$json.Email}}` from lead data.  
      - Credential: Microsoft Outlook OAuth2 credential configured for sending emails.  
    - *Key Expressions/Variables:*  
      - `toRecipients` uses expression to map to lead’s email.  
    - *Input/Output Connections:* Input from Append or update row node; final output of the workflow.  
    - *Version:* 2  
    - *Edge Cases/Potential Failures:*  
      - OAuth token expiration or permission errors leading to failed sends.  
      - Invalid email addresses causing send failures.  
      - API rate limiting by Microsoft Graph.  
      - Network timeouts.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                              | Input Node(s)           | Output Node(s)                      | Sticky Note                                                                                                 |
|----------------------------|-----------------------------|----------------------------------------------|-------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | scheduleTrigger             | Initiates workflow daily at 9 AM              | —                       | Get row(s) in sheet3               | ## Email new leads from Google Sheets via Outlook on a schedule (shared with entire workflow block)          |
| Get row(s) in sheet3       | googleSheets                | Retrieves leads from Google Sheet “New Leads”| Schedule Trigger        | Filter1                           | See notes on Google Sheets OAuth2 setup, spreadsheet and sheet selection (Sticky Note70)                     |
| Filter1                    | filter                      | Filters leads with empty “Contacted” field   | Get row(s) in sheet3    | Append or update row in sheet1     | Shared sticky note about workflow overview and Gmail/O365 setup (Sticky Note58)                              |
| Append or update row in sheet1 | googleSheets             | Marks leads as contacted in tracking sheet   | Filter1                 | Send a message                    | Shared sticky note about workflow overview and Gmail/O365 setup (Sticky Note58)                              |
| Send a message             | microsoftOutlook            | Sends outreach email to leads                  | Append or update row in sheet1 | —                             | Shared sticky note about workflow overview and Gmail/O365 setup (Sticky Note58); Outlook OAuth2 setup details (Sticky Note7) |
| Sticky Note58              | stickyNote                  | Workflow overview and summary instructions    | —                       | —                                | ## Email new leads from Google Sheets via Outlook on a schedule ... workflow summary and architecture details|
| Sticky Note6               | stickyNote                  | Setup instructions for Google Sheets & Outlook | —                     | —                                | Detailed setup guide for Google Sheets and Outlook OAuth2 credentials, self-hosted and n8n cloud instructions|
| Sticky Note70              | stickyNote                  | Google Sheets OAuth2 setup quick guide        | —                       | —                                | Step-by-step instructions for Google Sheets OAuth2 credential creation and usage                            |
| Sticky Note7               | stickyNote                  | Outlook OAuth2 credential setup details       | —                       | —                                | Detailed instructions for Microsoft Outlook OAuth2 credential creation and permissions                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: `Schedule Trigger`  
   - Set to run daily at 9:00 AM (triggerAtHour: 9).  
   - No credentials required.

2. **Create Google Sheets Node to Get Rows**  
   - Type: `Google Sheets`  
   - Operation: Read Rows (default).  
   - Connect to Schedule Trigger.  
   - Set `Document ID` to your Google Sheet with leads (e.g., "New Leads").  
   - Set `Sheet Name` to the relevant worksheet (e.g., gid=0 or "Sheet1").  
   - Select OAuth2 credentials for Google Sheets. Ensure credentials have access to the spreadsheet.

3. **Create Filter Node**  
   - Type: `Filter`  
   - Connect input from Google Sheets node.  
   - Add condition: Field `Contacted` is empty (string empty check).  
   - Output: Only rows where `Contacted` is empty will pass through.

4. **Create Google Sheets Node to Append or Update Row**  
   - Type: `Google Sheets`  
   - Operation: `appendOrUpdate`.  
   - Connect input from Filter node.  
   - Set `Document ID` to the tracking Google Sheet (could be same or different sheet).  
   - Set `Sheet Name` to the leads tracking worksheet.  
   - Define matching column as `Email`.  
   - Map fields: set `Email` to lead email, `Contacted` to “Yes”.  
   - Use same or different Google Sheets OAuth2 credentials with write access.

5. **Create Microsoft Outlook Node to Send Email**  
   - Type: `Microsoft Outlook`  
   - Operation: Send Message.  
   - Connect input from Append/Update Google Sheets node.  
   - Set recipient field `To` to expression: `{{$json.Email}}`.  
   - Define email subject and body with your outreach message, including any personalization or links.  
   - Select Microsoft Outlook OAuth2 credentials with `Mail.Send` permissions.

6. **Link Connections**  
   - Schedule Trigger → Get row(s) in sheet3  
   - Get row(s) in sheet3 → Filter1  
   - Filter1 → Append or update row in sheet1  
   - Append or update row in sheet1 → Send a message

7. **Configure Credentials**  
   - Google Sheets OAuth2: Create in n8n credentials, sign in with Google account, grant access to target spreadsheet(s).  
   - Microsoft Outlook OAuth2:  
     - For n8n Cloud, use quick connect and authorize.  
     - For self-hosted, register Azure app with `offline_access`, `Mail.Send`, `User.Read` delegated permissions, add redirect URI `https://YOUR_N8N_URL/rest/oauth2-credential/callback`, create client secret, and enter details in n8n.

8. **Test Workflow**  
   - Trigger manually or wait for scheduled run.  
   - Verify leads with empty `Contacted` receive emails.  
   - Confirm that those leads have their `Contacted` field updated to “Yes” in the tracking sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates lead outreach by emailing new Google Sheets leads via Outlook on a schedule, then marks leads as contacted to avoid duplicates. Built with a simple chain: Schedule Trigger → Google Sheets → Filter → Outlook Send Email → Google Sheets update.                                                                                                                                                                                                                                                                                                                                                                          | Workflow overview and main architecture                                                                                          |
| Setup instructions for Google Sheets OAuth2 credential: create new OAuth2 credential in n8n, sign in, select appropriate spreadsheet and sheet in each Google Sheets node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                | https://docs.google.com/spreadsheets/d/1sXXVbl2kKdYTzCmZDe7QyeMp1N9wZg9K63oGK2UIaeU/edit#gid=0                                   |
| Microsoft Outlook OAuth2 credential setup: For n8n Cloud, use quick connect; for self-hosted, register app in Azure portal with required API permissions (`offline_access`, `Mail.Send`, `User.Read`), add redirect URI, create a client secret, and configure credential in n8n. Keep `To` parameter dynamically mapped to lead email. Customize subject and body to brand.                                                                                                                                                                                                                                 | https://www.linkedin.com/in/robert-breen-29429625/ (author contact)                                                              |
| Contact for workflow author: Robert Breen, email: rbreen@ynteractive.com, website: https://ynteractive.com                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Author contact and branding                                                                                                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.