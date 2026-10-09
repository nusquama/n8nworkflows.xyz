Split and track WhatsApp restaurant bills with WAHA, Gemini and Sheets

https://n8nworkflows.xyz/workflows/split-and-track-whatsapp-restaurant-bills-with-waha--gemini-and-sheets-20449


# Split and track WhatsApp restaurant bills with WAHA, Gemini and Sheets

### 1. Workflow Overview

This workflow automates the management, tracking, and collection of shared restaurant bills over WhatsApp using WAHA (WhatsApp HTTP API), Google Gemini for receipt optical character recognition (OCR), OpenRouter (DeepSeek) for generative humour and payment reminders, and Google Sheets for relational data persistence. 

The application logic is partitioned into six functional blocks:
- **1.1 Input Reception & Whitelisting:** Captures incoming webhooks from WAHA, enforces security via whitelist validation, and standardizes payloads with localized currency parameters.
- **1.2 Receipt Intake & OCR:** Downloads receipt images, extracts structured line items, tax, and totals using Google Gemini, responds to the chat, and persists a temporary session state.
- **1.3 Bill Split Parsing & Generation:** Evaluates assignment formatting, queries session contexts, calculates proportional cost splits including taxes, and utilizes OpenRouter LLMs to compose humorous, itemized payment requests.
- **1.4 Dispatch & Logging:** Batches and sends individual bill messages to contacts via WhatsApp while concurrently recording financial ledgers inside Google Sheets.
- **1.5 Payment Confirmation Processing:** Parses payment alerts from inbound text, matches transaction senders against unpaid logs in Google Sheets, and updates statuses accordingly.
- **1.6 Automated Daily Reminders:** Executes a scheduled cron job to evaluate aging unpaid debts, applies anti-spam guards, escalates reminder severity dynamically, dispatches messages via OpenRouter, and updates tracking timestamps.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Whitelisting
- **Overview:** Listens for incoming WhatsApp messages via webhook, checks sender credentials against an explicit security whitelist, and sets global currency configurations for downstream accounting.
- **Nodes Involved:** 
  - `When Message Received (WAHA)`
  - `Whitelisted People`
  - `Config`

- **Node Details:**
  - **When Message Received (WAHA)**
    - *Type and Role:* `@devlikeapro/n8n-nodes-waha.wahaTrigger` (Trigger). Listens for incoming WhatsApp events.
    - *Configuration:* Uses default webhook event listeners configured via WAHA instance settings.
    - *Expressions:* None.
    - *Connections:* Input: None; Output: `Whitelisted People`.
    - *Failure Modes:* Webhook delivery dropouts, invalid WAHA session bindings, or network timeouts.
  - **Whitelisted People**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Filters incoming messages to authorized phone numbers/JIDs.
    - *Configuration:* Evaluates `{{ $json.payload._data.key.remoteJidAlt }}` against explicitly defined allowed user strings using OR combinators.
    - *Expressions:* `{{ $json.payload._data.key.remoteJidAlt }}`
    - *Connections:* Input: `When Message Received (WAHA)`; Output: `Config` (on true).
    - *Failure Modes:* Changes to WAHA internal payload structures breaking JSON path evaluation.
  - **Config**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Establishes default currency parameters.
    - *Configuration:* Sets explicit variables `currencyCode` to `IDR` and `currencySymbol` to `Rp`.
    - *Expressions:* None.
    - *Connections:* Input: `Whitelisted People`; Output: `Normalize WAHA Payload`.
    - *Failure Modes:* Missing downstream mapping if fields are renamed.

---

#### 2.2 Receipt Intake & OCR
- **Overview:** Normalizes WAHA message payloads, checks for media attachments, downloads receipt photos, extracts structured line items via Google Gemini, prompts chat participants for input, and saves the OCR data to a Google Sheets session tab.
- **Nodes Involved:**
  - `Normalize WAHA Payload`
  - `IF Has Photo (Receipt)?`
  - `Download Receipt Media (WAHA)`
  - `Analyze document`
  - `send message : chatting`
  - `Save Session (temp OCR)`

- **Node Details:**
  - **Normalize WAHA Payload**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation). Extracts and flattens message metadata.
    - *Configuration:* Custom JavaScript mapping inputs from nested WAHA webhook payloads into standard properties (`chatId`, `text`, `hasMedia`, `mediaUrl`, `mimetype`).
    - *Expressions:* `$json.body || $json`, `payload.media ? payload.media.url : null`
    - *Connections:* Input: `Config`; Output: `IF Has Photo (Receipt)?`.
    - *Failure Modes:* Unhandled null pointer exceptions if media objects are malformed.
  - **IF Has Photo (Receipt)?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Branches execution based on media presence.
    - *Configuration:* Evaluates `{{$json.hasMedia}}` equals `true`.
    - *Expressions:* `{{$json.hasMedia}}`
    - *Connections:* Input: `Normalize WAHA Payload`; Output: `Download Receipt Media (WAHA)` (true branch), `IF Assignment Format (contains ':')?` (false branch).
    - *Failure Modes:* Boolean type mismatches.
  - **Download Receipt Media (WAHA)**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (Integration). Fetches binary media files from WAHA servers.
    - *Configuration:* HTTP GET targeting `{{$json.mediaUrl}}`, returning data formatted as a file utilizing predefined `wahaApi` credentials.
    - *Expressions:* `{{$json.mediaUrl}}`
    - *Connections:* Input: `IF Has Photo (Receipt)?`; Output: `Analyze document`.
    - *Failure Modes:* HTTP 404/401 errors, expired media tokens, or large file size timeouts.
  - **Analyze document**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.googleGemini` (AI / Multimodal OCR). Parses images into structured financial JSON.
    - *Configuration:* Uses model `models/gemini-3.1-flash-lite`, reading binary property `data` with an explicit prompt enforcing JSON-only output schemas (`items`, `tax`, `total`).
    - *Expressions:* `=You read a photo of a shopping/food receipt... currencyCode: {{$('Config').item.json.currencyCode}}`
    - *Connections:* Input: `Download Receipt Media (WAHA)`; Output: `send message : chatting`.
    - *Failure Modes:* API rate limits, API key expiration, or Gemini returning non-JSON conversational prose breaking down-stream parsers.
  - **send message : chatting**
    - *Type and Role:* `@devlikeapro/n8n-nodes-waha.WAHA` (Messaging Integration). Sends instructions back to the WhatsApp group.
    - *Configuration:* Dispatches text via WAHA resource `Chatting`, operation `Send Text`.
    - *Expressions:* `chatId: {{$('When Message Received (WAHA)').item.json.payload.from}}`, `session: {{$('When Message Received (WAHA)').item.json.session}}`
    - *Connections:* Input: `Analyze document`; Output: `Save Session (temp OCR)`.
    - *Failure Modes:* WhatsApp session disconnection or invalid chat JIDs.
  - **Save Session (temp OCR)**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Database Operation). Persists temporary OCR states.
    - *Configuration:* Appends or updates rows inside the `Session` spreadsheet document using `chatId` as the matching column.
    - *Expressions:* `chatId: {{$('Normalize WAHA Payload').item.json.chatId}}`, `ocrData: {{$('Analyze document').item.json.content.parts[0].text}}`, `updatedAt: {{$now.toISO()}}`
    - *Connections:* Input: `send message : chatting`; Output: None (Terminal branch).
    - *Failure Modes:* Google Sheets API quota exhaustion or schema column mismatch.

---

#### 2.3 Bill Split Parsing & Generation
- **Overview:** Validates assignment text syntax, retrieves matching OCR session states, calculates individualized financial shares with proportional tax distribution, and routes data to an LLM chain to draft personalized humorous bill notices.
- **Nodes Involved:**
  - `IF Assignment Format (contains ':')?`
  - `Get Session (saved OCR)`
  - `Find Contact (waNumber)`
  - `Parse Assignment & Calculate Split`
  - `Basic LLM Chain`
  - `OpenRouter Chat Model`

- **Node Details:**
  - **IF Assignment Format (contains ':')?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Checks if incoming messages contain item assignment declarations.
    - *Configuration:* Regex evaluation matching `.*:.*` against `{{$json.text}}`.
    - *Expressions:* `{{$json.text}}`
    - *Connections:* Input: `IF Has Photo (Receipt)?` (false branch); Output: `Get Session (saved OCR)` & `Find Contact (waNumber)` (true branch), `IF Payment Confirmation?` (false branch).
    - *Failure Modes:* Regex string parsing edge cases.
  - **Get Session (saved OCR)**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Database Operation). Retrieves the active session context.
    - *Configuration:* Reads rows from the `Session` sheet within the configured document ID.
    - *Expressions:* None explicitly set in parameters (fetches all rows for downstream code evaluation).
    - *Connections:* Input: `IF Assignment Format (contains ':')?`; Output: `Parse Assignment & Calculate Split`.
    - *Failure Modes:* Missing rows or unpopulated session stores.
  - **Find Contact (waNumber)**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Database Operation). Retrieves contact definitions.
    - *Configuration:* Reads entries from the `Contacts` sheet to establish database cross-references.
    - *Expressions:* None.
    - *Connections:* Input: `IF Assignment Format (contains ':')?`; Output: `Merge`.
    - *Failure Modes:* Missing sheets or misaligned headers.
  - **Parse Assignment & Calculate Split**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation). Core mathematical and lexical splitting engine.
    - *Configuration:* Custom JavaScript filtering sessions by `chatId`, cleaning markdown JSON block wrappers from OCR strings, performing fuzzy matching on item keywords or numeric indices, calculating sub-totals, and applying proportional tax distribution.
    - *Expressions:* `const currencySymbol = $('Config').item.json.currencySymbol;`
    - *Connections:* Input: `Get Session (saved OCR)`; Output: `Basic LLM Chain`.
    - *Failure Modes:* Syntax errors in user reply strings resulting in fallback handling or zero-sum calculations.
  - **Basic LLM Chain**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Orchestration). Generates contextual humor for bills.
    - *Configuration:* Defines systemic chat instructions restricting responses to max 2 sentences, avoiding sensitive topics, and focusing exclusively on ordered food items and monetary figures.
    - *Expressions:* `Write a funny bill message for {{$json.name}} who ordered: {{$json.items.join(', ')}}. Total amount owed: {{ $('Config').item.json.currencySymbol }} {{$json.amount}}`
    - *Connections:* Input: `Parse Assignment & Calculate Split`; Output: `Compose Final Message`.
    - *Failure Modes:* LLM outages or timeouts via OpenRouter integrations.
  - **OpenRouter Chat Model**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (AI Model Provider). Provides the underlying language model infrastructure.
    - *Configuration:* Configured with model `deepseek/deepseek-v4-flash`.
    - *Expressions:* None.
    - *Connections:* Input: Linked to `Basic LLM Chain` via AI language model connection.
    - *Failure Modes:* API balance depletion or credential invalidation.

---

#### 2.4 Dispatch & Logging
- **Overview:** Combines generated roasting text with contact assignments, loops through items in batches, dispatches payment request messages to individual WhatsApp targets, and writes audit records to the Google Sheets Bills ledger.
- **Nodes Involved:**
  - `Compose Final Message`
  - `Merge`
  - `Loop Over Items`
  - `send message : chatting1`
  - `Log Bill to Sheet`

- **Node Details:**
  - **Compose Final Message**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation). Formats payload structures for delivery.
    - *Configuration:* Runs once for each item, assembling roast narratives, appending monetary calculations, and assigning default `unpaid` tracking statuses.
    - *Expressions:* `const data = $json; const roast = data.text || ...`
    - *Connections:* Input: `Basic LLM Chain`; Output: `Merge`.
    - *Failure Modes:* Missing input fields from preceding AI execution steps.
  - **Merge**
    - *Type and Role:* `n8n-nodes-base.merge` (Data Combination). Joins contact datasets with bill calculations.
    - *Configuration:* Combined mode, matching by fields `name` and `name`.
    - *Expressions:* None.
    - *Connections:* Input 1: `Find Contact (waNumber)`; Input 2: `Compose Final Message`; Output: `Loop Over Items`.
    - *Failure Modes:* Unmatched keys resulting in dropped items during join operations.
  - **Loop Over Items**
    - *Type and Role:* `n8n-nodes-base.splitInBatches` (Flow Control). Iterates through calculated ledger payloads sequentially.
    - *Configuration:* Standard batch iteration settings.
    - *Expressions:* None.
    - *Connections:* Input: `Merge` and `Log Bill to Sheet`; Output: `send message : chatting1`.
    - *Failure Modes:* Infinite iteration loops if batch pointers fail to advance.
  - **send message : chatting1**
    - *Type and Role:* `@devlikeapro/n8n-nodes-waha.WAHA` (Messaging Integration). Sends generated payment demands to debtors.
    - *Configuration:* Dispatches text via WAHA resource `Chatting`, operation `Send Text`.
    - *Expressions:* `text: {{$('Merge').item.json.text}}`, `chatId: {{$('Merge').item.json.waNumber}}`
    - *Connections:* Input: `Loop Over Items`; Output: `Log Bill to Sheet`.
    - *Failure Modes:* Invalid target WhatsApp phone numbers (`waNumber`).
  - **Log Bill to Sheet**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Database Operation). Appends financial audit rows.
    - *Configuration:* Appends row items to the `Bills` sheet mapping fields (`name`, `waNumber`, `items`, `amount`, `status`, `timestamp`, `lastReminderSent`).
    - *Expressions:* `name: {{$('Merge').item.json.name}}`, `items: {{$('Compose Final Message').item.json.items.join(', ')}}`, `amount: {{$('Compose Final Message').item.json.amount}}`
    - *Connections:* Input: `send message : chatting1`; Output: `Loop Over Items` (completes loop cycle).
    - *Failure Modes:* Google Sheets rate-limiting or schema violations.

---

#### 2.5 Payment Confirmation Processing
- **Overview:** Extracts payment confirmation intents from incoming text, queries unpaid bills in Google Sheets, matches entries via telephone numbers or name text fallbacks, and updates bill tracking records to a paid status.
- **Nodes Involved:**
  - `IF Payment Confirmation?`
  - `Extract Payment Identifiers`
  - `Get All Bills`
  - `Match Row`
  - `IF Row Found?`
  - `Update Status to Paid`

- **Node Details:**
  - **IF Payment Confirmation?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Identifies payment assertions.
    - *Configuration:* Regex evaluation matching `=(paid|done|Paid|Done)` against `{{$json.text}}`.
    - *Expressions:* `{{$json.text}}`
    - *Connections:* Input: `IF Assignment Format (contains ':')?` (false branch); Output: `Extract Payment Identifiers` (true branch).
    - *Failure Modes:* Non-matching natural language text confirming payment.
  - **Extract Payment Identifiers**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation). Parses metadata to identify payers.
    - *Configuration:* Custom JavaScript extracting phone digits from `remoteJidAlt` attributes and parsing textual name strings matching patterns like `Name: paid`.
    - *Expressions:* `const remoteJidAlt = $('When Message Received (WAHA)').item.json.payload._data.key.remoteJidAlt || ""`
    - *Connections:* Input: `IF Payment Confirmation?`; Output: `Get All Bills`.
    - *Failure Modes:* Absence of `remoteJidAlt` data within payload objects.
  - **Get All Bills**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Database Operation). Pulls active billing ledgers.
    - *Configuration:* Reads all records from the `Bills` spreadsheet.
    - *Expressions:* None.
    - *Connections:* Input: `Extract Payment Identifiers`; Output: `Match Row`.
    - *Failure Modes:* Large spreadsheet reading latency.
  - **Match Row**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation). Correlates payment claims with outstanding debts.
    - *Configuration:* Custom JavaScript filtering rows by `status === 'unpaid'`, matching by phone number or falling back to parsed name strings, and selecting the most recent entry based on timestamps.
    - *Expressions:* None.
    - *Connections:* Input: `Get All Bills`; Output: `IF Row Found?`.
    - *Failure Modes:* Ambiguous names causing false positive row matches.
  - **IF Row Found?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Verifies successful row matching.
    - *Configuration:* Evaluates `{{$json.found}}` equals `true`.
    - *Expressions:* `{{$json.found}}`
    - *Connections:* Input: `Match Row`; Output: `Update Status to Paid` (true branch).
    - *Failure Modes:* Boolean evaluation errors.
  - **Update Status to Paid**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Database Operation). Modifies financial ledger statuses.
    - *Configuration:* Updates spreadsheet rows targeting sheet `Bills`, matching by `row_number`, and setting `status` to `paid`.
    - *Expressions:* `row_number: {{$json.row_number}}`
    - *Connections:* Input: `IF Row Found?`; Output: None (Terminal step).
    - *Failure Modes:* Row index drift or API write authorization failure.

---

#### 2.6 Automated Daily Reminders
- **Overview:** Executes a daily cron schedule at 7 PM, evaluates outstanding unpaid bills, computes aging reminder levels with anti-spam safeguards, generates escalating collection text via OpenRouter LLMs, sends reminders over WhatsApp, and updates dispatch timestamps.
- **Nodes Involved:**
  - `Cron - Check Reminders (Daily 7PM)`
  - `Config1`
  - `Get Unpaid Bills`
  - `Calculate Reminder Level & Anti-Spam Guard`
  - `IF Reminder Needed?`
  - `Basic LLM Chain1`
  - `OpenRouter Chat Model1`
  - `send message : chatting2`
  - `Update lastReminderSent`

- **Node Details:**
  - **Cron - Check Reminders (Daily 7PM)**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Fires daily reminder cycles.
    - *Configuration:* Schedule rule configured to trigger at hour 19:00.
    - *Expressions:* None.
    - *Connections:* Input: None; Output: `Config1`.
    - *Failure Modes:* Server timezone misconfigurations causing unexpected trigger execution windows.
  - **Config1**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Sets reminder run contexts.
    - *Configuration:* Establishes variables `currencyCode` (`IDR`) and `currencySymbol` (`Rp`).
    - *Expressions:* None.
    - *Connections:* Input: `Cron - Check Reminders (Daily 7PM)`; Output: `Get Unpaid Bills`.
    - *Failure Modes:* None.
  - **Get Unpaid Bills**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Database Operation). Retrieves outstanding financial liabilities.
    - *Configuration:* Reads rows from sheet `Bills`.
    - *Expressions:* None.
    - *Connections:* Input: `Config1`; Output: `Calculate Reminder Level & Anti-Spam Guard`.
    - *Failure Modes:* Unreachable Google Sheets instances.
  - **Calculate Reminder Level & Anti-Spam Guard**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation). Computes aging brackets and enforces anti-spam rules.
    - *Configuration:* JavaScript calculation comparing `lastReminderSent` dates to current UTC days, calculating day differences (`daysDiff`), and establishing severity levels (`gentle` for >=1 day, `firm` for >=3 days, `savage` for >=7 days). Returns empty arrays if spam guards trigger.
    - *Expressions:* `const today = new Date().toISOString().slice(0, 10);`
    - *Connections:* Input: `Get Unpaid Bills`; Output: `IF Reminder Needed?`.
    - *Failure Modes:* ISO date conversion discrepancies across execution environments.
  - **IF Reminder Needed?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Filters out throttled debt records.
    - *Configuration:* Evaluates `{{$json.level}}` not equal to empty string.
    - *Expressions:* `{{$json.level}}`
    - *Connections:* Input: `Calculate Reminder Level & Anti-Spam Guard`; Output: `Basic LLM Chain1` (true branch).
    - *Failure Modes:* Schema mapping mismatches.
  - **Basic LLM Chain1**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Orchestration). Generates escalating collection scripts.
    - *Configuration:* Prompts LLM to produce max 2 sentences of comedic, escalating reminders based on severity levels.
    - *Expressions:* `Write a '{{$json.level}}' level reminder for {{$json.name}} who still owes {{ $('Config1').item.json.currencySymbol }} {{$json.amount}} for: {{$json.items}}.`
    - *Connections:* Input: `IF Reminder Needed?`; Output: `send message : chatting2`.
    - *Failure Modes:* Upstream OpenRouter service outages.
  - **OpenRouter Chat Model1**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (AI Model Provider). Provides reminder language model definitions.
    - *Configuration:* Model configuration set to `deepseek/deepseek-v4-flash`.
    - *Expressions:* None.
    - *Connections:* Input: Linked to `Basic LLM Chain1` via AI language model wire.
    - *Failure Modes:* Invalid API keys or quota limits.
  - **send message : chatting2**
    - *Type and Role:* `@devlikeapro/n8n-nodes-waha.WAHA` (Messaging Integration). Dispatches automated debt reminders.
    - *Configuration:* Sends text via WhatsApp chatting operation targeting sessions.
    - *Expressions:* `text: {{$json.text}}`, `chatId: {{$('IF Reminder Needed?').item.json.waNumber}}`
    - *Connections:* Input: `Basic LLM Chain1`; Output: `Update lastReminderSent`.
    - *Failure Modes:* Network interruptions or invalid destination chats.
  - **Update lastReminderSent**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Database Operation). Records reminder execution timestamps.
    - *Configuration:* Updates row entries inside the `Bills` sheet matching `row_number`, setting `lastReminderSent` to current message timestamps.
    - *Expressions:* `row_number: {{$('Get Unpaid Bills').item.json.row_number}}`, `lastReminderSent: {{$json.messageTimestamp}}`
    - *Connections:* Input: `send message : chatting2`; Output: None (Terminal node).
    - *Failure Modes:* Row locking conflicts or schema mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Bill Roaster - WhatsApp (WAHA)<br><br>### How it works<br><br>This workflow manages shared bills over WhatsApp using WAHA... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## WhatsApp intake gate<br><br>Receives incoming WAHA WhatsApp events, filters them to whitelisted people... |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Normalize and route message<br><br>Standardizes the WAHA payload into common fields... |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Receipt OCR session<br><br>Downloads receipt media, analyzes the document with Gemini... |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Text intent routing<br><br>Routes non-photo messages by checking for assignment syntax... |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Calculate bill split<br><br>Retrieves the saved OCR session, parses the assignment message... |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Match contact data<br><br>Looks up the payer contact by WhatsApp number... |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Send and log bills<br><br>Iterates over generated bill items, sends each WhatsApp payment request... |
| Sticky Note8 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Find payment record<br><br>Extracts identifiers from a payment confirmation message... |
| Sticky Note9 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Mark bill paid<br><br>Checks whether a matching row was found and updates that bill’s Google Sheets status to paid. |
| Sticky Note10 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Reminder schedule fetch<br><br>Runs the daily reminder trigger, applies the reminder configuration... |
| Sticky Note11 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Evaluate reminder need<br><br>Calculates reminder levels, applies anti-spam rules, and branches only bills that should receive a reminder. |
| Sticky Note12 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Send reminder update<br><br>Generates reminder copy with an LLM, sends the WhatsApp reminder, and records the latest reminder timestamp in Google Sheets. |
| When Message Received (WAHA) | `@devlikeapro/n8n-nodes-waha.wahaTrigger` | Trigger inbound messages | None | Whitelisted People | |
| Whitelisted People | `n8n-nodes-base.if` | Filter authorized users | When Message Received (WAHA) | Config | |
| Config | `n8n-nodes-base.set` | Establish default currency | Whitelisted People | Normalize WAHA Payload | |
| Normalize WAHA Payload | `n8n-nodes-base.code` | Standardize WAHA payload fields | Config | IF Has Photo (Receipt)? | ## Normalize and route message<br><br>Standardizes the WAHA payload into common fields and decides whether the incoming message includes a receipt photo or should be handled as text. |
| IF Has Photo (Receipt)? | `n8n-nodes-base.if` | Route by media attachment | Normalize WAHA Payload | Download Receipt Media (WAHA), IF Assignment Format (contains ':')? | ## Normalize and route message<br><br>Standardizes the WAHA payload into common fields and decides whether the incoming message includes a receipt photo or should be handled as text. |
| Download Receipt Media (WAHA) | `n8n-nodes-base.httpRequest` | Fetch image binary from WAHA | IF Has Photo (Receipt)? | Analyze document | ## Receipt OCR session<br><br>Downloads receipt media, analyzes the document with Gemini, replies to the chat, and saves the OCR result as temporary session context for a later assignment message. |
| Analyze document | `@n8n/n8n-nodes-langchain.googleGemini` | Extract OCR from image | Download Receipt Media (WAHA) | send message : chatting | ## Receipt OCR session<br><br>Downloads receipt media, analyzes the document with Gemini, replies to the chat, and saves the OCR result as temporary session context for a later assignment message. |
| send message : chatting | `@devlikeapro/n8n-nodes-waha.WAHA` | Prompt group for assignment format | Analyze document | Save Session (temp OCR) | ## Receipt OCR session<br><br>Downloads receipt media, analyzes the document with Gemini, replies to the chat, and saves the OCR result as temporary session context for a later assignment message. |
| Save Session (temp OCR) | `n8n-nodes-base.googleSheets` | Store session data to sheets | send message : chatting | None | ## Receipt OCR session<br><br>Downloads receipt media, analyzes the document with Gemini, replies to the chat, and saves the OCR result as temporary session context for a later assignment message. |
| IF Assignment Format (contains ':')? | `n8n-nodes-base.if` | Detect assignment message | IF Has Photo (Receipt)? | Get Session (saved OCR), Find Contact (waNumber), IF Payment Confirmation? | ## Text intent routing<br><br>Routes non-photo messages by checking for assignment syntax and, if not an assignment, checking whether the text is a payment confirmation. |
| Get Session (saved OCR) | `n8n-nodes-base.googleSheets` | Retrieve active OCR state | IF Assignment Format (contains ':')? | Parse Assignment & Calculate Split | ## Calculate bill split<br><br>Retrieves the saved OCR session, parses the assignment message, calculates shares, uses an LLM to refine the bill text, and composes the final message payload. |
| Find Contact (waNumber) | `n8n-nodes-base.googleSheets` | Look up contact info | IF Assignment Format (contains ':')? | Merge | ## Match contact data<br><br>Looks up the payer contact by WhatsApp number and merges contact details with the composed bill message before sending. |
| Parse Assignment & Calculate Split | `n8n-nodes-base.code` | Calculate individual shares & tax | Get Session (saved OCR) | Basic LLM Chain | ## Calculate bill split<br><br>Retrieves the saved OCR session, parses the assignment message, calculates shares, uses an LLM to refine the bill text, and composes the final message payload. |
| Basic LLM Chain | `@n8n/n8n-nodes-langchain.chainLlm` | Generate roasting bill text | Parse Assignment & Calculate Split | Compose Final Message | ## Calculate bill split<br><br>Retrieves the saved OCR session, parses the assignment message, calculates shares, uses an LLM to refine the bill text, and composes the final message payload. |
| OpenRouter Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | Language model provider | None (AI Link) | None (AI Link) | ## Calculate bill split<br><br>Retrieves the saved OCR session, parses the assignment message, calculates shares, uses an LLM to refine the bill text, and composes the final message payload. |
| Compose Final Message | `n8n-nodes-base.code` | Format message payloads | Basic LLM Chain | Merge | ## Calculate bill split<br><br>Retrieves the saved OCR session, parses the assignment message, calculates shares, uses an LLM to refine the bill text, and composes the final message payload. |
| Merge | `n8n-nodes-base.merge` | Combine contact and bill data | Find Contact (waNumber), Compose Final Message | Loop Over Items | ## Send and log bills<br><br>Iterates over generated bill items, sends each WhatsApp payment request, and logs the resulting bill records to Google Sheets. |
| Loop Over Items | `n8n-nodes-base.splitInBatches` | Iterate bill dispatches | Merge, Log Bill to Sheet | send message : chatting1 | ## Send and log bills<br><br>Iterates over generated bill items, sends each WhatsApp payment request, and logs the resulting bill records to Google Sheets. |
| send message : chatting1 | `@devlikeapro/n8n-nodes-waha.WAHA` | Send bill to debtor via WhatsApp | Loop Over Items | Log Bill to Sheet | ## Send and log bills<br><br>Iterates over generated bill items, sends each WhatsApp payment request, and logs the resulting bill records to Google Sheets. |
| Log Bill to Sheet | `n8n-nodes-base.googleSheets` | Log bill to Google Sheets | send message : chatting1 | Loop Over Items | ## Send and log bills<br><br>Iterates over generated bill items, sends each WhatsApp payment request, and logs the resulting bill records to Google Sheets. |
| IF Payment Confirmation? | `n8n-nodes-base.if` | Detect payment keywords | IF Assignment Format (contains ':')? | Extract Payment Identifiers | ## Text intent routing<br><br>Routes non-photo messages by checking for assignment syntax and, if not an assignment, checking whether the text is a payment confirmation. |
| Extract Payment Identifiers | `n8n-nodes-base.code` | Extract sender numbers/names | IF Payment Confirmation? | Get All Bills | ## Find payment record<br><br>Extracts identifiers from a payment confirmation message, loads bill records, and matches the sender/message to the relevant bill row. |
| Get All Bills | `n8n-nodes-base.googleSheets` | Load unpaid records | Extract Payment Identifiers | Match Row | ## Find payment record<br><br>Extracts identifiers from a payment confirmation message, loads bill records, and matches the sender/message to the relevant bill row. |
| Match Row | `n8n-nodes-base.code` | Match bill row to sender | Get All Bills | IF Row Found? | ## Find payment record<br><br>Extracts identifiers from a payment confirmation message, loads bill records, and matches the sender/message to the relevant bill row. |
| IF Row Found? | `n8n-nodes-base.if` | Validate row match | Match Row | Update Status to Paid | ## Mark bill paid<br><br>Checks whether a matching row was found and updates that bill’s Google Sheets status to paid. |
| Update Status to Paid | `n8n-nodes-base.googleSheets` | Update bill status to paid | IF Row Found? | None | ## Mark bill paid<br><br>Checks whether a matching row was found and updates that bill’s Google Sheets status to paid. |
| Cron - Check Reminders (Daily 7PM) | `n8n-nodes-base.scheduleTrigger` | Daily schedule trigger | None | Config1 | ## Reminder schedule fetch<br><br>Runs the daily reminder trigger, applies the reminder configuration, and retrieves currently unpaid bills from Google Sheets. |
| Config1 | `n8n-nodes-base.set` | Set reminder currency settings | Cron - Check Reminders (Daily 7PM) | Get Unpaid Bills | ## Reminder schedule fetch<br><br>Runs the daily reminder trigger, applies the reminder configuration, and retrieves currently unpaid bills from Google Sheets. |
| Get Unpaid Bills | `n8n-nodes-base.googleSheets` | Retrieve unpaid bill rows | Config1 | Calculate Reminder Level & Anti-Spam Guard | ## Reminder schedule fetch<br><br>Runs the daily reminder trigger, applies the reminder configuration, and retrieves currently unpaid bills from Google Sheets. |
| Calculate Reminder Level & Anti-Spam Guard | `n8n-nodes-base.code` | Compute reminder levels & anti-spam | Get Unpaid Bills | IF Reminder Needed? | ## Evaluate reminder need<br><br>Calculates reminder levels, applies anti-spam rules, and branches only bills that should receive a reminder. |
| IF Reminder Needed? | `n8n-nodes-base.if` | Evaluate reminder flags | Calculate Reminder Level & Anti-Spam Guard | Basic LLM Chain1 | ## Evaluate reminder need<br><br>Calculates reminder levels, applies anti-spam rules, and branches only bills that should receive a reminder. |
| Basic LLM Chain1 | `@n8n/n8n-nodes-langchain.chainLlm` | Generate reminder message | IF Reminder Needed? | send message : chatting2 | ## Send reminder update<br><br>Generates reminder copy with an LLM, sends the WhatsApp reminder, and records the latest reminder timestamp in Google Sheets. |
| OpenRouter Chat Model1 | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | Language model provider | None (AI Link) | None (AI Link) | ## Send reminder update<br><br>Generates reminder copy with an LLM, sends the WhatsApp reminder, and records the latest reminder timestamp in Google Sheets. |
| send message : chatting2 | `@devlikeapro/n8n-nodes-waha.WAHA` | Send reminder via WhatsApp | Basic LLM Chain1 | Update lastReminderSent | ## Send reminder update<br><br>Generates reminder copy with an LLM, sends the WhatsApp reminder, and records the latest reminder timestamp in Google Sheets. |
| Update lastReminderSent | `n8n-nodes-base.googleSheets` | Update reminder timestamp | send message : chatting2 | None | ## Send reminder update<br><br>Generates reminder copy with an LLM, sends the WhatsApp reminder, and records the latest reminder timestamp in Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Core Triggers and Input Gate
1. Create a new n8n workflow.
2. Add a **When Message Received (WAHA)** node (`@devlikeapro/n8n-nodes-waha.wahaTrigger`).
3. Add an **IF** node named `Whitelisted People` (`n8n-nodes-base.if`). Configure conditions to check `{{ $json.payload._data.key.remoteJidAlt }}` against your authorized user JIDs using an `or` combinator. Connect the trigger output to this node.
4. Add a **Set** node named `Config` (`n8n-nodes-base.set`). Set fields `currencyCode` (`IDR`) and `currencySymbol` (`Rp`). Connect `Whitelisted People` (true branch) to this node.

#### Step 2: Build the Receipt Intake and OCR Branch
1. Add a **Code** node named `Normalize WAHA Payload` (`n8n-nodes-base.code`). Add the payload sanitization script mapping webhook structures into `chatId`, `text`, `hasMedia`, and `mediaUrl`. Connect `Config` to it.
2. Add an **IF** node named `IF Has Photo (Receipt)?` (`n8n-nodes-base.if`) evaluating `{{$json.hasMedia}}` equals `true`. Connect `Normalize WAHA Payload` to it.
3. Add an **HTTP Request** node named `Download Receipt Media (WAHA)` (`n8n-nodes-base.httpRequest`). Set method to `GET`, URL to `{{$json.mediaUrl}}`, set response format to `File`, and configure credentials using a predefined `wahaApi` type. Connect the true branch of `IF Has Photo (Receipt)?` here.
4. Add a **Google Gemini Chat Model** node named `Analyze document` (`@n8n/n8n-nodes-langchain.googleGemini`). Set resource to `Document`, input type to `Binary`, binary property to `data`, model to `models/gemini-3.1-flash-lite`, and supply the currency-aware OCR prompt. Connect `Download Receipt Media (WAHA)` here.
5. Add a **WAHA** node named `send message : chatting` (`@devlikeapro/n8n-nodes-waha.WAHA`). Set resource to `Chatting`, operation to `Send Text`, chat ID to `{{$('When Message Received (WAHA)').item.json.payload.from}}`, and session references. Connect `Analyze document` here.
6. Add a **Google Sheets** node named `Save Session (temp OCR)` (`n8n-nodes-base.googleSheets`). Set operation to `Append or Update`, document ID to your target Google Sheet, sheet name to `Session`, matching column to `chatId`, and map fields (`chatId`, `state`, `ocrData`, `updatedAt`). Connect `send message : chatting` here.

#### Step 3: Build the Assignment Parsing and Calculation Branch
1. From the false branch of `IF Has Photo (Receipt)?`, connect to an **IF** node named `IF Assignment Format (contains ':')?` (`n8n-nodes-base.if`). Configure a regex condition matching `.*:.*` against `{{$json.text}}`.
2. From the true branch of `IF Assignment Format (contains ':')?`, connect to two parallel nodes:
   - **Get Session (saved OCR)** (`n8n-nodes-base.googleSheets`): Set operation to `Get Many` (or read) targeting the `Session` sheet.
   - **Find Contact (waNumber)** (`n8n-nodes-base.googleSheets`): Set operation to `Get Many` targeting the `Contacts` sheet.
3. Connect `Get Session (saved OCR)` to a **Code** node named `Parse Assignment & Calculate Split` (`n8n-nodes-base.code`). Insert the assignment parsing and proportional tax-splitting script.
4. Add a **Basic LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`) connected to `Parse Assignment & Calculate Split`. Define prompt instructions restricting responses to max 2 sentences of light-hearted humor focused only on food items.
5. Connect an **OpenRouter Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) to the AI model connector input of `Basic LLM Chain`, specifying model `deepseek/deepseek-v4-flash` with valid OpenRouter credentials.
6. Connect `Basic LLM Chain` to a **Code** node named `Compose Final Message` (`n8n-nodes-base.code`) running once per item to consolidate attributes.

#### Step 4: Build Dispatch and Ledger Logging
1. Add a **Merge** node (`n8n-nodes-base.merge`) combining outputs from `Find Contact (waNumber)` and `Compose Final Message` using combined matching on field `name`.
2. Add a **Split In Batches** node named `Loop Over Items` (`n8n-nodes-base.splitInBatches`). Connect the output of `Merge` here.
3. Add a **WAHA** node named `send message : chatting1` (`@devlikeapro/n8n-nodes-waha.WAHA`). Set resource to `Chatting`, operation to `Send Text`, chat ID to `{{$lar('Merge').item.json.waNumber}}`, and text to `{{$('Merge').item.json.text}}`. Connect the batch loop output here.
4. Add a **Google Sheets** node named `Log Bill to Sheet` (`n8n-nodes-base.googleSheets`). Set operation to `Append`, targeting the `Bills` sheet, mapping fields (`name`, `waNumber`, `items`, `amount`, `status`, `timestamp`, `lastReminderSent`). Connect `send message : chatting1` here, and loop its completion output back to `Loop Over Items` to close the batch cycle.

#### Step 5: Build Payment Confirmation Processing
1. From the false branch of `IF Assignment Format (contains ':')?`, connect to an **IF** node named `IF Payment Confirmation?` (`n8n-nodes-base.if`) checking if text matches regex `=(paid|done|Paid|Done)`.
2. Add a **Code** node named `Extract Payment Identifiers` (`n8n-nodes-base.code`). Connect the true branch of `IF Payment Confirmation?` here.
3. Add a **Google Sheets** node named `Get All Bills` (`n8n-nodes-base.googleSheets`) to read all rows from the `Bills` sheet. Connect `Extract Payment Identifiers` here.
4. Add a **Code** node named `Match Row` (`n8n-nodes-base.code`) to correlate unpaid statuses with matching phone identifiers or name fallbacks. Connect `Get All Bills` here.
5. Add an **IF** node named `IF Row Found?` (`n8n-nodes-base.if`) evaluating `{{$json.found}}` equals `true`. Connect `Match Row` here.
6. Add a **Google Sheets** node named `Update Status to Paid` (`n8n-nodes-base.googleSheets`). Set operation to `Update`, sheet to `Bills`, matching columns to `row_number`, setting field `status` to `paid`. Connect the true branch of `IF Row Found?` here.

#### Step 6: Build Automated Daily Reminders
1. Add a **Schedule Trigger** node named `Cron - Check Reminders (Daily 7PM)` (`n8n-nodes-base.scheduleTrigger`) configured to run daily at hour 19:00.
2. Add a **Set** node named `Config1` (`n8n-nodes-base.set`) setting currency parameters (`currencyCode` = `IDR`, `currencySymbol` = `Rp`). Connect the schedule trigger here.
3. Add a **Google Sheets** node named `Get Unpaid Bills` (`n8n-nodes-base.googleSheets`) reading rows from sheet `Bills`. Connect `Config1` here.
4. Add a **Code** node named `Calculate Reminder Level & Anti-Spam Guard` (`n8n-nodes-base.code`). Insert the anti-spam aging calculation script determining `gentle`, `firm`, or `savage` levels. Connect `Get Unpaid Bills` here.
5. Add an **IF** node named `IF Reminder Needed?` (`n8n-nodes-base.if`) verifying `{{$json.level}}` is not empty. Connect the previous code node here.
6. Add a **Basic LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`) connected to the true branch, prompting for humorous escalating reminder copy.
7. Connect an **OpenRouter Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) using model `deepseek/deepseek-v4-flash` to the LLM chain.
8. Add a **WAHA** node named `send message : chatting2` (`@devlikeapro/n8n-nodes-waha.WAHA`) sending text to debtor JIDs. Connect the LLM chain here.
9. Add a **Google Sheets** node named `Update lastReminderSent` (`n8n-nodes-base.googleSheets`). Set operation to `Update`, targeting sheet `Bills`, matching by `row_number`, and updating `lastReminderSent`. Connect `send message : chatting2` here to terminate the branch.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Spreadsheet Template Reference | Target Google Sheets Document ID: `1vnI1Y4HoW2zQXEqDP9fP80-K_sH3HJQDvhAVtQARJLo` containing tabs: `Session`, `Contacts`, and `Bills`. |
| WAHA API Documentation | WAHA (WhatsApp HTTP API) integration reference: [DevLikeAPro WAHA Documentation](https://waha.devlike.pro/) |
| OpenRouter Integration Setup | OpenRouter API endpoint configurations for accessing DeepSeek models: [OpenRouter Documentation](https://openrouter.ai/docs) |