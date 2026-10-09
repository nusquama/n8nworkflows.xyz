Compare Google SERPs between markets using HasData and Google Sheets

https://n8nworkflows.xyz/workflows/compare-google-serps-between-markets-using-hasdata-and-google-sheets-20475


# Compare Google SERPs between markets using HasData and Google Sheets

### 1. Workflow Overview

This workflow is designed for SEO editors to compare Google search engine results pages (SERPs) for paired English queries across two distinct geographic markets. The primary goal is to gather search overlap data and page text samples to evaluate whether content can be adapted into a single page or if separate localized pages are required for each market.

The workflow executes logically through the following distinct functional blocks:

- **1.1 Initialization & Configuration:** Receptions of manual triggers, setting up environment parameters (spreadsheet ID, target locations, country codes, extraction rules), and validating settings via custom JavaScript code.
- **1.2 Input Validation & Queue Prep:** Reading raw topic data from a Google Sheets "Topics" tab, validating input batches against constraints, and preparing data structures to read historical results.
- **1.3 SERP Data Collection & Comparison:** Fetching Google SERP data for both markets via the HasData API, parsing organic search results, and planning page samples using a round-robin approach.
- **1.4 Web Scraping & Content Extraction:** Filtering eligible pages, splitting requests, scraping target URLs for HTML/text content using HasData web scraping, and extracting relevant HTML content (main text, titles, H1 headings).
- **1.5 Review Queue Persistence:** Building final consolidated review rows combining search evidence, overlapping domains, and page excerpts, then appending or updating the results in a Google Sheets "Results" tab.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization & Configuration
- **Overview:** Establishes the initial configuration parameters, sets execution options, and verifies configuration integrity before calling external services.
- **Nodes Involved:** 
  - `Run manually`
  - `Settings`
  - `Validate settings`
- **Node Details:**
  - **Run manually**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Trigger)
    - *Configuration:* Default parameters.
    - *Inputs/Outputs:* No inputs; outputs to `Settings`.
    - *Edge Cases:* None.
  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Set / Edit Fields)
    - *Configuration:* Assigns workflow constants including `spreadsheet_id`, market A/B locations (`New York,New York,United States`, `London,England,United Kingdom`), country codes (`us`, `gb`), `max_items` (3), `max_pages` (6), content selectors, and JS rendering flags.
    - *Inputs/Outputs:* Input from `Run manually`; output to `Validate settings`.
    - *Edge Cases:* Placeholder strings in `spreadsheet_id` will trigger validation failures downstream.
  - **Validate settings**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Validates spreadsheet ID format, bounds numerical limits, checks canonical locations and two-letter country codes (`gl` and `gl_b`), and establishes config mode `markets`.
    - *Expressions Used:* `{{ $input.first().json }}`
    - *Inputs/Outputs:* Input from `Settings`; output to `Read input`.
    - *Edge Cases:* Throws explicit runtime errors if parameters violate structural requirements (e.g., matching country codes).

#### Block 1.2: Input Validation & Queue Prep
- **Overview:** Reads raw topic configurations from the Google Sheets "Topics" tab, validates row items and identifiers, and prepares downstream operations to query historical results.
- **Nodes Involved:**
  - `Read input`
  - `Validate input batch`
  - `Prepare queue read`
  - `Read Results`
  - `Check saved keys`
- **Node Details:**
  - **Read input**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration:* Reads sheet named `Topics` using Document ID from `Validate settings`.
    - *Inputs/Outputs:* Input from `Validate settings`; output to `Validate input batch`.
    - *Credentials Required:* Google Sheets OAuth2 / Service Account.
    - *Edge Cases:* API permission limits or missing headers (`topic_id`, `query_a`, `query_b`).
  - **Validate input batch**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Parses input rows, verifies batch size limits against `max_items`, and checks for duplicate topic identifiers.
    - *Expressions Used:* `{{ $('Validate settings').first().json.spreadsheet_id }}`
    - *Inputs/Outputs:* Input from `Read input`; output to `Prepare queue read`.
    - *Edge Cases:* Incomplete input rows or duplicate IDs throw explicit errors.
  - **Prepare queue read**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Passes execution flag object downstream.
    - *Inputs/Outputs:* Input from `Validate input batch`; output to `Read Results`.
  - **Read Results**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration:* Reads sheet named `Results` to check previously saved processing keys.
    - *Inputs/Outputs:* Input from `Prepare queue read`; output to `Check saved keys`.
    - *Credentials Required:* Google Sheets OAuth2 / Service Account.
  - **Check saved keys**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Scans existing results for duplicate `result_key` values and passes validated search configurations to the API fetch node.
    - *Inputs/Outputs:* Input from `Read Results`; output to `Fetch search evidence`.

#### Block 1.3: SERP Data Collection & Comparison
- **Overview:** Executes Google SERP searches across both markets using HasData, pairs responses, evaluates overlapping URLs/domains, and plans page sampling.
- **Nodes Involved:**
  - `Fetch search evidence`
  - `Attach search context`
  - `Plan page sample`
- **Node Details:**
  - **Fetch search evidence**
    - *Type and Technical Role:* `@hasdata/n8n-nodes-hasdata.hasData` (HasData Community Node)
    - *Configuration:* Performs `google_serp` search operation with query parameter `{{ $json.query }}` and dynamic additional fields (`deviceType`, `location`, `gl`, `hl`). Error handling set to continue regular output.
    - *Expressions Used:* `{{ $json.query }}`, `{{ $json.fields }}`
    - *Inputs/Outputs:* Input from `Check saved keys`; output to `Attach search context`.
    - *Credentials Required:* HasData API.
    - *Edge Cases:* API quota exhaustion, network timeouts, or malformed queries.
  - **Attach search context**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Maps API responses to search structures, filters organic results, and validates positional data.
    - *Inputs/Outputs:* Input from `Fetch search evidence`; output to `Plan page sample`.
  - **Plan page sample**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Executes a round-robin selection strategy across top 3 results from both markets to build a constrained list of unique target pages up to `max_pages`.
    - *Inputs/Outputs:* Input from `Attach search context`; output to `Pages to inspect`.

#### Block 1.4: Web Scraping & Content Extraction
- **Overview:** Determines whether target pages exist, scrapes page content using HasData API when applicable, extracts structured HTML elements, and aggregates evaluation metrics.
- **Nodes Involved:**
  - `Pages to inspect`
  - `Split page requests`
  - `Read source pages`
  - `Pair source pages`
  - `Extract page evidence`
  - `Collect page evidence`
  - `No eligible pages`
  - `Keep empty page inventory`
- **Node Details:**
  - **Pages to inspect**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Condition evaluator)
    - *Configuration:* Evaluates if `pages.length > 0`. Splits workflow path into scraping route (`true`) or empty inventory route (`false`).
    - *Expressions Used:* `{{ $json.pages.length > 0 }}`
    - *Inputs/Outputs:* Input from `Plan page sample`; outputs to `Split page requests` and `No eligible pages`.
  - **Split page requests**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Flattens the array of pages into individual items for parallel/batch HTTP execution.
    - *Inputs/Outputs:* Input from `Pages to inspect` (true branch); output to `Read source pages`.
  - **Read source pages**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration:* POST request to `https://api.hasdata.com/scrape/web` with batching configuration (batch size: 1, interval: 250ms), timeout (330,000ms), and pre-defined HasData credentials.
    - *Expressions Used:* `{{ {url:$json.source_url || $json.url,outputFormat:["json","html"],jsRendering:$("Validate settings").first().json.js_rendering} }}`
    - *Inputs/Outputs:* Input from `Split page requests`; output to `Pair source pages`.
    - *Credentials Required:* HasData API.
    - *Edge Cases:* HTTP 4xx/5xx errors or request timeouts on heavy pages.
  - **Pair source pages**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Re-aligns HTTP scraping responses with their original planning contexts.
    - *Inputs/Outputs:* Input from `Read source pages`; output to `Extract page evidence`.
  - **Extract page evidence**
    - *Type and Technical Role:* `n8n-nodes-base.html` (HTML Extraction)
    - *Configuration:* Extracts main body text (using content selector from settings, skipping scripts/nav/headers/footers), page title, H1 header, and anchor links.
    - *Expressions Used:* `{{ $('Validate settings').first().json.content_selector }}`
    - *Inputs/Outputs:* Input from `Pair source pages`; output to `Collect page evidence`.
    - *Edge Cases:* Missing main selectors resulting in empty text payloads.
  - **Collect page evidence**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Evaluates extraction success, truncates text excerpts securely, and normalizes metadata.
    - *Inputs/Outputs:* Input from `Extract page evidence`; output to `Build review rows`.
  - **No eligible pages**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Generates an empty page inventory payload when no pages meet inclusion criteria.
    - *Inputs/Outputs:* Input from `Pages to inspect` (false branch); output to `Keep empty page inventory`.
  - **Keep empty page inventory**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Passes empty page state downstream.
    - *Inputs/Outputs:* Input from `No eligible pages`; output to `Build review rows`.

#### Block 1.5: Review Queue Persistence
- **Overview:** Assembles all collected search data, overlapping domain metrics, and scraped page samples into structured rows for review, then writes them back to Google Sheets.
- **Nodes Involved:**
  - `Build review rows`
  - `Save review queue`
- **Node Details:**
  - **Build review rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Consolidates market search comparisons, shared URLs, shared domains, and editorial questions into row objects matching the Results schema.
    - *Inputs/Outputs:* Inputs from `Collect page evidence` and `Keep empty page inventory`; output to `Save review queue`.
  - **Save review queue**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration:* Appends or updates rows in the `Results` spreadsheet tab using `result_key` as the matching column.
    - *Expressions Used:* Row mappings for `topic`, `status`, `summary`, `checked_at`, `result_key`, `page_samples`, `evidence_json`, `market_a_query`, `market_b_query`, `shared_domains`, and `shared_url_count`. Document ID references `{{ $('Validate settings').first().json.spreadsheet_id }}`.
    - *Inputs/Outputs:* Input from `Build review rows`; no further downstream nodes.
    - *Credentials Required:* Google Sheets OAuth2 / Service Account.
    - *Edge Cases:* Google Sheets cell character limits (44,000 bytes max per JSON evidence cell) or authorization token expiration.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run manually | n8n-nodes-base.manualTrigger | Manual workflow trigger | None | Settings | |
| Settings | n8n-nodes-base.set | Assigns initial configuration parameters | Run manually | Validate settings | |
| Validate settings | n8n-nodes-base.code | Validates configuration parameters and limits | Settings | Read input | |
| Read input | n8n-nodes-base.googleSheets | Reads topic rows from Google Sheets | Validate settings | Validate input batch | |
| Validate input batch | n8n-nodes-base.code | Validates input rows against constraints | Read input | Prepare queue read | |
| Prepare queue read | n8n-nodes-base.code | Prepares execution context for historical check | Validate input batch | Read Results | |
| Read Results | n8n-nodes-base.googleSheets | Reads historical results sheet | Prepare queue read | Check saved keys | |
| Check saved keys | n8n-nodes-base.code | Validates unique result keys | Read Results | Fetch search evidence | |
| Fetch search evidence | @hasdata/n8n-nodes-hasdata.hasData | Fetches Google SERP results via HasData API | Check saved keys | Attach search context | |
| Attach search context | n8n-nodes-base.code | Parses organic results and search metadata | Fetch search evidence | Plan page sample | |
| Plan page sample | n8n-nodes-base.code | Selects unique pages across markets using round-robin | Attach search context | Pages to inspect | |
| Pages to inspect | n8n-nodes-base.if | Routes execution depending on available pages | Plan page sample | Split page requests, No eligible pages | |
| Split page requests | n8n-nodes-base.code | Flattens page inspection list | Pages to inspect | Read source pages | |
| Read source pages | n8n-nodes-base.httpRequest | Scrapes target web pages using HasData API | Split page requests | Pair source pages | |
| Pair source pages | n8n-nodes-base.code | Pairs scraping responses with planning data | Read source pages | Extract page evidence | |
| Extract page evidence | n8n-nodes-base.html | Extracts text, headings, titles, and links from HTML | Pair source pages | Collect page evidence | |
| Collect page evidence | n8n-nodes-base.code | Formats page status, metadata, and truncated excerpts | Extract page evidence | Build review rows | |
| No eligible pages | n8n-nodes-base.code | Generates empty page inventory payload | Pages to inspect | Keep empty page inventory | |
| Keep empty page inventory | n8n-nodes-base.code | Passes empty inventory downstream | No eligible pages | Build review rows | |
| Build review rows | n8n-nodes-base.code | Consolidates market comparisons and page evidence | Collect page evidence, Keep empty page inventory | Save review queue | |
| Save review queue | n8n-nodes-base.googleSheets | Appends or updates final review rows in Results sheet | Build review rows | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger:**
   - Add a **Manual Trigger** (`Run manually`) node as the entry point.
2. **Configure Workflow Settings:**
   - Add a **Set** (`Settings`) node connected to the trigger.
   - Configure assignments: `spreadsheet_id` (string), `location` (string, e.g., `New York,New York,United States`), `gl` (string, `us`), `location_b` (string, `London,England,United Kingdom`), `gl_b` (string, `gb`), `max_items` (number, `3`), `max_pages` (number, `6`), `content_selector` (string, `main, article, [role="main"]`), and `js_rendering` (boolean, `false`).
3. **Add Settings Validator:**
   - Add a **Code** (`Validate settings`) node. Paste the configuration validation and helper functions (markets, publishers, url validation, etc.) to evaluate inputs and restrict parameters.
4. **Read Input Topics:**
   - Add a **Google Sheets** (`Read input`) node. Set resource to `sheet`, operation to `read`, sheet name to `Topics`, and document ID to the expression referencing the validated spreadsheet ID.
5. **Validate Input Batch:**
   - Add a **Code** (`Validate input batch`) node to parse and validate batch row structures, checking for duplicate identifiers and max item limits.
6. **Read Historical Results:**
   - Add a **Code** (`Prepare queue read`) node followed by a **Google Sheets** (`Read Results`) node configured to read the `Results` sheet.
   - Add a **Code** (`Check saved keys`) node to verify result key uniqueness.
7. **Fetch and Process Search Evidence:**
   - Add a **HasData** (`Fetch search evidence`) node. Configure resource as `google_serp`, operation as `serp`, query to `{{ $json.query }}`, and additional fields to `{{ $json.fields }}`. Enable error continuation (`onError: continueRegularOutput`).
   - Add a **Code** (`Attach search context`) node to structure organic search results.
   - Add a **Code** (`Plan page sample`) node to execute round-robin URL sampling across markets.
8. **Configure Page Inspection Routing:**
   - Add an **If** (`Pages to inspect`) node with condition `{{ $json.pages.length > 0 }}`.
   - *True Branch:* Add a **Code** (`Split page requests`) node, then an **HTTP Request** (`Read source pages`) node configured for POST to `https://api.hasdata.com/scrape/web` using HasData API credentials and batching enabled.
   - Add a **Code** (`Pair source pages`) node, an **HTML** (`Extract page evidence`) node (extracting `text`, `title`, `h1`, and `links`), and a **Code** (`Collect page evidence`) node.
   - *False Branch:* Add a **Code** (`No eligible pages`) node connected to a **Code** (`Keep empty page inventory`) node.
9. **Build and Save Review Queue:**
   - Merge both branches into a **Code** (`Build review rows`) node to compile final comparison objects.
   - Add a **Google Sheets** (`Save review queue`) node. Set resource to `sheet`, operation to `appendOrUpdate`, matching columns to `result_key`, and map schema columns (`topic`, `status`, `summary`, `checked_at`, `result_key`, `market_a_query`, `market_b_query`, `shared_url_count`, `shared_domains`, `page_samples`, `evidence_json`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Compare search evidence before adapting content for another market | Primary workflow overview and SEO localization guidance. |
| HasData Community Node & API | Required for executing paid Google SERP and Web Scraping requests. |
| Google Sheets Integration | Requires configured OAuth2 / Service Account credentials with read/write access to Topics and Results sheets. |