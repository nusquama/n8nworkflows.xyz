Airtable Batch Update / Insert rows (send faster + save API call requests)

https://n8nworkflows.xyz/workflows/airtable-batch-update---insert-rows--send-faster---save-api-call-requests--3394


# Airtable Batch Update / Insert rows (send faster + save API call requests)

### 1. Workflow Overview

This workflow is designed to efficiently batch update or insert rows into an Airtable base by grouping records in batches of 10. Its primary goal is to reduce the number of API calls to Airtable, thus improving performance and respecting Airtable’s rate limits.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception & Preparation:** Starts with manual triggering or input reception, data setup, and initial aggregation.
- **1.2 Subprocess Execution:** Executes a subprocess workflow that handles the batching and actual Airtable API calls.
- **1.3 Batch Processing & Branching:** Splits records into batches, routes them based on operation mode (`upsert`, `insert`, `update`), prepares HTTP requests accordingly.
- **1.4 API Request Handling with Rate Limiting:** Sends HTTP requests to Airtable API with retry logic and rate limit handling.
- **1.5 Output Aggregation:** Collects and merges responses from batch requests.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preparation

**Overview:**  
This block initiates the workflow manually or with test data, sets the fields to be updated/inserted, and aggregates incoming records into a collection for batch processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- random data  
- Set Fields  
- Aggregate

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual workflow execution to test the flow.  
  - Inputs: None  
  - Outputs: Triggers `random data` node  
  - Failure Modes: None (manual trigger)  

- **random data**  
  - Type: Debug Helper (Random Data)  
  - Role: Generates random address data for testing purposes.  
  - Inputs: Triggered by manual trigger  
  - Outputs: Generates sample data to be processed  
  - Failure Modes: None expected  
  - Note: Used only for demonstration/testing.

- **Set Fields**  
  - Type: Set  
  - Role: Prepares and defines the fields matching Airtable columns to be updated or inserted.  
  - Configuration: User must input key-value pairs matching Airtable table columns exactly (case and spelling), with correct data types.  
  - Inputs: From random data or other upstream sources  
  - Outputs: Passes structured data downstream for aggregation  
  - Failure Modes: Misaligned or incorrect field names/types will cause API errors downstream.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all incoming items into a single array under the field `records`.  
  - Inputs: From `Set Fields`  
  - Outputs: Single aggregated output of all records  
  - Failure Modes: None expected

---

#### 2.2 Subprocess Execution

**Overview:**  
Executes a dedicated subprocess workflow to handle batch processing, splitting, and API interactions. This modular approach allows reusability and isolation of batch logic.

**Nodes Involved:**  
- Airtable Batch (Execute Workflow)

**Node Details:**

- **Airtable Batch**  
  - Type: Execute Workflow  
  - Role: Invokes the subprocess workflow (same workflow ID) with inputs: `mode`, `baseId`, `tableIdOrName`, `fieldsToMergeOn`, and `records`.  
  - Configuration:  
    - `mode`: Operation mode (`upsert` by default)  
    - `baseId`: Airtable base identifier (user to specify)  
    - `tableIdOrName`: Airtable table identifier or name  
    - `fieldsToMergeOn`: Array of field names to match records for update/upsert  
    - `records`: Array of records to operate on  
  - Inputs: Aggregated records from previous block  
  - Outputs: None directly; subprocess handles API calls and returns results  
  - Failure Modes: Invalid credentials, incorrect base/table IDs, or malformed inputs may cause failures.

---

#### 2.3 Batch Processing & Branching

**Overview:**  
Inside the subprocess, this block splits the records into batches of 10, then routes each batch through different paths depending on the operation mode (`upsert`, `insert`, `update`). Each path prepares the data for its corresponding Airtable API request.

**Nodes Involved:**  
- Airtable Subprocess (Execute Workflow Trigger)  
- Split Out  
- batch 10 (Split In Batches)  
- return merged output (Code)  
- Switch  
- Edit Fields1  
- Edit Fields2  
- Edit Fields4  
- Aggregate1  
- Aggregate2  
- Aggregate3

**Node Details:**

- **Airtable Subprocess**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point of subprocess, receives parameters and records  
  - Inputs: From `Airtable Batch` (parent workflow)  
  - Outputs: Sends all records to `Split Out` node  
  - Failure Modes: Incorrect input schema may cause errors

- **Split Out**  
  - Type: Split Out  
  - Role: Extracts the `records` array from input and prepares them for batch processing  
  - Configuration: Splits field `records` into individual items, keeps other fields  
  - Inputs: From `Airtable Subprocess`  
  - Outputs: Sends individual records to `batch 10` node

- **batch 10**  
  - Type: Split In Batches  
  - Role: Splits incoming records into batches of 10 for efficient API calls  
  - Configuration: Batch size set to 10  
  - Inputs: Individual records from `Split Out`  
  - Outputs: Batches to `return merged output` and then `Switch` node

- **return merged output**  
  - Type: Code (JavaScript)  
  - Role: Merges batch outputs into a single object containing arrays for all `records`, `updatedRecords`, and `createdRecords` from batch responses  
  - Inputs: Batches from `batch 10`  
  - Outputs: Merged object to `Switch` node  
  - Key Expression: Concatenates arrays from all input batches

- **Switch**  
  - Type: Switch  
  - Role: Routes data based on `mode` field (`update`, `upsert`, `insert`) to the corresponding Edit Fields node  
  - Inputs: Merged batch data from `return merged output`  
  - Outputs:  
    - `update` → `Edit Fields4`  
    - `upsert` → `Edit Fields1`  
    - `insert` → `Edit Fields2`

- **Edit Fields1 (upsert)**  
  - Type: Set  
  - Role: Prepares the payload for upsert API call  
  - Configuration:  
    - Assigns `records.fields` to all fields in `fields` except `id`  
    - Sets `records.id` to the `id` field value (if available)  
  - Inputs: From `Switch` (upsert path)  
  - Outputs: To `Aggregate1`

- **Edit Fields2 (insert)**  
  - Type: Set  
  - Role: Prepares the payload for insert API call  
  - Configuration: Assigns all fields as-is to `records.fields`  
  - Inputs: From `Switch` (insert path)  
  - Outputs: To `Aggregate2`

- **Edit Fields4 (update)**  
  - Type: Set  
  - Role: Prepares the payload for update API call  
  - Configuration: Similar to `Edit Fields1`, assigns fields except `id` and sets `records.id`  
  - Inputs: From `Switch` (update path)  
  - Outputs: To `Aggregate3`

- **Aggregate1, Aggregate2, Aggregate3**  
  - Type: Aggregate  
  - Role: Collects batch outputs for respective modes before sending API requests  
  - Inputs: From respective `Edit Fields` nodes  
  - Outputs: To HTTP request nodes (`upsert`, `insert`, `update`)

---

#### 2.4 API Request Handling with Rate Limiting

**Overview:**  
This block executes the HTTP requests to Airtable API for each batch, handles rate limit errors (HTTP 429), and retries requests with appropriate waits to avoid exceeding limits.

**Nodes Involved:**  
- upsert (HTTP Request)  
- insert (HTTP Request)  
- update (HTTP Request)  
- rate limit? (If)  
- rate limit?1 (If)  
- rate limit?2 (If)  
- Wait 5s, Wait 5s1, Wait 5s2 (Wait)  
- Wait 0.2s to prevent rate limits (Wait)  
- retry request, retry request1, retry request2 (Merge)

**Node Details:**

- **upsert, insert, update**  
  - Type: HTTP Request  
  - Role: Sends PATCH or POST requests to Airtable API depending on mode  
  - Configuration:  
    - URL dynamically constructed from `baseId` and `tableIdOrName` inputs  
    - Method: PATCH for `upsert` and `update`, POST for `insert`  
    - Body: JSON with records prepared in previous blocks  
    - Authentication: Uses Airtable API token credential  
    - Retries: Up to 5 retries with 5-second delay between tries  
  - Inputs: Aggregated records from respective `Aggregate` nodes or retry merges  
  - Outputs: Response objects forwarded to `rate limit?` checks  
  - Failure Modes: Network errors, auth failures, invalid data, rate limits (429)

- **rate limit?, rate limit?1, rate limit?2**  
  - Type: If  
  - Role: Checks if HTTP response status code is 429 (rate limit error)  
  - Inputs: From respective HTTP request nodes  
  - Outputs:  
    - True (rate limited) → Wait node (5 seconds) or short wait (0.2s)  
    - False → Retry merge nodes (`retry request`, etc)  
  - Failure Modes: Misinterpretation of status codes

- **Wait 5s, Wait 5s1, Wait 5s2**  
  - Type: Wait  
  - Role: Pauses execution for 5 seconds to respect API rate limits before retrying  
  - Inputs: If node true output (rate limited)  
  - Outputs: To retry merge nodes

- **Wait 0.2s to prevent rate limits**  
  - Type: Wait  
  - Role: Short delay (0.2 seconds) to prevent hitting rate limits on subsequent calls  
  - Inputs: False outputs of rate limit If nodes  
  - Outputs: To batch processing to continue

- **retry request, retry request1, retry request2**  
  - Type: Merge  
  - Role: Combines original and retried requests choosing the correct branch to retry failed API calls  
  - Inputs: From Wait nodes and HTTP request nodes  
  - Outputs: To HTTP request nodes for re-execution  
  - Failure Modes: Potential infinite retry if rate limit conditions persist (should be monitored)

---

#### 2.5 Output Aggregation

**Overview:**  
Aggregates all batch API responses to return a combined summary of inserted and updated records.

**Nodes Involved:**  
- return merged output (Code, inside batch processing)  
- Aggregate (top-level after Set Fields)  

**Node Details:**

- **return merged output** (already described in 2.3)  
  - Merges batch results for cleaner output downstream.

- **Aggregate** (top-level node)  
  - Aggregates all incoming batch outputs before invoking subprocess for batch processing.

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                                           | Input Node(s)                    | Output Node(s)                       | Sticky Note                                                                                                    |
|-----------------------------|---------------------------|-----------------------------------------------------------|---------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger            | Start workflow manually for testing                       | None                            | random data                        |                                                                                                               |
| random data                 | Debug Helper (random data)| Generate sample test data                                  | When clicking ‘Test workflow’   | Set Fields                        |                                                                                                               |
| Set Fields                 | Set                       | Define/update fields matching Airtable columns            | random data                    | Aggregate                        | ## Set Fields Enter your row data you want to send to Airtable. The key needs to correspond to exact column name. ⚠️ Only use fields which exist in the table ⚠️ |
| Aggregate                  | Aggregate                 | Aggregate all input records into one array                 | Set Fields                     | Airtable Batch                  |                                                                                                               |
| Airtable Batch             | Execute Workflow          | Runs subprocess for batch processing                        | Aggregate                     | None                           | # Subprocess [[API Docs](https://airtable.com/developers/web/api/update-multiple-records)]                    |
| Airtable Subprocess        | Execute Workflow Trigger  | Entry point for batch subprocess                            | Airtable Batch                | Split Out                      |                                                                                                               |
| Split Out                  | Split Out                 | Extracts individual records from batch                      | Airtable Subprocess           | batch 10                      |                                                                                                               |
| batch 10                   | Split In Batches          | Splits records into batches of 10                           | Split Out                    | return merged output, Switch     |                                                                                                               |
| return merged output       | Code                      | Merges batch outputs into unified response                  | batch 10                     | Switch                        |                                                                                                               |
| Switch                     | Switch                    | Routes batches based on mode (upsert, insert, update)      | return merged output          | Edit Fields1, Edit Fields2, Edit Fields4 |                                                                                                               |
| Edit Fields1               | Set                       | Prepare payload for upsert mode                             | Switch (upsert)               | Aggregate1                   |                                                                                                               |
| Edit Fields2               | Set                       | Prepare payload for insert mode                             | Switch (insert)               | Aggregate2                   |                                                                                                               |
| Edit Fields4               | Set                       | Prepare payload for update mode                             | Switch (update)               | Aggregate3                   |                                                                                                               |
| Aggregate1                 | Aggregate                 | Aggregate upsert mode batches                               | Edit Fields1                 | upsert                      |                                                                                                               |
| Aggregate2                 | Aggregate                 | Aggregate insert mode batches                               | Edit Fields2                 | insert                      |                                                                                                               |
| Aggregate3                 | Aggregate                 | Aggregate update mode batches                               | Edit Fields4                 | update                      |                                                                                                               |
| upsert                     | HTTP Request              | Sends PATCH request for upsert                              | Aggregate1                   | rate limit?                 |                                                                                                               |
| insert                     | HTTP Request              | Sends POST request for insert                               | Aggregate2                   | rate limit?1                |                                                                                                               |
| update                     | HTTP Request              | Sends PATCH request for update                              | Aggregate3                   | rate limit?2                |                                                                                                               |
| rate limit?                | If                        | Checks for 429 status (rate limit) on upsert requests      | upsert                       | Wait 5s, Wait 0.2s to prevent rate limits |                                                                                                               |
| rate limit?1               | If                        | Checks for 429 status on insert requests                    | insert                       | Wait 5s1, Wait 0.2s to prevent rate limits |                                                                                                               |
| rate limit?2               | If                        | Checks for 429 status on update requests                    | update                       | Wait 5s2, Wait 0.2s to prevent rate limits |                                                                                                               |
| Wait 5s                    | Wait                      | Waits 5 seconds after rate limit on upsert                  | rate limit?                  | retry request               | ### Adjust if your monthly call limit exceeded On the Team plan this means 2 requests per second [Source](https://support.airtable.com/docs/managing-api-call-limits-in-airtable#monthly-call-limits-for-free-and-team-plans) -> 0.5 second wait |
| Wait 5s1                   | Wait                      | Waits 5 seconds after rate limit on insert                  | rate limit?1                 | retry request1              | See above (same note applies to all rate limit waits)                                                        |
| Wait 5s2                   | Wait                      | Waits 5 seconds after rate limit on update                  | rate limit?2                 | retry request2              | See above                                                                                                     |
| Wait 0.2s to prevent rate limits | Wait                | Short wait (0.2s) to prevent rate limit exceedance          | rate limit? (false branch)   | batch 10                   | See above                                                                                                     |
| retry request              | Merge                     | Merges retry branch for upsert requests                      | Wait 5s, upsert              | upsert                     |                                                                                                               |
| retry request1             | Merge                     | Merges retry branch for insert requests                      | Wait 5s1, insert             | insert                     |                                                                                                               |
| retry request2             | Merge                     | Merges retry branch for update requests                      | Wait 5s2, update             | update                     |                                                                                                               |
| Sticky Note                | Sticky Note               | Note about rate limit handling                               | None                        | None                       | ### Adjust if your monthly call limit exceeded On the Team plan this means 2 requests per second [Source](https://support.airtable.com/docs/managing-api-call-limits-in-airtable#monthly-call-limits-for-free-and-team-plans) -> 0.5 second wait |
| Sticky Note1               | Sticky Note               | Subprocess API docs link and info                            | None                        | None                       | # Subprocess [[API Docs](https://airtable.com/developers/web/api/update-multiple-records)]                    |
| Sticky Note2               | Sticky Note               | Run with test data instruction                               | None                        | None                       | ## Run with test data Connect to Set Fields                                                                  |
| Sticky Note3               | Sticky Note               | Set Fields usage instruction                                 | None                        | None                       | ## Set Fields Enter your row data you want to send to Airtable. The key needs to correspond to the exact column name ⚠️ Only use fields which exist in the table ⚠️ |
| Sticky Note4               | Sticky Note               | Copy nodes instructions                                      | None                        | None                       | # Copy to your workflow                                                                                         |
| Sticky Note5               | Sticky Note               | Explanation of `mode`, `fieldsToMergeOn`, baseId, tableIdOrName | None                        | None                       | ## Airtable Batch mode possible values: `upsert|insert|update`... details on usage and field requirements     |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create a Manual Trigger Node**  
- Type: Manual Trigger  
- Name: `When clicking ‘Test workflow’`  
- Purpose: To manually start the workflow for testing.

**Step 2: Add Random Data Node (Optional for Testing)**  
- Type: Debug Helper (Random Data)  
- Name: `random data`  
- Configuration: Generate sample address data or any test data.  
- Connect: Output from Manual Trigger to this node.

**Step 3: Add Set Node to Define Fields**  
- Type: Set  
- Name: `Set Fields`  
- Configuration: Define key-value pairs matching Airtable table columns exactly (case-sensitive).  
- Connect: Output from `random data` or other input to this node.

**Step 4: Add Aggregate Node to Collect Records**  
- Type: Aggregate  
- Name: `Aggregate`  
- Configuration: Aggregate all incoming items into a single array under `records`.  
- Connect: Output from `Set Fields` to this node.

**Step 5: Add Execute Workflow Node for Subprocess**  
- Type: Execute Workflow  
- Name: `Airtable Batch`  
- Configuration:  
  - Workflow ID: Set to the subprocess workflow (can be the same workflow in this case).  
  - Inputs: Define expected inputs (`mode`, `baseId`, `tableIdOrName`, `fieldsToMergeOn`, `records`).  
  - Example default values:  
    - `mode`: `upsert`  
    - `baseId`: Your Airtable Base ID (e.g., `appXXXXXXXXXXXXX`)  
    - `tableIdOrName`: Your Airtable Table ID or name (e.g., `tblXXXXXXXXXXXXX`)  
    - `fieldsToMergeOn`: Array of strings for matching, e.g., `["field1","field2"]`  
    - `records`: From aggregated records  
- Connect: Output from `Aggregate` to this node.

---

**Inside Subprocess (Same Workflow or Separate)**

**Step 6: Add Execute Workflow Trigger Node**  
- Type: Execute Workflow Trigger  
- Name: `Airtable Subprocess`  
- Configuration: Define inputs as above, to accept parameters and records.  
- Connect: Input from `Airtable Batch` node.

**Step 7: Add Split Out Node**  
- Type: Split Out  
- Name: `Split Out`  
- Configuration: Split field `records` into individual items, keep other data.  
- Connect: From `Airtable Subprocess`.

**Step 8: Add Split In Batches Node**  
- Type: Split In Batches  
- Name: `batch 10`  
- Configuration: Batch size = 10.  
- Connect: From `Split Out`.

**Step 9: Add Code Node to Merge Batch Output**  
- Type: Code  
- Name: `return merged output`  
- Code: Concatenate arrays of `records`, `updatedRecords`, `createdRecords` from batch results.  
- Connect: From `batch 10`.

**Step 10: Add Switch Node**  
- Type: Switch  
- Name: `Switch`  
- Configuration: Route based on `mode` field: `update`, `upsert`, `insert`.  
- Connect: From `return merged output`.

**Step 11: Add Edit Fields Nodes for Each Mode**  
- Type: Set  
- Names: `Edit Fields1` (upsert), `Edit Fields2` (insert), `Edit Fields4` (update)  
- Configuration:  
  - For `upsert` and `update`: Assign `records.fields` excluding `id`, set `records.id` if available.  
  - For `insert`: Assign all fields as-is.  
- Connect: Each output of `Switch` to corresponding Edit Fields node.

**Step 12: Add Aggregate Nodes for Each Mode**  
- Type: Aggregate  
- Names: `Aggregate1` (upsert), `Aggregate2` (insert), `Aggregate3` (update)  
- Connect: From corresponding Edit Fields node.

**Step 13: Add HTTP Request Nodes for Each Mode**  
- Type: HTTP Request  
- Names: `upsert` (PATCH), `insert` (POST), `update` (PATCH)  
- Configuration:  
  - URL: `https://api.airtable.com/v0/{{ baseId }}/{{ tableIdOrName }}`  
  - Method: PATCH for `upsert` and `update`, POST for `insert`  
  - Body: JSON with `records` from Aggregate nodes  
  - Authentication: Airtable Token Credential (OAuth or API key)  
  - Retry: Max 5 tries, 5 seconds wait between retries  
- Connect: From respective Aggregate nodes.

**Step 14: Add If Nodes to Check Rate Limits**  
- Type: If  
- Names: `rate limit?`, `rate limit?1`, `rate limit?2`  
- Configuration: Check if HTTP status code equals 429 (rate limit)  
- Connect: From respective HTTP Request nodes.

**Step 15: Add Wait Nodes for Rate Limit Handling**  
- Type: Wait  
- Names: `Wait 5s`, `Wait 5s1`, `Wait 5s2` (5 seconds wait after 429)  
- Name: `Wait 0.2s to prevent rate limits` (short wait to prevent hitting limits)  
- Connect: True branch of If nodes to 5s Wait, False branch to 0.2s Wait.

**Step 16: Add Merge Nodes for Retry Logic**  
- Type: Merge  
- Names: `retry request`, `retry request1`, `retry request2`  
- Configuration: Mode “chooseBranch” to retry failed requests  
- Connect: From Wait nodes and HTTP Request nodes respectively.

**Step 17: Connect Retry Merge Nodes Back to HTTP Requests**  
- Connect merge nodes back to HTTP Request nodes to enable retry loop.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| # Subprocess [[API Docs](https://airtable.com/developers/web/api/update-multiple-records)]                                         | Reference link to Airtable API documentation for batch record updates.                                       |
| ### Adjust if your monthly call limit exceeded On the Team plan this means 2 requests per second [Source](https://support.airtable.com/docs/managing-api-call-limits-in-airtable#monthly-call-limits-for-free-and-team-plans) -> 0.5 second wait | Explanation about Airtable API call rate limits and recommended wait times to avoid errors.                  |
| ## Set Fields Enter your row data you want to send to Airtable. The key needs to correspond to the exact column name ⚠️ Only use fields which exist in the table ⚠️ | Instruction emphasizing correct field naming and existence in Airtable tables.                               |
| ## Run with test data Connect to Set Fields                                                                                         | Guidance on how to test the workflow using sample data.                                                     |
| ## Airtable Batch mode possible values: `upsert|insert|update` with details on `fieldsToMergeOn`, `baseId`, and `tableIdOrName` usage | Important operational parameters for the batch workflow.                                                    |

---

This detailed documentation should enable users and AI agents to understand, reproduce, and customize the Airtable batch update/insert workflow fully, while anticipating typical errors and integration challenges.