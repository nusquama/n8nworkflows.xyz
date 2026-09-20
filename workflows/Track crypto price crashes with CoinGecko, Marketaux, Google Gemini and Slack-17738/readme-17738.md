Track crypto price crashes with CoinGecko, Marketaux, Google Gemini and Slack

https://n8nworkflows.xyz/workflows/track-crypto-price-crashes-with-coingecko--marketaux--google-gemini-and-slack-17738


# Track crypto price crashes with CoinGecko, Marketaux, Google Gemini and Slack

### 1. Workflow Overview

This workflow is designed to automate cryptocurrency market monitoring by tracking price movements against predefined crash thresholds. Operating on a recurring schedule, it evaluates digital assets using market data providers, assesses volatility conditions, contextualizes price changes with real-time news headlines, synthesizes the information via an AI model, and posts structured alerts directly to Slack.

The logic is grouped into six functional blocks:
- **1.1 Schedule and Configuration Initialization:** Triggers the pipeline on a fixed schedule and sets parameters for the target asset and evaluation criteria.
- **1.2 Price Retrieval and Analysis:** Fetches current spot prices and 24-hour historical market charts, computes key performance metrics, and determines whether crash thresholds have been breached.
- **1.3 Conditional Evaluation:** Evaluates whether market conditions warrant deeper analysis by filtering out non-critical fluctuations.
- **1.4 News Retrieval and Extraction:** Pulls recent news articles for the target asset ticker from an external media API and formats them for downstream consumption.
- **1.5 AI Market Analysis:** Uses a Google Gemini language model paired with a structured output parser to generate a factual market assessment, sentiment categorization, key drivers, and recommendations.
- **1.6 Notification Dispatch:** Formats the compiled metrics, AI assessment, and top headlines into a readable markdown message and posts it to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Configuration Initialization
- **Overview:** Initializes the automated pipeline on a fixed time interval and defines the static parameters required for API calls and threshold evaluations.
- **Nodes Involved:** `Every 4 Hours Trigger`, `Set Asset Parameters`
- **Node Details:**
  - **Every 4 Hours Trigger**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger) — Acts as the primary execution entry point.
    - *Configuration:* Interval set to fire every `4` hours.
    - *Input/Output:* No inputs; outputs execution timestamp to the configuration node.
    - *Edge Cases:* Ensure n8n instance time zones match expected execution cadences.
  - **Set Asset Parameters**
    - *Type and Role:* `n8n-nodes-base.set` (Edit Fields) — Defines asset-specific properties used throughout the workflow.
    - *Configuration:* Sets manual string variables: `Asset` (`bitcoin`), `Ticker` (`BTC`), `Crash_threshold` (`65000`), `Percent_drop_threshold` (`5`), and `News_limit` (`15`).
    - *Input/Output:* Input from `Every 4 Hours Trigger`; outputs parameters to `Fetch Current Asset Price`.
    - *Edge Cases:* Ensure `Asset` matches CoinGecko’s slug format precisely.

#### 2.2 Price Retrieval and Analysis
- **Overview:** Queries CoinGecko for live pricing and historical trend data, then computes statistical metrics (such as 24-hour percentage change and drop from high) to evaluate crash conditions.
- **Nodes Involved:** `Fetch Current Asset Price`, `Fetch 24h Market Data`, `Calculate Crash Severity`, `If Crash Threshold Met`
- **Node Details:**
  - **Fetch Current Asset Price**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Retrieves real-time price metrics in USD.
    - *Configuration:* GET request to `https://api.coingecko.com/api/v3/simple/price?vs_currencies=usd&ids={{ $json.Asset }}` using generic HTTP Header Authentication.
    - *Input/Output:* Input from `Set Asset Parameters`; outputs current price JSON to `Fetch 24h Market Data`.
    - *Credentials Required:* CoinGecko (`httpHeaderAuth`).
    - *Edge Cases:* Rate limiting (HTTP 429) if queried too frequently without a paid tier.
  - **Fetch 24h Market Data**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Pulls 24-hour historical market chart arrays.
    - *Configuration:* GET request to `https://api.coingecko.com/api/v3/coins/{{ $('Set Asset Parameters').item.json.Asset }}/market_chart` with query parameters `vs_currency=usd` and `days=1`.
    - *Input/Output:* Input from `Fetch Current Asset Price`; outputs historical pricing series to `Calculate Crash Severity`.
    - *Credentials Required:* CoinGecko (`httpHeaderAuth`).
  - **Calculate Crash Severity**
    - *Type and Role:* `n8n-nodes-base.code` (Code) — Executes JavaScript to compute period highs, lows, 24-hour change, and drop from high percentages. Determines crash state based on thresholds.
    - *Configuration:* Contains custom JavaScript parsing `prices` arrays, applying downsampling on normal days or passing full raw points during crash states.
    - *Key Expressions:* Accesses preceding node data via `$('Fetch Current Asset Price')` and `$('Set Asset Parameters')`.
    - *Input/Output:* Input from `Fetch 24h Market Data`; outputs analytical summary object to `If Crash Threshold Met`.
    - *Edge Cases:* Throws an error if the CoinGecko array returns empty or undefined.
  - **If Crash Threshold Met**
    - *Type and Role:* `n8n-nodes-base.if` (If) — Gates workflow execution based on the evaluated crash boolean.
    - *Configuration:* Evaluates condition where `{{ $json.is_crash }}` equals `true`.
    - *Input/Output:* Input from `Calculate Crash Severity`; outputs to `Fetch Market News` on true condition.

#### 2.3 News Retrieval and Extraction
- **Overview:** Pulls recent news headlines associated with the asset ticker from Marketaux and parses them into structured text.
- **Nodes Involved:** `Fetch Market News`, `Extract News Articles`
- **Node Details:**
  - **Fetch Market News**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Retrieves recent media publications matching the ticker.
    - *Configuration:* GET request to `https://api.marketaux.com/v1/news/all` with query parameters `symbols` linked to the asset ticker and `language` set to `en`. Uses generic HTTP Query Authentication.
    - *Input/Output:* Input from `If Crash Threshold Met` (true branch); outputs raw news payload to `Extract News Articles`.
    - *Credentials Required:* Marketaux (`httpQueryAuth`).
    - *Edge Cases:* Empty search results returned if no recent articles exist for the ticker.
  - **Extract News Articles**
    - *Type and Role:* `n8n-nodes-base.code` (Code) — Truncates article descriptions and constructs a formatted summary string.
    - *Configuration:* Limits articles based on configuration parameters (capped at a maximum of 5) and formats titles and sources for downstream AI ingestion.
    - *Input/Output:* Input from `Fetch Market News`; outputs structured articles and formatted title strings to `Market Analyst Agent`.

#### 2.4 AI Market Analysis
- **Overview:** Synthesizes price metrics and news headlines using a Google Gemini model, enforcing a strict JSON output schema via a structured output parser.
- **Nodes Involved:** `Google Gemini Model`, `Define Output Schema`, `Market Analyst Agent`
- **Node Details:**
  - **Google Gemini Model**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Google Gemini Chat Model) — Language model provider configuration.
    - *Configuration:* Model set to `models/gemini-3.1-flash-lite`.
    - *Credentials Required:* Google PaLM API (`googlePalmApi`).
    - *Input/Output:* Connected to `Market Analyst Agent` via AI language model connection.
  - **Define Output Schema**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser) — Enforces a strict JSON response schema from the LLM.
    - *Configuration:* Manual JSON schema defining required properties: `headline` (string), `severity` (enum: `watch`, `moderate`, `severe`), `summary` (string), `key_drivers` (array of strings, max 3), and `recommendation` (string).
    - *Input/Output:* Connected to `Market Analyst Agent` via AI output parser connection.
  - **Market Analyst Agent**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Orchestrates the analysis using price data metrics and formatted news headlines.
    - *Configuration:* System message enforces a factual, concise persona. Prompt uses dynamic variable injection referencing previous node values.
    - *Input/Output:* Inputs from `Extract News Articles` and `Calculate Crash Severity`; outputs structured JSON response to `Prepare Slack Message`.

#### 2.5 Notification Dispatch
- **Overview:** Formats the compiled analytics and AI assessment into a markdown-enabled Slack message and posts it to the designated notification channel.
- **Nodes Involved:** `Prepare Slack Message`, `Post Message to Slack`
- **Node Details:**
  - **Prepare Slack Message**
    - *Type and Role:* `n8n-nodes-base.set` (Edit Fields) — Compiles metrics, AI evaluation, and news articles into a single Slack-ready markdown string.
    - *Configuration:* Maps expression variables from `Calculate Crash Severity`, `Market Analyst Agent` output, and `Extract News Articles`.
    - *Input/Output:* Input from `Market Analyst Agent`; outputs `slack_message` field to `Post Message to Slack`.
  - **Post Message to Slack**
    - *Type and Role:* `n8n-nodes-base.slack` (Slack) — Sends the final notification to Slack.
    - *Configuration:* Action set to post text to a channel (`channelId`: `C0B8VH1M5PX`, cached name `general`) with markdown enabled (`mrkdwn: true`).
    - *Credentials Required:* Slack account (`slackApi`).
    - *Input/Output:* Input from `Prepare Slack Message`; workflow termination point.
    - *Edge Cases:* Missing channel permissions or invalid user/bot tokens.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation wrapper and overview | None | None | ## Automated Crypto Coin Crash Tracker<br><br>### How it works<br><br>1. The workflow triggers on a schedule to monitor digital assets.<br>2. Price data is fetched and analyzed against crash thresholds.<br>3. Recent market news is retrieved if volatility is detected.<br>4. An AI agent synthesizes the price and news data into insights.<br>5. A summary is sent to Slack as an alert.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with your desired monitoring interval.<br>- Update the Config node with your target asset, ticker, and threshold percentages.<br>- Set up HTTP Request credentials for CoinGecko and Marketaux APIs.<br>- Connect your Google Gemini credentials for the AI Market Analyst agent.<br>- Configure your Slack credentials and target channel for notifications.<br><br>### Customization<br><br>Adjust the crash thresholds in the Config node or modify the AI prompt to change the depth of market analysis. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Section note for schedule and config | None | None | ## Schedule and configuration setup<br><br>Triggers the workflow and initializes configuration parameters. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Section note for price analysis | None | None | ## Fetch and analyze price data<br><br>Analyzes metrics to evaluate crash thresholds. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Section note for market news | None | None | ## Fetch and extract market news<br><br>Retrieves recent news for AI analysis. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Section note for AI agent | None | None | ## AI market analyst agent<br><br>Uses Gemini to formulate market insights. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Section note for Slack alert | None | None | ## Format and send Slack alert<br><br>Sends formatted notification to Slack channel. |
| Every 4 Hours Trigger | `n8n-nodes-base.scheduleTrigger` | Triggers execution every 4 hours | None | Set Asset Parameters | |
| Set Asset Parameters | `n8n-nodes-base.set` | Assigns static asset and threshold config | Every 4 Hours Trigger | Fetch Current Asset Price | |
| Fetch Current Asset Price | `n8n-nodes-base.httpRequest` | Fetches spot price from CoinGecko | Set Asset Parameters | Fetch 24h Market Data | |
| Fetch 24h Market Data | `n8n-nodes-base.httpRequest` | Fetches 24-hour market chart from CoinGecko | Fetch Current Asset Price | Calculate Crash Severity | |
| Calculate Crash Severity | `n8n-nodes-base.code` | Computes price statistics and crash status | Fetch 24h Market Data | If Crash Threshold Met | |
| If Crash Threshold Met | `n8n-nodes-base.if` | Evaluates if a crash condition occurred | Calculate Crash Severity | Fetch Market News | |
| Prepare Slack Message | `n8n-nodes-base.set` | Formats final markdown string for Slack | Market Analyst Agent | Post Message to Slack | |
| Post Message to Slack | `n8n-nodes-base.slack` | Posts alert to the Slack channel | Prepare Slack Message | None | |
| Google Gemini Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides Gemini LLM backend | None | Market Analyst Agent | |
| Define Output Schema | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces structured JSON output schema | None | Market Analyst Agent | |
| Fetch Market News | `n8n-nodes-base.httpRequest` | Retrieves news articles from Marketaux | If Crash Threshold Met | Extract News Articles | |
| Market Analyst Agent | `@n8n/n8n-nodes-langchain.agent` | Synthesizes price and news data via AI | Google Gemini Model, Define Output Schema, Extract News Articles | Prepare Slack Message | |
| Extract News Articles | `n8n-nodes-base.code` | Parses and truncates news article feeds | Fetch Market News | Market Analyst Agent | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node. Set the interval to every `4` hours.
2. **Configure Asset Parameters:**
   - Add an **Edit Fields (Set)** node connected to the trigger.
   - Define the following string fields:
     - `Asset`: `bitcoin`
     - `Ticker`: `BTC`
     - `Crash_threshold`: `65000`
     - `Percent_drop_threshold`: `5`
     - `News_limit`: `15`
3. **Fetch Current Price:**
   - Add an **HTTP Request** node. Connect it to the Set node.
   - Method: `GET`
   - URL: `https://api.coingecko.com/api/v3/simple/price?vs_currencies=usd&ids={{ $json.Asset }}`
   - Authentication: Generic Credential Type -> HTTP Header Auth (Credential name: `CoinGecko`).
4. **Fetch 24h Market Data:**
   - Add an **HTTP Request** node. Connect it to the previous HTTP node.
   - Method: `GET`
   - URL: `https://api.coingecko.com/api/v3/coins/{{ $('Set Asset Parameters').item.json.Asset }}/market_chart`
   - Query Parameters: `vs_currency` = `usd`, `days` = `1`.
   - Authentication: HTTP Header Auth.
5. **Calculate Crash Severity:**
   - Add a **Code** node. Connect it to the 24h market data node.
   - Insert JavaScript to compute 24-hour highs, lows, percentage drops, and determine `is_crash` based on configured thresholds.
6. **Evaluate Crash Condition:**
   - Add an **If** node. Connect it to the Code node.
   - Set condition to evaluate if `{{ $json.is_crash }}` equals `true`.
7. **Fetch Market News:**
   - Add an **HTTP Request** node connected to the `true` branch of the If node.
   - URL: `https://api.marketaux.com/v1/news/all`
   - Query Parameters: `symbols` = `{{ $('Set Asset Parameters').item.json.Ticker }}`, `language` = `en`.
   - Authentication: Generic Credential Type -> HTTP Query Auth (Credential name: `marketaux`).
8. **Extract News Articles:**
   - Add a **Code** node connected to the news HTTP node.
   - Insert JavaScript to slice items up to the news limit, truncate article descriptions to 200 characters, and generate a formatted titles string.
9. **Configure AI Model & Parser:**
   - Add a **Google Gemini Chat Model** node (`models/gemini-3.1-flash-lite`) using `googlePalmApi` credentials.
   - Add a **Structured Output Parser** node and define the manual JSON schema specifying `headline`, `severity`, `summary`, `key_drivers`, and `recommendation`.
10. **Set Up Market Analyst Agent:**
    - Add an **AI Agent** node.
    - Connect the Google Gemini Model to its `ai_languageModel` input and the Structured Output Parser to its `ai_outputParser` input.
    - Connect the `Extract News Articles` node to its main input.
    - Configure prompt instructions and system messages instructing the agent to evaluate the price move using the injected metrics and news headlines.
11. **Prepare Slack Message:**
    - Add an **Edit Fields (Set)** node connected to the AI Agent.
    - Create a `slack_message` string field combining emoji indicators, price metrics, structured AI assessment properties, and formatted news articles.
12. **Post to Slack:**
    - Add a **Slack** node connected to the Set node.
    - Resource: `Message` -> Action: `Post`.
    - Select channel and set text to `={{ $json.slack_message }}` with `mrkdwn` enabled. Configure Slack OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Automated Crypto Coin Crash Tracker Workflow | Built with n8n for monitoring digital assets using CoinGecko, Marketaux, Google Gemini, and Slack. |