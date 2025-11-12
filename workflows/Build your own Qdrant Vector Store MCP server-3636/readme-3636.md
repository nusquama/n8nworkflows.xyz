Build your own Qdrant Vector Store MCP server

https://n8nworkflows.xyz/workflows/build-your-own-qdrant-vector-store-mcp-server-3636


# Build your own Qdrant Vector Store MCP server

### 1. Workflow Overview

This workflow implements a custom MCP (Machine Comprehension Protocol) server for Qdrant vector stores, extending the official Qdrant MCP server capabilities. It enables advanced business intelligence use cases by exposing additional Qdrant API features such as facet search, grouped search, and recommendations. The workflow is designed to support MCP clients (e.g., Claude Desktop) for managing and querying a Qdrant collection of customer reviews.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger & Operation Routing**: Entry point that receives MCP client requests and routes them based on the requested operation.
- **1.2 Core Tools as Sub-Workflows**: Five custom tool workflows handling specific operations: insert reviews, search reviews, compare reviews, recommend reviews, and list available companies.
- **1.3 Qdrant Collection Setup (Manual Trigger)**: Manual flow to create the Qdrant collection and facet index if not already present.
- **1.4 Advanced Qdrant API Integrations**: HTTP request nodes to leverage Qdrant’s facet search, grouped search, and recommendation APIs beyond basic vector store operations.
- **1.5 Data Processing & Response Formatting**: Nodes to process, aggregate, and simplify Qdrant API responses into formats suitable for MCP clients.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger & Operation Routing

**Overview:**  
This block receives incoming MCP client requests and routes them to the appropriate operation workflow based on the `operation` field in the request payload.

**Nodes Involved:**  
- Qdrant MCP Server (MCP Trigger)  
- Operation (Switch)  
- When Executed by Another Workflow (Execute Workflow Trigger)

**Node Details:**

- **Qdrant MCP Server**  
  - Type: MCP Trigger  
  - Role: Entry point for MCP client requests via webhook.  
  - Configuration: Webhook path set uniquely; requires authentication before production.  
  - Inputs: MCP client requests with JSON containing operation and parameters.  
  - Outputs: Routes to tool workflows via ai_tool connections.  
  - Edge Cases: Missing or invalid operation field; authentication failures; webhook connectivity issues.

- **Operation**  
  - Type: Switch  
  - Role: Routes workflow execution based on `operation` value (`listCompanies`, `insert`, `search`, `compare`, `recommend`).  
  - Configuration: Conditions check exact match of `operation` string from input JSON.  
  - Inputs: JSON from MCP trigger.  
  - Outputs: Routes to different nodes for each operation.  
  - Edge Cases: Unknown operation values; case sensitivity issues.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be invoked as a sub-workflow with inputs (`operation`, `text`, `text2`, `companyIds`).  
  - Inputs: Parameters passed from parent workflow.  
  - Outputs: Feeds into the Operation switch node.  
  - Edge Cases: Missing required inputs; invocation errors.

---

#### 1.2 Core Tools as Sub-Workflows

**Overview:**  
This block contains five custom tool workflows encapsulated as nodes. Each tool handles a specific operation on the Qdrant collection: inserting reviews, searching reviews, comparing reviews, recommending reviews, and listing companies.

**Nodes Involved:**  
- Insert (Tool Workflow)  
- Search (Tool Workflow)  
- Compare (Tool Workflow)  
- Recommend (Tool Workflow)  
- ListCompanies (Tool Workflow)

**Node Details:**

- **Insert**  
  - Type: Tool Workflow  
  - Role: Inserts a customer review document into the Qdrant collection.  
  - Configuration: Accepts inputs `text` (review content), `companyIds` (company identifiers), and `operation` set to `insert`.  
  - Inputs: Text review and metadata from MCP client.  
  - Outputs: Confirmation response "ok".  
  - Edge Cases: Invalid or missing review text; Qdrant API insertion errors.

- **Search**  
  - Type: Tool Workflow  
  - Role: Searches the Qdrant collection for reviews matching query terms.  
  - Configuration: Inputs include `text` (search query), optional `companyIds` to filter companies, and `operation` set to `search`.  
  - Inputs: Search terms and optional company filters.  
  - Outputs: Aggregated search results.  
  - Edge Cases: Empty or ambiguous search queries; no matching results.

- **Compare**  
  - Type: Tool Workflow  
  - Role: Compares search results across two or more specified companies.  
  - Configuration: Inputs `text` (search terms), `companyIds` (comma-separated list of companies), `operation` set to `compare`.  
  - Inputs: Query and multiple company IDs.  
  - Outputs: Aggregated comparison results or "no results." if empty.  
  - Edge Cases: Less than two companies specified; no results found.

- **Recommend**  
  - Type: Tool Workflow  
  - Role: Generates recommendations based on positive and/or negative preferences.  
  - Configuration: Inputs `text` (positive preferences), `text2` (negative preferences), optional `companyIds`, `operation` set to `recommend`.  
  - Inputs: Preference texts and optional company filters.  
  - Outputs: Aggregated recommendation results or "no results." if empty.  
  - Edge Cases: Empty preferences; API errors; no matching recommendations.

- **ListCompanies**  
  - Type: Tool Workflow  
  - Role: Lists all available companies in the reviews database using facet search.  
  - Configuration: Operation set to `listCompanies`.  
  - Inputs: None required.  
  - Outputs: List of company IDs.  
  - Edge Cases: Empty collection; API connectivity issues.

---

#### 1.3 Qdrant Collection Setup (Manual Trigger)

**Overview:**  
This manual flow allows users to create the Qdrant collection and facet index if they do not already exist, facilitating initial setup.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Create Collection (HTTP Request)  
- Create Facet Index (HTTP Request)  
- Sticky Note3 (Instructional)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: User-initiated trigger to run setup steps.  
  - Inputs: None.  
  - Outputs: Triggers collection creation.

- **Create Collection**  
  - Type: HTTP Request  
  - Role: Creates a Qdrant collection named `trustpilot_reviews` with cosine distance and vector size 1536.  
  - Configuration: PUT request to Qdrant API endpoint `/collections/trustpilot_reviews`.  
  - Inputs: None.  
  - Outputs: Response from Qdrant API.  
  - Edge Cases: Collection already exists; network errors.

- **Create Facet Index**  
  - Type: HTTP Request  
  - Role: Creates an index on the `metadata.company_id` field to enable facet search.  
  - Configuration: PUT request to `/collections/trustpilot_reviews/index` with JSON body specifying field name and schema.  
  - Inputs: None.  
  - Outputs: Response from Qdrant API.  
  - Edge Cases: Index already exists; API errors.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Provides instructions for prerequisite setup.  
  - Content: Explains the manual steps to create collection and index.

---

#### 1.4 Advanced Qdrant API Integrations

**Overview:**  
This block uses HTTP Request nodes to call advanced Qdrant API endpoints for grouped search, facet search, and recommendations, extending beyond basic vector store operations.

**Nodes Involved:**  
- List by Facet API (HTTP Request)  
- Group Search API (HTTP Request)  
- Recommend API (HTTP Request)  
- Get Embeddings (HTTP Request)  
- Get Embeddings1 (HTTP Request)  
- Preferences to Items (Code)  
- Aggregate Embeddings (Aggregate)  
- Aggregate Embeddings1 (Aggregate)  
- Simplify Group Results (Set)  
- Split Out Companies (Split Out)  
- Filter By CompanyId (Filter)  
- Has Results? (If)  
- Has Results?1 (If)  
- Simplify Recommend Response (Set)  
- Aggregate Recommend Response (Aggregate)

**Node Details:**

- **List by Facet API**  
  - Type: HTTP Request  
  - Role: Queries Qdrant facet API to list unique `company_id` values.  
  - Configuration: POST to `/collections/trustpilot_reviews/facet` with JSON body specifying key.  
  - Inputs: None.  
  - Outputs: List of companies.  
  - Edge Cases: Empty collection; API errors.

- **Group Search API**  
  - Type: HTTP Request  
  - Role: Performs grouped search by `company_id` with vector query.  
  - Configuration: POST to `/collections/trustpilot_reviews/points/search/groups` with vector, group size, limit, and payload options.  
  - Inputs: Embeddings from OpenAI.  
  - Outputs: Grouped search results.  
  - Edge Cases: Empty embeddings; no results.

- **Recommend API**  
  - Type: HTTP Request  
  - Role: Calls Qdrant recommendation API with positive and negative embeddings and optional filters.  
  - Configuration: POST to `/collections/trustpilot_reviews/points/recommend` with strategy, limit, positive/negative vectors, filters, and payload inclusion.  
  - Inputs: Aggregated embeddings.  
  - Outputs: Recommendation results.  
  - Edge Cases: Empty embeddings; API errors.

- **Get Embeddings / Get Embeddings1**  
  - Type: HTTP Request  
  - Role: Calls OpenAI embeddings API to convert text input into vector embeddings.  
  - Configuration: POST to OpenAI `/v1/embeddings` with model `text-embedding-3-small`.  
  - Inputs: Text from MCP client or preferences.  
  - Outputs: Embeddings array.  
  - Edge Cases: API rate limits; invalid input text.

- **Preferences to Items**  
  - Type: Code  
  - Role: Converts preference texts into array items for embedding requests.  
  - Inputs: `text` and `text2` fields from MCP client.  
  - Outputs: Array of objects with text properties.  
  - Edge Cases: Empty or null preferences.

- **Aggregate Embeddings / Aggregate Embeddings1**  
  - Type: Aggregate  
  - Role: Extracts and renames embedding vectors from OpenAI responses for downstream use.  
  - Inputs: OpenAI embedding responses.  
  - Outputs: Aggregated embeddings array.  
  - Edge Cases: Missing embedding data.

- **Simplify Group Results**  
  - Type: Set  
  - Role: Transforms grouped search results into simplified objects with category and results arrays.  
  - Inputs: Grouped search API response.  
  - Outputs: Simplified JSON for MCP client.  
  - Edge Cases: Empty groups.

- **Split Out Companies**  
  - Type: Split Out  
  - Role: Splits grouped search results into individual company groups for filtering.  
  - Inputs: Grouped search API response.  
  - Outputs: Individual company group items.  
  - Edge Cases: Empty groups.

- **Filter By CompanyId**  
  - Type: Filter  
  - Role: Filters company groups based on requested company IDs from MCP client.  
  - Inputs: Company groups and requested company IDs.  
  - Outputs: Filtered groups or all if no filter applied.  
  - Edge Cases: Invalid or missing company IDs.

- **Has Results? / Has Results?1**  
  - Type: If  
  - Role: Checks if results exist; routes to aggregation or empty response accordingly.  
  - Inputs: Filtered or recommendation results.  
  - Outputs: Aggregated response or "no results." message.  
  - Edge Cases: Empty results.

- **Simplify Recommend Response**  
  - Type: Set  
  - Role: Simplifies recommendation API response to content and metadata fields.  
  - Inputs: Recommendation API response.  
  - Outputs: Simplified JSON for MCP client.  
  - Edge Cases: Empty or malformed response.

- **Aggregate Recommend Response**  
  - Type: Aggregate  
  - Role: Aggregates all recommendation items into a single response field.  
  - Inputs: Simplified recommendation items.  
  - Outputs: Aggregated response.  
  - Edge Cases: Empty input.

---

#### 1.5 Data Processing & Response Formatting

**Overview:**  
This block contains nodes that process raw Qdrant or OpenAI API data and format it into responses suitable for MCP clients.

**Nodes Involved:**  
- Get Insert Response (Set)  
- Get Search Response (Aggregate)  
- Aggregate Compare Response (Aggregate)  
- Empty Compare Response (Set)  
- Empty Compare Response1 (Set)

**Node Details:**

- **Get Insert Response**  
  - Type: Set  
  - Role: Returns a simple "ok" response after successful insert operation.  
  - Inputs: Confirmation from insert operation.  
  - Outputs: Response string.  
  - Edge Cases: Insert failure not handled here.

- **Get Search Response**  
  - Type: Aggregate  
  - Role: Aggregates all search result items into a single response field.  
  - Inputs: Search results from Qdrant.  
  - Outputs: Aggregated search response.  
  - Edge Cases: Empty search results.

- **Aggregate Compare Response**  
  - Type: Aggregate  
  - Role: Aggregates comparison results across companies into a single response.  
  - Inputs: Filtered and simplified comparison data.  
  - Outputs: Aggregated response.  
  - Edge Cases: Empty comparison results.

- **Empty Compare Response / Empty Compare Response1**  
  - Type: Set  
  - Role: Returns "no results." string when no comparison or recommendation results are found.  
  - Inputs: Triggered when no results exist.  
  - Outputs: Response string.  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                    | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                          |
|-----------------------------|----------------------------------|---------------------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Qdrant MCP Server           | MCP Trigger                      | Entry point for MCP client requests                | —                            | Insert, Search, Compare, Recommend, ListCompanies | 1. Set up an MCP Server Trigger [Read more](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger) |
| Operation                   | Switch                          | Routes requests based on operation field           | When Executed by Another Workflow | List by Facet API, Insert Reviews, Search Reviews, Get Embeddings1, Preferences to Items |                                                                                                    |
| When Executed by Another Workflow | Execute Workflow Trigger        | Allows invocation as sub-workflow                   | —                            | Operation                     |                                                                                                    |
| Insert                      | Tool Workflow                   | Inserts review documents into Qdrant               | Qdrant MCP Server            | —                             |                                                                                                    |
| Search                      | Tool Workflow                   | Searches reviews in Qdrant                          | Qdrant MCP Server            | —                             |                                                                                                    |
| Compare                     | Tool Workflow                   | Compares reviews across companies                   | Qdrant MCP Server            | —                             |                                                                                                    |
| Recommend                   | Tool Workflow                   | Generates recommendations based on preferences     | Qdrant MCP Server            | —                             |                                                                                                    |
| ListCompanies               | Tool Workflow                   | Lists available companies in the collection        | Qdrant MCP Server            | —                             |                                                                                                    |
| When clicking ‘Test workflow’ | Manual Trigger                 | Manual trigger for collection setup                 | —                            | Create Collection             |                                                                                                    |
| Create Collection           | HTTP Request                   | Creates Qdrant collection                            | When clicking ‘Test workflow’ | Create Facet Index            |                                                                                                    |
| Create Facet Index          | HTTP Request                   | Creates facet index on company_id                    | Create Collection            | —                             |                                                                                                    |
| List by Facet API           | HTTP Request                   | Retrieves list of companies via facet search        | Operation                   | Insert Reviews                |                                                                                                    |
| Insert Reviews              | VectorStore Qdrant             | Inserts documents into Qdrant collection            | List by Facet API, Default Data Loader, Embeddings OpenAI | Get Insert Response           |                                                                                                    |
| Get Insert Response         | Set                            | Returns confirmation response "ok"                   | Insert Reviews               | —                             |                                                                                                    |
| Search Reviews              | VectorStore Qdrant             | Searches documents in Qdrant collection              | Operation, Embeddings OpenAI1 | Get Search Response           |                                                                                                    |
| Get Search Response         | Aggregate                      | Aggregates search results                             | Search Reviews              | —                             |                                                                                                    |
| Get Embeddings1             | HTTP Request                   | Gets embeddings from OpenAI for search queries       | Operation                   | Aggregate Embeddings1         |                                                                                                    |
| Aggregate Embeddings1       | Aggregate                      | Extracts embeddings from OpenAI response              | Get Embeddings1             | Group Search API              |                                                                                                    |
| Group Search API            | HTTP Request                   | Performs grouped search by company_id                 | Aggregate Embeddings1       | Split Out Companies           | 2. Expand Functionality Beyond Vendor Implementation [Read more](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreqdrant) |
| Split Out Companies         | Split Out                      | Splits grouped search results into company groups    | Group Search API            | Simplify Group Results        |                                                                                                    |
| Simplify Group Results      | Set                            | Simplifies grouped search results                     | Split Out Companies         | Filter By CompanyId           |                                                                                                    |
| Filter By CompanyId         | Filter                         | Filters groups by requested company IDs               | Simplify Group Results      | Has Results?                 |                                                                                                    |
| Has Results?                | If                             | Checks if filtered results exist                       | Filter By CompanyId         | Aggregate Compare Response, Empty Compare Response |                                                                                                    |
| Aggregate Compare Response  | Aggregate                      | Aggregates comparison results                          | Has Results?                | —                             |                                                                                                    |
| Empty Compare Response      | Set                            | Returns "no results." when no comparison results      | Has Results?                | —                             |                                                                                                    |
| Preferences to Items        | Code                           | Converts preference texts to array for embeddings     | Operation                   | Get Embeddings               |                                                                                                    |
| Get Embeddings              | HTTP Request                   | Gets embeddings from OpenAI for preferences           | Preferences to Items        | Aggregate Embeddings          |                                                                                                    |
| Aggregate Embeddings        | Aggregate                      | Extracts embeddings from OpenAI response              | Get Embeddings              | Recommend API                |                                                                                                    |
| Recommend API              | HTTP Request                   | Calls Qdrant recommendation API                        | Aggregate Embeddings        | Simplify Recommend Response   |                                                                                                    |
| Simplify Recommend Response | Set                            | Simplifies recommendation results                      | Recommend API               | Has Results?1                |                                                                                                    |
| Has Results?1               | If                             | Checks if recommendation results exist                 | Simplify Recommend Response | Aggregate Recommend Response, Empty Compare Response1 |                                                                                                    |
| Aggregate Recommend Response| Aggregate                      | Aggregates recommendation results                      | Has Results?1               | —                             |                                                                                                    |
| Empty Compare Response1     | Set                            | Returns "no results." when no recommendation results   | Has Results?1               | —                             |                                                                                                    |
| Default Data Loader         | Document Default Data Loader   | Loads review text and metadata for insertion           | List by Facet API           | Insert Reviews               |                                                                                                    |
| Embeddings OpenAI           | OpenAI Embeddings              | Generates embeddings for inserted review text          | Default Data Loader         | Insert Reviews               |                                                                                                    |
| Embeddings OpenAI1          | OpenAI Embeddings              | Generates embeddings for search queries                 | Operation                   | Search Reviews              |                                                                                                    |
| Recursive Character Text Splitter | Text Splitter              | Splits text into chunks for processing                  | —                          | Default Data Loader          |                                                                                                    |
| Sticky Note                 | Sticky Note                   | Instructional note about MCP Server Trigger            | —                          | —                             | 1. Set up an MCP Server Trigger [Read more](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger) |
| Sticky Note1                | Sticky Note                   | Explains extending Qdrant MCP server functionality      | —                          | —                             | 2. Expand Functionality Beyond Vendor Implementation [Read more](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreqdrant) |
| Sticky Note2                | Sticky Note                   | Overview and usage instructions for the workflow        | —                          | —                             |                                                                                                    |
| Sticky Note3                | Sticky Note                   | Instructions for manual Qdrant collection setup         | —                          | —                             |                                                                                                    |
| Sticky Note4                | Sticky Note                   | Reminder to authenticate MCP server before production   | —                          | —                             |                                                                                                    |
| Sticky Note8                | Sticky Note                   | Reminder to configure Qdrant connection endpoint        | —                          | —                             |                                                                                                    |
| Sticky Note9                | Sticky Note                   | Reminder to configure Qdrant connection endpoint        | —                          | —                             |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Type: MCP Trigger  
   - Configure webhook path uniquely (e.g., UUID)  
   - Enable authentication before production  
   - This node will receive MCP client requests.

2. **Add a Switch Node (Operation)**  
   - Type: Switch  
   - Add rules for `operation` field with values: `listCompanies`, `insert`, `search`, `compare`, `recommend`  
   - Route each output to corresponding tool workflows or nodes.

3. **Add Execute Workflow Trigger Node**  
   - Type: Execute Workflow Trigger  
   - Configure inputs: `operation`, `text`, `text2`, `companyIds`  
   - Connect output to the Operation switch node.

4. **Create Tool Workflows for Each Operation**  
   For each tool workflow (`Insert`, `Search`, `Compare`, `Recommend`, `ListCompanies`):

   - Create a new workflow with inputs matching the schema: `operation`, `text`, `text2`, `companyIds`.  
   - Implement the logic for each operation as per the original workflow (see below).  
   - Save and note the workflow ID.

5. **Insert Tool Workflow**  
   - Accepts review text and company IDs.  
   - Use a Recursive Character Text Splitter node to chunk text if needed.  
   - Use Default Data Loader node to prepare documents with metadata (`company_id`).  
   - Use OpenAI Embeddings node to generate embeddings for the review text.  
   - Use VectorStore Qdrant node in insert mode to insert documents into the `trustpilot_reviews` collection.  
   - Use a Set node to return a simple "ok" response.

6. **Search Tool Workflow**  
   - Accepts search query text and optional company IDs.  
   - Use OpenAI Embeddings node to generate embeddings for the query.  
   - Use VectorStore Qdrant node in load/search mode with a filter on `metadata.company_id` if company IDs are provided.  
   - Aggregate results and return them.

7. **Compare Tool Workflow**  
   - Accepts search query and multiple company IDs.  
   - Use OpenAI Embeddings node for query embeddings.  
   - Use Qdrant HTTP API for grouped search by `company_id`.  
   - Split grouped results, filter by requested company IDs.  
   - Aggregate or return "no results." if empty.

8. **Recommend Tool Workflow**  
   - Accepts positive and negative preference texts and optional company IDs.  
   - Use a Code node to convert preferences into array items.  
   - Use OpenAI Embeddings node to get embeddings for preferences.  
   - Aggregate embeddings and call Qdrant recommendation API via HTTP Request node.  
   - Simplify and aggregate recommendation results or return "no results." if empty.

9. **ListCompanies Tool Workflow**  
   - Calls Qdrant facet API via HTTP Request node to list unique `company_id` values.  
   - Returns the list to MCP client.

10. **Connect Tool Workflows to MCP Server Trigger**  
    - Use Tool Workflow nodes in the main workflow for each operation.  
    - Configure each Tool Workflow node with the corresponding workflow ID.  
    - Connect outputs of MCP Server Trigger to these Tool Workflow nodes via ai_tool connections.

11. **Manual Setup for Qdrant Collection (Optional)**  
    - Create Manual Trigger node.  
    - Add HTTP Request node to create collection `trustpilot_reviews` with cosine distance and vector size 1536.  
    - Add HTTP Request node to create facet index on `metadata.company_id`.  
    - Connect nodes sequentially.

12. **Configure Credentials**  
    - Set up OpenAI API credentials for embedding nodes.  
    - Set up Qdrant API credentials for HTTP Request and VectorStore Qdrant nodes.  
    - Ensure endpoint URLs point to your Qdrant instance.

13. **Add Data Processing Nodes**  
    - Use Set, Aggregate, Filter, Split Out, and If nodes as needed to process API responses and format MCP client responses.

14. **Add Sticky Notes**  
    - Add instructional sticky notes for setup, usage, and reminders about authentication and endpoint configuration.

15. **Test Workflow**  
    - Use MCP client (e.g., Claude Desktop) to connect to the MCP server webhook.  
    - Test queries such as listing companies, inserting reviews, searching, comparing, and recommending.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                        |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is based on the official Qdrant MCP server reference implementation, extended with additional API features like facet search, grouped search, and recommendations.                                                                                                                                                                                               | https://github.com/qdrant/mcp-server-qdrant                                                                                            |
| To integrate an MCP client such as Claude Desktop, follow the n8n guidelines for MCP Trigger node integration.                                                                                                                                                                                                                                                               | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#integrating-with-claude-desktop                     |
| Always enable authentication on the MCP server trigger before deploying to production to secure access.                                                                                                                                                                                                                                                                      | Sticky Note in workflow                                                                                                                |
| Configure your Qdrant API endpoint correctly in all HTTP Request and VectorStore Qdrant nodes to ensure connectivity.                                                                                                                                                                                                                                                        | Sticky Notes in workflow                                                                                                               |
| The OpenAI embedding model used is `text-embedding-3-small`. Ensure your OpenAI API key has access to this model.                                                                                                                                                                                                                                                             | Node configuration                                                                                                                    |
| This workflow demonstrates how to customize MCP servers beyond vendor defaults, enabling tailored business intelligence applications.                                                                                                                                                                                                                                        | Sticky Note1 content                                                                                                                  |
| Prerequisite: You must have a Qdrant collection named `trustpilot_reviews` with a facet index on `metadata.company_id` before using this MCP server. Use the manual setup flow if needed.                                                                                                                                                                                    | Sticky Note3 content                                                                                                                  |
| MCP clients can query this server with example queries like: "Can you help me list the available companies in the collection?" or "What do customers say about product deliveries from company X?"                                                                                                                                                                              | Sticky Note2 content                                                                                                                  |

---

This document provides a detailed, structured reference to understand, reproduce, and customize the Qdrant MCP server workflow implemented in n8n. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with this workflow.