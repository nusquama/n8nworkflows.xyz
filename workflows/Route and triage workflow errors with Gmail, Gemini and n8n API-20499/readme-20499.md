Route and triage workflow errors with Gmail, Gemini and n8n API

https://n8nworkflows.xyz/workflows/route-and-triage-workflow-errors-with-gmail--gemini-and-n8n-api-20499


# Route and triage workflow errors with Gmail, Gemini and n8n API

### 1. Workflow Overview

The **Golden Error Handler** workflow functions as a centralized, enterprise-grade error interception and incident management system for n8n instances. Its primary purpose is to capture unhandled execution failures across multiple monitored workflows, evaluate their recurrence and severity, generate AI-powered diagnostic insights using Google Gemini, deliver formatted multi-channel notifications (Gmail, Slack, Telegram, Discord, Microsoft Teams, and webhooks), and supply an interactive browser-based administration dashboard.

The operational logic is organized into the following functional blocks:
- **1.1 Setup and Storage Initialization:** Provisions the core n8n Data Tables (`geh_config`, `geh_fingerprints`, `geh_incidents`) and seeds default settings upon first execution.
- **1.2 Settings Web Interface & API:** Serves the interactive configuration dashboard via webhooks (`/geh-config`), handles form validation, and updates persistent settings.
- **1.3 Workflow Connection Manager:** Lists workflows using the n8n API, enabling operators to bulk-assign or disconnect the error handler via webhooks (`/geh-assign`, `/geh-workflows`, `/geh-disconnect`).
- **1.4 Error Ingestion, Guards, & Categorization:** Intercepts unhandled errors via the Error Trigger or simulation webhooks, validates payloads, normalizes execution metrics, calculates error hashes (fingerprints), and categorizes failures (auth, rate limits, timeouts, syntax errors, or exceptions).
- **1.5 Severity, Pacing, & AI Diagnostics:** Evaluates recurrence counts over a 24-hour window, enforces exponential backoff pacing to suppress alert floods, checks mute status, and queries Google Gemini for root-cause analysis and recovery recommendations.
- **1.6 Multi-Channel Alert Dispatch & Actions:** Formats rich HTML/Markdown payloads and routes notifications to Gmail and other communication channels, while exposing webhooks for one-click execution retries (`/geh-retry`), alert muting (`/geh-mute`), and unmuting (`/geh-unmute`).
- **1.7 Observability Dashboards & Retention:** Serves live telemetry pages (`/geh-activity`, `/geh-guide`, and JSON endpoints) and runs a scheduled daily cron job to purge expired incidents based on retention settings.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Storage Initialization
- **Overview:** Initializes the database tables required for system configuration, error fingerprinting, and incident logging, seeding default parameters if none exist.
- **Nodes Involved:** `Run Setup Once`, `Create Config Table`, `Create Fingerprints`, `Create Incidents`, `Read Existing Config`, `Check Config Seed`, `Seed Default Config`.
- **Node Details:**
  - **Run Setup Once** (`n8n-nodes-base.manualTrigger`)
    - *Role:* Entry point to manually initialize the database schema.
    - *Config:* Standard manual execution trigger.
    - *Connections:* Input: None | Output: `Create Config Table`.
    - *Failure Modes:* None.
  - **Create Config Table** (`n8n-nodes-base.dataTable`)
    - *Role:* Creates the `geh_config` Data Table if it does not exist.
    - *Config:* Resource: Table, Operation: Create, Table Name: `geh_config`.
    - *Connections:* Input: `Run Setup Once` | Output: `Create Fingerprints`.
    - *Failure Modes:* Database access error or table collision.
  - **Create Fingerprints** (`n8n-nodes-base.dataTable`)
    - *Role:* Creates the `geh_fingerprints` Data Table for deduplication and mute states.
    - *Config:* Resource: Table, Operation: Create, Table Name: `geh_fingerprints`.
    - *Connections:* Input: `Create Config Table` | Output: `Create Incidents`.
    - *Failure Modes:* Database error.
  - **Create Incidents** (`n8n-nodes-base.dataTable`)
    - *Role:* Creates the `geh_incidents` Data Table for logging execution audit trails.
    - *Config:* Resource: Table, Operation: Create, Table Name: `geh_incidents`.
    - *Connections:* Input: `Create Fingerprints` | Output: `Read Existing Config`.
    - *Failure Modes:* Database error.
  - **Read Existing Config** (`n8n-nodes-base.dataTable`)
    - *Role:* Retrieves the main configuration row.
    - *Config:* Operation: Get, Filters: `config_key` = `main`.
    - *Connections:* Input: `Create Incidents` | Output: `Check Config Seed`.
    - *Failure Modes:* Table missing or read failure.
  - **Check Config Seed** (`n8n-nodes-base.if`)
    - *Role:* Verifies whether the configuration record exists.
    - *Config:* Condition evaluates if `config_key` is empty (`{{ !$json.config_key }}`).
    - *Connections:* Input: `Read Existing Config` | Output: True -> `Seed Default Config`, False -> None.
    - *Failure Modes:* Evaluation error.
  - **Seed Default Config** (`n8n-nodes-base.dataTable`)
    - *Role:* Populates the initial configuration row in `geh_config`.
    - *Config:* Operation: Upsert, Table Name: `geh_config`, Key: `config_key` = `main`.
    - *Connections:* Input: `Check Config Seed` | Output: None.
    - *Failure Modes:* Database upsert error.

#### 2.2 Settings Web Interface & API
- **Overview:** Renders an interactive HTML configuration form via webhooks, parses incoming POST submissions, validates parameter constraints, and persists updates.
- **Nodes Involved:** `Config Page Webhook`, `Get Config For Form`, `Config HTML Chunks`, `Prepare Config Fields`, `Build Duration Hints`, `Build Config Form`, `Serve Config Form`, `Save Config Webhook`, `Check Config Body`, `Save Config Row`, `Redirect To Assign`, `Build Config Error`, `Config Save Error`.
- **Node Details:**
  - **Config Page Webhook** (`n8n-nodes-base.webhook`)
    - *Role:* Exposes the public endpoint (`/geh-config`) for the settings dashboard.
    - *Config:* Path: `geh-config`, Response Mode: Response Node.
    - *Connections:* Input: None | Output: `Get Config For Form`.
    - *Failure Modes:* Unauthenticated public access if security is omitted.
  - **Get Config For Form** (`n8n-nodes-base.dataTable`)
    - *Role:* Fetches current settings to populate the HTML form.
    - *Config:* Operation: Get, Table: `geh_config`, Filter: `config_key` = `main`.
    - *Connections:* Input: `Config Page Webhook` | Output: `Config HTML Chunks`.
    - *Failure Modes:* Database error.
  - **Config HTML Chunks** (`n8n-nodes-base.set`)
    - *Role:* Assembles HTML markup templates and chunks for the settings UI.
    - *Config:* Sets static and templated HTML string assignments (`h1` through `h16`).
    - *Connections:* Input: `Get Config For Form` | Output: `Prepare Config Fields`.
    - *Failure Modes:* Expression evaluation failure.
  - **Prepare Config Fields** (`n8n-nodes-base.set`)
    - *Role:* Sanitizes and parses duration values and user inputs (escaping HTML entities).
    - *Config:* Sets variables like `safe_email_to`, `safe_base_url`, and backoff duration components.
    - *Connections:* Input: `Config HTML Chunks` | Output: `Build Duration Hints`.
    - *Failure Modes:* Type conversion errors.
  - **Build Duration Hints** (`n8n-nodes-base.set`)
    - *Role:* Computes human-readable duration strings (days, hours, minutes) for backoff fields.
    - *Config:* Evaluates `backoff_start` and `backoff_max` against 60-minute thresholds.
    - *Connections:* Input: `Prepare Config Fields` | Output: `Build Config Form`.
    - *Failure Modes:* Expression failure.
  - **Build Config Form** (`n8n-nodes-base.set`)
    - *Role:* Merges HTML chunks and sanitized field values into a complete HTML document.
    - *Config:* Concatenates variables `p1` through `p5b`.
    - *Connections:* Input: `Build Duration Hints` | Output: `Serve Config Form`.
    - *Failure Modes:* String concatenation error.
  - **Serve Config Form** (`n8n-nodes-base.respondToWebhook`)
    - *Role:* Returns the settings page HTML to the browser with `Content-Type: text/html`.
    - *Config:* Response Content: Text, Header: `text/html`.
    - *Connections:* Input: `Build Config Form` | Output: None.
    - *Failure Modes:* Network timeout or serialization failure.
  - **Save Config Webhook** (`n8n-nodes-base.webhook`)
    - *Role:* Receives submitted form data via HTTP POST at `/geh-config-save`.
    - *Config:* Method: POST, Path: `geh-config-save`.
    - *Connections:* Input: None | Output: `Check Config Body`.
    - *Failure Modes:* Malformed payload body.
  - **Check Config Body** (`n8n-nodes-base.if`)
    - *Role:* Validates numerical ranges (backoff, thresholds, retention days) and URL formats.
    - *Config:* Multiple rule conditions validating integers and URL regex patterns.
    - *Connections:* Input: `Save Config Webhook` | Output: True -> `Save Config Row`, False -> `Build Config Error`.
    - *Failure Modes:* Evaluation error.
  - **Save Config Row** (`n8n-nodes-base.dataTable`)
    - *Role:* Persists validated configuration changes to the `geh_config` table.
    - *Config:* Operation: Upsert, Table: `geh_config`, Key: `main`.
    - *Connections:* Input: `Check Config Body` | Output: `Redirect To Assign`.
    - *Failure Modes:* Database write error.
  - **Redirect To Assign** (`n8n-nodes-base.respondToWebhook`)
    - *Role:* Serves an HTML redirect page pointing to `/geh-assign`.
    - *Config:* Response HTML with a meta refresh tag.
    - *Connections:* Input: `Save Config Row` | Output: None.
    - *Failure Modes:* None.
  - **Build Config Error** (`n8n-nodes-base.set`)
    - *Role:* Compiles descriptive error validation messages if configuration input is invalid.
    - *Config:* Sets conditional validation error strings (`seg1` through `seg9`).
    - *Connections:* Input: `Check Config Body` | Output: `Config Save Error`.
    - *Failure Modes:* Expression failure.
  - **Config Save Error** (`n8n-nodes-base.respondToWebhook`)
    - *Role:* Returns an HTML page displaying validation errors and a link back to settings.
    - *Config:* Response Content: Text (`text/html`).
    - *Connections:* Input: `Build Config Error` | Output: None.
    - *Failure Modes:* None.

#### 2.3 Workflow Connection Manager
- **Overview:** Interacts with the n8n API to list registered workflows, rendering a management interface for bulk assignment and disconnection.
- **Nodes Involved:** `Assign Page Webhook`, `Get All Workflows`, `Format Workflow Item`, `Build Workflow Rows`, `Collect Rows`, `Assemble Assign Page`, `Serve Assign Page`, `Workflow List Webhook`, `Get Wfs For API`, `Collect Wfs For API`, `Format Wfs For API`, `Serve Workflow List`, `Save Assign Webhook`, `Parse Workflow IDs`, `Check Assign Empty`, `Split Workflow IDs`, `Get Workflow Record`, `Set Error Handler`, `Update N8n Workflow`, `Assign Confirmed`, `Assign Empty`, `Assign Denied`, `Disconnect Webhook`, `Parse Disconnect IDs`, `Check Disconnect ID`, `Get Disconnect Record`, `Check Disconnect Wf`, `Clear Error Handler`, `Update Disconnect Wf`, `Disconnect Confirmed`, `Disconnect Denied`, `Disconnect Empty`, `Response/Error Nodes`.
- **Node Details:**
  - **Assign Page Webhook** (`n8n-nodes-base.webhook`)
    - *Role:* Exposes the UI endpoint (`/geh-assign`) for managing workflow bindings.
    - *Config:* Path: `geh-assign`.
    - *Connections:* Input: None | Output: `Get All Workflows`.
    - *Failure Modes:* Unauthorized access.
  - **Get All Workflows** & **Get Wfs For API** (`n8n-nodes-base.n8n`)
    - *Role:* Queries the n8n API to retrieve all workflows instance-wide.
    - *Config:* Limit: 10,000, Exclude Pinned Data: True. Uses n8n API credentials (`n8n.workflows.ac`).
    - *Connections:* Input: `Assign Page Webhook` / `Workflow List Webhook` | Output: `Format Workflow Item` / `Collect Wfs For API`.
    - *Failure Modes:* Invalid API credentials or timeout.
  - **Format Workflow Item** (`n8n-nodes-base.set`)
    - *Role:* Sanitizes workflow names and generates status badges (active vs. inactive, existing error handler bindings).
    - *Config:* Sets `safe_name`, `avail_badge`, and `btn_disc`.
    - *Connections:* Input: `Get All Workflows` | Output: `Build Workflow Rows`.
    - *Failure Modes:* Null reference errors on workflow settings.
  - **Build Workflow Rows** (`n8n-nodes-base.set`)
    - *Role:* Categorizes workflows into connected lists and available selection lists.
    - *Config:* Builds HTML list item strings (`conn_row`, `avail_row`).
    - *Connections:* Input: `Format Workflow Item` | Output: `Collect Rows`.
    - *Failure Modes:* Expression error.
  - **Collect Rows** & **Assemble Assign Page** (`n8n-nodes-base.aggregate` / `n8n-nodes-base.set`)
    - *Role:* Aggregates rows and compiles the complete Connect management page HTML.
    - *Config:* Destination field: `all`, generates empty states and capability warning notes.
    - *Connections:* Input: `Build Workflow Rows` -> `Collect Rows` -> `Assemble Assign Page` -> `Serve Assign Page`.
    - *Failure Modes:* Aggregation failure.
  - **Save Assign Webhook** & **Parse Workflow IDs** (`n8n-nodes-base.webhook` / `n8n-nodes-base.set`)
    - *Role:* Receives POST submissions from `/geh-assign-save` and extracts target workflow IDs.
    - *Config:* Path: `geh-assign-save`, extracts array of workflow IDs from form body.
    - *Connections:* Input: None | Output: `Check Assign Empty`.
    - *Failure Modes:* Empty payload.
  - **Check Assign Empty** (`n8n-nodes-base.if`)
    - *Role:* Ensures at least one workflow ID was selected.
    - *Config:* Evaluates `{{ ($json.workflow_ids || []).length > 0 }}`.
    - *Connections:* Input: `Parse Workflow IDs` | Output: True -> `Split Workflow IDs`, False -> `Assign Empty`.
    - *Failure Modes:* None.
  - **Split Workflow IDs** (`n8n-nodes-base.splitOut`)
    - *Role:* Splits the array of workflow IDs into individual items for batch processing.
    - *Config:* Field to split out: `workflow_ids`.
    - *Connections:* Input: `Check Assign Empty` | Output: `Get Workflow Record`.
    - *Failure Modes:* Non-array input.
  - **Get Workflow Record** (`n8n-nodes-base.n8n`)
    - *Role:* Fetches the full JSON object for a specific workflow via the n8n API.
    - *Config:* Operation: Get, Workflow ID: `{{ $json.workflow_ids }}`, On Error: Continue error output.
    - *Connections:* Input: `Split Workflow IDs` | Output: Success -> `Set Error Handler`, Error -> `Assign Denied`.
    - *Failure Modes:* Workflow deleted or API failure.
  - **Set Error Handler** & **Update N8n Workflow** (`n8n-nodes-base.set` / `n8n-nodes-base.n8n`)
    - *Role:* Injects this workflow's ID into the target workflow's `settings.errorWorkflow` property and updates it via API.
    - *Config:* Operation: Update, assigns `settings.errorWorkflow = $workflow.id`.
    - *Connections:* Input: `Get Workflow Record` -> `Set Error Handler` -> `Update N8n Workflow` -> `Assign Confirmed`.
    - *Failure Modes:* API permission restriction or concurrency lock.
  - **Disconnect Webhook** Group (`Disconnect Webhook`, `Parse Disconnect IDs`, `Check Disconnect ID`, `Get Disconnect Record`, `Check Disconnect Wf`, `Clear Error Handler`, `Update Disconnect Wf`, `Disconnect Confirmed`)
    - *Role:* Receives POST requests at `/geh-disconnect`, verifies ownership, clears the `errorWorkflow` setting on the target workflow, and updates it via API.
    - *Config:* Path: `geh-disconnect`, operation: Update with empty `errorWorkflow`.
    - *Connections:* Input: Webhook | Output: Success/Failure HTML responses.
    - *Failure Modes:* Invalid workflow ID or API connection failure.

#### 2.4 Error Ingestion, Guards, & Categorization
- **Overview:** Intercepts runtime failures from connected workflows, validates payload integrity, prevents self-referential error loops, computes cryptographic fingerprints, and categorizes errors.
- **Nodes Involved:** `Error Trigger`, `Simulate Webhook`, `Build Sim Payload`, `Serve Simulate HTML`, `Ingest Error Payload`, `Guard Payload`, `Fail Loud Guard`, `Read Config`, `Filter Manual Runs`, `Format Error Data`, `Hash Fingerprint`, `Lookup Fingerprint`, `Check If Muted`, `Log Incident Muted`, `Update Muted Seen`, `Calculate Counts`, `Categorize Error`, `Category Auth`, `Category Rate Limit`, `Category Timeout`, `Category Payload`, `Category Other`.
- **Node Details:**
  - **Error Trigger** (`n8n-nodes-base.errorTrigger`)
    - *Role:* Automatic entry point triggered whenever any attached monitored workflow fails.
    - *Config:* Standard n8n error trigger.
    - *Connections:* Input: None | Output: `Ingest Error Payload`.
    - *Failure Modes:* None.
  - **Simulate Webhook** & **Build Sim Payload** (`n8n-nodes-base.webhook` / `n8n-nodes-base.set`)
    - *Role:* Generates synthetic error payloads for testing via `/geh-simulate-error`.
    - *Config:* Path: `geh-simulate-error`, scenario parameters (rate limit, auth, timeout, payload, exception).
    - *Connections:* Input: None -> `Build Sim Payload` -> `Serve Simulate HTML` -> `Ingest Error Payload`.
    - *Failure Modes:* Malformed query parameters.
  - **Ingest Error Payload** (`n8n-nodes-base.set`)
    - *Role:* Normalizes incoming error attributes from either real execution failures or simulations.
    - *Config:* Maps raw workflow ID, execution ID, error message, stack trace, and mode (`production`/`manual`).
    - *Connections:* Input: `Error Trigger` or `Serve Simulate HTML` | Output: `Guard Payload`.
    - *Failure Modes:* Missing payload properties.
  - **Guard Payload** (`n8n-nodes-base.if`)
    - *Role:* Validates that essential execution identifiers are present and prevents loops.
    - *Config:* Checks that `raw_ex_id`, `raw_wf_id`, and `raw_err_msg` are not empty.
    - *Connections:* Input: `Ingest Error Payload` | Output: True -> `Read Config`, False -> `Fail Loud Guard`.
    - *Failure Modes:* Evaluation error.
  - **Fail Loud Guard** (`n8n-nodes-base.stopAndError`)
    - *Role:* Halts execution and logs a critical error if the ingested error payload is malformed.
    - *Config:* Custom error message regarding missing error trigger data.
    - *Connections:* Input: `Guard Payload` (False) | Output: None.
    - *Failure Modes:* Halts flow intentionally.
  - **Filter Manual Runs** (`n8n-nodes-base.if`)
    - *Role:* Filters out manual test executions unless `include_manual_runs` is enabled in configuration.
    - *Config:* Evaluates `raw_mode !== 'manual' || include_manual_runs`.
    - *Connections:* Input: `Read Config` | Output: True -> `Format Error Data`, False -> None.
    - *Failure Modes:* Evaluation error.
  - **Format Error Data** (`n8n-nodes-base.set`)
    - *Role:* Cleans workflow names, extracts configuration flags, and constructs the hash input string.
    - *Config:* Sets `hash_input` as `workflow_id|node_name|error_message`.
    - *Connections:* Input: `Filter Manual Runs` | Output: `Hash Fingerprint`.
    - *Failure Modes:* Null reference errors.
  - **Hash Fingerprint** (`n8n-nodes-base.crypto`)
    - *Role:* Generates a stable SHA-256 cryptographic fingerprint for deduplication.
    - *Config:* Operation: SHA-256, Source: `hash_input`, Output Property: `fingerprint`.
    - *Connections:* Input: `Format Error Data` | Output: `Lookup Fingerprint`.
    - *Failure Modes:* Crypto operation failure.
  - **Lookup Fingerprint** & **Check If Muted** (`n8n-nodes-base.dataTable` / `n8n-nodes-base.if`)
    - *Role:* Queries `geh_fingerprints` to determine if the error signature is currently muted.
    - *Config:* Operation: Get by `fingerprint`. Checks if `muted === true` and `mute_until` is in the future.
    - *Connections:* Input: `Hash Fingerprint` -> `Lookup Fingerprint` -> `Check If Muted`.
    - *Output Routing:*
      - **True (Muted):** `Log Incident Muted` -> `Update Muted Seen` (halts further alerting).
      - **False (Active):** `Calculate Counts`.
    - *Failure Modes:* Database read error.
  - **Categorize Error** (`n8n-nodes-base.switch`)
    - *Role:* Classifies error messages into functional categories based on keywords.
    - *Config:* Rules checking for:
      - `auth` (401, 403, unauthorized, forbidden)
      - `rate_limit` (429, rate limit)
      - `timeout` (timeout, ETIMEDOUT, ECONNRESET)
      - `payload` (JSON, parse, syntax)
      - Fallback: `exception` / `other`.
    - *Connections:* Input: `Calculate Counts` | Output: Routes to respective `Category *` set nodes.
    - *Failure Modes:* Regex/matching failure.

#### 2.5 Severity, Pacing, & AI Diagnostics
- **Overview:** Computes dynamic severity levels based on 24-hour recurrence counts, enforces exponential backoff pacing, and queries Google Gemini for diagnostic analysis.
- **Nodes Involved:** `Calculate Severity`, `Check Backoff`, `Update Suppressed`, `Log Suppressed`, `Check AI Enabled`, `Read Retry History`, `Collect Retry History`, `Check History Read`, `Count Retry Outcomes`, `Build Retry Context`, `Build AI Prompt`, `Generate AI Summary`, `Gemini Model`, `AI Output Parser`, `Parse AI Tokens`, `Derive AI Verdict`, `Format AI Output`, `Escape Alert Fields`, `Alert HTML Chunks`, `Build History Fallback`, `Skip AI Summary`.
- **Node Details:**
  - **Calculate Severity** (`n8n-nodes-base.set`)
    - *Role:* Determines incident severity (low, medium, high) by comparing 24-hour occurrence counts against configured thresholds (`sev_medium_count`, `sev_high_count`).
    - *Config:* Computes exponential backoff delay milliseconds using `backoff_start * 2^(attempt_count - 1)`.
    - *Connections:* Input: Category nodes | Output: `Check Backoff`.
    - *Failure Modes:* Arithmetic evaluation errors.
  - **Check Backoff** (`n8n-nodes-base.if`)
    - *Role:* Suppresses alerts if an error fingerprint has alerted recently within its backoff window or hit max alerts.
    - *Config:* Evaluates `elapsed_ms < delay_ms || is_capped`.
    - *Connections:* Input: `Calculate Severity` | Output: True -> `Update Suppressed` & `Log Suppressed`, False -> `Check AI Enabled`.
    - *Failure Modes:* Time evaluation errors.
  - **Check AI Enabled** (`n8n-nodes-base.if`)
    - *Role:* Checks whether AI diagnostics are enabled in configuration.
    - *Config:* Evaluates `config_ai_enabled`.
    - *Connections:* Input: `Check Backoff` (False) | Output: True -> `Read Retry History`, False -> `Skip AI Summary`.
    - *Failure Modes:* None.
  - **Read Retry History** & **Collect Retry History** (`n8n-nodes-base.dataTable` / `n8n-nodes-base.aggregate`)
    - *Role:* Retrieves recent incident records for the fingerprint to provide context on past retry success or failure rates.
    - *Config:* Limit: 20 records, grouped by status.
    - *Connections:* Input: `Check AI Enabled` -> `Read Retry History` -> `Collect Retry History` -> `Check History Read` -> `Count Retry Outcomes`.
    - *Failure Modes:* Database timeout or missing table rows.
  - **Generate AI Summary** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Role:* Invokes Google Gemini to analyze the failure context, stack trace, and retry history.
    - *Config:* Uses structured output parsing to return JSON containing `likely_cause`, `next_step`, `retry_verdict`, and `severity`.
    - *Credentials:* Google PaLM API (`n8n-server`).
    - *Connections:* Input: `Build AI Prompt` & `Gemini Model` + `AI Output Parser` | Output: `Parse AI Tokens`.
    - *Failure Modes:* API rate limit, quota exhaustion, or invalid API key.
  - **Derive AI Verdict** & **Format AI Output** (`n8n-nodes-base.set`)
    - *Role:* Interprets Gemini's JSON response, normalizes retry actions (`auto_retry`, `recommend`, `none`), and optionally adjusts severity.
    - *Config:* Evaluates verdict strings (`RETRY_NOW`, `DO_NOT_RETRY`).
    - *Connections:* Input: `Parse AI Tokens` -> `Derive AI Verdict` -> `Format AI Output` -> `Escape Alert Fields`.
    - *Failure Modes:* Malformed JSON response from LLM.

#### 2.6 Multi-Channel Alert Dispatch & Actions
- **Overview:** Formats notification templates, routes messages across enabled communication channels, executes automated retries, and processes user action links (retry, mute, unmute).
- **Nodes Involved:** `Escape Alert Fields`, `Alert HTML Chunks`, `Format Alert`, `Route Notifications`, `Send Gmail Alert`, `Send Telegram Alert`, `Send Slack Alert`, `Send Discord Alert`, `Send Teams Alert`, `Send Webhook Alert`, `Check Delivered`, `Check First Delivery`, `Update Fingerprint`, `Log Incident Alerted`, `Limit Auto Retry`, `Check Auto Retry`, `Execute Auto Retry`, `Check Auto Result`, `Log Auto Retried`, `Log Auto Failed`, `Retry Webhook`, `Find Incident`, `Check Sim Retry`, `Serve Sim Retry`, `Check Incident Found`, `Check Already Retried`, `Serve Already Retried`, `Execute Retry`, `Check Retry`, `Log Incident Retried`, `Serve Retry Success`, `Log Retry Failed`, `Serve Retry Failed`, `Serve Retry Missing`, `Mute Webhook`, `Check Mute Params`, `Format Mute Data`, `Save Mute State`, `Serve Mute Page`, `Serve Invalid Mute`, `Unmute Webhook`, `Check Unmute Params`, `Clear Mute State`, `Serve Unmute Page`, `Serve Invalid Unmute`.
- **Node Details:**
  - **Format Alert** (`n8n-nodes-base.set`)
    - *Role:* Compiles subjects, alert text, webhook payloads, and action URLs for outgoing notifications.
    - *Config:* Combines HTML header/body chunks and constructs webhook JSON payloads.
    - *Connections:* Input: `Escape Alert Fields` | Output: `Route Notifications`.
    - *Failure Modes:* Expression evaluation errors.
  - **Route Notifications** (`n8n-nodes-base.switch`)
    - *Role:* Evaluates enabled channels and routes the alert payload to active notification nodes concurrently.
    - *Config:* Evaluates `config_email_enabled`, `config_tg_enabled`, `config_sl_enabled`, `config_dc_enabled`, `config_tm_enabled`, `config_wh_enabled`.
    - *Connections:* Input: `Format Alert` | Output: Routes to active notification nodes (`Send Gmail Alert`, etc.).
    - *Failure Modes:* None.
  - **Notification Nodes** (`Send Gmail Alert`, `Send Telegram Alert`, `Send Slack Alert`, `Send Discord Alert`, `Send Teams Alert`, `Send Webhook Alert`)
    - *Role:* Delivers formatted alerts to external platforms.
    - *Config:* 
      - *Gmail:* Uses Gmail OAuth2 credentials (`lucas.peyrin@gmail.com`), sends HTML email with sender name "Golden Error Handler".
      - *Other Channels:* Telegram, Slack, Discord, Microsoft Teams, and Webhook nodes (disabled by default on canvas, configurable by operators).
    - *Connections:* Input: `Route Notifications` | Output: `Check Delivered`.
    - *Failure Modes:* Authentication revocation, API downtime, or invalid recipient addresses.
  - **Execute Auto Retry** (`n8n-nodes-base.httpRequest`)
    - *Role:* Automatically retries the failed n8n execution via the n8n REST API if recommended by AI or rules.
    - *Config:* POST request to `${origin}/api/v1/executions/${execution_id}/retry` with `{"loadWorkflow": true}`. Uses n8n API credentials.
    - *Connections:* Input: `Check Auto Retry` | Output: `Check Auto Result`.
    - *Failure Modes:* API permission error or execution not found.
  - **Retry Webhook** Group (`Retry Webhook`, `Find Incident`, `Execute Retry`, etc.)
    - *Role:* Intercepts manual retry clicks from email/dashboard links (`/geh-retry`), verifies the incident, and triggers execution retry via n8n API.
    - *Config:* Path: `geh-retry`, queries `geh_incidents` by `execution_id`.
    - *Connections:* Input: Webhook | Output: Success/Failure HTML response pages.
    - *Failure Modes:* Expired or missing execution ID.
  - **Mute Webhook** & **Unmute Webhook** Groups (`Mute Webhook`, `Unmute Webhook`, etc.)
    - *Role:* Handle requests from alert links to mute (`/geh-mute`) or unmute (`/geh-unmute`) specific error fingerprints for 1h, 24h, 7d, or forever.
    - *Config:* Updates `geh_fingerprints` table (`muted` boolean and `mute_until` timestamp).
    - *Connections:* Input: Webhooks | Output: Confirmation HTML pages.
    - *Failure Modes:* Missing fingerprint query parameters.

#### 2.7 Observability Dashboards & Retention
- **Overview:** Serves activity logs, JSON telemetry feeds, architecture guides, and executes automated daily retention cleanups.
- **Nodes Involved:** `Activity Page Webhook`, `Get Recent Incidents`, `Prepare Incidents`, `Build Incident Action`, `Format Incident Row`, `Collect Incidents`, `Assemble Activity Page`, `Serve Activity Page`, `Activity Data Webhook`, `Get Incidents JSON`, `Collect Incidents JSON`, `Format Incidents JSON`, `Serve Incidents JSON`, `Guide Page Webhook`, `Get Config for Guide`, `Build Guide Page`, `Serve Guide Page`, `Daily Cleanup`, `Read Cleanup Config`, `Check Retention`, `Calculate Cutoff`, `Delete Old Incidents`.
- **Node Details:**
  - **Activity Page Webhook** (`n8n-nodes-base.webhook`)
    - *Role:* Serves the live observability dashboard at `/geh-activity`.
    - *Config:* Path: `geh-activity`.
    - *Connections:* Input: None | Output: `Get Recent Incidents`.
    - *Failure Modes:* Unauthenticated public access.
  - **Get Recent Incidents** (`n8n-nodes-base.dataTable`)
    - *Role:* Fetches the 30 most recent incident records for telemetry rendering.
    - *Config:* Operation: Get, Limit: 30, Ordered by creation date descending.
    - *Connections:* Input: `Activity Page Webhook` | Output: `Prepare Incidents`.
    - *Failure Modes:* Database error.
  - **Activity Data Webhook** Group (`Activity Data Webhook`, `Get Incidents JSON`, `Collect Incidents JSON`, `Format Incidents JSON`, `Serve Incidents JSON`)
    - *Role:* Exposes a raw JSON API feed of recent incidents and 24-hour summary metrics at `/geh-activity-data`.
    - *Config:* Limit: 10,000 records max for metrics computation.
    - *Connections:* Input: Webhook | Output: JSON response.
    - *Failure Modes:* Large dataset memory limits.
  - **Guide Page Webhook** (`n8n-nodes-base.webhook`)
    - *Role:* Serves the architecture reference and test console documentation at `/geh-guide`.
    - *Config:* Path: `geh-guide`.
    - *Connections:* Input: None | Output: `Get Config for Guide`.
    - *Failure Modes:* None.
  - **Daily Cleanup** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Triggers scheduled maintenance daily at 03:00 UTC.
    - *Config:* Cron expression: `0 3 * * *`.
    - *Connections:* Input: None | Output: `Read Cleanup Config`.
    - *Failure Modes:* None.
  - **Delete Old Incidents** (`n8n-nodes-base.dataTable`)
    - *Role:* Prunes incident history records older than the configured `retention_days` threshold.
    - *Config:* Operation: Delete Rows, Filter: `created_at` < `cutoff_iso`.
    - *Connections:* Input: `Calculate Cutoff` | Output: None.
    - *Failure Modes:* Database error.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| READ THIS | n8n-nodes-base.stickyNote | Documentation & quick start guide | None | None | ## READ THIS<br><br>The Golden Error Handler (geh) workflow looks huge, but the setup takes 2 minutes and it's very straight forward.<br><br>#### Here's the 3 steps you need to do:<br>1. Setup the Gmail, Gemini, and n8n API credentials (see red sticky notes)<br>2. Press the execute manual trigger<br>3. Publish the workflow<br><br>#### Then, just go to the "Config Page Webhook" node and open the production url!<br><br>You can then configure everything from that dashboard.<br><br>It should look something like: https://n8n.example.com/webhook/geh-config<br><br>Everything else is good to go! If in doubt, you can simply go to the 'Guide' page from the Golden Error Handler's dashboard |
| Endpoint Protection | n8n-nodes-base.stickyNote | Security warning regarding public webhook endpoints | None | None | ## Endpoint Protection<br><br>Public endpoints: anyone with these URLs can view activity, change settings, connect workflows, or send test alerts.<br><br>**Before sharing, protect every Webhook node with _Authentication > Basic Auth_ and use the same credential, or require sign-in at your reverse proxy.**<br><br>Email actions will then prompt for sign-in. |
| Run Setup Once | n8n-nodes-base.manualTrigger | Manual trigger to initialize tables | None | Create Config Table | |
| Create Config Table | n8n-nodes-base.dataTable | Creates `geh_config` table | Run Setup Once | Create Fingerprints | |
| Create Fingerprints | n8n-nodes-base.dataTable | Creates `geh_fingerprints` table | Create Config Table | Create Incidents | |
| Create Incidents | n8n-nodes-base.dataTable | Creates `geh_incidents` table | Create Fingerprints | Read Existing Config | |
| Read Existing Config | n8n-nodes-base.dataTable | Fetches main configuration row | Create Incidents | Check Config Seed | |
| Check Config Seed | n8n-nodes-base.if | Checks if config seed exists | Read Existing Config | Seed Default Config | |
| Seed Default Config | n8n-nodes-base.dataTable | Upserts default configuration | Check Config Seed | None | |
| Config Page Webhook | n8n-nodes-base.webhook | Serves Settings UI webhook | None | Get Config For Form | Hosted Config UI |
| Get Config For Form | n8n-nodes-base.dataTable | Reads config for form rendering | Config Page Webhook | Config HTML Chunks | Hosted Config UI |
| Config HTML Chunks | n8n-nodes-base.set | Builds HTML chunks for settings UI | Get Config For Form | Prepare Config Fields | Hosted Config UI |
| Prepare Config Fields | n8n-nodes-base.set | Sanitizes form inputs | Config HTML Chunks | Build Duration Hints | Hosted Config UI |
| Build Duration Hints | n8n-nodes-base.set | Computes backoff duration strings | Prepare Config Fields | Build Config Form | Hosted Config UI |
| Build Config Form | n8n-nodes-base.set | Assembles settings HTML page | Build Duration Hints | Serve Config Form | Hosted Config UI |
| Serve Config Form | n8n-nodes-base.respondToWebhook | Returns settings HTML page | Build Config Form | None | Hosted Config UI |
| Save Config Webhook | n8n-nodes-base.webhook | Receives settings form POST | None | Check Config Body | Hosted Config UI |
| Check Config Body | n8n-nodes-base.if | Validates configuration parameters | Save Config Webhook | Save Config Row, Build Config Error | Hosted Config UI |
| Save Config Row | n8n-nodes-base.dataTable | Saves updated settings | Check Config Body | Redirect To Assign | Hosted Config UI |
| Redirect To Assign | n8n-nodes-base.respondToWebhook | Redirects to Connect page | Save Config Row | None | Hosted Config UI |
| Build Config Error | n8n-nodes-base.set | Formats validation error messages | Check Config Body | Config Save Error | Hosted Config UI |
| Config Save Error | n8n-nodes-base.respondToWebhook | Returns validation error HTML | Build Config Error | None | Hosted Config UI |
| Assign Page Webhook | n8n-nodes-base.webhook | Serves Connect UI webhook | None | Get All Workflows | Bulk Assignment |
| Get All Workflows | n8n-nodes-base.n8n | Fetches instance workflows via API | Assign Page Webhook | Format Workflow Item | Bulk Assignment |
| Format Workflow Item | n8n-nodes-base.set | Formats workflow item attributes | Get All Workflows | Build Workflow Rows | Bulk Assignment |
| Build Workflow Rows | n8n-nodes-base.set | Generates workflow table rows | Format Workflow Item | Collect Rows | Bulk Assignment |
| Collect Rows | n8n-nodes-base.aggregate | Aggregates workflow rows | Build Workflow Rows | Assemble Assign Page | Bulk Assignment |
| Assemble Assign Page | n8n-nodes-base.set | Compiles Connect UI HTML page | Collect Rows | Serve Assign Page | Bulk Assignment |
| Serve Assign Page | n8n-nodes-base.respondToWebhook | Returns Connect UI HTML page | Assemble Assign Page | None | Bulk Assignment |
| Workflow List Webhook | n8n-nodes-base.webhook | Serves workflow JSON feed | None | Get Wfs For API | Bulk Assignment |
| Get Wfs For API | n8n-nodes-base.n8n | Fetches workflows for API JSON feed | Workflow List Webhook | Collect Wfs For API | Bulk Assignment |
| Collect Wfs For API | n8n-nodes-base.aggregate | Aggregates workflows for JSON feed | Get Wfs For API | Format Wfs For API | Bulk Assignment |
| Format Wfs For API | n8n-nodes-base.set | Formats workflow JSON feed | Collect Wfs For API | Serve Workflow List | Bulk Assignment |
| Serve Workflow List | n8n-nodes-base.respondToWebhook | Returns workflow JSON feed | Format Wfs For API | None | Bulk Assignment |
| Save Assign Webhook | n8n-nodes-base.webhook | Receives workflow connection POST | None | Parse Workflow IDs | Bulk Assignment |
| Parse Workflow IDs | n8n-nodes-base.set | Extracts selected workflow IDs | Save Assign Webhook | Check Assign Empty | Bulk Assignment |
| Check Assign Empty | n8n-nodes-base.if | Validates non-empty selection | Parse Workflow IDs | Split Workflow IDs, Assign Empty | Bulk Assignment |
| Split Workflow IDs | n8n-nodes-base.splitOut | Splits workflow IDs for iteration | Check Assign Empty | Get Workflow Record | Bulk Assignment |
| Get Workflow Record | n8n-nodes-base.n8n | Fetches individual workflow definition | Split Workflow IDs | Set Error Handler, Assign Denied | Bulk Assignment |
| Set Error Handler | n8n-nodes-base.set | Injects errorWorkflow setting | Get Workflow Record | Update N8n Workflow | Bulk Assignment |
| Update N8n Workflow | n8n-nodes-base.n8n | Updates workflow via API | Set Error Handler | Assign Confirmed | Bulk Assignment |
| Assign Confirmed | n8n-nodes-base.respondToWebhook | Returns connection success HTML | Update N8n Workflow | None | Bulk Assignment |
| Assign Empty | n8n-nodes-base.respondToWebhook | Returns empty selection error HTML | Check Assign Empty | None | Bulk Assignment |
| Assign Denied | n8n-nodes-base.respondToWebhook | Returns workflow not found HTML | Get Workflow Record | None | Bulk Assignment |
| Disconnect Webhook | n8n-nodes-base.webhook | Receives disconnection POST | None | Parse Disconnect IDs | Bulk Assignment |
| Parse Disconnect IDs | n8n-nodes-base.set | Extracts target workflow ID | Disconnect Webhook | Check Disconnect ID | Bulk Assignment |
| Check Disconnect ID | n8n-nodes-base.if | Validates disconnect ID presence | Parse Disconnect IDs | Get Disconnect Record, Disconnect Empty | Bulk Assignment |
| Get Disconnect Record | n8n-nodes-base.n8n | Fetches workflow to disconnect | Check Disconnect ID | Check Disconnect Wf | Bulk Assignment |
| Check Disconnect Wf | n8n-nodes-base.if | Verifies current error handler binding | Get Disconnect Record | Clear Error Handler, Disconnect Denied | Bulk Assignment |
| Clear Error Handler | n8n-nodes-base.set | Clears errorWorkflow setting | Check Disconnect Wf | Update Disconnect Wf | Bulk Assignment |
| Update Disconnect Wf | n8n-nodes-base.n8n | Updates workflow via API | Clear Error Handler | Disconnect Confirmed | Bulk Assignment |
| Disconnect Confirmed | n8n-nodes-base.respondToWebhook | Returns disconnection success HTML | Update Disconnect Wf | None | Bulk Assignment |
| Disconnect Denied | n8n-nodes-base.respondToWebhook | Returns invalid disconnect HTML | Check Disconnect Wf | None | Bulk Assignment |
| Disconnect Empty | n8n-nodes-base.respondToWebhook | Returns missing ID error HTML | Check Disconnect ID | None | Bulk Assignment |
| Gemini Diagnostics | n8n-nodes-base.stickyNote | Architecture note for Gemini AI integration | None | None | ## Gemini Diagnostics<br><br>Gemini analyzes the failure and explains what went wrong in plain English. Connect your key in the Gemini Model node below. If AI fails, your alert still sends. |
| Error Trigger | n8n-nodes-base.errorTrigger | Triggers on monitored workflow failure | None | Ingest Error Payload | Error Trigger & Guard |
| Ingest Error Payload | n8n-nodes-base.set | Normalizes error execution payload | Error Trigger, Serve Simulate HTML | Guard Payload | Error Trigger & Guard |
| Simulate Webhook | n8n-nodes-base.webhook | Serves simulation test endpoint | None | Build Sim Payload | Workflow Guide |
| Build Sim Payload | n8n-nodes-base.set | Builds synthetic test error payload | Simulate Webhook | Serve Simulate HTML | Workflow Guide |
| Serve Simulate HTML | n8n-nodes-base.respondToWebhook | Returns simulation preview HTML | Build Sim Payload | Ingest Error Payload | Workflow Guide |
| Guard Payload | n8n-nodes-base.if | Validates execution ID and message | Ingest Error Payload | Read Config, Fail Loud Guard | Error Trigger & Guard |
| Read Config | n8n-nodes-base.dataTable | Reads configuration for error pipeline | Guard Payload | Filter Manual Runs | |
| Filter Manual Runs | n8n-nodes-base.if | Filters manual test runs | Read Config | Format Error Data | |
| Format Error Data | n8n-nodes-base.set | Cleans name and prepares hash input | Filter Manual Runs | Hash Fingerprint | |
| Hash Fingerprint | n8n-nodes-base.crypto | Generates SHA-256 error fingerprint | Format Error Data | Lookup Fingerprint | |
| Lookup Fingerprint | n8n-nodes-base.dataTable | Looks up existing error fingerprint | Hash Fingerprint | Check If Muted | |
| Check If Muted | n8n-nodes-base.if | Checks mute state and expiration | Lookup Fingerprint | Log Incident Muted, Calculate Counts | |
| Log Incident Muted | n8n-nodes-base.dataTable | Logs muted incident execution | Check If Muted | Update Muted Seen | |
| Update Muted Seen | n8n-nodes-base.dataTable | Updates last seen on muted fingerprint | Log Incident Muted | None | |
| Calculate Counts | n8n-nodes-base.set | Computes 24h occurrence windows | Check If Muted | Categorize Error | |
| Categorize Error | n8n-nodes-base.switch | Classifies error category | Calculate Counts | Category Auth, Category Rate Limit, Category Timeout, Category Payload, Category Other | |
| Category Auth | n8n-nodes-base.set | Sets auth error category and severity | Categorize Error | Calculate Severity | |
| Calculate Severity | n8n-nodes-base.set | Evaluates severity and backoff delay | Category Auth, Category Rate Limit, Category Timeout, Category Payload, Category Other | Check Backoff | |
| Check Backoff | n8n-nodes-base.if | Enforces pacing and max alerts | Calculate Severity | Update Suppressed, Check AI Enabled | |
| Update Suppressed | n8n-nodes-base.dataTable | Updates suppressed fingerprint count | Check Backoff | Log Suppressed | |
| Log Suppressed | n8n-nodes-base.dataTable | Logs suppressed backoff incident | Update Suppressed | None | |
| Check AI Enabled | n8n-nodes-base.if | Checks if AI diagnostics are enabled | Check Backoff | Read Retry History, Skip AI Summary | Gemini Diagnostics |
| Read Retry History | n8n-nodes-base.dataTable | Reads recent retry history | Check AI Enabled | Collect Retry History, Build History Fallback | Gemini Diagnostics |
| Collect Retry History | n8n-nodes-base.aggregate | Aggregates retry history records | Read Retry History | Check History Read | Gemini Diagnostics |
| Check History Read | n8n-nodes-base.if | Validates history read success | Collect Retry History | Count Retry Outcomes, Build History Fallback | Gemini Diagnostics |
| Count Retry Outcomes | n8n-nodes-base.set | Counts past retried outcomes | Check History Read | Build Retry Context | Gemini Diagnostics |
| Build Retry Context | n8n-nodes-base.set | Builds retry context string | Count Retry Outcomes | Build AI Prompt | Gemini Diagnostics |
| Build AI Prompt | n8n-nodes-base.set | Compiles diagnostic prompt for AI | Build Retry Context | Generate AI Summary, Build History Fallback | Gemini Diagnostics |
| Generate AI Summary | @n8n/n8n-nodes-langchain.chainLlm | Generates AI diagnostic summary | Build AI Prompt, Gemini Model, AI Output Parser | Parse AI Tokens | Gemini Diagnostics |
| Gemini Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Google Gemini language model | None | Generate AI Summary | Gemini Diagnostics |
| AI Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured JSON output parser | None | Generate AI Summary | Gemini Diagnostics |
| Parse AI Tokens | n8n-nodes-base.set | Parses AI diagnostic tokens | Generate AI Summary | Derive AI Verdict | Gemini Diagnostics |
| Derive AI Verdict | n8n-nodes-base.set | Derives retry action and severity | Parse AI Tokens | Format AI Output | Gemini Diagnostics |
| Format AI Output | n8n-nodes-base.set | Formats AI output attributes | Derive AI Verdict | Escape Alert Fields | Gemini Diagnostics |
| Escape Alert Fields | n8n-nodes-base.set | Escapes HTML entities for alerts | Format AI Output, Build History Fallback, Skip AI Summary | Alert HTML Chunks | |
| Alert HTML Chunks | n8n-nodes-base.set | Builds HTML chunks for email alerts | Escape Alert Fields | Format Alert | Multi-Channel Alert |
| Build History Fallback | n8n-nodes-base.set | Provides fallback if history read fails | Read Retry History, Check History Read, Build Retry Context, Build AI Prompt | Escape Alert Fields | Gemini Diagnostics |
| Skip AI Summary | n8n-nodes-base.set | Bypasses AI if disabled | Check AI Enabled | Escape Alert Fields | |
| Category Rate Limit | n8n-nodes-base.set | Sets rate limit category and severity | Categorize Error | Calculate Severity | |
| Category Timeout | n8n-nodes-base.set | Sets timeout category and severity | Categorize Error | Calculate Severity | |
| Category Payload | n8n-nodes-base.set | Sets payload category and severity | Categorize Error | Calculate Severity | |
| Category Other | n8n-nodes-base.set | Sets fallback category and severity | Categorize Error | Calculate Severity | |
| Fail Loud Guard | n8n-nodes-base.stopAndError | Halts workflow on invalid payload | Guard Payload | None | Error Trigger & Guard |
| Format Alert | n8n-nodes-base.set | Compiles final alert parameters | Alert HTML Chunks | Route Notifications | Multi-Channel Alert |
| Route Notifications | n8n-nodes-base.switch | Routes alert to enabled channels | Format Alert | Send Gmail Alert, Send Telegram Alert, Send Slack Alert, Send Discord Alert, Send Teams Alert, Send Webhook Alert | Multi-Channel Alert |
| Send Gmail Alert | n8n-nodes-base.gmail | Sends email notification via Gmail | Route Notifications | Check Delivered | Multi-Channel Alert |
| Send Telegram Alert | n8n-nodes-base.telegram | Sends Telegram notification | Route Notifications | Check Delivered | Multi-Channel Alert |
| Send Slack Alert | n8n-nodes-base.slack | Sends Slack notification | Route Notifications | Check Delivered | Multi-Channel Alert |
| Send Discord Alert | n8n-nodes-base.discord | Sends Discord notification | Route Notifications | Check Delivered | Multi-Channel Alert |
| Send Teams Alert | n8n-nodes-base.microsoftTeams | Sends Microsoft Teams notification | Route Notifications | Check Delivered | Multi-Channel Alert |
| Send Webhook Alert | n8n-nodes-base.httpRequest | Sends HTTP webhook notification | Route Notifications | Check Delivered | Multi-Channel Alert |
| Check Delivered | n8n-nodes-base.if | Verifies delivery output | Send Gmail Alert, Send Telegram Alert, Send Slack Alert, Send Discord Alert, Send Teams Alert, Send Webhook Alert | Check First Delivery | |
| Check First Delivery | n8n-nodes-base.if | Checks if first time alerting | Check Delivered | Update Fingerprint, Limit Auto Retry | |
| Update Fingerprint | n8n-nodes-base.dataTable | Updates fingerprint last alert timestamp | Check First Delivery | Log Incident Alerted | |
| Log Incident Alerted | n8n-nodes-base.dataTable | Logs alerted incident execution | Update Fingerprint | Limit Auto Retry | |
| Limit Auto Retry | n8n-nodes-base.limit | Limits execution items for auto-retry | Check First Delivery, Log Incident Alerted | Check Auto Retry | |
| Check Auto Retry | n8n-nodes-base.if | Checks if auto-retry is recommended | Limit Auto Retry | Execute Auto Retry | |
| Execute Auto Retry | n8n-nodes-base.httpRequest | Triggers auto-retry via n8n API | Check Auto Retry | Check Auto Result | |
| Check Auto Result | n8n-nodes-base.if | Verifies auto-retry API response | Execute Auto Retry | Log Auto Retried, Log Auto Failed | |
| Log Auto Retried | n8n-nodes-base.dataTable | Logs successful auto-retry status | Check Auto Result | None | |
| Log Auto Failed | n8n-nodes-base.dataTable | Logs failed auto-retry status | Check Auto Result | None | |
| Retry Webhook | n8n-nodes-base.webhook | Serves manual retry webhook | None | Find Incident | Security & Actions |
| Find Incident | n8n-nodes-base.dataTable | Finds incident by execution ID | Retry Webhook | Check Sim Retry | Security & Actions |
| Check Sim Retry | n8n-nodes-base.if | Checks if simulation retry | Find Incident | Serve Sim Retry, Check Incident Found | Security & Actions |
| Serve Sim Retry | n8n-nodes-base.respondToWebhook | Returns simulated retry error HTML | Check Sim Retry | None | Security & Actions |
| Check Incident Found | n8n-nodes-base.if | Validates incident existence | Check Sim Retry | Check Already Retried, Serve Retry Missing | Security & Actions |
| Check Already Retried | n8n-nodes-base.if | Checks if already retried | Check Incident Found | Serve Already Retried, Execute Retry | Security & Actions |
| Serve Already Retried | n8n-nodes-base.respondToWebhook | Returns already retried HTML | Check Already Retried | None | Security & Actions |
| Execute Retry | n8n-nodes-base.httpRequest | Triggers manual execution retry | Check Already Retried | Check Retry | Security & Actions |
| Check Retry | n8n-nodes-base.if | Verifies manual retry API result | Execute Retry | Log Incident Retried, Log Retry Failed | Security & Actions |
| Log Incident Retried | n8n-nodes-base.dataTable | Logs manual retry success | Check Retry | Serve Retry Success | Security & Actions |
| Serve Retry Success | n8n-nodes-base.respondToWebhook | Returns retry success HTML | Log Incident Retried | None | Security & Actions |
| Log Retry Failed | n8n-nodes-base.dataTable | Logs manual retry failure | Check Retry | Serve Retry Failed | Security & Actions |
| Serve Retry Failed | n8n-nodes-base.respondToWebhook | Returns retry failure HTML | Log Retry Failed | None | Security & Actions |
| Serve Retry Missing | n8n-nodes-base.respondToWebhook | Returns incident not found HTML | Check Incident Found | None | Security & Actions |
| Mute Webhook | n8n-nodes-base.webhook | Serves mute webhook endpoint | None | Check Mute Params | Security & Actions |
| Check Mute Params | n8n-nodes-base.if | Validates mute query parameters | Mute Webhook | Format Mute Data, Serve Invalid Mute | Security & Actions |
| Format Mute Data | n8n-nodes-base.set | Computes mute expiration timestamp | Check Mute Params | Save Mute State | Security & Actions |
| Save Mute State | n8n-nodes-base.dataTable | Saves mute state to fingerprint | Format Mute Data | Serve Mute Page | Security & Actions |
| Serve Mute Page | n8n-nodes-base.respondToWebhook | Returns mute confirmation HTML | Save Mute State | None | Security & Actions |
| Serve Invalid Mute | n8n-nodes-base.respondToWebhook | Returns invalid mute HTML | Check Mute Params | None | Security & Actions |
| Unmute Webhook | n8n-nodes-base.webhook | Serves unmute webhook endpoint | None | Check Unmute Params | Security & Actions |
| Check Unmute Params | n8n-nodes-base.if | Validates unmute query parameters | Unmute Webhook | Clear Mute State, Serve Invalid Unmute | Security & Actions |
| Clear Mute State | n8n-nodes-base.dataTable | Clears mute state on fingerprint | Check Unmute Params | Serve Unmute Page | Security & Actions |
| Serve Unmute Page | n8n-nodes-base.respondToWebhook | Returns unmute confirmation HTML | Clear Mute State | None | Security & Actions |
| Serve Invalid Unmute | n8n-nodes-base.respondToWebhook | Returns invalid unmute HTML | Check Unmute Params | None | Security & Actions |
| Activity Page Webhook | n8n-nodes-base.webhook | Serves activity dashboard webhook | None | Get Recent Incidents | Activity Dashboard |
| Get Recent Incidents | n8n-nodes-base.dataTable | Fetches recent incident records | Activity Page Webhook | Prepare Incidents | Activity Dashboard |
| Prepare Incidents | n8n-nodes-base.set | Formats incident table attributes | Get Recent Incidents | Build Incident Action | Activity Dashboard |
| Build Incident Action | n8n-nodes-base.set | Builds action buttons for incidents | Prepare Incidents | Format Incident Row | Activity Dashboard |
| Format Incident Row | n8n-nodes-base.set | Formats HTML table rows | Build Incident Action | Collect Incidents | Activity Dashboard |
| Collect Incidents | n8n-nodes-base.aggregate | Aggregates incident rows | Format Incident Row | Assemble Activity Page | Activity Dashboard |
| Assemble Activity Page | n8n-nodes-base.set | Compiles activity dashboard HTML | Collect Incidents | Serve Activity Page | Activity Dashboard |
| Serve Activity Page | n8n-nodes-base.respondToWebhook | Returns activity dashboard HTML | Assemble Activity Page | None | Activity Dashboard |
| Activity Data Webhook | n8n-nodes-base.webhook | Serves incident JSON feed webhook | None | Get Incidents JSON | Activity Dashboard |
| Get Incidents JSON | n8n-nodes-base.dataTable | Fetches incidents for JSON feed | Activity Data Webhook | Collect Incidents JSON | Activity Dashboard |
| Collect Incidents JSON | n8n-nodes-base.aggregate | Aggregates incidents for JSON feed | Get Incidents JSON | Format Incidents JSON | Activity Dashboard |
| Format Incidents JSON | n8n-nodes-base.set | Formats incidents JSON feed | Collect Incidents JSON | Serve Incidents JSON | Activity Dashboard |
| Serve Incidents JSON | n8n-nodes-base.respondToWebhook | Returns incidents JSON feed | Format Incidents JSON | None | Activity Dashboard |
| Guide Page Webhook | n8n-nodes-base.webhook | Serves guide page webhook | None | Get Config for Guide | Workflow Guide |
| Get Config for Guide | n8n-nodes-base.dataTable | Fetches config for guide rendering | Guide Page Webhook | Build Guide Page | Workflow Guide |
| Build Guide Page | n8n-nodes-base.set | Compiles guide page HTML | Get Config for Guide | Serve Guide Page | Workflow Guide |
| Serve Guide Page | n8n-nodes-base.respondToWebhook | Returns guide page HTML | Build Guide Page | None | Workflow Guide |
| Daily Cleanup | n8n-nodes-base.scheduleTrigger | Scheduled daily trigger (03:00 UTC) | None | Read Cleanup Config | Safety & Retention |
| Read Cleanup Config | n8n-nodes-base.dataTable | Reads retention settings | Daily Cleanup | Check Retention | Safety & Retention |
| Check Retention | n8n-nodes-base.if | Validates retention setting | Read Cleanup Config | Calculate Cutoff | Safety & Retention |
| Calculate Cutoff | n8n-nodes-base.set | Computes retention cutoff ISO date | Check Retention | Delete Old Incidents | Safety & Retention |
| Delete Old Incidents | n8n-nodes-base.dataTable | Deletes expired incident records | Calculate Cutoff | None | Safety & Retention |
| Activity Dashboard | n8n-nodes-base.stickyNote | Architecture note for activity dashboard | None | None | ## Activity Dashboard<br><br>Track real-time error telemetry and failure rates. View incident history and trigger execution retries with one click at /geh-activity. |
| Workflow Guide | n8n-nodes-base.stickyNote | Architecture note for workflow guide | None | None | ## Workflow Guide<br><br>Comprehensive operator documentation, architecture walkthrough, quick links, and simulated error generator at /geh-guide. |
| Hosted Config UI | n8n-nodes-base.stickyNote | Architecture note for configuration UI | None | None | ## Hosted Config UI<br><br>Open the production webhook URL for /geh-config in your browser to tune notification thresholds, recipient email, Telegram channel, and alert pacing anytime without touching the canvas. |
| Bulk Assignment | n8n-nodes-base.stickyNote | Architecture note for bulk workflow assignment | None | None | ## Bulk Assignment<br><br>Visit /geh-assign to view your workflows and attach the Golden Error Handler to them in one click. It uses your native n8n API credential to update their settings safely. |
| Safety & Retention | n8n-nodes-base.stickyNote | Architecture note for safety limits and retention | None | None | ## Safety Limits & Retention<br><br>High-volume queries cap at 10,000 records to protect workflow execution. If limits are reached, totals show an incomplete count indicator. Scheduled retention checks run daily and delete incidents older than your configured duration. Retention defaults to Never until explicitly chosen. |
| Error Trigger & Guard | n8n-nodes-base.stickyNote | Architecture note for error intake | None | None | ## Error Trigger & Guard<br><br>Catches unhandled errors from connected workflows. A payload guard validates incoming execution data and prevents self-referential error loops. |
| Multi-Channel Alert | n8n-nodes-base.stickyNote | Architecture note for multi-channel alerts | None | None | ## Multi-Channel Alert<br><br>Alerts route to Gmail, Telegram, Teams, or Webhook with links to retry or mute. Native Slack, Discord, and Teams nodes are ready on canvas. |
| Security & Actions | n8n-nodes-base.stickyNote | Architecture note for security and action webhooks | None | None | ## Security & Actions<br><br>Action webhooks let you retry executions or mute recurring alerts directly from notifications or the dashboard without opening n8n. |
| Sticky Note | n8n-nodes-base.stickyNote | Unlabeled note | None | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild the Golden Error Handler workflow manually in n8n without importing the JSON, follow these step-by-step instructions:

#### Step 1: Initialize Database Tables & Setup Trigger
1. Create a **Manual Trigger** node (`Run Setup Once`).
2. Add three **n8n Data Table** nodes in sequence:
   - **Create Config Table**: Operation `Create`, Table Name `geh_config`. Define columns: `config_key` (string), `email_enabled` (boolean), `email_to` (string), `telegram_enabled` (boolean), `telegram_chat_id` (string), `slack_enabled` (boolean), `slack_channel` (string), `discord_enabled` (boolean), `discord_channel` (string), `teams_enabled` (boolean), `teams_channel` (string), `webhook_enabled` (boolean), `webhook_url` (string), `ai_enabled` (boolean), `ai_auto_retry` (boolean), `ai_severity_adjust` (boolean), `include_manual_runs` (boolean), `cooldown_minutes` (number), `sev_medium_count` (number), `sev_high_count` (number), `base_url` (string), `backoff_start` (number), `backoff_max` (number), `max_alerts` (number), `retention_days` (number). Enable `createIfNotExists`.
   - **Create Fingerprints**: Operation `Create`, Table Name `geh_fingerprints`. Columns: `fingerprint`, `workflow_id`, `workflow_name`, `node_name`, `error_message`, `first_seen`, `last_seen`, `count_24h` (number), `window_start`, `last_alert_at`, `attempt_count` (number), `suppressed_since_alert` (number), `muted` (boolean), `mute_until`.
   - **Create Incidents**: Operation `Create`, Table Name `geh_incidents`. Columns: `fingerprint`, `execution_id`, `execution_url`, `workflow_id`, `workflow_name`, `node_name`, `error_message`, `category`, `severity`, `status`, `ai_summary`, `created_at`, `severity_rules`, `severity_ai`, `severity_reason`.
3. Add a **Data Table** node (`Read Existing Config`) to get the row where `config_key` = `main`.
4. Add an **If** node (`Check Config Seed`) checking `{{ !$json.config_key }}`.
5. Connect True branch to a **Data Table** node (`Seed Default Config`) with operation `Upsert`, table `geh_config`, key `config_key` = `main`, and default values.

#### Step 2: Build Settings Web UI & Save Endpoints
1. Create a **Webhook** node (`Config Page Webhook`) with path `geh-config`, response mode `Response Node`.
2. Chain a **Data Table** (`Get Config For Form`) to fetch the main config row.
3. Add **Set** nodes (`Config HTML Chunks`, `Prepare Config Fields`, `Build Duration Hints`, `Build Config Form`) to generate and sanitize the HTML form markup.
4. Add a **Respond to Webhook** node (`Serve Config Form`) returning Content-Type `text/html`.
5. Create a second **Webhook** node (`Save Config Webhook`) with method `POST` and path `geh-config-save`.
6. Add an **If** node (`Check Config Body`) to validate parameters (backoff ranges, retention days, URL syntax).
7. Connect True to a **Data Table** (`Save Config Row`) with operation `Upsert`, followed by a **Respond to Webhook** (`Redirect To Assign`) returning a meta-refresh redirect page.
8. Connect False to a **Set** (`Build Config Error`) and **Respond to Webhook** (`Config Save Error`).

#### Step 3: Build Workflow Connection Manager
1. Create a **Webhook** node (`Assign Page Webhook`) at path `geh-assign`.
2. Add an **n8n** node (`Get All Workflows`) configured to return up to 10,000 workflows using an **n8n API credential** (`n8nApi`).
3. Chain **Set** and **Aggregate** nodes (`Format Workflow Item`, `Build Workflow Rows`, `Collect Rows`, `Assemble Assign Page`) and a **Respond to Webhook** (`Serve Assign Page`) to render the Connect dashboard.
4. Create a **Webhook** node (`Save Assign Webhook`) at path `geh-assign-save` (POST).
5. Add **Set**, **If**, and **Split Out** nodes to parse and iterate over `workflows_to_update`.
6. Use an **n8n** node (`Get Workflow Record`) to fetch each workflow definition, a **Set** node (`Set Error Handler`) to inject `$workflow.id` into `settings.errorWorkflow`, and an **n8n** node (`Update N8n Workflow`) to save changes via the API.
7. Implement matching webhooks and logic for `/geh-workflows`, `/geh-disconnect`, and `/geh-assign-save`.

#### Step 4: Build Error Ingestion & Categorization Pipeline
1. Create an **Error Trigger** node (`Error Trigger`).
2. Create a **Webhook** node (`Simulate Webhook`) at path `geh-simulate-error` with scenario handling (`rate_limit`, `auth`, `timeout`, `payload`, `exception`) for testing.
3. Add an **Ingest Error Payload** **Set** node to normalize incoming execution attributes.
4. Add an **If** node (`Guard Payload`) verifying `raw_ex_id`, `raw_wf_id`, and `raw_err_msg` are present. Route failures to a **Stop and Error** node (`Fail Loud Guard`).
5. Chain Data Table lookup and If nodes (`Read Config`, `Filter Manual Runs`, `Format Error Data`, `Hash Fingerprint` using SHA-256 crypto, `Lookup Fingerprint`, `Check If Muted`).
6. If muted, log to `geh_incidents` (`Log Incident Muted`) and update seen timestamp. If active, calculate occurrence counts (`Calculate Counts`).
7. Add a **Switch** node (`Categorize Error`) with rules matching error strings for `auth`, `rate_limit`, `timeout`, and `payload`, with a fallback to `exception`.
8. Assign base severities (`auth` -> high; `rate_limit`, `timeout`, `payload` -> medium; `exception`/`other` -> low).

#### Step 5: Build Severity, Backoff, & AI Diagnostics
1. Add a **Set** node (`Calculate Severity`) to evaluate 24-hour recurrence counts against `sev_medium_count` and `sev_high_count`, and compute exponential backoff delay.
2. Add an **If** node (`Check Backoff`) to suppress alerting if within the backoff window or alert cap.
3. If not suppressed, check if AI is enabled (`Check AI Enabled`).
4. If AI is enabled, read recent incident retry history (`Read Retry History`, `Collect Retry History`), compile the prompt (`Build AI Prompt`), and execute the **Advanced AI / LangChain Chain** node (`Generate AI Summary`) linked to a **Google Gemini Chat Model** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini` using credential `googlePalmApi`) and a **Structured Output Parser** (`@n8n/n8n-nodes-langchain.outputParserStructured`).
5. Parse AI tokens, derive retry actions (`auto_retry`, `recommend`, `none`), and format alert fields.

#### Step 6: Multi-Channel Alert Dispatch & Action Handlers
1. Format email/webhook payloads using **Set** nodes (`Escape Alert Fields`, `Alert HTML Chunks`, `Format Alert`).
2. Add a **Switch** node (`Route Notifications`) evaluating `config_email_enabled`, `config_tg_enabled`, `config_sl_enabled`, `config_dc_enabled`, `config_tm_enabled`, and `config_wh_enabled`.
3. Connect the email output to a **Gmail** node (`Send Gmail Alert`) configured with a **Gmail OAuth2 credential**, setting sender name to "Golden Error Handler".
4. Add disabled placeholder nodes for Telegram, Slack, Discord, Microsoft Teams, and HTTP Webhook (enable and configure credentials as needed).
5. Verify delivery, update fingerprint state, log alerted incidents in `geh_incidents`, and evaluate auto-retry conditions (`Check Auto Retry` -> `Execute Auto Retry` via n8n API HTTP request).
6. Create action webhooks for manual retries (`/geh-retry`), muting (`/geh-mute`), and unmuting (`/geh-unmute`) communicating with Data Tables and responding with confirmation HTML pages.

#### Step 7: Observability Dashboards & Daily Cleanup
1. Create Webhook nodes for `/geh-activity`, `/geh-activity-data`, and `/geh-guide`, chaining Data Table, aggregate, and set nodes to render telemetry dashboards and JSON feeds.
2. Create a **Schedule Trigger** node (`Daily Cleanup`) set to cron expression `0 3 * * *`.
3. Chain Data Table and Set nodes (`Read Cleanup Config`, `Check Retention`, `Calculate Cutoff`) to an **n8n Data Table** node (`Delete Old Incidents`) configured with operation `Delete Rows` where `created_at` < `cutoff_iso`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Golden Error Handler (geh) Repository & Overview | Centralized error handler and browser-based triage dashboard for n8n. |
| Google Gemini API Documentation | Used for AI-powered error diagnostics and severity guidance. |
| n8n REST API Documentation | Required for querying workflows, updating error workflow bindings, and retrying executions. |