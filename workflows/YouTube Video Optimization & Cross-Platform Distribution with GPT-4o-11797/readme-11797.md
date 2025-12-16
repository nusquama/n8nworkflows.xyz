YouTube Video Optimization & Cross-Platform Distribution with GPT-4o

https://n8nworkflows.xyz/workflows/youtube-video-optimization---cross-platform-distribution-with-gpt-4o-11797


# YouTube Video Optimization & Cross-Platform Distribution with GPT-4o

### 1. Workflow Overview

This workflow automates the post-publication process for YouTube videos with a focus on SEO optimization, competitor analysis, cross-platform promotion, and engagement monitoring. It targets content creators and digital marketers who want to enhance YouTube video visibility and streamline multi-channel distribution while maintaining analytics and feedback loops.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Configuration**: Receives new video publish events and loads environment variables and shared settings.
- **1.2 Video Metadata Intake**: Fetches detailed video metadata from YouTube and prepares context for SEO and comparison.
- **1.3 Competitor & Trend Data Retrieval**: Gathers competitor and trending topic data via external APIs for enriched context.
- **1.4 AI-Powered SEO Asset Generation**: Uses GPT-4o to generate SEO-optimized title variations, descriptions, and tags.
- **1.5 SEO Scoring & A/B Title Selection**: Scores the SEO assets and selects an A/B test variant for the video title.
- **1.6 YouTube Metadata Update**: Updates the YouTube video with selected SEO-optimized metadata.
- **1.7 Clip & Thumbnail Processing**: If the video length exceeds a threshold, runs thumbnail analysis and clip extraction.
- **1.8 Cross-Platform Content Generation & Publishing**: Creates and formats promotional posts per platform and publishes them.
- **1.9 Comments Monitoring & Response**: Fetches video comments, uses AI to generate responses, detects negative comments, and logs details.
- **1.10 Analytics Aggregation & Reporting**: Runs scheduled weekly analytics aggregation, calculates engagement and viral potential, logs insights, and sends email reports.
- **1.11 Notifications & Logging**: Sends Slack alerts for new videos and viral potential, and logs video, comments, and analytics data to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Configuration

- **Overview:** Listens for new YouTube video publish events via webhook; loads environment variables and IDs for workflow-wide use.
- **Nodes Involved:**  
  - `Video Published Webhook`  
  - `Workflow Configuration`  
  - Sticky Note: "Triggers & Configuration"

- **Node Details:**

  - **Video Published Webhook**
    - Type: Webhook (HTTP receive)
    - Role: Entry trigger for new published video event
    - Configuration: Receives HTTP POST at path `dffce599-768d-4471-a26e-3da64e7aded0`
    - Inputs: External YouTube publish event
    - Outputs: Passes videoId, channelId, etc.
    - Failures: Missing or malformed webhook payload, network issues

  - **Workflow Configuration**
    - Type: Set node
    - Role: Load and assign environment variables (API URLs, keys, IDs) and webhook data into workflow context for reuse
    - Key expressions: Assigns values from `$json` and `$env` variables
    - Inputs: From webhook
    - Outputs: Provides configuration context downstream
    - Failures: Missing environment variables cause incomplete config

---

#### 1.2 Video Metadata Intake

- **Overview:** Retrieves detailed video metadata (title, description, tags, statistics, duration) from YouTube API and prepares SEO prompt data.
- **Nodes Involved:**  
  - `Get Video Details`  
  - `Prepare SEO Prompts`  
  - Sticky Note: "Video Intake"

- **Node Details:**

  - **Get Video Details**
    - Type: YouTube node
    - Role: Fetch video snippet, statistics, and content details using videoId
    - Configuration: `operation=get`, `resource=video`, parts requested: snippet, statistics, contentDetails
    - Inputs: videoId from `Workflow Configuration`
    - Outputs: Full video metadata JSON
    - Failures: API quota exceeded, invalid videoId

  - **Prepare SEO Prompts**
    - Type: Set node
    - Role: Extracts and simplifies video metadata into variables: title, description, tags, view count, duration
    - Inputs: Video metadata JSON
    - Outputs: Context for competitor and trending data fetch
    - Failures: Unexpected metadata structure

---

#### 1.3 Competitor & Trend Data Retrieval

- **Overview:** Fetches competitor video data and trending topic keywords from external APIs using authenticated HTTP requests.
- **Nodes Involved:**  
  - `Fetch Competitor Data`  
  - `Fetch Trending Topics`  
  - `Competitor Comparison` (merges data)  
  - Sticky Note: "Competitor & Trend Signals"

- **Node Details:**

  - **Fetch Competitor Data**
    - Type: HTTP Request
    - Role: Pull competitor data relevant to current video title
    - Configuration: GET with `topic` query, Bearer token authentication header
    - Inputs: videoTitle from `Prepare SEO Prompts`
    - Outputs: Competitor data JSON
    - Failures: API timeout, auth failure

  - **Fetch Trending Topics**
    - Type: HTTP Request
    - Role: Retrieve trending keywords based on video category
    - Configuration: GET with `category` query, Bearer token authentication
    - Inputs: snippet.categoryId from video metadata
    - Outputs: Trending topics JSON
    - Failures: API timeout, auth failure

  - **Competitor Comparison**
    - Type: Code
    - Role: Combines video metadata with competitor and trending data into unified context object
    - Inputs: Outputs from competitor and trending APIs and video metadata
    - Outputs: Combined JSON for AI SEO generation
    - Failures: Missing competitor or trending data (handled by defaults)

---

#### 1.4 AI-Powered SEO Asset Generation

- **Overview:** Uses GPT-4o to generate SEO-optimized title variations, video description, and tags based on combined context.
- **Nodes Involved:**  
  - `Generate Title Variations`  
  - `Generate SEO Description`  
  - `Generate Tags`  
  - `Merge SEO Data` (combine outputs)  
  - Sticky Note: "Generate SEO Assets (AI)"

- **Node Details:**

  - **Generate Title Variations**
    - Type: OpenAI (LangChain)
    - Role: Generate 3 SEO-optimized YouTube title variants from original title, competitor titles, and trending keywords
    - Configuration: Prompt template with JSON output structure `{ "titles": [...] }`
    - Inputs: Combined competitor/trending/video data
    - Outputs: JSON array of titles
    - Failures: OpenAI API errors, invalid JSON parse

  - **Generate SEO Description**
    - Type: OpenAI (LangChain)
    - Role: Generate keyword-rich, engaging video description (150-200 words)
    - Configuration: Prompt requesting JSON `{ "description": "..." }`
    - Inputs: Original description + trending keywords
    - Outputs: SEO description JSON
    - Failures: OpenAI API errors, invalid JSON parse

  - **Generate Tags**
    - Type: OpenAI (LangChain)
    - Role: Generate 10-15 SEO tags
    - Configuration: Prompt requesting JSON `{ "tags": [...] }`
    - Inputs: Video title + trending keywords
    - Outputs: Tag array JSON
    - Failures: OpenAI API errors, invalid JSON parse

  - **Merge SEO Data**
    - Type: Merge node (combine)
    - Role: Combines outputs from title, description, and tags generation into one payload
    - Inputs: Outputs of the three AI nodes
    - Outputs: Consolidated SEO asset JSON
    - Failures: Mismatch in inputs, partial data

---

#### 1.5 SEO Scoring & A/B Title Selection

- **Overview:** Scores SEO assets based on criteria and selects an A/B test variant for the video title.
- **Nodes Involved:**  
  - `Calculate SEO Score`  
  - `A/B Title Routing Logic`  
  - `A/B Test Variant Selection`  
  - `Set Variant B Data`  
  - Sticky Note: "SEO Scoring & A/B Selection"

- **Node Details:**

  - **Calculate SEO Score**
    - Type: Code node
    - Role: Parses AI responses, evaluates SEO quality (title count, description length, tag count), assigns grade A-D
    - Inputs: Merged SEO data + original metadata
    - Outputs: SEO score, grade, parsed SEO assets
    - Failures: Invalid JSON parsing, missing fields

  - **A/B Title Routing Logic**
    - Type: Code node
    - Role: Deterministically selects A or B variant based on videoId hash
    - Logic: If variant A, selects first title variation; else defaults to original title
    - Outputs: Variant label and selected title
    - Failures: Missing title variations

  - **A/B Test Variant Selection**
    - Type: If node
    - Role: Routes flow based on variant (A or B)
    - Outputs: Variant A continues; Variant B triggers `Set Variant B Data`

  - **Set Variant B Data**
    - Type: Set node
    - Role: Sets variant to B and selects second title variation if available, else original title
    - Outputs: Updated title for variant B

---

#### 1.6 YouTube Metadata Update

- **Overview:** Updates the YouTube video metadata (title, description, tags) with SEO-optimized data.
- **Nodes Involved:**  
  - `Update Video Metadata`  
  - Sticky Note: "Update YouTube Metadata"

- **Node Details:**

  - **Update Video Metadata**
    - Type: YouTube node
    - Role: Performs video update operation with selected title, SEO description, and tags
    - Inputs: Selected title, SEO description, SEO tags, videoId
    - Outputs: Confirmation from YouTube API
    - Failures: API quota limits, invalid metadata, network errors

---

#### 1.7 Clip & Thumbnail Processing

- **Overview:** For videos longer than 10 minutes, runs thumbnail analysis and clip extraction to generate promotional assets.
- **Nodes Involved:**  
  - `Check Video Length` (If node)  
  - `Thumbnail Analysis API`  
  - `Clip Extraction Service`  
  - `Prepare Cross-Platform Context`  
  - Sticky Note: "Clip & Thumbnail Processing"

- **Node Details:**

  - **Check Video Length**
    - Type: If node
    - Role: Checks if video duration is greater than 10 minutes (ISO 8601 duration format)
    - Inputs: Duration from updated metadata
    - Outputs: Triggers thumbnail and clip processing if true
    - Failures: Incorrect duration format

  - **Thumbnail Analysis API**
    - Type: HTTP Request
    - Role: Sends videoId and thumbnail URL to external service for thumbnail scoring
    - Inputs: videoId, max resolution thumbnail URL
    - Outputs: Thumbnail score and related data
    - Failures: API timeout, auth failure

  - **Clip Extraction Service**
    - Type: HTTP Request
    - Role: Requests video clip extraction based on videoId and duration
    - Inputs: videoId, duration string
    - Outputs: Clip URL for promo use
    - Failures: API timeout, auth failure

  - **Prepare Cross-Platform Context**
    - Type: Set node
    - Role: Prepares data for social post generation including video URL, clip URL, and thumbnail score
    - Outputs: Context for cross-platform content AI generation

---

#### 1.8 Cross-Platform Content Generation & Publishing

- **Overview:** Generates platform-specific promotional posts using AI, formats posts, checks platform availability, and publishes to multiple channels.
- **Nodes Involved:**  
  - `Generate Cross-Platform Content1` (OpenAI GPT-4o)  
  - `Format Platform Posts` (Set node)  
  - `Check Platform Availability` (If node)  
  - Social Posting Nodes:  
    - `Post to LinkedIn`  
    - `Post Twitter Thread`  
    - `Upload Instagram Reel`  
    - `Post to Facebook Group`  
  - Sticky Note: "Cross-Platform Promotion"

- **Node Details:**

  - **Generate Cross-Platform Content1**
    - Type: OpenAI (LangChain)
    - Role: Generates JSON containing post texts optimized per platform (LinkedIn, Twitter, Instagram, TikTok, Facebook)
    - Inputs: Video title, URL, clip URL, thumbnail score
    - Outputs: JSON with platform posts
    - Failures: OpenAI errors, JSON parse errors

  - **Format Platform Posts**
    - Type: Set node
    - Role: Extracts and formats the posts from AI JSON to individual fields per platform
    - Outputs: Separate fields for each platform post content

  - **Check Platform Availability**
    - Type: If node
    - Role: Checks which social media posts are non-empty and only publishes to those platforms
    - Outputs: Branches to post nodes only if content exists

  - **Post to LinkedIn**
    - Type: LinkedIn node
    - Role: Posts the LinkedIn post text as the authenticated user
    - Inputs: linkedinPost text, LinkedIn person ID credential
    - Failures: API auth errors, rate limits

  - **Post Twitter Thread**
    - Type: Twitter node
    - Role: Posts Twitter thread content
    - Inputs: twitterPost text, configured Twitter credentials
    - Failures: API limits, auth errors

  - **Upload Instagram Reel**
    - Type: Facebook Graph API node
    - Role: Uploads Instagram Reel media post
    - Inputs: Instagram account ID, post content
    - Failures: API permission errors

  - **Post to Facebook Group**
    - Type: Facebook Graph API node
    - Role: Posts to specified Facebook group
    - Inputs: Facebook group ID, post content
    - Failures: API auth, permission errors

---

#### 1.9 Comments Monitoring & Response

- **Overview:** Retrieves recent video comments, generates AI responses, detects negative comments, sends alerts, and logs data for monitoring.
- **Nodes Involved:**  
  - `Fetch Video Comments`  
  - `Generate Comment Responses1` (OpenAI)  
  - `Detect Negative Comments` (If node)  
  - `Slack - Viral Potential Alert`  
  - `Log Comments` (Google Sheets)  
  - Sticky Note: "Comments Monitoring & Logging"

- **Node Details:**

  - **Fetch Video Comments**
    - Type: YouTube node
    - Role: Fetches video comments
    - Inputs: Video ID
    - Outputs: List of comments

  - **Generate Comment Responses1**
    - Type: OpenAI (LangChain)
    - Role: Analyzes comments, generates response suggestions, flags toxic/negative comments
    - Outputs: JSON with responses array and negative flags

  - **Detect Negative Comments**
    - Type: If node
    - Role: Checks if any comment is flagged negative
    - Outputs:  
      - If negative comments detected: triggers Slack alert and logs comments  
      - Else: only logs comments

  - **Slack - Viral Potential Alert**
    - Type: Slack node
    - Role: Sends alert message to Slack channel about viral potential and negative comments

  - **Log Comments**
    - Type: Google Sheets node
    - Role: Appends comment responses and metadata to comments tracking sheet using service account authentication

---

#### 1.10 Analytics Aggregation & Reporting

- **Overview:** Runs a weekly scheduled trigger to fetch analytics data, compute engagement and viral potential scores, log insights, and email a detailed report.
- **Nodes Involved:**  
  - `Weekly Analytics Schedule`  
  - `Fetch Video Analytics`  
  - `Aggregate Analytics Data`  
  - `Calculate Engagement Score`  
  - `Detect High Engagement` (If)  
  - `Check Viral Potential` (If)  
  - `Generate Weekly Report1` (OpenAI)  
  - `Log Strategic Insights` (Google Sheets)  
  - `Send Email Newsletter` (SendGrid)  
  - Sticky Note: "Weekly Analytics & Reporting"

- **Node Details:**

  - **Weekly Analytics Schedule**
    - Type: Schedule Trigger
    - Role: Fires every Monday at 9:00 AM weekly

  - **Fetch Video Analytics**
    - Type: YouTube node
    - Role: Fetches latest video statistics and content details

  - **Aggregate Analytics Data**
    - Type: Code node
    - Role: Parses and aggregates analytics data; provides defaults if missing

  - **Calculate Engagement Score**
    - Type: Code node
    - Role: Computes engagement rate and assigns grade (High/Medium/Low)

  - **Detect High Engagement**
    - Type: If node
    - Role: Branches flow if engagement grade is High

  - **Check Viral Potential**
    - Type: If node
    - Role: Checks if views exceed 10,000 to detect viral potential

  - **Generate Weekly Report1**
    - Type: OpenAI (LangChain)
    - Role: Creates detailed HTML report and summary based on analytics data

  - **Log Strategic Insights**
    - Type: Google Sheets
    - Role: Logs weekly analytics metrics and viral potential to sheet

  - **Send Email Newsletter**
    - Type: SendGrid node
    - Role: Sends the weekly YouTube analytics report email

---

#### 1.11 Notifications & Logging

- **Overview:** Sends Slack alerts on video publish and viral potential; logs video metadata and analytics to Google Sheets.
- **Nodes Involved:**  
  - `Slack - Video Published Alert`  
  - `Slack - Viral Potential Alert`  
  - `Log to Video Database`  
  - `Log Strategic Insights`  
  - Sticky Notes: "Alerts & Notifications", "Storage (Video Database)"

- **Node Details:**

  - **Slack - Video Published Alert**
    - Type: Slack node
    - Role: Notifies Slack channel immediately after metadata update
    - Message includes video title, SEO score, A/B variant, and URL

  - **Log to Video Database**
    - Type: Google Sheets
    - Role: Appends video record with SEO score, variant, publish date, and selected title

  - **Log Strategic Insights**
    - Type: Google Sheets
    - Role: Logs analytics insights (likes, views, comments, engagement score, viral potential)

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                           | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                     |
|-----------------------------|---------------------------|-----------------------------------------|--------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Video Published Webhook      | Webhook                   | Trigger on new YouTube video publish    | -                              | Workflow Configuration                 | Triggers & Configuration                                                                        |
| Workflow Configuration       | Set                       | Load env vars and initial variables     | Video Published Webhook         | Get Video Details                     | Triggers & Configuration                                                                        |
| Get Video Details            | YouTube                   | Fetch video metadata                     | Workflow Configuration          | Prepare SEO Prompts                   | Video Intake                                                                                   |
| Prepare SEO Prompts          | Set                       | Extract video data for SEO context       | Get Video Details               | Fetch Competitor Data, Fetch Trending Topics | Video Intake                                                                                   |
| Fetch Competitor Data        | HTTP Request              | Retrieve competitor signals              | Prepare SEO Prompts             | Competitor Comparison                 | Competitor & Trend Signals                                                                     |
| Fetch Trending Topics        | HTTP Request              | Retrieve trending keywords               | Prepare SEO Prompts             | Competitor Comparison                 | Competitor & Trend Signals                                                                     |
| Competitor Comparison        | Code                      | Merge competitor and trending data       | Fetch Competitor Data, Fetch Trending Topics | Generate Title Variations, Generate SEO Description, Generate Tags | Competitor & Trend Signals                                                                     |
| Generate Title Variations    | OpenAI LangChain          | Create SEO title variants                 | Competitor Comparison           | Merge SEO Data                       | Generate SEO Assets (AI)                                                                       |
| Generate SEO Description     | OpenAI LangChain          | Create SEO video description              | Competitor Comparison           | Merge SEO Data                       | Generate SEO Assets (AI)                                                                       |
| Generate Tags               | OpenAI LangChain          | Create SEO tags                           | Competitor Comparison           | Merge SEO Data                       | Generate SEO Assets (AI)                                                                       |
| Merge SEO Data              | Merge                     | Combine SEO assets                        | Generate Title Variations, Generate SEO Description, Generate Tags | Calculate SEO Score                 | Generate SEO Assets (AI)                                                                       |
| Calculate SEO Score          | Code                      | Score SEO assets and parse AI responses | Merge SEO Data                 | A/B Title Routing Logic              | SEO Scoring & A/B Selection                                                                   |
| A/B Title Routing Logic      | Code                      | Select A/B test variant and title        | Calculate SEO Score             | A/B Test Variant Selection           | SEO Scoring & A/B Selection                                                                   |
| A/B Test Variant Selection   | If                        | Route based on variant (A or B)           | A/B Title Routing Logic         | Update Video Metadata, Set Variant B Data | SEO Scoring & A/B Selection                                                                   |
| Set Variant B Data           | Set                       | Set variant B title                      | A/B Test Variant Selection       | Update Video Metadata                | SEO Scoring & A/B Selection                                                                   |
| Update Video Metadata        | YouTube                   | Update YouTube video metadata            | A/B Test Variant Selection, Set Variant B Data | Check Video Length, Log to Video Database, Slack - Video Published Alert, Fetch Video Comments | Update YouTube Metadata                                                                       |
| Check Video Length           | If                        | Check if video duration > 10 minutes     | Update Video Metadata           | Thumbnail Analysis API, Clip Extraction Service | Clip & Thumbnail Processing                                                                  |
| Thumbnail Analysis API       | HTTP Request              | Analyze thumbnail quality                 | Check Video Length              | Prepare Cross-Platform Context       | Clip & Thumbnail Processing                                                                  |
| Clip Extraction Service      | HTTP Request              | Extract video clips/shorts                | Check Video Length              | Prepare Cross-Platform Context       | Clip & Thumbnail Processing                                                                  |
| Prepare Cross-Platform Context | Set                     | Prepare data for social post generation  | Thumbnail Analysis API, Clip Extraction Service | Generate Cross-Platform Content1     | Clip & Thumbnail Processing                                                                  |
| Generate Cross-Platform Content1 | OpenAI LangChain      | Generate social media posts per platform | Prepare Cross-Platform Context  | Format Platform Posts               | Cross-Platform Promotion                                                                     |
| Format Platform Posts        | Set                       | Format AI JSON posts into platform fields | Generate Cross-Platform Content1 | Check Platform Availability          | Cross-Platform Promotion                                                                     |
| Check Platform Availability  | If                        | Check if posts exist for each platform   | Format Platform Posts           | Post to LinkedIn, Post Twitter Thread, Upload Instagram Reel, Post to Facebook Group, Send Email Newsletter | Cross-Platform Promotion                                                                     |
| Post to LinkedIn             | LinkedIn                  | Publish post to LinkedIn                  | Check Platform Availability     | -                                   | Cross-Platform Promotion                                                                     |
| Post Twitter Thread          | Twitter                   | Publish Twitter thread                    | Check Platform Availability     | -                                   | Cross-Platform Promotion                                                                     |
| Upload Instagram Reel        | Facebook Graph API        | Post Instagram reel                       | Check Platform Availability     | -                                   | Cross-Platform Promotion                                                                     |
| Post to Facebook Group       | Facebook Graph API        | Post to Facebook group                    | Check Platform Availability     | -                                   | Cross-Platform Promotion                                                                     |
| Send Email Newsletter        | SendGrid                  | Send email with weekly report             | Check Platform Availability     | -                                   | Cross-Platform Promotion                                                                     |
| Fetch Video Comments         | YouTube                   | Retrieve video comments                    | Update Video Metadata           | Generate Comment Responses1          | Comments Monitoring & Logging                                                               |
| Generate Comment Responses1  | OpenAI LangChain          | Analyze comments, generate responses      | Fetch Video Comments            | Detect Negative Comments             | Comments Monitoring & Logging                                                               |
| Detect Negative Comments     | If                        | Check presence of negative comments       | Generate Comment Responses1     | Slack - Viral Potential Alert, Log Comments | Comments Monitoring & Logging                                                               |
| Slack - Viral Potential Alert | Slack                    | Notify Slack channel of viral/negatives   | Detect Negative Comments        | -                                   | Alerts & Notifications                                                                      |
| Log Comments                | Google Sheets              | Log comment responses and metadata        | Detect Negative Comments        | -                                   | Comments Monitoring & Logging                                                               |
| Log to Video Database        | Google Sheets             | Log video metadata and SEO score           | Update Video Metadata           | -                                   | Storage (Video Database)                                                                    |
| Weekly Analytics Schedule    | Schedule Trigger          | Weekly trigger for analytics processing   | -                              | Fetch Video Analytics               | Weekly Analytics & Reporting                                                               |
| Fetch Video Analytics        | YouTube                   | Retrieve video statistics                  | Weekly Analytics Schedule       | Aggregate Analytics Data            | Weekly Analytics & Reporting                                                               |
| Aggregate Analytics Data     | Code                      | Aggregate and clean analytics data         | Fetch Video Analytics           | Calculate Engagement Score          | Weekly Analytics & Reporting                                                               |
| Calculate Engagement Score   | Code                      | Compute engagement rate and grade          | Aggregate Analytics Data        | Detect High Engagement              | Weekly Analytics & Reporting                                                               |
| Detect High Engagement       | If                        | Branch if engagement is high                | Calculate Engagement Score      | Check Viral Potential               | Weekly Analytics & Reporting                                                               |
| Check Viral Potential        | If                        | Detect viral video by view count            | Detect High Engagement          | Generate Weekly Report1             | Weekly Analytics & Reporting                                                               |
| Generate Weekly Report1      | OpenAI LangChain          | Generate weekly analytics report            | Check Viral Potential           | Log Strategic Insights              | Weekly Analytics & Reporting                                                               |
| Log Strategic Insights       | Google Sheets             | Append analytics insights to sheet          | Generate Weekly Report1         | Send Email Newsletter              | Weekly Analytics & Reporting                                                               |
| Slack - Video Published Alert | Slack                    | Notify Slack on video publish                | Update Video Metadata           | -                                   | Alerts & Notifications                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Path: `dffce599-768d-4471-a26e-3da64e7aded0`  
   - Purpose: Receive video published event with videoId and channelId

2. **Create Configuration Set Node**  
   - Type: Set  
   - Purpose: Assign variables from webhook JSON and environment variables for all API keys, URLs, IDs (YouTube, competitor APIs, social platforms, Google Sheets, Slack, SendGrid)  
   - Include all environment variables as per the original

3. **Create YouTube Node to Fetch Video Details**  
   - Operation: `get`  
   - Resource: `video`  
   - Parts: `snippet, statistics, contentDetails`  
   - Video ID: from webhook (`videoId`)

4. **Create Set Node to Prepare SEO Prompts**  
   - Extract title, description, tags, view count, duration from previous node output

5. **Create HTTP Request Nodes to Fetch Competitor Data and Trending Topics**  
   - Auth: HTTP Header Bearer Token  
   - URLs and keys from environment config  
   - Query Parameters: `topic` (video title), `category` (video categoryId)

6. **Create Code Node to Merge Competitor, Trending, and Video Data**  
   - Combine inputs into unified JSON object with fields for AI SEO generation

7. **Create OpenAI Nodes for SEO Asset Generation** (3 Nodes)  
   - Model: GPT-4o  
   - Use prompts to generate:  
     a) Title variations (3 titles)  
     b) SEO description (150-200 words)  
     c) SEO tags (10-15 tags)  
   - Configure JSON-only outputs per prompt instructions

8. **Create Merge Node**  
   - Combine AI outputs by position into single payload

9. **Create Code Node for SEO Scoring**  
   - Parse AI JSON outputs and calculate SEO score based on number of titles, description length, tag count  
   - Assign grade (A-D)

10. **Create Code Node for A/B Title Routing Logic**  
    - Hash videoId to select variant A or B  
    - For A variant, select first title variation; for B, keep original title

11. **Create If Node for A/B Variant Selection**  
    - If variant A: continue  
    - If variant B: route to Set Node

12. **Create Set Node for Variant B Data**  
    - Set variant to B and select second title variation if available, else original title

13. **Create YouTube Node to Update Video Metadata**  
    - Operation: `update`  
    - Update fields: title (selected), description, tags  
    - Video ID from context

14. **Create If Node to Check Video Length > 10m**  
    - Condition: duration string greater than `PT10M`

15. **Create HTTP Request Nodes for Thumbnail Analysis and Clip Extraction**  
    - POST requests with videoId and thumbnail or duration  
    - Auth with respective API keys

16. **Create Set Node to Prepare Cross-Platform Context**  
    - Include video URL, clip URL, thumbnail score

17. **Create OpenAI Node to Generate Cross-Platform Content**  
    - Model: GPT-4o  
    - Prompt to generate JSON with posts for LinkedIn, Twitter, Instagram, TikTok, Facebook

18. **Create Set Node to Format Platform Posts**  
    - Parse JSON string and assign individual post texts to respective fields

19. **Create If Node to Check Platform Availability**  
    - Confirm non-empty post text per platform for publishing

20. **Create Social Posting Nodes**  
    - LinkedIn node: post text and person ID  
    - Twitter node: post text  
    - Facebook Graph API nodes for Instagram Reel and Facebook Group posts  
    - Configure credentials accordingly

21. **Create YouTube Node to Fetch Video Comments**  
    - Resource: videoComment  
    - Video ID from context

22. **Create OpenAI Node to Generate Comment Responses**  
    - Analyze comments, generate responses, flag negatives in JSON output

23. **Create If Node to Detect Negative Comments**  
    - Condition: check for any flagged negative comment in response array

24. **Create Slack Node for Viral Potential Alert**  
    - Send message to Slack channel if negative comments detected or viral potential found

25. **Create Google Sheets Node to Log Comments**  
    - Append rows with comment IDs, responses, negative flags  
    - Authenticate with service account

26. **Create Google Sheets Node to Log Video Data**  
    - Append videoId, SEO score, variant, publish date, selected title

27. **Create Schedule Trigger Node for Weekly Analytics**  
    - Set to every Monday 9:00 AM weekly

28. **Create YouTube Node to Fetch Video Analytics**  
    - Parts: statistics, contentDetails  
    - Video ID from stored context or environment variable

29. **Create Code Node to Aggregate Analytics Data**  
    - Parse and normalize analytics JSON

30. **Create Code Node to Calculate Engagement Score and Grade**  
    - Compute engagement rate from views, likes, comments

31. **Create If Nodes to Detect High Engagement and Viral Potential**  
    - Engagement grade = High  
    - Views > 10,000

32. **Create OpenAI Node to Generate Weekly Report**  
    - Generate HTML and summary report from analytics data

33. **Create Google Sheets Node to Log Strategic Insights**  
    - Append engagement metrics, viral potential, timestamp

34. **Create SendGrid Node to Send Weekly Email Newsletter**  
    - From and to emails from configuration

35. **Create Slack Node to Alert on New Video Published**  
    - Sends immediate notification after metadata update with SEO score and variant info

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                              | Context or Link                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow automates the full post-publish process for YouTube videos including SEO, cross-platform promotion, and engagement monitoring. Setup requires connecting YouTube, OpenAI (GPT-4o), Slack, Google Sheets, SendGrid, and social media APIs.                       | Workflow description (Sticky Note)                                                                                  |
| The workflow uses environment variables extensively for API keys, URLs, and platform IDs; all must be configured before use.                                                                                                                                              | Workflow Configuration notes                                                                                        |
| AI prompt outputs are strictly expected in JSON format with exact keys; any deviation can cause parsing errors—monitor OpenAI API usage and response formats.                                                                                                             | AI nodes prompt instructions                                                                                        |
| Slack alerts provide immediate notification of new videos and viral potential, supporting timely monitoring by the content team.                                                                                                                                          | Slack alert nodes                                                                                                   |
| Google Sheets integration uses service account authentication for logging video metadata, comments, and analytics, allowing centralized tracking and reporting.                                                                                                         | Google Sheets nodes                                                                                                 |
| Weekly analytics report is generated as HTML email via SendGrid with strategic recommendations, enabling data-driven content strategy refinement.                                                                                                                        | Weekly report generation                                                                                            |
| Video length check uses ISO 8601 duration format; ensure input format correctness to avoid branch failures.                                                                                                                                                                 | Duration check node                                                                                                |
| For social media publishing, enable only the platforms you use and configure credentials accordingly; unsupported platforms will be skipped by availability checks.                                                                                                      | Cross-platform promotion nodes                                                                                      |

---

**Disclaimer:**  
The text above is an analysis and documentation of an automated workflow created with n8n, adhering strictly to content policies. All data processed is legal and public.