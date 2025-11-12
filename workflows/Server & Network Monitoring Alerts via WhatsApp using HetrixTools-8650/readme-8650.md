Server & Network Monitoring Alerts via WhatsApp using HetrixTools

https://n8nworkflows.xyz/workflows/server---network-monitoring-alerts-via-whatsapp-using-hetrixtools-8650


# Server & Network Monitoring Alerts via WhatsApp using HetrixTools

### 1. Workflow Overview

This workflow automates real-time monitoring alerts from HetrixTools by sending WhatsApp notifications using the GOWA WhatsApp REST API. It is designed to receive uptime and resource usage monitoring data from HetrixTools via a webhook, process the incoming data according to type, format a notification message, and send it as a WhatsApp message to a predefined number.

Logical blocks:

- **1.1 Input Reception**: Receives webhook POST requests from HetrixTools containing monitoring alerts.
- **1.2 Notification Type Decision**: Determines if the incoming alert is about server resource usage or uptime monitoring.
- **1.3 Message Preparation**: Formats the message text differently based on the alert type.
- **1.4 WhatsApp Sending Process**: Sends typing indicators, waits a randomized delay, stops typing, and then sends the formatted message via WhatsApp.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives incoming HTTP POST requests from HetrixTools containing monitoring alerts data.

- **Nodes Involved:**  
  - Webhook  
  - Sticky Note (Webhook instructions)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (n8n native)  
    - Role: Entry point to receive POST notifications from HetrixTools  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook path (`e03dc76d-ccdb-45cd-919f-f9f3fe3e2a20`)  
    - Inputs: External HTTP POST request  
    - Outputs: JSON with body containing monitoring alert data  
    - Edge Cases:  
      - Invalid or malformed JSON payloads may cause failures downstream  
      - Unauthorized or unexpected source requests are not explicitly filtered  
    - Notes:  
      - Requires HetrixTools contact configuration to point to this webhook URL  
      - See sticky note for setup instructions

  - **Sticky Note (Webhook instructions)**  
    - Content:  
      Explains prerequisites for HetrixTools account, monitor creation, and default contact setup with webhook URL.

---

#### 1.2 Notification Type Decision

- **Overview:**  
  This block checks the type of monitoring alert to branch processing: server resource usage notifications vs uptime monitoring.

- **Nodes Involved:**  
  - If Server Resource Usage Monitoring  
  - Sticky Note (If Node explanation)

- **Node Details:**

  - **If Server Resource Usage Monitoring**  
    - Type: If (conditional branching)  
    - Role: Routes execution based on presence of `resource_usage` object in the webhook payload  
    - Configuration:  
      - Condition: Checks if `$json.body.resource_usage` exists (object presence)  
      - Output 1 (True): Resource usage alert path  
      - Output 2 (False): Uptime alert path  
    - Inputs: Output from Webhook node  
    - Outputs: Two branches, one for resource usage alerts, one for others  
    - Edge Cases:  
      - If `resource_usage` is absent or null, the workflow assumes uptime alert  
      - Unexpected payload formats may cause this condition to fail silently  
    - Sticky Note explains this node’s role

---

#### 1.3 Message Preparation

- **Overview:**  
  Two parallel sets of nodes format the WhatsApp message text according to alert type, enriching the data with timestamp conversion and relevant fields.

- **Nodes Involved:**  
  - Edit Fields (resource usage)  
  - Edit Fields1 (uptime monitoring)  
  - Set Message Text  
  - Sticky Notes (Edit Fields explanations, Set Message Text explanation)

- **Node Details:**

  - **Edit Fields** (Resource Usage branch)  
    - Type: Set  
    - Role: Extracts and assigns key monitoring data fields and constructs a formatted multi-line text message for resource usage alerts  
    - Configuration:  
      - Assigns `body.monitor_name`, `body.resource_usage.resource_type`, `current_usage`, `average_usage`, `average_minutes` from webhook JSON  
      - Builds `teks` string combining these fields with a timestamp converted from UNIX epoch to localized string (UTC+08:00 timezone, en-GB locale)  
    - Inputs: True output of If node (resource usage alerts)  
    - Outputs: JSON with enriched fields and `teks` message string  
    - Edge Cases:  
      - Timestamp conversion assumes numeric UNIX timestamp in seconds  
      - Missing fields may cause incomplete message text  
    - Sticky Note describes this node’s purpose

  - **Edit Fields1** (Uptime Monitoring branch)  
    - Type: Set  
    - Role: Extracts uptime monitoring fields and builds a formatted multi-line message string for uptime alerts  
    - Configuration:  
      - Assigns `monitor_name`, `monitor_target`, `monitor_type`, `monitor_status`  
      - Builds `teks` string with alert status, target, type, timestamp (converted same as above), and optional downtime duration if present  
    - Inputs: False output of If node (uptime alerts)  
    - Outputs: JSON with enriched fields and `teks` message string  
    - Edge Cases:  
      - Optional `down_minutes` field handled conditionally  
      - Timestamp conversion same assumptions as above  
    - Sticky Note describes this node’s purpose

  - **Set Message Text**  
    - Type: Set  
    - Role: Copies the `teks` field from previous node to unify message format before sending  
    - Configuration:  
      - Assigns `teks` from previous node’s JSON  
    - Inputs: Output from either Edit Fields or Edit Fields1  
    - Outputs: JSON with final `teks` string ready for WhatsApp sending  
    - Sticky Note describes this node’s purpose

---

#### 1.4 WhatsApp Sending Process

- **Overview:**  
  This block simulates user typing presence on WhatsApp, delays sending for more natural interaction, then sends the formatted message to a target number via GOWA WhatsApp API.

- **Nodes Involved:**  
  - Send chat presence typing indicator  
  - Delay Typing  
  - Stop Typing1  
  - Send Message1  
  - Sticky Note (Sender Message Nodes)

- **Node Details:**

  - **Send chat presence typing indicator**  
    - Type: GOWA WhatsApp API node (`sendChatPresence` operation)  
    - Role: Sends "typing" presence indicator to WhatsApp contact  
    - Configuration:  
      - Phone Number set as expression placeholder `"=Change me [Whatsapp Number]"` (must be replaced)  
      - Operation: sendChatPresence (start typing)  
    - Inputs: Output from Set Message Text  
    - Outputs: Passes data to Delay Typing  
    - Credentials: GOWA API OAuth2 credential configured (named "GOWA account 2")  
    - Edge Cases:  
      - Invalid phone number or credential failure will block sending  
      - API rate limits or network errors possible

  - **Delay Typing**  
    - Type: Wait node  
    - Role: Introduces randomized delay between 3 to 8 seconds to mimic human typing delay  
    - Configuration:  
      - Amount: `={{ Math.random()*5+3 }}` seconds  
    - Inputs: Output from Send chat presence typing indicator  
    - Outputs: Next to Stop Typing1  
    - Edge Cases:  
      - Very low delay values possible but unlikely due to random range  
      - Long delays may cause timeout if external constraints exist

  - **Stop Typing1**  
    - Type: GOWA WhatsApp API node (`sendChatPresence` operation)  
    - Role: Stops the "typing" presence indicator  
    - Configuration:  
      - Phone Number: same placeholder as above (must be replaced)  
      - Operation: stop typing  
    - Inputs: Output from Delay Typing  
    - Outputs: Send Message1  
    - Credentials: Same as above  
    - Edge Cases: Same as Send chat presence typing indicator

  - **Send Message1**  
    - Type: GOWA WhatsApp API node (`sendMessage` operation)  
    - Role: Sends the constructed message text to the specified WhatsApp number  
    - Configuration:  
      - Message: Uses expression `={{ $('Set Message Text').item.json.teks }}` to get final message text  
      - Phone Number: same placeholder as above (must be replaced)  
    - Inputs: Output from Stop Typing1  
    - Outputs: End of workflow  
    - Credentials: Same as above  
    - Edge Cases:  
      - Text message length or content restrictions apply  
      - API or network errors possible  
    - Sticky Note explains this block’s role

---

### 3. Summary Table

| Node Name                         | Node Type                    | Functional Role                                   | Input Node(s)                 | Output Node(s)                     | Sticky Note                                    |
|----------------------------------|------------------------------|-------------------------------------------------|------------------------------|----------------------------------|------------------------------------------------|
| Webhook                          | Webhook                      | Entry point receiving HetrixTools POST alerts   | External HTTP POST            | If Server Resource Usage Monitoring | Explains webhook setup and HetrixTools config  |
| If Server Resource Usage Monitoring | If                          | Branches flow by alert type (resource usage or uptime) | Webhook                      | Edit Fields (true), Edit Fields1 (false) | Explains conditional branching                 |
| Edit Fields                     | Set                          | Formats message text for resource usage alerts  | If Server Resource Usage Monitoring (true) | Set Message Text                 | Describes message formatting for resource usage |
| Edit Fields1                    | Set                          | Formats message text for uptime monitoring alerts | If Server Resource Usage Monitoring (false) | Set Message Text                | Describes message formatting for uptime alerts |
| Set Message Text                | Set                          | Unifies message text field before sending       | Edit Fields, Edit Fields1     | Send chat presence typing indicator | Explains message text setting                   |
| Send chat presence typing indicator | GOWA WhatsApp API node       | Sends WhatsApp typing indicator (start)          | Set Message Text              | Delay Typing                     | Explains WhatsApp sending process               |
| Delay Typing                   | Wait                         | Adds randomized delay simulating typing delay    | Send chat presence typing indicator | Stop Typing1                   |                                                |
| Stop Typing1                  | GOWA WhatsApp API node       | Stops WhatsApp typing indicator                   | Delay Typing                  | Send Message1                   |                                                |
| Send Message1                 | GOWA WhatsApp API node       | Sends the final message text via WhatsApp        | Stop Typing1                  | (end)                          |                                                |
| Sticky Note                   | Sticky Note                  | Workflow setup instructions                       | -                            | -                              | Explains webhook setup                          |
| Sticky Note1                  | Sticky Note                  | Explains If Node                                  | -                            | -                              | Explains branching logic                        |
| Sticky Note2                  | Sticky Note                  | Explains Edit Fields Node (resource usage)       | -                            | -                              | Explains message formatting                     |
| Sticky Note3                  | Sticky Note                  | Explains Edit Fields Node (uptime monitoring)    | -                            | -                              | Explains message formatting                     |
| Sticky Note4                  | Sticky Note                  | Explains Set Message Text Node                    | -                            | -                              | Explains message text setting                   |
| Sticky Note5                  | Sticky Note                  | Explains WhatsApp message sending block           | -                            | -                              | Explains WhatsApp sending process               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Generate unique path (e.g., `e03dc76d-ccdb-45cd-919f-f9f3fe3e2a20`)  
   - Purpose: Receive HetrixTools alerts via HTTP POST  

2. **Add If Node to Branch by Alert Type:**  
   - Type: If  
   - Condition: Check if `{{$json.body.resource_usage}}` exists (object presence)  
   - Output 1: True branch (resource usage)  
   - Output 2: False branch (uptime monitoring)  

3. **Create Edit Fields Node for Resource Usage Alerts:**  
   - Type: Set  
   - Assign the following fields from webhook JSON:  
     - `body.monitor_name` = `{{$json.body.monitor_name}}`  
     - `body.resource_usage.resource_type` = `{{$json.body.resource_usage.resource_type}}`  
     - `body.resource_usage.current_usage` = `{{$json.body.resource_usage.current_usage}}`  
     - `body.resource_usage.average_usage` = `{{$json.body.resource_usage.average_usage}}`  
     - `body.resource_usage.average_minutes` = `{{$json.body.resource_usage.average_minutes}}`  
   - Set `teks` as a multi-line string combining above fields with timestamp formatted as local string in UTC+08:00 timezone:  
     ```
     {{$json.body.monitor_name}}  
     Resource Type : {{$json.body.resource_usage.resource_type}}  
     Current Usage : {{$json.body.resource_usage.current_usage}}  
     Average Usage : {{$json.body.resource_usage.average_usage}}  
     Noticed at : {{ new Date($json.body.timestamp * 1000).toLocaleString('en-GB', {year:'numeric', month:'2-digit', day:'2-digit', hour:'2-digit', minute:'2-digit', second:'2-digit'}) }} (UTC+08:00)
     ```  
   - Connect True output of If node here

4. **Create Edit Fields Node for Uptime Monitoring Alerts:**  
   - Type: Set  
   - Assign fields:  
     - `body.monitor_name` = `{{$json.body.monitor_name}}`  
     - `body.monitor_target` = `{{$json.body.monitor_target}}`  
     - `body.monitor_type` = `{{$json.body.monitor_type}}`  
     - `body.monitor_status` = `{{$json.body.monitor_status}}`  
   - Set `teks` as a multi-line string with conditional downtime display and timestamp:  
     ```
     {{$json.body.monitor_name}} is now {{$json.body.monitor_status}}  
     Target : {{$json.body.monitor_target}}  
     Type : {{$json.body.monitor_type}}  
     Noticed at : {{ new Date($json.body.timestamp * 1000).toLocaleString('en-GB', {year:'numeric', month:'2-digit', day:'2-digit', hour:'2-digit', minute:'2-digit', second:'2-digit'}) }} (UTC+08:00)  
     {{!$json.body.down_minutes ? '' : 'Downtime : ' + $json.body.down_minutes + ' minutes'}}
     ```  
   - Connect False output of If node here

5. **Create Set Message Text Node:**  
   - Type: Set  
   - Assign `teks` field from previous node: `{{$json.teks}}`  
   - Connect both Edit Fields and Edit Fields1 nodes outputs to this Set node input

6. **Configure GOWA WhatsApp API Nodes:**  
   - Ensure you have GOWA WhatsApp API credentials configured (OAuth2 with correct access tokens) in n8n, named e.g., "GOWA account 2"

7. **Add Send chat presence typing indicator Node:**  
   - Type: GOWA WhatsApp API node  
   - Operation: `sendChatPresence`  
   - Phone Number: Replace placeholder with actual WhatsApp number (format `+1234567890`)  
   - Connect output from Set Message Text node here

8. **Add Delay Node:**  
   - Type: Wait  
   - Duration: Expression `={{ Math.random()*5+3 }}` seconds (random delay between 3 and 8 seconds)  
   - Connect output from Send chat presence typing indicator node here

9. **Add Stop Typing Node:**  
   - Type: GOWA WhatsApp API node  
   - Operation: `sendChatPresence` with action `stop`  
   - Phone Number: Same as above  
   - Connect output from Delay node here

10. **Add Send Message Node:**  
    - Type: GOWA WhatsApp API node  
    - Operation: `sendMessage`  
    - Phone Number: Same as above  
    - Message: Expression `={{ $('Set Message Text').item.json.teks }}`  
    - Connect output from Stop Typing node here

11. **Test full workflow by triggering webhook POST with HetrixTools alert JSON**  
    - Validate message delivery on WhatsApp

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| HetrixTools account required to create monitors and configure alert contacts with webhook URL.  | https://hetrixtools.com/register/                               |
| GOWA WhatsApp REST API node requires OAuth2 credentials configured in n8n.                       | https://github.com/aldinokemal2104/gowa-n8n                     |
| Timestamp conversion assumes UNIX timestamp in seconds and converts to en-GB locale, UTC+08:00. | JavaScript `toLocaleString` options used in Set nodes           |
| WhatsApp number placeholders must be replaced with valid international format numbers.           | E.g., `+1234567890`                                             |
| This workflow simulates human typing indicators for better UX before sending messages.          | Delay and chat presence nodes                                   |

---

This document fully describes the workflow “Server & Network Monitoring Alerts via WhatsApp using HetrixTools” and allows for its understanding, reproduction, or modification without needing the original JSON.