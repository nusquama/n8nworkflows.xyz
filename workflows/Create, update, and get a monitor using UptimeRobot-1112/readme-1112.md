Create, update, and get a monitor using UptimeRobot

https://n8nworkflows.xyz/workflows/create--update--and-get-a-monitor-using-uptimerobot-1112


# Create, update, and get a monitor using UptimeRobot

### 1. Workflow Overview

This n8n workflow automates the lifecycle management of an UptimeRobot monitor for HTTP(S) website monitoring. It performs three sequential actions:

- **1.1 Create Monitor:** Creates a new HTTP(S) monitor for a given URL (`https://n8n.io`) with a specified friendly name.
- **1.2 Update Monitor:** Updates the friendly name of the newly created monitor.
- **1.3 Get Monitor Info:** Retrieves detailed information about the updated monitor.

The workflow is linear and depends on passing the monitor ID from one step to the next, ensuring the exact monitor created is updated and then queried.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Create Monitor

- **Overview:**  
  This block creates a new HTTP(S) type monitor in UptimeRobot for the URL `https://n8n.io` with the friendly name "n8n". It initiates the workflow by generating the monitor and outputting its unique ID for subsequent operations.

- **Nodes Involved:**  
  - `UptimeRobot`

- **Node Details:**  
  - **Type and Technical Role:**  
    UptimeRobot node — creates a new monitor resource.
  
  - **Configuration Choices:**  
    - Operation: `create`  
    - Resource: `monitor`  
    - URL: `https://n8n.io`  
    - Type: `HTTP(S)` (represented by `1`)  
    - Friendly Name: `"n8n"`
  
  - **Key Expressions or Variables Used:**  
    None explicitly set; static values configured.
  
  - **Input and Output Connections:**  
    - Input: None (start node)  
    - Output: Passes monitor creation result including the monitor ID to `UptimeRobot1`.
  
  - **Version-specific Requirements:**  
    Requires valid UptimeRobot API credentials configured in n8n.
  
  - **Edge Cases / Potential Failures:**  
    - API authentication failure or invalid credentials  
    - Invalid URL or malformed request  
    - API rate limiting or downtime  
    - Network timeouts

  - **Sub-workflow Reference:**  
    None

---

#### Block 1.2: Update Monitor

- **Overview:**  
  This block updates the friendly name of the monitor created in Block 1.1. It receives the monitor ID dynamically from the previous node output and renames the monitor to "n8n website".

- **Nodes Involved:**  
  - `UptimeRobot1`

- **Node Details:**  
  - **Type and Technical Role:**  
    UptimeRobot node — updates an existing monitor resource.
  
  - **Configuration Choices:**  
    - Operation: `update`  
    - Resource: `monitor`  
    - ID: dynamic expression `={{$json["id"]}}` — passed from output of `UptimeRobot` node  
    - Update Fields: `friendly_name` set to `"n8n website"`
  
  - **Key Expressions or Variables Used:**  
    - `{{$json["id"]}}` to retrieve the monitor ID from the previous node’s output.
  
  - **Input and Output Connections:**  
    - Input: From `UptimeRobot` node  
    - Output: Passes updated monitor data including ID to `UptimeRobot2`.
  
  - **Version-specific Requirements:**  
    Requires consistent API credential usage and valid monitor ID.
  
  - **Edge Cases / Potential Failures:**  
    - Invalid or missing monitor ID causing update failure  
    - API authentication failure  
    - Update conflicts or validation errors if friendly name not accepted  
    - Network or API timeouts
  
  - **Sub-workflow Reference:**  
    None

---

#### Block 1.3: Get Monitor Info

- **Overview:**  
  This block retrieves the current information of the monitor after the update operation. It uses the monitor ID passed forward to ensure the correct monitor is queried.

- **Nodes Involved:**  
  - `UptimeRobot2`

- **Node Details:**  
  - **Type and Technical Role:**  
    UptimeRobot node — retrieves ("get") monitor resource information.
  
  - **Configuration Choices:**  
    - Operation: `get`  
    - Resource: `monitor`  
    - ID: dynamic expression `={{$json["id"]}}` — from previous node’s output
  
  - **Key Expressions or Variables Used:**  
    - `{{$json["id"]}}` to dynamically fetch monitor ID.
  
  - **Input and Output Connections:**  
    - Input: From `UptimeRobot1` node  
    - Output: Final output of the workflow with monitor details.
  
  - **Version-specific Requirements:**  
    API credentials required and valid monitor ID necessary.
  
  - **Edge Cases / Potential Failures:**  
    - Invalid monitor ID causing retrieval failure  
    - API authentication issues  
    - Network or API errors leading to timeouts or empty responses
  
  - **Sub-workflow Reference:**  
    None

---

### 3. Summary Table

| Node Name      | Node Type                | Functional Role         | Input Node(s)  | Output Node(s) | Sticky Note                                  |
|----------------|--------------------------|------------------------|----------------|----------------|----------------------------------------------|
| UptimeRobot    | UptimeRobot (create)     | Create HTTP(S) monitor | None           | UptimeRobot1   | This node creates a new monitor of type HTTP(S). |
| UptimeRobot1   | UptimeRobot (update)     | Update monitor name    | UptimeRobot    | UptimeRobot2   | This node updates the monitor created in the previous node. |
| UptimeRobot2   | UptimeRobot (get)        | Get monitor info       | UptimeRobot1   | None           | This node retrieves information about the monitor created earlier. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Create the first node:**
   - Add a node of type **UptimeRobot**.
   - Configure parameters:
     - Resource: `monitor`
     - Operation: `create`
     - URL: `https://n8n.io`
     - Type: `1` (HTTP(S))
     - Friendly Name: `n8n`
   - Assign credentials:
     - Select or create **UptimeRobot API Credentials** with your API key.
   - Position this node as the starting point.

3. **Create the second node:**
   - Add another **UptimeRobot** node.
   - Configure parameters:
     - Resource: `monitor`
     - Operation: `update`
     - ID: Use expression editor to set `{{$json["id"]}}` (this dynamically grabs the monitor ID output from the previous node)
     - Update Fields:
       - Friendly Name: `n8n website`
   - Assign the same **UptimeRobot API Credentials**.
   - Connect the output of the first node (`UptimeRobot`) to the input of this second node.

4. **Create the third node:**
   - Add another **UptimeRobot** node.
   - Configure parameters:
     - Resource: `monitor`
     - Operation: `get`
     - ID: Use expression editor `{{$json["id"]}}` to pull the monitor ID dynamically.
   - Assign the same **UptimeRobot API Credentials**.
   - Connect the output of the second node (`UptimeRobot1`) to this node's input.

5. **Save and activate the workflow.**

6. **Run the workflow manually** or trigger it as needed.  
   - The workflow will create a monitor, update its friendly name, then retrieve and output the monitor details.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The workflow uses the UptimeRobot node which requires an API key credential setup in n8n for authentication. | UptimeRobot API: https://uptimerobot.com/api/ |
| Monitor type `1` corresponds to HTTP(S) monitors as per UptimeRobot documentation. | https://uptimerobot.com/api/ |
| Friendly names help identify monitors in UptimeRobot dashboard; updating them allows dynamic renaming. | UptimeRobot Dashboard UI |
| Always check API rate limits and error messages in n8n execution logs when troubleshooting API failures. | n8n Execution Logs |

---

This reference document provides a complete, clear understanding of the workflow structure, node functions, and configuration necessary to implement, modify, or troubleshoot the UptimeRobot monitor management automation in n8n.