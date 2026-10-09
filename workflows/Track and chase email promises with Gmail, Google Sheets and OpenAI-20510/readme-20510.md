Track and chase email promises with Gmail, Google Sheets and OpenAI

https://n8nworkflows.xyz/workflows/track-and-chase-email-promises-with-gmail--google-sheets-and-openai-20510


# Track and chase email promises with Gmail, Google Sheets and OpenAI

### 1. Workflow Overview

This workflow automates the tracking, logging, and follow-up management of promises made by external contacts via email. It extracts commitments using OpenAI, maintains a stateful ledger in Google Sheets, generates daily compliance sweeps and follow-up drafts for operator approval, and issues real-time alerts or digests through Gmail.

The workflow logic is categorized into the following functional blocks:
- **1.1 Configuration & Entry Control:** Initializes global parameters (spreadsheet IDs, email targets, labels) and splits execution paths based on the schedule (hourly intake vs. daily sweep at 08:00).
- **1.2 Inbox Intake & AI Processing:** Polls recent unlabelled inbox messages, strips HTML formatting, formats structured extraction prompts, queries OpenAI to extract commitments, and parses strict JSON verdicts.
- **1.3 Ledger Updates & Fulfillment Routing:** Directs extracted data into commitments (upserts) or fulfills existing active rows when completion responses are detected, marking processed emails with an exclusion label.
- **1.4 Daily Aging Sweep & Escalation:** Reads active ledger items during the 08:00 execution, calculates calendar-day aging (due today, overdue, 3+ days overdue), triggers escalations, and builds daily digests.
- **1.5 Chase Draft Generation & Queueing:** Constructs follow-up email drafts for due and overdue items using OpenAI, stages them in an approval queue, notifies the owner, and updates the reminder counters.
- **1.6 First-Run Setup & Offline Testing:** Provides standalone lanes to provision the Google Sheets ledger structure automatically and execute offline mock data simulations for testing without credentials.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Configuration & Entry Control

- **Overview:** Initializes global variables, manages runtime triggers on an hourly cadence, and routes execution to either the daily sweep branch (at 08:00) or the hourly inbox intake branch.
- **Nodes Involved:** 
  - `Hourly tick (sweep at 08:00)`
  - `Workflow Config`
  - `Lane entry`

- **Node Details:**
  - **Hourly tick (sweep at 08:00)**
    - *Type and Technical Role:* Schedule Trigger (n8n-nodes-base.scheduleTrigger). Initiates execution every hour.
    - *Configuration:* Interval configured for hourly firing.
    - *Input/Output:* Output connects to `Workflow Config`.
    - *Edge Cases/Failures:* Timezone offsets must align with n8n system settings to ensure the 08:00 sweep occurs at the intended local hour.
  - **Workflow Config**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Serves as the single configuration source supplying sheet IDs, target owner emails, and Gmail label IDs.
    - *Configuration:* Hardcoded placeholder fields (`sheet_id`, `owner_email`, `label_id`) edited by the user during setup.
    - *Input/Output:* Input from `Hourly tick (sweep at 08:00)`; output connects to `Lane entry`.
    - *Edge Cases/Failures:* Missing or invalid placeholder values break downstream Google Sheets and Gmail API calls.
  - **Lane entry**
    - *Type and Technical Role:* Switch Node (n8n-nodes-base.switch). Evaluates the current hour and bifurcates the workflow.
    - *Configuration:* Rule condition routes execution based on `{{ $now.hour }}` matching `8` (outputs to Daily Sweep) or not matching `8` (outputs to Inbox Intake).
    - *Input/Output:* Input from `Workflow Config`; outputs connect to `Read ledger rows` (sweep) and `Fetch new inbox emails` (intake).
    - *Edge Cases/Failures:* System timezone mismatches may cause the 08:00 branch to trigger at unexpected times.

---

#### Block 1.2: Inbox Intake & AI Processing

- **Overview:** Fetches recent unprocessed inbox messages, strips raw MIME/HTML formatting, builds system-enforced extraction prompts, and submits data to OpenAI for structured promise evaluation.
- **Nodes Involved:**
  - `Fetch new inbox emails`
  - `Build extraction prompt`
  - `Extract promises (AI)`
  - `Parse commitment verdict`

- **Node Details:**
  - **Fetch new inbox emails**
    - *Type and Technical Role:* Gmail Node (n8n-nodes-base.gmail). Retrieves raw inbox messages matching specific criteria.
    - *Configuration:* `operation: getAll`, `limit: 25`, `simple: false`, query filter `in:inbox -label:promises-processed newer_than:1d`. Retry enabled with 3 maximum tries and 1500ms delay.
    - *Input/Output:* Input from `Lane entry`; output connects to `Build extraction prompt`.
    - *Edge Cases/Failures:* Authentication revocation or strict API rate limits.
  - **Build extraction prompt**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Cleans HTML, extracts email headers, and structures system/user prompts for OpenAI.
    - *Configuration:* Uses JavaScript functions to strip scripts/styles and parse sender metadata.
    - *Input/Output:* Input from `Fetch new inbox emails`; output connects to `Extract promises (AI)`.
    - *Edge Cases/Failures:* Malformed email payloads or excessively large bodies truncated at 2000 characters.
  - **Extract promises (AI)**
    - *Type and Technical Role:* OpenAI Node (n8n-nodes-base.openAi). Sends structured payloads to OpenAI for natural language analysis.
    - *Configuration:* Model set to `gpt-4o-mini`, resource set to `chat`. Retry enabled with 3 maximum tries. Uses OpenAI API credentials.
    - *Input/Output:* Input from `Build extraction prompt`; output connects to `Parse commitment verdict`.
    - *Edge Cases/Failures:* API timeouts, token limit overflows, or insufficient credits.
  - **Parse commitment verdict**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Parses JSON responses from OpenAI, handling edge cases such as unparseable output, missing due dates, fulfillment notices, or duplicates.
    - *Configuration:* Implements error catching via `try/catch` on JSON parsing and constructs a deterministic deduplication hash using a custom hashing loop.
    - *Input/Output:* Input from `Extract promises (AI)`; output connects to `Route intake`.
    - *Edge Cases/Failures:* AI returning unstructured text instead of strict JSON triggers alerting routines.

---

#### Block 1.3: Ledger Updates & Fulfillment Routing

- **Overview:** Routes parsed commitment states (new commitment, fulfillment, skip, or alert), updates Google Sheets state records, and marks processed emails with an exclusion label.
- **Nodes Involved:**
  - `Route intake`
  - `Merge write streams`
  - `Live write?`
  - `Write kind?`
  - `Find open row`
  - `Pick open row`
  - `Row matched?`
  - `Mark done in ledger`
  - `Write ledger row`
  - `Demo: record ledger write`
  - `Skip is live?`
  - `Compose alert`
  - `Alert is live?`
  - `Email owner: intake alert`
  - `Label email processed`

- **Node Details:**
  - **Route intake**
    - *Type and Technical Role:* Switch Node (n8n-nodes-base.switch). Directs items based on their status (`commitment`, `fulfillment`, `skipped`, `alerted`).
    - *Input/Output:* Input from `Parse commitment verdict`; outputs connect to write merges, skip validators, and alert composers.
  - **Merge write streams**
    - *Type and Technical Role:* Merge Node (n8n-nodes-base.merge). Combines commitment and fulfillment streams.
    - *Input/Output:* Inputs from `Route intake`; output connects to `Live write?`.
  - **Live write?**
    - *Type and Technical Role:* If Node (n8n-nodes-base.if). Determines whether items are from live chat flows or offline demo runs.
    - *Configuration:* Checks if `{{ $json.live }}` equals `true`.
    - *Input/Output:* Input from `Merge write streams`; outputs connect to `Write kind?` (true) and `Demo: record ledger write` (false).
  - **Write kind?**
    - *Type and Technical Role:* Switch Node (n8n-nodes-base.switch). Separates fulfillments from general commitments.
    - *Input/Output:* Input from `Live write?`; outputs connect to `Find open row` (fulfillment) and `Write ledger row` (default/commitment).
  - **Find open row**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Reads current commitments to evaluate matching fulfillment rows.
    - *Configuration:* `operation: get` or read list on `Commitments` sheet.
    - *Input/Output:* Input from `Write kind?`; output connects to `Pick open row`.
  - **Pick open row**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Identifies the oldest open row matching the fulfillment sender.
    - *Input/Output:* Input from `Find open row`; output connects to `Row matched?`.
  - **Row matched?**
    - *Type and Technical Role:* If Node (n8n-nodes-base.if). Evaluates whether a matching open dedupe key was found.
    - *Input/Output:* Input from `Pick open row`; outputs connect to `Mark done in ledger` (true) and `Label email processed` (false).
  - **Mark done in ledger**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Updates status to `done` on matched rows.
    - *Configuration:* `operation: update`, matching columns set to `dedupe_key`. Error output configured to continue on error (`continueErrorOutput`).
    - *Input/Output:* Input from `Row matched?`; output connects to `Label email processed`.
  - **Write ledger row**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Appends or updates commitment records.
    - *Configuration:* `operation: appendOrUpdate`, matching columns set to `dedupe_key`. Error output configured to continue on error (`continueErrorOutput`).
    - *Input/Output:* Input from `Write kind?`; output connects to `Label email processed`.
  - **Demo: record ledger write**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Simulates write outputs for offline testing.
    - *Input/Output:* Input from `Live write?`.
  - **Skip is live?**
    - *Type and Technical Role:* If Node (n8n-nodes-base.if). Checks if skipped non-promises require labeling.
    - *Configuration:* Evaluates `{{ $json.live }}` is true.
    - *Input/Output:* Input from `Route intake`; output connects to `Label email processed`.
  - **Compose alert**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Passes through alert payloads.
    - *Input/Output:* Input from `Route intake`; output connects to `Alert is live?`.
  - **Alert is live?**
    - *Type and Technical Role:* If Node (n8n-nodes-base.if). Verifies if alerts originate from live execution.
    - *Input/Output:* Input from `Compose alert`; outputs connect to `Label email processed` and `Email owner: intake alert`.
  - **Email owner: intake alert**
    - *Type and Technical Role:* Gmail Node (n8n-nodes-base.gmail). Sends error or missing-data notifications to the owner.
    - *Configuration:* Sends plain text email to `owner_email`. Error output set to continue regular output.
    - *Input/Output:* Input from `Alert is live?`.
  - **Label email processed**
    - *Type and Technical Role:* Gmail Node (n8n-nodes-base.gmail). Applies the exclusion label (`promises-processed`) to messages.
    - *Configuration:* `operation: addLabels`, label ID read dynamically from `Workflow Config`.
    - *Input/Output:* Inputs from multiple branches (`Write ledger row`, `Mark done in ledger`, `Skip is live?`, `Alert is live?`).

---

#### Block 1.4: Daily Aging Sweep & Escalation

- **Overview:** Reads all ledger entries during the 08:00 execution window, calculates calendar-day aging, classifies items into operational buckets, manages escalations, and issues summaries.
- **Nodes Involved:**
  - `Read ledger rows`
  - `Classify aging`
  - `Sweep digest`
  - `Digest worth sending?`
  - `Email owner: daily digest`
  - `Route sweep`
  - `Escalate to owner`
  - `Mark escalated in ledger`

- **Node Details:**
  - **Read ledger rows**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Reads all records from the `Commitments` tab.
    - *Configuration:* `operation: getAll`, document ID from `Workflow Config`.
    - *Input/Output:* Input from `Lane entry`; output connects to `Classify aging`.
  - **Classify aging**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Performs UTC calendar-day difference calculations to bucket active items into `due-today`, `overdue`, `escalate` (>=3 days), `on-track`, or `closed`.
    - *Configuration:* Uses `Date.parse()` and `$now.toISODate()` to establish delta thresholds.
    - *Input/Output:* Input from `Read ledger rows`; outputs connect to `Route sweep` and `Sweep digest`.
  - **Sweep digest**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Aggregates metrics for the daily review email.
    - *Input/Output:* Input from `Classify aging`; output connects to `Digest worth sending?`.
  - **Digest worth sending?**
    - *Type and Technical Role:* If Node (n8n-nodes-base.if). Suppresses the digest email when no items require attention.
    - *Configuration:* Evaluates `{{ $json.worth_sending }}`.
    - *Input/Output:* Input from `Sweep digest`; outputs connect to `Email owner: daily digest` (true).
  - **Email owner: daily digest**
    - *Type and Technical Role:* Gmail Node (n8n-nodes-base.gmail). Delivers the structured morning status summary.
    - *Configuration:* Plain text email sent to `owner_email`. Uses Gmail OAuth2 credentials.
    - *Input/Output:* Input from `Digest worth sending?`.
  - **Route sweep**
    - *Type and Technical Role:* Switch Node (n8n-nodes-base.switch). Separates items for chase queueing versus direct escalation.
    - *Configuration:* Routes based on `sweep` value (`due-today`, `overdue`, `escalate`).
    - *Input/Output:* Input from `Classify aging`; outputs connect to `Merge chase streams` and `Escalate to owner`.
  - **Escalate to owner**
    - *Type and Technical Role:* Gmail Node (n8n-nodes-base.gmail). Immediately notifies the owner of severely overdue items (>=3 days).
    - *Configuration:* Plain text email sent to `owner_email`.
    - *Input/Output:* Input from `Route sweep`; output connects to `Mark escalated in ledger`.
  - **Mark escalated in ledger**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Updates status to `escalated` to prevent repeated daily alerts.
    - *Configuration:* `operation: update`, matching columns set to `dedupe_key`.
    - *Input/Output:* Input from `Escalate to owner`.

---

#### Block 1.5: Chase Draft Generation & Queueing

- **Overview:** Generates contextual follow-up email drafts for due and overdue commitments using OpenAI, appends them to the approval queue, notifies the operator, and updates ledger states.
- **Nodes Involved:**
  - `Merge chase streams`
  - `Build chase prompt`
  - `Draft chase email (AI)`
  - `Parse chase draft`
  - `Queue chase for approval`
  - `Notify owner of queue`
  - `Mark chased in ledger`

- **Node Details:**
  - **Merge chase streams**
    - *Type and Technical Role:* Merge Node (n8n-nodes-base.merge). Combines due-today and overdue streams into a single processing wave.
    - *Input/Output:* Inputs from `Route sweep`; output connects to `Build chase prompt`.
  - **Build chase prompt**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Constructs tone-adjusted drafting prompts based on delinquency duration.
    - *Input/Output:* Input from `Merge chase streams`; output connects to `Draft chase email (AI)`.
  - **Draft chase email (AI)**
    - *Type and Technical Role:* OpenAI Node (n8n-nodes-base.openAi). Generates follow-up drafts via model calls.
    - *Configuration:* Model set to `gpt-4o-mini`, resource set to `chat`. Uses OpenAI API credentials.
    - *Input/Output:* Input from `Build chase prompt`; output connects to `Parse chase draft`.
  - **Parse chase draft**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Extracts subject and body JSON properties from the AI output while preserving indexed ledger metadata.
    - *Input/Output:* Input from `Draft chase email (AI)`; output connects to `Queue chase for approval`.
  - **Queue chase for approval**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Appends or updates rows in the `Chase-Queue` tab with `pending-approval` status.
    - *Configuration:* `operation: appendOrUpdate`, matching columns set to `dedupe_key`.
    - *Input/Output:* Input from `Parse chase draft`; output connects to `Notify owner of queue`.
  - **Notify owner of queue**
    - *Type and Technical Role:* Gmail Node (n8n-nodes-base.gmail). Sends a notification containing the generated chase draft preview.
    - *Configuration:* Plain text email sent to `owner_email`.
    - *Input/Output:* Input from `Queue chase for approval`; output connects to `Mark chased in ledger`.
  - **Mark chased in ledger**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Advances the commitment state machine (open $\rightarrow$ reminded / chased) and increments chase counters.
    - *Configuration:* `operation: update`, matching columns set to `dedupe_key`.
    - *Input/Output:* Input from `Notify owner of queue`.

---

#### Block 1.6: First-Run Setup & Offline Testing

- **Overview:** Provides tools to automatically provision the Google Sheets ledger schema with tabs and example data, and simulates end-to-end workflows offline.
- **Nodes Involved:**
  - `When clicking 'Test workflow'`
  - `Demo inbox emails`
  - `Demo extract (mirrors AI contract)`
  - `Run digest`
  - `Create promise ledger`
  - `Seed headers and examples`
  - `Route by tab`
  - `Columns for Commitments`
  - `Columns for Chase-Queue`
  - `Write Commitments tab`
  - `Write Chase-Queue tab`
  - `Print sheet id + next step`

- **Node Details:**
  - **When clicking 'Test workflow'**
    - *Type and Technical Role:* Manual Trigger (n8n-nodes-base.manualTrigger). Initiates offline demonstration runs.
    - *Input/Output:* Output connects to `Demo inbox emails`.
  - **Demo inbox emails**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Generates mock inbound emails covering various test scenarios (clear promise, vague promise, fulfillment, repeat, non-promise, bad AI).
    - *Input/Output:* Input from `When clicking 'Test workflow'`; output connects to `Demo extract (mirrors AI contract)`.
  - **Demo extract (mirrors AI contract)**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Mocks deterministic AI extraction results without requiring an active OpenAI key.
    - *Input/Output:* Input from `Demo inbox emails`; outputs connect to `Route intake` and `Run digest`.
  - **Run digest**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Summarizes demo test runs.
    - *Input/Output:* Input from `Demo extract (mirrors AI contract)`.
  - **Create promise ledger**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Creates the primary tracking spreadsheet containing the `Commitments` and `Chase-Queue` tabs.
    - *Configuration:* `resource: spreadsheet`, sheets configured with titles `Commitments` and `Chase-Queue`. Google Sheets credential required.
    - *Input/Output:* Output connects to `Seed headers and examples`.
  - **Seed headers and examples**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Generates initial rows and schema headers for both tabs.
    - *Input/Output:* Input from `Create promise ledger`; output connects to `Route by tab`.
  - **Route by tab**
    - *Type and Technical Role:* Switch Node (n8n-nodes-base.switch). Directs seed rows to their respective spreadsheet tabs.
    - *Input/Output:* Input from `Seed headers and examples`; outputs connect to `Columns for Commitments` and `Columns for Chase-Queue`.
  - **Columns for Commitments**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Removes routing metadata prior to writing.
    - *Input/Output:* Input from `Route by tab`; output connects to `Write Commitments tab`.
  - **Columns for Chase-Queue**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Removes routing metadata prior to writing.
    - *Input/Output:* Input from `Route by tab`; output connects to `Write Chase-Queue tab`.
  - **Write Commitments tab**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Appends seeded example rows to the `Commitments` sheet.
    - *Configuration:* `operation: append`.
    - *Input/Output:* Input from `Columns for Commitments`; output connects to `Print sheet id + next step`.
  - **Write Chase-Queue tab**
    - *Type and Technical Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Appends seeded example rows to the `Chase-Queue` sheet.
    - *Configuration:* `operation: append`.
    - *Input/Output:* Input from `Columns for Chase-Queue`; output connects to `Print sheet id + next step`.
  - **Print sheet id + next step**
    - *Type and Technical Role:* Code Node (n8n-nodes-base.code). Outputs the generated spreadsheet ID and setup instructions.
    - *Input/Output:* Inputs from both sheet write nodes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `START HERE — 4 steps` | n8n-nodes-base.stickyNote | Setup instructions across 4 steps | — | — | # START HERE — 4 steps<br><br>**Step 1** · Attach credentials: Gmail, Google Sheets, OpenAI (3 total).<br><br>**Step 2** · Press play on the orange **SETUP** trigger (Lane D, bottom). It creates your ledger spreadsheet — two tabs, headers, example rows — and prints a `spreadsheet_id`.<br><br>**Step 3** · Create a Gmail label `promises-processed`, copy its label ID. Open **Workflow Config** — paste `spreadsheet_id`, `label_id`, and your `owner_email`. One node; everything reads it.<br><br>**Step 4** · Delete Lane D, set the workflow **Active**. Hourly scans find promises in your inbox; at 08:00 (workflow timezone) the sweep drafts chases into `Chase-Queue` for your approval.<br><br>_Preview first? Press **Test workflow** — the demo lane runs six emails offline._ |
| `Sticky Note` | n8n-nodes-base.stickyNote | Detailed conceptual overview and usage guide | — | — | ## Promise ledger — remember what people owe you, chase it before it goes cold<br><br>**Who it's for:** founders and operators who lose follow-ups in their inbox — vendors who "will send it Friday", partners who owe a document, clients who promised a reply.<br><br>**How it works:** hourly, the inbox is scanned and an AI extracts concrete promises {who, what, due-by}. Each lands in the `Commitments` ledger (deduped on person+promise). At 08:00 the sweep classifies open rows — due today, overdue, 3+ days overdue — and drafts a polite chase email into `Chase-Queue`. You approve the draft, it goes out; the ledger row moves open→reminded→chased→escalated. When the person finally delivers, the intake marks the row `done` and it drops out of your digest.<br><br>**The approval gate:** nothing is ever sent automatically — chase drafts wait for your go-ahead, so the automation can never embarrass you with a wrong chase.<br><br>**Morning digest:** a daily email groups open promises by person with age — you see at a glance who habitually leaves commitments hanging.<br><br>**Setup:** 3 credentials, one button, one paste — see START HERE. |
| `Sticky Note1` | n8n-nodes-base.stickyNote | Documentation for Lane A (inbox intake) | — | — | ### Lane A — inbox intake (runs every hour except 08:00)<br><br>Gmail poll → decode raw MIME → AI extracts promises per message (strict JSON) → parse → route: new commitment → ledger appendOrUpdate on `dedupe_key`; fulfillment reply → oldest open row for that person marked `done`; vague/low-confidence → review alert (emailed); not a promise → skip.<br><br>Processed messages get the `promises-processed` label so they never rescan. |
| `Sticky Note2` | n8n-nodes-base.stickyNote | Documentation for Lane B (daily chase sweep) | — | — | ### Lane B — daily chase sweep (08:00 workflow timezone)<br><br>Read ledger → classify due-today / overdue / >=3d → AI drafts the chase → `Chase-Queue` (pending-approval) + owner notification → the row is marked reminded/chased; >=3d escalates straight to you and is marked `escalated` so it fires once.<br><br>One digest email per run: due/overdue/escalated counts + aging by person. |
| `Sticky Note3` | n8n-nodes-base.stickyNote | Documentation for Lane C (demo lane) | — | — | ### Lane C — demo (no credentials)<br><br>"Test workflow" fires six emails: a clear promise, a vague one (no date), a fulfillment reply, a duplicate, a non-promise, and a simulated bad AI answer — every route runs offline.<br><br>Delete the demo lane when live. |
| `Sticky Note4` | n8n-nodes-base.stickyNote | Warning note regarding manual approval before sending | — | — | ### ⚠ Approval before send<br><br>Chase drafts land in `Chase-Queue` with `pending-approval`. To send one: copy the draft to a Gmail compose (or wire a send node of your own). The workflow deliberately never sends on your behalf. |
| `Sticky Note5` | n8n-nodes-base.stickyNote | Documentation for Lane D (first-run setup) | — | — | ### Lane D — first-run setup (run once)<br><br>Attach a Google Sheets credential, press play on the orange trigger. Creates `Commitments` + `Chase-Queue` tabs with headers and done-marked example rows, prints the `spreadsheet_id` — paste into **Workflow Config**, delete this lane. |
| `When clicking 'Test workflow'` | n8n-nodes-base.manualTrigger | Manual entry point for offline testing | — | `Demo inbox emails` | |
| `Demo inbox emails` | n8n-nodes-base.code | Generates mock inbound emails for testing | `When clicking 'Test workflow'` | `Demo extract (mirrors AI contract)` | |
| `Demo extract (mirrors AI contract)` | n8n-nodes-base.code | Simulates deterministic AI extraction without API keys | `Demo inbox emails` | `Route intake`, `Run digest` | |
| `Run digest` | n8n-nodes-base.code | Compiles summaries for demo runs | `Demo extract (mirrors AI contract)` | — | |
| `Hourly tick (sweep at 08:00)` | n8n-nodes-base.scheduleTrigger | Triggers execution on an hourly schedule | — | `Workflow Config` | |
| `Workflow Config` | n8n-nodes-base.code | Supplies global configuration variables | `Hourly tick (sweep at 08:00)` | `Lane entry` | The only node you edit: spreadsheet_id + label_id from setup + your owner_email. All Sheets and Gmail nodes read from here. |
| `Lane entry` | n8n-nodes-base.switch | Routes execution based on hour (08:00 sweep vs intake) | `Workflow Config` | `Read ledger rows`, `Fetch new inbox emails` | hour==8 → daily sweep; every other hour → inbox intake. |
| `Fetch new inbox emails` | n8n-nodes-base.gmail | Polls unprocessed messages from the inbox | `Lane entry` | `Build extraction prompt` | Unlabeled recent inbox mail, raw format — full body is decoded for the LLM, not just the snippet. |
| `Build extraction prompt` | n8n-nodes-base.code | Normalizes email bodies and formats system prompts | `Fetch new inbox emails` | `Extract promises (AI)` | |
| `Extract promises (AI)` | n8n-nodes-base.openAi | Queries OpenAI for structured commitment extraction | `Build extraction prompt` | `Parse commitment verdict` | Strict-JSON verdict per message: has_promise, promise, due_date, confidence, closes_previous. |
| `Parse commitment verdict` | n8n-nodes-base.code | Parses OpenAI JSON responses and handles validation rules | `Extract promises (AI)` | `Route intake` | |
| `Route intake` | n8n-nodes-base.switch | Directs items based on parsed classification status | `Parse commitment verdict` | `Merge write streams` (x2), `Skip is live?`, `Compose alert` | |
| `Merge write streams` | n8n-nodes-base.merge | Combines commitments and fulfillment writes | `Route intake` | `Live write?` | New commitments and fulfillment closures share one credential gate. |
| `Live write?` | n8n-nodes-base.if | Distinguishes between live writes and offline demo runs | `Merge write streams` | `Write kind?`, `Demo: record ledger write` | Demo items skip real writes; production items write the ledger and label the email. |
| `Write kind?` | n8n-nodes-base.switch | Separates fulfillment closures from general commitments | `Live write?` | `Find open row`, `Write ledger row` | fulfillment → close the matching open row; everything else (commitment) → upsert the ledger. |
| `Write ledger row` | n8n-nodes-base.googleSheets | Upserts commitment rows into the ledger spreadsheet | `Write kind?` | `Label email processed` | appendOrUpdate on dedupe_key (person+promise hash) — repeats update the row, never duplicate it. continueErrorOutput: a failed write must NOT reach the label — the unlabelled message retries next hour. |
| `Find open row` | n8n-nodes-base.googleSheets | Reads ledger rows to evaluate fulfillment matches | `Write kind?` | `Pick open row` | Reads the ledger so the fulfillment can close the right row. |
| `Pick open row` | n8n-nodes-base.code | Selects the oldest active row for fulfillment closing | `Find open row` | `Row matched?` | Closes the OLDEST open row for closes_person; emits nothing when nothing is open. |
| `Row matched?` | n8n-nodes-base.if | Checks if a valid matching open ledger row was located | `Pick open row` | `Mark done in ledger`, `Label email processed` | Matched → close the ledger row; unmatched fulfillment → label anyway so it is not re-scanned every hour. |
| `Mark done in ledger` | n8n-nodes-base.googleSheets | Updates commitment rows to a `done` status | `Row matched?` | `Label email processed` | update semantics — closes the row without touching promise/due_date/confidence. continueErrorOutput: no label without a written ledger row. |
| `Label email processed` | n8n-nodes-base.gmail | Applies the exclusion label to processed inbox messages | `Row matched?`, `Write ledger row`, `Skip is live?`, `Alert is live?`, `Mark done in ledger` | — | Label ID (not name) from Workflow Config. Applied AFTER the ledger write — a failed run re-scans instead of losing the message. |
| `Demo: record ledger write` | n8n-nodes-base.code | Simulates ledger writes during offline demo testing | `Live write?` | — | |
| `Skip is live?` | n8n-nodes-base.if | Verifies if skipped non-promises require labeling | `Route intake` | `Label email processed` | Not-a-promise mail is labelled too (otherwise it is re-fetched and re-sent to the AI every hour for a day). Demo items stop here. |
| `Compose alert` | n8n-nodes-base.code | Passes through alert data for verification | `Route intake` | `Alert is live?` | |
| `Alert is live?` | n8n-nodes-base.if | Checks if alerts originate from live execution | `Compose alert` | `Label email processed`, `Email owner: intake alert` | Demo alerts stop here; live alerts email the owner. |
| `Email owner: intake alert` | n8n-nodes-base.gmail | Sends review alerts for unparseable AI verdicts | `Alert is live?` | — | |
| `Read ledger rows` | n8n-nodes-base.googleSheets | Retrieves all commitment rows from the spreadsheet | `Lane entry` | `Classify aging` | Reads every row — the classifier owns the status filter (open/reminded/chased are active) because a single-column equality can't express the active set. |
| `Classify aging` | n8n-nodes-base.code | Calculates UTC calendar aging and buckets statuses | `Read ledger rows` | `Route sweep`, `Sweep digest` | |
| `Sweep digest` | n8n-nodes-base.code | Compiles operational metrics for the morning digest | `Classify aging` | `Digest worth sending?` | One digest per sweep — fed directly by the classifier wave. |
| `Digest worth sending?` | n8n-nodes-base.if | Suppresses morning digests when no action is needed | `Sweep digest` | `Email owner: daily digest` | Skip the morning email when nothing is due, overdue, escalated or flagged. |
| `Email owner: daily digest` | n8n-nodes-base.gmail | Sends the structured morning status summary | `Digest worth sending?` | — | The sweep digest is delivered here — it is not a dead end. |
| `Route sweep` | n8n-nodes-base.switch | Routes aging items for drafting or direct escalation | `Classify aging` | `Merge chase streams` (x2), `Escalate to owner` | |
| `Merge chase streams` | n8n-nodes-base.merge | Combines due-today and overdue streams | `Route sweep` | `Build chase prompt` | due-today and overdue both route here — one wave into Build chase prompt, so Parse chase draft's index mapping stays correct (a double-wired node runs twice and corrupts the index alignment). |
| `Build chase prompt` | n8n-nodes-base.code | Constructs follow-up drafting prompts based on aging | `Merge chase streams` | `Draft chase email (AI)` | |
| `Draft chase email (AI)` | n8n-nodes-base.openAi | Generates follow-up email drafts using OpenAI | `Build chase prompt` | `Parse chase draft` | Drafts only — lands in Chase-Queue as pending-approval. Never sends. |
| `Parse chase draft` | n8n-nodes-base.code | Parses AI-generated follow-up draft responses | `Draft chase email (AI)` | `Queue chase for approval` | |
| `Queue chase for approval` | n8n-nodes-base.googleSheets | Upserts follow-up drafts into the `Chase-Queue` tab | `Parse chase draft` | `Notify owner of queue` | Owner reviews here — approve by sending the draft, or delete the row. |
| `Notify owner of queue` | n8n-nodes-base.gmail | Notifies the owner that a chase draft is ready | `Queue chase for approval` | `Mark chased in ledger` | Reads the pre-write item — sheet output only carries mapped columns. |
| `Mark chased in ledger` | n8n-nodes-base.googleSheets | Advances commitment states and increments chase counts | `Notify owner of queue` | — | State machine: open → reminded (due-today) / chased (overdue). |
| `Escalate to owner` | n8n-nodes-base.gmail | Sends direct alerts for items overdue by 3+ days | `Route sweep` | `Mark escalated in ledger` | 3+ days overdue skips the draft queue — straight to you. |
| `Mark escalated in ledger` | n8n-nodes-base.googleSheets | Updates status to `escalated` to prevent repeated alerts | `Escalate to owner` | — | escalated rows leave the active set — the alert fires once, not daily. |
| `Create promise ledger` | n8n-nodes-base.googleSheets | Provisions the tracking spreadsheet with two tabs | — | `Seed headers and examples` | One call creates the spreadsheet with both tabs. |
| `Seed headers and examples` | n8n-nodes-base.code | Generates example rows and schema headers | `Create promise ledger` | `Route by tab` | |
| `Route by tab` | n8n-nodes-base.switch | Routes seed records to their respective tabs | `Seed headers and examples` | `Columns for Commitments`, `Columns for Chase-Queue` | |
| `Columns for Commitments` | n8n-nodes-base.code | Strips routing metadata for commitments | `Route by tab` | `Write Commitments tab` | Drops the `tab` routing key — an empty sheet autoMaps headers from item keys. |
| `Columns for Chase-Queue` | n8n-nodes-base.code | Strips routing metadata for chase queue items | `Route by tab` | `Write Chase-Queue tab` | |
| `Write Commitments tab` | n8n-nodes-base.googleSheets | Writes seed rows into the `Commitments` tab | `Columns for Commitments` | `Print sheet id + next step` | |
| `Write Chase-Queue tab` | n8n-nodes-base.googleSheets | Writes seed rows into the `Chase-Queue` tab | `Columns for Chase-Queue` | `Print sheet id + next step` | |
| `Print sheet id + next step` | n8n-nodes-base.code | Outputs the provisioned spreadsheet ID | `Write Commitments tab`, `Write Chase-Queue tab` | — | Output shows the new spreadsheet_id — paste it into the Workflow Config node. |

---

### 4. Reproducing the Workflow from Scratch

Follow these manual steps to rebuild the workflow in n8n without importing the JSON:

1. **Prerequisite Setup:**
   - Create valid credentials in n8n for **Gmail (OAuth2)**, **Google Sheets (OAuth2)**, and **OpenAI**.
   - Create a Gmail label named `promises-processed` and copy its unique label ID.

2. **Step 1: Create the Setup Lane (Lane D):**
   - Add a **Google Sheets** node named `Create promise ledger`:
     - Resource: `spreadsheet`
     - Operation: `create`
     - Sheet Values: Add two sheets titled `Commitments` and `Chase-Queue`.
   - Add a **Code** node named `Seed headers and examples` connected to `Create promise ledger`. Paste JS code to initialize example rows with `dedupe_key` and schema fields (`Commitments` and `Chase-Queue`).
   - Add a **Switch** node named `Route by tab` connected to `Seed headers and examples`. Set rules to split items where `tab === 'Commitments'` and `tab === 'Chase-Queue'`.
   - Add **Code** nodes named `Columns for Commitments` and `Columns for Chase-Queue` to strip the `tab` property.
   - Add **Google Sheets** nodes named `Write Commitments tab` and `Write Chase-Queue tab`:
     - Operation: `append`
     - Document ID: `={{ $('Create promise ledger').first().json.spreadsheetId }}`
   - Add a **Code** node named `Print sheet id + next step` connected to both sheet write nodes to output the spreadsheet ID. Run this lane once, copy the resulting `spreadsheet_id`, and delete Lane D afterward.

3. **Step 2: Create Configuration and Triggers:**
   - Add a **Schedule Trigger** node named `Hourly tick (sweep at 08:00)` configured with an hourly interval.
   - Add a **Code** node named `Workflow Config` connected to the schedule trigger:
     - Return a JSON array containing `sheet_id` (paste the ID from step 1), `owner_email` (your email), and `label_id` (your Gmail label ID).
   - Add a **Switch** node named `Lane entry`:
     - Condition: `{{ $now.hour }}` equals `8` (`sweep`), otherwise (`intake`).

4. **Step 3: Build the Inbox Intake Lane (Lane A):**
   - Connect the `intake` output of `Lane entry` to a **Gmail** node named `Fetch new inbox emails`:
     - Operation: `getAll`, limit `25`, simple `false`, query filter `in:inbox -label:promises-processed newer_than:1d`.
   - Connect to a **Code** node named `Build extraction prompt` to clean HTML bodies and format LLM messages.
   - Connect to an **OpenAI** node named `Extract promises (AI)`:
     - Model: `gpt-4o-mini`, resource: `chat`. Attach OpenAI credentials.
   - Connect to a **Code** node named `Parse commitment verdict` to process JSON outputs and assign deduplication hashes.
   - Connect to a **Switch** node named `Route intake` (`commitment`, `fulfillment`, `skipped`, `alerted`).

5. **Step 4: Build Ledger Write & Fulfillment Logic:**
   - Merge `commitment` and `fulfillment` outputs using a **Merge** node named `Merge write streams`.
   - Connect to an **If** node named `Live write?` (`{{ $json.live === true }}`).
   - For live writes, route to a **Switch** node named `Write kind?` (`fulfillment` vs default).
   - For fulfillments:
     - Read open rows using a **Google Sheets** node named `Find open row`.
     - Select the oldest matching row using a **Code** node named `Pick open row`.
     - Evaluate matches using an **If** node named `Row matched?`.
     - Update the row to `done` using a **Google Sheets** node named `Mark done in ledger` (`operation: update`, match on `dedupe_key`).
   - For commitments:
     - Upsert records using a **Google Sheets** node named `Write ledger row` (`operation: appendOrUpdate`, match on `dedupe_key`).
   - Connect write nodes and skip/alert handlers to a **Gmail** node named `Label email processed` (`operation: addLabels`, label IDs: `={{ [$('Workflow Config').first().json.label_id] }}`).
   - Configure alert flows to email the owner via an additional **Gmail** node named `Email owner: intake alert`.

6. **Step 5: Build the Daily Chase Sweep Lane (Lane B):**
   - Connect the `sweep` output of `Lane entry` to a **Google Sheets** node named `Read ledger rows` (`operation: getAll`).
   - Connect to a **Code** node named `Classify aging` to calculate UTC calendar-day deltas.
   - Connect to a **Switch** node named `Route sweep` (`due-today`, `overdue`, `escalate`).
   - For escalations (>=3 days overdue):
     - Send a direct notification using a **Gmail** node named `Escalate to owner`.
     - Update the ledger status to `escalated` using a **Google Sheets** node named `Mark escalated in ledger`.
   - For `due-today` and `overdue` items:
     - Merge streams using a **Merge** node named `Merge chase streams`.
     - Build prompt texts using a **Code** node named `Build chase prompt`.
     - Generate drafts using an **OpenAI** node named `Draft chase email (AI)`.
     - Parse drafts using a **Code** node named `Parse chase draft`.
     - Queue drafts in Google Sheets using a **Google Sheets** node named `Queue chase for approval` (`operation: appendOrUpdate`, tab: `Chase-Queue`).
     - Notify the owner using a **Gmail** node named `Notify owner of queue`.
     - Update ledger states (`reminded` or `chased`) using a **Google Sheets** node named `Mark chased in ledger`.
   - Aggregate sweep metrics into a digest using `Sweep digest`, check worthiness via an **If** node, and deliver morning summaries via `Email owner: daily digest`.

7. **Step 6: Build the Demo Lane (Lane C):**
   - Add a **Manual Trigger** node named `When clicking 'Test workflow'`.
   - Connect to a **Code** node named `Demo inbox emails` supplying mock test payloads.
   - Connect to a **Code** node named `Demo extract (mirrors AI contract)` to simulate verdicts offline without API consumption.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Manual Approval Gate** | Chase drafts are stored in the `Chase-Queue` spreadsheet tab with a `pending-approval` status. The workflow never transmits emails automatically on your behalf. |
| **Deduplication Strategy** | Commitments are uniquely identified and updated using a generated `dedupe_key` hash derived from the sender's address and the promise text. |
| **UTC Calendar Aging** | Aging calculations are computed strictly in UTC across calendar-day boundaries to prevent timezone offset discrepancies. |