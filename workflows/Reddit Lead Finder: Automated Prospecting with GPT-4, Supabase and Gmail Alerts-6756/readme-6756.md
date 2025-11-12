Reddit Lead Finder: Automated Prospecting with GPT-4, Supabase and Gmail Alerts

https://n8nworkflows.xyz/workflows/reddit-lead-finder--automated-prospecting-with-gpt-4--supabase-and-gmail-alerts-6756


# Reddit Lead Finder: Automated Prospecting with GPT-4, Supabase and Gmail Alerts

### 1. Workflow Overview

This workflow automates the process of prospecting leads from Reddit posts by leveraging GPT-4 for content analysis, Supabase for data storage and deduplication, and Gmail for notification alerts. It targets users who want to discover relevant Reddit threads that might benefit from a specific product or service, streamline lead qualification, and maintain organized records with automated email alerts.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Defines the subreddits and keywords to search and triggers execution on schedule or manual start.
- **1.2 Reddit Data Retrieval and Filtering:** Searches Reddit posts based on parameters, filters out low-value posts, and batches results for processing.
- **1.3 Duplicate Detection and Data Enrichment:** Checks the posts against Supabase database to detect duplicates and enrich data accordingly.
- **1.4 AI-Based Content Analysis:** Uses GPT-4 to evaluate whether each Reddit post is relevant for the product offering.
- **1.5 Data Recording and Notification:** Stores relevant posts in Supabase and Google Sheets, then sends summarized email alerts via Gmail.
- **1.6 Control Flow and Batching:** Manages iterative processing with batch loops, conditional branching, and wait nodes to control execution pacing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** This block initializes the process by defining subreddits and keywords to search, and triggers the workflow via schedule or manual trigger.
- **Nodes Involved:**  
  - `Schedule Trigger`  
  - `When clicking ‘Test workflow’`  
  - `Code`

- **Node Details:**

1. **Schedule Trigger**  
   - *Type:* Schedule Trigger  
   - *Role:* Automatically triggers the workflow daily at 8 AM.  
   - *Configuration:* Set to trigger hourly at 8:00 (hourly interval array with triggerAtHour = 8).  
   - *Connections:* Outputs to `Code`.  
   - *Failures:* Possible timing misconfiguration; ensure timezone settings align with desired schedule.

2. **When clicking ‘Test workflow’**  
   - *Type:* Manual Trigger  
   - *Role:* Allows manual execution for testing purposes.  
   - *Configuration:* Default (no parameters).  
   - *Connections:* Outputs to `Code`.  
   - *Failures:* None expected.

3. **Code**  
   - *Type:* Code (JavaScript)  
   - *Role:* Provides subreddit and keyword definitions for search.  
   - *Configuration:* Returns two JSON objects, each with `subreddit` and `keywords` fields. Placeholder keywords are `"....."` and subreddits are `"smallbusiness"` and `"startup"`.  
   - *Key Expressions:* Hardcoded subreddit and keyword pairs.  
   - *Input:* None, triggered by Schedule or Manual.  
   - *Output:* Feeds into `Loop Over Items`.  
   - *Failures:* If modified, ensure valid JavaScript syntax.

---

#### 2.2 Reddit Data Retrieval and Filtering

- **Overview:** Searches Reddit for posts matching keywords in specified subreddits, filters posts with no upvotes or empty content, and batches them for further processing.
- **Nodes Involved:**  
  - `Loop Over Items`  
  - `Get Posts`  
  - `Filter Posts`  
  - `Loop Over Items1`

- **Node Details:**

1. **Loop Over Items**  
   - *Type:* SplitInBatches  
   - *Role:* Processes each subreddit and keyword pair sequentially to control data flow.  
   - *Configuration:* Default batch size, no reset.  
   - *Input:* From `Code`.  
   - *Output:* Splits into individual subreddit/keyword JSONs to `Get Posts` and `Filter Posts`.  
   - *Failures:* Large batch sizes or missing data can cause delays.

2. **Get Posts**  
   - *Type:* Reddit  
   - *Role:* Searches Reddit posts by keyword and subreddit, sorting by newest, limited to 3 posts per search.  
   - *Configuration:*  
     - Operation: Search  
     - Limit: 3  
     - Sort: new  
     - Keyword and subreddit are dynamic expressions from input JSON.  
   - *Credentials:* OAuth2 Reddit API.  
   - *Input:* From `Loop Over Items`.  
   - *Output:* Posts flow to `Loop Over Items` (for batch splitting).  
   - *Failures:* API rate limits, authentication errors.

3. **Filter Posts**  
   - *Type:* If  
   - *Role:* Filters posts to only keep those with more than 0 upvotes and non-empty selftext.  
   - *Conditions:*  
     - `ups` > 0  
     - `selftext` not empty  
   - *Input:* From `Loop Over Items`.  
   - *Output:* True branch to `Loop Over Items1`.  
   - *Failures:* Expression errors if fields missing.

4. **Loop Over Items1**  
   - *Type:* SplitInBatches  
   - *Role:* Further splits filtered posts for individual processing.  
   - *Configuration:* No reset.  
   - *Input:* From `Filter Posts`.  
   - *Output:* To `If` node for duplicate check and `Get many rows`.  
   - *Failures:* Large batch sizes may slow processing.

---

#### 2.3 Duplicate Detection and Data Enrichment

- **Overview:** Checks if a Reddit post is already stored in Supabase (deduplication) and marks duplicates to avoid redundant processing.
- **Nodes Involved:**  
  - `Get many rows`  
  - `Code1`  
  - `If`

- **Node Details:**

1. **Get many rows**  
   - *Type:* Supabase  
   - *Role:* Queries `reddit_posts` table for existing entries matching current post ID and subreddit.  
   - *Filters:* `post_id` equals current post's `id`, and `subreddit` equals current post's `subreddit`.  
   - *Credentials:* Supabase API with production environment.  
   - *Input:* From `Loop Over Items1`.  
   - *Output:* To `Code1`.  
   - *Failures:* Connection errors, query failures.

2. **Code1**  
   - *Type:* Code (JavaScript)  
   - *Role:* Determines if current post is a duplicate based on whether any Supabase rows were returned.  
   - *Logic:* Sets `duplicate` property true if query result contains any keys; false otherwise.  
   - *Input:* From `Get many rows`.  
   - *Output:* To `If`.  
   - *Failures:* Logic errors if input structure changes.

3. **If**  
   - *Type:* If  
   - *Role:* Branches posts based on `duplicate` flag - false branch continues processing new posts.  
   - *Condition:* `duplicate` equals false.  
   - *Input:* From `Code1`.  
   - *Output:* True branch to `Loop Over Items2` (next block).  
   - *Failures:* Expression errors if `duplicate` missing.

---

#### 2.4 AI-Based Content Analysis

- **Overview:** Evaluates whether each Reddit post is relevant to the product using GPT-4, producing a binary yes/no output.
- **Nodes Involved:**  
  - `Loop Over Items2`  
  - `Wait1`  
  - `Analysis Content  By AI`  
  - `OpenAI Chat Model`  
  - `Code3`  
  - `If1`

- **Node Details:**

1. **Loop Over Items2**  
   - *Type:* SplitInBatches  
   - *Role:* Batches posts for sequential AI analysis.  
   - *Input:* From `If` (non-duplicates).  
   - *Output:* To `If1` and `Wait1`.  
   - *Failures:* Large batch size may cause API overload.

2. **Wait1**  
   - *Type:* Wait  
   - *Role:* Adds a 1-second delay before AI analysis to manage rate limits.  
   - *Input:* From `Loop Over Items2`.  
   - *Output:* To `Analysis Content  By AI`.  
   - *Failures:* Delays may accumulate in large batches.

3. **Analysis Content  By AI**  
   - *Type:* LangChain Agent (AI Agent)  
   - *Role:* Sends Reddit post text to GPT-4 with prompt asking if the post benefits from the product, expects yes/no answer.  
   - *Prompt:*  
     ```
     Reddit post: {{ $('Loop Over Items1').item.json.selftext }}

     Is this a post that might benefit from [PRODUCT]?

     Output only yes or no
     ```  
   - *Input:* From `Wait1`.  
   - *Output:* To `Code3`.  
   - *Credentials:* Uses linked OpenAI API credentials.  
   - *Failures:* API errors, prompt misformatting.

4. **OpenAI Chat Model**  
   - *Type:* LangChain LM Chat OpenAI  
   - *Role:* Provides the GPT-4.1 language model backend for the AI node.  
   - *Parameters:* Model set to "gpt-4.1" with max tokens 3000.  
   - *Credentials:* OpenAI API credentials.  
   - *Input:* From `Analysis Content  By AI`.  
   - *Output:* Back to `Analysis Content  By AI`.  
   - *Failures:* Rate limits, authentication errors.

5. **Code3**  
   - *Type:* Code (JavaScript)  
   - *Role:* Merges AI output (`output`) with original post JSON under `relevant` field.  
   - *Input:* From `Analysis Content  By AI` and `Loop Over Items2`.  
   - *Output:* To `If1`.  
   - *Failures:* Reference errors if input missing.

6. **If1**  
   - *Type:* If  
   - *Role:* Checks if AI response is "Yes" (case-sensitive contains).  
   - *Condition:* `relevant` contains "Yes".  
   - *Input:* From `Code3`.  
   - *Output:* True branch to `Loop Over Items3`.  
   - *Failures:* Case sensitivity may cause false negatives.

---

#### 2.5 Data Recording and Notification

- **Overview:** Stores relevant posts in Supabase and Google Sheets, compiles an email summary, and sends notification via Gmail.
- **Nodes Involved:**  
  - `Loop Over Items3`  
  - `Code4`  
  - `Append row in sheet1`  
  - `Code5`  
  - `Create a row`  
  - `Code6`  
  - `Send a message`

- **Node Details:**

1. **Loop Over Items3**  
   - *Type:* SplitInBatches  
   - *Role:* Processes relevant posts individually for storage and email preparation.  
   - *Input:* From `If1`.  
   - *Output:* Two parallel outputs to `Code4` and `Code5`.  
   - *Failures:* Batch size affects throughput.

2. **Code4**  
   - *Type:* Code (JavaScript)  
   - *Role:* Creates HTML-formatted email content by concatenating post titles and URLs across all input items.  
   - *Input:* From `Loop Over Items3`.  
   - *Output:* To `Send a message`.  
   - *Failures:* Empty inputs produce empty emails.

3. **Append row in sheet1**  
   - *Type:* Google Sheets  
   - *Role:* Appends post data to a specific Google Sheet for record keeping.  
   - *Parameters:*  
     - Columns appended: URL, Date (current date-time), Post snippet (first 200 chars), Subreddit.  
     - Document ID and Sheet ID linked to a specific Google Sheet.  
   - *Credentials:* Google Sheets OAuth2.  
   - *Input:* From `Loop Over Items3`.  
   - *Output:* None downstream.  
   - *Failures:* API quotas, invalid sheet ID.

4. **Code5**  
   - *Type:* Code (JavaScript)  
   - *Role:* Passes through the items unchanged to the next node.  
   - *Input:* From `Loop Over Items3`.  
   - *Output:* To `Create a row`.  
   - *Failures:* None expected.

5. **Create a row**  
   - *Type:* Supabase  
   - *Role:* Inserts a new row in the `reddit_posts` Supabase table with post details for persistence.  
   - *Fields:*  
     - `post_id` from post `id`  
     - `username` hardcoded as "optl12"  
     - `subreddit` from post  
     - `url` from post  
   - *Credentials:* Supabase API production.  
   - *Input:* From `Code5`.  
   - *Output:* To `Code6`.  
   - *Failures:* DB constraints, connection errors.

6. **Code6**  
   - *Type:* Code (JavaScript)  
   - *Role:* Selects the first item from `Code5` output for further processing (feeds back to `Loop Over Items3`).  
   - *Input:* From `Create a row`.  
   - *Output:* To `Loop Over Items3`.  
   - *Failures:* Empty input leads to errors.

7. **Send a message**  
   - *Type:* Gmail  
   - *Role:* Sends an email alert with relevant Reddit threads to a fixed recipient email.  
   - *Parameters:*  
     - To: team@bluepro.app  
     - Subject: "Relevant Reddit Threads"  
     - Message: HTML content from `Code4`.  
   - *Credentials:* Gmail OAuth2.  
   - *Input:* From `Code4`.  
   - *Output:* None.  
   - *Failures:* Authentication errors, Gmail API limits.

---

#### 2.6 Control Flow and Batching

- **Overview:** Manages the iteration, batching, and pacing between nodes to ensure smooth execution.
- **Nodes Involved:**  
  - `Loop Over Items`  
  - `Loop Over Items1`  
  - `Loop Over Items2`  
  - `Loop Over Items3`  
  - `Wait1`

- **Node Details:**

All `Loop Over Items*` nodes are SplitInBatches nodes that handle the processing of arrays in manageable chunks to avoid overload and rate limits. The `Wait1` node inserts a delay to throttle requests to the AI service.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                             | Input Node(s)                  | Output Node(s)                  | Sticky Note                        |
|---------------------------|--------------------------------|---------------------------------------------|--------------------------------|---------------------------------|----------------------------------|
| Schedule Trigger          | Schedule Trigger               | Starts workflow automatically at 8 AM       | -                              | Code                           |                                  |
| When clicking ‘Test workflow’ | Manual Trigger               | Allows manual workflow start                 | -                              | Code                           |                                  |
| Code                      | Code                           | Defines subreddit and keyword inputs         | Schedule Trigger, Manual Trigger | Loop Over Items                |                                  |
| Loop Over Items           | SplitInBatches                 | Batches subreddit/keyword pairs              | Code                           | Get Posts, Filter Posts         |                                  |
| Get Posts                 | Reddit                         | Searches Reddit posts by keyword/subreddit  | Loop Over Items                | Loop Over Items                |                                  |
| Filter Posts              | If                             | Filters posts with upvotes and non-empty text | Loop Over Items                | Loop Over Items1                |                                  |
| Loop Over Items1          | SplitInBatches                 | Processes filtered posts individually        | Filter Posts                   | If, Get many rows               |                                  |
| Get many rows             | Supabase                       | Queries DB for duplicate posts                | Loop Over Items1               | Code1                         |                                  |
| Code1                     | Code                           | Flags duplicates if found in DB               | Get many rows                  | If                            |                                  |
| If                        | If                             | Allows only non-duplicate posts to proceed    | Code1                         | Loop Over Items2                |                                  |
| Loop Over Items2          | SplitInBatches                 | Batches posts for AI analysis                  | If                            | If1, Wait1                    |                                  |
| Wait1                     | Wait                           | Delays execution for rate limiting            | Loop Over Items2               | Analysis Content  By AI         |                                  |
| Analysis Content  By AI    | LangChain Agent (AI)           | Uses GPT-4 to analyze post relevance           | Wait1                         | Code3                         |                                  |
| OpenAI Chat Model         | LM Chat OpenAI                 | Provides GPT-4.1 model backend                 | Analysis Content  By AI        | Analysis Content  By AI         |                                  |
| Code3                     | Code                           | Merges AI output with post data                | Analysis Content  By AI        | If1                           |                                  |
| If1                       | If                             | Filters posts where AI output is “Yes”         | Code3                         | Loop Over Items3               |                                  |
| Loop Over Items3          | SplitInBatches                 | Processes relevant posts for storage and emails | If1                           | Code4, Code5                   |                                  |
| Code4                     | Code                           | Formats email content HTML from posts          | Loop Over Items3               | Send a message                 |                                  |
| Append row in sheet1      | Google Sheets                  | Appends post data to Google Sheet              | Loop Over Items3               | -                             |                                  |
| Code5                     | Code                           | Passes posts through for DB insertion          | Loop Over Items3               | Create a row                  |                                  |
| Create a row              | Supabase                       | Inserts post info into Supabase DB              | Code5                         | Code6                         |                                  |
| Code6                     | Code                           | Selects first item to feed back into Loop      | Create a row                  | Loop Over Items3               |                                  |
| Send a message            | Gmail                          | Sends email alerts about relevant Reddit posts | Code4                         | -                             |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 08:00 hours (adjust timezone if needed).

2. **Create a Manual Trigger node:**  
   - Type: Manual Trigger  
   - No additional configuration needed.

3. **Create a Code node:**  
   - Type: Code (JavaScript)  
   - Paste code returning subreddit and keywords as JSON array objects:  
     ```javascript
     return [
       { json: { subreddit: "smallbusiness", keywords: "....." }},
       { json: { subreddit: "startup", keywords: "....." }}
     ];
     ```  
   - Connect both `Schedule Trigger` and `Manual Trigger` outputs to this node.

4. **Create a SplitInBatches node (Loop Over Items):**  
   - Default batch size.  
   - Connect input from Code node.

5. **Create a Reddit node (Get Posts):**  
   - Operation: Search  
   - Limit: 3  
   - Subreddit: Expression `={{ $json.subreddit }}`  
   - Keyword: Expression `={{ $json.keywords }}`  
   - Sort by 'new'  
   - Set Reddit OAuth2 credentials.  
   - Connect input from Loop Over Items.

6. **Create an If node (Filter Posts):**  
   - Conditions:  
     - `$json.ups` > 0  
     - `$json.selftext` NOT empty  
   - Connect input from Loop Over Items.  
   - True branch proceeds to next node.

7. **Create a SplitInBatches node (Loop Over Items1):**  
   - Default batch size.  
   - Connect input from Filter Posts true output.

8. **Create a Supabase node (Get many rows):**  
   - Operation: Get All  
   - Table: `reddit_posts`  
   - Filters:  
     - `post_id` equals `={{ $json.id }}`  
     - `subreddit` equals `={{ $json.subreddit }}`  
   - Set Supabase credentials.  
   - Connect input from Loop Over Items1.

9. **Create a Code node (Code1):**  
   - JavaScript to flag duplicates:  
     ```javascript
     const original = $input.item.json;
     return [{
       json: {
         ...original,
         duplicate: Object.keys($input.all()[0].json).length ? true : false
       }
     }];
     ```  
   - Connect input from Get many rows.

10. **Create an If node:**  
    - Condition: `duplicate` equals false  
    - Connect input from Code1.  
    - True branch proceeds.

11. **Create a SplitInBatches node (Loop Over Items2):**  
    - Default batch size.  
    - Connect input from If true output.

12. **Create a Wait node (Wait1):**  
    - Delay: 1 second  
    - Connect input from Loop Over Items2.

13. **Create a LangChain Agent node (Analysis Content By AI):**  
    - Prompt type: Define  
    - Text:  
      ```
      Reddit post: {{ $('Loop Over Items1').item.json.selftext }}

      Is this a post that might benefit from [PRODUCT]?

      Output only yes or no
      ```  
    - Connect input from Wait1.  
    - Link to OpenAI Chat Model node.

14. **Create a LangChain LM Chat OpenAI node (OpenAI Chat Model):**  
    - Model: gpt-4.1  
    - Max tokens: 3000  
    - Set OpenAI API credentials.  
    - Connect output to Analysis Content By AI.

15. **Create a Code node (Code3):**  
    - JavaScript to merge AI output:  
      ```javascript
      const original = $('Loop Over Items2').item.json;
      original.relevant = $input.first().json.output;
      return [{ json: original }];
      ```  
    - Connect input from Analysis Content By AI.

16. **Create an If node (If1):**  
    - Condition: `relevant` contains "Yes" (case-sensitive)  
    - Connect input from Code3.  
    - True branch proceeds.

17. **Create a SplitInBatches node (Loop Over Items3):**  
    - Default batch size.  
    - Connect input from If1 true output.

18. **Create a Code node (Code4):**  
    - JavaScript to create email HTML content:  
      ```javascript
      let email = '';
      for (const item of $input.all()) {
        email += '<p>' + item.json.title + ' - ' + item.json.url + '</p>\n\n';
      }
      return { email };
      ```  
    - Connect input from Loop Over Items3.

19. **Create a Google Sheets node (Append row in sheet1):**  
    - Operation: Append  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: `gid=0` or relevant sheet  
    - Columns: URL, Date (current local time), Post (first 200 chars), Subreddit  
    - Set Google Sheets OAuth2 credentials.  
    - Connect input from Loop Over Items3.

20. **Create a Code node (Code5):**  
    - JavaScript: `return $input.all();` (pass-through)  
    - Connect input from Loop Over Items3.

21. **Create a Supabase node (Create a row):**  
    - Operation: Insert  
    - Table: `reddit_posts`  
    - Fields:  
      - `post_id`: `={{ $json.id }}`  
      - `username`: "optl12" (hardcoded)  
      - `subreddit`: `={{ $json.subreddit }}`  
      - `url`: `={{ $json.url }}`  
    - Set Supabase credentials.  
    - Connect input from Code5.

22. **Create a Code node (Code6):**  
    - JavaScript: `return $('Code5').first();`  
    - Connect input from Create a row.  
    - Connect output back to Loop Over Items3 to continue batching.

23. **Create a Gmail node (Send a message):**  
    - To: team@bluepro.app  
    - Subject: "Relevant Reddit Threads"  
    - Message: Use expression `{{ $json.email }}` from Code4  
    - Set Gmail OAuth2 credentials.  
    - Connect input from Code4.

24. **Connect all nodes according to the flow described in section 2.**

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                                          |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| The workflow uses LangChain nodes for OpenAI GPT-4 interaction, requiring OpenAI API credentials.    | https://docs.n8n.io/integrations/ai/langchain/                                                                         |
| Google Sheets integration requires OAuth2 setup with appropriate permissions for appending rows.     | https://docs.n8n.io/integrations/applications/google/google-sheets/                                                     |
| Supabase nodes require API keys and proper table schema (`reddit_posts` with fields post_id, subreddit, url, username). | https://supabase.com/docs/reference/javascript/insert                                                                    |
| Gmail node requires OAuth2 credentials with send email scope.                                         | https://docs.n8n.io/integrations/applications/google/gmail/                                                              |
| Reddit API OAuth2 authentication is mandatory for accessing Reddit search endpoints.                  | https://www.reddit.com/wiki/api/oauth                                                                               |
| This workflow is designed for automated daily prospecting with manual test capability.                 |                                                                                                                         |

---

This document provides a comprehensive analysis and stepwise reproduction guide for the "Reddit Lead Finder: Automated Prospecting with GPT-4, Supabase and Gmail Alerts" workflow, enabling advanced users and AI agents to understand, reproduce, and extend the workflow reliably.