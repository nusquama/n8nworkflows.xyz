Automated Trending Cryptocurrency Updates from CoinGecko to Telegram

https://n8nworkflows.xyz/workflows/automated-trending-cryptocurrency-updates-from-coingecko-to-telegram-8271


# Automated Trending Cryptocurrency Updates from CoinGecko to Telegram

### 1. Workflow Overview

This workflow automates the retrieval and dissemination of trending cryptocurrency data from the CoinGecko public API to a Telegram group or channel. It is designed for community managers, traders, or crypto enthusiasts who want to keep their Telegram audiences updated with the latest popular cryptocurrencies.

The workflow logic is divided into the following functional blocks:

- **1.1 Triggering Mechanism**: Supports both manual execution for testing and scheduled automatic runs twice daily.
- **1.2 Data Retrieval**: Fetches trending cryptocurrency search data from CoinGecko’s public API.
- **1.3 Message Formatting**: Processes and formats the retrieved data into a user-friendly text message.
- **1.4 Telegram Notification**: Sends the formatted message to a configured Telegram chat via a bot.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggering Mechanism

**Overview:**  
This block initiates the workflow execution either manually (for tests) or automatically at scheduled times (8:30 AM and 8:30 PM IST).

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- 8:30 AM/PM IST (Schedule Trigger)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows user to manually start the workflow for testing or immediate execution.  
  - *Configuration:* No parameters needed; triggers on user action.  
  - *Input/Output:* No input; outputs to "Get Trending" node.  
  - *Failures:* None typical; user-dependent trigger.

- **8:30 AM/PM IST**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the workflow twice daily at specified times.  
  - *Configuration:* Two trigger times configured: 8:30 AM and 8:30 PM IST.  
  - *Input/Output:* No input; outputs to "Get Trending" node.  
  - *Version:* Uses typeVersion 1.2 for improved scheduling options.  
  - *Failures:* Possible timezone misconfiguration; cron expression errors if modified improperly.

---

#### 1.2 Data Retrieval

**Overview:**  
This block fetches the currently trending cryptocurrencies from CoinGecko’s public API endpoint.

**Nodes Involved:**  
- Get Trending (HTTP Request)

**Node Details:**

- **Get Trending**  
  - *Type:* HTTP Request  
  - *Role:* Sends a GET request to CoinGecko’s trending search endpoint to retrieve trending coin data.  
  - *Configuration:*  
    - URL: `https://api.coingecko.com/api/v3/search/trending`  
    - Method: GET  
    - No authentication required (CoinGecko public API).  
  - *Inputs:* Trigger nodes ("When clicking ‘Execute workflow’" and "8:30 AM/PM IST")  
  - *Outputs:* Passes response JSON to "Format Message".  
  - *Failures & Edge Cases:*  
    - API downtime or rate limits (CoinGecko public API has rate limits).  
    - Network issues causing timeouts or failed requests.  
    - Unexpected response format leading to parsing errors downstream.

---

#### 1.3 Message Formatting

**Overview:**  
Transforms the raw API JSON response into a formatted string message summarizing the trending coins, suitable for Telegram.

**Nodes Involved:**  
- Format Message (Function Node)

**Node Details:**

- **Format Message**  
  - *Type:* Function  
  - *Role:* Processes the `coins` array from the API response and composes a readable text message listing each trending coin with its name, symbol, price (if available), and market cap rank.  
  - *Configuration Highlights:*  
    - Accesses `items[0].json.coins` to retrieve trending coins.  
    - Iterates over each coin, extracting:  
      - `name`  
      - `symbol` (converted to uppercase)  
      - `market_cap_rank` (fallback to 'N/A' if missing)  
      - `price` (formatted to 4 decimals, or 'N/A')  
    - Constructs a multiline string with numbered entries.  
  - *Expressions/Variables:* Uses JavaScript to build the message string dynamically.  
  - *Input:* Output from "Get Trending".  
  - *Output:* JSON object containing the formatted message under `message` key.  
  - *Failures & Edge Cases:*  
    - Missing or malformed `coins` array could cause runtime errors.  
    - Null or undefined price or rank handled gracefully by fallback values.  
    - Unexpected API schema changes may break formatting logic.

---

#### 1.4 Telegram Notification

**Overview:**  
Sends the formatted trending coins message to a specified Telegram chat using a bot.

**Nodes Involved:**  
- Send Telegram Message

**Node Details:**

- **Send Telegram Message**  
  - *Type:* Telegram Node  
  - *Role:* Posts the formatted message to a Telegram chat/channel.  
  - *Configuration:*  
    - Text: Expression referencing the formatted message `={{$json["message"]}}`  
    - Chat ID: Placeholder `<CHAT-ID>` to be replaced by user with actual group or channel ID.  
    - Additional Fields: None set.  
    - Credentials: Uses Telegram Bot API credentials configured in n8n under "TG-CC-Cryptofind".  
  - *Input:* Output from "Format Message".  
  - *Output:* None (end of workflow).  
  - *Failures & Edge Cases:*  
    - Invalid or missing bot token credentials will cause authentication errors.  
    - Incorrect chat ID will result in message delivery failure.  
    - Network issues or Telegram API downtime may cause sending failures.

---

#### Auxiliary

- **Sticky Note**  
  - *Purpose:* Provides detailed documentation and instructions directly visible in the workflow canvas.  
  - *Content Summary:* Explains workflow purpose, usage steps, requirements, and relevant links.  
  - *Position:* Visually separate, no direct connections.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                     | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                                                         |
|-------------------------|---------------------|-----------------------------------|-----------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Manual start for testing            | None                        | Get Trending            | This workflow fetches the latest trending cryptocurrency searches from CoinGecko and automatically sends them to your Telegram group/channel. |
| 8:30 AM/PM IST          | Schedule Trigger    | Automatic twice daily trigger       | None                        | Get Trending            | This workflow fetches the latest trending cryptocurrency searches from CoinGecko and automatically sends them to your Telegram group/channel. |
| Get Trending            | HTTP Request        | Fetch trending coins from CoinGecko| When clicking ‘Execute workflow’, 8:30 AM/PM IST | Format Message          | This workflow fetches the latest trending cryptocurrency searches from CoinGecko and automatically sends them to your Telegram group/channel. |
| Format Message          | Function            | Format API data into Telegram text  | Get Trending                | Send Telegram Message   | This workflow fetches the latest trending cryptocurrency searches from CoinGecko and automatically sends them to your Telegram group/channel. |
| Send Telegram Message   | Telegram            | Send formatted message to Telegram | Format Message              | None                    | This workflow fetches the latest trending cryptocurrency searches from CoinGecko and automatically sends them to your Telegram group/channel. |
| Sticky Note             | Sticky Note         | Documentation & usage instructions  | None                        | None                    | This workflow fetches the latest trending cryptocurrency searches from CoinGecko and automatically sends them to your Telegram group/channel. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed.

2. **Create Schedule Trigger Node**  
   - Node Type: Schedule Trigger  
   - Name: `8:30 AM/PM IST`  
   - Set two trigger times: 8:30 AM and 8:30 PM  
   - Set timezone to `Asia/Kolkata` in workflow settings or node if available.

3. **Create HTTP Request Node**  
   - Node Type: HTTP Request  
   - Name: `Get Trending`  
   - HTTP Method: GET  
   - URL: `https://api.coingecko.com/api/v3/search/trending`  
   - No authentication or additional headers required.

4. **Connect Trigger Nodes to HTTP Request**  
   - Connect outputs of both `When clicking ‘Execute workflow’` and `8:30 AM/PM IST` nodes to the input of `Get Trending`.

5. **Create Function Node for Formatting**  
   - Node Type: Function  
   - Name: `Format Message`  
   - Paste the following JavaScript code into the function editor:

```javascript
const trending = items[0].json.coins;
let message = "🔥 Trending Crypto Searches on CoinGecko:\n\n";

trending.forEach((coin, index) => {
  const item = coin.item;
  const name = item.name;
  const symbol = item.symbol.toUpperCase();
  const marketCapRank = item.market_cap_rank || 'N/A';
  const price = item.data && item.data.price ? `$${item.data.price.toFixed(4)}` : 'N/A';

  message += `${index + 1}. ${name} (${symbol}) - Price: ${price} - Market Cap Rank: ${marketCapRank}\n`;
});

return [{ json: { message } }];
```

6. **Connect HTTP Request to Function Node**  
   - Connect output of `Get Trending` to input of `Format Message`.

7. **Create Telegram Node**  
   - Node Type: Telegram  
   - Name: `Send Telegram Message`  
   - Credentials: Set up Telegram API credentials with your bot token (e.g., named "TG-CC-Cryptofind").  
   - Parameters:  
     - Chat ID: Replace `<CHAT-ID>` with your actual Telegram group or channel ID.  
     - Text: Use expression `={{$json["message"]}}` to send formatted text from previous node.

8. **Connect Function Node to Telegram Node**  
   - Connect output of `Format Message` to input of `Send Telegram Message`.

9. **(Optional) Add Sticky Note**  
   - Add a Sticky Note node anywhere on the canvas.  
   - Paste the descriptive content about workflow purpose, usage, and requirements for future reference.

10. **Workflow Activation**  
    - Set workflow timezone to `Asia/Kolkata` (if not already set).  
    - Activate the workflow to enable scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow fetches the latest trending cryptocurrency searches from CoinGecko and automatically sends them to your Telegram group/channel. No API key required, uses free public API, and supports easy schedule customization. Perfect for community managers and traders.                                                                                  | Documentation Sticky Note inside the workflow     |
| CoinGecko API endpoint used: https://api.coingecko.com/api/v3/search/trending                                                                                                                                                                                                                                                                               | API Documentation Link                            |
| Telegram Bot Token must be created via BotFather on Telegram. Replace `<CHAT-ID>` with your Telegram group or channel ID to receive messages.                                                                                                                                                                                                             | Telegram Bot API: https://core.telegram.org/bots/api |
| Ensure your n8n instance has internet access for API calls and Telegram messaging. Verify credential correctness to avoid authentication errors.                                                                                                                                                                                                             | Operational considerations                         |

---

**Disclaimer:** The content provided comes exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.