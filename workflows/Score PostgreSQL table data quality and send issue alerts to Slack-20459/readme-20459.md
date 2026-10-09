Score PostgreSQL table data quality and send issue alerts to Slack

https://n8nworkflows.xyz/workflows/score-postgresql-table-data-quality-and-send-issue-alerts-to-slack-20459


# Score PostgreSQL table data quality and send issue alerts to Slack

### 1. Workflow Overview

This workflow functions as an automated data quality engine designed to inspect PostgreSQL tables on a scheduled hourly basis or via an on-demand webhook. It evaluates data against specific rules—checking for missing required attributes, duplicate business keys, numeric anomalies using robust statistical methods, and general table health metrics (row volume and freshness). The system scores the table across four core dimensions (completeness, uniqueness, validity, and timeliness), persists run history and tracked issues back into PostgreSQL, automatically clears resolved problems, and selectively notifies teams via Slack only when issues require attention.

The logic is organized into five functional blocks:
- **1.1 One-Time Setup:** Initializes required PostgreSQL schema tables (`dq_runs`, `dq_issues`) and provisions an optional demo order dataset with injected anomalies.
- **1.2 Triggers & Configuration:** Manages workflow execution via schedule, webhook, or manual action, defining and validating table rules and parameters.
- **1.3 Data Quality Checks:** Executes sequential PostgreSQL queries to detect missing values, duplicates, numeric outliers, and operational table health indicators.
- **1.4 Scoring, Persistence & Resolution:** Aggregates checks, calculates dimensional scores, saves audit records, identifies newly discovered issues, and auto-resolves corrected records.
- **1.5 Reporting & Alerting:** Compiles formatted quality report summaries and conditionally dispatches notifications to Slack based on severity thresholds.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: One-Time Setup

##### Overview
This block runs manually to prepare the PostgreSQL environment, establishing historical tracking tables and populating a demo table containing pre-configured data quality defects.

##### Nodes Involved
- `Manual Trigger - Run Setup Once`
- `Postgres - Create DQ Tables`

##### Node Details

- **Manual Trigger - Run Setup Once**
  - **Type and Technical Role:** `n8n-nodes-base.manualTrigger` — Initiates the workflow initialization manually.
  - **Configuration Choices:** Default setup with no parameters.
  - **Key Expressions / Variables:** None.
  - **Input / Output Connections:** Input: None | Output: `Postgres - Create DQ Tables`.
  - **Version-Specific Requirements:** Type version 1.
  - **Edge Cases / Potential Failure Types:** None.

- **Postgres - Create DQ Tables**
  - **Type and Technical Role:** `n8n-nodes-base.postgres` — Executes DDL/DML statements to construct database tables and indexes.
  - **Configuration Choices:** Operation set to `Execute Query`. Always outputs data enabled.
  - **Key Expressions / Variables:** Raw SQL creating `dq_runs`, `dq_issues`, and `dq_demo_orders` with sample rows.
  - **Input / Output Connections:** Input: `Manual Trigger - Run Setup Once` | Output: None.
  - **Version-Specific Requirements:** Type version 2.5. Requires active PostgreSQL credentials.
  - **Edge Cases / Potential Failure Types:** Database permission failures, existing conflicting schemas, or connection timeouts.

---

#### Block 1.2: Triggers & Configuration

##### Overview
This block serves as the entry point for hourly operations or HTTP requests, establishing configuration parameters and validating target identifiers before query generation.

##### Nodes Involved
- `Schedule Trigger - Hourly`
- `Webhook - Run Check On Demand`
- `Set DQ Config`
- `Code - Build Check Queries`
- `Config Valid?`
- `HTTP - Slack Config Error Alert`

##### Node Details

- **Schedule Trigger - Hourly**
  - **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` — Triggers automated execution every hour.
  - **Configuration Choices:** Rule interval set to every 1 hour.
  - **Key Expressions / Variables:** None.
  - **Input / Output Connections:** Input: None | Output: `Set DQ Config`.
  - **Version-Specific Requirements:** Type version 1.2.
  - **Edge Cases / Potential Failure Types:** None.

- **Webhook - Run Check On Demand**
  - **Type and Technical Role:** `n8n-nodes-base.webhook` — Exposes an HTTP endpoint to accept external requests.
  - **Configuration Choices:** HTTP Method `POST`, Path `dq-run`.
  - **Key Expressions / Variables:** Webhook ID `ca5dc7dc-33e9-52e7-ac78-8582c30d6641`.
  - **Input / Output Connections:** Input: None | Output: `Set DQ Config`.
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases / Potential Failure Types:** Network routing issues, invalid request payloads, or webhook path collisions.

- **Set DQ Config**
  - **Type and Technical Role:** `n8n-nodes-base.set` — Sets operational parameters and threshold rules for the data quality engine.
  - **Configuration Choices:** Assigns configuration variables including `tableName`, `pkColumn`, `timestampColumn`, `requiredColumns`, `duplicateKeyColumns`, `numericColumns`, thresholds, and `slackWebhookUrl`.
  - **Key Expressions / Variables:** Explicit static assignments for target database metrics.
  - **Input / Output Connections:** Input: `Schedule Trigger - Hourly`, `Webhook - Run Check On Demand` | Output: `Code - Build Check Queries`.
  - **Version-Specific Requirements:** Type version 3.4.
  - **Edge Cases / Potential Failure Types:** Misconfigured table names or columns resulting in downstream SQL parsing failures.

- **Code - Build Check Queries**
  - **Type and Technical Role:** `n8n-nodes-base.code` — Validates configuration input against injection rules and dynamically compiles parameterized SQL queries.
  - **Configuration Choices:** Executes JavaScript processing using regex safeguards against SQL injection (`ID` regex pattern matching).
  - **Key Expressions / Variables:** Accesses `$('Set DQ Config').first().json`.
  - **Input / Output Connections:** Input: `Set DQ Config` | Output: `Config Valid?`.
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases / Potential Failure Types:** Throws validation errors if table or column naming conventions violate allowed characters.

- **Config Valid?**
  - **Type and Technical Role:** `n8n-nodes-base.if` — Evaluates configuration validation status.
  - **Configuration Choices:** Condition checks if `$json.valid` equals `true`.
  - **Key Expressions / Variables:** `={{ $json.valid }}`
  - **Input / Output Connections:** Input: `Code - Build Check Queries` | Output (True): `Postgres - Detect Missing Values` | Output (False): `HTTP - Slack Config Error Alert`.
  - **Version-Specific Requirements:** Type version 2.2.
  - **Edge Cases / Potential Failure Types:** None.

- **HTTP - Slack Config Error Alert**
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` — Dispatches an immediate notification to Slack when configuration validation fails.
  - **Configuration Choices:** Method `POST`, JSON body sent via expression, error handling set to `Continue Regular Output`.
  - **Key Expressions / Variables:** `={{ $('Set DQ Config').first().json.slackWebhookUrl }}` and `={{ JSON.stringify({ text: $json.alertText }) }}`.
  - **Input / Output Connections:** Input: `Config Valid?` (False branch) | Output: None.
  - **Version-Specific Requirements:** Type version 4.2.
  - **Edge Cases / Potential Failure Types:** Invalid Slack webhook URLs resulting in HTTP 4xx/5xx responses.

---

#### Block 1.3: Detect Missing, Duplicate & Anomalous Records

##### Overview
This block executes targeted SQL queries against the specified PostgreSQL table to identify missing mandatory fields, duplicate business keys, robust statistical anomalies, and table health stats.

##### Nodes Involved
- `Postgres - Detect Missing Values`
- `Postgres - Detect Duplicate Records`
- `Postgres - Detect Anomalous Values`
- `Postgres - Table Health Stats`

##### Node Details

- **Postgres - Detect Missing Values**
  - **Type and Technical Role:** `n8n-nodes-base.postgres` — Executes dynamically generated SQL to locate NULL or blank values in required columns.
  - **Configuration Choices:** Operation set to `Execute Query`, `Continue On Fail` enabled, `Always Output Data` enabled, `Execute Once` enabled.
  - **KeyExpressions / Variables:** `={{ $('Code - Build Check Queries').first().json.missingSql }}`
  - **Input / Output Connections:** Input: `Config Valid?` (True branch) | Output: `Postgres - Detect Duplicate Records`.
  - **Version-Specific Requirements:** Type version 2.5. Requires PostgreSQL credentials.
  - **Edge Cases / Potential Failure Types:** Syntax errors if column definitions do not exist on the target table.

- **Postgres - Detect Duplicate Records**
  - **Type and Technical Role:** `n8n-nodes-base.postgres` — Queries the target table for duplicate business keys, optionally normalized (trimmed and lowercased).
  - **Configuration Choices:** Operation set to `Execute Query`, `Continue On Fail` enabled, `Always Output Data` enabled, `Execute Once` enabled.
  - **Key Expressions / Variables:** `={{ $('Code - Build Check Queries').first().json.duplicateSql }}`
  - **Input / Output Connections:** Input: `Postgres - Detect Missing Values` | Output: `Postgres - Detect Anomalous Values`.
  - **Version-Specific Requirements:** Type version 2.5. Requires PostgreSQL credentials.
  - **Edge Cases / Potential Failure Types:** Memory overhead on very large unindexed tables.

- **Postgres - Detect Anomalous Values**
  - **Type and Technical Role:** `n8n-nodes-base.postgres` — Calculates robust z-scores (using median and Median Absolute Deviation) to detect numeric outliers.
  - **Configuration Choices:** Operation set to `Execute Query`, `Continue On Fail` enabled, `Always Output Data` enabled, `Execute Once` enabled.
  - **Key Expressions / Variables:** `={{ $('Code - Build Check Queries').first().json.anomalySql }}`
  - **Input / Output Connections:** Input: `Postgres - Detect Duplicate Records` | Output: `Postgres - Table Health Stats`.
  - **Version-Specific Requirements:** Type version 2.5. Requires PostgreSQL credentials.
  - **Edge Cases / Potential Failure Types:** Division-by-zero errors if numeric columns contain uniform values where MAD equals zero.

- **Postgres - Table Health Stats**
  - **Type and Technical Role:** `n8n-nodes-base.postgres` — Evaluates table row counts, timestamp freshness, and volume deviations against recent historical runs.
  - **Configuration Choices:** Operation set to `Execute Query`, query replacement parameter passing table name, `Continue On Fail` enabled, `Always Output Data` enabled, `Execute Once` enabled.
  - **Key Expressions / Variables:** `={{ $('Code - Build Check Queries').first().json.healthSql }}` and query replacement `={{ [ $('Code - Build Check Queries').first().json.tableLabel ] }}`.
  - **Input / Output Connections:** Input: `Postgres - Detect Anomalous Values` | Output: `Code - Score & Classify`.
  - **Version-Specific Requirements:** Type version 2.5. Requires PostgreSQL credentials.
  - **Edge Cases / Potential Failure Types:** Missing timestamp columns returning NULL freshness metrics.

---

#### Block 1.4: Score, Persist & Resolve

##### Overview
This block aggregates all check outputs, computes dimension scores (completeness, uniqueness, validity, timeliness) and an overall health score, saves run telemetry, inserts newly identified issues, and automatically resolves cleared items.

##### Nodes Involved
- `Code - Score & Classify`
- `Postgres - Save Run & Issues`
- `Postgres - Auto-Resolve Cleared Issues`

##### Node Details

- **Code - Score & Classify**
  - **Type and Technical Role:** `n8n-nodes-base.code` — Processes raw check results, compiles dimension metrics, assigns final health classifications (`pass`, `warn`, `fail`, `error`), and structures payloads for persistence.
  - **Configuration Choices:** JavaScript data transformation processing arrays from preceding database check nodes.
  - **Key Expressions / Variables:** Reads data via `$(nodeName).all()`.
  - **Input / Output Connections:** Input: `Postgres - Table Health Stats` | Output: `Postgres - Save Run & Issues`.
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases / Potential Failure Types:** Type coercion errors if database statistics return null objects.

- **Postgres - Save Run & Issues**
  - **Type and Technical Role:** `n8n-nodes-base.postgres` — Persists aggregate run metadata into `dq_runs` and logs newly observed item-level issues into `dq_issues`.
  - **Configuration Choices:** Operation set to `Execute Query`, parameterized query replacement using JSON payload.
  - **Key Expressions / Variables:** `={{ [ $json.tableName, $json.totalRows, $json.scopedRows, $json.score, $json.status, JSON.stringify($json.dims), $json.issuesJson ] }}`.
  - **Input / Output Connections:** Input: `Code - Score & Classify` | Output: `Postgres - Auto-Resolve Cleared Issues`.
  - **Version-Specific Requirements:** Type version 2.5. Requires PostgreSQL credentials.
  - **Edge Cases / Potential Failure Types:** JSON payload size limits or database constraint violations.

- **Postgres - Auto-Resolve Cleared Issues**
  - **Type and Technical Role:** `n8n-nodes-base.postgres` — Updates open issues to `resolved` status when they no longer appear in a clean, full table scan.
  - **Configuration Choices:** Operation set to `Execute Query`, parameterized inputs passing state flags and current active key sets.
  - **Key Expressions / Variables:** `={{ [ $('Code - Score & Classify').first().json.tableName, $('Code - Score & Classify').first().json.resolveEnabled, $('Code - Score & Classify').first().json.keysJson ] }}`.
  - **Input / Output Connections:** Input: `Postgres - Save Run & Issues` | Output: `Code - Build Quality Report`.
  - **Version-Specific Requirements:** Type version 2.5. Requires PostgreSQL credentials.
  - **Edge Cases / Potential Failure Types:** Concurrency locking on `dq_issues`.

---

#### Block 1.5: Report & Alert

##### Overview
This block compiles human-readable quality report summaries and conditionally triggers Slack notifications based on run status and severity thresholds.

##### Nodes Involved
- `Code - Build Quality Report`
- `Alert Needed?`
- `HTTP - Slack Quality Alert`

##### Node Details

- **Code - Build Quality Report**
  - **Type and Technical Role:** `n8n-nodes-base.code` — Generates formatted Markdown text strings and evaluates alert criteria.
  - **Configuration Choices:** JavaScript transformation mapping status emojis, dimension scores, and top issues.
  - **Key Expressions / Variables:** References prior node JSON payloads for scores, counts, and run IDs.
  - **Input / Output Connections:** Input: `Postgres - Auto-Resolve Cleared Issues` | Output: `Alert Needed?`.
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases / Potential Failure Types:** Undefined property references if upstream steps fail silently.

- **Alert Needed?**
  - **Type and Technical Role:** `n8n-nodes-base.if` — Evaluates whether the calculated alert flag requires downstream notification dispatch.
  - **Configuration Choices:** Condition checks if `={{ $json.alertNeeded }}` is true.
  - **Key Expressions / Variables:** `={{ $json.alertNeeded }}`
  - **Input / Output Connections:** Input: `Code - Build Quality Report` | Output (True): `HTTP - Slack Quality Alert`.
  - **Version-Specific Requirements:** Type version 2.2.
  - **Edge Cases / Potential Failure Types:** None.

- **HTTP - Slack Quality Alert**
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` — Posts the formatted data quality alert payload to the designated Slack incoming webhook URL.
  - **Configuration Choices:** Method `POST`, JSON body sent via expression, timeout 15000ms, error handling set to `Continue Regular Output`.
  - **Key Expressions / Variables:** `={{ $('Set DQ Config').first().json.slackWebhookUrl }}` and `={{ JSON.stringify({ text: $json.alertText }) }}`.
  - **Input / Output Connections:** Input: `Alert Needed?` (True branch) | Output: None.
  - **Version-Specific Requirements:** Type version 4.2.
  - **Edge Cases / Potential Failure Types:** Network timeouts or revoked Slack webhook URLs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow Overview & Instructions | None | None | Data Quality Engine: Missing, Duplicate and Anomalous Records<br><br>### How it works<br><br>Scans any PostgreSQL table on a schedule (or on demand) and scores its quality.<br><br>1. **Missing:** required columns that are NULL or blank.<br>2. **Duplicates:** records that share the same business key, compared after trimming, lower-casing and collapsing spaces (so `USER5@x.com ` matches `user5@x.com`).<br>3. **Anomalies:** numeric outliers found with a robust z-score (median and MAD, so the outliers themselves do not distort the baseline). Above 2x the threshold is critical.<br>4. **Table health:** freshness (is new data arriving?) and volume (is today's load far from the recent average?).<br><br>Each run gets a 0-100 score from four dimensions (completeness, uniqueness, validity, timeliness), stored in `dq_runs`. Individual problems go to `dq_issues`, only once while they stay open, and are resolved automatically when fixed. Slack is notified only when something needs attention.<br><br>### Setup steps<br><br>- Add Postgres credentials to all Postgres nodes and run the manual setup once.<br>- Edit **Set DQ Config**: table, primary key, timestamp column, required, duplicate-key and numeric columns, thresholds, Slack webhook. The defaults check the demo table.<br>- Set `scopeHours` to e.g. 24 to check only recent rows (0 = whole table, which also enables auto-resolve).<br>- Activate the workflow. Trigger it any time with `POST /dq-run`.<br><br>### Layout<br><br>1 Setup<br>2 Triggers and configuration<br>3 Checks<br>4-5 Score, persist and alert |
| Note: 1. One-time setup | n8n-nodes-base.stickyNote | Setup Block Documentation | None | None | ## 1. One-time setup<br><br>Run manually once. Creates the run history and issue tables, plus a small demo table (dq_demo_orders) with a missing value, a duplicate and an outlier so you can see the engine work immediately. |
| Note: 2. Triggers & configuration | n8n-nodes-base.stickyNote | Trigger & Config Documentation | None | None | ## 2. Triggers & configuration<br><br>Runs every hour or on demand via `POST /dq-run`. Settings define the table, key columns, required columns, numeric columns and thresholds. The config is validated before any SQL is built. |
| Note: 3. Detect missing, duplicate & anomalous records | n8n-nodes-base.stickyNote | Check Block Documentation | None | None | ## 3. Detect missing, duplicate & anomalous records<br><br>Four SQL checks: missing (NULL or blank required columns), duplicates (normalized business key), anomalies (robust z-score on numeric columns) and table health (row counts, freshness, volume). |
| Note: 4. Score, persist & resolve | n8n-nodes-base.stickyNote | Persistence Documentation | None | None | ## 4. Score, persist & resolve<br><br>Adds freshness and volume issues, scores four dimensions (completeness, uniqueness, validity, timeliness) and saves the run. Only new issues are inserted, and issues that disappeared are resolved automatically. |
| Note: 5. Report & alert | n8n-nodes-base.stickyNote | Reporting Documentation | None | None | ## 5. Report & alert<br><br>Builds the quality report and sends a Slack alert on failures, critical issues or new warnings. Clean runs stay quiet. |
| Manual Trigger - Run Setup Once | n8n-nodes-base.manualTrigger | Manual Initialization | None | Postgres - Create DQ Tables | ## 1. One-time setup<br><br>Run manually once. Creates the run history and issue tables, plus a small demo table (dq_demo_orders) with a missing value, a duplicate and an outlier so you can see the engine work immediately. |
| Postgres - Create DQ Tables | n8n-nodes-base.postgres | Database Initialization DDL | Manual Trigger - Run Setup Once | None | ## 1. One-time setup<br><br>Run manually once. Creates the run history and issue tables, plus a small demo table (dq_demo_orders) with a missing value, a duplicate and an outlier so you can see the engine work immediately. |
| Schedule Trigger - Hourly | n8n-nodes-base.scheduleTrigger | Scheduled Execution Trigger | None | Set DQ Config | ## 2. Triggers & configuration<br><br>Runs every hour or on demand via `POST /dq-run`. Settings define the table, key columns, required columns, numeric columns and thresholds. The config is validated before any SQL is built. |
| Webhook - Run Check On Demand | n8n-nodes-base.webhook | On-Demand Webhook Trigger | None | Set DQ Config | ## 2. Triggers & configuration<br><br>Runs every hour or on demand via `POST /dq-run`. Settings define the table, key columns, required columns, numeric columns and thresholds. The config is validated before any SQL is built. |
| Set DQ Config | n8n-nodes-base.set | Parameter Configuration | Schedule Trigger - Hourly, Webhook - Run Check On Demand | Code - Build Check Queries | ## 2. Triggers & configuration<br><br>Runs every hour or on demand via `POST /dq-run`. Settings define the table, key columns, required columns, numeric columns and thresholds. The config is validated before any SQL is built. |
| Code - Build Check Queries | n8n-nodes-base.code | SQL Generation & Sanitization | Set DQ Config | Config Valid? | ## 2. Triggers & configuration<br><br>Runs every hour or on demand via `POST /dq-run`. Settings define the table, key columns, required columns, numeric columns and thresholds. The config is validated before any SQL is built. |
| Config Valid? | n8n-nodes-base.if | Configuration Validation Branch | Code - Build Check Queries | Postgres - Detect Missing Values, HTTP - Slack Config Error Alert | ## 2. Triggers & configuration<br><br>Runs every hour or on demand via `POST /dq-run`. Settings define the table, key columns, required columns, numeric columns and thresholds. The config is validated before any SQL is built. |
| HTTP - Slack Config Error Alert | n8n-nodes-base.httpRequest | Slack Error Notification | Config Valid? | None | ## 2. Triggers & configuration<br><br>Runs every hour or on demand via `POST /dq-run`. Settings define the table, key columns, required columns, numeric columns and thresholds. The config is validated before any SQL is built. |
| Postgres - Detect Missing Values | n8n-nodes-base.postgres | Missing Value Detection | Config Valid? | Postgres - Detect Duplicate Records | ## 3. Detect missing, duplicate & anomalous records<br><br>Four SQL checks: missing (NULL or blank required columns), duplicates (normalized business key), anomalies (robust z-score on numeric columns) and table health (row counts, freshness, volume). |
| Postgres - Detect Duplicate Records | n8n-nodes-base.postgres | Duplicate Key Detection | Postgres - Detect Missing Values | Postgres - Detect Anomalous Values | ## 3. Detect missing, duplicate & anomalous records<br><br>Four SQL checks: missing (NULL or blank required columns), duplicates (normalized business key), anomalies (robust z-score on numeric columns) and table health (row counts, freshness, volume). |
| Postgres - Detect Anomalous Values | n8n-nodes-base.postgres | Numeric Anomaly Detection | Postgres - Detect Duplicate Records | Postgres - Table Health Stats | ## 3. Detect missing, duplicate & anomalous records<br><br>Four SQL checks: missing (NULL or blank required columns), duplicates (normalized business key), anomalies (robust z-score on numeric columns) and table health (row counts, freshness, volume). |
| Postgres - Table Health Stats | n8n-nodes-base.postgres | Table Health Metric Gathering | Postgres - Detect Anomalous Values | Code - Score & Classify | ## 3. Detect missing, duplicate & anomalous records<br><br>Four SQL checks: missing (NULL or blank required columns), duplicates (normalized business key), anomalies (robust z-score on numeric columns) and table health (row counts, freshness, volume). |
| Code - Score & Classify | n8n-nodes-base.code | Metric Scoring & Classification | Postgres - Table Health Stats | Postgres - Save Run & Issues | ## 4. Score, persist & resolve<br><br>Adds freshness and volume issues, scores four dimensions (completeness, uniqueness, validity, timeliness) and saves the run. Only new issues are inserted, and issues that disappeared are resolved automatically. |
| Postgres - Save Run & Issues | n8n-nodes-base.postgres | Database Run Persistence | Code - Score & Classify | Postgres - Auto-Resolve Cleared Issues | ## 4. Score, persist & resolve<br><br>Adds freshness and volume issues, scores four dimensions (completeness, uniqueness, validity, timeliness) and saves the run. Only new issues are inserted, and issues that disappeared are resolved automatically. |
| Postgres - Auto-Resolve Cleared Issues | n8n-nodes-base.postgres | Automated Issue Resolution | Postgres - Save Run & Issues | Code - Build Quality Report | ## 4. Score, persist & resolve<br><br>Adds freshness and volume issues, scores four dimensions (completeness, uniqueness, validity, timeliness) and saves the run. Only new issues are inserted, and issues that disappeared are resolved automatically. |
| Code - Build Quality Report | n8n-nodes-base.code | Quality Report Markdown Generator | Postgres - Auto-Resolve Cleared Issues | Alert Needed? | ## 5. Report & alert<br><br>Builds the quality report and sends a Slack alert on failures, critical issues or new warnings. Clean runs stay quiet. |
| Alert Needed? | n8n-nodes-base.if | Alert Condition Evaluator | Code - Build Quality Report | HTTP - Slack Quality Alert | ## 5. Report & alert<br><br>Builds the quality report and sends a Slack alert on failures, critical issues or new warnings. Clean runs stay quiet. |
| HTTP - Slack Quality Alert | n8n-nodes-base.httpRequest | Slack Notification Dispatcher | Alert Needed? | None | ## 5. Report & alert<br><br>Builds the quality report and sends a Slack alert on failures, critical issues or new warnings. Clean runs stay quiet. |

---

### 4. Reproducing the Workflow from Scratch

Follow this sequential procedure to rebuild the workflow manually inside n8n:

1. **Create the One-Time Setup Nodes:**
   - Add a `Manual Trigger` node (`Manual Trigger - Run Setup Once`).
   - Add a `Postgres` node (`Postgres - Create DQ Tables`), set operation to `Execute Query`, connect credentials, and paste the DDL script creating `dq_runs`, `dq_issues`, and `dq_demo_orders`. Connect `Manual Trigger` to this node and execute it once to provision the tables.

2. **Establish Triggers & Configuration:**
   - Add a `Schedule Trigger` node (`Schedule Trigger - Hourly`) configured with an hourly interval.
   - Add a `Webhook` node (`Webhook - Run Check On Demand`) with HTTP method `POST` and path `dq-run`.
   - Add a `Set` node (`Set DQ Config`) and configure assignments for `tableName` (`dq_demo_orders`), `pkColumn` (`id`), `timestampColumn` (`created_at`), `requiredColumns` (`customer_email,amount`), `duplicateKeyColumns` (`customer_email`), `numericColumns` (`amount`), `normalizeDuplicateKeys` (`true`), `scopeHours` (`0`), `freshnessHours` (`24`), `anomalyThreshold` (`3.5`), `volumeLowRatio` (`0.5`), `volumeHighRatio` (`2`), `maxIssuesPerCheck` (`500`), `warnScore` (`95`), `failScore` (`80`), and `slackWebhookUrl`.
   - Connect both `Schedule Trigger` and `Webhook` to `Set DQ Config`.

3. **Add Query Compilation & Validation Logic:**
   - Add a `Code` node (`Code - Build Check Queries`), connect `Set DQ Config` to it, and insert the JavaScript snippet validating identifiers and constructing parameterized SQL checks for missing values, duplicates, robust z-score anomalies, and table health.
   - Add an `If` node (`Config Valid?`), connect `Code - Build Check Queries` to it, and set the condition to evaluate `$json.valid === true`.

4. **Implement Config Error Handling:**
   - Add an `HTTP Request` node (`HTTP - Slack Config Error Alert`). Connect the `false` branch of `Config Valid?` to it. Set method to `POST`, URL to `={{ $('Set DQ Config').first().json.slackWebhookUrl }}`, body format to JSON, and error handling to `Continue Regular Output`.

5. **Build the Sequential Data Quality Check Pipeline:**
   - Add four consecutive `Postgres` nodes:
     - `Postgres - Detect Missing Values` (Query: `={{ $('Code - Build Check Queries').first().json.missingSql }}`)
     - `Postgres - Detect Duplicate Records` (Query: `={{ $('Code - Build Check Queries').first().json.duplicateSql }}`)
     - `Postgres - Detect Anomalous Values` (Query: `={{ $('Code - Build Check Queries').first().json.anomalySql }}`)
     - `Postgres - Table Health Stats` (Query: `={{ $('Code - Build Check Queries').first().json.healthSql }}`)
   - Configure all four check nodes with `Continue On Fail` = true, `Always Output Data` = true, and `Execute Once` = true.
   - Chain them sequentially: connect `Config Valid?` (true branch) -> `Postgres - Detect Missing Values` -> `Postgres - Detect Duplicate Records` -> `Postgres - Detect Anomalous Values` -> `Postgres - Table Health Stats`.

6. **Implement Scoring and Persistence Logic:**
   - Add a `Code` node (`Code - Score & Classify`), connect `Postgres - Table Health Stats` to it, and implement dimension scoring, classification logic, and truncation handlers.
   - Add a `Postgres` node (`Postgres - Save Run & Issues`). Connect `Code - Score & Classify` to it. Configure execution query with parameterized query replacement inserting into `dq_runs` and `dq_issues`.
   - Add a `Postgres` node (`Postgres - Auto-Resolve Cleared Issues`). Connect `Postgres - Save Run & Issues` to it. Configure execution query to update resolved statuses where records no longer violate quality rules.

7. **Implement Reporting and Alert Dispatching:**
   - Add a `Code` node (`Code - Build Quality Report`), connect `Postgres - Auto-Resolve Cleared Issues` to it, and generate Markdown text reports and alert triggers.
   - Add an `If` node (`Alert Needed?`), connect `Code - Build Quality Report` to it, setting the condition to evaluate `={{ $json.alertNeeded }}`.
   - Add an `HTTP Request` node (`HTTP - Slack Quality Alert`). Connect the `true` branch of `Alert Needed?` to it. Set method to `POST`, URL to `={{ $('Set DQ Config').first().json.slackWebhookUrl }}`, body format to JSON, timeout to 15000ms, and error handling to `Continue Regular Output`.

8. **Credentials Setup:**
   - Ensure valid PostgreSQL database credentials are assigned across all PostgreSQL nodes.
   - Provide a valid Slack incoming webhook URL in configuration parameters.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Database Table Initialization | Manual run required once via `Manual Trigger - Run Setup Once` to establish `dq_runs`, `dq_issues`, and `dq_demo_orders`. |
| Custom Table Configuration | Update parameters in `Set DQ Config` (`tableName`, `pkColumn`, `timestampColumn`, `requiredColumns`, `duplicateKeyColumns`, `numericColumns`) to target production PostgreSQL tables. |
| Scope Window Behavior | Setting `scopeHours` to `0` enables full table scans and activates automatic issue resolution for cleared defects. |