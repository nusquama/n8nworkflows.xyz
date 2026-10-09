Track bank SMS expenses with Google Sheets, Telegram and Claude

https://n8nworkflows.xyz/workflows/track-bank-sms-expenses-with-google-sheets--telegram-and-claude-20482


# Track bank SMS expenses with Google Sheets, Telegram and Claude

### 1. Workflow Overview

This workflow functions as an automated personal expense tracking and financial assistant system named **CashLens**. It captures incoming bank SMS alerts via an authenticated webhook, normalizes and parses the text into structured transaction data, and securely logs them into Google Sheets. Additionally, it features an interactive Telegram bot powered by an Anthropic Claude AI agent with memory and tools, allowing users to query their financial ledgers and review queues using natural language.

The operational logic is organized into two primary functional blocks:

- **1.1 Record Bank Transactions:** Receives raw SMS payloads through a secured HTTP endpoint, processes and validates the text using custom JavaScript rules, filters out OTPs and unlisted senders, and routes the data to either the main transaction ledger, a manual review queue, or an HTTP 422 rejection response.
- **1.2 Personal Finance Assistant:** Captures user queries via Telegram, leverages an advanced AI agent configured with Anthropic Claude, short-term conversation memory, and Google Sheets tools to fetch financial records, executes exact arithmetic calculations via a calculator tool, and replies with formatted plain-text responses directly in the Telegram chat.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Record Bank Transactions

#### Overview
This block intercepts forwarded bank SMS text via webhook, executes comprehensive parsing logic to extract financial fields (amount, transaction type, date, merchant, payment method, account details, and reference IDs), and conditionally writes records to Google Sheets or rejects invalid inputs.

#### Nodes Involved
- `Receive Bank SMS`
- `Parse & Normalize Transaction`
- `Is Transaction Valid?`
- `Save to Transaction Ledger`
- `Confirm Transaction Saved`
- `Needs Manual Review?`
- `Save to Review Queue`
- `Confirm Review Queue`
- `Reject Invalid Transaction`

#### Node Details

##### Receive Bank SMS
- **Type and Technical Role:** `n8n-nodes-base.webhook` — Acts as the secure ingestion point for bank SMS messages sent via HTTP POST requests.
- **Configuration Choices:** Configured with path `cashlens-sms`, response mode set to use a response node, and HTTP Header Authentication enabled.
- **Key Expressions or Variables:** N/A (Endpoint ingestion point).
- **Input and Output Connections:** 
  - Input: External HTTP POST requests.
  - Output: Connects to `Parse & Normalize Transaction`.
- **Version-specific Requirements:** Version 2.1.
- **Edge Cases / Potential Failure Types:** Authentication failures return 401/403; malformed or missing payloads must be handled downstream.

##### Parse & Normalize Transaction
- **Type and Technical Role:** `n8n-nodes-base.code` — Custom JavaScript block executing regular expressions and conditional logic to parse bank SMS strings.
- **Configuration Choices:** Executes JavaScript operating on `$input.all()`, extracting amounts, identifying debit/credit movements, extracting localized dates (supporting standard formats, Kotak Bank DD-MM-YY formats, and Unix timestamp fallbacks), extracting payment rails (UPI, IMPS, NEFT, RTGS, ATM), counterparties, reference IDs, and masked account numbers.
- **Key Expressions or Variables:** Processes `item.json.body ?? item.json`, referencing fields like `message`, `sender`, `sent_at`, and `event_id`.
- **Input and Output Connections:** 
  - Input: `Receive Bank SMS`.
  - Output: Connects to `Is Transaction Valid?`.
- **Version-specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Unrecognized bank text formats or edge cases in SMS layouts may yield null values, which are caught by the validation step and routed to review or rejection queues.

##### Is Transaction Valid?
- **Type and Technical Role:** `n8n-nodes-base.if` — Conditional router checking whether a processed transaction meets all validation requirements.
- **Configuration Choices:** Evaluates boolean condition based on `$json.is_valid_transaction`.
- **KeyExpressions or Variables:** `={{ $json.is_valid_transaction }}`
- **Input and Output Connections:** 
  - Input: `Parse & Normalize Transaction`.
  - Output (True): `Save to Transaction Ledger`.
  - Output (False): `Needs Manual Review?`.
- **Version-specific Requirements:** Version 2.3.
- **Edge Cases / Potential Failure Types:** Type mismatch or undefined boolean flags evaluate to false, correctly triggering the secondary review path.

##### Save to Transaction Ledger
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` — Inserts or updates validated financial records into the main database sheet.
- **Configuration Choices:** Operation set to `appendOrUpdate`, targeting document ID `1F6x02uqmFqE4wtn12-nvOk0ZFkXO6CJbkGItIEzJ_yY`, sheet name `Expenses`, matching rows based on `source_event_id`.
- **Key Expressions or Variables:** Auto-maps input data mapping mode.
- **Input and Output Connections:** 
  - Input: `Is Transaction Valid?` (True branch).
  - Output: Connects to `Confirm Transaction Saved`.
- **Version-specific Requirements:** Version 4.7. Requires valid Google Sheets OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** API rate limits, revoked authentication tokens, or schema mismatches between incoming data and spreadsheet columns.

##### Confirm Transaction Saved
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` — Returns an HTTP 200 confirmation payload back to the SMS forwarder client.
- **Configuration Choices:** Response code set to 200, respond with JSON containing success status and message.
- **Key Expressions or Variables:** Static JSON response body.
- **Input and Output Connections:** 
  - Input: `Save to Transaction Ledger`.
  - Output: End of branch.
- **Version-specific Requirements:** Version 1.5.

##### Needs Manual Review?
- **Type and Technical Role:** `n8n-nodes-base.if` — Conditional router identifying if a rejected or incomplete SMS requires human intervention or complete rejection.
- **Configuration Choices:** Evaluates string condition matching `NEEDS_REVIEW` against `$json.needs_manual_review`.
- **Key Expressions or Variables:** `={{ $json.needs_manual_review }}`
- **Input and Output Connections:** 
  - Input: `Is Transaction Valid?` (False branch).
  - Output (True): `Save to Review Queue`.
  - Output (False): `Reject Invalid Transaction`.
- **Version-specific Requirements:** Version 2.3.

##### Save to Review Queue
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` — Logs incomplete or unparsable SMS messages into a dedicated staging tab.
- **Configuration Choices:** Operation set to `appendOrUpdate`, targeting sheet `Review_Queue`, matching rows based on `source_event_id`.
- **Key Expressions or Variables:** Auto-maps input data.
- **Input and Output Connections:** 
  - Input: `Needs Manual Review?` (True branch).
  - Output: Connects to `Confirm Review Queue`.
- **Version-specific Requirements:** Version 4.7.

##### Confirm Review Queue
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` — Returns an HTTP 200 JSON confirmation indicating the item has been staged for manual review.
- **Configuration Choices:** Response code set to 200, response body configured with structured status flags.
- **Key Expressions or Variables:** Static JSON response body.
- **Input and Output Connections:** 
  - Input: `Save to Review Queue`.
  - Output: End of branch.
- **Version-specific Requirements:** Version 1.5.

##### Reject Invalid Transaction
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` — Returns an HTTP 422 Unprocessable Entity response for OTPs or unauthorized senders.
- **Configuration Choices:** Response code set to 422, response body contains rejection metadata.
- **Key Expressions or Variables:** Static JSON response body.
- **Input and Output Connections:** 
  - Input: `Needs Manual Review?` (False branch).
  - Output: End of branch.
- **Version-specific Requirements:** Version 1.5.

---

### Block 1.2: Personal Finance Assistant

#### Overview
This block listens for incoming Telegram chat messages, processes questions using an advanced LangChain AI agent powered by Anthropic Claude, queries Google Sheets ledger tools and performs mathematical computations, and replies directly to the user in Telegram.

#### Nodes Involved
- `Telegram Message`
- `CashLens Financial Assistant`
- `Claude Model`
- `Conversation Memory`
- `get_transactions`
- `get_review_queue`
- `calculator`
- `Send Telegram Reply`

#### Node Details

##### Telegram Message
- **Type and Technical Role:** `n8n-nodes-base.telegramTrigger` — Triggers the workflow whenever a new message is received by the Telegram bot.
- **Configuration Choices:** Updates parameter set to listen for `message` events.
- **Key Expressions or Variables:** N/A (Trigger node).
- **Input and Output Connections:** 
  - Input: Telegram user interactions.
  - Output: Connects to `CashLens Financial Assistant`.
- **Version-specific Requirements:** Version 1.5. Requires configured Telegram API credentials.
- **Edge Cases / Potential Failure Types:** Webhook registration conflicts if multiple bot instances run simultaneously.

##### CashLens Financial Assistant
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.agent` — Advanced AI Agent orchestrating tools, memory, and LLM reasoning.
- **Configuration Choices:** Text input mapped to incoming message, max iterations set to 10, prompt type defined with explicit system instructions regarding INR currency formatting, tool calling hierarchies, and strict plain-text response constraints (no markdown headers, tables, or asterisks).
- **Key Expressions or Variables:** `={{ $json.message.text }}` and `={{ $now.setZone("Asia/Kolkata").toFormat("cccc, dd LLL yyyy") }}`.
- **Input and Output Connections:** 
  - Input: `Telegram Message`, connected to AI Language Model (`Claude Model`), AI Memory (`Conversation Memory`), and AI Tools (`calculator`, `get_transactions`, `get_review_queue`).
  - Output: Connects to `Send Telegram Reply`.
- **Version-specific Requirements:** Version 3.1.

##### Claude Model
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — Provides the foundational LLM intelligence utilizing Claude Sonnet.
- **Configuration Choices:** Model set to `claude-sonnet-5`, maximum tokens to sample configured at 2048.
- **Key Expressions or Variables:** N/A.
- **Input and Output Connections:** Connects as an AI language model provider to `CashLens Financial Assistant`.
- **Version-specific Requirements:** Version 1.6. Requires Anthropic API credentials.

##### Conversation Memory
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Maintains short-term conversational context across message exchanges.
- **Configuration Choices:** Session key set using custom key pattern per Telegram chat ID, context window length set to 10 messages.
- **Key Expressions or Variables:** `=cashlens-{{ $("Telegram Message").item.json.message.chat.id }}`.
- **Input and Output Connections:** Connects as an AI memory provider to `CashLens Financial Assistant`.
- **Version-specific Requirements:** Version 1.4.

##### get_transactions
- **Type and Technical Role:** `n8n-nodes-base.googleSheetsTool` — Exposes the main transaction ledger to the AI agent as a tool.
- **Configuration Choices:** Targets document ID `1F6x02uqmFqE4wtn12-nvOk0ZFkXO6CJbkGItIEzJ_yY`, sheet `Expenses` (gid=0), with descriptive manual tool explanation specifying column structures and debit/credit definitions.
- **Key Expressions or Variables:** N/A.
- **Input and Output Connections:** Connects as an AI tool to `CashLens Financial Assistant`.
- **Version-specific Requirements:** Version 4.7.

##### get_review_queue
- **Type and Technical Role:** `n8n-nodes-base.googleSheetsTool` — Exposes the review queue staging sheet to the AI agent as a tool.
- **Configuration Choices:** Targets document ID `1F6x02uqmFqE4wtn12-nvOk0ZFkXO6CJbkGItIEzJ_yY`, sheet `Review_Queue`, with descriptive manual tool explanation for unparsed bank SMS alerts.
- **Key Expressions or Variables:** N/A.
- **Input and Output Connections:** Connects as an AI tool to `CashLens Financial Assistant`.
- **Version-specific Requirements:** Version 4.7.

##### calculator
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.toolCalculator` — Provides exact arithmetic computation capabilities to the agent.
- **Configuration Choices:** Standard default parameters.
- **Key Expressions or Variables:** N/A.
- **Input and Output Connections:** Connects as an AI tool to `CashLens Financial Assistant`.
- **Version-specific Requirements:** Version 1.

##### Send Telegram Reply
- **Type and Technical Role:** `n8n-nodes-base.telegram` — Sends the agent's final text response back to the user via Telegram.
- **Configuration Choices:** Action set to send message, chat ID and reply-to message ID mapped dynamically from the trigger payload, attribution disabled.
- **Key Expressions or Variables:** 
  - Text: `={{ $json.output }}`
  - Chat ID: `={{ $("Telegram Message").item.json.message.chat.id }}`
  - Additional Fields (Reply To): `={{ $("Telegram Message").item.json.message.message_id }}`
- **Input and Output Connections:** 
  - Input: `CashLens Financial Assistant`.
  - Output: End of workflow branch.
- **Version-specific Requirements:** Version 1.2. Requires valid Telegram API credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Receive Bank SMS` | `n8n-nodes-base.webhook` | Ingests forwarded bank SMS messages via HTTP POST | None (Webhook) | `Parse & Normalize Transaction` | ## CashLens: Bank SMS expense tracker and Telegram finance assistant<br><br>Turns your bank SMS alerts into a clean transaction ledger in Google Sheets. You can then ask a Telegram bot questions about your spending in plain language.<br><br>**Who it's for:** anyone in India who wants automatic expense tracking from bank SMS without connecting a bank account or a paid app.<br><br>### How it works<br>1. An SMS forwarder app on your Android phone POSTs each bank SMS to the webhook.<br>2. The parser pulls out amount, debit/credit, date, merchant, payment method, account last 4 and reference ID. It also rejects OTPs and unknown senders.<br>3. Complete transactions go to the **Expenses** tab. Incomplete ones go to **Review_Queue**. Rejected ones get a 422 response.<br>4. On Telegram, an AI agent reads the sheet, calculates totals and trends, and replies in the same chat.<br><br>### Setup steps<br>1. Make a Google Sheet with two tabs, **Expenses** and **Review_Queue**, using the column names from the parser output. Then select the sheet in all four Google Sheets nodes.<br>2. Create a Header Auth credential for the webhook. Have your SMS forwarder send the same header with `message`, `sender`, `event_id` and `sent_at` in the body.<br>3. In **Parse & Normalize Transaction**, add your bank's sender IDs to `allowedSenders`.<br>4. Create a bot with @BotFather and connect it in the Telegram trigger and the reply node.<br>5. Connect an Anthropic credential to the chat model, then publish.<br><br>### Customization<br>- Add more SMS formats to the parser for other banks.<br>- Replace UNCATEGORIZED with your own category rules.<br>- Edit the agent's system message to change its tone or rules. |
| `Parse & Normalize Transaction` | `n8n-nodes-base.code` | Parses SMS text into structured financial properties | `Receive Bank SMS` | `Is Transaction Valid?` | ## 1. Record bank transactions<br>Receives each forwarded SMS through a secured webhook and parses it into structured fields. Then it routes it:<br>- **Valid**: upserted into the ledger (de-duplicated on `source_event_id`)<br>- **Incomplete**: saved to the review queue<br>- **OTP or unknown sender**: rejected<br><br>Every path returns a JSON response to the SMS forwarder. |
| `Is Transaction Valid?` | `n8n-nodes-base.if` | Routes valid transactions versus incomplete/invalid records | `Parse & Normalize Transaction` | `Save to Transaction Ledger`, `Needs Manual Review?` | ## 1. Record bank transactions<br>Receives each forwarded SMS through a secured webhook and parses it into structured fields. Then it routes it:<br>- **Valid**: upserted into the ledger (de-duplicated on `source_event_id`)<br>- **Incomplete**: saved to the review queue<br>- **OTP or unknown sender**: rejected<br><br>Every path returns a JSON response to the SMS forwarder. |
| `Save to Transaction Ledger` | `n8n-nodes-base.googleSheets` | Appends or updates valid transactions in the Expenses sheet | `Is Transaction Valid?` | `Confirm Transaction Saved` | ## 1. Record bank transactions<br>Receives each forwarded SMS through a secured webhook and parses it into structured fields. Then it routes it:<br>- **Valid**: upserted into the ledger (de-duplicated on `source_event_id`)<br>- **Incomplete**: saved to the review queue<br>- **OTP or unknown sender**: rejected<br><br>Every path returns a JSON response to the SMS forwarder. |
| `Confirm Transaction Saved` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 200 JSON confirmation for saved transactions | `Save to Transaction Ledger` | None | ## 1. Record bank transactions<br>Receives each forwarded SMS through a secured webhook and parses it into structured fields. Then it routes it:<br>- **Valid**: upserted into the ledger (de-duplicated on `source_event_id`)<br>- **Incomplete**: saved to the review queue<br>- **OTP or unknown sender**: rejected<br><br>Every path returns a JSON response to the SMS forwarder. |
| `Needs Manual Review?` | `n8n-nodes-base.if` | Determines if unparsed items go to review or rejection | `Is Transaction Valid?` | `Save to Review Queue`, `Reject Invalid Transaction` | ## 1. Record bank transactions<br>Receives each forwarded SMS through a secured webhook and parses it into structured fields. Then it routes it:<br>- **Valid**: upserted into the ledger (de-duplicated on `source_event_id`)<br>- **Incomplete**: saved to the review queue<br>- **OTP or unknown sender**: rejected<br><br>Every path returns a JSON response to the SMS forwarder. |
| `Save to Review Queue` | `n8n-nodes-base.googleSheets` | Logs incomplete messages into the Review_Queue sheet | `Needs Manual Review?` | `Confirm Review Queue` | ## 1. Record bank transactions<br>Receives each forwarded SMS through a secured webhook and parses it into structured fields. Then it routes it:<br>- **Valid**: upserted into the ledger (de-duplicated on `source_event_id`)<br>- **Incomplete**: saved to the review queue<br>- **OTP or unknown sender**: rejected<br><br>Every path returns a JSON response to the SMS forwarder. |
| `Confirm Review Queue` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 200 JSON confirmation for review queue staging | `Save to Review Queue` | None | ## 1. Record bank transactions<br>Receives each forwarded SMS through a secured webhook and parses it into structured fields. Then it routes it:<br>- **Valid**: upserted into the ledger (de-duplicated on `source_event_id`)<br>- **Incomplete**: saved to the review queue<br>- **OTP or unknown sender**: rejected<br><br>Every path returns a JSON response to the SMS forwarder. |
| `Reject Invalid Transaction` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 422 JSON response for rejected messages | `Needs Manual Review?` | None | ## 1. Record bank transactions<br>Receives each forwarded SMS through a secured webhook and parses it into structured fields. Then it routes it:<br>- **Valid**: upserted into the ledger (de-duplicated on `source_event_id`)<br>- **Incomplete**: saved to the review queue<br>- **OTP or unknown sender**: rejected<br><br>Every path returns a JSON response to the SMS forwarder. |
| `Telegram Message` | `n8n-nodes-base.telegramTrigger` | Triggers workflow on incoming Telegram messages | None (Trigger) | `CashLens Financial Assistant` | ## 2. Personal finance assistant<br>Answers Telegram questions such as "How much did I spend on Swiggy last month?" It reads the ledger and review queue (read-only), uses the calculator for exact totals, and remembers the last 10 messages per chat.<br><br>Replies come back as plain text in the same chat. |
| `CashLens Financial Assistant` | `@n8n/n8n-nodes-langchain.agent` | Orchestrates LLM, memory, and tool calls for chat queries | `Telegram Message`, `Claude Model`, `Conversation Memory`, `get_transactions`, `get_review_queue`, `calculator` | `Send Telegram Reply` | ## 2. Personal finance assistant<br>Answers Telegram questions such as "How much did I spend on Swiggy last month?" It reads the ledger and review queue (read-only), uses the calculator for exact totals, and remembers the last 10 messages per chat.<br><br>Replies come back as plain text in the same chat. |
| `Claude Model` | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Provides Claude Sonnet language model capability | None (AI Model) | `CashLens Financial Assistant` | ## 2. Personal finance assistant<br>Answers Telegram questions such as "How much did I spend on Swiggy last month?" It reads the ledger and review queue (read-only), uses the calculator for exact totals, and remembers the last 10 messages per chat.<br><br>Replies come back as plain text in the same chat. |
| `Conversation Memory` | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Maintains sliding window conversation history per chat | None (AI Memory) | `CashLens Financial Assistant` | ## 2. Personal finance assistant<br>Answers Telegram questions such as "How much did I spend on Swiggy last month?" It reads the ledger and review queue (read-only), uses the calculator for exact totals, and remembers the last 10 messages per chat.<br><br>Replies come back as plain text in the same chat. |
| `get_transactions` | `n8n-nodes-base.googleSheetsTool` | Exposes the Expenses sheet as a tool to the AI agent | None (AI Tool) | `CashLens Financial Assistant` | ## 2. Personal finance assistant<br>Answers Telegram questions such as "How much did I spend on Swiggy last month?" It reads the ledger and review queue (read-only), uses the calculator for exact totals, and remembers the last 10 messages per chat.<br><br>Replies come back as plain text in the same chat. |
| `get_review_queue` | `n8n-nodes-base.googleSheetsTool` | Exposes the Review_Queue sheet as a tool to the AI agent | None (AI Tool) | `CashLens Financial Assistant` | ## 2. Personal finance assistant<br>Answers Telegram questions such as "How much did I spend on Swiggy last month?" It reads the ledger and review queue (read-only), uses the calculator for exact totals, and remembers the last 10 messages per chat.<br><br>Replies come back as plain text in the same chat. |
| `calculator` | `@n8n/n8n-nodes-langchain.toolCalculator` | Provides mathematical computation tool for precise sums | None (AI Tool) | `CashLens Financial Assistant` | ## 2. Personal finance assistant<br>Answers Telegram questions such as "How much did I spend on Swiggy last month?" It reads the ledger and review queue (read-only), uses the calculator for exact totals, and remembers the last 10 messages per chat.<br><br>Replies come back as plain text in the same chat. |
| `Send Telegram Reply` | `n8n-nodes-base.telegram` | Delivers agent text responses back to the user on Telegram | `CashLens Financial Assistant` | None | ## 2. Personal finance assistant<br>Answers Telegram questions such as "How much did I spend on Swiggy last month?" It reads the ledger and review queue (read-only), uses the calculator for exact totals, and remembers the last 10 messages per chat.<br><br>Replies come back as plain text in the same chat. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually recreate the entire workflow in an n8n instance:

1. **Create the Google Sheets Document:**
   - Create a new Google Sheet named `CashLens | Personal Transaction Ledger`.
   - Create two tabs named `Expenses` and `Review_Queue`.
   - Populate the `Expenses` tab header columns: `source_event_id`, `transaction_date`, `received_at`, `amount`, `currency`, `transaction_type`, `merchant`, `category`, `payment_method`, `account_last4`, `reference_id`, `source`, `sender`, `parse_status`, `cleaned_message`, `is_valid_transaction`, `missing_fields`, `transaction_date_source`, `sender_allowlisted`, `rejection_reason`, `needs_manual_review`.
   - Populate the `Review_Queue` tab header columns: `source_event_id`, `received_at`, `sender`, `amount`, `transaction_type`, `transaction_date`, `merchant`, `payment_method`, `reference_id`, `missing_fields`, `parse_status`, `review_status`.

2. **Set up Credentials:**
   - Configure a Google Sheets OAuth2 credential (`googleSheetsOAuth2Api`) and connect it.
   - Configure an HTTP Header Authentication credential (`httpHeaderAuth`) for webhook security.
   - Configure a Telegram API credential (`telegramApi`) obtained via BotFather.
   - Configure an Anthropic API credential (`anthropicApi`) for Claude.

3. **Build Block 1.1 (Record Bank Transactions):**
   - **Step 3.1:** Create a **Webhook** node named `Receive Bank SMS` (Path: `cashlens-sms`, Method: `POST`, Authentication: `Header Auth`). Connect HTTP Header credentials.
   - **Step 3.2:** Create a **Code** node named `Parse & Normalize Transaction`. Paste the transaction extraction JavaScript parsing logic processing `item.json.body ?? item.json`. Connect input from `Receive Bank SMS`.
   - **Step 3.3:** Create an **If** node named `Is Transaction Valid?`. Configure condition: `{{ $json.is_valid_transaction }}` equals `true`. Connect input from `Parse & Normalize Transaction`.
   - **Step 3.4:** Create a **Google Sheets** node named `Save to Transaction Ledger` (Operation: `appendOrUpdate`, Document ID: select your sheet, Sheet Name: `Expenses`, Matching Column: `source_event_id`). Connect input from the **True** branch of `Is Transaction Valid?`.
   - **Step 3.5:** Create a **Respond to Webhook** node named `Confirm Transaction Saved` (Response Code: 200, Body: JSON success confirmation). Connect input from `Save to Transaction Ledger`.
   - **Step 3.6:** Create an **If** node named `Needs Manual Review?`. Configure condition: `{{ $json.needs_manual_review }}` equals `NEEDS_REVIEW`. Connect input from the **False** branch of `Is Transaction Valid?`.
   - **Step 3.7:** Create a **Google Sheets** node named `Save to Review Queue` (Operation: `appendOrUpdate`, Sheet Name: `Review_Queue`, Matching Column: `source_event_id`). Connect input from the **True** branch of `Needs Manual Review?`.
   - **Step 3.8:** Create a **Respond to Webhook** node named `Confirm Review Queue` (Response Code: 200, Body: JSON review confirmation). Connect input from `Save to Review Queue`.
   - **Step 3.9:** Create a **Respond to Webhook** node named `Reject Invalid Transaction` (Response Code: 422, Body: JSON rejection confirmation). Connect input from the **False** branch of `Needs Manual Review?`.

4. **Build Block 1.2 (Personal Finance Assistant):**
   - **Step 4.1:** Create a **Telegram Trigger** node named `Telegram Message` (Updates: `message`). Connect Telegram credentials.
   - **Step 4.2:** Create an **AI Agent** node named `CashLens Financial Assistant` (Prompt Type: Define, Text: `={{ $json.message.text }}`, max iterations: 10, with system instructions enforcing INR currency, plain text, and role definitions). Connect input from `Telegram Message`.
   - **Step 4.3:** Create a **Claude Chat Model** node named `Claude Model` (Model: `claude-sonnet-5`). Connect as an AI language model to `CashLens Financial Assistant`.
   - **Step 4.4:** Create a **Window Buffer Memory** node named `Conversation Memory` (Session Key: `=cashlens-{{ $("Telegram Message").item.json.message.chat.id }}`, Context Window: 10). Connect as AI memory to `CashLens Financial Assistant`.
   - **Step 4.5:** Create two **Google Sheets Tool** nodes named `get_transactions` and `get_review_queue` targeting the `Expenses` and `Review_Queue` sheets respectively with descriptive tool descriptions. Connect both as AI tools to `CashLens Financial Assistant`.
   - **Step 4.6:** Create a **Calculator Tool** node named `calculator`. Connect as an AI tool to `CashLens Financial Assistant`.
   - **Step 4.7:** Create a **Telegram** node named `Send Telegram Reply` (Resource: `message`, Operation: `send`, Text: `={{ $json.output }}`, Chat ID: `={{ $("Telegram Message").item.json.message.chat.id }}`, Reply Message ID: `={{ $("Telegram Message").item.json.message.message_id }}`, Attributions disabled). Connect input from the output of `CashLens Financial Assistant`. Connect Telegram credentials.

5. **Activation:**
   - Save the workflow and toggle it to **Active**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| CashLens: Bank SMS expense tracker and Telegram finance assistant repository and template overview. | Template implementation designed for Indian banking SMS formats using n8n. |