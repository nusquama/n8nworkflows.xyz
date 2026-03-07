Score property investments using Claude (Anthropic), Google Sheets and Slack

https://n8nworkflows.xyz/workflows/score-property-investments-using-claude--anthropic---google-sheets-and-slack-13686


# Score property investments using Claude (Anthropic), Google Sheets and Slack

This document provides a technical breakdown of the **AI Property Investment Scorer** workflow in n8n. This system automates the process of discovery, data enrichment, and financial analysis for real estate investments using Claude AI.

### 1. Workflow Overview

The workflow is designed to identify high-potential property investments by combining web scraping with multi-source market data enrichment. It evaluates properties against financial metrics (yield, cash flow) and qualitative factors (demographics, risk) using an AI-driven scoring model.

**Logical Phases:**
*   **1.1 Data Acquisition:** Triggers a search based on pre-defined parameters and scrapes property listing data.
*   **1.2 Market Enrichment:** Simultaneously fetches suburb-level market stats, demographics, and vacancy rates from external APIs.
*   **1.3 Financial Modeling:** Normalizes all data and calculates estimated net yields and weekly cash flow.
*   **1.4 AI Evaluation:** Uses Claude AI (Anthropic) to perform a SWOT analysis and generate a numerical investment score.
*   **1.5 Reporting & Archiving:** Filters top-performing properties, logs all analyzed data to Google Sheets, and sends a formatted digest to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Listing Scrape
This block defines the search criteria and retrieves the raw HTML from property portals.
*   **Nodes Involved:** `Daily Schedule Trigger`, `Configure Scrape Targets`, `Scrape Property Listings`, `Parse Listing Data`.
*   **Node Details:**
    *   **Configure Scrape Targets (Set):** Defines the `searchUrl`, `suburb`, `maxListings` (default 15), and `scoreThreshold` (default 65). It generates a unique `jobId` for tracking.
    *   **Scrape Property Listings (HTTP Request):** Performs a GET request to the target portal. It includes headers (User-Agent) to mimic a browser.
    *   **Parse Listing Data (Code):** Uses JavaScript to extract JSON-LD structured data from the HTML. It includes a fallback mechanism to generate synthetic data if the site's structure fails to parse.
*   **Potential Failures:** Blocked requests (403), changed HTML selectors, or timeouts.

#### 2.2 Market Data Enrichment
Enriches individual listings with macro and micro-economic data.
*   **Nodes Involved:** `Fetch Suburb Market Data`, `Fetch Area Demographics`, `Fetch Rental Yield Data`, `Merge Enrichment Sources`, `Combine All Property Data`.
*   **Node Details:**
    *   **HTTP Requests:** Connects to RapidAPI (Realty data), ABS (Demographics), and SQM Research (Vacancy rates).
    *   **Combine All Property Data (Code):** This is the financial engine. It calculates:
        *   **Net Yield:** Gross rent minus estimated expenses (28% default).
        *   **Cash Flow:** Weekly rent minus mortgage interest (80% LVR at 6.5% interest) and expenses.
*   **Potential Failures:** API credit exhaustion, invalid postcodes, or non-matching property types across different APIs.

#### 2.3 AI Investment Scoring (Claude)
The "brain" of the workflow, where raw data is converted into professional investment advice.
*   **Nodes Involved:** `Score Investment with Claude AI`, `Claude AI Model`, `Parse AI Investment Score`.
*   **Node Details:**
    *   **Claude AI Model:** Uses `claude-sonnet-4-20250514` with a low temperature (0.15) for consistency.
    *   **Score Investment Node (AI Agent):** Takes a comprehensive prompt including listing details, market stats, and financials. It forces the AI to return a specific JSON schema including an `investmentScore` (0–100), `rating`, and `riskFlags`.
    *   **Parse AI Investment Score (Code):** Sanitizes the AI output, stripping any markdown or conversational text to ensure a clean JSON object.

#### 2.4 Filter, Report & Notify
Finalizes the data flow by separating "Top Picks" from the general list.
*   **Nodes Involved:** `Filter Top Picks by Score`, `Save All Scored Listings to Sheets`, `Format Investment Report`, `Send Top Picks Digest to Slack`.
*   **Node Details:**
    *   **Filter Top Picks:** Only allows properties through if the `investmentScore` is >= threshold and the recommendation is not "AVOID".
    *   **Save All Scored Listings (Google Sheets):** Appends *every* analyzed property to a master sheet for historical audit.
    *   **Format Investment Report (Code):** Transforms the top results into "Slack Blocks," creating a visually structured message with ranks, scores, and deep links.
    *   **Send Top Picks Digest (HTTP Request):** Posts the report to a dedicated Slack channel via Webhook/API.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily Schedule Trigger | Schedule Trigger | Workflow Entry | (None) | Configure Scrape Targets | 1. Trigger & Listing Scrape |
| Configure Scrape Targets | Set | Parameter Config | Daily Schedule Trigger | Scrape Property Listings | 1. Trigger & Listing Scrape |
| Scrape Property Listings | HTTP Request | Web Scraping | Configure Scrape Targets | Parse Listing Data | 1. Trigger & Listing Scrape |
| Parse Listing Data | Code | Data Extraction | Scrape Property Listings | Enrichment Nodes | 1. Trigger & Listing Scrape |
| Fetch Suburb Market Data | HTTP Request | Market API | Parse Listing Data | Merge Enrichment | 2. Market Data Enrichment |
| Fetch Area Demographics | HTTP Request | Demographics API | Parse Listing Data | Merge Enrichment | 2. Market Data Enrichment |
| Fetch Rental Yield Data | HTTP Request | Rental API | Parse Listing Data | Merge Enrichment | 2. Market Data Enrichment |
| Merge Enrichment Sources | Merge | Synchronization | Enrichment Nodes | Combine Data | 2. Market Data Enrichment |
| Combine All Property Data | Code | Financial Modeling | Merge Enrichment | AI Scoring | 2. Market Data Enrichment |
| Score Investment | AI Agent | Investment Analysis | Combine Data | Parse AI Score | 3. AI Investment Scoring |
| Claude AI Model | ChatAnthropic | LLM Engine | (Used by Agent) | (Used by Agent) | 3. AI Investment Scoring |
| Parse AI Investment Score | Code | JSON Cleaning | AI Scoring | Filter / Google Sheets | 3. AI Investment Scoring |
| Filter Top Picks by Score | Filter | Quality Control | Parse AI Score | Format Report | 4. Filter, Report & Notify |
| Save All Scored Listings | Google Sheets | Data Archiving | Parse AI Score | (None) | 4. Filter, Report & Notify |
| Format Investment Report | Code | Slack Block Builder | Filter Top Picks | Send to Slack | 4. Filter, Report & Notify |
| Send Top Picks Digest | HTTP Request | Communication | Format Report | Return Final Report | 4. Filter, Report & Notify |

---

### 4. Reproducing the Workflow from Scratch

1.  **Initialize Parameters:** Add a `Schedule Trigger` and connect a `Set` node. Create fields for `searchUrl`, `suburb`, and `scoreThreshold`.
2.  **Web Scrape:** Add an `HTTP Request` node. Set the method to GET and provide the URL from the previous step. In headers, add a `User-Agent`.
3.  **Parsing Logic:** Add a `Code` node. Use regex or a library like Cheerio (if available in your n8n environment) to find `<script type="application/ld+json">`. Map these to standard fields (price, beds, address).
4.  **API Enrichment:** Create three parallel `HTTP Request` nodes:
    *   **API 1 (Market):** Target a property data API (e.g., Domain/Zillow via RapidAPI).
    *   **API 2 (Demographics):** Target a census or demographic API.
    *   **API 3 (Yield):** Target a rental stats API.
5.  **Synchronization:** Use a `Merge` node (Merge by Position) to bring the three API outputs together.
6.  **Financial Calc:** Add a `Code` node. Use the output from the Merge node to calculate `netYield` (Net Rent / Price) and `cashFlow` (Rent - Interest - Expenses).
7.  **AI Setup:** 
    *   Add an `AI Agent` node. 
    *   Connect the `ChatAnthropic` model node (requires Anthropic API Key).
    *   Set the System Prompt to act as a "Property Analyst" and enforce JSON output.
8.  **Data Extraction:** Add a `Code` node to parse the AI's string response into a JSON object using `JSON.parse()`.
9.  **Storage:** Connect a `Google Sheets` node. Set the operation to `Append`. Map the properties and AI scores to your sheet columns.
10. **Filtering:** Add a `Filter` node. Check if `aiScore.investmentScore` >= your threshold.
11. **Notification:** Add a `Code` node to generate Slack "Blocks" based on the filtered results. Connect an `HTTP Request` node to send this payload to the Slack `chat.postMessage` endpoint.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **API Keys Required** | Anthropic (Claude), RapidAPI (Property), Google (Sheets), Slack (OAuth/Bot). |
| **Regional Logic** | The current logic is tuned for the Australian market (NSW, ABS data). |
| **Scoring Logic** | 80-100: Exceptional, 65-79: Good, <50: Weak. |
| **Prompt Engineering** | Uses a strictly objective, data-driven system message to prevent AI hallucinations. |