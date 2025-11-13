Detect Reddit Problems & Auto-Comment Solutions with Google Gemini & Sheets

https://n8nworkflows.xyz/workflows/detect-reddit-problems---auto-comment-solutions-with-google-gemini---sheets-10343


# Detect Reddit Problems & Auto-Comment Solutions with Google Gemini & Sheets

### 1. Workflow Overview

This workflow automates the detection of Reddit posts discussing problems related to AI automation, specifically within the "n8n" subreddit, and generates actionable solutions using Google Gemini AI models. It optionally archives these interactions in Google Sheets and can post replies directly on Reddit. The use cases include monitoring user feedback, automating support or community engagement, and maintaining a database of issues and solutions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Data Source**: Initiates workflow manually, searches Reddit for relevant posts.
- **1.2 Post Filtering & Normalization**: Filters posts by relevance and normalizes data for AI processing.
- **1.3 AI Problem Classification**: Uses AI to classify whether a post describes a problem.
- **1.4 AI Solution Generation**: Generates actionable solutions for posts flagged as problems.
- **1.5 Result Consolidation and Output**: Combines AI output with original data, archives results, and optionally posts comments to Reddit.
- **1.6 Supportive Notes & Credentials**: Embedded documentation and safety notes for workflow users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Source

- **Overview:**  
This block starts the workflow manually and searches Reddit for posts in the r/n8n subreddit containing the phrase "Why i stopped using", fetching up to 10 posts.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Post Searching  
  - If Condition (filtering posts)  

- **Node Details:**  

1. **When clicking ‘Execute workflow’**  
   - Type: Manual Trigger  
   - Role: Starts the workflow on demand  
   - Config: No parameters; manual execution only  
   - Input: None  
   - Output: Trigger flow to Reddit search  
   - Edge cases: None for trigger, manual execution only  
   - Notes: Can be replaced by Schedule or Webhook triggers for automation  

2. **Post Searching**  
   - Type: Reddit node (search operation)  
   - Role: Searches Reddit posts in r/n8n with keyword "Why i stopped using"  
   - Config: Limit 10 posts; subreddit = "n8n"; keyword = "Why i stopped using"  
   - Input: Trigger from Manual node  
   - Output: Array of Reddit posts  
   - Edge cases: API rate limits, auth errors, empty search results  

3. **If Condition**  
   - Type: Conditional filter  
   - Role: Filters posts with at least 2 upvotes and non-empty selftext  
   - Config:  
     - Condition 1: `ups` (upvotes) >= 2  
     - Condition 2: `selftext` not empty  
     - Both conditions must be true (AND)  
   - Input: Posts from Reddit search  
   - Output: Filtered posts only  
   - Edge cases: Posts without `ups` or `selftext` fields; missing data handling  

---

#### 1.2 Post Filtering & Normalization

- **Overview:**  
Transforms filtered Reddit posts into a uniform structure with key fields extracted for easier AI consumption.

- **Nodes Involved:**  
  - Value setup  

- **Node Details:**  

1. **Value setup**  
   - Type: Set node  
   - Role: Extracts and defines key post properties (`title`, `selftext`, `ups`, `created`, `url`, `subreddit_id`, `id`) as named fields  
   - Config: Assigns fields with expressions from Reddit JSON input  
   - Input: Filtered posts from If Condition  
   - Output: Posts with normalized fields  
   - Edge cases: Missing or malformed Reddit data fields  

---

#### 1.3 AI Problem Classification

- **Overview:**  
Uses an AI agent powered by Google Gemini to classify whether each Reddit post discusses a real problem faced by AI automation users or just feature requests.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Merge  

- **Node Details:**  

1. **AI Agent**  
   - Type: LangChain Agent (AI text classifier)  
   - Role: Classifies if post discusses user problems or feature requests  
   - Config: Prompt instructs answering "Yes" or "No" based on whether the post mentions a specific user problem with AI automation  
   - Key expressions: `Reddit Post: {{ $json.selftext }}`  
   - Input: Normalized post data from Value setup  
   - Output: Classification result (`Yes` or `No`) attached to JSON  
   - Edge cases: Ambiguous posts, AI misclassification, prompt failures  

2. **Google Gemini Chat Model**  
   - Type: LangChain Google Gemini LM Chat Model  
   - Role: Provides the underlying AI language model for the AI Agent  
   - Config: Default Gemini model  
   - Input/Output: Connected internally to AI Agent  
   - Edge cases: Model unavailability, API errors, rate limits  

3. **Merge**  
   - Type: Merge node (combine mode)  
   - Role: Combines original post data with AI classification output by position in array  
   - Config: Include unpaired enabled to ensure no data loss  
   - Input: AI Agent output + original posts  
   - Output: Combined data for next steps  
   - Edge cases: Mismatched array lengths, missing data  

---

#### 1.4 AI Solution Generation

- **Overview:**  
For posts classified as problems, generates concise, actionable solutions using a second AI agent with a specialized Gemini model optimized for speed.

- **Nodes Involved:**  
  - If Condition 2  
  - AI Agent1  
  - Google Gemini Chat Model1  

- **Node Details:**  

1. **If Condition 2**  
   - Type: Conditional node  
   - Role: Filters posts where the classifier output contains "Yes" (indicating a problem)  
   - Config: Checks if `$json.output` contains "Yes"  
   - Input: Combined data from Merge  
   - Output: Posts that are problems only  
   - Edge cases: Case sensitivity issues, partial matches  

2. **AI Agent1**  
   - Type: LangChain Agent  
   - Role: Generates a solution plan for the problem post, referencing the Reddit post text  
   - Config: Prompt asks for a concise plan to fix user issues with "Sora 2" based on the post text  
   - Key expressions: `Reddit Post: {{ $json.selftext }}`  
   - Input: Filtered problem posts  
   - Output: Solution text in JSON field `output`  
   - Edge cases: AI hallucination, irrelevant solutions, prompt tuning needed  

3. **Google Gemini Chat Model1**  
   - Type: LangChain Google Gemini LM Chat Model  
   - Role: Provides a faster, flash-optimized Gemini model (`gemini-2.0-flash`) for AI Agent1  
   - Config: Model name set to `models/gemini-2.0-flash`  
   - Input/Output: Connected internally to AI Agent1  
   - Edge cases: Model availability, API errors  

---

#### 1.5 Result Consolidation and Output

- **Overview:**  
Combines original post data with AI-generated solutions, appends the results to a Google Sheet for archival, and optionally posts the AI-generated comment back to the Reddit post.

- **Nodes Involved:**  
  - Merge1  
  - Append row in sheet  
  - Create a comment in a post  

- **Node Details:**  

1. **Merge1**  
   - Type: Merge node (combine mode)  
   - Role: Combines original Reddit post data with generated AI solutions, preferring solution data on field clashes  
   - Config: Clash handling set to prefer Input1 (AI Agent1 output) values  
   - Input: AI Agent1 output + original post data filtered as problems  
   - Output: Combined data for final output steps  
   - Edge cases: Data conflicts, missing fields  

2. **Append row in sheet**  
   - Type: Google Sheets node (append operation)  
   - Role: Archives key post and AI solution data into a specified Google Sheet (Sheet2)  
   - Config:  
     - Document ID: Spreadsheet ID provided  
     - Sheet Name: "Sheet2"  
     - Columns mapped:  
       - Bio ← `created` (post timestamp)  
       - username ← `selftext` (post content)  
       - Full Name ← AI solution text (`output`)  
       - Profile ID ← `title` (post title)  
       - Follower Count ← `subreddit_id` (post subreddit identifier)  
   - Input: Merged data from Merge1  
   - Output: Success/failure of append operation  
   - Edge cases: Credential issues, quota limits, column mapping errors  

3. **Create a comment in a post**  
   - Type: Reddit node (postComment resource)  
   - Role: Posts AI-generated solution as a comment to the Reddit post  
   - Config:  
     - `postId`: Currently mapped to `Bio` (likely incorrect, should be Reddit post ID)  
     - `commentText`: Mapped to `Full Name` (AI solution)  
   - Input: Output from Append row in sheet  
   - Output: Reddit API response to comment posting  
   - Edge cases: Comment posting disabled by default for safety, potential Reddit rate limits, auth errors, mapping corrections needed before enabling  

---

#### 1.6 Supportive Notes & Credentials

- **Overview:**  
Several sticky notes provide detailed documentation, usage tips, and credential requirements embedded within the workflow for ease of understanding and safe operation.

- **Nodes Involved:**  
  - Sticky Note (Manual Trigger explanation)  
  - Sticky Note2 (Reddit search filtering explanation)  
  - Sticky Note3 (Problem classifier AI explanation)  
  - Sticky Note4 (Solution generator AI explanation)  
  - Sticky Note5 (Result storage and comment posting explanation)  
  - Sticky Note - Credentials  
  - Sticky Note - How to Reuse  

- **Node Details:**  

Each sticky note contains carefully crafted documentation for its respective logical block, including:

- Recommended credential setup: Reddit OAuth2, Google Sheets OAuth2, Google Gemini API credentials  
- Safety warnings about testing with read-only actions and disabling automatic commenting until verified  
- Instructions on how to customize keyword search, AI prompt tuning, and Google Sheets mapping  
- Links to Google Sheets document as example and notes on rate limits and community rules  

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                          | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                     |
|--------------------------|----------------------------------|----------------------------------------|--------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Starts workflow manually                |                          | Post Searching             | This workflow starts with a **Manual Trigger** for easy testing. Replace with Schedule or Webhook in production. |
| Post Searching           | Reddit (search)                  | Searches Reddit posts in r/n8n          | When clicking ‘Execute workflow’ | If Condition             | Search for posts with keyword "Why i stopped using" in r/n8n.                                  |
| If Condition             | If (filter)                     | Filters posts with ups >= 2 and non-empty selftext | Post Searching           | Value setup                | Filters posts for relevance based on upvotes and content presence.                             |
| Value setup              | Set                             | Normalizes Reddit post fields           | If Condition             | AI Agent                   | Extracts key fields like title, selftext, ups, created for AI processing.                      |
| AI Agent                 | LangChain Agent                 | Classifies if post is about a user problem | Value setup              | Merge                      | AI answers Yes/No if post describes a user problem with AI automation.                        |
| Google Gemini Chat Model | LM Chat (Google Gemini)         | Provides AI model for classification    |                          | AI Agent                   | AI model used by AI Agent node.                                                                |
| Merge                    | Merge (combine)                 | Combines original post data with AI classification | AI Agent + Value setup    | If Condition 2             | Combines AI result with original post data.                                                   |
| If Condition 2           | If (filter)                     | Filters posts classified as problems    | Merge                    | AI Agent1                  | Passes posts with classification output containing "Yes".                                    |
| AI Agent1                | LangChain Agent                 | Generates solutions for problem posts   | If Condition 2           | Merge1                     | Generates concise plans to fix user issues based on Reddit post.                              |
| Google Gemini Chat Model1| LM Chat (Google Gemini)         | Provides AI model for solution generation |                          | AI Agent1                  | Uses Gemini 2.0 Flash model for faster responses.                                              |
| Merge1                   | Merge (combine)                 | Combines problem posts with solution output | AI Agent1 + If Condition 2 | Append row in sheet        | Merges AI-generated solution with original post data.                                         |
| Append row in sheet      | Google Sheets (append)          | Archives posts and solutions             | Merge1                   | Create a comment in a post  | Stores key data and AI outputs in Google Sheets for review.                                   |
| Create a comment in a post | Reddit (postComment)           | Posts AI-generated comment on Reddit post | Append row in sheet       |                            | Disabled by default for safety; maps comment text and postId to Reddit fields.                |
| Sticky Note              | Sticky Note                    | Manual trigger explanation               |                          |                            | Explains manual trigger usage for template testing.                                           |
| Sticky Note2             | Sticky Note                    | Reddit search and filtering explanation  |                          |                            | Describes post filtering logic for quality control.                                          |
| Sticky Note3             | Sticky Note                    | AI problem classifier explanation        |                          |                            | Details AI classification logic and prompt.                                                  |
| Sticky Note4             | Sticky Note                    | AI solution generation explanation        |                          |                            | Describes solution generation and model selection.                                           |
| Sticky Note5             | Sticky Note                    | Result storage and Reddit comment explanation |                          |                            | Explains data archiving and commenting features with safety notes.                           |
| Sticky Note - Credentials| Sticky Note                    | Credentials and safety instructions       |                          |                            | Lists required credentials and safety tips for testing.                                      |
| Sticky Note - How to Reuse| Sticky Note                    | Customization and reuse instructions      |                          |                            | Explains how to adapt the workflow for other subreddits, keywords, and AI prompts.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Allows manual execution of the workflow for testing  
   - No parameters needed  

2. **Add Reddit Node - Search Operation**  
   - Type: Reddit node  
   - Operation: Search posts  
   - Parameters:  
     - Subreddit: `n8n`  
     - Keyword: `"Why i stopped using"`  
     - Limit: 10 posts  
   - Connect output of Manual Trigger to this node  
   - Configure Reddit OAuth2 credentials  

3. **Add If Node for Filtering Posts**  
   - Type: If  
   - Conditions (AND):  
     - `ups` (upvotes) >= 2  
     - `selftext` is not empty  
   - Connect output of Reddit Search to this node  

4. **Add Set Node for Normalizing Data**  
   - Type: Set  
   - Define fields with expressions:  
     - `title` = `{{$json["title"]}}`  
     - `selftext` = `{{$json["selftext"]}}`  
     - `ups` = `{{$json["ups"]}}`  
     - `created` = `{{$node["Post Searching"].json["created"]}}`  
     - `url` = `{{$json["url"]}}`  
     - `subreddit_id` = `{{$json["subreddit_id"]}}`  
     - `id` = `{{$json["id"]}}`  
   - Connect output of If filter to this node  

5. **Add Google Gemini Chat Model Node (Classifier)**  
   - Type: LangChain Google Gemini LM Chat  
   - Default model (no special model name)  

6. **Add AI Agent Node (Classifier)**  
   - Type: LangChain Agent  
   - Prompt:  
     ```
     Define whether the reddit post is talking about the problems faced by the users of AI Automation or they need a solution on how they want new features in AI Automation. The post should mention a specific problem faced by the users.

     Reddit Post: {{ $json.selftext }}

     Is this post about a problem that users are facing about AI Automation or do they need new features? Just answer with Yes or No.
     ```  
   - Connect AI Agent to Google Gemini Chat Model node  
   - Connect Set node output to AI Agent input  

7. **Add Merge Node (Combine)**  
   - Mode: Combine by position  
   - Include unpaired enabled  
   - Connect AI Agent output and Set node output as inputs  
   - Output will have combined post and classification data  

8. **Add If Node for Problem Detection**  
   - Type: If  
   - Condition: `$json.output` contains "Yes" (case sensitive)  
   - Input: Merge node output  

9. **Add Google Gemini Chat Model Node (Solution Generator)**  
   - Type: LangChain Google Gemini LM Chat  
   - Model name: `models/gemini-2.0-flash`  

10. **Add AI Agent Node (Solution Generator)**  
    - Type: LangChain Agent  
    - Prompt:  
      ```
      Based on the reddit post, suggest a plan or a solution on how I fix the issues the users are facing with Sora 2.

      Reddit Post: {{ $json.selftext }}

      Provide a concise solution on how we can fix the problems in slack based on the reddit post. Explain the solution.
      ```  
    - Connect AI Agent1 to Google Gemini Chat Model1  
    - Connect If Condition 2 output to AI Agent1 input  

11. **Add Merge Node (Combine) for Solution**  
    - Mode: Combine by position  
    - Clash handling: prefer Input1 values  
    - Connect AI Agent1 output and If Condition 2 output (original post data) as inputs  

12. **Add Google Sheets Node (Append Row)**  
    - Operation: Append  
    - Document ID: Provide your Google Sheets document ID  
    - Sheet Name: `Sheet2` (or your preferred sheet)  
    - Map columns:  
      - Bio: `={{ $json.created }}`  
      - username: `={{ $json.selftext }}`  
      - Full Name: `={{ $json.output }}` (AI generated solution)  
      - Profile ID,: `={{ $json.title }}`  
      - Follower Count: `={{ $json.subreddit_id }}`  
    - Connect Merge1 output to Google Sheets node  
    - Configure Google Sheets OAuth2 credentials  

13. **Add Reddit Node (Post Comment)**  
    - Resource: postComment  
    - Parameters:  
      - postId: Map to Reddit post ID field (should be `id`, verify and correct mapping)  
      - commentText: Map to AI generated solution (`output` or `Full Name`)  
    - Connect Google Sheets output to this node  
    - Configure Reddit OAuth2 credentials  
    - Keep disabled until testing complete to avoid unintended posting  

14. **Add Sticky Notes (Optional)**  
    - Add sticky notes in your n8n editor to document each block’s purpose, credential requirements, and usage tips as per the original workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow starts with a **Manual Trigger** for testing ease but can be replaced by Schedule or Webhook triggers in production. | Sticky Note explaining trigger setup.                                                                 |
| Filtering posts by `ups >= 2` and non-empty `selftext` improves AI classification accuracy by focusing on relevant content.       | Sticky Note2 describing Reddit search filters.                                                        |
| AI classification prompt focuses on distinguishing between user problems and feature requests related to AI Automation.           | Sticky Note3 detailing AI problem classifier logic.                                                   |
| Solution generation prompt asks for concise, actionable plans based on Reddit post content, using a fast Gemini model variant.     | Sticky Note4 explaining solution generation.                                                          |
| Google Sheets archival enables auditing and tracking of AI-generated solutions alongside original Reddit posts.                   | Sticky Note5 describing data archiving and optional Reddit commenting.                                |
| Credentials needed: Reddit OAuth2 for Reddit nodes, Google Sheets OAuth2 for Sheets node, Google Gemini API credentials for AI nodes. | Sticky Note - Credentials.                                                                             |
| Customize subreddit, keywords, AI prompts, and Google Sheets mapping to target your own use case and community.                    | Sticky Note - How to Reuse / Customize.                                                               |
| Test extensively with read-only operations first; keep Reddit comment posting disabled until confident with outputs and rate limits. | General best practice recommended in Sticky Notes.                                                    |

---

**Disclaimer:**  
The text provided is derived solely from an n8n automated workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data manipulated is lawful and publicly available.