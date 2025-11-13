Build an endpoint to perform CRUD operations with multiple HTTP methods

https://n8nworkflows.xyz/workflows/build-an-endpoint-to-perform-crud-operations-with-multiple-http-methods-2490


# Build an endpoint to perform CRUD operations with multiple HTTP methods

### 1. Workflow Overview

This workflow implements a multi-method HTTP API endpoint to perform CRUD (Create, Read, Update, Delete) operations on a data source, specifically Airtable in this template. It exposes two main webhook endpoints: one for collection-level operations (list all records, create a new record) and one for record-level operations (get, update, delete a single record by ID). The workflow is designed for easy customization to target any other database or service by replacing the Airtable nodes.

**Logical Blocks:**

- **1.1 Webhook Input Reception**: Two webhook nodes receive HTTP requests with different methods and paths.
- **1.2 Read Operations**: Nodes to retrieve one record or multiple records.
- **1.3 Create Operation**: Node to create a new record.
- **1.4 Update Operation**: Node to update an existing record.
- **1.5 Delete Operation**: Node to delete a record.
- **1.6 Response Handling**: Dedicated Respond to Webhook nodes provide HTTP responses for each operation.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Input Reception

- **Overview:**  
  Accepts incoming HTTP requests on two endpoints with multiple HTTP methods enabled to distinguish CRUD actions.

- **Nodes Involved:**  
  - Webhook  
  - Webhook (with ID)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (HTTP entry point)  
    - Configuration:  
      - Path: `customers`  
      - HTTP Methods Allowed: All (GET, POST, etc.) via "Allow Multiple HTTP Methods"  
      - Response Mode: Uses downstream Respond to Webhook nodes  
    - Role: Handles collection-level requests for "get many" (GET) and "create" (POST) operations.  
    - Input: External HTTP requests without an ID parameter.  
    - Output: Routes to "Get All" or "Create" Airtable nodes based on HTTP method.  
    - Edge Cases:  
      - Request with unsupported HTTP method may cause failure.  
      - Invalid payload for creation can cause Airtable errors.  

  - **Webhook (with ID)**  
    - Type: Webhook (HTTP entry point)  
    - Configuration:  
      - Path: `customers/:id` (dynamic parameter for record ID)  
      - HTTP Methods Allowed: GET, PUT, DELETE  
      - Response Mode: Uses downstream Respond to Webhook nodes  
    - Role: Handles record-level requests: read single record (GET), update (PUT), delete (DELETE).  
    - Input: External HTTP requests with an ID URL parameter.  
    - Output: Routes to "Get Single", "Get Single1", or "Airtable" nodes depending on method.  
    - Edge Cases:  
      - Missing or invalid `id` parameter leads to no data or failure.  
      - Unsupported HTTP method leads to failure.  

---

#### 1.2 Read Operations

- **Overview:**  
  Retrieves records from Airtable, either all records or a single record filtered by ID.

- **Nodes Involved:**  
  - Get All (Airtable)  
  - Get Single (Airtable)  
  - Get Single1 (Airtable)  

- **Node Details:**

  - **Get All**  
    - Type: Airtable node (search operation)  
    - Configuration:  
      - Operation: `search` (retrieves matching records)  
      - No filter, fetches all records from the selected base and table.  
    - Input: Triggered by Webhook node for GET on collection path.  
    - Output: Passes all records to Respond to Webhook4 node.  
    - Edge Cases:  
      - Airtable API rate limits or connectivity issues.  
      - Empty table returns empty array.  

  - **Get Single**  
    - Type: Airtable node (search operation)  
    - Configuration:  
      - Operation: `search` with filterByFormula: `=({customer_id} = {{ $json.params.id }})`  
      - Limit: 1 record  
      - Retrieves single record matching `customer_id` from URL parameter.  
    - Input: Triggered by Webhook (with ID) node on GET method.  
    - Output: Passes record to Respond to Webhook node.  
    - Edge Cases:  
      - No matching record returns empty result.  
      - Invalid ID format may cause formula error.  

  - **Get Single1**  
    - Type: Airtable node (search operation)  
    - Configuration: Same as Get Single, but used in DELETE operation workflow chain.  
    - Input: Triggered by Webhook (with ID) node on DELETE method.  
    - Output: Passes record data (including Airtable internal ID) to Airtable1 node for deletion.  
    - Edge Cases: Same as Get Single.

---

#### 1.3 Create Operation

- **Overview:**  
  Creates a new record in Airtable using fields passed as query parameters in the HTTP request.

- **Nodes Involved:**  
  - Create (Airtable)  

- **Node Details:**

  - **Create**  
    - Type: Airtable node (create operation)  
    - Configuration:  
      - Operation: `create`  
      - Fields mapped from HTTP query parameters: email, phone, address, last_name, first_name, customer_id  
      - Base and Table specified for customer records.  
    - Input: Triggered by Webhook node on POST method.  
    - Output: Passes created record to Respond to Webhook1 node with HTTP 201 response code.  
    - Edge Cases:  
      - Missing required fields may cause Airtable API errors.  
      - Duplicate customer_id if uniqueness is expected.  
      - Malformed query parameters.  

---

#### 1.4 Update Operation

- **Overview:**  
  Updates an existing record in Airtable identified by customer_id URL parameter with provided query parameters.

- **Nodes Involved:**  
  - Airtable (Update)  

- **Node Details:**

  - **Airtable (Update)**  
    - Type: Airtable node (update operation)  
    - Configuration:  
      - Operation: `update`  
      - Matching column: `customer_id` matched to URL parameter `id`  
      - Fields updated from query parameters: email, phone, address, last_name, first_name, customer_id  
    - Input: Triggered by Webhook (with ID) node on PUT method.  
    - Output: Passes updated record to Respond to Webhook2 node with HTTP 200 response.  
    - Edge Cases:  
      - No matching record found causes no update.  
      - Invalid fields or data types cause Airtable errors.  
      - Partial updates possible if some query parameters omitted.  

---

#### 1.5 Delete Operation

- **Overview:**  
  Deletes a record from Airtable identified by internal Airtable record ID.

- **Nodes Involved:**  
  - Get Single1 (to retrieve record)  
  - Airtable1 (delete operation)  

- **Node Details:**

  - **Get Single1**  
    - Retrieves record to get Airtable internal record ID needed for deletion.  
    - Input: Triggered by Webhook (with ID) on DELETE method.  
    - Output: Passes record ID to Airtable1 node.  
    - Edge Cases: No record found means nothing to delete.

  - **Airtable1 (Delete)**  
    - Type: Airtable node (deleteRecord operation)  
    - Configuration:  
      - Uses internal Airtable record ID from previous node (`{{$json.id}}`)  
    - Input: From Get Single1 node  
    - Output: Passes deletion confirmation to Respond to Webhook5 node with HTTP 200 response code.  
    - Edge Cases:  
      - Invalid or missing record ID causes deletion failure.  
      - Airtable API rate limits or errors.  

---

#### 1.6 Response Handling

- **Overview:**  
  Dedicated Respond to Webhook nodes send HTTP responses with appropriate status codes and data after each CRUD operation.

- **Nodes Involved:**  
  - Respond to Webhook  
  - Respond to Webhook1  
  - Respond to Webhook2  
  - Respond to Webhook4  
  - Respond to Webhook5  

- **Node Details:**

  - **Respond to Webhook**  
    - Used to respond after single record retrieval (GET /customers/:id)  
    - HTTP status: default 200  

  - **Respond to Webhook1**  
    - Used to respond after record creation (POST /customers)  
    - HTTP status: 201 Created  

  - **Respond to Webhook2**  
    - Used to respond after record update (PUT /customers/:id)  
    - HTTP status: 200 OK  

  - **Respond to Webhook4**  
    - Used to respond after get all records (GET /customers)  
    - HTTP status: default 200  

  - **Respond to Webhook5**  
    - Used to respond after record deletion (DELETE /customers/:id)  
    - HTTP status: 200 OK  

- **Edge Cases:**  
  - If upstream nodes fail or return empty data, response may be empty or error.  
  - Proper HTTP codes help clients interpret results.  

---

### 3. Summary Table

| Node Name         | Node Type               | Functional Role                     | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                  |
|-------------------|-------------------------|-----------------------------------|---------------------|------------------------|---------------------------------------------------------------------------------------------|
| Webhook           | Webhook                 | Entry point for collection-level API (GET all, POST create) | -                   | Get All, Create        |                                                                                             |
| Webhook (with ID)  | Webhook                 | Entry point for record-level API (GET, PUT, DELETE by ID)  | -                   | Get Single, Get Single1, Airtable |                                                                                             |
| Get All           | Airtable                | Retrieve all records               | Webhook             | Respond to Webhook4     | #### Get All Retrieves all records                                                          |
| Create            | Airtable                | Create new record                 | Webhook             | Respond to Webhook1     | #### Creation Creates a new record                                                          |
| Get Single        | Airtable                | Retrieve single record by customer_id | Webhook (with ID)    | Respond to Webhook      | #### Get Retrieves a single record                                                          |
| Get Single1       | Airtable                | Retrieve record for deletion (internal ID) | Webhook (with ID)    | Airtable1              | #### Delete Deletes a record                                                                |
| Airtable          | Airtable                | Update existing record by customer_id | Webhook (with ID)    | Respond to Webhook2     | #### Update Updates of an existing record                                                   |
| Airtable1         | Airtable                | Delete record by Airtable internal ID | Get Single1          | Respond to Webhook5     | #### Delete Deletes a record                                                                |
| Respond to Webhook | Respond to Webhook      | Send HTTP response after Get Single | Get Single           | -                      |                                                                                             |
| Respond to Webhook1| Respond to Webhook      | Send HTTP 201 response after Create | Create               | -                      |                                                                                             |
| Respond to Webhook2| Respond to Webhook      | Send HTTP response after Update   | Airtable             | -                      |                                                                                             |
| Respond to Webhook4| Respond to Webhook      | Send HTTP response after Get All  | Get All               | -                      |                                                                                             |
| Respond to Webhook5| Respond to Webhook      | Send HTTP response after Delete   | Airtable1             | -                      |                                                                                             |
| Sticky Note       | Sticky Note             | Notes for Create operation         | -                   | -                      | #### Creation Creates a new record                                                          |
| Sticky Note1      | Sticky Note             | Notes for Get All operation        | -                   | -                      | #### Get All Retrieves all records                                                          |
| Sticky Note2      | Sticky Note             | Notes for Get Single operation     | -                   | -                      | #### Get Retrieves a single record                                                          |
| Sticky Note3      | Sticky Note             | Notes for Update operation         | -                   | -                      | #### Update Updates of an existing record                                                   |
| Sticky Note4      | Sticky Note             | Notes for Delete operation         | -                   | -                      | #### Delete Deletes a record                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node for Collection Operations**  
   - Name: `Webhook`  
   - Type: Webhook  
   - Parameters:  
     - Path: `customers`  
     - HTTP Methods: Enable "Allow Multiple HTTP Methods" (support GET, POST)  
     - Response Mode: `responseNode` (to send response downstream)  

2. **Create Webhook Node for Record Operations**  
   - Name: `Webhook (with ID)`  
   - Type: Webhook  
   - Parameters:  
     - Path: `customers/:id` (dynamic parameter named `id`)  
     - HTTP Methods: Select GET, PUT, DELETE  
     - Enable "Allow Multiple HTTP Methods"  
     - Response Mode: `responseNode`  

3. **Add Airtable Credential**  
   - Setup Airtable credentials as per [Airtable n8n credentials guide](https://docs.n8n.io/integrations/builtin/credentials/airtable/)  
   - Authenticate with API key and ensure access to your target base and table.

4. **Create Get All Node**  
   - Name: `Get All`  
   - Type: Airtable  
   - Operation: `search` (default)  
   - Base: Select your Airtable base  
   - Table: Select your table  
   - Connect input from `Webhook` node (for GET method)  

5. **Create Create Node**  
   - Name: `Create`  
   - Type: Airtable  
   - Operation: `create`  
   - Base and Table: same as above  
   - Map fields under Columns: define fields (e.g., email, phone, address, last_name, first_name, customer_id) with expressions like `{{$json.query.email}}` to read from HTTP query parameters  
   - Connect input from `Webhook` node (for POST method)  

6. **Create Get Single Node**  
   - Name: `Get Single`  
   - Type: Airtable  
   - Operation: `search`  
   - Limit: 1 record  
   - Base and Table: same  
   - FilterByFormula: `=({customer_id} = {{ $json.params.id }})` to filter by URL parameter `id`  
   - Connect input from `Webhook (with ID)` node (for GET method)  

7. **Create Get Single1 Node** (for deletion)  
   - Name: `Get Single1`  
   - Copy configuration from `Get Single`  
   - Connect input from `Webhook (with ID)` node (for DELETE method)  

8. **Create Update Node**  
   - Name: `Airtable`  
   - Type: Airtable  
   - Operation: `update`  
   - Base and Table: same  
   - Matching Columns: `customer_id`  
   - Map columns for update fields (email, phone, address, last_name, first_name, customer_id) from `{{$json.query.FIELD_NAME}}`  
   - Connect input from `Webhook (with ID)` node (for PUT method)  

9. **Create Delete Node**  
   - Name: `Airtable1`  
   - Type: Airtable  
   - Operation: `deleteRecord`  
   - Base and Table: same  
   - Record ID: `{{$json.id}}` (from previous node)  
   - Connect input from `Get Single1` node  

10. **Create Respond to Webhook Nodes for Each Operation**  
    - For Get Single: `Respond to Webhook` (default 200) connected from `Get Single`  
    - For Create: `Respond to Webhook1` (set Response Code 201) connected from `Create`  
    - For Update: `Respond to Webhook2` (default 200) connected from `Airtable` (update)  
    - For Get All: `Respond to Webhook4` (default 200) connected from `Get All`  
    - For Delete: `Respond to Webhook5` (default 200) connected from `Airtable1` (delete)  

11. **Connect the Nodes Appropriately**  
    - `Webhook` node output: connect to `Get All` and `Create` nodes  
    - `Webhook (with ID)` node output: connect to `Get Single` (GET), `Get Single1` (DELETE), and `Airtable` (PUT) nodes  
    - Connect each Airtable operation node to its respective Respond to Webhook node  

12. **Activate the Workflow**  
    - Save and activate the workflow to enable live HTTP endpoints.

13. **Testing**  
    - Use provided webhook URLs (test and production) for sending HTTP requests using GET, POST, PUT, DELETE methods as per your needs.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This template uses Airtable as backend but can be easily customized to use other databases like Postgres, MySQL, Notion, Coda by replacing Airtable nodes. | Workflow description                                                                                   |
| The Webhook nodes use the "Allow Multiple HTTP Methods" feature to enable multiple API actions on the same endpoint path. | n8n Webhook node official docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/ |
| The record-level webhook path includes a dynamic parameter `:id`. This is a current n8n limitation requiring a separate webhook path for operations on a single record. | Workflow description                                                                                   |
| Useful guide on Airtable credentials setup: https://docs.n8n.io/integrations/builtin/credentials/airtable/      | n8n documentation                                                                                      |
| Testing URLs are active only during "Test workflow" mode. Use production webhook URL after activation for real API usage. | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/#webhook-urls                |

---

This completes the structured, comprehensive documentation of the provided n8n workflow enabling multi-method API endpoints for CRUD operations.