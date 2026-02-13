Detect offboarded client links on PBN sites with Google Sheets and HTTP

https://n8nworkflows.xyz/workflows/detect-offboarded-client-links-on-pbn-sites-with-google-sheets-and-http-13186


# Detect offboarded client links on PBN sites with Google Sheets and HTTP

## 1. Workflow Overview

**Title:** Detect offboarded client links on PBN sites with Google Sheets and HTTP

**Purpose:**  
This workflow checks each PBN site listed in a Google Sheet, fetches the live HTML, and detects whether any **offboarded client domains** (from a separate sheet) still appear on the site. It writes the matched domain (or `0` if none) back into the **“Offboarded Links”** column for that PBN row, processing rows sequentially.

**Target use cases**
- Auditing PBNs for “dead” / offboarded client links that must be removed
- Ongoing compliance checks after client offboarding
- A lightweight link presence scan using raw HTML matching (no crawler required)

### Logical Blocks
**1.1 Trigger & Input Reception**  
Manual start → read PBN rows from Google Sheets.

**1.2 Row Filtering (Resume Capability)**  
Skip already-processed rows (where “Offboarded Links” is not blank) and continue from the first unprocessed row onward.

**1.3 Per-PBN Loop: Fetch HTML + Load Offboarded Domains**  
Iterate PBNs one-by-one; fetch HTML for current PBN; read offboarded domains list from a second sheet.

**1.4 Matching & Writing Result + Throttling**  
Find which offboarded domains occur in HTML; write match back to the PBN row; wait, then continue the loop.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Input Reception
**Overview:** Starts the workflow manually and loads the full list of PBN sites from the “PBNs” sheet.

**Nodes Involved**
- Run Workflow Manually
- Read PBN Sites from Sheet

#### Node: Run Workflow Manually
- **Type / Role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) – entry point
- **Configuration:** No parameters; user runs from n8n UI
- **Connections:**  
  - **Output →** Read PBN Sites from Sheet
- **Edge cases / failures:** None (only runs when manually executed)
- **Version notes:** TypeVersion 1

#### Node: Read PBN Sites from Sheet
- **Type / Role:** Google Sheets (`n8n-nodes-base.googleSheets`) – read PBN inventory
- **Configuration (interpreted):**
  - Reads rows from Google Spreadsheet:
    - **Document:** `1N9xfOX8V6JFATJFaoc-_0xUnYWt-gWD9FmvHKQ0hNfc`
    - **Sheet tab:** `PBNs` (gid `879625869`)
  - Uses default read behavior (no custom range specified here), so it returns all available rows with headers mapped to fields.
- **Key fields expected in output:**
  - `Site URL`
  - `Offboarded Links`
  - `row_number` (provided by n8n Sheets node when enabled/read-only)
- **Connections:**  
  - **Input ←** Run Workflow Manually  
  - **Output →** Filter Unprocessed PBN Rows
- **Credentials:** Google Sheets OAuth2 (`Prakash @iwantonlinemarketing.com`)
- **Edge cases / failures:**
  - OAuth token expired/invalid
  - Spreadsheet permission issues
  - Sheet headers changed (field names like `Site URL` must match downstream usage)
- **Version notes:** TypeVersion 4.7

---

### 2.2 Row Filtering (Resume Capability)
**Overview:** Finds the first PBN row that has not yet been processed (blank “Offboarded Links”) and returns that row plus all subsequent rows.

**Nodes Involved**
- Filter Unprocessed PBN Rows

#### Node: Filter Unprocessed PBN Rows
- **Type / Role:** Code (`n8n-nodes-base.code`) – custom filtering logic
- **Configuration (interpreted):**
  - Converts incoming items to `rows`
  - Locates the **first** row where `Offboarded Links` is `""`, `null`, or `undefined`
  - If none found, returns `[]` (workflow effectively stops—no items to process)
  - Returns `rows.slice(startIndex)` to continue from the first unprocessed onward
- **Key expressions / variables:**
  - Checks `row["Offboarded Links"]`
  - Outputs each remaining row as `{ json: row }`
- **Connections:**  
  - **Input ←** Read PBN Sites from Sheet  
  - **Output →** Loop Through Each PBN
- **Edge cases / failures:**
  - If “Offboarded Links” column name changes or has trailing spaces, filter will fail and may treat everything as “processed” or “unprocessed”.
  - If sheet contains blank rows / malformed objects, `row["Offboarded Links"]` may behave unexpectedly (handled as `undefined` → treated as unprocessed).
- **Version notes:** TypeVersion 2

---

### 2.3 Per-PBN Loop: Fetch HTML + Load Offboarded Domains
**Overview:** Processes PBN rows sequentially, fetches each PBN’s HTML, and loads the list of offboarded domains from a separate sheet for matching.

**Nodes Involved**
- Loop Through Each PBN
- Fetch PBN Site HTML
- Read Offboarded Project Domains

#### Node: Loop Through Each PBN
- **Type / Role:** Split In Batches (`n8n-nodes-base.splitInBatches`) – loop controller
- **Configuration (interpreted):**
  - Batch size is not explicitly set in JSON; n8n default is commonly **1** for “Split in Batches” unless configured otherwise.
  - Workflow wiring indicates iterative looping: after each iteration, it returns here to fetch the next item.
- **Connections:**  
  - **Input ←** Filter Unprocessed PBN Rows  
  - **Output (loop iteration) →** Fetch PBN Site HTML (wired from the node’s second main output index in this workflow)
  - **Input (continue) ←** Pause Before Next Iteration
- **Edge cases / failures:**
  - If batch size is >1, downstream logic that assumes “current row” via `$('Loop Through Each PBN').first()` may become ambiguous.
- **Version notes:** TypeVersion 3

#### Node: Fetch PBN Site HTML
- **Type / Role:** HTTP Request (`n8n-nodes-base.httpRequest`) – retrieve live site HTML
- **Configuration (interpreted):**
  - **URL:** `=https://{{ $json['Site URL'] }}`
    - The sheet value must be a hostname or domain without protocol; otherwise you risk `https://https://...` or malformed URLs.
  - **Retry on fail:** enabled
  - **Wait between tries:** 3000 ms
  - **Error handling:** `onError: continueRegularOutput` (workflow continues even if request fails)
- **Connections:**  
  - **Input ←** Loop Through Each PBN  
  - **Output →** Read Offboarded Project Domains
- **Output expectations:** The node’s response body is later referenced as `$('Fetch PBN Site HTML').first().json.data`
- **Edge cases / failures:**
  - DNS/timeout/5xx responses (will retry; then continue with output even on error)
  - Non-HTML responses, compressed bodies, very large pages
  - If request errors, `json.data` may be missing/empty → matching step can throw unless handled
- **Version notes:** TypeVersion 4.2

#### Node: Read Offboarded Project Domains
- **Type / Role:** Google Sheets (`n8n-nodes-base.googleSheets`) – loads offboarded domain list
- **Configuration (interpreted):**
  - Spreadsheet: same document as above
  - Sheet tab: `offboard projects` (gid `0`)
  - Range: `A1:A` explicitly
  - Expects a column header like `Website URLs` (used downstream)
- **Connections:**  
  - **Input ←** Fetch PBN Site HTML  
  - **Output →** Match Domains in HTML
- **Credentials:** Google Sheets OAuth2 (`Prakash @iwantonlinemarketing.com`)
- **Edge cases / failures:**
  - If column A does not have header `Website URLs`, downstream code won’t parse domains correctly.
  - If values are not valid URLs (e.g., bare domains), the code attempts `new URL(...)` and will drop invalid entries.
- **Version notes:** TypeVersion 4.7

---

### 2.4 Matching & Writing Result + Throttling
**Overview:** Extracts hostnames from offboarded URLs, checks whether any appear in the PBN HTML, writes the result into the correct PBN row, then waits briefly before continuing.

**Nodes Involved**
- Match Domains in HTML
- Prepare Row Update Payload
- Write Matched Domain to PBNs Sheet
- Pause Before Next Iteration

#### Node: Match Domains in HTML
- **Type / Role:** Code (`n8n-nodes-base.code`) – domain extraction + substring match
- **Configuration (interpreted):**
  - Reads all input items (offboard projects rows)
  - For each item, attempts `new URL(item.json["Website URLs"])`
    - Extracts `hostname`, strips leading `www.`
  - Gets HTML via: `$('Fetch PBN Site HTML').first().json.data`
  - Performs `htmlData.includes(domain)` for each domain
  - Outputs:
    - One item per matched domain as `{ matched_domain: domain }`
    - If none, outputs `{ matched_domain: 0 }`
- **Connections:**  
  - **Input ←** Read Offboarded Project Domains  
  - **Output →** Prepare Row Update Payload
- **Edge cases / failures:**
  - If the HTTP node failed and `data` is undefined, `htmlData.includes(...)` will throw. (No guard exists.)
  - If offboarded list contains bare domains like `example.com` (not `https://example.com`), `new URL()` throws and the domain is ignored → false negatives.
  - Substring matching can cause false positives (e.g., `ample.com` inside `example.com`), and misses if content is encoded/obfuscated.
- **Version notes:** TypeVersion 2

#### Node: Prepare Row Update Payload
- **Type / Role:** Set (`n8n-nodes-base.set`) – shape data for the update
- **Configuration (interpreted):**
  - Creates fields:
    - `url` = `$('Loop Through Each PBN').first().json["row_number"]`
    - `mach` = current `$json.matched_domain`
    - `HTML` = `$('Fetch PBN Site HTML').first().json["data"]` (prepared but not written by update mapping)
- **Connections:**  
  - **Input ←** Match Domains in HTML  
  - **Output →** Write Matched Domain to PBNs Sheet
- **Edge cases / failures:**
  - If multiple matched domains are output, this node runs once per match, causing **multiple updates** to the same row; the final value in “Offboarded Links” may end up being the last matched domain processed (non-deterministic ordering).
  - `row_number` must exist from the initial sheet read; if missing, update will fail to match.
- **Version notes:** TypeVersion 3.4

#### Node: Write Matched Domain to PBNs Sheet
- **Type / Role:** Google Sheets (`n8n-nodes-base.googleSheets`) – update row with match result
- **Configuration (interpreted):**
  - **Operation:** Update
  - **Target sheet:** `PBNs` (gid `879625869`)
  - **Matching column:** `row_number`
  - Updates:
    - `row_number` = `{{$json.url}}` (used for match)
    - `Offboarded Links` = `{{$json.mach}}`
  - `alwaysOutputData: true` ensures an output item continues even if update returns minimal data.
- **Connections:**  
  - **Input ←** Prepare Row Update Payload  
  - **Output →** Pause Before Next Iteration
- **Credentials:** Google Sheets OAuth2 (`Prakash @iwantonlinemarketing.com`)
- **Edge cases / failures:**
  - If `row_number` mismatches or is missing, update may not affect intended row.
  - If sheet is protected or user lacks edit permissions, update fails.
- **Version notes:** TypeVersion 4.7

#### Node: Pause Before Next Iteration
- **Type / Role:** Wait (`n8n-nodes-base.wait`) – throttle requests / pacing
- **Configuration (interpreted):**
  - Wait node is present with default settings (no explicit duration in JSON).
  - Used to pause before returning to the batch loop.
- **Connections:**  
  - **Input ←** Write Matched Domain to PBNs Sheet  
  - **Output →** Loop Through Each PBN (to continue with next item)
- **Edge cases / failures:**
  - If configured as “Wait for webhook” in UI, it may stall indefinitely unless resumed. (JSON doesn’t expose a duration here.)
- **Version notes:** TypeVersion 1.1

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Workflow Manually | manualTrigger | Manual entry point | — | Read PBN Sites from Sheet | ## PBN Offboarded Link Checker<br>This workflow automatically checks all PBN sites to detect if any offboarded client domain is still linked on them... (full note content applies to whole workflow) |
| Read PBN Sites from Sheet | googleSheets | Read PBN list from “PBNs” sheet | Run Workflow Manually | Filter Unprocessed PBN Rows | ## Data Fetching<br>Reads all PBN rows from Google Sheets, then filters out already-processed rows so only new/unchecked PBNs move forward. |
| Filter Unprocessed PBN Rows | code | Skip processed rows; resume from first blank “Offboarded Links” | Read PBN Sites from Sheet | Loop Through Each PBN | ## Data Fetching<br>Reads all PBN rows from Google Sheets, then filters out already-processed rows so only new/unchecked PBNs move forward. |
| Loop Through Each PBN | splitInBatches | Iterate through PBN rows sequentially | Filter Unprocessed PBN Rows; Pause Before Next Iteration | Fetch PBN Site HTML | ## HTML Fetch & Domain Matching<br>Loops through each PBN one by one, fetches its live HTML, then checks if any offboarded client domain is present inside that HTML. |
| Fetch PBN Site HTML | httpRequest | Download live HTML for current PBN site | Loop Through Each PBN | Read Offboarded Project Domains | ## HTML Fetch & Domain Matching<br>Loops through each PBN one by one, fetches its live HTML, then checks if any offboarded client domain is present inside that HTML. |
| Read Offboarded Project Domains | googleSheets | Load offboarded domain list from separate sheet | Fetch PBN Site HTML | Match Domains in HTML | ## HTML Fetch & Domain Matching<br>Loops through each PBN one by one, fetches its live HTML, then checks if any offboarded client domain is present inside that HTML. |
| Match Domains in HTML | code | Extract hostnames; substring match against HTML | Read Offboarded Project Domains | Prepare Row Update Payload | ## HTML Fetch & Domain Matching<br>Loops through each PBN one by one, fetches its live HTML, then checks if any offboarded client domain is present inside that HTML. |
| Prepare Row Update Payload | set | Build fields for sheet update (row_number + match) | Match Domains in HTML | Write Matched Domain to PBNs Sheet | ## Update & Loop Back<br>Structures the matched result, writes it into the PBNs sheet, then pauses briefly before moving to the next PBN in the loop. |
| Write Matched Domain to PBNs Sheet | googleSheets | Update “Offboarded Links” for the current PBN row | Prepare Row Update Payload | Pause Before Next Iteration | ## Update & Loop Back<br>Structures the matched result, writes it into the PBNs sheet, then pauses briefly before moving to the next PBN in the loop. |
| Pause Before Next Iteration | wait | Throttle / pacing before next loop iteration | Write Matched Domain to PBNs Sheet | Loop Through Each PBN | ## Update & Loop Back<br>Structures the matched result, writes it into the PBNs sheet, then pauses briefly before moving to the next PBN in the loop. |
| Sticky Note | stickyNote | Documentation | — | — | ## PBN Offboarded Link Checker<br>(Contains workflow purpose, how it works, setup steps) |
| Sticky Note1 | stickyNote | Documentation for data fetching block | — | — | ## Data Fetching<br>(Explains sheet read + filtering) |
| Sticky Note2 | stickyNote | Documentation for fetch/match block | — | — | ## HTML Fetch & Domain Matching<br>(Explains loop + HTML + match) |
| Sticky Note3 | stickyNote | Documentation for update/loop block | — | — | ## Update & Loop Back<br>(Explains update + wait + continue) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Manual Trigger**
   - Add node: **Manual Trigger**
   - Name: `Run Workflow Manually`

2) **Add Google Sheets node to read PBNs**
   - Add node: **Google Sheets**
   - Name: `Read PBN Sites from Sheet`
   - Credentials: configure **Google Sheets OAuth2** (sign in to the Google account that can access the spreadsheet)
   - Document: paste the spreadsheet URL containing both tabs
   - Sheet: select tab **PBNs**
   - Operation: **Read/Get many** (default “read rows” behavior)
   - Ensure your PBNs sheet has at least these columns:
     - `Site URL`
     - `Offboarded Links`
   - Connect: `Run Workflow Manually` → `Read PBN Sites from Sheet`

3) **Add Code node to filter unprocessed rows**
   - Add node: **Code**
   - Name: `Filter Unprocessed PBN Rows`
   - Paste logic (configure equivalently):
     - Find first row where `Offboarded Links` is blank/null/undefined
     - Return from that row onward; if none, return empty list
   - Connect: `Read PBN Sites from Sheet` → `Filter Unprocessed PBN Rows`

4) **Add Split in Batches to loop rows**
   - Add node: **Split In Batches**
   - Name: `Loop Through Each PBN`
   - Set **Batch Size = 1** (recommended to match how the workflow references `.first()`)
   - Connect: `Filter Unprocessed PBN Rows` → `Loop Through Each PBN`

5) **Add HTTP Request to fetch HTML**
   - Add node: **HTTP Request**
   - Name: `Fetch PBN Site HTML`
   - Method: GET
   - URL expression: `https://{{$json["Site URL"]}}`
     - (If your sheet already contains full URLs with protocol, use `{{$json["Site URL"]}}` instead.)
   - Enable **Retry on fail**, set **Wait Between Tries = 3000ms**
   - Set **On Error** to **Continue (regular output)**
   - Connect: `Loop Through Each PBN` → `Fetch PBN Site HTML`

6) **Add Google Sheets node to read offboarded domains**
   - Add node: **Google Sheets**
   - Name: `Read Offboarded Project Domains`
   - Same credentials as step 2
   - Same document (spreadsheet)
   - Sheet: select tab **offboard projects**
   - Configure range: `A1:A`
   - Ensure the sheet has a header that matches what you’ll parse (in the provided workflow it expects `Website URLs`), and values in that column are **valid URLs** like `https://clientdomain.com/`.
   - Connect: `Fetch PBN Site HTML` → `Read Offboarded Project Domains`

7) **Add Code node to match domains inside HTML**
   - Add node: **Code**
   - Name: `Match Domains in HTML`
   - Implement:
     - Parse each row’s `Website URLs` into `hostname` (strip `www.`)
     - Load HTML from `Fetch PBN Site HTML` node output (`.json.data`)
     - For each domain, check `includes(domain)`
     - Output one item per match; if none output `matched_domain = 0`
   - Connect: `Read Offboarded Project Domains` → `Match Domains in HTML`

8) **Add Set node to prepare the update payload**
   - Add node: **Set**
   - Name: `Prepare Row Update Payload`
   - Add fields:
     - `url` (string/number): value from current PBN row’s `row_number`
       - Expression: `{{$("Loop Through Each PBN").first().json["row_number"]}}`
     - `mach` (string): `{{$json.matched_domain}}`
   - (Optional) Keep an `HTML` field if you later want to store HTML somewhere, but note the provided sheet-update does not write it.
   - Connect: `Match Domains in HTML` → `Prepare Row Update Payload`

9) **Add Google Sheets update node to write result**
   - Add node: **Google Sheets**
   - Name: `Write Matched Domain to PBNs Sheet`
   - Credentials: same Google OAuth2
   - Document: same spreadsheet
   - Sheet: **PBNs**
   - Operation: **Update**
   - Matching column: `row_number`
   - Set updated columns:
     - `row_number` = `{{$json.url}}`
     - `Offboarded Links` = `{{$json.mach}}`
   - Connect: `Prepare Row Update Payload` → `Write Matched Domain to PBNs Sheet`

10) **Add Wait node for pacing**
   - Add node: **Wait**
   - Name: `Pause Before Next Iteration`
   - Configure a duration appropriate for your environment (for example, a few seconds) to reduce request bursts.
   - Connect: `Write Matched Domain to PBNs Sheet` → `Pause Before Next Iteration`

11) **Close the loop**
   - Connect: `Pause Before Next Iteration` → `Loop Through Each PBN`
   - This continues until Split In Batches has no items left.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **PBN Offboarded Link Checker**: Reads PBNs from “PBNs” sheet, skips processed rows, fetches HTML, compares with offboard project list, writes matched domain to “Offboarded Links”, repeats row-by-row. Setup: connect Google credentials, set sheet URL/tab names, ensure offboard projects domains in column A, run manually. | From workflow sticky note (applies to overall design) |
| Data Fetching: reads PBN rows then filters already-processed rows. | From “Data Fetching” sticky note |
| HTML Fetch & Domain Matching: loops PBNs, fetches HTML, checks offboarded domains. | From “HTML Fetch & Domain Matching” sticky note |
| Update & Loop Back: structures result, writes to sheet, pauses, then continues loop. | From “Update & Loop Back” sticky note |