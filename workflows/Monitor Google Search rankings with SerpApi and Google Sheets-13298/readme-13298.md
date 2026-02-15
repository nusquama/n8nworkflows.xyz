Monitor Google Search rankings with SerpApi and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-google-search-rankings-with-serpapi-and-google-sheets-13298


# Monitor Google Search rankings with SerpApi and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Monitor Google Search rankings with SerpApi and Google Sheets  
**Workflow name (in JSON):** Monitor Google Search rank with SerpApi & Google Sheets  
**Purpose:** Automatically monitor Google organic rankings for a list of **keyword + target (domain or page URL)** pairs stored in Google Sheets, using **SerpApi** for Google Search results. It writes outputs to:
1) a **Rank Log** sheet (append historical results), and  
2) a **Latest Run** sheet (update “current” rank per row).

**Primary use cases**
- Daily SEO rank tracking for multiple keywords/domains.
- Lightweight “position monitoring” without operating your own scraper.
- Keeping an audit trail (log) plus a current snapshot (latest run).

### 1.1 Logical Blocks
1. **Trigger & Input Acquisition**
   - Scheduled trigger (daily at 10:00 UTC) and Google Sheets read of keyword/target rows.
2. **Iteration / Loop Control**
   - Batch loop over rows (one keyword/target pair at a time).
3. **Pagination Configuration**
   - Initialize pagination variables (start offset), page limit, and matching mode (domain vs page).
4. **SERP Query & Rank Parsing**
   - Query SerpApi Google Search API and parse organic results to find rank for target.
5. **Stop/Continue Decision**
   - Decide whether to paginate further for the same keyword or move to the next keyword.
6. **Persistence to Google Sheets + Rate-Limit Buffer**
   - Append to Rank Log, update Latest Run, wait 4s, then continue loop.

---

## 2. Block-by-Block Analysis

### Block A — Trigger & Input Acquisition
**Overview:** Starts the workflow on a schedule and loads keyword/target rows from a Google Sheet (the “Latest Run” sheet) that acts as the input list.

**Nodes Involved**
- Schedule Workflow
- Get Keywords and Domains to Match
- Sticky Note1 (commentary)
- Sticky Note6 (commentary)

#### Node: Schedule Workflow
- **Type / role:** `Schedule Trigger` — entry point; runs automatically.
- **Configuration (interpreted):**
  - Runs **daily at 10:00 UTC** (`triggerAtHour: 10`).
- **Connections:**
  - Output → **Get Keywords and Domains to Match**
- **Potential failures / edge cases:**
  - Timezone assumptions: schedule uses instance timezone/UTC behavior; verify instance settings.
  - If workflow is inactive, schedule won’t run.
- **Version notes:** typeVersion `1.2` (modern schedule trigger behavior).

#### Node: Get Keywords and Domains to Match
- **Type / role:** `Google Sheets` — fetches the table of keywords and targets to process.
- **Configuration (interpreted):**
  - Uses **Document URL** and **Sheet URL** pointing to a spreadsheet (gid=1750873622).
  - Operation is not explicitly shown in JSON snippet (node parameters only show documentId/sheetName/options). In n8n v4+ Google Sheets nodes, this typically implies a **Read / Get Many Rows**-style operation when used as an input list. When recreating, set it to **Read → Get Many** (or equivalent) to return all rows.
- **Key fields expected in each output item (based on downstream expressions):**
  - `keyword`
  - `target`
  - `row_number` (used to update the Latest Run sheet by row)
  - (optional) `match_type` (overrides global default per row if present)
- **Connections:**
  - Input ← Schedule Workflow
  - Output → Loop Over Keywords
- **Potential failures / edge cases:**
  - Google Sheets OAuth not connected / expired.
  - Wrong sheet selected (missing required columns).
  - If empty sheet: loop will have nothing to process.
  - `row_number` missing: later update-by-row will fail or update wrong rows.
- **Version notes:** typeVersion `4.5` (newer Google Sheets node).

#### Sticky Note6 (covers overall workflow guidance)
- **Purpose:** Setup instructions and links.
- **Notable resources:**
  - SerpApi account: https://serpapi.com/
  - API key: https://serpapi.com/manage-api-key
  - Google Sheets integration help: https://n8n.io/integrations/google-sheets/
  - Template sheet: https://docs.google.com/spreadsheets/d/148gjSSqSY5x9Gz5JWE_FDXHOuB7ASTomSyxkZjrjuNc/edit?gid=1750873622#gid=1750873622
  - SerpApi Google Search API: https://serpapi.com/search-api
  - SerpApi n8n node guide: https://serpapi.com/blog/boost-your-n8n-workflows-with-serpapis-verified-node/

#### Sticky Note1
- **Purpose:** Confirms schedule and explains you should plug in your own sheet; workflow loops keyword/domain pairs.

---

### Block B — Iteration / Loop Control
**Overview:** Iterates through each row (keyword/target pair). The loop is controlled by a Split In Batches node and a Wait node that feeds back into it.

**Nodes Involved**
- Loop Over Keywords
- Wait Before Next Keyword

#### Node: Loop Over Keywords
- **Type / role:** `Split In Batches` — controls iteration over incoming items.
- **Configuration (interpreted):**
  - Uses default options; batch size not explicitly set (in n8n, default is typically 1 unless configured).
  - This node has two outputs:
    - **Output 0** (main batch output) is unused in this workflow.
    - **Output 1** (“noItemsLeft” / continuation branch depending on node behavior) is used to start processing each item.
  - Connection mapping shows **Output index 1** goes to “Set Page Limit & Match Type”.
- **Connections:**
  - Input ← Get Keywords and Domains to Match
  - Output (index 1) → Set Page Limit & Match Type
  - Input (loop-back) ← Wait Before Next Keyword
- **Potential failures / edge cases:**
  - If batch size is not 1, the rest of the workflow assumes “current item” semantics; test carefully.
  - If output wiring is wrong (using output 0 vs 1), the loop won’t run as intended.
- **Version notes:** typeVersion `3`.

#### Node: Wait Before Next Keyword
- **Type / role:** `Wait` — throttles between keywords to reduce Google Sheets API rate limit issues.
- **Configuration (interpreted):**
  - Waits **4 seconds** (`amount: 4`). (Unit defaults to seconds for this node version.)
- **Connections:**
  - Input ← Update Latest Run
  - Output → Loop Over Keywords (to request next item/batch)
- **Potential failures / edge cases:**
  - Wait nodes can prolong executions; ensure your n8n execution time limits and queue capacity handle it.
  - If running large keyword lists, this will significantly increase total runtime (4s * N).
- **Version notes:** typeVersion `1.1`.

---

### Block C — Pagination Configuration
**Overview:** Initializes global/static variables used for rank calculation and for paging through SERPs.

**Nodes Involved**
- Set Page Limit & Match Type
- Sticky Note7 (commentary)

#### Node: Set Page Limit & Match Type
- **Type / role:** `Code` — sets workflow static data used across subsequent nodes.
- **Configuration (interpreted):**
  - Uses `$getWorkflowStaticData('global')` to store:
    - `data.start = 0` (pagination offset)
    - `data.page_limit = 10` (means check up to 10 pages; each page assumed 10 results → up to 100 results)
    - `data.match_type = 'domain'` (default matching behavior)
  - Returns `data` so downstream nodes can use `start`, `page_limit`, `match_type`.
- **Key variables:**
  - `start`: used by SerpApi `start` parameter and rank computation.
  - `page_limit`: used to stop pagination.
  - `match_type`: `'domain'` or `'page'`.
- **Connections:**
  - Input ← Loop Over Keywords
  - Output → Search Google
- **Potential failures / edge cases:**
  - Static data is global and persists between executions in some scenarios; but this node resets `start` to 0 each keyword iteration (good).
  - If per-row `match_type` is present later, it may override; ensure values are only `'domain'` or `'page'`.
- **Version notes:** typeVersion `2`.

#### Sticky Note7
- **Purpose:** Explains how to change `data.page_limit` and `data.match_type`.

---

### Block D — SERP Query & Rank Parsing
**Overview:** Calls SerpApi Google Search for the keyword (optionally paginated) and parses organic results to find the rank of the target domain or exact page.

**Nodes Involved**
- Search Google
- Parse Rank for Target Domain or Page
- Sticky Note2 (commentary)

#### Node: Search Google
- **Type / role:** `SerpApi` (verified community/partner node) — performs Google Search API request.
- **Configuration (interpreted):**
  - Query `q` is taken from the current row:
    - `q = {{ $('Get Keywords and Domains to Match').item.json.keyword }}`
  - Location fixed: `"Austin, Texas, United States"` (can be changed).
  - Additional fields:
    - `start = {{ $json.start }}` (pagination offset passed in; comes from the previous Code node output)
    - `no_cache = false`
- **Connections:**
  - Input ← Set Page Limit & Match Type (and later, If node’s “continue paging” branch also returns here)
  - Output → Parse Rank for Target Domain or Page
- **Potential failures / edge cases:**
  - SerpApi credential missing/invalid; quota exceeded.
  - Location may affect rankings; consider making it configurable per row.
  - `start` must be numeric and aligned to SerpApi expectations (typically 0, 10, 20...).
- **Version notes:** typeVersion `1`.

#### Node: Parse Rank for Target Domain or Page
- **Type / role:** `Code` — inspects SerpApi results, normalizes URLs, finds match, computes rank, and increments pagination.
- **Configuration (interpreted):**
  - Reads SerpApi response from `$input.first().json`.
  - Uses global static data: `const data = $getWorkflowStaticData('global');`
  - Guard: if `search_information.organic_results_state != "Fully empty"`, it tries to match.
  - Pulls target and match type:
    - `targetUrl = $('Loop Over Keywords').first().json.target`
    - `matchType = $('Loop Over Keywords').first().json.match_type || data.match_type`
  - URL normalization:
    - trims quotes
    - strips `http(s)://`
    - strips `www.`
    - strips trailing `/`
    - lowercases
  - Matching logic:
    - If `matchType === 'domain'` OR target has no path: match by **domain** only.
    - Else (page mode with path): match by **exact normalized URL** (domain + path).
  - Rank calculation:
    - If found at `index >= 0`: `data.rank = data.start + index + 1`
    - Else: `data.rank = "N/A"`
  - Pagination increment:
    - `data.start += 10`
  - If organic results “Fully empty”:
    - `data.rank = "N/A"`
    - `data.start = 100000` (forces the pagination stop condition downstream)
  - Returns `data` (so downstream nodes read `rank`, `start`, `page_limit`, etc.)
- **Connections:**
  - Input ← Search Google
  - Output → If - Next Page or Next Keyword
- **Potential failures / edge cases:**
  - If SerpApi response structure changes or `organic_results` is missing, code can throw.
  - The variable `index` is not declared with `let/const` in the code (it becomes a global variable in JS runtime). This can cause unexpected behavior; best practice is `let index = -1;`.
  - If `target` is empty/invalid, normalization returns null and matching may fail.
  - “Page mode” exact match is strict after normalization; query-string differences or redirects won’t match.
- **Version notes:** typeVersion `2`.

#### Sticky Note2
- **Purpose:** Explains search + parsing behavior, URL matching modes, and URL normalization.

---

### Block E — Stop/Continue Decision (Pagination vs Next Keyword)
**Overview:** Determines whether to continue paginating SERP pages for the same keyword or stop and record results / move to next keyword.

**Nodes Involved**
- If - Next Page or Next Keyword
- Sticky Note8 (commentary)

#### Node: If - Next Page or Next Keyword
- **Type / role:** `IF` — branching based on whether a rank was found or the pagination limit was reached.
- **Configuration (interpreted):**
  - Conditions use **OR** combinator:
    1) `rank != "N/A"`  (string notEquals)
    2) `page_limit * 10 <= start` (number lte)  
       - Note: because `start` increments by 10, and page_limit=10, stop when `start` reaches 100.
  - Behavior:
    - **True branch:** go to Google Sheets updates (rank found OR limit hit)
    - **False branch:** go back to Search Google (next page of results)
- **Connections:**
  - Input ← Parse Rank for Target Domain or Page
  - True → Update Rank Log
  - False → Search Google
- **Potential failures / edge cases:**
  - If `page_limit` or `start` are strings (type coercion), numeric comparison may behave unexpectedly (loose validation is enabled).
  - The condition uses `page_limit * 10 <= start`. With `start` incrementing after each parse, ensure off-by-one behavior matches your intended number of pages.
- **Version notes:** typeVersion `2.3`.

#### Sticky Note8
- **Purpose:** Explains branching rule: next keyword if match found or pagination limit hit; otherwise keep paging.

---

### Block F — Persistence to Google Sheets + Rate-Limit Buffer
**Overview:** Writes results to two sheets: append to historical log and update the “Latest Run” overview row for the current keyword/target.

**Nodes Involved**
- Update Rank Log
- Update Latest Run
- Sticky Note4 (commentary)

#### Node: Update Rank Log
- **Type / role:** `Google Sheets` — appends a new log record per keyword run.
- **Configuration (interpreted):**
  - **Operation:** Append
  - Writes columns:
    - `searched_at = {{ $now.toISO() }}`
    - `target = {{ $('Get Keywords and Domains to Match').item.json.target }}`
    - `keyword = {{ $('Search Google').item.json.search_parameters.q }}`
    - `rank = {{ $json.rank }}` (rank computed by parse node)
  - Target sheet uses gid=0 (likely “Rank Log” tab).
- **Connections:**
  - Input ← If (True branch)
  - Output → Update Latest Run
- **Potential failures / edge cases:**
  - Rate limits (Google Sheets API).
  - If sheet columns don’t match mapping, append may put values in wrong columns.
  - If `rank` can be `"N/A"` ensure column type accepts string.
- **Version notes:** typeVersion `4.5`.

#### Node: Update Latest Run
- **Type / role:** `Google Sheets` — updates the corresponding row in the “Latest Run” sheet so it always shows the most recent rank.
- **Configuration (interpreted):**
  - **Operation:** Update
  - **Matching column:** `row_number`
  - Match value expression (implicitly): `row_number = {{ $('Get Keywords and Domains to Match').item.json.row_number }}`
  - Writes columns:
    - `searched_at = {{ $now.toISO() }}`
    - `target = {{ $('Get Keywords and Domains to Match').item.json.target }}`
    - `keyword = {{ $('Search Google').item.json.search_parameters.q }}`
    - `rank = {{ $('Parse Rank for Target Domain or Page').item.json.rank }}`
    - `row_number = {{ $('Get Keywords and Domains to Match').item.json.row_number }}`
- **Connections:**
  - Input ← Update Rank Log
  - Output → Wait Before Next Keyword
- **Potential failures / edge cases:**
  - If `row_number` is missing or not aligned with the sheet’s actual row numbering, updates will fail or overwrite wrong rows.
  - If you replace the sheet and mappings reset, you must reapply expressions (see Sticky Note4).
- **Version notes:** typeVersion `4.5`.

#### Sticky Note4
- **Purpose:** Documents exact mappings (expressions) to reapply if they get cleared, and notes the 4-second wait for rate limiting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note6 | Sticky Note | Global setup notes & links |  |  | ## Google Keyword SEO Monitoring … (includes SerpApi/Sheets links and template sheet link) |
| Schedule Workflow | Schedule Trigger | Daily trigger at 10:00 UTC |  | Get Keywords and Domains to Match | ## Kick Off the Workflow … (schedule + bring your own sheet) |
| Get Keywords and Domains to Match | Google Sheets | Read keyword/target rows | Schedule Workflow | Loop Over Keywords | ## Kick Off the Workflow … (bring your own sheet; loops pairs) |
| Loop Over Keywords | Split In Batches | Iterate rows / loop control | Get Keywords and Domains to Match; Wait Before Next Keyword | Set Page Limit & Match Type | ## Kick Off the Workflow … (loops pairs) |
| Sticky Note1 | Sticky Note | Notes about kickoff & looping |  |  | ## Kick Off the Workflow … |
| Set Page Limit & Match Type | Code | Initialize pagination + matching defaults | Loop Over Keywords | Search Google | ## Set Page Limit & Match Type … (page_limit and match_type guidance) |
| Sticky Note7 | Sticky Note | Notes for page limit and match type |  |  | ## Set Page Limit & Match Type … |
| Search Google | SerpApi | Query Google via SerpApi (paged) | Set Page Limit & Match Type; If - Next Page or Next Keyword (false) | Parse Rank for Target Domain or Page | ## Search & Parse Rank … (matching behavior + normalization) |
| Parse Rank for Target Domain or Page | Code | Normalize URLs, find rank, advance pagination | Search Google | If - Next Page or Next Keyword | ## Search & Parse Rank … |
| Sticky Note2 | Sticky Note | Notes for parsing and URL logic |  |  | ## Search & Parse Rank … |
| If - Next Page or Next Keyword | IF | Stop paging or continue | Parse Rank for Target Domain or Page | Update Rank Log (true); Search Google (false) | ## Next Page or Next Keyword … |
| Sticky Note8 | Sticky Note | Notes on branching logic |  |  | ## Next Page or Next Keyword … |
| Update Rank Log | Google Sheets | Append historical log entry | If - Next Page or Next Keyword (true) | Update Latest Run | ## Update Google Sheet … (mappings + row_number + rate limit note) |
| Update Latest Run | Google Sheets | Update current snapshot row by row_number | Update Rank Log | Wait Before Next Keyword | ## Update Google Sheet … (mappings + row_number + rate limit note) |
| Sticky Note4 | Sticky Note | Notes for sheet update mappings |  |  | ## Update Google Sheet … |
| Wait Before Next Keyword | Wait | Throttle then continue loop | Update Latest Run | Loop Over Keywords | ## Update Google Sheet … (4-second wait rationale) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Monitor Google Search rank with SerpApi & Google Sheets* (or your preferred title).

2. **Add Trigger: “Schedule Workflow”**
   - Node type: **Schedule Trigger**
   - Configure: run daily at **10:00** (UTC or your instance timezone; align with your needs).
   - Connect → (next step) Google Sheets read node.

3. **Add Google Sheets read: “Get Keywords and Domains to Match”**
   - Node type: **Google Sheets**
   - Credentials: connect Google via OAuth2 in n8n.
   - Operation: **Read / Get Many Rows** (wording depends on your n8n version).
   - Select your spreadsheet and the sheet/tab that contains the input list (often “Latest Run”).
   - Ensure the sheet has at least columns:
     - `keyword` (string)
     - `target` (string; domain or full URL)
     - `row_number` (number; used for updating the same row later)
     - optional `match_type` (`domain` or `page`)
   - Connect → Split In Batches.

4. **Add loop controller: “Loop Over Keywords”**
   - Node type: **Split In Batches**
   - Batch size: set to **1** (recommended for per-row processing).
   - Connect **the output that represents “next item”** to the next node (in the provided workflow, output index 1 is used—verify your node’s port labels).
   - Later you will connect a Wait node back into this node to continue iteration.

5. **Add code initializer: “Set Page Limit & Match Type”**
   - Node type: **Code**
   - Paste/implement logic to set:
     - `start = 0`
     - `page_limit = 10` (or your desired page count)
     - `match_type = 'domain'` (or `'page'`)
   - Use `$getWorkflowStaticData('global')` and return the object.
   - Connect from loop → this node → SerpApi node.

6. **Add SerpApi node: “Search Google”**
   - Node type: **SerpApi**
   - Credentials: add SerpApi API key in n8n credentials.
   - Configure:
     - `q = {{ $('Get Keywords and Domains to Match').item.json.keyword }}`
     - Location: set to your desired location (or make it dynamic later).
     - Additional field `start = {{ $json.start }}`
     - `no_cache = false` (optional)
   - Connect → parsing Code node.

7. **Add parsing logic: “Parse Rank for Target Domain or Page”**
   - Node type: **Code**
   - Implement logic to:
     - Read SerpApi organic results
     - Normalize target and result URLs (strip protocol, `www.`, trailing slash; lowercase)
     - Match by:
       - domain match if `match_type == 'domain'` or target has no path
       - exact URL match if `match_type == 'page'` and target includes path
     - Set `rank` to computed rank or `"N/A"`
     - Increment `start` by 10 for next page
     - If results empty, force stop by setting `start` to a very large number
   - Return the data object containing `rank`, `start`, `page_limit`, etc.
   - Connect → IF node.

8. **Add decision node: “If - Next Page or Next Keyword”**
   - Node type: **IF**
   - Condition group: **OR**
     1) `{{ $json.rank }}` **not equals** `N/A`
     2) `{{ $json.page_limit * 10 }}` **<=** `{{ $json.start }}`
   - True branch → Google Sheets writes (log/update).
   - False branch → back to **Search Google** (pagination loop).

9. **Add Google Sheets append: “Update Rank Log”**
   - Node type: **Google Sheets**
   - Operation: **Append**
   - Choose your spreadsheet and the “Rank Log” sheet/tab.
   - Map fields:
     - `searched_at = {{ $now.toISO() }}`
     - `target = {{ $('Get Keywords and Domains to Match').item.json.target }}`
     - `keyword = {{ $('Search Google').item.json.search_parameters.q }}`
     - `rank = {{ $json.rank }}`
   - Connect → Update Latest Run node.

10. **Add Google Sheets update: “Update Latest Run”**
   - Node type: **Google Sheets**
   - Operation: **Update**
   - Choose your spreadsheet and the “Latest Run” sheet/tab.
   - Set **matching column** to `row_number`
   - Set match value:
     - `{{ $('Get Keywords and Domains to Match').item.json.row_number }}`
   - Map fields:
     - `searched_at = {{ $now.toISO() }}`
     - `target = {{ $('Get Keywords and Domains to Match').item.json.target }}`
     - `keyword = {{ $('Search Google').item.json.search_parameters.q }}`
     - `rank = {{ $('Parse Rank for Target Domain or Page').item.json.rank }}`
     - `row_number = {{ $('Get Keywords and Domains to Match').item.json.row_number }}`

11. **Add throttle: “Wait Before Next Keyword”**
   - Node type: **Wait**
   - Configure: **4 seconds** (adjust to your Sheets API quota).
   - Connect: Update Latest Run → Wait → back to **Loop Over Keywords** to fetch the next item.

12. **(Optional) Add sticky notes / documentation nodes**
   - Not required for execution, but helpful for maintainability.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SerpApi account creation | https://serpapi.com/ |
| SerpApi API key management | https://serpapi.com/manage-api-key |
| n8n Google Sheets integration info | https://n8n.io/integrations/google-sheets/ |
| Example spreadsheet template (copy to your account) | https://docs.google.com/spreadsheets/d/148gjSSqSY5x9Gz5JWE_FDXHOuB7ASTomSyxkZjrjuNc/edit?gid=1750873622#gid=1750873622 |
| SerpApi Google Search API docs | https://serpapi.com/search-api |
| SerpApi node intro/guide for n8n | https://serpapi.com/blog/boost-your-n8n-workflows-with-serpapis-verified-node/ |
| Rate limiting | Workflow waits 4 seconds between keywords to reduce Google Sheets API rate limit errors; adjust if needed. |
| Matching modes | `domain` matches any result on the domain; `page` matches an exact normalized URL when target includes a path. |

