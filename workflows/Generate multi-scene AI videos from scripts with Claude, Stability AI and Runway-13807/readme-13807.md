Generate multi-scene AI videos from scripts with Claude, Stability AI and Runway

https://n8nworkflows.xyz/workflows/generate-multi-scene-ai-videos-from-scripts-with-claude--stability-ai-and-runway-13807


# Generate multi-scene AI videos from scripts with Claude, Stability AI and Runway

# n8n Runway AI — Script to Video Generator Reference

### 1. Workflow Overview
This workflow is a comprehensive AI-driven video production pipeline. It automates the transition from a raw text script to a multi-scene video package. The process involves intelligent scene planning, consistent image generation for visual anchoring, video synthesis, and multi-channel delivery.

**Logical Blocks:**
*   **1.1 Intake & Validation:** Captures the script and configuration via Webhook, sets global variables/credentials, and ensures required data is present.
*   **1.2 AI Scene Planning:** Uses Anthropic’s Claude AI to transform the script into a structured JSON plan containing prompts for image and video generation.
*   **1.3 Image Generation Loop:** Iterates through scenes to generate high-resolution reference images via Stability AI.
*   **1.4 Video Generation & Polling:** Submits prompts and reference images to Runway Gen-3 Alpha, then enters a recursive polling loop until the video is rendered.
*   **1.5 Aggregation & Delivery:** Assembles all scene assets into a final package, logs the job to Google Sheets, sends notifications (Slack/Email), and returns the final payload.

---

### 2. Block-by-Block Analysis

#### 2.1 Stage 1 — Script Intake & Validation
*   **Overview:** Handles the initial entry point and global configuration.
*   **Nodes Involved:** `Receive Script`, `Set Credentials and Config`, `Validate Script`, `Return Validation Error`.
*   **Node Details:**
    *   **Receive Script (Webhook):** Listens for `POST` requests at `/runway-video`.
    *   **Set Credentials and Config:** Centralizes API keys (Anthropic, Runway, Stability, SendGrid, Slack, Google Sheets) and normalizes input fields (script, style, aspect ratio).
    *   **Validate Script (IF):** Checks if the script field is empty to prevent unnecessary API spend.
*   **Edge Cases:** Missing script triggers a 400 Bad Request via the `Return Validation Error` node.

#### 2.2 Stage 2 — Claude AI Scene Breakdown
*   **Overview:** Transforms text into a cinematic storyboard.
*   **Nodes Involved:** `Claude Scene Planner`, `Parse Scenes JSON`.
*   **Node Details:**
    *   **Claude Scene Planner (HTTP Request):** Sends the script to `claude-sonnet-4-20250514`. The system prompt mandates a specific JSON schema (4-6 scenes) including camera movement, mood, and specialized prompts for Stability and Runway.
    *   **Parse Scenes JSON (Code):** Uses regex and JSON parsing to extract the scene array. It includes a robust fallback mechanism that generates two "demo" scenes if the AI response is malformed, preventing workflow failure.
*   **Edge Cases:** Malformed JSON from AI is caught and replaced by a fallback object.

#### 2.3 Stage 3 & 4 — Scene Loop & Image Generation
*   **Overview:** Processes each scene to create a visual reference frame.
*   **Nodes Involved:** `Split Scenes Into Batches`, `Format Scene Prompts`, `Generate Scene Image Stability`, `Extract Image Data`.
*   **Node Details:**
    *   **Split Scenes Into Batches:** Handles scenes one by one.
    *   **Format Scene Prompts (Code):** Appends style keywords (e.g., "cinematic", "4K", "8K") to the AI-generated prompts for better results.
    *   **Generate Scene Image Stability (HTTP Request):** Calls SDXL 1.0 to create a reference image. This image ensures "frame consistency" for the subsequent video generation.
    *   **Extract Image Data (Code):** Extracts the Base64 image data. *Note: In a production environment, this node identifies where the Base64 should be converted to a public URL (S3/Cloudinary).*

#### 2.4 Stage 5 — Runway Video Generation & Polling
*   **Overview:** Manages the asynchronous nature of video rendering.
*   **Nodes Involved:** `Submit to Runway Gen3`, `Store Runway Task ID`, `Wait 30 Seconds`, `Poll Runway Job Status`, `Check Job Complete`, `Is Video Ready`.
*   **Node Details:**
    *   **Submit to Runway Gen3 (HTTP):** Uses the `gen3a_turbo` model. Submits both the scene prompt and the reference image URL.
    *   **Polling Loop:** The workflow waits 30 seconds, then checks status via `Poll Runway Job Status`.
    *   **Check Job Complete (Code):** Increments `pollAttempts`. If status is `SUCCEEDED`, it captures the URL. If it reaches 10 attempts (300s total) without success, it marks a `TIMEOUT`.
*   **Edge Cases:** Runway API timeouts or failures route to the "isComplete" check to allow the loop to eventually terminate or continue.

#### 2.5 Stage 6 & 7 — Package, Deliver & Log
*   **Overview:** Finalizes the job and notifies stakeholders.
*   **Nodes Involved:** `Collect Completed Scene`, `Assemble Video Package`, `Notify Slack`, `Email Video Report`, `Log Job to Sheets`, `Return Final Response`.
*   **Node Details:**
    *   **Assemble Video Package (Code):** Sorts scenes by number, calculates total duration, and estimates cost (calculated at $0.05/sec for Runway Gen-3).
    *   **Delivery Nodes:** Parallel execution of Slack (formatted blocks), Email (SendGrid), and Google Sheets logging.
    *   **Return Final Response (Respond to Webhook):** Provides the full JSON package to the original caller.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Receive Script** | Webhook | Entry Point | - | Set Credentials and Config | Webhook accepts POST with script... |
| **Set Credentials and Config** | Set | Configuration | Receive Script | Validate Script | Set node normalizes all inputs... |
| **Validate Script** | IF | Validation | Set Credentials | Claude Scene Planner, Return Validation Error | IF node checks that a script field is not empty... |
| **Return Validation Error** | Respond to Webhook | Error Handling | Validate Script | - | Invalid requests return a 400 error. |
| **Claude Scene Planner** | HTTP Request | AI Planning | Validate Script | Parse Scenes JSON | Claude reads the full script and returns structured JSON... |
| **Parse Scenes JSON** | Code | Data Formatting | Claude Scene Planner | Split Scenes Into Batches | Parse node extracts and validates the scene array. |
| **Split Scenes Into Batches** | SplitInBatches | Loop Control | Parse Scenes JSON | Format Scene Prompts | SplitInBatches processes each scene individually. |
| **Format Scene Prompts** | Code | Prompt Engineering | Split Scenes Into Batches | Generate Scene Image Stability | Format node builds the exact prompt strings. |
| **Generate Scene Image Stability** | HTTP Request | Image Gen | Format Scene Prompts | Extract Image Data | Stability AI SDXL generates a high-resolution reference frame... |
| **Extract Image Data** | Code | Data Extraction | Generate Scene Image Stability | Submit to Runway Gen3 | This image anchors the visual style. |
| **Submit to Runway Gen3** | HTTP Request | Video Gen | Extract Image Data | Store Runway Task ID | Scene prompt and reference image URL are submitted... |
| **Store Runway Task ID** | Code | State Management | Submit to Runway Gen3 | Wait 30 Seconds | Runway returns a task ID. |
| **Wait 30 Seconds** | Wait | Polling Delay | Store Runway Task ID, Is Video Ready | Poll Runway Job Status | Wait node holds 30 seconds. |
| **Poll Runway Job Status** | HTTP Request | API Polling | Wait 30 Seconds | Check Job Complete | Poll node checks status. |
| **Check Job Complete** | Code | Logic | Poll Runway Job Status | Is Video Ready | Max 10 poll attempts before timeout. |
| **Is Video Ready** | IF | Loop Logic | Check Job Complete | Collect Completed Scene, Wait 30 Seconds | PENDING loops back to Wait, SUCCEEDED continues. |
| **Collect Completed Scene** | Code | Data Collection | Is Video Ready | Assemble Video Package | After all scenes complete, Code node assembles... |
| **Assemble Video Package** | Code | Final Aggregation | Collect Completed Scene | Slack, Email, Sheets | After all scenes complete, Code node assembles... |
| **Notify Slack** | HTTP Request | Notification | Assemble Video Package | Return Final Response | Slack notification with job summary... |
| **Email Video Report** | HTTP Request | Notification | Assemble Video Package | Return Final Response | SendGrid email with full delivery report. |
| **Log Job to Sheets** | Google Sheets | Auditing | Assemble Video Package | Return Final Response | Google Sheets audit log entry. |
| **Return Final Response** | Respond to Webhook | Response | Slack, Email, Sheets | - | Webhook returns complete JSON. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Webhook node**, set Method to `POST`, and path to `runway-video`. Set "Response Mode" to "When Last Node Finishes".
2.  **Configuration:** Add a **Set node** to define `ANTHROPIC_KEY`, `RUNWAY_KEY`, `STABILITY_KEY`, `SENDGRID_KEY`, `SLACK_WEBHOOK`, and `SHEET_ID`. Use expressions to capture `{{ $json.body.script }}`.
3.  **Input Logic:** Add an **IF node** to check if `script` is not empty. Connect an **Respond to Webhook node** to the `false` output for 400 errors.
4.  **AI Orchestrator:** Add an **HTTP Request node** (Claude).
    *   URL: `https://api.anthropic.com/v1/messages`
    *   Method: `POST`
    *   Headers: `x-api-key`, `anthropic-version: 2023-06-01`.
    *   Body: Request a JSON array of scenes based on the input script.
5.  **Data Parsing:** Add a **Code node** to parse Claude's response. Ensure it returns an array of items where each item represents a scene.
6.  **The Loop:** Add a **Split in Batches node**.
7.  **Visual Reference:**
    *   Add a **Code node** to format Stability AI prompts.
    *   Add an **HTTP Request node** for Stability AI (`/v1/generation/stable-diffusion-xl-1024-v1-0/text-to-image`).
8.  **Video Synthesis:**
    *   Add an **HTTP Request node** for Runway Gen-3 (`/v1/image_to_video`).
    *   Add a **Code node** to save the `id` as `runwayTaskId`.
9.  **Polling Mechanism:**
    *   Add a **Wait node** (30 seconds).
    *   Add an **HTTP Request node** for Runway Status (`/v1/tasks/{taskId}`).
    *   Add a **Code node** to evaluate if the status is `SUCCEEDED`.
    *   Add an **IF node**: If `isComplete` is true, continue; if false, loop back to the **Wait node**.
10. **Delivery:**
    *   Add a **Code node** to `Assemble Video Package` (combines all items from the loop).
    *   Add **HTTP Request nodes** for Slack and SendGrid.
    *   Add a **Google Sheets node** to append the row.
    *   Add a final **Respond to Webhook node**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **API Model (Claude)** | `claude-sonnet-4-20250514` |
| **API Model (Stability)** | `stable-diffusion-xl-1024-v1-0` |
| **API Model (Runway)** | `gen3a_turbo` |
| **Cost Estimation** | Estimated at $0.05 per second of generated video. |
| **Production Tip** | For production, replace Base64 storage with an upload to S3/Cloudinary in the `Extract Image Data` node. |
| **Runway Version Header** | Requires `X-Runway-Version: 2024-11-06` |