Generate Personalized Sales Outreach with GPT-4, Explorium MCP & Telegram

https://n8nworkflows.xyz/workflows/generate-personalized-sales-outreach-with-gpt-4--explorium-mcp---telegram-5421


# Generate Personalized Sales Outreach with GPT-4, Explorium MCP & Telegram

### 1. Workflow Overview

This workflow automates personalized sales outreach by enriching lead data using AI and multiple data sources, then generating a structured email sequence for targeted cold outreach. It integrates Telegram as the user input/output interface, Explorium MCP and Tavily for B2B enrichment and web intelligence, and GPT-4 for advanced reasoning and personalized message generation.

**Logical Blocks:**

- **1.1 Input Reception:** Receives lead data via Telegram messages from users.
- **1.2 AI Lead Enrichment Agent:** Central AI agent orchestrating multi-source data enrichment, memory management, and email sequence generation.
- **1.3 Data Enrichment Tools:** External data retrieval using Explorium MCP (B2B database) and Tavily (web intelligence).
- **1.4 Memory Persistence:** Stores conversation context and session data in PostgreSQL for continuity.
- **1.5 Output Delivery:** Sends the enriched lead profile and email sequence back to the user on Telegram.
- **1.6 Documentation and Setup Guidance:** Sticky notes providing usage overview and configuration instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures incoming Telegram messages containing lead information to initiate enrichment.
- **Nodes Involved:** Telegram Input
- **Node Details:**
  - **Telegram Input**
    - Type: Telegram Trigger  
    - Role: Listens for new Telegram messages to start the workflow.
    - Configuration: Listens for "message" updates; webhook auto-generated upon activation.
    - Input: User messages with company and contact info.
    - Output: Message JSON forwarded to AI Lead Enrichment Agent.
    - Edge Cases: Invalid or incomplete messages, Telegram API downtime, webhook misconfiguration.
    - Version: 1.2

#### 2.2 AI Lead Enrichment Agent

- **Overview:** Core AI node driving the lead data processing by interpreting input text, coordinating calls to external data tools, managing conversational memory, and generating a detailed, structured JSON output with personalized email outreach sequences.
- **Nodes Involved:** AI Lead Enrichment Agent
- **Node Details:**
  - Type: LangChain Agent (AI orchestrator)
  - Role: Executes prompt logic, invokes AI language model and external tools, synthesizes data.
  - Configuration:
    - Input text bound dynamically from Telegram message text.
    - System prompt defines multi-step enrichment: data collection, research strategy, deep intelligence gathering, email sequence creation, and output formatting as JSON.
    - Uses Explorium MCP and Tavily tools as data sources.
    - Calls OpenAI GPT-4 for natural language processing with temperature 0.3.
    - Manages conversation memory via PostgreSQL.
  - Input: Text from Telegram Input node.
  - Output: JSON with enriched lead profile and email sequence.
  - Edge Cases:
    - AI prompt failure or misinterpretation.
    - Tool invocation failures (auth errors, rate limits).
    - JSON formatting errors.
    - Missing or incomplete user input.
  - Version: 2

#### 2.3 Data Enrichment Tools

- **Overview:** Provide external data sources to the AI agent for comprehensive lead enrichment.
- **Nodes Involved:** Explorium B2B Data, Tavily Web Intelligence
- **Node Details:**
  
  - **Explorium B2B Data**
    - Type: MCP Client Tool (API client)
    - Role: Fetches firmographic, contact, and technology stack data from Explorium’s B2B database.
    - Configuration: SSE endpoint with header authorization.
    - Input: Queries from AI Lead Enrichment Agent.
    - Output: Structured data streamed back to the AI agent.
    - Edge Cases: Authentication failure, endpoint unavailability, data completeness issues.
    - Version: 1
  
  - **Tavily Web Intelligence**
    - Type: HTTP Request
    - Role: Executes targeted web searches for recent news, social presence, and competitor analysis.
    - Configuration:
      - POST to Tavily API with JSON body dynamically built from input query.
      - Includes filters for domains and search depth.
      - Authorization header with API key.
    - Input: Search queries from AI Lead Enrichment Agent.
    - Output: Search results including answers and raw content.
    - Edge Cases: API key invalid, request throttling, network errors.
    - Version: 4.2

#### 2.4 Memory Persistence

- **Overview:** Maintains session state and conversation memory to provide context-aware AI responses and enrichments.
- **Nodes Involved:** Conversation Memory
- **Node Details:**
  - Type: LangChain PostgreSQL Memory Node
  - Role: Stores chat session data keyed by Telegram user ID concatenated with 'lead_enrichment' suffix.
  - Configuration: Uses a custom session key derived from Telegram user ID.
  - Input: Conversation data from AI Lead Enrichment Agent.
  - Output: Provides historical context for subsequent AI requests.
  - Edge Cases: Database connection failure, schema issues.
  - Version: 1.3

#### 2.5 Output Delivery

- **Overview:** Sends the final enriched lead information and email sequence back to the originating Telegram chat.
- **Nodes Involved:** Telegram Response
- **Node Details:**
  - Type: Telegram Node (send message)
  - Role: Posts AI-generated structured JSON output as a Markdown-formatted message.
  - Configuration:
    - Target chat ID dynamically extracted from the incoming Telegram message.
    - Message text bound to AI Lead Enrichment Agent output field named "output".
    - Markdown formatting enabled, no attribution appended.
  - Input: AI-generated JSON from AI Lead Enrichment Agent.
  - Output: User-visible Telegram message.
  - Edge Cases: Chat ID missing, Telegram API rate limits, message formatting errors.
  - Version: 1.2

#### 2.6 Documentation and Setup Guidance

- **Overview:** Provides inline documentation and setup instructions for users and maintainers.
- **Nodes Involved:** 🚀 Workflow Overview, ⚙️ Setup Guide, Research Tools Info, AI Engine Info
- **Node Details:**
  - Type: Sticky Note
  - Role: Descriptive notes explaining workflow purpose, setup steps, tool descriptions, and AI capabilities.
  - Content Highlights:
    - Workflow purpose and ideal use cases.
    - Required API keys and credentials.
    - Configuration and testing instructions.
    - Descriptions of Explorium MCP and Tavily tools.
    - GPT-4 model parameters and AI capabilities.
  - Edge Cases: Outdated instructions if workflow is modified without updating notes.

---

### 3. Summary Table

| Node Name             | Node Type                            | Functional Role                      | Input Node(s)          | Output Node(s)      | Sticky Note                                                                                              |
|-----------------------|------------------------------------|------------------------------------|-----------------------|---------------------|--------------------------------------------------------------------------------------------------------|
| Telegram Input        | Telegram Trigger                   | Receives lead info from Telegram   | —                     | AI Lead Enrichment Agent | See “🚀 Workflow Overview” and “⚙️ Setup Guide” for usage and configuration details                     |
| AI Lead Enrichment Agent | LangChain Agent                   | Core AI processing and orchestration | Telegram Input, Explorium B2B Data, Tavily Web Intelligence, Conversation Memory, OpenAI GPT-4 | Telegram Response    | See “AI Engine Info” and “Research Tools Info” for AI capabilities and tool descriptions                |
| OpenAI GPT-4          | LangChain LM Chat OpenAI           | GPT-4 model for advanced reasoning | AI Lead Enrichment Agent | AI Lead Enrichment Agent | See “AI Engine Info” for model details                                                                  |
| Explorium B2B Data    | MCP Client Tool                   | B2B data enrichment tool            | AI Lead Enrichment Agent | AI Lead Enrichment Agent | See “Research Tools Info” for Explorium MCP description                                                |
| Tavily Web Intelligence | HTTP Request Tool                 | Web intelligence for recent news and competitors | AI Lead Enrichment Agent | AI Lead Enrichment Agent | See “Research Tools Info” for Tavily API details                                                      |
| Conversation Memory   | LangChain PostgreSQL Memory        | Maintains session state and memory | AI Lead Enrichment Agent | AI Lead Enrichment Agent | See “AI Engine Info” for memory functionality                                                         |
| Telegram Response     | Telegram Node                      | Sends enriched data back to Telegram chat | AI Lead Enrichment Agent | —                   | See “🚀 Workflow Overview” and “⚙️ Setup Guide”                                                        |
| 🚀 Workflow Overview  | Sticky Note                       | Workflow purpose and high-level summary | —                     | —                   | Detailed overview of workflow purpose, use cases, and input format                                    |
| ⚙️ Setup Guide        | Sticky Note                       | Configuration & setup instructions | —                     | —                   | Step-by-step setup guide for API keys, DB, Telegram Bot, testing, and customization                    |
| Research Tools Info   | Sticky Note                       | Describes data enrichment tools     | —                     | —                   | Explains Explorium MCP and Tavily tool capabilities                                                   |
| AI Engine Info        | Sticky Note                       | Details AI model and memory setup   | —                     | —                   | Describes GPT-4 model parameters and memory storage                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Input Node**
   - Type: Telegram Trigger
   - Configure webhook to listen for "message" updates.
   - No additional parameters.

2. **Create AI Lead Enrichment Agent Node**
   - Type: LangChain Agent
   - Set input text parameter to `={{ $json.message.text }}`
   - Define system prompt with detailed multi-step enrichment and output JSON format (as per systemMessage in the original config).
   - Configure tools:
     - Explorium MCP (API client)
     - Tavily web search (HTTP request)
   - Configure AI language model to use OpenAI GPT-4 (linked in next step).
   - Configure conversation memory (linked to PostgreSQL node).
   - Connect Telegram Input as main input.
   - Output to Telegram Response node.

3. **Create OpenAI GPT-4 Node**
   - Type: LangChain LM Chat OpenAI
   - Model: `gpt-4o`
   - Max tokens: 4000
   - Temperature: 0.3
   - Connect as AI language model for AI Lead Enrichment Agent node.

4. **Create Explorium B2B Data Node**
   - Type: MCP Client Tool
   - SSE endpoint: `mcp.explorium.ai/sse`
   - Authentication: Header with API key (configured in credentials)
   - Connect as AI tool for AI Lead Enrichment Agent.

5. **Create Tavily Web Intelligence Node**
   - Type: HTTP Request
   - URL: `https://api.tavily.com/search`
   - Method: POST
   - Headers: Authorization with Bearer token (Tavily API key), Content-Type: application/json
   - Body (JSON):
     ```json
     {
       "query": "{{ $parameter.query }}",
       "auto_parameters": true,
       "topic": "general",
       "search_depth": "advanced",
       "chunks_per_source": 5,
       "max_results": 3,
       "include_answer": true,
       "include_raw_content": true,
       "include_images": false,
       "include_domains": ["linkedin.com", "crunchbase.com", "bloomberg.com"],
       "exclude_domains": ["pinterest.com", "instagram.com"]
     }
     ```
   - Connect as AI tool for AI Lead Enrichment Agent.

6. **Create Conversation Memory Node**
   - Type: LangChain PostgreSQL Memory
   - Session Key: `={{ $json.message.from.id + '_lead_enrichment' }}`
   - Configure PostgreSQL credentials and connection.
   - Connect as AI memory for AI Lead Enrichment Agent.

7. **Create Telegram Response Node**
   - Type: Telegram
   - Text: `={{ $json.output }}`
   - Chat ID: `={{ $item("0").$node["Telegram Input"].json["message"]["chat"]["id"] }}`
   - Parse mode: Markdown
   - Connect main output from AI Lead Enrichment Agent.

8. **Add Sticky Notes for Documentation**
   - Create notes for:
     - Workflow Overview (purpose, use cases)
     - Setup Guide (API keys, DB, Telegram bot)
     - Research Tools Info (Explorium MCP, Tavily)
     - AI Engine Info (GPT-4 model, memory)

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow uses GPT-4 via LangChain with PostgreSQL memory for persistent, context-aware AI lead enrichment.       | AI Engine Info sticky note                                                                             |
| Explorium MCP offers comprehensive B2B firmographic and technology data for lead enrichment.                         | Research Tools Info sticky note                                                                        |
| Tavily API is leveraged for real-time web intelligence including news and competitive analysis.                      | Research Tools Info sticky note                                                                        |
| Telegram bot integration enables seamless user input and output for sales outreach automation.                       | ⚙️ Setup Guide sticky note                                                                              |
| Compose your Telegram bot via @BotFather and provide the token in n8n credentials.                                   | ⚙️ Setup Guide sticky note                                                                              |
| PostgreSQL database is required for conversation memory persistence; tables auto-create on first run.                 | ⚙️ Setup Guide sticky note                                                                              |
| Use professional yet conversational tone in AI prompts to ensure client-appropriate outreach content.                | Included in AI Lead Enrichment Agent prompt system message                                            |
| Output JSON structure includes detailed lead profile, pain points, automation opportunities, and a 4-email sequence. | Defined explicitly in the AI Lead Enrichment Agent’s system prompt                                     |

---

**Disclaimer:** The text provided is exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or offensive material. All manipulated data is public and legal.