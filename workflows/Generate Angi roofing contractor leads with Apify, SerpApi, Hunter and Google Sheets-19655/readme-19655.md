Generate Angi roofing contractor leads with Apify, SerpApi, Hunter and Google Sheets

https://n8nworkflows.xyz/workflows/generate-angi-roofing-contractor-leads-with-apify--serpapi--hunter-and-google-sheets-19655


# Generate Angi roofing contractor leads with Apify, SerpApi, Hunter and Google Sheets

### 1. Workflow Overview

This workflow automates the daily generation, deduplication, enrichment, and storage of roofing contractor leads. It runs on a scheduled trigger, scrapes contractor profile data from an external API, cross-references existing entries to prevent duplication, enriches new leads with web domains and contact emails, persists final records to a spreadsheet, and broadcasts status updates to a messaging platform.

The workflow logic is divided into the following functional blocks:
- **1.1 Schedule Trigger & Data Acquisition**: Initiates execution on a daily interval and fetches raw contractor listings via an HTTP API request.
- **1.2 Data Normalization & Database Comparison**: Cleans scraped entries, eliminates intra-batch duplicates, reads existing historical entries from a spreadsheet, and flags incoming records as new or duplicate.
- **1.3 Filtering & Duplicate Handling**: Diverts new leads down the processing pipeline while instantly alerting operators via Slack regarding skipped duplicate records.
- **1.4 Domain Discovery**: Queries search engines to locate candidate website URLs for newly identified leads and strips out blocked directory or social media domains.
- **1.5 Contact Enrichment**: Branches execution based on domain availability; queries a domain intelligence API for executive or decision-maker contact details, or establishes a phone-only profile fallback.
- **1.6 Persistence & Notification**: Appends finalized lead profiles to the central spreadsheet and broadcasts a confirmation alert to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger & Data Acquisition
- **Overview:** Initializes the pipeline daily at 06:00 and retrieves raw business listings from the Apify actor endpoint targeting roofing contractors in Austin, TX.
- **Nodes Involved:** 
  - `When Schedule Reaches 6am`
  - `Fetch Angi Data`
- **Node Details:**
  - `When Schedule Reaches 6am`
    - **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)
    - **Technical Role:** Time-based trigger node initiating the pipeline execution.
    - **Configuration:** Configured with a daily rule triggering at hour 6.
    - **Input/Output:** Input: None | Output: Triggers downstream HTTP request.
    - **Failure Types:** Missed schedules due to platform downtime.
  - `Fetch Angi Data`
    - **Type:** `n8n-nodes-base.httpRequest` (v4.5)
    - **Technical Role:** External API client requesting scraped dataset results.
    - **Configuration:** Synchronous HTTP POST request to Apify API (`https://api.apify.com/v2/acts/R01HadmgmJV93ycd5/run-sync-get-dataset-items`) with JSON body defining search parameters (`searchTerms`: ["roofing"], `location`: "Austin, TX", `maxResults`: 100). Uses Generic HTTP Header Authentication.
    - **Credentials:** Apify (`httpHeaderAuth`)
    - **Input/Output:** Input: Schedule Trigger | Output: Raw list of contractor JSON objects.
    - **Failure Types:** API rate limits, authentication errors, network timeouts, invalid JSON payloads.

#### 2.2 Data Normalization & Database Comparison
- **Overview:** Standardizes phone numbers, cleans company names, builds fallback keys, removes duplicate profiles within the current scrape batch, and loads historical leads from Google Sheets to evaluate cross-batch uniqueness.
- **Nodes Involved:**
  - `Normalize and Deduplicate`
  - `Read Leads from Sheets`
  - `Check Existing Lead Duplicates`
- **Node Details:**
  - `Normalize and Deduplicate`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Technical Role:** JavaScript transformer executing custom data cleansing and intra-batch filtering.
    - **Configuration:** Iterates through items, strips legal suffixes from names (`inc`, `llc`, etc.), formats phone numbers to E.164 (`+1XXXXXXXXXX`), constructs fallback keys (`cleanName_city_state`), and tracks `proId`, phone, and fallback keys in `Set` structures to skip batch duplicates.
    - **Input/Output:** Input: `Fetch Angi Data` | Output: Normalized unique item collection.
    - **Failure Types:** Unhandled runtime exceptions due to missing or malformed input properties.
  - `Read Leads from Sheets`
    - **Type:** `n8n-nodes-base.googleSheets` (v4.5)
    - **Technical Role:** Database reader retrieving existing historical lead rows.
    - **Configuration:** Spreadsheet operation set to read all rows from sheet/tab named `Leads` within the target document (`REPLACE_WITH_SHEET_ID`).
    - **Credentials:** Google Sheets OAuth2 (`googleSheetsOAuth2Api`)
    - **Input/Output:** Input: `Normalize and Deduplicate` | Output: Rows of existing lead records.
    - **Failure Types:** Invalid Sheet ID, permission/OAuth token revocation, API quota exhaustion.
  - `Check Existing Lead Duplicates`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Technical Role:** JavaScript deduplication validator comparing incoming items against historical spreadsheet data.
    - **Configuration:** Maps historical `proId`, `phone`, and `fallbackKey` values into lookup sets, checks incoming records against these sets, and appends an `isDuplicate: boolean` property to each item.
    - **Input/Output:** Input: `Read Leads from Sheets` | Output: Items augmented with `isDuplicate` flags.
    - **Failure Types:** Data type mismatches between incoming string and historical database types.

#### 2.3 Filtering & Duplicate Handling
- **Overview:** Evaluates the duplicate status flag, routes duplicate leads to a notification alert, and forwards novel leads to domain discovery.
- **Nodes Involved:**
  - `If Lead is New`
  - `Notify Slack Duplicate Skipped`
- **Node Details:**
  - `If Lead is New`
    - **Type:** `n8n-nodes-base.if` (v2.2)
    - **Technical Role:** Conditional branch router.
    - **Configuration:** Evaluates condition `{{ $json.isDuplicate }} equals false`.
    - **Input/Output:** Input: `Check Existing Lead Duplicates` | Output: True branch (New leads), False branch (Duplicate leads).
    - **Failure Types:** Expression evaluation errors.
  - `Notify Slack Duplicate Skipped`
    - **Type:** `n8n-nodes-base.slack` (v2.3)
    - **Technical Role:** Messaging notification broadcaster for skipped items.
    - **Configuration:** Posts text message to channel ID `C0B8VH1M5PX` formatted with company name and phone status.
    - **Credentials:** Slack account (`slackApi`)
    - **Input/Output:** Input: `If Lead is New` (False branch) | Output: Slack API response.
    - **Failure Types:** Missing channel scope permissions, invalid Slack token, token expiration.

#### 2.4 Domain Discovery
- **Overview:** Queries an external search engine API using company names and locations to discover official corporate domains, filtering out general directory and social platform URLs.
- **Nodes Involved:**
  - `Fetch Domain Search Results`
  - `Validate and Extract Domains`
- **Node Details:**
  - `Fetch Domain Search Results`
    - **Type:** `n8n-nodes-base.httpRequest` (v4.5)
    - **Technical Role:** Search API client.
    - **Configuration:** GET request to SerpApi endpoint (`https://serpapi.com/search.json`) with query parameters (`engine`: google, `q`: `{{ $json.companyName }} roofing {{ $json.city }} {{ $json.state }}`, `num`: 5). Uses Generic Query Authentication and configured to never error on non-2xx responses.
    - **Credentials:** SerpApi/Query Auth (`httpQueryAuth`)
    - **Input/Output:** Input: `If Lead is New` (True branch) | Output: Raw search results JSON.
    - **Failure Types:** Search API rate limiting, network timeout, authentication failure.
  - `Validate and Extract Domains`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Technical Role:** Custom JavaScript filter parsing organic search results.
    - **Configuration:** Iterates over search links, excludes blocklisted domains (e.g., Yelp, BBB, Angi, Facebook), validates token matches against the company name, and assigns the best candidate to `discoveredDomain`.
    - **Input/Output:** Input: `Fetch Domain Search Results` | Output: Item augmented with `discoveredDomain`.
    - **Failure Types:** Malformed URLs throwing exceptions during `URL` constructor parsing.

#### 2.5 Contact Enrichment
- **Overview:** Checks whether a domain was successfully discovered. If valid, queries an email discovery service, ranks contacts by job seniority/title, and extracts executive details; if invalid, sets fallback parameters for a phone-only record.
- **Nodes Involved:**
  - `If Domain is Valid`
  - `Fetch Hunter Domain Data`
  - `Rank and Enrich Contacts`
  - `Set Contact Details`
- **Node Details:**
  - `If Domain is Valid`
    - **Type:** `n8n-nodes-base.if` (v2.2)
    - **Technical Role:** Conditional branch router.
    - **Configuration:** Evaluates condition `{{ $json.discoveredDomain }}` is not empty.
    - **Input/Output:** Input: `Validate and Extract Domains` | Output: True branch (Valid domain), False branch (No domain found).
    - **Failure Types:** Expression evaluation errors.
  - `Fetch Hunter Domain Data`
    - **Type:** `n8n-nodes-base.httpRequest` (v4.5)
    - **Technical Role:** Domain intelligence API client.
    - **Configuration:** GET request to Hunter Domain Search API (`https://api.hunter.io/v2/domain-search`) using query parameter `domain: {{ $json.discoveredDomain }}`. Uses Generic Query Authentication. Never errors on failure.
    - **Credentials:** Hunter (`httpQueryAuth`)
    - **Input/Output:** Input: `If Domain is Valid` (True branch) | Output: Raw domain contacts JSON.
    - **Failure Types:** Hunter API credit exhaustion, rate limit errors, invalid domain format.
  - `Rank and Enrich Contacts`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Technical Role:** JavaScript contact ranker.
    - **Configuration:** Scores contact objects based on title and seniority regular expressions (prioritizing owners, CEOs, and executives), selects the highest-scoring contact, and maps profile details to output fields.
    - **Input/Output:** Input: `Fetch Hunter Domain Data` | Output: Enriched contact lead object.
    - **Failure Types:** Empty email arrays or unexpected JSON schemas.
  - `Set Contact Details`
    - **Type:** `n8n-nodes-base.set` (v3.4)
    - **Technical Role:** Fallback value assignment node.
    - **Configuration:** Assigns default empty strings to website, email, contactName, and contactTitle, setting `enrichmentStatus` to `phone_only`. Includes all other incoming fields.
    - **Input/Output:** Input: `If Domain is Valid` (False branch) | Output: Standardized phone-only lead object.
    - **Failure Types:** None.

#### 2.6 Persistence & Notification
- **Overview:** Appends the finalized, enriched lead record to Google Sheets and sends a success notification message to Slack.
- **Nodes Involved:**
  - `Append Lead to Sheets`
  - `Notify Slack New Lead Posted`
- **Node Details:**
  - `Append Lead to Sheets`
    - **Type:** `n8n-nodes-base.googleSheets` (v4.5)
    - **Technical Role:** Database writer.
    - **Configuration:** Operation set to `append` on spreadsheet `REPLACE_WITH_SHEET_ID`, sheet name `Leads`. Mapped explicitly to schema columns (`proId`, `companyName`, `phone`, `email`, `website`, `contactName`, `contactTitle`, `enrichmentStatus`, etc.).
    - **Credentials:** Google Sheets OAuth2 (`googleSheetsOAuth2Api`)
    - **Input/Output:** Input: `Rank and Enrich Contacts` or `Set Contact Details` | Output: Appended row confirmation.
    - **Failure Types:** Schema mismatch, Google Sheets API rate limits, invalid OAuth credentials.
  - `Notify Slack New Lead Posted`
    - **Type:** `n8n-nodes-base.slack` (v2.3)
    - **Technical Role:** Messaging notification broadcaster for newly saved leads.
    - **Configuration:** Posts text message to channel ID `C0B8VH1M5PX` incorporating company name, city, state, enrichment status, email address, and contact title.
    - **Credentials:** Slack account (`slackApi`)
    - **Input/Output:** Input: `Append Lead to Sheets` | Output: Slack API response.
    - **Failure Types:** Token expiration, insufficient channel permissions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Roofing Contractor Leads Production Pipeline<br><br>### How it works<br><br>This workflow runs on a schedule to pull roofing contractor leads from an Apify Angi scrape, normalize them, and compare them against existing Google Sheets records. It filters out duplicates, searches for each new lead’s website domain, enriches valid domains with Hunter contact data, and falls back to phone-only records when no valid domain is found. New leads are saved to Google Sheets and Slack notifications are sent for both saved leads and skipped duplicates.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Add the Apify API token and confirm the Angi scrape actor or dataset endpoint used by the Fetch Angi Scrape HTTP request.<br>- Connect Google Sheets credentials and set the spreadsheet, sheet/tab, and column mappings for reading existing leads and saving new ones.<br>- Add SerpAPI credentials for the domain search request and Hunter.io credentials for contact enrichment.<br>- Configure Slack credentials and select the channels or users for new-lead and duplicate-skipped notifications.<br>- Review the code nodes to ensure field names match the Apify output and Google Sheets schema.<br><br>### Customization<br><br>Adjust the dedupe keys, lead normalization rules, domain validation logic, Hunter contact ranking criteria, fallback phone-only fields, and Slack message formats to match the preferred lead qualification process. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Scheduled Angi scrape<br><br>Starts the pipeline on a recurring schedule and fetches the latest roofing contractor lead data from the Apify Angi scrape endpoint. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Normalize and load records<br><br>Cleans and batches scraped lead data, then loads existing Google Sheets records so incoming leads can be compared against the database. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Dedupe and skip duplicates<br><br>Performs multi-key deduplication, decides whether each lead is new, and sends a Slack notification when a duplicate is skipped. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Find valid domain<br><br>Searches the web for a new lead’s website and extracts or validates the best matching domain for enrichment. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Enrich contact details<br><br>Branches based on domain validity, enriches valid domains through Hunter, ranks contact results, or prepares a phone-only lead when no valid domain is available. |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Save and notify lead<br><br>Writes the completed new lead record to Google Sheets and sends a Slack notification for the newly saved lead. |
| `When Schedule Reaches 6am` | `n8n-nodes-base.scheduleTrigger` | Time-based trigger | None | `Fetch Angi Data` | |
| `Fetch Angi Data` | `n8n-nodes-base.httpRequest` | Fetch scraped leads | `When Schedule Reaches 6am` | `Normalize and Deduplicate` | |
| `Normalize and Deduplicate` | `n8n-nodes-base.code` | Clean and filter records | `Fetch Angi Data` | `Read Leads from Sheets` | |
| `Read Leads from Sheets` | `n8n-nodes-base.googleSheets` | Read historical leads | `Normalize and Deduplicate` | `Check Existing Lead Duplicates` | |
| `Check Existing Lead Duplicates` | `n8n-nodes-base.code` | Compare against history | `Read Leads from Sheets` | `If Lead is New` | |
| `If Lead is New` | `n8n-nodes-base.if` | Route new vs duplicate | `Check Existing Lead Duplicates` | `Fetch Domain Search Results`, `Notify Slack Duplicate Skipped` | |
| `Fetch Domain Search Results` | `n8n-nodes-base.httpRequest` | Search company website | `If Lead is New` (True) | `Validate and Extract Domains` | |
| `Validate and Extract Domains` | `n8n-nodes-base.code` | Filter blocklisted domains | `Fetch Domain Search Results` | `If Domain is Valid` | |
| `If Domain is Valid` | `n8n-nodes-base.if` | Route domain availability | `Validate and Extract Domains` | `Fetch Hunter Domain Data`, `Set Contact Details` | |
| `Fetch Hunter Domain Data` | `n8n-nodes-base.httpRequest` | Fetch domain contacts | `If Domain is Valid` (True) | `Rank and Enrich Contacts` | |
| `Rank and Enrich Contacts` | `n8n-nodes-base.code` | Score and rank contacts | `Fetch Hunter Domain Data` | `Append Lead to Sheets` | |
| `Set Contact Details` | `n8n-nodes-base.set` | Set phone-only defaults | `If Domain is Valid` (False) | `Append Lead to Sheets` | |
| `Append Lead to Sheets` | `n8n-nodes-base.googleSheets` | Save final lead record | `Rank and Enrich Contacts`, `Set Contact Details` | `Notify Slack New Lead Posted` | |
| `Notify Slack New Lead Posted` | `n8n-nodes-base.slack` | Send success alert | `Append Lead to Sheets` | None | |
| `Notify Slack Duplicate Skipped` | `n8n-nodes-base.slack` | Send duplicate alert | `If Lead is New` (False) | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node named `When Schedule Reaches 6am`.
   - Set interval rule to trigger daily at hour `6`.
2. **Configure Apify HTTP Request:**
   - Add an **HTTP Request** node named `Fetch Angi Data`.
   - Set Method to `POST`, URL to `https://api.apify.com/v2/acts/R01HadmgmJV93ycd5/run-sync-get-dataset-items`.
   - Set Authentication to Generic Credential Type -> HTTP Header Auth (`Apify`).
   - Specify Body as JSON with parameters (`searchTerms`: `["roofing"]`, `location`: `Austin, TX`, `maxResults`: `100`, `detailLevel`: `full`, `keepPartialProfiles`: `false`, `includeReviews`: `false`).
   - Connect `When Schedule Reaches 6am` -> `Fetch Angi Data`.
3. **Add Normalization Code Node:**
   - Add a **Code** node named `Normalize and Deduplicate`.
   - Paste JavaScript snippet to clean names, format phones, build fallback keys (`cleanName_city_state`), and remove intra-batch duplicates.
   - Connect `Fetch Angi Data` -> `Normalize and Deduplicate`.
4. **Configure Historical Sheet Reader:**
   - Add a **Google Sheets** node named `Read Leads from Sheets`.
   - Set Operation to `Get Many` (or read rows), Spreadsheet ID to `REPLACE_WITH_SHEET_ID`, Sheet Name to `Leads`.
   - Configure Google Sheets OAuth2 credentials.
   - Connect `Normalize and Deduplicate` -> `Read Leads from Sheets`.
5. **Add Deduplication Check Code Node:**
   - Add a **Code** node named `Check Existing Lead Duplicates`.
   - Paste JavaScript to verify incoming records against historical spreadsheet data and assign `isDuplicate: boolean`.
   - Connect `Read Leads from Sheets` -> `Check Existing Lead Duplicates`.
6. **Configure Duplicate Branching:**
   - Add an **If** node named `If Lead is New`.
   - Set condition: `{{ $json.isDuplicate }} equals false`.
   - Connect `Check Existing Lead Duplicates` -> `If Lead is New`.
   - **False Branch:** Connect to a **Slack** node named `Notify Slack Duplicate Skipped` (Channel ID: `C0B8VH1M5PX`, configure Slack OAuth2 credentials, set text message for skipped duplicate).
7. **Configure Domain Search:**
   - Add an **HTTP Request** node named `Fetch Domain Search Results`.
   - Set Method to `GET`, URL to `https://serpapi.com/search.json`.
   - Set Authentication to Generic Credential Type -> HTTP Query Auth. Add parameters: `engine` (`google`), `q` (`{{ $json.companyName }} roofing {{ $json.city }} {{ $json.state }}`), `num` (`5`). Enable "Never Error on Response".
   - Connect `If Lead is New` (True branch) -> `Fetch Domain Search Results`.
8. **Add Domain Validation Code Node:**
   - Add a **Code** node named `Validate and Extract Domains`.
   - Paste JavaScript to filter out blocklisted domains and check token matches.
   - Connect `Fetch Domain Search Results` -> `Validate and Extract Domains`.
9. **Configure Domain Validity Branch:**
   - Add an **If** node named `If Domain is Valid`.
   - Set condition: `{{ $json.discoveredDomain }}` is not empty.
   - Connect `Validate and Extract Domains` -> `If Domain is Valid`.
10. **Configure Hunter Enrichment (True Branch):**
    - Add an **HTTP Request** node named `Fetch Hunter Domain Data`.
    - Set Method to `GET`, URL to `https://api.hunter.io/v2/domain-search`.
    - Set Authentication to Generic Credential Type -> HTTP Query Auth. Query parameter: `domain` (`{{ $json.discoveredDomain }}`). Enable "Never Error on Response".
    - Connect `If Domain is Valid` (True branch) -> `Fetch Hunter Domain Data`.
    - Add a **Code** node named `Rank and Enrich Contacts` to rank email results by seniority/title. Connect `Fetch Hunter Domain Data` -> `Rank and Enrich Contacts`.
11. **Configure Phone-Only Fallback (False Branch):**
    - Add a **Set** node named `Set Contact Details`.
    - Assign default empty values to website, email, contactName, contactTitle, and set `enrichmentStatus` to `phone_only`. Include other fields.
    - Connect `If Domain is Valid` (False branch) -> `Set Contact Details`.
12. **Configure Spreadsheet Appending & Success Notification:**
    - Add a **Google Sheets** node named `Append Lead to Sheets`.
    - Operation: `Append`. Spreadsheet ID: `REPLACE_WITH_SHEET_ID`, Sheet Name: `Leads`.
    - Map all schema columns (`proId`, `companyName`, `phone`, `email`, `website`, `contactName`, `contactTitle`, `enrichmentStatus`, `scrapedAt`, `angiUrl`, `description`, `fallbackKey`, `fullAddress`, `city`, `state`, `zip`).
    - Connect both `Rank and Enrich Contacts` and `Set Contact Details` -> `Append Lead to Sheets`.
    - Add a **Slack** node named `Notify Slack New Lead Posted` (Channel ID: `C0B8VH1M5PX`, message template showing saved lead name, city, state, enrichment status, and contact email).
    - Connect `Append Lead to Sheets` -> `Notify Slack New Lead Posted`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video walkthrough and overview of the automation pipeline | [Video Link](https://lnkd.in/p/dCCv3avg) |