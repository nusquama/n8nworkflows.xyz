Video Transcript Search and Q&A with VLM Run, GPT-4 & Google Workspace

https://n8nworkflows.xyz/workflows/video-transcript-search-and-q-a-with-vlm-run--gpt-4---google-workspace-9325


# Video Transcript Search and Q&A with VLM Run, GPT-4 & Google Workspace

### 1. Workflow Overview

This workflow automates the process of transcribing videos stored in a specific Google Drive folder and enables a question-and-answer interface based on the transcripts. It integrates Google Drive, VLM Run video transcription, Google Sheets for storage, and OpenAI’s GPT-4 for natural language Q&A.

Logical blocks:

- **1.1 Input Reception and File Download**  
  Watches a designated Google Drive folder for new video files and downloads them.

- **1.2 Video Transcription via VLM Run**  
  Sends downloaded videos asynchronously to VLM Run’s video transcription service and receives transcripts through a webhook callback.

- **1.3 Transcript Storage in Google Sheets**  
  Appends received transcripts to a Google Sheets document, creating a searchable knowledge base.

- **1.4 User Q&A Interface with AI Agent**  
  Provides a chat webhook that receives user questions, queries stored transcript data from Sheets, and generates answers strictly from those transcripts using GPT-4.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Download

**Overview:**  
This block monitors a specific Google Drive folder for newly created files (presumably videos), downloads the new files as binary data to be processed downstream.

**Nodes Involved:**  
- Google Drive Trigger  
- Download file

**Node Details:**

- **Google Drive Trigger**  
  - Type: Trigger node monitoring Google Drive  
  - Configurations: Watches a specific folder (ID: `1E8rvLEWKguorMT36yCD1jY78G0u8g6g7`) for new files created, polling every minute  
  - Input: None (trigger)  
  - Output: Emits new file metadata (including file ID)  
  - Credentials: Google Drive OAuth2  
  - Potential Failures: OAuth token expiration, API rate limits, folder permission issues  
  - Notes: Triggers on specific folder to limit scope and optimize polling

- **Download file**  
  - Type: Google Drive operation node  
  - Configurations: Downloads the file identified by the incoming file ID as binary data under property `data`  
  - Input: Receives file metadata from Google Drive Trigger  
  - Output: Binary file data for next processing node  
  - Credentials: Same Google Drive OAuth2 credentials  
  - Potential Failures: File access errors, download timeout, large file handling issues  
  - Notes: Must support common video formats compatible with VLM Run

---

#### 2.2 Video Transcription via VLM Run

**Overview:**  
This block uploads the video file to VLM Run’s transcription service, which processes the video asynchronously and calls back the workflow webhook when transcription finishes.

**Nodes Involved:**  
- VLM Run for Video Processing  
- Receives JSON Data (webhook)

**Node Details:**

- **VLM Run for Video Processing**  
  - Type: VLM Run node specialized for video transcription  
  - Configurations:  
    - Domain: `video.transcription`  
    - Operation: `video` (process video asynchronously)  
    - Async with callback URL: `https://playground.attensys.ai/webhook/transcript-video`  
  - Input: Receives binary video data from Download file node  
  - Output: Starts async job, no immediate output—results received via webhook  
  - Credentials: VLM Run API  
  - Potential Failures: API errors, invalid video format, callback delivery issues, network timeouts

- **Receives JSON Data (Webhook)**  
  - Type: HTTP webhook node  
  - Configurations: Listens for POST requests at path `/transcript-video`  
  - Input: Receives JSON payload containing transcription results from VLM Run callback  
  - Output: Passes transcript data downstream  
  - Potential Failures: Webhook unreachable, malformed payloads, security concerns (public webhook)  
  - Notes: Publicly accessible URL required for VLM Run callback

---

#### 2.3 Transcript Storage in Google Sheets

**Overview:**  
This block appends each completed transcription job’s data as a new row in a Google Sheet, maintaining a structured log of video transcripts for later querying.

**Nodes Involved:**  
- Append row in sheet

**Node Details:**

- **Append row in sheet**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Append  
    - Document ID: `1N8T3rF6DvDKQnRA93Hk-bvjhIu-nu89YveqEBLY9Lp4`  
    - Sheet Name: `gid=0` (Sheet1)  
    - Columns mapped:  
      - `Video Namae`: from `body[0].body.id` (video identifier)  
      - `Data`: from `body[0].body.response` (transcript text)  
  - Input: Receives transcription JSON from webhook node  
  - Output: Confirms row append success  
  - Credentials: Google Sheets OAuth2  
  - Potential Failures: API quota limits, malformed data, missing expected fields, transient network errors  
  - Notes: Validation recommended to ensure `body[0].body.response` exists before appending

---

#### 2.4 User Q&A Interface with AI Agent

**Overview:**  
This block provides an HTTP chat webhook that receives user questions, fetches relevant transcript rows from Google Sheets, and uses an AI agent with GPT-4 to answer strictly based on those rows.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- Get row(s) in sheet in Google Sheets  
- OpenAI Chat Model

**Node Details:**

- **When chat message received**  
  - Type: LangChain chat trigger (webhook)  
  - Configurations: Public webhook listening for chat messages  
  - Input: User question via webhook  
  - Output: Passes chat input to AI Agent  
  - Potential Failures: Webhook availability, malformed chat input

- **AI Agent**  
  - Type: LangChain agent node  
  - Configurations:  
    - System message defines role as a video-QA assistant  
    - Uses `Get rows` tool to fetch relevant transcript segments from Google Sheets  
    - Enforces that answers come strictly from the fetched sheets data  
  - Input: User question  
  - Output: Generated answer  
  - Potential Failures: Expression failures in system message, tool invocation errors, data retrieval issues

- **Get row(s) in sheet in Google Sheets**  
  - Type: Google Sheets tool node  
  - Configurations:  
    - Reads all rows from the same Google Sheet and Sheet1 (`gid=0`) as transcripts are stored  
  - Input: Invoked by AI Agent to fetch data  
  - Output: Provides rows to AI Agent for context  
  - Credentials: Google Sheets OAuth2  
  - Potential Failures: API quota, empty or inconsistent data, latency

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI chat model node  
  - Configurations: Uses GPT-4.1 model for generating responses  
  - Credentials: OpenAI API key  
  - Input: AI Agent language model input  
  - Output: Textual answer to user question  
  - Potential Failures: API rate limits, model unavailability, network issues

---

### 3. Summary Table

| Node Name                   | Node Type                               | Functional Role                         | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                             |
|-----------------------------|---------------------------------------|---------------------------------------|-----------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------|
| Google Drive Trigger         | n8n-nodes-base.googleDriveTrigger     | Watch Google Drive folder for new files | None                        | Download file                    | 📁 Input Processing: watches specified Drive folder every minute for new video files                   |
| Download file               | n8n-nodes-base.googleDrive             | Download new video file as binary     | Google Drive Trigger        | VLM Run for Video Processing      | 📁 Input Processing: downloads video file binary for VLM Run                                           |
| VLM Run for Video Processing | @vlm-run/n8n-nodes-vlmrun.vlmRun      | Upload video to VLM Run for transcription (async) | Download file               | None (async job)                  | 🤖 VLM Run Execute Agent: async transcription with callback URL                                        |
| Receives JSON Data           | n8n-nodes-base.webhook                 | Receive transcription results callback | None (external webhook)     | Append row in sheet               | 🤖 VLM Run Execute Agent: receives callback with transcript data                                      |
| Append row in sheet          | n8n-nodes-base.googleSheets            | Store transcript in Google Sheets     | Receives JSON Data          | None                            | 📊 Data Storage: append new row with Video Name and transcript Data                                    |
| When chat message received   | @n8n/n8n-nodes-langchain.chatTrigger  | Chat webhook for user questions       | None (external webhook)     | AI Agent                        | 💬 Chat Q&A Agent: receives user question for transcript Q&A                                          |
| AI Agent                    | @n8n/n8n-nodes-langchain.agent         | AI assistant: fetch sheets rows and answer | When chat message received  | OpenAI Chat Model, Get row(s) in sheet | 💬 Chat Q&A Agent: uses sheets data to answer questions strictly from transcripts                      |
| Get row(s) in sheet in Google Sheets | n8n-nodes-base.googleSheetsTool     | Fetch transcript rows from Sheets     | Invoked by AI Agent        | AI Agent                       | 💬 Chat Q&A Agent: data source for AI answers                                                        |
| OpenAI Chat Model            | @n8n/n8n-nodes-langchain.lmChatOpenAi | Generate natural language answer using GPT-4 | AI Agent (ai_languageModel) | AI Agent                      | 💬 Chat Q&A Agent: GPT-4 model generates answer based on transcript data                               |
| Sticky Note                 | n8n-nodes-base.stickyNote              | Documentation note                    | None                        | None                           | 🎥 Video Transcription + Q&A Pipeline overview                                                        |
| Sticky Note1                | n8n-nodes-base.stickyNote              | Documentation note                    | None                        | None                           | 📁 Input Processing explanation                                                                        |
| Sticky Note2                | n8n-nodes-base.stickyNote              | Documentation note                    | None                        | None                           | 🤖 VLM Run Execute Agent explanation                                                                   |
| Sticky Note3                | n8n-nodes-base.stickyNote              | Documentation note                    | None                        | None                           | 📊 Data Storage explanation                                                                             |
| Sticky Note4                | n8n-nodes-base.stickyNote              | Documentation note                    | None                        | None                           | 💬 Chat Q&A Agent explanation                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**
   - Type: Google Drive Trigger  
   - Settings:  
     - Event: `fileCreated`  
     - Folder to watch: Select folder by ID `1E8rvLEWKguorMT36yCD1jY78G0u8g6g7`  
     - Poll interval: every minute  
   - Credentials: Google Drive OAuth2 API  
   - Connect output to Download file node

2. **Create Download file node:**
   - Type: Google Drive node  
   - Operation: `download`  
   - File ID: use expression `{{$json["id"]}}` from trigger  
   - Binary property name: `data`  
   - Credentials: same Google Drive OAuth2  
   - Connect output to VLM Run node

3. **Create VLM Run for Video Processing node:**
   - Type: VLM Run node (`@vlm-run/n8n-nodes-vlmrun.vlmRun`)  
   - Domain: `video.transcription`  
   - Operation: `video`  
   - Enable asynchronous processing  
   - Callback URL: set to your publicly accessible webhook URL, e.g., `https://yourdomain.com/webhook/transcript-video`  
   - Credentials: VLM Run API credentials  
   - Connect input from Download file node

4. **Create HTTP Webhook node (Receives JSON Data):**
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `transcript-video`  
   - Public: enabled  
   - Connect output to Append row in sheet node

5. **Create Append row in sheet node:**
   - Type: Google Sheets node  
   - Operation: `append`  
   - Document ID: Google Sheet ID for transcript storage (`1N8T3rF6DvDKQnRA93Hk-bvjhIu-nu89YveqEBLY9Lp4`)  
   - Sheet Name: `gid=0` or `Sheet1`  
   - Columns mapping:  
     - `Video Namae` → `{{$json["body"][0]["body"]["id"]}}`  
     - `Data` → `{{$json["body"][0]["body"]["response"]}}`  
   - Credentials: Google Sheets OAuth2  
   - Connect output back-end to no further nodes (end of transcription flow)

6. **Create When chat message received node:**
   - Type: LangChain Chat Trigger  
   - Mode: Webhook, public enabled  
   - Connect output to AI Agent node

7. **Create AI Agent node:**
   - Type: LangChain Agent  
   - System message: set prompt to instruct agent as "video-QA assistant" who uses a Get rows tool to fetch relevant transcript segments and answer strictly based on those  
   - Connect `ai_languageModel` output to OpenAI Chat Model node  
   - Connect `ai_tool` output to Get row(s) in sheet node

8. **Create Get row(s) in sheet in Google Sheets node:**
   - Type: Google Sheets Tool node  
   - Operation: read rows from same Google Sheet and sheet as Append row node (`1N8T3rF6DvDKQnRA93Hk-bvjhIu-nu89YveqEBLY9Lp4`, `gid=0`)  
   - Credentials: Google Sheets OAuth2  
   - Connect output back to AI Agent node

9. **Create OpenAI Chat Model node:**
   - Type: LangChain OpenAI Chat Model  
   - Model: GPT-4.1 or latest GPT-4 model available  
   - Credentials: OpenAI API key  
   - Connect output back to AI Agent node (for final answer output)

10. **Credentials setup:**
    - Google Drive OAuth2: with access to watched folder  
    - Google Sheets OAuth2: with read/write access to target Sheet  
    - VLM Run API: valid API key for `video.transcription` domain  
    - OpenAI API: key with GPT-4 access

11. **Webhook public URL:**
    - Ensure your n8n instance or proxy is publicly reachable for both the chat webhook and the VLM Run callback webhook  
    - Use HTTPS URLs for security

12. **Validation and Testing:**
    - Upload a video file into the watched Drive folder and verify download and transcription  
    - Confirm webhook receives transcript and appends to Sheet  
    - Test chat endpoint with a question about a processed video and verify GPT-4 answers correctly based on sheet data

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                 | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow designed for team knowledge capture, course indexing, creator workflow automation, and support Q&A from video demos                                               | Overview Sticky Note                                                                                         |
| Ensure clear and consistent video naming to map transcripts accurately for Q&A                                                                                               | Chat Q&A Agent Sticky Note                                                                                   |
| Public webhook URL must be accessible by VLM Run and chat users                                                                                                              | Workflow webhook nodes configuration                                                                         |
| Consider adding timestamps in transcript data for improved retrieval and relevance                                                                                           | Chat Q&A Agent Sticky Note                                                                                   |
| Add retry logic and validation to handle transient errors and avoid data duplication                                                                                         | Chat Q&A Agent Sticky Note                                                                                   |
| VLM Run API documentation: https://vlm.run/docs                                                                                                                             | For domain and operation details                                                                              |
| Google API docs: https://developers.google.com/drive/api, https://developers.google.com/sheets/api                                                                           | For OAuth2 and API usage                                                                                       |
| OpenAI GPT-4 model info: https://platform.openai.com/docs/models/gpt-4                                                                                                       | For chat model capabilities and usage                                                                         |

---

**Disclaimer:** The provided text is based exclusively on an automated n8n workflow. It complies with all relevant content policies and handles only legal and public data.