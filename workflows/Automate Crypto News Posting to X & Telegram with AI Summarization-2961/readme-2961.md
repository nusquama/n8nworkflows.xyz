Automate Crypto News Posting to X & Telegram with AI Summarization

https://n8nworkflows.xyz/workflows/automate-crypto-news-posting-to-x---telegram-with-ai-summarization-2961


# Automate Crypto News Posting to X & Telegram with AI Summarization

### 1. Workflow Overview

This workflow automates the curation and posting of the latest cryptocurrency news to social media platforms X (formerly Twitter) and Telegram, using AI to summarize content for concise and engaging updates. It is designed for content creators, crypto influencers, marketers, and communities who want to maintain an active social media presence without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & News Retrieval:** Periodically triggers the workflow and fetches recent crypto news from the CryptoPanic API.
- **1.2 News Filtering & Content Extraction:** Filters news items by recency, extracts titles and URLs, visits news pages, and uses AI to extract full article content.
- **1.3 Content Aggregation & AI Summarization:** Aggregates extracted content and uses AI to generate two distinct summaries optimized for X and Telegram.
- **1.4 Automated Posting:** Posts the AI-generated summaries to X and Telegram automatically.
- **1.5 Workflow Termination:** Ends the workflow cleanly after posting.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & News Retrieval

**Overview:**  
This block initiates the workflow every 90 minutes (configurable) and retrieves the latest cryptocurrency news from the CryptoPanic API.

**Nodes Involved:**  
- Set the posting interval (Schedule Trigger)  
- Get Crypto news from CryptoPanic (HTTP Request)  

**Node Details:**

- **Set the posting interval**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow execution on a fixed interval (default 90 minutes)  
  - Configuration: Interval set to 90 minutes (can be customized)  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "Get Crypto news from CryptoPanic"  
  - Edge Cases: Workflow will not run if n8n instance is down or paused; misconfiguration of interval may cause too frequent or infrequent runs.

- **Get Crypto news from CryptoPanic**  
  - Type: HTTP Request  
  - Role: Fetches latest crypto news JSON from CryptoPanic API  
  - Configuration: URL includes API token (replace `"YOURTOKEN"` with actual token); GET method; expects JSON response  
  - Inputs: Trigger from Schedule node  
  - Outputs: Connects to "Extract title and URL"  
  - Edge Cases: API token invalid or expired; API rate limits; network issues; malformed response; empty news list.

---

#### 2.2 News Filtering & Content Extraction

**Overview:**  
Filters news articles to only those published within the last 30 minutes, extracts titles and URLs, visits each news page, and uses AI to extract the core article content.

**Nodes Involved:**  
- Extract title and URL (Code node)  
- Visit the News Page (HTTP Request)  
- ContentExtraction GPT3.5 (OpenAI node)  

**Node Details:**

- **Extract title and URL**  
  - Type: Code (JavaScript)  
  - Role: Parses CryptoPanic API response, filters news published within last 30 minutes, extracts title and URL for each article  
  - Configuration: Custom JavaScript code filtering by timestamp and mapping to title/URL pairs  
  - Inputs: Output from "Get Crypto news from CryptoPanic"  
  - Outputs: Connects to "Visit the News Page"  
  - Edge Cases: Timezone mismatches; empty or malformed API data; code errors; no news items within timeframe.

- **Visit the News Page**  
  - Type: HTTP Request  
  - Role: Fetches full HTML content of each news article URL  
  - Configuration: GET request to each URL from previous node; no output data always set to false (outputs response body)  
  - Inputs: From "Extract title and URL"  
  - Outputs: Connects to "ContentExtraction GPT3.5"  
  - Edge Cases: Broken URLs; HTTP errors (404, 500); slow response or timeouts; redirects; paywalls or login requirements.

- **ContentExtraction GPT3.5**  
  - Type: OpenAI (GPT-3.5)  
  - Role: Uses AI to extract and summarize the core content from the raw HTML of news articles  
  - Configuration: Connected to OpenAI credentials; prompt designed to extract main article text from HTML input  
  - Inputs: HTML content from "Visit the News Page"  
  - Outputs: Connects to "Merge all the news together"  
  - Edge Cases: API rate limits; malformed input; AI misinterpretation; network issues; credential errors.

---

#### 2.3 Content Aggregation & AI Summarization

**Overview:**  
Aggregates all extracted article contents into a single dataset and uses AI to generate two tailored summaries: a concise post for X and a detailed report for Telegram.

**Nodes Involved:**  
- Merge all the news together (Aggregate)  
- Summary news GPT (OpenAI node)  

**Node Details:**

- **Merge all the news together**  
  - Type: Aggregate  
  - Role: Combines multiple AI-extracted article contents into one aggregated input for summarization  
  - Configuration: Aggregates all incoming items into a single array or concatenated string  
  - Inputs: From "ContentExtraction GPT3.5"  
  - Outputs: Connects to "Summary news GPT"  
  - Edge Cases: Large data volume causing performance issues; empty input if no articles extracted.

- **Summary news GPT**  
  - Type: OpenAI (GPT)  
  - Role: Generates two outputs: a concise, engaging post for X and a detailed summary for Telegram, based on aggregated news content  
  - Configuration: Uses OpenAI credentials; prompt includes instructions for style, length, and platform optimization  
  - Inputs: Aggregated news content from previous node  
  - Outputs: Connects to both "Send a news report to Telegram" and "Automatically post to X"  
  - Edge Cases: API limits; prompt failures; output exceeding platform limits; credential errors.

---

#### 2.4 Automated Posting

**Overview:**  
Posts the AI-generated summaries to X and Telegram automatically.

**Nodes Involved:**  
- Send a news report to Telegram (Telegram node)  
- Automatically post to X (Twitter node)  

**Node Details:**

- **Send a news report to Telegram**  
  - Type: Telegram  
  - Role: Sends the detailed Telegram summary to a specified Telegram chat or channel  
  - Configuration: Telegram Bot API token credential; chat ID parameter set to target chat; message content from "Summary news GPT" output  
  - Inputs: From "Summary news GPT"  
  - Outputs: Connects to "No Operation, do nothing" (workflow end)  
  - Edge Cases: Invalid bot token; incorrect chat ID; Telegram API downtime; message size limits.

- **Automatically post to X**  
  - Type: Twitter (X)  
  - Role: Posts the concise summary to the connected X account  
  - Configuration: Twitter API credentials; message content from "Summary news GPT" output; error handling set to continue on failure  
  - Inputs: From "Summary news GPT"  
  - Outputs: Connects to "No Operation, do nothing" (workflow end)  
  - Edge Cases: Twitter API rate limits; authentication errors; message exceeding character limit; network issues.

---

#### 2.5 Workflow Termination

**Overview:**  
Ends the workflow cleanly after posting to both platforms.

**Nodes Involved:**  
- No Operation, do nothing (NoOp node)  

**Node Details:**

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Acts as a terminal node to end the workflow without further action  
  - Configuration: No parameters  
  - Inputs: From both posting nodes  
  - Outputs: None  
  - Edge Cases: None (safe termination)

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                          | Input Node(s)                 | Output Node(s)                      | Sticky Note                          |
|----------------------------|----------------------|----------------------------------------|------------------------------|-----------------------------------|------------------------------------|
| Set the posting interval    | Schedule Trigger     | Triggers workflow every 90 minutes     | None                         | Get Crypto news from CryptoPanic  |                                    |
| Get Crypto news from CryptoPanic | HTTP Request        | Fetches latest crypto news JSON        | Set the posting interval     | Extract title and URL             |                                    |
| Extract title and URL       | Code                 | Filters recent news, extracts title/URL| Get Crypto news from CryptoPanic | Visit the News Page               |                                    |
| Visit the News Page         | HTTP Request         | Fetches full HTML content of news URLs | Extract title and URL        | ContentExtraction GPT3.5          |                                    |
| ContentExtraction GPT3.5    | OpenAI (GPT-3.5)     | Extracts core article content via AI   | Visit the News Page          | Merge all the news together       |                                    |
| Merge all the news together | Aggregate            | Aggregates all extracted article content| ContentExtraction GPT3.5     | Summary news GPT                  |                                    |
| Summary news GPT            | OpenAI (GPT)         | Generates X and Telegram summaries      | Merge all the news together  | Send a news report to Telegram, Automatically post to X |                                    |
| Send a news report to Telegram | Telegram             | Sends detailed summary to Telegram chat| Summary news GPT             | No Operation, do nothing          |                                    |
| Automatically post to X     | Twitter (X)           | Posts concise summary to X              | Summary news GPT             | No Operation, do nothing          |                                    |
| No Operation, do nothing    | NoOp                  | Ends workflow                          | Send a news report to Telegram, Automatically post to X | None                              |                                    |
| Sticky Note1                | Sticky Note           | (Empty)                               |                              |                                   |                                    |
| Sticky Note5                | Sticky Note           | (Empty)                               |                              |                                   |                                    |
| Sticky Note6                | Sticky Note           | (Empty)                               |                              |                                   |                                    |
| Sticky Note7                | Sticky Note           | (Empty)                               |                              |                                   |                                    |
| Sticky Note8                | Sticky Note           | (Empty)                               |                              |                                   |                                    |
| Sticky Note9                | Sticky Note           | (Empty)                               |                              |                                   |                                    |
| Sticky Note10               | Sticky Note           | (Empty)                               |                              |                                   |                                    |
| Sticky Note11               | Sticky Note           | (Empty)                               |                              |                                   |                                    |
| Sticky Note12               | Sticky Note           | (Empty)                               |                              |                                   |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Name: `Set the posting interval`  
   - Type: Schedule Trigger  
   - Configure to run every 90 minutes (adjust as needed).

2. **Create HTTP Request node**  
   - Name: `Get Crypto news from CryptoPanic`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://cryptopanic.com/api/v1/posts/?auth_token=YOURTOKEN` (replace `YOURTOKEN` with your CryptoPanic API token)  
   - Connect output of Schedule Trigger to this node.

3. **Create Code node**  
   - Name: `Extract title and URL`  
   - Type: Code (JavaScript)  
   - Purpose: Filter news published within last 30 minutes and extract title and URL  
   - Sample logic: Parse input JSON, filter by timestamp, output array of objects with `title` and `url` fields  
   - Connect output of HTTP Request node to this node.

4. **Create HTTP Request node**  
   - Name: `Visit the News Page`  
   - Type: HTTP Request  
   - Method: GET  
   - Purpose: Fetch full HTML content of each news article URL  
   - Connect output of Code node to this node.

5. **Create OpenAI node**  
   - Name: `ContentExtraction GPT3.5`  
   - Type: OpenAI (GPT-3.5)  
   - Credentials: Connect your OpenAI API key  
   - Purpose: Extract main article content from HTML input  
   - Connect output of HTTP Request node to this node.

6. **Create Aggregate node**  
   - Name: `Merge all the news together`  
   - Type: Aggregate  
   - Purpose: Combine all extracted article contents into a single input for summarization  
   - Connect output of OpenAI node to this node.

7. **Create OpenAI node**  
   - Name: `Summary news GPT`  
   - Type: OpenAI (GPT)  
   - Credentials: Connect your OpenAI API key  
   - Purpose: Generate two outputs: concise X post and detailed Telegram report  
   - Configure prompt to instruct AI accordingly (e.g., character limits, tone)  
   - Connect output of Aggregate node to this node.

8. **Create Telegram node**  
   - Name: `Send a news report to Telegram`  
   - Type: Telegram  
   - Credentials: Connect your Telegram Bot API token  
   - Parameters: Set Chat ID to your target Telegram chat  
   - Message content: Use detailed summary output from `Summary news GPT`  
   - Connect output of OpenAI summary node to this node.

9. **Create Twitter node**  
   - Name: `Automatically post to X`  
   - Type: Twitter (X)  
   - Credentials: Connect your Twitter developer credentials  
   - Message content: Use concise summary output from `Summary news GPT`  
   - Error handling: Set to continue on failure to avoid workflow stop  
   - Connect output of OpenAI summary node to this node.

10. **Create No Operation node**  
    - Name: `No Operation, do nothing`  
    - Type: NoOp  
    - Purpose: Terminal node to end workflow cleanly  
    - Connect outputs of both Telegram and Twitter nodes to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Obtain CryptoPanic API token at https://cryptopanic.com/                                             | Required for news retrieval                                                                        |
| Obtain OpenAI API key at https://platform.openai.com/                                                | Required for AI content extraction and summarization                                             |
| Create Twitter Developer account at https://developer.twitter.com/                                   | Required for posting to X (Twitter)                                                              |
| Create Telegram bot with BotFather and get Chat ID                                                  | Required for posting to Telegram                                                                  |
| Customize prompts in "Summary news GPT" to adapt tone, style, and platform-specific requirements    | Allows tailoring content for different audiences and platforms                                   |
| Adjust schedule trigger interval to control posting frequency                                        | Enables flexible update timing                                                                     |
| Workflow by Tianyi (muzi) - n8n Creators Profile: https://n8n.io/creators/muzi/                       | Author and source                                                                                  |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.