Generate 360° Virtual Try-on Videos for Clothing with Kling API (unofficial)

https://n8nworkflows.xyz/workflows/generate-360--virtual-try-on-videos-for-clothing-with-kling-api--unofficial--3411


# Generate 360° Virtual Try-on Videos for Clothing with Kling API (unofficial)

### 1. Workflow Overview

This workflow automates the generation of 360° virtual try-on videos for clothing using the unofficial Kling API provided by PiAPI. It is designed primarily for e-commerce platforms, fashion brands, content creators, and influencers who want to visualize garments on models in an immersive, rotating video format. The workflow accepts image URLs of a model and clothing items, submits these to the Kling API for AI-based virtual try-on image generation, then triggers video generation based on the virtual try-on results, and finally retrieves the completed video URL.

The workflow is logically divided into these blocks:

- **1.1 Input Initialization:** Manual trigger and preset parameter setup including API key and image URLs.
- **1.2 Virtual Try-On Image Generation:** Submitting the model and clothing images to the Kling API to generate a virtual try-on image.
- **1.3 Image Generation Status Polling:** Waiting and checking for the completion of the virtual try-on image generation task.
- **1.4 Video Generation Request:** Once the image is ready, sending a request to generate a 360° video based on that image.
- **1.5 Video Generation Status Polling:** Waiting and checking for the video generation task to complete.
- **1.6 Final Output Retrieval:** Extracting and outputting the final video URL for use or download.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** This block starts the workflow manually and sets essential parameters such as the PiAPI X-API-Key and image URLs for the model and clothing items.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Preset Parameters  
  - Sticky Note (introductory information)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow for testing or execution.  
    - Configuration: No parameters; simply triggers downstream nodes.  
    - Inputs: None  
    - Outputs: Connects to Preset Parameters node.  
    - Edge Cases: None; manual trigger.

  - **Preset Parameters**  
    - Type: Set  
    - Role: Holds static or user-provided parameters such as the API key and image URLs.  
    - Configuration: Raw JSON with keys:  
      - `x-api-key`: User’s PiAPI API key (required for authentication).  
      - `model_input`: URL of the model image.  
      - `dress_input`: URL of dress image (optional).  
      - `upper_input` and `lower_input`: URLs for upper and lower clothing items (optional).  
      - `generate_video_prompt`: Text prompt guiding video generation, defaulting to "Walk on the catwalk, turn around, and finally stand still and pose".  
    - Inputs: From manual trigger.  
    - Outputs: To Kling Virtual Try-On Task node.  
    - Edge Cases: Missing or invalid API key or image URLs will cause API request failures.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides a high-level description of the workflow purpose and target users.  
    - Inputs/Outputs: None.

---

#### 2.2 Virtual Try-On Image Generation

- **Overview:** Sends a POST request to the Kling API to initiate the AI try-on task using the provided images and parameters.
- **Nodes Involved:**  
  - Kling Virtual Try-On Task

- **Node Details:**

  - **Kling Virtual Try-On Task**  
    - Type: HTTP Request  
    - Role: Calls the Kling API endpoint `/api/v1/task` with task type `ai_try_on` to generate a virtual try-on image.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.piapi.ai/api/v1/task`  
      - Headers: `x-api-key` from preset parameters for authentication.  
      - Body (JSON): Includes `model_input`, `dress_input`, `upper_input`, `lower_input`, and `batch_size`=1. Uses expressions to inject values from input JSON.  
    - Inputs: From Preset Parameters node.  
    - Outputs: Task creation response JSON, including a `task_id` and status.  
    - Edge Cases:  
      - API key invalid or missing → 401 Unauthorized.  
      - Invalid or missing image URLs → API error or failed task.  
      - Network issues or timeouts.  
    - Version: HTTP Request node version 4.2.

---

#### 2.3 Image Generation Status Polling

- **Overview:** Waits for the virtual try-on image generation to complete by polling the task status repeatedly.
- **Nodes Involved:**  
  - Wait for Image Generation  
  - Get Kling Virtual Try-On Task  
  - Check Data Status  
  - Switch

- **Node Details:**

  - **Wait for Image Generation**  
    - Type: Wait  
    - Role: Pauses workflow execution until an external webhook triggers continuation, effectively polling asynchronously.  
    - Configuration: Uses a webhook ID to resume when triggered externally.  
    - Inputs: From Kling Virtual Try-On Task node.  
    - Outputs: To Get Kling Virtual Try-On Task node.  
    - Edge Cases: Webhook not triggered → workflow stalls.

  - **Get Kling Virtual Try-On Task**  
    - Type: HTTP Request  
    - Role: Retrieves the current status of the virtual try-on task using the `task_id`.  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
      - Headers: `x-api-key` from Preset Parameters.  
    - Inputs: From Wait for Image Generation node.  
    - Outputs: Task status JSON.  
    - Edge Cases: API errors, invalid task_id, or network issues.

  - **Check Data Status**  
    - Type: If  
    - Role: Checks if the task status is either `completed` or `failed`.  
    - Configuration: Logical OR condition on `$json.data.status` equals `completed` or `failed`.  
    - Inputs: From Get Kling Virtual Try-On Task.  
    - Outputs:  
      - If true: To Switch node.  
      - If false: Loops back to Wait for Image Generation node to continue polling.  
    - Edge Cases: Unexpected status values.

  - **Switch**  
    - Type: Switch  
    - Role: Routes workflow based on whether the image generation was successful (`completed`) or not.  
    - Configuration: Checks if `$json.data.status` equals `completed`.  
    - Inputs: From Check Data Status node.  
    - Outputs:  
      - If completed: To Generate kling video node.  
      - Else: (No explicit else branch; workflow ends or errors.)  
    - Edge Cases: No handling for failed status explicitly.

---

#### 2.4 Video Generation Request

- **Overview:** Initiates the video generation task by sending the virtual try-on image URL and a prompt to the Kling API.
- **Nodes Involved:**  
  - Generate kling video

- **Node Details:**

  - **Generate kling video**  
    - Type: HTTP Request  
    - Role: Sends a POST request to create a video generation task with the virtual try-on image.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.piapi.ai/api/v1/task`  
      - Headers: `x-api-key` from Preset Parameters.  
      - Body (JSON):  
        - `model`: "kling"  
        - `task_type`: "video_generation"  
        - `input`: includes `version` "1.6", `image_url` from the previous image generation output, and `prompt` from Preset Parameters.  
    - Inputs: From Switch node (on image generation completion).  
    - Outputs: Task creation response JSON with `task_id` for video generation.  
    - Edge Cases: API errors, invalid image URL, prompt issues, auth failures.

---

#### 2.5 Video Generation Status Polling

- **Overview:** Similar to image polling, this block waits for the video generation task to complete by polling the task status.
- **Nodes Involved:**  
  - Wait for Video Generation  
  - Get Kling Video Task  
  - Get Video Data Status  
  - Check Video Data Status

- **Node Details:**

  - **Wait for Video Generation**  
    - Type: Wait  
    - Role: Pauses workflow until webhook triggers continuation, enabling asynchronous polling.  
    - Configuration: Uses the same webhook ID as image wait node.  
    - Inputs: From Generate kling video node or Get Video Data Status node (if not completed).  
    - Outputs: To Get Kling Video Task node.  
    - Edge Cases: Webhook not triggered → workflow stalls.

  - **Get Kling Video Task**  
    - Type: HTTP Request  
    - Role: Retrieves current status of the video generation task using `task_id`.  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
      - Headers: `x-api-key` from Preset Parameters.  
    - Inputs: From Wait for Video Generation node.  
    - Outputs: Task status JSON.  
    - Edge Cases: API errors, invalid task_id.

  - **Get Video Data Status**  
    - Type: If  
    - Role: Checks if video generation status is `completed` or `failed`.  
    - Configuration: Logical OR condition on `$json.data.status` equals `completed` or `failed`.  
    - Inputs: From Get Kling Video Task node.  
    - Outputs:  
      - If true: To Check Video Data Status node.  
      - If false: Loops back to Wait for Video Generation node to continue polling.  
    - Edge Cases: Unexpected status values.

  - **Check Video Data Status**  
    - Type: Switch  
    - Role: Routes workflow based on whether video generation succeeded (`completed`).  
    - Configuration: Checks if `$json.data.status` equals `completed`.  
    - Inputs: From Get Video Data Status node.  
    - Outputs:  
      - If completed: To Get Final Video URL node.  
      - Else: (No explicit else branch; workflow ends or errors.)  
    - Edge Cases: No explicit handling for failure.

---

#### 2.6 Final Output Retrieval

- **Overview:** Extracts the final video URL from the completed video generation task output and formats it for downstream use.
- **Nodes Involved:**  
  - Get Final Video URL  
  - Sticky Note2 (final instructions)

- **Node Details:**

  - **Get Final Video URL**  
    - Type: Set  
    - Role: Extracts and outputs the `video_url` from the API response JSON.  
    - Configuration: Raw JSON output with expression:  
      ```json
      {
        "video_url": "{{ $json.data.output.video_url }}"
      }
      ```  
    - Inputs: From Check Video Data Status node.  
    - Outputs: Final video URL JSON.  
    - Edge Cases: Missing or malformed `video_url` field in response.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Provides a note about waiting for generation and retrieving the final video URL.  
    - Inputs/Outputs: None.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                              | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                              |
|---------------------------|--------------------|----------------------------------------------|----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Entry point to start workflow manually       | None                             | Preset Parameters                 | ## Generate 360° Virtual Try-on Videos for Clothing with Kling API (unofficial) This tool is designed for e-commerce platforms, fashion brands, content creators, and content influencers. By uploading model and clothing images and linking your PiAPI account, you can swiftly generate a realistic video of the model sporting the outfit with a 360° turn, offering an immersive viewing experience. |
| Preset Parameters         | Set                | Holds API key and input image URLs            | When clicking ‘Test workflow’    | Kling Virtual Try-On Task         |                                                                                                        |
| Kling Virtual Try-On Task | HTTP Request       | Submits virtual try-on image generation task | Preset Parameters                | Wait for Image Generation         |                                                                                                        |
| Wait for Image Generation | Wait               | Pauses workflow until webhook triggers        | Kling Virtual Try-On Task        | Get Kling Virtual Try-On Task     | ## Generate Virtual Try-on Image Upload model url, users have two solutions to upload clothing url: 1. Upload `dress_input` 2. Upload 'upper_input` and 'lower_input` |
| Get Kling Virtual Try-On Task | HTTP Request       | Polls virtual try-on task status               | Wait for Image Generation        | Check Data Status                 |                                                                                                        |
| Check Data Status         | If                 | Checks if virtual try-on task is completed or failed | Get Kling Virtual Try-On Task    | Switch (if completed), Wait for Image Generation (if not) |                                                                                                        |
| Switch                   | Switch             | Routes workflow based on virtual try-on completion | Check Data Status                | Generate kling video              |                                                                                                        |
| Generate kling video     | HTTP Request       | Submits video generation task                   | Switch                         | Wait for Video Generation         |                                                                                                        |
| Wait for Video Generation | Wait               | Pauses workflow until webhook triggers          | Generate kling video, Get Video Data Status | Get Kling Video Task             | ## Generate Final Video Wait for generation and get the output url in the final node.                   |
| Get Kling Video Task     | HTTP Request       | Polls video generation task status               | Wait for Video Generation        | Get Video Data Status             |                                                                                                        |
| Get Video Data Status    | If                 | Checks if video generation task is completed or failed | Get Kling Video Task             | Check Video Data Status, Wait for Video Generation |                                                                                                        |
| Check Video Data Status  | Switch             | Routes workflow based on video generation completion | Get Video Data Status            | Get Final Video URL               |                                                                                                        |
| Get Final Video URL      | Set                | Extracts and outputs final video URL             | Check Video Data Status          | None                            |                                                                                                        |
| Sticky Note              | Sticky Note        | Introductory note about workflow purpose         | None                             | None                            | ## Generate 360° Virtual Try-on Videos for Clothing with Kling API (unofficial) This tool is designed for e-commerce platforms, fashion brands, content creators, and content influencers. By uploading model and clothing images and linking your PiAPI account, you can swiftly generate a realistic video of the model sporting the outfit with a 360° turn, offering an immersive viewing experience. |
| Sticky Note1             | Sticky Note        | Notes on virtual try-on image inputs             | None                             | None                            | ## Generate Virtual Try-on Image Upload model url, users have two solutions to upload clothing url: 1. Upload `dress_input` 2. Upload 'upper_input` and 'lower_input` |
| Sticky Note2             | Sticky Note        | Notes on final video generation and output        | None                             | None                            | ## Generate Final Video Wait for generation and get the output url in the final node.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Create a Set Node for Preset Parameters**  
   - Name: `Preset Parameters`  
   - Mode: Raw JSON  
   - JSON Content:  
     ```json
     {
       "x-api-key": "",
       "model_input": "",
       "dress_input": "",
       "upper_input": "",
       "lower_input": "",
       "generate_video_prompt": "Walk on the catwalk, turn around, and finally stand still and pose"
     }
     ```  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node for Virtual Try-On Task**  
   - Name: `Kling Virtual Try-On Task`  
   - Method: POST  
   - URL: `https://api.piapi.ai/api/v1/task`  
   - Authentication: None (use header parameter)  
   - Headers:  
     - `x-api-key`: Expression `={{ $json['x-api-key'] }}` from Preset Parameters  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "model": "kling",
       "task_type": "ai_try_on",
       "input": {
         "model_input": "{{ $json.model_input }}",
         "dress_input": "{{ $json.dress_input }}",
         "upper_input": "{{ $json.upper_input }}",
         "lower_input": "{{ $json.lower_input }}",
         "batch_size": 1
       }
     }
     ```  
   - Connect Preset Parameters output to this node.

4. **Create Wait Node for Image Generation**  
   - Name: `Wait for Image Generation`  
   - Configure with a webhook ID (auto-generated or custom) to pause workflow until externally triggered.  
   - Connect output of Kling Virtual Try-On Task to this node.

5. **Create HTTP Request Node to Get Virtual Try-On Task Status**  
   - Name: `Get Kling Virtual Try-On Task`  
   - Method: GET  
   - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}` (use expression to inject task_id)  
   - Headers:  
     - `x-api-key`: Expression `={{ $('Preset Parameters').item.json['x-api-key'] }}`  
   - Connect output of Wait for Image Generation to this node.

6. **Create If Node to Check Image Task Status**  
   - Name: `Check Data Status`  
   - Condition: `$json.data.status` equals `completed` OR equals `failed`  
   - Connect output of Get Kling Virtual Try-On Task to this node.

7. **Connect True Output of If Node to a Switch Node**  
   - Name: `Switch`  
   - Condition: `$json.data.status` equals `completed`  
   - Connect True output of Check Data Status to Switch node.

8. **Connect False Output of If Node Back to Wait for Image Generation**  
   - To continue polling until completion or failure.

9. **Create HTTP Request Node for Video Generation**  
   - Name: `Generate kling video`  
   - Method: POST  
   - URL: `https://api.piapi.ai/api/v1/task`  
   - Headers:  
     - `x-api-key`: Expression `={{ $('Preset Parameters').item.json['x-api-key'] }}`  
   - Body (JSON):  
     ```json
     {
       "model": "kling",
       "task_type": "video_generation",
       "input": {
         "version": "1.6",
         "image_url": "{{ $json.data.output.works[0].image.resource }}",
         "prompt": "{{ $('Preset Parameters').item.json.generate_video_prompt }}"
       }
     }
     ```  
   - Connect output of Switch node (completed branch) to this node.

10. **Create Wait Node for Video Generation**  
    - Name: `Wait for Video Generation`  
    - Use the same webhook ID as the image wait node to pause until webhook triggers.  
    - Connect output of Generate kling video node to this node.

11. **Create HTTP Request Node to Get Video Task Status**  
    - Name: `Get Kling Video Task`  
    - Method: GET  
    - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
    - Headers:  
      - `x-api-key`: Expression `={{ $('Preset Parameters').item.json['x-api-key'] }}`  
    - Connect output of Wait for Video Generation to this node.

12. **Create If Node to Check Video Task Status**  
    - Name: `Get Video Data Status`  
    - Condition: `$json.data.status` equals `completed` OR equals `failed`  
    - Connect output of Get Kling Video Task to this node.

13. **Connect True Output of Video Status If Node to Switch Node**  
    - Name: `Check Video Data Status`  
    - Condition: `$json.data.status` equals `completed`  
    - Connect True output of Get Video Data Status to this node.

14. **Connect False Output of Video Status If Node Back to Wait for Video Generation**  
    - To continue polling.

15. **Create Set Node to Extract Final Video URL**  
    - Name: `Get Final Video URL`  
    - Mode: Raw JSON  
    - JSON Content:  
      ```json
      {
        "video_url": "{{ $json.data.output.video_url }}"
      }
      ```  
    - Connect output of Check Video Data Status node (completed branch) to this node.

16. **Optional: Add Sticky Notes**  
    - Add notes describing workflow purpose, input instructions, and output instructions as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| For basic settings of virtual try-on, check the official API documentation to get best practice.                                                                                                                                              | https://piapi.ai/docs/kling-api/virtual-try-on-api                                               |
| This workflow enables efficient testing of garment-on-model presentations, reducing business model validation costs.                                                                                                                        | Workflow description                                                                              |
| Input images example: Model and dress images are provided in the workflow description for testing.                                                                                                                                             | Model: https://i.ibb.co/gL5ZrjYH/model.jpg, Dress: https://i.ibb.co/qL7RsS86/dress.jpg            |
| Output video example demonstrates a rotating runway-style view of the model wearing the clothing.                                                                                                                                              | https://static.piapi.ai/n8n-instruction/virtual-try-on/example1.mp4                              |
| Users can provide either a single dress image (`dress_input`) or separate upper and lower clothing images (`upper_input` and `lower_input`) for flexibility.                                                                                 | Sticky Note1 content                                                                             |
| The workflow uses webhook-based wait nodes for asynchronous polling, requiring the external webhook trigger to resume execution. Ensure webhook URLs are accessible and properly triggered by the API or external process.                     | Wait nodes configuration                                                                         |

---

This documentation provides a detailed, structured reference to understand, reproduce, and maintain the "Generate 360° Virtual Try-on Videos for Clothing with Kling API" workflow in n8n. It covers all nodes, their configurations, data flow, and potential failure points to assist advanced users and AI agents alike.