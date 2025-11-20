Simple error workflow

https://n8nworkflows.xyz/workflows/simple-error-workflow-10755


# Simple error workflow

---

### 1. Workflow Overview

This workflow is designed to provide error alert notifications when any monitored workflow execution fails within the n8n environment. It captures error events and sends customizable alerts via connected notification tools such as Gmail and Slack. The logical structure is organized into two main blocks:

- **1.1 Error Detection and Message Preparation**: Listens for workflow errors and formats a notification message.
- **1.2 Alert Dispatching**: Sends the prepared error message to configured external communication tools (e.g., Gmail, Slack).

Additional sticky notes provide user guidance on setup and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Error Detection and Message Preparation

**Overview:**  
This block detects execution errors using the built-in Error Trigger node and formats a user-friendly error message for notifications.

**Nodes Involved:**  
- Error Trigger  
- Set error message  
- Sticky Note (Edit the error message)

**Node Details:**  

- **Error Trigger**  
  - Type: `Error Trigger` (event trigger node)  
  - Role: Listens globally for any workflow errors that occur in n8n.  
  - Configuration: Default, no parameters needed. Activates when an error happens anywhere in the instance.  
  - Inputs: None (event-based trigger)  
  - Outputs: Connects to "Set error message" node  
  - Edge Cases: Will not trigger if error notifications are disabled in monitored workflows; no input data to validate; failure unlikely unless n8n internal error occurs.  
  - Version: v1  

- **Set error message**  
  - Type: `Set` node  
  - Role: Constructs a formatted string containing workflow name and execution URL to describe the error.  
  - Configuration:  
    - Sets a single string field named `message`.  
    - Expression:  
      ```
      ⚠️ Workflow `{{$json["workflow"]["name"]}}` failed to run! [execution]({{ $json.execution.url }})
      ```  
    - Keeps only the `message` field to keep data payload minimal.  
  - Inputs: Receives error data from "Error Trigger"  
  - Outputs: Passes the message to alerting nodes  
  - Edge Cases: Expression errors may occur if `$json.workflow.name` or `$json.execution.url` is missing or malformed; however, these fields are standard in error payloads.  
  - Version: v1  

- **Sticky Note (Edit the error message)**  
  - Type: `Sticky Note`  
  - Role: User instruction to customize the error message in the "Set error message" node.  
  - No connections.  
  - No version requirements.

---

#### 1.2 Alert Dispatching

**Overview:**  
This block sends the formatted error message to external channels configured by the user for alerting purposes.

**Nodes Involved:**  
- Alert on Gmail  
- Alert on Slack  
- Sticky Note (Connect the tool where you want to be alerted in)  
- Sticky Note (Or pick a different node ...)

**Node Details:**  

- **Alert on Gmail**  
  - Type: `Gmail` node  
  - Role: Sends an email alert containing the error message.  
  - Configuration:  
    - Subject is set dynamically using the expression `={{ $json.message }}` to include the error message.  
    - Other parameters are default or empty.  
    - Requires valid Gmail OAuth2 credentials configured in n8n.  
  - Inputs: Receives from "Set error message" node  
  - Outputs: None (end node)  
  - Edge Cases: Authentication errors (expired or missing Gmail credentials), email sending limits, or Gmail API rate limits.  
  - Version: 2.1  

- **Alert on Slack**  
  - Type: `Slack` node  
  - Role: Sends a Slack message alert with the error information.  
  - Configuration:  
    - Uses default options; no specific message field is set in this workflow, so the actual alert content may need additional configuration by the user.  
    - Requires Slack webhook or OAuth2 credentials configured.  
  - Inputs: None connected in current workflow JSON (isolated node)  
  - Outputs: None  
  - Edge Cases: Authentication or permission errors, invalid webhook URL, Slack channel misconfiguration.  
  - Version: 2.3  

- **Sticky Note (Connect the tool where you want to be alerted in)**  
  - Type: `Sticky Note`  
  - Role: Instruction for users to connect and configure the alerting tools nodes as desired.  
  - No connections.  

- **Sticky Note (Or pick a different node ...)**  
  - Type: `Sticky Note`  
  - Role: Suggests users can select different alerting nodes if preferred.  
  - No connections.  

---

### 3. Summary Table

| Node Name         | Node Type       | Functional Role                          | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                     |
|-------------------|-----------------|----------------------------------------|----------------------|----------------------|------------------------------------------------------------------------------------------------|
| Error Trigger     | Error Trigger   | Detects workflow execution errors      | None                 | Set error message    |                                                                                                |
| Set error message | Set             | Formats error message for alerts       | Error Trigger        | Alert on Gmail       | Edit the error message                                                                          |
| Alert on Gmail    | Gmail           | Sends error alert email                 | Set error message    | None                 | Connect the tool where you want to be alerted in                                               |
| Alert on Slack    | Slack           | Sends error alert to Slack (not connected) | None                 | None                 | Connect the tool where you want to be alerted in                                               |
| Sticky Note       | Sticky Note     | Instruction to edit error message      | None                 | None                 | Edit the error message                                                                          |
| Sticky Note2      | Sticky Note     | Overview and usage instructions         | None                 | None                 | ## Error workflow alert This workflow sends an alert to the channel of your choice when an execution fails. How to use ☑️ Connect the tool where you want alerts to be sent ☑️ Turn on error notification in the workflows you want to monitor Help Step-by-step [tutorial](https://www.youtube.com/watch?v=bTF3tACqPRU) |
| Sticky Note3      | Sticky Note     | Instruction to connect alert tools      | None                 | None                 | Connect the tool where you want to be alerted in                                               |
| Sticky Note4      | Sticky Note     | Suggestion for alternative alert nodes  | None                 | None                 | Or pick a different node ...                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named appropriately (e.g., "Simple error workflow").

2. **Add an Error Trigger node**:  
   - Node Type: `Error Trigger`  
   - Parameters: Use default (no configuration needed)  
   - Position: Place it as the first node (e.g., top-left)  
   - Purpose: To catch error events from all workflows when enabled.

3. **Add a Set node**:  
   - Node Name: `Set error message`  
   - Parameters:  
     - Add a string field named `message`.  
     - Use the following expression for the value:  
       ```
       ⚠️ Workflow `{{$json["workflow"]["name"]}}` failed to run! [execution]({{ $json.execution.url }})
       ```  
     - Enable “Keep Only Set” to remove other fields.  
   - Connect the output of `Error Trigger` to this node.

4. **Add an email alert node**:  
   - Node Type: `Gmail`  
   - Node Name: `Alert on Gmail`  
   - Configure credentials: Set up and select valid Gmail OAuth2 credentials in n8n.  
   - Parameters:  
     - Subject: Use expression `={{ $json.message }}` to include the error message dynamically.  
     - Leave body or other fields empty or configure as needed.  
   - Connect output of `Set error message` to this node.

5. **Optionally add a Slack node** for Slack alerts:  
   - Node Type: `Slack`  
   - Node Name: `Alert on Slack`  
   - Configure credentials: Use Slack webhook or OAuth2 credentials.  
   - Parameters: Configure message content appropriately (not connected by default).  
   - Connect as needed if you want Slack alerts.

6. **Add Sticky Notes for documentation and instructions**:  
   - Add notes to guide users to customize the error message and connect alert nodes.  
   - Suggested contents:  
     - Edit error message in `Set error message` node.  
     - Connect alert tools where you want to be notified.  
     - Overview and usage instructions with link to tutorial: https://www.youtube.com/watch?v=bTF3tACqPRU  
     - Suggest alternative alert nodes.

7. **Activate error notifications** in the workflows you want to monitor:  
   - In those workflows, enable “Error Trigger” notifications so this workflow will receive their error events.

8. **Test the workflow** by causing an error in a monitored workflow and verify that an alert email is sent with the correct message.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow sends alerts when any monitored workflow fails, enabling immediate error awareness.   | General workflow purpose                         |
| Step-by-step video guide for setup and usage of error alert workflows in n8n.                        | https://www.youtube.com/watch?v=bTF3tACqPRU     |
| Users must configure OAuth2 credentials for Gmail and Slack nodes to enable alert delivery.         | Credential configuration requirement             |
| Customize the error message in the “Set error message” node to fit organizational needs.            | Workflow customization                            |
| Slack node included but not connected by default; user can connect and configure as desired.        | Optional alerting channel                         |

---

**Disclaimer:** This document summarizes an n8n workflow created for automated error alerting. All data handled is lawful and publicly accessible. The workflow respects all applicable content policies and contains no illegal or offensive material.