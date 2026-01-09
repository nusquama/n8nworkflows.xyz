Market Intelligence Engine with AI Sentiment Detection & Competitor Analysis

https://n8nworkflows.xyz/workflows/market-intelligence-engine-with-ai-sentiment-detection---competitor-analysis-10765


# Market Intelligence Engine with AI Sentiment Detection & Competitor Analysis

### 1. Workflow Overview

This workflow, titled **"Market Intelligence Engine with AI Sentiment Detection & Competitor Analysis"**, is designed to automate the collection, normalization, analysis, and alerting of market intelligence data from a variety of sources. It targets competitive intelligence, technology trend tracking, and sentiment evaluation use cases in AI and related domains.

The workflow is organized into the following logical blocks:

- **1.1 Scheduled Data Collection:** Triggers periodic data fetching from multiple external APIs.
- **1.2 Data Aggregation and Normalization:** Merges data from diverse sources and normalizes them into a unified schema.
- **1.3 Content Deduplication and Enrichment:** Removes duplicate entries and enriches data with entity extraction and technical signal parsing.
- **1.4 Data Storage:** Persists raw enriched data into a database.
- **1.5 Analytical Processing:** Performs time-series analytics, topic modeling, sentiment analysis, changepoint detection, and novelty scoring on the collected data.
- **1.6 Signal Merging and Ranking:** Combines multiple analytics signals using multi-criteria decision making (MCDM) methods and ranks trends by impact and relevance.
- **1.7 Alerting and Reporting:** Checks alert thresholds, generates trend summaries using AI, and sends alerts via Slack, email, and workflow triggers.
- **1.8 Dashboard and KPI Tracking:** Stores trend ranking data and tracks KPIs for continuous monitoring.

Each block is implemented with a series of dedicated n8n nodes linked sequentially or in parallel, enabling modular processing and extensibility.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Collection

**Overview:**  
This block initiates the workflow execution every 6 hours to capture fresh market intelligence from multiple sources.

**Nodes Involved:**  
- Schedule Data Collection  
- Workflow Configuration

**Node Details:**

- **Schedule Data Collection**  
  - Type: Schedule Trigger  
  - Role: Triggers the entire workflow every 6 hours.  
  - Configuration: Interval set to 6 hours.  
  - Input: None (trigger node).  
  - Output: Outputs a trigger signal to "Workflow Configuration".  
  - Edge Cases: Workflow will not run if n8n instance is down or trigger is disabled.

- **Workflow Configuration**  
  - Type: Set  
  - Role: Defines and outputs configuration parameters used throughout the workflow.  
  - Configuration: Sets API URLs for news, blog, social media, academic papers, GitHub, forums, product docs; alert threshold; and database table names. Placeholder values must be replaced with actual URLs and table names.  
  - Key Variables: `newsApiUrl`, `blogApiUrl`, `socialApiUrl`, `academicApiUrl`, `githubApiUrl`, `forumApiUrl`, `docsApiUrl`, `alertThreshold`, `dbTable`, `dashboardTable`, `kpiTable`, `trainingTable`.  
  - Input: Trigger from Schedule Data Collection.  
  - Output: Configuration JSON used by subsequent fetch nodes.  
  - Edge Cases: Missing or incorrect URLs will cause HTTP request failures downstream.

---

#### 1.2 Data Aggregation and Normalization

**Overview:**  
Fetches content from multiple external data sources concurrently and merges the results into one dataset, then normalizes the data into a consistent schema.

**Nodes Involved:**  
- Fetch News Articles  
- Fetch Blog Posts  
- Fetch Social Media  
- Fetch Academic Papers  
- Fetch Code Repos  
- Fetch Forum Discussions  
- Fetch Product Docs  
- Merge All Sources  
- Normalize Content Schema

**Node Details:**

- **Fetch News Articles**  
  - Type: HTTP Request  
  - Role: Fetches news articles from configured API endpoint.  
  - Configuration: URL from `newsApiUrl` in Workflow Configuration; sends JSON header.  
  - Input: Workflow Configuration output.  
  - Output: Raw news data JSON.  
  - Edge Cases: API downtime, rate limits, malformed responses.

- **Fetch Blog Posts**  
  - Same as above for blogs, using `blogApiUrl`.

- **Fetch Social Media**  
  - Same as above for social media, using `socialApiUrl`.

- **Fetch Academic Papers**  
  - Same as above for academic papers, using `academicApiUrl`.

- **Fetch Code Repos**  
  - Same as above for code repositories, using `githubApiUrl`.

- **Fetch Forum Discussions**  
  - Same as above for forums, using `forumApiUrl`.

- **Fetch Product Docs**  
  - Same as above for product documentation, using `docsApiUrl`.

- **Merge All Sources**  
  - Type: Merge  
  - Role: Combines data from all seven fetch nodes into a single stream.  
  - Configuration: Number inputs set to 7 (one for each source).  
  - Input: Outputs from all fetch nodes.  
  - Output: Combined data items.  
  - Edge Cases: Unequal lengths of input arrays; missing data from some sources.

- **Normalize Content Schema**  
  - Type: Code  
  - Role: Transforms heterogeneous incoming data items into a unified schema with consistent fields such as id, title, content, url, source, publishedDate, author, tags, and metadata.  
  - Key Logic:  
    - Generates unique IDs if absent.  
    - Safely extracts nested fields with fallbacks.  
    - Normalizes date formats.  
    - Extracts tags from different possible fields.  
  - Input: Merged data from multiple sources.  
  - Output: Normalized items ready for deduplication.  
  - Edge Cases: Malformed or missing fields, date parsing errors.

---

#### 1.3 Content Deduplication and Enrichment

**Overview:**  
Removes duplicate content based on URL, extracts entities and topics using AI, and parses technical signals from text content.

**Nodes Involved:**  
- Deduplicate Content  
- Extract Entities & Topics  
- Parse Technical Signals

**Node Details:**

- **Deduplicate Content**  
  - Type: Remove Duplicates  
  - Role: Eliminates duplicate entries by comparing the `url` field.  
  - Configuration: Compare field set to `url`.  
  - Input: Normalized content items.  
  - Output: Unique content items.  
  - Edge Cases: Different URLs for identical content may evade deduplication.

- **Extract Entities & Topics**  
  - Type: OpenAI (LangChain)  
  - Role: Uses OpenAI API to extract named entities and topics from the content.  
  - Configuration: Operation set to "message" for chatbot-style prompt.  
  - Credentials: Requires valid OpenAI API key.  
  - Input: Deduplicated content.  
  - Output: Items enriched with extracted entities and topics.  
  - Edge Cases: API rate limits, ambiguous extraction, and misclassifications.

- **Parse Technical Signals**  
  - Type: Code  
  - Role: Parses AI-related technical signals such as AI models, datasets, benchmarks, libraries, and version numbers from content text using regex patterns.  
  - Input: Items with extracted entities and topics.  
  - Output: Enriched items with `technicalSignals` and signal counts.  
  - Edge Cases: False positives/negatives due to regex limitations, missing content fields.

---

#### 1.4 Data Storage

**Overview:**  
Stores the enriched raw data into a PostgreSQL database for persistence and historical analysis.

**Nodes Involved:**  
- Store Raw Data

**Node Details:**

- **Store Raw Data**  
  - Type: PostgreSQL  
  - Role: Inserts or updates normalized and enriched data into the specified table (`ai_trends` by default).  
  - Configuration: Table name from Workflow Configuration (`dbTable`), schema `"public"`.  
  - Input: Output from Parse Technical Signals.  
  - Output: Database operation result.  
  - Edge Cases: Database connection failures, schema mismatches, data size limits.

---

#### 1.5 Analytical Processing

**Overview:**  
Performs multiple analytic computations on stored data to derive insights on trends, sentiment, novelty, and change detection.

**Nodes Involved:**  
- Time-Series Analytics  
- Topic Modeling & Velocity  
- Sentiment Analysis  
- Changepoint Detection  
- Novelty Scoring

**Node Details:**

- **Time-Series Analytics**  
  - Type: Code  
  - Role: Aggregates mention frequencies over daily, weekly, and monthly windows for entities and topics; computes growth rates, moving averages, and trend classification (rising, declining, stable).  
  - Input: Stored raw data items.  
  - Output: Time-series analytics per entity/topic.  
  - Edge Cases: Sparse or irregular timestamp data affects accuracy.

- **Topic Modeling & Velocity**  
  - Type: Code  
  - Role: Performs TF-IDF based topic extraction with velocity and lifecycle stage analysis (emerging, peaking, declining, stable).  
  - Input: Items with content and timestamps.  
  - Output: Topic metrics including velocity, acceleration, lifecycle stage, and trend direction.  
  - Edge Cases: Noisy text or sparse data may skew topic modeling.

- **Sentiment Analysis**  
  - Type: Code  
  - Role: Lexicon-based sentiment scoring (-1 to 1) with intensifiers and negations; classifies sentiment as positive, neutral, or negative; calculates confidence and tracks trends.  
  - Input: Content items.  
  - Output: Sentiment scores and labels attached to items.  
  - Edge Cases: Limited lexicon coverage; sarcasm or complex sentiment may be misclassified.

- **Changepoint Detection**  
  - Type: Code  
  - Role: Detects sudden shifts in trends using CUSUM and Bayesian methods on time-series data; outputs changepoints and momentum status.  
  - Input: Time-series metrics (values array).  
  - Output: List of changepoints, statistics, and trend momentum.  
  - Edge Cases: Requires minimum 20 data points; insufficient data returns status message.

- **Novelty Scoring**  
  - Type: Code  
  - Role: Calculates how novel current items are versus historical baseline using Jaccard similarity on topics, entities, signals, and keywords; outputs composite novelty score and explanations.  
  - Input: Analytics outputs.  
  - Output: Novelty metrics and flags for new or unique content.  
  - Edge Cases: First item always marked as novel; small datasets limit baseline reliability.

---

#### 1.6 Signal Merging and Ranking

**Overview:**  
Combines different analytic signals into a composite impact score using MCDM (TOPSIS) and ranks trends by relevance and maturity.

**Nodes Involved:**  
- Merge Analytics Signals  
- MCDM Signal Fusion  
- Rank Trends by Impact

**Node Details:**

- **Merge Analytics Signals**  
  - Type: Merge  
  - Role: Combines time-series analytics, topic modeling, sentiment, changepoint detection, and novelty scoring outputs into unified items for signal fusion.  
  - Configuration: 5 inputs for the five analytics nodes.  
  - Input: Outputs from analytical processing nodes.  
  - Output: Merged signal data.  
  - Edge Cases: Mismatched item counts or missing signals.

- **MCDM Signal Fusion**  
  - Type: Code  
  - Role: Implements TOPSIS method to combine velocity, sentiment, changepoint magnitude, novelty, and time-series growth signals into a weighted impact score; assigns trend strength category.  
  - Input: Merged analytics signals.  
  - Output: Enriched data with composite MCDM scores and breakdowns.  
  - Edge Cases: Signal normalization depends on assumed ranges; abnormal values may skew results.

- **Rank Trends by Impact**  
  - Type: Code  
  - Role: Ranks trends by combined impact score, relevance to AI domain keywords, and maturity stage; assigns ranks and timestamps.  
  - Input: MCDM fusion output.  
  - Output: Ranked trends with detailed metrics.  
  - Edge Cases: Keyword-based relevance may miss emerging jargon; ranking logic may need tuning.

---

#### 1.7 Alerting and Reporting

**Overview:**  
Checks if trend impact scores exceed alert thresholds, generates AI-driven trend summaries, and sends notifications via Slack, email, and HTTP requests for downstream workflows.

**Nodes Involved:**  
- Store Trend Rankings  
- Check Alert Threshold  
- Generate Trend Summary  
- Send Slack Alert  
- Send Email Report  
- Trigger Workflow Actions

**Node Details:**

- **Store Trend Rankings**  
  - Type: PostgreSQL  
  - Role: Persists ranked trend data into a dedicated rankings table.  
  - Input: Ranked trends.  
  - Output: DB operation result.  
  - Edge Cases: DB connectivity and schema compliance.

- **Check Alert Threshold**  
  - Type: If  
  - Role: Filters trends with impact score greater than configured alert threshold.  
  - Input: Store Trend Rankings output.  
  - Output: Passes only high-impact trends downstream.  
  - Edge Cases: Threshold misconfiguration may suppress or flood alerts.

- **Generate Trend Summary**  
  - Type: OpenAI (LangChain)  
  - Role: Generates human-readable trend summaries using OpenAI models.  
  - Credentials: Requires OpenAI API key.  
  - Input: Filtered high-impact trends.  
  - Output: Text summaries.  
  - Edge Cases: API limits, prompt tuning required.

- **Send Slack Alert**  
  - Type: Slack  
  - Role: Sends alert messages to configured Slack channel via OAuth2 authentication.  
  - Configuration: Slack channel ID/name placeholder to be replaced.  
  - Input: Generated summaries.  
  - Output: Slack message delivery confirmation.  
  - Edge Cases: Slack API limits, authentication expiry.

- **Send Email Report**  
  - Type: Gmail  
  - Role: Emails detailed trend reports to specified recipients using Gmail OAuth2.  
  - Configuration: Recipient email placeholder to be set.  
  - Input: Generated summaries.  
  - Output: Email send status.  
  - Edge Cases: Gmail API restrictions, authentication expiry.

- **Trigger Workflow Actions**  
  - Type: HTTP Request  
  - Role: Sends JSON payload to external API endpoint to trigger additional workflows or actions.  
  - Configuration: API endpoint URL placeholder.  
  - Input: Summary JSON.  
  - Output: HTTP response.  
  - Edge Cases: Endpoint availability, authentication, request failures.

---

#### 1.8 Dashboard and KPI Tracking

**Overview:**  
Stores data for dashboards and tracks key performance indicators (KPIs) related to trends and alert accuracy.

**Nodes Involved:**  
- Store Dashboard Data  
- Track KPIs

**Node Details:**

- **Store Dashboard Data**  
  - Type: PostgreSQL  
  - Role: Saves dashboard visualization data, timelines, leaderboards, and configuration for UI consumption.  
  - Configuration: Table name from `dashboardTable` config variable; schema "public".  
  - Input: Output from trend ranking and analytics.  
  - Output: DB operation result.  
  - Edge Cases: Schema changes may require adjustments.

- **Track KPIs**  
  - Type: PostgreSQL  
  - Role: Inserts KPI metrics such as recall, precision, alert accuracy, lead time, latency, false positive rate, and downstream impact for ongoing performance evaluation.  
  - Configuration: Table name from `kpiTable`.  
  - Input: KPI data generated downstream.  
  - Output: DB operation result.  
  - Edge Cases: Data quality and consistency required for meaningful KPIs.

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                                  | Input Node(s)                          | Output Node(s)                             | Sticky Note                                                                                                    |
|--------------------------|---------------------------|-------------------------------------------------|--------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Data Collection  | Schedule Trigger          | Triggers workflow every 6 hours                  | None                                 | Workflow Configuration                     | Schedule Daily Collection: Triggers workflow to fetch news, blogs, social media, academic papers, code, docs. Why: Daily cadence captures emerging trends before competitors react |
| Workflow Configuration    | Set                       | Sets API URLs, alert threshold, DB table names  | Schedule Data Collection             | Fetch News Articles, Fetch Blog Posts, Fetch Social Media, Fetch Academic Papers, Fetch Code Repos, Fetch Forum Discussions, Fetch Product Docs | Setup Steps: Connect data sources, configure AI, dashboards, storage. |
| Fetch News Articles       | HTTP Request              | Fetches news data                                | Workflow Configuration              | Merge All Sources                          | Fetch News & Blog Content: Retrieves articles from tech news sites, industry publications. Why: News breaks market shifts earliest |
| Fetch Blog Posts          | HTTP Request              | Fetches blog posts                               | Workflow Configuration              | Merge All Sources                          | (Same as above)                                                                                               |
| Fetch Social Media        | HTTP Request              | Fetches social media posts                        | Workflow Configuration              | Merge All Sources                          | (Same as above)                                                                                               |
| Fetch Academic Papers     | HTTP Request              | Fetches academic papers                           | Workflow Configuration              | Merge All Sources                          | (Same as above)                                                                                               |
| Fetch Code Repos          | HTTP Request              | Fetches code repository data                      | Workflow Configuration              | Merge All Sources                          | Aggregate Market Discussions: Collects earnings calls, investor presentations. Why: Direct stakeholder commentary reveals strategy and positioning |
| Fetch Forum Discussions   | HTTP Request              | Fetches forum discussions                         | Workflow Configuration              | Merge All Sources                          | (Same as above)                                                                                               |
| Fetch Product Docs        | HTTP Request              | Fetches product documentation                     | Workflow Configuration              | Merge All Sources                          | (Same as above)                                                                                               |
| Merge All Sources         | Merge                     | Combines all fetched data streams                 | Fetch News Articles, Blog, Social, Academic, Code, Forum, Docs | Normalize Content Schema                   | Extract Content & Parse Structure, Storage: Converts multi-format data into machine-readable text with metadata. Why: Unified parsing enables cross-source analysis; surfaces patterns |
| Normalize Content Schema  | Code                      | Normalizes all data into unified schema           | Merge All Sources                   | Deduplicate Content                        | Normalize All Formats and Duplicate: Standardizes timestamps, authorship, classification. Why: Enables comparison |
| Deduplicate Content       | Remove Duplicates         | Removes duplicate items by URL                     | Normalize Content Schema            | Extract Entities & Topics                  | (Same as above)                                                                                               |
| Extract Entities & Topics | OpenAI (LangChain)        | Extracts named entities and topics                | Deduplicate Content                | Parse Technical Signals                    | (Same as above)                                                                                               |
| Parse Technical Signals   | Code                      | Extracts AI model names, datasets, benchmarks etc.| Extract Entities & Topics          | Store Raw Data                            | (Same as above)                                                                                               |
| Store Raw Data            | PostgreSQL                | Stores enriched raw data                           | Parse Technical Signals            | Time-Series Analytics                      | (Same as above)                                                                                               |
| Time-Series Analytics     | Code                      | Analyzes mention frequency, growth rates, trends | Store Raw Data                     | Merge Analytics Signals                    | Sentiment Analysis Across Sources: Scores tone for each piece. Why: Sentiment reveals market mood               |
| Topic Modeling & Velocity | Code                      | TF-IDF topic extraction and lifecycle velocity    | Store Raw Data                     | Merge Analytics Signals                    | (Same as above)                                                                                               |
| Sentiment Analysis        | Code                      | Lexicon-based sentiment scoring                    | Store Raw Data                     | Merge Analytics Signals                    | (Same as above)                                                                                               |
| Changepoint Detection     | Code                      | Detects trend shift points in time-series          | Store Raw Data                     | Merge Analytics Signals                    | (Same as above)                                                                                               |
| Novelty Scoring           | Code                      | Calculates content novelty vs historical baseline | Store Raw Data                     | Merge Analytics Signals                    | (Same as above)                                                                                               |
| Merge Analytics Signals   | Merge                     | Combines multiple analytics outputs                 | Time-Series Analytics, Topic Modeling, Sentiment Analysis, Changepoint Detection, Novelty Scoring | MCDM Signal Fusion                        | Merge Datasets for Analysis: Combines sources into searchable repository. Why: Reveals connections             |
| MCDM Signal Fusion        | Code                      | Combines signals using TOPSIS method                | Merge Analytics Signals           | Rank Trends by Impact                      | MCDN Signal Processing: Ingests market signals, ranks by impact, stores snapshots. Why: Consistent pipeline    |
| Rank Trends by Impact     | Code                      | Ranks trends by combined impact and relevance       | MCDM Signal Fusion               | Store Trend Rankings                       | (Same as above)                                                                                               |
| Store Trend Rankings      | PostgreSQL                | Persists ranked trends                              | Rank Trends by Impact             | Check Alert Threshold, Store Dashboard Data, Track KPIs | Alert Evaluation: Checks signals against thresholds, generates summaries. Why: Ensures meaningful alerts only  |
| Check Alert Threshold     | If                        | Filters trends exceeding alert threshold            | Store Trend Rankings              | Generate Trend Summary                     | (Same as above)                                                                                               |
| Generate Trend Summary    | OpenAI (LangChain)        | Creates AI-generated trend summaries                 | Check Alert Threshold             | Send Slack Alert, Send Email Report, Trigger Workflow Actions | Trend Summary: Monitors KPIs and competitive metrics. Why: KPI tracking reveals trends                         |
| Send Slack Alert          | Slack                     | Sends alert message to Slack channel                 | Generate Trend Summary            | None                                       | Output & Alerts: Sends critical signals via Slack, emails, triggers. Why: Fast visibility, detailed context   |
| Send Email Report         | Gmail                     | Sends email report for trends                         | Generate Trend Summary            | None                                       | (Same as above)                                                                                               |
| Trigger Workflow Actions  | HTTP Request              | Sends data to external API to trigger workflows       | Generate Trend Summary            | None                                       | (Same as above)                                                                                               |
| Store Dashboard Data      | PostgreSQL                | Stores dashboard data and visualizations              | Store Trend Rankings              | None                                       | (Same as above)                                                                                               |
| Track KPIs                | PostgreSQL                | Tracks KPIs related to trends and alerts               | Store Trend Rankings              | None                                       | (Same as above)                                                                                               |
| Sticky Note               | Sticky Note               | Provides descriptive notes about workflow structure  | None                             | None                                       | How It Works: Scheduled aggregation from 8 sources; AI sentiment scoring; MCDN routing; dashboards; KPI tracking; reports. |
| Sticky Note3              | Sticky Note               | Notes setup steps                                      | None                             | None                                       | Setup Steps: Connect data sources, AI, dashboards, storage                                                   |
| Sticky Note4              | Sticky Note               | Notes customization and benefits                       | None                             | None                                       | Customization: Adjust thresholds, sources, AI models; Benefits: reduces research time, consolidates intelligence |
| Sticky Note5              | Sticky Note               | Prerequisites and use cases                             | None                             | None                                       | Prerequisites: API keys, dashboard access, email, repos; Use Cases: competitive intelligence, market trends, misinformation filtering |
| Sticky Note1              | Sticky Note               | Notes schedule trigger purpose                         | None                             | None                                       | Schedule Daily Collection: Daily triggers capture emerging trends early                                      |
| Sticky Note2              | Sticky Note               | Notes on news & blog content fetching                   | None                             | None                                       | Fetch News & Blog Content: Early market shifts detection                                                    |
| Sticky Note6              | Sticky Note               | Notes on aggregating market discussions                 | None                             | None                                       | Aggregate Market Discussions: Earnings calls, presentations reveal strategy                                  |
| Sticky Note7              | Sticky Note               | Notes on content parsing and storage                     | None                             | None                                       | Extract Content & Parse Structure: Converts diverse formats into machine-readable text                       |
| Sticky Note8              | Sticky Note               | Notes on normalization and deduplication                 | None                             | None                                       | Normalize All Formats and Duplicate: Standardizes and removes duplicates                                    |
| Sticky Note9              | Sticky Note               | Notes on sentiment analysis                               | None                             | None                                       | Sentiment Analysis Across Sources: Scores tone and urgency                                                 |
| Sticky Note10             | Sticky Note               | Notes on merging datasets                                 | None                             | None                                       | Merge Datasets for Analysis: Reveals cross-source connections                                              |
| Sticky Note11             | Sticky Note               | Notes on MCDN signal processing                           | None                             | None                                       | MCDN Signal Processing: Ranks market signals by impact                                                     |
| Sticky Note12             | Sticky Note               | Notes on alert evaluation                                 | None                             | None                                       | Alert Evaluation: Filters for meaningful alerts only                                                       |
| Sticky Note13             | Sticky Note               | Notes on trend monitoring                                 | None                             | None                                       | Trend Summary: Monitors KPIs and competitive metrics                                                      |
| Sticky Note14             | Sticky Note               | Notes on outputs and alerts                               | None                             | None                                       | Output & Alerts: Sends alerts via Slack, email, and triggers workflows                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node ("Schedule Data Collection")**  
   - Type: Schedule Trigger  
   - Set interval to every 6 hours.

2. **Add a Set Node ("Workflow Configuration")**  
   - Define string parameters for API URLs: `newsApiUrl`, `blogApiUrl`, `socialApiUrl`, `academicApiUrl`, `githubApiUrl`, `forumApiUrl`, `docsApiUrl` with placeholder values.  
   - Define number parameter `alertThreshold` default 0.75.  
   - Define string parameters for DB table names: `dbTable` (default "ai_trends"), `dashboardTable`, `kpiTable`, `trainingTable`.  
   - Connect from Schedule Data Collection trigger.

3. **Create HTTP Request Nodes for Each Data Source:**  
   - For each of News Articles, Blog Posts, Social Media, Academic Papers, Code Repos, Forum Discussions, Product Docs:  
     - Type: HTTP Request  
     - URL: Use expression to get corresponding URL from Workflow Configuration (e.g., `={{ $('Workflow Configuration').first().json.newsApiUrl }}`)  
     - Set header `Content-Type: application/json`.  
     - Connect all from Workflow Configuration node.

4. **Add a Merge Node ("Merge All Sources")**  
   - Configure to merge 7 inputs (one from each fetch node).  
   - Connect all fetch nodes to this merge node.

5. **Add a Code Node ("Normalize Content Schema")**  
   - Paste the provided JavaScript code that normalizes incoming data into a consistent schema (id, title, content, url, source, publishedDate, author, tags, metadata).  
   - Set mode to runOnceForEachItem.  
   - Connect from Merge All Sources.

6. **Add a Remove Duplicates Node ("Deduplicate Content")**  
   - Configure to compare by `url` field.  
   - Connect from Normalize Content Schema.

7. **Add OpenAI Node ("Extract Entities & Topics")**  
   - Use OpenAI credentials.  
   - Set operation to "message" for entity and topic extraction.  
   - Connect from Deduplicate Content.

8. **Add a Code Node ("Parse Technical Signals")**  
   - Paste provided JavaScript code that extracts AI models, datasets, benchmarks, libraries, and versions from content text.  
   - Set mode to runOnceForEachItem.  
   - Connect from Extract Entities & Topics.

9. **Add a PostgreSQL Node ("Store Raw Data")**  
   - Configure connection to PostgreSQL database.  
   - Table: Use expression from Workflow Configuration `dbTable`.  
   - Map all input fields.  
   - Connect from Parse Technical Signals.

10. **Add Analytical Processing Nodes:**  
    - Time-Series Analytics (Code node with provided JS). Connect from Store Raw Data.  
    - Topic Modeling & Velocity (Code node). Connect from Store Raw Data.  
    - Sentiment Analysis (Code node). Connect from Store Raw Data.  
    - Changepoint Detection (Code node). Connect from Store Raw Data.  
    - Novelty Scoring (Code node). Connect from Store Raw Data.

11. **Add a Merge Node ("Merge Analytics Signals")**  
    - Number of inputs: 5 (one from each analytic node above).  
    - Connect all analytic nodes to this merge node.

12. **Add a Code Node ("MCDM Signal Fusion")**  
    - Paste provided TOPSIS method JS code for multi-criteria decision making.  
    - Set mode to runOnceForEachItem.  
    - Connect from Merge Analytics Signals.

13. **Add a Code Node ("Rank Trends by Impact")**  
    - Paste ranking JS code that scores trends by impact, relevance, and maturity.  
    - Connect from MCDM Signal Fusion.

14. **Add PostgreSQL Node ("Store Trend Rankings")**  
    - Configure to write to a rankings table (e.g., `${dbTable}_rankings`).  
    - Map relevant ranking fields.  
    - Connect from Rank Trends by Impact.

15. **Add an If Node ("Check Alert Threshold")**  
    - Condition: `impactScore` > `alertThreshold` from configuration.  
    - Connect from Store Trend Rankings.

16. **Add OpenAI Node ("Generate Trend Summary")**  
    - Use OpenAI credentials.  
    - Operation: "message" to generate human-readable summaries.  
    - Connect from If Node (true branch).

17. **Add Alert Nodes:**  
    - Slack Node ("Send Slack Alert"): Configure with OAuth2 and channel ID. Connect from Generate Trend Summary.  
    - Gmail Node ("Send Email Report"): Configure with OAuth2 and recipient address. Connect from Generate Trend Summary.  
    - HTTP Request Node ("Trigger Workflow Actions"): Configure URL endpoint. Connect from Generate Trend Summary.

18. **Add PostgreSQL Node ("Store Dashboard Data")**  
    - Configure to save dashboard visualization data.  
    - Connect from Store Trend Rankings.

19. **Add PostgreSQL Node ("Track KPIs")**  
    - Configure to save KPI metrics table.  
    - Connect from Store Trend Rankings.

20. **Add Sticky Note nodes** at appropriate positions with the provided content for documentation and guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| How It Works: Scheduled aggregation from 8 sources; AI sentiment scoring; MCDN routing; dashboards; KPI tracking; reports. | Workflow overview sticky note describing main functional flow.                                                      |
| Setup Steps: Connect data sources, configure AI, dashboards, storage                                            | Setup instructions for initial configuration and credential setup.                                                  |
| Customization: Adjust thresholds, sources, AI models; Benefits: reduces research time, consolidates intelligence | Guidance for adapting workflow to specific needs.                                                                    |
| Prerequisites: API keys, dashboard access, email, repos; Use Cases: competitive intelligence, market trends, misinformation filtering | Required credentials and typical use cases.                                                                           |
| Schedule Daily Collection: Daily triggers capture emerging trends early                                         | Explains rationale for the 6-hour data fetch schedule.                                                              |
| Fetch News & Blog Content: Early market shifts detection                                                       | Highlights importance of news and blog data for early signals.                                                      |
| Aggregate Market Discussions: Earnings calls, presentations reveal strategy                                    | Emphasizes the value of direct stakeholder commentary.                                                               |
| Extract Content & Parse Structure: Converts diverse formats into machine-readable text                         | Importance of unified parsing for cross-source analysis.                                                            |
| Normalize All Formats and Duplicate: Standardizes and removes duplicates                                      | Enables consistent data comparison.                                                                                   |
| Sentiment Analysis Across Sources: Scores tone and urgency                                                    | Importance of multi-source sentiment evaluation.                                                                     |
| Merge Datasets for Analysis: Reveals cross-source connections                                                 | Benefits of combining datasets for richer insights.                                                                  |
| MCDN Signal Processing: Ranks market signals by impact                                                        | Ensures consistent prioritization of signals.                                                                         |
| Alert Evaluation: Filters for meaningful alerts only                                                          | Prevents alert fatigue by thresholding.                                                                                |
| Trend Summary: Monitors KPIs and competitive metrics                                                          | Continuous monitoring for trend validation.                                                                            |
| Output & Alerts: Sends alerts via Slack, email, and triggers workflows                                        | Facilitates timely and detailed communication of critical insights.                                                  |

---

This documentation provides a comprehensive, detailed reference for understanding, reproducing, maintaining, and extending the "Market Intelligence Engine with AI Sentiment Detection & Competitor Analysis" workflow in n8n.