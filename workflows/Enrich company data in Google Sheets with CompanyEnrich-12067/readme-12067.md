Enrich company data in Google Sheets with CompanyEnrich

https://n8nworkflows.xyz/workflows/enrich-company-data-in-google-sheets-with-companyenrich-12067


# Enrich company data in Google Sheets with CompanyEnrich

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow enriches rows in a Google Sheet with firmographic/company data pulled from the **CompanyEnrich** API, using a company **domain** as the lookup key. It avoids re-processing rows already marked as completed and updates only the columns that exist in the sheet.

**Typical use cases:**
- Enrich lead/account lists with company name, revenue, employee count, location, LinkedIn URL, logo URL, etc.
- Keep a spreadsheet “self-updating” by re-running enrichment for new/unprocessed domains.

### Logical blocks
**1.1 Input Reception & Validation**
- Triggered manually
- Reads rows from Google Sheets
- Filters out rows whose `Status` is `Done` to save API credits

**1.2 Batch Looping**
- Iterates through remaining rows one-by-one (batch node)
- Provides looping control so each domain is enriched and then the next item is processed

**1.3 Enrichment (API Call)**
- Calls CompanyEnrich `/companies/enrich` with `domain` as a query parameter
- Continues workflow even if the API request errors (so one bad domain won’t stop the run)

**1.4 Data Normalization & Column Mapping**
- Flattens nested JSON keys into underscore-separated columns (e.g. `location.country.name` → `location_country_name`)
- Converts arrays into comma-separated strings
- Filters the API payload to only columns that exist in the sheet headers
- Forces `Domain`, `Status`, and `Last Updated` outputs

**1.5 Update Back to Google Sheets**
- Updates the matching row using `Domain` as the matching key
- Writes the cleaned/enriched values into corresponding columns

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Validation
**Overview:** Reads all rows from a specified sheet and filters out those already processed (`Status = Done`). This protects API usage and prevents repeated updates.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Get row(s) in sheet
- Filter

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — starts execution on demand.
- **Configuration:** No parameters.
- **Inputs/Outputs:** No input; outputs one execution event into Google Sheets read node.
- **Edge cases:** None (other than manual execution context).

#### Node: Get row(s) in sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads rows from a spreadsheet.
- **Configuration (interpreted):**
- Uses OAuth2 credential: **googleSheetsOAuth2Api Credential**
- Targets a specific spreadsheet (document ID `1aZXmlXO...`) and sheet/tab `Sayfa1` (`gid=0`).
- Operation is implied by node name: “Get row(s)” (read rows).
- **Key data assumptions:**
- Sheet has a header row including at least: `Domain`, `Status`, `Last Updated`.
- Each output item represents one row; includes `Domain`, `Status`, and possibly `row_number` depending on Google Sheets node output.
- **Inputs/Outputs:**
- Input: Manual Trigger
- Output: Filter node
- **Edge cases / failures:**
- OAuth permission issues (invalid/expired token)
- Spreadsheet/tab renamed or permissions revoked
- Missing expected columns (especially `Domain` / `Status`), causing later expressions to fail or filter logic to behave unexpectedly
- Large sheets can slow executions (pagination/limits depend on node configuration)

#### Node: Filter
- **Type / role:** Filter (`n8n-nodes-base.filter`) — removes rows already marked complete.
- **Configuration (interpreted):**
- Condition: `{{$json.Status}}` **notEquals** `"Done"`.
- Strict validation enabled (type validation “strict”).
- **Inputs/Outputs:**
- Input: Get row(s) in sheet
- Output: Loop Over Items
- **Edge cases / failures:**
- If `Status` is missing or null, it is typically treated as “not equal Done” → row will pass (often desired).
- Case sensitivity is enabled; `done` won’t match `Done`, so it will be reprocessed unless standardized.

---

### Block 1.2 — Batch Looping
**Overview:** Iterates through filtered rows, allowing controlled per-row enrichment and update. This is the workflow’s looping backbone.

**Nodes involved:**
- Loop Over Items

#### Node: Loop Over Items
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — batch/loop controller.
- **Configuration (interpreted):**
- Batch size not explicitly set in parameters shown (defaults apply; in many n8n versions default is 1).
- **Connections (important):**
- Has **two outgoing connections**:
  1) To **Update row in sheet**
  2) To **Fetch Company Data**
- Also receives an incoming connection from **Data Cleaning (JS)** back to **Loop Over Items**, enabling iteration.
- **Inputs/Outputs:**
- Input: Filter
- Output: Both Update row in sheet and Fetch Company Data
- **Edge cases / workflow logic concern:**
- Because it outputs to **Update row in sheet** *and* **Fetch Company Data** in parallel, the update node may run **before** enrichment data exists (depending on execution order and data availability). In practice, `Update row in sheet` uses `{{$json.status_update}}` which is not produced by the code node (it produces `Status`), so this branch is likely miswired/misconfigured.
- The intended pattern in n8n is usually:
  `SplitInBatches → HTTP Request → Code (clean) → Google Sheets Update → SplitInBatches (next batch)`
- As currently connected, ensure you validate the update node’s input source and field mapping (see Block 1.5).

---

### Block 1.3 — Enrichment (API Call)
**Overview:** Calls CompanyEnrich to enrich each domain. Errors are tolerated so processing continues for other rows.

**Nodes involved:**
- Fetch Company Data

#### Node: Fetch Company Data
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls CompanyEnrich API.
- **Configuration (interpreted):**
- Method: not shown; with v4 HTTP Request, default is typically **GET** unless otherwise set.
- URL: `https://api.companyenrich.com/companies/enrich`
- Sends query parameters:
  - `domain = {{$json.Domain}}`
- Sends header:
  - `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder; must be replaced)
- `onError: continueRegularOutput`:
  - If the request fails (HTTP error, timeout), workflow continues and outputs an error-shaped item (depending on n8n’s HTTP Request behavior/version).
- **Inputs/Outputs:**
- Input: Loop Over Items (current row item; must include `Domain`)
- Output: Data Cleaning (JS)
- **Edge cases / failures:**
- 401/403 if token missing/invalid
- 429 rate limiting if too many calls
- 400 if domain invalid/empty
- Response schema variations (object vs array) are handled in the code node
- Network timeouts; consider adding retry/backoff in HTTP node options if needed

---

### Block 1.4 — Data Normalization & Column Mapping
**Overview:** Flattens API response, converts arrays, filters only columns present in the sheet, and injects required fields (`Domain`, `Status`, `Last Updated`) for updating.

**Nodes involved:**
- Data Cleaning (JS)

#### Node: Data Cleaning (JS)
- **Type / role:** Code (`n8n-nodes-base.code`) — transforms the HTTP response into an “update payload” matching sheet headers.
- **Configuration (interpreted):**
- Mode: `runOnceForEachItem` (each domain processed independently)
- Key logic:
  1) Reads API result from `$input.item.json`
  2) Reads the current sheet row from `$('Get row(s) in sheet').item.json`
     - Preserves `Domain` from the sheet row as `originalDomain` for matching
  3) If API returns an array, uses the first element
  4) `flattenObject()` converts:
     - nested objects to underscore keys
     - arrays to comma-separated strings (objects inside arrays are JSON-stringified)
     - null/undefined to `""`
  5) Filters flattened keys to only those whose column headers exist in the sheet (`allowedColumns`)
  6) Always sets:
     - `Domain = originalDomain`
     - `Status = "Done"`
     - `Last Updated = new Date().toISOString()`
- **Inputs/Outputs:**
- Input: Fetch Company Data
- Output: Loop Over Items (feeds next batch iteration)
- **Key expressions/variables used:**
- `$('Get row(s) in sheet').item.json` is a **cross-node item reference**; depending on n8n item linking, it may not always correspond to the same row currently being processed unless item pairing is maintained. Prefer using the current item’s domain directly when possible.
- **Edge cases / failures:**
- If `Get row(s) in sheet` output cannot be paired to the current loop item, `sheetData` may reference the wrong row.
- If sheet header names don’t match flattened keys exactly, those fields are silently dropped (by design).
- If API returns an error object, flattening may produce unexpected keys; the filtering step prevents most sheet pollution, but `Status` will still be forced to `Done` even if enrichment failed (unless you change logic).

---

### Block 1.5 — Update Back to Google Sheets
**Overview:** Updates the sheet row matching the company domain and writes enrichment data into corresponding columns.

**Nodes involved:**
- Update row in sheet

#### Node: Update row in sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — updates an existing row.
- **Configuration (interpreted):**
- Operation: **update**
- Matching column: `Domain`
- Mapping mode: `autoMapInputData`
- “Columns” mapping includes explicit expressions:
  - `Domain = {{ $('Get row(s) in sheet').item.json.Domain }}`
  - `Status = {{ $json.status_update }}`
  - `Last Updated = {{ $now.toISO() }}`
- Also has schema listing expected columns (name, industries, employees, revenue, etc.).
- Credential: **googleSheetsOAuth2Api Credential**
- **Inputs/Outputs:**
- Input: Loop Over Items (as currently wired)
- Output: none shown (end of this branch)
- **Critical configuration issue (likely bug):**
- The code node outputs `Status` (capital S) and `Last Updated`, but **this Update node uses** `{{$json.status_update}}`, which is not set anywhere. Result: Status may be written blank/undefined.
- The update node is connected directly from **Loop Over Items**, not from **Data Cleaning (JS)**. This means it may not receive the cleaned/enriched payload at all.
- **Recommended intended wiring:**
- `Loop Over Items → Fetch Company Data → Data Cleaning (JS) → Update row in sheet → Loop Over Items`
- **Edge cases / failures:**
- If the `Domain` value isn’t unique, multiple matches may behave unexpectedly (depending on Google Sheets node behavior).
- If `Domain` is blank, update will fail or update the wrong row.
- OAuth issues, sheet protection, or missing headers cause update failures.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual start | — | Get row(s) in sheet | ## 1. Input & Validation<br>Retrieves data from your spreadsheet and filters out processed rows to save API credits. |
| Get row(s) in sheet | Google Sheets | Read rows from spreadsheet | When clicking ‘Execute workflow’ | Filter | ## 1. Input & Validation<br>Retrieves data from your spreadsheet and filters out processed rows to save API credits. |
| Filter | Filter | Exclude rows with `Status = Done` | Get row(s) in sheet | Loop Over Items | ## 1. Input & Validation<br>Retrieves data from your spreadsheet and filters out processed rows to save API credits. |
| Loop Over Items | Split In Batches | Iterate rows | Filter; Data Cleaning (JS) | Update row in sheet; Fetch Company Data | ## 2. Enrichment & Update<br>Loops through domains, fetches data, flattens the JSON structure (e.g. `location.city.name` → `location_city_name`), and updates matching columns in Google Sheets. |
| Fetch Company Data | HTTP Request | Call CompanyEnrich API with domain | Loop Over Items | Data Cleaning (JS) | ## 2. Enrichment & Update<br>Loops through domains, fetches data, flattens the JSON structure (e.g. `location.city.name` → `location_city_name`), and updates matching columns in Google Sheets. |
| Data Cleaning (JS) | Code | Flatten/filter API response to sheet headers | Fetch Company Data | Loop Over Items | ## 2. Enrichment & Update<br>Loops through domains, fetches data, flattens the JSON structure (e.g. `location.city.name` → `location_city_name`), and updates matching columns in Google Sheets. |
| Update row in sheet | Google Sheets | Update row by Domain | Loop Over Items | — | ## 2. Enrichment & Update<br>Loops through domains, fetches data, flattens the JSON structure (e.g. `location.city.name.name` → `location_country_name`), and updates matching columns in Google Sheets. |
| Overview | Sticky Note | Documentation/overview | — | — | ## 🚀 Company Enrichment Workflow<br><br>This workflow automatically populates your Google Sheet with firmographic data based on a company's domain name.<br><br>## How it works<br>1. **Reads** domains from Google Sheets & **Filters** out completed rows.<br>2. **Fetches** data using the CompanyEnrich API.<br>3. **Cleans & Maps** the data to your specific Sheet headers.<br>4. **Updates** the row with new info.<br><br>## Setup steps<br>1. Create a Google Sheet with headers: `Domain`, `Status`, and `Last Updated`.<br>2. **Add columns for data you want.** The workflow matches API fields to your headers automatically. Examples:<br>   * `revenue`<br>   * `employees`<br>   * `socials_linkedin_url` (Nested fields use underscores)<br>   * `location_country_name`<br>3. Configure the nodes with your Sheet ID and API Key. |
| Input Group | Sticky Note | Section label | — | — | ## 1. Input & Validation<br>Retrieves data from your spreadsheet and filters out processed rows to save API credits. |
| Enrich Group | Sticky Note | Section label | — | — | ## 2. Enrichment & Update<br>Loops through domains, fetches data, flattens the JSON structure (e.g. `location.city.name` → `location_city_name`), and updates matching columns in Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Enrich Company Data in Google Sheets with CompanyEnrich**
- (Optional) Add sticky notes with the “Overview”, “Input Group”, and “Enrich Group” text.

2) **Add Manual Trigger**
- Node: **Manual Trigger**
- Name: `When clicking ‘Execute workflow’`

3) **Add Google Sheets (Read)**
- Node: **Google Sheets**
- Name: `Get row(s) in sheet`
- Credentials: create/select **Google Sheets OAuth2** credential with access to the target spreadsheet.
- Configure:
  - Resource/Operation: read/get rows (as per Google Sheets node UI)
  - Document: select your spreadsheet
  - Sheet/Tab: select your tab (e.g., `Sayfa1`)
- Ensure your sheet has headers at least:
  - `Domain`, `Status`, `Last Updated`
- Connect: Manual Trigger → Get row(s) in sheet

4) **Add Filter**
- Node: **Filter**
- Name: `Filter`
- Condition:
  - Left value: `={{ $json.Status }}`
  - Operator: `String → not equals`
  - Right value: `Done`
- Connect: Get row(s) in sheet → Filter

5) **Add Split In Batches**
- Node: **Split In Batches**
- Name: `Loop Over Items`
- Set batch size (recommended): **1** (or keep default)
- Connect: Filter → Loop Over Items

6) **Add HTTP Request (CompanyEnrich)**
- Node: **HTTP Request**
- Name: `Fetch Company Data`
- Configure:
  - Method: **GET** (or whatever CompanyEnrich requires for this endpoint)
  - URL: `https://api.companyenrich.com/companies/enrich`
  - Query parameter:
    - `domain` = `={{ $json.Domain }}`
  - Header:
    - `Authorization` = `Bearer <YOUR_COMPANYENRICH_API_TOKEN>`
  - Error handling:
    - Set **On Error** to “Continue” (equivalent to `continueRegularOutput`)
- Connect: Loop Over Items → Fetch Company Data

7) **Add Code node (clean/flatten/filter)**
- Node: **Code**
- Name: `Data Cleaning (JS)`
- Mode: **Run Once for Each Item**
- Paste the logic (adapted as needed). Ensure:
  - You output keys matching your sheet headers
  - You set:
    - `Domain`
    - `Status` (e.g. `Done`, or `Error` when API failed)
    - `Last Updated`
- Connect: Fetch Company Data → Data Cleaning (JS)

8) **Add Google Sheets (Update)**
- Node: **Google Sheets**
- Name: `Update row in sheet`
- Credentials: same Google Sheets OAuth2 credential
- Configure:
  - Operation: **Update**
  - Document: same spreadsheet
  - Sheet: same tab
  - Matching columns: `Domain`
  - Mapping: **Auto-map input data**
- Important adjustments (to align with the Code output):
  - Map `Status` from `={{ $json.Status }}` (not `status_update`)
  - Map `Last Updated` from `={{ $json["Last Updated"] }}` or compute `={{ $now.toISO() }}` (either is fine, but be consistent)
  - For `Domain`, use `={{ $json.Domain }}` (prefer current item output from Code)
- Connect: Data Cleaning (JS) → Update row in sheet

9) **Close the loop**
- Connect: Update row in sheet → Loop Over Items  
  This makes Split In Batches proceed to the next item.

10) **Test**
- Put a few rows with `Domain` filled and `Status` empty.
- Execute workflow manually.
- Verify:
  - Rows become `Status = Done`
  - Enrichment fields populate where columns exist.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically populates your Google Sheet with firmographic data based on a company's domain name. | Sticky note: “Overview” |
| Create a Google Sheet with headers: `Domain`, `Status`, and `Last Updated`. Add enrichment columns you want; nested fields use underscores (e.g. `socials_linkedin_url`, `location_country_name`). | Sticky note: “Overview” |
| Replace `Bearer YOUR_TOKEN_HERE` with your real CompanyEnrich API token. | HTTP Request node configuration |
| Current wiring/mapping likely needs correction: Update should consume the Code node output; `status_update` is not produced anywhere (use `Status`). | Observed from node connections + expressions |