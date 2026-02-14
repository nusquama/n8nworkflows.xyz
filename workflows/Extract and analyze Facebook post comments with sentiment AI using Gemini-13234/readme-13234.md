Extract and analyze Facebook post comments with sentiment AI using Gemini

https://n8nworkflows.xyz/workflows/extract-and-analyze-facebook-post-comments-with-sentiment-ai-using-gemini-13234


# Extract and analyze Facebook post comments with sentiment AI using Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Extract and analyze Facebook post comments with sentiment AI using Gemini  
**Workflow name (in JSON):** Analyze All Facebook Post Comments

**Purpose:**  
Fetch **all comments** from a specific Facebook Page post (with pagination), run **AI sentiment analysis** on each comment using **Google Gemini**, and **store/update** results in **Google Sheets** (deduplicated by Comment ID).

**Execution modes (two entry points):**
1. **Manual mode:** User runs the workflow, sets a Post ID, fetches the post + all comments (paged), and delegates comment-batch processing to a reusable “processing” path (here represented by an Execute Workflow call, but effectively it routes into the batch processing flow).
2. **Triggered/sub-workflow mode:** Another workflow sends comment payloads into this workflow; it splits and processes them in batches.

### Logical blocks
**1.1 Manual Input & Post Context**  
Manual start → set Post ID → fetch post metadata.

**1.2 Comment Extraction + Pagination**  
Fetch comments → detect “next page” cursor → keep fetching until all comments are retrieved.

**1.3 Batch Processing (Split + Loop)**  
Normalize comments array to items → iterate through items in batches.

**1.4 AI Sentiment Analysis (Gemini)**  
Send each comment text to Gemini-backed sentiment node → get category.

**1.5 Persistence + Rate Safety**  
Append-or-update into Google Sheets using COMMENT ID as key → wait to reduce API pressure → continue.

---

## 2. Block-by-Block Analysis

### 2.1 Manual Input & Post Context
**Overview:** Starts the workflow manually and prepares the Facebook Post ID used to retrieve the post and its comments.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set Fb post ID
- Get Fb Post

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger; entry point for ad-hoc runs.
- **Config:** No parameters.
- **Outputs:** Triggers **Set Fb post ID**.
- **Failure / edge cases:** None (only manual execution).

#### Node: Set Fb post ID
- **Type / role:** Set node; stores the post identifier used in Graph API calls.
- **Config choices:** Creates a string field:
  - `post_id` (currently empty in JSON; must be filled by the user)
- **Key variables/expressions:** None (static assignment).
- **Output:** Feeds **Get Fb Post**.
- **Failure / edge cases:**
  - Empty or invalid `post_id` will cause Facebook Graph API errors downstream.

#### Node: Get Fb Post
- **Type / role:** Facebook Graph API; fetches post metadata (context/verification).
- **Config choices:**
  - **Node (object id):** `={{ $json.post_id }}`
  - **Fields requested:** `id,message,created_time,permalink_url,attachments{title,description,media_type,url,unshimmed_url}`
  - **Graph API version:** `v21.0`
  - **Credentials:** Facebook Graph API credential “Facebook Graph (GF.diretta)”
- **Connections:** Output → **Get Fb comments**
- **Failure / edge cases:**
  - OAuth/token expired, missing permissions (e.g., Page read permissions).
  - Post not found or not accessible by the token.
  - Fields may be missing depending on post type (e.g., no attachments).

---

### 2.2 Comment Extraction + Pagination
**Overview:** Retrieves comments for the post and loops through pages using the `paging.cursors.after` cursor to ensure **all comments** are collected.

**Nodes involved:**
- Get Fb comments
- Next page?
- Get next comments page
- Call 'Facebook' (Execute Workflow)

#### Node: Get Fb comments
- **Type / role:** Facebook Graph API; reads the `comments` edge for the post.
- **Config choices:**
  - **Edge:** `comments`
  - **Node (post id):** `={{ $('Set Fb post ID').item.json.post_id }}`
  - **Fields requested per comment:** `id,from,message,created_time,comment_count`
  - **Query parameters:**
    - `order = reverse_chronological`
    - `after = {{$json.after || ""}}` (cursor-driven pagination)
    - `summary = true`
  - **Graph API version:** `v21.0`
  - **Credentials:** same Facebook Graph API credential
- **Outputs:**
  - Branch 1 → **Call 'Facebook'** (to process data)
  - Branch 2 → **Next page?** (to decide whether to request another page)
- **Failure / edge cases:**
  - Rate limiting (Facebook API throttling).
  - Cursor field may be missing; comments may be disabled.
  - Some comments can lack `message` (e.g., stickers/attachments-only), impacting sentiment input.

#### Node: Next page?
- **Type / role:** IF node; checks whether another page exists.
- **Condition logic:** String “exists” check on:
  - `={{ $('Get Fb comments').item.json.paging.cursors.after }}`
- **True output:** → **Get next comments page**
- **False output:** (not connected; pagination stops naturally)
- **Failure / edge cases:**
  - If the response structure differs (no `paging`), the expression can evaluate to non-existent; the IF is designed to check existence but upstream schema changes can still break.

#### Node: Get next comments page
- **Type / role:** Set node; prepares the `after` cursor for the next request.
- **Config choices:**
  - Sets `after = {{ $json.paging.cursors.after }}`
- **Output:** → **Get Fb comments** (creates the pagination loop)
- **Failure / edge cases:**
  - If `paging.cursors.after` is undefined, it will set an empty/invalid cursor and may cause repeated pages or API errors.

#### Node: Call 'Facebook'
- **Type / role:** Execute Workflow; delegates processing to another workflow (sub-workflow).
- **Config choices:**
  - `waitForSubWorkflow = true` (synchronous; main workflow waits for completion)
  - **Important:** The JSON does not show a `workflowId` selection; in the n8n UI this must point to an existing workflow (likely the same workflow’s triggered path or a dedicated processor).
- **Input:** The raw output from **Get Fb comments** (likely includes `data[]` plus `paging`).
- **Output:** Not explicitly used downstream in this workflow’s manual path (processing happens in the called workflow).
- **Failure / edge cases:**
  - No workflow selected / wrong workflow selected.
  - The called workflow expects a specific input shape (typically `{ data: [...] }`).
  - If the sub-workflow errors, pagination may still continue unless errors stop execution (depends on n8n error settings).

**Sub-workflow reference:** This node **invokes** a separate workflow named “Facebook” (as per node name). It should accept the comments payload and return success/failure.

---

### 2.3 Triggered/Sub-workflow Entry + Normalization
**Overview:** Allows this workflow to be run as a processing component: it receives comment pages from another workflow, extracts the comment array, and prepares items for batching.

**Nodes involved:**
- When Executed by Another Workflow
- Split Out
- Loop Over Items

#### Node: When Executed by Another Workflow
- **Type / role:** Execute Workflow Trigger; entry point for sub-workflow usage.
- **Config choices:** `inputSource = passthrough` (keeps inbound JSON as-is).
- **Output:** → **Split Out**
- **Failure / edge cases:**
  - Upstream workflow may not send the expected structure (must include `data` array for the next node).

#### Node: Split Out
- **Type / role:** Split Out; converts an array field into individual n8n items.
- **Config choices:**
  - `fieldToSplitOut = data`
- **Input expectation:** A JSON object containing `data: [ ...comments ]`
- **Output:** One item per comment → **Loop Over Items**
- **Failure / edge cases:**
  - If `data` is missing or not an array, the node will output nothing or error depending on runtime.

#### Node: Loop Over Items
- **Type / role:** Split In Batches; processes items in controllable batches.
- **Config choices:** Batch options not set (defaults apply; in many n8n versions default batch size is 1 unless configured).
- **Connections:**
  - **Main output (index 1)** → **Sentiment Analysis** (this is the “current batch/item” path)
  - **Main output (index 0)** unused here (often used to continue/finish loops)
- **Failure / edge cases:**
  - If batch size is too large, may hit rate limits (Gemini/Sheets).
  - Miswiring: Only output index 1 is connected, so ensure it’s the correct output for “items to process” in your n8n version.

---

### 2.4 AI Sentiment Analysis (Gemini)
**Overview:** Uses Google Gemini as the language model behind a Sentiment Analysis node, classifying each comment as Positive/Neutral/Negative.

**Nodes involved:**
- Google Gemini Chat Model
- Sentiment Analysis

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain LLM connector for Google Gemini; provides the model to downstream AI nodes.
- **Config choices:**
  - Uses credential: “Google Gemini(PaLM) Api account”
  - Options empty (defaults: model/version chosen by credential/node defaults)
- **Connection:** `ai_languageModel` output → **Sentiment Analysis** `ai_languageModel` input.
- **Failure / edge cases:**
  - Invalid/expired API key.
  - Model availability/region restrictions.
  - Quota limits.

#### Node: Sentiment Analysis
- **Type / role:** LangChain Sentiment Analysis; creates structured sentiment output.
- **Config choices:**
  - **Input text:** `={{ $json.message || 'empty' }}`
  - **Categories:** `Positive, Neutral, Negative`
  - **System prompt:** Instructs: *Only output the JSON* and use provided formatting instructions.
- **Outputs:** Connected to **Add comment** on **three separate outputs** (likely corresponding to categories). In practice, each category route leads to the same storage node.
- **Failure / edge cases:**
  - Comments without `message` become `'empty'`, which can skew sentiment and pollute the sheet.
  - Model may return invalid JSON despite instructions (rare but possible); downstream references like `$json.sentimentAnalysis.category` can break if output schema differs.
  - Non-English comments can reduce accuracy unless Gemini handles multilingual well (it often does, but results vary).

---

### 2.5 Persistence + Rate Safety
**Overview:** Stores each analyzed comment in Google Sheets using an upsert keyed by comment ID, then waits briefly before continuing to mitigate rate limits.

**Nodes involved:**
- Add comment
- Wait

#### Node: Add comment
- **Type / role:** Google Sheets; append or update rows to avoid duplicates.
- **Operation:** `appendOrUpdate`
- **Sheet/document:**
  - Spreadsheet: “Facebook comments” (Document ID: `1xMLAtBgSdxh7Cc--J6fExGb2ul48QTvN-KgGitEGgZw`)
  - Sheet tab: “Foglio1” (`gid=0`)
- **Mapping:**
  - `COMMENT` = `={{ $json.message }}`
  - `SENTIMENT` = `={{ $json.sentimentAnalysis.category }}`
  - `COMMENT ID` = `={{ $json.id }}`
  - (Schema includes `POST ID`, but it is **not mapped** here—so it will remain empty unless sheet formulas/defaults fill it.)
- **Matching columns:** `COMMENT ID` (used for update vs append)
- **Credentials:** Google Sheets OAuth2 “Google Sheets account”
- **Output:** → **Wait**
- **Failure / edge cases:**
  - Spreadsheet permissions (no access) or expired OAuth token.
  - Column headers mismatch: appendOrUpdate requires the sheet to have matching column names.
  - If `sentimentAnalysis.category` is missing (model failure), updates may write blank sentiment.

#### Node: Wait
- **Type / role:** Wait/delay; throttling control between writes/batches.
- **Config:** `amount = 3` (default unit in Wait node is typically seconds unless configured otherwise in UI/version).
- **Connections:** Output → **Loop Over Items** (continues processing next item/batch)
- **Failure / edge cases:**
  - Very large volumes: fixed waits can make executions exceed max runtime on some n8n plans.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Manual entry point | — | Set Fb post ID | Extract & Analyze All Facebook Post Comments with Sentiment AI\n\nThis workflow automates the process of collecting, analyzing, and storing **Facebook post comments** with AI-powered **sentiment analysis* about YOUR Facebook Page.\n\nTypical Use Cases: Social media sentiment monitoring; Brand reputation analysis; Campaign performance evaluation; Community management and moderation insights; Reporting and analytics for marketing teams.\n\nHow it works: fetches comments with pagination, delegates batch processing, runs Gemini sentiment, writes to Google Sheets with upsert, uses batching and waits for rate safety.\n\nSetup: configure Facebook Graph API, Gemini API, Google Sheet columns, verify “Call ‘Facebook’” sub-workflow, adjust batch/wait, test with Manual Trigger. |
| Set Fb post ID | set | Stores the Facebook post ID | When clicking ‘Execute workflow’ | Get Fb Post | STEP 1 - Set Facebook Post\nSet your Facebook post ID to analyze and add your Facebook Graph API credentials to both \"Get Fb Post\" and \"Get Fb comments\" nodes. |
| Get Fb Post | facebookGraphApi | Fetch post metadata | Set Fb post ID | Get Fb comments | STEP 1 - Set Facebook Post\nSet your Facebook post ID to analyze and add your Facebook Graph API credentials to both \"Get Fb Post\" and \"Get Fb comments\" nodes. |
| Get Fb comments | facebookGraphApi | Fetch comments (paged) | Get Fb Post; Get next comments page | Call 'Facebook'; Next page? | STEP 2 - Pagination\nIt's very IMPORTANT to get ALL comments |
| Next page? | if | Detects if paging cursor exists | Get Fb comments | Get next comments page | STEP 2 - Pagination\nIt's very IMPORTANT to get ALL comments |
| Get next comments page | set | Stores `after` cursor for next page | Next page? | Get Fb comments | STEP 2 - Pagination\nIt's very IMPORTANT to get ALL comments |
| Call 'Facebook' | executeWorkflow | Delegates processing to sub-workflow | Get Fb comments | — | Extract & Analyze All Facebook Post Comments with Sentiment AI\n\n(See overview/setup: verify this node references an existing sub-workflow capable of processing comment batches.) |
| When Executed by Another Workflow | executeWorkflowTrigger | Sub-workflow entry point | — | Split Out | STEP 3 - Get single comment\nProcess each comment batch |
| Split Out | splitOut | Splits `data[]` comments into items | When Executed by Another Workflow | Loop Over Items | STEP 3 - Get single comment\nProcess each comment batch |
| Loop Over Items | splitInBatches | Batch/loop controller for per-comment processing | Split Out; Wait | Sentiment Analysis | STEP 3 - Get single comment\nProcess each comment batch |
| Google Gemini Chat Model | lmChatGoogleGemini | Provides Gemini LLM to AI nodes | — | Sentiment Analysis (ai_languageModel) | STEP 4 - Calculate sentiment\nClone https://docs.google.com/spreadsheets/d/1xMLAtBgSdxh7Cc--J6fExGb2ul48QTvN-KgGitEGgZw/edit?usp=sharing and set the POST ID as in the \"Set Fb post ID\" node. Processes each comment through the sentiment analysis model (Google Gemini) |
| Sentiment Analysis | sentimentAnalysis | Classifies sentiment into 3 categories | Loop Over Items; Google Gemini Chat Model | Add comment | STEP 4 - Calculate sentiment\nClone https://docs.google.com/spreadsheets/d/1xMLAtBgSdxh7Cc--J6fExGb2ul48QTvN-KgGitEGgZw/edit?usp=sharing and set the POST ID as in the \"Set Fb post ID\" node. Processes each comment through the sentiment analysis model (Google Gemini) |
| Add comment | googleSheets | Upsert row by COMMENT ID | Sentiment Analysis | Wait | STEP 4 - Calculate sentiment\nClone https://docs.google.com/spreadsheets/d/1xMLAtBgSdxh7Cc--J6fExGb2ul48QTvN-KgGitEGgZw/edit?usp=sharing and set the POST ID as in the \"Set Fb post ID\" node. Processes each comment through the sentiment analysis model (Google Gemini) |
| Wait | wait | Throttle loop to reduce rate limiting | Add comment | Loop Over Items | STEP 3 - Get single comment\nProcess each comment batch |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Analyze All Facebook Post Comments* (or your preferred name).

2) **Add Manual Trigger**
- Node: **Manual Trigger**
- Name: **When clicking ‘Execute workflow’**

3) **Add Set node for the post ID**
- Node: **Set**
- Name: **Set Fb post ID**
- Add field:
  - `post_id` (String) = `YOUR_FACEBOOK_POST_ID`
- Connect: Manual Trigger → Set Fb post ID

4) **Add Facebook Graph API node to fetch the post**
- Node: **Facebook Graph API**
- Name: **Get Fb Post**
- Configure:
  - Graph API Version: **v21.0**
  - Node/Object ID: expression `{{$json.post_id}}`
  - Fields: `id,message,created_time,permalink_url,attachments{title,description,media_type,url,unshimmed_url}`
- Credentials:
  - Add **Facebook Graph API** credentials (Page access as required)
- Connect: Set Fb post ID → Get Fb Post

5) **Add Facebook Graph API node to fetch comments**
- Node: **Facebook Graph API**
- Name: **Get Fb comments**
- Configure:
  - Edge: **comments**
  - Node/Object ID: `{{$('Set Fb post ID').item.json.post_id}}`
  - Fields: `id,from,message,created_time,comment_count`
  - Query parameters:
    - `order` = `reverse_chronological`
    - `after` = `{{$json.after || ""}}`
    - `summary` = `true`
  - Graph API Version: **v21.0**
- Credentials: same Facebook credential
- Connect: Get Fb Post → Get Fb comments

6) **Add IF node for pagination**
- Node: **IF**
- Name: **Next page?**
- Condition: “exists” on:
  - Left value: `{{$('Get Fb comments').item.json.paging.cursors.after}}`
- Connect: Get Fb comments → Next page?

7) **Add Set node to store the cursor**
- Node: **Set**
- Name: **Get next comments page**
- Field:
  - `after` (String) = `{{$json.paging.cursors.after}}`
- Connect: Next page? (true) → Get next comments page
- Connect: Get next comments page → Get Fb comments (this creates the loop)

8) **Decide how you process comment pages**
- **Option A (as in JSON): use a sub-workflow call**
  - Add node: **Execute Workflow**
  - Name: **Call 'Facebook'**
  - Set **Wait for sub-workflow** = true
  - Select the target workflow (must exist) that can accept the comments payload (typically with a `data` array).
  - Connect: Get Fb comments → Call 'Facebook'

- **Option B (single-workflow design): process comments directly in the same workflow**
  - Instead of Execute Workflow, connect Get Fb comments to a **Split Out** node (described below).  
  - (The provided JSON processes comments in the triggered path; Option A is consistent with that architecture.)

9) **Create the processing entry (sub-workflow mode)**
- Add node: **Execute Workflow Trigger**
- Name: **When Executed by Another Workflow**
- Set Input Source: **passthrough**

10) **Split comment array into items**
- Add node: **Split Out**
- Name: **Split Out**
- Field to split out: `data`
- Connect: When Executed by Another Workflow → Split Out

11) **Batch loop**
- Add node: **Split In Batches**
- Name: **Loop Over Items**
- Set batch size as you prefer (commonly 1–20; start with 1–5 to avoid rate limits).
- Connect: Split Out → Loop Over Items

12) **Add Gemini Chat Model**
- Add node: **Google Gemini Chat Model** (LangChain)
- Name: **Google Gemini Chat Model**
- Credentials: configure **Google Gemini (PaLM/Gemini) API** credential.

13) **Add Sentiment Analysis node**
- Add node: **Sentiment Analysis** (LangChain)
- Name: **Sentiment Analysis**
- Configure:
  - Categories: `Positive, Neutral, Negative`
  - Input Text: `{{$json.message || 'empty'}}`
  - System prompt: sentiment classifier, JSON-only output (as per your needs)
- Connect:
  - Loop Over Items → Sentiment Analysis
  - Google Gemini Chat Model (ai_languageModel) → Sentiment Analysis (ai_languageModel)

14) **Add Google Sheets upsert**
- Add node: **Google Sheets**
- Name: **Add comment**
- Operation: **Append or Update**
- Select Spreadsheet + Sheet (or paste IDs):
  - Spreadsheet ID: create/choose your document
  - Sheet tab: the tab containing headers
- Mapping:
  - `COMMENT ID` = `{{$json.id}}`
  - `COMMENT` = `{{$json.message}}`
  - `SENTIMENT` = `{{$json.sentimentAnalysis.category}}`
  - (Optional but recommended) `POST ID` = `{{$json.post_id}}` or pass it along earlier
- Matching columns: `COMMENT ID`
- Credentials: **Google Sheets OAuth2**

15) **Add Wait node for throttling**
- Add node: **Wait**
- Name: **Wait**
- Amount: `3` seconds (adjust as needed)
- Connect: Add comment → Wait

16) **Close the loop**
- Connect: Wait → Loop Over Items (so it continues to next item/batch)

17) **Testing**
- Manual mode:
  - Enter a real `post_id` in **Set Fb post ID**
  - Execute the workflow
  - Ensure your **Call 'Facebook'** sub-workflow is correctly set and receives the `data` array.
- Triggered mode:
  - Call this workflow from another workflow using **Execute Workflow** and pass an object shaped like Facebook comments response (must include `data: [...]`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Clone the Google Sheet template and use it for storing results. | https://docs.google.com/spreadsheets/d/1xMLAtBgSdxh7Cc--J6fExGb2ul48QTvN-KgGitEGgZw/edit?usp=sharing |
| Pagination is critical to retrieve *all* comments; the workflow uses `paging.cursors.after` and loops until it no longer exists. | Applies to the “Get Fb comments” → “Next page?” → “Get next comments page” loop |
| The sheet schema includes POST ID, but the current mapping does not populate it; consider mapping POST ID explicitly if you need per-post filtering. | Google Sheets node (“Add comment”) mapping |
| The Execute Workflow node “Call 'Facebook'” must point to an existing workflow; otherwise manual mode will not process comments. | Execute Workflow node configuration |

If you want, I can also propose a cleaned-up single-workflow architecture (no Execute Workflow call) that processes comments directly after “Get Fb comments”, while keeping pagination and batching intact.