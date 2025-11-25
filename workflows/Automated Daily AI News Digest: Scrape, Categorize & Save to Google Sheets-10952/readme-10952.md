Automated Daily AI News Digest: Scrape, Categorize & Save to Google Sheets

https://n8nworkflows.xyz/workflows/automated-daily-ai-news-digest--scrape--categorize---save-to-google-sheets-10952


# Automated Daily AI News Digest: Scrape, Categorize & Save to Google Sheets

### 1. Workflow Overview

This workflow automates the processing of daily AI news digest emails, specifically designed to extract, summarize, categorize, and archive AI-related news articles into a Google Sheet. It is targeted at users who receive daily newsletters (e.g., from AlphaSignal) and want to maintain a structured, searchable archive with categorized summaries and concise links.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Conversion**: Fetch incoming emails containing the AI newsletter and convert the HTML content into Markdown for easier text processing.
- **1.2 Article Extraction and Summarization**: Use AI agents and scraping tools to split the newsletter into individual articles, summarize each article in Italian, and validate the structured JSON output.
- **1.3 Categorization and Archiving**: Categorize each summarized article using OpenAI, shorten article URLs, and append all data (title, summary, category, short link, date) to a Google Sheet for daily tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Conversion

**Overview:**  
This block retrieves the daily AI news email from Gmail, extracts the full HTML content, and converts it into Markdown format. This prepares the text for AI-based article segmentation.

**Nodes Involved:**  
- Get email  
- Convert HTML to MD

**Node Details:**

- **Get email**  
  - Type: Gmail node (OAuth2)  
  - Role: Fetches a specific email by message ID from the Gmail inbox.  
  - Configuration: Uses the incoming message ID dynamically (`{{$json.id}}`) to retrieve the full email content.  
  - Credentials: Gmail OAuth2 (named "Gmail account (n3w.it)").  
  - Input: Triggered externally or by webhook ID.  
  - Output: Complete email JSON, including `textAsHtml`.  
  - Edge cases: Email not found (invalid ID), OAuth token expiration, quota exceeded.  
  - Version-specific: v2.1, supports Gmail OAuth.

- **Convert HTML to MD**  
  - Type: Markdown node  
  - Role: Converts the email's HTML body (`$json.textAsHtml`) to Markdown format stored in the key `md`.  
  - Configuration: HTML source is dynamically referenced.  
  - Input: Receives email JSON output.  
  - Output: JSON with Markdown text.  
  - Edge cases: Malformed HTML, conversion failures.

---

#### 2.2 Article Extraction and Summarization

**Overview:**  
This block uses an AI-based agent powered by Google Gemini and ScrapegraphAI to parse the Markdown newsletter content, split it into individual articles, and generate Italian summaries. It then validates and structures the output as JSON for further processing.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Scrape  
- Scrape Agent  
- Validate Json  
- Loop Over Items

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini chat node  
  - Role: Provides AI chat functionality as part of the agent’s reasoning pipeline.  
  - Configuration: Uses Google Palm API credentials.  
  - Input: Receives prompts within the Scrape Agent node.  
  - Output: AI-generated text used for article extraction.  
  - Edge cases: API quota, network errors, prompt parsing issues.

- **Scrape**  
  - Type: ScrapegraphAI tool node  
  - Role: Given an article URL, fetches and summarizes the article in Italian with main concepts.  
  - Configuration: Reads URL dynamically from AI prompt input (`Website_URL`), uses ScrapegraphAI API credentials.  
  - Input: URLs extracted by the agent.  
  - Output: Article summary text.  
  - Edge cases: URL inaccessible, rate limits, malformed responses.

- **Scrape Agent**  
  - Type: LangChain agent node  
  - Role: Core logic to split newsletter Markdown text into individual articles, extract title, link, and summary fields, and call the "Scrape" tool for detailed article analysis.  
  - Configuration: Custom system message instructs splitting the newsletter and summarizing in Italian.  
  - Input: Markdown text from previous node.  
  - Output: JSON string with article arrays.  
  - Edge cases: Parsing errors, invalid JSON output, AI misinterpretation.

- **Validate Json**  
  - Type: Code node (JavaScript)  
  - Role: Cleans the AI response from code block markers, parses JSON, and outputs one item per article with relevant fields (title, link, summary).  
  - Configuration: Custom JS code to sanitize and parse AI output.  
  - Input: Raw text JSON string from Scrape Agent.  
  - Output: Array of article objects as separate items.  
  - Edge cases: JSON parse errors, unexpected AI output formatting.

- **Loop Over Items**  
  - Type: SplitInBatches node  
  - Role: Iterates over each article item to process them individually in subsequent nodes.  
  - Configuration: Default batch options.  
  - Input: List of article JSON items.  
  - Output: Single article items sequentially.  
  - Edge cases: Large batch sizes causing delays, empty input array.

---

#### 2.3 Categorization and Archiving

**Overview:**  
For each article, this block categorizes the content into a predefined category using OpenAI, generates a short URL for the article link, and saves the enriched data into a Google Sheet for archival.

**Nodes Involved:**  
- OpenAI Chat Model  
- Categorization Chain  
- Short url  
- Add article to sheet

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI chat node  
  - Role: Provides AI language model capabilities for article categorization.  
  - Configuration: Uses "gpt-5-mini" model, OpenAI API credentials.  
  - Input: Article title and summary from Loop Over Items.  
  - Output: Categorization prompt result.  
  - Edge cases: API rate limits, prompt failures.

- **Categorization Chain**  
  - Type: LangChain LLM chain node  
  - Role: Defines the prompt instructing the model to assign exactly one category from a fixed list based on title and summary.  
  - Configuration: Hardcoded categories including "LLM & Foundation Models", "Developer Tools & Code Generation", etc.  
  - Input: Title and summary from Loop Over Items, optionally chained from OpenAI Chat Model.  
  - Output: Category text string.  
  - Edge cases: Ambiguous content, multiple applicable categories, misclassification.

- **Short url**  
  - Type: HTTP Request node  
  - Role: Calls the CleanURI API to shorten the original article URL for concise storage.  
  - Configuration: POST request to `https://cleanuri.com/api/v1/shorten` with form-urlencoded body parameter `url` dynamically set from the current article’s link.  
  - Input: Article link from Loop Over Items.  
  - Output: JSON containing shortened URL in `result_url`.  
  - Edge cases: API downtime, invalid URL, network errors.

- **Add article to sheet**  
  - Type: Google Sheets node (OAuth2)  
  - Role: Appends a new row to a specified Google Sheet with the article date, short link, title, summary, and category.  
  - Configuration: Uses Google Sheets OAuth2 credentials.  
  - Sheet ID: `1CvsUadsmoLKZUUYd2KEEGMnJidqRqnLW-BvIw66_LX4` (archive sheet).  
  - Columns mapped: DATE (today’s date), LINK (shortened URL), TITLE, SUMMARY, CATEGORY.  
  - Input: Data from Short url and Categorization Chain nodes.  
  - Output: Confirmation of append operation.  
  - Edge cases: Sheet access errors, quota limits, invalid data types.

---

### 3. Summary Table

| Node Name            | Node Type                        | Functional Role                             | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                     |
|----------------------|---------------------------------|---------------------------------------------|-----------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------|
| Get email            | Gmail                           | Fetch incoming AI newsletter email          | External trigger/webhook | Convert HTML to MD         | "STEP 1 - Get email and covert to MD\nGet the email and convert text from HTML to Markdown. **Email Credentials:** Set up a Gmail OAuth2 credential (named \"Gmail account\" in the workflow) to allow n8n to access and read emails from the specified inbox." |
| Convert HTML to MD    | Markdown                       | Convert email HTML to Markdown               | Get email             | Scrape Agent              | Same as above                                                                                                    |
| Google Gemini Chat Model | LangChain Google Gemini Chat | AI Chat for article extraction and summarization | Scrape Agent (AI tool) | Scrape Agent (AI tool)    | "STEP 2 - Split the text nto individual articles\nSplit the text of this newsletter into individual articles and output a valid JSON.   **Scraping Tool:** Set up the [ScrapegraphAI account](https://dashboard.scrapegraphai.com/?via=n3witalia) credential with its required API key to enable the agent to access and scrape content from the article URLs." |
| Scrape               | ScrapegraphAI                   | Scrape and summarize each article URL       | Scrape Agent (ai_tool) | Scrape Agent (ai_tool)    | Same as above                                                                                                    |
| Scrape Agent          | LangChain Agent                 | Split newsletter Markdown into articles, summarize | Convert HTML to MD     | Validate Json             | Same as above                                                                                                    |
| Validate Json        | Code                           | Clean and parse AI JSON output                | Scrape Agent          | Loop Over Items           |                                                                                                                 |
| Loop Over Items       | SplitInBatches                 | Iterate over article items                    | Validate Json         | Categorization Chain, Add article to sheet (via short url) | "STEP 3 - Categorization article and save to Google Sheet\nFor each article coming from the JSON, I categorize it using the Categorization Chain, generate a short URL from the newsletter link, and finally insert the article’s content into the [Google Sheets archive](https://docs.google.com/spreadsheets/d/15VKWMbd2cOb74uwYncfXkwLZam-dXD5sBiAttVpKIQw/edit?usp=sharing)." |
| OpenAI Chat Model     | LangChain OpenAI Chat          | Provide chat model for categorization        | Loop Over Items       | Categorization Chain (ai_languageModel) | Same as above                                                                                                    |
| Categorization Chain  | LangChain Chain LLM            | Categorize article based on title and summary | Loop Over Items, OpenAI Chat Model | Short url               | Same as above                                                                                                    |
| Short url             | HTTP Request                   | Shorten article URL using CleanURI API       | Categorization Chain  | Add article to sheet      | Same as above                                                                                                    |
| Add article to sheet  | Google Sheets                  | Append article data to Google Sheet archive  | Short url             | Loop Over Items           | Same as above                                                                                                    |
| Sticky Note           | Sticky Note                   | Documentation and overview notes              | None                  | None                      | "Automated Daily AI News Digest: Scrape, Categorize & Save to Google Sheets\nThis workflow is designed to automatically process AI news emails..." |
| Sticky Note3          | Sticky Note                   | Step 1 description                            | None                  | None                      | "STEP 1 - Get email and covert to MD\nGet the email and convert text from HTML to Markdown. **Email Credentials:** Set up a Gmail OAuth2 credential ..." |
| Sticky Note4          | Sticky Note                   | Step 2 description                            | None                  | None                      | "STEP 2 - Split the text nto individual articles\nSplit the text of this newsletter into individual articles and output a valid JSON.   **Scraping Tool:** ..." |
| Sticky Note6          | Sticky Note                   | Step 3 description                            | None                  | None                      | "STEP 3 - Categorization article and save to Google Sheet\nFor each article coming from the JSON, I categorize it using the Categorization Chain, generate a short URL from the newsletter link, and finally insert the article’s content into the Google Sheets archive." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Node "Get email"**  
   - Type: Gmail  
   - Operation: Get email by ID  
   - Parameters: Set messageId to `{{$json.id}}` (dynamic)  
   - Credentials: Configure Gmail OAuth2 ("Gmail account (n3w.it)")  
   - Trigger: Setup webhook or external trigger that provides message IDs.

2. **Add Markdown Node "Convert HTML to MD"**  
   - Convert HTML content from email: Set "html" parameter to `{{$json.textAsHtml}}`  
   - Output key: `md`

3. **Add LangChain Google Gemini Chat Node "Google Gemini Chat Model"**  
   - Credentials: Connect Google Palm API (Google Gemini)  
   - Use default options.

4. **Add ScrapegraphAI Node "Scrape"**  
   - Credentials: Setup ScrapegraphAI API key  
   - Parameters: Set `websiteUrl` dynamically from AI input (`Website_URL`)  
   - User prompt: "Entra nell'url di ogni articolo e fai il summary con i concetti principali in italiano dell'articolo setesso"

5. **Add LangChain Agent Node "Scrape Agent"**  
   - Input text: `{{$json.md}}` (Markdown newsletter)  
   - System message: Instructions to split newsletter into articles with fields title, link, summary in Italian, and use "Scrape" tool for each URL  
   - Link AI tools: Use "Google Gemini Chat Model" and connect "Scrape" as AI tool.

6. **Add Code Node "Validate Json"**  
   - Paste provided JavaScript code that cleans and parses AI JSON output and emits one item per article.

7. **Add SplitInBatches Node "Loop Over Items"**  
   - Default options to process one article at a time.

8. **Add LangChain OpenAI Chat Node "OpenAI Chat Model"**  
   - Credentials: OpenAI API key ("OpenAi account (Eure)")  
   - Model: "gpt-5-mini"  
   - Connect input from "Loop Over Items" output.

9. **Add LangChain Chain LLM Node "Categorization Chain"**  
   - Prompt: Provide article title and summary, instruct model to categorize into one of seven fixed categories (as per list in the prompt)  
   - Connect input from "Loop Over Items" (title & summary) and connect AI model input from "OpenAI Chat Model".

10. **Add HTTP Request Node "Short url"**  
    - Method: POST  
    - URL: `https://cleanuri.com/api/v1/shorten`  
    - Content Type: form-urlencoded  
    - Body Parameter: `url` set to current article link (`{{$node["Loop Over Items"].item.json.link}}`)

11. **Add Google Sheets Node "Add article to sheet"**  
    - Operation: Append  
    - Document ID: `1CvsUadsmoLKZUUYd2KEEGMnJidqRqnLW-BvIw66_LX4`  
    - Sheet Name: `gid=0` (or actual sheet)  
    - Columns:  
      - DATE: `{{$now.format('dd/LL/yyyy')}}`  
      - LINK: `{{$json.result_url}}` (shortened URL)  
      - TITLE: `{{$node["Loop Over Items"].item.json.title}}`  
      - SUMMARY: `{{$node["Loop Over Items"].item.json.summary}}`  
      - CATEGORY: `{{$node["Categorization Chain"].item.json.text}}`  
    - Credentials: Google Sheets OAuth2

12. **Wire Nodes Together**  
    - Get email → Convert HTML to MD → Scrape Agent → Validate Json → Loop Over Items  
    - Loop Over Items → OpenAI Chat Model → Categorization Chain → Short url → Add article to sheet  
    - Loop Over Items also connects to Categorization Chain  
    - Scrape Agent uses Google Gemini Chat Model and Scrape tool nodes internally.

13. **Set up Credentials**  
    - Gmail OAuth2 with access to inbox.  
    - Google Palm API for Gemini.  
    - ScrapegraphAI API key.  
    - OpenAI API key.  
    - Google Sheets OAuth2 with edit rights to the target spreadsheet.

14. **Test and Activate**  
    - Use an example email ID to test the flow end-to-end.  
    - Confirm articles are extracted, categorized, URLs shortened, and appended to Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow is designed to automatically process AI news emails, extract and summarize articles, categorize them, and store the results in a structured Google Sheet for daily tracking and insights.                                               | Overview sticky note in workflow.                                                                            |
| Please clone [this Google Sheet](https://docs.google.com/spreadsheets/d/15VKWMbd2cOb74uwYncfXkwLZam-dXD5sBiAttVpKIQw/edit?usp=sharing) to use as the archive destination.                                                                                 | Google Sheets archive template.                                                                               |
| ScrapegraphAI account setup is required to enable scraping and summarization of article URLs. Register and obtain API key at https://dashboard.scrapegraphai.com/?via=n3witalia                                                                        | Sticky note 2 and Scrape node credentials info.                                                              |
| The categorization uses a fixed set of categories relevant to AI news; these categories must be maintained consistently in the prompt to ensure stable classification results.                                                                          | Categorization Chain prompt description.                                                                     |
| CleanURI is used for URL shortening via a simple public API; consider API limits or alternative shortening services if volume is high.                                                                                                               | Short url node description.                                                                                   |
| Credentials must be tested and authorized before activating the workflow to avoid runtime failures.                                                                                                                                                    | General best practice.                                                                                        |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, respecting all content policies and containing no illegal, offensive, or protected elements. All processed data is legal and public.