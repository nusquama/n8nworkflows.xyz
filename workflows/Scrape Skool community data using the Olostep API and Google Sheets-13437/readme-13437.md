Scrape Skool community data using the Olostep API and Google Sheets

https://n8nworkflows.xyz/workflows/scrape-skool-community-data-using-the-olostep-api-and-google-sheets-13437


# Scrape Skool community data using the Olostep API and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Skool Scraper with Olostep*  
**Purpose:** Given a **Skool category URL**, this workflow collects all community “About” page URLs from that category using **Olostep Map**, batches the URLs into groups of up to **100**, runs an **Olostep batch scrape**, polls batch status until completed, retrieves each page’s HTML content, uses **Google Gemini + Information Extractor** to extract structured community fields, and appends results to **Google Sheets**.

**Typical use cases**
- Build a dataset of Skool communities in a niche (research/competitive analysis).
- Enrich community lists with structured fields (owner, membership price, members count, etc.).
- Automate ingestion of public community landing/about page data into spreadsheets.

### 1.1 Input Reception
Receives the category URL via an n8n Form Trigger.

### 1.2 Category Mapping (URL Discovery)
Olostep “map” scraping extracts a list of URLs from the category page.

### 1.3 Parse & Normalize (Community URL list)
Splits and filters URLs to keep “about” pages; normalizes the final community page URL.

### 1.4 Custom ID Generation (for Olostep batching)
Creates a numeric index array aligned to the URL list, then merges URL + custom id by position.

### 1.5 Prepare For Batching (100 URLs per batch request)
Aggregates items, converts them into a JSON array string formatted as required by Olostep batch endpoint, and iterates in batches of 100.

### 1.6 Batch Creating + Polling + Retrieval
Creates an Olostep batch scrape, waits, checks batch status repeatedly until `completed`, then iterates over batch items and retrieves full page content.

### 1.7 AI Extraction + Storage
Uses Gemini-backed Information Extractor to structure data and appends one row per community into Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Collects the Skool category URL from a user-facing form and starts the workflow.  
**Nodes involved:** `On form submission`

#### Node: On form submission
- **Type / role:** `Form Trigger` — entry point; produces form submission payload.
- **Configuration choices:**
  - Form title: **“Skool Scraper With Olostep”**
  - One field: **“Category URL”**
- **Key expressions/variables:** Downstream nodes reference `{{$json['Category URL']}}`.
- **Connections:**
  - **Output →** `Create a map`
- **Edge cases / failures:**
  - Empty or invalid URL input leads to scraping errors later (Olostep/API failure or empty map).
- **Sticky note:** “## Trigger Start the workflow manually or via form input entering a Skool category URL”

---

### Block 2 — Category Mapping (URL Discovery)
**Overview:** Uses Olostep Map scraping on the category page to extract all candidate URLs.  
**Nodes involved:** `Create a map`

#### Node: Create a map
- **Type / role:** `Olostep Scrape (resource=map)` — extracts structured map data (including URLs) from a target page.
- **Configuration choices:**
  - **URL:** `={{ $json['Category URL'] }}`
  - `top_n`: not set (null) → no explicit limit.
  - Retries: `maxTries=2`, `retryOnFail=true`
- **Credentials:** `Olostep Scrape account`
- **Connections:**
  - **Input ←** `On form submission`
  - **Output →** `Split Out1`
- **Edge cases / failures:**
  - Olostep auth/credit exhaustion.
  - Category pages that load URLs dynamically might yield incomplete URL lists depending on Olostep rendering.
  - Response schema changes could break `urls` field assumptions downstream.
- **Sticky note:** “## Olostep Map Olostep fetches and parses Skool category and gets all the community pages URLs.”

---

### Block 3 — Parse & Normalize (Community URL list)
**Overview:** Splits the URL array into items, filters for “about” URLs, and normalizes URL format.  
**Nodes involved:** `Split Out1`, `Filter`, `Edit Fields1`

#### Node: Split Out1
- **Type / role:** `Split Out` — turns an array into multiple items.
- **Configuration choices:** `fieldToSplitOut = "urls"`
- **Connections:**
  - **Input ←** `Create a map`
  - **Output →** `Filter`
- **Edge cases / failures:**
  - If `urls` is missing or not an array, node errors.

#### Node: Filter
- **Type / role:** `Filter` — keeps only URLs containing a substring.
- **Configuration choices:**
  - Condition: `{{ $json.urls }}` **contains** `"about"`
- **Connections:**
  - **Input ←** `Split Out1`
  - **Output →** `Edit Fields1`
- **Edge cases / failures:**
  - If URL field is not a string (strict validation enabled), filter may fail or drop items unexpectedly.

#### Node: Edit Fields1
- **Type / role:** `Set` — creates a normalized community URL field.
- **Configuration choices:**
  - Sets `communities` to:  
    `={{ $json.urls.split("?")[0] }}`
  - This strips query parameters.
- **Connections:**
  - **Input ←** `Filter`
  - **Output →**
    - `Aggregate` (for building URL list)
    - `Merge` (as Merge input 1)
- **Edge cases / failures:**
  - If `$json.urls` is undefined or not a string, `.split()` throws.
- **Sticky note (applies to all nodes in this block):**  
  “## Parse & Normalize Cleans raw data, filters it, and converts it into structured fields.”

---

### Block 4 — Custom ID Generation (for Olostep batching)
**Overview:** Builds an array of indices aligned with the URL list, then combines indices + URLs so each URL has a stable `custom_id` for Olostep batch processing.  
**Nodes involved:** `Aggregate`, `Edit Fields3`, `Code in JavaScript`, `Split Out`, `Merge`, `Edit Fields2`

#### Node: Aggregate
- **Type / role:** `Aggregate` — aggregates all `communities` values into a single array field.
- **Configuration choices:**
  - Aggregates `communities` → output field renamed to `url`
- **Connections:**
  - **Input ←** `Edit Fields1`
  - **Output →** `Edit Fields3`
- **Edge cases / failures:**
  - If no items pass filter, output `url` may be empty or missing.

#### Node: Edit Fields3
- **Type / role:** `Set` — computes total count of URLs.
- **Configuration choices:**
  - `total = {{ $json.url.length }}`
- **Connections:**
  - **Input ←** `Aggregate`
  - **Output →** `Code in JavaScript`
- **Edge cases / failures:**
  - If `url` is not an array, `.length` may not reflect count or could error in strict contexts.

#### Node: Code in JavaScript
- **Type / role:** `Code` — generates index array `numbers` from 0..total-1.
- **Configuration choices (logic):**
  - Reads `targetNumber = $json.total`
  - Creates `numbers = [0, 1, ..., targetNumber-1]`
  - Returns single item: `{ numbers: newArray }`
- **Connections:**
  - **Input ←** `Edit Fields3`
  - **Output →** `Split Out`
- **Edge cases / failures:**
  - If `total` is null/undefined/non-numeric, loop size may be wrong.
  - Large totals could generate large arrays (memory/time).

#### Node: Split Out
- **Type / role:** `Split Out` — converts `numbers` array to one item per number.
- **Configuration choices:** `fieldToSplitOut = "numbers"`
- **Connections:**
  - **Input ←** `Code in JavaScript`
  - **Output →** `Merge`
- **Edge cases / failures:**
  - Missing `numbers` array → error.

#### Node: Merge
- **Type / role:** `Merge (combineByPosition)` — pairs each number with the corresponding URL item by index.
- **Configuration choices:**
  - Mode: **combine**
  - Combine by: **position**
- **Connections:**
  - **Input 1 (index 0) ←** `Split Out` (numbers stream)
  - **Input 2 (index 1) ←** `Edit Fields1` (communities stream)
  - **Output →** `Edit Fields2`
- **Edge cases / failures:**
  - If counts differ, extra items may be dropped or merged incorrectly.
  - Ordering must be stable; any upstream parallelization could desync positions.

#### Node: Edit Fields2
- **Type / role:** `Set` — final per-URL record format for batching.
- **Configuration choices:**
  - `url = {{ $json.communities }}`
  - `custom_id = {{ $json.numbers }}`
- **Connections:**
  - **Input ←** `Merge`
  - **Output →** `Loop Over Items1`
- **Edge cases / failures:**
  - If merge didn’t produce `numbers` or `communities`, fields become null.
- **Sticky note (mainly for this block):**  
  “## Custom Id Generation Generates an array of numbers from 0 to the number of the URLs - 1. Then combine it with the actual list of URLs so we can use it as a custom id for the Olostep /batches endpoint.”

---

### Block 5 — Prepare For Batching (100 URLs per batch request)
**Overview:** Iterates through URL items in batches of 100, aggregates them into a single JSON array string formatted for Olostep batch creation.  
**Nodes involved:** `Loop Over Items1`, `Aggregate1`, `Edit Fields4`, `Edit Fields5`

#### Node: Loop Over Items1
- **Type / role:** `Split In Batches` — batching controller for outgoing batch creation.
- **Configuration choices:**
  - `batchSize = 100`
- **Connections:**
  - **Input ←** `Edit Fields2`
  - **Output 1 (current batch) →** (not directly connected; workflow uses Output 2 completion path)
  - **Output 2 (when no items left) →** `Aggregate1`
- **Operational note:** In this workflow, the “done/no items left” output is used to trigger aggregation steps; this pattern is unusual and can be confusing. Typically, aggregation happens per-batch, not at the end.
- **Edge cases / failures:**
  - If intended per-100 batching is required, ensure downstream aggregation runs per batch rather than only after completion (see Block 6).

#### Node: Aggregate1
- **Type / role:** `Aggregate (aggregateAllItemData)` — collects items to build an array payload.
- **Configuration choices:**
  - Aggregates **all item data** into field `test`
- **Connections:**
  - **Input ←** `Loop Over Items1` (completion output)
  - **Output →** `Edit Fields4`
- **Edge cases / failures:**
  - Can create very large payloads if it aggregates all items at once (contrary to batching intent).

#### Node: Edit Fields4
- **Type / role:** `Set` — converts item JSON into a string.
- **Configuration choices:**
  - `last = {{ $json.toJsonString() }}`
- **Connections:**
  - **Input ←** `Aggregate1`
  - **Output →** `Edit Fields5`
- **Edge cases / failures:**
  - `toJsonString()` output includes the entire item object; downstream string parsing depends on its structure.

#### Node: Edit Fields5
- **Type / role:** `Set` — extracts only the JSON array content inside square brackets.
- **Configuration choices:**
  - `last = {{ '['.concat( $json.last.split("[")[1].split(']')[0] ).concat(']') }}`
  - This is a fragile string manipulation to isolate the array portion.
- **Connections:**
  - **Input ←** `Edit Fields4`
  - **Output →** `Batch scrape urls`
- **Edge cases / failures:**
  - If the serialized string format changes, `split("[")[1]` can be undefined → runtime error.
  - Nested brackets in data could corrupt parsing.
- **Sticky note (applies to nodes in this block):**  
  “## Prepare For Batching Doing a loop to batch 100 URLs at a time not all the list at one run, and prepare the list for the Olostep batch process.”

---

### Block 6 — Batch Creating + Polling + Retrieval
**Overview:** Creates an Olostep batch scrape request, waits, polls for completion, then iterates through batch items and retrieves each page’s full content.  
**Nodes involved:** `Batch scrape urls`, `Wait`, `HTTP Request`, `If`, `Split Out2`, `Loop Over Items`, `HTTP Request1`

#### Node: Batch scrape urls
- **Type / role:** `Olostep Scrape (resource=batch)` — submits a batch scraping job.
- **Configuration choices:**
  - `formats = "json"`
  - `batch_array = {{ $json.last }}` (string expected to be JSON array)
  - Retries enabled (`maxTries=2`, `retryOnFail=true`)
- **Credentials:** `Olostep Scrape account`
- **Connections:**
  - **Input ←** `Edit Fields5`
  - **Output →** `Wait`
- **Edge cases / failures:**
  - If `batch_array` is not valid JSON, Olostep API will reject.
  - Large batches may be refused or time out.

#### Node: Wait
- **Type / role:** `Wait` — delay between creating batch and polling status.
- **Configuration choices:** wait `amount = 10` (seconds)
- **Connections:**
  - **Input ←** `Batch scrape urls` and also `If` (false branch loop)
  - **Output →** `HTTP Request`
- **Edge cases / failures:**
  - Too short may cause extra polling cycles; too long slows throughput.

#### Node: HTTP Request
- **Type / role:** `HTTP Request` — polls Olostep batch items endpoint.
- **Configuration choices:**
  - URL: `https://api.olostep.com/v1/batches/{{ $json.batch_id }}/items`
  - Header: `Authorization: Bearer <token>` (placeholder; should be stored as credential/variable)
  - Retries enabled
- **Connections:**
  - **Input ←** `Wait`
  - **Output →** `If`
- **Edge cases / failures:**
  - Hardcoded token risks leakage and expiry; prefer n8n credentials or env vars.
  - If `batch_id` missing from prior node output, URL becomes invalid.

#### Node: If
- **Type / role:** `If` — checks whether batch status is completed.
- **Configuration choices:**
  - Condition 1: `status` exists
  - Condition 2: `status == "completed"`
- **Connections:**
  - **True →** `Split Out2`
  - **False →** `Wait` (poll again)
- **Edge cases / failures:**
  - If API returns different statuses (e.g., `finished`) this loop never ends.
  - No max loop count is enforced; could run indefinitely.

#### Node: Split Out2
- **Type / role:** `Split Out` — expands `items` array from the batch response.
- **Configuration choices:** `fieldToSplitOut = "items"`
- **Connections:**
  - **Input ←** `If` (true)
  - **Output →** `Loop Over Items`
- **Edge cases / failures:**
  - Missing `items` field breaks processing.

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — iterates through batch items (poll/retrieve loop driver).
- **Configuration choices:** default batch size (not set).
- **Connections:**
  - **Input ←** `Split Out2`
  - **Output 1 →** `Loop Over Items1` (feeds back into the higher-level batch controller)
  - **Output 2 →** `HTTP Request1` (process current item by retrieving content)
- **Edge cases / failures:**
  - Wiring is non-standard: the loop’s outputs are used in a way that can be hard to reason about and may not behave as intended if modified.

#### Node: HTTP Request1
- **Type / role:** `HTTP Request` — retrieves full scraped content for a given `retrieve_id`.
- **Configuration choices:**
  - URL: `https://api.olostep.com/v1/retrieve`
  - Query: `retrieve_id = {{ $json.retrieve_id }}`
  - Header: `Authorization: Bearer <token>`
  - `onError = continueRegularOutput` (won’t stop workflow on failure)
- **Connections:**
  - **Input ←** `Loop Over Items` (item-processing output)
  - **Output →** `Information Extractor`
- **Edge cases / failures:**
  - Missing `retrieve_id` → API error.
  - Rate limits / transient errors; node continues but may pass incomplete data downstream.
- **Sticky notes:**
  - For `Batch scrape urls`, `Wait`, `HTTP Request`, `If`:  
    “## Batch Creating Creates the batch and scrapes each community page URL from the list.”
  - For `Split Out2`, `Loop Over Items`, `HTTP Request1`:  
    “## Batch Creating Parses and gets the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data…”

---

### Block 7 — AI Extraction + Storage
**Overview:** Extracts structured community attributes from HTML content using Gemini-powered extraction, then stores results in Google Sheets.  
**Nodes involved:** `Google Gemini Chat Model`, `Information Extractor`, `Append row in sheet`, `Wait1`

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain `lmChatGoogleGemini` — LLM provider for downstream extraction.
- **Configuration choices:** default options (no explicit model parameters shown).
- **Credentials:** `Gemini(PaLM) Api`
- **Connections:**
  - **AI Language Model output →** `Information Extractor` (ai_languageModel input)
- **Edge cases / failures:**
  - API quota/auth errors.
  - Model output variability can reduce extraction consistency.

#### Node: Information Extractor
- **Type / role:** LangChain `informationExtractor` — schema-based extraction from text.
- **Configuration choices:**
  - Input text: `={{ $json.html_content }}`
  - System prompt emphasizes only extracting relevant info and omitting unknowns.
  - Attributes requested (all marked required in node):  
    `page_title, description, owner, links, lpDescription, membership, num_courses, num_models, retention_video, total_admins, total_members, total_posts, total_rules, created_at, updated_at`
  - `onError = continueRegularOutput`, retries enabled
- **Connections:**
  - **Input ←** `HTTP Request1`
  - **LLM provider ←** `Google Gemini Chat Model`
  - **Output →** `Append row in sheet`
- **Edge cases / failures:**
  - If `html_content` is missing, extraction returns empty.
  - Marking attributes “required” conflicts with prompt “you may omit”; depending on node behavior/version, it may still output nulls or fail validation.
  - HTML may include noise; consider pre-cleaning to improve accuracy.

#### Node: Append row in sheet
- **Type / role:** `Google Sheets` — appends a row per community.
- **Configuration choices:**
  - Operation: **append**
  - Document: “Skool scraper” (spreadsheet ID `18D8rFRg8...`)
  - Sheet/tab: “الورقة1” (`gid=0`)
  - Column mapping uses `{{$json.output.<field>}}` from extractor result.
  - Notable mapping issue: `num_courses` mapped from `{{$json.output.num_couses}}` (typo: `num_couses` vs `num_courses`).
  - `onError = continueRegularOutput`, retries enabled
- **Credentials:** `Google Sheets account` (OAuth2)
- **Connections:**
  - **Input ←** `Information Extractor`
  - **Output →** `Wait1`
- **Edge cases / failures:**
  - Spreadsheet permission errors.
  - Column mismatch if sheet headers differ.
  - The `num_couses` typo will likely write empty values unless extractor actually outputs that misspelled key.

#### Node: Wait1
- **Type / role:** `Wait` — small delay between sheet writes / loop continuation.
- **Configuration choices:** wait `amount = 2` seconds
- **Connections:**
  - **Input ←** `Append row in sheet`
  - **Output →** `Loop Over Items` (continues processing more retrieved items)
- **Edge cases / failures:**
  - Adds throughput throttling; increase if hitting Google Sheets rate limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Collect Category URL and start workflow | — | Create a map | ## Trigger  \nStart the workflow manually or via form input entering a Skool category URL |
| Create a map | Olostep Scrape (map) | Extract all URLs from Skool category page | On form submission | Split Out1 | ## Olostep Map\nOlostep fetches and parses Skool category and gets all the community pages URLs. |
| Split Out1 | Split Out | Expand `urls` array into items | Create a map | Filter | ## Parse & Normalize\nCleans raw data, filters it, and converts it into structured fields. |
| Filter | Filter | Keep only URLs containing “about” | Split Out1 | Edit Fields1 | ## Parse & Normalize\nCleans raw data, filters it, and converts it into structured fields. |
| Edit Fields1 | Set | Normalize URL (strip query) into `communities` | Filter | Aggregate; Merge | ## Parse & Normalize\nCleans raw data, filters it, and converts it into structured fields. |
| Aggregate | Aggregate | Collect `communities` into array `url` | Edit Fields1 | Edit Fields3 | ## Custom Id Generation\nGenerates an array of numbers from 0 to the number of the URLs - 1. Then combine it with the actual list of URLs so we can use it as a custom id for the Olostep /batches endpoint. |
| Edit Fields3 | Set | Compute `total` URLs count | Aggregate | Code in JavaScript | ## Custom Id Generation\nGenerates an array of numbers from 0 to the number of the URLs - 1. Then combine it with the actual list of URLs so we can use it as a custom id for the Olostep /batches endpoint. |
| Code in JavaScript | Code | Create `numbers` array 0..total-1 | Edit Fields3 | Split Out | ## Custom Id Generation\nGenerates an array of numbers from 0 to the number of the URLs - 1. Then combine it with the actual list of URLs so we can use it as a custom id for the Olostep /batches endpoint. |
| Split Out | Split Out | Expand `numbers` into items | Code in JavaScript | Merge | ## Custom Id Generation\nGenerates an array of numbers from 0 to the number of the URLs - 1. Then combine it with the actual list of URLs so we can use it as a custom id for the Olostep /batches endpoint. |
| Merge | Merge (combineByPosition) | Pair each URL with its index | Split Out; Edit Fields1 | Edit Fields2 |  |
| Edit Fields2 | Set | Final per-URL record: `url` + `custom_id` | Merge | Loop Over Items1 |  |
| Loop Over Items1 | Split In Batches | Batch URLs (100) for payload prep | Edit Fields2 | Aggregate1 (completion output) | ## Prepare For Batching\nDoing a loop to batch 100 URLs at a time not all the list at one run, and prepare the list for the Olostep batch process. |
| Aggregate1 | Aggregate | Aggregate data to build batch array string | Loop Over Items1 | Edit Fields4 | ## Prepare For Batching\nDoing a loop to batch 100 URLs at a time not all the list at one run, and prepare the list for the Olostep batch process. |
| Edit Fields4 | Set | Convert aggregated JSON to string (`last`) | Aggregate1 | Edit Fields5 | ## Prepare For Batching\nDoing a loop to batch 100 URLs at a time not all the list at one run, and prepare the list for the Olostep batch process. |
| Edit Fields5 | Set | Extract JSON array substring into `last` | Edit Fields4 | Batch scrape urls | ## Prepare For Batching\nDoing a loop to batch 100 URLs at a time not all the list at one run, and prepare the list for the Olostep batch process. |
| Batch scrape urls | Olostep Scrape (batch) | Submit batch scrape job | Edit Fields5 | Wait | ## Batch Creating\nCreates the batch and scrapes each community page URL from the list. |
| Wait | Wait | Delay before polling batch status | Batch scrape urls; If (false) | HTTP Request | ## Batch Creating\nCreates the batch and scrapes each community page URL from the list. |
| HTTP Request | HTTP Request | Poll `/batches/{batch_id}/items` for status/items | Wait | If | ## Batch Creating\nCreates the batch and scrapes each community page URL from the list. |
| If | If | Loop until `status == completed` | HTTP Request | Split Out2 (true); Wait (false) | ## Batch Creating\nCreates the batch and scrapes each community page URL from the list. |
| Split Out2 | Split Out | Expand batch `items` array | If (true) | Loop Over Items | ## Batch Creating\nParses and gets the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data from the community that you can use (page_title, owner, membership, etc). |
| Loop Over Items | Split In Batches | Iterate batch items; retrieve content per item | Split Out2; Wait1 | Loop Over Items1; HTTP Request1 | ## Batch Creating\nParses and gets the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data from the community that you can use (page_title, owner, membership, etc). |
| HTTP Request1 | HTTP Request | Retrieve page content via `retrieve_id` | Loop Over Items | Information Extractor | ## Batch Creating\nParses and gets the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data from the community that you can use (page_title, owner, membership, etc). |
| Google Gemini Chat Model | Gemini Chat Model (LangChain) | Provides LLM for extractor | — | Information Extractor (ai_languageModel) | ## Batch Creating\nParses and gets the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data from the community that you can use (page_title, owner, membership, etc). |
| Information Extractor | Information Extractor (LangChain) | Extract structured fields from `html_content` | HTTP Request1; Gemini model | Append row in sheet | ## Batch Creating\nParses and gets the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data from the community that you can use (page_title, owner, membership, etc). |
| Append row in sheet | Google Sheets | Append extracted fields to spreadsheet | Information Extractor | Wait1 | ## Batch Creating\nParses and gets the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data from the community that you can use (page_title, owner, membership, etc). |
| Wait1 | Wait | Throttle between sheet writes and loop continuation | Append row in sheet | Loop Over Items | ## Batch Creating\nParses and gets the content of each pages in markdown. Then passes the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data from the community that you can use (page_title, owner, membership, etc). |
| Sticky Note | Sticky Note | Canvas documentation | — | — | # Skool Community Scraper Using Olostep API  \n\nThis n8n template automates scraping content from **Skool communities** using the Olostep API.  \nIt collects structured data from Skool pages and stores it in a clean format, making it easy to analyze communities, extract insights, or build datasets for research and outreach.\n\n## Who’s it for  \n- Community builders researching Skool groups  \n- Marketers analyzing competitor or niche communities  \n- SaaS founders validating ideas through community data  \n- Automation builders collecting structured social data  \n- Anyone who wants Skool data without manual scraping  \n\n## How it works / What it does  \n1. **Trigger**  \n   - The workflow starts with a manual trigger or form input containing a Skool URL or query.\n\n2. **Skool Page Scraping**  \n   - The workflow uses the Olostep API to scrape Skool community pages.  \n   - Extracts structured data using LLM-based parsing.\n\n3. **Data Extraction**  \n   - Depending on configuration, the workflow can extract:  \n     - Community name  \n     - Post titles and content  \n     - Author names  \n     - Engagement metrics (likes, comments)  \n     - URLs to posts or discussions  \n\n4. **Parse & Normalize**  \n   - The raw response is cleaned and split into individual items.  \n   - Ensures consistent fields across all scraped entries.\n\n5. **Deduplication**  \n   - Duplicate posts or entries are automatically removed.\n\n6. **Data Storage**  \n   - The final structured data is stored in a table (Google Sheets or n8n Data Table).  \n   - Ready for filtering, exporting, or further automation.\n\nThis workflow allows you to turn Skool communities into structured datasets without browser automation or manual copy/paste.\n\n## How to set up  \n1. Import the template into your n8n workspace.  \n2. Add your **Olostep API key**.  \n3. Define the Skool page or community URL you want to scrape.  \n4. Connect your storage destination (Google Sheets or Data Table).  \n5. Run the workflow and collect structured Skool data automatically.\n\n## Requirements  \n- n8n account (cloud or self-hosted)  \n- Olostep API key  \n- Google Sheets account or n8n Data Table  \n\n## How to customize the workflow  \n- Change extraction schema to capture more fields (timestamps, tags, replies).  \n- Add pagination to scrape older posts.  \n- Store data in Airtable, Notion, or a database.  \n- Trigger scraping on a schedule instead of manually.  \n- Combine with AI agents to summarize or analyze community discussions.\n\n---\n\n👉 This template makes it easy to extract, analyze, and reuse Skool community data at scale. |
| Sticky Note1 | Sticky Note | Canvas documentation | — | — | ## Trigger  \nStart the workflow manually or via form input entering a Skool category URL |
| Sticky Note2 | Sticky Note | Canvas documentation | — | — | ## Olostep Map\nOlostep fetches and parses Skool category and gets all the community pages URLs. |
| Sticky Note3 | Sticky Note | Canvas documentation | — | — | ## Parse & Normalize\nCleans raw data, filters it, and converts it into structured fields. |
| Sticky Note4 | Sticky Note | Canvas documentation | — | — | ## Custom Id Generation\nGenerates an array of numbers from 0 to the number of the URLs - 1. Then combine it with the actual list of URLs so we can use it as a custom id for the Olostep /batches endpoint. |
| Sticky Note5 | Sticky Note | (Empty) | — | — |  |
| Sticky Note6 | Sticky Note | Canvas documentation | — | — | ## Prepare For Batching\nDoing a loop to batch 100 URLs at a time not all the list at one run, and prepare the list for the Olostep batch process. |
| Sticky Note7 | Sticky Note | Canvas documentation | — | — | ## Batch Creating\nCreates the batch and scrapes each community page URL from the list. |
| Sticky Note8 | Sticky Note | Canvas documentation | — | — | ## Batch Creating\nParses and gets the content of each pages in markdown. Then passes the content of the page to an LLM to extract the most useful data from the community that you can use (page_title, owner, membership, etc). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add node **Form Trigger**  
      - Title: *Skool Scraper With Olostep*  
      - Field: *Category URL* (text)  
   2) This is the entry node.

2. **Discover URLs with Olostep Map**
   1) Add **Olostep Scrape** node (resource **Map**) named *Create a map*  
      - URL: `{{$json["Category URL"]}}`  
      - Credentials: create **Olostep Scrape API** credential (API key) and select it.
   2) Connect: **Form Trigger → Create a map**

3. **Split and filter for “about” pages**
   1) Add **Split Out** named *Split Out1*  
      - Field to split out: `urls`
   2) Add **Filter** named *Filter*  
      - Condition: `{{$json.urls}}` contains `about`
   3) Add **Set** named *Edit Fields1*  
      - Add field `communities` (string): `{{$json.urls.split("?")[0]}}`
   4) Connect: **Create a map → Split Out1 → Filter → Edit Fields1**

4. **Aggregate URLs and generate matching numeric IDs**
   1) Add **Aggregate** named *Aggregate*  
      - Aggregate field: `communities`  
      - Rename output field to: `url` (array)
   2) Add **Set** named *Edit Fields3*  
      - Field `total` (number): `{{$json.url.length}}`
   3) Add **Code** node named *Code in JavaScript* with code generating `numbers` array 0..total-1 (same logic as provided).
   4) Add **Split Out** named *Split Out*  
      - Field to split out: `numbers`
   5) Add **Merge** named *Merge*  
      - Mode: **Combine**  
      - Combine by: **Position**
   6) Add **Set** named *Edit Fields2*  
      - `url` = `{{$json.communities}}`  
      - `custom_id` = `{{$json.numbers}}`
   7) Connections:
      - **Edit Fields1 → Aggregate → Edit Fields3 → Code → Split Out → Merge (Input 1)**
      - **Edit Fields1 → Merge (Input 2)**
      - **Merge → Edit Fields2**

5. **Batch preparation (target: 100 URLs per Olostep batch)**
   1) Add **Split In Batches** named *Loop Over Items1*  
      - Batch size: `100`
   2) Add **Aggregate** named *Aggregate1*  
      - Mode: “Aggregate All Item Data”  
      - Destination field: `test`
   3) Add **Set** named *Edit Fields4*  
      - `last` (string): `{{$json.toJsonString()}}`
   4) Add **Set** named *Edit Fields5*  
      - `last` (string): `{{ '['.concat($json.last.split('[')[1].split(']')[0]).concat(']') }}`
   5) Connect: **Edit Fields2 → Loop Over Items1 → (done output) Aggregate1 → Edit Fields4 → Edit Fields5**

   > Important: If you truly need *per-100* batching, adjust wiring so `Aggregate1/Edit Fields4/Edit Fields5` runs for each batch, not only after completion.

6. **Create Olostep batch scrape**
   1) Add **Olostep Scrape** node (resource **Batch**) named *Batch scrape urls*  
      - Formats: `json`  
      - Batch array: `{{$json.last}}`  
      - Select Olostep credentials.
   2) Connect: **Edit Fields5 → Batch scrape urls**

7. **Polling for completion**
   1) Add **Wait** named *Wait* (10 seconds)
   2) Add **HTTP Request** named *HTTP Request*  
      - GET `https://api.olostep.com/v1/batches/{{$json.batch_id}}/items`  
      - Header `Authorization: Bearer <token>`  
      - Prefer storing token as an n8n credential or environment variable rather than inline.
   3) Add **If** node named *If*  
      - Condition: `status` exists AND equals `completed`
   4) Connect:
      - **Batch scrape urls → Wait → HTTP Request → If**
      - **If false → Wait** (loop)

8. **Iterate items and retrieve page content**
   1) Add **Split Out** named *Split Out2*  
      - Field to split out: `items`
   2) Add **Split In Batches** named *Loop Over Items*
   3) Add **HTTP Request** named *HTTP Request1*  
      - GET `https://api.olostep.com/v1/retrieve?retrieve_id={{$json.retrieve_id}}`  
      - Header `Authorization: Bearer <token>`  
      - Set **On Error**: “Continue (regular output)”
   4) Connect:
      - **If true → Split Out2 → Loop Over Items → HTTP Request1**

9. **LLM setup (Gemini) and structured extraction**
   1) Add **Google Gemini Chat Model** node  
      - Configure **Google PaLM/Gemini** credentials.
   2) Add **Information Extractor** node  
      - Text: `{{$json.html_content}}`  
      - Define attributes listed in the workflow (page_title, description, owner, etc.)  
      - Connect Gemini node to the extractor via the **ai_languageModel** connection.
   3) Connect: **HTTP Request1 → Information Extractor** and **Gemini → Information Extractor (ai_languageModel)**

10. **Write to Google Sheets**
   1) Add **Google Sheets** node named *Append row in sheet*  
      - Operation: Append  
      - Authenticate via Google Sheets OAuth2 credentials  
      - Select the target document and sheet  
      - Map columns from `{{$json.output.<field>}}`
      - Fix/verify field names (notably `num_courses` is mapped from `num_couses` in the provided workflow; correct if needed).
   2) Add **Wait** node named *Wait1* (2 seconds) to reduce rate limit risk.
   3) Connect: **Information Extractor → Append row in sheet → Wait1 → Loop Over Items** (to continue processing next item)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Template description and customization ideas (change extraction schema, pagination, store to Airtable/Notion/DB, schedule trigger, combine with AI agents). | From main canvas sticky note content (Skool Community Scraper Using Olostep API). |
| Authorization header uses `Bearer <token>` directly in HTTP Request nodes. Replace with a secure credential/variable. | Applies to `HTTP Request` and `HTTP Request1`. |
| Potential mapping typo: Google Sheets `num_courses` uses `{{$json.output.num_couses}}`. | Check and correct to `num_courses` if required. |
| Polling loop has no maximum retries/timebox; batch statuses that never become `completed` can loop indefinitely. | Add a counter/time limit if running in production. |