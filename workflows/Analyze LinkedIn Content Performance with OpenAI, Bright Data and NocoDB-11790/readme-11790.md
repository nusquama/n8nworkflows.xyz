Analyze LinkedIn Content Performance with OpenAI, Bright Data and NocoDB

https://n8nworkflows.xyz/workflows/analyze-linkedin-content-performance-with-openai--bright-data-and-nocodb-11790


# Analyze LinkedIn Content Performance with OpenAI, Bright Data and NocoDB

---

### 1. Workflow Overview

This workflow is designed as an **AI-powered LinkedIn Content Assistant** that automates the process of scraping LinkedIn posts, storing real engagement metrics, and enabling natural-language querying of the content performance data. It is targeted at content creators, marketers, and social media managers who want to analyze their LinkedIn posts' impact and receive AI-driven insights based on actual performance data.

The workflow consists of two main logical blocks:

- **1.1 Data Scraping & Storage Block**: This block orchestrates the extraction of LinkedIn post data via Bright Data, monitors scraping progress, downloads snapshots, and updates a structured database in NocoDB with post content and engagement metrics.

- **1.2 AI Chat Assistant Block**: This block manages incoming chat messages, maintains conversational memory, leverages OpenAI's GPT model, and utilizes stored LinkedIn profile and top post data to provide insightful responses about LinkedIn content performance.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Scraping & Storage Block

- **Overview:**  
  Automates LinkedIn posts data extraction using Bright Data scrapers, monitors snapshot status until ready, downloads the snapshot data, processes it, and updates stored records in NocoDB with fresh post content and engagement metrics.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Config (Set)  
  - Get posts (NocoDB)  
  - Edit Fields (Set)  
  - Aggregate (Aggregate)  
  - Scrape LinkedIn Posts (Bright Data)  
  - Returned Posts? (If)  
  - Get Snapshot Status (Bright Data)  
  - Is Ready? (If)  
  - Wait (Wait)  
  - Download Snapshot (Bright Data)  
  - Fetch Post Id (HTTP Request)  
  - Update Post (HTTP Request)  
  - Sticky Note  
  - Sticky Note2  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual trigger  
    - Role: Initiates the scraping workflow manually  
    - Input: None  
    - Output: Triggers Config node  
    - Potential failures: None (manual)  

  - **Config**  
    - Type: Set  
    - Role: Stores configuration data such as NocoDB base URL and posts table ID as empty strings to be filled by the user  
    - Key parameters: `nocodbBaseUrl`, `nocodbPostsTableId`  
    - Input: Trigger from manual start  
    - Output: Starts Get posts node  
    - Edge cases: Missing or incorrect config values will break downstream HTTP requests  

  - **Get posts**  
    - Type: NocoDB  
    - Role: Fetches all existing posts from NocoDB database to operate on current data  
    - Configured with project and table IDs, returns all records  
    - Input: Config node  
    - Output: Edit Fields node  
    - Failures: Credential/auth errors, invalid project/table IDs  

  - **Edit Fields**  
    - Type: Set  
    - Role: Transforms the post data, specifically setting a `url` field extracted from the JSON input's `Link` property  
    - Key expression: `={{ $json.Link }}` assigned to `url`  
    - Input: Get posts  
    - Output: Aggregate node  
    - Edge cases: Missing `Link` properties in posts can cause empty URLs  

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates all items into one data set, preparing list of URLs for scraping  
    - Operation: `aggregateAllItemData`  
    - Input: Edit Fields  
    - Output: Scrape LinkedIn Posts  
    - Failures: Empty input array leads to no URLs to scrape  

  - **Scrape LinkedIn Posts**  
    - Type: Bright Data (web scraper)  
    - Role: Uses Bright Data to scrape LinkedIn posts data in bulk based on aggregated URLs  
    - Parameters: Dataset ID for LinkedIn posts, includes errors in output  
    - Input: Aggregate  
    - Output: Returned Posts? If node  
    - Failures: API rate limits, invalid dataset ID, network errors, scraper errors  

  - **Returned Posts?**  
    - Type: If  
    - Role: Checks if the scraping returned a `snapshot_id` to proceed or monitor progress  
    - Condition: Checks non-existence of `snapshot_id` in JSON  
    - Input: Scrape LinkedIn Posts  
    - Outputs:  
      - True: Fetch Post Id node  
      - False: Get Snapshot Status node (to monitor progress)  

  - **Get Snapshot Status**  
    - Type: Bright Data  
    - Role: Polls Bright Data API for snapshot progress based on `snapshot_id`  
    - Input: Returned Posts?  
    - Output: Is Ready? node  
    - Failures: Snapshot not found, API errors  

  - **Is Ready?**  
    - Type: If  
    - Role: Checks if snapshot status equals "ready"  
    - Input: Get Snapshot Status  
    - Outputs:  
      - True: Download Snapshot node  
      - False: Wait node (delay before next poll)  

  - **Wait**  
    - Type: Wait  
    - Role: Introduces a 1-second delay before rechecking snapshot status  
    - Input: Is Ready?  
    - Output: Get Snapshot Status (loop until ready)  

  - **Download Snapshot**  
    - Type: Bright Data  
    - Role: Downloads the completed snapshot data from Bright Data for processing  
    - Input: Is Ready?  
    - Output: Fetch Post Id node  

  - **Fetch Post Id**  
    - Type: HTTP Request  
    - Role: Queries NocoDB API to fetch the record ID matching the scraped post URL  
    - URL composed dynamically from Config variables and scraped URL  
    - Input: Download Snapshot or Returned Posts?  
    - Output: Update Post node  
    - Failures: Auth errors, 404 if record not found, malformed URLs  

  - **Update Post**  
    - Type: HTTP Request  
    - Role: Updates NocoDB record with scraped post text, likes, and comments  
    - PATCH method to NocoDB API, uses record ID from Fetch Post Id  
    - Input: Fetch Post Id  
    - Output: None (end of chain)  
    - Failures: Auth errors, invalid data, network  

  - **Sticky Note & Sticky Note2**  
    - Type: Sticky Note (documentation)  
    - Role: Provides a detailed README and usage instructions within the workflow UI for users  

---

#### 2.2 AI Chat Assistant Block

- **Overview:**  
  Provides an interactive AI chatbot interface that listens for chat messages, maintains conversational memory, leverages OpenAI GPT-5.2 for responses, and uses tools to fetch LinkedIn profile and top post data to respond with insightful feedback.

- **Nodes Involved:**  
  - When chat message received (LangChain Chat Trigger)  
  - Chat memory (Memory Buffer Window)  
  - Agent memory (Memory Buffer Window)  
  - AI Agent (LangChain Agent)  
  - OpenAI Chat Model (LangChain LM Chat OpenAI)  
  - Get LinkedIn Profile (Bright Data Tool)  
  - Get Top LinkedIn Posts (NocoDB Tool)  
  - Sticky Note1  

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point for incoming chat messages, supports public webhook with authentication  
    - Parameters: Loads previous session from memory, initial greeting message  
    - Outputs: AI Agent node  
    - Failures: Webhook auth errors, message load failures  

  - **Chat memory**  
    - Type: Memory Buffer Window (LangChain)  
    - Role: Maintains buffer of last 10 messages for context in chat conversations  
    - Input: Connected as AI memory input for chat trigger  
    - Output: Back to When chat message received node (memory cycle)  
    - Edge cases: Memory overflow handled by buffer limit  

  - **Agent memory**  
    - Type: Memory Buffer Window (LangChain)  
    - Role: Stores AI agent's contextual memory window of 10 entries for reasoning continuity  
    - Input: Connected as AI memory to AI Agent  
    - Output: AI Agent node  
    - Edge cases: Same as above  

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Core AI reasoning node that processes chat messages, uses system prompt to act as LinkedIn content assistant, and integrates tools for data retrieval  
    - System message: Defines role as expert content assistant able to fetch profile and top posts info  
    - Inputs: Chat message, chat memory, agent memory, OpenAI language model, AI tools  
    - Output: Response sent back to chat trigger node  
    - Failures: OpenAI API errors, tool invocation failures, expression errors  

  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides GPT-5.2 model with web search capability as language model for AI Agent  
    - Credentials: OpenAI API key  
    - Input: AI Agent (ai_languageModel)  
    - Output: AI Agent  
    - Failures: API limits, auth errors  

  - **Get LinkedIn Profile**  
    - Type: Bright Data Tool (web scraper)  
    - Role: Tool integrated into AI Agent to fetch LinkedIn profile data for context during chat  
    - Input: AI Agent (ai_tool)  
    - Output: AI Agent  
    - Failures: API errors, invalid URLs, network  

  - **Get Top LinkedIn Posts**  
    - Type: NocoDB Tool  
    - Role: Tool used by AI Agent to fetch top performing posts from NocoDB database to assist in chat responses  
    - Input: AI Agent (ai_tool)  
    - Output: AI Agent  
    - Failures: Credential errors, empty results  

  - **Sticky Note1**  
    - Type: Sticky Note (documentation)  
    - Role: Marks and documents the AI Chatbot section  

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                         | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                   |
|---------------------------|----------------------------------|---------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Starts scraping workflow manually     | None                          | Config                      |                                                                                              |
| Config                    | Set                              | Holds configuration values            | When clicking ‘Execute workflow’ | Get posts                   |                                                                                              |
| Get posts                 | NocoDB                           | Fetches existing posts from DB        | Config                        | Edit Fields                 |                                                                                              |
| Edit Fields               | Set                              | Sets URL field from post data          | Get posts                    | Aggregate                   |                                                                                              |
| Aggregate                 | Aggregate                        | Aggregates all URLs into one set      | Edit Fields                   | Scrape LinkedIn Posts       |                                                                                              |
| Scrape LinkedIn Posts     | Bright Data                      | Scrapes LinkedIn post data             | Aggregate                    | Returned Posts?              |                                                                                              |
| Returned Posts?            | If                               | Checks if snapshot_id exists           | Scrape LinkedIn Posts         | Fetch Post Id (true), Get Snapshot Status (false) |                                                                                              |
| Get Snapshot Status       | Bright Data                      | Gets scraping snapshot progress        | Returned Posts?               | Is Ready?                   |                                                                                              |
| Is Ready?                 | If                               | Checks if snapshot is ready            | Get Snapshot Status           | Download Snapshot (true), Wait (false) |                                                                                              |
| Wait                      | Wait                             | Delay before checking snapshot again   | Is Ready?                    | Get Snapshot Status         |                                                                                              |
| Download Snapshot         | Bright Data                      | Downloads scraping snapshot data       | Is Ready?                    | Fetch Post Id               |                                                                                              |
| Fetch Post Id             | HTTP Request                    | Gets NocoDB record ID by post URL      | Download Snapshot, Returned Posts? | Update Post              |                                                                                              |
| Update Post               | HTTP Request                    | Updates post content & stats in DB     | Fetch Post Id                | None                        |                                                                                              |
| When chat message received | LangChain Chat Trigger           | Receives chat messages                  | None                         | AI Agent                   |                                                                                              |
| Chat memory               | LangChain Memory Buffer Window   | Maintains chat context                 | (ai_memory from chat trigger) | When chat message received  |                                                                                              |
| Agent memory              | LangChain Memory Buffer Window   | Maintains AI agent's memory            | (ai_memory from AI Agent)     | AI Agent                   |                                                                                              |
| AI Agent                  | LangChain Agent                  | Processes chat, reasons, uses tools    | When chat message received, Chat memory, Agent memory, OpenAI Chat Model, Get LinkedIn Profile, Get Top LinkedIn Posts | When chat message received |                                                                                              |
| OpenAI Chat Model         | LangChain LM Chat OpenAI          | GPT-5.2 language model                  | AI Agent (ai_languageModel)   | AI Agent                   |                                                                                              |
| Get LinkedIn Profile      | Bright Data Tool                 | Provides LinkedIn profile data          | AI Agent (ai_tool)            | AI Agent                   |                                                                                              |
| Get Top LinkedIn Posts    | NocoDB Tool                     | Provides top posts data                  | AI Agent (ai_tool)            | AI Agent                   |                                                                                              |
| Sticky Note               | Sticky Note                     | Documentation for scraping block       | None                         | None                       | # Get Linkedin Posts w/ Bright Data                                                         |
| Sticky Note1              | Sticky Note                     | Documentation for AI Chatbot block     | None                         | None                       | # AI Chatbot                                                                                 |
| Sticky Note2              | Sticky Note                     | README and usage instructions           | None                         | None                       | See README in node content for detailed setup and usage instructions                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Type: Manual Trigger (n8n core)  
   - No parameters needed  

2. **Create Config Node**  
   - Type: Set  
   - Assign two string variables:  
     - `nocodbBaseUrl` (initial empty string)  
     - `nocodbPostsTableId` (initial empty string)  
   - Connect "When clicking ‘Execute workflow’" → "Config"  

3. **Create Get posts Node**  
   - Type: NocoDB  
   - Operation: Get All  
   - Project ID: set to your NocoDB project ID  
   - Table ID: set to your posts table ID (link to Config variables)  
   - Authentication: NocoDB API Token credentials  
   - Connect "Config" → "Get posts"  

4. **Create Edit Fields Node**  
   - Type: Set  
   - Mode: Raw  
   - JSON output: `{"url": "{{$json.Link}}"}`  
   - Connect "Get posts" → "Edit Fields"  

5. **Create Aggregate Node**  
   - Type: Aggregate  
   - Operation: Aggregate All Item Data  
   - Connect "Edit Fields" → "Aggregate"  

6. **Create Scrape LinkedIn Posts Node**  
   - Type: Bright Data (web scraper)  
   - Resource: Web Scraper  
   - Operation: Use dataset ID for LinkedIn posts scraping  
   - URLs: `={{$json.data.toJsonString()}}` (aggregate output)  
   - Authentication: Bright Data API credentials  
   - Connect "Aggregate" → "Scrape LinkedIn Posts"  

7. **Create Returned Posts? If Node**  
   - Type: If  
   - Condition: Check if `snapshot_id` does NOT exist in JSON  
   - Connect "Scrape LinkedIn Posts" → "Returned Posts?"  

8. **Create Fetch Post Id Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{$('Config').item.json.nocodbBaseUrl}}/api/v2/tables/{{$('Config').item.json.nocodbPostsTableId}}/records?where=(Link,eq,{{$json.input.url}})`  
   - Authentication: NocoDB API Token credentials  
   - Connect "Returned Posts?" (true) → "Fetch Post Id"  
   - Connect "Download Snapshot" (step below) → "Fetch Post Id"  

9. **Create Update Post Node**  
   - Type: HTTP Request  
   - Method: PATCH  
   - URL: `={{$('Config').item.json.nocodbBaseUrl}}/api/v2/tables/{{$('Config').item.json.nocodbPostsTableId}}/records`  
   - Body Parameters:  
     - Id: `={{$json.list[0].Id}}`  
     - Content: `={{$('Scrape LinkedIn Posts').item.json.post_text}}`  
     - Likes: `={{$('Scrape LinkedIn Posts').item.json.num_likes}}`  
     - Comments: `={{$('Scrape LinkedIn Posts').item.json.num_comments}}`  
   - Authentication: NocoDB API Token credentials  
   - Connect "Fetch Post Id" → "Update Post"  

10. **Create Get Snapshot Status Node**  
    - Type: Bright Data  
    - Operation: Monitor Progress Snapshot  
    - Parameter: `snapshot_id` from previous node JSON  
    - Authentication: Bright Data API credentials  
    - Connect "Returned Posts?" (false) → "Get Snapshot Status"  

11. **Create Is Ready? If Node**  
    - Type: If  
    - Condition: Check if `status` equals "ready"  
    - Connect "Get Snapshot Status" → "Is Ready?"  

12. **Create Download Snapshot Node**  
    - Type: Bright Data  
    - Operation: Download Snapshot  
    - Parameter: `snapshot_id` from previous node JSON  
    - Authentication: Bright Data API credentials  
    - Connect "Is Ready?" (true) → "Download Snapshot"  

13. **Create Wait Node**  
    - Type: Wait  
    - Duration: 1 second  
    - Connect "Is Ready?" (false) → "Wait"  
    - Connect "Wait" → "Get Snapshot Status" (loop)  

14. **Create Chat Trigger Node**  
    - Name: "When chat message received"  
    - Type: LangChain Chat Trigger  
    - Parameters:  
      - Public webhook with authentication  
      - Load previous session memory  
      - Initial message: "Hey, I'm your AI content assistant. What can I help you with?"  

15. **Create Chat memory Node**  
    - Type: LangChain Memory Buffer Window  
    - Context window length: 10  
    - Connect `ai_memory` output of Chat Trigger → Chat memory input  

16. **Create Agent memory Node**  
    - Type: LangChain Memory Buffer Window  
    - Context window length: 10  
    - Connect `ai_memory` output of AI Agent → Agent memory input  

17. **Create OpenAI Chat Model Node**  
    - Type: LangChain LM Chat OpenAI  
    - Model: GPT-5.2  
    - Enable built-in web search tool  
    - Credentials: OpenAI API key  

18. **Create AI Agent Node**  
    - Type: LangChain Agent  
    - System message: Defines AI as LinkedIn content assistant with access to tools  
    - Connect:  
      - `main` input from Chat Trigger  
      - `ai_memory` input from Agent memory  
      - `ai_languageModel` input from OpenAI Chat Model  
      - `ai_tool` inputs from Get LinkedIn Profile and Get Top LinkedIn Posts nodes  
    - Output: Sends response back to Chat Trigger node  

19. **Create Get LinkedIn Profile Node**  
    - Type: Bright Data Tool  
    - Resource: Web Scraper  
    - Dataset ID: LinkedIn profile scraping dataset  
    - URLs: Your LinkedIn profile URL (replace placeholder)  
    - Credentials: Bright Data API  

20. **Create Get Top LinkedIn Posts Node**  
    - Type: NocoDB Tool  
    - Operation: Get All  
    - Project and Table IDs set to posts collection  
    - Credentials: NocoDB API Token  

21. **Add Sticky Notes**  
    - Add documentation sticky notes as per workflow for clear segment identification and user guidance  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                | Context or Link                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow uses **Bright Data** for LinkedIn data scraping, **NocoDB** for storing scraped post data with engagement metrics, and **OpenAI GPT-5.2** for AI-driven analysis and interactive chat insights.                                                                                                  | Workflow description                                         |
| Detailed README sticky note includes setup instructions, credential linking, and usage guidance for users new to the workflow.                                                                                                                                                                              | README sticky note node content                              |
| Requires proper credentials setup for: Bright Data API, NocoDB API token, and OpenAI API Key for full functionality.                                                                                                                                                                                       | Workflow credential requirements                             |
| AI Agent system prompt is tailored for LinkedIn content assistant role, enabling honest feedback and data-driven recommendations.                                                                                                                                                                          | AI Agent node system message                                |
| For support, customization, or advanced AI workflow development, users are encouraged to join the **Tech Builders Club** Discord community: [https://link.kornel.me/discord-tbc](https://link.kornel.me/discord-tbc)                                                                                          | README sticky note link                                      |
| Bright Data dataset IDs for LinkedIn profiles and posts scraping must be replaced with user-specific dataset IDs from their Bright Data account before activation.                                                                                                                                         | Config and scraper nodes                                      |
| NocoDB project and table IDs must be accurately configured to match user database schema to ensure data synchronization and updates work correctly.                                                                                                                                                        | Config node and NocoDB nodes                                 |
| The workflow respects n8n’s memory buffer window limits (10 messages) for chat context to optimize performance and avoid excessive memory consumption.                                                                                                                                                      | Chat memory and Agent memory node parameters                 |
| Polling snapshot status with 1-second wait loops ensures scraping completion before data download, handling asynchronous scraping delays gracefully.                                                                                                                                                       | Wait node configuration                                      |

---

**Disclaimer:** The provided content is an automated workflow built with n8n, adhering strictly to content policies and handling only legal and public data.

---