Send verified Paddle sale alerts with signature checks to Slack, Discord and Telegram

https://n8nworkflows.xyz/workflows/send-verified-paddle-sale-alerts-with-signature-checks-to-slack--discord-and-telegram-17695


# Send verified Paddle sale alerts with signature checks to Slack, Discord and Telegram

### 1. Workflow Overview

This workflow is designed to securely capture, verify, and broadcast real-time sale alerts from Paddle webhooks. It protects internal endpoints against forgery by performing strict cryptographic signature checks before executing downstream notification tasks. Once validated, completed sales are parsed, filtered, formatted into precise net-earnings reports (accounting for standard and zero-decimal currencies), and simultaneously broadcast to multiple channels.

The architecture is divided into three sequential logical blocks:
- **1.1 Input Reception & Security:** Receives incoming POST webhook events from Paddle and cryptographically validates the message source using HMAC-SHA256.
- **1.2 Filtering & Message Formatting:** Inspects the event type to isolate completed transactions and transforms raw transaction payloads into structured, human-readable alert messages.
- **1.3 Multi-Channel Dispatch:** Fans out the compiled alert message in parallel across three independent communication platforms: Slack, Discord, and Telegram.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Security

##### Overview
This block serves as the single secure entry point for the workflow, capturing incoming HTTP webhook payloads and ensuring that requests originate exclusively from authentic Paddle services through timing-safe cryptographic verification.

##### Nodes Involved
- `When Paddle Sale Alert`
- `Verify Paddle Signature`

##### Node Details

- **When Paddle Sale Alert**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (Version 2) — acts as the HTTP listener for incoming webhook triggers.
  - **Configuration Choices:** Configured to listen for `POST` requests on the path `paddle-sale-alerts` with the **Raw Body** option explicitly enabled to preserve the exact binary string needed for cryptographic verification.
  - **Key Expressions or Variables:** None (triggers on external request body).
  - **Input & Output Connections:** 
    - Input: None (Trigger node).
    - Output: Connects to `Verify Paddle Signature`.
  - **Version-Specific Requirements:** Requires execution order v1 settings enabled globally.
  - **Edge Cases & Potential Failures:** Missing raw body configuration causes downstream signature verification to fail immediately. Unsecured exposure allows unauthenticated traffic if URL paths are leaked.

- **Verify Paddle Signature**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Version 2) — custom Node.js script evaluating cryptographic integrity.
  - **Configuration Choices:** Validates the `paddle-signature` header against a hardcoded webhook secret (`WEBHOOK_SECRET`) using Node's native `crypto` module. Computes an HMAC-SHA256 hash over the string concatenation of the timestamp (`ts`) and raw body (`rawBody`), comparing it using `crypto.timingSafeEqual()`.
  - **Key Expressions or Variables:** 
    - `const WEBHOOK_SECRET = 'PASTE_YOUR_PADDLE_WEBHOOK_SECRET_HERE';`
    - `{{ $input.first() }}`
  - **Input & Output Connections:**
    - Input: Connected from `When Paddle Sale Alert`.
    - Output: Connects to `If Sale Completed`.
  - **Version-Specific Requirements:** Standard Node.js `crypto` library must be available in the execution environment.
  - **Edge Cases & Potential Failures:** Throws explicit errors if the binary data is missing, if headers lack the signature string, or if signatures do not match (rejecting forged requests).

---

#### Block 1.2: Filtering & Message Formatting

##### Overview
This block filters incoming event types to isolate successful checkouts and parses financial data, handling decimal scaling for specialized currencies before generating a unified alert string.

##### Nodes Involved
- `If Sale Completed`
- `Build Alert Message`

##### Node Details

- **If Sale Completed**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Version 2.2) — conditional router.
  - **Configuration Choices:** Evaluates whether the incoming event payload matches the `transaction.completed` string.
  - **Key Expressions or Variables:** 
    - Left value: `={{ $json.event_type }}`
    - Operator: Equals string `transaction.completed`
  - **Input & Output Connections:**
    - Input: Connected from `Verify Paddle Signature`.
    - Output (True): Connects to `Build Alert Message`.
    - Output (False): Unconnected (drops non-completion events).
  - **Version-Specific Requirements:** Version 2.2 condition syntax.
  - **Edge Cases & Potential Failures:** API schema changes in Paddle event types will cause events to be silently dropped if strings drift.

- **Build Alert Message**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Version 2) — custom data transformer.
  - **Configuration Choices:** Extracts transaction totals, line items, and fees. Automatically identifies zero-decimal currencies (`JPY`, `KRW`, `VND`, `ISK`) to format minor currency units correctly without shifting decimal places. Compiles a text template containing product names, revenue numbers, earnings after fees, and transaction IDs.
  - **Key Expressions or Variables:**
    - `{{ $input.first().json.data }}`
  - **Input & Output Connections:**
    - Input: Connected from `If Sale Completed`.
    - Output: Connects in parallel to `Post to Slack Webhook`, `Post to Discord Webhook`, and `Post to Telegram Bot API`.
  - **Version-Specific Requirements:** JavaScript ES6 support.
  - **Edge Cases & Potential Failures:** Missing nested properties (`data`, `details`, `totals`) default safely to fallback strings (`?` or `0`), preventing runtime crashes.

---

#### Block 1.3: Multi-Channel Dispatch

##### Overview
This block takes the compiled alert text and broadcasts it across multiple external chat and messaging integrations simultaneously via HTTP requests.

##### Nodes Involved
- `Post to Slack Webhook`
- `Post to Discord Webhook`
- `Post to Telegram Bot API`

##### Node Details

- **Post to Slack Webhook**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Version 4.2) — outbound API connector.
  - **Configuration Choices:** Sends a `POST` request to a custom Slack incoming webhook URL with a JSON body parameter named `text`.
  - **Key Expressions or Variables:** 
    - Body parameter value: `={{ $json.message }}`
  - **Input & Output Connections:**
    - Input: Connected from `Build Alert Message`.
    - Output: None (Terminal node).
  - **Edge Cases & Potential Failures:** HTTP timeouts or invalid webhook URLs will fail this specific branch without blocking other notification channels.

- **Post to Discord Webhook**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Version 4.2) — outbound API connector.
  - **Configuration Choices:** Sends a `POST` request to a Discord channel webhook URL using the `content` body parameter.
  - **Key Expressions or Variables:** 
    - Body parameter value: `={{ $json.message }}`
  - **Input & Output Connections:**
    - Input: Connected from `Build Alert Message`.
    - Output: None (Terminal node).
  - **Edge Cases & Potential Failures:** Rate limits imposed by Discord channels on high-frequency transactions.

- **Post to Telegram Bot API**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Version 4.2) — outbound API connector.
  - **Configuration Choices:** Sends a `POST` request to the Telegram Bot API endpoint (`/sendMessage`), passing both the target channel/user `chat_id` and the formatted message `text` in the request body.
  - **Key Expressions or Variables:** 
    - Body parameter value (`text`): `={{ $json.message }}`
    - Body parameter value (`chat_id`): Hardconfigured target ID.
  - **Input & Output Connections:**
    - Input: Connected from `Build Alert Message`.
    - Output: None (Terminal node).
  - **Edge Cases & Potential Failures:** Invalid Telegram Bot tokens or incorrect `chat_id` values result in 401/404 HTTP errors from Telegram.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **When Paddle Sale Alert** | `n8n-nodes-base.webhook` | Webhook Trigger | None | Verify Paddle Signature | Paddle new-sale alerts → Slack, Discord & Telegram (with signature verification)<br><br>### How it works<br><br>This workflow receives Paddle webhook events, verifies their HMAC signature (`Paddle-Signature: ts=…;h1=…`, SHA-256 over `ts:rawBody`, timing-safe compare), and filters for completed sales. When a valid completed sale is detected, it formats the transaction into a concise alert — showing what you actually keep after Paddle's fee, with zero-decimal currencies (JPY, KRW…) handled correctly — and sends it to Slack, Discord and Telegram in parallel.<br><br>Anyone who discovers your webhook URL can POST fake "sales" to it. Paddle signs every request, and this workflow **rejects anything unsigned or forged** — most webhook templates skip this.<br><br>### Setup steps<br><br>Takes ≈5 minutes. Plain HTTP everywhere — no OAuth and no n8n credentials needed.<br><br>- Open **Verify Paddle Signature** and paste your webhook secret. Find it in Paddle → Developer tools → Notifications → your destination → *Edit destination*; it starts with `pdl_ntfset_…`.<br>- Copy this workflow's **Production webhook URL** into Paddle → Developer tools → Notifications → *New destination*, subscribed to `transaction.completed`. Keep the Webhook node's **Raw Body** option enabled — the signature is computed over the raw payload.<br>- **Slack**: create an incoming webhook at api.slack.com/apps and paste its URL into **Post to Slack Webhook**.<br>- **Discord**: channel → Settings → Integrations → Webhooks → paste the URL into **Post to Discord Webhook**.<br>- **Telegram**: create a bot with @BotFather, put its token in the URL of **Post to Telegram Bot API** and your chat id in `chat_id`.<br>- **Delete the channel branches you don't use**, activate the workflow, then send a test event from Paddle's notification screen or make a sandbox purchase.<br><br>### Customization<br><br>Adjust the completed-sale condition or the **Build Alert Message** Code node to include different Paddle fields, currencies, customer details, or channel-specific message text. |
| **Verify Paddle Signature** | `n8n-nodes-base.code` | HMAC-SHA256 Validation | When Paddle Sale Alert | If Sale Completed | Receive and verify webhook<br><br>Accepts incoming Paddle webhook events and verifies the Paddle signature before allowing the payload to continue. |
| **If Sale Completed** | `n8n-nodes-base.if` | Event Filtering | Verify Paddle Signature | Build Alert Message | Filter and format sale<br><br>Checks whether the verified event represents a completed sale, then builds a friendly alert message from the transaction details. |
| **Build Alert Message** | `n8n-nodes-base.code` | Message String Builder | If Sale Completed | Post to Slack Webhook<br>Post to Discord Webhook<br>Post to Telegram Bot API | Filter and format sale<br><br>Checks whether the verified event represents a completed sale, then builds a friendly alert message from the transaction details. |
| **Post to Slack Webhook** | `n8n-nodes-base.httpRequest` | Slack Outbound Notification | Build Alert Message | None | Send channel alerts<br><br>Fans the formatted sale alert out to Slack, Discord and Telegram. The three nodes are independent — delete the ones you don't use. |
| **Post to Discord Webhook** | `n8n-nodes-base.httpRequest` | Discord Outbound Notification | Build Alert Message | None | Send channel alerts<br><br>Fans the formatted sale alert out to Slack, Discord and Telegram. The three nodes are independent — delete the ones you don't use. |
| **Post to Telegram Bot API** | `n8n-nodes-base.httpRequest` | Telegram Outbound Notification | Build Alert Message | None | Send channel alerts<br><br>Fans the formatted sale alert out to Slack, Discord and Telegram. The three nodes are independent — delete the ones you don't use. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `paddle-sale-alerts`.
   - Under **Options**, enable **Raw Body** (`rawBody: true`).
   - Rename the node to `When Paddle Sale Alert`.

2. **Add Signature Verification:**
   - Add a **Code** node connected to the output of `When Paddle Sale Alert`.
   - Rename it to `Verify Paddle Signature`.
   - Insert the cryptographic verification script provided in the node details, ensuring the placeholder `WEBHOOK_SECRET` is updated with your destination secret from Paddle.

3. **Add Event Filtering:**
   - Add an **If** node connected to the output of `Verify Paddle Signature`.
   - Rename it to `If Sale Completed`.
   - Configure a string condition where `{{ $json.event_type }}` equals `transaction.completed`.

4. **Add Message Formatting:**
   - Add a **Code** node connected to the `true` output of `If Sale Completed`.
   - Rename it to `Build Alert Message`.
   - Insert the parsing script that handles zero-decimal currencies (`JPY`, `KRW`, `VND`, `ISK`) and computes net earnings.

5. **Configure Notification Channels (Parallel Branching):**
   - **Slack:** Add an **HTTP Request** node. Set method to `POST`, URL to your Slack incoming webhook endpoint, and add a JSON body parameter `text` set to `={{ $json.message }}`. Connect it to `Build Alert Message`.
   - **Discord:** Add an **HTTP Request** node. Set method to `POST`, URL to your Discord webhook endpoint, and add a JSON body parameter `content` set to `={{ $json.message }}`. Connect it to `Build Alert Message`.
   - **Telegram:** Add an **HTTP Request** node. Set method to `POST`, URL to `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage`, and add body parameters `chat_id` (your target ID) and `text` set to `={{ $json.message }}`. Connect it to `Build Alert Message`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Slack Incoming Webhooks Documentation | [Slack API Apps](https://api.slack.com/apps) |
| Telegram BotFather Setup Instructions | [@BotFather via Telegram](https://t.me/BotFather) |