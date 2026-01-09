Monitor eBay deals with GPT-4 scoring,Decodo & Telegram notifications

https://n8nworkflows.xyz/workflows/monitor-ebay-deals-with-gpt-4-scoring-decodo---telegram-notifications-10959


# Monitor eBay deals with GPT-4 scoring,Decodo & Telegram notifications

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow scrapes the **eBay Deals** page using **Decodo**, extracts a compact list of deal items in JavaScript (so **no raw HTML** is sent to the model), asks an **LLM (GPT-4.1-nano via LangChain)** to classify and score each deal, then applies **rule-based filtering** and sends **Telegram alerts** for high-quality deals.

**Target use cases:**
- Automated deal monitoring (consumer electronics, home, fashion, etc.)
- Low-token AI enrichment (structured text instead of HTML)
- Alerting via Telegram based on combined **price** + **AI score**

### 1.1 Triggers (Manual + Scheduled)
Two entry points start the same scraping flow:
- Manual execution for testing
- Scheduled execution every 3 hours for continuous monitoring

### 1.2 Scrape eBay Deals (Decodo)
Decodo fetches the eBay Deals HTML reliably (proxy/anti-bot support).

### 1.3 Pre-extract & Compact Payload (JavaScript)
A Code node parses the HTML, extracts up to ~60 “tile” items (ID/title/price/original price/url/image), and generates a compact, line-based text payload for the LLM.

### 1.4 AI Enrichment (LLM Agent)
The LangChain Agent sends the compact text to GPT-4.1-nano with a strict system message requiring **pure JSON output** containing:
- listing_id
- category_ai (tech|home|fashion|other)
- deal_score (0..10)
- score_reason (<=120 chars)

### 1.5 Merge Base + AI Output, Normalize, Compute Derived Fields
A Merge node combines base extraction + LLM output; then a JavaScript node joins them by `listing_id`, clamps/normalizes values, and computes:
- discount percentage
- per-category price thresholds
- `price_below_threshold` flag
No filtering happens here; it outputs **one n8n item per deal**.

### 1.6 Business Filtering + Telegram Notification
An If node filters deals with:
- `price < 100`
- `deal_score > 7`
Only passing items produce a Telegram message.

---

## 2. Block-by-Block Analysis

### Block A — Workflow Entry Points (Manual + Schedule)
**Overview:** Provides two ways to start the workflow: manual testing and periodic execution.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Schedule Trigger

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) used for interactive testing.
- **Key configuration:** No parameters.
- **Outputs:** Connects to **Decodo**.
- **Failure/edge cases:** None (only runs when manually executed).
- **Version notes:** typeVersion 1.

#### Node: Schedule Trigger
- **Type / role:** Scheduled trigger (`n8n-nodes-base.scheduleTrigger`) for automation.
- **Key configuration:** Runs every **3 hours** (`hoursInterval: 3`).
- **Outputs:** Connects to **Decodo**.
- **Failure/edge cases:** Schedule drift/disabled workflow; timezone considerations depending on n8n instance settings.
- **Version notes:** typeVersion 1.2.

---

### Block B — Scraping (Decodo)
**Overview:** Fetches the eBay Deals page HTML through Decodo to reduce bot-blocking issues.

**Nodes involved:**
- Decodo

#### Node: Decodo
- **Type / role:** Decodo scraper node (`@decodo/n8n-nodes-decodo.decodo`) to retrieve page content.
- **Key configuration (interpreted):**
  - **URL:** `https://www.ebay.es/deals`
  - **Geo:** `en` (Decodo geo/locale setting)
  - **Headless:** `false`
- **Credentials:** `decodoApi` (API key or similar).
- **Inputs:** From either trigger (manual or schedule).
- **Outputs:** Sends scrape result to **Code in JavaScript1**.
- **Failure/edge cases:**
  - Credential/auth failures (invalid Decodo API key)
  - Timeouts, blocked responses, changed page structure
  - Output structure changes (this workflow expects `$json.results[0].content`)
- **Version notes:** typeVersion 1.

---

### Block C — Pre-extraction & Compacting (JavaScript)
**Overview:** Converts large HTML into a compact structured list of items plus a token-efficient text payload for the LLM.

**Nodes involved:**
- Code in JavaScript1

#### Node: Code in JavaScript1
- **Type / role:** Code node (`n8n-nodes-base.code`) for HTML parsing and data extraction.
- **Key configuration (interpreted behavior):**
  - Reads HTML from **`$json.results[0].content`**
  - Finds deal tiles using a regex around `data-listing-id` and `dne-itemtile`
  - Extracts:
    - `listing_id`
    - `title` (from `dne-itemtile-title ... title="..."`)
    - `price` + `currency` (from `itemprop=price` and `priceCurrency`, plus heuristics for €/$)
    - `original_price` if a strikethrough price exists
    - `url` (fallback: `https://www.ebay.es/itm/{listing_id}`)
    - `image_url`
  - Limits extraction to **60 items**
  - Builds `compact_text` lines: `ID=... | TITLE=... | PRICE=... | ...`
  - Outputs:
    - `compact_text` (string)
    - `pre_extracted` (array of base items)
    - `stats` (html length, item count, unique ids)
- **Connections:**
  - Output goes to **AI Agent** (main)
  - Output also goes directly to **Merge** as input index 1
- **Failure/edge cases:**
  - If HTML missing/not a string → outputs `compact_text: "NO_HTML_INPUT"` and empty `pre_extracted`
  - eBay markup changes can break regex-based extraction (common long-term risk)
  - Currency detection may be null if symbols/codes are missing
  - URLs may be incomplete; normalization attempts to fix `//`, `/`, relative paths
- **Version notes:** typeVersion 2.

---

### Block D — AI Enrichment (LangChain Agent + OpenAI Chat Model)
**Overview:** Uses an LLM to classify each item into a small taxonomy and assign a 0–10 “deal quality” score, returning strict JSON.

**Nodes involved:**
- OpenAI Chat Model
- AI Agent

#### Node: OpenAI Chat Model
- **Type / role:** LangChain OpenAI chat model node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) used as the language model for the agent.
- **Key configuration:**
  - **Model:** `gpt-4.1-nano`
  - No special options/tools enabled.
- **Credentials:** `openAiApi` (OpenAI API key).
- **Connections:**
  - Provides `ai_languageModel` connection into **AI Agent**
- **Failure/edge cases:**
  - Auth/insufficient quota
  - Model name not available in your OpenAI account/region
  - Transient 5xx/timeouts
- **Version notes:** typeVersion 1.3.

#### Node: AI Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) that sends the compact text to the LLM with a strict system message.
- **Key configuration:**
  - **Prompt type:** “define”
  - **Input text expression:**
    - `HTML:{{ $json.compact_text }}`
    - `NOW_ISO: {{$now}}`
    (Despite the label `HTML:`, it actually sends the compact extracted text.)
  - **System message constraints:**
    - Input format: each line includes `ID, TITLE, PRICE, CURRENCY, ORIGINAL_PRICE, URL, IMG`
    - Output: **array of objects** with only:
      - `listing_id` (must match)
      - `category_ai` in {tech, home, fashion, other}
      - `deal_score` number 0..10 (with scoring heuristics)
      - `score_reason` max 120 chars, no emojis
    - Must output **pure JSON only** (no markdown)
- **Connections:**
  - Main output goes to **Merge** input index 0
- **Failure/edge cases:**
  - The model may still return non-JSON or wrap in code fences; downstream code attempts to recover.
  - If extraction output is empty, the agent may return empty array or invalid JSON.
- **Version notes:** typeVersion 3.

---

### Block E — Merge + Post-processing Join (JavaScript)
**Overview:** Combines base extracted items with LLM enrichment, normalizes values, computes discount metrics and category thresholds, and outputs one item per listing.

**Nodes involved:**
- Merge
- Code in JavaScript

#### Node: Merge
- **Type / role:** Merge node (`n8n-nodes-base.merge`) to combine two streams.
- **Key configuration:**
  - **Mode:** Combine
  - **Combine by:** Position (`combineByPosition`)
  - Practically, it merges the first item from AI Agent stream with the first item from pre-extraction stream into one JSON object containing both sets of fields.
- **Inputs:**
  - Input 0: AI Agent output
  - Input 1: Code in JavaScript1 output (contains `compact_text`, `pre_extracted`, `stats`)
- **Outputs:** To **Code in JavaScript**
- **Failure/edge cases:**
  - If one branch returns no items, combine-by-position can produce empty output or mismatched pairing.
  - If AI Agent returns multiple items while pre-extract returns one item, only position-based pairing may keep unexpected structure (this workflow expects a single “bundle” item with the LLM response + `pre_extracted` array).
- **Version notes:** typeVersion 3.2.

#### Node: Code in JavaScript
- **Type / role:** Code node (`n8n-nodes-base.code`) to join AI results with base items and compute derived fields.
- **Key configuration (interpreted behavior):**
  - Expects after Merge:
    - `$json.pre_extracted` = base array of items
    - `$json.output` = LLM output string (JSON array, ideally)
  - Robust parsing:
    - Strips code fences if present
    - Attempts JSON.parse; if it fails, extracts first `[...]` or `{...}` substring and parses
  - Normalizes AI enrichment:
    - `category_ai` forced to one of: tech/home/fashion/other (fallback other)
    - `deal_score` parsed and clamped 0..10 (fallback 5)
    - `score_reason` trimmed and max 120 chars (fallback message)
  - For each base item:
    - Builds output with price/currency/original price/url/image
    - Computes:
      - `discount_pct` if original_price > price
      - Category thresholds: `{ tech: 30, home: 20, fashion: 15, other: 20 }`
      - `effective_threshold` based on `category_ai`
      - `price_below_threshold` only when `currency === "EUR"` and `price < effective_threshold`
    - Adds `scraped_at` timestamp
  - Deduplicates by `listing_id`
  - **Important:** The node explicitly returns **ALL items, no filtering**
- **Connections:** Output to **If** node.
- **Failure/edge cases:**
  - If `pre_extracted` is missing/empty → outputs 0 items (no Telegram messages)
  - If LLM returns malformed JSON → fallback enrichment is applied per item
  - Currency not EUR → `price_below_threshold` will be false (by design)
- **Version notes:** typeVersion 2.

---

### Block F — Filtering & Notification
**Overview:** Applies simple business rules (price + AI score) and sends a formatted Telegram message for each passing deal.

**Nodes involved:**
- If
- Send a text message

#### Node: If
- **Type / role:** Conditional router (`n8n-nodes-base.if`) for deal filtering.
- **Key configuration:**
  - Condition 1 (number): `{{$json.price}} < 100`
  - Condition 2 (number): `{{$json.deal_score}} > 7`
  - Combinator: **AND**
  - Strict type validation enabled (typeValidation: “strict”)
- **Connections:**
  - True branch → **Send a text message**
  - False branch → unused
- **Failure/edge cases:**
  - If `price` is null/non-numeric, strict numeric comparison can fail or evaluate unexpectedly.
  - If `deal_score` is null, comparison may fail; the prior Code node defaults to 5, which reduces risk.
- **Version notes:** typeVersion 2.2.

#### Node: Send a text message
- **Type / role:** Telegram node (`n8n-nodes-base.telegram`) to send alerts.
- **Key configuration:**
  - **chatId:** `123456789` (must be replaced with your target chat/user/channel ID)
  - **Message template:** Includes title, price, AI category, threshold, discount line (conditional), score, reason, URL, and scrape timestamp.
  - Uses expressions like:
    - `{{ $json.title || 'Product' }}`
    - Conditional discount block: `{{ $json.discount_pct ? (...) : '' }}`
- **Credentials:** `telegramApi` (bot token).
- **Inputs:** Items that pass the If node.
- **Failure/edge cases:**
  - Wrong bot token / revoked token
  - chat_id not reachable (bot not started, channel permissions missing)
  - Message formatting issues if some fields are null (mostly handled)
- **Version notes:** typeVersion 1.2.

---

### Block G — Documentation/Annotations (Sticky Notes)
**Overview:** Sticky notes explain intent, configuration, and show example output. They don’t execute.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Decodo | ## What this workflow does<br>This workflow automatically monitors **eBay Deals** and sends **Telegram alerts** when relevant, high-quality deals are detected.<br><br>It combines:<br>- **Web scraping with Decodo**<br>- **JavaScript pre-processing (no raw HTML sent to the LLM)**<br>- **AI-based product classification and deal scoring**<br>- **Rule-based filtering using price and score**<br><br>Only valuable deals reach the final notification.<br>## How to configure it<br>### 1. Decodo<br>- Add your **Decodo API credentials** to the Decodo node.<br>- Optionally change the target eBay URL.<br>### 2. AI Agent<br>- Add your LLM credentials (e.g. Google Gemini).<br>- No HTML is sent to the model — only compact, structured data.<br>### 3. Telegram<br>- Add your **Telegram Bot Token**.<br>- Set your **chat_id** in the Telegram node.<br>- Customize the alert message if needed.<br>### 4. Filtering rules<br>- Adjust price limits and minimum deal score in the **IF node**. |
| Schedule Trigger | Schedule Trigger | Scheduled entry point (every 3h) | — | Decodo | The workflow starts manually or on a schedule.<br>Decodo is used to scrape the eBay Deals page reliably, handling proxies and anti-bot protections. |
| Decodo | Decodo | Scrape eBay deals HTML | When clicking ‘Execute workflow’; Schedule Trigger | Code in JavaScript1 | The workflow starts manually or on a schedule.<br>Decodo is used to scrape the eBay Deals page reliably, handling proxies and anti-bot protections. |
| Code in JavaScript1 | Code | Extract deals + build compact LLM input | Decodo | AI Agent; Merge | Raw HTML is reduced using JavaScript to extract only key product data.<br>An AI Agent enriches each item with category classification and deal quality scoring. |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM backend for agent | — | AI Agent (ai_languageModel) | Raw HTML is reduced using JavaScript to extract only key product data.<br>An AI Agent enriches each item with category classification and deal quality scoring. |
| AI Agent | LangChain Agent | Classify + score items, output JSON | Code in JavaScript1 | Merge | Raw HTML is reduced using JavaScript to extract only key product data.<br>An AI Agent enriches each item with category classification and deal quality scoring. |
| Merge | Merge | Combine base extraction + LLM output | AI Agent; Code in JavaScript1 | Code in JavaScript | Business rules are applied using price and score conditions.<br>Only high-quality deals pass the filter and trigger a Telegram alert. |
| Code in JavaScript | Code | Join by listing_id, normalize, compute thresholds/discount | Merge | If | Business rules are applied using price and score conditions.<br>Only high-quality deals pass the filter and trigger a Telegram alert. |
| If | If | Filter: price < 100 AND score > 7 | Code in JavaScript | Send a text message (true) | Business rules are applied using price and score conditions.<br>Only high-quality deals pass the filter and trigger a Telegram alert. |
| Send a text message | Telegram | Send alert message | If (true) | — | Business rules are applied using price and score conditions.<br>Only high-quality deals pass the filter and trigger a Telegram alert. |
| Sticky Note | Sticky Note | Documentation | — | — |  |
| Sticky Note1 | Sticky Note | Documentation | — | — |  |
| Sticky Note2 | Sticky Note | Documentation | — | — |  |
| Sticky Note3 | Sticky Note | Documentation | — | — |  |
| Sticky Note4 | Sticky Note | Example output image | — | — | ##  Output<br>![txt](https://ik.imagekit.io/agbb7sr41/telegram_output.png) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it (e.g., *Automated eBay Deals Monitoring with AI Scoring*). Keep workflow **inactive** until credentials are set.

2. **Add Trigger (Manual):**
   - Add node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add Trigger (Scheduled):**
   - Add node: **Schedule Trigger**
   - Configure interval: **Every 3 hours**

4. **Add Decodo scraper node:**
   - Add node: **Decodo** (`@decodo/n8n-nodes-decodo.decodo`)
   - Configure:
     - URL: `https://www.ebay.es/deals` (change if you want another eBay domain)
     - Geo: `en`
     - Headless: `false`
   - **Credentials:** create/select **Decodo API** credential (your Decodo key)
   - Connect:
     - Manual Trigger → Decodo
     - Schedule Trigger → Decodo

5. **Add Pre-extraction Code node:**
   - Add node: **Code** (JavaScript)
   - Name: `Code in JavaScript1`
   - Paste logic that:
     - Reads `$json.results[0].content`
     - Extracts items (listing_id, title, price, currency, original_price, url, image_url)
     - Outputs `compact_text`, `pre_extracted`, and `stats`
   - Connect: Decodo → Code in JavaScript1

6. **Add OpenAI Chat Model node (LangChain):**
   - Add node: **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
   - Model: `gpt-4.1-nano` (or another supported OpenAI chat model)
   - **Credentials:** create/select **OpenAI API** credential

7. **Add AI Agent node:**
   - Add node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Set Prompt mode to **Define**
   - In “Text” input, use an expression combining:
     - `HTML:{{ $json.compact_text }}`
     - `NOW_ISO: {{$now}}`
   - Set the **System Message** enforcing:
     - Output must be **pure JSON array**
     - Fields: listing_id, category_ai (enum), deal_score 0..10, score_reason <=120 chars
   - Connect:
     - Code in JavaScript1 → AI Agent (main)
     - OpenAI Chat Model → AI Agent (ai_languageModel)

8. **Add Merge node:**
   - Add node: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - AI Agent → Merge (Input 0)
     - Code in JavaScript1 → Merge (Input 1)
   - Expected result: a single merged item containing `pre_extracted` and the agent’s `output` field.

9. **Add Post-processing Join Code node:**
   - Add node: **Code** (JavaScript)
   - Name: `Code in JavaScript`
   - Implement logic that:
     - Parses `$json.output` robustly (strip code fences, parse JSON, fallback regex)
     - Indexes AI enrichment by `listing_id`
     - Emits one n8n item per base listing with normalized fields
     - Computes `discount_pct`, `effective_threshold`, `price_below_threshold`, `scraped_at`
   - Connect: Merge → Code in JavaScript

10. **Add If node (business filter):**
    - Add node: **If**
    - Conditions (AND):
      - Number: `{{$json.price}}` **lt** `100`
      - Number: `{{$json.deal_score}}` **gt** `7`
    - Connect: Code in JavaScript → If

11. **Add Telegram node (send message):**
    - Add node: **Telegram** → “Send a text message”
    - **Credentials:** create/select **Telegram API** credential (bot token)
    - Set **chatId** to your destination chat/user/channel ID
    - Message text: include expressions for title/price/category/discount/score/url/timestamp as needed
    - Connect: If (true output) → Telegram

12. **Test manually:**
    - Click **Execute workflow**
    - Verify:
      - Decodo returns HTML in `results[0].content`
      - Extraction produces non-empty `pre_extracted`
      - AI Agent returns JSON
      - Telegram sends messages only when conditions match

13. **Activate workflow** once stable, so the schedule trigger runs.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Example Telegram output image | https://ik.imagekit.io/agbb7sr41/telegram_output.png |
| Configuration guidance embedded in workflow notes: Decodo creds, LLM creds, Telegram bot/chat_id, filtering rules | Present in the main sticky note (“What this workflow does / How to configure it”) |