Binance Spot Market Quant AI Agent | GPT-4o + Telegram  (Main Interface)

https://n8nworkflows.xyz/workflows/binance-spot-market-quant-ai-agent---gpt-4o---telegram---main-interface--4739


# Binance Spot Market Quant AI Agent | GPT-4o + Telegram  (Main Interface)

---

# 1. Workflow Overview

This workflow, titled **Binance Spot Market Quant AI Agent**, serves as an advanced AI-driven trading assistant specialized in producing structured, actionable swing-trading reports for any cryptocurrency trading pair on the Binance Spot Market. It is designed to receive user input via Telegram, validate user identity, analyze multi-timeframe technical indicators, order book data, and crypto news sentiment, then generate a detailed trading report formatted for Telegram delivery.

### Logical Blocks:

- **1.1 Input Reception and Authentication**  
  Receives Telegram messages, validates the sender, and attaches session metadata.

- **1.2 AI Agent Orchestration**  
  The core AI agent receives user queries, invokes multiple analytical sub-tools concurrently, and synthesizes a comprehensive market report.

- **1.3 Market Data & Sentiment Analysis Tools**  
  Sub-workflows providing detailed financial technical indicators and news sentiment analysis.

- **1.4 Message Processing and Output Delivery**  
  Handles splitting of large messages if needed and sends the final formatted report back to the user on Telegram.

- **1.5 Memory and Language Model Support**  
  Maintains session state for multi-turn interactions and configures the AI language model used for reasoning and report generation.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Authentication

### Overview  
This block listens for incoming Telegram messages, authenticates the user by Telegram ID, and sets a session identifier for downstream processing.

### Nodes Involved  
- Telegram Trigger  
- User Authentication (Replace Telegram ID)  
- Adds "SessionId"

### Node Details  

**Telegram Trigger**  
- *Type:* Telegram Trigger  
- *Role:* Entry point that listens for all incoming Telegram messages (updates of type "message").  
- *Config:* Uses a webhook to receive updates; credential linked to the Telegram bot "BinanceSpotTradingAIAgent_Bot".  
- *Inputs:* None (trigger node).  
- *Outputs:* Passes raw Telegram message JSON downstream.  
- *Failure modes:* Telegram API connection issues, webhook misconfiguration.  
- *Sticky Note:* "Listens for new Telegram messages from users. Triggers full agent process."  

**User Authentication (Replace Telegram ID)**  
- *Type:* Code (JavaScript)  
- *Role:* Validates that the Telegram message sender matches a specific Telegram user ID (replace placeholder with actual ID).  
- *Config:* JavaScript compares `message.from.id` with hardcoded Telegram ID; blocks unauthorized users by returning `{unauthorized: true}`.  
- *Inputs:* Raw Telegram message JSON.  
- *Outputs:* Passes original data if authorized; blocks otherwise.  
- *Edge Cases:* If ID not replaced or multiple users needed, requires modification. Failure if input JSON structure changes.  
- *Sticky Note:* "Checks incoming Telegram ID against the approved user list."  

**Adds "SessionId"**  
- *Type:* Set  
- *Role:* Assigns a `sessionId` derived from the Telegram chat ID, and extracts the message text to a clean `message` field for downstream tools.  
- *Config:* Assignments: `sessionId = message.chat.id`, `message = message.text`.  
- *Inputs:* Validated Telegram message JSON.  
- *Outputs:* Enriched JSON with `sessionId` and clean `message`.  
- *Sticky Note:* "Creates a sessionId using the Telegram chat_id for memory and routing."  

---

## 2.2 AI Agent Orchestration

### Overview  
This is the core orchestrator that receives user queries, calls multiple analytical tools concurrently, processes the results with OpenAI GPT-4o-mini, and produces a formatted trading report.

### Nodes Involved  
- Binance Spot Market Quant AI Agent  
- OpenAI Chat Model  
- Simple Memory

### Node Details  

**Binance Spot Market Quant AI Agent**  
- *Type:* LangChain Agent Node  
- *Role:* Central AI orchestrator that receives the user message, calls connected analytical tools, processes their structured outputs, and synthesizes a final report.  
- *Config:* Passes user message text as input; system message defines the agent’s role as a professional quant analyst using multiple Binance market tools (financial indicators, price/order book data, news sentiment). Calls two tools concurrently:  
  - Binance SM Financial Analyst Tool  
  - News and Sentiment Analysis Request  
- *Prompt:* Detailed system message instructs on multi-timeframe analysis, technical indicators (RSI, MACD, BBANDS, SMA, EMA, ADX), sentiment integration, report formatting in Telegram HTML, and strict rules (no fabrications, no raw JSON output).  
- *Inputs:* `text` from session message.  
- *Outputs:* Final trading report string (HTML formatted).  
- *Failure modes:* API quota limits, tool failures, malformed inputs, agent prompt errors.  
- *Sticky Note:* "Core orchestrator calling financial and sentiment tools, synthesizes final report."  

**OpenAI Chat Model**  
- *Type:* LangChain OpenAI Chat Model Node  
- *Role:* Provides GPT-4o-mini language model for reasoning within the agent.  
- *Config:* Model set to `gpt-4o-mini`; uses OpenAI API credentials.  
- *Inputs:* Connected as AI language model for the agent node.  
- *Outputs:* Language model completions to agent.  
- *Sticky Note:* "Model used to interpret signals, generate HTML, and recommend trades."  

**Simple Memory**  
- *Type:* LangChain Memory Buffer Window Node  
- *Role:* Maintains session state including sessionId and symbols for multi-turn interactions and indicator tracking.  
- *Config:* Default parameters; no explicit customizations shown.  
- *Inputs:* Connected as AI memory for the agent.  
- *Outputs:* Provides memory context for the agent’s reasoning.  
- *Sticky Note:* "Stores sessionId and state; useful for multi-turn Telegram interactions."  

---

## 2.3 Market Data & Sentiment Analysis Tools

### Overview  
These tools supply the agent with validated, structured market indicators and sentiment data required for comprehensive analysis.

### Nodes Involved  
- Binance SM Financial Analyst Tool  
- News and Sentiment Analysis Request

### Node Details  

**Binance SM Financial Analyst Tool**  
- *Type:* LangChain Tool Workflow Node  
- *Role:* Calls a sub-workflow that aggregates multiple Binance technical indicator tools across timeframes (15m, 1h, 4h, 1d) and price/order book data.  
- *Config:* Calls workflow with ID `"Nm41n4i6VMhrixZs"`; passes `message` and `sessionId` fields as inputs.  
- *Inputs:* Symbol (from user input), sessionId.  
- *Outputs:* Structured technical indicator data for agent use.  
- *Failure modes:* Sub-workflow errors, API limits, data freshness issues.  
- *Sticky Note:* "Calls all connected indicator agents per timeframe plus price/orderbook data."  

**News and Sentiment Analysis Request**  
- *Type:* HTTP Request Tool Node  
- *Role:* Posts symbol to external webhook `https://treasurium.app.n8n.cloud/webhook/newsanalyst` to retrieve sentiment score, news headlines, and summary.  
- *Config:* HTTP POST with body parameter `message` containing the symbol text.  
- *Inputs:* Symbol string.  
- *Outputs:* JSON with sentiment summary and headlines.  
- *Failure modes:* HTTP errors, webhook downtime, malformed responses.  
- *Sticky Note:* "Sends symbol to external service; returns sentiment and crypto news."  

---

## 2.4 Message Processing and Output Delivery

### Overview  
This block manages the final report size, splits large messages safely for Telegram limits, and sends the output back to the user.

### Nodes Involved  
- Splits message is more than 4000 characters  
- Telegram

### Node Details  

**Splits message is more than 4000 characters**  
- *Type:* Code (JavaScript)  
- *Role:* Splits the final report message into chunks of ≤4000 characters to comply with Telegram message size limits.  
- *Config:* Checks length of incoming `output` message; splits into array chunks if oversized; otherwise forwards as single message.  
- *Inputs:* `output` field from agent node.  
- *Outputs:* Array of JSON objects with `message` strings, each safe for Telegram.  
- *Edge Cases:* If message is exactly 4000 chars or slightly longer, ensures no data loss.  
- *Sticky Note:* "Checks if GPT output exceeds 4000 chars; splits into safe chunks."  

**Telegram**  
- *Type:* Telegram Node (Send Message)  
- *Role:* Sends the formatted trading report (full or chunked) back to the Telegram user via bot.  
- *Config:* Uses chat ID from original Telegram message; sends HTML formatted text; disables attribution.  
- *Inputs:* Message chunks from splitter node.  
- *Outputs:* Telegram message delivery.  
- *Failure modes:* Telegram API throttling, message send failures, invalid chat IDs.  
- *Sticky Note:* "Sends formatted HTML report directly to authenticated user."  

---

# 3. Summary Table

| Node Name                              | Node Type                                 | Functional Role                          | Input Node(s)                       | Output Node(s)                            | Sticky Note                                                 |
|--------------------------------------|------------------------------------------|----------------------------------------|-----------------------------------|-------------------------------------------|-------------------------------------------------------------|
| Telegram Trigger                     | Telegram Trigger                         | Listen for new Telegram messages       | None                              | User Authentication (Replace Telegram ID) | Listens for new Telegram messages from users. Triggers full agent process. |
| User Authentication (Replace Telegram ID) | Code                                     | Validate user Telegram ID               | Telegram Trigger                  | Adds "SessionId"                          | Checks incoming Telegram ID against the approved user list. |
| Adds "SessionId"                    | Set                                      | Add sessionId and clean message text   | User Authentication (Replace Telegram ID) | Binance Spot Market Quant AI Agent        | Creates a sessionId using the Telegram chat_id for memory and routing. |
| Binance Spot Market Quant AI Agent  | LangChain Agent Node                     | Core AI orchestrator for report        | Adds "SessionId", Simple Memory, OpenAI Chat Model, Financial Analyst Tool, News and Sentiment Analysis Request | Splits message is more than 4000 characters | Core orchestrator calling financial and sentiment tools, synthesizes final report. |
| OpenAI Chat Model                   | LangChain OpenAI Chat Model              | Language model for reasoning            | Binance Spot Market Quant AI Agent (AI languageModel) | Binance Spot Market Quant AI Agent (AI languageModel) | Model used to interpret signals, generate HTML, and recommend trades. |
| Simple Memory                      | LangChain Memory Buffer Window           | Stores session state                    | Binance Spot Market Quant AI Agent (ai_memory) | Binance Spot Market Quant AI Agent (ai_memory) | Stores sessionId and state; useful for multi-turn Telegram interactions. |
| Binance SM Financial Analyst Tool  | LangChain Tool Workflow                   | Provides multi-timeframe technical data | Binance Spot Market Quant AI Agent (ai_tool) | Binance Spot Market Quant AI Agent (ai_tool) | Calls all connected indicator agents per timeframe plus price/orderbook data. |
| News and Sentiment Analysis Request | HTTP Request Tool                        | Provides news sentiment data            | Binance Spot Market Quant AI Agent (ai_tool) | Binance Spot Market Quant AI Agent (ai_tool) | Sends symbol to external service; returns sentiment and crypto news. |
| Splits message is more than 4000 characters | Code                                     | Splits large output messages            | Binance Spot Market Quant AI Agent | Telegram                                  | Checks if GPT output exceeds 4000 chars; splits into safe chunks. |
| Telegram                          | Telegram Node (Send Message)              | Sends final report to Telegram user    | Splits message is more than 4000 characters | None                                      | Sends formatted HTML report directly to authenticated user. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credential**  
   - Create a Telegram bot via BotFather.  
   - Add Telegram API credential to n8n with bot token.  

2. **Add Telegram Trigger Node**  
   - Type: Telegram Trigger.  
   - Configure webhook with updates set to "message".  
   - Assign Telegram API credential created.  

3. **Add User Authentication Node (Code)**  
   - Type: Code node with JavaScript:  
     ```js
     if ($input.first().json.message.from.id !== <Your Telegram ID>) {
       return { unauthorized: true };
     } else {
       return $input.all();
     }
     ```  
   - Replace `<Your Telegram ID>` with your Telegram user ID.  
   - Connect Telegram Trigger → User Authentication.  

4. **Add Set Node to Add SessionId**  
   - Type: Set node.  
   - Assign `sessionId` = `{{$json.message.chat.id}}`.  
   - Assign `message` = `{{$json.message.text}}`.  
   - Connect User Authentication → Adds "SessionId".  

5. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model.  
   - Set model to `gpt-4o-mini` (or equivalent).  
   - Add OpenAI API credentials.  

6. **Add Simple Memory Node**  
   - Type: LangChain Memory Buffer Window.  
   - Use default settings.  

7. **Add Binance SM Financial Analyst Tool Node**  
   - Type: LangChain Tool Workflow.  
   - Link to sub-workflow that aggregates Binance indicators and price/order book data.  
   - Inputs: `message` (symbol), `sessionId`.  

8. **Add News and Sentiment Analysis Request Node**  
   - Type: HTTP Request Tool.  
   - Method: POST.  
   - URL: `https://treasurium.app.n8n.cloud/webhook/newsanalyst`.  
   - Body parameter: `message` containing the symbol.  

9. **Add Binance Spot Market Quant AI Agent Node**  
   - Type: LangChain Agent Node.  
   - Configure system message instructing the agent’s role, connected tools, analysis scope, output formatting in Telegram HTML style, and strict no-fabrication rules (copy from overview).  
   - Inputs: `text` from Adds "SessionId".  
   - Connect AI Language Model, AI Memory, and AI Tools (Financial Analyst Tool, News and Sentiment) to the agent.  

10. **Add Code Node to Split Large Messages**  
    - Type: Code node with JavaScript to split strings > 4000 characters into chunks.  
    - Input: Agent output message.  

11. **Add Telegram Send Node**  
    - Type: Telegram node (Send Message).  
    - Configure to send to chat ID from Telegram Trigger.  
    - Set text to chunked messages with HTML formatting enabled.  
    - Connect splitter node → Telegram send node.  

12. **Connect Workflow Nodes**  
    - Telegram Trigger → User Authentication → Adds "SessionId" → Binance Spot Market Quant AI Agent → Split Message → Telegram Send.  
    - Connect OpenAI Chat Model, Simple Memory, Financial Analyst Tool, and News and Sentiment nodes as AI model/memory/tools inputs to the agent node.  

13. **Activate Workflow**  
    - Deploy and activate the workflow.  
    - Test by sending messages from the authorized Telegram user to trigger report generation.  

14. **Import and Activate Sub-Workflows**  
    - Import and activate all dependent workflows: Financial Analyst Tool, Binance SM Indicators Webhook Tool, News & Sentiment Analyst Webhook, Price/Kline/OrderBook Tool, Timeframe Indicator Tools (15m, 1h, 4h, 1d).  
    - Ensure webhook endpoints for each are active and reachable.  

---

# 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                               | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Full system documentation is available in the final sticky note node, describing setup, included workflows, installation steps, and output formatting.                                                                                                                                                   | See Sticky Note10 content in workflow description above.                                       |
| Telegram messages exceeding 4000 characters are automatically split to comply with Telegram limits.                                                                                                                                                                                                       | Sticky Note4.                                                                                   |
| The agent uses a multi-tool architecture: a financial analyst tool for multi-timeframe technical indicators and a news/sentiment tool for market sentiment and headlines.                                                                                                                                 | Sticky Notes 7 and 8.                                                                           |
| The workflow is protected by copyright and licensing; reuse or resale is prohibited without a license.                                                                                                                                                                                                    | Copyright and licensing note in sticky note10.                                                |
| The system requires multiple interconnected workflows to function properly, including Binance indicator tools for various timeframes and price/order book data.                                                                                                                                          | Included workflows list in sticky note10.                                                     |
| For professional use, replace the Telegram ID check with your own ID and secure OpenAI and Telegram credentials properly.                                                                                                                                                                               | User Authentication node and credentials notes.                                               |
| The system outputs reports formatted in Telegram HTML style, including bold headers and bullet points, to ensure readability and clarity in Telegram chats.                                                                                                                                              | Output format described in agent system message and sticky notes.                             |
| LinkedIn contact for support and further information: Don Jayamaha.                                                                                                                                                                                                                                      | [linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr)                        |

---

**Disclaimer:** The content analyzed and described here is extracted exclusively from an automated n8n workflow. It respects all current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.