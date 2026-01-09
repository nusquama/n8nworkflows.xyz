AI-powered review sentiment analysis to Salesforce & Google Sheets with Decodo

https://n8nworkflows.xyz/workflows/ai-powered-review-sentiment-analysis-to-salesforce---google-sheets-with-decodo-11489


# AI-powered review sentiment analysis to Salesforce & Google Sheets with Decodo

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *LLM-Driven Review Summary & KPI Extractor*  
**User title:** *AI-powered review sentiment analysis to Salesforce & Google Sheets with Decodo*

**Purpose:**  
This workflow reads Trustpilot review-page URLs (and optional Salesforce Account IDs) from a Google Sheet, scrapes each page via **Decodo**, cleans the HTML into plain text, asks an **LLM (OpenAI via LangChain Agent)** to extract aggregated review KPIs and sentiment insights, then writes results to:
1) an **“output” Google Sheet** (structured metrics), and  
2) **Salesforce Account** (a short sentiment/trend summary in a custom field), only when a Salesforce Account ID is provided.

### Logical blocks
1.1 **Manual start & input acquisition**  
1.2 **Batch looping over rows**  
1.3 **Scrape review pages (Decodo)**  
1.4 **HTML → clean text + metadata extraction**  
1.5 **LLM aggregation (sentiment/KPIs JSON)**  
1.6 **Flatten + persist to Google Sheets**  
1.7 **Conditional Salesforce update (truncate to 255 chars + upsert)**

---

## 2. Block-by-Block Analysis

### 2.1 Manual start & input acquisition
**Overview:** Starts the workflow manually and loads all input rows (Trustpilot URLs + Salesforce IDs) from a Google Sheet.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Get row(s) in sheet

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — workflow entry point.
- **Configuration:** No parameters; execution begins when manually run.
- **Connections:**
  - **Output →** Get row(s) in sheet
- **Failure modes / edge cases:** None (beyond workflow not being executed).

#### Node: Get row(s) in sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads input dataset.
- **Configuration choices (interpreted):**
  - **Operation:** Get rows (default “read” behavior).
  - **Document:** Spreadsheet named `url_feedback_truspilot` (ID `1TqXzrR4...`).
  - **Sheet/tab:** `input` (`gid=0`).
  - No filters/options configured (reads all rows available under the node’s default behavior).
- **Expected input columns (implied by later expressions):**
  - `url` (Trustpilot review page URL)
  - `salesforce_account_id` (Salesforce Account Id; may be blank)
- **Connections:**
  - **Input ←** Manual Trigger
  - **Output →** Loop Over Items
- **Credentials:** Google Sheets OAuth2.
- **Failure modes / edge cases:**
  - OAuth token expired / insufficient permissions
  - Sheet ID/tab mismatch
  - Missing expected columns (`url`, `salesforce_account_id`) causing downstream expressions to resolve to empty strings

---

### 2.2 Batch looping over rows
**Overview:** Iterates through each input row (each URL/account pair) and orchestrates “per-row” processing; loops back after each item is processed.

**Nodes involved:**
- Loop Over Items

#### Node: Loop Over Items
- **Type / role:** Split in Batches (`n8n-nodes-base.splitInBatches`) — batching/loop controller.
- **Configuration choices:**
  - Batch size not explicitly set (uses node default). In many n8n versions, default batch size is **1**, meaning one row processed per loop iteration.
- **Connections:**
  - **Input ←** Get row(s) in sheet
  - **Output (main, index 1) →** Decodo (this is the “current batch/items” path in this workflow’s wiring)
  - **Input loop-back ←** If (false path) and Create or update an account (after Salesforce update)
- **Failure modes / edge cases:**
  - If batch size is >1, downstream nodes must handle multiple items per run; most nodes here do, but scraping/LLM cost increases.
  - Mis-wiring risk: this workflow uses the **second output** of SplitInBatches to go forward (common pattern, but easy to misconnect when rebuilding).

---

### 2.3 Scrape review pages (Decodo)
**Overview:** Fetches Trustpilot page HTML reliably using Decodo scraping infrastructure.

**Nodes involved:**
- Decodo

#### Node: Decodo
- **Type / role:** Decodo (`@decodo/n8n-nodes-decodo.decodo`) — web scraping/fetching.
- **Configuration choices:**
  - **URL:** `={{ $('Get row(s) in sheet').item.json.url }}`
    - Uses the current item from the Google Sheets read node (important: relies on `.item` scoping).
- **Connections:**
  - **Input ←** Loop Over Items
  - **Output →** Code in JavaScript (HTML cleaning)
- **Credentials:** Decodo API credential.
- **Failure modes / edge cases:**
  - Invalid URL / empty `url` field → request fails or returns error/empty content
  - Decodo auth errors (API key invalid)
  - Rate limiting / anti-bot changes / page structure changes
  - Returned payload shape differences (the next node expects either `{results:[{url, content}]}` or `{url, content}`)

---

### 2.4 HTML → clean text + metadata extraction
**Overview:** Normalizes Decodo output, extracts URL/domain/entity identifiers, removes HTML noise, and outputs a cleaned `page_text` used by the LLM.

**Nodes involved:**
- Code in JavaScript

#### Node: Code in JavaScript
- **Type / role:** Code (`n8n-nodes-base.code`) — parsing + cleaning + enrichment.
- **Key configuration choices:**
  - Removes `<script>`/`<style>`, converts block tags to newlines, strips remaining HTML tags.
  - Decodes some common HTML entities.
  - Produces:
    - `url`
    - `domain` (hostname without `www.`)
    - `entity_slug` extracted from Trustpilot-like `/review/<slug>`
    - `entity_name` derived from `<h1>` or `<title>` (cleaned), falls back to `entity_slug`
    - `page_text` (cleaned text)
    - `raw_html_length`
  - Supports Decodo payload shape:
    - If `json.results` exists, takes `json.results[0]`.
- **Expressions/variables used:** Pure JS functions; no n8n expressions inside code.
- **Connections:**
  - **Input ←** Decodo
  - **Output →** AI Agent
- **Failure modes / edge cases:**
  - If Decodo returns no `content/html`, `page_text` becomes empty; the LLM output will likely be mostly zeros/nulls.
  - `new URL(url)` can throw; handled by try/catch returning empty domain.
  - HTML parsing is regex-based; may miss some entity names or content on heavily scripted pages.

---

### 2.5 LLM aggregation (sentiment/KPIs JSON)
**Overview:** Sends the cleaned page text to an LLM agent instructed to return a single JSON object with metrics: ratings, sentiment distribution, keywords, topics, trends, and data-quality notes.

**Nodes involved:**
- OpenAI Chat Model
- AI Agent

#### Node: OpenAI Chat Model
- **Type / role:** LangChain OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM backend.
- **Configuration choices:**
  - **Model:** `gpt-4.1-nano`
  - No additional options/tools enabled.
- **Connections:**
  - **Output (ai_languageModel) →** AI Agent (as the model provider)
- **Credentials:** OpenAI API credential.
- **Failure modes / edge cases:**
  - Auth errors / quota exceeded
  - Model availability changes
  - Timeouts with very large `page_text`

#### Node: AI Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — runs the prompt and returns LLM output.
- **Configuration choices:**
  - **Prompt:** Injects:
    - `url` and `domain`
    - `page_text` (cleaned)
  - Strict instruction: **return ONLY a JSON object** with a fixed schema.
  - System message emphasizes conservative assumptions and strict JSON output.
- **Connections:**
  - **Input ←** Code in JavaScript
  - **Model input ←** OpenAI Chat Model
  - **Output →** Code in JavaScript1 (parse/flatten)
- **Failure modes / edge cases:**
  - The model may still output non-JSON or extra text; downstream parser handles this with `parse_error`.
  - If `page_text` is empty/noisy, metrics will be 0/null and `data_quality_notes` should describe limitations.

---

### 2.6 Flatten + persist to Google Sheets
**Overview:** Parses the agent output (string JSON), flattens nested objects/arrays into spreadsheet-friendly columns, then appends a row to the “output” sheet.

**Nodes involved:**
- Code in JavaScript1
- Append row in sheet

#### Node: Code in JavaScript1
- **Type / role:** Code (`n8n-nodes-base.code`) — JSON parsing + flattening.
- **Configuration choices:**
  - Reads `item.json.output` from the AI Agent result.
  - If output is a string: `JSON.parse`.
  - On parse failure: emits an item with:
    - `parse_error: true`, `error_message`, `original_output`
  - Flattens:
    - rating_distribution → `rating_1..rating_5`
    - sentiment_distribution → `sentiment_positive/neutral/negative`
    - topic_distribution → `topic_pricing/usability/features/support/performance/other`
    - arrays → comma-separated strings or newline-separated blocks
- **Connections:**
  - **Input ←** AI Agent
  - **Output →** Append row in sheet
- **Failure modes / edge cases:**
  - If AI Agent output field name differs (not `output`), parser will mark as unexpected type.
  - If parse_error items are passed to the sheet append, columns used there may be empty.

#### Node: Append row in sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — append analytics row.
- **Configuration choices:**
  - **Operation:** Append
  - **Document:** same spreadsheet `url_feedback_truspilot`
  - **Sheet/tab:** `output` (`gid=697979134`)
  - **Column mapping:** explicitly maps:
    - `source_url`, `estimated_total_reviews`, `average_rating`
    - `sentiment_positive`, `sentiment_neutral`, `sentiment_negative`
    - `top_positive_keywords`, `top_negative_keywords`
    - `recent_trend_summary`
  - Does **not** write all flattened fields (e.g., topic/rating buckets) even though they exist.
- **Connections:**
  - **Input ←** Code in JavaScript1
  - **Output →** If (Salesforce gating)
- **Credentials:** Google Sheets OAuth2.
- **Failure modes / edge cases:**
  - Permission issues, sheet/tab mismatch
  - Schema mismatch if the output sheet headers differ from mapped column IDs
  - If `parse_error` item arrives, it may append mostly blank cells (no guard exists)

---

### 2.7 Conditional Salesforce update (truncate + upsert)
**Overview:** If a Salesforce Account ID is present in the current input row, the workflow truncates the trend summary to fit a 255-character Salesforce text field and upserts the Account’s custom feedback field.

**Nodes involved:**
- If
- Code in JavaScript2
- Create or update an account

#### Node: If
- **Type / role:** IF (`n8n-nodes-base.if`) — conditional routing.
- **Condition (interpreted):**
  - Checks that `salesforce_account_id` from the current “Get row(s) in sheet” item is **not equal to** empty string:
    - `={{ $('Get row(s) in sheet').item.json.salesforce_account_id }} != ""`
- **Connections:**
  - **Input ←** Append row in sheet
  - **True →** Code in JavaScript2
  - **False →** Loop Over Items (to continue loop)
- **Failure modes / edge cases:**
  - If the field is missing (undefined), strict validation may treat it differently depending on n8n version; most often it becomes `""` or `null`. Here it compares to `""`.
  - If whitespace IDs exist (e.g. `" "`), condition passes but Salesforce will fail.

#### Node: Code in JavaScript2
- **Type / role:** Code (`n8n-nodes-base.code`) — Salesforce field safety.
- **Configuration choices:**
  - Truncates `recent_trend_summary` to **<= 255 chars**, tries not to cut mid-word (cuts at last space if >200).
  - Outputs `{ recent_trend_summary: safeTrendSummary }` only.
- **Connections:**
  - **Input ←** If (true)
  - **Output →** Create or update an account
- **Failure modes / edge cases:**
  - If `recent_trend_summary` is missing, returns empty string (fine).
  - Only truncates the summary; other Salesforce fields are not set here.

#### Node: Create or update an account
- **Type / role:** Salesforce (`n8n-nodes-base.salesforce`) — upsert existing account.
- **Configuration choices:**
  - **Resource:** Account
  - **Operation:** Upsert
  - **External ID field:** `Id` (Salesforce native record ID)
  - **External ID value:** `={{ $('Get row(s) in sheet').item.json.salesforce_account_id }}`
  - **Name field:** `={{ $('Code in JavaScript').item.json.entity_slug }}`
    - Note: setting `Name` during an upsert by `Id` can overwrite the Account name if allowed.
  - **Custom field update:** sets `feedback__c` to `={{ $json.recent_trend_summary }}`
    - Assumes a custom Account field API name `feedback__c` exists and is writable.
- **Connections:**
  - **Input ←** Code in JavaScript2
  - **Output →** Loop Over Items (continue loop)
- **Credentials:** Salesforce OAuth2.
- **Failure modes / edge cases:**
  - Invalid/empty Account Id → upsert fails
  - Field-level security: cannot write `feedback__c` or `Name`
  - If `feedback__c` is not Text(255) (or is longer/shorter), truncation may be unnecessary or insufficient
  - Using `Id` as “externalId” is atypical but valid for update semantics; ensure node supports upsert-by-Id in your n8n/Salesforce node version

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get row(s) in sheet | ## Input  \n![txt](https://ik.imagekit.io/agbb7sr41/input_job_2-png.png) |
| Get row(s) in sheet | Google Sheets | Read Trustpilot URLs + Salesforce Account IDs | When clicking ‘Execute workflow’ | Loop Over Items | ## Input  \n![txt](https://ik.imagekit.io/agbb7sr41/input_job_2-png.png) |
| Loop Over Items | Split in Batches | Per-row iteration / looping | Get row(s) in sheet; If (false); Create or update an account | Decodo | ##  Data Extraction  \nReads Trustpilot URLs and Salesforce Account IDs from Google Sheets and controls the execution flow |
| Decodo | Decodo | Scrape Trustpilot page content | Loop Over Items | Code in JavaScript | ## AI Analysis  \nScrapes Trustpilot reviews via **Decodo**, cleans the text, and generates sentiment insights using OpenAI |
| Code in JavaScript | Code | Clean HTML, extract metadata (domain/slug/name), output page_text | Decodo | AI Agent | ## AI Analysis  \nScrapes Trustpilot reviews via **Decodo**, cleans the text, and generates sentiment insights using OpenAI |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM backend for agent | — | AI Agent | ## AI Analysis  \nScrapes Trustpilot reviews via **Decodo**, cleans the text, and generates sentiment insights using OpenAI |
| AI Agent | LangChain Agent | Prompt LLM to output strict KPI JSON | Code in JavaScript; OpenAI Chat Model | Code in JavaScript1 | ## AI Analysis  \nScrapes Trustpilot reviews via **Decodo**, cleans the text, and generates sentiment insights using OpenAI |
| Code in JavaScript1 | Code | Parse agent JSON output and flatten for storage | AI Agent | Append row in sheet | ## CRM + Google sheets update\nUpdates the Salesforce Account with the sentiment summary and stores full analytics in Google Sheets |
| Append row in sheet | Google Sheets | Append metrics row to output tab | Code in JavaScript1 | If | ## CRM + Google sheets update\nUpdates the Salesforce Account with the sentiment summary and stores full analytics in Google Sheets |
| If | IF | Only update Salesforce when Account ID exists | Append row in sheet | Code in JavaScript2 (true); Loop Over Items (false) | ## CRM + Google sheets update\nUpdates the Salesforce Account with the sentiment summary and stores full analytics in Google Sheets |
| Code in JavaScript2 | Code | Truncate summary to 255 chars for Salesforce field | If (true) | Create or update an account | ## CRM + Google sheets update\nUpdates the Salesforce Account with the sentiment summary and stores full analytics in Google Sheets |
| Create or update an account | Salesforce | Upsert Account by Id and write feedback__c | Code in JavaScript2 | Loop Over Items | ## Data Output  \n![txt](https://ik.imagekit.io/agbb7sr41/salesforce_feedback.png)\n![txt](https://ik.imagekit.io/agbb7sr41/n8n_output_2.png) |
| Sticky Note | Sticky Note | Documentation block | — | — | ## What this workflow does\nThis workflow scrapes **customer reviews from Trustpilot**, analyzes them with AI, and keeps both **Salesforce** and **Google Sheets** automatically updated with customer sentiment insights.\n\nIt uses **Decodo** to reliably extract review content from Trustpilot, processes the text with **OpenAI**, and orchestrates everything using **n8n**.\n\nDecodo (scraping layer):  \n👉 https://visit.decodo.com/raqXGD\n\n\n## How it works (high level)\n\n1. Reads **Trustpilot review URLs** and **Salesforce Account IDs** from Google Sheets  \n2. Scrapes Trustpilot reviews using **Decodo**  \n3. Uses AI to summarize sentiment, trends, and key positives/negatives  \n4. Generates **two outputs in parallel**  \n\n## Outputs generated\n\n### 1. Salesforce Account update  \nThe workflow updates an **existing Salesforce Account** by writing the AI-generated sentiment summary into a **custom text field** (e.g. `recent_trend_summary__c`).\n\nThis brings external customer feedback directly into Salesforce, so teams can:\n- See real customer sentiment inside the CRM  \n- Make faster, better-informed decisions  \n- Avoid manual review checks  \n\n### 2. Google Sheets analytics dataset  \nAt the same time, the workflow appends structured review metrics into Google Sheets, including:\n- Estimated total reviews  \n- Average rating  \n- Positive / neutral / negative sentiment counts  \n- Top positive and negative keywords  \n- Trend summary  \n\nThis sheet becomes a **central analytics layer** for dashboards, reporting, and historical comparison.\n\n\n## How to configure it (general)\n\n- **Google Sheets**: provide review URLs + Salesforce Account IDs  \n- **Decodo**: add your API key to scrape Trustpilot reliably  \n  👉 https://visit.decodo.com/raqXGD  \n- **OpenAI**: add your API key for sentiment summarization  \n- **Salesforce**:\n  - Create a custom **Text (255)** field on Account\n  - Connect Salesforce credentials in n8n\n  - The workflow updates existing Accounts only (no creation) |
| Sticky Note1 | Sticky Note | Documentation label | — | — | ##  Data Extraction  \nReads Trustpilot URLs and Salesforce Account IDs from Google Sheets and controls the execution flow |
| Sticky Note2 | Sticky Note | Documentation label | — | — | ## AI Analysis  \nScrapes Trustpilot reviews via **Decodo**, cleans the text, and generates sentiment insights using OpenAI |
| Sticky Note3 | Sticky Note | Documentation label | — | — | ## CRM + Google sheets update\nUpdates the Salesforce Account with the sentiment summary and stores full analytics in Google Sheets |
| Sticky Note4 | Sticky Note | Documentation label | — | — | ## Input  \n![txt](https://ik.imagekit.io/agbb7sr41/input_job_2-png.png) |
| Sticky Note5 | Sticky Note | Documentation label | — | — | ## Data Output  \n![txt](https://ik.imagekit.io/agbb7sr41/salesforce_feedback.png)\n![txt](https://ik.imagekit.io/agbb7sr41/n8n_output_2.png) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n named: `LLM-Driven Review Summary & KPI Extractor`.

2) **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3) **Add Google Sheets (input read)**
   - Node: **Google Sheets**
   - Name: `Get row(s) in sheet`
   - **Credentials:** Google Sheets OAuth2 (connect the Google account that can access the spreadsheet)
   - **Document:** select spreadsheet `url_feedback_truspilot`
   - **Sheet:** select tab `input`
   - Ensure your `input` sheet has (at minimum) columns:
     - `url`
     - `salesforce_account_id` (can be blank)

4) **Add Split in Batches**
   - Node: **Split in Batches**
   - Name: `Loop Over Items`
   - Keep default batch size (or set to 1 for controlled cost).
   - Connect: **Manual Trigger → Get row(s) in sheet → Loop Over Items**

5) **Add Decodo scraper**
   - Node: **Decodo**
   - Name: `Decodo`
   - **Credentials:** Decodo API credential (API key)
   - **URL field:** set expression:
     - `{{ $('Get row(s) in sheet').item.json.url }}`
   - Connect: **Loop Over Items → Decodo**

6) **Add Code node to clean HTML**
   - Node: **Code**
   - Name: `Code in JavaScript`
   - Paste the HTML cleaning + extraction code (functions `htmlToText`, `getDomain`, `extractEntitySlugFromUrl`, etc.).
   - Connect: **Decodo → Code in JavaScript**

7) **Add OpenAI chat model**
   - Node: **OpenAI Chat Model** (LangChain)
   - Name: `OpenAI Chat Model`
   - **Credentials:** OpenAI API key
   - **Model:** `gpt-4.1-nano`

8) **Add AI Agent**
   - Node: **AI Agent** (LangChain Agent)
   - Name: `AI Agent`
   - Set the **System Message** to the provided “specialized in analyzing customer reviews…” content.
   - Set the **User prompt** to the provided schema/prompt that injects:
     - `url`, `domain`, and `page_text` via expressions.
   - In the AI Agent node, connect the **Language Model** input to `OpenAI Chat Model`.
   - Connect: **Code in JavaScript → AI Agent**

9) **Add Code node to parse + flatten**
   - Node: **Code**
   - Name: `Code in JavaScript1`
   - Paste the parser/flatten code that reads `item.json.output`, parses JSON, and outputs flat fields.
   - Connect: **AI Agent → Code in JavaScript1**

10) **Add Google Sheets (output append)**
   - Node: **Google Sheets**
   - Name: `Append row in sheet`
   - **Credentials:** same Google Sheets OAuth2 (or another permitted account)
   - **Operation:** Append
   - **Document:** `url_feedback_truspilot`
   - **Sheet:** `output`
   - Map columns (define below) using expressions from the flattened item:
     - `source_url` → `{{ $json.source_url }}`
     - `estimated_total_reviews` → `{{ $json.estimated_total_reviews }}`
     - `average_rating` → `{{ $json.average_rating }}`
     - `sentiment_positive` → `{{ $json.sentiment_positive }}`
     - `sentiment_neutral` → `{{ $json.sentiment_neutral }}`
     - `sentiment_negative` → `{{ $json.sentiment_negative }}`
     - `top_positive_keywords` → `{{ $json.top_positive_keywords }}`
     - `top_negative_keywords` → `{{ $json.top_negative_keywords }}`
     - `recent_trend_summary` → `{{ $json.recent_trend_summary }}`
   - Connect: **Code in JavaScript1 → Append row in sheet**

11) **Add IF node (Salesforce gating)**
   - Node: **IF**
   - Name: `If`
   - Condition: String **notEquals**
     - Left value: `{{ $('Get row(s) in sheet').item.json.salesforce_account_id }}`
     - Right value: empty string
   - Connect: **Append row in sheet → If**

12) **Add Code node to truncate Salesforce summary**
   - Node: **Code**
   - Name: `Code in JavaScript2`
   - Paste truncation logic (truncate to 255, avoid cutting words).
   - Connect: **If (true) → Code in JavaScript2**

13) **Add Salesforce upsert**
   - Node: **Salesforce**
   - Name: `Create or update an account`
   - **Credentials:** Salesforce OAuth2 (connected user must have rights to edit Accounts and the custom field)
   - **Resource:** Account
   - **Operation:** Upsert
   - **External ID field:** `Id`
   - **External ID value:** `{{ $('Get row(s) in sheet').item.json.salesforce_account_id }}`
   - **Name field:** `{{ $('Code in JavaScript').item.json.entity_slug }}`
   - **Custom field:** set `feedback__c` to `{{ $json.recent_trend_summary }}`
     - In Salesforce, create this field on Account if it doesn’t exist:
       - API name: `feedback__c`
       - Type: **Text (255)** (matches truncation strategy)
   - Connect: **Code in JavaScript2 → Create or update an account**

14) **Close the loop**
   - Connect **Create or update an account → Loop Over Items** (continue to next row)
   - Connect **If (false) → Loop Over Items** (skip Salesforce update and continue)

15) **(Optional but recommended) Add protections**
   - Add an IF before Decodo to skip empty URLs.
   - Add an IF after `Code in JavaScript1` to route `parse_error` items to a separate “errors” sheet.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Decodo (scraping layer): 👉 https://visit.decodo.com/raqXGD | Sticky note “What this workflow does” |
| Output screenshots: https://ik.imagekit.io/agbb7sr41/salesforce_feedback.png | Sticky note “Data Output” |
| Output screenshots: https://ik.imagekit.io/agbb7sr41/n8n_output_2.png | Sticky note “Data Output” |
| Input screenshot: https://ik.imagekit.io/agbb7sr41/input_job_2-png.png | Sticky note “Input” |