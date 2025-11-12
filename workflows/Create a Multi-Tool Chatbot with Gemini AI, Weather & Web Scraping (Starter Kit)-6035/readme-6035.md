Create a Multi-Tool Chatbot with Gemini AI, Weather & Web Scraping (Starter Kit)

https://n8nworkflows.xyz/workflows/create-a-multi-tool-chatbot-with-gemini-ai--weather---web-scraping--starter-kit--6035


# Create a Multi-Tool Chatbot with Gemini AI, Weather & Web Scraping (Starter Kit)

### 1. Workflow Overview

This workflow, titled **"Create a Multi-Tool Chatbot with Gemini AI, Weather & Web Scraping (Starter Kit)"**, is designed to showcase an AI-powered chatbot agent built with n8n. The chatbot intelligently interacts with users via a public chat interface, processes their requests using a Large Language Model (LLM) as its brain, and dynamically invokes various integrated tools to fetch relevant data or perform actions.

**Target Use Cases:**  
- Conversational AI assistant demonstrating multi-tool integration  
- Fetching weather forecasts based on user queries  
- Retrieving latest news from RSS feeds  
- Serving as a demo and template for building AI agents with n8n  

**Logical Blocks:**  
- **1.1 Chat Interface:** Public-facing UI node enabling user interaction  
- **1.2 AI Brain (Agent & LLM):** Core logic interpreting user inputs and selecting tools  
- **1.3 Memory:** Short-term conversation memory for context retention  
- **1.4 Tools (Agent’s Toolbox):** External capabilities like weather data and news feed retrieval  
- **1.5 Supporting Nodes & Documentation:** Sticky notes providing instructions, credentials info, and optional tools  

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Interface

- **Overview:**  
  This block provides the conversational frontend where users send messages to the AI agent. It is a public chat window with customizable appearance.

- **Nodes Involved:**  
  - Example Chat Window

- **Node Details:**  
  - **Example Chat Window**  
    - Type: Langchain Chat Trigger (chat UI node)  
    - Role: Public chat interface; receives user messages and sends back AI responses  
    - Configuration:  
      - Public access enabled  
      - Custom CSS applied for glassmorphism theme and branding (n8n red primary color)  
      - Title set to "Your first AI Agent 🚀" with a friendly subtitle  
      - Placeholder text: "Type your message here.."  
      - Shows the last node's response only  
      - Initial greeting message: "Hi there! 👋"  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Your First AI Agent" node main input  
    - Edge Cases:  
      - Network connectivity issues could block message delivery  
      - Styling customization errors if CSS is malformed  
      - Webhook URL must be accessible publicly  
    - Version: 1.1  

---

#### 2.2 AI Brain (Agent & LLM)

- **Overview:**  
  This is the core intelligence block. It receives user messages from the chat, interprets intent using a Large Language Model, decides which tool to invoke, and returns a conversational response.

- **Nodes Involved:**  
  - Your First AI Agent  
  - Gemini (Google’s Gemini LLM node)  
  - OpenAI (disabled alternative LLM node)  

- **Node Details:**  
  - **Your First AI Agent**  
    - Type: Langchain Agent node  
    - Role: Central orchestrator AI agent that uses tools and memory to fulfill user requests  
    - Configuration:  
      - Custom system message defining personality, behavior, goals, and instructions  
      - Uses tools connected via its `Tool` input  
      - Uses memory connected via its `Memory` input  
      - Conversational, friendly, educational tone  
      - Error handling instructions included to give feedback link if tools fail  
    - Inputs:  
      - From chat trigger node (user messages)  
      - From Gemini node (LLM responses) via `ai_languageModel` input  
      - From tools nodes via `ai_tool` input  
      - From memory node via `ai_memory` input  
    - Outputs:  
      - Response to chat window (main output)  
    - Edge Cases:  
      - If no tools match user intent, agent might not perform correctly  
      - Misconfiguration of system message can affect behavior  
      - Tool node failures propagate here  
    - Version: 2.2  

  - **Gemini**  
    - Type: Langchain Google Gemini LLM node  
    - Role: Provides the language understanding and generation capabilities for the agent  
    - Configuration:  
      - Temperature set to 0 for deterministic output  
      - Uses Google Palm API credentials (stored securely)  
    - Inputs: From "Your First AI Agent" as LLM model  
    - Outputs: To "Your First AI Agent" for generating responses  
    - Edge Cases:  
      - API key missing or invalid causes auth failures  
      - API rate limits or downtime can cause timeouts  
    - Version: 1  

  - **OpenAI** (disabled)  
    - Type: Langchain OpenAI LLM node  
    - Role: Alternative LLM to Gemini, disabled by default  
    - Configuration:  
      - Model: GPT-4.1-mini  
      - Temperature: 0  
      - Requires OpenAI API key credentials  
    - Edge Cases: Same as Gemini for auth and API issues  
    - Version: 1.2  

---

#### 2.3 Memory

- **Overview:**  
  Enables the agent to retain conversational context by keeping a buffer of recent messages, allowing for a more natural and coherent interaction.

- **Nodes Involved:**  
  - Simple Memory  

- **Node Details:**  
  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window node  
    - Role: Stores last 30 messages of conversation for the agent to reference  
    - Configuration:  
      - Context window length set to 30 messages  
    - Inputs: Connected to "Your First AI Agent" `ai_memory` input  
    - Outputs: Feeds memory data back to "Your First AI Agent"  
    - Edge Cases:  
      - Excessive context length could impact performance or cost  
      - Loss of context if node is disabled or misconfigured  
    - Version: 1.3  

---

#### 2.4 Tools (Agent’s Toolbox)

- **Overview:**  
  These nodes provide the external capabilities (called "tools") that the AI agent can invoke to answer user queries or perform actions, such as fetching weather or news.

- **Nodes Involved:**  
  - Get Weather (HTTP Request Tool)  
  - Get News (RSS Feed Read Tool)  
  - Get Upcoming Events (Google Calendar Tool) [disabled]  
  - Send Email (Gmail Tool) [disabled]  

- **Node Details:**  
  - **Get Weather**  
    - Type: HTTP Request Tool node  
    - Role: Fetches weather forecast data from Open-Meteo API  
    - Configuration:  
      - URL: https://api.open-meteo.com/v1/forecast  
      - Query parameters dynamically set from AI-generated expressions to infer latitude, longitude, weather variables, dates, and units from user input  
      - Supports current, hourly, and daily weather data by selecting variables dynamically  
    - Inputs: Connected to "Your First AI Agent" `ai_tool` input  
    - Outputs: Responses returned to the agent for user presentation  
    - Edge Cases:  
      - Invalid location or date parameters may yield empty or error responses  
      - API downtime or network errors  
      - Expression evaluation errors if AI provides malformed parameters  
    - Version: 4.2  

  - **Get News**  
    - Type: RSS Feed Read Tool node  
    - Role: Retrieves the latest articles from a dynamic RSS feed URL  
    - Configuration:  
      - URL parameter dynamically set by AI with multiple news source options provided as hints  
      - Returns all entries from the feed  
    - Inputs: Connected to "Your First AI Agent" `ai_tool` input  
    - Outputs: Feed items sent back for user display  
    - Edge Cases:  
      - Invalid or unreachable RSS URLs  
      - Empty or malformed feeds  
    - Version: 1.2  

  - **Get Upcoming Events** (disabled)  
    - Type: Google Calendar Tool node  
    - Role: Would fetch calendar events for the next 7 days  
    - Configuration:  
      - Requires Google Calendar credentials  
      - Disabled by default to prevent errors without setup  
    - Edge Cases: Auth errors if credentials missing  
    - Version: 1.3  

  - **Send Email** (disabled)  
    - Type: Gmail Tool node  
    - Role: Would send an email based on user input  
    - Configuration:  
      - Email recipient, subject, and message dynamically set from AI expressions  
      - Requires Gmail OAuth2 credentials  
      - Disabled by default  
    - Edge Cases: Auth failures, invalid email addresses, quota limits  
    - Version: 2.1  

---

#### 2.5 Supporting Nodes & Documentation

- **Overview:**  
  This block contains explanatory sticky notes that guide users on how to operate, customize, and extend the workflow, including credential setup and optional tool integration.

- **Nodes Involved:**  
  - Multiple Sticky Notes (e.g., Sticky Note, Sticky Note10, Sticky Note12-18, Bonus Tools Note, Introduction Note)

- **Node Details:**  
  - They provide:  
    - Step-by-step user introduction and workflow explanation  
    - Instructions for obtaining Google Gemini and OpenAI API keys  
    - Guidance on activating the workflow and accessing the chat URL  
    - Optional tools description and how to add them  
    - Contact and feedback links with coaching and consulting offers  
    - Branding, styling, and video tutorial placeholders  
  - These notes do not affect workflow execution but are essential for user onboarding and customization  
  - Edge Cases: None (informational only)  
  - Version: 1  

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                         | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                                |
|---------------------|-----------------------------------|---------------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Example Chat Window  | Langchain Chat Trigger             | Public chat UI for user interaction   | None                  | Your First AI Agent      | ## 💬 1. The Chat Interface - How to test and customize the chat window                                                      |
| Your First AI Agent  | Langchain Agent                   | Central AI brain and orchestrator     | Example Chat Window, Gemini, Get Weather, Get News, Simple Memory | Example Chat Window       | ## 🧠 2. The Brain: Your AI Agent - Explains the agent role and system message                                              |
| Gemini              | Langchain Google Gemini LLM       | Provides language understanding       | Your First AI Agent (ai_languageModel) | Your First AI Agent       | ## 🤖 3. The AI Brainpower (LLM) - How to add credentials and select model                                                   |
| OpenAI (disabled)    | Langchain OpenAI LLM              | Alternative LLM (disabled)             | Your First AI Agent (ai_languageModel) | Your First AI Agent       | ## 🤖 3. The AI Brainpower (LLM) - Alternative model instructions                                                           |
| Simple Memory        | Langchain Memory Buffer Window    | Maintains short-term conversation context | Your First AI Agent (ai_memory) | Your First AI Agent       | ## 🗂️ 4. Short-Term Memory - Importance for conversational context                                                         |
| Get Weather          | HTTP Request Tool                 | Fetches weather data from API         | Your First AI Agent (ai_tool) | Your First AI Agent       | ## 🛠️ 5. The Agent's Toolbox (Superpowers) - How tools empower the agent                                                    |
| Get News             | RSS Feed Read Tool                | Retrieves latest news from RSS feeds  | Your First AI Agent (ai_tool) | Your First AI Agent       | ## 🛠️ 5. The Agent's Toolbox (Superpowers) - Tool examples                                                                  |
| Get Upcoming Events (disabled) | Google Calendar Tool           | Fetches upcoming Google Calendar events | Your First AI Agent (ai_tool) | Your First AI Agent       | ### **Get Upcoming Events (Google Calendar)** - How to enable and configure the tool                                        |
| Send Email (disabled) | Gmail Tool                      | Sends emails on user behalf            | Your First AI Agent (ai_tool) | Your First AI Agent       | ### **Send an Email (Gmail)** - How to enable and configure the tool                                                        |
| Sticky Note          | Sticky Note                      | User guidance and explanations         | None                  | None                    | # 🤔 Feeling Lost? Your Step-by-Step Guide - Workflow explanation and usage instructions                                      |
| Sticky Note10        | Sticky Note                      | Creator credit and feedback solicitation | None                  | None                    | © 2025 Lucas Peyrin - Feedback and coaching offers with links                                                               |
| Sticky Note12        | Sticky Note                      | Chat interface explanation             | None                  | None                    | ## 💬 1. The Chat Interface - How to test and customize                                                                      |
| Sticky Note13        | Sticky Note                      | AI Agent overview                      | None                  | None                    | ## 🧠 2. The Brain: Your AI Agent - Central node explanation                                                                 |
| Sticky Note14        | Sticky Note                      | LLM setup instructions                 | None                  | None                    | ## 🤖 3. The AI Brainpower (LLM) - Credential setup instructions                                                             |
| Sticky Note15        | Sticky Note                      | Memory explanation                     | None                  | None                    | ## 🗂️ 4. Short-Term Memory - Role of memory                                                                                   |
| Sticky Note16        | Sticky Note                      | Agent toolbox description              | None                  | None                    | ## 🛠️ 5. The Agent's Toolbox (Superpowers) - Tool usage                                                                      |
| Sticky Note17        | Sticky Note                      | Google Gemini credential instructions  | None                  | None                    | ### 🔑 How to Get Google Gemini Credentials - Stepwise guide with link                                                       |
| Sticky Note18        | Sticky Note                      | OpenAI credential instructions         | None                  | None                    | ### 🔑 How to Get OpenAI Credentials - Stepwise guide with link                                                              |
| Bonus Tools Note     | Sticky Note                      | Encouragement to add more tools        | None                  | None                    | ## 🚀 Bonus Tools: Add More Superpowers! - Ideas for expanding the agent's capabilities                                       |
| Sticky Note1         | Sticky Note                      | Get Upcoming Events tool instructions  | None                  | None                    | ### **Get Upcoming Events (Google Calendar)** - How to add and configure                                                     |
| Sticky Note2         | Sticky Note                      | Send Email tool instructions           | None                  | None                    | ### **Send an Email (Gmail)** - How to add and configure                                                                     |
| Sticky Note7         | Sticky Note                      | Video tutorial placeholder             | None                  | None                    | ## Video Tutorial - Coming soon                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Interface Node:**  
   - Add a **Langchain Chat Trigger** node named "Example Chat Window".  
   - Set it to public with a welcoming title "Your first AI Agent 🚀" and subtitle.  
   - Customize the CSS for glassmorphism effect using the provided CSS snippet or your own styling.  
   - Set the initial message to "Hi there! 👋".  
   - Save and note the webhook URL for user access.

2. **Create the AI Agent Node:**  
   - Add a **Langchain Agent** node named "Your First AI Agent".  
   - Configure the system message with detailed personality, instructions, goals, and error handling as per the provided example.  
   - Ensure the node is set to conversational mode and outputs the last response only.

3. **Add the Large Language Model Node:**  
   - Add a **Langchain Google Gemini LLM** node named "Gemini".  
   - Set temperature to 0 for deterministic replies.  
   - Create and assign Google Palm API credentials with your API key (see Sticky Note17 instructions).  
   - Connect "Gemini" node's output to the "Your First AI Agent" node's `ai_languageModel` input.

4. **Add the Short-Term Memory Node:**  
   - Add a **Langchain Memory Buffer Window** node named "Simple Memory".  
   - Set context window length to 30 messages.  
   - Connect "Simple Memory" node to "Your First AI Agent" node's `ai_memory` input.

5. **Add Tools Nodes:**  
   - **Get Weather:** Add an **HTTP Request Tool** node named "Get Weather".  
     - URL: https://api.open-meteo.com/v1/forecast  
     - Configure query parameters with expressions to dynamically extract latitude, longitude, weather variables, and date range from AI inputs.  
     - Connect its output to "Your First AI Agent" node's `ai_tool` input.

   - **Get News:** Add an **RSS Feed Read Tool** node named "Get News".  
     - URL is dynamically set from AI input with multiple news feed options.  
     - Connect its output to "Your First AI Agent" node's `ai_tool` input.

   - Optional:  
     - Add **Google Calendar Tool** ("Get Upcoming Events") and **Gmail Tool** ("Send Email") nodes if desired, set up credentials and connect to `ai_tool` input.

6. **Connect Chat Interface to Agent:**  
   - Connect "Example Chat Window" main output to "Your First AI Agent" main input.

7. **Final Connections:**  
   - Ensure "Gemini" node output connects to "Your First AI Agent" `ai_languageModel` input.  
   - Ensure "Simple Memory" output connects to `ai_memory` input of the agent.  
   - Ensure tools nodes outputs connect to `ai_tool` input of the agent.

8. **Add Sticky Notes for Documentation:**  
   - Add multiple sticky notes with instructional content covering:  
     - Introductory guidance  
     - Credential setup for Gemini and OpenAI  
     - Explanation of chat interface and agent role  
     - Tool descriptions and how to add more  
     - Contact, feedback, and coaching links  

9. **Activate the Workflow:**  
   - Turn the workflow status to active.  
   - Open the chat URL from the "Example Chat Window" node panel.  
   - Test by typing queries related to weather, news, or general questions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| This workflow is created and maintained by Lucas Peyrin © 2025. Feedback is encouraged via the form: [Give Feedback about this Template](https://api.ia2s.app/form/templates/feedback?template=AI%20Agent). Coaching and consulting offers are available through: [Book a Coaching Session](https://api.ia2s.app/form/templates/coaching?template=AI%20Agent) and [Inquire About Consulting Services](https://api.ia2s.app/form/templates/consulting?template=AI%20Agent). Additional templates: [More n8n Templates](https://n8n.io/creators/lucaspeyrin). | Workflow author and support resources                                                                                     |
| For Google Gemini API key creation: Navigate to https://aistudio.google.com/app/apikey to create and obtain a free API key required for the Gemini node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Credential setup instructions for Gemini AI                                                                               |
| For OpenAI API key creation: Go to https://platform.openai.com/api-keys to create your OpenAI secret key if you want to use OpenAI models instead of Gemini.                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Credential setup instructions for OpenAI                                                                                   |
| Video tutorial for this template is forthcoming.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Placeholder for future video tutorial                                                                                      |
| Recommended best practice: Keep the agent’s toolset focused (under 10-15 tools) for reliability and maintainability. For complex tasks, structured step-by-step workflows are preferable over pure agent reasoning.                                                                                                                                                                                                                                                                                                                                                                                                            | Design guidance for AI agents                                                                                              |
| Styling uses a glassmorphism theme with n8n brand colors for a modern and accessible chat UI experience. CSS customization can be adjusted in the chat trigger node’s options.                                                                                                                                                                                                                                                                                                                                                                                                                                                  | UI branding and styling details                                                                                            |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.