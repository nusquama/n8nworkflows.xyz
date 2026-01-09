Optimize Instagram hashtags with GPT-4o & real engagement data via Graph API

https://n8nworkflows.xyz/workflows/optimize-instagram-hashtags-with-gpt-4o---real-engagement-data-via-graph-api-11994


# Optimize Instagram hashtags with GPT-4o & real engagement data via Graph API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Optimize Instagram hashtags with GPT-4o & real engagement data via Graph API  
**Workflow name (internal):** Generate Instagram hashtags using OpenAI and Graph API

This workflow generates **5 optimized Instagram hashtags** for a given post caption by:
1) generating initial hashtag ideas with OpenAI,  
2) enriching/ranking them using **real engagement signals** (average likes/comments) fetched via the **Instagram Graph API**,  
3) caching computed metrics in **Google Sheets** to reduce repeated API calls,  
4) asking an LLM to select the final top 5 balancing relevance + engagement.

### 1.1 Input Reception (two entry modes)
- Manual execution for testing (caption set in-node)
- Sub-workflow execution (caption passed as input to an Execute Workflow Trigger)

### 1.2 AI Candidate Generation
- LLM generates 10 candidate hashtags from the caption
- Code parses and normalizes tags

### 1.3 Cache Lookup & Routing
- Pull cached hashtag metrics from Google Sheets
- Split candidates into “cached” vs “new” based on sheet content

### 1.4 Instagram Metrics Fetch + Cache Write-back (for new tags)
- For each uncached tag: search hashtag ID, fetch top media metrics, compute averages
- Append computed metrics back into Google Sheets
- Rate limiting wait and batch iteration

### 1.5 Aggregate + Final AI Selection
- Merge cached + newly fetched metrics
- Rank candidates by average likes, keep top 15
- LLM selects best 5 hashtags based on caption relevance + engagement
- Format output as a single string of “#tag #tag …”

---

## 2. Block-by-Block Analysis

### Block 1 — Input Processing
**Overview:** Accepts a caption either from a manual test path or as input when invoked by another workflow.

**Nodes involved:**
- Manual Trigger
- Set Dummy Caption
- Workflow Trigger (ExecuteWorkflowTrigger)

#### Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — starts workflow manually.
- **Config:** No parameters.
- **Outputs:** To **Set Dummy Caption**.
- **Edge cases:** None (manual only).

#### Set Dummy Caption
- **Type / role:** `n8n-nodes-base.set` — sets `caption` for manual testing and also triggers cache fetch in this workflow design.
- **Config choices:** Creates a single field:
  - `caption` (string): demo caption text.
- **Outputs:** 
  - To **Generate Initial Hashtags**
  - To **Fetch Cached Hashtags**
- **Edge cases:** If you forget to update caption here in Manual mode, results will be based on the dummy text.

#### Workflow Trigger
- **Type / role:** `n8n-nodes-base.executeWorkflowTrigger` — allows this workflow to be called as a sub-workflow.
- **Config choices:** Defines one expected input:
  - `caption`
- **Outputs:** To **Generate Initial Hashtags**
- **Edge cases / integration note:** In the current connections, the sub-workflow path **does not trigger “Fetch Cached Hashtags”**, so caching logic can be partially bypassed unless you add a connection from this trigger to the Sheets fetch (see Section 4).

---

### Block 2 — Idea Generation (LLM → 10 tags → parsing)
**Overview:** Uses OpenAI chat model to propose 10 hashtags, then parses the JSON list and normalizes each tag.

**Nodes involved:**
- OpenAI Chat Model
- Generate Initial Hashtags
- Parse & Clean Tags

#### OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat model resource for LangChain nodes.
- **Config choices:**
  - Model: `gpt-4o-mini`
  - Uses OpenAI API credential.
- **Connections:**
  - Output (ai_languageModel) → **Generate Initial Hashtags**
- **Failure modes:** credential missing/invalid, rate limits, OpenAI service errors.

#### Generate Initial Hashtags
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — prompts LLM to output hashtags in strict JSON array format.
- **Key prompt logic:**
  - Uses `{{ $json.caption }}`
  - Requires output exactly like: `["keyword1","keyword2",...]`
- **Config choices:**
  - `promptType: define`
  - `hasOutputParser: true` (expects structured output)
- **Input:** Caption from Manual/Workflow trigger path.
- **Output:** `text` field containing model output.
- **Failure modes / edge cases:**
  - Model returns markdown fences or extra prose → downstream parser may fail.
  - Non-JSON output will break parsing.

#### Parse & Clean Tags
- **Type / role:** `n8n-nodes-base.code` — extracts JSON array from LLM text and emits one item per tag.
- **Key code behavior:**
  - Reads: `$input.first().json.text`
  - Regex extracts first `[...]` block: `/\[.*\]/s`
  - `JSON.parse(...)`
  - Removes `#` prefix from each tag and outputs items as `{ tag: "..." }`
- **Output:** Many items, one per tag.
- **Failure modes:**
  - Throws `No JSON array found` if regex fails.
  - `JSON.parse` error if malformed JSON.
- **Mitigation idea:** Strip ```json fences, or use a stricter output parser.

---

### Block 3 — Caching Logic (Google Sheets + cache decision)
**Overview:** Loads cached hashtag metrics, merges them with generated tags, and decides which tags need fresh Graph API calls.

**Nodes involved:**
- Fetch Cached Hashtags
- Merge Input
- Check Cache Status
- Route Cached vs New

#### Fetch Cached Hashtags
- **Type / role:** `n8n-nodes-base.googleSheets` — reads cached tag metrics.
- **Config choices:**
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet: `YOUR_SHEET_NAME_OR_ID`
  - `onError: continueRegularOutput` (workflow continues even if read fails)
- **Inputs:** Connected from **Set Dummy Caption** only (manual path).
- **Outputs:** To **Merge Input**
- **Failure modes:**
  - Auth errors (OAuth)
  - Sheet not found / permissions
  - Changed column names
- **Edge case:** In sub-workflow mode, this node may never run unless connected.

#### Merge Input
- **Type / role:** `n8n-nodes-base.merge` — combines two streams:
  - stream A: parsed AI tags (from **Parse & Clean Tags**)
  - stream B: sheet rows (from **Fetch Cached Hashtags**)
- **Config choices:** default merge behavior (n8n Merge node default).
- **Output:** To **Check Cache Status**
- **Failure modes:** If one input is missing (e.g., sub-workflow path), behavior depends on merge mode; could produce only one side or wait for both depending on node settings (not specified here).

#### Check Cache Status
- **Type / role:** `n8n-nodes-base.code` — determines for each AI tag whether cache has valid metrics.
- **Key logic:**
  - Separates `sheetRows` by presence of `tag` and (`average_likes` or `media_count`)
  - Treats items without those metrics as AI tags
  - Normalizes tags to lowercase and strips `#`
  - Builds map: `cleanTag -> { average_likes, average_comments, media_count }`
  - Outputs items with `source: 'cache'` if cached and `average_likes` is a number; else `source: 'new'`
- **Outputs:** To **Route Cached vs New**
- **Failure modes / edge cases:**
  - If Sheets returns strings, numeric conversion may yield `NaN` → treated as uncached.
  - Cache is only considered valid if `average_likes` is a valid number.

#### Route Cached vs New
- **Type / role:** `n8n-nodes-base.if` — routes items based on cache status.
- **Condition:** `{{ $json.source }} equals "cache"`
- **Outputs:**
  - **True** (cached) → **Merge Cache & Fresh** (input index 1)
  - **False** (new) → **Split In Batches**
- **Failure modes:** If `source` missing, item goes to “new” path implicitly (condition fails).

---

### Block 4 — Instagram Data & Caching (uncached tags)
**Overview:** For each new tag, query Instagram Graph API to find hashtag, fetch top media engagement, compute average likes/comments, store into Google Sheets, and iterate.

**Nodes involved:**
- Split In Batches
- Get Hashtag Info
- Check if Tag Exists
- Get Hashtag Metrics
- Calculate Average Metrics
- Format New Hashtag Data
- Save to Cache
- Rate Limit Wait

#### Split In Batches
- **Type / role:** `n8n-nodes-base.splitInBatches` — processes uncached tags sequentially.
- **Config:** default options (batch size not explicitly set → n8n default is typically 1).
- **Inputs:** From **Route Cached vs New** (new tags).
- **Outputs:**
  - To **Merge Cache & Fresh** (so new tags can still be aggregated)
  - To **Get Hashtag Info** (to fetch metrics)
- **Edge cases:** If batch size > 1, Graph API calls can spike and rate limit more easily.

#### Get Hashtag Info
- **Type / role:** `n8n-nodes-base.facebookGraphApi` — `ig_hashtag_search`
- **Config choices:**
  - Graph API version: `v18.0`
  - Query params:
    - `user_id = YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID`
    - `q = {{ $('Split In Batches').item.json.tag }}`
  - `onError: continueErrorOutput` (continues even when API fails, producing an error output structure)
- **Outputs:**
  - To **Check if Tag Exists**
  - Also to **Rate Limit Wait** (regardless, per current connections)
- **Failure modes:**
  - Invalid/expired token
  - Wrong IG business account id
  - Permissions missing (`instagram_basic`, etc.)
  - Rate limiting / transient 5xx

#### Check if Tag Exists
- **Type / role:** `n8n-nodes-base.if` — verifies that hashtag search returned results.
- **Condition:** `={{ $json.data && $json.data.length > 0 }}` is true
- **Outputs:**
  - **True** → **Get Hashtag Metrics**
  - **False** → **Rate Limit Wait**
- **Edge cases:** If `Get Hashtag Info` returns an error payload without `data`, condition is false and the tag is skipped.

#### Get Hashtag Metrics
- **Type / role:** `n8n-nodes-base.facebookGraphApi` — pulls engagement from `top_media`
- **Config choices:**
  - Node: `={{ $json.data[0].id }}` (uses first hashtag search result)
  - Edge: `top_media`
  - Query params:
    - `fields = like_count, comments_count`
    - `user_id = YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID`
- **Outputs:** To **Calculate Average Metrics**
- **Failure modes:**
  - If `data[0].id` missing → expression error
  - API permissions/rate limits
  - Empty top_media array

#### Calculate Average Metrics
- **Type / role:** `n8n-nodes-base.code` — compute averages from returned media.
- **Key logic:**
  - Reads: `$input.all()[0].json.data`
  - If empty: returns `{ average_likes: 0, average_comments: 0 }`
  - Else: mean of like_count/comments_count rounded
- **Output:** Object with averages.
- **Edge cases:** Node returns a single object; in n8n Code node this is typically expected as `{ json: {...} }`. If n8n does not auto-wrap this in your version, adjust to `return [{ json: {...} }]`. (Test once in your environment.)

#### Format New Hashtag Data
- **Type / role:** `n8n-nodes-base.set` — shapes data for Sheets append.
- **Key expressions:**
  - `tag = {{ $('Split In Batches').item.json.tag }}`
  - `average_likes = {{ $json.average_likes }}`
  - `average_comments = {{ $json.average_comments }}`
- **Output:** To **Save to Cache**
- **Edge cases:** If SplitInBatches item context changes, tag mapping may fail.

#### Save to Cache
- **Type / role:** `n8n-nodes-base.googleSheets` — appends metrics row for new tag.
- **Config choices:**
  - Operation: Append
  - Columns mapped: `tag`, `average_likes`, `average_comments`
  - Spreadsheet + sheet placeholders to fill
- **Outputs:** To **Rate Limit Wait**
- **Failure modes:** OAuth, permission denied, schema mismatch, quota.

#### Rate Limit Wait
- **Type / role:** `n8n-nodes-base.wait` — throttling (2 seconds).
- **Config:** `2 seconds`
- **Output:** Back to **Split In Batches** (continues loop)
- **Edge cases:** Adds latency; adjust based on Graph API limits and batch size.

---

### Block 5 — Final Aggregation & AI Selection
**Overview:** Merge cached + newly fetched metrics, rank candidates, ask an LLM to select 5 best, then format final hashtags string.

**Nodes involved:**
- Merge Cache & Fresh
- Aggregate & Rank Candidates
- AI Model Selector
- Select Top 5 Hashtags
- Format Output

#### Merge Cache & Fresh
- **Type / role:** `n8n-nodes-base.merge` — joins two inputs:
  - input 0: from **Split In Batches** (flow control / new items)
  - input 1: cached items from **Route Cached vs New**
- **Output:** To **Aggregate & Rank Candidates**
- **Edge cases:** Merge behavior depends on settings; ensure it actually passes both cached and new data onward as intended.

#### Aggregate & Rank Candidates
- **Type / role:** `n8n-nodes-base.code` — builds candidate list for final LLM selection.
- **Key logic:**
  - Collects cached items from `$input.all()` where `source === 'cache'`
  - Collects fresh items by directly referencing node: `$('Format New Hashtag Data').all()` (in try/catch)
  - Sorts all items by `average_likes` desc
  - Keeps top 15 candidates: `{ tag, likes, comments }`
  - Retrieves caption from **Set Dummy Caption** or **Workflow Trigger** (fallback)
- **Output:** `{ caption, candidates }`
- **Edge cases / failure modes:**
  - If `Format New Hashtag Data` didn’t run (all cached), freshItems empty (handled).
  - If neither caption source exists (miswired run), caption becomes empty → poorer LLM selection.
  - If cached items didn’t include `average_likes`, sorting degenerates.

#### AI Model Selector
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides model for final selection.
- **Config choices:**
  - Model: `gpt-5-mini`
- **Outputs:** (ai_languageModel) → **Select Top 5 Hashtags**
- **Failure modes:** model availability, credential/rate limits.

#### Select Top 5 Hashtags
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — selects final 5 based on relevance + likes.
- **Key expressions:**
  - Caption: `{{ $json.caption }}`
  - Candidates: `{{ JSON.stringify($json.candidates) }}`
- **Output format required:** JSON array `["tag1","tag2","tag3","tag4","tag5"]`
- **Failure modes:** Non-JSON output; model returning markdown fences.

#### Format Output
- **Type / role:** `n8n-nodes-base.set` — converts selected list into hashtag string.
- **Key expression:**
  - `hashtags = {{ JSON.parse($json.text.replace(/```json/g,'').replace(/```/g,'')).map(t => '#' + t.replace('#','')).join(' ') }}`
- **Output:** `hashtags` string like `#tag1 #tag2 ...`
- **Failure modes:** JSON parse error if LLM output deviates; partial sanitization is included (removes ```json fences).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | manualTrigger | Manual entry point | — | Set Dummy Caption | ## 🚀 Instagram Hashtag Generator… (full note) |
| Set Dummy Caption | set | Provide caption in manual mode | Manual Trigger | Generate Initial Hashtags; Fetch Cached Hashtags | ## 1. Input Processing… |
| Workflow Trigger | executeWorkflowTrigger | Sub-workflow entry point (caption input) | — | Generate Initial Hashtags | ## 1. Input Processing… |
| OpenAI Chat Model | lmChatOpenAi | LLM resource for initial hashtag generation | — | Generate Initial Hashtags (ai_languageModel) | ## 2. Idea Generation… |
| Generate Initial Hashtags | chainLlm | Produce 10 candidate hashtags | Set Dummy Caption / Workflow Trigger; OpenAI Chat Model | Parse & Clean Tags | ## 2. Idea Generation… |
| Parse & Clean Tags | code | Parse JSON list and normalize tags | Generate Initial Hashtags | Merge Input | ## 2. Idea Generation… |
| Fetch Cached Hashtags | googleSheets | Read cached metrics | Set Dummy Caption | Merge Input | ## 3. Caching Logic… |
| Merge Input | merge | Combine AI tags + sheet rows | Parse & Clean Tags; Fetch Cached Hashtags | Check Cache Status | ## 3. Caching Logic… |
| Check Cache Status | code | Mark tags as cached/new | Merge Input | Route Cached vs New | ## 3. Caching Logic… |
| Route Cached vs New | if | Branch cached vs new | Check Cache Status | Merge Cache & Fresh; Split In Batches | ## 3. Caching Logic… |
| Split In Batches | splitInBatches | Iterate over new tags | Route Cached vs New; Rate Limit Wait | Merge Cache & Fresh; Get Hashtag Info | ## 4. Instagram Data & Caching… |
| Get Hashtag Info | facebookGraphApi | Search hashtag ID | Split In Batches | Check if Tag Exists; Rate Limit Wait | ## 4. Instagram Data & Caching… |
| Check if Tag Exists | if | Skip if hashtag not found | Get Hashtag Info | Get Hashtag Metrics; Rate Limit Wait | ## 4. Instagram Data & Caching… |
| Get Hashtag Metrics | facebookGraphApi | Fetch top_media metrics | Check if Tag Exists | Calculate Average Metrics | ## 4. Instagram Data & Caching… |
| Calculate Average Metrics | code | Compute average likes/comments | Get Hashtag Metrics | Format New Hashtag Data | ## 4. Instagram Data & Caching… |
| Format New Hashtag Data | set | Shape data for Sheets append | Calculate Average Metrics | Save to Cache | ## 4. Instagram Data & Caching… |
| Save to Cache | googleSheets | Append new metrics to cache | Format New Hashtag Data | Rate Limit Wait | ## 4. Instagram Data & Caching… |
| Rate Limit Wait | wait | Throttle & loop | Save to Cache / Get Hashtag Info / Check if Tag Exists | Split In Batches | ## 4. Instagram Data & Caching… |
| Merge Cache & Fresh | merge | Merge cached + fresh streams | Split In Batches; Route Cached vs New | Aggregate & Rank Candidates | ## 5. Final Selection… |
| Aggregate & Rank Candidates | code | Rank and prepare candidates + caption | Merge Cache & Fresh | Select Top 5 Hashtags | ## 5. Final Selection… |
| AI Model Selector | lmChatOpenAi | LLM resource for final selection | — | Select Top 5 Hashtags (ai_languageModel) | ## 5. Final Selection… |
| Select Top 5 Hashtags | chainLlm | Choose best 5 hashtags | Aggregate & Rank Candidates; AI Model Selector | Format Output | ## 5. Final Selection… |
| Format Output | set | Output “#tag …” string | Select Top 5 Hashtags | — | ## 5. Final Selection… |
| Sticky Note1 | stickyNote | Documentation | — | — | ## 🚀 Instagram Hashtag Generator… (full note) |
| Sticky Note2 | stickyNote | Documentation | — | — | ## 1. Input Processing… |
| Sticky Note3 | stickyNote | Documentation | — | — | ## 2. Idea Generation… |
| Sticky Note4 | stickyNote | Documentation | — | — | ## 3. Caching Logic… |
| Sticky Note5 | stickyNote | Documentation | — | — | ## 4. Instagram Data & Caching… |
| Sticky Note6 | stickyNote | Documentation | — | — | ## 5. Final Selection… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   - Name it: *Generate Instagram hashtags using OpenAI and Graph API* (or your preferred name).
   - Ensure workflow setting `Execution order` is `v1` (as in the JSON).

2) **Add entry nodes (two modes)**
   1. Add **Manual Trigger**.
   2. Add **Execute Workflow Trigger** (“Workflow Trigger”):
      - Add workflow input parameter: `caption` (string).

3) **Manual caption setup**
   - Add **Set** node “Set Dummy Caption”:
     - Add field `caption` (string) with a sample caption (replace later).

4) **Initial LLM generation**
   1. Add **OpenAI Chat Model** (LangChain) “OpenAI Chat Model”:
      - Model: `gpt-4o-mini`
      - Credentials: OpenAI API key.
   2. Add **Chain LLM** node “Generate Initial Hashtags”:
      - Connect **OpenAI Chat Model** to its **AI Language Model** input.
      - Prompt text (define prompt) that:
        - requests 10 English hashtags
        - includes `{{ $json.caption }}`
        - requires JSON array output only
      - Enable structured output parsing if available (`hasOutputParser`).

5) **Parse tags**
   - Add **Code** node “Parse & Clean Tags” with logic:
     - Extract JSON array from the LLM output text
     - Parse it
     - Emit one item per tag: `{ tag: '...' }` without `#`

6) **Google Sheets cache read**
   1. Add **Google Sheets** node “Fetch Cached Hashtags”:
      - Set Spreadsheet ID: your sheet
      - Set Sheet name/tab: your cache tab
      - Turn on “Continue on fail” (equivalent to `onError: continueRegularOutput`) if desired.
   2. Prepare your Google Sheet with columns:
      - `tag` (string)
      - `average_likes` (number or string convertible)
      - `average_comments` (number or string convertible)

7) **Merge AI tags + cache rows**
   - Add **Merge** node “Merge Input”
   - Connect:
     - “Parse & Clean Tags” → Merge Input (input 0)
     - “Fetch Cached Hashtags” → Merge Input (input 1)

8) **Determine cached vs new**
   1. Add **Code** node “Check Cache Status”:
      - Build map from sheet rows keyed by normalized tag
      - For each AI tag emit `{ tag, source: 'cache', average_likes, average_comments }` or `{ tag, source: 'new' }`
   2. Add **IF** node “Route Cached vs New”:
      - Condition: `{{ $json.source }}` equals `cache`
      - True → cached path; False → new path

9) **Batch loop for new tags**
   1. Add **Split In Batches** node:
      - Batch size: 1 (recommended for rate limits)
   2. Add **Wait** node “Rate Limit Wait”:
      - 2 seconds
      - Connect Wait → Split In Batches to loop

10) **Instagram Graph API calls**
   1. Add **Facebook Graph API** node “Get Hashtag Info”:
      - Node/operation: `ig_hashtag_search`
      - Query:
        - `user_id = YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID`
        - `q = {{ $('Split In Batches').item.json.tag }}`
      - Graph API version: `v18.0`
      - Enable continue-on-error if you want to skip failures.
   2. Add **IF** node “Check if Tag Exists”:
      - Condition: `{{ $json.data && $json.data.length > 0 }}` is true
   3. Add **Facebook Graph API** node “Get Hashtag Metrics”:
      - Edge: `top_media`
      - Node id: `{{ $json.data[0].id }}`
      - Query:
        - `fields = like_count, comments_count`
        - `user_id = YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID`
   4. Add **Code** node “Calculate Average Metrics”:
      - Average likes/comments across returned media items (fallback to 0 if none)

11) **Write fresh metrics back to Sheets**
   1. Add **Set** node “Format New Hashtag Data”:
      - `tag = {{ $('Split In Batches').item.json.tag }}`
      - `average_likes = {{ $json.average_likes }}`
      - `average_comments = {{ $json.average_comments }}`
   2. Add **Google Sheets** node “Save to Cache”:
      - Operation: Append
      - Map columns: `tag`, `average_likes`, `average_comments`
      - Same Spreadsheet ID + Sheet tab as the fetch node

12) **Aggregate and rank candidates**
   1. Add **Merge** node “Merge Cache & Fresh”
      - Connect cached path (IF True) into one input
      - Connect SplitInBatches stream into the other input (as in the JSON)
   2. Add **Code** node “Aggregate & Rank Candidates”
      - Combine cached items (`source=cache`) with fresh items (from “Format New Hashtag Data”)
      - Sort by average likes desc
      - Select top 15
      - Retrieve caption from either Manual or Workflow Trigger path

13) **Final AI selection**
   1. Add **OpenAI Chat Model** node “AI Model Selector”:
      - Model: `gpt-5-mini`
   2. Add **Chain LLM** “Select Top 5 Hashtags”:
      - Connect “AI Model Selector” to its AI model input
      - Prompt includes:
        - Caption
        - Candidate list with likes/comments
        - Strict JSON output: `["tag1",...,"tag5"]`
   3. Add **Set** node “Format Output”:
      - Parse model JSON and join into `hashtags` string with leading `#`

14) **Wire connections (match logic)**
   - Manual Trigger → Set Dummy Caption
   - Workflow Trigger → Generate Initial Hashtags
   - Set Dummy Caption → Generate Initial Hashtags **and** Fetch Cached Hashtags
   - Generate Initial Hashtags → Parse & Clean Tags → Merge Input
   - Fetch Cached Hashtags → Merge Input
   - Merge Input → Check Cache Status → Route Cached vs New
   - IF False (new) → Split In Batches → Get Hashtag Info → Check if Tag Exists → (True) Get Hashtag Metrics → Calculate Average Metrics → Format New Hashtag Data → Save to Cache → Wait → Split In Batches
   - IF True (cached) → Merge Cache & Fresh
   - Split In Batches → Merge Cache & Fresh (so new items can be aggregated)
   - Merge Cache & Fresh → Aggregate & Rank Candidates → Select Top 5 Hashtags → Format Output

15) **Important fix for sub-workflow mode (recommended)**
   - Also connect **Workflow Trigger → Fetch Cached Hashtags** (or redesign to fetch cache regardless of entry mode).  
   Otherwise, sub-workflow runs may not load cached metrics.

**Credentials needed**
- OpenAI: API key (for both model nodes)
- Google Sheets OAuth2: access to the spreadsheet
- Facebook Graph API: token with IG Graph permissions and correct IG Business Account ID

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically generates 5 optimized hashtags… combines AI suggestions with real-time performance data… caches results in Google Sheets…” | Sticky Note: overview/purpose |
| Setup: connect OpenAI, Google Sheets, Facebook Graph API credentials; create sheet columns `tag`, `average_likes`, `average_comments`; update Spreadsheet ID; set IG Business Account ID in Graph nodes | Sticky Note: setup steps |
| Usage modes: Manual Mode (Manual Trigger + Set Dummy Caption) and Sub-workflow Mode (Execute Workflow passing `{ "caption": "..." }`) | Sticky Note: usage modes |
| How it works: Generate → Metrics (cache then Graph API) → Select | Sticky Note: process summary |