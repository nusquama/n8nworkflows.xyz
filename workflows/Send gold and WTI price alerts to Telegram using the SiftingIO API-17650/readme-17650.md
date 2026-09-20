Send gold and WTI price alerts to Telegram using the SiftingIO API

https://n8nworkflows.xyz/workflows/send-gold-and-wti-price-alerts-to-telegram-using-the-siftingio-api-17650


# Send gold and WTI price alerts to Telegram using the SiftingIO API

### 1. Workflow Overview

This workflow is designed to monitor financial commodity prices—specifically Gold (XAUUSD) and WTI crude oil (WTIUSD)—on a regular schedule, evaluate them against configured threshold limits, and send formatted alert notifications via Telegram. 

The execution logic is structured into the following functional blocks:
- **1.1 Schedule Trigger & Data Acquisition:** Triggers the workflow every 15 minutes and concurrently requests the latest market trade data for both Gold and WTI crude oil from the SiftingIO API using HTTP Header authentication.
- **1.2 Data Synchronization & Evaluation:** Synchronizes the parallel API responses using a Merge node, then processes the payloads through a JavaScript code node to extract numeric prices and evaluate them against predefined upper and lower bounds (or generates a test message if `TEST_MODE` is enabled).
- **1.3 Notification Dispatch:** Receives the generated alert objects and sends them sequentially to a designated Telegram chat using the Telegram Bot API.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule Trigger & Data Acquisition

##### Overview
This block initiates the execution cycle on a fixed 15-minute interval and simultaneously queries the SiftingIO REST API for the latest trade values of Gold and WTI crude oil.

##### Nodes Involved
- `Every 15 minutes` (`n8n-nodes-base.scheduleTrigger`)
- `Get gold price` (`n8n-nodes-base.httpRequest`)
- `Get WTI price` (`n8n-nodes-base.httpRequest`)

##### Node Details

- **Every 15 minutes**
  - **Type & Technical Role:** Schedule Trigger node; acts as the primary time-based entry point for the workflow.
  - **Configuration:** Configured to fire an execution rule every 15 minutes.
  - **Input/Output:** Has no input connections; outputs main execution flow to both HTTP request nodes simultaneously.
  - **Edge Cases:** Missed executions due to system downtime depend on host infrastructure configuration.

- **Get gold price**
  - **Type & Technical Role:** HTTP Request node; fetches trade data for the XAUUSD symbol.
  - **Configuration:** Uses GET method targeting `https://api.sifting.io/v1/last/trade/commodities/XAUUSD`. Uses generic HTTP Header Authentication.
  - **Input/Output:** Input connected from `Every 15 minutes`; output connects to input index 0 of `Wait for both prices`.
  - **Edge Cases & Failure Types:** Authentication failures (invalid API key), network timeouts, rate-limiting, or unexpected API response schemas.

- **Get WTI price**
  - **Type & Technical Role:** HTTP Request node; fetches trade data for the WTIUSD symbol.
  - **Configuration:** Uses GET method targeting `https://api.sifting.io/v1/last/trade/commodities/WTIUSD`. Uses generic HTTP Header Authentication.
  - **Input/Output:** Input connected from `Every 15 minutes`; output connects to input index 1 of `Wait for both prices`.
  - **Edge Cases & Failure Types:** Same as the gold price request node (auth errors, timeouts, rate limits).

---

#### Block 1.2: Data Synchronization & Evaluation

##### Overview
This block aggregates the parallel HTTP responses, extracts valid numeric price indicators using a recursive traversal helper function, and checks values against boundary conditions or test-mode parameters.

##### Nodes Involved
- `Wait for both prices` (`n8n-nodes-base.merge`)
- `Check alert thresholds` (`n8n-nodes-base.code`)

##### Node Details

- **Wait for both prices**
  - **Type & Technical Role:** Merge node; synchronizes multi-input data streams.
  - **Configuration:** Standard merge configuration combining parallel inputs.
  - **Input/Output:** Input 0 receives data from `Get gold price`; Input 1 receives data from `Get WTI price`. Output connects to `Check alert thresholds`.
  - **Edge Cases:** If one branch fails or times out, the merge node may stall unless configured otherwise.

- **Check alert thresholds**
  - **Type & Technical Role:** Code node; executes custom JavaScript logic for data parsing, threshold evaluation, and message formatting.
  - **Configuration:** Contains a recursive `readPrice` function that scans common price keys (`p`, `price`, `last`, `close`, etc.) or calculates midpoints from bid/ask spreads. Evaluates items against a `thresholds` configuration object and checks the `TEST_MODE` boolean flag.
  - **Key Expressions/Variables:** Uses `$('[Node Name]').first().json` expressions to reference upstream HTTP request payloads.
  - **Input/Output:** Input connected from `Wait for both prices`; output connects to `Send Telegram alert`.
  - **Edge Cases & Failure Types:** Throws an explicit runtime error (`No numeric price found in response...`) if the target payload does not contain a recognizable price field or structure.

---

#### Block 1.3: Notification Dispatch

##### Overview
This block receives the formatted alert messages produced by the evaluation code node and delivers them to the target Telegram chat.

##### Nodes Involved
- `Send Telegram alert` (`n8n-nodes-base.telegram`)

##### Node Details

- **Send Telegram alert**
  - **Type & Technical Role:** Telegram node; interacts with the Telegram Bot API to send messages.
  - **Configuration:** Configured with `maxTries: 3`, `retryOnFail: true`, and a `waitBetweenTries` delay of 1000ms. Disables attribution appending.
  - **Key Expressions/Variables:** Message text uses `={{ $json.message }}` to evaluate incoming alert payloads. Chat ID must be configured with a valid target chat identifier.
  - **Input/Output:** Input connected from `Check alert thresholds`; has no downstream node outputs.
  - **Edge Cases & Failure Types:** Invalid Telegram Bot credentials, incorrect chat IDs, or network failures during API transmission (mitigated partially by built-in retry parameters).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Workflow overview | `n8n-nodes-base.stickyNote` | Documentation & requirement guidelines | None | None | Gold & WTI price alerts — SiftingIO → Telegram<br><br>Monitor **Gold (XAUUSD)** and **WTI crude oil (WTIUSD)** every 15 minutes and send Telegram alerts when your chosen thresholds are crossed.<br><br>### Requirements<br>- An n8n Cloud workspace<br>- A SiftingIO API key<br>- A Telegram bot credential<br>- A Telegram chat ID<br><br>This template uses only built-in n8n nodes. No community package or hardcoded secret is required.<br><br>Get a SiftingIO API key:<br>https://sifting.io/register?utm_source=n8n&utm_medium=workflow_template&utm_campaign=commodity_price_alerts |
| SiftingIO setup | `n8n-nodes-base.stickyNote` | Configuration instructions for API authentication | None | None | ### 1. Connect SiftingIO<br><br>Create one **HTTP Header Auth** credential:<br><br>- **Header name:** `X-API-Key`<br>- **Header value:** your SiftingIO API key<br><br>Select the same credential in both HTTP Request nodes:<br>- **Get gold price**<br>- **Get WTI price** |
| Threshold setup | `n8n-nodes-base.stickyNote` | Instructions for managing alert thresholds and test mode | None | None | ### 2. Configure alert thresholds<br><br>Open **Check alert thresholds**.<br><br>For the first manual test, keep:<br>`TEST_MODE = true`<br><br>After Telegram receives the test message:<br>1. Set `TEST_MODE = false`<br>2. Edit the Gold and WTI `above` / `below` values<br>3. Save the workflow |
| Telegram setup | `n8n-nodes-base.stickyNote` | Instructions for connecting Telegram and activation | None | None | ### 3. Connect Telegram and activate<br><br>1. Select or create a Telegram bot credential<br>2. Replace `REPLACE_WITH_YOUR_CHAT_ID`<br>3. Click **Execute workflow** once<br>4. Confirm both API requests succeed and Telegram receives the message<br>5. Disable test mode, then activate the workflow<br><br>The workflow imports inactive and contains no credentials or API keys. |
| Every 15 minutes | `n8n-nodes-base.scheduleTrigger` | Time-based workflow entry point | None | Get gold price, Get WTI price | |
| Get gold price | `n8n-nodes-base.httpRequest` | Fetches XAUUSD commodity price data | Every 15 minutes | Wait for both prices | |
| Get WTI price | `n8n-nodes-base.httpRequest` | Fetches WTIUSD commodity price data | Every 15 minutes | Wait for both prices | |
| Wait for both prices | `n8n-nodes-base.merge` | Synchronizes parallel API data responses | Get gold price, Get WTI price | Check alert thresholds | |
| Check alert thresholds | `n8n-nodes-base.code` | Parses prices and evaluates thresholds or test mode | Wait for both prices | Send Telegram alert | |
| Send Telegram alert | `n8n-nodes-base.telegram` | Delivers notification messages to Telegram chat | Check alert thresholds | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Add a **Schedule Trigger** node named `Every 15 minutes`.
   - Set interval rule to run every `15` minutes.

2. **Create SiftingIO HTTP Request Nodes**
   - Add an **HTTP Request** node named `Get gold price`.
     - Set Method: `GET`
     - Set URL: `https://api.sifting.io/v1/last/trade/commodities/XAUUSD`
     - Configure Authentication to use **Generic Credential Type** -> **HTTP Header Auth**.
   - Add an **HTTP Request** node named `Get WTI price`.
     - Set Method: `GET`
     - Set URL: `https://api.sifting.io/v1/last/trade/commodities/WTIUSD`
     - Configure Authentication to use the same **HTTP Header Auth** credential.
   - Connect output of `Every 15 minutes` to the input of both HTTP Request nodes.

3. **Configure SiftingIO Credentials**
   - Create an **HTTP Header Auth** credential in n8n.
   - Set Header Name: `X-API-Key`
   - Set Header Value: Your SiftingIO API key.
   - Assign this credential to both HTTP Request nodes.

4. **Create Merge Node**
   - Add a **Merge** node named `Wait for both prices`.
   - Connect input index `0` to the output of `Get gold price`.
   - Connect input index `1` to the output of `Get WTI price`.

5. **Create Code Evaluation Node**
   - Add a **Code** node named `Check alert thresholds`.
   - Connect its input to the output of `Wait for both prices`.
   - Paste the following JavaScript code into the node parameters:
     ```javascript
     const TEST_MODE = true;

     const thresholds = {
       XAUUSD: { label: 'Gold', above: 4000, below: 2000 },
       WTIUSD: { label: 'WTI crude oil', above: 120, below: 40 },
     };

     function readPrice(payload) {
       const queue = [payload];
       while (queue.length) {
         const current = queue.shift();
         if (!current || typeof current !== 'object') continue;

         const candidates = [
           current.p, current.price, current.last, current.last_price,
           current.c, current.close, current.mid, current.m
         ];
         for (const value of candidates) {
           const n = Number(value);
           if (Number.isFinite(n)) return n;
         }

         const bid = Number(current.b ?? current.bid);
         const ask = Number(current.a ?? current.ask);
         if (Number.isFinite(bid) && Number.isFinite(ask)) return (bid + ask) / 2;

         for (const key of ['data', 'result', 'trade', 'quote']) {
           if (current[key] && typeof current[key] === 'object') queue.push(current[key]);
         }
       }

       throw new Error(`No numeric price found in response: ${JSON.stringify(payload)}`);
     }

     const rows = [
       { symbol: 'XAUUSD', payload: $('Get gold price').first().json },
       { symbol: 'WTIUSD', payload: $('Get WTI price').first().json },
     ];

     const alerts = [];
     for (const row of rows) {
       const price = readPrice(row.payload);
       const cfg = thresholds[row.symbol];

       if (TEST_MODE) {
         alerts.push({
           symbol: row.symbol,
           price,
           message: `SiftingIO test alert\n${cfg.label} (${row.symbol}): ${price}\nUTC: ${new Date().toISOString()}`
         });
         continue;
       }

       if (price >= cfg.above) {
         alerts.push({
           symbol: row.symbol,
           price,
           message: `${cfg.label} alert\n${row.symbol} is ${price}, at or above ${cfg.above}.\nUTC: ${new Date().toISOString()}`
         });
       } else if (price <= cfg.below) {
         alerts.push({
           symbol: row.symbol,
           price,
           message: `${cfg.label} alert\n${row.symbol} is ${price}, at or below ${cfg.below}.\nUTC: ${new Date().toISOString()}`
         });
       }
     }

     return alerts.map((alert) => ({ json: alert }));
     ```

6. **Create Telegram Notification Node**
   - Add a **Telegram** node named `Send Telegram alert`.
   - Connect its input to the output of `Check alert thresholds`.
   - Set parameters:
     - Resource: `Message`
     - Operation: `Send`
     - Text: `={{ $json.message }}`
     - Chat ID: Enter your target Telegram chat identifier (replace placeholder).
     - Additional Fields -> Append Attribution: `false`
   - Configure retry parameters: Enable retry on fail (`true`), set max tries to `3`, and wait between tries to `1000` ms.

7. **Credential Setup & Activation Sequence**
   - Create and select a valid **Telegram API** credential on the Telegram node.
   - Execute the workflow manually once in `TEST_MODE = true` to confirm connection stability.
   - Modify the code node to set `TEST_MODE = false` and adjust commodity thresholds according to operational requirements.
   - Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Get a SiftingIO API key | [SiftingIO Registration Page](https://sifting.io/register?utm_source=n8n&utm_medium=workflow_template&utm_campaign=commodity_price_alerts) |
| Workflow import state | Imported inactive; contains no hardcoded credentials or API keys. |
| Initial test recommendation | Keep `TEST_MODE = true` on the first manual run to verify Telegram message delivery before activating automated threshold monitoring. |