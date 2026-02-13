Debounce Telegram support messages and aggregate replies with OpenAI

https://n8nworkflows.xyz/workflows/debounce-telegram-support-messages-and-aggregate-replies-with-openai-13070


# Debounce Telegram support messages and aggregate replies with OpenAI

## 1. Workflow Overview

**Purpose:** Debounce/aggregate rapid Telegram support messages from the same user into a single “conversation”, wait for a short window (60s), then generate one comprehensive reply using OpenAI, send it back on Telegram, and close the session in PostgreSQL.

**Typical use cases**
- Support bots where users often send multiple short messages in a row
- Reducing LLM calls and preventing fragmented bot replies
- Creating a “single coherent answer” after the user finishes typing

### 1.1 Entry & Message Normalization
Receives inbound messages (Telegram Trigger and/or a Webhook) and normalizes fields (user id, text, timestamp).

### 1.2 Session Detection & Persistence (PostgreSQL)
Checks if an active session exists for the user; if not, creates one and starts a 60s collection window; if yes, appends the message and immediately acknowledges receipt.

### 1.3 Debounce Window (Wait + Resume URL)
Stores the execution resume URL in the session record (for traceability) and waits 60 seconds before processing.

### 1.4 Aggregation & AI Response
Fetches the latest session messages, formats them into a conversation string, calls the OpenAI-backed agent, and outputs a reply.

### 1.5 Delivery & Cleanup
Sends the AI response to Telegram and marks the session as completed in PostgreSQL.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Message Normalization
**Overview:** Ingests user messages from Telegram (primary path) or via a webhook (alternate entry), then extracts a consistent schema used by downstream nodes.

**Nodes involved:**
- Telegram Trigger
- Incoming Message Webhook
- Extract User Data

#### Node: Telegram Trigger
- **Type / role:** `n8n-nodes-base.telegramTrigger` — entry point receiving Telegram updates.
- **Configuration choices:**
  - Updates: `message` (only standard message updates)
  - Uses Telegram credentials (`[DebounceAI/Naveen]Telegram account`)
- **Inputs/outputs:**
  - No inputs (trigger).
  - Output connected to **Extract User Data**.
- **Edge cases / failures:**
  - Telegram webhook misconfiguration (wrong URL, revoked token)
  - Non-text messages (photos, stickers) may not have `message.text` and will break downstream expressions unless handled.
- **Version notes:** TypeVersion `1.2`.

#### Node: Incoming Message Webhook
- **Type / role:** `n8n-nodes-base.webhook` — alternate entry point via HTTP.
- **Configuration choices:**
  - Path: `support-webhook`
  - Response mode: `responseNode` (expects a Respond to Webhook node in that execution path)
- **Inputs/outputs:**
  - No inputs (trigger).
  - Output connected to **Extract User Data**.
- **Edge cases / failures:**
  - If this webhook receives payloads not shaped like Telegram updates, `Extract User Data` expressions will fail.
  - Without a reachable Respond to Webhook in the same execution path, callers may time out.
- **Version notes:** TypeVersion `2`.

#### Node: Extract User Data
- **Type / role:** `n8n-nodes-base.set` — normalizes incoming payload.
- **Configuration choices (interpreted):**
  - Creates fields:
    - `user_id` (number): `{{$json.message.from.id}}`
    - `message` (string): `{{$json.message.text}}`
    - `timestamp` (string): `{{$now}}`
- **Inputs/outputs:**
  - Input from Telegram Trigger and Incoming Message Webhook.
  - Output to **Check Active Session**.
- **Edge cases / failures:**
  - Missing `message.from.id` or `message.text` (non-text updates, edited messages, callback queries, custom webhook payloads).
  - `timestamp` stored as string; DB schema uses timestamps—this workflow inserts timestamp strings into JSONB, which is fine, but be consistent.
- **Version notes:** TypeVersion `3.4`.

---

### Block 2 — Session Detection & Persistence (PostgreSQL)
**Overview:** Determines whether the user already has an active session within the debounce window. Creates a new session if none exists; otherwise appends the message to the existing session. The “append” path immediately responds to the caller.

**Nodes involved:**
- Check Active Session
- If
- Create New Session
- Append Message
- Respond to Webhook

#### Node: Check Active Session
- **Type / role:** `n8n-nodes-base.postgres` — queries for an active session.
- **Configuration choices:**
  - Operation: Execute Query
  - Query checks:
    - `user_id = '{{ $json.user_id }}'`
    - `status IN ('LISTENING', 'AGGREGATING')`
    - `wait_expires_at > NOW()`
- **Inputs/outputs:**
  - Input from **Extract User Data**.
  - Output to **If**.
- **Important behavior:**
  - `alwaysOutputData: true` ensures output continues even if query returns no rows (commonly needed for “no session found” branching).
- **Edge cases / failures:**
  - SQL injection risk is low here because user_id is numeric from Telegram, but the query is string-interpolated; parameterized queries are safer.
  - DB connectivity/auth issues; missing table/index; `wait_expires_at` null handling.
- **Version notes:** TypeVersion `2.5`.
- **Credentials:** Postgres (`[Railway/Naveen]Postgres account`).

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — branches based on whether the session query returned rows.
- **Configuration choices (interpreted):**
  - Condition uses: `{{$json.isEmpty()}}` and checks it is `true`.
  - **True branch** → Create New Session (means: no active session found).
  - **False branch** → Append Message (means: session exists).
- **Inputs/outputs:**
  - Input from **Check Active Session**.
  - Output(0/true) to **Create New Session**, Output(1/false) to **Append Message**.
- **Edge cases / failures:**
  - `$json.isEmpty()` depends on the shape of the Postgres node output. If Postgres returns an item with metadata instead of an empty object/array, this may not behave as intended.
  - Consider explicitly checking number of returned rows if needed.
- **Version notes:** TypeVersion `2.3`.

#### Node: Create New Session
- **Type / role:** `n8n-nodes-base.postgres` — inserts a new session row.
- **Configuration choices:**
  - Inserts:
    - `user_id` from `Extract User Data`
    - `session_id` as `gen_random_uuid()`
    - `messages` as a JSONB array with one object `{message, timestamp}`
    - `first_message_at = NOW()`
    - `wait_expires_at = NOW() + INTERVAL '60 seconds'`
    - `status = 'LISTENING'`
  - Returns inserted row (`RETURNING *`)
- **Inputs/outputs:**
  - Input from **If** (true branch).
  - Output to **Store Resume URL**.
- **Edge cases / failures:**
  - Requires `gen_random_uuid()` (typically from `pgcrypto` extension). If not enabled, insert will fail.
  - If `user_id` is PRIMARY KEY (as in the provided sticky schema), repeated sessions per user conflict. The workflow assumes one row per user; that’s compatible, but then inserting when a row exists will fail unless you delete/overwrite. (Your schema sets `user_id` as PK; this insert will fail if user already exists.)
  - The workflow uses `messages` type `jsonb[]`; schema matches that.
- **Version notes:** TypeVersion `2.5`.

#### Node: Append Message
- **Type / role:** `n8n-nodes-base.postgres` — updates existing session by adding another message.
- **Configuration choices:**
  - Updates `messages = messages || jsonb_build_object(...)::jsonb`
  - Sets `status = 'AGGREGATING'`
  - Filters by `user_id` and `status IN ('LISTENING','AGGREGATING')`
  - Returns updated row
- **Inputs/outputs:**
  - Input from **If** (false branch).
  - Output to **Respond to Webhook**.
- **Edge cases / failures:**
  - `messages || jsonb` against a `jsonb[]` column is type-sensitive; in PostgreSQL, concatenating arrays is usually `messages || ARRAY[jsonb_build_object(...)]`. As written, this may error depending on how PostgreSQL resolves types.
  - No update to `wait_expires_at` to extend the window; this means the 60s window is anchored to session creation and does not “slide” with each new message.
- **Version notes:** TypeVersion `2.5`.

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — responds to webhook calls.
- **Configuration choices:**
  - Respond with JSON: `{"status":"accepted","message":"Message received"}`
- **Inputs/outputs:**
  - Input from **Append Message**.
  - No downstream nodes.
- **Edge cases / failures:**
  - Only useful for the Webhook trigger path. In the Telegram trigger path, this node is not used.
  - If the Incoming Message Webhook trigger is used and execution goes through “Create New Session” path, there is no Respond to Webhook on that branch; the HTTP caller may not receive a response (depending on n8n behavior/timeouts).
- **Version notes:** TypeVersion `1.1`.

---

### Block 3 — Debounce Window (Wait + Resume URL)
**Overview:** After creating a session, stores the execution resume URL in the database and waits 60 seconds before continuing processing.

**Nodes involved:**
- Store Resume URL
- Wait 60s (New Session)

#### Node: Store Resume URL
- **Type / role:** `n8n-nodes-base.postgres` — persists `$execution.resumeUrl` in the session row.
- **Configuration choices:**
  - Updates `resume_url` using `{{$execution.resumeUrl}}`
  - Targets row by `session_id = '{{$json.session_id}}'` (expects prior node returned it)
- **Inputs/outputs:**
  - Input from **Create New Session**.
  - Output to **Wait 60s (New Session)**.
- **Edge cases / failures:**
  - If Create New Session did not return `session_id` (or naming differs), update will affect 0 rows.
  - `$execution.resumeUrl` can be empty/undefined depending on environment/execution mode.
- **Version notes:** TypeVersion `2.5`.

#### Node: Wait 60s (New Session)
- **Type / role:** `n8n-nodes-base.wait` — debouncing timer.
- **Configuration choices:**
  - Amount: `60` (seconds)
- **Inputs/outputs:**
  - Input from **Store Resume URL**.
  - Output to **Fetch All Messages**.
- **Edge cases / failures:**
  - If n8n restarts, wait state handling depends on your n8n queue/execution persistence setup.
  - High volume can create many waiting executions.
- **Version notes:** TypeVersion `1.1`.

---

### Block 4 — Aggregation & AI Response
**Overview:** Reads the latest session from PostgreSQL, converts the stored messages into a single conversation string, and sends it to an OpenAI-backed agent to generate a consolidated support reply.

**Nodes involved:**
- Fetch All Messages
- Format All Messages
- OpenAI Chat Model
- AI Agent

#### Node: Fetch All Messages
- **Type / role:** `n8n-nodes-base.postgres` — fetches the most recent session row for a user.
- **Configuration choices:**
  - Query: select by `user_id` from Extract User Data, order by `first_message_at DESC`, limit 1
- **Inputs/outputs:**
  - Input from **Wait 60s (New Session)**.
  - Output to **Format All Messages**.
- **Edge cases / failures:**
  - Relies on `Extract User Data` still being in scope; n8n expressions referencing other nodes are fine, but ensure the node exists and naming is unchanged.
  - If multiple rows per user exist, “latest” works; if schema uses `user_id` as primary key, there will only be one row.
- **Version notes:** TypeVersion `2.5`.

#### Node: Format All Messages
- **Type / role:** `n8n-nodes-base.code` — transforms messages array into a single promptable text.
- **Configuration choices (interpreted):**
  - Takes first input item as `session`
  - Assumes `session.messages` is an array
  - Builds:
    - `conversation`: “Message 1: …” blocks joined with blank lines
    - `user_id`, `message_count`
- **Inputs/outputs:**
  - Input from **Fetch All Messages**.
  - Output to **AI Agent**.
- **Edge cases / failures:**
  - If `messages` is not an array (or null), code throws.
  - If message objects don’t have `.message`, conversation will contain `undefined`.
- **Version notes:** TypeVersion `2`.

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM for the agent node.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - No extra model options configured
- **Inputs/outputs:**
  - Connected to **AI Agent** via the `ai_languageModel` channel.
- **Edge cases / failures:**
  - Invalid/expired API key, quota exceeded
  - Model name not available in your OpenAI account/region
  - Rate limiting or timeouts
- **Version notes:** TypeVersion `1.3`.
- **Credentials:** OpenAI (`iRocket OpenAi account`).

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates the final response text.
- **Configuration choices (interpreted):**
  - Prompt is defined directly (“define”)
  - Prompt template includes `{{$json.conversation}}`
  - Asks for a “helpful and comprehensive response addressing all messages”
- **Inputs/outputs:**
  - Main input from **Format All Messages**.
  - Language model input from **OpenAI Chat Model**.
  - Output to **Send a text message**.
- **Edge cases / failures:**
  - If `conversation` missing, agent prompt degrades.
  - Output field name is assumed later as `{{$json.output}}` in Telegram send; if agent returns a different structure, sending will fail.
- **Version notes:** TypeVersion `1.7`.

---

### Block 5 — Delivery & Cleanup
**Overview:** Sends the AI-generated response back to the Telegram user and marks the session as completed in PostgreSQL.

**Nodes involved:**
- Send a text message
- Clear Session

#### Node: Send a text message
- **Type / role:** `n8n-nodes-base.telegram` — sends Telegram message to the user.
- **Configuration choices:**
  - Text: `{{$json.output}}` (expects AI Agent output)
  - Chat ID: `{{$('Format All Messages').item.json.user_id}}`
  - Append attribution: disabled
- **Inputs/outputs:**
  - Input from **AI Agent**.
  - Output to **Clear Session**.
- **Edge cases / failures:**
  - If `user_id` is numeric Telegram chat id, it works for direct messages; for groups/supergroups, chat id differs.
  - If `Format All Messages` is renamed or produces multiple items, the expression may resolve unexpectedly.
  - Bot can’t message users who never started the bot / blocked bot.
- **Version notes:** TypeVersion `1.2`.
- **Credentials:** Telegram (`[DebounceAI/Naveen]Telegram account`).

#### Node: Clear Session
- **Type / role:** `n8n-nodes-base.postgres` — marks session completed.
- **Configuration choices:**
  - `UPDATE user_sessions SET status = 'COMPLETED' WHERE user_id = '{{ $('Extract User Data').item.json.user_id }}'`
- **Inputs/outputs:**
  - Input from **Send a text message**.
  - No downstream nodes.
- **Edge cases / failures:**
  - If schema uses enum `session_status`, ensure `'COMPLETED'` is a valid enum member (it is in the provided schema note).
  - This does not clear the messages array; it only updates status.
- **Version notes:** TypeVersion `2.5`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Receive Telegram messages | — | Extract User Data | Step 1: Receive Message — Telegram Trigger listens for incoming messages from users. Extract user data including: User ID, Message text, Timestamp |
| Incoming Message Webhook | webhook | Alternate HTTP entry point | — | Extract User Data |  |
| Extract User Data | set | Normalize payload fields | Telegram Trigger; Incoming Message Webhook | Check Active Session | Step 1: Receive Message — Telegram Trigger listens for incoming messages from users. Extract user data including: User ID, Message text, Timestamp |
| Check Active Session | postgres | Find active debounce session | Extract User Data | If | Step 2: Check for Active Session — Queries PostgreSQL to see if user has an active session within the last 60 seconds. If NO session → Create new session; If session EXISTS → Append message to existing session. Status options: LISTENING, AGGREGATING, COMPLETED |
| If | if | Branch on session existence | Check Active Session | Create New Session; Append Message | Step 2: Check for Active Session — Queries PostgreSQL to see if user has an active session within the last 60 seconds. If NO session → Create new session; If session EXISTS → Append message to existing session. Status options: LISTENING, AGGREGATING, COMPLETED |
| Create New Session | postgres | Insert a new session row | If | Store Resume URL | Step 2: Check for Active Session — Queries PostgreSQL to see if user has an active session within the last 60 seconds. If NO session → Create new session; If session EXISTS → Append message to existing session. Status options: LISTENING, AGGREGATING, COMPLETED |
| Store Resume URL | postgres | Save execution resume URL | Create New Session | Wait 60s (New Session) | Step 3: Wait for More Messages — 60-second window to collect all user messages. Adjust timer here based on your use case! |
| Wait 60s (New Session) | wait | Debounce timer | Store Resume URL | Fetch All Messages | Step 3: Wait for More Messages — 60-second window to collect all user messages. Adjust timer here based on your use case! |
| Fetch All Messages | postgres | Load latest session for user | Wait 60s (New Session) | Format All Messages | Step 4: Process with AI — Fetch all messages, format, send to OpenAI. Tip: Customize the AI prompt in the Agent node for different response styles |
| Format All Messages | code | Build conversation string | Fetch All Messages | AI Agent | Step 4: Process with AI — Fetch all messages, format, send to OpenAI. Tip: Customize the AI prompt in the Agent node for different response styles |
| OpenAI Chat Model | lmChatOpenAi | LLM provider for agent | — | AI Agent (ai_languageModel) | Step 4: Process with AI — Fetch all messages, format, send to OpenAI. Tip: Customize the AI prompt in the Agent node for different response styles |
| AI Agent | langchain.agent | Generate consolidated reply | Format All Messages; OpenAI Chat Model | Send a text message | Step 4: Process with AI — Fetch all messages, format, send to OpenAI. Tip: Customize the AI prompt in the Agent node for different response styles |
| Send a text message | telegram | Deliver reply to user | AI Agent | Clear Session | Step 5: Deliver Response — Send AI-generated response back to user via Telegram, then clear the session status to COMPLETED. Ready for the next conversation! |
| Clear Session | postgres | Mark session completed | Send a text message | — | Step 5: Deliver Response — Send AI-generated response back to user via Telegram, then clear the session status to COMPLETED. Ready for the next conversation! |
| Append Message | postgres | Add message to existing session | If | Respond to Webhook | Step 2: Check for Active Session — Queries PostgreSQL to see if user has an active session within the last 60 seconds. If NO session → Create new session; If session EXISTS → Append message to existing session. Status options: LISTENING, AGGREGATING, COMPLETED |
| Respond to Webhook | respondToWebhook | HTTP response for webhook path | Append Message | — |  |
| Sticky Note | stickyNote | Canvas documentation | — | — | # Intelligent Message Debouncing for Telegram Support Bot … 📺 [Setup Video Guide] (if you create one) |
| Sticky Note1 | stickyNote | Canvas documentation | — | — | ## Step 1: Receive Message … |
| Sticky Note2 | stickyNote | Canvas documentation | — | — | ## Step 2: Check for Active Session … |
| Sticky Note3 | stickyNote | Canvas documentation | — | — | ## Step 3: Wait for More Messages … |
| Sticky Note4 | stickyNote | Canvas documentation | — | — | ## Step 4: Process with AI … |
| Sticky Note5 | stickyNote | Canvas documentation | — | — | ## Step 5: Deliver Response … |
| Sticky Note6 | stickyNote | Canvas documentation | — | — | ⚠️ IMPORTANT: Database Setup Required … (SQL schema block) |
| Sticky Note7 | stickyNote | Canvas documentation | — | — | 🔐 Security Checklist … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **Telegram API credential** (Bot token from BotFather).
   2. **OpenAI API credential** (key with access to `gpt-4.1-mini` or adjust model).
   3. **PostgreSQL credential** (host/port/db/user/password/SSL as required).

2) **Prepare PostgreSQL**
   1. Create the `user_sessions` table and related types/indexes (see note in “General Notes & Resources”).
   2. Ensure `gen_random_uuid()` works (enable `pgcrypto` if needed).

3) **Add Trigger: Telegram Trigger**
   1. Node type: **Telegram Trigger**
   2. Updates: **message**
   3. Select your Telegram credential.
   4. This is the primary entry point.

4) **(Optional) Add Trigger: Webhook**
   1. Node type: **Webhook**
   2. Path: `support-webhook`
   3. Response mode: **Using “Respond to Webhook” node**
   4. Use only if you intend to post Telegram-like payloads to HTTP.

5) **Add “Extract User Data” (Set node)**
   1. Node type: **Set**
   2. Add fields:
      - `user_id` (Number) = `{{$json.message.from.id}}`
      - `message` (String) = `{{$json.message.text}}`
      - `timestamp` (String) = `{{$now}}`
   3. Connect:
      - Telegram Trigger → Extract User Data
      - (Optional) Webhook → Extract User Data

6) **Add “Check Active Session” (Postgres)**
   1. Node type: **Postgres** → Operation: *Execute Query*
   2. Query:
      - Select active session for the user where `status IN ('LISTENING','AGGREGATING')` and `wait_expires_at > NOW()`.
   3. Enable **Always Output Data**.
   4. Connect: Extract User Data → Check Active Session.

7) **Add “If”**
   1. Node type: **If**
   2. Condition to detect “no rows” result from Postgres.
      - In the provided workflow this is `{{$json.isEmpty()}} is true`.
   3. Connect: Check Active Session → If.

8) **Add “Create New Session” (Postgres)**
   1. Node type: **Postgres** → Execute Query
   2. Insert a new row with:
      - `user_id` from Extract User Data
      - new `session_id` (`gen_random_uuid()`)
      - `messages` initialized as JSONB array containing `{message,timestamp}`
      - `wait_expires_at = NOW() + INTERVAL '60 seconds'`
      - `status = 'LISTENING'`
      - `RETURNING *`
   3. Connect: If (true/no session) → Create New Session.

9) **Add “Store Resume URL” (Postgres)**
   1. Node type: **Postgres** → Execute Query
   2. Update `resume_url = {{$execution.resumeUrl}}` where `session_id = {{$json.session_id}}`.
   3. Connect: Create New Session → Store Resume URL.

10) **Add “Wait 60s (New Session)”**
   1. Node type: **Wait**
   2. Amount: `60` seconds
   3. Connect: Store Resume URL → Wait 60s.

11) **Add “Fetch All Messages” (Postgres)**
   1. Node type: **Postgres** → Execute Query
   2. Query latest session for the user: order by `first_message_at DESC limit 1`
   3. Connect: Wait 60s → Fetch All Messages.

12) **Add “Format All Messages” (Code)**
   1. Node type: **Code**
   2. JS logic:
      - read `session.messages`
      - build `conversation` string (Message 1…, Message 2…)
      - return `{ user_id, conversation, message_count }`
   3. Connect: Fetch All Messages → Format All Messages.

13) **Add “OpenAI Chat Model”**
   1. Node type: **OpenAI Chat Model** (LangChain)
   2. Model: `gpt-4.1-mini` (or change)
   3. Select your OpenAI credential.

14) **Add “AI Agent”**
   1. Node type: **AI Agent** (LangChain Agent)
   2. Prompt type: **Define**
   3. Prompt includes `{{$json.conversation}}`
   4. Connect:
      - Format All Messages → AI Agent (main)
      - OpenAI Chat Model → AI Agent (ai_languageModel connection)

15) **Add “Send a text message” (Telegram)**
   1. Node type: **Telegram** → operation “Send Message” (text)
   2. Text: `{{$json.output}}`
   3. Chat ID: `{{$('Format All Messages').item.json.user_id}}`
   4. Connect: AI Agent → Send a text message.

16) **Add “Clear Session” (Postgres)**
   1. Node type: **Postgres** → Execute Query
   2. Update: `status='COMPLETED'` for the user.
   3. Connect: Send a text message → Clear Session.

17) **Add “Append Message” (Postgres)**
   1. Node type: **Postgres** → Execute Query
   2. Update existing session row:
      - append `{message,timestamp}` into `messages`
      - set status `AGGREGATING`
   3. Connect: If (false/session exists) → Append Message.

18) **Add “Respond to Webhook”**
   1. Node type: **Respond to Webhook**
   2. Respond with JSON body like `{"status":"accepted","message":"Message received"}`
   3. Connect: Append Message → Respond to Webhook.
   4. Note: if you use the Webhook trigger, consider also responding on the “Create New Session” branch to avoid HTTP timeouts.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| # Intelligent Message Debouncing for Telegram Support Bot — Prevents responding to every message by aggregating messages, waiting 60s, then generating one comprehensive OpenAI response; requirements: Telegram bot token, OpenAI API key, PostgreSQL; customization: adjust wait time, prompt style. “📺 [Setup Video Guide] (if you create one)” | Sticky note (canvas overview) |
| ⚠️ IMPORTANT: Database Setup Required — Includes full PostgreSQL schema (enum `session_status`, table `user_sessions`, indexes, updated_at trigger, cleanup function, optional pg_cron schedule). PostgreSQL 13+ required for JSONB support. | Sticky note (database schema block) |
| 🔐 Security Checklist — Remove test credentials, clear personal Telegram chat IDs, remove DB connection strings, use n8n credentials instead of hardcoding. | Sticky note (security) |
| Disclaimer (provided): The text comes exclusively from an automated n8n workflow; compliant with applicable content policies; no illegal/offensive/protected elements; all manipulated data is legal and public. | Project/context note |