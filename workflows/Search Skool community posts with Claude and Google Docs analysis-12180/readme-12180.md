Search Skool community posts with Claude and Google Docs analysis

https://n8nworkflows.xyz/workflows/search-skool-community-posts-with-claude-and-google-docs-analysis-12180


# Search Skool community posts with Claude and Google Docs analysis

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Intelligent Skool Community Search with AI Analysis (provided title: “Search Skool community posts with Claude and Google Docs analysis”)  
**Purpose:** Turn a Skool community’s historical posts into a queryable knowledge base. Users submit a question via an n8n form. The workflow validates authentication via a Skool session cookie, auto-detects Skool’s current Next.js BuildId, uses Anthropic Claude to extract search keywords and then to synthesize insights from the most engaged posts. It saves a full report into Google Docs and returns a polished HTML results page.

**Primary use cases**
- Fast retrieval of “community wisdom” from large Skool communities
- Summarizing patterns/sentiment from discussions at scale
- Creating shareable reports (Google Docs) for later reference

### Logical blocks
1.1 **Input Reception & Configuration**  
Receives form inputs (question, search depth, optional Drive folder), applies defaults and limits.

1.2 **Skool Session Validation & BuildId Extraction**  
Fetches the community homepage using the cookie, detects expired sessions, and extracts the dynamic Next.js BuildId needed for Skool’s internal JSON endpoints.

1.3 **Keyword Extraction (Claude Haiku)**  
Uses Claude to generate 1–2 concise search keywords for Skool’s search endpoint.

1.4 **Search Retrieval, Aggregation & Ranking**  
Builds search URLs for multiple pages, fetches them, deduplicates posts, truncates content to protect tokens, filters by age, and sorts by engagement.

1.5 **No-Results Handling**  
If no posts are found, returns a helpful HTML response suggesting ways to improve the query.

1.6 **Deep Analysis (Claude Sonnet)**  
Builds a prompt with top posts + comments and requests a comprehensive answer and ranked references.

1.7 **Report Generation (Google Docs) & HTML Response**  
Creates a Google Doc, inserts the full report, then returns an HTML page with links to the document and previews.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Configuration
**Overview:** Collects the user query and parameters, centralizes secrets/settings, and enforces safe defaults and limits.

**Nodes involved:**  
- Form Trigger  
- Config  

#### Node: Form Trigger
- **Type / role:** `Form Trigger` (webhook-backed form entry point)
- **Configuration (interpreted):**
  - Path: `skool-search` (webhookId also `skool-search`)
  - Fields:
    - `question` (required)
    - `folder_id` (optional Google Drive folder ID override)
    - `search_depth` (optional; intended 1–10, default 5)
  - Response mode: **Response Node** (execution responds later via Respond to Webhook nodes)
- **Inputs/Outputs:** Entry point → outputs to **Config**
- **Edge cases / failures:**
  - Missing `question` (form enforces required)
  - Users entering non-numeric depth; handled in Config

#### Node: Config
- **Type / role:** `Code` node; central configuration and input normalization
- **Configuration choices:**
  - Defines a `CONFIG` object containing:
    - Skool: `COMMUNITY` slug, `COOKIE`, `FALLBACK_BUILD_ID`
    - Anthropic: `ANTHROPIC_API_KEY`, models + max token settings
    - Google Drive: `DEFAULT_FOLDER_ID`
    - Safety limits: content truncation, max posts/comments, max post age
  - Parses form inputs from `$json.question` or `$json.body.question` (supports both direct and webhook body)
  - Normalizes:
    - `searchDepth` clamped to `[MIN_SEARCH_DEPTH..MAX_SEARCH_DEPTH]`
    - `folderId` defaulted to `DEFAULT_FOLDER_ID` if empty
- **Key variables produced:**
  - `config`, `question`, `folderId`, `searchDepth`, `timestamp`
- **Inputs/Outputs:** Form Trigger → Config → Fetch Skool Homepage
- **Edge cases / failures:**
  - Missing or invalid `DEFAULT_FOLDER_ID` will break later Google Docs creation
  - Hardcoding `ANTHROPIC_API_KEY` is functional but risky; env var is recommended (commented option exists)

---

### 2.2 Skool Session Validation & BuildId Extraction
**Overview:** Loads Skool community homepage with session cookie, checks if authentication is still valid, and extracts the current Next.js `buildId` required to call Skool’s `_next/data/.../search.json` endpoint.

**Nodes involved:**  
- Fetch Skool Homepage  
- Extract BuildId  
- Cookie Valid?  
- Cookie Error Response  
- Respond with Error  

#### Node: Fetch Skool Homepage
- **Type / role:** `HTTP Request` to Skool web page
- **Configuration choices:**
  - GET `https://www.skool.com/{{COMMUNITY}}`
  - Response format: **text** (HTML)
  - Headers:
    - `Cookie: {{$json.config.COOKIE}}`
    - User-Agent set to desktop Chrome-like UA
- **Inputs/Outputs:** Config → Fetch Skool Homepage → Extract BuildId
- **Edge cases / failures:**
  - 403/401 if cookie invalid or bot protection changes
  - Timeout/network errors
  - Skool HTML structure changes can affect later extraction logic

#### Node: Extract BuildId
- **Type / role:** `Code` node; validates cookie and extracts BuildId
- **Configuration choices / logic:**
  - Reads HTML from `$json.data` (n8n HTTP node typically stores response body under `data`)
  - Cookie expiration detection heuristics:
    - looks for `"isAuthenticated":false`, “Sign in…”, “Log in…”, or short login pages
  - If expired: returns `{ error: true, errorType: "COOKIE_EXPIRED", ... }`
  - BuildId extraction attempts (in order):
    1. JSON-like `"buildId":"..."` regex
    2. `_next/data/{buildId}/` regex
    3. `_next/static/{buildId}/_buildManifest` regex
    4. fallback to `config.FALLBACK_BUILD_ID`
- **Key variables:**
  - Output: `error`, `buildId`, `buildIdSource`, plus propagated `question`, `folderId`, `searchDepth`, `timestamp`, `config`
- **Inputs/Outputs:** Fetch Skool Homepage → Extract BuildId → Cookie Valid?
- **Edge cases / failures:**
  - If Skool changes Next.js delivery patterns, BuildId regex may fail (fallback used, but may be stale)
  - If HTTP node response field changes from `data` (instance settings), extraction could break

#### Node: Cookie Valid?
- **Type / role:** `IF` branching
- **Condition:**
  - Checks `{{$json.error}} equals true`
  - **True branch = cookie expired path**
  - **False branch = continue processing**
- **Inputs/Outputs:**
  - Extract BuildId → Cookie Valid?
  - True → Cookie Error Response
  - False → Build Keyword Request
- **Edge cases / failures:**
  - If Extract BuildId doesn’t set `error`, strict validation might behave unexpectedly; current code always sets it.

#### Node: Cookie Error Response
- **Type / role:** `Code` node; generates friendly HTML error page
- **Configuration choices:**
  - Produces `htmlReport` containing instructions to refresh cookie
  - Returns `errorType` and `errorMessage` for debugging
- **Inputs/Outputs:** Cookie Valid? (true) → Cookie Error Response → Respond with Error
- **Edge cases / failures:**
  - Includes an icon character in HTML; otherwise safe

#### Node: Respond with Error
- **Type / role:** `Respond to Webhook` (ends request)
- **Configuration choices:**
  - Content-Type header: `text/html`
  - Respond with: text body = `{{$json.htmlReport}}`
- **Inputs/Outputs:** Cookie Error Response → Respond with Error
- **Edge cases / failures:**
  - Only valid because Form Trigger is set to “response node” mode

---

### 2.3 Keyword Extraction (Claude Haiku)
**Overview:** Calls Anthropic Messages API with a short prompt to extract 1–2 keywords for Skool search.

**Nodes involved:**  
- Build Keyword Request  
- Claude - Extract Keywords  

#### Node: Build Keyword Request
- **Type / role:** `Code` node; assembles Anthropic API request
- **Configuration choices:**
  - Model: `config.KEYWORD_MODEL` (default `claude-3-haiku-20240307`)
  - `max_tokens`: `config.KEYWORD_MAX_TOKENS` (50)
  - Prompt instructs: “Return ONLY keywords separated by spaces”
  - Outputs `requestBody` + `apiKey` + spreads previous data
- **Inputs/Outputs:** Cookie Valid? (false) → Build Keyword Request → Claude - Extract Keywords
- **Edge cases / failures:**
  - If Claude returns extra text, later keyword parsing uses `content[0].text` directly; could pollute query

#### Node: Claude - Extract Keywords
- **Type / role:** `HTTP Request` to Anthropic Messages API
- **Configuration choices:**
  - POST `https://api.anthropic.com/v1/messages`
  - Timeout: 30s
  - Headers:
    - `x-api-key`
    - `anthropic-version: 2023-06-01`
    - `Content-Type: application/json`
  - Body: JSON stringified requestBody
- **Inputs/Outputs:** Build Keyword Request → Claude - Extract Keywords → Build Page URLs
- **Edge cases / failures:**
  - 401 if key invalid; 429 rate limits; network timeouts
  - Response schema changes could break keyword extraction code

---

### 2.4 Search Retrieval, Aggregation & Ranking
**Overview:** Builds multiple `_next/data/.../search.json` URLs across pages, fetches them, aggregates posts + comments, deduplicates, filters by age, truncates content, and sorts by engagement.

**Nodes involved:**  
- Build Page URLs  
- Fetch All Pages  
- Aggregate All Posts  
- Has Results?  

#### Node: Build Page URLs
- **Type / role:** `Code` node; generates one item per page to fetch
- **Configuration choices / logic:**
  - Reads keyword text: `keywordResponse.content?.[0]?.text` else fallback `"automation agency"`
  - Constructs base URL:
    - `https://www.skool.com/_next/data/${buildId}/${community}/-/search.json`
  - For page 1..`searchDepth`, creates items with:
    - `url` including `q`, `group`, `page`
    - `cookie`, `keywords`, `buildId`, `folderId`, etc.
- **Inputs/Outputs:** Claude - Extract Keywords → Build Page URLs → Fetch All Pages
- **Edge cases / failures:**
  - If keywords include newlines/punctuation, encoding protects URL but may reduce result quality
  - If buildId wrong, Skool may return error JSON/HTML

#### Node: Fetch All Pages
- **Type / role:** `HTTP Request` (executed once per page item)
- **Configuration choices:**
  - GET `{{$json.url}}`
  - Timeout: 15s
  - Headers: Cookie + User-Agent
- **Inputs/Outputs:** Build Page URLs → Fetch All Pages → Aggregate All Posts
- **Edge cases / failures:**
  - Some pages may fail; aggregator counts `errorPages`
  - If Skool returns non-JSON or unexpected structure, aggregator treats as error (`!json.pageProps`)

#### Node: Aggregate All Posts
- **Type / role:** `Code` node; merges results and applies limits
- **Key logic:**
  - Iterates all fetched pages
  - Skips items with `json.error` or missing `pageProps`
  - Extracts from `pageProps.postTrees`:
    - Post metadata, author, comments (from `children`)
  - Token protection:
    - post content truncated to `MAX_POST_CONTENT_LENGTH` (default 2000)
    - comment content truncated to `MAX_COMMENT_LENGTH` (default 500)
    - comments limited to `MAX_COMMENTS_PER_POST` (default 5)
  - Optional age filter:
    - excludes posts older than `MAX_POST_AGE_DAYS` (default 730)
  - Deduplication by post `id`
  - Sort by engagement: `(upvotes + commentCount)` descending
- **Outputs:**
  - `posts` array (unique, sorted)
  - `totalResults`, `pagesSearched` (validPages), `errorPages`
  - plus propagated config/question/folderId/buildId/keywords/timestamp
- **Inputs/Outputs:** Fetch All Pages → Aggregate All Posts → Has Results?
- **Edge cases / failures:**
  - If `post.id` missing, dedupe may keep duplicates
  - If `post.name` missing, constructed URL may be incomplete
  - If Skool schema changes (`postTrees` shape), extraction breaks silently (empty results)

#### Node: Has Results?
- **Type / role:** `IF` branching
- **Condition:**
  - Checks `{{$json.posts.length}} equals 0`
  - **True branch = no results**
  - **False branch = proceed to analysis**
- **Inputs/Outputs:**
  - Aggregate All Posts → Has Results?
  - True → No Results Response
  - False → Build Analysis Request
- **Edge cases / failures:**
  - If `posts` undefined (shouldn’t happen), expression error could occur; current aggregator always sets it.

---

### 2.5 No-Results Handling
**Overview:** Returns a user-friendly HTML page when the search yields no posts.

**Nodes involved:**  
- No Results Response  
- Respond No Results  

#### Node: No Results Response
- **Type / role:** `Code` node; builds HTML response
- **Configuration choices:**
  - Displays the question, keywords used, pages checked, error count
  - Offers suggestions (broader keywords, spelling, general question)
- **Inputs/Outputs:** Has Results? (true) → No Results Response → Respond No Results
- **Edge cases / failures:** Minimal; HTML generation only.

#### Node: Respond No Results
- **Type / role:** `Respond to Webhook`
- **Configuration choices:**
  - Content-Type: `text/html`
  - Respond body: `{{$json.htmlReport}}`
- **Inputs/Outputs:** No Results Response → Respond No Results

---

### 2.6 Deep Analysis (Claude Sonnet)
**Overview:** Builds a rich prompt using top engaged posts and requests a structured analysis from Claude Sonnet.

**Nodes involved:**  
- Build Analysis Request  
- Claude - Analyze  
- Format Response  

#### Node: Build Analysis Request
- **Type / role:** `Code` node; constructs analysis prompt and request body
- **Configuration choices:**
  - Takes top posts: `posts.slice(0, config.MAX_POSTS_FOR_ANALYSIS)` (default 20)
  - Prompt includes for each post:
    - title, author, engagement, URL
    - truncated post content (from aggregator)
    - top 3 comments (from already-trimmed comments list)
  - Asks for:
    1. Direct answer
    2. Patterns across posts
    3. Rank most useful posts (by title)
    4. Actionable advice
  - Model: `config.ANALYSIS_MODEL` (set to `claude-sonnet-4-20250514`)
  - Max tokens: `config.ANALYSIS_MAX_TOKENS` (2500)
- **Inputs/Outputs:** Has Results? (false) → Build Analysis Request → Claude - Analyze
- **Edge cases / failures:**
  - Prompt can still be large if many posts have long content (mitigated by truncation settings)
  - If model name is not available in a given Anthropic account/region, API returns error

#### Node: Claude - Analyze
- **Type / role:** `HTTP Request` to Anthropic Messages API
- **Configuration choices:**
  - Timeout: 60s
  - Same headers as keyword call
- **Inputs/Outputs:** Build Analysis Request → Claude - Analyze → Format Response
- **Edge cases / failures:**
  - 429 / timeouts; consider retry strategy if needed

#### Node: Format Response
- **Type / role:** `Code` node; consolidates Claude output and restores full post content for reporting
- **Configuration choices:**
  - Extracts analysis text from `claudeResponse.content[0].text`, else fallback message
  - Rebuilds `posts` list including `fullContent` when available (from aggregator)
  - Computes counts:
    - `postsAnalyzed: Math.min(prevData.posts.length, 20)`
- **Inputs/Outputs:** Claude - Analyze → Format Response → Prepare Document Content
- **Edge cases / failures:**
  - If Anthropic response format differs (e.g., content array missing), analysis becomes “Analysis not available”

---

### 2.7 Report Generation (Google Docs) & HTML Response
**Overview:** Produces a full text report, saves it as a Google Doc in Drive, and returns a styled HTML page with links and previews.

**Nodes involved:**  
- Prepare Document Content  
- Create Google Doc  
- Add Content to Doc  
- Generate Final HTML  
- Respond with HTML  

#### Node: Prepare Document Content
- **Type / role:** `Code` node; creates long-form report text + HTML snippets
- **Configuration choices / logic:**
  - Builds:
    - `docTitle` = `Skool Research - {question first 40 chars} - {YYYY-MM-DD}`
    - `docContent` = plain text report including:
      - metadata (question, timestamp, totals, buildId, keywords)
      - AI analysis
      - all posts full content + comments
      - quick links summary
  - Creates preview HTML:
    - `analysisHtml` via a simple `markdownToHtml()` converter (basic headings/bold/italics/lists)
    - `postsHtml` cards with a 500-char preview + up to 3 comments (200 chars)
- **Inputs/Outputs:** Format Response → Prepare Document Content → Create Google Doc
- **Edge cases / failures:**
  - The markdown conversion is simplistic (e.g., list conversion may produce `<li>` without surrounding `<ul>`)
  - Post content may include characters that should be HTML-escaped; current code injects raw text into HTML (risk of broken HTML if content contains `<`/`&`)

#### Node: Create Google Doc
- **Type / role:** `Google Docs` node; creates a new document
- **Configuration choices:**
  - Title from `{{$json.docTitle}}`
  - Folder ID from `{{$json.targetFolderId}}`
- **Credentials required:** Google Docs OAuth2 in n8n
- **Inputs/Outputs:** Prepare Document Content → Create Google Doc → Add Content to Doc
- **Edge cases / failures:**
  - Invalid folder ID → 404/permission error
  - OAuth token expired/revoked

#### Node: Add Content to Doc
- **Type / role:** `Google Docs` node; updates document by inserting text
- **Configuration choices:**
  - Operation: update / insert
  - Document URL uses created doc `id` (`{{$json.id}}`)
  - Insert text is taken from Prepare Document Content: `{{ $('Prepare Document Content').first().json.docContent }}`
- **Inputs/Outputs:** Create Google Doc → Add Content to Doc → Generate Final HTML
- **Edge cases / failures:**
  - Large content may hit Google Docs API limits; consider splitting inserts if needed

#### Node: Generate Final HTML
- **Type / role:** `Code` node; creates final HTML response page
- **Configuration choices:**
  - Builds `docUrl` from created `docData.id`
  - Builds `folderUrl` from `targetFolderId` (or My Drive fallback)
  - HTML includes:
    - Stats badges
    - Buttons to open Doc and Drive folder
    - Preview of AI analysis + “copy to clipboard”
    - Posts preview cards
- **Inputs/Outputs:** Add Content to Doc → Generate Final HTML → Respond with HTML
- **Edge cases / failures:**
  - Same HTML injection risk as above (analysisHtml/postsHtml not escaped)
  - Uses emoji-like glyphs in HTML; mostly safe but may not render uniformly

#### Node: Respond with HTML
- **Type / role:** `Respond to Webhook` final response
- **Configuration choices:**
  - Content-Type: `text/html`
  - Respond body: `{{$json.htmlReport}}`
- **Inputs/Outputs:** Generate Final HTML → Respond with HTML

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | Sticky Note | Documentation block | — | — | ## Skool Community Search with AI Analysis… (overview, setup steps, pro tip) |
| Sticky Note - Input | Sticky Note | Documentation block | — | — | ## Input and Validate Data… |
| Sticky Note - Errors | Sticky Note | Documentation block | — | — | ## Cookie Expired… |
| Sticky Note - Keywords | Sticky Note | Documentation block | — | — | ## Extract Keywords… |
| Sticky Note - Search | Sticky Note | Documentation block | — | — | ## Search & Aggregate Data… |
| Sticky Note - No Results | Sticky Note | Documentation block | — | — | ## No Output… |
| Sticky Note - Analysis | Sticky Note | Documentation block | — | — | ## Perform AI Analysis… |
| Sticky Note - Output | Sticky Note | Documentation block | — | — | ## Generate Output… |
| Form Trigger | Form Trigger | Collect question/depth/folder and start execution | — | Config | ## Input and Validate Data… |
| Config | Code | Central config + input validation/defaulting | Form Trigger | Fetch Skool Homepage | ## Input and Validate Data… |
| Fetch Skool Homepage | HTTP Request | Load Skool HTML with cookie for auth + BuildId discovery | Config | Extract BuildId | ## Input and Validate Data… |
| Extract BuildId | Code | Detect cookie expiry + extract Next.js BuildId | Fetch Skool Homepage | Cookie Valid? | ## Input and Validate Data… |
| Cookie Valid? | IF | Branch on expired cookie | Extract BuildId | Cookie Error Response; Build Keyword Request | ## Cookie Expired… |
| Cookie Error Response | Code | Generate HTML error page with refresh steps | Cookie Valid? | Respond with Error | ## Cookie Expired… |
| Respond with Error | Respond to Webhook | Return error HTML to form request | Cookie Error Response | — | ## Cookie Expired… |
| Build Keyword Request | Code | Build Anthropic request for keyword extraction | Cookie Valid? | Claude - Extract Keywords | ## Extract Keywords… |
| Claude - Extract Keywords | HTTP Request | Call Anthropic to extract keywords | Build Keyword Request | Build Page URLs | ## Extract Keywords… |
| Build Page URLs | Code | Create Skool search.json URLs for N pages | Claude - Extract Keywords | Fetch All Pages | ## Search & Aggregate Data… |
| Fetch All Pages | HTTP Request | Fetch each search result page JSON | Build Page URLs | Aggregate All Posts | ## Search & Aggregate Data… |
| Aggregate All Posts | Code | Merge, truncate, filter, dedupe, sort posts | Fetch All Pages | Has Results? | ## Search & Aggregate Data… |
| Has Results? | IF | Branch on empty results | Aggregate All Posts | No Results Response; Build Analysis Request | ## No Output… |
| No Results Response | Code | Generate HTML for empty search result | Has Results? | Respond No Results | ## No Output… |
| Respond No Results | Respond to Webhook | Return no-results HTML | No Results Response | — | ## No Output… |
| Build Analysis Request | Code | Build Anthropic request for deep analysis | Has Results? | Claude - Analyze | ## Perform AI Analysis… |
| Claude - Analyze | HTTP Request | Call Anthropic for synthesis/analysis | Build Analysis Request | Format Response | ## Perform AI Analysis… |
| Format Response | Code | Normalize AI output + prepare posts for reporting | Claude - Analyze | Prepare Document Content | ## Perform AI Analysis… |
| Prepare Document Content | Code | Build Google Doc text + HTML preview fragments | Format Response | Create Google Doc | ## Generate Output… |
| Create Google Doc | Google Docs | Create report document in Drive folder | Prepare Document Content | Add Content to Doc | ## Generate Output… |
| Add Content to Doc | Google Docs | Insert report text into created doc | Create Google Doc | Generate Final HTML | ## Generate Output… |
| Generate Final HTML | Code | Build final HTML results page with doc links | Add Content to Doc | Respond with HTML | ## Generate Output… |
| Respond with HTML | Respond to Webhook | Return final HTML page | Generate Final HTML | — | ## Generate Output… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it e.g. **“Intelligent Skool Community Search with AI Analysis”**
- Set workflow setting **Execution Order** to `v1` (matches JSON)

2) **Add the entry node: Form Trigger**
- Node: **Form Trigger**
- Path: `skool-search`
- Form title: “Skool Community Deep Search”
- Fields:
  - `question` (required)
  - `folder_id` (optional)
  - `search_depth` (optional; user hint: 1–10, default 5)
- Response Mode: **Using “Respond to Webhook” node** (response node mode)

3) **Add Config (Code)**
- Node: **Code** → “Config”
- Paste logic that:
  - Defines `CONFIG` with:
    - `COMMUNITY` (your Skool slug)
    - `COOKIE` (copy from browser DevTools for `www.skool.com`)
    - `FALLBACK_BUILD_ID` (any known value; used only if extraction fails)
    - `ANTHROPIC_API_KEY` (prefer `$env.ANTHROPIC_API_KEY` if possible)
    - Models:
      - keyword: `claude-3-haiku-20240307`, max tokens 50
      - analysis: `claude-sonnet-4-20250514`, max tokens 2500
    - Drive: `DEFAULT_FOLDER_ID`
    - Limits: truncation lengths, max posts/comments, max age days (e.g., 730)
  - Reads inputs from `$json.question` / `$json.body.question`, etc.
  - Clamps `searchDepth` to 1–10 and defaults it to 5
  - Defaults `folderId` to `DEFAULT_FOLDER_ID` if blank
  - Outputs: `{ config, question, folderId, searchDepth, timestamp }`
- Connect: **Form Trigger → Config**

4) **Fetch Skool homepage (HTTP Request)**
- Node: **HTTP Request** → “Fetch Skool Homepage”
- Method: GET
- URL: `https://www.skool.com/{{ $json.config.COMMUNITY }}`
- Response: **Text**
- Headers:
  - `Cookie: {{ $json.config.COOKIE }}`
  - `User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36`
- Connect: **Config → Fetch Skool Homepage**

5) **Extract BuildId + validate cookie (Code)**
- Node: **Code** → “Extract BuildId”
- Implement:
  - Detect cookie expiry by searching HTML for login markers
  - If expired: output `error=true` and error metadata
  - Else: extract BuildId via regex (3 patterns) with fallback
  - Output: `error=false`, `buildId`, `question`, `folderId`, `searchDepth`, `timestamp`, `config`
- Connect: **Fetch Skool Homepage → Extract BuildId**

6) **Branch on cookie validity (IF)**
- Node: **IF** → “Cookie Valid?”
- Condition: Boolean equals
  - Left: `{{ $json.error }}`
  - Right: `true`
- True output = expired path
- False output = continue path
- Connect: **Extract BuildId → Cookie Valid?**

7) **Expired cookie response**
- Node: **Code** → “Cookie Error Response”
  - Build an HTML page explaining how to refresh cookie
  - Output: `htmlReport`
- Node: **Respond to Webhook** → “Respond with Error”
  - Respond with text body: `{{ $json.htmlReport }}`
  - Add header `Content-Type: text/html`
- Connect: **Cookie Valid? (true) → Cookie Error Response → Respond with Error**

8) **Build keyword request (Code)**
- Node: **Code** → “Build Keyword Request”
- Create request body for Anthropic:
  - model = `config.KEYWORD_MODEL`
  - max_tokens = `config.KEYWORD_MAX_TOKENS`
  - message: “Extract 1-2 main search keywords only…”
- Output: `requestBody`, `apiKey`, and carry forward prior fields
- Connect: **Cookie Valid? (false) → Build Keyword Request**

9) **Call Anthropic to extract keywords (HTTP Request)**
- Node: **HTTP Request** → “Claude - Extract Keywords”
- Method: POST
- URL: `https://api.anthropic.com/v1/messages`
- Timeout: 30000
- Headers:
  - `x-api-key: {{ $json.apiKey }}`
  - `anthropic-version: 2023-06-01`
  - `Content-Type: application/json`
- Body: JSON, set to `{{ JSON.stringify($json.requestBody) }}`
- Connect: **Build Keyword Request → Claude - Extract Keywords**

10) **Build Skool search URLs (Code)**
- Node: **Code** → “Build Page URLs”
- Parse `keywords = content[0].text` from the Anthropic response
- Construct base:
  - `https://www.skool.com/_next/data/${buildId}/${community}/-/search.json`
- Emit one item per page from 1..`searchDepth` with:
  - `url` including query (`q`), `group`, and `page`
  - plus cookie and context fields
- Connect: **Claude - Extract Keywords → Build Page URLs**

11) **Fetch all pages (HTTP Request)**
- Node: **HTTP Request** → “Fetch All Pages”
- URL: `{{ $json.url }}`
- Timeout: 15000
- Headers:
  - `Cookie: {{ $json.cookie }}`
  - `User-Agent: ...`
- Connect: **Build Page URLs → Fetch All Pages**

12) **Aggregate posts (Code)**
- Node: **Code** → “Aggregate All Posts”
- Logic:
  - Loop over all page results
  - Extract `pageProps.postTrees`
  - Build post objects with author, title, content, url, comments
  - Enforce truncation + max comments
  - Optional age filter via `MAX_POST_AGE_DAYS`
  - Deduplicate by post id
  - Sort by engagement (upvotes + commentCount)
- Connect: **Fetch All Pages → Aggregate All Posts**

13) **Branch on results (IF)**
- Node: **IF** → “Has Results?”
- Condition: Number equals
  - Left: `{{ $json.posts.length }}`
  - Right: `0`
- True = no results
- False = analysis path
- Connect: **Aggregate All Posts → Has Results?**

14) **No-results response**
- Node: **Code** → “No Results Response”
  - Generate HTML showing question, keywords, pages searched, suggestions
- Node: **Respond to Webhook** → “Respond No Results”
  - Content-Type: text/html
  - Body: `{{ $json.htmlReport }}`
- Connect: **Has Results? (true) → No Results Response → Respond No Results**

15) **Build analysis request (Code)**
- Node: **Code** → “Build Analysis Request”
- Create prompt with top `MAX_POSTS_FOR_ANALYSIS` posts (20)
- Model = `config.ANALYSIS_MODEL`, max_tokens = `config.ANALYSIS_MAX_TOKENS`
- Output: `requestBody`, `apiKey`, and context fields
- Connect: **Has Results? (false) → Build Analysis Request**

16) **Call Anthropic for deep analysis (HTTP Request)**
- Node: **HTTP Request** → “Claude - Analyze”
- POST `https://api.anthropic.com/v1/messages`
- Timeout: 60000
- Same headers as keyword call
- Body: `{{ JSON.stringify($json.requestBody) }}`
- Connect: **Build Analysis Request → Claude - Analyze**

17) **Format response (Code)**
- Node: **Code** → “Format Response”
- Extract `analysis = content[0].text`
- Rebuild posts list with full content and comments for reporting
- Connect: **Claude - Analyze → Format Response**

18) **Prepare document + preview HTML (Code)**
- Node: **Code** → “Prepare Document Content”
- Build:
  - `docTitle`, `targetFolderId`
  - `docContent` (plain text full report)
  - `analysisHtml` and `postsHtml` (preview)
- Connect: **Format Response → Prepare Document Content**

19) **Google Docs: create document**
- Node: **Google Docs** → “Create Google Doc”
- Operation: Create
- Title: `{{ $json.docTitle }}`
- Folder ID: `{{ $json.targetFolderId }}`
- **Credentials:** Connect Google Docs OAuth2 in n8n with Drive/Docs scopes
- Connect: **Prepare Document Content → Create Google Doc**

20) **Google Docs: insert content**
- Node: **Google Docs** → “Add Content to Doc”
- Operation: Update → Insert text
- Document URL/ID: `{{ $json.id }}` (from Create node output)
- Text to insert: `{{ $('Prepare Document Content').first().json.docContent }}`
- Connect: **Create Google Doc → Add Content to Doc**

21) **Generate final HTML (Code)**
- Node: **Code** → “Generate Final HTML”
- Build:
  - Google Doc URL from doc id
  - Drive folder URL from `targetFolderId`
  - Final HTML with analysis and posts preview
- Connect: **Add Content to Doc → Generate Final HTML**

22) **Return HTML to the form request**
- Node: **Respond to Webhook** → “Respond with HTML”
- Content-Type: `text/html`
- Body: `{{ $json.htmlReport }}`
- Connect: **Generate Final HTML → Respond with HTML**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Authentication: copy `www.skool.com` cookie from DevTools → Application → Cookies | Required for Skool requests; cookie typically expires every 7–14 days (workflow includes a friendly error page) |
| BuildId is auto-detected from Skool HTML; fallback BuildId exists | Prevents breaking when Skool updates frontend; fallback may become stale |
| Anthropic: Haiku used for keyword extraction; Sonnet used for deep analysis | Cost/quality tradeoff; models configured in Config node |
| Google Drive folder setup: create a folder and put its ID into `DEFAULT_FOLDER_ID` | Used when user doesn’t supply `folder_id` in the form |
| Token safety limits exist (max post length, comment length, max posts analyzed) | Reduces prompt size and cost; adjust in Config if needed |