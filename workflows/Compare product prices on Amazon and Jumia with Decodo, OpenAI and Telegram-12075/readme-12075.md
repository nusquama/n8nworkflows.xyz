Compare product prices on Amazon and Jumia with Decodo, OpenAI and Telegram

https://n8nworkflows.xyz/workflows/compare-product-prices-on-amazon-and-jumia-with-decodo--openai-and-telegram-12075


# Compare product prices on Amazon and Jumia with Decodo, OpenAI and Telegram

## 1. Workflow Overview

**Purpose:** This workflow receives a product name from Telegram, finds likely product pages on **Amazon Egypt** and **Jumia Egypt** via **Decodo Google Search**, scrapes page HTML via **Decodo**, filters the best matching page per store using a heuristic scoring algorithm, then asks an **OpenAI (LangChain) agent** to extract product + price and returns a concise comparison message back to Telegram.

**Target use cases:**
- Quick price comparison for a user-provided product query (English/Arabic supported in tokenization).
- Monitoring or on-demand aggregation of prices across a small set of e-commerce sites.

### 1.1 Input Reception (Telegram)
Receives a Telegram message containing the product name to search.

### 1.2 Search & URL Preparation (Decodo Google Search → URL extraction)
Builds store-specific Google queries and extracts candidate URLs from Decodo results.

### 1.3 Routing & Parallel Scraping (Amazon vs Other)
Routes URLs into an Amazon-specific branch and a generic “Other” branch (Jumia/others), then scrapes and cleans HTML.

### 1.4 Relevance Scoring & Best-URL Selection (per branch)
Scores each scraped page against the user query and selects the single “best match” URL per branch.

### 1.5 AI Extraction & Reply (OpenAI Agent → Telegram)
Merges best results, formats context, calls the AI agent to extract product/prices, and replies in Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Query Building
**Overview:** Captures the incoming Telegram message and turns it into two Google search queries (Amazon EG and Jumia EG).  
**Nodes involved:** `Telegram Trigger`, `Build Queries`

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — entry point; receives Telegram updates.
- **Config (interpreted):**
  - Update type: `message`
  - Webhook-based trigger (webhookId placeholder in template).
- **Key data used downstream:**
  - `message.text` (product query)
  - `message.chat.id` (reply destination)
- **Outputs:** To `Build Queries`.
- **Failure/edge cases:**
  - Webhook not set/disabled → workflow never triggers.
  - Non-text messages (stickers, photos) → `message.text` missing; next node returns no items.
  - Telegram credential/auth issues.

#### Node: Build Queries
- **Type / role:** `code` — transforms the Telegram message into per-store search query items.
- **Config choices:**
  - Reads `const product = $json.message.text?.trim();`
  - Defines stores array:
    - `{ store: "amazon", site: "amazon.eg" }`
    - `{ store: "jumia", site: "jumia.com.eg" }`
  - Emits one item per store with:
    - `store`, `product`, `query: "${product} site:${s.site}"`
- **Outputs:** To `Google Search` (one item per store).
- **Failure/edge cases:**
  - Empty/undefined product → returns `[]` (workflow stops with no reply unless you add handling).
  - Users entering very short terms may yield poor search relevance.

---

### Block 2 — Google Search & URL Extraction
**Overview:** Uses Decodo to run Google searches and extracts up to 20 non-Google URLs from the returned structure.  
**Nodes involved:** `Google Search`, `Extract & Filter URLs`

#### Node: Google Search
- **Type / role:** `@decodo/n8n-nodes-decodo.decodo` — performs Google search via Decodo.
- **Config choices:**
  - Operation: `google_search`
  - Query: `={{ $json.query }}`
  - `results_limit: 1`
  - `headless: false`
- **Outputs:** To `Extract & Filter URLs`.
- **Failure/edge cases:**
  - Decodo credential/account limitations (Google Search not enabled).
  - Rate limits / captchas / upstream Google restrictions surfaced by Decodo.
  - `results_limit: 1` may produce too few URLs depending on Decodo’s response format.

#### Node: Extract & Filter URLs
- **Type / role:** `code` — recursively crawls the Decodo output to find any `url` fields.
- **Config choices:**
  - Collects all `url` strings starting with `http`.
  - Deduplicates, filters out `google.com/search`, and limits to **20** URLs.
  - Returns one item per `{ url }`.
- **Outputs:** To `Router (Amazon vs. Other)`.
- **Failure/edge cases:**
  - If Decodo output schema changes and no longer uses `url` fields, extraction may return 0 URLs.
  - Might capture irrelevant URLs if Decodo includes tracking/redirect URLs.
  - Hard cap 20 can exclude relevant results if noise is high.

---

### Block 3 — Routing & Scraping (Amazon-specific vs Generic)
**Overview:** Splits candidate URLs into Amazon vs non-Amazon, scrapes each, and cleans HTML into text.  
**Nodes involved:** `Router (Amazon vs. Other)`, `Loop (Amazon)`, `Scrape Amazon`, `Clean HTML (Amazon)`, `Loop (Other)`, `Scrape Other`, `Clean HTML (Other)`

#### Node: Router (Amazon vs. Other)
- **Type / role:** `if` — routes items based on whether the URL contains `"amazon"`.
- **Config choices:**
  - Condition: `{{ $json.url }}` **contains** `"amazon"`
  - True branch = Amazon; False branch = Other.
- **Outputs:**
  - True → `Loop (Amazon)`
  - False → `Loop (Other)`
- **Failure/edge cases:**
  - Non-Amazon URLs containing “amazon” (edge rare but possible) misrouted.
  - Amazon short links or regional domains not matching the substring logic if they don’t include “amazon” (unlikely).

#### Node: Loop (Amazon)
- **Type / role:** `splitInBatches` — iterates through Amazon URLs and also provides an end-of-loop output.
- **Config choices:** Default batch options (not explicitly set).
- **Outputs:**
  - Main output (per batch item) → `Scrape Amazon`
  - Second output (when done) → `Pick Best (Amazon)`
- **Failure/edge cases:**
  - If batch size defaults to 1, it will loop one-by-one (slower but safe).
  - Incorrect looping can occur if downstream doesn’t connect back; here, the loop-back is done via `Score Match (Amazon)` → `Loop (Amazon)`.

#### Node: Scrape Amazon
- **Type / role:** Decodo node — scrapes Amazon page.
- **Config choices:**
  - Operation: `amazon` (Decodo Amazon scraping mode)
  - `parse: false` (keeps raw HTML-like content in results)
  - URL: `={{ $('Extract & Filter URLs').item.json.url }}`
    - Note: This expression references `Extract & Filter URLs` rather than the current loop item directly.
- **Outputs:** To `Clean HTML (Amazon)`.
- **Failure/edge cases:**
  - **Potential item mismatch risk:** using `$('Extract & Filter URLs').item.json.url` can desync inside loops if execution context differs. Safer is typically `={{ $json.url }}` from `Loop (Amazon)` items.
  - Amazon bot protections: Decodo may return incomplete HTML or errors.
  - Missing `results[0].content` leads to empty cleaned text downstream.

#### Node: Clean HTML (Amazon)
- **Type / role:** `code` — strips scripts/styles/tags and normalizes whitespace.
- **Config choices:**
  - Reads HTML from: `item.results[0].content || ""` guarded by try/catch.
  - Produces:
    - `text_clean`: cleaned visible text (not lowercased here; later scoring lowercases)
    - `url: $('Loop (Amazon)').first().json.url`
- **Outputs:** To `Score Match (Amazon)`.
- **Failure/edge cases:**
  - If Decodo response doesn’t include `results[0].content`, cleaned text becomes empty → scoring marks as non-match.
  - URL assignment uses `$('Loop (Amazon)').first()`; if multiple URLs, “first” may not be the one being processed (context risk). Prefer `$json.url` passed through the loop item.

#### Node: Loop (Other)
- **Type / role:** `splitInBatches` — iterates through non-Amazon URLs.
- **Outputs:**
  - Main output → `Scrape Other`
  - Done output → `Pick Best (Other)`
- **Failure/edge cases:** Similar to Amazon loop.

#### Node: Scrape Other
- **Type / role:** Decodo node — generic scraping for arbitrary pages (Jumia, etc.).
- **Config choices:**
  - URL: `={{ $json.url }}`
  - Operation not explicitly set in JSON (Decodo default for the node will apply).
- **Outputs:** To `Clean HTML (Other)`.
- **Failure/edge cases:**
  - Some sites require JS rendering; if Decodo default doesn’t render dynamic content, price may be missing.
  - Possible 403/anti-bot pages; returns irrelevant text.

#### Node: Clean HTML (Other)
- **Type / role:** `code` — same cleaning approach as Amazon branch.
- **Config choices:**
  - Produces `text_clean` and `url: $('Loop (Other)').first().json.url` (same “first-item” caveat).
- **Outputs:** To `Score Match (Other)`.
- **Failure/edge cases:** Same as Amazon cleaning.

---

### Block 4 — Relevance Scoring & Best Match Selection
**Overview:** Computes a heuristic relevance score for each cleaned page vs the user query, then selects the top-scoring matched page per branch.  
**Nodes involved:** `Score Match (Amazon)`, `Pick Best (Amazon)`, `Score Match (Other)`, `Pick Best (Other)`

#### Node: Score Match (Amazon)
- **Type / role:** `code` — determines whether a page likely matches the requested product.
- **Config choices / algorithm:**
  - User query source: `$('Telegram Trigger').first().json.message.text`
  - Normalization: lowercase, remove punctuation (keeps Arabic range `\u0600-\u06FF`).
  - Tokenization: split on whitespace; keep tokens length ≥ 3.
  - Page text normalized similarly.
  - Scoring:
    - For each token: adds `min(occurrences*3, 15)`
    - Bonuses:
      - `+5` if pageText length > 1500
      - `-10` if length < 400
      - `+5` if includes “add to cart” or Arabic variants
    - Threshold: `max(12, userTokens.length * 4)`
  - Outputs: `{ match, score, threshold, url, matched_tokens, text_clean }`
- **Connections:**
  - Output goes back into `Loop (Amazon)` to continue iteration.
- **Failure/edge cases:**
  - If pageText is huge, token occurrence counting via `split(token)` can be expensive.
  - Generic product names (“iPhone”, “Samsung”) may match many pages; threshold may still allow false positives.
  - If page contains mostly boilerplate and little product text (dynamic rendering), may fail match.

#### Node: Pick Best (Amazon)
- **Type / role:** `code` — chooses the highest-scoring matched Amazon page.
- **Config choices:**
  - Consumes all items produced during the loop.
  - Filters `match === true`, sorts by `score desc`, returns top 1.
  - If none matched: returns `[]`.
- **Outputs:** To `Merge Results` (input 0).
- **Failure/edge cases:**
  - If no match, Amazon branch contributes nothing; downstream merge may wait depending on merge behavior (see Merge node notes).

#### Node: Score Match (Other)
- **Type / role:** `code` — identical to Amazon scoring.
- **Connections:** Output loops back to `Loop (Other)`.
- **Failure/edge cases:** Same as Amazon scoring.

#### Node: Pick Best (Other)
- **Type / role:** `code` — chooses top matched non-Amazon page.
- **Outputs:** To `Merge Results` (input 1).
- **Failure/edge cases:** Same as Amazon pick best.

---

### Block 5 — Merge, Context Formatting, AI Extraction, Telegram Reply
**Overview:** Merges the selected pages, builds a combined context string, uses an AI agent to extract price details and creates a Telegram-friendly message, then sends it.  
**Nodes involved:** `Merge Results`, `Format Context`, `GPT Model`, `AI Extraction Agent`, `Send Reply`

#### Node: Merge Results
- **Type / role:** `merge` — combines Amazon best item and Other best item.
- **Config choices:** No explicit mode shown; n8n Merge defaults are important here.
- **Inputs:**
  - Input 0: `Pick Best (Amazon)`
  - Input 1: `Pick Best (Other)`
- **Outputs:** To `Format Context`.
- **Failure/edge cases:**
  - If either branch returns **no items**, some merge modes will output nothing or wait. To make the workflow robust, configure Merge mode to behave well with missing inputs (e.g., “Append” where available).
  - If both return items, output will be a combined item set (consumed by `Format Context` via `$input.all()`).

#### Node: Format Context
- **Type / role:** `code` — creates a single text payload combining sources for the AI agent.
- **Config choices:**
  - Reads all merged items: `const items = $input.all().map(i => i.json);`
  - Builds `combinedText` sections containing:
    - `SOURCE URL`, `RELEVANCE SCORE`, and first 5000 chars of `text_clean`
  - Outputs `{ text: combinedText }` with explicit comment: “THIS NAME IS CRITICAL”
- **Outputs:** To `AI Extraction Agent`.
- **Failure/edge cases:**
  - If `text_clean` is missing, `.slice()` may throw unless it’s always a string (here it assumes existence).
  - AI context can still be very large; truncation to 5000 chars per source helps.

#### Node: GPT Model
- **Type / role:** `lmChatOpenAi` — provides the OpenAI chat model used by the agent.
- **Config choices:**
  - Model: `gpt-4o-mini`
- **Connections:**
  - Connected to `AI Extraction Agent` via `ai_languageModel`.
- **Failure/edge cases:**
  - Missing/invalid OpenAI credentials.
  - Model availability changes; choose another model if unavailable.
  - Rate limits/timeouts.

#### Node: AI Extraction Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompt-driven extraction and final message composition.
- **Config choices:**
  - Prompt includes:
    - User intent: `{{ $('Telegram Trigger').first().json.message.text }}`
    - Scraped data: `{{ $json.text }}`
  - System message enforces:
    - No invention; “Price not found” if missing
    - Extract per platform: name, price, platform, URL
    - Telegram-friendly concise output; moderate emojis
    - Output must be a **single formatted message string**, no JSON, no markdown blocks
- **Outputs:** To `Send Reply`
- **Failure/edge cases:**
  - If `Format Context` outputs empty/poor text, agent may output “Price not found” for all.
  - If the agent returns in an unexpected field name, downstream mapping may fail (see next node).

#### Node: Send Reply
- **Type / role:** `telegram` — sends the AI-produced message back to the originating chat.
- **Config choices:**
  - `chatId`: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
  - `text`: `={{ $json.output }}`
  - `appendAttribution: false`
- **Inputs:** From `AI Extraction Agent`.
- **Failure/edge cases:**
  - **Field mismatch risk:** Many LangChain agent nodes output content under fields like `output`, `text`, or `result` depending on node/version. This workflow expects `output`.
  - Telegram message length limits; if AI response is too long, sending fails—consider truncation.
  - Chat ID extraction assumes a normal message update.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Sticky | stickyNote | Workflow description & setup notes | — | — | ### 🛍️ AI Price Monitor & Aggregator / How it works / Setup steps / Customization tips |
| Section 1 | stickyNote | Section header: links & routing | — | — | ## 1. Get Links & Route Receives the user request, searches Google, and separates Amazon links from others for specialized handling. |
| Section 2 | stickyNote | Section header: Amazon scraping | — | — | ## 2. Amazon Scraping Specialized HTML cleaning and scoring logic optimized for Amazon product pages. |
| Section 3 | stickyNote | Section header: General scraping | — | — | ## 3. General Scraping Generic scraping logic for Jumia and other supported e-commerce sites. |
| Section 4 | stickyNote | Section header: AI extraction & reply | — | — | ## 4. AI Extraction & Reply Merges results and uses an AI Agent to extract structured price data and reply to the user. |
| Telegram Trigger | telegramTrigger | Receive product name from Telegram | — | Build Queries | ## 1. Get Links & Route Receives the user request, searches Google, and separates Amazon links from others for specialized handling. |
| Build Queries | code | Build Google queries per store | Telegram Trigger | Google Search | ## 1. Get Links & Route Receives the user request, searches Google, and separates Amazon links from others for specialized handling. |
| Google Search | decodo | Google search via Decodo | Build Queries | Extract & Filter URLs | ## 1. Get Links & Route Receives the user request, searches Google, and separates Amazon links from others for specialized handling. |
| Extract & Filter URLs | code | Extract and dedupe candidate URLs | Google Search | Router (Amazon vs. Other) | ## 1. Get Links & Route Receives the user request, searches Google, and separates Amazon links from others for specialized handling. |
| Router (Amazon vs. Other) | if | Route Amazon URLs vs others | Extract & Filter URLs | Loop (Amazon); Loop (Other) | ## 1. Get Links & Route Receives the user request, searches Google, and separates Amazon links from others for specialized handling. |
| Loop (Amazon) | splitInBatches | Iterate Amazon URLs / end-of-loop trigger | Router (Amazon vs. Other); Score Match (Amazon) | Scrape Amazon; Pick Best (Amazon) | ## 2. Amazon Scraping Specialized HTML cleaning and scoring logic optimized for Amazon product pages. |
| Scrape Amazon | decodo | Fetch Amazon page HTML via Decodo | Loop (Amazon) | Clean HTML (Amazon) | ## 2. Amazon Scraping Specialized HTML cleaning and scoring logic optimized for Amazon product pages. |
| Clean HTML (Amazon) | code | Strip tags/scripts and normalize text | Scrape Amazon | Score Match (Amazon) | ## 2. Amazon Scraping Specialized HTML cleaning and scoring logic optimized for Amazon product pages. |
| Score Match (Amazon) | code | Score relevance vs user query | Clean HTML (Amazon) | Loop (Amazon) | ## 2. Amazon Scraping Specialized HTML cleaning and scoring logic optimized for Amazon product pages. |
| Pick Best (Amazon) | code | Select best matching Amazon page | Loop (Amazon) | Merge Results | ## 2. Amazon Scraping Specialized HTML cleaning and scoring logic optimized for Amazon product pages. |
| Loop (Other) | splitInBatches | Iterate non-Amazon URLs / end-of-loop trigger | Router (Amazon vs. Other); Score Match (Other) | Scrape Other; Pick Best (Other) | ## 3. General Scraping Generic scraping logic for Jumia and other supported e-commerce sites. |
| Scrape Other | decodo | Fetch page HTML via Decodo | Loop (Other) | Clean HTML (Other) | ## 3. General Scraping Generic scraping logic for Jumia and other supported e-commerce sites. |
| Clean HTML (Other) | code | Strip tags/scripts and normalize text | Scrape Other | Score Match (Other) | ## 3. General Scraping Generic scraping logic for Jumia and other supported e-commerce sites. |
| Score Match (Other) | code | Score relevance vs user query | Clean HTML (Other) | Loop (Other) | ## 3. General Scraping Generic scraping logic for Jumia and other supported e-commerce sites. |
| Pick Best (Other) | code | Select best matching non-Amazon page | Loop (Other) | Merge Results | ## 3. General Scraping Generic scraping logic for Jumia and other supported e-commerce sites. |
| Merge Results | merge | Combine best Amazon + best Other | Pick Best (Amazon); Pick Best (Other) | Format Context | ## 4. AI Extraction & Reply Merges results and uses an AI Agent to extract structured price data and reply to the user. |
| Format Context | code | Create combined text context for AI | Merge Results | AI Extraction Agent | ## 4. AI Extraction & Reply Merges results and uses an AI Agent to extract structured price data and reply to the user. |
| GPT Model | lmChatOpenAi | Provide OpenAI model to agent | — | AI Extraction Agent (AI languageModel connection) | ## 4. AI Extraction & Reply Merges results and uses an AI Agent to extract structured price data and reply to the user. |
| AI Extraction Agent | langchain agent | Extract product/price and craft message | Format Context; GPT Model | Send Reply | ## 4. AI Extraction & Reply Merges results and uses an AI Agent to extract structured price data and reply to the user. |
| Send Reply | telegram | Send final message to Telegram chat | AI Extraction Agent | — | ## 4. AI Extraction & Reply Merges results and uses an AI Agent to extract structured price data and reply to the user. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named *Price Monitor (Template)* (or your preferred name). Ensure workflow settings use a standard execution order (default is fine).

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Updates: **message**
   - Credentials: configure **Telegram API** (bot token).
   - Enable/verify webhook in node (n8n typically manages this when the workflow is active).

3. **Add Code node: “Build Queries”**
   - Connect: `Telegram Trigger → Build Queries`
   - Paste logic to:
     - Read `message.text`, trim it
     - Build two items for:
       - Amazon EG: `site:amazon.eg`
       - Jumia EG: `site:jumia.com.eg`
     - Output items `{ store, product, query }`

4. **Add Decodo node: “Google Search”**
   - Connect: `Build Queries → Google Search`
   - Operation: **google_search**
   - Query: `{{ $json.query }}`
   - Results limit: **1**
   - Credentials: configure **Decodo** credentials/account.

5. **Add Code node: “Extract & Filter URLs”**
   - Connect: `Google Search → Extract & Filter URLs`
   - Implement recursive extraction of any `url` fields, then:
     - Deduplicate
     - Remove `google.com/search`
     - Limit to 20
     - Output one item per `{ url }`

6. **Add IF node: “Router (Amazon vs. Other)”**
   - Connect: `Extract & Filter URLs → Router (Amazon vs. Other)`
   - Condition: `{{ $json.url }}` **contains** `amazon`
   - True output = Amazon, False output = Other

7. **Amazon branch loop**
   1) Add **Split In Batches**: “Loop (Amazon)”
      - Connect: `Router true → Loop (Amazon)`
   2) Add **Decodo**: “Scrape Amazon”
      - Connect: `Loop (Amazon) main → Scrape Amazon`
      - Operation: **amazon**
      - Parse: **false**
      - URL expression: ideally `{{ $json.url }}` (recommended to avoid context mismatch).
   3) Add **Code**: “Clean HTML (Amazon)”
      - Connect: `Scrape Amazon → Clean HTML (Amazon)`
      - Strip scripts/styles/tags, normalize whitespace
      - Output fields: `text_clean`, `url` (pass-through current `$json.url` from the loop item is recommended)
   4) Add **Code**: “Score Match (Amazon)”
      - Connect: `Clean HTML (Amazon) → Score Match (Amazon)`
      - Implement scoring vs `Telegram Trigger` text
   5) Loop back:
      - Connect: `Score Match (Amazon) → Loop (Amazon)` (to continue batches)
   6) Add **Code**: “Pick Best (Amazon)”
      - Connect: `Loop (Amazon) done output → Pick Best (Amazon)`
      - Filter `match===true`, sort by score desc, return top item

8. **Other branch loop**
   1) Add **Split In Batches**: “Loop (Other)”
      - Connect: `Router false → Loop (Other)`
   2) Add **Decodo**: “Scrape Other”
      - Connect: `Loop (Other) main → Scrape Other`
      - URL: `{{ $json.url }}`
   3) Add **Code**: “Clean HTML (Other)”
      - Connect: `Scrape Other → Clean HTML (Other)`
   4) Add **Code**: “Score Match (Other)”
      - Connect: `Clean HTML (Other) → Score Match (Other)`
   5) Loop back:
      - Connect: `Score Match (Other) → Loop (Other)`
   6) Add **Code**: “Pick Best (Other)”
      - Connect: `Loop (Other) done output → Pick Best (Other)`

9. **Merge the two best results**
   - Add **Merge** node: “Merge Results”
   - Connect:
     - `Pick Best (Amazon) → Merge Results (Input 0)`
     - `Pick Best (Other) → Merge Results (Input 1)`
   - Configure merge mode to handle missing inputs robustly (commonly “Append” is safest for “one-or-none from each branch”).

10. **Add Code node: “Format Context”**
    - Connect: `Merge Results → Format Context`
    - Build a single string combining per-source:
      - URL
      - score
      - first ~5000 chars of `text_clean`
    - Output as `{ text: combinedText }` (the field name `text` must match the agent prompt mapping).

11. **Add OpenAI Chat Model node: “GPT Model”**
    - Node: **OpenAI Chat Model (LangChain)**
    - Model: `gpt-4o-mini` (or substitute)
    - Credentials: configure **OpenAI** credentials.

12. **Add LangChain Agent node: “AI Extraction Agent”**
    - Connect: `Format Context → AI Extraction Agent`
    - In the agent, set:
      - Prompt text referencing:
        - Telegram message text
        - `{{ $json.text }}`
      - System message with strict extraction rules and “return only a single message string”.
    - Connect the **GPT Model** node to the agent via the **AI Language Model** connection.

13. **Add Telegram node: “Send Reply”**
    - Connect: `AI Extraction Agent → Send Reply`
    - Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
    - Text: map to the agent output field (this template uses `{{ $json.output }}`; adjust if your agent outputs a different property).
    - Disable attribution if desired.

14. **Activate the workflow**
    - Ensure Telegram webhook becomes active and Decodo/OpenAI credentials are valid.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Configure `Telegram API`, `OpenAI`, and `Decodo` credentials. | Mentioned in workflow sticky note (“Setup steps”). |
| Ensure your Decodo account allows Google Search operations. | Mentioned in workflow sticky note (“Setup steps”). |
| Verify the Telegram webhook is active. | Mentioned in workflow sticky note (“Setup steps”). |
| To add more stores (e.g., noon.com), edit the `Build Queries` node and extend routing/scraping logic as needed. | Mentioned in workflow sticky note (“Customization tips”). |
| Adjust the router if you want domain-specific scraping logic. | Mentioned in workflow sticky note (“Customization tips”). |

Disclaimer (provided by user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.