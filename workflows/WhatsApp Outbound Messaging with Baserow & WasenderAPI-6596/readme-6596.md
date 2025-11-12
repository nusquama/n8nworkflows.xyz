WhatsApp Outbound Messaging with Baserow & WasenderAPI

https://n8nworkflows.xyz/workflows/whatsapp-outbound-messaging-with-baserow---wasenderapi-6596


# WhatsApp Outbound Messaging with Baserow & WasenderAPI

### 1. Workflow Overview

This workflow enables automated outbound WhatsApp messaging using n8n, integrating Baserow as the message data source and WasenderAPI as the WhatsApp sending service. It listens for status updates in a Baserow table (‘Messages’), specifically for records marked ‘Sent’, then sends the corresponding WhatsApp message via WasenderAPI and logs the outbound event back into Baserow. This setup centralizes message management and tracking while automating communication.

Logical blocks:

- **1.1 Input Reception:** Receives webhook calls from Baserow when a message status changes.
- **1.2 Message Status Filtering:** Checks if the message’s status is ‘Sent’ to proceed.
- **1.3 Outbound Message Sending:** Uses WasenderAPI to send the WhatsApp message.
- **1.4 Outbound Logging:** Updates Baserow to confirm the message was sent.
- **1.5 Fallback Handling:** Handles cases where status is not ‘Sent’.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for incoming POST webhook requests from Baserow automation when a message record updates, triggering the workflow.

- **Nodes Involved:**  
  - `Webhook: Baserow Outbound Trigger`

- **Node Details:**  
  - **Type:** Webhook  
  - **Configuration:**  
    - HTTP Method: POST  
    - Path: Customizable (user-defined string)  
    - Authentication: Header-based API key (`whatsapp_hook` credential)  
  - **Expressions/Variables:**  
    - Receives JSON payload containing Baserow message record data, including fields like `Status`, `WhatsApp Number (Linked Contact)`, and `Message Content`.  
  - **Inputs:** None (starting point)  
  - **Outputs:** To the `Filter: Message Status 'Sent'` node  
  - **Version Requirements:** n8n webhook node version 2+  
  - **Potential Failure Modes:**  
    - Authentication failure if header missing/incorrect  
    - Payload format changes breaking downstream expressions  
  - **Sub-workflows:** None  

#### 2.2 Message Status Filtering

- **Overview:**  
  Evaluates the incoming message record’s `Status` field to determine if it equals ‘Sent’. Only proceeds with outbound messaging if true; otherwise, routes to fallback.

- **Nodes Involved:**  
  - `Filter: Message Status 'Sent'`

- **Node Details:**  
  - **Type:** Switch node  
  - **Configuration:**  
    - Condition: `$json.body.items[0].Status.value === "Sent"`  
    - Outputs:  
      - `SENT` if true  
      - `extra` fallback if false or unmatched  
  - **Expressions:** Used to access nested JSON fields from webhook data  
  - **Inputs:** From `Webhook: Baserow Outbound Trigger`  
  - **Outputs:**  
    - On `SENT` output: to `WasenderAPI: Send Outbound Message`  
    - On fallback output: to `fallback` node  
  - **Version Requirements:** Switch node version supporting condition type validation 3.2+  
  - **Potential Failure Modes:**  
    - Missing `Status` field or unexpected format may cause expression failure  
    - Case sensitivity could cause mismatches  
  - **Sub-workflows:** None  

#### 2.3 Outbound Message Sending

- **Overview:**  
  Sends the WhatsApp message via WasenderAPI using details from the Baserow record.

- **Nodes Involved:**  
  - `WasenderAPI: Send Outbound Message`

- **Node Details:**  
  - **Type:** HTTP Request node  
  - **Configuration:**  
    - URL: `https://wasenderapi.com/api/send-message`  
    - Method: POST  
    - Body Parameters:  
      - `to`: Extracted from `WhatsApp Number (Linked Contact)` linked field in Baserow record  
      - `text`: Extracted from `Message Content` field  
    - Headers: Authorization Bearer token (user’s WasenderAPI token must be inserted)  
  - **Expressions:**  
    - `to`: `={{ $json.body.items[0]['WhatsApp Number (Linked Contact)'][0].value }}`  
    - `text`: `={{ $json.body.items[0]['Message Content'] }}`  
  - **Inputs:** From `Filter: Message Status 'Sent'` (SENT output)  
  - **Outputs:** To `Baserow: Confirm Outbound Log` node  
  - **Version Requirements:** HTTP Request node version 4.2+ to support dynamic body and headers  
  - **Potential Failure Modes:**  
    - API authentication failure (invalid or expired token)  
    - API rate limits or service downtime  
    - Missing or malformed phone numbers or message text causing API errors  
  - **Sub-workflows:** None  

#### 2.4 Outbound Logging

- **Overview:**  
  Updates the originating Baserow message record to confirm the outbound message was sent, logging status, timestamp, and message ID returned by WasenderAPI.

- **Nodes Involved:**  
  - `Baserow: Confirm Outbound Log`

- **Node Details:**  
  - **Type:** Baserow node for updating a record  
  - **Configuration:**  
    - Operation: Update  
    - Database ID: 264981 (user-specific, must match target Baserow database)  
    - Table ID: 622533 (target ‘Messages’ table)  
    - Row ID: Derived dynamically from webhook data (`$('Webhook: Baserow Outbound Trigger').item.json.body.items[0].id`)  
    - Fields updated:  
      - Status field set to ‘Outbound’  
      - Timestamp field set to current local time in `nl-NL` locale  
      - Message ID field set from WasenderAPI response (`$json.data.msgId`)  
  - **Credentials:** Baserow API credentials configured (user-specific)  
  - **Inputs:** From `WasenderAPI: Send Outbound Message`  
  - **Outputs:** None (end of chain)  
  - **Version Requirements:** Baserow node version 1+  
  - **Potential Failure Modes:**  
    - API authentication errors  
    - Row ID mismatch or missing causing update failure  
    - Field ID mismatches due to Baserow schema changes  
  - **Sub-workflows:** None  

#### 2.5 Fallback Handling

- **Overview:**  
  A no-operation node that acts as the fallback for messages whose status is not ‘Sent’, effectively terminating or ignoring these cases.

- **Nodes Involved:**  
  - `fallback`

- **Node Details:**  
  - **Type:** NoOp node (does nothing)  
  - **Configuration:** Default, no parameters  
  - **Inputs:** From `Filter: Message Status 'Sent'` fallback output  
  - **Outputs:** None  
  - **Version Requirements:** None  
  - **Potential Failure Modes:** None  

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                   | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                  |
|-------------------------------|--------------------|---------------------------------|---------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note                   | Sticky Note        | Documentation / Instructions     | None                            | None                           | ## Master Your Outbound WhatsApp: Baserow & n8n Automation... See full content for detailed setup instructions and links. |
| Webhook: Baserow Outbound Trigger | Webhook            | Receives Baserow webhook trigger | None                            | Filter: Message Status 'Sent'  |                                                                                                              |
| Filter: Message Status 'Sent' | Switch             | Filters messages by 'Sent' status | Webhook: Baserow Outbound Trigger | WasenderAPI: Send Outbound Message, fallback |                                                                                                              |
| WasenderAPI: Send Outbound Message | HTTP Request       | Sends WhatsApp message via API  | Filter: Message Status 'Sent' (SENT output) | Baserow: Confirm Outbound Log  |                                                                                                              |
| Baserow: Confirm Outbound Log | Baserow            | Logs outbound message in Baserow | WasenderAPI: Send Outbound Message | None                           |                                                                                                              |
| fallback                     | NoOp               | Handles non-'Sent' statuses      | Filter: Message Status 'Sent' (fallback output) | None                           |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Sticky Note node** (optional but recommended):  
   - Content: Paste the provided master instructions and setup guide for future reference.

3. **Create a Webhook node:**  
   - Name: `Webhook: Baserow Outbound Trigger`  
   - HTTP Method: POST  
   - Path: Set a custom string, e.g., `whatsapp-outbound`  
   - Authentication: Header Authentication  
     - Create a new HTTP Header Auth credential (e.g., `whatsapp_hook`) with a secure API key  
   - Save credentials in n8n credentials manager.

4. **Add a Switch node:**  
   - Name: `Filter: Message Status 'Sent'`  
   - Condition:  
     - Type: String equals  
     - Expression: `{{$json.body.items[0].Status.value}}` equals `"Sent"`  
   - Define two outputs:  
     - Output 1: Rename to `SENT`  
     - Output 2: Fallback output (default)  

5. **Add an HTTP Request node:**  
   - Name: `WasenderAPI: Send Outbound Message`  
   - HTTP Method: POST  
   - URL: `https://wasenderapi.com/api/send-message`  
   - Authentication: None (use header parameter)  
   - Headers:  
     - `Authorization: Bearer <Your Token>` (create credential for WasenderAPI token)  
   - Body Parameters (JSON or form-data):  
     - `to`: Expression: `{{$json.body.items[0]['WhatsApp Number (Linked Contact)'][0].value}}`  
     - `text`: Expression: `{{$json.body.items[0]['Message Content']}}`

6. **Add a Baserow node:**  
   - Name: `Baserow: Confirm Outbound Log`  
   - Operation: Update  
   - Database ID: Your Baserow database ID (e.g., 264981)  
   - Table ID: Your Messages table ID (e.g., 622533)  
   - Row ID: Expression: `{{$node["Webhook: Baserow Outbound Trigger"].json.body.items[0].id}}`  
   - Fields to update:  
     - Status: “Outbound” (hardcoded)  
     - Timestamp: Expression: `{{$now.setLocale("nl-NL").toLocal()}}`  
     - Message ID: Expression: `{{$json.data.msgId}}` (from WasenderAPI response)  
   - Link the Baserow API credential in n8n.

7. **Add a NoOp node:**  
   - Name: `fallback`  
   - No parameters required.

8. **Connect nodes:**  
   - Webhook → Switch  
   - Switch `SENT` output → WasenderAPI node  
   - Switch fallback output → fallback node  
   - WasenderAPI node → Baserow update node

9. **Configure credentials:**  
   - Create and configure HTTP Header Auth credential for the webhook authentication.  
   - Create and configure WasenderAPI token credential for WhatsApp sending.  
   - Create and configure Baserow API credential for database updates.

10. **Deploy and test:**  
    - Copy the webhook URL (displayed in the Webhook node)  
    - Configure Baserow automation to POST to this URL when message `Status` changes to `Sent`.  
    - Verify WhatsApp messages send and logs update correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Baserow public grids for Contacts and Messages tables are provided for quick setup and testing.                                                                                                                                           | Contacts: https://baserow.io/public/grid/a5iWkAQpu8QljUlgwgm_pour_Au5BKd3mtkfu-B6N7Y <br> Messages: https://baserow.io/public/grid/0H22XZitFDWnrVNnKwBfiI7M6XX5CugHrXHEzdCY4xY                                  |
| Optional: Use the Baserow Message Form for inputting message data easily.                                                                                                                                                                 | https://baserow.io/form/B2TUPV0S_Fx3PKyNiKOQR4YAdo77RnvAxZyMw8jN7Uc                                                                                                           |
| Ensure the WasenderAPI token and Baserow API tokens are kept secure and stored only in n8n Credentials.                                                                                                                                   | Security best practice                                                                                                                                                          |
| Maintain the exact node flow and connection order to ensure the workflow functions correctly.                                                                                                                                             | Critical for correct operation                                                                                                                                                |
| This workflow assumes Baserow fields have specific IDs and names; verify and update field/table IDs to match your environment before deploying.                                                                                          | Customization note                                                                                                                                                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This content strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is lawful and public.