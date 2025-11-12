AI-Powered Instagram Content Repurposing with OpenAI GPT-4O & Perplexity Research

https://n8nworkflows.xyz/workflows/ai-powered-instagram-content-repurposing-with-openai-gpt-4o---perplexity-research-9445


# AI-Powered Instagram Content Repurposing with OpenAI GPT-4O & Perplexity Research

### 1. Workflow Overview

This workflow automates the repurposing of Instagram video content for an AI & automation-focused Instagram channel by leveraging OpenAI’s GPT-4O model and Perplexity Research API. It retrieves Instagram posts from specified profiles, downloads and transcribes videos, analyzes transcript content for AI tools or technologies, enriches the data with external research, and generates polished video scripts optimized for social media engagement. Finally, it updates a Google Sheet with metadata and new scripts for editorial use.

The workflow is logically divided into four main blocks:

- **1.1 Data Input & Video Retrieval**: Fetch Instagram handles from Google Sheets, loop through them, and retrieve the latest video posts.
- **1.2 Video Processing & AI Transcription**: Download videos and transcribe audio to text using OpenAI Whisper.
- **1.3 AI Content Analysis & Enhancement**: Analyze transcripts via GPT-4O to detect AI tools, research them with Perplexity AI, and generate refined scripts.
- **1.4 Data Storage & Loop Management**: Store results back into Google Sheets and manage batch processing loop for multiple profiles.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Input & Video Retrieval

**Overview:**  
This block initiates the workflow, reads Instagram profile handles from a Google Sheet, processes each handle in batches, and fetches the latest Instagram posts containing videos for further processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get row(s) in sheet1 (Google Sheets)  
- Loop Over Items (SplitInBatches)  
- Get an instagram user s posts1 (Scrape Creators API)  
- Edit Fields (Set)  
- Sticky Note (Documentation)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow execution on user demand  
  - Inputs: None  
  - Outputs: Triggers "Get row(s) in sheet1"  
  - Failure modes: User must manually trigger, no automation trigger here.

- **Get row(s) in sheet1**  
  - Type: Google Sheets (Read)  
  - Role: Reads Instagram handles from the "profiles" sheet in a specific Google Spreadsheet  
  - Configuration: Reads sheet ID and document ID pointing to Instagram profiles  
  - Outputs: Array of profile rows passed to "Loop Over Items"  
  - Potential failures: Authentication errors with Google Sheets OAuth2, empty or invalid sheet data.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each Instagram profile row individually  
  - Inputs: Array of Instagram profiles  
  - Outputs: Single profile to "Get an instagram user s posts1" per batch  
  - Edge cases: Handles large profile sets without overwhelming APIs; batch size defaults to 1 (or as configured).

- **Get an instagram user s posts1**  
  - Type: Scrape Creators API node  
  - Role: Fetches the latest Instagram post(s) for the given handle, limited to 1 post  
  - Configuration: Uses Instagram handle from current item, Scrape Creators API credentials  
  - Outputs: Instagram post data including video URLs and metadata  
  - Potential failures: API rate limits, invalid Instagram handles, network errors, expired credentials.

- **Edit Fields**  
  - Type: Set node  
  - Role: Extracts video URLs from nested Instagram post data to simplify downstream processing  
  - Configuration: Assigns `items[0].video_versions[0].url` and `items[0].video_versions[2].url` to top-level JSON fields for easier access  
  - Inputs: Instagram post JSON  
  - Outputs: JSON with extracted video URLs  
  - Edge cases: Missing video_versions fields, posts without videos.

- **Sticky Note (Step 1)**  
  - Provides high-level documentation for this block.

---

#### 2.2 Video Processing & AI Transcription

**Overview:**  
This block downloads the Instagram video file and transcribes its audio content to text using OpenAI's Whisper model, enabling content analysis and repurposing.

**Nodes Involved:**  
- Download Video (HTTP Request)  
- Transcribe Video (OpenAI Whisper)  
- Sticky Note1

**Node Details:**  

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads the video file from the Instagram CDN URL extracted earlier  
  - Configuration: URL set dynamically from `items[0].video_versions[0].url`  
  - Outputs: Binary video data for transcription  
  - Failure modes: Download failures (network, URL invalid), file size limits, timeouts  
  - On error: set to "continueRegularOutput" to prevent workflow halt on failure.

- **Transcribe Video**  
  - Type: OpenAI (LangChain)  
  - Role: Uses OpenAI Whisper to transcribe audio from downloaded video  
  - Configuration: Resource set to "audio", operation "transcribe", using OpenAI API credentials  
  - Outputs: JSON transcript text  
  - Failure modes: API quota exceeded, audio format issues, transcription errors  
  - On error: set to "continueRegularOutput" for resilience.

- **Sticky Note1**  
  - Documents the purpose and components of this block.

---

#### 2.3 AI Content Analysis & Enhancement

**Overview:**  
This block processes the transcript with GPT-4O to identify mentions of AI/automation tools, generates usage instructions, enriches the content with Perplexity AI research, and finally crafts an optimized Instagram script.

**Nodes Involved:**  
- Filter & Generate Suggestions (OpenAI GPT-4O)  
- Search Perplexity (HTTP Request)  
- Write New Script (OpenAI GPT-4O)  
- Sticky Note2

**Node Details:**  

- **Filter & Generate Suggestions**  
  - Type: OpenAI GPT-4O (LangChain)  
  - Role: Analyzes transcript to determine if it concerns AI tools, extracts tool names, generates step-by-step usage guides and content suggestions  
  - Configuration: System prompt frames the assistant as an admin helper, input transcript passed dynamically  
  - Outputs: JSON with verdict, tools list, step-by-step instructions, suggestions, and search prompt  
  - Failure modes: API failures, malformed transcript inputs, invalid JSON output  
  - On error: "continueRegularOutput" to avoid halting.

- **Search Perplexity**  
  - Type: HTTP Request  
  - Role: Queries Perplexity AI API to retrieve three interesting facts about identified tools using their search prompt  
  - Configuration: POST request with Bearer token authorization, JSON body using `searchPrompt` from previous node  
  - Outputs: JSON content with research results for script enrichment  
  - Failure modes: API key missing or invalid, request throttling, malformed responses  
  - On error: "continueRegularOutput".

- **Write New Script**  
  - Type: OpenAI GPT-4O (LangChain)  
  - Role: Synthesizes a polished Instagram video script combining tool info, step-by-step guides, Perplexity findings, and editorial suggestions  
  - Configuration: Uses temperature 0.7, system prompt defines tone and formatting, input merges multiple prior outputs  
  - Outputs: JSON with final script (~100 words) including a call to action  
  - Failure modes: API errors, JSON formatting issues  
  - On error: "continueErrorOutput" to capture errors explicitly.

- **Sticky Note2**  
  - Explains this block’s AI-driven analysis and script generation purpose.

---

#### 2.4 Data Storage & Loop Management

**Overview:**  
Final block updates the Google Sheet with comprehensive metadata, original and rewritten scripts, and loops back to process the next Instagram profile in the batch.

**Nodes Involved:**  
- Update Entries2 (Google Sheets)  
- Loop Over Items (SplitInBatches, for continuation)  
- Sticky Note3

**Node Details:**  

- **Update Entries2**  
  - Type: Google Sheets (Append/Update)  
  - Role: Writes processed video metadata, original transcript, and rewritten script to a "phantom output" sheet  
  - Configuration: Maps fields like id, caption, video URL, timestamp, likes, comments, view counts, original and rewritten scripts, user handle  
  - Inputs: Data aggregated from multiple prior nodes, including GPT outputs and Instagram post metadata  
  - Outputs: Signals next batch iteration via Loop Over Items  
  - Failure modes: Google Sheets API errors, schema mismatches, authentication problems  
  - On error: "continueRegularOutput" to ensure workflow continuity.

- **Loop Over Items**  
  - Receives batch continuation signal from Update Entries2 to process next profile until all profiles complete.

- **Sticky Note3**  
  - Details this block’s role in data persistence and workflow looping.

---

### 3. Summary Table

| Node Name                   | Node Type                       | Functional Role                                   | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                           |
|-----------------------------|--------------------------------|-------------------------------------------------|-------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                | Initiates workflow on demand                      | None                          | Get row(s) in sheet1             | STEP 1: Data Input & Video Retrieval                                                                |
| Get row(s) in sheet1         | Google Sheets (Read)           | Reads Instagram profiles from Google Sheet       | When clicking ‘Execute workflow’ | Loop Over Items                  | STEP 1: Data Input & Video Retrieval                                                                |
| Loop Over Items              | SplitInBatches                 | Processes each Instagram profile one by one      | Get row(s) in sheet1           | Get an instagram user s posts1, Loop continuation | STEP 1: Data Input & Video Retrieval; STEP 4: Data Storage & Loop Management                       |
| Get an instagram user s posts1 | Scrape Creators API            | Fetches latest Instagram video post for profile  | Loop Over Items                | Edit Fields                     | STEP 1: Data Input & Video Retrieval                                                                |
| Edit Fields                 | Set                            | Extracts video URLs from Instagram post data     | Get an instagram user s posts1 | Download Video                  | STEP 1: Data Input & Video Retrieval                                                                |
| Download Video              | HTTP Request                   | Downloads Instagram video file                    | Edit Fields                   | Transcribe Video                | STEP 2: Video Processing & AI Transcription                                                         |
| Transcribe Video            | OpenAI Whisper (LangChain)    | Transcribes video audio to text                    | Download Video                | Filter & Generate Suggestions   | STEP 2: Video Processing & AI Transcription                                                         |
| Filter & Generate Suggestions | OpenAI GPT-4O (LangChain)      | Analyzes transcript for AI tools, generates guides | Transcribe Video             | Search Perplexity               | STEP 3: AI Content Analysis & Enhancement                                                           |
| Search Perplexity           | HTTP Request                   | Retrieves facts about tools from Perplexity API  | Filter & Generate Suggestions | Write New Script               | STEP 3: AI Content Analysis & Enhancement                                                           |
| Write New Script            | OpenAI GPT-4O (LangChain)     | Creates polished Instagram video script           | Search Perplexity             | Update Entries2                | STEP 3: AI Content Analysis & Enhancement                                                           |
| Update Entries2             | Google Sheets (Append/Update) | Saves metadata, transcripts, and scripts          | Write New Script, Get an instagram user s posts1, Get row(s) in sheet1 | Loop Over Items (for next batch) | STEP 4: Data Storage & Loop Management                                                              |
| Sticky Note                 | Sticky Note                   | Documentation                                     | None                          | None                          | STEP 1: Data Input & Video Retrieval                                                                |
| Sticky Note1                | Sticky Note                   | Documentation                                     | None                          | None                          | STEP 2: Video Processing & AI Transcription                                                         |
| Sticky Note2                | Sticky Note                   | Documentation                                     | None                          | None                          | STEP 3: AI Content Analysis & Enhancement                                                           |
| Sticky Note3                | Sticky Note                   | Documentation                                     | None                          | None                          | STEP 4: Data Storage & Loop Management                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named *When clicking ‘Execute workflow’* to start execution manually.

2. **Add a Google Sheets node** configured as *Get row(s)*:  
   - Connect input from the Manual Trigger.  
   - Set credentials for Google Sheets OAuth2.  
   - Set Document ID to `"1tH-0MOe-Y_BsB3coqtFhhmZVHmchD0jTH8A3w1he3o8"`.  
   - Set Sheet Name to `"profiles"` (Sheet ID: 1977263029).  
   - Purpose: Read Instagram handles from sheet.

3. **Add a SplitInBatches node** named *Loop Over Items*:  
   - Connect input from Google Sheets node.  
   - Purpose: Process each Instagram profile individually.

4. **Add Scrape Creators API node** named *Get an instagram user s posts1*:  
   - Connect input from *Loop Over Items* node’s first output.  
   - Configure operation to `getInstagramUserPosts` with limit 1.  
   - Set `handle` parameter dynamically: `={{ $json['Instagram Handles'] }}`.  
   - Set credentials for Scrape Creators API.  
   - Purpose: Get the latest Instagram posts for each handle.

5. **Add a Set node** named *Edit Fields*:  
   - Connect input from Scrape Creators node.  
   - Assign values:  
     - `items[0].video_versions[0].url` = `={{ $json.items[0].video_versions[0].url }}`  
     - `items[0].video_versions[2].url` = `={{ $json.items[0].video_versions[2].url }}`  
   - Purpose: Extract video URLs for ease of access.

6. **Add an HTTP Request node** named *Download Video*:  
   - Connect input from *Edit Fields*.  
   - Set method to `GET`.  
   - Set URL dynamically to `={{ $json.items[0].video_versions[0].url }}`.  
   - Purpose: Download the video file.

7. **Add OpenAI node** named *Transcribe Video*:  
   - Connect input from *Download Video*.  
   - Set resource to `audio`, operation to `transcribe`.  
   - Use OpenAI credentials configured with valid API key.  
   - Purpose: Transcribe video audio to text using Whisper.

8. **Add OpenAI GPT-4O node** named *Filter & Generate Suggestions*:  
   - Connect input from *Transcribe Video*.  
   - Configure messages to include system prompt describing task (detect AI tools, generate usage steps, suggestions).  
   - Pass transcript text dynamically as input.  
   - Enable JSON output parsing.  
   - Use same OpenAI credentials.  
   - Purpose: Analyze transcripts to extract relevant AI tool info.

9. **Add HTTP Request node** named *Search Perplexity*:  
   - Connect input from *Filter & Generate Suggestions*.  
   - Set method to POST, URL to `https://api.perplexity.ai/chat/completions`.  
   - Add headers `accept: application/json` and `Authorization: Bearer API KEY` (replace with actual key).  
   - JSON body includes model `sonar-pro`, system prompt `"Be precise and concise."`, user prompt: `"Tell me three interesting (peculiar) things about {{ $json.message.content.searchPrompt }}"`.  
   - Purpose: Research additional info about detected tools.

10. **Add OpenAI GPT-4O node** named *Write New Script*:  
    - Connect input from *Search Perplexity*.  
    - Configure system prompt to write a casual, straightforward script with a call to action.  
    - Input merges tool names, rough draft script (transcript), Perplexity output, step-by-step guide, and improvement suggestions.  
    - Use temperature 0.7 for creativity balance.  
    - Purpose: Generate final polished Instagram video script.

11. **Add Google Sheets node** named *Update Entries2*:  
    - Connect input from *Write New Script*.  
    - Configure operation to `appendOrUpdate`.  
    - Set document and sheet ID to `"1tH-0MOe-Y_BsB3coqtFhhmZVHmchD0jTH8A3w1he3o8"` and `phantom output` sheet (ID: 375581874).  
    - Map fields with expressions pulling from Instagram post metadata and AI-generated content, including: post id, caption, video duration, video URL, username, timestamp, likes, comments, original transcript, video views, and rewritten script.  
    - Purpose: Save enriched data and scripts for editorial use.

12. **Connect *Update Entries2* output back to *Loop Over Items* second output**:  
    - Enables processing of the next Instagram profile until all are done.

13. **Add Sticky Notes** at appropriate places for documentation and clarity.

14. **Credential Setup:**  
    - Google Sheets OAuth2 Credential with read/write access.  
    - Scrape Creators API Credential with valid API key.  
    - OpenAI API Credential with GPT-4O and Whisper access.  
    - Perplexity API key configured in HTTP Request node header.

15. **Error Handling:**  
    - Set critical nodes (Download Video, Transcribe Video, Filter & Generate Suggestions, Search Perplexity) to continue on error to avoid workflow halt.  
    - Monitor logs for failures and retry if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                    |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Workflow automates repurposing Instagram video content for an AI & automation Instagram channel using OpenAI and Perplexity APIs. | Workflow purpose                                                                                                  |
| Scrape Creators API used to fetch Instagram posts programmatically.                                                  | https://scrapecreators.com/docs/                                                                                  |
| OpenAI GPT-4O model used for advanced content analysis and script generation; Whisper model used for transcription. | https://platform.openai.com/docs/models/gpt-4o and https://platform.openai.com/docs/guides/speech-to-text         |
| Perplexity AI API used to enrich content with external research facts.                                               | https://www.perplexity.ai/api                                                                                      |
| Google Sheets used as both input source and output repository for Instagram handles and processed data.               | https://developers.google.com/sheets/api                                                                            |
| Error handling configured to continue workflow on non-critical failures to maximize data processing completeness.   | Best practice for resilience in automation workflows                                                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.