Pause risky Instantly sending accounts with Slack alerts and Google Sheets logs

https://n8nworkflows.xyz/workflows/pause-risky-instantly-sending-accounts-with-slack-alerts-and-google-sheets-logs-20555


# Pause risky Instantly sending accounts with Slack alerts and Google Sheets logs

### 1. Workflow Overview

This workflow automates daily risk assessment for email sending accounts managed via the Instantly API. It evaluates deliverability health metrics every morning, auto-pauses accounts exceeding risk thresholds, notifies administrators through Slack alerts and summaries, and logs historical health data to Google Sheets. 

The execution logic is divided into five functional blocks:
- **1.1 Schedule & Configuration Initialization:** Initiates the workflow on a daily schedule and defines operational parameters and thresholds.
- **1.2 Account Ingestion & Batching:** Retrieves sending accounts from Instantly using pagination and organizes them into processable batches.
- **1.3 Health & Analytics Data Collection:** Queries Instantly endpoints sequentially to gather warmup statistics, DNS vitals, and bounce metrics for the account batches.
- **1.4 Scoring & Decision Engine:** Aggregates metrics, assigns a risk verdict to each account, and branches execution flows based on predefined policies.
- **1.5 Action, Notification & Logging:** Automatically pauses flagged accounts, dispatches Slack alerts and digests, and appends structured records to Google Sheets.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule & Configuration Initialization
- **Overview:** Triggers the workflow execution daily at 7:00 AM and sets all critical operational thresholds, rules, and destination URLs.
- **Nodes Involved:** `Every Morning at 7am`, `Set Bounce Alert Thresholds`
- **Node Details:**
  - `Every Morning at 7am`
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Configured to trigger daily at hour 7.
    - *Input/Output:* No inputs; outputs execution flow to the configuration node.
    - *Edge Cases:* Ensure n8n timezone settings match the desired local execution time.
  - `Set Bounce Alert Thresholds`
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation / Parameter Definition)
    - *Configuration:* Defines global numeric variables (`maxBounceRatePct: 3`, `minSentForBounceCheck: 30`, `lookbackDays: 7`, `warnBelowWarmupScore: 90`, `pauseBelowWarmupScore: 75`, `batchSize: 50`), boolean toggles (`pauseOnDnsFailure: false`, `autoPause: true`, `notifyWhenAllHealthy: true`), and string settings (`slackChannel: "#deliverability"`, `trackerSheetUrl`).
    - *Input/Output:* Input from `Every Morning at 7am`; outputs configuration data to `Fetch Sending Accounts`.
    - *Edge Cases:* Invalid configuration types may cause downstream evaluation errors in script nodes.

#### Block 1.2: Account Ingestion & Batching
- **Overview:** Fetches all sending accounts from the Instantly API v2, managing pagination automatically, and packages them into clean data arrays for bulk processing.
- **Nodes Involved:** `Fetch Sending Accounts`, `Package Account Batches`
- **Node Details:**
  - `Fetch Sending Accounts`
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration:* Calls `https://api.instantly.ai/api/v2/accounts` using `httpBearerAuth`. Implements custom pagination querying `next_starting_after` up to 50 requests with a page limit of 100.
    - *Input/Output:* Input from `Set Bounce Alert Thresholds`; outputs raw API pages to `Package Account Batches`.
    - *Edge Cases:* Authentication failures if the Instantly API key lacks read privileges or has expired. Rate limiting if pagination requests exceed provider limits.
  - `Package Account Batches`
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation)
    - *Configuration:* Deduplicates accounts by email address, extracts core properties (`email`, `status`, `warmupStatus`, `dailyLimit`, `warmupScore`), and chunks them into arrays based on the configured `batchSize`.
    - *Input/Output:* Input from `Fetch Sending Accounts`; outputs account chunks to `Post Warmup Health Check`.
    - *Edge Cases:* Empty account lists will terminate processing if no items are returned.

#### Block 1.3: Health & Analytics Data Collection
- **Overview:** Executes sequential HTTP POST and GET requests to Instantly endpoints to gather warmup analytics, DNS vitals (MX, SPF, DKIM, DMARC), and daily bounce statistics over the defined lookback window.
- **Nodes Involved:** `Post Warmup Health Check`, `Post DNS Records Check`, `Fetch Bounce Stats`
- **Node Details:**
  - `Post Warmup Health Check`
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration:* POST request to `https://api.instantly.ai/api/v2/accounts/warmup-analytics` passing an array of batch emails. `onError` is set to `continueRegularOutput` with `alwaysOutputData` enabled.
    - *Input/Output:* Input from `Package Account Batches`; outputs aggregate warmup data to `Post DNS Records Check`.
    - *Edge Cases:* API timeouts or network blips; handled via `continueRegularOutput`.
  - `Post DNS Records Check`
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration:* POST request to `https://api.instantly.ai/api/v2/accounts/test/vitals` transmitting account emails. `onError` set to `continueRegularOutput` with `alwaysOutputData` enabled.
    - *Input/Output:* Input from `Post Warmup Health Check`; outputs domain validation success/failure lists to `Fetch Bounce Stats`.
    - *Edge Cases:* Failure responses from individual domains are parsed downstream in the scoring node.
  - `Fetch Bounce Stats`
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration:* GET request to `https://api.instantly.ai/api/v2/accounts/analytics/daily` with query parameters calculating `start_date` and `end_date` dynamically using Luxon expressions (`$now.minus({ days: ... })`). Executed once.
    - *Input/Output:* Input from `Post DNS Records Check`; outputs bounce logs to `Calculate Account Scores`.
    - *Edge Cases:* Date range format mismatches or empty analytics records during low-volume periods.

#### Block 1.4: Scoring & Decision Engine
- **Overview:** Combines metrics from all collected signals, compares them against user-defined thresholds, evaluates risk levels, and generates definitive verdicts (`healthy`, `warn`, `pause`, `paused`).
- **Nodes Involved:** `Calculate Account Scores`
- **Node Details:**
  - `Calculate Account Scores`
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation)
    - *Configuration:* Aggregates warmup scores, DNS validation results, and bounce rates. Evaluates bounce percentage against volume minimums, checks warmup degradation, and flags status anomalies. Assigns a `willPause` boolean property based on auto-pause settings and account active status.
    - *Input/Output:* Input from `Fetch Bounce Stats`; outputs structured account evaluation objects to `Filter Accounts to Pause`, `Compile Daily Slack Digest`, and `Format Health Metrics`.
    - *Edge Cases:* Missing metric payloads are safely defaulted to null or zero to prevent arithmetic exceptions.

#### Block 1.5: Action, Notification & Logging
- **Overview:** Filters risky accounts for automated pausing via the Instantly API, triggers immediate incident alerts on Slack, compiles and dispatches a workspace-level daily summary, and records history rows to Google Sheets.
- **Nodes Involved:** `Filter Accounts to Pause`, `Post Pause Account Request`, `Notify Slack on Paused Account`, `Compile Daily Slack Digest`, `Post Daily Digest to Slack`, `Format Health Metrics`, `Append Logs to Health Sheet`
- **Node Details:**
  - `Filter Accounts to Pause`
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Data Filtering)
    - *Configuration:* Evaluates condition where `{{ $json.willPause }}` equals `true`.
    - *Input/Output:* Input from `Calculate Account Scores`; outputs filtered accounts to `Post Pause Account Request`.
-  - `Post Pause Account Request`
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration:* POST request targeting `https://api.instantly.ai/api/v2/accounts/{{ encodeURIComponent($json.email) }}/pause` using Bearer authentication. `onError` set to `continueRegularOutput`.
    - *Input/Output:* Input from `Filter Accounts to Pause`; outputs API response data to `Notify Slack on Paused Account`.
    - *Edge Cases:* API failures when attempting to pause an already suspended account.
  - `Notify Slack on Paused Account`
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Notification Integration)
    - *Configuration:* Sends formatted markdown alert to the channel defined in `slackChannel` indicating an auto-paused account and its triggers.
    - *Input/Output:* Input from `Post Pause Account Request`; final destination branch.
    - *Edge Cases:* Slack API rate limits or invalid channel identifiers.
  - `Compile Daily Slack Digest`
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation)
    - *Configuration:* Summarizes account statuses into categorized lists (Paused, Recommended Pause, Watch, Healthy) and crafts a unified Markdown message body. Respects the `notifyWhenAllHealthy` toggle.
    - *Input/Output:* Input from `Calculate Account Scores`; outputs text payload to `Post Daily Digest to Slack`.
  - `Post Daily Digest to Slack`
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Notification Integration)
    - *Configuration:* Dispatches the compiled summary text to the designated Slack channel.
    - *Input/Output:* Input from `Compile Daily Slack Digest`; final destination branch.
  - `Format Health Metrics`
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Maps raw account evaluation JSON into standardized column attributes (`Date`, `Account`, `Domain`, `Status`, `Sent`, `Bounced`, `Bounce rate %`, `Warmup health`, `Warmup spam %`, `DNS`, `Verdict`, `Action`, `Issues`).
    - *Input/Output:* Input from `Calculate Account Scores`; outputs formatted rows to `Append Logs to Health Sheet`.
  - `Append Logs to Health Sheet`
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet Integration)
    - *Configuration:* Appends rows using auto-mapped input data into the spreadsheet specified by `trackerSheetUrl` targeting the sheet tab named `Health log`.
    - *Input/Output:* Input from `Format Health Metrics`; final destination branch.
    - *Edge Cases:* Missing or incorrectly named sheet tabs (`Health log`) will cause append errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Overview | None | None | Auto-pause risky Instantly sending accounts and report deliverability to Slack... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Schedule risk thresholds... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Fetch account batches... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Collect health signals... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Score account risk... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Pause risky accounts... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Send daily digest... |
| Sticky Note7 | n8n-nodes-base.stickyNote | Block Documentation | None | None | Log health history... |
| Every Morning at 7am | n8n-nodes-base.scheduleTrigger | Trigger workflow daily | None | Set Bounce Alert Thresholds | Schedule risk thresholds |
| Set Bounce Alert Thresholds | n8n-nodes-base.set | Define risk thresholds | Every Morning at 7am | Fetch Sending Accounts | Schedule risk thresholds |
| Fetch Sending Accounts | n8n-nodes-base.httpRequest | Retrieve Instantly accounts | Set Bounce Alert Thresholds | Package Account Batches | Fetch account batches |
| Package Account Batches | n8n-nodes-base.code | Deduplicate and chunk accounts | Fetch Sending Accounts | Post Warmup Health Check | Fetch account batches |
| Post Warmup Health Check | n8n-nodes-base.httpRequest | Query warmup health metrics | Package Account Batches | Post DNS Records Check | Collect health signals |
| Post DNS Records Check | n8n-nodes-base.httpRequest | Validate domain DNS vitals | Post Warmup Health Check | Fetch Bounce Stats | Collect health signals |
| Fetch Bounce Stats | n8n-nodes-base.httpRequest | Query daily bounce statistics | Post DNS Records Check | Calculate Account Scores | Collect health signals |
| Calculate Account Scores | n8n-nodes-base.code | Score risk and assign verdict | Fetch Bounce Stats | Filter Accounts to Pause, Compile Daily Slack Digest, Format Health Metrics | Score account risk |
| Filter Accounts to Pause | n8n-nodes-base.filter | Filter accounts meeting pause criteria | Calculate Account Scores | Post Pause Account Request | Pause risky accounts |
| Post Pause Account Request | n8n-nodes-base.httpRequest | Pause account in Instantly | Filter Accounts to Pause | Notify Slack on Paused Account | Pause risky accounts |
| Notify Slack on Paused Account | n8n-nodes-base.slack | Send individual pause alert | Post Pause Account Request | None | Pause risky accounts |
| Compile Daily Slack Digest | n8n-nodes-base.code | Compile workspace health summary | Calculate Account Scores | Post Daily Digest to Slack | Send daily digest |
| Post Daily Digest to Slack | n8n-nodes-base.slack | Send daily summary report | Compile Daily Slack Digest | None | Send daily digest |
| Format Health Metrics | n8n-nodes-base.set | Format row attributes for logging | Calculate Account Scores | Append Logs to Health Sheet | Log health history |
| Append Logs to Health Sheet | n8n-nodes-base.googleSheets | Log history to Google Sheets | Format Health Metrics | None | Log health history |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**
   - Add a `Schedule Trigger` node. Name it `Every Morning at 7am`.
   - Configure rule interval to trigger at hour `7`.

2. **Configure Thresholds:**
   - Add a `Set` node named `Set Bounce Alert Thresholds`.
   - Add numeric assignments: `maxBounceRatePct` (3), `minSentForBounceCheck` (30), `lookbackDays` (7), `warnBelowWarmupScore` (90), `pauseBelowWarmupScore` (75), `batchSize` (50).
   - Add boolean assignments: `pauseOnDnsFailure` (false), `autoPause` (true), `notifyWhenAllHealthy` (true).
   - Add string assignments: `slackChannel` (`#deliverability`), `trackerSheetUrl` (your Google Sheet URL).

3. **Fetch Accounts:**
   - Add an `HTTP Request` node named `Fetch Sending Accounts`.
   - Set method to `GET`, URL to `https://api.instantly.ai/api/v2/accounts`.
   - Configure authentication as Generic Credential Type -> HTTP Bearer Auth.
   - Add query parameter `limit` with value `100`.
   - Enable pagination mode: Update a parameter in each request (`starting_after` mapped to `{{ $response.body.next_starting_after }}`), complete expression `={{ !$response.body.next_starting_after }}`.

4. **Package Account Batches:**
   - Add a `Code` node named `Package Account Batches`.
   - Insert JavaScript to flatten account pages, filter out duplicates by email, and chunk them into arrays according to `batchSize`.

5. **Collect Health Metrics (Warmup, DNS, Bounce):**
   - Add an `HTTP Request` node named `Post Warmup Health Check`: POST to `https://api.instantly.ai/api/v2/accounts/warmup-analytics`, JSON body `={{ JSON.stringify({ emails: $json.emails }) }}`. Set `onError` to `Continue (Using Error Output)` or continue regular output with always output data. Use HTTP Bearer Auth.
   - Add an `HTTP Request` node named `Post DNS Records Check`: POST to `https://api.instantly.ai/api/v2/accounts/test/vitals`, JSON body `={{ JSON.stringify({ accounts: $('Package Account Batches').item.json.emails }) }}`. Configure Bearer Auth and error handling.
   - Add an `HTTP Request` node named `Fetch Bounce Stats`: GET from `https://api.instantly.ai/api/v2/accounts/analytics/daily`. Query parameters: `start_date` = `={{ $now.minus({ days: $('Set Bounce Alert Thresholds').first().json.lookbackDays }).toFormat('yyyy-MM-dd') }}`, `end_date` = `={{ $now.toFormat('yyyy-MM-dd') }}`. Configure Bearer Auth.

6. **Calculate Scores & Verdicts:**
   - Add a `Code` node named `Calculate Account Scores`.
   - Implement evaluation logic matching bounce rate thresholds, warmup scores, domain DNS validation results, and status flags to produce verdicts and determine `willPause`.

7. **Branch 1: Auto-Pause & Instant Alerts:**
   - Add a `Filter` node named `Filter Accounts to Pause`: condition where `{{ $json.willPause }}` is true.
   - Add an `HTTP Request` node named `Post Pause Account Request`: POST to `=https://api.instantly.ai/api/v2/accounts/{{ encodeURIComponent($json.email) }}/pause` using Bearer Auth.
   - Add a `Slack` node named `Notify Slack on Paused Account`: Resource message, action post, select channel name from `{{ $('Set Bounce Alert Thresholds').first().json.slackChannel }}`.

8. **Branch 2: Daily Digest:**
   - Add a `Code` node named `Compile Daily Slack Digest` to summarize all accounts.
   - Add a `Slack` node named `Post Daily Digest to Slack` to send the summary text.

9. **Branch 3: History Logging:**
   - Add a `Set` node named `Format Health Metrics` to map fields (`Date`, `Account`, `Domain`, `Status`, `Sent`, `Bouncer rate %`, etc.).
   - Add a `Google Sheets` node named `Append Logs to Health Sheet`: Operation `Append`, select Document URL from expression `={{ $('Set Bounce Alert Thresholds').first().json.trackerSheetUrl }}`, Sheet Name `Health log`, auto-map input data.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Instantly API v2 Documentation | Required for endpoint references, scopes, and authentication setup. |
| Slack Integration Setup | Requires valid Slack bot credentials with permissions to post messages in target channels. |
| Google Sheets Integration Setup | Requires a properly shared spreadsheet containing a tab named exactly `Health log`. |