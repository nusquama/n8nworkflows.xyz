Create Bilingual Newsletters with GPT-4o, AI Images & Videos for HubSpot & SharePoint

https://n8nworkflows.xyz/workflows/create-bilingual-newsletters-with-gpt-4o--ai-images---videos-for-hubspot---sharepoint-5988


# Create Bilingual Newsletters with GPT-4o, AI Images & Videos for HubSpot & SharePoint

### 1. Workflow Overview

This n8n workflow automates the creation and distribution of bilingual newsletters (German and English) featuring AI-generated content, images, and optional videos. It leverages GPT-4o-powered AI agents, Microsoft SharePoint for templating and storage, HubSpot for contact management, and FAL AI services for multimedia asset generation. The workflow is designed for marketing, growth, and automation teams aiming to efficiently produce polished, multilingual newsletters with rich media assets and flexible multi-channel distribution.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Intent Analysis:** Receives a user request via webhook and extracts user intent and parameters for video inclusion, distribution channels (HubSpot, SharePoint), and video duration.
- **1.2 AI Content Generation:** Uses GPT-4o to generate structured bilingual newsletter copy and AI prompts for multimedia asset creation.
- **1.3 Template Retrieval & Newsletter Assembly:** Fetches a SharePoint-hosted HTML newsletter template and injects AI-generated content into placeholders to produce the final newsletter HTML.
- **1.4 Multimedia Asset Generation:** Optionally generates images (via Pollinations AI), videos and audio (via FAL AI), merges video and audio assets, and handles processing waits and retries.
- **1.5 Approval Workflow:** Sends the draft newsletter and assets for approval via Gmail and waits for the approver's confirmation before proceeding.
- **1.6 Distribution & Storage:** Based on intent, sends the newsletter email via Outlook to HubSpot contacts and/or uploads the newsletter HTML, image, and video URL to SharePoint.
- **1.7 Utility & Conversion Nodes:** Handle base64 to binary conversions, file formatting, and orchestrate node merges and splits.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Intent Analysis

**Overview:**  
Receives the initial request containing newsletter instructions and user preferences. Extracts key parameters such as whether to include video, video duration, and which distribution channels to use (HubSpot, SharePoint).

**Nodes Involved:**  
- Receive Request (Webhook)  
- chatgpt-4o (OpenAI GPT-4o for intent extraction)  
- Determine Intent (Langchain AI Agent)  
- Parse Intent Fields (Code)  
- Configuration Settings (Set node)

**Node Details:**  

- **Receive Request**  
  - Type: Webhook  
  - Role: Entry point accepting HTTP POST requests with JSON body containing newsletter instructions.  
  - Config: Path set to a unique webhook path; rawBody enabled to accept raw JSON.  
  - Output: JSON body with user text under `.body.text`.  
  - Edge Cases: Invalid or missing JSON input; malformed requests.

- **chatgpt-4o**  
  - Type: Langchain OpenAI Chat model (GPT-4o)  
  - Role: Extract 4 specific intent parameters from user input: include_video (bool), video_duration (5 or 10 seconds), hubspot (bool), sharepoint (bool).  
  - Configuration: Prompt asks for JSON output with default values if fields are undetermined.  
  - Edge Cases: AI parsing errors, unexpected output format.

- **Determine Intent**  
  - Type: Langchain AI Agent  
  - Role: Processes the raw user text and outputs structured JSON with intent flags.  
  - Key Expression: Uses expression to pull text from Receive Request node.  
  - Version: v2 prompt type.

- **Parse Intent Fields**  
  - Type: Code  
  - Role: Cleans AI output (removes markdown), parses JSON, assigns fields (include_video, video_duration, hubspot, sharepoint) to JSON properties for downstream use.  
  - Edge Cases: JSON parse errors if AI output malformed.

- **Configuration Settings**  
  - Type: Set node  
  - Role: Holds environment variables including:  
    - ENV_FALAI_VIDEO_DURATION (video length default from intent)  
    - ENV_APPROVER_EMAIL (email for newsletter approval)  
    - ENV_INCLUDE_VIDEO (boolean flag from intent)  
    - ENV_FALAI_APIKEY (FAL AI API key placeholder)  
  - Edge Cases: Missing or invalid API keys or email addresses.

---

#### 2.2 AI Content Generation

**Overview:**  
Generates the bilingual newsletter copy JSON using GPT-4o, and separately generates prompts for image, video, and audio creation consistent with the newsletter theme.

**Nodes Involved:**  
- AI Agent (Langchain GPT-4o for newsletter content)  
- chatgpt-4o-2 (Langchain GPT-4o for multimedia prompts)  
- Generate Video and Audio Prompt (Langchain AI Agent)  
- Parse Fields (Code)

**Node Details:**  

- **AI Agent**  
  - Type: Langchain AI Agent (GPT-4o)  
  - Role: Generates structured bilingual newsletter JSON per strict schema for German and English content.  
  - Prompt: Detailed instructions with placeholders and tone/audience guidance, output in JSON format.  
  - Input: User text from Receive Request.  
  - Output: JSON string with keys like INTRODUCTION_DE, TOPIC_EN, TEASER1_EN, etc.  
  - Edge Cases: AI could produce malformed JSON or incomplete fields.

- **chatgpt-4o-2**  
  - Type: Langchain GPT-4o  
  - Role: Creates short prompts for video, audio, and image generation based on user input.  
  - Output: JSON with video_prompt, audio_prompt, image_prompt.

- **Generate Video and Audio Prompt**  
  - Role: Similar to chatgpt-4o-2, produces creative prompts for multimedia assets.  
  - Output: JSON with three prompt fields.

- **Parse Fields**  
  - Type: Code  
  - Role: Removes markdown wrappers, parses JSON, assigns prompts as separate JSON properties, and generates random 8-char filename for asset naming.  
  - Edge Cases: Parsing failures if AI output is malformed.

---

#### 2.3 Template Retrieval & Newsletter Assembly

**Overview:**  
Downloads the HTML newsletter template from SharePoint and injects AI-generated bilingual content into designated placeholders, producing the final newsletter HTML.

**Nodes Involved:**  
- Get Newsletter Template (SharePoint)  
- Get Template Content (Extract From File)  
- Build Newsletter (Code)

**Node Details:**  

- **Get Newsletter Template**  
  - Type: Microsoft SharePoint node  
  - Role: Downloads an HTML template file from a specified SharePoint site, folder, and file ID.  
  - Credentials: Microsoft SharePoint OAuth2.  
  - Edge Cases: Authentication issues, file not found, permission errors.

- **Get Template Content**  
  - Type: Extract From File  
  - Role: Extracts raw HTML text content from downloaded template file.  
  - Output: Template HTML string with placeholders.

- **Build Newsletter**  
  - Type: Code  
  - Role:  
    - Parses AI JSON output from AI Agent node.  
    - Replaces placeholders in template HTML with corresponding bilingual content fields.  
    - Adds current date formatted as German locale.  
    - Outputs final newsletter HTML content, topic names, generated date, and full AI content for debugging.  
  - Key expressions: Uses regex replacements for placeholders such as {{INTRODUCTION_DE}}, {{TOPIC_EN}}, etc.  
  - Edge Cases: Missing or undefined keys replaced with string "undefined"; malformed AI JSON; template missing required placeholders.

---

#### 2.4 Multimedia Asset Generation

**Overview:**  
Generates image, audio, and video assets using external AI services. The video and audio are merged into a final video file. Includes logic for processing delays and retries to handle asynchronous asset generation.

**Nodes Involved:**  
- Generate Image (Pollinations AI HTTP Request)  
- Set Base64 Field (Code)  
- Convert Base64 to File (ConvertToFile)  
- Send Data to Download Service (HTTP Request)  
- Switch (conditional on ENV_INCLUDE_VIDEO)  
- Create FAL Video (HTTP Request)  
- Wait For Video (Wait)  
- Get Video (HTTP Request)  
- Video Still Processing (If)  
- Create Audio (HTTP Request)  
- Wait For Audio (Wait)  
- Get Audio (HTTP Request)  
- Audio Still Processing (If)  
- Create Merge Request (HTTP Request)  
- Wait for Merge (Wait)  
- Get Merged Video (HTTP Request)

**Node Details:**  

- **Generate Image**  
  - Type: HTTP Request  
  - Role: Calls Pollinations AI image API with image_prompt; 1080x1350 resolution, model "flux", no logo.  
  - Retry: Up to 5 tries on failure.  
  - Edge Cases: API downtime, malformed prompt, network errors.

- **Set Base64 Field**  
  - Converts image binary data to base64 string for upload.

- **Convert Base64 to File**  
  - Converts base64 string back to binary file format.

- **Send Data to Download Service**  
  - Uploads image binary to a custom download service to receive a public download URL.  
  - TLS verification disabled (allowUnauthorizedCerts=true) to accept self-signed certs.  
  - Edge Cases: Upload failures, certificate issues.

- **Switch**  
  - Conditional branch based on ENV_INCLUDE_VIDEO:  
    - If true → proceeds to video generation branch.  
    - If false → skips video generation, proceeds to merge.

- **Create FAL Video**  
  - Posts to FAL AI video generation API with image URL, video prompt, and duration.  
  - Requires FAL AI API key from configuration.

- **Wait For Video**  
  - Waits 30 seconds before checking video generation status.

- **Get Video**  
  - Polls FAL AI video generation queue for completion and video URL.

- **Video Still Processing**  
  - If video not ready (error exists), loops back to wait again.

- **Create Audio**  
  - Posts to FAL AI Lyria2 audio generation API with audio prompt.

- **Wait For Audio**  
  - Waits 59 seconds before checking audio status.

- **Get Audio**  
  - Polls audio generation status.

- **Audio Still Processing**  
  - If audio not ready (error exists), loops back to wait again, else proceeds.

- **Create Merge Request**  
  - Posts to FAL AI FFmpeg API to merge audio and video URLs into final video.

- **Wait for Merge**  
  - Waits 50 seconds for merge completion.

- **Get Merged Video**  
  - Retrieves merged video URL.

- Edge Cases: API rate limits, timeouts, malformed URLs, missing API keys, network errors.

---

#### 2.5 Approval Workflow

**Overview:**  
Sends the draft bilingual newsletter with image and optional video preview to an approver via Gmail. Waits for explicit approval before continuing.

**Nodes Involved:**  
- Send message and wait for response (Gmail)  
- Newsletter Approved (If)

**Node Details:**  

- **Send message and wait for response**  
  - Type: Gmail node (sendAndWait)  
  - Role: Sends newsletter email to approver email from configuration with HTML body, embedded image, and video link if available.  
  - Approval type: double approval required.  
  - Credentials: Gmail OAuth2.  
  - Edge Cases: Email delivery failures, no response, timeout.

- **Newsletter Approved**  
  - Type: If  
  - Role: Checks if approval response contains "true" to continue execution.  
  - Edge Cases: Approval denied or no response stops workflow progression.

---

#### 2.6 Distribution & Storage

**Overview:**  
Distributes the finalized newsletter either by emailing it to HubSpot contacts or uploading assets to SharePoint, or both, based on user intent.

**Nodes Involved:**  
- WF Result (Switch)  
- HTML to Binary (Code)  
- HubSpot (HubSpot node)  
- Loop Over Items (SplitInBatches)  
- Send a message (Microsoft Outlook)  
- JPG to Binary (Code)  
- Upload HTML (SharePoint)  
- Upload JPG (SharePoint)  
- Save Video URL if exists (If)  
- TXT to Binary (Code)  
- Upload Video URL (SharePoint)

**Node Details:**  

- **WF Result**  
  - Switch node directing workflow based on boolean flags parsed from intent:  
    - Hubspot and SharePoint both true → upload HTML + send emails  
    - Only HubSpot true → send emails only  
    - Only SharePoint true → upload files only

- **HTML to Binary**  
  - Converts newsletter HTML content to binary buffer for upload.

- **HubSpot**  
  - Retrieves last 10 contacts from HubSpot using App Token authentication for email dispatch.

- **Loop Over Items**  
  - Splits contacts to send individual emails.

- **Send a message**  
  - Sends newsletter email via Outlook to each HubSpot contact with HTML content, embedded image URL, and optional video link.  
  - Credentials: Microsoft Outlook OAuth2.

- **JPG to Binary**  
  - Prepares binary image data for SharePoint upload.

- **Upload HTML / JPG / Video URL**  
  - Uploads newsletter HTML, image JPG, and video URL (text file) to SharePoint document library.  
  - Credentials: Microsoft SharePoint OAuth2.

- **Save Video URL if exists**  
  - Checks if video inclusion flag is true before uploading video URL.

- Edge Cases: API rate limits, authentication failures, file size limits, missing contacts.

---

#### 2.7 Utility & Conversion Nodes

**Overview:**  
Handle conversions between data formats and orchestrate merges for parallel data flows.

**Nodes Involved:**  
- Set Base64 Field (Code)  
- Convert Base64 to File (ConvertToFile)  
- Merge (Merge node)  
- TXT to Binary (Code)

**Node Details:**  

- **Set Base64 Field**  
  - Converts image binary data to base64 string to prepare for file conversion.

- **Convert Base64 to File**  
  - Converts base64 string into binary file for HTTP upload.

- **Merge**  
  - Combines outputs from multiple nodes by position to aggregate data for final newsletter assembly.

- **TXT to Binary**  
  - Converts video URL text into binary format for SharePoint upload.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                                  | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                                                                        |
|----------------------------|--------------------------------------|-------------------------------------------------|-----------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Receive Request            | Webhook                              | Entry point; receives user newsletter request   |                             | chatgpt-4o                      |                                                                                                                                                    |
| chatgpt-4o                | Langchain GPT-4o Agent               | Extracts intent flags from user input           | Receive Request             | Determine Intent                |                                                                                                                                                    |
| Determine Intent           | Langchain AI Agent                   | Processes user query into structured JSON intent| chatgpt-4o                  | Parse Intent Fields             |                                                                                                                                                    |
| Parse Intent Fields        | Code                                | Cleans and parses intent JSON                    | Determine Intent            | Configuration Settings          |                                                                                                                                                    |
| Configuration Settings     | Set                                 | Holds environment variables & flags             | Parse Intent Fields         | AI Agent, Generate Video and Audio Prompt |                                                                                                                                                    |
| AI Agent                  | Langchain GPT-4o Agent               | Generates bilingual newsletter content JSON     | Configuration Settings      | Get Newsletter Template         |                                                                                                                                                    |
| Get Newsletter Template    | Microsoft SharePoint                 | Downloads HTML newsletter template               | AI Agent                   | Get Template Content            |                                                                                                                                                    |
| Get Template Content       | Extract From File                   | Extracts raw HTML from downloaded template      | Get Newsletter Template     | Build Newsletter               |                                                                                                                                                    |
| Build Newsletter           | Code                                | Injects AI content into template placeholders   | Get Template Content, AI Agent, Merge | Merge                        |                                                                                                                                                    |
| chatgpt-4o-2              | Langchain GPT-4o                    | Generates prompts for image, video, audio       | Configuration Settings      | Generate Video and Audio Prompt |                                                                                                                                                    |
| Generate Video and Audio Prompt | Langchain AI Agent             | Creates short creative prompts for assets       | chatgpt-4o-2                | Parse Fields                   |                                                                                                                                                    |
| Parse Fields               | Code                                | Cleans & parses multimedia prompt JSON          | Generate Video and Audio Prompt | Generate Image, Set Base64 Field |                                                                                                                                                    |
| Generate Image             | HTTP Request (Pollinations AI)      | Generates 1080x1350 image from prompt            | Parse Fields                | Set Base64 Field               |                                                                                                                                                    |
| Set Base64 Field           | Code                                | Converts image binary to base64 string           | Generate Image              | Convert Base64 to File         |                                                                                                                                                    |
| Convert Base64 to File     | ConvertToFile                       | Converts base64 string to binary file            | Set Base64 Field            | Send Data to Download Service, Merge |                                                                                                                                                    |
| Send Data to Download Service | HTTP Request                    | Uploads image binary to download service          | Convert Base64 to File      | Switch                        |                                                                                                                                                    |
| Switch                    | Switch                              | Branches workflow based on video inclusion flag | Send Data to Download Service | Create FAL Video / Merge       | Covered by Sticky Note1: Video Assets Branch                                                                                                      |
| Create FAL Video           | HTTP Request (FAL AI)               | Requests video creation from FAL AI              | Switch                     | Wait For Video                |                                                                                                                                                    |
| Wait For Video             | Wait                               | Waits 30 seconds for video generation             | Create FAL Video            | Get Video                    |                                                                                                                                                    |
| Get Video                  | HTTP Request (FAL AI)               | Polls video generation status                      | Wait For Video             | Video Still Processing        |                                                                                                                                                    |
| Video Still Processing     | If                                 | Checks if video is still processing                | Get Video                  | Wait For Video / Create Audio  |                                                                                                                                                    |
| Create Audio               | HTTP Request (FAL AI)               | Requests audio generation from FAL AI             | Video Still Processing      | Wait For Audio                |                                                                                                                                                    |
| Wait For Audio             | Wait                               | Waits 59 seconds for audio generation              | Create Audio               | Get Audio                    |                                                                                                                                                    |
| Get Audio                  | HTTP Request (FAL AI)               | Polls audio generation status                      | Wait For Audio             | Audio Still Processing        |                                                                                                                                                    |
| Audio Still Processing     | If                                 | Checks if audio is still processing                | Get Audio                  | Wait For Audio / Create Merge Request |                                                                                                                                                    |
| Create Merge Request       | HTTP Request (FAL AI)               | Requests merging of audio and video                | Audio Still Processing      | Wait for Merge               |                                                                                                                                                    |
| Wait for Merge             | Wait                               | Waits 50 seconds for merge completion              | Create Merge Request        | Get Merged Video             |                                                                                                                                                    |
| Get Merged Video           | HTTP Request (FAL AI)               | Retrieves merged video URL                          | Wait for Merge             | Merge                       |                                                                                                                                                    |
| Merge                     | Merge                              | Combines multiple inputs for final newsletter build| Build Newsletter, Convert Base64 to File, Get Merged Video | Build Newsletter           |                                                                                                                                                    |
| HTML to Binary             | Code                               | Converts final newsletter HTML to binary           | Build Newsletter           | Upload HTML, HubSpot          | Covered by Sticky Note2: SharePoint Storage Branch                                                                                                |
| JPG to Binary              | Code                               | Converts image to binary for upload                 | Upload HTML                | Upload JPG                   | Covered by Sticky Note2                                                                                                                            |
| TXT to Binary              | Code                               | Converts video URL text to binary                    | Save Video URL if exists    | Upload Video URL             | Covered by Sticky Note2                                                                                                                            |
| Save Video URL if exists   | If                                 | Checks if video should be saved                       | Upload JPG                 | TXT to Binary                | Covered by Sticky Note2                                                                                                                            |
| Upload HTML               | Microsoft SharePoint                | Uploads newsletter HTML file                          | HTML to Binary             | JPG to Binary                | Covered by Sticky Note2                                                                                                                            |
| Upload JPG                | Microsoft SharePoint                | Uploads image JPG file                                | JPG to Binary              | Save Video URL if exists     | Covered by Sticky Note2                                                                                                                            |
| Upload Video URL          | Microsoft SharePoint                | Uploads video URL as text file                        | TXT to Binary              |                              | Covered by Sticky Note2                                                                                                                            |
| HubSpot                   | HubSpot Node                      | Retrieves contacts to email newsletter                | HTML to Binary             | Loop Over Items              | Covered by Sticky Note3: HubSpot Distribution Branch                                                                                              |
| Loop Over Items           | Split In Batches                  | Iterates over HubSpot contacts to send emails        | HubSpot                    | Send a message               | Covered by Sticky Note3                                                                                                                            |
| Send a message            | Microsoft Outlook                 | Sends the bilingual newsletter email to contacts      | Loop Over Items            | Loop Over Items              | Covered by Sticky Note3                                                                                                                            |
| Send message and wait for response | Gmail                      | Sends newsletter for approval and waits for response | Merge                      | Newsletter Approved          |                                                                                                                                                    |
| Newsletter Approved       | If                                | Proceeds only if newsletter is approved               | Send message and wait for response | WF Result                   |                                                                                                                                                    |
| WF Result                 | Switch                            | Routes workflow to HubSpot, SharePoint, or both       | Newsletter Approved        | HubSpot, HTML to Binary      |                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Receive Request"**  
   - Type: Webhook (POST)  
   - Path: unique identifier (e.g., "b1e65fc7-fb83-4e9d-8568-26778d6be83a")  
   - Options: enable rawBody to accept raw JSON.

2. **Add Langchain Node "chatgpt-4o"**  
   - Model: GPT-4o latest  
   - Prompt: Extract 4 fields (include_video, video_duration, hubspot, sharepoint) from webhook text input in JSON format.  
   - Input: Expression `={{ $('Receive Request').first().json.body.text }}`  
   - Credentials: OpenAI API key.

3. **Add Langchain Node "Determine Intent"**  
   - Prompt: Parses user query into intent JSON object with defaults.  
   - Input: Use output from "chatgpt-4o".  
   - Credentials: OpenAI API key.

4. **Add Code Node "Parse Intent Fields"**  
   - JavaScript: Strip markdown, parse JSON, assign `.include_video`, `.video_duration`, `.hubspot`, `.sharepoint`.  
   - Input: output of "Determine Intent".

5. **Add Set Node "Configuration Settings"**  
   - Assign variables:  
     - ENV_FALAI_VIDEO_DURATION = `{{$json.video_duration}}`  
     - ENV_APPROVER_EMAIL = your approver email (e.g., "test@mail.de")  
     - ENV_INCLUDE_VIDEO = `{{$json.include_video}}`  
     - ENV_FALAI_APIKEY = your FAL AI API key (string)  

6. **Add Langchain Node "AI Agent"**  
   - Prompt: Generate bilingual newsletter JSON content with German and English placeholders, tone, and structure.  
   - Input: Expression `={{ $('Receive Request').first().json.body.text }}`  
   - Credentials: OpenAI API key.

7. **Add Microsoft SharePoint Node "Get Newsletter Template"**  
   - Operation: Download file  
   - Configuration: Set site, folder, and file ID to your HTML template location.  
   - Credentials: Microsoft SharePoint OAuth2.

8. **Add Extract From File Node "Get Template Content"**  
   - Operation: Extract text content from SharePoint file.  
   - Input: Output of "Get Newsletter Template".

9. **Add Code Node "Build Newsletter"**  
   - JavaScript:  
     - Parse AI Agent JSON output (remove markdown)  
     - Replace placeholders in template with AI content fields in German and English  
     - Add current date (`toLocaleDateString('de-DE')`)  
     - Return final newsletter HTML and metadata.  
   - Inputs: Output from "Get Template Content" and AI Agent.

10. **Add Langchain Node "chatgpt-4o-2"**  
    - Prompt: Generate short video, audio, and image prompts from user input.  
    - Input: `={{ $('Receive Request').first().json.body.text }}`  
    - Credentials: OpenAI API key.

11. **Add Langchain Node "Generate Video and Audio Prompt"**  
    - Same as above; outputs JSON with video_prompt, audio_prompt, image_prompt.

12. **Add Code Node "Parse Fields"**  
    - Parse JSON output from above node, assign prompts to `.video_prompt`, `.audio_prompt`, `.image_prompt`  
    - Generate `.random_filename` (8-char random string).

13. **Add HTTP Request Node "Generate Image"**  
    - URL: `https://image.pollinations.ai/prompt/{{ $json.image_prompt.replaceAll(' ','-').replaceAll(',','').replaceAll('.','').slice(0,100) }}?width=1080&height=1350&model=flux&nologo=true`  
    - Method: GET  
    - Retry: max 5 attempts on failure.

14. **Add Code Node "Set Base64 Field"**  
    - Converts image binary data to base64 string for upload.

15. **Add ConvertToFile Node "Convert Base64 to File"**  
    - Converts base64 string to binary file.

16. **Add HTTP Request Node "Send Data to Download Service"**  
    - URL: `https://operator.velvetsyndicate.de:8090/api/upload-image`  
    - Method: POST, multipart-form-data  
    - Attach binary image file  
    - Options: allowUnauthorizedCerts = true.

17. **Add Switch Node "Switch"**  
    - Condition: ENV_INCLUDE_VIDEO boolean flag  
    - True branch: proceed to video generation  
    - False branch: skip video generation and proceed to merge.

18. **Add HTTP Request Node "Create FAL Video"**  
    - URL: `https://queue.fal.run/fal-ai/kling-video/v2/master/image-to-video`  
    - Method: POST  
    - Body: image_url (from previous upload), prompt (video_prompt), duration (ENV_FALAI_VIDEO_DURATION)  
    - Headers: Authorization with FAL AI API key.

19. **Add Wait Node "Wait For Video"**  
    - Wait 30 seconds.

20. **Add HTTP Request Node "Get Video"**  
    - Poll video generation status by request_id.

21. **Add If Node "Video Still Processing"**  
    - Check for error in response; if still processing loop back to wait.

22. **Add HTTP Request Node "Create Audio"**  
    - Posts audio prompt to FAL AI Lyria2 API.

23. **Add Wait Node "Wait For Audio"**  
    - Wait 59 seconds.

24. **Add HTTP Request Node "Get Audio"**  
    - Polls audio generation status.

25. **Add If Node "Audio Still Processing"**  
    - Checks audio processing status; loops back if not ready.

26. **Add HTTP Request Node "Create Merge Request"**  
    - Posts video and audio URLs to FAL AI FFmpeg merge API.

27. **Add Wait Node "Wait for Merge"**  
    - Wait 50 seconds for merge completion.

28. **Add HTTP Request Node "Get Merged Video"**  
    - Retrieve merged video URL.

29. **Add Merge Node "Merge"**  
    - Combine outputs from Build Newsletter, converted image, and merged video for downstream use.

30. **Add Code Node "HTML to Binary"**  
    - Converts newsletter HTML to binary for upload.

31. **Add Microsoft SharePoint Nodes "Upload HTML", "Upload JPG", "Upload Video URL"**  
    - Upload newsletter HTML, image JPG, and video URL text file to SharePoint document library.  
    - Use binary data from previous conversions.  
    - Use Microsoft SharePoint OAuth2 credentials.

32. **Add Code Nodes "JPG to Binary" and "TXT to Binary"**  
    - Convert image and video URL to binary respectively for SharePoint upload.

33. **Add If Node "Save Video URL if exists"**  
    - Checks ENV_INCLUDE_VIDEO before proceeding with video URL upload.

34. **Add HubSpot Node "HubSpot"**  
    - Retrieves last 10 contacts using HubSpot App Token.

35. **Add SplitInBatches Node "Loop Over Items"**  
    - Loops over HubSpot contacts to send individual emails.

36. **Add Microsoft Outlook Node "Send a message"**  
    - Sends newsletter email with embedded image and optional video link to each contact.  
    - Uses Outlook OAuth2 credentials.

37. **Add Gmail Node "Send message and wait for response"**  
    - Sends newsletter draft for approval to configured email.  
    - Waits for approval before proceeding.

38. **Add If Node "Newsletter Approved"**  
    - Checks approval status; continues only if approved.

39. **Add Switch Node "WF Result"**  
    - Routes workflow based on user intent: HubSpot only, SharePoint only, or both.

40. **Add Sticky Notes**  
    - Add detailed notes with setup instructions, credentials needed, and customization tips at key points for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow is designed for marketing, growth, and automation teams to generate polished bilingual newsletters with rich media assets and multi-channel distribution without coding.                                                                                                                                                                                                                                                                                                                                | Sticky Note at workflow start                                                                                         |
| Requires credentials for OpenAI, HubSpot (App Token), Microsoft 365 (SharePoint & Outlook OAuth2), Gmail (OAuth2), and FAL AI (video & audio generation). Store credentials securely in n8n Credential Manager.                                                                                                                                                                                                                                                                                                         | Setup instructions in Sticky Note                                                                                     |
| Newsletter HTML template must contain exact placeholders as specified (e.g., {{INTRODUCTION_DE}}, {{TOPIC_EN}}) to ensure correct content injection. Template is stored and fetched from SharePoint.                                                                                                                                                                                                                                                                                                               | Sticky Note5: Newsletter Template Branch                                                                              |
| To customize language or content style, edit the AI Agent prompt accordingly. To change storage providers, replace SharePoint nodes. To skip video generation, set include_video to false in incoming requests. Modify distribution logic by adjusting the WF Result Switch node. Additional channels like Slack can be added after the Switch node and intent extraction.                                                                                                                                                 | Sticky Notes throughout workflow                                                                                       |
| Pollinations AI is used for image generation without requiring authentication. FAL AI handles video and audio generation, merging audio-video via FFmpeg API.                                                                                                                                                                                                                                                                                                                                                         | Sticky Note1: Video Assets Branch; Sticky Note4: Image Assets Branch                                                  |
| HubSpot node fetches up to 10 contacts by default; batch size and limit can be increased or adjusted in node parameters. Emails are sent individually via Microsoft Outlook node.                                                                                                                                                                                                                                                                                                                                | Sticky Note3: HubSpot Distribution Branch                                                                              |
| Approval process is implemented via Gmail node with double approval type; workflow halts if newsletter is not approved.                                                                                                                                                                                                                                                                                                                                                                                           |                                                                                                                                                              |
| Video generation is asynchronous and can take several minutes; workflow includes waits and polling with retry logic to handle processing times.                                                                                                                                                                                                                                                                                                                                                                   |                                                                                                                                                              |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a no-code automation platform. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.