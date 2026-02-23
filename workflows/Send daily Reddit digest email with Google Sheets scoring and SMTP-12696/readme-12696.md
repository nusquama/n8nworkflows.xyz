Send daily Reddit digest email with Google Sheets scoring and SMTP

https://n8nworkflows.xyz/workflows/send-daily-reddit-digest-email-with-google-sheets-scoring-and-smtp-12696


# Send daily Reddit digest email with Google Sheets scoring and SMTP

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send daily Reddit digest email with Google Sheets scoring and SMTP  
**Workflow name (in JSON):** `RedditEmailSender - Template`  
**Purpose:** Every morning at a fixed time, the workflow pulls posts from configured Reddit subreddits via RSS (Atom XML), normalizes post data, scores posts using include/exclude keyword lists stored in Google Sheets, deduplicates against a “Seen” Google Sheet, appends newly seen posts, and sends an HTML digest email via SMTP.

**Target use cases**
- Daily monitoring of multiple subreddits for lead signals, questions, pain points, or brand/category mentions.
- Lightweight scoring/ranking without using an LLM.
- Ensuring no duplicate alerts using a persistent “Seen” store (Google Sheets).

### Logical blocks
1. **1.1 Scheduling / Trigger**
2. **1.2 Source configuration (Google Sheets)**
3. **1.3 Reddit RSS feed construction and retrieval**
4. **1.4 XML parsing + post normalization**
5. **1.5 Keyword loading + merging streams**
6. **1.6 Scoring + selection (top posts per subreddit)**
7. **1.7 Deduplication via “Seen” sheet + persistence**
8. **1.8 Digest email generation + SMTP send**

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling / Trigger
**Overview:** Triggers the workflow automatically every day at a specific hour (7:00).  
**Nodes involved:** `Schedule`

#### Node: Schedule
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration (interpreted):** Runs on an interval rule that triggers at hour **7** (server timezone of the n8n instance unless otherwise configured).
- **Inputs / outputs:** No inputs. Outputs to **Read Sources**.
- **Version requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - Timezone mismatch (expected “7am local” vs instance timezone).
  - Workflow inactive (`active: false` in JSON) means it won’t run until enabled.

**Sticky note covering block:**  
“## SCORE REDDIT POSTS AND EMAIL THEM TO YOURSELF EVERY MORNING TO RESPOND … Setup steps … Set up Google Sheets credentials … Set up SMTP credentials”

---

### 2.2 Source configuration (Google Sheets)
**Overview:** Reads the list of subreddit sources to monitor from a Google Sheet and filters to only enabled rows.  
**Nodes involved:** `Read Sources`

#### Node: Read Sources
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — fetch configuration rows.
- **Configuration (interpreted):**
  - **Document:** “RedditMonitor” (placeholder doc ID in JSON: `[yourdocid]`)
  - **Sheet:** “Sources” (`gid=0`)
  - **Filter:** `enabled` column must equal `"true"` (string comparison).
- **Credentials:** Google Sheets OAuth2 (`googleSheetsOAuth2Api`).
- **Inputs / outputs:** Input from **Schedule**. Output to **Build Feed URL** (one item per enabled source row).
- **Version requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - OAuth token expiry / missing permissions.
  - Sheet columns missing (`enabled`, `subreddit`, `feed_type` expected downstream).
  - Filter value is string `"true"`; if sheet stores TRUE boolean, filter may not match depending on how values are returned.

**Sticky note:** “### Read in sources”

---

### 2.3 Reddit RSS feed construction and retrieval
**Overview:** Builds the RSS (Atom) feed URL for each subreddit source row and fetches the XML via HTTP.  
**Nodes involved:** `Build Feed URL`, `Fetch Feed XML1`

#### Node: Build Feed URL
- **Type / role:** Set (`n8n-nodes-base.set`) — adds `rss_url`.
- **Configuration (interpreted):**
  - Keeps all existing fields (`includeOtherFields: true`)
  - Adds:
    - `rss_url = https://www.reddit.com/r/{{$json.subreddit}}/{{$json.feed_type}}/.rss`
- **Inputs / outputs:** Input from **Read Sources**. Output to **Fetch Feed XML1**.
- **Version requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Missing/blank `subreddit` or `feed_type` yields invalid URL.
  - `feed_type` must be a valid Reddit listing path segment (commonly `new`, `hot`, `top`, etc.).

#### Node: Fetch Feed XML1
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — pulls feed XML.
- **Configuration (interpreted):**
  - URL: uses expression `{{$json.rss_url}}` (note: JSON includes a trailing newline in the template; typically harmless but best removed).
  - Sends header `User-Agent: Mozilla/5.0 (compatible; RedditListener/1.0)`
- **Inputs / outputs:** Input from **Build Feed URL**. Output to **Parse XML**.
- **Version requirements:** Type version `4.3`.
- **Edge cases / failures:**
  - Reddit rate limiting / 429 responses; User-Agent helps but is not a guarantee.
  - Network timeouts, DNS issues.
  - Reddit can return HTML (e.g., bot protection) instead of XML; XML parse will fail downstream.

**Sticky note:** “### Build XML feed”

---

### 2.4 XML parsing + post normalization
**Overview:** Converts the RSS/Atom XML into JSON, splits each feed into one item per post entry, and normalizes common fields for scoring and tracking.  
**Nodes involved:** `Parse XML`, `Split Atom Entries`, `Normalize Post Fields`

#### Node: Parse XML
- **Type / role:** XML (`n8n-nodes-base.xml`) — parses XML string into JSON structure.
- **Configuration (interpreted):** Default options (no explicit XPath transforms).
- **Inputs / outputs:** Input from **Fetch Feed XML1**. Output to **Split Atom Entries**.
- **Version requirements:** Type version `1`.
- **Edge cases / failures:**
  - Invalid XML (HTML or partial content) leads to parsing errors.
  - Atom vs RSS shape differences; downstream code attempts to handle both.

#### Node: Split Atom Entries
- **Type / role:** Code (`n8n-nodes-base.code`) — “Run once for all items” behavior by iterating `$input.all()`; outputs one item per Reddit post across all fetched feeds.
- **Configuration (interpreted):**
  - For each parsed feed:
    - Locates entries under `feed.entry` (fallbacks: `rss`, or root)
    - Attempts to infer subreddit name from `feed.category.label`, `feed.title`, or existing fields
    - Extracts a post URL from `entry.link` (handles string/object/array shapes)
    - Emits normalized post fields: `subreddit`, `subreddit_key`, `title`, `url`, `published`, `author`, `post_id`, `date_found`
- **Inputs / outputs:** Input from **Parse XML** (many items, one per feed). Output to **Normalize Post Fields** (many items, one per post).
- **Version requirements:** Type version `2`.
- **Edge cases / failures:**
  - Feed entry structures may vary; code is defensive but may still produce empty `url` or `post_id`.
  - Some feeds might return a single entry object instead of array; code wraps into array.
  - Subreddit inference may return “unknown” if feed metadata differs.

#### Node: Normalize Post Fields
- **Type / role:** Set (`n8n-nodes-base.set`) — ensures consistent field names and sets `date_found`.
- **Configuration (interpreted):**
  - Assigns:
    - `date_found = {{$now}}`
    - `subreddit = {{$json.subreddit}}`
    - `title`, `url`, `published`, `author`, `post_id` from the current item
- **Inputs / outputs:**
  - Input from **Split Atom Entries**
  - Outputs to **Read Keywords** and to **Merge** (two parallel branches)
- **Version requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If `post_id` is blank, deduplication later may drop the post (Filter New Only requires ID).
  - `published` may not be parseable as date; scoring uses fallback to `date_found`.

**Sticky note:** “### Clean and normalize posts”

---

### 2.5 Keyword loading + merging streams
**Overview:** Loads include/exclude keywords from Google Sheets and merges them with the stream of posts so the scoring node can see both datasets at once.  
**Nodes involved:** `Read Keywords`, `Merge`

#### Node: Read Keywords
- **Type / role:** Google Sheets — reads keyword configuration rows.
- **Configuration (interpreted):**
  - **Document:** “RedditMonitor” (placeholder `[docid]`)
  - **Sheet:** “Keywords” (gid `1446769317`)
  - Expected columns: `include` and/or `exclude`
- **Credentials:** Google Sheets OAuth2.
- **Inputs / outputs:** Input from **Normalize Post Fields**. Output to **Merge** (input index 1).
- **Version requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Empty sheets cause scoring to become permissive (only pain/question heuristics may apply, but include list empty means score may be 0 and filtered out by MIN_SCORE=1).
  - Mixed data types (blank cells, booleans) — code normalizes to strings.

#### Node: Merge
- **Type / role:** Merge (`n8n-nodes-base.merge`) — combines two incoming streams for joint processing.
- **Configuration (interpreted):** Default merge behavior (no explicit mode shown). Practically, this workflow relies on the downstream **Scoring** code reading **all items** and separating “keyword rows” vs “post rows” based on presence of `include`/`exclude` fields.
- **Inputs / outputs:**
  - Input 0: posts from **Normalize Post Fields**
  - Input 1: keyword rows from **Read Keywords**
  - Output: combined item list to **Scoring**
- **Version requirements:** Type version `3.2`.
- **Edge cases / failures:**
  - If merge mode were changed to a pairing mode, it could break the “all items” assumption in Scoring. Keep it in a mode that outputs all incoming items.
  - Column naming: if posts accidentally have `include` or `exclude` fields, Scoring may misclassify them as keyword rows.

**Sticky note:** “### Keywords for every pos” (typo in note; intent: posts)

---

### 2.6 Scoring + selection (top posts per subreddit)
**Overview:** Scores each post based on include keyword hits (and bonuses), excludes posts containing exclude keywords, groups by subreddit, and selects top N per subreddit.  
**Nodes involved:** `Scoring`, `Collect Candidates`

#### Node: Scoring
- **Type / role:** Code — main ranking logic.
- **Configuration (interpreted):**
  - Splits merged input into:
    - `keywordRows` = items having `include` or `exclude`
    - `postsRaw` = everything else
  - Normalization: lowercase, whitespace normalization, dash normalization.
  - **Exclude logic:** if any exclude keyword is found in title/body text ⇒ drop post.
  - **Include scoring:** each include hit adds `12`, title hit adds `8`.
  - **Heuristics:**
    - `?` in title adds `6`
    - “pain phrases” add `10` once if any matches (examples: “overwhelmed”, “forgot to follow up”, etc.)
  - Determines a `subreddit_key` from multiple possible fields; falls back to parsing from URL.
  - Takes **TOP_N = 5 per subreddit**, sorted by score desc then recency desc.
  - Filters by `MIN_SCORE = 1`.
  - Outputs scored posts as individual items with fields:
    - `score`, `include_hits`, `exclude_hits`, `subreddit_key`, plus original post fields
- **Inputs / outputs:** Input from **Merge**. Output to **Collect Candidates**.
- **Version requirements:** Type version `2`.
- **Edge cases / failures:**
  - If include list is empty and no heuristics fire, most posts score 0 and are filtered out (MIN_SCORE=1).
  - Keyword matching is substring-based; may overmatch (e.g., “crm” matches “sacramento” if not careful).
  - The code references `content_html`/`content`, but upstream normalization does not populate content from the Atom feed, so scoring may effectively only use titles unless you extend parsing.

#### Node: Collect Candidates
- **Type / role:** Code — converts multiple scored items into one item containing an array.
- **Configuration (interpreted):**
  - `candidates = $input.all().map(i => i.json)`
  - Returns one item: `{ candidates: [...] }`
- **Inputs / outputs:** Input from **Scoring**. Output to **Read Seen**.
- **Version requirements:** Type version `2`.
- **Edge cases / failures:**
  - If Scoring returns zero items, `candidates` becomes empty array; downstream still works.

**Sticky note:** “### Score and filter posts”

---

### 2.7 Deduplication via “Seen” sheet + persistence
**Overview:** Reads the historical list of seen post IDs from Google Sheets, filters the candidate list to new items only, then appends new items into the Seen sheet.  
**Nodes involved:** `Read Seen`, `Filter New Only`, `Append Seen`

#### Node: Read Seen
- **Type / role:** Google Sheets — loads the deduplication registry.
- **Configuration (interpreted):**
  - Document: “RedditMonitor”
  - Sheet: “Seen” (gid `25579448`)
  - `alwaysOutputData: true` ensures the node outputs even if the sheet is empty (important so filtering doesn’t break).
- **Credentials:** Google Sheets OAuth2.
- **Inputs / outputs:** Input from **Collect Candidates**. Output to **Filter New Only**.
- **Version requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Large Seen sheet can slow down execution; consider limiting rows or using a database if it grows.
  - If `post_id` values in Seen contain whitespace/formatting differences, dedupe may fail (the code trims).

#### Node: Filter New Only
- **Type / role:** Code — removes candidates whose `post_id` already exists in Seen.
- **Configuration (interpreted):**
  - `seenRows` read from current input (Read Seen output)
  - `candidates` read via cross-node reference: `$node["Collect Candidates"].json.candidates`
  - Builds `seen` Set of `post_id`
  - Keeps only candidates with non-empty `post_id` not in seen
  - Outputs one item per fresh post
- **Inputs / outputs:** Input from **Read Seen**. Outputs to **Append Seen** and **Build Email Digest** in parallel.
- **Version requirements:** Type version `2`.
- **Edge cases / failures:**
  - If a candidate post lacks `post_id`, it is dropped (`return id && !seen.has(id)`).
  - Cross-node reference assumes **Collect Candidates** executed in same run and has `.json.candidates`. If the graph changes, this can break.
  - If Read Seen fails and doesn’t output, the node won’t run; `alwaysOutputData` reduces (but doesn’t eliminate) this risk.

#### Node: Append Seen
- **Type / role:** Google Sheets — appends newly processed posts to Seen sheet.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Writes columns:
    - `title` = post title
    - `post_id`
    - `post_url` = post url
    - `subreddit`
    - `first_seen_utc` = `new Date().toISOString()`
  - Mapping mode: “define below” (explicit mapping)
- **Credentials:** Google Sheets OAuth2.
- **Inputs / outputs:** Input from **Filter New Only**. No downstream nodes.
- **Version requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Sheet schema mismatch (column headers must match).
  - Append rate limits if many posts.
  - The `sheetName` cachedResultUrl in JSON appears malformed (`...[docid]edit#gid=...` missing a slash). Usually harmless because n8n uses `documentId` + `gid`, but verify in UI.

**Sticky note:** “### Keep only new posts”

---

### 2.8 Digest email generation + SMTP send
**Overview:** Builds a single HTML digest email grouped by subreddit, with clickable titles, then sends via SMTP.  
**Nodes involved:** `Build Email Digest`, `Send email`

#### Node: Build Email Digest
- **Type / role:** Code — constructs one email payload (`subject`, `html`) for all posts.
- **Configuration (interpreted):**
  - Formats current time in `America/Chicago` timezone for the subject/date label.
  - If no posts: sends a “no new posts” email.
  - Otherwise:
    - Groups posts by `subreddit`
    - Sorts within group by `score`
    - Sorts subreddits by top score
    - HTML-escapes titles and metadata
    - Uses `url` (or `post_url`) to create clickable title; if missing, renders non-clickable title
  - Output is exactly one item with `json.subject` and `json.html`.
- **Inputs / outputs:** Input from **Filter New Only** (many items). Output to **Send email**.
- **Version requirements:** Type version `2`.
- **Edge cases / failures:**
  - Subreddit formatting: it prints `r/${sub}` but also prepends `r/` in HTML; if `subreddit` already includes `r/`, it may become `r/r/foo`.
  - The code uses a “📬” icon in HTML; some mail clients may render inconsistently.
  - If URLs are not HTTPS or contain special characters, ensure they are valid; no URL encoding is performed.

#### Node: Send email
- **Type / role:** Email Send (`n8n-nodes-base.emailSend`) — sends the digest.
- **Configuration (interpreted):**
  - To: `[youremail]@info.com`
  - From: `[youremail]@info.com`
  - Subject: `{{$json.subject}}`
  - HTML body: `{{$json.html}}`
  - Uses SMTP credentials (`SMTP account`)
- **Inputs / outputs:** Input from **Build Email Digest**. No downstream nodes.
- **Version requirements:** Type version `2.1`.
- **Edge cases / failures:**
  - SMTP auth failures, TLS/port mismatch, blocked outbound SMTP.
  - Some SMTP providers require `fromEmail` to match authenticated user/domain.
  - If you send a “no new posts” email daily, ensure that’s desired (can be noisy).

**Sticky note:** “### Build and Send email”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule | scheduleTrigger | Daily trigger at 7:00 | — | Read Sources | ## SCORE REDDIT POSTS AND EMAIL THEM TO YOURSELF EVERY MORNING TO RESPOND … Setup steps … Set up Google Sheets credentials … Set up SMTP credentials |
| Read Sources | googleSheets | Load enabled subreddit/feed sources | Schedule | Build Feed URL | ### Read in sources |
| Build Feed URL | set | Build Reddit RSS URL per source | Read Sources | Fetch Feed XML1 | ### Build XML feed |
| Fetch Feed XML1 | httpRequest | Fetch RSS/Atom XML from Reddit | Build Feed URL | Parse XML | ### Build XML feed |
| Parse XML | xml | Parse XML to JSON | Fetch Feed XML1 | Split Atom Entries | ### Clean and normalize posts |
| Split Atom Entries | code | Split feed entries into one item per post | Parse XML | Normalize Post Fields | ### Clean and normalize posts |
| Normalize Post Fields | set | Standardize key post fields | Split Atom Entries | Read Keywords; Merge | ### Clean and normalize posts |
| Read Keywords | googleSheets | Load include/exclude keyword rows | Normalize Post Fields | Merge | ### Keywords for every pos |
| Merge | merge | Combine keyword rows + post rows | Normalize Post Fields; Read Keywords | Scoring | ### Keywords for every pos |
| Scoring | code | Score/filter, top N per subreddit | Merge | Collect Candidates | ### Score and filter posts |
| Collect Candidates | code | Wrap scored items into `candidates[]` | Scoring | Read Seen | ### Score and filter posts |
| Read Seen | googleSheets | Load already-seen post IDs | Collect Candidates | Filter New Only | ### Keep only new posts |
| Filter New Only | code | Remove candidates already in Seen | Read Seen (and references Collect Candidates) | Append Seen; Build Email Digest | ### Keep only new posts |
| Append Seen | googleSheets | Append new posts to Seen sheet | Filter New Only | — | ### Keep only new posts |
| Build Email Digest | code | Build single HTML digest | Filter New Only | Send email | ### Build and Send email |
| Send email | emailSend | Send digest via SMTP | Build Email Digest | — | ### Build and Send email |
| Sticky Note | stickyNote | Comment / instructions | — | — | ## SCORE REDDIT POSTS AND EMAIL THEM TO YOURSELF EVERY MORNING TO RESPOND … Setup steps … |
| Sticky Note1 | stickyNote | Comment | — | — | ### Read in sources |
| Sticky Note2 | stickyNote | Comment | — | — | ### Build XML feed |
| Sticky Note3 | stickyNote | Comment | — | — | ### Clean and normalize posts |
| Sticky Note4 | stickyNote | Comment | — | — | ### Keywords for every pos |
| Sticky Note5 | stickyNote | Comment | — | — | ### Score and filter posts |
| Sticky Note6 | stickyNote | Comment | — | — | ### Keep only new posts |
| Sticky Note7 | stickyNote | Comment | — | — | ### Build and Send email |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it e.g. **“RedditEmailSender”**.
   - Ensure you can add Google Sheets OAuth2 credentials and SMTP credentials in n8n.

2. **Add Trigger: “Schedule”**
   - Node: **Schedule Trigger**
   - Configure it to trigger **daily at 07:00** (confirm timezone in n8n settings).

3. **Create Google Sheet structure**
   - Spreadsheet (example name): `RedditMonitor`
   - Sheet **Sources** with at least:
     - `enabled` (use `"true"` as text, or adjust filter accordingly)
     - `subreddit` (e.g., `sales`, `SaaS`, etc. without `r/`)
     - `feed_type` (e.g., `new`, `hot`, `top`)
   - Sheet **Keywords** with columns:
     - `include` (one keyword/phrase per row; can be blank)
     - `exclude` (one keyword/phrase per row; can be blank)
   - Sheet **Seen** with columns:
     - `post_id`
     - `first_seen_utc`
     - `subreddit`
     - `title`
     - `post_url`

4. **Add node: “Read Sources” (Google Sheets)**
   - Node: **Google Sheets**
   - Operation: read/get many rows (use the standard “Read” operation in UI)
   - Select document: `RedditMonitor`
   - Select sheet: `Sources`
   - Add filter: `enabled` **equals** `true` (if your sheet stores booleans, set filter accordingly; otherwise keep string `"true"`).
   - Connect: **Schedule → Read Sources**

5. **Add node: “Build Feed URL” (Set)**
   - Add field `rss_url` with expression:  
     `https://www.reddit.com/r/{{$json.subreddit}}/{{$json.feed_type}}/.rss`
   - Keep other fields enabled.
   - Connect: **Read Sources → Build Feed URL**

6. **Add node: “Fetch Feed XML1” (HTTP Request)**
   - Method: GET
   - URL expression: `{{$json.rss_url}}`
   - Add header: `User-Agent = Mozilla/5.0 (compatible; RedditListener/1.0)`
   - Connect: **Build Feed URL → Fetch Feed XML1**

7. **Add node: “Parse XML” (XML)**
   - Use default parse options.
   - Connect: **Fetch Feed XML1 → Parse XML**

8. **Add node: “Split Atom Entries” (Code)**
   - Paste the “Split Atom Entries” JS logic from the workflow (the code that iterates `$input.all()` and outputs one item per entry).
   - Connect: **Parse XML → Split Atom Entries**

9. **Add node: “Normalize Post Fields” (Set)**
   - Set fields:
     - `date_found = {{$now}}`
     - `subreddit = {{$json.subreddit}}`
     - `title = {{$json.title}}`
     - `url = {{$json.url}}`
     - `published = {{$json.published}}`
     - `author = {{$json.author}}`
     - `post_id = {{$json.post_id}}`
   - Connect: **Split Atom Entries → Normalize Post Fields**

10. **Add node: “Read Keywords” (Google Sheets)**
    - Document: `RedditMonitor`
    - Sheet: `Keywords`
    - Connect: **Normalize Post Fields → Read Keywords**

11. **Add node: “Merge” (Merge)**
    - Connect **Normalize Post Fields → Merge (Input 0)**
    - Connect **Read Keywords → Merge (Input 1)**
    - Keep a merge mode that outputs both streams (so the downstream code can see all items).

12. **Add node: “Scoring” (Code)**
    - Paste the scoring JS code from the workflow (include/exclude scanning, bonuses, top 5 per subreddit, MIN_SCORE=1).
    - Connect: **Merge → Scoring**

13. **Add node: “Collect Candidates” (Code)**
    - Code:
      - `const candidates = $input.all().map(i => i.json); return [{ json: { candidates } }];`
    - Connect: **Scoring → Collect Candidates**

14. **Add node: “Read Seen” (Google Sheets)**
    - Document: `RedditMonitor`
    - Sheet: `Seen`
    - Enable **Always Output Data** (important for empty sheets).
    - Connect: **Collect Candidates → Read Seen**

15. **Add node: “Filter New Only” (Code)**
    - Paste the “Filter New Only” code:
      - Builds a Set of `post_id` from Seen rows
      - Reads candidates from `$node["Collect Candidates"].json.candidates`
      - Outputs only unseen posts
    - Connect: **Read Seen → Filter New Only**

16. **Add node: “Append Seen” (Google Sheets)**
    - Operation: **Append**
    - Document: `RedditMonitor`
    - Sheet: `Seen`
    - Map columns:
      - `title = {{$json.title}}`
      - `post_id = {{$json.post_id}}`
      - `post_url = {{$json.url}}`
      - `subreddit = {{$json.subreddit}}`
      - `first_seen_utc = {{new Date().toISOString()}}`
    - Connect: **Filter New Only → Append Seen**

17. **Add node: “Build Email Digest” (Code)**
    - Paste the HTML digest builder code (group by subreddit, build subject/html, Chicago timezone formatting).
    - Connect: **Filter New Only → Build Email Digest**

18. **Add node: “Send email” (Email Send)**
    - Configure SMTP credentials:
      - Host, port, username/password or OAuth depending on provider; enable TLS as needed.
    - Set:
      - **To Email:** your destination address
      - **From Email:** your sender address (must be permitted by SMTP provider)
      - **Subject:** `{{$json.subject}}`
      - **HTML:** `{{$json.html}}`
    - Connect: **Build Email Digest → Send email**

19. **Credentials**
    - **Google Sheets OAuth2:** authorize access to the Google account containing `RedditMonitor`.
    - **SMTP:** ensure your provider allows the chosen From/To, and that n8n can reach the SMTP server.

20. **Activate workflow**
    - Toggle workflow to **Active** to run daily.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “SCORE REDDIT POSTS AND EMAIL THEM TO YOURSELF EVERY MORNING TO RESPOND … Setup steps … Create google sheet … Set up Google Sheets credentials … Set up SMTP credentials” | Global workflow intent and setup guidance (from main sticky note) |
| “Read in sources” | Sticky note labeling the sources block |
| “Build XML feed” | Sticky note labeling RSS construction and fetch |
| “Clean and normalize posts” | Sticky note labeling XML parsing + normalization |
| “Keywords for every pos” | Sticky note labeling keyword configuration (typo preserved) |
| “Score and filter posts” | Sticky note labeling scoring block |
| “Keep only new posts” | Sticky note labeling deduplication block |
| “Build and Send email” | Sticky note labeling email output block |