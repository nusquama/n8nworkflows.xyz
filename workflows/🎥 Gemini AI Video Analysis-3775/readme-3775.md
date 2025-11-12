🎥 Gemini AI Video Analysis

https://n8nworkflows.xyz/workflows/---gemini-ai-video-analysis-3775


# 🎥 Gemini AI Video Analysis

### 1. Workflow Overview

This workflow, titled **🎥 Gemini AI Video Analysis**, automates the process of generating detailed textual descriptions of video content by leveraging Google Gemini 2.0 Flash multimodal AI. It is designed to analyze any publicly accessible video URL, producing rich metadata suitable for accessibility, content moderation, media cataloging, and marketing insights.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts a public video URL and prepares it for processing.
- **1.2 Video Download:** Downloads the video content as binary data from the provided URL.
- **1.3 Video Upload to Gemini:** Uploads the downloaded video file to Google Gemini’s servers for AI processing.
- **1.4 Processing Wait:** Waits for the video to be fully processed by Gemini before continuing.
- **1.5 AI Video Analysis:** Sends a request to Gemini AI with a prompt to generate a detailed description of the video content.
- **1.6 Result Extraction:** Extracts and formats the AI-generated video description for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initializes the workflow by accepting the video URL input, which can be customized per use case. It also ensures the Gemini API key is securely referenced from environment variables.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Input

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Connects to `Set Input` node.  
    - Edge cases: None; manual trigger requires user interaction.

  - **Set Input**  
    - Type: Set  
    - Role: Defines the input variable `video_url` with a sample public video URL.  
    - Configuration: Assigns a string variable `video_url` with a sample Facebook video URL.  
    - Key expressions: `video_url` is set statically here but intended to be replaced or parameterized.  
    - Inputs: From manual trigger  
    - Outputs: Connects to `Download video` node.  
    - Edge cases: If the URL is invalid or inaccessible, subsequent download will fail.

---

#### 2.2 Video Download

- **Overview:**  
  Downloads the video from the provided URL as binary data, preparing it for upload to Gemini.

- **Nodes Involved:**  
  - Download video

- **Node Details:**

  - **Download video**  
    - Type: HTTP Request  
    - Role: Fetches the video file from the URL specified in `video_url`.  
    - Configuration:  
      - HTTP method: GET (default)  
      - URL: Expression `={{ $json.video_url }}` dynamically uses the input URL.  
      - Options: Default, no special headers or authentication.  
      - Output: Binary data of the video file.  
    - Inputs: From `Set Input`  
    - Outputs: Connects to `Upload video Gemini`.  
    - Edge cases:  
      - Network errors or invalid URLs cause failure.  
      - Video size or format unsupported by Gemini may cause issues downstream.  
      - No authentication handled; only public URLs supported.

---

#### 2.3 Video Upload to Gemini

- **Overview:**  
  Uploads the downloaded video binary data to Gemini’s file upload endpoint for processing.

- **Nodes Involved:**  
  - Upload video Gemini

- **Node Details:**

  - **Upload video Gemini**  
    - Type: HTTP Request  
    - Role: Uploads video binary data to Gemini API’s upload endpoint.  
    - Configuration:  
      - HTTP method: POST  
      - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files?key={{ $vars.GeminiKey }}` (uses environment variable `GeminiKey` for API key)  
      - Content-Type: `video/mp4` (hardcoded)  
      - Headers:  
        - `X-Goog-Upload-Command`: `"start, upload, finalize"` (indicates upload lifecycle)  
        - `X-Goog-Upload-Header-Content-Length`: dynamic from binary file size  
        - `X-Goog-Upload-Header-Content-Type`: dynamic from binary file extension  
      - Body: Binary data from previous node  
      - Input data field: `data` (binary)  
    - Inputs: From `Download video`  
    - Outputs: Connects to `Wait` node.  
    - Edge cases:  
      - API key missing or invalid causes auth errors.  
      - Upload failures due to network or file size limits.  
      - Binary data must be correctly passed; expression failures may occur if metadata missing.

---

#### 2.4 Processing Wait

- **Overview:**  
  Waits for a predefined duration to allow Gemini to process the uploaded video before requesting analysis.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Introduces a delay to ensure video processing completes on Gemini’s servers.  
    - Configuration: Default wait with no parameters set (likely minimal or default delay).  
    - Inputs: From `Upload video Gemini`  
    - Outputs: Connects to `Analyze video Gemini` node.  
    - Edge cases:  
      - Insufficient wait time may cause analysis request to fail if processing incomplete.  
      - Excessive wait increases workflow runtime unnecessarily.

---

#### 2.5 AI Video Analysis

- **Overview:**  
  Sends a request to Gemini’s AI model to generate a detailed textual description of the uploaded video using a customizable prompt.

- **Nodes Involved:**  
  - Analyze video Gemini

- **Node Details:**

  - **Analyze video Gemini**  
    - Type: HTTP Request  
    - Role: Calls Gemini AI’s content generation endpoint with a prompt and video file reference.  
    - Configuration:  
      - HTTP method: POST  
      - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key={{ $vars.GeminiKey }}`  
      - Headers: Content-Type `application/json`  
      - JSON Body:  
        - `contents`: Array with two parts:  
          1. `fileData` referencing the uploaded video URI and MIME type from previous upload response.  
          2. Text prompt requesting a detailed description of visual elements, style, tone, branding, etc.  
        - `generationConfig`: Parameters tuning AI response (temperature 1.4, topK 40, topP 0.95, max tokens 8192, response modality Text)  
    - Inputs: From `Wait` node (which carries upload response)  
    - Outputs: Connects to `Get Result` node.  
    - Edge cases:  
      - API key errors or quota limits.  
      - Improper file URI or MIME type causes request failure.  
      - Timeout or large response size may cause node failure.  
      - Prompt customization errors if JSON malformed.

---

#### 2.6 Result Extraction

- **Overview:**  
  Extracts the AI-generated video description text from the Gemini response and stores it in a workflow variable for downstream use.

- **Nodes Involved:**  
  - Get Result

- **Node Details:**

  - **Get Result**  
    - Type: Set  
    - Role: Assigns the detailed video description text to the variable `videoDescription`.  
    - Configuration:  
      - Expression extracts `candidates[0].content.parts[0].text` from the JSON response of the previous node.  
    - Inputs: From `Analyze video Gemini`  
    - Outputs: None (end of workflow)  
    - Edge cases:  
      - If Gemini returns no candidates or unexpected response structure, expression will fail.  
      - Empty or partial descriptions may occur depending on AI output.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                     | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                             |
|-------------------------|---------------------|-----------------------------------|---------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Workflow entry point               | -                         | Set Input                |                                                                                                                                         |
| Set Input               | Set                 | Defines input video URL            | When clicking ‘Test workflow’ | Download video           | ## Configuration Define the video URL you want to analyze.                                                                             |
| Download video          | HTTP Request        | Downloads video from URL           | Set Input                 | Upload video Gemini      | ## Video Processing Pipeline 1. DOWNLOAD: Fetch video from URL as binary data.                                                          |
| Upload video Gemini     | HTTP Request        | Uploads video binary to Gemini API | Download video            | Wait                     | ## Video Processing Pipeline 2. UPLOAD: Send binary video to Gemini servers.                                                             |
| Wait                    | Wait                | Waits for video processing         | Upload video Gemini       | Analyze video Gemini     | ## Video Processing Pipeline 3. ANALYZE: Ensures video processing completes before analysis.                                             |
| Analyze video Gemini    | HTTP Request        | Sends AI prompt to Gemini for analysis | Wait                      | Get Result               | ## Video Processing Pipeline 3. ANALYZE: Customize prompt to focus on relevant video details.                                            |
| Get Result              | Set                 | Extracts AI-generated video description | Analyze video Gemini      | -                        |                                                                                                                                         |
| Sticky Note3            | Sticky Note         | Workflow overview and security notes | -                         | -                        | ## Video Analysis with Gemini AI This workflow demonstrates how to analyze video content using Google's Gemini 2.0 Flash API...          |
| Sticky Note4            | Sticky Note         | Configuration note on video URL    | -                         | -                        | ## Configuration Define the video URL you want to analyze.                                                                             |
| Sticky Note1            | Sticky Note         | Explanation of video processing pipeline | -                         | -                        | ## Video Processing Pipeline This section handles the complete video processing workflow: DOWNLOAD, UPLOAD, ANALYZE steps explained.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Start the workflow manually.

2. **Create a Set node:**  
   - Name: `Set Input`  
   - Purpose: Define the input video URL.  
   - Configuration: Add a string field `video_url` with a sample public video URL (replaceable).  
   - Connect output of `When clicking ‘Test workflow’` to this node.

3. **Create an HTTP Request node:**  
   - Name: `Download video`  
   - Purpose: Download the video file from the URL.  
   - Configuration:  
     - HTTP Method: GET (default)  
     - URL: Expression `={{ $json.video_url }}` to dynamically use the input URL.  
     - No authentication or special headers.  
   - Connect output of `Set Input` to this node.

4. **Create an HTTP Request node:**  
   - Name: `Upload video Gemini`  
   - Purpose: Upload video binary to Gemini API.  
   - Configuration:  
     - HTTP Method: POST  
     - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files?key={{ $vars.GeminiKey }}`  
     - Content-Type: `video/mp4`  
     - Headers:  
       - `X-Goog-Upload-Command`: `start, upload, finalize`  
       - `X-Goog-Upload-Header-Content-Length`: Expression `={{ $binary.data.fileSize }}`  
       - `X-Goog-Upload-Header-Content-Type`: Expression `=video/{{ $binary.data.fileExtension }}`  
     - Body: Binary data from `Download video` node (field name `data`)  
   - Connect output of `Download video` to this node.  
   - **Credential Setup:** Ensure `GeminiKey` API key is set as an environment variable in n8n or use credentials manager.

5. **Create a Wait node:**  
   - Name: `Wait`  
   - Purpose: Pause to allow video processing.  
   - Configuration: Default wait time (adjust if needed based on video size and processing time).  
   - Connect output of `Upload video Gemini` to this node.

6. **Create an HTTP Request node:**  
   - Name: `Analyze video Gemini`  
   - Purpose: Request Gemini AI to analyze the uploaded video.  
   - Configuration:  
     - HTTP Method: POST  
     - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key={{ $vars.GeminiKey }}`  
     - Headers: Content-Type `application/json`  
     - JSON Body:  
       ```json
       {
         "contents": [
           {
             "role": "user",
             "parts": [
               {
                 "fileData": {
                   "fileUri": "{{ $json.file.uri }}",
                   "mimeType": "{{ $json.file.mimeType }}"
                 }
               },
               {
                 "text": "Describe in detail what is visually happening in the video, including key elements, actions, colors, and branding. Note the style, tone, and any notable creative techniques being used."
               }
             ]
           }
         ],
         "generationConfig": {
           "temperature": 1.4,
           "topK": 40,
           "topP": 0.95,
           "maxOutputTokens": 8192,
           "responseModalities": ["Text"]
         }
       }
       ```  
     - Use expressions to dynamically insert `file.uri` and `file.mimeType` from the previous node’s output.  
   - Connect output of `Wait` node to this node.

7. **Create a Set node:**  
   - Name: `Get Result`  
   - Purpose: Extract the detailed video description text from Gemini’s response.  
   - Configuration:  
     - Add a string field `videoDescription` with expression:  
       `={{ $json.candidates[0].content.parts[0].text }}`  
   - Connect output of `Analyze video Gemini` to this node.

8. **Environment Variable Setup:**  
   - Define `GeminiKey` in your n8n environment variables or credentials manager with your Google Gemini API key.  
   - Do **not** hardcode the API key in any node.

9. **Test the workflow:**  
   - Trigger manually via the manual trigger node.  
   - Confirm the video URL is accessible and the API key is valid.  
   - Inspect the output in `Get Result` node for the generated video description.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow demonstrates how to analyze video content using Google's Gemini 2.0 Flash API: download, upload, process, and extract AI-generated descriptions. Use cases include content moderation, accessibility, and cataloging. | Sticky Note3 content in the workflow.                                                           |
| Before running, set the `GeminiKey` environment variable with your Gemini API key to ensure secure API access and avoid hardcoding sensitive credentials.                                                                         | Setup Instructions section and Sticky Note3.                                                   |
| Customize the prompt in the `Analyze video Gemini` node to tailor the AI’s focus on specific video aspects such as branding, actions, or style.                                                                                   | Workflow description and Sticky Note1.                                                          |
| Video URLs must be publicly accessible without authentication; private or restricted videos will cause download failures.                                                                                                         | Setup Instructions and `Download video` node analysis.                                          |
| For more information on Google Gemini API and registration, visit [https://ai.google.dev/](https://ai.google.dev/).                                                                                                              | Setup Instructions section.                                                                     |

---

This structured documentation enables advanced users and AI agents to understand, reproduce, and modify the **🎥 Gemini AI Video Analysis** workflow efficiently, while anticipating potential integration issues and security best practices.