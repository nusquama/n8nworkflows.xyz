Audit web pages for SEO issues with Decodo, GPT-4 and Google Sheets

https://n8nworkflows.xyz/workflows/audit-web-pages-for-seo-issues-with-decodo--gpt-4-and-google-sheets-11402


# Audit web pages for SEO issues with Decodo, GPT-4 and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow audits a list of web pages for SEO issues. It reads URLs from a Google Sheet, scrapes each page with **Decodo**, extracts key on-page SEO elements from the HTML, asks **GPT‑4.1‑mini** to generate an executive-friendly SEO summary (as strict JSON), validates/repairs the JSON, then **emails** the report (Gmail) and **stores** the results in a Google Sheet.

**Target use cases:**
- Ongoing SEO monitoring (“watchdog”) for a set of important pages
- Fast executive reporting (non-technical summaries)
- Batch audits from a spreadsheet

### Logical Blocks
**1.1 Input Reception (Google Sheets → Batch loop)**  
Manual start → read URL list → iterate one by one.

**1.2 Web Scraping (Decodo)**  
Fetch HTML reliably per URL.

**1.3 SEO Extraction (Code)**  
Reduce HTML into a compact payload (title/meta/H1/H2/canonical/OG + visible text excerpt).

**1.4 AI Analysis (LangChain Agent + OpenAI model)**  
LLM produces an executive JSON report for each page.

**1.5 Output Processing (Parse/repair → Email → Save to Sheets → loop)**  
Parse AI JSON safely, send formatted email, append row to output sheet, continue next URL.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Google Sheets → batching)
**Overview:**  
Starts the workflow manually, fetches URLs from an input Google Sheet, and prepares them for one-by-one processing via Split in Batches.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Fetch URLs from Google Sheet
- Process URLs one by one

#### Node: **When clicking ‘Execute workflow’**
- **Type / role:** Manual Trigger; entry point for manual runs.
- **Configuration:** No parameters.
- **Connections:**  
  - **Output →** Fetch URLs from Google Sheet
- **Failure modes / edge cases:** None (only manual start).

#### Node: **Fetch URLs from Google Sheet**
- **Type / role:** Google Sheets node; reads the input dataset containing URLs to audit.
- **Configuration choices (interpreted):**
  - Connects to a spreadsheet named (cached) **“urls_to_scrape”**
  - Uses sheet tab **“Hoja 1”** (`gid=0`)
  - Uses “data location on sheet” with a specified range mechanism (range not explicitly shown in JSON; relies on node UI/config).
- **Key data assumption:** Each returned item has a field used later as `item.json.urls` (the workflow references `$('Fetch URLs from Google Sheet').item.json.urls`).
- **Connections:**  
  - **Input ←** Manual Trigger  
  - **Output →** Process URLs one by one
- **Version:** 4.7
- **Failure modes / edge cases:**
  - OAuth credential invalid/expired
  - Spreadsheet permissions (403), missing sheet/tab
  - The column name **must** be `urls` (or the expression references must be updated)
  - Empty sheet → no items → downstream nodes won’t run

#### Node: **Process URLs one by one**
- **Type / role:** SplitInBatches; controls per-URL processing loop.
- **Configuration choices:** Default batch options (batch size not set explicitly; n8n default behavior applies).
- **Connections:**  
  - **Input ←** Fetch URLs from Google Sheet  
  - **Output (main, index 1) →** Decodo  
  - **Receives loop-back input ←** Save SEO Report to Google Sheets1 (to proceed to next batch)
- **Version:** 3
- **Failure modes / edge cases:**
  - If there is no loop-back (or it breaks), only the first batch may run
  - If batch size is >1, Decodo/code/LLM will process multiple items per run; verify email/sheet logic is intended per item

---

### 2.2 Web Scraping (Decodo)
**Overview:**  
For each URL, fetches the HTML content using Decodo’s scraping infrastructure.

**Nodes involved:**
- Decodo

#### Node: **Decodo**
- **Type / role:** Decodo scraper node; retrieves page content (HTML).
- **Configuration choices:**
  - **URL:** `={{ $('Fetch URLs from Google Sheet').item.json.urls }}`
    - This references the current item’s `urls` field originating from the Google Sheets read.
- **Expected output shape:** The next node reads `results[0].content`, so Decodo is expected to return something like:
  - `$json.results[0].content` = HTML string
- **Connections:**  
  - **Input ←** Process URLs one by one  
  - **Output →** Extract SEO Elements from HTML
- **Version:** 1
- **Failure modes / edge cases:**
  - Invalid Decodo API credentials
  - URL blocked, timeout, non-HTML response, JS-heavy pages returning minimal HTML
  - Output structure changes (if Decodo returns a different schema, `results[0].content` will be undefined)
  - Rate limits / quota exhaustion

---

### 2.3 SEO Extraction (HTML → compact SEO payload)
**Overview:**  
Transforms raw HTML into a compact, LLM-friendly text plus structured fields (title, meta description, canonical, H1/H2, OG tags, and a visible text excerpt).

**Nodes involved:**
- Extract SEO Elements from HTML

#### Node: **Extract SEO Elements from HTML**
- **Type / role:** Code node (JavaScript); parses key SEO elements using regex and text cleanup.
- **Configuration choices (interpreted):**
  - Reads HTML from: `const html = $json?.results?.[0]?.content;`
  - Extracts:
    - `<title>`
    - `<meta name="description" content="...">` (supports both attribute orders)
    - `<link rel="canonical" href="...">` (supports both attribute orders)
    - `og:title`, `og:description` meta tags (supports both attribute orders)
    - First `<h1>` and up to 15 `<h2>` tags
  - Builds:
    - `seo_fields` object containing extracted fields + `text_excerpt` (first 6000 chars of visible text)
    - `seo_compact_text`: a newline-delimited summary used for LLM input
  - If HTML missing/not string:
    - Outputs `seo_compact_text: "NO_HTML_INPUT"` and null fields
- **Key outputs:**
  - `json.seo_compact_text`
  - `json.seo_fields` (object)
- **Connections:**  
  - **Input ←** Decodo  
  - **Output →** Generate AI SEO Executive Summary
- **Version:** 2
- **Failure modes / edge cases:**
  - Regex parsing can miss data on malformed HTML or unusual attribute quoting
  - Multiple `<h1>`: only first captured
  - Meta tags with no quotes, single quotes, or unusual spacing may be missed
  - Very large HTML: although excerpt is truncated, initial `stripTags` still processes entire document (potential performance hit)
  - Non-UTF8 entities: decoder is basic; may not decode all entities

---

### 2.4 AI Analysis (LLM executive SEO summary)
**Overview:**  
Uses an LLM (OpenAI chat model) through a LangChain Agent node to generate a strict JSON executive summary.

**Nodes involved:**
- OpenAI Chat Model
- Generate AI SEO Executive Summary

#### Node: **OpenAI Chat Model**
- **Type / role:** LangChain OpenAI chat model provider for the agent.
- **Configuration choices:**
  - Model: **gpt-4.1-mini**
  - No special options set.
- **Connections:**  
  - **AI languageModel output →** Generate AI SEO Executive Summary (as the agent’s model)
- **Version:** 1.3
- **Failure modes / edge cases:**
  - OpenAI credential/auth errors
  - Model availability / name changes
  - Rate limits, token limits (excerpt is capped at 6000 chars to reduce risk, but still may be large with system instructions)

#### Node: **Generate AI SEO Executive Summary**
- **Type / role:** LangChain Agent; prompts the LLM with the compact SEO extract and demands JSON output.
- **Configuration choices:**
  - **Text input:** `={{ $json.seo_compact_text }}`
  - **System message:** instructs executive tone and **returns ONLY valid JSON** with schema:
    - `overall_status`: `"good"|"needs_attention"|"critical"`
    - `top_issues`: array max 5
    - `quick_wins`: array max 5
    - `title_recommendation`: string or null
    - `meta_description_recommendation`: string or null
    - `notes`: string or null
  - Rules: short, no markdown, no emojis; mention missing info in notes.
- **Connections:**  
  - **Main input ←** Extract SEO Elements from HTML  
  - **AI model input ←** OpenAI Chat Model  
  - **Main output →** Parse & Validate AI JSON Output
- **Version:** 3
- **Failure modes / edge cases:**
  - LLM may still output non-JSON (extra text, code fences, trailing commas) → handled by the next parsing node
  - If `seo_compact_text` is `"NO_HTML_INPUT"`, the model should report missing info; ensure it does not hallucinate page content

---

### 2.5 Output Processing (parse/repair → email → Sheets → loop)
**Overview:**  
Normalizes the LLM output into valid JSON, then sends an HTML email and appends the results to an output Google Sheet. Finally, loops back to process the next URL.

**Nodes involved:**
- Parse & Validate AI JSON Output
- Send SEO Report by Email
- Save SEO Report to Google Sheets1

#### Node: **Parse & Validate AI JSON Output**
- **Type / role:** Code node (JavaScript); extracts the first `{...}` JSON object from the LLM output, strips code fences, and attempts minor repairs.
- **Configuration choices (interpreted):**
  - Reads raw from `item.json.output` (preferred) or falls back to `item.json`
  - Strips starting/ending triple backticks (```json … ```)
  - Extracts the first JSON object by slicing from first `{` to last `}`
  - Repair attempt if JSON.parse fails:
    - Removes BOM
    - Removes trailing commas before `}` or `]`
  - Output is a single item `{ json: parsedObject }`
- **Connections:**  
  - **Input ←** Generate AI SEO Executive Summary  
  - **Output →** Send SEO Report by Email
- **Version:** 2
- **Failure modes / edge cases:**
  - If the LLM response contains multiple JSON objects or braces in text, “first `{` to last `}`” can capture invalid spans
  - If the agent returns an array instead of an object, extraction logic may break expectations
  - If no braces exist → throws error and stops execution

#### Node: **Send SEO Report by Email**
- **Type / role:** Gmail node; sends an HTML-formatted executive summary email.
- **Configuration choices:**
  - **To:** `user@example.com` (placeholder; should be replaced)
  - **Subject:** `=SEO Watchdog Report — {{$now}}`
  - **Message:** A prewritten **static HTML template** (currently hardcoded example content, including emojis in headings like “🔎” and “⚡”).
    - Important: This template is not dynamically mapped to the AI JSON fields in the provided workflow.
- **Connections:**  
  - **Input ←** Parse & Validate AI JSON Output  
  - **Output →** Save SEO Report to Google Sheets1
- **Version:** 2.2
- **Failure modes / edge cases:**
  - Gmail OAuth expired/invalid
  - Sending limits, blocked “from”, or restricted scopes
  - Because content is static, recipients may receive the same example report for every URL unless you replace it with expressions

#### Node: **Save SEO Report to Google Sheets1**
- **Type / role:** Google Sheets append row; persists the AI results (and the audited URL) to an output spreadsheet.
- **Configuration choices (interpreted):**
  - Operation: **Append**
  - Spreadsheet (cached): **“SEO Analyzer”**
  - Sheet tab: **“Hoja 1”** (`gid=0`)
  - Column mapping (explicit expressions):
    - `brand_url` → `{{ $('Fetch URLs from Google Sheet').item.json.urls }}`
    - `status` → `{{ $('Parse & Validate AI JSON Output').item.json.overall_status }}`
    - `top_issues` → `{{ $('Parse & Validate AI JSON Output').item.json.top_issues }}`
    - `quick_wins` → `{{ $('Parse & Validate AI JSON Output').item.json.quick_wins }}`
    - `title_recommendation` → `{{ $('Parse & Validate AI JSON Output').item.json.title_recommendation }}`
    - `meta_description` → `{{ $('Parse & Validate AI JSON Output').item.json.meta_description_recommendation }}`
    - `notes` → `{{ $('Parse & Validate AI JSON Output').item.json.notes }}`
  - **Note on arrays:** `top_issues` and `quick_wins` are arrays. Google Sheets often expects a string; with `convertFieldsToString: false`, n8n may still stringify or may error depending on node behavior/version. Safest is to join arrays explicitly.
- **Connections:**  
  - **Input ←** Send SEO Report by Email  
  - **Output →** Process URLs one by one (loop continuation)
- **Version:** 4.7
- **Failure modes / edge cases:**
  - OAuth/permissions, missing document/tab
  - Column name mismatches between node mapping and actual sheet headers
  - Array-to-cell mapping issues (may need `={{ $json.top_issues.join('; ') }}`-style conversion)
  - If you want the URL from the *current batch item*, referencing `$('Fetch URLs...').item` can be fragile; prefer using the current item context from SplitInBatches

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Fetch URLs from Google Sheet |  |
| Fetch URLs from Google Sheet | Google Sheets | Read list of URLs to audit | When clicking ‘Execute workflow’ | Process URLs one by one | ## AI SEO Watchdog — Overview & Configuration; ## Input![txt](https://ik.imagekit.io/agbb7sr41/input.png) |
| Process URLs one by one | SplitInBatches | Iterate URLs sequentially | Fetch URLs from Google Sheet; Save SEO Report to Google Sheets1 (loop-back) | Decodo | ## AI SEO Watchdog — Overview & Configuration; ## Input![txt](https://ik.imagekit.io/agbb7sr41/input.png) |
| Decodo | Decodo | Scrape HTML content per URL | Process URLs one by one | Extract SEO Elements from HTML | ###  Content Extraction; URLs are read from Google Sheets.; Each page is scraped using Decodo to reliably fetch the HTML content. |
| Extract SEO Elements from HTML | Code | Parse/condense HTML into SEO elements + excerpt | Decodo | Generate AI SEO Executive Summary | ###  AI Analysis; JavaScript reduces the HTML to key SEO elements.; The AI Agent analyzes the data and generates an executive SEO summary. |
| OpenAI Chat Model | LangChain OpenAI Chat Model | LLM provider for agent | — | Generate AI SEO Executive Summary (AI model input) | ###  AI Analysis; JavaScript reduces the HTML to key SEO elements.; The AI Agent analyzes the data and generates an executive SEO summary. |
| Generate AI SEO Executive Summary | LangChain Agent | Produce executive JSON SEO report | Extract SEO Elements from HTML; OpenAI Chat Model | Parse & Validate AI JSON Output | ###  AI Analysis; JavaScript reduces the HTML to key SEO elements.; The AI Agent analyzes the data and generates an executive SEO summary. |
| Parse & Validate AI JSON Output | Code | Strip fences/extract/repair JSON from LLM output | Generate AI SEO Executive Summary | Send SEO Report by Email | ### Data Output; Results are saved to Google Sheets.; A formatted SEO summary is sent by email using Gmail. |
| Send SEO Report by Email | Gmail | Email executive summary (HTML) | Parse & Validate AI JSON Output | Save SEO Report to Google Sheets1 | ## Email; ![txt](https://ik.imagekit.io/agbb7sr41/send_email.png) |
| Save SEO Report to Google Sheets1 | Google Sheets | Append report row to output sheet | Send SEO Report by Email | Process URLs one by one (loop-back) | ## Google sheet; ![txt](https://ik.imagekit.io/agbb7sr41/output_seo_watachdog.png) |

Sticky notes not tied to execution:
- **📝 OVERVIEW**, **📝 DECODE NOTE**, **📝 AI AGENT NOTE**, **📝 PARSE & REPAIR**, plus the image-based sticky notes.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Manual Trigger**
   - Name: “When clicking ‘Execute workflow’”.

3. **Add node: Google Sheets (Read)**
   - Name: “Fetch URLs from Google Sheet”
   - **Credentials:** Google Sheets OAuth2
   - **Document:** choose your input spreadsheet (e.g., `urls_to_scrape`)
   - **Sheet/Tab:** select the tab with URLs (e.g., `Hoja 1`)
   - Ensure the sheet has a column header named **`urls`** (one URL per row).
   - Connect: Manual Trigger → Fetch URLs from Google Sheet.

4. **Add node: Split In Batches**
   - Name: “Process URLs one by one”
   - Leave default options (or set Batch Size = 1 explicitly if desired).
   - Connect: Fetch URLs from Google Sheet → Process URLs one by one.

5. **Add node: Decodo**
   - Name: “Decodo”
   - **Credentials:** Decodo API credential
   - **URL field:** set expression to the current URL field, ideally:
     - `={{ $json.urls }}`
     - (If you keep the original approach, it used `$('Fetch URLs from Google Sheet').item.json.urls`, but `$json.urls` is typically safer inside a batch loop.)
   - Connect: Process URLs one by one → Decodo.

6. **Add node: Code (JavaScript)**
   - Name: “Extract SEO Elements from HTML”
   - Paste the extraction code logic (title/meta/canonical/H1/H2/OG + visible text excerpt).
   - Confirm it reads from the Decodo output path: `$json.results[0].content`.
   - Connect: Decodo → Extract SEO Elements from HTML.

7. **Add node: OpenAI Chat Model (LangChain)**
   - Name: “OpenAI Chat Model”
   - **Credentials:** OpenAI API
   - **Model:** `gpt-4.1-mini` (or your preferred model).
   - No main connection needed; it will connect to the agent via the AI port.

8. **Add node: AI Agent (LangChain Agent)**
   - Name: “Generate AI SEO Executive Summary”
   - **Text input:** `={{ $json.seo_compact_text }}`
   - **System message:** set to the workflow’s constraints requiring ONLY valid JSON with the specified schema.
   - Connect:
     - Extract SEO Elements from HTML → Generate AI SEO Executive Summary (main)
     - OpenAI Chat Model → Generate AI SEO Executive Summary (AI languageModel connection)

9. **Add node: Code (JavaScript)**
   - Name: “Parse & Validate AI JSON Output”
   - Add code that:
     - Strips ``` fences
     - Extracts first JSON object
     - Parses JSON and applies small repairs (remove trailing commas, etc.)
   - Connect: Generate AI SEO Executive Summary → Parse & Validate AI JSON Output.

10. **Add node: Gmail (Send)**
   - Name: “Send SEO Report by Email”
   - **Credentials:** Gmail OAuth2
   - **To:** set your recipient(s)
   - **Subject:** `SEO Watchdog Report — {{$now}}`
   - **Message:** set to HTML.
   - (Recommended) Replace the static template with expressions that inject:
     - `{{$json.overall_status}}`, loop through `top_issues`, `quick_wins`, etc.
   - Connect: Parse & Validate AI JSON Output → Send SEO Report by Email.

11. **Add node: Google Sheets (Append)**
   - Name: “Save SEO Report to Google Sheets1”
   - **Credentials:** Google Sheets OAuth2
   - **Document:** choose your output spreadsheet (e.g., `SEO Analyzer`)
   - **Sheet/Tab:** output tab (e.g., `Hoja 1`)
   - **Operation:** Append
   - **Map columns** to AI output fields and URL:
     - brand_url: `={{ $json.urls }}` (preferred) or carry URL forward with a Merge/Set node if needed
     - status: `={{ $json.overall_status }}`
     - top_issues: `={{ Array.isArray($json.top_issues) ? $json.top_issues.join('; ') : $json.top_issues }}`
     - quick_wins: `={{ Array.isArray($json.quick_wins) ? $json.quick_wins.join('; ') : $json.quick_wins }}`
     - title_recommendation: `={{ $json.title_recommendation }}`
     - meta_description: `={{ $json.meta_description_recommendation }}`
     - notes: `={{ $json.notes }}`
   - Connect: Send SEO Report by Email → Save SEO Report to Google Sheets1.

12. **Close the loop**
   - Connect: Save SEO Report to Google Sheets1 → Process URLs one by one
   - This continues processing until all URLs are audited.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Decodo – Web Scraper for n8n | https://visit.decodo.com/raqXGD |
| Workflow sends HTML email through Gmail; current email body is hardcoded example text and should be mapped to AI JSON fields for real reporting. | Gmail node “Send SEO Report by Email” |
| Image references in sticky notes: Input | https://ik.imagekit.io/agbb7sr41/input.png |
| Image references in sticky notes: Email | https://ik.imagekit.io/agbb7sr41/send_email.png |
| Image references in sticky notes: Google sheet output | https://ik.imagekit.io/agbb7sr41/output_seo_watachdog.png |
| Execution mode | Workflow is inactive; uses manual trigger but can be scheduled by replacing/adding a Cron trigger. |