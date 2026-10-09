Dispatch and track cold-chain incidents with Google Sheets, Twilio, and Gmail

https://n8nworkflows.xyz/workflows/dispatch-and-track-cold-chain-incidents-with-google-sheets--twilio--and-gmail-20428


# Dispatch and track cold-chain incidents with Google Sheets, Twilio, and Gmail

### 1. Workflow Overview

This workflow is a comprehensive, automated cold-chain incident management, technician dispatch, and SLA compliance engine. Its primary purpose is to ingest real-time temperature telemetry or sensor alerts from monitoring systems, validate client contracts, filter out harmless defrost cycles, calculate incident severity, and intelligently assign certified technicians based on geographic zone and availability. 

The workflow supports end-to-end incident lifecycles, including automated reassignments, escalation workflows via SMS and voice calls, technician status callbacks, daily SLA reporting, and robust error handling.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Validation:** Ingests inbound webhook alerts, loads runtime settings, normalizes the payload, and validates payload integrity.
- **1.2 Client SLA Lookup:** Queries Google Sheets to retrieve client contracts, match setpoints, and verify customer validity.
- **1.3 Defrost Filtering & Severity Triage:** Identifies routine defrost cycles (logging them as observations without dispatching) or computes incident severity (P1–P3) and SLA deadlines, followed by duplicate alert suppression.
- **1.4 Technician Matching & Dispatch:** Queries technician rosters, matches qualified and available personnel, assigns tickets, locks technicians, and sends automated SMS messages to technicians and client dispatch desks.
- **1.5 Technician Callback Intake:** Processes technician status updates (accept, decline, on-site, resolved) via interactive webhook links, updates database states, and renders HTML confirmation pages.
- **1.6 SLA Heartbeat & Escalation Engine:** Runs on a 5-minute schedule to inspect open tickets, trigger 50% warning alerts, reassign ignored tickets, escalate breaches, and place automated Twilio voice calls.
- **1.7 Error Handling:** Captures runtime exceptions globally, texts system administrators via Twilio, and writes audit logs.
- **1.8 Daily Reporting:** Runs daily at 08:00 to aggregate 24-hour SLA performance metrics, archive daily snapshots, and email structured HTML performance reports via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Receives inbound POST requests containing sensor telemetry, loads operational configuration parameters, standardizes data fields into an internal ticket draft format, and filters out malformed requests.
- **Nodes Involved:** 
  - `Receive ColdChain Alert`
  - `Set Intake Configuration`
  - `Normalize Alert Payload`
  - `Check Payload Validity`
  - `Return 400 Bad Payload`

- **Node Details:**
  - **Receive ColdChain Alert**
    - *Type and Technical Role:* Webhook node acting as the primary entry point (`/coldchain-alert`).
    - *Configuration Choices:* Listens to HTTP POST requests using a response node pattern.
    - *Key Expressions or Variables:* Extracts body parameters.
    - *Input/Output Connections:* Input none (trigger); output connects to `Set Intake Configuration`.
    - *Edge Cases / Failures:* Network dropouts, malformed JSON payloads resulting in missing attributes.
  - **Set Intake Configuration**
    - *Type and Technical Role:* Set node used to define local runtime variables.
    - *Configuration Choices:* Injects baseline constants including Twilio sender numbers, base URL endpoints, and operational escalation contact numbers.
    - *Input/Output Connections:* Input from `Receive ColdChain Alert`; output to `Normalize Alert Payload`.
  - **Normalize Alert Payload**
    - *Type and Technical Role:* Code node (JavaScript) processing raw payloads.
    - *Configuration Choices:* Generates a unique ticket ID (`TCK-`), parses temperatures, manages sensor fault flags, and sets validation status flags.
    - *Key Expressions/Variables:* Uses `$json.body`, `Date.now()`.
    - *Input/Output Connections:* Input from `Set Intake Configuration`; output to `Check Payload Validity`.
  - **Check Payload Validity**
    - *Type and Technical Role:* If node routing valid vs. invalid payloads.
    - *Configuration Choices:* Validates presence of `client_id` and numeric temperature or sensor faults.
    - *Input/Output Connections:* Input from `Normalize Alert Payload`; outputs to `Read Client and Contract Data` (true) or `Return 400 Bad Payload` (false).
  - **Return 400 Bad Payload**
    - *Type and Technical Role:* Respond to Webhook node.
    - *Configuration Choices:* Returns HTTP 400 status with a JSON error body detailing missing required parameters.
    - *Input/Output Connections:* Input from `Check Payload Validity` (false path); output none (terminates request).

---

#### 2.2 Client SLA Lookup
- **Overview:** Connects to Google Sheets to verify client identifiers against contract databases, pulls specific threshold boundaries, and combines operational metadata with the ticket payload.
- **Nodes Involved:**
  - `Read Client and Contract Data`
  - `Combine Client Data`
  - `Verify Client Information`
  - `Record Unknown Client`
  - `Return 404 Client Not Found`

- **Node Details:**
  - **Read Client and Contract Data**
    - *Type and Technical Role:* Google Sheets node.
    - *Configuration Choices:* Performs a lookup on the `Clients_SLA` tab filtered by `client_id`.
    - *Input/Output Connections:* Input from `Check Payload Validity`; output to `Combine Client Data`.
  - **Combine Client Data**
    - *Type and Technical Role:* Code node.
    - *Configuration Choices:* Merges contract constraints, critical temperatures, allowed thresholds, service zones, and computes temperature deltas.
    - *Input/Output Connections:* Input from `Read Client and Contract Data`; output to `Verify Client Information`.
  - **Verify Client Information**
    - *Type and Technical Role:* If node checking if the client was found.
    - *Configuration Choices:* Evaluates `client_found` boolean.
    - *Input/Output Connections:* Input from `Combine Client Data`; outputs to `Check Defrost Cycle Status` (true) or `Record Unknown Client` (false).
  - **Record Unknown Client**
    - *Type and Technical Role:* Google Sheets node.
    - *Configuration Choices:* Appends an audit trail record to the `Escalation_Audit_Log` tab.
    - *Input/Output Connections:* Input from `Verify Client Information` (false); output to `Return 404 Client Not Found`.
  - **Return 404 Client Not Found**
    - *Type and Technical Role:* Respond to Webhook node.
    - *Configuration Choices:* Responds with HTTP status code 404 and a JSON payload noting the unknown client ID.
    - *Input/Output Connections:* Input from `Record Unknown Client`; output none.

---

#### 2.3 Defrost Filtering, Severity Triage & De-duplication
- **Overview:** Evaluates whether temperature anomalies are part of routine defrost cycles, calculates incident severity levels (P1–P3) and SLA deadlines, and suppresses duplicate alerts for open tickets.
- **Nodes Involved:**
  - `Check Defrost Cycle Status`
  - `Append Defrost Observation Log`
  - `Return 202 Observation Accepted`
  - `Assess Alert Severity`
  - `Fetch Open Tickets`
  - `Identify Duplicate Alerts`
  - `Check Alert Duplicity`
  - `Record Suppressed Duplicates`
  - `Return 200 Duplicate Acknowledged`

- **Node Details:**
  - **Check Defrost Cycle Status**
    - *Type and Technical Role:* If node evaluating defrost parameters.
    - *Configuration Choices:* Checks if `defrost_active` is true and temperature delta is within acceptable limits ($\le 8^\circ\text{C}$).
    - *Input/Output Connections:* Input from `Verify Client Information`; outputs to `Append Defrost Observation Log` (true) or `Assess Alert Severity` (false).
  - **Append Defrost Observation Log & Return 202 Observation Accepted**
    - *Type and Technical Role:* Google Sheets and Webhook response nodes.
    - *Configuration Choices:* Appends log entries to `Escalation_Audit_Log` and returns HTTP 202 confirming the observation without dispatching technicians.
  - **Assess Alert Severity**
    - *Type and Technical Role:* Code node calculating priority tiers.
    - *Configuration Choices:* Assigns `P1_CRITICAL`, `P2_HIGH`, or `P3_NORMAL` based on temperature deltas, product classifications (e.g., pharmaceuticals), and rate-of-change metrics, computing exact SLA deadlines.
    - *Input/Output Connections:* Input from `Check Defrost Cycle Status`; output to `Fetch Open Tickets`.
  - **Fetch Open Tickets & Identify Duplicate Alerts / Check Alert Duplicity**
    - *Type and Technical Role:* Google Sheets, Code, and If nodes.
    - *Configuration Choices:* Queries open tickets for the given client to detect duplicates on the same equipment. Suppresses duplicates if found by writing to audit logs and returning HTTP 200.

---

#### 2.4 Technician Matching & Dispatch
- **Overview:** Queries the technician roster, matches certified and available personnel based on geographic zones, creates new dispatch records, locks technicians as busy, and dispatches SMS alerts to both technicians and client dispatch desks.
- **Nodes Involved:**
  - `Fetch Technician List`
  - `Match Technician to Task`
  - `Verify Technician Assignment`
  - `Log Unassigned Ticket`
  - `Notify On-Call Lead via SMS`
  - `Return 202 Unassigned Ticket`
  - `Compile Ticket Information`
  - `Assign Ticket Timestamps`
  - `Append New Ticket Record`
  - `Update Technician Status`
  - `Send Dispatch SMS to Technician`
  - `Alert Client Dispatch Desk via SMS`
  - `Record Dispatch Activity`
  - `Return 200 Dispatch Confirmed`

- **Node Details:**
  - **Fetch Technician List & Match Technician to Task**
    - *Type and Technical Role:* Google Sheets and Code nodes.
    - *Configuration Choices:* Filters active, available technicians matching required certifications, prioritizing those in the same service zone.
    - *Edge Cases/Failures:* No technician available triggers fallback path (`Log Unassigned Ticket` and SMS notification to the on-call lead).
  - **Compile Ticket Information & Assign Ticket Timestamps**
    - *Type and Technical Role:* Code and Set nodes.
    - *Configuration Choices:* Formulates rich SMS body templates containing interactive callback URLs for accepting, declining, arriving on-site, or resolving the ticket.
  - **Append New Ticket Record & Update Technician Status**
    - *Type and Technical Role:* Google Sheets nodes.
    - *Configuration Choices:* Appends the new ticket entry to `Tickets_Dispatch` and updates the assigned technician's status to `ON_JOB`.
  - **Send Dispatch SMS to Technician & Alert Client Dispatch Desk via SMS**
    - *Type and Technical Role:* Twilio nodes.
    - *Configuration Choices:* Sends formatted SMS messages using credentials from intake configuration.
  - **Record Dispatch Activity & Return 200 Dispatch Confirmed**
    - *Type and Technical Role:* Google Sheets and Webhook response nodes.
    - *Configuration Choices:* Logs successful dispatch to audit logs and returns HTTP 200 confirmation.

---

#### 2.5 Technician Callback Intake
- **Overview:** Exposes a webhook endpoint (`/tech-ack`) for technicians to interactively update incident statuses via SMS links, validates token integrity, updates database states, and renders HTML confirmation pages.
- **Nodes Involved:**
  - `Receive Technician Callback`
  - `Decode Callback Data`
  - `Retrieve Ticket Info`
  - `Calculate State Changes`
  - `Modify Ticket Status`
  - `Manage Technician Availability`
  - `Log Callback to Sheet`
  - `Create Confirmation Page Content`
  - `Return Confirmation Page`

- **Node Details:**
  - **Receive Technician Callback**
    - *Type and Technical Role:* Webhook node listening on path `tech-ack`.
    - *Configuration Choices:* Receives query parameters (`ticket_id`, `tech_id`, `action`).
  - **Decode Callback Data & Calculate State Changes**
    - *Type and Technical Role:* Code nodes.
    - *Configuration Choices:* Validates actions (`accept`, `decline`, `onsite`, `resolved`), prevents stale links from unassigned technicians, calculates timestamps, and sets appropriate technician availability states (`AVAILABLE` vs `ON_JOB`).
  - **Modify Ticket Status & Manage Technician Availability**
    - *Type and Technical Role:* Google Sheets nodes.
    - *Configuration Choices:* Updates rows in `Tickets_Dispatch` and `Technicians`.
  - **Create Confirmation Page Content & Return Confirmation Page**
    - *Type and Technical Role:* Code and Respond to Webhook nodes.
    - *Configuration Choices:* Renders a mobile-friendly HTML confirmation page confirming state updates with an explicit `text/html` header.

---

#### 2.6 SLA Heartbeat & Escalation Engine
- **Overview:** Executes every 5 minutes to evaluate SLA compliance progress across all active tickets, issuing 50% threshold warnings, reassigning ignored or declined tickets, recording audit logs, and initiating automated Twilio voice calls for critical escalations.
- **Nodes Involved:**
  - `Schedule Every 5 Minutes`
  - `Set Heartbeat Configuration`
  - `Fetch All Ticket Data`
  - `Retrieve Technician Roster`
  - `Evaluate SLA Metrics`
  - `Filter SLA Actionable Tickets`
  - `Apply Ticket Updates`
  - `Refresh Technician Updates`
  - `Apply Roster Modifications`
  - `Refresh Alert Data`
  - `Dispatch Escalation SMS`
  - `Document Escalation Log`
  - `Evaluate Voice Call Needs`
  - `Initiate Escalation Call`

- **Node Details:**
  - **Schedule Every 5 Minutes**
    - *Type and Technical Role:* Schedule Trigger node executing periodically.
  - **Evaluate SLA Metrics**
    - *Type and Technical Role:* Complex Code node inspecting ticket elapsed times against SLA deadlines, managing acknowledgement timeouts, handling reassignments, and building TwiML instructions for voice calls.
  - **Dispatch Escalation SMS & Initiate Escalation Call**
    - *Type and Technical Role:* Twilio nodes.
    - *Configuration Choices:* Sends SMS warnings/reassignments and places outbound voice calls (`resource: call`) using generated TwiML strings.

---

#### 2.7 Error Handling
- **Overview:** Captures unhandled workflow exceptions globally, formats error notifications, texts system administrators via Twilio, and records error events to audit logs.
- **Nodes Involved:**
  - `Error Detection Trigger`
  - `Setup Error Configuration`
  - `Create Error Notification Message`
  - `Alert Admin of Workflow Error`
  - `Record Error to Audit Sheet`

- **Node Details:**
  - **Error Detection Trigger**
    - *Type and Technical Role:* Error Trigger node acting as the workflow catch-all.
  - **Alert Admin of Workflow Error & Record Error to Audit Sheet**
    - *Type and Technical Role:* Twilio and Google Sheets nodes.
    - *Configuration Choices:* Dispatches failure notifications to administrative phone numbers and appends entries to `Escalation_Audit_Log`.

---

#### 2.8 Daily Reporting
- **Overview:** Executes daily at 08:00 to gather ticket statistics from the past 24 hours, computes compliance metrics, archives daily performance snapshots to Google Sheets, and distributes formatted HTML performance reports via Gmail.
- **Nodes Involved:**
  - `Schedule Daily 08:00 Report`
  - `Configure Report Settings`
  - `Gather Report Ticket Data`
  - `Analyze SLA Compliance`
  - `Generate HTML Report`
  - `Save Daily Snapshot`
  - `Send Operations Report Email`

- **Node Details:**
  - **Analyze SLA Compliance & Generate HTML Report**
    - *Type and Technical Role:* Code nodes.
    - *Configuration Choices:* Computes 24-hour totals, severity breakdowns, average response/site arrival times, SLA adherence percentages, and builds a styled HTML summary table.
  - **Send Operations Report Email**
    - *Type and Technical Role:* Gmail node.
    - *Configuration Choices:* Sends emails via OAuth2 authentication to configured report recipients with subject lines reflecting daily compliance percentages.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive ColdChain Alert | webhook | Inbound webhook trigger for sensor alerts | None | Set Intake Configuration | Receive & validate alert |
| Set Intake Configuration | set | Defines local runtime variables and phone numbers | Receive ColdChain Alert | Normalize Alert Payload | Receive & validate alert |
| Normalize Alert Payload | code | Standardizes incoming webhook payload into ticket draft | Set Intake Configuration | Check Payload Validity | Receive & validate alert |
| Check Payload Validity | if | Validates required fields and payload structure | Normalize Alert Payload | Read Client and Contract Data, Return 400 Bad Payload | Receive & validate alert |
| Return 400 Bad Payload | respondToWebhook | Returns 400 error response for invalid payloads | Check Payload Validity | None | Receive & validate alert |
| Read Client and Contract Data | googleSheets | Queries client contracts and setpoints | Check Payload Validity | Combine Client Data | Client lookup & contract context |
| Combine Client Data | code | Merges contract rules and calculates temperature delta | Read Client and Contract Data | Verify Client Information | Client lookup & contract context |
| Verify Client Information | if | Checks whether client ID exists in system | Combine Client Data | Check Defrost Cycle Status, Record Unknown Client | Client lookup & contract context |
| Record Unknown Client | googleSheets | Logs unknown client attempt to audit sheet | Verify Client Information | Return 404 Client Not Found | Unknown client rejection |
| Return 404 Client Not Found | respondToWebhook | Returns 404 error response for unknown clients | Record Unknown Client | None | Unknown client rejection |
| Check Defrost Cycle Status | if | Filters out routine defrost-cycle spikes | Verify Client Information | Append Defrost Observation Log, Assess Alert Severity | Defrost filter, severity scoring & duplicate check |
| Append Defrost Observation Log | googleSheets | Logs defrost observation event | Check Defrost Cycle Status | Return 202 Observation Accepted | Defrost filter, severity scoring & duplicate check |
| Return 202 Observation Accepted | respondToWebhook | Returns 202 acceptance response for defrost observations | Append Defrost Observation Log | None | Defrost filter, severity scoring & duplicate check |
| Assess Alert Severity | code | Assigns P1-P3 severity and calculates SLA deadlines | Check Defrost Cycle Status | Fetch Open Tickets | Defrost filter, severity scoring & duplicate check |
| Fetch Open Tickets | googleSheets | Retrieves open tickets for duplicate check | Assess Alert Severity | Identify Duplicate Alerts | Defrost filter, severity scoring & duplicate check |
| Identify Duplicate Alerts | code | Determines if alert is a duplicate of an open ticket | Fetch Open Tickets | Check Alert Duplicity | Defrost filter, severity scoring & duplicate check |
| Check Alert Duplicity | if | Routes duplicate vs new alerts | Identify Duplicate Alerts | Record Suppressed Duplicates, Fetch Technician List | Defrost filter, severity scoring & duplicate check |
| Record Suppressed Duplicates | googleSheets | Logs suppressed duplicate alerts | Check Alert Duplicity | Return 200 Duplicate Acknowledged | Defrost filter, severity scoring & duplicate check |
| Return 200 Duplicate Acknowledged | respondToWebhook | Returns 200 response for suppressed duplicates | Record Suppressed Duplicates | None | Defrost filter, severity scoring & duplicate check |
| Fetch Technician List | googleSheets | Loads technician roster | Check Alert Duplicity | Match Technician to Task | Duplicate handling & technician matching |
| Match Technician to Task | code | Matches certified available technician (zone preference) | Fetch Technician List | Verify Technician Assignment | Duplicate handling & technician matching |
| Verify Technician Assignment | if | Checks if a technician was successfully matched | Match Technician to Task | Compile Ticket Information, Log Unassigned Ticket | Duplicate handling & technician matching |
| Log Unassigned Ticket | googleSheets | Logs ticket as unassigned | Verify Technician Assignment | Notify On-Call Lead via SMS | Dispatch ticket & notify |
| Notify On-Call Lead via SMS | twilio | Alerts on-call lead via SMS when no tech is available | Log Unassigned Ticket | Return 202 Unassigned Ticket | Dispatch ticket & notify |
| Return 202 Unassigned Ticket | respondToWebhook | Returns 202 response for unassigned tickets | Notify On-Call Lead via SMS | None | Dispatch ticket & notify |
| Compile Ticket Information | code | Builds formatted SMS templates with action links | Verify Technician Assignment | Assign Ticket Timestamps | Dispatch ticket & notify |
| Assign Ticket Timestamps | set | Sets initial dispatch timestamps and metadata | Compile Ticket Information | Append New Ticket Record | Dispatch ticket & notify |
| Append New Ticket Record | googleSheets | Creates new ticket record in dispatch sheet | Assign Ticket Timestamps | Update Technician Status | Dispatch ticket & notify |
| Update Technician Status | googleSheets | Locks assigned technician as ON_JOB | Append New Ticket Record | Send Dispatch SMS to Technician | Dispatch ticket & notify |
| Send Dispatch SMS to Technician | twilio | Sends dispatch SMS to assigned technician | Update Technician Status | Alert Client Dispatch Desk via SMS | Dispatch ticket & notify |
| Alert Client Dispatch Desk via SMS | twilio | Sends alert SMS to client emergency contact | Send Dispatch SMS to Technician | Record Dispatch Activity | Dispatch ticket & notify |
| Record Dispatch Activity | googleSheets | Logs successful dispatch to audit log | Alert Client Dispatch Desk via SMS | Return 200 Dispatch Confirmed | Dispatch ticket & notify |
| Return 200 Dispatch Confirmed | respondToWebhook | Returns 200 response confirming dispatch | Record Dispatch Activity | None | Dispatch ticket & notify |
| Receive Technician Callback | webhook | Webhook endpoint receiving technician status updates | None | Decode Callback Data | Technician callback intake |
| Decode Callback Data | code | Parses and validates technician action query parameters | Receive Technician Callback | Retrieve Ticket Info | Technician callback intake |
| Retrieve Ticket Info | googleSheets | Fetches ticket details for callback verification | Decode Callback Data | Calculate State Changes | Technician callback intake |
| Calculate State Changes | code | Calculates new status, timestamps, and availability | Retrieve Ticket Info | Modify Ticket Status | Update state & confirm |
| Modify Ticket Status | googleSheets | Updates ticket status in dispatch sheet | Calculate State Changes | Manage Technician Availability | Update state & confirm |
| Manage Technician Availability | googleSheets | Frees or locks technician availability | Modify Ticket Status | Log Callback to Sheet | Update state & confirm |
| Log Callback to Sheet | googleSheets | Appends callback event to audit log | Manage Technician Availability | Create Confirmation Page Content | Update state & confirm |
| Create Confirmation Page Content | code | Generates HTML confirmation response page | Log Callback to Sheet | Return Confirmation Page | Update state & confirm |
| Return Confirmation Page | respondToWebhook | Returns HTML confirmation page to technician browser | Create Confirmation Page Content | None | Update state & confirm |
| Schedule Every 5 Minutes | scheduleTrigger | Triggers heartbeat check every 5 minutes | None | Set Heartbeat Configuration | Heartbeat trigger & data load |
| Set Heartbeat Configuration | set | Defines heartbeat and escalation settings | Schedule Every 5 Minutes | Fetch All Ticket Data | Heartbeat trigger & data load |
| Fetch All Ticket Data | googleSheets | Fetches all tickets for SLA evaluation | Set Heartbeat Configuration | Retrieve Technician Roster | Heartbeat trigger & data load |
| Retrieve Technician Roster | googleSheets | Fetches full technician roster for reassignments | Fetch All Ticket Data | Evaluate SLA Metrics | Heartbeat trigger & data load |
| Evaluate SLA Metrics | code | Evaluates SLA progress, warnings, and reassignments | Retrieve Technician Roster | Filter SLA Actionable Tickets | Evaluate SLA & apply updates |
| Filter SLA Actionable Tickets | filter | Filters actionable ticket items | Evaluate SLA Metrics | Apply Ticket Updates | Evaluate SLA & apply updates |
| Apply Ticket Updates | googleSheets | Updates ticket records with escalation states | Filter SLA Actionable Tickets | Refresh Technician Updates | Evaluate SLA & apply updates |
| Refresh Technician Updates | code | Refreshes technician update payloads | Apply Ticket Updates | Apply Roster Modifications | Evaluate SLA & apply updates |
| Apply Roster Modifications | googleSheets | Updates roster status for unassigned/reassigned techs | Refresh Technician Updates | Refresh Alert Data | Escalation SMS, audit log & voice call |
| Refresh Alert Data | code | Prepares escalation SMS payloads | Apply Roster Modifications | Dispatch Escalation SMS | Escalation SMS, audit log & voice call |
| Dispatch Escalation SMS | twilio | Sends escalation SMS messages | Refresh Alert Data | Document Escalation Log | Escalation SMS, audit log & voice call |
| Document Escalation Log | googleSheets | Logs escalation event to audit log | Dispatch Escalation SMS | Evaluate Voice Call Needs | Escalation SMS, audit log & voice call |
| Evaluate Voice Call Needs | code | Filters tickets requiring phone escalation | Document Escalation Log | Initiate Escalation Call | Escalation SMS, audit log & voice call |
| Initiate Escalation Call | twilio | Places outbound voice call via Twilio | Evaluate Voice Call Needs | None | Escalation SMS, audit log & voice call |
| Error Detection Trigger | errorTrigger | Triggers on workflow execution failure | None | Setup Error Configuration | Workflow error alerting |
| Setup Error Configuration | set | Configures administrator phone numbers for error alerts | Error Detection Trigger | Create Error Notification Message | Workflow error alerting |
| Create Error Notification Message | code | Formats error notification text | Setup Error Configuration | Alert Admin of Workflow Error | Workflow error alerting |
| Alert Admin of Workflow Error | twilio | Sends error notification SMS to administrator | Create Error Notification Message | Record Error to Audit Sheet | Workflow error alerting |
| Record Error to Audit Sheet | googleSheets | Appends workflow error to audit log | Alert Admin of Workflow Error | None | Workflow error alerting |
| Schedule Daily 08:00 Report | scheduleTrigger | Triggers daily report at 08:00 AM | None | Configure Report Settings | Daily report data (08:00) |
| Configure Report Settings | set | Sets report recipient email address | Schedule Daily 08:00 Report | Gather Report Ticket Data | Daily report data (08:00) |
| Gather Report Ticket Data | googleSheets | Fetches ticket data for reporting | Configure Report Settings | Analyze SLA Compliance | Daily report data (08:00) |
| Analyze SLA Compliance | code | Aggregates 24-hour compliance metrics and HTML rows | Gather Report Ticket Data | Generate HTML Report | Report, snapshot & email |
| Generate HTML Report | code | Builds complete HTML email report structure | Analyze SLA Compliance | Save Daily Snapshot | Report, snapshot & email |
| Save Daily Snapshot | googleSheets | Saves daily SLA snapshot record | Generate HTML Report | Send Operations Report Email | Report, snapshot & email |
| Send Operations Report Email | gmail | Emails daily SLA report via Gmail | Save Daily Snapshot | None | Report, snapshot & email |

---

### 4. Reproducing the Workflow from Scratch

To recreate this workflow manually in n8n without using the JSON, follow these sequential steps:

1. **Create Entry Points & Triggers:**
   - Create a **Webhook** node named `Receive ColdChain Alert` (`POST`, path: `coldchain-alert`).
   - Create a **Webhook** node named `Receive Technician Callback` (`GET`, path: `tech-ack`).
   - Create a **Schedule Trigger** node named `Schedule Every 5 Minutes` (interval: every 1 minute/minute field).
   - Create a **Schedule Trigger** node named `Schedule Daily 08:00 Report` (interval: hour 8).
   - Create an **Error Trigger** node named `Error Detection Trigger`.

2. **Configure Global Settings & Intake:**
   - Add **Set** nodes (`Set Intake Configuration`, `Set Heartbeat Configuration`, `Setup Error Configuration`, `Configure Report Settings`) to define local variables (Twilio sender phone numbers, base URLs, ops manager phone, on-call lead phone, ack timeout minutes, and report email addresses). Connect respective triggers to these configuration nodes.

3. **Build Inbound Alert Validation & Client Lookup:**
   - Add a **Code** node (`Normalize Alert Payload`) to parse incoming JSON payloads, generate ticket IDs (`TCK-`), and set validation flags.
   - Add an **If** node (`Check Payload Validity`) to route valid payloads to Google Sheets.
   - Add a **Google Sheets** node (`Read Client and Contract Data`) connected to your Google Sheet document ID, using the `Clients_SLA` sheet with operation `get/lookup` by `client_id`.
   - Add a **Code** node (`Combine Client Data`) to merge contract rules and compute temperature deltas.
   - Add an **If** node (`Verify Client Information`). If false, record the unknown client in `Escalation_Audit_Log` (`Record Unknown Client`) and respond with a 404 (`Return 404 Client Not Found`).

4. **Implement Defrost Filtering, Severity Triage & De-duplication:**
   - Add an **If** node (`Check Defrost Cycle Status`) to filter defrost-cycle spikes ($\le 8^\circ\text{C}$). If true, append an observation log (`Append Defrost Observation Log`) and respond with 202 (`Return 202 Observation Accepted`).
   - If false, add a **Code** node (`Assess Alert Severity`) to calculate P1-P3 severity levels and SLA deadlines.
   - Query `Tickets_Dispatch` via **Google Sheets** (`Fetch Open Tickets`), check for duplicates using a **Code** node (`Identify Duplicate Alerts`) and **If** node (`Check Alert Duplicity`). Suppress duplicates by logging to `Escalation_Audit_Log` (`Record Suppressed Duplicates`) and returning 200 (`Return 200 Duplicate Acknowledged`).

5. **Build Technician Matching & Dispatch:**
   - Query the `Technicians` sheet using **Google Sheets** (`Fetch Technician List`).
   - Add a **Code** node (`Match Technician to Task`) to filter available technicians matching required certifications, prioritizing matching service zones.
   - Add an **If** node (`Verify Technician Assignment`). 
     - *If no tech available:* Log unassigned ticket (`Log Unassigned Ticket`), notify on-call lead via Twilio (`Notify On-Call Lead via SMS`), and return 202 (`Return 202 Unassigned Ticket`).
     - *If tech found:* Compile message bodies with callback links (`Compile Ticket Information`), assign timestamps (`Assign Ticket Timestamps`), append to `Tickets_Dispatch` (`Append New Ticket Record`), update technician status to `ON_JOB` (`Update Technician Status`), send Twilio SMS to technician and client dispatch desk (`Send Dispatch SMS to Technician`, `Alert Client Dispatch Desk via SMS`), log activity (`Record Dispatch Activity`), and return 200 (`Return 200 Dispatch Confirmed`).

6. **Implement Technician Callback Intake:**
   - Connect `Receive Technician Callback` to a **Code** node (`Decode Callback Data`), query ticket info from `Tickets_Dispatch` (`Retrieve Ticket Info`), calculate state changes (`Calculate State Changes`), update ticket status (`Modify Ticket Status`), update technician availability (`Manage Technician Availability`), append audit logs (`Log Callback to Sheet`), generate HTML content (`Create Confirmation Page Content`), and respond via Webhook (`Return Confirmation Page`).

7. **Implement Heartbeat & Escalation Engine:**
   - Connect `Schedule Every 5 Minutes` to fetch all tickets (`Fetch All Ticket Data`) and technician rosters (`Retrieve Technician Roster`).
   - Add a **Code** node (`Evaluate SLA Metrics`) to evaluate breaches, 50% warnings, and reassignments.
   - Filter actionable tickets (`Filter SLA Actionable Tickets`), update ticket and roster records (`Apply Ticket Updates`, `Apply Roster Modifications`), dispatch escalation SMS messages (`Dispatch Escalation SMS`, `Document Escalation Log`), evaluate voice call requirements (`Evaluate Voice Call Needs`), and place outbound Twilio voice calls (`Initiate Escalation Call`).

8. **Configure Error Handling & Daily Reporting:**
   - Connect `Error Detection Trigger` to format error messages (`Create Error Notification Message`), alert administrators via Twilio (`Alert Admin of Workflow Error`), and log errors (`Record Error to Audit Sheet`).
   - Connect `Schedule Daily 08:00 Report` to gather ticket data (`Gather Report Ticket Data`), analyze SLA compliance (`Analyze SLA Compliance`), generate HTML reports (`Generate HTML Report`), save snapshots to `SLA_Daily_Snapshots` (`Save Daily Snapshot`), and email reports via Gmail (`Send Operations Report Email`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video Walkthrough Tutorial | YouTube Video: https://youtu.be/s79Y5rty2WM |
| Required Google Sheets Tabs | `Clients_SLA`, `Technicians`, `Tickets_Dispatch`, `Escalation_Audit_Log`, `SLA_Daily_Snapshots` |