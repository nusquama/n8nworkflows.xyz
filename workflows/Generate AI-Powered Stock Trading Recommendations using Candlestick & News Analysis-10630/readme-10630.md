Generate AI-Powered Stock Trading Recommendations using Candlestick & News Analysis

https://n8nworkflows.xyz/workflows/generate-ai-powered-stock-trading-recommendations-using-candlestick---news-analysis-10630


# Generate AI-Powered Stock Trading Recommendations using Candlestick & News Analysis

### 1. Workflow Overview

This workflow is designed to generate AI-powered stock trading recommendations by analyzing candlestick data at multiple time intervals combined with relevant news analysis. It integrates real-time market data and news feeds, processes the information through AI language models, and delivers actionable trading advice via Telegram messages.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Account Validation:** Receives user commands or triggers via Telegram and validates user access.
- **1.2 Market Data Collection:** Fetches candlestick data from three different intervals (1 min, 15 min, 1 hour) concurrently.
- **1.3 Data Aggregation & Preprocessing:** Merges and aggregates candlestick data for coherent analysis.
- **1.4 News Aggregation & AI Preprocessing:** Collects stock-related news and processes it through a language model.
- **1.5 AI Fusion & Recommendation Generation:** Combines processed market data and news insights, runs an AI agent to generate trading recommendations.
- **1.6 Message Dispatch:** Sends the AI-generated recommendations back to the user via Telegram.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Account Validation

- **Overview:**  
  This block listens for incoming Telegram messages, triggers workflow execution on user input, and checks user account validity to control access.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Account Check

- **Node Details:**

  - **Telegram Trigger**  
    - *Type & Role:* Trigger node for incoming Telegram messages.  
    - *Configuration:* Uses a specific webhook ID to receive Telegram updates. No additional filters configured.  
    - *Inputs:* External Telegram messages.  
    - *Outputs:* Passes Telegram data downstream for validation.  
    - *Edge Cases:* Possible failure if webhook is not properly configured or Telegram API is unreachable.  

  - **Account Check (Switch Node)**  
    - *Type & Role:* Switch node to validate user permissions or account state before proceeding.  
    - *Configuration:* Custom rules (not explicitly detailed) evaluate the Telegram message or user metadata to decide whether to continue.  
    - *Inputs:* Telegram Trigger output.  
    - *Outputs:* Four downstream paths directing to various interval data fetch nodes and news aggregator.  
    - *Edge Cases:* Misconfiguration can cause unauthorized access or block valid users; lack of fallback route could halt the workflow.

---

#### 2.2 Market Data Collection

- **Overview:**  
  Concurrently fetches candlestick data from three intervals (1 min, 15 min, 1 hour) to provide multi-timeframe market context.

- **Nodes Involved:**  
  - 1 min Interval  
  - 15 min Interval  
  - 1 hour Interval

- **Node Details:**

  - **1 min Interval, 15 min Interval, 1 hour Interval (HTTP Request Nodes)**  
    - *Type & Role:* HTTP request nodes to fetch candlestick data for respective intervals from an external market data API (details not specified).  
    - *Configuration:* API endpoints and parameters for respective timeframes (not fully detailed).  
    - *Inputs:* From Account Check switch node.  
    - *Outputs:* Each passes data to the Merge node for consolidation.  
    - *Edge Cases:* HTTP errors, rate limiting, API downtime, or malformed responses might occur.

---

#### 2.3 Data Aggregation & Preprocessing

- **Overview:**  
  Merges candlestick data received from different interval requests and aggregates the data to prepare for AI processing.

- **Nodes Involved:**  
  - Merge  
  - Aggregate  
  - Code in JavaScript

- **Node Details:**

  - **Merge**  
    - *Type & Role:* Merges inputs from the three interval nodes.  
    - *Configuration:* Default merge operation; presumably merges by waiting for all inputs before forwarding.  
    - *Inputs:* Outputs from 1 min, 15 min, and 1 hour Interval nodes.  
    - *Outputs:* Passes merged data to Aggregate.  
    - *Edge Cases:* If any input is missing or delayed, merging could stall.  

  - **Aggregate**  
    - *Type & Role:* Aggregates merged candlestick data; likely to compute summaries or combine data points.  
    - *Configuration:* Specific aggregation functions (e.g., sum, average) are not detailed.  
    - *Inputs:* From Merge node.  
    - *Outputs:* Passes aggregated data to Code in JavaScript node.  
    - *Edge Cases:* Improper aggregation logic could skew data; empty input sets may cause errors.  

  - **Code in JavaScript**  
    - *Type & Role:* Custom JavaScript code for additional data processing or formatting.  
    - *Configuration:* User-defined code (not shown) presumably refines aggregated candlestick data for AI input.  
    - *Inputs:* From Aggregate node.  
    - *Outputs:* Passes processed data to Merge1 node for fusion with news data.  
    - *Edge Cases:* Syntax errors, runtime exceptions, or invalid data formats could break the flow.

---

#### 2.4 News Aggregation & AI Preprocessing

- **Overview:**  
  Fetches relevant news articles, sends them to a language model for interpretation, and merges the output with processed market data.

- **Nodes Involved:**  
  - News Aggregator  
  - Message a model  
  - Merge1

- **Node Details:**

  - **News Aggregator (HTTP Request)**  
    - *Type & Role:* Fetches recent news articles related to stocks from an external news API.  
    - *Configuration:* API endpoint and parameters not detailed; assumed to filter news by topic or keywords.  
    - *Inputs:* From Account Check node.  
    - *Outputs:* Passes news data to Message a model node.  
    - *Edge Cases:* API failures, empty news feeds, or rate limits.  

  - **Message a model (Google Gemini language model)**  
    - *Type & Role:* Sends news data to Google Gemini AI for semantic analysis and summarization.  
    - *Configuration:* Uses Google Gemini credentials (not shown). Input is news articles; output is AI-processed insights.  
    - *Inputs:* From News Aggregator.  
    - *Outputs:* Connects to Merge1 node to combine with candlestick data.  
    - *Edge Cases:* Authentication failures, model timeouts, or malformed inputs.  

  - **Merge1**  
    - *Type & Role:* Merges AI-processed news data with JavaScript-processed candlestick data.  
    - *Configuration:* Two inputs expected: one from Code in JavaScript node (candlestick data), one from Message a model node (news analysis).  
    - *Inputs:* From Code in JavaScript and Message a model.  
    - *Outputs:* Passes combined data to Aggregate1.  
    - *Edge Cases:* If either input is missing or delayed, merging may fail or stall.

---

#### 2.5 AI Fusion & Recommendation Generation

- **Overview:**  
  Aggregates combined market and news data, runs an AI Agent to generate trading recommendations, and manages language model interactions.

- **Nodes Involved:**  
  - Aggregate1  
  - AI Agent  
  - Google Gemini Chat Model

- **Node Details:**

  - **Aggregate1**  
    - *Type & Role:* Aggregates merged data to prepare a unified input for the AI Agent.  
    - *Configuration:* Specific aggregation logic not detailed; likely formats and consolidates data.  
    - *Inputs:* From Merge1 node.  
    - *Outputs:* Passes data to AI Agent node.  
    - *Edge Cases:* Empty or malformed data could cause AI Agent input issues.  

  - **AI Agent (Langchain Agent)**  
    - *Type & Role:* Executes AI logic to synthesize data and produce stock trading recommendations.  
    - *Configuration:* Uses Langchain framework; connected to Google Gemini Chat Model for language generation.  
    - *Inputs:* From Aggregate1 node.  
    - *Outputs:* Sends recommendation text to Telegram messaging node.  
    - *Version Requirements:* Version 2.2 of AI Agent node used.  
    - *Edge Cases:* AI logic failures, response timeouts, or API quota exhaustion.  

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model backend for the AI Agent, providing advanced chat-based natural language processing.  
    - *Configuration:* Authenticated with Google Gemini credentials. Acts as language model provider to AI Agent.  
    - *Inputs:* Connected as AI language model for AI Agent node.  
    - *Outputs:* Model responses routed internally to AI Agent.  
    - *Edge Cases:* Authentication, rate limits, or model unavailability.

---

#### 2.6 Message Dispatch

- **Overview:**  
  Final block sends the AI-generated stock trading recommendations as text messages to the user via Telegram.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**

  - **Send a text message (Telegram Node)**  
    - *Type & Role:* Sends text message to Telegram users.  
    - *Configuration:* Uses Telegram credentials and webhook ID. Message content is AI Agent output.  
    - *Inputs:* From AI Agent node.  
    - *Outputs:* None (terminal node).  
    - *Edge Cases:* Telegram API issues, invalid chat IDs, or message size limits.

---

### 3. Summary Table

| Node Name              | Node Type                        | Functional Role                         | Input Node(s)                                   | Output Node(s)                 | Sticky Note                          |
|------------------------|---------------------------------|---------------------------------------|------------------------------------------------|-------------------------------|------------------------------------|
| Telegram Trigger       | Telegram Trigger                | Initiates workflow on Telegram input  | -                                              | Account Check                 |                                    |
| Account Check          | Switch                         | Validates user account/access          | Telegram Trigger                               | 1 min Interval, 15 min Interval, 1 hour Interval, News Aggregator |                                    |
| 1 min Interval         | HTTP Request                   | Fetches 1 min interval candlestick data | Account Check                                  | Merge                        |                                    |
| 15 min Interval        | HTTP Request                   | Fetches 15 min interval candlestick data | Account Check                                  | Merge                        |                                    |
| 1 hour Interval        | HTTP Request                   | Fetches 1 hour interval candlestick data | Account Check                                  | Merge                        |                                    |
| Merge                  | Merge                         | Merges candlestick data from intervals | 1 min Interval, 15 min Interval, 1 hour Interval | Aggregate                   |                                    |
| Aggregate              | Aggregate                     | Aggregates merged candlestick data    | Merge                                           | Code in JavaScript           |                                    |
| Code in JavaScript     | Code                          | Processes aggregated data via JS code | Aggregate                                        | Merge1                       |                                    |
| News Aggregator        | HTTP Request                   | Fetches stock-related news             | Account Check                                   | Message a model              |                                    |
| Message a model        | Google Gemini Language Model  | Analyzes news via AI                   | News Aggregator                                 | Merge1                       |                                    |
| Merge1                 | Merge                         | Combines processed market & news data | Code in JavaScript, Message a model             | Aggregate1                   |                                    |
| Aggregate1             | Aggregate                     | Aggregates combined data for AI Agent | Merge1                                          | AI Agent                    |                                    |
| AI Agent               | Langchain Agent               | Generates trading recommendations     | Aggregate1                                       | Send a text message          |                                    |
| Google Gemini Chat Model| Google Gemini LM              | Language model backend for AI Agent   | Connected internally to AI Agent                 | AI Agent (internal)          |                                    |
| Send a text message    | Telegram                      | Sends AI recommendations to user      | AI Agent                                         | -                           |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with Telegram Bot credentials and webhook ID to listen for incoming messages.

2. **Add Account Check Node (Switch)**  
   - Type: Switch  
   - Connect Telegram Trigger output to this node.  
   - Configure rules to validate user permissions based on Telegram user data.

3. **Create HTTP Request Nodes for Market Data**  
   - Create three nodes named: "1 min Interval", "15 min Interval", "1 hour Interval".  
   - For each, configure HTTP Request with appropriate API endpoints and parameters to fetch candlestick data for respective time intervals.  
   - Connect each from the corresponding output branch of Account Check node.

4. **Create Merge Node for Candlestick Data**  
   - Type: Merge  
   - Connect outputs of all three HTTP Request interval nodes to this Merge node.

5. **Create Aggregate Node for Merging Data**  
   - Type: Aggregate  
   - Connect Merge node output to Aggregate node.  
   - Configure aggregation logic (e.g., compute averages, totals) relevant to candlestick data.

6. **Add Code Node (JavaScript)**  
   - Type: Code  
   - Connect Aggregate output to this node.  
   - Write JavaScript code to further process or format aggregated data for AI input.

7. **Create News Aggregator HTTP Request Node**  
   - Type: HTTP Request  
   - Connect from Account Check node.  
   - Configure API request to fetch recent stock-related news articles.

8. **Create Message a model Node (Google Gemini LLM)**  
   - Type: Langchain Google Gemini node  
   - Connect News Aggregator output to this node.  
   - Configure with Google Gemini credentials.  
   - Input: news articles; Output: AI-processed news insights.

9. **Add Merge Node to Combine Market and News Data**  
   - Type: Merge  
   - Connect Code node output and Message a model output to this node.

10. **Create Aggregate Node for Combined Data**  
    - Type: Aggregate  
    - Connect Merge node output to this node.  
    - Aggregate combined data as needed for AI Agent input.

11. **Add AI Agent Node**  
    - Type: Langchain Agent (version 2.2)  
    - Connect Aggregate node output.  
    - Configure AI Agent with appropriate prompt, tools, and logic to generate trading recommendations.

12. **Add Google Gemini Chat Model Node**  
    - Type: Langchain Google Gemini Chat Model  
    - Connect as language model backend for AI Agent node.  
    - Configure with Google Gemini credentials.

13. **Create Telegram Send Message Node**  
    - Type: Telegram  
    - Connect AI Agent output to this node.  
    - Configure with Telegram Bot credentials to send messages back to users.  
    - Set message content to AI Agent's generated text.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                           |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------|
| Workflow leverages Google Gemini language models integrated via Langchain nodes for advanced AI processing.  | Requires Google Gemini credentials setup |
| Telegram integration requires Bot token and webhook configuration.                                            | Telegram Bot API documentation           |
| Candlestick data API details and credentials need to be configured externally; ensure API permits multiple interval queries. | Depends on chosen stock data provider     |
| Ensure rate limits and error handling are implemented on HTTP Request nodes to avoid workflow stalls.          | n8n HTTP Request node best practices     |

---

This completes the detailed analysis and reference documentation for the "Generate AI-Powered Stock Trading Recommendations using Candlestick & News Analysis" workflow.