Personalize Instagram Trends for Custom Skin Tones with GPT-4o, DALL-E & Slack

https://n8nworkflows.xyz/workflows/personalize-instagram-trends-for-custom-skin-tones-with-gpt-4o--dall-e---slack-11271


# Personalize Instagram Trends for Custom Skin Tones with GPT-4o, DALL-E & Slack

### 1. Workflow Overview

This workflow automates the process of discovering trending Instagram nail art designs, personalizing them for specific skin tones, generating new images with AI, and delivering the results to a Slack channel. It targets beauty professionals, stylists, or content creators who want to remix current Instagram trends into personalized visual content.

The workflow is logically divided into three main blocks:

- **1.1 Setup & Data Retrieval:** Configure search parameters (hashtags and skin tone), and scrape Instagram posts using Apify’s Instagram hashtag scraper actor.
- **1.2 AI Analysis & Image Generation:** Use GPT-4o to analyze the scraped trend images and generate a personalized prompt; then generate a new image with DALL-E 3 based on that prompt.
- **1.3 Result Delivery:** Upload the generated image to a specified Slack channel for team review or sharing.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Data Retrieval

**Overview:**  
This block initializes configuration parameters such as Instagram hashtags and target skin tone, triggers the Apify actor to scrape Instagram posts with those hashtags, and prepares data for AI processing.

**Nodes Involved:**  
- When clicking 'Test workflow' (Manual Trigger)  
- Workflow Configuration (Set node)  
- Run an Actor and get dataset (HTTP Request)  
- Combine Data (Merge node)

**Node Details:**

- **When clicking 'Test workflow'**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow for testing purposes.  
  - Config: No parameters; simply triggers the flow.  
  - Connections: Outputs to "Workflow Configuration".  
  - Failure cases: None typical as no external API calls.  

- **Workflow Configuration**  
  - Type: Set  
  - Role: Holds key workflow input parameters to customize behavior.  
  - Config:  
    - `apifyApiToken`: Placeholder for Apify API token (string).  
    - `hashtags`: Comma-separated Instagram hashtags to scrape, e.g., "nuancenail,trendnails".  
    - `resultsLimit`: Number of posts to retrieve per hashtag (default 1).  
    - `skinTone`: Target skin tone personalization, e.g., "Yellow Base Spring".  
  - Connections: Input from Manual Trigger; output to "Run an Actor and get dataset" and "Combine Data".  
  - Edge cases: Missing or invalid API token will cause downstream HTTP failures. Improper hashtag format may result in empty data.  

- **Run an Actor and get dataset**  
  - Type: HTTP Request  
  - Role: Calls Apify’s Instagram hashtag scraper actor API to perform scraping based on configured hashtags and limit.  
  - Config:  
    - Method: POST to Apify API endpoint for running the actor.  
    - URL: Dynamically includes Apify token from Workflow Configuration.  
    - Body: JSON with `hashtags` parsed as an array from Workflow Configuration and `resultsLimit`.  
    - Wait for finish: 120 seconds max.  
  - Connections: Input from Workflow Configuration; output to "Combine Data".  
  - Failure types: HTTP errors (401 Unauthorized if token invalid, 408 timeout if scraping slow), malformed body errors, or API rate limiting.  
  - Version: Requires Apify actor `instagram-hashtag-scraper` available in user account.  

- **Combine Data**  
  - Type: Merge  
  - Role: Combines data streams from "Workflow Configuration" and "Run an Actor and get dataset" for downstream AI processing.  
  - Config: Combine mode by position (combines one item from each input by their order).  
  - Inputs: From both Workflow Configuration (position 1) and Run an Actor and get dataset (position 0).  
  - Outputs: To "OpenAI - Analyze Image & Generate Prompt".  
  - Failure cases: Mismatched data or empty inputs can cause no output or errors in following nodes.  

---

#### 2.2 AI Analysis & Image Generation

**Overview:**  
This block uses OpenAI’s GPT-4o to analyze the scraped Instagram trends and generate a personalized textual prompt. Then DALL-E 3 creates a high-resolution image based on this prompt.

**Nodes Involved:**  
- OpenAI - Analyze Image & Generate Prompt (LangChain OpenAI node)  
- OpenAI - Generate DALL-E Image (LangChain OpenAI node)

**Node Details:**

- **OpenAI - Analyze Image & Generate Prompt**  
  - Type: LangChain OpenAI (Chat Completion)  
  - Role: Sends a message request to GPT-4o to analyze the Instagram trend data and generate a prompt customized by skin tone.  
  - Config:  
    - Operation: Message (chat-based language model).  
    - Input: Receives combined data with Instagram post data and user skin tone.  
    - Credentials: Uses OpenAI API key with GPT-4o access.  
  - Connections: Input from "Combine Data"; output to "OpenAI - Generate DALL-E Image".  
  - Failure cases: API key invalid, rate limit, model unavailable, or malformed input causing prompt generation failures.  
  - Version: Requires access to GPT-4o model via OpenAI.  

- **OpenAI - Generate DALL-E Image**  
  - Type: LangChain OpenAI (Image Generation)  
  - Role: Generates a new image from the prompt text provided by GPT-4o analysis.  
  - Config:  
    - Prompt: Uses expression to extract generated prompt text from previous node output.  
    - Image size: 1024x1024 pixels.  
    - Quality: High definition (HD).  
    - Credentials: Uses same OpenAI API key with DALL-E access.  
  - Connections: Input from "OpenAI - Analyze Image & Generate Prompt"; output to "Post to Slack".  
  - Failure cases: Invalid prompt, API errors, quota limits, or image generation timeouts.  
  - Version: Requires OpenAI DALL-E 3 support.  

---

#### 2.3 Result Delivery

**Overview:**  
This block uploads the generated AI image to a specified Slack channel using OAuth2 authentication for team collaboration and review.

**Nodes Involved:**  
- Post to Slack (Slack node)

**Node Details:**

- **Post to Slack**  
  - Type: Slack  
  - Role: Uploads the generated image file to a designated Slack channel.  
  - Config:  
    - Resource: File upload.  
    - Channel ID: Hardcoded to Slack channel ID "C09UDKC3G2W".  
    - Authentication: OAuth2 with Slack credentials.  
    - Input: Uses output from DALL-E image generation node.  
  - Connections: Input from "OpenAI - Generate DALL-E Image".  
  - Failure cases: OAuth token expiration, channel permission issues, file size limits on Slack, or network errors.  
  - Version: Requires configured OAuth2 Slack app with file upload permissions.  

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                 | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                             |
|-------------------------------|---------------------------------|--------------------------------|---------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow'  | Manual Trigger                  | Entry point for manual execution | None                            | Workflow Configuration                  |                                                                                                       |
| Workflow Configuration         | Set                             | Define API token, hashtags, skin tone, and result limits | When clicking 'Test workflow'    | Run an Actor and get dataset, Combine Data | ## 1. Setup & Get Data Configure your search tags and style settings. The workflow scrapes Instagram trends via Apify and merges them with your preferences. |
| Run an Actor and get dataset   | HTTP Request                    | Call Apify Instagram scraper actor API | Workflow Configuration           | Combine Data                           | ## 1. Setup & Get Data Configure your search tags and style settings. The workflow scrapes Instagram trends via Apify and merges them with your preferences. |
| Combine Data                  | Merge                           | Combine config and scraped data | Workflow Configuration, Run an Actor and get dataset | OpenAI - Analyze Image & Generate Prompt | ## 1. Setup & Get Data Configure your search tags and style settings. The workflow scrapes Instagram trends via Apify and merges them with your preferences. |
| OpenAI - Analyze Image & Generate Prompt | LangChain OpenAI (Chat Completion) | Analyze Instagram trend and create personalized prompt | Combine Data                     | OpenAI - Generate DALL-E Image          | ## 2. AI Analysis & Remix GPT-4 analyzes the visual style of the Instagram trends. Then, DALL-E 3 generates a brand new design based on that analysis. |
| OpenAI - Generate DALL-E Image | LangChain OpenAI (Image Generation) | Generate image from prompt text | OpenAI - Analyze Image & Generate Prompt | Post to Slack                         | ## 2. AI Analysis & Remix GPT-4 analyzes the visual style of the Instagram trends. Then, DALL-E 3 generates a brand new design based on that analysis. |
| Post to Slack                 | Slack                           | Upload generated image to Slack channel | OpenAI - Generate DALL-E Image  | None                                   | ## 3. Deliver Result Finally, the generated image is uploaded directly to your Slack channel for review. |
| Sticky Note                   | Sticky Note                    | Documentation and instructions | None                            | None                                   | # Remix Instagram Trends with AI to Slack This workflow automates the process of finding visual trends on Instagram and remixing them to suit specific personal features (like skin tone) using AI. It is perfect for stylists, beauty consultants, or content creators. [See Setup Requirements](https://apify.com/actors/instagram-hashtag-scraper) |
| Sticky Note1                  | Sticky Note                    | Block 1 description             | None                            | None                                   | ## 1. Setup & Get Data Configure your search tags and style settings. The workflow scrapes Instagram trends via Apify and merges them with your preferences. |
| Sticky Note2                  | Sticky Note                    | Block 2 description             | None                            | None                                   | ## 2. AI Analysis & Remix GPT-4 analyzes the visual style of the Instagram trends. Then, DALL-E 3 generates a brand new design based on that analysis. |
| Sticky Note3                  | Sticky Note                    | Block 3 description             | None                            | None                                   | ## 3. Deliver Result Finally, the generated image is uploaded directly to your Slack channel for review. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: "When clicking 'Test workflow'"  
   - Type: Manual trigger (no parameters).  

2. **Create Set Node for Configuration:**  
   - Name: "Workflow Configuration"  
   - Type: Set  
   - Assign variables:  
     - `apifyApiToken` (string): Your Apify API token (replace placeholder).  
     - `hashtags` (string): Comma-separated Instagram hashtags, e.g., "nuancenail,trendnails".  
     - `resultsLimit` (number): Number of posts to retrieve (default 1).  
     - `skinTone` (string): Target skin tone, e.g., "Yellow Base Spring".  
   - Connect output of Manual Trigger to this node.  

3. **Create HTTP Request Node to Run Apify Actor:**  
   - Name: "Run an Actor and get dataset"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/reGe1ST3OBgYZSsZJ/runs?token={{ $json.apifyApiToken }}&waitForFinish=120`  
   - Body type: JSON  
   - Body content:  
     ```json
     {
       "hashtags": {{ $json.hashtags.split(',') }},
       "resultsLimit": {{ $json.resultsLimit }}
     }
     ```  
   - Connect output of "Workflow Configuration" to this node.  

4. **Create Merge Node:**  
   - Name: "Combine Data"  
   - Type: Merge  
   - Mode: Combine  
   - Combine by: Position  
   - Connect outputs of both "Workflow Configuration" (to second input) and "Run an Actor and get dataset" (to first input) to this node.  

5. **Create OpenAI Chat Node for Analysis:**  
   - Name: "OpenAI - Analyze Image & Generate Prompt"  
   - Type: LangChain OpenAI (Chat Completion)  
   - Operation: Message  
   - Credentials: Configure OpenAI API key with GPT-4o access.  
   - Connect "Combine Data" output to this node input.  

6. **Create OpenAI Image Generation Node:**  
   - Name: "OpenAI - Generate DALL-E Image"  
   - Type: LangChain OpenAI (Image Generation)  
   - Prompt: Use expression to extract prompt text from previous node output, e.g., `={{ $json.output[0].content[0].text }}`  
   - Options:  
     - Size: 1024x1024  
     - Quality: HD  
   - Credentials: Use same OpenAI API key with DALL-E access.  
   - Connect output of "OpenAI - Analyze Image & Generate Prompt" to this node.  

7. **Create Slack Node for Posting:**  
   - Name: "Post to Slack"  
   - Type: Slack  
   - Resource: File upload  
   - Channel ID: Set to your Slack channel ID (e.g., "C09UDKC3G2W").  
   - Authentication: OAuth2  
   - Credentials: Configure Slack OAuth2 account with required permissions for file upload.  
   - Connect output of "OpenAI - Generate DALL-E Image" to this node.  

8. **Optional:** Add Sticky Notes for documentation and grouping as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow automates remixing Instagram trends personalized by skin tone using Apify, OpenAI GPT-4o, DALL-E, and Slack integration. | Project overview and purpose.                                                                                     |
| Setup requires: Apify account with instagram-hashtag-scraper actor; OpenAI API key with GPT-4o & DALL-E access; Slack OAuth2 app with file upload permissions. | Setup Requirements.                                                                                               |
| See Apify Instagram Hashtag Scraper actor details and registration at https://apify.com/actors/instagram-hashtag-scraper | Apify actor documentation and signup.                                                                            |
| Slack OAuth2 app must have scopes: `files:write` and channel access for uploading images.                         | Slack app configuration notes.                                                                                    |
| The workflow is well-suited for stylists, beauty consultants, and content creators looking to generate personalized Instagram-inspired content. | Target audience and use case.                                                                                     |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.