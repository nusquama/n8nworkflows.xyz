ClickMeter Link Tracking & Analytics API with Complete Operation Support

https://n8nworkflows.xyz/workflows/clickmeter-link-tracking---analytics-api-with-complete-operation-support-5556


# ClickMeter Link Tracking & Analytics API with Complete Operation Support

### 1. Workflow Overview

This comprehensive n8n workflow named **ClickMeter API MCP Server** is designed to expose the full ClickMeter Link Tracking & Analytics API as an MCP (Modular Connector Protocol) server for AI agents or other clients. It implements **104 distinct API operations** covering all ClickMeter functionalities, enabling granular control and data retrieval.

The workflow is logically structured into functional blocks representing major ClickMeter API resource categories. Each block contains HTTP Request nodes configured to perform CRUD and analytics operations on that resource. The workflow uses a single MCP trigger node as the entry point to serve all AI requests.

**Logical Blocks:**

- **1.1 MCP Server Setup**
  - MCP Trigger node as the server endpoint
  - Setup and advanced usage instructions (Sticky Notes)

- **1.2 Account Management**
  - Nodes to retrieve, update, create, delete accounts and related data (domain whitelist, guests, IP blacklist, plan)

- **1.3 Aggregated Statistics**
  - Nodes to retrieve aggregated account statistics over various timeframes and groupings

- **1.4 Click Stream**
  - Single node to retrieve latest clickstream events

- **1.5 Conversions**
  - Full CRUD and analytics operations on conversions and related datapoints/hits

- **1.6 Data Points**
  - Operations on datapoints including list, create, update, delete, and stats retrieval

- **1.7 Domains**
  - CRUD operations on domains associated with the account

- **1.8 Groups**
  - Nodes to manage groups and retrieve group-related aggregated data

- **1.9 Hits**
  - Single node to retrieve hits/events related to the account

- **1.10 Me**
  - Nodes to retrieve current account and plan information

- **1.11 Re Targeting**
  - Operations on retargeting scripts and their associated datapoints

- **1.12 Tags**
  - Complete management of tags, including association/deassociation with datapoints and groups

Each HTTP Request node uses `$fromAI()` expressions to dynamically populate parameters, enabling AI-driven flexible querying. Authentication is via API key in HTTP headers.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Setup

- **Overview:**  
  This block provides the core MCP server trigger node and documentation nodes to guide advanced users on setup and usage.

- **Nodes Involved:**  
  - ClickMeter MCP Server (MCP Trigger)  
  - Advanced Warning (Sticky Note)  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)

- **Node Details:**  
  - **ClickMeter MCP Server**  
    - Type: MCP Trigger  
    - Role: Entry point for MCP client requests (AI agents)  
    - Config: Webhook path `clickmeter-mcp`  
    - Inputs: None  
    - Outputs: All HTTP request tool nodes  
    - Notes: Serves as single unified entry for all 104 API operations  
    - Failure Modes: Webhook connectivity issues, authentication failures  
  - **Sticky Notes**  
    - Provide detailed instructions, warnings, and overview documentation for users  
    - No functional role in workflow logic

---

#### 1.2 Account Management

- **Overview:**  
  Implements 19 operations related to account data, including retrieving and updating account info, managing domain whitelists, guests, IP blacklists, and plan details.

- **Nodes Involved:**  
  - Get Account 9, Create Account 6, Get Account 10, Create Account 7  
  - Delete Account 3, Get Account 11, Create Account 8, Get Account 12  
  - Delete Account 4, Get Account 13, Create Account 9, Get Account 14  
  - Get Account 15, Create Account 10, Update Account 1, Get Account 16  
  - Create Account 11, Delete Account 5, Get Account 17

- **Node Details (example):**  
  - **Get Account 9**  
    - Type: HTTP Request Tool  
    - Role: Retrieve current account data via GET `/account`  
    - Parameters: None (dynamic via AI expressions if needed)  
    - Input: MCP Trigger  
    - Output: Response data to AI client  
    - Failure: HTTP errors, authentication issues

  - Other nodes follow similar patterns for CRUD operations on account-related resources, using path and query parameters dynamically populated with `$fromAI()`.

- **Potential Failures:**  
  - Missing or invalid API key causing auth errors  
  - Invalid parameter values leading to 4xx errors  
  - Network timeouts or API rate limits

---

#### 1.3 Aggregated Statistics

- **Overview:**  
  Provides 5 endpoints to retrieve aggregated account statistics over specific timeframes with optional grouping.

- **Nodes Involved:**  
  - Get Aggregated 5, Get Aggregated 6, Get Aggregated 7  
  - Get Aggregated 8, Get Aggregated 9

- **Node Details:**  
  - All are HTTP Request Tools issuing GET requests to `/aggregated` endpoints  
  - Use `$fromAI()` to populate parameters like `timeFrame`, `fromDay`, `toDay`, `groupBy`, `status`, etc.  
  - Return statistical data directly to AI clients

- **Failure Handling:**  
  - Invalid timeframes or date formats must be validated on client side  
  - API limit or response errors handled by n8n error handling

---

#### 1.4 Click Stream

- **Overview:**  
  Single operation to retrieve the most recent clickstream events for an account.

- **Nodes Involved:**  
  - Get Clickstream 1

- **Node Details:**  
  - HTTP Request Tool GET `/clickstream` with optional filters for group, datapoint, conversion, pageSize, and filter  
  - Inputs from MCP Trigger  
  - Outputs event list to client

---

#### 1.5 Conversions

- **Overview:**  
  Implements 15 operations managing conversions and their related datapoints and hits, including list, create, update, delete, and aggregated stats.

- **Nodes Involved:**  
  - Get Conversions 3, Create Conversion 2, Get Conversions 4, Get Conversions 5  
  - Delete Conversion 1, Get Conversion 6, Create Conversion 3, Get Conversion 7  
  - Get Conversion 8, Get Conversion 9, Update Conversion 3, Get Conversion 10  
  - Update Conversion 4, Get Conversion 11, Update Conversion 5

- **Node Details:**  
  - Mix of GET, POST, PUT, DELETE HTTP Request Tools targeting `/conversions` and related paths  
  - Extensive use of query and path parameters populated via `$fromAI()`  
  - Supports batch operations and fast update patches  
  - Retrieves hits and datapoints linked to conversions

- **Edge Cases:**  
  - Large data volumes may require pagination (`offset`, `limit`)  
  - Invalid conversion IDs or status values cause errors  
  - Network/API issues typical of REST calls

---

#### 1.6 Data Points

- **Overview:**  
  Provides 16 operations to manage datapoints associated with the user, including retrieval, creation, update, deletion, counts, and aggregated statistics.

- **Nodes Involved:**  
  - Get Datapoints 4, Create Datapoint 3, Get Datapoints 5, Get Datapoints 6  
  - Delete Datapoint 2, Create Datapoint 4, Update Datapoint 3, Get Datapoints 7  
  - Delete Datapoint 3, Get Datapoint 4, Create Datapoint 5, Get Datapoint 5  
  - Get Datapoint 6, Update Datapoint 4, Get Datapoint 7, Update Datapoint 5

- **Node Details:**  
  - HTTP Request Tools targeting `/datapoints` endpoints  
  - Parameters for filtering, pagination, status, tags, and timeframes dynamically handled with `$fromAI()`  
  - Supports batch operations for efficient data updates/deletion

---

#### 1.7 Domains

- **Overview:**  
  Six operations covering CRUD for domains related to the account.

- **Nodes Involved:**  
  - Get Domains 2, Create Domain 2, Get Domains 3, Delete Domain 1  
  - Get Domain 1, Create Domain 3

- **Node Details:**  
  - HTTP Request Tools for `/domains` endpoints  
  - Query parameters for filtering by type, name, offset, limit  
  - Supports domain creation, retrieval, update, and deletion

---

#### 1.8 Groups

- **Overview:**  
  Seventeen nodes managing groups and their datapoints, aggregated statistics, counts, favorites, and notes.

- **Nodes Involved:**  
  - Get Groups 4, Create Group 3, Get Groups 5, Get Groups 6, Get Groups 7  
  - Delete Group 1, Get Group 7, Create Group 4, Get Group 8, Get Group 9  
  - Get Group 10, Get Group 11, Create Group 5, Get Group 12, Update Group 2  
  - Get Group 13, Update Group 3

- **Node Details:**  
  - Extensive support for group data retrieval and manipulation, including aggregation and event hits  
  - Query parameters populated dynamically  
  - Supports favorites toggling and notes patching

---

#### 1.9 Hits

- **Overview:**  
  Single node retrieving list of hits/events related to the account.

- **Nodes Involved:**  
  - Get Hits 1

- **Node Details:**  
  - HTTP Request Tool GET `/hits`  
  - Query parameters for timeframe, limit, offset, filters

---

#### 1.10 Me

- **Overview:**  
  Two nodes providing current authenticated user account and plan information.

- **Nodes Involved:**  
  - Get Me 2, Get Me 3

- **Node Details:**  
  - Simple GET requests to `/me` and `/me/plan` endpoints

---

#### 1.11 Re Targeting

- **Overview:**  
  Eight operations managing retargeting scripts and their datapoints.

- **Nodes Involved:**  
  - Get Retargeting 5, Create Retargeting 2, Get Retargeting 6, Delete Retargeting 1  
  - Get Retargeting 7, Create Retargeting 3, Get Retargeting 8, Get Retargeting 9

- **Node Details:**  
  - Full CRUD and list/count operations on retargeting scripts  
  - Also manages datapoints associated with retargeting scripts

---

#### 1.12 Tags

- **Overview:**  
  Fourteen nodes managing tags and their association with datapoints and groups.

- **Nodes Involved:**  
  - Get Tags 2, Create Tag 1, Get Tags 3, Delete Tag 3, Delete Tag 4, Delete Tag 5  
  - Get Tag 5, Get Tag 6, Get Tag 7, Get Tag 8, Get Tag 9  
  - Update Tag 3, Update Tag 4, Update Tag 5

- **Node Details:**  
  - Comprehensive tag management including create, delete, retrieve, count, and association patching  
  - Dynamic filtering and pagination with `$fromAI()` parameters

---

### 3. Summary Table

| Node Name          | Node Type                  | Functional Role                              | Input Node(s)          | Output Node(s)                               | Sticky Note                                                                                                           |
|--------------------|----------------------------|----------------------------------------------|------------------------|----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| ClickMeter MCP Server | MCP Trigger                | Entry point for AI agent requests             | None                   | All HTTP Request Tool nodes                   |                                                                                                                       |
| Advanced Warning    | Sticky Note                | Usage warnings and advanced user instructions| None                   | None                                         | ⚠️ This workflow is for advanced users only! Recommended to disable unused nodes to maintain performance.              |
| Setup Instructions  | Sticky Note                | Setup and configuration instructions          | None                   | None                                         | Steps to import workflow, configure API key, activate workflow, and connect AI agent with MCP URL.                   |
| Workflow Overview   | Sticky Note                | Project and operations overview                | None                   | None                                         | Overview of MCP server functionality and available API operations grouped by resource type.                           |
| Get Account 9       | HTTP Request Tool          | Retrieve current account data                  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Account 6    | HTTP Request Tool          | Update current account data                     | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Account 10      | HTTP Request Tool          | Retrieve domain whitelist with pagination      | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Account 7    | HTTP Request Tool          | Create a domain whitelist entry                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Account 3    | HTTP Request Tool          | Delete a domain whitelist entry                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Account 11      | HTTP Request Tool          | Retrieve guest list with filters                | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Account 8    | HTTP Request Tool          | Create a guest                                  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Account 12      | HTTP Request Tool          | Retrieve guest count with filter                | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Account 4    | HTTP Request Tool          | Delete a guest                                  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Account 13      | HTTP Request Tool          | Retrieve guest by ID                            | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Account 9    | HTTP Request Tool          | Update guest by ID                              | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Account 14      | HTTP Request Tool          | Retrieve permissions for guest                  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Account 15      | HTTP Request Tool          | Count permissions for guest                      | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Account 10   | HTTP Request Tool          | Change permission on shared object              | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Account 1    | HTTP Request Tool          | Change permission on shared object (PUT)        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Account 16      | HTTP Request Tool          | Retrieve IP blacklist list                       | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Account 11   | HTTP Request Tool          | Create IP blacklist entry                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Account 5    | HTTP Request Tool          | Delete IP blacklist entry                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Account 17      | HTTP Request Tool          | Retrieve current account plan                    | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Aggregated 5    | HTTP Request Tool          | Retrieve aggregated customer statistics          | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Aggregated 6    | HTTP Request Tool          | Retrieve aggregated statistics grouped by time  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Aggregated 7    | HTTP Request Tool          | Retrieve conversion subset stats                  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Aggregated 8    | HTTP Request Tool          | Retrieve datapoints subset statistics             | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Aggregated 9    | HTTP Request Tool          | Retrieve groups subset statistics                  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Clickstream 1   | HTTP Request Tool          | Retrieve latest clickstream events                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversions 3   | HTTP Request Tool          | Retrieve list of conversions                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Conversion 2 | HTTP Request Tool          | Create a conversion                                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversions 4   | HTTP Request Tool          | Retrieve aggregated conversion statistics           | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversions 5   | HTTP Request Tool          | Retrieve count of conversions                         | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Conversion 1 | HTTP Request Tool          | Delete a conversion by ID                             | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversion 6    | HTTP Request Tool          | Retrieve a conversion by ID                           | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Conversion 3 | HTTP Request Tool          | Update a conversion by ID                             | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversion 7    | HTTP Request Tool          | Retrieve aggregated conversion stats by timeframe     | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversion 8    | HTTP Request Tool          | Retrieve aggregated conversion stats grouped by time | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversion 9    | HTTP Request Tool          | Retrieve datapoints linked to conversion              | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Conversion 3 | HTTP Request Tool          | Batch modify datapoints association with conversion    | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversion 10   | HTTP Request Tool          | Count datapoints linked to conversion                   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Conversion 4 | HTTP Request Tool          | Modify association between conversion and datapoint      | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Conversion 11   | HTTP Request Tool          | Retrieve hits/events related to conversion               | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Conversion 5 | HTTP Request Tool          | Fast patch notes field of conversion                       | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Datapoints 4    | HTTP Request Tool          | List datapoints with filters                               | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Datapoint 3  | HTTP Request Tool          | Create a datapoint                                         | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Datapoints 5    | HTTP Request Tool          | Retrieve aggregated datapoints stats                      | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Datapoints 6    | HTTP Request Tool          | Retrieve aggregated datapoints list grouped by time      | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Datapoint 2  | HTTP Request Tool          | Delete multiple datapoints                                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Datapoint 4  | HTTP Request Tool          | Update multiple datapoints                                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Datapoint 3  | HTTP Request Tool          | Create multiple datapoints                                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Datapoints 7    | HTTP Request Tool          | Count datapoints                                           | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Datapoint 3  | HTTP Request Tool          | Delete a datapoint by ID                                   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Datapoint 4     | HTTP Request Tool          | Retrieve a datapoint by ID                                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Datapoint 5  | HTTP Request Tool          | Update a datapoint by ID                                   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Datapoint 5     | HTTP Request Tool          | Retrieve aggregated datapoint stats                         | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Datapoint 6     | HTTP Request Tool          | Retrieve aggregated datapoint stats list grouped by time  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Datapoint 4  | HTTP Request Tool          | Toggle favourite status of datapoint                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Datapoint 7     | HTTP Request Tool          | Retrieve hits/events related to datapoint                   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Datapoint 5  | HTTP Request Tool          | Fast patch notes field of datapoint                         | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Domains 2       | HTTP Request Tool          | Retrieve list of domains with filters                       | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Domain 2     | HTTP Request Tool          | Create a domain                                             | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Domains 3       | HTTP Request Tool          | Retrieve count of domains                                   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Domain 1     | HTTP Request Tool          | Delete a domain by ID                                       | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Domain 1        | HTTP Request Tool          | Get domain by ID                                           | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Domain 3     | HTTP Request Tool          | Update domain by ID                                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Groups 4        | HTTP Request Tool          | List groups with filtering and pagination                   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Group 3      | HTTP Request Tool          | Create a group                                             | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Groups 5        | HTTP Request Tool          | Retrieve aggregated statistics for groups                   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Groups 6        | HTTP Request Tool          | Retrieve groups stats grouped by time                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Groups 7        | HTTP Request Tool          | Count groups                                               | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Group 1      | HTTP Request Tool          | Delete group by ID                                         | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Group 7         | HTTP Request Tool          | Get group by ID                                           | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Group 4      | HTTP Request Tool          | Update group by ID                                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Group 8         | HTTP Request Tool          | Retrieve aggregated stats for a group                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Group 9         | HTTP Request Tool          | Retrieve aggregated stats list for a group grouped by time   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Group 10        | HTTP Request Tool          | Retrieve subset datapoints statistics for a group            | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Group 11        | HTTP Request Tool          | List datapoints in a group                                  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Group 5      | HTTP Request Tool          | Create a datapoint in a group                               | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Group 12        | HTTP Request Tool          | Count datapoints in a group                                  | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Group 2      | HTTP Request Tool          | Toggle favourite status of a group                           | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Group 13        | HTTP Request Tool          | Retrieve hits related to a group                             | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Group 3      | HTTP Request Tool          | Fast patch notes field of a group                            | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Hits 1          | HTTP Request Tool          | Retrieve list of hits for account                            | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Me 2            | HTTP Request Tool          | Retrieve current authenticated user account info            | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Me 3            | HTTP Request Tool          | Retrieve current authenticated user plan info               | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Retargeting 5   | HTTP Request Tool          | List retargeting scripts                                    | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Retargeting 2| HTTP Request Tool          | Create a retargeting script                                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Retargeting 6   | HTTP Request Tool          | Retrieve retargeting script count                            | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Retargeting 1| HTTP Request Tool          | Delete retargeting script                                    | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Retargeting 7   | HTTP Request Tool          | Get retargeting script by ID                                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Retargeting 3| HTTP Request Tool          | Update retargeting script by ID                              | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Retargeting 8   | HTTP Request Tool          | List datapoints associated with retargeting script          | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Retargeting 9   | HTTP Request Tool          | Count datapoints associated with retargeting script          | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Tags 2          | HTTP Request Tool          | List tags with filtering and pagination                      | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Create Tag 1        | HTTP Request Tool          | Create a tag                                               | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Tags 3          | HTTP Request Tool          | Count tags                                               | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Tag 3        | HTTP Request Tool          | Delete tag by ID                                         | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Tag 4        | HTTP Request Tool          | Delete association of tag with all datapoints              | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Delete Tag 5        | HTTP Request Tool          | Delete association of tag with all groups                   | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Tag 5           | HTTP Request Tool          | Retrieve tag by ID                                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Tag 6           | HTTP Request Tool          | List datapoints filtered by tag                             | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Tag 7           | HTTP Request Tool          | Count datapoints filtered by tag                            | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Tag 8           | HTTP Request Tool          | List groups filtered by tag                                 | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Get Tag 9           | HTTP Request Tool          | Count groups filtered by tag                                | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Tag 3        | HTTP Request Tool          | Associate/Deassociate tag with datapoint                    | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Tag 4        | HTTP Request Tool          | Associate/Deassociate tag with group                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |
| Update Tag 5        | HTTP Request Tool          | Fast patch tag name                                        | ClickMeter MCP Server   | AI client                                    |                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add MCP Trigger node:**
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`
   - Parameters:
     - Path: `clickmeter-mcp`
   - This node will serve as the single entry point for all AI requests.

3. **Setup HTTP Header Authentication Credentials:**
   - Create new credentials of type `HTTP Header Auth`
   - Parameter:  
     - Header Name: `X-Clickmeter-AuthKey`  
     - API Key value: Your ClickMeter API key

4. **Add Sticky Note nodes for documentation:**
   - Add notes for advanced warnings, setup instructions, and workflow overview.
   - Use markdown content to explain usage, setup steps, and warnings.

5. **For each ClickMeter API resource group (Account, Aggregated, Click Stream, Conversions, Data Points, Domains, Groups, Hits, Me, Re Targeting, Tags):**

   - Create corresponding HTTP Request Tool nodes for each API endpoint.
   - Configure each node as follows:
     - HTTP Method: GET, POST, PUT, or DELETE as per API
     - URL: Base URL `http://apiv2.clickmeter.com:80` + resource path (e.g., `/account`, `/conversions`)
     - Authentication: Select the HTTP Header Auth credentials created in step 3
     - Query Parameters and Path Parameters: Use n8n expression editor to insert `$fromAI()` expressions to dynamically take parameters from AI requests.
     - Description: Add descriptions to each node explaining its purpose and parameters.
     - Enable `Send Query` when query parameters are used.

6. **Connect all HTTP Request nodes back to the MCP Trigger node:**
   - Set the MCP Trigger node to output to all HTTP Request Tool nodes.
   - Each HTTP Request node represents a tool that AI can invoke.

7. **Configure node parameters carefully:**
   - Use `$fromAI('parameterName', 'Description', 'type', defaultValue)` for every parameter that must be dynamically provided.
   - Validate types and provide default values where applicable.

8. **Test each node individually:**
   - Provide sample parameters and verify API responses are correctly returned.
   - Handle errors gracefully on node level to prevent workflow breaks.

9. **Optimize workflow:**
   - Disable or delete nodes not required for your use case to improve performance.
   - Group nodes logically in the editor with Sticky Notes for clarity.

10. **Activate workflow:**
    - Enable the workflow in n8n to make the MCP server endpoint live.

11. **Connect AI agent or client:**
    - Use the webhook URL from the MCP Trigger node in your AI client's MCP configuration.
    - Ensure your AI client selectively enables only the needed tools to avoid performance degradation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                 | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow contains 104 API operations, exceeding the recommended maximum of 40 tools for most AI clients. Disabling unused nodes is advised for performance reasons.  | Advanced Warning Sticky Note                                                                              |
| Setup requires configuring API Key credentials with header name `X-Clickmeter-AuthKey`.                                                                                      | Setup Instructions Sticky Note                                                                            |
| MCP Trigger node path is `clickmeter-mcp`. Use this webhook URL in your AI agent configuration to connect.                                                                   | Setup Instructions Sticky Note                                                                            |
| Parameters for API calls are populated using `$fromAI()` expressions which enable dynamic AI-driven parameter injection.                                                     | Setup Instructions Sticky Note                                                                            |
| For detailed MCP and n8n integration info, consult official n8n documentation on MCP nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions Sticky Note                                                                            |
| For custom integration help or workflow optimizations, contact the author on Discord: https://discord.me/cfomodz                                                           | Setup Instructions & Advanced Warning Sticky Notes                                                       |
| Group related operations and enable only necessary tools to maintain efficient AI client performance. Creating multiple smaller MCP servers for distinct use cases is recommended. | Advanced Warning Sticky Note                                                                              |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.