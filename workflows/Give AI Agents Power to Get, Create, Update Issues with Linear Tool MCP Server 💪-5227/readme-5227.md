Give AI Agents Power to Get, Create, Update Issues with Linear Tool MCP Server 💪

https://n8nworkflows.xyz/workflows/give-ai-agents-power-to-get--create--update-issues-with-linear-tool-mcp-server----5227


# Give AI Agents Power to Get, Create, Update Issues with Linear Tool MCP Server 💪

### 1. Workflow Overview

This workflow enables AI agents to interact with the Linear issue tracking system through a centralized MCP (Multi-Channel Platform) server node named "Linear Tool MCP Server." It is designed to empower AI-driven automation to create, retrieve, update, and delete issues in Linear seamlessly.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Input Reception:** The "Linear Tool MCP Server" node acts as the entry point, receiving requests from AI agents.
- **1.2 Linear Issue Operations:** Five distinct nodes perform CRUD operations on Linear issues:
  - Create an issue
  - Get an issue
  - Get many issues
  - Update an issue
  - Delete an issue

Each of these nodes is directly connected back to the MCP server node as an AI tool output, facilitating bidirectional communication with AI agents.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Input Reception

- **Overview:**  
  This block receives and routes commands from AI agents related to issue management in Linear. It serves as the central orchestrator node that triggers the appropriate issue operation nodes based on AI input.

- **Nodes Involved:**  
  - Linear Tool MCP Server

- **Node Details:**  

  - **Node Name:** Linear Tool MCP Server  
    - **Type and Role:** MCP Trigger node from the Langchain n8n integration, designed to handle AI agent requests via MCP protocol.  
    - **Configuration:** No additional parameters configured explicitly; it acts as a catch-all trigger for AI tool commands.  
    - **Expressions/Variables:** None specified; acts as a generic entry point.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Connects to all Linear Tool nodes (“Create an issue,” “Get an issue,” “Get many issues,” “Update an issue,” “Delete an issue”) via the “ai_tool” output channel.  
    - **Version Requirements:** Requires n8n version supporting MCP trigger nodes and Langchain integration.  
    - **Failure Modes:** Potential failures include authentication errors if MCP credentials are invalid, network timeouts, or malformed AI requests that do not map to any operation.  
    - **Sub-workflow:** None.

---

#### 2.2 Linear Issue Operations

- **Overview:**  
  This block contains nodes that perform specific CRUD actions on Linear issues. Each node corresponds to a distinct API action against the Linear tool and is invoked via the MCP Server node.

- **Nodes Involved:**  
  - Create an issue  
  - Get an issue  
  - Get many issues  
  - Update an issue  
  - Delete an issue

- **Node Details:**  

  - **Node Name:** Create an issue  
    - **Type and Role:** Linear Tool node; creates a new issue in Linear.  
    - **Configuration:** Default settings, likely requiring issue fields like title, description, project ID, etc., to be passed dynamically.  
    - **Expressions/Variables:** Input data expected from MCP Server via AI agent command.  
    - **Input Connections:** ai_tool input from "Linear Tool MCP Server" node.  
    - **Output Connections:** None (end node).  
    - **Version Requirements:** Requires n8n Linear Tool node version 1 or higher.  
    - **Failure Modes:** Failure if required fields missing, authentication errors, or Linear API rate limiting.  
    - **Sub-workflow:** None.
  
  - **Node Name:** Get an issue  
    - **Type and Role:** Linear Tool node; retrieves details of a single issue by ID.  
    - **Configuration:** Expects issue ID as input from MCP Server.  
    - **Expressions/Variables:** Issue ID dynamically supplied from AI agent input.  
    - **Input Connections:** ai_tool input from "Linear Tool MCP Server" node.  
    - **Output Connections:** None (end node).  
    - **Failure Modes:** Issue not found errors, authentication errors, or network issues.
  
  - **Node Name:** Get many issues  
    - **Type and Role:** Linear Tool node; fetches multiple issues, potentially with filters or pagination.  
    - **Configuration:** Supports parameters such as filters, pagination, query strings dynamically provided by AI agents.  
    - **Input Connections:** ai_tool input from "Linear Tool MCP Server" node.  
    - **Output Connections:** None (end node).  
    - **Failure Modes:** Large result sets causing timeouts, API rate limits.
  
  - **Node Name:** Update an issue  
    - **Type and Role:** Linear Tool node; updates fields of an existing issue.  
    - **Configuration:** Inputs include issue ID and fields to update, received from MCP Server.  
    - **Input Connections:** ai_tool input from "Linear Tool MCP Server" node.  
    - **Output Connections:** None (end node).  
    - **Failure Modes:** Attempting to update non-existent issues, partial updates failing due to invalid fields.
  
  - **Node Name:** Delete an issue  
    - **Type and Role:** Linear Tool node; deletes an existing issue by ID.  
    - **Configuration:** Requires issue ID passed in from MCP Server.  
    - **Input Connections:** ai_tool input from "Linear Tool MCP Server" node.  
    - **Output Connections:** None (end node).  
    - **Failure Modes:** Deleting issues that do not exist, permission errors.

---

### 3. Summary Table

| Node Name            | Node Type                    | Functional Role                        | Input Node(s)          | Output Node(s)                     | Sticky Note |
|----------------------|------------------------------|-------------------------------------|-----------------------|----------------------------------|-------------|
| Workflow Overview 0  | Sticky Note                  | General workflow overview placeholder | None                  | None                             |             |
| Linear Tool MCP Server | MCP Trigger (Langchain)      | Entry point and AI input router     | None                  | Create an issue, Delete an issue, Get an issue, Get many issues, Update an issue |             |
| Create an issue      | Linear Tool Node             | Creates a new Linear issue          | Linear Tool MCP Server | None                             |             |
| Delete an issue      | Linear Tool Node             | Deletes a Linear issue              | Linear Tool MCP Server | None                             |             |
| Get an issue         | Linear Tool Node             | Retrieves one Linear issue          | Linear Tool MCP Server | None                             |             |
| Get many issues      | Linear Tool Node             | Retrieves multiple Linear issues    | Linear Tool MCP Server | None                             |             |
| Update an issue      | Linear Tool Node             | Updates an existing Linear issue    | Linear Tool MCP Server | None                             |             |
| Sticky Note 1        | Sticky Note                  | Placeholder or comment node          | None                  | None                             |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Server Node:**  
   - Add a node of type "MCP Trigger" from the Langchain n8n integration.  
   - Name it "Linear Tool MCP Server."  
   - No additional parameters are required.  
   - This node will serve as the AI agent interface.

2. **Create Linear Tool Nodes for Issue Operations:**  
   For each operation below, add a node of type "Linear Tool" and configure as follows:

   - **Create an issue:**  
     - Name: "Create an issue"  
     - Configure to perform the "Create Issue" action.  
     - Configure input parameters as dynamic, to receive from MCP Server.  
   
   - **Delete an issue:**  
     - Name: "Delete an issue"  
     - Action: "Delete Issue"  
     - Receive issue ID from MCP Server.

   - **Get an issue:**  
     - Name: "Get an issue"  
     - Action: "Get Issue"  
     - Receive issue ID from MCP Server.

   - **Get many issues:**  
     - Name: "Get many issues"  
     - Action: "List Issues"  
     - Configure to accept filters/pagination dynamically.

   - **Update an issue:**  
     - Name: "Update an issue"  
     - Action: "Update Issue"  
     - Receive issue ID and update fields dynamically.

3. **Connect Nodes:**  
   - For each Linear Tool node above, set their input connection from the "ai_tool" output of the "Linear Tool MCP Server" node.  
   - No further outputs are connected as these are terminal operations.

4. **Configure Credentials:**  
   - For all Linear Tool nodes, configure the Linear API credentials with appropriate OAuth2 tokens or API keys.  
   - For the MCP Server node, configure any required MCP credentials or AI access tokens to enable AI agent communication.

5. **Add Sticky Notes (Optional):**  
   - Add sticky notes to document or annotate the workflow as desired.

6. **Set Defaults and Constraints:**  
   - Ensure that issue field requirements are respected (e.g., title is mandatory on create).  
   - Handle error cases like missing fields or invalid issue IDs gracefully in the AI agent logic.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                       |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow uses the Langchain MCP Trigger node for AI agent integration with Linear Tool operations. | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain.mcpTrigger/ |
| Ensure Linear API credentials have sufficient permissions for issue CRUD operations.                     | https://linear.app/docs/api/                                          |
| To extend this workflow, consider adding nodes for comments, labels, or project management in Linear.   | Linear API documentation                                              |

---

**Disclaimer:**  
The text provided is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and includes no illegal, offensive, or protected material. All data handled is legal and public.