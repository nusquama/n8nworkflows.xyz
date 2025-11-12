Flux Dev Image Generation (Fal.ai) to Google Drive

https://n8nworkflows.xyz/workflows/flux-dev-image-generation--fal-ai--to-google-drive-2644


# Flux Dev Image Generation (Fal.ai) to Google Drive

---
### 1. Workflow Overview

This workflow automates AI-driven image generation using the Fal.ai Flux API and saves the resulting images directly to Google Drive. It is designed to simplify and streamline the entire process from defining image parameters to final storage, targeting content creators, developers, and automation experts who want to integrate AI image generation into their creative workflows.

**Logical Blocks:**

- **1.1 Input Parameter Setup:** Defines the AI image generation parameters such as prompt text, image resolution, inference steps, and guidance scale.
- **1.2 Image Generation Request:** Sends a POST request to Fal.ai’s Flux API to initiate image generation.
- **1.3 Status Monitoring Loop:** Polls the API to check the generation status, waiting and retrying until the image is ready.
- **1.4 Image Retrieval:** Once complete, fetches the generated image URL and downloads the image.
- **1.5 Storage:** Uploads the downloaded image to a specified folder in Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Parameter Setup

- **Overview:**  
  This block captures and sets the customizable image generation parameters required by the Fal.ai Flux API.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)  
  - `Edit Fields` (Set Node)  
  - `Sticky Note` (Instructional note)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type & Role:* Manual Trigger node to start the workflow.  
    - *Configuration:* No parameters; triggers workflow execution manually.  
    - *Connections:* Outputs to `Edit Fields`.  
    - *Failure Modes:* None; manual trigger.  

  - **Edit Fields**  
    - *Type & Role:* Set node that defines the image generation parameters.  
    - *Configuration:*  
      - `Prompt`: "Thai young woman net idol 25 yrs old, walking on the street"  
      - `Width`: 1024  
      - `Height`: 768  
      - `Steps`: 30 (number of inference steps)  
      - `Guidance`: 3.5 (guidance scale controlling prompt adherence)  
    - *Expressions/Variables:* Static values set as per user input.  
    - *Connections:* Receives from `When clicking ‘Test workflow’`, outputs to `Fal Flux`.  
    - *Failure Modes:* Incorrect data types or missing fields may cause API request errors.  

  - **Sticky Note**  
    - *Type & Role:* Instructional note prompting users to set image parameters here.  
    - *Content:* "## Set Parameter Here \nset Image Prompt and related settings"  
    - *Connections:* None (informational only).

#### 2.2 Image Generation Request

- **Overview:**  
  Sends the formatted request to Fal.ai Flux API to initiate image generation using the parameters defined earlier.

- **Nodes Involved:**  
  - `Fal Flux` (HTTP Request)  
  - `Sticky Note2` (Credential instructions)

- **Node Details:**

  - **Fal Flux**  
    - *Type & Role:* HTTP Request node; performs POST to initiate image generation.  
    - *Configuration:*  
      - URL: `https://queue.fal.run/fal-ai/flux/dev`  
      - Method: POST  
      - Body (JSON): Uses expressions to insert parameters from `Edit Fields` JSON:  
        - `prompt` ← `{{$json.Prompt}}`  
        - `image_size.width` ← `{{$json.Width}}`  
        - `image_size.height` ← `{{$json.Height}}`  
        - `num_inference_steps` ← `{{$json.Steps}}`  
        - `guidance_scale` ← `{{$json.Guidance}}`  
        - Other fixed: `num_images`=1, `enable_safety_checker`=true  
      - Authentication: Generic Credential (HTTP Header Auth) using Fal.ai API Key.  
    - *Input:* Receives from `Edit Fields`.  
    - *Output:* Passes to `Wait 3 Sec`.  
    - *Failure Modes:*  
      - Authorization errors if API key invalid.  
      - API errors if parameters invalid or service unavailable.  
    - *Sticky Note2:*  
      - Contains instructions on setting up generic credential with Authorization header key `$FAL_KEY`, e.g., `Key 6f2960baxxxxxxxxx`.  

  - **Sticky Note2**  
    - *Role:* Provides credential setup instructions relevant to `Fal Flux`.  
    - *Content:*  
      ```
      ### Generic Credential Type
      ### Header : Authorization
      Key $FAL_KEY"

      for example:
      Key 6f2960baxxxxxxxxx
      ```

#### 2.3 Status Monitoring Loop

- **Overview:**  
  Implements a polling mechanism to repeatedly check the status of the image generation request until it completes.

- **Nodes Involved:**  
  - `Wait 3 Sec` (Wait node)  
  - `Check Status` (HTTP Request)  
  - `Completed?` (If node)  

- **Node Details:**

  - **Wait 3 Sec**  
    - *Type & Role:* Wait node; delays execution by 3 seconds between status checks.  
    - *Configuration:* Fixed 3-second delay.  
    - *Input:* Receives from `Fal Flux` initially and loops back from `Completed?` (if not complete).  
    - *Output:* Outputs to `Check Status`.  
    - *Failure Modes:* None expected; standard delay.  

  - **Check Status**  
    - *Type & Role:* HTTP Request node; queries the current generation status of the request.  
    - *Configuration:*  
      - URL: `https://queue.fal.run/fal-ai/flux/requests/{{$json.request_id}}/status`  
      - Method: GET (default)  
      - Authentication: Generic Credential (same as Fal Flux)  
    - *Input:* From `Wait 3 Sec`.  
    - *Output:* To `Completed?`.  
    - *Failure Modes:*  
      - Authorization errors.  
      - Network timeout.  
      - Incorrect request_id format leading to 404.  

  - **Completed?**  
    - *Type & Role:* If node; evaluates if generation status is "COMPLETED".  
    - *Configuration:* Checks if `$json.status === "COMPLETED"`  
    - *Input:* From `Check Status`.  
    - *Outputs:*  
      - True branch: `Get Image Result URL` (next step)  
      - False branch: loops back to `Wait 3 Sec` to continue polling  
    - *Failure Modes:* Expression evaluation errors if `status` field missing.  

#### 2.4 Image Retrieval

- **Overview:**  
  Retrieves the final image URL and downloads the actual image file once the generation is complete.

- **Nodes Involved:**  
  - `Get Image Result URL` (HTTP Request)  
  - `Download Image` (HTTP Request)  
  - `Sticky Note1` (Instruction to set Drive folder)

- **Node Details:**

  - **Get Image Result URL**  
    - *Type & Role:* HTTP Request; fetches the image metadata including the URL.  
    - *Configuration:*  
      - URL: `https://queue.fal.run/fal-ai/flux/requests/{{$json.request_id}}`  
      - Method: GET (default)  
      - Authentication: Generic Credential (Fal Flux API key)  
    - *Input:* From true branch of `Completed?`.  
    - *Output:* To `Download Image`.  
    - *Failure Modes:* Authorization failure, invalid `request_id`, or API downtime.  

  - **Download Image**  
    - *Type & Role:* HTTP Request to download the image binary data.  
    - *Configuration:*  
      - URL: `{{$json.images[0].url}}` (direct URL to generated image)  
      - Method: GET (default)  
      - Options: default (no authentication)  
    - *Input:* From `Get Image Result URL`.  
    - *Output:* To `Google Drive`.  
    - *Failure Modes:* Broken image URL, network failures, large file timeouts.  

  - **Sticky Note1**  
    - *Role:* Instruction to configure Google Drive folder for image saving.  
    - *Content:* "## Set Drive Folder Here"  

#### 2.5 Storage

- **Overview:**  
  Uploads the downloaded image file to a specified folder in Google Drive.

- **Nodes Involved:**  
  - `Google Drive` (Google Drive node)

- **Node Details:**

  - **Google Drive**  
    - *Type & Role:* Uploads binary image data to Google Drive.  
    - *Configuration:*  
      - File Name: `{{$binary.data.fileName}}` (filename derived from downloaded binary)  
      - Drive: "My Drive"  
      - Folder ID: `1R3PSyHXWHlY9DRFdOUEAPEop2fZy-_-K` (user must set desired folder)  
      - Credentials: OAuth2 Google Drive account  
    - *Input:* Receives binary data from `Download Image`.  
    - *Output:* End of workflow.  
    - *Failure Modes:*  
      - OAuth token expiration or invalid credentials.  
      - Folder ID incorrect or inaccessible.  
      - File size limits or quota exceeded.  

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                      | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                           |
|----------------------------|------------------------|------------------------------------|------------------------|--------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger         | Starts the workflow manually       | —                      | Edit Fields              |                                                                                                     |
| Edit Fields                | Set                    | Defines AI image generation parameters | When clicking ‘Test workflow’ | Fal Flux                 | ## Set Parameter Here \nset Image Prompt and related settings                                       |
| Sticky Note                | Sticky Note            | Instruction to set image parameters | —                      | —                        | ## Set Parameter Here \nset Image Prompt and related settings                                       |
| Fal Flux                   | HTTP Request           | Sends image generation request to Fal.ai Flux API | Edit Fields             | Wait 3 Sec               | ### Generic Credential Type\n### Header : Authorization\nKey $FAL_KEY\"\n\nfor example:\nKey 6f2960baxxxxxxxxx |
| Sticky Note2               | Sticky Note            | Credential setup instructions       | —                      | —                        | ### Generic Credential Type\n### Header : Authorization\nKey $FAL_KEY\"\n\nfor example:\nKey 6f2960baxxxxxxxxx |
| Wait 3 Sec                 | Wait                   | Waits 3 seconds between status checks | Fal Flux, Completed? (False branch) | Check Status             |                                                                                                     |
| Check Status               | HTTP Request           | Checks image generation status      | Wait 3 Sec             | Completed?                |                                                                                                     |
| Completed?                 | If                     | Determines if image generation is complete | Check Status            | Get Image Result URL (True), Wait 3 Sec (False) |                                                                                                     |
| Get Image Result URL       | HTTP Request           | Retrieves the generated image URL   | Completed? (True)       | Download Image            | ## Set Drive Folder Here                                                                             |
| Download Image             | HTTP Request           | Downloads the generated image binary | Get Image Result URL    | Google Drive              | ## Set Drive Folder Here                                                                             |
| Google Drive               | Google Drive           | Uploads image to Google Drive folder | Download Image          | —                        | ## Set Drive Folder Here                                                                             |
| Sticky Note1               | Sticky Note            | Instruction to set Google Drive folder | —                      | —                        | ## Set Drive Folder Here                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a `Manual Trigger` node. No configuration needed. This starts the workflow manually.

2. **Create Set Node for Image Parameters (Edit Fields)**  
   - Add a `Set` node connected to the Manual Trigger.  
   - Define fields:  
     - `Prompt` (string): e.g., "Thai young woman net idol 25 yrs old, walking on the street"  
     - `Width` (number): 1024  
     - `Height` (number): 768  
     - `Steps` (number): 30  
     - `Guidance` (number): 3.5  

3. **Create HTTP Request Node to Call Fal.ai Flux API (Fal Flux)**  
   - Add an `HTTP Request` node connected to the `Set` node.  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/flux/dev`  
   - Authentication: Create or select a Generic Credential with HTTP Header Auth.  
     - Header Name: `Authorization`  
     - Header Value: `Key YOUR_FALAI_API_KEY` (replace with your actual Fal.ai API key)  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "prompt": "={{$json.Prompt}}",
       "image_size": {
         "width": {{$json.Width}},
         "height": {{$json.Height}}
       },
       "num_inference_steps": {{$json.Steps}},
       "guidance_scale": {{$json.Guidance}},
       "num_images": 1,
       "enable_safety_checker": true
     }
     ```
   - Enable "Send Body" as JSON.

4. **Create Wait Node (Wait 3 Sec)**  
   - Add a `Wait` node connected to `Fal Flux`.  
   - Configure to wait 3 seconds.

5. **Create HTTP Request Node to Check Status (Check Status)**  
   - Add an `HTTP Request` node connected to `Wait 3 Sec`.  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/flux/requests/{{$json.request_id}}/status`  
   - Use the same Generic Credential as `Fal Flux`.

6. **Create If Node to Check Completion (Completed?)**  
   - Add an `If` node connected to `Check Status`.  
   - Condition: Check if `{{$json.status}}` equals `"COMPLETED"`.  
   - True branch: Connect to next step.  
   - False branch: Connect back to `Wait 3 Sec` to continue polling.

7. **Create HTTP Request Node to Get Image URL (Get Image Result URL)**  
   - Add an `HTTP Request` node connected to the True branch of `Completed?`.  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/flux/requests/{{$json.request_id}}`  
   - Use the same Generic Credential as before.

8. **Create HTTP Request Node to Download Image (Download Image)**  
   - Add an `HTTP Request` node connected to `Get Image Result URL`.  
   - Method: GET  
   - URL: `{{$json.images[0].url}}`  
   - No authentication required.

9. **Create Google Drive Node to Upload Image (Google Drive)**  
   - Add a `Google Drive` node connected to `Download Image`.  
   - Operation: Upload File  
   - File Name: `{{$binary.data.fileName}}` (ensure binary property name matches downloaded data)  
   - Target Drive: "My Drive" or your specific Google Drive.  
   - Folder ID: Set the folder ID where you want to save the image.  
   - Authentication: Connect your Google Drive OAuth2 credentials.

10. **Add Sticky Notes for User Guidance (Optional)**  
    - Add sticky notes near `Edit Fields` node reminding to set image parameters.  
    - Add sticky notes near `Fal Flux` node documenting API key usage.  
    - Add sticky notes near `Google Drive` node reminding to set Drive folder ID.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Fal.ai API Key must be obtained from https://fal.ai and configured in n8n as a Generic Credential with HTTP Header Auth type.             | Fal.ai API Credential Setup                       |
| Google Drive OAuth2 credentials must be set up in n8n to enable uploading files to Drive folders.                                          | Google Drive OAuth2 Setup                         |
| To customize image generation, modify the `Edit Fields` node parameters: prompt, image dimensions, steps, and guidance scale.             | Workflow Customization                            |
| Consider adding notification nodes (email, Slack) after the Google Drive node to alert when images are ready.                             | Workflow Enhancement Ideas                        |
| For advanced usage, the workflow can be extended to support multiple images or destinations like Dropbox or Slack.                        | Scalability and Integration                       |
| Documentation and API reference for Fal.ai Flux API: https://docs.fal.ai/                                                                | Fal.ai API Documentation                          |

---

This comprehensive breakdown enables users and automation agents to fully understand, reproduce, and customize the AI image generation workflow with Fal.ai and Google Drive integration.