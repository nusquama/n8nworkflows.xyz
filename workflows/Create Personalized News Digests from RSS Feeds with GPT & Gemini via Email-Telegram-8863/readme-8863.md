Create Personalized News Digests from RSS Feeds with GPT & Gemini via Email/Telegram

https://n8nworkflows.xyz/workflows/create-personalized-news-digests-from-rss-feeds-with-gpt---gemini-via-email-telegram-8863


# Create Personalized News Digests from RSS Feeds with GPT & Gemini via Email/Telegram

### 1. Workflow Overview

This workflow automates the process of creating personalized daily news digests by fetching recent articles from an RSS feed, extracting and cleaning their content, summarizing key information using AI models (OpenAI GPT and Google Gemini), and delivering the final digest via email and Telegram. It targets users who want to receive concise, curated news summaries automatically from their favorite RSS feeds.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Feed Filtering:** Trigger and fetch RSS feed items, filtering only those published within the last 24 hours.
- **1.2 Link Extraction & Limiting:** Extract article links and limit processing to the top 3 items for performance and relevance.
- **1.3 Content Scraping & Extraction:** Use ScrapeGraphAI to convert article webpages into clean markdown, then extract the main article content using Google Gemini and Langchain’s Information Extractor.
- **1.4 Aggregation & Summarization:** Aggregate extracted contents and generate a summarized email body and subject via OpenAI GPT-5 Mini.
- **1.5 Delivery:** Send the generated news digest simultaneously via Gmail email and Telegram message.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Feed Filtering

**Overview:**  
This block initiates the workflow manually, reads an RSS feed URL, and filters articles published within the past 24 hours to ensure freshness.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual trigger)  
- RSS Read  
- Filter  
- Sticky Note1 (instructional)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual execution.  
  - Config: No parameters needed.  
  - Input: None (start node)  
  - Output: Triggers RSS Read node.  
  - Failures: User must execute manually; no automation trigger.  
  - Notes: Entry point for the entire workflow.

- **RSS Read**  
  - Type: RSS Feed Read  
  - Role: Fetches RSS feed items from a configured URL.  
  - Config: URL_FEED placeholder to be replaced with actual RSS feed URL.  
  - Input: Trigger from Manual Trigger  
  - Output: Emits JSON items representing feed entries.  
  - Failures: Invalid URL, feed downtime, malformed XML.  
  - Notes: Set your RSS feed URL here.

- **Filter**  
  - Type: Filter  
  - Role: Filters articles published after the current date minus one day (last 24 hours).  
  - Config: Condition comparing item’s `pubDate` to `today.minus({ days: 1 })`.  
  - Input: RSS Read output  
  - Output: Only recent articles pass through.  
  - Failures: Date parsing errors if `pubDate` missing or malformed.  
  - Notes: Ensures relevance by date filtering.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Instruction to set up and filter latest RSS feed (24 hours)  
  - Content: "Set up and filter the latest RSS feed (24 hours)"

---

#### 1.2 Link Extraction & Limiting

**Overview:**  
Extract the article link from each item and limit processing to the top 3 articles to optimize resource usage.

**Nodes Involved:**  
- Set link  
- Limit  
- Sticky Note5 (instructional)

**Node Details:**

- **Set link**  
  - Type: Set  
  - Role: Extracts the `link` field from the RSS item JSON and assigns it to a variable `link`.  
  - Config: Expression `={{ $json.link }}`  
  - Input: Filter output  
  - Output: Passes item with duplicated `link` field.  
  - Failures: Missing `link` attribute in RSS item causes empty value.  
  - Notes: Prepares data for downstream nodes using the link.

- **Limit**  
  - Type: Limit  
  - Role: Limits the number of items to 3.  
  - Config: `maxItems = 3`  
  - Input: Set link output  
  - Output: Only top 3 items proceed.  
  - Failures: None expected.  
  - Notes: Controls workflow execution load.

- **Sticky Note5**  
  - Type: Sticky Note  
  - Role: Instruction to set RSS feed URL and Telegram chat ID.  
  - Content:  
    "## STEP 2  
    - Set URL_FEED in \"RSS Read\" node  
    - Set YOUR_CHAT_ID in \"Telegram\" node"

---

#### 1.3 Content Scraping & Extraction

**Overview:**  
For each limited article link, scrape the webpage to extract clean markdown, then use AI models to extract the full article content for summarization.

**Nodes Involved:**  
- Loop Over Items1  
- Convert a webpage or article to clean markdown useful for blogs dev docs and more (ScrapeGraphAI)  
- Information Extractor (Langchain)  
- Google Gemini Chat Model (Langchain)  
- Sticky Note2 (instructional)

**Node Details:**

- **Loop Over Items1**  
  - Type: Split In Batches  
  - Role: Processes each article individually in batches (default batch size 1).  
  - Config: Default options (no batch size override).  
  - Input: Limit node output  
  - Output: Sends one item at a time downstream to scraping and aggregation.  
  - Failures: Batch size misconfiguration can cause delays or overload.  
  - Notes: Allows sequential processing of articles.

- **Convert a webpage or article to clean markdown useful for blogs dev docs and more**  
  - Type: ScrapeGraphAI node  
  - Role: Scrapes the article URL and converts webpage content to clean markdown format.  
  - Config:  
    - Resource: `markdownify` (clean markdown output)  
    - websiteUrl: Expression `={{ $json.link }}` (dynamic URL per article)  
  - Credentials: ScrapeGraphAI API key required.  
  - Input: Loop Over Items1 output (one article link)  
  - Output: Markdown content of the article.  
  - Failures: API key invalid, network issues, site blocking scraper, malformed page.  
  - Notes: Requires free registration with ScrapeGraphAI.

- **Google Gemini Chat Model**  
  - Type: Langchain GPT model (Google Gemini / PaLM)  
  - Role: Processes scraped markdown to extract relevant article text.  
  - Config: Default options, no special prompt.  
  - Credentials: Google Gemini API via PaLM with OAuth credentials.  
  - Input: Markdown content from ScrapeGraphAI  
  - Output: Text result for extraction.  
  - Failures: API quota limits, authentication failure, latency.  
  - Notes: Leverages Google Gemini for advanced NLP extraction.

- **Information Extractor**  
  - Type: Langchain Information Extractor  
  - Role: Extracts the complete article text from Google Gemini output using a defined system prompt.  
  - Config:  
    - Text: Expression `={{ $json.result }}` (Google Gemini output)  
    - System prompt: Guides the extractor to return only relevant content, omitting unknown attributes.  
    - Attributes: Extract only `content` (full article text).  
  - Input: Google Gemini output  
  - Output: Extracted plain article content under `output.content`  
  - Failures: Incomplete extraction if input text is malformed, extraction errors.  
  - Notes: Critical for clean, focused article content retrieval.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Instruction to scrape and extract relevant content.  
  - Content: "Scrape and extract relevant content"

---

#### 1.4 Aggregation & Summarization

**Overview:**  
Aggregate all extracted article texts into a single dataset and generate a summarized newsletter content using OpenAI GPT models, parsing structured output with subject and body.

**Nodes Involved:**  
- Aggregate  
- Generate content (Langchain Chain LLM)  
- Structured Output Parser  
- OpenAI Chat Model (GPT-5 Mini)  
- Sticky Note3 (instructional)

**Node Details:**

- **Aggregate**  
  - Type: Aggregate  
  - Role: Merges all extracted article contents (`output.content`) into a single JSON array or document for summarization.  
  - Config: Aggregates field `output.content` and renames aggregated field to `text`.  
  - Input: Loop Over Items1 output (from the extracted content)  
  - Output: Single aggregated item with all article contents.  
  - Failures: Missing extracted content causes incomplete aggregation.  
  - Notes: Prepares data for summarization stage.

- **Generate content**  
  - Type: Langchain Chain LLM  
  - Role: Uses OpenAI GPT-5 Mini to summarize and generate the email subject and body based on aggregated article texts.  
  - Config:  
    - Input text: JSON stringified aggregated `text` field  
    - Prompt: Instructs model to act as expert summarizer extracting key concepts and formatting as an email.  
    - Output parser enabled (structured output).  
  - Credentials: OpenAI API key (Eure region)  
  - Input: Aggregate output  
  - Output: Structured summary with `subject` and `body`.  
  - Failures: API quota, malformed input, prompt failure.  
  - Notes: Core AI summarization step.

- **Structured Output Parser**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses GPT output into JSON with strict schema requiring `subject` and `body` strings.  
  - Config:  
    - Schema requires `subject` and `body`  
    - AutoFix enabled to correct minor format issues  
    - Customized retry prompt enabled for robustness  
  - Input: OpenAI Chat Model1 output (GPT-4.1 Mini)  
  - Output: Parsed structured JSON for email and message.  
  - Failures: Parsing errors if GPT output malformed, retry attempts possible.  
  - Notes: Ensures reliable structured data extraction.

- **OpenAI Chat Model1**  
  - Type: Langchain Chat Model (GPT-4.1 Mini)  
  - Role: Intermediate AI model (likely for refining or formatting the output before parsing).  
  - Config: Default GPT-4.1 Mini model, no special prompt.  
  - Input: Possibly from earlier stages, feeds into output parser.  
  - Output: Feeds Structured Output Parser.  
  - Failures: API errors, latency.  
  - Notes: Used for reliable structured output generation.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Instruction about generating content for sending via email and Telegram.  
  - Content: "Generate content to send via email and Telegram"

---

#### 1.5 Delivery

**Overview:**  
Simultaneously sends the summarized news digest via Gmail email and Telegram message.

**Nodes Involved:**  
- Send a message (Gmail)  
- Send to Telegram  
- Sticky Note4 (general workflow description)  

**Node Details:**

- **Send a message**  
  - Type: Gmail  
  - Role: Sends the summarized email digest to a fixed email address.  
  - Config:  
    - Recipient: `info@n3w.it` (can be customized)  
    - Subject: `={{ $json.output.subject }}` (from structured parser output)  
    - Message body: `={{ $json.output.body }}`  
  - Credentials: OAuth2 Gmail account configured.  
  - Input: Generate content output (structured summary)  
  - Output: Sends email, no output nodes.  
  - Failures: Authentication errors, quota limits, invalid email.  
  - Notes: Requires Gmail OAuth2 credentials.

- **Send to Telegram**  
  - Type: Telegram  
  - Role: Sends the summarized digest as a Telegram message to a chat ID.  
  - Config:  
    - Chat ID: Placeholder `YOUR_CHAT_ID` to be replaced  
    - Message text: `={{ $json.output.body }}`  
  - Credentials: Telegram Bot API configured.  
  - Input: Generate content output (structured summary)  
  - Output: Sends message, no output nodes.  
  - Failures: Invalid chat ID, bot not authorized, API limits.  
  - Notes: Requires Telegram Bot credentials and chat ID.

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: General workflow description.  
  - Content:  
    "## Daily News Digest: Summarize RSS Feeds and Deliver to Email/Telegram with ScrapeGraphAI  
    This workflow provides an intelligent automation solution for **processing RSS feeds** using **ScrapeGraph API** and **delivering personalized news summaries** via email and Telegram."

---

### 3. Summary Table

| Node Name                                                               | Node Type                                    | Functional Role                                | Input Node(s)                     | Output Node(s)                                | Sticky Note                                                                                                   |
|-------------------------------------------------------------------------|----------------------------------------------|------------------------------------------------|----------------------------------|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’                                        | Manual Trigger                              | Workflow entry trigger                         | None                             | RSS Read                                      |                                                                                                               |
| RSS Read                                                               | RSS Feed Read                               | Fetches RSS feed items                         | When clicking ‘Execute workflow’ | Filter                                        |                                                                                                               |
| Filter                                                                | Filter                                      | Filters articles published within last 24h   | RSS Read                        | Set link                                      |                                                                                                               |
| Set link                                                              | Set                                         | Extracts and assigns article link             | Filter                          | Limit                                         |                                                                                                               |
| Limit                                                                | Limit                                       | Limits items to top 3                          | Set link                       | Loop Over Items1                              |                                                                                                               |
| Loop Over Items1                                                     | Split In Batches                            | Processes articles one by one                  | Limit                          | Aggregate, Convert a webpage or article to clean markdown |                                                                                                               |
| Convert a webpage or article to clean markdown useful for blogs dev docs and more | ScrapeGraphAI                               | Scrapes articles and converts to clean markdown | Loop Over Items1 (second output) | Information Extractor                          | "## STEP 1\n- Install the ScrapeGraph Community node\n- Register for FREE to [ScrapeGraphAI](https://dashboard.scrapegraphai.com/?via=n3witalia) and get API KEY" |
| Information Extractor                                               | Langchain Information Extractor              | Extracts full article text from scraped markdown | Convert a webpage or article to clean markdown | Loop Over Items1 (first output)               |                                                                                                               |
| Aggregate                                                            | Aggregate                                   | Aggregates all extracted article contents     | Loop Over Items1 (first output) | Generate content                              |                                                                                                               |
| Generate content                                                    | Langchain Chain LLM                         | Summarizes aggregated contents into digest    | Aggregate                      | Send a message, Send to Telegram               | "Generate content to send via email and Telegram"                                                             |
| Send a message                                                     | Gmail                                       | Sends email with news digest                   | Generate content                | None                                          |                                                                                                               |
| Send to Telegram                                                  | Telegram                                    | Sends Telegram message with news digest       | Generate content                | None                                          |                                                                                                               |
| OpenAI Chat Model                                                  | Langchain Chat Model (OpenAI GPT-5 Mini)   | AI model for content generation                 | Structured Output Parser        | Generate content                              |                                                                                                               |
| OpenAI Chat Model1                                                 | Langchain Chat Model (OpenAI GPT-4.1 Mini) | Intermediate AI model for structured output    |                               | Structured Output Parser                       |                                                                                                               |
| Structured Output Parser                                          | Langchain Output Parser Structured           | Parses GPT output into structured JSON          | OpenAI Chat Model1              | OpenAI Chat Model                              |                                                                                                               |
| Google Gemini Chat Model                                         | Langchain Chat Model (Google Gemini)         | AI model to process cleaned markdown content   | Convert a webpage or article to clean markdown | Information Extractor                          |                                                                                                               |
| Sticky Note                                                      | Sticky Note                                 | Instruction for ScrapeGraphAI setup            | None                           | None                                          | "## STEP 1\n- Install the ScrapeGraph Community node\n- Register for FREE to [ScrapeGraphAI](https://dashboard.scrapegraphai.com/?via=n3witalia) and get API KEY" |
| Sticky Note1                                                     | Sticky Note                                 | Instruction to filter latest RSS feed (24h)    | None                           | None                                          | "Set up and filter the latest RSS feed (24 hours)"                                                             |
| Sticky Note2                                                     | Sticky Note                                 | Instruction to scrape and extract content       | None                           | None                                          | "Scrape and extract relevant content"                                                                           |
| Sticky Note3                                                     | Sticky Note                                 | Instruction on generating content               | None                           | None                                          | "Generate content to send via email and Telegram"                                                               |
| Sticky Note4                                                     | Sticky Note                                 | Workflow description                            | None                           | None                                          | "## Daily News Digest: Summarize RSS Feeds and Deliver to Email/Telegram with ScrapeGraphAI\n\nThis workflow provides an intelligent automation solution for **processing RSS feeds** using **ScrapeGraph API** and **delivering personalized news summaries** via email and Telegram." |
| Sticky Note5                                                     | Sticky Note                                 | Instruction for feed URL and Telegram setup     | None                           | None                                          | "## STEP 2\n- Set URL_FEED in \"RSS Read\" node\n- Set YOUR_CHAT_ID in \"Telegram\" node"                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Add RSS Feed Read Node**  
   - Node Type: RSS Feed Read  
   - Parameters: Set `URL` to your RSS feed (replace `URL_FEED`).  
   - Connect Manual Trigger → RSS Read.

3. **Add Filter Node**  
   - Node Type: Filter  
   - Condition: Filter items where `pubDate` is after `={{ $today.minus({ days: 1 }) }}` to get last 24h articles.  
   - Connect RSS Read → Filter.

4. **Add Set Node for Link Extraction**  
   - Node Type: Set  
   - Assign field `link` equal to `={{ $json.link }}`.  
   - Connect Filter → Set link.

5. **Add Limit Node**  
   - Node Type: Limit  
   - Parameter: Max Items = 3 (limit to top 3 articles).  
   - Connect Set link → Limit.

6. **Add Split In Batches Node**  
   - Node Type: Split In Batches  
   - Default batch size 1 (process one article at a time).  
   - Connect Limit → Loop Over Items1.

7. **Add ScrapeGraphAI Node**  
   - Node Type: ScrapeGraphAI  
   - Parameters:  
     - Resource: `markdownify`  
     - websiteUrl: `={{ $json.link }}`  
   - Set up ScrapeGraphAI credentials with your API key.  
   - Connect Loop Over Items1 (second output) → ScrapeGraphAI.

8. **Add Google Gemini Chat Model Node**  
   - Node Type: Langchain Chat Model (Google Gemini)  
   - Configure Google Gemini credentials using Google PaLM API access.  
   - Connect ScrapeGraphAI → Google Gemini Chat Model.

9. **Add Information Extractor Node**  
   - Node Type: Langchain Information Extractor  
   - Parameters:  
     - Text: `={{ $json.result }}` (output from Gemini)  
     - System Prompt: Instruct to extract relevant content only, omit unknowns.  
     - Attributes: Extract `content` (full article text).  
   - Connect Google Gemini Chat Model → Information Extractor.

10. **Connect Information Extractor → Loop Over Items1 (first output)**  
    - This output feeds back to batch processing for aggregation.

11. **Add Aggregate Node**  
    - Node Type: Aggregate  
    - Parameters: Aggregate field `output.content` into a single field, rename to `text`.  
    - Connect Loop Over Items1 (first output) → Aggregate.

12. **Add Langchain Chain LLM Node ("Generate content")**  
    - Parameters:  
      - Text input: `={{ JSON.stringify($json.text) }}`  
      - Prompt: "You are an expert in summarizing texts and identifying main ideas. Extract the key concepts from the listed articles and present them in the form of an email."  
      - Output parser enabled.  
    - Connect Aggregate → Generate content.  
    - Configure OpenAI API credentials with GPT-5 Mini model.

13. **Add OpenAI Chat Model Node (GPT-4.1 Mini)**  
    - Node Type: Langchain Chat Model (OpenAI GPT-4.1 Mini)  
    - Connect Generate content → OpenAI Chat Model1 (if used for intermediate refinement).

14. **Add Structured Output Parser Node**  
    - Node Type: Langchain Output Parser Structured  
    - Setup JSON Schema requiring `subject` and `body` strings.  
    - Enable auto-fix and customize retry prompt.  
    - Connect OpenAI Chat Model1 → Structured Output Parser.

15. **Connect Structured Output Parser → Generate content**  
    - To finalize structured summary.

16. **Add Gmail Node to Send Email**  
    - Parameters:  
      - Send To: `info@n3w.it` (customize as needed)  
      - Subject: `={{ $json.output.subject }}`  
      - Message: `={{ $json.output.body }}`  
    - Configure Gmail OAuth2 credentials.  
    - Connect Generate content → Send a message.

17. **Add Telegram Node to Send Message**  
    - Parameters:  
      - Chat ID: Replace `YOUR_CHAT_ID` with your Telegram chat ID.  
      - Text: `={{ $json.output.body }}`  
    - Configure Telegram Bot API credentials.  
    - Connect Generate content → Send to Telegram.

18. **Add Sticky Notes for Documentation**  
    - Add notes for setup instructions and workflow overview as per original workflow to aid future users.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Install the ScrapeGraph Community node and register for FREE at ScrapeGraphAI to get your API key.             | https://dashboard.scrapegraphai.com/?via=n3witalia                                                                                                   |
| Set the `URL_FEED` in the RSS Read node and `YOUR_CHAT_ID` in the Telegram node before running the workflow.  | Workflow configuration instruction.                                                                                                                  |
| This workflow provides an intelligent automation solution for processing RSS feeds using ScrapeGraph API and delivering personalized news summaries via email and Telegram. | Overview of project purpose and integrations.                                                                                                        |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated n8n workflow built with legal, public data and strictly comply with content policies. No illegal or offensive content is included.