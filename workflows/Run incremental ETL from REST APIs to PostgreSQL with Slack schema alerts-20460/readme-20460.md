Run incremental ETL from REST APIs to PostgreSQL with Slack schema alerts

https://n8nworkflows.xyz/workflows/run-incremental-etl-from-rest-apis-to-postgresql-with-slack-schema-alerts-20460


# Run incremental ETL from REST APIs to PostgreSQL with Slack schema alerts

### 1. Workflow Overview

This workflow executes an automated, incremental Extract, Transform, Load (ETL) pipeline running every 15 minutes. It extracts updated records from a paginated REST API, dynamically adapts a PostgreSQL destination table and schema registry when source schemas change, securely upserts records, and sends structured alert notifications via Slack.

The workflow logic is divided into the following functional blocks:
- **1.1 One-Time Setup:** Initializes the PostgreSQL database tables required for pipeline state tracking, schema mapping, and data storage.
- **1.2 Schedule & Load State:** Initiates execution on a 15-minute cron schedule, loads pipeline parameters, retrieves the last successfully processed watermark, and fetches the current schema registry.
- **1.3 Incremental Extract:** Builds extraction requests using incremental watermarks, calls the REST API, parses/flattens responses, de-duplicates payloads, and handles extraction errors.
- **1.4 Schema-Change Detection & Automatic Mapping:** Infers incoming data types, compares fields against the schema registry, generates and applies DDL modifications, updates tracking logs, and notifies Slack of changes.
- **1.5 Upsert & Verify:** Prepares and executes batch upserts into the target table, evaluates operation success, and flags database-level load failures for alerting.
- **1.6 Advance Watermark & Page Loop:** Conditionally updates the pipeline watermark upon successful writes and implements pagination loops to consume remaining source pages safely.

---

### 2. Block-by-Block Analysis

#### Block 1.1: One-Time Setup
- **Overview:** Initializes necessary PostgreSQL tracking tables (`etl_state`, `etl_schema_registry`, `etl_schema_changes`) and the base destination table (`dw_orders`) manually.
- **Nodes Involved:** 
  - `Manual Trigger - Run Setup Once`
  - `Postgres - Create ETL Tables`
- **Node Details:**
  - `Manual Trigger - Run Setup Once`: 
    - Type: `n8n-nodes-base.manualTrigger`
    - Role: Entry point for manual schema creation.
    - Input: None. Output: Triggers creation query.
  - `Postgres - Create ETL Tables`:
    - Type: `n8n-nodes-base.postgres`
    - Role: Executes DDL statements to construct state and destination schemas.
    - Configuration: Executes multiple `CREATE TABLE IF NOT EXISTS` commands.
    - Input: `Manual Trigger - Run Setup Once`. Output: `Postgres - Get Watermark` (via logical dependency).

#### Block 1.2: Schedule & Load State
- **Overview:** Periodically triggers the ETL pipeline, establishes core configurations, fetches the current synchronization watermark, and loads existing column mappings from PostgreSQL.
- **Nodes Involved:**
  - `Schedule Trigger - Every 15 Minutes`
  - `Set ETL Config`
  - `Postgres - Get Watermark`
  - `Postgres - Load Schema Registry`
- **Node Details:**
  - `Schedule Trigger - Every 15 Minutes`:
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Triggers pipeline execution every 15 minutes.
  - `Set ETL Config`:
    - Type: `n8n-nodes-base.set`
    - Role: Defines static variables including source endpoints, primary keys, and batch sizes.
  - `Postgres - Get Watermark`:
    - Type: `n8n-nodes-base.postgres`
    - Role: Queries `etl_state` to retrieve the last sync timestamp (`watermark`).
  - `Postgres - Load Schema Registry`:
    - Type: `n8n-nodes-base.postgres`
    - Role: Loads active field-to-column mappings from `etl_schema_registry`.

#### Block 1.3: Incremental Extract
- **Overview:** Constructs API query payloads incorporating watermark overlap windows, executes HTTP requests, flattens nested records, and de-duplicates items.
- **Nodes Involved:**
  - `Code - Build Extract Request`
  - `HTTP - Extract Source Page`
  - `Code - Parse & Flatten Records`
  - `Switch - Route Extract Result`
  - `HTTP - Slack Extract Failure Alert`
- **Node Details:**
  - `Code - Build Extract Request`:
    - Type: `n8n-nodes-base.code`
    - Role: Calculates the overlap window and builds query string objects.
  - `HTTP - Extract Source Page`:
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Fetches data pages from the REST API endpoint using Generic Header Auth.
    - Edge Cases: Network timeouts, invalid authentication credentials, non-200 HTTP statuses.
  - `Code - Parse & Flatten Records`:
    - Type: `n8n-nodes-base.code`
    - Role: Validates HTTP responses, flattens single-level JSON objects, and de-duplicates records by primary key.
  - `Switch - Route Extract Result`:
    - Type: `n8n-nodes-base.switch`
    - Role: Routes execution based on whether records require loading or if an error occurred.
  - `HTTP - Slack Extract Failure Alert`:
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Sends extraction error notifications to the configured Slack webhook.

#### Block 1.4: Schema-Change Detection & Automatic Mapping
- **Overview:** Analyzes data types, detects schema modifications (new columns, widened types, missing fields), applies DDL changes, and logs changes to PostgreSQL and Slack.
- **Nodes Involved:**
  - `Code - Detect Schema Changes`
  - `Schema Changed?`
  - `Postgres - Apply Schema Changes`
  - `Postgres - Update Schema Registry`
  - `HTTP - Slack Schema Change Alert`
- **Node Details:**
  - `Code - Detect Schema Changes`:
    - Type: `n8n-nodes-base.code`
    - Role: Infers data types, detects schema alterations, and prepares DDL statements.
  - `Schema Changed?`:
    - Type: `n8n-nodes-base.if`
    - Role: Evaluates whether schema alterations were detected.
  - `Postgres - Apply Schema Changes`:
    - Type: `n8n-nodes-base.postgres`
    - Role: Executes dynamic DDL queries (`ALTER TABLE`).
  - `Postgres - Update Schema Registry`:
    - Type: `n8n-nodes-base.postgres`
    - Role: Persists updated schema definitions and change history logs.
  - `HTTP - Slack Schema Change Alert`:
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Posts schema change summaries to Slack.

#### Block 1.5: Upsert & Verify
- **Overview:** Prepares idempotent batch upserts using `jsonb_populate_recordset`, writes data to the target table, and validates execution success.
- **Nodes Involved:**
  - `Code - Build Upsert Statement`
  - `Postgres - Upsert Records`
  - `Code - Check Load Result`
  - `Load Succeeded?`
  - `HTTP - Slack Load Failure Alert`
- **Node Details:**
  - `Code - Build Upsert Statement`:
    - Type: `n8n-nodes-base.code`
    - Role: Formats SQL UPSERT queries matching target columns.
  - `Postgres - Upsert Records`:
    - Type: `n8n-nodes-base.postgres`
    - Role: Executes the batched upsert query.
  - `Code - Check Load Result`:
    - Type: `n8n-nodes-base.code`
    - Role: Verifies operation status and formats load error messages.
  - `Load Succeeded?`:
    - Type: `n8n-nodes-base.if`
    - Role: Determines if the batch write completed successfully.
  - `HTTP - Slack Load Failure Alert`:
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Alerts Slack if record upserts fail.

#### Block 1.6: Advance Watermark & Page Loop
- **Overview:** Advances the pipeline watermark state upon successful loads and loops back to fetch subsequent pages until exhaustion or page caps are met.
- **Nodes Involved:**
  - `Postgres - Advance Watermark`
  - `Code - Page Summary`
  - `More Pages To Fetch?`
- **Node Details:**
  - `Postgres - Advance Watermark`:
    - Type: `n8n-nodes-base.postgres`
    - Role: Updates the stored watermark in `etl_state`.
  - `Code - Page Summary`:
    - Type: `n8n-nodes-base.code`
    - Role: Calculates whether additional pages require fetching based on batch size and run limits.
  - `More Pages To Fetch?`:
    - Type: `n8n-nodes-base.if`
    - Role: Loops execution back to `Postgres - Get Watermark` if further pages exist.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation covering architectural overview and setup steps. | None | None | Incremental ETL with Schema-Change Detection and Automatic Mapping |
| Note: 1. One-time setup | n8n-nodes-base.stickyNote | Visual container for setup documentation. | None | None | 1. One-time setup |
| Note: 2. Schedule & load state | n8n-nodes-base.stickyNote | Visual container for trigger and state loading steps. | None | None | 2. Schedule & load state |
| Note: 3. Incremental extract | n8n-nodes-base.stickyNote | Visual container for extraction processes. | None | None | 3. Incremental extract |
| Note: 4. Schema-change detection & automatic mapping | n8n-nodes-base.stickyNote | Visual container for schema analysis steps. | None | None | 4. Schema-change detection & automatic mapping |
| Note: 5. Upsert & verify | n8n-nodes-base.stickyNote | Visual container for batch upsert workflows. | None | None | 5. Upsert & verify |
| Note: 6. Advance watermark & page loop | n8n-nodes-base.stickyNote | Visual container for watermark advancement logic. | None | None | 6. Advance watermark & page loop |
| Manual Trigger - Run Setup Once | n8n-nodes-base.manualTrigger | Initiates one-time database setup manually. | None | Postgres - Create ETL Tables | 1. One-time setup |
| Postgres - Create ETL Tables | n8n-nodes-base.postgres | Creates state, registry, change log, and destination tables. | Manual Trigger - Run Setup Once | None | 1. One-time setup |
| Schedule Trigger - Every 15 Minutes | n8n-nodes-base.scheduleTrigger | Triggers pipeline every 15 minutes. | None | Set ETL Config | 2. Schedule & load state |
| Set ETL Config | n8n-nodes-base.set | Sets configuration variables for the ETL pipeline. | Schedule Trigger - Every 15 Minutes | Postgres - Get Watermark | 2. Schedule & load state |
| Postgres - Get Watermark | n8n-nodes-base.postgres | Retrieves the latest synchronization watermark from state. | Set ETL Config, More Pages To Fetch? | Postgres - Load Schema Registry | 2. Schedule & load state |
| Postgres - Load Schema Registry | n8n-nodes-base.postgres | Loads field-to-column mappings. | Postgres - Get Watermark | Code - Build Extract Request | 2. Schedule & load state |
| Code - Build Extract Request | n8n-nodes-base.code | Prepares extraction request parameters and overlap windows. | Postgres - Load Schema Registry | HTTP - Extract Source Page | 3. Incremental extract |
| HTTP - Extract Source Page | n8n-nodes-base.httpRequest | Requests data pages from the REST API endpoint. | Code - Build Extract Request | Code - Parse & Flatten Records | 3. Incremental extract |
| Code - Parse & Flatten Records | n8n-nodes-base.code | Parses HTTP output, flattens nesting, and de-duplicates records. | HTTP - Extract Source Page | Switch - Route Extract Result | 3. Incremental extract |
| Switch - Route Extract Result | n8n-nodes-base.switch | Routes execution based on extraction outcomes. | Code - Parse & Flatten Records | Code - Detect Schema Changes, HTTP - Slack Extract Failure Alert | 3. Incremental extract |
| HTTP - Slack Extract Failure Alert | n8n-nodes-base.httpRequest | Sends extraction failure alerts to Slack. | Switch - Route Extract Result | None | 3. Incremental extract |
| Code - Detect Schema Changes | n8n-nodes-base.code | Infers types and identifies schema alterations. | Switch - Route Extract Result | Schema Changed? | 4. Schema-change detection & automatic mapping |
| Schema Changed? | n8n-nodes-base.if | Evaluates if schema modifications were detected. | Code - Detect Schema Changes | Postgres - Apply Schema Changes, Code - Build Upsert Statement | 4. Schema-change detection & automatic mapping |
| Postgres - Apply Schema Changes | n8n-nodes-base.postgres | Executes generated DDL statements against PostgreSQL. | Schema Changed? | Postgres - Update Schema Registry | 4. Schema-change detection & automatic mapping |
| Postgres - Update Schema Registry | n8n-nodes-base.postgres | Updates schema registry entries and logs schema changes. | Postgres - Apply Schema Changes | Code - Build Upsert Statement, HTTP - Slack Schema Change Alert | 4. Schema-change detection & automatic mapping |
| HTTP - Slack Schema Change Alert | n8n-nodes-base.httpRequest | Posts schema change alerts to Slack. | Postgres - Update Schema Registry | None | 4. Schema-change detection & automatic mapping |
| Code - Build Upsert Statement | n8n-nodes-base.code | Builds SQL query strings for batch upserts. | Schema Changed?, Postgres - Update Schema Registry | Postgres - Upsert Records | 5. Upsert & verify |
| Postgres - Upsert Records | n8n-nodes-base.postgres | Executes batch upserts against the destination table. | Code - Build Upsert Statement | Code - Check Load Result | 5. Upsert & verify |
| Code - Check Load Result | n8n-nodes-base.code | Evaluates upsert success metrics. | Postgres - Upsert Records | Load Succeeded? | 5. Upsert & verify |
| Load Succeeded? | n8n-nodes-base.if | Routes execution based on database load status. | Code - Check Load Result | Postgres - Advance Watermark, HTTP - Slack Load Failure Alert | 5. Upsert & verify |
| HTTP - Slack Load Failure Alert | n8n-nodes-base.httpRequest | Sends load failure notifications to Slack. | Load Succeeded? | None | 5. Upsert & verify |
| Postgres - Advance Watermark | n8n-nodes-base.postgres | Updates synchronization watermark state upon successful load. | Load Succeeded? | Code - Page Summary | 6. Advance watermark & page loop |
| Code - Page Summary | n8n-nodes-base.code | Determines whether additional pages require retrieval. | Postgres - Advance Watermark | More Pages To Fetch? | 6. Advance watermark & page loop |
| More Pages To Fetch? | n8n-nodes-base.if | Loops back to retrieve subsequent pages if available. | Code - Page Summary | Postgres - Get Watermark | 6. Advance watermark & page loop |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Setup Trigger Node:** Add a `Manual Trigger` node named `Manual Trigger - Run Setup Once`.
2. **Create Table Initialization Node:** Add a `Postgres` node named `Postgres - Create ETL Tables`. Connect it to the manual trigger. Configure it to execute the SQL DDL statements creating `etl_state`, `etl_schema_registry`, `etl_schema_changes`, and `dw_orders`. Attach valid PostgreSQL credentials.
3. **Create Schedule Trigger Node:** Add a `Schedule Trigger` node named `Schedule Trigger - Every 15 Minutes`. Configure it with an interval of 15 minutes.
4. **Create Configuration Node:** Add a `Set` node named `Set ETL Config`. Connect the schedule trigger to it. Define parameters: `pipelineName` ("orders_etl"), `sourceUrl` ("https://api.example.com/v1/orders"), `recordsPath` ("data"), `pkField` ("id"), `updatedAtField` ("updated_at"), `pageSize` (200), `maxPagesPerRun` (10), `overlapMinutes` (5), `minSampleForRemoval` (20), `destTable` ("dw_orders"), and `slackWebhookUrl`.
5. **Create State Retrieval Node:** Add a `Postgres` node named `Postgres - Get Watermark`. Connect it after `Set ETL Config` and loop return points. Query the watermark using expression `={{ [ $('Set ETL Config').first().json.pipelineName ] }}`.
6. **Create Registry Loading Node:** Add a `Postgres` node named `Postgres - Load Schema Registry`. Connect it after `Postgres - Get Watermark`. Query records matching the pipeline name.
7. **Create Extract Request Builder:** Add a `Code` node named `Code - Build Extract Request`. Connect it after schema registry loading. Implement extraction parameter and overlap timestamp generation logic.
8. **Create HTTP Request Node:** Add an `HTTP Request` node named `HTTP - Extract Source Page`. Connect it after the request builder. Set URL to `={{ $json.url }}`, configure Generic Header Auth, enable JSON query parameters, and set `OnError` to continue regular output.
9. **Create Parser and Flattener:** Add a `Code` node named `Code - Parse & Flatten Records`. Connect it after the HTTP request. Implement response validation, single-level property flattening, and primary key de-duplication.
10. **Create Routing Switch:** Add a `Switch` node named `Switch - Route Extract Result`. Connect it after parsing. Route payloads where `$json.route === 'load'` to schema detection and errors to Slack alerts.
11. **Create Extract Alert Node:** Add an `HTTP Request` node named `HTTP - Slack Extract Failure Alert`. Connect it to the error output of the switch to POST error text to Slack.
12. **Create Schema Detector:** Add a `Code` node named `Code - Detect Schema Changes`. Connect it to the load output of the switch. Implement type inference, DDL generation, and registry comparison.
13. **Create Schema Evaluation Node:** Add an `If` node named `Schema Changed?`. Connect it after schema detection. Evaluate `={{ $json.hasChanges }}`.
14. **Create Schema Applicator:** Add a `Postgres` node named `Postgres - Apply Schema Changes`. Connect to the true branch of `Schema Changed?`. Execute `={{ $json.ddl }}`.
15. **Create Registry Updater:** Add a `Postgres` node named `Postgres - Update Schema Registry`. Connect it after applying schema changes. Run upsert queries updating registry and change logs.
16. **Create Schema Alert Node:** Add an `HTTP Request` node named `HTTP - Slack Schema Change Alert`. Connect it after updating the registry to post change notifications to Slack.
17. **Create Upsert Builder:** Add a `Code` node named `Code - Build Upsert Statement`. Connect it from both the false branch of `Schema Changed?` and the registry updater node. Construct SQL statements utilizing `jsonb_populate_recordset`.
18. **Create Upsert Execution Node:** Add a `Postgres` node named `Postgres - Upsert Records`. Connect it after the upsert builder. Execute `={{ $json.sql }}` with query replacement parameters and enable `continueOnFail`.
19. **Create Result Checker:** Add a `Code` node named `Code - Check Load Result`. Connect it after upsert execution. Verify operation execution and format failure messages.
20. **Create Load Verification Node:** Add an `If` node named `Load Succeeded?`. Connect it after result checking. Evaluate `={{ $json.ok }}`.
21. **Create Load Failure Alert Node:** Add an `HTTP Request` node named `HTTP - Slack Load Failure Alert`. Connect it to the false branch of `Load Succeeded?` to send failure alerts.
22. **Create Watermark Advancement Node:** Add a `Postgres` node named `Postgres - Advance Watermark`. Connect it to the true branch of `Load Succeeded?`. Update `etl_state` watermarks and metrics.
23. **Create Page Summary Node:** Add a `Code` node named `Code - Page Summary`. Connect it after watermark advancement. Compute whether additional pages remain based on batch sizes and page limits.
24. **Create Loop Condition Node:** Add an `If` node named `More Pages To Fetch?`. Connect it after page summary. Evaluate `={{ $json.more }}`. Connect the true output back to `Postgres - Get Watermark` to complete the pagination loop.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Incremental ETL Pipeline with Schema-Change Detection and Auto Mapping | Primary architecture pattern managing schema drift and watermarked synchronization in n8n. |