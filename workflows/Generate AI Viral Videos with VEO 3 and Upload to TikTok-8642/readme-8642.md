Generate AI Viral Videos with VEO 3 and Upload to TikTok

https://n8nworkflows.xyz/workflows/generate-ai-viral-videos-with-veo-3-and-upload-to-tiktok-8642


# Generate AI Viral Videos with VEO 3 and Upload to TikTok

---

### 1. Workflow Overview

This workflow automates the creation of viral short-form videos optimized for TikTok using AI-generated concepts and the VEO 3 video generation API. It streamlines the entire content creation pipeline from ideation through video production to publishing on TikTok, enabling users to scale their content output with minimal manual intervention.

The workflow is logically divided into three main blocks:

- **1.1 Daily Trigger & Idea Generation:** Automatically triggers daily to generate a creative video idea using OpenAI GPT-5 and LangChain AI tools, parses structured AI output, and saves metadata to Google Sheets.
- **1.2 Video Script Generation & VEO 3 Video Creation:** Based on the saved idea, an AI agent crafts a detailed video prompt. The prompt is formatted and sent to the VEO 3 API to generate a cinematic vertical video. It includes a wait period to allow rendering, then downloads the finished video.
- **1.3 Video Upload & Status Update:** The downloaded video URL is stored in Google Sheets, uploaded to Blotato (a TikTok integration node), then posted to TikTok. The production status is updated to reflect completion.

Supporting nodes include sticky notes with detailed instructions and documentation links.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger & Idea Generation

**Overview:**  
This block initiates the workflow on a daily schedule. It generates a creative "before/after transformation" video idea using AI, parses the structured output, and records the idea and metadata to a Google Sheet.

**Nodes Involved:**  
- Trigger: Start Daily Content Generation  
- Tool: Inject Creative Perspective (Idea)  
- LLM: Generate Raw Idea (GPT-5)  
- Generate Creative Video Idea (LangChain agent)  
- Parse AI Output (Idea, Environment, Sound)  
- Save Idea & Metadata to Google Sheets  

**Node Details:**

- **Trigger: Start Daily Content Generation**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow daily (interval set to 1 day)  
  - Inputs: None  
  - Outputs: Triggers "Generate Creative Video Idea" node  
  - Edge cases: Misconfigured scheduling or disabled workflow can prevent triggering  

- **Tool: Inject Creative Perspective (Idea)**  
  - Type: LangChain Tool Node (Think Tool)  
  - Role: Provides AI with creative perspective to improve idea generation  
  - Inputs: Trigger output  
  - Outputs: Connected to "Generate Creative Video Idea" for enhanced ideation  
  - Edge cases: AI tool failures or timeouts  

- **LLM: Generate Raw Idea (GPT-5)**  
  - Type: LangChain Chat OpenAI (GPT-5 mini)  
  - Role: Generates an initial raw before/after transformation idea using GPT-5  
  - Configuration: Uses OpenAI API with GPT-5 mini model  
  - Inputs: Invoked by "Generate Creative Video Idea"  
  - Outputs: Raw AI text passed forward  
  - Edge cases: API quota exceeded, auth errors, model response delays  

- **Generate Creative Video Idea**  
  - Type: LangChain Agent Node  
  - Role: Processes raw idea, enriches it with structured rules and formatting for TikTok content  
  - Configuration: Complex system message enforcing strict formatting and content rules (e.g., 12 hashtags, emoji, environment, sound prompt)  
  - Inputs: Output from LLM and Think tool nodes  
  - Outputs: Structured JSON array with keys: Caption, Idea, Environment, Sound, Status  
  - Edge cases: Parsing errors if AI output does not conform to expected schema  

- **Parse AI Output (Idea, Environment, Sound)**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the AI-generated JSON array into structured data for further processing  
  - Inputs: AI output string from "Generate Creative Video Idea"  
  - Outputs: Parsed structured JSON with fields usable by downstream nodes  
  - Edge cases: JSON schema mismatch or malformed AI output  

- **Save Idea & Metadata to Google Sheets**  
  - Type: Google Sheets Node  
  - Role: Appends the generated idea and metadata (caption, status, prompts) to a Google Sheet for tracking  
  - Configuration: Defined column mapping for idea, caption, production status, environment, sound  
  - Inputs: Parsed AI output JSON  
  - Outputs: Triggers next block ("Set Master Prompt")  
  - Edge cases: Google Sheets API quota, auth expiration, network issues  

---

#### 2.2 Video Script Generation & VEO 3 Video Creation

**Overview:**  
Generates a detailed video script prompt based on the saved idea metadata, formats it for VEO 3 API consumption, triggers video generation, waits for rendering, and downloads the final video.

**Nodes Involved:**  
- Set Master Prompt  
- AI Agent: Generate Video Script  
- Format Prompt  
- Generate Video with VEO3 (HTTP Request)  
- Wait for VEO3 Rendering  
- Download Video from VEO3  

**Node Details:**

- **Set Master Prompt**  
  - Type: Set Node  
  - Role: Defines a comprehensive JSON master prompt schema describing cinematic video generation parameters such as description, style, camera, lighting, environment, motion, VFX, audio, ending, and format (aspect ratio 9:16)  
  - Configuration: Static JSON string assigned to variable `json_master` plus model and aspect ratio keys  
  - Inputs: Triggered after saving idea metadata  
  - Outputs: Passes master prompt to AI Agent node  
  - Edge cases: Syntax errors in JSON, missing fields affecting downstream processing  

- **AI Agent: Generate Video Script**  
  - Type: LangChain Agent Node  
  - Role: Uses the previously saved idea, environment, and sound prompts to generate a detailed VEO 3-compatible video prompt JSON with fields such as BEFORE scene, AFTER scene, TRANSITION, CAMERA, LIGHTING, PALETTE, STYLE, and SOUND  
  - Configuration: System prompt enforcing strict JSON output with only two top-level keys: `title` and `final_prompt` (stringified JSON)  
  - Inputs: Idea metadata + master prompt  
  - Outputs: Structured AI JSON object parsed by next node  
  - Edge cases: AI output formatting errors, token limit issues, API failures  

- **Format Prompt**  
  - Type: Code Node (JavaScript)  
  - Role: Converts the AI agent’s JSON output into a single JSON string escaped properly to be used as the `prompt` parameter for VEO 3 API  
  - Inputs: AI Agent output  
  - Outputs: Prepared HTTP request body fields (prompt, model, aspectRatio)  
  - Edge cases: JSON stringify errors, null inputs  

- **Generate Video with VEO3**  
  - Type: HTTP Request Node  
  - Role: Sends POST request to VEO 3 API `https://api.kie.ai/api/v1/veo/generate` to start video generation with the crafted prompt  
  - Configuration: Uses HTTP Header Authentication with Kie AI credentials  
  - Inputs: Formatted prompt and model/aspect ratio settings  
  - Outputs: Returns a `taskId` for video rendering tracking  
  - Edge cases: API rate limits, auth token expiration, request timeouts  

- **Wait for VEO3 Rendering**  
  - Type: Wait Node  
  - Role: Pauses workflow execution for 3 minutes to allow VEO 3 to process and render the video  
  - Inputs: Triggered after video generation request  
  - Outputs: After wait, triggers download node  
  - Edge cases: Fixed wait time may be insufficient for longer renders  

- **Download Video from VEO3**  
  - Type: HTTP Request Node  
  - Role: Queries `https://api.kie.ai/api/v1/veo/record-info` with `taskId` to retrieve rendering status and download URL of the generated video  
  - Configuration: HTTP Header Authentication with Kie AI credentials  
  - Inputs: taskId from generation response  
  - Outputs: Returns JSON containing final video URLs  
  - Edge cases: Video not ready yet, API errors, malformed responses  

---

#### 2.3 Video Upload & Status Update

**Overview:**  
Stores the final video URL in Google Sheets, uploads the video to Blotato for TikTok, posts the video to TikTok, and updates the production status in Google Sheets to “DONE”.

**Nodes Involved:**  
- URL Final Video (Google Sheets)  
- Upload Video to BLOTATO  
- TikTok (Blotato node)  
- Update Status to "DONE" (Google Sheets)  

**Node Details:**

- **URL Final Video**  
  - Type: Google Sheets Node  
  - Role: Updates the same row in Google Sheets with the final video URL and marks production as complete (“done”)  
  - Inputs: Video download response JSON  
  - Outputs: Triggers video upload node  
  - Edge cases: Sheet update conflicts, API limits  

- **Upload Video to BLOTATO**  
  - Type: Blotato Node (Community node)  
  - Role: Uploads the video media URL to Blotato platform, preparing it for TikTok posting  
  - Configuration: Uses Blotato API credentials, resource set to "media"  
  - Inputs: Final video URL  
  - Outputs: Provides media URL required by TikTok posting node  
  - Edge cases: Upload failures, network issues, incorrect credentials  

- **TikTok**  
  - Type: Blotato Node (TikTok integration)  
  - Role: Creates a TikTok post with the uploaded video and caption from Google Sheets, marking the post as AI-generated  
  - Configuration: TikTok platform, account ID parameter, caption text pulled from saved idea metadata, media URL from previous node  
  - Inputs: Uploaded media URL and caption  
  - Outputs: Triggers status update node  
  - Edge cases: TikTok API limits, authentication errors, content policy rejections  

- **Update Status to "DONE"**  
  - Type: Google Sheets Node  
  - Role: Updates the production status column in Google Sheets to “Publish” indicating completion of the entire workflow cycle  
  - Inputs: Confirmation from TikTok post node  
  - Outputs: Workflow ends here for this iteration  
  - Edge cases: Sheet locking, API errors  

---

### 3. Summary Table

| Node Name                            | Node Type                           | Functional Role                             | Input Node(s)                          | Output Node(s)                           | Sticky Note                                                                                          |
|------------------------------------|-----------------------------------|--------------------------------------------|--------------------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------------|
| Trigger: Start Daily Content Generation | Schedule Trigger                  | Initiates daily workflow trigger            | None                                 | Generate Creative Video Idea             |                                                                                                    |
| Tool: Inject Creative Perspective (Idea) | LangChain Tool (Think)            | Adds creative perspective to AI idea       | Trigger: Start Daily Content Generation | Generate Creative Video Idea             |                                                                                                    |
| LLM: Generate Raw Idea (GPT-5)     | LangChain LLM Chat OpenAI          | Generates raw AI idea using GPT-5           | Generate Creative Video Idea          | Generate Creative Video Idea             |                                                                                                    |
| Generate Creative Video Idea        | LangChain Agent                    | Produces structured video idea JSON         | Tool: Inject Creative Perspective, LLM: Generate Raw Idea | Parse AI Output (Idea, Environment, Sound) |                                                                                                    |
| Parse AI Output (Idea, Environment, Sound) | LangChain Output Parser Structured | Parses AI JSON output into structured data  | Generate Creative Video Idea          | Save Idea & Metadata to Google Sheets    |                                                                                                    |
| Save Idea & Metadata to Google Sheets | Google Sheets                     | Saves generated idea and metadata           | Parse AI Output                      | Set Master Prompt                       |                                                                                                    |
| Set Master Prompt                  | Set Node                          | Defines master prompt JSON schema            | Save Idea & Metadata to Google Sheets | AI Agent: Generate Video Script          |                                                                                                    |
| AI Agent: Generate Video Script     | LangChain Agent                    | Generates detailed video script prompt       | Set Master Prompt, OpenAI Chat Model, Think, Structured Output Parser | Format Prompt                            |                                                                                                    |
| Format Prompt                     | Code                             | Formats prompt JSON string for API           | AI Agent: Generate Video Script      | Generate Video with VEO3                 |                                                                                                    |
| Generate Video with VEO3            | HTTP Request                     | Calls VEO 3 API to generate video            | Format Prompt                       | Wait for VEO3 Rendering                  |                                                                                                    |
| Wait for VEO3 Rendering             | Wait                             | Waits fixed time for video rendering         | Generate Video with VEO3             | Download Video from VEO3                 |                                                                                                    |
| Download Video from VEO3            | HTTP Request                     | Retrieves video URL after rendering          | Wait for VEO3 Rendering              | URL Final Video                         |                                                                                                    |
| URL Final Video                   | Google Sheets                     | Updates Google Sheet with final video URL    | Download Video from VEO3             | Upload Video to BLOTATO                  |                                                                                                    |
| Upload Video to BLOTATO             | Blotato Node                     | Uploads video to Blotato for TikTok          | URL Final Video                     | TikTok                                  |                                                                                                    |
| TikTok                           | Blotato Node                     | Posts video to TikTok with caption           | Upload Video to BLOTATO              | Update Status to "DONE"                  |                                                                                                    |
| Update Status to "DONE"            | Google Sheets                     | Marks production status as complete          | TikTok                            | None                                    |                                                                                                    |
| Sticky Note8                      | Sticky Note                      | Contains tutorial video, documentation link, and detailed setup instructions | None                               | None                                    | See links for tutorial and setup: https://automatisation.notion.site/Generate-AI-Viral-Videos-with-VEO-3-and-Upload-to-TikTok-2703d6550fd980aa9ea1dd7867c1cccf?source=copy_link |
| Sticky Note                      | Sticky Note                      | Explains workflow purpose and use case       | None                               | None                                    | Explains entire workflow’s problem solving scope                                                   |
| Sticky Note7                     | Sticky Note                      | Marks input idea section                      | None                               | None                                    |                                                                                                    |
| Sticky Note9                     | Sticky Note                      | Marks TikTok publish step                     | None                               | None                                    |                                                                                                    |
| Sticky Note1                     | Sticky Note                      | Marks VEO3 video generation step              | None                               | None                                    |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily (interval: 1 day)  
   - Name: "Trigger: Start Daily Content Generation"

2. **Add LangChain Think Tool node**  
   - Type: LangChain Tool Think  
   - Name: "Tool: Inject Creative Perspective (Idea)"  
   - Connect from Schedule Trigger node

3. **Add LangChain Chat OpenAI node**  
   - Type: LangChain LLM Chat OpenAI  
   - Model: GPT-5 mini  
   - Name: "LLM: Generate Raw Idea (GPT-5)"  
   - Connect from Think Tool node  
   - Set OpenAI credentials with valid API key

4. **Add LangChain Agent node for idea generation**  
   - Name: "Generate Creative Video Idea"  
   - Set system message with detailed prompt enforcing strict output format (see Block 2.1)  
   - Connect inputs from LLM: Generate Raw Idea and Think Tool nodes

5. **Add LangChain Output Parser Structured node**  
   - Name: "Parse AI Output (Idea, Environment, Sound)"  
   - Configure JSON schema example matching expected AI output structure  
   - Connect from "Generate Creative Video Idea"

6. **Add Google Sheets node (append)**  
   - Name: "Save Idea & Metadata to Google Sheets"  
   - Configure document ID and sheet name  
   - Map columns: id (ROW()-1), idea, caption, production status, environment_prompt, sound_prompt  
   - Connect from "Parse AI Output"

7. **Add Set node**  
   - Name: "Set Master Prompt"  
   - Create JSON master prompt variable with detailed cinematic video schema (see Block 2.2)  
   - Set model to "veo3_fast" and aspectRatio to "9:16"  
   - Connect from "Save Idea & Metadata to Google Sheets"

8. **Add LangChain Agent node for video script generation**  
   - Name: "AI Agent: Generate Video Script"  
   - Use system message enforcing output of JSON object with `title` and `final_prompt` keys  
   - Inputs: idea, environment_prompt, sound_prompt from Google Sheets; master prompt from Set node  
   - Connect from "Set Master Prompt" node  
   - Provide OpenAI GPT-4.1-mini credentials

9. **Add Code node**  
   - Name: "Format Prompt"  
   - JavaScript code: stringify and escape the `final_prompt` for API request (see Block 2.2)  
   - Connect from "AI Agent: Generate Video Script"

10. **Add HTTP Request node**  
    - Name: "Generate Video with VEO3"  
    - Method: POST  
    - URL: `https://api.kie.ai/api/v1/veo/generate`  
    - Body: JSON with keys `prompt`, `model`, `aspectRatio` using expressions from "Format Prompt"  
    - Authentication: HTTP Header Auth with Kie AI API key  
    - Connect from "Format Prompt"

11. **Add Wait node**  
    - Name: "Wait for VEO3 Rendering"  
    - Duration: 3 minutes  
    - Connect from "Generate Video with VEO3"

12. **Add HTTP Request node**  
    - Name: "Download Video from VEO3"  
    - Method: GET  
    - URL: `https://api.kie.ai/api/v1/veo/record-info`  
    - Query Parameter: `taskId` from previous node response  
    - Authentication: HTTP Header Auth with Kie API key  
    - Connect from "Wait for VEO3 Rendering"

13. **Add Google Sheets node (update)**  
    - Name: "URL Final Video"  
    - Updates Google Sheet row with final video URL and status "done"  
    - Map columns accordingly  
    - Connect from "Download Video from VEO3"

14. **Add Blotato node**  
    - Name: "Upload Video to BLOTATO"  
    - Resource: media  
    - Media URL: from Google Sheets final_output field  
    - Credentials: Blotato API key  
    - Connect from "URL Final Video"

15. **Add Blotato TikTok node**  
    - Name: "TikTok"  
    - Platform: TikTok  
    - Account ID: your TikTok account identifier  
    - Post content text: caption from Google Sheets  
    - Post content media URLs: URL from Upload Video node  
    - Set option "Is AI Generated"  
    - Credentials: Blotato API key  
    - Connect from "Upload Video to BLOTATO"

16. **Add Google Sheets node (append or update)**  
    - Name: "Update Status to \"DONE\""  
    - Update production status to "Publish"  
    - Connect from "TikTok"

17. **Add Sticky Notes**  
    - Add notes with setup instructions, tutorial links, and workflow explanations as per original workflow for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| 🎥 Watch This Tutorial on YouTube and access full documentation on Notion for detailed setup and usage instructions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | YouTube: https://youtu.be/E-_8KZ_FSeY  Notion Documentation: https://automatisation.notion.site/Generate-AI-Viral-Videos-with-VEO-3-and-Upload-to-TikTok-2703d6550fd980aa9ea1dd7867c1cccf?source=copy_link |
| Workflow automates entire viral video production pipeline from AI ideation, video generation with VEO 3, to TikTok publishing via Blotato, saving significant manual time and enabling content scaling.                                                                                                                                                                                                                                                                                                                                                                                                                             | General description and use case                                                                                                            |
| OpenAI API key setup instructions: obtain API key from https://platform.openai.com/api-keys, ensure billing is activated, and add the key to n8n credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | OpenAI API Keys page and billing setup                                                                                                     |
| VEO 3 (Kie AI) integration requires API key creation on the Kie dashboard with base URL `https://api.kie.ai/api/v1/veo/generate`. Use HTTP header authentication in n8n.                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Kie AI (VEO 3) API documentation                                                                                                           |
| Blotato node installation instructions for TikTok integration: install community node `@blotato/n8n-nodes-blotato`, obtain API key from Blotato, and configure in n8n credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Blotato Community Node and API key setup                                                                                                  |
| For support or customization inquiries, contact Dr. Firas on LinkedIn or YouTube.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | LinkedIn: https://www.linkedin.com/in/dr-firas/  YouTube: https://www.youtube.com/@DRFIRASS                                                  |

---

**Disclaimer:** This text is exclusively derived from an n8n automated workflow. It fully complies with content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly available.

---