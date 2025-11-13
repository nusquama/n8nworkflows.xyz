🛠️ CircleCI Tool MCP Server

https://n8nworkflows.xyz/workflows/----circleci-tool-mcp-server-5325


# 🛠️ CircleCI Tool MCP Server

### 1. Workflow Overview

This workflow, titled **"🛠️ CircleCI Tool MCP Server"**, serves as a backend automation server that exposes CircleCI pipeline operations through a Model-Connected-Process (MCP) interface. It is designed to integrate with AI agents that invoke CircleCI operations dynamically by passing parameters via AI expressions.

**Target Use Cases:**  
- Automate retrieval and triggering of CircleCI pipelines via an AI-driven interface.  
- Provide a zero-configuration server where AI agents can request CircleCI pipeline data or trigger pipelines without manual input.  
- Offer native error handling and response formatting for CircleCI API interactions.  

**Logical Blocks:**  
- **1.1 Workflow Overview and Instructions:** Documentation and user guidance via sticky note.  
- **1.2 MCP Trigger Reception:** Handles incoming MCP requests from AI agents or other clients.  
- **1.3 CircleCI Pipeline Operations:** Three distinct nodes implement the operations:  
  - Get a single pipeline  
  - Get many pipelines (list)  
  - Trigger a pipeline  

---

### 2. Block-by-Block Analysis

#### 1.1 Workflow Overview and Instructions

- **Overview:**  
  This block provides an embedded documentation note within the workflow, explaining its purpose, setup instructions, available operations, and support resources. It serves as an in-workflow reference for users importing or modifying the workflow.

- **Nodes Involved:**  
  - `Workflow Overview 0` (Sticky Note)

- **Node Details:**  
  - **Type:** Sticky Note  
  - **Configuration:** Large note (420x780 px) containing Markdown-formatted text that explains:  
    - Available CircleCI operations (get, get all, trigger pipelines)  
    - Setup instructions including credential configuration and webhook URL usage  
    - Highlighted features like zero configuration, AI parameter population, and error handling  
    - Links to official n8n documentation and Discord for support  
  - **Inputs/Outputs:** None (informational only)  
  - **Edge Cases:** None (non-executable node)  

#### 1.2 MCP Trigger Reception

- **Overview:**  
  This node acts as the entry point of the workflow, waiting for incoming MCP requests on a specific webhook path. It receives AI-driven requests and passes them downstream for processing.

- **Nodes Involved:**  
  - `CircleCI Tool MCP Server` (MCP Trigger node)

- **Node Details:**  
  - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - **Role:** Listens for HTTP webhook calls on path `/circleci-tool-mcp`, initiating the workflow upon requests.  
  - **Configuration:**  
    - `path`: "circleci-tool-mcp"  
    - `webhookId`: Unique identifier for webhook management  
  - **Inputs:** External HTTP requests formatted as MCP calls  
  - **Outputs:** Passes data to CircleCI operation nodes via AI tool connections  
  - **Version Requirements:** Requires n8n version supporting MCP trigger nodes and LangChain integration  
  - **Potential Failures:**  
    - Webhook not reachable if server offline  
    - Authentication failures if MCP requests lack proper tokens (depending on setup)  
    - Malformed MCP requests could cause expression errors downstream  

#### 1.3 CircleCI Pipeline Operations

- **Overview:**  
  Three nodes perform specific CircleCI API operations, each configured to dynamically receive parameters from AI expressions. They share a common credential configuration for CircleCI API access.

- **Nodes Involved:**  
  - `Get a pipeline`  
  - `Get many pipelines`  
  - `Trigger a pipeline`  
  - `Sticky Note 1` (annotational, related to pipelines)

- **Node Details:**  

  1. **Get a pipeline**  
     - **Type:** `n8n-nodes-base.circleCiTool`  
     - **Role:** Retrieves details about a specific CircleCI pipeline by its number.  
     - **Configuration:**  
       - VCS (Version Control System) specified dynamically via AI: `$fromAI('Vcs')` (string)  
       - Project slug dynamically via AI: `$fromAI('Project_Slug')` (string)  
       - Pipeline number dynamically via AI: `$fromAI('Pipeline_Number')` (number)  
       - Operation implicitly "get" (default for this node)  
     - **Credentials:** Uses CircleCI API credentials (placeholder ID to be replaced)  
     - **Inputs:** Receives MCP trigger data via AI tool connection  
     - **Outputs:** CircleCI pipeline data JSON output  
     - **Failures:**  
       - Authentication errors if credentials misconfigured  
       - Invalid pipeline number or project slug causing API errors  
       - Expression evaluation failures if AI input missing or malformed  

  2. **Get many pipelines**  
     - **Type:** `n8n-nodes-base.circleCiTool`  
     - **Role:** Retrieves a list of pipelines for a given project.  
     - **Configuration:**  
       - VCS, project slug from AI expressions  
       - `limit` to control number of pipelines fetched, from AI input  
       - `returnAll` boolean from AI input to decide if all pipelines should be returned  
       - Empty filters object (no filtering by default)  
       - Operation explicitly set to `"getAll"`  
     - **Credentials:** CircleCI API credentials (same as above)  
     - **Inputs:** MCP trigger data  
     - **Outputs:** List of pipelines in JSON array  
     - **Failures:**  
       - API rate limits or auth errors  
       - Invalid expressions for limit or returnAll parameters  
       - Empty or invalid project slug or VCS  

  3. **Trigger a pipeline**  
     - **Type:** `n8n-nodes-base.circleCiTool`  
     - **Role:** Triggers a new pipeline for a given project.  
     - **Configuration:**  
       - VCS and project slug from AI inputs  
       - Operation set to `"trigger"`  
       - Additional fields empty (can be configured as needed)  
     - **Credentials:** CircleCI API credentials  
     - **Inputs:** MCP trigger data  
     - **Outputs:** Response from CircleCI API confirming pipeline trigger  
     - **Failures:**  
       - Unauthorized errors due to credential issues  
       - Operation failures if project slug or VCS invalid  
       - Timeouts or API downtime  

  4. **Sticky Note 1**  
     - **Type:** Sticky Note  
     - **Content:** Single word "Pipeline" as a visual label for the pipeline operation nodes grouping  
     - **Inputs/Outputs:** None  

---

### 3. Summary Table

| Node Name             | Node Type                       | Functional Role         | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                         |
|-----------------------|--------------------------------|------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow Overview 0   | Sticky Note                    | Documentation & Setup  | None                        | None                          | ## 🛠️ CircleCI Tool MCP Server<br>Setup instructions and features with helpful links              |
| CircleCI Tool MCP Server | MCP Trigger (`mcpTrigger`)      | Entry point / trigger  | None                        | Get a pipeline, Get many pipelines, Trigger a pipeline |                                                                                                   |
| Get a pipeline        | CircleCI Tool Node             | Get single pipeline    | CircleCI Tool MCP Server    | None                          |                                                                                                   |
| Get many pipelines    | CircleCI Tool Node             | List pipelines         | CircleCI Tool MCP Server    | None                          |                                                                                                   |
| Trigger a pipeline    | CircleCI Tool Node             | Trigger pipeline       | CircleCI Tool MCP Server    | None                          |                                                                                                   |
| Sticky Note 1         | Sticky Note                    | Annotation             | None                        | None                          | ## Pipeline                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note "Workflow Overview 0":**  
   - Type: Sticky Note  
   - Content: Paste the detailed markdown describing the workflow, setup instructions, available operations, and support links (see section 2.1).  
   - Position: Place left side for visibility (e.g., X: -1460, Y: -220)  
   - Size: Width 420, Height 780  

2. **Add MCP Trigger Node "CircleCI Tool MCP Server":**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Path: `circleci-tool-mcp`  
     - Webhook ID: auto-generated or use a unique ID  
   - Position: Center-left (e.g., X: -440, Y: -160)  
   - No credentials required for trigger itself  

3. **Add CircleCI Tool Node "Get a pipeline":**  
   - Type: `n8n-nodes-base.circleCiTool`  
   - Parameters:  
     - `vcs`: `={{ $fromAI('Vcs', '', 'string') }}`  
     - `projectSlug`: `={{ $fromAI('Project_Slug', '', 'string') }}`  
     - `pipelineNumber`: `={{ $fromAI('Pipeline_Number', '', 'number') }}`  
     - Operation defaults to "get"  
   - Credentials: Assign CircleCI API credentials (OAuth or Personal Token)  
   - Position: Left-middle (e.g., X: -800, Y: 140)  

4. **Add CircleCI Tool Node "Get many pipelines":**  
   - Type: `n8n-nodes-base.circleCiTool`  
   - Parameters:  
     - `vcs`: `={{ $fromAI('Vcs', '', 'string') }}`  
     - `projectSlug`: `={{ $fromAI('Project_Slug', '', 'string') }}`  
     - `limit`: `={{ $fromAI('Limit', '', 'number') }}`  
     - `returnAll`: `={{ $fromAI('Return_All', '', 'boolean') }}`  
     - `filters`: `{}` (empty object)  
     - Operation: `"getAll"`  
   - Credentials: Same CircleCI API credentials  
   - Position: Center (e.g., X: -580, Y: 140)  

5. **Add CircleCI Tool Node "Trigger a pipeline":**  
   - Type: `n8n-nodes-base.circleCiTool`  
   - Parameters:  
     - `vcs`: `={{ $fromAI('Vcs', '', 'string') }}`  
     - `projectSlug`: `={{ $fromAI('Project_Slug', '', 'string') }}`  
     - Operation: `"trigger"`  
     - Additional fields: leave empty or configure as needed  
   - Credentials: Same CircleCI API credentials  
   - Position: Right-middle (e.g., X: -360, Y: 140)  

6. **Connect MCP Trigger output to each CircleCI Tool node:**  
   - Draw connections from the `CircleCI Tool MCP Server` node to each of the three CircleCI nodes (`Get a pipeline`, `Get many pipelines`, `Trigger a pipeline`) using the AI tool output connection.

7. **Add Sticky Note "Pipeline" for annotation:**  
   - Type: Sticky Note  
   - Content: "## Pipeline"  
   - Position: Near the three CircleCI nodes (e.g., X: -1000, Y: 120)  
   - Size: Width 840, Height 180  

8. **Credential Setup:**  
   - Create CircleCI API credentials in n8n (Personal API Token or OAuth2) and assign to each CircleCI Tool node.  
   - Replace placeholder credential IDs with actual credential references.  

9. **Activate Workflow:**  
   - Enable the workflow to start listening on the MCP webhook path.  
   - Share or use the webhook URL (`/circleci-tool-mcp`) in AI agent configurations to invoke CircleCI pipeline operations dynamically.  

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                                       |
|--------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Zero configuration: all 3 CircleCI operations pre-built and ready to use with AI parameter expressions.            | See Sticky Note "Workflow Overview 0"                                                                                 |
| AI agents automatically populate parameters using `$fromAI()` expressions, streamlining input without manual entry.| Sticky Note "Workflow Overview 0"                                                                                     |
| Native n8n error handling and response formatting applied to CircleCI API responses.                              | Sticky Note "Workflow Overview 0"                                                                                     |
| Modify parameter defaults in any CircleCI Tool node as needed for custom workflows or specific project requirements.| Sticky Note "Workflow Overview 0"                                                                                     |
| For detailed MCP integration guidance and customizations, consult the official n8n documentation and Discord support.| [n8n MCP Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) <br> [Discord](https://discord.me/cfomodz) |

---

**Disclaimer:**  
The text provided is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.