Sync Toggl Track time entries with Google Sheets detail and summary tabs

https://n8nworkflows.xyz/workflows/sync-toggl-track-time-entries-with-google-sheets-detail-and-summary-tabs-13497


# Sync Toggl Track time entries with Google Sheets detail and summary tabs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
Synchronize **Toggl Track time entries** into **two Google Sheets files** organized by **monthly tabs (YYYY-MM)**:
- **Detail spreadsheet:** one row per time entry
- **Summary spreadsheet:** one row per day (daily total duration + merged descriptions/tags)

**Target use cases:**
- Monthly reporting by project
- Timesheet auditing (raw detail + daily totals)
- Maintaining clean monthly tabs (auto-create, clear, optionally delete non-month tabs)

### 1.1 Trigger & Date Range
Creates a `start_date` and `end_date` used to query Toggl.

### 1.2 Toggl Data Fetch
Fetches time entries and project metadata from Toggl Track API using HTTP Basic Auth.

### 1.3 Merge & Transform
Enriches time entries with project info, filters to one project name, generates:
- normalized detail rows
- daily summaries
- list of months present
Stores those datasets in **workflow static data**.

### 1.4 Detail Sheet Preparation (tabs management)
Reads existing tabs in the **detail spreadsheet**, then:
- creates missing month tabs
- clears month tabs content
- deletes non-month tabs (when at least one month exists)

### 1.5 Detail Sheet Write Loop
Groups detail rows by month, expands to rows, appends to the corresponding month tab.

### 1.6 Summary Sheet Preparation (tabs management)
Same tab lifecycle management as detail, but for the **summary spreadsheet**.

### 1.7 Summary Sheet Write Loop
Groups summary rows by month, expands to rows, appends to the corresponding month tab.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Date Range
**Overview:** Starts the workflow manually and defines the time window used to query Toggl.  
**Nodes involved:** `Start`, `Set Date Range`

#### Node: Start
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) to run on-demand.
- **Config:** No parameters.
- **Outputs:** One empty item to `Set Date Range`.
- **Edge cases:** None (manual execution only).
- **Version notes:** TypeVersion 1.

#### Node: Set Date Range
- **Type / role:** Set (`n8n-nodes-base.set`) defines `start_date`, `end_date`.
- **Config choices:**
  - `start_date`: hard-coded string `2025-01-01` (meant to be edited)
  - `end_date`: dynamic: `{{ $now.toFormat('yyyy-MM-dd') }}`
- **Key variables/expressions:** `$now.toFormat('yyyy-MM-dd')`
- **Outputs:** Sends to both Toggl HTTP nodes.
- **Edge cases / failures:**
  - Invalid `start_date` format causes Toggl API errors or empty results.
  - If `start_date` > `end_date`, likely returns no entries.
- **Version notes:** TypeVersion 3.4.

---

### Block 2 — Toggl Data Fetch
**Overview:** Fetches time entries and projects from Toggl Track API.  
**Nodes involved:** `Fetch Time Entries`, `Fetch Projects`

#### Node: Fetch Time Entries
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) to Toggl v9 API.
- **Config choices:**
  - URL: `https://api.track.toggl.com/api/v9/me/time_entries`
  - Query params:
    - `start_date={{ $json.start_date }}`
    - `end_date={{ $json.end_date }}`
  - Response format: JSON
  - Auth: **HTTP Basic Auth** via generic credential type
  - Retries: `maxTries=3`, `waitBetweenTries=1000ms`, `retryOnFail=true`
- **Inputs:** From `Set Date Range`
- **Outputs:** To `Merge With Projects` (input 1 / index 0)
- **Edge cases / failures:**
  - 401/403 if Toggl token is wrong or Basic Auth not set correctly.
  - 429 rate limiting (retries may not be sufficient).
  - Large date ranges can return many entries; may hit memory/execution limits.
- **Version notes:** TypeVersion 4.3.

#### Node: Fetch Projects
- **Type / role:** HTTP Request to fetch project list for enrichment.
- **Config choices:**
  - URL: `https://api.track.toggl.com/api/v9/me/projects`
  - Response: JSON
  - Auth: HTTP Basic Auth
  - Retries: same as above
- **Inputs:** From `Set Date Range`
- **Outputs:** To `Merge With Projects` (input 2 / index 1)
- **Edge cases / failures:** Same auth/rate-limit considerations.
- **Version notes:** TypeVersion 4.3.

---

### Block 3 — Merge & Transform
**Overview:** Joins time entries with their project metadata and transforms them into detail rows, daily summaries, and a list of months. Uses workflow static data as a handoff mechanism to later blocks.  
**Nodes involved:** `Merge With Projects`, `Process Data`

#### Node: Merge With Projects
- **Type / role:** Merge (`n8n-nodes-base.merge`) enriches time entries with project fields.
- **Config choices:**
  - Mode: `combine` with **advanced join**
  - Join mode: `enrichInput1` (time entries are primary)
  - Merge by fields:
    - time entry `project_id` == project `id`
  - Clash handling: shallow merge, prefer input1 on conflicts
  - Fuzzy compare: enabled
- **Inputs:**
  - Input 1: `Fetch Time Entries`
  - Input 2: `Fetch Projects`
- **Outputs:** To `Process Data`
- **Edge cases / failures:**
  - If Toggl entry lacks `project_id` or project list doesn’t include the ID, enrichment may fail (resulting item may miss `name`).
  - If Toggl returns arrays (likely), n8n merge behavior can be tricky depending on item structure; ensure both streams produce compatible item lists.
- **Version notes:** TypeVersion 3.2.

#### Node: Process Data
- **Type / role:** Code (`n8n-nodes-base.code`) transforms and aggregates.
- **Configuration choices (interpreted):**
  - User must set:
    - `PROJECT_NAME = 'YOUR_PROJECT_NAME'`
    - `TIMEZONE = 'Europe/Warsaw'`
  - Filters:
    - only items where `d.name === PROJECT_NAME` (project name match)
    - skips entries missing `start/stop` or with `duration < 0` (running timers)
  - Produces **detail entry object** fields:
    - `description, project, start_date, start_time, start_decimal_hour, stop_date, stop_time, stop_decimal_hour, duration_hhmm, duration_hours, tags, month`
  - Produces **daily summary** grouped by `start_date`:
    - `date, project, description (unique joined), duration_hhmm, duration_hours, tags (unique joined), month`
  - Tracks months encountered from entries (`yyyy-MM`)
  - Stores datasets into **global static data**:
    - `staticData.entries = JSON.stringify(allEntries)`
    - `staticData.summaries = JSON.stringify(dailySummaries)`
    - `staticData.months = JSON.stringify(months)`
  - Returns `{ ready: true, months }` for downstream steps.
- **Key expressions/variables:**
  - Uses `DateTime.fromISO(...).setZone(TIMEZONE)` (Luxon `DateTime`)
  - Uses `$getWorkflowStaticData('global')`
- **Inputs:** From `Merge With Projects`
- **Outputs:** To `Get Sheet Info 1`
- **Edge cases / failures:**
  - If Luxon `DateTime` is not available in your n8n Code node runtime, the script will fail (some n8n versions require importing; others provide it).
  - If project enrichment didn’t provide `d.name`, filtering may drop everything.
  - Static data persists across executions; this workflow deletes parts later, but partial failures can leave stale data.
  - Timezone choice affects month/day grouping near midnight.
- **Version notes:** TypeVersion 2.

---

### Block 4 — Detail Sheet Preparation (tabs management)
**Overview:** Ensures detail spreadsheet has one tab per month found in data, clears month tab contents, and deletes tabs not in the current months list.  
**Nodes involved:** `Get Sheet Info 1`, `Build Sheet Ops 1`, `Apply Sheet Ops 1`

#### Node: Get Sheet Info 1
- **Type / role:** HTTP Request to Google Sheets API to list sheet properties.
- **Config choices:**
  - URL: `https://sheets.googleapis.com/v4/spreadsheets/YOUR_DETAIL_SPREADSHEET_ID`
  - Query: `fields=sheets.properties` (reduces payload)
  - Auth: Google Sheets OAuth2 (`googleSheetsOAuth2Api`)
- **Inputs:** From `Process Data`
- **Outputs:** To `Build Sheet Ops 1`
- **Edge cases / failures:**
  - 401 if OAuth is not authorized.
  - 404 if spreadsheet ID is wrong or access not granted.
- **Version notes:** TypeVersion 4.3.

#### Node: Build Sheet Ops 1
- **Type / role:** Code node building `batchUpdate` requests for the detail spreadsheet.
- **Configuration choices (interpreted):**
  - Reads `months` from global static data.
  - Builds request list:
    1. `addSheet` for months not present
    2. `updateCells` to clear **all userEnteredValue** for sheets whose title is in `months`
    3. If `months.length > 0`: `deleteSheet` for any existing sheet whose title is not a month in the list
- **Inputs:** From `Get Sheet Info 1`
- **Outputs:** To `Apply Sheet Ops 1` with `{ requests: [...] }`
- **Edge cases / failures:**
  - **Risk of deleting important tabs** (e.g., “Instructions”, “Config”) because deletion removes any sheet not named exactly like a month, whenever at least one month exists.
  - `updateCells` clears values but not formatting; also clears the whole sheet range (no dimension limit specified).
- **Version notes:** TypeVersion 2.

#### Node: Apply Sheet Ops 1
- **Type / role:** HTTP Request calling Google Sheets `:batchUpdate`.
- **Config choices:**
  - POST `https://sheets.googleapis.com/v4/spreadsheets/YOUR_DETAIL_SPREADSHEET_ID:batchUpdate`
  - Body: `{"requests": $json.requests}` (stringified via expression)
  - Auth: Google Sheets OAuth2
- **Inputs:** From `Build Sheet Ops 1`
- **Outputs:** To `Get Sheet Info 2` (begins summary sheet prep)
- **Edge cases / failures:**
  - Invalid request payload (malformed requests) → 400.
  - Attempt to delete last remaining sheet can error in Google Sheets.
- **Version notes:** TypeVersion 4.3.

---

### Block 5 — Summary Sheet Preparation (tabs management)
**Overview:** Same lifecycle operations as detail sheet, but for summary spreadsheet.  
**Nodes involved:** `Get Sheet Info 2`, `Build Sheet Ops 2`, `Apply Sheet Ops 2`

#### Node: Get Sheet Info 2
- **Type / role:** HTTP Request to list summary spreadsheet tabs.
- **Config choices:**
  - URL: `https://sheets.googleapis.com/v4/spreadsheets/YOUR_SUMMARY_SPREADSHEET_ID`
  - Query: `fields=sheets.properties`
  - Auth: Google Sheets OAuth2
- **Inputs:** From `Apply Sheet Ops 1`
- **Outputs:** To `Build Sheet Ops 2`
- **Edge cases:** same as detail sheet.
- **Version notes:** TypeVersion 4.3.

#### Node: Build Sheet Ops 2
- **Type / role:** Code node building batchUpdate requests for the summary spreadsheet.
- **Logic:** Identical to `Build Sheet Ops 1` (add missing month tabs, clear month tabs, delete non-month tabs).
- **Inputs:** From `Get Sheet Info 2`
- **Outputs:** To `Apply Sheet Ops 2`
- **Edge cases:** same deletion risk.
- **Version notes:** TypeVersion 2.

#### Node: Apply Sheet Ops 2
- **Type / role:** HTTP Request to summary spreadsheet `:batchUpdate`.
- **Config choices:** POST with `{"requests": ...}` using Sheets OAuth2.
- **Inputs:** From `Build Sheet Ops 2`
- **Outputs:** To `Read Detail Data`
- **Edge cases:** same as detail.
- **Version notes:** TypeVersion 4.3.

---

### Block 6 — Detail Sheet Write Loop
**Overview:** Reads prepared detail dataset from static data, groups by month, loops month-by-month, expands rows, appends to the corresponding month tab.  
**Nodes involved:** `Read Detail Data`, `Detail Write Loop`, `Expand Entries`, `Write Details`

#### Node: Read Detail Data
- **Type / role:** Code node to retrieve and group detail rows.
- **Config choices:**
  - Reads `sd.entries` from global static data and then `delete sd.entries` to reduce stale persistence.
  - Groups by `item.month` and outputs one item per month: `{ month, entries: [...] }`
  - If no entries: returns `{ info: 'No data', entries: [] }`
- **Inputs:** From `Apply Sheet Ops 2`
- **Outputs:** To `Detail Write Loop`
- **Edge cases / failures:**
  - If static data missing (prior node failed), parse error or empty results.
  - The “No data” output still goes to loop node; behavior depends on SplitInBatches and the structure.
- **Version notes:** TypeVersion 2.

#### Node: Detail Write Loop
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) used as an iteration controller.
- **Config choices:** Default batch options (not explicitly set).
- **Connections (important):**
  - Output 0 → `Read Summary Data` (signals completion of detail loop stream)
  - Output 1 → `Expand Entries` (items for current batch)
  - `Write Details` feeds back into this node (continue loop)
- **Inputs:** From `Read Detail Data` and loop-back from `Write Details`
- **Edge cases / failures:**
  - If batching is misconfigured or receives unexpected shape, it may loop incorrectly or stop early.
  - If there are many months, batch size defaults matter (n8n defaults can vary by version).
- **Version notes:** TypeVersion 3.

#### Node: Expand Entries
- **Type / role:** Code node expanding `{month, entries:[...]}` into individual entry items.
- **Config choices:** `return b.entries.map(e=>({json:e}));`
- **Inputs:** From `Detail Write Loop` (output 1)
- **Outputs:** To `Write Details`
- **Edge cases / failures:**
  - If `entries` missing or not an array, runtime error.
- **Version notes:** TypeVersion 2.

#### Node: Write Details
- **Type / role:** Google Sheets node appending rows to detail spreadsheet.
- **Config choices:**
  - Operation: `append`
  - Document: URL containing `YOUR_DETAIL_SPREADSHEET_ID`
  - Sheet name: dynamic `={{ $json.month }}`
  - Column mapping: `autoMapInputData`
- **Inputs:** From `Expand Entries`
- **Outputs:** Back to `Detail Write Loop` (loop continuation)
- **Edge cases / failures:**
  - If the month tab does not exist (prep failed), append fails.
  - Header row is not created by this workflow; if the sheet is empty, rows still append but without headers.
  - Google API quota / rate limits on large datasets.
- **Version notes:** TypeVersion 4.7.

---

### Block 7 — Summary Sheet Write Loop
**Overview:** Reads summary dataset from static data, groups by month, loops through months, expands rows, appends to summary spreadsheet month tabs.  
**Nodes involved:** `Read Summary Data`, `Summary Write Loop`, `Expand Summaries`, `Write Summaries`

#### Node: Read Summary Data
- **Type / role:** Code node to retrieve and group daily summaries.
- **Config choices:**
  - Reads `sd.summaries`, then deletes:
    - `sd.summaries`
    - `sd.months`
  - Groups by `month` and outputs `{ month, entries:[...] }` per month
  - If none: `{ info: 'No summaries', entries: [] }`
- **Inputs:** From `Detail Write Loop` (output 0, after details complete)
- **Outputs:** To `Summary Write Loop`
- **Edge cases / failures:**
  - If detail loop never reaches completion output, summaries won’t run.
  - Same static-data consistency considerations.
- **Version notes:** TypeVersion 2.

#### Node: Summary Write Loop
- **Type / role:** Split In Batches loop controller.
- **Config choices:** Default options.
- **Connections:**
  - Output 1 → `Expand Summaries`
  - Output 0 → (nothing; end)
  - `Write Summaries` loops back into this node
- **Inputs:** From `Read Summary Data` and loop-back
- **Edge cases:** same as detail loop.
- **Version notes:** TypeVersion 3.

#### Node: Expand Summaries
- **Type / role:** Code node expanding grouped entries to one item per summary row.
- **Config:** `return b.entries.map(e=>({json:e}));`
- **Inputs:** From `Summary Write Loop` (output 1)
- **Outputs:** To `Write Summaries`
- **Edge cases:** if `entries` missing/not array.
- **Version notes:** TypeVersion 2.

#### Node: Write Summaries
- **Type / role:** Google Sheets node appending rows to summary spreadsheet.
- **Config choices:**
  - Operation: `append`
  - Document: URL containing `YOUR_SUMMARY_SPREADSHEET_ID`
  - Sheet name: `={{ $json.month }}`
  - Column mapping: auto-map; schema lists expected fields:
    `date, project, description, duration_hhmm, duration_hours, tags, month`
- **Inputs:** From `Expand Summaries`
- **Outputs:** Back to `Summary Write Loop`
- **Edge cases / failures:**
  - Month tab missing → append fails.
  - Data types are not forced (`attemptToConvertTypes=false`), so durations remain strings/numbers as produced.
- **Version notes:** TypeVersion 4.7.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start | Manual Trigger | Manual entry point | — | Set Date Range | ## Trigger & Date Range\nStarts the workflow and builds dynamic `start_date` / `end_date` values for API queries. |
| Set Date Range | Set | Define start/end dates for Toggl query | Start | Fetch Time Entries; Fetch Projects | ## Trigger & Date Range\nStarts the workflow and builds dynamic `start_date` / `end_date` values for API queries. |
| Fetch Time Entries | HTTP Request | Pull time entries from Toggl | Set Date Range | Merge With Projects | ## Toggl Data Fetch\nRequests time entries and projects from Toggl Track API using HTTP Basic Auth credentials. |
| Fetch Projects | HTTP Request | Pull project list from Toggl | Set Date Range | Merge With Projects | ## Toggl Data Fetch\nRequests time entries and projects from Toggl Track API using HTTP Basic Auth credentials. |
| Merge With Projects | Merge | Enrich entries with project metadata | Fetch Time Entries; Fetch Projects | Process Data | ## Merge & Transform\nMerges entries with project metadata, filters by project name, and prepares detail + summary datasets grouped by month. |
| Process Data | Code | Filter/transform; compute monthly detail + daily summary; store in static data | Merge With Projects | Get Sheet Info 1 | ## Merge & Transform\nMerges entries with project metadata, filters by project name, and prepares detail + summary datasets grouped by month. |
| Get Sheet Info 1 | HTTP Request | Read detail spreadsheet tabs | Process Data | Build Sheet Ops 1 | ## Detail Sheet Preparation\nEnsures monthly tabs exist in the detail spreadsheet and clears previous monthly tab values before writing. |
| Build Sheet Ops 1 | Code | Build batchUpdate requests for detail spreadsheet | Get Sheet Info 1 | Apply Sheet Ops 1 | ## Detail Sheet Preparation\nEnsures monthly tabs exist in the detail spreadsheet and clears previous monthly tab values before writing. |
| Apply Sheet Ops 1 | HTTP Request | Apply tab create/clear/delete for detail spreadsheet | Build Sheet Ops 1 | Get Sheet Info 2 | ## Detail Sheet Preparation\nEnsures monthly tabs exist in the detail spreadsheet and clears previous monthly tab values before writing. |
| Get Sheet Info 2 | HTTP Request | Read summary spreadsheet tabs | Apply Sheet Ops 1 | Build Sheet Ops 2 | ## Summary Sheet Preparation\nEnsures monthly tabs exist in the summary spreadsheet and clears previous monthly tab values before writing. |
| Build Sheet Ops 2 | Code | Build batchUpdate requests for summary spreadsheet | Get Sheet Info 2 | Apply Sheet Ops 2 | ## Summary Sheet Preparation\nEnsures monthly tabs exist in the summary spreadsheet and clears previous monthly tab values before writing. |
| Apply Sheet Ops 2 | HTTP Request | Apply tab create/clear/delete for summary spreadsheet | Build Sheet Ops 2 | Read Detail Data | ## Summary Sheet Preparation\nEnsures monthly tabs exist in the summary spreadsheet and clears previous monthly tab values before writing. |
| Read Detail Data | Code | Load detail rows from static data and group by month | Apply Sheet Ops 2 | Detail Write Loop | ## Detail Sheet Write Loop\nSplits detail data by month, expands entries, and appends rows to each monthly tab in the detail spreadsheet. |
| Detail Write Loop | Split In Batches | Iterate month groups for detail writing | Read Detail Data; Write Details | Expand Entries (loop output); Read Summary Data (completion output) | ## Detail Sheet Write Loop\nSplits detail data by month, expands entries, and appends rows to each monthly tab in the detail spreadsheet. |
| Expand Entries | Code | Expand grouped month entries into row items | Detail Write Loop | Write Details | ## Detail Sheet Write Loop\nSplits detail data by month, expands entries, and appends rows to each monthly tab in the detail spreadsheet. |
| Write Details | Google Sheets | Append detail rows into month tab | Expand Entries | Detail Write Loop | ## Detail Sheet Write Loop\nSplits detail data by month, expands entries, and appends rows to each monthly tab in the detail spreadsheet. |
| Read Summary Data | Code | Load daily summaries from static data and group by month | Detail Write Loop | Summary Write Loop | ## Summary Sheet Write Loop\nSplits summary data by month, expands entries, and appends rows to each monthly tab in the summary spreadsheet. |
| Summary Write Loop | Split In Batches | Iterate month groups for summary writing | Read Summary Data; Write Summaries | Expand Summaries (loop output) | ## Summary Sheet Write Loop\nSplits summary data by month, expands entries, and appends rows to each monthly tab in the summary spreadsheet. |
| Expand Summaries | Code | Expand grouped month summaries into row items | Summary Write Loop | Write Summaries | ## Summary Sheet Write Loop\nSplits summary data by month, expands entries, and appends rows to each monthly tab in the summary spreadsheet. |
| Write Summaries | Google Sheets | Append summary rows into month tab | Expand Summaries | Summary Write Loop | ## Summary Sheet Write Loop\nSplits summary data by month, expands entries, and appends rows to each monthly tab in the summary spreadsheet. |
| Instructions | Sticky Note | Embedded setup/instructions | — | — | ## Sync Toggl Track time entries with Google Sheets monthly tabs\n\nThis workflow fetches Toggl time entries, filters by project, and writes:\n- **Detail Sheet**: individual entries\n- **Summary Sheet**: daily totals\n\nData is organized into monthly tabs (YYYY-MM).\n\n### Setup after import\n1. Set your Toggl HTTP Basic Auth credential.\n2. Set your Google Sheets OAuth2 credential.\n3. Set `start_date` in **Set Date Range**.\n4. In **Process Data**, set `PROJECT_NAME` and `TIMEZONE`.\n5. Replace placeholders:\n   - `YOUR_DETAIL_SPREADSHEET_ID`\n   - `YOUR_SUMMARY_SPREADSHEET_ID`\n\n### Detail columns\n`description | project | start_date | start_time | start_decimal_hour | stop_date | stop_time | stop_decimal_hour | duration_hhmm | duration_hours | tags | month`\n\n### Summary columns\n`date | project | description | duration_hhmm | duration_hours | tags | month` |
| Section: Trigger Date | Sticky Note | Visual section label | — | — | ## Trigger & Date Range\nStarts the workflow and builds dynamic `start_date` / `end_date` values for API queries. |
| Section: Toggl Fetch | Sticky Note | Visual section label | — | — | ## Toggl Data Fetch\nRequests time entries and projects from Toggl Track API using HTTP Basic Auth credentials. |
| Section: Merge Transform | Sticky Note | Visual section label | — | — | ## Merge & Transform\nMerges entries with project metadata, filters by project name, and prepares detail + summary datasets grouped by month. |
| Section: Detail Prep | Sticky Note | Visual section label | — | — | ## Detail Sheet Preparation\nEnsures monthly tabs exist in the detail spreadsheet and clears previous monthly tab values before writing. |
| Section: Detail Loop | Sticky Note | Visual section label | — | — | ## Detail Sheet Write Loop\nSplits detail data by month, expands entries, and appends rows to each monthly tab in the detail spreadsheet. |
| Section: Summary Prep | Sticky Note | Visual section label | — | — | ## Summary Sheet Preparation\nEnsures monthly tabs exist in the summary spreadsheet and clears previous monthly tab values before writing. |
| Section: Summary Loop | Sticky Note | Visual section label | — | — | ## Summary Sheet Write Loop\nSplits summary data by month, expands entries, and appends rows to each monthly tab in the summary spreadsheet. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow**
   - Name: “Sync Toggl Track time entries with Google Sheets monthly tabs” (or your title).

2. **Add Trigger**
   - Add **Manual Trigger** node named **Start**.

3. **Add Set node for date range**
   - Add **Set** node named **Set Date Range**
   - Add fields:
     - `start_date` (String): `2025-01-01` (edit as needed)
     - `end_date` (String): `{{ $now.toFormat('yyyy-MM-dd') }}`
   - Connect: **Start → Set Date Range**

4. **Add Toggl API calls**
   - Add **HTTP Request** node **Fetch Time Entries**
     - URL: `https://api.track.toggl.com/api/v9/me/time_entries`
     - Authentication: **HTTP Basic Auth** (generic credential)
     - Query parameters:
       - `start_date = {{ $json.start_date }}`
       - `end_date = {{ $json.end_date }}`
     - Response: JSON, enable retries (3 tries, 1000ms)
   - Add **HTTP Request** node **Fetch Projects**
     - URL: `https://api.track.toggl.com/api/v9/me/projects`
     - Authentication: same HTTP Basic Auth
     - Response: JSON, retries enabled
   - Connect: **Set Date Range → Fetch Time Entries** and **Set Date Range → Fetch Projects**

5. **Create Toggl credential (HTTP Basic Auth)**
   - In n8n Credentials, create **HTTP Basic Auth** for Toggl.
   - Typical Toggl pattern: username = API token, password = `api_token` (confirm for your Toggl plan/API usage).
   - Select this credential in both HTTP nodes.

6. **Merge time entries with project metadata**
   - Add **Merge** node named **Merge With Projects**
     - Mode: advanced combine
     - Join: `enrichInput1`
     - Match fields: `project_id` (entries) with `id` (projects)
     - Prefer input1 on clashes
   - Connect:
     - **Fetch Time Entries → Merge With Projects (Input 1)**
     - **Fetch Projects → Merge With Projects (Input 2)**

7. **Transform and aggregate**
   - Add **Code** node named **Process Data**
   - Paste logic equivalent to the workflow:
     - Configure constants: `PROJECT_NAME`, `TIMEZONE`
     - Filter to that project, skip running timers (`duration < 0`)
     - Produce:
       - detail rows per entry
       - daily summaries (grouped by day)
       - `months` list
     - Store in `global` static data: `entries`, `summaries`, `months`
     - Return `{ ready: true, months }`
   - Connect: **Merge With Projects → Process Data**
   - Important: ensure Luxon `DateTime` is available in your Code node runtime; if not, adapt by parsing dates with native JS or import Luxon if supported in your environment.

8. **Detail spreadsheet tab preparation (Google Sheets API via HTTP)**
   - Add **HTTP Request** node **Get Sheet Info 1**
     - URL: `https://sheets.googleapis.com/v4/spreadsheets/<DETAIL_SPREADSHEET_ID>`
     - Query: `fields = sheets.properties`
     - Auth: Google Sheets OAuth2 (predefined credential type)
   - Add **Code** node **Build Sheet Ops 1**
     - Read months from static data
     - Build `requests[]`:
       - add missing month sheets
       - clear month sheets (`updateCells` with `fields:'userEnteredValue'`)
       - delete non-month sheets if months exist
   - Add **HTTP Request** node **Apply Sheet Ops 1**
     - URL: `https://sheets.googleapis.com/v4/spreadsheets/<DETAIL_SPREADSHEET_ID>:batchUpdate`
     - Method: POST
     - Body (JSON): `{"requests": {{$json.requests}}}` (stringify if required by node setting)
     - Auth: Google Sheets OAuth2
   - Connect: **Process Data → Get Sheet Info 1 → Build Sheet Ops 1 → Apply Sheet Ops 1**

9. **Summary spreadsheet tab preparation**
   - Add **HTTP Request** node **Get Sheet Info 2**
     - URL: `https://sheets.googleapis.com/v4/spreadsheets/<SUMMARY_SPREADSHEET_ID>`
     - Query: `fields=sheets.properties`
     - Auth: Google Sheets OAuth2
   - Add **Code** node **Build Sheet Ops 2** (same logic as ops 1)
   - Add **HTTP Request** node **Apply Sheet Ops 2**
     - URL: `https://sheets.googleapis.com/v4/spreadsheets/<SUMMARY_SPREADSHEET_ID>:batchUpdate`
     - POST with `{"requests": ...}`
   - Connect: **Apply Sheet Ops 1 → Get Sheet Info 2 → Build Sheet Ops 2 → Apply Sheet Ops 2**

10. **Create Google Sheets OAuth2 credential**
   - Create/select **Google Sheets OAuth2 API** credential.
   - Ensure scopes include Google Sheets access (typical: `https://www.googleapis.com/auth/spreadsheets`).
   - Use this credential in:
     - Get Sheet Info 1/2
     - Apply Sheet Ops 1/2
     - Write Details / Write Summaries

11. **Detail write path**
   - Add **Code** node **Read Detail Data**
     - Read `sd.entries`, delete it, group by `month`, output items `{month, entries:[...]}`.
   - Add **Split In Batches** node **Detail Write Loop**
     - Keep defaults (or set batch size to 1 to iterate month-by-month predictably).
   - Add **Code** node **Expand Entries**
     - Convert `{month, entries}` into multiple items (one per entry).
   - Add **Google Sheets** node **Write Details**
     - Operation: Append
     - Document: detail spreadsheet
     - Sheet name: `={{ $json.month }}`
     - Mapping: auto-map input
   - Connect:
     - **Apply Sheet Ops 2 → Read Detail Data → Detail Write Loop**
     - **Detail Write Loop (loop output) → Expand Entries → Write Details**
     - **Write Details → Detail Write Loop** (loop-back)

12. **Summary write path**
   - Add **Code** node **Read Summary Data**
     - Read `sd.summaries`, delete it and `sd.months`, group by `month`, output `{month, entries}`.
   - Add **Split In Batches** node **Summary Write Loop**
   - Add **Code** node **Expand Summaries**
   - Add **Google Sheets** node **Write Summaries**
     - Operation: Append
     - Document: summary spreadsheet
     - Sheet name: `={{ $json.month }}`
     - Mapping: auto-map (optionally define schema columns)
   - Connect:
     - **Detail Write Loop (done/completion output) → Read Summary Data → Summary Write Loop**
     - **Summary Write Loop (loop output) → Expand Summaries → Write Summaries**
     - **Write Summaries → Summary Write Loop**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Setup steps: configure Toggl HTTP Basic Auth, Google Sheets OAuth2, set `start_date`, set `PROJECT_NAME` + `TIMEZONE`, replace spreadsheet IDs. | From sticky note “Instructions” |
| Detail columns expected: `description | project | start_date | start_time | start_decimal_hour | stop_date | stop_time | stop_decimal_hour | duration_hhmm | duration_hours | tags | month` | From sticky note “Instructions” |
| Summary columns expected: `date | project | description | duration_hhmm | duration_hours | tags | month` | From sticky note “Instructions” |
| Caution: sheet-prep code deletes any tab not in the detected months list (when months exist). Keep separate “config/instructions” in another spreadsheet or change deletion logic. | Derived from `Build Sheet Ops 1/2` logic |
| Workflow uses global static data as an internal cache between nodes; partial failures can leave stale values until next successful run (though some keys are deleted later). | Derived from `Process Data`, `Read Detail Data`, `Read Summary Data` |