🛠️ ProfitWell Tool MCP Server 💪 both operations

https://n8nworkflows.xyz/workflows/----profitwell-tool-mcp-server----both-operations-5092


# 🛠️ ProfitWell Tool MCP Server 💪 both operations

### 1. Workflow Overview

This workflow, titled **🛠️ ProfitWell Tool MCP Server**, is designed to serve as a Multi-Channel Platform (MCP) server for ProfitWell Tool operations through n8n. It exposes two primary operations via a webhook endpoint:

- **Company**: Fetches settings related to a company.
- **Metric**: Retrieves specific metrics with flexible parameters.

The workflow is structured into three logical blocks:

- **1.1 MCP Trigger Reception:** Listens for incoming MCP requests on a dedicated webhook path.
- **1.2 Company Data Retrieval:** Handles requests to get company settings.
- **1.3 Metric Retrieval:** Handles requests to get metric data based on AI-provided parameters.

This design allows AI agents or other clients to interact with ProfitWell Tool operations seamlessly, with all parameters populated dynamically via `$fromAI()` expressions, supporting zero-configuration operation out-of-the-box.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger Reception

- **Overview:**  
  This block sets up the entry point for all incoming MCP requests by defining a webhook listener node. It acts as the gateway, routing requests to the appropriate operation nodes.

- **Nodes Involved:**  
  - ProfitWell Tool MCP Server (MCP Trigger node)

- **Node Details:**

  - **ProfitWell Tool MCP Server**  
    - **Type & Role:** `@n8n/n8n-nodes-langchain.mcpTrigger` — Listens for HTTP requests on a specified webhook path and dispatches them as MCP tool operations.  
    - **Configuration:**  
      - Webhook path: `profitwell-tool-mcp`  
      - Webhook ID: fixed UUID (used internally by n8n)  
    - **Input / Output:**  
      - Input: HTTP requests from MCP clients.  
      - Output: MCP operation requests, routed to operation nodes via the `ai_tool` output.  
    - **Version-specific Requirements:** Requires n8n version supporting MCP trigger node and Langchain integration.  
    - **Potential Failures:**  
      - Webhook URL misconfiguration or network issues causing unavailability.  
      - Authentication or permission errors if credentials are missing or invalid (handled downstream).  
    - **Sub-Workflow:** Not applicable.

---

#### 1.2 Company Data Retrieval

- **Overview:**  
  This block processes requests to retrieve company-level settings from the ProfitWell Tool API.

- **Nodes Involved:**  
  - Get settings for your company (ProfitWell Tool node)  
  - Sticky Note 1 (annotation only)

- **Node Details:**

  - **Get settings for your company**  
    - **Type & Role:** `n8n-nodes-base.profitWellTool` — Executes the "company" resource operation to fetch company settings.  
    - **Configuration:**  
      - Resource set to `company` (fixed, no dynamic parameters).  
      - Credentials: ProfitWell Tool credentials must be configured here for API access.  
    - **Input / Output:**  
      - Input: MCP trigger node’s `ai_tool` output connection.  
      - Output: Company settings data returned to MCP client.  
    - **Expressions:** None; static resource selection.  
    - **Version-specific Requirements:** Requires n8n ProfitWell Tool node version supporting "company" resource.  
    - **Potential Failures:**  
      - Credential authentication failure.  
      - API rate limits or downtime.  
      - Network timeouts.  
    - **Sub-Workflow:** None.

  - **Sticky Note 1**  
    - Color-coded annotation labeled "Company" to clarify this block’s purpose. No runtime effect.

---

#### 1.3 Metric Retrieval

- **Overview:**  
  This block manages requests to retrieve metrics data from the ProfitWell Tool API with flexible parameters dynamically supplied by AI agents.

- **Nodes Involved:**  
  - Get a metric (ProfitWell Tool node)  
  - Sticky Note 2 (annotation only)

- **Node Details:**

  - **Get a metric**  
    - **Type & Role:** `n8n-nodes-base.profitWellTool` — Executes the "metric" resource operation with dynamic parameters.  
    - **Configuration:**  
      - Resource: `metric` (fixed)  
      - Parameters dynamically injected from AI via expressions:  
        - `type`: `{{$fromAI('Type', '', 'string')}}`  
        - `month`: `{{$fromAI('Month', '', 'string')}}`  
        - `simple`: `{{$fromAI('Simple', '', 'boolean')}}`  
      - Options: empty by default but can be extended.  
      - Credentials: ProfitWell Tool credentials to be configured.  
    - **Input / Output:**  
      - Input: MCP trigger node’s `ai_tool` output connection.  
      - Output: Metric data returned to MCP client.  
    - **Expressions:** Uses `$fromAI()` to automatically populate parameters based on AI agent context.  
    - **Version-specific Requirements:** Requires n8n ProfitWell Tool node version supporting “metric” resource and $fromAI expression.  
    - **Potential Failures:**  
      - Parameter parsing or type mismatches if AI provides invalid data.  
      - Credential authentication issues.  
      - API call failures (rate limits, invalid metric types).  
    - **Sub-Workflow:** None.

  - **Sticky Note 2**  
    - Color-coded annotation labeled "Metric" for clarity.

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                     | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                          |
|------------------------------|-----------------------------------------|-----------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| Workflow Overview 0           | Sticky Note                             | Documentation / Overview          | —                           | —                         | ## 🛠️ ProfitWell Tool MCP Server... See full content in workflow overview section above.          |
| ProfitWell Tool MCP Server    | MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`) | Entry point webhook for MCP requests | —                           | Get settings for your company, Get a metric (via `ai_tool` output) |                                                                                                    |
| Get settings for your company | ProfitWell Tool node (`n8n-nodes-base.profitWellTool`) | Fetch company settings from API  | ProfitWell Tool MCP Server  | ProfitWell Tool MCP Server (response) |                                                                                                    |
| Sticky Note 1                | Sticky Note                             | Annotation: Company block         | —                           | —                         | ## Company                                                                                         |
| Get a metric                 | ProfitWell Tool node (`n8n-nodes-base.profitWellTool`) | Fetch metric data with AI parameters | ProfitWell Tool MCP Server  | ProfitWell Tool MCP Server (response) |                                                                                                    |
| Sticky Note 2                | Sticky Note                             | Annotation: Metric block          | —                           | —                         | ## Metric                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note for Overview:**  
   - Add a Sticky Note node.  
   - Set width to 420, height to 800.  
   - Paste the provided overview content describing operations, setup instructions, features, and help links.

2. **Add MCP Trigger Node:**  
   - Create a node of type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Configure Webhook Path: `profitwell-tool-mcp`.  
   - Ensure your n8n instance supports MCP triggers (requires Langchain integration).  
   - This node will listen to HTTP requests from AI agents or clients.

3. **Create “Get settings for your company” Node:**  
   - Add a node of type `ProfitWell Tool` (`n8n-nodes-base.profitWellTool`).  
   - Set **Resource** to `company`.  
   - No additional parameters are needed.  
   - Configure ProfitWell Tool API credentials here (API key or OAuth as per your setup).  
   - Connect the MCP Trigger node’s `ai_tool` output to this node’s input.

4. **Add Sticky Note for Company Block:**  
   - Create a Sticky Note node.  
   - Set color to a distinguishable color (e.g., color code 4).  
   - Enter content: `## Company`.  
   - Position next to the “Get settings for your company” node.

5. **Create “Get a metric” Node:**  
   - Add another `ProfitWell Tool` node.  
   - Set **Resource** to `metric`.  
   - For parameters:  
     - Set `type` to `{{$fromAI('Type', '', 'string')}}`  
     - Set `month` to `{{$fromAI('Month', '', 'string')}}`  
     - Set `simple` to `{{$fromAI('Simple', '', 'boolean')}}`  
   - Options remain default or empty.  
   - Configure the same ProfitWell Tool credentials as above.  
   - Connect the MCP Trigger node’s `ai_tool` output to this node’s input.

6. **Add Sticky Note for Metric Block:**  
   - Create a Sticky Note node.  
   - Set color to a different distinguishable color (e.g., color code 5).  
   - Enter content: `## Metric`.  
   - Position next to the “Get a metric” node.

7. **Finalize Connections and Activate:**  
   - Verify connections: MCP Trigger node outputs `ai_tool` to both operation nodes independently.  
   - Save the workflow.  
   - Activate the workflow to expose the webhook endpoint.

8. **Credentials Setup:**  
   - In n8n settings, add or configure ProfitWell Tool credentials (API key or OAuth2).  
   - Attach these credentials to both ProfitWell Tool nodes.

9. **Testing and Usage:**  
   - Use the webhook URL `https://<your-n8n-domain>/webhook/profitwell-tool-mcp` as the MCP URL in your AI agent configuration.  
   - Parameters for the metric operation will be auto-populated using AI context via `$fromAI()` expressions.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow supports zero-configuration usage with pre-built operations and AI-driven parameter injection.       | Workflow overview sticky note and setup instructions.                                                           |
| For detailed MCP integration guidance or customizations, contact via Discord: https://discord.me/cfomodz     | Provided in workflow overview sticky note.                                                                       |
| Official n8n documentation on MCP nodes and Langchain tool integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Useful for troubleshooting and extending MCP workflows.                                                         |

---

This documentation enables advanced users and automation agents to fully understand, reproduce, and maintain the **ProfitWell Tool MCP Server** workflow with clear insights into its structure, configurations, and operational logic.