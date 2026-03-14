Send multi-tenant reminders via Telegram from webhooks and schedule with Postgres logging

https://n8nworkflows.xyz/workflows/send-multi-tenant-reminders-via-telegram-from-webhooks-and-schedule-with-postgres-logging-13877


# Send multi-tenant reminders via Telegram from webhooks and schedule with Postgres logging

# 1. Workflow Overview

This workflow implements a **multi-tenant reminder delivery system** using **n8n**, **PostgreSQL**, **Telegram**, a **webhook trigger**, a **schedule trigger**, and a built-in **registration form**.

Its main purpose is to let multiple clients/tenants define reminder rules and receive outbound reminders through Telegram. The workflow supports three entry points:

1. **Webhook trigger** for immediate reminder sending from external systems
2. **Schedule trigger** for periodic checks of due events
3. **Form trigger** for self-service tenant registration

The workflow is structured into the following logical blocks:

## 1.1 Foundation and Setup Notes
Provides internal documentation in sticky notes, including the overall behavior and the SQL schema required in PostgreSQL.

## 1.2 Webhook-Based Reminder Flow
Accepts incoming event payloads, validates tenant identity through the `x-tenant-token` header, loads the tenant’s active webhook rule, normalizes data, and prepares the outbound message.

## 1.3 Scheduled Reminder Flow
Runs every minute, queries PostgreSQL for events falling within each tenant’s configured reminder window, and forwards eligible events for message rendering.

## 1.4 Message Rendering and Channel Routing
Builds final message text from the configured template and event variables, then routes the item according to the configured channel. In the current implementation, only Telegram is actually handled.

## 1.5 Send and Logging
Sends the reminder through Telegram, records successful sends in an idempotency table, logs successful attempts, and logs delivery errors separately.

## 1.6 Tenant Registration Flow
Provides a native n8n form to create tenants and their initial reminder rule directly in PostgreSQL, without requiring an external backend.

---

# 2. Block-by-Block Analysis

## 2.1 Foundation and Setup Notes

### Overview
This block contains workflow-level documentation embedded as sticky notes. These notes explain the architecture, onboarding steps, database schema, and the function of each visual section.

### Nodes Involved
- Overview
- Database Schema
- Webhook flow group
- Schedule flow group
- Registration group
- Send and log group

### Node Details

#### Overview
- **Type and technical role:** Sticky Note; provides a high-level description of the system.
- **Configuration choices:** Contains setup steps, entry points, and operational summary.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky note only; no runtime dependency.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Database Schema
- **Type and technical role:** Sticky Note; documents required PostgreSQL tables.
- **Configuration choices:** Includes SQL definitions for:
  - `tenants`
  - `tenant_rules`
  - `tenant_events`
  - `tenant_reminders_sent`
  - `tenant_logs`
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** Runtime issues will occur if these tables are missing or incompatible.
- **Sub-workflow reference:** None.

#### Webhook flow group
- **Type and technical role:** Sticky Note; labels the webhook section.
- **Configuration choices:** Documents token validation and immediate reminder behavior.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Schedule flow group
- **Type and technical role:** Sticky Note; labels the scheduled section.
- **Configuration choices:** Documents one-minute polling and idempotency behavior.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Registration group
- **Type and technical role:** Sticky Note; labels the tenant registration section.
- **Configuration choices:** Notes that registration is exposed at `/form/multi-tenant-register`.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Send and log group
- **Type and technical role:** Sticky Note; labels sending and logging section.
- **Configuration choices:** Explains channel routing and how to extend to new channels.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Webhook-Based Reminder Flow

### Overview
This block receives events from external systems through an HTTP POST webhook. It validates the tenant token, looks up the corresponding tenant and active webhook rule in PostgreSQL, and reshapes the data into a common format for downstream rendering.

### Nodes Involved
- Receive event via webhook
- Validate tenant token
- Fetch tenant config
- Map webhook fields

### Node Details

#### Receive event via webhook
- **Type and technical role:** Webhook trigger; entry point for immediate reminders.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `multi-tenant-webhook`
- **Key expressions or variables used:** Expects request headers and body.
- **Input and output connections:** Outputs to **Validate tenant token**.
- **Version-specific requirements:** Uses webhook node version `2.1`.
- **Edge cases or potential failure types:**
  - Requests to the wrong path or method will fail at HTTP level.
  - Invalid or missing JSON body may cause downstream expression/code issues.
  - If the workflow is inactive, production webhook behavior depends on n8n environment and test mode.
- **Sub-workflow reference:** None.

#### Validate tenant token
- **Type and technical role:** Code node; validates auth header and extracts inbound event fields.
- **Configuration choices:**
  - Reads `headers['x-tenant-token']`
  - Throws error if header is missing
  - Returns:
    - `token`
    - `entity_name`
    - `event_datetime`
    - `extra_data` defaulting to `{}` if absent
- **Key expressions or variables used:**
  - `$input.item.json.headers['x-tenant-token']`
  - `$input.item.json.body.entity_name`
  - `$input.item.json.body.event_datetime`
  - `$input.item.json.body.extra_data || {}`
- **Input and output connections:** Input from **Receive event via webhook**; output to **Fetch tenant config**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Missing header throws hard error.
  - Missing body fields are not fully validated; malformed `event_datetime` may fail later.
  - Header casing behavior depends on n8n’s normalized webhook headers.
- **Sub-workflow reference:** None.

#### Fetch tenant config
- **Type and technical role:** PostgreSQL query node; loads tenant and webhook rule by API token.
- **Configuration choices:**
  - Executes a SQL `SELECT` joining `tenants` and `tenant_rules`
  - Filters:
    - `t.api_token = '{{ $json.token }}'`
    - `r.trigger_type = 'webhook'`
    - `r.active = true`
    - `t.active = true`
- **Key expressions or variables used:**
  - `{{ $json.token }}`
- **Input and output connections:** Input from **Validate tenant token**; output to **Map webhook fields**.
- **Version-specific requirements:** Postgres node version `2.6`; requires valid PostgreSQL credentials.
- **Edge cases or potential failure types:**
  - No matching tenant/rule returns zero items, which silently stops this branch.
  - SQL injection risk exists because token is interpolated directly into SQL.
  - Auth/network/database permission failures will stop execution.
  - If multiple active webhook rules exist for one tenant token, multiple reminder attempts may result.
- **Sub-workflow reference:** None.

#### Map webhook fields
- **Type and technical role:** Set node; normalizes webhook query results and event payload into the common reminder structure.
- **Configuration choices:**
  - Creates:
    - `tenants_id`
    - `rule_id`
    - `channel`
    - `chat_id`
    - `message_template`
    - `entity_name`
    - `event_datetime`
    - `extra_data`
    - `channel_config`
  - Mixes values from the SQL result and the earlier code node.
- **Key expressions or variables used:**
  - `{{ $json.tenant_id }}`
  - `{{ $('Validate tenant token').item.json.entity_name }}`
  - `{{ $('Validate tenant token').item.json.event_datetime }}`
  - `{{ $('Validate tenant token').item.json.extra_data }}`
  - `{{ $json.channel_config }}`
- **Input and output connections:** Input from **Fetch tenant config**; output to **Render Template**.
- **Version-specific requirements:** Set node version `3.4`.
- **Edge cases or potential failure types:**
  - If `Fetch tenant config` returns no rows, this node never runs.
  - If `channel_config` is malformed or missing `chat_id`, Telegram sending will fail later.
  - `chat_id` is mapped but not used downstream because the Telegram node reads from `channel_config.chat_id`.
- **Sub-workflow reference:** None.

---

## 2.3 Scheduled Reminder Flow

### Overview
This block polls the database every minute for pending events whose reminder window has been reached. It avoids duplicates by excluding reminders already recorded in `tenant_reminders_sent`.

### Nodes Involved
- Every minute - check due events
- Fetch events due for reminder

### Node Details

#### Every minute - check due events
- **Type and technical role:** Schedule trigger; starts periodic reminder evaluation.
- **Configuration choices:** Runs every minute.
- **Key expressions or variables used:** None.
- **Input and output connections:** Outputs to **Fetch events due for reminder**.
- **Version-specific requirements:** Schedule Trigger version `1.3`.
- **Edge cases or potential failure types:**
  - If workflow is inactive, no scheduled execution occurs.
  - High event volume every minute may create DB load.
- **Sub-workflow reference:** None.

#### Fetch events due for reminder
- **Type and technical role:** PostgreSQL query node; returns all pending scheduled reminders due now.
- **Configuration choices:**
  - Joins:
    - `tenant_rules r`
    - `tenants t`
    - `tenant_events e`
  - Filters:
    - `r.trigger_type = 'schedule'`
    - `r.active = true`
    - `t.active = true`
    - `e.status = 'pending'`
  - Uses a reminder window:
    - `event_datetime BETWEEN NOW() + timing_minutes_before - 5 minutes`
    - `AND NOW() + timing_minutes_before + 5 minutes`
  - Excludes rows already present in `tenant_reminders_sent`
- **Key expressions or variables used:** SQL-only; no n8n expression interpolation.
- **Input and output connections:** Input from **Every minute - check due events**; output to **Render Template**.
- **Version-specific requirements:** Postgres node version `2.6`; valid Postgres credentials required.
- **Edge cases or potential failure types:**
  - Query window may produce duplicates across overlapping schedule runs if idempotency insert fails.
  - A ±5 minute window is broad relative to a 1-minute schedule; this is intentional but makes successful logging/idempotency essential.
  - If `timing_minutes_before` is null, behavior may be undefined or filtered out by SQL arithmetic.
  - Timezone handling depends on DB server timezone and inserted event values.
- **Sub-workflow reference:** None.

---

## 2.4 Message Rendering and Channel Routing

### Overview
This block transforms event and tenant data into the final user-facing message. It computes formatted date/time values, substitutes placeholders in the tenant’s message template, and branches according to delivery channel.

### Nodes Involved
- Render Template
- Route by channel type
- No events due - skip

### Node Details

#### Render Template
- **Type and technical role:** Code node; formats event variables and resolves template placeholders.
- **Configuration choices:**
  - Executes once per item
  - Parses `event_datetime`
  - Formats:
    - `event_time` in `en-US`, timezone `America/Sao_Paulo`
    - `event_date` in `en-US`, timezone `America/Sao_Paulo`
  - Builds variables object from:
    - `entity_name`
    - `entity_contact`
    - `tenants_id`
    - `event_time`
    - `event_date`
    - spread of `extra_data`
  - Replaces `{{variable}}` placeholders using `replaceAll`
  - Returns:
    - `event_id`
    - `rule_id`
    - `tenants_id`
    - `channel`
    - `channel_config`
    - `message`
- **Key expressions or variables used:** JavaScript with `$input.item.json`.
- **Input and output connections:** Inputs from **Map webhook fields** and **Fetch events due for reminder**; output to **Route by channel type**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Invalid `event_datetime` can produce `Invalid Date`.
  - `replaceAll` requires modern JS runtime support, normally available in current n8n versions.
  - Missing placeholders remain unreplaced.
  - Non-string template values or unexpected object values in `extra_data` may render poorly.
  - The formatting timezone is hardcoded to `America/Sao_Paulo`, which may not fit all tenants.
- **Sub-workflow reference:** None.

#### Route by channel type
- **Type and technical role:** IF node; dispatches items by configured channel.
- **Configuration choices:**
  - Condition: `{{$json.channel}} === 'telegram'`
  - True branch goes to Telegram send
  - False branch goes to a no-op skip node
- **Key expressions or variables used:**
  - `={{ $json.channel }}`
- **Input and output connections:** Input from **Render Template**; outputs to:
  - **Send reminder via Telegram** on true
  - **No events due - skip** on false
- **Version-specific requirements:** IF node version `2.2`.
- **Edge cases or potential failure types:**
  - Any non-`telegram` channel is silently skipped.
  - Despite registration form offering `whatsapp` and `email`, they are not implemented here.
- **Sub-workflow reference:** None.

#### No events due - skip
- **Type and technical role:** No Operation node; dead-end branch for unsupported channel types.
- **Configuration choices:** None.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from false output of **Route by channel type**; no outputs.
- **Version-specific requirements:** NoOp version `1`.
- **Edge cases or potential failure types:**
  - Misleading name: it also handles unsupported channels, not only “no events due”.
- **Sub-workflow reference:** None.

---

## 2.5 Send and Logging

### Overview
This block sends the rendered reminder through Telegram and writes both success and error outcomes to PostgreSQL. It also inserts a row into `tenant_reminders_sent` for idempotency after successful delivery.

### Nodes Involved
- Send reminder via Telegram
- Mark reminder as sent
- Log success to database
- Log error to database

### Node Details

#### Send reminder via Telegram
- **Type and technical role:** Telegram node; sends the outbound reminder message.
- **Configuration choices:**
  - Text: `={{ $json.message }}`
  - Chat ID: `={{ $json.channel_config.chat_id }}`
  - Attribution disabled
  - `onError: continueErrorOutput`
  - `retryOnFail: false`
- **Key expressions or variables used:**
  - `{{ $json.message }}`
  - `{{ $json.channel_config.chat_id }}`
- **Input and output connections:**
  - Input from **Route by channel type**
  - Success output to:
    - **Mark reminder as sent**
    - **Log success to database**
  - Error output to:
    - **Log error to database**
- **Version-specific requirements:** Telegram node version `1.2`; valid Telegram API credentials required.
- **Edge cases or potential failure types:**
  - Invalid bot credentials
  - Invalid or unauthorized `chat_id`
  - Telegram rate limits
  - Message formatting/content restrictions
  - Network/API downtime
- **Sub-workflow reference:** None.

#### Mark reminder as sent
- **Type and technical role:** PostgreSQL query node; records a successful send for deduplication.
- **Configuration choices:**
  - Inserts into `tenant_reminders_sent`:
    - `tenant_id`
    - `rule_id`
    - `event_id`
- **Key expressions or variables used:**
  - `{{ $('Render Template').item.json.tenants_id }}`
  - `{{ $('Render Template').item.json.rule_id }}`
  - `{{ $('Render Template').item.json.event_id || null }}`
- **Input and output connections:** Input from successful output of **Send reminder via Telegram**; no downstream nodes.
- **Version-specific requirements:** Postgres node version `2.6`.
- **Edge cases or potential failure types:**
  - If this insert fails but the Telegram send succeeded, future schedule runs may resend the same reminder.
  - No uniqueness constraint is defined in schema, so duplicate inserts are possible under concurrency.
  - Webhook-triggered reminders may insert `null` event IDs.
- **Sub-workflow reference:** None.

#### Log success to database
- **Type and technical role:** PostgreSQL query node; stores a successful send attempt in `tenant_logs`.
- **Configuration choices:**
  - Inserts:
    - `tenant_id`
    - `rule_id`
    - `event_id`
    - status `'success'`
    - `message_sent`
- **Key expressions or variables used:**
  - `{{ $('Render Template').item.json.tenants_id }}`
  - `{{ $('Render Template').item.json.rule_id }}`
  - `{{ $('Render Template').item.json.event_id || null }}`
  - `{{ $('Render Template').item.json.message }}`
- **Input and output connections:** Input from successful output of **Send reminder via Telegram**; no downstream nodes.
- **Version-specific requirements:** Postgres node version `2.6`.
- **Edge cases or potential failure types:**
  - Unescaped quotes inside message text may break the SQL statement because values are interpolated directly.
  - Logging failure does not undo message delivery.
- **Sub-workflow reference:** None.

#### Log error to database
- **Type and technical role:** PostgreSQL query node; records delivery errors.
- **Configuration choices:**
  - Inserts:
    - `tenant_id`
    - `rule_id`
    - `event_id`
    - status `'error'`
    - `error_message`
- **Key expressions or variables used:**
  - `{{ $json.tenants_id }}`
  - `{{ $json.rule_id }}`
  - `{{ $json.event_id }}`
  - `{{ $json.error }}`
- **Input and output connections:** Input from error output of **Send reminder via Telegram**; no downstream nodes.
- **Version-specific requirements:** Postgres node version `2.6`.
- **Edge cases or potential failure types:**
  - Assumes the Telegram error output preserves `tenants_id`, `rule_id`, and `event_id`.
  - If error text contains quotes, the SQL insert may fail.
  - Logging failure can hide operational issues.
- **Sub-workflow reference:** None.

---

## 2.6 Tenant Registration Flow

### Overview
This block provides a built-in n8n form for onboarding new tenants and creating an initial reminder rule. It stores tenant settings and rule configuration in PostgreSQL, then returns a success confirmation page.

### Nodes Involved
- Tenant registration form
- Organize form data
- Insert tenant
- Insert tenant rule
- Show registration success

### Node Details

#### Tenant registration form
- **Type and technical role:** Form Trigger; public entry point for tenant self-registration.
- **Configuration choices:**
  - Title: `Register New Tenant`
  - Fields:
    - Client name
    - Channel (`telegram`, `whatsapp`, `email`)
    - Chat ID / Contact
    - API Token
    - Trigger type (`schedule`, `webhook`)
    - Minutes before event
    - Message template
  - Response mode: `lastNode`
- **Key expressions or variables used:** Form field labels are used as keys downstream.
- **Input and output connections:** Outputs to **Organize form data**.
- **Version-specific requirements:** Form Trigger version `2.3`.
- **Edge cases or potential failure types:**
  - Channel options exceed actual sending support.
  - API Token is not marked required, but webhook flow depends on it.
  - `Minutes before event` may be empty even for schedule rules.
- **Sub-workflow reference:** None.

#### Organize form data
- **Type and technical role:** Code node; maps form values into DB-ready fields.
- **Configuration choices:**
  - Converts contact into JSON string:
    - `{"chat_id": "<value>"}`
  - Returns:
    - `name`
    - `channel`
    - `channel_config`
    - `api_token`
    - `trigger_type`
    - `timing_minutes_before`
    - `message_template`
- **Key expressions or variables used:** Accesses fields by their form labels such as:
  - `item['Client name']`
  - `item['Chat ID / Contact']`
  - `item['Trigger type']`
- **Input and output connections:** Input from **Tenant registration form**; output to **Insert tenant**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Stores `channel_config` as a JSON string, relying on PostgreSQL JSONB coercion during insert.
  - Empty API token can create a webhook tenant that cannot authenticate meaningfully.
- **Sub-workflow reference:** None.

#### Insert tenant
- **Type and technical role:** PostgreSQL query node; inserts a tenant and returns its generated ID.
- **Configuration choices:**
  - Inserts:
    - `name`
    - `channel`
    - `channel_config`
    - `api_token`
    - `active = true`
  - Uses `RETURNING id`
- **Key expressions or variables used:**
  - `{{ $json.name }}`
  - `{{ $json.channel }}`
  - `{{ $json.channel_config }}`
  - `{{ $json.api_token }}`
- **Input and output connections:** Input from **Organize form data**; output to **Insert tenant rule**.
- **Version-specific requirements:** Postgres node version `2.6`.
- **Edge cases or potential failure types:**
  - Direct SQL interpolation creates quote-breaking and injection risk.
  - Invalid JSON string may fail to cast into JSONB.
  - No uniqueness rule on token or tenant name is enforced in schema.
- **Sub-workflow reference:** None.

#### Insert tenant rule
- **Type and technical role:** PostgreSQL query node; creates the initial rule for the newly created tenant.
- **Configuration choices:**
  - Inserts:
    - `tenant_id` from previous node
    - `trigger_type`
    - `timing_minutes_before`
    - `message_template`
    - `active = true`
- **Key expressions or variables used:**
  - `{{ $json.id }}`
  - `{{ $('Organize form data').item.json.trigger_type }}`
  - `{{ $('Organize form data').item.json.timing_minutes_before || 'NULL' }}`
  - `{{ $('Organize form data').item.json.message_template }}`
- **Input and output connections:** Input from **Insert tenant**; output to **Show registration success**.
- **Version-specific requirements:** Postgres node version `2.6`.
- **Edge cases or potential failure types:**
  - SQL injection/quote issues possible in `message_template`.
  - `timing_minutes_before` may be null for scheduled rules, which may not be operationally valid.
- **Sub-workflow reference:** None.

#### Show registration success
- **Type and technical role:** Form node in completion mode; returns a success page to the registrant.
- **Configuration choices:**
  - Completion title: `Tenant registered! ID: {{ $('Insert tenant').item.json.id }}`
- **Key expressions or variables used:**
  - `{{ $('Insert tenant').item.json.id }}`
- **Input and output connections:** Input from **Insert tenant rule**; terminal node for form response.
- **Version-specific requirements:** Form node version `2.3`.
- **Edge cases or potential failure types:**
  - If insert succeeds but ID extraction fails, confirmation text may be wrong.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | High-level documentation and setup guidance |  |  | ## Multi-tenant reminder system<br>## How it works<br>This workflow has **3 entry points**:<br><br>**1. Webhook trigger** — an external system sends an event (appointment, deadline, order) and the reminder fires immediately based on the tenant's configured rules.<br><br>**2. Schedule trigger** — runs every minute, queries events approaching their deadline window, and sends reminders automatically. Uses idempotency to ensure the same reminder is never sent twice.<br><br>**3. Registration form** — a built-in n8n form to register new tenants with their channel config and message template. No external backend needed.<br><br>After the template is rendered with the event variables, the workflow routes to the correct channel (currently Telegram, easily extendable to WhatsApp, email, etc.). Every send — success or error — is logged to the database.<br><br>## Setup steps<br>1. Add your **PostgreSQL credentials** to all Postgres nodes (~2 min)<br>2. Add your **Telegram credentials** to the Send Message node (~2 min)<br>3. Create the required tables using the SQL schema in the sticky note below (~10 min)<br>4. Register your first tenant at `/form/multi-tenant-register`<br>5. Send events via `POST /webhook/multi-tenant-webhook` with `x-tenant-token` header |
| Database Schema | Sticky Note | Documents required PostgreSQL tables |  |  | ## Required database schema<br><br>```sql<br>CREATE TABLE tenants (<br>  id SERIAL PRIMARY KEY,<br>  name VARCHAR(255) NOT NULL,<br>  channel VARCHAR(50) NOT NULL,<br>  channel_config JSONB NOT NULL,<br>  api_token VARCHAR(255),<br>  active BOOLEAN DEFAULT true,<br>  created_at TIMESTAMP DEFAULT NOW()<br>);<br><br>CREATE TABLE tenant_rules (<br>  id SERIAL PRIMARY KEY,<br>  tenant_id INTEGER REFERENCES tenants(id),<br>  trigger_type VARCHAR(20) NOT NULL,<br>  timing_minutes_before INTEGER,<br>  message_template TEXT NOT NULL,<br>  active BOOLEAN DEFAULT true<br>);<br><br>CREATE TABLE tenant_events (<br>  id SERIAL PRIMARY KEY,<br>  tenant_id INTEGER REFERENCES tenants(id),<br>  entity_name VARCHAR(255),<br>  entity_contact VARCHAR(255),<br>  event_datetime TIMESTAMP NOT NULL,<br>  extra_data JSONB,<br>  status VARCHAR(20) DEFAULT 'pending'<br>);<br><br>CREATE TABLE tenant_reminders_sent (<br>  id SERIAL PRIMARY KEY,<br>  tenant_id INTEGER,<br>  rule_id INTEGER,<br>  event_id INTEGER,<br>  sent_at TIMESTAMP DEFAULT NOW()<br>);<br><br>CREATE TABLE tenant_logs (<br>  id SERIAL PRIMARY KEY,<br>  tenant_id INTEGER,<br>  rule_id INTEGER,<br>  event_id INTEGER,<br>  status VARCHAR(20),<br>  message_sent TEXT,<br>  error_message TEXT,<br>  created_at TIMESTAMP DEFAULT NOW()<br>);<br>``` |
| Webhook flow group | Sticky Note | Labels webhook processing area |  |  | ## Webhook flow<br>Receives events from external systems.<br>Validates the tenant token via `x-tenant-token` header,<br>fetches tenant config from the database,<br>and fires the reminder immediately. |
| Schedule flow group | Sticky Note | Labels scheduled processing area |  |  | ## Schedule flow<br>Runs every minute.<br>Queries events within the configured timing window per tenant.<br>Idempotency via `tenant_reminders_sent` — same reminder is never sent twice. |
| Registration group | Sticky Note | Labels tenant registration area |  |  | ## Tenant registration<br>Built-in n8n form to register new tenants.<br>No external backend needed.<br>Access at `/form/multi-tenant-register` |
| Send and log group | Sticky Note | Labels delivery and audit area |  |  | ## Send and log<br>Routes to the correct channel based on tenant config.<br>Logs every attempt — success and error — to the database.<br><br>To add a new channel: duplicate the IF node<br>and add the corresponding send node. |
| Every minute - check due events | Schedule Trigger | Fires every minute to poll due scheduled events |  | Fetch events due for reminder | ## Schedule flow<br>Runs every minute.<br>Queries events within the configured timing window per tenant.<br>Idempotency via `tenant_reminders_sent` — same reminder is never sent twice. |
| Log success to database | PostgreSQL | Writes successful send log entry | Send reminder via Telegram |  | ## Send and log<br>Routes to the correct channel based on tenant config.<br>Logs every attempt — success and error — to the database.<br><br>To add a new channel: duplicate the IF node<br>and add the corresponding send node. |
| Mark reminder as sent | PostgreSQL | Writes idempotency record after successful send | Send reminder via Telegram |  | ## Send and log<br>Routes to the correct channel based on tenant config.<br>Logs every attempt — success and error — to the database.<br><br>To add a new channel: duplicate the IF node<br>and add the corresponding send node. |
| Render Template | Code | Formats event variables and renders final message | Map webhook fields; Fetch events due for reminder | Route by channel type | ## Send and log<br>Routes to the correct channel based on tenant config.<br>Logs every attempt — success and error — to the database.<br><br>To add a new channel: duplicate the IF node<br>and add the corresponding send node. |
| Log error to database | PostgreSQL | Writes failed send log entry | Send reminder via Telegram |  | ## Send and log<br>Routes to the correct channel based on tenant config.<br>Logs every attempt — success and error — to the database.<br><br>To add a new channel: duplicate the IF node<br>and add the corresponding send node. |
| Send reminder via Telegram | Telegram | Sends outbound reminder via Telegram bot | Route by channel type | Mark reminder as sent; Log success to database; Log error to database | ## Send and log<br>Routes to the correct channel based on tenant config.<br>Logs every attempt — success and error — to the database.<br><br>To add a new channel: duplicate the IF node<br>and add the corresponding send node. |
| Receive event via webhook | Webhook | Receives external event POST requests |  | Validate tenant token | ## Webhook flow<br>Receives events from external systems.<br>Validates the tenant token via `x-tenant-token` header,<br>fetches tenant config from the database,<br>and fires the reminder immediately. |
| Validate tenant token | Code | Extracts and validates tenant API token from webhook headers | Receive event via webhook | Fetch tenant config | ## Webhook flow<br>Receives events from external systems.<br>Validates the tenant token via `x-tenant-token` header,<br>fetches tenant config from the database,<br>and fires the reminder immediately. |
| Map webhook fields | Set | Normalizes webhook event and tenant rule fields | Fetch tenant config | Render Template | ## Webhook flow<br>Receives events from external systems.<br>Validates the tenant token via `x-tenant-token` header,<br>fetches tenant config from the database,<br>and fires the reminder immediately. |
| Tenant registration form | Form Trigger | Accepts tenant onboarding data from a hosted form |  | Organize form data | ## Tenant registration<br>Built-in n8n form to register new tenants.<br>No external backend needed.<br>Access at `/form/multi-tenant-register` |
| Organize form data | Code | Maps form fields into DB-ready tenant and rule values | Tenant registration form | Insert tenant | ## Tenant registration<br>Built-in n8n form to register new tenants.<br>No external backend needed.<br>Access at `/form/multi-tenant-register` |
| Show registration success | Form | Returns completion page after registration | Insert tenant rule |  | ## Tenant registration<br>Built-in n8n form to register new tenants.<br>No external backend needed.<br>Access at `/form/multi-tenant-register` |
| Fetch events due for reminder | PostgreSQL | Queries due scheduled reminders not yet sent | Every minute - check due events | Render Template | ## Schedule flow<br>Runs every minute.<br>Queries events within the configured timing window per tenant.<br>Idempotency via `tenant_reminders_sent` — same reminder is never sent twice. |
| Fetch tenant config | PostgreSQL | Loads tenant and active webhook rule by API token | Validate tenant token | Map webhook fields | ## Webhook flow<br>Receives events from external systems.<br>Validates the tenant token via `x-tenant-token` header,<br>fetches tenant config from the database,<br>and fires the reminder immediately. |
| Insert tenant rule | PostgreSQL | Inserts initial reminder rule for a new tenant | Insert tenant | Show registration success | ## Tenant registration<br>Built-in n8n form to register new tenants.<br>No external backend needed.<br>Access at `/form/multi-tenant-register` |
| Insert tenant | PostgreSQL | Inserts a new tenant and returns its ID | Organize form data | Insert tenant rule | ## Tenant registration<br>Built-in n8n form to register new tenants.<br>No external backend needed.<br>Access at `/form/multi-tenant-register` |
| No events due - skip | No Operation | Terminal branch for unsupported channels | Route by channel type |  | ## Send and log<br>Routes to the correct channel based on tenant config.<br>Logs every attempt — success and error — to the database.<br><br>To add a new channel: duplicate the IF node<br>and add the corresponding send node. |
| Route by channel type | IF | Routes rendered reminder by channel | Render Template | Send reminder via Telegram; No events due - skip | ## Send and log<br>Routes to the correct channel based on tenant config.<br>Logs every attempt — success and error — to the database.<br><br>To add a new channel: duplicate the IF node<br>and add the corresponding send node. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something equivalent to: `Send automated reminders to multiple clients via Telegram using webhook and schedule triggers`.

2. **Add documentation sticky notes**
   - Add one sticky note for the overview and setup explanation.
   - Add one sticky note with the PostgreSQL schema.
   - Add group labels for:
     - Webhook flow
     - Schedule flow
     - Tenant registration
     - Send and log

3. **Prepare PostgreSQL**
   - Create a PostgreSQL database accessible from n8n.
   - Run the following tables exactly or with compatible schema:
     - `tenants`
     - `tenant_rules`
     - `tenant_events`
     - `tenant_reminders_sent`
     - `tenant_logs`
   - Use the schema documented in the workflow notes.
   - Create PostgreSQL credentials in n8n and assign them to all Postgres nodes.

4. **Create the webhook entry node**
   - Add a **Webhook** node.
   - Set:
     - **HTTP Method:** `POST`
     - **Path:** `multi-tenant-webhook`
   - Name it: `Receive event via webhook`.

5. **Create the webhook validation node**
   - Add a **Code** node after the webhook.
   - Name it: `Validate tenant token`.
   - Use this logic:
     - Read `x-tenant-token` from the request headers
     - Throw an error if missing
     - Return:
       - `token`
       - `entity_name`
       - `event_datetime`
       - `extra_data` defaulting to empty object
   - Connect `Receive event via webhook` → `Validate tenant token`.

6. **Create the webhook tenant lookup node**
   - Add a **Postgres** node.
   - Name it: `Fetch tenant config`.
   - Operation: `Execute Query`.
   - Use a query that:
     - Joins `tenants` and `tenant_rules`
     - Filters by `api_token`
     - Restricts to active webhook rules and active tenants
   - Equivalent logic:
     - select tenant ID, name, channel, channel config, rule ID, message template
     - where token matches the validated token
   - Connect `Validate tenant token` → `Fetch tenant config`.

7. **Create the webhook field mapping node**
   - Add a **Set** node.
   - Name it: `Map webhook fields`.
   - Create these fields:
     - `tenants_id` as number from tenant ID
     - `rule_id` as number
     - `channel` as string
     - `chat_id` as string from `channel_config.chat_id`
     - `message_template` as string
     - `entity_name` from the validation node
     - `event_datetime` from the validation node
     - `extra_data` from the validation node
     - `channel_config` as object from DB result
   - Connect `Fetch tenant config` → `Map webhook fields`.

8. **Create the schedule trigger**
   - Add a **Schedule Trigger** node.
   - Name it: `Every minute - check due events`.
   - Set it to run **every minute**.

9. **Create the scheduled event query**
   - Add a **Postgres** node.
   - Name it: `Fetch events due for reminder`.
   - Operation: `Execute Query`.
   - Build a query that:
     - joins `tenant_rules`, `tenants`, and `tenant_events`
     - only uses active schedule rules and active tenants
     - only uses pending events
     - checks whether the event falls within the rule’s timing window
     - excludes events already present in `tenant_reminders_sent`
   - Connect `Every minute - check due events` → `Fetch events due for reminder`.

10. **Create the template rendering node**
    - Add a **Code** node.
    - Name it: `Render Template`.
    - Set mode to **Run Once for Each Item**.
    - Implement logic to:
      - parse `event_datetime`
      - format `event_time` and `event_date` in timezone `America/Sao_Paulo`
      - build variables from:
        - `entity_name`
        - `entity_contact`
        - `tenants_id`
        - `event_time`
        - `event_date`
        - all keys from `extra_data`
      - replace `{{variable}}` placeholders inside `message_template`
      - return:
        - `event_id`
        - `rule_id`
        - `tenants_id`
        - `channel`
        - `channel_config`
        - `message`
    - Connect both:
      - `Map webhook fields` → `Render Template`
      - `Fetch events due for reminder` → `Render Template`

11. **Create the channel router**
    - Add an **IF** node.
    - Name it: `Route by channel type`.
    - Condition:
      - left value: `{{$json.channel}}`
      - operator: `equals`
      - right value: `telegram`
    - Connect `Render Template` → `Route by channel type`.

12. **Create the unsupported-channel sink**
    - Add a **No Operation** node.
    - Name it: `No events due - skip`.
    - Connect the **false** output of `Route by channel type` to this node.

13. **Create Telegram credentials**
    - In n8n, create Telegram bot credentials.
    - Ensure the bot can message the target `chat_id`.
    - Assign these credentials to the Telegram node you will create next.

14. **Create the Telegram send node**
    - Add a **Telegram** node.
    - Name it: `Send reminder via Telegram`.
    - Configure:
      - message text = `{{$json.message}}`
      - chat ID = `{{$json.channel_config.chat_id}}`
      - disable attribution/branding if supported
    - Set the node to **continue on error output**.
    - Keep retry disabled, matching the source workflow.
    - Connect the **true** output of `Route by channel type` → `Send reminder via Telegram`.

15. **Create the idempotency insert node**
    - Add a **Postgres** node.
    - Name it: `Mark reminder as sent`.
    - Operation: `Execute Query`.
    - Insert into `tenant_reminders_sent`:
      - `tenant_id`
      - `rule_id`
      - `event_id`
    - Pull values from `Render Template`.
    - Connect the **success output** of `Send reminder via Telegram` → `Mark reminder as sent`.

16. **Create the success log node**
    - Add a **Postgres** node.
    - Name it: `Log success to database`.
    - Operation: `Execute Query`.
    - Insert into `tenant_logs`:
      - `tenant_id`
      - `rule_id`
      - `event_id`
      - status = `success`
      - `message_sent`
    - Pull values from `Render Template`.
    - Connect the **success output** of `Send reminder via Telegram` → `Log success to database`.

17. **Create the error log node**
    - Add a **Postgres** node.
    - Name it: `Log error to database`.
    - Operation: `Execute Query`.
    - Insert into `tenant_logs`:
      - `tenant_id`
      - `rule_id`
      - `event_id`
      - status = `error`
      - `error_message`
    - Pull values from the Telegram error output item.
    - Connect the **error output** of `Send reminder via Telegram` → `Log error to database`.

18. **Create the tenant registration form trigger**
    - Add a **Form Trigger** node.
    - Name it: `Tenant registration form`.
    - Configure:
      - Form title: `Register New Tenant`
      - Fields:
        1. `Client name` — required
        2. `Channel` — dropdown with `telegram`, `whatsapp`, `email`
        3. `Chat ID / Contact` — required
        4. `API Token`
        5. `Trigger type` — dropdown with `schedule`, `webhook`
        6. `Minutes before event` — number
        7. `Message template` — required
      - Response mode: `Last Node`

19. **Create the form data transformer**
    - Add a **Code** node after the form.
    - Name it: `Organize form data`.
    - Map form labels into structured fields:
      - `name`
      - `channel`
      - `channel_config` as JSON string with `chat_id`
      - `api_token`
      - `trigger_type`
      - `timing_minutes_before`
      - `message_template`
    - Connect `Tenant registration form` → `Organize form data`.

20. **Create the tenant insert node**
    - Add a **Postgres** node.
    - Name it: `Insert tenant`.
    - Operation: `Execute Query`.
    - Insert into `tenants`:
      - `name`
      - `channel`
      - `channel_config`
      - `api_token`
      - `active = true`
    - Return the inserted ID with `RETURNING id`.
    - Connect `Organize form data` → `Insert tenant`.

21. **Create the rule insert node**
    - Add a **Postgres** node.
    - Name it: `Insert tenant rule`.
    - Operation: `Execute Query`.
    - Insert into `tenant_rules`:
      - `tenant_id` from inserted tenant
      - `trigger_type`
      - `timing_minutes_before`
      - `message_template`
      - `active = true`
    - Connect `Insert tenant` → `Insert tenant rule`.

22. **Create the registration completion page**
    - Add a **Form** node.
    - Name it: `Show registration success`.
    - Set operation to **Completion**.
    - Set completion title to show the inserted tenant ID, such as:
      - `Tenant registered! ID: {{ $('Insert tenant').item.json.id }}`
    - Connect `Insert tenant rule` → `Show registration success`.

23. **Assign credentials**
    - Attach the same PostgreSQL credential to:
      - `Fetch tenant config`
      - `Fetch events due for reminder`
      - `Mark reminder as sent`
      - `Log success to database`
      - `Log error to database`
      - `Insert tenant`
      - `Insert tenant rule`
    - Attach Telegram credentials to:
      - `Send reminder via Telegram`

24. **Test the registration flow**
    - Open the form endpoint for the Form Trigger.
    - Register a Telegram tenant with:
      - a valid bot-accessible `chat_id`
      - a usable `API Token`
      - a message template such as `Hello {{entity_name}}, your event is on {{event_date}} at {{event_time}}`
    - Confirm the success screen shows a tenant ID.

25. **Test the webhook flow**
    - Send a POST request to `/webhook/multi-tenant-webhook`.
    - Include header:
      - `x-tenant-token: <tenant api token>`
    - Include body with at least:
      - `entity_name`
      - `event_datetime`
      - optionally `extra_data`
    - Confirm a Telegram message is sent and success rows appear in logging tables.

26. **Test the schedule flow**
    - Insert a row into `tenant_events` for a tenant with a `schedule` rule.
    - Ensure:
      - `status = 'pending'`
      - `event_datetime` falls inside the expected reminder window
    - Wait for the next minute tick.
    - Confirm the message is sent, then check:
      - `tenant_reminders_sent`
      - `tenant_logs`

27. **Activate the workflow**
    - Once all three entry points are tested, activate the workflow so the webhook and schedule trigger operate continuously.

### Important implementation constraints
- The current routing only sends `telegram`. `whatsapp` and `email` are accepted by the form but not implemented.
- The workflow uses direct SQL string interpolation in multiple nodes. For production use, parameterized queries are strongly recommended.
- There is no explicit uniqueness constraint on `tenant_reminders_sent`; adding one for `(rule_id, event_id)` would improve idempotency.
- The rendering timezone is fixed to `America/Sao_Paulo`.
- Webhook-based sends do not create a `tenant_events` row by themselves; they send immediately from the inbound payload.

### Sub-workflow setup
- This workflow does **not** invoke any sub-workflows.
- It contains **multiple entry points**:
  - Webhook
  - Schedule Trigger
  - Form Trigger

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Multi-tenant reminder system with 3 entry points: webhook, schedule, and registration form | Internal workflow architecture note |
| Register your first tenant at `/form/multi-tenant-register` | Registration endpoint |
| Send events via `POST /webhook/multi-tenant-webhook` with `x-tenant-token` header | Webhook endpoint |
| Required database tables: `tenants`, `tenant_rules`, `tenant_events`, `tenant_reminders_sent`, `tenant_logs` | PostgreSQL setup |
| Channel routing is currently implemented only for Telegram, but the structure is designed to be extendable to WhatsApp, email, or other channels | Extension guidance |
| To add a new channel: duplicate the IF/routing logic and add the corresponding delivery node | Internal extension note |