Multi-Channel Event Countdown Manager with Telegram, Slack and Email

https://n8nworkflows.xyz/workflows/multi-channel-event-countdown-manager-with-telegram--slack-and-email-10346


# Multi-Channel Event Countdown Manager with Telegram, Slack and Email

### 1. Workflow Overview

This workflow, titled **"Multi-Channel Event Countdown Manager with Telegram, Slack and Email"**, is designed to automate countdown notifications for upcoming events across multiple communication channels. It targets organizations or teams that want to keep stakeholders informed about important dates like product launches, birthdays, or conferences through Slack messages, emails, or potentially Telegram (though Telegram is not explicitly configured in the current workflow).

The workflow consists of two main operational modes:

- **1.1 Scheduled Notification Dispatch:** Runs daily at 9 AM, fetching pre-defined events, calculating their countdown, filtering those requiring notifications, and distributing messages via Slack or email depending on event configuration.
  
- **1.2 Webhook Event Reception:** Listens for external POST requests to trigger custom event processing and responds with a confirmation.

Logical blocks:

- **1.1 Input Reception:** Scheduled trigger and webhook trigger nodes initiating workflow runs.
- **1.2 Events Processing:** Fetching and filtering the event list with countdown calculations.
- **1.3 Channel Routing:** Logic nodes that determine whether to notify via Slack or Email.
- **1.4 Message Formatting:** Preparing message content tailored for Slack or email.
- **1.5 Dispatching Notifications:** Sending messages to Slack channels or email recipients.
- **1.6 Webhook Response:** Handling external webhook responses gracefully.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception

**Overview:**  
This block initiates the workflow either on a scheduled daily basis or upon receiving an external webhook POST request.

**Nodes Involved:**  
- Schedule Trigger  
- Webhook Trigger

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically starts the workflow daily at 9:00 AM.  
  - Configuration: Cron expression set to "0 9 * * *" (9 AM every day).  
  - Inputs: None (trigger node).  
  - Outputs: Connects to "Events Database" node.  
  - Edge Cases: Cron misconfiguration or server downtime could delay triggers.

- **Webhook Trigger**  
  - Type: Webhook Trigger  
  - Role: Listens for external POST requests at path `/event-countdown`.  
  - Configuration: HTTP POST method, triggers workflow on receiving data. Response mode set to respond from a later node.  
  - Inputs: External HTTP POST requests.  
  - Outputs: Connects to "Process Webhook Event" node.  
  - Edge Cases: Incorrect or malformed POST requests, authentication not implemented here so open endpoint risks.

---

#### Block 1.2: Events Processing

**Overview:**  
Fetches the event data from a hardcoded list, calculates how many days remain until each event, and filters the events to only those requiring notification based on specific countdown thresholds.

**Nodes Involved:**  
- Events Database

**Node Details:**

- **Events Database**  
  - Type: Code Node (JavaScript)  
  - Role: Defines event details and computes days remaining until each event. Filters for events that need notification (within 7 days, or exactly 14 or 30 days).  
  - Configuration:  
    - Events are hardcoded with properties: name, date, type, channel, and communication details (Slack webhook URL or email recipients).  
    - Dates normalized to midnight to avoid timezone issues.  
    - Logic to set flags: isPast, isToday, shouldNotify.  
    - Returns only events flagged for notification.  
  - Inputs: Trigger from Schedule Trigger.  
  - Outputs: Connects to "Is Slack?" and "Is Email?" nodes for routing.  
  - Edge Cases:  
    - Hardcoded events limit extensibility; no external data source integration.  
    - Date parsing errors if event dates are malformed.  
    - Timezone issues if server timezone differs from event timezone.  
  - Version: Uses typeVersion 2.

---

#### Block 1.3: Channel Routing

**Overview:**  
Determines the notification channel based on event data and routes the processing accordingly.

**Nodes Involved:**  
- Is Slack? (IF Node)  
- Is Email? (IF Node)

**Node Details:**

- **Is Slack?**  
  - Type: IF Node  
  - Role: Checks if the event's channel property equals "slack".  
  - Configuration: String equality check on `$json.channel` against "slack".  
  - Inputs: From Events Database.  
  - Outputs: True branch to "Format Slack Message", False branch unused.  
  - Edge Cases: Case sensitivity may cause mismatches; no fallback channel.

- **Is Email?**  
  - Type: IF Node  
  - Role: Checks if the event's channel property equals "email".  
  - Configuration: String equality check on `$json.channel` against "email".  
  - Inputs: From Events Database.  
  - Outputs: True branch to "Format Email", False branch unused.  
  - Edge Cases: Same as above; no support for other channels configured.

---

#### Block 1.4: Message Formatting

**Overview:**  
Prepares customized notification messages formatted for Slack or email, incorporating emojis, colors, and structured content based on event type and urgency.

**Nodes Involved:**  
- Format Slack Message  
- Format Email

**Node Details:**

- **Format Slack Message**  
  - Type: Code Node  
  - Role: Generates a Slack-compatible message text with emoji and countdown logic.  
  - Configuration:  
    - Emoji chosen by event type: Calendar (default), Birthday Cake, or Rocket.  
    - Message varies if event is today, tomorrow, or further out.  
    - Outputs JSON with `text` (message) and `webhookUrl`.  
  - Inputs: From Is Slack? true output.  
  - Outputs: Connects to "Send to Slack" node.  
  - Edge Cases: Missing or invalid webhook URL could cause HTTP errors downstream.

- **Format Email**  
  - Type: Code Node  
  - Role: Creates an HTML email body with styling and subject lines reflecting event urgency and type.  
  - Configuration:  
    - Emoji and color vary by event type and days remaining (e.g., red if <=3 days).  
    - Subject and body adjust if event is today, tomorrow, or later.  
    - Outputs JSON with `subject`, `html` body, and `recipients` array.  
  - Inputs: From Is Email? true output.  
  - Outputs: Connects to "Send Email" node.  
  - Edge Cases: Email HTML compatibility across clients; missing recipients cause send failures.

---

#### Block 1.5: Dispatching Notifications

**Overview:**  
Sends the formatted notifications to their respective channels: Slack via webhook POST and email via SMTP.

**Nodes Involved:**  
- Send to Slack  
- Send Email

**Node Details:**

- **Send to Slack**  
  - Type: HTTP Request Node  
  - Role: Posts the formatted message to the Slack webhook URL.  
  - Configuration:  
    - POST method to dynamic URL from `$json.webhookUrl`.  
    - Body parameter `text` set from `$json.text`.  
  - Inputs: From "Format Slack Message".  
  - Outputs: None (endpoint).  
  - Edge Cases:  
    - Slack webhook URL invalid or revoked leads to HTTP errors.  
    - Network or timeout issues.  
  - Version: v4.2

- **Send Email**  
  - Type: Email Send Node  
  - Role: Sends the HTML email to all specified recipients.  
  - Configuration:  
    - Subject from `$json.subject`.  
    - Recipient list joined by commas from `$json.recipients`.  
    - From address fixed as `noreply@yourcompany.com`.  
    - Uses SMTP credentials named "SMTP -test".  
  - Inputs: From "Format Email".  
  - Outputs: None (endpoint).  
  - Edge Cases:  
    - SMTP authentication errors or connection issues.  
    - Empty recipient list causes failure.  
    - HTML formatting may be blocked by spam filters.

---

#### Block 1.6: Webhook Response

**Overview:**  
Processes incoming webhook data and returns a JSON confirmation response.

**Nodes Involved:**  
- Process Webhook Event  
- Webhook Response

**Node Details:**

- **Process Webhook Event**  
  - Type: Code Node  
  - Role: Parses incoming webhook JSON body and prepares a success response with timestamp and echoed data.  
  - Configuration: Accesses `$input.first().json.body` to read payload.  
  - Inputs: From Webhook Trigger.  
  - Outputs: Connects to "Webhook Response".  
  - Edge Cases: Missing or malformed body could cause errors.

- **Webhook Response**  
  - Type: Respond to Webhook Node  
  - Role: Sends back the processed JSON response to the external webhook caller.  
  - Configuration: Responds with JSON body from previous node.  
  - Inputs: From "Process Webhook Event".  
  - Outputs: None (endpoint).  
  - Edge Cases: None significant; ensures communication closure.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                        | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                      |
|----------------------|---------------------|-------------------------------------|-----------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger    | Triggers workflow daily at 9 AM      | None                  | Events Database          | Triggers the workflow automatically on a defined schedule (e.g., daily at 9 AM).                |
| Events Database      | Code                | Defines event list and filters for notification | Schedule Trigger       | Is Slack?, Is Email?     | Fetches upcoming events (like launches or birthdays) from a data source.                        |
| Is Slack?            | IF Node             | Checks if event channel is Slack     | Events Database        | Format Slack Message      | Checks if the event’s notification channel is set to Slack.                                    |
| Is Email?            | IF Node             | Checks if event channel is Email     | Events Database        | Format Email             | Checks if the event’s notification channel is set to Email.                                    |
| Format Slack Message | Code                | Formats Slack message content         | Is Slack?              | Send to Slack            | Formats event details into a Slack message layout.                                             |
| Send to Slack        | HTTP Request        | Sends message to Slack webhook        | Format Slack Message   | None                     | Sends the formatted message to the target Slack channel.                                      |
| Format Email         | Code                | Formats email content for event       | Is Email?              | Send Email               | Formats event details into an email-friendly message.                                         |
| Send Email           | Email Send          | Sends formatted email to recipients   | Format Email           | None                     | Send Email – Sends the formatted email to the recipient list.                                 |
| Webhook Trigger      | Webhook Trigger     | Starts workflow on external POST request | None                  | Process Webhook Event    | Starts the workflow when an external system sends a POST request.                             |
| Process Webhook Event| Code                | Parses and validates webhook data     | Webhook Trigger        | Webhook Response         | Parses and validates the incoming webhook data.                                               |
| Webhook Response     | Respond to Webhook  | Sends confirmation back to webhook sender | Process Webhook Event | None                     | Sends a confirmation or status response back to the webhook sender.                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a "Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Set cron expression to `0 9 * * *` to run daily at 9:00 AM.

2. **Create a "Events Database" node**  
   - Type: Code Node  
   - Paste JavaScript code that:  
     - Defines events array with properties (name, date, type, channel, webhookUrl/emailRecipients)  
     - Calculates days remaining for each event (normalized to midnight)  
     - Flags events needing notification (<=7 days or exactly 14 or 30 days)  
     - Returns filtered events as output JSON.

3. **Link Schedule Trigger output to Events Database input.**

4. **Create an "Is Slack?" node**  
   - Type: IF Node  
   - Condition: `$json.channel` equals `"slack"` (case sensitive).

5. **Create an "Is Email?" node**  
   - Type: IF Node  
   - Condition: `$json.channel` equals `"email"`.

6. **Connect Events Database outputs to both "Is Slack?" and "Is Email?" nodes in parallel.**

7. **Create "Format Slack Message" node**  
   - Type: Code Node  
   - Script to format Slack message text:  
     - Select emoji by event type (Calendar default, Birthday Cake, Rocket).  
     - Create message text depending on daysRemaining (today, tomorrow, or countdown).  
     - Output JSON with `text` and `webhookUrl`.

8. **Connect "Is Slack?" node true output to "Format Slack Message".**

9. **Create "Send to Slack" node**  
   - Type: HTTP Request Node  
   - Method: POST  
   - URL: use expression `{{$json.webhookUrl}}`  
   - Body parameters: include `text` from `{{$json.text}}`.  
   - No authentication needed (Slack webhook URL is sufficient).

10. **Connect "Format Slack Message" output to "Send to Slack".**

11. **Create "Format Email" node**  
    - Type: Code Node  
    - Script to generate HTML email:  
      - Choose emoji and color by event type and urgency  
      - Compose subject and HTML body with event details and countdown  
      - Output JSON with `subject`, `html`, and `recipients` (array).

12. **Connect "Is Email?" node true output to "Format Email".**

13. **Create "Send Email" node**  
    - Type: Email Send Node  
    - Subject: expression from `{{$json.subject}}`  
    - To email: join array `{{$json.recipients.join(',')}}`  
    - From email: fixed as `noreply@yourcompany.com`  
    - Set SMTP credentials (create if necessary, e.g., "SMTP -test").

14. **Connect "Format Email" output to "Send Email".**

15. **Create "Webhook Trigger" node**  
    - Type: Webhook Trigger  
    - HTTP Method: POST  
    - Path: `event-countdown`  
    - Response mode: Respond from a later node.

16. **Create "Process Webhook Event" node**  
    - Type: Code Node  
    - Script to parse incoming webhook body and prepare JSON response including status, message, timestamp, and echoed data.

17. **Connect "Webhook Trigger" output to "Process Webhook Event".**

18. **Create "Webhook Response" node**  
    - Type: Respond to Webhook Node  
    - Respond with JSON body from previous node.

19. **Connect "Process Webhook Event" output to "Webhook Response".**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow automates countdown notifications across Slack and Email channels with visual styling. | Useful for teams needing multi-channel event reminders.                                           |
| Slack webhook URLs must be configured correctly for Slack messaging to work.                         | Slack Incoming Webhooks documentation: https://api.slack.com/messaging/webhooks                    |
| SMTP credentials must be valid and allow sending from the configured "from" address.                 | Set up SMTP with your email provider or use test credentials for development.                      |
| Webhook endpoint `/event-countdown` allows integration with external systems triggering event logic.| Useful for dynamic event processing or extending to other channels like Telegram with minor edits.|

---

**Disclaimer:**  
The provided workflow is created exclusively with n8n, adhering strictly to content policies. It contains no illegal or protected data. All event data is public or example information.