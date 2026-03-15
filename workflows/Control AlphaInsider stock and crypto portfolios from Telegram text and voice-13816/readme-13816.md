Control AlphaInsider stock and crypto portfolios from Telegram text and voice

https://n8nworkflows.xyz/workflows/control-alphainsider-stock-and-crypto-portfolios-from-telegram-text-and-voice-13816


# Control AlphaInsider stock and crypto portfolios from Telegram text and voice

# 1. Workflow Overview

This workflow creates a Telegram-driven trading assistant that listens to both direct messages and channel posts, accepts text and voice inputs, converts them into normalized text, retrieves current AlphaInsider portfolio positions, asks an OpenAI-powered agent to decide what the message means, and then executes one of three outcomes:

- place updated order allocations in AlphaInsider,
- publish a post to the AlphaInsider audience,
- or send an informational reply back to the Telegram user with no trading/post action.

It is designed for use cases such as:

- controlling a stock or crypto portfolio from Telegram,
- reacting to trading signals from monitored Telegram channels,
- converting voice notes into actionable instructions,
- answering position-related questions in direct chat,
- posting market commentary to an AlphaInsider audience.

## 1.1 Telegram Input Reception and Source Routing
The workflow starts from a Telegram trigger and determines whether the incoming update is a direct message or a channel post. Channel posts can be restricted to a specific Telegram channel ID.

## 1.2 Message Normalization and Voice Handling
Incoming Telegram payloads are normalized into a common message structure. Text messages continue directly; voice messages are downloaded and transcribed with OpenAI so both paths produce text input for downstream processing.

## 1.3 Global Execution Context
The workflow centralizes key runtime values such as session ID, input text, strategy ID, and optional security whitelist in one Set node, so all later nodes can reference the same consistent data.

## 1.4 AlphaInsider Portfolio Retrieval and Position Parsing
Before the AI agent decides what to do, the workflow fetches the current positions from AlphaInsider and converts them into simplified percentage allocations, including leverage-aware sizing.

## 1.5 AI Decision and Action Classification
A LangChain agent backed by OpenAI analyzes the message, current positions, optional whitelist, and memory context. It outputs a structured JSON decision with one of three actions: `trade`, `post`, or `none`.

## 1.6 AlphaInsider Execution
If the AI returns `trade`, the workflow formats allocations and submits a new AlphaInsider order allocation request. If it returns `post`, it creates an AlphaInsider audience post.

## 1.7 Telegram Reply Handling
If the original input came from a direct message, the bot replies in Telegram with the AI-generated confirmation, explanation, or answer. Channel posts do not receive a Telegram reply.

---

# 2. Block-by-Block Analysis

## 2.1 Telegram Input Reception and Source Routing

### Overview
This block receives Telegram updates and separates direct messages from channel posts. For channel posts, it can optionally enforce a specific channel ID before allowing the message to continue.

### Nodes Involved
- Telegram Channel Listener
- Message Check
- Channel Check

### Node Details

#### Telegram Channel Listener
- **Type and role:** `n8n-nodes-base.telegramTrigger`; entry-point trigger node for Telegram updates.
- **Configuration choices:** Configured to listen for both `message` and `channel_post` updates.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point; outputs to **Message Check**.
- **Version-specific requirements:** Type version `1.2`; requires valid Telegram bot credentials.
- **Edge cases / failures:**
  - Invalid Telegram credentials or revoked bot token.
  - Webhook registration issues if another consumer already owns the webhook.
  - Bot not present in target channel, or not granted admin status where channel listening is expected.
  - Telegram update payload shape may vary; downstream routing depends on `message` or `channel_post` existing.
- **Sub-workflow reference:** None.

#### Message Check
- **Type and role:** `n8n-nodes-base.switch`; routes Telegram updates by payload type.
- **Configuration choices:** Two named outputs:
  - `Direct Message` when `$json.message` exists
  - `Channel Post` when `$json.channel_post` exists
- **Key expressions or variables used:**
  - `={{ $json.message }}`
  - `={{ $json.channel_post }}`
- **Input and output connections:**
  - Input from **Telegram Channel Listener**
  - Direct Message output -> **Message Details**
  - Channel Post output -> **Channel Check**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If Telegram sends another update type not covered here, nothing proceeds.
  - Strict type validation means malformed payloads can prevent matching.
- **Sub-workflow reference:** None.

#### Channel Check
- **Type and role:** `n8n-nodes-base.if`; optionally filters channel posts by a specific channel ID.
- **Configuration choices:** Compares `channel_post.chat.id` to a hard-coded numeric channel ID placeholder `-1001234567890`.
- **Key expressions or variables used:**
  - `={{ $json.channel_post.chat.id }}`
- **Input and output connections:**
  - Input from **Message Check** channel-post branch
  - True output -> **Message Details**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - If the configured channel ID is not updated, real channel posts will be rejected.
  - If the bot is not admin in the channel, updates may never arrive.
  - If the user wants all channel posts, this node currently still enforces a specific ID unless edited.
- **Sub-workflow reference:** None.

---

## 2.2 Message Normalization and Voice Handling

### Overview
This block creates a unified message structure and detects whether the content is text or voice. Voice notes are downloaded from Telegram and transcribed before continuing.

### Nodes Involved
- Message Details
- Message Details Check
- Get Voice Message
- Transcribe Voice Message

### Node Details

#### Message Details
- **Type and role:** `n8n-nodes-base.set`; standardizes the incoming Telegram payload into a common structure.
- **Configuration choices:**
  - Creates `session_id` from either direct-message chat ID or channel-post chat ID.
  - Creates `message` object from either `$json.message` or `$json.channel_post`.
- **Key expressions or variables used:**
  - `={{ $json.message.chat.id }}{{ $json.channel_post.chat.id  }}`
  - `={{ $json.message || $json.channel_post }}`
- **Input and output connections:**
  - Input from **Message Check** direct-message branch or **Channel Check**
  - Output -> **Message Details Check**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - The `session_id` expression concatenates two values as text before storing as a number; if both are empty or formatting changes, coercion may behave unexpectedly.
  - If neither `message` nor `channel_post` exists, downstream nodes break.
- **Sub-workflow reference:** None.

#### Message Details Check
- **Type and role:** `n8n-nodes-base.switch`; detects whether the normalized message is text or voice.
- **Configuration choices:** Two outputs:
  - `Text Message` if `message.text` exists
  - `Voice Message` if `message.voice` exists
- **Key expressions or variables used:**
  - `={{ $json.message.text }}`
  - `={{ $json.message.voice }}`
- **Input and output connections:**
  - Input from **Message Details**
  - Text Message -> **Global Settings**
  - Voice Message -> **Get Voice Message**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Non-text/non-voice messages such as photos, documents, stickers, captions, etc. are ignored.
  - If a Telegram message contains neither field, no branch is triggered.
- **Sub-workflow reference:** None.

#### Get Voice Message
- **Type and role:** `n8n-nodes-base.telegram`; retrieves the binary voice file from Telegram using the file ID.
- **Configuration choices:**
  - Resource: `file`
  - File ID pulled from `message.voice.file_id`
- **Key expressions or variables used:**
  - `={{ $json.message.voice.file_id }}`
- **Input and output connections:**
  - Input from **Message Details Check** voice branch
  - Output -> **Transcribe Voice Message**
- **Version-specific requirements:** Type version `1.2`; uses Telegram API credentials.
- **Edge cases / failures:**
  - Missing or invalid `file_id`.
  - Telegram file retrieval errors.
  - File too large or inaccessible.
- **Sub-workflow reference:** None.

#### Transcribe Voice Message
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi`; sends the voice file to OpenAI transcription.
- **Configuration choices:**
  - Resource: `audio`
  - Operation: `transcribe`
- **Key expressions or variables used:** None directly in parameters.
- **Input and output connections:**
  - Input from **Get Voice Message**
  - Output -> **Global Settings**
- **Version-specific requirements:** Type version `2.1`; requires OpenAI API credentials.
- **Edge cases / failures:**
  - Unsupported audio encoding or malformed binary.
  - OpenAI auth, quota, timeout, or rate-limit errors.
  - Poor transcription quality for noisy or multilingual audio.
- **Sub-workflow reference:** None.

---

## 2.3 Global Execution Context

### Overview
This block assembles the normalized text input and global runtime settings used everywhere else. It is the single source of truth for session memory, strategy selection, and optional trade whitelist.

### Nodes Involved
- Global Settings

### Node Details

#### Global Settings
- **Type and role:** `n8n-nodes-base.set`; creates workflow-wide fields for downstream use.
- **Configuration choices:**
  - `session_id` comes from **Message Details**
  - `input` concatenates text message content and transcription text
  - `strategy_id` is currently blank and must be filled manually
  - `whitelist` defaults to an empty array
- **Key expressions or variables used:**
  - `={{ $('Message Details').item.json.session_id }}`
  - `={{ $json.message.text }}{{ $json.text }}`
- **Input and output connections:**
  - Input from **Message Details Check** text branch or **Transcribe Voice Message**
  - Output -> **Get Positions**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If `strategy_id` is left blank, AlphaInsider API calls will fail.
  - Concatenating `message.text` and transcription `text` can produce unexpected strings if both exist.
  - Empty input causes weak or invalid AI results.
  - Whitelist must be valid array syntax if changed manually.
- **Sub-workflow reference:** None.

---

## 2.4 AlphaInsider Portfolio Retrieval and Position Parsing

### Overview
This block retrieves the current AlphaInsider positions for the configured strategy and transforms them into simplified allocations the AI can reason about. It removes the cash placeholder and computes percentages relative to total strategy value.

### Nodes Involved
- Get Positions
- Parse Position Percents

### Node Details

#### Get Positions
- **Type and role:** `n8n-nodes-base.httpRequest`; calls the AlphaInsider positions endpoint.
- **Configuration choices:**
  - GET request to `https://alphainsider.com/api/getPositions`
  - Query parameter `strategy_id` from **Global Settings**
  - Auth via generic bearer token credential
- **Key expressions or variables used:**
  - `={{ $('Global Settings').item.json.strategy_id }}`
- **Input and output connections:**
  - Input from **Global Settings**
  - Output -> **Parse Position Percents**
- **Version-specific requirements:** Type version `4.3`; requires HTTP Bearer Auth credential.
- **Edge cases / failures:**
  - Missing/invalid bearer token.
  - Blank or invalid strategy ID.
  - API downtime or non-200 responses.
  - Unexpected response shape can break parsing.
- **Sub-workflow reference:** None.

#### Parse Position Percents
- **Type and role:** `n8n-nodes-base.code`; converts raw AlphaInsider positions into normalized position percentages.
- **Configuration choices / logic:**
  - Reads positions from `$input.first().json.response`
  - Computes total strategy value using:
    - `bid` price for long/positive amounts
    - `ask` price for short/negative amounts
  - Throws an error if total strategy value is `<= 0`
  - Excludes `USD:ALPHAINSIDER`
  - Returns each position as:
    - `stock_id`
    - `stock`
    - `exchange`
    - `action` as `long` or `short`
    - `percent` as absolute fraction of total strategy value
- **Key expressions or variables used:** Internal JavaScript only.
- **Input and output connections:**
  - Input from **Get Positions**
  - Output -> **Parse Stock Allocations**
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - If the API response is not an array, it returns empty positions.
  - Missing numeric fields like `amount`, `bid`, or `ask` can produce `NaN`.
  - Highly leveraged or unusual portfolio structures may produce percentages the AI must interpret carefully.
  - The thrown object-style error may surface differently depending on n8n execution mode.
- **Sub-workflow reference:** None.

---

## 2.5 AI Decision and Action Classification

### Overview
This is the core decision block. It sends the normalized user input, whitelist, and current positions to an AI agent that can use memory and lookup tools, then parses the result into strict structured JSON for deterministic branching.

### Nodes Involved
- Parse Stock Allocations
- OpenAI Model
- Simple Memory
- Search Stocks
- Calculator
- ParseOutput

### Node Details

#### Parse Stock Allocations
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LangChain agent that determines whether the message is a trade, a post, or no action.
- **Configuration choices:**
  - Prompt includes:
    - `Input`
    - `Whitelist`
    - `Positions`
  - Uses an extensive system message defining:
    - trade/post/none semantics
    - whitelisting behavior
    - stock vs crypto consistency rules
    - leverage constraints
    - off-topic handling
    - message formatting rules
    - ASCII-only post requirement
    - mandatory stock lookup for unfamiliar securities
  - Structured output enabled through output parser.
- **Key expressions or variables used:**
  - `{{ $('Global Settings').item.json.input }}`
  - `{{ $('Global Settings').item.json.whitelist }}`
  - `{{ JSON.stringify($json.positions) }}`
- **Input and output connections:**
  - Main input from **Parse Position Percents**
  - AI language model input from **OpenAI Model**
  - AI memory input from **Simple Memory**
  - AI tool inputs from **Search Stocks** and **Calculator**
  - AI output parser input from **ParseOutput**
  - Main output -> **Switch**
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain and parser/tool node versions.
- **Edge cases / failures:**
  - LLM may still misclassify ambiguous messages.
  - Tool calls may fail and degrade output quality.
  - If model output violates schema, parsing may fail.
  - Prompt is large; token usage and latency can be significant.
- **Sub-workflow reference:** None.

#### OpenAI Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; provides the chat model for the agent.
- **Configuration choices:**
  - Model: `gpt-5.4`
  - Built-in web search enabled with `medium` search context
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - AI language model output -> **Parse Stock Allocations**
- **Version-specific requirements:** Type version `1.3`; requires OpenAI API credential and model availability in the account.
- **Edge cases / failures:**
  - Model unavailability or account access mismatch.
  - Web search may add latency or nondeterminism.
  - Rate limiting and quota exhaustion.
- **Sub-workflow reference:** None.

#### Simple Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; stores short conversation history keyed by session.
- **Configuration choices:**
  - Custom session key from global `session_id`
  - Context window length: `10`
- **Key expressions or variables used:**
  - `={{ $('Global Settings').item.json.session_id }}`
- **Input and output connections:**
  - AI memory output -> **Parse Stock Allocations**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - Session collisions if session IDs are not unique enough.
  - Memory persistence behavior depends on execution environment and node implementation.
  - Old context may bias ambiguous decisions.
- **Sub-workflow reference:** None.

#### Search Stocks
- **Type and role:** `n8n-nodes-base.httpRequestTool`; AI tool for AlphaInsider security search.
- **Configuration choices:**
  - POST `https://alphainsider.com/api/searchStocks`
  - JSON body generated via `$fromAI(...)`
  - Tool description instructs the agent to use it for stock/crypto discovery
- **Key expressions or variables used:**
  - `$fromAI('JSON', ...)`
- **Input and output connections:**
  - AI tool output -> **Parse Stock Allocations**
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases / failures:**
  - No authentication configured in this node; if AlphaInsider search requires auth in the future, this will fail.
  - AI may provide malformed tool arguments.
  - Search results may be ambiguous and require careful model selection.
- **Sub-workflow reference:** None.

#### Calculator
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCalculator`; auxiliary AI math tool.
- **Configuration choices:** Default configuration.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - AI tool output -> **Parse Stock Allocations**
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Usually low-risk; errors would mainly occur from malformed tool invocation.
- **Sub-workflow reference:** None.

#### ParseOutput
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces a JSON schema on AI output.
- **Configuration choices:**
  - Manual schema requiring:
    - `action`
    - `allocations`
    - `message`
  - Optional `post` object with `description` and `url`
- **Key expressions or variables used:** Manual JSON schema only.
- **Input and output connections:**
  - AI output parser output -> **Parse Stock Allocations**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - If the model returns invalid JSON or schema-mismatched data, the agent can fail.
  - The schema allows flexible `action` values inside each allocation object, but business logic expects `long`, `short`, or `close`.
- **Sub-workflow reference:** None.

---

## 2.6 AlphaInsider Execution Routing and API Actions

### Overview
This block receives the AI’s structured decision and routes it into trade execution, post creation, or no-action handling. Trading allocations are reformatted before submission to AlphaInsider.

### Nodes Involved
- Switch
- Parse Orders Allocations
- Create Orders
- Create Post

### Node Details

#### Switch
- **Type and role:** `n8n-nodes-base.switch`; branches by AI output action.
- **Configuration choices:** Three named outputs:
  - `Trade Action` when `output.action = trade`
  - `Post Action` when `output.action = post`
  - `No Action` when `output.action = none`
- **Key expressions or variables used:**
  - `={{ $json.output.action }}`
- **Input and output connections:**
  - Input from **Parse Stock Allocations**
  - Trade -> **Parse Orders Allocations**
  - Post -> **Create Post**
  - None -> **If DM**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If AI output parsing fails upstream, this node never runs.
  - Unexpected action values result in no branch.
- **Sub-workflow reference:** None.

#### Parse Orders Allocations
- **Type and role:** `n8n-nodes-base.code`; normalizes allocation percentages before AlphaInsider order submission.
- **Configuration choices / logic:**
  - Reads `output.allocations`
  - Converts `percent` to a fixed 4-decimal string
  - Returns `{ allocations }`
- **Key expressions or variables used:** Internal JavaScript only.
- **Input and output connections:**
  - Input from **Switch** trade branch
  - Output -> **Create Orders**
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - If `output.allocations` is missing or not an array, code will error.
  - `toFixed(4)` returns strings, not numbers; this may or may not match AlphaInsider API expectations.
- **Sub-workflow reference:** None.

#### Create Orders
- **Type and role:** `n8n-nodes-base.httpRequest`; submits trade allocations to AlphaInsider.
- **Configuration choices:**
  - POST `https://alphainsider.com/api/newOrderAllocations`
  - JSON body:
    - `strategy_id`
    - `allocations`
  - Bearer auth via generic credential
- **Key expressions or variables used:**
  - `$('Global Settings').item.json.strategy_id`
  - `$json.allocations`
- **Input and output connections:**
  - Input from **Parse Orders Allocations**
  - Output -> **If DM**
- **Version-specific requirements:** Type version `4.3`; requires AlphaInsider bearer credential.
- **Edge cases / failures:**
  - Invalid strategy ID or auth token.
  - API validation error if allocation format is rejected.
  - Network timeout or server-side error.
- **Sub-workflow reference:** None.

#### Create Post
- **Type and role:** `n8n-nodes-base.httpRequest`; posts audience content to AlphaInsider.
- **Configuration choices:**
  - POST `https://alphainsider.com/api/newPost`
  - Body parameters:
    - `description`
    - `url`
    - `strategy_id`
  - Bearer auth via generic credential
- **Key expressions or variables used:**
  - `={{ $json.output.post.description }}`
  - `={{ $json.output.post.url }}`
  - `={{ $('Global Settings').item.json.strategy_id }}`
- **Input and output connections:**
  - Input from **Switch** post branch
  - Output -> **If DM**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Missing `post.description`.
  - Optional `url` may be empty or invalid.
  - API may reject non-ASCII content if the AI fails to comply.
  - Invalid auth or strategy ID.
- **Sub-workflow reference:** None.

---

## 2.7 Telegram Reply Handling

### Overview
This final block decides whether the bot should reply on Telegram. Only direct messages receive a response; channel posts are processed silently.

### Nodes Involved
- If DM
- User Reply

### Node Details

#### If DM
- **Type and role:** `n8n-nodes-base.if`; checks whether the original Telegram update was a direct message.
- **Configuration choices:** Verifies that `Telegram Channel Listener.message.chat.id` exists.
- **Key expressions or variables used:**
  - `={{ $('Telegram Channel Listener').item.json.message.chat.id }}`
- **Input and output connections:**
  - Inputs from:
    - **Switch** no-action branch
    - **Create Orders**
    - **Create Post**
  - True output -> **User Reply**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - For channel posts, the field does not exist, so no reply is sent.
  - If the original direct message payload is missing unexpectedly, reply is skipped.
- **Sub-workflow reference:** None.

#### User Reply
- **Type and role:** `n8n-nodes-base.telegram`; sends a Telegram message back to the originating DM user.
- **Configuration choices:**
  - Text from AI output message
  - Chat ID from original Telegram trigger message
- **Key expressions or variables used:**
  - `={{ $('Parse Stock Allocations').item.json.output.message }}`
  - `={{ $('Telegram Channel Listener').item.json.message.chat.id }}`
- **Input and output connections:**
  - Input from **If DM**
  - No downstream output
- **Version-specific requirements:** Type version `1.2`; requires Telegram credentials.
- **Edge cases / failures:**
  - Telegram send errors if bot cannot message the user.
  - If AI output message is missing, the sent text may be empty or fail validation.
  - Chat ID lookup assumes DM path originated from `message`, not `channel_post`.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Channel Listener | n8n-nodes-base.telegramTrigger | Receives Telegram direct messages and channel posts |  | Message Check | ## Telegram Chat Bot<br>## How it works<br>This workflow monitors Telegram channels and direct messages for trading signals. When a message arrives, it routes through message type detection, transcribes voice messages if needed, and sends the content to AI analysis for intelligent action routing.<br>## Setup steps<br>- **(Required)** Create Telegram bot by messaging @BotFather and get your auth token<br>- **(Required)** Add the auth token to "Telegram Channel Listener" node credentials<br>- **(Optional)** To monitor a specific channel: set channel_id in "Channel Check" node and add bot as admin with NO permissions |
| Message Check | n8n-nodes-base.switch | Routes updates between DM and channel-post paths | Telegram Channel Listener | Message Details, Channel Check | ## Telegram Chat Bot<br>## How it works<br>This workflow monitors Telegram channels and direct messages for trading signals. When a message arrives, it routes through message type detection, transcribes voice messages if needed, and sends the content to AI analysis for intelligent action routing.<br>## Setup steps<br>- **(Required)** Create Telegram bot by messaging @BotFather and get your auth token<br>- **(Required)** Add the auth token to "Telegram Channel Listener" node credentials<br>- **(Optional)** To monitor a specific channel: set channel_id in "Channel Check" node and add bot as admin with NO permissions |
| Channel Check | n8n-nodes-base.if | Filters channel posts to a configured channel ID | Message Check | Message Details | ## Telegram Chat Bot<br>## How it works<br>This workflow monitors Telegram channels and direct messages for trading signals. When a message arrives, it routes through message type detection, transcribes voice messages if needed, and sends the content to AI analysis for intelligent action routing.<br>## Setup steps<br>- **(Required)** Create Telegram bot by messaging @BotFather and get your auth token<br>- **(Required)** Add the auth token to "Telegram Channel Listener" node credentials<br>- **(Optional)** To monitor a specific channel: set channel_id in "Channel Check" node and add bot as admin with NO permissions |
| Message Details | n8n-nodes-base.set | Normalizes Telegram payload into shared message object and session ID | Message Check, Channel Check | Message Details Check | ## Telegram Chat Bot<br>## How it works<br>This workflow monitors Telegram channels and direct messages for trading signals. When a message arrives, it routes through message type detection, transcribes voice messages if needed, and sends the content to AI analysis for intelligent action routing.<br>## Setup steps<br>- **(Required)** Create Telegram bot by messaging @BotFather and get your auth token<br>- **(Required)** Add the auth token to "Telegram Channel Listener" node credentials<br>- **(Optional)** To monitor a specific channel: set channel_id in "Channel Check" node and add bot as admin with NO permissions |
| Message Details Check | n8n-nodes-base.switch | Detects text versus voice messages | Message Details | Global Settings, Get Voice Message | ## Telegram Chat Bot<br>## How it works<br>This workflow monitors Telegram channels and direct messages for trading signals. When a message arrives, it routes through message type detection, transcribes voice messages if needed, and sends the content to AI analysis for intelligent action routing.<br>## Setup steps<br>- **(Required)** Create Telegram bot by messaging @BotFather and get your auth token<br>- **(Required)** Add the auth token to "Telegram Channel Listener" node credentials<br>- **(Optional)** To monitor a specific channel: set channel_id in "Channel Check" node and add bot as admin with NO permissions |
| Get Voice Message | n8n-nodes-base.telegram | Downloads Telegram voice file | Message Details Check | Transcribe Voice Message | ## Telegram Chat Bot<br>## How it works<br>This workflow monitors Telegram channels and direct messages for trading signals. When a message arrives, it routes through message type detection, transcribes voice messages if needed, and sends the content to AI analysis for intelligent action routing.<br>## Setup steps<br>- **(Required)** Create Telegram bot by messaging @BotFather and get your auth token<br>- **(Required)** Add the auth token to "Telegram Channel Listener" node credentials<br>- **(Optional)** To monitor a specific channel: set channel_id in "Channel Check" node and add bot as admin with NO permissions |
| Transcribe Voice Message | @n8n/n8n-nodes-langchain.openAi | Transcribes Telegram voice input into text | Get Voice Message | Global Settings | ## Telegram Chat Bot<br>## How it works<br>This workflow monitors Telegram channels and direct messages for trading signals. When a message arrives, it routes through message type detection, transcribes voice messages if needed, and sends the content to AI analysis for intelligent action routing.<br>## Setup steps<br>- **(Required)** Create Telegram bot by messaging @BotFather and get your auth token<br>- **(Required)** Add the auth token to "Telegram Channel Listener" node credentials<br>- **(Optional)** To monitor a specific channel: set channel_id in "Channel Check" node and add bot as admin with NO permissions |
| Global Settings | n8n-nodes-base.set | Stores session ID, normalized input text, strategy ID, and whitelist | Message Details Check, Transcribe Voice Message | Get Positions | ## Global Settings<br>## How it works<br>Centralizes workflow configuration including strategy ID, security whitelist, session tracking, and message input. All downstream nodes reference these global settings for consistent execution context.<br>## Setup steps<br>- **(Required)** Set strategy_id by copying it from your AlphaInsider strategy URL (e.g., "niAlE-cMI8TdsYQllZLmf")<br>- **(Optional)** Set whitelist array with "STOCK:EXCHANGE" format to restrict tradeable securities, or leave as [] for no restrictions |
| Get Positions | n8n-nodes-base.httpRequest | Retrieves current AlphaInsider positions | Global Settings | Parse Position Percents | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Parse Position Percents | n8n-nodes-base.code | Converts AlphaInsider positions into normalized percentages | Get Positions | Parse Stock Allocations | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| OpenAI Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the chat model for the AI agent |  | Parse Stock Allocations | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Supplies short conversation memory keyed by session |  | Parse Stock Allocations | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Search Stocks | n8n-nodes-base.httpRequestTool | AI tool for looking up securities in AlphaInsider |  | Parse Stock Allocations | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Calculator | @n8n/n8n-nodes-langchain.toolCalculator | AI math helper for allocation calculations |  | Parse Stock Allocations | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| ParseOutput | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output from the agent |  | Parse Stock Allocations | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Parse Stock Allocations | @n8n/n8n-nodes-langchain.agent | Interprets user intent and emits structured trade/post/none output | Parse Position Percents; OpenAI Model; Simple Memory; Search Stocks; Calculator; ParseOutput | Switch | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Switch | n8n-nodes-base.switch | Routes AI output to trade, post, or no-action path | Parse Stock Allocations | Parse Orders Allocations, Create Post, If DM | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Parse Orders Allocations | n8n-nodes-base.code | Formats AI allocation percentages before order submission | Switch | Create Orders | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Create Orders | n8n-nodes-base.httpRequest | Sends new portfolio allocations to AlphaInsider | Parse Orders Allocations | If DM | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| Create Post | n8n-nodes-base.httpRequest | Publishes an audience post to AlphaInsider | Switch | If DM | ## AlphaInsider Position Manager<br>## How it works<br>Fetches current portfolio positions from AlphaInsider, calculates allocation percentages with leverage, then uses OpenAI GPT-5.2-pro to analyze trading signals and route to three actions: execute trades, create audience posts, or provide direct Q&A responses.<br>## Setup steps<br>- **(Required)** Get AlphaInsider API token from developer settings (click n8n button). Add to "Get Positions", "Create Post", and "Create Orders" nodes<br>- **(Required)** Get OpenAI API key and add to "OpenAI Model" and "Transcribe Voice Message" nodes |
| If DM | n8n-nodes-base.if | Sends replies only for direct messages | Switch, Create Orders, Create Post | User Reply | ## Telegram Reply<br>## How it works<br>Detects if the incoming message was a direct message (DM) to the bot. If yes, sends the AI-generated response back to the user via Telegram with trade confirmations, post results, or answers to questions.<br>## Setup steps<br>- No configuration needed - automatic detection<br>- Replies only sent for direct messages, not channel posts<br>- Uses same Telegram credentials from "Telegram Channel Listener" node |
| User Reply | n8n-nodes-base.telegram | Sends Telegram reply back to the DM user | If DM |  | ## Telegram Reply<br>## How it works<br>Detects if the incoming message was a direct message (DM) to the bot. If yes, sends the AI-generated response back to the user via Telegram with trade confirmations, post results, or answers to questions.<br>## Setup steps<br>- No configuration needed - automatic detection<br>- Replies only sent for direct messages, not channel posts<br>- Uses same Telegram credentials from "Telegram Channel Listener" node |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for Telegram bot section |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for global settings section |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note for AlphaInsider/AI section |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Telegram reply section |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Telegram Chat Bot`.

2. **Add a Telegram Trigger node**
   - Node type: `Telegram Trigger`
   - Name: `Telegram Channel Listener`
   - Configure Telegram bot credentials using the token from `@BotFather`.
   - Set updates to:
     - `message`
     - `channel_post`

3. **Add a Switch node to separate DMs from channel posts**
   - Node type: `Switch`
   - Name: `Message Check`
   - Create output rule 1:
     - Output key: `Direct Message`
     - Condition: object exists
     - Left value: `{{ $json.message }}`
   - Create output rule 2:
     - Output key: `Channel Post`
     - Condition: object exists
     - Left value: `{{ $json.channel_post }}`
   - Connect `Telegram Channel Listener -> Message Check`.

4. **Add an If node for optional channel restriction**
   - Node type: `If`
   - Name: `Channel Check`
   - Add condition:
     - `{{ $json.channel_post.chat.id }}`
     - number equals
     - your Telegram channel ID, e.g. `-1001234567890`
   - Connect:
     - `Message Check (Channel Post) -> Channel Check`
   - If you want to monitor a specific channel, keep this check.
   - If you want all channel posts, modify or remove this filter.

5. **Add a Set node to normalize message structure**
   - Node type: `Set`
   - Name: `Message Details`
   - Add fields:
     - `session_id` as number:
       - `{{ $json.message.chat.id }}{{ $json.channel_post.chat.id }}`
     - `message` as object:
       - `{{ $json.message || $json.channel_post }}`
   - Connect:
     - `Message Check (Direct Message) -> Message Details`
     - `Channel Check (true) -> Message Details`

6. **Add a Switch node to detect text versus voice**
   - Node type: `Switch`
   - Name: `Message Details Check`
   - Output 1:
     - `Text Message`
     - condition: string exists
     - left value: `{{ $json.message.text }}`
   - Output 2:
     - `Voice Message`
     - condition: object exists
     - left value: `{{ $json.message.voice }}`
   - Connect `Message Details -> Message Details Check`.

7. **Add a Telegram node to download voice files**
   - Node type: `Telegram`
   - Name: `Get Voice Message`
   - Reuse the same Telegram credentials.
   - Resource: `File`
   - File ID:
     - `{{ $json.message.voice.file_id }}`
   - Connect:
     - `Message Details Check (Voice Message) -> Get Voice Message`

8. **Add an OpenAI transcription node**
   - Node type: `OpenAI` from the LangChain/OpenAI integration
   - Name: `Transcribe Voice Message`
   - Credential: OpenAI API key
   - Resource: `audio`
   - Operation: `transcribe`
   - Connect:
     - `Get Voice Message -> Transcribe Voice Message`

9. **Add a Set node for global runtime settings**
   - Node type: `Set`
   - Name: `Global Settings`
   - Add these assignments:
     - `session_id` as number:
       - `{{ $('Message Details').item.json.session_id }}`
     - `input` as string:
       - `{{ $json.message.text }}{{ $json.text }}`
     - `strategy_id` as string:
       - fill with your AlphaInsider strategy ID
     - `whitelist` as array:
       - `[]` by default
   - Examples for whitelist:
     - `["TSLA:NASDAQ","MSFT:NASDAQ"]`
     - or leave empty `[]`
   - Connect:
     - `Message Details Check (Text Message) -> Global Settings`
     - `Transcribe Voice Message -> Global Settings`

10. **Create AlphaInsider bearer credentials**
    - In n8n, create an `HTTP Bearer Auth` credential.
    - Paste your AlphaInsider API token.
    - This credential will be reused in:
      - `Get Positions`
      - `Create Orders`
      - `Create Post`

11. **Add an HTTP Request node to fetch positions**
    - Node type: `HTTP Request`
    - Name: `Get Positions`
    - Method: GET
    - URL: `https://alphainsider.com/api/getPositions`
    - Authentication: Generic Credential Type -> HTTP Bearer Auth
    - Query parameter:
      - `strategy_id = {{ $('Global Settings').item.json.strategy_id }}`
    - Connect `Global Settings -> Get Positions`.

12. **Add a Code node to compute position percentages**
    - Node type: `Code`
    - Name: `Parse Position Percents`
    - Paste the logic that:
      - reads positions from `json.response`
      - computes total strategy value
      - excludes `USD:ALPHAINSIDER`
      - maps positions into `{ stock_id, stock, exchange, action, percent }`
    - Connect `Get Positions -> Parse Position Percents`.

13. **Add the AI Agent node**
    - Node type: `AI Agent` / `LangChain Agent`
    - Name: `Parse Stock Allocations`
    - Set prompt type to define your own text.
    - In the main text input, pass:
      - Input text from `Global Settings`
      - Whitelist from `Global Settings`
      - Positions from `Parse Position Percents`
   - Use:
     - `Input: {{ $('Global Settings').item.json.input }}`
     - `Whitelist: {{ $('Global Settings').item.json.whitelist }}`
     - `Positions: {{ JSON.stringify($json.positions) }}`
   - Paste the large system instruction set defining:
     - action types (`trade`, `post`, `none`)
     - trading rules
     - whitelist behavior
     - search-tool requirement
     - direct Q&A behavior
     - ASCII-only post rules
     - message format requirements
   - Enable structured output parser.
   - Connect `Parse Position Percents -> Parse Stock Allocations`.

14. **Add the OpenAI chat model node**
    - Node type: `OpenAI Chat Model`
    - Name: `OpenAI Model`
    - Credential: OpenAI API key
    - Choose model `gpt-5.4`
    - Enable built-in web search if desired
      - Search context size: `medium`
    - Connect AI language model output to `Parse Stock Allocations`.

15. **Add conversation memory**
    - Node type: `Simple Memory` / `Memory Buffer Window`
    - Name: `Simple Memory`
    - Session ID type: custom key
    - Session key:
      - `{{ $('Global Settings').item.json.session_id }}`
    - Context window length: `10`
    - Connect AI memory output to `Parse Stock Allocations`.

16. **Add a stock-search tool node**
    - Node type: `HTTP Request Tool`
    - Name: `Search Stocks`
    - Method: POST
    - URL: `https://alphainsider.com/api/searchStocks`
    - Send body as JSON
    - Configure the JSON body to be AI-generated with parameters:
      - `search` required
      - `type` optional
      - `limit` optional
    - Add a clear tool description so the agent uses it for unknown symbols.
    - Connect AI tool output to `Parse Stock Allocations`.

17. **Add a calculator tool**
    - Node type: `Calculator Tool`
    - Name: `Calculator`
    - Default config is fine.
    - Connect AI tool output to `Parse Stock Allocations`.

18. **Add a structured output parser**
    - Node type: `Structured Output Parser`
    - Name: `ParseOutput`
    - Use manual schema.
    - Define schema with:
      - `action`: enum `trade`, `post`, `none`
      - `allocations`: array
      - `post`: optional object with `description`, optional `url`
      - `message`: string
    - Connect parser output to `Parse Stock Allocations`.

19. **Add a Switch node to branch on AI action**
    - Node type: `Switch`
    - Name: `Switch`
    - Add outputs:
      - `Trade Action` where `{{ $json.output.action }}` equals `trade`
      - `Post Action` where `{{ $json.output.action }}` equals `post`
      - `No Action` where `{{ $json.output.action }}` equals `none`
    - Connect `Parse Stock Allocations -> Switch`.

20. **Add a Code node to normalize order allocations**
    - Node type: `Code`
    - Name: `Parse Orders Allocations`
    - Logic:
      - read `output.allocations`
      - convert each `percent` to `parseFloat(...).toFixed(4)`
      - return `{ allocations }`
    - Connect:
      - `Switch (Trade Action) -> Parse Orders Allocations`

21. **Add an HTTP Request node to submit orders**
    - Node type: `HTTP Request`
    - Name: `Create Orders`
    - Method: POST
    - URL: `https://alphainsider.com/api/newOrderAllocations`
    - Authentication: HTTP Bearer Auth
    - Send JSON body:
      - `strategy_id: {{ $('Global Settings').item.json.strategy_id }}`
      - `allocations: {{ $json.allocations }}`
    - Connect:
      - `Parse Orders Allocations -> Create Orders`

22. **Add an HTTP Request node to submit posts**
    - Node type: `HTTP Request`
    - Name: `Create Post`
    - Method: POST
    - URL: `https://alphainsider.com/api/newPost`
    - Authentication: HTTP Bearer Auth
    - Send body parameters:
      - `description = {{ $json.output.post.description }}`
      - `url = {{ $json.output.post.url }}`
      - `strategy_id = {{ $('Global Settings').item.json.strategy_id }}`
    - Connect:
      - `Switch (Post Action) -> Create Post`

23. **Add an If node to reply only to direct messages**
    - Node type: `If`
    - Name: `If DM`
    - Condition:
      - `{{ $('Telegram Channel Listener').item.json.message.chat.id }}`
      - number exists
    - Connect all relevant inputs:
      - `Switch (No Action) -> If DM`
      - `Create Orders -> If DM`
      - `Create Post -> If DM`

24. **Add a Telegram send-message node**
    - Node type: `Telegram`
    - Name: `User Reply`
    - Reuse the same Telegram credentials.
    - Chat ID:
      - `{{ $('Telegram Channel Listener').item.json.message.chat.id }}`
    - Text:
      - `{{ $('Parse Stock Allocations').item.json.output.message }}`
    - Connect:
      - `If DM (true) -> User Reply`

25. **Optionally add documentation sticky notes**
    - Add sticky notes for:
      - Telegram input section
      - Global settings section
      - AlphaInsider/AI section
      - Telegram reply section

26. **Test direct-message text flow**
    - Send a text DM such as:
      - `What are my current positions?`
      - `Sell TSLA and buy MSFT`
    - Verify:
      - `Message Check` chooses DM
      - `Message Details Check` chooses text
      - AI output is parsed
      - Telegram reply is sent

27. **Test voice flow**
    - Send a voice note to the bot.
    - Verify:
      - `Get Voice Message` downloads the file
      - `Transcribe Voice Message` returns text
      - `Global Settings.input` contains the transcript

28. **Test channel-post flow**
    - Add the bot to the target channel as admin if required.
    - Post a message in the channel.
    - Verify:
      - `Message Check` chooses channel-post branch
      - `Channel Check` passes the correct channel ID
      - no Telegram reply is sent

29. **Validate required configuration before production**
    - Telegram bot token present
    - OpenAI API key present
    - AlphaInsider bearer token present
    - `strategy_id` filled in
    - optional `whitelist` formatted as an array of `STOCK:EXCHANGE`

30. **Recommended hardening changes**
    - Add error handling branches after:
      - OpenAI transcription
      - Get Positions
      - Create Orders
      - Create Post
    - Consider handling non-text Telegram content types.
    - Consider making channel ID optional instead of hard-coded.
    - Consider converting `session_id` explicitly to string to avoid numeric coercion issues.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Telegram bot by messaging `@BotFather` and use the returned token in Telegram nodes. | Telegram setup |
| To monitor a specific Telegram channel, add the bot as admin with no permissions and place the channel ID in `Channel Check`. | Telegram channel monitoring |
| `strategy_id` must be copied from the AlphaInsider strategy URL. | AlphaInsider configuration |
| `whitelist` should use `STOCK:EXCHANGE` format such as `TSLA:NASDAQ` or `BTC:COINBASE`. | Global settings |
| AlphaInsider API token is required for `Get Positions`, `Create Orders`, and `Create Post`. | AlphaInsider authentication |
| OpenAI API key is required for both the chat model and voice transcription. | OpenAI authentication |
| The workflow processes both stocks and cryptocurrencies, but the AI is instructed not to mix them in the same allocation output. | Business rule |
| The AI-generated `message` field is the single reply payload used for Telegram DM responses. | Reply behavior |
| Posts generated for AlphaInsider are required by the prompt to contain ASCII-only content. | Post content rule |