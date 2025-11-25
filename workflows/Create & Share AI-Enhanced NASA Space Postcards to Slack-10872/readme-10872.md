Create & Share AI-Enhanced NASA Space Postcards to Slack

https://n8nworkflows.xyz/workflows/create---share-ai-enhanced-nasa-space-postcards-to-slack-10872


# Create & Share AI-Enhanced NASA Space Postcards to Slack

### 1. Workflow Overview

This workflow, titled **"NASA APOD Space Postcard Generator"**, automates the creation and sharing of AI-enhanced space-themed postcards derived from NASA’s Astronomy Picture of the Day (APOD). It is designed for users who want to generate unique, poetic space images and share them instantly on Slack.

**Target Use Cases:**

- Automatically generate visually enriched space postcards with AI-generated poetic captions.
- Share engaging space content to Slack channels for teams or communities interested in astronomy.
- Demonstrate integration of NASA API, AI text generation, image processing, and Slack API within a seamless automation.

**Logical Blocks:**

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 NASA APOD Retrieval:** Fetch a random APOD image from NASA’s API and verify media type.
- **1.3 AI Message Generation:** Use AI to craft a poetic, concise message and determine text placement coordinates on the image.
- **1.4 Image Processing:** Overlay the AI-generated text onto the NASA image.
- **1.5 Slack Sharing:** Upload the final edited image to Slack and send a confirmation message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow via a manual trigger node, allowing users to start the process on-demand.

- **Nodes Involved:**  
  - When clicking 'Test workflow'

- **Node Details:**  
  - **When clicking 'Test workflow'**  
    - Type: Manual Trigger  
    - Role: Starts workflow execution manually.  
    - Configuration: No parameters required.  
    - Connections: Outputs to "Get NASA APOD" node.  
    - Edge cases: None; manual start.  
    - Version: 1  

---

#### 2.2 NASA APOD Retrieval

- **Overview:**  
  Retrieves a random NASA Astronomy Picture of the Day from the past 10 years and ensures the media type is an image (not video).

- **Nodes Involved:**  
  - Get NASA APOD  
  - Check if Media Type is Image  
  - Edit Image1  
  - Edit Fields

- **Node Details:**  

  - **Get NASA APOD**  
    - Type: NASA API node  
    - Role: Fetch NASA APOD for a random date within last 10 years (3650 days).  
    - Configuration: Uses an expression to generate a random date by subtracting a random number of days from current date.  
    - Key Expression:  
      `"={{ $now.minus({ days: Math.floor(Math.random() * 3650) }).toFormat('yyyy-MM-dd') }}"`  
    - Connections: Outputs to "Check if Media Type is Image".  
    - Edge cases: API key or quota errors, date without image media (video instead).  
    - Version: 1  

  - **Check if Media Type is Image**  
    - Type: If node  
    - Role: Conditional filter to proceed only if media type is "image".  
    - Configuration: Checks if `media_type` from NASA APOD response equals "image" (case-sensitive, strict).  
    - Connections: True branch to "Edit Image1".  
    - Edge cases: Missing or malformed media_type field, non-image media types.  
    - Version: 2.2  

  - **Edit Image1**  
    - Type: Edit Image  
    - Role: Retrieves information about the image (dimensions) to assist AI agent in text placement.  
    - Configuration: Operation set to "information", uses binary image data from NASA APOD.  
    - Input: Receives binary data from NASA APOD after media type check.  
    - Output: Passes image metadata to "AI Agent".  
    - Edge cases: Corrupted image data, unsupported image formats.  
    - Version: 1  

  - **Edit Fields**  
    - Type: Set node  
    - Role: Prepares the binary image data for further editing by assigning it to a new property named `image_file`.  
    - Configuration: Assigns NASA APOD binary data (`data`) to binary property `image_file`.  
    - Input: Receives from "AI Agent".  
    - Output: Passes to "Edit Image".  
    - Edge cases: Missing binary data, assignment failures.  
    - Version: 3.4  

---

#### 2.3 AI Message Generation

- **Overview:**  
  Uses an AI language model to craft a short, poetic message inspired by the NASA APOD image and calculates coordinates for placing this text on the image.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**  

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Expert image processor tasked with generating a poetic message and text coordinates for overlay.  
    - Configuration:  
      - Input prompt includes image dimensions, photo title, and explanation from "Edit Image1" and "Get NASA APOD".  
      - Instructions specify generating a short poem (<50 chars, no line breaks) and calculating fixed margin coordinates (x=40, y=Image Height - 60).  
      - Output must be strict JSON with keys: `text`, `x`, `y`.  
    - Key Expressions:  
      - Uses expressions to pull width and height dynamically and to include NASA photo metadata.  
    - Input: Receives image metadata from "Edit Image1".  
    - Output: Passes JSON result to "Edit Fields".  
    - Edge cases: AI service downtime, malformed AI output, parsing errors on JSON result, or API quota limits.  
    - Version: 3  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the backend AI model (GPT-4.1-mini) for the AI Agent.  
    - Configuration: Model preset to "gpt-4.1-mini", no additional options.  
    - Input: Receives prompt from "AI Agent".  
    - Output: AI response back to "AI Agent".  
    - Credential: Requires valid OpenAI API credentials.  
    - Edge cases: Authentication errors, rate limits, unexpected API changes.  
    - Version: 1.2  

---

#### 2.4 Image Processing

- **Overview:**  
  Overlays the AI-generated poetic message onto the NASA APOD image at calculated coordinates, creating a personalized space postcard.

- **Nodes Involved:**  
  - Edit Image  
  - Upload a file

- **Node Details:**  

  - **Edit Image**  
    - Type: Edit Image  
    - Role: Adds the poetic text onto the image with specified font size, color, and position.  
    - Configuration:  
      - Text extracted from parsed AI Agent JSON output.  
      - Font size set to 50, white color (#FFFFFF).  
      - Position X and Y set dynamically from AI Agent coordinates.  
      - Uses binary property `image_file` as the image source.  
    - Input: Receives image binary and AI text coordinates from "Edit Fields".  
    - Output: Passes modified image to "Upload a file" and "Send a message".  
    - Edge cases: Parsing errors from AI output, image format compatibility, large image sizes causing processing delays.  
    - Version: 1  

  - **Upload a file**  
    - Type: Slack node (file upload)  
    - Role: Uploads the edited image file to a specified Slack channel.  
    - Configuration:  
      - Authentication via OAuth2 credentials for Slack.  
      - Binary property name set to `image_file`.  
      - Channel ID must be manually selected.  
    - Input: Receives edited image binary from "Edit Image".  
    - Output: None (ends this branch).  
    - Edge cases: Slack authentication failures, wrong channel ID, file size limits by Slack, network errors.  
    - Version: 2.3  

---

#### 2.5 Slack Sharing

- **Overview:**  
  Sends a simple confirmation message to the same Slack channel after posting the postcard.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**  

  - **Send a message**  
    - Type: Slack node (message sending)  
    - Role: Posts a text message ("Have a good day!") to the Slack channel to confirm sharing.  
    - Configuration:  
      - Authentication via OAuth2 credentials.  
      - Channel ID must match the upload channel (manual selection required).  
      - Static message text.  
    - Input: Receives control from "Edit Image" node output branch.  
    - Output: None (workflow end).  
    - Edge cases: Slack API errors, authentication failures, channel ID mismatch.  
    - Version: 2.3  

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                         | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                                             |
|---------------------------|-------------------------------------|---------------------------------------|----------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow' | Manual Trigger                      | Workflow start trigger                 | —                          | Get NASA APOD                   |                                                                                                                        |
| Get NASA APOD             | NASA API node                       | Fetch random NASA APOD image           | When clicking 'Test workflow' | Check if Media Type is Image    |                                                                                                                        |
| Check if Media Type is Image | If node                            | Verify media is image type             | Get NASA APOD              | Edit Image1                    |                                                                                                                        |
| Edit Image1               | Edit Image (information operation) | Get image metadata                     | Check if Media Type is Image | AI Agent                      |                                                                                                                        |
| AI Agent                  | LangChain AI Agent                  | Generate poetic message and coordinates| Edit Image1                | Edit Fields                   |                                                                                                                        |
| OpenAI Chat Model         | LangChain OpenAI Chat Model         | AI language model backend              | AI Agent                   | AI Agent                      |                                                                                                                        |
| Edit Fields               | Set node                           | Prepare binary image property          | AI Agent                   | Edit Image                    |                                                                                                                        |
| Edit Image                | Edit Image (text overlay)           | Overlay AI text on NASA image          | Edit Fields                | Upload a file, Send a message  |                                                                                                                        |
| Upload a file             | Slack (file upload)                 | Upload postcard image to Slack channel | Edit Image                 | —                             | **IMPORTANT:** You must select your channel in both Slack nodes!                                                       |
| Send a message            | Slack (message send)                | Send confirmation message to Slack     | Edit Image                 | —                             | **IMPORTANT:** You must select your channel in both Slack nodes!                                                       |
| Main Overview             | Sticky Note                        | Overview and setup instructions       | —                          | —                             | ## How it works\nThis workflow creates a unique "space postcard" and sends it to Slack...                              |
| Group 1                   | Sticky Note                        | Block 1 summary                       | —                          | —                             | ## 1. Get a space picture\nGets a random picture from NASA's Picture of the Day and checks if it's an image (not video).|
| Group 2                   | Sticky Note                        | Block 2 summary                       | —                          | —                             | ## 2. Write a message with AI\nAsks an AI to write a short, poetic message inspired by the space picture.               |
| Group 3                   | Sticky Note                        | Block 3 summary                       | —                          | —                             | ## 3. Create the postcard\nAdds the poetic message directly onto the NASA image, creating your unique postcard.         |
| Group 4                   | Sticky Note                        | Block 4 summary                       | —                          | —                             | ## 4. Share on Slack\nPosts the final postcard image to your Slack channel. **IMPORTANT:** You must select your channel in both Slack nodes! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.  
   - No special parameters.  

2. **Add NASA APOD Node**  
   - Type: NASA API node  
   - Parameters: Set `date` to expression:  
     ```  
     {{$now.minus({ days: Math.floor(Math.random() * 3650) }).toFormat('yyyy-MM-dd')}}  
     ```  
   - Add NASA API credentials to the node.  
   - Connect Manual Trigger → NASA APOD.  

3. **Add If Node: Check if Media Type is Image**  
   - Type: If node  
   - Condition:  
     - Left value: `={{ $('Get NASA APOD').item.json.media_type }}`  
     - Operator: Equals  
     - Right value: `"image"`  
   - Connect NASA APOD → If node.  

4. **Add Edit Image Node (Information)**  
   - Type: Edit Image  
   - Operation: "information"  
   - Data Property Name: `={{ $('Get NASA APOD').item.binary.data }}`  
   - Connect If node (True output) → Edit Image1.  

5. **Add AI Agent Node**  
   - Type: LangChain AI Agent  
   - Text parameter:  
     ```
     ## Your Role
     You are an expert image processor. Your task is to calculate the coordinates for placing a text overlay on an image and provide a poetic message.

     ## Image and Text Information
     *   Image Width: {{ $('Edit Image1').item.json.size.width }}
     *   Image Height: {{ $('Edit Image1').item.json.size.height }}
     *   Photo Title: {{ $('Get NASA APOD').item.json.title }}
     *   Photo Explanation: {{ $('Get NASA APOD').item.json.explanation }}

     ## Instructions
     1.  Generate a short, poetic message (under 50 characters, no line breaks) inspired by the photo information.
     2.  Calculate the coordinates (x, y) for placing this text in the **bottom-left corner** of the image, with a small margin.
         *   `x` should be a fixed margin from the left, like `40`.
         *   `y` should be calculated based on the image height. A good position would be `Image Height - 60`.
     3.  You MUST output the result in the following JSON format. Do not include any other text or explanations.

     {
       "text": "Your poetic message here",
       "x": 40,
       "y": {{ $('Edit Image1').item.json.size.height - 60 }}
     }
     ```  
   - Connect Edit Image1 → AI Agent.  

6. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: Select `gpt-4.1-mini`  
   - Connect AI Agent → OpenAI Chat Model (ai_languageModel input).  
   - Connect OpenAI Chat Model → AI Agent (ai_languageModel output).  
   - Provide OpenAI API credentials.  

7. **Add Set Node (Edit Fields)**  
   - Type: Set node  
   - Assign `image_file` (binary) = `={{ $('Get NASA APOD').item.binary.data }}`  
   - Include other fields (default).  
   - Connect AI Agent → Edit Fields.  

8. **Add Edit Image Node (Text Overlay)**  
   - Type: Edit Image  
   - Operation: "text"  
   - Text:  
     ```
     ={{ JSON.parse($json.output.slice($json.output.indexOf('{'), $json.output.lastIndexOf('}') + 1)).text }}
     ```  
   - Font Size: 50  
   - Font Color: #FFFFFF  
   - Position X:  
     ```
     ={{ JSON.parse($json.output.slice($json.output.indexOf('{'), $json.output.lastIndexOf('}') + 1)).x }}
     ```  
   - Position Y:  
     ```
     ={{ JSON.parse($json.output.slice($json.output.indexOf('{'), $json.output.lastIndexOf('}') + 1)).y }}
     ```  
   - Data Property Name: `image_file`  
   - Connect Edit Fields → Edit Image.  

9. **Add Slack Upload a File Node**  
   - Type: Slack  
   - Resource: file  
   - Authentication: OAuth2 (set Slack credentials)  
   - Binary Property Name: `image_file`  
   - Select Slack Channel ID manually.  
   - Connect Edit Image → Upload a file.  

10. **Add Slack Send a Message Node**  
    - Type: Slack  
    - Text: "Have a good day!"  
    - Select same Slack Channel ID as upload node.  
    - Authentication: OAuth2 (Slack credentials)  
    - Connect Edit Image → Send a message.  

11. **Review Connections:**  
    - Manual Trigger → Get NASA APOD  
    - Get NASA APOD → Check if Media Type is Image  
    - If True → Edit Image1 → AI Agent → Edit Fields → Edit Image  
    - Edit Image → Upload a file  
    - Edit Image → Send a message  
    - AI Agent uses OpenAI Chat Model as language model backend.  

12. **Test the workflow** by running the manual trigger. Ensure all credentials are correctly configured and Slack channel is selected in both Slack nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                    |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow creates a unique "space postcard" and sends it to Slack, combining NASA APOD images with AI poetry.| Sticky Note “Main Overview” node inside the workflow.                                                              |
| Setup requires NASA API, OpenAI API, and Slack credentials with OAuth2 authentication and channel selection.     | Instructions in “Main Overview” sticky note.                                                                       |
| Slack channel selection is mandatory in both Slack nodes to ensure proper file upload and message posting.       | Sticky Notes “Group 4” and repeated sticky note comments in table.                                                |
| NASA APOD API returns videos occasionally; the workflow filters only images to process.                          | Sticky Note “Group 1”, and logic in “Check if Media Type is Image” node.                                           |
| AI prompt design enforces JSON-only response format to facilitate parsing and reduce errors.                     | AI Agent node configuration.                                                                                       |
| Font size and color for text overlay are fixed for visual consistency but can be customized in the Edit Image node.| Node “Edit Image” parameters.                                                                                       |
| This workflow demonstrates best practices for combining API data, AI-generated content, image editing, and Slack integration.| Overall workflow design.                                                                                           |

---

**Disclaimer:**  
The text and content described here originate exclusively from an automated workflow built with n8n, adhering strictly to content policies, containing no illegal or offensive material. All processed data are legally sourced and publicly available.