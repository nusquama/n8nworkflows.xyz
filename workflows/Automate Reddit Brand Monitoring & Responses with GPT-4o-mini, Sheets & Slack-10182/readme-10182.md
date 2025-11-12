Automate Reddit Brand Monitoring & Responses with GPT-4o-mini, Sheets & Slack

https://n8nworkflows.xyz/workflows/automate-reddit-brand-monitoring---responses-with-gpt-4o-mini--sheets---slack-10182


# Automate Reddit Brand Monitoring & Responses with GPT-4o-mini, Sheets & Slack

### 1. Workflow Overview

This workflow automates monitoring Reddit for brand mentions, analyzes the posts using AI, engages with relevant discussions by posting helpful comments, logs interactions in Google Sheets, and sends a daily performance summary report to a Slack channel. It is designed for marketing teams aiming to maintain active brand presence on Reddit, capture engagement opportunities, and measure sentiment and effectiveness of outreach.

The workflow’s logic is organized into the following functional blocks:

- **1.1 Scheduled Trigger and Keyword Configuration**: Periodically initiates the workflow and sets the target brand keywords.
- **1.2 Reddit Search and AI Analysis**: Searches Reddit for brand-related posts and applies AI to assess sentiment and generate responses.
- **1.3 Filtering and Engagement**: Filters posts worthy of engagement, processes each post individually, posts AI-generated comments, and logs data.
- **1.4 Reporting**: Aggregates interaction data and sends a formatted daily summary report to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Keyword Configuration

**Overview:**  
This block triggers the workflow once every 24 hours and configures the specific brand keywords to monitor on Reddit.

**Nodes Involved:**  
- Daily Marketing Check  
- Brand Keywords Config

**Node Details:**

- **Daily Marketing Check**  
  - *Type & Role:* Schedule Trigger — initiates workflow daily.  
  - *Configuration:* Runs every 24 hours (hoursInterval=24).  
  - *Key Expressions:* None (fixed interval).  
  - *Connections:* Output → Brand Keywords Config.  
  - *Failures:* Possible schedule misconfiguration or workflow not activated.  
  - *Notes:* Ensures automated daily scans without manual intervention.

- **Brand Keywords Config**  
  - *Type & Role:* Code Node — defines array of brand keywords to search.  
  - *Configuration:* JavaScript array listing brand-related keywords (e.g., "YourBrandName", "your-product-name", "industry-keyword").  
  - *Key Expressions:* Returns array with property `brandKeyword` for each keyword.  
  - *Connections:* Output → Search Brand Mentions.  
  - *Failures:* JavaScript errors if keywords array malformed.  
  - *Notes:* Customizable to include any keywords relevant to brand monitoring.

---

#### 1.2 Reddit Search and AI Analysis

**Overview:**  
Searches Reddit for posts mentioning configured brand keywords, then uses AI to analyze each post’s sentiment, relevance, and to generate contextual responses.

**Nodes Involved:**  
- Search Brand Mentions  
- AI Post Analysis

**Node Details:**

- **Search Brand Mentions**  
  - *Type & Role:* Reddit Node — searches Reddit posts containing brand keywords.  
  - *Configuration:*  
    - Operation: search across all subreddits.  
    - Keyword: dynamically set from `brandKeyword` (from Brand Keywords Config).  
    - Limit: 50 posts per keyword.  
    - Sort: newest first.  
  - *Key Expressions:* `={{ $json.brandKeyword }}` dynamic keyword injection.  
  - *Connections:* Output → AI Post Analysis.  
  - *Failures:* Reddit API rate limits, authentication errors, or empty results.  
  - *Notes:* Captures fresh mentions for timely response.

- **AI Post Analysis**  
  - *Type & Role:* OpenAI Node (LangChain) — performs sentiment analysis, relevance scoring, and generates responses.  
  - *Configuration:*  
    - Model: GPT-4o-mini (or configured OpenAI model).  
    - Temperature: 0.7 for balanced creativity.  
    - Messages: configured to analyze post content and output JSON fields such as `isRelevant`, `engagementScore`, `responseType`, `sentiment`, `suggestedResponse`, `reasoning`.  
  - *Key Expressions:* Utilizes post data from Reddit node; outputs analyzed data with multiple relevant fields.  
  - *Connections:* Output → Filter Engagement-Worthy.  
  - *Failures:* API key errors, timeouts, or invalid prompt responses.  
  - *Notes:* Central AI processing node for decision making.

---

#### 1.3 Filtering and Engagement

**Overview:**  
Filters posts by relevance and quality, processes posts one-by-one to respect rate limits, posts AI-generated comments on worthy posts, and logs all interactions to Google Sheets.

**Nodes Involved:**  
- Filter Engagement-Worthy  
- Loop Through Posts  
- Post Helpful Comment  
- Log to Google Sheets

**Node Details:**

- **Filter Engagement-Worthy**  
  - *Type & Role:* If Node — filters posts that meet engagement criteria.  
  - *Configuration:* Combines three conditions (all must be true):  
    - `isRelevant` is true (AI analysis).  
    - `engagementScore` > 60 (threshold for quality).  
    - `responseType` is not "pass" (indicating actionable post).  
  - *Key Expressions:* Uses evaluated AI output fields.  
  - *Connections:* True → Loop Through Posts.  
  - *Failures:* Misconfiguration could block all posts; expression evaluation errors.  
  - *Notes:* Prevents irrelevant or low-quality engagement.

- **Loop Through Posts**  
  - *Type & Role:* SplitInBatches — processes posts individually to avoid API rate limiting.  
  - *Configuration:* Default batch size = 1 (one post at a time).  
  - *Connections:* Outputs to Post Helpful Comment (main output 0) and Generate Daily Summary (main output 1).  
  - *Failures:* Batch processing errors, throttling.  
  - *Notes:* Ensures safe, sequential posting.

- **Post Helpful Comment**  
  - *Type & Role:* Reddit Node — posts AI-generated comment on Reddit posts.  
  - *Configuration:*  
    - Resource: postComment.  
    - postId: taken from Search Brand Mentions node.  
    - commentText: AI-generated response from AI Post Analysis.  
  - *Key Expressions:*  
    - Post ID: `={{ $('Search Brand Mentions').item.json.data.id }}`  
    - Comment Text: `={{ $('AI Post Analysis').item.json.suggestedResponse }}`  
  - *Connections:* Output → Log to Google Sheets.  
  - *Failures:* Reddit posting errors, rate limits, invalid post IDs.  
  - *Notes:* Engages Reddit community automatically with AI-crafted comments.

- **Log to Google Sheets**  
  - *Type & Role:* Google Sheets Node — appends interaction data to a spreadsheet for tracking.  
  - *Configuration:*  
    - Operation: append row to "Sheet1".  
    - Columns mapped include postId, postUrl, postTitle, reasoning, sentiment, subreddit, timestamp, responseType, commentPosted (success or failure), engagementScore.  
    - Requires Google Sheets OAuth credentials and document ID configured.  
  - *Key Expressions:*  
    - Dynamic fields referencing Reddit and AI nodes.  
    - Timestamp: current ISO date-time.  
  - *Connections:* Output → Loop Through Posts (continuation).  
  - *Failures:* Spreadsheet permission errors, API quota limits.  
  - *Notes:* Builds a permanent record of interactions for analytics.

---

#### 1.4 Reporting

**Overview:**  
Aggregates all daily processed data into a summary report, then sends the report formatted with key metrics and top engagement opportunities to a Slack channel.

**Nodes Involved:**  
- Generate Daily Summary  
- Send Slack Report

**Node Details:**

- **Generate Daily Summary**  
  - *Type & Role:* Code Node — aggregates daily metrics from all processed posts.  
  - *Configuration:*  
    - Counts total posts, posts engaged (comments posted), computes engagement rate, average engagement score.  
    - Generates sentiment breakdown counts.  
    - Lists top 5 posts by engagement score with title, subreddit, score, and URL.  
    - Returns a JSON summary object with these metrics and current date.  
  - *Key Expressions:* Uses `$input.all()` to get all input items.  
  - *Connections:* Output → Send Slack Report.  
  - *Failures:* JavaScript errors or empty input array.  
  - *Notes:* Generates a concise overview for team review.

- **Send Slack Report**  
  - *Type & Role:* Slack Node — sends formatted daily report message to a configured Slack channel.  
  - *Configuration:*  
    - Message text includes date, total posts analyzed, comments posted, engagement rate, average score, sentiment breakdown, and top 5 posts with links.  
    - Channel selected by configured channel ID.  
  - *Key Expressions:* Template literals and JS expressions to iterate over summary data.  
  - *Connections:* Final node (no outputs).  
  - *Failures:* Slack API authentication errors, invalid channel ID.  
  - *Notes:* Keeps marketing team informed with automated daily insights.

---

### 3. Summary Table

| Node Name              | Node Type                       | Functional Role                                     | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                  |
|------------------------|--------------------------------|----------------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Daily Marketing Check   | Schedule Trigger               | Triggers workflow every 24 hours                    | —                           | Brand Keywords Config        | Schedule trigger runs workflow every 24 hours automatically daily                            |
| Brand Keywords Config   | Code                           | Defines brand-related keywords to monitor           | Daily Marketing Check        | Search Brand Mentions        | JavaScript code node defining brand keywords to monitor Reddit                              |
| Search Brand Mentions   | Reddit                         | Searches Reddit for posts mentioning brand keywords | Brand Keywords Config        | AI Post Analysis             | Reddit node searches all subreddits for brand keyword mentions                              |
| AI Post Analysis       | OpenAI (LangChain)             | Analyzes sentiment, relevance; generates responses  | Search Brand Mentions        | Filter Engagement-Worthy     | OpenAI analyzes sentiment, relevance, generates contextual helpful comment responses + Conditional node filters only high-quality relevant posts worth engaging |
| Filter Engagement-Worthy| If                             | Filters posts worth engaging                         | AI Post Analysis             | Loop Through Posts           |                                                                                            |
| Loop Through Posts      | SplitInBatches                 | Processes posts individually to respect rate limits | Filter Engagement-Worthy     | Post Helpful Comment, Generate Daily Summary | Split in batches processes each post individually respecting limits                      |
| Post Helpful Comment    | Reddit                         | Posts AI-generated comment on Reddit                | Loop Through Posts           | Log to Google Sheets         | Reddit node posts AI-generated comment to worthy Reddit discussions                         |
| Log to Google Sheets    | Google Sheets                  | Logs all interaction data for tracking              | Post Helpful Comment         | Loop Through Posts           | Appends all interaction data to spreadsheet for permanent tracking                         |
| Generate Daily Summary  | Code                           | Aggregates daily data into report                    | Loop Through Posts (second output) | Send Slack Report         | JavaScript aggregates metrics, sentiment breakdown, generates comprehensive daily report   |
| Send Slack Report       | Slack                          | Sends the daily summary report to Slack             | Generate Daily Summary       | —                           | Posts formatted daily summary with metrics to team Slack channel                           |
| Sticky Note             | Sticky Note                    | Overview and setup notes                             | —                           | —                           | Reddit Brand Marketing Workflow purpose, features, setup required                          |
| Sticky Note1            | Sticky Note                    | Notes on schedule trigger                            | —                           | —                           | Schedule trigger runs workflow every 24 hours automatically daily                          |
| Sticky Note2            | Sticky Note                    | Notes on brand keywords config code                  | —                           | —                           | JavaScript code node defining brand keywords to monitor Reddit                            |
| Sticky Note3            | Sticky Note                    | Notes on Reddit search node                          | —                           | —                           | Reddit node searches all subreddits for brand keyword mentions                             |
| Sticky Note4            | Sticky Note                    | Notes on AI analysis and filtering                   | —                           | —                           | OpenAI analyzes sentiment, relevance, generates contextual helpful comment responses + Conditional node filters only high-quality relevant posts worth engaging |
| Sticky Note5            | Sticky Note                    | Notes on batch processing                            | —                           | —                           | Split in batches processes each post individually respecting limits                        |
| Sticky Note6            | Sticky Note                    | Notes on posting comments                            | —                           | —                           | Reddit node posts AI-generated comment to worthy Reddit discussions                        |
| Sticky Note7            | Sticky Note                    | Notes on logging to Google Sheets                    | —                           | —                           | Appends all interaction data to spreadsheet for permanent tracking                        |
| Sticky Note8            | Sticky Note                    | Notes on daily summary generation                     | —                           | —                           | JavaScript aggregates metrics, sentiment breakdown, generates comprehensive daily report  |
| Sticky Note9            | Sticky Note                    | Notes on Slack reporting                             | —                           | —                           | Posts formatted daily summary with metrics to team Slack channel                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run every 24 hours (hoursInterval=24)  
   - Name it "Daily Marketing Check"

2. **Create Code Node for Brand Keywords**  
   - Type: Code (JavaScript)  
   - Name: "Brand Keywords Config"  
   - Paste code to define array of keywords, e.g.:  
     ```js
     const brandKeywords = [
       "YourBrandName",
       "your-product-name",
       "industry-keyword"
     ];
     return brandKeywords.map(keyword => ({ json: { brandKeyword: keyword } }));
     ```  
   - Connect output of "Daily Marketing Check" → "Brand Keywords Config"

3. **Create Reddit Search Node**  
   - Type: Reddit  
   - Name: "Search Brand Mentions"  
   - Operation: Search posts  
   - Location: allReddit  
   - Keyword: `={{ $json.brandKeyword }}` (dynamic from previous node)  
   - Limit: 50  
   - Sort: new  
   - Connect output of "Brand Keywords Config" → "Search Brand Mentions"

4. **Create OpenAI Node for AI Analysis**  
   - Type: OpenAI (LangChain)  
   - Name: "AI Post Analysis"  
   - Model: GPT-4o-mini or preferred GPT-4 variant  
   - Temperature: 0.7  
   - Configure prompt/messages to analyze posts for: relevance, sentiment, engagementScore, suggested AI response, responseType, reasoning  
   - Connect output of "Search Brand Mentions" → "AI Post Analysis"

5. **Create If Node to Filter Posts**  
   - Type: If  
   - Name: "Filter Engagement-Worthy"  
   - Conditions (all must be met):  
     - `isRelevant` == true  
     - `engagementScore` > 60  
     - `responseType` != "pass"  
   - Connect output of "AI Post Analysis" → "Filter Engagement-Worthy"

6. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Name: "Loop Through Posts"  
   - Batch size: 1 (default)  
   - Connect True output of "Filter Engagement-Worthy" → "Loop Through Posts"

7. **Create Reddit Node to Post Comments**  
   - Type: Reddit  
   - Name: "Post Helpful Comment"  
   - Operation: postComment  
   - Post ID: `={{ $('Search Brand Mentions').item.json.data.id }}`  
   - Comment Text: `={{ $('AI Post Analysis').item.json.suggestedResponse }}`  
   - Connect output of "Loop Through Posts" → "Post Helpful Comment"

8. **Create Google Sheets Node to Log Interactions**  
   - Type: Google Sheets  
   - Name: "Log to Google Sheets"  
   - Operation: Append  
   - Sheet Name: e.g., "Sheet1"  
   - Document ID: Your Google Sheets document ID  
   - Map columns: postId, postUrl, postTitle, reasoning, sentiment, subreddit, timestamp, responseType, commentPosted, engagementScore  
   - Use expressions to pull data from Reddit and AI nodes, e.g., postUrl: `=https://reddit.com{{ $('Search Brand Mentions').item.json.data.permalink }}`  
   - Connect output of "Post Helpful Comment" → "Log to Google Sheets"  
   - Connect output of "Log to Google Sheets" → "Loop Through Posts" (to continue batch processing)

9. **Create Code Node to Generate Daily Summary**  
   - Type: Code (JavaScript)  
   - Name: "Generate Daily Summary"  
   - Use code to aggregate data from all batch items, computing totals, engagement rates, sentiment breakdown, top posts, and formatting report date  
   - Connect second output of "Loop Through Posts" → "Generate Daily Summary"

10. **Create Slack Node to Send Report**  
    - Type: Slack  
    - Name: "Send Slack Report"  
    - Configure Slack credentials (OAuth2)  
    - Channel: Set your Slack channel ID  
    - Message: Template including date, totals, sentiment breakdown, top posts with links, using expressions from summary node  
    - Connect output of "Generate Daily Summary" → "Send Slack Report"

11. **Optional: Add Sticky Notes**  
    - Add notes throughout workflow to document each block’s purpose and setup requirements.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                             |
|------------------------------------------------------------------------------------------------------|--------------------------------------------|
| Requires Reddit OAuth credentials for search and posting operations                                  | Setup Reddit app and OAuth credentials     |
| Requires OpenAI API key with access to GPT-4o-mini or compatible OpenAI models                        | OpenAI API portal                          |
| Google Sheets document must be shared with n8n Google OAuth client                                  | Google Sheets API setup                     |
| Slack OAuth2 credentials and channel ID required for report posting                                 | Slack API and App configuration             |
| Customize brand keywords in "Brand Keywords Config" code node                                        | Adjust keywords for your brand and industry|
| Monitors Reddit continuously with daily automated scans and AI-driven engagement                    | Workflow purpose summary                     |
| Detailed Slack daily report includes engagement metrics and top posts for team transparency          | Communication best practices                 |

---

**Disclaimer:** This document is derived exclusively from an n8n automation workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.