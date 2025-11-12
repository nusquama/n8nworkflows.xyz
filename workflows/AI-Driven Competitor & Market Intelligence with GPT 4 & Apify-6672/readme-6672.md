AI-Driven Competitor & Market Intelligence with GPT 4 & Apify

https://n8nworkflows.xyz/workflows/ai-driven-competitor---market-intelligence-with-gpt-4---apify-6672


# AI-Driven Competitor & Market Intelligence with GPT 4 & Apify

---

### 1. Workflow Overview

This workflow automates competitor and market intelligence gathering and analysis for IT companies using AI (GPT-4) and web scraping services (Apify). It targets sales, marketing, product, and strategy teams who need timely, actionable insights about competitors’ offerings, pricing, customer sentiment, and market trends.

The workflow is logically divided into these functional blocks:

- **1.1 Scheduled Data Collection:** Automatic triggering of data fetching from multiple online sources at set intervals.

- **1.2 Data Gathering:** Retrieval of competitor data via web scraping API and industry news/blog data via RSS feeds.

- **1.3 Data Aggregation & Pre-processing:** Combining and cleaning the heterogeneous data into a unified, AI-ready format.

- **1.4 AI-Powered Market & Competitor Analysis:** Using GPT-4 to analyze the cleaned data and produce structured insights.

- **1.5 Report Generation & Notification:** Creating a Google Docs report from AI output and notifying sales/marketing teams via Slack.

- **1.6 Data Storage & Personalized Sales Enablement:** Storing insights in a PostgreSQL database and optionally generating personalized sales talking points using AI.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Data Collection

**Overview:**  
Initiates the workflow on a scheduled basis (e.g., daily or weekly) to ensure data collection and analysis happen automatically and continuously.

**Nodes Involved:**  
- Scheduled Analysis Trigger

**Node Details:**  
- **Scheduled Analysis Trigger**  
  - *Type:* Schedule Trigger  
  - *Configuration:* Set to trigger on an interval schedule (default is daily; configurable).  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connects to both Apify and RSS Feed nodes to start data gathering.  
  - *Potential Failures:* Misconfiguration of schedule, node downtime, platform maintenance windows.

---

#### 2.2 Data Gathering

**Overview:**  
Fetches competitor-related data from web sources and industry news blogs. Uses external web scraping API (Apify) for competitor sites and RSS feed reader for news/blogs.

**Nodes Involved:**  
- Apify (HTTP Request)  
- Fetch News & Blog RSS

**Node Details:**  
- **Apify**  
  - *Type:* HTTP Request  
  - *Configuration:* POST request to Apify’s web scraper act endpoint with JSON body specifying URLs to scrape (competitor pricing pages, reviews, blogs).  
  - *Key Parameters:*  
    - `startUrls`: Array of URLs for competitor data sources.  
    - `pseudoUrls`: Pattern matching for blog articles on competitor site.  
    - `maxRequestsPerCrawl`: Limits crawl to 100 requests.  
    - Authorization header includes Apify API key (placeholder `[YOUR-API-KEY]`).  
  - *Input:* Trigger from Scheduled Analysis Trigger.  
  - *Output:* JSON data with scraped content.  
  - *Failures:* API key missing/invalid, rate limiting, network errors, malformed JSON response, website structure changes affecting scraping.  
  - *Version:* HTTP Request node v4.2.

- **Fetch News & Blog RSS**  
  - *Type:* RSS Feed Read  
  - *Configuration:* Reads multiple RSS feed URLs (placeholders: `[RSS_FEED_URL_1], [RSS_FEED_URL_2], [GOOGLE_ALERTS_RSS_URL]`) to gather latest news and blog posts.  
  - *Input:* Trigger from Scheduled Analysis Trigger.  
  - *Output:* RSS feed items with titles, descriptions, and links.  
  - *Failures:* Invalid RSS URLs, feed downtime, malformed feed data.  
  - *Version:* RSS Feed Read node v1.2.

---

#### 2.3 Data Aggregation & Pre-processing

**Overview:**  
Merges data from scraping and RSS feeds, then cleans and formats text content into a consistent structure ready for AI analysis.

**Nodes Involved:**  
- Combine Data Sources (Merge)  
- Pre-process Data for AI (Code)

**Node Details:**  
- **Combine Data Sources**  
  - *Type:* Merge  
  - *Configuration:* Merges two inputs by position (Apify and RSS feeds).  
  - *Input:* Outputs from Apify and Fetch News & Blog RSS nodes.  
  - *Output:* Combined item stream.  
  - *Failures:* Mismatched input lengths, null inputs.

- **Pre-process Data for AI**  
  - *Type:* Code (JavaScript)  
  - *Configuration:*  
    - Iterates through merged items.  
    - Strips HTML tags from scraped content or truncates text to 4000 characters (AI input limit).  
    - Extracts relevant fields: content, title, url, source.  
    - Filters out entries with less than 50 characters to avoid noise.  
  - *Key Expressions:* Uses regex to strip HTML, conditional checks on item structure.  
  - *Input:* From Combine Data Sources node.  
  - *Output:* Array of cleaned JSON objects ready for AI.  
  - *Failures:* Unexpected data structures, empty inputs, code runtime errors.

---

#### 2.4 AI-Powered Market & Competitor Analysis

**Overview:**  
Uses OpenAI GPT-4 to analyze the cleaned data and extract structured, actionable competitor insights including strengths, weaknesses, pricing, sentiment, and sales suggestions.

**Nodes Involved:**  
- AI Analysis & Competitor Insights

**Node Details:**  
- **AI Analysis & Competitor Insights**  
  - *Type:* OpenAI (Chat Completion) via Langchain node  
  - *Configuration:*  
    - Model: GPT-4  
    - System prompt sets AI role as expert market intelligence analyst.  
    - User prompt dynamically inserts concatenated cleaned data points from previous node.  
    - Requests output in structured JSON format including multiple intelligence aspects (competitor name, offerings, pricing, sentiment, etc.).  
  - *Input:* From Pre-process Data for AI node.  
  - *Output:* Raw AI JSON response with detailed competitor insights.  
  - *Credentials:* OpenAI API key required.  
  - *Failures:* API rate limits, malformed prompts, invalid JSON in output, network issues.

---

#### 2.5 Report Generation & Notification

**Overview:**  
Transforms AI insights into a formal Google Docs report and notifies the sales and marketing teams on Slack with a link to the report and key insights.

**Nodes Involved:**  
- Generate Market Intelligence Report (Google Docs)  
- Sales & Marketing Team Notification (Slack)

**Node Details:**  
- **Generate Market Intelligence Report**  
  - *Type:* Google Docs  
  - *Configuration:*  
    - Creates a new Google Doc from a predefined template.  
    - Populates placeholders with parsed values from AI JSON output (competitor name, strengths, weaknesses, opportunities, threats, sales talking points).  
    - Uses expressions like `{{ JSON.parse($json.choices[0].message.content).Strengths.join('\n- ') }}` to format lists.  
  - *Input:* From AI Analysis & Competitor Insights.  
  - *Output:* Document metadata including URL.  
  - *Credentials:* Google Docs OAuth2.  
  - *Failures:* Missing or invalid template, OAuth permission errors, expression parsing errors.

- **Sales & Marketing Team Notification**  
  - *Type:* Slack  
  - *Configuration:*  
    - Sends a message to a specified Slack channel (channel ID placeholder `[YOUR_SALES_TEAM_SLACK_CHANNEL_ID]`).  
    - Message includes competitor name from AI output and link to generated report.  
  - *Input:* From Generate Market Intelligence Report.  
  - *Output:* Slack message delivery confirmation.  
  - *Credentials:* Slack API token.  
  - *Failures:* Invalid channel ID, Slack API errors, connectivity issues.

---

#### 2.6 Data Storage & Personalized Sales Enablement

**Overview:**  
Stores the AI-generated intelligence into a PostgreSQL database for historic analysis and triggers an optional AI-powered node to generate customized sales talking points based on stored insights and sales context.

**Nodes Involved:**  
- Store Insights to Database (PostgreSQL)  
- Generate Personalized Sales Talking Points (OpenAI)

**Node Details:**  
- **Store Insights to Database**  
  - *Type:* PostgreSQL (Execute Query)  
  - *Configuration:*  
    - SQL query inserts or updates competitor profile records with latest AI insights (name, strengths, weaknesses, opportunities, threats, sales talking points).  
    - Uses `ON CONFLICT (name) DO UPDATE` to maintain up-to-date records.  
    - Maps data from AI JSON output with expressions.  
  - *Input:* From Sales & Marketing Team Notification.  
  - *Output:* Database operation result.  
  - *Credentials:* PostgreSQL connection credentials.  
  - *Failures:* Database connection failure, SQL syntax errors, data type mismatches.

- **Generate Personalized Sales Talking Points**  
  - *Type:* OpenAI (Chat Completion) via Langchain  
  - *Configuration:*  
    - Model: GPT-4  
    - System prompt defines AI as sales coach.  
    - User prompt includes competitor insights and a specific sales context with placeholders (e.g., prospect’s current competitor, prospect values).  
    - Generates 3-5 concise personalized sales talking points for reps.  
  - *Input:* From Store Insights to Database node.  
  - *Output:* AI-generated talking points for sales reps.  
  - *Credentials:* OpenAI API key.  
  - *Failures:* Prompt construction errors, API failures, missing context data.

---

### 3. Summary Table

| Node Name                         | Node Type                     | Functional Role                       | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                         |
|----------------------------------|-------------------------------|-------------------------------------|----------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------|
| Scheduled Analysis Trigger        | Schedule Trigger              | Initiates scheduled workflow runs   | None                             | Apify, Fetch News & Blog RSS           |                                                                                                   |
| Apify                            | HTTP Request                  | Scrapes competitor websites         | Scheduled Analysis Trigger        | Combine Data Sources                   |                                                                                                   |
| Fetch News & Blog RSS             | RSS Feed Read                 | Fetches industry news/blog data     | Scheduled Analysis Trigger        | Combine Data Sources                   |                                                                                                   |
| Combine Data Sources              | Merge                        | Combines scraped and RSS data       | Apify, Fetch News & Blog RSS      | Pre-process Data for AI                |                                                                                                   |
| Pre-process Data for AI           | Code                         | Cleans and formats data for AI      | Combine Data Sources              | AI Analysis & Competitor Insights     |                                                                                                   |
| AI Analysis & Competitor Insights | OpenAI (Chat Completion)     | Analyzes data to generate insights  | Pre-process Data for AI           | Generate Market Intelligence Report   |                                                                                                   |
| Generate Market Intelligence Report | Google Docs                | Creates report from AI insights     | AI Analysis & Competitor Insights | Sales & Marketing Team Notification   |                                                                                                   |
| Sales & Marketing Team Notification | Slack                      | Notifies team with report link      | Generate Market Intelligence Report | Store Insights to Database            |                                                                                                   |
| Store Insights to Database        | PostgreSQL                   | Stores insights in database          | Sales & Marketing Team Notification | Generate Personalized Sales Talking Points |                                                                                                   |
| Generate Personalized Sales Talking Points | OpenAI (Chat Completion) | Creates customized sales talking points | Store Insights to Database       | None                                  |                                                                                                   |
| Sticky Note                      | Sticky Note                  | Visual annotation                    | None                             | None                                  | ## Flow                                                                                           |
| Sticky Note1                     | Sticky Note                  | Workflow description and setup notes| None                             | None                                  | # Automated AI-Driven Competitor & Market Intelligence System (Detailed description and setup)    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n** and name it "AI-Driven Competitor & Market Intelligence with GPT 4 & Apify".

2. **Add a Schedule Trigger node** ("Scheduled Analysis Trigger"):  
   - Set interval to daily or weekly (e.g., every 1 day at 00:00).  
   - This node is the workflow’s entry point.

3. **Add an HTTP Request node** ("Apify"):  
   - Set HTTP Method to POST.  
   - URL: `https://api.apify.com/v2/acts/apify~web-scraper/run-sync`.  
   - Body Content Type: JSON (raw).  
   - Body:  
     ```json
     {
       "startUrls": [
         { "url": "https://www.competitorA.com/pricing" },
         { "url": "https://www.g2.com/products/competitorB/reviews" },
         { "url": "https://techcrunch.com/tag/it-trends/" }
       ],
       "pseudoUrls": [
         { "url": "https://www.competitorA.com/blog/[.*]", "selector": "article" }
       ],
       "maxRequestsPerCrawl": 100
     }
     ```  
   - Add HTTP Header: `Authorization: Bearer [YOUR-API-KEY]` (replace with your Apify API key).  
   - Connect "Scheduled Analysis Trigger" output to this node.

4. **Add an RSS Feed Read node** ("Fetch News & Blog RSS"):  
   - Enter multiple RSS feed URLs separated by commas (e.g., competitor blogs, Google Alerts RSS).  
   - Connect "Scheduled Analysis Trigger" output to this node.

5. **Add a Merge node** ("Combine Data Sources"):  
   - Set Mode to "Merge By Position".  
   - Connect outputs from "Apify" and "Fetch News & Blog RSS" inputs to this node.

6. **Add a Code node** ("Pre-process Data for AI"):  
   - Paste the JavaScript code that:  
     - Iterates over merged inputs.  
     - Extracts and cleans HTML content or text.  
     - Limits content length to 4000 characters.  
     - Filters out short content (<50 chars).  
     - Outputs an array of objects with `content`, `title`, `url`, `source`.  
   - Connect output of "Combine Data Sources" to this node.

7. **Add an OpenAI node** ("AI Analysis & Competitor Insights"):  
   - Select OpenAI credentials.  
   - Set Resource to "Chat Completion".  
   - Choose model "gpt-4".  
   - Configure messages:  
     - System prompt defining AI as expert market analyst.  
     - User prompt dynamically inserting cleaned data as formatted text for analysis.  
     - Request structured JSON output containing fields like Competitor Name, Strengths, Weaknesses, etc.  
   - Connect output of "Pre-process Data for AI" to this node.

8. **Add a Google Docs node** ("Generate Market Intelligence Report"):  
   - Select Google Docs OAuth2 credential.  
   - Operation: "Create document from template".  
   - Provide template document ID (Google Docs template with placeholders).  
   - Map AI output fields (parsed JSON) to template placeholders (e.g., competitorName, strengths, weaknesses).  
   - Connect output of "AI Analysis & Competitor Insights" to this node.

9. **Add a Slack node** ("Sales & Marketing Team Notification"):  
   - Select Slack API credential.  
   - Set channel to your sales/marketing Slack channel ID.  
   - Compose message referencing competitor name and document URL from previous node output.  
   - Connect output of "Generate Market Intelligence Report" to this node.

10. **Add a PostgreSQL node** ("Store Insights to Database"):  
    - Select PostgreSQL credentials.  
    - Set Operation to "Execute Query".  
    - Use SQL INSERT ... ON CONFLICT query to insert/update competitor insights from AI output.  
    - Connect output of "Sales & Marketing Team Notification" to this node.

11. **Add another OpenAI node** ("Generate Personalized Sales Talking Points") [optional]:  
    - Configure similarly to the main AI node but with prompt tailored for sales coaching.  
    - Input competitor insights and sales context placeholders.  
    - Connect output of "Store Insights to Database" to this node.

12. **Activate and test the workflow:**  
    - Run manually to verify each node’s output.  
    - Check data correctness, AI output validity, report generation, Slack notifications, and database storage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Automated AI-Driven Competitor & Market Intelligence System solves the problem of manual competitor tracking by providing continuous, automated data collection and AI-powered analysis for IT companies. It supports sales, marketing, product, and business strategy teams. Setup involves API keys for OpenAI, Apify, Google Docs, Slack, and database credentials. The workflow runs on a schedule and produces structured, actionable insights and reports, enabling proactive market strategies.                                                                                                                                                                                                                                         | Detailed workflow description and setup instructions included in Sticky Note1 within the workflow.                   |
| For accurate scraping, maintain updated Apify actor configurations and monitor any structural changes to competitor websites. Consider edge cases such as API rate limits, malformed data, and network failures. Use robust error handling and retries in production environments.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Best practices related to scraping and API reliability.                                                              |
| Google Docs template must have matching placeholder variables corresponding to the AI output fields (e.g., `{{competitorName}}`, `{{strengths}}`). Slack channel ID and credential must be valid and have permissions to post messages. PostgreSQL table `competitor_profiles` must be pre-created with columns matching the insert query.                                                                                                                                                                                                                                                                                                                                                                                                                         | Configuration notes for external services integration.                                                                |
| OpenAI prompts are designed to output strictly structured JSON. Validate AI output for JSON parsing errors, and consider adding fallback or error handling nodes for robustness.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | AI prompt engineering advice for consistent output.                                                                   |
| Sample competitor URLs and RSS feeds used in the workflow are placeholders; replace them with real URLs relevant to your market and competitors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Reminder to customize URLs according to use case.                                                                     |

---

**Disclaimer:** The provided content is extracted and documented from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected data. All data processed is legal and publicly available.

---