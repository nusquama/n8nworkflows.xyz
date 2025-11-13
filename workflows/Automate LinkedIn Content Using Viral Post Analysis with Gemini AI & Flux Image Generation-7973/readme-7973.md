Automate LinkedIn Content Using Viral Post Analysis with Gemini AI & Flux Image Generation

https://n8nworkflows.xyz/workflows/automate-linkedin-content-using-viral-post-analysis-with-gemini-ai---flux-image-generation-7973


# Automate LinkedIn Content Using Viral Post Analysis with Gemini AI & Flux Image Generation

### 1. Workflow Overview

This workflow automates the generation and publication of LinkedIn content by analyzing viral posts from specified LinkedIn profiles. It leverages Google Gemini AI for content strategy and Flux for image generation, orchestrating the entire pipeline from data acquisition to publishing. The workflow is designed for social media managers and marketers seeking to automate content curation and creation based on trending LinkedIn posts.

**Logical Blocks:**

- **1.1 Scheduled Profile Data Fetching**: Triggered on a schedule, it fetches LinkedIn profile URLs from Google Sheets and processes them in batches.
- **1.2 Post Scraping and Filtering**: Scrapes posts from LinkedIn profiles via API, applies rate limiting, and filters for high-engagement viral posts.
- **1.3 Viral Posts Storage**: Saves filtered viral posts back to Google Sheets.
- **1.4 New Viral Posts Trigger and Analysis**: Watches for new viral posts, filters recent ones, aggregates content trends, and applies AI-driven content strategy.
- **1.5 Content Generation and Publishing**: Converts AI output to structured JSON, generates post images, downloads them, publishes posts on LinkedIn, and updates analyzed posts status.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Profile Data Fetching

**Overview:**  
This block initiates the workflow periodically to retrieve LinkedIn profile URLs for post analysis.

**Nodes Involved:**  
- LinkedIn Content Automation Scheduler  
- Fetch LinkedIn Profile URLs  
- Process Profiles in Batches

**Node Details:**

- **LinkedIn Content Automation Scheduler**  
  - Type: Schedule Trigger  
  - Role: Periodically triggers the workflow (e.g., daily or hourly) to start automation.  
  - Configuration: Default schedule settings (no parameters specified).  
  - Inputs: None (trigger node)  
  - Outputs: Triggers "Fetch LinkedIn Profile URLs"  
  - Failure modes: Schedule misconfiguration or disabled workflow stops triggering.

- **Fetch LinkedIn Profile URLs**  
  - Type: Google Sheets Node  
  - Role: Reads LinkedIn profile URLs stored in a Google Sheet.  
  - Configuration: Uses Google Sheets credentials with read access; sheet and range must be configured to target profile URLs.  
  - Inputs: Trigger from Scheduler  
  - Outputs: Data passed to "Process Profiles in Batches"  
  - Failure modes: Authentication errors, missing or misconfigured sheet/range, empty data.

- **Process Profiles in Batches**  
  - Type: SplitInBatches  
  - Role: Splits LinkedIn profile URLs into manageable batches, enabling controlled API requests.  
  - Configuration: Batch size set (default or custom); allows partial batch processing.  
  - Inputs: Profile URLs array from Sheets  
  - Outputs: Batch sent to "Scrape LinkedIn Posts API"  
  - Failure modes: Batch size misconfiguration, empty input data.

---

#### 2.2 Post Scraping and Filtering

**Overview:**  
This block scrapes posts from LinkedIn profiles, applies rate limiting to comply with API constraints, and filters posts to isolate those with high engagement.

**Nodes Involved:**  
- Scrape LinkedIn Posts API  
- Rate Limiting Delay  
- Filter High-Engagement Posts  
- Save Viral Posts to Sheets

**Node Details:**

- **Scrape LinkedIn Posts API**  
  - Type: HTTP Request  
  - Role: Calls LinkedIn or related API to fetch recent posts for each profile batch.  
  - Configuration: HTTP method, headers (including auth tokens), and API endpoint configured for LinkedIn post retrieval.  
  - Inputs: Batch of LinkedIn profiles from "Process Profiles in Batches"  
  - Outputs: Raw post data to "Rate Limiting Delay"  
  - Failure modes: API rate limits, authentication failures, HTTP errors, malformed responses.

- **Rate Limiting Delay**  
  - Type: Wait  
  - Role: Introduces delay between API calls to avoid hitting LinkedIn API rate limits.  
  - Configuration: Delay duration set (seconds or minutes) to throttle request rate.  
  - Inputs: Posts data from API node  
  - Outputs: Passes to "Filter High-Engagement Posts"  
  - Failure modes: Incorrect delay causing throttling or workflow slowdown.

- **Filter High-Engagement Posts**  
  - Type: Filter  
  - Role: Filters posts based on engagement metrics (likes, comments, shares) to identify viral content.  
  - Configuration: Filter criteria (e.g., likes > threshold).  
  - Inputs: Post data after delay  
  - Outputs: Viral posts to "Save Viral Posts to Sheets"  
  - Failure modes: Missing or incorrect engagement data, filter misconfiguration.

- **Save Viral Posts to Sheets**  
  - Type: Google Sheets  
  - Role: Saves filtered viral post data into a dedicated Google Sheet for downstream processing.  
  - Configuration: Write mode, target sheet and range, column mapping for post data.  
  - Inputs: Filtered viral posts  
  - Outputs: Triggers “Process Profiles in Batches” to loop or end batch  
  - Failure modes: Authentication issues, sheet write errors, quota limits.

---

#### 2.3 New Viral Posts Trigger and Analysis

**Overview:**  
Triggered by new entries in the viral posts sheet, this block filters recent posts, aggregates trending content, and uses AI to devise a LinkedIn content strategy.

**Nodes Involved:**  
- New Post Data Trigger  
- Filter Recent Posts (3 Days)  
- Aggregate Trending Content  
- Google Gemini Chat Model  
- LinkedIn Content Strategy AI  
- JSON Output Parser

**Node Details:**

- **New Post Data Trigger**  
  - Type: Google Sheets Trigger  
  - Role: Watches the viral posts sheet for new rows to process.  
  - Configuration: Sheet and range monitoring, polling or webhook-based trigger.  
  - Inputs: None (trigger node)  
  - Outputs: Detects new viral posts to "Filter Recent Posts (3 Days)"  
  - Failure modes: Trigger misconfiguration, polling delays.

- **Filter Recent Posts (3 Days)**  
  - Type: Filter  
  - Role: Filters posts to only those created within the last 3 days.  
  - Configuration: Date comparison filter on post timestamp.  
  - Inputs: New viral posts  
  - Outputs: Recent posts to "Aggregate Trending Content"  
  - Failure modes: Incorrect date formatting, timezone issues.

- **Aggregate Trending Content**  
  - Type: Aggregate  
  - Role: Aggregates and summarizes post data to identify trending topics or themes.  
  - Configuration: Grouping by topic, counting engagement metrics.  
  - Inputs: Filtered recent posts  
  - Outputs: Aggregated data to "LinkedIn Content Strategy AI"  
  - Failure modes: Aggregation errors, empty input.

- **Google Gemini Chat Model**  
  - Type: AI Language Model (Google Gemini)  
  - Role: Provides AI natural language understanding and processing capabilities.  
  - Configuration: Connected as language model input for the next node, no direct parameters.  
  - Inputs: None (used as resource)  
  - Outputs: LinkedIn Content Strategy AI uses this model.  
  - Failure modes: API connectivity, quota limits, auth errors.

- **LinkedIn Content Strategy AI**  
  - Type: LangChain Agent (AI-driven)  
  - Role: Processes aggregated trending content to generate LinkedIn content strategy and post ideas.  
  - Configuration: Uses Google Gemini for language understanding, configured agent prompts.  
  - Inputs: Aggregated content and AI model reference  
  - Outputs: Structured AI output to "JSON Output Parser" and "Generate Post Image"  
  - Failure modes: AI response errors, prompt misconfiguration.

- **JSON Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI-generated content into structured JSON for downstream processing.  
  - Configuration: Output format enforced (JSON schema).  
  - Inputs: AI content from LinkedIn Content Strategy AI  
  - Outputs: Structured data back to LinkedIn Content Strategy AI (loop or validation)  
  - Failure modes: Parsing errors, malformed AI output.

---

#### 2.4 Content Generation and Publishing

**Overview:**  
Generates images for posts, downloads them, publishes posts on LinkedIn, and updates the status of analyzed posts.

**Nodes Involved:**  
- Generate Post Image  
- Download Generated Image  
- Publish to LinkedIn  
- Extract Analyzed Post URLs  
- Mark Posts as Analyzed

**Node Details:**

- **Generate Post Image**  
  - Type: HTTP Request  
  - Role: Calls Flux Image Generation API to create images based on AI content.  
  - Configuration: HTTP method, headers, payload including AI-generated prompts.  
  - Inputs: AI output from LinkedIn Content Strategy AI  
  - Outputs: Image URL or ID to "Download Generated Image"  
  - Failure modes: API failures, invalid prompts, rate limits.

- **Download Generated Image**  
  - Type: HTTP Request  
  - Role: Downloads the generated image file for use in the LinkedIn post.  
  - Configuration: GET request to image URL, binary data output.  
  - Inputs: Image URL from image generation node  
  - Outputs: Binary image data to "Publish to LinkedIn"  
  - Failure modes: Download failures, broken links.

- **Publish to LinkedIn**  
  - Type: LinkedIn Node  
  - Role: Publishes the composed post with image on LinkedIn using OAuth2 authentication.  
  - Configuration: LinkedIn credentials, post text and image attachment.  
  - Inputs: Post content and image binary data  
  - Outputs: Post confirmation to "Extract Analyzed Post URLs"  
  - Failure modes: Auth errors, API rate limits, post content violations.

- **Extract Analyzed Post URLs**  
  - Type: Code Node  
  - Role: Extracts URLs or IDs of published posts for tracking.  
  - Configuration: Custom JavaScript code parsing LinkedIn API response.  
  - Inputs: LinkedIn post response  
  - Outputs: Post URLs to "Mark Posts as Analyzed"  
  - Failure modes: Parsing errors, API response changes.

- **Mark Posts as Analyzed**  
  - Type: Google Sheets  
  - Role: Updates the viral posts sheet to mark posts that have been processed and published.  
  - Configuration: Write/update mode, target cells for status flags.  
  - Inputs: Extracted post URLs and metadata  
  - Outputs: End of workflow step  
  - Failure modes: Sheet write errors, concurrency issues.

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                                  | Input Node(s)                     | Output Node(s)                         | Sticky Note                                                                                      |
|--------------------------------|-----------------------------------|-------------------------------------------------|----------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| LinkedIn Content Automation Scheduler | Schedule Trigger                  | Starts the workflow on schedule                  | None                             | Fetch LinkedIn Profile URLs            |                                                                                                |
| Fetch LinkedIn Profile URLs    | Google Sheets                     | Reads LinkedIn profile URLs from Sheets          | LinkedIn Content Automation Scheduler | Process Profiles in Batches            |                                                                                                |
| Process Profiles in Batches    | SplitInBatches                   | Splits profiles into batches for API calls       | Fetch LinkedIn Profile URLs       | Scrape LinkedIn Posts API (batch 2), Save Viral Posts to Sheets (batch 1) |                                                                                                |
| Scrape LinkedIn Posts API      | HTTP Request                    | Calls LinkedIn API to scrape posts                | Process Profiles in Batches (batch 2) | Rate Limiting Delay                   |                                                                                                |
| Rate Limiting Delay            | Wait                             | Delays to avoid API rate limits                    | Scrape LinkedIn Posts API         | Filter High-Engagement Posts           |                                                                                                |
| Filter High-Engagement Posts   | Filter                          | Filters posts with high engagement                  | Rate Limiting Delay               | Save Viral Posts to Sheets              |                                                                                                |
| Save Viral Posts to Sheets     | Google Sheets                   | Stores viral posts for further analysis            | Filter High-Engagement Posts      | Process Profiles in Batches (batch 1) |                                                                                                |
| New Post Data Trigger          | Google Sheets Trigger           | Triggers on new viral post data                     | None                             | Filter Recent Posts (3 Days)            |                                                                                                |
| Filter Recent Posts (3 Days)   | Filter                          | Filters posts from last 3 days                       | New Post Data Trigger             | Aggregate Trending Content              |                                                                                                |
| Aggregate Trending Content     | Aggregate                      | Aggregates trending topics from recent posts        | Filter Recent Posts (3 Days)      | LinkedIn Content Strategy AI            |                                                                                                |
| Google Gemini Chat Model       | AI Language Model (Google Gemini) | Provides language model for AI agent                 | None                             | LinkedIn Content Strategy AI (languageModel input) |                                                                                                |
| LinkedIn Content Strategy AI   | LangChain Agent                | Generates content strategy using AI                  | Aggregate Trending Content, Google Gemini Chat Model | Generate Post Image, JSON Output Parser |                                                                                                |
| JSON Output Parser             | LangChain Output Parser        | Parses AI output into structured JSON                | LinkedIn Content Strategy AI      | LinkedIn Content Strategy AI (output parser input) |                                                                                                |
| Generate Post Image            | HTTP Request                    | Calls image generation API (Flux)                    | LinkedIn Content Strategy AI      | Download Generated Image                |                                                                                                |
| Download Generated Image       | HTTP Request                    | Downloads generated image                             | Generate Post Image               | Publish to LinkedIn                     |                                                                                                |
| Publish to LinkedIn            | LinkedIn Node                  | Publishes post with image on LinkedIn                | Download Generated Image          | Extract Analyzed Post URLs              |                                                                                                |
| Extract Analyzed Post URLs     | Code                           | Extracts URLs of published LinkedIn posts             | Publish to LinkedIn               | Mark Posts as Analyzed                  |                                                                                                |
| Mark Posts as Analyzed         | Google Sheets                   | Marks posts as processed in the viral posts sheet    | Extract Analyzed Post URLs        | None                                  |                                                                                                |
| Sticky Note                   | Sticky Note                    | No content                                           | None                             | None                                  |                                                                                                |
| Sticky Note1                  | Sticky Note                    | No content                                           | None                             | None                                  |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named "LinkedIn Content Automation Scheduler" with default parameters to run the workflow periodically.

2. **Add a Google Sheets node** named "Fetch LinkedIn Profile URLs":  
   - Configure with Google Sheets credentials.  
   - Set to read the sheet containing LinkedIn profile URLs.

3. **Add a SplitInBatches node** named "Process Profiles in Batches":  
   - Connect from "Fetch LinkedIn Profile URLs".  
   - Set batch size (e.g., 5 or 10) to limit concurrency.

4. **Add an HTTP Request node** named "Scrape LinkedIn Posts API":  
   - Connect batch output from "Process Profiles in Batches" (second output).  
   - Configure API endpoint for LinkedIn post scraping with necessary headers and auth tokens.

5. **Add a Wait node** named "Rate Limiting Delay":  
   - Connect from "Scrape LinkedIn Posts API".  
   - Set delay duration (e.g., 2 seconds) to comply with API rate limits.

6. **Add a Filter node** named "Filter High-Engagement Posts":  
   - Connect from "Rate Limiting Delay".  
   - Set filter to pass only posts exceeding engagement threshold (e.g., likes > 100).

7. **Add a Google Sheets node** named "Save Viral Posts to Sheets":  
   - Connect from "Filter High-Engagement Posts".  
   - Configure to append viral post data to a dedicated sheet.

8. **Connect the first output of "Process Profiles in Batches" back to "Save Viral Posts to Sheets"** for batch iteration completion.

9. **Add a Google Sheets Trigger node** named "New Post Data Trigger":  
   - Configure to monitor the viral posts sheet for new entries.

10. **Add a Filter node** named "Filter Recent Posts (3 Days)":  
    - Connect from "New Post Data Trigger".  
    - Filter posts with timestamp within last 3 days.

11. **Add an Aggregate node** named "Aggregate Trending Content":  
    - Connect from "Filter Recent Posts (3 Days)".  
    - Configure to aggregate posts by topic, hashtags, or engagement.

12. **Add a Google Gemini Chat Model node** named "Google Gemini Chat Model":  
    - Configure with Google Gemini credentials/APIs.

13. **Add a LangChain Agent node** named "LinkedIn Content Strategy AI":  
    - Connect main input from "Aggregate Trending Content".  
    - Connect AI language model input from "Google Gemini Chat Model".  
    - Configure prompts to generate LinkedIn post ideas and strategy.

14. **Add a LangChain Output Parser node** named "JSON Output Parser":  
    - Connect from "LinkedIn Content Strategy AI".  
    - Configure to parse AI output into structured JSON.

15. **Add an HTTP Request node** named "Generate Post Image":  
    - Connect from "LinkedIn Content Strategy AI".  
    - Configure to call Flux Image Generation API with AI content.

16. **Add an HTTP Request node** named "Download Generated Image":  
    - Connect from "Generate Post Image".  
    - Configure to download image binary.

17. **Add a LinkedIn node** named "Publish to LinkedIn":  
    - Connect from "Download Generated Image".  
    - Configure with LinkedIn OAuth2 credentials.  
    - Set post content and attach downloaded image.

18. **Add a Code node** named "Extract Analyzed Post URLs":  
    - Connect from "Publish to LinkedIn".  
    - Implement JavaScript code to extract published post URLs/IDs from response.

19. **Add a Google Sheets node** named "Mark Posts as Analyzed":  
    - Connect from "Extract Analyzed Post URLs".  
    - Configure to update viral posts sheet marking posts as processed.

20. **Verify connections:**  
    - The scheduler triggers the profile fetch, batch processing, scraping, filtering, saving viral posts.  
    - The sheet trigger detects new viral posts, leading to AI analysis, image generation, publishing, and marking posts as analyzed.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                           |
|--------------------------------------------------------------------------------------------------|------------------------------------------|
| Workflow automates LinkedIn content creation by analyzing viral posts with AI and image generation. | Workflow description                      |
| Google Gemini AI integration requires appropriate API credentials and quota management.          | Google Gemini Chat Model node             |
| LinkedIn API usage requires OAuth2 authentication and compliance with LinkedIn policies.         | Publish to LinkedIn node                  |
| Rate limiting is critical to avoid API throttling on LinkedIn and image generation APIs.         | Rate Limiting Delay node                  |
| Use Google Sheets to store and trigger viral post data for flexible, cloud-based data handling.  | Google Sheets nodes                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.