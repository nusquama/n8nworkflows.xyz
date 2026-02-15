Collect reviews, analyze sentiment, and generate reports with Decodo and Google Gemini

https://n8nworkflows.xyz/workflows/collect-reviews--analyze-sentiment--and-generate-reports-with-decodo-and-google-gemini-13189


# Collect reviews, analyze sentiment, and generate reports with Decodo and Google Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Collect reviews, analyze sentiment, and generate reports with Decodo and Google Gemini  
**Purpose:** This workflow runs two automated cycles:  
1) **Daily collection & storage** of customer reviews from URLs stored in Airtable, extraction via Decodo, structuring via Google Gemini, then archiving in Google Sheets.  
2) **Periodic insight generation** where all stored reviews are pulled from Google Sheets, summarized by a Gemini-powered AI Agent, and emailed via Gmail.

### Logical Blocks
**1.1 Scheduling & Execution Control**
- Two schedule triggers drive the two cycles (daily scraping and periodic summary reporting).

**1.2 Review Link Retrieval (Airtable)**
- Pulls all Airtable records where the `Links` field is not empty.

**1.3 Review Scraping + Batch Control (Decodo + Split + Wait)**
- Decodo fetches review-page raw content.
- Items are processed in batches (size 3) with a wait to throttle/pace extraction.

**1.4 Review Structuring (Gemini “Analyzer” Agent)**
- Gemini agent extracts only `date`, `rating`, and `review text` from scraped raw content.

**1.5 Storage (Google Sheets Append)**
- Appends each extracted review as a row in a Google Sheet.

**1.6 Report Generation & Delivery (Sheets → Code → AI Agent → Gmail)**
- Reads all rows from the sheet, restructures them into a `reviews[]` array, asks Gemini to summarize sentiment and themes, then emails the summary.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Execution Control
**Overview:** Starts two independent automation cycles: one for daily scraping and one for periodic review summarization.

**Nodes Involved:**
- `Run Daily` (Schedule Trigger)
- `Schedule Trigger1` (Schedule Trigger)

**Node Details**

1) **Run Daily**
- **Type / Role:** `n8n-nodes-base.scheduleTrigger` — triggers the collection cycle automatically.
- **Config (interpreted):** `rule.interval` is set but appears generic (`[{}]`). In n8n this typically means “every day” if configured in UI; confirm in the node UI because the JSON alone is ambiguous.
- **Outputs:** To `Search records`.
- **Edge cases / failures:** Misconfigured schedule interval may result in no runs or unexpected frequency.

2) **Schedule Trigger1**
- **Type / Role:** `n8n-nodes-base.scheduleTrigger` — triggers the summarization cycle.
- **Config (interpreted):** Runs on an interval measured in **months** (`field: months`), but the exact cadence isn’t specified beyond that.
- **Outputs:** To `Get row(s) in sheet`.
- **Edge cases / failures:** If interval details are incomplete, the trigger may not fire as intended.

---

### 2.2 Review Link Retrieval (Airtable)
**Overview:** Searches Airtable for all records containing review URLs in a `Links` field.

**Nodes Involved:**
- `Search records` (Airtable)

**Node Details**

1) **Search records**
- **Type / Role:** `n8n-nodes-base.airtable` — searches records in a base/table.
- **Key configuration:**
  - **Operation:** Search
  - **Return all:** `true`
  - **Filter formula:** `NOT({Links} = '')` (only records where Links is not blank)
  - **Auth:** Airtable OAuth2 / Personal Access Token credential
  - **Target:** Base `appUZW7ajDzzAXOQn`, Table `Table 1` (`tbl4VejzJZ2olbGtr`)
- **Inputs:** From `Run Daily`.
- **Outputs:** To `Decodo`.
- **Edge cases / failures:**
  - Airtable auth/token expired or insufficient table permissions.
  - Field name mismatch (`Links` must exist exactly).
  - If `Links` contains multiple URLs or non-URL text, downstream scraping may fail unless Decodo handles it.

---

### 2.3 Review Scraping + Batch Control (Decodo + Split + Wait)
**Overview:** Fetches raw content from review URLs and processes them in small batches with an enforced delay.

**Nodes Involved:**
- `Decodo`
- `Loop Over Items` (Split In Batches)
- `Delay until next cycle` (Wait)

**Node Details**

1) **Decodo**
- **Type / Role:** `@decodo/n8n-nodes-decodo.decodo` — external scraping/extraction node that visits URLs and returns raw content.
- **Key configuration:** No explicit parameters shown (defaults).
- **Inputs:** From `Search records`.
- **Outputs:** To `Loop Over Items`.
- **Edge cases / failures:**
  - Missing Decodo credentials/config (none are attached in JSON).
  - Target websites blocking scraping, CAPTCHA, rate limits.
  - Returned data structure may vary; the downstream `Analyzer` assumes `results[0].content` exists.

2) **Loop Over Items**
- **Type / Role:** `n8n-nodes-base.splitInBatches` — processes items in batches to control throughput.
- **Key configuration:** `batchSize: 3`
- **Inputs:** From `Decodo`.
- **Outputs:**
  - **Main output 0:** to `Analyzer` (process current batch)
  - **Main output 1:** to `Delay until next cycle` (continue loop control)
- **Edge cases / failures:**
  - If Decodo returns zero items, the batch loop may not behave as expected (no iterations).
  - If downstream node errors mid-batch, remaining items may not process.

3) **Delay until next cycle**
- **Type / Role:** `n8n-nodes-base.wait` — throttles processing between batches.
- **Key configuration:** `amount: 50` (time unit depends on node settings; in many n8n Wait modes this is seconds/minutes—verify in UI).
- **Inputs:** From `Loop Over Items` output 1.
- **Outputs:** Back into `Analyzer` (continues processing after wait).
- **Edge cases / failures:**
  - Wait node may pause execution and require n8n to be running with queue/executions persistence (depending on hosting mode).
  - If “amount” unit is misinterpreted, delays may be too long/short.

---

### 2.4 Review Structuring (Gemini “Analyzer” Agent)
**Overview:** Uses a LangChain Agent powered by Google Gemini to extract structured review fields from scraped text.

**Nodes Involved:**
- `Analyzer` (AI Agent)
- `Google Gemini Chat Model` (LLM connection)

**Node Details**

1) **Google Gemini Chat Model**
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides Gemini chat LLM to agents.
- **Key configuration:** `modelName: models/gemini-2.0-flash-lite`
- **Credential:** Google Gemini(PaLM) API
- **Connections:** LLM output wired to `Analyzer` via `ai_languageModel`.
- **Edge cases / failures:** API key invalid, quota exceeded, model name not available in your region/project.

2) **Analyzer**
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — prompts Gemini to extract structured fields.
- **Key configuration:**
  - **Prompt:** `Extract only reviews ratings with dates from {{ $json.results[0].content }}`
  - **Reliability:** `retryOnFail: true`, `maxTries: 5`, `waitBetweenTries: 5000ms`
- **Inputs:** From `Loop Over Items` and also from `Delay until next cycle` (loop continuation).
- **Outputs:** To `Append row in sheet`.
- **Key expressions / variables:** `$json.results[0].content` (assumes Decodo output includes `results` array with `content`).
- **Edge cases / failures:**
  - If `results` is missing/empty, the expression resolves to `undefined` and the agent may hallucinate or fail.
  - The agent output format is not enforced (no JSON schema). If it returns unstructured text, Google Sheets mapping will fail because append expects `json.date`, `json.rating`, `json.review`.
  - LLM extraction variability: dates/ratings might be misread if page format differs.

---

### 2.5 Storage (Google Sheets Append)
**Overview:** Writes each extracted review to a Google Sheet to build a historical dataset.

**Nodes Involved:**
- `Append row in sheet` (Google Sheets)

**Node Details**

1) **Append row in sheet**
- **Type / Role:** `n8n-nodes-base.googleSheets` — appends a row.
- **Key configuration:**
  - **Operation:** Append
  - **Document:** Spreadsheet named “Reviews” (`1JCkvUpgFVG4zo6JYijLAzLH0IyHz2qMP7thdAKrKidk`)
  - **Sheet:** `Sheet1` (gid=0)
  - **Column mapping:**
    - `Date` ← `json.date`
    - `Rating` ← `json.rating`
    - `Reviews` ← `json.review`
- **Inputs:** From `Analyzer`.
- **Outputs:** None downstream (end of collection cycle).
- **Edge cases / failures:**
  - OAuth credential expired/revoked.
  - Sheet columns must exist exactly as `Date`, `Rating`, `Reviews`.
  - If `json.rating` is non-numeric text, it still writes (type conversion is disabled), but later calculations might fail.
  - If Analyzer returns multiple reviews in one item, this node (as configured) will append one row per incoming item; you may need an Item Lists / Split Out step if the agent returns an array.

---

### 2.6 Report Generation & Delivery (Sheets → Code → AI Agent → Gmail)
**Overview:** Periodically pulls all archived reviews, formats them into a single payload, summarizes insights with Gemini, and emails the results.

**Nodes Involved:**
- `Get row(s) in sheet` (Google Sheets)
- `Code in JavaScript` (Code)
- `AI Agent1` (AI Agent)
- `Google Gemini Chat Model1` (LLM connection)
- `Send a message` (Gmail)

**Node Details**

1) **Get row(s) in sheet**
- **Type / Role:** `n8n-nodes-base.googleSheets` — reads rows from the sheet.
- **Key configuration:** Document “Reviews”, Sheet1. Operation is “get row(s)” (exact filtering not specified; defaults typically fetch all rows).
- **Inputs:** From `Schedule Trigger1`.
- **Outputs:** To `Code in JavaScript`.
- **Edge cases / failures:** Large sheets can cause slow runs/timeouts; permissions/auth issues.

2) **Code in JavaScript**
- **Type / Role:** `n8n-nodes-base.code` — restructures rows into a `reviews` array for AI summarization.
- **Key configuration (logic):**
  - Builds: `reviews = items.map(item => ({ date: item.json.Date, review: item.json.Reviews, rating: item.json.Rating }))`
  - Calculates `avgRating` but **does not output it** (computed then unused).
  - Returns **one single item**: `{ json: { reviews: [...] } }`
- **Inputs:** From `Get row(s) in sheet`.
- **Outputs:** To `AI Agent1`.
- **Edge cases / failures:**
  - `rating` is likely a string from Sheets; `sum + r.rating` can produce string concatenation or `NaN`. (Currently irrelevant since `avgRating` isn’t returned, but it signals a bug.)
  - Empty sheet → `reviews.length` is 0; division by zero if avgRating were used.

3) **Google Gemini Chat Model1**
- **Type / Role:** Gemini LLM connector for the reporting agent.
- **Key configuration:** defaults (model not explicitly set here; it will use node default unless configured in UI).
- **Inputs/Outputs:** Connected to `AI Agent1` via `ai_languageModel`.
- **Edge cases:** Same as the other Gemini node (quota, auth, model availability).

4) **AI Agent1**
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — generates the review insights report.
- **Key configuration:**
  - Prompt includes `REVIEWS DATA: {{ JSON.stringify($json.reviews, null, 2) }}`
  - Required output sections:
    1. Overall sentiment
    2. Common positive points
    3. Common complaints
    4. Suggested improvements
- **Inputs:** From `Code in JavaScript` (single item with `reviews` array).
- **Outputs:** To `Send a message`.
- **Edge cases / failures:**
  - Very large `reviews` array may exceed model input limits; consider summarizing per batch or limiting to recent N reviews.
  - If `reviews` is missing, the prompt will embed `undefined` and reduce output quality.

5) **Send a message**
- **Type / Role:** `n8n-nodes-base.gmail` — emails the AI-generated report.
- **Key configuration:**
  - To: `user@example.com` (must be updated)
  - Subject: `Reviews Summary`
  - Body: `{{ $json.output }}`
- **Inputs:** From `AI Agent1`.
- **Outputs:** End of reporting cycle.
- **Edge cases / failures:**
  - Gmail OAuth expired / missing scopes.
  - If AI Agent output field name differs (not `output`), email body will be blank. (LangChain agent nodes typically output in `output`, but confirm in your n8n version.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Daily | Schedule Trigger | Starts daily scraping cycle | — | Search records | ## Purpose: Controls when the workflow runs / Includes: / * Run Daily (review scraping trigger) / * Scheduled Review Analysis trigger / * Manages automated execution without manual intervention. |
| Search records | Airtable | Find Airtable records with review links | Run Daily | Decodo | ## Purpose: Fetches and extracts raw review data / Includes: / * Airtable Search Records / * Decodo / * Loop Over Items / * Delay until next cycle / / Responsible for pulling review links and safely processing them in batches. |
| Decodo | Decodo | Scrape/extract raw review content from URLs | Search records | Loop Over Items | ## Purpose: Fetches and extracts raw review data / Includes: / * Airtable Search Records / * Decodo / * Loop Over Items / * Delay until next cycle / / Responsible for pulling review links and safely processing them in batches. |
| Loop Over Items | Split In Batches | Batch processing control (size 3) | Decodo | Analyzer; Delay until next cycle | ## Purpose: Fetches and extracts raw review data / Includes: / * Airtable Search Records / * Decodo / * Loop Over Items / * Delay until next cycle / / Responsible for pulling review links and safely processing them in batches. |
| Delay until next cycle | Wait | Throttle/pace batches | Loop Over Items | Analyzer | ## Purpose: Fetches and extracts raw review data / Includes: / * Airtable Search Records / * Decodo / * Loop Over Items / * Delay until next cycle / / Responsible for pulling review links and safely processing them in batches. |
| Google Gemini Chat Model | Google Gemini Chat Model (LangChain) | LLM for extraction agent | — | Analyzer (ai_languageModel) |  |
| Analyzer | AI Agent (LangChain) | Extract date/rating/review from raw scraped content | Loop Over Items; Delay until next cycle; Google Gemini Chat Model (LLM) | Append row in sheet |  |
| Append row in sheet | Google Sheets | Persist extracted reviews to sheet | Analyzer | — | ## Purpose: Saves data and generates insights / Includes: / * Append row in Google Sheets / * Get row(s) from Google Sheets / * Code (data structuring) / * AI Agent (review analysis) / * Gmail (send summary email) / * Stores reviews and delivers AI-generated insights. |
| Schedule Trigger1 | Schedule Trigger | Starts periodic analysis/report cycle | — | Get row(s) in sheet | ## Purpose: Controls when the workflow runs / Includes: / * Run Daily (review scraping trigger) / * Scheduled Review Analysis trigger / * Manages automated execution without manual intervention. |
| Get row(s) in sheet | Google Sheets | Read all stored review rows | Schedule Trigger1 | Code in JavaScript | ## Purpose: Saves data and generates insights / Includes: / * Append row in Google Sheets / * Get row(s) from Google Sheets / * Code (data structuring) / * AI Agent (review analysis) / * Gmail (send summary email) / * Stores reviews and delivers AI-generated insights. |
| Code in JavaScript | Code | Convert sheet rows into `reviews[]` payload | Get row(s) in sheet | AI Agent1 | ## Purpose: Saves data and generates insights / Includes: / * Append row in Google Sheets / * Get row(s) from Google Sheets / * Code (data structuring) / * AI Agent (review analysis) / * Gmail (send summary email) / * Stores reviews and delivers AI-generated insights. |
| Google Gemini Chat Model1 | Google Gemini Chat Model (LangChain) | LLM for reporting agent | — | AI Agent1 (ai_languageModel) |  |
| AI Agent1 | AI Agent (LangChain) | Summarize sentiment and themes across reviews | Code in JavaScript; Google Gemini Chat Model1 (LLM) | Send a message | ## Purpose: Saves data and generates insights / Includes: / * Append row in Google Sheets / * Get row(s) from Google Sheets / * Code (data structuring) / * AI Agent (review analysis) / * Gmail (send summary email) / * Stores reviews and delivers AI-generated insights. |
| Send a message | Gmail | Email the summary report | AI Agent1 | — | ## Purpose: Saves data and generates insights / Includes: / * Append row in Google Sheets / * Get row(s) from Google Sheets / * Code (data structuring) / * AI Agent (review analysis) / * Gmail (send summary email) / * Stores reviews and delivers AI-generated insights. |
| Sticky Note | Sticky Note | Comment block (scheduling) | — | — |  |
| Sticky Note1 | Sticky Note | Comment block (how it works + setup steps) | — | — |  |
| Sticky Note3 | Sticky Note | Comment block (storage + insights) | — | — |  |
| Sticky Note4 | Sticky Note | Comment block (fetch & extract raw data) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Schedule Trigger (daily)**
   - Add node: **Schedule Trigger**
   - Name: `Run Daily`
   - Set schedule to **Daily** (or your desired cadence).

2) **Add Airtable Search**
   - Add node: **Airtable**
   - Name: `Search records`
   - Authentication: **Airtable OAuth2 / Personal Access Token**
   - Select **Base** and **Table**
   - Operation: **Search**
   - Return All: **true**
   - Filter formula: `NOT({Links} = '')`
   - Connect: `Run Daily` → `Search records`

3) **Add Decodo scraping node**
   - Add node: **Decodo** (`@decodo/n8n-nodes-decodo.decodo`)
   - Name: `Decodo`
   - Configure Decodo credentials/settings as required by your Decodo account (the JSON shows none, so you must add them).
   - Connect: `Search records` → `Decodo`

4) **Add batch loop control**
   - Add node: **Split In Batches**
   - Name: `Loop Over Items`
   - Batch size: `3`
   - Connect: `Decodo` → `Loop Over Items`

5) **Add Gemini model for extraction**
   - Add node: **Google Gemini Chat Model** (LangChain)
   - Name: `Google Gemini Chat Model`
   - Credentials: **Google Gemini(PaLM) API**
   - Model: `models/gemini-2.0-flash-lite`

6) **Add extraction agent**
   - Add node: **AI Agent** (LangChain Agent)
   - Name: `Analyzer`
   - Prompt (Define):
     - `Extract only reviews ratings with dates from {{ $json.results[0].content }}`
   - Enable retries (to match workflow intent):
     - Retry on fail: **on**
     - Max tries: **5**
     - Wait between tries: **5000 ms**
   - Connect LLM: `Google Gemini Chat Model` → `Analyzer` (via **ai_languageModel**)
   - Connect data: `Loop Over Items` (output 0) → `Analyzer`

7) **Add wait/throttle**
   - Add node: **Wait**
   - Name: `Delay until next cycle`
   - Configure delay amount: `50` (confirm time unit in UI)
   - Connect: `Loop Over Items` (output 1) → `Delay until next cycle`
   - Connect: `Delay until next cycle` → `Analyzer`  
   (This preserves the workflow’s loop pacing design.)

8) **Add Google Sheets append**
   - Add node: **Google Sheets**
   - Name: `Append row in sheet`
   - Credentials: **Google Sheets OAuth2**
   - Operation: **Append**
   - Select Spreadsheet (document) and Sheet tab
   - Map columns (must exist in the sheet header):
     - Date ← `{{$json.date}}`
     - Rating ← `{{$json.rating}}`
     - Reviews ← `{{$json.review}}`
   - Connect: `Analyzer` → `Append row in sheet`

9) **Create Schedule Trigger (reporting)**
   - Add node: **Schedule Trigger**
   - Name: `Schedule Trigger1`
   - Set to run **monthly** (or your desired cadence).

10) **Add Google Sheets read**
   - Add node: **Google Sheets**
   - Name: `Get row(s) in sheet`
   - Credentials: same Sheets OAuth2
   - Operation: **Get row(s)** (configure to read the full dataset or a range)
   - Select same Spreadsheet and Sheet
   - Connect: `Schedule Trigger1` → `Get row(s) in sheet`

11) **Add Code node to structure data**
   - Add node: **Code** (JavaScript)
   - Name: `Code in JavaScript`
   - Paste logic to output one item containing `{ reviews: [...] }` using sheet columns `Date`, `Rating`, `Reviews`.
   - Connect: `Get row(s) in sheet` → `Code in JavaScript`

12) **Add Gemini model for reporting**
   - Add node: **Google Gemini Chat Model** (LangChain)
   - Name: `Google Gemini Chat Model1`
   - Credentials: same Gemini API credential
   - Optionally set an explicit model to avoid default changes.

13) **Add reporting AI Agent**
   - Add node: **AI Agent**
   - Name: `AI Agent1`
   - Prompt (Define) using `{{$json.reviews}}`, requesting:
     - overall sentiment, positives, complaints, improvements
   - Connect LLM: `Google Gemini Chat Model1` → `AI Agent1` (ai_languageModel)
   - Connect data: `Code in JavaScript` → `AI Agent1`

14) **Add Gmail send**
   - Add node: **Gmail**
   - Name: `Send a message`
   - Credentials: **Gmail OAuth2**
   - To: set your recipient email (replace `user@example.com`)
   - Subject: `Reviews Summary`
   - Message/body: `{{$json.output}}`
   - Connect: `AI Agent1` → `Send a message`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Purpose: Controls when the workflow runs. Includes Run Daily + Scheduled Review Analysis trigger. | Sticky note “Purpose: Controls when the workflow runs” |
| Purpose: Fetches and extracts raw review data (Airtable → Decodo → batching → delay). | Sticky note “Purpose: Fetches and extracts raw review data” |
| Purpose: Saves data and generates insights (Sheets append/read → code → AI → Gmail). | Sticky note “Purpose: Saves data and generates insights” |
| How It Works + Setup Steps (two cycles; connect Airtable, Sheets, Gemini API key, Gmail; adjust triggers). | Sticky note “How It Works” |
| Implementation caution: the extraction agent must output fields matching `date`, `rating`, `review` for Google Sheets mapping to work. | Derived from node mappings |
| Scaling caution: summarization prompt may exceed token limits if the sheet grows large; consider limiting to recent rows or summarizing in chunks. | Derived from design |

