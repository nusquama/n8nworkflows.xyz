Product Video Creator with Nano Banana & Veo 3.1 via Telegram

https://n8nworkflows.xyz/workflows/product-video-creator-with-nano-banana---veo-3-1-via-telegram-9896


# Product Video Creator with Nano Banana & Veo 3.1 via Telegram

### 1. Workflow Overview

This workflow automates the creation of professional product marketing videos from user-submitted product photos via Telegram. It leverages AI models (Google Gemini / Nano Banana for image analysis and enhancement, and Veo 3.1 for video generation) to analyze product images, generate enhanced visuals, and produce short promotional videos with dynamic motion and audio. The workflow is designed for e-commerce marketers, product photographers, and content creators who want to quickly transform simple photos into eye-catching marketing media.

**Logical Blocks:**

- **1.1 Input Reception & Download:** Receives product photos via Telegram bot, downloads the image file, and captures caption metadata.
- **1.2 AI Image Analysis:** Converts the image to Base64 and uses Nano Banana (Google Gemini) to analyze the product photo and generate a detailed, commercial-grade image generation prompt.
- **1.3 Image Enhancement:** Merges original image data and AI analysis, prepares payload, and submits to Nano Banana to create an enhanced, studio-quality product image optimized for 9:16 aspect ratio.
- **1.4 Video Generation:** Sends the enhanced image to Veo 3.1 video generation AI to create an 8-second vertical marketing video with audio and watermark.
- **1.5 Processing Loop:** Polls the video generation status periodically, waits if not ready, and validates completion.
- **1.6 Delivery:** Converts the Base64 video output to file format and sends the final video back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Download

- **Overview:** Captures incoming Telegram messages with product photos, downloads the photo file, and retrieves caption text for context.
- **Nodes Involved:**  
  - Telegram Trigger Received  
  - Download Image File  
  - Image to Base64  
  - AI Design Analysis (starts here due to use of caption)
- **Node Details:**

1. **Telegram Trigger Received**  
   - Type: Telegram Trigger  
   - Role: Entry point, listens for new messages with photos on Telegram.  
   - Config: Watches for "message" updates; captures message object containing photo and caption.  
   - Inputs: External Telegram message  
   - Outputs: Telegram message JSON with photo array and caption text.  
   - Failures: Missing photo or caption; Telegram API auth errors.  

2. **Download Image File**  
   - Type: Telegram node (file download)  
   - Role: Downloads the actual image file from Telegram servers using the first photo's file_id.  
   - Config: Uses the file_id from incoming message photo array.  
   - Inputs: Telegram Trigger output  
   - Outputs: Binary image data file  
   - Failures: File not found, Telegram API limits, auth errors.  

3. **Image to Base64**  
   - Type: Extract From File  
   - Role: Converts binary image file into Base64 string for AI processing.  
   - Config: Operation binaryToProperty, converts binary data to JSON property.  
   - Inputs: Download Image File binary output  
   - Outputs: JSON with Base64 image data  
   - Failures: File conversion errors, corrupted binary data.  

4. **AI Design Analysis**  
   - Type: Google Gemini (Nano Banana) Langchain Node  
   - Role: Analyzes the product image and caption text to create a detailed, structured prompt for image enhancement.  
   - Config: Uses Gemini 2.5-flash-lite model with an extensive prompt that demands JSON output describing product features, lighting, composition, style references, and a 500+ word creative prompt.  
   - Inputs: Base64 image (binary) and caption string expression  
   - Outputs: JSON containing structured product analysis and image generation prompt  
   - Failures: API quota limits, malformed prompt, JSON parse errors in AI response.

---

#### 2.2 Image Enhancement

- **Overview:** Combines the Base64 image and the AI analysis output, prepares the API payload, and calls the Google Gemini image generation endpoint to create an enhanced product image.
- **Nodes Involved:**  
  - Combine Image & Analysis  
  - Prepare API Payload  
  - Generate Enhanced Image  
  - Convert Base64 to Image (only image)  
  - Convert Base64 to Image (image and text)
- **Node Details:**

1. **Combine Image & Analysis**  
   - Type: Merge (combine data streams)  
   - Role: Joins the outputs of Image to Base64 and AI Design Analysis into one composite JSON for downstream use.  
   - Inputs: Image to Base64 output (index 0), AI Design Analysis output (index 1)  
   - Outputs: Merged data array combining both inputs  
   - Failures: Synchronization issues (if inputs are not aligned), data mismatches.  

2. **Prepare API Payload**  
   - Type: Code (JavaScript)  
   - Role: Constructs a single JSON object combining the merged inputs for the image generation API.  
   - Config: Collects items from the merged node, packages input1 and input2 arrays into one item.  
   - Inputs: Combine Image & Analysis output  
   - Outputs: JSON payload with fields input1 and input2 containing Base64 data and prompt text  
   - Failures: Coding errors, undefined inputs.

3. **Generate Enhanced Image**  
   - Type: HTTP Request  
   - Role: Calls Google Gemini image generation API to produce a commercial-grade enhanced image using the detailed prompt and original image data.  
   - Config: POST to Google Generative Language API endpoint with JSON body including the prompt text and embedded Base64 image; specifies response modalities TEXT and IMAGE, aspect ratio 9:16.  
   - Inputs: Prepare API Payload output  
   - Outputs: JSON with candidates array containing image data and textual response  
   - Failures: API authentication, rate limits, malformed request, network timeout.

4. **Convert Base64 to Image (only image)**  
   - Type: Convert To File  
   - Role: Converts the Base64 encoded enhanced image (first image part) into a binary image file.  
   - Config: Source property points to "candidates[0].content.parts[0].inlineData.data"  
   - Inputs: Generate Enhanced Image output  
   - Outputs: Binary image file  
   - Failures: Conversion errors, missing data.

5. **Convert Base64 to Image (image and text)**  
   - Type: Convert To File  
   - Role: Converts the Base64 encoded image from the second part (usually image with text overlay or metadata) of the candidates array to binary.  
   - Config: Source property "candidates[0].content.parts[1].inlineData.data"  
   - Inputs: Generate Enhanced Image output  
   - Outputs: Binary image file  
   - Failures: Same as above.

---

#### 2.3 Video Generation

- **Overview:** Uses the enhanced image to request Veo 3.1 AI to generate a short marketing video, then polls for completion.
- **Nodes Involved:**  
  - Initiate veo 3.1 Video Generation  
  - Check Video Status  
  - Video Ready Validator  
  - Processing Delay (30s)
- **Node Details:**

1. **Initiate veo 3.1 Video Generation**  
   - Type: HTTP Request  
   - Role: Starts video generation by sending enhanced image (Base64) and prompt to Veo 3.1 AI endpoint.  
   - Config: POST to Google AI Platform endpoint with JSON body specifying prompt, image bytes, aspect ratio 9:16, duration 8 seconds, audio enabled, watermark added, resolution 1080p.  
   - Inputs: Convert Base64 to Image nodes output (Base64 string via expression)  
   - Outputs: JSON containing operation name for asynchronous video generation  
   - Failures: OAuth2 auth errors, quota limits, malformed request.

2. **Check Video Status**  
   - Type: HTTP Request  
   - Role: Polls Veo 3.1 API with operation name to get current video generation status/result.  
   - Config: POST to fetchPredictOperation endpoint with JSON body containing operationName from previous node’s output.  
   - Inputs: Initiate veo 3.1 Video Generation output or Processing Delay output  
   - Outputs: JSON with video generation status, including Base64 video if ready  
   - Failures: API rate limit, network issues, missing operationName.

3. **Video Ready Validator**  
   - Type: If (conditional)  
   - Role: Checks if video generation response includes Base64 video data (indicating completion).  
   - Config: Condition checks if "response.videos[0].bytesBase64Encoded" exists and is non-empty.  
   - Inputs: Check Video Status output  
   - Outputs: True branch for ready video; False branch to wait and retry.  
   - Failures: Condition logic errors, missing fields.

4. **Processing Delay (30s)**  
   - Type: Wait  
   - Role: Introduces a 30-second delay before re-polling video status if video not ready.  
   - Config: Wait time set to 30 seconds.  
   - Inputs: False branch from Video Ready Validator  
   - Outputs: Triggers Check Video Status again  
   - Failures: Workflow timeout if video takes too long.

---

#### 2.4 Delivery

- **Overview:** Converts the Base64 video output to a binary video file and sends it back to the user on Telegram.
- **Nodes Involved:**  
  - Convert Base64 to video  
  - Send veo 3.1 video
- **Node Details:**

1. **Convert Base64 to video**  
   - Type: Convert To File  
   - Role: Converts the Base64 encoded video from Veo 3.1 response into a binary file for Telegram.  
   - Config: Source property "response.videos[0].bytesBase64Encoded"  
   - Inputs: True branch from Video Ready Validator  
   - Outputs: Binary video file  
   - Failures: Conversion errors, corrupted Base64.

2. **Send veo 3.1 video**  
   - Type: Telegram node (sendVideo)  
   - Role: Sends the generated video file back to the Telegram user who initiated the request.  
   - Config: Chat ID taken dynamically from original Telegram message sender ID; sends binary video data.  
   - Inputs: Convert Base64 to video output  
   - Outputs: Telegram message confirming video sent  
   - Failures: Telegram API errors, invalid chat ID, file size limits.

---

### 3. Summary Table

| Node Name                        | Node Type                      | Functional Role                                    | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                                        |
|---------------------------------|-------------------------------|---------------------------------------------------|-------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger Received        | Telegram Trigger              | Entry point, receives Telegram photo message      | External Telegram              | Download Image File                |                                                                                                                   |
| Download Image File              | Telegram                      | Downloads product photo from Telegram              | Telegram Trigger Received      | Image to Base64, AI Design Analysis |                                                                                                                   |
| Image to Base64                 | Extract From File             | Converts image binary to Base64                     | Download Image File            | Combine Image & Analysis           |                                                                                                                   |
| AI Design Analysis              | Google Gemini (Nano Banana)   | Analyzes product image and caption, creates prompt | Download Image File (caption)  | Combine Image & Analysis           |                                                                                                                   |
| Combine Image & Analysis        | Merge                        | Merges image data with AI analysis output          | Image to Base64, AI Design Analysis | Prepare API Payload              |                                                                                                                   |
| Prepare API Payload             | Code (JavaScript)             | Prepares combined payload for image enhancement    | Combine Image & Analysis       | Generate Enhanced Image            |                                                                                                                   |
| Generate Enhanced Image          | HTTP Request                 | Calls AI API to generate enhanced product image   | Prepare API Payload            | Convert Base64 to Image (only image) |                                                                                                                   |
| Convert Base64 to Image (only image) | Convert To File           | Converts enhanced image Base64 to binary           | Generate Enhanced Image        | Initiate veo 3.1 Video Generation, Convert Base64 to Image (image and text) |                                                                                                                   |
| Convert Base64 to Image (image and text) | Convert To File         | Converts second enhanced image Base64 to binary    | Generate Enhanced Image        | Initiate veo 3.1 Video Generation  |                                                                                                                   |
| Initiate veo 3.1 Video Generation | HTTP Request               | Starts video generation using enhanced image       | Convert Base64 to Image nodes  | Check Video Status                |                                                                                                                   |
| Check Video Status              | HTTP Request                 | Polls video generation status                       | Initiate veo 3.1 Video Generation, Processing Delay (30s) | Video Ready Validator           |                                                                                                                   |
| Video Ready Validator           | If                          | Checks if video is ready                            | Check Video Status             | Convert Base64 to video (true), Processing Delay (false) |                                                                                                                   |
| Processing Delay (30s)          | Wait                        | Waits before retrying video status check           | Video Ready Validator (false)  | Check Video Status                |                                                                                                                   |
| Convert Base64 to video         | Convert To File             | Converts Base64 video to binary file                | Video Ready Validator (true)   | Send veo 3.1 video               |                                                                                                                   |
| Send veo 3.1 video              | Telegram                    | Sends final video to Telegram user                  | Convert Base64 to video        |                                   |                                                                                                                   |
| Sticky Note                    | Sticky Note                 | Documentation and workflow overview note            | None                          | None                             | # Product Video Creator with Nano Banana & Veo 3.1 via Telegram ... (Full content as detailed in section 5 below) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook to receive "message" updates.  
   - Authenticate with Telegram API credentials.  
   - Position: Start of workflow.

2. **Create Download Image File Node**  
   - Type: Telegram  
   - Operation: Download file using file_id from photo[0] of Telegram Trigger message.  
   - Credentials: Telegram API account.  
   - Connect from Telegram Trigger output.

3. **Create Image to Base64 Node**  
   - Type: Extract From File  
   - Operation: binaryToProperty (convert binary file to Base64 string in JSON property).  
   - Connect from Download Image File output.

4. **Create AI Design Analysis Node**  
   - Type: Google Gemini (Langchain)  
   - Model: "models/gemini-2.5-flash-lite"  
   - Input: Pass Base64 image as binary input, use caption text from Telegram Trigger message via expression.  
   - Prompt: Use detailed prompt provided (see 2.1) requesting JSON structured product analysis and 500+ word image generation prompt.  
   - Credentials: Google Gemini API credentials.  
   - Connect from Download Image File output (for binary) and Telegram Trigger (for caption).

5. **Create Combine Image & Analysis Node**  
   - Type: Merge  
   - Mode: Combine inputs as separate streams (input 0: Image to Base64, input 1: AI Design Analysis).  
   - Connect Image to Base64 and AI Design Analysis nodes as inputs.

6. **Create Prepare API Payload Node**  
   - Type: Code (JavaScript)  
   - Code: Combine the merged inputs into one JSON object with keys input1 and input2 carrying respective data arrays.  
   - Connect from Combine Image & Analysis output.

7. **Create Generate Enhanced Image Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Google Gemini image generation endpoint  
   - Body: JSON containing prompt text and Base64 image embedded as per API spec, requesting TEXT and IMAGE response modalities, aspect ratio 9:16.  
   - Credentials: Google Gemini API key.  
   - Connect from Prepare API Payload output.

8. **Create Convert Base64 to Image (only image) Node**  
   - Type: Convert To File  
   - Operation: toBinary  
   - Source Property: "candidates[0].content.parts[0].inlineData.data" from Generate Enhanced Image output  
   - Connect from Generate Enhanced Image output.

9. **Create Convert Base64 to Image (image and text) Node**  
   - Type: Convert To File  
   - Operation: toBinary  
   - Source Property: "candidates[0].content.parts[1].inlineData.data"  
   - Connect from Generate Enhanced Image output.

10. **Create Initiate veo 3.1 Video Generation Node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Veo 3.1 generation API endpoint  
    - Body: JSON with prompt for 8-second marketing video, includes Base64 image from Convert Base64 to Image nodes, aspect ratio 9:16, resolution 1080p, audio enabled, watermark true.  
    - Credentials: Google OAuth2 API (Google account) and Google Gemini API.  
    - Connect from Convert Base64 to Image nodes (both).

11. **Create Check Video Status Node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Veo 3.1 fetchPredictOperation endpoint  
    - Body: JSON with operationName from Initiate veo 3.1 output.  
    - Credentials: Same as Initiate veo 3.1.  
    - Connect from Initiate veo 3.1 output and Processing Delay node.

12. **Create Video Ready Validator Node**  
    - Type: If (conditional)  
    - Condition: Check if "response.videos[0].bytesBase64Encoded" exists and is non-empty.  
    - Connect from Check Video Status output.  
    - True branch → Convert Base64 to video node  
    - False branch → Processing Delay node

13. **Create Processing Delay (30s) Node**  
    - Type: Wait  
    - Duration: 30 seconds  
    - Connect from Video Ready Validator false branch.  
    - Output connects back to Check Video Status node (loop).

14. **Create Convert Base64 to video Node**  
    - Type: Convert To File  
    - Operation: toBinary  
    - Source Property: "response.videos[0].bytesBase64Encoded"  
    - Connect from Video Ready Validator true branch.

15. **Create Send veo 3.1 video Node**  
    - Type: Telegram  
    - Operation: sendVideo  
    - Chat ID: Extract from original Telegram Trigger message sender ID dynamically  
    - Binary Data: Enable to send converted video file  
    - Credentials: Telegram API  
    - Connect from Convert Base64 to video output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| # Product Video Creator with Nano Banana & Veo 3.1 via Telegram  \n\n## 🎯 What This Does\nTransforms ordinary product photos into professional marketing videos with AI enhancement and video generation.\n\n---\n\n## 🔄 Workflow Stages\n\n### **Stage 1: Input & Download** 📥\n- User sends product photo via Telegram bot\n- Bot downloads the image file\n- Caption is captured for context\n\n### **Stage 2: AI Analysis** 🧠\n- **Image → Base64**: Converts image to data format\n- **AI Design Analysis**: Nano Banana analyzes the product and creates detailed enhancement prompt\n  - Identifies product type, materials, colors\n  - Suggests professional lighting setup\n  - Recommends composition improvements\n  - Generates 500+ word creative prompt\n\n### **Stage 3: Image Enhancement** ✨\n- **Merge Data**: Combines original image + AI analysis\n- **Generate Enhanced Image**: Nano Banana creates professional product photo\n  - Aspect ratio: 9:16 (vertical/mobile-optimized)\n  - Studio-quality lighting\n  - Commercial-grade composition\n\n### **Stage 4: Video Generation** 🎬\n- **Initiate Veo 3.1**: Sends enhanced image to Veo 3.1 video AI\n  - Duration: 8 seconds\n  - Resolution: 1080p\n  - With audio\n  - 9:16 vertical format\n\n### **Stage 5: Processing Loop** ⏳\n- **Check Status**: Polls video generation progress\n- **Validator**: Checks if video is ready\n  - ✅ **Ready** → Convert & send\n  - ❌ **Not Ready** → Wait 30s → Check again\n\n### **Stage 6: Delivery** 📤\n- Converts Base64 video to binary file\n- Sends final video back to user on Telegram\n\n---\n\n## ⏱️ Expected Timeline\n- Image enhancement: ~5-10 seconds\n- Video generation: ~30-90 seconds\n- Total: 1-2 minutes per video\n\n---\n\n## 💡 Best Practices\n1. **Caption matters**: Include product details in photo caption\n2. **Clear photos**: Better input = better output\n3. **Patience**: Video generation takes time, bot will auto-retry\n4. **Vertical photos**: Work best for 9:16 format\n\n---\n\n## 🎨 Fully Customizable\n**Change based on your requirements:**\n- **Prompt**: Modify AI analysis instructions\n- **Aspect Ratio**: 16:9, 9:16 etc.\n- **Duration**: 4s to 8s videos\n- **Quality**: 720p, 1080p resolution\n- **Audio**: Enable/disable background music\n- **Watermark**: Add/remove branding | Sticky note in workflow covering overview and best practices.                                      |

---

**Disclaimer:** The provided text strictly originates from an n8n automated workflow. All content complies with current content policies and contains no illegal, offensive, or protected material. Only legal and public data are processed.