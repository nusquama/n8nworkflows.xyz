🏛️ Daily US Congress Members Stock Trades Report via Firecrawl + OpenAI + Gmail

https://n8nworkflows.xyz/workflows/----daily-us-congress-members-stock-trades-report-via-firecrawl---openai---gmail-4509


# 🏛️ Daily US Congress Members Stock Trades Report via Firecrawl + OpenAI + Gmail

### 1. Workflow Overview

This workflow automates the daily extraction, formatting, and email delivery of notable US Congress members’ stock trades sourced from Quiver Quantitative via the Firecrawl API. It targets trades exceeding $50,000 USD in the last month and transforms raw extracted data into a clear, human-readable summary before sending it via Gmail.

**Use Cases:**  
- Investors, researchers, and newsrooms seeking automated daily insights into congressional stock trading activities  
- Automating data extraction and report generation from web data with AI-powered formatting  
- Hands-free monitoring of congressional financial activities for decision-making or journalistic purposes

**Logical Blocks:**  
- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specified hour (6 PM)  
- **1.2 Firecrawl Data Extraction:** Sends a POST request to Firecrawl to extract structured congressional trading data from Quiver Quant page  
- **1.3 Wait for Processing:** Pauses the workflow to allow Firecrawl to complete data extraction  
- **1.4 Retrieve Extracted Results:** Sends a GET request to Firecrawl to fetch the processed data  
- **1.5 Data Conversion & Condition Check:** Converts retrieved data into array format and checks if results are ready; implements a wait-and-retry loop if data is not yet available  
- **1.6 AI Formatting:** Uses OpenAI GPT-4o to format the raw trade data into a readable report format  
- **1.7 Send Email:** Sends the formatted summary report via Gmail to a specified recipient

---

### 2. Block-by-Block Analysis

---

#### 1.1 Scheduled Trigger

**Overview:**  
Begins the workflow automatically every day at 6 PM to start the data extraction process.

**Nodes Involved:**  
- Schedule Trigger

**Node Details:**  
- **Schedule Trigger**  
  - Type: scheduleTrigger  
  - Configuration: Executes daily, triggering at hour 18 (6 PM)  
  - Input: None (entry point)  
  - Output: To "Extract" node  
  - Edge Cases: Misconfigured schedule could cause trigger failure or mistimed execution

---

#### 1.2 Firecrawl Data Extraction

**Overview:**  
Sends a POST request to Firecrawl API, instructing it to scrape the Quiver Quant congressional trading page for trades over $50,000 USD in the last month, extracting key fields.

**Nodes Involved:**  
- Extract (HTTP Request POST)  
- Sticky Note2 (comment)

**Node Details:**  
- **Extract**  
  - Type: httpRequest (POST)  
  - Configuration:  
    - URL: https://api.firecrawl.dev/v1/extract  
    - Body: JSON with target URLs (Quiver Quant Congress Trading page) and a detailed prompt specifying data fields to extract (member name, party, stock/asset, amount, date)  
    - Authentication: HTTP header with Firecrawl API key  
  - Input: From Schedule Trigger  
  - Output: To "Wait 30 Secs" node  
  - Edge Cases: API key invalid or quota exceeded; network errors; malformed prompt leading to incomplete extraction  

---

#### 1.3 Wait for Processing

**Overview:**  
Inserts a delay to allow Firecrawl time to complete the extraction process before fetching results.

**Nodes Involved:**  
- Wait 30 Secs  
- Sticky Note7 (comment)

**Node Details:**  
- **Wait 30 Secs**  
  - Type: wait  
  - Configuration: Pauses workflow for 30 seconds  
  - Input: From Extract node  
  - Output: To Get Results node  
  - Edge Cases: Insufficient wait time could cause premature data retrieval; too long delays reduce efficiency  

---

#### 1.4 Retrieve Extracted Results

**Overview:**  
Sends a GET request to Firecrawl API to fetch the structured extraction result using the extraction job ID.

**Nodes Involved:**  
- Get Results (HTTP Request GET)  
- Sticky Note1 (comment)

**Node Details:**  
- **Get Results**  
  - Type: httpRequest (GET)  
  - Configuration:  
    - URL: https://api.firecrawl.dev/v1/extract/{{ extractionJobId }}  
    - Extraction Job ID is dynamically taken from the “Extract” node response (`id` field)  
    - Authentication: HTTP header with Firecrawl API key  
  - Input: From Wait 30 Secs  
  - Output: To Code node  
  - Edge Cases: Invalid job ID; Firecrawl API errors; job not completed yet (empty or missing data)  

---

#### 1.5 Data Conversion & Condition Check

**Overview:**  
Processes the retrieved data by ensuring it is an array. Checks if the extraction results are empty (i.e., not ready), and loops with a wait if so.

**Nodes Involved:**  
- Code (JavaScript)  
- If (Conditional)  
- Wait 15 secs  
- Sticky Note (Converting to Array)  
- Sticky Note6 (Waiting loop comment)  
- Sticky Note4 (Conditional check comment)

**Node Details:**  
- **Code**  
  - Type: code  
  - Configuration: Runs once per item, converts the raw `data` field to array if necessary, outputs `trades` array  
  - Input: From Get Results  
  - Output: To If node  
  - Edge Cases: Unexpected data structure causing conversion errors  

- **If**  
  - Type: if condition  
  - Configuration: Checks if `trades` array is empty  
  - Input: From Code  
  - Output:  
    - If empty: to Wait 15 secs (to retry after delay)  
    - If not empty: to Edit Fields  
  - Edge Cases: Incorrect condition logic could cause infinite looping or premature exit  

- **Wait 15 secs**  
  - Type: wait  
  - Configuration: Pauses workflow 15 seconds before retrying data retrieval  
  - Input: From If (empty condition)  
  - Output: To Get Results node (looping back)  
  - Edge Cases: Could cause long delays if data is slow to appear or never arrives  

---

#### 1.6 AI Formatting

**Overview:**  
Formats the raw array of congressional trade records into a human-readable summary using OpenAI GPT-4o chat model with a system prompt specifying desired output structure.

**Nodes Involved:**  
- Edit Fields (Set)  
- OpenAI (Langchain OpenAI node)  
- Sticky Note5 (Formatting + Email comment)

**Node Details:**  
- **Edit Fields**  
  - Type: set  
  - Configuration: Assigns `data` field from the `trades` array for input to OpenAI  
  - Input: From If (non-empty branch)  
  - Output: To OpenAI node  

- **OpenAI**  
  - Type: OpenAI Langchain node  
  - Configuration:  
    - Model: chatgpt-4o-latest  
    - Messages:  
      - User message contains raw data blob (`data`)  
      - System message instructs the assistant to format the data into readable text showing transaction date, stock/asset, amount, purchaser name, and party  
  - Input: From Edit Fields  
  - Output: To Gmail node  
  - Edge Cases: API key limits, timeouts, or malformed prompts can cause formatting failure  

---

#### 1.7 Send Email

**Overview:**  
Sends the formatted congressional trading summary as a plain text email via Gmail.

**Nodes Involved:**  
- Gmail  
- Sticky Note5 (also covers email send)

**Node Details:**  
- **Gmail**  
  - Type: gmail  
  - Configuration:  
    - Recipient email: your_email@example.com (replace with real recipient)  
    - Subject: "Congress Trade Updates - QQ"  
    - Message body: from OpenAI formatted content  
    - Email type: plain text  
    - Authentication: Gmail OAuth2 credentials  
  - Input: From OpenAI node  
  - Output: None (end of workflow)  
  - Edge Cases: Invalid Gmail credentials, quota exceeded, network errors  

---

### 3. Summary Table

| Node Name       | Node Type            | Functional Role                           | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                    |
|-----------------|----------------------|-----------------------------------------|------------------------|------------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger| scheduleTrigger      | Starts workflow daily at 6 PM            | None                   | Extract                | Scheduled Trigger                                                                                              |
| Extract         | httpRequest (POST)   | Sends POST to Firecrawl to extract data | Schedule Trigger       | Wait 30 Secs           | Post Request to Firecrawl                                                                                      |
| Wait 30 Secs    | wait                 | Waits 30 seconds for extraction          | Extract                | Get Results            | Wait for result                                                                                                |
| Get Results     | httpRequest (GET)    | Retrieves extraction result               | Wait 30 Secs, Wait 15 secs | Code                  | Get Request to Firecrawl                                                                                       |
| Code            | code                 | Converts raw data to array                | Get Results            | If                     | Converting to Array                                                                                            |
| If              | if                   | Checks if trades array is empty           | Code                   | Wait 15 secs (if empty), Edit Fields (if not empty) | Conditional check if crawl has completed                                                                        |
| Wait 15 secs    | wait                 | Waits 15 seconds before retrying          | If (empty condition)    | Get Results            | Result not ready - loop to wait for result                                                                    |
| Edit Fields     | set                  | Prepares data field for OpenAI            | If (not empty)          | OpenAI                 | Formatting + Email Send                                                                                        |
| OpenAI          | OpenAI Langchain     | Formats trades into readable summary      | Edit Fields             | Gmail                  | Formatting + Email Send                                                                                        |
| Gmail           | gmail                | Sends formatted summary email             | OpenAI                  | None                   | Formatting + Email Send                                                                                        |
| Sticky Note     | stickyNote           | Comment node                             | None                   | None                   | Converting to Array                                                                                            |
| Sticky Note1    | stickyNote           | Comment node                             | None                   | None                   | Get Request to Firecrawl                                                                                       |
| Sticky Note2    | stickyNote           | Comment node                             | None                   | None                   | Post Request to Firecrawl                                                                                      |
| Sticky Note3    | stickyNote           | Comment node                             | None                   | None                   | Scheduled Trigger                                                                                              |
| Sticky Note4    | stickyNote           | Comment node                             | None                   | None                   | Conditional check if crawl has completed                                                                      |
| Sticky Note5    | stickyNote           | Comment node                             | None                   | None                   | Formatting + Email Send                                                                                        |
| Sticky Note6    | stickyNote           | Comment node                             | None                   | None                   | Result not ready - loop to wait for result                                                                    |
| Sticky Note7    | stickyNote           | Comment node                             | None                   | None                   | Wait for result                                                                                                |
| Sticky Note8    | stickyNote           | Overview and setup instructions          | None                   | None                   | 🔧 How It Works … [See full content in Section 5]                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: scheduleTrigger  
   - Set to trigger daily at hour 18 (6 PM)  

2. **Add HTTP Request node named "Extract":**  
   - Type: httpRequest  
   - Method: POST  
   - URL: https://api.firecrawl.dev/v1/extract  
   - Authentication: Set HTTP Header Auth with Firecrawl API key  
   - Body type: JSON  
   - Body content:  
     ```json
     {
       "urls": ["https://www.quiverquant.com/congresstrading/*"],
       "prompt": "Extract all notable congress member trades over $50,000 USD in the past 1 month. Include the Congress Member's name, party, the Stock/Asset purchased or sold, the Amount of the transaction, and the date of the transaction.",
       "schema": {
         "type": "object",
         "properties": {
           "notable_trades": {
             "type": "array",
             "items": {
               "type": "object",
               "properties": {
                 "congress_member_name": {"type": "string"},
                 "party": {"type": "string"},
                 "stock_or_asset": {"type": "string"},
                 "amount": {"type": "number"},
                 "transaction_date": {"type": "string"}
               },
               "required": ["congress_member_name", "stock_or_asset", "amount", "transaction_date"]
             }
           }
         },
         "required": ["notable_trades"]
       }
     }
     ```  
   - Connect Schedule Trigger → Extract  

3. **Add Wait node "Wait 30 Secs":**  
   - Type: wait  
   - Duration: 30 seconds  
   - Connect Extract → Wait 30 Secs  

4. **Add HTTP Request node "Get Results":**  
   - Type: httpRequest  
   - Method: GET  
   - URL: `https://api.firecrawl.dev/v1/extract/{{ $json["id"] }}` (use dynamic expression to get `id` from Extract response)  
   - Authentication: HTTP Header Auth with Firecrawl API key  
   - Connect Wait 30 Secs → Get Results  

5. **Add Code node "Code":**  
   - Type: code  
   - Mode: runOnceForEachItem  
   - JavaScript code:  
     ```javascript
     let trades = $json.data;
     if (!Array.isArray(trades)) {
       trades = trades ? [trades] : [];
     }
     return { trades };
     ```  
   - Connect Get Results → Code  

6. **Add If node "If":**  
   - Type: if  
   - Condition: Check if `trades` is empty array (use 'array empty' operator on `{{$json.trades}}`)  
   - Connect Code → If  

7. **Add Wait node "Wait 15 secs":**  
   - Type: wait  
   - Duration: 15 seconds  
   - Connect If (true branch - empty) → Wait 15 secs  

8. **Loop back Wait 15 secs → Get Results:**  
   - Connect Wait 15 secs → Get Results (to retry fetching results)  

9. **Add Set node "Edit Fields":**  
   - Type: set  
   - Assign field: `data` = `{{$json.trades}}`  
   - Include other fields: true  
   - Connect If (false branch - non-empty) → Edit Fields  

10. **Add OpenAI node "OpenAI":**  
    - Use Langchain OpenAI node or standard OpenAI node  
    - Model: chatgpt-4o-latest  
    - Messages:  
      - User: `{{$json.data}}` (raw trade data)  
      - System: "You are a helpful text editing assistant. Your job is to format the data into an easily readable format showing Transaction Date, Stock/Asset, Amount, Purchaser Name, and Party."  
    - Connect Edit Fields → OpenAI  

11. **Add Gmail node "Gmail":**  
    - Set recipient email (e.g., your_email@example.com)  
    - Subject: "Congress Trade Updates - QQ"  
    - Message: `{{$json.message.content}}` (formatted summary from OpenAI)  
    - Email type: plain text  
    - Configure Gmail OAuth2 credentials  
    - Connect OpenAI → Gmail  

12. **Verify all connections and credentials:**  
    - Firecrawl API key (HTTP Header Auth)  
    - OpenAI API key  
    - Gmail OAuth2 credentials  

13. **Activate and test the workflow:**  
    - Trigger manually or wait for scheduled time  
    - Confirm data extraction, formatting, and email delivery  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 🔧 How It Works: Scheduled trigger initiates daily at 6 PM → Firecrawl POST request extracts congressional trades over $50K → wait for Firecrawl processing → GET request fetches results → data converted to array and checked → loops wait if result not ready → OpenAI formats data → Gmail sends summary email.                                                                                                                                                                  | See full explanation in Sticky Note8                                                             |
| Watch Full Setup Video Tutorial: https://www.youtube.com/@Automatewithmarc                                                                                                                                                                                                                                                                                                                                                                                                  | Video tutorial by Automate with Marc                                                             |
| Why Useful: Congressional trading data often signals market insights. This workflow automates timely data extraction and formatting, saving manual effort and providing daily readable reports to investors, researchers, and newsrooms.                                                                                                                                                                                                                                       | Workflow purpose and benefits                                                                    |
| Requirements: Firecrawl API Key (with extract access), OpenAI API Key, Gmail OAuth2 credentials, n8n environment (self-hosted or cloud)                                                                                                                                                                                                                                                                                                                                      | Credential and environment prerequisites                                                        |
| Sample Output Format: "Nancy Pelosi (D) sold TSLA for $85,000 on April 28\nJohn Raynor (R) purchased AAPL worth $120,000 on May 2..."                                                                                                                                                                                                                                                                                                                                          | Example of the email summary content                                                             |

---

**Disclaimer:** The provided text and data originate exclusively from an automated n8n workflow, adhering strictly to content policies with no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.