[eBay] Recommendation API MCP Server

https://n8nworkflows.xyz/workflows/-ebay--recommendation-api-mcp-server-5577


# [eBay] Recommendation API MCP Server

### 1. Workflow Overview

This workflow, titled **[eBay] Recommendation API MCP Server**, serves as a Microservice Control Plane (MCP) server interface that exposes eBay’s Recommendation API to AI agents. It specifically supports the **findListingRecommendations** method to provide sellers with insights on how to optimize their Promoted Listings campaigns on eBay. The workflow is designed to receive AI agent requests via an MCP trigger, forward these requests to the eBay API, and return the API response directly to the AI agent, maintaining the original API response structure.

The workflow is organized into the following logical blocks:

- **1.1 Setup and Documentation:** Contains sticky notes with setup instructions and an overview of the workflow functionality.
- **1.2 MCP Server Trigger:** An MCP trigger node that acts as the server endpoint accepting incoming AI agent requests.
- **1.3 API Request Execution:** An HTTP Request Tool node that issues POST requests to the eBay Recommendation API endpoint, dynamically populating parameters using AI expressions.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Documentation

- **Overview:** This block provides users with essential setup instructions, an overview of the workflow's purpose, available API operations, and usage notes. It ensures users understand how to configure and deploy the workflow.
- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  
  - Sticky Note ("Listing Recommendation")

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Provides detailed step-by-step instructions on importing, configuring authentication, activating the workflow, obtaining the MCP URL, and connecting AI agents.  
    - Content Highlights:  
      - Import workflow into n8n.  
      - Configure OAuth2 credentials for authentication.  
      - Activate workflow to start MCP server.  
      - Copy webhook URL for AI agent use.  
      - Notes on auto-populating parameters using `$fromAI()` expressions.  
      - Suggestions for customization and error handling.  
      - Support link to Discord and n8n documentation for assistance.  
    - Connections: None (informational only)  
    - Edge Cases: None

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Describes the Recommendation API’s functionality, the single endpoint supported, and the workflow’s internal workings.  
    - Content Highlights:  
      - Explanation of the findListingRecommendations method for Promoted Listings.  
      - Workflow components: MCP Trigger, HTTP Request nodes, AI expression usage, native integration.  
      - Lists available operations (one endpoint).  
    - Connections: None (informational only)  
    - Edge Cases: None

  - **Sticky Note ("Listing Recommendation")**  
    - Type: Sticky Note  
    - Role: Simple label indicating the next block relates to Listing Recommendation functionality.  
    - Connections: None  
    - Edge Cases: None

---

#### 2.2 MCP Server Trigger

- **Overview:** This node acts as the server endpoint for AI agents, exposing a webhook that listens for incoming requests under the path `/recommendation-mcp`. It initiates the workflow execution upon receiving requests.
- **Nodes Involved:**  
  - Recommendation MCP Server (MCP Trigger)

- **Node Details:**

  - **Recommendation MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Entry point for MCP-based AI agent requests; listens at a webhook path.  
    - Configuration:  
      - Webhook Path: `recommendation-mcp`  
      - No additional parameters configured  
    - Input Connections: None (trigger node)  
    - Output Connections: Connects to the HTTP Request Tool node ("Get Promoted Listings Recommendations") via the `ai_tool` interface.  
    - Version-Specific Requirements: Requires n8n version supporting MCP Trigger node (Langchain integration).  
    - Edge Cases / Potential Failures:  
      - Webhook not reachable if workflow inactive or n8n instance down.  
      - Authentication failure if credentials are misconfigured downstream.  
      - Request payload missing or malformed from AI agent.

---

#### 2.3 API Request Execution

- **Overview:** This block sends the incoming request parameters to the eBay Recommendation API’s `findListingRecommendations` endpoint via an HTTP POST request. It dynamically fills query parameters and headers with AI-provided values, enabling flexible, AI-driven requests.
- **Nodes Involved:**  
  - Get Promoted Listings Recommendations (HTTP Request Tool)

- **Node Details:**

  - **Get Promoted Listings Recommendations**  
    - Type: `n8n-nodes-base.httpRequestTool`  
    - Role: Executes an HTTP POST request to the eBay Recommendation API to retrieve listing recommendations.  
    - Configuration:  
      - URL: `https://api.ebay.com{basePath}/find` (dynamic basePath concatenation)  
      - Method: POST  
      - Authentication: Generic HTTP Header Auth (configured with OAuth2 credentials externally)  
      - Query Parameters (populated via `$fromAI()` expressions):  
        - `filter`: Specifies criteria for filtering the response; defaults to `recommendationTypes:{AD}`  
        - `limit`: Max number of ads per page (default 10, max 500)  
        - `offset`: Number of ads to skip before starting response (default 0)  
      - Header Parameters (populated via `$fromAI()`):  
        - `X-EBAY-C-MARKETPLACE-ID`: Required header specifying the eBay marketplace  
      - Options: Send headers and query parameters enabled  
      - Tool Description: Detailed explanation of the `findListingRecommendations` method, including how `promoteWithAd` and `bidPercentage` fields inform sellers about Promoted Listings.  
    - Input Connections: Receives input from MCP Trigger node  
    - Output Connections: Returns response directly to MCP Trigger output for AI agent consumption  
    - Version-Specific Requirements: Requires HTTP Request Tool node version supporting dynamic expressions and OAuth2 header authentication  
    - Edge Cases / Potential Failures:  
      - Authentication failures due to invalid or expired OAuth2 tokens  
      - API rate limiting or throttling from eBay endpoint  
      - Missing or invalid query or header parameters leading to API errors  
      - Network timeouts or connectivity issues  
      - Unexpected API response formats or errors  
    - Sub-workflow Reference: None

---

### 3. Summary Table

| Node Name                        | Node Type                         | Functional Role                                      | Input Node(s)             | Output Node(s)                       | Sticky Note                                                                                             |
|---------------------------------|----------------------------------|-----------------------------------------------------|---------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------|
| Setup Instructions              | Sticky Note                      | Provides setup and usage instructions                | None                      | None                               | Contains detailed setup steps, usage notes, customization tips, and support links                     |
| Workflow Overview              | Sticky Note                      | Describes API functionality and workflow overview    | None                      | None                               | Explains the Recommendation API and workflow architecture                                            |
| Sticky Note ("Listing Recommendation") | Sticky Note                      | Labels the listing recommendation section             | None                      | None                               |                                                                                                       |
| Recommendation MCP Server       | MCP Trigger                     | Entry point webhook for AI agent requests             | None                      | Get Promoted Listings Recommendations |                                                                                                       |
| Get Promoted Listings Recommendations | HTTP Request Tool               | Calls eBay Recommendation API with AI-populated params | Recommendation MCP Server | Returns API response to MCP trigger | Details API method usage, parameters, and response data; dynamically fills query and headers via $fromAI |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: "Setup Instructions"**  
   - Type: Sticky Note  
   - Content: Add detailed instructions for importing, authentication setup (OAuth2), activating workflow, MCP URL extraction, AI agent connection, usage notes, customization tips, and support links (Discord, n8n docs).  
   - Position: Left side, for user reference.

2. **Create Sticky Note: "Workflow Overview"**  
   - Type: Sticky Note  
   - Content: Describe the Recommendation API, the single endpoint supported, workflow components (MCP trigger, HTTP request nodes), and AI parameter population via `$fromAI()`.  
   - Position: Near setup instructions.

3. **Create Sticky Note: "Listing Recommendation"**  
   - Type: Sticky Note  
   - Content: Simple label naming the following block.  
   - Position: Above API request node.

4. **Add MCP Trigger Node: "Recommendation MCP Server"**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Webhook Path: `recommendation-mcp`  
   - Position: Center-left  
   - Note: This node acts as the entry endpoint for AI agent requests.

5. **Add HTTP Request Tool Node: "Get Promoted Listings Recommendations"**  
   - Type: `n8n-nodes-base.httpRequestTool`  
   - Parameters:  
     - URL: `https://api.ebay.com{basePath}/find` (Ensure `basePath` is provided in the request context or set as a variable)  
     - HTTP Method: POST  
     - Authentication: Generic HTTP Header Auth configured with OAuth2 credentials for eBay API  
     - Query Parameters:  
       - `filter`: Use expression `{{$fromAI('filter', 'Provide a list of key-value pairs... Default: recommendationTypes:{AD}', 'string')}}`  
       - `limit`: Use expression `{{$fromAI('limit', 'Set max ads per page (Default:10, Max:500)', 'string')}}`  
       - `offset`: Use expression `{{$fromAI('offset', 'Number of ads to skip, Default:0', 'string')}}`  
     - Headers:  
       - `X-EBAY-C-MARKETPLACE-ID`: Use expression `{{$fromAI('X-EBAY-C-MARKETPLACE-ID', 'Specify eBay marketplace', 'string')}}`  
     - Enable sending query parameters and headers  
   - Position: Right of MCP Trigger node

6. **Connect Nodes:**  
   - Connect `Recommendation MCP Server` output `ai_tool` to `Get Promoted Listings Recommendations` input.

7. **Configure Credentials:**  
   - Create or configure OAuth2 credential for eBay API with HTTP header authentication for use in the HTTP Request Tool node.

8. **Activate Workflow:**  
   - Save and activate the workflow in n8n.  
   - Copy the webhook URL from the MCP Trigger node to provide to AI agents for integration.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Parameters for API calls are auto-populated using `$fromAI()` expressions from the AI agent requests. | Usage note in Setup Instructions                                                                            |
| For integration help and custom automation support, contact via Discord: [discord.me/cfomodz](https://discord.me/cfomodz) | Setup Instructions sticky note                                                                              |
| Official n8n documentation for Langchain MCP nodes: [n8n MCP Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) | Setup Instructions sticky note                                                                              |
| eBay Recommendation API details: Helps sellers configure Promoted Listings campaigns based on recommendations. | Workflow Overview sticky note                                                                                |

---

**Disclaimer:** The provided content is generated exclusively from an n8n automated workflow. It complies strictly with all applicable content policies and contains no illegal, offensive, or protected material. All processed data are lawful and public.