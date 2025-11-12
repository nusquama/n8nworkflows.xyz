Generate Articles from Google Sheets to WordPress with Web Scraping and GPT-4 Mini

https://n8nworkflows.xyz/workflows/generate-articles-from-google-sheets-to-wordpress-with-web-scraping-and-gpt-4-mini-8921


# Generate Articles from Google Sheets to WordPress with Web Scraping and GPT-4 Mini

### 1. Workflow Overview

This workflow automates the generation of SEO-optimized news articles by fetching new article requests from a Google Sheets document, scraping the corresponding web pages, summarizing the content with AI, creating full articles with GPT-4 Mini, and finally publishing drafts to WordPress. It is designed to streamline content creation for news or blog sites with minimal manual intervention.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Article Retrieval**: Receives trigger requests, fetches new article rows from Google Sheets, and iterates through each article.

- **1.2 Status Update Before Processing**: Marks articles as “Processing” in Google Sheets before scraping to avoid duplication.

- **1.3 Web Scraping and Error Handling**: Fetches HTML content from the article source URLs and branches depending on success or failure.

- **1.4 Content Extraction and AI Summarization**: Uses AI to extract and summarize main article content from raw HTML.

- **1.5 AI Article Creation**: Generates a fully formatted, SEO-optimized news article using GPT-4 Mini based on the summarized content.

- **1.6 Article Formatting and WordPress Draft Creation**: Cleans and formats the article output, then creates a draft post in WordPress.

- **1.7 Final Status Update**: Updates Google Sheets with the article publish status and draft link.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Article Retrieval

- **Overview:**  
  This block starts the workflow via a webhook and fetches new articles marked as “New” from a specific Google Sheets document. It splits the fetched data into batches for iterative processing.

- **Nodes Involved:**  
  - Webhook  
  - Get New Articles (Google Sheets)  
  - Loop Over Items (Split In Batches)  
  - Sticky Note2 (comment)

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Trigger  
    - Configuration: Listens on path `a1889360-e1bf-4b53-a96b-5eafa4b165a1` to start workflow on demand.  
    - Inputs: External HTTP request  
    - Outputs: Triggers fetching new articles  
    - Edge cases: Network timeout, unauthorized access if webhook is exposed  
    - No sub-workflow

  - **Get New Articles**  
    - Type: Google Sheets (Read)  
    - Configuration: Reads sheet named "Cheapnail" filtering rows where column "Flow Status" equals "New"  
    - Inputs: Trigger from webhook  
    - Outputs: Array of new article rows with metadata  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: API quota limits, empty result sets, sheet access errors  
    - No sub-workflow

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Configuration: Splits the fetched array into single items for sequential processing  
    - Inputs: New articles array from Google Sheets  
    - Outputs: Single article items for downstream processing  
    - Edge cases: Empty input array, batch size misconfiguration  
    - No sub-workflow

  - **Sticky Note2**  
    - Functional comment: "Get 'New' Articles data from G Sheet" (visual aid)

#### 2.2 Status Update Before Processing

- **Overview:**  
  Updates the Google Sheet row of each article to mark it as "Processing" and records the current timestamp, preventing multiple processing attempts.

- **Nodes Involved:**  
  - Update Processing (Google Sheets)  
  - Sticky Note1 (comment)

- **Node Details:**

  - **Update Processing**  
    - Type: Google Sheets (Update)  
    - Configuration: Updates the current article row (matched by `row_number`) setting "Publish Status" to "Processing", "Flow Status" as blank (?), and current Asia/Kolkata timestamp in "Flow Timing"  
    - Inputs: Single article item from Loop Over Items  
    - Outputs: Updated row confirmation  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Row not found, API limits, concurrency conflicts  
    - No sub-workflow

  - **Sticky Note1**  
    - Functional comment: "🕸  Fetch the article information from web" (placed near this block)

#### 2.3 Web Scraping and Error Handling

- **Overview:**  
  Fetches the raw HTML content of the article source URL. If the fetch fails (e.g., timeout, HTTP error), the workflow updates the sheet marking the article as failed and moves on.

- **Nodes Involved:**  
  - Fetch HTML (HTTP Request)  
  - If (Conditional)  
  - Update Error (Google Sheets)  
  - Loop Over Items (to continue)  

- **Node Details:**

  - **Fetch HTML**  
    - Type: HTTP Request  
    - Configuration: GET request to URL from the article's "Source" field with 10-second timeout  
    - Inputs: Article item with Source URL from Update Processing  
    - Outputs: Response JSON with either `data` containing HTML or `error` status  
    - Edge cases: Timeout, 404/500 HTTP errors, network issues  
    - On error: "continueRegularOutput" so workflow continues with error data

  - **If**  
    - Type: Conditional  
    - Configuration: Checks if `$json.data` exists (i.e., fetch succeeded)  
    - Inputs: Output of Fetch HTML  
    - Outputs:  
      - True branch → Article Summarizer  
      - False branch → Update Error  
    - Edge cases: Expression failures if `data` field missing or malformed  

  - **Update Error**  
    - Type: Google Sheets (Update)  
    - Configuration: Marks Flow Status = "Error", Publish Status = "Not Processed Because of an Error", and logs HTTP error code in Publish Link field for the corresponding row  
    - Inputs: Failure data from If node  
    - Outputs: Confirmation, then loops back to Loop Over Items for next article  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Same as previous Google Sheets updates  

#### 2.4 Content Extraction and AI Summarization

- **Overview:**  
  Sends the fetched HTML to an AI model (GPT-4 Mini) to extract and convert the article title and body text into clean Markdown format.

- **Nodes Involved:**  
  - Article Summarizer (OpenAI)  
  - Article Creator (OpenAI)  
  - Sticky Note (comment)

- **Node Details:**

  - **Article Summarizer**  
    - Type: OpenAI via Langchain node  
    - Configuration: Uses GPT-4 Mini model with a system prompt instructing it to parse HTML, extract the title (from `<title>` or `<h1>`) and main text content excluding images and links, and output Markdown with heading and paragraphs. Includes example input and output. Input HTML is passed from Fetch HTML's `.data` field.  
    - Outputs: Markdown text of article content  
    - Credentials: OpenAI API key  
    - Edge cases: Model timeouts, malformed HTML input, token limits

  - **Article Creator**  
    - Type: OpenAI via Langchain node  
    - Configuration: Uses GPT-4 Mini with a complex system prompt to generate a full news article in JSON output format, including SEO-optimized headline, bolded key points, bullet points, blockquotes, and a custom call-to-action. The prompt references the summarized Markdown content as "Source Data."  
    - Inputs: Output from Article Summarizer  
    - Outputs: JSON with `title` and HTML `content` fields for final article  
    - Credentials: OpenAI API key  
    - Edge cases: Token limits, output parsing errors

  - **Sticky Note**  
    - Content: "🤖 These Two AI Brother is creating the Article ✌ - Article Summarizer - Article Creator"

#### 2.5 Article Formatting and WordPress Draft Creation

- **Overview:**  
  Processes the AI-generated article JSON to extract title and content separately, removes redundant `<h1>` tags from content, then creates a WordPress draft post with the generated article.

- **Nodes Involved:**  
  - Format Article (Code)  
  - Create a Draft (WordPress)  
  - Sticky Note3 (comment)

- **Node Details:**

  - **Format Article**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Extracts `message.content` (HTML) and optional title from the AI output.  
      - Removes first `<h1>` tag from content if present to avoid duplicate titles.  
      - Outputs clean `title` and `content` fields for WordPress.  
    - Inputs: Output from Article Creator  
    - Outputs: JSON with separate `title` and `content` fields  
    - Edge cases: Missing or malformed HTML content, regex failures

  - **Create a Draft**  
    - Type: WordPress node  
    - Configuration: Creates a WordPress post with `title`, `content`, author ID 12, and status set to "draft".  
    - Inputs: Cleaned article from Format Article node  
    - Credentials: WordPress API credentials  
    - Edge cases: WordPress API failures, permission errors

  - **Sticky Note3**  
    - Content: "📂 Make Draft Article and Update the Google Sheet"

#### 2.6 Final Status Update

- **Overview:**  
  Updates the Google Sheet row once the WordPress draft is created with the draft link (URL) and status "Flow Complete," signaling successful processing.

- **Nodes Involved:**  
  - Update Draft Details (Google Sheets)  
  - Loop Over Items (to continue next article)

- **Node Details:**

  - **Update Draft Details**  
    - Type: Google Sheets (Update)  
    - Configuration: Updates the row with "Flow Status" = "Flow Complete", "Publish Status" from WordPress post status, and "Publish Link" from WordPress post URL (draft link).  
    - Inputs: Output of Create a Draft node (WordPress) and original row number from Update Processing node.  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Row update conflicts, API rate limits

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                                  | Input Node(s)          | Output Node(s)           | Sticky Note                                      |
|--------------------|-------------------------------|-------------------------------------------------|------------------------|--------------------------|-------------------------------------------------|
| Webhook            | n8n-nodes-base.webhook         | Trigger workflow start                           | External HTTP Request  | Get New Articles         |                                                 |
| Get New Articles    | n8n-nodes-base.googleSheets    | Fetch new articles marked "New" from Google Sheets | Webhook                | Loop Over Items          | 📑 Get 'New' Articles data from G Sheet          |
| Loop Over Items    | n8n-nodes-base.splitInBatches  | Iterate over each article row                     | Get New Articles       | Update Processing (failure continues to loop) |                                                 |
| Update Processing  | n8n-nodes-base.googleSheets    | Mark article as "Processing" in sheet            | Loop Over Items        | Fetch HTML               | 🕸  Fetch the article information from web        |
| Fetch HTML         | n8n-nodes-base.httpRequest     | Fetch article source HTML                          | Update Processing      | If                       |                                                 |
| If                 | n8n-nodes-base.if              | Check if HTML fetch succeeded                      | Fetch HTML             | Article Summarizer (true), Update Error (false) |                                                 |
| Update Error       | n8n-nodes-base.googleSheets    | Mark article row as error in sheet                | If                     | Loop Over Items          |                                                 |
| Article Summarizer | @n8n/n8n-nodes-langchain.openAi | Extract and summarize article content from HTML  | If (true)              | Article Creator          | 🤖 These Two AI Brother is creating the Article ✌ - Article Summarizer - Article Creator |
| Article Creator    | @n8n/n8n-nodes-langchain.openAi | Generate full SEO-optimized article from summary | Article Summarizer     | Format Article           |                                                 |
| Format Article     | n8n-nodes-base.code            | Clean and format AI output for WordPress          | Article Creator        | Create a Draft           | 📂 Make Draft Article and Update the Google Sheet |
| Create a Draft     | n8n-nodes-base.wordpress       | Create WordPress draft post                        | Format Article         | Update Draft Details     |                                                 |
| Update Draft Details| n8n-nodes-base.googleSheets    | Update row with final status and draft link       | Create a Draft         | Loop Over Items          |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `a1889360-e1bf-4b53-a96b-5eafa4b165a1`  
   - Purpose: Receive external triggers to start workflow

2. **Add Google Sheets Node "Get New Articles"**  
   - Operation: Read rows  
   - Document ID: Google Sheets ID `"1CnYbGz4Xq0D0eI6T4jelZkL1c3ZY5hKB2wjhWtmjAOw"`  
   - Sheet Name: `"Cheapnail"`  
   - Filter: Column "Flow Status" equals `"New"`  
   - Credentials: Google Sheets OAuth2

3. **Add Split In Batches Node "Loop Over Items"**  
   - Batch Size: 1 (default)  
   - Input: Output from "Get New Articles"  
   - Purpose: Process one article at a time

4. **Add Google Sheets Node "Update Processing"**  
   - Operation: Update row  
   - Document ID and Sheet as above  
   - Matching Column: `row_number` from current item  
   - Columns to update:  
     - "Topic" = from current item  
     - "Source" = from current item  
     - "Flow Timing" = current datetime in `Asia/Kolkata` timezone formatted `dd-MMM-yyyy HH:mm:ss`  
     - "Publish Status" = `"Processing"`  
   - Credentials: Google Sheets OAuth2  
   - Connect: Output of "Loop Over Items" (second output branch)

5. **Add HTTP Request Node "Fetch HTML"**  
   - Method: GET  
   - URL: `={{ $json.Source }}` (source URL from current item)  
   - Timeout: 10 seconds  
   - Error Handling: Continue on error (to handle failures gracefully)  
   - Connect: Output of "Update Processing"

6. **Add If Node "If"**  
   - Condition: Check if `$json.data` exists (string exists operation)  
   - Connect: Output of "Fetch HTML"  
   - True output: Continue processing  
   - False output: Handle error

7. **Add Google Sheets Node "Update Error"**  
   - Operation: Update row  
   - Document ID and Sheet as above  
   - Matching Column: `row_number` from "Update Processing" item  
   - Columns to update:  
     - "Flow Status" = `"Error"`  
     - "Publish Status" = `"Not Processed Because of an Error"`  
     - "Publish Link" = HTTP error status code from "Fetch HTML" node  
   - Credentials: Google Sheets OAuth2  
   - Connect: False branch of "If" node

8. **Add OpenAI Node "Article Summarizer"**  
   - Provider: OpenAI API (GPT-4 Mini)  
   - Prompt: System message to extract title and main text from HTML input as Markdown, following detailed instructions and example  
   - Input: Pass `data` field from "Fetch HTML" node  
   - Credentials: OpenAI API key  
   - Connect: True branch of "If" node

9. **Add OpenAI Node "Article Creator"**  
   - Provider: OpenAI API (GPT-4 Mini)  
   - Prompt: System message instructing to generate a detailed news article with SEO optimization, headings, bolding, bullet points, blockquotes, and a custom CTA, referencing the summarized Markdown from previous node  
   - Output: JSON with `title` and HTML `content`  
   - Credentials: OpenAI API key  
   - Connect: Output of "Article Summarizer"

10. **Add Code Node "Format Article"**  
    - JavaScript code to:  
      - Extract `title` and `content` from AI JSON output  
      - Remove first `<h1>` from content if present to avoid duplicates  
    - Connect: Output of "Article Creator"

11. **Add WordPress Node "Create a Draft"**  
    - Operation: Create post  
    - Title: `={{ $json.title }}`  
    - Content: `={{ $json.content }}`  
    - Status: Draft  
    - Author ID: 12  
    - Credentials: WordPress API credentials  
    - Connect: Output of "Format Article"

12. **Add Google Sheets Node "Update Draft Details"**  
    - Operation: Update row  
    - Document ID and Sheet as above  
    - Matching Column: `row_number` from "Update Processing" node  
    - Columns to update:  
      - "Flow Status" = `"Flow Complete"`  
      - "Publish Status" = WordPress post status  
      - "Publish Link" = WordPress post link  
    - Credentials: Google Sheets OAuth2  
    - Connect: Output of "Create a Draft"

13. **Connect "Update Draft Details" back to "Loop Over Items"**  
    - To continue processing next article

14. **Add Sticky Notes** (optional visualization aid)  
    - Near "Get New Articles": "📑 Get 'New' Articles data from G Sheet"  
    - Near "Update Processing" & "Fetch HTML": "🕸  Fetch the article information from web"  
    - Near "Article Summarizer" & "Article Creator": "🤖 These Two AI Brother is creating the Article ✌ - Article Summarizer - Article Creator"  
    - Near formatting & WordPress nodes: "📂 Make Draft Article and Update the Google Sheet"

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow uses Google Sheets OAuth2 and WordPress API credentials; ensure proper permissions and quota limits. | Google Sheets API docs: https://developers.google.com/sheets/api                                   |
| OpenAI GPT-4 Mini model is used via Langchain integration for both summarization and article generation.       | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4-mini                               |
| The workflow is designed for timezone Asia/Kolkata; adjust as needed for other deployments.                    | Moment.js or equivalent library used for timezone formatting                                     |
| Error handling in HTTP Request node allows graceful continuation, marking failed articles in Google Sheets.    | n8n docs on HTTP Request error handling: https://docs.n8n.io/nodes/n8n-nodes-base.httprequest/    |
| Sticky notes are included for workflow clarity and team communication; they do not affect logic.                | n8n Sticky Note node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.stickyNote/          |

---

This completes the detailed analysis and reconstruction guide for the “🤖 Automated AI Article Generation from Google Sheets to WordPressnail” workflow. It enables complete understanding, modification, and re-implementation without the original JSON export.