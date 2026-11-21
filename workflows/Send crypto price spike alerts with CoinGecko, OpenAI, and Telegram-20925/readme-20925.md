Send crypto price spike alerts with CoinGecko, OpenAI, and Telegram

https://n8nworkflows.xyz/workflows/send-crypto-price-spike-alerts-with-coingecko--openai--and-telegram-20925


# Send crypto price spike alerts with CoinGecko, OpenAI, and Telegram

### 1. Workflow Overview

This workflow automates the monitoring of cryptocurrency market movements, using the public CoinGecko API to check the top 30 coins by market cap every 6 hours. It identifies coins experiencing significant 24-hour price swings beyond a pre-configured percentage threshold, generates professional analyst commentary using OpenAI (`gpt-4o-mini`), and broadcasts a structured Markdown report to a designated Telegram chat.

The logic is divided into four sequential functional blocks:
- **1.1 Triggers and Configuration:** Handles automated periodic scheduling or manual execution, storing global parameters like currency, threshold, and chat ID.
- **1.2 Fetch Market Data:** Retrieves current market statistics from the external CoinGecko API without requiring an API key.
- **1.3 Spike Detection and AI Processing:** Filters and ranks anomalous movers, compiles a prompt, and queries OpenAI for contextual market insights.
- **1.4 Formatting and Delivery:** Synthesizes the AI analysis and sorted market data into a clean Telegram message and dispatches it.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers and Configuration
- **Overview:** Initializes the execution flow either automatically on a schedule or manually for debugging, while centralizing runtime configuration settings.
- **Nodes Involved:** `Price Check Schedule`, `Manual Test Trigger`, `Configuration`

##### Node Details:
- **Price Check Schedule**
  - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) - Initiates workflow execution periodically.
  - *Configuration:* Set to trigger every 6 hours.
  - *Inputs / Outputs:* None input; outputs connection to `Configuration`.
  - *Potential Failures:* None typical; dependent on n8n instance uptime.

- **Manual Test Trigger**
  - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (v1) - Allows users to manually run the workflow on demand.
  - *Configuration:* Default execution parameters.
  - *Inputs / Outputs:* None input; outputs connection to `Configuration`.
  - *Potential Failures:* None.

- **Configuration**
  - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Acts as a global data store for workflow constants and toggles.
  - *Configuration:* Defines properties: `alertTitle` ("Crypto Movers Alert"), `vsCurrency` ("usd"), `perPage` (30), `spikeThresholdPercent` (5), `maxAlerts` (8), and `telegramChatId` ("YOUR_TELEGRAM_CHAT_ID").
  - *Key Expressions:* Outputs explicit string and numeric data parameters utilized downstream.
  - *Inputs / Outputs:* Inputs from `Price Check Schedule` or `Manual Test Trigger`; outputs to `Fetch Market Data`.
  - *Potential Failures:* Missing or unupdated placeholders (e.g., chat ID) will cause downstream delivery errors.

---

#### 1.2 Fetch Market Data
- **Overview:** Queries the CoinGecko public API to pull the latest pricing and market cap rankings using parameters passed from the configuration step.
- **Nodes Involved:** `Fetch Market Data`

##### Node Details:
- **Fetch Market Data**
  - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) - Performs a GET request to an external REST API endpoint.
  - *Configuration:* Endpoint set to `https://api.coingecko.com/api/v3/coins/markets`. Timeout configured to 20,000ms. Query parameters dynamically bind to configuration variables (`vs_currency`, `per_page`, `order=market_cap_desc`, `page=1`, `price_change_percentage=24h`).
  - *Key Expressions:* 
    - `={{ $json.vsCurrency }}`
    - `={{ $json.perPage }}`
  - *Inputs / Outputs:* Input from `Configuration`; output to `Detect Price Spikes`.
  - *Potential Failures:* Rate limiting by CoinGecko (HTTP 429) if triggered too frequently, or network timeouts.

---

#### 1.3 Spike Detection and AI Processing
- **Overview:** Parses the market data array, filters tokens exceeding the volatility threshold, formats an AI prompt, and queries OpenAI to generate natural-language market commentary.
- **Nodes Involved:** `Detect Price Spikes`, `Build AI Prompt`, `AI Market Take`

##### Node Details:
- **Detect Price Spikes**
  - *Type and Technical Role:* `n8n-nodes-base.code` (v2) - Executes custom JavaScript to process, filter, and sort array data.
  - *Configuration:* Extracts configuration thresholds dynamically from the upstream node, filters coins based on absolute 24h percentage change, sorts movers by magnitude, and slices up to `maxAlerts`.
  - *Key Expressions / Variables:* Pulls `$('Configuration').first().json` to access thresholds.
  - *Inputs / Outputs:* Input from `Fetch Market Data`; output to `Build AI Prompt`.
  - *Potential Failures:* Malformed API response structures if CoinGecko alters its payload schema.

- **Build AI Prompt**
  - *Type and Technical Role:* `n8n-nodes-base.code` (v2) - Synthesizes structured JSON items into an aggregated string format suitable for LLM consumption.
  - *Configuration:* Maps processed movers into newline-delimited text blocks containing name, symbol, price, and 24h performance.
  - *Inputs / Outputs:* Input from `Detect Price Spikes`; output to `AI Market Take`.
  - *Potential Failures:* Empty inputs if no coins exceed the threshold (though workflow proceeds unless filtered upstream).

- **AI Market Take**
  - *Type and Technical Role:* `n8n-nodes-base.openAi` (v1.1) - Interfaces with the OpenAI Chat Completions API.
  - *Configuration:* Uses model `gpt-4o-mini` with operation `complete`. System prompt defines the persona as a plain-language crypto market analyst.
  - *Key Expressions:* 
    - `={{ 'These coins moved sharply in the last 24 hours. Write 3 sharp one-line takes on what might be driving these moves.\n\n' + $json.moversText }}`
  - *Inputs / Outputs:* Input from `Build AI Prompt`; output to `Format Telegram Alert`.
  - *Credentials Required:* OpenAI API Key.
  - *Potential Failures:* API authentication errors, credit exhaustion, or model response timeouts.

---

#### 1.4 Formatting and Delivery
- **Overview:** Merges AI-generated insights with raw market metrics into a Markdown-formatted message payload and sends it via Telegram.
- **Nodes Involved:** `Format Telegram Alert`, `Send Telegram Alert`

##### Node Details:
- **Format Telegram Alert**
  - *Type and Technical Role:* `n8n-nodes-base.code` (v2) - JavaScript-based transformer combining outputs from multiple upstream nodes into a final delivery object.
  - *Configuration:* Pulls raw tokens from `Detect Price Spikes` and the LLM response from `AI Market Take`, appending directional indicators (`^`/`v`) and current dates.
  - *Key Expressions / Variables:* 
    - `$('Detect Price Spikes').all()`
    - `$('Configuration').first().json`
  - *Inputs / Outputs:* Input from `AI Market Take`; output to `Send Telegram Alert`.
  - *Potential Failures:* Syntax formatting errors or missing choice keys from the OpenAI response.

- **Send Telegram Alert**
  - *Type and Technical Role:* `n8n-nodes-base.telegram` (v1.2) - Sends text notifications through a Telegram bot.
  - *Configuration:* Uses Markdown parse mode for message styling.
  - *Key Expressions:*
    - Text: `={{ $json.message }}`
    - Chat ID: `={{ $('Configuration').first().json.telegramChatId }}`
  - *Inputs / Outputs:* Input from `Format Telegram Alert`; no downstream outputs.
  - *Credentials Required:* Telegram Bot Token.
  - *Potential Failures:* Invalid chat ID, incorrect bot token permissions, or message length constraints exceeded.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Workflow documentation and configuration guide | None | None | Crypto Price Spike Alerts Watch the crypto market around the clock. When a top coin moves sharply, AI writes the take and Telegram delivers the alert. ... |
| Sticky Note 1 | n8n-nodes-base.stickyNote | Visual grouping for triggers and configuration | None | None | ### 1 - Triggers and configuration The schedule checks prices every 6 hours. Use the manual trigger to test any time. Edit every setting in the Configuration node. |
| Price Check Schedule | n8n-nodes-base.scheduleTrigger | Triggers the workflow automatically every 6 hours | None | Configuration | ### 1 - Triggers and configuration The schedule checks prices every 6 hours. Use the manual trigger to test any time. Edit every setting in the Configuration node. |
| Manual Test Trigger | n8n-nodes-base.manualTrigger | Allows manual execution for testing | None | Configuration | ### 1 - Triggers and configuration The schedule checks prices every 6 hours. Use the manual trigger to test any time. Edit every setting in the Configuration node. |
| Configuration | n8n-nodes-base.set | Stores global parameters and settings | Price Check Schedule, Manual Test Trigger | Fetch Market Data | ### 1 - Triggers and configuration The schedule checks prices every 6 hours. Use the manual trigger to test any time. Edit every setting in the Configuration node. |
| Sticky Note 2 | n8n-nodes-base.stickyNote | Visual grouping for fetching market data | None | None | ### 2 - Fetch market data Pulls the top coins by market cap from the free CoinGecko API, including 24h price change. No API key required. |
| Fetch Market Data | n8n-nodes-base.httpRequest | Pulls coin pricing data from CoinGecko API | Configuration | Detect Price Spikes | ### 2 - Fetch market data Pulls the top coins by market cap from the free CoinGecko API, including 24h price change. No API key required. |
| Sticky Note 3 | n8n-nodes-base.stickyNote | Visual grouping for spike detection and AI processing | None | None | ### 3 - Detect spikes and AI take Flags coins that moved beyond your threshold, ranks them, then OpenAI writes a sharp analyst take. |
| Detect Price Spikes | n8n-nodes-base.code | Filters and ranks coins based on price volatility | Fetch Market Data | Build AI Prompt | ### 3 - Detect spikes and AI take Flags coins that moved beyond your threshold, ranks them, then OpenAI writes a sharp analyst take. |
| Build AI Prompt | n8n-nodes-base.code | Formats mover data into an AI prompt string | Detect Price Spikes | AI Market Take | ### 3 - Detect spikes and AI take Flags coins that moved beyond your threshold, ranks them, then OpenAI writes a sharp analyst take. |
| AI Market Take | n8n-nodes-base.openAi | Generates analyst take via OpenAI GPT-4o-mini | Build AI Prompt | Format Telegram Alert | ### 3 - Detect spikes and AI take Flags coins that moved beyond your threshold, ranks them, then OpenAI writes a sharp analyst take. |
| Sticky Note 4 | n8n-nodes-base.stickyNote | Visual grouping for message formatting and delivery | None | None | ### 4 - Format and deliver Builds a Markdown alert with the AI take and the movers list, then sends it to your Telegram chat. |
| Format Telegram Alert | n8n-nodes-base.code | Compiles AI output and market data into Markdown | AI Market Take | Send Telegram Alert | ### 4 - Format and deliver Builds a Markdown alert with the AI take and the movers list, then sends it to your Telegram chat. |
| Send Telegram Alert | n8n-nodes-base.telegram | Delivers formatted alert to Telegram chat | Format Telegram Alert | None | ### 4 - Format and deliver Builds a Markdown alert with the AI take and the movers list, then sends it to your Telegram chat. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Triggers and Configuration Nodes:**
   - Add a **Schedule Trigger** node named `Price Check Schedule`. Set interval rules to run every 6 hours.
   - Add a **Manual Trigger** node named `Manual Test Trigger`.
   - Add an **Edit Fields (Set)** node named `Configuration`. Configure string and numeric assignments: `alertTitle` ("Crypto Movers Alert"), `vsCurrency` ("usd"), `perPage` (30), `spikeThresholdPercent` (5), `maxAlerts` (8), and `telegramChatId` ("YOUR_TELEGRAM_CHAT_ID"). Connect both triggers to this node.

2. **Set Up Market Data Retrieval:**
   - Add an **HTTP Request** node named `Fetch Market Data`.
   - Set Method to `GET`, URL to `https://api.coingecko.com/api/v3/coins/markets`.
   - Add query parameters: `vs_currency` (`={{ $json.vsCurrency }}`), `order` (`market_cap_desc`), `per_page` (`={{ $json.perPage }}`), `page` (`1`), `price_change_percentage` (`24h`).
   - Connect `Configuration` output to `Fetch Market Data`.

3. **Implement Spike Detection Logic:**
   - Add a **Code** node named `Detect Price Spikes`. Paste JavaScript code to parse items, filter by `spikeThresholdPercent`, sort by absolute performance, and slice by `maxAlerts`.
   - Connect `Fetch Market Data` to `Detect Price Spikes`.

4. **Configure AI Prompt Construction and OpenAI Integration:**
   - Add a **Code** node named `Build AI Prompt` to convert the filtered JSON objects into a text summary. Connect `Detect Price Spikes` to it.
   - Add an **OpenAI** node named `AI Market Take`. Set Resource to `Chat`, Model to `gpt-4o-mini`, Operation to `Complete`. Add system prompt and user prompt expression binding to `{{ 'These coins moved sharply in the last 24 hours. Write 3 sharp one-line takes on what might be driving these moves.\n\n' + $json.moversText }}`.
   - Configure OpenAI Credentials with a valid API key. Connect `Build AI Prompt` to `AI Market Take`.

5. **Format and Deliver the Output:**
   - Add a **Code** node named `Format Telegram Alert`. Use JavaScript to pull data from both `Detect Price Spikes` and `AI Market Take`, structuring it into a Markdown checklist format. Connect `AI Market Take` to it.
   - Add a **Telegram** node named `Send Telegram Alert`. Set Text to `={{ $json.message }}`, Chat ID to `={{ $('Configuration').first().json.telegramChatId }}`, and additional field `parse_mode` to `Markdown`.
   - Configure Telegram Bot API credentials. Connect `Format Telegram Alert` to `Send Telegram Alert`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| OpenAI Platform API Keys | Obtain credentials at [platform.openai.com](https://platform.openai.com) |
| Telegram Bot Creation | Create a bot instance via [@BotFather](https://t.me/BotFather) |
| Telegram User ID Discovery | Retrieve chat IDs using [@userinfobot](https://t.me/userinfobot) |