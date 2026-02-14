Build a LinkedIn job intelligence pipeline with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/build-a-linkedin-job-intelligence-pipeline-with-apify-and-google-sheets-12838


# Build a LinkedIn job intelligence pipeline with Apify and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Build a LinkedIn job intelligence pipeline with Apify and Google Sheets  
**Workflow name (in JSON):** Linkedin Job Scraper

This workflow scrapes LinkedIn job postings via an Apify actor, standardizes and enriches the resulting job data, then stores only *new* postings into Google Sheets to build a deduplicated hiring-intelligence dataset over time.

### 1.1 Trigger & Input
Starts manually (optionally replaced by a Cron trigger later). The LinkedIn job search URL(s) and scrape settings are provided in the Apify node payload.

### 1.2 Scraping (Apify Actor Execution)
Runs the “Linkedin Jobs Scraper - PPR (curious_coder/linkedin-jobs-scraper)” actor and retrieves the resulting dataset (jobs list).

### 1.3 Data Normalization & Enrichment
Selects only relevant fields and computes “days since posted” based on `postedAt`.

### 1.4 Batch Processing + Google Sheets Persistence
Processes jobs in batches, appends rows, then checks for duplicates in the sheet and loops until all items are processed. Deduplication is based on the `Id` column matching.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & Run Scraper
**Overview:** Initiates the workflow and launches the Apify actor to scrape LinkedIn Jobs from provided URL(s).  
**Nodes involved:**  
- Manual trigger (can be replaces to cron trigger)  
- Extract linkedin data using Apify

#### Node: Manual trigger (can be replaces to cron trigger)
- **Type / role:** `Manual Trigger` — entry point for manual executions.
- **Configuration (interpreted):** No parameters; click “Execute workflow” to start.
- **Inputs / outputs:**  
  - **Output →** Extract linkedin data using Apify
- **Edge cases / failures:** None (beyond standard n8n execution startup).
- **Version notes:** TypeVersion 1.

#### Node: Extract linkedin data using Apify
- **Type / role:** `Apify` (`@apify/n8n-nodes-apify.apify`) — runs an Apify actor and fetches its dataset output.
- **Configuration (interpreted):**
  - **Operation:** “Run actor and get dataset”
  - **Actor:** `hKByXkMQaC5Qt9UMN` (Linkedin Jobs Scraper - PPR / curious_coder/linkedin-jobs-scraper)
  - **Authentication:** Apify OAuth2
  - **Custom input body:**  
    - `count: 100` (max jobs to extract; user can change)
    - `scrapeCompany: true`
    - `urls: [...]` (one LinkedIn Jobs search URL provided; supports up to ~10 per sticky note guidance)
- **Key variables / expressions:** none; static JSON body.
- **Inputs / outputs:**  
  - **Input ←** Manual trigger  
  - **Output →** Choose data field that are relevant
- **Edge cases / failures:**
  - Apify auth failure / expired OAuth token.
  - Actor run failure (rate limits, LinkedIn blocking, invalid URL format).
  - Dataset schema changes from the actor (fields renamed/removed).
  - Timeout or long-running scrape (node timeout not explicitly set here).
- **Version notes:** TypeVersion 1; requires Apify n8n community node package and valid Apify credentials.

---

### Block 2.2 — Normalize & Enrich Job Records
**Overview:** Reduces the scraped dataset to a stable schema used downstream and calculates “days since posted”.  
**Nodes involved:**  
- Choose data field that are relevant

#### Node: Choose data field that are relevant
- **Type / role:** `Set` — maps/renames fields and computes derived values.
- **Configuration (interpreted):**
  - Creates these output fields:
    - `id` = `$json.id`
    - `linkJob` = `$json.link`
    - `title` = `$json.title`
    - `companyName` = `$json.companyName`
    - `location` = `$json.location`
    - `postedAt` = `$json.postedAt`
    - `How many days that was posted?` = calculated day difference between `$now` and `postedAt`
- **Key expression (days since posted):**
  - Uses JavaScript date math:
    - `Math.floor((new Date($now) - new Date($json.postedAt)) / (1000*60*60*24))`
- **Inputs / outputs:**  
  - **Input ←** Extract linkedin data using Apify  
  - **Output →** Looping
- **Edge cases / failures:**
  - If `postedAt` is missing or not parseable by `new Date(...)`, the calculation may yield `NaN`.
  - Timezone/locale differences: “days since” may shift by 1 depending on timestamp precision.
- **Version notes:** TypeVersion 3.4.

---

### Block 2.3 — Batch Loop + Append + De-duplication in Google Sheets
**Overview:** Processes items in batches, appends to Google Sheets, then searches the sheet to detect duplicates and uses an IF condition to control looping.  
**Nodes involved:**  
- Looping  
- Append new rows to Gsheet  
- Search for existing records in Gsheet  
- Condition to remove dup in gsheet

#### Node: Looping
- **Type / role:** `Split In Batches` — throttles processing to avoid overloading Google Sheets and keeps runs stable.
- **Configuration (interpreted):**
  - **Batch size:** 5
- **Inputs / outputs (as wired):**
  - **Input1 ←** Choose data field that are relevant
  - **Output 0 (next batch) →** (not connected)
  - **Output 1 (current batch items) →** Append new rows to Gsheet
- **Important behavior note:** In typical patterns, SplitInBatches uses one output to send the current batch and then loops back to fetch the next batch. Here the looping is controlled by a connection from the IF node back into `Looping`.
- **Edge cases / failures:**
  - If loop wiring is incorrect, workflow may process only one batch or may create an unintended loop.
  - With `batchSize=5`, many executions may be needed for large datasets.
- **Version notes:** TypeVersion 3.

#### Node: Append new rows to Gsheet
- **Type / role:** `Google Sheets` — appends job rows to a sheet.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Spreadsheet:** “Linedin Job Post” (documentId provided)
  - **Sheet/tab:** “Sheet1” (`gid=0`)
  - **Mapping mode:** Define below; maps fields into columns.
  - **Matching columns configured:** `["Id"]` (note: for append this setting typically matters more for upsert/update operations; still present in config)
  - **Column mappings:**
    - `Id` = `{{$json.id}}`
    - `title` = `{{$json.title}}`
    - `linkJob` = `{{$json.linkJob}}`
    - `location` = `{{$json.location}}`
    - `postedAt` = `{{$json.postedAt}}`
    - `compnayName` = `{{$json.companyName}}` (typo in column name: “compnayName”)
    - `How many days that was posted?` = `{{$json['How many days that was posted?']}}`
- **Inputs / outputs:**  
  - **Input ←** Looping  
  - **Output →** Search for existing records in Gsheet
- **Edge cases / failures:**
  - Google auth failure / expired OAuth token.
  - Sheet permissions (no edit access).
  - Column name mismatch (especially `compnayName` typo must exist in the sheet header exactly).
  - Appending *before* checking duplicates: this workflow’s order means duplicates may already be inserted if they exist in the dataset or in the sheet.
- **Version notes:** TypeVersion 4.7; requires Google Sheets OAuth2 credentials.

#### Node: Search for existing records in Gsheet
- **Type / role:** `Google Sheets` — searches for existing records (used for dedup control).
- **Configuration (interpreted):**
  - **Error handling:** `onError: continueRegularOutput` + `alwaysOutputData: true`
    - Ensures the workflow continues even if search fails or returns nothing.
  - **Spreadsheet / sheet:** same document and tab as append.
  - **Operation:** Not explicitly visible in JSON snippet; based on the node name and downstream usage, it is intended to search/lookup rows and return a `row_number` when found.
- **Inputs / outputs:**  
  - **Input ←** Append new rows to Gsheet  
  - **Output →** Condition to remove dup in gsheet
- **Edge cases / failures:**
  - Misconfigured search criteria could always return nothing, making dedup logic ineffective.
  - If Google Sheets API returns unexpected schema, `row_number` may be absent.
  - Because errors are swallowed, silent failures can occur (harder to detect).
- **Version notes:** TypeVersion 4.7.

#### Node: Condition to remove dup in gsheet
- **Type / role:** `IF` — branching logic to decide whether to continue looping.
- **Configuration (interpreted):**
  - Condition checks whether `{{$json.row_number}}` **does not exist**.
  - In other words: if the lookup did not find an existing row, treat as “new”.
- **Inputs / outputs (as wired):**
  - **Input ←** Search for existing records in Gsheet
  - **Output (true) →** (not connected)
  - **Output (false) →** Looping
- **Important behavior note (potential logic inversion):**
  - Common dedup flow is: *if NOT found → append; if found → skip*.  
  - Here the sheet append happens **before** the search, and the IF node loops back on the **false** branch. This suggests the IF is being used more as a *loop controller* than as a strict “skip duplicates before insert” gate.
- **Edge cases / failures:**
  - If `row_number` is never present, IF may always evaluate “notExists” and stop looping (or behave unexpectedly depending on output wiring).
  - If `row_number` exists for appended rows (because the search finds the row just appended), the condition may repeatedly route to the loop, potentially causing extra iterations.
- **Version notes:** TypeVersion 2.3.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Comment: input URLs guidance |  |  | ## Paste job Linkedin URL\n\nIn this node you can add up to 10 URLs to be scraped from linkedin jobs\n\nBy default the item count set to 100 you can change it to whatever number of jobs you want to extract |
| Sticky Note1 | Sticky Note | Comment: dedup branch explanation |  |  | ## De-Duplication\nThis branch to ensure no duplicates will go over the same rows |
| Sticky Note2 | Sticky Note | Comment: video link |  |  | ## [Video Tutorial](https://youtu.be/tXvcCoM5igQ)\n@[youtube](tXvcCoM5igQ) |
| Sticky Note3 | Sticky Note | Comment: full workflow description & setup |  |  | # LinkedIn job intelligence workflow\n\n### This workflow automatically collects LinkedIn job postings and turns them into structured hiring intelligence...\n\n(Contains setup steps and description; see full sticky note in workflow.) |
| Manual trigger (can be replaces to cron trigger) | Manual Trigger | Entry point | — | Extract linkedin data using Apify |  |
| Extract linkedin data using Apify | Apify | Scrape LinkedIn jobs via actor + fetch dataset | Manual trigger (can be replaces to cron trigger) | Choose data field that are relevant | ## Paste job Linkedin URL\n\nIn this node you can add up to 10 URLs... |
| Choose data field that are relevant | Set | Normalize fields + compute “days since posted” | Extract linkedin data using Apify | Looping |  |
| Looping | Split In Batches | Batch processing + loop control | Choose data field that are relevant; Condition to remove dup in gsheet (false branch) | Append new rows to Gsheet |  |
| Append new rows to Gsheet | Google Sheets | Append rows to sheet | Looping | Search for existing records in Gsheet |  |
| Search for existing records in Gsheet | Google Sheets | Lookup existing rows / dedup support | Append new rows to Gsheet | Condition to remove dup in gsheet | ## De-Duplication\nThis branch to ensure no duplicates will go over the same rows |
| Condition to remove dup in gsheet | IF | Decide whether to continue looping based on `row_number` existence | Search for existing records in Gsheet | Looping (false branch) | ## De-Duplication\nThis branch to ensure no duplicates will go over the same rows |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“Linkedin Job Scraper”** (or your preferred name).

2. **Add trigger**
   1. Add node: **Manual Trigger**  
      - No configuration required.  
      - (Optional) Later replace with **Cron** to run on a schedule.

3. **Add Apify scraping node**
   1. Add node: **Apify** (community node package `@apify/n8n-nodes-apify`)  
   2. Configure:
      - **Authentication:** Apify OAuth2 (connect your Apify account in Credentials)
      - **Actor source:** Store
      - **Actor:** *Linkedin Jobs Scraper - PPR (curious_coder/linkedin-jobs-scraper)*  
      - **Operation:** *Run actor and get dataset*
      - **Input body** (example; edit URLs/count):
        - `count`: `100`
        - `scrapeCompany`: `true`
        - `urls`: add one or more LinkedIn job search URLs (up to 10 as per sticky note guidance)
   3. Connect: **Manual Trigger → Apify**

4. **Add data normalization/enrichment**
   1. Add node: **Set**
   2. Add assignments (fields):
      - `id` = `{{$json.id}}`
      - `linkJob` = `{{$json.link}}`
      - `title` = `{{$json.title}}`
      - `companyName` = `{{$json.companyName}}`
      - `location` = `{{$json.location}}`
      - `postedAt` = `{{$json.postedAt}}`
      - `How many days that was posted?` = expression:
        - `{{ Math.floor((new Date($now).getTime() - new Date($json.postedAt).getTime()) / (1000*60*60*24)) }}`
   3. Connect: **Apify → Set**

5. **Add batch processing**
   1. Add node: **Split In Batches**
   2. Configure:
      - **Batch size:** `5`
   3. Connect: **Set → Split In Batches**

6. **Prepare Google Sheet**
   1. Create a Google Sheet (example name: “Linedin Job Post”).
   2. In the first row (headers), add columns matching your mapping exactly, including the typo if you keep it:
      - `Id`, `title`, `linkJob`, `location`, `postedAt`, `compnayName`, `How many days that was posted?`
   3. Ensure your n8n Google account has edit access.

7. **Add Google Sheets append**
   1. Add node: **Google Sheets**
   2. Connect Google Sheets OAuth2 credentials.
   3. Configure:
      - **Operation:** Append
      - **Document:** select your spreadsheet
      - **Sheet:** select the target tab (e.g., Sheet1)
      - **Mapping:** “Define below” and map:
        - `Id` → `{{$json.id}}`
        - `title` → `{{$json.title}}`
        - `linkJob` → `{{$json.linkJob}}`
        - `location` → `{{$json.location}}`
        - `postedAt` → `{{$json.postedAt}}`
        - `compnayName` → `{{$json.companyName}}`
        - `How many days that was posted?` → `{{$json['How many days that was posted?']}}`
   4. Connect: **Split In Batches (items output) → Google Sheets (append)**

8. **Add Google Sheets lookup/search**
   1. Add another node: **Google Sheets**
   2. Configure:
      - Select the same **Document** and **Sheet**
      - Configure it to **search/lookup** by `Id` (so it can return a `row_number` when a row exists)
      - Enable:
        - **Always Output Data**
        - **On Error:** Continue (so the IF node still receives data)
   3. Connect: **Append → Search**

9. **Add IF condition**
   1. Add node: **IF**
   2. Condition:
      - Check `{{$json.row_number}}` **not exists**
   3. Connect: **Search → IF**

10. **Close the loop (loop controller)**
   1. Connect **IF (false output)** → **Split In Batches**
      - This matches the JSON wiring and is intended to drive continued batch processing.

11. **(Optional but recommended) Improve dedup order**
   - To prevent inserting duplicates, a more robust rebuild would place the **Search/IF before Append**, and append only on the “not found” branch.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video resource included in workflow notes | https://youtu.be/tXvcCoM5igQ |
| Sticky note explains the workflow purpose, steps, and suggests replacing Manual Trigger with Cron | In-workflow “LinkedIn job intelligence workflow” sticky note |
| Input guidance: up to 10 LinkedIn job URLs; default count=100 but can be changed | In-workflow “Paste job Linkedin URL” sticky note |