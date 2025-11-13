Real-time Crypto Market Analysis with GPT-4o-mini and CoinMarketCap APIs via Telegram

https://n8nworkflows.xyz/workflows/real-time-crypto-market-analysis-with-gpt-4o-mini-and-coinmarketcap-apis-via-telegram-6160


# Real-time Crypto Market Analysis with GPT-4o-mini and CoinMarketCap APIs via Telegram

---

## 1. Workflow Overview

This workflow, titled **"Real-time Crypto Market Analysis with GPT-4o-mini and CoinMarketCap APIs via Telegram"**, is designed as an advanced, AI-powered crypto market analysis and chatbot system. It integrates real-time cryptocurrency data from multiple CoinMarketCap API endpoints and delivers insights directly to users via Telegram.

The workflow’s core use cases include:  
- Responding to user queries about cryptocurrency prices, metadata, market rankings, conversions, and global metrics.  
- Leveraging an AI agent architecture that intelligently selects and queries appropriate CoinMarketCap API tools based on user intent.  
- Maintaining conversational context with memory buffers and session IDs to provide coherent multi-turn dialogues.  
- Supporting multi-agent coordination, combining centralized exchange, decentralized exchange (DEX), and crypto data analysis.  

### Logical Blocks:

- **1.1 Input Reception & Preprocessing**: Receives user messages from Telegram and adds session context.  
- **1.2 AI Multi-Agent Core**: The main AI analyst agent that orchestrates specialized sub-agents for crypto data analysis using LangChain and GPT-4o-mini.  
- **1.3 CoinMarketCap Tools Layer**: A suite of HTTP request nodes representing CoinMarketCap API endpoints to supply live data feeds.  
- **1.4 Telegram Messaging**: Sends processed AI responses back to the user’s Telegram chat.  
- **1.5 Sub-Workflow Invocations**: Modular sub-agents invoked via toolWorkflow nodes for separation of concerns and scalability.  
- **1.6 Executable Trigger for External Workflow Calls**: Allows this workflow to be executed by other workflows with message/session inputs.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Preprocessing

**Overview:**  
This block captures incoming Telegram messages, extracts session information to maintain chat context, and prepares the input for AI processing.

**Nodes Involved:**  
- Telegram Input  
- Adds SessionId

**Node Details:**

- **Telegram Input**  
  - Type: Telegram Trigger  
  - Role: Entry point listening for Telegram messages (updates of type “message”).  
  - Configuration: Uses Telegram API credentials to receive messages in real time.  
  - Inputs: External Telegram messages.  
  - Outputs: JSON containing message details (including chat ID and text).  
  - Potential Failures: Telegram API downtime, webhook misconfiguration, credential expiry.  

- **Adds SessionId**  
  - Type: Set node  
  - Role: Adds a sessionId field derived from `message.chat.id` to the data payload for tracking user conversation context.  
  - Configuration: Assigns `sessionId` = `{{$json.message.chat.id}}` and passes through other fields.  
  - Inputs: Output from Telegram Input node.  
  - Outputs: Enriched message JSON with sessionId.  
  - Edge Cases: Missing or malformed chat ID in incoming message might cause sessionId assignment failure.

---

### 2.2 AI Multi-Agent Core

**Overview:**  
This block contains the central AI analyst agent that interprets user queries and decides how to query CoinMarketCap data sources. It uses GPT-4o-mini models and manages conversational memory.

**Nodes Involved:**  
- CoinMarketCap AI Data Analyst Agent  
- CoinMarketCap Agent Brain (LM Chat OpenAI)  
- CoinMarketCap Memory (Memory Buffer Window)  
- Crypto Agent Brain (LM Chat OpenAI)  
- Crypto Agent Memory (Memory Buffer Window)

**Node Details:**

- **CoinMarketCap AI Data Analyst Agent**  
  - Type: LangChain Agent  
  - Role: Main AI agent coordinating multiple sub-agents specialized in different CoinMarketCap data domains (crypto, exchange/community, DEX).  
  - Configuration:  
    - Input text: user message text from Adds SessionId node.  
    - System prompt specifies multi-agent roles, capabilities, and usage guidelines to ensure proper API queries.  
    - Integrated tools: sub-agents via toolWorkflow nodes.  
  - Inputs: User message text and sessionId for context.  
  - Outputs: Structured AI response text for Telegram.  
  - Failure Modes: OpenAI API errors, rate limits, malformed queries causing 400 errors in sub-tools.

- **CoinMarketCap Agent Brain**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Language model backend (GPT-4o-mini) used by the main AI Data Analyst Agent for generating responses.  
  - Configuration: Model set to `gpt-4o-mini`.  
  - Credentials: OpenAI API key configured.  
  - Inputs: Messages from AI agent nodes.  
  - Outputs: Generated text for AI agent.

- **CoinMarketCap Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation history and state for the AI Data Analyst Agent, enabling multi-turn dialogues.  
  - Inputs/Outputs: Connected to the AI agent node.

- **Crypto Agent Brain**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Language model for the Crypto-specific sub-agent.  
  - Configuration: Model `gpt-4o-mini-2024-07-18`.  
  - Credentials: OpenAI API key.  
  - Connected to the CoinMarketCap Crypto Agent tool for generating crypto-focused responses.

- **Crypto Agent Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Stores conversational context for the Crypto Agent.

---

### 2.3 CoinMarketCap Tools Layer

**Overview:**  
This block implements direct HTTP requests to CoinMarketCap APIs wrapped as LangChain tools, providing live crypto data feeding the AI agents.

**Nodes Involved:**  
- CoinMarketCap Crypto Agent  
- Crypto Map  
- Crypto Info  
- Crypto Listings  
- CoinMarketCap Price  
- Global Metrics  
- Price Conversion

**Node Details:**

- **CoinMarketCap Crypto Agent**  
  - Type: LangChain Agent  
  - Role: AI agent dedicated to cryptocurrency-centric queries, dynamically selecting among six CoinMarketCap API tools to answer user intents.  
  - Configuration: System message describes each tool’s purpose, supported inputs, use cases, and API endpoints.  
  - Inputs: Text input from AI models or upstream nodes.  
  - Outputs: API responses or structured results for higher-level agents.

- **Crypto Map**  
  - Type: LangChain HTTP Request Tool  
  - Role: Retrieves mapping data for cryptocurrencies (IDs, symbols, names).  
  - HTTP Request: `GET /v1/cryptocurrency/map`  
  - Parameters: Optional `symbol`, `listing_status`, `start`, `limit` for filtering.  
  - Config: Uses CoinMarketCap API key via HTTP header authentication.  
  - Outputs: JSON with crypto map data.

- **Crypto Info**  
  - Type: LangChain HTTP Request Tool  
  - Role: Fetches detailed metadata for cryptocurrencies (description, whitepaper, social links).  
  - HTTP Request: `GET /v2/cryptocurrency/info`  
  - Parameters: Required `symbol`.  
  - Authentication: CoinMarketCap API key header.  

- **Crypto Listings**  
  - Type: LangChain HTTP Request Tool  
  - Role: Returns ranked lists of cryptocurrencies by market cap.  
  - HTTP Request: `GET /v1/cryptocurrency/listings/latest`  
  - Parameters: `start`, `limit`, `convert` (currency).  
  - Use Case: Top coins queries.

- **CoinMarketCap Price**  
  - Type: LangChain HTTP Request Tool  
  - Role: Retrieves real-time price, volume, market cap data for coins.  
  - HTTP Request: `GET /v2/cryptocurrency/quotes/latest`  
  - Parameters: `symbol`, `convert`.  
  - Authentication: CoinMarketCap API key header.

- **Global Metrics**  
  - Type: LangChain HTTP Request Tool  
  - Role: Provides global crypto market statistics (total market cap, BTC dominance, etc).  
  - HTTP Request: `GET /v1/global-metrics/quotes/latest`  
  - No parameters required.  
  - Auth: CoinMarketCap API key header.

- **Price Conversion**  
  - Type: LangChain HTTP Request Tool  
  - Role: Converts amounts between cryptocurrencies and fiat currencies.  
  - HTTP Request: `GET /v1/tools/price-conversion`  
  - Parameters: `amount`, `symbol`, `convert`.  
  - Auth: CoinMarketCap API key header.

**Edge Cases & Failures for Tools:**  
- API rate limits or quota exceeded.  
- Invalid or missing required parameters causing 400 errors.  
- Network timeouts or connectivity issues.  
- Authentication failures due to expired or invalid API keys.  
- Large response payloads exceeding model context limits.

---

### 2.4 Telegram Messaging

**Overview:**  
This block delivers the AI-generated analysis or answers back to the Telegram user.

**Nodes Involved:**  
- Telegram Send Message

**Node Details:**

- **Telegram Send Message**  
  - Type: Telegram Node  
  - Role: Sends text messages to the chat identified by the user’s chat ID.  
  - Configuration:  
    - Chat ID dynamically taken from the original Telegram message (`$('Telegram Input').item.json.message.chat.id`).  
    - Message text is the AI agent’s output (`{{$json.output}}`).  
  - Inputs: AI Data Analyst Agent output.  
  - Outputs: Confirmation of message sent or error.  
  - Failures: Telegram API errors, invalid chat ID, messaging limits.

---

### 2.5 Sub-Workflow Invocations

**Overview:**  
For modularity and separation of responsibilities, the AI Data Analyst Agent calls three sub-workflows representing specialized agents for different CoinMarketCap data domains.

**Nodes Involved:**  
- CoinMarketCap Crypto Agent Tool  
- CoinMarketCap Exchange and Community Agent Tool  
- CoinMarketCap DEXScan Agent Tool  
- When Executed by Another Workflow (trigger for external calls)

**Node Details:**

- **CoinMarketCap Crypto Agent Tool**  
  - Type: LangChain Tool Workflow Node  
  - Role: Invokes a sub-workflow (`CoinMarketCap_Crypto_Agent_Tool`) dedicated to crypto-level data analysis.  
  - Inputs: Passes user message text and sessionId.  
  - Outputs: Returns processed crypto agent results to main AI agent.  
  - Configuration: Workflow ID references the crypto subflow.

- **CoinMarketCap Exchange and Community Agent Tool**  
  - Type: LangChain Tool Workflow Node  
  - Role: Invokes a sub-workflow handling exchange and community data analysis.  
  - Inputs/Outputs similar to Crypto Agent Tool.

- **CoinMarketCap DEXScan Agent Tool**  
  - Type: LangChain Tool Workflow Node  
  - Role: Invokes sub-workflow for decentralized exchange (DEX) data and insights.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be triggered externally, providing `message` and `sessionId` as inputs for integration with other workflows or automation systems.

---

## 3. Summary Table

| Node Name                                   | Node Type                                | Functional Role                             | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                                           |
|---------------------------------------------|-----------------------------------------|---------------------------------------------|--------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Telegram Input                              | Telegram Trigger                        | Receives Telegram user messages             | (External)                     | Adds SessionId                   |                                                                                                                       |
| Adds SessionId                             | Set                                    | Adds sessionId for conversation context     | Telegram Input                | CoinMarketCap AI Data Analyst Agent |                                                                                                                       |
| CoinMarketCap AI Data Analyst Agent        | LangChain Agent                        | Main AI agent coordinating sub-agents       | Adds SessionId                | Telegram Send Message            |                                                                                                                       |
| Telegram Send Message                       | Telegram Node                         | Sends AI response back to Telegram chat     | CoinMarketCap AI Data Analyst Agent | (External)                     |                                                                                                                       |
| CoinMarketCap Agent Brain                   | LM Chat OpenAI (GPT-4o-mini)           | Language model backend for AI analyst       | CoinMarketCap AI Data Analyst Agent | CoinMarketCap AI Data Analyst Agent |                                                                                                                       |
| CoinMarketCap Memory                        | Memory Buffer Window                   | Conversation memory for main AI agent       | CoinMarketCap AI Data Analyst Agent | CoinMarketCap AI Data Analyst Agent |                                                                                                                       |
| Crypto Agent Brain                          | LM Chat OpenAI (GPT-4o-mini-2024-07-18) | Language model for Crypto sub-agent          | CoinMarketCap Crypto Agent    | CoinMarketCap Crypto Agent      |                                                                                                                       |
| Crypto Agent Memory                         | Memory Buffer Window                   | Conversation memory for Crypto sub-agent     | CoinMarketCap Crypto Agent    | CoinMarketCap Crypto Agent      |                                                                                                                       |
| CoinMarketCap Crypto Agent                  | LangChain Agent                        | Crypto-focused AI agent with CoinMarketCap tools | When Executed by Another Workflow, Crypto Map, Crypto Info, Crypto Listings, CoinMarketCap Price, Global Metrics, Price Conversion | CoinMarketCap AI Data Analyst Agent |                                                                                                                       |
| Crypto Map                                 | LangChain HTTP Request Tool            | Retrieves crypto mapping data                 | CoinMarketCap Crypto Agent    | CoinMarketCap Crypto Agent      |                                                                                                                       |
| Crypto Info                                | LangChain HTTP Request Tool            | Retrieves crypto metadata                      | CoinMarketCap Crypto Agent    | CoinMarketCap Crypto Agent      |                                                                                                                       |
| Crypto Listings                            | LangChain HTTP Request Tool            | Retrieves ranked crypto listings               | CoinMarketCap Crypto Agent    | CoinMarketCap Crypto Agent      |                                                                                                                       |
| CoinMarketCap Price                        | LangChain HTTP Request Tool            | Retrieves real-time crypto price data          | CoinMarketCap Crypto Agent    | CoinMarketCap Crypto Agent      |                                                                                                                       |
| Global Metrics                             | LangChain HTTP Request Tool            | Retrieves global crypto market stats           | CoinMarketCap Crypto Agent    | CoinMarketCap Crypto Agent      |                                                                                                                       |
| Price Conversion                           | LangChain HTTP Request Tool            | Converts crypto/fiat values                     | CoinMarketCap Crypto Agent    | CoinMarketCap Crypto Agent      |                                                                                                                       |
| CoinMarketCap Crypto Agent Tool            | LangChain Tool Workflow                | Invokes crypto sub-workflow                      | CoinMarketCap AI Data Analyst Agent | CoinMarketCap AI Data Analyst Agent |                                                                                                                       |
| CoinMarketCap Exchange and Community Agent Tool | LangChain Tool Workflow                | Invokes exchange/community sub-workflow          | CoinMarketCap AI Data Analyst Agent | CoinMarketCap AI Data Analyst Agent |                                                                                                                       |
| CoinMarketCap DEXScan Agent Tool            | LangChain Tool Workflow                | Invokes DEXScan sub-workflow                      | CoinMarketCap AI Data Analyst Agent | CoinMarketCap AI Data Analyst Agent |                                                                                                                       |
| When Executed by Another Workflow           | Execute Workflow Trigger               | Allows external workflows to trigger this one | (External)                   | CoinMarketCap Crypto Agent      |                                                                                                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Input Node**  
   - Type: Telegram Trigger  
   - Configure with Telegram API credentials.  
   - Set updates to listen for `"message"`.  
   - Position near workflow start.

2. **Create Adds SessionId Node**  
   - Type: Set  
   - Assign field `sessionId` = `{{$json.message.chat.id}}`.  
   - Include all other incoming fields.  
   - Connect output of Telegram Input to this node.

3. **Create CoinMarketCap AI Data Analyst Agent Node**  
   - Type: LangChain Agent  
   - Configure input text: `{{$json.message.text}}` from Adds SessionId.  
   - Paste or define system prompt detailing the multi-agent architecture, tool descriptions, and usage instructions.  
   - Connect Adds SessionId output to this node.

4. **Configure CoinMarketCap Agent Brain Node**  
   - Type: LangChain LM Chat OpenAI  
   - Set model to `gpt-4o-mini`.  
   - Set OpenAI API credentials.  
   - Connect output to CoinMarketCap AI Data Analyst Agent’s language model input.

5. **Configure CoinMarketCap Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Connect input and output to the AI Data Analyst Agent for conversation history.

6. **Create Crypto Agent Brain Node**  
   - Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o-mini-2024-07-18`.  
   - Set OpenAI credentials.  
   - Connect to CoinMarketCap Crypto Agent Tool.

7. **Create Crypto Agent Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Connect to Crypto Agent Brain node.

8. **Create CoinMarketCap Crypto Agent Node**  
   - Type: LangChain Agent  
   - Paste system message describing the six CoinMarketCap API tools (Crypto Map, Crypto Info, Listings, Price, Global Metrics, Price Conversion) with their inputs and use cases.  
   - Connect language model (Crypto Agent Brain) and memory nodes (Crypto Agent Memory).  
   - Connect to sub-tools (HTTP Request nodes below).

9. **Create HTTP Request Nodes for CoinMarketCap APIs:**

   - For each of the following nodes, configure as HTTP Request with:  
     - URL set to the CoinMarketCap API endpoint.  
     - Method: GET.  
     - Send query parameters as specified (e.g., symbol, convert).  
     - Send headers with: `Accept: application/json` and API key via header authentication.  
     - Use HTTP Header Auth credentials holding CoinMarketCap API key.

   a. **Crypto Map**: `/v1/cryptocurrency/map`  
      - Optional query params: symbol, listing_status, start, limit.

   b. **Crypto Info**: `/v2/cryptocurrency/info`  
      - Required query param: symbol.

   c. **Crypto Listings**: `/v1/cryptocurrency/listings/latest`  
      - Query params: start, limit, convert.

   d. **CoinMarketCap Price**: `/v2/cryptocurrency/quotes/latest`  
      - Required: symbol, convert.

   e. **Global Metrics**: `/v1/global-metrics/quotes/latest`  
      - No params.

   f. **Price Conversion**: `/v1/tools/price-conversion`  
      - Required: amount, symbol, convert.

   - Connect all these HTTP nodes as tools to the CoinMarketCap Crypto Agent node.

10. **Create Sub-Workflow Tool Nodes for Multi-Agent Architecture:**

    - Create three toolWorkflow nodes, each invoking a separate workflow:

      - CoinMarketCap Crypto Agent Tool: Pass message text and sessionId.  
      - CoinMarketCap Exchange and Community Agent Tool: Pass same inputs.  
      - CoinMarketCap DEXScan Agent Tool: Pass same inputs.

    - Configure workflow IDs accordingly.

    - Connect these toolWorkflow nodes as tools to the main CoinMarketCap AI Data Analyst Agent node.

11. **Create CoinMarketCap Agent Brain and Memory Nodes:**

    - LM Chat OpenAI node with model `gpt-4o-mini`.  
    - Memory Buffer Window node.  
    - Connect to main AI Data Analyst Agent node.

12. **Create Telegram Send Message Node:**

    - Type: Telegram Node  
    - Configure with Telegram API credentials.  
    - Set `chatId` to `{{$json.message.chat.id}}` from original Telegram Input node.  
    - Set text to output from AI Data Analyst Agent node's generated response.  
    - Connect AI Data Analyst Agent node output to this node.

13. **Create Execute Workflow Trigger Node (Optional):**

    - To enable external invocation, create an Execute Workflow Trigger node.  
    - Define inputs `message` and `sessionId`.  
    - Connect output to CoinMarketCap Crypto Agent node or appropriate start node.

14. **Set Workflow Execution Order:**  
    - Ensure Telegram Input → Adds SessionId → AI Data Analyst Agent → Telegram Send Message is the main execution chain.  
    - Tools and sub-agents connected internally as per LangChain conventions.

15. **Test End-to-End:**  
    - Validate Telegram bot receives messages, AI processes queries, CoinMarketCap APIs are called, and responses return to Telegram chat.  
    - Monitor for API errors and adjust parameters accordingly.

---

## 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses GPT-4o-mini, an advanced openAI model optimized for multi-turn conversations with crypto data.| OpenAI official docs: https://platform.openai.com/docs/models/gpt-4o-mini                          |
| CoinMarketCap API key is required with appropriate subscription to access endpoints used in this workflow.     | CoinMarketCap API docs: https://coinmarketcap.com/api/documentation/v1/                            |
| Telegram Bot must be created and authorized with appropriate permissions and webhook configured.               | Telegram Bot API: https://core.telegram.org/bots/api                                                |
| The multi-agent architecture allows for scalable expansion with additional CoinMarketCap data domains.         | LangChain multi-agent design patterns: https://docs.langchain.com/docs/multi-agent                 |
| To avoid 400 errors, queries must respect parameter validation, especially required fields and data types.     | Refer to system prompt instructions embedded in AI Agent nodes.                                   |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.

---