Send TheHive Alerts Using SIGNL4

https://n8nworkflows.xyz/workflows/send-thehive-alerts-using-signl4-1630


# Send TheHive Alerts Using SIGNL4

### 1. Workflow Overview

This workflow integrates TheHive 5, a security incident response platform, with SIGNL4, a mobile alerting and incident response solution. It forwards alerts from TheHive to SIGNL4 to ensure reliable team notifications and alert resolution synchronization.

The workflow is organized into the following logical blocks:

- **1.1 Testing TheHive Connection:** Nodes that allow manual testing of TheHive API connectivity by creating and reading alerts.
- **1.2 Webhook Reception:** A webhook node configured to receive real-time alert notifications from TheHive.
- **1.3 Alert Condition Evaluation:** An IF node that evaluates whether the incoming alert from TheHive is closed or still active.
- **1.4 Alert Forwarding to SIGNL4:** Nodes that send new alerts to SIGNL4 or resolve alerts in SIGNL4 based on TheHive alert status.

---

### 2. Block-by-Block Analysis

#### 2.1 Testing TheHive Connection

**Overview:**  
This block provides manual trigger nodes to test TheHive API connectivity by creating a sample alert and reading existing alerts from TheHive.

**Nodes Involved:**  
- Start (Testing)  
- TheHive Create Alert  
- TheHive Read Alerts

**Node Details:**

- **Start (Testing)**  
  - *Type & Role:* Manual Trigger node to initiate the test workflow manually.  
  - *Configuration:* No parameters; simply triggers downstream nodes.  
  - *Inputs/Outputs:* No input; outputs to "TheHive Create Alert" node.  
  - *Edge Cases:* None; manual trigger.  

- **TheHive Create Alert**  
  - *Type & Role:* TheHive node for creating an alert in TheHive system.  
  - *Configuration:*  
    - Date hardcoded to 2022-04-25T08:53:18Z  
    - Tags set to "tlp:pwhite" (indicates a traffic light protocol marking for public information)  
    - Type set to "misp" (indicating a MISP event type)  
    - Title, source, sourceRef, and description describe a sample security issue on server A2.  
    - No additional fields configured.  
  - *Credentials:* Uses TheHive API credentials named "The Hive account".  
  - *Inputs/Outputs:* Input from "Start (Testing)"; no downstream nodes connected.  
  - *Edge Cases:* Potential API authentication failure, invalid date or field values, network timeouts.  

- **TheHive Read Alerts**  
  - *Type & Role:* TheHive node to fetch all alerts from TheHive.  
  - *Configuration:* No filters or options; retrieves all alerts.  
  - *Credentials:* Same TheHive API credentials.  
  - *Inputs/Outputs:* No upstream node connected; standalone for testing purposes.  
  - *Edge Cases:* API authentication errors, large result sets causing timeouts or memory issues.  

---

#### 2.2 Webhook Reception

**Overview:**  
Receives incoming HTTP POST requests from TheHive containing alert notifications. Acts as the entry point for live alert processing.

**Nodes Involved:**  
- TheHive Webhook Request

**Node Details:**

- **TheHive Webhook Request**  
  - *Type & Role:* Webhook node configured to accept POST requests from TheHive alert notifications.  
  - *Configuration:*  
    - Path set to a unique webhook ID: "22c76955-3f52-469e-a8ae-3f62e8e87ebe"  
    - HTTP Method: POST  
    - No special options configured.  
  - *Inputs/Outputs:* No inputs; outputs to "IF" node for condition evaluation.  
  - *Edge Cases:*  
    - Incoming payload may not conform to expected schema.  
    - Authentication or IP whitelisting to be configured externally in TheHive settings.  
    - Network or webhook server downtime could cause missed alerts.  

---

#### 2.3 Alert Condition Evaluation

**Overview:**  
Evaluates whether the incoming TheHive alert is already closed. If not closed, the alert is sent to SIGNL4; if closed, the corresponding SIGNL4 alert is resolved.

**Nodes Involved:**  
- IF

**Node Details:**

- **IF**  
  - *Type & Role:* Conditional logic node to check alert status.  
  - *Configuration:*  
    - Condition compares the "stage" field inside the webhook JSON payload at `body.object.stage` to the string "Closed".  
    - Uses a boolean condition: the value must *not equal* "Closed" to proceed to sending an alert.  
  - *Inputs/Outputs:*  
    - Input from "TheHive Webhook Request".  
    - Outputs:  
      - True branch (stage != Closed) connects to "SIGNL4 Send Alert".  
      - False branch (stage == Closed) connects to "SIGNL4 Resolve Alert".  
  - *Edge Cases:*  
    - Missing or malformed "stage" field in webhook payload.  
    - Case sensitivity issues with "Closed" string.  
    - Unexpected alert stages not handled explicitly.  

---

#### 2.4 Alert Forwarding to SIGNL4

**Overview:**  
Sends new alerts to SIGNL4 or resolves alerts in SIGNL4 based on TheHive alert status evaluated in the IF node.

**Nodes Involved:**  
- SIGNL4 Send Alert  
- SIGNL4 Resolve Alert

**Node Details:**

- **SIGNL4 Send Alert**  
  - *Type & Role:* SIGNL4 node to create/send an alert notification to SIGNL4.  
  - *Configuration:*  
    - Message is set dynamically from the TheHive webhook payload at `body.details.description`.  
    - Additional fields:  
      - Title is set from `body.details.title`.  
      - External ID is set from `body.objectId` to uniquely identify the alert in SIGNL4.  
  - *Credentials:* Uses SIGNL4 API credentials named "SIGNL4 Webhook account".  
  - *Inputs/Outputs:* Input from IF node true branch; no downstream nodes.  
  - *Edge Cases:*  
    - Missing or malformed fields in webhook payload causing empty messages or titles.  
    - SIGNL4 API authentication failure or webhook misconfiguration.  
    - Duplicate alert external IDs causing conflicts.  

- **SIGNL4 Resolve Alert**  
  - *Type & Role:* SIGNL4 node to resolve/close an alert in SIGNL4.  
  - *Configuration:*  
    - Operation set to "resolve".  
    - External ID pulled from `body.objectId` to match and close the correct alert.  
  - *Credentials:* Same SIGNL4 API credentials.  
  - *Inputs/Outputs:* Input from IF node false branch; no downstream nodes.  
  - *Edge Cases:*  
    - Alert external ID not found in SIGNL4 causing resolution failure.  
    - API connectivity issues or authentication errors.  

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                     | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                   |
|------------------------|--------------------|-----------------------------------|--------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Start (Testing)         | Manual Trigger     | Initiates test alert creation      |                          | TheHive Create Alert       |                                                                                              |
| TheHive Create Alert    | TheHive            | Creates a sample alert in TheHive  | Start (Testing)           |                            |                                                                                              |
| TheHive Read Alerts     | TheHive            | Reads all alerts from TheHive      |                          |                            |                                                                                              |
| TheHive Webhook Request | Webhook            | Receives live alerts from TheHive  |                          | IF                         | You need to configure the webhook and notifications in TheHive accordingly.                 |
| IF                     | IF                 | Checks if alert is closed or active| TheHive Webhook Request  | SIGNL4 Send Alert (true)   |                                                                                              |
|                        |                    |                                   |                          | SIGNL4 Resolve Alert (false)|                                                                                              |
| SIGNL4 Send Alert       | SIGNL4             | Sends new alert to SIGNL4          | IF (true branch)          |                            |                                                                                              |
| SIGNL4 Resolve Alert    | SIGNL4             | Resolves alert in SIGNL4           | IF (false branch)         |                            |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Start (Testing)" node:**  
   - Type: Manual Trigger  
   - No parameters needed.  

2. **Add "TheHive Create Alert" node:**  
   - Type: TheHive  
   - Operation: Create alert  
   - Parameters:  
     - Date: 2022-04-25T08:53:18.000Z  
     - Tags: tlp:pwhite  
     - Type: misp  
     - Title: TheHive Alert  
     - Source: 1311  
     - SourceRef: 1330  
     - Description: Security issue detected on server A2. Please check and take care.  
     - Additional fields: none  
   - Credentials: Select or create TheHive API credentials ("The Hive account")  
   - Connect "Start (Testing)" output to this node input.  

3. **Add "TheHive Read Alerts" node:**  
   - Type: TheHive  
   - Operation: getAll  
   - Filters: empty  
   - Options: empty  
   - Credentials: Same TheHive API credentials  
   - No input connections (can be triggered manually or by other means).  

4. **Create "TheHive Webhook Request" node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Set a unique path, e.g., "22c76955-3f52-469e-a8ae-3f62e8e87ebe" (must match TheHive notification configuration)  
   - No authentication configured here; secure via TheHive or network.  

5. **Add "IF" node for alert stage check:**  
   - Type: IF  
   - Condition: Boolean  
     - Value 1: Expression `{{$node["TheHive Webhook Request"].json["body"]["object"]["stage"]}}`  
     - Operation: Not Equal  
     - Value 2: "Closed"  
   - Connect output of "TheHive Webhook Request" to input of IF node.  

6. **Add "SIGNL4 Send Alert" node:**  
   - Type: SIGNL4  
   - Operation: Send alert (default)  
   - Message: Expression `{{$node["TheHive Webhook Request"].json["body"]["details"]["description"]}}`  
   - Additional Fields:  
     - Title: Expression `{{$node["TheHive Webhook Request"].json["body"]["details"]["title"]}}`  
     - External ID: Expression `{{$node["TheHive Webhook Request"].json["body"]["objectId"]}}`  
   - Credentials: Select or create SIGNL4 API credentials ("SIGNL4 Webhook account")  
   - Connect IF node "true" output to this node input.  

7. **Add "SIGNL4 Resolve Alert" node:**  
   - Type: SIGNL4  
   - Operation: Resolve alert  
   - External ID: Expression `{{$node["TheHive Webhook Request"].json["body"]["objectId"]}}`  
   - Credentials: Same SIGNL4 API credentials  
   - Connect IF node "false" output to this node input.  

8. **Configure TheHive to send notifications:**  
   - In TheHive 5, set up notifications to POST to your n8n webhook URL with the exact path configured in "TheHive Webhook Request".  
   - Ensure the payload sent matches the expected JSON structure used in expressions.  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires configuration of TheHive 5 webhook and notifications to forward alerts to n8n.                 | TheHive 5 official docs on Notifications: https://docs.thehive-project.org/thehive/notifications/  |
| SIGNL4 API credentials must be generated and configured in n8n for this workflow to function correctly.               | SIGNL4 API documentation: https://www.signl4.com/api/                                              |
| The unique webhook path in n8n must be kept secret and secured to prevent unauthorized alert injections.              | n8n Webhook node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/                   |
| For large alert volumes, consider adding rate limiting or queue management to avoid API throttling or duplicate alerts.|                                                                                                    |

---

This documentation provides a complete understanding of the "Send TheHive Alerts Using SIGNL4" workflow, enabling reproduction, modification, and troubleshooting of the integration between TheHive and SIGNL4 in n8n.