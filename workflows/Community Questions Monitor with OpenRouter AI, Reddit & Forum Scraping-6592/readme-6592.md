Community Questions Monitor with OpenRouter AI, Reddit & Forum Scraping

https://n8nworkflows.xyz/workflows/community-questions-monitor-with-openrouter-ai--reddit---forum-scraping-6592


# Community Questions Monitor with OpenRouter AI, Reddit & Forum Scraping

### 1. Workflow Overview

This workflow automates the monitoring and summarization of community questions and discussions from two primary sources: the n8n Community Forum and the n8n subreddit on Reddit. It runs two distinct but related pipelines daily to fetch, filter, classify, and summarize posts, then sends email digests highlighting user problems or questions.

**Target Use Cases:**  
- Community managers or support teams wanting to stay updated on user issues and discussions.  
- Automating the summarization of large volumes of forum and Reddit posts to identify key pain points.  
- Delivering concise reports to stakeholders or support staff via email.

**Logical Blocks:**  
- **1.1 Triggering and Scheduling:** Initiates the workflow either manually or on scheduled intervals (morning 9am and afternoon 5pm).  
- **1.2 n8n Forum Scraping and Processing:** Scrapes recent posts from the n8n forum, extracts relevant details, filters recent posts, uses AI to summarize user problems, and compiles an email report.  
- **1.3 Reddit Posts Retrieval and Processing:** Retrieves the latest Reddit posts from the n8n subreddit, selects and structures post data, classifies posts by type, filters for questions, summarizes with AI, and sends an email digest.  
- **1.4 AI Processing:** Uses OpenRouter AI models for text classification and summarization for both forum and Reddit content.  
- **1.5 Email Delivery:** Prepares HTML email bodies with summaries and sends emails via SMTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering and Scheduling

- **Overview:**  
  This block starts the workflows either manually via a trigger or automatically on predefined schedules to run the scraping and summarization twice daily.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Send every morning at 9am (Schedule Trigger)  
  - Send every afternoon at 5pm (Schedule Trigger)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual start of the workflow for testing or ad-hoc runs.  
    - Configuration: Default manual trigger, no parameters.  
    - Input: None  
    - Output: Connects to both forum and Reddit data fetch nodes.  
    - Failures: None expected; user-initiated.

  - **Send every morning at 9am**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow every day at 9:00 AM.  
    - Configuration: Cron expression `0 0 9 * * *` (9:00 AM daily).  
    - Input: None  
    - Output: Triggers "Get Posts sorted by replies" and "Get latest 50 reddit posts" nodes.  
    - Failures: Cron misconfiguration or n8n scheduling service issues.

  - **Send every afternoon at 5pm**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow every day at 5:00 PM.  
    - Configuration: Cron expression `0 0 17 * * *` (5:00 PM daily).  
    - Input: None  
    - Output: Triggers the same nodes as the morning schedule for data fetching.  
    - Failures: Same as above.

---

#### 2.2 n8n Forum Scraping and Processing

- **Overview:**  
  Scrapes the n8n community forum for posts, extracts post links, fetches detailed post data, filters posts from the last 14 days, summarizes user problems with AI, and sends an email summary.

- **Nodes Involved:**  
  - Get Posts sorted by replies  
  - Extract links  
  - Process Each Forum Link  
  - Fetch Individual Forum Posts  
  - Extract Post details  
  - Filter recent posts  
  - Summarize n8n Forum Posts  
  - Structured Output Parser  
  - Combine Forum Post & Summary  
  - Prepare Forum Email Body  
  - Send Forum Summary Email

- **Node Details:**

  - **Get Posts sorted by replies**  
    - Type: HTTP Request  
    - Role: Fetches the main forum page with questions sorted by replies.  
    - Configuration: URL points to n8n community questions page with max_posts=1 (likely a parameter to limit posts).  
    - Input: Trigger from schedule or manual trigger.  
    - Output: HTML content of the questions page.  
    - Failures: HTTP errors, page structure changes.

  - **Extract links**  
    - Type: HTML Extractor  
    - Role: Extracts all post links from the fetched forum page HTML.  
    - Configuration: CSS selector `.raw-topic-link`, extracting href attributes as an array.  
    - Input: HTML content from previous node.  
    - Output: Array of post URLs.  
    - Failures: Selector mismatch if forum markup changes.

  - **Process Each Forum Link**  
    - Type: SplitOut  
    - Role: Splits array of links into individual items for processing.  
    - Configuration: Field to split out is `link`.  
    - Input: Array of links.  
    - Output: One item per forum post URL.  
    - Failures: None typical.

  - **Fetch Individual Forum Posts**  
    - Type: HTTP Request  
    - Role: Fetches each forum post page individually using the links.  
    - Configuration: URL taken from current item's `link`. Batching enabled (5 posts per batch, 10 seconds interval). Retries set to 2 with 5 seconds between tries, error continues workflow on failure.  
    - Input: Individual post URLs.  
    - Output: HTML content of each post page.  
    - Failures: HTTP errors, rate limits, network timeouts.

  - **Extract Post details**  
    - Type: HTML Extractor  
    - Role: Parses each forum post page to extract relevant details: title, category, body, posted date, and link.  
    - Configuration: Uses CSS selectors for each field, cleans up text for body.  
    - Input: HTML content of individual forum posts.  
    - Output: Structured JSON with extracted fields.  
    - Failures: Selector mismatch, missing fields.

  - **Filter recent posts**  
    - Type: Filter  
    - Role: Filters posts to only those posted within the last 14 days.  
    - Configuration: Compares `posted` date field to current date minus 14 days.  
    - Input: Extracted post details.  
    - Output: Only recent posts pass through.  
    - Failures: Date parsing errors.

  - **Summarize n8n Forum Posts**  
    - Type: LangChain AI Chain LLM  
    - Role: Uses OpenRouter AI to generate a concise summary of user problems from post title and body.  
    - Configuration: Prompt instructs the AI to output JSON with a "userProblem" field, keep summary one sentence max, highlight pain points.  
    - Input: Recent post details.  
    - Output: AI-generated JSON summary.  
    - Failures: AI service errors, prompt issues.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser  
    - Role: Parses AI output to ensure JSON schema compliance with "userProblem" field.  
    - Configuration: JSON schema example supplied in parameters.  
    - Input: AI output.  
    - Output: Parsed JSON.  
    - Failures: Parsing errors if AI output deviates.

  - **Combine Forum Post & Summary**  
    - Type: Merge (combine by position)  
    - Role: Combines original post data with the AI summary by item position.  
    - Configuration: Mode set to combine, no special options.  
    - Input: Original post details and parsed summary.  
    - Output: Enriched post data with user problem summary.  
    - Failures: Mismatched lengths cause empty merges.

  - **Prepare Forum Email Body**  
    - Type: Code  
    - Role: Transforms combined data into a simplified JSON array containing relevant fields and extracted user problems, removing large AI output objects.  
    - Configuration: JavaScript code mapping data items.  
    - Input: Combined post and summary data.  
    - Output: Cleaned data array for email.  
    - Failures: Code errors if input structure changes.

  - **Send Forum Summary Email**  
    - Type: Email Send  
    - Role: Sends an HTML email summarizing forum issues, sorted by posting date.  
    - Configuration: Uses SMTP credentials, email content built with inline CSS styling and dynamic content insertion, subject "Latest n8n User Problems (Forum)", recipient and sender emails configured.  
    - Input: Prepared email body.  
    - Output: Email sent.  
    - Failures: SMTP auth errors, invalid email addresses.

---

#### 2.3 Reddit Posts Retrieval and Processing

- **Overview:**  
  This pipeline fetches the latest 50 new posts from the n8n subreddit, selects and structures relevant post data, classifies posts by category, filters for questions with zero comments, summarizes these questions with AI, and sends a summary email.

- **Nodes Involved:**  
  - Get latest 50 reddit posts  
  - Select Post Title & Body  
  - Text Classifier  
  - Filter for Questions  
  - OpenRouter Chat Model (for classification)  
  - Summarize Reddit Questions  
  - Format Reddit Summary (Output Parser)  
  - Combine Reddit Post & Summary  
  - Prepare Reddit Email Body  
  - Send Reddit Summary Email

- **Node Details:**

  - **Get latest 50 reddit posts**  
    - Type: Reddit node  
    - Role: Fetches the newest 50 posts from the configured subreddit (default: n8n).  
    - Configuration: Filter category "new", limit 50 posts. OAuth2 Reddit credentials required.  
    - Input: Trigger from schedule or manual trigger.  
    - Output: Reddit posts JSON.  
    - Failures: Reddit API errors, auth expiry, rate limiting.

  - **Select Post Title & Body**  
    - Type: Set node  
    - Role: Extracts and maps selected fields from Reddit JSON (subreddit, title, post body, creator, created date, ups, comments, and link). Converts Unix timestamp to Date format.  
    - Configuration: Assignments with expressions using `$json` references.  
    - Input: Reddit posts JSON.  
    - Output: Simplified structured JSON per post.  
    - Failures: Missing fields if Reddit API changes.

  - **Text Classifier**  
    - Type: LangChain Text Classifier  
    - Role: Classifies Reddit posts into four categories: QUESTION_HELP, JOB_POST, SELF_PROMOTION, QUESTION_GENERAL.  
    - Configuration: Uses OpenRouter AI credentials; input text concatenates title and post body with prompt descriptions for categories.  
    - Input: Selected post title and body.  
    - Output: Classification category per post.  
    - Failures: AI service errors, prompt misconfiguration.

  - **Filter for Questions**  
    - Type: Filter node  
    - Role: Filters posts that have zero comments (likely new questions).  
    - Configuration: Checks if `comments` field equals 0 (converted to number).  
    - Input: Classified posts.  
    - Output: Posts with zero comments pass.  
    - Failures: Incorrect data types, missing field.

  - **Summarize Reddit Questions**  
    - Type: LangChain AI Chain LLM  
    - Role: Summarizes user problems from Reddit post title and body with AI.  
    - Configuration: Prompt instructs AI to output concise JSON summary with one sentence max. Uses OpenRouter AI credentials.  
    - Input: Filtered question posts.  
    - Output: AI-generated JSON summary.  
    - Failures: AI service errors.

  - **Format Reddit Summary**  
    - Type: LangChain Output Parser  
    - Role: Parses AI output to JSON format according to schema `{ "userProblem": "string" }`.  
    - Configuration: JSON schema example provided.  
    - Input: AI output.  
    - Output: Parsed summary.  
    - Failures: Parsing errors if AI output is malformed.

  - **Combine Reddit Post & Summary**  
    - Type: Merge (combine by position)  
    - Role: Merges original post data with AI summaries.  
    - Configuration: Mode combine, no special options.  
    - Input: Original posts and summaries.  
    - Output: Posts enriched with user problem summaries.  
    - Failures: Data misalignment.

  - **Prepare Reddit Email Body**  
    - Type: Code  
    - Role: Cleans merged data to extract only relevant fields and userProblem summary for email.  
    - Configuration: JavaScript code mapping array and removing nested output objects.  
    - Input: Combined Reddit post and summary data.  
    - Output: Cleaned array for email.  
    - Failures: Code errors.

  - **Send Reddit Summary Email**  
    - Type: Email Send  
    - Role: Sends an HTML email summarizing Reddit questions, sorted by upvotes, with dynamic content and styling. Subject: "Latest n8n User Problems". Uses SMTP credentials.  
    - Configuration: Email addresses configured for sender and recipient, includes inline CSS styling.  
    - Input: Prepared email data.  
    - Output: Email sent.  
    - Failures: SMTP errors, invalid emails.

---

#### 2.4 AI Processing

- **Overview:**  
  This block involves AI text classification and summarization using OpenRouter models for both Reddit and forum content.

- **Nodes Involved:**  
  - OpenRouter Chat Model (for Text Classifier)  
  - OpenRouter Chat Model1 (for Summarize Reddit Questions)  
  - OpenRouter Chat Model2 (for Summarize n8n Forum Posts)  
  - Structured Output Parser (for forum)  
  - Format Reddit Summary (Output Parser for Reddit)

- **Node Details:**

  - **OpenRouter Chat Model, Chat Model1, Chat Model2**  
    - Type: LangChain OpenRouter Chat Model  
    - Role: Executes language model prompts for classification and summarization.  
    - Configuration: Uses OpenRouter API credentials. Prompts tailored per use case (classification or summarization).  
    - Input: Text data or prompt messages.  
    - Output: AI-generated text or json-like text.  
    - Failures: API key invalid, rate limits, prompt failures.

  - **Structured Output Parser & Format Reddit Summary**  
    - Type: LangChain Output Parsers  
    - Role: Parse AI output to structured JSON according to defined schemas.  
    - Configuration: JSON schema examples provided to ensure output consistency.  
    - Failures: Parsing errors if AI output is malformed or incomplete.

---

#### 2.5 Email Delivery

- **Overview:**  
  Sends formatted HTML email reports for both Reddit and forum summaries using SMTP credentials.

- **Nodes Involved:**  
  - Send Reddit Summary Email  
  - Send Forum Summary Email

- **Node Details:**

  - **Send Reddit Summary Email**  
    - Type: Email Send  
    - Role: Sends Reddit summary email with dynamic HTML content, sorted by upvotes.  
    - Configuration: SMTP credentials, from/to emails configured, subject line set.  
    - Input: Prepared email content from code node.  
    - Failures: SMTP authentication errors, invalid recipient address.

  - **Send Forum Summary Email**  
    - Type: Email Send  
    - Role: Sends forum summary email with dynamic HTML content, sorted by post date.  
    - Configuration: SMTP credentials, from/to emails configured, subject line set.  
    - Input: Prepared email content from code node.  
    - Failures: Same as above.

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                                   | Input Node(s)                         | Output Node(s)                           | Sticky Note                                                                                                           |
|-----------------------------|----------------------------------|-------------------------------------------------|-------------------------------------|-----------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                   | Manual start trigger                             | None                                | Get Posts sorted by replies, Get latest 50 reddit posts |                                                                                                                       |
| Send every morning at 9am    | Schedule Trigger                 | Scheduled trigger at 9am                         | None                                | Get Posts sorted by replies, Get latest 50 reddit posts |                                                                                                                       |
| Send every afternoon at 5pm  | Schedule Trigger                 | Scheduled trigger at 5pm                         | None                                | Get latest 50 reddit posts, Get Posts sorted by replies |                                                                                                                       |
| Get Posts sorted by replies  | HTTP Request                    | Fetches n8n forum questions page sorted by replies| When clicking ‘Test workflow’, Send every morning at 9am, Send every afternoon at 5pm| Extract links                        |                                                                                                                       |
| Extract links               | HTML Extractor                  | Extracts post links from forum page HTML        | Get Posts sorted by replies          | Process Each Forum Link                  |                                                                                                                       |
| Process Each Forum Link      | SplitOut                       | Splits array of links for individual processing | Extract links                       | Fetch Individual Forum Posts             |                                                                                                                       |
| Fetch Individual Forum Posts | HTTP Request                   | Fetches individual forum post pages             | Process Each Forum Link             | Extract Post details                     |                                                                                                                       |
| Extract Post details         | HTML Extractor                 | Extracts detailed info from forum post pages    | Fetch Individual Forum Posts         | Filter recent posts                      |                                                                                                                       |
| Filter recent posts          | Filter                        | Filters posts younger than 14 days              | Extract Post details                 | Summarize n8n Forum Posts                | "n8n Forum Digest: This flow scrapes the n8n community forum. It filters posts based on a keyword in the \"Filter for recent Posts\" node and summarizes them using AI. Remember to configure / adjust the prompt in the \"Summarize n8n Forum Posts\" node." |
| Summarize n8n Forum Posts    | LangChain Chain LLM            | Summarizes forum posts to highlight user issues| Filter recent posts                  | Combine Forum Post & Summary             |                                                                                                                       |
| Structured Output Parser     | LangChain Output Parser        | Parses AI output for forum summary               | Summarize n8n Forum Posts            | Combine Forum Post & Summary             |                                                                                                                       |
| Combine Forum Post & Summary | Merge                         | Combines forum post data with AI summary         | Summarize n8n Forum Posts, Structured Output Parser | Prepare Forum Email Body            |                                                                                                                       |
| Prepare Forum Email Body     | Code                          | Prepares cleaned data for forum summary email   | Combine Forum Post & Summary         | Send Forum Summary Email                 |                                                                                                                       |
| Send Forum Summary Email     | Email Send                    | Sends forum summary email                         | Prepare Forum Email Body             | None                                    |                                                                                                                       |
| Get latest 50 reddit posts   | Reddit Node                   | Fetches latest 50 posts from subreddit           | When clicking ‘Test workflow’, Send every morning at 9am, Send every afternoon at 5pm | Select Post Title & Body               | "Reddit Digest: Set the Subreddit name in the \"Get Latest Reddit Posts\" node. You'll also need to configure / adjust the AI prompt in the \"Summarize Reddit Questions\" node." |
| Select Post Title & Body     | Set                           | Extracts relevant post fields for Reddit posts   | Get latest 50 reddit posts           | Text Classifier                         |                                                                                                                       |
| Text Classifier             | LangChain Text Classifier      | Classifies Reddit posts into categories          | Select Post Title & Body             | Filter for Questions                    |                                                                                                                       |
| Filter for Questions         | Filter                        | Filters Reddit posts with zero comments           | Text Classifier                     | Summarize Reddit Questions              |                                                                                                                       |
| Summarize Reddit Questions   | LangChain Chain LLM            | Summarizes Reddit questions to highlight issues  | Filter for Questions                 | Combine Reddit Post & Summary            |                                                                                                                       |
| Format Reddit Summary        | LangChain Output Parser        | Parses AI output for Reddit summary               | Summarize Reddit Questions           | Combine Reddit Post & Summary            |                                                                                                                       |
| Combine Reddit Post & Summary| Merge                         | Combines Reddit posts with AI summaries           | Summarize Reddit Questions, Format Reddit Summary | Prepare Reddit Email Body           |                                                                                                                       |
| Prepare Reddit Email Body    | Code                          | Prepares cleaned data for Reddit summary email    | Combine Reddit Post & Summary        | Send Reddit Summary Email                |                                                                                                                       |
| Send Reddit Summary Email    | Email Send                    | Sends Reddit summary email                         | Prepare Reddit Email Body            | None                                    |                                                                                                                       |
| OpenRouter Chat Model        | LangChain Chat Model          | AI model for text classification                   | Select Post Title & Body            | Text Classifier                         |                                                                                                                       |
| OpenRouter Chat Model1       | LangChain Chat Model          | AI model for Reddit question summarization         | Filter for Questions                | Summarize Reddit Questions              |                                                                                                                       |
| OpenRouter Chat Model2       | LangChain Chat Model          | AI model for forum post summarization               | Filter recent posts                 | Summarize n8n Forum Posts                |                                                                                                                       |
| Structured Output Parser     | LangChain Output Parser       | Parses AI output for forum post summaries           | Summarize n8n Forum Posts           | Combine Forum Post & Summary             |                                                                                                                       |
| Sticky Note                 | Sticky Note                  | Explains workflow overview                          | None                              | None                                    | "This workflow runs two separate automations:\n\n**Reddit Digest**: Fetches, classifies, and summarizes new Reddit posts daily.\n\n**n8n Forum Digest**: Scrapes, filters, and summarizes new n8n forum posts daily.\nBoth automations send their findings in separate email reports." |
| Sticky Note1                | Sticky Note                  | Notes about Reddit Digest configuration             | None                              | None                                    | "Reddit Digest\nSet the Subreddit name in the \"Get Latest Reddit Posts\" node. You'll also need to configure / adjust the AI prompt in the \"Summarize Reddit Questions\" node." |
| Sticky Note2                | Sticky Note                  | Notes about n8n Forum Digest configuration           | None                              | None                                    | "n8n Forum Digest\nThis flow scrapes the n8n community forum. It filters posts based on a keyword in the \"Filter for recent Posts\" node and summarizes them using AI. Remember to configure / adjust the prompt in the \"Summarize n8n Forum Posts\" node." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To allow manual execution for testing.

2. **Create Two Schedule Trigger Nodes**  
   - Type: Schedule Trigger  
   - Configure one with cron `0 0 9 * * *` (9 AM daily).  
   - Configure the other with cron `0 0 17 * * *` (5 PM daily).

3. **Set up Forum Scraping Pipeline:**  
   a. Create an HTTP Request node named "Get Posts sorted by replies".  
      - URL: `https://community.n8n.io/c/questions/12?max_posts=1` (adjust max_posts as needed).  
      - Connect from manual and both schedule triggers.

   b. Create an HTML Extractor node "Extract links".  
      - Operation: Extract HTML Content.  
      - Extraction: Use CSS selector `.raw-topic-link`, attribute `href`, return as array.  
      - Connect from "Get Posts sorted by replies".

   c. Create a SplitOut node "Process Each Forum Link".  
      - Field to split out: `link`.  
      - Connect from "Extract links".

   d. Create an HTTP Request node "Fetch Individual Forum Posts".  
      - URL: Expression `{{$json["link"]}}`.  
      - Enable batching: batch size 5, batch interval 10000ms.  
      - Set max retries to 2, wait 5000ms between tries, continue on error.  
      - Connect from "Process Each Forum Link".

   e. Create an HTML Extractor node "Extract Post details".  
      - Extract fields:  
        - title: `#topic-title a`  
        - category: `.topic-category .category-name`  
        - body: `.topic-body .post` (clean up text enabled)  
        - posted: meta tag `datePublished` content attribute  
        - link: `link[itemprop="url"]` href attribute  
      - Connect from "Fetch Individual Forum Posts".

   f. Create a Filter node "Filter recent posts".  
      - Condition: Posted date is after current time minus 14 days.  
      - Use expression: `{{$json.posted}}` > `{{$now.minus(14, 'days')}}`.  
      - Connect from "Extract Post details".

   g. Create LangChain Chain LLM node "Summarize n8n Forum Posts".  
      - Prompt: Provide purpose and instructions to summarize user problems in JSON, one sentence max.  
      - Input text: Title and body.  
      - Use OpenRouter API credentials.  
      - Connect from "Filter recent posts".

   h. Add LangChain Output Parser node "Structured Output Parser".  
      - JSON schema example: `{ "userProblem": "string" }`.  
      - Connect from "Summarize n8n Forum Posts".

   i. Add Merge node "Combine Forum Post & Summary".  
      - Mode: Combine by position.  
      - Connect first input: "Summarize n8n Forum Posts" output.  
      - Connect second input: "Structured Output Parser" output.

   j. Add Code node "Prepare Forum Email Body".  
      - JavaScript code to map data, flatten, and remove nested output objects.  
      - Connect from "Combine Forum Post & Summary".

   k. Add Email Send node "Send Forum Summary Email".  
      - Configure SMTP credentials.  
      - Set from and to emails.  
      - Compose HTML email body with inline CSS; include issue title, problem summary, post date, and link.  
      - Subject: "Latest n8n User Problems (Forum)".  
      - Connect from "Prepare Forum Email Body".

4. **Set up Reddit Posts Pipeline:**  
   a. Create Reddit node "Get latest 50 reddit posts".  
      - Operation: getAll, category: new, limit 50 posts.  
      - Use Reddit OAuth2 credentials.  
      - Connect from manual and both schedule triggers.

   b. Create Set node "Select Post Title & Body".  
      - Map subreddit, title, post (selftext), creator (name), createdDate (convert from UNIX timestamp), ups, comments, link.  
      - Connect from "Get latest 50 reddit posts".

   c. Create LangChain Chat Model node "OpenRouter Chat Model" (for classification).  
      - Use OpenRouter API credentials.  
      - Connect from "Select Post Title & Body".

   d. Create LangChain Text Classifier node "Text Classifier".  
      - Categories defined: QUESTION_HELP, JOB_POST, SELF_PROMOTION, QUESTION_GENERAL with descriptions.  
      - Input text: Concatenate title and post.  
      - Connect from "OpenRouter Chat Model".

   e. Create Filter node "Filter for Questions".  
      - Condition: Number of comments equals 0.  
      - Connect from "Text Classifier".

   f. Create LangChain Chat Model node "OpenRouter Chat Model1" (for summarization).  
      - Use OpenRouter API credentials.  
      - Connect from "Filter for Questions".

   g. Create LangChain Chain LLM node "Summarize Reddit Questions".  
      - Prompt to summarize user problems in JSON, concise one-sentence max.  
      - Connect from "OpenRouter Chat Model1".

   h. Add LangChain Output Parser node "Format Reddit Summary".  
      - JSON schema example matching userProblem field.  
      - Connect from "Summarize Reddit Questions".

   i. Add Merge node "Combine Reddit Post & Summary".  
      - Combine by position.  
      - Connect from "Summarize Reddit Questions" and "Format Reddit Summary".

   j. Add Code node "Prepare Reddit Email Body".  
      - JavaScript code to map and flatten data, remove nested output objects.  
      - Connect from "Combine Reddit Post & Summary".

   k. Add Email Send node "Send Reddit Summary Email".  
      - Configure SMTP credentials, from/to emails.  
      - Compose HTML email with inline CSS, sorted by upvotes descending.  
      - Subject: "Latest n8n User Problems".  
      - Connect from "Prepare Reddit Email Body".

5. **Connect Triggers to Both Pipelines:**  
   - From manual trigger and both schedule triggers to:  
     - "Get Posts sorted by replies" (Forum pipeline)  
     - "Get latest 50 reddit posts" (Reddit pipeline)

6. **Configure Credentials:**  
   - Reddit OAuth2 API credentials for Reddit node.  
   - OpenRouter API credentials for all LangChain nodes using OpenRouter.  
   - SMTP credentials for both Email Send nodes.

7. **Test Workflow:**  
   - Run manually first to verify data extraction, classification, summarization, and email delivery.  
   - Adjust prompts if summaries are inaccurate or incomplete.  
   - Monitor for HTTP or API errors and adjust retry parameters if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow runs two separate automations: Reddit Digest (fetches, classifies, summarizes new Reddit posts daily) and n8n Forum Digest (scrapes, filters, summarizes forum posts). Both send separate email reports. | Sticky Note covering general workflow overview.                                                              |
| Reddit Digest: Set the Subreddit name in the "Get Latest Reddit Posts" node. Adjust the AI prompt in the "Summarize Reddit Questions" node as needed to tune summaries.           | Sticky Note attached near Reddit-related nodes.                                                               |
| n8n Forum Digest: This flow scrapes the n8n community forum. It filters posts based on recency and summarizes them using AI. Adjust the prompt in the "Summarize n8n Forum Posts" node. | Sticky Note attached near forum scraping nodes.                                                               |
| Ensure SMTP credentials are valid and tested to avoid email delivery failures.                                                                                                   | General operational note for email sending nodes.                                                             |
| OpenRouter API keys require active subscription and valid usage limits to avoid rate-limiting or failures.                                                                        | General operational note for AI nodes.                                                                         |
| Reddit OAuth2 credentials must be kept secure and refreshed as needed to maintain uninterrupted API access.                                                                       | General operational note for Reddit integration.                                                              |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.