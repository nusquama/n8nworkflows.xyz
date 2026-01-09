Automate product Hunt lead generation with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/automate-product-hunt-lead-generation-with-apify-and-google-sheets-11947


# Automate product Hunt lead generation with Apify and Google Sheets

## 1. Workflow Overview

**Workflow name:** Product Hunt Lead Generator  
**Purpose:** Automatically scrape Product Hunt (daily + optional weekly), wait for Apify run completion via webhook, pull datasets, deduplicate and validate products, enrich each product with contact details (Apify contact scraper), optionally send outreach emails via Gmail, and log every outcome + daily summary into Google Sheets. A small n8n **Data Table** is used as persistent state (webhook created flag, current run IDs/dataset IDs, and sheet metadata).

**Typical use cases**
- Daily lead generation from Product Hunt with contact enrichment.
- Weekly “top products” expansion (runs on Monday in this config).
- Outreach automation (Gmail) with tracking in Google Sheets.

### Logical Blocks
1.1 **Entry points & config building** (Schedule trigger vs webhook trigger)  
1.2 **Webhook existence check & creation** (one-time Apify webhook creation)  
1.3 **Run Apify scrapes (daily/weekly) and store run state**  
1.4 **Webhook event handling: update RUNS state and decide next action**  
1.5 **Fetch datasets, deduplicate products, batch loop**  
1.6 **Reachability check + category mapping**  
1.7 **Contact enrichment (Apify contact scraper)**  
1.8 **Validation & shaping output** (skip logic, final record body)  
1.9 **Logging to Google Sheets + daily summary + cleanup**  
1.10 **Optional lead emailing and email status logging**

---

## 2. Block-by-Block Analysis

### 1.1 Entry points & config building

**Overview:** The workflow can start either on a daily schedule (9AM) or when Apify calls back the webhook indicating a run succeeded. A shared configuration object is built and used everywhere.

**Nodes involved**
- Schedule Trigger1
- Receive Apify Webhook2
- Build Config2
- Check Schedule Trigger2

#### Node: Schedule Trigger1
- **Type/role:** `Schedule Trigger` – starts daily automation
- **Config:** runs **every day at 9 AM**
- **Outputs:** to **Build Config2**
- **Edge cases:** server timezone differences; if n8n timezone is not expected, “9AM” may not match local time.

#### Node: Receive Apify Webhook2
- **Type/role:** `Webhook` – entry point for Apify run events
- **Config:** POST `path=product-hunt-scraper-workflow`
- **Payload expectation:** includes `resource.id` (Apify run ID) and `eventType`
- **Outputs:** to **Build Config2**
- **Edge cases:** Apify may send different payload shapes; the downstream code attempts both `body.resource.id` and `resource.id`.

#### Node: Build Config2
- **Type/role:** `Code` – builds global config/state
- **Key outputs (interpreted):**
  - `isScheduleTrigger`: true if input contains `timestamp` (schedule execution)
  - `dataTableId`: hardcoded to `3mOJkgv9zEcXiZP3`
  - `weeklyDay`: `"Monday"` and `currentDay` from `$now`
  - `isSameDay`: computed (currentDay == weeklyDay)
  - `externalScraper`: Apify base URL + payloads:
    - `payload.day`: daily scrape with `archiveDate`, `topNProducts=3`, `saveWebsiteContent=true`
    - `payload.week`: weekly scrape with `archiveWeek=$now.weekNumber - 1`, `topNProducts=100`
    - `payload.webhook`: Apify webhook definition (points to this n8n webhook URL)
    - `payload.contacts`: contact-scraper preferences (socials, depth, browser, etc.)
  - `email`: metadata URLs (privacy/terms), fromEmail, etc. (some not used directly later)
  - `batch`: size=10, delay=6.0 (not fully implemented as delay node here)
- **Outputs:** to **Check Schedule Trigger2**
- **Edge cases/failures:**
  - `dataTableId` must exist and have expected schema.
  - Weekly logic uses `$now.weekNumber - 1`; at week 1 this becomes 0 (may be invalid for Apify actor).
  - Webhook `requestUrl` must be public/reachable from Apify.

#### Node: Check Schedule Trigger2
- **Type/role:** `IF` – route logic depending on entry point
- **Condition:** `$json.isScheduleTrigger == true`
- **True branch:** schedule flow → webhook setup + create log sheet + start runs  
- **False branch:** webhook flow → fetch RUNS rows and update status
- **Edge cases:** if the schedule trigger payload format changes (no timestamp), it may incorrectly route to webhook branch.

---

### 1.2 Webhook existence check & creation (schedule branch)

**Overview:** On scheduled runs, the workflow checks a Data Table flag to avoid creating the Apify webhook multiple times. If missing, it creates it and stores a flag row.

**Nodes involved**
- Fetch Webhook Status2
- Check Webhook Exists2
- Create External Webhook2
- Mark Webhook as Created2

#### Node: Fetch Webhook Status2
- **Type/role:** `Data Table` – query state
- **Config:** `get`, limit 1, filters: `key=IS_WEBHOOK_CREATED` and `value=1`
- **Input:** from **Check Schedule Trigger2** (true branch)
- **Output:** to **Check Webhook Exists2**
- **Edge cases:** Data Table credentials/permissions; missing table; filter mismatch.

#### Node: Check Webhook Exists2
- **Type/role:** `IF`
- **Condition:** `$json` is **not empty**
- **True branch:** webhook exists → proceed to **Check Weekly Run (Monday)2**  
- **False branch:** create webhook → **Create External Webhook2**
- **Edge cases:** `alwaysOutputData` on Fetch Webhook Status2 means it can output empty structures; “notEmpty” against `$json` can behave unexpectedly depending on n8n version/item shape.

#### Node: Create External Webhook2
- **Type/role:** `HTTP Request` – create Apify webhook
- **Config:**
  - POST `{{ baseUrl }}/webhooks`
  - JSON body: `externalScraper.payload.webhook` from Build Config2
  - Auth: Bearer token (`httpBearerAuth`)
- **Output:** to **Mark Webhook as Created2**
- **Failure modes:** invalid Apify token; webhook quota; invalid requestUrl; Apify API outage.

#### Node: Mark Webhook as Created2
- **Type/role:** `Data Table` – insert state flag
- **Config:** insert row `{ key: IS_WEBHOOK_CREATED, value: 1 }`
- **Output:** to **Check Weekly Run (Monday)2**
- **Edge cases:** duplicates: this inserts a new row each time if called repeatedly; ideally should upsert/update.

---

### 1.3 Run Apify scrapes (daily/weekly) and store run state

**Overview:** Depending on day-of-week, the workflow launches a weekly run (Monday) plus the daily run. It then merges run IDs/dataset IDs into a tracking object and stores it in the Data Table.

**Nodes involved**
- Check Weekly Run (Monday)2
- Run Actor "Weekly Top Products"2
- Run Actor "Daily Top Products"2
- Combine Run IDs2
- Store Run Data2

#### Node: Check Weekly Run (Monday)2
- **Type/role:** `IF`
- **Condition:** `Build Config2.json.isSameDay == true`
- **True branch (Monday):** run weekly → then daily  
- **False branch:** run daily only
- **Edge cases:** locale of `$now.toFormat('cccc')` affects day strings; ensure “Monday” matches locale.

#### Node: Run Actor "Weekly Top Products"2
- **Type/role:** `HTTP Request` – starts Apify actor run
- **Config:** POST `.../acts/danpoletaev~product-hunt-scraper/runs`
- **Body:** `externalScraper.payload.week`
- **Auth:** Bearer (Apify)
- **Output:** chained into **Run Actor "Daily Top Products"2**
- **Failures:** token, invalid payload, Apify actor not found, rate limits.

#### Node: Run Actor "Daily Top Products"2
- **Type/role:** `HTTP Request` – starts daily Apify actor run
- **Body:** `externalScraper.payload.day`
- **Output:** to **Combine Run IDs2**
- **Edge cases:** daily payload uses `archiveDate` and `topNProducts=3` (small sample); adjust as needed.

#### Node: Combine Run IDs2
- **Type/role:** `Code` – unify run IDs and datasets
- **Logic:**
  - Creates `ids` object keyed by runId:
    - `status: false`
    - `datasetId: <defaultDatasetId>`
  - If Monday: includes weekly run + daily run (expects 2 total)
  - Else: only daily run (expects 1 total)
  - Adds `succeeded: 0`
- **Output:** to **Store Run Data2**
- **Failure modes:**
  - If weekly branch didn’t run but `isSameDay` is true due to config error, it will reference missing node output.
  - Assumes Apify response shape: `data.id`, `data.defaultDatasetId`.

#### Node: Store Run Data2
- **Type/role:** `Data Table` – persistent storage
- **Config:** insert `{ key: RUNS, value: JSON.stringify($json) }`
- **Purpose:** allow webhook-triggered runs to update status later
- **Edge cases:** multiple RUNS rows can accumulate; later fetch uses `returnAll: true`.

---

### 1.4 Webhook event handling: update RUNS and decide next action

**Overview:** When Apify webhook arrives, the workflow loads RUNS rows, updates the matching run’s status and succeeded count, writes back, then routes based on whether all required runs are completed.

**Nodes involved**
- Fetch RUNS Data2
- Update Run Status2
- Save Updated RUNS Data1
- Switch

#### Node: Fetch RUNS Data2
- **Type/role:** `Data Table` – fetch run tracking rows
- **Config:** `get`, `returnAll=true`, filter `key = RUNS`
- **Input:** from **Check Schedule Trigger2** (false branch)
- **Output:** to **Update Run Status2**
- **Edge cases:** multiple RUNS rows; the code scans all.

#### Node: Update Run Status2
- **Type/role:** `Code`
- **Key expressions/variables:**
  - Reads `webId` from webhook: `wh.body.resource.id` fallback `wh.resource.id`
  - Parses each row `item.json.value` as JSON
  - If `payload.ids[webId]` exists:
    - sets `status=true`
    - increments `payload.succeeded`
    - re-stringifies and returns the updated row (only one)
  - If no match: returns empty array (prevents updates)
- **Output:** to **Save Updated RUNS Data1**
- **Failure modes:** malformed JSON in Data Table; unexpected webhook payload; runId not found; returning empty means nothing proceeds downstream.

#### Node: Save Updated RUNS Data1
- **Type/role:** `Data Table` – update the matched RUNS row
- **Config:** `update` where `id == $json.id`, auto-map columns
- **Output:** to **Switch**
- **Edge cases:** If Update Run Status2 returns nothing, this node receives no items and won’t run.

#### Node: Switch
- **Type/role:** `Switch` – routing after status update
- **Rules (intended):**
  - **D&W:** Monday and succeeded == 2 → proceed
  - **onlyDay:** not Monday and succeeded == 1 → proceed
  - **wrD&W / wrD:** waiting cases (should re-wait; here they still route to dataset extraction)
- **Configured behavior:** only two outputs are actually wired and both go to **Extract Dataset IDs2**.
- **Important issue:** The “waiting” rules are logically inconsistent:
  - Some compare boolean expressions but set `rightValue` to `"false"` or blank.
  - First rule uses string equals against `"true"` while leftValue is a boolean expression.
- **Practical consequence:** In many cases, the switch may still emit to one of the first two outputs or default behavior depending on n8n evaluation. Given both connected outputs go to the same next node, the switch currently doesn’t truly “wait” and can attempt dataset processing before all runs finished.
- **Edge cases:** race condition: daily run finishes first on Monday, triggers processing while weekly still running.

---

### 1.5 Fetch datasets, deduplicate products, batch loop

**Overview:** Once routing proceeds, the workflow extracts dataset IDs from RUNS, fetches items from Apify datasets, removes duplicates, then processes items in batches.

**Nodes involved**
- Extract Dataset IDs2
- Fetch Dataset Items2
- Deduplicate2
- Loop Over Sub Items2
- Aggregate Batch Results2

#### Node: Extract Dataset IDs2
- **Type/role:** `Code`
- **Logic:** parses `value` from first RUNS row, iterates `Object.values(parsed.ids)` and returns one item per `{ datasetId }`
- **Output:** to **Fetch Dataset Items2**
- **Edge cases:** if multiple RUNS rows exist, it only reads the first; could miss newer row.

#### Node: Fetch Dataset Items2
- **Type/role:** `HTTP Request`
- **Config:** GET `{{ baseUrl }}/datasets/{{ $json.datasetId }}/items`
- **Auth:** Bearer (Apify)
- **Output:** to **Deduplicate2**
- **Failure modes:** dataset not ready yet; permission denied; large dataset responses.

#### Node: Deduplicate2
- **Type/role:** `Code` – removes duplicates by product name
- **Logic:** unique key `name`, normalized by stripping numeric prefix `^\d+\.\s*`
- **Output:** to **Loop Over Sub Items2**
- **Failure modes:** if `name` is null, `.replace()` will throw (code assumes string). This is a real edge case for incomplete dataset items.

#### Node: Loop Over Sub Items2
- **Type/role:** `Split In Batches`
- **Config:** defaults (batch size not explicitly set in node params here)
- **Connections:**
  - Main output 0 → **Aggregate Batch Results2** (acts as “done/collect” path)
  - Main output 1 → **HTTP Request** (per-item processing path)
- **Edge cases:** if batch size default is not desired, processing volume/time may explode.

#### Node: Aggregate Batch Results2
- **Type/role:** `Aggregate`
- **Purpose:** collect results after loop iterations
- **Output:** to **Get Daily Activities to Sheet2**
- **Edge cases:** fieldsToAggregate config is essentially empty; depending on n8n version, it may not aggregate as intended.

---

### 1.6 Reachability check + category mapping

**Overview:** For each product, the workflow pings the website URL (simple HTTP GET) to mark reachability, then maps categories based on keyword matching.

**Nodes involved**
- HTTP Request
- If
- Map Category by Topic2

#### Node: HTTP Request
- **Type/role:** `HTTP Request` – website reachability check
- **Config:** URL = `Loop Over Sub Items2.item.json.websiteUrl`
- **Options:** `neverError: true`, `fullResponse: true`, `onError: continueErrorOutput`
- **Outputs:**
  - Success output → **If**
  - Error output → **Loop Over Sub Items2** (continues loop)
- **Failure modes:** DNS/timeouts; non-200 responses treated as “not reachable”.

#### Node: If
- **Type/role:** `IF` – classify reachability
- **Condition:** `statusCode == 200`
- **True branch:** → **Map Category by Topic2**
- **False branch:** → **Loop Over Sub Items2** (skips further processing for that item)
- **Important limitation:** This “reachability” is strict; 301/302/403/429 are common and may still mean the site exists.

#### Node: Map Category by Topic2
- **Type/role:** `Code` – keyword-based categorization
- **Logic:**
  - Builds mapping of topic keywords → categories (AI, Dev Tools, etc.)
  - Computes scores from item `categories` field (accepts array or comma-separated string)
  - Selects top category by score and priority
  - Adds `category`, `categories` (1–3), and `isReachable`
- **Input:** uses `$('Loop Over Sub Items2').all()` inside code, then maps items
- **Edge cases/bugs:**
  - It sets `const isReachable = $input.first().json.isReachable;` but reachability is not actually injected from the HTTP check in this workflow. The HTTP node returns statusCode, not `isReachable`.
  - It also reads from `Loop Over Sub Items2.all()` rather than current item, which is unusual and can lead to incorrect cross-item data.

---

### 1.7 Contact enrichment (Apify contact scraper)

**Overview:** For each categorized product, the workflow builds a payload and calls Apify’s contact-info-scraper synchronously to retrieve emails, phones, and social profiles.

**Nodes involved**
- Prepare Scraper Payload1
- Get Contact Scraper1
- If1

#### Node: Prepare Scraper Payload1
- **Type/role:** `Code` – build payload for contact scraper
- **Config (interpreted):**
  - contact scraping options: depth, max requests, merge contacts, sameDomain, scrape social profiles, useBrowser
  - `startUrls[0].url` set to `Map Category by Topic2.first().json.websiteUrl`
- **Output:** to **Get Contact Scraper1**
- **Edge cases:** It references `websiteUrl` but later nodes use `project_url` / `item.website.url`. Inconsistent field naming can break scraping.

#### Node: Get Contact Scraper1
- **Type/role:** `HTTP Request` – synchronous Apify actor endpoint
- **Config:** POST `https://api.apify.com/v2/acts/vdrmota~contact-info-scraper/run-sync-get-dataset-items`
- **Body:** `{{$json}}` from previous code node
- **Auth:** Bearer (Apify)
- **Options:** `retryOnFail: true`, `alwaysOutputData: true`
- **Output:** to **If1**
- **Failure modes:** long runtime/timeouts; actor quota; returns empty dataset if blocked by bot protection.

#### Node: If1
- **Type/role:** `IF` – check contact scraper result exists
- **Condition:** `$json` is not empty
- **True:** → **Prepare Request Body2**
- **False:** → **Loop Over Sub Items2** (skip item)
- **Edge cases:** If actor returns empty array vs empty object, “notEmpty” behavior may vary.

---

### 1.8 Validation & shaping output (skip logic, final record)

**Overview:** This block validates the product (description length, URL exists, reachability), trims short description, and shapes a final normalized object containing product + contact info. Items can be marked `_skip=true`.

**Nodes involved**
- Prepare Request Body2
- Get Sheet Metadata2
- Parse Sheet IDs2
- Check if Skipped2
- Prepare Error Row2
- Format API Result for Sheet2

#### Node: Prepare Request Body2
- **Type/role:** `Code` – data validation and normalization
- **Key logic:**
  - Reads product from `Map Category by Topic2` and contact from `Get Contact Scraper1`
  - Skip conditions:
    - description < 50 chars
    - missing website URL
    - website not accessible (`isReachable === false`)
  - If skipped: outputs `{ title, project_url, skip_reason, _skip: true }`
  - Else: outputs normalized object with:
    - title, descriptions, project_url, logo/banner
    - `category` + `categories` (up to 3)
    - social links from contact scraper
    - `contact_info` with primary + other emails/phones
- **Output:** to **Get Sheet Metadata2**
- **Edge cases/bugs:**
  - It pulls `const item = $("Map Category by Topic2").all()[0]` (always the first item), not current loop item.
  - `isReachable` is taken from item, but upstream does not properly assign it.
  - The skip error uses `skip_reason`, but later Prepare Error Row2 expects `_skip` content as errorMessage (it uses `req._skip` instead of `skip_reason`).

#### Node: Get Sheet Metadata2
- **Type/role:** `Data Table` – retrieve stored sheet identifiers
- **Config:** get row where `key=SHEET`, limit 1
- **Output:** to **Parse Sheet IDs2**
- **Failure modes:** missing SHEET record if schedule branch didn’t create it first.

#### Node: Parse Sheet IDs2
- **Type/role:** `Code`
- **Logic:** `sheet = JSON.parse($input.first().json.value)`
- **Output:** to **Check if Skipped2**
- **Edge cases:** malformed JSON in Data Table.

#### Node: Check if Skipped2
- **Type/role:** `IF`
- **Condition:** `Prepare Request Body2.json._skip` exists and is true
- **True:** → **Prepare Error Row2**
- **False:** → **Format API Result for Sheet2**
- **Edge cases:** `_skip` is boolean; if it’s missing, it routes to non-skip path.

#### Node: Prepare Error Row2
- **Type/role:** `Code`
- **Purpose:** build a Google Sheets row for skipped items
- **Outputs fields:** timestamp, productName, productHuntUrl, websiteUrl, contactEmail, category, status=error, errorMessage, projectId=null, magicLink=null, emailSent=no
- **Issue:** sets `errorMessage: req._skip || ""` (boolean) instead of `skip_reason`
- **Output:** to **Log API Result to Google Sheet2**

#### Node: Format API Result for Sheet2
- **Type/role:** `Code`
- **Purpose:** create a standardized “body” object for logging + later email decisions
- **Output:** to **Check Owner Email Exists2**
- **Edge cases:** It references `input.owner_email` (commented) but current workflow mostly uses `contact_info.primary_email`.

---

### 1.9 Logging to Google Sheets + daily summary + cleanup

**Overview:** The schedule branch creates a new Google spreadsheet daily and stores its IDs in Data Table. Later, each processed item appends a row to “Daily Activities”. At end, the workflow reads all activity rows, summarizes by date, appends summary, and clears old Data Table rows.

**Nodes involved**
- Create Daily Log Sheet2
- Extract Sheet Info2
- Store Sheet Metadata2
- Log API Result to Google Sheet2
- Get Daily Activities to Sheet2
- Summarize Daily Stats2
- Log Daily Summary to Sheet2
- Clear Old Run Data2

#### Node: Create Daily Log Sheet2
- **Type/role:** `Google Sheets` – create spreadsheet
- **Config:**
  - Title: `product_import_log_YYYY_MM_DD` based on Schedule Trigger timestamp
  - Creates 2 tabs: “Daily Activities”, “Daily Summary”
- **Output:** to **Extract Sheet Info2**
- **Failure modes:** OAuth scope/consent; Google API quota; invalid timestamp.

#### Node: Extract Sheet Info2
- **Type/role:** `Code`
- **Logic:** extracts `spreadsheetId` and both `sheetId`s, stores as stringified JSON in field `sheet`
- **Output:** to **Store Sheet Metadata2**

#### Node: Store Sheet Metadata2
- **Type/role:** `Data Table` – store sheet identifiers
- **Config:** insert `{ key: SHEET, value: <stringified> }`
- **Edge cases:** repeated schedule runs create multiple SHEET rows; later fetch uses limit 1 (may pick older one).

#### Node: Log API Result to Google Sheet2
- **Type/role:** `Google Sheets` – append row into “Daily Activities”
- **Config:**
  - operation: append
  - documentId = `Parse Sheet IDs2.sheet.spreadsheetId`
  - sheetName uses the **sheet ID** for “dailyActivities”
  - auto map input columns (expects keys timestamp/productName/…)
- **Output:** back to **Loop Over Sub Items2** (continues processing)
- **Failure modes:** mismatch between schema and input keys; using sheet ID vs name depending on n8n resource locator behavior.

#### Node: Get Daily Activities to Sheet2
- **Type/role:** `Google Sheets` – read all activity rows
- **Output:** to **Summarize Daily Stats2**

#### Node: Summarize Daily Stats2
- **Type/role:** `Code`
- **Logic:** group by date (YYYY-MM-DD from timestamp), compute totals, success/duplicate/error counts, email sent/failed, failureRate
- **Output:** to **Log Daily Summary to Sheet2**
- **Edge cases:** expects `status` and `emailSent` fields in rows; if missing, counts may be wrong.

#### Node: Log Daily Summary to Sheet2
- **Type/role:** `Google Sheets` – append summary rows to “Daily Summary”
- **Output:** to **Clear Old Run Data2**

#### Node: Clear Old Run Data2
- **Type/role:** `Data Table` – cleanup
- **Config:** delete rows where `key != IS_WEBHOOK_CREATED`
- **Purpose:** reset state for next run while preserving webhook-created flag
- **Edge cases:** if webhook run arrives after cleanup, RUNS row may be missing.

---

### 1.10 Optional lead emailing and email status logging

**Overview:** For non-skipped items, the workflow checks if a primary email exists, prepares CC addresses, sends an HTML outreach email via Gmail, then records email result (yes/failed/no) into the Google Sheet activity row.

**Nodes involved**
- Check Owner Email Exists2
- Prepare CC
- Send a message
- Verify Email Sent Successfully2
- Handle No Email
- Handle Email Failed
- Handle Email Success

#### Node: Check Owner Email Exists2
- **Type/role:** `IF`
- **Condition:** `Prepare Request Body2.json.contact_info.primary_email` exists
- **True:** → **Prepare CC** then **Send a message**
- **False:** → **Handle No Email**
- **Edge cases:** “exists” does not ensure non-empty string; may attempt to email `""`.

#### Node: Prepare CC
- **Type/role:** `Set` – format CC list
- **Logic:** `cc = other_emails.join(',') || null` from `Prepare Request Body2.contact_info.other_emails`
- **Output:** to **Send a message**
- **Edge cases:** if `other_emails` is not an array, `.join` fails; code uses optional chaining, so safer.

#### Node: Send a message
- **Type/role:** `Gmail` – send outreach email
- **Config highlights:**
  - To: `Prepare Request Body2.contact_info.primary_email`
  - CC: `{{$json.cc}}` (from Prepare CC)
  - Subject: `AI + Automation services for "<title>"`
  - HTML template embedded (contains “Product Hunt launch” pitch)
  - `appendAttribution=false`
  - `onError=continueRegularOutput` (so failures still produce output)
- **Failure modes:** Gmail OAuth expired; sending limits; invalid recipient; HTML template uses a greeting line that injects email address as name.

#### Node: Verify Email Sent Successfully2
- **Type/role:** `IF`
- **Condition (as configured):** checks `$json.error` exists and notEmpty
- **Outputs:**
  - True → **Handle Email Failed**
  - False → **Handle Email Success**
- **Important issue:** Naming is misleading: it checks for presence of `error`, so the “failed” branch triggers when `error` exists. That is consistent, but the node’s description says “accepted and delivered” which is not what’s implemented.
- **Edge cases:** Gmail node output format may not include `error` even when partial failure occurs.

#### Node: Handle No Email
- **Type/role:** `Code`
- **Output:** merges formatted record and sets `emailSent="no"`
- **Output to:** **Log API Result to Google Sheet2**

#### Node: Handle Email Failed
- **Type/role:** `Code`
- **Output:** `status="success"` but `emailSent="failed"` and `errorMessage=<error>`
- **Issue:** status remains “success” even if email failed; you may want a separate status or keep error but clarify.
- **Output to:** **Log API Result to Google Sheet2**

#### Node: Handle Email Success
- **Type/role:** `Code`
- **Output:** `emailSent="yes"`, clears errorMessage
- **Output to:** **Log API Result to Google Sheet2**

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger1 | Schedule Trigger | Daily 9AM entry point | — | Build Config2 |  |
| Receive Apify Webhook2 | Webhook | Apify callback entry point | — | Build Config2 | ## Get the dataset after the scraper has completed |
| Build Config2 | Code | Build global config/state | Schedule Trigger1; Receive Apify Webhook2 | Check Schedule Trigger2 | # Product Hunt Lead Generator … (links incl. README/Setup/Apify/Community) |
| Check Schedule Trigger2 | IF | Route schedule vs webhook path | Build Config2 | Fetch Webhook Status2; Create Daily Log Sheet2 (true) / Fetch RUNS Data2 (false) |  |
| Fetch Webhook Status2 | Data Table | Check webhook-created flag | Check Schedule Trigger2 (true) | Check Webhook Exists2 | ## Initiate the scrape |
| Check Webhook Exists2 | IF | Decide to create webhook or skip | Fetch Webhook Status2 | Check Weekly Run (Monday)2; Create External Webhook2 | ## Initiate the scrape |
| Create External Webhook2 | HTTP Request | Create Apify webhook | Check Webhook Exists2 | Mark Webhook as Created2 | ## Initiate the scrape |
| Mark Webhook as Created2 | Data Table | Persist webhook-created flag | Create External Webhook2 | Check Weekly Run (Monday)2 | ## Initiate the scrape |
| Check Weekly Run (Monday)2 | IF | Decide weekly+daily vs daily only | Check Webhook Exists2 / Mark Webhook as Created2 | Run Actor "Weekly Top Products"2 (true); Run Actor "Daily Top Products"2 (false) | ## Initiate the scrape |
| Run Actor "Weekly Top Products"2 | HTTP Request | Start weekly Apify run | Check Weekly Run (Monday)2 (true) | Run Actor "Daily Top Products"2 | ## Initiate the scrape |
| Run Actor "Daily Top Products"2 | HTTP Request | Start daily Apify run | Check Weekly Run (Monday)2 (false) or Weekly run | Combine Run IDs2 | ## Initiate the scrape |
| Combine Run IDs2 | Code | Merge run/dataset IDs into tracking object | Run Actor "Daily Top Products"2 (+ Weekly via reference) | Store Run Data2 | ## Initiate the scrape |
| Store Run Data2 | Data Table | Store RUNS tracking payload | Combine Run IDs2 | — | ## Initiate the scrape |
| Fetch RUNS Data2 | Data Table | Load RUNS rows on webhook event | Check Schedule Trigger2 (false) | Update Run Status2 | ## Get the dataset after the scraper has completed |
| Update Run Status2 | Code | Mark specific run as succeeded | Fetch RUNS Data2 | Save Updated RUNS Data1 | ## Get the dataset after the scraper has completed |
| Save Updated RUNS Data1 | Data Table | Persist updated RUNS JSON | Update Run Status2 | Switch | ## Get the dataset after the scraper has completed |
| Switch | Switch | Route based on completion count | Save Updated RUNS Data1 | Extract Dataset IDs2 (two outputs wired) | ## Get the dataset after the scraper has completed |
| Extract Dataset IDs2 | Code | Emit datasetId items from RUNS | Switch | Fetch Dataset Items2 | ## Get the dataset after the scraper has completed |
| Fetch Dataset Items2 | HTTP Request | Pull Apify dataset items | Extract Dataset IDs2 | Deduplicate2 | ## Get the dataset after the scraper has completed |
| Deduplicate2 | Code | Remove duplicate products by name | Fetch Dataset Items2 | Loop Over Sub Items2 | ## Retreive the contact details from each website |
| Loop Over Sub Items2 | Split In Batches | Batch iteration over products | Deduplicate2; Log API Result to Google Sheet2; If1 (false); HTTP Request (error); If (false) | HTTP Request (per-item); Aggregate Batch Results2 (collect) | ## Retreive the contact details from each website |
| HTTP Request | HTTP Request | Website reachability check | Loop Over Sub Items2 | If (success); Loop Over Sub Items2 (error) | ## Retreive the contact details from each website |
| If | IF | Require statusCode 200 to continue | HTTP Request | Map Category by Topic2 (true); Loop Over Sub Items2 (false) | ## Retreive the contact details from each website |
| Map Category by Topic2 | Code | Map topics to categories | If (true) | Prepare Scraper Payload1 | ## Retreive the contact details from each website |
| Prepare Scraper Payload1 | Code | Build Apify contact scraper payload | Map Category by Topic2 | Get Contact Scraper1 | ## Retreive the contact details from each website |
| Get Contact Scraper1 | HTTP Request | Get contact details (sync) | Prepare Scraper Payload1 | If1 | ## Retreive the contact details from each website |
| If1 | IF | Continue only if contact data not empty | Get Contact Scraper1 | Prepare Request Body2 (true); Loop Over Sub Items2 (false) | ## Retreive the contact details from each website |
| Prepare Request Body2 | Code | Validate + normalize final record / skip | If1 (true) | Get Sheet Metadata2 | ## Retreive the contact details from each website |
| Get Sheet Metadata2 | Data Table | Load SHEET metadata | Prepare Request Body2 | Parse Sheet IDs2 | ## Retreive the contact details from each website |
| Parse Sheet IDs2 | Code | Parse sheet metadata JSON | Get Sheet Metadata2 | Check if Skipped2 | ## Retreive the contact details from each website |
| Check if Skipped2 | IF | Route skipped vs non-skipped | Parse Sheet IDs2 | Prepare Error Row2 (true); Format API Result for Sheet2 (false) | ## Retreive the contact details from each website |
| Prepare Error Row2 | Code | Build error log row | Check if Skipped2 (true) | Log API Result to Google Sheet2 | ## Retreive the contact details from each website |
| Format API Result for Sheet2 | Code | Build standardized record for email/log | Check if Skipped2 (false) | Check Owner Email Exists2 | ## Email potential leads |
| Check Owner Email Exists2 | IF | Decide whether to email | Format API Result for Sheet2 | Prepare CC (true); Handle No Email (false) | ## Email potential leads |
| Prepare CC | Set | Convert other_emails to cc string | Check Owner Email Exists2 (true) | Send a message | ## Email potential leads |
| Send a message | Gmail | Send outreach email | Prepare CC | Verify Email Sent Successfully2 | ## Email potential leads; ![Email Blast Template](https://articles.emp0.com/wp-content/uploads/2025/12/product-hunt-scraper-email-blast.png) |
| Verify Email Sent Successfully2 | IF | Branch on email error presence | Send a message | Handle Email Failed (true); Handle Email Success (false) | ## Email potential leads |
| Handle Email Failed | Code | Mark email failed for logging | Verify Email Sent Successfully2 | Log API Result to Google Sheet2 | ## Email potential leads |
| Handle Email Success | Code | Mark email success for logging | Verify Email Sent Successfully2 | Log API Result to Google Sheet2 | ## Email potential leads |
| Handle No Email | Code | Mark no email case for logging | Check Owner Email Exists2 (false) | Log API Result to Google Sheet2 | ## Email potential leads |
| Log API Result to Google Sheet2 | Google Sheets | Append activity row | Prepare Error Row2 / Handle No Email / Handle Email Failed / Handle Email Success | Loop Over Sub Items2 | ## Log crawling status and results to Gsheet and DataTables |
| Aggregate Batch Results2 | Aggregate | Collect batch results | Loop Over Sub Items2 | Get Daily Activities to Sheet2 | ## Log crawling status and results to Gsheet and DataTables |
| Get Daily Activities to Sheet2 | Google Sheets | Read all activity rows | Aggregate Batch Results2 | Summarize Daily Stats2 | ## Log crawling status and results to Gsheet and DataTables |
| Summarize Daily Stats2 | Code | Compute daily summary metrics | Get Daily Activities to Sheet2 | Log Daily Summary to Sheet2 | ## Log crawling status and results to Gsheet and DataTables |
| Log Daily Summary to Sheet2 | Google Sheets | Append summary rows | Summarize Daily Stats2 | Clear Old Run Data2 | ## Log crawling status and results to Gsheet and DataTables |
| Clear Old Run Data2 | Data Table | Delete old state rows | Log Daily Summary to Sheet2 | — | ## Log crawling status and results to Gsheet and DataTables |
| Create Daily Log Sheet2 | Google Sheets | Create daily spreadsheet | Check Schedule Trigger2 (true) | Extract Sheet Info2 | ## Log crawling status and results to Gsheet and DataTables |
| Extract Sheet Info2 | Code | Extract spreadsheetId/sheetIds | Create Daily Log Sheet2 | Store Sheet Metadata2 | ## Log crawling status and results to Gsheet and DataTables |
| Store Sheet Metadata2 | Data Table | Persist SHEET metadata | Extract Sheet Info2 | — | ## Log crawling status and results to Gsheet and DataTables |
| Sticky Note133 | Sticky Note | Comment | — | — |  |
| Sticky Note | Sticky Note | Comment | — | — |  |
| Sticky Note1 | Sticky Note | Comment | — | — |  |
| Sticky Note2 | Sticky Note | Comment | — | — |  |
| Sticky Note3 | Sticky Note | Comment | — | — |  |
| Sticky Note4 | Sticky Note | Comment | — | — |  |
| Sticky Note5 | Sticky Note | Comment | — | — |  |

> Sticky note rows are included for completeness; they do not connect to other nodes.

---

## 4. Reproducing the Workflow from Scratch

1) **Create Data Table**
   1. In n8n, create a **Data Table** with columns:
      - `key` (string)
      - `value` (string)
   2. Note the Data Table ID and set it in the config node later.

2) **Create the two entry points**
   1. Add **Schedule Trigger** named `Schedule Trigger1`
      - Run daily at **09:00**.
   2. Add **Webhook** named `Receive Apify Webhook2`
      - Method: POST
      - Path: `product-hunt-scraper-workflow`
      - Activate later and copy the production URL for Apify.

3) **Add global config**
   1. Add a **Code** node `Build Config2`.
   2. Implement config fields:
      - `dataTableId`
      - `weeklyDay` (“Monday”)
      - `externalScraper.baseUrl` (`https://api.apify.com/v2`)
      - `externalScraper.payload.day`, `.week`, `.webhook`, `.contacts`
      - `email` metadata and `batch`
   3. Connect **Schedule Trigger1 → Build Config2** and **Receive Apify Webhook2 → Build Config2**.

4) **Route based on trigger type**
   1. Add **IF** node `Check Schedule Trigger2`:
      - Condition: `{{$json.isScheduleTrigger}}` is true
   2. Connect **Build Config2 → Check Schedule Trigger2**.

5) **Schedule branch: ensure Apify webhook exists**
   1. Add **Data Table** node `Fetch Webhook Status2`:
      - Operation: Get
      - Filter: `key=IS_WEBHOOK_CREATED` and `value=1`
      - Limit 1
   2. Add **IF** node `Check Webhook Exists2`:
      - Condition: input object not empty
   3. Add **HTTP Request** node `Create External Webhook2`:
      - POST `{{$('Build Config2').json.externalScraper.baseUrl}}/webhooks`
      - Body: `{{$('Build Config2').json.externalScraper.payload.webhook}}`
      - Auth: **HTTP Bearer Auth** (Apify token)
   4. Add **Data Table** node `Mark Webhook as Created2`:
      - Insert `{key:IS_WEBHOOK_CREATED,value:1}`
   5. Connect:
      - `Check Schedule Trigger2 (true) → Fetch Webhook Status2 → Check Webhook Exists2`
      - `Check Webhook Exists2 (false) → Create External Webhook2 → Mark Webhook as Created2`
      - `Check Webhook Exists2 (true) → Check Weekly Run (Monday)2`
      - `Mark Webhook as Created2 → Check Weekly Run (Monday)2`

6) **Schedule branch: create daily Google Sheet and store metadata**
   1. Add **Google Sheets** node `Create Daily Log Sheet2`:
      - Resource: Spreadsheet
      - Create spreadsheet with 2 sheets: “Daily Activities”, “Daily Summary”
      - Title: `product_import_log_YYYY_MM_DD` based on schedule timestamp
      - Credential: Google Sheets OAuth2
   2. Add **Code** node `Extract Sheet Info2` to output `{sheet: JSON.stringify({spreadsheetId, dailyActivities, dailySummary})}`
   3. Add **Data Table** node `Store Sheet Metadata2` inserting `{key:SHEET,value:<sheet json>}`
   4. Connect:
      - `Check Schedule Trigger2 (true) → Create Daily Log Sheet2 → Extract Sheet Info2 → Store Sheet Metadata2`

7) **Run Apify Product Hunt actor(s)**
   1. Add **IF** `Check Weekly Run (Monday)2` checking `Build Config2.json.isSameDay`.
   2. Add **HTTP Request** `Run Actor "Weekly Top Products"2`:
      - POST `/acts/danpoletaev~product-hunt-scraper/runs`
      - Body: `externalScraper.payload.week`
      - Bearer auth: Apify token
   3. Add **HTTP Request** `Run Actor "Daily Top Products"2`:
      - POST same endpoint
      - Body: `externalScraper.payload.day`
   4. Add **Code** `Combine Run IDs2` to build `{ids:{runId:{status:false,datasetId}},succeeded:0}`
   5. Add **Data Table** `Store Run Data2` inserting `{key:RUNS,value:JSON.stringify(payload)}`
   6. Connect:
      - `Check Weekly Run (Monday)2 (true) → Weekly → Daily → Combine Run IDs2`
      - `Check Weekly Run (Monday)2 (false) → Daily → Combine Run IDs2`
      - `Combine Run IDs2 → Store Run Data2`

8) **Webhook branch: update RUNS and proceed**
   1. Add **Data Table** `Fetch RUNS Data2` (get all where key=RUNS).
   2. Add **Code** `Update Run Status2` (find runId from webhook payload, mark succeeded).
   3. Add **Data Table** `Save Updated RUNS Data1` (update by row id).
   4. Add **Switch** node (optional but included) to route based on `succeeded` count.
   5. Connect:
      - `Check Schedule Trigger2 (false) → Fetch RUNS Data2 → Update Run Status2 → Save Updated RUNS Data1 → Switch`

9) **Extract dataset IDs and fetch dataset items**
   1. Add **Code** `Extract Dataset IDs2` parsing the RUNS payload.
   2. Add **HTTP Request** `Fetch Dataset Items2`:
      - GET `/datasets/{{datasetId}}/items`
      - Bearer auth Apify
   3. Add **Code** `Deduplicate2` (unique by normalized name).
   4. Add **Split In Batches** `Loop Over Sub Items2`.
   5. Connect:
      - `Switch → Extract Dataset IDs2 → Fetch Dataset Items2 → Deduplicate2 → Loop Over Sub Items2`

10) **Per-item processing: reachability, categorization, contact enrichment**
   1. Add **HTTP Request** node `HTTP Request`:
      - URL: current item’s `websiteUrl`
      - Configure response to never throw and provide full response.
   2. Add **IF** `If` checking `statusCode == 200`.
   3. Add **Code** `Map Category by Topic2` (topic keyword mapping).
   4. Add **Code** `Prepare Scraper Payload1` (build Apify contact scraper payload).
   5. Add **HTTP Request** `Get Contact Scraper1` to Apify contact-info-scraper `run-sync-get-dataset-items` with Bearer auth.
   6. Add **IF** `If1` checking contact result not empty.
   7. Connect:
      - `Loop Over Sub Items2 (item output) → HTTP Request → If → Map Category by Topic2 → Prepare Scraper Payload1 → Get Contact Scraper1 → If1`
      - `If (false) → Loop Over Sub Items2`
      - `If1 (false) → Loop Over Sub Items2`

11) **Prepare final record, load sheet metadata, and branch on skip**
   1. Add **Code** `Prepare Request Body2` (validation + normalized structure + `_skip`).
   2. Add **Data Table** `Get Sheet Metadata2` (get key=SHEET).
   3. Add **Code** `Parse Sheet IDs2` to parse sheet object.
   4. Add **IF** `Check if Skipped2` on `_skip`.
   5. Add **Code** `Prepare Error Row2` (error row fields).
   6. Add **Code** `Format API Result for Sheet2` (standard success record structure).
   7. Connect:
      - `If1 (true) → Prepare Request Body2 → Get Sheet Metadata2 → Parse Sheet IDs2 → Check if Skipped2`
      - `Check if Skipped2 (true) → Prepare Error Row2`
      - `Check if Skipped2 (false) → Format API Result for Sheet2`

12) **Optional Gmail outreach**
   1. Add **IF** `Check Owner Email Exists2`:
      - Condition: `Prepare Request Body2.contact_info.primary_email` exists
   2. Add **Set** `Prepare CC`:
      - `cc = other_emails.join(',') || null`
   3. Add **Gmail** `Send a message`:
      - To: primary_email
      - CC: from Prepare CC
      - Subject + HTML body template
      - OAuth2 credential for Gmail
   4. Add **IF** `Verify Email Sent Successfully2`:
      - Branch if `$json.error` exists/notEmpty
   5. Add **Code** nodes:
      - `Handle No Email` → emailSent=no
      - `Handle Email Failed` → emailSent=failed, store error
      - `Handle Email Success` → emailSent=yes
   6. Connect:
      - `Format API Result for Sheet2 → Check Owner Email Exists2`
      - True: `Check Owner Email Exists2 → Prepare CC → Send a message → Verify Email Sent Successfully2 → (Failed/Success handlers)`
      - False: `Check Owner Email Exists2 → Handle No Email`

13) **Append activity row to Google Sheets and continue loop**
   1. Add **Google Sheets** `Log API Result to Google Sheet2`:
      - Operation: append
      - DocumentId: parsed spreadsheetId
      - Sheet: dailyActivities (by ID)
      - Auto-map columns
   2. Connect:
      - `Prepare Error Row2 → Log API Result to Google Sheet2`
      - `Handle No Email/Failed/Success → Log API Result to Google Sheet2`
      - `Log API Result to Google Sheet2 → Loop Over Sub Items2` (continue batching)

14) **After loop: summarize and cleanup**
   1. Add **Aggregate** `Aggregate Batch Results2` and connect from `Loop Over Sub Items2` “done” output.
   2. Add **Google Sheets** `Get Daily Activities to Sheet2` (read all rows).
   3. Add **Code** `Summarize Daily Stats2`.
   4. Add **Google Sheets** `Log Daily Summary to Sheet2` (append).
   5. Add **Data Table** `Clear Old Run Data2` to delete all rows except `IS_WEBHOOK_CREATED`.
   6. Connect: `Aggregate → Get Daily Activities → Summarize → Log Daily Summary → Clear Old Run Data`

15) **Credentials to configure**
   - **Apify**: HTTP Bearer Auth credential with your Apify token.
   - **Google Sheets OAuth2**: access to create/read/write spreadsheets.
   - **Gmail OAuth2**: permission to send emails from configured account.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Product overview, setup checklist, cost estimate, and support contacts | Included in workflow sticky note “Product Hunt Lead Generator” |
| How it works → README.md | https://github.com/Jharilela/n8n-workflows/blob/main/Product%20Hunt%20Lead%20Generator/README.md |
| Full Setup Guide → Technical Setup Guide | https://github.com/Jharilela/n8n-workflows/blob/main/Product%20Hunt%20Lead%20Generator/Technical%20Setup.md |
| Apify Product Hunt Scraper actor | https://console.apify.com/actors/ralZdliP7uuVVznKq/input |
| Apify Contact Info Scraper actor | https://apify.com/vdrmota/contact-info-scraper |
| Purchase link | https://0emp0.gumroad.com/l/product-hunt-lead-generator |
| Community | https://www.skool.com/aia-ai-automation-2762 |
| Apify affiliate link / referral code | https://www.apify.com/?fpr=99h7ds (referral code: 99h7ds) |
| Support Discord | https://discord.gg/RqMKqA3jMS |
| Support email | mailto:jay@emp0.com |
| Email template preview image | https://articles.emp0.com/wp-content/uploads/2025/12/product-hunt-scraper-email-blast.png |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.