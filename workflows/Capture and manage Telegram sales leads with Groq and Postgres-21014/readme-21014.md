Capture and manage Telegram sales leads with Groq and Postgres

https://n8nworkflows.xyz/workflows/capture-and-manage-telegram-sales-leads-with-groq-and-postgres-21014


# Capture and manage Telegram sales leads with Groq and Postgres

### 1. Workflow Overview

This workflow is a multilingual sales lead management assistant designed to capture, process, and store prospective client information from Telegram. It allows users to send voice notes (in English, Hindi, or Hinglish), business card images, or typed text, utilizing Groq AI models to structure the unstructured input. The workflow subsequently queries a PostgreSQL database to detect duplicates, prompts the user with inline Telegram buttons to confirm lead creation or updates, and runs a scheduled daily digest at 08:30 for follow-up reminders and stale lead tracking.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Authorization:** Captures Telegram messages, button interactions, and cron triggers; normalizes input payloads; injects configuration variables; and performs chat ID whitelist validation.
- **1.2 Input Normalization & Routing:** Routes incoming payloads depending on type (voice note, image, text message, callback query, or command) and handles file downloads or help responses.
- **1.3 AI Processing & Transcription:** Converts raw audio via Groq Whisper or business card images via Groq Vision into textual descriptions, then extracts structured JSON CRM parameters using Groq Chat Completions.
- **1.4 Duplicate Matching & Confirmation:** Queries PostgreSQL to find up to three matching existing leads based on fuzzy matching and contact details, builds an interactive preview message, saves temporary pending state, and dispatches Telegram inline keyboards.
- **1.5 Action Handling & Database Persistence:** Intercepts button callbacks, claims expired/pending note tokens safely, updates or creates lead records and interaction logs in PostgreSQL, and replies with confirmation statuses.
- **1.6 Scheduled & On-Demand Digests:** Runs database queries to pull due/overdue follow-ups or cold leads, formatting a summary message delivered automatically every morning or via command input.

---

### 2. Block-by-Block Analysis

---

### 2.1 Input Reception & Authorization

#### Overview
This block serves as the dual entry point for the automation. It accepts real-time webhooks from Telegram users and a daily cron schedule, initializes global operational parameters, cleans up expired database tokens, and screens out unauthorized users.

#### Nodes Involved
- `Telegram Trigger`
- `Daily 8:30 Digest`
- `Config`
- `Is Telegram Update?`
- `Normalize Update`
- `Fraud ?`
- `Reply: Unauthorized`
- `Cleanup Pending`
- `redirect`

#### Node Details

- **Telegram Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.telegramTrigger` (Trigger Node). Listens for incoming updates (`message`, `callback_query`) via webhook.
  - **Configuration Choices:** Configured to watch messages and callback queries.
  - **Key Expressions:** Uses webhook registration.
  - **Connections:** Output connects to `Config`.
  - **Edge Cases:** Webhook delivery failures if Telegram API endpoints are unreachable.

- **Daily 8:30 Digest**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node). Fires a cron schedule every day at 08:30.
  - **Configuration Choices:** Cron expression set to `30 8 * * *`.
  - **Connections:** Output branches to `Config` and `Cleanup Pending`.

- **Config**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Injects global settings (`ALLOWED_CHAT_IDS`, `TIMEZONE`, `COLD_AFTER_DAYS`, Groq model identifiers, and Whisper translation prompts) into the execution stream.
  - **Configuration Choices:** JavaScript execution block defining user whitelist and model slugs (`whisper-large-v3`, `openai/gpt-oss-120b`, `qwen/qwen3.8-27b`).
  - **Connections:** Input from triggers (`Telegram Trigger`, `Daily 8:30 Digest`); output connects to `Is Telegram Update?`.
  - **Edge Cases:** Throws syntax issues if JSON objects are improperly formatted.

- **Is Telegram Update?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Evaluates whether the incoming payload contains a valid Telegram update structure.
  - **Configuration Choices:** Checks if `update_id` is defined and not null.
  - **Connections:** True branch leads to `Normalize Update`; false branch leads to `redirect` (running digest query).

- **Normalize Update**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Standardizes disparate Telegram input payloads into a unified format (`kind`, `chat_id`, `user_name`, `text`, `file_id`, `callback_data`, etc.).
  - **Configuration Choices:** Enforces a 3MB image size limit on attachments. Classifies commands like `/today`.
  - **Connections:** Input from `Is Telegram Update?`; output connects to `Fraud ?`.

- **Fraud ?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Security gate verifying if the sender's chat ID is included in the authorized whitelist.
  - **Configuration Choices:** Evaluates `authorized` boolean flag.
  - **Connections:** True (unauthorized) branch goes to `Reply: Unauthorized`; false (authorized) branch goes to `Route by Kind`.

- **Reply: Unauthorized**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Sends an automated refusal message revealing the unauthorized user's chat ID for onboarding.
  - **Configuration Choices:** Parses HTML markup displaying the unwhitelisted `chat_id`.
  - **Connections:** Input from `Fraud ?`.

- **Cleanup Pending**
  - **Type & Technical Role:** `n8n-nodes-base.postgres` (Database Node). Deletes pending confirmation notes older than 2 days from the database.
  - **Configuration Choices:** Runs raw SQL delete statement against `pending_notes`. Requires PostgreSQL credentials.
  - **Connections:** Input from `Daily 8:30 Digest`.

- **redirect**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` (Utility Node). Acts as a pass-through connection bypass point.
  - **Connections:** Input from `Is Telegram Update?` false branch; output routes to `Digest Query`.

---

### 2.2 Input Normalization & Routing

#### Overview
This block intercepts verified Telegram payloads and branches execution down distinct processing pipelines based on media type, user command, or button interaction.

#### Nodes Involved
- `Route by Kind`
- `Get Voice File`
- `Fix Audio Filename`
- `Transcribe (Groq Whisper)`
- `Set Voice Text`
- `Get Photo File`
- `Build Vision Request`
- `Read Card (Groq Vision)`
- `Set Card Text`
- `Set Text`
- `Reply: Help`
- `Reply: Unsupported`
- `redirect1`
- `redirect2`
- `Reply: Error`

#### Node Details

- **Route by Kind**
  - **Type & Technical Role:** `n8n-nodes-base.switch` (Switch Node). Directs workflow execution based on the normalized `kind` property (`voice`, `photo`, `text`, `callback`, `cmd_help`, `cmd_today`, or `other`).
  - **Connections:** Inputs from `Fraud ?`; outputs branch to corresponding handlers (Voice, Photo, Text, Callback, Help, Digest query, or Unsupported reply).

- **Get Voice File**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Integration Node). Downloads binary voice/audio files from Telegram servers using the provided `file_id`.
  - **Connections:** Input from `Route by Kind` (voice branch); output connects to `Fix Audio Filename`.

- **Fix Audio Filename**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Renames downloaded Telegram `.oga` audio files to `.ogg` to ensure compliance with downstream Groq Whisper API requirements.
  - **Connections:** Input from `Get Voice File`; output connects to `Transcribe (Groq Whisper)`.
  - **Edge Cases:** Throws runtime errors if binary buffers are empty.

- **Transcribe (Groq Whisper)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request Node). Sends audio binary data to Groq's Whisper transcription API.
  - **Configuration Choices:** HTTP POST request to `https://api.groq.com/openai/v1/audio/transcriptions` utilizing generic Header Authentication. Sends multipart form-data containing the audio file, whisper model ID, and prompt hints. Configured with 3 max retries.
  - **Connections:** Input from `Fix Audio Filename`; output connects to `Set Voice Text` (success) or `redirect1` (error).

- **Set Voice Text**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Wraps the Whisper transcription result into a unified text payload object.
  - **Connections:** Input from `Transcribe (Groq Whisper)`; output connects to `Has Text?`.

- **Get Photo File**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Integration Node). Downloads image binary streams (business cards) using `file_id`.
  - **Connections:** Input from `Route by Kind` (photo branch); output connects to `Build Vision Request`.

- **Build Vision Request**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Converts binary image buffers into base64 data strings and constructs the JSON payload required by Groq’s Vision model.
  - **Connections:** Input from `Get Photo File`; output connects to `Read Card (Groq Vision)`.

- **Read Card (Groq Vision)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request Node). Calls Groq’s chat completion endpoint with image data to read text off business cards.
  - **Configuration Choices:** HTTP POST to `https://api.groq.com/openai/v1/chat/completions` using Header Auth.
  - **Connections:** Input from `Build Vision Request`; output connects to `Set Card Text` (success) or `redirect1` (error).

- **Set Card Text**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Parses Vision model responses, strips out reasoning tokens (`<think>` blocks), and prepends optional user captions.
  - **Connections:** Input from `Read Card (Groq Vision)`; output connects to `Has Text?`.

- **Set Text**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Extracts direct text messages from normalized input objects.
  - **Connections:** Input from `Route by Kind` (text branch); output connects to `Has Text?`.

- **Reply: Help**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Sends instructions and guidance text regarding voice notes, business cards, and commands to the user.
  - **Connections:** Input from `Route by Kind`.

- **Reply: Unsupported**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Informs users when sent media types fall outside supported parameters.
  - **Connections:** Input from `Route by Kind`.

- **redirect1** & **redirect2**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` (Utility Nodes). Pass-through junction nodes used to multiplex error paths from external HTTP requests into a single unified error handler.
  - **Connections:** Aggregate failure outputs from Whisper and Vision nodes.

- **Reply: Error**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Sends a fallback warning notification when API transcription or vision analysis fails.
  - **Connections:** Input from `redirect2`.

---

### 2.3 AI Extraction

#### Overview
This block processes unstructured transcription or card text through an LLM extraction pipeline, enforcing rigid JSON schemas to generate structured contact profiles, intent classifications, and confidence vectors.

#### Nodes Involved
- `Has Text?`
- `Reply: Nothing Heard`
- `Reply User`
- `Build Extraction Request`
- `Extract Lead (Groq)`
- `Parse & Validate`
- `Parse OK?`
- `Reply: Parse Failed`

#### Node Details

- **Has Text?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Verifies if extracted or typed text actually contains content.
  - **Connections:** Inputs from `Set Voice Text`, `Set Card Text`, and `Set Text`; true branch goes to `Build Extraction Request`, false branch goes to `Reply: Nothing Heard`.

- **Reply: Nothing Heard**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Prompts the user when an audio note or image yielded zero processable text.
  - **Connections:** Input from `Has Text?` false branch; output routes to `Reply User`.

- **Reply User**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` (Utility Node). Termination pass-through node for minor notification branches.

- **Build Extraction Request**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Builds the comprehensive system prompt, current timestamp constraints, timezone references, and JSON schema structure for lead attribute extraction.
  - **Connections:** Input from `Has Text?` true branch; output connects to `Extract Lead (Groq)`.

- **Extract Lead (Groq)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request Node). Calls Groq Chat Completions API enforcing strict JSON schema output formatting (`json_schema`).
  - **Configuration Choices:** HTTP POST to `https://api.groq.com/openai/v1/chat/completions` using Header Authentication. Retries up to 3 times on failure.
  - **Connections:** Input from `Build Extraction Request`; outputs connect to `Parse & Validate` (success) or `redirect2` (error).

- **Parse & Validate**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Parses AI response strings, sanitizes phone numbers and email formats, verifies calendar dates against past constraints, calculates confidence metrics, and flags low-confidence variables.
  - **Connections:** Input from `Extract Lead (Groq)`; output connects to `Parse OK?`.

- **Parse OK?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Checks whether the lead payload was parsed and validated successfully.
  - **Connections:** True branch leads to `Find Matches`; false branch leads to `Reply: Parse Failed`.

- **Reply: Parse Failed**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Sends an error notification alerting the user that extraction failed or lacked essential identifiers.
  - **Connections:** Input from `Parse OK?` false branch.

---

### 2.4 Match & Confirm

#### Overview
This block checks PostgreSQL for existing leads matching the extracted parameters using fuzzy string similarity calculations, packages the metadata into a temporary staging table, and dispatches interactive Telegram confirmation cards with inline keyboards.

#### Nodes Involved
- `Find Matches`
- `Build Confirmation`
- `Save Pending`
- `Prepare Keyboard`
- `Candidate Count`
- `Send Confirm (0)`
- `Send Confirm (1)`
- `Send Confirm (2)`
- `Send Confirm (3)`
- `Reply: Error` (Shared failure node)

#### Node Details

- **Find Matches**
  - **Type & Technical Role:** `n8n-nodes-base.postgres` (Database Node). Queries PostgreSQL to search for up to 3 potential matching leads using pg_trgm similarity metrics (`similarity`, `word_similarity`) and exact phone/email matching.
  - **Configuration Choices:** Executes a parameterized SQL query substituting extracted lead details. Requires PostgreSQL credentials.
  - **Connections:** Input from `Parse OK?` true branch; output connects to `Build Confirmation`.

- **Build Confirmation**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Filters candidate matches scoring above 60% similarity, formats HTML warning flags for low-confidence properties, and compiles candidate callback metadata.
  - **Connections:** Input from `Find Matches`; output connects to `Save Pending`.

- **Save Pending**
  - **Type & Technical Role:** `n8n-nodes-base.postgres` (Database Node). Inserts the proposed lead payload and chat ID into the `pending_notes` staging table and returns a unique UUID.
  - **Configuration Choices:** Executes parameterized SQL `INSERT INTO pending_notes`. Requires PostgreSQL credentials.
  - **Connections:** Input from `Build Confirmation`; output connects to `Prepare Keyboard`.

- **Prepare Keyboard**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Dynamically maps candidate match indices to Telegram inline keyboard button payloads (`u:pid:lead_id`, `n:pid`, `c:pid`).
  - **Connections:** Input from `Save Pending`; output connects to `Candidate Count`.

- **Candidate Count**
  - **Type & Technical Role:** `n8n-nodes-base.switch` (Switch Node). Directs workflow routing based on the total number of matched duplicate leads found (0, 1, 2, or 3 matches).
  - **Connections:** Input from `Prepare Keyboard`; outputs connect to matching `Send Confirm` nodes.

- **Send Confirm (0) / Send Confirm (1) / Send Confirm (2) / Send Confirm (3)**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Nodes). Dispatches the formatted HTML confirmation message accompanied by custom inline keypads (updating existing leads, saving as a new lead, or cancelling).
  - **Connections:** Inputs from `Candidate Count` branches.

---

### 2.5 Action Handling & Database Persistence

#### Overview
This block manages incoming button callbacks from users, validates and claims pending note proposals to prevent double-execution race conditions, routes actions to update or create SQL database entries, records interaction history, and dispatches final execution receipts.

#### Nodes Involved
- `Answer Callback`
- `Parse Callback`
- `Valid Callback?`
- `Reply: Bad Button`
- `Claim Pending`
- `Pending Found?`
- `Reply: Expired`
- `Action`
- `Update Lead`
- `Create Lead`
- `Reply: Cancelled`
- `Saved?`
- `Format Saved Reply`
- `Send Saved`
- `Reply: Save Failed`

#### Node Details

- **Answer Callback**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Acknowledges Telegram callback queries to clear loading spinners on the user's client.
  - **Connections:** Input from `Route by Kind` (callback branch); output connects to `Parse Callback`.

- **Parse Callback**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Uses Regular Expressions to extract action codes (`u`, `n`, `c`), pending note UUIDs, and target lead IDs from callback data strings.
  - **Connections:** Input from `Answer Callback`; output connects to `Valid Callback?`.

- **Valid Callback?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Validates structural integrity of parsed callback tokens.
  - **Connections:** True branch connects to `Claim Pending`; false branch connects to `Reply: Bad Button`.

- **Reply: Bad Button**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Notifies the user that an expired or malformed inline button was pressed.
  - **Connections:** Input from `Valid Callback?` false branch.

- **Claim Pending**
  - **Type & Technical Role:** `n8n-nodes-base.postgres` (Database Node). Atomically fetches and deletes the pending note payload from `pending_notes` using row-locking mechanisms to prevent double-processing.
  - **Configuration Choices:** Executes SQL `DELETE FROM pending_notes WHERE id = ... RETURNING payload`. Requires PostgreSQL credentials.
  - **Connections:** Input from `Valid Callback?` true branch; output connects to `Pending Found?`.

- **Pending Found?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Checks whether the pending item successfully existed and was claimed.
  - **Connections:** True branch connects to `Action`; false branch connects to `Reply: Expired`.

- **Reply: Expired**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Alerts the user that the confirmation card has already been processed or expired.
  - **Connections:** Input from `Pending Found?` false branch.

- **Action**
  - **Type & Technical Role:** `n8n-nodes-base.switch` (Switch Node). Directs execution according to the selected user choice (`u` for update, `n` for new lead, `c` for cancel).
  - **Connections:** Input from `Pending Found?` true branch; outputs lead to `Update Lead`, `Create Lead`, or `Reply: Cancelled`.

- **Update Lead**
  - **Type & Technical Role:** `n8n-nodes-base.postgres` (Database Node). Updates existing lead records in PostgreSQL, recalculates lead scores via stored functions (`compute_lead_score`), updates touch timestamps, and logs an interaction record.
  - **Configuration Choices:** Executes custom PostgreSQL CTE query combining `UPDATE leads`, `compute_lead_score()`, and `INSERT INTO interactions`. Requires PostgreSQL credentials.
  - **Connections:** Input from `Action` (update branch); output connects to `Saved?`.

- **Create Lead**
  - **Type & Technical Role:** `n8n-nodes-base.postgres` (Database Node). Inserts a new row into the `leads` table, computes initial lead scores, and logs the corresponding interaction record.
  - **Configuration Choices:** Executes custom PostgreSQL CTE query. Requires PostgreSQL credentials.
  - **Connections:** Input from `Action` (new branch); output connects to `Saved?`.

- **Reply: Cancelled**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Sends a cancellation notification confirming that data was discarded.
  - **Connections:** Input from `Action` (cancel branch).

- **Saved?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Verifies whether database insert/update operations returned valid record IDs.
  - **Connections:** Inputs from `Update Lead` and `Create Lead`; true branch connects to `Format Saved Reply`, false branch connects to `Reply: Save Failed`.

- **Format Saved Reply**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Generates HTML receipts detailing lead status changes, updated pipeline stages, and computed lead scores.
  - **Connections:** Input from `Saved?` true branch; output connects to `Send Saved`.

- **Send Saved**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Delivers final success confirmations to the user via Telegram.
  - **Connections:** Input from `Format Saved Reply`.

- **Reply: Save Failed**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Sends failure notifications if database write operations fail.
  - **Connections:** Input from `Saved?` false branch.

---

### 2.6 Scheduled & On-Demand Digests

#### Overview
This block queries PostgreSQL for due/overdue follow-ups and stale leads, structures them into prioritized digest groupings, and dispatches daily summary briefings to authorized Telegram chats.

#### Nodes Involved
- `Digest Query`
- `Build Digest`
- `Send Digest`

#### Node Details

- **Digest Query**
  - **Type & Technical Role:** `n8n-nodes-base.postgres` (Database Node). Queries PostgreSQL to pull non-won/lost leads where follow-up dates match today, are overdue, or exceed stale contact thresholds (`COLD_AFTER_DAYS`).
  - **Configuration Choices:** Executes a parameterized SQL query utilizing timezone adjustments and interaction logs. Requires PostgreSQL credentials.
  - **Connections:** Inputs from `Daily 8:30 Digest` (via redirect) and `Route by Kind` (today command branch); output connects to `Build Digest`.

- **Build Digest**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Code Node). Groups query results into categorized sections (`📅 Follow up today`, `⏰ Overdue`, `🧊 Going cold`) and formats messages with snippet previews.
  - **Connections:** Input from `Digest Query`; output connects to `Send Digest`.

- **Send Digest**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (Action Node). Posts the formatted morning or on-demand digest report to the designated Telegram chat.
  - **Connections:** Input from `Build Digest`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram Trigger | `n8n-nodes-base.telegramTrigger` | Receives incoming Telegram webhooks | None | Config | 1. Receive & Authorize |
| Daily 8:30 Digest | `n8n-nodes-base.scheduleTrigger` | Triggers daily at 08:30 cron | None | Config, Cleanup Pending | 1. Receive & Authorize |
| Config | `n8n-nodes-base.code` | Injects global config variables | Telegram Trigger, Daily 8:30 Digest | Is Telegram Update? | 1. Receive & Authorize |
| Is Telegram Update? | `n8n-nodes-base.if` | Validates if update exists | Config | Normalize Update, redirect | 1. Receive & Authorize |
| Normalize Update | `n8n-nodes-base.code` | Standardizes update payloads | Is Telegram Update? | Fraud ? | 1. Receive & Authorize |
| Fraud ? | `n8n-nodes-base.if` | Screens authorized chat whitelist | Normalize Update | Reply: Unauthorized, Route by Kind | 1. Receive & Authorize |
| Reply: Unauthorized | `n8n-nodes-base.telegram` | Replies with onboarding chat ID | Fraud ? | None | 1. Receive & Authorize |
| Cleanup Pending | `n8n-nodes-base.postgres` | Deletes expired pending records | Daily 8:30 Digest | None | 1. Receive & Authorize |
| redirect | `n8n-nodes-base.noOp` | Pass-through routing node | Is Telegram Update? | Digest Query | |
| Route by Kind | `n8n-nodes-base.switch` | Routes by message media type | Fraud ? | Get Voice File, Get Photo File, Set Text, Answer Callback, Reply: Help, Digest Query, Reply: Unsupported | 2. Input to Text |
| Get Voice File | `n8n-nodes-base.telegram` | Downloads voice note binaries | Route by Kind | Fix Audio Filename | 2. Input to Text |
| Fix Audio Filename | `n8n-nodes-base.code` | Renames audio extension to .ogg | Get Voice File | Transcribe (Groq Whisper) | 2. Input to Text |
| Transcribe (Groq Whisper) | `n8n-nodes-base.httpRequest` | Calls Groq Whisper API | Fix Audio Filename | Set Voice Text, redirect1 | 2. Input to Text |
| Set Voice Text | `n8n-nodes-base.code` | Wraps Whisper text output | Transcribe (Groq Whisper) | Has Text? | 2. Input to Text |
| Get Photo File | `n8n-nodes-base.telegram` | Downloads business card image | Route by Kind | Build Vision Request | 2. Input to Text |
| Build Vision Request | `n8n-nodes-base.code` | Encodes image to base64 for Vision | Get Photo File | Read Card (Groq Vision) | 2. Input to Text |
| Read Card (Groq Vision) | `n8n-nodes-base.httpRequest` | Calls Groq Vision LLM API | Build Vision Request | Set Card Text, redirect1 | 2. Input to Text |
| Set Card Text | `n8n-nodes-base.code` | Cleans LLM reasoning tags | Read Card (Groq Vision) | Has Text? | 2. Input to Text |
| Set Text | `n8n-nodes-base.code` | Extracts raw text message | Route by Kind | Has Text? | 2. Input to Text |
| Reply: Help | `n8n-nodes-base.telegram` | Sends usage help text | Route by Kind | None | |
| Reply: Unsupported | `n8n-nodes-base.telegram` | Replies to unsupported formats | Route by Kind | None | |
| redirect1 | `n8n-nodes-base.noOp` | Error routing junction | Transcribe (Groq Whisper), Read Card (Groq Vision) | redirect2 | |
| redirect2 | `n8n-nodes-base.noOp` | Error routing junction | redirect1, Extract Lead (Groq) | Reply: Error | |
| Reply: Error | `n8n-nodes-base.telegram` | Sends API failure message | redirect2 | None | 10. Error Handling |
| Has Text? | `n8n-nodes-base.if` | Checks if text exists | Set Voice Text, Set Card Text, Set Text | Build Extraction Request, Reply: Nothing Heard | 3. AI Extraction |
| Reply: Nothing Heard | `n8n-nodes-base.telegram` | Prompts user when text is empty | Has Text? | Reply User | 3. AI Extraction |
| Reply User | `n8n-nodes-base.noOp` | Pass-through termination node | Reply: Nothing Heard | None | |
| Build Extraction Request | `n8n-nodes-base.code` | Constructs LLM prompt & JSON schema | Has Text? | Extract Lead (Groq) | 3. AI Extraction |
| Extract Lead (Groq) | `n8n-nodes-base.httpRequest` | Calls Groq Chat Completion API | Build Extraction Request | Parse & Validate, redirect2 | 3. AI Extraction |
| Parse & Validate | `n8n-nodes-base.code` | Validates JSON & checks confidence | Extract Lead (Groq) | Parse OK? | 3. AI Extraction |
| Parse OK? | `n8n-nodes-base.if` | Verifies parse success | Parse & Validate | Find Matches, Reply: Parse Failed | 3. AI Extraction |
| Reply: Parse Failed | `n8n-nodes-base.telegram` | Sends extraction failure warning | Parse OK? | None | 3. AI Extraction |
| Find Matches | `n8n-nodes-base.postgres` | Fuzzy matches leads in Postgres | Parse OK? | Build Confirmation | 4. Match & Confirm |
| Build Confirmation | `n8n-nodes-base.code` | Formats preview card messages | Find Matches | Save Pending | 4. Match & Confirm |
| Save Pending | `n8n-nodes-base.postgres` | Saves pending note proposal | Build Confirmation | Prepare Keyboard | 4. Match & Confirm |
| Prepare Keyboard | `n8n-nodes-base.code` | Builds inline keyboard mappings | Save Pending | Candidate Count | 4. Match & Confirm |
| Candidate Count | `n8n-nodes-base.switch` | Routes by number of matches | Prepare Keyboard | Send Confirm (0), Send Confirm (1), Send Confirm (2), Send Confirm (3) | 4. Match & Confirm |
| Send Confirm (0) | `n8n-nodes-base.telegram` | Sends confirmation (0 matches) | Candidate Count | None | 5. Send Confirmation |
| Send Confirm (1) | `n8n-nodes-base.telegram` | Sends confirmation (1 match) | Candidate Count | None | 5. Send Confirmation |
| Send Confirm (2) | `n8n-nodes-base.telegram` | Sends confirmation (2 matches) | Candidate Count | None | 5. Send Confirmation |
| Send Confirm (3) | `n8n-nodes-base.telegram` | Sends confirmation (3 matches) | Candidate Count | None | 5. Send Confirmation |
| Answer Callback | `n8n-nodes-base.telegram` | Clears loading spinner | Route by Kind | Parse Callback | 6. Button Handling |
| Parse Callback | `n8n-nodes-base.code` | Parses action and ID from callback | Answer Callback | Valid Callback? | 6. Button Handling |
| Valid Callback? | `n8n-nodes-base.if` | Validates callback structure | Parse Callback | Claim Pending, Reply: Bad Button | 6. Button Handling |
| Reply: Bad Button | `n8n-nodes-base.telegram` | Replies on invalid button | Valid Callback? | None | 6. Button Handling |
| Claim Pending | `n8n-nodes-base.postgres` | Claims pending note atomically | Valid Callback? | Pending Found? | 6. Button Handling |
| Pending Found? | `n8n-nodes-base.if` | Verifies pending item exists | Claim Pending | Action, Reply: Expired | 6. Button Handling |
| Reply: Expired | `n8n-nodes-base.telegram` | Replies when pending token expired | Pending Found? | None | 6. Button Handling |
| Action | `n8n-nodes-base.switch` | Routes button action type | Pending Found? | Update Lead, Create Lead, Reply: Cancelled | 6. Button Handling |
| Update Lead | `n8n-nodes-base.postgres` | Updates lead record in database | Action | Saved? | 7. Save Lead |
| Create Lead | `n8n-nodes-base.postgres` | Creates new lead record in database | Action | Saved? | 7. Save Lead |
| Reply: Cancelled | `n8n-nodes-base.telegram` | Sends cancellation notice | Action | None | 7. Save Lead |
| Saved? | `n8n-nodes-base.if` | Verifies database write success | Update Lead, Create Lead | Format Saved Reply, Reply: Save Failed | 7. Save Lead |
| Format Saved Reply | `n8n-nodes-base.code` | Formats success receipt message | Saved? | Send Saved | 7. Save Lead |
| Send Saved | `n8n-nodes-base.telegram` | Sends success notification | Format Saved Reply | None | 7. Save Lead |
| Reply: Save Failed | `n8n-nodes-base.telegram` | Sends database save failure note | Saved? | None | 7. Save Lead |
| Digest Query | `n8n-nodes-base.postgres` | Queries due/overdue/cold leads | redirect, Route by Kind | Build Digest | 8. Daily Digest |
| Build Digest | `n8n-nodes-base.code` | Formats digest message categories | Digest Query | Send Digest | 8. Daily Digest |
| Send Digest | `n8n-nodes-base.telegram` | Sends digest report | Build Digest | None | 8. Daily Digest |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Triggers and Global Configuration
1. Create a `Telegram Trigger` node. Set parameters to listen for `message` and `callback_query` updates. Configure Telegram credentials.
2. Create a `Schedule Trigger` node. Set interval rules using a cron expression (`30 8 * * *`) for the daily digest.
3. Create a Code node named `Config`. Insert JavaScript code returning the global configuration block (`ALLOWED_CHAT_IDS`, `TIMEZONE`, `COLD_AFTER_DAYS`, `WHISPER_MODEL`, `EXTRACT_MODEL`, `VISION_MODEL`, and `WHISPER_PROMPT`). Leave `ALLOWED_CHAT_IDS` empty initially to capture your chat ID on the first run.
4. Create an IF node named `Is Telegram Update?` checking if `update_id` is present. Connect the true branch to normalization; route the false branch to a No-Op node named `redirect`.
5. Create a Code node named `Normalize Update` to clean incoming message parameters.
6. Create an IF node named `Fraud ?` verifying `authorized === true`. Connect the true branch to a Telegram node named `Reply: Unauthorized`; connect the false branch to a Switch node named `Route by Kind`.
7. Connect the schedule trigger to a PostgreSQL node named `Cleanup Pending` running a query to delete records from `pending_notes` older than 2 days.

#### Step 2: Build the Input Routing & Transcription Pipeline
1. Configure the `Route by Kind` Switch node with output rules for `voice`, `photo`, `text`, `callback`, `cmd_help`, `cmd_today`, and fallback/other.
2. **Voice pipeline:** Connect `Route by Kind` (voice) to a Telegram node (`Get Voice File`, resource: `file`), followed by a Code node (`Fix Audio Filename`), and an HTTP Request node (`Transcribe (Groq Whisper)`) posting multipart data to `https://api.groq.com/openai/v1/audio/transcriptions` with HTTP Header Authentication. Connect the output to a Code node (`Set Voice Text`).
3. **Photo pipeline:** Connect `Route by Kind` (photo) to a Telegram node (`Get Photo File`), a Code node (`Build Vision Request`), and an HTTP Request node (`Read Card (Groq Vision)`) calling `https://api.groq.com/openai/v1/chat/completions` with Header Auth. Connect the output to a Code node (`Set Card Text`).
4. **Text pipeline:** Connect `Route by Kind` (text) to a Code node (`Set Text`).
5. Wire help, unsupported commands, and error catch nodes (`redirect1`, `redirect2`, `Reply: Error`) accordingly.

#### Step 3: Implement AI Lead Extraction
1. Merge the outputs of `Set Voice Text`, `Set Card Text`, and `Set Text` into an IF node named `Has Text?`.
2. Configure the true branch to connect to a Code node named `Build Extraction Request` containing system prompts, schema definitions, and date references.
3. Connect `Build Extraction Request` to an HTTP Request node named `Extract Lead (Groq)` posting JSON payloads to `https://api.groq.com/openai/v1/chat/completions` using Header Authentication and strict JSON schema settings.
4. Connect the output to a Code node named `Parse & Validate` to sanitize phone numbers, dates, and confidence limits.
5. Create an IF node named `Parse OK?`. Route failures to a Telegram node named `Reply: Parse Failed`.

#### Step 4: Set Up Database Matching and Confirmation Cards
1. Connect the true branch of `Parse OK?` to a PostgreSQL node named `Find Matches` executing the pg_trgm similarity SQL query against the `leads` table.
2. Connect `Find Matches` to a Code node named `Build Confirmation` to construct HTML summary strings and filter candidates.
3. Connect `Build Confirmation` to a PostgreSQL node named `Save Pending` inserting proposals into `pending_notes`.
4. Connect `Save Pending` to a Code node named `Prepare Keyboard` to map callback data strings.
5. Connect `Prepare Keyboard` to a Switch node (`Candidate Count`) directing flow to four distinct Telegram action nodes (`Send Confirm (0)` through `Send Confirm (3)`) configured with inline keyboards.

#### Step 5: Implement Button Callbacks and Persistence Logic
1. Connect `Route by Kind` (callback branch) to a Telegram node (`Answer Callback`), a Code node (`Parse Callback`), and an IF node (`Valid Callback?`).
2. Connect the valid branch to a PostgreSQL node named `Claim Pending` executing a `DELETE ... RETURNING payload` query on `pending_notes`.
3. Create an IF node named `Pending Found?`. Route failures to a Telegram node (`Reply: Expired`).
4. Connect the success branch to a Switch node (`Action`) routing updates (`u`), new leads (`n`), and cancellations (`c`).
5. Connect update and create branches to PostgreSQL nodes (`Update Lead`, `Create Lead`) executing custom CTE queries that write to `leads` and `interactions`, utilizing the `compute_lead_score` function.
6. Connect database write outputs to an IF node (`Saved?`), wiring success receipts through a Code node (`Format Saved Reply`) and Telegram node (`Send Saved`), and failures to `Reply: Save Failed`.

#### Step 6: Configure Scheduled and On-Demand Digests
1. Connect `Schedule Trigger` (via redirect) and `Route by Kind` (today command branch) to a PostgreSQL node named `Digest Query`.
2. Connect `Digest Query` to a Code node named `Build Digest` to structure overdue, today, and cold lead buckets.
3. Connect `Build Digest` to a Telegram node (`Send Digest`) to deliver briefings.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Voice Lead Manager Overview & Setup Guide | Designed for sales professionals and freelancers capturing leads via voice notes (English, Hindi, Hinglish), business card photos, or text on Telegram. Requires a Telegram bot token via BotFather, a PostgreSQL database initialized with the required schema and `compute_lead_score` function, and a Groq API key configured via HTTP Header Auth. |