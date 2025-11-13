Automated TikTok Video Creation Pipeline with GPT-4o-mini and Sisif.ai

https://n8nworkflows.xyz/workflows/automated-tiktok-video-creation-pipeline-with-gpt-4o-mini-and-sisif-ai-4969


# Automated TikTok Video Creation Pipeline with GPT-4o-mini and Sisif.ai

### 1. Workflow Overview

This workflow automates the creation of short TikTok videos using AI-generated content ideas and video generation services. Its primary use case is to regularly produce engaging, trendy short-form videos by leveraging GPT-4o-mini for creative idea generation and Sisif.ai for AI-driven video creation.

The workflow is logically grouped into these blocks:

- **1.1 Scheduling and Triggering**: Periodically triggers the workflow every 6 hours to initiate the video creation process.

- **1.2 AI Idea Generation**: Uses an OpenAI GPT-4o-mini chat model to generate a detailed, trendy, and visual video idea prompt formatted as structured JSON.

- **1.3 Video Generation Request**: Sends the AI-generated prompt to Sisif.ai API to request video creation with specified duration and resolution.

- **1.4 Video Status Polling and Completion**: Waits and polls Sisif.ai’s API until the video status is "READY," then prepares and outputs the final video data including URLs and metadata.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduling and Triggering

- **Overview:**  
  This block schedules the workflow to run every 6 hours, ensuring regular generation of new TikTok video content.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates the workflow automatically every 6 hours.  
    - *Configuration:* Interval trigger set to 6 hours.  
    - *Key Expressions:* None.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Connects to "Idea creator" node.  
    - *Version-specific:* Version 1.1, stable for schedule triggers.  
    - *Potential Failures:* Scheduler misconfiguration, n8n instance downtime.  
    - *Sub-workflow:* None.

---

#### 2.2 AI Idea Generation

- **Overview:**  
  This block uses a GPT-4o-mini chat model to generate a creative, detailed TikTok video idea prompt in JSON format, parsed by a structured output parser.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Idea creator

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat LLM node  
    - *Role:* Generates creative video ideas using GPT-4o-mini.  
    - *Configuration:* Model set to "gpt-4o-mini" with default options.  
    - *Key Expressions:* None beyond model selection.  
    - *Inputs:* Receives trigger from Schedule Trigger (indirect via Idea creator).  
    - *Outputs:* Sends output to "Idea creator" node’s language model input.  
    - *Credentials:* OpenAI API key configured.  
    - *Failures:* API rate limits, authentication errors, timeouts.  
    - *Sub-workflow:* None.

  - **Structured Output Parser**  
    - *Type:* LangChain Structured Output Parser  
    - *Role:* Parses raw LLM output into structured JSON for downstream consumption.  
    - *Configuration:* Example JSON schema expecting an object with an "idea" string property.  
    - *Key Expressions:* Uses example schema as a guide.  
    - *Inputs:* Receives AI model raw output (not direct trigger).  
    - *Outputs:* Feeds structured JSON to "Idea creator" node output parser input.  
    - *Failures:* Parsing errors if output doesn’t match schema; malformed JSON.  
    - *Sub-workflow:* None.

  - **Idea creator**  
    - *Type:* LangChain Chain LLM node  
    - *Role:* Orchestrates prompt to LLM and applies output parser.  
    - *Configuration:*  
      - Prompt instructs generation of a single detailed TikTok video idea prompt, visual and action-packed, 3-7 seconds, trendy, formatted as JSON with a single "idea" property.  
      - Output parser set to Structured Output Parser node.  
    - *Key Expressions:* Uses multi-line prompt with detailed instructions and JSON format.  
    - *Inputs:* Triggered by Schedule Trigger.  
    - *Outputs:* Sends generated idea JSON to "Create Sisif Video" node.  
    - *Failures:* Prompt misinterpretation, parser failure, connection errors.  
    - *Sub-workflow:* None.

---

#### 2.3 Video Generation Request

- **Overview:**  
  Sends the AI-generated video idea prompt to Sisif.ai to request video generation with fixed parameters for duration and resolution.

- **Nodes Involved:**  
  - Create Sisif Video  
  - Sticky Note2 (informational)

- **Node Details:**

  - **Create Sisif Video**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to Sisif.ai API endpoint to generate video.  
    - *Configuration:*  
      - URL: `https://sisif.ai/api/videos/generate/`  
      - Method: POST  
      - Body: JSON object with parameters:  
        - "prompt": from `{{$json.output.idea}}` (the AI-generated idea)  
        - "duration": fixed at 5 seconds  
        - "resolution": fixed at 540x960 (portrait format for TikTok)  
      - Authentication: HTTP Bearer Token (configured in credentials)  
    - *Key Expressions:* Uses expression to insert idea prompt dynamically.  
    - *Inputs:* Receives structured idea JSON from "Idea creator".  
    - *Outputs:* Sends response (containing video ID and metadata) to "Wait 10s".  
    - *Credentials:* HTTP Bearer Auth configured via n8n credentials.  
    - *Failures:* API auth errors, invalid prompt, network errors, rate limits.  
    - *Sub-workflow:* None.

  - **Sticky Note2**  
    - *Type:* Sticky Note (visual only)  
    - *Purpose:* Provides a reminder that this node interacts with Sisif.ai to generate videos from ideas.  
    - *Content:* "# Sisif.ai\n\n## Generate new video based on idea creator"  
    - *Inputs/Outputs:* None.

---

#### 2.4 Video Status Polling and Completion

- **Overview:**  
  This block waits for 10 seconds between polling attempts, repeatedly checks the Sisif.ai video generation status, and once the video is ready, prepares the final output including URLs and timestamps.

- **Nodes Involved:**  
  - Wait 10s  
  - Check Video Status  
  - Video Ready? (If node)  
  - Prepare Final Data

- **Node Details:**

  - **Wait 10s**  
    - *Type:* Wait node  
    - *Role:* Pauses the workflow for 10 seconds before polling again.  
    - *Configuration:* Wait for 10 seconds.  
    - *Inputs:* Receives video creation response or "Video Ready?" false output.  
    - *Outputs:* Passes data to "Check Video Status".  
    - *Failures:* None expected, except workflow interruption.  
    - *Sub-workflow:* None.

  - **Check Video Status**  
    - *Type:* HTTP Request  
    - *Role:* Sends GET request to Sisif.ai API to check video status by ID.  
    - *Configuration:*  
      - URL: `https://sisif.ai/api/videos/{{ $json.id }}/` (dynamic video ID extraction)  
      - Method: GET  
      - Authentication: HTTP Bearer Token as per credentials.  
    - *Inputs:* Receives from "Wait 10s".  
    - *Outputs:* Sends video status JSON to "Video Ready?" node.  
    - *Failures:* API auth errors, invalid video ID, network issues.  
    - *Sub-workflow:* None.

  - **Video Ready?**  
    - *Type:* If node  
    - *Role:* Checks if the video status equals "READY".  
    - *Configuration:* Condition: `$json.status == "READY"`  
    - *Inputs:* Receives video status JSON.  
    - *Outputs:*  
      - True: proceeds to "Prepare Final Data"  
      - False: loops back to "Wait 10s" for another poll cycle  
    - *Failures:* Expression evaluation failure if status field missing.  
    - *Sub-workflow:* None.

  - **Prepare Final Data**  
    - *Type:* Function  
    - *Role:* Extracts and formats the final video data from Sisif.ai response for output.  
    - *Configuration:* JavaScript code that:  
      - Extracts fields: prompt, duration, resolution, sisif_id, video_url, status  
      - Adds a watch URL to Sisif.ai’s video web player  
      - Adds a timestamp for completion  
    - *Inputs:* Receives video metadata JSON when ready.  
    - *Outputs:* Outputs a clean JSON object with all relevant video info.  
    - *Failures:* Code errors, missing expected fields.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                         | Input Node(s)               | Output Node(s)              | Sticky Note                                                    |
|---------------------|----------------------------------|---------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                 | Periodically triggers workflow start  | None                        | Idea creator                |                                                                |
| Idea creator        | LangChain Chain LLM              | Generates TikTok video ideas (JSON)   | Schedule Trigger             | Create Sisif Video          |                                                                |
| OpenAI Chat Model    | LangChain OpenAI Chat LLM        | Runs GPT-4o-mini to generate ideas    | Idea creator (ai_languageModel) | Idea creator (ai_outputParser) |                                                                |
| Structured Output Parser | LangChain Output Parser Structured | Parses GPT output into JSON           | OpenAI Chat Model (ai_outputParser) | Idea creator (ai_outputParser) |                                                                |
| Create Sisif Video   | HTTP Request                    | Sends video generation request to Sisif.ai | Idea creator               | Wait 10s                   | # Sisif.ai\n\n## Generate new video based on idea creator     |
| Wait 10s             | Wait                           | Waits 10 seconds before polling again | Create Sisif Video, Video Ready? (false branch) | Check Video Status          |                                                                |
| Check Video Status   | HTTP Request                   | Polls Sisif.ai API for video status   | Wait 10s                    | Video Ready?                |                                                                |
| Video Ready?         | If                             | Checks if video generation is complete | Check Video Status           | Prepare Final Data (true), Wait 10s (false) |                                                                |
| Prepare Final Data   | Function                       | Formats final video metadata output   | Video Ready? (true)          | None                       |                                                                |
| Sticky Note2         | Sticky Note                    | Informational note about Sisif.ai API | None                        | None                       | # Sisif.ai\n\n## Generate new video based on idea creator     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger every 6 hours (interval: hours, amount: 6).  
   - This node starts the workflow.

2. **Create an OpenAI Chat Model node**  
   - Use the LangChain OpenAI Chat LLM node.  
   - Select model: "gpt-4o-mini".  
   - Connect credentials with your OpenAI API key.  
   - No additional prompt input here — it will be used as a language model by the next node.

3. **Create a Structured Output Parser node**  
   - Use LangChain Output Parser Structured node.  
   - Provide an example JSON schema as:  
     ```json
     {
       "idea": "Generate video ..."
     }
     ```  
   - This will parse the raw LLM output into structured JSON.

4. **Create the Idea creator node**  
   - Use LangChain Chain LLM node.  
   - Configure prompt with detailed instructions to generate a single detailed TikTok video idea prompt (3-7 seconds, visual, trendy, suitable for AI video generation).  
   - Set output parser to the Structured Output Parser node created above.  
   - Connect the Schedule Trigger node to this node’s main input.  
   - Connect this node's ai_languageModel input to the OpenAI Chat Model node.  
   - Connect this node's ai_outputParser input to the Structured Output Parser node.

5. **Create the Create Sisif Video node (HTTP Request)**  
   - Set method to POST.  
   - URL: `https://sisif.ai/api/videos/generate/`  
   - Authentication: HTTP Bearer Token (configure with Sisif.ai API token in n8n credentials).  
   - Body Parameters (send as JSON):  
     - `prompt`: use expression `{{$json.output.idea}}` from Idea creator output.  
     - `duration`: 5 (seconds)  
     - `resolution`: "540x960" (portrait format).  
   - Connect Idea creator output to this node.

6. **Add a Sticky Note node**  
   - Content:  
     ```
     # Sisif.ai

     ## Generate new video based on idea creator
     ```  
   - Place near the Create Sisif Video node for clarity.

7. **Create a Wait node (Wait 10s)**  
   - Configure to wait 10 seconds.  
   - Connect output of Create Sisif Video node to this Wait node.

8. **Create the Check Video Status node (HTTP Request)**  
   - Method: GET  
   - URL: `https://sisif.ai/api/videos/{{ $json.id }}/` — dynamically insert returned video ID.  
   - Authentication: Same HTTP Bearer token as above.  
   - Connect Wait node output here.

9. **Create an If node (Video Ready?)**  
   - Condition: check if `$json.status === "READY"`  
   - Connect Check Video Status output here.

10. **Create a Function node (Prepare Final Data)**  
    - JavaScript code:  
      ```javascript
      const videoData = items[0].json;

      return [{
        json: {
          prompt: videoData.prompt,
          duration: videoData.duration,
          resolution: videoData.resolution,
          sisif_id: videoData.sisif_id,
          video_url: videoData.video_url,
          status: videoData.status,
          watch_url: `https://sisif.ai/app/watch/${videoData.sisif_id}/`,
          completed_at: new Date().toISOString()
        }
      }];
      ```  
    - Connect If node’s true branch to this node.

11. **Create looping for polling**  
    - Connect If node’s false branch back to the Wait 10s node to poll again.

12. **Final workflow connections recap:**  
    - Schedule Trigger → Idea creator  
    - Idea creator → Create Sisif Video  
    - Create Sisif Video → Wait 10s  
    - Wait 10s → Check Video Status  
    - Check Video Status → Video Ready?  
    - Video Ready? true → Prepare Final Data  
    - Video Ready? false → Wait 10s  

13. **Credentials setup:**  
    - OpenAI API key credential for GPT-4o-mini node.  
    - Sisif.ai API Bearer token configured as HTTP Bearer Auth credential for HTTP Request nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Sisif.ai is used for AI-based video generation by providing prompts and video parameters via API.  | https://sisif.ai/api                                           |
| The workflow leverages GPT-4o-mini for generating trendy, concise TikTok video ideas in JSON form. | OpenAI GPT-4o-mini model usage                                 |
| Video resolution is set to 540x960 to fit TikTok’s preferred vertical video format.                 | TikTok video format best practice                              |
| Polling interval is fixed at 10 seconds to avoid frequent API calls and allow video generation time.| Workflow performance and API rate limiting considerations     |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.