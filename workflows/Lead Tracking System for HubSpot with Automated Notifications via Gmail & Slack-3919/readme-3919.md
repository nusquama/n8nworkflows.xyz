Lead Tracking System for HubSpot with Automated Notifications via Gmail & Slack

https://n8nworkflows.xyz/workflows/lead-tracking-system-for-hubspot-with-automated-notifications-via-gmail---slack-3919


# Lead Tracking System for HubSpot with Automated Notifications via Gmail & Slack

### 1. Workflow Overview

This workflow automates the entire lead intake and notification process for businesses using Google Forms and Google Sheets as the frontend and database, respectively. It captures new lead submissions, logs them, integrates leads into HubSpot CRM, sends immediate notifications via Slack and Gmail, and tracks follow-ups with reminder emails if leads remain unattended.

**Logical Blocks:**

- **1.1 Lead Input Detection and Data Logging**  
  Captures new lead entries from Google Sheets triggered by form submissions.

- **1.2 CRM Integration**  
  Automatically adds new leads into HubSpot CRM with detailed lead information.

- **1.3 Notification Dispatch**  
  Sends immediate alerts about new leads to Slack channels and Gmail inboxes.

- **1.4 Follow-up Reminder System**  
  Waits a predefined period, checks if a lead has been followed up, and sends reminder emails if necessary.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Input Detection and Data Logging

- **Overview:**  
  This block detects new lead entries submitted via Google Forms and recorded in a Google Sheet. It triggers the workflow when new data arrives.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**

  - **Google Sheets Trigger**  
    - *Type:* Trigger node for Google Sheets  
    - *Role:* Detects new rows added to the Google Sheet linked with the Google Form.  
    - *Configuration:*  
      - Polls every minute to check for new entries in the "Form Responses 1" sheet.  
      - Uses OAuth2 credentials for Google Sheets API access.  
    - *Input/Output:* No input; outputs new row data as JSON.  
    - *Edge Cases:*  
      - OAuth token expiration or permission issues can prevent triggering.  
      - Polling frequency may cause slight delay in detection.  
      - If the sheet or form is deleted or renamed, trigger will fail.  
    - *Sticky Note:*  
      - Positioned with a note explaining the lead submission is via Google Forms. ([Google Forms](https://forms.gle/VLhKeRySSWNKo2aR8))

---

#### 1.2 CRM Integration

- **Overview:**  
  Processes the incoming lead data by adding leads into HubSpot CRM, enriching them with notes, phone numbers, and interest level.

- **Nodes Involved:**  
  - HubSpot

- **Node Details:**

  - **HubSpot**  
    - *Type:* CRM integration node  
    - *Role:* Creates or updates a contact in HubSpot using lead information.  
    - *Configuration:*  
      - Maps form fields to HubSpot contact properties: Email, phone, salutation (used for lead source), relationship status (interest level), and message (notes).  
      - Uses OAuth2 for authentication.  
    - *Key Expressions:*  
      - Email: `={{ $json['E-Mail'] }}`  
      - Phone: `={{ $json.Phone }}`  
      - Interest Level: `={{ $json['  Interest Level  '] }}`  
    - *Input/Output:* Takes lead data from Google Sheets Trigger; outputs HubSpot API response.  
    - *Edge Cases:*  
      - API rate limits or authentication errors.  
      - Missing or malformed email may cause failure.  
      - Network issues impacting API calls.  
    - *Sticky Note:*  
      - Highlights automatic addition to HubSpot with relevant fields.

---

#### 1.3 Notification Dispatch

- **Overview:**  
  Sends immediate notifications to the sales/support team via Slack and Gmail to alert them of new leads.

- **Nodes Involved:**  
  - Slack  
  - Gmail

- **Node Details:**

  - **Slack**  
    - *Type:* Messaging node for Slack  
    - *Role:* Sends a formatted message to a specified Slack channel with lead details.  
    - *Configuration:*  
      - Uses OAuth2 credentials.  
      - Posts to channel ID "C08FJNLQP5G" (named "test-automation-workflow").  
      - Message template includes name, email, phone, interest level, lead source, and notes.  
    - *Key Expressions:* Uses JSON fields from trigger node for dynamic content.  
    - *Edge Cases:*  
      - OAuth token expiration or invalid webhook ID.  
      - Slack API rate limits or channel permissions.  
      - Missing fields may render incomplete messages.  

  - **Gmail**  
    - *Type:* Email sending node  
    - *Role:* Sends an email notification about the new lead.  
    - *Configuration:*  
      - Recipient: dataplusminuss@gmail.com  
      - Subject: Includes lead’s name dynamically.  
      - HTML formatted message body with lead details.  
      - Uses OAuth2 credentials.  
    - *Edge Cases:*  
      - Authentication failures.  
      - Gmail API quota limits.  
      - Email delivery delays or spam filtering.

  - *Connections:* Both Slack and Gmail nodes receive input from Google Sheets Trigger.

  - *Sticky Note:*  
    - Describes that alerts are sent simultaneously via Slack and Gmail.

---

#### 1.4 Follow-up Reminder System

- **Overview:**  
  Waits for a period (3 minutes in this demo) and verifies if the lead has been followed up by checking a specific column in the Google Sheet. If the lead is marked as not followed up and the interest level is "Hot," it sends a reminder email; otherwise, it performs no operation.

- **Nodes Involved:**  
  - Wait  
  - If  
  - Gmail_Reminder  
  - No Operation, do nothing

- **Node Details:**

  - **Wait**  
    - *Type:* Delay node  
    - *Role:* Pauses workflow execution for 3 minutes (configurable).  
    - *Configuration:* Wait unit set to minutes; amount = 3.  
    - *Edge Cases:* Workflow execution timeout if delayed too long in production.  

  - **If**  
    - *Type:* Conditional node  
    - *Role:* Checks two conditions:  
      1. The "Followed Up?" column is empty (lead not yet followed up).  
      2. The "Interest Level" contains "Hot".  
    - *Configuration:*  
      - Case-sensitive strict string validation.  
      - Logical AND combinator between conditions.  
    - *Output:*  
      - True branch if both conditions met.  
      - False branch otherwise.  
    - *Edge Cases:*  
      - Missing or malformed columns; expression failures.  
      - Case sensitivity might miss variants like "hot" lowercase.  

  - **Gmail_Reminder**  
    - *Type:* Email node  
    - *Role:* Sends a reminder email to prompt follow-up on hot leads.  
    - *Configuration:*  
      - Recipient: dataplusminuss@gmail.com  
      - Subject: "⏰ *Follow-up Reminder!*"  
      - Sender name customized as "N_01_tester".  
      - HTML body includes lead details and follow-up prompt.  
      - Uses OAuth2 credentials.  
    - *Edge Cases:* Same as Gmail node above.  

  - **No Operation, do nothing**  
    - *Type:* NoOp node  
    - *Role:* Ends the workflow without action for leads not meeting reminder criteria.  
    - *Edge Cases:* None.  

  - *Connections:*  
    - Wait node output connects to If node.  
    - If node true branch connects to Gmail_Reminder.  
    - If node false branch connects to No Operation.  

  - *Sticky Note:*  
    - Explains follow-up tracking and reminder logic based on the "Followed Up?" column.

---

### 3. Summary Table

| Node Name                | Node Type                    | Functional Role                         | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                          |
|--------------------------|------------------------------|---------------------------------------|------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Google Sheets Trigger     | Google Sheets Trigger         | Detect new lead entries from sheet    | —                      | Slack, Gmail, HubSpot, Wait    | # Lead Submission: User submits lead via Google Forms (https://forms.gle/VLhKeRySSWNKo2aR8)        |
| Slack                    | Slack                        | Send Slack notification about lead    | Google Sheets Trigger   | —                              | Alerts sent simultaneously via Slack and Gmail                                                    |
| Gmail                    | Gmail                        | Send email notification about lead    | Google Sheets Trigger   | —                              | Alerts sent simultaneously via Slack and Gmail                                                    |
| HubSpot                  | HubSpot                      | Add lead to HubSpot CRM                | Google Sheets Trigger   | —                              | Lead automatically added to HubSpot with relevant fields                                          |
| Wait                     | Wait                         | Delay workflow before follow-up check | Google Sheets Trigger   | If                            | # Follow-up Tracking: Reminder sent if lead not followed up after wait                             |
| If                       | If                           | Conditional check for follow-up status| Wait                   | Gmail_Reminder, No Operation   | # Follow-up Tracking: Checks "Followed Up?" column and interest level                              |
| Gmail_Reminder           | Gmail                        | Send follow-up reminder email          | If                      | —                              | Reminder sent if lead is hot and not followed up                                                  |
| No Operation, do nothing | No Operation                 | End workflow without action            | If                      | —                              | No action taken if lead followed up or interest not hot                                           |
| Sticky Note              | Sticky Note                  | Informational notes                    | —                      | —                              | Various notes explaining blocks; see detailed sticky note content in sections above              |
| Sticky Note1             | Sticky Note                  | Informational note                     | —                      | —                              | Automation trigger description                                                                     |
| Sticky Note2             | Sticky Note                  | Informational note                     | —                      | —                              | Data logging information                                                                           |
| Sticky Note3             | Sticky Note                  | Informational note                     | —                      | —                              | CRM integration details                                                                            |
| Sticky Note4             | Sticky Note                  | Informational note                     | —                      | —                              | Notifications explanation                                                                          |
| Sticky Note5             | Sticky Note                  | Informational note                     | —                      | —                              | Follow-up tracking and reminder explanation                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node**  
   - Type: Google Sheets Trigger  
   - Credentials: OAuth2 account with access to the Google Sheet  
   - Poll every minute for new rows in the sheet named "Form Responses 1" within the document ID corresponding to the lead spreadsheet.  
   - No manual input; outputs lead data as JSON.

2. **Add HubSpot node**  
   - Type: HubSpot  
   - Credentials: OAuth2 HubSpot account  
   - Configure to create or update a contact with fields:  
     - Email: `={{ $json['E-Mail'] }}`  
     - Phone: `={{ $json.Phone }}`  
     - Salutation: `={{ $json['  Lead Source  '] }}` (used to capture lead source)  
     - Relationship Status: `={{ $json['  Interest Level  '] }}`  
     - Message: `={{ $json['Notes '] }}`  
   - Connect Google Sheets Trigger output to HubSpot input.

3. **Add Slack node**  
   - Type: Slack  
   - Credentials: OAuth2 Slack account with posting permissions  
   - Set channel ID to the target Slack channel (e.g., "C08FJNLQP5G")  
   - Message template:  
     ```
     🎯 *New Lead Alert!*

     *Name:* {{ $json['Name Surname'] }}
     *Email:* {{ $json['E-Mail'] }}
     *Phone:* {{ $json['Phone'] }}
     *Interest Level:* {{ $json['  Interest Level  '] }}
     *Source:* {{ $json['  Lead Source  '] }}

     📝 Notes:
     {{ $json['Notes '] }}
     ```
   - Connect Google Sheets Trigger output to Slack input.

4. **Add Gmail node for lead notification**  
   - Type: Gmail  
   - Credentials: OAuth2 Gmail account  
   - Recipient: dataplusminuss@gmail.com  
   - Subject: `📩 New Lead Received: {{ $json['Name Surname'] }}`  
   - Message (HTML):  
     ```
     <h3>New Lead Received!</h3> 
     <ul>   
       <li><strong>Name:</strong> {{ $json['Name Surname'] }}</li>   
       <li><strong>Email:</strong> {{ $json['E-Mail'] }}</li>   
       <li><strong>Phone:</strong> {{ $json['Phone'] }}</li>   
       <li><strong>Interest Level:</strong> {{ $json['  Interest Level  '] }}</li>   
       <li><strong>Source:</strong> {{ $json['  Lead Source  '] }}</li> 
     </ul> 
     <p><strong>Notes:</strong> {{ $json['Notes '] }}</p>
     ```
   - Connect Google Sheets Trigger output to Gmail input.

5. **Add Wait node**  
   - Type: Wait  
   - Set to wait for 3 minutes (demo duration; can be increased for real follow-up timing)  
   - Connect Google Sheets Trigger output to Wait input.

6. **Add If node**  
   - Type: If  
   - Conditions:  
     - Check if "Followed Up?" field exists and is empty: `={{ $json['Followed Up?'] }}` exists and is empty string  
     - AND "Interest Level" contains "Hot" (case sensitive)  
   - Connect Wait output to If input.

7. **Add Gmail node for follow-up reminder (Gmail_Reminder)**  
   - Type: Gmail  
   - Credentials: Same Gmail OAuth2 account  
   - Recipient: dataplusminuss@gmail.com  
   - Subject: `⏰ *Follow-up Reminder!*`  
   - Sender name: "N_01_tester"  
   - Message (HTML):  
     ```
     <h3>🔔 The following lead has not been followed up yet! 🔥 Interest level is hot </h3>
     <ul>
       <li><strong>Name:</strong> {{ $json['Name Surname'] }}</li>
       <li><strong>Email:</strong> {{ $json['E-Mail'] }}</li>
       <li><strong>Interest Level:</strong> {{ $json['  Interest Level  '] }}</li>
     </ul>
     <p><strong>Please follow up and update the spreadsheet ✅</p>
     ```
   - Connect If node's "true" output to this Gmail node.

8. **Add No Operation node**  
   - Type: No Operation (NoOp)  
   - Connect If node's "false" output to this node.

9. **Connect all trigger outputs**  
   - From Google Sheets Trigger node, connect outputs in parallel to Slack, Gmail (new lead), HubSpot, and Wait nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Workflow built by Adem Tasin, available on Upwork: [Adem T.](https://www.upwork.com/freelancers/~01807137bc7526bcb2?mp_source=share) | Creator contact and profile                                                                                       |
| Website for project updates and contact: [Dataplusminus+-](https://www.dataplusminus.com/)                                         | Official project site                                                                                              |
| Slack channel used named "test-automation-workflow"                                                                                 | Slack workspace/channel for notifications                                                                         |
| Google Forms link for lead submission: https://forms.gle/VLhKeRySSWNKo2aR8                                                         | Frontend lead capture form                                                                                         |
| Google Sheet used for data logging: https://docs.google.com/spreadsheets/d/16xNeIG_QLUtOoFulbWemXrUAOKwxaHaGU7DywJLDiRk/edit        | Backend data storage                                                                                               |
| Suggested improvements include email validation via third-party APIs, CRM deeper integration, lead scoring, and follow-up automation | Possible feature roadmap                                                                                           |