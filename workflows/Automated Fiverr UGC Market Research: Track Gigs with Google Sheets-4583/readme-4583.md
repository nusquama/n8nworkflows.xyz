Automated Fiverr UGC Market Research: Track Gigs with Google Sheets

https://n8nworkflows.xyz/workflows/automated-fiverr-ugc-market-research--track-gigs-with-google-sheets-4583


# Automated Fiverr UGC Market Research: Track Gigs with Google Sheets

### 1. Workflow Overview

This workflow automates the process of scraping Fiverr for User Generated Content (UGC) gigs, extracting relevant data from the search results, and logging this data into a Google Sheet for tracking and analysis. It is designed to run daily without manual intervention, providing an up-to-date dataset of Fiverr gigs related to UGC creators.

The workflow can be logically divided into two main blocks:

- **1.1 Trigger & Data Fetching**: This block handles the scheduled start of the workflow and performs an HTTP GET request to Fiverr’s search page to retrieve gig listings related to UGC content creators.

- **1.2 HTML Parsing & Data Logging**: This block parses the raw HTML response to extract structured gig information (price, title, seller, URL) and appends this data to a specified Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Fetching

- **Overview:**  
  This block initiates the workflow on a schedule (daily at 9 AM UTC) and fetches raw HTML search results from Fiverr for gigs related to UGC content creators.

- **Nodes Involved:**  
  - Daily Fiverr Scrape Trigger  
  - Fetch Fiverr Search Results

- **Node Details:**

  - **Daily Fiverr Scrape Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow automatically every day at 09:00 AM UTC.  
    - *Configuration:* Scheduled with a fixed interval triggering once daily at the 9th hour (9 AM).  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Triggers the next node to fetch Fiverr data.  
    - *Edge Cases:* Trigger may fail if the n8n instance is offline or misconfigured schedule; no direct error handling needed.  
    - *Notes:* Acts as the entry point to automate the entire process without manual start.

  - **Fetch Fiverr Search Results**  
    - *Type:* HTTP Request  
    - *Role:* Sends an HTTP GET request to Fiverr’s search endpoint to retrieve gig listings for the query “ugc”.  
    - *Configuration:*  
      - URL: `https://www.fiverr.com/search/gigs`  
      - Query Parameter: `query=ugc` (search term)  
      - Method: GET  
      - Headers: Includes User-Agent, Accept, and Accept-Language headers to mimic a standard browser request and reduce scraping blocks.  
    - *Inputs:* Activated by the Schedule Trigger node.  
    - *Outputs:* Raw HTML response from Fiverr’s search page, passed to the HTML parsing node.  
    - *Edge Cases:*  
      - Potential IP blocking by Fiverr if too many requests occur or scraping is detected.  
      - HTML structure might change, affecting downstream parsing.  
      - Possible network timeouts or HTTP errors.  
    - *Pro Tips:* Consider rotating user agents or using proxies to improve reliability and avoid blocking.  
    - *Notes:* The query parameter can be adjusted to refine the search (e.g., different keywords, categories).

---

#### 2.2 HTML Parsing & Data Logging

- **Overview:**  
  This block extracts key gig information from the raw HTML and appends the structured data into a Google Sheet for tracking purposes.

- **Nodes Involved:**  
  - Extract Data from HTML  
  - Append Gig Data to Sheet

- **Node Details:**

  - **Extract Data from HTML**  
    - *Type:* HTML Extract  
    - *Role:* Parses the HTML content to extract gig details using CSS selectors.  
    - *Configuration:*  
      - Extraction operation set to "extractHtmlContent".  
      - Extracted fields with CSS selectors:  
        - Price: `div.basic-gig-card a._Z7OVIW span.text-bold > span`  
        - Title: `div.basic-gig-card p.f2YMuU6`  
        - Seller: `div.basic-gig-card a._1lc1p3l2 span.vp9lqtk`  
        - Gig URL: `div.basic-gig-card > a.media` (link to the gig)  
    - *Inputs:* Receives raw HTML from the HTTP Request node.  
    - *Outputs:* Structured JSON objects with the extracted gig data fields.  
    - *Edge Cases:*  
      - If Fiverr changes the site’s HTML or CSS classes, extraction will fail or produce empty data—selectors must be updated accordingly.  
      - Malformed or incomplete HTML may cause partial or failed extraction.  
    - *Notes:* Regular maintenance is required to keep selectors up-to-date.

  - **Append Gig Data to Sheet**  
    - *Type:* Google Sheets (Append Operation)  
    - *Role:* Appends the extracted gig data as new rows into a specified Google Sheet.  
    - *Configuration:*  
      - Document ID: Google Sheet ID specified (linked to "Fiverr UGC Scraper" sheet).  
      - Sheet Name: Uses the first sheet (`gid=0`).  
      - Columns mapped to extracted fields: Price, Title, Seller Name, Gig URL.  
      - Mapping mode: Defined explicitly for each column.  
      - Credential: Uses OAuth2 credentials configured for Google Sheets access.  
    - *Inputs:* Receives structured gig data from the HTML Extract node.  
    - *Outputs:* None (end of workflow).  
    - *Edge Cases:*  
      - Google API quota limits or authentication failures.  
      - Incorrect sheet permissions or invalid document ID.  
      - Data type mismatches or empty fields.  
    - *Notes:*  
      - Consider adding error handling for quota or auth issues.  
      - Optionally enhance with deduplication or timestamping for historical tracking.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                            | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                  |
|---------------------------|---------------------|------------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Daily Fiverr Scrape Trigger| Schedule Trigger    | Starts workflow daily at 9 AM UTC         | None                        | Fetch Fiverr Search Results | ⚙️ Section 1: **Trigger & Data Fetching** - Automatically starts the workflow daily.          |
| Fetch Fiverr Search Results| HTTP Request        | Fetches Fiverr gigs for query "ugc"       | Daily Fiverr Scrape Trigger | Extract Data from HTML      | ⚙️ Section 1: **Trigger & Data Fetching** - Fetches Fiverr gig listings with headers set.    |
| Extract Data from HTML     | HTML Extract        | Parses HTML to extract gig details         | Fetch Fiverr Search Results | Append Gig Data to Sheet    | 🧠 Section 2: **HTML Parsing & Data Logging** - Extracts price, title, seller, gig URL.       |
| Append Gig Data to Sheet   | Google Sheets       | Appends extracted gig data to Google Sheet| Extract Data from HTML      | None                       | 🧠 Section 2: **HTML Parsing & Data Logging** - Logs data into Google Sheets for analysis.    |
| Sticky Note                | Sticky Note         | Informational note                        | None                        | None                       | ⚙️ Section 1: **Trigger & Data Fetching** detailed explanation.                              |
| Sticky Note1               | Sticky Note         | Informational note                        | None                        | None                       | 🧠 Section 2: **HTML Parsing & Data Logging** detailed explanation.                          |
| Sticky Note4               | Sticky Note         | Full section overview combined           | None                        | None                       | Combined detailed notes for both sections 1 and 2, including tips and summary.               |
| Sticky Note9               | Sticky Note         | Workflow assistance contact and links    | None                        | None                       | Contact info and useful links for support and further learning by Yaron Been.                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Daily Fiverr Scrape Trigger`  
   - Set to trigger every 24 hours at 09:00 AM UTC  
   - No credentials needed  
   - This node will start the workflow daily.

3. **Add an HTTP Request node:**  
   - Name: `Fetch Fiverr Search Results`  
   - Connect input from `Daily Fiverr Scrape Trigger` output  
   - Method: GET  
   - URL: `https://www.fiverr.com/search/gigs`  
   - Query Parameters:  
     - Key: `query`  
     - Value: `ugc`  
   - Headers:  
     - `User-Agent`: Use a common browser user-agent string (e.g., Chrome on Windows)  
     - `Accept`: `text/html`  
     - `Accept-Language`: `en-US,en;q=0.9`  
   - No authentication required  
   - Set to return the full HTML response for parsing.

4. **Add an HTML Extract node:**  
   - Name: `Extract Data from HTML`  
   - Connect input from `Fetch Fiverr Search Results` output  
   - Operation: Extract HTML content  
   - Add extraction values with keys and CSS selectors:  
     - `price`: `div.basic-gig-card a._Z7OVIW span.text-bold > span`  
     - `title`: `div.basic-gig-card p.f2YMuU6`  
     - `Selller`: `div.basic-gig-card a._1lc1p3l2 span.vp9lqtk`  
     - `Gig URL`: `div.basic-gig-card > a.media`  
   - No credentials necessary  
   - This node outputs structured gig data for further use.

5. **Add a Google Sheets node:**  
   - Name: `Append Gig Data to Sheet`  
   - Connect input from `Extract Data from HTML` output  
   - Operation: Append  
   - Google Sheets OAuth2 credential setup required: create or select a credential with access to your Google Drive and Sheets.  
   - Document ID: Enter the ID of the Google Sheet where data will be appended  
   - Sheet Name: Use the first sheet or specify by GID (e.g., `gid=0`)  
   - Define columns and map to extracted fields:  
     - Price → `price`  
     - Title → `title`  
     - Seller Name → `Selller`  
     - Gig URL → `Gig URL`  
   - Configure mapping mode as "define below" to explicitly map fields.

6. **Save and activate the workflow.**

7. **Optional: Add sticky notes for documentation and clarity.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Workflow assistance and support contact: Yaron@nofluff.online                                                | Contact email for workflow questions/support                                                                        |
| Explore more tips and tutorials by Yaron Been                                                                | YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/               |
| Consider adding enhancements such as deduplication, alerts (email/Slack), and data cleanup in Google Sheets | Suggested improvements to increase workflow robustness and usability                                                 |
| Keep CSS selectors updated regularly due to possible Fiverr UI changes                                        | Maintenance note to avoid data extraction failures                                                                  |
| Use proxies or rotate user agents to avoid IP blocks during scraping                                          | Practical scraping tip to improve HTTP request reliability                                                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.