Extract employee emails from company domains with BrowserAct and Google Sheets

https://n8nworkflows.xyz/workflows/extract-employee-emails-from-company-domains-with-browseract-and-google-sheets-12362


# Extract employee emails from company domains with BrowserAct and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow iterates over a list of company domains stored in Google Sheets, triggers a BrowserAct workflow to extract employee email leads (via Hunter.io logic inside BrowserAct), parses the returned data, and archives the extracted contacts into a newly created per-company results sheet. It also handles human verification (CAPTCHA/security checks) by waiting and retrying, and notifies completion via Slack.

**Primary use cases:**
- Batch enrichment of company domains into employee contact leads.
- Human-in-the-loop scraping where automated browsing may pause for verification.
- Centralized lead storage in Google Sheets with per-company tabs.

### 1.1 Input Reception & Batch Looping
Reads “Company URLs” from Google Sheets and loops item-by-item over each company domain.

### 1.2 Results Sheet Setup (Per Company)
For each domain, creates a new sheet (tab) named after the domain and initializes headers.

### 1.3 BrowserAct Lead Extraction + Human Verification Handling
Runs a BrowserAct workflow per domain; if BrowserAct returns “paused”, waits to allow manual verification and then fetches task results; if “failed”, sends Telegram alert.

### 1.4 Parsing & Archiving to Google Sheets
Parses a JSON string returned by BrowserAct into individual lead items and appends each lead as a row into the company-specific sheet.

### 1.5 Completion Notification
Once the batch finishes, sends a Slack message indicating extraction is completed for the company.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Batch Looping
**Overview:** Loads the list of company domains from Google Sheets and processes them one at a time using Split In Batches.

**Nodes involved:**
- Execute Manually
- Reading company data
- Loop Over Items

#### Node: Execute Manually
- **Type / role:** Manual Trigger; starts the workflow on demand.
- **Configuration:** Default manual trigger.
- **Outputs:** Sends one execution signal to **Reading company data**.
- **Edge cases:** None (manual start only).

#### Node: Reading company data
- **Type / role:** Google Sheets; reads rows from the “Company URLs” sheet.
- **Configuration choices:**
  - Document: `Company Email List` (Spreadsheet ID `1olY9B0qt_GWLcWitUW49j8qZFilM-rP4IRhr6du6RH4`)
  - Sheet/tab: `Company URLs` (gid=0)
  - Operation: read (implied by node usage; parameters show sheet selection and options).
- **Expected input schema:** A header named **`Company url`** (case-sensitive).
- **Outputs:** One item per row to **Loop Over Items**.
- **Potential failures / edge cases:**
  - OAuth/auth failure to Google Sheets.
  - Sheet name/gid mismatch.
  - Missing column header `Company url` causing downstream expressions to fail.

#### Node: Loop Over Items
- **Type / role:** Split In Batches; iterates through companies sequentially.
- **Configuration choices:**
  - TypeVersion 3; default options (batch size not explicitly set → n8n default applies unless set in UI).
- **Connections:**
  - Output 0 → **Send Finishing Message** (Slack)  
  - Output 1 → **Create Database Sheet** (Google Sheets create tab)
- **Important behavior note:** In Split In Batches, one output is typically used to process the current batch and then loop back until done; the second output is used when there are no items left. Here, the wiring implies:
  - One branch drives processing (create sheet and extraction),
  - Another branch sends Slack completion.
  Ensure the output indices correspond to your intended “main vs done” behavior in your n8n version.
- **Edge cases:**
  - If batch size > 1, sheet creation and downstream nodes must be able to handle multiple items (this workflow assumes one company at a time).
  - Incorrect loop-back wiring can cause premature completion notification or infinite loops.

---

### Block 2 — Results Sheet Setup (Per Company)
**Overview:** For each company domain, creates a dedicated sheet tab and initializes the header row (Name, Email, Position).

**Nodes involved:**
- Create Database Sheet
- Define Headers
- Add Headers

#### Node: Create Database Sheet
- **Type / role:** Google Sheets; creates a new sheet/tab inside the spreadsheet.
- **Configuration choices:**
  - Operation: **Create**
  - Document: same spreadsheet (`Company Email List`)
  - Title (sheet name): `={{ $json["Company url"] }}`
- **Outputs:** Provides a `sheetId` used downstream by **Add Headers** and **Append row in sheet**.
- **Potential failures / edge cases:**
  - Creating a sheet with a duplicate name (Google Sheets may reject or auto-adjust depending on API behavior).
  - Invalid characters or overly long sheet names derived from `Company url`.
  - Missing `Company url` value → expression resolves to empty → sheet creation failure.

#### Node: Define Headers
- **Type / role:** Set; prepares a header object.
- **Configuration choices:**
  - Sets fields `Email`, `Position`, `Name` to empty strings.
- **Purpose in this workflow:** Produces a structured item so the next node can append a row containing these keys (used as headers in the results sheet).
- **Connections:** Output → **Add Headers**.
- **Edge cases:**
  - If downstream expects specific order, appending may not guarantee column ordering.

#### Node: Add Headers
- **Type / role:** Google Sheets; appends a row (intended as the header row) to the newly created sheet.
- **Configuration choices:**
  - Operation: **Append**
  - SheetName: `={{ $('Create Database Sheet').item.json.sheetId }}` (select by sheet ID)
  - Columns mapping: empty in parameters, but it will append whatever data is present in the incoming item (from **Define Headers**). In practice, ensure the node is configured in UI to map fields properly; otherwise it may append blank data.
- **Connections:** Output → **Domain-Specific Lead Extraction**
- **Potential failures / edge cases:**
  - If the node is not configured to map incoming JSON fields, it may append an empty row.
  - Using `sheetId` in a “sheetName” parameter relies on the resource locator mode being “id”; mismatch can break targeting.

---

### Block 3 — BrowserAct Lead Extraction + Human Verification Handling
**Overview:** Calls a BrowserAct workflow per domain. Depending on the returned status, it either proceeds to parse results, waits for human verification and retries fetching results, or sends a failure alert.

**Nodes involved:**
- Domain-Specific Lead Extraction
- Human verification Switch
- Give Time to Complete Verification
- Get Data From BrowserAct
- Send Failure Alert

#### Node: Domain-Specific Lead Extraction
- **Type / role:** BrowserAct node; triggers an external BrowserAct workflow run.
- **Configuration choices:**
  - Type: `WORKFLOW`
  - BrowserAct workflowId: `68914748077345770`
  - Timeout: 7200 seconds (2 hours)
  - Workflow config input mapping:
    - `input-Domain` = `={{ $('Loop Over Items').item.json["Company url"] }}`
  - `open_incognito_mode`: false
- **Outputs:** Returns a task object including an `id` and `status` (used by the Switch).
- **Edge cases / failures:**
  - BrowserAct credential/auth errors.
  - Workflow ID mismatch or deleted template.
  - Browser automation blocked → status `paused` or `failed`.
  - Timeouts if extraction takes too long or network issues.

#### Node: Human verification Switch
- **Type / role:** Switch; routes depending on `{{$json.status}}`.
- **Rules:**
  1. If status equals `finished` → output 0 → **Splitting Items**
  2. If status equals `paused` → output 1 → **Give Time to Complete Verification**
  3. If status equals `failed` → output 2 → **Send Failure Alert**
- **Edge cases:**
  - Unexpected statuses (e.g., `running`, `queued`) are not handled and will produce no output, stalling the workflow.
  - Case sensitivity is enabled; `Finished` vs `finished` would fail routing.

#### Node: Give Time to Complete Verification
- **Type / role:** Wait; pauses execution to allow manual CAPTCHA/security verification.
- **Configuration choices:** Wait **10 minutes**.
- **Connections:** Output → **Get Data From BrowserAct**
- **Edge cases:**
  - Verification may take longer than 10 minutes; workflow will fetch too early.
  - If verification is completed quickly, the workflow still waits the full time.

#### Node: Get Data From BrowserAct
- **Type / role:** HTTP Request; fetches task result from BrowserAct API.
- **Configuration choices:**
  - GET `https://api.browseract.com/v2/workflow/get-task`
  - Query parameter: `task_id={{ $('Domain-Specific Lead Extraction').item.json.id }}`
  - Authentication: predefined credential type `browserActApi`
- **Connections:** Output → **Splitting Items**
- **Edge cases / failures:**
  - If `Domain-Specific Lead Extraction` did not return an `id`, expression fails.
  - API rate limits, timeouts, transient 5xx.
  - Task may still be `paused`/`running`; current workflow does not re-check status here (it proceeds to parsing).

#### Node: Send Failure Alert
- **Type / role:** Telegram; notifies failures.
- **Configuration choices:**
  - Text: `Sorry we facing problem right now.`
  - ChatId field appears as a placeholder: `parameters.chatId==@Channel_ID (Use Channel ID or Chat ID )`
  - Parse mode: HTML, attribution disabled
- **Connections:** None (terminal).
- **Edge cases:**
  - Chat ID placeholder is invalid as-is; must be replaced with a real chat/channel ID.
  - Telegram credential/auth errors.

---

### Block 4 — Parsing & Archiving to Google Sheets
**Overview:** Takes BrowserAct’s returned JSON string, parses it into individual lead items, then appends each lead to the company-specific sheet.

**Nodes involved:**
- Splitting Items
- Append row in sheet

#### Node: Splitting Items
- **Type / role:** Code; parses a JSON string into multiple n8n items.
- **Configuration choices (interpreted):**
  - Reads: `$input.first().json.output.string`
  - Validates non-empty string.
  - `JSON.parse` into an array.
  - Ensures parsed result is an array.
  - Maps each element to `{ json: item }` to “split” into separate n8n items.
- **Connections:** Output → **Append row in sheet**
- **Key variables / expressions:**
  - `$input.first().json.output.string`
- **Edge cases / failures:**
  - If BrowserAct response structure differs (no `output.string`), the node throws an error and stops execution.
  - If the string is not valid JSON, it throws with parse error.
  - If parsed JSON is not an array, it throws.
  - Large payloads may hit memory/runtime limits.

#### Node: Append row in sheet
- **Type / role:** Google Sheets; appends extracted lead rows into the created sheet.
- **Configuration choices:**
  - Operation: **Append**
  - Document: same spreadsheet
  - SheetName (by id): `={{ $('Create Database Sheet').item.json.sheetId }}`
  - Column mapping:
    - Name = `={{ $json.item[0].name }}`
    - Email = `={{ $json.item[0].email }}`
    - Position = `={{ $json.item[0].position }}`
- **Connections:** Output → **Loop Over Items** (loop-back to process next company)
- **Critical data-shape note:**  
  The code node outputs each parsed element as `json: item`. However, this Sheets mapping references `$json.item[0].name`, which implies the parsed element itself contains an `item` array. This is only correct if BrowserAct returns objects shaped like `{ item: [ { name, email, position } ] }`. If BrowserAct returns a simpler shape like `{ name, email, position }`, these expressions will be `undefined`.
- **Edge cases / failures:**
  - Missing fields → blank cells or expression errors depending on n8n settings.
  - Appending many rows can be slow and can hit Google API quotas.
  - If the sheetId reference points to the wrong item context (due to batching), rows may go to the wrong tab.

---

### Block 5 — Completion Notification
**Overview:** After all company domains are processed, sends a message to Slack.

**Nodes involved:**
- Send Finishing Message

#### Node: Send Finishing Message
- **Type / role:** Slack; posts a completion message to a channel.
- **Configuration choices:**
  - Channel: `all-browseract-workflow-test` (channel ID `C09KLV9DJSX`)
  - Message text: `The Email extarction for  {{ $json["Company url"] }} is Finished.`
- **Connections:** None (terminal).
- **Edge cases:**
  - If this node runs on the “done” output of Split In Batches, `$json["Company url"]` may be unavailable or may refer to the last processed item depending on n8n behavior; message could be incorrect or blank.
  - Slack credential/auth errors; missing channel permissions.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Documentation | Sticky Note | Embedded documentation | — | — | ## ⚡ Workflow Overview & Setup / Requirements / How to Use / Need Help? Links: https://docs.browseract.com |
| Sticky Note | Sticky Note | Video link | — | — | @[youtube](_7FlHGfwzWo) |
| Step 1 Explanation | Sticky Note | Explains data prep | — | — | ### 📋 Step 1: Data Preparation (reads domains, creates sheet, sets headers) |
| Sticky Note1 | Sticky Note | Google Sheet structure requirements | — | — | ### 📋 Google Sheet Requirements — Row 1 headers (`Company url`), data from Row 2 |
| Step 2 Explanation | Sticky Note | Explains scraping/parsing/archiving and human verification | — | — | ### 🌐 Step 2: Automated Scraping, Parsing & Archiving (Human Verification Switch, parse, append) |
| Step 4 Explanation | Sticky Note | Explains completion notification | — | — | ### 🔔 Step 3: Completion Notification (Slack message when processed) |
| Execute Manually | Manual Trigger | Entry point | — | Reading company data | ### 📋 Step 1: Data Preparation (reads domains, creates sheet, sets headers) |
| Reading company data | Google Sheets | Read company domains | Execute Manually | Loop Over Items | ### 📋 Step 1: Data Preparation (reads domains, creates sheet, sets headers) / ### 📋 Google Sheet Requirements — Row 1 headers (`Company url`), data from Row 2 |
| Loop Over Items | Split In Batches | Iterate domains + loop-back control | Reading company data; Append row in sheet | Create Database Sheet; Send Finishing Message | ### 📋 Step 1: Data Preparation (reads domains, creates sheet, sets headers) |
| Create Database Sheet | Google Sheets | Create per-company results tab | Loop Over Items | Define Headers | ### 📋 Step 1: Data Preparation (reads domains, creates sheet, sets headers) |
| Define Headers | Set | Create header fields | Create Database Sheet | Add Headers | ### 📋 Step 1: Data Preparation (reads domains, creates sheet, sets headers) |
| Add Headers | Google Sheets | Append header row | Define Headers | Domain-Specific Lead Extraction | ### 📋 Step 1: Data Preparation (reads domains, creates sheet, sets headers) |
| Domain-Specific Lead Extraction | BrowserAct | Run BrowserAct extraction workflow | Add Headers | Human verification Switch | ### 🌐 Step 2: Automated Scraping, Parsing & Archiving (Human Verification Switch, parse, append) |
| Human verification Switch | Switch | Route by BrowserAct status | Domain-Specific Lead Extraction | Splitting Items; Give Time to Complete Verification; Send Failure Alert | ### 🌐 Step 2: Automated Scraping, Parsing & Archiving (Human Verification Switch, parse, append) |
| Give Time to Complete Verification | Wait | Pause for CAPTCHA/security checks | Human verification Switch | Get Data From BrowserAct | ### 🌐 Step 2: Automated Scraping, Parsing & Archiving (Human Verification Switch, parse, append) |
| Get Data From BrowserAct | HTTP Request | Fetch BrowserAct task result | Give Time to Complete Verification | Splitting Items | ### 🌐 Step 2: Automated Scraping, Parsing & Archiving (Human Verification Switch, parse, append) |
| Send Failure Alert | Telegram | Notify failure | Human verification Switch | — | ### 🌐 Step 2: Automated Scraping, Parsing & Archiving (Human Verification Switch, parse, append) |
| Splitting Items | Code | Parse JSON string to items | Human verification Switch; Get Data From BrowserAct | Append row in sheet | ### 🌐 Step 2: Automated Scraping, Parsing & Archiving (Human Verification Switch, parse, append) |
| Append row in sheet | Google Sheets | Append leads to results tab | Splitting Items | Loop Over Items | ### 🌐 Step 2: Automated Scraping, Parsing & Archiving (Human Verification Switch, parse, append) |
| Send Finishing Message | Slack | Completion notification | Loop Over Items | — | ### 🔔 Step 3: Completion Notification (Slack message when processed) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1. Google Sheets OAuth2 credential (access to the target spreadsheet).
   2. BrowserAct API credential (API key from BrowserAct).
   3. Slack API credential (bot/user token with permission to post in the target channel).
   4. Telegram credential (bot token) and decide the target `chatId`.

2. **Prepare Google Sheets**
   1. Create (or use) a spreadsheet (e.g., “Company Email List”).
   2. Add a tab named **“Company URLs”**.
   3. In row 1, add header **`Company url`**.
   4. From row 2 onward, list one company domain/URL per row under `Company url`.

3. **Add nodes in n8n and configure them**

4. **Node: Manual Trigger**  
   - Add **Manual Trigger** node named “Execute Manually”.

5. **Node: Google Sheets (Read)** — “Reading company data”
   - Operation: **Read** (read rows)
   - Document: select your spreadsheet.
   - Sheet: select “Company URLs”.
   - Connect: **Execute Manually → Reading company data**.

6. **Node: Split In Batches** — “Loop Over Items”
   - Add **Split In Batches**.
   - Set batch size to **1** (recommended to ensure one company processed at a time).
   - Connect: **Reading company data → Loop Over Items**.

7. **Node: Google Sheets (Create sheet/tab)** — “Create Database Sheet”
   - Operation: **Create**
   - Document: same spreadsheet
   - Title: use expression `{{ $json["Company url"] }}`
   - Connect: **Loop Over Items (main output for processing) → Create Database Sheet**.

8. **Node: Set** — “Define Headers”
   - Add fields:
     - `Name` = empty string
     - `Email` = empty string
     - `Position` = empty string
   - Connect: **Create Database Sheet → Define Headers**.

9. **Node: Google Sheets (Append)** — “Add Headers”
   - Operation: **Append**
   - Document: same spreadsheet
   - Sheet selector: choose **by ID**, and set it to:
     - `{{ $('Create Database Sheet').item.json.sheetId }}`
   - Map columns to the incoming fields (`Name`, `Email`, `Position`) so the first row becomes headers.
   - Connect: **Define Headers → Add Headers**.

10. **Node: BrowserAct** — “Domain-Specific Lead Extraction”
   - Resource/type: **WORKFLOW**
   - Workflow ID: `68914748077345770` (replace if yours differs)
   - Timeout: 7200 seconds
   - Workflow Config input:
     - `input-Domain` = `{{ $('Loop Over Items').item.json["Company url"] }}`
   - Connect: **Add Headers → Domain-Specific Lead Extraction**.

11. **Node: Switch** — “Human verification Switch”
   - Switch on: `{{ $json.status }}`
   - Rules:
     1. equals `finished` → output 1 (or first route) to parsing
     2. equals `paused` → route to wait
     3. equals `failed` → route to Telegram
   - Connect: **Domain-Specific Lead Extraction → Human verification Switch**.

12. **Node: Wait** — “Give Time to Complete Verification”
   - Wait: 10 minutes
   - Connect: **Human verification Switch (paused) → Give Time to Complete Verification**.

13. **Node: HTTP Request** — “Get Data From BrowserAct”
   - Method: GET
   - URL: `https://api.browseract.com/v2/workflow/get-task`
   - Authentication: BrowserAct predefined credential
   - Query parameter:
     - `task_id` = `{{ $('Domain-Specific Lead Extraction').item.json.id }}`
   - Connect: **Give Time to Complete Verification → Get Data From BrowserAct**.

14. **Node: Code** — “Splitting Items”
   - Paste logic to parse the BrowserAct output string located at `output.string` into multiple items (one per lead).
   - Ensure the incoming path matches the HTTP response / BrowserAct node response. This workflow expects:  
     - `{{ $input.first().json.output.string }}`
   - Connect:
     - **Human verification Switch (finished) → Splitting Items**
     - **Get Data From BrowserAct → Splitting Items**

15. **Node: Google Sheets (Append)** — “Append row in sheet”
   - Operation: **Append**
   - Document: same spreadsheet
   - Sheet selector by ID:
     - `{{ $('Create Database Sheet').item.json.sheetId }}`
   - Map columns:
     - `Name` = expression matching your parsed output
     - `Email` = expression matching your parsed output
     - `Position` = expression matching your parsed output  
   - In the provided workflow, mappings are `{{ $json.item[0].name }}`, etc. Adjust to match the actual parsed structure (commonly `{{ $json.name }}`).
   - Connect: **Splitting Items → Append row in sheet**.

16. **Loop back for next company**
   - Connect: **Append row in sheet → Loop Over Items** (so it requests the next batch item).

17. **Node: Telegram** — “Send Failure Alert” (optional but recommended)
   - Configure bot credential
   - Set `chatId` to a valid channel/user/group ID
   - Message text as desired
   - Connect: **Human verification Switch (failed) → Send Failure Alert**.

18. **Node: Slack** — “Send Finishing Message”
   - Channel: pick your target channel
   - Message: `The Email extarction for {{ $json["Company url"] }} is Finished.`
   - Connect: **Loop Over Items (done/no more items output) → Send Finishing Message**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| BrowserAct help links (API key, workflow ID, connecting n8n, templates) | https://docs.browseract.com |
| Video reference embedded in sticky note | @[youtube](_7FlHGfwzWo) |
| Google Sheet must have header `Company url` in Row 1, data from Row 2 | Applies to “Reading company data” and all downstream expressions relying on `Company url` |
| Workflow relies on a BrowserAct template/workflow (“Company Domain to Email Enrichment”) | Must exist in BrowserAct and match the configured workflow ID |

