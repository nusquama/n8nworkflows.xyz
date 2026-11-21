Extract Xiaohongshu comments to CSV with Apify and UnitBytes

https://n8nworkflows.xyz/workflows/extract-xiaohongshu-comments-to-csv-with-apify-and-unitbytes-21008


# Extract Xiaohongshu comments to CSV with Apify and UnitBytes

### 1. Workflow Overview

This workflow automates the extraction of comments from Xiaohongshu (RedNote) posts using the UnitBytes Apify actor, processes the resulting dataset items, and converts them into a structured CSV format. Designed for market research, customer sentiment tracking, and social media monitoring, the pipeline operates on a fixed daily schedule to maintain up-to-date datasets without manual intervention.

The logical flow is organized into two primary blocks:
- **1.1 Trigger & Extraction Pipeline:** Handles the automated time-based schedule, initializes the required API authentication tokens and scraper configurations, executes the synchronous Apify actor call, and maps the returned raw dataset objects.
- **1.2 Data Normalization & Export:** Cleans and standardizes the individual extracted fields and transforms the structured records into a downloadable CSV file.

---

### 2. Block-by-Block Analysis

#### Block 1: Trigger & Extraction Pipeline
- **Overview:** This block initiates the automated process on a 24-hour interval, defines the necessary parameters for the scraping job, and communicates with the Apify API to fetch Xiaohongshu comment data synchronously.
- **Nodes Involved:** 
  - `Schedule Trigger`
  - `1. Set Apify Token & Input`
  - `2. Run UnitBytes (xiaohongshu-rednote-comments-scraper)`

- **Node Details:**
  - **Schedule Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration Choices:* Configured to fire at an interval of every 24 hours.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: None (entry point). Output: Connected to `1. Set Apify Token & Input`.
    - *Version-Specific Requirements:* Version 1.2.
    - *Edge Cases or Potential Failure Types:* Missed executions if the n8n instance is offline during the scheduled trigger time.
    - *Sub-Workflow Reference:* None.

  - **1. Set Apify Token & Input**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation / Initialization Node)
    - *Configuration Choices:* Sets two persistent variables: `apifyToken` (string placeholder) and `actorInput` (JSON object containing target URLs, comment limits, proxy configurations, and regional proxy rules).
    - *Key Expressions or Variables:* Static strings and JSON object assignments.
    - *Input and Output Connections:* Input: `Schedule Trigger`. Output: Connected to `2. Run UnitBytes (xiaohongshu-rednote-comments-scraper)`.
    - *Version-Specific Requirements:* Version 3.4.
    - *Edge Cases or Potential Failure Types:* Authentication failures if `YOUR_APIFY_API_TOKEN` is not replaced with a valid Apify token.
    - *Sub-Workflow Reference:* None.

  - **2. Run UnitBytes (xiaohongshu-rednote-comments-scraper)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration Node)
    - *Configuration Choices:* Executes an HTTP POST request to the Apify synchronous dataset run endpoint (`run-sync-get-dataset-items`). Sends the `actorInput` payload as a JSON string and authenticates via the query parameter token.
    - *Key Expressions or Variables:* `={{ $json.apifyToken }}` for authentication and `={{ JSON.stringify($json.actorInput) }}` for the request body.
    - *Input and Output Connections:* Input: `1. Set Apify Token & Input`. Output: Connected to `3. Normalize Output Fields`.
    - *Version-Specific Requirements:* Version 4.2.
    - *Edge Cases or Potential Failure Types:* Request timeouts (configured to 300,000ms), invalid Apify actor input syntax, insufficient Apify account credits, or target URL blocks by the platform.
    - *Sub-Workflow Reference:* None.

---

#### Block 2: Data Normalization & Export
- **Overview:** This block processes the raw API payload, standardizes specific attributes into explicit top-level fields, and packages the refined data into a formatted CSV document.
- **Nodes Involved:** 
  - `3. Normalize Output Fields`
  - `4. Export to CSV File`

- **Node Details:**
  - **3. Normalize Output Fields**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node)
    - *Configuration Choices:* Maps incoming JSON properties to strict field names (`id`, `noteId`, `noteUrl`, `content`, `likeCount`, `subCommentCount`) while retaining other incoming fields.
    - *Key Expressions or Variables:* 
      - `id`: `={{ $json.id }}`
      - `noteId`: `={{ $json.noteId }}`
      - `noteUrl`: `={{ $json.noteUrl }}`
      - `content`: `={{ $json.content }}`
      - `likeCount`: `={{ $json.likeCount }}`
      - `subCommentCount`: `={{ $json.subCommentCount }}`
    - *Input and Output Connections:* Input: `2. Run UnitBytes (xiaohongshu-rednote-comments-scraper)`. Output: Connected to `4. Export to CSV File`.
    - *Version-Specific Requirements:* Version 3.4.
    - *Edge Cases or Potential Failure Types:* Property mapping failures if the incoming dataset schema from Apify changes unexpectedly.
    - *Sub-Workflow Reference:* None.

  - **4. Export to CSV File**
    - *Type and Technical Role:* `n8n-nodes-base.convertToFile` (Data Formatting Node)
    - *Configuration Choices:* Converts the incoming structured items into a CSV format and sets the output file name to `xiaohongshu-comments-scraper-dataset-export.csv`.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: `3. Normalize Output Fields`. Output: None (Terminal export node).
    - *Version-Specific Requirements:* Version 1.1.
    - *Edge Cases or Potential Failure Types:* Empty dataset inputs resulting in empty or malformed CSV files.
    - *Sub-Workflow Reference:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Triggers pipeline every 24 hours | None | 1. Set Apify Token & Input | ## ⚡ Xiaohongshu (RedNote) Comments & Customer Sentiment Scraper — Automated n8n Pipeline<br>**Powered by UnitBytes on Apify** ($0.00249 / item)<br><br>### 🚀 30-Second Setup Instructions<br>1. **Get your free Apify API Token**: Open xiaohongshu-rednote-comments-scraper on Apify Store and copy your API token from **Settings → Integrations**.<br>2. **Paste your token & search parameters**: Double-click the **"1. Set Apify Token & Input"** node and replace `YOUR_APIFY_API_TOKEN` with your token.<br>3. **Click "Test Workflow"**: The HTTP Request node calls the synchronous UnitBytes Apify endpoint, normalizes the extracted records, and exports a clean CSV file ready for Google Sheets, Airtable, Slack, or OpenAI.<br><br>📖 **Documentation & Schema Reference**: https://unitbytes.com/actors/xiaohongshu-comments-scraper/ |
| 1. Set Apify Token & Input | `n8n-nodes-base.set` | Initializes Apify token and actor parameters | Schedule Trigger | 2. Run UnitBytes (xiaohongshu-rednote-comments-scraper) | ## ⚡ Xiaohongshu (RedNote) Comments & Customer Sentiment Scraper — Automated n8n Pipeline<br>**Powered by UnitBytes on Apify** ($0.00249 / item)<br><br>### 🚀 30-Second Setup Instructions<br>1. **Get your free Apify API Token**: Open xiaohongshu-rednote-comments-scraper on Apify Store and copy your API token from **Settings → Integrations**.<br>2. **Paste your token & search parameters**: Double-click the **"1. Set Apify Token & Input"** node and replace `YOUR_APIFY_API_TOKEN` with your token.<br>3. **Click "Test Workflow"**: The HTTP Request node calls the synchronous UnitBytes Apify endpoint, normalizes the extracted records, and exports a clean CSV file ready for Google Sheets, Airtable, Slack, or OpenAI.<br><br>📖 **Documentation & Schema Reference**: https://unitbytes.com/actors/xiaohongshu-comments-scraper/ |
| 2. Run UnitBytes (xiaohongshu-rednote-comments-scraper) | `n8n-nodes-base.httpRequest` | Calls Apify actor synchronously to scrape comments | 1. Set Apify Token & Input | 3. Normalize Output Fields | ## ⚡ Xiaohongshu (RedNote) Comments & Customer Sentiment Scraper — Automated n8n Pipeline<br>**Powered by UnitBytes on Apify** ($0.00249 / item)<br><br>### 🚀 30-Second Setup Instructions<br>1. **Get your free Apify API Token**: Open xiaohongshu-rednote-comments-scraper on Apify Store and copy your API token from **Settings → Integrations**.<br>2. **Paste your token & search parameters**: Double-click the **"1. Set Apify Token & Input"** node and replace `YOUR_APIFY_API_TOKEN` with your token.<br>3. **Click "Test Workflow"**: The HTTP Request node calls the synchronous UnitBytes Apify endpoint, normalizes the extracted records, and exports a clean CSV file ready for Google Sheets, Airtable, Slack, or OpenAI.<br><br>📖 **Documentation & Schema Reference**: https://unitbytes.com/actors/xiaohongshu-comments-scraper/ |
| 3. Normalize Output Fields | `n8n-nodes-base.set` | Flattens and standardizes dataset attributes | 2. Run UnitBytes (xiaohongshu-rednote-comments-scraper) | 4. Export to CSV File | ### 📊 Connect Your Destination<br>The **"3. Normalize Output Fields"** node outputs flat records (`id, noteId, noteUrl, content, likeCount`).<br><br>- **Google Sheets / Airtable**: Connect a Google Sheets (`Append or Update Row`) node after Step 3.<br>- **Slack / Telegram / Email Alerts**: Connect a Filter node + Slack node to alert your team on new deals or reviews.<br>- **AI Agent / OpenAI**: Connect an OpenAI / LLM Chain node to summarize or classify extracted items. |
| 4. Export to CSV File | `n8n-nodes-base.convertToFile` | Converts normalized data into a CSV file | 3. Normalize Output Fields | None | ### 📊 Connect Your Destination<br>The **"3. Normalize Output Fields"** node outputs flat records (`id, noteId, noteUrl, content, likeCount`).<br><br>- **Google Sheets / Airtable**: Connect a Google Sheets (`Append or Update Row`) node after Step 3.<br>- **Slack / Telegram / Email Alerts**: Connect a Filter node + Slack node to alert your team on new deals or reviews.<br>- **AI Agent / OpenAI**: Connect an OpenAI / LLM Chain node to summarize or classify extracted items. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node.
   - Set the interval rule to trigger every `24` hours.

2. **Create the Configuration Node:**
   - Add a **Set** node named `1. Set Apify Token & Input`.
   - Add a string assignment named `apifyToken` with the value `YOUR_APIFY_API_TOKEN`.
   - Add an object assignment named `actorInput` containing the JSON payload:
     ```json
     {
       "startUrls": [{"url": "https://www.xiaohongshu.com/explore/64256e08+1234567890e?xsec_token=ABivcR3TWdHqjbjX2xyNQkAS3_CZsrXHUX9r1tNqo4148=&xsec_source=pc_feed"}],
       "maxCommentsPerNote": 50,
       "maxSubCommentsPerComment": 10,
       "proxyConfiguration": {
         "useApifyProxy": true,
         "apifyProxyGroups": ["RESIDENTIAL"],
         "apifyProxyCountry": "HK"
       }
     }
     ```
   - Connect **Schedule Trigger** output to this node.

3. **Create the API Request Node:**
   - Add an **HTTP Request** node named `2. Run UnitBytes (xiaohongshu-rednote-comments-scraper)`.
   - Set Method to `POST`.
   - Set URL to: `https://api.apify.com/v2/acts/unitbytes~xiaohongshu-rednote-comments-scraper/run-sync-get-dataset-items?token={{ $json.apifyToken }}&format=json&clean=true`.
   - Add Header Parameter: `Content-Type` = `application/json`.
   - Configure Body to send JSON, using the expression: `={{ JSON.stringify($json.actorInput) }}`.
   - Under Options, set Timeout to `300000` ms.
   - Connect `1. Set Apify Token & Input` output to this node.

4. **Create the Normalization Node:**
   - Add a **Set** node named `3. Normalize Output Fields`.
   - Enable "Include Other Fields".
   - Add string assignments:
     - `id` = `={{ $json.id }}`
     - `noteId` = `={{ $json.noteId }}`
     - `noteUrl` = `={{ $json.noteUrl }}`
     - `content` = `={{ $json.content }}`
     - `likeCount` = `={{ $json.likeCount }}`
     - `subCommentCount` = `={{ $json.subCommentCount }}`
   - Connect `2. Run UnitBytes (xiaohongshu-rednote-comments-scraper)` output to this node.

5. **Create the Export Node:**
   - Add a **Convert to File** node named `4. Export to CSV File`.
   - Set Operation to `CSV`.
   - Set File Name to `xiaohongshu-comments-scraper-dataset-export.csv`.
   - Connect `3. Normalize Output Fields` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Actor Documentation & Schema Reference | [UnitBytes Xiaohongshu Comments Scraper](https://unitbytes.com/actors/xiaohongshu-comments-scraper/) |
| Apify Store Listing & Token Generation | [Apify Store Page](https://apify.com/unitbytes/xiaohongshu-rednote-comments-scraper?fpr=939u3w&fp_sid=n8n_template) |