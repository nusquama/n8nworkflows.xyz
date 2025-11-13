Auto-Clean Gmail Spam Folder - Monthly using Gmail API

https://n8nworkflows.xyz/workflows/auto-clean-gmail-spam-folder---monthly-using-gmail-api-6122


# Auto-Clean Gmail Spam Folder - Monthly using Gmail API

### 1. Workflow Overview

This workflow automates the monthly cleaning of the Gmail Spam folder by permanently deleting all emails labeled as spam. It is designed for Gmail users who want to maintain a clean inbox and optimize storage by removing spam emails regularly without manual intervention.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger:** Automatically initiates the workflow on the first day of every month at 6 AM.
- **1.2 Fetch Spam Emails:** Retrieves all emails currently in the Gmail Spam folder using the Gmail API.
- **1.3 Throttling Pause:** Inserts a short delay to prevent API rate limiting and ensure stability in email fetching.
- **1.4 Delete Spam Emails:** Iteratively deletes each fetched spam email permanently from the Gmail account.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow automatically on a monthly schedule, specifically on the first day of each month at 6 AM, ensuring regular execution without manual start.

- **Nodes Involved:**  
  - Monthly Trigger (1st Day)  

- **Node Details:**  
  - **Monthly Trigger (1st Day)**  
    - **Type:** Schedule Trigger  
    - **Technical Role:** Start node that initiates the workflow based on a time schedule.  
    - **Configuration:** Set to trigger monthly, at hour 6, on the first day.  
    - **Expressions/Variables:** None.  
    - **Input connections:** None (start node).  
    - **Output connections:** Connects to "Fetch SPAM Emails (Gmail)".  
    - **Version Requirements:** Uses n8n schedule trigger v1.2, which supports monthly intervals.  
    - **Potential Failure Modes:** Scheduling misconfiguration, node runtime failure.  
    - **Sticky Note Content:** "Starts the workflow automatically on the 1st of every month to perform email cleanup."

#### 1.2 Fetch Spam Emails

- **Overview:**  
  This block accesses the Gmail API to retrieve all emails labeled as SPAM in the authenticated Gmail account. It aims to collect the entire spam folder contents for deletion.

- **Nodes Involved:**  
  - Fetch SPAM Emails (Gmail)

- **Node Details:**  
  - **Fetch SPAM Emails (Gmail)**  
    - **Type:** Gmail Node (API integration)  
    - **Technical Role:** Retrieves all emails with the "SPAM" label using Gmail OAuth2 credentials.  
    - **Configuration:**  
      - Operation: "getAll" to fetch all messages.  
      - Label Filter: "SPAM" folder (labelIds set to "SPAM").  
      - Return All: True (fetches all emails in spam folder).  
    - **Expressions/Variables:** None dynamic; static label filter.  
    - **Input connections:** From "Monthly Trigger (1st Day)".  
    - **Output connections:** To "Pause Before Deletion (5s)".  
    - **Version Requirements:** Uses Gmail node v2.1 for updated API methods.  
    - **Potential Failure Modes:**  
      - OAuth token expiration or invalid credentials.  
      - Gmail API rate limits or quota exceeded.  
      - Network errors.  
      - Empty spam folder returns empty dataset (should be handled gracefully).  
    - **Sticky Note Content:** "Uses Gmail API to retrieve all emails in the SPAM folder for the authenticated Gmail account."

#### 1.3 Throttling Pause

- **Overview:**  
  This block adds a fixed 5-second wait before proceeding to deletion, mitigating possible issues with Gmail API rate limiting and ensuring all emails are fetched properly before deletion begins.

- **Nodes Involved:**  
  - Pause Before Deletion (5s)

- **Node Details:**  
  - **Pause Before Deletion (5s)**  
    - **Type:** Wait node  
    - **Technical Role:** Delays workflow execution for a fixed duration.  
    - **Configuration:** Default wait time (5 seconds). No expressions used.  
    - **Input connections:** From "Fetch SPAM Emails (Gmail)".  
    - **Output connections:** To "Delete Fetched SPAM Emails".  
    - **Version Requirements:** Wait node v1.1 or later.  
    - **Potential Failure Modes:** Minimal; mostly timeouts or workflow interruptions.  
    - **Sticky Note Content:** "Introduces a short delay to avoid hitting Gmail’s API rate limits or ensure all emails are fetched properly."

#### 1.4 Delete Spam Emails

- **Overview:**  
  This block iterates over each fetched spam email and permanently deletes it from the Gmail account, cleaning the spam folder and freeing storage.

- **Nodes Involved:**  
  - Delete Fetched SPAM Emails

- **Node Details:**  
  - **Delete Fetched SPAM Emails**  
    - **Type:** Gmail Node (API integration)  
    - **Technical Role:** Deletes individual emails by their ID using Gmail API.  
    - **Configuration:**  
      - Operation: "delete" message.  
      - Message ID: Set dynamically using an expression referencing the email’s id property (={{ $json.id }}).  
    - **Expressions/Variables:** Uses expression to fetch current email’s message ID from input data.  
    - **Input connections:** From "Pause Before Deletion (5s)".  
    - **Output connections:** None (terminal node).  
    - **Version Requirements:** Uses Gmail node v2.1.  
    - **Potential Failure Modes:**  
      - Invalid or missing message IDs.  
      - OAuth token expiry or permission issues.  
      - Gmail API limits or quota exceeded.  
      - Partial failures if some emails cannot be deleted.  
    - **Sticky Note Content:** "Permanently deletes all emails retrieved from the SPAM folder to free up Gmail space and improve email hygiene."

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                     | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                   |
|---------------------------|----------------------|-----------------------------------|------------------------|----------------------------|-----------------------------------------------------------------------------------------------|
| Monthly Trigger (1st Day)  | Schedule Trigger     | Starts workflow monthly           | —                      | Fetch SPAM Emails (Gmail)  | Starts the workflow automatically on the 1st of every month to perform email cleanup.         |
| Fetch SPAM Emails (Gmail)  | Gmail API Node       | Fetches all emails in Spam folder | Monthly Trigger (1st Day) | Pause Before Deletion (5s)  | Uses Gmail API to retrieve all emails in the SPAM folder for the authenticated Gmail account.  |
| Pause Before Deletion (5s) | Wait Node           | Adds 5-second delay                | Fetch SPAM Emails (Gmail) | Delete Fetched SPAM Emails | Introduces a short delay to avoid hitting Gmail’s API rate limits or ensure all emails fetched.|
| Delete Fetched SPAM Emails | Gmail API Node       | Deletes fetched spam emails       | Pause Before Deletion (5s) | —                          | Permanently deletes all emails retrieved from the SPAM folder to free up Gmail space.          |
| Sticky Note               | Sticky Note          | Comment                          | —                      | —                          | Starts the workflow automatically on the 1st of every month to perform email cleanup.         |
| Sticky Note1              | Sticky Note          | Comment                          | —                      | —                          | Uses Gmail API to retrieve all emails in the SPAM folder for the authenticated Gmail account.  |
| Sticky Note2              | Sticky Note          | Comment                          | —                      | —                          | Introduces a short delay to avoid hitting Gmail’s API rate limits or ensure all emails fetched.|
| Sticky Note3              | Sticky Note          | Comment                          | —                      | —                          | Permanently deletes all emails retrieved from the SPAM folder to free up Gmail space.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it "Auto-Clean Gmail Spam Folder Monthly with Gmail API".**

2. **Add the "Schedule Trigger" node:**
   - Set node name: `Monthly Trigger (1st Day)`
   - Set trigger rule to repeat monthly:  
     - Interval: Months  
     - Trigger at hour: 6 (6 AM)  
     - The day defaults to the first of the month.
   - No credentials required.
   - Connect this node as the start of the workflow.

3. **Add the "Gmail" node to fetch spam emails:**
   - Set node name: `Fetch SPAM Emails (Gmail)`
   - Set operation: `getAll`
   - Under filters, set labelIds to `SPAM` to target the Spam folder.
   - Enable "Return All" to fetch all spam emails.
   - Connect input from `Monthly Trigger (1st Day)`.
   - Set credentials: Use a Gmail OAuth2 credential with read and delete permissions configured for the Gmail account to be cleaned.
   - Use node version 2.1 or later.

4. **Add a "Wait" node for throttling:**
   - Set node name: `Pause Before Deletion (5s)`
   - Set wait time to 5 seconds (default).
   - Connect input from `Fetch SPAM Emails (Gmail)`.

5. **Add another "Gmail" node to delete emails:**
   - Set node name: `Delete Fetched SPAM Emails`
   - Set operation: `delete`
   - Set messageId parameter dynamically with the expression: `={{ $json.id }}`
     - This assumes the input data has an `id` property for each email.
   - Connect input from `Pause Before Deletion (5s)`.
   - Set the same Gmail OAuth2 credential as for fetching emails.
   - Use node version 2.1 or later.

6. **Ensure node connections:**
   - `Monthly Trigger (1st Day)` → `Fetch SPAM Emails (Gmail)` → `Pause Before Deletion (5s)` → `Delete Fetched SPAM Emails`

7. **Add sticky notes (optional) for documentation:**
   - Near `Monthly Trigger (1st Day)`: "Starts the workflow automatically on the 1st of every month to perform email cleanup."
   - Near `Fetch SPAM Emails (Gmail)`: "Uses Gmail API to retrieve all emails in the SPAM folder for the authenticated Gmail account."
   - Near `Pause Before Deletion (5s)`: "Introduces a short delay to avoid hitting Gmail’s API rate limits or ensure all emails are fetched properly."
   - Near `Delete Fetched SPAM Emails`: "Permanently deletes all emails retrieved from the SPAM folder to free up Gmail space and improve email hygiene."

8. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                          |
|----------------------------------------------------------------------------------------------|----------------------------------------------------------|
| Gmail API OAuth2 credentials must have scopes for reading and deleting emails (e.g., `https://www.googleapis.com/auth/gmail.modify`). | Credential setup for Gmail API integration.              |
| Gmail API has rate limits; the 5-second wait node helps mitigate potential quota errors.      | Gmail API documentation: https://developers.google.com/gmail/api/guides/quotas |
| Workflow runs monthly at 6 AM by default; adjust schedule trigger if a different time is needed.| n8n Schedule Trigger documentation.                      |
| This workflow permanently deletes emails from Spam; ensure compliance with user data policies.| Data management best practices and compliance guidelines.|

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.