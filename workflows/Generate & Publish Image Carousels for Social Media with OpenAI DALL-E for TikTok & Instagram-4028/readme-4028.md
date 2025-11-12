Generate & Publish Image Carousels for Social Media with OpenAI DALL-E for TikTok & Instagram

https://n8nworkflows.xyz/workflows/generate---publish-image-carousels-for-social-media-with-openai-dall-e-for-tiktok---instagram-4028


# Generate & Publish Image Carousels for Social Media with OpenAI DALL-E for TikTok & Instagram

### 1. Workflow Overview

This workflow automates the generation and publishing of a five-image carousel for social media platforms TikTok and Instagram using OpenAI’s GPT-based image generation and editing APIs. It sequentially creates a thematic series of images that visually narrate a story, generates a concise AI-driven caption based on the image prompts, and publishes the combined carousel with the caption to both platforms.

**Target Use Cases:**  
- Social Media Managers and Content Creators aiming for efficient, visually cohesive carousel posts.  
- Digital Marketers seeking automated campaign asset generation.  
- Small Businesses requiring unique promotional content without graphic design expertise.

**Logical Blocks:**  
- **1.1 Input Initialization:** Manual trigger and prompt setup for the image sequence.  
- **1.2 AI Caption Generation:** Uses OpenAI GPT to create a social media caption from prompts.  
- **1.3 API Variable Configuration:** Sets parameters for image generation calls.  
- **1.4 Sequential Image Generation and Processing:** Generates image 1 (generation), then images 2-5 (edits), converting results to binary and renaming for downstream handling.  
- **1.5 Data Aggregation:** Merges all generated images into a single item.  
- **1.6 Multi-Platform Publishing:** Sends the merged images and caption to Instagram and TikTok with platform-specific configurations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** Starts the workflow manually and sets the five distinct prompts that define the narrative for the image carousel.  
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’`  
  - `Set All Prompts`  

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Inputs: None  
    - Outputs: To `Set All Prompts`  
    - Failures: None expected; manual trigger.  

  - **Set All Prompts**  
    - Type: Set  
    - Role: Defines five specific text prompts for each image in the carousel, describing scenes in detailed, artistic language tailored for vertical TikTok carousels (9:16).  
    - Configuration: Assigns string values to fields `prompt1` through `prompt5`.  
    - Inputs: From manual trigger  
    - Outputs: To `Generate Description for Tiktok and Instagram`  
    - Edge Cases: Prompts must be valid and appropriate for image generation; overly long or complex prompts may impact generation quality.

#### 2.2 AI Caption Generation

- **Overview:** Generates a concise social media caption (≤ 90 characters) summarizing the carousel story using GPT-4.1.  
- **Nodes Involved:**  
  - `Generate Description for Tiktok and Instagram`  

- **Node Details:**  
  - Type: OpenAI (Langchain) node  
  - Role: Uses GPT-4.1 model to produce a short caption based on the concatenated image prompts.  
  - Configuration:  
    - System prompt defines role as expert social media caption writer.  
    - User message embeds all five prompts and requests a description only, capped at 90 characters.  
  - Inputs: From `Set All Prompts` (prompts JSON)  
  - Outputs: To `Set API Variables`  
  - Credentials: OpenAI API key required.  
  - Edge Cases: API rate limits, potential incomplete or nonsensical captions if prompts are ambiguous.

#### 2.3 API Variable Configuration

- **Overview:** Prepares and sets variables controlling image generation parameters such as model type, image size, and response format.  
- **Nodes Involved:**  
  - `Set API Variables`  

- **Node Details:**  
  - Type: Set  
  - Role: Defines OpenAI image generation parameters (model, number of images, size, response format) and passes the entire JSON from prompts as `image_prompt`.  
  - Configuration:  
    - `openai_image_model` default: `gpt-image-1` (placeholder for actual OpenAI image model like `dall-e-2`)  
    - `number_of_images`: 1 (per call)  
    - `size_of_image`: 1024x1536 (vertical)  
    - `response_format_image`: b64_json (base64 encoded JSON)  
  - Inputs: From `Generate Description for Tiktok and Instagram`  
  - Outputs: To `OpenAI - Generate Image 1`  
  - Edge Cases: Parameters must align with OpenAI API specs; incorrect model or size values cause API failures.

#### 2.4 Sequential Image Generation and Processing

- **Overview:** Generates the first image from scratch using the generation endpoint, then creates images 2 to 5 by editing the previously generated image with new prompts. Each image is separated from batch output, converted to binary, and renamed for consistent referencing.  
- **Nodes Involved:**  
  - `OpenAI - Generate Image 1`  
  - `Separate Image Outputs 1`  
  - `Convert to File 1`  
  - `Change name to photo1`  
  - `OpenAI - Generate Image 2` through `OpenAI - Generate Image 5` and corresponding `Separate Image Outputs`, `Convert to File`, and `Change name to photoX` nodes  
  - `Merge`  

- **Node Details:**  
  For each image (1 to 5), the following pattern applies:

  - **OpenAI Image Generation / Edit Nodes:**  
    - Type: HTTP Request  
    - Role: Calls OpenAI’s image generation (`/v1/images/generations`) for image 1 and image editing (`/v1/images/edits`) for images 2-5.  
    - Configuration:  
      - Uses multipart/form-data for edits, JSON for generation.  
      - Passes appropriate prompt (prompt1 for image 1, prompt2 for image 2, etc.)  
      - Passes previously generated image binary (for images 2-5) as `image[]` parameter.  
      - Uses OpenAI API key credentials.  
    - Inputs/Outputs:  
      - Image 1 input: API variables and prompt1.  
      - Image N input: previous image binary and promptN.  
      - Output: JSON response with base64 encoded image data array.  
    - Edge Cases: Auth errors, invalid model, prompt issues, large payloads causing timeout or memory errors.

  - **Separate Image Outputs Nodes:**  
    - Type: SplitOut  
    - Role: Extracts individual image data from the OpenAI API response array.  
    - Inputs: Response from HTTP request node  
    - Outputs: Single image JSON item per output  
    - Edge Cases: Empty or malformed response arrays.

  - **Convert to File Nodes:**  
    - Type: ConvertToFile  
    - Role: Converts base64 JSON image data into binary file format for downstream usage.  
    - Inputs: Base64 JSON data  
    - Outputs: Binary data containing image file  
    - Edge Cases: Data corruption or invalid base64 strings.

  - **Change Name to photoX Nodes:**  
    - Type: Code  
    - Role: Rename binary field key from `data` to `photoX` (photo1, photo2, etc.) to distinguish images.  
    - Inputs: Binary file from Convert to File node  
    - Outputs: Item with renamed binary data key  
    - Edge Cases: Missing binary data or unexpected structure causing script errors.

  - **Merge Node:**  
    - Type: Merge  
    - Role: Combines all renamed photo binary items (photos 1–5) into one item with multiple binary fields for upload.  
    - Inputs: Receives each photo’s renamed binary item on separate inputs  
    - Outputs: Single merged item with all photos  
    - Edge Cases: Mismatched input counts or missing photos.

#### 2.5 Data Aggregation

- **Overview:** Merges all generated photos into a single item and prepares for multi-platform publishing.  
- **Nodes Involved:**  
  - `Send as 1 merged file`  

- **Node Details:**  
  - Type: Code  
  - Role: Iterates over incoming items each containing one photo binary, merges their binaries into one item with keys photo1 through photo5.  
  - Inputs: From `Merge` node output  
  - Outputs: Single item with all photos in binary  
  - Edge Cases: Missing binary data, input items count mismatch.

#### 2.6 Multi-Platform Publishing

- **Overview:** Uploads the merged image carousel and AI-generated caption to Instagram and TikTok via `upload-post.com` API, handling caption length limits and platform-specific options like auto music for TikTok.  
- **Nodes Involved:**  
  - `POST TO INSTAGRAM`  
  - `POST TO TIKTOK`  

- **Node Details:**  

  - **POST TO INSTAGRAM:**  
    - Type: HTTP Request  
    - Role: Sends multipart-form data to upload photos as Instagram carousel.  
    - Configuration:  
      - URL: `https://api.upload-post.com/api/upload_photos`  
      - Authorization header: API key (replace `"Apikey add_api_key_here"` with real key)  
      - Fields: title (caption from GPT), user (upload_post), platform: instagram, photos[] (photo1 to photo5 binaries)  
    - Inputs: From `Send as 1 merged file` (images) and caption node  
    - Outputs: API response  
    - Edge Cases: Auth failures, network errors, API limits, invalid photo format.

  - **POST TO TIKTOK:**  
    - Type: HTTP Request  
    - Role: Similar to Instagram upload, but posts to TikTok with caption truncated to ≤ 90 characters and can auto-add music.  
    - Configuration:  
      - URL: same as Instagram  
      - Authorization header: API key  
      - Fields: title (truncated caption), user, platform: tiktok, photos[], auto_add_music=true  
    - Inputs: From `Send as 1 merged file` and caption node  
    - Outputs: API response  
    - Edge Cases: Caption truncation logic failures, API auth issues, network problems.

---

### 3. Summary Table

| Node Name                          | Node Type                 | Functional Role                              | Input Node(s)                            | Output Node(s)                             | Sticky Note                                          |
|-----------------------------------|---------------------------|----------------------------------------------|----------------------------------------|--------------------------------------------|------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger            | Workflow manual start point                   | -                                      | Set All Prompts                            |                                                      |
| Set All Prompts                   | Set                       | Defines all 5 image prompts                    | When clicking ‘Test workflow’           | Generate Description for Tiktok and Instagram |                                                      |
| Generate Description for Tiktok and Instagram | OpenAI (Langchain)        | Generates social media caption ≤ 90 chars     | Set All Prompts                         | Set API Variables                           |                                                      |
| Set API Variables                | Set                       | Sets image generation parameters               | Generate Description for Tiktok and Instagram | OpenAI - Generate Image 1                  |                                                      |
| OpenAI - Generate Image 1        | HTTP Request              | Generates first image from prompt1             | Set API Variables                      | Separate Image Outputs 1                    |                                                      |
| Separate Image Outputs 1          | SplitOut                  | Extracts image1 from API response               | OpenAI - Generate Image 1              | Convert to File 1                          |                                                      |
| Convert to File 1                | ConvertToFile             | Converts base64 JSON to binary file image      | Separate Image Outputs 1                | Change name to photo1                      |                                                      |
| Change name to photo1             | Code                      | Renames binary key to photo1                    | Convert to File 1                      | OpenAI - Generate Image 2, Merge           |                                                      |
| OpenAI - Generate Image 2        | HTTP Request              | Edits image1 with prompt2 to create image2     | Change name to photo1                   | Separate Image Outputs 2                    |                                                      |
| Separate Image Outputs 2          | SplitOut                  | Extracts image2 from API response               | OpenAI - Generate Image 2              | Convert to File 2                          |                                                      |
| Convert to File 2                | ConvertToFile             | Converts base64 JSON to binary file image      | Separate Image Outputs 2                | Change name to photo2                      |                                                      |
| Change name to photo2             | Code                      | Renames binary key to photo2                    | Convert to File 2                      | Merge, OpenAI - Generate Image 3            |                                                      |
| OpenAI - Generate Image 3        | HTTP Request              | Edits image2 with prompt3 to create image3     | Change name to photo2                   | Separate Image Outputs 3                    |                                                      |
| Separate Image Outputs 3          | SplitOut                  | Extracts image3 from API response               | OpenAI - Generate Image 3              | Convert to File 3                          |                                                      |
| Convert to File 3                | ConvertToFile             | Converts base64 JSON to binary file image      | Separate Image Outputs 3                | Change name to photo3                      |                                                      |
| Change name to photo3             | Code                      | Renames binary key to photo3                    | Convert to File 3                      | Merge, OpenAI - Generate Image 4            |                                                      |
| OpenAI - Generate Image 4        | HTTP Request              | Edits image3 with prompt4 to create image4     | Change name to photo3                   | Separate Image Outputs 4                    |                                                      |
| Separate Image Outputs 4          | SplitOut                  | Extracts image4 from API response               | OpenAI - Generate Image 4              | Convert to File 4                          |                                                      |
| Convert to File 4                | ConvertToFile             | Converts base64 JSON to binary file image      | Separate Image Outputs 4                | Change name to photo4                      |                                                      |
| Change name to photo4             | Code                      | Renames binary key to photo4                    | Convert to File 4                      | Merge, OpenAI - Generate Image 5            |                                                      |
| OpenAI - Generate Image 5        | HTTP Request              | Edits image4 with prompt5 to create image5     | Change name to photo4                   | Separate Image Outputs 5                    |                                                      |
| Separate Image Outputs 5          | SplitOut                  | Extracts image5 from API response               | OpenAI - Generate Image 5              | Convert to File 5                          |                                                      |
| Convert to File 5                | ConvertToFile             | Converts base64 JSON to binary file image      | Separate Image Outputs 5                | Change name to photo5                      |                                                      |
| Change name to photo5             | Code                      | Renames binary key to photo5                    | Convert to File 5                      | Merge                                     |                                                      |
| Merge                          | Merge                     | Combines photo1 to photo5 into single item      | Change name to photo1-5 (various)      | Send as 1 merged file                      |                                                      |
| Send as 1 merged file            | Code                      | Merges all photo binaries into one item         | Merge                                 | POST TO INSTAGRAM, POST TO TIKTOK          |                                                      |
| POST TO INSTAGRAM               | HTTP Request              | Uploads carousel and caption to Instagram       | Send as 1 merged file                  | -                                          | Replace “Apikey add_api_key_here” with your API key. |
| POST TO TIKTOK                 | HTTP Request              | Uploads carousel and caption to TikTok, auto-adds music | Send as 1 merged file              | -                                          | Replace “Apikey add_api_key_here” with your API key. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - Purpose: Manual start.

2. **Create Set Node for Prompts:**  
   - Name: `Set All Prompts`  
   - Type: Set  
   - Assign five string fields: `prompt1` to `prompt5` with detailed narrative prompts for each carousel image.  
   - Connect from manual trigger.

3. **Create OpenAI GPT Node for Caption:**  
   - Name: `Generate Description for Tiktok and Instagram`  
   - Type: OpenAI (Langchain)  
   - Credentials: Configure OpenAI API key credentials.  
   - Model: `gpt-4.1`  
   - Messages:  
     - System: "You are an expert assistant for creating good social media video titles."  
     - User: Provide concatenated prompts `prompt1` to `prompt5` and request a ≤ 90 character description only.  
   - Connect from `Set All Prompts`.

4. **Create Set Node for API Variables:**  
   - Name: `Set API Variables`  
   - Type: Set  
   - Assign fields:  
     - `image_prompt` = full JSON from prompts (for reference)  
     - `number_of_images` = 1  
     - `size_of_image` = `1024x1536` (vertical format)  
     - `openai_image_model` = `gpt-image-1` (replace with valid OpenAI image model like `dall-e-2`)  
     - `response_format_image` = `b64_json`  
   - Connect from `Generate Description for Tiktok and Instagram`.

5. **Create OpenAI HTTP Request Node for Image 1 Generation:**  
   - Name: `OpenAI - Generate Image 1`  
   - Type: HTTP Request  
   - URL: `https://api.openai.com/v1/images/generations`  
   - Method: POST  
   - Authentication: Use OpenAI API credentials  
   - Headers: Content-Type: application/json  
   - Body (JSON):  
     ```json
     {
       "model": "{{ $json.openai_image_model }}",
       "prompt": "{{ $('Set All Prompts').item.json.prompt1 }}",
       "n": {{ $json.number_of_images }},
       "size": "{{ $json.size_of_image }}"
     }
     ```  
   - Connect from `Set API Variables`.

6. **Create SplitOut Node:**  
   - Name: `Separate Image Outputs 1`  
   - Type: SplitOut  
   - Field to split out: `data` (array in OpenAI response)  
   - Connect from `OpenAI - Generate Image 1`.

7. **Create ConvertToFile Node:**  
   - Name: `Convert to File 1`  
   - Type: ConvertToFile  
   - Operation: toBinary  
   - Source property: `b64_json`  
   - Connect from `Separate Image Outputs 1`.

8. **Create Code Node to Rename Binary:**  
   - Name: `Change name to photo1`  
   - Type: Code  
   - JavaScript: Rename binary field from `data` to `photo1`.  
   - Connect from `Convert to File 1`.

9. **Repeat Steps 5–8 for Images 2 to 5 with modifications:**  
   - Use HTTP Request node to call OpenAI `/v1/images/edits` endpoint.  
   - Method: POST  
   - Content Type: multipart/form-data  
   - Body parameters:  
     - model: `{{ $('Set API Variables').item.json.openai_image_model }}`  
     - prompt: `{{ $('Set All Prompts').item.json.promptN }}` (N=2..5)  
     - n: `{{ $('Set API Variables').item.json.number_of_images }}`  
     - size: `{{ $('Set API Variables').item.json.size_of_image }}`  
     - image[]: binary input from previous photo (photo1 for image2, photo2 for image3, etc.)  
   - Connect `Change name to photo1` → `OpenAI - Generate Image 2` → `Separate Image Outputs 2` → `Convert to File 2` → `Change name to photo2`, and so forth, chaining images 2 through 5 sequentially.  
   - Rename binaries to `photo2`, `photo3`, `photo4`, `photo5` respectively.

10. **Create Merge Node:**  
    - Name: `Merge`  
    - Type: Merge  
    - Number of Inputs: 6 (5 images + initial trigger input)  
    - Mode: Merge by Index  
    - Connect all `Change name to photoX` nodes to this merge node.

11. **Create Code Node to Merge All Binary Photos into One Item:**  
    - Name: `Send as 1 merged file`  
    - Type: Code  
    - JavaScript: Iterate all incoming items, merge their binary fields into one item with keys photo1..photo5.  
    - Connect from `Merge`.

12. **Create HTTP Request Node to POST to Instagram:**  
    - Name: `POST TO INSTAGRAM`  
    - Type: HTTP Request  
    - URL: `https://api.upload-post.com/api/upload_photos`  
    - Method: POST  
    - Content-Type: multipart-form-data  
    - Authorization header: `Apikey YOUR_UPLOAD_POST_API_KEY`  
    - Body parameters:  
      - `title`: caption from `Generate Description for Tiktok and Instagram` node  
      - `user`: your upload_post user ID  
      - `platform[]`: `instagram`  
      - `photos[]`: binary data fields `photo1` to `photo5`  
    - Connect from `Send as 1 merged file`.

13. **Create HTTP Request Node to POST to TikTok:**  
    - Name: `POST TO TIKTOK`  
    - Type: HTTP Request  
    - URL and method same as Instagram node  
    - Authorization header: same as Instagram  
    - Body parameters:  
      - `title`: truncated caption (≤90 chars) from `Generate Description for Tiktok and Instagram` with truncation logic in expression  
      - `user`: your upload_post user ID  
      - `platform[]`: `tiktok`  
      - `photos[]`: binaries photo1 to photo5  
      - `auto_add_music`: `true` (optional, can be set to false)  
    - Connect from `Send as 1 merged file`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                         |
|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Replace `"Apikey add_api_key_here"` in POST TO INSTAGRAM and POST TO TIKTOK nodes with your actual `upload-post.com` API key.     | API key configuration                                  |
| The OpenAI model identifier `"gpt-image-1"` is a placeholder; use valid OpenAI image generation/edit models such as `dall-e-2`.   | OpenAI models documentation: https://platform.openai.com/docs/models |
| TikTok caption length is limited to 90 characters; the workflow truncates exceeding captions gracefully with an ellipsis.        | Social media platform constraints                       |
| The workflow assumes vertical carousel images (1024x1536) suitable for TikTok and Instagram reels/carousels.                      | Image size parameter                                    |
| The `auto_add_music` feature in TikTok upload node enables automatic background music addition; toggle as needed.                 | TikTok platform feature                                 |
| This workflow requires valid API keys and credentials for OpenAI and `upload-post.com` services configured in n8n.                | Credential setup                                       |

---

**Disclaimer:**  
The text and workflow content provided stem exclusively from an automated n8n workflow using legal and publicly available data and APIs. It complies with current content policies, containing no illegal, offensive, or protected material.