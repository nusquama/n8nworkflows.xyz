AI-Powered Viral Trend Analysis for TikTok and Instagram with GPT-4

https://n8nworkflows.xyz/workflows/ai-powered-viral-trend-analysis-for-tiktok-and-instagram-with-gpt-4-11411


# AI-Powered Viral Trend Analysis for TikTok and Instagram with GPT-4

---

# AI-Powered Viral Trend Analysis for TikTok and Instagram with GPT-4

---

### 1. Workflow Overview

This workflow automates the daily monitoring and analysis of fast-growing TikTok and Instagram videos within a specified niche. It scrapes content via Apify APIs, normalizes data from both platforms, filters high-engagement videos, and applies GPT-4 AI models to analyze visual hooks and trend mechanics. The results are stored and summarized in Google Sheets and emailed as a daily digest. Additionally, “super viral” content triggers instant Slack alerts for timely action.

**Target Use Cases:**  
- Social media managers seeking daily viral content opportunities  
- Trend analysts wanting AI-driven content insights  
- Brands and creators aiming to adapt trending content rapidly

**Logical Blocks:**  
- 1.1 Schedule Trigger & Configuration Setup  
- 1.2 Data Scraping & Normalization  
- 1.3 Filtering for High-Growth Content  
- 1.4 AI-Based Visual and Trend Analysis  
- 1.5 Super Viral Content Identification  
- 1.6 Data Storage & Reporting (Google Sheets & Email)  
- 1.7 Instant Slack Alerts for Super Viral Content

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Configuration Setup

**Overview:**  
Triggers the workflow daily at 8:00 AM. Sets core configuration parameters such as API tokens, search filters, and thresholds.

**Nodes Involved:**  
- Daily Trigger (08:00)  
- Workflow Configuration  
- Search Config

**Node Details:**  

- **Daily Trigger (08:00)**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every day at 8:00 AM  
  - Configuration: Fixed hour trigger set to 8:00  
  - Inputs: None  
  - Outputs: Connects to Workflow Configuration  
  - Edge Cases: Timezone mismatches may affect trigger timing  
  - Version: 1.3

- **Workflow Configuration**  
  - Type: Set Node  
  - Role: Defines global parameters (Apify Token, thresholds for views, engagement, viral score, max video age)  
  - Configuration: Reads `APIFY_TOKEN` from environment variables; numeric thresholds set (e.g., minViews=10000)  
  - Inputs: From Daily Trigger  
  - Outputs: To Search Config  
  - Edge Cases: Missing or invalid environment variables may cause downstream failures  
  - Version: 3.4

- **Search Config**  
  - Type: Set Node  
  - Role: Defines search parameters including niche keywords, hashtags, language, and timeframe  
  - Configuration: Placeholders for user to set search keywords/hashtags, default language ‘en’, and 24h time window  
  - Inputs: From Workflow Configuration  
  - Outputs: To TikTok and Instagram scraping nodes  
  - Edge Cases: Empty or malformed keywords/hashtags affect scraping results  
  - Version: 3.4

---

#### 1.2 Data Scraping & Normalization

**Overview:**  
Scrapes TikTok and Instagram videos using Apify scraper APIs. Normalizes data into a consistent format for downstream processing.

**Nodes Involved:**  
- Scrape TikTok  
- Scrape Instagram  
- Normalize TikTok Results  
- Normalize Instagram Results  
- Combine Platforms

**Node Details:**  

- **Scrape TikTok**  
  - Type: HTTP Request  
  - Role: Sends POST request to Apify TikTok scraper API with keywords, hashtags, and time window  
  - Configuration: Uses `apifyToken` for Authorization header; JSON body constructed dynamically from Search Config  
  - Inputs: From Search Config  
  - Outputs: To Normalize TikTok Results  
  - Edge Cases: API rate limits, invalid token, or endpoint errors; malformed JSON body  
  - Version: 4.3

- **Scrape Instagram**  
  - Type: HTTP Request  
  - Role: Sends POST request to Apify Instagram Reels scraper API with same search parameters  
  - Configuration: Similar to Scrape TikTok, with Instagram-specific endpoint  
  - Inputs: From Search Config  
  - Outputs: To Normalize Instagram Results  
  - Edge Cases: Same as TikTok scraping  
  - Version: 4.3

- **Normalize TikTok Results**  
  - Type: Set Node  
  - Role: Transforms TikTok API output into unified fields including platform, video URL, author, views, likes, comments, shares, posted date, caption, and thumbnail URL  
  - Configuration: Uses expressions to handle optional fields and fallback values  
  - Inputs: From Scrape TikTok  
  - Outputs: To Combine Platforms  
  - Edge Cases: Missing fields in source data (e.g., null views) handled by fallback defaults  
  - Version: 3.4

- **Normalize Instagram Results**  
  - Type: Set Node  
  - Role: Same normalization for Instagram results into consistent schema  
  - Configuration: Similar field mapping with fallbacks; shares default to 0 (not typically provided)  
  - Inputs: From Scrape Instagram  
  - Outputs: To Combine Platforms  
  - Edge Cases: Same as TikTok normalization  
  - Version: 3.4

- **Combine Platforms**  
  - Type: Merge Node  
  - Role: Merges normalized TikTok and Instagram data into a single stream  
  - Configuration: Default merge mode (combine inputs)  
  - Inputs: From Normalize TikTok and Normalize Instagram (branch 0 and 1)  
  - Outputs: To Filter High-Growth Videos  
  - Edge Cases: None significant; handles empty inputs gracefully  
  - Version: 3.2

---

#### 1.3 Filtering for High-Growth Content

**Overview:**  
Filters merged video data to keep only those with sufficient views, engagement rate, and recency based on configured thresholds.

**Nodes Involved:**  
- Filter High-Growth Videos

**Node Details:**  

- **Filter High-Growth Videos**  
  - Type: Code Node (JavaScript)  
  - Role: Filters videos by minimum views, engagement rate, and maximum age (hours). Calculates engagement rate as (likes + comments + shares) / views and age from posted date. Also appends calculated fields (engagement_rate, age_hours)  
  - Configuration: Reads thresholds dynamically from Workflow Configuration node; processes all merged inputs  
  - Inputs: From Combine Platforms  
  - Outputs: To Analyze Visual Hook (GPT-4 Vision)  
  - Edge Cases: Invalid or missing date fields can cause parsing errors; zero views handled to avoid division by zero  
  - Version: 2

---

#### 1.4 AI-Based Visual and Trend Analysis

**Overview:**  
Uses GPT-4 Vision and GPT-4 language models to analyze video thumbnails and captions, then generates strategic insights about trends and adaptation ideas.

**Nodes Involved:**  
- OpenAI GPT-4 Vision Model  
- Analyze Visual Hook (GPT-4 Vision)  
- OpenAI GPT-4 Model  
- Analyze Trend & Adaptation (GPT-4)

**Node Details:**  

- **OpenAI GPT-4 Vision Model**  
  - Type: LangChain OpenAI Chat LM Node  
  - Role: Provides GPT-4 Vision model for image analysis  
  - Configuration: Model set as “gpt-4o” (GPT-4 with vision capabilities)  
  - Inputs: None (used as AI model resource for agent node)  
  - Outputs: Connected internally to Analyze Visual Hook node  
  - Credentials: Requires OpenAI API credentials  
  - Version: 1.3

- **Analyze Visual Hook (GPT-4 Vision)**  
  - Type: LangChain Agent Node  
  - Role: Analyzes video thumbnail and caption to extract visual hook, primary emotion, and formatting pattern as JSON  
  - Configuration: Uses system message defining role and expected JSON output; input text includes thumbnail URL and caption from filtered video data  
  - Inputs: From Filter High-Growth Videos  
  - Outputs: To Analyze Trend & Adaptation (GPT-4)  
  - Edge Cases: Network or API errors; malformed image URLs; AI interpretation errors  
  - Version: 3

- **OpenAI GPT-4 Model**  
  - Type: LangChain OpenAI Chat LM Node  
  - Role: Provides GPT-4 language model for text-based trend analysis  
  - Configuration: Model “gpt-4o” without vision capabilities  
  - Inputs: None (used as AI model resource for agent node)  
  - Outputs: Connected internally to Analyze Trend & Adaptation node  
  - Credentials: OpenAI API credentials required  
  - Version: 1.3

- **Analyze Trend & Adaptation (GPT-4)**  
  - Type: LangChain Agent Node  
  - Role: Analyzes video details and visual hook analysis; returns JSON with summary, reasons for success, trend type, adaptation ideas, hook examples, and posting angles  
  - Configuration: System message defines analyst role and output schema; inputs include video metadata and visual analysis JSON from previous node  
  - Inputs: From Analyze Visual Hook (GPT-4 Vision) and data fields from filtered videos  
  - Outputs: To Check Super Viral Threshold  
  - Edge Cases: API or network errors; malformed input data; AI output parsing errors  
  - Version: 3

---

#### 1.5 Super Viral Content Identification

**Overview:**  
Determines if videos surpass the “super viral” engagement threshold and flags them accordingly for special handling.

**Nodes Involved:**  
- Check Super Viral Threshold  
- Mark as Super Viral  
- Mark as Normal  
- Merge Viral Flags

**Node Details:**  

- **Check Super Viral Threshold**  
  - Type: If Node  
  - Role: Compares engagement rate of video to super viral threshold from configuration  
  - Configuration: Condition checks if engagement_rate ≥ superViralThreshold (e.g., 0.15)  
  - Inputs: From Analyze Trend & Adaptation (GPT-4)  
  - Outputs: True to Mark as Super Viral; False to Mark as Normal  
  - Edge Cases: Missing engagement_rate field causes false negatives  
  - Version: 2.2

- **Mark as Super Viral**  
  - Type: Set Node  
  - Role: Adds boolean field `super_viral=true` to flagged items  
  - Configuration: Simple assignment  
  - Inputs: From Check Super Viral Threshold (true branch)  
  - Outputs: To Merge Viral Flags and Send Super Viral Alert  
  - Version: 3.4

- **Mark as Normal**  
  - Type: Set Node  
  - Role: Adds `super_viral=false` flag for other videos  
  - Configuration: Simple assignment  
  - Inputs: From Check Super Viral Threshold (false branch)  
  - Outputs: To Merge Viral Flags  
  - Version: 3.4

- **Merge Viral Flags**  
  - Type: Merge Node  
  - Role: Combines super viral and normal flagged videos back into a single stream  
  - Configuration: Default merge mode  
  - Inputs: From Mark as Super Viral and Mark as Normal  
  - Outputs: To Save to Google Sheets and Select Top 5-10 Videos  
  - Version: 3.2

---

#### 1.6 Data Storage & Reporting (Google Sheets & Email)

**Overview:**  
Appends all shortlisted videos with AI insights into Google Sheets and compiles a daily email digest with top videos and detailed analysis.

**Nodes Involved:**  
- Save to Google Sheets  
- Select Top 5-10 Videos  
- Prepare Email Data  
- Send Daily Digest Email

**Node Details:**  

- **Save to Google Sheets**  
  - Type: Google Sheets Node  
  - Role: Appends or updates rows in a specified Google Sheet with video and AI analysis data  
  - Configuration: Maps fields such as date, views, author, engagement rate, AI summary, super viral flag, adaptation ideas, etc.  
  - Inputs: From Merge Viral Flags  
  - Outputs: None (endpoint node)  
  - Credentials: Google Sheets service account required  
  - Edge Cases: API rate limits, invalid sheet ID or name, credential expiration  
  - Version: 4.7

- **Select Top 5-10 Videos**  
  - Type: Aggregate Node  
  - Role: Aggregates all data items for further processing and sorting  
  - Configuration: Uses aggregateAllItemData to collect all videos in one array  
  - Inputs: From Merge Viral Flags  
  - Outputs: To Prepare Email Data  
  - Version: 1

- **Prepare Email Data**  
  - Type: Code Node (JavaScript)  
  - Role: Sorts videos by engagement and calculates growth score (engagement rate / age hours), parses AI JSON outputs, and selects top 5-10 videos for digest  
  - Configuration: Extracts and joins array fields into strings; sorts descending by growth score  
  - Inputs: From Select Top 5-10 Videos  
  - Outputs: To Send Daily Digest Email  
  - Edge Cases: Parsing errors on AI JSON fields; empty input array  
  - Version: 2

- **Send Daily Digest Email**  
  - Type: Gmail Node  
  - Role: Sends a styled HTML email with detailed video cards including AI insights and metrics  
  - Configuration: Uses Handlebars templating with conditionals and loops; subject and recipient email are configurable placeholders  
  - Inputs: From Prepare Email Data  
  - Credentials: Google API service account with Gmail access  
  - Edge Cases: Email sending failures, invalid recipient address  
  - Version: 2.1

---

#### 1.7 Instant Slack Alerts for Super Viral Content

**Overview:**  
Sends immediate Slack notifications for videos flagged as “super viral” with key statistics and AI insights.

**Nodes Involved:**  
- Send Super Viral Alert

**Node Details:**  

- **Send Super Viral Alert**  
  - Type: Slack Node  
  - Role: Posts message to configured Slack channel containing video metadata and AI analysis summary for super viral videos  
  - Configuration: Message text templated with video platform, views, engagement rate, link, reason for virality, and top adaptation idea  
  - Inputs: From Mark as Super Viral  
  - Credentials: Slack API token with appropriate channel posting permissions  
  - Edge Cases: Slack API rate limits, invalid channel ID or token, message template errors  
  - Version: 2.3

---

### 3. Summary Table

| Node Name                    | Node Type                        | Functional Role                           | Input Node(s)                 | Output Node(s)                          | Sticky Note                                                                                              |
|------------------------------|---------------------------------|-----------------------------------------|------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------|
| Daily Trigger (08:00)         | Schedule Trigger                 | Triggers workflow daily at 8:00 AM      | None                         | Workflow Configuration                 | ## Schedule & Config Daily trigger and core settings: search keywords, countries, timeframe, and API credentials. Adjust these nodes to match your niche and data sources. |
| Workflow Configuration        | Set                             | Sets API tokens and threshold parameters| Daily Trigger (08:00)         | Search Config                        | ## Schedule & Config                                                                                      |
| Search Config                 | Set                             | Defines search keywords, hashtags, etc. | Workflow Configuration        | Scrape TikTok, Scrape Instagram        | ## Schedule & Config                                                                                      |
| Scrape TikTok                | HTTP Request                    | Scrapes TikTok videos via Apify API     | Search Config                 | Normalize TikTok Results               | ## Scrape & Normalize Scrapes TikTok and Instagram via Apify, then normalizes results into a single list with consistent fields (views, engagement, author, URL, caption). |
| Scrape Instagram            | HTTP Request                    | Scrapes Instagram reels via Apify API   | Search Config                 | Normalize Instagram Results           | ## Scrape & Normalize                                                                                      |
| Normalize TikTok Results      | Set                             | Normalizes TikTok API data               | Scrape TikTok                 | Combine Platforms                      | ## Scrape & Normalize                                                                                      |
| Normalize Instagram Results   | Set                             | Normalizes Instagram API data            | Scrape Instagram             | Combine Platforms                      | ## Scrape & Normalize                                                                                      |
| Combine Platforms             | Merge                           | Merges TikTok and Instagram data        | Normalize TikTok Results, Normalize Instagram Results | Filter High-Growth Videos            | ## Scrape & Normalize                                                                                      |
| Filter High-Growth Videos     | Code                            | Filters videos by views, engagement, age| Combine Platforms             | Analyze Visual Hook (GPT-4 Vision)    | ## Filter & AI Insights Keeps only fast-growing videos, then uses AI to analyze the visual hook and trend mechanics and suggest how to adapt each idea for your brand. |
| OpenAI GPT-4 Vision Model    | LangChain LM Chat OpenAI        | Provides GPT-4 Vision model              | None                         | Analyze Visual Hook (GPT-4 Vision)    | ## Filter & AI Insights                                                                                     |
| Analyze Visual Hook (GPT-4 Vision) | LangChain Agent              | Analyzes video thumbnail and caption    | Filter High-Growth Videos, OpenAI GPT-4 Vision Model | Analyze Trend & Adaptation (GPT-4)  | ## Filter & AI Insights                                                                                     |
| OpenAI GPT-4 Model           | LangChain LM Chat OpenAI        | Provides GPT-4 language model            | None                         | Analyze Trend & Adaptation (GPT-4)    | ## Filter & AI Insights                                                                                     |
| Analyze Trend & Adaptation (GPT-4) | LangChain Agent              | Analyzes trend data and suggests adaptations | Analyze Visual Hook (GPT-4 Vision), OpenAI GPT-4 Model, Filter High-Growth Videos | Check Super Viral Threshold          | ## Filter & AI Insights                                                                                     |
| Check Super Viral Threshold   | If                              | Determines if video is “super viral”     | Analyze Trend & Adaptation (GPT-4) | Mark as Super Viral, Mark as Normal   | ## Super Viral Logic Marks videos as “super viral” or “normal” based on your thresholds so they can be highlighted separately in alerts and reports. |
| Mark as Super Viral           | Set                             | Flags video with super_viral = true      | Check Super Viral Threshold (true) | Merge Viral Flags, Send Super Viral Alert | ## Super Viral Logic                                                                                         |
| Mark as Normal                | Set                             | Flags video with super_viral = false     | Check Super Viral Threshold (false) | Merge Viral Flags                     | ## Super Viral Logic                                                                                         |
| Merge Viral Flags             | Merge                           | Merges flagged videos back into one stream | Mark as Super Viral, Mark as Normal | Save to Google Sheets, Select Top 5-10 Videos | ## Super Viral Logic                                                                                         |
| Save to Google Sheets         | Google Sheets                   | Stores video data and AI insights        | Merge Viral Flags             | None                                  | ## Sheet & Daily Email Stores all shortlisted videos in Google Sheets and sends a daily digest email with the top picks and AI insights.                      |
| Select Top 5-10 Videos        | Aggregate                      | Aggregates all items for email digest    | Merge Viral Flags             | Prepare Email Data                    | ## Sheet & Daily Email                                                                                      |
| Prepare Email Data            | Code                            | Sorts and formats data for email digest  | Select Top 5-10 Videos        | Send Daily Digest Email               | ## Sheet & Daily Email                                                                                      |
| Send Daily Digest Email       | Gmail                          | Sends daily email digest with insights   | Prepare Email Data            | None                                  | ## Sheet & Daily Email                                                                                      |
| Send Super Viral Alert        | Slack                          | Sends Slack alert for super viral videos | Mark as Super Viral           | None                                  | ## Instant Alerts Sends a quick Slack notification whenever a “super viral” video is detected so you can react immediately.                                  |
| Sticky Note                  | Sticky Note                    | Workflow overview and setup instructions | None                         | None                                  | ## How it works This workflow monitors TikTok and Instagram for fast-growing videos in your niche and sends you a daily digest (plus instant alerts for “super viral” content). Each morning, a schedule trigger runs the search with your keywords/hashtags. Apify Actors scrape fresh TikTok and Instagram videos, then the results are normalized into a single unified dataset. The workflow filters for high-growth content based on views, engagement, and recency. For each shortlisted video, AI analyzes the visual hook and caption, then explains why the trend works and how you can adapt it for your brand. The best videos are saved to Google Sheets and a short email digest is generated with links and AI insights. If any video crosses your “super viral” threshold, a separate alert is sent so you can react quickly. Setup steps: 1. Add your Apify tokens and select the TikTok/Instagram Actors. 2. Configure keywords/hashtags, countries, and timeframe in the Search Config nodes. 3. Set your thresholds for views, engagement, and “super viral”. 4. Connect Google Sheets, email, and Slack credentials. 5. Run once in test mode, review the Sheet + email, then enable the daily trigger. |
| Sticky Note1                 | Sticky Note                    | Notes on schedule and configuration nodes | None                         | None                                  | ## Schedule & Config                                                                                      |
| Sticky Note2                 | Sticky Note                    | Notes on scraping and normalization       | None                         | None                                  | ## Scrape & Normalize                                                                                      |
| Sticky Note3                 | Sticky Note                    | Notes on filtering and AI insights         | None                         | None                                  | ## Filter & AI Insights                                                                                     |
| Sticky Note5                 | Sticky Note                    | Notes on super viral logic                  | None                         | None                                  | ## Super Viral Logic                                                                                         |
| Sticky Note6                 | Sticky Note                    | Notes on Google Sheets and email reporting | None                         | None                                  | ## Sheet & Daily Email                                                                                      |
| Sticky Note7                 | Sticky Note                    | Notes on Slack instant alerts               | None                         | None                                  | ## Instant Alerts                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 08:00 AM (local timezone)  
   - Connect output to next node  

2. **Create a Set Node for Workflow Configuration**  
   - Assign variables:  
     - `apifyToken` from environment variable `APIFY_TOKEN`  
     - `minViews` = 10000  
     - `minEngagementRate` = 0.05  
     - `superViralThreshold` = 0.15  
     - `maxAgeHours` = 24  
   - Connect input from Daily Trigger node  

3. **Create a Set Node for Search Configuration**  
   - Assign variables with placeholders for user input:  
     - `nicheKeywords` (e.g., "fitness tips, productivity hacks")  
     - `hashtags` (e.g., "#fitness #workout")  
     - `languages` = "en"  
     - `timeWindow` = "24h"  
   - Connect input from Workflow Configuration node  

4. **Create HTTP Request Node for TikTok Scraping**  
   - Method: POST  
   - URL: Apify TikTok scraper API endpoint (user provides)  
   - Headers: Authorization with Bearer token from `apifyToken`, Content-Type: application/json  
   - Body JSON: keywords, hashtags, timeWindow from Search Config node using expressions  
   - Connect input from Search Config node  

5. **Create HTTP Request Node for Instagram Scraping**  
   - Same setup as TikTok scraping but with Instagram Reels scraper endpoint  
   - Connect input from Search Config node  

6. **Create Set Node to Normalize TikTok Results**  
   - Map fields from TikTok response to unified schema: platform, video_url, author, views, likes, comments, shares, posted_at, caption, thumbnail_url  
   - Use fallback expressions for missing fields  
   - Connect input from Scrape TikTok node  

7. **Create Set Node to Normalize Instagram Results**  
   - Map Instagram response fields similarly to unified schema  
   - Set shares to 0 as Instagram API may not provide shares count  
   - Connect input from Scrape Instagram node  

8. **Create Merge Node to Combine Platforms**  
   - Merge both normalized TikTok and Instagram data  
   - Connect inputs from Normalize TikTok and Normalize Instagram nodes  

9. **Create Code Node to Filter High-Growth Videos**  
   - Implement JavaScript logic to filter by minViews, minEngagementRate, maxAgeHours  
   - Calculate engagement rate and age in hours  
   - Append calculated fields to output  
   - Connect input from Combine Platforms node  

10. **Create LangChain OpenAI Node for GPT-4 Vision Model**  
    - Use model id "gpt-4o" for vision-capable model  
    - Provide OpenAI API credentials  
    - No input connection (used as sub-node for agent)  

11. **Create LangChain Agent Node for Visual Hook Analysis**  
    - Input text: thumbnail URL and caption from filtered videos  
    - System message specifying analysis role and structured JSON output (visual_hook, main_emotion, formatting_pattern)  
    - Connect input from Filter High-Growth Videos node  
    - Use GPT-4 Vision model node as AI language model  

12. **Create LangChain OpenAI Node for GPT-4 Text Model**  
    - Use model id "gpt-4o" without vision  
    - Provide OpenAI API credentials  
    - No input connection (used as sub-node)  

13. **Create LangChain Agent Node for Trend & Adaptation Analysis**  
    - Input includes niche keywords, video metadata, visual analysis JSON from prior node  
    - System message defines viral content strategist role and required JSON output (summary, why_it_works, trend_type, adaptation_ideas, hook_examples, posting_angles)  
    - Connect input from Analyze Visual Hook node  
    - Use GPT-4 Text model node as AI language model  

14. **Create If Node to Check Super Viral Threshold**  
    - Condition: engagement_rate ≥ superViralThreshold from Workflow Configuration  
    - Connect input from Analyze Trend & Adaptation node  

15. **Create Set Node to Mark as Super Viral**  
    - Set field `super_viral` to true  
    - Connect input from If node's True output  

16. **Create Set Node to Mark as Normal**  
    - Set field `super_viral` to false  
    - Connect input from If node's False output  

17. **Create Merge Node to Combine Viral Flags**  
    - Merge outputs from Mark as Super Viral and Mark as Normal nodes  
    - Connect output to Save to Google Sheets and Select Top 5-10 Videos nodes  

18. **Create Google Sheets Node to Save Data**  
    - Operation: Append or Update rows  
    - Map fields: date, likes, niche, views, author, shares, comments, platform, video_url, AI summary fields, super viral flag, engagement rate, adaptation ideas, hook examples  
    - Configure with Google service account credentials  
    - Specify sheet name and document ID as placeholders  

19. **Create Aggregate Node to Select Top 5-10 Videos**  
    - Aggregate all items for further processing  
    - Connect input from Merge Viral Flags node  

20. **Create Code Node to Prepare Email Data**  
    - Sort videos by engagement rate and growth score (engagement_rate / age_hours)  
    - Parse AI JSON fields safely  
    - Select top 10 videos for email digest  
    - Connect input from Select Top 5-10 Videos node  

21. **Create Gmail Node to Send Daily Digest Email**  
    - Configure recipient email address (placeholder)  
    - Compose HTML email with Handlebars templating showing video cards and AI insights  
    - Use Google service account credentials for Gmail  
    - Connect input from Prepare Email Data node  

22. **Create Slack Node to Send Super Viral Alert**  
    - Configure Slack channel ID or name (placeholder)  
    - Compose alert message with video stats and AI insights  
    - Use Slack API credentials  
    - Connect input from Mark as Super Viral node  

23. **Add Sticky Notes**  
    - Add explanatory sticky notes describing workflow sections and setup instructions (optional for user clarity)  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow monitors TikTok and Instagram for fast-growing videos, analyzes trends using GPT-4, and sends daily reports plus instant alerts. | Overview sticky note in workflow |
| Users must provide Apify tokens and API endpoints for TikTok and Instagram scrapers. | Workflow Configuration note |
| Google Sheets document ID and sheet name must be configured to store results. | Google Sheets node configuration |
| Email recipient and Slack channel must be set to receive reports and alerts. | Gmail and Slack node configuration |
| AI analysis uses GPT-4 models requiring valid OpenAI credentials. | AI nodes credentials section |
| Video growth filtering is based on views, engagement rate, and age thresholds adjustable in Workflow Configuration. | Filter High-Growth Videos node |
| Slack alerts enable rapid reaction to “super viral” content for marketing advantage. | Slack alert sticky note |
| Email digest uses HTML with Handlebars templating to present data clearly and visually. | Send Daily Digest Email node |

---

**Disclaimer:** The text is exclusively derived from an automated n8n workflow respecting content policies and legality. All processed data is public and lawful.