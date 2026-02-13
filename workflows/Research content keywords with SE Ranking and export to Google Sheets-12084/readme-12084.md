Research content keywords with SE Ranking and export to Google Sheets

https://n8nworkflows.xyz/workflows/research-content-keywords-with-se-ranking-and-export-to-google-sheets-12084


# Research content keywords with SE Ranking and export to Google Sheets

## 1. Workflow Overview

**Purpose:** This workflow performs keyword research in SE Ranking for a given seed topic, consolidates multiple keyword suggestion types, computes SEO/content-focused scores and classifications, and exports the final prioritized list to **Google Sheets**.

**Primary use cases:**
- Building content calendars from SEO opportunities
- Finding question/FAQ topics with good search demand and manageable difficulty
- Prioritizing keywords using a repeatable scoring model

### 1.1 Trigger & Parallel Data Collection
Manual trigger starts four SE Ranking keyword research calls in parallel (longtail, questions, similar, related).

### 1.2 Consolidation
All four keyword result sets are merged into one combined stream for unified processing.

### 1.3 Keyword Analysis, Scoring & Enrichment
A Code node deduplicates keywords and adds:
- opportunity score (0–100)
- intent classification
- content type suggestion
- trend analysis (if history is present)
- estimated monthly traffic & priority label

### 1.4 Export
The enriched keyword list is appended/updated into a Google Sheets document.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & SE Ranking Fetch (Parallel)
**Overview:** Starts the workflow manually and fetches four complementary keyword datasets from SE Ranking using the same seed keyword (“digital marketing”) with different endpoints/operations.

**Nodes involved:**
- When clicking 'Execute workflow'
- Get longtail keywords
- Get question keywords
- Get similar keywords
- Get related keywords

#### Node: When clicking 'Execute workflow'
- **Type / role:** Manual Trigger (entry point)
- **Configuration:** No parameters; execution begins when run manually in the editor.
- **Connections:**
  - **Outputs to:** Get longtail keywords, Get question keywords, Get similar keywords, Get related keywords (in parallel)
- **Edge cases / failures:**
  - None inherent; only runs when manually executed.

#### Node: Get longtail keywords
- **Type / role:** SE Ranking community node (`@seranking/n8n-nodes-seranking.seRanking`) — keyword research fetch
- **Configuration choices:**
  - **Resource:** Keyword Research
  - **Operation:** Get Longtail
  - **Seed keyword:** `digital marketing`
  - **Limit:** 50 results
- **Credentials:** SE Ranking API credential required
- **Connections:**
  - **Input:** Manual Trigger
  - **Output:** Merge all keyword types (input index 0)
- **Edge cases / failures:**
  - Invalid/expired SE Ranking API key
  - SE Ranking rate limits / quota exhaustion
  - API response schema changes (community node compatibility)
  - Empty results for the seed keyword
  - Network/timeouts

#### Node: Get question keywords
- **Type / role:** SE Ranking community node — fetch question-style keywords
- **Configuration choices:**
  - **Operation:** Get Questions
  - **Seed keyword:** `digital marketing`
  - **Filters:** limit 50, **volumeFrom 500**, **difficultyTo 50**
  - **historyTrend:** enabled (requests historical trend data when available)
- **Credentials:** SE Ranking API credential required
- **Connections:**
  - **Input:** Manual Trigger
  - **Output:** Merge all keyword types (input index 1)
- **Edge cases / failures:**
  - Same as above, plus:
  - Trend data may be absent or partial even when requested

#### Node: Get similar keywords
- **Type / role:** SE Ranking community node — fetch semantically similar keywords
- **Configuration choices:**
  - **Operation:** Get Similar
  - **Seed keyword:** `digital marketing`
  - **Filters:** limit 50, volumeFrom 500, difficultyTo 50, historyTrend true
- **Credentials:** SE Ranking API credential required
- **Connections:**
  - **Input:** Manual Trigger
  - **Output:** Merge all keyword types (input index 2)
- **Edge cases / failures:** Same as “Get question keywords”.

#### Node: Get related keywords
- **Type / role:** SE Ranking community node — fetch related keyword suggestions
- **Configuration choices:**
  - **Operation:** Get Related
  - **Seed keyword:** `digital marketing`
  - **Filters:** limit 50, volumeFrom 500, difficultyTo 50, historyTrend true
- **Credentials:** SE Ranking API credential required
- **Connections:**
  - **Input:** Manual Trigger
  - **Output:** Merge all keyword types (input index 3)
- **Edge cases / failures:** Same as “Get question keywords”.

---

### Block 2 — Merge Results
**Overview:** Combines the four SE Ranking responses into a single stream so the next step can deduplicate and score across all sources.

**Nodes involved:**
- Merge all keyword types

#### Node: Merge all keyword types
- **Type / role:** Merge node (multi-input aggregator)
- **Configuration choices:**
  - **Mode:** Multi-input merge with **4 inputs** (`numberInputs: 4`)
  - Purpose here is not record-level joining; it’s collecting the 4 datasets together for processing.
- **Connections:**
  - **Inputs:** longtail (0), questions (1), similar (2), related (3)
  - **Output:** Analyze & Score Keywords for Content
- **Type/version notes:** Merge node **typeVersion 3** (behavior can differ from older versions; this setup relies on the “number of inputs” feature).
- **Edge cases / failures:**
  - If any upstream SE Ranking node errors, the workflow stops unless error handling is added.
  - If an upstream node returns an unexpected structure, downstream Code logic may ignore it (it checks for `section.keywords` array).

---

### Block 3 — Analyze, Score, Classify, Deduplicate
**Overview:** Deduplicates keywords across all sources and enriches each keyword with scoring and content recommendations, then sorts by opportunity.

**Nodes involved:**
- Analyze & Score Keywords for Content

#### Node: Analyze & Score Keywords for Content
- **Type / role:** Code node (JavaScript) — transforms merged API responses into row-ready objects
- **Configuration choices (interpreted):**
  - Reads **all incoming items** via `$input.all()`. Each item is treated as a “section” corresponding to one SE Ranking operation (by **arrival/order**).
  - Expects each section to contain: `json.keywords` as an array.
  - **Deduplication:** uses a `Set()` keyed by `keyword.toLowerCase().trim()`
  - Supports two keyword formats:
    - **Object keyword** (preferred): `{ keyword, volume, cpc, difficulty, ... }`
    - **String keyword**: `"some keyword"` (gets default metrics = 0)
- **Key computed fields:**
  - `date`: ISO date string `YYYY-MM-DD`
  - **Opportunity score (0–100)**:
    - volumeScore up to 40: `min(volume/10000, 1) * 40`
    - difficultyScore up to 40: `(100-difficulty)/100*40`
    - cpcScore up to 20: `min(cpc/20, 1) * 20`
  - `priority`: HIGH (>=70), MEDIUM (>=50), else LOW; string keywords => UNKNOWN
  - `search_intent`: regex-based classification (transactional/commercial/informational/navigational), overridden by `kw.intents` if present
  - `content_type`: heuristic based on keyword patterns (how-to, explainer, FAQ, listicle, comparison, etc.)
  - Trend:
    - reads `kw.history_trend` if an object
    - compares last 3 values vs first 3 values to mark “trending up/down”
  - `estimated_monthly_traffic`: `round(volume * 0.15)` (assumes ~15% CTR as a simplified top-3/top-5 potential)
  - `serp_features`: joined if array
  - `section_source`: assigned by `sectionIndex` mapping:
    - 0 longtail, 1 questions, 2 similar, 3 related
- **Connections:**
  - **Input:** Merge all keyword types
  - **Output:** Export to Google Sheets
- **Type/version notes:** Code node **typeVersion 2**
- **Edge cases / failure types:**
  - **Ordering risk:** `section_source` assumes sectionIndex matches longtail/questions/similar/related. With merges, item order can change; labels may be wrong if upstream execution order changes.
  - If SE Ranking returns results under a different property than `keywords`, the node outputs nothing (silently skips).
  - If `kw.keyword` is missing for object entries, those entries are treated like “string keywords” path only if they are strings; otherwise may be skipped or partially processed.
  - Trend calc division by zero risk: if `older3` average is 0, `(recent3-older3)/older3` would be problematic; current code only checks length, not zero denominator.
  - Large datasets could increase execution time (still moderate at 4×50).

---

### Block 4 — Export to Google Sheets
**Overview:** Writes the enriched keyword rows into a Google Sheet using append-or-update behavior.

**Nodes involved:**
- Export to Google Sheets

#### Node: Export to Google Sheets
- **Type / role:** Google Sheets node — persist results
- **Configuration choices:**
  - **Operation:** Append or Update
  - **Document:** not specified in JSON (must be selected)
  - **Sheet name:** not specified in JSON (must be selected)
- **Credentials:** Google Sheets OAuth2 required
- **Connections:**
  - **Input:** Analyze & Score Keywords for Content
  - **Output:** None (end)
- **Type/version notes:** Google Sheets node **typeVersion 4.7**
- **Edge cases / failures:**
  - OAuth permission issues (missing scopes, revoked token)
  - Spreadsheet not shared with the OAuth account
  - Sheet/tab not found
  - Append/Update requires a configured matching key/column behavior in-node; if not configured, may append duplicates rather than update
  - Column mismatch if headers are absent or different than the fields being sent

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual entry point | — | Get longtail keywords; Get question keywords; Get similar keywords; Get related keywords | ## Research keywords for content ideas  \n\n## Who is this for\n- Content creators planning articles\n- SEO professionals finding opportunities\n- Marketing teams building content calendars\n\n## What you'll get\n- Longtail, question, similar, and related keywords\n- Opportunity scores based on volume, difficulty, and CPC\n- Search intent classification\n- Content type suggestions\n- Trend analysis with historical data\n- Traffic estimates and priority rankings\n- All results exported to Google Sheets\n\n## How it works\n1. Fetches 4 keyword types from SE Ranking\n2. Merges and deduplicates results\n3. Analyzes with scoring algorithm\n4. Detects search intent and suggests content types\n5. Exports to Google Sheets\n\n## Setup steps\n1. Install SE Ranking community node\n2. Add SE Ranking API credentials\n3. Update seed keyword in all 4 nodes\n4. Connect Google Sheets credentials\n5. Select your spreadsheet\n\n## Customization\nAdjust volume and difficulty filters in nodes or modify scoring formula in Code node. |
| Get longtail keywords | SE Ranking (community) | Fetch longtail keyword variations | When clicking 'Execute workflow' | Merge all keyword types | ### Longtail Keywords\nRetrieves up to 50 longtail variations. Update the keyword parameter with your target topic. |
| Get question keywords | SE Ranking (community) | Fetch question keywords with filters + trend | When clicking 'Execute workflow' | Merge all keyword types | ### Question Keywords\nFinds question-based keywords with min 500 volume and max 50 difficulty. Perfect for FAQ content. |
| Get similar keywords | SE Ranking (community) | Fetch similar keywords with filters + trend | When clicking 'Execute workflow' | Merge all keyword types | ### Similar Keywords\nGets semantically similar keywords with the same filters. Helps expand topic coverage. |
| Get related keywords | SE Ranking (community) | Fetch related keywords with filters + trend | When clicking 'Execute workflow' | Merge all keyword types | ### Related Keywords\nFinds related suggestions based on your seed term. Uncovers adjacent topics. |
| Merge all keyword types | Merge | Aggregate 4 keyword datasets | Get longtail keywords; Get question keywords; Get similar keywords; Get related keywords | Analyze & Score Keywords for Content | ### Merge Results\nCombines all 4 keyword sources into a single dataset. |
| Analyze & Score Keywords for Content | Code | Deduplicate + scoring + intent + content recommendations | Merge all keyword types | Export to Google Sheets | ### Analysis\nScores keywords by volume, difficulty, and CPC. Detects search intent, suggests content types, and assigns priority rankings. |
| Export to Google Sheets | Google Sheets | Persist analyzed keywords into a spreadsheet | Analyze & Score Keywords for Content | — | ### Export\nAppends analyzed keywords to Google Sheets with 17 columns including metrics and recommendations. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Research keywords for content ideas with SE Ranking* (or your preferred title).

2. **Add trigger**
   - Add node: **Manual Trigger**
   - Name: *When clicking 'Execute workflow'*

3. **Install and enable SE Ranking community node**
   - In n8n: go to **Settings → Community nodes**
   - Install package: `@seranking/n8n-nodes-seranking`
   - Restart n8n if required.

4. **Create SE Ranking credentials**
   - Create credential: **SE Ranking API**
   - Paste your SE Ranking API key/token as required by the credential UI.
   - Save as e.g. “SE Ranking API”.

5. **Add 4 SE Ranking nodes (all connected from the Manual Trigger in parallel)**

   5.1 **Get longtail keywords**
   - Node type: **SE Ranking**
   - Resource: **Keyword Research**
   - Operation: **Get Longtail**
   - Keyword: `digital marketing` (replace with your seed topic)
   - Additional fields: `limit = 50`
   - Select the **SE Ranking API** credentials

   5.2 **Get question keywords**
   - Operation: **Get Questions**
   - Keyword: same seed
   - Additional fields:
     - `limit = 50`
     - `volumeFrom = 500`
     - `difficultyTo = 50`
     - `historyTrend = true`
   - Credentials: SE Ranking API

   5.3 **Get similar keywords**
   - Operation: **Get Similar**
   - Keyword: same seed
   - Additional fields: limit 50, volumeFrom 500, difficultyTo 50, historyTrend true
   - Credentials: SE Ranking API

   5.4 **Get related keywords**
   - Operation: **Get Related**
   - Keyword: same seed
   - Additional fields: limit 50, volumeFrom 500, difficultyTo 50, historyTrend true
   - Credentials: SE Ranking API

6. **Add Merge node**
   - Node type: **Merge**
   - Name: *Merge all keyword types*
   - Set **Number of Inputs = 4**
   - Connect:
     - longtail → merge input 0
     - questions → merge input 1
     - similar → merge input 2
     - related → merge input 3

7. **Add Code node**
   - Node type: **Code**
   - Name: *Analyze & Score Keywords for Content*
   - Paste the JavaScript logic (the workflow’s code) that:
     - reads `$input.all()`
     - extracts `json.keywords`
     - deduplicates by normalized keyword text
     - computes scoring/intent/content type/trend/traffic fields
     - sorts by `opportunity_score` descending
     - outputs one item per keyword row
   - Connect: Merge → Code

8. **Create Google Sheets OAuth2 credentials**
   - Credential type: **Google Sheets OAuth2 API**
   - Complete OAuth consent and authorization.
   - Ensure the Google account has access to the target spreadsheet.

9. **Add Google Sheets node**
   - Node type: **Google Sheets**
   - Name: *Export to Google Sheets*
   - Operation: **Append or Update**
   - Select:
     - **Document** (Spreadsheet)
     - **Sheet** (Tab name)
   - Map columns/fields:
     - Ensure the sheet has headers matching the produced JSON fields (e.g. `date`, `keyword`, `volume`, `cpc`, `difficulty`, `competition`, `relevance`, `opportunity_score`, `priority`, `search_intent`, `content_type`, `trend_status`, `trend_direction_pct`, `trend_avg_volume`, `estimated_monthly_traffic`, `serp_features`, `section_source`)
   - Connect: Code → Google Sheets

10. **Run and validate**
   - Click **Execute workflow**
   - Confirm SE Ranking nodes return data
   - Confirm the sheet receives rows and columns align

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install SE Ranking community node package `@seranking/n8n-nodes-seranking` | Required for the SE Ranking nodes to appear in n8n |
| Update the seed keyword in all 4 SE Ranking nodes (currently “digital marketing”) | Ensures all sources query the same topic |
| Filters used: volumeFrom=500 and difficultyTo=50 (questions/similar/related) | Biases results toward higher demand and lower difficulty |
| Scoring model weights: Volume 40%, Difficulty 40%, CPC 20% | Implemented in the Code node; adjust to your strategy |
| Export uses “Append or Update”; ensure correct key/headers to avoid duplicates | Google Sheets node configuration may require a chosen matching column/key |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.