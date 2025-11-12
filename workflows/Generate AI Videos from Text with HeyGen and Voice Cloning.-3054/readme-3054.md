Generate AI Videos from Text with HeyGen and Voice Cloning.

https://n8nworkflows.xyz/workflows/generate-ai-videos-from-text-with-heygen-and-voice-cloning--3054


# Generate AI Videos from Text with HeyGen and Voice Cloning.

### 1. Workflow Overview

This workflow automates the generation of AI-powered videos using the HeyGen platform within n8n. It is designed for content creators, marketers, and educators who want to create personalized, realistic AI avatar videos from text scripts without manual video editing.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger to start the workflow and set up video parameters (text, avatar ID, voice ID).
- **1.2 Video Creation Request:** Sends a request to HeyGen API to generate a video based on the input parameters.
- **1.3 Video Processing Wait Loop:** Waits and polls HeyGen API to check the status of the video generation until completion.
- **1.4 Output Preparation:** Once the video is ready, extracts and outputs the direct video URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block starts the workflow manually and sets the necessary input parameters for video generation, including the text script, avatar ID, and voice ID.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Config (Set node)  
  - Sticky Note2 (Documentation)  
  - Sticky Note3 (Documentation)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow for testing or execution.  
    - Configuration: No parameters; triggers workflow on manual action.  
    - Inputs: None  
    - Outputs: Connects to Config node  
    - Edge Cases: None typical; user must manually trigger.

  - **Config**  
    - Type: Set  
    - Role: Defines and assigns the input parameters for the video generation:  
      - `avatar_id`: The ID of the chosen AI avatar (default example: "7895d2d9f4f9453899e1d80e5accb6be")  
      - `voice_id`: The ID of the selected voice (default example: "PBgwoAVFZIC0UB6sU914")  
      - `text`: The script text to be spoken by the avatar (example text provided)  
    - Configuration: Static values set for demonstration; can be replaced with dynamic inputs.  
    - Inputs: From Manual Trigger  
    - Outputs: To Create Video node  
    - Edge Cases: If any parameter is missing or invalid, the API request may fail.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Provides detailed documentation on workflow purpose, setup instructions, and HeyGen API key configuration.  
    - Content includes:  
      - Overview of HeyGen integration  
      - Steps to create HeyGen account and API key  
      - Instructions to create n8n credentials with custom auth header `X-Api-Key`  
      - How to select avatar and voice IDs  
    - Inputs/Outputs: None (documentation only)

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Reminds users to provide the three key video details: Avatar ID, Voice ID, and Text.  
    - Inputs/Outputs: None (documentation only)

---

#### 2.2 Video Creation Request

- **Overview:**  
  Sends a POST request to HeyGen API to initiate video generation using the provided avatar, voice, and text parameters.

- **Nodes Involved:**  
  - Create Video (HTTP Request)  
  - Sticky Note1 (Documentation)

- **Node Details:**

  - **Create Video**  
    - Type: HTTP Request  
    - Role: Calls HeyGen’s `/v2/video/generate` endpoint to create a new AI video.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.heygen.com/v2/video/generate`  
      - Body: JSON containing:  
        - `video_inputs`: array with one object specifying avatar type, avatar_id, avatar_style ("normal"), voice type ("text"), input_text, voice_id, and speed (1)  
        - `caption`: true (enables captions)  
        - `dimension`: width 1080, height 1920 (portrait video)  
      - Authentication: Custom HTTP header auth with `X-Api-Key` credential  
    - Inputs: From Config node (parameters)  
    - Outputs: To Wait node  
    - Edge Cases:  
      - API key invalid or missing → authentication error  
      - Invalid avatar_id or voice_id → API error response  
      - Text too long or malformed → API rejection  
      - Network timeout or API downtime

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Indicates the following block checks video status after creation.  
    - Content: "## Check video status"  
    - Inputs/Outputs: None (documentation only)

---

#### 2.3 Video Processing Wait Loop

- **Overview:**  
  After requesting video creation, this block waits for a fixed interval, then polls HeyGen API for the video generation status repeatedly until the video is marked as completed.

- **Nodes Involved:**  
  - Wait (Wait node)  
  - Get Video Status (HTTP Request)  
  - is Completed (If node)

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for 10 seconds before polling video status again.  
    - Configuration: Wait time set to 10 seconds.  
    - Inputs: From Create Video node or from is Completed node (if video not completed)  
    - Outputs: To Get Video Status node  
    - Edge Cases:  
      - Excessive waiting if video generation is slow  
      - Workflow timeout if HeyGen delays too long

  - **Get Video Status**  
    - Type: HTTP Request  
    - Role: Calls HeyGen’s `/v1/video_status.get` endpoint to retrieve current status of the video generation.  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.heygen.com/v1/video_status.get`  
      - Query Parameter: `video_id` extracted dynamically from the Create Video node response (`{{$node["Create Video"].json.data.video_id}}`)  
      - Authentication: Custom HTTP header auth with `X-Api-Key` credential  
    - Inputs: From Wait node  
    - Outputs: To is Completed node  
    - Edge Cases:  
      - Invalid or expired video_id → API error  
      - Network or authentication failures  
      - Rate limiting by HeyGen API

  - **is Completed**  
    - Type: If  
    - Role: Checks if the video status returned by Get Video Status node equals `"completed"`.  
    - Configuration:  
      - Condition: `$json.data.status === "completed"`  
    - Inputs: From Get Video Status node  
    - Outputs:  
      - True branch: To Output node (video ready)  
      - False branch: Loops back to Wait node (continue polling)  
    - Edge Cases:  
      - Status other than "completed" or unexpected values  
      - Infinite loop if video never completes (no timeout implemented)

---

#### 2.4 Output Preparation

- **Overview:**  
  Once the video is completed, this block extracts the direct video URL from the API response and prepares it as output.

- **Nodes Involved:**  
  - Output (Set node)

- **Node Details:**

  - **Output**  
    - Type: Set  
    - Role: Extracts and assigns the `video_url` from the video status response to the output JSON.  
    - Configuration:  
      - Sets `data.video_url` to `{{$json.data.video_url}}` from the input data.  
    - Inputs: From is Completed node (true branch)  
    - Outputs: Final output of the workflow  
    - Edge Cases:  
      - Missing or null `video_url` if API response is malformed  
      - Downstream nodes or consumers must handle empty or invalid URLs

---

### 3. Summary Table

| Node Name                 | Node Type        | Functional Role                     | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                  |
|---------------------------|------------------|-----------------------------------|-----------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger   | Starts workflow manually           | None                        | Config                    |                                                                                                              |
| Config                    | Set              | Defines input parameters (avatar_id, voice_id, text) | When clicking ‘Test workflow’ | Create Video              |                                                                                                              |
| Create Video              | HTTP Request     | Sends video generation request to HeyGen API | Config                      | Wait                      | ## Check video status                                                                                        |
| Wait                      | Wait             | Pauses workflow before polling status | Create Video, is Completed (false branch) | Get Video Status          |                                                                                                              |
| Get Video Status          | HTTP Request     | Polls HeyGen API for video generation status | Wait                       | is Completed              |                                                                                                              |
| is Completed              | If               | Checks if video generation is complete | Get Video Status            | Output (true), Wait (false) |                                                                                                              |
| Output                    | Set              | Extracts and outputs final video URL | is Completed (true branch)  | None                      |                                                                                                              |
| Sticky Note1              | Sticky Note      | Documentation: "## Check video status" | None                        | None                      | ## Check video status                                                                                        |
| Sticky Note2              | Sticky Note      | Documentation: Workflow overview, setup, API key instructions | None                        | None                      | # Generate AI Videos with HeyGen in n8n ... [full content as in node details]                                |
| Sticky Note3              | Sticky Note      | Documentation: Reminder to provide Avatar ID, Voice ID, Text | None                        | None                      | # ☝️ Provide Video Details - Avatar ID, Voice ID, Text                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for Input Configuration**  
   - Name: `Config`  
   - Type: Set  
   - Add three string fields:  
     - `avatar_id` with a default value (e.g., `"7895d2d9f4f9453899e1d80e5accb6be"`)  
     - `voice_id` with a default value (e.g., `"PBgwoAVFZIC0UB6sU914"`)  
     - `text` with a sample script text or leave empty for dynamic input.

3. **Connect Manual Trigger → Config**

4. **Create HTTP Request Node to Generate Video**  
   - Name: `Create Video`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Authentication: Create new credentials of type **Custom Auth** with:  
     - Name: `X-Api-Key`  
     - Value: Your HeyGen API key  
   - Body Content Type: JSON  
   - Body (raw JSON with expressions):  
     ```json
     {
       "video_inputs": [
         {
           "character": {
             "type": "avatar",
             "avatar_id": "{{ $json.avatar_id }}",
             "avatar_style": "normal"
           },
           "voice": {
             "type": "text",
             "input_text": "{{ $json.text }}",
             "voice_id": "{{ $json.voice_id }}",
             "speed": 1
           }
         }
       ],
       "caption": true,
       "dimension": {
         "width": 1080,
         "height": 1920
       }
     }
     ```
   - Connect Config → Create Video

5. **Create Wait Node**  
   - Name: `Wait`  
   - Type: Wait  
   - Set wait time to 10 seconds.

6. **Connect Create Video → Wait**

7. **Create HTTP Request Node to Get Video Status**  
   - Name: `Get Video Status`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.heygen.com/v1/video_status.get`  
   - Authentication: Use the same Custom Auth credentials as Create Video node.  
   - Add Query Parameter:  
     - Name: `video_id`  
     - Value: `={{ $node["Create Video"].json.data.video_id }}`  
   - Connect Wait → Get Video Status

8. **Create If Node to Check Completion**  
   - Name: `is Completed`  
   - Type: If  
   - Condition: Check if `$json.data.status` equals `"completed"`  
   - Connect Get Video Status → is Completed

9. **Create Set Node for Output**  
   - Name: `Output`  
   - Type: Set  
   - Assign field: `data.video_url` = `={{ $json.data.video_url }}`  
   - Connect is Completed (true) → Output

10. **Connect is Completed (false) → Wait**  
    - This creates a polling loop until video status is "completed".

11. **Add Sticky Notes (Optional but Recommended)**  
    - Add notes with setup instructions, API key configuration, and reminders for input parameters as per Sticky Note2 and Sticky Note3 content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To use HeyGen API, you must purchase API credits.                                                                                       | Workflow Sticky Note2 content                                                                   |
| HeyGen platform: Create AI-powered videos with customizable avatars and voices.                                                          | https://www.heygen.com                                                                          |
| Setup instructions for HeyGen API key and n8n Custom Auth credentials: Use header name `X-Api-Key` with your HeyGen API key as value.  | Workflow Sticky Note2 content                                                                   |
| Example text script used in the workflow demonstrates creative storytelling with AI avatars and voices.                                  | Config node default value                                                                       |
| Video dimension is set to 1080x1920 (portrait mode), suitable for mobile and social media vertical videos.                              | Create Video node configuration                                                                 |
| Polling interval is set to 10 seconds; adjust as needed based on expected video generation time and API rate limits.                    | Wait node configuration                                                                         |
| No explicit timeout or maximum retry count is implemented for video status polling; consider adding for production workflows.           | General best practice note                                                                       |

---

This documentation provides a complete, structured reference to understand, reproduce, and maintain the "Generate AI Videos from Text with HeyGen and Voice Cloning" workflow in n8n.