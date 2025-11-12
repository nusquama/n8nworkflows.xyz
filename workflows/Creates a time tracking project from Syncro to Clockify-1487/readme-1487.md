Creates a time tracking project from Syncro to Clockify

https://n8nworkflows.xyz/workflows/creates-a-time-tracking-project-from-syncro-to-clockify-1487


# Creates a time tracking project from Syncro to Clockify

### 1. Workflow Overview

This workflow automates the creation of a time tracking project in Clockify based on ticket creation events received from Syncro. When a new ticket is created in Syncro, a webhook triggers this workflow, which extracts relevant ticket information and creates a corresponding project in Clockify. This enables users to track time against the newly created project seamlessly.

Logical blocks:

- **1.1 Input Reception:** Receives incoming ticket creation data from Syncro via a webhook.
- **1.2 Project Creation:** Uses the received data to create a new project in Clockify with a descriptive name.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block sets up a webhook to capture ticket creation notifications from Syncro. It listens for POST requests containing ticket data and passes this data downstream.

**Nodes Involved:**  
- Webhook

**Node Details:**

- **Webhook**  
  - Type: Webhook node (HTTP listener)  
  - Configuration:  
    - HTTP Method: POST  
    - Path: `43d196b0-63c4-440a-aaf6-9d893907cf3c` (unique webhook endpoint)  
    - Response Mode: Returns the data from the last node (here itself) as the response  
  - Key expressions/variables: None (simply receives JSON payload)  
  - Input Connections: None (entry point of the workflow)  
  - Output Connections: Connects to the Clockify node  
  - Version-specific requirements: Version 1; ensure n8n version supports webhook with these parameters  
  - Potential failures:  
    - Misconfiguration of webhook path or method causing failure to receive events  
    - Network issues blocking POST requests from Syncro  
    - Payload format changes in Syncro breaking downstream processing  
  - Sub-workflow: None

#### 1.2 Project Creation

**Overview:**  
This block creates a new project in Clockify using details extracted from the Syncro ticket data received via the webhook. The project name is dynamically composed from the ticket number, customer business or name, and ticket ID.

**Nodes Involved:**  
- Clockify

**Node Details:**

- **Clockify**  
  - Type: Clockify node (API integration)  
  - Configuration:  
    - Action: Create Project  
    - Project Name: Constructed dynamically as  
      `Ticket {ticket number} - {customer business or name} [{ticket id}]`  
      Specifically:  
      `Ticket {{$json["body"]["attributes"]["number"]}} - {{$json["body"]["attributes"]["customer_business_then_name"]}} [{{$json["body"]["attributes"]["id"]}}]`  
    - Workspace ID: Fixed string `"xxx"` (placeholder; must be replaced with actual Clockify workspace ID)  
    - Additional Fields: None  
  - Credentials: Uses Clockify API credentials configured under "Clockify"  
  - Input Connections: Receives data from Webhook node  
  - Output Connections: None (end of workflow)  
  - Version-specific requirements: Version 1; ensure Clockify node supports dynamic expressions as used  
  - Potential failures:  
    - Invalid or missing Clockify API credentials causing auth errors  
    - Invalid workspace ID ("xxx" placeholder must be replaced) causing API errors  
    - Missing or unexpected ticket fields breaking expression evaluation  
    - API rate limits or network timeouts  
  - Sub-workflow: None

---

### 3. Summary Table

| Node Name | Node Type | Functional Role          | Input Node(s) | Output Node(s) | Sticky Note                                                                                       |
|-----------|-----------|--------------------------|---------------|----------------|-------------------------------------------------------------------------------------------------|
| Webhook   | Webhook   | Receives Syncro ticket data via POST webhook | None          | Clockify       | This workflow creates a project in Clockify for any user to track time against. Syncro must be setup with a webhook via Notification Set for Ticket - created (for anyone). The original workflow can be found here: https://github.com/bionemesis/n8nsyncro |
| Clockify  | Clockify  | Creates a new project in Clockify using ticket info | Webhook       | None           |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Node Type: Webhook  
   - Parameters:  
     - HTTP Method: POST  
     - Path: Use a unique identifier for the webhook path, e.g., `43d196b0-63c4-440a-aaf6-9d893907cf3c`  
     - Response Mode: `lastNode`  
   - Purpose: To receive POST requests from Syncro when a ticket is created  
   - No credentials needed  
   - Place this node as the first node (entry point)

2. **Create Clockify Node**  
   - Node Type: Clockify  
   - Credentials: Setup Clockify API credentials in n8n under the name `"Clockify"` with a valid API key  
   - Parameters:  
     - Workspace ID: Replace `"xxx"` with your actual Clockify workspace ID  
     - Project Name: Use the following expression to dynamically build the project name:  
       ```
       Ticket {{$json["body"]["attributes"]["number"]}} - {{$json["body"]["attributes"]["customer_business_then_name"]}} [{{$json["body"]["attributes"]["id"]}}]
       ```  
     - Additional Fields: Leave empty unless customization is needed  
   - Connect the input of this node to the output of the Webhook node

3. **Connect Nodes**  
   - Connect the Webhook node’s main output to the Clockify node’s main input

4. **Activate Webhook Integration in Syncro**  
   - In Syncro, configure a webhook (Notification Set) for the event "Ticket - created" to POST to the webhook URL provided by n8n (e.g., `https://your-n8n-instance/webhook/43d196b0-63c4-440a-aaf6-9d893907cf3c`)

5. **Test the Workflow**  
   - Create a ticket in Syncro  
   - Confirm that a new project with the correct name appears in Clockify under the specified workspace  

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                               |
|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow is part of an MSP (Managed Service Provider) collection for Syncro and Clockify integration. | https://github.com/bionemesis/n8nsyncro                       |
| Ensure your Clockify workspace ID is correctly set to avoid creation errors.                      | Internal configuration note                                   |
| Syncro webhook setup requires the Notification Set for the "Ticket - created" event.             | Syncro platform webhook configuration                          |

---

This documentation provides a comprehensive understanding of the “Syncro to Clockify” workflow, enabling reproduction, modification, and error anticipation for efficient time tracking project automation.