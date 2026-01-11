Aggregate web product reviews and sentiment with Decodo and Google Gemini

https://n8nworkflows.xyz/workflows/aggregate-web-product-reviews-and-sentiment-with-decodo-and-google-gemini-12068


# Aggregate web product reviews and sentiment with Decodo and Google Gemini

## 1. Workflow Overview

**Purpose:**  
This workflow lets a Telegram user send a company name, then automatically searches the web for review pages (via Decodo Google Search), scrapes each discovered review URL (via Decodo scraping), cleans the HTML into readable text, runs per-page sentiment analysis with **Google Gemini**, aggregates all per-page results, and generates a concise final report returned to the user on Telegram.

**Target use cases:**
- Quick due diligence on brand reputation from publicly available review pages
- Lightweight social proof / customer feedback scan before outreach, partnerships, or procurement
- Automated “review sentiment snapshot” on-demand in chat

### 1.1 Input & Search
Receives a company name from Telegram, builds a search query, performs a Google search using Decodo.

### 1.2 URL Extraction & Scraping
Extracts all URLs from the Decodo search response (recursively), then scrapes each page content through Decodo.

### 1.3 Content Cleaning
Removes HTML tags/scripts/styles and normalizes whitespace to get a clean text block for LLM analysis.

### 1.4 Sentiment Analysis (per page, loop)
Iterates through each scraped page, uses a Gemini-backed AI Agent to classify sentiment, score, and topics.

### 1.5 Aggregation & Final Report
Combines all sentiment outputs into one aggregated dataset, asks Gemini to produce a strict bullet report, then sends it back via Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Search

**Overview:**  
Triggers on incoming Telegram messages, converts the user’s text into a Google search query, and runs Decodo Google Search to find review-related pages.

**Nodes involved:**
- Get Message
- Set Search Query
- Google Search1

#### Node: **Get Message**
- **Type / Role:** Telegram Trigger (`n8n-nodes-base.telegramTrigger`) — workflow entry point.
- **Configuration (interpreted):**
  - Listens for Telegram `message` updates.
  - Requires Telegram bot credentials in n8n (not shown in JSON).
- **Key data used:**
  - Incoming company name is read from: `$json.message.text`
  - Chat id used later: `$('Get Message').first().json.message.chat.id`
- **Outputs:** One item per Telegram message event, containing `message` object.
- **Failure/edge cases:**
  - Bot not connected / invalid token → trigger never fires.
  - Non-text messages (stickers, photos) → `$json.message.text` may be undefined.
  - Very long messages → may create overly broad/expensive searches.

#### Node: **Set Search Query**
- **Type / Role:** Set (`n8n-nodes-base.set`) — prepares the query string.
- **Configuration (interpreted):**
  - Creates field `query` from the Telegram message text:
    - `query = "{{ $json.message.text }} reviews"`
- **Inputs:** Get Message
- **Outputs:** Same item augmented with `query`.
- **Failure/edge cases:**
  - If `message.text` is missing → query becomes `"undefined reviews"`.
  - Consider trimming/validating user input or adding a guard (IF node) for empty text.

#### Node: **Google Search1**
- **Type / Role:** Decodo (`@decodo/n8n-nodes-decodo.decodo`) — performs Google search via Decodo.
- **Configuration (interpreted):**
  - **Operation:** `google_search`
  - **Query:** `={{ $json.query }}`
  - **Credentials:** `Decodo Credentials account` (Decodo API key)
- **Inputs:** Set Search Query
- **Outputs:** Decodo search results JSON (structure depends on Decodo API).
- **Failure/edge cases:**
  - Invalid/expired Decodo API key → 401/403.
  - Rate limits or quota exceeded.
  - Search results structure changes → downstream URL extraction may miss URLs.
  - Some results may be irrelevant or include aggregator/ads.

---

### Block 2 — URL Extraction & Scraping

**Overview:**  
Finds all URLs embedded anywhere in the Decodo search response (deep recursive scan), deduplicates, skips the first URL, then scrapes each URL with Decodo.

**Nodes involved:**
- Extract URLs
- Scrape Page Content1

#### Node: **Extract URLs**
- **Type / Role:** Code (`n8n-nodes-base.code`) — parses Decodo response and emits one item per URL.
- **Configuration (interpreted):**
  - Recursively traverses each item’s JSON, collecting any field named `url` with an `http` prefix.
  - Deduplicates (`new Set`).
  - **Skips the first URL** (`urls = urls.slice(1)`), assuming it is Decodo’s own search URL.
  - Returns: `[{ json: { url } }, ...]`
- **Inputs:** Google Search1
- **Outputs:** Many items; each item has `{ url: "https://..." }`.
- **Failure/edge cases:**
  - If Decodo response does not use `url` fields → zero URLs returned.
  - Skipping the first URL may accidentally drop a legitimate first result depending on response ordering.
  - URLs can include non-review pages; no filtering by domain/path/keywords.
  - Very large search response may generate many URLs → high scraping cost.

#### Node: **Scrape Page Content1**
- **Type / Role:** Decodo (`@decodo/n8n-nodes-decodo.decodo`) — fetches page content for each URL.
- **Configuration (interpreted):**
  - **URL:** `={{ $json.url }}`
  - Uses Decodo credentials (same as search).
- **Inputs:** Extract URLs (one item per URL)
- **Outputs:** Scrape result per URL (expected to include something like `results[0].content`).
- **Failure/edge cases:**
  - Target site blocks scraping; Decodo may return errors, empty content, or CAPTCHA pages.
  - Non-HTML content (PDF, JSON, etc.) → cleaner may produce poor output.
  - Timeouts or payload too large.
  - If `results[0].content` is missing, the next node returns empty text.

---

### Block 3 — Content Cleaning

**Overview:**  
Converts scraped HTML into plain text by removing scripts/styles/tags and normalizing entities and whitespace.

**Nodes involved:**
- Clean HTML

#### Node: **Clean HTML**
- **Type / Role:** Code (`n8n-nodes-base.code`) — sanitizes and extracts readable text.
- **Configuration (interpreted):**
  - **Mode:** `runOnceForEachItem`
  - Reads HTML from `item.results[0].content` (in a try/catch).
  - Removes:
    - `<script>...</script>`, `<style>...</style>`, `<noscript>...</noscript>`
    - Any HTML tags
  - Decodes some common HTML entities and collapses whitespace.
  - Outputs `{ text_clean: cleaned }`
- **Inputs:** Scrape Page Content1
- **Outputs:** One item per URL with `text_clean`.
- **Failure/edge cases:**
  - If scrape output schema differs, `results[0].content` may be undefined → empty `text_clean`.
  - No language detection; non-English reviews may affect sentiment classification.
  - Extremely long pages can exceed LLM context or raise cost—no truncation is applied.

---

### Block 4 — Sentiment Analysis (per page loop)

**Overview:**  
Iterates over cleaned pages and uses a Gemini-backed AI Agent to produce structured JSON sentiment results for each page.

**Nodes involved:**
- Loop Over Items
- Analyze Sentiment
- Google Gemini Chat Model

#### Node: **Loop Over Items**
- **Type / Role:** Split in Batches (`n8n-nodes-base.splitInBatches`) — controls per-item looping.
- **Configuration (interpreted):**
  - Uses default options (batch size not explicitly set in JSON).
  - Receives cleaned items and feeds them to sentiment analysis one-by-one (loop pattern).
- **Inputs:** Clean HTML
- **Outputs:**
  - Output 1 → Merge Results (when looping completes / or per-batch completion depending on n8n behavior)
  - Output 2 → Analyze Sentiment (per item/batch)
- **Failure/edge cases:**
  - If batch size defaults unexpectedly, you may process many items at once (cost).
  - Loop wiring is a classic pattern; misconfiguration can cause partial aggregation or unexpected execution order if modified.

#### Node: **Google Gemini Chat Model**
- **Type / Role:** LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides the LLM for the agent.
- **Configuration (interpreted):**
  - Uses default options (model/version not specified here; depends on credential defaults).
  - Connected via **AI Language Model** port to `Analyze Sentiment`.
- **Inputs/Outputs:** Not in main flow; supplies model to the agent node.
- **Failure/edge cases:**
  - Missing/invalid Google Gemini API credentials.
  - Model availability/region restrictions.
  - Content length limits; large `text_clean` may be truncated or error.

#### Node: **Analyze Sentiment**
- **Type / Role:** AI Agent (`@n8n/n8n-nodes-langchain.agent`) — structured sentiment extraction per page.
- **Configuration (interpreted):**
  - Prompt instructs: “You are a sentiment engine… return JSON”
  - Injects page text: `{{ $json.text_clean }}`
  - Expected schema:
    ```json
    {
      "sentiment": "positive | neutral | negative",
      "score": 0-1,
      "topics": ["..."]
    }
    ```
  - Uses **Google Gemini Chat Model** via AI connection.
- **Inputs:** Loop Over Items (current item)
- **Outputs:** Agent output stored under `item.json.output` (as used later by Merge Results).
- **Failure/edge cases:**
  - The agent may return non-JSON or malformed JSON (despite instruction).
  - Empty `text_clean` → vague or useless sentiment output.
  - Multi-topic pages (forums, mixed reviews) can yield inconsistent sentiment.
  - No explicit retry/fallback behavior defined.

---

### Block 5 — Aggregation & Final Report

**Overview:**  
Aggregates all per-page sentiment outputs into a single text blob, then uses Gemini to produce a concise, strict bullet report, and replies to the user in Telegram.

**Nodes involved:**
- Merge Results
- Draft Final Report
- Google Gemini Chat Model1
- Reply to User

#### Node: **Merge Results**
- **Type / Role:** Code (`n8n-nodes-base.code`) — combines all agent outputs.
- **Configuration (interpreted):**
  - Collects all incoming items: `$input.all()`
  - Builds `aggregated_report` string:
    - Header + per-item section `--- Search Result N ---`
    - Appends `item.json.output` or “No data”
  - Returns a **single item** with `{ aggregated_report }`
- **Inputs:** Loop Over Items (completion path)
- **Outputs:** One aggregated item to Draft Final Report
- **Failure/edge cases:**
  - If `Analyze Sentiment` output is not in `json.output`, aggregation becomes “No data”.
  - Large number of items → aggregated text may exceed LLM input limits.

#### Node: **Google Gemini Chat Model1**
- **Type / Role:** LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — LLM for final summarization.
- **Configuration (interpreted):**
  - Default options; connected to `Draft Final Report` via AI port.
- **Failure/edge cases:** Same as the other Gemini model node.

#### Node: **Draft Final Report**
- **Type / Role:** AI Agent (`@n8n/n8n-nodes-langchain.agent`) — generates the final Telegram-optimized report.
- **Configuration (interpreted):**
  - Strict instruction: “do not use flowery language; only extract facts”
  - Data source: `{{ $json.aggregated_report }}`
  - Constraints:
    - Use only provided data
    - Quote numbers/scores if present
    - Max 200 words
    - Bullet format for Telegram
  - Output sections:
    - Sentiment Score
    - Customer Voice (2 bullets)
    - Actionable Verdict (1 sentence)
- **Inputs:** Merge Results
- **Outputs:** `json.output` (used directly by Telegram send node)
- **Failure/edge cases:**
  - If aggregated data is unstructured or empty, the model may produce low-value output.
  - The “only use provided data” constraint is best-effort; LLMs can still infer—consider adding validation.

#### Node: **Reply to User**
- **Type / Role:** Telegram (`n8n-nodes-base.telegram`) — sends message back to the user.
- **Configuration (interpreted):**
  - **Text:** `={{ $json.output }}`
  - **Chat ID:** `={{ $('Get Message').first().json.message.chat.id }}`
- **Inputs:** Draft Final Report
- **Outputs:** Telegram API response (message sent)
- **Failure/edge cases:**
  - Invalid Telegram credentials / revoked bot token.
  - Chat ID missing if trigger payload differs.
  - Telegram message length limits; output may need truncation if expanded.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get Message | telegramTrigger | Entry point: receive company name from Telegram | — | Set Search Query | ## Input & Search<br>Receives the company name from Telegram and performs a Google search for relevant review pages using Decodo. |
| Set Search Query | set | Build Google query from Telegram text | Get Message | Google Search1 | ## Input & Search<br>Receives the company name from Telegram and performs a Google search for relevant review pages using Decodo. |
| Google Search1 | decodo | Google search via Decodo | Set Search Query | Extract URLs | ## Input & Search<br>Receives the company name from Telegram and performs a Google search for relevant review pages using Decodo. |
| Extract URLs | code | Recursively extract/dedupe URLs from search response | Google Search1 | Scrape Page Content1 | ## URL Extraction & Scraping<br>Extracts all review URLs from search results, fetches each page, and cleans the content for analysis. |
| Scrape Page Content1 | decodo | Scrape each discovered URL | Extract URLs | Clean HTML | ## URL Extraction & Scraping<br>Extracts all review URLs from search results, fetches each page, and cleans the content for analysis. |
| Clean HTML | code | Strip HTML/scripts/styles; normalize text | Scrape Page Content1 | Loop Over Items | ## URL Extraction & Scraping<br>Extracts all review URLs from search results, fetches each page, and cleans the content for analysis. |
| Loop Over Items | splitInBatches | Loop controller for per-page analysis + completion path | Clean HTML and Analyze Sentiment (loop back) | Analyze Sentiment; Merge Results | ## Sentiment Analysis<br>Processes each review page individually to classify sentiment, assign a score, and extract key topics. |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM provider for sentiment agent | — | Analyze Sentiment (AI port) | ## Sentiment Analysis<br>Processes each review page individually to classify sentiment, assign a score, and extract key topics. |
| Analyze Sentiment | langchain agent | Per-page sentiment JSON extraction | Loop Over Items | Loop Over Items (loop back) | ## Sentiment Analysis<br>Processes each review page individually to classify sentiment, assign a score, and extract key topics. |
| Merge Results | code | Aggregate all per-page outputs into one report string | Loop Over Items (completion output) | Draft Final Report | ## Final Report<br>Combines all sentiment results into a single summary and generates an actionable verdict sent via Telegram. |
| Google Gemini Chat Model1 | lmChatGoogleGemini | LLM provider for final report agent | — | Draft Final Report (AI port) | ## Final Report<br>Combines all sentiment results into a single summary and generates an actionable verdict sent via Telegram. |
| Draft Final Report | langchain agent | Produce final strict bullet summary | Merge Results | Reply to User | ## Final Report<br>Combines all sentiment results into a single summary and generates an actionable verdict sent via Telegram. |
| Reply to User | telegram | Send final message to Telegram chat | Draft Final Report | — | ## Final Report<br>Combines all sentiment results into a single summary and generates an actionable verdict sent via Telegram. |
| Sticky Note | stickyNote | Comment / grouping | — | — | ## Input & Search<br>Receives the company name from Telegram and performs a Google search for relevant review pages using Decodo. |
| Sticky Note1 | stickyNote | Comment / grouping | — | — | ## URL Extraction & Scraping<br>Extracts all review URLs from search results, fetches each page, and cleans the content for analysis. |
| Sticky Note2 | stickyNote | Comment / grouping | — | — | ## Sentiment Analysis<br>Processes each review page individually to classify sentiment, assign a score, and extract key topics. |
| Sticky Note3 | stickyNote | Comment / grouping | — | — | ## Final Report<br>Combines all sentiment results into a single summary and generates an actionable verdict sent via Telegram. |
| Sticky Note4 | stickyNote | Global explanation + setup/customization notes | — | — | ## How it works<br>When a user sends a company name to the Telegram bot, the workflow automatically performs a Google search for review-related pages using Decodo. All discovered URLs are extracted dynamically, regardless of their depth in the search response. Each page is then fetched and cleaned by removing HTML, scripts, and unnecessary markup to produce readable text.<br><br>The workflow processes each cleaned review page individually using an AI sentiment agent. For every source, it determines the sentiment (positive, neutral, or negative), assigns a confidence score, and extracts key discussion topics. All individual analyses are then merged into a single aggregated dataset.<br><br>A final AI analysis step reviews the combined data and generates a structured executive-style report. The report includes an overall sentiment score, direct customer voice observations, and a clear actionable verdict. The final result is sent back to the user via Telegram.<br><br>## Setup steps<br>1. Connect your Telegram Bot credentials.<br>2. Add your Decodo API key for Google search and page fetching.<br>3. Configure your Google Gemini API credentials.<br>4. Deploy the workflow and send a company name via Telegram to start analysis.<br><br>## Customization<br>You can adjust prompt strictness, sentiment thresholds, or limit the number of analyzed pages to control cost and depth. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Trigger**
   - Add node: **Telegram Trigger** → name it **Get Message**
   - Updates: **message**
   - Configure **Telegram credentials** (Bot Token) in n8n.

2) **Create “Set Search Query”**
   - Add node: **Set**
   - Add a string field:
     - Name: `query`
     - Value (expression): `{{ $json.message.text }} reviews`
   - Connect: **Get Message → Set Search Query**

3) **Add Decodo Google Search**
   - Add node: **Decodo**
   - Operation: **google_search**
   - Query (expression): `{{ $json.query }}`
   - Configure **Decodo API credentials** (API key).
   - Connect: **Set Search Query → Google Search1**

4) **Add URL extraction Code node**
   - Add node: **Code** → name **Extract URLs**
   - Paste logic that:
     - Recursively collects `url` fields starting with `http`
     - Deduplicates
     - Slices off first URL
     - Outputs one item per URL as `{ json: { url } }`
   - Connect: **Google Search1 → Extract URLs**

5) **Add Decodo scraper**
   - Add node: **Decodo** → name **Scrape Page Content1**
   - Operation: (default scrape/fetch by URL for Decodo node version)
   - URL (expression): `{{ $json.url }}`
   - Use same **Decodo credentials**
   - Connect: **Extract URLs → Scrape Page Content1**

6) **Add HTML cleaning Code node**
   - Add node: **Code** → name **Clean HTML**
   - Mode: **Run once for each item**
   - Implement:
     - Read from `results[0].content`
     - Strip scripts/styles/noscript and tags
     - Decode basic entities, normalize whitespace
     - Output: `{ text_clean: cleaned }`
   - Connect: **Scrape Page Content1 → Clean HTML**

7) **Add loop controller**
   - Add node: **Split in Batches** → name **Loop Over Items**
   - Set batch size as desired (not set in JSON; consider 5–10 to limit cost).
   - Connect: **Clean HTML → Loop Over Items**

8) **Add Google Gemini chat model for sentiment**
   - Add node: **Google Gemini Chat Model** (LangChain)
   - Configure **Gemini credentials** (Google AI Studio / Gemini API key in n8n).
   - Keep defaults unless you need a specific model.

9) **Add AI Agent for sentiment**
   - Add node: **AI Agent** → name **Analyze Sentiment**
   - Prompt type: **Define**
   - Prompt includes `{{ $json.text_clean }}`
   - Connect AI model port: **Google Gemini Chat Model → Analyze Sentiment (AI language model)**
   - Main connection: **Loop Over Items → Analyze Sentiment**
   - Loop back: **Analyze Sentiment → Loop Over Items** (so it continues to next item)

10) **Add aggregation Code node**
   - Add node: **Code** → name **Merge Results**
   - Collect all items (`$input.all()`), concatenate `item.json.output` into a single `aggregated_report` string.
   - Connect: **Loop Over Items (completion output) → Merge Results**
     - In n8n, use the appropriate SplitInBatches output that fires when done (matches the provided workflow wiring).

11) **Add Google Gemini chat model for final report**
   - Add node: **Google Gemini Chat Model** → name **Google Gemini Chat Model1**
   - Use same Gemini credentials (or separate if desired).

12) **Add AI Agent for final report**
   - Add node: **AI Agent** → name **Draft Final Report**
   - Prompt type: **Define**
   - Use `{{ $json.aggregated_report }}` as the only data source
   - Enforce max 200 words + bullet formatting for Telegram
   - Connect AI model port: **Google Gemini Chat Model1 → Draft Final Report**
   - Connect: **Merge Results → Draft Final Report**

13) **Send Telegram reply**
   - Add node: **Telegram** → name **Reply to User**
   - Chat ID (expression): `{{ $('Get Message').first().json.message.chat.id }}`
   - Text (expression): `{{ $json.output }}`
   - Connect: **Draft Final Report → Reply to User**

14) **(Optional but recommended) Add safeguards**
   - IF node after trigger: ensure `message.text` exists and is not empty.
   - Limit number of URLs analyzed (e.g., take first N).
   - Truncate `text_clean` to a safe character count to prevent LLM input overflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow flow: Telegram company name → Decodo Google search → recursive URL extraction → scrape each URL → clean HTML → per-page Gemini sentiment JSON → aggregate → Gemini final bullet report → Telegram reply. | Sticky Note “How it works” |
| Setup steps: connect Telegram Bot credentials; add Decodo API key; configure Google Gemini API credentials; deploy and send a company name to start. | Sticky Note “Setup steps” |
| Customization: adjust prompt strictness, sentiment thresholds, or limit number of analyzed pages to control cost and depth. | Sticky Note “Customization” |