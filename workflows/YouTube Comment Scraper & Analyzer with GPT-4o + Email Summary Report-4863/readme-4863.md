YouTube Comment Scraper & Analyzer with GPT-4o + Email Summary Report

https://n8nworkflows.xyz/workflows/youtube-comment-scraper---analyzer-with-gpt-4o---email-summary-report-4863


# YouTube Comment Scraper & Analyzer with GPT-4o + Email Summary Report

---

### 1. Workflow Overview

This workflow automates the process of collecting YouTube video comments, performing sentiment and thematic analysis with GPT-4o, and sending a summarized email report. It is designed for content creators, marketers, or analysts who want to monitor and understand viewer feedback on YouTube videos efficiently.

**Key Use Cases:**
- Automate YouTube comment data retrieval and analysis
- Generate actionable insights and sentiment breakdowns using GPT-4o AI
- Deliver summarized reports via email
- Manage processing status through Google Sheets to avoid duplicates

**Logical Blocks:**

- **1.1 Trigger & Input Reception:** Monitors a Google Sheet for new or updated video entries marked as "Pending".
- **1.2 Validation & Limiting:** Filters for relevant rows and limits processing to one item at a time to prevent API overload.
- **1.3 YouTube Data Collection:** Retrieves video metadata and top comments via YouTube API.
- **1.4 Data Preparation & Basic Analysis:** Extracts comment text, computes basic stats, and performs initial sentiment counts.
- **1.5 AI-Powered Sentiment & Thematic Analysis:** Uses GPT-4o to analyze comments and generate detailed structured insights.
- **1.6 Email Formatting & Sending:** Converts AI output to a rich HTML email and sends it via Gmail.
- **1.7 Status Update:** Marks videos as processed in Google Sheets to avoid repeated processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

- **Overview:**  
  This block monitors a designated Google Sheet every minute for new or updated video IDs flagged for processing.

- **Nodes Involved:**  
  - Pick Video Ids from Google sheet  
  - If (filter for "Pending" status)

- **Node Details:**

  - **Pick Video Ids from Google sheet**  
    - Type: Google Sheets Trigger  
    - Role: Polls the Google Sheet (gid=0) every minute to detect new or updated rows.  
    - Configuration: Uses OAuth2 credentials; polls all rows; expects columns including `ID`, `Video Title`, `YouTube Video ID`, and `Status`.  
    - Input: External trigger from Google Sheets changes  
    - Output: Rows from sheet  
    - Edge cases: Missed polling intervals, OAuth token expiry, sheet access permissions  

  - **If**  
    - Type: Conditional filter  
    - Role: Filters rows to only those with `Status` exactly "Pending".  
    - Configuration: Strict string equality on `Status` field.  
    - Input: Rows from Google Sheet trigger  
    - Output: Only "Pending" rows to next block  
    - Edge cases: Case sensitivity issues, missing `Status` field, rows with unexpected statuses  

#### 2.2 Validation & Limiting

- **Overview:**  
  Limits processing concurrency to protect API usage and system stability.

- **Nodes Involved:**  
  - Limit  
  - Set Video Details

- **Node Details:**

  - **Limit**  
    - Type: Limit node  
    - Role: Processes only one item at a time (concurrency limit = 1).  
    - Configuration: Default settings.  
    - Input: Filtered rows from "If" node  
    - Output: Single row passed forward  
    - Edge cases: Long queues if many videos pending  

  - **Set Video Details**  
    - Type: Set node  
    - Role: Extracts and sets `videoId` (from "Youtube video id" field) and sets max comments limit (100).  
    - Configuration:  
      - `videoId` = value of Google Sheet column `Youtube video id`  
      - `maxComments` = 100 (number)  
    - Input: Single pending row from Limit node  
    - Output: Structured data for YouTube API nodes  
    - Edge cases: Missing or malformed video ID  

#### 2.3 YouTube Data Collection

- **Overview:**  
  Retrieves video metadata and top comments via YouTube API.

- **Nodes Involved:**  
  - Get Youtube Video Details  
  - Get Youtube Video Comments

- **Node Details:**

  - **Get Youtube Video Details**  
    - Type: YouTube node  
    - Role: Fetches video metadata such as title and channel  
    - Configuration:  
      - Operation: get video details  
      - Input videoId from previous node  
      - Auth: OAuth2 YouTube credentials  
    - Input: `videoId` and `maxComments` from "Set Video Details"  
    - Output: Video metadata JSON  
    - Edge cases: Invalid video ID, API rate limits, authentication errors  

  - **Get Youtube Video Comments**  
    - Type: HTTP Request node  
    - Role: Fetches top 100 comments ordered by relevance (top-level only)  
    - Configuration:  
      - URL: YouTube API endpoint for commentThreads  
      - Query parameters: `part=snippet`, `videoId`, `maxResults=100`, `order=relevance`  
      - Auth: OAuth2 YouTube credentials  
    - Input: videoId from "Set Video Details"  
    - Output: Raw comments JSON  
    - Edge cases: API quota exceeded, comments disabled, network errors  

#### 2.4 Data Preparation & Basic Analysis

- **Overview:**  
  Processes raw comment data, extracts useful fields, calculates statistics, performs basic keyword sentiment analysis, and limits comments to 50 for AI.

- **Nodes Involved:**  
  - Prepare Comments Data

- **Node Details:**

  - **Prepare Comments Data**  
    - Type: Code node (JavaScript)  
    - Role: Parses raw comments, extracts text, author, likes, replies; computes total comments, average likes, total replies; performs simple sentiment word counts; prepares comments text for AI input limited to 50 comments.  
    - Key expressions: Uses `$input`, references previous nodes by name to get video title and comments  
    - Input: Raw comments from "Get Youtube Video Comments" and metadata from "Get Youtube Video Details"  
    - Output: JSON object with processed comments, stats, sentiment counts, and text block for AI  
    - Edge cases: Empty comments, unexpected data structures, large comments sets exceeding token limits  
    - Notes: Limits AI input to 50 comments to control token usage  

#### 2.5 AI-Powered Sentiment & Thematic Analysis

- **Overview:**  
  Uses GPT-4o to analyze comment texts, generating structured sentiment breakdown, main themes, viewer questions, key feedback, and actionable insights.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Sends prepared comment text and stats to GPT-4o for analysis  
    - Configuration:  
      - Text prompt includes video title, total comments, average likes, and comment texts  
      - System message defines analysis tasks and output keys: overall sentiment breakdown, main themes, common questions, key feedback, actionable insights  
      - Max iterations: 100  
    - Input: Processed data from "Prepare Comments Data"  
    - Output: AI generated structured JSON analysis  
    - Edge cases: API errors, malformed AI responses, rate limits, prompt failures  
    - Notes: Prompt customizable for different analysis needs  

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node  
    - Role: Provides GPT-4o-mini model interface for AI Agent  
    - Configuration:  
      - Model: gpt-4o-mini  
      - Response format: JSON object  
      - Credentials: OpenAI API key  
    - Input: AI Agent node’s language model request  
    - Output: GPT-4o response JSON  
    - Edge cases: API quota, connectivity issues, credential errors  

#### 2.6 Email Formatting & Sending

- **Overview:**  
  Formats AI analysis and statistics into a professional HTML email, then sends it via Gmail.

- **Nodes Involved:**  
  - Prepare HTML for Email  
  - Gmail Account Configuration

- **Node Details:**

  - **Prepare HTML for Email**  
    - Type: Code node (JavaScript)  
    - Role: Parses AI JSON output, builds structured HTML email including: video title, quick stats, sentiment summary, common questions, feedback points, actionable insights with styling  
    - Key expressions: Uses JSON parsing, template literals, and data from "Prepare Comments Data" for stats  
    - Input: AI Agent output JSON  
    - Output: JSON with `emailHTML`, `subject`, `videoTitle`  
    - Edge cases: Malformed AI JSON, missing fields, HTML injection risks  

  - **Gmail Account Configuration**  
    - Type: Gmail node  
    - Role: Sends formatted email report  
    - Configuration:  
      - Recipient email address (must be updated)  
      - Email body: HTML content from previous node  
      - Subject line includes video title and comment count  
      - Credentials: Gmail OAuth2 account  
    - Input: Prepared email JSON  
    - Output: Email sent confirmation  
    - Edge cases: Authentication failure, invalid recipient, email quota reached  

#### 2.7 Status Update

- **Overview:**  
  Updates the Google Sheet row status to “Mail Sent” to mark completion and prevent repeated processing.

- **Nodes Involved:**  
  - Update Status on Google Sheet

- **Node Details:**

  - **Update Status on Google Sheet**  
    - Type: Google Sheets node  
    - Role: Updates the `Status` column of the corresponding row identified by `Id` to “Mail Sent”  
    - Configuration:  
      - Operation: update  
      - Matching column: `Id` from initial Google Sheet row  
      - Sheet and document IDs specified  
      - Credentials: Google Sheets OAuth2 account  
    - Input: Original row `Id` from initial Google Sheet trigger  
    - Output: Confirmation of update  
    - Edge cases: Row not found, permission errors, concurrent edits  

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                                    | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                  |
|-----------------------------|-----------------------------|---------------------------------------------------|--------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Pick Video Ids from Google sheet | Google Sheets Trigger       | Trigger on new/updated Google Sheet rows          | External (Google Sheet changes) | If                              | Triggers on new YouTube videos added to spreadsheet. Polls every minute for changes.                         |
| If                          | If Node                    | Filters rows to only those with Status = "Pending"| Pick Video Ids from Google sheet | Limit                          | Filters rows to process only videos with 'Pending' status                                                   |
| Limit                       | Limit Node                 | Limits to processing one item at a time            | If                             | Set Video Details               | Limits processing to 1 item at a time to prevent API overload                                               |
| Set Video Details            | Set Node                   | Extracts videoId, sets max comments limit          | Limit                          | Get Youtube Video Details       | Prepares video ID and sets max comments limit (100)                                                         |
| Get Youtube Video Details    | YouTube Node               | Fetches video metadata for the video                | Set Video Details              | Get Youtube Video Comments      | Fetches video metadata including title, channel name, and other details                                     |
| Get Youtube Video Comments   | HTTP Request Node          | Fetches top 100 comments ordered by relevance      | Get Youtube Video Details      | Prepare Comments Data           | Retrieves top 100 comments ordered by relevance using YouTube API                                           |
| Prepare Comments Data        | Code Node                  | Extracts comment text, calculates stats, sentiment | Get Youtube Video Comments     | AI Agent                      | ⚠️ Data Preparation: extracts comment texts, calculates stats, basic sentiment analysis, limits to 50 comments|
| AI Agent                    | Langchain Agent Node       | Analyzes comments with GPT-4o for insights         | Prepare Comments Data          | Prepare HTML for Email          | Uses OpenAI GPT-4o to analyze comments and generate insights. Customize the prompt for different analysis needs |
| OpenAI Chat Model            | Langchain OpenAI Chat Model| Provides GPT-4o-mini model interface                | AI Agent                      | AI Agent                      |                                                                                                              |
| Prepare HTML for Email       | Code Node                  | Converts AI analysis to formatted HTML email        | AI Agent                      | Gmail Account Configuration    | Converts AI analysis into formatted HTML email with statistics, insights, and professional styling          |
| Gmail Account Configuration  | Gmail Node                 | Sends email report via Gmail                         | Prepare HTML for Email         | Update Status on Google Sheet   | Sends formatted analysis report via Gmail. Update SENDER_EMAIL_ADDRESS with actual recipient                 |
| Update Status on Google Sheet| Google Sheets Node          | Updates status to 'Mail Sent' in Google Sheet       | Gmail Account Configuration    | (none)                        | Updates video status to 'Mail Sent' to prevent duplicate processing                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger Node:**
   - Type: Google Sheets Trigger  
   - Configure OAuth2 credentials for Google Sheets  
   - Set document and sheet to your YouTube videos sheet (gid=0)  
   - Poll every minute  
   - Sheet must have columns: `ID`, `Video Title`, `Youtube video id`, and `Status`  

2. **Add an If Node:**
   - Filter incoming rows where `Status` equals `"Pending"` (case-sensitive, strict match)  

3. **Add a Limit Node:**
   - Set concurrency limit to 1 item to prevent API overload  

4. **Add a Set Node ("Set Video Details"):**
   - Map `videoId` to the value from the Google Sheet column `Youtube video id`  
   - Add a number field `maxComments` with value 100  

5. **Add a YouTube Node ("Get Youtube Video Details"):**
   - Operation: Get Video  
   - Input: `videoId` from Set node  
   - Authenticate with your YouTube OAuth2 credentials  

6. **Add an HTTP Request Node ("Get Youtube Video Comments"):**
   - URL: `https://www.googleapis.com/youtube/v3/commentThreads`  
   - Query parameters:  
     - `part=snippet`  
     - `videoId` from "Set Video Details"  
     - `maxResults=100`  
     - `order=relevance`  
   - Authentication: YouTube OAuth2 credentials  

7. **Add a Code Node ("Prepare Comments Data"):**
   - JavaScript code to:  
     - Extract comment text, author, likes, replies  
     - Compute total comments, average likes, total replies  
     - Perform basic sentiment word counts (simple positive/negative keywords)  
     - Limit comment texts to first 50 for AI input  
     - Output structured JSON with all above data  

8. **Add a Langchain Agent Node ("AI Agent"):**
   - Use OpenAI GPT-4o agent  
   - Configure prompt with system message specifying JSON output keys:  
     - overall_sentiment_breakdown, main_themes, common_questions, key_feedback_points, actionable_insights  
   - Input the prepared comment texts and stats from previous node  

9. **Add a Langchain OpenAI Chat Model Node:**
   - Model: `gpt-4o-mini`  
   - Response format: JSON object  
   - Use OpenAI API credentials  

10. **Connect OpenAI Chat Model as the language model provider for AI Agent node.**

11. **Add a Code Node ("Prepare HTML for Email"):**
    - Parse AI JSON output  
    - Construct an HTML email with styling, including:  
      - Video title  
      - Quick stats (total comments, average likes, replies, sentiment summary)  
      - Lists for common questions, feedback points, actionable insights  
    - Output JSON with `emailHTML`, `subject`, `videoTitle`  

12. **Add a Gmail Node ("Gmail Account Configuration"):**
    - Set recipient email (update placeholder `SENDER_EMAIL_ADDRESS` to actual email)  
    - Message: HTML from previous node  
    - Subject: Dynamic with video title and comment count  
    - Use Gmail OAuth2 credentials  

13. **Add a Google Sheets Node ("Update Status on Google Sheet"):**
    - Operation: Update  
    - Match row by `Id` from the initial Google Sheet trigger row  
    - Set `Status` column to `"Mail Sent"`  
    - Use same Google Sheets credentials and document as trigger node  

14. **Connect nodes in order:**
    - Google Sheets Trigger → If → Limit → Set Video Details → Get Youtube Video Details → Get Youtube Video Comments → Prepare Comments Data → AI Agent → Prepare HTML for Email → Gmail → Update Status on Google Sheet  

15. **Test end-to-end:**
    - Add a new row to Google Sheet with a valid YouTube video ID and Status = "Pending"  
    - Observe workflow execution, email delivery, and status update  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow automates YouTube comment collection, AI analysis with GPT-4o, and email reporting triggered by Google Sheet changes.           | Workflow Overview sticky note                                                                        |
| Google Sheet structure must include columns: ID, Video Title, Youtube video id, Status. Set Status to "Pending" to trigger processing.         | Trigger Documentation sticky note                                                                   |
| YouTube API limits: max 100 comments per request, ordered by relevance, top-level comments only.                                               | YouTube API Section sticky note                                                                      |
| GPT-4o analyzes comments for sentiment breakdown, main themes, common questions, feedback points, and actionable insights. Output is JSON.     | AI Analysis Documentation sticky note                                                               |
| Email includes video stats, sentiment summary, key insights, and formatted HTML. Recipient email must be updated in Gmail node.                | Email & Status Update sticky note                                                                    |
| Data Preparation node limits comments to 50 for AI analysis to manage token usage and performs basic sentiment keyword counts.                | Data Processing Note sticky note                                                                     |
| Limits concurrency to 1 item to avoid API overload and rate limiting.                                                                           | Limit node sticky note                                                                                |
| Customize AI prompt in AI Agent node to adjust analysis focus as needed.                                                                        | AI Agent node note                                                                                   |
| Gmail node requires valid OAuth2 credentials and correct recipient email address for successful delivery.                                       | Gmail Account Configuration node note                                                               |
| Google Sheets update node prevents duplicate email sends by marking processed videos as "Mail Sent".                                            | Update Status on Google Sheet node note                                                             |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with relevant content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---