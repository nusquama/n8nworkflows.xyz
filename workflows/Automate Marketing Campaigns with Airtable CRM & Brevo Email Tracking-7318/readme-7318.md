Automate Marketing Campaigns with Airtable CRM & Brevo Email Tracking

https://n8nworkflows.xyz/workflows/automate-marketing-campaigns-with-airtable-crm---brevo-email-tracking-7318


# Automate Marketing Campaigns with Airtable CRM & Brevo Email Tracking

# Automate Marketing Campaigns with Airtable CRM & Brevo Email Tracking

---

### 1. Workflow Overview

This workflow automates marketing email campaigns by integrating Airtable as a CRM system and Brevo (formerly SendInBlue) for email sending and event tracking. It is designed for marketing teams that manage contacts and campaigns in Airtable and want to send personalized email campaigns while tracking delivery, open, and click events to update CRM records automatically.

The workflow is logically divided into two main blocks:

- **1.1 Email Sending Workflow**: Retrieves eligible contacts from Airtable, sends marketing emails using Brevo templates, logs interactions in Airtable, and manages batching and timing for sending emails.

- **1.2 Email Event Tracking Workflow**: Listens for incoming webhook events from Brevo about email delivery, opens, clicks, and unsubscribes, then updates Airtable records accordingly to reflect user engagement and opt-in status.

Each block handles a distinct phase of the marketing campaign lifecycle, ensuring data consistency between the CRM and email platform.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Sending Workflow

**Overview:**  
This block fetches contacts from the Airtable CRM who have opted-in, belong to a campaign, and have verified emails. It then sends marketing emails using a predefined Brevo template, extracts message IDs for tracking, creates corresponding interaction records in Airtable, and controls the sending rate using batching and wait times.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Search Company (Airtable)  
- Loop Over Items (SplitInBatches)  
- Send an email with an existing Template (SendInBlue)  
- Edit Fields (Set)  
- Create Interaction (Airtable)  
- Wait (Wait)  
- Sticky Note (Prerequisites & Overview)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs/Outputs: Output connected to Search Company node.  
  - Edge Cases: None; manual start only.

- **Search Company**  
  - Type: Airtable (Search)  
  - Role: Retrieves contacts from Airtable "COMPANY" table meeting filter criteria.  
  - Configuration: Searches records where `{Opt-in} = 1`, `{Campaign} = 1`, `{Checked Email} = 1`.  
  - Inputs: Trigger node output.  
  - Outputs: List of contacts passed to Loop Over Items.  
  - Edge Cases: Empty results if no contacts match; Airtable API limits or auth errors possible.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes contacts one by one or in smaller batches to control email sending load.  
  - Configuration: Default batching without additional options.  
  - Inputs: Output from Search Company.  
  - Outputs: Main output empty (for control flow), secondary output to Send an email node.  
  - Edge Cases: Possible delay if batch size is large; ensure batch size fits API limits.

- **Send an email with an existing Template**  
  - Type: SendInBlue (Brevo)  
  - Role: Sends email using a predefined Brevo template (Template ID 2) to each contact’s email address.  
  - Configuration: Uses template ID 2; recipient determined from `{{$json.Email}}` extracted from Airtable record.  
  - Credentials: Brevo API credentials required.  
  - Inputs: Items from Loop Over Items.  
  - Outputs: Email sending response to Edit Fields node.  
  - Edge Cases: API auth errors, invalid email addresses, template ID changes, rate limits.

- **Edit Fields**  
  - Type: Set  
  - Role: Extracts `messageId` from Brevo’s email sending response for tracking purposes.  
  - Configuration: Uses regex to parse `messageId` from the `messageId` string matching `<([^@]+)@`.  
  - Inputs: Output of Send an email node.  
  - Outputs: Enriched data passed to Create Interaction node.  
  - Edge Cases: If `messageId` format changes or missing, regex may fail.

- **Create Interaction**  
  - Type: Airtable (Create)  
  - Role: Inserts a new record into Airtable's "INTERACTION" table to log the email sending event.  
  - Configuration: Fields include current date/time, campaign name with timestamp, "Email" as media, linked company record, and Brevo message ID.  
  - Inputs: Output from Edit Fields.  
  - Outputs: Leads to Wait node for pacing.  
  - Edge Cases: Airtable API errors, incorrect field mapping, missing linked records.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 3 seconds between sending emails to avoid rate limits or overload.  
  - Configuration: Fixed 3-second wait.  
  - Inputs: Output from Create Interaction.  
  - Outputs: Loops back to Loop Over Items to continue batch processing.  
  - Edge Cases: Workflow delays accumulate if many contacts; may require adjustment.

- **Sticky Note (Email Sending Workflow Prerequisites)**  
  - Provides essential prerequisites and notes:  
    - Airtable & Brevo credentials required.  
    - CRM tables involved (Company, Interaction, Campaign).  
    - Brevo email template preconfigured.  
    - Search filter conditions for contacts.

---

#### 2.2 Email Event Tracking Workflow

**Overview:**  
This block handles incoming webhook events from Brevo about email delivery, opens, clicks, and unsubscribes. It processes the event type, searches corresponding interaction records in Airtable, updates relevant date fields (delivery, open, click), and updates company opt-in status on unsubscribe events.

**Nodes Involved:**  
- Webhook (HTTP POST)  
- Edit Fields1 (Set)  
- Search Interaction (Airtable)  
- Switch (Routing by event type)  
- Update Delivred (Airtable)  
- Update Opened (Airtable)  
- Update Clicked (Airtable)  
- Search Company2 (Airtable)  
- Update Company (Airtable)  
- Sticky Notes (Event Tracking Prerequisites and status labels)

**Node Details:**

- **Webhook**  
  - Type: Webhook (HTTP POST)  
  - Role: Receives email event notifications from Brevo.  
  - Configuration: POST method; path named "TO-BE-MODIFIED" (to be customized).  
  - Inputs: Incoming HTTP requests from Brevo.  
  - Outputs: Passes raw event JSON to Edit Fields1.  
  - Edge Cases: Must be publicly accessible; path must match Brevo configuration; handle possible malformed payloads.

- **Edit Fields1**  
  - Type: Set  
  - Role: Extracts the internal message ID from the webhook payload’s `body['message-id']` using regex `<([^@]+)@`.  
  - Inputs: Webhook JSON.  
  - Outputs: Provides cleaned `message-id` for Airtable search.  
  - Edge Cases: Similar to Edit Fields; regex failure if format changes.

- **Search Interaction**  
  - Type: Airtable (Search)  
  - Role: Finds the interaction record in Airtable matching the extracted Brevo message ID.  
  - Configuration: Filter by formula `{Brevo Id} = '<message-id>'`.  
  - Inputs: Output of Edit Fields1.  
  - Outputs: Passes found record to Switch node.  
  - Edge Cases: No match found if message ID missing or outdated; Airtable API issues.

- **Switch**  
  - Type: Switch  
  - Role: Routes workflow based on event type from webhook JSON `body.event`.  
  - Configuration: Outputs for "delivered", "opened", "click", and fallback ("extra").  
  - Inputs: Search Interaction output.  
  - Outputs: Routes to corresponding update nodes or company search for unsubscribes.  
  - Edge Cases: Unexpected event types; fallback output handles these.

- **Update Delivred**  
  - Type: Airtable (Update)  
  - Role: Updates the "Email Délivré" field with current timestamp for the matched interaction record.  
  - Configuration: Uses `{Brevo Id}` to match record; sets delivery date to now.  
  - Inputs: Switch output for "delivered".  
  - Edge Cases: Airtable write failures; missing record IDs.

- **Update Opened**  
  - Type: Airtable (Update)  
  - Role: Updates the "Email Ouvert" field with current timestamp for the matched interaction record.  
  - Configuration: Similar to Update Delivred but for "opened" events.  
  - Inputs: Switch output for "opened".  
  - Edge Cases: Same as above.

- **Update Clicked**  
  - Type: Airtable (Update)  
  - Role: Updates the "Email Cliqué" field with current timestamp for the matched interaction record.  
  - Configuration: Similar update for "click" events.  
  - Inputs: Switch output for "click".  
  - Edge Cases: Same as above.

- **Search Company2**  
  - Type: Airtable (Search)  
  - Role: On unsubscribe or fallback events, finds the company record linked to the interaction by record ID.  
  - Configuration: Filters by record ID stored in interaction’s "Société" field.  
  - Inputs: Switch fallback output.  
  - Outputs: Passes company record to Update Company.  
  - Edge Cases: Missing linked company; API errors.

- **Update Company**  
  - Type: Airtable (Update)  
  - Role: Updates the company’s "Opt-in" field to `false` reflecting unsubscribe request.  
  - Configuration: Matches by record ID; sets Opt-in to false.  
  - Inputs: Output from Search Company2.  
  - Edge Cases: Write failures; ensure correct record ID mapping.

- **Sticky Notes (Event Tracking Workflow Prerequisites and Labels)**  
  - Notes prerequisites: Brevo webhook configuration needed to call this n8n webhook and enable email event sending.  
  - Labels on nodes update states for clarity: Delivered, Opened, Clicked, Unsubscribed Request.

---

### 3. Summary Table

| Node Name                         | Node Type            | Functional Role                                   | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                          |
|----------------------------------|----------------------|-------------------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Entry point to manually start email sending     | None                          | Search Company               | ## Email Sending Workflow 🛠️ Prerequisites: Credentials (Airtable, Brevo), CRM tables, Template     |
| Search Company                   | Airtable (Search)    | Retrieve contacts filtered for campaign & opt-in| When clicking ‘Execute workflow’ | Loop Over Items             | ## Email Sending Workflow 🛠️ Prerequisites: Credentials (Airtable, Brevo), CRM tables, Template     |
| Loop Over Items                  | SplitInBatches       | Batch processing of contacts for sending emails | Search Company                | Send an email with existing Template (secondary output), also loops from Wait node | ## Email Sending Workflow 🛠️ Prerequisites: Credentials (Airtable, Brevo), CRM tables, Template     |
| Send an email with an existing Template | SendInBlue (Brevo) | Send marketing emails using Brevo template       | Loop Over Items (secondary)   | Edit Fields                 | ## Email Sending Workflow 🛠️ Prerequisites: Credentials (Airtable, Brevo), CRM tables, Template     |
| Edit Fields                     | Set                  | Extract messageId from Brevo response            | Send an email with existing Template | Create Interaction       | ## Email Sending Workflow 🛠️ Prerequisites: Credentials (Airtable, Brevo), CRM tables, Template     |
| Create Interaction              | Airtable (Create)    | Create interaction record in Airtable            | Edit Fields                   | Wait                        | ## Email Sending Workflow 🛠️ Prerequisites: Credentials (Airtable, Brevo), CRM tables, Template     |
| Wait                           | Wait                 | Pause between sends to throttle email sending    | Create Interaction            | Loop Over Items             | ## Email Sending Workflow 🛠️ Prerequisites: Credentials (Airtable, Brevo), CRM tables, Template     |
| Sticky Note                    | Sticky Note          | Email sending prerequisites note                  | None                         | None                       | ## Email Sending Workflow 🛠️ Prerequisites: Credentials (Airtable, Brevo), CRM tables, Template     |
| Webhook                        | Webhook              | Receives Brevo email event webhooks               | None                         | Edit Fields1                | ## Email Event Tracking Workflow 🛠️ Prerequisites: Brevo webhook config and event sending enabled   |
| Edit Fields1                   | Set                  | Extract message-id from webhook payload           | Webhook                      | Search Interaction          | ## Email Event Tracking Workflow 🛠️ Prerequisites: Brevo webhook config and event sending enabled   |
| Search Interaction             | Airtable (Search)    | Find interaction record matching message-id       | Edit Fields1                 | Switch                     | ## Email Event Tracking Workflow 🛠️ Prerequisites: Brevo webhook config and event sending enabled   |
| Switch                        | Switch               | Route event types: delivered, opened, click, unsubscribe | Search Interaction          | Update Delivred, Update Opened, Update Clicked, Search Company2 | ## Email Event Tracking Workflow 🛠️ Prerequisites: Brevo webhook config and event sending enabled   |
| Update Delivred               | Airtable (Update)    | Update delivery timestamp in interaction record   | Switch (delivered)           | None                       | ### ✉️ Delivred Email                                                                              |
| Update Opened                | Airtable (Update)    | Update open timestamp in interaction record       | Switch (opened)              | None                       | ### ✉️ Opened Email                                                                               |
| Update Clicked               | Airtable (Update)    | Update click timestamp in interaction record      | Switch (click)               | None                       | ### ✉️ Clicked Email                                                                              |
| Search Company2              | Airtable (Search)    | Find company linked to interaction for unsubscribe | Switch (extra/fallback)      | Update Company              | ### ✉️ Unsubscribed Request Update of Opt-in Status                                              |
| Update Company              | Airtable (Update)    | Update company Opt-in status to false on unsubscribe | Search Company2             | None                       | ### ✉️ Unsubscribed Request Update of Opt-in Status                                              |
| Sticky Note1                 | Sticky Note          | Email event tracking prerequisites note            | None                         | None                       | ## Email Event Tracking Workflow 🛠️ Prerequisites: Brevo webhook config and event sending enabled   |
| Sticky Note2                 | Sticky Note          | Label for delivered email update block             | None                         | None                       | ### ✉️ Delivred Email                                                                            |
| Sticky Note3                 | Sticky Note          | Label for clicked email update block                | None                         | None                       | ### ✉️ Clicked Email                                                                             |
| Sticky Note4                 | Sticky Note          | Label for opened email update block                 | None                         | None                       | ### ✉️ Opened Email                                                                              |
| Sticky Note5                 | Sticky Note          | Label for unsubscribe request block                  | None                         | None                       | ### ✉️ Unsubscribed Request Update of Opt-in Status                                            |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a new workflow in n8n.

---

**Step 2:** Build the Email Sending Workflow

1. **Add Manual Trigger node** named `When clicking ‘Execute workflow’`.  
   - No configuration needed.

2. **Add Airtable node** named `Search Company`.  
   - Operation: Search.  
   - Base: Select your Airtable base containing companies (e.g. "Prospection").  
   - Table: The "COMPANY" table.  
   - Filter Formula: `AND({Opt-in} = 1, {Campaign} = 1, {Checked Email} = 1)`.  
   - Credentials: Configure Airtable API credentials.

3. Connect `When clicking ‘Execute workflow’` output to `Search Company` input.

4. **Add SplitInBatches node** named `Loop Over Items`.  
   - No specific options set (default batch size).  
   - Connect `Search Company` output to `Loop Over Items` input.

5. **Add SendInBlue node** named `Send an email with an existing Template`.  
   - Operation: `sendTemplate`.  
   - Template ID: `2` (ensure this matches your Brevo template).  
   - Recipients: Set to expression `{{$json.Email}}` (maps to Airtable email field).  
   - Credentials: Configure Brevo API credentials.  
   - Connect `Loop Over Items` secondary output to this node.

6. **Add Set node** named `Edit Fields`.  
   - Add assignment:  
     - Name: `message_Id`  
     - Value: Use expression to extract messageId: `{{$json.messageId.match(/<([^@]+)@/)[1]}}`.  
   - Connect `Send an email with an existing Template` output to `Edit Fields`.

7. **Add Airtable node** named `Create Interaction`.  
   - Operation: Create record.  
   - Base: Airtable base for interactions ("Prospection").  
   - Table: "INTERACTION".  
   - Columns/Fields to set:  
     - Date: Current timestamp in ISO format (expression: `{{$now.format('yyyy-MM-dd\'T\'HH:mm:ss.000\'Z\'')}}`).  
     - Name: `"Campaign {{$now}}"` (string concatenation with timestamp).  
     - Media: `"Email"` (static string).  
     - COMPANY: Link to company record using `=["{{$('Search Company').item.json.Name}}"]`.  
     - Brevo Id: Use expression `{{$json.message_Id}}`.  
   - Enable typecasting.  
   - Credentials: Airtable API configured.  
   - Connect `Edit Fields` output to `Create Interaction`.

8. **Add Wait node** named `Wait`.  
   - Set wait time to 3 seconds.  
   - Connect `Create Interaction` output to `Wait`.

9. Connect `Wait` output back to `Loop Over Items` input to continue batch processing.

---

**Step 3:** Build the Email Event Tracking Workflow

1. **Add Webhook node** named `Webhook`.  
   - HTTP Method: POST.  
   - Path: Customize to a unique path, e.g., `email-events`.  
   - No authentication configured (or add if needed).  

2. **Add Set node** named `Edit Fields1`.  
   - Add assignment:  
     - Name: `body['message-id']`  
     - Value: Extract message ID with regex: `{{$json.body['message-id'].match(/<([^@]+)@/)[1]}}`.  
   - Connect `Webhook` output to `Edit Fields1`.

3. **Add Airtable node** named `Search Interaction`.  
   - Operation: Search.  
   - Base: Airtable base for interactions.  
   - Table: "INTERACTION".  
   - Filter Formula: `=({Brevo Id} = '{{ $json.body['message-id'] }}')`.  
   - Credentials: Airtable API credentials.  
   - Connect `Edit Fields1` output to `Search Interaction`.

4. **Add Switch node** named `Switch`.  
   - Add rules based on event type (`{{$json.body.event}}`):  
     - Output "Delivred Email" for `delivered`.  
     - Output "Opened Email" for `opened`.  
     - Output "Clicked Email" for `click`.  
     - Fallback output "extra" for others (e.g., unsubscribe).  
   - Connect `Search Interaction` output to `Switch`.

5. **Add Airtable node** named `Update Delivred`.  
   - Operation: Update.  
   - Base and Table: same as "INTERACTION".  
   - Matching Column: `Brevo Id`.  
   - Fields to update:  
     - `Email Délivré` with current timestamp (expression: `{{$now.format('yyyy-MM-dd\'T\'HH:mm:ss.000\'Z\'')}}`).  
   - Credentials: Airtable API credentials.  
   - Connect `Switch` output for "Delivred Email" to this node.

6. **Add Airtable node** named `Update Opened`.  
   - Same as above but update `Email Ouvert`.  
   - Connect `Switch` "Opened Email" output to this node.

7. **Add Airtable node** named `Update Clicked`.  
   - Same as above but update `Email Cliqué`.  
   - Connect `Switch` "Clicked Email" output to this node.

8. **Add Airtable node** named `Search Company2`.  
   - Operation: Search.  
   - Base and Table: "COMPANY".  
   - Filter Formula: `RECORD_ID() = "{{ $json['Société'][0] }}"` (using linked company record from interaction).  
   - Credentials: Airtable API credentials.  
   - Connect `Switch` fallback output ("extra") to this node.

9. **Add Airtable node** named `Update Company`.  
   - Operation: Update.  
   - Base and Table: "COMPANY".  
   - Matching Column: Record ID.  
   - Fields to update:  
     - `Opt-in` set to `false` (boolean) to reflect unsubscribe.  
   - Credentials: Airtable API credentials.  
   - Connect `Search Company2` output to `Update Company`.

---

**Step 4:** Add Sticky Notes for Documentation

- Add sticky notes near each major block to document prerequisites and clarify node functions, matching the original notes content.

---

**Step 5:** Validate Credentials and API Limits

- Ensure Airtable API keys have permission to all relevant bases and tables.  
- Brevo API key must have send and webhook permissions.  
- Brevo webhook must be configured to send events to your n8n webhook URL with the correct path.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow requires Airtable CRM setup with tables: COMPANY, INTERACTION, and Campaign.                                                                    | Airtable base schema prerequisite.                                                                       |
| Brevo email template must be pre-created and template ID configured (default used is ID 2).                                                              | Brevo SendInBlue template management.                                                                    |
| Brevo webhook must be configured to call the n8n webhook endpoint for event tracking (delivery, open, click, unsubscribe).                               | Brevo webhook setup documentation.                                                                       |
| Regex extraction of message-id assumes format `<uniqueId@domain>`; changes in Brevo response format require updating regex accordingly.                  | Message-id parsing in Set nodes.                                                                          |
| Wait node uses a 3-second delay to comply with sending rate limits; adjust if sending volume or API limits change.                                        | Rate limiting considerations.                                                                             |
| Airtable field mappings depend on exact field names and types; ensure schema matches or update field keys accordingly.                                    | Airtable field configuration.                                                                             |
| n8n workflow path for webhook must be set in Brevo dashboard to receive event notifications.                                                             | Webhook path naming consistency important.                                                               |
| For unsubscribe events, Opt-in field is set to false to reflect user opt-out, enabling compliance with GDPR and email marketing best practices.           | GDPR and compliance note.                                                                                  |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.