Cryptocurrency Dip Alerts for Bitcoin & Ethereum via Telegram, Slack & SMS

https://n8nworkflows.xyz/workflows/cryptocurrency-dip-alerts-for-bitcoin---ethereum-via-telegram--slack---sms-8386


# Cryptocurrency Dip Alerts for Bitcoin & Ethereum via Telegram, Slack & SMS

### 1. Workflow Overview

This workflow monitors cryptocurrency price dips for Bitcoin (BTC) and Ethereum (ETH) by checking their 24-hour percentage change every 30 minutes. When a dip threshold (default: -2.5%) is met or exceeded (price decline), it sends alert notifications through multiple messaging channels: Telegram, Slack, and SMS (via Twilio). The workflow is designed for traders or enthusiasts who want timely dip alerts without continuously watching market charts.

**Logical blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow every 30 minutes.
- **1.2 Price Retrieval:** Fetches current price and 24-hour change data for BTC and ETH from CoinGecko API.
- **1.3 Dip Detection:** Uses a code node to evaluate if either cryptocurrency’s 24h change meets the dip threshold.
- **1.4 Conditional Check:** Routes execution only if a dip alert condition is true.
- **1.5 Notifications:** Sends dip alert messages to Telegram, Slack, and SMS (Twilio).

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow every 30 minutes to initiate the price check and alert process.

- **Nodes Involved:**  
  - Every 30 Minutes

- **Node Details:**  
  - Type: Schedule Trigger  
  - Configuration: Interval set to every 30 minutes (minutes field used)  
  - Expressions/Variables: None  
  - Input Connections: None (start node)  
  - Output Connections: Connects to "Get Crypto Prices" node  
  - Version: 1.1  
  - Edge Cases: Possible time drift if n8n instance is paused or under heavy load; can be adjusted by changing the cron interval.

#### 2.2 Price Retrieval

- **Overview:**  
  Fetches the latest USD price and 24h percentage change for Bitcoin and Ethereum from the CoinGecko public API.

- **Nodes Involved:**  
  - Get Crypto Prices

- **Node Details:**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`  
    - Request Method: GET (default)  
    - No authentication or additional headers required  
  - Expressions/Variables: None  
  - Input: Receives trigger from "Every 30 Minutes"  
  - Output: JSON containing BTC and ETH prices and 24h changes, e.g.:  
    ```json
    {
      "bitcoin": {"usd": 27000, "usd_24h_change": -3.2},
      "ethereum": {"usd": 1800, "usd_24h_change": -2.8}
    }
    ```  
  - Version: 4.1  
  - Edge Cases:  
    - API downtime or rate limiting may cause request failure or empty data  
    - Network errors/timeouts

#### 2.3 Dip Detection

- **Overview:**  
  Processes the API response to determine whether either BTC or ETH’s 24h change is less than or equal to -2.5%. Constructs an alert text summarizing the dip.

- **Nodes Involved:**  
  - Dip Check

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Defines constant `DIP = -2.5` (dip threshold)  
    - Reads BTC and ETH 24h change values from input JSON  
    - Evaluates if either change ≤ DIP threshold  
    - Returns an object with:  
      - `dip`: boolean indicating if dip threshold is met  
      - `text`: formatted string e.g. `"Dip Alert — BTC -3.20%, ETH -2.80%"`  
  - Expressions/Variables: Uses `$json` to access input data  
  - Input: Output of "Get Crypto Prices"  
  - Output: JSON with `dip` boolean and `text` message  
  - Version: 2  
  - Edge Cases:  
    - Missing or malformed numeric values could cause `NaN` or unexpected behavior  
    - Defensive checks use `isFinite()` to mitigate this

#### 2.4 Conditional Check

- **Overview:**  
  Evaluates the boolean `dip` value to determine if notification nodes should be executed.

- **Nodes Involved:**  
  - Is Dip?

- **Node Details:**  
  - Type: IF  
  - Configuration:  
    - Condition: checks if `dip` equals `true` (note: in the JSON condition fields are blank but presumably set properly in the live node)  
    - Combinator: AND  
  - Expressions/Variables: Evaluates expression referencing output of "Dip Check" node  
  - Input: Output of "Dip Check"  
  - Output:  
    - True branch: Connects to all notification nodes (Telegram, Slack, Twilio)  
    - False branch: Workflow ends  
  - Version: 2  
  - Edge Cases:  
    - Misconfiguration may cause no notifications or false positives  
    - Expression evaluation errors if data is missing

#### 2.5 Notifications

- **Overview:**  
  Sends the dip alert message to three different channels: Telegram, Slack, and SMS via Twilio.

- **Nodes Involved:**  
  - Send Telegram  
  - Send a message (Slack)  
  - Send an SMS/MMS/WhatsApp message (Twilio)

- **Node Details:**

  1. **Send Telegram**  
     - Type: Telegram  
     - Configuration:  
       - Text: uses expression `{{$json.text}}` with appended disclaimer `_Not financial advice._`  
       - Chat ID: configured to a Telegram channel or chat identifier (`@your_channel_or_chatid`)  
       - Credentials: Telegram API credentials (OAuth token or bot token) named "zmeena2"  
     - Input: True output of "Is Dip?" node  
     - Output: None  
     - Version: 1  
     - Edge Cases:  
       - Invalid chat ID or revoked token causes message failure  
       - Telegram API rate limits

  2. **Send a message (Slack)**  
     - Type: Slack  
     - Configuration:  
       - Text: same as Telegram  
       - Authentication: OAuth2 using "Slack account" credentials  
     - Input: True output of "Is Dip?" node  
     - Output: None  
     - Version: 2.3  
     - Edge Cases:  
       - OAuth token expiration or revocation  
       - Slack rate limits or channel permissions

  3. **Send an SMS/MMS/WhatsApp message (Twilio)**  
     - Type: Twilio  
     - Configuration:  
       - Message: same text as others  
       - Credentials: presumably Twilio account API keys (not shown)  
     - Input: True output of "Is Dip?" node  
     - Output: None  
     - Version: 1  
     - Edge Cases:  
       - Invalid phone number format or unverified sender  
       - Twilio API quota exceeded or authentication failure

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                 | Input Node(s)          | Output Node(s)                                   | Sticky Note                                                                                                  |
|-------------------------------|---------------------------|--------------------------------|-----------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Every 30 Minutes               | Schedule Trigger          | Initiates workflow every 30 min| None                  | Get Crypto Prices                               |                                                                                                              |
| Get Crypto Prices             | HTTP Request              | Fetches BTC/ETH price & 24h change | Every 30 Minutes      | Dip Check                                       |                                                                                                              |
| Dip Check                    | Code                      | Checks if dip threshold met     | Get Crypto Prices      | Is Dip?                                         |                                                                                                              |
| Is Dip?                      | IF                        | Routes flow based on dip boolean| Dip Check             | Send Telegram, Send a message (Slack), Send SMS |                                                                                                              |
| Send Telegram                | Telegram                  | Sends alert message to Telegram | Is Dip? (true)         | None                                           |                                                                                                              |
| Send a message               | Slack                     | Sends alert message to Slack    | Is Dip? (true)         | None                                           |                                                                                                              |
| Send an SMS/MMS/WhatsApp message | Twilio                 | Sends alert message via SMS     | Is Dip? (true)         | None                                           |                                                                                                              |
| Sticky Note                  | Sticky Note               | Documentation note              | None                  | None                                           | **📉 Buy the Dip Alert (Telegram/Slack/SMS)**  Sends alerts when BTC/ETH dip ≤ -2.5%. Trigger every 30 min.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**
   - Name: `Every 30 Minutes`
   - Type: Schedule Trigger
   - Parameters: Set interval trigger to every 30 minutes (select minutes, enter 30)
   - No credentials necessary.

2. **Create the HTTP Request node:**
   - Name: `Get Crypto Prices`
   - Type: HTTP Request
   - Parameters:  
     - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`  
     - Method: GET (default)  
     - No authentication needed.
   - Connect `Every 30 Minutes` node output to this node input.

3. **Create the Code node:**
   - Name: `Dip Check`
   - Type: Code (JavaScript)
   - Parameters:  
     Use the following code snippet:
     ```javascript
     const DIP = -2.5; // dip threshold in percent
     const j = $json;
     const btc = Number(j.bitcoin?.usd_24h_change);
     const eth = Number(j.ethereum?.usd_24h_change);
     return [{
       json: {
         dip: (isFinite(btc) && btc <= DIP) || (isFinite(eth) && eth <= DIP),
         text: `Dip Alert — BTC ${btc?.toFixed(2)}%, ETH ${eth?.toFixed(2)}%`
       }
     }];
     ```
   - Connect `Get Crypto Prices` output to this node input.

4. **Create the IF node:**
   - Name: `Is Dip?`
   - Type: IF
   - Parameters:  
     - Set condition to: Expression  
     - Expression: `$json["dip"] === true`  
       (This checks the boolean `dip` from the code node output.)  
     - Connect `Dip Check` output to `Is Dip?` node input.

5. **Create the Telegram node:**
   - Name: `Send Telegram`
   - Type: Telegram
   - Parameters:  
     - Text: `={{ $json.text }}\n_Not financial advice._`  
     - Chat ID: set to your Telegram channel or chat ID (e.g., `@your_channel_or_chatid`)  
   - Credentials: Create or select Telegram API credentials (bot token)  
   - Connect IF node’s “true” output to this node.

6. **Create the Slack node:**
   - Name: `Send a message`
   - Type: Slack
   - Parameters:  
     - Text: `={{ $json.text }} _Not financial advice._`  
     - Authentication: OAuth2  
   - Credentials: Add Slack OAuth2 credentials with permission to post messages  
   - Connect IF node’s “true” output to this node.

7. **Create the Twilio node:**
   - Name: `Send an SMS/MMS/WhatsApp message`
   - Type: Twilio
   - Parameters:  
     - Message: `={{ $json.text }} _Not financial advice._`  
     - Configure sender and recipient numbers in credentials or options as needed  
   - Credentials: Add Twilio API credentials (Account SID and Auth Token)  
   - Connect IF node’s “true” output to this node.

8. **Add a Sticky Note (optional):**
   - Content: Use the provided detailed description for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                     | Context or Link                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow is authored by David Olusola and intended to help users catch significant BTC and ETH dips automatically.                                                        | Author contact: sales@daexai.com               |
| Alert text includes a disclaimer “_Not financial advice._” to clarify the message intent.                                                                                        | Compliance note                                |
| Adjust the dip threshold by modifying the `DIP` constant in the Code node to suit personal risk tolerance or strategy.                                                          | Customization tip                              |
| Telegram, Slack, and Twilio credentials must be properly configured with necessary API keys and permissions to enable message delivery.                                        | Credential setup                               |
| CoinGecko API is public and free but subject to rate limits; consider caching or alternate APIs if scaling.                                                                      | API usage note                                 |

---

**Disclaimer:** The provided text is exclusively sourced from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly available.