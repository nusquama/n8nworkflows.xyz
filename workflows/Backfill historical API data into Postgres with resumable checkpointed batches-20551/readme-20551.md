Backfill historical API data into Postgres with resumable checkpointed batches

https://n8nworkflows.xyz/workflows/backfill-historical-api-data-into-postgres-with-resumable-checkpointed-batches-20551


# Backfill historical API data into Postgres with resumable checkpointed batches

### 1. Workflow Overview

This workflow is designed to execute on-demand historical data backfills from an external HTTP API into a PostgreSQL database. It features a resumable, checkpointed batch processing architecture that persists progress metrics. If an execution is interrupted, fails, or reaches a per-run batch threshold, subsequent runs automatically continue from the last successfully saved checkpoint cursor or date.

The logical execution follows these functional blocks:
- **1.1 Initialization & Configuration:** Triggers the job manually or via webhook, ensures the PostgreSQL tracking table exists, and loads runtime configurations alongside historical checkpoints.
- **1.2 Batch Retrieval & Normalization:** Fetches chunks of historical records from the source API, normalizes the structure, maps fields, and determines the next pagination cursor.
- **1.3 Data Persistence & State Tracking:** Upserts incoming batches into the target database, updates progress counters, and manages conditional looping behavior before finalizing or pausing execution.

---

### 2. Block-by-Block Analysis

---

### 2.1 Initialization & Configuration

#### Overview
This block initializes the backfill job via manual action or webhook, establishes the state tracking table in PostgreSQL, and defines the primary configuration boundaries (API parameters, date ranges, and database mappings).

#### Nodes Involved
- `Webhook Trigger`
- `Manual Trigger`
- `Postgres - Ensure Progress Table`
- `Code - Load Config & Checkpoint`
- `IF - Job Already Complete?`

#### Node Details

##### Webhook Trigger
- **Type and Technical Role:** `n8n-nodes-base.webhook` (Trigger) — Listens for incoming HTTP requests to initiate backfill runs externally.
- **Configuration Choices:** Uses path `44c54505-5976-4abe-bd76-2ae944040c74`.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: None (Entry point). Output: `Postgres - Ensure Progress Table`.
- **Version-Specific Requirements:** Version 2.1.
- **Edge Cases or Potential Failure Types:** Unauthorized or malformed incoming requests if external authentication is layered outside n8n.

##### Manual Trigger
- **Type and Technical Role:** `n8n-nodes-base.manualTrigger` (Trigger) — Initiates the workflow interactively from the n8n editor.
- **Configuration Choices:** Default parameters.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: None (Entry point). Output: `Postgres - Ensure Progress Table`.
- **Version-Specific Requirements:** Version 1.
- **Edge Cases or Potential Failure Types:** None.

##### Postgres - Ensure Progress Table
- **Type and Technical Role:** `n8n-nodes-base.postgres` (Database) — Executes a DDL query to create the `backfill_progress` tracking table if it does not already exist.
- **Configuration Choices:** Executes custom SQL (`CREATE TABLE IF NOT EXISTS backfill_progress (...)`). `Continue On Fail` is enabled.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: `Manual Trigger`, `Webhook Trigger`. Output: `Code - Load Config & Checkpoint`.
- **Version-Specific Requirements:** Version 2.5. Requires active Postgres credentials.
- **Edge Cases or Potential Failure Types:** Database connection drops or permission errors preventing table creation.

##### Code - Load Config & Checkpoint
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation) — JavaScript node housing core backfill parameters (API endpoints, field maps, batch limits, date ranges) and constructing initial request payloads.
- **Configuration Choices:** Embedded configuration object defining API settings, checkpoint strategies (`date`), and target table configurations.
- **Key Expressions or Variables:** Uses internal JavaScript object definitions.
- **Input and Output Connections:** Input: `Postgres - Ensure Progress Table`. Output: `IF - Job Already Complete?`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases or Potential Failure Types:** Syntax errors in configuration scripts or misconfigured mapping keys.

##### IF - Job Already Complete?
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control) — Evaluates whether the tracking state indicates the job is already finished.
- **Configuration Choices:** Compares `{{ $json.state.status }}` against the string `completed`.
- **Key Expressions or Variables:** `={{ $json.state.status }}`
- **Input and Output Connections:** Input: `Code - Load Config & Checkpoint`. Outputs: True branch connects to `Code - Finalize Job`; False branch connects to `HTTP Request - Fetch Batch`.
- **Version-Specific Requirements:** Version 2.2.
- **Edge Cases or Potential Failure Types:** Evaluation errors if the state object is missing properties.

---

### 2.2 Batch Retrieval & Normalization

#### Overview
This block executes paginated HTTP requests to fetch historical records, standardizes payloads into uniform schemas, validates record counts, and routes valid datasets toward database upsert operations.

#### Nodes Involved
- `HTTP Request - Fetch Batch`
- `Code - Normalize & Validate Batch`
- `IF - Batch Has Records?`

#### Node Details

##### HTTP Request - Fetch Batch
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (Integration) — Calls the external API to retrieve a slice of historical data using dynamic query parameters.
- **Configuration Choices:** Configured for automatic retries (`maxTries: 3`, wait between tries: 3000ms), 45-second timeout, and non-error throwing response returns (`neverError: true`).
- **Key Expressions or Variables:** 
  - URL: `={{ $json.request.url }}`
  - Method: `={{ $json.config.method || 'GET' }}`
  - Query Parameters mapped dynamically from configuration.
- **Input and Output Connections:** Input: `IF - Job Already Complete?` (False branch), `Wait - Between Batches`. Output: `Code - Normalize & Validate Batch`.
- **Version-Specific Requirements:** Version 4.2.
- **Edge Cases or Potential Failure Types:** Rate limiting, API timeouts, invalid authentication tokens, or schema changes from the upstream provider.

##### Code - Normalize & Validate Batch
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation) — Parses raw API response bodies, normalizes fields based on the configuration mapping dictionary, and calculates the next pagination checkpoint.
- **Configuration Choices:** JavaScript execution parsing JSON payloads and handling nested path lookups safely.
- **Key Expressions or Variables:** References prior node outputs via helper functions.
- **Input and Output Connections:** Input: `HTTP Request - Fetch Batch`. Output: `IF - Batch Has Records?`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases or Potential Failure Types:** Malformed JSON bodies or unexpected data types returned by the upstream API.

##### IF - Batch Has Records?
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control) — Verifies whether the retrieved batch contains one or more records before attempting database insertions.
- **Configuration Choices:** Evaluates `={{ $json.records.length }}` > `0`.
- **Key Expressions or Variables:** `={{ $json.records.length }}`
- **Input and Output Connections:** Input: `Code - Normalize & Validate Batch`. Outputs: True branch connects to `Postgres - Upsert Batch`; False branch connects to `Code - Update Running Stats`.
- **Version-Specific Requirements:** Version 2.2.
- **Edge Cases or Potential Failure Types:** Undefined array evaluations if normalization returns null.

---

### 2.3 Data Persistence & State Tracking

#### Overview
This block handles database upsert operations, commits progress checkpoints to storage, updates cumulative statistics, and manages execution loops via delays or finalization triggers.

#### Nodes Involved
- `Postgres - Upsert Batch`
- `Postgres - Save Checkpoint`
- `Code - Update Running Stats`
- `IF - More Batches Left?`
- `Wait - Between Batches`
- `Code - Finalize Job`
- `Wait For Data`
- `NoOp - End`
- `Send Data to Webhook`

#### Node Details

##### Postgres - Upsert Batch
- **Type and Technical Role:** `n8n-nodes-base.postgres` (Database) — Performs bulk `INSERT ... ON CONFLICT` operations against the target historical table.
- **Configuration Choices:** Executes dynamically constructed SQL statements with conflict resolution on the configured unique key. `Continue On Fail` is enabled.
- **Key Expressions or Variables:** JavaScript-evaluated query string mapping configuration column names and data arrays.
- **Input and Output Connections:** Input: `IF - Batch Has Records?` (True branch). Output: `Postgres - Save Checkpoint`.
- **Version-Specific Requirements:** Version 2.5.
- **Edge Cases or Potential Failure Types:** Primary key conflicts without proper unique index definitions, or data type mismatch violations against destination columns.

##### Postgres - Save Checkpoint
- **Type and Technical Role:** `n8n-nodes-base.postgres` (Database) — Persists the latest checkpoint cursor, batch count, and record count metrics to the `backfill_progress` table.
- **Configuration Choices:** Parameterized SQL insert/update statement returning updated row identifiers. `Continue On Fail` is enabled.
- **Key Expressions or Variables:** 
  - Query Replacement: `={{ [ $json.config.jobId, $json.state.nextCheckpoint, $json.config.checkpointType, ($json.state.batchesDone || 0) + 1, ($json.state.recordsDone || 0) + ($json.records || []).length ] }}`
- **Input and Output Connections:** Input: `Postgres - Upsert Batch`. Output: `Code - Update Running Stats`.
- **Version-Specific Requirements:** Version 2.5.
- **Edge Cases or Potential Failure Types:** Database write locks or connection drops during state persistence.

##### Code - Update Running Stats
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation) — Increments run counters, maps forward the next checkpoint cursor, and prepares parameters for subsequent iterations.
- **Configuration Choices:** JavaScript execution updating state objects and query string generators.
- **Key Expressions or Variables:** References data from normalization and parsing nodes.
- **Input and Output Connections:** Input: `Postgres - Save Checkpoint`, `IF - Batch Has Records?` (False branch). Output: `IF - More Batches Left?`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases or Potential Failure Types:** State mutation bugs if downstream processes expect immutable objects.

##### IF - More Batches Left?
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control) — Determines if the workflow should loop back to fetch another batch or proceed to job completion.
- **Configuration Choices:** Evaluates boolean state `={{ $json.shouldContinue }}`.
- **Key Expressions or Variables:** `={{ $json.shouldContinue }}`
- **Input and Output Connections:** Input: `Code - Update Running Stats`. Outputs: True branch connects to `Wait - Between Batches`; False branch connects to `Code - Finalize Job`.
- **Version-Specific Requirements:** Version 2.2.
- **Edge Cases or Potential Failure Types:** Infinite loop risks if limit flags are misconfigured.

##### Wait - Between Batches
- **Type and Technical Role:** `n8n-nodes-base.wait` (Flow Control) — Pauses execution briefly to prevent rate-limiting or overloading the upstream API source.
- **Configuration Choices:** Fixed duration amount of `2` seconds.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: `IF - More Batches Left?` (True branch). Output: `HTTP Request - Fetch Batch`.
- **Version-Specific Requirements:** Version 1.1.
- **Edge Cases or Potential Failure Types:** Execution duration timeouts on massive datasets due to cumulative wait delays.

##### Code - Finalize Job
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation) — Compiles a final execution summary payload marking the job status as completed.
- **Configuration Choices:** JavaScript object creation generating summary statistics and messages.
- **Key Expressions or Variables:** References running state counters and configuration IDs.
- **Input and Output Connections:** Input: `IF - Job Already Complete?` (True branch), `IF - More Batches Left?` (False branch). Output: `Wait For Data`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases or Potential Failure Types:** Missing state keys if accessed prematurely.

##### Wait For Data
- **Type and Technical Role:** `n8n-nodes-base.wait` (Flow Control) — Final utility pause configured with minute units before dispatching completion outcomes.
- **Configuration Choices:** Unit set to minutes.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: `Code - Finalize Job`. Outputs: Connects to `NoOp - End` and `Send Data to Webhook`.
- **Version-Specific Requirements:** Version 1.1.
- **Edge Cases or Potential Failure Types:** None.

##### NoOp - End
- **Type and Technical Role:** `n8n-nodes-base.noOp` (Utility) — Terminates the internal workflow branch cleanly.
- **Configuration Choices:** Default parameters.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: `Wait For Data`. Output: None.
- **Version-Specific Requirements:** Version 1.
- **Edge Cases or Potential Failure Types:** None.

##### Send Data to Webhook
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Integration) — Responds to incoming webhook callers with the finalized backfill execution summary.
- **Configuration Choices:** Default response options.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: `Wait For Data`. Output: None.
- **Version-Specific Requirements:** Version 1.5.
- **Edge Cases or Potential Failure Types:** Connection dropped errors if webhook callers disconnect before response transmission.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - Overview` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## Historical Data Backfill System... |
| `Sticky Note - Setup & Checkpoint` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 1. Setup & Checkpoint... |
| `Sticky Note - Fetch & Process` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 2. Fetch & Process Batch... |
| `Sticky Note - Progress & Loop` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 3. Progress, Loop & Finish... |
| `Manual Trigger` | `n8n-nodes-base.manualTrigger` | Trigger | None | `Postgres - Ensure Progress Table` | 1. Setup & Checkpoint |
| `Webhook Trigger` | `n8n-nodes-base.webhook` | Trigger | None | `Postgres - Ensure Progress Table` | 1. Setup & Checkpoint |
| `Postgres - Ensure Progress Table` | `n8n-nodes-base.postgres` | Database | `Manual Trigger`, `Webhook Trigger` | `Code - Load Config & Checkpoint` | 1. Setup & Checkpoint |
| `Code - Load Config & Checkpoint` | `n8n-nodes-base.code` | Data Transformation | `Postgres - Ensure Progress Table` | `IF - Job Already Complete?` | 1. Setup & Checkpoint |
| `IF - Job Already Complete?` | `n8n-nodes-base.if` | Flow Control | `Code - Load Config & Checkpoint` | `Code - Finalize Job`, `HTTP Request - Fetch Batch` | 1. Setup & Checkpoint |
| `HTTP Request - Fetch Batch` | `n8n-nodes-base.httpRequest` | Integration | `IF - Job Already Complete?`, `Wait - Between Batches` | `Code - Normalize & Validate Batch` | 2. Fetch & Process Batch |
| `Code - Normalize & Validate Batch` | `n8n-nodes-base.code` | Data Transformation | `HTTP Request - Fetch Batch` | `IF - Batch Has Records?` | 2. Fetch & Process Batch |
| `IF - Batch Has Records?` | `n8n-nodes-base.if` | Flow Control | `Code - Normalize & Validate Batch` | `Postgres - Upsert Batch`, `Code - Update Running Stats` | 2. Fetch & Process Batch |
| `Postgres - Upsert Batch` | `n8n-nodes-base.postgres` | Database | `IF - Batch Has Records?` | `Postgres - Save Checkpoint` | 2. Fetch & Process Batch |
| `Postgres - Save Checkpoint` | `n8n-nodes-base.postgres` | Database | `Postgres - Upsert Batch` | `Code - Update Running Stats` | 3. Progress, Loop & Finish |
| `Code - Update Running Stats` | `n8n-nodes-base.code` | Data Transformation | `Postgres - Save Checkpoint`, `IF - Batch Has Records?` | `IF - More Batches Left?` | 3. Progress, Loop & Finish |
| `IF - More Batches Left?` | `n8n-nodes-base.if` | Flow Control | `Code - Update Running Stats` | `Wait - Between Batches`, `Code - Finalize Job` | 3. Progress, Loop & Finish |
| `Wait - Between Batches` | `n8n-nodes-base.wait` | Flow Control | `IF - More Batches Left?` | `HTTP Request - Fetch Batch` | 3. Progress, Loop & Finish |
| `Code - Finalize Job` | `n8n-nodes-base.code` | Data Transformation | `IF - Job Already Complete?`, `IF - More Batches Left?` | `Wait For Data` | 3. Progress, Loop & Finish |
| `Wait For Data` | `n8n-nodes-base.wait` | Flow Control | `Code - Finalize Job` | `NoOp - End`, `Send Data to Webhook` | 3. Progress, Loop & Finish |
| `NoOp - End` | `n8n-nodes-base.noOp` | Utility | `Wait For Data` | None | 3. Progress, Loop & Finish |
| `Send Data to Webhook` | `n8n-nodes-base.respondToWebhook` | Integration | `Wait For Data` | None | 3. Progress, Loop & Finish |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Triggers:**
   - Add a **Manual Trigger** node.
   - Add a **Webhook Trigger** node configured with path `44c54505-5976-4abe-bd76-2ae944040c74`.

2. **Initialize Progress Table:**
   - Add a **Postgres** node named `Postgres - Ensure Progress Table` (Version 2.5).
   - Configure operation to **Execute Query**.
   - Set the query to create the `backfill_progress` table if it does not exist (including columns: `job_id`, `status`, `checkpoint`, `checkpoint_type`, `batches_done`, `records_done`, `last_error`, `started_at`, `updated_at`, `completed_at`).
   - Enable **Continue On Fail**.
   - Connect both triggers to this node.

3. **Load Configuration and State:**
   - Add a **Code** node named `Code - Load Config & Checkpoint` (Version 2).
   - Insert JavaScript configuration logic defining `jobId`, API `baseUrl`, headers, checkpoint strategy (`date`), date boundaries (`startDate`, `endDate`), limits, target table name, unique key, and field maps.
   - Connect `Postgres - Ensure Progress Table` output to this node.

4. **Evaluate Completion Status:**
   - Add an **IF** node named `IF - Job Already Complete?` (Version 2.2).
   - Set condition: Left Value `={{ $json.state.status }}`, Operator **Equals**, Right Value `completed`.
   - Connect `Code - Load Config & Checkpoint` output to this node.

5. **Fetch Historical Batch:**
   - Add an **HTTP Request** node named `HTTP Request - Fetch Batch` (Version 4.2).
   - Configure Method to `={{ $json.config.method || 'GET' }}` and URL to `={{ $json.request.url }}`.
   - Configure query parameters and header parameters dynamically referencing configuration objects.
   - Enable retries (`maxTries: 3`, wait 3000ms) and set response options to never error. Enable **Continue On Fail**.
   - Connect the **False** output of `IF - Job Already Complete?` to this node.

6. **Normalize and Validate Data:**
   - Add a **Code** node named `Code - Normalize & Validate Batch` (Version 2).
   - Insert JavaScript to parse response bodies, map source fields to destination fields, and calculate the next pagination checkpoint.
   - Connect `HTTP Request - Fetch Batch` output to this node.

7. **Verify Record Presence:**
   - Add an **IF** node named `IF - Batch Has Records?` (Version 2.2).
   - Set condition: Left Value `={{ $json.records.length }}`, Operator **Larger Than**, Right Value `0`.
   - Connect `Code - Normalize & Validate Batch` output to this node.

8. **Upsert Batch into Database:**
   - Add a **Postgres** node named `Postgres - Upsert Batch` (Version 2.5).
   - Configure operation to **Execute Query** using a dynamic query builder script that constructs bulk `INSERT ... ON CONFLICT (...) DO UPDATE` statements based on configuration mappings.
   - Enable **Continue On Fail**.
   - Connect the **True** output of `IF - Batch Has Records?` to this node.

9. **Persist Checkpoint State:**
   - Add a **Postgres** node named `Postgres - Save Checkpoint` (Version 2.5).
   - Configure operation to **Execute Query** with a parameterized upsert statement targeting `backfill_progress`.
   - Set query replacements referencing `jobId`, `nextCheckpoint`, `checkpointType`, `batchesDone`, and `recordsDone`.
   - Enable **Continue On Fail**.
   - Connect `Postgres - Upsert Batch` output to this node.

10. **Update Running Statistics:**
    - Add a **Code** node named `Code - Update Running Stats` (Version 2).
    - Insert JavaScript to increment batch and record counters and build subsequent request parameter structures.
    - Connect `Postgres - Save Checkpoint` output, as well as the **False** output of `IF - Batch Has Records?`, to this node.

11. **Evaluate Loop Continuation:**
    - Add an **IF** node named `IF - More Batches Left?` (Version 2.2).
    - Set condition: Left Value `={{ $json.shouldContinue }}`, Operator **True**.
    - Connect `Code - Update Running Stats` output to this node.

12. **Add Loop Delay and Finalization:**
    - Add a **Wait** node named `Wait - Between Batches` (Version 1.1) set to `2` seconds. Connect the **True** output of `IF - More Batches Left?` to this node, and feed its output back into `HTTP Request - Fetch Batch`.
    - Add a **Code** node named `Code - Finalize Job` (Version 2) to compile summary execution payloads. Connect the **True** output of `IF - Job Already Complete?` and the **False** output of `IF - More Batches Left?` to this node.
    - Add a **Wait** node named `Wait For Data` (Version 1.1) set to minute units, connected to `Code - Finalize Job`.
    - Add a **NoOp** node named `NoOp - End` and a **Respond to Webhook** node named `Send Data to Webhook` (Version 1.5). Connect `Wait For Data` output to both nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Historical Data Backfill System architecture notes | Complete backfill system for handling large datasets in resumable batches with persistent state checkpoints. |