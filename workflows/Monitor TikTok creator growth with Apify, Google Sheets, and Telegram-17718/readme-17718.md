Monitor TikTok creator growth with Apify, Google Sheets, and Telegram

https://n8nworkflows.xyz/workflows/monitor-tiktok-creator-growth-with-apify--google-sheets--and-telegram-17718


# Monitor TikTok creator growth with Apify, Google Sheets, and Telegram

### 1. Workflow Overview

This workflow automates the monitoring of TikTok creator growth, profile metrics, and qualification states. It is designed to track a watchlist of creators or competitors, detect significant growth, declines, or threshold breaches, log historical snapshots, and send formatted alerts via Telegram. 

The execution logic is divided into the following sequential blocks:
- **1.1 Input Reception & Watchlist Validation:** Initiates via manual action or a weekly cron schedule, reads creator handles from Google Sheets, and validates/normalizes the configuration parameters.
- **1.2 Data Scraping:** Triggers the external Apify actor (`FetchCat TikTok Profile Scraper`) to extract real-time public metrics and downloads the resulting dataset.
- **1.3 Metrics Comparison & Evaluation:** Normalizes scraped data, retrieves previous baseline data from Google Sheets, and evaluates delta changes against configured alert thresholds and qualification rules.
- **1.4 Conditional Alert Dispatch:** Evaluates whether changes meet alerting criteria and dispatches structured HTML messages to a designated Telegram chat.
- **1.5 Data Persistence & Finalization:** Appends point-in-time snapshot records to a history ledger, upserts the current state into the profile registry, and reports execution completion metrics.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Watchlist Validation
This block handles workflow triggering, reads the raw creator watchlist from a connected spreadsheet, and sanitizes input data to construct a valid batch payload for the scraper.

- **Manual Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` (Trigger Node). Allows on-demand manual execution of the workflow.
  - **Configuration:** Default settings.
  - **Connections:** Output connects to `1. Read Creator Watchlist`.
  - **Edge Cases:** None.

- **Weekly Monday Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node). Automatically executes the workflow every Monday at 09:00.
  - **Configuration:** Interval set to weeks, triggered on day 1 (Monday) at hour 9.
  - **Connections:** Output connects to `1. Read Creator Watchlist`.
  - **Edge Cases:** Execution timezone depends on the n8n instance configuration.

- **1. Read Creator Watchlist**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Action Node). Fetches active creator records from the "Creators" worksheet.
  - **Configuration:** Uses Google Sheets OAuth2 credentials. Operation reads data from sheet ID `933636260` ("Creators") within document ID `1YV806viFdmh0J6m2OuzZXhgDvrrWL8-GqU-DKh5ygKE`. Parameter `alwaysOutputData` is enabled.
  - **Connections:** Inputs from triggers; output connects to `Validate Watchlist and Build Actor Input`.
  - **Edge Cases:** Authentication token expiration; missing worksheet columns; network timeouts connecting to Google APIs.

- **Validate Watchlist and Build Actor Input**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation Node). Parses raw rows, filters active profiles, validates TikTok handles against regex patterns, handles bounds for follower limits/alert thresholds, and compiles an Apify actor input payload.
  - **Configuration:** Custom JavaScript filtering active rows, extracting handles, and establishing default values (e.g., minimum followers, growth alert percentage). Throws errors if zero active creators are found or if the count exceeds 500.
  - **Connections:** Input from `1. Read Creator Watchlist`; output connects to `2. Run FetchCat TikTok Profile Scraper`.
  - **Edge Cases:** Malformed TikTok handles/URLs throw explicit runtime errors.

---

#### 2.2 Data Scraping
This block interfaces with the Apify API to execute the scraper actor and download the raw output dataset.

- **2. Run FetchCat TikTok Profile Scraper**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Action Node). Submits a POST request to the Apify API to run the FetchCat TikTok Profile Scraper actor (`B5qu4d1iteAp2TaF1`) with synchronous waiting.
  - **Configuration:** Uses HTTP Header Authentication (`Bearer` token). Target URL: `https://api.apify.com/v2/acts/B5qu4d1iteAp2TaF1/runs`. Query parameter `waitForFinish` set to `300`. Timeout configured to 310,000ms. Body accepts JSON payload from `actorInput`.
  - **Connections:** Input from `Validate Watchlist and Build Actor Input`; output connects to `Download Current TikTok Profiles`.
  - **Edge Cases:** Apify run timeouts (exceeding 300 seconds); insufficient API credits; invalid authentication tokens.

- **Download Current TikTok Profiles**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Action Node). Downloads the resulting JSON dataset items from the completed Apify run.
  - **Configuration:** Uses HTTP Header Authentication. Target URL: `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items`. Query parameters set `clean=true` and limit set to the total number of processed configuration entries.
  - **Connections:** Input from `2. Run FetchCat TikTok Profile Scraper`; output connects to `Normalize Current Creator Metrics`.
  - **Edge Cases:** Dataset generation failures; empty datasets returned by Apify.

---

#### 2.3 Metrics Comparison & Evaluation
This block normalizes raw scraper output, fetches previous baseline records, and calculates deltas for growth and qualification.

- **Normalize Current Creator Metrics**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation Node). Cleans scraped profile data, formats hyperlinks, generates serial epoch dates for Google Sheets compatibility, and structures object properties.
  - **Configuration:** Custom JavaScript filtering out errored entries and mapping profile fields (followers, likes, bio, verification flags, etc.).
  - **Connections:** Input from `Download Current TikTok Profiles`; output connects to `Read Previous Creator Metrics`.
  - **Edge Cases:** Missing optional profile properties returning null values.

- **Read Previous Creator Metrics**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Action Node). Reads existing profile records from the "Current Profiles" sheet to serve as a baseline for delta calculations.
  - **Configuration:** Uses Google Sheets OAuth2 credentials. Reads from sheet ID `1783850914` ("Current Profiles"). Configured with `executeOnce: true` and `alwaysOutputData: true`.
  - **Connections:** Input from `Normalize Current Creator Metrics`; output connects to `Compare Growth and Qualification`.
  - **Edge Cases:** Empty previous metrics table (handled as a first-run baseline condition).

- **Compare Growth and Qualification**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation Node). Merges current metrics with previous rows, calculates follower and video deltas, determines qualification tiers, checks alert triggers, and constructs the Telegram notification payload.
  - **Configuration:** Custom JavaScript performing hash-map lookups by handle, computing percentage growths, evaluating thresholds, and compiling HTML-formatted messages.
  - **Connections:** Input from `Read Previous Creator Metrics`; output connects to `Has Baseline or Meaningful Changes`.
  - **Edge Cases:** Division by zero during growth percentage calculation on accounts with zero previous followers.

---

#### 2.4 Conditional Alert Dispatch
This block evaluates whether changes are significant enough to warrant messaging and sends structured notifications to Telegram.

- **Has Baseline or Meaningful Changes**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Determines if an alert should be dispatched based on the computed `shouldAlert` boolean property.
  - **Configuration:** Condition checks if `{{ $json.shouldAlert }}` equals `true`.
  - **Connections:** Input from `Compare Growth and Qualification`. True branch connects to `3. Send Creator Changes to Telegram`; false branch bypasses the telegram step and connects directly to `Prepare Snapshot Rows`.
  - **Edge Cases:** Evaluation errors if input payload is missing the boolean property.

- **3. Send Creator Changes to Telegram**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Sends the formatted HTML notification message to a specified Telegram chat ID.
  - **Configuration:** Uses Telegram API credentials. Target text mapped from `{{ $json.telegramMessage }}` with `parse_mode` set to `HTML`.
  - **Connections:** Input from the true branch of `Has Baseline or Meaningful Changes`; output connects to `Prepare Snapshot Rows`.
  - **Edge Cases:** Invalid chat IDs; Telegram API rate limits; parsing errors due to unescaped HTML characters in creator names or labels.

---

#### 2.5 Data Persistence & Finalization
This block records historical metrics, updates the current profiles registry, and exits the workflow.

- **Prepare Snapshot Rows**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation Node). Acts as a routing junction to unify execution paths from both conditional branches and prepare snapshot records.
  - **Configuration:** Custom JavaScript passing through the primary comparison output object with `executeOnce: true`.
  - **Connections:** Inputs from `Has Baseline or Meaningful Changes` (false path) and `3. Send Creator Changes to Telegram`; output connects to `Expand Snapshot History`.
  - **Edge Cases:** None.

- **Expand Snapshot History**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation Node). Expands the aggregated rows array into individual items for row-by-row insertion into Google Sheets.
  - **Configuration:** Custom JavaScript mapping `rows` to individual item nodes.
  - **Connections:** Input from `Prepare Snapshot Rows`; output connects to `4. Append Snapshot History`.
  - **Edge Cases:** Large array expansions consuming memory if watchlists contain thousands of items.

- **4. Append Snapshot History**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Action Node). Appends a new timestamped row for each creator into the historical tracking ledger.
  - **Configuration:** Uses Google Sheets OAuth2 credentials. Operation set to `append` on sheet ID `1786791067` ("Snapshot History"). Maps explicit schema columns (Name, Label, Videos, Profile, Followers, Snapshot at, Total likes, Snapshot key, Change status, Qualification, TikTok handle, Follower change, Follower growth %, Total likes change, Video count change).
  - **Connections:** Input from `Expand Snapshot History`; output connects to `Continue After History`.
  - **Edge Cases:** Google Sheets API write quota limits during large batch appends.

- **Continue After History**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation Node). Resets the workflow items context after batch history appends to prepare for upserting current profiles.
  - **Configuration:** Custom JavaScript returning the primary comparison object with `executeOnce: true`.
  - **Connections:** Input from `4. Append Snapshot History`; output connects to `Expand Current Profiles`.
  - **Edge Cases:** None.

- **Expand Current Profiles**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation Node). Unpacks the aggregated profile array into individual items for upserting.
  - **Configuration:** Custom JavaScript mapping rows to individual item nodes.
  - **Connections:** Input from `Continue After History`; output connects to `5. Update Current Profiles`.
  - **Edge Cases:** None.

- **5. Update Current Profiles**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Action Node). Upserts the latest profile metrics into the primary registry sheet based on matching TikTok handles.
  - **Configuration:** Uses Google Sheets OAuth2 credentials. Operation set to `appendOrUpdate` on sheet ID `1783850914` ("Current Profiles"). Matching column set to `TikTok handle`. Option `useAppend` enabled.
  - **Connections:** Input from `Expand Current Profiles`; output connects to `Creator Monitor Complete`.
  - **Edge Cases:** Handle mismatch causing duplicate rows instead of updates if handles change formatting between runs.

- **Creator Monitor Complete**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation Node). Formats a summary object confirming successful execution metrics.
  - **Configuration:** Custom JavaScript returning completion status, updated profile counts, and alert counts with `executeOnce: true`.
  - **Connections:** Input from `5. Update Current Profiles`; output terminates the workflow branch.
  - **Edge Cases:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manual Trigger | n8n-nodes-base.manualTrigger | Trigger Node | None | 1. Read Creator Watchlist | Monitor TikTok Creator Growth <br><br> Track a reusable TikTok creator or competitor watchlist with `fetch_cat/tiktok-profile-scraper`, Google Sheets, and Telegram. The workflow works on n8n Cloud or self-hosted n8n and uses deterministic thresholds rather than an AI model.<br><br>### How it works<br>1. Starts manually or every Monday at 09:00.<br>2. Reads active creators and alert thresholds from the Creators sheet.<br>3. Scrapes every profile in one efficient FetchCat Actor run.<br>4. Compares current followers, likes, videos, and qualification with the previous snapshot.<br>5. Sends a Telegram baseline on the first run and later alerts only for meaningful growth, decline, or qualification changes.<br>6. Appends dated history and updates the latest profile table after successful alert delivery.<br><br>### Setup<br>- [ ] Create the Creators, Current Profiles, and Snapshot History tabs with the documented headers.<br>- [ ] Add handles, labels, qualification limits, and alert thresholds to Creators.<br>- [ ] Select the same spreadsheet and appropriate tab in all Google Sheets nodes.<br>- [ ] Connect one Apify HTTP Header Auth credential to both FetchCat request nodes.<br>- [ ] Connect Telegram, run manually to create the baseline, then publish the schedule.<br><br>Recent-video monitoring is intentionally excluded because TikTok exposes that data inconsistently. |
| Weekly Monday Trigger | n8n-nodes-base.scheduleTrigger | Trigger Node | None | 1. Read Creator Watchlist | Monitor TikTok Creator Growth <br><br> Track a reusable TikTok creator or competitor watchlist with `fetch_cat/tiktok-profile-scraper`, Google Sheets, and Telegram. The workflow works on n8n Cloud or self-hosted n8n and uses deterministic thresholds rather than an AI model.<br><br>### How it works<br>1. Starts manually or every Monday at 09:00.<br>2. Reads active creators and alert thresholds from the Creators sheet.<br>3. Scrapes every profile in one efficient FetchCat Actor run.<br>4. Compares current followers, likes, videos, and qualification with the previous snapshot.<br>5. Sends a Telegram baseline on the first run and later alerts only for meaningful growth, decline, or qualification changes.<br>6. Appends dated history and updates the latest profile table after successful alert delivery.<br><br>### Setup<br>- [ ] Create the Creators, Current Profiles, and Snapshot History tabs with the documented headers.<br>- [ ] Add handles, labels, qualification limits, and alert thresholds to Creators.<br>- [ ] Select the same spreadsheet and appropriate tab in all Google Sheets nodes.<br>- [ ] Connect one Apify HTTP Header Auth credential to both FetchCat request nodes.<br>- [ ] Connect Telegram, run manually to create the baseline, then publish the schedule.<br><br>Recent-video monitoring is intentionally excluded because TikTok exposes that data inconsistently. |
| 1. Read Creator Watchlist | n8n-nodes-base.googleSheets | Action Node | Manual Trigger, Weekly Monday Trigger | Validate Watchlist and Build Actor Input | ## Load the creator watchlist<br><br>Starts manually or weekly and reads active handles, labels, qualification limits, and alert thresholds from Google Sheets. |
| Validate Watchlist and Build Actor Input | n8n-nodes-base.code | Data Transformation Node | 1. Read Creator Watchlist | 2. Run FetchCat TikTok Profile Scraper | ## Run one profile batch<br><br>Validates up to 500 unique creators, runs `fetch_cat/tiktok-profile-scraper` once, and downloads the current public profile metrics. |
| 2. Run FetchCat TikTok Profile Scraper | n8n-nodes-base.httpRequest | Action Node | Validate Watchlist and Build Actor Input | Download Current TikTok Profiles | ## Run one profile batch<br><br>Validates up to 500 unique creators, runs `fetch_cat/tiktok-profile-scraper` once, and downloads the current public profile metrics. |
| Download Current TikTok Profiles | n8n-nodes-base.httpRequest | Action Node | 2. Run FetchCat TikTok Profile Scraper | Normalize Current Creator Metrics | ## Run one profile batch<br><br>Validates up to 500 unique creators, runs `fetch_cat/tiktok-profile-scraper` once, and downloads the current public profile metrics. |
| Normalize Current Creator Metrics | n8n-nodes-base.code | Data Transformation Node | Download Current TikTok Profiles | Read Previous Creator Metrics | ## Compare growth and qualification<br><br>Normalizes current metrics, reads the previous profile table, and calculates follower, likes, video, and qualification changes. |
| Read Previous Creator Metrics | n8n-nodes-base.googleSheets | Action Node | Normalize Current Creator Metrics | Compare Growth and Qualification | ## Compare growth and qualification<br><br>Normalizes current metrics, reads the previous profile table, and calculates follower, likes, video, and qualification changes. |
| Compare Growth and Qualification | n8n-nodes-base.code | Data Transformation Node | Read Previous Creator Metrics | Has Baseline or Meaningful Changes | ## Compare growth and qualification<br><br>Normalizes current metrics, reads the previous profile table, and calculates follower, likes, video, and qualification changes. |
| Has Baseline or Meaningful Changes | n8n-nodes-base.if | Flow Control Node | Compare Growth and Qualification | 3. Send Creator Changes to Telegram, Prepare Snapshot Rows | ## Route meaningful alerts<br><br>Sends a readable first-run baseline or significant changes to Telegram. Quiet runs continue without a notification. |
| 3. Send Creator Changes to Telegram | n8n-nodes-base.telegram | Action Node | Has Baseline or Meaningful Changes | Prepare Snapshot Rows | ## Route meaningful alerts<br><br>Sends a readable first-run baseline or significant changes to Telegram. Quiet runs continue without a notification. |
| Prepare Snapshot Rows | n8n-nodes-base.code | Data Transformation Node | Has Baseline or Meaningful Changes, 3. Send Creator Changes to Telegram | Expand Snapshot History | ## Save dated history<br><br>Appends one dated row per creator so users can sort, chart, and audit changes over time. |
| Expand Snapshot History | n8n-nodes-base.code | Data Transformation Node | Prepare Snapshot Rows | 4. Append Snapshot History | ## Save dated history<br><br>Appends one dated row per creator so users can sort, chart, and audit changes over time. |
| 4. Append Snapshot History | n8n-nodes-base.googleSheets | Action Node | Expand Snapshot History | Continue After History | ## Save dated history<br><br>Appends one dated row per creator so users can sort, chart, and audit changes over time. |
| Continue After History | n8n-nodes-base.code | Data Transformation Node | 4. Append Snapshot History | Expand Current Profiles | ## Update the latest profile table<br><br>Upserts one current row per handle only after any required Telegram alert succeeds, then returns a concise execution summary. |
| Expand Current Profiles | n8n-nodes-base.code | Data Transformation Node | Continue After History | 5. Update Current Profiles | ## Update the latest profile table<br><br>Upserts one current row per handle only after any required Telegram alert succeeds, then returns a concise execution summary. |
| 5. Update Current Profiles | n8n-nodes-base.googleSheets | Action Node | Expand Current Profiles | Creator Monitor Complete | ## Update the latest profile table<br><br>Upserts one current row per handle only after any required Telegram alert succeeds, then returns a concise execution summary. |
| Creator Monitor Complete | n8n-nodes-base.code | Data Transformation Node | 5. Update Current Profiles | None | ## Update the latest profile table<br><br>Upserts one current row per handle only after any required Telegram alert succeeds, then returns a concise execution summary. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`).
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`) and configure it for weekly execution on Monday at 09:00.

2. **Add Watchlist Reader:**
   - Create a **Google Sheets** node named `1. Read Creator Watchlist`.
   - Configure credentials using Google Sheets OAuth2.
   - Set operation to read/get rows from the "Creators" sheet. Enable `Always Output Data`.

3. **Add Watchlist Validation Code:**
   - Create a **Code** node named `Validate Watchlist and Build Actor Input`.
   - Paste JavaScript logic to validate active handles, filter out inactive rows, enforce a 500-creator limit, and compile the Apify actor input payload.

4. **Add Apify Scraper HTTP Request Nodes:**
   - Create an **HTTP Request** node named `2. Run FetchCat TikTok Profile Scraper`.
   - Set method to `POST`, URL to `https://api.apify.com/v2/acts/B5qu4d1iteAp2TaF1/runs`, with query parameter `waitForFinish` = `300`.
   - Configure authentication as generic HTTP Header Auth (Header: `Authorization`, Value: `Bearer <Your_Apify_Token>`). Set request body to JSON from `{{ $json.actorInput }}`.
   - Create a second **HTTP Request** node named `Download Current TikTok Profiles`.
   - Set URL to `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items` with query parameters `clean=true` and limit set to the creator count expression. Use the same HTTP Header Auth credentials.

5. **Add Metrics Normalization and Previous Baseline Reading:**
   - Create a **Code** node named `Normalize Current Creator Metrics` to clean profile outputs, convert dates, and format hyperlinks.
   - Create a **Google Sheets** node named `Read Previous Creator Metrics`. Set operation to get rows from the "Current Profiles" sheet using Google Sheets OAuth2 credentials. Enable `Execute Once` and `Always Output Data`.

6. **Add Comparison and Routing Logic:**
   - Create a **Code** node named `Compare Growth and Qualification` to merge datasets, calculate follower/video deltas, assess qualification, and format the Telegram HTML message.
   - Create an **If** node named `Has Baseline or Meaningful Changes`. Set condition to evaluate if `{{ $json.shouldAlert }}` is `true`.

7. **Add Telegram Notification Node:**
   - Create a **Telegram** node named `3. Send Creator Changes to Telegram`.
   - Configure Telegram API credentials, map text to `{{ $json.telegramMessage }}`, and set `parse_mode` to `HTML`. Connect its input to the `true` output of the If node.

8. **Add Snapshot History Logging:**
   - Create a **Code** node named `Prepare Snapshot Rows` to unify execution paths from the If node branches.
   - Create a **Code** node named `Expand Snapshot History` to expand row arrays into individual items.
   - Create a **Google Sheets** node named `4. Append Snapshot History`. Set operation to `append` on the "Snapshot History" sheet. Map schema columns explicitly (Name, Label, Videos, Profile, Followers, Snapshot at, Total likes, Snapshot key, Change status, Qualification, TikTok handle, Follower change, Follower growth %, Total likes change, Video count change).

9. **Add Current Profiles Upsertion:**
   - Create a **Code** node named `Continue After History` to reset context.
   - Create a **Code** node named `Expand Current Profiles` to unpack profile rows.
   - Create a **Google Sheets** node named `5. Update Current Profiles`. Set operation to `appendOrUpdate` on the "Current Profiles" sheet, matching on column `TikTok handle`. Enable `Use Append` option.

10. **Add Completion Node:**
    - Create a **Code** node named `Creator Monitor Complete` to return final status and metrics.

11. **Establish Connections:**
    - Connect `Manual Trigger` and `Weekly Monday Trigger` -> `1. Read Creator Watchlist`.
    - Connect `1. Read Creator Watchlist` -> `Validate Watchlist and Build Actor Input` -> `2. Run FetchCat TikTok Profile Scraper` -> `Download Current TikTok Profiles` -> `Normalize Current Creator Metrics` -> `Read Previous Creator Metrics` -> `Compare Growth and Qualification` -> `Has Baseline or Meaningful Changes`.
    - Connect `Has Baseline or Meaningful Changes` (True) -> `3. Send Creator Changes to Telegram` -> `Prepare Snapshot Rows`.
    - Connect `Has Baseline or Meaningful Changes` (False) -> `Prepare Snapshot Rows`.
    - Connect `Prepare Snapshot Rows` -> `Expand Snapshot History` -> `4. Append Snapshot History` -> `Continue After History` -> `Expand Current Profiles` -> `5. Update Current Profiles` -> `Creator Monitor Complete`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| TikTok Creator Growth Tracker spreadsheet template | [Google Sheets Template](https://docs.google.com/spreadsheets/d/1bs2Nh_ewe0yrIVbDB0NX9Gc1GgOdOGuJe-7tNXSXSkA/edit?usp=sharing) |
| FetchCat TikTok Profile Scraper Actor | [Apify Actor Documentation](https://apify.com/fetch_cat/tiktok-profile-scraper) |
| Telegram Bot Creation | [BotFather](https://t.me/BotFather) |