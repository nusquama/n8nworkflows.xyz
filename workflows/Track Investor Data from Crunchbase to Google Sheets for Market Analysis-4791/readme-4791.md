Track Investor Data from Crunchbase to Google Sheets for Market Analysis

https://n8nworkflows.xyz/workflows/track-investor-data-from-crunchbase-to-google-sheets-for-market-analysis-4791


# Track Investor Data from Crunchbase to Google Sheets for Market Analysis

### 1. Workflow Overview

This workflow automates the process of collecting investor data from Crunchbase and storing it in a Google Sheets document for market analysis. It is designed for users who want to maintain an up-to-date, easy-to-access investor dataset without manual effort.

The workflow is logically divided into two main blocks:

- **1.1 Data Acquisition from Crunchbase:** Automatically triggers daily and fetches investor organization data via Crunchbase API.
- **1.2 Data Processing and Storage:** Cleans and formats the fetched data, then appends it to a Google Sheet for further analysis or reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Acquisition from Crunchbase

**Overview:**  
This block handles the scheduled triggering of the workflow and the retrieval of investor data from Crunchbase’s API with specific filters applied.

**Nodes Involved:**  
- Daily Investor Data Trigger  
- Fetch Crunchbase Investor Data

**Node Details:**

- **Daily Investor Data Trigger**  
  - *Type & Role:* Schedule Trigger; initiates the workflow automatically at a set time every day.  
  - *Configuration:* Set to trigger once daily at 9 AM (server time).  
  - *Key Parameters:* Interval set with `triggerAtHour: 9`.  
  - *Inputs:* None (start node).  
  - *Outputs:* Triggers `Fetch Crunchbase Investor Data` node.  
  - *Edge Cases:* Workflow will not run if n8n server is down at trigger time or if time zone differences affect scheduling.  
  - *Version:* 1.2

- **Fetch Crunchbase Investor Data**  
  - *Type & Role:* HTTP Request; sends a POST request to Crunchbase API to retrieve investor organization data.  
  - *Configuration:*  
    - URL: `https://api.crunchbase.com/api/v4/searches/organizations`  
    - Method: POST  
    - Request Body: JSON specifying requested fields (`identifier`, `name`, `short_description`, `location_identifiers`, `investment_stage`) and filter for `organization_types` including `investor`.  
    - Limit: 5 results per request (could be parameterized).  
    - Headers: Authorization with Bearer token for Crunchbase API key (placeholder `YOUR_CRUNCHBASE_API_KEY` must be replaced).  
  - *Inputs:* Triggered by schedule node.  
  - *Outputs:* JSON response containing entities array with investor data to `Extract Investor Fields`.  
  - *Edge Cases:*  
    - Authentication errors if API key is invalid or expired.  
    - Rate limiting or API quota exceeded.  
    - Network errors or API downtime.  
    - Response format changes by Crunchbase API.  
  - *Version:* 4.2

---

#### 1.2 Data Processing and Storage

**Overview:**  
This block processes the raw investor data, extracting relevant fields, and appends the cleaned data to a designated Google Sheet.

**Nodes Involved:**  
- Extract Investor Fields  
- Append to Investor Sheet

**Node Details:**

- **Extract Investor Fields**  
  - *Type & Role:* Code node (JavaScript); transforms the Crunchbase API response into a clean, row-ready array of investor objects.  
  - *Configuration:*  
    - JavaScript code accesses `items[0].json.entities`, maps each entity to extract:  
      - `name`  
      - `short_description`  
      - `location_identifiers`  
      - `investment_stage`  
    - Returns an array of JSON objects corresponding to each investor.  
  - *Inputs:* Receives JSON from HTTP Request node.  
  - *Outputs:* Cleaned items to Google Sheets node.  
  - *Edge Cases:*  
    - If `entities` array is missing or empty, returns empty array, resulting in no rows appended.  
    - Property access errors if response structure changes.  
  - *Version:* 2

- **Append to Investor Sheet**  
  - *Type & Role:* Google Sheets node; appends the cleaned investor data rows to a specific sheet.  
  - *Configuration:*  
    - Operation: Append  
    - Document ID: Specified Google Sheets document (ID provided)  
    - Sheet Name: GID=0 (default first sheet)  
    - Columns mapped:  
      - Name -> `{{$json.name}}`  
      - Location -> `{{$json.location_identifiers}}` (array serialized as string)  
      - Investment Stage -> `{{$json.investment_stage}}` (array serialized as string)  
      - Short description -> `{{$json.short_description}}`  
    - Mapping mode: Explicitly defined columns and values.  
    - Credential: Google Sheets OAuth2 account (credential ID provided).  
  - *Inputs:* Receives cleaned data from Code node.  
  - *Outputs:* None (end node).  
  - *Edge Cases:*  
    - Google Sheets API quota exceeded.  
    - Authentication errors with Google OAuth2 credential.  
    - Incorrect document ID or sheet name causing append failure.  
    - Data type mismatches, although conversion to string is disabled (could cause issues if arrays are not stringified properly).  
  - *Version:* 4.5

---

### 3. Summary Table

| Node Name                   | Node Type         | Functional Role                      | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                               |
|-----------------------------|-------------------|------------------------------------|----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Daily Investor Data Trigger  | Schedule Trigger  | Starts workflow daily at 9 AM      | None                       | Fetch Crunchbase Investor Data | **SECTION 1**: Automatically triggers data fetching daily.                                                                 |
| Fetch Crunchbase Investor Data | HTTP Request     | Fetches investor data from Crunchbase API | Daily Investor Data Trigger | Extract Investor Fields    | **SECTION 1**: Pulls live investor data automatically via Crunchbase API.                                                  |
| Extract Investor Fields      | Code              | Cleans and formats raw investor data | Fetch Crunchbase Investor Data | Append to Investor Sheet  | **SECTION 2**: Extracts relevant fields and prepares data for Google Sheets.                                               |
| Append to Investor Sheet     | Google Sheets     | Appends cleaned data to Google Sheet | Extract Investor Fields     | None                       | **SECTION 2**: Saves investor data into a Google Sheets document for analysis and tracking.                                |
| Sticky Note                 | Sticky Note       | Commentary on Section 1             | None                       | None                       | Details on Section 1: Scheduled trigger and Crunchbase data fetch explained.                                               |
| Sticky Note1                | Sticky Note       | Commentary on Section 2             | None                       | None                       | Details on Section 2: Data extraction and Google Sheets append explained.                                                  |
| Sticky Note9                | Sticky Note       | Workflow assistance contact info   | None                       | None                       | Support contact and resources: Yaron@nofluff.online, YouTube, LinkedIn links.                                              |
| Sticky Note4                | Sticky Note       | Full workflow explanation combining both sections | None                       | None                       | Comprehensive workflow overview integrating sections 1 and 2 with benefits and final outcomes.                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Node Type: Schedule Trigger  
   - Name: `Daily Investor Data Trigger`  
   - Parameters: Set to trigger daily at 9:00 AM (hour: 9)  
   - No credentials required.  

2. **Create HTTP Request Node**  
   - Node Type: HTTP Request  
   - Name: `Fetch Crunchbase Investor Data`  
   - Parameters:  
     - URL: `https://api.crunchbase.com/api/v4/searches/organizations`  
     - HTTP Method: POST  
     - Request Body Type: JSON  
     - Body:  
       ```json
       {
         "field_ids": [
           "identifier",
           "name",
           "short_description",
           "location_identifiers",
           "investment_stage"
         ],
         "query": [
           {
             "type": "predicate",
             "field_id": "organization_types",
             "operator_id": "includes",
             "values": ["investor"]
           }
         ],
         "limit": 5
       }
       ```  
     - Headers: Add header `Authorization` with value `Bearer YOUR_CRUNCHBASE_API_KEY` (replace with your valid API key).  
   - Connect input from `Daily Investor Data Trigger`.  

3. **Create Code Node**  
   - Node Type: Code  
   - Name: `Extract Investor Fields`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const entities = items[0].json.entities || [];
     return entities.map(entity => {
       return {
         json: {
           name: entity.name,
           short_description: entity.short_description,
           location_identifiers: entity.location_identifiers,
           investment_stage: entity.investment_stage
         }
       };
     });
     ```  
   - Connect input from `Fetch Crunchbase Investor Data`.  

4. **Create Google Sheets Node**  
   - Node Type: Google Sheets  
   - Name: `Append to Investor Sheet`  
   - Operation: Append  
   - Document ID: Paste your Google Sheets document ID where you want to store the data.  
   - Sheet Name: Use `gid=0` or sheet name as appropriate.  
   - Mapping Mode: Define below  
   - Columns Mapping:  
     - Name: `={{ $json.name }}`  
     - Location: `={{ $json.location_identifiers }}` (note: arrays will be stored as strings)  
     - Investment Stage: `={{ $json.investment_stage }}`  
     - Short description: `={{ $json.short_description }}`  
   - Credentials: Set up and select a valid Google Sheets OAuth2 credential with access to your spreadsheet.  
   - Connect input from `Extract Investor Fields`.  

5. **Connect Nodes in Sequence:**  
   `Daily Investor Data Trigger` → `Fetch Crunchbase Investor Data` → `Extract Investor Fields` → `Append to Investor Sheet`

6. **Credential Setup:**  
   - Crunchbase API Key: Obtain from Crunchbase developer portal and replace placeholder in HTTP Request node.  
   - Google Sheets OAuth2: Create OAuth2 credentials in n8n for your Google account with access to the target spreadsheet.  

7. **Test Run:**  
   - Trigger the workflow manually or wait for scheduled execution.  
   - Verify data appears correctly in Google Sheets.  
   - Check for errors in each node’s execution.  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow designed for fully automated daily investor data refresh from Crunchbase to Google Sheets.                   | Workflow purpose                                                                                |
| Crunchbase API documentation for reference: [Crunchbase API Docs](https://data.crunchbase.com/docs)                   | Crunchbase API usage                                                                            |
| Google Sheets API and OAuth2 setup required for appending data.                                                       | Google Sheets integration                                                                       |
| Workflow assistance and support contact: Yaron@nofluff.online                                                        | Support contact                                                                                |
| Additional learning resources: YouTube channel at https://www.youtube.com/@YaronBeen/videos and LinkedIn at https://www.linkedin.com/in/yaronbeen/ | External resources for n8n workflows and automation tips                                       |
| Ensure to replace placeholder API keys and document IDs with your actual credentials and spreadsheet info.            | Critical setup step                                                                            |
| Potential improvements: pagination handling for Crunchbase API to retrieve more than 5 investors per run.              | Suggested enhancement                                                                          |
| Edge cases to monitor: API rate limits, credential expiry, and data schema changes from Crunchbase API responses.     | Operational risks                                                                             |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.