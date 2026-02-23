Log new Google top-10 keywords to Airtable with DataForSEO and Slack alerts

https://n8nworkflows.xyz/workflows/log-new-google-top-10-keywords-to-airtable-with-dataforseo-and-slack-alerts-13433


# Log new Google top-10 keywords to Airtable with DataForSEO and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs on a schedule (weekly by default) to detect **new keywords for which one or more target domains started ranking in Google’s top 10**, using the **DataForSEO Labs API**. It then **stores the latest top-10 keyword rows in Airtable** and **sends a Slack alert listing only the newly discovered keywords** (difference vs previous run).

**Primary use cases:**
- SEO monitoring: detect new “wins” (top-10 rankings) per domain.
- Change tracking: compare current top-10 set against prior run.
- Team notifications via Slack.

### 1.1 Scheduled start
Runs every Monday at 09:00.

### 1.2 Snapshot old Airtable keywords and clear the table
Reads existing “Ranked Keywords” records from Airtable, and if any exist, deletes them (table reset). During deletion, it also builds a dataset representing the “previous run” keywords.

### 1.3 Load targets and iterate per target
Fetches target domains (with location/language), then loops over each target.

### 1.4 Pull DataForSEO ranked keywords (top-10) with pagination
For each target, fetches up to all ranked keywords where `rank_absolute <= 10`, paging in blocks of 1000 and merging all results into a single `items` array.

### 1.5 Write current snapshot to Airtable, compute “new keywords”, and notify Slack
Splits the consolidated items into rows, writes each to Airtable, aggregates writes, computes which keywords are new vs the previous snapshot, and sends Slack message only when new keywords exist.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled start
**Overview:** Triggers the workflow weekly.  
**Nodes involved:** `Run every Monday`

#### Node: Run every Monday
- **Type / role:** Schedule Trigger; starts executions on a timer.
- **Configuration (interpreted):** Weekly schedule; triggers on **Monday** at **09:00** (workflow/server timezone).
- **Outputs:** Sends an empty trigger item to `Search keywords`.
- **Edge cases / failures:**
  - Timezone mismatch (n8n instance timezone vs intended business timezone).
  - If n8n is down at trigger time, execution may be missed depending on n8n setup.

---

### Block 2 — Snapshot old keywords and clear Airtable table
**Overview:** Reads existing keyword rows from Airtable, deletes them (if any), and collects prior rows for later comparison (“previous run snapshot”).  
**Nodes involved:** `Search keywords`, `If (table is not empty)`, `Delete keyword`, `Merge`, `Set "keyword" and "target" fields`, `Aggregate`

#### Node: Search keywords
- **Type / role:** Airtable node (Search); pulls existing records from the “Ranked Keywords” table.
- **Configuration choices:**
  - **Base:** “Integrations” (`appZWQrfNksZ6S36L`)
  - **Table:** “Ranked Keywords” (`tblNstvG3A4NPvYSn`)
  - **Operation:** Search (no explicit formula/filter configured in the workflow JSON).
  - **Always Output Data:** enabled (important for downstream IF behavior).
- **Outputs:** Airtable records (each with `id` plus field values), to `If (table is not empty)`.
- **Edge cases / failures:**
  - Airtable PAT permission issues (401/403).
  - Large tables: Search pagination/limits may require configuration; otherwise could return partial results depending on node behavior/version.
  - Empty result set: downstream branch uses IF to avoid deletion loop.

#### Node: If (table is not empty)
- **Type / role:** IF node; determines whether any Airtable record exists.
- **Configuration choices:**
  - Condition: checks if `{{$json.id}}` **exists**.
  - **True output (0):** record exists → delete it.
  - **False output (1):** no record → skip deletion and go to merge.
- **Connections:**
  - True → `Delete keyword`
  - False → `Merge` (input index 1)
- **Edge cases:**
  - If Airtable node returns an unexpected structure (no `id`), condition may behave incorrectly.

#### Node: Delete keyword
- **Type / role:** Airtable node (Delete Record); deletes each prior “Ranked Keywords” record.
- **Configuration choices:**
  - Record ID: `={{ $json.id }}`
- **Connections:** Output → `Merge` (input index 0)
- **Edge cases / failures:**
  - Attempt to delete already-deleted records (retries / 404-like behavior depending on Airtable API response).
  - Rate limiting if many rows.
  - If Search returned partial results, some old rows may remain, impacting “old snapshot” logic and creating duplicates over time.

#### Node: Merge
- **Type / role:** Merge node; recombines flow after the IF/deletion paths.
- **Configuration:** default merge behavior (no explicit mode shown).
- **Inputs:**
  - Input 0 from `Delete keyword` (when table had data).
  - Input 1 from `If (table is not empty)` false branch (when empty).
- **Output:** to `Set "keyword" and "target" fields`.
- **Edge cases:**
  - Merge behavior depends on mode; with defaults, ensure it does not wait indefinitely for a second input that never arrives. (In practice, n8n Merge default is commonly “Append”; but this should be verified when recreating.)

#### Node: Set "keyword" and "target" fields
- **Type / role:** Set node; normalizes the old rows to only the fields needed for comparison later.
- **Configuration choices:**
  - Creates `keyword` = `{{ $('Search keywords').item.json.Keyword }}`
  - Creates `target` = `{{ $('Search keywords').item.json.Target }}`
  - Keeps other fields depending on Set node default behavior (here it’s “assignments”; typically it merges into existing item unless “Keep Only Set” is enabled—this workflow does not indicate “keepOnlySet”, so assume merge).
- **Output:** to `Aggregate`
- **Edge cases:**
  - Uses **cross-node item referencing** (`$('Search keywords').item...`) which is sensitive to item pairing. If deletion changes item ordering or counts, references can mismatch.
  - If your Airtable fields are named differently (e.g., “URL” vs “Url”), expressions may break.

#### Node: Aggregate
- **Type / role:** Aggregate node; collects all old keyword rows into one item for later diffing.
- **Configuration choices:**
  - `aggregateAllItemData` → outputs a single item with a `data` array containing all incoming items’ JSON.
- **Output:** to `Search Targets`
- **Edge cases:**
  - If there were no old keywords, `data` may be missing or empty; downstream code handles this (`if ($('Aggregate').first().json.data)`).

---

### Block 3 — Load targets and iterate
**Overview:** Reads the list of targets (domain + location + language) and loops through them one by one.  
**Nodes involved:** `Search Targets`, `Loop over targets`, `Initialize "items" field`, `Set "items" field`

#### Node: Search Targets
- **Type / role:** Airtable node (Search); loads target domains to check in DataForSEO.
- **Configuration choices:**
  - **Base:** “Integrations” (`appZWQrfNksZ6S36L`)
  - **Table:** “Targets (Ranked Keywords)” (`tblW8RaNBF4vKrWV1`)
  - Expected columns per sticky note: `Domain`, `Location`, `Language`
- **Output:** to `Loop over targets`
- **Edge cases / failures:**
  - Missing columns cause later expressions (Domain/Location/Language) to fail.
  - Multiple target rows produce multiple loop iterations.

#### Node: Loop over targets
- **Type / role:** SplitInBatches; iterates through targets.
- **Configuration choices:**
  - Options empty; default batch size is typically 1 (verify in UI).
- **Connections:**
  - Output 1 (loop “main” iteration) → `Initialize "items" field`
  - Output 0 (done) is not connected.
- **Edge cases:**
  - If batch size > 1, item linking with `$('Search Targets').item` becomes more complex and can misalign.

#### Node: Initialize "items" field
- **Type / role:** Set; initializes accumulator array for DataForSEO paging results.
- **Configuration choices:** sets `items = []`
- **Output:** to `Set "items" field`

#### Node: Set "items" field
- **Type / role:** Set; carries the accumulator array forward (initially empty, later reused).
- **Configuration choices:** sets `items = {{$json.items}}`
- **Output:** to `Get ranked keywords`

---

### Block 4 — DataForSEO pull (top-10) with pagination and merge
**Overview:** Calls DataForSEO Labs “ranked keywords” for each target with a filter `rank_absolute <= 10`, pages through results, and merges all pages into a single `items` list.  
**Nodes involved:** `Get ranked keywords`, `Merge "items" with DFS response`, `Has more pages?`, `Merge "items" with last response`

#### Node: Get ranked keywords
- **Type / role:** DataForSEO Labs API node; fetches ranked keywords for the target.
- **Configuration choices:**
  - Operation: `get-ranked-keywords`
  - `target_any = {{ $('Search Targets').item.json.Domain }}`
  - `language_name = {{ $('Search Targets').item.json.Language }}`
  - `location_name = {{ $('Search Targets').item.json.Location }}`
  - Filter: `["ranked_serp_element.serp_item.rank_absolute","<=",10]` (top 10)
  - Pagination:
    - `limit = 1000`
    - `offset = {{ $runIndex * 1000 }}`
- **Output:** to `Merge "items" with DFS response`
- **Edge cases / failures:**
  - Auth errors with DataForSEO (invalid login/password).
  - API quota/rate limits.
  - Unexpected response shape (e.g., missing `tasks[0].result[0].items`).
  - `$runIndex`-based paging assumes the node is re-entered in a loop-like fashion; correctness depends on the downstream “Has more pages?” routing and node re-execution behavior.

#### Node: Merge "items" with DFS response
- **Type / role:** Set; appends the newly returned DataForSEO items to the accumulator.
- **Key expression:**
  - `items = {{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items] }}`
- **Output:** to `Has more pages?`
- **Edge cases:**
  - If `tasks[0].result[0].items` is undefined (no results), spread may throw.
  - If `Set "items" field` item linkage is off, accumulator may reset or mix targets.

#### Node: Has more pages?
- **Type / role:** IF; decides whether to fetch the next DataForSEO page.
- **Condition (interpreted):**
  - `{{ $runIndex }} < {{ $('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
- **Outputs:**
  - True (0) → `Set "items" field` (intended to loop and fetch next page)
  - False (1) → `Merge "items" with last response` (finalize list)
- **Edge cases:**
  - `total_count` may be missing or 0; expression can become `NaN`.
  - The formula uses division by 1000 and `-1`; off-by-one risks when `total_count` is an exact multiple of 1000.
  - This “loop” relies on n8n execution semantics; when recreating, verify it actually re-calls `Get ranked keywords` multiple times.

#### Node: Merge "items" with last response
- **Type / role:** Set; creates the final `items` array from accumulator + last fetched page.
- **Key expression:**
  - `items = {{ [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items] }}`
- **Output:** to `Split out (items)`
- **Edge cases:**
  - Potentially duplicates last page if the accumulator already included it (depends on control flow).
  - Same undefined `items` risks as above.

---

### Block 5 — Persist snapshot, compute diff, Slack alert, loop continuation
**Overview:** Writes each keyword item to Airtable, then compares the set of keywords against the old snapshot for that target; sends Slack if there are new keywords; continues to next target.  
**Nodes involved:** `Split out (items)`, `Create a record (keyword)`, `Aggregate1`, `Find new keywords`, `Filter (has new keywords)`, `Send a message`

#### Node: Split out (items)
- **Type / role:** Split Out; converts the `items` array into individual items for record creation.
- **Configuration:** `fieldToSplitOut = items`
- **Output:** each item → `Create a record (keyword)`
- **Edge cases:**
  - If `items` is empty/missing, node may output nothing; downstream steps won’t run for that target.

#### Node: Create a record (keyword)
- **Type / role:** Airtable node (Create); stores each top-10 keyword row.
- **Configuration choices (field mapping):**
  - `Keyword` = `{{$json.keyword_data.keyword}}`
  - `Target` = `{{ $('Search Targets').item.json.Domain }}`
  - `Date` = `{{ new Date().toLocaleDateString('ISO') }}`
  - `Url` = `{{$json.ranked_serp_element.serp_item.url}}`
  - `Rank` = `{{$json.ranked_serp_element.serp_item.rank_absolute}}`
  - `Search Volume` = `{{$json.keyword_data.keyword_info.search_volume}}`
  - `Intent` = `{{$json.keyword_data.search_intent_info.main_intent}}`
- **Output:** to `Aggregate1`
- **Edge cases / failures:**
  - Airtable schema mismatch (field names/types differ from expected).
  - `toLocaleDateString('ISO')` is **not guaranteed** across JS runtimes; it may not output an ISO date. Prefer `new Date().toISOString()` when reproducing.
  - Missing nested fields from DataForSEO (e.g., no `search_intent_info`).

#### Node: Aggregate1
- **Type / role:** Aggregate; collects all created-record items for the target into one.
- **Configuration:** `aggregateAllItemData`
- **Output:** to `Find new keywords`

#### Node: Find new keywords
- **Type / role:** Code; computes the keyword difference (new vs old) for the current target.
- **Inputs used via expressions:**
  - Old snapshot: `$('Aggregate').first().json.data` (from Block 2)
  - Current pulled keywords: `$('Merge "items" with last response').first().json.items`
  - Current target filter: `$('Search Targets').first().json.Domain` (note: uses `.first()` not `.item`)
- **Logic (interpreted):**
  - Build `oldKeywords` set from old snapshot items where `item.target == currentDomain`, mapping to `item.keyword`.
  - Build `newKeywords` from current items mapping to `item.keyword_data.keyword`.
  - `diff = newKeywords.filter(x => !oldKeywords.has(x))` if oldKeywords exists, else `[]`.
  - Output: one item with `{ diff: [...] }`.
- **Configuration note:** `alwaysOutputData: true` ensures node outputs even if upstream is empty (but internal references can still be undefined).
- **Edge cases / risks:**
  - **Potential bug:** uses `$('Search Targets').first()` rather than the current loop item; with multiple targets, `.first()` always points to the first target, which can produce incorrect diffs/Slack messages.
  - If `Merge "items" with last response` did not run (e.g., paging branch differences), `items` may be undefined → diff becomes `[]`.
  - Case sensitivity and normalization: keywords differing only by casing/whitespace count as different.

#### Node: Filter (has new keywords)
- **Type / role:** Filter; passes through only if diff array is not empty.
- **Condition:** `{{$json.diff}}` is `notEmpty`
- **Output:** to `Send a message`
- **Edge cases:**
  - If diff is missing (undefined), condition may fail strict validation depending on node version.

#### Node: Send a message
- **Type / role:** Slack node; sends a summary notification.
- **Authentication:** OAuth2
- **Message text:**
  - `New keywords ranking in the top-10 Google's results for target {{ $('Search Targets').item.json.Domain }}:{{ $json.diff.join(', ') }} . See the full list in Airtable.`
- **Output / flow control:** after sending, connected back to `Loop over targets` to continue looping.
- **Edge cases:**
  - Slack OAuth token revoked / missing scopes.
  - Message length limits if `diff` is very long.
  - Again relies on `$('Search Targets').item` being the correct current target; ensure loop item context is correct.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run every Monday | Schedule Trigger | Start workflow on weekly schedule | — | Search keywords | This workflow uses the DataForSEO Labs API to detect new search queries for which your domain(s) started ranking in Google’s top 10 results. Newly discovered keywords are recorded in Airtable, and a short overview is sent to you in a Slack notification.  |
| Search keywords | Airtable (Search) | Load previous keyword snapshot from Airtable | Run every Monday | If (table is not empty) | ## Get previous keywords and clear the table  Select your table with Ranked Keywords. The table is expected to have these columns: Keyword, Target, Date, URL, Search Volume, Intent, Rank. |
| If (table is not empty) | IF | Branch if Airtable returned records | Search keywords | Delete keyword (true), Merge (false) | ## Get previous keywords and clear the table  Select your table with Ranked Keywords. The table is expected to have these columns: Keyword, Target, Date, URL, Search Volume, Intent, Rank. |
| Delete keyword | Airtable (Delete Record) | Delete existing keyword rows (table reset) | If (table is not empty) | Merge | ## Get previous keywords and clear the table  Select your table with Ranked Keywords. The table is expected to have these columns: Keyword, Target, Date, URL, Search Volume, Intent, Rank. |
| Merge | Merge | Rejoin delete/skip-delete paths | Delete keyword, If (table is not empty) | Set "keyword" and "target" fields | ## Get previous keywords and clear the table  Select your table with Ranked Keywords. The table is expected to have these columns: Keyword, Target, Date, URL, Search Volume, Intent, Rank. |
| Set "keyword" and "target" fields | Set | Normalize old snapshot fields for comparison | Merge | Aggregate | ## Get previous keywords and clear the table  Select your table with Ranked Keywords. The table is expected to have these columns: Keyword, Target, Date, URL, Search Volume, Intent, Rank. |
| Aggregate | Aggregate | Collect all old snapshot rows into one item | Set "keyword" and "target" fields | Search Targets | ## Get previous keywords and clear the table  Select your table with Ranked Keywords. The table is expected to have these columns: Keyword, Target, Date, URL, Search Volume, Intent, Rank. |
| Search Targets | Airtable (Search) | Load list of target domains/locations/languages | Aggregate | Loop over targets | ## Get targets  Select your table with targets. The table is expected to have these columns: Domain, Location, and Language. |
| Loop over targets | SplitInBatches | Iterate per target | Search Targets, Send a message | Initialize "items" field | ## Getn new keywords ranked in Google’s top 10 results with DataForSEO  Create a DataForSEO connection and set up additional parameters if needed. |
| Initialize "items" field | Set | Initialize paging accumulator array | Loop over targets | Set "items" field | ## Getn new keywords ranked in Google’s top 10 results with DataForSEO  Create a DataForSEO connection and set up additional parameters if needed. |
| Set "items" field | Set | Carry accumulator forward | Initialize "items" field, Has more pages? | Get ranked keywords | ## Getn new keywords ranked in Google’s top 10 results with DataForSEO  Create a DataForSEO connection and set up additional parameters if needed. |
| Get ranked keywords | DataForSEO Labs API | Fetch top-10 ranked keywords (paged) | Set "items" field | Merge "items" with DFS response | ## Getn new keywords ranked in Google’s top 10 results with DataForSEO  Create a DataForSEO connection and set up additional parameters if needed. |
| Merge "items" with DFS response | Set | Append API items to accumulator | Get ranked keywords | Has more pages? | ## Getn new keywords ranked in Google’s top 10 results with DataForSEO  Create a DataForSEO connection and set up additional parameters if needed. |
| Has more pages? | IF | Decide whether to fetch next API page | Merge "items" with DFS response | Set "items" field (true), Merge "items" with last response (false) | ## Getn new keywords ranked in Google’s top 10 results with DataForSEO  Create a DataForSEO connection and set up additional parameters if needed. |
| Merge "items" with last response | Set | Finalize consolidated items array | Has more pages? | Split out (items) | ## Getn new keywords ranked in Google’s top 10 results with DataForSEO  Create a DataForSEO connection and set up additional parameters if needed. |
| Split out (items) | Split Out | Convert consolidated array to individual items | Merge "items" with last response | Create a record (keyword) | ## Save keywords ranked in Google’s top 10 results to Airtable and send new keywords to Slack  Select your table with Ranked Keywords.  Create a Slack connection and set a receiver. Add additional information to the message if needed. |
| Create a record (keyword) | Airtable (Create) | Store each ranked keyword row | Split out (items) | Aggregate1 | ## Save keywords ranked in Google’s top 10 results to Airtable and send new keywords to Slack  Select your table with Ranked Keywords.  Create a Slack connection and set a receiver. Add additional information to the message if needed. |
| Aggregate1 | Aggregate | Aggregate created rows for diff calculation | Create a record (keyword) | Find new keywords | ## Save keywords ranked in Google’s top 10 results to Airtable and send new keywords to Slack  Select your table with Ranked Keywords.  Create a Slack connection and set a receiver. Add additional information to the message if needed. |
| Find new keywords | Code | Compute diff between previous and current keywords | Aggregate1 | Filter (has new keywords) | ## Save keywords ranked in Google’s top 10 results to Airtable and send new keywords to Slack  Select your table with Ranked Keywords.  Create a Slack connection and set a receiver. Add additional information to the message if needed. |
| Filter (has new keywords) | Filter | Only pass through when diff is non-empty | Find new keywords | Send a message | ## Save keywords ranked in Google’s top 10 results to Airtable and send new keywords to Slack  Select your table with Ranked Keywords.  Create a Slack connection and set a receiver. Add additional information to the message if needed. |
| Send a message | Slack | Notify Slack with new keywords | Filter (has new keywords) | Loop over targets | ## Save keywords ranked in Google’s top 10 results to Airtable and send new keywords to Slack  Select your table with Ranked Keywords.  Create a Slack connection and set a receiver. Add additional information to the message if needed. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Schedule Trigger** node named **“Run every Monday”**  
   - Set interval to **Weeks**
   - Day: **Monday**
   - Hour: **09:00**
3. **Add Airtable node** named **“Search keywords”** (operation: **Search**)  
   - Configure Airtable credentials: **Airtable Personal Access Token** with access to the base  
   - Select Base: your base (e.g., “Integrations”)  
   - Select Table: **Ranked Keywords**  
   - Leave search options default unless you need filtering.
4. **Add IF node** named **“If (table is not empty)”**  
   - Condition: String → **exists**  
   - Left value: `{{$json.id}}`
   - Connect: `Search keywords → If (table is not empty)`
5. **Add Airtable node** named **“Delete keyword”** (operation: **Delete Record**)  
   - Record ID: `{{$json.id}}`
   - Same Base/Table as Ranked Keywords  
   - Connect: IF **true** output → `Delete keyword`
6. **Add Merge node** named **“Merge”**  
   - Connect: `Delete keyword → Merge (Input 0)`  
   - Connect: IF **false** output → `Merge (Input 1)`  
   - Keep Merge mode as default *but verify in UI* it won’t wait for both inputs when one path is unused.
7. **Add Set node** named **“Set "keyword" and "target" fields”**  
   - Add fields:
     - `keyword` (String) = `{{ $('Search keywords').item.json.Keyword }}`
     - `target` (String) = `{{ $('Search keywords').item.json.Target }}`
   - Connect: `Merge → Set "keyword" and "target" fields`
8. **Add Aggregate node** named **“Aggregate”**  
   - Mode: **Aggregate All Item Data**
   - Connect: `Set "keyword" and "target" fields → Aggregate`
9. **Add Airtable node** named **“Search Targets”** (operation: **Search**)  
   - Same Airtable credential  
   - Select Table: **Targets (Ranked Keywords)** containing **Domain, Location, Language**
   - Connect: `Aggregate → Search Targets`
10. **Add SplitInBatches node** named **“Loop over targets”**  
    - Batch size: **1** (recommended for correct item context)
    - Connect: `Search Targets → Loop over targets`
11. **Add Set node** named **“Initialize "items" field”**  
    - Field: `items` (Array) = `{{ [] }}`
    - Connect: `Loop over targets (loop output) → Initialize "items" field`
12. **Add Set node** named **“Set "items" field”**  
    - Field: `items` (Array) = `{{ $json.items }}`
    - Connect: `Initialize "items" field → Set "items" field`
13. **Add DataForSEO node** named **“Get ranked keywords”**  
    - Create DataForSEO credentials using **API login + password**  
    - Operation: **Labs API → get-ranked-keywords**  
    - Parameters:
      - `target_any` = `{{ $('Search Targets').item.json.Domain }}`
      - `location_name` = `{{ $('Search Targets').item.json.Location }}`
      - `language_name` = `{{ $('Search Targets').item.json.Language }}`
      - `filters` = `["ranked_serp_element.serp_item.rank_absolute","<=",10]`
      - `limit` = `1000`
      - `offset` = `{{ $runIndex * 1000 }}`
    - Connect: `Set "items" field → Get ranked keywords`
14. **Add Set node** named **“Merge "items" with DFS response”**  
    - Field `items` (Array) = `{{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items] }}`
    - Connect: `Get ranked keywords → Merge "items" with DFS response`
15. **Add IF node** named **“Has more pages?”**  
    - Number condition: `{{ $runIndex }}` **lt** `{{ $('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
    - Connect: `Merge "items" with DFS response → Has more pages?`
16. **Loop pagination**  
    - Connect IF **true** → `Set "items" field` (to fetch next page)  
    - Add Set node **“Merge "items" with last response”** for IF **false**:
      - `items` = `{{ [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items]}}`
17. **Add Split Out node** named **“Split out (items)”**  
    - Field to split: `items`
    - Connect: `Merge "items" with last response → Split out (items)`
18. **Add Airtable node** named **“Create a record (keyword)”** (operation: **Create**)  
    - Base/Table: **Ranked Keywords**
    - Map fields:
      - Keyword = `{{$json.keyword_data.keyword}}`
      - Target = `{{ $('Search Targets').item.json.Domain }}`
      - Date = `{{ new Date().toISOString() }}` (recommended)  
      - Url = `{{$json.ranked_serp_element.serp_item.url}}`
      - Rank = `{{$json.ranked_serp_element.serp_item.rank_absolute}}`
      - Search Volume = `{{$json.keyword_data.keyword_info.search_volume}}`
      - Intent = `{{$json.keyword_data.search_intent_info.main_intent}}`
    - Connect: `Split out (items) → Create a record (keyword)`
19. **Add Aggregate node** named **“Aggregate1”**  
    - Mode: Aggregate All Item Data
    - Connect: `Create a record (keyword) → Aggregate1`
20. **Add Code node** named **“Find new keywords”**  
    - Paste the logic (adapt if you fix the multi-target `.first()` issue; see note below).  
    - Connect: `Aggregate1 → Find new keywords`
21. **Add Filter node** named **“Filter (has new keywords)”**  
    - Condition: Array notEmpty on `{{ $json.diff }}`
    - Connect: `Find new keywords → Filter (has new keywords)`
22. **Add Slack node** named **“Send a message”**  
    - Credentials: Slack OAuth2 (grant chat:write to the target channel/user)  
    - Text: `New keywords ranking in the top-10 Google's results for target {{ $('Search Targets').item.json.Domain }}:{{ $json.diff.join(', ') }} . See the full list in Airtable.`
    - Connect: `Filter (has new keywords) → Send a message`
23. **Close the target loop**  
    - Connect: `Send a message → Loop over targets` (so it continues to next target)
    - (Optionally) also connect the “no new keywords” path to the loop end; in the provided workflow it only loops after Slack, meaning targets with no new keywords may stop the loop depending on execution path. In practice you should add a second connection to ensure looping continues even when Filter blocks.

**Important reproduction note (logic correctness):**  
To correctly compute diffs per target, change references in the Code node from `$('Search Targets').first()` to `$('Search Targets').item` (or use the current input item’s domain passed along the loop), otherwise multi-target runs can compare against the wrong domain.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow uses the DataForSEO Labs API to detect new search queries for which your domain(s) started ranking in Google’s top 10 results. Newly discovered keywords are recorded in Airtable, and a short overview is sent to you in a Slack notification. | Sticky note (overall workflow description and setup guidance) |
| Select your table with Ranked Keywords. The table is expected to have these columns: Keyword, Target, Date, URL, Search Volume, Intent, Rank. | Sticky note for Airtable “Ranked Keywords” setup |
| Select your table with targets. The table is expected to have these columns: Domain, Location, and Language. | Sticky note for Airtable “Targets” setup |
| Create a DataForSEO connection and set up additional parameters if needed. | Sticky note for DataForSEO configuration |
| Create a Slack connection and set a receiver. Add additional information to the message if needed. | Sticky note for Slack notification configuration |