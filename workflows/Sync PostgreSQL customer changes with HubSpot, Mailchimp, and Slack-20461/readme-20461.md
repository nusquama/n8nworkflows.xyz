Sync PostgreSQL customer changes with HubSpot, Mailchimp, and Slack

https://n8nworkflows.xyz/workflows/sync-postgresql-customer-changes-with-hubspot--mailchimp--and-slack-20461


# Sync PostgreSQL customer changes with HubSpot, Mailchimp, and Slack

### 1. Workflow Overview

This workflow implements a robust Change Data Capture (CDC) pipeline using the transactional outbox pattern. It captures `INSERT`, `UPDATE`, and `DELETE` database operations from a PostgreSQL source table and synchronizes them to downstream SaaS platforms (HubSpot, Mailchimp, and Slack). Designed for reliability, the pipeline runs on a 1-minute schedule, uses atomic row-level leasing (`FOR UPDATE SKIP LOCKED`) to support concurrent execution safely, implements exponential backoff retries, filters out permanently failed targets, handles dead-letter alerts via Slack, and exposes an administrative webhook endpoint (`POST /cdc-requeue`) to manually reset and reprocess failed events.

The workflow logic is divided into the following functional blocks:
- **1.1 One-Time Setup:** Initializes the PostgreSQL outbox schema, capture function, and trigger bindings.
- **1.2 Poll & Claim Changes:** Executes on a 1-minute interval, loads configuration settings, and claims a leased batch of pending/retryable events from PostgreSQL while maintaining per-row order.
- **1.3 Plan Fan-Out:** Expands each claimed database event into targeted delivery tasks for enabled SaaS destinations, omitting targets that have already succeeded or permanently failed.
- **1.4 Deliver to SaaS Targets:** Iteratively routes and executes HTTP requests to HubSpot, Mailchimp, or Slack, capturing outcomes and edge-case status codes.
- **1.5 Finalize Events:** Evaluates delivery metrics, calculates exponential backoff retry delays, marks permanently failed targets, and updates event statuses (`delivered`, `retry`, or `dead`) in PostgreSQL.
- **1.6 Report & Alert:** Compiles an execution summary and triggers a Slack notification if any events are dead-lettered.
- **1.7 Requeue API:** Provides a webhook endpoint (`POST /cdc-requeue`) to validate, reset, and re-queue dead-lettered event IDs back to pending status.

---

### 2. Block-by-Block Analysis

---

#### 1.1 One-Time Setup

**Overview:**  
Initializes the target database environment by creating the `cdc_outbox` table, indices for operational speed, the `cdc_capture()` database trigger function, and an attached trigger on the source table (`public.customers`).

**Nodes Involved:**
- `Manual Trigger - Run Setup Once`
- `Postgres - Create CDC Objects`

**Node Details:**
- **Manual Trigger - Run Setup Once**
  - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` / Workflow entry point for manual execution.
  - *Configuration Choices:* Default manual initialization settings.
  - *Input / Output:* No input; outputs a single manual trigger execution signal to PostgreSQL.
  - *Failure Types:* None (manual execution).
- **Postgres - Create CDC Objects**
  - *Type & Technical Role:* `n8n-nodes-base.postgres` / Database execution node.
  - *Configuration Choices:* Executes a raw SQL DDL script creating `cdc_outbox`, indices `idx_cdc_outbox_pick` and `idx_cdc_outbox_row`, PL/pgSQL function `cdc_capture()`, and creates a row-level trigger on `public.customers`.
  - *Input / Output:* Input from Manual Trigger; outputs execution confirmation (`cdc_ready`).
  - *Failure Types:* Database permission errors, connection timeouts, or missing target source tables (`public.customers`). Requires standard PostgreSQL credentials.

---

#### 1.2 Poll & Claim Changes

**Overview:**  
Triggers every minute, establishes synchronization parameters, and atomically claims a batch of unprocessed or retryable outbox rows using row-level locks and strict chronological ordering.

**Nodes Involved:**
- `Schedule Trigger - Every Minute`
- `Set Sync Config`
- `Postgres - Claim Outbox Batch`
- `Events Claimed?`

**Node Details:**
- **Schedule Trigger - Every Minute**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` / Recurring time-based trigger.
  - *Configuration Choices:* Interval set to `1` minute.
  - *Input / Output:* No input; outputs execution pulse every minute.
- **Set Sync Config**
  - *Type & Technical Role:* `n8n-nodes-base.set` / Data transformer node.
  - *Configuration Choices:* Defines environment parameters such as `batchSize` (20), `leaseSeconds` (120), `maxAttempts` (5), `backoffBaseSeconds` (30), `backoffMaxSeconds` (3600), `enabledTargets` (`hubspot,mailchimp,slack`), Mailchimp credentials metadata, and Slack webhook URLs.
  - *Input / Output:* Input from Schedule Trigger; outputs global configuration properties.
- **Postgres - Claim Outbox Batch**
  - *Type & Technical Role:* `n8n-nodes-base.postgres` / Database execution node.
  - *Configuration Choices:* Executes a CTE query utilizing `FOR UPDATE SKIP LOCKED` to lock and claim up to `batchSize` records whose status is `pending`/`retry` (with past or current `next_attempt_at`) or whose processing lease has expired (`locked_until < NOW()`). Preserves ordering via ID checks.
  - *Key Expressions:* Query replacement bindings: `={{ [ $json.batchSize, $json.leaseSeconds ] }}`.
  - *Input / Output:* Input from Set Sync Config; outputs claimed outbox rows with MD5-hashed emails.
  - *Failure Types:* Database connection timeouts, syntax errors in dynamic limits.
- **Events Claimed?**
  - *Type & Technical Role:* `n8n-nodes-base.if` / Conditional router.
  - *Configuration Choices:* Evaluates whether returned records contain valid IDs.
  - *Key Expressions:* `={{ $json.id !== undefined }}`.
  - *Input / Output:* Input from Postgres Claim Node; outputs to Plan Deliveries if true (or stops execution if empty).

---

#### 1.3 Plan Fan-Out

**Overview:**  
Processes claimed database rows, sorting them chronologically and generating individual downstream SaaS integration tasks while filtering out targets that have already successfully processed the event.

**Nodes Involved:**
- `Code - Plan Deliveries`

**Node Details:**
- **Code - Plan Deliveries**
  - *Type & Technical Role:* `n8n-nodes-base.code` / JavaScript data processing node.
  - *Configuration Choices:* Parses event payloads, evaluates `enabledTargets`, inspects `delivered_targets` array strings (handling exclusion markers starting with `!`), and maps operations to target specific payloads (HubSpot batch upsert, Mailchimp member sync/unsubscribe, or Slack alerts).
  - *Input / Output:* Input from Events Claimed?; outputs an array of structured flat tasks (`eventId`, `attempts`, `op`, `target`, `request`).
  - *Failure Types:* JSON parsing errors on malformed payloads or schema validation exceptions.

---

#### 1.4 Deliver to SaaS Targets

**Overview:**  
Iterates over planned integration tasks sequentially, routes each request to the corresponding SaaS platform endpoint via HTTP, handles custom response parsing, and records outcomes.

**Nodes Involved:**
- `Loop Over Deliveries`
- `Switch - Route By Target`
- `HTTP - HubSpot Upsert Contact`
- `HTTP - Mailchimp Sync Member`
- `HTTP - Slack Change Notification`
- `NoOp - Nothing To Deliver`
- `Code - Record Delivery Result`

**Node Details:**
- **Loop Over Deliveries**
  - *Type & Technical Role:* `n8n-nodes-base.splitInBatches` / Batch loop controller.
  - *Configuration Choices:* Processes input items in batches (one task at a time) to respect rate limits.
  - *Input / Output:* Input from Plan Deliveries; outputs items individually to Switch node or loops back from result recording.
- **Switch - Route By Target**
  - *Type & Technical Role:* `n8n-nodes-base.switch` / Multi-way router.
  - *Configuration Choices:* Evaluates `$json.target` against rules: `hubspot`, `mailchimp`, `slack`, with a fallback for extra/empty values.
  - *Key Expressions:* `={{ $json.target }}`.
  - *Input / Output:* Input from Loop Over Deliveries; routes execution to target-specific HTTP nodes.
- **HTTP - HubSpot Upsert Contact**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` / API integration node.
  - *Configuration Choices:* Method and URL mapped dynamically from request objects. Configured to never error (`neverError: true`, full response enabled) with a 30-second timeout. Uses Generic HTTP Header Authentication (`Bearer <token>`).
  - *Key Expressions:* URL: `={{ $json.request.url }}`, Method: `={{ $json.request.method }}`, Body: `={{ JSON.stringify($json.request.body) }}`.
  - *Input / Output:* Input from Switch; outputs API response data to delivery result logging.
  - *Failure Types:* HTTP 4xx/5xx responses, invalid Bearer tokens, network timeouts.
- **HTTP - Mailchimp Sync Member**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` / API integration node.
  - *Configuration Choices:* Dynamically configured method/URL for Mailchimp API v3. Never errors out on failure codes, 30-second timeout. Uses Generic HTTP Basic Authentication.
  - *Key Expressions:* URL: `={{ $json.request.url }}`, Method: `={{ $json.request.method }}`, Body: `={{ JSON.stringify($json.request.body) }}`.
  - *Input / Output:* Input from Switch; outputs response data.
  - *Failure Types:* Invalid API keys, incorrect data center prefixes, list ID mismatches.
- **HTTP - Slack Change Notification**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` / Webhook integration node.
  - *Configuration Choices:* Posts change notifications to Slack webhooks. Never errors, 30-second timeout.
  - *Key Expressions:* URL: `={{ $json.request.url }}`, Method: `={{ $json.request.method }}`, Body: `={{ JSON.stringify($json.request.body) }}`.
  - *Input / Output:* Input from Switch; outputs response data.
  - *Failure Types:* Invalid incoming webhook URLs.
- **NoOp - Nothing To Deliver**
  - *Type & Technical Role:* `n8n-nodes-base.noOp` / Pass-through placeholder.
  - *Configuration Choices:* Default pass-through for events with no actionable targets.
  - *Input / Output:* Input from Switch fallback; outputs to result recorder.
- **Code - Record Delivery Result**
  - *Type & Technical Role:* `n8n-nodes-base.code` / JavaScript data evaluator.
  - *Configuration Choices:* Evaluates HTTP status codes (e.g., treating Mailchimp 404 on delete as success). Identifies permanent errors (4xx excluding 408 and 429) versus temporary failures.
  - *Input / Output:* Input from HTTP target nodes; outputs status object (`eventId`, `target`, `ok`, `permanent`, `error`, `statusCode`) back into the iteration loop.

---

#### 1.5 Finalize Events

**Overview:**  
Aggregates delivery outcomes for each event batch, computes exponential backoff retry intervals for temporary failures, marks permanent errors, and updates the event states in PostgreSQL.

**Nodes Involved:**
- `Code - Finalize Events`
- `Postgres - Save Event Status`

**Node Details:**
- **Code - Finalize Events**
  - *Type & Technical Role:* `n8n-nodes-base.code` / JavaScript state aggregator.
  - *Configuration Choices:* Compares execution results against original outbox claims. Determines if events should transition to `delivered`, `retry` (with randomized exponential backoff delay), or `dead` (if max attempts are reached or permanent errors occur).
  - *Input / Output:* Input from Loop Over Deliveries completion; outputs status update payloads.
- **Postgres - Save Event Status**
  - *Type & Technical Role:* `n8n-nodes-base.postgres` / Database execution node.
  - *Configuration Choices:* Executes an update statement targeting `cdc_outbox` using a strict fencing check (`status = 'processing' AND attempts = :attempts`) to prevent race conditions.
  - *Key Expressions:* Query replacement bindings: `={{ [ $json.id, $json.status, $json.deliveredTargets, $json.lastError, $json.delaySeconds, $json.attempts ] }}`.
  - *Input / Output:* Input from Finalize Events code node; outputs updated event IDs and statuses.
  - *Failure Types:* Database connection losses, query timeout errors.

---

#### 1.6 Report & Alert

**Overview:**  
Compiles a run summary of the synchronization cycle and dispatches a Slack alert if any events have fallen into the dead-letter queue.

**Nodes Involved:**
- `Code - Build Run Summary`
- `Any Dead Letters?`
- `HTTP - Slack Dead Letter Alert`

**Node Details:**
- **Code - Build Run Summary**
  - *Type & Technical Role:* `n8n-nodes-base.code` / Statistics builder.
  - *Configuration Choices:* Summarizes counts for delivered, retry, and dead events, creating a formatted text message for alerting.
  - *Input / Output:* Input from Save Event Status; outputs run statistics and `hasDead` boolean flag.
- **Any Dead Letters?**
  - *Type & Technical Role:* `n8n-nodes-base.if` / Conditional router.
  - *Configuration Choices:* Evaluates whether `hasDead` is true.
  - *Key Expressions:* `={{ $json.hasDead }}`.
  - *Input / Output:* Input from Run Summary; outputs to Slack alert node if true.
- **HTTP - Slack Dead Letter Alert**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` / Alert webhook node.
  - *Configuration Choices:* Posts dead-letter warnings to the configured Slack alert webhook URL with a 15-second timeout.
  - *Key Expressions:* URL: `={{ $('Set Sync Config').first().json.slackAlertUrl }}`, Body: `={{ JSON.stringify({ text: $json.text }) }}`.
  - *Input / Output:* Input from Any Dead Letters?; outputs API response.
  - *Failure Types:* Invalid webhook configurations or rate-limiting by Slack.

---

#### 1.7 Requeue API

**Overview:**  
Exposes a secure webhook endpoint (`POST /cdc-requeue`) that validates incoming dead-letter event IDs, resets their database status back to pending while preserving successful target histories, and returns the requeue count.

**Nodes Involved:**
- `Webhook - Requeue Dead Events`
- `Code - Validate Requeue Request`
- `Is Requeue Request Valid?`
- `Respond - Requeue Validation Error`
- `Postgres - Requeue Dead Events`
- `Respond - Requeue Result`

**Node Details:**
- **Webhook - Requeue Dead Events**
  - *Type & Technical Role:* `n8n-nodes-base.webhook` / HTTP webhook trigger endpoint.
  - *Configuration Choices:* Listens on path `cdc-requeue` for `POST` requests, using explicit response nodes.
  - *Input / Output:* External HTTP POST request; outputs raw webhook body payload.
- **Code - Validate Requeue Request**
  - *Type & Technical Role:* `n8n-nodes-base.code` / Request validator.
  - *Configuration Choices:* Verifies that request body `ids` is an array of 1 to 500 valid numeric strings.
  - *Input / Output:* Input from Webhook; outputs validation status and CSV id strings.
- **Is Requeue Request Valid?**
  - *Type & Technical Role:* `n8n-nodes-base.if` / Conditional router.
  - *Configuration Choices:* Evaluates `$json.valid`.
  - *Key Expressions:* `={{ $json.valid }}`.
  - *Input / Output:* Input from Validation Code; routes valid requests to database update and invalid requests to error response.
- **Respond - Requeue Validation Error**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` / HTTP response node.
  - *Configuration Choices:* Responds with HTTP status code `400` and a JSON error message.
  - *Input / Output:* Input from Validation IF; outputs HTTP response to caller.
- **Postgres - Requeue Dead Events**
  - *Type & Technical Role:* `n8n-nodes-base.postgres` / Database execution node.
  - *Configuration Choices:* Updates dead outbox records matching specified IDs back to `pending`, resets attempts to `0`, clears errors, and strips permanent target failure flags (`!`) from `delivered_targets`.
  - *Key Expressions:* Query replacement bindings: `={{ [ $json.idsCsv ] }}`.
  - *Input / Output:* Input from Validation IF; outputs updated row records.
  - *Failure Types:* Database connection drop or syntax error.
- **Respond - Requeue Result**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` / HTTP response node.
  - *Configuration Choices:* Responds with JSON containing the count of successfully requeued events (`requeued`).
  - *Input / Output:* Input from Postgres Requeue node; outputs final HTTP API response.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation container | None | None | CDC Pipeline: PostgreSQL to Multiple SaaS Platforms<br>### How it works... |
| `Note: 1. One-time setup` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## 1. One-time setup<br>Run manually once... |
| `Note: 2. Poll & claim changes` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## 2. Poll & claim changes<br>Every minute... |
| `Note: 3. Plan fan-out` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## 3. Plan fan-out<br>Turns each change into one delivery task... |
| `Note: 4. Deliver to SaaS targets` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## 4. Deliver to SaaS targets<br>Routes each task to HubSpot... |
| `Note: 5. Finalize events` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## 5. Finalize events<br>When all tasks are done... |
| `Note: 6. Report & alert` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## 6. Report & alert<br>Builds a run summary... |
| `Note: 7. Requeue API: validate` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## 7. Requeue API: validate<br>`POST /cdc-requeue`... |
| `Note: 8. Requeue API: reset & respond` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## 8. Requeue API: reset & respond<br>Resets dead events to pending... |
| `Manual Trigger - Run Setup Once` | `n8n-nodes-base.manualTrigger` | Manual setup entry point | None | `Postgres - Create CDC Objects` | |
| `Postgres - Create CDC Objects` | `n8n-nodes-base.postgres` | Create tables, functions, triggers | `Manual Trigger - Run Setup Once` | None | |
| `Schedule Trigger - Every Minute` | `n8n-nodes-base.scheduleTrigger` | 1-minute recurring timer | None | `Set Sync Config` | |
| `Set Sync Config` | `n8n-nodes-base.set` | Define global sync variables | `Schedule Trigger - Every Minute` | `Postgres - Claim Outbox Batch` | |
| `Postgres - Claim Outbox Batch` | `n8n-nodes-base.postgres` | Atomically claim batch using locks | `Set Sync Config` | `Events Claimed?` | |
| `Events Claimed?` | `n8n-nodes-base.if` | Check if batch contains events | `Postgres - Claim Outbox Batch` | `Code - Plan Deliveries` | |
| `Code - Plan Deliveries` | `n8n-nodes-base.code` | Expand changes into target tasks | `Events Claimed?` | `Loop Over Deliveries` | |
| `Loop Over Deliveries` | `n8n-nodes-base.splitInBatches` | Iterate tasks one-by-one | `Code - Plan Deliveries`<br>`Code - Record Delivery Result` | `Code - Finalize Events`<br>`Switch - Route By Target` | |
| `Switch - Route By Target` | `n8n-nodes-base.switch` | Route task by target platform | `Loop Over Deliveries` | `HTTP - HubSpot Upsert Contact`<br>`HTTP - Mailchimp Sync Member`<br>`HTTP - Slack Change Notification`<br>`NoOp - Nothing To Deliver` | |
| `HTTP - HubSpot Upsert Contact` | `n8n-nodes-base.httpRequest` | Upsert HubSpot contact | `Switch - Route By Target` | `Code - Record Delivery Result` | |
| `HTTP - Mailchimp Sync Member` | `n8n-nodes-base.httpRequest` | Sync or unsubscribe Mailchimp member | `Switch - Route By Target` | `Code - Record Delivery Result` | |
| `HTTP - Slack Change Notification` | `n8n-nodes-base.httpRequest` | Send Slack change notification | `Switch - Route By Target` | `Code - Record Delivery Result` | |
| `NoOp - Nothing To Deliver` | `n8n-nodes-base.noOp` | Pass-through for empty targets | `Switch - Route By Target` | `Code - Record Delivery Result` | |
| `Code - Record Delivery Result` | `n8n-nodes-base.code` | Evaluate HTTP response outcome | `HTTP - HubSpot Upsert Contact`<br>`HTTP - Mailchimp Sync Member`<br>`HTTP - Slack Change Notification`<br>`NoOp - Nothing To Deliver` | `Loop Over Deliveries` | |
| `Code - Finalize Events` | `n8n-nodes-base.code` | Calculate retries & backoff delays | `Loop Over Deliveries` | `Postgres - Save Event Status` | |
| `Postgres - Save Event Status` | `n8n-nodes-base.postgres` | Update event status in database | `Code - Finalize Events` | `Code - Build Run Summary` | |
| `Code - Build Run Summary` | `n8n-nodes-base.code` | Generate run statistics & alerts | `Postgres - Save Event Status` | `Any Dead Letters?` | |
| `Any Dead Letters?` | `n8n-nodes-base.if` | Check if dead-lettered events exist | `Code - Build Run Summary` | `HTTP - Slack Dead Letter Alert` | |
| `HTTP - Slack Dead Letter Alert` | `n8n-nodes-base.httpRequest` | Send Slack dead-letter alert | `Any Dead Letters?` | None | |
| `Webhook - Requeue Dead Events` | `n8n-nodes-base.webhook` | API endpoint for requeuing | None | `Code - Validate Requeue Request` | |
| `Code - Validate Requeue Request` | `n8n-nodes-base.code` | Validate incoming IDs array | `Webhook - Requeue Dead Events` | `Is Requeue Request Valid?` | |
| `Is Requeue Request Valid?` | `n8n-nodes-base.if` | Verify requeue request validity | `Code - Validate Requeue Request` | `Postgres - Requeue Dead Events`<br>`Respond - Requeue Validation Error` | |
| `Respond - Requeue Validation Error` | `n8n-nodes-base.respondToWebhook` | Return 400 validation error | `Is Requeue Request Valid?` | None | |
| `Postgres - Requeue Dead Events` | `n8n-nodes-base.postgres` | Reset dead events to pending | `Is Requeue Request Valid?` | `Respond - Requeue Result` | |
| `Respond - Requeue Result` | `n8n-nodes-base.respondToWebhook` | Return requeued count JSON | `Postgres - Requeue Dead Events` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these manual steps to rebuild the workflow in n8n from scratch:

#### Step 1: Database Setup
1. Create a **Manual Trigger** node named `Manual Trigger - Run Setup Once`.
2. Create a **PostgreSQL** node named `Postgres - Create CDC Objects` and connect it to the manual trigger.
3. Configure the PostgreSQL node with valid database credentials and set the operation to **Execute Query**. Paste the SQL DDL script to create the `cdc_outbox` table, indices (`idx_cdc_outbox_pick`, `idx_cdc_outbox_row`), the `cdc_capture()` function, and the trigger on `public.customers`.

#### Step 2: Polling & Claiming Logic
1. Create a **Schedule Trigger** node named `Schedule Trigger - Every Minute` configured with an interval of `1` minute.
2. Create a **Set** node named `Set Sync Config` connected after the schedule trigger. Configure assignment fields:
   - `batchSize` (Number: `20`)
   - `leaseSeconds` (Number: `120`)
   - `maxAttempts` (Number: `5`)
   - `backoffBaseSeconds` (Number: `30`)
   - `backoffMaxSeconds` (Number: `3600`)
   - `enabledTargets` (String: `hubspot,mailchimp,slack`)
   - `mailchimpDc` (String: e.g., `us1`)
   - `mailchimpListId` (String: your Mailchimp list ID)
   - `slackWebhookUrl` (String: your Slack incoming webhook URL)
   - `slackAlertUrl` (String: your Slack alert webhook URL)
3. Create a **PostgreSQL** node named `Postgres - Claim Outbox Batch` connected after `Set Sync Config`. Set operation to **Execute Query**, using the outbox leasing query with query replacements: `={{ [ $json.batchSize, $json.leaseSeconds ] }}`.
4. Create an **If** node named `Events Claimed?` connected after the claim node with condition: `={{ $json.id !== undefined }}`.

#### Step 3: Fan-Out & Task Planning
1. Create a **Code** node named `Code - Plan Deliveries` connected to the "True" branch of `Events Claimed?`. Paste the JavaScript logic that reads configuration settings, sorts input rows chronologically, filters completed targets using `delivered_targets`, and constructs task objects for HubSpot, Mailchimp, and Slack.

#### Step 4: Iteration & Delivery Execution
1. Create a **Split In Batches** node named `Loop Over Deliveries` connected after `Code - Plan Deliveries`.
2. Create a **Switch** node named `Switch - Route By Target` connected to output index 0 of the loop node. Set rules to route items where `target` equals `hubspot`, `mailchimp`, or `slack`.
3. Create three **HTTP Request** nodes connected to the switch outputs:
   - **HTTP - HubSpot Upsert Contact:** Method and URL dynamic (`={{ $json.request.url }}`), JSON body `={{ JSON.stringify($json.request.body) }}`, 30s timeout, response options set to "Never Error" and "Full Response", configured with **Generic Credential Type: HTTP Header Auth** (`Authorization: Bearer <token>`).
   - **HTTP - Mailchimp Sync Member:** Method and URL dynamic, JSON body dynamic, 30s timeout, response options set to "Never Error" and "Full Response", configured with **Generic Credential Type: HTTP Basic Auth** (any username, API key as password).
   - **HTTP - Slack Change Notification:** Method and URL dynamic, JSON body dynamic, 30s timeout, response options set to "Never Error" and "Full Response".
4. Create a **NoOp** node named `NoOp - Nothing To Deliver` connected to the fallback output of the Switch node.
5. Create a **Code** node named `Code - Record Delivery Result` connected from all three HTTP nodes and the NoOp node. Paste the result evaluation script to verify HTTP status codes and permanent failures.
6. Connect the output of `Code - Record Delivery Result` back into the input of `Loop Over Deliveries` to continue the loop.

#### Step 5: Finalization & Alerting
1. Connect output index 1 (finish loop signal) of `Loop Over Deliveries` to a **Code** node named `Code - Finalize Events`.
2. Create a **PostgreSQL** node named `Postgres - Save Event Status` connected after `Code - Finalize Events`. Set operation to **Execute Query** with query replacements: `={{ [ $json.id, $json.status, $json.deliveredTargets, $json.lastError, $json.delaySeconds, $json.attempts ] }}`.
3. Create a **Code** node named `Code - Build Run Summary` connected after `Postgres - Save Event Status`.
4. Create an **If** node named `Any Dead Letters?` connected after the run summary with condition: `={{ $json.hasDead }}`.
5. Create an **HTTP Request** node named `HTTP - Slack Dead Letter Alert` connected to the "True" branch of `Any Dead Letters?`, configured to send a POST request with `={{ JSON.stringify({ text: $json.text }) }}` to the Slack alert webhook.

#### Step 6: Requeue API Implementation
1. Create a **Webhook** node named `Webhook - Requeue Dead Events`. Set HTTP Method to `POST` and path to `cdc-requeue`.
2. Create a **Code** node named `Code - Validate Requeue Request` connected after the webhook.
3. Create an **If** node named `Is Requeue Request Valid?` connected after validation with condition: `={{ $json.valid }}`.
4. Create a **Respond to Webhook** node named `Respond - Requeue Validation Error` connected to the "False" branch, configured with Response Code `400` and JSON body `={{ JSON.stringify({ error: $json.validationError }) }}`.
5. Create a **PostgreSQL** node named `Postgres - Requeue Dead Events` connected to the "True" branch, configured to execute the requeue SQL update query with query replacement `={{ [ $json.idsCsv ] }}`.
6. Create a **Respond to Webhook** node named `Respond - Requeue Result` connected after the PostgreSQL requeue node, configured to return JSON: `={{ JSON.stringify({ requeued: $('Postgres - Requeue Dead Events').all().filter(i => i.json.id).length }) }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| CDC Pipeline: PostgreSQL to Multiple SaaS Platforms | Implements transactional outbox pattern to guarantee at-least-once delivery with idempotent upserts across HubSpot, Mailchimp, and Slack. |