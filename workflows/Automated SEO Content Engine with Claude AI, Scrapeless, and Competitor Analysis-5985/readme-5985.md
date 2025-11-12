Automated SEO Content Engine with Claude AI, Scrapeless, and Competitor Analysis

https://n8nworkflows.xyz/workflows/automated-seo-content-engine-with-claude-ai--scrapeless--and-competitor-analysis-5985


# Automated SEO Content Engine with Claude AI, Scrapeless, and Competitor Analysis

### 1. Workflow Overview

This workflow, titled **"Automated SEO Content Engine with Claude AI, Scrapeless, and Competitor Analysis"**, is designed to automate the end-to-end process of generating SEO-optimized content for a website. It focuses on leveraging trending keywords, analyzing their market interest, researching competitive content, and finally creating complete SEO articles with AI assistance. The workflow targets digital marketers, SEO specialists, and content strategists aiming to efficiently produce high-quality content based on data-driven insights.

The workflow is logically divided into three main phases:

- **1.1 Phase 1: Hot Topics Identification**  
  Starting from a seed keyword, it gathers related trending keywords using Google Trends data and evaluates their trend status and priority for content creation.

- **1.2 Phase 2: Competitive Content Research**  
  For prioritized trending keywords, the workflow performs competitor analysis by crawling top competitor URLs, extracting their content, and aggregating this data.

- **1.3 Phase 3: SEO Article Generation and Storage**  
  Using the competitive research and trend insights, an AI agent generates a complete SEO-optimized article, which is then parsed, cleaned, and stored in a database.

Each phase includes a set of nodes specialized for the tasks of data fetching, processing, filtering, AI-driven analysis, and final content creation.

---

### 2. Block-by-Block Analysis

#### 2.1 Phase 1: Hot Topics Identification

- **Overview:**  
  This block initiates the workflow with a seed keyword, fetches related trending keywords using Google Trends via Scrapeless API, and analyzes the trend data to prioritize keywords based on their current market interest.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Set seed keywords (Set)  
  - Google Trends (Scrapeless)  
  - Split Out (Split Out)  
  - Google Trends-Get heat data (Scrapeless)  
  - AI Agent (Langchain Agent)  
  - Structured Output Parser (Langchain Output Parser)  
  - Code1 (Code)  
  - Append or update row in sheet (Google Sheets)  
  - Get row(s) in sheet (Google Sheets)  
  - Split Out1 (Split Out)  
  - Filter out topics with priority above P2 (Filter)

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point for manual workflow execution.  
     - Input/Output: Starts the data flow.  
     - Edge cases: None specific.

  2. **Set seed keywords**  
     - Type: Set  
     - Role: Defines the initial seed keyword, e.g., "Project Management".  
     - Configuration: Assigns a string variable `seedKeyword`.  
     - Input: Manual Trigger  
     - Output: Passes seedKeyword downstream.

  3. **Google Trends**  
     - Type: Scrapeless (Google Trends operation)  
     - Role: Fetches related queries for the seed keyword from Google Trends with US geography and category 0 (default).  
     - Key Expression: Query parameter uses `{{$json.seedKeyword}}`.  
     - Output: JSON with related queries data.  
     - Edge cases: API rate limits, invalid keyword, network errors.

  4. **Split Out**  
     - Type: Split Out  
     - Role: Extracts the array `related_queries.top` from the Google Trends output into individual items for separate processing.  
     - Input: Google Trends node  
     - Output: Each related query as separate item.

  5. **Google Trends-Get heat data**  
     - Type: Scrapeless (Google Trends operation)  
     - Role: For each related keyword, fetch detailed interest over time (heat data) from Google Trends.  
     - Key Expression: Uses `{{$json.query}}` from split related queries.  
     - Edge cases: Empty or malformed query strings.

  6. **AI Agent**  
     - Type: Langchain Agent with Anthropic Claude Sonnet 4 model  
     - Role: Analyze the Google Trends time-series data to interpret trend status and recommend content creation priority (P0 to P3).  
     - Configuration:  
       - System message describes role as SEO content strategist interpreting time series data.  
       - Input message includes keyword and raw trends JSON data.  
       - Output is a strict JSON object with fields: data_interpretation, trends_status, recommended_priority, keyword.  
     - Edge cases: AI output formatting errors, API timeouts, rate limits.

  7. **Structured Output Parser**  
     - Type: Langchain Structured Output Parser  
     - Role: Parses the AI Agent’s raw response into structured JSON based on a schema example.  
     - Edge cases: Parsing failures if AI output deviates from expected format.

  8. **Code1**  
     - Type: Code (JavaScript)  
     - Role: Sorts the AI Agent results by priority level (P0 highest to P3 lowest) to organize downstream processing.  
     - Logic: Categorizes items into arrays by priority and concatenates them in priority order.  
     - Edge cases: Missing or malformed priority fields.

  9. **Append or update row in sheet**  
     - Type: Google Sheets  
     - Role: Stores the analyzed trending keyword data with priority and interpretation into a Google Sheets document ("Topics" spreadsheet).  
     - Columns mapped: Level (priority), Reason (interpretation), Seed Keywords, Related Keywords.  
     - Credentials: Google Sheets OAuth2.  
     - Edge cases: API quota, credential expiration.

  10. **Get row(s) in sheet**  
      - Type: Google Sheets  
      - Role: Retrieves current rows from the "Topics" sheet for further filtering.  
      - Edge cases: Empty sheet, read permissions.

  11. **Split Out1**  
      - Type: Split Out  
      - Role: Extracts the "Level" field from each sheet row for filtering.

  12. **Filter out topics with priority above P2**  
      - Type: Filter  
      - Role: Filters out keywords with priority level P3 (lowest priority), allowing only P0, P1, P2 to continue to the next phase.  
      - Logic: Checks if Level field does not contain "P3".

---

#### 2.2 Phase 2: Competitive Content Research

- **Overview:**  
  This block collects competitive intelligence by searching Google for related keywords, selecting top competitor URLs, crawling their content, and aggregating it for content generation.

- **Nodes Involved:**  
  - Google Search (Scrapeless)  
  - Filter TOP3 competitor links (Set)  
  - Split Out2 (Split Out)  
  - Crawl (Scrapeless crawler)  
  - Extract competitor content Markdown (Set)  
  - Aggregate (Aggregate)

- **Node Details:**

  1. **Google Search**  
     - Type: Scrapeless (Google Search operation)  
     - Role: Searches Google for the filtered related keyword from previous phase to find competitor links.  
     - Key Expression: Query uses `{{$json['Related Keywords']}}`.  
     - Edge cases: Empty query, API limits.

  2. **Filter TOP3 competitor links**  
     - Type: Set  
     - Role: Truncates the organic search results to top 3 competitor URLs for focused crawling.  
     - Logic: Slices `organic_results` array to first three entries.  
     - Edge cases: Less than 3 results available.

  3. **Split Out2**  
     - Type: Split Out  
     - Role: Splits the top 3 competitor URLs for individual crawling.

  4. **Crawl**  
     - Type: Scrapeless crawler  
     - Role: Crawls each competitor URL to extract page content for analysis.  
     - Key Expression: URL taken from `{{$json.link}}`.  
     - Edge cases: Dead links, crawl restrictions, timeouts.

  5. **Extract competitor content Markdown**  
     - Type: Set  
     - Role: Extracts the Markdown content field from the crawl output for further processing.  
     - Assigns field `markdown` = `{{$json[0].markdown}}`.  
     - Edge cases: Missing content, empty crawl result.

  6. **Aggregate**  
     - Type: Aggregate  
     - Role: Aggregates all competitor Markdown content into a single array to pass to the content writer AI.  
     - Aggregates the `markdown` field from each item.  
     - Edge cases: No input items causes empty aggregation.

---

#### 2.3 Phase 3: SEO Article Generation and Storage

- **Overview:**  
  The final phase uses an AI agent to write a complete SEO article by synthesizing trend insights and competitor content. The article output is parsed, cleaned, and stored in a Supabase database.

- **Nodes Involved:**  
  - Senior SEO content writer (Langchain Agent)  
  - Code (Code)  
  - Create a row (Supabase)  
  - Anthropic Chat Model1 (Langchain LM Chat Anthropic)  

- **Node Details:**

  1. **Senior SEO content writer**  
     - Type: Langchain Agent using Anthropic Claude Sonnet 4 model  
     - Role: Generates a full SEO-optimized article based on:  
       - Target keyword (from competitor markdown)  
       - SaaS product context (hardcoded as “SaaS Product”)  
       - Trend insights (passed as markdown text)  
       - Competitor content (top 3 competitor markdowns)  
     - Output: Strict JSON with fields for title, slug, meta_description, strategy_summary (key trend insight and content angle), and article_body (structured array of H2 and H3 sections).  
     - Edge cases: AI output format errors, rate limits, long response truncation.

  2. **Code**  
     - Type: Code (JavaScript)  
     - Role: Cleans and parses the AI-generated JSON string from the Senior SEO content writer node. It:  
       - Extracts valid JSON from AI output (usually wrapped in code blocks).  
       - Parses JSON and converts complex fields to strings for database compatibility.  
       - Throws errors if JSON parsing fails, logging raw AI output for debugging.  
     - Edge cases: Malformed AI output, JSON parse errors.

  3. **Create a row**  
     - Type: Supabase (Database node)  
     - Role: Stores the cleaned article data into a Supabase table named `seo_articles`.  
     - Fields stored: title, meta_description, body (article JSON string), slug.  
     - Credentials: Supabase API credentials required.  
     - Edge cases: Database connectivity, permission errors.

  4. **Anthropic Chat Model1**  
     - Type: Langchain LM Chat Anthropic (Claude Sonnet 4)  
     - Role: Underlying AI language model used by the Senior SEO content writer node (invoked as sub-node).  
     - Credentials: Anthropic API key required.

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                                  | Input Node(s)                         | Output Node(s)                   | Sticky Note                                                                                       |
|----------------------------------|----------------------------------|-------------------------------------------------|-------------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                   | Entry point to start workflow                    | -                                   | Set seed keywords, Get row(s)    | # Phase 1: Hot Topics                                                                            |
| Set seed keywords                | Set                              | Defines initial seed keyword                      | Manual Trigger                      | Google Trends                   | # Phase 1: Hot Topics                                                                            |
| Google Trends                   | Scrapeless                       | Fetch related Google Trends queries               | Set seed keywords                   | Split Out                      | # Phase 1: Hot Topics                                                                            |
| Split Out                      | Split Out                       | Split related queries into separate items        | Google Trends                      | Google Trends-Get heat data      | # Phase 1: Hot Topics                                                                            |
| Google Trends-Get heat data     | Scrapeless                       | Fetch time-series heat data for keywords          | Split Out                         | AI Agent                       | # Phase 1: Hot Topics                                                                            |
| AI Agent                       | Langchain Agent                 | Analyze trends data to assign priority            | Google Trends-Get heat data         | Code1, Structured Output Parser  | # Phase 1: Hot Topics                                                                            |
| Structured Output Parser        | Langchain Output Parser         | Parse AI Agent output into structured JSON        | AI Agent                          | AI Agent                       | # Phase 1: Hot Topics                                                                            |
| Code1                         | Code                            | Sorts keywords by priority level                   | AI Agent                          | Append or update row in sheet    | # Phase 1: Hot Topics                                                                            |
| Append or update row in sheet   | Google Sheets                   | Store keyword trends and priority to sheet        | Code1                            | -                              | # Phase 1: Hot Topics                                                                            |
| Get row(s) in sheet             | Google Sheets                   | Retrieve existing data rows for filtering          | Manual Trigger                    | Split Out1                    | # Phase 1: Hot Topics                                                                            |
| Split Out1                    | Split Out                       | Extract Level field for filtering                   | Get row(s) in sheet               | Filter out topics with priority above P2 | # Phase 1: Hot Topics                                                                            |
| Filter out topics with priority above P2 | Filter                      | Filter out low priority topics (P3)                | Split Out1                      | Google Search                   | # Phase 2: Competitive content research                                                        |
| Google Search                 | Scrapeless                       | Search Google for related keywords                  | Filter out topics with priority above P2 | Filter TOP3 competitor links    | # Phase 2: Competitive content research                                                        |
| Filter TOP3 competitor links    | Set                             | Select top 3 competitor links                        | Google Search                    | Split Out2                     | # Phase 2: Competitive content research                                                        |
| Split Out2                   | Split Out                       | Split competitor links for crawling                  | Filter TOP3 competitor links      | Crawl                         | # Phase 2: Competitive content research                                                        |
| Crawl                        | Scrapeless crawler              | Crawl competitor URLs for content extraction         | Split Out2                      | Extract competitor content Markdown | # Phase 2: Competitive content research                                                        |
| Extract competitor content Markdown | Set                             | Extract Markdown content from crawled pages          | Crawl                         | Aggregate                      | # Phase 2: Competitive content research                                                        |
| Aggregate                    | Aggregate                      | Aggregate competitor Markdown content                | Extract competitor content Markdown | Senior SEO content writer      | # Phase 3: Complete SEO article writing and store it in the database                            |
| Senior SEO content writer    | Langchain Agent                 | Generate full SEO article based on trend and competitor data | Aggregate                       | Code                          | # Phase 3: Complete SEO article writing and store it in the database                            |
| Code                         | Code                            | Parse and clean AI-generated article JSON            | Senior SEO content writer       | Create a row                   | # Phase 3: Complete SEO article writing and store it in the database                            |
| Create a row                  | Supabase                       | Save generated article to Supabase database           | Code                          | -                             | # Phase 3: Complete SEO article writing and store it in the database                            |
| Anthropic Chat Model           | Langchain LM Chat Anthropic    | AI model used by AI Agent                             | -                               | AI Agent                      |                                                                                                 |
| Anthropic Chat Model1          | Langchain LM Chat Anthropic    | AI model used by Senior SEO content writer            | -                               | Senior SEO content writer     |                                                                                                 |
| Sticky Note                   | Sticky Note                    | Label for Phase 1                                       | -                               | -                             | # Phase 1: Hot Topics                                                                            |
| Sticky Note1                  | Sticky Note                    | Label for Phase 2                                       | -                               | -                             | # Phase 2: Competitive content research                                                        |
| Sticky Note2                  | Sticky Note                    | Label for Phase 3                                       | -                               | -                             | # Phase 3: Complete SEO article writing and store it in the database                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a `Manual Trigger` node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - No special configuration.

2. **Add a `Set` node:**  
   - Name: `Set seed keywords`  
   - Set a string variable `seedKeyword` with default value `"Project Management"`.  
   - Connect from Manual Trigger.

3. **Add a `Scrapeless` node for Google Trends:**  
   - Name: `Google Trends`  
   - Operation: `googleTrends`  
   - Parameters:  
     - q: `={{ $json.seedKeyword }}`  
     - cat: `0`  
     - geo: `US`  
     - data_type: `related_queries`  
   - Credentials: Scrapeless API required.  
   - Connect from Set seed keywords.

4. **Add a `Split Out` node:**  
   - Name: `Split Out`  
   - Field to split out: `related_queries.top`  
   - Connect from Google Trends.

5. **Add a second `Scrapeless` node for detailed Google Trends data:**  
   - Name: `Google Trends-Get heat data`  
   - Operation: `googleTrends`  
   - Parameters:  
     - q: `={{ $json.query }}`  
     - cat: `0`  
     - geo: `US`  
   - Credentials: Scrapeless API.  
   - Connect from Split Out.

6. **Add a `Langchain Agent` node:**  
   - Name: `AI Agent`  
   - Model: Anthropic Claude Sonnet 4 (configured in Anthropic Chat Model sub-node)  
   - System message: SEO content strategist interpretation of time series data, with instructions to output JSON with fields: data_interpretation, trends_status, recommended_priority, keyword.  
   - Prompt includes keyword and raw trends JSON.  
   - Connect from Google Trends-Get heat data.

7. **Add a `Structured Output Parser` node:**  
   - Name: `Structured Output Parser`  
   - JSON schema example provided to parse AI output.  
   - Connect output `ai_outputParser` of this node to AI Agent node input `ai_languageModel`.

8. **Add a `Code` node:**  
   - Name: `Code1`  
   - JavaScript code to sort AI Agent outputs by priority P0 to P3.  
   - Connect from AI Agent main output.

9. **Add a `Google Sheets` node:**  
   - Name: `Append or update row in sheet`  
   - Operation: `appendOrUpdate`  
   - Document ID and Sheet Name set to your Google Sheet for topics.  
   - Map columns: `Level`, `Reason`, `Seed Keywords`, `Related Keywords`.  
   - Credentials: Google Sheets OAuth2.  
   - Connect from Code1.

10. **Add another `Google Sheets` node:**  
    - Name: `Get row(s) in sheet`  
    - Operation: `read` entire sheet.  
    - Credentials: Google Sheets OAuth2 (can be different account).  
    - Connect from Manual Trigger (parallel to Set seed keywords).

11. **Add a `Split Out` node:**  
    - Name: `Split Out1`  
    - Field to split out: `Level`  
    - Connect from Get row(s) in sheet.

12. **Add a `Filter` node:**  
    - Name: `Filter out topics with priority above P2`  
    - Condition: Exclude rows where Level is `"P3"` (lowest priority).  
    - Connect from Split Out1.

13. **Add a `Scrapeless` node for Google Search:**  
    - Name: `Google Search`  
    - Operation: Search Google with query `={{ $json['Related Keywords'] }}`  
    - Credentials: Scrapeless API.  
    - Connect from Filter node.

14. **Add a `Set` node:**  
    - Name: `Filter TOP3 competitor links`  
    - Assign `organic_results` = first 3 entries of `organic_results` array.  
    - Connect from Google Search.

15. **Add a `Split Out` node:**  
    - Name: `Split Out2`  
    - Field to split out: `organic_results`  
    - Connect from Filter TOP3 competitor links.

16. **Add a `Scrapeless` crawler node:**  
    - Name: `Crawl`  
    - Operation: `crawl` URL from `{{$json.link}}`.  
    - Credentials: Scrapeless API.  
    - Connect from Split Out2.

17. **Add a `Set` node:**  
    - Name: `Extract competitor content Markdown`  
    - Assign `markdown` = `{{$json[0].markdown}}`.  
    - Connect from Crawl.

18. **Add an `Aggregate` node:**  
    - Name: `Aggregate`  
    - Aggregate `markdown` fields into an array.  
    - Connect from Extract competitor content Markdown.

19. **Add a `Langchain Agent` node:**  
    - Name: `Senior SEO content writer`  
    - Model: Anthropic Claude Sonnet 4  
    - Prompt: Detailed instructions to write a full SEO article using competitor markdowns, trend insights, and SEO best practices.  
    - Output: Well-formed JSON with article fields.  
    - Connect from Aggregate.

20. **Add a `Code` node:**  
    - Name: `Code`  
    - JavaScript code to parse AI output JSON, clean it, and serialize JSONB fields for database storage.  
    - Connect from Senior SEO content writer.

21. **Add a `Supabase` node:**  
    - Name: `Create a row`  
    - Table: `seo_articles`  
    - Fields: title, meta_description, body, slug  
    - Credentials: Supabase API.  
    - Connect from Code.

22. **Add two `Langchain LM Chat Anthropic` nodes as sub-nodes:**  
    - One connected to AI Agent node.  
    - One connected to Senior SEO content writer node.  
    - Configure model to `claude-sonnet-4-20250514` with valid Anthropic credentials.

23. **Add `Sticky Note` nodes to label phases if desired, positioning them near relevant nodes for clarity.**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow uses the Scrapeless API extensively for Google Trends data, Google Search, and webpage crawling.       | Scrapeless API documentation: https://scrapeless.com/docs                                              |
| AI models are Anthropic Claude Sonnet 4 accessed through Langchain nodes in n8n.                                     | Anthropic API: https://www.anthropic.com/                                                              |
| Google Sheets nodes require OAuth2 credentials with write access to append or update the “Topics” spreadsheet.      | Google Sheets API: https://developers.google.com/sheets/api                                           |
| Supabase node requires API key and URL credentials for connecting to the backend database table `seo_articles`.      | Supabase Docs: https://supabase.com/docs                                                              |
| The AI output parsing step includes robust error handling to catch malformed JSON from AI responses.                 | Logs errors and halts workflow if parsing fails — critical for debugging AI-generated data.            |
| SEO content writing prompt enforces a strict JSON output format to ensure downstream processing compatibility.      | This enables automation of article publication workflows without manual intervention.                   |

---

This completes the detailed reference documentation for the **Automated SEO Content Engine with Claude AI, Scrapeless, and Competitor Analysis** workflow.