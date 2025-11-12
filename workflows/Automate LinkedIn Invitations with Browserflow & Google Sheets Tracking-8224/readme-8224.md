Automate LinkedIn Invitations with Browserflow & Google Sheets Tracking

https://n8nworkflows.xyz/workflows/automate-linkedin-invitations-with-browserflow---google-sheets-tracking-8224


# Automate LinkedIn Invitations with Browserflow & Google Sheets Tracking

### 1. Workflow Overview

This workflow automates the process of sending personalized LinkedIn connection invitations using Browserflow, while managing and tracking contacts and invitation statuses in a Google Sheet. It targets users such as recruiters, sales teams, and networking professionals who want to efficiently and systematically grow their LinkedIn network by avoiding duplicate invitations and maintaining progressive tracking.

The workflow is logically divided into these key blocks:

- **1.1 Initialization and Data Loading:** Starting the workflow manually, loading configuration and contacts from Google Sheets.
- **1.2 Data Filtering and Limiting:** Filtering contacts to only those not yet invited, and limiting the number of invitations per execution.
- **1.3 Invitation Processing Loop:** Iterating over each contact to verify connection status using Browserflow.
- **1.4 Connection Status Evaluation:** Using conditional logic to decide if an invitation should be sent based on connection or pending invitation status.
- **1.5 Sending Invitations:** Sending personalized LinkedIn invitations with customized messages via Browserflow.
- **1.6 Updating Google Sheet:** Updating contact records post-invitation with statuses such as “pending”, “connected”, or marking as invited.
- **1.7 Wait / Throttling:** Introducing delays between batch processing to mimic human-like behavior and respect rate limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Data Loading

- **Overview:** This block starts the workflow manually, sets the Google Sheet ID configuration, and loads the contact list from the Google Sheet.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’ (Manual Trigger)
  - settings (Set node)
  - contacts (Google Sheets Read)

- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Initiates the workflow manually.
    - Configuration: Default, no parameters.
    - Inputs: None.
    - Outputs: Triggers next node.
    - Edge cases: N/A.

  - **settings**
    - Type: Set
    - Role: Defines Google Sheet ID as a string parameter.
    - Configuration: Assigns `googlesheetid` variable with placeholder string `<input your google sheet ID here>`.
    - Inputs: Manual Trigger node.
    - Outputs: Passes config to next node.
    - Edge cases: Workflow will fail if Google Sheet ID is not replaced or is invalid.

  - **contacts**
    - Type: Google Sheets (Read)
    - Role: Fetches contacts data from the specified Google Sheet and sheet tab.
    - Configuration: Uses `googlesheetid` from settings node, sheet tab “contacts” (by ID).
    - Credentials: Google Sheets OAuth2.
    - Inputs: Settings node.
    - Outputs: JSON array of contact objects including LinkedIn URLs.
    - Edge cases: Google API errors, invalid sheet ID, empty sheets.

#### 2.2 Data Filtering and Limiting

- **Overview:** Filters contacts to those who have empty “invitation” field (not yet invited), limits the number of processed contacts per execution to 5.
- **Nodes Involved:** 
  - finallist (Filter)
  - Limit (Limit)

- **Node Details:**

  - **finallist**
    - Type: Filter
    - Role: Filters contacts where the “invitation” field is empty.
    - Configuration: Condition on `$json.invitation` being empty string.
    - Inputs: contacts node.
    - Outputs: Filtered contacts with no invitation status.
    - Edge cases: Empty or missing “invitation” fields may cause false negatives.

  - **Limit**
    - Type: Limit
    - Role: Restricts number of contacts processed to 5 per run.
    - Configuration: `maxItems` set to 5.
    - Inputs: finallist node.
    - Outputs: Limited subset of contacts.
    - Edge cases: Limits batch size; can be adjusted for rate control.

#### 2.3 Invitation Processing Loop

- **Overview:** Processes each selected contact one by one (splitting into batches of 1), and checks if the LinkedIn profile is already connected or pending connection using Browserflow.
- **Nodes Involved:**
  - Loop Over Items (SplitInBatches)
  - isnotconnected (Browserflow)

- **Node Details:**

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Controls batch size of 1 to process one contact at a time.
    - Configuration: Default batch size 1.
    - Inputs: Limit node.
    - Outputs: Single contact per iteration.
    - Edge cases: Ensures sequential processing to avoid rate limits.

  - **isnotconnected**
    - Type: Browserflow
    - Role: Checks LinkedIn connection status for the contact URL.
    - Configuration: Reads `linkedinUrl` from current item JSON `url`.
    - Credentials: Browserflow API.
    - Inputs: Loop Over Items.
    - Outputs: JSON including booleans `is_connection` and `is_pending`.
    - Edge cases: Browserflow API errors, invalid URLs, network issues.

#### 2.4 Connection Status Evaluation

- **Overview:** Determines if the contact is neither connected nor has a pending invitation; if true, prepares to send an invitation; otherwise, skips sending.
- **Nodes Involved:**
  - If (Conditional)

- **Node Details:**

  - **If**
    - Type: If (Boolean conditions)
    - Role: Evaluates two conditions:
      - `$json.is_connection` is false
      - `$json.is_pending` is false
    - Configuration: Both conditions combined with AND.
    - Inputs: isnotconnected node.
    - Outputs:
      - True branch: Contact not connected or pending → proceed to message composition.
      - False branch: Contact already connected or pending → skip invitation.
    - Edge cases: Boolean conversion errors if fields missing.

#### 2.5 Sending Invitations

- **Overview:** For contacts eligible for invitation, composes a personalized message and sends the connection invite via Browserflow.
- **Nodes Involved:**
  - message (Set)
  - sendinvite (Browserflow)

- **Node Details:**

  - **message**
    - Type: Set
    - Role: Constructs personalized invitation message using contact’s first name.
    - Configuration: Multiline string with embedded expression `{{ $('contacts').item.json.firstname }}` to insert the first name.
    - Inputs: If node (true branch).
    - Outputs: Passes message string for sending.
    - Edge cases: Missing first name may lead to awkward messages.

  - **sendinvite**
    - Type: Browserflow
    - Role: Sends LinkedIn connection invitation with message.
    - Configuration:
      - `operation`: sendConnectionInvite
      - `linkedinUrl`: from contact JSON.
      - `addMessage`: true (message included)
      - `message`: from message node.
    - Credentials: Browserflow API.
    - Inputs: message node.
    - Outputs: Invitation result.
    - Edge cases: API errors, LinkedIn rate limits, message sending failures.

#### 2.6 Updating Google Sheet

- **Overview:** Updates the contact’s invitation status after sending the invitation or if already connected/pending. Maintains tracking within the Google Sheet.
- **Nodes Involved:**
  - updateinvitation (Google Sheets Update)
  - alreadyconnectedorpending (NoOp)
  - updatecontact (Google Sheets Update)

- **Node Details:**

  - **updateinvitation**
    - Type: Google Sheets
    - Role: Marks contact as invited by setting `invitation` = "Y".
    - Configuration:
      - Updates row by matching `row_number`.
      - Sets invitation field to "Y".
    - Inputs: sendinvite node.
    - Outputs: Passes to wait node.
    - Credentials: Google Sheets OAuth2.
    - Edge cases: Sheet update failures, row mismatch.

  - **alreadyconnectedorpending**
    - Type: NoOp
    - Role: Pass-through node for contacts that are already connected or have pending invitations.
    - Inputs: If node (false branch).
    - Outputs: Passes to updatecontact node.
    - Edge cases: None.

  - **updatecontact**
    - Type: Google Sheets
    - Role: Updates the `invitation` field to either "pending" or "connected" based on Browserflow status.
    - Configuration:
      - Sets invitation field to "pending" if `is_pending` true, otherwise "connected".
      - Matches rows by `row_number`.
    - Inputs: alreadyconnectedorpending node.
    - Credentials: Google Sheets OAuth2.
    - Edge cases: Update failures, concurrency issues.

#### 2.7 Wait / Throttling

- **Overview:** Introduces a 60-second delay between processing batches to mimic human behavior and avoid LinkedIn limits.
- **Nodes Involved:**
  - Wait

- **Node Details:**

  - **Wait**
    - Type: Wait
    - Role: Pauses workflow for 60 seconds.
    - Configuration: `amount` = 60 seconds.
    - Inputs: updateinvitation and updatecontact nodes converge here.
    - Outputs: Loops back to Loop Over Items to process next contact.
    - Edge cases: Long waits might delay processing; too short waits may trigger LinkedIn restrictions.

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                                  | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                                                    |
|--------------------------------|---------------------------|-------------------------------------------------|----------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger            | Starts the workflow manually                     | None                             | settings                      |                                                                                                                               |
| settings                      | Set                       | Sets Google Sheet ID configuration              | When clicking ‘Execute workflow’ | contacts                      | Set the Google Sheet ID you want to use                                                                                       |
| contacts                     | Google Sheets Read         | Loads contacts from Google Sheet                  | settings                        | finallist                     | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing) |
| finallist                    | Filter                    | Filters contacts with empty "invitation" field  | contacts                       | Limit                        | Only contacts with empty "invitation"                                                                                        |
| Limit                       | Limit                     | Limits number of contacts processed per execution| finallist                      | Loop Over Items               | Only for test / debug                                                                                                           |
| Loop Over Items              | SplitInBatches            | Processes contacts one at a time                  | Limit                         | isnotconnected, (empty branch) |                                                                                                                               |
| isnotconnected              | Browserflow                | Checks if contact is connected or pending        | Loop Over Items               | If                           |                                                                                                                               |
| If                          | If                        | Determines invitation eligibility                 | isnotconnected               | message (true), alreadyconnectedorpending (false) |                                                                                                                               |
| message                     | Set                       | Builds personalized invitation message            | If (true)                    | sendinvite                   | Update the invitation message in this node !                                                                                   |
| sendinvite                  | Browserflow                | Sends LinkedIn invitation                          | message                      | updateinvitation             | Flag the contact as invited. invitation = "Y"                                                                                  |
| updateinvitation            | Google Sheets Update       | Updates contact as invited ("Y")                  | sendinvite                   | Wait                        | Flag the contact as "connected" or "pending"                                                                                   |
| alreadyconnectedorpending   | NoOp                      | Pass-through for contacts already connected/pending | If (false)                  | updatecontact                |                                                                                                                               |
| updatecontact               | Google Sheets Update       | Updates contact invitation status ("connected"/"pending") | alreadyconnectedorpending  | Wait                        |                                                                                                                               |
| Wait                       | Wait                      | Pauses workflow 60 seconds between iterations     | updateinvitation, updatecontact | Loop Over Items              | Update the Wait amount here                                                                                                    |
| Sticky Note                 | Sticky Note               | Documentation and overview                        | None                          | None                        | # Manage LinkedIn Invitations with Browserflow ... [Full content in node]                                                     |
| Sticky Note6                | Sticky Note               | Reminder to set Google Sheet ID                   | None                          | None                        | Set the Google Sheet ID you want to use                                                                                       |
| Sticky Note7                | Sticky Note               | Reminder about filtering contacts                  | None                          | None                        | Only contacts with empty "invitation"                                                                                        |
| Sticky Note8                | Sticky Note               | Link to Google Sheet template                      | None                          | None                        | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing) |
| Sticky Note9                | Sticky Note               | Reminder to update invitation message              | None                          | None                        | Update the invitation message in this node !                                                                                   |
| Sticky Note10               | Sticky Note               | Reminder about flagging contact as invited         | None                          | None                        | Flag the contact as invited. invitation = "Y"                                                                                  |
| Sticky Note11               | Sticky Note               | Note about Limit node use for testing/debugging    | None                          | None                        | Only for test / debug                                                                                                           |
| Sticky Note12               | Sticky Note               | Reminder about updating invitation status          | None                          | None                        | Flag the contact as "connected" or "pending"                                                                                   |
| Sticky Note13               | Sticky Note               | Reminder to adjust Wait node timing                 | None                          | None                        | Update the Wait amount here                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Node Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Add Set Node for Settings:**  
   - Node Type: Set  
   - Add field `googlesheetid` (string) with placeholder value `<input your google sheet ID here>`.  
   - Connect Manual Trigger → Set Node.

3. **Add Google Sheets Read Node to Load Contacts:**  
   - Node Type: Google Sheets  
   - Operation: Read  
   - Document ID: Use expression `={{ $json.googlesheetid }}` from settings node.  
   - Sheet Name: `"contacts"` (by sheet name or ID).  
   - Credentials: Configure Google Sheets OAuth2 with access to your sheet.  
   - Connect Set Node → Google Sheets Read node.

4. **Add Filter Node to Select Contacts Without Invitation:**  
   - Node Type: Filter  
   - Condition: `$json.invitation` is empty string (`""`).  
   - Connect Google Sheets Read → Filter.

5. **Add Limit Node to Restrict Number of Contacts:**  
   - Node Type: Limit  
   - Parameter: `maxItems` = 5 (adjust as needed).  
   - Connect Filter → Limit.

6. **Add SplitInBatches Node for Sequential Processing:**  
   - Node Type: SplitInBatches  
   - Batch Size: Default 1 (process one contact at a time).  
   - Connect Limit → SplitInBatches.

7. **Add Browserflow Node to Check Connection Status:**  
   - Node Type: Browserflow  
   - Operation: Custom flow that queries LinkedIn profile connection status.  
   - Parameter: `linkedinUrl` set to `={{ $('contacts').item.json.url }}`.  
   - Credentials: Configure Browserflow API credentials.  
   - Connect SplitInBatches → Browserflow node.

8. **Add If Node to Check Connection & Pending Status:**  
   - Node Type: If  
   - Conditions:  
     - `$json.is_connection` == false  
     - `$json.is_pending` == false  
   - Both conditions combined with AND.  
   - Connect Browserflow → If node.

9. **Add Set Node to Compose Personalized Message:**  
   - Node Type: Set  
   - Field: `message` (string)  
   - Value (multiline with expression):  
     ```
     Dear {{ $('contacts').item.json.firstname }},
     Right now building a framework to help revenu teams to drive efficient outbound strategies. This n8n automation is part of the solution. Happy to connect and get your feedback !
     The n8n About team.
     ```  
   - Connect If (true) → Set node.

10. **Add Browserflow Node to Send Invitation:**  
    - Node Type: Browserflow  
    - Operation: sendConnectionInvite  
    - Parameters:  
      - `linkedinUrl` = `={{ $('contacts').item.json.url }}`  
      - `addMessage` = true  
      - `message` = `={{ $json.message }}`  
    - Credentials: Browserflow API.  
    - Connect Set message → Browserflow send invitation.

11. **Add Google Sheets Update Node to Mark Invitation Sent:**  
    - Node Type: Google Sheets  
    - Operation: Update  
    - Document ID: `={{ $('settings').item.json.googlesheetid }}`  
    - Sheet Name: `"contacts"`  
    - Columns to update:  
      - `invitation` = "Y"  
      - `row_number` = `={{ $('contacts').item.json.row_number }}` (to match correct row)  
    - Credentials: Google Sheets OAuth2.  
    - Connect Browserflow send invitation → Update node.

12. **Add NoOp Node for Already Connected or Pending Contacts:**  
    - Node Type: NoOp  
    - Connect If (false) → NoOp node.

13. **Add Google Sheets Update Node to Mark Pending/Connected:**  
    - Node Type: Google Sheets  
    - Operation: Update  
    - Document ID and Sheet Name as above.  
    - Columns to update:  
      - `invitation` = `={{ $('isnotconnected').item.json.is_pending ? "pending" : "connected" }}`  
      - `row_number` = `={{ $('contacts').item.json.row_number }}`  
    - Credentials: Google Sheets OAuth2.  
    - Connect NoOp → Update node.

14. **Add Wait Node to Pause Between Each Contact:**  
    - Node Type: Wait  
    - Parameter: 60 seconds (adjustable).  
    - Connect both Google Sheets Update nodes (invitation sent and connection/pending update) → Wait node.

15. **Connect Wait Node back to SplitInBatches:**  
    - To continue processing next batch after wait.

16. **Add Sticky Notes for Documentation and Instructions:**  
    - Add as per workflow for user guidance and references, including links to Google Sheet template and notes on configuration.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                               | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow automates personalized LinkedIn outreach while maintaining human-like behavior to respect LinkedIn’s usage policies. It is recommended to use responsibly to avoid account restrictions.                                                                                                         | Sticky Note overview                                                                                     |
| Google Sheet Template to start with contacts list: [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing)                                                                                                                          | Sticky Note8                                                                                             |
| Browserflow account required with available credits for LinkedIn automation: [Browserflow Signup](https://browserflow.io/)                                                                                                                                                                                 | Sticky Note overview                                                                                     |
| n8n tested on version 1.109.1 for compatibility.                                                                                                                                                                                                                                                           | Sticky Note overview                                                                                     |
| Support & Community for workflow assistance: LinkedIn discussion [here](https://www.linkedin.com/posts/n8n-about_n8n-browserflow-activity-7368758690025320448-zupZ/), direct LinkedIn contact [here](https://www.linkedin.com/in/stephaneheckel/), and n8n Community Forum: https://community.n8n.io/              | Sticky Note overview                                                                                     |
| Adjust the Wait node timing based on your LinkedIn usage pattern to avoid triggering anti-spam mechanisms.                                                                                                                                                                                                  | Sticky Note13                                                                                            |
| Update the invitation message in the dedicated Set node to customize outreach language as needed.                                                                                                                                                                                                           | Sticky Note9                                                                                             |
| Limit node is set to 5 items by default for testing; increase cautiously for production use.                                                                                                                                                                                                                | Sticky Note11                                                                                            |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow designed with ethical and legal compliance. It contains no illegal, offensive, or protected content. All data processed is legitimate and publicly accessible.