Automate Lead Qualification with RetellAI Phone Agent, OpenAI GPT-4 & Google Sheets

https://n8nworkflows.xyz/workflows/automate-lead-qualification-with-retellai-phone-agent--openai-gpt-4---google-sheets-3912


# Automate Lead Qualification with RetellAI Phone Agent, OpenAI GPT-4 & Google Sheets

### 1. Workflow Overview

This n8n workflow automates lead qualification and call management for sales teams, call centers, and businesses managing outbound and inbound phone leads. It integrates RetellAI (for automated phone calls), Google Sheets (for lead data storage), OpenAI GPT-4 (for call summarization), and Gmail (for communication). The workflow is logically divided into three main blocks:

- **1.1 Outbound Lead Qualification Workflow**  
  Detects new leads in Google Sheets, sends SMS reminders, optionally waits, and initiates automated outbound calls using RetellAI.

- **1.2 Inbound Call Appointment Scheduler**  
  Receives inbound call events from RetellAI via webhook, verifies the caller’s phone number against Google Sheets, and responds to RetellAI with appropriate dynamic variables.

- **1.3 Post-Call Processing Workflow**  
  Receives analyzed call data from RetellAI via webhook, filters for analyzed calls, updates lead records in Google Sheets with call qualifications, generates call summaries using OpenAI GPT-4, and sends summary emails to team inboxes or reps.

---

### 2. Block-by-Block Analysis

#### 2.1 Outbound Lead Qualification Workflow

**Overview:**  
This block detects when a new lead is added to Google Sheets, sends an SMS reminder to the sales rep to call the lead shortly, optionally waits 5 minutes, then triggers RetellAI to place an automated outbound call to the lead.

**Nodes Involved:**  
- Detect new lead in Google Sheets  
- Send SMS reminder to call lead in 5 minutes  
- Wait 5 minutes before making call (optional)  
- Call new lead using RetellAI  
- Sticky Note (for labeling)

**Node Details:**

- **Detect new lead in Google Sheets**  
  - Type: Google Sheets Trigger  
  - Role: Watches for new rows added to a specified Google Sheet.  
  - Configuration: Polls every hour; triggers on "rowAdded" event. Spreadsheet ID and sheet name must be configured.  
  - Expressions: None explicitly; outputs the new row data including the lead’s phone number.  
  - Input: Trigger node (event-based).  
  - Output: Activates the SMS reminder node.  
  - Edge Cases: Network issues, Google Sheets API limits, malformed spreadsheet data, or missing phone numbers can cause failures.

- **Send SMS reminder to call lead in 5 minutes**  
  - Type: Twilio (SMS)  
  - Role: Sends SMS notification to the lead’s phone number to remind sales reps to call.  
  - Configuration:  
    - `To`: Filled dynamically from the lead’s "Phone Number" field in the trigger data.  
    - `From`: Fixed verified Twilio number (e.g., +33600000000).  
    - `Message`: Static text notifying about upcoming call.  
  - Input: Triggered from new lead detection.  
  - Output: Passes to the wait node.  
  - Edge Cases: Twilio authentication errors, invalid phone numbers, SMS limits or delivery failures.

- **Wait 5 minutes before making call**  
  - Type: Wait  
  - Role: Pauses the workflow for a configurable delay (default 1 minute here, optionally 5 minutes).  
  - Configuration: Wait duration in minutes.  
  - Input: SMS node output.  
  - Output: Triggers the RetellAI call node.  
  - Edge Cases: Workflow timeout if delay exceeds instance limits.

- **Call new lead using RetellAI**  
  - Type: HTTP Request  
  - Role: Sends API request to RetellAI to initiate a phone call to the lead.  
  - Configuration:  
    - URL: `https://api.retellai.com/v2/create-phone-call`  
    - Method: POST  
    - Headers: Authorization Bearer token (RetellAI API key), Content-Type application/json  
    - Body: JSON including `from_number`, `to_number` (lead’s phone), and dynamic variables including lead’s UUID from Google Sheets.  
  - Input: Wait node output.  
  - Output: Ends the outbound call trigger chain.  
  - Edge Cases: API authentication failure, invalid phone number format (must be E.164), RetellAI service downtime, malformed request body.

- **Sticky Note**  
  - Purpose: Visual label "Outbound lead qualification call workflow".  
  - No inputs or outputs.

---

#### 2.2 Inbound Call Appointment Scheduler

**Overview:**  
This block handles inbound calls received by RetellAI. It triggers via a webhook when an inbound call arrives, checks if the caller’s phone number exists in Google Sheets, and responds to RetellAI with caller-specific dynamic variables (e.g., caller’s name) or an error.

**Nodes Involved:**  
- Receive inbound call from RetellAI (webhook)  
- Check if phone number exists in Google Sheets  
- Send response to inbound webhook call  
- Sticky Note (for labeling)

**Node Details:**

- **Receive inbound call from RetellAI (webhook)**  
  - Type: Webhook  
  - Role: Receives HTTP POST requests from RetellAI for inbound call events.  
  - Configuration:  
    - HTTP Method: POST  
    - Path: Custom webhook path (unique identifier)  
    - Response Mode: Sends response via "Respond to Webhook" node downstream.  
  - Input: External RetellAI webhook calls.  
  - Output: Passes inbound call data to Google Sheets lookup.  
  - Edge Cases: Webhook URL misconfiguration, RetellAI failing to send webhook, malformed inbound data.

- **Check if phone number exists in Google Sheets**  
  - Type: Google Sheets (Read)  
  - Role: Searches Google Sheet for a row matching the inbound caller’s phone number.  
  - Configuration:  
    - Filter by column "Phone Number" equals the incoming number from webhook JSON path.  
    - Spreadsheet ID and sheet name configured.  
  - Input: Webhook data (caller number).  
  - Output: Passes lead information to response node.  
  - Edge Cases: Google Sheets API limits, no matching phone number found (empty result), malformed phone number formats.

- **Send response to inbound webhook call**  
  - Type: Respond to Webhook  
  - Role: Sends JSON response back to RetellAI including dynamic variables like the caller’s name.  
  - Configuration:  
    - Response body dynamically includes `name` field from matched Google Sheets row.  
  - Input: Google Sheets lookup output.  
  - Output: Final response to RetellAI webhook call.  
  - Edge Cases: Empty lookup results may yield empty or erroneous responses; malformed expressions.

- **Sticky Note**  
  - Purpose: Visual label "Inbound call appointment scheduler workflow".

---

#### 2.3 Post-Call Processing Workflow

**Overview:**  
Upon call completion, this block receives post-call data from RetellAI, filters only analyzed calls, determines if the call was outbound, updates the lead’s Google Sheet record with call qualification, generates a call summary using OpenAI GPT-4, and sends emails with the summary to team or lead.

**Nodes Involved:**  
- Receive post-call data from RetellAI (webhook)  
- Filter for analyzed calls only  
- Check if call was outbound (If node)  
- Update lead record in Google Sheets  
- Generate call summary with OpenAI  
- Send call summary email  
- Send confirmation email to lead  
- Sticky Note (for labeling and general explanation)

**Node Details:**

- **Receive post-call data from RetellAI (webhook)**  
  - Type: Webhook  
  - Role: Receives HTTP POST with call event data from RetellAI after call ends.  
  - Configuration:  
    - HTTP Method: POST  
    - Path: Custom unique webhook path  
  - Input: External RetellAI post-call webhook.  
  - Output: Passes data to filter node.  
  - Edge Cases: Webhook misconfiguration, incomplete data, RetellAI downtime.

- **Filter for analyzed calls only**  
  - Type: Filter  
  - Role: Allows only calls with event type "call_analyzed" to proceed.  
  - Configuration: Condition checking `$json.body.event === "call_analyzed"`.  
  - Input: Post-call webhook data.  
  - Output: Passes filtered data to the "Check if call was outbound" node.  
  - Edge Cases: No calls passing filter; unexpected event types.

- **Check if call was outbound**  
  - Type: If  
  - Role: Branches workflow based on call direction (outbound vs inbound).  
  - Configuration: Checks if `$json.body.call.direction === "outbound"`.  
  - Input: Filter node output.  
  - Output: Both true (outbound) or false (inbound) paths feed into the same subsequent nodes here.  
  - Edge Cases: Missing or malformed direction field.

- **Update lead record in Google Sheets**  
  - Type: Google Sheets (Update)  
  - Role: Updates the lead row matching the UUID with new qualification data from call analysis.  
  - Configuration:  
    - Matching column: UUID (from call dynamic variables)  
    - Updates "Qualification" column with analysis result.  
    - Spreadsheet ID and sheet name configured.  
  - Input: Output from If node.  
  - Output: Passes updated data to call summary email and OpenAI summary generation.  
  - Edge Cases: Matching row not found, Google Sheets API errors, data type mismatches.

- **Generate call summary with OpenAI**  
  - Type: OpenAI (GPT-4)  
  - Role: Generates a textual summary analyzing the call transcript and suggesting prompt improvements.  
  - Configuration:  
    - Model: GPT-4 (version 4.1)  
    - Prompt includes call transcript dynamically injected from webhook data.  
  - Input: Output from Google Sheets update node.  
  - Output: Passes AI-generated summary to confirmation email.  
  - Edge Cases: OpenAI API limits, malformed prompt data, timeout.

- **Send call summary email**  
  - Type: Gmail  
  - Role: Sends a plain-text email with call details and qualification to a fixed team inbox.  
  - Configuration:  
    - Recipient: Static email address (e.g., youremail@gmail.com)  
    - Subject: New Lead - Call Summary  
    - Message: Includes name, phone, qualification, and summary from call analysis data.  
  - Input: Output from Google Sheets update node.  
  - Output: Ends email notification chain.  
  - Edge Cases: Gmail auth errors, invalid recipient email, rate limits.

- **Send confirmation email to lead**  
  - Type: Gmail  
  - Role: Sends a detailed email including appointment details and AI call analysis summary, intended for lead or internal team.  
  - Configuration:  
    - Recipient: Static email (e.g., youremail@gmail.com)  
    - Subject: Roofing Appointment Scheduled  
    - Message: Combines client info, availabilities, call summary, and OpenAI analysis content.  
  - Input: Output from OpenAI node.  
  - Output: Ends email notification chain.  
  - Edge Cases: Gmail issues, email formatting errors.

- **Sticky Note**  
  - Purpose: General explanation of workflow dependencies and overview.

---

### 3. Summary Table

| Node Name                             | Node Type               | Functional Role                          | Input Node(s)                         | Output Node(s)                            | Sticky Note                                                                                      |
|-------------------------------------|-------------------------|----------------------------------------|-------------------------------------|------------------------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note                         | Sticky Note             | Label for Outbound Workflow             | —                                   | —                                        | "# Outbound lead qualification call workflow"                                                  |
| Detect new lead in Google Sheets    | Google Sheets Trigger   | Triggers on new lead added to Sheet    | —                                   | Send SMS reminder to call lead in 5 minutes |                                                                                                |
| Send SMS reminder to call lead in 5 minutes | Twilio (SMS)            | Sends SMS reminder to sales rep        | Detect new lead in Google Sheets    | Wait 5 minutes before making call          |                                                                                                |
| Wait 5 minutes before making call  | Wait                    | Optional delay before call              | Send SMS reminder                   | Call new lead using RetellAI               |                                                                                                |
| Call new lead using RetellAI        | HTTP Request            | Initiates automated outbound call      | Wait 5 minutes                     | —                                        |                                                                                                |
| Sticky Note4                        | Sticky Note             | Label for Inbound Workflow              | —                                   | —                                        | "# Inbound call appointment scheduler workflow"                                                |
| Receive inbound call from RetellAI (webhook) | Webhook                 | Receives inbound call webhook           | —                                   | Check if phone number exists in Google Sheets |                                                                                                |
| Check if phone number exists in Google Sheets | Google Sheets           | Looks up caller number in leads         | Receive inbound call from RetellAI  | Send response to inbound webhook call      |                                                                                                |
| Send response to inbound webhook call | Respond to Webhook      | Sends caller info response to RetellAI | Check if phone number exists        | —                                        |                                                                                                |
| Sticky Note2                       | Sticky Note             | Label for Post-Call Workflow            | —                                   | —                                        | "# Post-call workflow\nTriggers when a new lead is added in Google Sheets:\n1 -Sends SMS...\n💡 Requires phone numbers to be formatted in E.164" |
| Receive post-call data from RetellAI (webhook) | Webhook                 | Receives analyzed call data             | —                                   | Filter for analyzed calls only             |                                                                                                |
| Filter for analyzed calls only     | Filter                  | Allows only analyzed calls              | Receive post-call data from RetellAI | Check if call was outbound                  |                                                                                                |
| Check if call was outbound          | If                      | Branches depending on call direction   | Filter for analyzed calls only      | Update lead record in Google Sheets; Generate call summary with OpenAI |                                                                                                |
| Update lead record in Google Sheets | Google Sheets           | Updates lead qualification in Sheet    | Check if call was outbound          | Send call summary email                     |                                                                                                |
| Generate call summary with OpenAI  | OpenAI                  | Generates call summary and improvements | Check if call was outbound          | Send confirmation email to lead             |                                                                                                |
| Send call summary email            | Gmail                   | Emails call summary to team inbox       | Update lead record in Google Sheets | —                                        |                                                                                                |
| Send confirmation email to lead    | Gmail                   | Emails detailed appointment and AI analysis | Generate call summary with OpenAI  | —                                        |                                                                                                |
| Sticky Note1                       | Sticky Note             | General workflow explanation and dependencies | —                               | —                                        | "# ✅ General Workflow Explanation\nThis workflow automates outbound and inbound lead calls..." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger node**  
   - Set event to "rowAdded" to detect new lead additions.  
   - Configure with your Google Sheets credentials, spreadsheet ID, and target sheet name.  
   - Set polling interval (e.g., every hour).

2. **Add a Twilio node to send SMS reminders**  
   - Configure Twilio credentials with your account.  
   - Set the "To" field dynamically to `{{$json["Phone Number"]}}` from the trigger data.  
   - Set "From" to your Twilio verified number (e.g., +33600000000).  
   - Write a custom message, e.g., "Hello, thanks for your interest...".

3. **Insert a Wait node (optional delay)**  
   - Set to pause for 5 minutes (or your preferred delay).  
   - Connect it after the SMS node.

4. **Add an HTTP Request node to initiate outbound calls via RetellAI**  
   - Method: POST  
   - URL: `https://api.retellai.com/v2/create-phone-call`  
   - Headers: Add `Authorization: Bearer <Your RetellAI API Key>`, and `Content-Type: application/json`  
   - Body (JSON):  
     ```json
     {
       "from_number": "+33600000000",
       "to_number": "{{$json['Phone Number']}}",
       "retell_llm_dynamic_variables": {
         "uuid": "{{$json.UUID}}" 
       }
     }
     ```  
   - Connect after Wait node.

5. **Create a Webhook node to receive inbound call events**  
   - HTTP Method: POST  
   - Define a unique webhook path.  
   - Configure to forward output to next node.

6. **Add a Google Sheets node to lookup caller phone number**  
   - Use Google Sheets read operation.  
   - Filter rows where "Phone Number" equals `{{$json.body.call_inbound.from_number}}`.  
   - Configure with your credentials and spreadsheet.

7. **Add a Respond to Webhook node**  
   - Send JSON response to RetellAI including dynamic variables, e.g.:  
     ```json
     {
       "call_inbound": {
         "dynamic_variables": {
           "name": "{{$json.Name}}"
         }
       }
     }
     ```  
   - Connect after the lookup node.

8. **Create a Webhook node to receive post-call data**  
   - HTTP Method: POST  
   - Unique webhook path distinct from inbound call webhook.

9. **Add a Filter node to allow only events where `body.event === "call_analyzed"`**  
   - Connect after post-call webhook.

10. **Add an If node to check call direction**  
    - Condition: `body.call.direction === "outbound"`  
    - Connect after filter node.

11. **Add a Google Sheets Update node**  
    - Update lead record based on `UUID` matching `body.call.retell_llm_dynamic_variables.uuid`.  
    - Update "Qualification" field with `body.call.call_analysis.custom_analysis_data.qualification`.  
    - Configure with credentials and spreadsheet info.

12. **Add OpenAI node (GPT-4)**  
    - Configure OpenAI credentials with your API key.  
    - Model: GPT-4 (or latest available).  
    - Prompt:  
      ```
      Analyze this call transcript to identify how the call went and identify possible improvements to the voice prompt: 

      {{$json.body.call.transcript}}
      ```  
    - Connect after Google Sheets update node.

13. **Add Gmail node to send call summary email to team**  
    - Configure Gmail OAuth2 credentials.  
    - Recipient: e.g., "youremail@gmail.com" or dynamic email.  
    - Subject: "New Lead - Call Summary"  
    - Message body: Include name, phone number, qualification, and summary from call analysis.

14. **Add another Gmail node to send confirmation email to lead or rep**  
    - Use similar Gmail credentials.  
    - Message includes appointment details and OpenAI analysis content dynamically inserted.  
    - Connect after OpenAI node.

15. **Add Sticky Notes throughout the canvas**  
    - Label each major block for clarity: Outbound Workflow, Inbound Scheduler, Post-Call Workflow, and General Workflow Explanation.

16. **Connect all nodes as per the logic:**  
    - New lead trigger → SMS → Wait → RetellAI call node.  
    - Inbound webhook → Google Sheets lookup → Respond to webhook.  
    - Post-call webhook → Filter → If (outbound) → Update Sheets → Email and OpenAI → Email.

17. **Set up credentials:**  
    - Google Sheets: OAuth2 or API key with read/write access to your spreadsheet.  
    - Twilio: Account SID and Auth Token with SMS permissions.  
    - RetellAI: API key for HTTP request authentication.  
    - OpenAI: API key with GPT-4 access.  
    - Gmail: OAuth2 credentials for sending emails.

18. **Ensure phone numbers in Google Sheets and inputs are E.164 formatted** (e.g., +33600000000) for compatibility.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Requires RetellAI API key, Gmail OAuth2 credentials, Google Sheets with phone numbers, and Twilio account for SMS.                                      | Workflow dependencies                                                                                |
| Phone numbers must be stored and passed in E.164 format (+countrycode...) to ensure API compatibility with Twilio and RetellAI.                        | Phone number formatting requirement                                                                 |
| Video overview of this workflow is available: ![Workflow Screenshot](https://www.dr-firas.com/Build-a-Phone-Agent.png)                                 | Visual workflow screenshot                                                                           |
| Full setup guide and customization instructions: [Notion Documentation](https://automatisation.notion.site/Build-a-Phone-Agent-to-qualify-outbound-leads-and-schedule-inbound-calls-1eb3d6550fd9807993dce3c6ed111554) | Detailed configuration and customization guide                                                      |
| Contact for consulting and support: [LinkedIn - Dr. Firass](https://www.linkedin.com/in/doctor-firass/)                                                  | Support and customization help                                                                      |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automated workflow. It complies fully with content policies and contains no illegal or offensive material. All data processed is public and legal.