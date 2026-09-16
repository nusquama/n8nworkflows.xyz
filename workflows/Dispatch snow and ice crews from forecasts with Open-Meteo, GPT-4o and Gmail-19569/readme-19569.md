Dispatch snow and ice crews from forecasts with Open-Meteo, GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/dispatch-snow-and-ice-crews-from-forecasts-with-open-meteo--gpt-4o-and-gmail-19569


# Dispatch snow and ice crews from forecasts with Open-Meteo, GPT-4o and Gmail

### 1. Workflow Overview

This workflow automates the monitoring of winter weather forecasts, evaluates per-site dispatch triggers, generates AI-powered crew briefs and client notices, handles crew completion callbacks, and runs compliance/follow-up sweeps for unconfirmed dispatches.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Validation:** Initiates forecast sweeps via schedule or webhooks, reads property configurations from Google Sheets, and filters out malformed or incomplete rows.
- **1.2 Forecast Check & Rule Evaluation:** Queries Open-Meteo for hourly weather data, evaluates snow and ice criteria against property thresholds, and appends a check log summary.
- **1.3 Invalid Row Handling:** Processes, logs, and alerts administrators regarding rejected property configurations.
- **1.4 AI Dispatch Generation & Notifications:** Aggregates sites requiring action, generates custom briefs and client updates using OpenAI, updates the dispatch log, and notifies crews, property contacts, and owners.
- **1.5 Crew Confirmation & Status Update:** Receives completion webhooks from field crews, validates dispatch IDs, updates operational logs, and notifies clients and owners.
- **1.6 Unconfirmed Dispatch Monitoring:** Sweeps the dispatch log daily (or on demand) to flag stale operations exceeding established timeframes, sending operational nudges as needed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Triggers forecast checks on a fixed interval or via manual webhooks, reads site data from Google Sheets, and structurally validates required metadata before processing.
- **Nodes Involved:** 
  - Poll Weather Every 3 Hours
  - Run Weather Check Now
  - Read Site Registry
  - Validate Site Rows
  - Site Row Valid?

- **Node Details:**
  - **Poll Weather Every 3 Hours**
    - *Type & Role:* Schedule Trigger node executing every 3 hours (`0 */3 * * *`).
    - *Configuration:* Cron expression schedule.
    - *Input/Output:* No inputs; outputs execution signal to `Read Site Registry`.
    - *Edge Cases:* Timezone discrepancies if instance timezone is not synchronized.
  - **Run Weather Check Now**
    - *Type & Role:* Webhook node listening on path `snow/check` (POST).
    - *Configuration:* HTTP POST receiver.
    - *Input/Output:* Triggers `Read Site Registry`.
  - **Read Site Registry**
    - *Type & Role:* Google Sheets node reading the `Sites` tab.
    - *Configuration:* Document ID `1uy_a0M6kTCbzQI3k8p0tYwAyagaDBSTCeXLpM7UQk1w`, sheet name `Sites`.
    - *Input/Output:* Input from triggers; outputs raw sheet rows to `Validate Site Rows`.
    - *Edge Cases:* API rate limits, authentication failures, or missing spreadsheet tabs.
  - **Validate Site Rows**
    - *Type & Role:* Custom JavaScript Code node validating coordinates, triggers, active status, and email formatting.
    - *Configuration:* Iterates items, parses floats, validates latitude/longitude boundaries, checks email strings for `@`.
    - *Input/Output:* Input from `Read Site Registry`; outputs validated objects with boolean `valid` and `invalid_reason` flags.
  - **Site Row Valid?**
    - *Type & Role:* If node branching execution based on row validity.
    - *Configuration:* Evaluates `{{ $json.valid }}` (boolean true).
    - *Input/Output:* Inputs from `Validate Site Rows`; outputs valid rows to `Fetch Site Forecast` and invalid rows to `Build Invalid Site Alert`.

---

#### 2.2 Forecast Check & Rule Evaluation
- **Overview:** Fetches meteorological data for verified sites, applies custom threshold rules for snow and freezing precipitation, logs operational run summaries.
- **Nodes Involved:**
  - Fetch Site Forecast
  - Evaluate Dispatch Rules
  - Dispatch Needed?
  - No Dispatch Needed
  - Build Check Log Row
  - Append Check Log

- **Node Details:**
  - **Fetch Site Forecast**
    - *Type & Role:* HTTP Request node querying the Open-Meteo API.
    - *Configuration:* URL uses template `https://api.open-meteo.com/v1/forecast?latitude={{ $json.lat }}&longitude={{ $json.lon }}&hourly=snowfall,temperature_2m,precipitation,wind_speed_10m&forecast_days=2&timezone=auto`.
    - *Input/Output:* Inputs valid site rows; outputs hourly forecast arrays.
    - *Edge Cases:* API timeouts, upstream service downtime, rate limiting.
  - **Evaluate Dispatch Rules**
    - *Type & Role:* Custom JavaScript Code node processing forecast windows against site criteria.
    - *Configuration:* Examines the next 12 hours of snowfall, temperature, and precipitation against per-site thresholds (`trigger_cm`, `ice_risk`).
    - *Input/Output:* Inputs forecast payloads; outputs augmented site objects containing `needs_dispatch` flags and severity metadata.
  - **Dispatch Needed?**
    - *Type & Role:* If node routing sites based on dispatch necessity.
    - *Configuration:* Evaluates `{{ $json.needs_dispatch }}`.
    - *Input/Output:* Inputs evaluated rules; outputs actionable sites to `Combine Dispatch Sites` and inactive sites to `No Dispatch Needed`.
  - **No Dispatch Needed**
    - *Type & Role:* No-Op node terminating inactive site branches.
    - *Configuration:* Pass-through node.
  - **Build Check Log Row**
    - *Type & Role:* Custom JavaScript Code node compiling summary statistics of the current sweep.
    - *Configuration:* Aggregates counts for checked sites, dispatches raised, no-action sites, and invalid records.
    - *Input/Output:* Inputs evaluated rules; outputs check log payload.
  - **Append Check Log**
    - *Type & Role:* Google Sheets node writing summary records.
    - *Configuration:* Appends to sheet `CheckLog`.
    - *Input/Output:* Inputs check log payload; appends row to storage.

---

#### 2.3 Invalid Row Handling
- **Overview:** Formats error payloads and distributes notifications across administrator channels when site registry validation fails.
- **Nodes Involved:**
  - Build Invalid Site Alert
  - Alert Invalid Site On Telegram
  - Email Site Admin About Invalid Row
  - Log Site Error

- **Node Details:**
  - **Build Invalid Site Alert**
    - *Type & Role:* Set node formatting error messages, unique error IDs, and email bodies.
    - *Configuration:* Assigns error attributes, timestamps (`$now.toISO()`), and notification text templates.
    - *Input/Output:* Inputs invalid site objects; outputs formatted error objects.
  - **Alert Invalid Site On Telegram**
    - *Type & Role:* Telegram node sending operational warnings to chat channels.
    - *Configuration:* Uses chat ID `123456789`, message template `{{ $json.tg_text }}`.
    - *Input/Output:* Inputs formatted error object.
    - *Edge Cases:* Telegram API token expiration or invalid chat ID.
  - **Email Site Admin About Invalid Row**
    - *Type & Role:* Gmail node alerting administrators via email.
    - *Configuration:* Sends to `user@example.com`, subject `{{ $json.email_subject }}`, HTML body `{{ $json.email_body }}`.
    - *Input/Output:* Inputs formatted error object.
  - **Log Site Error**
    - *Type & Role:* Google Sheets node recording system exceptions.
    - *Configuration:* Appends record to sheet `Errors`.
    - *Input/Output:* Inputs formatted error object; writes error log.

---

#### 2.4 AI Dispatch Generation & Notifications
- **Overview:** Batches actionable sites, queries an LLM to generate customized operational briefs and client communications, logs entries, and dispatches multi-channel alerts.
- **Nodes Involved:**
  - Combine Dispatch Sites
  - Write Dispatch Briefs
  - Dispatch Brief Model
  - Split Dispatch Briefs
  - Log Dispatch
  - Email Crew Dispatch Brief
  - Email Property Contact Notice
  - Owner Alert Node (`Alert Owner On Dispatch`)

- **Node Details:**
  - **Combine Dispatch Sites**
    - *Type & Role:* Custom JavaScript Code node aggregating all sites requiring dispatch into a single text block.
    - *Configuration:* Maps and joins property criteria, forecasts, and contact details.
    - *Input/Output:* Inputs multiple dispatch records; outputs aggregated string payload.
  - **Write Dispatch Briefs**
    - *Type & Role:* Advanced AI LangChain Agent node.
    - *Configuration:* Uses system instructions defining role parameters, requiring strict JSON output (`{"briefs":[...]}`, containing `site_id`, `storm_summary`, `crew_brief`, `client_notice`).
    - *Input/Output:* Inputs combined site list; outputs raw text containing LLM JSON response.
  - **Dispatch Brief Model**
    - *Type & Role:* OpenAI Chat Model sub-node.
    - *Configuration:* Uses model `gpt-4o-mini`.
    - *Input/Output:* Linked via AI language model connection to `Write Dispatch Briefs`.
    - *Edge Cases:* API token limits, malformed JSON responses from LLM, or rate limits.
  - **Split Dispatch Briefs**
    - *Type & Role:* Custom JavaScript Code node parsing LLM output and combining it with site telemetry.
    - *Configuration:* Extracts JSON substring, maps briefs back to individual site metadata, and generates unique dispatch IDs (`DSP-[site_id]-[timestamp]`).
    - *Input/Output:* Inputs raw LLM response; outputs discrete dispatch site objects.
  - **Log Dispatch**
    - *Type & Role:* Google Sheets node recording active dispatches.
    - *Configuration:* Appends rows to sheet `DispatchLog` with status set to `dispatched`.
    - *Input/Output:* Inputs dispatch objects.
  - **Email Crew Dispatch Brief**
    - *Type & Role:* Gmail node sending operational instructions to field crew leads.
    - *Configuration:* Sends to `{{ $json.crew_email }}` with storm summary and generated crew brief.
    - *Input/Output:* Inputs dispatch objects.
  - **Email Property Contact Notice**
    - *Type & Role:* Gmail node notifying property clients of impending winter maintenance.
    - *Configuration:* Sends to `{{ $json.contact_email }}` with client notice messaging.
    - *Input/Output:* Inputs dispatch objects.
  - **Alert Owner On Dispatch**
    - *Type & Role:* Telegram node posting dispatch confirmation alerts.
    - *Configuration:* Uses chat ID `123456789`, message template summarizing dispatch ID and assignment metrics.
    - *Input/Output:* Inputs dispatch objects.

---

#### 2.5 Crew Confirmation & Status Update
- **Overview:** Ingests field completion webhooks, verifies dispatch identifiers against operational logs, updates status registers, and closes out client notifications.
- **Nodes Involved:**
  - Crew Marks Site Done
  - Normalize Crew Confirmation
  - Read Dispatch Log For Confirm
  - Match Dispatch Row
  - Confirmation Ready?
  - Update Dispatch Log
  - Email Client Completion Notice
  - Alert Owner On Completion
  - Build Confirmation Error
  - Alert Confirmation Error
  - Email Confirmation Error

- **Node Details:**
  - **Crew Marks Site Done**
    - *Type & Role:* Webhook node listening on path `snow/confirm` (POST).
    - *Configuration:* HTTP POST receiver for field crew callbacks.
    - *Input/Output:* Triggers `Normalize Crew Confirmation`.
  - **Normalize Crew Confirmation**
    - *Type & Role:* Set node standardizing incoming webhook payloads.
    - *Configuration:* Maps `body.dispatch_id`, `body.status`, `body.hours_to_clear`, `body.salt_used`, and `body.notes`.
    - *Input/Output:* Inputs raw webhook body; outputs normalized confirmation object.
  - **Read Dispatch Log For Confirm**
    - *Type & Role:* Google Sheets node reading existing dispatch logs.
    - *Configuration:* Reads sheet `DispatchLog`.
    - *Input/Output:* Outputs all log rows to matching logic.
  - **Match Dispatch Row**
    - *Type & Role:* Custom JavaScript Code node cross-referencing incoming dispatch IDs with historical logs.
    - *Configuration:* Checks if `dispatch_id` exists in the log and validates completeness.
    - *Input/Output:* Inputs confirmation object and sheet rows; outputs verified readiness flags.
  - **Confirmation Ready?**
    - *Type & Role:* If node branching based on match validation.
    - *Configuration:* Evaluates `{{ $json.ready }}`.
    - *Input/Output:* Outputs valid matches to update/notification nodes and invalid matches to error builders.
  - **Update Dispatch Log**
    - *Type & Role:* Google Sheets node updating dispatch records.
    - *Configuration:* Operation `update`, matching column `dispatch_id`, sets status, completed timestamp, hours to clear, salt used, and notes.
    - *Input/Output:* Inputs validated confirmation records.
  - **Email Client Completion Notice**
    - *Type & Role:* Gmail node confirming service completion to property contacts.
    - *Configuration:* Sends to `{{ $json.contact_email }}`.
    - *Input/Output:* Inputs validated confirmation records.
  - **Alert Owner On Completion**
    - *Type & Role:* Telegram node notifying management of completed work.
    - *Configuration:* Uses chat ID `123456789`.
    - *Input/Output:* Inputs validated confirmation records.
  - **Build Confirmation Error / Alert Confirmation Error / Email Confirmation Error**
    - *Type & Role:* Error handling sequence (Set, Telegram, and Gmail nodes).
    - *Configuration:* Compiles error context for missing or unknown dispatch IDs, alerting administrators via Telegram and email without modifying logs.
    - *Input/Output:* Inputs invalid confirmation objects.

---

#### 2.6 Unconfirmed Dispatch Monitoring
- **Overview:** Periodically sweeps open dispatches to identify stale operations exceeding configured time thresholds, issuing operational follow-up reminders.
- **Nodes Involved:**
  - Run Dispatch Follow Up Now
  - Daily Unconfirmed Dispatch Sweep
  - Read Dispatch Log For Sweep
  - Find Unconfirmed Dispatches
  - Unconfirmed Dispatches?
  - Email Unconfirmed Dispatch Report
  - Alert Unconfirmed On Telegram
  - All Dispatches Confirmed

- **Node Details:**
  - **Run Dispatch Follow Up Now**
    - *Type & Role:* Webhook node listening on path `snow/followup` (POST).
    - *Configuration:* Manual trigger endpoint.
  - **Daily Unconfirmed Dispatch Sweep**
    - *Type & Role:* Schedule Trigger node executing daily at 07:00 (`0 7 * * *`).
    - *Configuration:* Cron schedule expression.
  - **Read Dispatch Log For Sweep**
    - *Type & Role:* Google Sheets node reading log records.
    - *Configuration:* Reads sheet `DispatchLog`.
    - *Input/Output:* Outputs dispatch records to analysis code.
  - **Find Unconfirmed Dispatches**
    - *Type & Role:* Custom JavaScript Code node calculating dispatch age.
    - *Configuration:* Filters items where status is `dispatched` and elapsed time exceeds 6 hours.
    - *Input/Output:* Inputs dispatch logs; outputs aggregated stale dispatch lists and summary statistics.
  - **Unconfirmed Dispatches?**
    - *Type & Role:* If node routing based on stale count.
    - *Configuration:* Evaluates `{{ $json.open_count >= 1 }}`.
    - *Input/Output:* Routes open dispatches to warning notifications and zero-state conditions to `All Dispatches Confirmed`.
  - **Email Unconfirmed Dispatch Report**
    - *Type & Role:* Gmail node sending stale dispatch reports to administrators.
    - *Configuration:* Sends to `user@example.com`.
    - *Input/Output:* Inputs stale dispatch summary.
  - **Alert Unconfirmed On Telegram**
    - *Type & Role:* Telegram node posting follow-up warnings to chat.
    - *Configuration:* Uses chat ID `123456789`.
  - **All Dispatches Confirmed**
    - *Type & Role:* No-Op node handling empty states.
    - *Configuration:* Pass-through node indicating normal operations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Poll Weather Every 3 Hours | scheduleTrigger | Triggers forecast check on 3-hour interval | None | Read Site Registry | Triggers |
| Run Weather Check Now | webhook | Manual trigger for forecast sweep via webhook | None | Read Site Registry | Triggers |
| Crew Marks Site Done | webhook | Receives crew completion confirmation via webhook | None | Normalize Crew Confirmation | Crew confirmation |
| Run Dispatch Follow Up Now | webhook | Manual trigger for stale dispatch sweep via webhook | None | Read Dispatch Log For Sweep | Unconfirmed dispatches |
| Daily Unconfirmed Dispatch Sweep | scheduleTrigger | Triggers daily follow-up check at 07:00 | None | Read Dispatch Log For Sweep | Unconfirmed dispatches |
| Read Site Registry | googleSheets | Reads property configuration from Sites tab | Poll Weather Every 3 Hours, Run Weather Check Now | Validate Site Rows | |
| Validate Site Rows | code | Validates coordinates, triggers, and emails | Read Site Registry | Site Row Valid? | |
| Site Row Valid? | if | Branches execution based on row validity | Validate Site Rows | Fetch Site Forecast, Build Invalid Site Alert | |
| Build Invalid Site Alert | set | Formats error payload for rejected site rows | Site Row Valid? | Alert Invalid Site On Telegram | Rejected site rows |
| Alert Invalid Site On Telegram | telegram | Sends Telegram alert for rejected site rows | Build Invalid Site Alert | Email Site Admin About Invalid Row | Rejected site rows |
| Email Site Admin About Invalid Row | gmail | Emails admin regarding invalid site row | Alert Invalid Site On Telegram | Log Site Error | Rejected site rows |
| Log Site Error | googleSheets | Appends rejected row error to Errors tab | Email Site Admin About Invalid Row | None | Rejected site rows |
| Fetch Site Forecast | httpRequest | Queries Open-Meteo API for 2-day hourly forecast | Site Row Valid? | Evaluate Dispatch Rules | Forecast check and rule engine |
| Evaluate Dispatch Rules | code | Evaluates 12-hour weather window against triggers | Fetch Site Forecast | Build Check Log Row, Dispatch Needed? | Forecast check and rule engine |
| Build Check Log Row | code | Compiles sweep execution summary statistics | Evaluate Dispatch Rules | Append Check Log | Check log |
| Append Check Log | googleSheets | Writes check summary to CheckLog tab | Build Check Log Row | None | Check log |
| Dispatch Needed? | if | Branches workflow based on dispatch requirement | Evaluate Dispatch Rules | Combine Dispatch Sites, No Dispatch Needed | |
| No Dispatch Needed | noOp | Terminates execution branch for sites not requiring dispatch | Dispatch Needed? | None | |
| Combine Dispatch Sites | code | Aggregates all actionable sites into single payload | Dispatch Needed? | Write Dispatch Briefs | Dispatch, crew brief and client notice |
| Write Dispatch Briefs | agent | LangChain agent generating AI operational briefs | Combine Dispatch Sites | Split Dispatch Briefs | Dispatch, crew brief and client notice |
| Dispatch Brief Model | lmChatOpenAi | OpenAI GPT-4o-mini chat model provider | None | Write Dispatch Briefs | Dispatch, crew brief and client notice |
| Split Dispatch Briefs | code | Parses AI output and maps briefs to individual sites | Write Dispatch Briefs | Log Dispatch, Email Crew Dispatch Brief, Email Property Contact Notice, Alert Owner On Dispatch | Dispatch, crew brief and client notice |
| Log Dispatch | googleSheets | Records active dispatch entries in DispatchLog tab | Split Dispatch Briefs | None | Dispatch, crew brief and client notice |
| Email Crew Dispatch Brief | gmail | Emails operational brief to crew lead | Split Dispatch Briefs | None | Dispatch, crew brief and client notice |
| Email Property Contact Notice | gmail | Emails service notification to property contact | Split Dispatch Briefs | None | Dispatch, crew brief and client notice |
| Alert Owner On Dispatch | telegram | Posts dispatch alert to Telegram | Split Dispatch Briefs | None | Dispatch, crew brief and client notice |
| Normalize Crew Confirmation | set | Standardizes incoming crew confirmation webhook data | Crew Marks Site Done | Read Dispatch Log For Confirm | Crew confirmation |
| Read Dispatch Log For Confirm | googleSheets | Reads DispatchLog tab for ID verification | Normalize Crew Confirmation | Match Dispatch Row | Crew confirmation |
| Match Dispatch Row | code | Validates existence of dispatch ID in logs | Read Dispatch Log For Confirm | Confirmation Ready? | Crew confirmation |
| Confirmation Ready? | if | Branches execution based on dispatch ID match | Match Dispatch Row | Update Dispatch Log, Email Client Completion Notice, Alert Owner On Completion, Build Confirmation Error | Crew confirmation |
| Update Dispatch Log | googleSheets | Updates DispatchLog row with completion status | Confirmation Ready? | None | Crew confirmation |
| Email Client Completion Notice | gmail | Emails project completion notice to client | Confirmation Ready? | None | Crew confirmation |
| Alert Owner On Completion | telegram | Posts completion confirmation alert to Telegram | Confirmation Ready? | None | Crew confirmation |
| Build Confirmation Error | set | Formats error payload for unknown dispatch IDs | Confirmation Ready? | Alert Confirmation Error | Bad or unknown dispatch |
| Alert Confirmation Error | telegram | Posts unknown dispatch error alert to Telegram | Build Confirmation Error | Email Confirmation Error | Bad or unknown dispatch |
| Email Confirmation Error | gmail | Emails unknown dispatch error notification to admin | Alert Confirmation Error | None | Bad or unknown dispatch |
| Read Dispatch Log For Sweep | googleSheets | Reads DispatchLog tab for unconfirmed sweep | Run Dispatch Follow Up Now, Daily Unconfirmed Dispatch Sweep | Find Unconfirmed Dispatches | Unconfirmed dispatches |
| Find Unconfirmed Dispatches | code | Identifies dispatches open longer than 6 hours | Read Dispatch Log For Sweep | Unconfirmed Dispatches? | Unconfirmed dispatches |
| Unconfirmed Dispatches? | if | Branches execution based on stale dispatch count | Find Unconfirmed Dispatches | Email Unconfirmed Dispatch Report, All Dispatches Confirmed | Unconfirmed dispatches |
| Email Unconfirmed Dispatch Report | gmail | Emails stale dispatch report to administrator | Unconfirmed Dispatches? | Alert Unconfirmed On Telegram | Unconfirmed dispatches |
| Alert Unconfirmed On Telegram | telegram | Posts stale dispatch alert to Telegram | Email Unconfirmed Dispatch Report | None | Unconfirmed dispatches |
| All Dispatches Confirmed | noOp | Handles zero-state when all dispatches are confirmed | Unconfirmed Dispatches? | None | Unconfirmed dispatches |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create Spreadsheet and Tabs
1. Create a Google Sheets workbook.
2. Add four tabs named exactly: `Sites`, `DispatchLog`, `CheckLog`, and `Errors`.
3. Ensure column headers exist matching the names read/written in the code nodes (e.g., `site_id`, `site_name`, `address`, `lat`, `lon`, `priority`, `trigger_cm`, `ice_risk`, `crew_name`, `crew_email`, `contact_name`, `contact_email`, `active`, `dispatch_id`, `status`, `dispatched_at`, `completed_at`, `hours_to_clear`, `salt_used`, `notes`).

#### Step 2: Configure Triggers and Inputs
1. Create a **Schedule Trigger** node (`Poll Weather Every 3 Hours`) set to cron expression `0 */3 * * *`.
2. Create three **Webhook** nodes:
   - `Run Weather Check Now` (Path: `snow/check`, Method: `POST`)
   - `Crew Marks Site Done` (Path: `snow/confirm`, Method: `POST`)
   - `Run Dispatch Follow Up Now` (Path: `snow/followup`, Method: `POST`)
3. Create another **Schedule Trigger** node (`Daily Unconfirmed Dispatch Sweep`) set to cron expression `0 7 * * *`.

#### Step 3: Implement Site Validation Logic
1. Add a **Google Sheets** node (`Read Site Registry`) connected to `Poll Weather Every 3 Hours` and `Run Weather Check Now`. Configure it to read sheet `Sites` using your Google Sheets OAuth2 credential and Spreadsheet ID.
2. Add a **Code** node (`Validate Site Rows`) to process raw input rows, validate lat/lon boundaries, check email formatting, and output standard properties with a `valid` boolean flag.
3. Add an **If** node (`Site Row Valid?`) evaluating `{{ $json.valid }}`:
   - *True branch* connects to weather fetching.
   - *False branch* connects to error handling.

#### Step 4: Implement Invalid Row Handling
1. Add a **Set** node (`Build Invalid Site Alert`) to construct error metadata.
2. Add a **Telegram** node (`Alert Invalid Site On Telegram`) using a Telegram Bot credential and target chat ID.
3. Add a **Gmail** node (`Email Site Admin About Invalid Row`) using a Gmail OAuth2 credential.
4. Add a **Google Sheets** node (`Log Site Error`) appending records to the `Errors` tab.
5. Connect in sequence: `Build Invalid Site Alert` $\rightarrow$ `Alert Invalid Site On Telegram` $\rightarrow$ `Email Site Admin About Invalid Row` $\rightarrow$ `Log Site Error`.

#### Step 5: Implement Forecast Checking and Rule Evaluation
1. Add an **HTTP Request** node (`Fetch Site Forecast`) requesting `https://api.open-meteo.com/v1/forecast?latitude={{ $json.lat }}&longitude={{ $json.lon }}&hourly=snowfall,temperature_2m,precipitation,wind_speed_10m&forecast_days=2&timezone=auto`.
2. Add a **Code** node (`Evaluate Dispatch Rules`) to parse the 12-hour forecast against site-specific triggers (`trigger_cm`, `ice_risk`), returning augmented site objects with `needs_dispatch` flags.
3. Add two downstream paths from `Evaluate Dispatch Rules`:
   - Path A (Check Logging): Add a **Code** node (`Build Check Log Row`) and a **Google Sheets** node (`Append Check Log`) targeting the `CheckLog` tab.
   - Path B (Dispatch Routing): Add an **If** node (`Dispatch Needed?`) evaluating `{{ $json.needs_dispatch }}`. Connect the false branch to a **No-Op** node (`No Dispatch Needed`).

#### Step 6: Implement AI Generation and Dispatch Notifications
1. Connect the true branch of `Dispatch Needed?` to a **Code** node (`Combine Dispatch Sites`) to aggregate site lists.
2. Add an **Advanced AI Agent** node (`Write Dispatch Briefs`) configured with a system message instructing strict JSON output (`{"briefs":[...]}`).
3. Attach an **OpenAI Chat Model** sub-node (`Dispatch Brief Model`) using model `gpt-4o-mini` with an OpenAI API credential.
4. Connect the agent output to a **Code** node (`Split Dispatch Briefs`) to parse the JSON and generate unique dispatch IDs (`DSP-[site_id]-[timestamp]`).
5. Connect `Split Dispatch Briefs` to four parallel output nodes:
   - **Google Sheets** (`Log Dispatch`) appending to `DispatchLog`.
   - **Gmail** (`Email Crew Dispatch Brief`) sending to crew emails.
   - **Gmail** (`Email Property Contact Notice`) sending to client contacts.
   - **Telegram** (`Alert Owner On Dispatch`) posting owner alerts.

#### Step 7: Implement Crew Confirmation Handling
1. Connect `Crew Marks Site Done` webhook to a **Set** node (`Normalize Crew Confirmation`) mapping webhook body parameters (`dispatch_id`, `status`, `hours_to_clear`, `salt_used`, `notes`).
2. Add a **Google Sheets** node (`Read Dispatch Log For Confirm`) reading the `DispatchLog` tab.
3. Add a **Code** node (`Match Dispatch Row`) verifying dispatch ID existence.
4. Add an **If** node (`Confirmation Ready?`) evaluating `{{ $json.ready }}`:
   - *True branch* connects to:
     - **Google Sheets** (`Update Dispatch Log`) updating records where `dispatch_id` matches.
     - **Gmail** (`Email Client Completion Notice`) notifying property contacts.
     - **Telegram** (`Alert Owner On Completion`) notifying owners.
   - *False branch* connects to error handling (`Build Confirmation Error` $\rightarrow$ `Alert Confirmation Error` $\rightarrow$ `Email Confirmation Error`).

#### Step 8: Implement Unconfirmed Dispatch Sweep
1. Connect `Run Dispatch Follow Up Now` and `Daily Unconfirmed Dispatch Sweep` to a **Google Sheets** node (`Read Dispatch Log For Sweep`) reading the `DispatchLog` tab.
2. Add a **Code** node (`Find Unconfirmed Dispatches`) filtering dispatches open longer than 6 hours.
3. Add an **If** node (`Unconfirmed Dispatches?`) evaluating `{{ $json.open_count >= 1 }}`:
   - *True branch* connects to **Gmail** (`Email Unconfirmed Dispatch Report`) and **Telegram** (`Alert Unconfirmed On Telegram`).
   - *False branch* connects to a **No-Op** node (`All Dispatches Confirmed`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n Creator Link | Built with [n8n](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Consultation & Services | [Consultation Booking](https://khmuhtadin.com/consultation/) |