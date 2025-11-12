Scrape LinkedIn Job Listings for Hiring Signals & Prospecting with Bright Data

https://n8nworkflows.xyz/workflows/scrape-linkedin-job-listings-for-hiring-signals---prospecting-with-bright-data-3580


# Scrape LinkedIn Job Listings for Hiring Signals & Prospecting with Bright Data

### 1. Workflow Overview

This workflow automates the process of scraping recent LinkedIn job listings via Bright Data’s Dataset API, cleaning the data, and exporting it to Google Sheets. It serves two main use cases: helping job seekers find fresh, filtered job openings, and enabling sales or prospecting teams to identify companies actively hiring as buying signals.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input via a form trigger to specify job search filters (location, keyword, country).
- **1.2 Trigger Bright Data Snapshot:** Sends a POST request to Bright Data’s Dataset API to start a snapshot based on the input filters.
- **1.3 Polling & Snapshot Status Check:** Waits and polls Bright Data until the snapshot is ready.
- **1.4 Data Retrieval:** Fetches the completed snapshot data from Bright Data.
- **1.5 Data Cleaning:** Processes and flattens nested fields, strips HTML tags from job descriptions.
- **1.6 Export to Google Sheets:** Appends the cleaned job listings to a pre-configured Google Sheet.
- **1.7 Guidance & Documentation:** Sticky notes provide embedded instructions, API parameter explanations, and resource links.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user input through a form to define job search parameters such as location, keyword, and country code.

- **Nodes Involved:**  
  - `On form submission - Discover Jobs` (Form Trigger)  
  - `Edit Fields` (Set Node)

- **Node Details:**

  - **On form submission - Discover Jobs**  
    - Type: Form Trigger  
    - Role: Entry point capturing user input for job search filters.  
    - Configuration:  
      - Form titled "Linkedin High Intent Prospects And Job Post Hunt"  
      - Fields: Job Location (required), Keyword (required), Country (2 letters, required)  
      - Webhook ID configured for external trigger.  
    - Inputs: External form submission  
    - Outputs: JSON object with form fields  
    - Edge Cases: Missing required fields will prevent trigger; malformed input may cause API query issues.

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Prepares or resets any additional fields before sending to Bright Data (currently sets an empty string, possibly placeholder for future use).  
    - Inputs: Output from form trigger  
    - Outputs: Passes data unchanged to next node  
    - Edge Cases: None significant; can be extended for input validation or enrichment.

---

#### 2.2 Trigger Bright Data Snapshot

- **Overview:**  
  Sends a POST request to Bright Data’s Dataset API to initiate a new snapshot of LinkedIn job listings based on user filters.

- **Nodes Involved:**  
  - `HTTP Request- Post API call to Bright Data`

- **Node Details:**

  - **HTTP Request- Post API call to Bright Data**  
    - Type: HTTP Request  
    - Role: Initiates dataset snapshot on Bright Data with user-defined filters.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Headers: Authorization with Bearer token (Bright Data API Key)  
      - Body (JSON): Includes dynamic expressions for location, keyword, country from form input; fixed filters for `time_range` ("Past 24 hours") and `job_type` ("Part-time"); other filters empty.  
      - Query Parameters: dataset_id, endpoint, notify URL (webhook for notification), format=json, uncompressed webhook=true, type=discover_new, discover_by=keyword  
    - Inputs: JSON from `Edit Fields` node  
    - Outputs: JSON containing snapshot_id and status  
    - Edge Cases:  
      - Authentication failure if API key invalid or missing  
      - Network timeouts or API rate limits  
      - Invalid or empty filter values may return empty or broad datasets

---

#### 2.3 Polling & Snapshot Status Check

- **Overview:**  
  Implements a polling loop that waits and repeatedly checks the status of the Bright Data snapshot until it is complete.

- **Nodes Involved:**  
  - `Wait - Polling Bright Data`  
  - `Snapshot Progress` (HTTP Request)  
  - `If - Checking status of Snapshot - if data is ready or not`

- **Node Details:**

  - **Wait - Polling Bright Data**  
    - Type: Wait  
    - Role: Pauses workflow execution for a defined interval (minutes) before polling again.  
    - Configuration: Waits for a specified number of minutes (default unspecified, user can adjust).  
    - Inputs: Output from previous HTTP Request node  
    - Outputs: Triggers `Snapshot Progress` node after wait  
    - Edge Cases:  
      - Excessive waiting may delay workflow  
      - Too short wait intervals may cause API rate limiting

  - **Snapshot Progress**  
    - Type: HTTP Request  
    - Role: Queries Bright Data API for current snapshot status.  
    - Configuration:  
      - URL dynamically constructed using snapshot_id from previous response  
      - Headers: Authorization with Bright Data API Key  
      - Method: GET (default)  
    - Inputs: From Wait node  
    - Outputs: JSON with current snapshot status (e.g., "running", "completed")  
    - Edge Cases:  
      - Snapshot ID missing or invalid  
      - API errors or timeouts

  - **If - Checking status of Snapshot - if data is ready or not**  
    - Type: If  
    - Role: Branches workflow based on snapshot status.  
    - Configuration: Checks if `status` field equals "running".  
    - Inputs: Output from `Snapshot Progress`  
    - Outputs:  
      - True branch: snapshot still running → loops back to `Wait - Polling Bright Data`  
      - False branch: snapshot ready → proceeds to data retrieval  
    - Edge Cases:  
      - Unexpected status values may cause infinite loops or premature exit  
      - Expression evaluation errors if status field missing

---

#### 2.4 Data Retrieval

- **Overview:**  
  Fetches the completed snapshot data (job listings) from Bright Data once ready.

- **Nodes Involved:**  
  - `HTTP Request - Getting data from Bright Data`

- **Node Details:**

  - **HTTP Request - Getting data from Bright Data**  
    - Type: HTTP Request  
    - Role: Downloads the full dataset snapshot JSON from Bright Data.  
    - Configuration:  
      - URL dynamically constructed with snapshot_id  
      - Headers: Authorization with Bright Data API Key  
      - Query parameter: format=json  
      - Method: GET  
    - Inputs: From `If - Checking status` node (false branch)  
    - Outputs: JSON array of job listings  
    - Edge Cases:  
      - Snapshot ID invalid or expired  
      - Large payloads causing timeouts or memory issues  
      - API errors or malformed responses

---

#### 2.5 Data Cleaning

- **Overview:**  
  Processes raw job listing data to flatten nested objects and remove HTML tags from job descriptions for clean output.

- **Nodes Involved:**  
  - `Code - Cleaning Up`

- **Node Details:**

  - **Code - Cleaning Up**  
    - Type: Code (JavaScript)  
    - Role:  
      - Flattens nested fields: `job_poster` (extracts name, title, url), `base_salary` (min, max, currency, period)  
      - Strips HTML tags and entities from `job_description_formatted` to produce `job_description_plain`  
    - Key Code Highlights:  
      - Uses regex to remove HTML tags and decode entities  
      - Deletes original nested fields after flattening  
    - Inputs: JSON array from Bright Data snapshot  
    - Outputs: Cleaned JSON array ready for export  
    - Edge Cases:  
      - Missing nested fields handled gracefully (empty strings)  
      - Unexpected HTML or malformed text may cause incomplete cleaning  
      - Large datasets may impact performance

---

#### 2.6 Export to Google Sheets

- **Overview:**  
  Appends the cleaned job listings as rows into a designated Google Sheet for further use.

- **Nodes Involved:**  
  - `Google Sheets - Adding All Job Posts`

- **Node Details:**

  - **Google Sheets - Adding All Job Posts**  
    - Type: Google Sheets  
    - Role: Inserts job data rows into a Google Sheet tab.  
    - Configuration:  
      - Operation: Append rows  
      - Document ID: Points to the user’s copied Google Sheet template  
      - Sheet Name: First tab (gid=0)  
      - Columns: Auto-mapped from input JSON fields including job_title, company_name, location, salary_min, apply_link, job_description_plain, and many others (total ~30+ fields)  
      - Credential: Google Sheets OAuth2 connected account  
      - Handling Extra Data: Inserts any extra fields into new columns  
    - Inputs: Cleaned job listings JSON from Code node  
    - Outputs: Confirmation of rows appended  
    - Edge Cases:  
      - OAuth token expiration or permission errors  
      - Sheet access issues (wrong ID or permissions)  
      - Large batch inserts may hit API limits or timeouts

---

#### 2.7 Guidance & Documentation (Sticky Notes)

- **Overview:**  
  Provides embedded instructions, API parameter explanations, setup tips, and resource links to assist users in understanding and customizing the workflow.

- **Nodes Involved:**  
  - Multiple `Sticky Note` nodes scattered throughout the canvas, including:  
    - Workflow purpose and overview  
    - API parameter guide with examples  
    - Setup instructions for Google Sheets template  
    - Contact and support information  
    - Tips on filter usage and polling intervals

- **Node Details:**  
  - Type: Sticky Note  
  - Role: Documentation and user guidance  
  - Configuration: Text content with formatting, links, and examples  
  - Inputs/Outputs: None (informational only)  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                           | Input Node(s)                          | Output Node(s)                              | Sticky Note                                                                                      |
|-------------------------------------|---------------------|-----------------------------------------|--------------------------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note                         | Sticky Note         | Workflow overview and instructions      | None                                 | None                                        | LinkedIn Job Data Scraper to Google Sheets; usage, setup, tips, API body example, resources      |
| Sticky Note9                       | Sticky Note         | Workflow assistance contact & links     | None                                 | None                                        | Contact info and links to YouTube and LinkedIn for support                                     |
| On form submission - Discover Jobs | Form Trigger        | Captures user input for job filters     | External form submission             | Edit Fields                                 |                                                                                                 |
| Edit Fields                       | Set                 | Prepares input data for API call         | On form submission                   | HTTP Request- Post API call to Bright Data  |                                                                                                 |
| Sticky Note2                      | Sticky Note         | Explanation of Bright Data API call      | None                                 | None                                        | Bright Data Trigger – Customize Your Job Query; tips on filters and freshness                   |
| HTTP Request- Post API call to Bright Data | HTTP Request        | Triggers Bright Data snapshot            | Edit Fields                         | Wait - Polling Bright Data                   |                                                                                                 |
| Wait - Polling Bright Data         | Wait                | Waits between polling attempts           | HTTP Request- Post API call          | Snapshot Progress                            |                                                                                                 |
| Snapshot Progress                 | HTTP Request        | Checks snapshot status                    | Wait - Polling Bright Data           | If - Checking status of Snapshot             | Sticky Note3: "Bright Data Getting Jobs"                                                       |
| If - Checking status of Snapshot - if data is ready or not | If                  | Branches based on snapshot status        | Snapshot Progress                   | Wait - Polling Bright Data (if running), HTTP Request - Getting data from Bright Data (if ready) |                                                                                                 |
| HTTP Request - Getting data from Bright Data | HTTP Request        | Retrieves completed snapshot data        | If - Checking status (false branch) | Code - Cleaning Up                           |                                                                                                 |
| Code - Cleaning Up                | Code                 | Cleans and flattens job data             | HTTP Request - Getting data          | Google Sheets - Adding All Job Posts         |                                                                                                 |
| Google Sheets - Adding All Job Posts | Google Sheets        | Appends cleaned job listings to sheet   | Code - Cleaning Up                   | None                                        | Sticky Note10: Instructions to use Google Sheets template with link                            |
| Sticky Note1                      | Sticky Note         | API parameter guide and examples         | None                                 | None                                        | Detailed LinkedIn Jobs API parameter guide with JSON examples                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: `On form submission - Discover Jobs`  
   - Configure form title: "Linkedin High Intent Prospects And Job Post Hunt"  
   - Add form fields (all required):  
     - Job Location (placeholder: "example: new york")  
     - Keyword (placeholder: "example: CMO, AI architect")  
     - Country (2 letters) (placeholder: "example: US,UK,IL")  
   - Save and note webhook URL for external triggering.

2. **Add Set Node to Prepare Input**  
   - Type: Set  
   - Name: `Edit Fields`  
   - No changes needed initially; can add or reset fields as required.  
   - Connect output of Form Trigger to this node.

3. **Add HTTP Request Node to Trigger Bright Data Snapshot**  
   - Type: HTTP Request  
   - Name: `HTTP Request- Post API call to Bright Data`  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Headers: Add `Authorization: Bearer YOUR_BRIGHTDATA_API_KEY` (replace with your key)  
   - Query Parameters:  
     - dataset_id: `gd_lpfll7v5hcqtkxl6l`  
     - endpoint: your n8n webhook URL (for notification)  
     - notify: same webhook URL  
     - format: `json`  
     - uncompressed_webhook: `true`  
     - type: `discover_new`  
     - discover_by: `keyword`  
   - Body (JSON):  
     ```json
     [
       {
         "location": "{{ $json['Job Location'] }}",
         "keyword": "{{ $json.Keyword }}",
         "country": "{{ $json['Country (2 letters)'] }}",
         "time_range": "Past 24 hours",
         "job_type": "Part-time",
         "experience_level": "",
         "remote": "",
         "company": ""
       }
     ]
     ```  
   - Connect output of `Edit Fields` to this node.

4. **Add Wait Node for Polling Interval**  
   - Type: Wait  
   - Name: `Wait - Polling Bright Data`  
   - Set unit to minutes (default 1–3 minutes recommended)  
   - Connect output of HTTP Request node to this Wait node.

5. **Add HTTP Request Node to Check Snapshot Progress**  
   - Type: HTTP Request  
   - Name: `Snapshot Progress`  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $('HTTP Request- Post API call to Bright Data').item.json.snapshot_id }}`  
   - Headers: Authorization with Bearer token (same API key)  
   - Connect output of Wait node to this node.

6. **Add If Node to Check Snapshot Status**  
   - Type: If  
   - Name: `If - Checking status of Snapshot - if data is ready or not`  
   - Condition: Check if `status` equals `"running"`  
   - Connect output of `Snapshot Progress` to this node.

7. **Loop Back or Proceed Based on Status**  
   - True branch (status = running): connect back to `Wait - Polling Bright Data` to continue polling.  
   - False branch (status != running): connect to next node to fetch data.

8. **Add HTTP Request Node to Retrieve Snapshot Data**  
   - Type: HTTP Request  
   - Name: `HTTP Request - Getting data from Bright Data`  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Headers: Authorization with Bearer token  
   - Query Parameter: format=json  
   - Connect false branch of If node to this node.

9. **Add Code Node to Clean and Flatten Data**  
   - Type: Code (JavaScript)  
   - Name: `Code - Cleaning Up`  
   - Paste the provided JavaScript code that:  
     - Flattens `job_poster` and `base_salary` fields  
     - Strips HTML tags from `job_description_formatted`  
   - Connect output of data retrieval HTTP Request node to this node.

10. **Add Google Sheets Node to Append Data**  
    - Type: Google Sheets  
    - Name: `Google Sheets - Adding All Job Posts`  
    - Operation: Append  
    - Document ID: Use your copied Google Sheet template ID  
    - Sheet Name: First tab (gid=0)  
    - Map columns automatically from input JSON fields (including job_title, company_name, job_description_plain, salary_min, apply_link, etc.)  
    - Credentials: Connect your Google Sheets OAuth2 account  
    - Connect output of Code node to this node.

11. **Add Sticky Notes for Documentation (Optional)**  
    - Add Sticky Note nodes with content for user guidance, API parameter explanations, and links as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Google Sheet Template for storing job posts: Make a copy to use                                                      | https://docs.google.com/spreadsheets/d/1_jbr5zBllTy_pGbogfGSvyv1_0a77I8tU-Ai7BjTAw4/edit?usp=sharing      |
| Bright Data Dataset API documentation and signup                                                                    | https://brightdata.com                                                                                   |
| Workflow support contact and tutorial links                                                                          | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Recommended filter values for freshness: "Past 24 hours" or "Last 7 days"                                            | Improves relevance and timeliness of job listings                                                       |
| Use cleaned data for personalized outreach: cold emails, LinkedIn messages, lead list building                       | Enhances sales and job hunting effectiveness                                                            |
| Polling interval should be balanced to avoid API rate limits and excessive delays                                    | Adjust Wait node duration accordingly                                                                   |
| Avoid hardcoding filters in form; leave optional filters empty for broader results                                   | Increases flexibility                                                                                     |
| Flatten nested fields and strip HTML tags to ensure clean, readable data in Google Sheets                            | Improves data usability                                                                                   |
| Bright Data snapshots take approximately 1–3 minutes to complete                                                    | Workflow includes automated polling to handle this delay                                                |

---

This completes the comprehensive reference documentation for the "Scrape LinkedIn Job Listings for Hiring Signals & Prospecting with Bright Data" workflow.