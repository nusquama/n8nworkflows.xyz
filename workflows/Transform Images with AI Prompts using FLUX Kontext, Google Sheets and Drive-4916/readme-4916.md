Transform Images with AI Prompts using FLUX Kontext, Google Sheets and Drive

https://n8nworkflows.xyz/workflows/transform-images-with-ai-prompts-using-flux-kontext--google-sheets-and-drive-4916


# Transform Images with AI Prompts using FLUX Kontext, Google Sheets and Drive

### 1. Workflow Overview

This workflow automates the generation of AI-enhanced, contextualized images using the FLUX Kontext API. The primary use case is to take image generation prompts stored in a Google Sheet, send them to FLUX Kontext’s AI image generation service, retrieve the generated images, upload them to Google Drive, and write back the resulting image URLs into the Google Sheet for easy access and tracking. The workflow supports manual triggering or periodic scheduling.

The workflow logic is grouped into the following blocks:

- **1.1 Input Reception and Triggering**: Receives either manual or scheduled triggers to start the process.
- **1.2 Google Sheets Data Retrieval**: Reads prompts and parameters for image generation from a Google Sheet.
- **1.3 AI Image Generation Request**: Sends a request to the FLUX Kontext API to generate an image based on the prompt and parameters.
- **1.4 Polling for Image Generation Completion**: Waits and polls the FLUX Kontext service to check if image generation is completed.
- **1.5 Retrieving and Uploading the Generated Image**: Downloads the generated image from FLUX Kontext and uploads it to Google Drive.
- **1.6 Updating Google Sheets with Result URL**: Updates the Google Sheet with the URL of the uploaded image for reference.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Triggering

- **Overview:** This block initiates the workflow either manually or on a defined schedule.
- **Nodes Involved:** 
  - When clicking ‘Test workflow’
  - Schedule Trigger
  - Sticky Note5 (instructional note)

- **Node Details:**

  - **When clicking ‘Test workflow’**
    - Type: Manual Trigger
    - Role: Allows manual start of the workflow for testing or on-demand execution.
    - Inputs: None
    - Outputs: Connects to “Get new image” node initiating processing.
    - Notes: Useful for debugging or manual runs.

  - **Schedule Trigger**
    - Type: Scheduled Trigger
    - Role: Periodically triggers the workflow, set to trigger every minute (configurable; note suggests 5 minutes).
    - Inputs: None
    - Outputs: Not directly connected in the JSON snippet but intended to trigger the main flow.
    - Configuration: Interval field set to minutes, triggers every 1 minute by default.
    - Notes: Recommended to set at 5-minute intervals for production use.

  - **Sticky Note5**
    - Purpose: Instructional note explaining that the workflow can be started manually or periodically, recommending 5-minute intervals for scheduling.

---

#### 1.2 Google Sheets Data Retrieval

- **Overview:** This block fetches image generation prompts and parameters from a Google Sheet where users maintain the input data.
- **Nodes Involved:**
  - Get new image
  - Sticky Note4 (instructional note)

- **Node Details:**

  - **Get new image**
    - Type: Google Sheets node (read operation)
    - Role: Reads rows from the Google Sheet that do not yet have a result (filters on empty “RESULT” column).
    - Configuration:
      - Document ID and Sheet GID point to a specific Google Sheet.
      - Filters rows where the “RESULT” column is empty.
      - Retrieves columns: PROMPT, IMAGE_URL, ASCPECT RATIO, OUTPUT FORMAT.
    - Inputs: Trigger node (“When clicking ‘Test workflow’” or scheduled trigger)
    - Outputs: Passes data to “Create Image” node.
    - Potential failures: Google API authentication issues, empty result sets, sheet structure changes.

  - **Sticky Note4**
    - Purpose: Provides instructions and link to a sample Google Sheet template.
    - Content highlights columns required:
      - PROMPT (image description)
      - IMAGE_URL (starting image URL)
      - ASPECT RATIO (image ratio)
      - OUTPUT FORMAT (jpeg or png)

---

#### 1.3 AI Image Generation Request

- **Overview:** This block sends a request to the FLUX Kontext API to generate an image with the specified prompt and parameters.
- **Nodes Involved:**
  - Create Image
  - Wait 60 sec.
  - Sticky Note6, Sticky Note7 (instructional notes)

- **Node Details:**

  - **Create Image**
    - Type: HTTP Request (POST)
    - Role: Sends a JSON payload to FLUX Kontext API to initiate image generation.
    - Configuration:
      - URL: https://queue.fal.run/fal-ai/flux-pro/kontext
      - POST body includes prompt, image_url, output_format, aspect_ratio, interpolated from Google Sheets data.
      - Authentication: HTTP Header Auth with Authorization header set to “Key YOURAPIKEY” (credential stored securely).
      - Headers: Content-Type: application/json
    - Inputs: Receives data from “Get new image”
    - Outputs: Passes response containing request_id to “Wait 60 sec.”
    - Failure modes: API key invalid, network error, malformed prompt, API limits.

  - **Wait 60 sec.**
    - Type: Wait node
    - Role: Pauses workflow for 60 seconds to allow image generation to complete before status check.
    - Inputs: From “Create Image”
    - Outputs: Connects to “Get status”
    - Notes: Prevents aggressive polling.

  - **Sticky Note6**
    - Instructions for obtaining and configuring FLUX Kontext API key.

  - **Sticky Note7**
    - Reminder to set the API key in “Create Image” node.

---

#### 1.4 Polling for Image Generation Completion

- **Overview:** This block polls the FLUX Kontext API to check the status of the image generation request until it is completed.
- **Nodes Involved:**
  - Get status
  - Completed?
  - Wait 60 sec.

- **Node Details:**

  - **Get status**
    - Type: HTTP Request (GET)
    - Role: Queries FLUX Kontext API for the status of the image generation task using the request_id.
    - Configuration:
      - URL constructed with request_id from “Create Image” node.
      - Uses same HTTP Header authentication as “Create Image”.
    - Inputs: From “Wait 60 sec.”
    - Outputs: To “Completed?” node
    - Potential failures: API timeout, bad request_id, auth failure.

  - **Completed?**
    - Type: IF node
    - Role: Checks if the status returned from “Get status” equals “COMPLETED”.
    - Inputs: From “Get status”
    - Outputs:
      - If true: proceeds to “Get Image Url”
      - If false: loops back to “Wait 60 sec.” to wait and poll again
    - Notes: Implements polling loop with wait intervals until completion.

  - **Wait 60 sec.** (same as above)
    - Used again here to pause before rechecking status if not completed.

---

#### 1.5 Retrieving and Uploading the Generated Image

- **Overview:** Once image generation is complete, this block retrieves the generated image’s URL from FLUX Kontext, downloads the image, and uploads it to Google Drive.
- **Nodes Involved:**
  - Get Image Url
  - Get Image File
  - Upload Image

- **Node Details:**

  - **Get Image Url**
    - Type: HTTP Request (GET)
    - Role: Retrieves detailed information about the generated image using the request_id.
    - Configuration:
      - URL: https://queue.fal.run/fal-ai/flux-pro/requests/{{request_id}}
      - Auth: HTTP Header Auth (same as before)
    - Inputs: From “Completed?” IF node (true branch)
    - Outputs: To “Get Image File”
    - Failure modes: API errors, invalid request_id.

  - **Get Image File**
    - Type: HTTP Request (GET)
    - Role: Downloads the actual image file from the URL provided in “Get Image Url” response.
    - Configuration:
      - URL dynamically set to the first image URL in the JSON response.
    - Inputs: From “Get Image Url”
    - Outputs: To “Upload Image”
    - Failure cases: Network issues, file not found, invalid URL.

  - **Upload Image**
    - Type: Google Drive node (upload)
    - Role: Uploads the downloaded image to a specified Google Drive folder.
    - Configuration:
      - File name generated dynamically based on current timestamp and output format.
      - Destination folder ID specified in parameters.
      - Uses OAuth2 Google Drive credentials.
    - Inputs: From “Get Image File”
    - Outputs: To “Update result”
    - Potential failures: Google Drive auth errors, quota limits, file size limits.

---

#### 1.6 Updating Google Sheets with Result URL

- **Overview:** This block updates the original Google Sheet row with the URL of the uploaded image, closing the loop for data tracking.
- **Nodes Involved:**
  - Update result

- **Node Details:**

  - **Update result**
    - Type: Google Sheets node (update operation)
    - Role: Writes the image URL back to the corresponding row in the Google Sheet.
    - Configuration:
      - Matches the row using the “row_number” field.
      - Updates the “RESULT” column with the uploaded image URL retrieved from “Get Image Url”.
      - Uses same Google Sheets credentials as reading node.
    - Inputs: From “Upload Image”
    - Outputs: None (end of flow)
    - Failure modes: Sheet access errors, row matching failure, API limits.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                        |
|------------------------|----------------------|---------------------------------------|------------------------|-------------------------|---------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger       | Manual start of the workflow           | None                   | Get new image            |                                                   |
| Schedule Trigger       | Schedule Trigger     | Periodic workflow triggering           | None                   | Not explicitly connected | Step 4 - recommended 5-min intervals              |
| Get new image          | Google Sheets (read) | Reads image prompts and parameters     | When clicking ‘Test workflow’ | Create Image            | Step 1 - Google Sheet setup instructions          |
| Create Image           | HTTP Request (POST)  | Sends image generation request to FLUX Kontext | Get new image           | Wait 60 sec.             | Step 2 - API key setup instructions                |
| Wait 60 sec.           | Wait                 | Pauses before polling status           | Create Image / Completed? (false branch) | Get status / Get status |                                                   |
| Get status             | HTTP Request (GET)   | Checks image generation status         | Wait 60 sec.            | Completed?               |                                                   |
| Completed?             | IF                   | Checks if image generation is completed | Get status              | Get Image Url / Wait 60 sec. (loop) |                                                   |
| Get Image Url          | HTTP Request (GET)   | Retrieves generated image details      | Completed? (true branch) | Get Image File           |                                                   |
| Get Image File         | HTTP Request (GET)   | Downloads the image file                | Get Image Url           | Upload Image             |                                                   |
| Upload Image           | Google Drive Upload  | Uploads image to Google Drive           | Get Image File          | Update result            |                                                   |
| Update result          | Google Sheets (update) | Writes back image URL to Google Sheet | Upload Image            | None                    |                                                   |
| Sticky Note3           | Sticky Note          | Overview of workflow purpose            | None                   | None                    |                                                   |
| Sticky Note4           | Sticky Note          | Google Sheet setup instructions         | None                   | None                    | Includes link: https://docs.google.com/spreadsheets/d/1N1Yg7FA4tQ8mDll5HLKqwPBuHj31AGDKwzAOg8mLQKs/edit?usp=sharing |
| Sticky Note5           | Sticky Note          | Workflow triggering instructions        | None                   | None                    |                                                   |
| Sticky Note6           | Sticky Note          | API Key setup instructions              | None                   | None                    | Includes link: https://fal.ai/                     |
| Sticky Note7           | Sticky Note          | Reminder to set API key                  | None                   | None                    |                                                   |
| Sticky Note            | Sticky Note          | Example prompt and result images         | None                   | None                    |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**
   - Add a **Manual Trigger** node named “When clicking ‘Test workflow’”.
   - Add a **Schedule Trigger** node named “Schedule Trigger” configured to run every 5 minutes (or as desired).

2. **Add Google Sheets Read Node**
   - Add a **Google Sheets** node named “Get new image”.
   - Configure it to **Read Rows** from your Google Sheet (use the provided template link).
   - Set a filter to only retrieve rows where the “RESULT” column is empty.
   - Map columns: PROMPT, IMAGE_URL, ASCPECT RATIO, OUTPUT FORMAT.
   - Connect trigger nodes (“When clicking ‘Test workflow’” and “Schedule Trigger”) to this node.

3. **Add HTTP Request Node to Create Image**
   - Add an **HTTP Request** node named “Create Image”.
   - Set method to POST.
   - URL: `https://queue.fal.run/fal-ai/flux-pro/kontext`
   - Add JSON body with keys: prompt, image_url, output_format, aspect_ratio, pulling values from Google Sheets node using expressions.
   - Set header “Content-Type” to “application/json”.
   - Configure HTTP Header Authentication Credential with:
     - Header name: Authorization
     - Header value: `Key YOURAPIKEY` (replace YOURAPIKEY with your actual key from https://fal.ai/)
   - Connect “Get new image” node output to this node.

4. **Add Wait Node**
   - Add a **Wait** node named “Wait 60 sec.”
   - Configure to pause for 60 seconds.
   - Connect “Create Image” node output to this node.

5. **Add HTTP Request Node to Get Status**
   - Add an **HTTP Request** node named “Get status”.
   - Method: GET
   - URL: `https://queue.fal.run/fal-ai/flux-pro/requests/{{ $json.request_id }}/status`
   - Use same HTTP Header Auth credential as “Create Image”.
   - Connect “Wait 60 sec.” node output to this node.

6. **Add IF Node to Check Completion**
   - Add an **IF** node named “Completed?”.
   - Condition: Check if `{{$json.status}}` equals “COMPLETED” (case sensitive).
   - Connect “Get status” node output to this IF node.

7. **Configure IF Node Branches**
   - On TRUE branch (completed):
     - Connect to a new **HTTP Request** node “Get Image Url”.
       - Method: GET
       - URL: `https://queue.fal.run/fal-ai/flux-pro/requests/{{ $json.request_id }}`
       - Use same HTTP Header Auth credential.
   - On FALSE branch:
     - Connect back to “Wait 60 sec.” node to loop.

8. **Add HTTP Request Node to Download Image**
   - Add an **HTTP Request** node named “Get Image File”.
   - Method: GET
   - URL: set to `{{$json.images[0].url}}` from “Get Image Url” node output.
   - Connect “Get Image Url” output to this node.

9. **Add Google Drive Upload Node**
   - Add a **Google Drive** node named “Upload Image”.
   - Configure to upload a file with:
     - File name: `{{$now.format('yyyyLLddHHmmss')}}.{{$json['OUTPUT FORMAT']}}`
     - Folder ID: Use your Google Drive folder ID where images will be stored.
   - Use Google Drive OAuth2 credentials.
   - Connect “Get Image File” output to this node.

10. **Add Google Sheets Update Node**
    - Add a **Google Sheets** node named “Update result”.
    - Configure to **Update Row** in the same Google Sheet.
    - Match rows by “row_number” from the “Get new image” node.
    - Update the “RESULT” column with the image URL from the “Get Image Url” node’s JSON (`images[0].url`).
    - Use Google Sheets OAuth2 credentials.
    - Connect “Upload Image” node output to this node.

11. **Connect Workflow**
    - Connect the nodes in order:
      - Trigger(s) → Get new image → Create Image → Wait 60 sec. → Get status → Completed? 
      - Completed? TRUE → Get Image Url → Get Image File → Upload Image → Update result
      - Completed? FALSE → Wait 60 sec. (loop)

12. **Add Sticky Notes (Optional)**
    - Add notes with instructions for users:
      - Google Sheet template and setup instructions.
      - API key setup and usage.
      - Workflow operation and scheduling recommendations.
      - Example prompts and results.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Google Sheet template for input data setup including PROMPT, IMAGE_URL, ASPECT RATIO, OUTPUT FORMAT columns   | https://docs.google.com/spreadsheets/d/1N1Yg7FA4tQ8mDll5HLKqwPBuHj31AGDKwzAOg8mLQKs/edit?usp=sharing       |
| Sign up for FLUX Kontext API and obtain API key                                                              | https://fal.ai/                                                                                            |
| Recommended scheduling interval for workflow execution                                                        | Suggested 5 minutes interval to balance prompt processing and API usage                                    |
| Example prompt and result images illustrating workflow’s output                                               | Inline images in Sticky Note (image of girl sleeping and resulting AI image)                               |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.