Extract Zillow Property Data to Google Sheets with Scrape.do

https://n8nworkflows.xyz/workflows/extract-zillow-property-data-to-google-sheets-with-scrape-do-9182


# Extract Zillow Property Data to Google Sheets with Scrape.do

### 1. Workflow Overview

This workflow automates the extraction of property data from Zillow URLs listed in a Google Sheets document, using the Scrape.do API for web scraping. It reads URLs from a Google Sheet, scrapes each Zillow page, parses key property details from the HTML response, and writes the structured results back to another Google Sheets tab. The workflow is designed for real estate data collection, market analysis, or lead generation tasks where multiple Zillow listings need to be processed efficiently.

Logical blocks:

- **1.1 Input Reception:** Manual trigger initiates the workflow.
- **1.2 Reading Input Data:** Reads the Zillow URLs from a specified Google Sheets document.
- **1.3 Scraping Zillow Pages:** Uses Scrape.do API to fetch HTML content of each Zillow listing.
- **1.4 Validation Check:** Confirms successful scraping via HTTP status code.
- **1.5 Data Parsing:** Parses raw HTML to extract key property information such as price, address, days on Zillow, and Zestimate.
- **1.6 Output Writing:** Appends the parsed property data back into a separate Google Sheets tab.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Initiates the workflow manually, allowing user control over when the workflow runs.
- **Nodes Involved:**  
  - When clicking 'Test workflow'

- **Node Details:**  
  - **When clicking 'Test workflow'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow execution on user demand.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Read Zillow URLs from Google Sheets" node.  
    - Edge Cases: None specific, but workflow will not run without manual activation.

---

#### 2.2 Reading Input Data

- **Overview:** Retrieves the list of Zillow URLs from a Google Sheets document to process.
- **Nodes Involved:**  
  - Read Zillow URLs from Google Sheets

- **Node Details:**  
  - **Read Zillow URLs from Google Sheets**  
    - Type: Google Sheets node  
    - Role: Reads rows containing Zillow URLs from a specific Sheet tab.  
    - Configuration:  
      - Document ID points to a Google Sheets file with Zillow URLs.  
      - Sheet name corresponds to the tab with input URLs (gid=0).  
      - Reads all rows without stopping at the first match.  
    - Credentials: OAuth2 for Google Sheets (named "VisaTrack Sheets").  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: Passes URLs to HTTP Request node for scraping.  
    - Edge Cases:  
      - Authentication failure with Google Sheets OAuth.  
      - Empty or malformed URLs in the sheet.  
      - Large data sets may require pagination or batching (not implemented here).

---

#### 2.3 Scraping Zillow Pages

- **Overview:** Fetches the raw HTML content of each Zillow URL using the Scrape.do scraping API.
- **Nodes Involved:**  
  - Scrape Zillow URL via Scrape.do

- **Node Details:**  
  - **Scrape Zillow URL via Scrape.do**  
    - Type: HTTP Request node  
    - Role: Calls Scrape.do API to scrape the Zillow listing page HTML.  
    - Configuration:  
      - URL dynamically constructed using the Zillow URL from the previous node, URL-encoded and passed as a query parameter to Scrape.do.  
      - Timeout set to 120 seconds to allow for slower responses.  
      - Redirects are not followed.  
      - Authentication: HTTP query authentication used (credential named "Query Auth account 2").  
    - Inputs: Receives Zillow URLs from Google Sheets node.  
    - Outputs: Passes API response to the "Check if Scraping Succeeded" node.  
    - Edge Cases:  
      - Network timeouts or slow API responses.  
      - API rate limits or authentication errors.  
      - Unexpected API payload structure.

---

#### 2.4 Validation Check

- **Overview:** Verifies that the Scrape.do API call succeeded by checking for HTTP 200 status.
- **Nodes Involved:**  
  - Check if Scraping Succeeded

- **Node Details:**  
  - **Check if Scraping Succeeded**  
    - Type: If node  
    - Role: Filters the flow to continue only if the scrape was successful (HTTP status 200).  
    - Configuration: String equality condition checking if `statusCode` equals "200".  
    - Inputs: Receives scraping responses from HTTP Request node.  
    - Outputs: If true, continue to parse Zillow data; else, workflow stops or could be extended for error handling.  
    - Edge Cases:  
      - Non-200 status codes due to failures or anti-bot blocks.  
      - Missing or malformed status code fields.

---

#### 2.5 Data Parsing

- **Overview:** Extracts structured property data from the raw HTML response using JavaScript code.
- **Nodes Involved:**  
  - Parse Zillow Data

- **Node Details:**  
  - **Parse Zillow Data**  
    - Type: Code node (JavaScript)  
    - Role: Parses key data points from raw HTML text, formats them into JSON.  
    - Configuration:  
      - Extracts price using multiple regex patterns to improve matching reliability.  
      - Parses address components (street, city, state) from a regex matching common US addresses.  
      - Extracts days listed on Zillow and Zestimate values if present.  
      - Captures timestamp of scraping.  
      - Uses the original URL from the Google Sheets input for reference.  
    - Inputs: Receives raw HTML from the "Check if Scraping Succeeded" node.  
    - Outputs: Structured JSON object with fields: URL, Price, Address, City, State, Days on Zillow, Zestimate, Scraped At.  
    - Edge Cases:  
      - Missing or unexpected HTML structure causing regex misses.  
      - HTML encoding issues or incomplete pages.  
      - Input JSON path assumptions (like accessing previous node’s JSON) could cause errors if not consistent.

---

#### 2.6 Output Writing

- **Overview:** Appends the parsed Zillow property data into an output Google Sheets tab for record keeping.
- **Nodes Involved:**  
  - Write Results to Google Sheets

- **Node Details:**  
  - **Write Results to Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends each parsed property data record as a new row in a designated sheet tab.  
    - Configuration:  
      - Document ID same as input sheet.  
      - Sheet tab identified by numeric gid for "Outcome".  
      - Mapping mode set to auto-map fields from incoming JSON data.  
    - Credentials: OAuth2 Google Sheets (same as input).  
    - Inputs: Receives structured JSON from "Parse Zillow Data" node.  
    - Outputs: End of workflow.  
    - Edge Cases:  
      - Google Sheets API quota limits.  
      - Write failures due to locked sheets or permission issues.  
      - Data type mismatches if input data structure changes.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                    | Input Node(s)                   | Output Node(s)                  | Sticky Note                                               |
|-------------------------------|---------------------|----------------------------------|--------------------------------|--------------------------------|-----------------------------------------------------------|
| When clicking 'Test workflow'  | Manual Trigger      | Workflow manual start trigger    | —                              | Read Zillow URLs from Google Sheets |                                                           |
| Read Zillow URLs from Google Sheets | Google Sheets     | Reads Zillow URLs from sheet     | When clicking 'Test workflow'  | Scrape Zillow URL via Scrape.do | Credential: VisaTrack Sheets (OAuth2)                     |
| Scrape Zillow URL via Scrape.do| HTTP Request        | Scrapes Zillow URLs via API      | Read Zillow URLs from Google Sheets | Check if Scraping Succeeded | Auth: Query Auth account 2 (HTTP Query Auth)              |
| Check if Scraping Succeeded    | If                  | Validates scraping success       | Scrape Zillow URL via Scrape.do | Parse Zillow Data              | Checks for HTTP status code 200                            |
| Parse Zillow Data              | Code (JavaScript)   | Parses property data from HTML   | Check if Scraping Succeeded    | Write Results to Google Sheets | Uses regex to extract price, address, days on Zillow, Zestimate |
| Write Results to Google Sheets | Google Sheets       | Appends parsed data to sheet     | Parse Zillow Data              | —                              | Credential: VisaTrack Sheets (OAuth2)                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger  
   - No configuration needed. This node starts the workflow.

2. **Create Google Sheets Node to Read Input URLs**
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Document ID: Use your Google Sheets document ID containing Zillow URLs.  
   - Sheet Name: Use the tab with URLs, e.g., gid=0 or the tab name.  
   - Return First Match: False (read all rows).  
   - Credentials: Set up Google Sheets OAuth2 credentials (named e.g., "VisaTrack Sheets").  
   - Connect Manual Trigger node output to this node.

3. **Create HTTP Request Node for Scrape.do API**
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: Use expression to dynamically build URL:  
     `https://api.scrape.do/?url={{ encodeURIComponent($json.URLs) }}&super=true`  
   - Timeout: 120000 ms (120 seconds).  
   - Follow Redirects: Disabled.  
   - Authentication: HTTP Query Authentication  
   - Credentials: Create and use credentials for Scrape.do API (e.g., "Query Auth account 2").  
   - Connect Google Sheets read node output to this node.

4. **Create If Node to Validate Scraping Success**
   - Type: If  
   - Condition: String equals  
   - Value 1: `{{$json["statusCode"]}}`  
   - Value 2: `200`  
   - Connect HTTP Request node output to this node.

5. **Create Code Node to Parse Zillow Data**
   - Type: Code (JavaScript)  
   - Paste JavaScript code that:  
     - Extracts HTML from the scrape response.  
     - Uses regex patterns to extract price, address, city, state, days on Zillow, Zestimate.  
     - Returns JSON object with structured property data and timestamp.  
   - Connect If node "true" output to this node.

6. **Create Google Sheets Node to Write Output Data**
   - Type: Google Sheets  
   - Operation: Append Rows  
   - Document ID: Same as input sheet or another designated spreadsheet.  
   - Sheet Name: Use the tab for output data (e.g., gid=2048497939 or named tab "Outcome").  
   - Mapping Mode: Auto map input data fields to sheet columns.  
   - Credentials: Same Google Sheets OAuth2 credentials.  
   - Connect Code node output to this node.

7. **Save and Activate Workflow**

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow depends on the Scrape.do API, which may have rate limits and requires API key authentication. | https://scrape.do documentation and account setup                                                     |
| Google Sheets OAuth2 credentials must have proper permissions to read and write to the specified sheets. | Google Cloud Console API & Credentials Setup                                                          |
| Regex patterns used in parsing may need updating if Zillow changes website structure or HTML format. | Regex debugging and adjustment recommended when scraping fails                                        |
| For large volumes, consider batching URLs or adding error handling for failed scrapes or API limits. | Workflow scalability and robustness considerations                                                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.