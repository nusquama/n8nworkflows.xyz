Automated Competitor Price Monitoring with Bright Data & n8n

https://n8nworkflows.xyz/workflows/automated-competitor-price-monitoring-with-bright-data---n8n-5219


# Automated Competitor Price Monitoring with Bright Data & n8n

### 1. Workflow Overview

This n8n workflow automates the monitoring of a competitor’s pricing page by scraping data daily, extracting and comparing pricing information, and sending alerts when prices change. It targets businesses or analysts who want to track competitor pricing dynamics efficiently without manual intervention.

The workflow is organized into three main functional blocks:

- **1.1 Scheduled Data Retrieval:** Automatically triggers daily scraping of the competitor’s pricing page using Bright Data’s web unlocking API to bypass bot protection and obtain raw HTML content.

- **1.2 Data Extraction & Comparison:** Parses the HTML to isolate pricing information for specific plans, then compares this fresh data against the last stored prices in Google Sheets to detect any changes.

- **1.3 Alerting & Data Persistence:** If price changes are detected, updates the Google Sheet with the new prices and sends an email alert notification. If no changes are found, the workflow exits gracefully without further action.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Retrieval

- **Overview:**  
  This block runs daily at a set hour to scrape the competitor’s pricing page. It uses Bright Data’s API to fetch the HTML content of the page, circumventing anti-scraping measures.

- **Nodes Involved:**  
  - Check Pricing Every Day  
  - Fetch Page via Bright Data  
  - Extract HTML Content  
  - Format & Isolate Price Block

- **Node Details:**  

  - **Check Pricing Every Day**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates workflow daily at 9:00 AM automatically.  
    - *Config:* Trigger set to fire once every day at hour 9 (09:00).  
    - *Inputs:* None  
    - *Outputs:* Connects to “Fetch Page via Bright Data”  
    - *Edge Cases:* Workflow will not trigger if n8n instance is down or disabled.

  - **Fetch Page via Bright Data**  
    - *Type:* HTTP Request  
    - *Role:* Sends a POST request to Bright Data’s API to scrape the target pricing page.  
    - *Config:*  
      - URL: `https://api.brightdata.com/request`  
      - Method: POST  
      - JSON body specifying:  
        - Zone: `n8n_unblocker`  
        - Target URL: `https://www.shopify.com/uk/pricing`  
        - Country: `us`  
        - Format: `raw` (raw HTML response)  
        - Headers: User-Agent and Accept headers to mimic browser request  
      - Authorization: Bearer token header with API key (replace `API_KEY`)  
    - *Inputs:* Trigger from Schedule  
    - *Outputs:* Passes HTTP response to “Extract HTML Content”  
    - *Edge Cases:*  
      - Authentication failure if API key is invalid or expired  
      - Network timeouts or API rate limits from Bright Data  
      - Unexpected HTML structure if target page changes  
    - *Version:* Uses n8n HTTP Request node version 4.2 features (jsonBody, headerParameters)

  - **Extract HTML Content**  
    - *Type:* HTML Extract  
    - *Role:* Parses the raw HTML to extract arrays of plan names and pricing spans for further processing.  
    - *Config:*  
      - Extraction keys:  
        - `Plan name`: CSS selector `h3` (returns array)  
        - `Pricing`: CSS selector `span` (returns array)  
    - *Inputs:* Receives HTML from Bright Data node  
    - *Outputs:* Passes extracted arrays to “Format & Isolate Price Block”  
    - *Edge Cases:* If CSS selectors do not match due to page redesign, extraction will fail or return empty arrays.

  - **Format & Isolate Price Block**  
    - *Type:* Code (JavaScript)  
    - *Role:* Filters and structures pricing data into a key-value map of plan names to numeric prices.  
    - *Config:*  
      - Uses regex to identify prices in the format `$X,XXXUSD/month`  
      - Extracts numeric values from strings, converts to numbers  
      - Maps plans “Basic”, “Grow”, “Advanced”, and “Plus” to their respective prices  
      - Returns a single JSON object with plan-price pairs  
    - *Inputs:* Extracted HTML content arrays  
    - *Outputs:* Sends structured pricing data to “Read Last Saved Price”  
    - *Edge Cases:*  
      - Missing or malformed price strings cause null entries; filtered out  
      - If plans are missing from extraction, those keys will be absent or null  
      - Assumes order of prices matches plans; misalignment could cause incorrect mapping  
    - *Version:* Code node version 2

---

#### 2.2 Data Extraction & Comparison

- **Overview:**  
  This block reads the last saved prices from Google Sheets and compares them to the newly scraped prices to determine if any updates have occurred.

- **Nodes Involved:**  
  - Read Last Saved Price  
  - Has Price Changed?

- **Node Details:**  

  - **Read Last Saved Price**  
    - *Type:* Google Sheets  
    - *Role:* Retrieves the last stored pricing data from a Google Sheet as baseline for comparison.  
    - *Config:*  
      - Document ID linked to “Competitor price analyzer” Google Sheet  
      - Sheet name: `Sheet1` (`gid=0`)  
      - Uses OAuth2 credentials for Google Sheets API access  
    - *Inputs:* Receives structured pricing from previous node  
    - *Outputs:* Forwards stored price data to “Has Price Changed?”  
    - *Edge Cases:*  
      - Authentication errors if OAuth2 token expired or revoked  
      - Sheet access errors if document ID or permissions are incorrect  
      - Empty or malformed sheet data can cause comparison issues  
    - *Version:* Google Sheets node version 4.6

  - **Has Price Changed?**  
    - *Type:* If  
    - *Role:* Compares each pricing plan’s new value against the stored value using strict number equality.  
    - *Config:*  
      - Condition type: AND (all plans must be equal to trigger “no change”)  
      - Checks “Basic”, “Grow”, “Advanced”, “Plus” individually for equality  
      - Case sensitive and strict type validation enabled  
    - *Inputs:* Receives current scraped price and stored price data  
    - *Outputs:*  
      - True branch: to “No Change Detected – Stop” node (no update needed)  
      - False branch: to “Update Stored Price” node (price changed)  
    - *Edge Cases:*  
      - Strict equality means minor formatting differences or type mismatches could trigger false positives  
      - Missing plan values could cause condition evaluation errors  
    - *Version:* If node version 2.2

---

#### 2.3 Alerting & Data Persistence

- **Overview:**  
  This final block handles the two outcomes of the comparison: no action if prices are unchanged, or updating the Google Sheet and sending an alert email if prices have changed.

- **Nodes Involved:**  
  - No Change Detected – Stop  
  - Update Stored Price  
  - Send Price Change Alert Email

- **Node Details:**  

  - **No Change Detected – Stop**  
    - *Type:* No Operation (NoOp)  
    - *Role:* Terminates the workflow branch when no price changes are detected, minimizing unnecessary processing.  
    - *Config:* Defaults; no parameters needed  
    - *Inputs:* From “Has Price Changed?” true branch  
    - *Outputs:* None  
    - *Edge Cases:* None; safe termination node.

  - **Update Stored Price**  
    - *Type:* Google Sheets  
    - *Role:* Updates the existing Google Sheet row with the new pricing data to maintain an accurate record.  
    - *Config:*  
      - Document ID and Sheet name same as “Read Last Saved Price”  
      - Operation: Update  
      - Matching column: `row_number` to identify the correct sheet row  
      - Columns updated: “Basic”, “Grow”, “Advanced”, “Plus” with new numeric prices  
      - OAuth2 credentials for access  
    - *Inputs:* Receives new price data from “Has Price Changed?” false branch  
    - *Outputs:* Forwards data to “Send Price Change Alert Email”  
    - *Edge Cases:*  
      - Sheet write permission errors or connectivity issues  
      - Incorrect row_number could update wrong row  
    - *Version:* Google Sheets node version 4.6

  - **Send Price Change Alert Email**  
    - *Type:* Gmail  
    - *Role:* Sends a professional notification email alerting the team about detected price changes.  
    - *Config:*  
      - Recipient: `shahkar.genai@gmail.com`  
      - Subject: “Competitor Pricing Page Has Changed”  
      - HTML message body includes:  
        - Greeting  
        - Competitor name (“Wix”)  
        - Link to competitor pricing page (`https://www.wix.com/upgrade/website`)  
        - Timestamp of check (`{{ $now }}`)  
        - Instructions to review Google Sheets for detailed pricing  
        - Sign-off with sender’s name  
      - Gmail OAuth2 credentials configured  
      - Option to disable attribution appended to email  
    - *Inputs:* From “Update Stored Price”  
    - *Outputs:* None (workflow endpoint)  
    - *Edge Cases:*  
      - Gmail API authentication failure or quota exceeded  
      - Email delivery failure or spam filtering  
    - *Version:* Gmail node version 2.1

---

### 3. Summary Table

| Node Name                     | Node Type            | Functional Role                          | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                     |
|-------------------------------|----------------------|----------------------------------------|------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------|
| Check Pricing Every Day        | Schedule Trigger     | Initiate daily scraping                 | None                         | Fetch Page via Bright Data    | See Sticky Note at -1460,640: Section 1 - Scheduled scraping using Bright Data                                  |
| Fetch Page via Bright Data     | HTTP Request        | Scrape competitor pricing page via API | Check Pricing Every Day       | Extract HTML Content          | See Sticky Note at -1460,640: Section 1 - Scheduled scraping using Bright Data                                  |
| Extract HTML Content           | HTML Extract        | Extract plan names and pricing spans    | Fetch Page via Bright Data    | Format & Isolate Price Block  | See Sticky Note at -1460,640: Section 1 - Scheduled scraping using Bright Data                                  |
| Format & Isolate Price Block   | Code (JavaScript)   | Parse and map plan prices               | Extract HTML Content          | Read Last Saved Price         | See Sticky Note at -1460,640: Section 1 - Scheduled scraping using Bright Data                                  |
| Read Last Saved Price          | Google Sheets       | Read last stored prices for comparison | Format & Isolate Price Block  | Has Price Changed?            | See Sticky Note at -580,520: Section 2 - Price comparison logic                                                 |
| Has Price Changed?             | If                  | Compare new prices to stored prices    | Read Last Saved Price         | No Change Detected – Stop / Update Stored Price | See Sticky Note at -580,520: Section 2 - Price comparison logic                                                 |
| No Change Detected – Stop      | NoOp                 | Exit when no price change detected     | Has Price Changed? (true)     | None                         | See Sticky Note at 0,0: Section 3 - Alert and save new price                                                     |
| Update Stored Price            | Google Sheets       | Update sheet with new prices            | Has Price Changed? (false)    | Send Price Change Alert Email| See Sticky Note at 0,0: Section 3 - Alert and save new price                                                     |
| Send Price Change Alert Email | Gmail                | Send email notification about changes  | Update Stored Price           | None                         | See Sticky Note at 0,0: Section 3 - Alert and save new price                                                     |
| Sticky Note                   | Sticky Note          | Documentation and section summaries     | None                         | None                         | Various notes at positions -1460,640; -580,520; 0,0; -3160,640; -3160,980; 520,0                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to run once daily at 09:00 (9 AM).  
   - Name it “Check Pricing Every Day”.

2. **Create HTTP Request node:**  
   - Type: HTTP Request  
   - Name: “Fetch Page via Bright Data”  
   - Set method to POST, URL to `https://api.brightdata.com/request`.  
   - Body type: JSON  
   - JSON Body:  
     ```
     {
       "zone": "n8n_unblocker",
       "url": "https://www.shopify.com/uk/pricing",
       "country": "us",
       "format": "raw",
       "headers": {
         "User-Agent": "Mozilla/5.0",
         "Accept": "text/html"
       }
     }
     ```  
   - Add HTTP header “Authorization” with value “Bearer API_KEY” (replace with your actual Bright Data API key).  
   - Connect “Check Pricing Every Day” output to this node.

3. **Create HTML Extract node:**  
   - Type: HTML Extract  
   - Name: “Extract HTML Content”  
   - Configure extraction values:  
     - Key: `Plan name`, CSS selector: `h3`, return array: true  
     - Key: `Pricing`, CSS selector: `span`, return array: true  
   - Connect from “Fetch Page via Bright Data”.

4. **Create Code node:**  
   - Type: Code (JavaScript)  
   - Name: “Format & Isolate Price Block”  
   - Paste the provided JavaScript code to parse plan names and prices, filter and convert to numeric values, and build a JSON object with keys “Basic”, “Grow”, “Advanced”, “Plus”.  
   - Connect from “Extract HTML Content”.

5. **Create Google Sheets node (Read):**  
   - Type: Google Sheets  
   - Name: “Read Last Saved Price”  
   - Operation: Read / Get rows (default)  
   - Select your Google Sheets OAuth2 credentials.  
   - Document ID: Use your Google Sheet that stores competitor prices (e.g., “Competitor price analyzer”).  
   - Sheet Name: `Sheet1` (gid=0)  
   - Connect from “Format & Isolate Price Block”.

6. **Create If node:**  
   - Type: If  
   - Name: “Has Price Changed?”  
   - Condition: AND of multiple conditions, each comparing new price to stored price for plans “Basic”, “Grow”, “Advanced”, “Plus” using strict number equality.  
   - Expressions for conditions:  
     - Left: `={{ $('Format & Isolate Price Block').item.json.Basic }}`  
       Right: `={{ $json.Basic }}`  
     - Repeat similarly for “Grow”, “Advanced”, “Plus”.  
   - Connect from “Read Last Saved Price”.

7. **Create No Operation node:**  
   - Type: NoOp  
   - Name: “No Change Detected – Stop”  
   - Connect from “Has Price Changed?” true output.

8. **Create Google Sheets node (Update):**  
   - Type: Google Sheets  
   - Name: “Update Stored Price”  
   - Operation: Update  
   - Document ID and Sheet name same as read node.  
   - Map columns “Basic”, “Grow”, “Advanced”, “Plus” to their respective new values.  
   - Use “row_number” column to match the row to update.  
   - Connect from “Has Price Changed?” false output.

9. **Create Gmail node:**  
   - Type: Gmail  
   - Name: “Send Price Change Alert Email”  
   - Connect from “Update Stored Price”.  
   - Set recipient email (e.g., `shahkar.genai@gmail.com`).  
   - Subject: “Competitor Pricing Page Has Changed”.  
   - Message: Use HTML content with placeholders for competitor info, page URL, and timestamp `{{ $now }}`.  
   - Use Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow automatically bypasses scraping protections using Bright Data Web Unlocker API.                                          | Bright Data API documentation: https://brightdata.com/                                          |
| The workflow sends real-time alerts via Gmail only when pricing changes are detected, minimizing noise.                           | Gmail API & OAuth2 setup required for authentication.                                           |
| Google Sheets acts as persistent storage to keep a historical record of competitor prices for comparison.                         | Ensure Google Sheets OAuth2 credentials have read/write access to the designated spreadsheet.   |
| Workflow author and support contact: Yaron@nofluff.online, with additional tutorials on YouTube and LinkedIn.                   | https://www.youtube.com/@YaronBeen/videos, https://www.linkedin.com/in/yaronbeen/               |
| Commission note and affiliate link for Bright Data included for user support.                                                    | https://get.brightdata.com/1tndi4600b25                                                        |
| Visual workflow summary diagram available in Sticky Notes within the workflow for quick grasp of node connections and logic.    | Mermaid diagram embedded in sticky notes.                                                      |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created in n8n, adhering strictly to content policies, containing no illegal or offensive material. All data handled are lawful and publicly accessible.