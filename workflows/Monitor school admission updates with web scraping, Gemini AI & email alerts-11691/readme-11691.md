Monitor school admission updates with web scraping, Gemini AI & email alerts

https://n8nworkflows.xyz/workflows/monitor-school-admission-updates-with-web-scraping--gemini-ai---email-alerts-11691


# Monitor school admission updates with web scraping, Gemini AI & email alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor selected school websites daily, scrape webpage content, use **Google Gemini** to detect whether **pre-nursery admissions for 2026–2027** appear open, and **send an email alert** when openings are detected.

**Primary use cases:**
- Parents or administrators monitoring admissions pages that change unpredictably.
- “Lightweight” web monitoring without building a custom scraper per school, relying on an LLM to interpret page content.

### 1.1 Scheduling & School List Initialization
Runs daily, generates a list of schools (name + URL), then prepares looping.

### 1.2 Per-School Web Scraping & Text Normalization
Fetches HTML for each school URL, merges it with the school metadata, then strips HTML into a clean text blob.

### 1.3 AI Detection (Gemini) & Response Parsing
Sends cleaned text to Gemini with a strict JSON output requirement, then parses the model response safely.

### 1.4 Alerting & Loop Continuation
If admissions are open, emails the user; regardless of result, continues to the next school until done.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & School List Initialization

**Overview:** Triggers once per day and produces the list of school targets to process sequentially.

**Nodes involved:**
- Daily Trigger
- Shortlisted Schools
- Split In Batches

#### Node: Daily Trigger
- **Type / role:** `Cron` (time-based trigger). Starts the workflow automatically.
- **Configuration (interpreted):** Runs at `0 0 * * *` (daily at 00:00).
- **Connections:** Outputs to **Shortlisted Schools**.
- **Edge cases / failures:**
  - Timezone: cron uses the instance timezone; results may differ from user expectation if server timezone differs.
  - Workflow is **inactive** (`active:false`), so it will not run until activated.

#### Node: Shortlisted Schools
- **Type / role:** `Code` node that returns an array of items (school targets).
- **Configuration (interpreted):** Emits 3 items:
  - Sanskriti School → `https://www.sanskritischool.edu.in/`
  - DPS Mathura Road → `https://www.dpsmathuraroad.org/admission.php`
  - Sardar Patel Vidyalaya → `https://spvdelhi.org/`
- **Key variables produced:** Each item includes `school` and `url`.
- **Connections:** Outputs to **Split In Batches**.
- **Edge cases / failures:**
  - If you add an invalid URL or omit `school/url`, later nodes (HTTP request, merge, email) may fail or produce incorrect messages.

#### Node: Split In Batches
- **Type / role:** `SplitInBatches` for controlled looping over schools.
- **Configuration (interpreted):**
  - `batchSize: 1` → processes one school at a time.
- **Connections:**
  - Main output goes to **Get Website Content** and also provides the “metadata stream” into **Append school Name** input 1 (via a special wiring choice).
  - Receives loop-back input from **Continue Loop**.
- **Edge cases / failures:**
  - If the loop-back (**Continue Loop**) isn’t connected correctly, only the first item will be processed.
  - Batch behavior depends on node version; this workflow uses **typeVersion 2**.

---

### Block 2 — Per-School Web Scraping & Text Normalization

**Overview:** Downloads the target page HTML, attaches school metadata, and converts HTML to cleaned text for LLM analysis.

**Nodes involved:**
- Get Website Content
- Append school Name
- Clean HTML

#### Node: Get Website Content
- **Type / role:** `HTTP Request` to fetch webpage content.
- **Configuration (interpreted):**
  - URL expression: `={{$json["url"]}}` (from the current school item).
  - Default options (no special headers/user-agent configured).
  - `alwaysOutputData: false` → if the request fails, it will likely error rather than emit empty output.
- **Connections:** Outputs to **Append school Name** (input 0).
- **Edge cases / failures:**
  - Anti-bot/WAF blocks (403/503, CAPTCHA) are common for school sites.
  - Missing user-agent can cause denial; consider adding a browser-like header.
  - Large HTML pages may increase execution time and token usage downstream.
  - Network timeouts / TLS issues.

#### Node: Append school Name
- **Type / role:** `Merge` node combining fetched HTML with the original school metadata.
- **Configuration (interpreted):**
  - Mode: **Combine** using **position** (`combineByPosition`).
  - Input 0: HTTP response (HTML/body)
  - Input 1: school item from SplitInBatches (school + url)
- **Result:** Produces items that contain both website response fields (commonly `body` or `data`) and `school/url`.
- **Connections:** Outputs to **Clean HTML**.
- **Edge cases / failures:**
  - If either input emits a different item count, combining by position can misalign data.
  - If the HTTP node returns binary or unexpected structure, downstream “Clean HTML” may not find `data`/`body`.

#### Node: Clean HTML
- **Type / role:** `Code` node to remove HTML tags/scripts/styles and normalize whitespace.
- **Configuration (interpreted):**
  - Runs once per item.
  - Reads HTML from `$json["data"] || $json["body"] || ""`.
  - Strips `<script>…</script>`, `<style>…</style>`, then all tags, collapses whitespace.
  - Returns:
    - `cleaned_text`
    - `school`
    - `url`
- **Connections:** Outputs to **Are admissions Open**.
- **Edge cases / failures:**
  - If the HTTP node response doesn’t store HTML in `data` or `body`, the cleaned text becomes empty and the LLM will likely return “no”.
  - Regex-based stripping can lose meaningful structure (tables, headings). For some sites, extracting a specific admissions section would be more reliable.

---

### Block 3 — AI Detection (Gemini) & Response Parsing

**Overview:** Gemini evaluates the cleaned page text and attempts to output strict JSON indicating whether admissions are open; a code node parses that JSON with fallback handling.

**Nodes involved:**
- Are admissions Open
- Parse LLM response

#### Node: Are admissions Open
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` (Gemini chat/model call).
- **Configuration (interpreted):**
  - Model: `models/gemini-2.5-flash`
  - Prompt instructs the model to detect admissions openings for **2026–2027**, **Pre-nursery**, optional age criterion, and to return **ONLY JSON**:
    ```json
    {
      "admissions_open": "yes/no",
      "school:: {{ $json.school }}
      "reason": "",
      "classes": ""
    }
    ```
  - Includes page content: `{{ $json["cleaned_text"] }}`
- **Credentials:** `googlePalmApi` credential required (Gemini API key / Google AI Studio key depending on n8n credential type).
- **Connections:** Outputs to **Parse LLM response**.
- **Version-specific notes / risks:**
  - The node is **typeVersion 1**; output structure is assumed in parsing node.
- **Edge cases / failures:**
  - Prompt JSON is malformed: it contains `school::` and missing quotes/commas (as written). The model may still respond with valid JSON, but it increases the risk of non-JSON output.
  - LLM may wrap JSON in code fences (handled later).
  - Token limits: very long `cleaned_text` can cause truncation or higher costs.
  - Credential/auth errors.

#### Node: Parse LLM response
- **Type / role:** `Code` node to extract and parse the model’s JSON.
- **Configuration (interpreted):**
  - Reads: `$json.content.parts[0].text` (assumes Gemini node returns this structure).
  - Removes ```json fences and ``` fences, trims.
  - Attempts `JSON.parse`.
  - On failure returns:
    - `admissions_open: "no"`
    - `reason: "Gemini JSON parse failed"`
    - `raw: <unparsed text>`
- **Connections:** Outputs to **If**.
- **Edge cases / failures:**
  - If the Gemini node output schema differs (e.g., text is located elsewhere), `$json.content.parts[0].text` will be undefined and parsing will fail.
  - If the model returns additional commentary outside JSON, parsing fails (fallback covers it but will suppress true positives).
  - No validation of expected keys (`classes`, `school`, etc.).

---

### Block 4 — Alerting & Loop Continuation

**Overview:** Checks parsed AI result; if admissions are open, send an email, then continue to next school. Otherwise, continue without email.

**Nodes involved:**
- If
- Send email
- Continue Loop

#### Node: If
- **Type / role:** Conditional router.
- **Configuration (interpreted):**
  - Condition: `={{ $json.admissions_open }}` **equals** `"yes"` (case-sensitive).
- **Connections:**
  - **True** → Send email
  - **False** → Continue Loop
- **Edge cases / failures:**
  - If Gemini returns `"Yes"` / `"YES"` / boolean `true`, the condition fails and no email will be sent.
  - If `admissions_open` missing, condition evaluates to false.

#### Node: Send email
- **Type / role:** `Email Send` via SMTP for alerts.
- **Configuration (interpreted):**
  - To: `user@example.com` (placeholder)
  - From: `user@example.com` (placeholder)
  - Subject expression: `=“Admissions Open at {{  $json.school }}`
    - Note: subject starts with a curly quote `“` and seems to miss a closing quote/brace; you may want to normalize this.
  - HTML body includes:
    - School: `{{ $json.school }}`
    - Class(es): `{{ $json.classes }}`
    - Reason: `{{ $json.reason }}`
- **Credentials:** SMTP credential required.
- **Connections:** After sending, goes to **Continue Loop**.
- **Edge cases / failures:**
  - SMTP auth issues, blocked ports, TLS requirements.
  - Invalid From address may be rejected by the server.
  - If `classes` is absent, email renders blank field.

#### Node: Continue Loop
- **Type / role:** `SplitInBatches` used as a loop control “next batch” trigger.
- **Configuration (interpreted):** Default options; its purpose is to feed back into **Split In Batches** to fetch the next school.
- **Connections:** Outputs to **Split In Batches** (loop-back).
- **Edge cases / failures:**
  - If this node is removed/disconnected, the workflow will stop after first school.
  - Ensure no accidental additional paths feed into SplitInBatches, which can restart or duplicate processing.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Trigger | n8n-nodes-base.cron | Daily scheduler entry point | — | Shortlisted Schools | ## School Admission Tracker via Web Scraping<br>[Guide](https://docs.n8n.io/workflows/components/sticky-notes/)<br><br>**How it works**<br>**For each shortlisted school **, perform the following: <br>1. Get the website content.<br>2. Clean the HTML content <br>3. LLM parses the text to detect school admission openings for year 2026-2027 for Pre-nursery. Based on LLM response:<br>a. If **yes**, then trigger an email notifying the school for which admissions are open.<br>b. Else continue for next school.<br><br><br>**How to setup**<br>- Add Gemini API key within "Are admissions open" node<br>- Add SMTP account credentials within "Send Email" node<br>- Add From-Email and To-Email within "Send Email" node<br><br><br>**Customize**<br>- Update the school list within "Shortlisted Schools" node.<br>- Modify "Are admissions open" node for respective year/ class. |
| Shortlisted Schools | n8n-nodes-base.code | Produces list of schools (name + URL) | Daily Trigger | Split In Batches | ## School Admission Tracker via Web Scraping<br>[Guide](https://docs.n8n.io/workflows/components/sticky-notes/)<br><br>**How it works**<br>**For each shortlisted school **, perform the following: <br>1. Get the website content.<br>2. Clean the HTML content <br>3. LLM parses the text to detect school admission openings for year 2026-2027 for Pre-nursery. Based on LLM response:<br>a. If **yes**, then trigger an email notifying the school for which admissions are open.<br>b. Else continue for next school.<br><br><br>**How to setup**<br>- Add Gemini API key within "Are admissions open" node<br>- Add SMTP account credentials within "Send Email" node<br>- Add From-Email and To-Email within "Send Email" node<br><br><br>**Customize**<br>- Update the school list within "Shortlisted Schools" node.<br>- Modify "Are admissions open" node for respective year/ class. |
| Split In Batches | n8n-nodes-base.splitInBatches | Iterates through schools one-by-one | Shortlisted Schools, Continue Loop | Get Website Content, Append school Name | ## School Admission Tracker via Web Scraping<br>[Guide](https://docs.n8n.io/workflows/components/sticky-notes/)<br><br>**How it works**<br>**For each shortlisted school **, perform the following: <br>1. Get the website content.<br>2. Clean the HTML content <br>3. LLM parses the text to detect school admission openings for year 2026-2027 for Pre-nursery. Based on LLM response:<br>a. If **yes**, then trigger an email notifying the school for which admissions are open.<br>b. Else continue for next school.<br><br><br>**How to setup**<br>- Add Gemini API key within "Are admissions open" node<br>- Add SMTP account credentials within "Send Email" node<br>- Add From-Email and To-Email within "Send Email" node<br><br><br>**Customize**<br>- Update the school list within "Shortlisted Schools" node.<br>- Modify "Are admissions open" node for respective year/ class. |
| Get Website Content | n8n-nodes-base.httpRequest | Fetches HTML from school URL | Split In Batches | Append school Name | ## 1. Get School's Website Contents |
| Append school Name | n8n-nodes-base.merge | Merges HTTP content with school metadata | Get Website Content, Split In Batches | Clean HTML | ## 1. Get School's Website Contents |
| Clean HTML | n8n-nodes-base.code | Converts HTML into plain cleaned text | Append school Name | Are admissions Open | ## 1. Get School's Website Contents |
| Are admissions Open | @n8n/n8n-nodes-langchain.googleGemini | LLM classification: admissions open yes/no + details | Clean HTML | Parse LLM response | ## 2. Detect school admissions opening & trigger Alert  |
| Parse LLM response | n8n-nodes-base.code | Parses Gemini response into structured JSON | Are admissions Open | If | ## 2. Detect school admissions opening & trigger Alert  |
| If | n8n-nodes-base.if | Branching based on admissions_open | Parse LLM response | Send email, Continue Loop | ## 2. Detect school admissions opening & trigger Alert  |
| Send email | n8n-nodes-base.emailSend | Sends alert email via SMTP | If (true) | Continue Loop | ## 2. Detect school admissions opening & trigger Alert  |
| Continue Loop | n8n-nodes-base.splitInBatches | Triggers next batch iteration | If (false), Send email | Split In Batches | ## 2. Detect school admissions opening & trigger Alert  |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / comments | — | — | ## School Admission Tracker via Web Scraping<br>[Guide](https://docs.n8n.io/workflows/components/sticky-notes/)<br><br>**How it works**<br>**For each shortlisted school **, perform the following: <br>1. Get the website content.<br>2. Clean the HTML content <br>3. LLM parses the text to detect school admission openings for year 2026-2027 for Pre-nursery. Based on LLM response:<br>a. If **yes**, then trigger an email notifying the school for which admissions are open.<br>b. Else continue for next school.<br><br><br>**How to setup**<br>- Add Gemini API key within "Are admissions open" node<br>- Add SMTP account credentials within "Send Email" node<br>- Add From-Email and To-Email within "Send Email" node<br><br><br>**Customize**<br>- Update the school list within "Shortlisted Schools" node.<br>- Modify "Are admissions open" node for respective year/ class. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / section header | — | — | ## 1. Get School's Website Contents |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / section header | — | — | ## 2. Detect school admissions opening & trigger Alert  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: **School Admission Tracker**
- Ensure execution order setting is default (this workflow uses `v1`, but default typically works).

2) **Add Trigger**
- Add node: **Cron**
- Name: `Daily Trigger`
- Set schedule to **Custom cron**: `0 0 * * *` (daily at midnight)
- (Optional) adjust timezone at instance/workflow level as needed.

3) **Add school list generator**
- Add node: **Code**
- Name: `Shortlisted Schools`
- Code (conceptually): return an array of objects with `school` and `url`, e.g.
  - `{ school: "Sanskriti School", url: "https://www.sanskritischool.edu.in/" }`
- Connect: `Daily Trigger` → `Shortlisted Schools`

4) **Add looping**
- Add node: **Split In Batches**
- Name: `Split In Batches`
- Batch size: `1`
- Connect: `Shortlisted Schools` → `Split In Batches`

5) **Fetch website content**
- Add node: **HTTP Request**
- Name: `Get Website Content`
- URL: expression `{{$json.url}}`
- Connect: `Split In Batches` → `Get Website Content`

6) **Merge HTTP response with school metadata**
- Add node: **Merge**
- Name: `Append school Name`
- Mode: **Combine**
- Combine by: **Position**
- Connect:
  - `Get Website Content` → `Append school Name` (Input 1 / first input)
  - `Split In Batches` → `Append school Name` (Input 2 / second input)

7) **Clean HTML**
- Add node: **Code**
- Name: `Clean HTML`
- Mode: run once per item
- Logic:
  - Read HTML from `$json.data` or `$json.body`
  - Strip script/style/tags, collapse whitespace
  - Output: `cleaned_text`, plus `school` and `url`
- Connect: `Append school Name` → `Clean HTML`

8) **Add Gemini AI node**
- Add node: **Google Gemini (LangChain)**
- Name: `Are admissions Open`
- Credentials: configure **Google Palm/Gemini API** credential in n8n (API key)
- Model: `models/gemini-2.5-flash`
- Message/prompt:
  - Provide instructions to return ONLY JSON with keys: `admissions_open`, `school`, `reason`, `classes`
  - Include `{{$json.cleaned_text}}`
- Connect: `Clean HTML` → `Are admissions Open`

9) **Parse model output**
- Add node: **Code**
- Name: `Parse LLM response`
- Extract the text from the Gemini output (ensure it matches the node output schema in your n8n version), strip code fences, `JSON.parse`, fallback to `{admissions_open:"no" ...}`
- Connect: `Are admissions Open` → `Parse LLM response`

10) **Branch on result**
- Add node: **If**
- Name: `If`
- Condition: string equals
  - Left: `{{$json.admissions_open}}`
  - Right: `yes`
- Connect: `Parse LLM response` → `If`

11) **Send email alert**
- Add node: **Email Send**
- Name: `Send email`
- Credentials: set up **SMTP** credentials in n8n (host, port, user, password, TLS).
- To / From: set real addresses (replace placeholders).
- Subject: include `{{$json.school}}`
- HTML body: include `school`, `classes`, and `reason`.
- Connect: `If` (true) → `Send email`

12) **Continue loop**
- Add node: **Split In Batches** (used as a “continue” helper)
- Name: `Continue Loop`
- Connect:
  - `If` (false) → `Continue Loop`
  - `Send email` → `Continue Loop`
  - `Continue Loop` → `Split In Batches` (this is the loop-back that triggers the next school)

13) **(Optional) Add sticky notes**
- Add sticky notes as headers:
  - “## 1. Get School’s Website Contents”
  - “## 2. Detect school admissions opening & trigger Alert”
- Add a larger note describing setup (Gemini API key + SMTP + customization).

14) **Activate workflow**
- Turn workflow **Active** so Cron can run.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Guide | https://docs.n8n.io/workflows/components/sticky-notes/ |
| Setup requirements: add Gemini API key in “Are admissions open”; add SMTP credentials; set From/To emails | From main sticky note |
| Customization: update school list in “Shortlisted Schools”; modify detection prompt for year/class | From main sticky note |