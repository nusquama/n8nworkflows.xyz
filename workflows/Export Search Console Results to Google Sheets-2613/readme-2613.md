Export Search Console Results to Google Sheets

https://n8nworkflows.xyz/workflows/export-search-console-results-to-google-sheets-2613


# Export Search Console Results to Google Sheets

### 1. Workflow Overview

This n8n workflow automates the extraction of Google Search Console (GSC) performance data and exports the results into Google Sheets for easier visualization and SEO analysis. By using scheduled triggers, it fetches data on search queries, page performance, and daily metrics from Google Search Console via API calls and writes this structured data directly into designated Google Sheets documents.

The workflow is logically divided into these blocks:

- **1.1 Scheduling and Domain Setup:** Defines when the workflow runs and sets the target domain and date range.
- **1.2 Data Retrieval from Search Console:** Makes HTTP requests to GSC API to pull query, page, and date dimension data.
- **1.3 Data Transformation:** Splits and formats the raw API data into well-structured records.
- **1.4 Export to Google Sheets:** Writes or updates the structured data into corresponding Google Sheets tabs.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Domain Setup

- **Overview:**  
This block triggers the workflow on a schedule and sets key parameters such as the target domain and the number of days for which data should be fetched.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set your domain

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution based on a recurring schedule (default unspecified interval).  
    - Config: Uses n8n's scheduling with no specific interval set in JSON (user-configurable).  
    - Inputs: None (starting node)  
    - Outputs: Connects to "Set your domain"  
    - Edge Cases: Misconfiguration may cause workflow to never trigger or trigger too frequently.

  - **Set your domain**  
    - Type: Set (Data assignment)  
    - Role: Assigns workflow variables for domain and days to fetch data for.  
    - Config: Sets "domain" to "funautomations.io" and "days" to 30 by default.  
    - Inputs: From Schedule Trigger  
    - Outputs: Feeds three HTTP Request nodes for query, page, and date reports.  
    - Edge Cases: Invalid domain input may cause API calls to fail; days set to too large a number may cause excessive data or API quota issues.

---

#### 1.2 Data Retrieval from Search Console

- **Overview:**  
This block makes three separate HTTP POST requests to the Google Search Console API, each querying different dimensions ("query", "page", and "date") for the specified domain and date range.

- **Nodes Involved:**  
  - Get query Report  
  - Get Page Report  
  - date (HTTP Request node for date dimension)

- **Node Details:**

  - **Get query Report**  
    - Type: HTTP Request  
    - Role: Retrieves search analytics data aggregated by search queries.  
    - Config: POST to `https://www.googleapis.com/webmasters/v3/sites/sc-domain:<domain>/searchAnalytics/query`  
      - Body includes startDate (current date), endDate (current date minus days), dimensions: ["query"].  
    - Auth: Google OAuth2 API credentials (OAuth account)  
    - Inputs: From "Set your domain"  
    - Outputs: Connects to "Split Out" node for processing response.  
    - Edge Cases: OAuth token expiration, API quota limits, invalid domain, malformed date.

  - **Get Page Report**  
    - Type: HTTP Request  
    - Role: Retrieves search analytics data aggregated by page URLs.  
    - Config: Same endpoint as above, dimension set to ["page"].  
    - Auth: Google OAuth2 API credentials.  
    - Inputs: From "Set your domain"  
    - Outputs: Connects to "Split Out1" node.  
    - Edge Cases: Same as "Get query Report".

  - **date**  
    - Type: HTTP Request  
    - Role: Retrieves search analytics data aggregated by date.  
    - Config: Similar to above, dimension set to ["date"].  
    - Auth: Google OAuth2 API credentials.  
    - Inputs: From "Set your domain"  
    - Outputs: Connects to "Split Out2" node.  
    - Edge Cases: Same as above. This node has `retryOnFail` enabled on downstream Google Sheets node to handle transient errors.

---

#### 1.3 Data Transformation

- **Overview:**  
Each HTTP response contains arrays of data rows. This block splits these arrays into individual items and maps specific fields (keyword, page, date, clicks, impressions, CTR, position) into a clean format for exporting.

- **Nodes Involved:**  
  - Split Out  
  - Edit Fields  
  - Split Out1  
  - Edit Fields1  
  - Split Out2  
  - Edit Fields2

- **Node Details:**

  - **Split Out, Split Out1, Split Out2**  
    - Type: SplitOut  
    - Role: Each splits the "rows" array from the HTTP responses into individual JSON objects for processing.  
    - Config: Splits on field "rows".  
    - Inputs: From respective HTTP Request nodes.  
    - Outputs: To respective Edit Fields nodes.  
    - Edge Cases: If "rows" field is missing or empty, downstream nodes receive no data.

  - **Edit Fields, Edit Fields1, Edit Fields2**  
    - Type: Set (Field mapping)  
    - Role: Extracts and maps key metrics from each row item into named fields for Google Sheets.  
    - Configurations:  
      - Edit Fields (query data): maps "Keyword" from keys[0], clicks, impressions, ctr, position.  
      - Edit Fields1 (page data): maps "page" from keys[0], clicks, impressions, ctr, position.  
      - Edit Fields2 (date data): maps "date" from keys[0], clicks, impressions, ctr, position.  
    - Inputs: From respective Split Out nodes.  
    - Outputs: To respective Google Sheets update nodes.  
    - Expressions: Use `{{$json.keys[0]}}` to extract dimension value.  
    - Edge Cases: Missing or unexpected keys array format in data could cause expression errors.

---

#### 1.4 Export to Google Sheets

- **Overview:**  
Writes the processed data into specific tabs/sheets within Google Sheets documents. Uses append or update operations keyed on dimension columns to maintain data integrity.

- **Nodes Involved:**  
  - Update queries to Sheets  
  - Update Pages to Sheets  
  - Update date report to sheets

- **Node Details:**

  - **Update queries to Sheets**  
    - Type: Google Sheets  
    - Role: Appends or updates keyword query data into the "Query" sheet/tab.  
    - Config:  
      - Document ID: URL to shared Google Sheet.  
      - Sheet Name: "Query" tab identified by sheet ID 996986484.  
      - Columns: Keyword, clicks, impressions, ctr, position.  
      - Matching Columns: Keyword (to update existing rows).  
      - Operation: appendOrUpdate.  
    - Inputs: From Edit Fields (query data).  
    - Auth: Google Sheets OAuth2 credentials.  
    - Edge Cases: Incorrect sheet URL or permissions cause write failure; data type mismatches.

  - **Update Pages to Sheets**  
    - Type: Google Sheets  
    - Role: Appends or updates page-level data into the "PAGES" sheet/tab.  
    - Config:  
      - Document ID: Same Google Sheet URL.  
      - Sheet Name: "PAGES" tab (gid=0).  
      - Columns and matching column: page, clicks, impressions, ctr, position.  
    - Inputs: From Edit Fields1 (page data).  
    - Auth: Google Sheets OAuth2 credentials.  
    - Edge Cases: Same as above.

  - **Update date report to sheets**  
    - Type: Google Sheets  
    - Role: Appends or updates date-aggregated data into the "Dates" sheet/tab.  
    - Config:  
      - Document ID: Same Google Sheet URL.  
      - Sheet Name: "Dates" tab (gid=1823079319).  
      - Columns and matching column: date, clicks, impressions, ctr, position.  
    - Inputs: From Edit Fields2 (date data).  
    - Auth: Google Sheets OAuth2 credentials.  
    - Retry on Fail: Enabled to handle transient failures.  
    - Edge Cases: Same as above; retry helps mitigate temporary API or network issues.

---

### 3. Summary Table

| Node Name              | Node Type         | Functional Role                                  | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                          |
|------------------------|-------------------|-------------------------------------------------|-------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger  | Starts workflow on defined schedule             | None                    | Set your domain                 |                                                                                                    |
| Set your domain        | Set               | Sets domain and days parameters                  | Schedule Trigger        | Get query Report, Get Page Report, date | Sticky Note1: Setup instructions, usage, and troubleshooting details.                              |
| Get query Report       | HTTP Request      | Fetches search query data from GSC API           | Set your domain         | Split Out                      |                                                                                                    |
| Split Out              | Split Out         | Splits query data array into single entries      | Get query Report        | Edit Fields                   |                                                                                                    |
| Edit Fields            | Set               | Maps query data fields for Google Sheets         | Split Out               | Update queries to Sheets       |                                                                                                    |
| Update queries to Sheets| Google Sheets     | Writes query data into "Query" sheet              | Edit Fields             | None                          |                                                                                                    |
| Get Page Report        | HTTP Request      | Fetches page data from GSC API                    | Set your domain         | Split Out1                    |                                                                                                    |
| Split Out1             | Split Out         | Splits page data array into single entries       | Get Page Report         | Edit Fields1                  |                                                                                                    |
| Edit Fields1           | Set               | Maps page data fields for Google Sheets          | Split Out1              | Update Pages to Sheets         |                                                                                                    |
| Update Pages to Sheets | Google Sheets     | Writes page data into "PAGES" sheet                | Edit Fields1            | None                          |                                                                                                    |
| date                   | HTTP Request      | Fetches date-aggregated data from GSC API         | Set your domain         | Split Out2                    |                                                                                                    |
| Split Out2             | Split Out         | Splits date data array into single entries       | date                    | Edit Fields2                  |                                                                                                    |
| Edit Fields2           | Set               | Maps date data fields for Google Sheets          | Split Out2              | Update date report to sheets   |                                                                                                    |
| Update date report to sheets | Google Sheets | Writes date data into "Dates" sheet                | Edit Fields2            | None                          |                                                                                                    |
| Sticky Note            | Sticky Note       | Visual notes for workflow organization            | None                    | None                          |                                                                                                    |
| Sticky Note1           | Sticky Note       | Contains detailed usage, setup, and troubleshooting instructions | None                    | None                          | Contains link to Google Sheet template and setup instructions.                                     |
| Sticky Note17          | Sticky Note       | Workflow title visual label                        | None                    | None                          | "# Search console REPORTS"                                                                          |
| Sticky Note2           | Sticky Note       | Notes on setting domain and day frequency          | None                    | None                          |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set desired schedule interval (e.g., daily).  
   - This node will start the workflow automatically.

3. **Add a Set node named "Set your domain":**  
   - Create two fields:  
     - `domain` (string), e.g., "funautomations.io"  
     - `days` (number), e.g., 30  
   - Connect Schedule Trigger output to this node.

4. **Add three HTTP Request nodes for data retrieval:**

   - **Get query Report:**  
     - Method: POST  
     - URL: `https://www.googleapis.com/webmasters/v3/sites/sc-domain:{{$json.domain}}/searchAnalytics/query`  
     - Body (JSON):  
       ```json
       {
         "startDate": "{{ $now.format('yyyy-MM-dd') }}",
         "endDate": "{{ $now.minus($json.days, 'days').format('yyyy-MM-dd') }}",
         "dimensions": ["query"]
       }
       ```  
     - Authentication: Use Google OAuth2 credentials with Search Console API scopes.  
     - Connect output from "Set your domain" node.

   - **Get Page Report:**  
     - Same as above, but `"dimensions": ["page"]`.

   - **date:**  
     - Same as above, but `"dimensions": ["date"]`.

5. **Add three Split Out nodes:**

   - Each configured to split the HTTP Request response field `rows`.  
   - Connect each corresponding HTTP Request output to its Split Out node.

6. **Add three Set (Edit Fields) nodes to map fields:**

   - For query data:  
     - Create fields: Keyword, clicks, impressions, ctr, position.  
     - Map: Keyword = `{{$json.keys[0]}}`  
     - Other fields map from `$json.clicks`, `$json.impressions`, etc.  
     - Connect to Split Out (query).

   - For page data:  
     - Fields: page, clicks, impressions, ctr, position.  
     - Map page = `{{$json.keys[0]}}`  
     - Connect to Split Out1.

   - For date data:  
     - Fields: date, clicks, impressions, ctr, position.  
     - Map date = `{{$json.keys[0]}}`  
     - Connect to Split Out2.

7. **Add three Google Sheets nodes to export data:**

   - For queries:  
     - Document URL: Your Google Sheet URL (make a copy of the example sheet).  
     - Sheet Name: "Query" tab (use correct sheet ID or name).  
     - Operation: appendOrUpdate  
     - Columns: Keyword, clicks, impressions, ctr, position  
     - Matching Column: Keyword  
     - Connect to Edit Fields (query).

   - For pages:  
     - Same document, Sheet Name: "PAGES" (gid=0)  
     - Columns: page, clicks, impressions, ctr, position  
     - Matching Column: page  
     - Connect to Edit Fields1.

   - For dates:  
     - Same document, Sheet Name: "Dates" (gid=1823079319)  
     - Columns: date, clicks, impressions, ctr, position  
     - Matching Column: date  
     - Enable retry on fail.  
     - Connect to Edit Fields2.

8. **Configure Google OAuth2 credentials in n8n:**

   - Create credentials with scopes:  
     - `https://www.googleapis.com/auth/webmasters`  
     - `https://www.googleapis.com/auth/webmasters.readonly`  
     - `https://www.googleapis.com/auth/spreadsheets`  
   - Assign these credentials to HTTP Request and Google Sheets nodes accordingly.

9. **Adjust the domain and days in "Set your domain" node as needed.**

10. **Optionally, customize the schedule in "Schedule Trigger" to your preferred frequency.**

11. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Make a copy of the Google Sheet template used in this workflow: https://docs.google.com/spreadsheets/d/10hSuGOOf14YvVY2Bw8WXUIpsyXO614l7qNEjkyVY_Qg/edit?usp=sharing                                                                                                                                       | Google Sheets template for importing Search Console data                                                                 |
| Set Google OAuth2 credentials with scopes for Search Console and Sheets API access: `https://www.googleapis.com/auth/webmasters`, `https://www.googleapis.com/auth/webmasters.readonly`, `https://www.googleapis.com/auth/adwords`                                                                           | Required OAuth2 scopes for API authentication                                                                              |
| Common troubleshooting: check OAuth token validity, Google Sheets URL and access permissions, schedule conflicts                                                                                                                                                                                        | Troubleshooting tips for execution issues                                                                                  |
| The workflow is well-suited for SEO specialists and marketers to monitor site performance trends without logging into Search Console manually                                                                                                                     | Use case context                                                                                                          |
| The workflow uses "appendOrUpdate" in Google Sheets nodes keyed on dimension columns to avoid duplicate entries and keep data up to date                                                                                                                         | Data consistency approach                                                                                                 |

---

This completes the structured, detailed reference documentation for the "Export Search Console Results to Google Sheets" n8n workflow.