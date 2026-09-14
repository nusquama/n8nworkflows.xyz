Scrape Google Maps business leads with SearchApi, Hunter, and Google Sheets

https://n8nworkflows.xyz/workflows/scrape-google-maps-business-leads-with-searchapi--hunter--and-google-sheets-17670


# Scrape Google Maps business leads with SearchApi, Hunter, and Google Sheets

### 1. Workflow Overview

This workflow is designed to automate the collection, enrichment, and storage of local business leads from Google Maps. It serves sales and marketing use cases by generating targeted contact databases based on specific business categories and geographic locations. 

The execution flow is structured into five functional blocks:
- **1.1 Input Reception & Configuration:** Triggers the workflow either manually on demand or automatically on a weekly schedule, and defines parameters for target industries and regions.
- **1.2 Query Construction & Scraping:** Dynamically builds segmented search queries across multiple pages, then queries the SearchApi.io Google Maps engine to fetch live business profiles.
- **1.3 Data Extraction & Deduplication:** Parses raw search output, flattens multi-page results, removes duplicate business entries via unique platform identifiers, and structures the lead fields.
- **1.4 Email Enrichment:** Queries Hunter.io using domain intelligence to discover verified email addresses, evaluates confidence scores, and appends the highest-ranking email to each lead profile.
- **1.5 Data Persistence:** Appends finalized, structured lead records directly into a target Google Sheets worksheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** This block establishes the entry points for the workflow and sets the baseline configuration variables containing the desired business categories and geographic locations.
- **Nodes Involved:** 
  - `Run Now` (Manual Trigger)
  - `Schedule Weekly` (Schedule Trigger)
  - `Settings` (Set Node)

- **Node Details:**
  - **Run Now**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` — Initiates an on-demand execution.
    - *Configuration:* Default parameters.
    - *Input/Output:* No inputs; outputs a single execution trigger to `Settings`.
    - *Edge Cases:* None.
  - **Schedule Weekly**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` — Initiates an automated run based on a cron-like schedule.
    - *Configuration:* Configured to fire every Monday at 08:00 (`triggerAtDay: [1]`, `triggerAtHour: 8`).
    - *Input/Output:* No inputs; outputs a trigger signal to `Settings`.
    - *Edge Cases:* Requires the workflow to be active in the n8n environment to trigger automatically.
  - **Settings**
    - *Type & Technical Role:* `n8n-nodes-base.set` — Holds editable string assignments for target search criteria.
    - *Configuration:* Assigns two primary string values: `categories` ("coffee shop, cafe") and `locations` ("Austin, Texas, United States; Denver, Colorado, United States").
    - *Input/Output:* Inputs from `Run Now` or `Schedule Weekly`; outputs to `Build Search Queries`.
    - *Edge Cases:* Malformed strings (e.g., missing semicolons between locations) can cause downstream parsing errors.

---

#### 2.2 Query Construction & Scraping
- **Overview:** This block transforms user-defined configuration values into a matrix of paginated search queries and executes live calls against the SearchApi.io service.
- **Nodes Involved:**
  - `Build Search Queries` (Code Node)
  - `Scrape Google Maps` (Community SearchApi Node)

- **Node Details:**
  - **Build Search Queries**
    - *Type & Technical Role:* `n8n-nodes-base.code` — JavaScript execution environment used to split, iterate, and build combinatorial queries.
    - *Configuration:* Reads categories (comma-separated) and locations (semicolon-separated) from the `Settings` node. Iterates across a hardcoded pagination depth (`PAGES = 2`, yielding ~20 leads per page) to output unique query payloads.
    - *Key Expressions:* `const cfg = $('Settings').first().json;`
    - *Input/Output:* Input from `Settings`; outputs an array of objects containing `category`, `location`, `q`, and `page`.
    - *Edge Cases:* Throws a runtime error if either `categories` or `locations` evaluates to an empty list.
  - **Scrape Google Maps**
    - *Type & Technical Role:* `@searchapi/n8n-nodes-searchapi.searchApi` — External API integration node for querying Google Maps listings via SearchApi.io.
    - *Configuration:* Queries set via expression `={{ $json.q }}`, targeting resource `google_maps`, with pagination mapped to `={{ $json.page }}`. Configured with a retry policy (`maxTries: 3`, `waitBetweenTries: 2000`) and set to continue regular output on execution errors.
    - *Credentials Required:* SearchApi account (`searchApi`).
    - *Input/Output:* Input from `Build Search Queries`; outputs raw API responses containing `local_results`.
    - *Edge Cases:* Rate limits, exhausted API credits, or network timeouts. Errors are caught and handled smoothly via `continueRegularOutput`.

---

#### 2.3 Data Extraction & Deduplication
- **Overview:** This block processes raw search results, normalizes fields, extracts actionable attributes, and eliminates duplicate businesses using unique identifiers.
- **Nodes Involved:**
  - `Extract Leads` (Code Node)

- **Node Details:**
  - **Extract Leads**
    - *Type & Technical Role:* `n8n-nodes-base.code` — JavaScript execution node for data transformation and relational mapping.
    - *Configuration:* Iterates through input search payloads and maps them back against original query definitions. Utilizes a JavaScript `Set` tracking `place_id` (or fallback composite keys) to drop duplicate listings. Parses website URLs to extract root domains.
    - *Key Expressions:* Accesses preceding query metadata via `$('Build Search Queries').all()`.
    - *Input/Output:* Input from `Scrape Google Maps`; outputs structured lead objects containing keys such as `business`, `category`, `location`, `address`, `phone`, `website`, `domain`, `rating`, `reviews`, `maps_url`, and `place_id`.
    - *Edge Cases:* Missing metadata fields or malformed URLs during domain parsing are safely caught and defaulted to empty strings.

---

#### 2.4 Email Enrichment
- **Overview:** This block queries Hunter.io to discover and select the most reliable email address associated with each business domain.
- **Nodes Involved:**
  - `Find Emails (Hunter)` (Hunter Node)
  - `Attach Emails` (Code Node)

- **Node Details:**
  - **Find Emails (Hunter)**
    - *Type & Technical Role:* `n8n-nodes-base.hunter` — External API integration node for domain search operations.
    - *Configuration:* Configured to request up to 10 emails per domain (`limit: 10`), mapping the target domain parameter to `={{ $json.domain }}`. Configured with error continuity (`onError: continueRegularOutput`).
    - *Credentials Required:* Hunter account (`hunterApi`).
    - *Input/Output:* Input from `Extract Leads`; outputs domain search results containing nested email arrays and confidence ratings.
    - *Edge Cases:* Consumes one search credit per lead. Domains without websites or unrecognized structures pass through without throwing hard workflow blockers.
  - **Attach Emails**
    - *Type & Technical Role:* `n8n-nodes-base.code` — JavaScript execution node that matches Hunter results back to base leads by array index.
    - *Configuration:* Sorts returned email arrays by descending `confidence` score and selects the highest-scoring email address to populate the `email` field.
    - *Key Expressions:* Relies on positional matching against `$('Extract Leads').all()`.
    - *Input/Output:* Input from `Find Emails (Hunter)`; outputs fully enriched lead objects.
    - *Edge Cases:* Empty email arrays result in an empty string assigned to the lead's `email` property.

---

#### 2.5 Data Persistence
- **Overview:** This final block maps finalized lead objects and appends them as new rows into a designated Google Sheets document.
- **Nodes Involved:**
  - `Save Leads to Sheet` (Google Sheets Node)

- **Node Details:**
  - **Save Leads to Sheet**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` — Integration node for appending rows to spreadsheets.
    - *Configuration:* Operation set to `append`. Uses automatic field mapping (`mappingMode: autoMapInputData`) targeting document ID `1PLHGO9WBiLaWsSb8N00WB7DtB9mq-lmvezwWwnr6y1M` and worksheet `Sheet1`.
    - *Credentials Required:* Google Sheets OAuth2 API (`googleSheetsOAuth2Api`).
    - *Input/Output:* Input from `Attach Emails`; outputs confirmation of appended spreadsheet rows.
    - *Edge Cases:* Schema mismatches between incoming data keys and target spreadsheet column headers, or expired OAuth credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Now | `n8n-nodes-base.manualTrigger` | Run on demand when you need a fresh batch of leads. Click Execute Workflow. | None | Settings | Run on demand or on a weekly schedule. Edit your categories and locations in **Settings** — nothing else needs touching. |
| Schedule Weekly | `n8n-nodes-base.scheduleTrigger` | Optional: activate the workflow to pull new listings every Monday at 08:00. Change or delete this if you only run manually. | None | Settings | Run on demand or on a weekly schedule. Edit your categories and locations in **Settings** — nothing else needs touching. |
| Settings | `n8n-nodes-base.set` | The only node you normally edit. categories = comma-separated business types. locations = semicolon-separated places (names contain commas, so we split on ;). | Run Now, Schedule Weekly | Build Search Queries | Run on demand or on a weekly schedule. Edit your categories and locations in **Settings** — nothing else needs touching. |
| Build Search Queries | `n8n-nodes-base.code` | Expands your categories and locations into one Google Maps search per combination per page. Raise PAGES here for more leads per search. | Settings | Scrape Google Maps | Each category + location becomes a live Google Maps search via SearchApi. Results from every page are flattened into one row per business and deduplicated by place. |
| Scrape Google Maps | `@searchapi/n8n-nodes-searchapi.searchApi` | Live Google Maps results via SearchApi.io, one call per category + location + page. Returns business name, address, phone, website, rating and reviews. | Build Search Queries | Extract Leads | Each category + location becomes a live Google Maps search via SearchApi. Results from every page are flattened into one row per business and deduplicated by place. |
| Extract Leads | `n8n-nodes-base.code` | Reads local_results from every page, keeps the useful lead fields, and removes duplicate businesses (by place_id). This is the clean, structured lead list. | Scrape Google Maps | Find Emails (Hunter) | Each category + location becomes a live Google Maps search via SearchApi. Results from every page are flattened into one row per business and deduplicated by place. |
| Find Emails (Hunter) | `n8n-nodes-base.hunter` | Looks up the emails on file for each business domain via Hunter.io Domain Search. Businesses with no website/domain pass through with no email. Uses Hunter credits: one search per lead. | Extract Leads | Attach Emails | Hunter.io looks up the emails on file for each business domain; the highest-confidence one is attached to the lead. Delete these two nodes to skip email enrichment. |
| Attach Emails | `n8n-nodes-base.code` | Attaches the best email (highest Hunter confidence) to each lead and passes the full row on to the sheet. | Find Emails (Hunter) | Save Leads to Sheet | Hunter.io looks up the emails on file for each business domain; the highest-confidence one is attached to the lead. Delete these two nodes to skip email enrichment. |
| Save Leads to Sheet | `n8n-nodes-base.googleSheets` | Appends each lead as a new row. Select your Google Sheet and tab, and give it a header row: business, category, location, type, address, phone, website, rating, reviews, maps_url, place_id. | Attach Emails | None | Appends every lead to your Google Sheet: name, type, address, phone, website, email, rating, reviews and a Maps link. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) named `Run Now`.
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`) named `Schedule Weekly`. Configure its interval to trigger weekly on Mondays at 08:00.

2. **Create and Configure the Settings Node:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Settings`.
   - Add string assignments:
     - `categories`: `coffee shop, cafe`
     - `locations`: `Austin, Texas, United States; Denver, Colorado, United States`
   - Connect both `Run Now` and `Schedule Weekly` outputs to `Settings`.

3. **Build the Query Generation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Build Search Queries`.
   - Paste the pagination and query iteration logic into the JavaScript parameter to parse comma-separated categories and semicolon-separated locations.
   - Connect the output of `Settings` to `Build Search Queries`.

4. **Set Up the SearchApi Integration:**
   - Install the community package `@searchapi/n8n-nodes-searchapi`.
   - Add a **SearchApi** node (`@searchapi/n8n-nodes-searchapi.searchApi`) named `Scrape Google Maps`.
   - Configure the query parameter to `={{ $json.q }}`, resource to `google_maps`, and pagination page to `={{ $json.page }}`.
   - Configure retry options (3 tries, 2000ms wait) and enable execution continuation on error.
   - Attach your SearchApi credential.
   - Connect the output of `Build Search Queries` to `Scrape Google Maps`.

5. **Create the Extraction and Deduplication Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Extract Leads`.
   - Insert the flattening logic to parse `local_results`, rebuild category/location metadata by position, and filter duplicate entries using `place_id`.
   - Connect the output of `Scrape Google Maps` to `Extract Leads`.

6. **Configure Email Enrichment (Optional):**
   - Add a **Hunter** node (`n8n-nodes-base.hunter`) named `Find Emails (Hunter)`.
   - Set domain mapping to `={{ $json.domain }}`, limit to `10`, and configure error continuation. Attach your Hunter.io credential.
   - Connect `Extract Leads` to `Find Emails (Hunter)`.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Attach Emails`.
   - Insert the JavaScript logic to sort incoming email arrays by confidence score and assign the highest value to `lead.email`.
   - Connect `Find Emails (Hunter)` to `Attach Emails`.

7. **Configure Data Persistence:**
   - Prepare a Google Sheet containing a header row with fields: `business`, `category`, `location`, `type`, `address`, `phone`, `website`, `email`, `rating`, `reviews`, `maps_url`, `place_id`.
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Save Leads to Sheet`.
   - Set the operation to `append`, select your target Document ID and Worksheet, and configure schema mapping to auto-map input data.
   - Attach your Google Sheets OAuth2 API credential.
   - Connect `Attach Emails` (or `Extract Leads` if skipping email enrichment) to `Save Leads to Sheet`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| SearchApi.io Integration Platform | [SearchApi Website](https://searchapi.io) |
| Hunter.io Email Discovery Platform | [Hunter Website](https://hunter.io) |
| Community Node Package Identifier | `@searchapi/n8n-nodes-searchapi` |