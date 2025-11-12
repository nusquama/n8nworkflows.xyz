Get Real-time Stock Analysis and Rankings with Danelfin's AI Analytics API

https://n8nworkflows.xyz/workflows/get-real-time-stock-analysis-and-rankings-with-danelfin-s-ai-analytics-api-6541


# Get Real-time Stock Analysis and Rankings with Danelfin's AI Analytics API

### 1. Workflow Overview

This workflow, titled **Get Real-time Stock Analysis and Rankings with Danelfin's AI Analytics API**, is designed to integrate Danelfin’s AI-powered stock analytics platform into an automation workflow. It provides real-time access to stock rankings, sector performance data, and industry-level analysis via Danelfin’s MCP (Model Context Protocol) API endpoints.

The workflow is structured into these logical blocks:

- **1.1 MCP Trigger Reception:**  
  A webhook-based trigger node awaits incoming requests on the `/danelfin-api` path with header-based authentication. This node serves as the entry point for the workflow, receiving parameters to query different Danelfin API endpoints.

- **1.2 API Query Execution:**  
  The workflow dispatches requests to three distinct Danelfin API endpoints—`/ranking`, `/sectors`, and `/industries`—using HTTP Request nodes. Each node sends queries using parameters dynamically passed from the trigger and authenticated with HTTP header credentials.

- **1.3 Documentation and Context (Sticky Note):**  
  A detailed sticky note node describes the Danelfin platform, its AI analytics features, and the specifics of each MCP server endpoint for user reference and documentation within the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Reception

- **Overview:**  
  This block listens for incoming HTTP requests that initiate the workflow. It authenticates requests via headers and routes parameters for querying Danelfin’s API services.

- **Nodes Involved:**  
  - `Danelfin mcp`

- **Node Details:**

  - **Node Name:** Danelfin mcp  
  - **Type:** MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
  - **Role:** Webhook trigger that receives incoming API requests at the `/danelfin-api` path, secured with HTTP header authentication.  
  - **Configuration:**  
    - Webhook path: `/danelfin-api`  
    - Authentication: Header-based, using stored credentials named `danelfin`  
  - **Expressions/Variables:** Passes query parameters from incoming requests to downstream nodes.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Outputs to all three API request nodes (`ranking`, `sectors`, `industries`) via the `ai_tool` output.  
  - **Version Requirements:** Uses type version 2 of the MCP Trigger node.  
  - **Potential Failure Points:**  
    - Authentication failure if HTTP header credentials are missing or invalid.  
    - Webhook misconfiguration or path conflicts.  
    - Malformed or missing query parameters in incoming requests.

---

#### 2.2 API Query Execution

- **Overview:**  
  This block contains three HTTP Request Tool nodes that call Danelfin’s MCP REST API endpoints to fetch stock rankings, sector data, and industry data. Each node dynamically receives parameters from the trigger and authenticates via HTTP headers.

- **Nodes Involved:**  
  - `ranking`  
  - `sectors`  
  - `industries`

- **Node Details:**

  - **Node Name:** ranking  
    - **Type:** HTTP Request Tool (`n8n-nodes-base.httpRequestTool`)  
    - **Role:** Calls the `/ranking` endpoint to retrieve AI-powered stock rankings and ratings.  
    - **Configuration:**  
      - URL: `https://apirest.danelfin.com/ranking`  
      - Method: GET (default)  
      - Authentication: HTTP Header Auth with `danelfin` credentials  
      - Query Parameters: Two parameters dynamically injected from the trigger node’s input (names and values resolved via expressions)  
    - **Input:** Connected from `Danelfin mcp` node’s output (`ai_tool`).  
    - **Output:** Provides ranking data for downstream use or response.  
    - **Version Requirements:** Node version 4.2.  
    - **Potential Failures:**  
      - Authentication errors if token invalid or expired.  
      - Incorrect or missing query parameters leading to API errors.  
      - Network issues or endpoint downtime.  

  - **Node Name:** sectors  
    - **Type:** HTTP Request Tool  
    - **Role:** Calls the `/sectors` endpoint for sector-based stock analysis, including AI ratings and performance metrics.  
    - **Configuration:**  
      - URL: `https://apirest.danelfin.com/sectors`  
      - Method: GET  
      - Authentication: HTTP Header Auth with `danelfin` credentials  
      - Query Parameters: Up to three parameters dynamically injected from the trigger node’s input  
    - **Input:** Connected from `Danelfin mcp` node’s output (`ai_tool`).  
    - **Output:** Sector data response.  
    - **Version Requirements:** Node version 4.2.  
    - **Potential Failures:** Same as `ranking` node, plus potential parameter mismatches due to variable query count.  

  - **Node Name:** industries  
    - **Type:** HTTP Request Tool  
    - **Role:** Calls the `/industries` endpoint for granular industry-level stock analytics.  
    - **Configuration:**  
      - URL: `https://apirest.danelfin.com/industries`  
      - Method: GET  
      - Authentication: HTTP Header Auth with `danelfin` credentials  
      - Query Parameters: Two parameters dynamically injected from trigger input  
    - **Input:** Connected from `Danelfin mcp` node’s output (`ai_tool`).  
    - **Output:** Industry-specific analytics data.  
    - **Version Requirements:** Node version 4.2.  
    - **Potential Failures:** Similar to other HTTP Request nodes; API throttling or invalid parameters possible.  

---

#### 2.3 Documentation and Context (Sticky Note)

- **Overview:**  
  This block consists of a large sticky note node embedded in the workflow to provide comprehensive documentation about Danelfin’s AI analytics platform, its key features, and the three MCP endpoints used in the workflow. It serves as an internal knowledge base for users maintaining or enhancing the workflow.

- **Nodes Involved:**  
  - `Sticky Note`

- **Node Details:**

  - **Node Name:** Sticky Note  
  - **Type:** Sticky Note (`n8n-nodes-base.stickyNote`)  
  - **Role:** Documentation and explanation within the workflow interface.  
  - **Configuration:**  
    - Size: 848x1808 pixels  
    - Content: Detailed markdown text describing:  
      - Danelfin platform overview  
      - Key AI features (Stock Picker, Portfolio Optimization, Explainable AI, Predictive Analytics)  
      - Descriptions of `/ranking`, `/sectors`, and `/industries` endpoints  
      - Integration benefits and target use cases (investment research, algorithmic trading, advisory, risk management, market analysis)  
  - **Input/Output:** None (informational only).  
  - **Potential Failures:** None applicable; purely documentation.  

---

### 3. Summary Table

| Node Name      | Node Type                              | Functional Role                          | Input Node(s)     | Output Node(s)               | Sticky Note                                                                                                                       |
|----------------|--------------------------------------|----------------------------------------|-------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Danelfin mcp   | MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`) | Entry point webhook with header authentication | None              | ranking, sectors, industries |                                                                                                                                 |
| ranking        | HTTP Request Tool (`n8n-nodes-base.httpRequestTool`) | Calls `/ranking` endpoint for stock rankings | Danelfin mcp      | None                        |                                                                                                                                 |
| sectors        | HTTP Request Tool                    | Calls `/sectors` endpoint for sector analysis | Danelfin mcp      | None                        |                                                                                                                                 |
| industries     | HTTP Request Tool                    | Calls `/industries` endpoint for industry analysis | Danelfin mcp      | None                        |                                                                                                                                 |
| Sticky Note    | Sticky Note                        | Embedded documentation about Danelfin and API endpoints | None              | None                        | ## Danelfin MCP Server\n\n## About Danelfin\n\nDanelfin is an AI-powered stock analytics platform that helps investors find the best stocks and optimize their portfolios with explainable AI insights to make smarter, data-driven investment decisions. The platform uses AI-driven stock analytics to optimize investment strategies.\n\nDanelfin uses Explainable Artificial Intelligence to help everyone make smart and data-driven investment decisions. The platform provides API solutions for developers, analysts, and fintech firms wanting to integrate predictive stock data into their own applications.\n\n## Key Features\n\n- **AI Stock Picker**: Advanced algorithms to identify high-potential stocks\n- **Portfolio Optimization**: Data-driven portfolio management and optimization tools\n- **Explainable AI**: Transparent AI insights that show reasoning behind recommendations\n- **Predictive Analytics**: AI Scores and forecasting capabilities\n- **Multi-Asset Coverage**: Analysis of stocks and ETFs across various markets\n\n## MCP Server Endpoints\n\nThis Danelfin MCP (Model Context Protocol) server exposes three main endpoints that provide comprehensive market analysis capabilities:\n\n### 🏆 `/ranking` Endpoint\n... (full content in section 2.3) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Add a node of type `MCP Trigger` (`@n8n/n8n-nodes-langchain.mcpTrigger`).  
   - Set the webhook path to `/danelfin-api`.  
   - Configure authentication as `Header Auth`.  
   - Assign credentials using your Danelfin HTTP header authentication credentials (named `danelfin`).  
   - Position this node as the workflow entry point.

2. **Create HTTP Request Tool Node for Ranking**  
   - Add a node of type `HTTP Request Tool` (`n8n-nodes-base.httpRequestTool`).  
   - Set the URL to `https://apirest.danelfin.com/ranking`.  
   - Use HTTP GET method (default).  
   - Configure authentication as `HTTP Header Auth` using the same `danelfin` credentials.  
   - Add two query parameters, each with dynamic names and values mapped from the trigger node’s input data via expressions:  
     - Parameter 1: name and value expressions from input  
     - Parameter 2: name and value expressions from input  
   - Connect the output of the MCP Trigger node (`ai_tool` output) to this node’s input.

3. **Create HTTP Request Tool Node for Sectors**  
   - Add another `HTTP Request Tool` node.  
   - Set URL to `https://apirest.danelfin.com/sectors`.  
   - Use HTTP GET.  
   - Authenticate with `danelfin` HTTP Header credentials.  
   - Add up to three dynamic query parameters from the MCP trigger input, each with name and value expressions.  
   - Connect MCP Trigger node output (`ai_tool`) to this node’s input.

4. **Create HTTP Request Tool Node for Industries**  
   - Add a third `HTTP Request Tool` node.  
   - Set URL to `https://apirest.danelfin.com/industries`.  
   - Use HTTP GET.  
   - Authenticate again with `danelfin` HTTP Header credentials.  
   - Add two query parameters dynamically assigned from the trigger input.  
   - Connect MCP Trigger node output (`ai_tool`) to this node.

5. **Add Sticky Note for Documentation**  
   - Insert a `Sticky Note` node.  
   - Paste the detailed markdown documentation describing Danelfin’s platform, AI features, and the API endpoints `/ranking`, `/sectors`, and `/industries`.  
   - Size it to fit the content for easy readability (approx. width 848, height 1808).

6. **Configure Credentials**  
   - Create an HTTP Header Authentication credential named `danelfin`.  
   - This credential must include the API key or token required by Danelfin’s MCP API.  
   - Assign this credential to all HTTP Request Tool nodes and the MCP Trigger node.

7. **Set Workflow Settings**  
   - Set execution order to `v1` (default linear execution).  
   - Activate the workflow.

8. **Test the Workflow**  
   - Send authenticated HTTP requests to `/danelfin-api` with appropriate query parameters for any of the endpoints.  
   - Validate responses from each HTTP Request node correspond to expected data from Danelfin’s API.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| Danelfin is an AI-powered stock analytics platform focusing on explainable AI insights for smarter investment decisions. It offers API endpoints for stock rankings, sector analysis, and industry insights. The platform supports developers and financial professionals aiming to integrate advanced AI-driven market analysis into applications.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Provided in embedded Sticky Note node        |
| The workflow uses Danelfin’s MCP (Model Context Protocol) server endpoints: `/ranking`, `/sectors`, and `/industries` to fetch real-time stock data with AI-based analytics and ratings.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | API endpoint documentation                    |
| Authentication is handled via HTTP Header Auth with a custom API token credential named `danelfin`. Credentials must be securely stored and assigned to all relevant nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Credential configuration best practice        |
| The workflow is designed for extensibility and can be integrated into larger automation scenarios involving stock market data, portfolio management, algorithmic trading, and financial advisory services.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Use case suggestions                           |
| For additional information or updates about Danelfin’s API and MCP server, consult Danelfin’s official documentation or contact their support team.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | External reference (not linked in workflow)  |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow configuration. It respects all applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.