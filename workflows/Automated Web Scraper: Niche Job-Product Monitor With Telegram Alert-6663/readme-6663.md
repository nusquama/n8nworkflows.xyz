Automated Web Scraper: Niche Job/Product Monitor With Telegram Alert

https://n8nworkflows.xyz/workflows/automated-web-scraper--niche-job-product-monitor-with-telegram-alert-6663


# Automated Web Scraper: Niche Job/Product Monitor With Telegram Alert

### 1. Workflow Overview

This workflow is an **Automated Web Scraper** designed to monitor niche job boards, product pages, or any specified webpage at regular intervals and send alerts via Telegram when new or updated items are detected.

It is structured into the following logical blocks:

- **1.1 Hourly Trigger:** Automatically initiates the workflow on a recurring schedule (default every hour).
- **1.2 Web Content Retrieval:** Downloads the raw HTML content of the targeted webpage.
- **1.3 Data Extraction:** Parses the HTML to extract specific job or product information using CSS selectors.
- **1.4 Conditional Check:** Determines if any relevant items were found to proceed further.
- **1.5 Notification Formatting:** Formats the extracted data into a message suitable for Telegram.
- **1.6 Alert Dispatch:** Sends the formatted notification message to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Hourly Trigger

- **Overview:**  
  This block uses a Cron node to trigger the workflow execution every hour automatically, ensuring timely monitoring without manual intervention.

- **Nodes Involved:**  
  - Hourly Monitor Trigger

- **Node Details:**  
  - **Name:** Hourly Monitor Trigger  
  - **Type:** Cron Trigger  
  - **Configuration:**  
    - Mode set to `everyHour` to run once each hour.  
    - No additional parameters configured, but can be customized to run at different intervals (e.g., every 4 hours or daily).  
  - **Inputs:** None (Trigger node).  
  - **Outputs:** Connects to `Fetch Webpage Content`.  
  - **Edge Cases / Failures:**  
    - Cron misconfiguration may cause no triggers or too frequent triggers.  
    - Timezone differences may affect expected trigger times if server timezone differs.  
  - **Version Requirements:** Standard Cron node; compatible with n8n v0.133+.

---

#### 2.2 Web Content Retrieval

- **Overview:**  
  Downloads the full HTML content of the target webpage to enable subsequent data extraction.

- **Nodes Involved:**  
  - Fetch Webpage Content

- **Node Details:**  
  - **Name:** Fetch Webpage Content  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - URL set to `https://www.n8n.io/blog/` by default — must be replaced with the URL of the job board, product page, or any target site.  
    - Response Format: `string` (to get raw HTML).  
    - No authentication or headers configured by default; may require adding cookies, headers, or tokens if the site needs login or API access.  
  - **Inputs:** Triggered by `Hourly Monitor Trigger`.  
  - **Outputs:** Sends HTML content to `Extract Job/Product Info`.  
  - **Edge Cases / Failures:**  
    - HTTP errors (404, 500, timeout).  
    - Sites with JavaScript-rendered content will not be fully captured; may need headless browser approach or code node with Puppeteer/Playwright.  
  - **Version Requirements:** HTTP Request node v3 or higher recommended for stable response format options.

---

#### 2.3 Data Extraction

- **Overview:**  
  Parses the downloaded HTML and extracts key data points such as job titles and links using CSS selectors.

- **Nodes Involved:**  
  - Extract Job/Product Info

- **Node Details:**  
  - **Name:** Extract Job/Product Info  
  - **Type:** HTML Extract  
  - **Configuration:**  
    - HTML input set dynamically as `{{ $node["Fetch Webpage Content"].json.data }}` to consume prior node’s output.  
    - Extract Operations configured with two selectors:  
      1. Selector: `h3.BlogItem_title__d78Xb` → extracts job/product titles (`textContent`).  
      2. Selector: `a.BlogItem_blogItem__a_H6E` → extracts job/product links (`href`).  
    - Property names assigned as `JobTitle` and `JobLink`.  
    - Users must customize selectors and properties to match their target webpage’s structure.  
  - **Inputs:** Receives HTML from `Fetch Webpage Content`.  
  - **Outputs:** Passes extracted data to `If Items Found`.  
  - **Edge Cases / Failures:**  
    - Incorrect or outdated CSS selectors will result in no data extracted.  
    - If the page structure changes, the selectors must be updated.  
    - Empty extraction arrays if site content is missing or access is blocked.  
  - **Version Requirements:** HTML Extract node v1+; no specific version constraints.  
  - **Notes:** Testing this node independently is critical to ensure selectors are accurate.

---

#### 2.4 Conditional Check

- **Overview:**  
  Checks whether the extraction yielded any items before proceeding to notification; prevents unnecessary alerts.

- **Nodes Involved:**  
  - If Items Found

- **Node Details:**  
  - **Name:** If Items Found  
  - **Type:** If Node (Conditional)  
  - **Configuration:**  
    - Condition: Checks if the length of the extracted items array (`{{ $json.length }}`) is **not equal** to 0.  
  - **Inputs:** Receives extracted data from `Extract Job/Product Info`.  
  - **Outputs:**  
    - True path: continues to `Format Notification Message`.  
    - False path: workflow ends silently (no new items).  
  - **Edge Cases / Failures:**  
    - If extraction produces an unexpected output format, condition may fail.  
    - Empty extraction arrays correctly end workflow without alerts.  
  - **Version Requirements:** Standard If node; compatible with all recent n8n versions.

---

#### 2.5 Notification Formatting

- **Overview:**  
  Converts extracted items into a formatted Markdown message for Telegram, summarizing new or updated jobs/products.

- **Nodes Involved:**  
  - Format Notification Message

- **Node Details:**  
  - **Name:** Format Notification Message  
  - **Type:** Function Node  
  - **Configuration:**  
    - JavaScript code iterates over extracted items:  
      - Accesses `JobTitle` and `JobLink` (or fallback properties like `ProductName`).  
      - Optionally appends `StockStatus` if available.  
      - Builds a Markdown-formatted string with bold titles and links.  
    - Outputs a single JSON object with property `notificationMessage`.  
  - **Inputs:** Receives data from `If Items Found` (True path).  
  - **Outputs:** Sends formatted text to `Send Telegram Alert`.  
  - **Edge Cases / Failures:**  
    - Missing or misnamed properties in extracted data lead to placeholders like "N/A" or "No link".  
    - Empty input array leads to message: "No new items found during this check." (though this path should not occur due to prior conditional).  
  - **Version Requirements:** Function node v1+ standard.

---

#### 2.6 Alert Dispatch

- **Overview:**  
  Sends the formatted notification message via Telegram Bot API to a specified chat.

- **Nodes Involved:**  
  - Send Telegram Alert

- **Node Details:**  
  - **Name:** Send Telegram Alert  
  - **Type:** Telegram Node  
  - **Configuration:**  
    - Credentials: Requires a Telegram API credential with a Bot Token obtained from @BotFather on Telegram.  
    - Chat ID: Must be set to the Telegram chat's unique ID where alerts are sent.  
    - Text: Dynamically set as `{{ $json.notificationMessage }}` from previous node.  
    - Parse Mode: Set to `Markdown` to allow bold and link formatting.  
  - **Inputs:** Receives formatted message from `Format Notification Message`.  
  - **Outputs:** Terminal node; no further outputs.  
  - **Edge Cases / Failures:**  
    - Invalid bot token or chat ID causes authentication errors.  
    - Telegram API rate limits may delay or block message sending.  
    - Message formatting errors if Markdown syntax is incorrect.  
  - **Version Requirements:** Telegram node v1+, credentials must be configured.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role          | Input Node(s)            | Output Node(s)                | Sticky Note                                                                                                           |
|-------------------------|----------------------|-------------------------|--------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Hourly Monitor Trigger   | Cron Trigger         | Workflow trigger        | None                     | Fetch Webpage Content         | This `Cron` node will trigger the workflow automatically every **hour**. Adjust schedule in node parameters.          |
| Fetch Webpage Content    | HTTP Request         | Download webpage HTML   | Hourly Monitor Trigger   | Extract Job/Product Info      | Change URL to your target site. Response format set to `string`. May require auth or advanced scraping if JS is used. |
| Extract Job/Product Info | HTML Extract         | Extract data with CSS   | Fetch Webpage Content    | If Items Found                | Define CSS selectors carefully; test output to ensure correct data extraction.                                        |
| If Items Found           | If Node              | Conditional check       | Extract Job/Product Info | Format Notification Message   | Checks if any items were extracted to decide whether to send notification.                                            |
| Format Notification Msg  | Function             | Format alert message    | If Items Found           | Send Telegram Alert           | Formats extracted data into Markdown message for Telegram alert.                                                      |
| Send Telegram Alert      | Telegram             | Send notification       | Format Notification Msg  | None                         | Requires Telegram Bot token and Chat ID. Set Parse Mode to Markdown.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger Node**  
   - Name: `Hourly Monitor Trigger`  
   - Type: Cron Trigger  
   - Set Mode to `everyHour` (or customize as needed for your monitoring frequency).  

2. **Add HTTP Request Node**  
   - Name: `Fetch Webpage Content`  
   - Type: HTTP Request  
   - Connect input from `Hourly Monitor Trigger`.  
   - Set URL to the webpage you want to monitor (e.g., `https://your-target-site.com/jobs`).  
   - Set Response Format to `string` to get raw HTML.  
   - If needed, configure authentication headers, cookies, or tokens depending on the target site.  

3. **Add HTML Extract Node**  
   - Name: `Extract Job/Product Info`  
   - Type: HTML Extract  
   - Connect input from `Fetch Webpage Content`.  
   - Set HTML field to `{{ $node["Fetch Webpage Content"].json.data }}` (use expression editor).  
   - Add Extract Operations:  
     - For each data point, add an operation with:  
       - Selector: Use browser developer tools to find CSS selector for the target element (e.g., job title).  
       - Attribute: Usually `textContent` for text or `href` for links.  
       - Property Name: e.g., `JobTitle`, `JobLink`.  
   - Test extraction to verify selectors are correct.  

4. **Add If Node for Conditional Check**  
   - Name: `If Items Found`  
   - Connect input from `Extract Job/Product Info`.  
   - Configure condition:  
     - Value 1: `{{ $json.length }}` (expression)  
     - Operation: `notEqual`  
     - Value 2: `0`  
   - This checks if any items were extracted.  

5. **Add Function Node to Format Notification**  
   - Name: `Format Notification Message`  
   - Connect input from `If Items Found` (True path).  
   - Enter the provided JavaScript function code which:  
     - Iterates over items.  
     - Extracts title, link, and optional stock status.  
     - Builds a Markdown message summarizing all items.  

6. **Add Telegram Node to Send Alert**  
   - Name: `Send Telegram Alert`  
   - Connect input from `Format Notification Message`.  
   - Choose or create Telegram API credential:  
     - Obtain Bot Token from @BotFather on Telegram.  
   - Set Chat ID to your Telegram chat’s unique identifier.  
     - Obtain by sending a message to your bot and querying `getUpdates` via Telegram API.  
   - Set Text field to `{{ $json.notificationMessage }}` (expression).  
   - Set Parse Mode to `Markdown`.  

7. **Save and Activate the Workflow**  
   - Test each step individually to ensure proper data flow and outputs.  
   - Run the workflow manually first, then enable automatic triggering.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                          | Context or Link                                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| To find CSS selectors for extraction, use browser developer tools: right-click element → Inspect → right-click code → Copy → Copy selector.                         | Standard browser dev tools (Chrome, Firefox).                                                                 |
| Telegram Bot setup instructions: create a bot using @BotFather, get Bot Token, send a message to your bot, then obtain chat ID via `https://api.telegram.org/bot...` | Telegram API documentation and BotFather instructions.                                                        |
| For sites with JavaScript-loaded content, consider using Puppeteer or Playwright within a `Code` node for scraping dynamic content.                                | n8n documentation on headless browser scraping techniques.                                                     |
| Adjust schedule in Cron node to avoid rate limiting or excessive requests on the target website.                                                                     | Best practice for ethical scraping and avoiding bans.                                                          |
| Markdown formatting in Telegram supports bolding (`**text**`), italics, links, etc.; verify message correctness before sending alerts.                            | Telegram Markdown parse mode documentation.                                                                    |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.