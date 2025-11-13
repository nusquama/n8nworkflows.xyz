Smart RSS News Alert System with DeepSeek AI and Slack Notifications

https://n8nworkflows.xyz/workflows/smart-rss-news-alert-system-with-deepseek-ai-and-slack-notifications-8131


# Smart RSS News Alert System with DeepSeek AI and Slack Notifications

### 1. Workflow Overview

This workflow automates the processing and alerting of news articles from an RSS feed using AI analysis and Slack notifications. It is designed to:

- Monitor an RSS feed for new articles.
- Normalize and clean the article metadata and content.
- Check if an article is new to avoid duplication.
- Fetch full article content via HTTP.
- Prepare and send the article text to a Large Language Model (LLM) for summarization, sentiment analysis, keyword extraction, and flag detection.
- Filter articles by relevance based on sentiment and specific flagged topics.
- Send relevant news alerts to a specified Slack channel.

Logical blocks:

- **1.1 Input Reception: RSS Feed Trigger**
- **1.2 Data Normalization and Deduplication**
- **1.3 Article Content Fetching**
- **1.4 AI Processing and Analysis**
- **1.5 Relevance Filtering**
- **1.6 Slack Notification Dispatch**

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception: RSS Feed Trigger

**Overview:**  
This block initiates the workflow by polling a specified RSS feed at a scheduled time to retrieve the latest news articles.

**Nodes involved:**  
- RSS Feed

**Node details:**

- **RSS Feed**
  - Type: RSS Feed Trigger
  - Role: Polls the provided RSS feed URL and triggers workflow execution when new feed items are available.
  - Configuration:  
    - Feed URL set to TechCrunch feed (`https://techcrunch.com/feed/`).
    - Polling scheduled daily at 09:00 AM.
  - Inputs: None (trigger node).
  - Outputs: Emits feed items containing article metadata.
  - Edge Cases:  
    - Feed unreachable/network issues.
    - Empty or malformed RSS feed.
    - Rate limiting or feed changes.
  - Notes:  
    - Sticky note nearby outlines setup instructions related to this node.

#### 1.2 Data Normalization and Deduplication

**Overview:**  
This block cleans and normalizes incoming RSS feed items, extracts a text snippet, computes a URL hash to uniquely identify articles, and uses static workflow data to track and filter out already processed items.

**Nodes involved:**  
- normalize & hash  
- parse HTML  
- New?

**Node details:**

- **normalize & hash**  
  - Type: Code (JavaScript)  
  - Role: Cleans HTML tags from title and content, consolidates text, truncates to 6000 characters, and computes a consistent hash of the article URL using the FNV-1a algorithm for deduplication.  
  - Key expressions:  
    - `fnv1a(str)` implements FNV-1a hashing.  
    - Extracts URL, title, and content fields from RSS JSON item.  
    - Normalizes text by removing HTML tags and whitespace.  
  - Inputs: RSS feed items.  
  - Outputs: Items enriched with `url`, `title`, `text_rss`, and `url_hash` fields.  
  - Edge Cases:  
    - Missing or malformed fields in RSS item.  
    - Very large text content (truncated).  
    - Non-string URL values.

- **parse HTML**  
  - Type: Code (JavaScript)  
  - Role: Maintains a persistent global store of seen article hashes with timestamps to track article freshness; marks new articles with a boolean `_isNew`. Also purges old entries beyond 14 days or exceeding 800 entries.  
  - Key expressions:  
    - Uses workflow static data (`$getWorkflowStaticData('global')`).  
    - Cleans stale hashes beyond retention period or max count.  
  - Inputs: Items from previous node with hashes.  
  - Outputs: Items tagged with `_isNew` boolean.  
  - Edge Cases:  
    - Data store persistence issues.  
    - Hash collisions (unlikely).  
    - Workflow concurrency affecting static data integrity.

- **New?**  
  - Type: If  
  - Role: Filters items, passing only those marked as new (`_isNew == true`) downstream.  
  - Input: Items with `_isNew` flag.  
  - Output: Only new articles proceed forward.  
  - Edge Cases:  
    - Expression parsing failures.  
    - No new items available.

#### 1.3 Article Content Fetching

**Overview:**  
Fetches the full HTML content of the article URL for deeper analysis.

**Nodes involved:**  
- fetch article

**Node details:**

- **fetch article**  
  - Type: HTTP Request  
  - Role: Requests the article URL to retrieve the full HTML content.  
  - Configuration:  
    - URL dynamically set from article `url` field.  
    - Timeout set to 8000 ms (8 seconds).  
    - Response format: text.  
  - Input: New article items from "New?" node.  
  - Output: Items enriched with full HTML content in `data` field.  
  - Edge Cases:  
    - HTTP errors (404, 500, timeouts).  
    - Redirect loops or non-HTML content.  
    - Slow response exceeding timeout.

#### 1.4 AI Processing and Analysis

**Overview:**  
Prepares article text for LLM input, sends it for AI-based summarization and classification, and parses the resulting JSON output.

**Nodes involved:**  
- Prepare LLM Input  
- analyze with LLM  
- parse LLM JSON  
- DeepSeek Chat Model (credential node, auxiliary)

**Node details:**

- **Prepare LLM Input**  
  - Type: Code (JavaScript)  
  - Role: Cleans fetched HTML by stripping scripts/styles and tags, extracts plain text, truncates to 9000 characters, and prepares a `text_for_llm` field.  
  - Logic:  
    - Removes `<script>`, `<style>`, and other tags.  
    - Uses cleaned text if longer than 500 characters, else falls back to original RSS text snippet.  
  - Input: Items with fetched HTML in `data`.  
  - Output: Items with `text_for_llm` and updated `data` fields.  
  - Edge Cases:  
    - Malformed HTML.  
    - Very short or empty content.

- **analyze with LLM**  
  - Type: Langchain Chain LLM Node  
  - Role: Sends prepared text to a Large Language Model (DeepSeek) for generating a JSON-formatted analysis including summary (max 120 words), sentiment, keywords, and flags.  
  - Configuration:  
    - Prompt template includes title, URL, and text.  
    - Expects JSON response with fields: summary, sentiment, keywords, flags.  
    - Uses DeepSeek API credentials.  
  - Input: Items with `text_for_llm`.  
  - Output: Raw JSON text from LLM.  
  - Edge Cases:  
    - API authentication errors.  
    - Rate limits or timeouts.  
    - Malformed or unexpected LLM output.

- **parse LLM JSON**  
  - Type: Code (JavaScript)  
  - Role: Parses the raw JSON string returned by LLM into structured fields: summary, sentiment, keywords, and flags. Defaults are provided for robustness.  
  - Input: Raw LLM output.  
  - Output: Items with parsed JSON fields as native types.  
  - Edge Cases:  
    - JSON parse errors.  
    - Missing fields or wrong data types.

- **DeepSeek Chat Model**  
  - Type: Langchain Language Model Credential Node  
  - Role: Provides DeepSeek API credentials for the LLM node.  
  - Inputs/Outputs: Connected as ai_languageModel node input for the LLM chain.  
  - Edge Cases:  
    - Credential misconfiguration.

#### 1.5 Relevance Filtering

**Overview:**  
Filters analyzed articles to forward only those deemed relevant based on sentiment or presence of specific flags.

**Nodes involved:**  
- Relevant?

**Node details:**

- **Relevant?**  
  - Type: If  
  - Role: Passes articles that either have non-neutral sentiment or include any of the flags: acquisition, funding, product_launch.  
  - Logic:  
    - Checks `sentiment != neutral` OR flags contain any of the specified values.  
  - Input: Items with parsed LLM fields.  
  - Output: Only relevant items proceed.  
  - Edge Cases:  
    - Expression evaluation errors.  
    - Empty or missing flags array.

#### 1.6 Slack Notification Dispatch

**Overview:**  
Sends a formatted news alert message to a designated Slack channel including summary, sentiment, flags, and a link to the article.

**Nodes involved:**  
- send to Slack

**Node details:**

- **send to Slack**  
  - Type: Slack Node  
  - Role: Posts a formatted message to Slack channel with news alert details.  
  - Configuration:  
    - Message includes article title, summary, sentiment, flags, and clickable URL.  
    - Channel ID specified (hidden, replaced by placeholder "ID").  
    - Slack API credentials configured.  
    - Disables link to workflow in Slack message.  
  - Input: Relevant articles only.  
  - Output: None (terminal node).  
  - Edge Cases:  
    - Slack API authentication failure.  
    - Channel ID invalid or missing permissions.  
    - Message formatting issues.

---

### 3. Summary Table

| Node Name          | Node Type                         | Functional Role                           | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                      |
|--------------------|----------------------------------|------------------------------------------|------------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| RSS Feed           | RSS Feed Trigger                 | Triggers workflow on new RSS feed items | —                      | normalize & hash       | ## Setup steps 1. Add your RSS feed URL in the **RSS Feed** node                                 |
| normalize & hash   | Code                            | Cleans and hashes URL for deduplication | RSS Feed               | parse HTML             |                                                                                                 |
| parse HTML         | Code                            | Tracks seen articles and marks new ones | normalize & hash       | New?                   |                                                                                                 |
| New?               | If                              | Filters only new articles                 | parse HTML             | fetch article          |                                                                                                 |
| fetch article      | HTTP Request                    | Retrieves full article HTML content      | New?                   | Prepare LLM Input      |                                                                                                 |
| Prepare LLM Input  | Code                            | Cleans HTML and prepares text for LLM   | fetch article          | analyze with LLM       |                                                                                                 |
| DeepSeek Chat Model| Langchain Language Model         | Provides DeepSeek API credentials        | —                      | analyze with LLM (ai_languageModel) |                                                                                                 |
| analyze with LLM   | Langchain Chain LLM              | Performs summarization, sentiment, flags| Prepare LLM Input, DeepSeek Chat Model | parse LLM JSON         |                                                                                                 |
| parse LLM JSON     | Code                            | Parses LLM JSON response                  | analyze with LLM       | Relevant?              |                                                                                                 |
| Relevant?          | If                              | Filters articles by relevance criteria   | parse LLM JSON         | send to Slack          |                                                                                                 |
| send to Slack      | Slack Node                      | Sends news alert messages to Slack       | Relevant?              | —                      | ## Setup steps 2. Configure your Slack Bot Token in the **Send to Slack** node                   |
| Sticky Note        | Sticky Note                     | Overview of workflow steps                | —                      | —                      | ## How it works - Collects news from RSS feeds - Extracts and cleans article text - Sends text to LLM for analysis - Filters only relevant content - Posts alerts directly to Slack |
| Sticky Note1       | Sticky Note                     | Setup instructions                        | —                      | —                      | ## Setup steps 1. Add your RSS feed URL in the **RSS Feed** node 2. Configure your Slack Bot Token in the **Send to Slack** node 3. Add your LLM (DeepSeek/OpenAI) API key in the **analyze with LLM** node 4. Save & activate the workflow |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger node**  
   - Type: RSS Feed Trigger  
   - Parameters:  
     - Feed URL: `https://techcrunch.com/feed/`  
     - Poll Time: Daily at 09:00 AM

2. **Add Code node “normalize & hash”**  
   - Type: Code (JavaScript)  
   - Parameters:  
     - Implement FNV-1a hash function.  
     - Extract `url`, `title`, and clean content from RSS item.  
     - Normalize text by removing HTML tags and whitespace.  
     - Truncate text to 6000 chars.  
     - Assign `url_hash` as hash of lowercase URL.  
   - Connect input from RSS Feed node.

3. **Add Code node “parse HTML”**  
   - Type: Code (JavaScript)  
   - Parameters:  
     - Maintain global static data store for `seen` hashes with timestamps.  
     - Mark each item with `_isNew` boolean if not seen before.  
     - Purge entries older than 14 days or if total entries exceed 800.  
   - Connect input from “normalize & hash”.

4. **Add If node “New?”**  
   - Type: If  
   - Parameters:  
     - Condition: `$json["_isNew"] == true` (boolean true)  
   - Connect input from “parse HTML”.  
   - Connect output (true) to next node.

5. **Add HTTP Request node “fetch article”**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `={{$json["url"]}}` (dynamic from item)  
     - Timeout: 8000 ms  
     - Response format: text  
   - Connect input from “New?”.

6. **Add Code node “Prepare LLM Input”**  
   - Type: Code (JavaScript)  
   - Parameters:  
     - Remove `<script>`, `<style>`, and all HTML tags from fetched content.  
     - Extract plain text, trim whitespace, truncate to 9000 chars.  
     - If plain text length < 500 use previously extracted RSS text snippet.  
     - Assign cleaned text to `text_for_llm` and override `data` field.  
   - Connect input from “fetch article”.

7. **Add Langchain Language Model Credential node “DeepSeek Chat Model”**  
   - Type: Langchain Language Model Credential  
   - Parameters:  
     - Provide DeepSeek API key and configuration (set up credential in n8n).  
   - No input connections (credential node).

8. **Add Langchain Chain LLM node “analyze with LLM”**  
   - Type: Langchain Chain LLM  
   - Parameters:  
     - Prompt template requesting JSON with summary (≤120 words), sentiment (positive/negative/neutral), keywords (up to 6), and flags (from given list).  
     - Include dynamic fields for title, URL, and text from previous node.  
   - Connect input from “Prepare LLM Input” (main input).  
   - Connect ai_languageModel input to “DeepSeek Chat Model” (credential).  

9. **Add Code node “parse LLM JSON”**  
   - Type: Code (JavaScript)  
   - Parameters:  
     - Parse JSON string from LLM output safely.  
     - Extract `summary`, `sentiment` (default neutral lowercase), `keywords` (array), `flags` (array).  
   - Connect input from “analyze with LLM”.

10. **Add If node “Relevant?”**  
    - Type: If  
    - Parameters:  
      - Condition: Pass if sentiment is not "neutral" OR flags array contains any of: "acquisition", "funding", "product_launch".  
    - Connect input from “parse LLM JSON”.  
    - Connect output (true) to next node.

11. **Add Slack node “send to Slack”**  
    - Type: Slack Node  
    - Parameters:  
      - Channel ID: Specify target Slack channel ID.  
      - Message text:  
        ```
        =*News Alert* 📰  
        *{{ $('RSS Feed').item.json.title }}*  
        {{$json["summary"]}}  

        Sentiment: {{$json["sentiment"]}}  
        Flags: {{ $json["flags"].join(", ") || "—" }}  

        <{{ $('RSS Feed').item.json.link }}>
        ```
      - Disable workflow link inclusion in message.  
    - Credentials: Configure Slack API credentials with appropriate bot token.  
    - Connect input from “Relevant?”.

12. **Activate the workflow** after verifying all nodes and credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| Workflow collects news from RSS feeds, extracts and cleans article text, sends it to a Large Language Model for analysis, filters relevant content, and posts alerts to Slack. | Overview Sticky Note in the workflow  |
| Setup steps include adding RSS feed URL, configuring Slack bot token, adding LLM API key (DeepSeek/OpenAI), then saving and activating the workflow.                     | Setup Sticky Note in the workflow     |
| The workflow uses DeepSeek AI as the LLM provider but can be adapted to other LLM providers by changing the Langchain model node and credentials.                        | Implied by usage of DeepSeek node     |
| Slack messages are formatted in markdown style with clear alert structure, including clickable URLs and flags for easy monitoring by teams.                              | Slack node message formatting details |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow and complies strictly with content policies. All handled data is legal and public.