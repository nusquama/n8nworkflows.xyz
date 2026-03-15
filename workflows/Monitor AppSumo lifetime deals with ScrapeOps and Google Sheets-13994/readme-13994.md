Monitor AppSumo lifetime deals with ScrapeOps and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-appsumo-lifetime-deals-with-scrapeops-and-google-sheets-13994


# Monitor AppSumo lifetime deals with ScrapeOps and Google Sheets

# 1. Workflow Overview

This workflow monitors AppSumo lifetime deals once per day, extracts structured deal data from the AppSumo browse page using ScrapeOps with JavaScript rendering, filters the results to relevant categories, and synchronizes the data into Google Sheets.

Its primary use cases are:

- Tracking new AppSumo lifetime deals automatically
- Maintaining a spreadsheet of selected SaaS deals over time
- Refreshing existing deal rows with updated pricing, status, and metadata
- Supporting downstream alerting or analytics based on the Google Sheet

The workflow is organized into five logical blocks.

## 1.1 Trigger & Page Retrieval

The workflow starts on a daily schedule at 09:00 and fetches the AppSumo browse page for lifetime deals through ScrapeOps, with JavaScript rendering enabled so dynamically rendered content can be captured.

## 1.2 HTML Parsing

The raw HTML response is parsed in a Code node into structured deal objects. This block extracts product name, URL, category, price information, discount, review data, image, status, and a scrape timestamp.

## 1.3 Deal Relevance Filtering

The parsed deals are filtered to keep only those that match a configurable list of target categories or keywords such as No-Code, Automation, AI, Productivity, and Marketing.

## 1.4 Sheet Lookup & Deduplication

Each relevant deal is looked up in Google Sheets by its AppSumo URL. The workflow then merges the scraped deal data with the matching row, if one exists, so it can determine whether the deal is new or already tracked.

## 1.5 Routing & Sheet Writeback

A conditional node checks whether the merged record has an existing `Last Checked` value. If not, the workflow treats it as a new deal and appends it to the sheet. Otherwise, it updates the existing row in place after refreshing the `Last Checked` timestamp.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Page Retrieval

### Overview

This block is responsible for starting the workflow every day and retrieving the AppSumo browse page content. It uses ScrapeOps with JS rendering so the downstream parser receives a more complete HTML snapshot.

### Nodes Involved

- Daily Schedule Trigger (09:00)
- Fetch AppSumo Deals via ScrapeOps (JS Render)

### Node Details

#### Daily Schedule Trigger (09:00)

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point
- **Configuration choices:**
  - Uses a cron expression: `0 9 * * *`
  - Runs daily at 09:00 based on the n8n instance timezone
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input
  - Output to `Fetch AppSumo Deals via ScrapeOps (JS Render)`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`
  - Cron behavior depends on the instance timezone configuration
- **Edge cases or potential failure types:**
  - Unexpected execution hour if server timezone differs from expected local timezone
  - Trigger does not fire if workflow is inactive
- **Sub-workflow reference:** None

#### Fetch AppSumo Deals via ScrapeOps (JS Render)

- **Type and technical role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`; external page fetch through ScrapeOps proxy/scraping API
- **Configuration choices:**
  - URL set to `https://appsumo.com/browse/?tags=lifetime-deal`
  - Advanced option `render_js: true` enabled
  - This indicates the workflow expects client-side rendered content
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Daily Schedule Trigger (09:00)`
  - Output to `Parse AppSumo Deals HTML → Structured JSON`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires ScrapeOps node package installed in the n8n environment
  - Requires ScrapeOps credentials configured in n8n
- **Edge cases or potential failure types:**
  - Invalid or missing ScrapeOps credentials
  - Rate limits or account quota exhaustion
  - JS rendering timeout
  - AppSumo anti-bot changes or structure changes
  - Unexpected payload format returned by the ScrapeOps node
- **Sub-workflow reference:** None

---

## 2.2 HTML Parsing

### Overview

This block transforms the fetched HTML into structured JSON records. It uses defensive extraction logic to handle multiple possible response shapes from ScrapeOps and attempts to parse deal cards with regular expressions and string splitting.

### Nodes Involved

- Parse AppSumo Deals HTML → Structured JSON

### Node Details

#### Parse AppSumo Deals HTML → Structured JSON

- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript parser
- **Configuration choices:**
  - Reads the first incoming item and attempts to locate HTML in several possible properties:
    - direct string item
    - `response.body`
    - `body`
    - `content`
    - `data`
    - array joined as string
    - JSON string fallback if HTML markers are present
  - Splits HTML by a regex targeting deal card containers using a class pattern matching `relative h-full`
  - Extracts:
    - `Product Name`
    - `AppSumo URL`
    - `Category`
    - `Current Price`
    - `Last Price`
    - `Discount`
    - `Deal Status`
    - `Rating`
    - `Reviews Count`
    - `Image`
    - `scraped_at`
  - Computes discount when current and original price are both parseable
  - If no deals are found, returns one debug item instead of an empty dataset
- **Key expressions or variables used:**
  - `items[0].json`
  - `new Date().toISOString()`
  - Regex extraction against HTML chunks
- **Input and output connections:**
  - Input from `Fetch AppSumo Deals via ScrapeOps (JS Render)`
  - Output to `Filter Relevant Deal Categories`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Runs JavaScript in the n8n Code node runtime
- **Edge cases or potential failure types:**
  - HTML structure changes on AppSumo can break selectors and regex patterns
  - If ScrapeOps returns an unexpected structure, parsing may fail silently and produce only a debug item
  - The parser assumes `items[0]` contains the relevant payload; if multiple fetch items are present, only the first is processed
  - The split regex may match too broadly or too narrowly depending on class name changes
  - A debug item without `Category` or `AppSumo URL` will continue downstream and may be filtered out or produce confusing behavior
- **Sub-workflow reference:** None

---

## 2.3 Deal Relevance Filtering

### Overview

This block narrows the scraped dataset to deals relevant to the user’s interests. It checks both the `Category` and the `Product Name` against a configurable keyword list.

### Nodes Involved

- Filter Relevant Deal Categories

### Node Details

#### Filter Relevant Deal Categories

- **Type and technical role:** `n8n-nodes-base.code`; in-memory filter
- **Configuration choices:**
  - Defines target categories:
    - `No-Code`
    - `Automation`
    - `AI`
    - `Productivity`
    - `Marketing`
  - For each item, checks whether either:
    - `Category` contains one of the target strings, or
    - `Product Name` contains one of the target strings
  - Comparison is case-insensitive
- **Key expressions or variables used:**
  - `item.json['Category']`
  - `item.json['Product Name']`
  - `targetCategories.some(...)`
- **Input and output connections:**
  - Input from `Parse AppSumo Deals HTML → Structured JSON`
  - Outputs to:
    - `Lookup Deal in Google Sheet (by URL)`
    - `Merge Scrape + Existing Row (Enrich by URL)` as input 1
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Deals with missing category may still pass if the title matches
  - Relevant deals may be excluded if AppSumo categories use different naming
  - The keyword `AI` is very broad and may match unintended titles
  - If the parser outputs a debug item instead of deals, it will likely be filtered out because the expected fields are absent
- **Sub-workflow reference:** None

---

## 2.4 Sheet Lookup & Deduplication

### Overview

This block determines whether each filtered deal already exists in Google Sheets by looking it up on `AppSumo URL`. The merge node enriches the scraped item with data from the matching spreadsheet row so later logic can identify new versus existing deals.

### Nodes Involved

- Lookup Deal in Google Sheet (by URL)
- Merge Scrape + Existing Row (Enrich by URL)

### Node Details

#### Lookup Deal in Google Sheet (by URL)

- **Type and technical role:** `n8n-nodes-base.googleSheets`; row lookup in Google Sheets
- **Configuration choices:**
  - Uses the spreadsheet:
    - `https://docs.google.com/spreadsheets/d/1L_Fn_wm55wc5EXi_ZIVmmXyFrz9nJ-29Xd71Ndz_XIk/edit?usp=sharing`
  - Targets sheet `Sheet1` (`gid=0`)
  - Filters rows where:
    - `AppSumo URL` equals `{{$json["AppSumo URL"]}}`
  - `continueOnFail: true` enabled
- **Key expressions or variables used:**
  - `={{ $json["AppSumo URL"] }}`
- **Input and output connections:**
  - Input from `Filter Relevant Deal Categories`
  - Output to `Merge Scrape + Existing Row (Enrich by URL)` as input 2
- **Version-specific requirements:**
  - Uses `typeVersion: 4`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - OAuth token expired or insufficient permissions
  - Spreadsheet URL invalid or sheet missing
  - Column name mismatch for `AppSumo URL`
  - If lookup fails but `continueOnFail` is active, downstream merge behavior may be harder to diagnose
  - Duplicate rows in the sheet may produce multiple matches, though downstream merge is configured to use the first
- **Sub-workflow reference:** None

#### Merge Scrape + Existing Row (Enrich by URL)

- **Type and technical role:** `n8n-nodes-base.merge`; combines scraped records with sheet lookup results
- **Configuration choices:**
  - Mode: `combine`
  - Join mode: `enrichInput1`
  - Merge field:
    - Input 1 field: `AppSumo URL`
    - Input 2 field: `AppSumo URL`
  - Options:
    - `fuzzyCompare: false`
    - `multipleMatches: first`
    - `disableDotNotation: false`
  - This means the filtered scrape item is preserved and enriched with matching Google Sheet fields if present
- **Key expressions or variables used:** Field-based join on `AppSumo URL`
- **Input and output connections:**
  - Input 1 from `Filter Relevant Deal Categories`
  - Input 2 from `Lookup Deal in Google Sheet (by URL)`
  - Output to `IF New Deal (Last Checked is empty)`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.1`
- **Edge cases or potential failure types:**
  - Join will fail to enrich if URLs differ in formatting
  - If the lookup yields no row, the scraped item continues without existing-sheet fields
  - If multiple rows match the same URL, only the first is used
  - If `AppSumo URL` is missing in the scraped item, enrichment will not work
- **Sub-workflow reference:** None

---

## 2.5 Routing & Sheet Writeback

### Overview

This block determines whether the enriched deal is new or existing, stamps a fresh `Last Checked` timestamp, and writes the record to Google Sheets via append or update operations.

### Nodes Involved

- IF New Deal (Last Checked is empty)
- Stamp Last Checked (New Deal Path)
- Stamp Last Checked (Existing Deal Path)
- Append New Deal to Sheet
- Update Existing Deal Row in Sheet

### Node Details

#### IF New Deal (Last Checked is empty)

- **Type and technical role:** `n8n-nodes-base.if`; branching based on field presence
- **Configuration choices:**
  - Tests whether `{{$json['Last Checked']}}` is empty
  - True branch means the merged item did not contain an existing `Last Checked` value, so it is treated as a new deal
  - False branch means an existing row was found
- **Key expressions or variables used:**
  - `={{ $json['Last Checked'] }}`
- **Input and output connections:**
  - Input from `Merge Scrape + Existing Row (Enrich by URL)`
  - True output to `Stamp Last Checked (New Deal Path)`
  - False output to `Stamp Last Checked (Existing Deal Path)`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - This logic assumes existing rows always contain `Last Checked`
  - If a row exists in the sheet but the `Last Checked` cell is empty, it will be misclassified as new and appended again
  - If sheet schema changes and the field is renamed, routing breaks
- **Sub-workflow reference:** None

#### Stamp Last Checked (New Deal Path)

- **Type and technical role:** `n8n-nodes-base.code`; timestamp enrichment for new items
- **Configuration choices:**
  - Iterates over all items
  - Copies all existing fields and sets:
    - `Last Checked = new Date().toISOString()`
- **Key expressions or variables used:**
  - `new Date().toISOString()`
  - spread syntax `...item.json`
- **Input and output connections:**
  - Input from `IF New Deal (Last Checked is empty)` true branch
  - Output to `Append New Deal to Sheet`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Time is written in ISO UTC format, which may differ from spreadsheet locale expectations
- **Sub-workflow reference:** None

#### Stamp Last Checked (Existing Deal Path)

- **Type and technical role:** `n8n-nodes-base.code`; timestamp refresh for existing items
- **Configuration choices:**
  - Iterates over all items
  - Copies all existing fields and updates:
    - `Last Checked = new Date().toISOString()`
- **Key expressions or variables used:**
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input from `IF New Deal (Last Checked is empty)` false branch
  - Output to `Update Existing Deal Row in Sheet`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Same timestamp formatting caveat as above
- **Sub-workflow reference:** None

#### Append New Deal to Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`; append new rows
- **Configuration choices:**
  - Operation: `append`
  - Writes to `Sheet1` in the same spreadsheet
  - Uses explicit field mapping for:
    - `Product Name`
    - `AppSumo URL`
    - `Category`
    - `Last Price`
    - `Current Price`
    - `Discount`
    - `Deal Status`
    - `Rating`
    - `Reviews Count`
    - `Last Checked`
    - `Image`
  - Mapping mode is manually defined
- **Key expressions or variables used:**
  - `={{ $json['...'] }}`
- **Input and output connections:**
  - Input from `Stamp Last Checked (New Deal Path)`
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion: 4`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Missing spreadsheet permissions
  - Column names in the sheet must match configured schema
  - Appending can create duplicates if new/existing classification is wrong
  - `scraped_at` is not written, even though it exists upstream
- **Sub-workflow reference:** None

#### Update Existing Deal Row in Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates an existing row by matching column
- **Configuration choices:**
  - Operation: `update`
  - Writes to the same spreadsheet and sheet
  - Matching columns:
    - `AppSumo URL`
  - Updates the same mapped business fields as the append node
  - The schema includes `row_number` as a removed/read-only field but matching actually relies on `AppSumo URL`
- **Key expressions or variables used:**
  - `={{ $json['...'] }}`
- **Input and output connections:**
  - Input from `Stamp Last Checked (Existing Deal Path)`
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion: 4`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - If multiple rows share the same `AppSumo URL`, update behavior may be ambiguous
  - If the target column header changes, updates stop matching
  - Existing sheet rows missing `AppSumo URL` cannot be updated correctly
- **Sub-workflow reference:** None

---

## 2.6 Documentation / Visual Annotation Nodes

These nodes do not affect execution but provide important inline documentation for human users.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation
- **Configuration choices:**
  - Large overview note describing purpose, setup, and customization
  - Includes links to:
    - ScrapeOps registration: https://scrapeops.io/app/register/n8n
    - ScrapeOps docs: https://scrapeops.io/docs/n8n/overview/
    - Google Sheet duplication link: https://docs.google.com/spreadsheets/d/1L_Fn_wm55wc5EXi_ZIVmmXyFrz9nJ-29Xd71Ndz_XIk/edit#gid=0
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation
- **Configuration choices:**
  - Describes trigger and fetch stage
  - Includes link to ScrapeOps Proxy docs: https://scrapeops.io/docs/n8n/proxy-api/
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation
- **Configuration choices:**
  - Describes parse stage
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation
- **Configuration choices:**
  - Describes category filtering stage
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation
- **Configuration choices:**
  - Describes deduplication and merge stage
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note5

- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation
- **Configuration choices:**
  - Describes route and writeback stage
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Schedule Trigger (09:00) | Schedule Trigger | Starts workflow daily at 09:00 |  | Fetch AppSumo Deals via ScrapeOps (JS Render) | ## 1. Trigger & Fetch<br>Runs daily at 09:00 and loads the AppSumo browse page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with JavaScript rendering enabled. |
| Fetch AppSumo Deals via ScrapeOps (JS Render) | ScrapeOps | Fetches AppSumo browse page with JS rendering | Daily Schedule Trigger (09:00) | Parse AppSumo Deals HTML → Structured JSON | ## 1. Trigger & Fetch<br>Runs daily at 09:00 and loads the AppSumo browse page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with JavaScript rendering enabled. |
| Parse AppSumo Deals HTML → Structured JSON | Code | Parses HTML into structured deal records | Fetch AppSumo Deals via ScrapeOps (JS Render) | Filter Relevant Deal Categories | ## 2. Parse Deals<br>Extracts title, URL, prices, discount, category, rating, reviews, image, status, and timestamps from the raw HTML into clean structured JSON. |
| Filter Relevant Deal Categories | Code | Filters parsed deals by target categories/keywords | Parse AppSumo Deals HTML → Structured JSON | Lookup Deal in Google Sheet (by URL); Merge Scrape + Existing Row (Enrich by URL) | ## 3. Filter by Category<br>Keeps only deals that match your target categories or keywords. Edit the list inside this node to customize what gets tracked. |
| Lookup Deal in Google Sheet (by URL) | Google Sheets | Searches existing sheet rows by AppSumo URL | Filter Relevant Deal Categories | Merge Scrape + Existing Row (Enrich by URL) | ## 4. Deduplicate & Merge<br>Looks up each deal by its AppSumo URL in Google Sheets, then merges scraped data with any existing row to enrich the record. |
| Merge Scrape + Existing Row (Enrich by URL) | Merge | Enriches scraped deals with existing sheet row data | Filter Relevant Deal Categories; Lookup Deal in Google Sheet (by URL) | IF New Deal (Last Checked is empty) | ## 4. Deduplicate & Merge<br>Looks up each deal by its AppSumo URL in Google Sheets, then merges scraped data with any existing row to enrich the record. |
| IF New Deal (Last Checked is empty) | IF | Routes items into new vs existing paths | Merge Scrape + Existing Row (Enrich by URL) | Stamp Last Checked (New Deal Path); Stamp Last Checked (Existing Deal Path) | ## 5. Route & Write to Google Sheets<br>New deals are appended as fresh rows. Existing deals have their price, status, and timestamps updated in place. |
| Stamp Last Checked (New Deal Path) | Code | Adds current timestamp before append | IF New Deal (Last Checked is empty) | Append New Deal to Sheet | ## 5. Route & Write to Google Sheets<br>New deals are appended as fresh rows. Existing deals have their price, status, and timestamps updated in place. |
| Stamp Last Checked (Existing Deal Path) | Code | Refreshes timestamp before update | IF New Deal (Last Checked is empty) | Update Existing Deal Row in Sheet | ## 5. Route & Write to Google Sheets<br>New deals are appended as fresh rows. Existing deals have their price, status, and timestamps updated in place. |
| Append New Deal to Sheet | Google Sheets | Appends new deal rows to sheet | Stamp Last Checked (New Deal Path) |  | ## 5. Route & Write to Google Sheets<br>New deals are appended as fresh rows. Existing deals have their price, status, and timestamps updated in place. |
| Update Existing Deal Row in Sheet | Google Sheets | Updates existing deal row by AppSumo URL | Stamp Last Checked (Existing Deal Path) |  | ## 5. Route & Write to Google Sheets<br>New deals are appended as fresh rows. Existing deals have their price, status, and timestamps updated in place. |
| Sticky Note | Sticky Note | General workflow documentation on canvas |  |  | # 🛍️ AppSumo Lifetime Deal Monitor → Google Sheets<br>This workflow automatically tracks AppSumo lifetime deals daily. It fetches the AppSumo browse page using **ScrapeOps Proxy with JS rendering**, parses each deal into structured data, filters by your target categories, and saves new or updated deals to Google Sheets with full deduplication.<br><br>### How it works<br>1. ⏰ **Daily Schedule Trigger** fires every day at 09:00 automatically.<br>2. 🌐 **Fetch AppSumo Deals via ScrapeOps** loads the browse page with JavaScript rendering enabled.<br>3. 🔍 **Parse AppSumo Deals HTML → Structured JSON** extracts name, URL, category, prices, discount, rating, reviews, image, and timestamps.<br>4. 🔎 **Filter Relevant Deal Categories** keeps only deals matching your target keywords.<br>5. 📋 **Lookup Deal in Google Sheet (by URL)** checks if the deal already exists in your sheet.<br>6. 🔀 **Merge Scrape + Existing Row** combines scraped data with any existing sheet row.<br>7. 🔀 **IF New Deal** routes new vs. existing deals to separate write paths.<br>8. 💾 **Append New Deal to Sheet** adds a fresh row for new deals.<br>9. 🔄 **Update Existing Deal Row** refreshes pricing and status for known deals.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- [Duplicate the Google Sheet](https://docs.google.com/spreadsheets/d/1L_Fn_wm55wc5EXi_ZIVmmXyFrz9nJ-29Xd71Ndz_XIk/edit#gid=0) and paste your Sheet URL into the Lookup, Append, and Update nodes.<br>- Edit the category/keyword list in **Filter Relevant Deal Categories** to match your interests.<br>- Run once manually to confirm, then activate.<br><br>### Customization<br>- Add Slack, Gmail, or Discord alerts on the **New Deal** path.<br>- Change the schedule time in the trigger node to suit your timezone. |
| Sticky Note1 | Sticky Note | Annotation for trigger/fetch block |  |  | ## 1. Trigger & Fetch<br>Runs daily at 09:00 and loads the AppSumo browse page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with JavaScript rendering enabled. |
| Sticky Note2 | Sticky Note | Annotation for parsing block |  |  | ## 2. Parse Deals<br>Extracts title, URL, prices, discount, category, rating, reviews, image, status, and timestamps from the raw HTML into clean structured JSON. |
| Sticky Note3 | Sticky Note | Annotation for filtering block |  |  | ## 3. Filter by Category<br>Keeps only deals that match your target categories or keywords. Edit the list inside this node to customize what gets tracked. |
| Sticky Note4 | Sticky Note | Annotation for deduplication block |  |  | ## 4. Deduplicate & Merge<br>Looks up each deal by its AppSumo URL in Google Sheets, then merges scraped data with any existing row to enrich the record. |
| Sticky Note5 | Sticky Note | Annotation for writeback block |  |  | ## 5. Route & Write to Google Sheets<br>New deals are appended as fresh rows. Existing deals have their price, status, and timestamps updated in place. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `AppSumo Lifetime Deal Monitor with ScrapeOps and Google Sheets`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name it: `Daily Schedule Trigger (09:00)`
   - Configure:
     - Rule type: cron expression
     - Expression: `0 9 * * *`
   - Confirm your n8n instance timezone is the one you expect.

3. **Add the ScrapeOps node**
   - Node type: `ScrapeOps`
   - Name it: `Fetch AppSumo Deals via ScrapeOps (JS Render)`
   - Set URL to:
     - `https://appsumo.com/browse/?tags=lifetime-deal`
   - In advanced options, enable:
     - `render_js = true`
   - Create or select ScrapeOps credentials.
     - Register if needed: `https://scrapeops.io/app/register/n8n`
     - ScrapeOps n8n docs: `https://scrapeops.io/docs/n8n/overview/`

4. **Connect the trigger to the ScrapeOps node**
   - `Daily Schedule Trigger (09:00)` → `Fetch AppSumo Deals via ScrapeOps (JS Render)`

5. **Add a Code node for parsing**
   - Node type: `Code`
   - Name it: `Parse AppSumo Deals HTML → Structured JSON`
   - Paste JavaScript that:
     - Reads the incoming ScrapeOps payload
     - Detects HTML from possible keys like `response.body`, `body`, `content`, `data`
     - Splits the HTML into deal card chunks
     - Extracts the following fields:
       - `Product Name`
       - `AppSumo URL`
       - `Category`
       - `Current Price`
       - `Last Price`
       - `Discount`
       - `Deal Status`
       - `Rating`
       - `Reviews Count`
       - `Image`
       - `scraped_at`
     - Returns one item per deal
     - Returns a debug item if no deals are found
   - Use the same extraction logic as in the provided workflow, including the regex-based parsing.

6. **Connect ScrapeOps to the parser**
   - `Fetch AppSumo Deals via ScrapeOps (JS Render)` → `Parse AppSumo Deals HTML → Structured JSON`

7. **Add a Code node for filtering**
   - Node type: `Code`
   - Name it: `Filter Relevant Deal Categories`
   - Add JavaScript similar to:
     - Define `targetCategories = ['No-Code', 'Automation', 'AI', 'Productivity', 'Marketing']`
     - Keep items where `Category` or `Product Name` contains any target term, case-insensitively
   - This is the main customization point for what you want to track.

8. **Connect parser to filter**
   - `Parse AppSumo Deals HTML → Structured JSON` → `Filter Relevant Deal Categories`

9. **Prepare Google Sheets**
   - Create or duplicate a spreadsheet
   - Required sheet: `Sheet1` or any sheet you choose
   - Required columns:
     - `Product Name`
     - `AppSumo URL`
     - `Category`
     - `Last Price`
     - `Current Price`
     - `Discount`
     - `Deal Status`
     - `Rating`
     - `Reviews Count`
     - `Last Checked`
     - `Image`
   - Best practice:
     - Ensure `AppSumo URL` values are unique
     - Ensure existing rows have `Last Checked` populated to avoid misclassification as new

10. **Add a Google Sheets lookup node**
    - Node type: `Google Sheets`
    - Name it: `Lookup Deal in Google Sheet (by URL)`
    - Select Google Sheets OAuth2 credentials
    - Choose your spreadsheet by URL
    - Select the target sheet
    - Configure lookup/filter:
      - Column: `AppSumo URL`
      - Value: `={{ $json["AppSumo URL"] }}`
    - Enable `Continue On Fail`
    - Purpose: retrieve existing rows for matching deals

11. **Add a Merge node**
    - Node type: `Merge`
    - Name it: `Merge Scrape + Existing Row (Enrich by URL)`
    - Set:
      - Mode: `Combine`
      - Join mode: `Enrich Input 1`
      - Match field Input 1: `AppSumo URL`
      - Match field Input 2: `AppSumo URL`
      - Multiple matches: `First`
      - Fuzzy compare: off

12. **Connect filter to lookup and merge**
    - `Filter Relevant Deal Categories` → `Lookup Deal in Google Sheet (by URL)`
    - `Filter Relevant Deal Categories` → `Merge Scrape + Existing Row (Enrich by URL)` as Input 1
    - `Lookup Deal in Google Sheet (by URL)` → `Merge Scrape + Existing Row (Enrich by URL)` as Input 2

13. **Add an IF node**
    - Node type: `IF`
    - Name it: `IF New Deal (Last Checked is empty)`
    - Condition:
      - Type: string
      - Value 1: `={{ $json['Last Checked'] }}`
      - Operation: `is empty`
    - True branch = new deal
    - False branch = existing deal

14. **Connect merge to IF**
    - `Merge Scrape + Existing Row (Enrich by URL)` → `IF New Deal (Last Checked is empty)`

15. **Add Code node for new deals**
    - Node type: `Code`
    - Name it: `Stamp Last Checked (New Deal Path)`
    - JavaScript behavior:
      - Loop over all items
      - Return same fields plus:
        - `'Last Checked': new Date().toISOString()`

16. **Add Code node for existing deals**
    - Node type: `Code`
    - Name it: `Stamp Last Checked (Existing Deal Path)`
    - Use the same timestamping logic:
      - `'Last Checked': new Date().toISOString()`

17. **Connect IF branches**
    - True output of `IF New Deal (Last Checked is empty)` → `Stamp Last Checked (New Deal Path)`
    - False output of `IF New Deal (Last Checked is empty)` → `Stamp Last Checked (Existing Deal Path)`

18. **Add Google Sheets append node**
    - Node type: `Google Sheets`
    - Name it: `Append New Deal to Sheet`
    - Operation: `Append`
    - Select the same spreadsheet and sheet
    - Use manual column mapping
    - Map:
      - `Product Name` ← `={{ $json['Product Name'] }}`
      - `AppSumo URL` ← `={{ $json['AppSumo URL'] }}`
      - `Category` ← `={{ $json['Category'] }}`
      - `Last Price` ← `={{ $json['Last Price'] }}`
      - `Current Price` ← `={{ $json['Current Price'] }}`
      - `Discount` ← `={{ $json['Discount'] }}`
      - `Deal Status` ← `={{ $json['Deal Status'] }}`
      - `Rating` ← `={{ $json['Rating'] }}`
      - `Reviews Count` ← `={{ $json['Reviews Count'] }}`
      - `Last Checked` ← `={{ $json['Last Checked'] }}`
      - `Image` ← `={{ $json['Image'] }}`

19. **Connect new deal stamp to append**
    - `Stamp Last Checked (New Deal Path)` → `Append New Deal to Sheet`

20. **Add Google Sheets update node**
    - Node type: `Google Sheets`
    - Name it: `Update Existing Deal Row in Sheet`
    - Operation: `Update`
    - Select the same spreadsheet and sheet
    - Matching column:
      - `AppSumo URL`
    - Use manual column mapping for the same fields as the append node:
      - `Product Name`
      - `AppSumo URL`
      - `Category`
      - `Last Price`
      - `Current Price`
      - `Discount`
      - `Deal Status`
      - `Rating`
      - `Reviews Count`
      - `Last Checked`
      - `Image`

21. **Connect existing deal stamp to update**
    - `Stamp Last Checked (Existing Deal Path)` → `Update Existing Deal Row in Sheet`

22. **Optionally add sticky notes**
    - Add one large general note documenting purpose and setup
    - Add smaller notes above each logical block:
      - Trigger & Fetch
      - Parse Deals
      - Filter by Category
      - Deduplicate & Merge
      - Route & Write to Google Sheets

23. **Test manually**
    - Run the workflow manually once
    - Verify:
      - ScrapeOps returns HTML
      - Parser emits one item per deal
      - Filter keeps expected categories
      - Lookup finds existing rows
      - New rows append correctly
      - Existing rows update correctly

24. **Check common rebuild pitfalls**
    - The parser is tightly coupled to AppSumo HTML structure; test it after rebuilding
    - If your sheet already contains rows but no `Last Checked`, the IF node will treat them as new
    - If you rename sheet headers, update every Google Sheets mapping and lookup field
    - If timezone matters, do not rely solely on ISO timestamps without documenting UTC usage

25. **Activate the workflow**
    - After validation, activate it so the schedule trigger runs automatically each day

### Credential configuration required

- **ScrapeOps credentials**
  - Needed by: `Fetch AppSumo Deals via ScrapeOps (JS Render)`
  - Must be valid and allowed to use JS rendering

- **Google Sheets OAuth2 credentials**
  - Needed by:
    - `Lookup Deal in Google Sheet (by URL)`
    - `Append New Deal to Sheet`
    - `Update Existing Deal Row in Sheet`
  - The authenticated Google account must have access to the target spreadsheet

### Sub-workflow setup

This workflow does not invoke any sub-workflow and does not contain any sub-workflow trigger node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ScrapeOps registration for API key | https://scrapeops.io/app/register/n8n |
| ScrapeOps n8n overview documentation | https://scrapeops.io/docs/n8n/overview/ |
| ScrapeOps Proxy API documentation | https://scrapeops.io/docs/n8n/proxy-api/ |
| Duplicate the Google Sheet template used by the workflow | https://docs.google.com/spreadsheets/d/1L_Fn_wm55wc5EXi_ZIVmmXyFrz9nJ-29Xd71Ndz_XIk/edit#gid=0 |
| Suggested customization: add Slack, Gmail, or Discord alerts on the new deal path | Applies after `Stamp Last Checked (New Deal Path)` or after `Append New Deal to Sheet` |
| Suggested customization: change the schedule time to match your timezone | Applies to `Daily Schedule Trigger (09:00)` |

## Additional implementation notes

- The workflow has **one entry point**: `Daily Schedule Trigger (09:00)`.
- It has **no sub-workflows**.
- `scraped_at` is produced by the parser but is not written to Google Sheets in the current design.
- The deduplication logic depends on `AppSumo URL` being stable and unique.
- The new/existing classification depends on `Last Checked`, not directly on row existence. If an existing row lacks that value, duplicates may be appended.
- Because parsing is regex-based rather than DOM-based, it is lightweight but more fragile when AppSumo changes its page markup.