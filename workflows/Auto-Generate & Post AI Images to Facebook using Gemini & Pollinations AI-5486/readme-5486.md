Auto-Generate & Post AI Images to Facebook using Gemini & Pollinations AI

https://n8nworkflows.xyz/workflows/auto-generate---post-ai-images-to-facebook-using-gemini---pollinations-ai-5486


# Auto-Generate & Post AI Images to Facebook using Gemini & Pollinations AI

### 1. Workflow Overview

This workflow automates the creation and posting of AI-generated images to a Facebook Page using Google Gemini and Pollinations AI. It is designed for users who want to publish unique artistic images with imaginative prompts multiple times per day without manual intervention.

The workflow runs three times daily at fixed hours (7 AM, 11 AM, and 5 PM), executing the following logical blocks:

- **1.1 Schedule Trigger and Logging:** Initiates the workflow at set times and logs the trigger event.
- **1.2 AI Prompt Generation (Google Gemini):** Uses Google Gemini (via LangChain) to generate a single, creative image prompt in a specified artistic style.
- **1.3 Image URL Construction (Pollinations AI):** Builds a URL to generate an AI image based on the prompt, with randomized seed and fixed dimensions.
- **1.4 Image Retrieval:** Downloads the generated image from Pollinations AI.
- **1.5 Facebook Posting:** Uploads the retrieved image to a configured Facebook Page with the prompt text as caption and relevant hashtags.

This modular structure ensures clear separation of concerns: scheduling, prompt generation, image creation, and social media interaction.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger and Logging

- **Overview:** Triggers the workflow at predefined times daily and logs the event for monitoring purposes.
- **Nodes Involved:**  
  - Set The Schedule  
  - Log Schedule Time

- **Node Details:**

  - **Set The Schedule**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow at 7 AM, 11 AM, and 5 PM every day.  
    - Configuration: Interval trigger with triggerAtHour set to 7, 11, and 17.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to Log Schedule Time node  
    - Edge Cases: Misconfigured timezone settings may cause unexpected trigger times.

  - **Log Schedule Time**  
    - Type: Code (JavaScript)  
    - Role: Logs the time the trigger was activated and outputs a message.  
    - Configuration: Returns JSON with message "Schedule Trigger aktif.", current ISO timestamp, and captured trigger time if available.  
    - Inputs: From Schedule Trigger node  
    - Outputs: Connects to Generate Image Prompt (Gemini) node  
    - Edge Cases: None significant; minor risk if $json.time is undefined or missing.

#### 2.2 AI Prompt Generation (Google Gemini)

- **Overview:** Generates a unique AI art prompt in one of the specified styles using Google Gemini model via LangChain.
- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Generate Image Prompt (Gemini)

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Provides AI language model capabilities for prompt generation.  
    - Configuration: Uses model "models/gemini-2.5-flash-lite-preview-06-17" with default options. Credentials use Google PaLM API key.  
    - Inputs: None (used as a language model resource linked to the chain)  
    - Outputs: Linked internally to Generate Image Prompt (Gemini)  
    - Edge Cases: Possible API authentication errors, quota limits, or network timeouts.

  - **Generate Image Prompt (Gemini)**  
    - Type: LangChain Chain LLM  
    - Role: Defines and executes the prompt generation logic using the Google Gemini model.  
    - Configuration: Text prompt instructs to create one random AI image prompt in cinematic, surreal, steampunk, or retro futuristic style; output is strictly the prompt text without explanation.  
    - Inputs: From Log Schedule Time  
    - Outputs: Connects to Create Pollinations URL node  
    - Edge Cases: Expression evaluation failures or model response errors.

#### 2.3 Image URL Construction (Pollinations AI)

- **Overview:** Constructs a URL that requests an AI-generated image from Pollinations AI service using the prompt text.
- **Nodes Involved:**  
  - Create Pollinations URL

- **Node Details:**

  - **Create Pollinations URL**  
    - Type: Code (JavaScript)  
    - Role: Builds the image request URL dynamically with parameters: width=1024, height=1024, a random seed for variety, model=flux, and disables logo display.  
    - Configuration: Reads prompt text from input JSON, encodes it for URL, appends parameters, returns JSON with prompt text and image URL.  
    - Inputs: From Generate Image Prompt (Gemini)  
    - Outputs: Connects to Fetch AI Image node  
    - Edge Cases: Encoding errors if prompt contains unusual characters; random seed edge cases unlikely.

#### 2.4 Image Retrieval

- **Overview:** Downloads the AI-generated image from Pollinations AI using the constructed URL.
- **Nodes Involved:**  
  - Fetch AI Image

- **Node Details:**

  - **Fetch AI Image**  
    - Type: HTTP Request  
    - Role: Performs the HTTP GET request to download the image binary data.  
    - Configuration: URL is set dynamically from incoming JSON's imageUrl property.  
    - Inputs: From Create Pollinations URL  
    - Outputs: Connects to Post Image to Facebook node  
    - Edge Cases: HTTP failures (404 if image missing, 429 rate limits, network errors), connection timeouts.

#### 2.5 Facebook Posting

- **Overview:** Posts the downloaded AI image to a Facebook Page with the prompt as caption and hashtags.
- **Nodes Involved:**  
  - Post Image to Facebook

- **Node Details:**

  - **Post Image to Facebook**  
    - Type: Facebook Graph API  
    - Role: Uploads photo to Facebook Page's photos edge, posting it publicly.  
    - Configuration: Posts to "me/photos" edge (representing the authenticated page), sends binary image data, attaches prompt text as "prompt" and caption with hashtags. Uses Facebook Graph API v22.0, HTTP POST.  
    - Inputs: From Fetch AI Image (binary data expected in "data")  
    - Outputs: None (end of workflow)  
    - Credentials: Facebook Page token with `pages_manage_posts` permission required.  
    - Edge Cases: Auth token expiry, permission errors, API limits, binary data upload failures.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                            | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                                   |
|----------------------------|----------------------------------|--------------------------------------------|-----------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| Sticky Note                | Sticky Note                      | Informational note about workflow purpose | None                  | None                     | 🌟 Automated AI Image Creation with Gemini and Facebook Page Integration... Customize prompt, model, or post styling as needed. |
| Set The Schedule            | Schedule Trigger                 | Triggers workflow at 7 AM, 11 AM, 5 PM    | None                  | Log Schedule Time        |                                                                                                               |
| Log Schedule Time           | Code                            | Logs trigger event time                     | Set The Schedule       | Generate Image Prompt (Gemini) |                                                                                                               |
| Google Gemini Chat Model    | LangChain Google Gemini Model   | Provides AI LLM for prompt generation      | None (resource node)   | Generate Image Prompt (Gemini) |                                                                                                               |
| Generate Image Prompt (Gemini) | LangChain Chain LLM           | Generates AI image prompt text              | Log Schedule Time, Google Gemini Chat Model | Create Pollinations URL |                                                                                                               |
| Create Pollinations URL     | Code                            | Builds Pollinations AI image request URL   | Generate Image Prompt (Gemini) | Fetch AI Image           |                                                                                                               |
| Fetch AI Image              | HTTP Request                    | Downloads AI-generated image                | Create Pollinations URL | Post Image to Facebook   |                                                                                                               |
| Post Image to Facebook      | Facebook Graph API              | Uploads image and caption to Facebook Page | Fetch AI Image         | None                     |                                                                                                               |
| Sticky Note1               | Sticky Note                      | Recommended node renaming for clarity       | None                  | None                     | 🧱 Recommended Node Renaming (for clarity)... Suggested names include Schedule Trigger (3x daily), Generate Image Prompt (Gemini), etc. |
| Sticky Note2               | Sticky Note                      | Publishing tips                             | None                  | None                     | 🛠 Tips for Publishing: Add workflow image, record Loom video, remove sensitive info before export.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**:  
   - Name: Set The Schedule  
   - Type: Schedule Trigger  
   - Set interval triggers at 7:00, 11:00, and 17:00 (5 PM), recurring every day.  
   - Save.

2. **Create Code node**:  
   - Name: Log Schedule Time  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     return [{
       json: {
         message: "Schedule Trigger aktif.",
         triggeredAt: new Date().toISOString(),
         time: $json.time || "Not Found"
       }
     }];
     ```  
   - Connect input from Set The Schedule node.

3. **Add LangChain Google Gemini Chat Model node**:  
   - Name: Google Gemini Chat Model  
   - Type: LangChain Google Gemini Chat Model  
   - Configure credential with valid Google PaLM API key.  
   - Set modelName to `models/gemini-2.5-flash-lite-preview-06-17`.  
   - No additional options required.

4. **Add LangChain Chain LLM node**:  
   - Name: Generate Image Prompt (Gemini)  
   - Type: LangChain Chain LLM  
   - In parameters, set promptType to "define".  
   - Text prompt:  
     ```
     Create 1 random AI image prompt in the style of [cinematic / surreal / steampunk / retro futuristic]. The prompt should be unique, fantastic, and full of imagination. the prompt should be in good and correct English.

     Output Without any additional explanation
     ```  
   - Connect input from Log Schedule Time node.  
   - Link Google Gemini Chat Model node as the AI languageModel resource.

5. **Add Code node**:  
   - Name: Create Pollinations URL  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const width = 1024;
     const height = 1024;
     const randomSeed = Math.floor(Math.random() * 100000);

     const finalPrompt = $json.text;
     const imageUrl = `https://image.pollinations.ai/prompt/${encodeURIComponent(finalPrompt)}.jpg?width=${width}&height=${height}&seed=${randomSeed}&model=flux&nologo=true`;

     return [{
       json: {
         text: finalPrompt,
         imageUrl
       }
     }];
     ```  
   - Connect input from Generate Image Prompt (Gemini).

6. **Add HTTP Request node**:  
   - Name: Fetch AI Image  
   - Type: HTTP Request  
   - Set URL to `={{ $json.imageUrl }}` (expression).  
   - Method: GET (default)  
   - Expect binary response; no additional options needed for this configuration.  
   - Connect input from Create Pollinations URL.

7. **Add Facebook Graph API node**:  
   - Name: Post Image to Facebook  
   - Type: Facebook Graph API  
   - Set node to `me`, edge to `photos`.  
   - HTTP Method: POST  
   - Binary Property Name: `data` (this expects the binary image data from the previous node).  
   - In options, add query parameters:  
     - `prompt` = `={{ $json.text }}`  
     - `caption` = `={{ $json.text }}\n\n\n#FreeImage #AIGeneratedArt #CreativeFreedom #FreeToUse #freeimagegenerator`  
   - Configure credentials with a Facebook Page token authorized with `pages_manage_posts` permission.  
   - Connect input from Fetch AI Image.

8. **Connect the nodes sequentially**:  
   - Set The Schedule → Log Schedule Time → Generate Image Prompt (Gemini) → Create Pollinations URL → Fetch AI Image → Post Image to Facebook.

9. **Additional configuration**:  
   - Ensure all credentials (Google PaLM API key and Facebook token) are properly set up and tested.  
   - Confirm timezone settings for schedule trigger to match desired local time.  
   - Optional: Add sticky notes for documentation and clarity (as per original workflow).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 🌟 Automated AI Image Creation with Gemini and Facebook Page Integration. Customize prompt, model, or post styling as needed.                                                          | Sticky Note at workflow start explains overall purpose and requirements.                           |
| 🧱 Recommended Node Renaming for clarity: rename nodes to descriptive names like "Schedule Trigger (3x daily)", "Generate Image Prompt (Gemini)", "Post Image to Facebook", etc.          | Sticky Note1                                                                                       |
| 🛠 Tips for Publishing: Add a workflow image, record a Loom video for visual walkthrough, remove sensitive info from exports.                                                         | Sticky Note2                                                                                       |
| Pollinations AI image generation requires no API key; Google Gemini requires valid PaLM API key; Facebook posting requires Page token with `pages_manage_posts` permission.             | Workflow description and credentials notes.                                                       |
| Facebook Graph API used version v22.0; ensure token and permissions are current to avoid auth errors.                                                                                   | Post Image to Facebook node configuration.                                                        |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.