Run a WooCommerce store assistant with Telegram, OpenRouter and order digests

https://n8nworkflows.xyz/workflows/run-a-woocommerce-store-assistant-with-telegram--openrouter-and-order-digests-19348


# Run a WooCommerce store assistant with Telegram, OpenRouter and order digests

### 1. Workflow Overview

This workflow functions as an intelligent digital store manager for a WooCommerce ebook store. It serves two core operational pathways: an interactive Telegram-based assistant for store owners, and a scheduled daily reporting system for business metrics. 

The workflow is organized into the following logical blocks:
- **1.1 Input Reception & Authorization:** Captures inbound Telegram messages from store owners and verifies their access rights against a secure Google Sheets allowlist.
- **1.2 AI Store Assistant:** Processes authorized requests using an OpenRouter-backed Large Language Model equipped with conversation memory and specialized tools for WooCommerce management, Google Sheets synchronization, and Gmail delivery.
- **1.3 Assistant Response & Error Handling:** Validates the AI agent's output, routes successful responses back to the store owner via Telegram, and triggers administrative alerts on execution failure.
- **1.4 Daily Digest & Reporting:** Runs on a scheduled basis to retrieve recent WooCommerce transactions, compile a 24-hour revenue and order summary, distribute reports via Telegram and Email, and log activity metrics to Google Sheets.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Authorization

##### Overview
This block captures incoming webhook events from Telegram, standardizes essential message parameters, queries a Google Sheets allowlist, and evaluates whether the sender has authorization to interact with the store assistant.

##### Nodes Involved
- `Telegram Message Trigger`
- `Set Message Fields`
- `Read Allowlist from Sheets`
- `Evaluate Access Decision`
- `If Allowed Chat`

##### Node Details

- **Telegram Message Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.telegramTrigger` (Webhook Trigger). Listens for incoming message updates from Telegram.
  - **Configuration:** Configured to watch for `message` updates using the selected Telegram Bot credential.
  - **Key Expressions:** None.
  - **Input/Output:** Input: External Webhook | Output: Raw Telegram message payload.
  - **Edge Cases & Failures:** Webhook delivery failures if Telegram API endpoints change or network interruptions occur.

- **Set Message Fields**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation). Extracts and flattens core parameters from the nested Telegram payload.
  - **Configuration:** Maps payload elements to explicit custom variables.
  - **Key Expressions:** 
    - `chatId`: `={{ $json.message.chat.id }}`
    - `text`: `={{ $json.message.text }}`
    - `firstName`: `={{ $json.message.from.first_name }}`
  - **Input/Output:** Input: `Telegram Message Trigger` | Output: Standardized JSON object containing `chatId`, `text`, and `firstName`.
  - **Edge Cases & Failures:** Expression failure if incoming update format deviates from standard Telegram schemas (e.g., edited messages or service notifications lacking text fields).

- **Read Allowlist from Sheets**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Database Lookup). Queries the Google Sheet configured in environment variables to check authorization.
  - **Configuration:** Uses a dynamic URL lookup (`={{ $env.WOOCOMMERCE_SHEET_URL }}`) targeting the `Allowed Users` sheet. Filters records where `lookupColumn` (`chat_id`) matches the incoming `chatId`.
  - **Key Expressions:** `lookupValue`: `={{ $json.chatId }}`
  - **Input/Output:** Input: `Set Message Fields` | Output: Matching rows from the spreadsheet.
  - **Edge Cases & Failures:** Authentication token expiration, incorrect sheet names, or missing environment variables resulting in empty or failed API calls.

- **Evaluate Access Decision**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Logic). Collapses spreadsheet search results into a definitive boolean access flag.
  - **Configuration:** Custom JavaScript evaluation.
  - **Key Expressions:** Accesses prior node data via `$('Set Message Fields').first().json` and evaluates row array length.
  - **Input/Output:** Input: `Read Allowlist from Sheets` | Output: Single consolidated object containing `chatId`, `text`, `firstName`, and boolean `allowed`.
  - **Edge Cases & Failures:** Unhandled runtime exceptions if upstream nodes return null structures.

- **If Allowed Chat**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Branching). Directs the workflow path based on the evaluation result.
  - **Configuration:** Evaluates if `{{ $json.allowed }}` strictly equals `true`.
  - **Key Expressions:** `={{ $json.allowed }}` equals `true`.
  - **Input/Output:** Input: `Evaluate Access Decision` | Output: True branch proceeds to AI Agent; False branch proceeds to Refusal Message.
  - **Edge Cases & Failures:** Type coercion issues if data types change; strict boolean evaluation mitigates this.

---

#### 1.2 AI Store Assistant

##### Overview
This block powers the conversational intelligence of the workflow, utilizing an OpenRouter LLM model, sliding window conversation memory, and a suite of tools to interact with WooCommerce, Google Sheets, and Gmail.

##### Nodes Involved
- `Store Assistant Agent`
- `AI Chat Router`
- `Store Conversation Memory`
- `Retrieve Orders`
- `Search Products`
- `Update Product Info`
- `Order Database Sync`
- `Dispatch Customer Email`

##### Node Details

- **Store Assistant Agent**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` (AI LangChain Agent). Coordinates user prompts, system instructions, and tool orchestration.
  - **Configuration:** Configured with a defined system message establishing its persona as a digital store manager for a WooCommerce ebook store, supporting English, Arabic, and French, with constraints on pricing modifications and summary formatting.
  - **Key Expressions:** 
    - Text Prompt: `={{ $json.text }}`
    - System Message contains: `{{ $now }}`
  - **Input/Output:** Input: `If Allowed Chat` (True branch) | Output: Agent text response output.
  - **Edge Cases & Failures:** LLM provider rate limits, token length overflows, or tool execution timeouts.

- **AI Chat Router**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (Language Model Service). Connects the agent to the OpenRouter gateway.
  - **Configuration:** Selects the `deepseek/deepseek-chat` model via OpenRouter API credentials.
  - **Input/Output:** Input: Connected to `Store Assistant Agent` as a language model dependency.
  - **Edge Cases & Failures:** API key revocation, quota exhaustion, or upstream service outages.

- **Store Conversation Memory**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Memory Provider). Maintains chat history context across turns.
  - **Configuration:** Configured with a context window length of 20 and a custom session key type.
  - **Input/Output:** Input: Connected to `Store Assistant Agent` as a memory dependency.
  - **Edge Cases & Failures:** Memory fragmentation or session leakage if keys are not scoped per user chat ID.

- **Retrieve Orders**
  - **Type & Technical Role:** `n8n-nodes-base.wooCommerceTool` (AI Tool Integration). Allows the agent to query store orders dynamically.
  - **Configuration:** Operates on the `order` resource with `getAll` operation, utilizing AI override parameters.
  - **Input/Output:** Input: Connected to `Store Assistant Agent` as an AI tool.
  - **Edge Cases & Failures:** WooCommerce API connectivity drops or invalid permissions on consumer keys.

- **Search Products**
  - **Type & Technical Role:** `n8n-nodes-base.wooCommerceTool` (AI Tool Integration). Allows the agent to query catalog items.
  - **Configuration:** Operates on the `product` resource (implied context) with `getAll` operation.
  - **Input/Output:** Input: Connected to `Store Assistant Agent` as an AI tool.
  - **Edge Cases & Failures:** Malformed search queries or API throttling.

- **Update Product Info**
  - **Type & Technical Role:** `n8n-nodes-base.wooCommerceTool` (AI Tool Integration). Empowers the agent to modify catalog pricing, stock levels, and descriptions.
  - **Configuration:** Operates on the `product` resource with `update` operation, mapped via AI parameter overrides for product ID, pricing, and stock metadata.
  - **Input/Output:** Input: Connected to `Store Assistant Agent` as an AI tool.
  - **Edge Cases & Failures:** Unauthorized modification attempts, stock validation errors, or invalid ID exceptions.

- **Order Database Sync**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheetsTool` (AI Tool Integration). Synchronizes order data into a persistent Google Sheets archive.
  - **Configuration:** Uses `appendOrUpdate` operation matching on `Order ID`, mapping fields such as Status, Country, Quantity, and Customer details to the target sheet.
  - **Input/Output:** Input: Connected to `Store Assistant Agent` as an AI tool.
  - **Edge Cases & Failures:** Spreadsheet write locks, range mismatch errors, or invalid data types.

- **Dispatch Customer Email**
  - **Type & Technical Role:** `n8n-nodes-base.gmailTool` (AI Tool Integration). Enables the AI agent to draft and dispatch customer service emails.
  - **Configuration:** Configured with AI parameter mappings for recipient, subject, and message body, with custom sender name metadata.
  - **Input/Output:** Input: Connected to `Store Assistant Agent` as an AI tool.
  - **Edge Cases & Failures:** Gmail OAuth scope limitations, daily sending quotas exceeded, or invalid email address formats.

---

#### 1.3 Assistant Response & Error Handling

##### Overview
This block evaluates the output generated by the AI agent. Successful responses are transmitted back to the store owner via Telegram, while unauthorized requests or processing exceptions are routed to designated fallback handlers and administrative alerts.

##### Nodes Involved
- `If Agent Response`
- `Send Bot Reply to Owner`
- `Send Refusal Message`
- `No Operation`
- `Notify Admin Alert`

##### Node Details

- **If Agent Response**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Branching). Verifies that the AI agent successfully generated an output.
  - **Configuration:** Evaluates whether `{{ $json.output }}` is not empty.
  - **Key Expressions:** `={{ $json.output }}` (Operator: `notEmpty`).
  - **Input/Output:** Input: `Store Assistant Agent` | Output: True branch routes to Telegram reply; False branch routes to admin failure notification.
  - **Edge Cases & Failures:** Empty string responses caused by LLM timeouts or token truncation.

- **Send Bot Reply to Owner**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Messaging Action). Delivers the AI agent's final text response to the user.
  - **Configuration:** Uses Telegram API credentials, setting target chat ID dynamically from the initial intake node.
  - **Key Expressions:** 
    - Text: `={{ $json.output }}`
    - Chat ID: `={{ $('Set Message Fields').first().json.chatId }}`
  - **Input/Output:** Input: `If Agent Response` (True branch) | Output: Sent message confirmation payload.
  - **Edge Cases & Failures:** Telegram API rate limits or blocked bot interactions by the user.

- **Send Refusal Message**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Messaging Action). Sends a polite rejection notice to unauthorized chat IDs.
  - **Configuration:** Sends a static refusal text to the unauthorized sender's `chatId`.
  - **Key Expressions:** Chat ID: `={{ $json.chatId }}`
  - **Input/Output:** Input: `If Allowed Chat` (False branch) | Output: Refusal message delivery confirmation.
  - **Edge Cases & Failures:** Invalid chat ID format or network timeout.

- **No Operation**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` (Flow Termination). Acts as a terminal node for the unauthorized path.
  - **Configuration:** None.
  - **Input/Output:** Input: `Send Refusal Message` | Output: Terminates execution path.
  - **Edge Cases & Failures:** None.

- **Notify Admin Alert**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Messaging Action). Alerts the system administrator of processing errors or empty agent responses.
  - **Configuration:** Sends an alert message to the administrator chat ID defined in environment variables.
  - **Key Expressions:** Chat ID: `={{ $env.ADMIN_TELEGRAM_CHAT_ID }}`
  - **Input/Output:** Input: `If Agent Response` (False branch) or `Route by Digest Available` (Failure branch) | Output: Admin notification dispatch.
  - **Edge Cases & Failures:** Invalid admin chat ID environment variable.

---

#### 1.4 Daily Digest & Reporting

##### Overview
This block executes on a daily schedule to fetch recent store transactions, filter orders from the preceding 24 hours, generate a performance digest, distribute it via Telegram and Gmail, record metrics in a Google Sheets activity log, and manage error handling.

##### Nodes Involved
- `Trigger Daily Digest`
- `Fetch Latest Orders`
- `Filter Recent Orders`
- `Route by Digest Available`
- `Send Order Digest Message`
- `Email Daily Digest`
- `Log Digest Activity`

##### Node Details

- **Trigger Daily Digest**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Scheduled Trigger). Initiates the reporting workflow at a fixed interval.
  - **Configuration:** Cron expression set to run daily at 09:00 (`0 9 * * *`).
  - **Input/Output:** Input: Cron Timer | Output: Execution timestamp.
  - **Edge Cases & Failures:** Server time zone misconfigurations altering execution hours.

- **Fetch Latest Orders**
  - **Type & Technical Role:** `n8n-nodes-base.wooCommerce` (Data Retrieval). Fetches orders from the WooCommerce store instance.
  - **Configuration:** Operates on resource `order` with operation `get` (retrieving order datasets).
  - **Input/Output:** Input: `Trigger Daily Digest` | Output: Array of raw WooCommerce order objects.
  - **Edge Cases & Failures:** WooCommerce API timeout or credential rejection.

- **Filter Recent Orders**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation). Filters orders created within the last 24 hours, calculates total revenue, and formats digest text.
  - **Configuration:** Custom JavaScript calculating a 24-hour rolling cutoff timestamp, filtering array elements, summing revenues, and formatting summary strings.
  - **Key Expressions:** Utilizes JavaScript `Date.now()` and array reduction methods.
  - **Input/Output:** Input: `Fetch Latest Orders` | Output: Consolidated object with `orderCount`, `totalRevenue`, `digestText`, `status`, and `runDate`.
  - **Edge Cases & Failures:** Empty upstream datasets or malformed date strings handled via try/catch blocks returning a `failed` status.

- **Route by Digest Available**
  - **Type & Technical Role:** `n8n-nodes-base.switch` (Conditional Routing). Directs the workflow based on whether orders were found or if a processing failure occurred.
  - **Configuration:** Rule 1 checks if `orderCount >= 1`; Rule 2 checks if `status == 'failed'`.
  - **Key Expressions:** `={{ $json.orderCount }}` and `={{ $json.status }}`.
  - **Input/Output:** Input: `Filter Recent Orders` | Output: Branch 1 routes to digest delivery; Branch 2 routes to admin alert.
  - **Edge Cases & Failures:** Unhandled status values falling through routing rules.

- **Send Order Digest Message**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Messaging Action). Dispatches the formatted text digest to the administrator via Telegram.
  - **Configuration:** Sends message content to the admin chat ID.
  - **Key Expressions:** 
    - Text: `={{ $json.digestText }}`
    - Chat ID: `={{ $env.ADMIN_TELEGRAM_CHAT_ID }}`
  - **Input/Output:** Input: `Route by Digest Available` (Branch 1) | Output: Sent message payload.
  - **Edge Cases & Failures:** Telegram API transmission failures.

- **Email Daily Digest**
  - **Type & Technical Role:** `n8n-nodes-base.emailSend` (Email Action). Sends the daily digest via SMTP email.
  - **Configuration:** Configured with SMTP credentials, subject line, and recipient/sender variables from environment settings.
  - **Key Expressions:** 
    - To Email: `={{ $env.ADMIN_EMAIL }}`
    - From Email: `={{ $env.ADMIN_EMAIL }}`
  - **Input/Output:** Input: `Send Order Digest Message` | Output: Email dispatch confirmation.
  - **Edge Cases & Failures:** SMTP authentication failures or mail server rate limitations.

- **Log Digest Activity**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Database Logging). Appends execution metrics to the Google Sheets activity log.
  - **Configuration:** Targets the `Digest Log` sheet using `WOOCOMMERCE_SHEET_URL`, appending `status`, `revenue`, `run_date`, and `order_count`.
  - **Key Expressions:** Mapping values from input JSON properties.
  - **Input/Output:** Input: `Email Daily Digest` | Output: Spreadsheet row addition confirmation.
  - **Edge Cases & Failures:** Missing worksheet headers or sheet lock exceptions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Workflow documentation and setup instructions | None | None | woocommerce store assistant with order alerts. How it works, setup steps, and customization notes. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Section documentation for intake | None | None | Owner message intake: Receives incoming Telegram messages from the store owner and normalizes key fields. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Section documentation for authorization | None | None | Allowlist authorization: Looks up the sender in Google Sheets, collapses the lookup, and branches the workflow. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Section documentation for AI assistant | None | None | AI store assistant: Processes authorized owner requests with an AI agent backed by OpenRouter and conversation memory. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Section documentation for refusal | None | None | Unauthorized chat response: Handles the denied-access branch by sending a polite Telegram refusal and ending with a no-op. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Section documentation for response handling | None | None | Assistant response handling: Checks agent responses, sends successful replies, and triggers admin alerts on failure. |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Section documentation for digest fetch | None | None | Digest order fetch: Starts the scheduled daily digest path and retrieves WooCommerce orders. |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Section documentation for filtering | None | None | Fresh order filtering: Filters fetched orders to the last 24 hours, builds a payload, and routes via switch. |
| Sticky Note8 | `n8n-nodes-base.stickyNote` | Section documentation for logging | None | None | Digest delivery logging: Sends the daily order digest through Telegram and email, then records the run in Google Sheets. |
| Telegram Message Trigger | `n8n-nodes-base.telegramTrigger` | Webhook trigger for inbound Telegram messages | External Webhook | Set Message Fields | |
| Set Message Fields | `n8n-nodes-base.set` | Normalizes chat ID, text, and sender name | Telegram Message Trigger | Read Allowlist from Sheets | |
| Read Allowlist from Sheets | `n8n-nodes-base.googleSheets` | Queries Google Sheets allowlist by chat ID | Set Message Fields | Evaluate Access Decision | |
| Evaluate Access Decision | `n8n-nodes-base.code` | Evaluates spreadsheet match into a boolean flag | Read Allowlist from Sheets | If Allowed Chat | |
| If Allowed Chat | `n8n-nodes-base.if` | Branches execution based on authorization status | Evaluate Access Decision | Store Assistant Agent, Send Refusal Message | |
| Send Refusal Message | `n8n-nodes-base.telegram` | Sends rejection notice to unauthorized users | If Allowed Chat | No Operation | |
| No Operation | `n8n-nodes-base.noOp` | Terminates unauthorized execution branch | Send Refusal Message | None | |
| Store Assistant Agent | `@n8n/n8n-nodes-langchain.agent` | AI agent managing store operations | If Allowed Chat | If Agent Response | |
| AI Chat Router | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | OpenRouter language model provider | None (AI Dependency) | Store Assistant Agent | |
| Store Conversation Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Sliding window chat memory buffer | None (AI Dependency) | Store Assistant Agent | |
| Retrieve Orders | `n8n-nodes-base.wooCommerceTool` | AI tool to fetch WooCommerce orders | None (AI Dependency) | Store Assistant Agent | |
| Search Products | `n8n-nodes-base.wooCommerceTool` | AI tool to search WooCommerce catalog | None (AI Dependency) | Store Assistant Agent | |
| Update Product Info | `n8n-nodes-base.wooCommerceTool` | AI tool to modify product details | None (AI Dependency) | Store Assistant Agent | |
| Order Database Sync | `n8n-nodes-base.googleSheetsTool` | AI tool to sync orders to Google Sheets | None (AI Dependency) | Store Assistant Agent | |
| Dispatch Customer Email | `n8n-nodes-base.gmailTool` | AI tool to send customer emails via Gmail | None (AI Dependency) | Store Assistant Agent | |
| If Agent Response | `n8n-nodes-base.if` | Verifies that the AI agent produced an output | Store Assistant Agent | Send Bot Reply to Owner, Notify Admin Alert | |
| Send Bot Reply to Owner | `n8n-nodes-base.telegram` | Sends successful AI responses back to the user | If Agent Response | None | |
| Trigger Daily Digest | `n8n-nodes-base.scheduleTrigger` | Cron trigger for daily reporting at 09:00 | Cron Timer | Fetch Latest Orders | |
| Filter Recent Orders | `n8n-nodes-base.code` | Filters 24-hour orders and builds report payload | Fetch Latest Orders | Route by Digest Available | |
| Route by Digest Available | `n8n-nodes-base.switch` | Routes report based on order volume or failure | Filter Recent Orders | Send Order Digest Message, Notify Admin Alert | |
| Send Order Digest Message | `n8n-nodes-base.telegram` | Sends daily order digest to admin via Telegram | Route by Digest Available | Email Daily Digest | |
| Email Daily Digest | `n8n-nodes-base.emailSend` | Sends daily digest via SMTP email | Send Order Digest Message | Log Digest Activity | |
| Log Digest Activity | `n8n-nodes-base.googleSheets` | Records digest run metrics in Google Sheets | Email Daily Digest | None | |
| Notify Admin Alert | `n8n-nodes-base.telegram` | Sends system failure alerts to administrator | If Agent Response, Route by Digest Available | None | |
| Fetch Latest Orders | `n8n-nodes-base.wooCommerce` | Retrieves order records from WooCommerce | Trigger Daily Digest | Filter Recent Orders | |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the entire workflow manually in n8n:

1. **Create Environment Variables:** Define the following variables in your n8n instance settings:
   - `WOOCOMMERCE_SHEET_URL`: URL of your Google Sheets document containing the allowed users list and digest log.
   - `ADMIN_TELEGRAM_CHAT_ID`: Telegram chat ID for administrative alerts and digest delivery.
   - `ADMIN_EMAIL`: Email address for receiving daily digests and alerts.

2. **Set Up Credentials:** Configure credentials for:
   - Telegram API (Bot token)
   - Google Sheets OAuth2 / Service Account
   - WooCommerce API (Consumer Key and Secret)
   - OpenRouter API
   - Gmail OAuth2 or SMTP Account

3. **Build the Telegram Intake & Authorization Branch:**
   - Add a **Telegram Message Trigger** node. Set updates to `message`.
   - Add a **Set** node named `Set Message Fields`. Assign `chatId` (`={{ $json.message.chat.id }}`), `text` (`={{ $json.message.text }}`), and `firstName` (`={{ $json.message.from.first_name }}`). Connect trigger output here.
   - Add a **Google Sheets** node named `Read Allowlist from Sheets`. Set document ID to `={{ $env.WOOCOMMERCE_SHEET_URL }}`, sheet name to `Allowed Users`, and add a filter where `chat_id` equals `={{ $json.chatId }}`. Connect `Set Message Fields` output here.
   - Add a **Code** node named `Evaluate Access Decision`. Paste the JavaScript logic to evaluate matching rows and output an `allowed` boolean flag. Connect allowlist output here.
   - Add an **If** node named `If Allowed Chat`. Set condition to evaluate if `={{ $json.allowed }}` is `true`. Connect code node output here.
   - *Refusal Path:* Connect the false branch to a **Telegram** node named `Send Refusal Message` configured with a denial text and destination `={{ $json.chatId }}`, followed by a **No Operation** node.

4. **Build the AI Store Assistant Branch:**
   - Connect the true branch of `If Allowed Chat` to an **AI Agent** node named `Store Assistant Agent`. Set prompt type to define text `={{ $json.text }}` and provide the digital store manager system instructions including `{{ $now }}`.
   - Connect an **OpenRouter Chat Model** (`AI Chat Router`) to the agent's model input. Select model `deepseek/deepseek-chat`.
   - Connect a **Window Buffer Memory** (`Store Conversation Memory`) to the agent's memory input with context window length set to `20`.
   - Attach the following tools to the agent's tool inputs:
     - **WooCommerce Tool** (`Retrieve Orders`): Resource `order`, operation `getAll`.
     - **WooCommerce Tool** (`Search Products`): Resource `product`, operation `getAll`.
     - **WooCommerce Tool** (`Update Product Info`): Resource `product`, operation `update`, mapping metadata and pricing fields via AI overrides.
     - **Google Sheets Tool** (`Order Database Sync`): Operation `appendOrUpdate`, matching on `Order ID`.
     - **Gmail Tool** (`Dispatch Customer Email`): Mapping recipient, subject, and message fields via AI overrides.

5. **Build the Assistant Response & Error Handling Branch:**
   - Connect the output of `Store Assistant Agent` to an **If** node named `If Agent Response`. Set condition to verify `={{ $json.output }}` is not empty.
   - Connect the true branch to a **Telegram** node named `Send Bot Reply to Owner`. Set text to `={{ $json.output }}` and chat ID to `={{ $('Set Message Fields').first().json.chatId }}`.
   - Connect the false branch to a **Telegram** node named `Notify Admin Alert`. Set text to an error notification and chat ID to `={{ $env.ADMIN_TELEGRAM_CHAT_ID }}`.

6. **Build the Daily Digest & Reporting Branch:**
   - Add a **Schedule Trigger** node named `Trigger Daily Digest`. Set cron expression to `0 9 * * *`.
   - Connect to a **WooCommerce** node named `Fetch Latest Orders` (Resource: `order`, Operation: `get`).
   - Connect to a **Code** node named `Filter Recent Orders`. Paste the 24-hour revenue calculation and payload formatting script.
   - Connect to a **Switch** node named `Route by Digest Available`. Set Rule 1 for `orderCount >= 1` and Rule 2 for `status == 'failed'`.
   - Connect Rule 1 output to a **Telegram** node named `Send Order Digest Message`. Set text to `={{ $json.digestText }}` and chat ID to `={{ $env.ADMIN_TELEGRAM_CHAT_ID }}`.
   - Connect the Telegram node to an **Email Send** / SMTP node named `Email Daily Digest`. Set recipient and sender to `={{ $env.ADMIN_EMAIL }}` and subject to `Daily WooCommerce order digest`.
   - Connect the email node to a **Google Sheets** node named `Log Digest Activity`. Set document ID to `={{ $env.WOOCOMMERCE_SHEET_URL }}`, sheet name to `Digest Log`, operation to `append`, and map status, revenue, run date, and order count columns.
   - Connect Rule 2 output of the switch node to the existing `Notify Admin Alert` Telegram node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| WooCommerce store assistant with order alerts overview, setup instructions, and customization guidelines. | Main workflow sticky note documentation inside the n8n canvas. |