Create AI News Videos with HeyGen Avatars and Auto-Post to Social Media

https://n8nworkflows.xyz/workflows/create-ai-news-videos-with-heygen-avatars-and-auto-post-to-social-media-3538


# Create AI News Videos with HeyGen Avatars and Auto-Post to Social Media

### 1. Workflow Overview

This workflow automates the creation and distribution of short AI avatar videos based on trending news articles. It is designed for content creators, businesses, and freelancers who want to scale video content production and social media posting with minimal manual intervention. The workflow integrates multiple services—Hacker News for trending news, OpenAI’s ChatGPT for script and caption generation, HeyGen for AI avatar video creation, and Blotato for multi-platform social media posting.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & News Fetching:** Periodically triggers the workflow and retrieves trending news articles from Hacker News.
- **1.2 AI Script and Caption Generation:** Uses OpenAI’s language models to generate video scripts and captions tailored for social media platforms.
- **1.3 AI Avatar Video Creation:** Configures HeyGen parameters and creates AI avatar videos based on the generated scripts.
- **1.4 Video Processing and Preparation:** Waits for video processing completion and prepares the video data for publishing.
- **1.5 Social Media Publishing:** Uploads the video and posts it across multiple social media platforms via Blotato.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & News Fetching

**Overview:**  
This block initiates the workflow on a schedule and fetches trending news articles from Hacker News to serve as content sources.

**Nodes Involved:**  
- Schedule Trigger  
- AI Agent  
- Fetch HN Front Page  
- Fetch HN Article  
- Write Script  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow at defined intervals (e.g., daily or hourly).  
  - Configuration: Default scheduling parameters (not explicitly shown).  
  - Inputs: None (trigger node).  
  - Outputs: Connects to AI Agent.  
  - Edge Cases: Misconfiguration may cause no triggering or excessive runs.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Coordinates AI tasks, managing calls to language models and tools.  
  - Configuration: Uses integrated tools for fetching news and generating text.  
  - Inputs: Triggered by Schedule Trigger.  
  - Outputs: Sends data to "Write Long Caption" and accepts input from "Fetch HN Article" and "Fetch HN Front Page".  
  - Edge Cases: API rate limits, model errors, or tool failures.

- **Fetch HN Front Page**  
  - Type: Hacker News Tool  
  - Role: Retrieves the list of trending articles from Hacker News front page.  
  - Configuration: Default Hacker News front page fetch.  
  - Inputs: Invoked by AI Agent as a tool.  
  - Outputs: Provides article list to AI Agent.  
  - Edge Cases: Network errors, API downtime.

- **Fetch HN Article**  
  - Type: Hacker News Tool  
  - Role: Fetches detailed content of a selected Hacker News article.  
  - Configuration: Uses article ID from front page fetch.  
  - Inputs: Invoked by AI Agent as a tool.  
  - Outputs: Article content to AI Agent.  
  - Edge Cases: Missing article data, API errors.

- **Write Script**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Generates a short video script based on the fetched news article.  
  - Configuration: Uses OpenAI ChatGPT with a prompt tailored to produce a concise script.  
  - Inputs: Receives article content from AI Agent.  
  - Outputs: Sends script to AI Agent for further processing.  
  - Edge Cases: API errors, prompt failures, or unexpected output format.

---

#### 2.2 AI Script and Caption Generation

**Overview:**  
Generates both long and short captions for social media posts using OpenAI models, based on the script and article content.

**Nodes Involved:**  
- Write Long Caption  
- Write Short Caption  
- Setup Heygen  

**Node Details:**

- **Write Long Caption**  
  - Type: OpenAI Node (LangChain)  
  - Role: Creates a detailed caption for social media posts.  
  - Configuration: Uses OpenAI with a prompt designed for longer, engaging captions.  
  - Inputs: Receives data from AI Agent.  
  - Outputs: Passes output to "Write Short Caption".  
  - Edge Cases: API rate limits, generation errors.

- **Write Short Caption**  
  - Type: OpenAI Node (LangChain)  
  - Role: Produces a concise caption suitable for platforms with character limits.  
  - Configuration: Uses OpenAI with a prompt for brevity and impact.  
  - Inputs: Receives long caption output.  
  - Outputs: Connects to "Setup Heygen".  
  - Edge Cases: Similar to Write Long Caption.

- **Setup Heygen**  
  - Type: Set Node  
  - Role: Configures HeyGen API parameters including API key, avatar ID, and voice ID.  
  - Configuration: User must input HeyGen credentials and IDs here.  
  - Inputs: Receives short caption data.  
  - Outputs: Sends configuration to "Create Avatar Video".  
  - Edge Cases: Missing or invalid credentials cause API call failures.

---

#### 2.3 AI Avatar Video Creation

**Overview:**  
Creates the AI avatar video on HeyGen using the generated script and configured avatar/voice parameters.

**Nodes Involved:**  
- Create Avatar Video  
- Wait  

**Node Details:**

- **Create Avatar Video**  
  - Type: HTTP Request  
  - Role: Sends a request to HeyGen API to create a video with the script and avatar settings.  
  - Configuration: Uses POST method with JSON body containing script, avatar ID, voice ID, and other parameters.  
  - Inputs: Receives HeyGen setup data.  
  - Outputs: Passes video creation response to "Wait".  
  - Edge Cases: API errors, invalid parameters, rate limits.

- **Wait**  
  - Type: Wait Node  
  - Role: Pauses workflow to allow HeyGen video processing to complete.  
  - Configuration: Default or user-defined wait time (e.g., 2 minutes for testing).  
  - Inputs: Triggered after video creation request.  
  - Outputs: Connects to "Get Avatar Video".  
  - Edge Cases: Insufficient wait time may cause premature requests.

---

#### 2.4 Video Processing and Preparation

**Overview:**  
Retrieves the completed video from HeyGen and prepares it for publishing by setting necessary metadata.

**Nodes Involved:**  
- Get Avatar Video  
- Prepare for Publish  

**Node Details:**

- **Get Avatar Video**  
  - Type: HTTP Request  
  - Role: Fetches the processed video file or URL from HeyGen API.  
  - Configuration: Uses GET method with video ID from creation response.  
  - Inputs: Triggered after wait period.  
  - Outputs: Sends video data to "Prepare for Publish".  
  - Edge Cases: Video not ready, API errors.

- **Prepare for Publish**  
  - Type: Set Node  
  - Role: Sets parameters for Blotato API including API key, account IDs, page IDs, and formats video data for upload.  
  - Configuration: User must input Blotato credentials and target social media details.  
  - Inputs: Receives video data.  
  - Outputs: Connects to "Upload to Blotato".  
  - Edge Cases: Missing credentials, incorrect formatting.

---

#### 2.5 Social Media Publishing

**Overview:**  
Uploads the video and posts it to multiple social media platforms via Blotato’s API.

**Nodes Involved:**  
- Upload to Blotato  
- [Instagram] Publish via Blotato  
- [Facebook] Publish via Blotato (disabled by default)  
- [Linkedin] Publish via Blotato (disabled by default)  
- [Tiktok] Publish via Blotato  
- [Youtube] Publish via Blotato  
- [Threads] Publish via Blotato (disabled by default)  
- [Twitter] Publish via Blotato  
- [Bluesky] Publish via Blotato (disabled by default)  
- Upload to Blotato - Image (disabled)  
- [Pinterest] Publish via Blotato (disabled)  

**Node Details:**

- **Upload to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads the video file to Blotato’s media storage.  
  - Configuration: POST request with video file and metadata.  
  - Inputs: Receives prepared video data.  
  - Outputs: Triggers multiple platform-specific publish nodes.  
  - Edge Cases: Upload failures, API errors.

- **[Platform] Publish via Blotato** (e.g., Instagram, Facebook, TikTok, etc.)  
  - Type: HTTP Request  
  - Role: Posts the uploaded video to the respective social media platform via Blotato API.  
  - Configuration: POST requests with platform-specific parameters such as account ID, page ID, captions, and media references.  
  - Inputs: Triggered by "Upload to Blotato".  
  - Outputs: None (end nodes).  
  - Edge Cases: Platform-specific API errors, disabled nodes by default to allow selective publishing.

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                        | Input Node(s)             | Output Node(s)                                                                                   | Sticky Note                                                                                      |
|----------------------------|------------------------------|-------------------------------------|---------------------------|------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger             | Starts workflow on schedule          | None                      | AI Agent                                                                                       |                                                                                                |
| AI Agent                  | LangChain Agent              | Coordinates AI tasks                 | Schedule Trigger           | Write Long Caption, Fetch HN Article, Fetch HN Front Page                                      |                                                                                                |
| Fetch HN Front Page       | Hacker News Tool             | Fetch trending news list             | AI Agent (tool)            | AI Agent                                                                                       |                                                                                                |
| Fetch HN Article          | Hacker News Tool             | Fetch detailed article content       | AI Agent (tool)            | AI Agent                                                                                       |                                                                                                |
| Write Script              | LangChain OpenAI Chat Model  | Generate video script                | AI Agent                   | AI Agent                                                                                       |                                                                                                |
| Write Long Caption        | OpenAI Node (LangChain)      | Generate long social media caption  | AI Agent                   | Write Short Caption                                                                            |                                                                                                |
| Write Short Caption       | OpenAI Node (LangChain)      | Generate short social media caption | Write Long Caption         | Setup Heygen                                                                                   |                                                                                                |
| Setup Heygen              | Set Node                    | Configure HeyGen API parameters      | Write Short Caption        | Create Avatar Video                                                                           | User must add HeyGen API key, avatar ID, voice ID                                              |
| Create Avatar Video       | HTTP Request                | Request HeyGen to create video       | Setup Heygen               | Wait                                                                                          |                                                                                                |
| Wait                      | Wait Node                   | Pause for video processing           | Create Avatar Video        | Get Avatar Video                                                                             |                                                                                                |
| Get Avatar Video          | HTTP Request                | Retrieve processed video             | Wait                      | Prepare for Publish                                                                          |                                                                                                |
| Prepare for Publish       | Set Node                    | Configure Blotato API parameters     | Get Avatar Video           | Upload to Blotato                                                                            | User must add Blotato API key, account IDs, page IDs                                           |
| Upload to Blotato         | HTTP Request                | Upload video to Blotato              | Prepare for Publish        | [Instagram], [Facebook], [Linkedin], [Tiktok], OpenAI, [Youtube], [Threads], [Twitter], [Bluesky] |                                                                                                |
| [Instagram] Publish via Blotato | HTTP Request          | Post video to Instagram              | Upload to Blotato          | None                                                                                         |                                                                                                |
| [Facebook] Publish via Blotato | HTTP Request            | Post video to Facebook (disabled)    | Upload to Blotato          | None                                                                                         | Disabled by default                                                                            |
| [Linkedin] Publish via Blotato | HTTP Request            | Post video to LinkedIn (disabled)    | Upload to Blotato          | None                                                                                         | Disabled by default                                                                            |
| [Tiktok] Publish via Blotato | HTTP Request              | Post video to TikTok                 | Upload to Blotato          | None                                                                                         |                                                                                                |
| OpenAI                    | OpenAI Node                 | (Disabled) Unused node               | Upload to Blotato          | Upload to Blotato - Image                                                                    | Disabled                                                                                      |
| Upload to Blotato - Image | HTTP Request                | (Disabled) Upload image to Blotato   | OpenAI                    | [Pinterest] Publish via Blotato                                                              | Disabled                                                                                      |
| [Pinterest] Publish via Blotato | HTTP Request            | Post image to Pinterest (disabled)   | Upload to Blotato - Image  | None                                                                                         | Disabled by default                                                                            |
| [Youtube] Publish via Blotato | HTTP Request              | Post video to YouTube                | Upload to Blotato          | None                                                                                         |                                                                                                |
| [Threads] Publish via Blotato | HTTP Request              | Post video to Threads (disabled)     | Upload to Blotato          | None                                                                                         | Disabled by default                                                                            |
| [Twitter] Publish via Blotato | HTTP Request              | Post video to Twitter                | Upload to Blotato          | None                                                                                         |                                                                                                |
| [Bluesky] Publish via Blotato | HTTP Request              | Post video to Bluesky (disabled)     | Upload to Blotato          | None                                                                                         | Disabled by default                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set the desired interval (e.g., daily).  
   - No credentials needed.

3. **Add an AI Agent node (LangChain Agent):**  
   - Connect Schedule Trigger output to AI Agent input.  
   - Configure with OpenAI credentials for language model access.  
   - Set up tools for Hacker News fetching (see next steps).

4. **Add Hacker News Tool nodes:**  
   - "Fetch HN Front Page": Configure to fetch trending articles.  
   - "Fetch HN Article": Configure to fetch article details by ID.  
   - Connect these as tools to the AI Agent node.

5. **Add a LangChain OpenAI Chat Model node ("Write Script"):**  
   - Connect AI Agent’s language model output to this node.  
   - Configure prompt to generate a short video script from article content.  
   - Use OpenAI credentials.

6. **Connect "Write Script" output back to AI Agent for further processing.**

7. **Add OpenAI nodes for captions:**  
   - "Write Long Caption": Connect from AI Agent output. Configure prompt for long caption generation.  
   - "Write Short Caption": Connect from "Write Long Caption". Configure prompt for short caption generation.

8. **Add a Set node ("Setup Heygen"):**  
   - Connect from "Write Short Caption".  
   - Add parameters: HeyGen API key, Avatar ID, Voice ID (user inputs).  
   - This node prepares the payload for HeyGen API.

9. **Add HTTP Request node ("Create Avatar Video"):**  
   - Connect from "Setup Heygen".  
   - Configure POST request to HeyGen API endpoint to create video.  
   - Include script, avatar, and voice parameters in JSON body.  
   - Use HeyGen API credentials.

10. **Add Wait node:**  
    - Connect from "Create Avatar Video".  
    - Set wait time (e.g., 2 minutes for testing, longer for production).

11. **Add HTTP Request node ("Get Avatar Video"):**  
    - Connect from Wait node.  
    - Configure GET request to HeyGen API to retrieve video URL or file.  
    - Use video ID from previous response.

12. **Add Set node ("Prepare for Publish"):**  
    - Connect from "Get Avatar Video".  
    - Add Blotato API key, account IDs, page IDs, and prepare video data for upload.

13. **Add HTTP Request node ("Upload to Blotato"):**  
    - Connect from "Prepare for Publish".  
    - Configure POST request to upload video to Blotato media storage.  
    - Use Blotato API credentials.

14. **Add HTTP Request nodes for each social media platform:**  
    - Connect each from "Upload to Blotato".  
    - Configure POST requests to Blotato API endpoints for Instagram, Facebook, LinkedIn, TikTok, YouTube, Threads, Twitter, Bluesky, Pinterest (optional).  
    - Include platform-specific parameters such as account/page IDs and captions.  
    - Disable nodes for platforms not used initially.

15. **Test the workflow:**  
    - Start with a short script duration (5 seconds).  
    - Enable only one social media platform.  
    - Adjust wait time to 2 minutes.  
    - Verify video creation and posting.

16. **Scale up:**  
    - Enable additional platforms.  
    - Increase script length and wait times as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow requires self-hosted n8n due to Community Nodes usage.                                           | Important for deployment environment.                                                                 |
| HeyGen Avatar & Voice IDs must be obtained from HeyGen Dashboard under "Avatars" and "Voices" sections.       | https://heygen.com/dashboard                                                                          |
| Blotato API keys and account/page IDs are required for social media posting.                                  | https://blotato.com/api                                                                                |
| Initial test run recommendations: shorten script, enable one platform, reduce wait time.                      | Workflow Overview section                                                                               |
| Video creation and posting are fully automated, enabling scaling of short-form video content production.      | Use Cases section                                                                                       |
| Community Nodes may not work on n8n Cloud without modification.                                                | Disclaimer section                                                                                      |
| Workflow diagram available in original documentation for visual reference.                                    | Provided in user documentation (image not included here).                                             |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Create AI News Videos with HeyGen Avatars and Auto-Post to Social Media" workflow in n8n. It covers all nodes, their roles, configurations, and integration points with external services.