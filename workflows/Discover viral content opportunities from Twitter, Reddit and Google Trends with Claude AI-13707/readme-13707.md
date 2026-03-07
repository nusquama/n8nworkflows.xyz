Discover viral content opportunities from Twitter, Reddit and Google Trends with Claude AI

https://n8nworkflows.xyz/workflows/discover-viral-content-opportunities-from-twitter--reddit-and-google-trends-with-claude-ai-13707


# Discover viral content opportunities from Twitter, Reddit and Google Trends with Claude AI

This technical document provides a comprehensive breakdown of the **Smart Viral Content Opportunity Spotter** workflow. This system automates the discovery of trending topics across multiple social platforms, filters them based on niche relevance, and uses Claude AI to generate actionable content strategies.

---

### 1. Workflow Overview

The workflow is designed for content creators and marketing teams to automate the "ideation" phase of content creation. It scans three major data sources every 2 hours, applies a custom "Viral Potential" algorithm, and delivers a structured report.

**Logical Lifecycle:**
*   **1.1 Trend Collection:** Simultaneous fetching of data from X (Twitter), Reddit, and Google Trends.
*   **1.2 Analysis & Scoring:** Data normalization, keyword filtering, deduplication, and algorithmic scoring based on engagement and recency.
*   **1.3 AI Content Strategy:** Leveraging Claude 3.5 Sonnet to transform raw trends into 5 specific content ideas (hooks, key points, and platforms).
*   **1.4 Reporting & Delivery:** Multichannel distribution via Email, Slack, and database logging.

---

### 2. Block-by-Block Analysis

#### 2.1 Trend Collection
*   **Overview:** Fetches raw data from the web. This block ensures a diverse input stream from three distinct types of social signals (search intent, community discussion, and real-time news).
*   **Nodes Involved:** `Every 2 Hours - Trend Scanner`, `Fetch Twitter/X Trends`, `Fetch Reddit Hot Topics`, `Fetch Google Trends`, `Parse Twitter Data`, `Parse Reddit Data`, `Parse Google Trends`.
*   **Node Details:**
    *   **HTTP Request Nodes:** Use OAuth2 for X and Reddit. Google Trends uses a public daily trends endpoint.
    *   **Parsing (Code Nodes):** These nodes normalize disparate JSON structures into a unified format: `topic`, `description`, `url`, `engagement`, `timestamp`.
    *   **Edge Cases:** Handles Google's non-standard JSON prefix (`)迎', \n`) and calculates engagement differently per platform (Traffic for Google, Upvotes for Reddit, Volume for X).

#### 2.2 Filtering & Scoring
*   **Overview:** Sifts through the noise. It discards trends that don't match the user's defined niche and calculates a score to prioritize what to work on first.
*   **Nodes Involved:** `Merge All Trend Sources`, `Filter by Niche Keywords`, `Remove Duplicate Topics`, `Calculate Viral Potential Score`, `Filter High Potential Only`, `Sort by Viral Score`.
*   **Node Details:**
    *   **Filter by Niche Keywords:** Uses a standard array of keywords (configured in the `Load Niche Config` node) to perform a substring match against topics and descriptions.
    *   **Calculate Viral Potential Score:** A weighted algorithm assigning up to 100 points: Engagement (40), Recency (30), Source Credibility (15), and Keyword Relevance (15).
    *   **Filter High Potential Only:** Drops any trend with a score below 40.

#### 2.3 AI Content Generation
*   **Overview:** Transforms data into strategy. It checks if the topic was suggested recently and then sends it to Claude AI.
*   **Nodes Involved:** `Check Already Covered`, `AI - Generate Content Ideas`, `Parse AI Content Ideas`, `Add Research Links`.
*   **Node Details:**
    *   **Check Already Covered:** Uses n8n `staticData` to store a history of suggested topics (30-day window) to prevent duplicate suggestions in subsequent runs.
    *   **AI - Generate Content Ideas:** Sends a structured prompt to Claude (Anthropic). It requests a JSON response containing format, hook, key points, and platform.
    *   **Add Research Links:** Dynamically generates search URLs for Google, Twitter, YouTube, and Reddit to expedite the creator's research phase.

#### 2.4 Reporting & Delivery
*   **Overview:** Formats the final output for human consumption and long-term tracking.
*   **Nodes Involved:** `Create Opportunity Report`, `Aggregate All Opportunities`, `Send Email Digest`, `Send Slack Summary`, `Log to Content Database`, `Mark Topic as Suggested`.
*   **Node Details:**
    *   **Aggregation:** Consolidates multiple items into a single digest object.
    *   **Delivery:** Sends an HTML-formatted email and a Slack notification.
    *   **Persistence:** Inserts the data into a Postgres database for historical performance analysis.
    *   **Mark Topic as Suggested:** Saves the topic name to n8n's internal memory to ensure the "Deduplication over time" logic works.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every 2 Hours - Trend Scanner | Schedule Trigger | Workflow Entry | (None) | Fetch Twitter, Reddit, Google | Runs every 2 hours to capture fresh trending topics |
| Fetch Twitter/X Trends | HTTP Request | Data Fetching | Trend Scanner | Parse Twitter Data | Gets trending topics and hashtags from Twitter API |
| Fetch Reddit Hot Topics | HTTP Request | Data Fetching | Trend Scanner | Parse Reddit Data | Retrieves trending posts from relevant subreddits |
| Fetch Google Trends | HTTP Request | Data Fetching | Trend Scanner | Parse Google Trends | Gets trending searches from Google Trends API |
| Parse Twitter Data | Code | Normalization | Fetch Twitter | Merge Sources | Normalizes Twitter trend data into standard format |
| Parse Reddit Data | Code | Normalization | Fetch Reddit | Merge Sources | Normalizes Reddit post data into standard format |
| Parse Google Trends | Code | Normalization | Fetch Google | Merge Sources | Normalizes Google Trends data into standard format |
| Merge All Trend Sources | Merge | Data Union | Parse Nodes | Filter by Keywords | Combines data from Twitter, Reddit, and Google Trends |
| Filter by Niche Keywords | Code | Filtering | Merge Sources | Remove Duplicates | Matches trends against your niche keywords for relevance |
| Remove Duplicate Topics | Code | Cleaning | Filter Keywords | Viral Score Node | Deduplicates similar trends from different sources |
| Calculate Viral Potential Score | Code | Intelligence | Remove Duplicates | Filter High Pot. | Scores each trend based on engagement, recency, and growth |
| Filter High Potential Only | Filter | Quality Control | Viral Score Node | Sort by Score | Only keeps trends with viral score above 40 |
| Sort by Viral Score | Code | Ranking | Filter High Pot. | Check Covered | Ranks opportunities by score (highest first) |
| Check Already Covered | Code | History Check | Sort Node | AI Generate | Prevents suggesting topics you've already created content about |
| AI - Generate Content Ideas | HTTP Request | Generative AI | Check Covered | Parse AI Ideas | Uses Claude AI to generate creative content ideas for each trend |
| Parse AI Content Ideas | Code | Data Extraction | AI Generate | Add Research Links | Extracts and structures AI-generated content ideas |
| Add Research Links | Code | Enrichment | Parse AI Ideas | Create Report | Adds relevant research and reference links for each opportunity |
| Create Opportunity Report | Code | Formatting | Add Research | Aggregate | Generates comprehensive report for each content opportunity |
| Aggregate All Opportunities | Code | Data Union | Create Report | Email, Slack | Combines all opportunities into a single digest |
| Send Email Digest | Email Send | Delivery | Aggregate | Postgres | Sends comprehensive email with all content opportunities |
| Send Slack Summary | Slack | Delivery | Aggregate | Postgres | Posts quick summary to Slack channel |
| Log to Content Database | Postgres | Storage | Email, Slack | Mark Suggested | Stores opportunities in database for tracking and analytics |
| Mark Topic as Suggested | Code | Memory Update | Postgres | Load Niche Config | Updates workflow memory to track suggested topics |
| Load Niche Config | Code | Configuration | Mark Suggested | (End) | Defines your niche keywords, subreddits, and tracking parameters |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Ensure you have valid credentials for **Twitter (X) OAuth2**, **Reddit OAuth2**, **Anthropic (API Key)**, **SMTP (Email)**, and optionally **Slack** and **Postgres**.
2.  **Configuration Node:** Create a Code node named `Load Niche Config`. Set it to return an object containing an array of `keywords` (e.g., "AI", "SaaS"), a `niche` string, and `subreddits` (e.g., "technology+artificialintelligence").
3.  **Data Fetching:**
    *   Create a **Schedule Trigger** set to 2 hours.
    *   Create 3 **HTTP Request** nodes. 
        *   **X:** `GET https://api.twitter.com/2/trends/place?id=1`.
        *   **Reddit:** `GET https://oauth.reddit.com/r/{{subreddits}}/hot`.
        *   **Google:** `GET https://trends.google.com/trends/api/dailytrends`.
4.  **Normalization:** Connect each HTTP node to a **Code** node. Use JavaScript to map the unique API fields to a common schema: `{ topic, description, url, engagement, timestamp }`.
5.  **Logic Processing:**
    *   **Merge Node:** Use the "Combine" mode to bring all three paths together.
    *   **Filter Node:** Use JavaScript to iterate through the items and check if `topic` includes any keyword from the `Load Niche Config` node.
    *   **Deduplication Node:** Use a "Similarity" function in JavaScript to remove topics that have a >70% word overlap.
6.  **Scoring & Ranking:**
    *   **Code Node:** Implement the `Viral Score` logic (Weighted sum of engagement/1000 + recency).
    *   **Sort Node:** Sort by the new `viralScore` field descending.
7.  **AI Integration:**
    *   **HTTP Request (Anthropic):** URL: `https://api.anthropic.com/v1/messages`. Header: `x-api-key`. Body: Use a prompt asking Claude to return a JSON array of 5 content ideas based on the incoming trend topic.
8.  **Delivery:**
    *   **Aggregate Node:** Use a Code node to turn the array of individual opportunities into one large "Digest" object.
    *   **Email Node:** Use the `totalOpportunities` and the aggregated HTML list in the body.
9.  **Memory Management:** 
    *   In the final Code node, use `$getWorkflowStaticData('global')` to save the processed topics to the workflow's internal memory.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Claude Model Choice** | This workflow uses `claude-sonnet-4-20250514` for high-quality reasoning. |
| **API Rate Limits** | X (Twitter) API has strict limits; avoid running more frequently than every 15 minutes. |
| **Google Trends Parsing** | Note that Google's API response is not standard JSON; the code node must strip the first 6 characters before parsing. |
| **Static Data Storage** | Static data used for deduplication is only persisted on production executions, not during manual "Test" runs. |