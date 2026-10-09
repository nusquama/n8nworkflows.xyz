Review unlinked brand mentions from Google News with HasData and Google Sheets

https://n8nworkflows.xyz/workflows/review-unlinked-brand-mentions-from-google-news-with-hasdata-and-google-sheets-20478


# Review unlinked brand mentions from Google News with HasData and Google Sheets

### 1. Workflow Overview

This workflow automates the discovery and evaluation of brand mentions in Google News to find potential unlinked brand mentions. It queries Google News using HasData, aggregates and deduplicates results, filters out owned or excluded domains, scrapes a limited subset of target web pages, checks for the presence of brand aliases and direct website links, and stores the structured results in Google Sheets for manual review.

The logical processing is divided into five distinct blocks:
- **1.1 Initialization & Input Validation:** Triggers the execution, establishes global parameters (such as target domain, brand aliases, and market location), and validates configuration constraints against a Google Sheets input source.
- **1.2 Search Execution & Deduplication:** Fetches Google News search results via HasData API for each query, validates against existing results, combines duplicate articles, and prioritizes pages based on query frequency and publication date.
- **1.3 Page Sampling & Scraping:** Evaluates page budgets, selects eligible URLs to scrape, extracts raw HTML content and links using HasData, and sanitizes/parses the extracted text.
- **1.4 Analysis & Review Preparation:** Analyzes extracted page text for brand alias mentions and verifies if direct outbound links point to the target domain, categorizing each item into specific review statuses.
- **1.5 Persistence & Reporting:** Formulates unique result keys, packages the findings into structured JSON evidence objects, and upserts the records into a Google Sheets tracking queue.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Initialization & Input Validation
**Overview:**  
Initializes the manual run, defines global configuration settings (domain, aliases, regions, limits), validates integrity rules via custom code, and retrieves search queries from a Google Sheets document.

**Nodes Involved:**
- `Run manually`
- `Settings`
- `Validate settings`
- `Read input`
- `Validate input batch`
- `Prepare queue read`

#### Node Details:
- **Run manually**
  - **Type & Role:** `n8n-nodes-base.manualTrigger` (Trigger node). Starts the workflow on demand.
  - **Configuration:** Default parameters (no incoming connections).
  - **Connections:** Output connects to `Settings`.
  - **Edge Cases:** None.

- **Settings**
  - **Type & Role:** `n8n-nodes-base.set` (Data transformation/assignment). Sets configuration variables like `spreadsheet_id`, `own_domain`, `brand_aliases`, `excluded_domains`, `gl`, `hl`, `max_items`, `max_pages`, `content_selector`, and `js_rendering`.
  - **Configuration:** Assignments mode with string/boolean/number assignments.
  - **Connections:** Input from `Run manually`; output connects to `Validate settings`.
  - **Edge Cases:** Placeholder values (e.g., `REPLACE_WITH_SPREADSHEET_ID`) trigger validation errors in downstream nodes.

- **Validate settings**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Validates configuration variables (spreadsheet ID format, domain normalization, alias limits, ISO language/country codes).
  - **Configuration:** Run Once For All Items.
  - **Key Expressions/Variables:** Reads upstream JSON variables and outputs sanitized configuration parameters plus a `run_id` timestamp.
  - **Connections:** Input from `Settings`; output connects to `Read input`.
  - **Edge Cases:** Throws explicit errors if configuration parameters violate format or length constraints.

- **Read input**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data retrieval). Reads search queries from the `Queries` sheet.
  - **Configuration:** Uses resource `sheet`, operation `read`, referencing `spreadsheet_id` from the `Validate settings` node. Always outputs data.
  - **Connections:** Input from `Validate settings`; output connects to `Validate input batch`.
  - **Edge Cases:** API authentication errors if Google Sheets credentials are invalid or missing.

- **Validate input batch**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Validates query row quantities, detects duplicate queries, and structures the query parameters (`query`, `fields: { gl, hl }`).
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Read input`; output connects to `Prepare queue read`.
  - **Edge Cases:** Throws an error if zero queries are provided, if batch size exceeds `max_items`, or if duplicate queries exist.

- **Prepare queue read**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Prepares a dummy payload flag (`ready: true`) to sequence the subsequent Google Sheets read operation.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Validate input batch`; output connects to `Read Results`.
  - **Edge Cases:** None.

---

### Block 1.2: Search Execution & Deduplication
**Overview:**  
Reads existing historical results from Google Sheets, checks for data anomalies, queries Google News via HasData API for each input term, and aggregates/prioritizes the resulting articles.

**Nodes Involved:**
- `Read Results`
- `Check saved keys`
- `Fetch search evidence`
- `Attach search context`
- `Plan page sample`

#### Node Details:
- **Read Results**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data retrieval). Reads historical execution entries from the `Results` sheet.
  - **Configuration:** Uses resource `sheet`, operation `read`, referencing `spreadsheet_id`.
  - **Connections:** Input from `Prepare queue read`; output connects to `Check saved keys`.
  - **Edge Cases:** Fails if the `Results` tab is missing or credentials lack read access.

- **Check saved keys**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Validates that historical `result_key` rows in the `Results` sheet contain no duplicates.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Read Results`; output connects to `Fetch search evidence`.
  - **Edge Cases:** Throws an error if duplicate `result_key` entries are detected.

- **Fetch search evidence**
  - **Type & Role:** `@hasdata/n8n-nodes-hasdata.hasData` (External API integration / Community node). Queries Google News via the HasData API.
  - **Configuration:** Resource `google_serp`, Operation `news`. Passes query `={{ $json.query }}` and parameters `={{ $json.fields }}`. Configured to continue regular output on error.
  - **Connections:** Input from `Check saved keys`; output connects to `Attach search context`.
  - **Edge Cases:** API timeout, rate limiting, or invalid HasData credentials.

- **Attach search context**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Pairs search queries with API responses, normalizes article URLs, extracts story lists, and filters out malformed URLs.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Fetch search evidence`; output connects to `Plan page sample`.
  - **Edge Cases:** Ambiguous response pairing or missing response items trigger runtime exceptions.

- **Plan page sample**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Deduplicates article URLs across all queries, counts citing queries, sorts by query frequency and publication date, and assigns fetching statuses based on `max_pages` limits and domain exclusion rules.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Attach search context`; output connects to `Pages to inspect`.
  - **Edge Cases:** None.

---

### Block 1.3: Page Sampling & Scraping
**Overview:**  
Evaluates whether eligible pages exist for scraping, splits requests into individual payloads, scrapes web page content using HasData Web Scraping API, and extracts structured HTML elements (text, titles, links).

**Nodes Involved:**
- `Pages to inspect`
- `Split page requests`
- `Read source pages`
- `Pair source pages`
- `Extract page evidence`
- `No eligible pages`
- `Keep empty page inventory`

#### Node Details:
- **Pages to inspect**
  - **Type & Role:** `n8n-nodes-base.if` (Conditional routing). Branches workflow execution based on whether eligible pages (`pages.length > 0`) were found.
  - **Configuration:** Evaluates `={{ $json.pages.length > 0 }}`.
  - **Connections:** Input from `Plan page sample`; True output connects to `Split page requests`, False output connects to `No eligible pages`.
  - **Edge Cases:** None.

- **Split page requests**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Unpacks the array of pages slated for fetching into individual items for downstream processing.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Pages to inspect` (True branch); output connects to `Read source pages`.
  - **Edge Cases:** None.

- **Read source pages**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (HTTP API integration). Calls HasData Web Scraping API (`https://api.hasdata.com/scrape/web`) via POST.
  - **Configuration:** Uses predefined HasData API credentials, JSON body containing target URL, output formats (`json`, `html`), and `jsRendering` setting. Configured with a 330-second timeout, batching (batch size 1, 250ms interval), and continues regular output on error.
  - **Connections:** Input from `Split page requests`; output connects to `Pair source pages`.
  - **Edge Cases:** HTTP 4xx/5xx errors, timeouts on heavy pages, or blocked requests.

- **Pair source pages**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Reconnects scraping request payloads with their respective HTTP response bodies.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Read source pages`; output connects to `Extract page evidence`.
  - **Edge Cases:** Response-to-plan mismatch throws an error.

- **Extract page evidence**
  - **Type & Role:** `n8n-nodes-base.html` (HTML parsing/extraction). Extracts text, titles, H1 headings, and all anchor link `href` attributes from the scraped HTML content.
  - **Configuration:** Uses CSS selectors (e.g., `={{ $('Validate settings').first().json.content_selector }}` for main text, `title`, `h1`, `a[href]`), skips non-content tags (`script`, `style`, `nav`, `header`, `footer`, `noscript`), and cleans up text. Configured to continue regular output on error.
  - **Connections:** Input from `Pair source pages`; output connects to `Collect page evidence`.
  - **Edge Cases:** Missing DOM elements or unparseable HTML.

- **No eligible pages**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Initializes an empty pages array (`{ pages: [] }`) if no pages match the criteria.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Pages to inspect` (False branch); output connects to `Keep empty page inventory`.
  - **Edge Cases:** None.

- **Keep empty page inventory**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Passes through empty page inventory data.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `No eligible pages`; output connects to `Build review rows`.
  - **Edge Cases:** None.

---

### Block 1.4: Analysis & Review Preparation
**Overview:**  
Collects scraped page data, verifies text length and HTTP response status, performs regex-based brand alias matching with word boundaries, checks for direct outbound links to the target domain, and constructs comprehensive review rows.

**Nodes Involved:**
- `Collect page evidence`
- `Build review rows`

#### Node Details:
- **Collect page evidence**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Aggregates extracted HTML elements, checks read success criteria (minimum text length, HTTP status), truncates text/links if overly long, and compiles page metadata.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Input from `Extract page evidence`; output connects to `Build review rows`.
  - **Edge Cases:** Parsing failures or empty text content flagged as unknown status.

- **Build review rows**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution). Evaluates whether scraped articles mention brand aliases and contain outbound links to the target domain. Assigns statuses (e.g., `review_unlinked_mention`, `brand_link_observed`, `no_brand_match_in_text`) and builds JSON evidence packages for each row.
  - **Configuration:** Run Once For All Items.
  - **Connections:** Inputs from `Collect page evidence` and `Plan page sample` (via node reference); output connects to `Save review queue`.
  - **Edge Cases:** Throws an error if serialized `evidence_json` exceeds Google Sheets cell character limits (44,000 characters).

---

### Block 1.5: Persistence & Reporting
**Overview:**  
Saves and updates the final review queue and discovery inventory in Google Sheets using unique result keys.

**Nodes Involved:**
- `Save review queue`

#### Node Details:
- **Save review queue**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data persistence). Appends or updates records in the `Results` sheet.
  - **Configuration:** Resource `sheet`, operation `appendOrUpdate`, matching on `result_key`. Maps sheet columns (`result_key`, `checked_at`, `topic`, `status`, `title`, `source_url`, `publisher`, `published_at`, `brand_excerpt`, `observed_links`, `citing_queries`, `evidence_json`) using expression bindings. Cell formatting set to `RAW`.
  - **Connections:** Input from `Build review rows`; no outgoing connections.
  - **Edge Cases:** API write limits, permission issues, or data type mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run manually | n8n-nodes-base.manualTrigger | Starts workflow execution | None | Settings | # Review brand mentions that may be missing a website link<br><br>### How it works...<br>*(See Overview note)* |
| Settings | n8n-nodes-base.set | Sets global configuration variables | Run manually | Validate settings | ## 1. Choose your brand and searches<br>Create Queries with the header `query`. Add brand-focused Google News queries... |
| Validate settings | n8n-nodes-base.code | Validates configuration parameters | Settings | Read input | ## 1. Choose your brand and searches<br>Create Queries with the header `query`. Add brand-focused Google News queries... |
| Read input | n8n-nodes-base.googleSheets | Reads search queries from spreadsheet | Validate settings | Validate input batch | ## 1. Choose your brand and searches<br>Create Queries with the header `query`. Add brand-focused Google News queries... |
| Validate input batch | n8n-nodes-base.code | Validates query batch and constraints | Read input | Prepare queue read | ## 2. Validate the batch and saved keys<br>Reject duplicate queries and oversized batches before paid requests... |
| Prepare queue read | n8n-nodes-base.code | Prepares sequence flag for sheet read | Validate input batch | Read Results | ## 2. Validate the batch and saved keys<br>Reject duplicate queries and oversized batches before paid requests... |
| Read Results | n8n-nodes-base.googleSheets | Reads historical results | Prepare queue read | Check saved keys | ## 2. Validate the batch and saved keys<br>Reject duplicate queries and oversized batches before paid requests... |
| Check saved keys | n8n-nodes-base.code | Validates uniqueness of saved result keys | Read Results | Fetch search evidence | ## 2. Validate the batch and saved keys<br>Reject duplicate queries and oversized batches before paid requests... |
| Fetch search evidence | @hasdata/n8n-nodes-hasdata.hasData | Queries Google News via HasData API | Check saved keys | Attach search context | ## 3. Combine articles and limit page reads<br>Deduplicate article URLs and retain all queries that returned each one... |
| Attach search context | n8n-nodes-base.code | Normalizes news results and pairs queries | Fetch search evidence | Plan page sample | ## 3. Combine articles and limit page reads<br>Deduplicate article URLs and retain all queries that returned each one... |
| Plan page sample | n8n-nodes-base.code | Deduplicates articles and plans page fetch budget | Attach search context | Pages to inspect | ## 3. Combine articles and limit page reads<br>Deduplicate article URLs and retain all queries that returned each one... |
| Pages to inspect | n8n-nodes-base.if | Checks if eligible pages exist to fetch | Plan page sample | Split page requests, No eligible pages | ## 4. Inspect mentions and website links<br>Read selected articles and extract text and links... |
| Split page requests | n8n-nodes-base.code | Unpacks page list into individual items | Pages to inspect | Read source pages | ## 4. Inspect mentions and website links<br>Read selected articles and extract text and links... |
| Read source pages | n8n-nodes-base.httpRequest | Scrapes web page content via HasData API | Split page requests | Pair source pages | ## 4. Inspect mentions and website links<br>Read selected articles and extract text and links... |
| Pair source pages | n8n-nodes-base.code | Pairs scrape requests with HTTP responses | Read source pages | Extract page evidence | ## 4. Inspect mentions and website links<br>Read selected articles and extract text and links... |
| Extract page evidence | n8n-nodes-base.html | Extracts text, headings, and links from HTML | Pair source pages | Collect page evidence | ## 4. Inspect mentions and website links<br>Read selected articles and extract text and links... |
| Collect page evidence | n8n-nodes-base.code | Evaluates scrape success and truncates text | Extract page evidence | Build review rows | ## 4. Inspect mentions and website links<br>Read selected articles and extract text and links... |
| No eligible pages | n8n-nodes-base.code | Outputs empty page list if no pages match | Pages to inspect | Keep empty page inventory | ## Keep the discovery record<br>No selected articles means no page requests. Empty news responses... |
| Keep empty page inventory | n8n-nodes-base.code | Passes through empty page inventory | No eligible pages | Build review rows | ## Keep the discovery record<br>No selected articles means no page requests. Empty news responses... |
| Build review rows | n8n-nodes-base.code | Analyzes text mentions/links and builds rows | Collect page evidence, Plan page sample | Save review queue | ## 5. Review the article queue in Sheets<br>Create Results headers: `result_key`, `checked_at`, `topic`... |
| Save review queue | n8n-nodes-base.googleSheets | Appends or updates results in Google Sheets | Build review rows | None | ## 5. Review the article queue in Sheets<br>Create Results headers: `result_key`, `checked_at`, `topic`... |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run manually`.

2. **Add Settings Node:**
   - Add a **Set** node (`n8n-nodes-base.set`). Name it `Settings`.
   - Configure string assignments: `spreadsheet_id` (`REPLACE_WITH_SPREADSHEET_ID`), `own_domain` (`hasdata.com`), `brand_aliases` (`HasData\nHas Data`), `excluded_domains` (`facebook.com\nyoutube.com\nlinkedin.com`), `gl` (`us`), `hl` (`en`), `content_selector` (`main, article, [role="main"]`).
   - Configure number assignments: `max_items` (`3`), `max_pages` (`5`).
   - Configure boolean assignment: `js_rendering` (`false`).
   - *Connection:* Connect `Run manually` -> `Settings`.

3. **Add Settings Validation Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate settings`. Set execution mode to *Run Once For All Items*.
   - Paste validation logic ensuring proper spreadsheet ID format, domain cleaning, alias validation (1–10 aliases, 3–80 chars), and two-letter country/language codes.
   - *Connection:* Connect `Settings` -> `Validate settings`.

4. **Read Input Queries from Google Sheets:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`). Name it `Read input`.
   - Set Resource to `Sheet`, Operation to `Read`.
   - Set Document ID to expression: `={{ $('Validate settings').first().json.spreadsheet_id }}` and Sheet Name to `Queries`. Enable *Always Output Data*.
   - *Connection:* Connect `Validate settings` -> `Read input`.

5. **Validate Input Batch Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate input batch`. Set to *Run Once For All Items*.
   - Add validation code to check for empty queries, duplicate queries, and max item limits. Output mapped query objects with fields `{ gl, hl }`.
   - *Connection:* Connect `Read input` -> `Validate input batch`.

6. **Prepare Queue Read Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Prepare queue read`. Set to *Run Once For All Items*. Return `{ ready: true }`.
   - *Connection:* Connect `Validate input batch` -> `Prepare queue read`.

7. **Read Historical Results from Google Sheets:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`). Name it `Read Results`.
   - Set Resource to `Sheet`, Operation to `Read`.
   - Set Document ID to expression: `={{ $('Validate settings').first().json.spreadsheet_id }}` and Sheet Name to `Results`.
   - *Connection:* Connect `Prepare queue read` -> `Read Results`.

8. **Check Saved Keys Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Check saved keys`. Set to *Run Once For All Items*. Validates uniqueness of `result_key` column entries.
   - *Connection:* Connect `Read Results` -> `Check saved keys`.

9. **Fetch Search Evidence via HasData:**
   - Add a **HasData** community node (`@hasdata/n8n-nodes-hasdata.hasData`). Name it `Fetch search evidence`.
   - Set Resource to `Google Serp`, Operation to `News`.
   - Set Query parameter to `={{ $json.query }}` and Additional Fields to `={{ $json.fields }}`.
   - Configure error handling: Set *On Error* to `Continue Regular Output`. Configure HasData API credentials.
   - *Connection:* Connect `Check saved keys` -> `Fetch search evidence`.

10. **Attach Search Context Code Node:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Attach search context`. Set to *Run Once For All Items*. Normalizes news article URLs, extracts story lists, and handles error states.
    - *Connection:* Connect `Fetch search evidence` -> `Attach search context`.

11. **Plan Page Sample Code Node:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Plan page sample`. Set to *Run Once For All Items*. Deduplicates URLs, counts citing queries, sorts by date/frequency, and applies page budget limits.
    - *Connection:* Connect `Attach search context` -> `Plan page sample`.

12. **Branching Condition (Pages to Inspect):**
    - Add an **If** node (`n8n-nodes-base.if`). Name it `Pages to inspect`.
    - Set condition: Boolean evaluation `={{ $json.pages.length > 0 }}` is true.
    - *Connection:* Connect `Plan page sample` -> `Pages to inspect`.

13. **Handle Empty Pages Path (False Branch):**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `No eligible pages`. Return `{ pages: [] }`.
    - Connect to another **Code** node named `Keep empty page inventory` which passes through items.
    - *Connection:* Connect `Pages to inspect` (False) -> `No eligible pages` -> `Keep empty page inventory`.

14. **Split Page Requests Path (True Branch):**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Split page requests`. Unpacks `pages` array into individual items.
    - *Connection:* Connect `Pages to inspect` (True) -> `Split page requests`.

15. **Read Source Pages via HTTP Request:**
    - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Read source pages`.
    - Method: `POST`, URL: `https://api.hasdata.com/scrape/web`.
    - Authentication: Predefined credential type `hasDataApi`.
    - Request Body: JSON specify body with expression: `={{ {url:$json.source_url || $json.url,outputFormat:["json","html"],jsRendering:$("Validate settings").first().json.js_rendering} }}`.
    - Options: Timeout `330000`, batching enabled (batch size `1`, interval `250ms`). Set *On Error* to `Continue Regular Output`.
    - *Connection:* Connect `Split page requests` -> `Read source pages`.

16. **Pair Source Pages Code Node:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Pair source pages`. Reconnects request plans with HTTP responses.
    - *Connection:* Connect `Read source pages` -> `Pair source pages`.

17. **Extract Page Evidence via HTML Node:**
    - Add an **HTML** node (`n8n-nodes-base.html`). Name it `Extract page evidence`.
    - Operation: `Extract HTML Content`, Source Data: `json`, Data Property Name: `content`.
    - Extraction Values:
      - `text`: CSS Selector `={{ $('Validate settings').first().json.content_selector }}`, Return Value: `text`, Skip Selectors: `script, style, nav, header, footer, noscript`.
      - `title`: CSS Selector `title`, Return Value: `text`.
      - `h1`: CSS Selector `h1`, Return Value: `text`.
      - `links`: CSS Selector `a[href]`, Attribute: `href`, Return Value: `attribute`, Return Array: `true`.
    - Options: Trim values, clean up text. Set *On Error* to `Continue Regular Output`.
    - *Connection:* Connect `Pair source pages` -> `Extract page evidence`.

18. **Collect Page Evidence Code Node:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Collect page evidence`. Evaluates scrape success, checks text length ($\ge 150$ chars), and truncates content if needed.
    - *Connection:* Connect `Extract page evidence` -> `Collect page evidence`.

19. **Build Review Rows Code Node:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Build review rows`. Matches brand aliases with word boundaries, inspects outbound links for domain matches, assigns review statuses, and serializes evidence JSON objects.
    - *Connection:* Connect both `Collect page evidence` and `Keep empty page inventory` to this node (via upstream references).

20. **Save Review Queue to Google Sheets:**
    - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`). Name it `Save review queue`.
    - Set Resource to `Sheet`, Operation to `Append or Update`.
    - Set Document ID expression: `={{ $('Validate settings').first().json.spreadsheet_id }}`, Sheet Name to `Results`, Matching Columns to `result_key`.
    - Map columns:
      - `result_key`: `={{ $json.result_key }}`
      - `checked_at`: `={{ $json.checked_at }}`
      - `topic`: `={{ $json.topic }}`
      - `status`: `={{ $json.status }}`
      - `title`: `={{ $json.title }}`
      - `source_url`: `={{ $json.source_url }}`
      - `publisher`: `={{ $json.publisher }}`
      - `published_at`: `={{ $json.published_at }}`
      - `brand_excerpt`: `={{ $json.brand_excerpt }}`
      - `observed_links`: `={{ $json.observed_links }}`
      - `citing_queries`: `={{ $json.citing_queries }}`
      - `evidence_json`: `={{ $json.evidence_json }}`
    - Cell Format: `RAW`.
    - *Connection:* Connect `Build review rows` -> `Save review queue`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Verified HasData community node required for Google News SERP and web scraping requests. | Community node installation required in the n8n workspace. |
| Google Sheets setup requires two tabs (`Queries` and `Results`) with designated canvas headers. | Input sheet header: `query`. Results sheet headers: `result_key`, `checked_at`, `topic`, `status`, `title`, `source_url`, `publisher`, `published_at`, `brand_excerpt`, `observed_links`, `citing_queries`, `evidence_json`. |
| Research workflow disclaimer: Candidates represent text matches without observed direct domain links in the inspected snippet; manual verification is required before outreach. | Operational limitation & research context. |