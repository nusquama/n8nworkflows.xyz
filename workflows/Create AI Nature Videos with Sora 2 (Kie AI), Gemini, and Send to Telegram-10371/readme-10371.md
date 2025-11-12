Create AI Nature Videos with Sora 2 (Kie AI), Gemini, and Send to Telegram

https://n8nworkflows.xyz/workflows/create-ai-nature-videos-with-sora-2--kie-ai---gemini--and-send-to-telegram-10371


# Create AI Nature Videos with Sora 2 (Kie AI), Gemini, and Send to Telegram

---

### 1. Workflow Overview

This workflow automates the creation and delivery of 10-second AI-generated nature videos to a Telegram chat. It is designed for content creators, marketing teams, educators, or personal use cases needing fresh, high-quality nature footage without manual filming or editing. The workflow executes on a scheduled trigger and includes these logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow on a defined interval.
- **1.2 AI Prompt Generation:** Uses Google Gemini AI to generate a detailed, timestamped video prompt specifically tailored for 10-second nature stock footage.
- **1.3 Video Generation Submission:** Sends the generated prompt to the Kie AI API (Sora 2 text-to-video model) to asynchronously create the video.
- **1.4 Video Completion Await & Polling:** Waits for completion notification via webhook or polls the Kie AI API for task status.
- **1.5 Video Download:** Retrieves the completed video file from the Kie AI API.
- **1.6 Telegram Delivery:** Sends the downloaded video to a specified Telegram chat.

Supporting sticky notes provide contextual information, configuration tips, and troubleshooting guidelines.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Triggers the entire workflow periodically based on a configurable schedule interval.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger node (base)  
    - Configuration: Triggers at a specified interval; default is every interval (can be customized)  
    - Inputs: None (start node)  
    - Outputs: Connected to "Generate Video Prompt" node  
    - Edge Cases: Misconfiguration of interval may cause unexpected triggering frequency  
    - Version: 1.2

#### 2.2 AI Prompt Generation

- **Overview:**  
  Generates a professional, detailed video prompt for a 10-second nature stock footage video. It uses Google Gemini AI via LangChain to produce a structured prompt including mood, style, timestamped shots, and natural elements.

- **Nodes Involved:**  
  - Gemini Prompt Model  
  - Generate Video Prompt  
  - Note: Prompt Generation1 (Sticky Note)

- **Node Details:**  
  - **Gemini Prompt Model**  
    - Type: LangChain Google Gemini Chat model node  
    - Configuration: Uses Google Gemini API credentials to process AI prompts  
    - Inputs: None explicitly, but connected as AI language model for "Generate Video Prompt"  
    - Outputs: Provides AI model output to "Generate Video Prompt"  
    - Credentials: Google Gemini API (requires API key with Gemini 1.5 Flash or Pro access)  
    - Edge Cases: API quota limits, network errors, or malformed prompt inputs may fail generation  
    - Version: 1  
  - **Generate Video Prompt**  
    - Type: LangChain Chain LLM node  
    - Configuration: System prompt instructs AI to output a single, detailed, timestamped prompt for 10s nature footage; no extra commentary  
    - Key Expressions: Uses an elaborate text prompt specifying mood, style, shot descriptions, sound design, pacing, and constraints  
    - Inputs: Triggered by "Schedule Trigger" and uses "Gemini Prompt Model" as AI language model  
    - Outputs: JSON with generated prompt text sent to "Submit Video Generation"  
    - Edge Cases: AI text generation failures, incomplete prompt, or unexpected format violations  
    - Version: 1.7  
  - **Note: Prompt Generation1**  
    - Sticky Note explaining the purpose and customization of the prompt generation block  

#### 2.3 Video Generation Submission

- **Overview:**  
  Submits the generated prompt to Kie AI's video generation API (Sora 2 model), requesting a 10 frame, portrait aspect ratio video. The API call is asynchronous, and the workflow is designed to resume upon webhook callback.

- **Nodes Involved:**  
  - Submit Video Generation  
  - Note: API Submission (Sticky Note)

- **Node Details:**  
  - **Submit Video Generation**  
    - Type: HTTP Request node  
    - Configuration:  
      - POST to `https://api.kie.ai/api/v1/jobs/createTask`  
      - JSON body includes:  
        - model: "sora-2-text-to-video"  
        - callBackUrl: workflow resume URL for webhook callback  
        - input: prompt (cleaned from generated text), aspect ratio "portrait", 10 frames, watermark removal enabled  
      - Authentication: HTTP Header Auth using Kie AI API key  
    - Inputs: Receives prompt from "Generate Video Prompt"  
    - Outputs: Passes API response to "Await Video Completion"  
    - Edge Cases: Authentication errors, API rate limits, malformed prompt causing rejection  
    - Version: 4.3  
  - **Note: API Submission**  
    - Describes that this node posts prompt to Kie AI API for asynchronous video creation and sets portrait aspect ratio and 10 frames  
    - Notes that webhook resumes workflow on completion  

#### 2.4 Video Completion Await & Polling

- **Overview:**  
  Waits for Kie AI API to notify that the video task is complete via webhook; if webhook fails, fallback polling checks the status periodically.

- **Nodes Involved:**  
  - Await Video Completion  
  - Poll Video Status  
  - Note: Webhook Wait (Sticky Note)

- **Node Details:**  
  - **Await Video Completion**  
    - Type: Wait node with webhook resume  
    - Configuration:  
      - Pauses workflow execution until HTTP POST webhook callback is received from Kie AI API  
      - Requires publicly accessible n8n URL for webhook listening  
    - Inputs: Receives API response from "Submit Video Generation"  
    - Outputs: Passes webhook data to "Poll Video Status"  
    - Edge Cases: Webhook callback not received due to network or firewall issues  
    - Version: 1.1  
  - **Poll Video Status**  
    - Type: HTTP Request node  
    - Configuration:  
      - GET request to `https://api.kie.ai/api/v1/jobs/recordInfo` with query parameter `taskId` extracted from webhook data  
      - Uses same HTTP Header Auth credentials  
    - Inputs: Receives webhook callback data from "Await Video Completion"  
    - Outputs: Passes video info JSON to "Download Generated Video"  
    - Edge Cases: Task may be pending or failed; polling interval and retries must be managed externally or in extended workflows  
    - Version: 4.3  
  - **Note: Webhook Wait**  
    - Explains the wait node pauses execution until webhook notifies readiness and fallback polling is available  

#### 2.5 Video Download

- **Overview:**  
  Downloads the generated video file from the URL provided by the Kie AI API once the video generation is complete.

- **Nodes Involved:**  
  - Download Generated Video

- **Node Details:**  
  - **Download Generated Video**  
    - Type: HTTP Request node  
    - Configuration:  
      - GET request to the first URL in `resultWaterMarkUrls` parsed from the JSON response of the previous node  
      - No authentication required for the download URL (assumed public or tokenized)  
    - Inputs: Receives video info JSON from "Poll Video Status"  
    - Outputs: Pass binary video data to "Deliver Video to Telegram"  
    - Edge Cases: URL might expire, be invalid, or download might fail due to network issues  
    - Version: 4.3  

#### 2.6 Telegram Delivery

- **Overview:**  
  Sends the downloaded video file as a video message to a specified Telegram chat ID using a Telegram Bot.

- **Nodes Involved:**  
  - Deliver Video to Telegram  
  - Note: Telegram Delivery (Sticky Note)

- **Node Details:**  
  - **Deliver Video to Telegram**  
    - Type: Telegram node  
    - Configuration:  
      - Operation: sendVideo  
      - Chat ID: must be replaced with actual Telegram chat ID (personal or group)  
      - Uses binary data input to send video file  
      - Credentials: Telegram Bot API token  
    - Inputs: Receives binary video file from "Download Generated Video"  
    - Outputs: Final node (end of workflow)  
    - Edge Cases: Invalid/missing chat ID, bot permissions, or Telegram API outages  
    - Version: 1.2  
  - **Note: Telegram Delivery**  
    - Explains purpose and requirement to replace chat ID with actual target chat or group  

---

### 3. Summary Table

| Node Name               | Node Type                                   | Functional Role                      | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                   |
|-------------------------|---------------------------------------------|------------------------------------|---------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger (base)                      | Starts workflow on schedule         | None                      | Generate Video Prompt      |                                                                                              |
| Gemini Prompt Model     | LangChain Google Gemini Chat Model           | AI model for prompt generation      | None (AI model for prompt) | Generate Video Prompt (AI) |                                                                                              |
| Generate Video Prompt   | LangChain Chain LLM                          | Generates detailed video prompt     | Schedule Trigger, Gemini  | Submit Video Generation    | ## 🤖 Generate Video Prompt: Crafts detailed, timestamped prompts for 10s nature videos using Gemini. |
| Submit Video Generation | HTTP Request                                | Submits prompt to Kie AI API        | Generate Video Prompt      | Await Video Completion     | ## 🚀 Submit Video Generation: Posts prompt to Kie AI API to start async video creation. Sets portrait aspect and 10 frames; webhook resumes on completion. |
| Await Video Completion  | Wait (with webhook resume)                   | Waits for Kie AI webhook callback   | Submit Video Generation    | Poll Video Status          | ## ⏳ Await Video Completion: Pauses execution until Kie AI webhook notifies task readiness. Requires public n8n URL; fallback to polling if webhooks fail. |
| Poll Video Status       | HTTP Request                                | Polls Kie AI API for video status   | Await Video Completion     | Download Generated Video   |                                                                                              |
| Download Generated Video| HTTP Request                                | Downloads completed video file       | Poll Video Status          | Deliver Video to Telegram  |                                                                                              |
| Deliver Video to Telegram| Telegram                                    | Sends video to Telegram chat        | Download Generated Video   | None                      | ## 📥 Deliver Video to Telegram: Sends downloaded video file to specified chat. Replace [Your Chat ID] with actual ID; supports groups or personal bots. |
| Note: Prompt Generation1| Sticky Note                                 | Explains prompt generation block    | None                      | None                      | ## 🤖 Generate Video Prompt: Crafts detailed, timestamped prompts for 10s nature videos using Gemini. |
| Note: API Submission    | Sticky Note                                 | Explains API submission node        | None                      | None                      | ## 🚀 Submit Video Generation: Posts prompt to Kie AI API to start async video creation. Sets portrait aspect and 10 frames; webhook resumes on completion. |
| Note: Webhook Wait      | Sticky Note                                 | Explains wait node with webhook     | None                      | None                      | ## ⏳ Await Video Completion: Pauses execution until Kie AI webhook notifies task readiness. Requires public n8n URL; fallback to polling if webhooks fail. |
| Note: Telegram Delivery | Sticky Note                                 | Explains Telegram delivery node     | None                      | None                      | ## 📥 Deliver Video to Telegram: Sends downloaded video file to specified chat. Replace [Your Chat ID] with actual ID; supports groups or personal bots. |
| Overview Note7          | Sticky Note                                 | Full workflow overview and setup    | None                      | None                      | # Automated 10s Sora 2 Video Bot for Telegram: Detailed description, prerequisites, credentials setup, configuration, use cases, troubleshooting. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval as needed (e.g., daily at 9 AM)  
   - No credentials required

2. **Set Up Google Gemini API Credential**  
   - Obtain API key from [aistudio.google.com](https://aistudio.google.com) with Gemini 1.5 Flash or Pro access  
   - Add as "Google Gemini API" credential in n8n

3. **Create Gemini Prompt Model Node**  
   - Type: LangChain Google Gemini Chat Model  
   - Assign Google Gemini API credential  
   - No additional parameters needed  
   - Connect no input (used as AI model for prompt generation)

4. **Create Generate Video Prompt Node**  
   - Type: LangChain Chain LLM  
   - Paste the detailed system prompt text (see section 2.2) instructing AI to generate a 10-second nature video prompt only  
   - Use Gemini Prompt Model node as AI language model input  
   - Connect Schedule Trigger output to this node's input  
   - No credentials needed here if AI model linked

5. **Set Up Kie AI API Credential**  
   - Register at [kie.ai](https://kie.ai) and generate API key with video generation permissions  
   - Add as "HTTP Header Auth" credential in n8n with header: Authorization, value: Bearer [Your API Key]

6. **Create Submit Video Generation Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/jobs/createTask`  
   - Authentication: Use Kie AI HTTP Header Auth credential  
   - Body Type: JSON  
   - Body Parameters:  
     - model: "sora-2-text-to-video"  
     - callBackUrl: Use n8n’s execution resume URL expression `{{$execution.resumeUrl}}`  
     - input:  
       - prompt: cleaned prompt text from "Generate Video Prompt" node (remove newlines and stringify)  
       - aspect_ratio: "portrait"  
       - n_frames: "10"  
       - remove_watermark: true  
   - Connect output of "Generate Video Prompt" to this node

7. **Create Await Video Completion Node**  
   - Type: Wait node  
   - Set to resume on webhook POST  
   - Configure webhook path (auto-generated or manually set)  
   - Connect output of "Submit Video Generation" to this node

8. **Create Poll Video Status Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`  
   - Authentication: Kie AI HTTP Header Auth credential  
   - Add query parameter `taskId` with expression to extract from webhook body: `{{$json.body.data.taskId}}`  
   - Connect output of "Await Video Completion" to this node

9. **Create Download Generated Video Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression to parse first URL from `resultWaterMarkUrls` in the JSON response of "Poll Video Status" node:  
     `{{$json.data.resultJson.parseJson().resultWaterMarkUrls.first()}}`  
   - No authentication required  
   - Connect output of "Poll Video Status" to this node

10. **Set Up Telegram API Credential**  
    - Create Telegram Bot via @BotFather and obtain API token  
    - Add as "Telegram API" credential in n8n

11. **Create Deliver Video to Telegram Node**  
    - Type: Telegram node  
    - Operation: sendVideo  
    - Chat ID: Replace placeholder with actual Telegram chat ID (user or group)  
    - Enable sending binary data (connect binary video from previous node)  
    - Use Telegram API credential  
    - Connect output of "Download Generated Video" to this node

12. **Link Nodes Sequentially**  
    - Schedule Trigger → Generate Video Prompt  
    - Generate Video Prompt → Submit Video Generation  
    - Submit Video Generation → Await Video Completion  
    - Await Video Completion → Poll Video Status  
    - Poll Video Status → Download Generated Video  
    - Download Generated Video → Deliver Video to Telegram

13. **Test and Activate Workflow**  
    - Ensure public n8n URL for webhook callbacks  
    - Test run and verify video generation and Telegram delivery  
    - Adjust Schedule Trigger timing as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates creation and delivery of 10-second nature-themed AI videos using Kie AI's Sora 2 model and Google Gemini for prompt crafting. Useful for content creators, marketers, educators, and personal productivity.                                                                                                                | Overview Note7 sticky note content                                                                                                                                |
| Google Gemini API key setup instructions: Create API key on aistudio.google.com with access to Gemini 1.5 Flash or Pro models.                                                                                                                                                                                                                    | https://aistudio.google.com                                                                                                                                         |
| Kie AI API key setup: Signup on kie.ai, generate API key with video generation permissions for sora-2-text-to-video model.                                                                                                                                                                                                                       | https://kie.ai                                                                                                                                                      |
| Telegram Bot setup: Use @BotFather to create bot, obtain API token, and get target chat ID via @userinfobot.                                                                                                                                                                                                                                      | Telegram Bot API                                                                                                                                                |
| Troubleshooting tips: Check API usage limits if video generation fails; verify chat ID and bot token for Telegram errors; ensure n8n URL is public for webhook callbacks; test fallback polling if webhook unreliable.                                                                                                                               | Overview Note7 sticky note content                                                                                                                                |
| Prompt customization recommendations: Modify system prompt text in "Generate Video Prompt" node to tailor themes, seasons, or stylistic elements to your needs.                                                                                                                                                                                    | Note: Prompt Generation1 sticky note                                                                                                                              |
| Webhook callback requires n8n instance accessible from internet; use services like ngrok for local testing. Polling node provides fallback to check video status if webhook not triggered.                                                                                                                                                        | Note: Webhook Wait sticky note                                                                                                                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---