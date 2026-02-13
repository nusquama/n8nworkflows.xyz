Estimate construction costs from text with Telegram, OpenAI and DDC CWICR

https://n8nworkflows.xyz/workflows/estimate-construction-costs-from-text-with-telegram--openai-and-ddc-cwicr-12174


# Estimate construction costs from text with Telegram, OpenAI and DDC CWICR

## 1. Workflow Overview

**Workflow name (in JSON):** `DDC CWICR - Text Estimator v11 (AI Nodes)`  
**User-facing title:** *Estimate construction costs from text with Telegram, OpenAI and DDC CWICR*

### Purpose
This workflow implements a **Telegram bot** that accepts **free-form construction descriptions**, extracts a structured list of work items using an LLM, then estimates costs by performing **vector search in a Qdrant database (DDC CWICR collections)**, **LLM reranking**, and a **pricing/resources calculation**. It returns results in Telegram and supports **Exports (CSV/“Excel”, HTML, and an HTML-as-PDF placeholder)** plus a **details view** with resources and scope-of-work.

### Core integrations
- **Telegram Bot API**: trigger + message/callback handling + sending/editing/deleting bot messages.
- **OpenAI via n8n Credentials**:
  - Chat LLM for: parse user text, transform query, rerank.
  - Embeddings endpoint for vector search (HTTP request authenticated with `openAiApi` credential).
- **Qdrant**: similarity search against language-specific DDC CWICR collections.

### Logical blocks (high-level)
1. **1.1 Entry & Credentials Injection**  
   Telegram trigger → inject token/Qdrant URL → main router.
2. **1.2 Session, Routing & Localization**  
   Maintain per-chat session state, select language, route actions via Switch.
3. **1.3 Text → Work Items (LLM parsing)**  
   Build parsing prompt → AI parse → parse JSON → store works → show editable list.
4. **1.4 Editing & Session Operations**  
   Edit menu, quantity changes, delete/add work items, restart/help.
5. **1.5 Calculation Loop (per work item)**  
   Progress message → split in batches → per-item: “Searching…” message → query transform → embeddings → Qdrant search → rerank → calculate → update Telegram status → accumulate.
6. **1.6 Aggregation & Reporting**  
   Cleanup progress messages → aggregate totals → generate HTML report → generate Telegram final summary + send HTML file.
7. **1.7 Export & Details Views**  
   View detailed resources/scope; export CSV; export “PDF” (currently HTML file sent as document).

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & token/Qdrant configuration
**Overview:** Receives Telegram updates and injects basic configuration (bot token, Qdrant connection) into the workflow execution path.  
**Nodes involved:** `Telegram Trigger1`, `🔑 TOKEN`

#### Node: Telegram Trigger1
- **Type / role:** `telegramTrigger` — workflow entry point for Telegram updates.
- **Configuration:** Listens to `message` and `callback_query`.
- **Outputs:** Passes full Telegram update payload into next node.
- **Edge cases:**
  - Telegram webhook misconfigured (no updates).
  - Credential missing/invalid in n8n Telegram Trigger credential.

#### Node: 🔑 TOKEN
- **Type / role:** `Set` — stores runtime constants (token and URLs).
- **Configuration choices:**
  - `bot_token` (Telegram Bot API token) must be set.
  - `QDRANT_URL` and `QDRANT_API_KEY` for Qdrant search.
  - `OPENAI_API_KEY` is present but **not used** by OpenAI nodes (they use n8n credential), and embeddings request uses `openAiApi` credential.
- **Output:** Forwards these values to `Main`.
- **Edge cases:**
  - Invalid `bot_token` produces Telegram API “resource not found” / 401 errors.
  - Wrong `QDRANT_URL` or unreachable host (timeouts in Qdrant search).

---

### Block 2 — Session router + localization + action routing
**Overview:** Converts Telegram updates (messages and callbacks) into a single `action` command, maintains per-chat session state in workflow static data, and injects localization strings and DB collection mapping.  
**Nodes involved:** `Main`, `Config`, `Route`

#### Node: Main
- **Type / role:** `Code` — central state machine/router.
- **Key behavior:**
  - Reads Telegram update from `Telegram Trigger1`.
  - Uses `$getWorkflowStaticData('global')` to store per-chat session: `sd.sess[cid]`.
  - Detects `/start`, `/help`, language selection callbacks (`lang_XX`), editing callbacks (`edit_work_n`, `qty_work_n_change`, `delete_work_n`), exports, calculate, restart.
  - When user sends text in `wait_text` state, sets action `analyze_text`.
- **Key output fields:** `chatId`, `action`, `lang`, `works`, `callbackQueryId`, `text`, `description`.
- **Connections:** Output → `Config`.
- **Edge cases:**
  - Telegram update formats may vary; missing `message.chat.id` could break `chatId`.
  - Session data persists across runs; staticData can grow if many users never finish.

#### Node: Config
- **Type / role:** `Code` — localization + language-to-collection mapping.
- **Configuration choices:**
  - Defines `LANGS` object for 9 languages (DE, EN, RU, ES, FR, PT, ZH, AR, HI).
  - Each language config provides:
    - Telegram UI texts/buttons
    - currency (`cur`, `sym`), locale (`loc`), region
    - Qdrant collection name in `db`
    - `search_lang` used in prompts
  - Updates `sd.sess[chatId]` with chosen `db` and `L`.
- **Connections:** Output → `Route` switch.
- **Edge cases:**
  - Collection names must match Qdrant; otherwise all searches return empty.
  - Some English translations still contain Russian words (e.g., `rooms`/`general`)—cosmetic but confusing.

#### Node: Route
- **Type / role:** `Switch` — routes to one of 11 action outputs (plus fallback).
- **Actions handled:** `show_lang`, `lang_selected`, `works_updated`, `show_edit_menu`, `ask_new_work`, `start_calc`, `export_excel`, `export_pdf`, `show_help`, `view_details`, `analyze_text`.
- **Fallback output:** goes to `📤 Fallback`.
- **Edge cases:**
  - If `action` mismatches any rule, user gets fallback message.

---

### Block 3 — Language selection + initial prompt
**Overview:** Presents language selection menu, confirms selection, and provides sample input formatting.  
**Nodes involved:** `📤 Lang Menu`, `Answer Lang CB`, `Prep Lang OK`, `📤 Lang OK`

#### Node: 📤 Lang Menu
- **Type / role:** `HTTP Request` — calls Telegram `sendMessage`.
- **Behavior:** Sends a “Select language” message with inline keyboard.
- **Edge cases:** Telegram API errors (token, chatId). Markdown parsing errors are possible but message is mostly plain.

#### Node: Answer Lang CB
- **Type / role:** `HTTP Request` — Telegram `answerCallbackQuery`.
- **Behavior:** Acknowledges callback to remove loading state in Telegram client.
- **onError:** `continueRegularOutput`.
- **Edge cases:** Missing `callbackQueryId` (non-callback updates).

#### Node: Prep Lang OK
- **Type / role:** `Code` — builds language-specific “Describe your project” message body.
- **Output:** `_body` JSON that includes text + keyboard (Help, Language).
- **Edge cases:** Markdown issues if localized strings contain special characters (rare here).

#### Node: 📤 Lang OK
- **Type / role:** `HTTP Request` — sends the prepared `_body` as-is to Telegram.
- **Behavior:** Posts to `sendMessage`.
- **Edge cases:** Telegram API errors; malformed body would fail.

---

### Block 4 — Text parsing (LLM) → show works list
**Overview:** Converts user text into structured work items (name/qty/unit/room) with an LLM, stores them in session, and displays an editable list with inline buttons.  
**Nodes involved:** `Prep Text LLM`, `🔧 Config Parse`, `🤖 AI Parse Text`, `📝 Parse Text Response`, `📊 Show Works`, `📤 Send Works`

#### Node: Prep Text LLM
- **Type / role:** `Code` — builds parsing prompt.
- **Key inputs:** `cfg.text` / `session.textInput` / `cfg.description`.
- **Prompt:** Asks for **ONLY valid JSON array** in the selected language; includes language-specific example.
- **Output fields:** `_parse_prompt`, `description` (truncated), `L`, `chatId`.
- **Edge cases:**
  - Empty/too short input sets `_skip_llm` and returns no works.
  - LLM may still output non-JSON; handled downstream with parsing heuristics.

#### Node: 🔧 Config Parse
- **Type / role:** `Set` — maps `_parse_prompt` into `chatInput` used by AI chain node.
- **Expression:** `chatInput = {{$json._parse_prompt}}`.

#### Node: 🤖 AI Parse Text
- **Type / role:** `chainLlm` (LangChain) — sends prompt to connected model.
- **Connected model:** `OpenAI Model 1` via `ai_languageModel`.
- **Edge cases:** Missing OpenAI credential; model rate limits.

#### Node: 📝 Parse Text Response
- **Type / role:** `Code` — parses AI output into works array.
- **Parsing strategy:**
  - Reads `aiResponse.text` or `aiResponse.response.text` or `aiResponse.output`.
  - Removes ```json fences.
  - Attempts to parse a JSON array via regex match `\[.*\]` with dotall.
  - Fallback: parse lines that start with `{`.
- **Normalization:** ensures each work has `name`, `qty` numeric, `unit`, optional `room`.
- **Session writes:** `sd.sess[cid].works`, `.description`, `.state='parsed'`.
- **Edge cases:**
  - If AI returns invalid JSON, may produce empty list (user sees “0 works”).
  - Units could be inconsistent; later stages treat unit mostly as informational.

#### Node: 📊 Show Works
- **Type / role:** `Code` — formats works list grouped by room, builds edit buttons and action buttons.
- **Output:** `msg` and `keyboard` for inline edit.
- **Session writes:** stores works/rooms/L/db and sets state `wait_edit`.
- **Edge cases:**
  - Markdown length: large work lists may approach Telegram limit (4096 chars).
  - Work names containing markdown-sensitive characters could break formatting (some escaping is done via shortening but not full escaping).

#### Node: 📤 Send Works
- **Type / role:** `HTTP Request` — Telegram `sendMessage` with `inline_keyboard`.
- **Edge cases:** Markdown parse errors; Telegram returns error if malformed markdown.

---

### Block 5 — Editing, add-work, help, details routing
**Overview:** Allows editing quantities and deleting items via callbacks, adding a new item via text, viewing help, and viewing detailed breakdown.  
**Nodes involved:** `Edit Menu`, `📤 Edit Menu`, `Works Updated`, `📤 Works Updated`, `📤 Ask New Work`, `📤 Help`, `View Details`, `📤 Details`, `📤 Fallback`

#### Node: Edit Menu
- **Type / role:** `Code` — builds per-item edit UI with quantity controls.
- **Reads:** `sd.sess[cid].editingWorkIndex`.
- **Output:** `msg`, `keyboard`, `chatId`.
- **Edge cases:** Missing/invalid index returns `_skip: true` (but downstream still might send unless checked; here output returns with `_skip` but route doesn’t branch on it).

#### Node: 📤 Edit Menu
- **Type / role:** `HTTP Request` — send edit menu message.
- **Edge cases:** none beyond Telegram API.

#### Node: Works Updated
- **Type / role:** `Code` — prints updated list and rebuilds keyboard.
- **Output:** `msg`, `keyboard`.
- **Edge cases:** Work list large → Telegram length risk.

#### Node: 📤 Works Updated
- **Type / role:** `HTTP Request` — send updated works list.
- **Edge cases:** Markdown parse issues.

#### Node: 📤 Ask New Work
- **Type / role:** `HTTP Request` — asks user to enter a new work item as text.
- **Used when:** callback `add_work`.

#### Node: 📤 Help
- **Type / role:** `HTTP Request` — sends help text and Back button.
- **Edge cases:** Hard-coded help text includes links; usually safe.

#### Node: View Details
- **Type / role:** `Code` — builds a detailed Markdown message for all works including resources and scope.
- **Reads:** `sd.lastResults` generated after aggregation.
- **Edge cases:**
  - Telegram 4096 char limit risk (details can be long).
  - Markdown special chars in resource names may break formatting.

#### Node: 📤 Details
- **Type / role:** `HTTP Request` — sends detailed view with export buttons.
- **Edge cases:** Telegram Markdown errors.

#### Node: 📤 Fallback
- **Type / role:** `HTTP Request` — sends fallback message (“Use /start to begin” or localized).
- **Used when:** Route switch fallback output is hit.

---

### Block 6 — Calculation loop (transform → embed → Qdrant → rerank → calculate)
**Overview:** For each work item, posts progress messages to Telegram, transforms query using an LLM, creates embeddings via OpenAI embeddings endpoint, searches Qdrant, reranks candidates using an LLM, computes unit/total cost and resource totals, and accumulates results.  
**Nodes involved:** `Answer Calc CB`, `📝 Prep Progress`, `📤 Send Progress`, `Save Progress ID`, `Prep Works`, `Loop`, `📝 Prep Work Msg`, `🗑️ Delete Prev`, `📤 Send Work`, `💾 Save Work Msg`, `1️⃣ Prep Query`, `🔧 Config Transform`, `🤖 AI Transform`, `2️⃣ Extract Transform`, `🔧 Config Embed`, `3️⃣ Embeddings API`, `4️⃣ Extract Embedding`, `5️⃣ Qdrant Search`, `6️⃣ Prep Rerank`, `🔧 Config Rerank`, `🤖 AI Rerank`, `8️⃣ Apply Rerank`, `9️⃣ Calculate`, `📊 Update Result`, `📤 Edit Result`, `Acc`, `🧹 Prep Cleanup`, `🗑️ Delete Work Msg`, `🗑️ Delete Progress Msg`

#### Node: Answer Calc CB
- **Type / role:** `HTTP Request` — Telegram `answerCallbackQuery` with localized “loading” text.
- **onError:** `continueRegularOutput` to avoid breaking calc start.
- **Edge cases:** callbackQueryId missing.

#### Node: 📝 Prep Progress
- **Type / role:** `Code` — creates initial progress message (“Searching prices…”) and estimates time.
- **Writes:** `session.totalWorks = totalWorks` (note: it reads session but does not explicitly write back `sd.sess[cid]` reference; however `session` is a reference to stored object, so it persists).
- **Edge cases:** If session works are empty, totalWorks=0; message still sent.

#### Node: 📤 Send Progress
- **Type / role:** `HTTP Request` — Telegram `sendMessage` for progress message.

#### Node: Save Progress ID
- **Type / role:** `Code` — stores progress message id in `sd.progress[cid]` and initializes `sd.calcProgress[cid]`.
- **Edge cases:** Telegram response may not contain `result.message_id` if API error; then cleanup uses `0`.

#### Node: Prep Works
- **Type / role:** `Code` — converts session works into an array for looping.
- **Key:** initializes accumulator `sd.res[cid] = []`.
- **Output per item fields:** `sq`, `original_query`, `work_index`, `total_works`, `db`, `L`, `currency`, `chatId`.
- **Edge cases:** If session has no works, outputs empty array; loop ends quickly.

#### Node: Loop
- **Type / role:** `SplitInBatches` — iterates over works.
- **Options:** `reset: false` (meaning it continues batches; typical for manual looping).
- **Connections:** feeds `📝 Prep Work Msg`, and after loop completion path triggers cleanup/aggregation via `🧹 Prep Cleanup`.
- **Edge cases:** If not looped correctly in n8n UI, it may only process first batch; here wiring indicates a standard “continue” cycle via `Acc → Loop`.

#### Node: 📝 Prep Work Msg
- **Type / role:** `Code` — builds per-work “🔍 Searching …” message and remembers previous message to delete.
- **Reads:** `sd.calcProgress[cid].lastMsgId`.
- **Edge cases:** None; truncates long names.

#### Node: 🗑️ Delete Prev
- **Type / role:** `HTTP Request` — Telegram `deleteMessage` for prior work status message.
- **Options:** `neverError: true` so missing messages don’t break loop.

#### Node: 📤 Send Work
- **Type / role:** `HTTP Request` — Telegram `sendMessage` for current work’s “Searching…” message.

#### Node: 💾 Save Work Msg
- **Type / role:** `Code` — saves new message id as `sd.calcProgress[cid].lastMsgId` for future edits/deletes.

#### Node: 1️⃣ Prep Query
- **Type / role:** `Code` — builds query transform prompt and attaches Qdrant credentials.
- **Key outputs:**
  - `_transform_prompt`
  - `_collection` (from `db`)
  - `_qdrant_url`, `_qdrant_key` from `🔑 TOKEN`
  - `_db_lang` derived from collection name
- **Skip logic:** if missing query or collection → `_skip: true`.
- **Edge cases:**
  - Collection name mismatch or missing → work item skipped.
  - Qdrant key empty is allowed (header still sent; some servers reject empty key header).

#### Node: 🔧 Config Transform
- **Type / role:** `Set` — maps `_transform_prompt` into `chatInput`.

#### Node: 🤖 AI Transform
- **Type / role:** `chainLlm` — LLM call for search keyword optimization.
- **Connected model:** `OpenAI Model 2` (gpt-4o-mini).

#### Node: 2️⃣ Extract Transform
- **Type / role:** `Code` — extracts clean transformed keywords and combines with original query.
- **Combination:** original query + up to 10 “new” words from transformed output.
- **Outputs:** `_query` used for embeddings.
- **Edge cases:** LLM response parsing differences across AI nodes; code handles common variants.

#### Node: 🔧 Config Embed
- **Type / role:** `Set` — sets embedding input and model config:
  - `model: text-embedding-3-large`
  - `dimensions: 3072`
- **Important:** must match Qdrant vector size.

#### Node: 3️⃣ Embeddings API
- **Type / role:** `HTTP Request` — calls OpenAI embeddings endpoint.
- **Auth:** `predefinedCredentialType` → `openAiApi` credential.
- **Body:** includes `model`, `input`, `dimensions`.
- **Edge cases:**
  - Dimension mismatch vs model capabilities could error.
  - Rate limits/timeouts.
  - If credential missing, request fails.

#### Node: 4️⃣ Extract Embedding
- **Type / role:** `Code` — pulls `embedding` from OpenAI response.
- **Validation:** warns if embedding length < 256.
- **Outputs:** `_embedding`, plus Qdrant info.
- **Edge cases:** If OpenAI returns error, sets `_embedding: []` which causes Qdrant search to be invalid or error.

#### Node: 5️⃣ Qdrant Search
- **Type / role:** `HTTP Request` — posts vector search to Qdrant.
- **Endpoint:** `{{$json._qdrant_url}}/collections/{{$json._collection}}/points/search`
- **Headers:** `api-key`, `Content-Type`.
- **Body:** vector, limit=10, `with_payload=true`.
- **Options:** timeout 30s, `neverError: true`, JSON response.
- **Edge cases:**
  - Qdrant unreachable → timeout.
  - Collection missing → 404.
  - Wrong vector size → Qdrant error.

#### Node: 6️⃣ Prep Rerank
- **Type / role:** `Code` — formats top candidates and builds rerank prompt.
- **Handles:** different payload formats (`payload_full`, `metadata`, direct).
- **Early exits:**
  - Qdrant error → returns `QDRANT_ERROR`.
  - No results → returns `NOT_FOUND`.
- **Output:** `_rerank_prompt`, `_qdrant_results`.
- **Edge cases:** Candidate formatting may include very long names; prompt length can grow.

#### Node: 🔧 Config Rerank
- **Type / role:** `Set` — sets rerank system prompt and `chatInput`.
- **Note:** `system_prompt` is set but the `🤖 AI Rerank` node only uses `chatInput` in this workflow wiring.

#### Node: 🤖 AI Rerank
- **Type / role:** `chainLlm` — asks LLM to score candidates 0–100 in JSON.

#### Node: 8️⃣ Apply Rerank
- **Type / role:** `Code` — parses rerank JSON and picks best candidate using combined score.
- **Logic:**
  - If no Qdrant results: returns `_best_result=null`, scores 0, quality `not_found`.
  - Attempts JSON parse; fallback to Qdrant score-based ranking.
  - Combined score: 70% LLM + 30% Qdrant if LLM score exists, else Qdrant score.
- **Outputs:** `_best_result`, `_best_payload`, `_llm_score`, `_qdrant_score`, `ql`.
- **Edge cases:** LLM output not JSON → fallback used.
  - Payload normalization attempts to find rate_code/name/unit in various places.

#### Node: 9️⃣ Calculate
- **Type / role:** `Code` — computes unit cost, total cost, resource breakdown, labor hours, scope-of-work.
- **Key behavior:**
  - Searches for payload in multiple possible structures (`_best_result.payload`, `_best_payload`, deep search).
  - If payload missing: returns `PAYLOAD_NOT_FOUND` diagnostic fields.
  - Computes `totalCost` from `cost_summary.total_cost_position` or sums resources if missing.
  - Detects rate unit divisor for “100 m²” / “10 m” etc.
  - Scales resources by `(workQty / unitDivisor)`.
  - Categorizes resources into labor/material/machine using `row_type` and code prefixes (DXME, ME_, PU_, RI_).
  - Outputs `uc`, `tc`, totals, `resources[]`, `scope_of_work[]`, quality fields.
- **Edge cases:**
  - Unit conversion is heuristic and limited to 10/100 patterns.
  - Resource categorization is heuristic; may misclassify.
  - Non-numeric quantities can produce NaNs if not normalized upstream.

#### Node: 📊 Update Result
- **Type / role:** `Code` — formats a short per-work result line and decides “found” status.
- **Edits:** uses `sd.calcProgress[cid].lastMsgId` as message to edit.
- **Edge cases:** If lastMsgId missing, edit uses 0 → Telegram error but `neverError` is set downstream.

#### Node: 📤 Edit Result
- **Type / role:** `HTTP Request` — Telegram `editMessageText` to update the “Searching…” message into a “✓ Found” / “Not found” result.
- **Options:** `neverError: true`.

#### Node: Acc
- **Type / role:** `Code` — pushes each work result into `sd.res[cid]` and then returns item to continue loop.
- **Connection:** `Acc → Loop` to fetch next work item.
- **Edge cases:** StaticData growth if loop never finishes; but later cleanup clears `sd.res[cid]`.

#### Node: 🧹 Prep Cleanup
- **Type / role:** `Code` — after loop completion, prepares message IDs for deletion:
  - last work message id
  - initial progress message id
- **Edge cases:** ids null → downstream uses 0.

#### Node: 🗑️ Delete Work Msg / 🗑️ Delete Progress Msg
- **Type / role:** `HTTP Request` — Telegram `deleteMessage` calls, `neverError: true`.
- **Purpose:** remove progress UI clutter.

---

### Block 7 — Aggregation + reporting + final delivery
**Overview:** Aggregates all work results, generates an interactive HTML report (stored in static data), sends final summary message, and sends HTML file.  
**Nodes involved:** `Agg`, `Generate HTML`, `Final`, `📤 Final`, `Prep HTML File`, `📤 Send HTML`

#### Node: Agg
- **Type / role:** `Code` — aggregates `sd.res[cid]` into totals.
- **Computes:**
  - total cost, sums by workers/materials/machines, labor hours
  - found percentage
- **Writes / cleanup:**
  - deletes `sd.res[cid]`, `sd.calcProgress[cid]`, `sd.progress[cid]`
  - stores `sd.lastResults = { works, total, ... , L }` for exports/details
- **Edge cases:** If `sd.res[cid]` empty, totals are 0 and pct 0.

#### Node: Generate HTML
- **Type / role:** `Code` — builds HTML with expandable rows for resources and optional scope-of-work rows.
- **Stores:** `sd.html_report = html`.
- **Output:** `html_content` and full aggregated data.
- **Edge cases:** Very large resources lists → large HTML file; Telegram may reject very large documents.

#### Node: Final
- **Type / role:** `Code` — builds a compact Markdown summary safe for Telegram.
- **Important:** aggressively strips markdown special chars via `esc()` to prevent Telegram parse errors.
- **Limits:** truncates to ~3800 chars and only lists up to 30 items.
- **Session:** marks `sd.sess[cid].state = 'done'`.

#### Node: 📤 Final
- **Type / role:** `HTTP Request` — Telegram `sendMessage` with inline buttons:
  - Resources/details
  - Excel, PDF
  - Restart
- **Edge cases:** minimal due to sanitization in `Final`.

#### Node: Prep HTML File
- **Type / role:** `Code` — converts HTML string into binary attachment `html` and creates filename.
- **Output binary:** `binary.html` base64.

#### Node: 📤 Send HTML
- **Type / role:** `Telegram` node `sendDocument` — sends the HTML file to the chat.

---

### Block 8 — Export (CSV and PDF placeholder)
**Overview:** Exports results: CSV (Excel-compatible), and a “PDF” export that currently sends HTML as a document (not true PDF conversion).  
**Nodes involved:** `Generate Excel`, `📤 Send Excel`, `Generate PDF`, `IF PDF`, `📤 Send PDF`

#### Node: Generate Excel
- **Type / role:** `Code` — builds CSV with BOM for UTF-8.
- **Source:** `sd.lastResults`.
- **Binary output:** `binary.excel` (`text/csv`).
- **Edge cases:** Semicolon separator is used; good for many EU locales but not universal.

#### Node: 📤 Send Excel
- **Type / role:** `Telegram` sendDocument — sends CSV file.

#### Node: Generate PDF
- **Type / role:** `Code` — checks `sd.html_report`.
- **Current behavior:** If exists, sends HTML content as binary under property `pdf` with mimeType `text/html` and filename `.html`.
- **If missing:** sends a Telegram message “No report to export. Please calculate first.” and returns `{skip:true}`.
- **Edge cases:** Not a real PDF; naming/caption might confuse users.

#### Node: IF PDF
- **Type / role:** `IF` — only send if `skip != true`.

#### Node: 📤 Send PDF
- **Type / role:** `Telegram` sendDocument — sends the “pdf” binary (actually HTML).

---

## 3. Summary Table (all nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note1 | stickyNote | Branding / GitHub link |  |  | ⭐ **Star us on GitHub!** / [github.com/datadrivenconstruction/DDC-CWICR](https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR) / **DDC CWICR** — Open Source Construction Cost Database / - 55,000+ work items / - 9 languages / - Free forever |
| 🔐 Credentials Setup | stickyNote | Credentials instructions |  |  | ## 🔐 Credentials Setup / TOKEN: bot_token, QDRANT_URL, QDRANT_API_KEY / n8n Credentials: OpenAI, Anthropic (optional), Gemini (optional) / Switch models by enabling/disabling |
| Checklist | stickyNote | Setup checklist |  |  | ## ✅ SETUP CHECKLIST / Telegram Bot, n8n Credentials, Qdrant, Test flow |
| Intro | stickyNote | Product intro |  |  | ## 🚀 DDC CWICR Text Estimator / Version 11.0 AI Nodes / AI via n8n Credentials / Features / No API keys in code |
| 🔑 TOKEN | set | Stores bot/Qdrant config values | Telegram Trigger1 | Main | ## ⚠️ Setup 🔑 TOKEN / bot_token, QDRANT_URL / token via @BotFather |
| UI Messages | stickyNote | UI customization hint |  |  | ## 🌍 UI Messages / All localized in Config node / Edit LANG object |
| Route Switch | stickyNote | Action routing doc |  |  | ## 🔀 Route Switch / 11 actions table |
| Config & Localization | stickyNote | Localization description |  |  | ## 🌐 Config & Localization / 9 languages / auto-selects Qdrant collection + currency |
| Main Router | stickyNote | Router description |  |  | ## 🧠 Main Router / parses updates, sessions, routes action codes |
| Agg | code | Aggregate per-work results and totals | 🗑️ Delete Progress Msg | Generate HTML | ## 📊 Block 7: Reports / Aggregate → Generate HTML → Send |
| 🗑️ Delete Progress Msg | httpRequest | Telegram delete progress message | 🗑️ Delete Work Msg | Agg | ## 🔄 Block 6: Calculation Loop |
| 🗑️ Delete Work Msg | httpRequest | Telegram delete last work message | 🧹 Prep Cleanup | 🗑️ Delete Progress Msg | ## 🔄 Block 6: Calculation Loop |
| 🧹 Prep Cleanup | code | Prepare message IDs for cleanup | Loop | 🗑️ Delete Work Msg | ## 🔄 Block 6: Calculation Loop |
| Acc | code | Accumulate each calculated work into staticData | 📤 Edit Result | Loop | ## 🔄 Block 6: Calculation Loop |
| 📤 Edit Result | httpRequest | Edit per-work Telegram message with result | 📊 Update Result | Acc | ## 🔄 Block 6: Calculation Loop |
| 📊 Update Result | code | Format “found/not found” result text | 9️⃣ Calculate | 📤 Edit Result | ## 🔄 Block 6: Calculation Loop |
| 1️⃣ Prep Query | code | Build transform prompt + add Qdrant creds | 💾 Save Work Msg | 🔧 Config Transform | ## 🔄 Block 6: Calculation Loop / (also Qdrant Info nearby) |
| 💾 Save Work Msg | code | Save message_id for later edit/delete | 📤 Send Work | 1️⃣ Prep Query | ## 🔄 Block 6: Calculation Loop |
| 📤 Send Work | httpRequest | Send per-work “searching” message | 🗑️ Delete Prev | 💾 Save Work Msg | ## 🔄 Block 6: Calculation Loop |
| 🗑️ Delete Prev | httpRequest | Delete previous per-work message | 📝 Prep Work Msg | 📤 Send Work | ## 🔄 Block 6: Calculation Loop |
| 📝 Prep Work Msg | code | Create localized “searching” message | Loop | 🗑️ Delete Prev | ## 🔄 Block 6: Calculation Loop |
| Loop | splitInBatches | Iterate works | Prep Works, Acc | 🧹 Prep Cleanup, 📝 Prep Work Msg | ## 🔄 Block 6: Calculation Loop |
| Prep Works | code | Prepare array of works for loop | Save Progress ID | Loop | ## 🔄 Block 6: Calculation Loop |
| Save Progress ID | code | Store progress message id, init trackers | 📤 Send Progress | Prep Works | ## 🔄 Block 6: Calculation Loop |
| 📤 Send Progress | httpRequest | Send calculation progress message | 📝 Prep Progress | Save Progress ID | ## 🔄 Block 6: Calculation Loop |
| 📝 Prep Progress | code | Build localized progress header | Answer Calc CB | 📤 Send Progress | ## 🔄 Block 6: Calculation Loop |
| 📤 Send Works | httpRequest | Show works list with edit keyboard | 📊 Show Works |  |  |
| 📊 Show Works | code | Format works list + inline keyboard | 📝 Parse Text Response | 📤 Send Works |  |
| 9️⃣ Calculate | code | Compute costs/resources from best payload | 8️⃣ Apply Rerank | 📊 Update Result | ## 🔄 Block 6: Calculation Loop |
| 8️⃣ Apply Rerank | code | Parse LLM ranking + pick best candidate | 🤖 AI Rerank | 9️⃣ Calculate | ## 🔄 Block 6: Calculation Loop |
| 6️⃣ Prep Rerank | code | Build rerank prompt from Qdrant results | 5️⃣ Qdrant Search | 🔧 Config Rerank | ## 🔄 Block 6: Calculation Loop |
| 5️⃣ Qdrant Search | httpRequest | Vector similarity search in Qdrant | 4️⃣ Extract Embedding | 6️⃣ Prep Rerank | ## 🔄 Block 6: Calculation Loop / ## 🔍 Qdrant Vector DB |
| 4️⃣ Extract Embedding | code | Extract embedding array from OpenAI response | 3️⃣ Embeddings API | 5️⃣ Qdrant Search | ## 🔄 Block 6: Calculation Loop / ## 🧮 Embeddings |
| 2️⃣ Extract Transform | code | Clean transform output + combine query words | 🤖 AI Transform | 🔧 Config Embed | ## 🔄 Block 6: Calculation Loop |
| 📤 Details | httpRequest | Send detailed breakdown message + export buttons | View Details |  | ## 📥 Block 8: Export |
| View Details | code | Build detailed Markdown of works/resources/scope | Route (DETAILS) | 📤 Details | ## 📥 Block 8: Export |
| 📤 Fallback | httpRequest | Fallback “use /start” message | Route (fallback) |  |  |
| 📤 Help | httpRequest | Send help message | Route (HELP) |  |  |
| 📤 Send PDF | telegram | Send “PDF” document (actually HTML) | IF PDF |  | ## 📥 Block 8: Export |
| IF PDF | if | Skip sending if no report | Generate PDF | 📤 Send PDF | ## 📥 Block 8: Export |
| Generate PDF | code | Prepare “PDF” payload or skip | Route (PDF) | IF PDF | ## 📥 Block 8: Export |
| 📤 Send Excel | telegram | Send CSV document | Generate Excel |  | ## 📥 Block 8: Export |
| Generate Excel | code | Generate CSV from lastResults | Route (EXCEL) | 📤 Send Excel | ## 📥 Block 8: Export |
| 📤 Send HTML | telegram | Send HTML report file | Prep HTML File |  | ## 📥 Block 8: Export |
| Prep HTML File | code | Convert HTML string to binary attachment | Final | 📤 Send HTML | ## 📊 Block 7: Reports |
| 📤 Final | httpRequest | Send final Telegram summary + buttons | Final |  | ## 📊 Block 7: Reports |
| Final | code | Build compact safe Markdown summary | Generate HTML | Prep HTML File, 📤 Final | ## 📊 Block 7: Reports |
| Generate HTML | code | Build interactive HTML report + store in staticData | Agg | Final | ## 📊 Block 7: Reports |
| Answer Calc CB | httpRequest | answerCallbackQuery “loading…” | Route (CALC) | 📝 Prep Progress |  |
| 📤 Works Updated | httpRequest | Send updated works list | Works Updated |  |  |
| Works Updated | code | Format updated works list + keyboard | Route (WORKS_UPD) | 📤 Works Updated |  |
| 📤 Ask New Work | httpRequest | Prompt user to add work item | Route (ADD_WORK) |  |  |
| Edit Menu | code | Build edit menu for selected work | Route (EDIT_MENU) | 📤 Edit Menu |  |
| 📤 Lang OK | httpRequest | Send language confirmation prompt | Prep Lang OK |  |  |
| Answer Lang CB | httpRequest | answerCallbackQuery (language) | Route (LANG_OK) | Prep Lang OK |  |
| 📤 Lang Menu | httpRequest | Send language selection menu | Route (LANG) |  |  |
| Route | switch | Route by action to blocks | Config | many | ## 🔀 Route Switch |
| Config | code | Localization + DB mapping | Main | Route | ## 🌐 Config & Localization |
| Main | code | Session router/state machine | 🔑 TOKEN | Config | ## 🧠 Main Router |
| Telegram Trigger1 | telegramTrigger | Entry point |  | 🔑 TOKEN |  |
| Prep Lang OK | code | Build language OK message body | Answer Lang CB | 📤 Lang OK |  |
| Prep Text LLM | code | Build parse prompt for works extraction | Route (ANALYZE_TEXT) | 🔧 Config Parse | ## 🤖 AI Parse Text |
| 📤 Edit Menu | httpRequest | Send edit menu message | Edit Menu |  |  |
| 🔧 Config Parse | set | Map parse prompt to AI input | Prep Text LLM | 🤖 AI Parse Text | ## 🤖 AI Parse Text |
| 🤖 AI Parse Text | chainLlm | LLM call to extract works JSON | 🔧 Config Parse + OpenAI Model 1 | 📝 Parse Text Response | ## 🤖 AI Parse Text |
| OpenAI Model 1 | lmChatOpenAi | LLM model for parsing |  | 🤖 AI Parse Text (ai_languageModel) | ## 🤖 AI Parse Text |
| 🔧 Config Transform | set | Map transform prompt to AI input | 1️⃣ Prep Query | 🤖 AI Transform | ## 🤖 AI Transform & Rerank |
| 🤖 AI Transform | chainLlm | LLM call to optimize search query | 🔧 Config Transform + OpenAI Model 2 | 2️⃣ Extract Transform | ## 🤖 AI Transform & Rerank |
| OpenAI Model 2 | lmChatOpenAi | LLM model for transform & rerank |  | 🤖 AI Transform, 🤖 AI Rerank (ai_languageModel) | ## 🤖 AI Transform & Rerank |
| 🔧 Config Rerank | set | Map rerank prompt to AI input | 6️⃣ Prep Rerank | 🤖 AI Rerank | ## 🤖 AI Transform & Rerank |
| 🤖 AI Rerank | chainLlm | LLM call to score candidates | 🔧 Config Rerank + OpenAI Model 2 | 8️⃣ Apply Rerank | ## 🤖 AI Transform & Rerank |
| 📝 Parse Text Response | code | Parse LLM output into works[] + store session | 🤖 AI Parse Text | 📊 Show Works |  |
| 🔧 Config Embed | set | Configure embeddings model/dimensions | 2️⃣ Extract Transform | 3️⃣ Embeddings API | ## 🧮 Embeddings |
| 3️⃣ Embeddings API | httpRequest | OpenAI embeddings API call | 🔧 Config Embed | 4️⃣ Extract Embedding | ## 🧮 Embeddings |
| Google Gemini Chat Model | lmChatGoogleGemini | Disabled alt model |  |  | ## 🧠 Available AI Models |
| Anthropic Chat Model2 | lmChatAnthropic | Disabled alt model |  |  | ## 🧠 Available AI Models |
| OpenRouter Chat Model1 | lmChatOpenRouter | Disabled alt model |  |  | ## 🧠 Available AI Models |
| LLM Models | stickyNote | Model availability info |  |  | ## 🧠 Available AI Models |
| DeepSeek Chat Model | lmChatDeepSeek | Disabled alt model |  |  | ## 🧠 Available AI Models |
| Sticky Note | stickyNote | Empty note |  |  |  |
| Block 6 - Calculation | stickyNote | Block documentation |  |  | ## 🔄 Block 6: Calculation Loop (content as shown) |
| Block 7 - Reports | stickyNote | Block documentation |  |  | ## 📊 Block 7: Reports (content as shown) |
| Block 8 - Export | stickyNote | Block documentation |  |  | ## 📥 Block 8: Export (content as shown) |
| Qdrant Info | stickyNote | Qdrant collections/setup |  |  | ## 🔍 Qdrant Vector DB (content as shown) |
| Sticky AI Parse | stickyNote | AI parse instructions |  |  | ## 🤖 AI Parse Text (content as shown) |
| Sticky AI Transform Rerank | stickyNote | AI transform/rerank instructions |  |  | ## 🤖 AI Transform & Rerank (content as shown) |
| Token Setup | stickyNote | Token setup instructions |  |  | ## ⚠️ Setup 🔑 TOKEN (content as shown) |
| Sticky Embeddings | stickyNote | Embeddings config note |  |  | ## 🧮 Embeddings (content as shown) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger**
   1. Add node **Telegram Trigger**.
   2. Updates: enable **message** and **callback_query**.
   3. Configure **Telegram API credential** in n8n (BotFather token).

2. **Add TOKEN config node**
   1. Add **Set** node named `🔑 TOKEN`.
   2. Add string fields:
      - `bot_token` = your Telegram bot token (used by raw HTTP Telegram API nodes).
      - `QDRANT_URL` = e.g. `http://localhost:6333`
      - `QDRANT_API_KEY` = optional
   3. Connect: `Telegram Trigger1 → 🔑 TOKEN`.

3. **Add main router (session state machine)**
   1. Add **Code** node named `Main`.
   2. Paste logic that:
      - reads `Telegram Trigger1` update payload
      - manages `sd.sess[cid]`
      - outputs `action` among: `show_lang`, `lang_selected`, `works_updated`, `show_edit_menu`, `ask_new_work`, `start_calc`, `export_excel`, `export_pdf`, `show_help`, `view_details`, `analyze_text`
   3. Connect: `🔑 TOKEN → Main`.

4. **Add localization/config node**
   1. Add **Code** node named `Config`.
   2. Implement `LANGS` dictionary with 9 languages and fields:
      - `db` (Qdrant collection name)
      - `cur`, `sym`, `loc`, `region`, UI strings
   3. Update session: `sd.sess[chatId].db = L.db; sd.sess[chatId].L = L`.
   4. Connect: `Main → Config`.

5. **Add Route Switch**
   1. Add **Switch** node named `Route`.
   2. Switch on `{{$json.action}}`.
   3. Create named outputs for each action:
      - `show_lang`, `lang_selected`, `works_updated`, `show_edit_menu`, `ask_new_work`, `start_calc`, `export_excel`, `export_pdf`, `show_help`, `view_details`, `analyze_text`
   4. Set fallback output enabled.
   5. Connect: `Config → Route`.

6. **Language menu and confirmation**
   1. Add **HTTP Request** `📤 Lang Menu` posting to Telegram `sendMessage` with inline keyboard of languages.
   2. Add **HTTP Request** `Answer Lang CB` posting to `answerCallbackQuery` using `callbackQueryId`.
   3. Add **Code** `Prep Lang OK` generating `_body` message.
   4. Add **HTTP Request** `📤 Lang OK` sending `_body`.
   5. Connect: `Route(show_lang) → 📤 Lang Menu` and `Route(lang_selected) → Answer Lang CB → Prep Lang OK → 📤 Lang OK`.

7. **Text → works parsing with AI**
   1. Add **Code** `Prep Text LLM` to produce `_parse_prompt` based on `cfg.text`/session.
   2. Add **Set** `🔧 Config Parse` mapping `chatInput = {{$json._parse_prompt}}`.
   3. Add **AI Chain (chainLlm)** node `🤖 AI Parse Text`.
   4. Add **OpenAI Chat Model** node `OpenAI Model 1`:
      - Model: `chatgpt-4o-latest` (or your choice)
      - Temperature ~0.15
      - Attach **OpenAI API credential** (n8n credential).
   5. Connect model to chain via **ai_languageModel** connection.
   6. Add **Code** `📝 Parse Text Response` to parse returned JSON into `works[]` and save to `sd.sess[cid].works`.
   7. Add **Code** `📊 Show Works` to format message and keyboard.
   8. Add **HTTP Request** `📤 Send Works` to Telegram `sendMessage`.
   9. Connect: `Route(analyze_text) → Prep Text LLM → 🔧 Config Parse → 🤖 AI Parse Text → 📝 Parse Text Response → 📊 Show Works → 📤 Send Works`.

8. **Editing and add-work**
   1. Add **Code** `Edit Menu` and **HTTP Request** `📤 Edit Menu`; connect: `Route(show_edit_menu) → Edit Menu → 📤 Edit Menu`.
   2. Add **Code** `Works Updated` and **HTTP Request** `📤 Works Updated`; connect: `Route(works_updated) → Works Updated → 📤 Works Updated`.
   3. Add **HTTP Request** `📤 Ask New Work`; connect: `Route(ask_new_work) → 📤 Ask New Work`.
   4. Add **HTTP Request** `📤 Help`; connect: `Route(show_help) → 📤 Help`.
   5. Add **Code** `View Details` and **HTTP Request** `📤 Details`; connect: `Route(view_details) → View Details → 📤 Details`.
   6. Add **HTTP Request** `📤 Fallback`; connect `Route(fallback) → 📤 Fallback`.

9. **Start calculation (callback acknowledgement + progress message)**
   1. Add **HTTP Request** `Answer Calc CB` to `answerCallbackQuery` with localized loading.
   2. Add **Code** `📝 Prep Progress` and **HTTP Request** `📤 Send Progress`.
   3. Add **Code** `Save Progress ID` to store `sd.progress[cid].message_id` and init `sd.calcProgress[cid]`.
   4. Connect: `Route(start_calc) → Answer Calc CB → 📝 Prep Progress → 📤 Send Progress → Save Progress ID`.

10. **Prepare works and loop**
    1. Add **Code** `Prep Works` to output one item per work and init `sd.res[cid]=[]`.
    2. Add **SplitInBatches** `Loop` (batch size default; reset=false).
    3. Connect: `Save Progress ID → Prep Works → Loop`.

11. **Per-work Telegram “searching” message**
    1. Add **Code** `📝 Prep Work Msg` (reads current Loop item and `sd.calcProgress[cid].lastMsgId`).
    2. Add **HTTP Request** `🗑️ Delete Prev` to Telegram `deleteMessage` (`neverError: true`).
    3. Add **HTTP Request** `📤 Send Work` to Telegram `sendMessage`.
    4. Add **Code** `💾 Save Work Msg` to store new `lastMsgId`.
    5. Connect: `Loop → 📝 Prep Work Msg → 🗑️ Delete Prev → 📤 Send Work → 💾 Save Work Msg`.

12. **Transform query (AI)**
    1. Add **Code** `1️⃣ Prep Query` to build `_transform_prompt` and attach Qdrant URL/key and collection name.
    2. Add **Set** `🔧 Config Transform` mapping prompt into `chatInput`.
    3. Add **AI Chain** `🤖 AI Transform`.
    4. Add **OpenAI Chat Model** `OpenAI Model 2` (e.g. `gpt-4o-mini`, temp 0.3) and connect to `🤖 AI Transform` (ai_languageModel).
    5. Add **Code** `2️⃣ Extract Transform` to produce `_query`.
    6. Connect: `💾 Save Work Msg → 1️⃣ Prep Query → 🔧 Config Transform → 🤖 AI Transform → 2️⃣ Extract Transform`.

13. **Embeddings**
    1. Add **Set** `🔧 Config Embed`:
       - `text = {{$json._query}}`
       - `model = text-embedding-3-large`
       - `dimensions = 3072`
    2. Add **HTTP Request** `3️⃣ Embeddings API`:
       - POST `https://api.openai.com/v1/embeddings`
       - Auth: **OpenAI credential** (predefined credential type `openAiApi`)
       - Body includes model, input, dimensions.
    3. Add **Code** `4️⃣ Extract Embedding`.
    4. Connect: `2️⃣ Extract Transform → 🔧 Config Embed → 3️⃣ Embeddings API → 4️⃣ Extract Embedding`.

14. **Qdrant search**
    1. Add **HTTP Request** `5️⃣ Qdrant Search`:
       - POST `{{$json._qdrant_url}}/collections/{{$json._collection}}/points/search`
       - Headers: `Content-Type: application/json`, `api-key: {{$json._qdrant_key}}`
       - Body: `{ vector: _embedding, limit: 10, with_payload: true, with_vector: false }`
       - timeout 30000, `neverError: true`.
    2. Connect: `4️⃣ Extract Embedding → 5️⃣ Qdrant Search`.

15. **Rerank candidates (AI)**
    1. Add **Code** `6️⃣ Prep Rerank` to build `_rerank_prompt` from top candidates.
    2. Add **Set** `🔧 Config Rerank` mapping `_rerank_prompt` to `chatInput` (and optionally system prompt).
    3. Add **AI Chain** `🤖 AI Rerank` using same `OpenAI Model 2` connection (ai_languageModel).
    4. Add **Code** `8️⃣ Apply Rerank` to select `_best_payload`.
    5. Connect: `5️⃣ Qdrant Search → 6️⃣ Prep Rerank → 🔧 Config Rerank → 🤖 AI Rerank → 8️⃣ Apply Rerank`.

16. **Calculate and update per-work message**
    1. Add **Code** `9️⃣ Calculate`.
    2. Add **Code** `📊 Update Result`.
    3. Add **HTTP Request** `📤 Edit Result` to Telegram `editMessageText` with `neverError: true`.
    4. Add **Code** `Acc` to push into `sd.res[cid]` and return item.
    5. Connect: `8️⃣ Apply Rerank → 9️⃣ Calculate → 📊 Update Result → 📤 Edit Result → Acc`.
    6. Connect loop continuation: `Acc → Loop` (main output index 0).

17. **Cleanup messages after loop completes**
    1. Connect `Loop` completion output to `🧹 Prep Cleanup`.
    2. Add `🗑️ Delete Work Msg` and `🗑️ Delete Progress Msg` (both Telegram `deleteMessage`, `neverError: true`).
    3. Connect: `🧹 Prep Cleanup → 🗑️ Delete Work Msg → 🗑️ Delete Progress Msg`.

18. **Aggregate + generate HTML + send final**
    1. Add **Code** `Agg` to aggregate `sd.res[cid]` and store `sd.lastResults`.
    2. Add **Code** `Generate HTML` to generate report and store `sd.html_report`.
    3. Add **Code** `Final` to build compact Telegram message.
    4. Add **HTTP Request** `📤 Final` to Telegram `sendMessage` with inline buttons.
    5. Add **Code** `Prep HTML File` to create binary `html`.
    6. Add **Telegram** node `📤 Send HTML` to send document from binary property `html`.
    7. Connect: `🗑️ Delete Progress Msg → Agg → Generate HTML → Final → (📤 Final and Prep HTML File) → 📤 Send HTML`.

19. **Exports**
    - **Excel (CSV):**
      1. Add **Code** `Generate Excel` reading `sd.lastResults` and emitting `binary.excel`.
      2. Add **Telegram** node `📤 Send Excel` (sendDocument, binary property `excel`).
      3. Connect: `Route(export_excel) → Generate Excel → 📤 Send Excel`.
    - **PDF (currently HTML-as-document):**
      1. Add **Code** `Generate PDF` reading `sd.html_report`; if missing, return `{skip:true}`.
      2. Add **IF** `IF PDF` condition: `{{$json.skip}} != true`.
      3. Add **Telegram** node `📤 Send PDF` sending binary property `pdf`.
      4. Connect: `Route(export_pdf) → Generate PDF → IF PDF → 📤 Send PDF`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DDC CWICR repository link | https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR |
| DDC CWICR site | https://DataDrivenConstruction.io |
| Workflow supports 9 languages and selects Qdrant collection per language | Implemented in `Config` node (`LANGS[lang].db`) |
| Embeddings configured for 3072 dimensions (must match Qdrant collection vectors) | `🔧 Config Embed` + Qdrant collections |
| Export “PDF” is not a real PDF conversion in this version | `Generate PDF` sends HTML as document |

**Disclaimer (provided):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.