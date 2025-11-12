Multilingual YouTube Metadata Translator with Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/multilingual-youtube-metadata-translator-with-gemini-ai-and-google-sheets-5902


# Multilingual YouTube Metadata Translator with Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow automates the process of translating and updating YouTube video metadata (titles, descriptions, etc.) into multiple languages using Google Gemini AI and manages the data via Google Sheets. It is designed for content creators or managers who want to localize their YouTube video metadata efficiently and track the translation status. The workflow integrates with YouTube's API, Google Sheets, and Google Gemini's AI chat model for translation and metadata generation.

Logical blocks:

- **1.1 Trigger & Initialization:** Scheduling and preparing input data.
- **1.2 Data Acquisition & Validation:** Extracting video information from YouTube and validating video IDs.
- **1.3 Language Data Handling:** Fetching and processing target language lists from Google Sheets.
- **1.4 AI Processing:** Utilizing Google Gemini AI and Langchain to translate and generate metadata for each language.
- **1.5 Metadata Update & Status Tracking:** Updating YouTube video metadata via API and reflecting success/error statuses in Google Sheets.
- **1.6 Looping & Data Merging:** Handling batch processing of multiple languages and merging results.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization

- **Overview:**  
  This block initiates the workflow on a scheduled basis and sets up initial data for processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Prepare Values

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Trigger node  
    - Role: Starts the workflow on a defined schedule (cron or interval, default not specified)  
    - Configuration: Default scheduling trigger, no parameters detailed  
    - Inputs: None (trigger)  
    - Outputs: Connects to "Prepare Values"  
    - Edge cases: Trigger misconfiguration or failure to start at scheduled time.

  - **Prepare Values**  
    - Type: Set node  
    - Role: Prepares initial values or variables for subsequent steps  
    - Configuration: Sets key-value pairs or variables, exact parameters not shown  
    - Inputs: Output from Schedule Trigger  
    - Outputs: Connects to "Get Auto Crawl Status" and "Get Video URL"  
    - Edge cases: Missing or incorrect default values could cause downstream errors.

---

#### 1.2 Data Acquisition & Validation

- **Overview:**  
  Retrieves the latest YouTube video data, extracts video IDs, and verifies if the video's metadata has already been processed.

- **Nodes Involved:**  
  - Get Auto Crawl Status  
  - Check Auto Crawl Status  
  - Youtube - Get Lastest Video  
  - Check Latest Video ID  
  - If Video ID Not Exist Yet  
  - Insert Video Info  
  - Get Video URL  
  - Extract Video ID  
  - Get Video Info

- **Node Details:**  

  - **Get Auto Crawl Status**  
    - Type: Google Sheets node  
    - Role: Reads the status of automatic crawling from a Google Sheet  
    - Configuration: Reads a specified sheet and range (not detailed)  
    - Inputs: From "Prepare Values"  
    - Outputs: Connects to "Check Auto Crawl Status"  
    - Edge cases: Sheet access permissions, empty or malformed data.

  - **Check Auto Crawl Status**  
    - Type: If node  
    - Role: Conditional check on whether auto crawl is enabled  
    - Configuration: Condition based on the value retrieved from the previous node  
    - Inputs: From "Get Auto Crawl Status"  
    - Outputs: True branch to "Youtube - Get Lastest Video", false branch terminates or skips  
    - Edge cases: Unexpected values causing wrong conditional flow.

  - **Youtube - Get Lastest Video**  
    - Type: YouTube node  
    - Role: Fetches the latest video uploaded to the YouTube channel  
    - Configuration: Uses YouTube API with channel credentials (likely OAuth2), parameters to fetch recent video  
    - Inputs: From "Check Auto Crawl Status"  
    - Outputs: Connects to "Check Latest Video ID"  
    - Edge cases: API quota limits, authorization errors.

  - **Check Latest Video ID**  
    - Type: Google Sheets node  
    - Role: Checks if the latest video ID exists in the sheet (i.e., if already processed)  
    - Configuration: Reads sheet and looks for the video ID  
    - Inputs: From "Youtube - Get Lastest Video"  
    - Outputs: Connects to "If Video ID Not Exist Yet"  
    - Edge cases: Missing or inconsistent data, authentication issues.

  - **If Video ID Not Exist Yet**  
    - Type: If node  
    - Role: Branches workflow depending on video ID existence  
    - Configuration: Condition checks presence or absence of video ID in sheet  
    - Inputs: From "Check Latest Video ID"  
    - Outputs: True to "Insert Video Info", false skips or continues workflow  
    - Edge cases: Logic errors leading to repeated inserts or skipped updates.

  - **Insert Video Info**  
    - Type: Google Sheets node  
    - Role: Inserts new video information into Google Sheets for tracking  
    - Configuration: Writes data into specified sheet and range  
    - Inputs: From "If Video ID Not Exist Yet"  
    - Outputs: Connects to "Get Video URL"  
    - Edge cases: Write permission errors, rate limits.

  - **Get Video URL**  
    - Type: Google Sheets node  
    - Role: Retrieves URLs of videos to process from the sheet  
    - Configuration: Reads specific columns containing video URLs  
    - Inputs: From "Prepare Values" or "Insert Video Info"  
    - Outputs: Connects to "Extract Video ID"  
    - Edge cases: Malformed URLs, missing data.

  - **Extract Video ID**  
    - Type: Code node (JavaScript)  
    - Role: Parses YouTube video ID from URL strings  
    - Configuration: Uses regex or string manipulation to extract video ID  
    - Inputs: From "Get Video URL"  
    - Outputs: Connects to "Get Video Info"  
    - Edge cases: Invalid URL format, empty input.

  - **Get Video Info**  
    - Type: YouTube node  
    - Role: Fetches detailed video information using YouTube API  
    - Configuration: Uses video ID to query YouTube API for metadata  
    - Inputs: From "Extract Video ID"  
    - Outputs: Connects to "Get Language List"  
    - Edge cases: API failures, missing video, quota limits.

---

#### 1.3 Language Data Handling

- **Overview:**  
  Retrieves the list of target languages for translation from Google Sheets and prepares the language data for processing.

- **Nodes Involved:**  
  - Get Language List  
  - Loop Over Items  
  - Parse Data To JSON  
  - Set Languages - New

- **Node Details:**  

  - **Get Language List**  
    - Type: Google Sheets node  
    - Role: Reads a list of languages (codes and names) from a sheet  
    - Configuration: Reads a defined range with language data  
    - Inputs: From "Get Video Info"  
    - Outputs: Connects to "Loop Over Items"  
    - Edge cases: Empty lists, sheet access issues.

  - **Loop Over Items**  
    - Type: SplitInBatches node  
    - Role: Processes the language list in batches (handles each language as an item)  
    - Configuration: No specific batch size shown (default or customized)  
    - Inputs: From "Get Language List" and also connected from "Set Languages - New"  
    - Outputs: Two outputs: one to "Merge All Languages," another to "AI" node  
    - Edge cases: Batch size limits, performance bottlenecks.

  - **Parse Data To JSON**  
    - Type: Code node  
    - Role: Converts sheet data into JSON format for easier manipulation  
    - Configuration: Custom JavaScript to parse raw sheet data  
    - Inputs: From "AI" node  
    - Outputs: Connects to "Set Languages - New"  
    - Edge cases: Parsing errors if data is malformed.

  - **Set Languages - New**  
    - Type: Set node  
    - Role: Sets or modifies language-related variables for the batch  
    - Configuration: Defines or updates variables related to language selection  
    - Inputs: From "Parse Data To JSON"  
    - Outputs: Connects back to "Loop Over Items" (creating a loop)  
    - Edge cases: Misconfiguration causing infinite loops or missing data.

---

#### 1.4 AI Processing

- **Overview:**  
  Uses Google Gemini AI and Langchain integration to translate and generate localized metadata for each language batch.

- **Nodes Involved:**  
  - Google Gemini's Chat Model  
  - AI

- **Node Details:**  

  - **Google Gemini's Chat Model**  
    - Type: Langchain Google Gemini AI chat model node  
    - Role: Calls Google Gemini AI chat model to process translation requests  
    - Configuration: Uses credentials for Google Gemini AI, parameters include prompts and context (not detailed)  
    - Inputs: AI language model input from "AI" node  
    - Outputs: Connects to "AI" node as ai_languageModel output  
    - Edge cases: API limits, authentication failures, prompt formatting errors.

  - **AI**  
    - Type: Langchain chain LLM node  
    - Role: Processes language model chains for metadata translation and generation  
    - Configuration: Likely configured with prompt templates and parameters for translation tasks  
    - Inputs: From "Loop Over Items" (batch languages) and AI language model output  
    - Outputs: Connects to "Parse Data To JSON" for further data processing  
    - Edge cases: Chain execution failures, invalid outputs, timeout.

---

#### 1.5 Metadata Update & Status Tracking

- **Overview:**  
  Updates the YouTube video metadata with the translated content and manages the update status in Google Sheets.

- **Nodes Involved:**  
  - Merge All Languages  
  - Update Video Metadata with Selected Languages  
  - Update Status - Success  
  - Update Status - Error

- **Node Details:**  

  - **Merge All Languages**  
    - Type: Code node  
    - Role: Combines translated metadata from all language batches into a single structured payload  
    - Configuration: Custom JavaScript merging arrays or objects  
    - Inputs: From "Loop Over Items" (main output)  
    - Outputs: Connects to "Update Video Metadata with Selected Languages"  
    - Edge cases: Merge conflicts, empty results.

  - **Update Video Metadata with Selected Languages**  
    - Type: HTTP Request node  
    - Role: Sends API requests to YouTube to update video metadata localized fields  
    - Configuration: HTTP PATCH or PUT to YouTube API endpoints, includes authorization headers (OAuth2), payload includes localization data  
    - Inputs: From "Merge All Languages"  
    - Outputs:  
      - Success branch to "Update Status - Success"  
      - Error branch to "Update Status - Error" (onError set to continue with error output)  
    - Edge cases: API authentication errors, rate limits, malformed requests, partial update failures.

  - **Update Status - Success**  
    - Type: Google Sheets node  
    - Role: Logs successful metadata update status in the sheet  
    - Configuration: Writes success flags or timestamps in Google Sheets  
    - Inputs: From success output of "Update Video Metadata with Selected Languages"  
    - Outputs: None (end of success flow)  
    - Edge cases: Write permission issues.

  - **Update Status - Error**  
    - Type: Google Sheets node  
    - Role: Logs errors or failure status in Google Sheets  
    - Configuration: Writes error detail or flags in the sheet for monitoring  
    - Inputs: From error output of "Update Video Metadata with Selected Languages"  
    - Outputs: None (end of error flow)  
    - Edge cases: Same as above.

---

#### 1.6 Looping & Data Merging

- **Overview:**  
  Manages the batching and looping of language processing and consolidates results.

- **Nodes Involved:**  
  - Loop Over Items  
  - Merge All Languages

- **Node Details:**  

  - **Loop Over Items**  
    - Already described above; manages iterative processing per language.

  - **Merge All Languages**  
    - Already described above; merges processed language data before updating YouTube.

---

### 3. Summary Table

| Node Name                             | Node Type                    | Functional Role                            | Input Node(s)                       | Output Node(s)                                | Sticky Note                                             |
|-------------------------------------|------------------------------|-------------------------------------------|-----------------------------------|-----------------------------------------------|---------------------------------------------------------|
| Schedule Trigger                    | Schedule Trigger             | Initiates workflow on schedule            | None                              | Prepare Values                                |                                                         |
| Prepare Values                     | Set                          | Prepares initial variables                 | Schedule Trigger                  | Get Auto Crawl Status, Get Video URL          |                                                         |
| Get Auto Crawl Status              | Google Sheets                | Reads auto crawl status                     | Prepare Values                   | Check Auto Crawl Status                        |                                                         |
| Check Auto Crawl Status            | If                           | Checks if auto crawl is enabled            | Get Auto Crawl Status            | Youtube - Get Lastest Video (true), none (false) |                                                         |
| Youtube - Get Lastest Video        | YouTube                      | Fetches latest video info                   | Check Auto Crawl Status           | Check Latest Video ID                          |                                                         |
| Check Latest Video ID              | Google Sheets                | Checks if video ID already exists           | Youtube - Get Lastest Video       | If Video ID Not Exist Yet                      |                                                         |
| If Video ID Not Exist Yet          | If                           | Branches if video ID is new                  | Check Latest Video ID             | Insert Video Info                             |                                                         |
| Insert Video Info                 | Google Sheets                | Inserts new video info                       | If Video ID Not Exist Yet         | Get Video URL                                 |                                                         |
| Get Video URL                    | Google Sheets                | Retrieves video URLs                         | Prepare Values, Insert Video Info | Extract Video ID                              |                                                         |
| Extract Video ID                 | Code                         | Extracts YouTube video ID from URL          | Get Video URL                    | Get Video Info                                |                                                         |
| Get Video Info                  | YouTube                      | Fetches detailed video metadata              | Extract Video ID                 | Get Language List                             |                                                         |
| Get Language List               | Google Sheets                | Retrieves list of target languages           | Get Video Info                  | Loop Over Items                               |                                                         |
| Loop Over Items                | SplitInBatches               | Processes languages in batches               | Get Language List, Set Languages - New | Merge All Languages, AI                     |                                                         |
| Parse Data To JSON             | Code                         | Converts raw data to JSON                     | AI                             | Set Languages - New                           |                                                         |
| Set Languages - New            | Set                          | Sets language variables                       | Parse Data To JSON              | Loop Over Items                               |                                                         |
| Google Gemini's Chat Model     | Langchain Google Gemini AI   | Calls Google Gemini AI for translation       | AI languageModel input           | AI                                            |                                                         |
| AI                            | Langchain Chain LLM          | Controls AI chain for translation             | Loop Over Items, Google Gemini's Chat Model | Parse Data To JSON                     |                                                         |
| Merge All Languages            | Code                         | Merges translated metadata from batches      | Loop Over Items (main output)    | Update Video Metadata with Selected Languages |                                                         |
| Update Video Metadata with Selected Languages | HTTP Request              | Sends API request to update YouTube metadata | Merge All Languages             | Update Status - Success (success), Update Status - Error (error) |                                                         |
| Update Status - Success        | Google Sheets                | Logs successful update in sheet               | Update Video Metadata with Selected Languages (success) | None                                  |                                                         |
| Update Status - Error          | Google Sheets                | Logs error update in sheet                     | Update Video Metadata with Selected Languages (error) | None                                  |                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure desired schedule (e.g., daily at a specific time).  

2. **Create Set Node ("Prepare Values")**  
   - Type: Set  
   - Define initial variables (e.g., sheet IDs, ranges, default flags).  
   - Connect Schedule Trigger output to this node.

3. **Create Google Sheets Node ("Get Auto Crawl Status")**  
   - Type: Google Sheets (Read)  
   - Configure credentials for Google Sheets API.  
   - Set sheet ID and range to read auto crawl status flag.  
   - Connect from Prepare Values.

4. **Create If Node ("Check Auto Crawl Status")**  
   - Type: If  
   - Condition: Check if auto crawl status flag equals enabled value.  
   - Connect from Get Auto Crawl Status.

5. **Create YouTube Node ("Youtube - Get Lastest Video")**  
   - Type: YouTube API node  
   - Configure OAuth2 credentials for YouTube.  
   - Set API call to fetch the most recent video of the channel.  
   - Connect from true branch of Check Auto Crawl Status.

6. **Create Google Sheets Node ("Check Latest Video ID")**  
   - Type: Google Sheets (Read)  
   - Configure to read sheet containing processed video IDs.  
   - Connect from Youtube - Get Lastest Video.

7. **Create If Node ("If Video ID Not Exist Yet")**  
   - Type: If  
   - Condition: Check if the latest video ID exists in the sheet or not.  
   - Connect from Check Latest Video ID.

8. **Create Google Sheets Node ("Insert Video Info")**  
   - Type: Google Sheets (Write)  
   - Configure to insert new video info (ID, URL, timestamps).  
   - Connect from true branch of If Video ID Not Exist Yet.

9. **Create Google Sheets Node ("Get Video URL")**  
   - Type: Google Sheets (Read)  
   - Configure to read video URLs from sheet.  
   - Connect from Insert Video Info and Prepare Values.

10. **Create Code Node ("Extract Video ID")**  
    - Type: Code  
    - JavaScript to parse video ID from URL string using regex.  
    - Connect from Get Video URL.

11. **Create YouTube Node ("Get Video Info")**  
    - Type: YouTube API node  
    - Use video ID to fetch detailed video metadata.  
    - Connect from Extract Video ID.

12. **Create Google Sheets Node ("Get Language List")**  
    - Type: Google Sheets (Read)  
    - Configure to retrieve list of target languages (codes and names).  
    - Connect from Get Video Info.

13. **Create SplitInBatches Node ("Loop Over Items")**  
    - Type: SplitInBatches  
    - Configure batch size as appropriate (default 1 or more).  
    - Connect from Get Language List.

14. **Create Code Node ("Parse Data To JSON")**  
    - Type: Code  
    - JavaScript to parse raw data to JSON format for language processing.  
    - Connect from AI node (see below).

15. **Create Set Node ("Set Languages - New")**  
    - Type: Set  
    - Set language-specific variables for each batch (e.g., language code, name).  
    - Connect from Parse Data To JSON.

16. **Loop Back Connection**  
    - Connect the output of Set Languages - New back to Loop Over Items to ensure batch processing.

17. **Create Langchain Google Gemini AI Node ("Google Gemini's Chat Model")**  
    - Type: Langchain Google Gemini Chat Model  
    - Configure credentials for Google Gemini AI.  
    - Define prompt templates for translation or metadata generation.  
    - Connect ai_languageModel output to AI node.

18. **Create Langchain Chain LLM Node ("AI")**  
    - Type: Langchain Chain LLM  
    - Configure chain with necessary prompt templates, parameters, and language batch data.  
    - Connect from Loop Over Items (batch input) and Google Gemini's Chat Model (languageModel input).  
    - Connect main output to Parse Data To JSON.

19. **Create Code Node ("Merge All Languages")**  
    - Type: Code  
    - Merge individual translated metadata objects into a single payload.  
    - Connect from Loop Over Items main output.

20. **Create HTTP Request Node ("Update Video Metadata with Selected Languages")**  
    - Type: HTTP Request  
    - Configure to PATCH or PUT YouTube API endpoint for video localization update.  
    - Use OAuth2 credentials with YouTube scope.  
    - Payload includes merged localized metadata.  
    - Connect from Merge All Languages.  
    - Configure error handling: On error, continue with error output.

21. **Create Google Sheets Node ("Update Status - Success")**  
    - Type: Google Sheets (Write)  
    - Logs success status with timestamp or flag.  
    - Connect from success output of Update Video Metadata with Selected Languages.

22. **Create Google Sheets Node ("Update Status - Error")**  
    - Type: Google Sheets (Write)  
    - Logs error details for failed updates.  
    - Connect from error output of Update Video Metadata with Selected Languages.

23. **Verify all connections and credentials:**  
    - Ensure OAuth2 credentials for YouTube and Google Sheets are properly set and authorized.  
    - Validate access to Google Gemini AI with correct API keys or OAuth tokens.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini AI via Langchain integration for advanced multilingual translation and generation.   | n8n integration with Langchain and Google Gemini API                                            |
| Google Sheets is the central data store for video URLs, languages, status flags, and metadata tracking.               | Google Sheets API v4 configured with OAuth2 credentials                                          |
| YouTube API calls require OAuth2 credentials with appropriate scopes for reading video info and updating metadata.   | YouTube Data API v3 documentation: https://developers.google.com/youtube/v3                     |
| Proper handling of API quota limits and error conditions is essential to avoid partial failures or data loss.         | Consider using n8n retry and error handling features                                           |
| Batch processing with SplitInBatches node is crucial for handling multiple languages without API overload.            | n8n SplitInBatches documentation: https://docs.n8n.io/nodes/n8n-nodes-base.splitInBatches/       |

---

**Disclaimer:**  
The text provided derives exclusively from an automated workflow built with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.