Automate ASMR Glass Fruit Video Creation & Publishing with Veo, Shotstack & Postiz

https://n8nworkflows.xyz/workflows/automate-asmr-glass-fruit-video-creation---publishing-with-veo--shotstack---postiz-10175


# Automate ASMR Glass Fruit Video Creation & Publishing with Veo, Shotstack & Postiz

### 1. Workflow Overview

This workflow automates the end-to-end creation and publishing of ASMR-style glass fruit videos using Google Veo 3.0, Shotstack, Google Cloud Storage, and Postiz API. It is designed for content creators or marketing teams seeking to produce visually compelling, cinematic short-form videos (vertical 9:16 aspect ratio) without manual intervention.

The workflow is logically divided into five main blocks:

- **1.1 Automation & Data Input:** Fetches previously used fruit concepts from a Google Sheet to avoid repeats.
- **1.2 AI Generation (Idea + Prompt Creation):** Uses OpenAI GPT-4-based agents to generate a unique glass fruit idea and a detailed video prompt for Google Veo.
- **1.3 Video Generation with Google Veo:** Authenticates with Google Cloud, submits the AI prompt to Veo, polls for video completion, and retrieves the base64 video.
- **1.4 Video Conversion & Upload:** Converts the raw video output to a file, uploads it to Google Cloud Storage, then sends it to Shotstack for cropping to a vertical format, waiting for rendering completion.
- **1.5 Auto-Publishing to Social Media:** Uploads the final video to Postiz, fetches social media integrations, and schedules posts on TikTok, YouTube, and Instagram.

---

### 2. Block-by-Block Analysis

#### 2.1 Automation & Data Input

- **Overview:**  
  Reads historical fruit objects from a Google Sheet, aggregates the data, and prepares a list of used fruits to inform AI generation.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Past Objects (Google Sheets)  
  - Aggregate (Aggregate)  
  - Set Object List (Set)

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or on-demand execution.  
    - Inputs: None  
    - Outputs: Triggers "Get Past Objects"

  - **Get Past Objects**  
    - Type: Google Sheets  
    - Role: Reads from a specific Google Sheet (documentId: "1hNzWblAWsI_5u4RUK7kC3tSbR9riex94ZUuslhhDb8o", sheet gid=0) containing previously used fruit objects.  
    - Configuration: Reads entire sheet1 data.  
    - Inputs: Trigger from manual node.  
    - Outputs: Passes rows to "Aggregate."  
    - Edge cases: Possible authentication errors, empty sheet, or API rate limits.

  - **Aggregate**  
    - Type: Aggregate Node  
    - Role: Aggregates all sheet rows into one combined data object.  
    - Inputs: Data from "Get Past Objects."  
    - Outputs: Passes aggregated data to "Set Object List."

  - **Set Object List**  
    - Type: Set Node  
    - Role: Extracts an array of used fruit objects from the aggregated data for feeding into AI.  
    - Configuration: Creates an array `objects` with values from the first two entries of the aggregated data (assumes at least two rows).  
    - Inputs: Aggregated data from "Aggregate."  
    - Outputs: To "Idea Agent."  
    - Edge case: If fewer than two objects exist, expression may fail or produce incomplete list.

---

#### 2.2 AI Generation (Idea + Prompt Creation)

- **Overview:**  
  Generates a new unique glass fruit idea not previously used, then creates a detailed cinematic prompt for Google Veo video generation.

- **Nodes Involved:**  
  - Set Object List (from previous block)  
  - Idea Agent (LangChain Agent with OpenAI GPT-4.1-mini)  
  - Object & Caption (Structured Output Parser)  
  - Prompt Agent (LangChain Agent with GPT-4.1-mini)  
  - OpenAI Chat Model (shared LM for both agents)

- **Node Details:**  
  - **Idea Agent**  
    - Type: LangChain Agent Node  
    - Role: Receives the array of used fruits, generates one new fruit name not in the list, formatted as JSON with fields `object` and `caption`.  
    - Configuration: Uses system instructions to avoid repeats, ensure aesthetic and feasible fruit choice.  
    - Input: JSON array `objects` from "Set Object List."  
    - Output: JSON object for new fruit idea.  
    - Edge cases: Parsing errors if output is malformed; repetition if AI ignores constraints.

  - **Object & Caption**  
    - Type: LangChain Structured Output Parser  
    - Role: Validates and parses the JSON output from Idea Agent to ensure it matches expected schema (`object` and `caption` as strings).  
    - Input: Raw AI output from Idea Agent.  
    - Output: Clean JSON to feed downstream.  
    - Edge cases: Parsing failure if AI output deviates from schema.

  - **Prompt Agent**  
    - Type: LangChain Agent Node  
    - Role: Receives the new fruit name and generates a detailed, compliant text prompt for Google Veo 3.0, describing the ASMR-style video with strict policy adherence.  
    - Configuration: Long system message enforcing content safety, style, and cinematic quality, including ASMR sound layers and CGI realism.  
    - Input: New fruit object name from "Object & Caption."  
    - Output: Text prompt string for video generation.  
    - Edge cases: Token limits, prompt truncation, or generation failure.

  - **OpenAI Chat Model**  
    - Type: Language Model node (GPT-4.1-mini)  
    - Role: Shared LM for both AI agents to generate responses.  
    - Configuration: Uses GPT-4.1-mini model.  
    - Inputs/Outputs: Connected to both Idea Agent and Prompt Agent.  
    - Edge cases: API quota limits, network errors.

---

#### 2.3 Video Generation with Google Veo

- **Overview:**  
  Authenticates with Google Cloud using JWT, exchanges for OAuth token, submits the AI-generated prompt to Veo API, and polls for video generation completion.

- **Nodes Involved:**  
  - Prompt Agent (from previous block)  
  - SET (Set Google Cloud credentials and config)  
  - JWT (JWT generation node)  
  - GET TOKEN (HTTP Request to Google OAuth endpoint)  
  - Generate Video (HTTP Request to Veo API)  
  - Wait (Wait node for polling interval)  
  - Fetch Status (HTTP Request to check video generation status)  
  - Switch (Check if video is ready or still processing)

- **Node Details:**  
  - **SET**  
    - Type: Set Node  
    - Role: Holds Google Cloud credentials: `PROJECT_ID`, `CLIENT_EMAIL`, `LOCATION_ID` (default "us-central1"), and API endpoint.  
    - Inputs: Prompt text from Prompt Agent.  
    - Outputs: To JWT node.  
    - Edge cases: Empty credentials cause auth failure.

  - **JWT**  
    - Type: JWT Node  
    - Role: Creates JSON Web Token for Google OAuth2 token exchange with claims including issuer, scope, audience, issued-at, and expiration.  
    - Inputs: Credentials from "SET."  
    - Outputs: JWT token string to "GET TOKEN."  
    - Edge cases: Time skew causing invalid expiration, invalid claims.

  - **GET TOKEN**  
    - Type: HTTP Request  
    - Role: Exchanges JWT for OAuth access token via Google OAuth2 token endpoint.  
    - Inputs: JWT token from "JWT."  
    - Outputs: Access token JSON to "Generate Video."  
    - Edge cases: Auth errors, invalid JWT.

  - **Generate Video**  
    - Type: HTTP Request  
    - Role: Sends POST request to Veo model endpoint with prompt to initiate video generation (long-running operation).  
    - Inputs: OAuth token, prompt text.  
    - Outputs: Operation name for polling.  
    - Edge cases: API errors, invalid prompt, rate limits.

  - **Wait**  
    - Type: Wait Node  
    - Role: Delays workflow 20 seconds between status polls to avoid excessive API calls.  
    - Inputs: From "Generate Video."  
    - Outputs: To "Fetch Status."

  - **Fetch Status**  
    - Type: HTTP Request  
    - Role: Queries Veo API for operation status using operation name.  
    - Inputs: OAuth token, operation name.  
    - Outputs: Video generation status and base64 video data (if done).  
    - Edge cases: Network errors, operation not found.

  - **Switch**  
    - Type: Switch Node  
    - Role: Checks if video generation status is "done" or not.  
    - Outputs: If done → "Convert to File"; else → loops back to "Wait."  
    - Edge cases: Unexpected statuses, missing fields.

---

#### 2.4 Video Conversion & Upload

- **Overview:**  
  Converts base64 video to binary file, uploads to Google Cloud Storage, sends to Shotstack for vertical cropping, waits for rendering, and downloads final output.

- **Nodes Involved:**  
  - Convert to File  
  - Upload to GCS (To be accessible via URL)  
  - Turn video to 9:16 (Shotstack API)  
  - Rendering... (Wait for Shotstack render)  
  - Done?1 (Check Shotstack status)  
  - Done? (If done, proceed)  
  - Configure me (Set Postiz API endpoint and share title)  
  - Download final video  
  - Upload video to Postiz

- **Node Details:**  
  - **Convert to File**  
    - Type: Convert to File Node  
    - Role: Converts base64 encoded video string from Veo output to binary video file for upload.  
    - Input: `response.videos[0].bytesBase64Encoded` from Veo API.  
    - Output: Binary video file to GCS upload.  
    - Edge cases: Base64 decode errors.

  - **Upload to GCS (To be accessible via URL)**  
    - Type: Google Cloud Storage Node  
    - Role: Uploads binary video file to configured Google Storage bucket `veo_courses`.  
    - Input: Binary video from Convert to File.  
    - Output: Media link for Shotstack.  
    - Edge cases: Permission errors, bucket not found.

  - **Turn video to 9:16**  
    - Type: HTTP Request Node (Shotstack API)  
    - Role: Sends request to Shotstack to crop video to vertical 9:16 aspect ratio with HD resolution, 8 seconds duration.  
    - Input: Media link from GCS upload.  
    - Output: Shotstack render job ID.  
    - Credentials: Shotstack API key via HTTP header auth.  
    - Edge cases: API errors, invalid URL, quota limits.

  - **Rendering...**  
    - Type: Wait Node  
    - Role: Waits after submission to Shotstack before checking status.  
    - Output: To "Done?1."

  - **Done?1**  
    - Type: HTTP Request  
    - Role: Queries Shotstack render endpoint with job ID to check render status.  
    - Output: To "Done?" node.

  - **Done?**  
    - Type: If Node  
    - Role: Checks if Shotstack render status is "done."  
    - If yes → proceeds to "Configure me" node.  
    - If no → loops back to "Rendering..." wait node.  
    - Edge cases: Unexpected status values or network issues.

  - **Configure me**  
    - Type: Set Node  
    - Role: Sets Postiz API URL and prepares share title from AI-generated caption.  
    - Outputs: To "Download final video."

  - **Download final video**  
    - Type: HTTP Request  
    - Role: Downloads the finished video file from Shotstack using returned URL.  
    - Outputs: Binary data to "Upload video to Postiz."

  - **Upload video to Postiz**  
    - Type: HTTP Request  
    - Role: Uploads video binary to Postiz public API for later scheduling and publishing.  
    - Authentication: Generic HTTP header auth with Postiz API key.  
    - Outputs: Postiz upload response to next block.  
    - Edge cases: Upload failures, file size limits.

---

#### 2.5 Auto-Publishing to Social Media

- **Overview:**  
  Retrieves available Postiz integrations, routes based on platform type, and schedules the video posts to TikTok, YouTube, and Instagram for next-day publishing.

- **Nodes Involved:**  
  - Get Postiz integrations  
  - Switch1 (routes by platform: TikTok, YouTube, Instagram)  
  - Schedule TikTok  
  - Schedule YouTube  
  - Schedule Instagram

- **Node Details:**  
  - **Get Postiz integrations**  
    - Type: HTTP Request  
    - Role: Fetches all active Postiz social media integrations linked to user account.  
    - Input: Authenticated request with Postiz API key.  
    - Output: List of integrations to "Switch1."  
    - Edge cases: Auth failure, no integrations.

  - **Switch1**  
    - Type: Switch Node  
    - Role: Routes each integration object to the correct scheduling node based on `identifier` field matching "tiktok", "youtube", or "instagram" (case sensitive for TikTok and YouTube; contains for Instagram).  
    - Outputs: Three outputs for respective scheduling nodes.

  - **Schedule TikTok**  
    - Type: HTTP Request  
    - Role: Posts scheduling request to Postiz API for TikTok.  
    - Body includes scheduled date (next day), video media ID, caption, and TikTok-specific settings like privacy, comments, duet/stitch disabled, etc.  
    - Edge cases: Scheduling errors, invalid integration ID.

  - **Schedule YouTube**  
    - Type: HTTP Request  
    - Role: Posts scheduling request to Postiz API for YouTube.  
    - Body includes scheduled date (next day), video media ID, caption, and public visibility settings.  
    - Edge cases: Scheduling errors.

  - **Schedule Instagram**  
    - Type: HTTP Request  
    - Role: Posts scheduling request to Postiz API for Instagram.  
    - Body includes scheduled date (next day), video media ID, caption, and post type.  
    - Edge cases: Scheduling errors.

---

### 3. Summary Table

| Node Name                    | Node Type                             | Functional Role                                           | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                   |
|------------------------------|-------------------------------------|-----------------------------------------------------------|------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                      | Manual workflow start                                     |                              | Get Past Objects               |                                                                                               |
| Get Past Objects              | Google Sheets                       | Reads previously used fruit objects                       | When clicking ‘Execute workflow’ | Aggregate                     |                                                                                               |
| Aggregate                    | Aggregate Node                      | Aggregates sheet rows into single data array              | Get Past Objects              | Set Object List                |                                                                                               |
| Set Object List              | Set Node                           | Prepares array of used fruit objects                       | Aggregate                    | Idea Agent                    |                                                                                               |
| Idea Agent                  | LangChain Agent                     | Generates new unique glass fruit idea                      | Set Object List              | Object & Caption              |                                                                                               |
| Object & Caption            | LangChain Output Parser             | Validates/parses AI JSON output                            | Idea Agent                  | Idea Agent (for next step)     |                                                                                               |
| Prompt Agent                | LangChain Agent                     | Creates detailed cinematic video prompt for Veo           | Object & Caption             | SET                          |                                                                                               |
| OpenAI Chat Model           | LM Chat Model                      | Language model for AI agents                               | Idea Agent, Prompt Agent     | Idea Agent, Prompt Agent      |                                                                                               |
| SET                        | Set Node                           | Holds Google Cloud credentials and API endpoint           | Prompt Agent                | JWT                          | # ❗ Config                                                                                   |
| JWT                        | JWT Node                          | Creates JWT for Google OAuth2                              | SET                         | GET TOKEN                    |                                                                                               |
| GET TOKEN                  | HTTP Request                      | Exchanges JWT for OAuth token                              | JWT                         | Generate Video               |                                                                                               |
| Generate Video             | HTTP Request                      | Sends prompt to Veo API to generate video                 | GET TOKEN                   | Wait                         | 🎬 PART 3 — Video Generation (Google Veo 3.0)                                                 |
| Wait                       | Wait Node                        | Pauses before polling status                               | Generate Video              | Fetch Status                 |                                                                                               |
| Fetch Status               | HTTP Request                      | Checks Veo video generation status                         | Wait                        | Switch                       |                                                                                               |
| Switch                     | Switch Node                      | Checks if video generation is done                         | Fetch Status                | Convert to File, Wait         |                                                                                               |
| Convert to File            | Convert To File                   | Converts base64 video string to binary                     | Switch                      | Upload to GCS (To be accessible via URL) | 📱 PART 4 — Video Conversion & Upload                                                        |
| Upload to GCS (To be accessible via URL) | Google Cloud Storage              | Uploads video to GCS bucket                                | Convert to File             | Turn video to 9:16            |                                                                                               |
| Turn video to 9:16          | HTTP Request (Shotstack API)     | Crops video to vertical 9:16 format                        | Upload to GCS               | Rendering...                 |                                                                                               |
| Rendering...                | Wait Node                        | Waits for Shotstack render completion                      | Turn video to 9:16          | Done?1                      |                                                                                               |
| Done?1                     | HTTP Request                     | Checks Shotstack render status                             | Rendering...                | Done?                       |                                                                                               |
| Done?                      | If Node                         | Determines if render is done or needs more waiting        | Done?1                      | Configure me, Rendering...    |                                                                                               |
| Configure me               | Set Node                        | Prepares Postiz API URL and post title                     | Done?                       | Download final video          |                                                                                               |
| Download final video       | HTTP Request                    | Downloads the final rendered video                         | Configure me                | Upload video to Postiz        |                                                                                               |
| Upload video to Postiz     | HTTP Request                    | Uploads video to Postiz for publishing                     | Download final video        | Get Postiz integrations       |                                                                                               |
| Get Postiz integrations    | HTTP Request                    | Retrieves connected social media integrations              | Upload video to Postiz      | Switch1                     | ## Checking the available channels                                                           |
| Switch1                    | Switch Node                    | Routes integrations by platform (TikTok, YouTube, Instagram) | Get Postiz integrations     | Schedule TikTok, Schedule YouTube, Schedule Instagram | ## Scheduling the posts                                                                        |
| Schedule TikTok            | HTTP Request                    | Schedules TikTok post via Postiz                           | Switch1 (tiktok output)     |                             |                                                                                               |
| Schedule YouTube           | HTTP Request                    | Schedules YouTube post via Postiz                          | Switch1 (youtube output)    |                             |                                                                                               |
| Schedule Instagram         | HTTP Request                    | Schedules Instagram post via Postiz                        | Switch1 (instagram output)  |                             |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node** named `When clicking ‘Execute workflow’` for manual start.

2. **Add Google Sheets node** `Get Past Objects`:  
   - Connect from manual trigger.  
   - Configure with document ID: `1hNzWblAWsI_5u4RUK7kC3tSbR9riex94ZUuslhhDb8o` and sheet gid=0.  
   - No filters to read entire sheet.

3. **Add Aggregate node** `Aggregate`:  
   - Connect from Google Sheets node.  
   - Set it to aggregate all item data into one array.

4. **Add Set node** `Set Object List`:  
   - Connect from Aggregate node.  
   - Create a field `objects` as array with values:  
     `["{{$json.data[0].object}}", "{{$json.data[1].object}}"]`  
   - This extracts used fruits for AI input.

5. **Add LangChain Agent node** `Idea Agent`:  
   - Connect from `Set Object List`.  
   - Input text: `=Objects: {{ $json.objects.join(", ") }}`  
   - System prompt instructs to generate one unique fruit not in the list, output JSON with `object` and `caption`.  
   - Use OpenAI GPT-4.1-mini model (link below).  
   - Set output parser to structured JSON with fields `object` and `caption`.

6. **Add LangChain Output Parser node** `Object & Caption`:  
   - Connect as output parser for `Idea Agent`.  
   - Use manual schema with properties:  
     - `object`: string, lowercase, no punctuation  
     - `caption`: string, exact match format `"glass [object] ASMR"`.

7. **Add LangChain Agent node** `Prompt Agent`:  
   - Connect from `Object & Caption`.  
   - Pass the `object` field as input text.  
   - System prompt instructs creation of a safe, cinematic Google Veo 3.0 video prompt describing the ASMR glass fruit video with detailed policy compliance and ASMR sound layers.  
   - Use GPT-4.1-mini model.

8. **Add Set node** `SET`:  
   - Connect from `Prompt Agent`.  
   - Define Google Cloud credentials placeholders:  
     - `PROJECT_ID` (string, empty by default)  
     - `CLIENT_EMAIL` (string, empty by default)  
     - `LOCATION_ID` (string, default `"us-central1"`)  
     - `API_ENDPOINT` (string, default `"us-central1-aiplatform.googleapis.com"`)

9. **Add JWT node** `JWT`:  
   - Connect from `SET`.  
   - Configure JWT claims in JSON:  
     - `iss`: `CLIENT_EMAIL`  
     - `scope`: `"https://www.googleapis.com/auth/cloud-platform"`  
     - `aud`: `"https://www.googleapis.com/oauth2/v4/token"`  
     - `exp`: current time + 3500 sec  
     - `iat`: current time

10. **Add HTTP Request node** `GET TOKEN`:  
    - Connect from `JWT`.  
    - POST to `https://www.googleapis.com/oauth2/v4/token` with form-urlencoded body:  
      - `grant_type`: `urn:ietf:params:oauth:grant-type:jwt-bearer`  
      - `assertion`: JWT token  
    - Parse response for access token.

11. **Add HTTP Request node** `Generate Video`:  
    - Connect from `GET TOKEN`.  
    - POST to Google Veo API endpoint constructed from `SET` node variables.  
    - JSON body includes prompt text from `Prompt Agent` output JSON stringified.  
    - Parameters: aspectRatio "16:9", duration 8 sec, sampleCount 1, personGeneration "allow_all", watermark false, generateAudio true.  
    - Authorization header: Bearer token from `GET TOKEN`.

12. **Add Wait node** `Wait`:  
    - Connect from `Generate Video`.  
    - Wait 20 seconds before checking status.

13. **Add HTTP Request node** `Fetch Status`:  
    - Connect from `Wait`.  
    - POST to Veo API fetchPredictOperation endpoint with operation name from "Generate Video" response.  
    - Authorization header: Bearer token.

14. **Add Switch node** `Switch`:  
    - Connect from `Fetch Status`.  
    - If `response.videos[0].bytesBase64Encoded` exists → proceed to `Convert to File`.  
    - Else → loop back to `Wait`.

15. **Add Convert to File node** `Convert to File`:  
    - Connect from `Switch`.  
    - Convert base64 video string (`response.videos[0].bytesBase64Encoded`) to binary file.

16. **Add Google Cloud Storage node** `Upload to GCS (To be accessible via URL)`:  
    - Connect from `Convert to File`.  
    - Upload binary video to bucket `veo_courses` with object name `ViralReelz`.  
    - Requires configured Google Cloud Storage credentials.

17. **Add HTTP Request node** `Turn video to 9:16`:  
    - Connect from `Upload to GCS`.  
    - POST to Shotstack API `https://api.shotstack.io/v1/render` with JSON body:  
      - Timeline with clip referencing uploaded video URL  
      - Output format mp4, aspectRatio 9:16, resolution hd  
    - Authenticate using Shotstack API key (generic HTTP header auth).

18. **Add Wait node** `Rendering...`:  
    - Connect from `Turn video to 9:16`.  
    - Wait for Shotstack render process.

19. **Add HTTP Request node** `Done?1`:  
    - Connect from `Rendering...`.  
    - GET Shotstack render status using render ID from previous step.

20. **Add If node** `Done?`:  
    - Connect from `Done?1`.  
    - Check if render status equals "done".  
    - If yes → `Configure me` node; if no → loop back to `Rendering...`.

21. **Add Set node** `Configure me`:  
    - Connect from `Done?` (true).  
    - Set Postiz API base URL: `https://api.postiz.com/public/v1`.  
    - Set `share_title` from AI caption output.

22. **Add HTTP Request node** `Download final video`:  
    - Connect from `Configure me`.  
    - Download final video file from Shotstack URL.

23. **Add HTTP Request node** `Upload video to Postiz`:  
    - Connect from `Download final video`.  
    - POST to Postiz upload endpoint with video binary.  
    - Authentication: Postiz API key via HTTP header.

24. **Add HTTP Request node** `Get Postiz integrations`:  
    - Connect from `Upload video to Postiz`.  
    - GET list of social media integrations.

25. **Add Switch node** `Switch1`:  
    - Connect from `Get Postiz integrations`.  
    - Route by integration identifier: "tiktok", "youtube", or "instagram".

26. **Add HTTP Request nodes** `Schedule TikTok`, `Schedule YouTube`, `Schedule Instagram`:  
    - Connect to respective outputs of `Switch1`.  
    - POST scheduling requests with next day’s date, video ID, captions, and platform-specific settings.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| 🎥 Veo Machine Reels Factory is an automated content creation system built in n8n that creates ASMR-style glass fruit reels for TikTok, YouTube Shorts, and Instagram Reels. It integrates AI agents (OpenAI GPT-4), Google Veo 3.0, Shotstack video editing API, Google Cloud Storage, and Postiz social media scheduling API.                                                                                                                                                                                                                                                        | Full project overview and architecture summary.                                                                              |
| ⚙️ Requires Google Cloud project credentials, Veo API access, Google Sheets with used fruits, OpenAI API key, Shotstack API key, Postiz account with integrations, and n8n instance with support for LangChain nodes and HTTP requests.                                                                                                                                                                                                                                                                                                                                     | Setup prerequisites.                                                                                                         |
| The AI Prompt Agent strictly enforces content policy to avoid triggering Google Veo filters by describing the fruit object as resin-like, using choreographed blade motion without destructive language, emphasizing CGI stylization and ASMR sound layers.                                                                                                                                                                                                                                                                                                                | Prompt Agent system message instructions.                                                                                    |
| Video generation polling interval is 20 seconds by default; can be adjusted if videos take longer.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Polling wait time configuration.                                                                                            |
| Shotstack API crops the video to vertical 9:16 HD format.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Video conversion step.                                                                                                      |
| Postiz API supports scheduling posts on TikTok, YouTube, and Instagram with platform-specific settings such as privacy, comments, duet/stitch for TikTok, and post type for Instagram.                                                                                                                                                                                                                                                                                                                                                                                        | Social media publishing API.                                                                                                |
| For more information about the Veo 3.0 API, consult Google Vertex AI documentation. Shotstack docs: https://shotstack.io/docs/api/ Postiz docs: https://postiz.com/docs/ OpenAI docs: https://platform.openai.com/docs/                                                                                                                                                                                                                                                                                                                                                         | Useful official documentation links.                                                                                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.