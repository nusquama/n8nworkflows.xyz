Clone Viral TikToks with AI Avatars & Auto-Post to 9 Platforms using Perplexity & Blotato

https://n8nworkflows.xyz/workflows/clone-viral-tiktoks-with-ai-avatars---auto-post-to-9-platforms-using-perplexity---blotato-4110


# Clone Viral TikToks with AI Avatars & Auto-Post to 9 Platforms using Perplexity & Blotato

### 1. Workflow Overview

This workflow automates cloning a viral TikTok video using AI-generated avatars and posts the new content automatically to nine social media platforms via Blotato. It targets content creators, social media managers, and marketers aiming to scale short-form video content efficiently by replicating viral formats with personalized branding and distributing them broadly without manual effort.

The workflow is logically divided into four main blocks:

- **1.1 Clone Viral TikTok Video**: Receives a TikTok URL via Telegram, downloads video assets, transcribes audio, extracts overlays, and saves original metadata.
- **1.2 Suggest New Content Idea**: Uses Perplexity AI to generate a fresh but related content concept, then rewrites the script, caption, and overlay text with GPT-4o.
- **1.3 Create New Video with Avatar**: Fetches available AI avatars, generates a new video using the rewritten script, adds subtitles and overlay text, and retrieves the final rendered video.
- **1.4 Publish to 9 Platforms**: Assigns social media account IDs, uploads the final video to Blotato, and auto-publishes posts to Instagram, YouTube, TikTok, Facebook, Threads, X (Twitter), LinkedIn, Pinterest, and Bluesky; also sends preview links back to Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Clone Viral TikTok Video

**Overview:**  
This block triggers on a Telegram message containing a TikTok URL. It downloads the TikTok video and audio, extracts the thumbnail, uploads the thumbnail to Cloudinary, transcribes audio to text, performs image analysis on the thumbnail to extract overlay text, generates a unique template ID, and saves all original video metadata to Google Sheets.

**Nodes Involved:**  
- Trigger: Get TikTok URL via Telegram  
- Download TikTok Video (RapidAPI)  
- Extract Video Thumbnail  
- Upload Thumbnail to Cloudinary  
- Analyze Thumbnail (GPT-4o Vision)  
- Extract Overlay Text (GPT-4o)  
- Download TikTok Audio  
- Transcribe Audio to Script (GPT)  
- Generate Unique Template ID  
- Save Original Video to Google Sheets  

**Node Details:**

- **Trigger: Get TikTok URL via Telegram**  
  - Type: Telegram Trigger  
  - Role: Entry point receiving TikTok URL from user message  
  - Config: Listens to "message" update type; linked to Telegram Bot credentials  
  - Input: Telegram message with TikTok link  
  - Output: JSON containing message text (URL) and chat details  
  - Failures: Telegram API issues, missing URL in message  

- **Download TikTok Video (RapidAPI)**  
  - Type: HTTP Request  
  - Role: Downloads TikTok video data via RapidAPI service  
  - Config: GET request to TikTok download API with URL parameter extracted from Telegram message  
  - Headers: Requires RapidAPI host and API key  
  - Input: TikTok URL  
  - Output: Video content JSON including video URL, thumbnail URL, audio link  
  - Failures: Invalid URL, API rate limits, network errors  

- **Extract Video Thumbnail**  
  - Type: HTTP Request  
  - Role: Downloads video thumbnail image from URL provided by previous node  
  - Input: Thumbnail URL extracted from TikTok video data  
  - Output: Binary image data  
  - Failures: URL invalid or unavailable, network errors  

- **Upload Thumbnail to Cloudinary**  
  - Type: HTTP Request  
  - Role: Uploads thumbnail image to Cloudinary for hosted access  
  - Config: POST multipart/form-data with image file and upload preset  
  - Authentication: HTTP Basic Auth with Cloudinary credentials  
  - Output: Cloudinary URL for the uploaded thumbnail  
  - Failures: Authentication errors, upload limits, network issues  

- **Analyze Thumbnail (GPT-4o Vision)**  
  - Type: OpenAI GPT-4o Vision node  
  - Role: Performs image content analysis on thumbnail URL  
  - Input: Cloudinary image URL  
  - Output: JSON with content description of the thumbnail image  
  - Failures: OpenAI API quota, network errors  

- **Extract Overlay Text (GPT-4o)**  
  - Type: OpenAI GPT-4o text node  
  - Role: Parses analyzed thumbnail description to extract primary overlay text at the top of the image  
  - Input: Content from Analyze Thumbnail node  
  - Output: Text string of overlay text  
  - Failures: Text extraction errors, API limits  

- **Download TikTok Audio**  
  - Type: HTTP Request  
  - Role: Downloads TikTok audio file from URL in video data  
  - Input: Audio URL from TikTok video JSON  
  - Output: Audio binary or stream data  
  - Failures: Audio URL invalid, network errors  

- **Transcribe Audio to Script (GPT)**  
  - Type: OpenAI audio transcription node  
  - Role: Transcribes TikTok audio into text script  
  - Input: Audio data from Download TikTok Audio  
  - Output: Text transcription of audio  
  - Failures: Audio format issues, API transcription limits  

- **Generate Unique Template ID**  
  - Type: Code (JavaScript)  
  - Role: Creates unique alphanumeric ID for original video template  
  - Output: JSON with generated ID string  
  - Failures: None expected  

- **Save Original Video to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends original video metadata including caption, script, overlay text, template ID, and TikTok URL to a sheet for tracking  
  - Input: Mapped from previous nodes, including transcription and RapidAPI data  
  - Output: Confirmation of append operation  
  - Failures: Google Sheets API quota, permission errors  

---

#### 1.2 Suggest New Content Idea

**Overview:**  
This block uses Perplexity AI to suggest a new content idea based on the original video script, then rewrites the script, caption, and overlay text following the new idea with GPT-4o, preserving the original style but refreshing the content.

**Nodes Involved:**  
- Suggest Similar Idea (Perplexity)  
- Clean Perplexity Response (Code)  
- Rewrite Script, Caption, Overlay (GPT-4o)  
- Split Rewritten Content into Sections (Code)  
- Generate New Video ID (Code)  
- Save Rewritten Video to Google Sheets  

**Node Details:**

- **Suggest Similar Idea (Perplexity)**  
  - Type: HTTP Request  
  - Role: Sends original script to Perplexity's chat completions API, requesting a fresh but related content idea in the same niche  
  - Input: Original video script text  
  - Output: Raw suggestion response from Perplexity  
  - Failures: API key errors, rate limits, malformed requests  

- **Clean Perplexity Response**  
  - Type: Code (JavaScript)  
  - Role: Cleans raw Perplexity response by removing any <think> blocks and trimming whitespace  
  - Output: Cleaned text string with new content idea  
  - Failures: Parsing errors (unlikely)  

- **Rewrite Script, Caption, Overlay (GPT-4o)**  
  - Type: OpenAI GPT-4o  
  - Role: Rewrites the original video script, caption, and overlay text to match the new content idea, strictly preserving the original format, narration style, and length constraints (max 700 chars for script)  
  - Input: Cleaned content idea, original script, caption, and overlay text  
  - Output: Single text block containing rewritten overlay, script, and caption formatted as plain text sections  
  - Failures: API rate limits, formatting issues if special characters are included  

- **Split Rewritten Content into Sections**  
  - Type: Code (JavaScript)  
  - Role: Parses the rewritten text block into three separate JSON fields: text overlay, video script, and caption text  
  - Output: Structured JSON with separate fields  
  - Failures: Pattern matching errors if input format deviates  

- **Generate New Video ID**  
  - Type: Code (JavaScript)  
  - Role: Generates a unique alphanumeric ID for the new rewritten video version  
  - Output: JSON with new ID string  

- **Save Rewritten Video to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends the rewritten video metadata (new script, caption, overlay, IDs) to a dedicated sheet for tracking  
  - Input: Parsed rewritten content and generated ID  
  - Failures: Google Sheets API issues  

---

#### 1.3 Create New Video with Avatar

**Overview:**  
This block fetches available AI avatars, generates a new video with the chosen avatar using the rewritten script, waits for rendering completion, adds subtitles and overlay text using JSON2Video, and fetches the final rendered video URL.

**Nodes Involved:**  
- Fetch Available Avatars  
- Generate Video with Avatar  
- Wait for Avatar Rendering (3 min)  
- Fetch Avatar Video URL  
- Add Overlay Text with JSON2Video  
- Wait for Caption Rendering (2 min)  
- Fetch Final Video from JSON2Video  
- Update Final Video URL in Sheet  

**Node Details:**

- **Fetch Available Avatars**  
  - Type: HTTP Request  
  - Role: Queries Captions.ai API to retrieve available avatar creators  
  - Authentication: Captions.ai API key in header  
  - Output: JSON list of supported avatar names  
  - Failures: API key errors, network issues  

- **Generate Video with Avatar**  
  - Type: HTTP Request  
  - Role: Submits video generation request to Captions.ai with rewritten script, selected avatar name (first from list), and resolution 'fhd'  
  - Input: Script text from Save Rewritten Video node, avatar name from Fetch Avatars  
  - Output: Operation ID for rendering job  
  - Failures: API errors, invalid avatar name, script length errors  

- **Wait for Avatar Rendering (3 min)**  
  - Type: Wait node  
  - Role: Pauses workflow for 3 minutes to allow video rendering  
  - Failures: Timeouts if rendering takes longer, no direct failure  

- **Fetch Avatar Video URL**  
  - Type: HTTP Request  
  - Role: Polls Captions.ai API for rendered video URL using operation ID  
  - Output: JSON with video URL  
  - Failures: API errors, rendering failures, timeouts  

- **Add Overlay Text with JSON2Video**  
  - Type: HTTP Request  
  - Role: Calls JSON2Video API to add subtitles and overlay text to the avatar video  
  - Input: Video URL from Fetch Avatar Video URL, overlay text from Google Sheets  
  - Authentication: Custom HTTP authentication (presumably API key)  
  - Config: Defines video resolution, font styles, subtitle styles, overlay text position and formatting  
  - Output: JSON with new project ID for final video rendering  
  - Failures: API errors, invalid video URL, authentication errors  

- **Wait for Caption Rendering (2 min)**  
  - Type: Wait node  
  - Role: Pauses workflow for 2 minutes for JSON2Video rendering completion  
  - Failures: Timeout if rendering slower than expected  

- **Fetch Final Video from JSON2Video**  
  - Type: HTTP Request  
  - Role: Retrieves final video URL from JSON2Video using project ID  
  - Output: JSON containing final video URL  
  - Failures: API errors, project ID invalid  

- **Update Final Video URL in Sheet**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet row with the final rendered video URL linked to the video ID  
  - Operation: Append or update based on video ID matching  
  - Failures: Sheet access errors, update conflicts  

---

#### 1.4 Publish to 9 Platforms

**Overview:**  
This block assigns social media platform IDs, uploads the final video URL to Blotato media endpoint, then creates posts on Instagram, YouTube, TikTok, Facebook, Threads, Twitter (X), LinkedIn, Pinterest, and Bluesky using Blotato API. It also sends the video URL and preview back to the original Telegram chat.

**Nodes Involved:**  
- Assign Social Media IDs  
- Upload Video to Blotato  
- INSTAGRAM  
- YOUTUBE  
- TIKTOK  
- FACEBOOK  
- THREADS  
- TWETTER (Twitter)  
- LINKEDIN  
- BLUESKY  
- PINTEREST  
- Send Video URL via Telegram  
- Send Final Video Preview  

**Node Details:**

- **Assign Social Media IDs**  
  - Type: Set node  
  - Role: Defines account IDs and page/board IDs for all 9 platforms required by Blotato  
  - Config: Hardcoded placeholders (default "0000" or page/board IDs) to be replaced by user  
  - Output: JSON with platform IDs  
  - Failures: Misconfiguration leads to posting failures  

- **Upload Video to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads final video URL to Blotato media endpoint to obtain media reference for posts  
  - Headers: Requires Blotato API key  
  - Input: Video URL from Google Sheets updated final URL  
  - Output: Confirmation or media ID (implicit)  
  - Failures: API key errors, URL invalid, network issues  

- **INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWETTER, LINKEDIN, BLUESKY, PINTEREST**  
  - Type: HTTP Request (one per platform)  
  - Role: Each node creates a post on the respective platform via Blotato API using assigned account IDs, caption text, and media URL  
  - Config: JSON bodies tailored with platform-specific fields (e.g., YouTube privacy status, Facebook page ID, Pinterest board ID)  
  - Headers: Blotato API key required  
  - Output: Response from Blotato confirming post creation  
  - Failures: Invalid account IDs, API key errors, platform-specific policy violations  

- **Send Video URL via Telegram**  
  - Type: Telegram Node (Send Message)  
  - Role: Sends message with final video URL back to Telegram chat where TikTok URL was received  
  - Input: URL from Google Sheets update  
  - Failures: Telegram API errors, chat ID invalid  

- **Send Final Video Preview**  
  - Type: Telegram Node (Send Video)  
  - Role: Sends the final video as a video file to Telegram chat for preview  
  - Input: Video URL from Google Sheets update  
  - Failures: File size limits, Telegram API errors  

---

### 3. Summary Table

| Node Name                          | Node Type                     | Functional Role                                      | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                 |
|-----------------------------------|-------------------------------|-----------------------------------------------------|--------------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------|
| Trigger: Get TikTok URL via Telegram | Telegram Trigger              | Receives TikTok URL via Telegram message            |                                      | Download TikTok Video (RapidAPI)      | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Download TikTok Video (RapidAPI)  | HTTP Request                  | Downloads TikTok video data                          | Trigger: Get TikTok URL via Telegram | Extract Video Thumbnail               | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Extract Video Thumbnail           | HTTP Request                  | Downloads thumbnail image                            | Download TikTok Video (RapidAPI)      | Upload Thumbnail to Cloudinary        | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Upload Thumbnail to Cloudinary    | HTTP Request                  | Uploads thumbnail to Cloudinary                      | Extract Video Thumbnail               | Analyze Thumbnail (GPT-4o Vision)     | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Analyze Thumbnail (GPT-4o Vision) | OpenAI GPT-4o Vision          | Analyzes thumbnail image content                     | Upload Thumbnail to Cloudinary        | Extract Overlay Text (GPT-4o)         | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Extract Overlay Text (GPT-4o)     | OpenAI GPT-4o                 | Extracts top overlay text from thumbnail analysis   | Analyze Thumbnail (GPT-4o Vision)     | Download TikTok Audio                 | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Download TikTok Audio             | HTTP Request                  | Downloads TikTok audio                               | Extract Overlay Text (GPT-4o)         | Transcribe Audio to Script (GPT)      | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Transcribe Audio to Script (GPT) | OpenAI GPT                   | Transcribes audio to text script                      | Download TikTok Audio                 | Generate Unique Template ID           | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Generate Unique Template ID       | Code                         | Generates unique ID for original template            | Transcribe Audio to Script (GPT)      | Save Original Video to Google Sheets  | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Save Original Video to Google Sheets | Google Sheets                | Saves original video metadata                         | Generate Unique Template ID           | Suggest Similar Idea (Perplexity)     | 🟫 STEP 1 — Clone a viral TikTok video                                                      |
| Suggest Similar Idea (Perplexity) | HTTP Request                  | Suggests new content idea in same niche               | Save Original Video to Google Sheets  | Clean Perplexity Response             | 🟦 STEP 2 — Suggest new content idea                                                       |
| Clean Perplexity Response         | Code                         | Cleans and formats Perplexity response                | Suggest Similar Idea (Perplexity)     | Rewrite Script, Caption, Overlay (GPT-4o) | 🟦 STEP 2 — Suggest new content idea                                                       |
| Rewrite Script, Caption, Overlay (GPT-4o) | OpenAI GPT-4o               | Rewrites script, caption, overlay for new content     | Clean Perplexity Response             | Split Rewritten Content into Sections | 🟦 STEP 2 — Suggest new content idea                                                       |
| Split Rewritten Content into Sections | Code                         | Splits rewritten content into overlay, script, caption | Rewrite Script, Caption, Overlay (GPT-4o) | Generate New Video ID               | 🟦 STEP 2 — Suggest new content idea                                                       |
| Generate New Video ID             | Code                         | Generates unique ID for new video                      | Split Rewritten Content into Sections | Save Rewritten Video to Google Sheets | 🟦 STEP 2 — Suggest new content idea                                                       |
| Save Rewritten Video to Google Sheets | Google Sheets                | Saves rewritten video metadata                         | Generate New Video ID                 | Fetch Available Avatars               | 🟦 STEP 2 — Suggest new content idea                                                       |
| Fetch Available Avatars           | HTTP Request                  | Retrieves available AI avatars                         | Save Rewritten Video to Google Sheets | Generate Video with Avatar            | 🟪 STEP 3 — Create the new video with your avatar                                         |
| Generate Video with Avatar        | HTTP Request                  | Submits video generation request with avatar          | Fetch Available Avatars               | Wait for Avatar Rendering (3 min)     | 🟪 STEP 3 — Create the new video with your avatar                                         |
| Wait for Avatar Rendering (3 min) | Wait                         | Waits 3 minutes for avatar video rendering             | Generate Video with Avatar            | Fetch Avatar Video URL                | 🟪 STEP 3 — Create the new video with your avatar                                         |
| Fetch Avatar Video URL            | HTTP Request                  | Retrieves rendered avatar video URL                    | Wait for Avatar Rendering (3 min)     | Add Overlay Text with JSON2Video     | 🟪 STEP 3 — Create the new video with your avatar                                         |
| Add Overlay Text with JSON2Video | HTTP Request                  | Adds subtitles and overlay text to video               | Fetch Avatar Video URL                | Wait for Caption Rendering            | 🟪 STEP 3 — Create the new video with your avatar                                         |
| Wait for Caption Rendering        | Wait                         | Waits 2 minutes for JSON2Video rendering                | Add Overlay Text with JSON2Video     | Fetch Final Video from JSON2Video     | 🟪 STEP 3 — Create the new video with your avatar                                         |
| Fetch Final Video from JSON2Video | HTTP Request                  | Retrieves final video URL from JSON2Video               | Wait for Caption Rendering            | Update Final Video URL in Sheet        | 🟪 STEP 3 — Create the new video with your avatar                                         |
| Update Final Video URL in Sheet   | Google Sheets                | Updates metadata sheet with final video URL             | Fetch Final Video from JSON2Video     | Send Video URL via Telegram           | 🟪 STEP 3 — Create the new video with your avatar                                         |
| Send Video URL via Telegram       | Telegram                     | Sends final video URL message back to Telegram chat    | Update Final Video URL in Sheet       | Send Final Video Preview              | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| Send Final Video Preview          | Telegram                     | Sends final video as video file to Telegram chat       | Send Video URL via Telegram           | Assign Social Media IDs               | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| Assign Social Media IDs           | Set                          | Sets social media platform account IDs                  | Send Final Video Preview              | Upload Video to Blotato               | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| Upload Video to Blotato           | HTTP Request                  | Uploads video URL to Blotato media endpoint             | Assign Social Media IDs               | INSTAGRAM, YOUTUBE, TIKTOK, ...      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| INSTAGRAM                        | HTTP Request                  | Posts video to Instagram via Blotato                     | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| YOUTUBE                         | HTTP Request                  | Posts video to YouTube via Blotato                       | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| TIKTOK                          | HTTP Request                  | Posts video to TikTok via Blotato                        | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| FACEBOOK                       | HTTP Request                  | Posts video to Facebook via Blotato                      | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| THREADS                        | HTTP Request                  | Posts video to Threads via Blotato                       | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| TWETTER (Twitter)              | HTTP Request                  | Posts video to Twitter (X) via Blotato                   | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| LINKEDIN                      | HTTP Request                  | Posts video to LinkedIn via Blotato                      | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| BLUESKY                       | HTTP Request                  | Posts video to Bluesky via Blotato                       | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |
| PINTEREST                    | HTTP Request                  | Posts video to Pinterest via Blotato                     | Upload Video to Blotato               |                                      | 🟥 STEP 4 — Publish to 9 platforms                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Config: Listen to "message" updates  
   - Credential: Connect your Telegram Bot API credentials

2. **Download TikTok Video (RapidAPI)**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://tiktok-download-video1.p.rapidapi.com/getVideo?url={{ $json.message.text }}`  
   - Headers:  
     - x-rapidapi-host: tiktok-download-video1.p.rapidapi.com  
     - x-rapidapi-key: YOUR_RAPIDAPI_KEY  
   - Connect from Telegram Trigger

3. **Extract Video Thumbnail**  
   - Type: HTTP Request  
   - URL: `{{$json.data.origin_cover}}` (from previous node)  
   - Connect from Download TikTok Video

4. **Upload Thumbnail to Cloudinary**  
   - Type: HTTP Request (POST, multipart/form-data)  
   - URL: `https://api.cloudinary.com/v1_1/YOUR_CLOUDINARY_ID/image/upload`  
   - Body: file (binary from previous node), upload_preset = "n8n_clone"  
   - Auth: HTTP Basic Auth with Cloudinary credentials  
   - Connect from Extract Video Thumbnail

5. **Analyze Thumbnail (GPT-4o Vision)**  
   - Type: OpenAI GPT-4o Vision node  
   - Model: gpt-4o  
   - Resource: Image  
   - Image URLs: `{{$json.url}}` (Cloudinary URL)  
   - Auth: OpenAI credentials  
   - Connect from Upload Thumbnail to Cloudinary

6. **Extract Overlay Text (GPT-4o)**  
   - Type: OpenAI text node (gpt-4o)  
   - Prompt: Extract primary top overlay text from image description  
   - Input: Output content from Analyze Thumbnail  
   - Auth: OpenAI credentials  
   - Connect from Analyze Thumbnail

7. **Download TikTok Audio**  
   - Type: HTTP Request  
   - URL: `{{$('Download TikTok Video (RapidAPI)').item.json.data.music}}`  
   - Connect from Extract Overlay Text

8. **Transcribe Audio to Script (GPT)**  
   - Type: OpenAI audio transcription node  
   - Input: Audio data from previous node  
   - Auth: OpenAI credentials  
   - Connect from Download TikTok Audio

9. **Generate Unique Template ID**  
   - Type: Code node  
   - JS: Generates random alphanumeric 12-char ID  
   - Connect from Transcribe Audio

10. **Save Original Video to Google Sheets**  
    - Type: Google Sheets append node  
    - Map columns: Caption, Template ID, Video URL, Script, Overlay Text  
    - Auth: Google Sheets OAuth2  
    - Connect from Generate Unique Template ID

11. **Suggest Similar Idea (Perplexity)**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.perplexity.ai/chat/completions`  
    - Body: JSON with model sonar-reasoning and prompt using original script from Google Sheets  
    - Headers: Authorization: Bearer YOUR_PERPLEXITY_KEY, Content-Type: application/json  
    - Connect from Save Original Video

12. **Clean Perplexity Response**  
    - Type: Code node  
    - JS: Remove <think> tags and trim  
    - Connect from Suggest Similar Idea

13. **Rewrite Script, Caption, Overlay (GPT-4o)**  
    - Type: OpenAI GPT-4o text node  
    - Prompt: Rewrite script, caption, overlay using cleaned Perplexity idea and original content, respecting strict format and length limits  
    - Auth: OpenAI credentials  
    - Connect from Clean Perplexity Response

14. **Split Rewritten Content into Sections**  
    - Type: Code node  
    - JS: Parse rewritten text into overlay, script, caption JSON fields  
    - Connect from Rewrite Script node

15. **Generate New Video ID**  
    - Type: Code node  
    - JS: Generate new unique alphanumeric ID  
    - Connect from Split Rewritten Content

16. **Save Rewritten Video to Google Sheets**  
    - Type: Google Sheets append node  
    - Map new video ID, script, caption, overlay, subject  
    - Auth: Google Sheets OAuth2  
    - Connect from Generate New Video ID

17. **Fetch Available Avatars (Captions.ai)**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.captions.ai/api/creator/list`  
    - Headers: Content-Type: application/json, x-api-key: YOUR_CAPTIONS_KEY  
    - Connect from Save Rewritten Video

18. **Generate Video with Avatar**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.captions.ai/api/creator/submit`  
    - Body: JSON with script, creatorName (first avatar), resolution 'fhd'  
    - Headers: Content-Type: application/json, x-api-key: YOUR_CAPTIONS_KEY  
    - Connect from Fetch Available Avatars

19. **Wait for Avatar Rendering (3 min)**  
    - Type: Wait node, 3 minutes  
    - Connect from Generate Video with Avatar

20. **Fetch Avatar Video URL**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.captions.ai/api/creator/poll`  
    - Body: JSON with operationId from previous node  
    - Headers: Content-Type: application/json, x-api-key: YOUR_CAPTIONS_KEY  
    - Connect from Wait for Avatar Rendering

21. **Add Overlay Text with JSON2Video**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.json2video.com/v2/movies`  
    - Body: JSON configuration including video URL, overlay text, subtitles, styling  
    - Auth: Custom API key via HTTP Custom Auth  
    - Connect from Fetch Avatar Video URL

22. **Wait for Caption Rendering (2 min)**  
    - Type: Wait node, 2 minutes  
    - Connect from Add Overlay Text with JSON2Video

23. **Fetch Final Video from JSON2Video**  
    - Type: HTTP Request (GET)  
    - URL: `https://api.json2video.com/v2/movies?id={{ $json.project }}`  
    - Auth: Custom API key  
    - Connect from Wait for Caption Rendering

24. **Update Final Video URL in Google Sheets**  
    - Type: Google Sheets appendOrUpdate node  
    - Match on video ID, update URL column with final video URL  
    - Auth: Google Sheets OAuth2  
    - Connect from Fetch Final Video

25. **Send Video URL via Telegram**  
    - Type: Telegram send message node  
    - Message: "Url VIDEO : {{ $json['URL de la vidéo'] }}"  
    - Chat ID: From initial Telegram trigger message  
    - Connect from Update Final Video URL

26. **Send Final Video Preview**  
    - Type: Telegram send video node  
    - File: Final video URL  
    - Chat ID: From Telegram trigger  
    - Connect from Send Video URL

27. **Assign Social Media IDs**  
    - Type: Set node  
    - Set platform account IDs and page/board IDs as per user configuration  
    - Connect from Send Final Video Preview

28. **Upload Video to Blotato**  
    - Type: HTTP Request (POST)  
    - URL: `https://backend.blotato.com/v2/media`  
    - Body: JSON with video URL from Google Sheets  
    - Headers: blotato-api-key: YOUR_BLOTATO_KEY  
    - Connect from Assign Social Media IDs

29. **Publish Posts to Platforms (9 HTTP Requests)**  
    - Nodes: INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWITTER, LINKEDIN, BLUESKY, PINTEREST  
    - Type: HTTP Request (POST)  
    - URL: `https://backend.blotato.com/v2/posts`  
    - Body: Platform-specific JSON with accountId, targetType, caption, mediaUrls  
    - Headers: blotato-api-key: YOUR_BLOTATO_KEY  
    - Connect all from Upload Video to Blotato

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is ideal for content creators and marketers to scale TikTok-style videos fast.     | Workflow overview and use cases                                                                |
| Documentation with detailed setup and customization instructions available in Notion guide.      | [Notion Guide](https://automatisation.notion.site/WORKFLOW-n8n-1f13d6550fd980a7ab0dce650796ebaa?pvs=4) |
| Contact for consulting and support via LinkedIn and YouTube channels.                            | [LinkedIn](https://www.linkedin.com/in/dr-firas/), [YouTube](https://www.youtube.com/@DRFIRASS) |
| Replace all placeholder API keys with your actual credentials: Telegram, OpenAI, RapidAPI, etc.  | Important for workflow to function properly                                                    |
| Avatar customization can be done by changing the avatar ID and resolution in Captions.ai nodes. | Customizable video generation parameters                                                       |
| Google Sheets must have appropriate columns matching the workflow's data mapping.                | To track and store metadata correctly                                                         |
| Blotato platform supports multi-platform posting with a single API key; configure account IDs.  | Enables publishing to multiple social networks in one workflow                                 |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. All data processed are legal and public, complying strictly with content policies. No raw JSON is included except for necessary examples in the reproduction section.