Track GitHub Trending Repositories with ScrapeOps & Google Sheets

https://n8nworkflows.xyz/workflows/track-github-trending-repositories-with-scrapeops---google-sheets-11706


# Track GitHub Trending Repositories with ScrapeOps & Google Sheets

### 1. Workflow Overview

This workflow, titled **"GitHub Trending Dev Tools"**, automates the process of tracking trending GitHub repositories over daily, weekly, or monthly intervals and stores the results in Google Sheets. It is designed for developers or teams interested in monitoring popular open source projects or developer tools across various programming languages.

The workflow logic is grouped into the following functional blocks:

- **1.1 Input Configuration & URL Construction:** Defines time window and programming languages, then builds GitHub Trending URLs accordingly.
- **1.2 Fetching Trending Pages:** Uses ScrapeOps to scrape the HTML content of the GitHub Trending pages with polite delays to avoid rate-limiting.
- **1.3 Parsing and Normalizing Data:** Extracts repository metadata from the HTML, cleans and enriches the data (e.g., parsing star counts, scoring), and generates deduplication keys.
- **1.4 Data Storage to Google Sheets:** Upserts the parsed repository data into a raw data sheet and optionally generates a weekly summary brief saved in another sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Configuration & URL Construction

- **Overview:**  
  This block sets the parameters for the trending time window (`since` daily/weekly/monthly) and the list of programming languages to filter by. It then constructs the corresponding GitHub Trending URLs for each language and time frame combination.

- **Nodes Involved:**  
  - Cron (daily/weekly)  
  - Set Inputs  
  - Build URLs  
  - Split URLs (loop)  

- **Node Details:**  

  - **Cron (daily/weekly)**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a scheduled interval (default daily, can be customized).  
    - Configuration: Interval trigger with no specific time set (runs daily by default).  
    - Inputs: None  
    - Outputs: Triggers "Set Inputs" node.  
    - Potential failures: Cron misconfigurations or disabled scheduler.

  - **Set Inputs**  
    - Type: Set  
    - Role: Defines key inputs such as `since` ("monthly" by default) and `languages_csv` ("any" by default).  
    - Configuration: Static values set — `since` and `languages_csv`.  
    - Inputs: Trigger from Cron.  
    - Outputs: Feeds parameters into "Build URLs" and "Read raw (for weekly)" nodes.  
    - Edge cases: Invalid or unsupported language names; defaults applied in "Build URLs".

  - **Build URLs**  
    - Type: Code  
    - Role: Constructs GitHub Trending URLs based on inputs. Normalizes inputs, handles duplicates, encodes language slugs for URLs, and generates a `week_id` for reference.  
    - Key Expressions: Uses JavaScript to parse languages, normalize `since`, and build URLs like `https://github.com/trending/{language}?since={since}`.  
    - Inputs: Receives parameters from "Set Inputs".  
    - Outputs: Produces array of URL objects with metadata for scraping.  
    - Edge cases: Unknown languages are skipped; invalid `since` values default to "daily".  
    - Notes: Adds metadata such as sheet names and captured timestamps for downstream use.

  - **Split URLs (loop)**  
    - Type: SplitInBatches  
    - Role: Iterates over each URL individually to process sequentially (batch size = 1).  
    - Inputs: Receives array of URLs from "Build URLs".  
    - Outputs: Feeds individual URLs one-by-one to "ScrapeOps: Fetch HTML".  
    - Edge cases: Empty URL list results in no processing.

---

#### 2.2 Fetching Trending Pages

- **Overview:**  
  This block downloads the HTML content of each GitHub Trending page using the ScrapeOps service, then enforces a polite delay between requests to respect rate limits and avoid blocking.

- **Nodes Involved:**  
  - ScrapeOps: Fetch HTML  
  - Polite Delay  

- **Node Details:**  

  - **ScrapeOps: Fetch HTML**  
    - Type: ScrapeOps Community Node  
    - Role: Fetches the raw HTML of the given GitHub Trending URL using ScrapeOps API for reliability and anti-blocking features.  
    - Configuration: URL parameterized from input JSON, uses ScrapeOps API credentials.  
    - Inputs: Receives one URL per execution from "Split URLs (loop)".  
    - Outputs: Raw HTML content in JSON data field.  
    - Edge cases: Network errors, API key limits, invalid URLs, or blocked requests.  
    - Version: Requires ScrapeOps credentials set up in n8n.

  - **Polite Delay**  
    - Type: Wait  
    - Role: Pauses the workflow for 2 seconds between scraping requests to prevent overloading GitHub or triggering anti-scraping defenses.  
    - Configuration: 2 seconds delay.  
    - Inputs: Receives HTML from "ScrapeOps: Fetch HTML".  
    - Outputs: Passes data forward to "Extract Trending Repos".  
    - Edge cases: Delay node failure is rare; main risk is setting delay too low causing rate limiting.

---

#### 2.3 Parsing and Normalizing Data

- **Overview:**  
  Parses the scraped HTML to extract relevant repository metadata, then normalizes numeric fields, enriches data with keyword matching and scoring, and generates deduplication keys to avoid duplicate records.

- **Nodes Involved:**  
  - Extract Trending Repos  
  - Normalize + Enrich-from-page  

- **Node Details:**  

  - **Extract Trending Repos**  
    - Type: Code  
    - Role: Uses regex to parse each repository's HTML `<article>` block, extracting owner, repo name, description, language, star and fork counts, and star increments over the period.  
    - Key Expressions: JavaScript regex matching with careful HTML cleansing; filters out invalid entries (e.g., "sponsors").  
    - Inputs: Raw HTML from "Polite Delay" and upstream metadata from "Split URLs (loop)".  
    - Outputs: Array of parsed repo objects with metadata fields.  
    - Edge cases: Changes in GitHub HTML structure may break regex; missing fields handled gracefully by empty strings.

  - **Normalize + Enrich-from-page**  
    - Type: Code  
    - Role: Parses raw star/fork counts into numbers (supports k/M suffixes), calculates a custom score based on stars in period, keyword matches, and language boosts, and creates a unique dedupe key combining `since` and `full_name`.  
    - Key Expressions: JavaScript functions for number parsing, keyword filtering, scoring logic.  
    - Inputs: Parsed repo data from "Extract Trending Repos".  
    - Outputs: Enriched repo metadata ready for storage.  
    - Edge cases: Non-numeric or malformed counts default to zero; missing keywords or language info handled.

---

#### 2.4 Data Storage to Google Sheets

- **Overview:**  
  Stores the enriched repository data into a Google Sheet, using "Append or Update Row" operation keyed by a deduplication key to avoid inserting duplicate rows. Additionally, it optionally reads raw data to generate and store weekly summary briefs.

- **Nodes Involved:**  
  - Write to Google Sheets (raw tab)  
  - Read raw (for weekly)  
  - Generate Weekly  
  - Write to Google Sheets (weekly tab)  

- **Node Details:**  

  - **Write to Google Sheets (raw tab)**  
    - Type: Google Sheets  
    - Role: Upserts each repository record into the raw data tab (`trending_raw`) of the specified Google Sheet document. Matches rows by `dedupe_key` to update existing entries or append new ones.  
    - Configuration: Maps multiple columns including repo info, counts, scores, timestamps, and URLs. Document ID and sheet name configured.  
    - Inputs: Enriched repo data from "Normalize + Enrich-from-page".  
    - Outputs: Data stored in Google Sheets; no further output.  
    - Edge cases: Google API quota limits, credential expiration, schema mismatch.

  - **Read raw (for weekly)**  
    - Type: Google Sheets  
    - Role: Reads the entire raw sheet data to use as input for generating weekly summaries.  
    - Configuration: Points to `trending_raw` sheet of the same Google Sheet document.  
    - Inputs: Parallel input from "Set Inputs" (triggered initially).  
    - Outputs: Passes raw data JSON array to "Generate Weekly".  
    - Edge cases: Large sheet size may slow reading; permission issues.

  - **Generate Weekly**  
    - Type: Code  
    - Role: Filters raw rows to current week, identifies developer tools based on keyword matches, calculates top repositories by score, and extracts trending themes. Produces summary text and metadata.  
    - Key Expressions: JavaScript filtering, sorting, aggregation.  
    - Inputs: Raw sheet data from "Read raw (for weekly)" and week id from "Build URLs".  
    - Outputs: JSON summary object with top repos, themes, counts, and notes.  
    - Edge cases: Empty data results in empty summaries; keyword matching relies on populated keywords.

  - **Write to Google Sheets (weekly tab)**  
    - Type: Google Sheets  
    - Role: Appends weekly summary data to the `weekly_brief` tab in Google Sheets.  
    - Configuration: Maps summary fields like week_id, notes, top repos, themes, and generation timestamp. No deduplication key (append only).  
    - Inputs: Summary JSON from "Generate Weekly".  
    - Outputs: Data saved in weekly summary sheet.  
    - Edge cases: Same as other Google Sheets node; no update, only append.

---

### 3. Summary Table

| Node Name                      | Node Type                   | Functional Role                               | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                                     |
|--------------------------------|-----------------------------|-----------------------------------------------|------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Cron (daily/weekly)             | Schedule Trigger            | Starts scheduled workflow run                  | None                         | Set Inputs                      |                                                                                                                                 |
| Set Inputs                     | Set                         | Defines inputs (since, languages_csv)          | Cron (daily/weekly)           | Build URLs, Read raw (for weekly) | ## Inputs & URL Builder Set `since` + languages, then generate GitHub Trending URLs.                                            |
| Build URLs                    | Code                        | Builds GitHub Trending URLs based on inputs    | Set Inputs                   | Split URLs (loop)               | ## Inputs & URL Builder Set `since` + languages, then generate GitHub Trending URLs.                                            |
| Split URLs (loop)              | SplitInBatches              | Loops over URLs to process each one individually | Build URLs                   | ScrapeOps: Fetch HTML           | ## Scrape Trending Pages Fetch HTML via ScrapeOps, then wait briefly to be polite.                                              |
| ScrapeOps: Fetch HTML          | ScrapeOps API               | Fetches HTML of GitHub Trending pages          | Split URLs (loop)             | Polite Delay                   | ## Scrape Trending Pages Fetch HTML via ScrapeOps, then wait briefly to be polite.                                              |
| Polite Delay                  | Wait                        | Adds delay between scrape requests              | ScrapeOps: Fetch HTML         | Extract Trending Repos           | ## Scrape Trending Pages Fetch HTML via ScrapeOps, then wait briefly to be polite.                                              |
| Extract Trending Repos         | Code                        | Parses HTML to extract repo metadata            | Polite Delay                 | Normalize + Enrich-from-page     | ## Parse & Normalize Extract repos + rank from HTML and normalize numbers + scoring + dedupe key.                              |
| Normalize + Enrich-from-page   | Code                        | Cleans and enriches extracted data, scores repos | Extract Trending Repos       | Write to Google Sheets (raw tab) | ## Parse & Normalize Extract repos + rank from HTML and normalize numbers + scoring + dedupe key.                              |
| Write to Google Sheets (raw tab) | Google Sheets              | Upserts parsed repo data into raw data sheet    | Normalize + Enrich-from-page | None                           | ## Save Results Upsert rows into Google Sheets using `dedupe_key` to avoid duplicates.                                          |
| Read raw (for weekly)          | Google Sheets               | Reads raw data sheet to prepare weekly summary  | Set Inputs                   | Generate Weekly                | ## Weekly Brief (Optional) Reads raw data and writes a summary into the weekly_brief tab.                                       |
| Generate Weekly               | Code                        | Generates weekly summary from raw data          | Read raw (for weekly)         | Write to Google Sheets (weekly tab) | ## Weekly Brief (Optional) Reads raw data and writes a summary into the weekly_brief tab.                                       |
| Write to Google Sheets (weekly tab) | Google Sheets           | Appends weekly summary to weekly brief sheet   | Generate Weekly              | None                           | ## Weekly Brief (Optional) Reads raw data and writes a summary into the weekly_brief tab.                                       |
| Sticky Note                   | Sticky Note                 | Overview and setup instructions                  | None                         | None                           | # GitHub Trending Tracker (Daily/Weekly/Monthly) → Google Sheets … (see full sticky note text in Block 1.1)                      |
| Sticky Note1                  | Sticky Note                 | Notes on Inputs & URL Builder block              | None                         | None                           | ## Inputs & URL Builder Set `since` + languages, then generate GitHub Trending URLs.                                            |
| Sticky Note2                  | Sticky Note                 | Notes on Fetching Trending Pages block           | None                         | None                           | ## Scrape Trending Pages Fetch HTML via ScrapeOps, then wait briefly to be polite.                                              |
| Sticky Note3                  | Sticky Note                 | Notes on Parsing & Normalizing block              | None                         | None                           | ## Parse & Normalize Extract repos + rank from HTML and normalize numbers + scoring + dedupe key.                              |
| Sticky Note4                  | Sticky Note                 | Notes on Save Results block                        | None                         | None                           | ## Save Results Upsert rows into Google Sheets using `dedupe_key` to avoid duplicates.                                          |
| Sticky Note5                  | Sticky Note                 | Notes on Weekly Brief optional summary            | None                         | None                           | ## Weekly Brief (Optional) Reads raw data and writes a summary into the weekly_brief tab.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Cron (daily/weekly)`  
   - Set schedule interval (default daily or customize to weekly/monthly).  
   - No inputs, outputs connected to "Set Inputs".

3. **Add a Set node:**  
   - Name: `Set Inputs`  
   - Configure two string fields:  
     - `since` = `monthly` (can be daily/weekly/monthly)  
     - `languages_csv` = `any` (comma-separated list of languages)  
   - Connect input from `Cron (daily/weekly)` node.

4. **Add a Code node:**  
   - Name: `Build URLs`  
   - Paste the provided JavaScript code (from the "Build URLs" node) that:  
       - Normalizes `since` value.  
       - Parses `languages_csv`.  
       - Maps languages to GitHub URL slugs (with encoding for C++).  
       - Generates URLs with query parameter `since`.  
       - Adds metadata like `week_id`, `sheet_id`, and sheet names.  
   - Connect input from `Set Inputs`.

5. **Add a SplitInBatches node:**  
   - Name: `Split URLs (loop)`  
   - Batch Size: 1 (to process URLs sequentially).  
   - Connect input from `Build URLs`.

6. **Add a ScrapeOps node (community node):**  
   - Name: `ScrapeOps: Fetch HTML`  
   - Configure with your ScrapeOps API credentials.  
   - URL parameter: `={{$json.url}}`  
   - Connect input from `Split URLs (loop)`.

7. **Add a Wait node:**  
   - Name: `Polite Delay`  
   - Configure to wait 2 seconds.  
   - Connect input from `ScrapeOps: Fetch HTML`.

8. **Add a Code node:**  
   - Name: `Extract Trending Repos`  
   - Paste the JavaScript code to parse HTML articles extracting repo details, owner, stars, forks, language, description, and rank.  
   - Connect input from `Polite Delay`.

9. **Add a Code node:**  
   - Name: `Normalize + Enrich-from-page`  
   - Paste the JavaScript code to parse numbers, calculate scores, keyword matches, and generate dedupe keys.  
   - Connect input from `Extract Trending Repos`.

10. **Add a Google Sheets node:**  
    - Name: `Write to Google Sheets (raw tab)`  
    - Credential: Your Google Sheets OAuth2 credentials.  
    - Document ID: ID of your target Google Sheet.  
    - Sheet Name: `trending_raw` (ensure sheet exists with matching columns).  
    - Operation: `Append or Update Row`  
    - Matching Columns: `dedupe_key`  
    - Map fields from input JSON to sheet columns as per the node configuration.  
    - Connect input from `Normalize + Enrich-from-page`.

11. **Optional Weekly Summary Generation:**

    - **Add Google Sheets node:**  
      - Name: `Read raw (for weekly)`  
      - Credential: same Google Sheets credentials.  
      - Document ID: same as above.  
      - Sheet Name: `trending_raw`  
      - Operation: Read all rows.  
      - Connect input from `Set Inputs`. (Runs in parallel with "Build URLs".)

    - **Add Code node:**  
      - Name: `Generate Weekly`  
      - Paste the JavaScript code to generate weekly summary including top repos and themes.  
      - Connect input from `Read raw (for weekly)`.

    - **Add Google Sheets node:**  
      - Name: `Write to Google Sheets (weekly tab)`  
      - Credential: same as above.  
      - Document ID: same as above.  
      - Sheet Name: `weekly_brief`  
      - Operation: Append row (no matching columns).  
      - Map summary fields (`week_id`, `notes`, `top_repos`, `top_themes`, `generated_at`).  
      - Connect input from `Generate Weekly`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Free ScrapeOps account & API key needed: https://scrapeops.io/app/register/n8n                                                                                 | Setup ScrapeOps credentials for reliable scraping.                                                                       |
| ScrapeOps n8n node documentation: https://scrapeops.io/docs/n8n/overview/                                                                                      | Details about using ScrapeOps node in n8n.                                                                                |
| Duplicate this Google Sheet template with correct tabs and columns: https://docs.google.com/spreadsheets/d/1GhCbbPilZXMVDox0hQ0Ncqf5-g3AdFFy55Ld30gPD-E/edit?usp=sharing | Contains `trending_raw` and `weekly_brief` sheets with required columns and formatting.                                   |
| Use `dedupe_key` column in Google Sheets for upserting to avoid duplicates                                                                                     | Ensures no duplicate rows when appending or updating data.                                                                |
| Adjust `since` input to daily, weekly, or monthly as needed                                                                                                   | Customizes trending window of GitHub Trending.                                                                            |
| Add or remove languages in `languages_csv` input                                                                                                             | Filters trending repos by specific programming languages.                                                                 |
| Consider increasing polite delay if GitHub blocks scraping                                                                                                   | Helps avoid rate limits or temporary bans from GitHub.                                                                    |
| Watch for changes in GitHub Trending HTML structure which may require updating regex in "Extract Trending Repos" node                                         | Workflow depends heavily on stable HTML scraping; changes can break parsing.                                              |

---

**Disclaimer:** The provided text exclusively derives from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.