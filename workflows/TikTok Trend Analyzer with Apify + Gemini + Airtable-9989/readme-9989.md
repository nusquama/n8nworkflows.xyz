TikTok Trend Analyzer with Apify + Gemini + Airtable

https://n8nworkflows.xyz/workflows/tiktok-trend-analyzer-with-apify---gemini---airtable-9989


# TikTok Trend Analyzer with Apify + Gemini + Airtable

### 1. Workflow Overview

This workflow, titled **TikTok Trend Analyzer with Apify + Gemini + Airtable**, is designed to automatically scrape trending TikTok videos, analyze why they go viral using Google Gemini AI, and store both raw data and AI-generated insights into Airtable. This enables marketing analysts, creators, and agencies to research viral content, understand key success factors, and plan creative strategies accordingly.

The workflow is logically divided into three main blocks:

- **1.1 Scheduled Trend Scraper and Data Storage**: Periodically triggers the Apify TikTok Trends Scraper, waits for the scraping to complete, fetches the dataset, and saves trending video metadata into Airtable.

- **1.2 Webhook-Triggered Deep Video Analysis**: Receives a TikTok video URL via webhook, fetches corresponding Airtable record, and sends the video URL to Gemini AI for detailed virality analysis.

- **1.3 AI Insight Extraction and Airtable Update**: Processes Gemini AI output with a language model to extract structured insights (summary, visual hook, audio, and subtitle analysis), then updates the Airtable record with these insights.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trend Scraper and Data Storage

**Overview:**  
This block automates scraping trending TikTok videos every week using Apify, waits to ensure the scrape completes, then fetches the scraped data and stores it into Airtable.

**Nodes Involved:**  
- Weekly Trend Trigger  
- Start TikTok Trends Scraper (Apify)  
- Wait  
- Fetch TikTok Dataset (Apify)  
- Save Trending TikToks to Airtable  
- Wait1  

**Node Details:**

- **Weekly Trend Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow every Monday at midnight (cron expression `0 0 * * 1`) to start scraping trends.  
  - Input: None (time-based trigger)  
  - Output: Starts the next node.  
  - Edge Cases: Cron misconfiguration would prevent triggering; timezone differences may affect schedule.  
  - Version: 1.2

- **Start TikTok Trends Scraper (Apify)**  
  - Type: HTTP Request  
  - Role: Initiates the Apify TikTok Trends Scraper by POSTing parameters like country ("United States"), number of videos (100), proxy type, etc.  
  - Config: Authenticated via HTTP Query Auth with Apify API key credential.  
  - Input: Trigger from schedule node.  
  - Output: Passes execution to "Wait" node.  
  - Edge Cases: API key errors, rate limits, or network issues could cause failure.  
  - Version: 4.2

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 4 minutes to allow the Apify scraping task to complete.  
  - Config: Waits 4 minutes (unit: minutes).  
  - Input: From Apify scraper start node.  
  - Output: Proceeds to fetch dataset.  
  - Edge Cases: If scraping takes longer than 4 minutes, dataset may be incomplete.  
  - Version: 1.1

- **Fetch TikTok Dataset (Apify)**  
  - Type: HTTP Request  
  - Role: GET request to Apify API to retrieve the latest TikTok trends dataset.  
  - Config: Uses same Apify API key credential; continues on error (continueErrorOutput) to avoid halting workflow if data unavailable.  
  - Input: From Wait node or Wait1 node (see branching).  
  - Output: Passes data to Airtable node and also triggers Wait1 node.  
  - Edge Cases: API errors, empty dataset, or rate limiting.  
  - Version: 4.2

- **Save Trending TikToks to Airtable**  
  - Type: Airtable node  
  - Role: Inserts newly fetched TikTok metadata into Airtable table "TikTok V2" within base "Creative Ideation."  
  - Config: Maps key fields (likes, views, shares, caption, username, trending position, sound title, etc.) from JSON data. Typecasting enabled to match Airtable field types.  
  - Input: Dataset from Apify fetch.  
  - Output: None (end of this branch).  
  - Edge Cases: Airtable API limits, invalid field mappings, missing base/table IDs.  
  - Version: 2.1

- **Wait1**  
  - Type: Wait  
  - Role: Additional 10-second pause after dataset fetch to allow any asynchronous processes to stabilize or for retries.  
  - Config: Waits 10 seconds (default unit).  
  - Input: Secondary branch from dataset fetch node.  
  - Output: Loops back to "Fetch TikTok Dataset (Apify)"—likely for retrying or polling until data is ready.  
  - Edge Cases: Infinite loops if dataset never becomes available; consider max retries.  
  - Version: 1.1

---

#### 2.2 Webhook-Triggered Deep Video Analysis

**Overview:**  
This block listens for manual video submissions via webhook, finds the corresponding video record in Airtable, and sends it to Gemini AI for a deep objective analysis of virality factors.

**Nodes Involved:**  
- Receive TikTok Video for Analysis (Webhook)  
- Find Airtable Record by URL  
- Analyze TikTok Video (Gemini 2.5)  

**Node Details:**

- **Receive TikTok Video for Analysis**  
  - Type: Webhook  
  - Role: Listens on path `/creative-ideation` for incoming HTTP requests containing TikTok video URLs to analyze.  
  - Config: No authentication or specific options.  
  - Input: External HTTP POST/GET.  
  - Output: Passes query parameter `media` containing TikTok URL to next node.  
  - Edge Cases: Missing or malformed URL in query; unauthorized access if webhook is public.  
  - Version: 2

- **Find Airtable Record by URL**  
  - Type: Airtable (Search)  
  - Role: Searches Airtable table "TikTok V2" for a record matching the TikTok URL passed via webhook query parameter.  
  - Config: Uses a formula filter: `{Tiktok URL} = "{{ $json.query.media }}"` to find the exact record. Returns max 1 record.  
  - Input: From webhook node.  
  - Output: Passes record data and record ID to AI analysis node.  
  - Edge Cases: No matching record found, multiple matches, Airtable API errors.  
  - Version: 2.1

- **Analyze TikTok Video (Gemini 2.5)**  
  - Type: Google Gemini AI node (Langchain)  
  - Role: Sends the TikTok video URL to Gemini's video analysis model `models/gemini-2.5-flash` to get an objective, data-driven analysis of virality factors.  
  - Config: System prompt instructs Gemini to analyze why the video went viral, focusing on hook, storytelling, trend alignment, and reusability of patterns. Constraints prohibit guessing and require justification.  
  - Input: Airtable record with `Tiktok URL`.  
  - Output: Raw AI textual analysis content.  
  - Edge Cases: API quota limits, model errors, invalid video URLs, network issues.  
  - Version: 1

---

#### 2.3 AI Insight Extraction and Airtable Update

**Overview:**  
This block extracts structured insights from Gemini AI output using a language model extractor, then updates the original Airtable record with these insights for easy access and review.

**Nodes Involved:**  
- Extract Insights (LLM)  
- Update Airtable with AI Insights  

**Node Details:**

- **Extract Insights (LLM)**  
  - Type: Langchain Information Extractor (LLM)  
  - Role: Parses Gemini AI's textual analysis into four structured attributes: AI Summary, Visual Hook Analysis, Audio Analysis, and Subtitle Analysis.  
  - Config: Uses the first part of the textual content (`content.parts[0].text`) as input; requires all four attributes to be extracted with precise descriptions.  
  - Input: Gemini AI output JSON.  
  - Output: JSON with extracted attributes under `output` key.  
  - Edge Cases: LLM extraction errors, incomplete or ambiguous AI output, rate limits.  
  - Version: 1.2

- **Update Airtable with AI Insights**  
  - Type: Airtable (Update)  
  - Role: Updates the matched record in Airtable with extracted AI insights in respective columns (AI Summary, Visual Hook Analysis, Audio Analysis, Subtitle Analysis).  
  - Config: Uses record ID from "Find Airtable Record by URL"; maps extracted fields precisely; typecasting disabled to avoid conversion errors.  
  - Input: Extracted insights JSON and record ID.  
  - Output: None (end of workflow).  
  - Edge Cases: Airtable update failure, mismatch of record ID, API limits.  
  - Version: 2.1

---

### 3. Summary Table

| Node Name                           | Node Type                         | Functional Role                                | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                           |
|-----------------------------------|----------------------------------|------------------------------------------------|-----------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| Weekly Trend Trigger               | Schedule Trigger                 | Initiate weekly TikTok trend scraping          | -                           | Start TikTok Trends Scraper (Apify) | Adjust the cron schedule for how often you want to scrape TikTok trends (default: weekly).           |
| Start TikTok Trends Scraper (Apify) | HTTP Request                    | Start Apify scraper with parameters             | Weekly Trend Trigger         | Wait                            | Ensure your Apify API key is stored in the Credential Manager. Choose your region or modify scrape parameters. |
| Wait                              | Wait                            | Pause to allow Apify scrape to complete         | Start TikTok Trends Scraper  | Fetch TikTok Dataset (Apify)    |                                                                                                     |
| Fetch TikTok Dataset (Apify)      | HTTP Request                    | Retrieve latest trending TikTok data            | Wait, Wait1                 | Save Trending TikToks to Airtable, Wait1 |                                                                                                     |
| Save Trending TikToks to Airtable | Airtable                        | Store TikTok metadata into Airtable             | Fetch TikTok Dataset (Apify) | -                              | Update your Airtable base and table IDs; map fields for Likes, Views, Caption, Username, etc.       |
| Wait1                             | Wait                            | Short pause before retrying fetch                | Fetch TikTok Dataset (Apify) | Fetch TikTok Dataset (Apify)    |                                                                                                     |
| Receive TikTok Video for Analysis | Webhook                        | Receive video URL for AI analysis                | -                           | Find Airtable Record by URL     |                                                                                                     |
| Find Airtable Record by URL       | Airtable                       | Search Airtable record matching TikTok URL      | Receive TikTok Video for Analysis | Analyze TikTok Video (Gemini 2.5) |                                                                                                     |
| Analyze TikTok Video (Gemini 2.5) | Google Gemini AI (Langchain)   | Analyze video virality and extract insights     | Find Airtable Record by URL  | Extract Insights (LLM)          | Review the system prompt to refine how Gemini analyzes virality factors.                             |
| Extract Insights (LLM)            | Langchain Information Extractor | Parse Gemini AI output into structured insights | Analyze TikTok Video (Gemini 2.5) | Update Airtable with AI Insights |                                                                                                     |
| Update Airtable with AI Insights  | Airtable                       | Update Airtable record with AI-generated insights | Extract Insights (LLM)       | -                              | Confirm mapping of AI Summary, Hook, Audio, and Subtitle Analysis columns.                           |
| Sticky Note                      | Sticky Note                    | Documentation and overview                       | -                           | -                              | ## TikTok Trend Analyzer with Apify + Gemini + Airtable... (full content in sticky note)             |
| Sticky Note1                     | Sticky Note                    | Apify API key and scraping parameters reminder  | -                           | -                              | Ensure your Apify API key is stored in the Credential Manager. Choose your region or modify scrape parameters. |
| Sticky Note2                     | Sticky Note                    | Airtable base and table ID mapping reminder     | -                           | -                              | Update your Airtable base and table IDs; map fields for Likes, Views, Caption, Username, etc.       |
| Sticky Note3                     | Sticky Note                    | Cron schedule customization note                 | -                           | -                              | Adjust the cron schedule for how often you want to scrape TikTok trends (default: weekly).           |
| Sticky Note4                     | Sticky Note                    | Airtable AI columns mapping confirmation         | -                           | -                              | Confirm mapping of AI Summary, Hook, Audio, and Subtitle Analysis columns.                           |
| Sticky Note5                     | Sticky Note                    | Gemini system prompt review note                  | -                           | -                              | Review the system prompt to refine how Gemini analyzes virality factors.                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation**  
   - Add a sticky note with the full workflow overview and instructions.  
   - Add sticky notes reminding about Apify API key, Airtable base/table IDs, schedule, and AI mapping.

2. **Setup Scheduled TikTok Trend Scraper**  
   - Add a **Schedule Trigger** node: Set cron expression to `0 0 * * 1` to run weekly on Mondays at midnight.  
   - Add an **HTTP Request** node ("Start TikTok Trends Scraper (Apify)"):  
     - Method: POST  
     - URL: `https://api.apify.com/v2/acts/fetchly~tiktok-trends-api/runs`  
     - Body (JSON):  
       ```json
       {
         "country": "United States",
         "downloadCoverImage": false,
         "downloadVideo": false,
         "generatePermalinks": false,
         "numberOfVideos": 100,
         "proxyType": "RESIDENTIAL",
         "useApifyProxy": true
       }
       ```  
     - Authentication: HTTP Query Auth with Apify API key credential.  
   - Connect Schedule Trigger to this node.

3. **Add Wait Node to Allow Scraping Completion**  
   - Add a **Wait** node: Wait for 4 minutes.  
   - Connect from Apify HTTP Request node.

4. **Fetch TikTok Dataset**  
   - Add an **HTTP Request** node ("Fetch TikTok Dataset (Apify)"):  
     - Method: GET  
     - URL: `https://api.apify.com/v2/acts/fetchly~tiktok-trends-api/runs/last/dataset/items`  
     - Authentication: HTTP Query Auth with same Apify API key.  
     - On Error: Continue (to avoid workflow halt).  
   - Connect from Wait node.

5. **Save Trending TikToks to Airtable**  
   - Add an **Airtable** node:  
     - Operation: Create  
     - Base: Your Airtable base ID (e.g., Creative Ideation)  
     - Table: TikTok V2  
     - Map fields accordingly: Likes, Views, Shares, Caption, Username, TikTok URL, Sound Title, Trending Position, Downloaded Video URL etc.  
     - Enable typecast for correct data typing.  
     - Use Airtable API token credentials.  
   - Connect from Fetch TikTok Dataset node.

6. **Add a Short Wait and Loop for Dataset Availability**  
   - Add a **Wait** node ("Wait1"): Wait 10 seconds.  
   - Connect secondary output of Fetch TikTok Dataset node to Wait1.  
   - Connect Wait1 output back to Fetch TikTok Dataset node to poll until data is ready or max retries implemented.

7. **Setup Webhook for Manual Video Analysis**  
   - Add a **Webhook** node:  
     - Path: `creative-ideation`  
     - No authentication by default (secure as needed).  
   - This node receives requests with a query parameter `media` containing the TikTok URL.

8. **Search Airtable Record by URL**  
   - Add an **Airtable** node:  
     - Operation: Search  
     - Base and Table same as above.  
     - Filter By Formula: `{Tiktok URL} = "{{ $json.query.media }}"`  
     - Limit: 1  
     - Use Airtable credentials.  
   - Connect webhook output to this node.

9. **Analyze TikTok Video Using Gemini AI**  
   - Add a **Google Gemini AI (Langchain)** node:  
     - Model: `models/gemini-2.5-flash`  
     - Resource: video  
     - Operation: analyze  
     - Video URLs: Use expression `={{ $json['Tiktok URL'] }}` from Airtable record.  
     - System prompt: Provide detailed instructions to analyze virality factors objectively, no guessing, justify insights, focus on hook, storytelling, trend alignment, reuse patterns.  
   - Connect Airtable search node output to this node.

10. **Extract Structured Insights with LLM**  
    - Add a **Langchain Information Extractor** node:  
      - Text input: Use Gemini AI output text (`{{$json.content.parts[0].text}}`).  
      - Attributes to extract:  
        - ai_summary (concise summary)  
        - visual_hook_analysis  
        - audio_analysis  
        - subtitle_analysis  
    - Connect Gemini AI node output to this node.

11. **Update Airtable Record with AI Insights**  
    - Add an **Airtable** node:  
      - Operation: Update  
      - Base and Table as before.  
      - Use record ID from the Airtable search node (`={{ $('Find Airtable Record by URL').item.json.id }}`).  
      - Map extracted attributes to respective fields: AI Summary, Visual Hook Analysis, Audio Analysis, Subtitle Analysis.  
      - Do not convert fields to string; disable typecasting to avoid errors.  
      - Use Airtable credentials.  
    - Connect extraction node output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automatically scrapes TikTok trends weekly, analyzes video virality with Gemini AI, and stores raw and AI-augmented data in Airtable for creative insights.                                                           | Workflow purpose                                                                                   |
| Ensure your Apify API key is securely stored in n8n Credential Manager and referenced in HTTP request nodes. You may adjust scraping parameters such as country, video count, and proxy type in the payload.                      | Apify integration                                                                                  |
| Update Airtable base and table IDs to match your own Airtable setup. Map fields carefully to avoid data loss or mapping errors.                                                                                                | Airtable configuration                                                                             |
| The webhook endpoint `/creative-ideation` allows manual submission for deep AI analysis of specific videos by URL. Secure this endpoint as needed to prevent unauthorized access.                                              | Webhook usage                                                                                      |
| Gemini AI system prompt is critical to extract objective and justified analysis. Review and adjust the prompt in the "Analyze TikTok Video (Gemini 2.5)" node to fine-tune AI behavior.                                        | AI prompt engineering                                                                              |
| Consider adding retry logic or maximum retry count to avoid infinite loops in dataset fetching. Monitor API rate limits for Apify, Airtable, and Google Gemini to avoid service disruptions.                                     | Reliability and error handling                                                                     |
| Airtable API token used should have sufficient permissions for read, create, and update operations on the specified base and tables.                                                                                          | Airtable credential requirements                                                                  |
| More information on Apify TikTok Trends API: https://apify.com/fetchly/tiktok-trends-api                                                                                                                                    | External resource                                                                                  |
| Google's Gemini AI model documentation and Langchain integration guidelines can help optimize AI node configuration.                                                                                                           | External resource                                                                                  |

---

This completes the detailed, structured reference for the **TikTok Trend Analyzer with Apify + Gemini + Airtable** workflow. It provides a clear understanding of the workflow's architecture, node configurations, potential failure points, and instructions to recreate the entire process from scratch.