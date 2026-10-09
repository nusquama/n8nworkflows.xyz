Answer finance questions from Xero via WhatsApp, Telegram, Slack, and OpenAI

https://n8nworkflows.xyz/workflows/answer-finance-questions-from-xero-via-whatsapp--telegram--slack--and-openai-20422


# Answer finance questions from Xero via WhatsApp, Telegram, Slack, and OpenAI

### 1. Workflow Overview

This workflow functions as an intelligent, multi-channel business finance assistant. It answers financial queries originating from WhatsApp (via WAHA) or Telegram, securely retrieves live reporting data from Xero, drafts natural-language summaries using OpenAI, validates financial figures to prevent hallucination, optionally posts updates to Slack, and maintains conversation context inside an n8n Data Table.

The logical execution follows ten sequential blocks:
- **1.1 Intake & Normalization:** Receives webhooks from WhatsApp (with HMAC validation) and Telegram, validates/computes security signatures, and normalizes payloads into a consistent format alongside configuration variables.
- **1.2 User Authorization:** Evaluates the incoming sender against an explicit whitelist of authorized users or chats.
- **1.3 Conversation State Management:** Loads prior dialogue state from an n8n Data Table and merges it with current message context.
- **1.4 Conversation Routing:** Inspects user text to determine whether the message is a Slack-sharing confirmation, a cancellation, or a new finance request.
- **1.5 Intent Classification & Slack Handlers:** Uses OpenAI (`gpt-4o-mini`) to categorize new requests or executes pre-configured branches for posting previous answers to Slack, handling denials, or returning help instructions.
- **1.6 Xero Context Preparation:** Connects to Xero, lists authorized organizations, selects the target tenant, and verifies environment readiness.
- **1.7 Financial Data Retrieval:** Queries specific Xero endpoints depending on the requested intent (Accounts Receivable, Bank Accounts & Cash Summary, or Payable Bills).
- **1.8 Response Generation & Validation:** Compiles retrieved datasets into a canonical text summary, prompts OpenAI to craft a conversational reply, and programmatically validates that no financial tokens or numbers were invented or omitted.
- **1.9 State Persistence:** Serializes updated conversation state and upserts records to the n8n Data Table.
- **1.10 Channel Response Delivery:** Dispatches the final formatted reply back to the originating transport layer (WhatsApp via WAHA or Telegram API).

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Intake & Normalization
- **Overview:** Receives raw webhook payloads from WhatsApp (WAHA) and Telegram, applies security checks (HMAC signature verification for WhatsApp), and transforms diverse message structures into a unified internal format.
- **Nodes Involved:** 
  - `Receive WhatsApp Message (WAHA Webhook)`
  - `Compute WAHA HMAC-SHA512`
  - `Is WAHA Signature Valid?`
  - `Receive Telegram Message`
  - `Normalize Channel Message`
  - `Set Workflow Config (Users, Slack, Xero)`

- **Node Details:**
  - **`Receive WhatsApp Message (WAHA Webhook)`**
    - *Type & Role:* `n8n-nodes-base.webhook` (Trigger) — Captures HTTP POST requests from the WAHA gateway.
    - *Configuration:* Path configured to `finance-bot-waha`, capturing raw request bodies for signature verification.
    - *Connections:* Input: None (Trigger); Output: `Compute WAHA HMAC-SHA512`.
    - *Edge Cases:* Missing request headers or malformed JSON payloads.

  - **`Compute WAHA HMAC-SHA512`**
    - *Type & Role:* `n8n-nodes-base.crypto` (Utility) — Generates an HMAC SHA-512 cryptographic hash of the incoming webhook body using binary data handling.
    - *Configuration:* Action set to `hmac`, algorithm set to `SHA512`.
    - *Connections:* Input: `Receive WhatsApp Message (WAHA Webhook)`; Output: `Is WAHA Signature Valid?`.

  - **`Is WAHA Signature Valid?`**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) — Evaluates whether the computed HMAC matches the `x-webhook-hmac` header and ensures the algorithm header equals `sha512`.
    - *Connections:* Input: `Compute WAHA HMAC-SHA512`; Output: `Normalize Channel Message`.

  - **`Receive Telegram Message`**
    - *Type & Role:* `n8n-nodes-base.telegramTrigger` (Trigger) — Listens for incoming updates from a configured Telegram Bot.
    - *Configuration:* Listens for `message` updates.
    - *Connections:* Input: None (Trigger); Output: `Normalize Channel Message`.

  - **`Normalize Channel Message`**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation) — JavaScript node that detects the source channel (Telegram vs. WhatsApp) and normalizes message objects into standard schema attributes (`channel`, `userId`, `chatId`, `messageId`, `messageText`, `timestamp`, `wahaSession`).
    - *Key Expressions:* Automatically filters out bot messages, empty strings, and self-sent WhatsApp messages (`p.fromMe === true`).
    - *Connections:* Input: `Is WAHA Signature Valid?`, `Receive Telegram Message`; Output: `Set Workflow Config (Users, Slack, Xero)`.

  - **`Set Workflow Config (Users, Slack, Xero)`**
    - *Type & Role:* `n8n-nodes-base.code` (Configuration) — Injects deployment-specific variables into the workflow execution context (WAHA base URL, Slack channel ID, allowed user whitelist, and optional Xero tenant ID).
    - *Connections:* Input: `Normalize Channel Message`; Output: `Build Authorization Result`.

---

#### Block 1.2: Authorize Finance User
- **Overview:** Validates whether the sender is authorized to execute financial queries by checking their user/chat identifier against an explicit whitelist.
- **Nodes Involved:**
  - `Build Authorization Result`
  - `Is User Authorized?`
  - `Prepare Unauthorized Reply`
  - `Prepare Unauthorized Reply (Alternate)` *(Referenced as `Prepare Unauthorized Reply`)*

- **Node Details:**
  - **`Build Authorization Result`**
    - *Type & Role:* `n8n-nodes-base.code` (Security Check) — Compares incoming `userId` and `chatId` against `config.allowedUsers`. Generates a composite `stateKey` and an `authorized` boolean flag.
    - *Connections:* Input: `Set Workflow Config (Users, Slack, Xero)`; Output: `Is User Authorized?`.

  - **`Is User Authorized?`**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) — Branches execution based on the `authorized` flag.
    - *Connections:* Input: `Build Authorization Result`; Output (True): `Load Conversation State`, Output (False): `Prepare Unauthorized Reply`.

  - **`Prepare Unauthorized Reply`**
    - *Type & Role:* `n8n-nodes-base.code` (Message Builder) — Assigns a standardized rejection message for unauthorized users.
    - *Connections:* Input: `Is User Authorized?` (False branch); Output: `Route: Reply Channel`.

---

#### Block 1.3: Load Conversation State
- **Overview:** Retrieves historical dialogue context and pending confirmation flags from an n8n Data Table based on the user's `stateKey`.
- **Nodes Involved:**
  - `Load Conversation State`
  - `Merge Message with Saved State`
  - `Prepare State Load Error Reply`

- **Node Details:**
  - **`Load Conversation State`**
    - *Type & Role:* `n8n-nodes-base.dataTable` (Database Operation) — Queries the n8n Data Table named `Finance Bot Conversation State` matching the current `stateKey`.
    - *Configuration:* Operation set to `get`, configured with error continuation (`continueErrorOutput`) to handle missing records gracefully.
    - *Connections:* Input: `Is User Authorized?` (True branch); Output: `Merge Message with Saved State` (success) or `Prepare State Load Error Reply` (failure).

  - **`Merge Message with Saved State`**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation) — Merges incoming message data with prior conversation metrics (`lastIntent`, `lastResponse`, `pendingSlackConfirmation`).
    - *Connections:* Input: `Load Conversation State`; Output: `Determine Conversation Route`.

  - **`Prepare State Load Error Reply`**
    - *Type & Role:* `n8n-nodes-base.code` (Error Handling) — Prepares a fallback notification message if data table retrieval fails.
    - *Connections:* Input: `Load Conversation State` (Error output); Output: `Route: Reply Channel`.

---

#### Block 1.4: Route Conversation
- **Overview:** Analyzes the incoming message text and active conversation flags to determine whether the user is answering a prompt to share a report to Slack, declining, or submitting a new finance question.
- **Nodes Involved:**
  - `Determine Conversation Route`
  - `Route: Slack Follow-up or New Request`

- **Node Details:**
  - **`Determine Conversation Route`**
    - *Type & Role:* `n8n-nodes-base.code` (Natural Language / Regex Parsing) — Normalizes input text and checks regex patterns for affirmative expressions (`yes`, `sure`), negative expressions (`no`, `cancel`), or explicit share commands (`post to slack`). Evaluates against `pendingSlackConfirmation` and `lastResponse`.
    - *Connections:* Input: `Merge Message with Saved State`; Output: `Route: Slack Follow-up or New Request`.

  - **`Route: Slack Follow-up or New Request`**
    - *Type & Role:* `n8n-nodes-base.switch` (Flow Control) — Directs workflow execution into four distinct pathways based on the evaluated `route` string (`slack_post`, `slack_no`, `no_previous`, or `classify`).
    - *Connections:* Input: `Determine Conversation Route`; Outputs:
      - Branch 0 (`slack_post`): `Prepare Slack Share Message`
      - Branch 1 (`slack_no`): `Prepare Slack Declined Reply`
      - Branch 2 (`no_previous`): `Prepare No Previous Answer Reply`
      - Branch 3 (`classify`): `OpenAI: Classify Finance Intent`

---

#### Block 1.5: Intent Classification & Slack Handlers
- **Overview:** Manages Slack-sharing confirmations, handles unrecognized intents, or invokes OpenAI to classify new user queries into specific financial domains.
- **Nodes Involved:**
  - `Prepare Slack Share Message`
  - `Post Answer to Slack`
  - `Prepare Slack Success Reply`
  - `Prepare Slack Failure Reply`
  - `Prepare Slack Declined Reply`
  - `Prepare No Previous Answer Reply`
  - `OpenAI: Classify Finance Intent`
  - `Parse Intent from AI Output`
  - `Prepare Intent Classification Error Reply`
  - `Route: By Finance Intent`
  - `Prepare Unknown Intent Help Reply`

- **Node Details:**
  - **`Prepare Slack Share Message`**
    - *Type & Role:* `n8n-nodes-base.code` — Prepares text payload for Slack transmission.
    - *Connections:* Input: `Route: Slack Follow-up or New Request`; Output: `Post Answer to Slack`.

  - **`Post Answer to Slack`**
    - *Type & Role:* `n8n-nodes-base.slack` (API Integration) — Posts message content to a designated Slack channel.
    - *Configuration:* Uses OAuth2/Bot credentials, targets channel ID from workflow config (`config.slackChannelId`), configured with retry logic (up to 3 tries) and `continueErrorOutput`.
    - *Connections:* Input: `Prepare Slack Share Message`; Output: `Prepare Slack Success Reply` or `Prepare Slack Failure Reply`.

  - **`Prepare Slack Success Reply`** / **`Prepare Slack Failure Reply`** / **`Prepare Slack Declined Reply`** / **`Prepare No Previous Answer Reply`**
    - *Type & Role:* `n8n-nodes-base.code` — Formulates appropriate user replies confirming success, failure, cancellation, or warning of missing prior answers.
    - *Connections:* Input: Slack action or routing branch; Output: `Build Conversation State Record`.

  - **`OpenAI: Classify Finance Intent`**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Model Node) — Utilizes `gpt-4o-mini` with strict JSON mode to classify queries into: `accounts_receivable`, `cash_position`, `bills_due_this_week`, `slack_confirmation`, or `unknown`.
    - *Configuration:* Temperature set to `0`, maximum 3 retries.
    - *Connections:* Input: `Route: Slack Follow-up or New Request`; Output: `Parse Intent from AI Output` (Success) or `Prepare Intent Classification Error Reply` (Error).

  - **`Parse Intent from AI Output`**
    - *Type & Role:* `n8n-nodes-base.code` — Sanitizes and parses LLM output JSON, verifying against permitted intent strings.
    - *Connections:* Input: `OpenAI: Classify Finance Intent`; Output: `Route: By Finance Intent`.

  - **`Route: By Finance Intent`**
    - *Type & Role:* `n8n-nodes-base.switch` (Flow Control) — Routes valid intents to Xero connection checks, or diverts unknown intents to help replies.
    - *Connections:* Input: `Parse Intent from AI Output`; Outputs:
      - Branch 0 (`accounts_receivable`): `Xero: Get Connected Organisations`
      - Branch 1 (`cash_position`): `Xero: Get Connected Organisations`
      - Branch 2 (`bills_due_this_week`): `Xero: Get Connected Organisations`
      - Branch 3 (`unknown`): `Prepare Unknown Intent Help Reply`

---

#### Block 1.6: Prepare Xero Context
- **Overview:** Connects to the Xero API to fetch authorized tenant organizations, selects the correct tenant ID based on configuration or auto-detection, and verifies environment readiness.
- **Nodes Involved:**
  - `Xero: Get Connected Organisations`
  - `Select Xero Tenant`
  - `Prepare Xero Connection Error Reply`
  - `Is Xero Tenant Ready?`
  - `Prepare Xero Tenant Error Reply`
  - `Route: Xero Report Type`

- **Node Details:**
  - **`Xero: Get Connected Organisations`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request) — Calls `https://api.xero.com/connections` using Xero OAuth2 credentials.
    - *Configuration:* Timeout 30s, retry on fail enabled.
    - *Connections:* Input: `Route: By Finance Intent`; Output: `Select Xero Tenant`.

  - **`Select Xero Tenant`**
    - *Type & Role:* `n8n-nodes-base.code` (Utility) — Matches configured `xeroTenantId` or auto-selects if a single organization is connected. Sets `xeroReady` status boolean.
    - *Connections:* Input: `Xero: Get Connected Organisations`; Output: `Is Xero Tenant Ready?`.

  - **`Is Xero Tenant Ready?`**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) — Evaluates `xeroReady`.
    - *Connections:* Input: `Select Xero Tenant`; Output (True): `Route: Xero Report Type`, Output (False): `Prepare Xero Tenant Error Reply`.

  - **`Route: Xero Report Type`**
    - *Type & Role:* `n8n-nodes-base.switch` (Flow Control) — Dispatches execution to the appropriate Xero data retrieval branch according to the user intent.
    - *Connections:* Input: `Is Xero Tenant Ready?`; Outputs:
      - Branch 0: `Xero: Get Receivable Invoices`
      - Branch 1: `Xero: Get Bank Accounts`
      - Branch 2: `Xero: Get Payable Bills`

---

#### Block 1.7: Financial Data Retrieval
- **Overview:** Queries Xero financial endpoints (Invoices, Bank Accounts, Bank Summary reports) with proper headers and pagination.
- **Nodes Involved:**
  - `Xero: Get Receivable Invoices`
  - `Xero: Get Bank Accounts`
  - `Xero: Get Bank Summary Report`
  - `Xero: Get Payable Bills`
  - `Summarize Receivables`
  - `Summarize Cash Position`
  - `Summarize Bills Due This Week`
  - `Section 7d. Xero API Errors` *(Handled by `Prepare Xero API Error Reply`)*

- **Node Details:**
  - **`Xero: Get Receivable Invoices`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request) — Queries Xero Invoices endpoint (`/api.xro/2.0/Invoices`) for unpaid accounts receivable (`Type=="ACCREC" && AmountDue>0`).
    - *Configuration:* Includes pagination logic (up to 100 items per page), sends `xero-tenant-id` header, uses OAuth2 authentication.
    - *Connections:* Input: `Route: Xero Report Type`; Output: `Summarize Receivables`.

  - **`Xero: Get Bank Accounts`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request) — Queries Xero Accounts (`/api.xro/2.0/Accounts`) where `Type=="BANK"` and `Status=="ACTIVE"`.
    - *Connections:* Input: `Route: Xero Report Type`; Output: `Xero: Get Bank Summary Report`.

  - **`Xero: Get Bank Summary Report`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request) — Fetches the Xero Bank Summary Report (`/api.xro/2.0/Reports/BankSummary`) for current dates.
    - *Connections:* Input: `Xero: Get Bank Accounts`; Output: `Summarize Cash Position`.

  - **`Xero: Get Payable Bills`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request) — Queries Xero Invoices endpoint for unpaid accounts payable bills (`Type=="ACCPAY" && AmountDue>0`).
    - *Connections:* Input: `Route: Xero Report Type`; Output: `Summarize Bills Due This Week`.

  - **`Summarize Receivables`** / **`Summarize Cash Position`** / **`Summarize Bills Due This Week`**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation) — Parses raw Xero JSON payloads, computes totals by currency, formats monetary values, generates a canonical human-readable fallback string (`safeFallback`), and extracts required validation tokens.
    - *Connections:* Input: Corresponding Xero HTTP nodes; Output: `Compile Finance Report Payload`.

---

#### Block 1.8: Draft & Validate Response
- **Overview:** Formats canonical financial data into an LLM prompt, requests a natural-language rewrite from OpenAI, and validates the output against source numbers to guarantee accuracy.
- **Nodes Involved:**
  - `Compile Finance Report Payload`
  - `OpenAI: Write Finance Response`
  - `Validate AI Response`
  - `Use Safe Fallback Response`

- **Node Details:**
  - **`Compile Finance Report Payload`**
    - *Type & Role:* `n8n-nodes-base.code` — Prepares unified payload containing canonical text and structured data.
    - *Connections:* Input: Summarizer nodes; Output: `OpenAI: Write Finance Response`.

  - **`OpenAI: Write Finance Response`**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Model Node) — Calls `gpt-4o-mini` with strict system instructions to rewrite financial summaries into concise chat responses without altering or hallucinating figures.
    - *Connections:* Input: `Compile Finance Report Payload`; Output: `Validate AI Response`.

  - **`Validate AI Response`**
    - *Type & Role:* `n8n-nodes-base.code` (Security / Integrity Check) — Programmatically verifies that all required tokens and monetary numbers present in the AI response exist within the source canonical text (`safeFallback`). If discrepancies or hallucinations are detected, it overrides the response with the safe fallback text.
    - *Connections:* Input: `OpenAI: Write Finance Response`; Output: `Build Conversation State Record`.

---

#### Block 1.9: Persist Conversation State
- **Overview:** Converges all successful execution paths, serializes state records, and upserts them into the n8n Data Table.
- **Nodes Involved:**
  - `Build Conversation State Record`
  - `Save Conversation State`
  - `Restore Reply Payload`

- **Node Details:**
  - **`Build Conversation State Record`**
    - *Type & Role:* `n8n-nodes-base.code` — Constructs the final state persistence object including `stateKey`, `lastIntent`, `lastResponse`, `pendingSlackConfirmation`, and current timestamp.
    - *Connections:* Input: Validation or reply generation nodes; Output: `Save Conversation State`.

  - **`Save Conversation State`**
    - *Type & Role:* `n8n-nodes-base.dataTable` (Database Operation) — Upserts conversation state records into the `Finance Bot Conversation State` data table matching on `stateKey`.
    - *Connections:* Input: `Build Conversation State Record`; Output: `Restore Reply Payload`.

  - **`Restore Reply Payload`**
    - *Type & Role:* `n8n-nodes-base.code` — Restores the active reply payload for final channel transmission.
    - *Connections:* Input: `Save Conversation State`; Output: `Route: Reply Channel`.

---

#### Block 1.10: Send Channel Response
- **Overview:** Dispatches the final message back to the originating messaging platform (WhatsApp via WAHA HTTP API or Telegram Bot API).
- **Nodes Involved:**
  - `Route: Reply Channel`
  - `Send WhatsApp Reply (WAHA)`
  - `Send Telegram Reply`

- **Node Details:**
  - **`Route: Reply Channel`**
    - *Type & Role:* `n8n-nodes-base.switch` (Flow Control) — Branches based on `channel === 'whatsapp'`.
    - *Connections:* Input: `Restore Reply Payload`, error handlers, or unauthorized handlers; Outputs:
      - Branch 0 (`whatsapp`): `Send WhatsApp Reply (WAHA)`
      - Branch 1 (`telegram`): `Send Telegram Reply`

  - **`Send WhatsApp Reply (WAHA)`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request) — Sends a text message via WAHA endpoint (`/api/sendText`) using HTTP Header Auth.
    - *Connections:* Input: `Route: Reply Channel`; Output: None (Terminal node).

  - **`Send Telegram Reply`**
    - *Type & Role:* `n8n-nodes-base.telegram` (Messaging Integration) — Sends a message using the Telegram Bot API (`sendMessage`).
    - *Connections:* Input: `Route: Reply Channel`; Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Overview` | `n8n-nodes-base.stickyNote` | Workflow documentation and setup instructions | None | None | ## Finance Bot - WhatsApp & Telegram, powered by Xero + OpenAI... |
| `Section 1. Intake & Normalization` | `n8n-nodes-base.stickyNote` | Section documentation for ingestion and normalization | None | None | ## 1. Intake & Normalization<br><br>WhatsApp (WAHA) requests are HMAC-verified... |
| `Section 2. Authorize Finance User` | `n8n-nodes-base.stickyNote` | Section documentation for user authorization | None | None | ## 2. Authorize Finance User<br><br>Checks the sender against the allowed-user list... |
| `Section 3. Load Conversation State` | `n8n-nodes-base.stickyNote` | Section documentation for state retrieval | None | None | ## 3. Load Conversation State<br><br>Loads the saved state for this chat... |
| `Section 4. Route Conversation` | `n8n-nodes-base.stickyNote` | Section documentation for conversation routing | None | None | ## 4. Route Conversation<br><br>Decides whether the message is a Slack-share follow-up... |
| `Section 5a. Slack Share Follow-ups` | `n8n-nodes-base.stickyNote` | Section documentation for Slack sharing logic | None | None | ## 5a. Slack Share Follow-ups<br><br>Posts the previous answer to Slack on 'Yes'... |
| `Section 5b. Classify Finance Intent` | `n8n-nodes-base.stickyNote` | Section documentation for intent classification | None | None | ## 5b. Classify Finance Intent<br><br>OpenAI classifies the request... |
| `Section 6. Prepare Xero Context` | `n8n-nodes-base.stickyNote` | Section documentation for Xero setup and tenant selection | None | None | ## 6. Prepare Xero Context<br><br>Fetches connected Xero organisations... |
| `Section 7a. Xero Receivables` | `n8n-nodes-base.stickyNote` | Section documentation for receivables report | None | None | ## 7a. Xero Receivables<br><br>Retrieves unpaid customer invoices... |
| `Section 7b. Xero Cash Position` | `n8n-nodes-base.stickyNote` | Section documentation for cash position report | None | None | ## 7b. Xero Cash Position<br><br>Retrieves active bank accounts and the bank summary... |
| `Section 7c. Xero Payable Bills` | `n8n-nodes-base.stickyNote` | Section documentation for payable bills report | None | None | ## 7c. Xero Payable Bills<br><br>Retrieves unpaid supplier bills... |
| `Section 7d. Xero API Errors` | `n8n-nodes-base.stickyNote` | Section documentation for Xero error handling | None | None | ## 7d. Xero API Errors<br><br>Any failed Xero request ends here with a standard message... |
| `Section 8. Draft & Validate Response` | `n8n-nodes-base.stickyNote` | Section documentation for OpenAI response drafting and validation | None | None | ## 8. Draft & Validate Response<br><br>Compiles the report, asks OpenAI to word it... |
| `Section 9. Persist Conversation State` | `n8n-nodes-base.stickyNote` | Section documentation for state persistence | None | None | ## 9. Persist Conversation State<br><br>Every path converges here... |
| `Section 10. Send Channel Response` | `n8n-nodes-base.stickyNote` | Section documentation for response dispatching | None | None | ## 10. Send Channel Response<br><br>Routes the reply to the originating channel... |
| `Receive WhatsApp Message (WAHA Webhook)` | `n8n-nodes-base.webhook` | Receives incoming WhatsApp messages via WAHA webhook | None | `Compute WAHA HMAC-SHA512` | ## 1. Intake & Normalization<br><br>WhatsApp (WAHA) requests are HMAC-verified... |
| `Compute WAHA HMAC-SHA512` | `n8n-nodes-base.crypto` | Computes HMAC SHA-512 signature for webhook validation | `Receive WhatsApp Message (WAHA Webhook)` | `Is WAHA Signature Valid?` | ## 1. Intake & Normalization<br><br>WhatsApp (WAHA) requests are HMAC-verified... |
| `Is WAHA Signature Valid?` | `n8n-nodes-base.if` | Validates WAHA HMAC signature headers | `Compute WAHA HMAC-SHA512` | `Normalize Channel Message` | ## 1. Intake & Normalization<br><br>WhatsApp (WAHA) requests are HMAC-verified... |
| `Receive Telegram Message` | `n8n-nodes-base.telegramTrigger` | Receives incoming messages from Telegram Bot | None | `Normalize Channel Message` | ## 1. Intake & Normalization<br><br>WhatsApp (WAHA) requests are HMAC-verified... |
| `Normalize Channel Message` | `n8n-nodes-base.code` | Normalizes channel-specific payloads into standard schema | `Is WAHA Signature Valid?`, `Receive Telegram Message` | `Set Workflow Config (Users, Slack, Xero)` | ## 1. Intake & Normalization<br><br>WhatsApp (WAHA) requests are HMAC-verified... |
| `Set Workflow Config (Users, Slack, Xero)` | `n8n-nodes-base.code` | Injects configuration settings (allowed users, Slack ID, Xero tenant) | `Normalize Channel Message` | `Build Authorization Result` | ## 1. Intake & Normalization<br><br>WhatsApp (WAHA) requests are HMAC-verified... |
| `Build Authorization Result` | `n8n-nodes-base.code` | Evaluates sender authorization against whitelist | `Set Workflow Config (Users, Slack, Xero)` | `Is User Authorized?` | ## 2. Authorize Finance User<br><br>Checks the sender against the allowed-user list... |
| `Is User Authorized?` | `n8n-nodes-base.if` | Branches execution based on user authorization | `Build Authorization Result` | `Load Conversation State`, `Prepare Unauthorized Reply` | ## 2. Authorize Finance User<br><br>Checks the sender against the allowed-user list... |
| `Load Conversation State` | `n8n-nodes-base.dataTable` | Retrieves prior conversation state from n8n Data Table | `Is User Authorized?` | `Merge Message with Saved State`, `Prepare State Load Error Reply` | ## 3. Load Conversation State<br><br>Loads the saved state for this chat... |
| `Merge Message with Saved State` | `n8n-nodes-base.code` | Merges loaded conversation state with incoming message | `Load Conversation State` | `Determine Conversation Route` | ## 3. Load Conversation State<br><br>Loads the saved state for this chat... |
| `Prepare State Load Error Reply` | `n8n-nodes-base.code` | Prepares fallback message on state load failure | `Load Conversation State` | `Route: Reply Channel` | ## 3. Load Conversation State<br><br>Loads the saved state for this chat... |
| `Determine Conversation Route` | `n8n-nodes-base.code` | Evaluates message text and flags to determine workflow route | `Merge Message with Saved State` | `Route: Slack Follow-up or New Request` | ## 4. Route Conversation<br><br>Decides whether the message is a Slack-share follow-up... |
| `Route: Slack Follow-up or New Request` | `n8n-nodes-base.switch` | Switches workflow path based on determined conversation route | `Determine Conversation Route` | `Prepare Slack Share Message`, `Prepare Slack Declined Reply`, `Prepare No Previous Answer Reply`, `OpenAI: Classify Finance Intent` | ## 4. Route Conversation<br><br>Decides whether the message is a Slack-share follow-up... |
| `Prepare Slack Share Message` | `n8n-nodes-base.code` | Formats previous answer for Slack sharing | `Route: Slack Follow-up or New Request` | `Post Answer to Slack` | ## 5a. Slack Share Follow-ups<br><br>Posts the previous answer to Slack on 'Yes'... |
| `Post Answer to Slack` | `n8n-nodes-base.slack` | Posts previous answer to configured Slack channel | `Prepare Slack Share Message` | `Prepare Slack Success Reply`, `Prepare Slack Failure Reply` | ## 5a. Slack Share Follow-ups<br><br>Posts the previous answer to Slack on 'Yes'... |
| `Prepare Slack Success Reply` | `n8n-nodes-base.code` | Formats success message after posting to Slack | `Post Answer to Slack` | `Build Conversation State Record` | ## 5a. Slack Share Follow-ups<br><br>Posts the previous answer to Slack on 'Yes'... |
| `Prepare Slack Failure Reply` | `n8n-nodes-base.code` | Formats failure message if Slack post fails | `Post Answer to Slack` | `Build Conversation State Record` | ## 5a. Slack Share Follow-ups<br><br>Posts the previous answer to Slack on 'Yes'... |
| `Prepare Slack Declined Reply` | `n8n-nodes-base.code` | Formats message when user declines Slack posting | `Route: Slack Follow-up or New Request` | `Build Conversation State Record` | ## 5a. Slack Share Follow-ups<br><br>Posts the previous answer to Slack on 'Yes'... |
| `Prepare No Previous Answer Reply` | `n8n-nodes-base.code` | Warns user when attempting to share without previous answer | `Route: Slack Follow-up or New Request` | `Build Conversation State Record` | ## 5a. Slack Share Follow-ups<br><br>Posts the previous answer to Slack on 'Yes'... |
| `OpenAI: Classify Finance Intent` | `@n8n/n8n-nodes-langchain.openAi` | Classifies user message into financial intent using OpenAI | `Route: Slack Follow-up or New Request` | `Parse Intent from AI Output`, `Prepare Intent Classification Error Reply` | ## 5b. Classify Finance Intent<br><br>OpenAI classifies the request... |
| `Parse Intent from AI Output` | `n8n-nodes-base.code` | Parses and validates JSON intent from AI output | `OpenAI: Classify Finance Intent` | `Route: By Finance Intent` | ## 5b. Classify Finance Intent<br><br>OpenAI classifies the request... |
| `Prepare Intent Classification Error Reply` | `n8n-nodes-base.code` | Formats error message if intent classification fails | `OpenAI: Classify Finance Intent` | `Build Conversation State Record` | ## 5b. Classify Finance Intent<br><br>OpenAI classifies the request... |
| `Route: By Finance Intent` | `n8n-nodes-base.switch` | Routes execution based on classified financial intent | `Parse Intent from AI Output` | `Xero: Get Connected Organisations`, `Prepare Unknown Intent Help Reply` | ## 5b. Classify Finance Intent<br><br>OpenAI classifies the request... |
| `Prepare Unknown Intent Help Reply` | `n8n-nodes-base.code` | Generates help text for unknown intents | `Route: By Finance Intent` | `Build Conversation State Record` | ## 5b. Classify Finance Intent<br><br>OpenAI classifies the request... |
| `Xero: Get Connected Organisations` | `n8n-nodes-base.httpRequest` | Fetches connected Xero organizations via OAuth2 | `Route: By Finance Intent` | `Select Xero Tenant` | ## 6. Prepare Xero Context<br><br>Fetches connected Xero organisations... |
| `Select Xero Tenant` | `n8n-nodes-base.code` | Selects and validates target Xero tenant ID | `Xero: Get Connected Organisations` | `Is Xero Tenant Ready?` | ## 6. Prepare Xero Context<br><br>Fetches connected Xero organisations... |
| `Prepare Xero Connection Error Reply` | `n8n-nodes-base.code` | Formats error message upon Xero connection failure | `Xero: Get Connected Organisations` | `Build Conversation State Record` | ## 6. Prepare Xero Context<br><br>Fetches connected Xero organisations... |
| `Is Xero Tenant Ready?` | `n8n-nodes-base.if` | Evaluates whether Xero tenant is successfully configured | `Select Xero Tenant` | `Route: Xero Report Type`, `Prepare Xero Tenant Error Reply` | ## 6. Prepare Xero Context<br><br>Fetches connected Xero organisations... |
| `Prepare Xero Tenant Error Reply` | `n8n-nodes-base.code` | Formats error message if Xero tenant is not ready | `Is Xero Tenant Ready?` | `Build Conversation State Record` | ## 6. Prepare Xero Context<br><br>Fetches connected Xero organisations... |
| `Route: Xero Report Type` | `n8n-nodes-base.switch` | Dispatches execution to specific Xero report endpoints | `Is Xero Tenant Ready?` | `Xero: Get Receivable Invoices`, `Xero: Get Bank Accounts`, `Xero: Get Payable Bills` | ## 6. Prepare Xero Context<br><br>Fetches connected Xero organisations... |
| `Xero: Get Receivable Invoices` | `n8n-nodes-base.httpRequest` | Retrieves unpaid customer invoices from Xero | `Route: Xero Report Type` | `Summarize Receivables`, `Prepare Xero API Error Reply` | ## 7a. Xero Receivables<br><br>Retrieves unpaid customer invoices and builds a receivables summary. |
| `Xero: Get Bank Accounts` | `n8n-nodes-base.httpRequest` | Retrieves active bank accounts from Xero | `Route: Xero Report Type` | `Xero: Get Bank Summary Report`, `Prepare Xero API Error Reply` | ## 7b. Xero Cash Position<br><br>Retrieves active bank accounts and the bank summary report... |
| `Xero: Get Bank Summary Report` | `n8n-nodes-base.httpRequest` | Fetches bank summary report from Xero | `Xero: Get Bank Accounts` | `Summarize Cash Position`, `Prepare Xero API Error Reply` | ## 7b. Xero Cash Position<br><br>Retrieves active bank accounts and the bank summary report... |
| `Xero: Get Payable Bills` | `n8n-nodes-base.httpRequest` | Retrieves unpaid supplier bills from Xero | `Route: Xero Report Type` | `Summarize Bills Due This Week`, `Prepare Xero API Error Reply` | ## 7c. Xero Payable Bills<br><br>Retrieves unpaid supplier bills and keeps those due this week. |
| `Summarize Receivables` | `n8n-nodes-base.code` | Parses receivables data and generates canonical text | `Xero: Get Receivable Invoices` | `Compile Finance Report Payload` | ## 7a. Xero Receivables<br><br>Retrieves unpaid customer invoices and builds a receivables summary. |
| `Summarize Cash Position` | `n8n-nodes-base.code` | Parses bank balances and generates canonical text | `Xero: Get Bank Summary Report` | `Compile Finance Report Payload` | ## 7b. Xero Cash Position<br><br>Retrieves active bank accounts and the bank summary report... |
| `Summarize Bills Due This Week` | `n8n-nodes-base.code` | Filters and summarizes bills due within current week | `Xero: Get Payable Bills` | `Compile Finance Report Payload` | ## 7c. Xero Payable Bills<br><br>Retrieves unpaid supplier bills and keeps those due this week. |
| `Compile Finance Report Payload` | `n8n-nodes-base.code` | Prepares payload for OpenAI response generation | `Summarize Receivables`, `Summarize Cash Position`, `Summarize Bills Due This Week` | `OpenAI: Write Finance Response` | ## 8. Draft & Validate Response<br><br>Compiles the report, asks OpenAI to word it... |
| `OpenAI: Write Finance Response` | `@n8n/n8n-nodes-langchain.openAi` | Generates conversational summary of financial report | `Compile Finance Report Payload` | `Validate AI Response` | ## 8. Draft & Validate Response<br><br>Compiles the report, asks OpenAI to word it... |
| `Validate AI Response` | `n8n-nodes-base.code` | Validates AI response against source canonical numbers | `OpenAI: Write Finance Response` | `Build Conversation State Record` | ## 8. Draft & Validate Response<br><br>Compiles the report, asks OpenAI to word it... |
| `Use Safe Fallback Response` | `n8n-nodes-base.code` | Provides safe fallback response if AI validation fails | `OpenAI: Write Finance Response` (Error) | `Build Conversation State Record` | ## 8. Draft & Validate Response<br><br>Compiles the report, asks OpenAI to word it... |
| `Prepare Xero API Error Reply` | `n8n-nodes-base.code` | Formats error message upon Xero API failure | Xero HTTP request error outputs | `Build Conversation State Record` | ## 7d. Xero API Errors<br><br>Any failed Xero request ends here with a standard user-facing message. |
| `Build Conversation State Record` | `n8n-nodes-base.code` | Serializes conversation state record for database persistence | Multiple upstream builders and error handlers | `Save Conversation State` | ## 9. Persist Conversation State<br><br>Every path converges here... |
| `Save Conversation State` | `n8n-nodes-base.dataTable` | Upserts conversation state record into n8n Data Table | `Build Conversation State Record` | `Restore Reply Payload` | ## 9. Persist Conversation State<br><br>Every path converges here... |
| `Restore Reply Payload` | `n8n-nodes-base.code` | Restores reply payload for final channel transmission | `Save Conversation State` | `Route: Reply Channel` | ## 9. Persist Conversation State<br><br>Every path converges here... |
| `Route: Reply Channel` | `n8n-nodes-base.switch` | Routes reply dispatch based on destination channel | `Restore Reply Payload`, `Prepare Unauthorized Reply`, `Prepare State Load Error Reply` | `Send WhatsApp Reply (WAHA)`, `Send Telegram Reply` | ## 10. Send Channel Response<br><br>Routes the reply to the originating channel... |
| `Send WhatsApp Reply (WAHA)` | `n8n-nodes-base.httpRequest` | Sends text message via WAHA HTTP API | `Route: Reply Channel` | None (Terminal) | ## 10. Send Channel Response<br><br>Routes the reply to the originating channel... |
| `Send Telegram Reply` | `n8n-nodes-base.telegram` | Sends message via Telegram Bot API | `Route: Reply Channel` | None (Terminal) | ## 10. Send Channel Response<br><br>Routes the reply to the originating channel... |
| `Prepare Unauthorized Reply` | `n8n-nodes-base.code` | Prepares rejection message for unauthorized users | `Is User Authorized?` (False) | `Route: Reply Channel` | ## 2. Authorize Finance User<br><br>Checks the sender against the allowed-user list... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create the n8n Data Table
1. In your n8n instance, create a new Data Table named `Finance Bot Conversation State`.
2. Add the following columns:
   - `stateKey` (Type: `string`)
   - `userId` (Type: `string`)
   - `channel` (Type: `string`)
   - `chatId` (Type: `string`)
   - `lastIntent` (Type: `string`)
   - `lastResponse` (Type: `string`)
   - `pendingSlackConfirmation` (Type: `boolean`)

#### Step 2: Set Up Triggers and Ingestion
1. **Webhook Node:** Create a Webhook node named `Receive WhatsApp Message (WAHA Webhook)`. Set HTTP method to `POST`, path to `finance-bot-waha`, and enable raw body options (`rawBody: true`).
2. **Crypto Node:** Add a Crypto node named `Compute WAHA HMAC-SHA512` connected to the webhook. Set action to `hmac`, algorithm to `SHA512`, and enable binary data processing.
3. **If Node:** Add an If node named `Is WAHA Signature Valid?` to compare `{{ $json.wahaComputedHmac }}` against header `x-webhook-hmac` and verify algorithm header equals `sha512`.
4. **Telegram Trigger Node:** Create a Telegram Trigger node named `Receive Telegram Message` listening for `message` updates.
5. **Code Node (`Normalize Channel Message`):** Combine inputs from the WAHA validation branch and Telegram trigger. Implement normalization JavaScript to output unified schema attributes (`channel`, `userId`, `chatId`, `messageText`, `timestamp`, `wahaSession`).
6. **Code Node (`Set Workflow Config`):** Add a configuration code node defining `wahaBaseUrl`, `slackChannelId`, `allowedUsers` array, and `xeroTenantId`.

#### Step 3: Authorization & State Management
1. **Code Node (`Build Authorization Result`):** Check sender `channel:userId` or `channel:chatId` against `allowedUsers` whitelist. Set boolean `authorized` and generate `stateKey`.
2. **If Node (`Is User Authorized?`):** Branch execution based on `authorized`. On failure, link to an unauthorized reply builder.
3. **Data Table Node (`Load Conversation State`):** Perform a `get` operation on `Finance Bot Conversation State` matching `stateKey`. Enable error continuation (`continueErrorOutput`).
4. **Code Node (`Merge Message with Saved State`):** Merge loaded row data (`lastIntent`, `lastResponse`, `pendingSlackConfirmation`) into the execution context.

#### Step 4: Routing & AI Intent Classification
1. **Code Node (`Determine Conversation Route`):** Evaluate incoming message text against regex patterns for affirmations (`yes`), negatives (`no`), or Slack-sharing commands.
2. **Switch Node (`Route: Slack Follow-up or New Request`):** Route based on `route` (`slack_post`, `slack_no`, `no_previous`, `classify`).
3. **OpenAI Node (`OpenAI: Classify Finance Intent`):** Configure with model `gpt-4o-mini`, temperature `0`, system prompt enforcing JSON output (`{"intent":"..."}`), and allowed intents (`accounts_receivable`, `cash_position`, `bills_due_this_week`, `slack_confirmation`, `unknown`).
4. **Code Node (`Parse Intent from AI Output`):** Parse LLM JSON response and validate against permitted intent strings.
5. **Switch Node (`Route: By Finance Intent`):** Branch valid intents to Xero connection setup.

#### Step 5: Xero Integration & Reporting
1. **HTTP Request Node (`Xero: Get Connected Organisations`):** Call `https://api.xero.com/connections` using Generic OAuth2 credentials.
2. **Code Node (`Select Xero Tenant`):** Select target tenant ID based on configuration or auto-selection.
3. **If Node (`Is Xero Tenant Ready?`):** Validate `xeroReady` status.
4. **Switch Node (`Route: Xero Report Type`):** Dispatch to report generation branches.
5. **Xero HTTP Nodes:**
   - Receivables: Query `/api.xro/2.0/Invoices` where `Type=="ACCREC" && AmountDue>0` with pagination.
   - Cash Position: Query `/api.xro/2.0/Accounts` (`Type=="BANK" && Status=="ACTIVE"`) followed by `/api.xro/2.0/Reports/BankSummary`.
   - Payable Bills: Query `/api.xro/2.0/Invoices` where `Type=="ACCPAY" && AmountDue>0` with pagination.
6. **Code Nodes (`Summarize ...`):** Parse datasets, calculate currency totals, format monetary strings, generate `safeFallback` canonical text, and compile required validation tokens.

#### Step 6: Response Drafting, Validation, & Persistence
1. **OpenAI Node (`OpenAI: Write Finance Response`):** Use `gpt-4o-mini` with strict system instructions to rewrite canonical reports into concise chat replies without altering figures.
2. **Code Node (`Validate AI Response`):** Verify that all required monetary tokens exist in the AI output. If hallucinated numbers or missing tokens are detected, fall back to `safeFallback`.
3. **Code Node (`Build Conversation State Record`):** Prepare state serialization object.
4. **Data Table Node (`Save Conversation State`):** Upsert record into `Finance Bot Conversation State` matching `stateKey`.

#### Step 7: Channel Response Dispatch
1. **Switch Node (`Route: Reply Channel`):** Branch based on `channel === 'whatsapp'`.
2. **HTTP Request Node (`Send WhatsApp Reply (WAHA)`):** POST to `{{ $json.config.wahaBaseUrl }}/api/sendText` using HTTP Header Auth (`session`, `chatId`, `text`).
3. **Telegram Node (`Send Telegram Reply`):** Send message via Telegram API (`chatId`, `text`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official Website & Workflow Templates | [Intuz Website](https://www.intuz.com/n8n-workflow-automation-templates/) |
| Contact Support Email | getstarted@intuz.com |
| Company LinkedIn Profile | [Intuz LinkedIn](https://www.linkedin.com/company/intuz) |
| Partner Links & Getting Started | [Intuz Partner Links](https://n8n.partnerlinks.io/intuz) |
| Custom Workflow Automation Services | [Intuz Custom Workflows](https://www.intuz.com/get-started/) |