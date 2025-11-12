Centralized Error Monitoring & Alerts via Telegram, Slack & Other Messengers

https://n8nworkflows.xyz/workflows/centralized-error-monitoring---alerts-via-telegram--slack---other-messengers-5852


# Centralized Error Monitoring & Alerts via Telegram, Slack & Other Messengers

### 1. Workflow Overview

This workflow, titled **Centralized Error Monitoring & Alerts via Telegram, Slack & Other Messengers**, is designed to centralize error notifications from multiple n8n workflows. Its primary purpose is to catch any workflow execution failures and automatically send detailed error alerts through a variety of messaging platforms, including Telegram, Slack, Gmail, WhatsApp, and Discord.

The workflow is logically divided into the following blocks:

- **1.1 Error Reception & Triggering:** Listens for errors from other workflows or manual triggers.
- **1.2 Error Data Preparation:** Processes and formats error details into a unified message.
- **1.3 Multi-Channel Notification Dispatch:** Sends the formatted error notifications to configured messaging channels.
- **1.4 Setup and Documentation:** Provides guidance notes and usage instructions embedded as sticky notes for users.

---

### 2. Block-by-Block Analysis

#### 1.1 Error Reception & Triggering

**Overview:**  
This block listens for error events either via a dedicated error trigger node (disabled by default) or through an execute workflow trigger node that receives input from other workflows. It acts as the entry point for error data into the notification system.

**Nodes Involved:**  
- Error Trigger  
- When Executed by Another Workflow  
- Execute Bag Alert Workflow (disabled)  

**Node Details:**

- **Error Trigger**  
  - *Type:* n8n Error Trigger  
  - *Role:* Automatically fires when any workflow error occurs in n8n.  
  - *Config:* Disabled by default; can be enabled to catch any workflow errors globally.  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Connects to "Execute Bag Alert Workflow" (disabled)  
  - *Potential Failures:* None intrinsic; however, if enabled, might capture unwanted errors if not filtered.  
  - *Notes:* Disabled in this workflow; alternative triggering used.

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Activated when called explicitly by other workflows to pass error data.  
  - *Config:* Set to passthrough input source, forwarding incoming data as-is.  
  - *Inputs:* External workflows invoking this workflow.  
  - *Outputs:* Connects to "Prepare Messages For Notify" node.  
  - *Potential Failures:* Input data format mismatch or missing fields may cause downstream errors.

- **Execute Bag Alert Workflow**  
  - *Type:* Execute Workflow  
  - *Role:* Appears intended to invoke itself or another workflow for alert processing  
  - *Config:* Disabled, references the same workflow ID as current (self)  
  - *Inputs:* Error Trigger output (disabled)  
  - *Outputs:* None (currently disabled node)  
  - *Notes:* This node is not active and may represent a legacy or alternative design approach.

---

#### 1.2 Error Data Preparation

**Overview:**  
This block extracts relevant error details from the incoming data, formats them into a comprehensive, readable message enriched with HTML for better readability in supported channels.

**Nodes Involved:**  
- Prepare Messages For Notify

**Node Details:**

- **Prepare Messages For Notify**  
  - *Type:* Code (JavaScript) Node  
  - *Role:* Parses incoming error data, extracts workflow name, node name, error message, description, timestamp, and execution URL. It then builds a formatted notification message.  
  - *Config:* Uses JavaScript to safely access nested properties and provide default values if missing. Formats timestamp according to Russian locale (ru-RU).  
  - *Key Expressions:*  
    - Extracts `workflow.name`, `execution.lastNodeExecuted`, `execution.error.message`, `error.context.messageTemplate`, `execution.url`, `execution.id`, and `error.level`.  
    - Constructs HTML-based message string with placeholders and links to execution URL.  
  - *Inputs:* Receives error data JSON from "When Executed by Another Workflow" node.  
  - *Outputs:* Outputs JSON containing a single property `message` (the formatted string).  
  - *Potential Failures:* Malformed or incomplete input JSON may cause undefined values. Timestamp conversion may fail if invalid date.  
  - *Version:* Uses Code node version 2 for better JavaScript support.

---

#### 1.3 Multi-Channel Notification Dispatch

**Overview:**  
This block sends the prepared error message to multiple configured messaging platforms simultaneously. Most notification nodes are disabled by default except Telegram.

**Nodes Involved:**  
- Send Notify to Telegram  
- Send Notify to Gmail (disabled)  
- Send Notify to Whatsapp (disabled)  
- Send Notify to Discord (disabled)  
- Send Notify to Slack (disabled)  

**Node Details:**

- **Send Notify to Telegram**  
  - *Type:* Telegram Node  
  - *Role:* Sends the formatted error message as an HTML-formatted Telegram message to a specific chat ID.  
  - *Config:*  
    - `text` parameter uses the expression `={{ $json.message }}` to inject the prepared message.  
    - `chatId` is set via an expression, placeholder `"YOUR_CHAT_ID"` to be replaced by user.  
    - `parse_mode` set to `HTML` for rich text.  
  - *Credentials:* Requires Telegram API credentials with chat access.  
  - *Inputs:* Receives message JSON from "Prepare Messages For Notify".  
  - *Outputs:* None (terminal notification node).  
  - *Potential Failures:* Invalid chat ID, expired credentials, or network timeouts.

- **Send Notify to Gmail**  
  - *Type:* Gmail Node  
  - *Role:* Sends an email with the error message as body and "Error Workflow" as subject.  
  - *Config:*  
    - `sendTo` set to a placeholder `@gmail.com` to be replaced.  
    - `message` injected from prepared message.  
  - *Credentials:* Requires OAuth2 Gmail credentials.  
  - *Disabled:* Yes, not active by default.  
  - *Potential Failures:* Authentication errors, invalid recipient email.

- **Send Notify to Whatsapp**  
  - *Type:* WhatsApp Node  
  - *Role:* Sends the error message text to a specified phone number.  
  - *Config:*  
    - `recipientPhoneNumber` set to a placeholder `+12345678901`.  
    - `textBody` populated with the prepared message.  
  - *Credentials:* Requires WhatsApp API credentials.  
  - *Disabled:* Yes.  
  - *Potential Failures:* API limits, invalid phone number, credential expiry.

- **Send Notify to Discord**  
  - *Type:* Discord Node  
  - *Role:* Sends the error message content via Discord webhook.  
  - *Config:*  
    - Uses webhook authentication with credentials.  
    - Message content injected from prepared message.  
  - *Disabled:* Yes.  
  - *Potential Failures:* Webhook invalidation, rate limits.

- **Send Notify to Slack**  
  - *Type:* Slack Node  
  - *Role:* Posts the error message text to Slack as a user message.  
  - *Config:*  
    - `text` uses prepared message.  
    - User selection mode enabled but no user specified (empty).  
  - *Credentials:* Slack API token required.  
  - *Disabled:* Yes.  
  - *Potential Failures:* Invalid token, missing user ID, rate limits.

---

#### 1.4 Setup and Documentation

**Overview:**  
This block consists of sticky notes providing users with instructions, usage guidelines, and links for further resources.

**Nodes Involved:**  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note (titled "ERROR ALERTER")

**Node Details:**

- **Sticky Note1**  
  - *Content:* "# ERROR NOTIFIER"  
  - *Purpose:* Title header for the workflow.

- **Sticky Note2**  
  - *Content:*  
    ```
    # How it works
    This workflow automatically sends notifications when any other workflow fails.

    Algorithm:
    - Create a workflow with "**ERROR NOTIFIER**".
    - Copy and paste "**ERROR ALERTER**" into your workflows and activate.
    - Add credentials using your preferred channel.

    Please check out my other items on [gumroad](https://boanse.gumroad.com/?section=k_Sn6LcT_dzJFnp5jmsM5A%3D%3D)
    You might also like something else☺️
    ```
  - *Purpose:* Explains how to set up the error notification system and points to additional resources.

- **Sticky Note3**  
  - *Content:*  
    ```
    # Setup
    - ## Add Credentials:
       - WhatsApp (OAuth & API)
       - Telegram
       - Gmail
       - Discord
       - Slack
    ```
  - *Purpose:* Lists messaging channels supported and reminds users to configure credentials accordingly.

- **Sticky Note (ERROR ALERTER)**  
  - *Content:* "# ERROR ALERTER"  
  - *Purpose:* Section header for the alerting nodes.

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                      | Input Node(s)                  | Output Node(s)                                          | Sticky Note                                                                                                                       |
|-----------------------------|---------------------------|------------------------------------|-------------------------------|---------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note1                | Sticky Note               | Workflow Title                     | None                          | None                                                    | # ERROR NOTIFIER                                                                                                                  |
| Error Trigger               | Error Trigger             | Global workflow error listener    | None                          | Execute Bag Alert Workflow (disabled)                   |                                                                                                                                  |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point for external workflow error data | None                          | Prepare Messages For Notify                               |                                                                                                                                  |
| Execute Bag Alert Workflow  | Execute Workflow          | (Disabled) Possibly self-invocation | Error Trigger (disabled)       | None                                                    |                                                                                                                                  |
| Prepare Messages For Notify | Code                      | Format error data into message    | When Executed by Another Workflow | Send Notify to Gmail, Whatsapp, Slack, Telegram, Discord |                                                                                                                                  |
| Send Notify to Gmail        | Gmail                     | Send error notification via email | Prepare Messages For Notify    | None                                                    |                                                                                                                                  |
| Send Notify to Whatsapp     | WhatsApp                  | Send error notification via WhatsApp | Prepare Messages For Notify    | None                                                    |                                                                                                                                  |
| Send Notify to Telegram     | Telegram                  | Send error notification via Telegram | Prepare Messages For Notify    | None                                                    |                                                                                                                                  |
| Send Notify to Discord      | Discord                   | Send error notification via Discord | Prepare Messages For Notify    | None                                                    |                                                                                                                                  |
| Send Notify to Slack        | Slack                     | Send error notification via Slack | Prepare Messages For Notify    | None                                                    |                                                                                                                                  |
| Sticky Note2                | Sticky Note               | Usage instructions and explanation | None                          | None                                                    | # How it works (includes link to https://boanse.gumroad.com/?section=k_Sn6LcT_dzJFnp5jmsM5A%3D%3D)                            |
| Sticky Note3                | Sticky Note               | Setup instructions for credentials | None                          | None                                                    | # Setup (lists WhatsApp, Telegram, Gmail, Discord, Slack credentials)                                                            |
| Sticky Note                 | Sticky Note               | Section header for alerting block | None                          | None                                                    | # ERROR ALERTER                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named "Error_Notifier-v1-no_db".

2. **Add Sticky Notes for documentation:**  
   - Create a sticky note titled "# ERROR NOTIFIER" as the main header.  
   - Add another sticky note with setup instructions for credentials (WhatsApp, Telegram, Gmail, Discord, Slack).  
   - Add a sticky note explaining "How it works" with the usage algorithm and a link to [gumroad](https://boanse.gumroad.com/?section=k_Sn6LcT_dzJFnp5jmsM5A%3D%3D).  
   - Add a sticky note titled "# ERROR ALERTER" above the alert nodes section.

3. **Configure the error reception nodes:**  
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow".  
     - Set `inputSource` to "passthrough" to accept any JSON input.  
   - Optionally add an **Error Trigger** node (disabled by default) to catch all global workflow errors.  
   - Do not add the "Execute Bag Alert Workflow" node since it is disabled and self-referential.

4. **Add the message preparation node:**  
   - Add a **Code** node named "Prepare Messages For Notify".  
   - Use the following JavaScript code to parse incoming error data and format a message:  

   ```javascript
   const errorData = $input.first().json;

   const execution = errorData.execution || {};
   const workflow = errorData.workflow || {};
   const error = execution.error || {};

   const workflowName = workflow.name || 'Unknown';
   const nodeName = execution?.lastNodeExecuted|| 'Unknown';
   const errorMessage = error.message || '-';
   const errorDescription = error?.context?.messageTemplate || 'No description';
   const timestamp = error.timestamp ? new Date(error.timestamp).toLocaleString('ru-RU') : 'Unknown';

   const executionUrl = execution.url || '#';
   const executionId = execution.id || 'Unknown';
   const errorLevel = error.level ? error.level.toUpperCase() : 'UNKNOWN';

   const message = 
   `🚨 <b>WORKFLOW ERROR (<a href="${executionUrl}">${executionId}</a>)</b>

   Workflow: <code>${workflowName}</code>
   Node: <code>${nodeName}</code>

   <b>${errorLevel}:</b>
   ${errorMessage}
   ${errorDescription}

   ${timestamp}`;

   return {
     json: {
       message,
     }
   };
   ```

   - Connect "When Executed by Another Workflow" node output to this Code node.

5. **Add notification nodes for each messaging platform:**  
   - Add a **Telegram** node named "Send Notify to Telegram".  
     - Set `text` to `={{ $json.message }}`.  
     - Set `chatId` to your actual Telegram chat ID (replace `"YOUR_CHAT_ID"`).  
     - Under Additional Fields, set `parse_mode` to `"HTML"`.  
     - Attach your Telegram API credentials to this node.  
   - Add optional nodes (disabled by default) for other channels:  
     - **Gmail** node "Send Notify to Gmail"  
       - Set recipient email in `sendTo`.  
       - Set `message` to `={{ $json.message }}`.  
       - Set subject to `"Error Workflow"`.  
       - Attach Gmail OAuth2 credentials.  
     - **WhatsApp** node "Send Notify to Whatsapp"  
       - Set `recipientPhoneNumber` to the intended phone number.  
       - Set `textBody` to `={{ $json.message }}`.  
       - Attach WhatsApp API credentials.  
     - **Discord** node "Send Notify to Discord"  
       - Use webhook authentication.  
       - Set `content` to `={{ $json.message }}`.  
       - Attach Discord Webhook credentials.  
     - **Slack** node "Send Notify to Slack"  
       - Set `text` to `={{ $json.message }}`.  
       - User selection can be left empty or set as needed.  
       - Attach Slack API credentials.

6. **Connect the "Prepare Messages For Notify" node output to all notification nodes inputs simultaneously.**

7. **Save and activate the workflow.**

8. **Integration in other workflows:**  
   - To use this error notifier, copy the "ERROR ALERTER" logic (e.g., the Execute Workflow Trigger and downstream nodes) or call this workflow explicitly on error events.  
   - Ensure credentials for each notification channel are properly set up in n8n.  
   - Replace all placeholder values (`chatId`, phone numbers, emails) with real ones.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow is designed to be used as a centralized error notification system for n8n workflows.               | Workflow purpose                                                                                        |
| To use, add the "**ERROR ALERTER**" logic to your workflows and activate it to forward errors to this workflow. | Usage Instructions                                                                                      |
| Messaging channels supported include WhatsApp (OAuth/API), Telegram, Gmail, Discord, and Slack.                   | Setup instructions                                                                                      |
| Please check out additional n8n workflow templates and tools on [Gumroad](https://boanse.gumroad.com/?section=k_Sn6LcT_dzJFnp5jmsM5A%3D%3D) | Author’s resource page                                                                                   |

---

**Disclaimer:**  
The provided text and workflow components originate exclusively from an automated workflow created with n8n, a workflow automation platform. This content complies strictly with applicable content policies and contains no illegal, offensive, or proprietary elements. All processed data is legal and public.