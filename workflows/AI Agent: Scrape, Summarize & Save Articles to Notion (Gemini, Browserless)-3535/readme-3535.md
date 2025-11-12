AI Agent: Scrape, Summarize & Save Articles to Notion (Gemini, Browserless)

https://n8nworkflows.xyz/workflows/ai-agent--scrape--summarize---save-articles-to-notion--gemini--browserless--3535


# AI Agent: Scrape, Summarize & Save Articles to Notion (Gemini, Browserless)

### 1. Workflow Overview

This n8n workflow automates capturing web articles or links shared in chat conversations and saving them as structured pages in a Notion database. It leverages Google’s Gemini AI to interpret chat messages and orchestrate the process, Browserless for web scraping, and Notion’s API to store enriched article data. Additionally, it sends a confirmation notification to a Discord channel upon completion.

**Target Use Cases:**  
- Users who want to streamline saving and organizing web research or articles shared in chat without manual copy-pasting.  
- Teams or individuals maintaining a knowledge base in Notion with AI-generated summaries and metadata.  
- Automating web clipping workflows with AI-enhanced content extraction and summarization.

**Logical Blocks:**

- **1.1 Input Reception:** Triggered by incoming chat messages.  
- **1.2 AI Processing & Orchestration:** Google Gemini AI interprets the message, decides if it contains a URL to save, and manages subsequent tool executions.  
- **1.3 Web Scraping:** Browserless API scrapes the full content of the provided URL.  
- **1.4 Data Saving:** The scraped and AI-processed content is saved as a new page in a Notion database with rich formatting and metadata.  
- **1.5 Notification:** A Discord message confirms the successful saving or reports errors.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new chat messages to initiate the workflow.

- **Nodes Involved:**  
  - `When chat message received`

- **Node Details:**  
  - **Type:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  - **Role:** Webhook trigger node that activates the workflow when a chat message is received.  
  - **Configuration:**  
    - Public webhook enabled (accessible externally).  
    - No additional filters or options set.  
  - **Input/Output:**  
    - No input (trigger node).  
    - Output connects to the AI Processing node (`Save Article To Notion`).  
  - **Edge Cases:**  
    - Incoming messages without URLs or relevant content may still trigger the workflow but will be handled downstream by the AI.  
    - Webhook availability and network issues could prevent triggering.  
  - **Version:** 1.1

---

#### 2.2 AI Processing & Orchestration

- **Overview:**  
  This block uses Google Gemini AI to analyze the chat message, detect URLs, scrape the webpage, summarize content, save to Notion, and notify Discord. It acts as the central orchestrator.

- **Nodes Involved:**  
  - `Gemini 2.5 PRO`  
  - `Save Article To Notion`

- **Node Details:**  

  - **Gemini 2.5 PRO**  
    - **Type:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - **Role:** Provides AI language model capabilities to understand message context and generate instructions or content.  
    - **Configuration:**  
      - Model: `models/gemini-2.5-pro-exp-03-25` selected for large context window and speed.  
      - Temperature set to 0 for deterministic output.  
      - Credential: Google Gemini API key configured.  
    - **Input/Output:**  
      - Input: Receives chat message data from trigger node.  
      - Output: Feeds AI-generated content and instructions to the `Save Article To Notion` agent node.  
    - **Edge Cases:**  
      - API authentication errors if credentials are invalid.  
      - Rate limits or quota exhaustion on Google Gemini API.  
      - Unexpected AI output or failure to detect URLs.  
    - **Version:** 1

  - **Save Article To Notion**  
    - **Type:** `@n8n/n8n-nodes-langchain.agent`  
    - **Role:** Acts as an AI agent node that coordinates the use of tools: web scraper, Notion saver, and Discord notifier.  
    - **Configuration:**  
      - System message instructs the agent to:  
        1. Use `website_scraper` to scrape the URL.  
        2. Use `save_to_notion` to save data.  
        3. Use `discord_notification` to send confirmation.  
      - Detailed parameters expected by `save_to_notion` are enumerated (title, description, tags, publication date, summary, objectives, concepts, technologies, code snippets, conclusions, icon).  
      - `executeOnce` enabled to ensure single execution per trigger.  
      - On error, continues with error output to avoid halting workflow.  
    - **Input/Output:**  
      - Input: Receives AI chat output from `Gemini 2.5 PRO`.  
      - Output: None directly; internally calls tools.  
    - **Edge Cases:**  
      - Failure in any tool call (scraping, saving, notification) may cause partial workflow success.  
      - AI misinterpretation leading to incomplete or incorrect data extraction.  
      - Timeout or API errors from tools.  
    - **Version:** 1.7

---

#### 2.3 Web Scraping

- **Overview:**  
  This block scrapes the full content of the URL detected in the chat message using Browserless via an HTTP request.

- **Nodes Involved:**  
  - `website_scraper`

- **Node Details:**  
  - **Type:** `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - **Role:** Sends a POST request to Browserless API to retrieve webpage content.  
  - **Configuration:**  
    - URL: `http://browserless:3000/content` (assumes Browserless is self-hosted and accessible at this address).  
    - Method: POST  
    - Body (JSON): Includes the URL to scrape and Puppeteer `gotoOptions` with `waitUntil: networkidle0` to ensure page fully loads.  
    - Tool description: "website_scraper: Scrape a website given it's URL"  
    - Parameter placeholder `url` dynamically replaced with the target URL.  
  - **Input/Output:**  
    - Input: Receives URL parameter from AI agent node.  
    - Output: Returns scraped HTML/content to the AI agent node.  
  - **Edge Cases:**  
    - Browserless API unreachable or authentication failure if cloud version used without proper headers.  
    - Timeout or network errors during scraping.  
    - Pages with heavy JavaScript or anti-bot measures may not scrape correctly.  
  - **Version:** 1.1

---

#### 2.4 Data Saving to Notion

- **Overview:**  
  This block creates a new page in a Notion database with the AI-generated summary, metadata, and structured content extracted from the article.

- **Nodes Involved:**  
  - `save_to_notion`

- **Node Details:**  
  - **Type:** `n8n-nodes-base.notionTool`  
  - **Role:** Saves structured data into a Notion database page.  
  - **Configuration:**  
    - Database ID set to a specific Notion database (e.g., "Knowledge Database").  
    - Title property mapped to AI-generated article title.  
    - Rich content blocks configured with multiple headings and text sections populated dynamically from AI outputs (summary, objectives, concepts, technologies, code snippets, conclusions).  
    - Properties mapped include Description, URL, Tags, Publication Date.  
    - Icon set dynamically from AI-chosen emoji.  
    - Credential: Notion API key configured with access to the target database.  
  - **Input/Output:**  
    - Input: Receives AI-generated content and metadata from the agent node.  
    - Output: Returns the URL or ID of the created Notion page to the AI agent node.  
  - **Edge Cases:**  
    - Notion API rate limits or permission errors if integration lacks database access.  
    - Incorrect property mappings causing data loss or errors.  
    - Large content blocks exceeding Notion API limits.  
  - **Version:** 2.2

---

#### 2.5 Notification

- **Overview:**  
  Sends a confirmation message to a Discord channel indicating the article was saved or if an error occurred.

- **Nodes Involved:**  
  - `discord_notification`

- **Node Details:**  
  - **Type:** `n8n-nodes-base.discordTool`  
  - **Role:** Posts a message with embedded content to a Discord channel via bot or webhook.  
  - **Configuration:**  
    - Webhook or Bot credentials configured.  
    - Target Guild and Channel IDs set to specific Discord server and channel.  
    - Message content includes an information emoji and confirmation text generated by AI.  
    - Embed includes article URL, title, and description from AI outputs.  
  - **Input/Output:**  
    - Input: Receives AI-generated notification message and article metadata.  
    - Output: None (terminal node).  
  - **Edge Cases:**  
    - Discord API rate limits or invalid credentials.  
    - Channel or guild ID misconfiguration.  
    - Message formatting errors.  
  - **Version:** 2

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                      | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                                  |
|-------------------------|--------------------------------------------|------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger      | Trigger on incoming chat messages  | None                        | Save Article To Notion       |                                                                                                              |
| Gemini 2.5 PRO          | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI language model for context understanding | When chat message received  | Save Article To Notion       | Google Gemini AI model: picked for large context window and speed; alternative AI models can be tested.      |
| Save Article To Notion  | @n8n/n8n-nodes-langchain.agent             | AI agent orchestrating scraping, saving, notification | Gemini 2.5 PRO, website_scraper, save_to_notion, discord_notification | None                        |                                                                                                              |
| website_scraper         | @n8n/n8n-nodes-langchain.toolHttpRequest   | Scrapes webpage content via Browserless API | Save Article To Notion       | Save Article To Notion       | Browserless used as self-hosted Docker or cloud API; HTTP Request node calls Browserless API.                |
| save_to_notion          | n8n-nodes-base.notionTool                   | Saves article data to Notion database | Save Article To Notion       | Save Article To Notion       | Setup requires Notion database with specific properties; maps AI data to Notion page blocks and properties.  |
| discord_notification    | n8n-nodes-base.discordTool                  | Sends confirmation message to Discord | Save Article To Notion       | None                        | Requires Discord webhook or bot credentials; posts embed with article info and confirmation message.         |
| Sticky Note             | n8n-nodes-base.stickyNote                    | Informational note on Google Gemini AI | None                        | None                        | Google Gemini AI model: picked for large context window and speed.                                           |
| Sticky Note1            | n8n-nodes-base.stickyNote                    | Informational note on Browserless usage | None                        | None                        | Browserless info: self-hosted or cloud; HTTP Request node used for API calls.                                |
| Sticky Note2            | n8n-nodes-base.stickyNote                    | Setup and customization instructions | None                        | None                        | Detailed setup instructions for credentials, Notion DB, Browserless, Discord, and customization tips.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `Chat Trigger` node (`@n8n/n8n-nodes-langchain.chatTrigger`).  
   - Set webhook to public.  
   - No additional filters.  
   - Position it as the workflow start.

2. **Add Google Gemini AI Node:**  
   - Add `Google Gemini` node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`).  
   - Select model `models/gemini-2.5-pro-exp-03-25`.  
   - Set temperature to 0.  
   - Attach Google Gemini API credentials (from Google AI Studio or Vertex AI).  
   - Connect trigger node output to this node input.

3. **Add AI Agent Node:**  
   - Add `Agent` node (`@n8n/n8n-nodes-langchain.agent`).  
   - Configure system message with instructions to:  
     - Use `website_scraper` tool to scrape URL.  
     - Use `save_to_notion` tool to save data.  
     - Use `discord_notification` tool to notify.  
   - Enable `executeOnce` to true.  
   - Set `onError` to continue error output.  
   - Connect Gemini AI node output to this agent node input.

4. **Add Website Scraper Tool Node:**  
   - Add `HTTP Request` node configured as a LangChain tool (`@n8n/n8n-nodes-langchain.toolHttpRequest`).  
   - Set method to POST.  
   - URL: `http://browserless:3000/content` (or your Browserless API endpoint).  
   - Body (JSON):  
     ```json
     {
       "url": "{url}",
       "gotoOptions": {
         "waitUntil": "networkidle0"
       }
     }
     ```  
   - Enable sending JSON body.  
   - Define placeholder parameter `url` as string input.  
   - Name the tool `website_scraper`.  
   - Connect this node as a tool in the AI agent node.

5. **Add Notion Save Tool Node:**  
   - Add `Notion` node (`n8n-nodes-base.notionTool`).  
   - Set resource to `databasePage`.  
   - Enter your Notion database ID where articles will be saved.  
   - Map properties:  
     - Title: AI-generated article title.  
     - Description: AI-generated short description.  
     - URL: Original article URL.  
     - Tags: Multi-select tags from AI.  
     - Publication Date: Date from AI.  
   - Configure content blocks with headings and text fields populated from AI outputs (summary, objectives, concepts, technologies, code snippets, conclusions).  
   - Set icon dynamically from AI emoji.  
   - Attach Notion API credentials with access to the database.  
   - Name the node `save_to_notion`.  
   - Connect this node as a tool in the AI agent node.

6. **Add Discord Notification Node:**  
   - Add `Discord` node (`n8n-nodes-base.discordTool`).  
   - Configure with Discord Bot or Webhook credentials.  
   - Set target Guild ID and Channel ID for notifications.  
   - Configure message content to include an information emoji and confirmation text from AI.  
   - Add embed with article URL, title, and description from AI.  
   - Name the node `discord_notification`.  
   - Connect this node as a tool in the AI agent node.

7. **Connect Nodes:**  
   - Connect `When chat message received` → `Gemini 2.5 PRO` → `Save Article To Notion` agent node.  
   - Inside the agent node, tools `website_scraper`, `save_to_notion`, and `discord_notification` are referenced and called as per system message instructions.

8. **Configure Credentials:**  
   - Add Google Gemini API credentials in n8n.  
   - Add Notion API credentials with database access.  
   - Add Discord Bot or Webhook credentials.  
   - Ensure Browserless API is accessible (self-hosted or cloud) and reachable at the configured URL.

9. **Test and Activate:**  
   - Save the workflow.  
   - Test by sending a chat message containing a URL to the webhook.  
   - Verify article scraping, Notion page creation, and Discord notification.  
   - Activate the workflow for continuous operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Google Gemini AI model chosen for its large context window and processing speed. Users can experiment with other AI models supported by n8n.                                                                                             | Sticky Note on `Gemini 2.5 PRO` node.                                                                 |
| Browserless is used as a web scraping tool, either self-hosted via Docker or as a cloud service. Since no pre-built node exists, the HTTP Request node calls Browserless API directly.                                                     | Sticky Note1 near `website_scraper` node.                                                             |
| Setup instructions include creating a Notion database with specific properties (Name, URL, Description, Tags, Publication Date) and configuring credentials for Notion, Google Gemini, Discord, and Browserless.                            | Sticky Note2 covers detailed setup and customization instructions.                                    |
| Discord notifications require a webhook URL or bot token with permissions to post in the target channel.                                                                                                                                | Setup section and `discord_notification` node configuration.                                          |
| For customization, users can change the AI model, adjust Notion property mappings, or swap Browserless with other scraping APIs by modifying the HTTP Request node parameters.                                                           | Setup and customization notes in Sticky Note2.                                                        |
| Google Gemini API keys can be obtained from [Google AI Studio](https://aistudio.google.com/app/apikey) or Google Cloud Console (Vertex AI).                                                                                              | Setup instructions.                                                                                   |
| Discord webhook setup instructions available at [Discord Webhooks](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks).                                                                                           | Setup instructions.                                                                                   |
| Browserless API details and signup at [Browserless](https://www.browserless.io/).                                                                                                                                                         | Sticky Note1 and setup instructions.                                                                 |

---

This document fully describes the workflow structure, node configurations, and setup instructions to enable advanced users and AI agents to understand, reproduce, and customize the workflow effectively.