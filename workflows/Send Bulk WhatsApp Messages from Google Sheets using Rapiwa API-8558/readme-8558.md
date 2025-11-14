Send Bulk WhatsApp Messages from Google Sheets using Rapiwa API

https://n8nworkflows.xyz/workflows/send-bulk-whatsapp-messages-from-google-sheets-using-rapiwa-api-8558


# Send Bulk WhatsApp Messages from Google Sheets using Rapiwa API

### 1. Workflow Overview

This workflow automates sending bulk WhatsApp messages using the Rapiwa API, integrating directly with Google Sheets for message and contact management. It is designed for users who want to send personalized WhatsApp campaigns without using the official WhatsApp Business API, leveraging their own WhatsApp number via Rapiwa's unofficial integration.

The workflow logically divides into these blocks:

- **1.1 Trigger & Data Fetching:** Automatically triggers every 5 minutes and fetches pending messages from a Google Sheet.
- **1.2 Message Processing & Validation:** Limits batch size, loops through each message, cleans WhatsApp numbers, and verifies their validity via Rapiwa API.
- **1.3 Conditional Sending & Status Update:** Based on verification, sends messages or marks as unverified, updates Google Sheets accordingly.
- **1.4 Rate Limiting & Looping:** Waits a few seconds between sends to control rate, then continues processing next items.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Fetching

- **Overview:** This block initiates the workflow on a fixed schedule and retrieves only those rows from the Google Sheet where the message status is "pending" for processing.
- **Nodes Involved:**
  - Trigger Every 5 Minute
  - Fetch All Pending Queries for Messaging
  - Limit

- **Node Details:**

  - **Trigger Every 5 Minute**
    - *Type:* Schedule Trigger
    - *Role:* Automatically starts the workflow every 5 minutes.
    - *Configuration:* Interval set to every 5 minutes.
    - *Inputs:* None (trigger node).
    - *Outputs:* Starts the chain leading to data fetch.
    - *Failure Potential:* Scheduler misconfiguration or n8n service downtime.
  
  - **Fetch All Pending Queries for Messaging**
    - *Type:* Google Sheets
    - *Role:* Reads from Google Sheet named "Message Queue" filtering rows where `Status = pending`.
    - *Configuration:* Uses OAuth2 credentials for Google Sheets, fetches rows from a specific document and sheet, applying filter on `Status` column.
    - *Input:* Trigger node output.
    - *Output:* Rows with pending messages.
    - *Failure Potential:* Google API quota limits, authentication expiration, incorrect sheet or filter setup.
  
  - **Limit**
    - *Type:* Limit
    - *Role:* Caps message processing to 60 items per workflow run.
    - *Configuration:* `maxItems` set to 60.
    - *Input:* Rows from Google Sheets node.
    - *Output:* Limited set of rows forwarded to next step.
    - *Failure Potential:* None significant; misconfiguration may limit throughput too much.

#### 2.2 Message Processing & Validation

- **Overview:** Processes each message individually by cleaning the WhatsApp number and verifying it via Rapiwa API to ensure messages are sent only to valid WhatsApp accounts.
- **Nodes Involved:**
  - Loop Over Items (Split In Batches)
  - Clean WhatsApp Number (Code)
  - Check valid whatsapp number Using Rapiwa1 (HTTP Request)
  - If (Conditional Check)

- **Node Details:**

  - **Loop Over Items**
    - *Type:* SplitInBatches
    - *Role:* Iterates over each message row one by one.
    - *Configuration:* Default batching.
    - *Input:* Limited rows from previous step.
    - *Output:* Single message data per iteration.
    - *Failure Potential:* Large batch sizes may cause memory or timeout issues.
  
  - **Clean WhatsApp Number**
    - *Type:* Code (JavaScript)
    - *Role:* Removes all non-digit characters from the WhatsApp number to ensure it is in valid numeric format.
    - *Configuration:* Custom JS script that replaces non-digits using regex.
    - *Input:* Single message from Loop Over Items.
    - *Output:* Message data with cleaned `WhatsApp No`.
    - *Failure Potential:* Unexpected data types or missing fields causing script errors.
  
  - **Check valid whatsapp number Using Rapiwa1**
    - *Type:* HTTP Request
    - *Role:* Sends POST request to Rapiwa API endpoint `/api/verify-whatsapp` to verify if the cleaned number is an active WhatsApp account.
    - *Configuration:*
      - URL: `https://app.rapiwa.com/api/verify-whatsapp`
      - Authentication: Bearer Token (Rapiwa API key)
      - Body: JSON with `number` field set to cleaned WhatsApp No.
    - *Input:* Output from cleaning node.
    - *Output:* Verification response containing `data.exists` boolean.
    - *Failure Potential:* Network errors, invalid API key, rate limiting by Rapiwa, malformed request.
  
  - **If**
    - *Type:* Conditional (If)
    - *Role:* Checks if `data.exists == true` to branch workflow.
    - *Configuration:* Expression checking boolean `data.exists` in JSON response.
    - *Input:* Verification API response.
    - *Outputs:* 
      - True branch: Verified number path.
      - False branch: Unverified number path.
    - *Failure Potential:* Expression evaluation errors if response format changes.

#### 2.3 Conditional Sending & Status Update

- **Overview:** Depending on verification, sends WhatsApp messages using Rapiwa API for verified numbers and updates the Google Sheet status accordingly for both verified and unverified numbers.
- **Nodes Involved:**
  - Send Message Using Rapiwa (HTTP Request)
  - Change State of Rows in Verified & Sent (Google Sheets Update)
  - Change State of Rows in Unverified & Not Sent (Google Sheets Update)

- **Node Details:**

  - **Send Message Using Rapiwa**
    - *Type:* HTTP Request
    - *Role:* Sends WhatsApp message via Rapiwa’s API endpoint.
    - *Configuration:*
      - Method: POST
      - URL: `https://app.rapiwa.com/api/send-message`
      - Authentication: Bearer Token (Rapiwa API key)
      - Query Parameters:
        - `number`: The cleaned WhatsApp number
        - `message`: Message text from sheet
        - `imageUrl`: Optional image URL from sheet
        - `message_type`: "text"
    - *Input:* True branch from If node (verified numbers).
    - *Output:* API response after sending message.
    - *Failure Potential:* API rate limits, invalid message content, network issues, auth failures.
  
  - **Change State of Rows in Verified & Sent**
    - *Type:* Google Sheets
    - *Role:* Updates the original message row with `Status` = "sent" and `Verification` = "verified".
    - *Configuration:* Uses row number from original data to update correct row.
    - *Input:* Output of Send Message node.
    - *Output:* Confirmation of update.
    - *Failure Potential:* Google Sheets API errors, concurrency issues if multiple runs overlap.
  
  - **Change State of Rows in Unverified & Not Sent**
    - *Type:* Google Sheets
    - *Role:* Updates the message row with `Status` = "not sent" and `Verification` = "unverified" if number is invalid.
    - *Configuration:* Uses row number from original data to update correct row.
    - *Input:* False branch from If node.
    - *Output:* Confirmation of update.
    - *Failure Potential:* Same as above.

#### 2.4 Rate Limiting & Looping

- **Overview:** Adds a delay between sending each message to avoid triggering WhatsApp blocking mechanisms or API rate limits, then continues processing remaining messages.
- **Nodes Involved:**
  - Wait
  - Loop Over Items (continues looping)

- **Node Details:**

  - **Wait**
    - *Type:* Wait
    - *Role:* Pauses workflow for a short duration before processing the next message.
    - *Configuration:* Default pause (no explicit duration set in JSON, but sticky note suggests ~5 seconds).
    - *Input:* After Google Sheets update nodes.
    - *Output:* Connects back to Loop Over Items to process next batch item.
    - *Failure Potential:* Unlikely; misconfiguration could cause long delays or zero delays.

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                              | Input Node(s)                        | Output Node(s)                         | Sticky Note                                                                                         |
|-------------------------------------|---------------------|----------------------------------------------|------------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------|
| Trigger Every 5 Minute               | Schedule Trigger    | Starts workflow every 5 minutes               | -                                  | Fetch All Pending Queries for Messaging | ## Trigger Every 5 Minutes: Starts workflow automatically every 5 minutes. Runs continuously.    |
| Fetch All Pending Queries for Messaging | Google Sheets       | Fetch rows with Status = pending               | Trigger Every 5 Minute              | Limit                                 | ## Fetch All Pending Queries: Pulls only pending messages from Google Sheet.                       |
| Limit                               | Limit               | Limits messages processed per run to 60       | Fetch All Pending Queries for Messaging | Loop Over Items                     | ## Limit: Prevents overload by limiting to 60 messages per run.                                   |
| Loop Over Items                     | SplitInBatches      | Processes messages one by one                   | Limit                             | Clean WhatsApp Number                  | ## Loop Over Items: Processes each message separately and in order.                               |
| Clean WhatsApp Number               | Code                | Cleans WhatsApp numbers (removes non-digits)   | Loop Over Items                   | Check valid whatsapp number Using Rapiwa1 | ## Clean and Verify WhatsApp Numbers: Clean numbers for valid format before verification.       |
| Check valid whatsapp number Using Rapiwa1 | HTTP Request        | Verifies WhatsApp number is active via API    | Clean WhatsApp Number              | If                                    | ## Check valid whatsapp number Using Rapiwa1: Validates numbers to avoid wasted credits.          |
| If                                  | If                  | Checks verification result to branch workflow | Check valid whatsapp number Using Rapiwa1 | Send Message Using Rapiwa (true branch), Change State of Rows in Unverified & Not Sent (false branch) | ## Conditional Logic: Sends message if verified; marks unverified otherwise.                      |
| Send Message Using Rapiwa           | HTTP Request        | Sends WhatsApp message via Rapiwa API          | If (true branch)                  | Change State of Rows in Verified & Sent | ## Send Message via HTTP Request: Core sending node connecting to WhatsApp through Rapiwa API.    |
| Change State of Rows in Verified & Sent | Google Sheets       | Updates Google Sheet row to status "sent" and "verified" | Send Message Using Rapiwa           | Wait                                  | ## Change State: Marks messages as sent and verified in sheet to avoid duplicates.                |
| Change State of Rows in Unverified & Not Sent | Google Sheets       | Updates Google Sheet row to status "not sent" and "unverified" | If (false branch)                 | Wait                                  | ## Change State: Marks messages as not sent and unverified for failed validation cases.           |
| Wait                                | Wait                | Pauses workflow to avoid rate limits           | Change State of Rows in Verified & Sent, Change State of Rows in Unverified & Not Sent | Loop Over Items                      | ## Wait: Prevents sending too many messages too quickly, avoiding blocking.                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Trigger Every 5 Minute` node**  
   - Type: Schedule Trigger  
   - Set interval to every 5 minutes.

2. **Add `Fetch All Pending Queries for Messaging` node**  
   - Type: Google Sheets  
   - Connect it to the trigger.  
   - Configure credentials with Google Sheets OAuth2.  
   - Select the Google Sheet document and the worksheet "Message Queue".  
   - Add filter: `Status` column equals `pending`.  
   - Retrieve all matching rows.

3. **Add `Limit` node**  
   - Type: Limit  
   - Connect to Google Sheets node.  
   - Set maxItems to 60 to limit messages per run.

4. **Add `Loop Over Items` node**  
   - Type: SplitInBatches  
   - Connect to Limit node.  
   - Default batch size (process one item at a time).

5. **Add `Clean WhatsApp Number` node**  
   - Type: Code (JavaScript)  
   - Connect to Loop Over Items.  
   - Paste the following JS code to clean the WhatsApp No field:  
     ```js
     const items = $input.all();
     const updatedItems = items.map((item) => {
       const waNo = item?.json["WhatsApp No"];
       const waNoStr = typeof waNo === 'string' ? waNo : (waNo !== undefined && waNo !== null ? String(waNo) : "");
       const cleanedNumber = waNoStr.replace(/\D/g, "");
       item.json["WhatsApp No"] = cleanedNumber;
       return item;
     });
     return updatedItems;
     ```

6. **Add `Check valid whatsapp number Using Rapiwa1` node**  
   - Type: HTTP Request  
   - Connect to Clean WhatsApp Number node.  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Authentication: HTTP Bearer Auth with Rapiwa API token.  
   - Body Parameters (JSON):  
     - `number`: Use expression `{{$json["WhatsApp No"]}}`.

7. **Add `If` node**  
   - Connect to Check valid whatsapp number node.  
   - Condition: Check if `{{$json["data"]["exists"]}} == true` (boolean true).  
   - This will split the workflow into two branches: verified and unverified.

8. **Add `Send Message Using Rapiwa` node**  
   - Type: HTTP Request  
   - Connect True output of If node.  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Authentication: HTTP Bearer Auth with Rapiwa API token.  
   - Query Parameters:  
     - `number`: expression `={{ $json["WhatsApp No"] }}`  
     - `message`: expression `={{ $json["Message"] }}`  
     - `imageUrl`: expression `={{ $json["Image URL"] }}` (optional)  
     - `message_type`: set to `"text"`  

9. **Add `Change State of Rows in Verified & Sent` node**  
   - Type: Google Sheets (Update)  
   - Connect to Send Message node.  
   - Configure to update the original row in "Message Queue" sheet:  
     - Set `Status` = "sent"  
     - Set `Verification` = "verified"  
   - Use `row_number` from the message row to identify correct row.

10. **Add `Change State of Rows in Unverified & Not Sent` node**  
    - Type: Google Sheets (Update)  
    - Connect False output of If node.  
    - Configure to update the original row in "Message Queue" sheet:  
      - Set `Status` = "not sent"  
      - Set `Verification` = "unverified"  
    - Use `row_number` as above.

11. **Add `Wait` node**  
    - Type: Wait  
    - Connect both Google Sheets update nodes to it.  
    - Set duration to about 5 seconds (or desired delay).

12. **Connect Wait node back to Loop Over Items node**  
    - This loops and processes the next message.

13. **Verify all credentials:**  
    - Google Sheets OAuth2 credentials configured and authorized.  
    - Rapiwa Bearer Token configured correctly in HTTP Request nodes.

14. **Test the workflow**  
    - Ensure your Google Sheet has a "Message Queue" tab with columns as required:  
      - `SL`, `WhatsApp No`, `Name`, `Message`, `Image URL`, `Verification`, `Status`  
    - Populate with test data including some pending messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow enables sending bulk WhatsApp messages using your own WhatsApp number through the unofficial Rapiwa API, avoiding costs and restrictions of the official WhatsApp Business API.                                                                                                                                                                            | General overview                                                                                 |
| Sample Google Sheet structure for message queue: includes columns `SL`, `WhatsApp No`, `Name`, `Message`, `Image URL`, `Verification`, and `Status`.                                                                                                                                                                                                                       | [Sample Sheet](https://docs.google.com/spreadsheets/d/1Ui4TzzI-Gq-bsEsrZELwW1Kyddw0IU9L1wxlHikktqw/edit?usp=sharing) |
| Google Sheets API access and OAuth2 credentials are required to fetch and update the message queue.                                                                                                                                                                                                                                                                         | Setup requirement                                                                              |
| Rapiwa API token must be valid and associated with a WhatsApp number authorized on Rapiwa platform.                                                                                                                                                                                                                                                                         | [Rapiwa Dashboard](https://app.rapiwa.com/login), [Official Website](https://rapiwa.com), [Rapiwa Docs](https://docs.rapiwa.com) |
| The workflow includes rate limiting via the Wait node to prevent WhatsApp number blocking due to rapid message sending.                                                                                                                                                                                                                                                    | Operational best practice                                                                       |
| The verification step prevents wasting API credits and sending messages to invalid or inactive WhatsApp numbers.                                                                                                                                                                                                                                                           | Cost-saving and efficiency measure                                                            |
| The workflow can be customized for batch sizes, message content, and additional metadata updates in Google Sheets.                                                                                                                                                                                                                                                        | Extensibility note                                                                             |

---

This documentation provides a detailed, stepwise understanding and reproduction guide for the "Send Bulk WhatsApp Messages from Google Sheets using Rapiwa API" n8n workflow. It covers every node, configuration rationale, and error considerations to facilitate both human and automated manipulation or extension.