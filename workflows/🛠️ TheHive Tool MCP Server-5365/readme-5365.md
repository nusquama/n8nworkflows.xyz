🛠️ TheHive Tool MCP Server

https://n8nworkflows.xyz/workflows/----thehive-tool-mcp-server-5365


# 🛠️ TheHive Tool MCP Server

---
### 1. Workflow Overview

This workflow, titled **🛠️ TheHive Tool MCP Server**, serves as an integration layer between an MCP (Managed Chat Platform) trigger and TheHive incident response platform. It is designed to process incoming requests or commands via the MCP interface and perform corresponding operations on TheHive, such as creating logs, retrieving logs, and executing responders.

The logical structure is organized into two main blocks:

- **1.1 MCP Trigger Reception:** Captures incoming requests or commands from the MCP interface.
- **1.2 TheHive Operations:** Executes specific TheHive actions based on the MCP input, such as creating logs, retrieving single or multiple logs, and executing responders.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger Reception

- **Overview:**  
  This block listens for incoming events or commands from the Managed Chat Platform via a specialized MCP trigger node. It acts as the entry point of the workflow, forwarding the received data to TheHive operation nodes.

- **Nodes Involved:**  
  - TheHive Tool MCP Server

- **Node Details:**  
  - **Node Name:** TheHive Tool MCP Server  
  - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP Trigger)  
  - **Technical Role:** Listens for and triggers workflow execution upon receiving an MCP event or command.  
  - **Configuration Choices:** Default configuration with a unique webhookId (`79203b81-fef2-423e-aaa9-356565708c94`) that facilitates external MCP connectivity. No additional parameters configured.  
  - **Key Expressions/Variables:** None explicitly configured; acts as a direct event trigger.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Outputs to the four TheHive operation nodes: "Create a log", "Execute a responder", "Get many logs", and "Get a log" via the `ai_tool` output port.  
  - **Version-Specific Requirements:** Requires n8n version supporting `@n8n/n8n-nodes-langchain` package and the MCP trigger node.  
  - **Potential Failure Types:** Webhook connectivity failure, invalid MCP event payloads, authentication or permission errors on MCP side.

#### 1.2 TheHive Operations

- **Overview:**  
  This block consists of nodes that perform specific actions on TheHive platform based on the MCP input. Each node corresponds to a distinct TheHive API operation: creating logs, retrieving multiple logs, retrieving a single log, and executing responders.

- **Nodes Involved:**  
  - Create a log  
  - Execute a responder  
  - Get many logs  
  - Get a log  

- **Node Details:**  

  - **Node Name:** Create a log  
    - **Type:** `n8n-nodes-base.theHiveTool`  
    - **Technical Role:** Creates a new log entry in TheHive.  
    - **Configuration Choices:** Default node settings with parameters presumably set dynamically at runtime based on the MCP trigger’s input (not detailed here).  
    - **Key Expressions/Variables:** Receives input from "TheHive Tool MCP Server" via the `ai_tool` output.  
    - **Input Connections:** From "TheHive Tool MCP Server" node.  
    - **Output Connections:** None (terminal node).  
    - **Version-Specific Requirements:** Requires TheHive node installed and configured with appropriate credentials.  
    - **Potential Failure Types:** API authentication failure, invalid input data, network timeout, TheHive service unavailability.

  - **Node Name:** Execute a responder  
    - **Type:** `n8n-nodes-base.theHiveTool`  
    - **Technical Role:** Executes a responder action on TheHive, typically to automate incident response steps.  
    - **Configuration Choices:** Default TheHive tool node settings; parameters expected to be set dynamically from MCP input.  
    - **Key Expressions/Variables:** Input from MCP trigger node.  
    - **Input Connections:** From "TheHive Tool MCP Server" node.  
    - **Output Connections:** None (terminal node).  
    - **Version-Specific Requirements:** Same as above.  
    - **Potential Failure Types:** API errors, invalid responder ID or parameters, permission issues.

  - **Node Name:** Get many logs  
    - **Type:** `n8n-nodes-base.theHiveTool`  
    - **Technical Role:** Retrieves multiple log entries from TheHive, likely for querying incident history or audit trails.  
    - **Configuration Choices:** Default TheHive node with parameters possibly set via MCP input.  
    - **Key Expressions/Variables:** Input from MCP trigger node.  
    - **Input Connections:** From "TheHive Tool MCP Server" node.  
    - **Output Connections:** None (terminal node).  
    - **Version-Specific Requirements:** Same as above.  
    - **Potential Failure Types:** Query syntax errors, large data response causing timeout, API limits.

  - **Node Name:** Get a log  
    - **Type:** `n8n-nodes-base.theHiveTool`  
    - **Technical Role:** Retrieves a single log entry by ID from TheHive.  
    - **Configuration Choices:** Parameters to specify log ID expected to be dynamic from MCP input.  
    - **Key Expressions/Variables:** Input from MCP trigger node.  
    - **Input Connections:** From "TheHive Tool MCP Server" node.  
    - **Output Connections:** None (terminal node).  
    - **Version-Specific Requirements:** Same as above.  
    - **Potential Failure Types:** Invalid ID, not found errors, permission errors.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                   | Input Node(s)           | Output Node(s)                  | Sticky Note |
|-----------------------|----------------------------------|---------------------------------|------------------------|-------------------------------|-------------|
| Workflow Overview 0   | stickyNote                       | Informational note (empty)       | None                   | None                          |             |
| TheHive Tool MCP Server | MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`) | Entry point, receives MCP events | None                   | Create a log, Execute a responder, Get many logs, Get a log |             |
| Create a log          | TheHive Tool (`n8n-nodes-base.theHiveTool`) | Creates logs in TheHive          | TheHive Tool MCP Server | None                          |             |
| Execute a responder   | TheHive Tool (`n8n-nodes-base.theHiveTool`) | Executes responders in TheHive   | TheHive Tool MCP Server | None                          |             |
| Get many logs         | TheHive Tool (`n8n-nodes-base.theHiveTool`) | Retrieves multiple logs          | TheHive Tool MCP Server | None                          |             |
| Get a log             | TheHive Tool (`n8n-nodes-base.theHiveTool`) | Retrieves a single log by ID     | TheHive Tool MCP Server | None                          |             |
| Sticky Note 1         | stickyNote                       | Informational note (empty)       | None                   | None                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add a node of type `@n8n/n8n-nodes-langchain.mcpTrigger` and name it `TheHive Tool MCP Server`.  
   - Configure without additional parameters; ensure the webhook ID is set (or auto-generated).  
   - This node will serve as the webhook endpoint to receive MCP events.

2. **Add TheHive Tool Nodes**  
   For each TheHive operation, add a separate node of type `n8n-nodes-base.theHiveTool`:

   - **Create a log**  
     - Name the node `Create a log`.  
     - Configure operation to "Create Log" (exact parameters depend on TheHive API version).  
     - Set credentials for TheHive access (API key or OAuth as required).  
     - Leave parameters to be dynamically set from MCP trigger input or define defaults.

   - **Execute a responder**  
     - Name the node `Execute a responder`.  
     - Configure operation to "Execute Responder".  
     - Set credentials for TheHive.  
     - Accept dynamic parameters for responder ID and inputs.

   - **Get many logs**  
     - Name the node `Get many logs`.  
     - Configure operation to "Get Logs" or equivalent.  
     - Set TheHive credentials.  
     - Parameters may include filters or pagination, configurable dynamically.

   - **Get a log**  
     - Name the node `Get a log`.  
     - Configure to "Get Log by ID".  
     - Set TheHive credentials.  
     - Parameter for log ID to be set dynamically.

3. **Connect Nodes**  
   - From the `TheHive Tool MCP Server` node’s output port `ai_tool`, create connections to all four TheHive Tool nodes: `Create a log`, `Execute a responder`, `Get many logs`, and `Get a log`.

4. **Credential Setup**  
   - Configure credential for TheHive nodes: API key, URL, and any required authentication parameters.  
   - Ensure MCP trigger webhook is publicly accessible or secured as per organizational policy.

5. **Add Sticky Notes** (Optional)  
   - Add sticky notes for documentation or explanation as needed; the original workflow contains empty notes which can be customized.

6. **Test the Workflow**  
   - Deploy and trigger the MCP webhook with sample payloads corresponding to each TheHive operation.  
   - Verify that logs are created, responders execute, and logs can be retrieved as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| TheHive integration nodes require correct API credentials and endpoint configuration.           | TheHive API documentation: https://docs.thehive-project.org/ |
| MCP Trigger requires MCP platform to send properly formatted webhook events for this workflow.  | MCP platform documentation (vendor-specific)    |
| Workflow timezone is set to America/New_York, adjust if needed for deployment region consistency | Workflow settings in n8n                         |
| The workflow is inactive by default and requires activation before use.                         | n8n workflow activation                          |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. Only legal and public data are handled.