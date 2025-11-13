TikTok Post Scraper via Keywords | Bright Data + Sheets Integration

https://n8nworkflows.xyz/workflows/tiktok-post-scraper-via-keywords---bright-data---sheets-integration-5153


# TikTok Post Scraper via Keywords | Bright Data + Sheets Integration

# TikTok Post Scraper via Keywords | Bright Data + Sheets Integration

---

### 1. Workflow Overview

This workflow automates the collection of TikTok posts based on user-provided keywords by leveraging Bright Data's scraping APIs and storing the results in Google Sheets. It is designed for social media analysts, marketers, or researchers who want to gather TikTok data without manual scraping.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** A form trigger node accepts keyword input from the user to initiate the process.
- **1.2 Scraping Trigger:** The keyword is sent to Bright Data's Discover API to start scraping TikTok posts related to that keyword.
- **1.3 Scraping Progress Monitoring:** Repeatedly checks the scraping task status via Bright Data's API until the scraping is complete.
- **1.4 Data Retrieval and Storage:** Once scraping is done, the workflow fetches the scraped data (snapshot) and appends or updates it in a designated Google Sheet.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
  This block captures the keyword input from the user via a web form, acting as the workflow’s entry point.

- **Nodes Involved:**  
  - Accepts keyword input from the user.

- **Node Details:**

  - **Accepts keyword input from the user**  
    - *Type & Role:* Form Trigger node — listens for user submissions through a form interface.  
    - *Configuration:* Single form field labeled "Keyword" to capture input.  
    - *Expressions/Variables:* Outputs the entered keyword in `$json.Keyword`.  
    - *Connections:* Output connected to the "Sends keyword to Bright Data's Discover API to start scraping." node.  
    - *Failure Modes:* Webhook trigger failures, missing input validation, or form submission errors. Requires the webhook to be publicly accessible.  
    - *Version:* 2.2

---

#### 2.2 Scraping Trigger

- **Overview:**  
  This block sends the user-provided keyword to Bright Data's Discover API to initiate the scraping process.

- **Nodes Involved:**  
  - Sends keyword to Bright Data's Discover API to start scraping.

- **Node Details:**

  - **Sends keyword to Bright Data's Discover API to start scraping.**  
    - *Type & Role:* HTTP Request node — performs POST request to Bright Data to trigger scraping.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Headers: Authorization with Bearer token for Bright Data API credentials (`BRIGHT_DATA_API_KEY`).  
      - Query parameters: dataset_id, include_errors, type (discover_new), discover_by (keyword), limit_per_input (2).  
      - Body: JSON array with three identical objects each containing `"search_keyword": "{{ $json.Keyword }}"` and empty country field.  
    - *Expressions/Variables:* Injects the keyword dynamically from the form input `$json.Keyword`.  
    - *Connections:* Output connects to "Checks the scraping status using snapshot ID." node.  
    - *Failure Modes:* Authorization errors, invalid API key, network timeouts, malformed JSON body, or API rate limits.  
    - *Version:* 4.2

---

#### 2.3 Scraping Progress Monitoring

- **Overview:**  
  This block repeatedly checks the status of the scraping task by querying Bright Data's progress API using the snapshot ID until scraping completes.

- **Nodes Involved:**  
  - Checks the scraping status using snapshot ID.  
  - Check Final Status (If node).  
  - Wait 1 minute.

- **Node Details:**

  - **Checks the scraping status using snapshot ID.**  
    - *Type & Role:* HTTP Request node — sends GET request to Bright Data API to check scraping progress.  
    - *Configuration:*  
      - URL template: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
      - Header: Authorization with Bearer token (`BRIGHT_DATA_API_KEY`).  
    - *Expressions/Variables:* Uses dynamic snapshot ID from previous API response `$json.snapshot_id`.  
    - *Connections:* Output connects to "Check Final Status" node.  
    - *Failure Modes:* Missing or invalid snapshot ID, expired tokens, network failures.  
    - *Version:* 4.2  
    - *Always Output Data:* true (to allow conditional checks even on empty responses)

  - **Check Final Status**  
    - *Type & Role:* If node — evaluates if the scraping status equals `"ready"`.  
    - *Configuration:* Condition checks if `$json.status == "ready"`.  
    - *Connections:*  
      - True branch: leads to "Decode Snapshot from Response" node (to retrieve final data).  
      - False branch: leads to "Wait 1 minute" node (to retry later).  
    - *Failure Modes:* Missing status field or unexpected status values.  
    - *Version:* 2.2

  - **Wait 1 minute**  
    - *Type & Role:* Wait node — delays workflow execution for 1 minute before retrying.  
    - *Configuration:* Wait unit set to minutes, amount set to 1.  
    - *Connections:* Loops back to "Check Final Status" node after delay for re-evaluation.  
    - *Failure Modes:* Node execution interruptions or timeout in n8n.  
    - *Version:* 1.1

---

#### 2.4 Data Retrieval and Storage

- **Overview:**  
  Once scraping is complete, this block retrieves the scraped TikTok posts snapshot and appends or updates the data in a Google Sheets document.

- **Nodes Involved:**  
  - Decode Snapshot from Response.  
  - Google Sheets.

- **Node Details:**

  - **Decode Snapshot from Response**  
    - *Type & Role:* HTTP Request node — fetches the final scraped data snapshot from Bright Data.  
    - *Configuration:*  
      - Method: GET  
      - URL: `https://api.brightdata.com/datasets/v3/snapshot/s_mbghznsg225nzgyhz8` (static snapshot path).  
      - Query parameter: `format=json`.  
      - Header: Authorization with Bearer token (`BRIGHT_DATA_API_KEY`).  
    - *Expressions/Variables:* Static URL, no dynamic input in this node.  
    - *Connections:* Output goes to "Google Sheets" node.  
    - *Failure Modes:* 404 if snapshot ID is invalid, authorization failure, network issues.  
    - *Version:* 4.2

  - **Google Sheets**  
    - *Type & Role:* Google Sheets node — appends or updates the scraped TikTok post data into a sheet.  
    - *Configuration:*  
      - Operation: appendOrUpdate  
      - Sheet Name: "Tiktok by keyword" (tab ID 1067534155)  
      - Document ID: Your Google Sheet ID (placeholder `YOUR_GOOGLE_SHEET_ID`)  
      - Columns mapped explicitly from JSON fields such as url, post_id, description, create_time, digg_count, share_count, collect_count, comment_count, play_count, video_duration, hashtags, original_sound, profile_id, profile_username, profile_url, profile_avatar, profile_biography, preview_image, post_type, discovery_input, offical_item, secu_id, original_item, shortcode, width, ratio, video_url, music, cdn_url, is_verified, account_id, profile_followers, tt_chain_token, region, timestamp, input.  
      - Matching column for updates: "url"  
      - Credential: Google Sheets OAuth2 credentials (placeholder `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID`).  
    - *Expressions/Variables:* Maps multiple properties from the scraped JSON `$json` to columns.  
    - *Connections:* Terminal node (no outputs).  
    - *Failure Modes:* Authorization errors, quota limits, network issues, schema mismatch.  
    - *Version:* 4.6

---

### 3. Summary Table

| Node Name                                             | Node Type              | Functional Role                             | Input Node(s)                        | Output Node(s)                          | Sticky Note                                                                                      |
|-------------------------------------------------------|------------------------|---------------------------------------------|------------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------|
| Accepts keyword input from the user.                  | Form Trigger           | Entry point - receives keyword input       | -                                  | Sends keyword to Bright Data's Discover API to start scraping. | 🟥 1. Form Trigger Node: User submits a keyword input to start scraping process.                 |
| Sends keyword to Bright Data's Discover API to start scraping. | HTTP Request           | Triggers TikTok scraping via Bright Data   | Accepts keyword input from the user. | Checks the scraping status using snapshot ID. | 🟩 2. HTTP Request (Trigger Scraping on Bright Data): Sends keyword to Bright Data to trigger scraping. |
| Checks the scraping status using snapshot ID.         | HTTP Request           | Checks scraping progress status             | Sends keyword to Bright Data's Discover API to start scraping. | Check Final Status                        | 🟦 3. Snapshot Progress Check: Fetches snapshot_id and checks if scraping is complete.          |
| Check Final Status                                     | If                     | Evaluates if scraping is complete           | Checks the scraping status using snapshot ID. | Decode Snapshot from Response (true branch) / Wait 1 minute (false branch) | 🟪 4. IF Node (Check Completion): If snapshot ready → continue, else wait and retry.            |
| Wait 1 minute                                         | Wait                   | Pauses 1 minute before retrying             | Check Final Status (false branch)  | Check Final Status                      | 🟥 5. Wait Node: Delays by 1 minute before rechecking scraping status.                           |
| Decode Snapshot from Response                         | HTTP Request           | Retrieves final scraped TikTok data         | Check Final Status (true branch)   | Google Sheets                         |                                                                                                |
| Google Sheets                                        | Google Sheets          | Stores TikTok data in Google Sheets         | Decode Snapshot from Response       | -                                      | 🟨 6. Final Data Storage: Stores extracted TikTok data with full post metrics and profile info. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure a form with one field: Label "Keyword" (string input)  
   - This node acts as the entry point and exposes a webhook URL for external submissions.

2. **Create HTTP Request Node to Trigger Scraping**  
   - Type: HTTP Request  
   - Name: "Sends keyword to Bright Data's Discover API to start scraping."  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Headers: Add `Authorization: Bearer BRIGHT_DATA_API_KEY` (replace with your Bright Data API key)  
   - Query Parameters:  
     - dataset_id: `gd_lu702nij2f790tmv9h`  
     - include_errors: `true`  
     - type: `discover_new`  
     - discover_by: `keyword`  
     - limit_per_input: `2`  
   - Body: Raw JSON, set "Specify Body" to JSON, with the following content, using expression for keyword:  
     ```json
     [
       {
         "search_keyword": "{{ $json.Keyword }}",
         "country": ""
       },
       {
         "search_keyword": "{{ $json.Keyword }}",
         "country": ""
       },
       {
         "search_keyword": "{{ $json.Keyword }}",
         "country": ""
       }
     ]
     ```  
   - Connect output of Form Trigger to this node.

3. **Create HTTP Request Node to Check Scraping Status**  
   - Type: HTTP Request  
   - Name: "Checks the scraping status using snapshot ID."  
   - Method: GET  
   - URL: Use expression: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Headers: Add `Authorization: Bearer BRIGHT_DATA_API_KEY`  
   - Enable "Always Output Data" to allow conditional checks regardless of response.  
   - Connect output of the Scraping Trigger node to this node.

4. **Create If Node to Evaluate Scraping Completion**  
   - Type: If  
   - Name: "Check Final Status"  
   - Condition: Check if `$json.status` equals `"ready"`  
   - Connect output of Progress Check node to this node.

5. **Create Wait Node**  
   - Type: Wait  
   - Name: "Wait 1 minute"  
   - Unit: Minutes  
   - Amount: 1  
   - Connect False branch of If node to this node.  
   - Connect output of Wait node back to the input of the If node to create a loop.

6. **Create HTTP Request Node to Retrieve Final Data Snapshot**  
   - Type: HTTP Request  
   - Name: "Decode Snapshot from Response"  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/s_mbghznsg225nzgyhz8` (replace static part with dynamic snapshot ID if needed)  
   - Query Parameter: `format=json`  
   - Headers: Add `Authorization: Bearer BRIGHT_DATA_API_KEY`  
   - Connect True branch of If node to this node.

7. **Create Google Sheets Node to Store Data**  
   - Type: Google Sheets  
   - Name: "Google Sheets"  
   - Operation: appendOrUpdate  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: "Tiktok by keyword" (or your chosen sheet/tab)  
   - Mapping: Map all TikTok data fields from the incoming JSON to columns as listed below:  
     - url, post_id, description, create_time, digg_count, share_count, collect_count, comment_count, play_count, video_duration, hashtags, original_sound, profile_id, profile_username, profile_url, profile_avatar, profile_biography, preview_image, post_type, discovery_input, offical_item, secu_id, original_item, shortcode, width, ratio, video_url, music, cdn_url, is_verified, account_id, profile_followers, tt_chain_token, region, timestamp, input  
   - Matching Columns: Use "url" to determine updates vs appends  
   - Credentials: Link your Google Sheets OAuth2 credentials  
   - Connect output of Snapshot Retrieval node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The Bright Data API requires a valid API key and correct dataset IDs. Make sure to replace placeholders accordingly. | Bright Data API documentation: https://brightdata.com/docs    |
| Google Sheets credentials must be set with OAuth2 and have edit permissions on the target spreadsheet.         | Google Sheets API documentation: https://developers.google.com/sheets/api |
| The workflow includes retry logic with a 1-minute delay to handle asynchronous scraping completion.            | Enables robust handling of scraping delays and prevents flooding Bright Data API. |
| The TikTok data schema is extensive; ensure your Google Sheet columns match the mapped fields exactly.         | Helps avoid data import errors or mismatches.                  |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.