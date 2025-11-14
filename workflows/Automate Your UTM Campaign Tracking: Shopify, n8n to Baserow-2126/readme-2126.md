Automate Your UTM Campaign Tracking: Shopify, n8n to Baserow

https://n8nworkflows.xyz/workflows/automate-your-utm-campaign-tracking--shopify--n8n-to-baserow-2126


# Automate Your UTM Campaign Tracking: Shopify, n8n to Baserow

### 1. Workflow Overview

This workflow automates the extraction and centralization of UTM campaign tracking data from Shopify orders into Baserow, enabling marketers to efficiently analyze marketing campaign effectiveness and revenue attribution. It is designed for users who want to streamline campaign data collection, avoid manual errors, and build dynamic campaign databases.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow once daily at midnight.
- **1.2 Shopify Data Retrieval:** Dynamically sets the Shopify subdomain and queries Shopify’s Admin API using GraphQL to retrieve orders created the day before, including detailed UTM parameters from the customer journey.
- **1.3 Data Processing:** Splits the bulk order data into individual items, transforms and normalizes the data structure, and filters for orders that contain campaign information.
- **1.4 Data Storage:** Inserts the filtered UTM campaign data along with order revenue into a Baserow database table; if no campaign data is present, the workflow performs no further action.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Triggers the workflow every day at 00:00 to process the previous day's Shopify orders.
- **Nodes Involved:**  
  - *Every day at 00:00* (Schedule Trigger)
- **Node Details:**  
  - **Every day at 00:00**  
    - Type: Schedule Trigger  
    - Configuration: Default daily interval with no custom time zone specified (assumes server time).  
    - Input: None (trigger node)  
    - Output: Activates downstream nodes once per day.  
    - Failure Modes: Scheduler downtime or misconfiguration may delay or prevent execution.  
    - Notes: Triggers the entire data retrieval and processing sequence.

---

#### 1.2 Shopify Data Retrieval

- **Overview:** Sets the Shopify subdomain dynamically, then queries Shopify’s Admin API using GraphQL to retrieve orders created the previous day, including detailed UTM parameters from the customer journey summary.
- **Nodes Involved:**  
  - *Set Shopify Subdomain* (Set)  
  - *Get orders from Shopify* (GraphQL)
- **Node Details:**  
  - **Set Shopify Subdomain**  
    - Type: Set  
    - Role: Defines the Shopify store subdomain as a string variable for API endpoint construction.  
    - Configuration: Hardcoded string value `"you-domain"` (to be replaced by user with actual domain).  
    - Input: Receives trigger from schedule node.  
    - Output: Provides subdomain string for the GraphQL node.  
    - Edge Cases: If domain is incorrect or missing, API calls will fail.  
    - Notes: Marked with sticky note "Set your Shopify Subdomain here" for user guidance.  

  - **Get orders from Shopify**  
    - Type: GraphQL  
    - Role: Calls Shopify’s Admin GraphQL API to fetch up to 100 orders created yesterday, retrieving order ID, name, total revenue, and nested customer journey UTM parameters.  
    - Configuration:  
      - Query uses a dynamic date filter `created_at:{{$today.minus({days: 1})}}`.  
      - Endpoint dynamically constructed using the subdomain from the previous node.  
      - Authentication: Header Auth with custom header `X-Shopify-Access-Token` using a Shopify API access token credential.  
    - Input: Receives subdomain from *Set Shopify Subdomain* node.  
    - Output: Returns a nested JSON structure with orders and their UTM data.  
    - Version Specifics: Uses Shopify Admin API version `2024-01`.  
    - Failure Modes: Authentication errors, API rate limits, incorrect endpoint, malformed GraphQL query, network issues.  
    - Notes: Sticky note references Shopify API docs and GraphiQL tool for query building.

---

#### 1.3 Data Processing

- **Overview:** Processes the bulk Shopify order data by splitting into individual order items, transforming variables to a flat structure with relevant UTM fields, and filtering only those orders which contain a campaign value.
- **Nodes Involved:**  
  - *Split Shopify data into n8n items* (Split Out)  
  - *Transform incoming data structure* (Set)  
  - *Check if "Campaign" is present* (If)
- **Node Details:**  
  - **Split Shopify data into n8n items**  
    - Type: Split Out  
    - Role: Converts array of orders (`data.orders.edges`) into individual workflow items for granular processing.  
    - Configuration: Splits on the JSON path `data.orders.edges`.  
    - Input: Receives bulk orders JSON from Shopify GraphQL node.  
    - Output: One n8n item per order edge.  
    - Failure Modes: If the path does not exist or data is empty, no items produced.

  - **Transform incoming data structure**  
    - Type: Set  
    - Role: Extracts and flattens relevant fields from each order item into a simplified format suitable for storage.  
    - Configuration:  
      - Maps:  
        - `order` → order name string  
        - `campaign` → UTM campaign string  
        - `content` → UTM content string or empty string if missing  
        - `medium` → UTM medium string or empty string if missing  
        - `source` → UTM medium string (likely a copy-paste mistake, should be UTM source) or empty string  
        - `term` → UTM term string or empty string  
        - `revenue` → totalReceived as number  
      - Includes only these fields (excludes input JSON).  
    - Input: Receives individual order JSON from Split node.  
    - Output: Normalized JSON with key UTM and revenue fields.  
    - Edge Cases: Missing UTM fields replaced with empty string; note that `source` field is assigned from `medium`, which may be a bug or oversight.  
    - Failure Modes: Expression evaluation errors if input JSON structure varies.  

  - **Check if "Campaign" is present**  
    - Type: If  
    - Role: Filters orders to proceed only if the `campaign` field exists and is non-empty.  
    - Configuration: Condition checks if `campaign` string exists in JSON and is not empty.  
    - Input: Receives normalized order data.  
    - Output:  
      - True branch: Orders with campaign data → proceed to storage.  
      - False branch: Orders without campaign → no operation.  
    - Edge Cases: Empty or null campaign fields cause routing to the no-op branch.  
    - Failure Modes: Expression evaluation failure or unexpected data types.

---

#### 1.4 Data Storage

- **Overview:** Stores the filtered order UTM data and revenue into a Baserow database table, or skips if no campaign data is present.
- **Nodes Involved:**  
  - *Baserow* (Baserow node)  
  - *No Operation, do nothing* (NoOp)
- **Node Details:**  
  - **Baserow**  
    - Type: Baserow  
    - Role: Creates new records in a specified Baserow table with mapped fields from the order data.  
    - Configuration:  
      - Database ID: 121  
      - Table ID: 646  
      - Fields mapped:  
        - Order name → fieldId 6164  
        - Campaign → 6165  
        - Content → 6166  
        - Medium → 6167  
        - Source → 6168  
        - Revenue → 6170  
      - Operation: Create record  
    - Credentials: Uses Baserow API credentials.  
    - Input: Receives campaign-filtered order JSON.  
    - Output: Newly created Baserow record info.  
    - Failure Modes: API errors, credential failures, field mapping mismatches, rate limits.  
    - Notes: Sticky note advises user to map fields according to their Baserow structure.

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Terminates the branch where no campaign data is present; effectively skips processing.  
    - Configuration: None  
    - Input: Receives items filtered as having no campaign data.  
    - Output: None  
    - Failure Modes: None (safe no-op).

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                           | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                                                                                                                                                                        |
|-------------------------------|----------------------|-----------------------------------------|---------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Every day at 00:00             | Schedule Trigger     | Initiates workflow daily at midnight    | None                      | Set Shopify Subdomain       |                                                                                                                                                                                                                                                    |
| Set Shopify Subdomain          | Set                  | Defines Shopify store subdomain          | Every day at 00:00        | Get orders from Shopify     | ## Set your Shopify Subdomain here                                                                                                                                                                                                                 |
| Get orders from Shopify        | GraphQL              | Retrieves yesterday's orders with UTM   | Set Shopify Subdomain      | Split Shopify data into n8n items | ## Shopify API  This workflow uses GraphQL calls to the Shopify Admin API. In order to get a better understanding for the queries and mutations please check the API Docs.  [Shopify GraphQL API docs](https://shopify.dev/docs/api/admin-graphql)  To make it easy to build queries for the GraphQL API easy please check out the [GraphiQL App for the Admin API](https://shopify.dev/docs/apps/tools/graphiql-admin-api) from Shopify |
| Split Shopify data into n8n items | Split Out          | Splits bulk order array into individual items | Get orders from Shopify   | Transform incoming data structure |                                                                                                                                                                                                                                                    |
| Transform incoming data structure | Set               | Flattens and maps relevant UTM/revenue fields | Split Shopify data into n8n items | Check if "Campaign" is present | ## Shopify  The n8n Shopify node cannot get the customer journey, so we get this from the Shopify GraphQL API                                                                                                                                       |
| Check if "Campaign" is present | If                   | Filters orders with valid campaign data | Transform incoming data structure | Baserow / No Operation       |                                                                                                                                                                                                                                                    |
| Baserow                       | Baserow               | Stores UTM campaign and revenue data    | Check if "Campaign" is present (true) | None                        | ## Baserow  Please map the fields coming from the IF node to your own structure in Baserow                                                                                                                                                          |
| No Operation, do nothing      | NoOp                  | Ends processing for orders without campaign | Check if "Campaign" is present (false) | None                        |                                                                                                                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Node type: Schedule Trigger  
   - Configure to run once daily at 00:00 (default daily interval).  
   - This node has no input, outputs to *Set Shopify Subdomain*.

2. **Create Set Node "Set Shopify Subdomain":**  
   - Node type: Set  
   - Add one field: Name = `Shopify Subdomain`, Type = string, Value = your actual Shopify subdomain (e.g., "`your-shop-name`").  
   - Connect output of Schedule Trigger to this node.

3. **Create GraphQL Node "Get orders from Shopify":**  
   - Node type: GraphQL  
   - Query: Use the provided GraphQL query (replace subdomain dynamically):  
     ```
     query yersterdaysOrders {
       orders(query: "created_at:{{$today.minus({days: 1})}}", first: 100) {
         edges {
           node {
             id
             name
             totalReceived
             customerJourneySummary {
               firstVisit {
                 id
                 source
                 referrerUrl
                 landingPage
                 utmParameters {
                   campaign
                   content
                   medium
                   source
                   term
                 }
               }
             }
           }
         }
       }
     }
     ```  
   - Endpoint URL:  
     `https://{{ $node["Set Shopify Subdomain"].json["Shopify Subdomain"] }}.myshopify.com/admin/api/2024-01/graphql.json`  
   - Authentication: Header Auth with header name `X-Shopify-Access-Token` and value your Shopify Admin API access token.  
   - Connect *Set Shopify Subdomain* output to this node.

4. **Create Split Out Node "Split Shopify data into n8n items":**  
   - Node type: Split Out  
   - Field to split out: `data.orders.edges` (the array of orders from Shopify API).  
   - Connect output of GraphQL node to this node.

5. **Create Set Node "Transform incoming data structure":**  
   - Node type: Set  
   - Add fields with expressions:  
     - `order`: `{{$json.node.name}}`  
     - `campaign`: `{{$json.node.customerJourneySummary.firstVisit.utmParameters.campaign}}`  
     - `content`: `{{$json.node.customerJourneySummary.firstVisit.utmParameters.content || ""}}`  
     - `medium`: `{{$json.node.customerJourneySummary.firstVisit.utmParameters.medium || ""}}`  
     - `source`: `{{$json.node.customerJourneySummary.firstVisit.utmParameters.source || ""}}` (Note: fix from original source which used medium)  
     - `term`: `{{$json.node.customerJourneySummary.firstVisit.utmParameters.term || ""}}`  
     - `revenue`: Number type, value `{{$json.node.totalReceived}}`  
   - Set to include only these fields, exclude others.  
   - Connect output of Split node to this node.

6. **Create If Node "Check if 'Campaign' is present":**  
   - Node type: If  
   - Condition: Check if `.campaign` field exists and is a non-empty string.  
   - Connect output of Set node to this node.

7. **Create Baserow Node "Baserow":**  
   - Node type: Baserow  
   - Operation: Create  
   - Database ID: your Baserow database (e.g., 121)  
   - Table ID: your Baserow table (e.g., 646)  
   - Map fields accordingly to your Baserow table fields:  
     - Order field → mapped to `order` value  
     - Campaign → `campaign`  
     - Content → `content`  
     - Medium → `medium`  
     - Source → `source`  
     - Revenue → `revenue`  
   - Credentials: Configure Baserow API credentials with API token.  
   - Connect *If* node’s True output (campaign present) to this node.

8. **Create No Operation Node "No Operation, do nothing":**  
   - Node type: NoOp  
   - Connect *If* node’s False output (campaign missing) to this node.  
   - No configuration needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow uses GraphQL calls to the Shopify Admin API. For a detailed understanding of queries and mutations, refer to the official Shopify Admin API documentation. To interactively build queries, use Shopify’s GraphiQL app for the Admin API.                                                             | [Shopify GraphQL API docs](https://shopify.dev/docs/api/admin-graphql)  [GraphiQL Admin API App](https://shopify.dev/docs/apps/tools/graphiql-admin-api) |
| The n8n Shopify node does not provide access to customer journey data, so this workflow uses direct GraphQL API calls instead to retrieve UTM parameters.                                                                                                                                                       | Sticky note in workflow                                                                                          |
| To use this workflow, you must create a custom Shopify app to get an API Access Token. Use this token in n8n Header Auth credentials with header name `X-Shopify-Access-Token`.                                                                                                                                    | Workflow prerequisites                                                                                           |
| You will need a Baserow instance with API access. You can sign up for a free account at [https://baserow.io/](https://baserow.io/). Map the Baserow fields in the node configuration according to your table structure.                                                                                             | Baserow setup instructions                                                                                       |
| Video walkthrough of this workflow is available here: [https://youtu.be/VBeN-3129RM](https://youtu.be/VBeN-3129RM)                                                                                                                                                                                                | Video resource                                                                                                   |

---

This structured documentation provides a detailed understanding of the workflow, enabling users and automation agents to reproduce, modify, and troubleshoot the automation of UTM campaign tracking from Shopify orders into Baserow.