Yelp Business Scraper by URL with Bright Data API and Google Sheets

https://n8nworkflows.xyz/workflows/yelp-business-scraper-by-url-with-bright-data-api-and-google-sheets-6373


# Yelp Business Scraper by URL with Bright Data API and Google Sheets

### 1. Workflow Overview

This workflow automates scraping detailed business information from Yelp pages given their URLs, leveraging the Bright Data scraping API and storing results in a centralized Google Sheets document. It is designed for users who want to submit a Yelp business URL via a form and receive structured data extracted from that page, including ratings, reviews count, images, and other business details.

Logical blocks:

- **1.1 Input Reception:** Captures Yelp business URL input from the user via a form trigger.
- **1.2 Scraping Trigger:** Sends the URL to Bright Data API to initiate the scraping process.
- **1.3 Scrape Status Monitoring:** Periodically checks if Bright Data has completed scraping via snapshot status polling, including wait and retry logic.
- **1.4 Data Retrieval:** Retrieves the scraped business data once ready.
- **1.5 Data Storage:** Appends the structured business data into a configured Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block waits for the user to submit a Yelp business URL through a form and triggers the workflow upon submission.
- **Nodes Involved:**  
  - 📥 Form Trigger  
  - Sticky Note (explaining the start trigger)

- **Node Details:**

  - **📥 Form Trigger**  
    - Type: Form Trigger (Webhook-based input capture)  
    - Configuration: Form titled "Yelp Service URL" with one field labeled "URL" for user input  
    - Key expressions: None (direct user input)  
    - Input: External form submission (webhook)  
    - Output: Passes form data (URL) to next node  
    - Edge cases: Invalid or empty URL submission, webhook connectivity issues, form field misconfiguration  
    - Version: 2.2  

  - **Sticky Note**  
    - Purpose: Descriptive note stating the workflow starts when user submits a Yelp business URL via form.

---

#### 1.2 Scraping Trigger

- **Overview:** This block sends the submitted Yelp business URL to Bright Data’s dataset API to start scraping the business page.
- **Nodes Involved:**  
  - 🔍 Trigger Bright Data Scrape  
  - Sticky Note (describes sending URL to Bright Data)

- **Node Details:**

  - **🔍 Trigger Bright Data Scrape**  
    - Type: HTTP Request (POST)  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Body: JSON array with the business URL to scrape (hardcoded example URL replaced by dynamic expression in practice)  
      - Query Parameters: dataset_id (Bright Data dataset ID), include_errors, limit_multiple_results, limit_per_input  
      - Headers: Authorization Bearer token referencing Bright Data API key credential  
    - Key expressions: The URL should be dynamically replaced with the form input URL (currently hardcoded in JSON body, should be parameterized)  
    - Input: Yelp URL from form trigger  
    - Output: Snapshot initiation response including snapshot_id  
    - Edge cases: Authentication failure, invalid URL format, API rate limits, network timeouts, JSON parse errors  
    - Version: 4.2  

  - **Sticky Note**  
    - Purpose: Explains this node sends the URL to Bright Data to begin scraping.

---

#### 1.3 Scrape Status Monitoring

- **Overview:** This block periodically checks the status of the Bright Data snapshot to determine if scraping is complete, using a wait-and-retry loop.
- **Nodes Involved:**  
  - 📡 Monitor Snapshot Status  
  - ⏳ Wait 30 Sec for Snapshot  
  - 🔁 Retry Until Ready (If node)  
  - Sticky Notes (explaining status check, wait, and retry logic)

- **Node Details:**

  - **📡 Monitor Snapshot Status**  
    - Type: HTTP Request (GET)  
    - Configuration:  
      - URL template: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (uses snapshot_id from previous node)  
      - Headers: Authorization Bearer token  
      - Always outputs data to allow conditional checking  
    - Input: Output from scraping trigger with snapshot_id  
    - Output: JSON including status field (e.g., "ready" or other statuses)  
    - Edge cases: Invalid snapshot_id, API errors, unauthorized access, network issues  
    - Version: 4.2  

  - **⏳ Wait 30 Sec for Snapshot**  
    - Type: Wait Node  
    - Configuration: Waits for 30 seconds before triggering the next check  
    - Input: From Monitor Snapshot Status node  
    - Output: Triggers Retry Until Ready node after wait  
    - Edge cases: Workflow timeouts if waiting too long, unhandled status states  
    - Version: 1.1  

  - **🔁 Retry Until Ready** (If node)  
    - Type: If Condition  
    - Configuration: Checks if status equals "ready" in the JSON response  
    - Outputs:  
      - True: Proceed to fetch data  
      - False: Loop back to Monitor Snapshot Status node to continue polling  
    - Edge cases: Status values other than expected ("ready"), infinite looping if status never resolves  
    - Version: 2.2  

  - **Sticky Notes**  
    - One describes checking whether the Bright Data snapshot is finished processing  
    - One explains waiting 30 seconds before re-checking  
    - One explains looping back if not ready, otherwise proceeding  

---

#### 1.4 Data Retrieval

- **Overview:** Once the snapshot status is "ready," this block fetches the scraped business data in JSON format.
- **Nodes Involved:**  
  - 📥 Fetch Scraped Business Data  
  - Sticky Note (explains data retrieval from snapshot)

- **Node Details:**

  - **📥 Fetch Scraped Business Data**  
    - Type: HTTP Request (GET)  
    - Configuration:  
      - URL template: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
      - Query Parameter: format=json  
      - Headers: Authorization Bearer token  
    - Input: Snapshot ID from readiness check  
    - Output: Structured business data JSON for downstream processing  
    - Edge cases: Snapshot not found, partial data, API limits, JSON parse errors  
    - Version: 4.2  

  - **Sticky Note**  
    - Describes that this node retrieves structured business details from the ready snapshot.

---

#### 1.5 Data Storage

- **Overview:** This block appends the scraped Yelp business data into a specific Google Sheet for centralized storage and future reference.
- **Nodes Involved:**  
  - 📊 Store to Google Sheet  
  - Sticky Note (explains Google Sheets update)

- **Node Details:**

  - **📊 Store to Google Sheet**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append rows  
      - Sheet Name: "gid=0" (Sheet1)  
      - Document ID: User’s Google Sheet ID (configured via credential)  
      - Columns defined: name, overall_rating, reviews_count, url, images_videos_urls  
      - Fields mapped from scraped data JSON  
      - Credentials: Google Sheets OAuth2 credentials required  
    - Input: Business data JSON from data retrieval node  
    - Output: Updated Google Sheet with new business entry  
    - Edge cases: Authentication failure, sheet not found, permission denied, data format mismatch  
    - Version: 4.6  

  - **Sticky Note**  
    - Explains that this node updates the Google Sheet with business data including name, website, phone, rating, address, etc.

---

### 3. Summary Table

| Node Name                     | Node Type            | Functional Role                    | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                         |
|-------------------------------|----------------------|----------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| Sticky Note                   | Sticky Note          | Describes workflow start trigger | -                           | 📥 Form Trigger                | Starts the workflow when the user submits a business Yelp URL through the form.                    |
| 📥 Form Trigger               | Form Trigger         | Receives user Yelp URL input     | Sticky Note                  | 🔍 Trigger Bright Data Scrape  |                                                                                                   |
| Sticky Note1                  | Sticky Note          | Describes scraping initiation    | 🔍 Trigger Bright Data Scrape | 🔍 Trigger Bright Data Scrape  | Sends the URL to Bright Data to begin scraping the Yelp business page.                            |
| 🔍 Trigger Bright Data Scrape | HTTP Request         | Initiates Yelp page scrape       | 📥 Form Trigger              | 📡 Monitor Snapshot Status     |                                                                                                   |
| Sticky Note2                  | Sticky Note          | Describes snapshot status check  | 📡 Monitor Snapshot Status   | 📡 Monitor Snapshot Status     | Checks whether the Bright Data snapshot is finished processing.                                   |
| 📡 Monitor Snapshot Status    | HTTP Request         | Polls scrape progress            | 🔍 Trigger Bright Data Scrape | ⏳ Wait 30 Sec for Snapshot    |                                                                                                   |
| ⏳ Wait 30 Sec for Snapshot   | Wait                 | Waits before retrying status     | 📡 Monitor Snapshot Status   | 🔁 Retry Until Ready           | Waits 30 seconds before re-checking the scrape status.                                           |
| Sticky Note3                  | Sticky Note          | Describes wait node              | ⏳ Wait 30 Sec for Snapshot  | ⏳ Wait 30 Sec for Snapshot    |                                                                                                   |
| 🔁 Retry Until Ready          | If                   | Checks if scrape is ready        | ⏳ Wait 30 Sec for Snapshot  | 📥 Fetch Scraped Business Data / 📡 Monitor Snapshot Status | If Bright Data says the data is not ready, loops back to wait again. If ready, proceeds.           |
| Sticky Note4                  | Sticky Note          | Describes retry logic            | 🔁 Retry Until Ready         | 🔁 Retry Until Ready           |                                                                                                   |
| 📥 Fetch Scraped Business Data| HTTP Request         | Retrieves scraped business data | 🔁 Retry Until Ready         | 📊 Store to Google Sheet       | Retrieves structured business details from the ready snapshot.                                   |
| Sticky Note5                  | Sticky Note          | Describes data retrieval         | 📥 Fetch Scraped Business Data | 📊 Store to Google Sheet     |                                                                                                   |
| 📊 Store to Google Sheet      | Google Sheets        | Appends data to Google Sheets    | 📥 Fetch Scraped Business Data | -                           | 📊 Updates your Google Sheet with business data (name, website, phone, rating, address, etc.)     |
| Sticky Note6                  | Sticky Note          | Describes data storage           | 📊 Store to Google Sheet     | -                             |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Type: Form Trigger  
   - Configure Form Title as "Yelp Service URL"  
   - Add one form field: Label "URL" (text input)  
   - Save and note webhook URL for external form submission.

2. **Create HTTP Request node to trigger Bright Data scrape:**  
   - Name: "🔍 Trigger Bright Data Scrape"  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Headers: Authorization Bearer token (use Bright Data API key credential)  
   - Query Parameters:  
     - `dataset_id` = your Bright Data dataset ID (e.g. gd_lgugwl0519h1p14rwk)  
     - `include_errors` = true  
     - `limit_multiple_results` = 5  
     - `limit_per_input` = 20  
   - Body (JSON): Array with an object containing `"url": "={{ $json.URL }}"` to use the submitted form URL dynamically  
   - Connect output of Form Trigger to this node.

3. **Create HTTP Request node to monitor snapshot status:**  
   - Name: "📡 Monitor Snapshot Status"  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (use snapshot_id from previous node)  
   - Headers: Authorization Bearer token (Bright Data API key)  
   - Set to always output data  
   - Connect output of scraping trigger node here.

4. **Create Wait node:**  
   - Name: "⏳ Wait 30 Sec for Snapshot"  
   - Set to wait 30 seconds  
   - Connect output of Monitor Snapshot Status node here.

5. **Create If node to check snapshot readiness:**  
   - Name: "🔁 Retry Until Ready"  
   - Condition: Check if `{{$json.status}}` equals "ready"  
   - True branch: proceeds to fetch data node  
   - False branch: loops back to Monitor Snapshot Status node (creating a retry loop)  
   - Connect output of Wait node to this If node.

6. **Create HTTP Request node to fetch scraped business data:**  
   - Name: "📥 Fetch Scraped Business Data"  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query Parameter: `format=json`  
   - Headers: Authorization Bearer token (Bright Data API key)  
   - Connect true output of If node here.

7. **Create Google Sheets node to append data:**  
   - Name: "📊 Store to Google Sheet"  
   - Operation: Append  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: Sheet1 (or "gid=0")  
   - Columns mapped: name, overall_rating, reviews_count, url, images_videos_urls (ensure fields map from scraped data correctly)  
   - Credential: Google Sheets OAuth2 with appropriate permissions  
   - Connect output of Fetch Scraped Business Data node here.

8. **Add Sticky Notes as needed:**  
   - To explain workflow sections: Input reception, scraping trigger, status monitoring, waiting logic, data retrieval, and data storage.

9. **Finalize connections:**  
   - Connect Form Trigger → Scraping Trigger → Monitor Snapshot Status → Wait → Retry If node  
   - Retry If node false branch → Monitor Snapshot Status (loop)  
   - Retry If node true branch → Fetch Data → Google Sheets

10. **Test workflow:**  
    - Submit a Yelp business URL via the form  
    - Monitor execution and confirm data appends to Google Sheet  

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                               |
|----------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow uses Bright Data’s dataset API for scraping Yelp business pages by URL.             | Bright Data API documentation: https://brightdata.com/docs  |
| Google Sheets OAuth2 credentials must have write access to the target sheet.                 | Google API OAuth2 setup guide: https://developers.google.com/sheets/api/guides/authorizing |
| Form Trigger node’s webhook URL must be exposed externally or via n8n cloud for user input.  | n8n Form Trigger documentation: https://docs.n8n.io/nodes/n8n-nodes-base.formTrigger/ |
| Potential infinite loop if Bright Data snapshot status never resolves to "ready". Consider timeout logic. | Best practice for polling loops.                              |
| Ensure API keys and credentials are stored securely in n8n credentials manager.               | n8n security best practices.                                  |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.