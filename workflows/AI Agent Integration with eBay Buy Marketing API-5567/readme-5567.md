AI Agent Integration with eBay Buy Marketing API

https://n8nworkflows.xyz/workflows/ai-agent-integration-with-ebay-buy-marketing-api-5567


# AI Agent Integration with eBay Buy Marketing API

### 1. Workflow Overview

This workflow is designed to provide an AI-compatible interface to eBay’s Buy Marketing API, specifically enabling the retrieval of merchandised products based on marketing metrics such as "Best Selling." It acts as an MCP (Model Control Plane) server that accepts AI agent requests and returns eBay product data in a standardized format.

The workflow is logically divided into these blocks:

- **1.1 Setup & Documentation Block**: Provides configuration instructions and workflow overview notes for users.
- **1.2 MCP Server Trigger Block**: Listens for incoming AI agent requests via an MCP-compatible webhook.
- **1.3 API Request Block**: Executes HTTP requests to the eBay Buy Marketing API endpoint to fetch merchandised products based on parameters dynamically provided by the AI agent.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Documentation Block

**Overview:**  
This block contains sticky notes that document setup instructions, usage notes, and an overview of the workflow’s functionality, helping users understand and correctly configure the workflow.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)  
- Sticky Note (Merchandised Product header)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Provides detailed step-by-step instructions for importing, configuring OAuth2 credentials, activating the workflow, obtaining the MCP server URL, and connecting the AI agent. Also includes tips on customization and links for help.  
  - Inputs: None  
  - Outputs: None  
  - Edge Cases: None (informational only)  

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Summarizes the purpose of the workflow, describing the Buy Marketing API’s capabilities and how this workflow exposes it as an MCP server endpoint with AI-driven parameters.  
  - Inputs: None  
  - Outputs: None  
  - Edge Cases: None  

- **Sticky Note (Merchandised Product)**  
  - Type: Sticky Note  
  - Role: Acts as a section header for the merchandised product API operation.  
  - Inputs: None  
  - Outputs: None  
  - Edge Cases: None  

---

#### 1.2 MCP Server Trigger Block

**Overview:**  
This block contains a single MCP trigger node that acts as the public-facing webhook endpoint. It receives AI agent requests and triggers the workflow.

**Nodes Involved:**  
- Buy Marketing MCP Server (MCP Trigger)

**Node Details:**

- **Buy Marketing MCP Server**  
  - Type: MCP Trigger (from n8n-nodes-langchain package)  
  - Role: Exposes a webhook path (`buy-marketing-mcp`) which AI agents call to invoke the workflow. It acts as the server endpoint for MCP operations.  
  - Configuration:  
    - Webhook path: `/buy-marketing-mcp`  
  - Inputs: None (trigger node)  
  - Outputs: Connects to the "Fetch Merchandised Products" HTTP request node as an AI tool.  
  - Version-specific: Requires n8n version supporting MCP trigger nodes from the Langchain integration.  
  - Edge Cases: Webhook URL misconfiguration, request authorization failures if credentials are invalid or missing, network timeouts.  

---

#### 1.3 API Request Block

**Overview:**  
This block performs the actual API call to eBay’s Buy Marketing API to fetch merchandised products based on dynamic parameters received from the AI agent.

**Nodes Involved:**  
- Fetch Merchandised Products (HTTP Request Tool)

**Node Details:**

- **Fetch Merchandised Products**  
  - Type: HTTP Request Tool (built-in HTTP Request node with tool integration)  
  - Role: Calls the eBay Buy Marketing API endpoint `/buy/marketing/v1_beta/merchandised_product` with query parameters dynamically populated using `$fromAI()` expressions.  
  - Configuration:  
    - URL: `https://api.ebay.com/buy/marketing/v1_beta/merchandised_product`  
    - Authentication: Generic OAuth2 Bearer token via HTTP header  
    - Query Parameters (all dynamically set via AI expressions):  
      - `aspect_filter`: Optional filter for product aspect name/value pairs (e.g., "Brand:Canon")  
      - `category_id`: Required eBay category ID to limit product results  
      - `limit`: Optional max number of products to return (default 8, max 100)  
      - `metric_name`: Required metric filter, currently only supports `BEST_SELLING`  
  - Key Expressions:  
    - `$fromAI('parameter_name', 'description', 'string')` used to auto-populate parameters from AI input.  
  - Inputs: Triggered by the MCP server node.  
  - Outputs: Returns eBay API response maintaining original structure back to the AI agent.  
  - Edge Cases:  
    - Authentication failures (invalid OAuth token)  
    - API rate limits or throttling by eBay  
    - Invalid or missing query parameters causing API errors  
    - Network errors/timeouts  
  - Additional Notes:  
    - Sandbox mode requires category_id `9355` for testing with mock data.  
    - API response contains product details like EPID, title, and user reviews.  
    - Returned EPID can be used for further Browse API queries.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                        | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                                                        |
|----------------------------|--------------------------------|--------------------------------------|------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions          | Sticky Note                    | Setup and configuration instructions | None                   | None                           | ### ⚙️ Setup Instructions<br>1. Import Workflow<br>2. Configure Authentication<br>3. Activate Workflow<br>4. Get MCP URL<br>5. Connect AI Agent<br>Usage notes and customization tips with helpful links. |
| Workflow Overview           | Sticky Note                    | Workflow purpose and API overview    | None                   | None                           | ## 🛠️ Buy Marketing MCP Server<br>1 operation<br>Operation details and how the workflow functions as an MCP server with AI integration. |
| Sticky Note (Merchandised Product) | Sticky Note                    | Section header for merchandised product operation | None                   | None                           | ## Merchandised Product                                                                                                           |
| Buy Marketing MCP Server    | MCP Trigger                   | AI agent webhook endpoint (MCP server) | None                   | Fetch Merchandised Products    |                                                                                                                                   |
| Fetch Merchandised Products | HTTP Request Tool             | Calls eBay Buy Marketing API to fetch merchandised products | Buy Marketing MCP Server | None                           | API call with dynamic parameters from AI; maintains original API response structure; supports OAuth2 authentication.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note - Setup Instructions**  
   - Add a Sticky Note node.  
   - Paste setup instructions detailing import, OAuth2 setup, workflow activation, webhook URL retrieval, AI agent connection, customization tips, and support links.  
   - Set color to blue (color=4), height ~1020, position at approximately [-1380, -240].

2. **Create Sticky Note - Workflow Overview**  
   - Add another Sticky Note node.  
   - Paste the overview text explaining the workflow purpose, API operations, MCP server role, and AI interaction details.  
   - Set color default, width 420, height ~920, position [-1120, -240].

3. **Create Sticky Note - Merchandised Product Section Header**  
   - Add a Sticky Note node.  
   - Content: "## Merchandised Product"  
   - Set color to green (color=2), width 320, height 220, position [-660, -100].

4. **Add MCP Trigger Node - Buy Marketing MCP Server**  
   - Add an MCP Trigger node from the Langchain integration package.  
   - Configure webhook path: `buy-marketing-mcp`.  
   - Position node around [-560, -240].  
   - This node will serve as the entry point for AI requests.

5. **Add HTTP Request Tool Node - Fetch Merchandised Products**  
   - Add an HTTP Request Tool node (HTTP Request with tool integration).  
   - Set URL to `https://api.ebay.com/buy/marketing/v1_beta/merchandised_product`.  
   - Authentication: Configure generic OAuth2 with HTTP Header authentication (Bearer token).  
   - Query Parameters: Add parameters with expressions:  
     - `aspect_filter`: `{{$fromAI('aspect_filter', 'Description', 'string')}}`  
     - `category_id`: `{{$fromAI('category_id', 'Description', 'string')}}` (required)  
     - `limit`: `{{$fromAI('limit', 'Description', 'string')}}` (optional, default 8)  
     - `metric_name`: `{{$fromAI('metric_name', 'Description', 'string')}}` (required, default `BEST_SELLING`)  
   - Position node near [-520, -40].

6. **Connect Nodes**  
   - Connect output of the MCP Trigger node "Buy Marketing MCP Server" to the input of "Fetch Merchandised Products" node as an AI tool. This sets up the flow from AI webhook to API call.

7. **Credential Setup**  
   - Create or configure OAuth2 credentials for eBay API with required scopes and token refresh settings. Assign these credentials to the HTTP Request Tool node.  

8. **Activate Workflow**  
   - Save and activate the workflow.  
   - Copy the webhook URL of the MCP Trigger node to configure the AI agent endpoint.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Ping on Discord for integration guidance and custom automations.                                                                                                                                                                                                                                                                                             | [Discord](https://discord.me/cfomodz)                                                                                |
| Refer to n8n documentation for MCP nodes and Langchain integration details.                                                                                                                                                                                                                                                                                  | [n8n Documentation: MCP Nodes](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) |
| eBay API documentation for Marketing API parameters, aspects, and categories.                                                                                                                                                                                                                                                                               | [eBay Marketing API Docs](https://developer.ebay.com/api-docs/buy/marketing/overview.html)                            |
| Use eBay Category Changes page or Taxonomy API to find valid category IDs.                                                                                                                                                                                                                                                                                   | [Category Changes](https://pages.ebay.com/sellerinformation/news/categorychanges.html)                                |

---

**Disclaimer:** The provided text is extracted solely from an n8n automated workflow created with respect to current content policies and contains no illegal or protected elements. All data processed are legal and publicly available.