Automated Freelance Gig Finder with Bright Data & n8n

https://n8nworkflows.xyz/workflows/automated-freelance-gig-finder-with-bright-data---n8n-5214


# Automated Freelance Gig Finder with Bright Data & n8n

### 1. Workflow Overview

This workflow automates the daily scraping of freelance job gigs from the website *We Work Remotely (WWR)* using the Bright Data proxy service to bypass bot detection. It targets users who want to track remote job listings filtered by specific skills (e.g., AI, Python, UI/UX). The workflow is structured into three main logical blocks:

- **1.1 Trigger and Filter Setup:** Automatically triggers the workflow daily and sets the skill filter for job searches.
- **1.2 Scraping and Extraction:** Uses Bright Data to scrape raw HTML job listings from WWR and extracts structured job information.
- **1.3 Data Persistence:** Saves the extracted job data into a Google Sheet for easy viewing, filtering, and record keeping.

This modular approach ensures automated, reliable, and scalable freelance gig tracking with minimal manual intervention.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Filter Setup

**Overview:**  
This block initiates the workflow daily at a scheduled time and defines the skill keyword used to filter job listings on We Work Remotely.

**Nodes Involved:**  
- Run Scraper Daily (Schedule Trigger)  
- Set Skill Filter (Set Node)

**Node Details:**

- **Run Scraper Daily**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow at 09:00 daily (UTC assumed)  
  - Configuration: Trigger set to hour 9 every day  
  - Input: None (start node)  
  - Output: Connects to "Set Skill Filter" node  
  - Potential Failure Modes: Time zone discrepancies may cause unexpected trigger times; node misconfiguration could stop automation.

- **Set Skill Filter**  
  - Type: Set Node  
  - Role: Manually sets the skill keyword to filter job listings (default set to "AI")  
  - Configuration: Assigns a string variable `Skills` with value `"AI"`  
  - Input: Receives trigger from "Run Scraper Daily"  
  - Output: Passes the skill filter keyword to the next HTTP Request node  
  - Potential Failure Modes: If the skill filter is left empty or malformed, resulting queries may be invalid or return no results.

---

#### 1.2 Scraping and Extraction

**Overview:**  
This block performs the core scraping operation using Bright Data's Web Unlocker API to fetch raw HTML from We Work Remotely filtered by the skill keyword, then parses the HTML to extract structured job data.

**Nodes Involved:**  
- Scrape WWR with Bright Data (HTTP Request)  
- Extract Jobs from HTML (HTML Extract)

**Node Details:**

- **Scrape WWR with Bright Data**  
  - Type: HTTP Request  
  - Role: Executes a POST request to Bright Data API to scrape WWR job listings filtered by skill  
  - Configuration:  
    - URL: `https://api.brightdata.com/request`  
    - Method: POST  
    - Body Parameters:  
      - `zone`: `"n8n_unblocker"` (Bright Data zone)  
      - `url`: Constructed dynamically as `https://weworkremotely.com/remote-jobs/search?term={{ $json.Skills }}` to filter jobs by skill  
      - `country`: `"us"` (simulate US location)  
      - `format`: `"raw"` (request raw HTML output)  
    - Headers: Authorization with Bearer token (placeholder `API_KEY` to be replaced)  
  - Input: Receives skill filter from "Set Skill Filter"  
  - Output: Delivers raw HTML content to "Extract Jobs from HTML"  
  - Potential Failure Modes:  
    - Authentication errors if API key is invalid or expired  
    - Network timeouts or Bright Data service disruptions  
    - API rate limits causing request failure  
    - Incorrectly formed URLs if skill filter contains unsafe characters  
  - Version Requirements: HTTP Request node version 4.2 or higher recommended for full compatibility with body and header parameter features.

- **Extract Jobs from HTML**  
  - Type: HTML Extract  
  - Role: Parses raw HTML from Bright Data response and extracts job details (title, company, location) using CSS selectors  
  - Configuration:  
    - Extraction operation: `extractHtmlContent`  
    - Extraction keys and CSS selectors:  
      - `Job`: selects job title text from selector `#category-18 > article > ul > li.new-listing-container.feature > a > div > div.new-listing__header > h4`  
      - `Company`: selects company name text from selector `#category-18 > article > ul > li.new-listing-container.feature > a > div > p.new-listing__company-name`  
      - `Country`: selects location text from selector `#category-18 > article > ul > li.new-listing-container.feature > a > div > p.new-listing__company-headquarters`  
  - Input: Receives raw HTML from "Scrape WWR with Bright Data"  
  - Output: Outputs structured JSON objects with job details to "Save Jobs to Google Sheets"  
  - Potential Failure Modes:  
    - Changes in WWR website HTML structure may invalidate CSS selectors causing extraction failure or no data  
    - Empty or malformed HTML input could break extraction  
    - Node version must support HTML extract operation (version 1.2 or later)  

---

#### 1.3 Data Persistence

**Overview:**  
This block appends the extracted job listings data into a Google Sheet for persistent daily record keeping and easy access.

**Nodes Involved:**  
- Save Jobs to Google Sheets (Google Sheets node)

**Node Details:**

- **Save Jobs to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends each extracted job listing as a new row in a specific Google Sheet  
  - Configuration:  
    - Operation: Append rows  
    - Document ID: `14JmN5gkBRW6Vgevf2oNXXvEuZXPwBCk4JxQqEfJOHSw` (Google Sheet ID)  
    - Sheet Name: `gid=0` (Sheet1)  
    - Columns mapped from JSON fields:  
      - `Job` → "Job" column  
      - `Company` → "Company" column  
      - `Country` → "Location" column  
    - Credential: Uses OAuth2 Google Sheets API credentials named "Google Sheets account"  
  - Input: Receives structured job data from "Extract Jobs from HTML"  
  - Output: None (end of workflow)  
  - Potential Failure Modes:  
    - Authentication errors if OAuth2 token expired or invalid  
    - API quota limits causing request rejection  
    - Google Sheet access permissions issues  
    - Data mapping errors if JSON keys mismatch columns  
  - Node Version: 4.5 or higher recommended for stable Google Sheets integration  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                      | Input Node(s)            | Output Node(s)              | Sticky Note                                                  |
|---------------------------|---------------------|------------------------------------|--------------------------|----------------------------|--------------------------------------------------------------|
| Run Scraper Daily          | Schedule Trigger    | Triggers workflow daily at 09:00   | None                     | Set Skill Filter            | See Sticky Note4 and Sticky Note3 for Section 1 explanation  |
| Set Skill Filter           | Set Node           | Defines skill keyword for filtering| Run Scraper Daily         | Scrape WWR with Bright Data | See Sticky Note4 and Sticky Note3 for Section 1 explanation  |
| Scrape WWR with Bright Data| HTTP Request       | Scrapes WWR job listings via Bright Data API | Set Skill Filter          | Extract Jobs from HTML      | See Sticky Note5 for Section 2 explanation                   |
| Extract Jobs from HTML     | HTML Extract       | Extracts structured jobs from HTML | Scrape WWR with Bright Data | Save Jobs to Google Sheets  | See Sticky Note5 for Section 2 explanation                   |
| Save Jobs to Google Sheets | Google Sheets      | Appends job data to Google Sheet   | Extract Jobs from HTML    | None                       | See Sticky Note6 for Section 3 explanation                   |
| Sticky Note9              | Sticky Note        | Workflow assistance contact info   | None                     | None                       | Contains support contact and useful tutorial links          |
| Sticky Note4              | Sticky Note        | Detailed overview and workflow sections | None                     | None                       | Full workflow assistance and detailed section breakdown     |
| Sticky Note3              | Sticky Note        | Section 1 detailed explanation      | None                     | None                       | Section 1: Trigger + Skill Filter explanation                |
| Sticky Note5              | Sticky Note        | Section 2 detailed explanation      | None                     | None                       | Section 2: Scraping & Extraction explanation                 |
| Sticky Note6              | Sticky Note        | Section 3 detailed explanation      | None                     | None                       | Section 3: Saving to Google Sheets explanation               |
| Sticky Note               | Sticky Note        | Affiliate link for Bright Data      | None                     | None                       | Contains Bright Data referral link                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node "Run Scraper Daily"**  
   - Node Type: Schedule Trigger  
   - Configuration: Set to trigger daily at 09:00 (hour 9)  
   - No credentials required  
   - Connect output to next node.

2. **Create Set Node "Set Skill Filter"**  
   - Node Type: Set  
   - Configuration: Create string field `Skills` with value `"AI"` (or your preferred skill keyword)  
   - Connect input from "Run Scraper Daily" output.

3. **Create HTTP Request Node "Scrape WWR with Bright Data"**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Body Parameters (as form or JSON):  
     - `zone`: `"n8n_unblocker"`  
     - `url`: Expression: `` `https://weworkremotely.com/remote-jobs/search?term=${$json["Skills"]}` ``  
     - `country`: `"us"`  
     - `format`: `"raw"`  
   - Headers:  
     - `Authorization`: `Bearer API_KEY` (replace `API_KEY` with your actual Bright Data API token)  
   - Connect input from "Set Skill Filter" output.

4. **Create HTML Extract Node "Extract Jobs from HTML"**  
   - Node Type: HTML Extract  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `Job` → CSS Selector: `#category-18 > article > ul > li.new-listing-container.feature > a > div > div.new-listing__header > h4`  
     - Key: `Company` → CSS Selector: `#category-18 > article > ul > li.new-listing-container.feature > a > div > p.new-listing__company-name`  
     - Key: `Country` → CSS Selector: `#category-18 > article > ul > li.new-listing-container.feature > a > div > p.new-listing__company-headquarters`  
   - Connect input from "Scrape WWR with Bright Data" output.

5. **Create Google Sheets Node "Save Jobs to Google Sheets"**  
   - Node Type: Google Sheets  
   - Operation: Append  
   - Document ID: `14JmN5gkBRW6Vgevf2oNXXvEuZXPwBCk4JxQqEfJOHSw` (or your own Google Sheet ID)  
   - Sheet Name / GID: `gid=0` for the first sheet  
   - Columns mapping:  
     - `Job` → "Job" column  
     - `Company` → "Company" column  
     - `Country` → "Location" column  
   - Credentials: Select or create Google Sheets OAuth2 credentials with access to the target sheet  
   - Connect input from "Extract Jobs from HTML" output.

6. **(Optional) Add Sticky Notes**  
   - Add descriptive sticky notes for workflow documentation and user guidance as per content in the original workflow.

7. **Save and activate the workflow**  
   - Test with manual trigger or wait for scheduled run  
   - Verify jobs append correctly to Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow assistance contact: Yaron@nofluff.online                                               | For questions or support                                                                        |
| Helpful tutorial videos and tips available on YouTube and LinkedIn by Yaron Been                | YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Bright Data affiliate link to support free content creation                                      | https://get.brightdata.com/1tndi4600b25                                                        |
| This workflow requires valid Bright Data API credentials and Google Sheets OAuth2 credentials    | Setup guides available on respective services' official docs                                  |
| CSS selectors are tightly coupled to current WWR HTML layout and may need updating if site changes| Monitor site changes for scraping reliability                                                  |


---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.