Automate Image Editing & Instagram Posting with Nano Banana & GPT-5 Captions

https://n8nworkflows.xyz/workflows/automate-image-editing---instagram-posting-with-nano-banana---gpt-5-captions-8690


# Automate Image Editing & Instagram Posting with Nano Banana & GPT-5 Captions

### 1. Workflow Overview

This workflow automates the process of editing images uploaded to a specific Google Drive folder using the Nano Banana image enhancement API and subsequently posting the enhanced images to Instagram with AI-generated captions powered by GPT-5. It is designed primarily for real-estate agents, ecommerce sellers, and social media marketers who want to streamline image cleanup, caption creation, and social posting in a single automated flow.

Logical blocks of the workflow:

- **1.1 Input Reception**: Triggered when a new image file is added to a specified Google Drive folder.
- **1.2 Image Enhancement via Nano Banana API**: Sends the new image to the Wavespeed Nano Banana API for cleanup and enhancement, then polls for the processed result.
- **1.3 Logging Edited Image**: Records the URL of the edited image along with a timestamp in a Google Sheets log.
- **1.4 Image Download and Upload to Postiz**: Downloads the edited image and uploads it to Postiz, a social media scheduling platform.
- **1.5 Caption Generation with GPT-5**: Generates a friendly, engaging Instagram caption for the image using an OpenAI GPT-5-powered LangChain agent.
- **1.6 Instagram Posting**: Publishes the image and caption to Instagram via Postiz API.
- **1.7 Workflow Logging**: Logs the publishing status and metadata to another Google Sheets document for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Detects new image files uploaded into a designated Google Drive folder and initiates the workflow.
- **Nodes Involved:** Google Drive Trigger
- **Node Details:**
  - **Google Drive Trigger**
    - Type: Trigger node for Google Drive events.
    - Config: Watches a specific Google Drive folder (folder ID parameterized) for newly created files; polls every minute.
    - Key parameters: event = `fileCreated`, triggerOn = `specificFolder`.
    - Input connections: None (trigger node).
    - Output connections: Sends file metadata to "Nano Banana POST Request".
    - Edge cases: Folder ID must be correctly set; permission errors if Google Drive credentials lack access; large files may slow trigger.
    - Credentials: Requires Google Drive OAuth2 credentials.

#### 2.2 Image Enhancement via Nano Banana API

- **Overview:** Sends the uploaded image URL to the Nano Banana API for auto-editing, waits, then polls the API for the processed image result.
- **Nodes Involved:** Nano Banana POST Request, Wait 15 Secs, GET Result from Nano Banana Node, If, Wait 15 Secs Again
- **Node Details:**

  - **Nano Banana POST Request**
    - Type: HTTP Request node.
    - Config: POST request to Wavespeed Nano Banana API endpoint with JSON body containing image URL and editing prompt.
    - Prompt: Instructs Nano Banana to clean clutter, remove trash and personal items, tidy surfaces, enhance lighting, while preserving structure.
    - Authentication: HTTP header auth (API key stored in credentials).
    - Input: Receives file data from Google Drive Trigger.
    - Output: Receives prediction job ID for polling.
    - Edge cases: API key invalid or missing; API downtime; malformed URL; unexpected response format.

  - **Wait 15 Secs**
    - Type: Wait node.
    - Config: Waits fixed 15 seconds before polling for result.
    - Input: From Nano Banana POST Request.
    - Output: To GET Result from Nano Banana Node.
    - Edge cases: Fixed delay may be insufficient if processing is longer.

  - **GET Result from Nano Banana Node**
    - Type: HTTP Request node.
    - Config: GET request to Wavespeed API polling endpoint using the prediction job ID.
    - Authentication: HTTP header auth (same as above).
    - Input: From Wait 15 Secs or Wait 15 Secs Again nodes.
    - Output: Sends response JSON to "If" node.
    - Edge cases: API rate limits; job ID expired; incomplete or error status responses.

  - **If**
    - Type: Conditional node.
    - Config: Checks if the Nano Banana job status equals "completed".
    - Input: From GET Result from Nano Banana Node.
    - Output:
      - True branch: proceeds to append result to Google Sheets.
      - False branch: loops back to "Wait 15 Secs Again" to retry polling.
    - Edge cases: Status may be "failed" or other; no timeout or max retries configured.

  - **Wait 15 Secs Again**
    - Same as above Wait node, used for polling retry delay.

#### 2.3 Logging Edited Image

- **Overview:** Appends the URL of the edited image and a timestamp to a Google Sheets document for tracking purposes.
- **Nodes Involved:** Append row in sheet
- **Node Details:**
  - Type: Google Sheets node (append operation).
  - Config: Appends columns "Image URL" (from Nano Banana API response) and "Timestamp" (current time).
  - Input: From "If" node on true branch.
  - Output: Passes data to "Get Image" node.
  - Edge cases: Sheet ID and sheet name must be correct; permissions required; rate limits possible.

#### 2.4 Image Download and Upload to Postiz

- **Overview:** Downloads the edited image using its URL, then uploads the binary image data to Postiz media library for later posting.
- **Nodes Involved:** Get Image, Upload to Postiz
- **Node Details:**

  - **Get Image**
    - Type: HTTP Request node.
    - Config: Downloads the image from the URL provided by Nano Banana result.
    - Input: From "Append row in sheet".
    - Output: Binary image data sent to "Upload to Postiz".
    - Edge cases: URL must be accessible; large images may cause timeouts.

  - **Upload to Postiz**
    - Type: HTTP Request node (multipart/form-data upload).
    - Config: POST to Postiz public API endpoint with binary image file under "file" form field.
    - Authentication: HTTP header auth using Postiz API credentials.
    - Input: From "Get Image".
    - Output: Provides uploaded media metadata to "Update Log".
    - Edge cases: API key invalid; upload size limits; network errors.

#### 2.5 Caption Generation with GPT-5

- **Overview:** Generates a concise, friendly, and engaging Instagram caption for the image using GPT-5 through LangChain integration.
- **Nodes Involved:** Update Log, Caption Agent, OpenAI Chat Model
- **Node Details:**

  - **Update Log**
    - Type: Google Sheets node (append operation).
    - Config: Logs the status "Uploaded to Postiz", timestamp, and video/image name.
    - Input: From "Upload to Postiz".
    - Output: Passes data to "Caption Agent".
    - Edge cases: Same as other Google Sheets node.

  - **Caption Agent**
    - Type: LangChain Agent node.
    - Config: Receives the image name/description as input text.
    - System message instructs the agent to generate one Instagram or TikTok caption with a friendly, impactful, exciting tone, avoiding special characters and explanations.
    - Input: From "Update Log".
    - Output: Sends caption text to "Post to Instagram".
    - Edge cases: API quotas; malformed prompt; generation failures.

  - **OpenAI Chat Model**
    - Type: LangChain language model node.
    - Config: Uses GPT-5 model.
    - Connected as AI language model backend to "Caption Agent".
    - Credentials: OpenAI API key.
    - Edge cases: API key issues; rate limits.

#### 2.6 Instagram Posting

- **Overview:** Posts the edited image and generated caption to Instagram via Postiz API.
- **Nodes Involved:** Post to Instagram
- **Node Details:**
  - Type: HTTP Request node.
  - Config: POST JSON body to Postiz posts endpoint; includes integration ID, caption content, image ID and path from upload.
  - Authentication: HTTP header auth with Postiz credentials.
  - Input: From "Caption Agent".
  - Output: None (end of posting chain).
  - Edge cases: Integration ID must be correct; API errors; rate limits.

#### 2.7 Workflow Logging and Documentation (Sticky Notes)

- Several sticky notes provide visual and textual guidance for the blocks:
  - Drive Trigger explanation.
  - Nano Banana results polling.
  - Image download and upload instructions.
  - Caption generation and posting clarifications.
  - A large sticky note summarizes the entire workflow, usage instructions, and links to tutorial videos.

---

### 3. Summary Table

| Node Name                   | Node Type                             | Functional Role                      | Input Node(s)               | Output Node(s)              | Sticky Note                                                      |
|-----------------------------|-------------------------------------|------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------|
| Google Drive Trigger         | Google Drive Trigger                 | Detect new images in Drive folder  | None                        | Nano Banana POST Request     | Drive Trigger                                                   |
| Nano Banana POST Request     | HTTP Request                        | Submit image for enhancement       | Google Drive Trigger         | Wait 15 Secs                | Drive Trigger                                                   |
| Wait 15 Secs                | Wait                               | Delay before polling result        | Nano Banana POST Request     | GET Result from Nano Banana Node | Get Results from Nano Banana                                   |
| GET Result from Nano Banana Node | HTTP Request                   | Poll for edited image result       | Wait 15 Secs, Wait 15 Secs Again | If                      | Get Results from Nano Banana                                   |
| If                         | If                                 | Check if image processing completed | GET Result from Nano Banana Node | Append row in sheet (true), Wait 15 Secs Again (false) | Get Results from Nano Banana                                   |
| Append row in sheet          | Google Sheets                      | Log edited image URL and timestamp | If (true branch)             | Get Image                   |                                                                 |
| Get Image                   | HTTP Request                       | Download edited image              | Append row in sheet          | Upload to Postiz            | Get Image and Upload                                           |
| Upload to Postiz            | HTTP Request                       | Upload image to Postiz platform    | Get Image                   | Update Log                  | Get Image and Upload                                           |
| Update Log                  | Google Sheets                      | Log upload status and metadata     | Upload to Postiz            | Caption Agent               |                                                                 |
| Caption Agent               | LangChain Agent                    | Generate Instagram caption         | Update Log                  | Post to Instagram           | Caption for IG                                                |
| OpenAI Chat Model           | LangChain Language Model           | GPT-5 model backend for captioning | Caption Agent (AI model input) | Caption Agent              | Caption for IG                                                |
| Post to Instagram           | HTTP Request                      | Publish image and caption to IG    | Caption Agent               | None                       | Post to IG                                                    |
| Wait 15 Secs Again          | Wait                              | Delay for polling retry            | If (false branch)            | GET Result from Nano Banana Node | Get Results from Nano Banana                                   |
| Sticky Note                 | Sticky Note                       | Visual aid and documentation       | None                        | None                       | See section 2.7                                               |
| Sticky Note1                | Sticky Note                       | Visual aid for Nano Banana results | None                        | None                       | Get Results from Nano Banana                                   |
| Sticky Note2                | Sticky Note                       | Visual aid for Drive Trigger        | None                        | None                       | Drive Trigger                                                 |
| Sticky Note3                | Sticky Note                       | Visual aid for Drive Trigger        | None                        | None                       | Drive Trigger                                                 |
| Sticky Note4                | Sticky Note                       | Visual aid for caption block        | None                        | None                       | Caption for IG                                                |
| Sticky Note5                | Sticky Note                       | Visual aid for posting block        | None                        | None                       | Post to IG                                                    |
| Sticky Note6                | Sticky Note                       | Full workflow summary & instructions | None                      | None                       | See section 2.7                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**
   - Type: Google Drive Trigger.
   - Event: fileCreated.
   - Folder to watch: set your target Google Drive folder ID.
   - Poll interval: every minute.
   - Credentials: configure Google Drive OAuth2 credentials with read access.

2. **Add HTTP Request node "Nano Banana POST Request":**
   - Method: POST.
   - URL: https://api.wavespeed.ai/api/v3/google/nano-banana/edit.
   - Body type: JSON.
   - Body content:
     ```json
     {
       "enable_base64_output": false,
       "enable_sync_mode": false,
       "images": ["{{ $json.webContentLink }}"],
       "output_format": "jpeg",
       "prompt": "Clean up and declutter this apartment unit: remove all mess, trash, and personal items while keeping the original furniture, layout, and design intact. Tidy up surfaces, straighten objects, and make the space look neat and inviting. Enhance the lighting to appear bright and natural, similar to professional real-estate photography, without altering the structure or style of the apartment."
     }
     ```
   - Authentication: HTTP header auth with Wavespeed API key.
   - Connect output of Google Drive Trigger to this node.

3. **Add "Wait 15 Secs" node:**
   - Wait time: 15 seconds.
   - Connect output of Nano Banana POST Request to this node.

4. **Add HTTP Request node "GET Result from Nano Banana Node":**
   - Method: GET.
   - URL: https://api.wavespeed.ai/api/v3/predictions/{{ $json.data.id }}/result (use expression to insert job ID).
   - Authentication: HTTP header auth (same Wavespeed API credentials).
   - Connect output of Wait 15 Secs to this node.

5. **Add "If" node:**
   - Condition: Check if $json.data.status equals "completed".
   - True branch: Connect to "Append row in sheet".
   - False branch: Connect to "Wait 15 Secs Again".

6. **Add second "Wait 15 Secs Again" node:**
   - Wait time: 15 seconds.
   - Connect false branch of "If" node to this node.
   - Connect output of this node back to "GET Result from Nano Banana Node" for polling retry.

7. **Add Google Sheets node "Append row in sheet":**
   - Operation: Append.
   - Document ID: your Google Sheets ID for edited images log.
   - Sheet name: your sheet name.
   - Columns: "Image URL" from $json.data.outputs[0], "Timestamp" as current time.
   - Connect true branch of "If" node to this node.

8. **Add HTTP Request node "Get Image":**
   - Method: GET.
   - URL: expression from "Image URL" in previous node.
   - Connect output of "Append row in sheet" node here.

9. **Add HTTP Request node "Upload to Postiz":**
   - Method: POST.
   - URL: https://api.postiz.com/public/v1/upload.
   - Body type: multipart/form-data.
   - Body parameter: "file" as binary data input.
   - Authentication: HTTP header auth with Postiz API key.
   - Connect output of "Get Image" to this node.

10. **Add Google Sheets node "Update Log":**
    - Operation: Append.
    - Document ID: your Google Sheets ID for publishing logs.
    - Sheet name: your sheet name.
    - Columns: "Status" = "Uploaded to Postiz", "Timestamp" = current time, "Video Name and Description" = image file name from Google Drive Trigger.
    - Connect output of "Upload to Postiz" to this node.

11. **Add LangChain "Caption Agent" node:**
    - Text input: "Video Name and Description" from "Update Log".
    - System message prompt:
      ```
      You are an expert Instagram caption writer.
      Your task is to take the image prompt provided by the user and generate a short, engaging caption for Instagram or Tik-Tok Reels
      
      Caption Guidelines
      
      
      Tone: friendly, impactful and exciting
      
      ##Output Rules
      Do not use special characters like {} ! %$&*
      Return only one final caption per request.
      Do not include explanations or formatting outside of the caption itself.
      ```
    - Prompt type: define.
    - Connect output of "Update Log" to this node.

12. **Add LangChain "OpenAI Chat Model" node:**
    - Model: GPT-5.
    - Connect to "Caption Agent" as AI language model backend.
    - Credentials: OpenAI API key.

13. **Add HTTP Request node "Post to Instagram":**
    - Method: POST.
    - URL: https://api.postiz.com/public/v1/posts.
    - Body type: JSON.
    - Body content:
      ```json
      {
        "type": "now",
        "shortLink": false,
        "date": "{{ new Date($now).toISOString() }}",
        "tags": [],
        "posts": [
          {
            "integration": { "id": "YOUR_POSTIZ_INTEGRATION_ID" },
            "value": [
              {
                "content": "{{ $json.output }}",
                "image": [
                  {
                    "id": "{{ $node['Upload to Postiz'].json.id }}",
                    "path": "{{ $node['Upload to Postiz'].json.path }}"
                  }
                ]
              }
            ],
            "settings": {
              "post_type": "post"
            }
          }
        ]
      }
      ```
    - Authentication: HTTP header auth with Postiz API key.
    - Connect output of "Caption Agent" to this node.

14. **Add sticky notes to annotate workflow blocks** as per your preference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| 📸 Auto-Edit Google Drive Images with Nano Banana + Social Auto-Post. Drop an image into Google Drive and let this workflow handle the rest: it auto-cleans and enhances the image with Google’s Nano Banana (via Wavespeed API), generates a catchy caption with GPT-5, and publishes directly to your connected social accounts using Postiz. Setup instructions and customization tips included. | See sticky note in workflow; video tutorials: https://www.youtube.com/watch?v=4wk6PYgBtBM&list=PL05w1TE8X3bb1H9lXBqUy98zmTrrPP-s1 |
| Replace all sample IDs (Google Drive folder ID, Google Sheets IDs, Postiz integration ID) with your own values to ensure workflow functionality.                                                                                                                                                                                                                                             | Workflow setup instructions.                                                                               |
| Keep all API keys and sensitive credentials stored securely in n8n credentials; do not embed keys in node parameters.                                                                                                                                                                                                                                                                          | Security best practice.                                                                                     |
| Adjust the Nano Banana prompt to customize image editing for different use cases (e.g., product photography, lighting enhancements).                                                                                                                                                                                                                                                           | Editing prompt in Nano Banana POST Request node.                                                          |
| Optionally, add approval loops (via Slack or Email) before posting to social media for manual review.                                                                                                                                                                                                                                                                                          | Customization suggestion.                                                                                   |
| Logs maintained in Google Sheets provide traceability for edited images and publishing status.                                                                                                                                                                                                                                                                                                 | Logging nodes Append row in sheet and Update Log.                                                         |

---

**Disclaimer**: The provided content is solely derived from an automated n8n workflow designed with proper content policies. It contains no illegal or protected materials. All processed data is legal and public.