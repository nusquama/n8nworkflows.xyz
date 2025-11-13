Transform Telegram Content to Multi-Platform Posts with OpenAI Whisper & Gemini

https://n8nworkflows.xyz/workflows/transform-telegram-content-to-multi-platform-posts-with-openai-whisper---gemini-9363


# Transform Telegram Content to Multi-Platform Posts with OpenAI Whisper & Gemini

### 1. Workflow Overview

This workflow automates the transformation of content received via Telegram into optimized posts for multiple social media platforms using AI models (OpenAI Whisper and Google Gemini). It supports various content types: voice messages, photos, videos, and text. The workflow processes incoming Telegram messages, determines their type, transcribes or analyzes media using AI, generates tailored descriptions and titles suitable for different platforms, sends previews back to the user for approval, and finally uploads approved content to social networks via the Upload-Post service.

**Logical Blocks:**

- **1.1 Input Reception and Message Type Detection:**  
  Listens for Telegram messages and classifies them by type (voice, photo, video, text, or command).

- **1.2 Media Retrieval:**  
  Downloads the relevant media files (voice, photo, video) from Telegram.

- **1.3 AI Processing:**  
  - Voice messages: Transcription using OpenAI Whisper.  
  - Photos and videos: Content analysis using Google Gemini.  
  - Text messages: Direct AI content generation.

- **1.4 Content Generation and Formatting:**  
  Uses AI agents configured for each content type to produce platform-specific post descriptions and titles.

- **1.5 User Confirmation:**  
  Sends generated content back to the user on Telegram, awaiting approval.

- **1.6 Upload to Social Platforms:**  
  Uploads approved posts (photos, videos, or text) to multiple social media platforms via Upload-Post API.

- **1.7 Notifications:**  
  Provides status updates and upload confirmations back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Message Type Detection

- **Overview:**  
  This block triggers the workflow upon receiving a Telegram message and determines the message type for routing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Edit Fields  
  - Send Regular Message  
  - Switch  
  - No Operation, do nothing

- **Node Details:**

1. **Telegram Trigger**  
   - Type: Telegram Trigger node  
   - Role: Listens for new Telegram messages (updates of type "message")  
   - Credentials: Telegram bot API credentials ("agentsocialmediaAIbot")  
   - Output: Emits Telegram message JSON  
   - Edge Cases: Possible webhook setup errors, missing permissions for bot, network failure

2. **Edit Fields**  
   - Type: Set node  
   - Role: Adds fixed fields for Upload-Post API user and Pinterest board ID  
   - Configuration: Sets `upload_post_user` to "influencersde" and `pinterest_board_id` to "995647498810363103"  
   - Input: Telegram message from trigger  
   - Output: Enriched JSON with these fields

3. **Send Regular Message**  
   - Type: Telegram node (send message)  
   - Role: Sends user notification "Asset received: Processing and generating descriptions"  
   - Input: From Edit Fields node  
   - Output: Confirmation of message sent to user  
   - Edge Cases: Telegram API limits, chat ID errors

4. **Switch**  
   - Type: Switch node  
   - Role: Routes workflow by message content type:  
     - "Do Nothing" for /start command  
     - "voice" if voice message exists  
     - "picture" if document exists (photo)  
     - "video" if video exists  
     - "text" for other text messages  
   - Inputs: Telegram message JSON  
   - Outputs: Multiple branches for next processing

5. **No Operation, do nothing**  
   - Type: NoOp  
   - Role: Ends workflow for /start command, no further processing

---

#### 2.2 Media Retrieval

- **Overview:**  
  Downloads media files from Telegram based on message type for further AI processing.

- **Nodes Involved:**  
  - Get a Audio File  
  - Get a Photo  
  - Get a Video File

- **Node Details:**

1. **Get a Audio File**  
   - Type: Telegram node (get file)  
   - Role: Downloads voice message file using `voice.file_id`  
   - Input: Telegram message with voice  
   - Credentials: Telegram API  
   - Output: Binary audio file  

2. **Get a Photo**  
   - Type: Telegram node (get file)  
   - Role: Downloads highest resolution photo or document file if photo missing  
   - Input: Telegram message with photo or document  
   - Output: Binary image file  

3. **Get a Video File**  
   - Type: Telegram node (get file)  
   - Role: Downloads video file using `video.file_id`  
   - Input: Telegram message with video  
   - Output: Binary video file  

- **Edge Cases:**  
  - File ID missing or expired  
  - Telegram API rate limits  
  - File size or format unsupported  

---

#### 2.3 AI Processing

- **Overview:**  
  Processes media or text using AI to extract or generate meaningful content.

- **Nodes Involved:**  
  - Transcribe a recording (OpenAI Whisper)  
  - Analyze an image (Google Gemini)  
  - Analyze video (Google Gemini)  

- **Node Details:**

1. **Transcribe a recording**  
   - Type: OpenAI node (Langchain)  
   - Role: Transcribes audio file to text using OpenAI Whisper  
   - Input: Audio binary from Get a Audio File  
   - Credentials: OpenAI API key  
   - Output: Transcribed text  
   - Edge Cases: Audio quality issues, API limits, transcription delays  

2. **Analyze an image**  
   - Type: Google Gemini node (Langchain)  
   - Role: Analyzes photo content and description  
   - Input: Binary image from Get a Photo  
   - Parameters: Includes message caption or text for context  
   - Credentials: Google Palm API  
   - Output: Image description text  
   - Edge Cases: API errors, image format issues  

3. **Analyze video**  
   - Type: Google Gemini node (Langchain)  
   - Role: Analyzes video content and describes it  
   - Input: Binary video from Get a Video File  
   - Credentials: Google Palm API  
   - Output: Video description text  
   - Edge Cases: Video format unsupported, API limits  

---

#### 2.4 Content Generation and Formatting

- **Overview:**  
  AI agents generate platform-optimized post descriptions and titles based on content analysis or transcription.

- **Nodes Involved:**  
  - AI Agent Photos  
  - AI Agent Videos  
  - AI Agent Text  
  - Google Gemini Chat Model (various instances)  
  - Structured Output Parser (various instances)  

- **Node Details:**

1. **AI Agent Photos**  
   - Type: Langchain Agent with Google Gemini Chat Model  
   - Role: Generates descriptions/titles for TikTok, Instagram, Pinterest from photo analysis  
   - Input: Image analysis text  
   - Output: Structured JSON with platform-specific descriptions  
   - System Prompt: Social media expert tone, fixed JSON output format  
   - Edge Cases: Parsing errors, AI model downtime  

2. **AI Agent Videos**  
   - Type: Langchain Agent with Google Gemini Chat Model  
   - Role: Generates descriptions/titles for TikTok, Instagram, YouTube from video analysis  
   - Input: Video analysis text  
   - Output: Structured JSON with titles and descriptions  
   - System Prompt: Social media expert tone, no special characters, fixed JSON output  
   - Edge Cases: Same as above  

3. **AI Agent Text**  
   - Type: Langchain Agent with Google Gemini Chat Model  
   - Role: Generates descriptions for Threads, LinkedIn, and X (Twitter) from text message  
   - Input: Text content from Telegram  
   - Output: Structured JSON with platform descriptions  
   - System Prompt: Social media expert, specific formatting, multiple languages  
   - Edge Cases: Text too long, formatting issues  

4. **Google Gemini Chat Model** nodes  
   - Serve as language model interface for AI Agents  
   - Connected with Structured Output Parser nodes for JSON validation and correction  

5. **Structured Output Parser** nodes  
   - Validate and autocorrect AI output JSON schemas  
   - Ensure consistent structured output for downstream processing  

---

#### 2.5 User Confirmation

- **Overview:**  
  Sends generated post content back to the Telegram user and waits for manual approval before upload.

- **Nodes Involved:**  
  - Send message and wait for response (multiple instances)  
  - If / If1 / If2  

- **Node Details:**

1. **Send message and wait for response** (Photos, Videos, Text variants)  
   - Type: Telegram node (sendAndWait)  
   - Role: Sends generated descriptions for platforms, awaits double approval from user  
   - Input: Output of AI Agents  
   - Output: User approval or rejection  
   - Edge Cases: User timeout, Telegram API errors  

2. **If / If1 / If2**  
   - Type: If node  
   - Role: Checks if user approved the content (`data.approved == true`) before proceeding to upload  
   - Input: Approval response  
   - Output: Proceed or stop workflow  

---

#### 2.6 Upload to Social Platforms

- **Overview:**  
  Uploads approved content to multiple social media platforms using Upload-Post API.

- **Nodes Involved:**  
  - Upload photos  
  - Upload video to Tiktok Instagram Youtube  
  - Upload a text post  

- **Node Details:**

1. **Upload photos**  
   - Type: Upload-Post node  
   - Role: Uploads photos with generated titles and descriptions to Instagram, Pinterest, TikTok  
   - Input: Photo files + AI descriptions  
   - Credentials: Upload-Post API ("Smoker")  
   - Edge Cases: API errors, platform limits, auth failures  

2. **Upload video to Tiktok Instagram Youtube**  
   - Type: Upload-Post node  
   - Role: Uploads video with generated titles and descriptions to TikTok, Instagram, YouTube  
   - Input: Video file + AI descriptions  
   - Credentials: Upload-Post API  
   - Edge Cases: Same as above  

3. **Upload a text post**  
   - Type: Upload-Post node  
   - Role: Uploads text posts to LinkedIn, X (Twitter), and Threads  
   - Input: Text content descriptions from AI Agent Text  
   - Credentials: Upload-Post API  
   - Edge Cases: API limits, format restrictions  

---

#### 2.7 Notifications

- **Overview:**  
  Sends status updates to user about uploading progress and final results.

- **Nodes Involved:**  
  - Send processing upload (multiple instances)  
  - Send Confirmation upload (multiple instances)  

- **Node Details:**

1. **Send processing upload**  
   - Type: Telegram node (send message)  
   - Role: Notify user that upload is in progress ("⌛ Uploading...")  
   - Input: Triggered before upload nodes  

2. **Send Confirmation upload**  
   - Type: Telegram node (send message)  
   - Role: Sends detailed upload success/error statuses and URLs back to user for each platform  
   - Input: Upload results from Upload-Post nodes  
   - Edge Cases: Telegram delivery failure, incomplete upload info  

---

### 3. Summary Table

| Node Name                   | Node Type                               | Functional Role                                              | Input Node(s)              | Output Node(s)                           | Sticky Note                                                                                                                |
|-----------------------------|---------------------------------------|--------------------------------------------------------------|----------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger                      | Receive Telegram messages to start workflow                  | -                          | Edit Fields                             | ### Waits for telegram message to start workflow                                                                           |
| Edit Fields                 | Set                                  | Add user profile and Pinterest board ID                      | Telegram Trigger           | Send Regular Message                    | ### Configure upload-post user profile created in app.upload-post.com                                                       |
| Send Regular Message        | Telegram                             | Notify user that processing started                          | Edit Fields                | Switch                                | ### Sent telegram notification starting process                                                                             |
| Switch                     | Switch                               | Determine message type and route                              | Send Regular Message       | No Operation, Get a Audio File, Get a Photo, Get a Video File | ### Determines what type of message was sent                                                                                  |
| No Operation, do nothing    | NoOp                                 | Handle /start command by doing nothing                       | Switch                    | -                                       |                                                                                                                            |
| Get a Audio File            | Telegram (Get file)                   | Download voice message audio file                             | Switch                    | Transcribe a recording                  | ### Get Audio File                                                                                                          |
| Transcribe a recording      | OpenAI (Langchain)                   | Transcribe audio to text                                      | Get a Audio File           | AI Agent Text                          | ### Use OpenAI to Transcribe Recording                                                                                     |
| Get a Photo                 | Telegram (Get file)                   | Download photo or document file                               | Switch                    | Analyze an image                       | ### Get Photo File                                                                                                          |
| Analyze an image            | Google Gemini (Langchain)            | Analyze photo content                                        | Get a Photo                | AI Agent Photos                       | ### Use Gemini to analyze photo                                                                                            |
| Get a Video File            | Telegram (Get file)                   | Download video file                                          | Switch                    | Analyze video                         | ### Get Video File                                                                                                          |
| Analyze video               | Google Gemini (Langchain)            | Analyze video content                                       | Get a Video File           | AI Agent Videos                      | ### Use Gemini to analyze Video                                                                                            |
| AI Agent Text               | Langchain Agent (Google Gemini)      | Generate social media text posts for Threads, LinkedIn, X   | Transcribe a recording     | Send message and wait for response2   | ### Base AI Agent. Adjust "System Prompt" to personalize this agent to your needs                                           |
| AI Agent Photos             | Langchain Agent (Google Gemini)      | Generate photo post descriptions for TikTok, Instagram, Pinterest | Analyze an image           | Send message and wait for response    | ### Base AI Agent. Adjust "System Prompt" to personalize this agent to your needs                                           |
| AI Agent Videos             | Langchain Agent (Google Gemini)      | Generate video post descriptions for TikTok, Instagram, YouTube | Analyze video              | Send message and wait for response1   | ### Base AI Agent. Adjust "System Prompt" to personalize this agent to your needs                                           |
| Send message and wait for response  | Telegram (sendAndWait)           | Send generated photo post descriptions, wait approval        | AI Agent Photos            | If                                   |                                                                                                                            |
| Send message and wait for response1 | Telegram (sendAndWait)           | Send generated video post descriptions, wait approval        | AI Agent Videos            | If1                                  |                                                                                                                            |
| Send message and wait for response2 | Telegram (sendAndWait)           | Send generated text post descriptions, wait approval         | AI Agent Text              | If2                                  |                                                                                                                            |
| If                         | If                                   | Check user approval for photo post                          | Send message and wait for response | Send processing upload1              |                                                                                                                            |
| If1                        | If                                   | Check user approval for video post                          | Send message and wait for response1 | Send processing upload               |                                                                                                                            |
| If2                        | If                                   | Check user approval for text post                           | Send message and wait for response2 | Send processing upload2              |                                                                                                                            |
| Send processing upload1     | Telegram                             | Notify user upload of photo post is starting                 | If                        | Get a Photo File                      |                                                                                                                            |
| Send processing upload      | Telegram                             | Notify user upload of video post is starting                 | If1                       | Get a Video File2                    |                                                                                                                            |
| Send processing upload2     | Telegram                             | Notify user upload of text post is starting                  | If2                       | Upload a text post                   |                                                                                                                            |
| Get a Photo File            | Telegram (Get file)                   | Download photo file for upload                               | Send processing upload1    | Upload photos                       |                                                                                                                            |
| Upload photos               | Upload-Post                         | Upload photos to Instagram, Pinterest, TikTok                | Get a Photo File           | Send Confirmation upload1             |                                                                                                                            |
| Get a Video File2           | Telegram (Get file)                   | Download video file for upload                               | Send processing upload     | Upload video to Tiktok Instagram Youtube |                                                                                                                            |
| Upload video to Tiktok Instagram Youtube | Upload-Post                   | Upload video to TikTok, Instagram, YouTube                   | Get a Video File2          | Send Confirmation upload              |                                                                                                                            |
| Upload a text post          | Upload-Post                         | Upload text posts to LinkedIn, X, Threads                    | Send processing upload2    | Send Confirmation upload2             |                                                                                                                            |
| Send Confirmation upload1   | Telegram                             | Notify user of photo post upload results                      | Upload photos              | -                                   |                                                                                                                            |
| Send Confirmation upload    | Telegram                             | Notify user of video post upload results                      | Upload video to Tiktok Instagram Youtube | -                                   |                                                                                                                            |
| Send Confirmation upload2   | Telegram                             | Notify user of text post upload results                       | Upload a text post         | -                                   |                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Set Updates to: "message"  
   - Credentials: Create Telegram API credentials by generating a bot token via @BotFather, then add in n8n  
   - Position: Start of workflow

2. **Add Set Node (Edit Fields):**  
   - Add fields:  
     - `upload_post_user` = "influencersde"  
     - `pinterest_board_id` = "995647498810363103"

3. **Add Telegram Node (Send Regular Message):**  
   - Message: "Asset received: Processing and generating descriptions"  
   - Chat ID: Use expression from Telegram Trigger `message.chat.id`  

4. **Add Switch Node:**  
   - Define outputs for message types:  
     - "Do Nothing": if message text equals "/start"  
     - "voice": if `message.voice` exists  
     - "picture": if `message.document` exists  
     - "video": if `message.video` exists  
     - "text": if message text exists and not "/start"

5. **Add No Operation Node:**  
   - Connect "Do Nothing" branch to this to end workflow for /start command

6. **Media Retrieval:**  
   - For "voice": Add Telegram Get file node, fileId from `message.voice.file_id`  
   - For "picture": Add Telegram Get file node, fileId from last photo or document `message.photo` or `message.document.file_id`  
   - For "video": Add Telegram Get file node, fileId from `message.video.file_id`  

7. **AI Processing:**  
   - For voice: Add OpenAI node (Langchain) with resource "audio" and operation "transcribe"  
   - For photo: Add Google Gemini node (Langchain) with resource "image" and operation "analyze", include caption or text from message  
   - For video: Add Google Gemini node (Langchain) with resource "video" and operation "analyze"  

8. **Content Generation AI Agents:**  
   - For photos: Langchain Agent node with Google Gemini Chat Model, system prompt to output JSON for TikTok, Instagram, Pinterest  
   - For videos: Langchain Agent node with Google Gemini Chat Model, system prompt to output JSON for TikTok, Instagram, YouTube  
   - For text: Langchain Agent node with Google Gemini Chat Model, system prompt to output JSON for Threads, LinkedIn, X  

9. **Structured Output Parsers:**  
   - Attach structured output parsers to each AI Agent node with JSON schema examples matching expected outputs

10. **User Confirmation:**  
    - For each content type, add Telegram Send Message node with "sendAndWait" operation to send generated descriptions and await approval  
    - Add If nodes to check if `data.approved == true` before proceeding  

11. **Upload Notifications:**  
    - Add Telegram Send Message nodes to notify "Uploading..." before upload nodes  

12. **Upload Nodes (Upload-Post):**  
    - For photos: Upload photos to Instagram, Pinterest, TikTok using user and board ID from Set node, set titles and descriptions from AI Agent output  
    - For videos: Upload video to TikTok, Instagram, YouTube with titles and descriptions from AI Agent output  
    - For text: Upload text posts to LinkedIn, X, Threads with descriptions from AI Agent output; set LinkedIn page ID as needed  

13. **Upload Confirmation Notifications:**  
    - Add Telegram Send Message nodes to send upload success/error information with URLs back to user  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Create your Telegram bot using @BotFather and paste the token in n8n credentials | Instructions in Sticky Note #1 |
| Configure Upload-Post user profile at https://app.upload-post.com to generate API token and credentials | Sticky Note #24 |
| Use any LLM supporting OpenAI or Google Gemini; adjust system prompts to personalize AI agents | Sticky Notes #5, #6, #15, #19 |
| Workflow designed for multi-platform social media posting: TikTok, Instagram, Pinterest, YouTube, LinkedIn, X (Twitter), Threads | Overall workflow purpose |
| Upload-Post API allows uploading photos, videos, and text posts with single API calls; supports multiple platforms simultaneously | Sticky Notes and Upload nodes |
| Telegram messages with /start command do not trigger further processing | Switch node logic |
| AI Agents use structured JSON outputs to ensure consistent formatting for social posts | Structured Output Parser nodes |
| User approval is required before uploading content to prevent unwanted posts | Telegram sendAndWait nodes |
| Potential error points include Telegram API limits, AI API quotas, file download failures, and Upload-Post authentication errors | General caution |
| Video example of workflow in action and template link: https://lnkd.in/dqSeZ7is | Provided in Sticky Note #8 |
| Upload-Post official documentation and integration API: https://upload-post.com | Sticky Note #8 and project references |

---

**Disclaimer:**  
The provided text is extracted solely from an automated workflow created with n8n, a tool for integration and automation. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.