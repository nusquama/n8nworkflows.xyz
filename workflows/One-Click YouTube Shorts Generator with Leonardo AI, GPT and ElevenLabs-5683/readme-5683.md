One-Click YouTube Shorts Generator with Leonardo AI, GPT and ElevenLabs

https://n8nworkflows.xyz/workflows/one-click-youtube-shorts-generator-with-leonardo-ai--gpt-and-elevenlabs-5683


# One-Click YouTube Shorts Generator with Leonardo AI, GPT and ElevenLabs

### 1. Workflow Overview

This workflow automates the creation of complete YouTube Shorts videos from a simple text prompt. It leverages multiple AI and cloud services to generate engaging video content in a one-click process. The workflow is structured into four main logical blocks:

**1.1 Content Generation**  
Receives a user’s text query and uses OpenAI to generate a structured video concept including a script (intro, base, call-to-action), title, and description.  

**1.2 Audio Processing**  
Converts the generated script text into high-quality speech audio with ElevenLabs TTS, obtains timing alignment data, and splits the audio into fixed-length segments for synchronization with video.

**1.3 Visual Generation**  
Transforms each audio segment into a visual prompt using OpenAI to guide Leonardo AI in creating images and short animated videos that sync with the audio timing.

**1.4 Final Assembly**  
Aggregates all video segments and audio, uploads assets to Cloudinary, and composes a final vertical video (1080x1920 MP4) using Creatomate, including smooth transitions and background audio.

Additional configuration and wait nodes handle API credentials, timing dependencies, and asynchronous processing delays.

---

### 2. Block-by-Block Analysis

#### 2.1 Content Generation

**Overview:**  
This block generates the video concept from a user’s text input using OpenAI’s language model. It outputs a JSON object containing the structured script, video title, and description optimized for YouTube engagement.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Ideator 🧠 (OpenAI node)  
- Script (Set node)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual trigger  
  - Role: Entry point to start the workflow on demand  
  - Inputs: None  
  - Outputs: Triggers the next node "Ideator 🧠"  
  - Failures: None expected

- **Ideator 🧠**  
  - Type: OpenAI (Langchain)  
  - Role: Generate video concept (script, title, description) in JSON format  
  - Configuration: Uses a custom system prompt that instructs the model to output JSON with fields "Script" (Intro, Base, CTA), "Title", and "Description". Uses a specific model version "o3-mini-2025-01-31".  
  - Key expressions: Injects the user query (sample: "Give me top 5 interesting facts about plastic") into the prompt  
  - Inputs: Trigger from Manual node  
  - Outputs: JSON with video concept  
  - Failures: Possible API key/auth errors, response parsing issues, or malformed JSON output

- **Script**  
  - Type: Set node  
  - Role: Concatenate the script segments (Intro, Base, CTA) into a single string for TTS input  
  - Configuration: Assigns a new field “Script” by concatenating the JSON fields from previous node using expression `{{ $json.message.content.Script.Intro }} {{ $json.message.content.Script.Base }} {{ $json.message.content.Script.CTA }}`  
  - Inputs: Output from Ideator 🧠  
  - Outputs: JSON with "Script" string ready for audio synthesis  
  - Failures: Expression evaluation errors if input JSON is malformed

---

#### 2.2 Audio Processing

**Overview:**  
This block converts the script text into speech audio with ElevenLabs, retrieves alignment data for timing, and slices the audio into 4-second segments to facilitate synchronized video generation.

**Nodes Involved:**  
- Script Generator (HTTP Request to ElevenLabs TTS)  
- HTTP Request (Python code runner for alignment slicing)  
- Split Out (Split node)  
- Upload Cloudinary (Upload audio to Cloudinary)

**Node Details:**

- **Script Generator**  
  - Type: HTTP Request  
  - Role: Call ElevenLabs TTS API to generate MP3 audio with timestamp alignment  
  - Configuration: POST request to ElevenLabs endpoint with parameters: text (from Script), output format mp3_44100_128, and model_id "eleven_multilingual_v2". Authentication via ElevenLabs API key  
  - Inputs: Script text from Set node  
  - Outputs: MP3 audio base64 and alignment data  
  - Failures: Auth errors, request timeouts, invalid text input

- **HTTP Request** (Python processing)  
  - Type: HTTP Request (executes Python code)  
  - Role: Runs custom Python code to parse ElevenLabs alignment JSON and slice audio into fixed 4-second caption segments  
  - Configuration: Sends Python code as a string with embedded expressions extracting ElevenLabs alignment data from previous node JSON  
  - Inputs: ElevenLabs TTS output JSON  
  - Outputs: JSON array of audio segments with words, id, and duration fields  
  - Failures: Code errors, JSON parsing failures, empty alignment data

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of audio segment objects into individual workflow items for parallel processing  
  - Inputs: Output from Python node  
  - Outputs: One item per audio segment  
  - Failures: Empty array input results in no output

- **Upload Cloudinary**  
  - Type: HTTP Request  
  - Role: Upload the generated audio file (base64) to Cloudinary for cloud storage and URL generation  
  - Configuration: POST to Cloudinary raw upload endpoint, using environment variable for cloud name, upload_preset "default"  
  - Inputs: Audio base64 string from ElevenLabs node or processed outputs  
  - Outputs: Cloudinary upload response containing secure_url  
  - Failures: Auth errors, upload failures, invalid base64 data

---

#### 2.3 Visual Generation

**Overview:**  
For each audio segment, this block creates a concise image prompt with OpenAI, then uses Leonardo AI to generate a high-quality image and converts it into a short animated video segment.

**Nodes Involved:**  
- image-prompter (OpenAI)  
- request image (Leonardo AI image generation)  
- Wait2 (wait 1 min)  
- request image1 (Leonardo AI generation status check)  
- Wait3 (wait 1 min)  
- request image2 (Leonardo AI motion video generation)  
- Wait4 (wait 2 min)  
- Request Video (Leonardo AI motion video status check)  
- Edit Fields (Set node)  
- Aggregate (aggregate video segments)  

**Node Details:**

- **image-prompter**  
  - Type: OpenAI  
  - Role: Generate a short, visually descriptive prompt for Leonardo AI based on current audio segment script  
  - Configuration: Uses a system prompt instructing to create simple, text-free image prompts under 240 characters, focusing on first frame of 4-second scene  
  - Inputs: Script segment text from Split Out  
  - Outputs: JSON with field "Prompt"  
  - Failures: API errors, prompt formatting issues

- **request image**  
  - Type: HTTP Request  
  - Role: Sends Leonardo AI image generation request using the prompt  
  - Configuration: POST to Leonardo API endpoint with JSON body specifying prompt, modelId, width (832), height (480), and num_images=1  
  - Inputs: Prompt from image-prompter  
  - Outputs: Generation job response with image ID  
  - Failures: Auth errors, API request failures, invalid prompt

- **Wait2**, **Wait3**, **Wait4**  
  - Type: Wait nodes  
  - Role: Delays to allow asynchronous Leonardo AI jobs to complete (1 or 2 minutes)  
  - Inputs: Previous node outputs  
  - Outputs: Triggers next steps after waiting

- **request image1**  
  - Type: HTTP Request  
  - Role: Poll Leonardo AI generation status for image generation job  
  - Inputs: Generation ID from previous node  
  - Outputs: Generation details including generated image IDs  
  - Failures: Request timeouts, job failure

- **request image2**  
  - Type: HTTP Request  
  - Role: Start motion SVD video generation from generated image ID  
  - Configuration: POST to Leonardo API motion endpoint with image ID and motionStrength=2  
  - Inputs: Generated image ID from previous step  
  - Outputs: Motion generation job response  
  - Failures: Motion generation failures, API errors

- **Request Video**  
  - Type: HTTP Request  
  - Role: Poll Leonardo AI for generated motion video status and URL  
  - Inputs: motionSvdGenerationJob.generationId from previous node  
  - Outputs: Final video URL (MP4)  
  - Failures: API failures, job timeouts

- **Edit Fields**  
  - Type: Set node  
  - Role: Extracts and sets the output field to the motion video URL for aggregation  
  - Inputs: Video generation response  
  - Outputs: JSON with "output" URL and metadata  
  - Failures: Missing URL or invalid response format

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Collects all video segment outputs into a single array for final composition  
  - Inputs: Multiple video segment nodes  
  - Outputs: Array of video segments  
  - Failures: Empty input arrays

---

#### 2.4 Final Assembly

**Overview:**  
This block merges audio and video segments, uploads necessary assets, and sends a render request to Creatomate to produce the final YouTube-ready vertical video.

**Nodes Involved:**  
- Merge (combine video segments and audio)  
- Create editor JSON (Python code generating Creatomate render config)  
- SET JSON VARIABLE (set Creatomate request payload)  
- Editor (HTTP Request to Creatomate API)  
- Rendering wait (Wait node)  
- Get final video (HTTP request to fetch render result)

**Node Details:**

- **Merge**  
  - Type: Merge node (combine mode)  
  - Role: Combine video segments (from Aggregate) and audio URL (from Upload Cloudinary) by position  
  - Inputs: Video segment array and audio URL  
  - Outputs: Combined dataset for render configuration  
  - Failures: Data mismatch, missing inputs

- **Create editor JSON**  
  - Type: HTTP Request (Python code runner)  
  - Role: Runs Python code to generate optimized Creatomate render JSON with video and audio timeline, crossfade animations, and vertical format settings  
  - Inputs: Combined video segments and audio URL  
  - Outputs: Creatomate-compatible JSON render configuration  
  - Failures: JSON parsing errors, input validation failures

- **SET JSON VARIABLE**  
  - Type: Set node  
  - Role: Assigns the render JSON to a variable for the Creatomate API call  
  - Inputs: Output from Create editor JSON  
  - Outputs: JSON object for API consumption  
  - Failures: Expression evaluation errors

- **Editor**  
  - Type: HTTP Request  
  - Role: Submit the video render job to Creatomate API  
  - Configuration: POST with JSON body containing the render configuration  
  - Inputs: Creatomate request JSON  
  - Outputs: Job ID and status from Creatomate  
  - Failures: API errors, auth failures

- **Rendering wait**  
  - Type: Wait node (70 seconds)  
  - Role: Delay to allow video render completion  
  - Inputs: Render job submission  
  - Outputs: Triggers final video retrieval  
  - Failures: Timeout too short or too long

- **Get final video**  
  - Type: HTTP Request  
  - Role: Poll Creatomate API for final render status and video URL using job ID  
  - Inputs: Job ID from Editor node  
  - Outputs: Final video metadata and URL  
  - Failures: Job failure, polling timeout

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                  | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                  |
|-----------------------------|--------------------------------|---------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                | Entry point to start workflow    | -                                | Ideator 🧠                      |                                                                                              |
| Ideator 🧠                  | OpenAI (Langchain)             | Generate video concept JSON      | When clicking ‘Execute workflow’ | Script                         | 🧠 Content Generation: Step 1: AI creates video concept, structured script, title, description |
| Script                      | Set                           | Concatenate script text          | Ideator 🧠                      | Script Generator               |                                                                                              |
| Script Generator            | HTTP Request                  | ElevenLabs TTS audio generation  | Script                          | HTTP Request, Upload Cloudinary | 🎙️ Audio Processing: Step 2: Convert text to speech, returns audio + timing alignment        |
| HTTP Request (Python code)  | HTTP Request (Python runner)  | Slice ElevenLabs alignment data  | Script Generator                | Split Out                     |                                                                                              |
| Split Out                   | Split Out                     | Split audio segments             | HTTP Request                   | image-prompter                |                                                                                              |
| image-prompter              | OpenAI (Langchain)            | Generate image prompt per segment| Split Out                      | request image                 | 🎨 Visual Generation: Step 3: Create video segments, AI generates image prompts               |
| request image               | HTTP Request                  | Leonardo AI image generation     | image-prompter                 | Wait2                         |                                                                                              |
| Wait2                       | Wait                          | Wait 1 minute for image gen      | request image                  | request image1                |                                                                                              |
| request image1              | HTTP Request                  | Poll Leonardo image generation   | Wait2                         | Wait3                         |                                                                                              |
| Wait3                       | Wait                          | Wait 1 minute                    | request image1                 | request image2                |                                                                                              |
| request image2              | HTTP Request                  | Leonardo AI motion video request | Wait3                         | Wait4                         |                                                                                              |
| Wait4                       | Wait                          | Wait 2 minutes                  | request image2                 | Request Video                 |                                                                                              |
| Request Video               | HTTP Request                  | Poll Leonardo motion video status| Wait4                         | Wait1                         |                                                                                              |
| Wait1                       | Wait                          | Wait 1 minute                   | Request Video                  | Edit Fields                   |                                                                                              |
| Edit Fields                 | Set                           | Extract video URL               | Wait1                         | Aggregate                     |                                                                                              |
| Aggregate                   | Aggregate                     | Aggregate video segments        | Edit Fields                   | Merge                        |                                                                                              |
| Upload Cloudinary           | HTTP Request                  | Upload audio to Cloudinary       | Script Generator               | Merge                        |                                                                                              |
| Merge                      | Merge                         | Combine audio and video segments | Aggregate, Upload Cloudinary   | Create editor JSON            | 🎬 Final Assembly: Step 4: Combine all segments, add audio, render final video                |
| Create editor JSON          | HTTP Request (Python runner)  | Generate Creatomate render config| Merge                        | SET JSON VARIABLE             |                                                                                              |
| SET JSON VARIABLE           | Set                           | Store Creatomate request JSON    | Create editor JSON             | Editor                       |                                                                                              |
| Editor                     | HTTP Request                  | Submit render job to Creatomate  | SET JSON VARIABLE             | Rendering wait                |                                                                                              |
| Rendering wait             | Wait                          | Wait for video render completion | Editor                       | Get final video               |                                                                                              |
| Get final video             | HTTP Request                  | Retrieve final video URL/status  | Rendering wait                | -                            |                                                                                              |
| sticky-main-info            | Sticky Note                   | Main workflow info               | -                            | -                            | 🎬 AI YouTube Video Generator - One-Click Automation Workflow overview                       |
| sticky-content-gen          | Sticky Note                   | Describes Content Generation block | -                            | -                            | 🧠 Content Generation overview                                                               |
| sticky-audio               | Sticky Note                   | Describes Audio Processing block | -                            | -                            | 🎙️ Audio Processing overview                                                                |
| sticky-visual              | Sticky Note                   | Describes Visual Generation block| -                            | -                            | 🎨 Visual Generation overview                                                               |
| sticky-assembly            | Sticky Note                   | Describes Final Assembly block  | -                            | -                            | 🎬 Final Assembly overview                                                                   |
| sticky-config              | Sticky Note                   | Configuration instructions      | -                            | -                            | ⚙️ Configuration notes: API keys, env variables, voice ID setup                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node** named “When clicking ‘Execute workflow’” to start manually.

2. **Add OpenAI node “Ideator 🧠”**  
   - Set model to `o3-mini-2025-01-31`  
   - System prompt: instruct model to output JSON with fields Script (Intro, Base, CTA), Title, Description as per the given prompt in the workflow  
   - User message: insert user query (e.g., "Give me top 5 interesting facts about plastic")  
   - Connect Manual Trigger -> Ideator 🧠  
   - Set credentials to your OpenAI API key.

3. **Add Set node “Script”**  
   - Assign new field “Script” by concatenating: `{{ $json.message.content.Script.Intro }} {{ $json.message.content.Script.Base }} {{ $json.message.content.Script.CTA }}`  
   - Connect Ideator 🧠 -> Script

4. **Add HTTP Request node “Script Generator”** for ElevenLabs TTS  
   - Method: POST  
   - URL: `https://api.elevenlabs.io/v1/text-to-speech/2qfp6zPuviqeCOZIE9RZ/with-timestamps?output_format=mp3_44100_128`  
   - Headers: Include `Content-Type: application/json`  
   - Body JSON: `{ "text": "{{ $json.Script }}", "output_format": "mp3_44100_128", "model_id": "eleven_multilingual_v2" }`  
   - Authentication: ElevenLabs API key via HTTP Header Auth  
   - Connect Script -> Script Generator

5. **Add HTTP Request node for Python code execution** (name it "HTTP Request")  
   - Method: POST  
   - URL: `https://prod.0codekit.com/code/python`  
   - Body: Paste the provided Python code that parses ElevenLabs alignment and creates 4-second audio chunks  
   - Authentication: Use appropriate HTTP header auth credential  
   - Connect Script Generator -> HTTP Request

6. **Add Split Out node “Split Out”**  
   - Field to split: `result.data` (from previous node output)  
   - Connect HTTP Request -> Split Out

7. **Add OpenAI node “image-prompter”**  
   - Model: `o3-mini-2025-01-31`  
   - System prompt: instruct to create concise visual prompts (<240 chars) based on current script segment  
   - Pass current segment words and full script for context  
   - Connect Split Out -> image-prompter  
   - OpenAI credentials set

8. **Add HTTP Request “request image”**  
   - Method: POST  
   - URL: `https://cloud.leonardo.ai/api/rest/v1/generations`  
   - Body JSON: include prompt from image-prompter, modelId, width=832, height=480, num_images=1  
   - Headers: `Content-Type: application/json`  
   - Auth: Leonardo AI API key  
   - Connect image-prompter -> request image

9. **Add Wait node “Wait2”** (1 minute)  
   - Connect request image -> Wait2

10. **Add HTTP Request “request image1”** to poll generation status  
    - URL: `https://cloud.leonardo.ai/api/rest/v1/generations/{{ generationId }}` (dynamic)  
    - Auth: Leonardo API  
    - Connect Wait2 -> request image1

11. **Add Wait node “Wait3”** (1 minute)  
    - Connect request image1 -> Wait3

12. **Add HTTP Request “request image2”** to request motion video generation  
    - POST `https://cloud.leonardo.ai/api/rest/v1/generations-motion-svd` with imageId and motionStrength=2  
    - Auth: Leonardo API  
    - Connect Wait3 -> request image2

13. **Add Wait node “Wait4”** (2 minutes)  
    - Connect request image2 -> Wait4

14. **Add HTTP Request “Request Video”** to poll motion video status  
    - URL: `https://cloud.leonardo.ai/api/rest/v1/generations/{{ generationId }}`  
    - Auth: Leonardo API  
    - Connect Wait4 -> Request Video

15. **Add Wait node “Wait1”** (1 minute)  
    - Connect Request Video -> Wait1

16. **Add Set node “Edit Fields”**  
    - Extract `motionMP4URL` from response to new field `output`  
    - Connect Wait1 -> Edit Fields

17. **Add Aggregate node “Aggregate”**  
    - Aggregate all video segment outputs into field “Videos”  
    - Connect Edit Fields -> Aggregate

18. **Add HTTP Request node “Upload Cloudinary”** to upload audio  
    - POST to `https://api.cloudinary.com/v1_1/{{ CLOUDINARY_CLOUD_NAME }}/raw/upload`  
    - Body form-urlencoded: file=`data:audio/mp3;base64,{{ $json.audio_base64 }}`, upload_preset=`default`  
    - Connect Script Generator -> Upload Cloudinary

19. **Add Merge node “Merge”**  
    - Mode: Combine by position, include unpaired  
    - Inputs: Aggregate (video segments), Upload Cloudinary (audio)  
    - Connect both nodes -> Merge

20. **Add HTTP Request node “Create editor JSON”**  
    - POST to `https://prod.0codekit.com/code/python`  
    - Body: Python code to generate Creatomate render JSON with timelines, transitions, audio track  
    - Auth: HTTP header auth credential  
    - Connect Merge -> Create editor JSON

21. **Add Set node “SET JSON VARIABLE”**  
    - Assign output from Create editor JSON to field “Creatomate Request”  
    - Connect Create editor JSON -> SET JSON VARIABLE

22. **Add HTTP Request node “Editor”**  
    - POST to `https://api.creatomate.com/v1/renders` with body containing `source` JSON from SET JSON VARIABLE  
    - Auth: Creatomate API key  
    - Connect SET JSON VARIABLE -> Editor

23. **Add Wait node “Rendering wait”** (70 seconds)  
    - Connect Editor -> Rendering wait

24. **Add HTTP Request node “Get final video”**  
    - GET from `https://api.creatomate.com/v1/renders/{{ Editor.id }}` to check render status and retrieve video URL  
    - Auth: Creatomate API  
    - Connect Rendering wait -> Get final video

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow requires API credentials for OpenAI, ElevenLabs, Leonardo AI, Creatomate, and Cloudinary.                 | See sticky note “Configuration” for setup details                                                        |
| Set environment variable `CLOUDINARY_CLOUD_NAME` to your Cloudinary cloud name.                                    | Environment variable required for Cloudinary upload node                                                 |
| ElevenLabs voice ID used: "2qfp6zPuviqeCOZIE9RZ" - customize this ID in Script Generator node as needed.           | ElevenLabs TTS voice configuration                                                                       |
| Video output format configured as vertical 1080x1920 MP4, optimized for YouTube Shorts and similar social platforms.| Final assembly sticky note                                                                               |
| Multiple wait nodes are used to handle asynchronous API processing and ensure job completion before proceeding.    | Leonardo AI jobs and Creatomate rendering require delays                                                 |
| Python code nodes run external code via HTTP request to `prod.0codekit.com` - ensure this service is accessible.   | Custom code for audio slicing and video JSON generation                                                  |
| Prompts are carefully crafted to avoid on-screen text in generated images to ensure smooth animation.              | Visual Generation sticky note                                                                            |
| Workflow tags indicate usage for “Selling” use case - likely commercial content generation.                        | Tag metadata                                                                                             |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, respecting all current content policies and legal requirements. It contains no illegal, offensive, or protected material. All data processed is legal and public.