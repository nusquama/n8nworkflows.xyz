Get Real-time Crypto Token Insights via Telegram with DexScreener and GPT-4o

https://n8nworkflows.xyz/workflows/get-real-time-crypto-token-insights-via-telegram-with-dexscreener-and-gpt-4o-3178


# Get Real-time Crypto Token Insights via Telegram with DexScreener and GPT-4o

### 1. Workflow Overview

This workflow, titled **"Get Real-time Crypto Token Insights via Telegram with DexScreener and GPT-4o"**, is designed to provide instant, AI-enhanced decentralized exchange (DEX) insights directly to users via Telegram. It targets crypto traders, DeFi analysts, and investors who require up-to-date token analytics, liquidity data, and market trends without leaving their messaging app.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Captures user queries from Telegram or chat messages.
- **1.2 Session Context Management**: Adds session identifiers to track user interactions.
- **1.3 AI Processing & Memory**: Uses GPT-4o-mini and window buffer memory to interpret queries and maintain conversational context.
- **1.4 DexScreener API Tool Integration**: Provides a suite of HTTP request nodes acting as tools to fetch various real-time blockchain DEX data from DexScreener.
- **1.5 Agent Coordination**: The LangChain Agent node orchestrates which DexScreener tools to invoke based on user input and AI interpretation.
- **1.6 Output Delivery**: Sends the AI-enhanced insights back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming user messages either from Telegram or direct chat triggers, serving as the entry point for user queries.

- **Nodes Involved:**  
  - Telegram Trigger  
  - When chat message received

- **Node Details:**  

  - **Telegram Trigger**  
    - Type: `telegramTrigger` (n8n core node)  
    - Role: Listens for Telegram messages (update type: "message")  
    - Configuration: Uses Telegram API credentials; webhook enabled for receiving messages  
    - Inputs: External Telegram messages  
    - Outputs: Passes message JSON to next node  
    - Edge Cases: Telegram API downtime, invalid bot token, webhook misconfiguration  
    - Notes: Requires Telegram bot token configured in n8n credentials

  - **When chat message received**  
    - Type: `chatTrigger` (LangChain node)  
    - Role: Listens for chat messages in LangChain context (alternative input)  
    - Configuration: Default options, webhook enabled  
    - Inputs: External chat messages  
    - Outputs: Passes chat message JSON to next node  
    - Edge Cases: Webhook failures, message format issues

#### 2.2 Session Context Management

- **Overview:**  
  Adds a session identifier to each incoming message to track user sessions and maintain context throughout the workflow.

- **Nodes Involved:**  
  - Adds SessionId

- **Node Details:**  

  - **Adds SessionId**  
    - Type: `set` (n8n core node)  
    - Role: Adds a `sessionId` field to the message JSON, set to the Telegram chat ID  
    - Configuration: Assigns `sessionId` = `{{$json.message.chat.id}}`  
    - Inputs: From Telegram Trigger or When chat message received  
    - Outputs: Passes enriched JSON to the Agent node  
    - Edge Cases: Missing or malformed chat ID, non-Telegram messages without chat ID  
    - Notes: Ensures session tracking for AI memory and logging

#### 2.3 AI Processing & Memory

- **Overview:**  
  This block interprets user queries using GPT-4o-mini and maintains conversational context with window buffer memory.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Window Buffer Memory

- **Node Details:**  

  - **OpenAI Chat Model**  
    - Type: `lmChatOpenAi` (LangChain node)  
    - Role: Processes user input with GPT-4o-mini model to generate AI responses  
    - Configuration: Model set to `gpt-4o-mini`, no additional options  
    - Credentials: Uses OpenAI API credentials configured in n8n  
    - Inputs: Connected as AI language model input to Agent node  
    - Outputs: AI-generated text for further processing  
    - Edge Cases: API rate limits, invalid API key, network timeouts, model unavailability

  - **Window Buffer Memory**  
    - Type: `memoryBufferWindow` (LangChain node)  
    - Role: Maintains a sliding window of recent conversation history to provide context to the AI model  
    - Configuration: Default settings (window size unspecified)  
    - Inputs: Connected as AI memory input to Agent node  
    - Outputs: Provides context data to AI model  
    - Edge Cases: Memory overflow if window size too large, loss of context if window too small

#### 2.4 DexScreener API Tool Integration

- **Overview:**  
  Implements multiple HTTP request nodes configured as LangChain tools to fetch specific real-time data from the DexScreener API. These tools provide granular blockchain DEX insights such as token profiles, boosted tokens, trading pairs, orders, and liquidity pools.

- **Nodes Involved:**  
  - DexScreener Latest Token Profiles  
  - DexScreener Latest Boosted Tokens  
  - DexScreener Top Token Boosts  
  - DexScreener Search Pairs  
  - DexScreener Check Orders Paid for Token  
  - DexScreener Get Pairs by Chain and Pair Address  
  - DexScreener Token Pools  
  - DexScreener Pairs by Token Address

- **Node Details:**  

  Each node is a LangChain `toolHttpRequest` node configured as follows (common attributes unless otherwise noted):

  - **Type:** `toolHttpRequest`  
  - **Role:** Fetches specific data from DexScreener API endpoints  
  - **Headers:** Sends `Accept: */*` header by default  
  - **Rate Limits:** Vary per endpoint (60 to 300 requests/minute)  
  - **Input Parameters:** Some nodes accept path or query parameters (e.g., chainId, tokenAddress, pairId, query string)  
  - **Outputs:** JSON responses from DexScreener API, passed to the Agent node as tools  
  - **Edge Cases:** API downtime, malformed parameters, rate limiting, network errors

  Specific nodes:

  - **DexScreener Latest Token Profiles**  
    - URL: `https://api.dexscreener.com/token-profiles/latest/v1`  
    - Purpose: Fetches latest token profiles with images, descriptions, links

  - **DexScreener Latest Boosted Tokens**  
    - URL: `https://api.dexscreener.com/token-boosts/latest/v1`  
    - Purpose: Retrieves latest boosted tokens data

  - **DexScreener Top Token Boosts**  
    - URL: `https://api.dexscreener.com/token-boosts/top/v1`  
    - Purpose: Gets tokens with most active boosts

  - **DexScreener Search Pairs**  
    - URL: `https://api.dexscreener.com/latest/dex/search`  
    - Query Parameter: `q` (search query, e.g., "SOL/USDC")  
    - Purpose: Searches trading pairs matching query

  - **DexScreener Check Orders Paid for Token**  
    - URL Template: `https://api.dexscreener.com/orders/v1/{chainId}/{tokenAddress}`  
    - Purpose: Checks paid orders for a token

  - **DexScreener Get Pairs by Chain and Pair Address**  
    - URL Template: `https://api.dexscreener.com/latest/dex/pairs/{chainId}/{pairId}`  
    - Purpose: Retrieves pairs by chain and pair address

  - **DexScreener Token Pools**  
    - URL Template: `https://api.dexscreener.com/token-pairs/v1/{chainId}/{tokenAddress}`  
    - Purpose: Fetches liquidity pools for a token

  - **DexScreener Pairs by Token Address**  
    - URL Template: `https://api.dexscreener.com/tokens/v1/{chainId}/{tokenAddresses}`  
    - Purpose: Retrieves pairs by one or multiple token addresses (comma-separated)

#### 2.5 Agent Coordination

- **Overview:**  
  The LangChain Agent node acts as the central orchestrator, receiving user input and deciding which DexScreener tools to invoke to fulfill the query. It integrates AI language understanding with real-time data fetching.

- **Nodes Involved:**  
  - Blockchain DEX Screener Insights Agent

- **Node Details:**  

  - **Blockchain DEX Screener Insights Agent**  
    - Type: `agent` (LangChain node)  
    - Role: Coordinates AI model, memory, and multiple DexScreener tools to generate comprehensive insights  
    - Configuration:  
      - Input text: User message text from Telegram Trigger or chat message  
      - System message: Detailed instructions describing each DexScreener tool, usage guidelines, rate limits, and expected behavior  
      - Prompt type: "define" (defines the agent’s behavior and toolset)  
    - Inputs:  
      - Main input: JSON with sessionId and user message  
      - AI language model: OpenAI Chat Model node  
      - AI memory: Window Buffer Memory node  
      - AI tools: All DexScreener HTTP request nodes  
    - Outputs: AI-generated output text with integrated DexScreener data  
    - Edge Cases: Tool invocation failures, API rate limits, ambiguous user queries, AI misinterpretation  
    - Notes: This node is the core intelligence of the workflow

#### 2.6 Output Delivery

- **Overview:**  
  Sends the AI-enhanced DEX insights back to the user on Telegram in a readable format.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**  

  - **Telegram**  
    - Type: `telegram` (n8n core node)  
    - Role: Sends messages to Telegram chat  
    - Configuration:  
      - Text: Uses the AI-generated output from the Agent node (`{{$json.output}}`)  
      - Chat ID: Extracted from the original Telegram message (`{{$('Telegram Trigger').item.json.message.chat.id}}`)  
      - Additional fields: Attribution disabled  
    - Credentials: Uses Telegram API credentials  
    - Inputs: From Agent node’s main output  
    - Outputs: Message sent confirmation  
    - Edge Cases: Telegram API errors, invalid chat ID, message length limits

---

### 3. Summary Table

| Node Name                              | Node Type                          | Functional Role                         | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                              |
|--------------------------------------|----------------------------------|---------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------|
| Telegram Trigger                     | telegramTrigger (n8n core)        | Receives Telegram user messages       | External Telegram webhook         | Adds SessionId                   |                                                                                                        |
| When chat message received           | chatTrigger (LangChain)           | Receives chat messages                 | External chat webhook             | Adds SessionId                   |                                                                                                        |
| Adds SessionId                      | set (n8n core)                    | Adds sessionId to track user session  | Telegram Trigger, When chat message received | Blockchain DEX Screener Insights Agent |                                                                                                        |
| OpenAI Chat Model                   | lmChatOpenAi (LangChain)          | AI language model (GPT-4o-mini)       | Connected as AI languageModel input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| Window Buffer Memory                | memoryBufferWindow (LangChain)    | Maintains conversation context        | Connected as AI memory input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| DexScreener Latest Token Profiles  | toolHttpRequest (LangChain)       | Fetches latest token profiles          | Connected as AI tool input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| DexScreener Latest Boosted Tokens  | toolHttpRequest (LangChain)       | Fetches latest boosted tokens          | Connected as AI tool input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| DexScreener Top Token Boosts        | toolHttpRequest (LangChain)       | Fetches tokens with most active boosts | Connected as AI tool input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| DexScreener Search Pairs            | toolHttpRequest (LangChain)       | Searches trading pairs by query        | Connected as AI tool input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| DexScreener Check Orders Paid for Token | toolHttpRequest (LangChain)    | Checks paid orders for a token         | Connected as AI tool input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| DexScreener Get Pairs by Chain and Pair Address | toolHttpRequest (LangChain) | Retrieves pairs by chain and pair address | Connected as AI tool input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| DexScreener Token Pools             | toolHttpRequest (LangChain)       | Fetches liquidity pools for a token   | Connected as AI tool input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| DexScreener Pairs by Token Address  | toolHttpRequest (LangChain)       | Retrieves pairs by one or multiple token addresses | Connected as AI tool input to Agent | Blockchain DEX Screener Insights Agent |                                                                                                        |
| Blockchain DEX Screener Insights Agent | agent (LangChain)               | Orchestrates AI and DexScreener tools | Adds SessionId, OpenAI Chat Model, Window Buffer Memory, all DexScreener tools | Telegram                        | Core node integrating AI and API tools to generate insights                                            |
| Telegram                           | telegram (n8n core)               | Sends AI-enhanced insights to user    | Blockchain DEX Screener Insights Agent | External Telegram chat           |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Configure Credentials**  
   - Use [@BotFather](https://t.me/BotFather) on Telegram to create a bot and obtain the API token.  
   - In n8n, add Telegram API credentials with this token.

2. **Add Telegram Trigger Node**  
   - Type: `telegramTrigger`  
   - Configure to listen for `"message"` updates.  
   - Assign Telegram API credentials.  
   - Enable webhook.

3. **Add When chat message received Node** (Optional alternative input)  
   - Type: `chatTrigger` (LangChain)  
   - Default options, enable webhook.

4. **Add Set Node to Add SessionId**  
   - Type: `set`  
   - Add field `sessionId` with value expression: `{{$json.message.chat.id}}`  
   - Connect outputs of Telegram Trigger and When chat message received to this node.

5. **Add OpenAI Chat Model Node**  
   - Type: `lmChatOpenAi` (LangChain)  
   - Set model to `gpt-4o-mini`  
   - Assign OpenAI API credentials.

6. **Add Window Buffer Memory Node**  
   - Type: `memoryBufferWindow` (LangChain)  
   - Use default settings.

7. **Add DexScreener API Tool Nodes**  
   For each of the following, create a `toolHttpRequest` node with the specified URL, headers, and description:

   - Latest Token Profiles: `https://api.dexscreener.com/token-profiles/latest/v1`  
   - Latest Boosted Tokens: `https://api.dexscreener.com/token-boosts/latest/v1`  
   - Top Token Boosts: `https://api.dexscreener.com/token-boosts/top/v1`  
   - Search Pairs: `https://api.dexscreener.com/latest/dex/search` (add query parameter `q`)  
   - Check Orders Paid for Token: `https://api.dexscreener.com/orders/v1/{chainId}/{tokenAddress}` (path params)  
   - Get Pairs by Chain and Pair Address: `https://api.dexscreener.com/latest/dex/pairs/{chainId}/{pairId}` (path params)  
   - Token Pools: `https://api.dexscreener.com/token-pairs/v1/{chainId}/{tokenAddress}` (path params)  
   - Pairs by Token Address: `https://api.dexscreener.com/tokens/v1/{chainId}/{tokenAddresses}` (path params)

   - For all, add header: `Accept: */*`  
   - Include tool descriptions for clarity.

8. **Add LangChain Agent Node**  
   - Type: `agent`  
   - Set input text to: `={{ $('Telegram Trigger').item.json.message.text }}` (or from Set node if preferred)  
   - Paste the detailed system message describing all DexScreener tools and usage guidelines (as in the original workflow).  
   - Set prompt type to "define".  
   - Connect inputs:  
     - Main input from Set node (with sessionId)  
     - AI language model input from OpenAI Chat Model node  
     - AI memory input from Window Buffer Memory node  
     - AI tool inputs from all DexScreener tool nodes

9. **Add Telegram Node to Send Response**  
   - Type: `telegram`  
   - Set text to: `={{ $json.output }}` (output from Agent node)  
   - Set chatId to: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Assign Telegram API credentials.

10. **Connect Nodes**  
    - Telegram Trigger and When chat message received → Adds SessionId  
    - Adds SessionId → Blockchain DEX Screener Insights Agent (main input)  
    - OpenAI Chat Model → Agent (ai_languageModel input)  
    - Window Buffer Memory → Agent (ai_memory input)  
    - All DexScreener tool nodes → Agent (ai_tool inputs)  
    - Agent → Telegram (main output)

11. **Activate Workflow and Test**  
    - Deploy workflow  
    - Send a message like `"SOL/USDC"` to your Telegram bot  
    - Verify receipt of AI-enhanced DEX insights

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Bot creation instructions available at [@BotFather](https://t.me/BotFather)                 | Telegram bot setup                                                                                |
| No API key required for DexScreener API usage                                                       | Simplifies integration                                                                            |
| Rate limits vary per DexScreener endpoint (60 to 300 requests/minute)                               | Important for scaling and avoiding throttling                                                   |
| Workflow uses GPT-4o-mini model for AI processing                                                   | Requires OpenAI API credentials                                                                  |
| System message in Agent node details all available DexScreener tools and usage guidelines          | Critical for understanding tool capabilities and usage                                          |
| Workflow designed for real-time, actionable insights for crypto traders and DeFi analysts           | Use case context                                                                                  |
| For detailed API documentation, visit DexScreener official site (not linked here)                   | External resource for API parameters and limits                                                 |

---

This document provides a complete, structured reference for understanding, reproducing, and modifying the "Get Real-time Crypto Token Insights via Telegram with DexScreener and GPT-4o" workflow. It covers all nodes, their roles, configurations, and integration details to ensure robust implementation and troubleshooting.