Forward Filtered Gmail Notifications to Telegram Chat

https://n8nworkflows.xyz/workflows/forward-filtered-gmail-notifications-to-telegram-chat-3301


# Forward Filtered Gmail Notifications to Telegram Chat

### 1. Workflow Overview

This workflow automates the forwarding of selected incoming Gmail emails to a Telegram chat based on subject keyword filtering. It is designed for real-time notification scenarios such as security alerts or urgent communications, enabling users to monitor important emails directly via Telegram while filtering out irrelevant messages.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures incoming Gmail messages using a Gmail Trigger node.
- **1.2 Conditional Filtering:** Evaluates the email subject to determine if it contains predefined keywords, using an IF node.
- **1.3 Notification Dispatch:** Sends a formatted message containing email details to a specified Telegram chat via a Telegram node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new incoming emails in the connected Gmail account and triggers the workflow whenever a new message arrives.

- **Nodes Involved:**  
  - Incoming Email Monitor (Gmail Trigger)

- **Node Details:**

  - **Incoming Email Monitor**  
    - **Type & Role:** Gmail Trigger node; initiates workflow execution on new email reception.  
    - **Configuration:**  
      - Trigger Event: "Message Received" (default for Gmail Trigger).  
      - Polling frequency: Every minute (configured under `pollTimes`).  
      - No additional filters applied at this stage (empty filters object).  
    - **Expressions/Variables:** Outputs the full email JSON including fields like `From`, `Subject`, and `snippet`.  
    - **Input/Output:** No input connections; outputs to the "Email Validation Check" node.  
    - **Version:** 1.2 (supports OAuth2 credentials).  
    - **Potential Failures:**  
      - Authentication errors if OAuth2 token expires or is invalid.  
      - API rate limits or quota exceeded errors from Gmail.  
      - Network timeouts or connectivity issues.  
    - **Credentials:** Uses OAuth2 credentials for Gmail API access.  
    - **Sub-workflow:** None.

#### 2.2 Conditional Filtering

- **Overview:**  
  This block filters incoming emails based on whether their subject contains specific keywords ("Urgent" or "Server Down"). Only emails matching these criteria proceed to notification.

- **Nodes Involved:**  
  - Email Validation Check (IF Node)

- **Node Details:**

  - **Email Validation Check**  
    - **Type & Role:** IF node; performs conditional logic to filter emails.  
    - **Configuration:**  
      - Condition Type: String contains or equals (version 2 conditions).  
      - Checks if the `Subject` field contains "Urgent" OR equals "Server Down".  
      - Case sensitivity: Enabled (case sensitive).  
      - Loose type validation enabled to handle missing or unexpected data gracefully.  
    - **Expressions/Variables:**  
      - Left value: `{{$json.Subject}}` (email subject).  
      - Right values: "Urgent" (contains), "Server Down" (equals).  
    - **Input/Output:**  
      - Input from "Incoming Email Monitor".  
      - True output connected to "Send Telegram Message".  
      - False output not connected (workflow stops for non-matching emails).  
    - **Version:** 2.2 (latest IF node features).  
    - **Potential Failures:**  
      - Expression evaluation errors if `Subject` is missing or malformed.  
      - Logical errors if keywords need updating or localization.  
    - **Sub-workflow:** None.

#### 2.3 Notification Dispatch

- **Overview:**  
  This block sends a formatted notification message to a Telegram chat, containing sender, subject, and a snippet of the email body.

- **Nodes Involved:**  
  - Send Telegram Message (Telegram node)

- **Node Details:**

  - **Send Telegram Message**  
    - **Type & Role:** Telegram node; sends messages via Telegram Bot API.  
    - **Configuration:**  
      - Action: "Send Message".  
      - Chat ID: Set to the target Telegram chat (personal or group).  
      - Message Text: Uses template expression to include:  
        ```
        From : {{ $json.From }}
        Subject :{{ $json.Subject }}
        Message : {{ $json.snippet }}
        ```  
      - Additional Fields: Attribution disabled to avoid appending "Sent via n8n".  
    - **Expressions/Variables:** Uses JSON fields from the Gmail Trigger node output.  
    - **Input/Output:**  
      - Input from "Email Validation Check" (true branch).  
      - No output connections (end of workflow).  
    - **Version:** 1.2.  
    - **Potential Failures:**  
      - Authentication errors if Telegram Bot Token is invalid or revoked.  
      - Invalid Chat ID causing message delivery failure.  
      - Telegram API rate limits or downtime.  
      - Message formatting errors if variables are missing.  
    - **Credentials:** Uses Telegram Bot Token credentials.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role           | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                  |
|-----------------------|--------------------|--------------------------|------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note           | Sticky Note        | Documentation header     | —                      | —                       | # Forward Filtered Gmail Notifications to Telegram Chat                                                     |
| Sticky Note1          | Sticky Note        | Workflow description     | —                      | —                       | ## Description : This n8n workflow automatically forwards incoming Gmail emails to a Telegram chat only if the email subject contains specific keywords (like "Urgent" or "Server Down"). The workflow extracts key details such as the sender, subject, and message body, and sends them as a formatted message to a specified Telegram chat. This is useful for real-time notifications, security alerts, or monitoring important emails directly from Telegram — filtering out unnecessary emails. |
| Incoming Email Monitor | Gmail Trigger      | Input Reception          | —                      | Email Validation Check   |                                                                                                              |
| Email Validation Check | IF                 | Conditional Filtering    | Incoming Email Monitor  | Send Telegram Message    |                                                                                                              |
| Send Telegram Message  | Telegram           | Notification Dispatch    | Email Validation Check  | —                       |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Forward Filtered Gmail Notifications to Telegram Chat".

2. **Add a Gmail Trigger node:**
   - Name: `Incoming Email Monitor`.
   - Set Trigger Event to "Message Received".
   - Configure polling to run every minute.
   - Authenticate with your Google account using OAuth2 credentials (ensure Gmail API is enabled).
   - Leave filters empty to capture all incoming emails.
   - Save and test the node to confirm connectivity.

3. **Add an IF node:**
   - Name: `Email Validation Check`.
   - Connect the output of `Incoming Email Monitor` to this node.
   - Configure conditions as follows:
     - Condition 1: Check if `Subject` contains "Urgent" (case sensitive).
     - Condition 2: Check if `Subject` equals "Server Down" (case sensitive).
     - Combine conditions with OR logic.
   - Enable loose type validation to handle missing fields gracefully.

4. **Add a Telegram node:**
   - Name: `Send Telegram Message`.
   - Connect the "true" output of `Email Validation Check` to this node.
   - Set Action to "Send Message".
   - Authenticate using your Telegram Bot Token credentials.
   - Set the Chat ID to your target Telegram chat (personal or group).
   - Configure the message text with the following template:  
     ```
     From : {{ $json.From }}
     Subject :{{ $json.Subject }}
     Message : {{ $json.snippet }}
     ```
   - Disable attribution (do not append "Sent via n8n").
   - Save and test the node by sending a test email matching the keywords.

5. **Connect nodes in order:**
   - `Incoming Email Monitor` → `Email Validation Check` → `Send Telegram Message`.

6. **Activate the workflow** and monitor for incoming emails that match the subject keywords. Verify messages appear correctly in Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Before setting up, ensure Gmail API is enabled and OAuth2 credentials are configured properly.                        | Prerequisite for Gmail Trigger node authentication.                                                         |
| Create a Telegram bot via @BotFather and obtain the Bot Token.                                                       | Required for Telegram node authentication.                                                                   |
| Retrieve the Telegram Chat ID for the target chat (personal or group).                                               | Needed to direct messages correctly.                                                                         |
| Customize subject keywords in the IF node to filter different emails as needed.                                      | Allows tailoring notifications to specific use cases.                                                       |
| Message formatting can be customized using Markdown or HTML supported by Telegram.                                   | Enhances readability of notifications.                                                                       |
| Workflow created by WeblineIndia's Agentic business process automation team.                                         | https://www.weblineindia.com/process-automation-solutions.html                                              |
| For hiring dedicated developers to customize workflows, visit WeblineIndia.                                         | https://www.weblineindia.com/hire-dedicated-developers.html                                                 |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Forward Filtered Gmail Notifications to Telegram Chat" workflow in n8n. It covers all nodes, configurations, and potential issues to ensure smooth operation and easy customization.