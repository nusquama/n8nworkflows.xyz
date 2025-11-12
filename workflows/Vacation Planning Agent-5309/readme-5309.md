Vacation Planning Agent

https://n8nworkflows.xyz/workflows/vacation-planning-agent-5309


# Vacation Planning Agent

### 1. Workflow Overview

The **Vacation Planning Agent** is an automated conversational assistant designed to help users plan their hotel accommodations for vacations. It uses a multi-step AI-driven interaction to systematically collect travel details, then performs an automatic hotel search and presents personalized recommendations. The workflow is tailored for travel agencies, hospitality services, or any platform requiring dynamic vacation planning assistance.

The logic is divided into the following blocks:

- **1.1 Input Reception:** Captures user input via a conversational chat trigger.
- **1.2 AI Processing:** Runs the AI agent that manages dialogue flow, gathers user requirements, and coordinates the search.
- **1.3 Language Model Integration:** Uses OpenAI’s GPT-4o-mini model for natural language understanding and generation.
- **1.4 Memory Buffer:** Maintains a conversation context window for coherent multi-turn dialogue.
- **1.5 Hotel Search Tool:** Executes an external hotel search via SerpAPI Google Hotels, based on collected parameters.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow when a user starts a chat session. It acts as the entry point, providing an initial greeting and prompt.

- **Nodes Involved:**  
  - **Chat Trigger**

- **Node Details:**

  - **Chat Trigger**  
    - **Type:** LangChain Chat Trigger node  
    - **Role:** Starts a chat session and sends an initial greeting message to the user.  
    - **Configuration:**  
      - Webhook ID: `urlaubsplaner-multi-step` (publicly accessible endpoint)  
      - Initial Message: "Hallo! Ich helfe dir, deinen perfekten Urlaub zu planen. Bitte beanworte mir die folgenden Fragen :)" (German for "Hello! I will help you plan your perfect vacation. Please answer the following questions :)")  
      - Public: true (allows external access)  
    - **Input/Output:**  
      - Input: HTTP request from user via webhook  
      - Output: User messages and interaction stream to AI Agent  
    - **Edge Cases:**  
      - Webhook failures or unavailability  
      - User input outside expected format (handled downstream)  
    - **Version:** 1.1

---

#### 2.2 AI Processing

- **Overview:**  
  Central logic block where the AI agent manages the conversational flow, collects travel details, uses the hotel search tool, and formulates responses based on user data and search results.

- **Nodes Involved:**  
  - AI Agent

- **Node Details:**

  - **AI Agent**  
    - **Type:** LangChain Agent node  
    - **Role:** Orchestrates the multi-step conversation and invokes tools (like hotel search) based on gathered information.  
    - **Configuration:**  
      - System message defines the agent’s persona as a professional vacation planner.  
      - Contains detailed prompt instructions for stepwise information gathering: destination, dates, guests, rooms, optional budget and preferences.  
      - Automatically triggers the Search for Hotels tool once essential info is complete, no user permission needed.  
      - Specifies response style guidelines and example interaction flow.  
      - Uses a conversational, professional, and helpful tone.  
    - **Input/Output:**  
      - Input: Messages from Chat Trigger, memory context, and OpenAI language model outputs.  
      - Output: Responses to user, tool invocations, and dialogue management.  
    - **Edge Cases:**  
      - Missing or ambiguous user inputs (agent prompts for clarification).  
      - Tool failures (e.g., hotel search API errors or empty results).  
      - User changing parameters mid-conversation (agent adapts).  
    - **Version:** 2

---

#### 2.3 Language Model Integration

- **Overview:**  
  Provides the natural language understanding and generation capabilities supporting the AI Agent’s conversation with the user.

- **Nodes Involved:**  
  - OpenAI Chat Model

- **Node Details:**

  - **OpenAI Chat Model**  
    - **Type:** LangChain OpenAI Chat Model node  
    - **Role:** Processes conversational prompts and generates AI responses using GPT-4o-mini.  
    - **Configuration:**  
      - Model: `gpt-4o-mini` (optimized GPT-4 variant for chat)  
      - No additional options enabled  
      - Credentials: Uses n8n’s configured OpenAI API key with free credits  
    - **Input/Output:**  
      - Input: Prompts from AI Agent  
      - Output: AI-generated text responses to AI Agent  
    - **Edge Cases:**  
      - API rate limits or quota exhaustion  
      - Network timeouts or connection errors  
      - Unexpected model output or hallucination (mitigated by agent prompt design)  
    - **Version:** 1.2

---

#### 2.4 Memory Buffer

- **Overview:**  
  Maintains a sliding window of prior conversation turns to provide context for coherent multi-turn dialogue.

- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**

  - **Simple Memory**  
    - **Type:** LangChain Memory Buffer Window node  
    - **Role:** Holds last 20 conversational exchanges to inform AI responses without overwhelming prompt size.  
    - **Configuration:**  
      - Context window length: 20 messages  
    - **Input/Output:**  
      - Input: Conversation messages from OpenAI Chat Model and user inputs  
      - Output: Contextual memory data to AI Agent  
    - **Edge Cases:**  
      - Context truncation if conversation is very long (older messages dropped)  
      - Memory synchronization issues (rare)  
    - **Version:** 1.3

---

#### 2.5 Hotel Search Tool

- **Overview:**  
  Automatically performs a hotel search using SerpAPI’s Google Hotels engine based on parameters collected by the AI Agent.

- **Nodes Involved:**  
  - Search for hotels

- **Node Details:**

  - **Search for hotels**  
    - **Type:** HTTP Request Tool node  
    - **Role:** Queries SerpAPI Google Hotels to fetch hotel listings matching user criteria.  
    - **Configuration:**  
      - URL: `https://serpapi.com/search`  
      - Query Parameters:  
        - `q`: Search query string (e.g., "hotels in Paris, France") dynamically set by AI Agent  
        - `check_in_date`, `check_out_date`: Dates in YYYY-MM-DD format  
        - `min_price`, `max_price`: Optional price range filters  
        - `currency`: Optional currency code (defaults to USD)  
        - `gl`: Country code set to `de` (Germany) for localization  
        - `engine`: set to `google_hotels` to specify search type  
      - Authentication: Uses predefined SerpAPI credentials  
    - **Input/Output:**  
      - Input: Parameters supplied by AI Agent via expressions  
      - Output: JSON response containing hotel search results to AI Agent  
    - **Edge Cases:**  
      - API key quota limits or invalid credentials  
      - No search results found for given parameters  
      - Invalid/malformed query parameters  
      - Network or service downtime  
    - **Version:** 4.2

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role                  | Input Node(s)      | Output Node(s)     | Sticky Note                        |
|--------------------|----------------------------------|--------------------------------|--------------------|--------------------|----------------------------------|
| Chat Trigger       | LangChain Chat Trigger            | Entry point, user input capture | HTTP webhook       | AI Agent           |                                  |
| AI Agent           | LangChain Agent                  | Conversation management & tool invocation | Chat Trigger, Simple Memory, OpenAI Chat Model, Search for hotels | -                  |                                  |
| OpenAI Chat Model  | LangChain OpenAI Chat Model       | NLP processing and response generation | AI Agent           | AI Agent           |                                  |
| Simple Memory      | LangChain Memory Buffer Window    | Context retention for multi-turn dialogue | AI Agent           | AI Agent           |                                  |
| Search for hotels  | HTTP Request Tool (SerpAPI)       | Hotel search based on user criteria | AI Agent           | AI Agent           | This tool searches for hotels based on user query and returns response |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Set Webhook ID: `urlaubsplaner-multi-step`  
   - Set Public: true  
   - Initial Message: `"Hallo! Ich helfe dir, deinen perfekten Urlaub zu planen. Bitte beanworte mir die folgenden Fragen :)"`  
   - Position: (e.g., x=780, y=0)

2. **Create OpenAI Chat Model node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `gpt-4o-mini`  
   - Credentials: Configure OpenAI API with valid key (e.g., use n8n free OpenAI API credits)  
   - Position: (e.g., x=960, y=260)

3. **Create Simple Memory node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Context Window Length: 20 messages  
   - Position: (e.g., x=1120, y=260)

4. **Create Search for hotels node**  
   - Type: `n8n-nodes-base.httpRequestTool`  
   - URL: `https://serpapi.com/search`  
   - Authentication: SerpAPI credentials (account must be set up)  
   - Query Parameters (all using expressions to get values from AI Agent):  
     - `q`: hotel search query string  
     - `check_in_date`: YYYY-MM-DD format  
     - `check_out_date`: YYYY-MM-DD format  
     - `min_price`: optional, string or empty  
     - `max_price`: optional, string or empty  
     - `currency`: optional, default USD  
     - `gl`: `de` (for German localization)  
     - `engine`: `google_hotels`  
   - Position: (e.g., x=1280, y=260)

5. **Create AI Agent node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - System Message: Paste the detailed vacation planning prompt (as in workflow description)  
   - Connect inputs:  
     - From Chat Trigger (main input)  
     - From Simple Memory (ai_memory)  
     - From OpenAI Chat Model (ai_languageModel)  
     - From Search for hotels (ai_tool)  
   - Connect output: to wherever responses should go (typically back to Chat Trigger)  
   - Position: (e.g., x=1080, y=0)

6. **Connect nodes in this order:**  
   - Chat Trigger → AI Agent (main)  
   - OpenAI Chat Model → AI Agent (ai_languageModel)  
   - Simple Memory → AI Agent (ai_memory)  
   - Search for hotels → AI Agent (ai_tool)

7. **Configure credentials:**  
   - OpenAI API key for OpenAI Chat Model node  
   - SerpAPI account credentials for Search for hotels node

8. **Save and activate the workflow** for testing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                              | Context or Link                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| The system message contains a comprehensive prompt detailing the agent's role, interaction style, and multi-step information gathering process for vacation planning.                                     | Workflow "AI Agent" node parameters                      |
| The hotel search uses SerpAPI's Google Hotels engine, localized for Germany (`gl=de`), but can be adjusted for other locales by changing this parameter.                                                  | SerpAPI documentation: https://serpapi.com/             |
| OpenAI GPT-4o-mini is chosen for balanced performance and cost-efficiency for conversational AI tasks.                                                                                                    | OpenAI API docs: https://platform.openai.com/docs/models |
| The conversation memory buffer is limited to 20 messages to avoid prompt size issues and maintain response relevance.                                                                                    | LangChain Memory docs                                      |
| The Chat Trigger node exposes a public webhook endpoint; ensure security considerations if deploying in production.                                                                                       | n8n webhook docs                                          |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated workflow built with n8n, adhering strictly to content policies without illegal, offensive, or protected elements. All data handled are legal and public.