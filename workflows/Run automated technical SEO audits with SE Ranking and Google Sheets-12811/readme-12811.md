Run automated technical SEO audits with SE Ranking and Google Sheets

https://n8nworkflows.xyz/workflows/run-automated-technical-seo-audits-with-se-ranking-and-google-sheets-12811


# Run automated technical SEO audits with SE Ranking and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically run a *technical SEO website audit* via **SE Ranking**, poll until the crawl is finished, retrieve the report and the most recent audit history (last 10), then export the history into **Google Sheets** for tracking and trend analysis.

**Typical use cases:**
- SEO agencies auditing client websites on a schedule
- Developers performing pre-launch checks
- Site owners monitoring technical health over time

### 1.1 Execution & Audit Creation
Starts manually, then creates a “standard” SE Ranking Website Audit for a target domain.

### 1.2 Crawl Waiting & Status Polling (Retry Loop)
Waits ~5 minutes, checks audit status, and if not finished waits 2 more minutes and checks again (loop).

### 1.3 Report Retrieval & Audit History Fetch
Once finished, fetches the full audit report and then lists the last 10 audits for the same domain.

### 1.4 Data Shaping & Google Sheets Export
Formats audit history into tabular rows and appends/updates them into a chosen Google Sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Execution & Audit Creation
**Overview:** Manually triggers the workflow and creates a new SE Ranking “standard” website audit for the configured domain.

**Nodes involved:**
- `When clicking 'Execute workflow'`
- `Create standard audit`

#### Node: When clicking 'Execute workflow'
- **Type / role:** Manual Trigger (entry point)
- **Configuration:** No parameters; starts on demand.
- **Outputs:** One execution item to `Create standard audit`.
- **Failure modes:** None (except workflow not executable / permissions).

#### Node: Create standard audit
- **Type / role:** SE Ranking node (`@seranking/n8n-nodes-seranking.seRanking`) — starts an audit job.
- **Configuration choices (interpreted):**
  - **Resource:** `websiteAudit`
  - **Operation:** `createStandard`
  - **Domain:** `example.com` (must be replaced with your real domain)
  - **Title:** “Weekly Technical SEO Audit”
- **Key values produced:** `id` (audit id), and `domain` (used later for searching history).
- **Connections:**
  - **Input:** Manual Trigger
  - **Output:** `Wait for crawl (5 min)`
- **Credentials:** `seRankingApi` required (API key/secret as provided by SE Ranking node setup).
- **Edge cases / failures:**
  - Invalid credentials or insufficient API permissions
  - Domain format mismatch (e.g., needs exact expected format by SE Ranking)
  - API rate limits / transient SE Ranking outages

---

### Block 2 — Crawl Waiting & Status Polling (Retry Loop)
**Overview:** Gives SE Ranking time to crawl the website, then checks audit status. If not finished, waits and retries until the status becomes `finished`.

**Nodes involved:**
- `Wait for crawl (5 min)`
- `Check audit status`
- `If audit complete`
- `Wait and retry (2 min)`

#### Node: Wait for crawl (5 min)
- **Type / role:** Wait node — pauses execution.
- **Configuration choices:**
  - Unit: `minutes`
  - Amount: (not explicitly set in JSON, but node name indicates 5 minutes; in n8n Wait node, leaving amount blank can be ambiguous—verify it’s set to 5 in UI)
  - Uses a `webhookId` internally for resuming execution.
- **Connections:**
  - **Input:** `Create standard audit`
  - **Output:** `Check audit status`
- **Failure modes:**
  - If amount is not set correctly, wait duration may not match expectations
  - Execution timeouts depending on n8n hosting limits (rare for Wait node, but platform constraints can apply)

#### Node: Check audit status
- **Type / role:** SE Ranking node — fetches status for a specific audit id.
- **Configuration choices:**
  - **Resource:** `websiteAudit`
  - **Operation:** `getStatus`
  - **Audit ID expression:** `{{ $('Create standard audit').item.json.id }}`
    - This explicitly references the output of *Create standard audit*.
- **Connections:**
  - **Input:** `Wait for crawl (5 min)` and also `Wait and retry (2 min)` (loop back)
  - **Output:** `If audit complete`
- **Credentials:** `seRankingApi`
- **Edge cases / failures:**
  - Expression failure if `Create standard audit` did not return an `id`
  - SE Ranking returns intermediate statuses (not `finished`) indefinitely (crawl stuck)
  - Rate limiting if retries happen too frequently

#### Node: If audit complete
- **Type / role:** IF node — branching based on audit status.
- **Condition:** `$json.status === "finished"`
- **Outputs:**
  - **True branch →** `Get audit report`
  - **False branch →** `Wait and retry (2 min)`
- **Edge cases:**
  - Status field absent or different naming (would route to false branch forever)
  - Status values could include other “terminal” states (e.g., “failed”, “canceled”) not handled here

#### Node: Wait and retry (2 min)
- **Type / role:** Wait node — pause before retry.
- **Configuration:** 2 minutes.
- **Connections:**
  - **Input:** False branch of `If audit complete`
  - **Output:** `Check audit status` (creates the retry loop)
- **Failure modes:**
  - Infinite loop risk if audit never reaches `finished`
  - Accumulated executions if triggered frequently without guardrails

---

### Block 3 — Report Retrieval & Audit History Fetch
**Overview:** Once the audit is finished, retrieves the full audit report (health score etc.) and then fetches the last 10 audits for the same domain for history/trends.

**Nodes involved:**
- `Get audit report`
- `List all audits for domain`

#### Node: Get audit report
- **Type / role:** SE Ranking node — fetches detailed audit results.
- **Configuration choices:**
  - **Resource:** `websiteAudit`
  - **Operation:** `getReport`
  - **Audit ID expression:** `{{ $('Create standard audit').item.json.id }}`
- **Connections:**
  - **Input:** True branch of `If audit complete`
  - **Output:** `List all audits for domain`
- **Credentials:** `seRankingApi`
- **Edge cases / failures:**
  - Report not ready even if status says finished (timing inconsistency)
  - Large reports may be heavy; API latency/timeouts possible

#### Node: List all audits for domain
- **Type / role:** SE Ranking node — queries audit list (history).
- **Configuration choices:**
  - **Resource:** `websiteAudit`
  - Operation is implicitly “list/get many” via additional fields (as configured by the node)
  - **Limit:** 10
  - **Search expression:** `{{ $('Create standard audit').item.json.domain }}`
    - Uses the domain returned from creation (not necessarily the same as the literal input).
- **Connections:**
  - **Input:** `Get audit report`
  - **Output:** `Format audits for Sheet`
- **Credentials:** `seRankingApi`
- **Edge cases / failures:**
  - Search may not match if SE Ranking stores URLs with scheme (`https://...`) or normalized host differently
  - Returned shape may be `items: []` or a single object; downstream Code node accounts for both

---

### Block 4 — Data Shaping & Google Sheets Export
**Overview:** Transforms the audit history objects into flat rows and writes them to Google Sheets using append-or-update logic.

**Nodes involved:**
- `Format audits for Sheet`
- `Export to Google Sheets`

#### Node: Format audits for Sheet
- **Type / role:** Code node (JavaScript) — normalize API output for sheet rows.
- **Core logic:**
  - Reads all incoming items: `const items = $input.all();`
  - For each item:
    - Uses `item.json.items` if present (list response), otherwise wraps `item.json` as a single audit
  - Produces one output item per audit with fields:
    - `audit_id`
    - `domain` (strips `http://` or `https://` from `audit.url`)
    - `title`, `last_update`, `status`
    - `health_score`, `errors`, `warnings`, `notices`, `pages_crawled` from `audit.stats.*` (defaulting to `0`)
- **Connections:**
  - **Input:** `List all audits for domain`
  - **Output:** `Export to Google Sheets`
- **Edge cases / failures:**
  - If SE Ranking response structure changes (e.g., `stats` missing or renamed)
  - `audit.url` missing → domain becomes `''`
  - If incoming list is nested differently than `items`, output could be empty

#### Node: Export to Google Sheets
- **Type / role:** Google Sheets node — writes audit rows.
- **Operation:** `appendOrUpdate`
- **Sheet/document selection:** Both `documentId` and `sheetName` are **empty** in the JSON and must be set in the UI.
- **Columns schema:** Predefined columns for the fields produced by the Code node:
  - `audit_id, domain, title, last_update, status, health_score, errors, warnings, notices, pages_crawled`
- **Mapping mode:** Auto-map input data to columns.
- **Credentials:** `googleSheetsOAuth2Api` (Google OAuth2)
- **Connections:**
  - **Input:** `Format audits for Sheet`
  - **Output:** none
- **Edge cases / failures:**
  - Missing/invalid Google OAuth credentials
  - Spreadsheet not shared with the authenticated Google account
  - “Append or update” requires correct matching configuration; here `matchingColumns` is empty, so updates may not behave as expected (often it will behave like append). Consider setting `audit_id` as a matching column.
  - Data type conversions disabled; numeric values may be inserted as strings depending on Google Sheets behavior and node settings

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documentation / context | — | — | ## Run automated technical SEO audits… (includes setup steps + link: https://www.npmjs.com/package/@seranking/n8n-nodes-seranking) |
| Step 1 | Sticky Note | Section label | — | — | ### 🔍 Run Audit & Wait |
| Step 2 | Sticky Note | Section label | — | — | ### 📊 Get Report & Last 10 reports |
| Step 3 | Sticky Note | Section label | — | — | ### 📤 Export to Sheets |
| When clicking 'Execute workflow' | Manual Trigger | Entry point | — | Create standard audit | ### 🔍 Run Audit & Wait |
| Create standard audit | SE Ranking | Start a standard website audit | When clicking 'Execute workflow' | Wait for crawl (5 min) | ### 🔍 Run Audit & Wait |
| Wait for crawl (5 min) | Wait | Pause before checking status | Create standard audit | Check audit status | ### 🔍 Run Audit & Wait |
| Check audit status | SE Ranking | Poll audit status | Wait for crawl (5 min); Wait and retry (2 min) | If audit complete | ### 🔍 Run Audit & Wait |
| If audit complete | IF | Branch on `status === finished` | Check audit status | Get audit report; Wait and retry (2 min) | ### 🔍 Run Audit & Wait |
| Wait and retry (2 min) | Wait | Delay before retry loop | If audit complete (false) | Check audit status | ### 🔍 Run Audit & Wait |
| Get audit report | SE Ranking | Fetch completed audit report | If audit complete (true) | List all audits for domain | ### 📊 Get Report & Last 10 reports |
| List all audits for domain | SE Ranking | Fetch last 10 audits for domain | Get audit report | Format audits for Sheet | ### 📊 Get Report & Last 10 reports |
| Format audits for Sheet | Code | Normalize audits into sheet rows | List all audits for domain | Export to Google Sheets | ### 📤 Export to Sheets |
| Export to Google Sheets | Google Sheets | Append/update rows in spreadsheet | Format audits for Sheet | — | ### 📤 Export to Sheets |

---

## 4. Reproducing the Workflow from Scratch

1. **Install the SE Ranking community node**
   - In n8n, install: `@seranking/n8n-nodes-seranking`  
   - Restart n8n if required.

2. **Create credentials**
   - **SE Ranking API credential**
     - Add the SE Ranking API credential in n8n (per node’s credential type).
   - **Google Sheets OAuth2 credential**
     - Create/enable a Google Cloud project, enable Google Sheets API, configure OAuth consent, create OAuth client, then add the OAuth2 credential in n8n.
     - Ensure the Google account has access to the target spreadsheet.

3. **Create nodes (in order) and configure them**
   1) **Manual Trigger** node  
      - Name: `When clicking 'Execute workflow'`

   2) **SE Ranking** node  
      - Name: `Create standard audit`  
      - Resource: `Website Audit`  
      - Operation: `Create Standard`  
      - Domain: replace `example.com` with your domain (e.g., `yourdomain.com`)  
      - Additional field: Title = `Weekly Technical SEO Audit`  
      - Select **SE Ranking API** credentials

   3) **Wait** node  
      - Name: `Wait for crawl (5 min)`  
      - Unit: Minutes  
      - Amount: 5

   4) **SE Ranking** node  
      - Name: `Check audit status`  
      - Resource: `Website Audit`  
      - Operation: `Get Status`  
      - Audit ID: expression referencing created audit id:  
        - `{{ $('Create standard audit').item.json.id }}`
      - Credentials: SE Ranking API

   5) **IF** node  
      - Name: `If audit complete`  
      - Condition (String):  
        - Left: `{{ $json.status }}`  
        - Operation: `equals`  
        - Right: `finished`

   6) **Wait** node  
      - Name: `Wait and retry (2 min)`  
      - Unit: Minutes  
      - Amount: 2

   7) **SE Ranking** node  
      - Name: `Get audit report`  
      - Resource: `Website Audit`  
      - Operation: `Get Report`  
      - Audit ID: `{{ $('Create standard audit').item.json.id }}`  
      - Credentials: SE Ranking API

   8) **SE Ranking** node  
      - Name: `List all audits for domain`  
      - Resource: `Website Audit`  
      - Operation: (list/search operation as provided by the node UI)  
      - Additional fields:
        - Limit: 10  
        - Search: `{{ $('Create standard audit').item.json.domain }}`
      - Credentials: SE Ranking API

   9) **Code** node  
      - Name: `Format audits for Sheet`  
      - Paste the JavaScript formatting logic (adapt field names only if SE Ranking API response differs).

   10) **Google Sheets** node  
      - Name: `Export to Google Sheets`  
      - Operation: `Append or Update`  
      - Select `Document` (Spreadsheet) and `Sheet` in the UI  
      - Keep Auto-map enabled  
      - (Recommended) Set matching column to `audit_id` if you want updates instead of repeated appends  
      - Credentials: Google Sheets OAuth2

4. **Connect nodes (exact wiring)**
   - Manual Trigger → Create standard audit → Wait for crawl (5 min) → Check audit status → If audit complete
   - If audit complete **true** → Get audit report → List all audits for domain → Format audits for Sheet → Export to Google Sheets
   - If audit complete **false** → Wait and retry (2 min) → Check audit status (loop)

5. **Validate**
   - Run once manually.
   - Confirm SE Ranking returns `status: finished` eventually.
   - Confirm Google Sheets writes rows and columns match expected names.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install SE Ranking node package | https://www.npmjs.com/package/@seranking/n8n-nodes-seranking |
| Replace `example.com` with your domain | In `Create standard audit` node |
| Optional: add a Schedule Trigger for weekly runs | Replace Manual Trigger with Schedule Trigger |
| Adjust wait times for large sites | Increase initial wait and/or retry delay |
| Workflow includes “last 10 audits” trend tracking | `List all audits for domain` uses limit=10 |
| Google Sheets export requires selecting document + sheet | `Export to Google Sheets` has empty documentId/sheetName in JSON |

