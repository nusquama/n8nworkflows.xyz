Generate MemeCoin Art with Gemini Flash & NanoBanana and Post to Twitter

https://n8nworkflows.xyz/workflows/generate-memecoin-art-with-gemini-flash---nanobanana-and-post-to-twitter-9470


# Generate MemeCoin Art with Gemini Flash & NanoBanana and Post to Twitter

### 1. Workflow Overview

This workflow automates the creation and posting of MemeCoin art on Twitter by leveraging Google Gemini's AI capabilities and the NanoBanana image generation model. It is designed to periodically generate a themed meme cryptocurrency mascot image and a catchy tweet about the coin’s potential, then upload both to Twitter.

Logical blocks:

- **1.1 Input Initialization:** Defines the MemeCoin parameters such as name, mascot description, and source image URL.
- **1.2 Scheduled Trigger:** Periodically triggers the workflow to run automatically.
- **1.3 AI Text Generation:** Uses LangChain with Google Gemini chat to generate a tweet in Gen Z slang and an AI prompt for the mascot image based on trending topics.
- **1.4 Image Processing:** Downloads the source mascot image, converts it to base64, then sends it along with the AI-generated image prompt to NanoBanana (Google Gemini Flash Image API) for enhanced meme art generation.
- **1.5 Image Conversion and Upload:** Converts the generated base64 image to a PNG binary, uploads it to Twitter’s media endpoint, and posts the tweet with the media attached.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** Sets the foundational parameters for the memecoin art generation, including the coin’s name, mascot description, and source image URL.
- **Nodes Involved:** Define Memecoin
- **Node Details:**

  - **Define Memecoin**
    - Type: Set node
    - Role: Assigns static values for `memecoin_name`, `mascot_description`, and `mascot_image` URL.
    - Configuration: Hardcoded values:
      - memecoin_name: "popcat"
      - mascot_description: "cat with open mouth"
      - mascot_image: "https://i.pinimg.com/736x/9d/05/6b/9d056b5b97c0513a4fc9d9cd93304a05.jpg"
    - Input Connections: From Schedule Trigger
    - Output Connections: To AI Agent
    - Edge Cases: Changes in source URL or incorrect URL format can cause HTTP request failures downstream.

#### 2.2 Scheduled Trigger

- **Overview:** Periodically initiates the workflow every hour to generate new content automatically.
- **Nodes Involved:** Schedule Trigger
- **Node Details:**

  - **Schedule Trigger**
    - Type: scheduleTrigger
    - Role: Time-based trigger set to run every hour.
    - Configuration: Interval set to hourly.
    - Input Connections: None (start node)
    - Output Connections: To Define Memecoin
    - Edge Cases: If workflow is paused or n8n instance is down, scheduled runs will be missed.

#### 2.3 AI Text Generation

- **Overview:** Uses Google Gemini chat model via LangChain to generate both a catchy tweet and an image prompt describing the mascot, based on current trending topics.
- **Nodes Involved:** AI Agent, Google Gemini Chat Model, Structured Output Parser
- **Node Details:**

  - **AI Agent**
    - Type: LangChain Agent node
    - Role: Generates output text including a 100-character Gen Z slang tweet about the memecoin and an image generation prompt.
    - Configuration:
      - Prompt includes variables from Define Memecoin node (`memecoin_name`, `mascot_description`).
      - Output uses structured output parser.
      - Prompt Type: define
    - Input Connections: From Define Memecoin
    - Output Connections: To Get source image
    - Edge Cases: AI model failure, API rate limits, or malformed prompt could cause generation errors.

  - **Google Gemini Chat Model**
    - Type: LangChain LM Chat Google Gemini
    - Role: Provides the underlying AI language model for the AI Agent node.
    - Configuration: Uses Google Palm API credentials.
    - Input Connections: To AI Agent (language model input)
    - Output Connections: None (used internally by AI Agent)
    - Edge Cases: API authentication failures, quota limits.

  - **Structured Output Parser**
    - Type: LangChain Structured Output Parser
    - Role: Parses the AI Agent output JSON with defined schema containing `tweet` and `imagePrompt`.
    - Configuration: Manual schema with string properties for tweet and imagePrompt.
    - Input Connections: To AI Agent (output parser)
    - Output Connections: None (used internally by AI Agent)
    - Edge Cases: Parsing errors if AI output deviates from schema.

#### 2.4 Image Processing

- **Overview:** Downloads the mascot source image, converts it to base64, then sends it along with the AI-generated image prompt to Google Gemini Flash Image API (NanoBanana) to generate enhanced meme art.
- **Nodes Involved:** Get source image, Convert Source image to base64, Generate image using NanoBanana, Convert Base64 to Png
- **Node Details:**

  - **Get source image**
    - Type: HTTP Request
    - Role: Downloads mascot image from the URL defined in Define Memecoin.
    - Configuration: URL dynamically set from `mascot_image` parameter.
    - Input Connections: From AI Agent
    - Output Connections: To Convert Source image to base64
    - Edge Cases: HTTP errors, broken image links, network timeouts.

  - **Convert Source image to base64**
    - Type: Extract From File
    - Role: Converts downloaded binary image data to base64 string stored in JSON property `Image`.
    - Configuration: Operation set to binaryToProperty.
    - Input Connections: From Get source image
    - Output Connections: To Generate image using NanoBanana
    - Edge Cases: File format incompatibility.

  - **Generate image using NanoBanana**
    - Type: HTTP Request
    - Role: Sends image generation request to Gemini Flash Image API with the base64 source image and AI-generated prompt.
    - Configuration:
      - URL: Gemini API endpoint for image generation.
      - Method: POST with JSON body.
      - Body includes:
        - `contents.parts` with text prompt and inline base64 image data.
        - Aspect ratio set to 16:9.
      - Uses Google Palm API credentials.
    - Input Connections: From Convert Source image to base64
    - Output Connections: To Convert Base64 to Png
    - Edge Cases: API authentication failure, API changes, rate limits, invalid base64 image data.

  - **Convert Base64 to Png**
    - Type: Convert To File
    - Role: Converts base64 image data returned from NanoBanana API to binary PNG format.
    - Configuration:
      - Source property dynamically extracts base64 from response candidates, accommodating varying JSON paths.
      - Operation: toBinary.
    - Input Connections: From Generate image using NanoBanana
    - Output Connections: To Upload to Twitter
    - Edge Cases: Unexpected API response structure causing property path errors.

#### 2.5 Image Upload and Tweet Creation

- **Overview:** Uploads the generated image to Twitter media API, then posts the generated tweet text with the image attached.
- **Nodes Involved:** Upload to Twitter, Create Tweet
- **Node Details:**

  - **Upload to Twitter**
    - Type: HTTP Request
    - Role: Uploads binary PNG image to Twitter’s media upload endpoint with media category set as TWEET_IMAGE.
    - Configuration:
      - URL: Twitter media upload endpoint.
      - Method: POST with multipart-form-data.
      - Auth via Twitter OAuth1 credentials.
      - Retries enabled on failure with 1s wait.
    - Input Connections: From Convert Base64 to Png
    - Output Connections: To Create Tweet
    - Edge Cases: Twitter API rate limits, auth token expiry, upload failures.

  - **Create Tweet**
    - Type: Twitter node (OAuth2)
    - Role: Posts the tweet using generated text and attaches the uploaded media by media_id_string.
    - Configuration:
      - Text: Uses expression to get tweet text from AI Agent structured output.
      - Attachments: Uses media_id_string from Upload to Twitter node.
      - Auth via Twitter OAuth2 credentials.
    - Input Connections: From Upload to Twitter
    - Output Connections: None (end node)
    - Edge Cases: Twitter posting failures, invalid media_id reference.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                      | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                          |
|---------------------------|----------------------------------|------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | scheduleTrigger                   | Triggers workflow every hour        | None                   | Define Memecoin          |                                                                                                                      |
| Define Memecoin           | Set                              | Defines memecoin parameters          | Schedule Trigger       | AI Agent                | # Define memecoin Prompt memecoin_name : popcat mascot_description : cat with open mouth mascot_image : https://i.pinimg.com/736x/9d/05/6b/9d056b5b97c0513a4fc9d9cd93304a05.jpg API docs https://ai.google.dev/gemini-api/docs/image-generation#gemini-image-editing Get API key https://aistudio.google.com/api-keys |
| AI Agent                 | LangChain agent                  | Generates tweet and image prompt     | Define Memecoin        | Get source image        |                                                                                                                      |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini | Provides AI language model           | To AI Agent (ai_languageModel) | None                   |                                                                                                                      |
| Structured Output Parser | LangChain Output Parser           | Parses AI agent output JSON          | To AI Agent (ai_outputParser) | None                   |                                                                                                                      |
| Get source image          | HTTP Request                     | Downloads mascot image from URL      | AI Agent               | Convert Source image to base64 | # Get source memecoin image ![mascot_image](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-original.jpg) |
| Convert Source image to base64 | Extract From File           | Converts image binary to base64      | Get source image       | Generate image using NanoBanana |                                                                                                                      |
| Generate image using NanoBanana | HTTP Request                | Calls Gemini Flash API to generate art | Convert Source image to base64 | Convert Base64 to Png    | # Generate Image using Nano Banana ![mascot_image](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-edit.png) |
| Convert Base64 to Png     | Convert To File                  | Converts base64 string to PNG binary | Generate image using NanoBanana | Upload to Twitter       |                                                                                                                      |
| Upload to Twitter         | HTTP Request                     | Uploads image binary to Twitter      | Convert Base64 to Png  | Create Tweet            | # Upload to Twitter ![mascot_image](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-edit-twitter-1.png)    |
| Create Tweet              | Twitter                          | Posts tweet text with media attached | Upload to Twitter      | None                    |                                                                                                                      |
| Sticky Note5              | Sticky Note                     | Informational note about memecoin definition | None                   | None                    | # Define memecoin Prompt memecoin_name : popcat mascot_description : cat with open mouth mascot_image : https://i.pinimg.com/736x/9d/05/6b/9d056b5b97c0513a4fc9d9cd93304a05.jpg API docs https://ai.google.dev/gemini-api/docs/image-generation#gemini-image-editing Get API key https://aistudio.google.com/api-keys |
| Sticky Note4              | Sticky Note                     | Shows source memecoin image          | None                   | None                    | # Get source memecoin image ![mascot_image](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-original.jpg)  |
| Sticky Note6              | Sticky Note                     | Branding note Emp 0 🦑                | None                   | None                    | # Emp 0 🦑                                                                                                            |
| Sticky Note7              | Sticky Note                     | Shows NanoBanana generated image     | None                   | None                    | # Generate Image using Nano Banana ![mascot_image](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-edit.png) |
| Sticky Note8              | Sticky Note                     | Shows image upload to Twitter         | None                   | None                    | # Upload to Twitter ![mascot_image](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-edit-twitter-1.png)    |
| Sticky Note12             | Sticky Note                     | Title: MemeCoin Art Generator 🍌      | None                   | None                    | # *MemeCoin Art Generator 🍌*                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Name: `Schedule Trigger`
   - Type: scheduleTrigger
   - Parameters: Set interval to every 1 hour.

2. **Create a Set node for Memecoin Definition:**
   - Name: `Define Memecoin`
   - Type: Set
   - Parameters:
     - memecoin_name: `popcat`
     - mascot_description: `cat with open mouth`
     - mascot_image: `https://i.pinimg.com/736x/9d/05/6b/9d056b5b97c0513a4fc9d9cd93304a05.jpg`
   - Connect output of `Schedule Trigger` to input of `Define Memecoin`.

3. **Create LangChain AI Agent node:**
   - Name: `AI Agent`
   - Type: @n8n/n8n-nodes-langchain.agent
   - Parameters:
     - Text prompt:  
       ```
       You are a twitter generating agent. Your job is to 1. Generate a 100 character tweet in gen z slang indicating this memecoin {{ $('Define Memecoin').item.json.memecoin_name }} will go up. 2. Generate an image generation prompt for my mascot {{ $('Define Memecoin').item.json.mascot_description }} based on current trending topics today.
       ```
     - Prompt Type: `define`
     - Enable structured output parser
   - Connect `Define Memecoin` output to `AI Agent` input.

4. **Set up LangChain Google Gemini Chat Model node:**
   - Name: `Google Gemini Chat Model`
   - Type: @n8n/n8n-nodes-langchain.lmChatGoogleGemini
   - Credentials: Configure with Google Palm API key.
   - Connect as language model input to `AI Agent` node (ai_languageModel channel).

5. **Set up LangChain Structured Output Parser node:**
   - Name: `Structured Output Parser`
   - Type: @n8n/n8n-nodes-langchain.outputParserStructured
   - Parameters:
     - Schema (manual):
       ```json
       {
         "type": "object",
         "properties": {
           "tweet": { "type": "string" },
           "imagePrompt": { "type": "string" }
         }
       }
       ```
   - Connect as output parser input to `AI Agent` node (ai_outputParser channel).

6. **Create HTTP Request node to get source image:**
   - Name: `Get source image`
   - Type: HTTP Request
   - Parameters:
     - URL: `={{ $('Define Memecoin').item.json.mascot_image }}`
     - Method: GET
   - Connect `AI Agent` output to this node.

7. **Create Extract From File node to convert image to base64:**
   - Name: `Convert Source image to base64`
   - Type: Extract From File
   - Parameters:
     - Operation: `binaryToProperty`
     - Destination Key: `Image`
   - Connect `Get source image` output to this node.

8. **Create HTTP Request node for NanoBanana image generation:**
   - Name: `Generate image using NanoBanana`
   - Type: HTTP Request
   - Parameters:
     - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent`
     - Method: POST
     - Body (JSON):
       ```json
       {
         "contents": [{
           "parts": [
             { "text": "{{ $('AI Agent').item.json.output.imagePrompt }}" },
             {
               "inline_data": {
                 "mime_type": "image/jpeg",
                 "data": "{{ $json.Image }}"
               }
             }
           ]
         }],
         "generationConfig": {
           "imageConfig": {
             "aspectRatio": "16:9"
           }
         }
       }
       ```
     - Authentication: Google Palm API credentials
   - Connect `Convert Source image to base64` output to this node.

9. **Create Convert To File node to convert base64 response to PNG:**
   - Name: `Convert Base64 to Png`
   - Type: Convert To File
   - Parameters:
     - Operation: `toBinary`
     - Source Property:  
       ```
       ={{ $if($json.candidates[0].content.parts[0].inlineData, 'candidates[0].content.parts[0].inlineData.data', 'candidates[0].content.parts[1].inlineData.data') }}
       ```
   - Connect `Generate image using NanoBanana` output to this node.

10. **Create HTTP Request node to upload image to Twitter:**
    - Name: `Upload to Twitter`
    - Type: HTTP Request
    - Parameters:
      - URL: `https://upload.twitter.com/1.1/media/upload.json?media_category=TWEET_IMAGE`
      - Method: POST
      - Content-Type: multipart/form-data
      - Body Parameters:
        - Name: `media`
        - Parameter Type: `formBinaryData`
        - Input Data Field Name: `data`
      - Authentication: Twitter OAuth1 API credentials
      - Retry on Fail: Enabled, 1s wait between tries
    - Connect `Convert Base64 to Png` output to this node.

11. **Create Twitter node to post the tweet:**
    - Name: `Create Tweet`
    - Type: Twitter (OAuth2)
    - Parameters:
      - Text: `={{ $('AI Agent').item.json.output.tweet }}`
      - Additional Fields → Attachments: `={{ $json.media_id_string }}`
      - Credentials: Twitter OAuth2 API
    - Connect `Upload to Twitter` output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| API documentation for Gemini image editing API: https://ai.google.dev/gemini-api/docs/image-generation#gemini-image-editing   | Google Gemini API docs                                                                               |
| To obtain Google API key for Gemini: https://aistudio.google.com/api-keys                                                      | Google AI Studio API key registration                                                              |
| MemeCoin Art Generator branding note: *MemeCoin Art Generator 🍌*                                                               | Project branding                                                                                     |
| Emp 0 branding note: Emp 0 🦑                                                                                                   | Project branding                                                                                     |
| Source mascot image used: ![popcat original](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-original.jpg)           | Source image reference                                                                               |
| NanoBanana generated image example: ![popcat edit](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-edit.png)          | Generated image example                                                                              |
| Twitter uploaded image example: ![popcat edit twitter](https://articles.emp0.com/wp-content/uploads/2025/10/popcat-edit-twitter-1.png) | Uploaded image preview                                                                                |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.