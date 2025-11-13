Stock Market Daily Digest with Bright Data Scraping & Gemini AI Email Reports

https://n8nworkflows.xyz/workflows/stock-market-daily-digest-with-bright-data-scraping---gemini-ai-email-reports-5305


# Stock Market Daily Digest with Bright Data Scraping & Gemini AI Email Reports

---

## 1. Workflow Overview

This workflow automates the generation and delivery of a **daily stock market digest** email, focusing on top U.S. stocks. It leverages Bright Data’s scraping API to gather the latest financial news and trends for a curated list of stocks, processes and aggregates the data, then uses Google Gemini AI to generate a polished HTML email summary. Finally, it sends the summary via Gmail and archives the raw data in Google Sheets.

**Target Use Cases:**

- Financial analysts and investors seeking automated daily market summaries.
- Teams needing timely and insightful stock reports without manual data collection.
- Integration of web scraping, AI summarization, and email automation into one seamless flow.

**Logical Blocks:**

- **1.1 Schedule & Stock List Initialization:** Trigger the workflow daily and define the stocks to track.
- **1.2 Keyword Preparation & Scraping Trigger:** Prepare keywords and trigger Bright Data scraping jobs for each stock.
- **1.3 Scraping Monitoring & Data Retrieval:** Poll Bright Data API to monitor scraping progress and retrieve results.
- **1.4 Data Aggregation & Persistence:** Aggregate collected data and store it in Google Sheets for record-keeping.
- **1.5 AI Summary Generation:** Use Google Gemini AI to analyze aggregated data and create an HTML email summary.
- **1.6 Email Delivery:** Send the AI-generated summary email through Gmail to the designated recipient.

---

## 2. Block-by-Block Analysis

### 2.1 Schedule & Stock List Initialization

**Overview:**  
This block triggers the workflow daily and sets up the list of stocks to monitor. It defines sample stock data with relevant metadata.

**Nodes Involved:**  
- Schedule Trigger  
- SAMPLE DATA  
- Sticky Note (Add stock to track)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution on a daily interval (default daily, no custom cron)  
  - Config: Interval set to run daily  
  - Inputs: None (trigger node)  
  - Outputs: SAMPLE DATA  
  - Edge Cases: None significant, but scheduling errors or misconfigurations could prevent runs.

- **SAMPLE DATA**  
  - Type: Set  
  - Role: Defines an array of top 10 US stocks with ticker, name, and approximate market cap  
  - Config: JSON array hardcoded for 10 stocks (e.g., NVDA, MSFT, AAPL)  
  - Inputs: Schedule Trigger output  
  - Outputs: Split Out  
  - Edge Cases: If modified incorrectly, could break data flow; ensure JSON format intact.

- **Sticky Note (Add stock to track)**  
  - Role: Documentation only, guides users to add or modify tracked stocks here.

---

### 2.2 Keyword Preparation & Scraping Trigger

**Overview:**  
Splits the stock array into individual items, sets the keyword field for each, and triggers scraping jobs on Bright Data for each stock keyword.

**Nodes Involved:**  
- Split Out  
- set keyword  
- Financial times scraper  
- Sticky Notes (Split and set field names, Financial Times scraper)

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of stocks into individual JSON objects for per-stock processing  
  - Config: Field to split on = `json`  
  - Inputs: SAMPLE DATA  
  - Outputs: set keyword  
  - Edge Cases: Empty array input leads to no further processing.

- **set keyword**  
  - Type: Set  
  - Role: Maps ticker symbol to a keyword field required by Bright Data API  
  - Config: Sets `keyword` = ticker symbol from each JSON item  
  - Inputs: Split Out  
  - Outputs: Financial times scraper  
  - Edge Cases: Missing ticker field breaks keyword creation.

- **Financial times scraper**  
  - Type: HTTP Request  
  - Role: Sends POST request to Bright Data API to trigger scraping jobs for each keyword (stock ticker)  
  - Config:  
    - URL: Bright Data dataset trigger endpoint  
    - Method: POST  
    - Headers: Authorization Bearer token (replace `YOUR_BRIGHTDATA_API_KEY`)  
    - Body: JSON array of keywords  
    - Query Params: dataset_id, include_errors, type=discover_new, discover_by=keyword  
    - Execute once for all keywords  
  - Inputs: set keyword  
  - Outputs: Get progress  
  - Edge Cases:  
    - Authorization errors if API key invalid  
    - Network timeouts or API rate limits  
    - Invalid keyword format leads to scraping failure  
  - Sticky Note: Reminds to run once and explains scraping purpose.

- **Sticky Notes (Split and set field names)**  
  - Role: Documentation emphasizing the importance of setting the `keyword` field exactly for API compatibility.

---

### 2.3 Scraping Monitoring & Data Retrieval

**Overview:**  
Monitors the progress of scraping jobs by polling the Bright Data API until the scraping status is `ready`. Once ready, retrieves the snapshot data.

**Nodes Involved:**  
- Get progress  
- Switch  
- Wait  
- Get snapshot + data  
- Sticky Notes (Check status and Get results, Get scraping results)

**Node Details:**

- **Get progress**  
  - Type: HTTP Request  
  - Role: Queries Bright Data API for scraping progress of each snapshot_id returned by the scraper  
  - Config:  
    - URL: Constructed dynamically with snapshot_id (e.g., `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`)  
    - Method: GET  
    - Headers: Authorization Bearer token  
  - Inputs: Financial times scraper  
  - Outputs: Switch  
  - Edge Cases:  
    - Missing or invalid snapshot_id causes errors  
    - Authorization or network errors possible  

- **Switch**  
  - Type: Switch  
  - Role: Routes flow based on scraping status:  
    - `ready`: proceed to fetch data  
    - `running`: wait and poll again  
  - Config: Checks `$json.status` equals "ready" or "running"  
  - Inputs: Get progress  
  - Outputs:  
    - `ready` → Get snapshot + data  
    - `running` → Wait  
  - Edge Cases: Unknown statuses or missing fields cause no routing.

- **Wait**  
  - Type: Wait  
  - Role: Delays the workflow for 20 seconds before rechecking progress  
  - Config: Wait for 20 seconds  
  - Inputs: Switch (running)  
  - Outputs: Get progress (loop)  
  - Edge Cases: Delays increase total workflow runtime; network failures during rechecks.

- **Get snapshot + data**  
  - Type: HTTP Request  
  - Role: Retrieves the scraped data snapshot from Bright Data once ready  
  - Config:  
    - URL with snapshot_id (`https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`)  
    - Query param format=json  
    - Header Authorization Bearer token  
  - Inputs: Switch (ready)  
  - Outputs: Google Sheets, Aggregate  
  - Edge Cases: Large data sets may cause timeouts; invalid snapshot_id.

- **Sticky Notes (Check status and Get results)**  
  - Role: Documentation explaining the polling loop and data retrieval logic.

---

### 2.4 Data Aggregation & Persistence

**Overview:**  
Aggregates all scraped data into a single JSON object for AI processing and appends raw data to a Google Sheet for record-keeping.

**Nodes Involved:**  
- Aggregate  
- Google Sheets  
- Sticky Notes (Aggregate results to pass to AI for our email summary, Save results to G sheets)

**Node Details:**

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines multiple JSON items into one aggregate object for AI input  
  - Config: Aggregate all item data (default aggregation)  
  - Inputs: Get snapshot + data  
  - Outputs: create summary  
  - Edge Cases: Large datasets may affect performance.

- **Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends the raw scraped data into a configured Google Sheet for logging  
  - Config:  
    - Operation: Append  
    - Document ID: Configured to a specific Google Sheets document  
    - Sheet Name: Sheet1  
    - Mapping Mode: Auto map input data columns  
    - Credentials: Google Sheets OAuth2 credentials required  
  - Inputs: Get snapshot + data  
  - Outputs: None (terminal storage node)  
  - Edge Cases: Authentication failures, quota limits, or invalid sheet IDs.

- **Sticky Notes**  
  - Role: Explain aggregation and data saving steps.

---

### 2.5 AI Summary Generation

**Overview:**  
Uses Google Gemini AI to analyze aggregated stock data and generate a detailed, styled HTML email summary of daily stock market trends.

**Nodes Involved:**  
- create summary (Langchain Agent)  
- Google Gemini Chat Model  
- Sticky Note (Create Draft email and notify admin)

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini AI Chat Model  
  - Role: The AI language model that processes the prompt and data to generate text  
  - Config: Uses model "models/gemini-2.0-flash-lite"  
  - Credentials: Google Palm API credentials required  
  - Inputs: create summary (ai_languageModel input)  
  - Outputs: create summary (ai_tool input)  
  - Edge Cases: API errors, rate limits, or malformed prompts.

- **create summary**  
  - Type: Langchain Agent  
  - Role: Defines the AI prompt and orchestrates the summarization task  
  - Config:  
    - Prompt includes system instructions for generating a clear, styled HTML email summary of top 10 US stocks with market insights, top movers, tables, icons, and constraints (HTML only, no plaintext).  
    - Input data is JSON stock data aggregated from scraping.  
  - Inputs: Aggregate (main), Google Gemini Chat Model (ai_languageModel)  
  - Outputs: Gmail1 (ai_tool)  
  - Edge Cases: Incorrect input formatting or AI model errors.

- **Sticky Note**  
  - Role: Describes creation of the draft email and notification to admin/user.

---

### 2.6 Email Delivery

**Overview:**  
Sends the AI-generated HTML summary via Gmail to a specified recipient.

**Nodes Involved:**  
- Gmail1

**Node Details:**

- **Gmail1**  
  - Type: Gmail  
  - Role: Sends email with AI-generated subject and message body  
  - Config:  
    - Recipient: `kimothozacharia5@gmail.com` (hardcoded)  
    - Subject and Message: Overridden dynamically by AI-generated content from create summary node outputs  
    - Credentials: Gmail OAuth2 credentials required  
    - Options: Append attribution disabled  
  - Inputs: create summary (ai_tool)  
  - Outputs: None (terminal node)  
  - Edge Cases: Authentication failures, email quota limits, malformed HTML causing rendering issues.

---

## 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                  | Input Node(s)           | Output Node(s)                          | Sticky Note                                                                                                    |
|-------------------------|----------------------------------|-------------------------------------------------|-------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                  | Initiates workflow daily                         | None                    | SAMPLE DATA                           |                                                                                                               |
| SAMPLE DATA             | Set                              | Defines top 10 stocks list                       | Schedule Trigger        | Split Out                            | Add stock to track                                                                                            |
| Split Out               | Split Out                        | Splits stock list into individual items         | SAMPLE DATA             | set keyword                         | Split and set the field names                                                                                  |
| set keyword             | Set                              | Sets `keyword` field for Bright Data API        | Split Out               | Financial times scraper             | Split and set the field names                                                                                  |
| Financial times scraper | HTTP Request                     | Triggers Bright Data scraping jobs               | set keyword             | Get progress                       | Financial Times scraper. Scrape latest trends. Set to run once.                                               |
| Get progress            | HTTP Request                     | Polls Bright Data API for scraping status        | Financial times scraper | Switch                            | Check the status of the progress and Get results. If running loop until done                                  |
| Switch                  | Switch                          | Routes flow based on scraping status             | Get progress            | Get snapshot + data, Wait           | Check the status of the progress and Get results. If running loop until done                                  |
| Wait                    | Wait                            | Delays before rechecking scraping status          | Switch (running)        | Get progress                      | Check the status of the progress and Get results. If running loop until done                                  |
| Get snapshot + data     | HTTP Request                     | Retrieves scraped data snapshot                   | Switch (ready)          | Google Sheets, Aggregate            | Get scraping results                                                                                           |
| Google Sheets           | Google Sheets                   | Append raw scraped data to spreadsheet            | Get snapshot + data     | None                              | Save results to G sheets                                                                                       |
| Aggregate               | Aggregate                       | Aggregates all scraped data for AI input          | Get snapshot + data     | create summary                    | Aggregate results to pass to AI for our email summary                                                        |
| Google Gemini Chat Model| Langchain AI Model               | Processes prompt and generates AI text            | create summary          | create summary                    |                                                                                                               |
| create summary          | Langchain Agent                 | Defines AI prompt and generates HTML email body   | Aggregate, Google Gemini Chat Model | Gmail1                        | Create Draft email and notify admin                                                                           |
| Gmail1                  | Gmail                          | Sends AI-generated summary email                   | create summary          | None                              |                                                                                                               |
| Sticky Note             | Sticky Note                    | Documentation nodes                                | None                    | None                              | Various notes distributed across workflow nodes as detailed above                                            |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to run daily (default interval)  
   - No inputs  

2. **Create a Set node named "SAMPLE DATA"**  
   - Connect input from Schedule Trigger  
   - Assign a JSON array to `json` field containing the top 10 US stocks with fields: ticker, name, market_cap (use provided sample data).  

3. **Create a Split Out node**  
   - Connect input from SAMPLE DATA  
   - Configure to split on the `json` field  

4. **Create a Set node named "set keyword"**  
   - Connect input from Split Out  
   - Set a new string field `keyword` = the ticker symbol (expression: `{{$json["ticker"]}}`)  

5. **Create an HTTP Request node named "Financial times scraper"**  
   - Connect input from "set keyword"  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Headers: Add Authorization header with `Bearer YOUR_BRIGHTDATA_API_KEY` (replace with actual API key)  
   - Query parameters:  
     - dataset_id: `gd_lmrpz3vxmz972ghd7`  
     - include_errors: `true`  
     - type: `discover_new`  
     - discover_by: `keyword`  
   - Body: JSON array of all keywords, set to execute once for all items (expression: `={{ $('set keyword').all().map(item => item.json)}}`)  
   - Send as JSON body  

6. **Create an HTTP Request node named "Get progress"**  
   - Connect input from Financial times scraper  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (dynamic with snapshot_id)  
   - Headers: Authorization Bearer token as above  

7. **Create a Switch node**  
   - Connect input from Get progress  
   - Condition: `$json.status` equals "ready" → output "ready"  
   - Condition: `$json.status` equals "running" → output "running"  

8. **Create a Wait node**  
   - Connect input from Switch "running" output  
   - Set wait time to 20 seconds  

9. **Connect Wait node output back to Get progress to form a polling loop**  

10. **Create an HTTP Request node named "Get snapshot + data"**  
    - Connect input from Switch "ready" output  
    - Method: GET  
    - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
    - Query parameter: format = json  
    - Headers: Authorization Bearer token  

11. **Create a Google Sheets node**  
    - Connect input from Get snapshot + data  
    - Operation: Append  
    - Document ID: specify your Google Sheet ID  
    - Sheet Name: Sheet1 (or your target sheet)  
    - Mapping mode: Auto map input data  
    - Credentials: Google Sheets OAuth2 credentials configured  

12. **Create an Aggregate node**  
    - Connect input from Get snapshot + data  
    - Aggregate all item data (default)  

13. **Create a Langchain Google Gemini Chat Model node**  
    - Connect input from create summary node (ai_languageModel input)  
    - Model: `models/gemini-2.0-flash-lite`  
    - Credentials: Google PaLM API credentials  

14. **Create a Langchain Agent node named "create summary"**  
    - Connect main input from Aggregate node  
    - Connect ai_languageModel input from Google Gemini Chat Model  
    - Configure with the detailed prompt to generate an HTML email body summarizing stock data (copy prompt content from the node details above)  

15. **Create a Gmail node named "Gmail1"**  
    - Connect ai_tool input from create summary node  
    - Set recipient email (e.g., `kimothozacharia5@gmail.com`)  
    - Use AI-generated subject and message content (expressions override default message and subject)  
    - Credentials: Gmail OAuth2 configured  

16. **Add Sticky Notes throughout the workflow** to document purpose and usage as indicated in the original workflow.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The Bright Data API key must be replaced with a valid key for scraping to work.                                                                                                                                                                            | Bright Data API documentation                                                                     |
| Google Sheets credential requires OAuth2 with permission to append data to the specified spreadsheet.                                                                                                                                                       | Google Sheets API and OAuth2 setup                                                                |
| Google Gemini AI (PaLM) credentials are required and should have access to `models/gemini-2.0-flash-lite`.                                                                                                                                                  | Google Cloud PaLM API documentation                                                               |
| Gmail node requires OAuth2 credentials with send email permissions.                                                                                                                                                                                        | Gmail API OAuth2 setup                                                                            |
| The AI prompt is carefully designed to output only HTML content suitable for direct email use, avoiding plaintext or markdown outputs.                                                                                                                    | Prompt engineering best practices                                                                |
| The workflow includes a polling loop with a 20-second wait to handle asynchronous scraping completion, which may extend overall execution time.                                                                                                            | Polling pattern for asynchronous API completion                                                  |
| Sample stocks and their metadata can be customized in the SAMPLE DATA node to track different or additional stocks.                                                                                                                                         | Workflow customization note                                                                       |
| The workflow’s design is modular, facilitating easy substitution of scraping sources, AI models, or email services, if needed.                                                                                                                            | Best practices for modular n8n workflows                                                        |
| For more about Bright Data datasets API: https://brightdata.com/docs/datasets-api/v3                                                                                                                                                                        | Official Bright Data API docs                                                                    |
| For Google Gemini (PaLM) integration details: https://developers.generativeai.google                                                                                                                           | Google Generative AI developer docs                                                             |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. It strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.