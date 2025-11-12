Find Engagement Opportunities from Skool Communities using Apify & GPT-4.1

https://n8nworkflows.xyz/workflows/find-engagement-opportunities-from-skool-communities-using-apify---gpt-4-1-6617


# Find Engagement Opportunities from Skool Communities using Apify & GPT-4.1

---

### 1. Workflow Overview

This workflow automates the identification of engagement opportunities from Skool community posts and comments by integrating data extraction from Skool communities via Apify with AI-driven analysis using OpenAI GPT-4.1. Its primary purpose is to scan recent posts and comments, evaluate them for relevant engagement chances, and store actionable insights for organic community interaction.

The workflow is logically divided into these blocks:

- **1.1 Configuration Retrieval and Preparation**: Fetches configuration data from Airtable, filters active entries, and prepares URLs and other parameters for scraping.
- **1.2 Data Extraction from Skool Communities**: Uses Apify’s Skool posts scraper to retrieve posts and comments based on the configuration.
- **1.3 Data Transformation and Filtering**: Extracts relevant post and comment fields, filters comments by length, and prepares data for AI analysis.
- **1.4 AI-Based Opportunity Evaluation and Comment Generation**: Uses GPT-4.1 to analyze posts/comments to identify engagement opportunities and suggest comments.
- **1.5 Results Processing and Storage**: Merges AI output with original data, filters for positive opportunities, and records them back into Airtable.
- **1.6 Workflow Trigger and Scheduling**: Defines the timing for workflow execution.

Sticky notes provide context and guidance across these blocks.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration Retrieval and Preparation

- **Overview**: Retrieves workflow configuration from Airtable, filters active configurations, and formats URLs and other parameters for downstream use.
- **Nodes Involved**: `Get Config`, `Is Active?`, `Extract Config`, `Split Out1`, `Sticky Note2`, `Sticky Note4`
  
##### Node Details:

- **Get Config**
  - *Type*: Airtable node (v2.1)
  - *Role*: Fetches records from the "config" table in the specified Airtable base.
  - *Config*: Searches entire table without filters, uses Airtable Personal Access Token credential.
  - *Input*: Triggered by `Schedule Trigger`.
  - *Output*: Raw configuration JSON records.
  - *Failures*: Airtable API rate limits, auth errors.
  
- **Is Active?**
  - *Type*: Filter node (v2.2)
  - *Role*: Filters config records to only those with `active` field set to `true`.
  - *Config*: Boolean condition on `active` attribute.
  - *Input*: Output from `Get Config`.
  - *Output*: Active config records.
  
- **Extract Config**
  - *Type*: Set node (v3.4)
  - *Role*: Transforms config record into structured fields for URLs, cookies, domain, tools, and name.
  - *Config*: 
    - Splits comma-separated Skool URLs into an array of trimmed objects.
    - Cleans `cookies` string.
    - Passes domain, tools used, and name fields as strings.
  - *Input*: From `Is Active?`.
  - *Output*: Cleaned config data.
  - *Edge Cases*: Empty or malformed cookies or URLs may cause issues downstream.
  
- **Split Out1**
  - *Type*: Split Out node (v1)
  - *Role*: Splits the array of Skool URLs into individual items to process separately.
  - *Config*: Splits on `Skool URLs` array field.
  - *Input*: From `Extract Config`.
  - *Output*: Individual URL items for scraping.

- **Sticky Note2** and **Sticky Note4**
  - Provide contextual notes about the Airtable config template and grouping URLs due to an Apify actor limitation.

---

#### 2.2 Data Extraction from Skool Communities

- **Overview**: Scrapes the latest posts and comments from the configured Skool community URLs using Apify’s actor with authentication and proxy.
- **Nodes Involved**: `Get Skool Posts`, `Sticky Note`, `Sticky Note1`
  
##### Node Details:

- **Get Skool Posts**
  - *Type*: HTTP Request node (v4.2)
  - *Role*: Calls Apify API to run a scraper that fetches posts and comments.
  - *Config*: 
    - POST request with JSON body specifying:
      - `commentsLimit`: 20 (max comments per post)
      - `cookies`: passed from config (for authentication)
      - `includeComments`: true
      - `includeMedia`: false (media not fetched)
      - `itemStartDate`: last 24 hours (dynamic)
      - `proxy`: use Apify proxy
      - `startUrls`: single Skool URL item per request (due to Apify bug)
      - `tab`: "community"
    - Uses HTTP Query Auth credential for Apify.
  - *Input*: From `Split Out1`.
  - *Output*: Raw posts and comments JSON.
  - *Failures/Edge Cases*: 
    - Authentication failure if cookies expire or invalid.
    - Rate limiting or proxy errors from Apify.
    - Empty or malformed response if no posts in timeframe.
  
- **Sticky Note** and **Sticky Note1**
  - Explain the data extraction purpose: fetching posts plus comments, filtering comments over 50 characters, and extracting URLs.

---

#### 2.3 Data Transformation and Filtering

- **Overview**: Processes the raw posts/comments data to cleanly extract relevant fields for AI analysis, focusing on content and comments longer than 50 characters.
- **Nodes Involved**: `Extract Post Data`
  
##### Node Details:

- **Extract Post Data**
  - *Type*: Set node (v3.4)
  - *Role*: Selects and formats key fields from raw API data.
  - *Config*:
    - Extracts post name, main content, filtered comments (post metadata content with length > 50 chars), likes (upvotes), creation date, post type, and URL.
  - *Input*: From `Get Skool Posts`.
  - *Output*: Cleaned post and comment data.
  - *Edge Cases*: Posts with no comments > 50 chars result in empty comment arrays; ensure AI handles empty comments gracefully.

---

#### 2.4 AI-Based Opportunity Evaluation and Comment Generation

- **Overview**: Uses GPT-4.1 via Langchain integration to analyze each post and its comments, detect engagement opportunities, and generate suggested comments aligned with domain and tools specified in the config.
- **Nodes Involved**: `EvaluateOpportunities And Generate Comments`, `OpenAI Chat Model`, `Structured Output Parser`, `Sticky Note3`
  
##### Node Details:

- **EvaluateOpportunities And Generate Comments**
  - *Type*: Langchain Chain LLM node (v1.7)
  - *Role*: Sends prompt to GPT-4.1 to evaluate post/comment data and generate structured JSON output.
  - *Config*:
    - Prompt template includes:
      - User domain and tools (dynamic from `Extract Config`)
      - Role as stealth engagement strategist
      - Task: analyze content and comments for engagement opportunities
      - Output JSON schema example for opportunity, reason, trigger sentence, suggested comment
      - Instructions to avoid direct promotion, translate comment to original language, and be insightful.
    - Input text combines post content and comments.
    - Output parsed by `Structured Output Parser`.
  - *Input*: From `Extract Post Data`.
  - *Output*: Parsed JSON with opportunity evaluation and suggested comment.
  - *Failures*: Possible prompt errors, API rate limits, parsing failures if AI output deviates from JSON schema.
  
- **OpenAI Chat Model**
  - *Type*: Langchain OpenAI Chat Model (v1.2)
  - *Role*: Provides GPT-4.1 language model functionality with credentials.
  - *Input*: Connected internally by Langchain node.
  - *Output*: Raw AI responses.

- **Structured Output Parser**
  - *Type*: Langchain Structured Output Parser (v1.3)
  - *Role*: Parses AI text completion into structured JSON format based on schema example.
  
- **Sticky Note3**
  - Highlights the AI analysis block purpose and dynamic nature based on config criteria.

---

#### 2.5 Results Processing and Storage

- **Overview**: Merges AI evaluation with original post data, filters only positive engagement opportunities, and upserts these results into an Airtable table for tracking.
- **Nodes Involved**: `Merge AI Answers With Extracted Data`, `Filter Opportunities To Comment`, `Record Results`
  
##### Node Details:

- **Merge AI Answers With Extracted Data**
  - *Type*: Set node (v3.4)
  - *Role*: Combines AI output fields (`opportunity`, `reason`, `trigger_sentence`, `suggested_comment`) with original post metadata (`name`, `createdAt`, `url`).
  - *Input*: From `EvaluateOpportunities And Generate Comments`.
  - *Output*: Enriched data object for filtering.

- **Filter Opportunities To Comment**
  - *Type*: Filter node (v2.2)
  - *Role*: Passes forward only items where `opportunity` is `true`.
  - *Input*: From `Merge AI Answers With Extracted Data`.
  - *Output*: Filtered engagement opportunities.

- **Record Results**
  - *Type*: Airtable node (v2.1)
  - *Role*: Upserts engagement opportunities into Airtable table "Table 1" within the same base.
  - *Config*:
    - Maps fields: title, date (current timestamp), url, reason, trigger sentence, suggested comment, and config name.
    - Uses "title" as matching column for upsert.
    - Includes a status Select field ("not commented" or "commented") default behavior unspecified.
    - Uses Airtable Personal Access Token credential.
  - *Input*: From `Filter Opportunities To Comment`.
  - *Output*: None (terminal).
  - *Failures*: Airtable API errors, upsert conflicts.

---

#### 2.6 Workflow Trigger and Scheduling

- **Overview**: Defines the scheduled execution time of the workflow (daily at 19:00).
- **Nodes Involved**: `Schedule Trigger`
  
##### Node Details:

- **Schedule Trigger**
  - *Type*: Schedule Trigger node (v1.2)
  - *Role*: Starts workflow at 19:00 daily.
  - *Config*: Hour-based trigger set to 19:00 local time.
  - *Output*: Triggers the start node `Get Config`.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                                  | Input Node(s)                  | Output Node(s)                   | Sticky Note                                               |
|----------------------------------|----------------------------------|-------------------------------------------------|-------------------------------|---------------------------------|------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger (v1.2)           | Initiates workflow daily at 19:00                | -                             | Get Config                      |                                                            |
| Get Config                      | Airtable (v2.1)                  | Fetches config records from Airtable             | Schedule Trigger              | Is Active?                     | ## Get config Get config from airtable group it by url...  |
| Is Active?                     | Filter (v2.2)                    | Filters active configurations                     | Get Config                   | Extract Config                 |                                                            |
| Extract Config                 | Set (v3.4)                      | Parses and structures config fields               | Is Active?                   | Split Out1                    |                                                            |
| Split Out1                    | Split Out (v1)                  | Splits Skool URLs array into individual items    | Extract Config               | Get Skool Posts               |                                                            |
| Get Skool Posts               | HTTP Request (v4.2)              | Calls Apify API to get posts and comments         | Split Out1                  | Extract Post Data             | ## Get posts + comments from the choosen communities       |
| Extract Post Data             | Set (v3.4)                      | Extracts and filters relevant post/comment data  | Get Skool Posts             | EvaluateOpportunities And Generate Comments | ## Extract data - Content - comments > 50chars - url       |
| EvaluateOpportunities And Generate Comments | Langchain Chain LLM (v1.7)        | Analyzes posts/comments with GPT-4.1 for engagement opportunities | Extract Post Data            | Merge AI Answers With Extracted Data | ## Analyze posts with AI This is dynamic based on the config criterias |
| OpenAI Chat Model             | Langchain OpenAI Chat Model (v1.2) | Provides GPT-4.1 model for AI analysis            | Internal to EvaluateOpportunities And Generate Comments | EvaluateOpportunities And Generate Comments |                                                            |
| Structured Output Parser      | Langchain Output Parser (v1.3)   | Parses AI response into structured JSON           | Internal to EvaluateOpportunities And Generate Comments | EvaluateOpportunities And Generate Comments |                                                            |
| Merge AI Answers With Extracted Data | Set (v3.4)                      | Combines AI output with original post data        | EvaluateOpportunities And Generate Comments | Filter Opportunities To Comment    |                                                            |
| Filter Opportunities To Comment | Filter (v2.2)                   | Filters only positive engagement opportunities    | Merge AI Answers With Extracted Data | Record Results                  |                                                            |
| Record Results                | Airtable (v2.1)                  | Upserts filtered opportunities into Airtable      | Filter Opportunities To Comment | -                               |                                                            |
| Sticky Note                   | Sticky Note (v1)                  | Context: Get posts + comments from communities    | -                             | -                               | ## Get posts + comments from the choosen communities       |
| Sticky Note1                  | Sticky Note (v1)                  | Context: Extract data - content and comments       | -                             | -                               | ## Extract data - Content - comments > 50chars - url       |
| Sticky Note2                  | Sticky Note (v1)                  | Context: Get config from Airtable                   | -                             | -                               | ## Get config Get config from airtable group it by url...  |
| Sticky Note3                  | Sticky Note (v1)                  | Context: AI analysis of posts and comments          | -                             | -                               | ## Analyze posts with AI This is dynamic based on the config criterias |
| Sticky Note4                  | Sticky Note (v1)                  | Airtable template link for config                    | -                             | -                               | ## Airtable tempplate https://airtable.com/appImGQn0rh53oCPE/shrqBY3WUtMxUZnYa |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to run daily at 19:00 local time.

2. **Create an Airtable node named `Get Config`**  
   - Operation: Search  
   - Base: `appImGQn0rh53oCPE` (Skool comments base)  
   - Table: `tblfSVVKzXJxrOXEY` (config table)  
   - Credentials: Airtable Personal Access Token  
   - Connect output of Schedule Trigger to this node.

3. **Create a Filter node named `Is Active?`**  
   - Condition: `active` field equals `true` (boolean strict)  
   - Connect input from `Get Config`.

4. **Create a Set node named `Extract Config`**  
   - Assignments:  
     - `Skool URLs` (array): split the string from the config field `Skool URLs` by comma, trim whitespace, create objects with `url` key.  
     - `cookies` (string): clean newlines from `cookies` config field.  
     - `Domain of Activity` (string): from config field.  
     - `Tools Used` (string): from config field.  
     - `Name` (string): from config field.  
   - Connect input from `Is Active?`.

5. **Create a Split Out node named `Split Out1`**  
   - Field to split out: `Skool URLs`  
   - Connect input from `Extract Config`.

6. **Create an HTTP Request node named `Get Skool Posts`**  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/memo23~skool-posts-with-comments-scraper/run-sync-get-dataset-items`  
   - Authentication: HTTP Query Auth with Apify credentials  
   - Body Content-Type: JSON  
   - Body JSON:  
     ```json
     {
       "commentsLimit": 20,
       "cookies": {{ $json.cookies }},
       "includeComments": true,
       "includeMedia": false,
       "itemStartDate": "{{ $now.minus(24,'hours') }}",
       "proxy": { "useApifyProxy": true },
       "startUrls": [{ "url": "{{ $json['Skool URLs'].url }}" }],
       "tab": "community"
     }
     ```
   - Connect input from `Split Out1`.

7. **Create a Set node named `Extract Post Data`**  
   - Assignments:  
     - `name`: post’s `name` field  
     - `content`: `metadata.content`  
     - `comments`: filter `comments` array for those with content length > 50 chars (map `item.post.metadata.content`)  
     - `likes`: `metadata.upvotes`  
     - `createdAt`: `createdAt`  
     - `postType`: `postType`  
     - `url`: `url`  
   - Connect input from `Get Skool Posts`.

8. **Create a Langchain Chain LLM node named `EvaluateOpportunities And Generate Comments`**  
   - Model: GPT-4.1 (OpenAI) with valid OpenAI API credentials  
   - Prompt: Use a custom prompt template that includes:  
     - Injected variables from `Extract Config` for domain and tools  
     - Instructions to act as stealth engagement strategist  
     - JSON output schema example (opportunity, reason, trigger_sentence, suggested_comment)  
     - Input text combining post content and comments  
   - Enable output parser with a structured output parser node.

9. **Create a Langchain OpenAI Chat Model node**  
   - Model: GPT-4.1  
   - Credentials: OpenAI API  
   - Connect internally to Langchain Chain LLM node.

10. **Create a Langchain Structured Output Parser node**  
    - JSON schema example matching the prompt’s expected output.  
    - Connect internally to Langchain Chain LLM node.

11. **Create a Set node named `Merge AI Answers With Extracted Data`**  
    - Assignments combining AI output (`opportunity`, `reason`, `trigger_sentence`, `suggested_comment`) with original post metadata (`name`, `createdAt`, `url`).  
    - Connect input from `EvaluateOpportunities And Generate Comments`.

12. **Create a Filter node named `Filter Opportunities To Comment`**  
    - Condition: `opportunity` equals `true` boolean strict  
    - Connect input from `Merge AI Answers With Extracted Data`.

13. **Create an Airtable node named `Record Results`**  
    - Operation: Upsert  
    - Base: `appImGQn0rh53oCPE`  
    - Table: `tblkB1UJ5dihXvDoi` (Table 1)  
    - Map fields:  
      - `url` from JSON url  
      - `date` current timestamp (`$now`)  
      - `title` from post name  
      - `config` array with config name  
      - `reason` from AI output  
      - `trigger` from AI output trigger sentence  
      - `suggested answer` from AI suggested comment  
    - Matching column: `title`  
    - Credentials: Airtable Personal Access Token  
    - Connect input from `Filter Opportunities To Comment`.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Airtable config template URL: https://airtable.com/appImGQn0rh53oCPE/shrqBY3WUtMxUZnYa                         | Provides structure for configuration data used by the workflow.                                                 |
| The Apify Skool posts scraper actor has a known limitation: it only processes the first URL if multiple URLs are passed in an array. Hence, URLs are processed individually. | Sticky Note2 explains this workaround.                                                                          |
| AI prompt and output are carefully designed to avoid direct promotion and encourage authentic, helpful engagement aligned with the user's domain and tools. | Sticky Note3 describes the AI analysis strategy and ethical guidelines.                                         |
| Comments shorter than 50 characters are filtered out to focus on meaningful engagement content.                 | Sticky Note1 specifies this data filtering criterion.                                                           |

---

*Disclaimer: The text provided is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.*