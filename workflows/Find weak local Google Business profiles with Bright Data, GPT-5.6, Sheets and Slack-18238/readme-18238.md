Find weak local Google Business profiles with Bright Data, GPT-5.6, Sheets and Slack

https://n8nworkflows.xyz/workflows/find-weak-local-google-business-profiles-with-bright-data--gpt-5-6--sheets-and-slack-18238


# Find weak local Google Business profiles with Bright Data, GPT-5.6, Sheets and Slack

### 1. Workflow Overview

This workflow automates weekly local business prospecting by discovering Google Maps business listings via Bright Data, auditing their profiles against local competitors, extracting customer complaint themes using OpenAI (GPT-5.6), filtering out recently contacted prospects via Google Sheets, and broadcasting a ranked intelligence report to Slack.

The workflow logic is categorized into the following functional blocks:
- **1.1 Schedule & Configuration:** Initializes the operational parameters (city, target business categories, scoring weights, and destination endpoints) on a weekly schedule.
- **1.2 Asynchronous Maps Discovery:** Triggers Bright Data Google Maps searches asynchronously, polling snapshot progress until the dataset is ready for extraction.
- **1.3 Profile Auditing & Scoring:** Downloads business records, benchmarks each listing against nearby peers using Google’s "people also search" metadata, and calculates a quantified gap score.
- **1.4 Duplicate Filtering:** Cross-references discovered prospects with historical entries stored in Google Sheets to exclude businesses contacted within the last 90 days.
- **1.5 AI-Powered Complaint Extraction:** Analyzes reviews rated three stars and below using OpenAI (GPT-5.6) to isolate specific negative customer experience themes.
- **1.6 Aggregation & Delivery:** Compiles a synthesized markdown digest, logs individual prospect rows to Google Sheets, and publishes the ranked report to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Initializes the scan schedule and defines all target criteria, geographic locations, and integration targets in a single configuration block.
- **Nodes Involved:** `Weekly Prospect Scan`, `Set Search Config`, `Build Discovery Input`
- **Node Details:**
  - **Weekly Prospect Scan** (`n8n-nodes-base.scheduleTrigger`)
    - *Type & Role:* Trigger node executing the workflow automatically every week (configured at 07:00 AM).
    - *Configuration:* Cron/interval rule set to execute daily at hour 7, minute 0.
    - *Connections:* Input: None; Output: `Set Search Config`.
    - *Edge Cases:* Server downtime or timezone shifts can delay execution.
  - **Set Search Config** (`n8n-nodes-base.set`)
    - *Type & Role:* Parameter declaration node establishing global variables.
    - *Configuration:* Manual assignment mode containing fields: `city` (Austin, TX), `categories` (dentist, chiropractor), `country` (US), `businesses_per_category` (20), `min_reviews` (10), `min_gap_score` (15), `prospects_per_run` (8), `recontact_after_days` (90), `sheet_url`, and `slack_channel`.
    - *Key Expressions:* Holds static configuration values.
    - *Connections:* Input: `Weekly Prospect Scan`; Output: `Build Discovery Input`.
    - *Edge Cases:* Invalid URLs or misconfigured threshold values.
  - **Build Discovery Input** (`n8n-nodes-base.code`)
    - *Type & Role:* JavaScript processing node transforming configuration variables into an array of search payloads.
    - *Configuration:* Executes once for all items, parsing comma-separated categories and formatting keywords for Bright Data.
    - *Key Expressions:* `keyword: \`${category} in ${city}\``, `country: String(cfg.country || 'US').toUpperCase()`
    - *Connections:* Input: `Set Search Config`; Output: `Discover Businesses With Bright Data`.
    - *Edge Cases:* Throws an error if `city` or `categories` are missing or empty.

---

#### 2.2 Asynchronous Maps Discovery
- **Overview:** Submits discovery requests to Bright Data for each category, monitors processing completion through a polling loop, and downloads the raw JSON dataset.
- **Nodes Involved:** `Discover Businesses With Bright Data`, `Check Discovery Accepted`, `Wait For Snapshot`, `Check Snapshot Progress`, `Evaluate Scrape Progress`, `Profiles Ready?`, `Download Business Profiles`
- **Node Details:**
  - **Discover Businesses With Bright Data** (`n8n-nodes-base.httpRequest`)
    - *Type & Role:* HTTP POST request initiating the Bright Data discovery dataset trigger.
    - *Configuration:* Uses generic HTTP Header Auth (`Authorization: Bearer <key>`), timeout set to 120000ms, max 3 retries, with `neverError: true` enabled.
    - *Key Expressions:* URL queries dataset ID and limit per input: `={{ $('Set Search Config').first().json.businesses_per_category }}`. Body serializes `$json.payload`.
    - *Connections:* Input: `Build Discovery Input`; Output: `Check Discovery Accepted`.
    - *Edge Cases:* API authentication failures, rate limits, or invalid dataset IDs.
  - **Check Discovery Accepted** (`n8n-nodes-base.code`)
    - *Type & Role:* Validation node verifying successful snapshot initialization.
    - *Configuration:* JavaScript evaluation checking response validity.
    - *Key Expressions:* Checks for `res.error`, `res.errors`, and validates the presence of `res.snapshot_id`.
    - *Connections:* Input: `Discover Businesses With Bright Data`; Output: `Wait For Snapshot`.
    - *Edge Cases:* Throws an explicit error if Bright Data rejects the payload or omits the snapshot ID.
  - **Wait For Snapshot** (`n8n-nodes-base.wait`)
    - *Type & Role:* Flow-control pause node to allow dataset generation.
    - *Configuration:* Pauses execution for 20 seconds between polling cycles.
    - *Connections:* Input: `Check Discovery Accepted` or `Profiles Ready?` (false branch); Output: `Check Snapshot Progress`.
    - *Edge Cases:* None.
  - **Check Snapshot Progress** (`n8n-nodes-base.httpRequest`)
    - *Type & Role:* HTTP GET request checking snapshot generation status.
    - *Configuration:* Uses HTTP Header Auth, timeout 60000ms, max 3 retries, `neverError: true`.
    - *Key Expressions:* URL queries progress via snapshot ID: `=https://api.brightdata.com/datasets/v3/progress/{{ $('Check Discovery Accepted').first().json.snapshot_id }}`.
    - *Connections:* Input: `Wait For Snapshot`; Output: `Evaluate Scrape Progress`.
    - *Edge Cases:* Network timeouts or transient API unavailability.
  - **Evaluate Scrape Progress** (`n8n-nodes-base.code`)
    - *Type & Role:* Loop-control and iteration-count validation node.
    - *Configuration:* Tracks polling attempts (capped at MAX_POLLS = 20, totaling ~7 minutes).
    - *Key Expressions:* Uses `$runIndex + 1` to track attempt numbers.
    - *Connections:* Input: `Check Snapshot Progress`; Output: `Profiles Ready?`.
    - *Edge Cases:* Throws an error if scraping fails or exceeds the maximum poll limit.
  - **Profiles Ready?** (`n8n-nodes-base.if`)
    - *Type & Role:* Conditional routing node checking snapshot status.
    - *Configuration:* Evaluates whether `$json.status` equals `ready`.
    - *Key Expressions:* `={{ $json.status }}` equals `ready`.
    - *Connections:* Input: `Evaluate Scrape Progress`; True Output: `Download Business Profiles`; False Output: `Wait For Snapshot`.
    - *Edge Cases:* Infinite waiting loops if status remains pending without error (guarded by MAX_POLLS).
  - **Download Business Profiles** (`n8n-nodes-base.httpRequest`)
    - *Type & Role:* HTTP GET request retrieving the completed JSON dataset.
    - *Configuration:* Uses HTTP Header Auth, 180000ms timeout, 3 retries, `neverError: true`.
    - *Key Expressions:* URL fetches snapshot data: `=https://api.brightdata.com/datasets/v3/snapshot/{{ $('Check Discovery Accepted').first().json.snapshot_id }}?format=json`.
    - *Connections:* Input: `Profiles Ready?` (true branch); Output: `Score Profile Gaps`.
    - *Edge Cases:* Payload size limitations or download failures on large datasets.

---

#### 2.3 Profile Auditing & Scoring
- **Overview:** Parses raw business profiles, benchmarks metrics against competitive peer sets extracted from Google Maps metadata, and assigns weighted gap scores.
- **Nodes Involved:** `Score Profile Gaps`
- **Node Details:**
  - **Score Profile Gaps** (`n8n-nodes-base.code`)
    - *Type & Role:* Heavy data-processing and scoring logic node.
    - *Configuration:* JavaScript execution processing all input items in a single pass.
    - *Key Expressions:* Filters closed businesses, applies minimum review thresholds (`minReviews`), computes median peer ratings/review counts, calculates negative review percentages, and accumulates gap penalty points (unclaimed: 25, no website: 20, low rating vs peers: 15, negative reviews: 15, few reviews: 12, no opening hours: 10, few photos: 8, single category: 5, no services: 5).
    - *Connections:* Input: `Download Business Profiles`; Output: `Read Reported Businesses`.
    - *Edge Cases:* Throws an error if zero business profiles meet the minimum gap score threshold.

---

#### 2.4 Duplicate Filtering
- **Overview:** Reads historical entries from Google Sheets and filters out any business profiles contacted within the configured cooldown window.
- **Nodes Involved:** `Read Reported Businesses`, `Select New Prospects`, `Any New Prospects?`, `Nothing New This Week`
- **Node Details:**
  - **Read Reported Businesses** (`n8n-nodes-base.googleSheets`)
    - *Type & Role:* Google Sheets read operation retrieving previously logged prospects.
    - *Configuration:* Sheet Name: `Prospects`, Document ID resolved via configuration sheet URL, `alwaysOutputData: true` enabled.
    - *Key Expressions:* Document ID: `={{ $('Set Search Config').first().json.sheet_url }}`.
    - *Connections:* Input: `Score Profile Gaps`; Output: `Select New Prospects`.
    - *Edge Cases:* Authentication errors or missing spreadsheet columns.
  - **Select New Prospects** (`n8n-nodes-base.code`)
    - *Type & Role:* Filtering node comparing audited profiles against historical contact logs.
    - *Configuration:* Enforces the 90-day cooldown window (`recontact_after_days`) and limits output volume (`prospects_per_run`).
    - *Key Expressions:* Parses `logged_at` timestamps, compares against `Date.now() - cooldownDays * 86400000`. Emits a marker object (`no_new_prospects: true`) if zero fresh prospects remain.
    - *Connections:* Input: `Read Reported Businesses`; Output: `Any New Prospects?`.
    - *Edge Cases:* Malformed dates in historical rows defaulting to active exclusion.
  - **Any New Prospects?** (`n8n-nodes-base.if`)
    - *Type & Role:* Conditional routing node verifying prospect availability.
    - *Configuration:* Evaluates whether `$json.place_id` exists.
    - *Key Expressions:* `={{ $json.place_id }}` existence check.
    - *Connections:* Input: `Select New Prospects`; True Output: `Read Complaint Themes`; False Output: `Nothing New This Week`.
    - *Edge Cases:* Branch divergence handling when no prospects qualify.
  - **Nothing New This Week** (`n8n-nodes-base.noOp`)
    - *Type & Role:* Operational placeholder node representing an empty execution path.
    - *Configuration:* No operational parameters (No-Op).
    - *Connections:* Input: `Any New Prospects?` (false branch); Output: None.
    - *Edge Cases:* None.

---

#### 2.5 AI-Powered Complaint Extraction
- **Overview:** Uses OpenAI (GPT-5.6) to analyze customer reviews rated three stars and below, extracting standardized complaint themes and review sentiments.
- **Nodes Involved:** `Read Complaint Themes`, `OpenAI Review Reader`
- **Node Details:**
  - **Read Complaint Themes** (`@n8n/n8n-nodes-langchain.informationExtractor`)
    - *Type & Role:* LangChain Information Extractor node executing structured text analysis via an LLM.
    - *Configuration:* Schema type set to JSON schema example enforcing output fields: `complaint_themes`, `biggest_complaint`, `review_sentiment`, and `has_complaints`.
    - *Key Expressions:* Dynamically injects business name, category, rating, total reviews, and negative review text into the prompt template.
    - *Connections:* Input: `Any New Prospects?` (true branch); AI Model Input: `OpenAI Review Reader`; Output: `Rank Prospects`.
    - *Edge Cases:* LLM model hallucinations or schema mismatch errors.
  - **OpenAI Review Reader** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
    - *Type & Role:* Language model chat model provider integration.
    - *Configuration:* Model set to `gpt-5.6-terra`, temperature configured to `0.2`.
    - *Connections:* Output (AI Model): Connected to `Read Complaint Themes`.
    - *Edge Cases:* API rate limits, model availability issues, or credential expiration.

---

#### 2.6 Aggregation & Delivery
- **Overview:** Ranks prospects, generates an AI-authored outreach brief, compiles a structured markdown report, logs records to Google Sheets, and publishes the digest to Slack.
- **Nodes Involved:** `Rank Prospects`, `Write Prospect Brief`, `OpenAI Brief Writer`, `Build Report`, `Split Out Prospect Rows`, `Log Prospects To Sheet`, `Post Digest To Slack`
- **Node Details:**
  - **Rank Prospects** (`n8n-nodes-base.code`)
    - *Type & Role:* Data normalization and tally aggregation node.
    - *Configuration:* Processes raw prospect attributes and LLM extraction outputs, standardizing sentiments and calculating common gap/complaint tallies.
    - *Key Expressions:* Combines gap arrays, formats pitches, and summarizes top gaps and complaints.
    - *Connections:* Input: `Read Complaint Themes`; Output: `Write Prospect Brief`.
    - *Edge Cases:* Empty extraction arrays or index mismatching.
  - **Write Prospect Brief** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Type & Role:* LangChain LLM chain node generating a contextual prose marketing brief.
    - *Configuration:* Prompt instructs the model to write 4–6 sentences evaluating outreach priorities based strictly on provided numbers.
    - *Key Expressions:* Injects city, categories, audit counts, common gaps, and serialized prospect objects into the prompt.
    - *Connections:* Input: `Rank Prospects`; AI Model Input: `OpenAI Brief Writer`; Output: `Build Report`.
    - *Edge Cases:* LLM timeout or generation exceeding expected constraints.
  - **OpenAI Brief Writer** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
    - *Type & Role:* Language model chat model provider integration.
    - *Configuration:* Model set to `gpt-5.6-terra`, temperature configured to `0.3`.
    - *Connections:* Output (AI Model): Connected to `Write Prospect Brief`.
    - *Edge Cases:* API quotas or downtime.
  - **Build Report** (`n8n-nodes-base.code`)
    - *Type & Role:* Output formatting node preparing dual payloads for Google Sheets and Slack.
    - *Configuration:* Assembles markdown text message and maps prospect objects into row arrays.
    - *Key Expressions:* Combines metrics, top prospects list, complaint tallies, and brief text into a single cohesive message structure.
    - *Connections:* Input: `Write Prospect Brief`; Output: `Split Out Prospect Rows`.
    - *Edge Cases:* Markdown formatting errors exceeding Slack character limits.
  - **Split Out Prospect Rows** (`n8n-nodes-base.splitOut`)
    - *Type & Role:* Data transformation node expanding array elements into individual items.
    - *Configuration:* Field to split out configured as `rows`.
    - *Connections:* Input: `Build Report`; Output: `Log Prospects To Sheet`.
    - *Edge Cases:* Empty row arrays resulting in zero downstream outputs.
  - **Log Prospects To Sheet** (`n8n-nodes-base.googleSheets`)
    - *Type & Role:* Google Sheets append operation logging individual prospect records.
    - *Configuration:* Operation: `append`, Sheet Name: `Prospects`, Document ID resolved via configuration URL, mapping mode set to auto-map input data.
    - *Key Expressions:* Document ID: `={{ $('Set Search Config').first().json.sheet_url }}`.
    - *Connections:* Input: `Split Out Prospect Rows`; Output: `Post Digest To Slack`.
    - *Edge Cases:* Spreadsheet column mismatches or API quota exhaustion.
  - **Post Digest To Slack** (`n8n-nodes-base.slack`)
    - *Type & Role:* Slack messaging integration node posting the formatted markdown digest.
    - *Configuration:* Resource: `message`, Operation: `post`, Select: `channel`, `executeOnce: true` enabled.
    - *Key Expressions:* Channel ID: `={{ $('Set Search Config').first().json.slack_channel }}`, Message text: `={{ $('Build Report').first().json.message }}`.
    - *Connections:* Input: `Log Prospects To Sheet`; Output: None.
    - *Edge Cases:* Invalid channel names, missing Slack permissions, or webhook delivery failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Prospect Scan | n8n-nodes-base.scheduleTrigger | Triggers workflow weekly | None | Set Search Config | ## Find local businesses with a weak Google profile, and the proof<br><br>Give it a city and a few categories and it comes back with the businesses whose Google Business Profile is measurably behind their neighbours, each one with the specific defect that makes it worth a call.<br><br>### How it works<br>A weekly schedule sends one Google Maps search per category through Bright Data's discovery API. Discovery takes about a minute and a half, so the workflow uses the asynchronous API properly: it triggers the search, then polls the snapshot on a Wait loop with a seven-minute cap until the rows are ready.<br><br>Every profile is then audited against the peers Google itself lists beside it, which is what turns a rating into a judgement: unclaimed listing, missing website, rating below the local median, a heavy tail of one and two star reviews, thin photos, no opening hours. Each gap carries a weight and the total is the score. GPT-5.6 reads only the reviews rated three stars and below, for complaint themes. Every factual claim in the output comes from a field, not from the model.<br><br>Prospects already reported are skipped for ninety days, the whole list lands in Google Sheets, and a ranked digest goes to Slack.<br><br>### Setup<br>Add a Bright Data API key as a Header Auth credential (`Authorization` / `Bearer YOUR_KEY`), connect OpenAI, Google Sheets and Slack, then set your city, categories and channel in Set Search Config.<br><br>### Customization<br>Change the weights in Score Profile Gaps to match what you sell: lead with unclaimed listings for a profile management offer, or with the review gaps for reputation work. |
| Set Search Config | n8n-nodes-base.set | Defines configuration variables | Weekly Prospect Scan | Build Discovery Input | ## 1. Pick the patch<br>One city, one or more categories, on a weekly schedule. Everything you would tune lives in one config node. |
| Build Discovery Input | n8n-nodes-base.code | Formats search payloads | Set Search Config | Discover Businesses With Bright Data | ## 1. Pick the patch<br>One city, one or more categories, on a weekly schedule. Everything you would tune lives in one config node. |
| Discover Businesses With Bright Data | n8n-nodes-base.httpRequest | Triggers Bright Data discovery | Build Discovery Input | Check Discovery Accepted | ## 2. Search Maps asynchronously<br>Bright Data runs one Google Maps search per category. Discovery took about 90 seconds in testing, so this polls the snapshot until the rows are ready.<br><br>---<br><br>## Search one city at a time<br>Bright Data's Maps discovery takes a keyword and a country, with no separate location field, so the city travels inside the keyword. Adding categories adds a search each, and every search adds to the wait. |
| Check Discovery Accepted | n8n-nodes-base.code | Validates snapshot ID | Discover Businesses With Bright Data | Wait For Snapshot | ## 2. Search Maps asynchronously<br>Bright Data runs one Google Maps search per category. Discovery took about 90 seconds in testing, so this polls the snapshot until the rows are ready. |
| Wait For Snapshot | n8n-nodes-base.wait | Pauses between poll requests | Check Discovery Accepted, Profiles Ready? | Check Snapshot Progress | ## 2. Search Maps asynchronously<br>Bright Data runs one Google Maps search per category. Discovery took about 90 seconds in testing, so this polls the snapshot until the rows are ready. |
| Check Snapshot Progress | n8n-nodes-base.httpRequest | Polls snapshot status | Wait For Snapshot | Evaluate Scrape Progress | ## 2. Search Maps asynchronously<br>Bright Data runs one Google Maps search per category. Discovery took about 90 seconds in testing, so this polls the snapshot until the rows are ready. |
| Evaluate Scrape Progress | n8n-nodes-base.code | Tracks poll attempts and limits | Check Snapshot Progress | Profiles Ready? | ## 2. Search Maps asynchronously<br>Bright Data runs one Google Maps search per category. Discovery took about 90 seconds in testing, so this polls the snapshot until the rows are ready. |
| Profiles Ready? | n8n-nodes-base.if | Checks if snapshot is ready | Evaluate Scrape Progress | Download Business Profiles, Wait For Snapshot | ## 2. Search Maps asynchronously<br>Bright Data runs one Google Maps search per category. Discovery took about 90 seconds in testing, so this polls the snapshot until the rows are ready. |
| Download Business Profiles | n8n-nodes-base.httpRequest | Downloads completed dataset | Profiles Ready? | Score Profile Gaps | ## 2. Search Maps asynchronously<br>Bright Data runs one Google Maps search per category. Discovery took about 90 seconds in testing, so this polls the snapshot until the rows are ready. |
| Score Profile Gaps | n8n-nodes-base.code | Audits and scores profiles | Download Business Profiles | Read Reported Businesses | ## 3. Audit every profile<br>Each listing is scored on the gaps you can prove, benchmarked against the peers Google lists beside it. |
| Read Reported Businesses | n8n-nodes-base.googleSheets | Reads historical log sheet | Score Profile Gaps | Select New Prospects | ## 4. Drop the ones you have had<br>The sheet doubles as the do-not-repeat list, so a business reported recently stays out of this week's digest. |
| Select New Prospects | n8n-nodes-base.code | Filters out recent contacts | Read Reported Businesses | Any New Prospects? | ## 4. Drop the ones you have had<br>The sheet doubles as the do-not-repeat list, so a business reported recently stays out of this week's digest. |
| Any New Prospects? | n8n-nodes-base.if | Checks for available prospects | Select New Prospects | Read Complaint Themes, Nothing New This Week | ## 4. Drop the ones you have had<br>The sheet doubles as the do-not-repeat list, so a business reported recently stays out of this week's digest. |
| Nothing New This Week | n8n-nodes-base.noOp | Fallback for zero prospects | Any New Prospects? | None | ## 4. Drop the ones you have had<br>The sheet doubles as the do-not-repeat list, so a business reported recently stays out of this week's digest. |
| Read Complaint Themes | @n8n/n8n-nodes-langchain.informationExtractor | Extracts review complaints via LLM | Any New Prospects? | Rank Prospects | ## 5. Read the bad reviews<br>The model sees only reviews rated three stars and below, and returns complaint themes. Everything stated as fact about a business comes from a field, not from the model. |
| OpenAI Review Reader | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for review reading | None | Read Complaint Themes | ## 5. Read the bad reviews<br>The model sees only reviews rated three stars and below, and returns complaint themes. Everything stated as fact about a business comes from a field, not from the model. |
| Rank Prospects | n8n-nodes-base.code | Aggregates and ranks prospects | Read Complaint Themes | Write Prospect Brief | ## 6. Deliver<br>One sheet row per prospect, which is what the next run reads back, plus a ranked Slack digest. |
| Write Prospect Brief | @n8n/n8n-nodes-langchain.chainLlm | Generates marketing brief prose | Rank Prospects | Build Report | ## 6. Deliver<br>One sheet row per prospect, which is what the next run reads back, plus a ranked Slack digest. |
| OpenAI Brief Writer | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for brief generation | None | Write Prospect Brief | ## 6. Deliver<br>One sheet row per prospect, which is what the next run reads back, plus a ranked Slack digest. |
| Build Report | n8n-nodes-base.code | Formats report and Slack message | Write Prospect Brief | Split Out Prospect Rows | ## 6. Deliver<br>One sheet row per prospect, which is what the next run reads back, plus a ranked Slack digest. |
| Split Out Prospect Rows | n8n-nodes-base.splitOut | Splits row array into items | Build Report | Log Prospects To Sheet | ## 6. Deliver<br>One sheet row per prospect, which is what the next run reads back, plus a ranked Slack digest. |
| Log Prospects To Sheet | n8n-nodes-base.googleSheets | Appends prospects to sheet | Split Out Prospect Rows | Post Digest To Slack | ## 6. Deliver<br>One sheet row per prospect, which is what the next run reads back, plus a ranked Slack digest. |
| Post Digest To Slack | n8n-nodes-base.slack | Posts Slack digest message | Log Prospects To Sheet | None | ## 6. Deliver<br>One sheet row per prospect, which is what the next row reads back, plus a ranked Slack digest. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) named `Weekly Prospect Scan`.
   - Set interval rule to run daily at hour `7`, minute `0`.

2. **Configure Global Variables:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Set Search Config`.
   - Add string assignments: `city` (`Austin, TX`), `categories` (`dentist, chiropractor`), `country` (`US`), `sheet_url` (`https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID`), `slack_channel` (`#prospecting`).
   - Add number assignments: `businesses_per_category` (`20`), `min_reviews` (`10`), `min_gap_score` (`15`), `prospects_per_run` (`8`), `recontact_after_days` (`90`).
   - Connect `Weekly Prospect Scan` to `Set Search Config`.

3. **Build Discovery Input:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Build Discovery Input`.
   - Set execution mode to **Run Once for All Items**.
   - Paste JavaScript to split categories by comma and map them into `{ keyword: '${category} in ${city}', country: ... }` payloads.
   - Connect `Set Search Config` to `Build Discovery Input`.

4. **Trigger Bright Data Discovery:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Discover Businesses With Bright Data`.
   - Set method to `POST`, URL to `https://api.brightdata.com/datasets/v3/trigger?dataset_id=gd_m8ebnr0q2qlklc02fz&format=json&include_errors=true&type=discover_new&discover_by=location&limit_per_input={{ $('Set Search Config').first().json.businesses_per_category }}`.
   - Configure Authentication to **Generic Credential Type** -> **HTTP Header Auth** (`Authorization: Bearer <key>`).
   - Enable `Specify Body` as `JSON`, passing `={{ JSON.stringify($json.payload) }}`. Set options: Timeout `120000ms`, `neverError: true`, retries `3`.
   - Connect `Build Discovery Input` to `Discover Businesses With Bright Data`.

5. **Validate Discovery Response:**
   - Add a **Code** node named `Check Discovery Accepted`.
   - Verify `res.snapshot_id` exists and throw errors if `res.error` is present.
   - Connect `Discover Businesses With Bright Data` to `Check Discovery Accepted`.

6. **Implement Polling Loop:**
   - Add a **Wait** node (`n8n-nodes-base.wait`) named `Wait For Snapshot` set to pause for `20` seconds.
   - Add an **HTTP Request** node named `Check Snapshot Progress` with method `GET`, URL `=https://api.brightdata.com/datasets/v3/progress/{{ $('Check Discovery Accepted').first().json.snapshot_id }}`. Use HTTP Header Auth, timeout `60000ms`, retries `3`, `neverError: true`.
   - Add a **Code** node named `Evaluate Scrape Progress` to track attempts (max 20 polls) and status.
   - Add an **If** node (`n8n-nodes-base.if`) named `Profiles Ready?` checking if `={{ $json.status }}` equals `ready`.
   - Connect `Check Discovery Accepted` -> `Wait For Snapshot` -> `Check Snapshot Progress` -> `Evaluate Scrape Progress` -> `Profiles Ready?`. Connect the False branch of `Profiles Ready?` back to `Wait For Snapshot`.

7. **Download Business Profiles:**
   - Add an **HTTP Request** node named `Download Business Profiles` with method `GET`, URL `=https://api.brightdata.com/datasets/v3/snapshot/{{ $('Check Discovery Accepted').first().json.snapshot_id }}?format=json`. Use HTTP Header Auth, timeout `180000ms`, retries `3`, `neverError: true`.
   - Connect the True branch of `Profiles Ready?` to `Download Business Profiles`.

8. **Score Profile Gaps:**
   - Add a **Code** node named `Score Profile Gaps`.
   - Implement logic to calculate peer medians from `people_also_search`, compute negative review percentages, and accumulate weighted gap penalties. Filter out profiles below `min_reviews` or `min_gap_score`.
   - Connect `Download Business Profiles` to `Score Profile Gaps`.

9. **Filter Historical Prospects:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Read Reported Businesses`. Operation: Get/Read, Sheet Name: `Prospects`, Document ID: `={{ $('Set Search Config').first().json.sheet_url }}`, enable `alwaysOutputData: true`.
   - Add a **Code** node named `Select New Prospects` to filter out records reported within `recontact_after_days` (90 days).
   - Add an **If** node named `Any New Prospects?` checking if `={{ $json.place_id }}` exists.
   - Add a **No-Op** node (`n8n-nodes-base.noOp`) named `Nothing New This Week` connected to the False branch of `Any New Prospects?`.
   - Connect `Score Profile Gaps` -> `Read Reported Businesses` -> `Select New Prospects` -> `Any New Prospects?`.

10. **Extract Complaint Themes via AI:**
    - Add an **Information Extractor** node (`@n8n/n8n-nodes-langchain.informationExtractor`) named `Read Complaint Themes`.
    - Configure JSON schema example with fields `complaint_themes`, `biggest_complaint`, `review_sentiment`, and `has_complaints`.
    - Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) named `OpenAI Review Reader`. Set model to `gpt-5.6-terra`, temperature `0.2`. Connect its AI model output to `Read Complaint Themes`.
    - Connect the True branch of `Any New Prospects?` to `Read Complaint Themes`.

11. **Rank Prospects & Generate Brief:**
    - Add a **Code** node named `Rank Prospects` to normalize themes, calculate sentiments, and build pitch texts.
    - Add a **LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`) named `Write Prospect Brief` prompting an AI marketing consultant to summarize outreach priorities.
    - Add an **OpenAI Chat Model** node named `OpenAI Brief Writer`. Set model to `gpt-5.6-terra`, temperature `0.3`. Connect its AI model output to `Write Prospect Brief`.
    - Connect `Read Complaint Themes` -> `Rank Prospects` -> `Write Prospect Brief`.

12. **Build Report, Log, and Deliver:**
    - Add a **Code** node named `Build Report` to compile markdown messages and format row objects.
    - Add a **Split Out** node (`n8n-nodes-base.splitOut`) named `Split Out Prospect Rows` splitting `rows`.
    - Add a **Google Sheets** node named `Log Prospects To Sheet`. Operation: `append`, Sheet Name: `Prospects`, Document ID: `={{ $('Set Search Config').first().json.sheet_url }}`, mapping mode: `autoMapInputData`.
    - Add a **Slack** node (`n8n-nodes-base.slack`) named `Post Digest To Slack`. Resource: `message`, Operation: `post`, Select: `channel`, Channel ID: `={{ $('Set Search Config').first().json.slack_channel }}`, Message: `={{ $('Build Report').first().json.message }}`, with `executeOnce: true`.
    - Connect `Write Prospect Brief` -> `Build Report` -> `Split Out Prospect Rows` -> `Log Prospects To Sheet` -> `Post Digest To Slack`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Bright Data Dataset API Documentation | Used for asynchronous Google Maps discovery (`gd_m8ebnr0q2qlklc02fz`). |
| OpenAI GPT-5.6 Model Integration | Utilized for structured review complaint extraction and brief generation (`gpt-5.6-terra`). |
| Google Sheets Do-Not-Repeat Tracking | The `Prospects` sheet functions simultaneously as an audit log and a 90-day cooldown filter. |