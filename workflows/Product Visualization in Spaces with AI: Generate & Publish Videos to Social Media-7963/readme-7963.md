Product Visualization in Spaces with AI: Generate & Publish Videos to Social Media

https://n8nworkflows.xyz/workflows/product-visualization-in-spaces-with-ai--generate---publish-videos-to-social-media-7963


# Product Visualization in Spaces with AI: Generate & Publish Videos to Social Media

### 1. Workflow Overview

This workflow automates the process of creating product visualization videos using AI, starting from user-uploaded photos and product descriptions. It generates AI-edited images, transforms these images into videos with animated camera movements, and finally publishes the resulting videos to multiple social media platforms (TikTok, Instagram Reels, YouTube). The workflow is designed for marketing teams, e-commerce sellers, or agencies aiming to create engaging visual content for product promotion with minimal manual effort.

The workflow logic is grouped into these main blocks:

- **1.1 Input Reception and Initialization**  
  Handles user input via a web form, sets API variables and prompts for AI services.

- **1.2 Original Image Upload and Preparation**  
  Uploads the user-provided photos to an image hosting service (imgbb) for external access.

- **1.3 AI Image Editing Generation**  
  Calls a Gemini 2.5 Flash AI image editing service to place the product image into a room scene.

- **1.4 Image Splitting and Secondary Upload**  
  Processes the AI-generated images, uploading them again to imgbb to prepare for video generation.

- **1.5 AI Video Generation and Monitoring**  
  Requests video creation from the FAL WAN AI image-to-video service, waits and polls for completion.

- **1.6 Video Download and Social Media Posting**  
  Downloads the completed video and uploads it to multiple social media platforms using UploadPost API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block captures user input (photos and product description) via a web form, then sets initial API parameters and AI prompt texts used in later AI calls.

- **Nodes Involved:**  
  - Photo Upload Form  
  - Set Prompts  
  - Set APIs Vars  
  - Merge

- **Node Details:**

  - **Photo Upload Form**  
    - Type: Form Trigger  
    - Role: Entry point; collects two photos and a product description via a web form at path `/generate-ad`.  
    - Config: Requires one mandatory photo (photo1), one optional photo (photo2), and a textarea for product description.  
    - Outputs: JSON containing binary files and text fields.  
    - Potential Failures: User submission errors, malformed uploads.

  - **Set Prompts**  
    - Type: Set node  
    - Role: Defines AI prompts for image editing and video animation.  
    - Config:  
      - `prompt-image-edit`: Instruction to place the painting in a room photograph realistically.  
      - `prompt-image-to-video`: Instructions for camera movement and framing in the generated video.  
    - Outputs variables used downstream.  
    - Edge Cases: Prompt syntax errors or misinterpretation by AI APIs.

  - **Set APIs Vars**  
    - Type: Set node  
    - Role: Sets configuration parameters for external services.  
    - Config:  
      - `number_of_images`: 1 (number of AI-generated images)  
      - `size_of_image`: "1024x1024" (resolution for AI image generation)  
      - `openai_image_model`: "gemini-25-flash-image" (AI model identifier)  
      - `format_image`: "webp" (preferred image format)  
      - `imgbb_api_key`: Placeholder for imgbb API key (must be replaced with a valid key)  
    - Potential Failures: Missing or invalid API keys.

  - **Merge**  
    - Type: Merge node (combine)  
    - Role: Combines outputs of `Photo Upload Form` and `Set APIs Vars` to prepare unified data for next steps.  
    - Config: Combines by position (index).  
    - Edge Cases: Mismatched input lengths causing merge failure.

---

#### 2.2 Original Image Upload and Preparation

- **Overview:**  
  Upload the original user photos to imgbb to obtain externally accessible URLs required by AI services.

- **Nodes Involved:**  
  - Upload Original Image to imgbb  
  - Upload Original Image to imgbb1  
  - Merge1

- **Node Details:**

  - **Upload Original Image to imgbb**  
    - Type: HTTP Request  
    - Role: Uploads `photo1` from user input to imgbb via multipart/form-data POST request.  
    - Config: Uses `imgbb_api_key` from variables; retries on failure enabled.  
    - Outputs: JSON including uploaded image URL (`data.url`).  
    - Potential Failures: Network issues, invalid API key, file size limits.

  - **Upload Original Image to imgbb1**  
    - Type: HTTP Request  
    - Role: Uploads `photo2` if provided, same method as above.  
    - Config: Same as above; input is `photo2`.  
    - Edge Cases: `photo2` may be missing; node should handle empty input gracefully.

  - **Merge1**  
    - Type: Merge node (combine)  
    - Role: Combines the two imgbb upload responses into one array for the next AI call.  
    - Config: Combine by position.  
    - Potential Failures: Missing second image causing index mismatch.

---

#### 2.3 AI Image Editing Generation

- **Overview:**  
  Calls the Gemini 2.5 Flash AI endpoint to generate an edited image where the product is placed realistically into a room photo.

- **Nodes Involved:**  
  - Gemini 2.5 Flash - Generate Image  
  - Separate Image Outputs

- **Node Details:**

  - **Gemini 2.5 Flash - Generate Image**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Gemini AI API with prompt and uploaded image URLs.  
    - Config:  
      - Uses prompt from `Set Prompts` (`prompt-image-edit`).  
      - Passes URLs of uploaded images from `Upload Original Image to imgbb` and `Upload Original Image to imgbb1`.  
      - Number of images defined in API vars.  
      - Authenticated via HTTP header auth (fal.ai credentials).  
    - Outputs: JSON containing an array of edited images.  
    - Potential Failures: Authentication errors, API downtime, malformed requests.

  - **Separate Image Outputs**  
    - Type: SplitOut node  
    - Role: Splits the array of images returned by Gemini AI into individual items for separate processing.  
    - Config: Splits on the field `images`.  
    - Edge Cases: Empty or malformed image array.

---

#### 2.4 Image Splitting and Secondary Upload

- **Overview:**  
  Processes each AI-edited image to upload it again to imgbb, obtaining URLs for video generation.

- **Nodes Involved:**  
  - HTTP Request (called "HTTP Request")  
  - Rename to photo  
  - Upload Image to imgbb

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Fetches the image URL from each separated image item (likely to retrieve or validate).  
    - Config: URL taken from the split image object.  
    - Output: JSON with image URL.  
    - Edge Cases: URL not accessible or returned 404.

  - **Rename to photo**  
    - Type: Code (JavaScript)  
    - Role: Re-maps the JSON and binary fields to standardize to `{json.url, binary.photo2}` for upload.  
    - Config: Uses JS to transform incoming items.  
    - Edge Cases: Missing binary data causing errors.

  - **Upload Image to imgbb**  
    - Type: HTTP Request  
    - Role: Uploads the processed image as binary `photo2` to imgbb.  
    - Config: Uses imgbb API key; multipart/form-data POST.  
    - Outputs: JSON with uploaded image URL for video generation.  
    - Edge Cases: API limits, network failures.

---

#### 2.5 AI Video Generation and Monitoring

- **Overview:**  
  Creates a video from the AI-edited image using the FAL WAN AI image-to-video API, then polls the status until completion.

- **Nodes Involved:**  
  - FAL WAN i2v (Queue)  
  - Wait i2v  
  - FAL WAN i2v (status)  
  - Animation Completed?  
  - FAL WAN i2v (Result)

- **Node Details:**

  - **FAL WAN i2v (Queue)**  
    - Type: HTTP Request  
    - Role: Sends a video generation request with image URL and video animation prompt.  
    - Config:  
      - Uses `image_url` from uploaded AI image.  
      - `prompt` from `Set Prompts` (`prompt-image-to-video`).  
      - Video parameters: 96 frames, 24 fps, 480p resolution.  
      - Authenticated with fal.ai credentials.  
    - Outputs: JSON with `request_id` for status polling.  
    - Potential Failures: Auth errors, malformed request, service unavailability.

  - **Wait i2v**  
    - Type: Wait node  
    - Role: Pauses workflow for 30 seconds between status checks.  
    - Config: Fixed 30 seconds delay.  
    - Edge Cases: Timeout if video takes too long.

  - **FAL WAN i2v (status)**  
    - Type: HTTP Request  
    - Role: Polls the status of the video generation request by `request_id`.  
    - Config: GET request to status endpoint with header auth.  
    - Outputs: Status JSON with state (e.g., "COMPLETED").  
    - Edge Cases: Request ID invalid, API errors.

  - **Animation Completed?**  
    - Type: If node  
    - Role: Branches workflow depending on whether video status is "COMPLETED".  
    - Config: Checks if `$json.status == "COMPLETED"`.  
    - True branch proceeds to fetch result; False branch loops back to wait.

  - **FAL WAN i2v (Result)**  
    - Type: HTTP Request  
    - Role: Retrieves final video metadata and URL once animation is completed.  
    - Config: GET request using `request_id`.  
    - Outputs: JSON with video URL for download.  
    - Potential Failures: Failure to retrieve video info.

---

#### 2.6 Video Download and Social Media Posting

- **Overview:**  
  Downloads the finalized video file and uploads it to social media platforms (TikTok, Instagram Reels, YouTube) via UploadPost API.

- **Nodes Involved:**  
  - Download Final Video  
  - Upload Post

- **Node Details:**

  - **Download Final Video**  
    - Type: HTTP Request  
    - Role: Downloads the video file from the URL provided by the previous node.  
    - Config: GET request to `$json.video.url`.  
    - Output: Binary video data.  
    - Potential Failures: URL not accessible, timeout, large file download issues.

  - **Upload Post**  
    - Type: UploadPost node (custom integration)  
    - Role: Uploads the video to TikTok, Instagram (as Reels), and YouTube.  
    - Config:  
      - User credential selected.  
      - Title and caption fields (caption empty here).  
      - Specifies platforms: TikTok, Instagram, YouTube.  
      - Instagram media type set to Reels.  
    - Credentials: UploadPost API configured with user credentials.  
    - Edge Cases: Platform API rate limits, credential expiry, video format incompatibility.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                            | Input Node(s)                         | Output Node(s)                     | Sticky Note                                                                 |
|-----------------------------|-------------------------|--------------------------------------------|-------------------------------------|-----------------------------------|-----------------------------------------------------------------------------|
| Photo Upload Form            | Form Trigger            | Receive photos and description input       | -                                   | Set Prompts, Merge                 | Upload photo and description                                                |
| Set Prompts                 | Set                     | Define AI prompts for image and video      | Photo Upload Form                   | Set APIs Vars                     |                                                                             |
| Set APIs Vars               | Set                     | Initialize API variables and keys           | Set Prompts                        | Merge                            |                                                                             |
| Merge                      | Merge                   | Combine form input and API vars             | Photo Upload Form, Set APIs Vars   | Upload Original Image to imgbb     |                                                                             |
| Upload Original Image to imgbb | HTTP Request           | Upload photo1 to imgbb                      | Merge                             | Merge1                          |                                                                             |
| Upload Original Image to imgbb1| HTTP Request           | Upload photo2 to imgbb                      | Merge                             | Merge1                          |                                                                             |
| Merge1                     | Merge                   | Combine imgbb upload results                 | Upload Original Image to imgbb, Upload Original Image to imgbb1 | Gemini 2.5 Flash - Generate Image |                                                                             |
| Gemini 2.5 Flash - Generate Image | HTTP Request           | Generate AI edited image                      | Merge1                           | Separate Image Outputs            |                                                                             |
| Separate Image Outputs      | SplitOut                | Split AI images array into individual items | Gemini 2.5 Flash - Generate Image  | HTTP Request                    |                                                                             |
| HTTP Request               | HTTP Request            | Fetch each image URL                         | Separate Image Outputs             | Rename to photo                 |                                                                             |
| Rename to photo            | Code                    | Rename and remap image data for upload      | HTTP Request                      | Upload Image to imgbb            |                                                                             |
| Upload Image to imgbb       | HTTP Request            | Upload AI-edited image to imgbb              | Rename to photo                   | FAL WAN i2v (Queue)              |                                                                             |
| FAL WAN i2v (Queue)         | HTTP Request            | Request video generation from AI service     | Upload Image to imgbb             | Wait i2v                       |                                                                             |
| Wait i2v                   | Wait                    | Wait 30 seconds between status polls        | FAL WAN i2v (Queue), Animation Completed? (false branch) | FAL WAN i2v (status)           |                                                                             |
| FAL WAN i2v (status)        | HTTP Request            | Poll video generation status                  | Wait i2v                        | Animation Completed?             |                                                                             |
| Animation Completed?        | If                      | Check if video generation is completed       | FAL WAN i2v (status)             | FAL WAN i2v (Result), Wait i2v  |                                                                             |
| FAL WAN i2v (Result)        | HTTP Request            | Retrieve final video URL and metadata         | Animation Completed? (true branch) | Download Final Video             |                                                                             |
| Download Final Video        | HTTP Request            | Download the generated video                   | FAL WAN i2v (Result)             | Upload Post                    |                                                                             |
| Upload Post                | UploadPost              | Upload video to TikTok, Instagram (Reels), YouTube | Download Final Video             | -                               |                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "Photo Upload Form"  
   - Path: `/generate-ad`  
   - Form fields:  
     - `photo1` (file, required, single file)  
     - `photo2` (file, optional, single file)  
     - `product description` (textarea)  
   - Form title and description as desired.

2. **Add a Set node** "Set Prompts"  
   - Set two string fields:  
     - `prompt-image-edit`: Place the painting in the room on the back wall, respecting the painting perfectly and the background room and the camera frame in the photo of the room.  
     - `prompt-image-to-video`: Camera pan right and gentle truck/dolly right for the whole clip; the painting stays centered and in focus; reveal the right edge and frame depth; camera height aligned to painting center, medium-wide shot, 35 mm lens, soft gallery lighting; straight vertical lines, natural slight motion blur; no zoom.

3. **Add a Set node** "Set APIs Vars"  
   - Variables and values:  
     - `number_of_images` (number): 1  
     - `size_of_image` (string): "1024x1024"  
     - `openai_image_model` (string): "gemini-25-flash-image"  
     - `format_image` (string): "webp"  
     - `imgbb_api_key` (string): Set your valid imgbb API key here.

4. **Add a Merge node** "Merge"  
   - Mode: Combine by position  
   - Connect outputs of "Photo Upload Form" and "Set APIs Vars" into this node.

5. **Add two HTTP Request nodes** for uploading images:  
   - "Upload Original Image to imgbb" (upload `photo1`)  
     - Method: POST  
     - URL: `https://api.imgbb.com/1/upload`  
     - Body type: multipart-form-data  
     - Parameters:  
       - `image`: formBinaryData, input field `photo1`  
       - `key`: value from `imgbb_api_key` variable  
     - Enable retry on fail.  
   - "Upload Original Image to imgbb1" (upload `photo2`)  
     - Same config, input field `photo2`.

6. **Add a Merge node** "Merge1"  
   - Mode: Combine by position  
   - Connect outputs of both imgbb upload nodes.

7. **Add an HTTP Request node** "Gemini 2.5 Flash - Generate Image"  
   - Method: POST  
   - URL: `https://fal.run/fal-ai/gemini-25-flash-image/edit`  
   - Body type: raw JSON with fields:  
     - `prompt`: from `Set Prompts` (`prompt-image-edit`)  
     - `image_urls`: array of URLs from "Upload Original Image to imgbb" and "Upload Original Image to imgbb1"  
     - `num_images`: from `Set APIs Vars`  
   - Authentication: HTTP Header Auth with fal.ai credentials  
   - Connect output of "Merge1" to this node.

8. **Add a SplitOut node** "Separate Image Outputs"  
   - Field to split out: `images`  
   - Connect from Gemini node.

9. **Add an HTTP Request node** "HTTP Request"  
   - Purpose: Fetch or validate each image URL  
   - URL: dynamic from split image JSON field `url`.

10. **Add a Code node** "Rename to photo"  
    - JS code to map items to standardized format with binary `photo2` and JSON `url`.

11. **Add an HTTP Request node** "Upload Image to imgbb"  
    - Same multipart-form-data POST to imgbb  
    - Upload `photo2` binary with key from variables.

12. **Add an HTTP Request node** "FAL WAN i2v (Queue)"  
    - POST to `https://queue.fal.run/fal-ai/wan/v2.2-a14b/image-to-video`  
    - Body JSON:  
      - `image_url`: from uploaded AI image URL  
      - `prompt`: from `Set Prompts` (`prompt-image-to-video`)  
      - `num_frames`: 96  
      - `frames_per_second`: 24  
      - `resolution`: "480p"  
    - Auth: HTTP Header Auth with fal.ai credentials.

13. **Add a Wait node** "Wait i2v"  
    - Wait 30 seconds, used for polling.

14. **Add an HTTP Request node** "FAL WAN i2v (status)"  
    - GET status endpoint using `request_id` from queue node.

15. **Add an If node** "Animation Completed?"  
    - Condition: `$json.status == "COMPLETED"`  
    - True branch: proceed to fetch result  
    - False branch: loop back to "Wait i2v"

16. **Add an HTTP Request node** "FAL WAN i2v (Result)"  
    - GET final video info endpoint with `request_id`.

17. **Add an HTTP Request node** "Download Final Video"  
    - GET video URL from previous node  
    - Configure to return binary data (video file).

18. **Add an UploadPost node** "Upload Post"  
    - Configure user credentials for UploadPost API.  
    - Set title, empty caption, and select platforms: TikTok, Instagram (Reels), YouTube.  
    - Connect input binary video data.

19. **Connect nodes according to the original workflow's connection graph**, ensuring data flows correctly from form input to final upload.

20. **Set credentials** for:  
    - imgbb API key (in Set APIs Vars node)  
    - fal.ai HTTP header auth (for Gemini and WAN i2v nodes)  
    - UploadPost API user credentials

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                       |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| The workflow relies on fal.ai APIs for AI image editing and video generation requiring valid creds. | https://fal.run/                                                                      |
| UploadPost node is a custom integration for social media video uploads. Ensure valid credentials.   | https://uploadpost.com/ (platform info)                                              |
| imgbb is used for temporary image hosting to provide URLs for AI services; requires API key setup.  | https://api.imgbb.com/                                                                |
| The video generation includes a 30-second wait and polling loop to accommodate processing time.     |                                                                                      |
| Prompt texts are tailored for product visualization in spaces; adjust as needed for different use cases. |                                                                                      |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal or protected content. All data processed is legal and public.