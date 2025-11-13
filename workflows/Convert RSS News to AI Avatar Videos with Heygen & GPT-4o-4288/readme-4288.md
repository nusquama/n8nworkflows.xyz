Convert RSS News to AI Avatar Videos with Heygen & GPT-4o

https://n8nworkflows.xyz/workflows/convert-rss-news-to-ai-avatar-videos-with-heygen---gpt-4o-4288


# Convert RSS News to AI Avatar Videos with Heygen & GPT-4o

---

### 1. Workflow Overview

This workflow automates the conversion of RSS news feed items into AI-generated avatar videos using Heygen and GPT-4o. It is designed for content creators, marketers, or news aggregators who want to transform textual news summaries into engaging video content automatically.

**Logical Blocks:**

- **1.1 Input Reception:** Trigger and RSS feed reading to ingest news items.
- **1.2 Data Limiting and Logging:** Controls the volume of news processed and logs news data to Google Sheets.
- **1.3 AI Script Generation:** Uses GPT-4o via Langchain to generate video scripts from news content.
- **1.4 Caption Parsing and Avatar Video Setup:** Processes AI-generated scripts to format captions and prepares Heygen avatar video creation requests.
- **1.5 Avatar Video Creation and Retrieval:** Sends requests to Heygen API to create avatar videos, waits for processing, retrieves videos, downloads them, and logs video metadata.
- **1.6 Optional Google Drive Upload:** (Disabled) Intended to upload final videos to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives manual trigger input and reads the latest RSS feed items to start the workflow.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- RSS Read (RSS Feed Reader)  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow execution on user command.  
  - Config: No parameters; triggers workflow when manually tested.  
  - Connections: Outputs to RSS Read.  
  - Edge Cases: None typical; manual trigger requires user action.

- **RSS Read**  
  - Type: RSS Feed Reader  
  - Role: Reads news items from configured RSS feed URL(s).  
  - Config: RSS feed URL(s) must be set in node parameters.  
  - Inputs: Triggered by Manual Trigger node.  
  - Outputs: News item data including title, link, description, publication date.  
  - Edge Cases: RSS feed unreachable, malformed feed, empty feed.

---

#### 2.2 Data Limiting and Logging

**Overview:**  
Limits the number of news items processed and logs initial news metadata to Google Sheets for tracking.

**Nodes Involved:**  
- Limit1  
- log news to sheets  
- Limit  

**Node Details:**

- **Limit1**  
  - Type: Limit  
  - Role: Restricts the maximum number of RSS items processed per run.  
  - Config: Limit count configured to avoid excess processing.  
  - Inputs: RSS Read output.  
  - Outputs: Limited subset of news items.  
  - Edge Cases: Limit too low or too high affecting throughput.

- **log news to sheets**  
  - Type: Google Sheets  
  - Role: Records news metadata (e.g., title, date) into a Google Sheets spreadsheet.  
  - Config: Requires Google Sheets credentials, sheet ID, and column mapping.  
  - Inputs: Output of Limit1 node.  
  - Outputs: Confirmation of logging.  
  - Edge Cases: Authentication errors, sheet access permissions, quota limits.

- **Limit**  
  - Type: Limit  
  - Role: Further limits or controls flow after logging.  
  - Inputs: Output of log news to sheets.  
  - Outputs: Proceeds to AI Agent.  
  - Edge Cases: Similar to Limit1.

---

#### 2.3 AI Script Generation

**Overview:**  
Generates video scripts for each news item by sending content to GPT-4o via Langchain integration.

**Nodes Involved:**  
- AI Agent  
- write script  

**Node Details:**

- **write script**  
  - Type: Langchain OpenAI Chat Model (lmChatOpenAi)  
  - Role: Interfaces with GPT-4o to generate a script from news content.  
  - Config: Uses GPT-4o model, prompt template likely configured for news-to-script transformation.  
  - Inputs: Feeds prompt to AI Agent.  
  - Outputs: Generated script text.  
  - Edge Cases: API rate limits, network errors, prompt failures.

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Coordinates Langchain chains and language models, processes script generation and downstream parsing.  
  - Config: Links to write script node for language model calls.  
  - Inputs: Receives limited news data from Limit node.  
  - Outputs: Sends generated scripts to parse caption node.  
  - Edge Cases: Langchain errors, expression issues, API errors.

---

#### 2.4 Caption Parsing and Avatar Video Setup

**Overview:**  
Parses AI-generated scripts to extract or format captions, then prepares Heygen API requests to create avatar videos.

**Nodes Involved:**  
- parse caption  
- Setup Heygen  
- Create Avatar Video  

**Node Details:**

- **parse caption**  
  - Type: Code (JavaScript)  
  - Role: Parses AI output scripts to extract captions or format text for Heygen.  
  - Config: Custom code processing the AI Agent output.  
  - Inputs: AI Agent output with script.  
  - Outputs: Parsed captions for video creation.  
  - Edge Cases: Code errors, unexpected input format, null data.

- **Setup Heygen**  
  - Type: Set  
  - Role: Sets up HTTP request parameters, headers, and body data required for Heygen video creation API.  
  - Config: Configures authentication tokens, video parameters (e.g., avatar ID, language).  
  - Inputs: Output from parse caption node.  
  - Outputs: HTTP request data for video creation.  
  - Edge Cases: Missing or expired credentials, invalid parameter values.

- **Create Avatar Video**  
  - Type: HTTP Request  
  - Role: Sends POST request to Heygen API to initiate avatar video generation.  
  - Config: Uses Setup Heygen data, endpoint URL, authentication headers.  
  - Inputs: Setup Heygen output.  
  - Outputs: Response containing video creation job ID or status.  
  - Edge Cases: API errors, timeouts, invalid request data.

---

#### 2.5 Avatar Video Creation and Retrieval

**Overview:**  
Waits for video generation, fetches the finished video, downloads it, and logs the video details.

**Nodes Involved:**  
- Wait  
- Get Avatar Video  
- Download video  
- Log video url and title to sheets  

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow to allow Heygen to process video creation asynchronously.  
  - Config: Wait time configured (e.g., fixed delay or webhook ID for async callback).  
  - Inputs: Create Avatar Video output.  
  - Outputs: Triggers Get Avatar Video.  
  - Edge Cases: Insufficient wait causing premature fetch, or excessive delay.

- **Get Avatar Video**  
  - Type: HTTP Request  
  - Role: Polls or requests the generated video URL/status from Heygen API.  
  - Config: Uses job ID or video ID from previous step.  
  - Inputs: Wait node output.  
  - Outputs: Video URL or metadata.  
  - Edge Cases: Job not ready, API errors.

- **Download video**  
  - Type: HTTP Request  
  - Role: Downloads the video file from Heygen to local or cloud storage.  
  - Config: Uses video URL from previous node.  
  - Inputs: Get Avatar Video output.  
  - Outputs: Video binary data or link.  
  - Edge Cases: Download failures, broken URLs.

- **Log video url and title to sheets**  
  - Type: Google Sheets  
  - Role: Logs video URL and associated title back to Google Sheets for record keeping.  
  - Config: Requires Sheets credentials, correct sheet and columns.  
  - Inputs: Get Avatar Video output.  
  - Outputs: Confirmation of logging.  
  - Edge Cases: Auth errors, quota limits.

---

#### 2.6 Optional Google Drive Upload (Disabled)

**Overview:**  
Intended to upload the downloaded video to Google Drive for storage or further distribution. Currently disabled.

**Nodes Involved:**  
- Google Drive (disabled)  

**Node Details:**

- **Google Drive**  
  - Type: Google Drive  
  - Role: Uploads video files to Google Drive folder.  
  - Config: Requires Google Drive OAuth2 credentials, target folder ID.  
  - Inputs: Download video output (currently disconnected/disabled).  
  - Outputs: Confirmation and file metadata.  
  - Edge Cases: Auth failures, storage quota exceeded.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                                | Input Node(s)               | Output Node(s)                      | Sticky Note |
|----------------------------|----------------------------------|------------------------------------------------|-----------------------------|------------------------------------|-------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Workflow start trigger                          | (none)                      | RSS Read                           |             |
| RSS Read                   | RSS Feed Reader                   | Reads news items from RSS feed                  | When clicking ‘Test workflow’ | Limit1                            |             |
| Limit1                     | Limit                            | Limits number of news items processed           | RSS Read                    | log news to sheets                 |             |
| log news to sheets         | Google Sheets                    | Logs news metadata to Google Sheets             | Limit1                      | Limit                             |             |
| Limit                      | Limit                            | Controls flow to AI processing                   | log news to sheets          | AI Agent                         |             |
| AI Agent                   | Langchain Agent                  | Coordinates GPT-4o script generation             | Limit                       | parse caption                    |             |
| write script               | Langchain OpenAI Chat Model      | Generates video scripts with GPT-4o              | AI Agent (ai_languageModel) | AI Agent                        |             |
| parse caption              | Code                            | Parses AI output to format captions              | AI Agent                    | Setup Heygen                     |             |
| Setup Heygen               | Set                             | Prepares Heygen API request parameters           | parse caption               | Create Avatar Video              |             |
| Create Avatar Video        | HTTP Request                    | Sends video creation request to Heygen API       | Setup Heygen                | Wait                            |             |
| Wait                      | Wait                            | Pauses to allow video processing                  | Create Avatar Video         | Get Avatar Video                 |             |
| Get Avatar Video           | HTTP Request                    | Retrieves generated video URL/status              | Wait                       | Download video, Log video url and title to sheets |             |
| Download video             | HTTP Request                    | Downloads video file                               | Get Avatar Video            | Google Drive (disabled)          |             |
| Log video url and title to sheets | Google Sheets                  | Logs video metadata                                | Get Avatar Video            | (none)                          |             |
| Google Drive (disabled)    | Google Drive                    | Uploads video to Google Drive (disabled)          | Download video              | (none)                          |             |
| Sticky Note                | Sticky Note                     | (Visual notes, no functional role)                | (none)                     | (none)                          |             |
| Sticky Note1               | Sticky Note                     | (Visual notes, no functional role)                | (none)                     | (none)                          |             |
| Sticky Note3               | Sticky Note                     | (Visual notes, no functional role)                | (none)                     | (none)                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: When clicking ‘Test workflow’  
   - Type: Manual Trigger  
   - No special parameters.

2. **Add RSS Feed Read Node:**  
   - Name: RSS Read  
   - Type: RSS Feed Reader  
   - Configure feed URL(s) for news source.  
   - Connect Manual Trigger output to RSS Read input.

3. **Add Limit Node (Limit1):**  
   - Type: Limit  
   - Configure max items per run (e.g., 5 or 10) to control processing volume.  
   - Connect RSS Read output to Limit1 input.

4. **Add Google Sheets Node (log news to sheets):**  
   - Configure Google Sheets credentials and target spreadsheet.  
   - Map fields like title, date, link from RSS items to sheet columns.  
   - Connect Limit1 output to this node.

5. **Add another Limit Node (Limit):**  
   - Configure as needed to control flow.  
   - Connect log news to sheets output to Limit node.

6. **Add Langchain OpenAI Chat Model Node (write script):**  
   - Name: write script  
   - Configure credentials for OpenAI GPT-4o model.  
   - Set prompt template to convert news items into video scripts.  
   - No direct input connection; it is called by AI Agent node.

7. **Add Langchain Agent Node (AI Agent):**  
   - Configure to link with write script node as its language model.  
   - Set to receive input from Limit node.  
   - Connect AI Agent output to parse caption node.

8. **Add Code Node (parse caption):**  
   - Use JavaScript to parse AI Agent output and extract captions or reformat text for Heygen API.  
   - Connect AI Agent output to parse caption input.

9. **Add Set Node (Setup Heygen):**  
   - Configure request parameters for Heygen API, including authentication tokens, avatar ID, language, video options.  
   - Connect parse caption output to Setup Heygen input.

10. **Add HTTP Request Node (Create Avatar Video):**  
    - Configure POST request to Heygen avatar video creation endpoint.  
    - Use parameters from Setup Heygen node.  
    - Connect Setup Heygen output to Create Avatar Video input.

11. **Add Wait Node:**  
    - Configure wait time (e.g., fixed delay of 30-60 seconds) to allow Heygen to process video.  
    - Connect Create Avatar Video output to Wait node.

12. **Add HTTP Request Node (Get Avatar Video):**  
    - Configure GET request to Heygen API to retrieve video URL/status using job ID from previous node.  
    - Connect Wait output to Get Avatar Video input.

13. **Add HTTP Request Node (Download video):**  
    - Configure to download video file from URL obtained in Get Avatar Video node.  
    - Connect Get Avatar Video output to Download video input.

14. **Add Google Sheets Node (Log video url and title to sheets):**  
    - Configure to log video metadata (URL, title) into Google Sheets.  
    - Connect Get Avatar Video output to this node.

15. **Optionally Add Google Drive Node:**  
    - Configure Google Drive credentials and target folder.  
    - Connect Download video output to Google Drive input.  
    - Note: This node is disabled in current workflow.

16. **Add Sticky Notes:**  
    - For documentation or visual grouping, add sticky notes with relevant content as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow uses GPT-4o via Langchain integration for script generation.                             | Ensure OpenAI GPT-4o API access and Langchain node configuration.                                        |
| Heygen API requires authentication tokens and avatar setup.                                          | Verify Heygen API documentation for required headers, endpoints, and avatar configuration.               |
| Rate limiting and error handling are critical due to multiple API calls (OpenAI, Heygen, Google Sheets). | Implement retries or error workflows for robustness.                                                     |
| Google Sheets nodes require OAuth2 credentials with appropriate permissions.                          | Ensure Google API access is granted for reading/writing sheets.                                         |
| The Wait node is a simple delay; for production, replace with webhook or polling for video readiness. | This avoids unnecessary delays or premature video fetching.                                             |
| Disabled Google Drive node indicates optional video storage integration.                             | Can be enabled and configured if cloud storage of videos is required.                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---