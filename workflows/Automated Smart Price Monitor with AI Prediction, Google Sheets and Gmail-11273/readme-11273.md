Automated Smart Price Monitor with AI Prediction, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/automated-smart-price-monitor-with-ai-prediction--google-sheets-and-gmail-11273


# Automated Smart Price Monitor with AI Prediction, Google Sheets and Gmail

### 1. Workflow Overview

This workflow, titled **"Automated Smart Price Monitor with AI Prediction, Google Sheets and Gmail"**, automates the monitoring of product prices on Amazon and provides AI-driven purchase recommendations via email. It is designed for users who track multiple products, want to keep historical price data, and receive actionable insights to optimize their buying decisions.

**Use Cases:**
- E-commerce shoppers tracking price fluctuations and deals.
- Price analysts maintaining historical price records.
- Automated alert systems for purchase opportunities based on AI predictions.

**Logical Blocks:**

- **1.1 Input Reception and Initialization:** Trigger to start the process and retrieve the list of products from Google Sheets.
- **1.2 Price Extraction:** Fetch product web pages and extract current price data.
- **1.3 Price History Management:** Retrieve historical pricing data and update the history sheet with new records.
- **1.4 AI Analysis and Prediction:** Prepare data and use OpenAI GPT to analyze price trends and generate buy/wait recommendations.
- **1.5 Notification and Data Update:** Send email alerts with AI insights and update the product tracking sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
  Starts the workflow manually and retrieves a list of products to monitor from the Google Sheets "Products" sheet.

- **Nodes Involved:**  
  - Manual Trigger  
  - Get Products  
  - Loop Products

- **Node Details:**

  - **Manual Trigger**  
    - *Type:* Trigger node to start workflow manually.  
    - *Configuration:* Default; no parameters required.  
    - *Inputs/Outputs:* No input; triggers "Get Products".  
    - *Failure modes:* None, unless manual start is forgotten.  

  - **Get Products**  
    - *Type:* Google Sheets node (Read operation).  
    - *Configuration:* Reads all rows from the "Products" sheet using OAuth2 credentials.  
    - *Key Variables:* Reads columns: URL_Product, Product_Name, Target_Price, User_Email.  
    - *Inputs:* Manual Trigger.  
    - *Outputs:* Sends product rows to "Loop Products".  
    - *Failure modes:* Authentication errors, invalid Google Sheet ID, empty or malformed sheet data.  
    - *Notes:* Requires replacing placeholder `YOUR_GOOGLE_SHEETS_DOCUMENT_ID` with actual ID.

  - **Loop Products**  
    - *Type:* SplitInBatches node.  
    - *Configuration:* Iterates through each product row one by one.  
    - *Inputs:* Data from "Get Products".  
    - *Outputs:* Feeds each product item into "Fetch Product Page".  
    - *Failure modes:* None inherent; depends on input data integrity.  

---

#### 1.2 Price Extraction

- **Overview:**  
  For each product, fetches the product page HTML and extracts the current price using a CSS selector.

- **Nodes Involved:**  
  - Fetch Product Page  
  - Extract Price

- **Node Details:**

  - **Fetch Product Page**  
    - *Type:* HTTP Request node.  
    - *Configuration:* Sends GET request to product URL from current item. Uses a browser-like User-Agent header. Timeout set to 30 seconds.  
    - *Inputs:* From "Loop Products" (each product URL).  
    - *Outputs:* HTML content to "Extract Price".  
    - *Failure modes:* Network errors, timeout, 404 or blocking by Amazon (anti-bot), invalid URL formats.  

  - **Extract Price**  
    - *Type:* HTML Extract node.  
    - *Configuration:* Extracts price using CSS selector `.a-price-whole` (Amazon price element).  
    - *Inputs:* HTML response from "Fetch Product Page".  
    - *Outputs:* Extracted price string (e.g., "123").  
    - *Failure modes:* Selector changes, price element missing, non-Amazon URLs require CSS selector adjustment.  
    - *Notes:* Sticky note warns to update CSS selector for non-Amazon sites.

---

#### 1.3 Price History Management

- **Overview:**  
  Retrieves past price records from the "Price History" Google Sheet, checks for duplicates for the current date, and appends new price records if needed.

- **Nodes Involved:**  
  - Get Price History  
  - Check Duplicate  
  - Add to History

- **Node Details:**

  - **Get Price History**  
    - *Type:* Google Sheets node (Read with filter).  
    - *Configuration:* Queries "Price History" sheet filtering by current product URL.  
    - *Inputs:* From "Extract Price".  
    - *Outputs:* Price history rows to "Check Duplicate".  
    - *Failure modes:* Auth errors, invalid filters, missing sheet or columns.  

  - **Check Duplicate**  
    - *Type:* Code node (JavaScript).  
    - *Configuration:* Checks if today's price is already recorded for this product in the price history.  
    - *Logic:* Parses and compares dates; outputs whether to update or append.  
    - *Inputs:* Data from "Get Products" and "Get Price History".  
    - *Outputs:* Flags and data for next steps.  
    - *Failure modes:* Date parsing errors, empty history data.  

  - **Add to History**  
    - *Type:* Google Sheets node (Append operation).  
    - *Configuration:* Appends a new row with URL_Product, Price, Date, Timestamp.  
    - *Inputs:* From "Check Duplicate". Only appends if no duplicate record for today.  
    - *Outputs:* Data to "Prepare AI Data".  
    - *Failure modes:* Auth errors, sheet write errors, concurrency issues.

---

#### 1.4 AI Analysis and Prediction

- **Overview:**  
  Prepares price data and sends it to an AI model (OpenAI GPT-4o-mini) to analyze trends and generate a buy/wait recommendation with confidence and urgency scores. Generates a visual price history chart URL.

- **Nodes Involved:**  
  - Prepare AI Data  
  - AI Agent  
  - OpenAI Chat Model  
  - Parse AI Response  
  - Generate Chart

- **Node Details:**

  - **Prepare AI Data**  
    - *Type:* Code node.  
    - *Configuration:* Extracts current price, calculates 60-day average, min, max prices from history, formats last 30 days data for AI input, includes target price.  
    - *Inputs:* From "Add to History" and product info.  
    - *Outputs:* JSON prepared for AI analysis.  
    - *Failure modes:* Empty or malformed history, parsing errors.  

  - **AI Agent**  
    - *Type:* Langchain Agent node (wrapper for AI prompt).  
    - *Configuration:* Sends prompt with price statistics and history text to AI model. System message instructs the AI to respond in strict JSON with fields: recommendation, confidence, reason, urgency_score, predicted_trend.  
    - *Inputs:* From "Prepare AI Data".  
    - *Outputs:* Raw AI response to "Parse AI Response".  
    - *Failure modes:* API errors, response format problems, rate limits.  

  - **OpenAI Chat Model**  
    - *Type:* Langchain OpenAI Chat Model node.  
    - *Configuration:* Uses GPT-4o-mini model with OpenAI API credentials.  
    - *Inputs:* From "AI Agent".  
    - *Outputs:* AI text response.  
    - *Failure modes:* API key errors, quota exhaustion, network issues.  

  - **Parse AI Response**  
    - *Type:* Code node.  
    - *Configuration:* Parses AI output JSON from text. Extracts recommendation and metrics, calculates 'should_buy' boolean (BUY recommendation and current price <= target price), prepares chart data arrays.  
    - *Inputs:* AI response from "AI Agent".  
    - *Outputs:* Parsed structured data for email and sheet update.  
    - *Failure modes:* JSON parse exceptions, missing keys, fallback defaults to WAIT recommendation.  

  - **Generate Chart**  
    - *Type:* Code node.  
    - *Configuration:* Builds a QuickChart.io URL for a line chart showing last 30 days price history and target price line. Encodes JSON chart configuration in URL.  
    - *Inputs:* From "Parse AI Response".  
    - *Outputs:* Adds chart_url to output JSON for email embedding.  
    - *Failure modes:* Encoding errors, malformed chart data.

---

#### 1.5 Notification and Data Update

- **Overview:**  
  Based on AI recommendation, conditionally sends an email alert with detailed analysis and chart. Updates the "Products" Google Sheet with the latest price check and AI results.

- **Nodes Involved:**  
  - Should Buy? (IF node)  
  - Send Email Alert  
  - Update Products Sheet

- **Node Details:**

  - **Should Buy?**  
    - *Type:* IF node.  
    - *Configuration:* Checks if `should_buy` boolean is true.  
    - *Inputs:* From "Generate Chart".  
    - *Outputs:* True branch to "Send Email Alert", False branch to "Update Products Sheet".  
    - *Failure modes:* Logic errors if `should_buy` is undefined.  

  - **Send Email Alert**  
    - *Type:* Gmail node.  
    - *Configuration:* Sends HTML email to user's email address with AI recommendation, price stats, reason, urgency score, and embedded chart image. Uses Gmail OAuth2 credentials.  
    - *Inputs:* True branch from "Should Buy?".  
    - *Outputs:* Proceeds to "Update Products Sheet".  
    - *Failure modes:* Email sending errors, OAuth token expiry, invalid recipient email.  

  - **Update Products Sheet**  
    - *Type:* Google Sheets node (Update operation).  
    - *Configuration:* Updates the product row in the "Products" sheet with fields: Last_Price, Last_Check timestamp, AI_Recommendation, AI_Confidence, Urgency_Score, Predicted_Trend, Should_Buy. Matches row by URL_Product.  
    - *Inputs:* From "Send Email Alert" or False branch of "Should Buy?".  
    - *Outputs:* Loops back to "Loop Products" to process next product.  
    - *Failure modes:* Auth errors, concurrent updates, missing rows.

---

### 3. Summary Table

| Node Name           | Node Type                        | Functional Role                             | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                          |
|---------------------|---------------------------------|--------------------------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Manual Trigger      | n8n-nodes-base.manualTrigger     | Starts the workflow manually                | -                     | Get Products             |                                                                                                    |
| Get Products        | n8n-nodes-base.googleSheets      | Reads product list from Google Sheets       | Manual Trigger        | Loop Products            | ⚠️ Replace placeholder IDs in Google Sheets nodes                                                  |
| Loop Products       | n8n-nodes-base.splitInBatches    | Iterates over each product                   | Get Products          | Fetch Product Page        |                                                                                                    |
| Fetch Product Page  | n8n-nodes-base.httpRequest       | Downloads product page HTML                   | Loop Products         | Extract Price            |                                                                                                    |
| Extract Price       | n8n-nodes-base.html              | Extracts price string from HTML               | Fetch Product Page    | Get Price History        | ⚠️ Update CSS selector for non-Amazon sites                                                        |
| Get Price History   | n8n-nodes-base.googleSheets      | Retrieves price history for product           | Extract Price         | Check Duplicate          | ⚠️ Replace placeholder IDs in Google Sheets nodes                                                  |
| Check Duplicate     | n8n-nodes-base.code              | Checks if today's price is already recorded  | Get Price History     | Add to History           |                                                                                                    |
| Add to History      | n8n-nodes-base.googleSheets      | Appends new price record to history sheet    | Check Duplicate       | Prepare AI Data          | ⚠️ Replace placeholder IDs in Google Sheets nodes                                                  |
| Prepare AI Data     | n8n-nodes-base.code              | Calculates stats and formats data for AI     | Add to History        | AI Agent                 |                                                                                                    |
| AI Agent            | @n8n/n8n-nodes-langchain.agent  | Sends prompt to AI model for price analysis  | Prepare AI Data       | Parse AI Response        |                                                                                                    |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi | Executes OpenAI GPT-4o-mini model         | AI Agent              | AI Agent                 | Requires OpenAI API key credential                                                                 |
| Parse AI Response   | n8n-nodes-base.code              | Parses AI JSON response and prepares data    | AI Agent              | Generate Chart           |                                                                                                    |
| Generate Chart      | n8n-nodes-base.code              | Builds QuickChart URL for price history      | Parse AI Response     | Should Buy?              |                                                                                                    |
| Should Buy?         | n8n-nodes-base.if                | Conditional branch based on AI buy decision | Generate Chart        | Send Email Alert, Update Products Sheet |                                                                                                    |
| Send Email Alert    | n8n-nodes-base.gmail             | Sends email alert with AI recommendation     | Should Buy? (true)    | Update Products Sheet    | Requires Gmail OAuth2 credential                                                                   |
| Update Products Sheet | n8n-nodes-base.googleSheets    | Updates product row with latest check data   | Send Email Alert, Should Buy? (false) | Loop Products          | ⚠️ Replace placeholder IDs in Google Sheets nodes                                                  |
| Sticky Note         | n8n-nodes-base.stickyNote        | Documentation and setup instructions          | -                     | -                        | Multiple sticky notes provide detailed guidance and caution notes on setup and customization       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** to start the workflow manually.

2. **Add a Google Sheets node ("Get Products")**:
   - Operation: Read rows
   - Sheet Name: "Products"
   - Document ID: Replace with your Google Sheets document ID.
   - OAuth2 Credentials: Connect with Google Sheets OAuth2.
   - Ensure sheet contains columns: URL_Product, Product_Name, Target_Price, User_Email.

3. **Add a SplitInBatches node ("Loop Products")**:
   - Connect from "Get Products".
   - Configure to iterate through each product item.

4. **Add an HTTP Request node ("Fetch Product Page")**:
   - Connect from "Loop Products".
   - URL: Expression `{{$json.URL_Product}}`
   - Request method: GET
   - Headers: Add "User-Agent" header with value `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36`
   - Timeout: 30000 ms

5. **Add an HTML Extract node ("Extract Price")**:
   - Connect from "Fetch Product Page".
   - Operation: Extract HTML content
   - Extraction by CSS selector: `.a-price-whole`
   - Output key: "price"
   - Note: Update CSS selector if monitoring non-Amazon sites.

6. **Add a Google Sheets node ("Get Price History")**:
   - Connect from "Extract Price".
   - Operation: Read rows with filter
   - Sheet Name: "Price History"
   - Filter: Filter rows where URL_Product equals `{{$json.URL_Product}}`
   - Document ID: Same Google Sheets document ID
   - OAuth2 Credentials: Use Google Sheets OAuth2

7. **Add a Code node ("Check Duplicate")**:
   - Connect from "Get Price History".
   - JavaScript code to check if a price record exists for today's date for this product.
   - Outputs flags for whether to update or append.

8. **Add a Google Sheets node ("Add to History")**:
   - Connect from "Check Duplicate".
   - Operation: Append row
   - Sheet Name: "Price History"
   - Columns: URL_Product, Price, Date (current date), Timestamp (current timestamp)
   - Document ID: Same as above
   - OAuth2 Credentials: Google Sheets OAuth2

9. **Add a Code node ("Prepare AI Data")**:
   - Connect from "Add to History".
   - JavaScript code to compute average, min, max prices over last 60 days and format last 30 days data.
   - Include current price and target price.

10. **Add a Langchain Agent node ("AI Agent")**:
    - Connect from "Prepare AI Data".
    - System message: Instruct AI to analyze price history and return JSON with recommendation, confidence, reason, urgency_score, predicted_trend.
    - Prompt: Pass prepared data with price stats and history.
  
11. **Add an OpenAI Chat Model node ("OpenAI Chat Model")**:
    - Connect as AI language model node for "AI Agent".
    - Model: GPT-4o-mini
    - Credentials: OpenAI API key

12. **Add a Code node ("Parse AI Response")**:
    - Connect from "AI Agent".
    - Parse the JSON response from AI.
    - Compute boolean flag `should_buy` (recommendation is BUY and current price <= target price).
    - Prepare arrays for chart labels and prices.

13. **Add a Code node ("Generate Chart")**:
    - Connect from "Parse AI Response".
    - Generate a QuickChart.io URL for a line chart of the last 30 days prices and target price line.

14. **Add an IF node ("Should Buy?")**:
    - Connect from "Generate Chart".
    - Condition: Check if `should_buy` is true.

15. **Add a Gmail node ("Send Email Alert")**:
    - Connect from "Should Buy?" true output.
    - Recipient email: `{{$json.user_email}}`
    - Subject: AI recommendation and product name.
    - Message: HTML formatted with product info, AI analysis, and embedded chart.
    - Credentials: Connect Gmail OAuth2.

16. **Add a Google Sheets node ("Update Products Sheet")**:
    - Connect from "Send Email Alert" and "Should Buy?" false output.
    - Operation: Update row matching URL_Product.
    - Update columns: Last_Check (timestamp), Last_Price, AI_Recommendation, AI_Confidence, Urgency_Score, Predicted_Trend, Should_Buy, etc.
    - Sheet Name: "Products"
    - Document ID: Same Google Sheets ID
    - Credentials: Google Sheets OAuth2

17. **Connect the "Update Products Sheet" node output back to the "Loop Products" node** to continue processing remaining products.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow requires Google Sheets OAuth2, Gmail OAuth2, and OpenAI API credentials properly configured.                  | Setup prerequisites                                                                                             |
| Create two Google Sheets tabs: "Products" and "Price History" with required columns as per node configurations.       | Google Sheets structure                                                                                         |
| For non-Amazon e-commerce sites, update the price CSS selector in "Extract Price" node using browser DevTools.        | Sticky Note7                                                                                                    |
| Replace all instances of `YOUR_GOOGLE_SHEETS_DOCUMENT_ID` in Google Sheets nodes with your actual Google Sheets ID.   | Sticky Note6                                                                                                    |
| AI model used is GPT-4o-mini; can be swapped with GPT-4o for more advanced analysis (adjust in OpenAI Chat Model).   | Sticky Note                                                                                                     |
| QuickChart.io is used to generate price history charts via URL; no external chart hosting required.                   | Visualization tool                                                                                              |
| Email alerts include detailed AI analysis and price charts for informed decision making.                              | User notification                                                                                               |
| Reference documentation is embedded in sticky notes within the workflow for ease of maintenance and customization.    | Documentation inside workflow                                                                                   |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.