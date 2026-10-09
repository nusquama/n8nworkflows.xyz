Sync warehouse customer data from Postgres to HubSpot and Segment

https://n8nworkflows.xyz/workflows/sync-warehouse-customer-data-from-postgres-to-hubspot-and-segment-20538


# Sync warehouse customer data from Postgres to HubSpot and Segment

### 1. Workflow Overview

This workflow implements a Reverse ETL pipeline, syncing customer data from a PostgreSQL warehouse table to external SaaS applications (HubSpot and Segment). It runs on a 10-minute schedule or on-demand via a webhook, ensures data idempotency using row hashing, maintains synchronization state, and alerts administrators via Slack on configuration or delivery errors.

The logical execution is organized into the following blocks:
- **1.1 One-Time Setup:** Initializes necessary PostgreSQL tracking tables and a demo warehouse dataset.
- **1.2 Triggers, Configuration & Run Lock:** Accepts scheduled or webhook entry points, defines sync parameters, validates the configuration, and acquires a concurrency lock to prevent overlapping executions.
- **1.3 Change Detection:** Computes and compares MD5 hashes of warehouse records against stored state to isolate new, modified, or retry-eligible rows.
- **1.4 SaaS Delivery:** Plans and executes batch API requests to HubSpot and Segment, capturing partial or total delivery failures.
- **1.5 State Tracking & Looping:** Evaluates success or failure per record, updates the state database with exponential backoff metrics, and loops through subsequent batches.
- **1.6 Lock Release & Alerting:** Frees the concurrency lock and dispatches Slack notifications when errors or dead records occur.

---

### 2. Block-by-Block Analysis

#### 2.1 One-Time Setup
- **Overview:** Initializes the database schema required for tracking sync state, execution logs, and concurrency locks, alongside an optional demo table for immediate testing.
- **Nodes Involved:**
  - `Manual Trigger - Run Setup Once`
  - `Postgres - Create Reverse ETL Tables`

##### Node Details:
- **Manual Trigger - Run Setup Once**
  - *Type & Role:* `n8n-nodes-base.manualTrigger` — Initiates the manual setup execution.
  - *Configuration:* Default.
  - *Connections:* Output connects to `Postgres - Create Reverse ETL Tables`.
  - *Edge Cases:* None.

- **Postgres - Create Reverse ETL Tables**
  - *Type & Role:* `n8n-nodes-base.postgres` — Executes SQL DDL statements to create tables (`retl_sync_state`, `retl_sync_runs`, `retl_sync_locks`, and `dw_customers_demo`) and populates the demo dataset.
  - *Configuration:* Operation set to `Execute Query`. Uses PostgreSQL credentials.
  - *Connections:* Input from Manual Trigger; no standard output dependency.
  - *Edge Cases:* Database permission issues or connection timeouts.

---

#### 2.2 Triggers, Configuration & Run Lock
- **Overview:** Ingests execution signals, establishes core synchronization parameters, validates format rules, and acquires a concurrency lock in PostgreSQL.
- **Nodes Involved:**
  - `Schedule Trigger - Every 10 Minutes`
  - `Webhook - Sync Now`
  - `Set Sync Config`
  - `Code - Validate Config & Build Queries`
  - `Config Valid?`
  - `HTTP - Slack Config Error Alert`
  - `Postgres - Acquire Sync Lock`
  - `Lock Acquired?`

##### Node Details:
- **Schedule Trigger - Every 10 Minutes**
  - *Type & Role:* `n8n-nodes-base.scheduleTrigger` — Triggers execution every 10 minutes.
  - *Configuration:* Interval set to 10 minutes.
  - *Connections:* Output connects to `Set Sync Config`.

- **Webhook - Sync Now**
  - *Type & Role:* `n8n-nodes-base.webhook` — Exposes an HTTP endpoint to trigger synchronization on demand.
  - *Configuration:* Method: `POST`, Path: `retl-sync-now`.
  - *Connections:* Output connects to `Set Sync Config`.

- **Set Sync Config**
  - *Type & Role:* `n8n-nodes-base.set` — Defines global variables including table names, primary keys, email columns, enabled destinations, JSON field mappings, and Slack endpoints.
  - *Configuration:* Custom key-value string and number assignments.
  - *Key Variables:* `syncName`, `warehouseRelation`, `pkColumn`, `emailColumn`, `enabledDestinations`, `hubspotMapping`, `segmentTraits`, `slackWebhookUrl`.
  - *Connections:* Inputs from triggers; output connects to `Code - Validate Config & Build Queries`.

- **Code - Validate Config & Build Queries**
  - *Type & Role:* `n8n-nodes-base.code` — Validates table identifiers, checks JSON schema validity for mappings, and constructs parameterized SQL detection queries.
  - *Configuration:* Custom JavaScript validation logic.
  - *Connections:* Input from `Set Sync Config`; output connects to `Config Valid?`.
  - *Edge Cases:* Malformed JSON in mapping configurations throws validation errors.

- **Config Valid?**
  - *Type & Role:* `n8n-nodes-base.if` — Branches the flow based on whether configuration validation passed.
  - *Configuration:* Checks `{{ $json.valid }}` equals `true`.
  - *Connections:* True branch to `Postgres - Acquire Sync Lock`; False branch to `HTTP - Slack Config Error Alert`.

- **HTTP - Slack Config Error Alert**
  - *Type & Role:* `n8n-nodes-base.httpRequest` — Dispatches error notifications to Slack if configuration validation fails.
  - *Configuration:* Method: `POST`, URL mapped to `slackWebhookUrl`, error handling set to continue regular output.
  - *Connections:* Input from `Config Valid?` (false).

- **Postgres - Acquire Sync Lock**
  - *Type & Role:* `n8n-nodes-base.postgres` — Attempts to insert or update an exclusive run lock for the sync name with a 15-minute expiry.
  - *Configuration:* Operation: `Execute Query`. Uses query replacement for `syncName`.
  - *Connections:* Input from `Config Valid?` (true); output connects to `Lock Acquired?`.

- **Lock Acquired?**
  - *Type & Role:* `n8n-nodes-base.if` — Verifies if the synchronization lock was successfully obtained.
  - *Configuration:* Checks `{{ $json.sync_name !== undefined }}`.
  - *Connections:* True branch to `Postgres - Detect Changed Records`.

---

#### 2.3 Change Detection
- **Overview:** Queries the database using computed hashes to isolate modified or retry-ready records while ignoring static data.
- **Nodes Involved:**
  - `Postgres - Detect Changed Records`
  - `Changes To Push?`
  - `Postgres - Release Lock (No Changes)`

##### Node Details:
- **Postgres - Detect Changed Records**
  - *Type & Role:* `n8n-nodes-base.postgres` — Executes the dynamic SQL change detection query built during validation.
  - *Configuration:* Operation: `Execute Query`. Uses query replacements for `syncName` and `batchSize`.
  - *Connections:* Input from `Lock Acquired?`; output connects to `Changes To Push?`.

- **Changes To Push?**
  - *Type & Role:* `n8n-nodes-base.if` — Determines whether any records require synchronization.
  - *Configuration:* Checks `{{ $json.record_key !== undefined }}`.
  - *Connections:* True branch to `Code - Plan Destination Requests`; False branch to `Postgres - Release Lock (No Changes)`.

- **Postgres - Release Lock (No Changes)**
  - *Type & Role:* `n8n-nodes-base.postgres` — Clears the concurrency lock when no changes are detected.
  - *Configuration:* Operation: `Execute Query`. Sets `locked_until = NULL`.
  - *Connections:* Input from `Changes To Push?` (false).

---

#### 2.4 SaaS Delivery
- **Overview:** Transforms warehouse rows into destination-specific payloads and executes batch upsert operations against HubSpot and Segment.
- **Nodes Involved:**
  - `Code - Plan Destination Requests`
  - `Loop Over Destinations`
  - `Switch - Route By Destination`
  - `HTTP - HubSpot Batch Upsert Contacts`
  - `HTTP - Segment Identify Batch`
  - `NoOp - No Deliverable Records`
  - `Code - Record Destination Result`

##### Node Details:
- **Code - Plan Destination Requests**
  - *Type & Role:* `n8n-nodes-base.code` — Validates emails, constructs property maps, and builds target API payloads for HubSpot and Segment.
  - *Connections:* Input from `Changes To Push?`; output connects to `Loop Over Destinations`.

- **Loop Over Destinations**
  - *Type & Role:* `n8n-nodes-base.splitInBatches` — Iterates through planned destination requests sequentially.
  - *Connections:* Inputs from planning and result recording nodes; output branches to `Switch - Route By Destination` or state handling.

- **Switch - Route By Destination**
  - *Type & Role:* `n8n-nodes-base.switch` — Routes execution based on the target destination type.
  - *Configuration:* Rules match `{{ $json.destination }}` against `hubspot`, `segment`, or fallback.
  - *Connections:* Outputs link to respective HTTP request nodes or the NoOp node.

- **HTTP - HubSpot Batch Upsert Contacts**
  - *Type & Role:* `n8n-nodes-base.httpRequest` — Submits batch contact upserts to the HubSpot API.
  - *Configuration:* Method: `POST`, URL: `https://api.hubapi.com/crm/v3/objects/contacts/batch/upsert`, generic HTTP Header Authentication. Error handling set to continue regular output.
  - *Connections:* Input from switch; output connects to `Code - Record Destination Result`.

- **HTTP - Segment Identify Batch**
  - *Type & Role:* `n8n-nodes-base.httpRequest` — Submits batch identify traits to the Segment API.
  - *Configuration:* Method: `POST`, URL: `https://api.segment.io/v1/batch`, generic HTTP Basic Authentication. Error handling set to continue regular output.
  - *Connections:* Input from switch; output connects to `Code - Record Destination Result`.

- **NoOp - No Deliverable Records**
  - *Type & Role:* `n8n-nodes-base.noOp` — Acts as a placeholder when no destination requests are deliverable.
  - *Connections:* Input from switch; output connects to `Code - Record Destination Result`.

- **Code - Record Destination Result**
  - *Type & Role:* `n8n-nodes-base.code` — Parses API response statuses (including HubSpot HTTP 207 multi-status codes) and compiles per-record success or failure maps.
  - *Connections:* Inputs from API/NoOp nodes; output loops back to `Loop Over Destinations`.

---

#### 2.5 State Tracking & Looping
- **Overview:** Evaluates execution results, calculates exponential backoffs for failing rows, and writes synchronization states and run metrics back to PostgreSQL.
- **Nodes Involved:**
  - `Code - Build State Updates`
  - `Postgres - Save Sync State`
  - `Code - Page Summary`
  - `More Changes To Push?`

##### Node Details:
- **Code - Build State Updates**
  - *Type & Role:* `n8n-nodes-base.code` — Aggregates destination results, determines record statuses (`synced`, `failed`, or `dead`), and calculates retry backoff delays.
  - *Connections:* Input from `Loop Over Destinations`; output connects to `Postgres - Save Sync State`.

- **Postgres - Save Sync State**
  - *Type & Role:* `n8n-nodes-base.postgres` — Upserts record sync states into `retl_sync_state` and appends run execution metrics to `retl_sync_runs`.
  - *Configuration:* Operation: `Execute Query`.
  - *Connections:* Input from state updates; output connects to `Code - Page Summary`.

- **Code - Page Summary**
  - *Type & Role:* `n8n-nodes-base.code` — Summarizes batch performance and determines if additional pagination loops or alerts are required.
  - *Connections:* Input from save state; outputs connect to `More Changes To Push?` and `Alert Needed?`.

- **More Changes To Push?**
  - *Type & Role:* `n8n-nodes-base.if` — Evaluates whether batch limits allow processing another page of warehouse data.
  - *Configuration:* Checks `{{ $json.more }}`.
  - *Connections:* True branch loops back to `Postgres - Detect Changed Records`; False branch connects to `Postgres - Release Lock (Run Complete)`.

---

#### 2.6 Lock Release & Alerting
- **Overview:** Cleans up the run concurrency lock and triggers Slack notifications if any record failures or dead states occurred during the run.
- **Nodes Involved:**
  - `Alert Needed?`
  - `Postgres - Release Lock (Run Complete)`
  - `Wait For Data`
  - `HTTP - Slack Sync Alert`

##### Node Details:
- **Alert Needed?**
  - *Type & Role:* `n8n-nodes-base.if` — Checks if error conditions require sending an alert.
  - *Configuration:* Checks `{{ $json.alertNeeded }}`.
  - *Connections:* True branch connects to `Wait For Data`.

- **Postgres - Release Lock (Run Complete)**
  - *Type & Role:* `n8n-nodes-base.postgres` — Releases the synchronization lock upon successful pipeline completion.
  - *Configuration:* Operation: `Execute Query`. Sets `locked_until = NULL`.
  - *Connections:* Input from pagination check (false).

- **Wait For Data**
  - *Type & Role:* `n8n-nodes-base.wait` — Introduces a controlled pause before dispatching alert notifications.
  - *Connections:* Input from `Alert Needed?` (true); output connects to `HTTP - Slack Sync Alert`.

- **HTTP - Slack Sync Alert**
  - *Type & Role:* `n8n-nodes-base.httpRequest` — Sends sync failure or dead-letter summaries to the configured Slack webhook URL.
  - *Configuration:* Method: `POST`, URL mapped from configuration. Error handling set to continue regular output.
  - *Connections:* Input from `Wait For Data`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation overview | None | None | ## Reverse ETL: Warehouse to SaaS Applications... |
| Note: 1. One-time setup | n8n-nodes-base.stickyNote | Visual container for setup nodes | None | None | ## 1. One-time setup... |
| Note: 2. Triggers, config & run lock | n8n-nodes-base.stickyNote | Visual container for triggers and locking | None | None | ## 2. Triggers, config & run lock... |
| Note: 3. Detect changed records | n8n-nodes-base.stickyNote | Visual container for change detection | None | None | ## 3. Detect changed records... |
| Note: 4. Deliver to SaaS destinations | n8n-nodes-base.stickyNote | Visual container for SaaS delivery | None | None | ## 4. Deliver to SaaS destinations... |
| Note: 5. Record state & loop | n8n-nodes-base.stickyNote | Visual container for state tracking | None | None | ## 5. Record state & loop... |
| Note: 6. Release lock & alert | n8n-nodes-base.stickyNote | Visual container for cleanup and alerts | None | None | ## 6. Release lock & alert... |
| Manual Trigger - Run Setup Once | n8n-nodes-base.manualTrigger | Initiates manual database setup | None | Postgres - Create Reverse ETL Tables | ## 1. One-time setup... |
| Postgres - Create Reverse ETL Tables | n8n-nodes-base.postgres | Creates synchronization and demo tables | Manual Trigger - Run Setup Once | None | ## 1. One-time setup... |
| Schedule Trigger - Every 10 Minutes | n8n-nodes-base.scheduleTrigger | Triggers sync schedule | None | Set Sync Config | ## 2. Triggers, config & run lock... |
| Webhook - Sync Now | n8n-nodes-base.webhook | Triggers sync on demand | None | Set Sync Config | ## 2. Triggers, config & run lock... |
| Set Sync Config | n8n-nodes-base.set | Assigns global sync parameters | Schedule Trigger, Webhook - Sync Now | Code - Validate Config & Build Queries | ## 2. Triggers, config & run lock... |
| Code - Validate Config & Build Queries | n8n-nodes-base.code | Validates configuration and builds SQL | Set Sync Config | Config Valid? | ## 2. Triggers, config & run lock... |
| Config Valid? | n8n-nodes-base.if | Branches on configuration validity | Code - Validate Config & Build Queries | Postgres - Acquire Sync Lock, HTTP - Slack Config Error Alert | ## 2. Triggers, config & run lock... |
| HTTP - Slack Config Error Alert | n8n-nodes-base.httpRequest | Alerts Slack on invalid config | Config Valid? | None | ## 2. Triggers, config & run lock... |
| Postgres - Acquire Sync Lock | n8n-nodes-base.postgres | Acquires execution concurrency lock | Config Valid? | Lock Acquired? | ## 2. Triggers, config & run lock... |
| Lock Acquired? | n8n-nodes-base.if | Verifies concurrency lock acquisition | Postgres - Acquire Sync Lock | Postgres - Detect Changed Records | ## 2. Triggers, config & run lock... |
| Postgres - Detect Changed Records | n8n-nodes-base.postgres | Queries changed warehouse records | Lock Acquired?, More Changes To Push? | Changes To Push? | ## 3. Detect changed records... |
| Changes To Push? | n8n-nodes-base.if | Checks if records require processing | Postgres - Detect Changed Records | Code - Plan Destination Requests, Postgres - Release Lock (No Changes) | ## 3. Detect changed records... |
| Code - Plan Destination Requests | n8n-nodes-base.code | Plans destination batch payloads | Changes To Push? | Loop Over Destinations | ## 4. Deliver to SaaS destinations... |
| Postgres - Release Lock (No Changes) | n8n-nodes-base.postgres | Releases lock when no changes exist | Changes To Push? | None | ## 3. Detect changed records... |
| Loop Over Destinations | n8n-nodes-base.splitInBatches | Iterates over destination tasks | Code - Plan Destination Requests, Code - Record Destination Result | Switch - Route By Destination, Code - Build State Updates | ## 4. Deliver to SaaS destinations... |
| Switch - Route By Destination | n8n-nodes-base.switch | Routes tasks by destination type | Loop Over Destinations | HTTP - HubSpot, HTTP - Segment, NoOp | ## 4. Deliver to SaaS destinations... |
| HTTP - HubSpot Batch Upsert Contacts | n8n-nodes-base.httpRequest | Upserts contacts to HubSpot | Switch - Route By Destination | Code - Record Destination Result | ## 4. Deliver to SaaS destinations... |
| HTTP - Segment Identify Batch | n8n-nodes-base.httpRequest | Sends identify batch to Segment | Switch - Route By Destination | Code - Record Destination Result | ## 4. Deliver to SaaS destinations... |
| NoOp - No Deliverable Records | n8n-nodes-base.noOp | Handles empty destination tasks | Switch - Route By Destination | Code - Record Destination Result | ## 4. Deliver to SaaS destinations... |
| Code - Record Destination Result | n8n-nodes-base.code | Parses destination API results | HTTP - HubSpot, HTTP - Segment, NoOp | Loop Over Destinations | ## 4. Deliver to SaaS destinations... |
| Code - Build State Updates | n8n-nodes-base.code | Builds state updates and backoffs | Loop Over Destinations | Postgres - Save Sync State | ## 5. Record state & loop... |
| Postgres - Save Sync State | n8n-nodes-base.postgres | Saves state updates and run metrics | Code - Build State Updates | Code - Page Summary | ## 5. Record state & loop... |
| Code - Page Summary | n8n-nodes-base.code | Evaluates pagination status | Postgres - Save Sync State | More Changes To Push?, Alert Needed? | ## 5. Record state & loop... |
| More Changes To Push? | n8n-nodes-base.if | Evaluates pagination continuation | Code - Page Summary | Postgres - Detect Changed Records, Postgres - Release Lock (Run Complete) | ## 5. Record state & loop... |
| Alert Needed? | n8n-nodes-base.if | Checks if error alert is required | Code - Page Summary | Wait For Data | ## 6. Release lock & alert... |
| Postgres - Release Lock (Run Complete) | n8n-nodes-base.postgres | Releases lock upon completion | More Changes To Push? | None | ## 6. Release lock & alert... |
| HTTP - Slack Sync Alert | n8n-nodes-base.httpRequest | Sends operational alerts to Slack | Wait For Data | None | ## 6. Release lock & alert... |
| Wait For Data | n8n-nodes-base.wait | Delays before sending alerts | Alert Needed? | HTTP - Slack Sync Alert | ## 6. Release lock & alert... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initial Setup Nodes
1. Create a **Manual Trigger** node named `Manual Trigger - Run Setup Once`.
2. Create a **PostgreSQL** node named `Postgres - Create Reverse ETL Tables`. Set operation to `Execute Query` and provide the SQL script to create `retl_sync_state`, `retl_sync_runs`, `retl_sync_locks`, and `dw_customers_demo` tables along with test inserts. Connect the Manual Trigger output to this node.

#### Step 2: Triggers, Configuration & Locking
3. Create a **Schedule Trigger** (`Schedule Trigger - Every 10 Minutes`) set to run every 10 minutes.
4. Create a **Webhook** node (`Webhook - Sync Now`) with HTTP method `POST` and path `retl-sync-now`.
5. Create a **Set** node (`Set Sync Config`) to define variables: `syncName` (`customers_to_saas`), `warehouseRelation` (`dw_customers_demo`), `pkColumn` (`id`), `emailColumn` (`email`), `enabledDestinations` (`hubspot,segment`), and mapping payloads. Connect both triggers to this node.
6. Create a **Code** node (`Code - Validate Config & Build Queries`) to validate configurations and construct detection queries. Connect `Set Sync Config` to it.
7. Create an **If** node (`Config Valid?`) checking `{{ $json.valid === true }}`. Connect the validation code node to it.
8. Create an **HTTP Request** node (`HTTP - Slack Config Error Alert`) using POST to send configuration errors to your Slack webhook URL. Connect the false branch of `Config Valid?` here.
9. Create a **PostgreSQL** node (`Postgres - Acquire Sync Lock`) executing an upsert on `retl_sync_locks` with a 15-minute expiry. Connect the true branch of `Config Valid?` here.
10. Create an **If** node (`Lock Acquired?`) checking `{{ $json.sync_name !== undefined }}`. Connect the acquire lock node output here.

#### Step 3: Change Detection Nodes
11. Create a **PostgreSQL** node (`Postgres - Detect Changed Records`) executing the query generated in validation. Connect the true branch of `Lock Acquired?` here.
12. Create an **If** node (`Changes To Push?`) checking `{{ $json.record_key !== undefined }}`. Connect the detect records node output here.
13. Create a **PostgreSQL** node (`Postgres - Release Lock (No Changes)`) setting `locked_until = NULL`. Connect the false branch of `Changes To Push?` here.

#### Step 4: SaaS Delivery Nodes
14. Create a **Code** node (`Code - Plan Destination Requests`) to parse records and build HubSpot/Segment payloads. Connect the true branch of `Changes To Push?` to it.
15. Create a **Split In Batches** node (`Loop Over Destinations`). Connect the planning code node to it.
16. Create a **Switch** node (`Switch - Route By Destination`) routing based on `{{ $json.destination }}` (`hubspot`, `segment`, or fallback). Connect the batch node to it.
17. Create an **HTTP Request** node (`HTTP - HubSpot Batch Upsert Contacts`) configured for `POST` to `https://api.hubapi.com/crm/v3/objects/contacts/batch/upsert` using Header Auth (`Authorization: Bearer <token>`). Enable `Never Error` in response options. Connect the HubSpot route here.
18. Create an **HTTP Request** node (`HTTP - Segment Identify Batch`) configured for `POST` to `https://api.segment.io/v1/batch` using Basic Auth (Write Key as username, empty password). Enable `Never Error`. Connect the Segment route here.
19. Create a **NoOp** node (`NoOp - No Deliverable Records`) for empty requests. Connect the fallback route here.
20. Create a **Code** node (`Code - Record Destination Result`) to parse API responses and multi-status codes. Connect all three destination handlers to this node, and loop its output back into `Loop Over Destinations`.

#### Step 5: State Tracking & Looping Nodes
21. Create a **Code** node (`Code - Build State Updates`) to calculate retry metrics and backoff delays. Connect the secondary loop output of `Loop Over Destinations` to it.
22. Create a **PostgreSQL** node (`Postgres - Save Sync State`) to execute state upserts and log run metrics. Connect the build state updates node to it.
23. Create a **Code** node (`Code - Page Summary`) to evaluate pagination thresholds and build alert texts. Connect the save state node to it.
24. Create an **If** node (`More Changes To Push?`) checking `{{ $json.more }}`. Connect the page summary node to it. Connect its true branch back to `Postgres - Detect Changed Records`.

#### Step 6: Lock Release & Alerting Nodes
25. Create a **PostgreSQL** node (`Postgres - Release Lock (Run Complete)`) setting `locked_until = NULL`. Connect the false branch of `More Changes To Push?` here.
26. Create an **If** node (`Alert Needed?`) checking `{{ $json.alertNeeded }}`. Connect the page summary node to it.
27. Create a **Wait** node (`Wait For Data`). Connect the true branch of `Alert Needed?` here.
28. Create an **HTTP Request** node (`HTTP - Slack Sync Alert`) configured for `POST` to your Slack webhook URL. Connect the wait node output here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Reverse ETL pipeline syncing PostgreSQL warehouse data to HubSpot and Segment with idempotency, concurrency locking, and backoff retry logic. | Workflow architecture overview |