Export Odoo Company Data via API Endpoint with JSON/Excel Output Options

https://n8nworkflows.xyz/workflows/export-odoo-company-data-via-api-endpoint-with-json-excel-output-options-9653


# Export Odoo Company Data via API Endpoint with JSON/Excel Output Options

### 1. Workflow Overview

This workflow implements an API endpoint `/api/v1/get-companies` that allows users to query company records stored in an Odoo ERP system. It supports filtering companies by name and optionally exporting results either as JSON or as an Excel (.xlsx) file.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Validation:** Receives the HTTP request via a webhook, extracts query parameters, and validates mandatory inputs.
- **1.2 Data Retrieval from Odoo:** Constructs filters dynamically based on input, queries Odoo’s company records (`res.company`), and fetches matching data.
- **1.3 Data Preparation:** Processes the raw data from Odoo, adds metadata like report generation timestamp, and prepares it for output.
- **1.4 Response Format Decision:** Checks whether the response should be JSON or Excel based on the `response_format` parameter.
- **1.5 Output Generation and Response:** Depending on format, either converts the data to an Excel file and responds with a downloadable file, or responds directly with JSON.

This design supports both integration use cases (JSON responses for APIs or syncs) and human/business reporting (Excel downloads), making it versatile for teams managing company information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
  Listens for incoming HTTP requests on the `/api/v1/get-companies` webhook path. Extracts query parameters and validates the presence of the mandatory `name` parameter (used as a partial match filter for companies). If missing, returns an error message immediately.

- **Nodes Involved:**  
  - Receive Company Request (Webhook)  
  - Prepare Dynamic Filter (Function)  
  - Overview Note10 (Sticky Note)

- **Node Details:**  
  - **Receive Company Request**  
    - Type: Webhook  
    - Role: Entry point for API requests  
    - Config: Path `/api/v1/get-companies`, responseMode set to respond via a subsequent node  
    - Inputs: External HTTP request  
    - Outputs: Forward JSON payload with query parameters  
    - Edge Cases: Missing requests, malformed query parameters  
  - **Prepare Dynamic Filter**  
    - Type: Function  
    - Role: Validate input and prepare Odoo filters dynamically  
    - Configuration:  
      - Checks if `query.name` exists and is non-empty  
      - If invalid, returns JSON with success=false and error message  
      - Otherwise, builds a filter array with a single "ilike" condition on company name  
      - Extracts optional `response_format` parameter (default "json")  
    - Expressions: Uses `$json["query"]` for input  
    - Inputs: Output of webhook node  
    - Outputs: JSON with `filters` array and `response_format` string  
    - Edge Cases: Missing or empty `name` parameter triggers early failure response  
  - **Overview Note10**  
    - Explains the purpose of this block and validation logic

---

#### 2.2 Data Retrieval from Odoo

- **Overview:**  
  Uses the prepared filter to query the Odoo `res.company` model, retrieving all matching company records. The query uses a partial match on the `name` field with case-sensitive "like" operator.

- **Nodes Involved:**  
  - Fetch Companies from Odoo (Odoo Node)  
  - Overview Note11 (Sticky Note)

- **Node Details:**  
  - **Fetch Companies from Odoo**  
    - Type: Odoo  
    - Role: Fetches company data from Odoo ERP system  
    - Configuration:  
      - Resource: `custom` with customResource set to `res.company`  
      - Operation: `getAll` with `returnAll: true`  
      - Filter: Uses the first filter from the previous node dynamically via expression  
      - Fields: Retrieves a predefined list of company fields (name, email, phone, parent relation, country codes, etc.)  
    - Inputs: Filters from Prepare Dynamic Filter node  
    - Outputs: Array of company records matching the filter  
    - Credentials: Uses stored Odoo API credentials (version 18)  
    - Edge Cases: No matching records (empty array), API connection/auth errors, field list customization needed if schema changes  
  - **Overview Note11**  
    - Describes the Odoo company search, highlights case sensitivity, and customization options

---

#### 2.3 Data Preparation

- **Overview:**  
  Validates that results exist; if none, returns an error message. Otherwise, enriches each company record by adding a timestamp (`report_generated_on`). Prepares data for the final output stage.

- **Nodes Involved:**  
  - Prepare Output Data (Function)  
  - Overview Note12 (Sticky Note)

- **Node Details:**  
  - **Prepare Output Data**  
    - Type: Function  
    - Role: Post-processing of Odoo records before output  
    - Configuration:  
      - Checks if input items array is empty or first item is empty  
      - Returns a failure JSON if no companies found  
      - Otherwise, maps each record to include a new field `report_generated_on` with current ISO timestamp  
    - Inputs: Company records from Odoo node  
    - Outputs: Array of enriched company records as JSON  
    - Edge Cases: Empty result sets, date formatting issues  
  - **Overview Note12**  
    - Notes data preparation and response formatting role

---

#### 2.4 Response Format Decision

- **Overview:**  
  Determines whether the client requested JSON or Excel output by examining the `response_format` parameter passed through the workflow.

- **Nodes Involved:**  
  - Check If Excel Required (If)  
  - Overview Note13 (Sticky Note)

- **Node Details:**  
  - **Check If Excel Required**  
    - Type: If  
    - Role: Branching logic based on response format  
    - Configuration: Checks if `response_format` equals `"excel"` (case sensitive)  
    - Inputs: Output from Prepare Output Data node  
    - Outputs:  
      - True branch: Excel export path  
      - False branch: JSON response path  
    - Edge Cases: Case sensitivity in `response_format`, missing parameter defaults to JSON  
  - **Overview Note13**  
    - Summarizes response format branching behavior

---

#### 2.5 Output Generation and Response

- **Overview:**  
  Depending on the branch:  
  - Converts JSON data to an Excel file and responds with binary download if Excel requested  
  - Otherwise, responds directly with the JSON array of company data

- **Nodes Involved:**  
  - Code (Code)  
  - Convert to Excel (ConvertToFile)  
  - Respond with File (RespondToWebhook)  
  - Respond with JSON (RespondToWebhook)  
  - Overview Note14, Overview Note15, Overview Note16, Overview Note17 (Sticky Notes)

- **Node Details:**  
  - **Code**  
    - Type: Code  
    - Role: Pass-through node to aggregate all input items for Excel conversion  
    - Configuration: Returns all input items as-is  
    - Inputs: True branch from If node (Check If Excel Required)  
    - Outputs: Data for Excel conversion  
  - **Convert to Excel**  
    - Type: ConvertToFile  
    - Role: Converts JSON data array to Excel (`.xlsx`) format  
    - Config: Operation set to XLSX conversion with default options  
    - Inputs: Output of Code node  
    - Outputs: Binary Excel file data  
    - Edge Cases: Large data sets might cause memory or timeout issues  
  - **Respond with File**  
    - Type: RespondToWebhook  
    - Role: Sends binary Excel file as HTTP response  
    - Config: Responds with binary data  
    - Inputs: Output of Convert to Excel node  
  - **Respond with JSON**  
    - Type: RespondToWebhook  
    - Role: Sends JSON company data as HTTP response  
    - Config: Responds with all incoming items serialized as JSON  
    - Inputs: False branch output of Check If Excel Required node  
  - **Overview Notes (14 to 17)**  
    - Describe preparation for binary file, Excel file creation, and respective response nodes

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                          | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                           |
|--------------------------|--------------------|----------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Receive Company Request   | Webhook            | API entry point, receives HTTP request | (external HTTP request)       | Prepare Dynamic Filter       | See Overview Note10: Request & Validation block                                                                        |
| Prepare Dynamic Filter    | Function           | Validates input, builds Odoo filters    | Receive Company Request       | Fetch Companies from Odoo    | See Overview Note10                                                                                                    |
| Fetch Companies from Odoo | Odoo               | Queries Odoo `res.company` model        | Prepare Dynamic Filter        | Prepare Output Data          | See Overview Note11: Odoo search logic                                                                                 |
| Prepare Output Data       | Function           | Prepares data, adds timestamp, validates | Fetch Companies from Odoo     | Check If Excel Required      | See Overview Note12                                                                                                    |
| Check If Excel Required   | If                 | Branches based on `response_format`     | Prepare Output Data           | Code (true), Respond with JSON (false) | See Overview Note13                                                                                                    |
| Code                     | Code               | Pass-through for Excel conversion       | Check If Excel Required (true) | Convert to Excel           |                                                                                                                       |
| Convert to Excel          | ConvertToFile      | Converts JSON data to Excel file         | Code                        | Respond with File            | See Overview Note15: Excel file creation                                                                               |
| Respond with File         | RespondToWebhook   | Sends Excel file as HTTP response        | Convert to Excel             | (end)                       | See Overview Note16                                                                                                    |
| Respond with JSON         | RespondToWebhook   | Sends JSON response                      | Check If Excel Required (false) | (end)                    | See Overview Note17                                                                                                    |
| Overview Notes (5,10-17)  | Sticky Notes       | Documentation and explanations           | N/A                         | N/A                         | Provide context, instructions, and detailed explanation of workflow blocks                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: `Webhook`  
   - Name: `Receive Company Request`  
   - Set path to `/api/v1/get-companies`  
   - Set response mode to "responseNode"  
   - Leave other options default

2. **Add Function Node to Prepare Dynamic Filter:**  
   - Name: `Prepare Dynamic Filter`  
   - Paste the following JavaScript code:  
     ```javascript
     const query = $json["query"] || {};
     if (!query.name || query.name.trim() === "") {
       return [{
         json: {
           success: false,
           message: "Missing required parameter: name",
         },
       }];
     }
     const filters = [];
     filters.push(["name", "ilike", query.name]); // mandatory name filter
     const response_format = query.response_format || "json";
     return [{
       json: { filters, response_format },
     }];
     ```  
   - Connect output of Webhook node to this node

3. **Add Odoo Node to Fetch Companies:**  
   - Type: `Odoo`  
   - Name: `Fetch Companies from Odoo`  
   - Credentials: Select or create Odoo API credentials (ensure compatibility with Odoo v18 or your version)  
   - Resource: `custom`  
   - Custom Resource: `res.company`  
   - Operation: `getAll`  
   - Return All: `true`  
   - Fields List: `display_name`, `name`, `email`, `phone`, `mobile`, `parent_id`, `partner_id`, `country_code`, `country_id`  
   - Filter Request:  
     - Field Name: `name`  
     - Operator: `like`  
     - Value: Expression: `={{ $json.filters[0][2] && $json.filters[0][2].toString().trim() !== '' ? $json.filters[0][2] : "False" }}`  
   - Connect output of Prepare Dynamic Filter node to this node

4. **Add Function Node to Prepare Output Data:**  
   - Name: `Prepare Output Data`  
   - Code:  
     ```javascript
     if (items.length === 0 || Object.keys(items[0].json).length === 0) {
       return [{ json: { success: false, message: 'No matching company records found' } }];
     }
     const data = items.map(item => ({ ...item.json, report_generated_on: new Date().toISOString() }));
     return data.map(d => ({ json: d }));
     ```  
   - Connect output of Odoo node to this node

5. **Add If Node to Check Response Format:**  
   - Name: `Check If Excel Required`  
   - Condition type: String  
   - Expression:  
     - `{{$json.response_format}}` equals `excel`  
   - Connect output of Prepare Output Data node to this node

6. **Add Code Node (Pass-through for Excel):**  
   - Name: `Code`  
   - JavaScript: `return $input.all();`  
   - Connect true output of If node to this node

7. **Add ConvertToFile Node:**  
   - Name: `Convert to Excel`  
   - Operation: `xlsx` conversion  
   - Connect output of Code node to this node

8. **Add RespondToWebhook Node for Excel:**  
   - Name: `Respond with File`  
   - Respond with: `binary`  
   - Connect output of Convert to Excel node to this node

9. **Add RespondToWebhook Node for JSON:**  
   - Name: `Respond with JSON`  
   - Respond with: `allIncomingItems`  
   - Connect false output of If node to this node

10. **Verify Connections:**  
    - Webhook -> Prepare Dynamic Filter -> Fetch Companies from Odoo -> Prepare Output Data -> Check If Excel Required  
    - True branch: Check If Excel Required -> Code -> Convert to Excel -> Respond with File  
    - False branch: Check If Excel Required -> Respond with JSON

11. **Test the Workflow:**  
    - Use browser or Postman to make GET requests to the webhook URL with parameters:  
      - Example JSON response: `/api/v1/get-companies?name=Tech&response_format=json`  
      - Example Excel response: `/api/v1/get-companies?name=Tech&response_format=excel`  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                     | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow provides a simple API interface to Odoo company data with optional Excel export, ideal for integration and reporting use cases.                                                                                                   | Overview Note sticky nodes in the workflow          |
| The Odoo node requires valid API credentials configured for your Odoo instance; ensure permissions allow access to `res.company`.                                                                                                               | Odoo node credential configuration                   |
| The name filter uses a case-sensitive `like` operator; be aware of this when searching.                                                                                                                                                          | Overview Note11 sticky                                |
| For large datasets, Excel file conversion may be slow or hit timeout limits; consider pagination or filtering accordingly.                                                                                                                      | General performance consideration                     |
| Quick test URLs example: `/api/v1/get-companies?name=Tech&response_format=json` or `/api/v1/get-companies?name=Tech&response_format=excel`                                                                                                      | Overview Note sticky                                  |
| The workflow adds a timestamp field `report_generated_on` to each returned record, useful for auditing or reporting freshness.                                                                                                                  | Prepare Output Data node                              |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.