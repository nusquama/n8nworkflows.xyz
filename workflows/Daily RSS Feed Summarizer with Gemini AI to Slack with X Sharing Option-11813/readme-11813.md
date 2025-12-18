Daily RSS Feed Summarizer with Gemini AI to Slack with X Sharing Option

https://n8nworkflows.xyz/workflows/daily-rss-feed-summarizer-with-gemini-ai-to-slack-with-x-sharing-option-11813


# Daily RSS Feed Summarizer with Gemini AI to Slack with X Sharing Option

### 1. Workflow Overview

This workflow automates the daily retrieval and summarization of news articles from specified RSS feeds, leveraging Gemini AI to generate concise and engaging summaries tailored for posting on X (formerly Twitter). The summarized content is then sent as formatted messages to a Slack channel, including a direct "Share on X" button for quick posting. The workflow is designed for social media managers or teams who want to monitor technology news feeds efficiently and share curated content on social media platforms.

The workflow logic is structured into these functional blocks:

- **1.1 Scheduling and Configuration**: Daily trigger and setup of RSS feed URLs, target date, and item limits.  
- **1.2 RSS Feed Retrieval and Filtering**: Fetch RSS items, filter by date, and select top items per feed.  
- **1.3 Iterative Processing of RSS Items**: Loop over each filtered RSS item to process individually.  
- **1.4 AI-Powered Article Summarization**: For each RSS item, fetch full article HTML, call Gemini AI to generate a title and description optimized for X, and parse the AI response.  
- **1.5 Slack Notification and Sharing**: Format a Slack message with the AI-generated summary and a button linking to share the post on X, then send it to Slack.  
- **1.6 Sub-Workflow Invocation**: The main workflow calls itself as a sub-workflow for each RSS item to isolate processing and ensure modularity.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Configuration

**Overview:**  
This block initiates the workflow daily at a specified hour and sets up key parameters such as the RSS feed URLs, the target date (yesterday), and how many articles to process per feed.

**Nodes Involved:**  
- Schedule Trigger  
- Config  
- Convert to loop items

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule trigger node  
  - Configured to trigger daily at 2 AM server time  
  - Input: None  
  - Output: Triggers the workflow flow downstream  
  - Edge cases: Timezone discrepancies may affect execution time; server downtime can cause missed triggers.

- **Config**  
  - Type: Set node  
  - Assigns three parameters:  
    - `targetDate`: Yesterday’s date in `yyyy-MM-dd` format using n8n date functions  
    - `rssUrls`: An array of RSS feed URLs to fetch (Google Technology News and CNBC Technology RSS)  
    - `takeCount`: Number of articles to process per feed (default "1")  
  - Edge cases: The RSS URLs must be valid; otherwise, subsequent fetch will fail. `takeCount` should be numeric; string input may cause issues unless handled properly.

- **Convert to loop items**  
  - Type: Code node  
  - Converts the array of RSS URLs into separate items for iteration by mapping each URL to a JSON object with a `url` property  
  - Input: JSON object with `rssUrls` array from Config  
  - Output: One item per RSS URL for loop processing  
  - Edge cases: If `rssUrls` is empty or malformed, the output will be empty, halting further processing.

---

#### 2.2 RSS Feed Retrieval and Filtering

**Overview:**  
This block retrieves RSS feed data from each URL, filters articles by date (only those on or after the target date), and limits the number of articles processed per feed.

**Nodes Involved:**  
- Loop Over Items (batch splitter)  
- RSS Read  
- Filter Rss Feeds  
- Is not empty (If node)

**Node Details:**  

- **Loop Over Items**  
  - Type: SplitInBatches node  
  - Processes each RSS URL one by one to avoid parallel overloads  
  - Input: Items from "Convert to loop items"  
  - Output: Passes each URL to "RSS Read" node  
  - Edge cases: Batch size defaults to 1, so no concurrency issues; large URLs array may increase total runtime.

- **RSS Read**  
  - Type: RSS Feed Read node  
  - Fetches RSS feed content from the URL in the JSON `url` property  
  - Input: Single RSS URL item  
  - Output: List of RSS feed items (articles)  
  - Edge cases: Connection failures, invalid or unreachable URLs, malformed RSS XML, or empty feeds.

- **Filter Rss Feeds**  
  - Type: Code node  
  - Filters fetched RSS items based on comparing each item's `isoDate` with `targetDate` from Config  
  - Limits output to top `takeCount` items  
  - Input: Array of RSS items  
  - Output: Filtered, limited list of RSS items  
  - Edge cases: Items without valid or missing dates will be excluded or cause parsing errors.

- **Is not empty**  
  - Type: If node  
  - Checks if the filtered RSS feed items array is not empty before proceeding  
  - Directs flow to either sub-workflow invocation or back to loop over items  
  - Edge cases: Prevents processing when no new articles are found, avoiding unnecessary AI calls and Slack notifications.

---

#### 2.3 Iterative Processing of RSS Items

**Overview:**  
This block manages the iteration over each RSS article item to process them individually through the AI summarization and Slack notification steps.

**Nodes Involved:**  
- Call Sub-workflow

**Node Details:**  

- **Call Sub-workflow**  
  - Type: Execute Workflow node  
  - Invokes the same workflow as a sub-workflow, passing the individual RSS feed item as input  
  - Configured to wait for sub-workflow completion before processing next item  
  - Input: Single RSS feed item JSON from "Is not empty" node  
  - Output: Receives processed output from sub-workflow for Slack messaging  
  - Edge cases: Sub-workflow failures or timeouts will affect main workflow execution; ensure robust error handling in the sub-workflow.

---

#### 2.4 AI-Powered Article Summarization (Sub-Workflow)

**Overview:**  
This block fetches the full article content from the RSS item's link, uses Gemini AI to generate an engaging title and description suited for X, and formats the AI response.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Format Request  
- Google Gemini Chat Model  
- HttpRequestTool  
- AI Agent (Access URL)  
- Format Response

**Node Details:**  

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger node  
  - Entry point for sub-workflow when invoked by main workflow  
  - Accepts `rssFeed` input object (the RSS item)  
  - Output: Passes RSS feed item downstream  
  - Edge cases: If input is missing or malformed, subsequent nodes may fail.

- **Format Request**  
  - Type: Code node  
  - Extracts and structures request parameters for AI processing, including the article link  
  - Input: `rssFeed` object  
  - Output: JSON object with `requestParameters` and `link` fields  
  - Edge cases: Missing or invalid `link` field could cause HTTP fetch failure.

- **Google Gemini Chat Model**  
  - Type: Language model node integrating Google Gemini AI  
  - Uses configured Google API credentials for authentication  
  - Receives prompt from "AI Agent (Access URL)" node to generate content  
  - Edge cases: API key invalid/expired, request timeouts, or quota limits.

- **HttpRequestTool**  
  - Type: HTTP Request Tool node  
  - Fetches full HTML content of the article URL with custom User-Agent header for better server acceptance  
  - Timeout set to 60 seconds, allows redirects, and ignores unauthorized SSL certs  
  - Output: Full HTTP response including body for AI processing  
  - Edge cases: Network errors, redirects loops, invalid certificates, or slow responses.

- **AI Agent (Access URL)**  
  - Type: LangChain Agent node (custom AI orchestration)  
  - Uses the HttpRequestTool to retrieve article content and feeds it to Google Gemini Chat Model  
  - Prompt instructs AI to create a catchy title and X post description under 200 characters, outputting strictly JSON code block  
  - Configured to continue on error with retries and 5-second wait between attempts  
  - Edge cases: AI output parsing errors, unexpected response format, or service interruptions.

- **Format Response**  
  - Type: Code node  
  - Parses the AI JSON response, extracts title and description, constructs the X sharing URL, and formats Slack message content including links and buttons  
  - Input: AI output and original request parameters  
  - Output: JSON with combined fields for Slack notification and sharing URL  
  - Edge cases: JSON parse failures if AI output is malformed, missing fields, or unexpected text.

---

#### 2.5 Slack Notification and Sharing

**Overview:**  
This block sends the AI-generated summary to a Slack channel as a rich block message, including a button to share the post on X.

**Nodes Involved:**  
- Send a message  
- Loop Over Items (for messages)

**Node Details:**  

- **Send a message**  
  - Type: Slack node  
  - Sends a block message to a specific Slack channel using Slack API credentials  
  - Message includes:  
    - The AI-generated summary as markdown text  
    - A divider block  
    - An action block with a button labeled "Share on X" linking to the generated X sharing URL  
  - Configured with Slack API OAuth2 credentials  
  - Edge cases: Slack API rate limiting, invalid channel ID, missing permissions, or expired tokens.

- **Loop Over Items** (message phase)  
  - Type: SplitInBatches node  
  - Ensures each Slack message is sent individually per processed article  
  - Edge cases: Large batches could cause delays or trigger rate limits.

---

#### 2.6 Sticky Notes (Documentation Nodes)

**Overview:**  
Informational notes embedded in the workflow for user guidance and overview.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**  

- **Sticky Note**  
  - Contains detailed instructions on workflow purpose, setup steps for Gemini AI and Slack credentials, and customization tips  
  - Positioned near the start for onboarding users

- **Sticky Note1**  
  - Summarizes the main workflow logic: fetching RSS and notifying Slack, calling sub-workflow for AI processing

- **Sticky Note2**  
  - Describes the sub-workflow role: AI summarization and formatting for X posting

---

### 3. Summary Table

| Node Name                | Node Type                      | Functional Role                                    | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                          |
|--------------------------|--------------------------------|---------------------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger               | Daily trigger to start workflow                    | —                      | Config                  | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| Config                   | Set                           | Sets targetDate, rssUrls, and takeCount parameters| Schedule Trigger       | Convert to loop items    | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| Convert to loop items     | Code                          | Converts rssUrls array into individual items       | Config                 | Loop Over Items          | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| Loop Over Items           | SplitInBatches                | Iterates over each RSS URL                          | Convert to loop items   | RSS Read                 | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| RSS Read                 | RSS Feed Read                 | Fetches RSS articles from feed URL                  | Loop Over Items        | Filter Rss Feeds         | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| Filter Rss Feeds          | Code                          | Filters RSS items by targetDate and takeCount      | RSS Read               | Is not empty             | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| Is not empty              | If                            | Checks if filtered RSS items exist                  | Filter Rss Feeds       | Call Sub-workflow / Loop Over Items | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| Call Sub-workflow         | Execute Workflow              | Calls sub-workflow for AI summarization             | Is not empty           | Send a message           | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| Send a message            | Slack                         | Sends formatted message with AI summary to Slack   | Call Sub-workflow      | Loop Over Items          | "Fetch RSS and Notify Slack" main workflow overview                                                                 |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point for sub-workflow                        | —                      | Format Request           | "Summarize Articles with AI" sub-workflow overview                                                                   |
| Format Request            | Code                          | Prepares input data for AI processing                | When Executed by Another Workflow | AI Agent (Access URL)    | "Summarize Articles with AI" sub-workflow overview                                                                   |
| Google Gemini Chat Model  | LM Chat (Google Gemini)       | Connects to Google Gemini AI for language model     | AI Agent (Access URL)   | AI Agent (Access URL)    | "Summarize Articles with AI" sub-workflow overview                                                                   |
| HttpRequestTool           | HTTP Request Tool             | Fetches full article HTML content                    | AI Agent (Access URL) (as AI tool) | AI Agent (Access URL)    | "Summarize Articles with AI" sub-workflow overview                                                                   |
| AI Agent (Access URL)     | LangChain Agent               | Orchestrates article fetch and AI summarization     | Format Request, HttpRequestTool, Google Gemini Chat Model | Format Response          | "Summarize Articles with AI" sub-workflow overview                                                                   |
| Format Response           | Code                          | Parses AI JSON output, builds Slack message and X share URL | AI Agent (Access URL)   | Call Sub-workflow        | "Summarize Articles with AI" sub-workflow overview                                                                   |
| Sticky Note               | Sticky Note                   | Documentation and usage instructions                 | —                      | —                       | "How it works" detailed instructions                                                                                 |
| Sticky Note1              | Sticky Note                   | Main workflow overview                               | —                      | —                       | Overview of RSS fetching and Slack notification                                                                       |
| Sticky Note2              | Sticky Note                   | Sub-workflow overview                                | —                      | —                       | Overview of AI summarization and formatting                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** and name it accordingly.

2. **Add a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 2:00 AM (server time).

3. **Add a Set node named "Config":**  
   - Add three parameters:  
     - `targetDate` (string): Expression `{{$now.minus({ days: 1 }).toFormat('yyyy-MM-dd')}}`  
     - `rssUrls` (array): Add your RSS feed URLs, e.g.  
       `['https://news.google.com/rss/headlines/section/topic/TECHNOLOGY?hl=en-US&gl=US&ceid=US:en', 'https://search.cnbc.com/rs/search/combinedcms/view.xml?partnerId=wrss01&id=19854910']`  
     - `takeCount` (string): e.g., "1"

4. **Add a Code node named "Convert to loop items":**  
   - JavaScript code:  
     ```js
     return $json.rssUrls.map((item) => ({
       json: { url: item }
     }));
     ```

5. **Add a SplitInBatches node named "Loop Over Items":**  
   - Defaults are fine (batch size = 1).

6. **Add an RSS Feed Read node named "RSS Read":**  
   - Set `URL` to `={{ $json.url }}`  
   - Leave other options default.

7. **Add a Code node named "Filter Rss Feeds":**  
   - JavaScript code:  
     ```js
     const config = $('Config').first().json;
     const targetDate = new Date(config.targetDate);
     return $input.all().filter(item => {
       const date = new Date(item.json.isoDate);
       return date >= targetDate;
     }).slice(0, config.takeCount);
     ```

8. **Add an If node named "Is not empty":**  
   - Condition: Check if `$json` is not empty (object not empty)  
   - On true: proceed to next step  
   - On false: loop back to "Loop Over Items" for next URL

9. **Add an Execute Workflow node named "Call Sub-workflow":**  
   - Set mode to "each" with "wait for sub-workflow" enabled  
   - Select the same workflow (self-call) to process each RSS item individually  
   - Map the current RSS feed item to a parameter named `rssFeed`

10. **Add a Slack node named "Send a message":**  
    - Configure Slack API OAuth2 credentials  
    - Set mode to send block message type to your target channel  
    - Use the JSON from the sub-workflow output to populate the message blocks, including the "Share on X" button with URL

11. **Add a SplitInBatches node named "Loop Over Items" (for Slack messages):**  
    - To iterate sending messages per article

---

#### Sub-Workflow Setup (Self-Reference)

12. **Add an Execute Workflow Trigger node named "When Executed by Another Workflow":**  
    - Accept input parameter `rssFeed` (object)

13. **Add a Code node named "Format Request":**  
    - JavaScript to extract link and prepare request object:  
      ```js
      return {
        json: {
          requestParameters: $input.item.json,
          link: $input.item.json.rssFeed.link,
        }
      };
      ```

14. **Add a LangChain Agent node named "AI Agent (Access URL)":**  
    - Configure prompt to:  
      - Fetch article HTML content using HttpRequestTool with `link` URL  
      - Use Google Gemini Chat Model to generate title and description (under 200 chars) in JSON format  
    - Configure retry on failure with a 5-second wait  
    - Connect Google Gemini Chat Model node as language model credentials  
    - Configure HttpRequestTool node to fetch article HTML with User-Agent header, timeout 60s, allow redirects

15. **Add a Code node named "Format Response":**  
    - Parse AI output JSON, build Slack message with markdown text, and generate X share URL  
    - Example code:  
      ```js
      const request = $('Format Request').first().json;
      const aiOutput = $input.first().json.output;
      const cleanJson = aiOutput.replace(/```json|```/g, '').trim();
      const result = JSON.parse(cleanJson);
      const articleUrl = request.requestParameters?.rssFeed?.link || request.link || "";
      const tweetText = `${result.title}\n${result.description}`;
      const xShareUrl = `https://x.com/intent/tweet?text=${encodeURIComponent(tweetText)}&url=${encodeURIComponent(articleUrl)}`;
      const slackMessage = `📢 *New Article Draft Ready*\n\nI have generated a draft for X based on the RSS feed.\n\n*Title:* ${result.title}\n*Content:* ${result.description}\n\n<${articleUrl}|📄 Read Original Article>`;
      return [{
        json: {
          ...request,
          ...result,
          xShareUrl,
          slackMessage
        }
      }];
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                         | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Obtain Gemini AI API key from Google AI Studio at [https://aistudio.google.com/api-keys](https://aistudio.google.com/api-keys)                                                                       | Credential setup for Google Gemini Chat Model node                                               |
| Create a Slack App and OAuth2 credentials at [https://api.slack.com/apps](https://api.slack.com/apps)                                                                                                 | Required for Slack API node authentication                                                      |
| Customize `rssUrls` and `takeCount` parameters in the Config node to control source feeds and number of articles processed                                                                           | User customization                                                                               |
| The workflow includes retry and error handling for AI calls, but network, API quota, and Slack rate limits should be monitored for production use                                                   | Operational considerations                                                                      |
| The Slack message uses blocks UI with markdown and a button that links to X sharing URL, enabling quick social media posting                                                                         | Enhances usability                                                                                |
| The workflow architecture uses a main workflow invoking itself as sub-workflow for modular processing and better error isolation                                                                     | Best practice for complex or long-running item processing                                       |

---

**Disclaimer:** The provided content derives exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and publicly available.