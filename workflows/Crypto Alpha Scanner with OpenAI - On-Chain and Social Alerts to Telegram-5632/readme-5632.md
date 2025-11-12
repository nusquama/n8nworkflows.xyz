Crypto Alpha Scanner with OpenAI - On-Chain and Social Alerts to Telegram

https://n8nworkflows.xyz/workflows/crypto-alpha-scanner-with-openai---on-chain-and-social-alerts-to-telegram-5632


# Crypto Alpha Scanner with OpenAI - On-Chain and Social Alerts to Telegram

### 1. Workflow Overview

This workflow, titled **"Crypto Alpha Scanner with OpenAI - On-Chain and Social Alerts to Telegram"**, is designed to aggregate and analyze cryptocurrency signals from multiple on-chain and social media sources. It processes data from whale wallet movements, Reddit crypto discussions, trending and new tokens from external APIs, and tweets. The aggregated data is then filtered and enhanced using AI analysis (OpenAI) to generate meaningful "alpha" alerts, which are formatted and conditionally sent to a Telegram chat or channel.

The workflow is logically divided into the following blocks:

- **1.1 Configuration and Scheduling**  
  Initializes configuration and triggers the workflow on a schedule.

- **1.2 Data Collection**  
  Fetches data from multiple sources: whale wallet movements (via Etherscan), Reddit crypto discussions, tweets, DexScreener for token monitoring, and token discovery endpoints for new and trending tokens.

- **1.3 Data Processing and Aggregation**  
  Processes raw data including splitting VIP wallets, batching requests, aggregating whale moves, Reddit posts, and other token data.

- **1.4 Advanced Filtering and AI Analysis**  
  Performs filtering and pattern detection on aggregated data, then applies OpenAI-based analysis to extract actionable insights.

- **1.5 Message Formatting and Conditional Sending**  
  Formats the AI-processed data into Telegram message format and conditionally sends messages based on evaluation criteria.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration and Scheduling

- **Overview:**  
  This block initializes the workflow configuration and triggers the data collection process on a scheduled basis.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Load Configuration1

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Role: Starts the workflow on a recurring schedule (configured in node parameters).  
    - Configuration: Default schedule (not specified in JSON) likely set to periodic intervals for scanning.  
    - Input: None (trigger node).  
    - Output: Triggers "Load Configuration1".  
    - Failures: None expected unless internal scheduler fails.

  - **Load Configuration1**  
    - Type: `Code`  
    - Role: Loads or initializes configuration parameters, possibly including API keys, wallet lists, and filtering options.  
    - Configuration: Custom JavaScript code (details abstracted).  
    - Input: Trigger from schedule.  
    - Output: Starts multiple branches for data collection: DexScreener monitoring, new/trending tokens discovery, Reddit scanning, wallet splitting, tweet fetching.  
    - Failures: Possible errors from missing config parameters or malformed code.

---

#### 1.2 Data Collection

- **Overview:**  
  Collects raw data from various external sources: whale wallet movements via Etherscan API, Reddit crypto posts via Reddit API, tweets, token monitoring, and token discovery endpoints.

- **Nodes Involved:**  
  - Split Out  
  - Loop Over Items  
  - Fetch Whale Moves1  
  - Reddit Crypto Scanner  
  - Get Tweets  
  - DexScreener My Tokens Monitor  
  - Discover New Tokens  
  - Discover Trending Tokens

- **Node Details:**

  - **Split Out**  
    - Type: `Split Out`  
    - Role: Splits an array of VIP wallet addresses into individual items for sequential processing.  
    - Configuration: Default splitting of input array.  
    - Input: Wallet array from configuration.  
    - Output: One wallet per item to Loop Over Items.  
    - Failures: None expected unless input is not an array.

  - **Loop Over Items**  
    - Type: `Split In Batches`  
    - Role: Processes wallets one at a time to respect API rate limits.  
    - Configuration: Batch size set to 1 (implied).  
    - Input: Individual wallet from Split Out.  
    - Output: Two branches: Whale Aggregate and Fetch Whale Moves1.  
    - Failures: Timeout or rate limit errors from API calls.

  - **Fetch Whale Moves1**  
    - Type: `HTTP Request`  
    - Role: Fetches on-chain whale wallet movements using the Etherscan API.  
    - Configuration: Requires Etherscan API key in credentials.  
    - Input: Wallet address from Loop Over Items.  
    - Output: Loops back to Loop Over Items for next wallet.  
    - Failures: API key missing or invalid, network errors, rate limits.

  - **Whale Aggregate**  
    - Type: `Aggregate`  
    - Role: Aggregates all whale movement data into a collective dataset.  
    - Input: From Loop Over Items.  
    - Output: To Merge node for combined processing.  
    - Failures: Malformed data leading to aggregation errors.

  - **Reddit Crypto Scanner**  
    - Type: `Reddit`  
    - Role: Pulls crypto-related posts from Reddit using OAuth2 credentials.  
    - Configuration: Requires Reddit OAuth2 credentials configured.  
    - Input: Trigger from Load Configuration1.  
    - Output: Goes to Reddit Aggregate.  
    - Failures: OAuth token expiration, API rate limits.

  - **Reddit Aggregate**  
    - Type: `Aggregate`  
    - Role: Aggregates Reddit posts data for further processing.  
    - Input: From Reddit Crypto Scanner.  
    - Output: To Merge node.  
    - Failures: Data format issues.

  - **Get Tweets**  
    - Type: `HTTP Request`  
    - Role: Fetches relevant tweets from Twitter API or other tweet sources.  
    - Configuration: Not explicitly detailed, likely requires Twitter API credentials (not in JSON).  
    - Input: From Load Configuration1.  
    - Output: To Merge node.  
    - Failures: API authentication, rate limits.

  - **DexScreener My Tokens Monitor**  
    - Type: `HTTP Request`  
    - Role: Fetches monitoring data for user tokens from DexScreener API.  
    - Input: From Load Configuration1.  
    - Output: To Merge node.  
    - Failures: API errors, invalid token list.

  - **Discover New Tokens**  
    - Type: `HTTP Request`  
    - Role: Retrieves data on recently discovered tokens from an external API.  
    - Input: From Load Configuration1.  
    - Output: To Merge node.  
    - Failures: API errors, data unavailability.

  - **Discover Trending Tokens**  
    - Type: `HTTP Request`  
    - Role: Retrieves trending tokens data from an external API.  
    - Input: From Load Configuration1.  
    - Output: To Merge node.  
    - Failures: API errors, data unavailability.

---

#### 1.3 Data Processing and Aggregation

- **Overview:**  
  This block merges the outputs from multiple data sources into a single dataset for unified processing.

- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - **Merge**  
    - Type: `Merge`  
    - Role: Combines multiple inputs: whale moves, Reddit posts, tweets, token monitoring, new and trending tokens into one dataset.  
    - Configuration: Merge mode defaults to "Append" (not explicitly specified).  
    - Input: Receives data streams from Whale Aggregate, Reddit Aggregate, Get Tweets, DexScreener My Tokens Monitor, Discover New Tokens, Discover Trending Tokens.  
    - Output: To Advanced Filtering & Pattern Detection1.  
    - Failures: Data format incompatibility, missing inputs.

---

#### 1.4 Advanced Filtering and AI Analysis

- **Overview:**  
  Applies custom filtering and pattern detection on the aggregated dataset, followed by AI-powered analysis using OpenAI to identify high-value trading signals ("alpha").

- **Nodes Involved:**  
  - Advanced Filtering & Pattern Detection1  
  - AI Alpha Analysis1

- **Node Details:**

  - **Advanced Filtering & Pattern Detection1**  
    - Type: `Code`  
    - Role: Runs custom JavaScript logic to filter out noise and detect meaningful patterns in the combined data.  
    - Configuration: Custom code (not shown) performing advanced heuristics.  
    - Input: From Merge node.  
    - Output: To AI Alpha Analysis1.  
    - Failures: Code errors, unexpected data structures.

  - **AI Alpha Analysis1**  
    - Type: `OpenAI` (Langchain node)  
    - Role: Sends filtered data to OpenAI API (GPT or similar) for natural language analysis to generate insights or summaries.  
    - Configuration: Requires OpenAI API credentials.  
    - Input: From Advanced Filtering node.  
    - Output: To Format for Telegram node.  
    - Failures: API key issues, rate limits, prompt errors.

---

#### 1.5 Message Formatting and Conditional Sending

- **Overview:**  
  Formats AI-generated insights into Telegram-friendly messages and sends them conditionally based on whether the message meets criteria.

- **Nodes Involved:**  
  - Format for Telegram  
  - Should Send Message?  
  - Send to Telegram1

- **Node Details:**

  - **Format for Telegram**  
    - Type: `Code`  
    - Role: Formats AI analysis output into Telegram message format (text, markdown, etc.).  
    - Configuration: Custom JavaScript code (details abstracted).  
    - Input: From AI Alpha Analysis1.  
    - Output: To Should Send Message? node.  
    - Failures: Formatting errors, expression failures.

  - **Should Send Message?**  
    - Type: `If`  
    - Role: Checks conditions to decide if the Telegram message should be sent (e.g., non-empty content, thresholds).  
    - Configuration: Conditional expressions (not shown).  
    - Input: From Format for Telegram.  
    - Output: On true, to Send to Telegram1; on false, workflow ends.  
    - Failures: Expression evaluation errors.

  - **Send to Telegram1**  
    - Type: `Telegram`  
    - Role: Sends the final message to a Telegram chat or channel.  
    - Configuration: Requires Telegram Bot credentials and chat/channel ID replacement with user's chat ID.  
    - Input: From Should Send Message? node.  
    - Output: End of workflow.  
    - Failures: Invalid chat ID, bot authorization errors, network issues.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                              | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                 |
|-------------------------------|---------------------------|----------------------------------------------|----------------------------------|---------------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger          | Initiates workflow on schedule                | None                             | Load Configuration1             |                                                                                             |
| Load Configuration1           | Code                      | Loads config & starts data collection         | Schedule Trigger                 | DexScreener My Tokens Monitor, Discover New Tokens, Discover Trending Tokens, Reddit Crypto Scanner, Split Out, Get Tweets |                                                                                             |
| Split Out                     | Split Out                 | Splits VIP wallet array into individual items | Load Configuration1             | Loop Over Items                 | Splits VIP wallet array into individual items for processing                               |
| Loop Over Items               | Split In Batches          | Processes wallets one by one                   | Split Out                      | Whale Aggregate, Fetch Whale Moves1 | Processes wallets one by one to avoid rate limits                                         |
| Fetch Whale Moves1            | HTTP Request              | Fetches whale wallet movements from Etherscan| Loop Over Items                | Loop Over Items                 | Add your Etherscan API key in credentials                                                  |
| Whale Aggregate               | Aggregate                 | Aggregates whale move data                      | Loop Over Items                | Merge                          |                                                                                             |
| Reddit Crypto Scanner         | Reddit                    | Fetches crypto posts from Reddit               | Load Configuration1            | Reddit Aggregate               | Configure Reddit OAuth2 credentials                                                        |
| Reddit Aggregate              | Aggregate                 | Aggregates Reddit posts                         | Reddit Crypto Scanner          | Merge                          |                                                                                             |
| Get Tweets                   | HTTP Request              | Fetches tweets                                  | Load Configuration1            | Merge                          |                                                                                             |
| DexScreener My Tokens Monitor | HTTP Request              | Fetches user token data from DexScreener       | Load Configuration1            | Merge                          |                                                                                             |
| Discover New Tokens           | HTTP Request              | Fetches new tokens data                         | Load Configuration1            | Merge                          |                                                                                             |
| Discover Trending Tokens      | HTTP Request              | Fetches trending tokens data                    | Load Configuration1            | Merge                          |                                                                                             |
| Merge                        | Merge                     | Combines all collected data                     | Whale Aggregate, Whale Aggregate, Reddit Aggregate, DexScreener My Tokens Monitor, Discover New Tokens, Discover Trending Tokens, Get Tweets | Advanced Filtering & Pattern Detection1 |                                                                                             |
| Advanced Filtering & Pattern Detection1 | Code          | Filters and detects patterns in data            | Merge                         | AI Alpha Analysis1             |                                                                                             |
| AI Alpha Analysis1           | OpenAI (Langchain)        | Performs AI analysis on filtered data           | Advanced Filtering & Pattern Detection1 | Format for Telegram        | Configure your OpenAI API credentials                                                      |
| Format for Telegram          | Code                      | Formats AI output into Telegram message format  | AI Alpha Analysis1             | Should Send Message?           |                                                                                             |
| Should Send Message?         | If                        | Conditional check to send Telegram message      | Format for Telegram            | Send to Telegram1              |                                                                                             |
| Send to Telegram1            | Telegram                  | Sends formatted message to Telegram             | Should Send Message?           | None                          | Replace chatId with your Telegram chat/channel ID and configure bot credentials            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to desired interval (e.g., every 15 minutes) to run the workflow automatically.

2. **Add a Code node named "Load Configuration1"**  
   - Write JavaScript to load your configuration parameters such as:  
     - VIP wallet addresses array  
     - API keys for Etherscan, Reddit OAuth2, OpenAI, Telegram bot token  
     - Other parameters like filters or thresholds  
   - Connect Schedule Trigger → Load Configuration1.

3. **Add a Split Out node named "Split Out"**  
   - Input: VIP wallet addresses array from Load Configuration1 node.  
   - Connect Load Configuration1 → Split Out.

4. **Add a Split In Batches node named "Loop Over Items"**  
   - Batch size: 1 (to process wallets one by one).  
   - Connect Split Out → Loop Over Items.

5. **Add an HTTP Request node named "Fetch Whale Moves1"**  
   - Method: GET  
   - URL: Etherscan API endpoint to fetch wallet transactions (e.g., `https://api.etherscan.io/api`)  
   - Parameters: wallet address from current item, your Etherscan API key from credentials.  
   - Credentials: Configure Etherscan API key credential.  
   - Connect Loop Over Items → Fetch Whale Moves1.

6. **Connect Fetch Whale Moves1 → Loop Over Items**  
   - This creates a loop over all wallets.

7. **Add an Aggregate node named "Whale Aggregate"**  
   - Purpose: Collect all whale moves from the batch processing.  
   - Connect Loop Over Items → Whale Aggregate.

8. **Add a Reddit node named "Reddit Crypto Scanner"**  
   - Configure Reddit OAuth2 credentials.  
   - Set subreddit or search criteria related to crypto.  
   - Connect Load Configuration1 → Reddit Crypto Scanner.

9. **Add an Aggregate node named "Reddit Aggregate"**  
   - Connect Reddit Crypto Scanner → Reddit Aggregate.

10. **Add an HTTP Request node named "Get Tweets"**  
    - Configure with Twitter API credentials (or relevant tweet source).  
    - Set query parameters for crypto-related tweets.  
    - Connect Load Configuration1 → Get Tweets.

11. **Add HTTP Request nodes for DexScreener and token discovery:**  
    - "DexScreener My Tokens Monitor"  
    - "Discover New Tokens"  
    - "Discover Trending Tokens"  
    - Configure each with respective API URLs and parameters.  
    - Connect Load Configuration1 → each of these nodes.

12. **Add a Merge node named "Merge"**  
    - Connect outputs from: Whale Aggregate, Reddit Aggregate, Get Tweets, DexScreener My Tokens Monitor, Discover New Tokens, Discover Trending Tokens → Merge.  
    - Set merge mode to append or default.

13. **Add a Code node named "Advanced Filtering & Pattern Detection1"**  
    - Add JavaScript logic to filter merged data and detect patterns.  
    - Connect Merge → Advanced Filtering & Pattern Detection1.

14. **Add an OpenAI node named "AI Alpha Analysis1"**  
    - Configure OpenAI API credentials.  
    - Set prompt to analyze and extract alpha insights from filtered data.  
    - Connect Advanced Filtering & Pattern Detection1 → AI Alpha Analysis1.

15. **Add a Code node named "Format for Telegram"**  
    - Format AI output into Telegram message text (e.g., markdown).  
    - Connect AI Alpha Analysis1 → Format for Telegram.

16. **Add an If node named "Should Send Message?"**  
    - Set condition to check if message content is present or meets criteria.  
    - Connect Format for Telegram → Should Send Message?.

17. **Add a Telegram node named "Send to Telegram1"**  
    - Configure Telegram bot credentials (token).  
    - Set chatId to your target Telegram chat or channel ID.  
    - Connect Should Send Message? (true) → Send to Telegram1.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                            |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Configure Reddit OAuth2 credentials to enable Reddit API access.                                                              | Related to "Reddit Crypto Scanner" node.                   |
| Add your Etherscan API key in credentials to fetch whale wallet movements.                                                    | Related to "Fetch Whale Moves1" node.                       |
| Configure your OpenAI API credentials to enable AI analysis of aggregated data.                                               | Related to "AI Alpha Analysis1" node.                       |
| Replace chatId with your Telegram chat/channel ID and configure bot credentials for message delivery.                        | Related to "Send to Telegram1" node.                        |
| Processes wallets one by one to avoid API rate limits and potential throttling issues.                                         | Related to "Loop Over Items" node.                          |
| Splits VIP wallet array into individual items for separate API calls and processing.                                          | Related to "Split Out" node.                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow constructed using n8n, a workflow automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or restricted elements. All handled data is legal and publicly accessible.