Triage Telegram voice and text support with Groq and Google Sheets

https://n8nworkflows.xyz/workflows/triage-telegram-voice-and-text-support-with-groq-and-google-sheets-17685


# Triage Telegram voice and text support with Groq and Google Sheets

### 1. Workflow Overview

This workflow automates customer support triage for a crypto wallet app (**CBox**) via Telegram. It handles incoming voice and text messages, transcribes and classifies them using **Groq LLMs** (Whisper and Llama models), handles automated intent-based answering, manages risk escalations with a deduplication lock, and logs all interactions and system errors to **Google Sheets** with **Slack** alerting.

The logical execution is grouped into the following functional blocks:
- **1.1 Message Intake & Routing:** Receives incoming Telegram updates, validates whether the message is a voice note, text, or unsupported format, and routes accordingly.
- **1.2 Transcription & Context Extraction:** Downloads voice binary files, fixes audio metadata, transcribes audio using Groq Whisper, or extracts text directly, bundling user identification data.
- **1.3 AI Classification & Escalation Evaluation:** Sends the transcript to Groq for strict JSON classification (intent, sentiment, and security risk), checks the Google Sheets “Escalations” log, and enforces a 30-minute escalation lock per user.
- **1.4 Escalation Path:** If locked, replies with a holding message. If high-risk or frustrated, locks the escalation in Google Sheets, notifies a human agent on Telegram, generates a Text-to-Speech (TTS) voice reply using Groq, and logs the interaction.
- **1.5 AI Intent Routing & Automated Reply:** For non-escalated queries, routes by intent (`faq`, `transaction_issue`, `account_kyc`, `wallet_security`, `trading_swap`), generates targeted AI responses via Groq, delivers the reply to the user via Telegram, and logs the interaction to Google Sheets.
- **1.6 Error Handling:** Intercepts failures from critical API nodes, posts alerts to Slack, logs errors to a dedicated Google Sheets tab, and sends a fallback message to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Message Intake & Routing
- **Overview:** Acts as the entry point, capturing incoming Telegram webhooks and verifying the media type to branch execution between voice processing, text processing, or rejection of unsupported formats.
- **Nodes Involved:** 
  - `Voice Message Trigger`
  - `Has Voice?`
  - `Has Text?`
  - `Unsupported Message Reply`
- **Node Details:**
  - **Voice Message Trigger**
    - *Type & Technical Role:* `n8n-nodes-base.telegramTrigger` (Trigger). Listens for incoming updates of type `message`.
    - *Configuration choices:* Subscribes to `message` updates.
    - *Input/Output:* No inputs; outputs to `Has Voice?`.
    - *Edge cases:* Webhook authentication failures or network drops from Telegram.
  - **Has Voice?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Checks if `={{ $json.message.voice }}` exists.
    - *Configuration choices:* Loose type validation; checks for existence of the voice object.
    - *Input/Output:* Input from `Voice Message Trigger`; True output to `Get Voice File`, False output to `Has Text?`.
  - **Has Text?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Checks if `={{ $json.message.text }}` exists.
    - *Configuration choices:* Strict type validation; checks for message text property.
    - *Input/Output:* Input from `Has Voice?` (False branch); True output to `Extract Transcript (Text)`, False output to `Unsupported Message Reply`.
  - **Unsupported Message Reply**
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (Action). Sends a notification that the message type is unsupported.
    - *Configuration choices:* Uses dynamic chat ID `={{ $json.message.chat.id }}` and a predefined text string.
    - *Input/Output:* Input from `Has Text?` (False branch); no downstream connections.

#### 2.2 Transcription & Context Extraction
- **Overview:** Prepares audio files for processing by downloading binaries, reformatting filenames, converting speech to text via Groq Whisper, or extracting raw text strings alongside sender context.
- **Nodes Involved:**
  - `Get Voice File`
  - `Fix Audio Filename`
  - `Transcribe (Whisper)`
  - `Extract Transcript (Voice)`
  - `Extract Transcript (Text)`
- **Node Details:**
  - **Get Voice File**
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (Action). Downloads binary audio from Telegram using `file_id`.
    - *Configuration choices:* Resource: `file`, Parameter `fileId: ={{ $json.message.voice.file_id }}`. Error handling set to continue error output.
    - *Input/Output:* Input from `Has Voice?` (True); Output to `Fix Audio Filename` (success) or `Handle API Error` (error branch).
  - **Fix Audio Filename**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Ensures binary payload contains proper filename (`voice.ogg`) and MIME type (`audio/ogg`) for Whisper processing.
    - *Configuration choices:* Custom JavaScript validating binary keys and remapping to `data`.
    - *Input/Output:* Input from `Get Voice File`; Output to `Transcribe (Whisper)` (success) or `Handle API Error` (error branch).
  - **Transcribe (Whisper)**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Calls Groq’s Whisper API to transcribe audio bytes.
    - *Configuration choices:* POST to `https://api.groq.com/openai/v1/audio/transcriptions` with `multipart/form-data`, attaching binary `data` and model `whisper-large-v3-turbo`. Authenticated via `groqApi`. Retry on fail enabled.
    - *Input/Output:* Input from `Fix Audio Filename`; Output to `Extract Transcript (Voice)` (success) or `Handle API Error` (error branch).
  - **Extract Transcript (Voice)**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Parses Whisper JSON response and extracts user identifiers (chat ID, user ID, first name).
    - *Input/Output:* Input from `Transcribe (Whisper)`; Output to `Classify Intent & Sentiment`.
  - **Extract Transcript (Text)**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Extracts message text and sender metadata for text-based submissions.
    - *Input/Output:* Input from `Has Text?` (True); Output to `Classify Intent & Sentiment`.

#### 2.3 AI Classification & Escalation Evaluation
- **Overview:** Uses a fast Groq Llama model to categorize message intent, sentiment, and security risk into strict JSON. It then checks Google Sheets to enforce a cooldown lock preventing duplicate escalations.
- **Nodes Involved:**
  - `Classify Intent & Sentiment`
  - `Parse Classification JSON`
  - `Get row(s) in sheet`
  - `Evaluate Escalation Lock`
  - `Already Escalated?`
- **Node Details:**
  - **Classify Intent & Sentiment**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Sends the transcript to Groq Chat Completions.
    - *Configuration choices:* POST to `https://api.groq.com/openai/v1/chat/completions` using model `llama-3.1-8b-instant`, `temperature: 0`, and `response_format: { type: "json_object" }`. Authenticated via `groqApi`.
    - *Input/Output:* Input from `Extract Transcript (Voice)` or `Extract Transcript (Text)`; Output to `Parse Classification JSON` (success) or `Handle API Error` (error branch).
  - **Parse Classification JSON**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Cleans markdown blocks from LLM response, parses JSON, and merges it back with upstream user context.
    - *Input/Output:* Input from `Classify Intent & Sentiment`; Output to `Get row(s) in sheet`.
  - **Get row(s) in sheet**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Database Lookup). Queries the "Escalations" sheet tab using the user's `chatId`.
    - *Configuration choices:* Lookup column: `chatId`, lookup value: `={{ $json.chatId }}`. Always outputs data.
    - *Input/Output:* Input from `Parse Classification JSON`; Output to `Evaluate Escalation Lock`.
  - **Evaluate Escalation Lock**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Logic/Data Transformation). Calculates whether an existing escalation occurred within the last 30 minutes.
    - *Configuration choices:* JavaScript evaluating timestamps against `30 * 60 * 1000` milliseconds.
    - *Input/Output:* Input from `Get row(s) in sheet`; Output to `Already Escalated?`.
  - **Already Escalated?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Checks `isLocked` boolean.
    - *Configuration choices:* Strict validation on `={{ $json.isLocked }}`.
    - *Input/Output:* Input from `Evaluate Escalation Lock`; True output to `Holding Reply`, False output to `Escalation Check`.

#### 2.4 Escalation Path
- **Overview:** Manages high-priority user states. If a lock is active, sends a holding text reply. If an escalation is newly triggered (angry/frustrated sentiment or security risk), it updates Google Sheets, alerts human agents, and generates a TTS voice acknowledgment.
- **Nodes Involved:**
  - `Holding Reply`
  - `Escalation Check`
  - `Set Escalation Lock`
  - `Notify Human Agent`
  - `Generate Voice Reply (TTS)`
  - `Send Voice Reply to User`
- **Node Details:**
  - **Holding Reply**
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (Action). Sends a polite holding message when an active escalation lock exists.
    - *Configuration choices:* Text message to `={{ $json.chatId }}`.
    - *Input/Output:* Input from `Already Escalated?` (True); no downstream connections.
  - **Escalation Check**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Evaluates if sentiment equals `angry` or `frustrated`, or if `security_risk` is `true`.
    - *Configuration choices:* Combinator `or`.
    - *Input/Output:* Input from `Already Escalated?` (False); True output to `Set Escalation Lock`, False output to `Route by Intent`.
  - **Set Escalation Lock**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Database Action). Appends or updates the Escalations sheet with the current timestamp.
    - *Configuration choices:* Operation `appendOrUpdate`, matching on `chatId`, updating `escalatedAt` with `={{ $now.toISO() }}`.
    - *Input/Output:* Input from `Escalation Check` (True); Output to `Notify Human Agent`.
  - **Notify Human Agent**
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (Action). Sends an alert to the internal support team channel or agent chat ID.
    - *Configuration choices:* HTML parse mode, structured details including transcript, intent, and risk flags.
    - *Input/Output:* Input from `Set Escalation Lock`; Output to `Generate Voice Reply (TTS)`.
  - **Generate Voice Reply (TTS)**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Calls Groq’s audio generation endpoint to convert support holding text into speech.
    - *Configuration choices:* POST to `https://api.groq.com/openai/v1/audio/speech` using model `canopylabs/orpheus-v1-english`, voice `troy`, WAV output. Authenticated via `groqApi`. Error handling set to continue error output.
    - *Input/Output:* Input from `Notify Human Agent`; Output to `Send Voice Reply to User` (success) or `Handle API Error` (error branch).
  - **Send Voice Reply to User**
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (Action). Sends the generated audio file back to the user via Telegram.
    - *Configuration choices:* Operation `sendAudio`, binary data enabled.
    - *Input/Output:* Input from `Generate Voice Reply (TTS)`; Output to `Log Interaction`.

#### 2.5 AI Intent Routing & Automated Reply
- **Overview:** Routes non-escalated messages based on classified intent to specific Groq Llama models (fast 8b model for FAQs, powerful 70b models for transaction, KYC, security, and trading support).
- **Nodes Involved:**
  - `Route by Intent`
  - `Fast Model Reply (FAQ)`
  - `Contextual Reply (Transaction/KYC)`
  - `Strong Model Reply (Security/Trading)`
  - `Extract Reply Text`
  - `Send Reply to User`
  - `Prepare Log Data`
  - `Log Interaction`
- **Node Details:**
  - **Route by Intent**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Flow Control). Routes execution based on `={{ $json.intent }}`.
    - *Configuration choices:* Rules for `faq`, `transaction_issue` / `account_kyc`, `wallet_security` / `trading_swap`, with a fallback output to `security_trading`.
    - *Input/Output:* Input from `Escalation Check` (False); Outputs route to respective AI generation nodes.
  - **Fast Model Reply (FAQ)**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Generates an FAQ response using `llama-3.1-8b-instant`.
    - *Configuration choices:* Max tokens 200, strict system prompt referencing FAQ data. Authenticated via `groqApi`. Retry on fail enabled.
    - *Input/Output:* Input from `Route by Intent`; Output to `Extract Reply Text` (success) or `Handle API Error` (error branch).
  - **Contextual Reply (Transaction/KYC)**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Generates an account/transaction response using `llama-3.3-70b-versatile`.
    - *Configuration choices:* Max tokens 300, specialized system prompt. Authenticated via `groqApi`. Retry on fail enabled.
    - *Input/Output:* Input from `Route by Intent`; Output to `Extract Reply Text` (success) or `Handle API Error` (error branch).
  - **Strong Model Reply (Security/Trading)**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Generates a security or trading response using `llama-3.3-70b-versatile`.
    - *Configuration choices:* Max tokens 400, high-security system prompt. Authenticated via `groqApi`. Retry on fail enabled.
    - *Input/Output:* Input from `Route by Intent`; Output to `Extract Reply Text` (success) or `Handle API Error` (error branch).
  - **Extract Reply Text**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Pulls the message content from the AI model choices array.
    - *Input/Output:* Input from any of the three AI model reply nodes; Output to `Send Reply to User`.
  - **Send Reply to User**
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (Action). Delivers the textual AI response to the user via Telegram.
    - *Configuration choices:* Uses dynamic text and chat ID.
    - *Input/Output:* Input from `Extract Reply Text`; Output to `Prepare Log Data`.
  - **Prepare Log Data**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Structures the final interaction object (timestamp, chat ID, transcript, intent, sentiment, risk flag, escalated status, and reply text) for spreadsheet logging.
    - *Input/Output:* Input from `Send Reply to User`; Output to `Log Interaction`.
  - **Log Interaction**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Database Action). Appends interaction records to the "Interactions" sheet tab.
    - *Configuration choices:* Operation `append`, mapping document ID and sheet ID `471911948`.
    - *Input/Output:* Input from `Prepare Log Data` (or `Send Voice Reply to User`); no further downstream connections.

#### 2.6 Error Handling
- **Overview:** Centralized error management intercepting failures across critical nodes, formatting error payloads, notifying Slack, logging errors to Google Sheets, and providing a graceful Telegram fallback response.
- **Nodes Involved:**
  - `Handle API Error`
  - `Alert Slack (Error)`
  - `Log Error to Sheet`
  - `Fallback Reply to User`
- **Node Details:**
  - **Handle API Error**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Extracts chat ID, transcript, failed node name, and error message across multiple previous execution scopes.
    - *Input/Output:* Inputs connected via error routing from `Get Voice File`, `Fix Audio Filename`, `Transcribe (Whisper)`, `Classify Intent & Sentiment`, `Fast Model Reply (FAQ)`, `Contextual Reply (Transaction/KYC)`, `Strong Model Reply (Security/Trading)`, and `Generate Voice Reply (TTS)`; Outputs to Slack, Sheets error logging, and fallback messaging simultaneously.
  - **Alert Slack (Error)**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Action). Posts an error alert to a designated Slack channel.
    - *Configuration choices:* Channel ID `C0ATQN2T51T`, OAuth2 authentication, structured markdown message template containing failed node name, chat ID, timestamp, error message, and transcript.
    - *Input/Output:* Input from `Handle API Error`; no downstream connections.
  - **Log Error to Sheet**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Database Action). Appends error details to the "Errors" sheet tab.
    - *Configuration choices:* Operation `append`, sheet ID `1638655937`, mapping `timestamp`, `chatId`, `failedNode`, `errorMessage`, and `transcript`.
    - *Input/Output:* Input from `Handle API Error`; no downstream connections.
  - **Fallback Reply to User**
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (Action). Sends an apology message to the user informing them of processing difficulties.
    - *Configuration choices:* Dynamic chat ID and predefined fallback support message.
    - *Input/Output:* Input from `Handle API Error`; no downstream connections.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Voice Message Trigger | n8n-nodes-base.telegramTrigger | Triggers on incoming Telegram message | None | Has Voice? | 🎙️ CBox Wallet Voice & Text Support Agent <br> 1️⃣ Message Intake & Transcription |
| Has Voice? | n8n-nodes-base.if | Checks if message contains a voice note | Voice Message Trigger | Get Voice File, Has Text? | 1️⃣ Message Intake & Transcription |
| Get Voice File | n8n-nodes-base.telegram | Downloads audio binary file from Telegram | Has Voice? | Fix Audio Filename, Handle API Error | 1️⃣ Message Intake & Transcription |
| Fix Audio Filename | n8n-nodes-base.code | Fixes binary name and MIME type to ogg | Get Voice File | Transcribe (Whisper), Handle API Error | 1️⃣ Message Intake & Transcription |
| Transcribe (Whisper) | n8n-nodes-base.httpRequest | Transcribes audio via Groq Whisper API | Fix Audio Filename | Extract Transcript (Voice), Handle API Error | 1️⃣ Message Intake & Transcription |
| Extract Transcript (Voice) | n8n-nodes-base.code | Extracts transcript and user metadata | Transcribe (Whisper) | Classify Intent & Sentiment | 1️⃣ Message Intake & Transcription |
| Classify Intent & Sentiment | n8n-nodes-base.httpRequest | Classifies intent and sentiment via Groq LLM | Extract Transcript (Voice), Extract Transcript (Text) | Parse Classification JSON, Handle API Error | 2️⃣ Understanding (Intent & Sentiment) |
| Parse Classification JSON | n8n-nodes-base.code | Parses LLM JSON output | Classify Intent & Sentiment | Get row(s) in sheet | 2️⃣ Understanding (Intent & Sentiment) |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Looks up chatId in Escalations sheet | Parse Classification JSON | Evaluate Escalation Lock | 2️⃣ Understanding (Intent & Sentiment) |
| Evaluate Escalation Lock | n8n-nodes-base.code | Evaluates 30-minute escalation lock window | Get row(s) in sheet | Already Escalated? | 2️⃣ Understanding (Intent & Sentiment) |
| Already Escalated? | n8n-nodes-base.if | Checks if escalation is locked | Evaluate Escalation Lock | Holding Reply, Escalation Check | 2️⃣ Understanding (Intent & Sentiment) |
| Holding Reply | n8n-nodes-base.telegram | Sends holding reply to user if locked | Already Escalated? | None | 3️⃣ Escalation & Human Handoff |
| Escalation Check | n8n-nodes-base.if | Checks if sentiment is angry/frustrated or risky | Already Escalated? | Set Escalation Lock, Route by Intent | 3️⃣ Escalation & Human Handoff |
| Set Escalation Lock | n8n-nodes-base.googleSheets | Appends or updates escalation timestamp | Escalation Check | Notify Human Agent | 3️⃣ Escalation & Human Handoff |
| Notify Human Agent | n8n-nodes-base.telegram | Alerts human agent on Telegram | Set Escalation Lock | Generate Voice Reply (TTS) | 3️⃣ Escalation & Human Handoff |
| Generate Voice Reply (TTS) | n8n-nodes-base.httpRequest | Generates TTS voice reply via Groq API | Notify Human Agent | Send Voice Reply to User, Handle API Error | 3️⃣ Escalation & Human Handoff |
| Send Voice Reply to User | n8n-nodes-base.telegram | Sends TTS voice reply to user | Generate Voice Reply (TTS) | Log Interaction | 3️⃣ Escalation & Human Handoff <br> 5️⃣ Reply Delivery & Logging |
| Log Interaction | n8n-nodes-base.googleSheets | Logs final interaction record to Google Sheets | Prepare Log Data, Send Voice Reply to User | None | 5️⃣ Reply Delivery & Logging |
| Handle API Error | n8n-nodes-base.code | Captures failure context for error handling | Get Voice File, Fix Audio Filename, Transcribe (Whisper), Classify Intent & Sentiment, Fast Model Reply (FAQ), Contextual Reply (Transaction/KYC), Strong Model Reply (Security/Trading), Generate Voice Reply (TTS) | Alert Slack (Error), Log Error to Sheet, Fallback Reply to User | ⚠️ Error Handling |
| Alert Slack (Error) | n8n-nodes-base.slack | Sends error notification to Slack channel | Handle API Error | None | ⚠️ Error Handling |
| Log Error to Sheet | n8n-nodes-base.googleSheets | Logs error details to Errors sheet tab | Handle API Error | None | ⚠️ Error Handling |
| Fallback Reply to User | n8n-nodes-base.telegram | Sends fallback support message to user | Handle API Error | None | ⚠️ Error Handling |
| Route by Intent | n8n-nodes-base.switch | Routes non-escalated messages by intent | Escalation Check | Fast Model Reply (FAQ), Contextual Reply (Transaction/KYC), Strong Model Reply (Security/Trading) | 4️⃣ Intent Routing & AI Replies |
| Fast Model Reply (FAQ) | n8n-nodes-base.httpRequest | Generates FAQ response via Groq 8b model | Route by Intent | Extract Reply Text, Handle API Error | 4️⃣ Intent Routing & AI Replies |
| Extract Reply Text | n8n-nodes-base.code | Extracts AI reply text from choices | Fast Model Reply (FAQ), Contextual Reply (Transaction/KYC), Strong Model Reply (Security/Trading) | Send Reply to User | 4️⃣ Intent Routing & AI Replies |
| Send Reply to User | n8n-nodes-base.telegram | Sends AI reply text to user | Extract Reply Text | Prepare Log Data | 5️⃣ Reply Delivery & Logging |
| Prepare Log Data | n8n-nodes-base.code | Prepares interaction log payload | Send Reply to User | Log Interaction | 5️⃣ Reply Delivery & Logging |
| Contextual Reply (Transaction/KYC) | n8n-nodes-base.httpRequest | Generates transaction/KYC reply via Groq 70b | Route by Intent | Extract Reply Text, Handle API Error | 4️⃣ Intent Routing & AI Replies |
| Strong Model Reply (Security/Trading) | n8n-nodes-base.httpRequest | Generates security/trading reply via Groq 70b | Route by Intent | Extract Reply Text, Handle API Error | 4️⃣ Intent Routing & AI Replies |
| Has Text? | n8n-nodes-base.if | Checks if message contains text | Has Voice? | Extract Transcript (Text), Unsupported Message Reply | 1️⃣ Message Intake & Transcription |
| Extract Transcript (Text) | n8n-nodes-base.code | Extracts text transcript and metadata | Has Text? | Classify Intent & Sentiment | 1️⃣ Message Intake & Transcription |
| Unsupported Message Reply | n8n-nodes-base.telegram | Replies that unsupported message type received | Has Text? | None | 1️⃣ Message Intake & Transcription |
| Overview | n8n-nodes-base.stickyNote | 🎙️ CBox Wallet Voice & Text Support Agent <br><br> Turns Telegram voice notes and text messages into a tiered crypto-wallet support agent. Voice is transcribed, then every message is classified for intent, sentiment and security risk. The flow escalates to a human or auto-answers with the right AI model, logging all interactions.<br><br>### How it works<br>1. Voice transcribes via Groq Whisper; text read directly; unknown types get an unsupported reply.<br>2. A Groq LLM classifies intent, sentiment and security risk.<br>3. Angry or risky messages escalate to a human — 30-minute dedupe lock plus voice reply.<br>4. Others receive fast, contextual or strong AI replies, logged to Sheets.<br><br>### Setup<br>- **Telegram** — two bot credentials via BotFather: the customer-facing bot receives messages and sends replies; the internal bot (Telegram account 2) powers Notify Human Agent.<br>- **Groq** — shared `groqApi` key used by all HTTP nodes.<br>- **Google Sheets** — a spreadsheet with Interactions, Errors and Escalations tabs.<br>- **Slack** — an alert channel.<br><br>### Customization<br>Edit the FAQ text in each reply node, tune escalation rules in Escalation Check, and adjust the 30-minute lock window in Evaluate Escalation Lock. | None | None | None |
| Demo Video | n8n-nodes-base.stickyNote | ## 🎥 Demo Video<br><br>Watch a full walkthrough of this workflow on YouTube:<br><br>▶️ **[Watch the demo](https://youtu.be/1KO_LBOGkVQ)**<br><br>https://youtu.be/1KO_LBOGkVQ | None | None | None |
| Section - Intake | n8n-nodes-base.stickyNote | ## 1️⃣ Message Intake & Transcription<br><br>Routes each Telegram message by type:<br>- **Voice** → download, fix filename, transcribe with **Groq Whisper**.<br>- **Text** → use the message text directly.<br>- **Other** → reply that it is unsupported.<br><br>Both paths feed the same classifier. | None | None | None |
| Section - Classify | n8n-nodes-base.stickyNote | ## 2️⃣ Understanding (Intent & Sentiment)<br><br>Extract the transcript + user context, then a Groq LLM classifies **intent**, **sentiment** and **security_risk** into strict JSON, which is parsed for routing. | None | None | None |
| Section - Escalation | n8n-nodes-base.stickyNote | ## 3️⃣ Escalation & Human Handoff<br><br>If sentiment is **angry/frustrated** OR a **security risk** is detected, notify a human agent and send the user a reassuring **voice reply (TTS)** while a specialist follows up. | None | None | None |
| Section - Routing | n8n-nodes-base.stickyNote | ## 4️⃣ Intent Routing & AI Replies<br><br>Non-escalated messages are routed by intent:<br>- **FAQ** → fast model (llama-3.1-8b)<br>- **Transaction / KYC** → contextual model (llama-3.3-70b)<br>- **Security / Trading** → strong model (llama-3.3-70b) | None | None | None |
| Section - Delivery | n8n-nodes-base.stickyNote | ## 5️⃣ Reply Delivery & Logging<br><br>Extract the AI reply text, send it back to the user on Telegram, and append the full interaction to the **Google Sheets** log. | None | None | None |
| Section - Errors | n8n-nodes-base.stickyNote | ## ⚠️ Error Handling<br><br>Any transcription/classification/reply/TTS failure routes here: build an error payload → **alert Slack**, **log to the Errors sheet**, and send the user a friendly **fallback message**. | None | None | None |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the workflow in n8n:

1. **Create the Trigger Node:**
   - Add a **Telegram Trigger** (`Voice Message Trigger`). Set updates to `message`. Connect Telegram Bot credentials.
2. **Set up Intake & Validation:**
   - Add an **If** node (`Has Voice?`). Set condition for `={{ $json.message.voice }}` to exist (loose validation).
   - Add an **If** node (`Has Text?`), connected to the False branch of `Has Voice?`. Set condition for `={{ $json.message.text }}` to exist (strict validation).
   - Add a **Telegram** node (`Unsupported Message Reply`) connected to the False branch of `Has Text?`. Set chat ID to `={{ $json.message.chat.id }}` and provide an unsupported message text.
3. **Build the Voice Pipeline:**
   - Connect the True branch of `Has Voice?` to a **Telegram** node (`Get Voice File`). Set resource to `file` and `fileId` to `={{ $json.message.voice.file_id }}`. Enable "On Error: Continue Error Output".
   - Connect success output to a **Code** node (`Fix Audio Filename`). Insert JS code to assign `voice.ogg` and `audio/ogg` MIME type. Enable "On Error: Continue Error Output".
   - Connect success output to an **HTTP Request** node (`Transcribe (Whisper)`). Set POST method to `https://api.groq.com/openai/v1/audio/transcriptions`. Use `multipart/form-data`, add parameter `file` (Binary Data, input field `data`) and `model` (`whisper-large-v3-turbo`). Authenticate using `groqApi`. Enable retry on fail and "On Error: Continue Error Output".
   - Connect success output to a **Code** node (`Extract Transcript (Voice)`) to extract transcript and sender metadata (`chatId`, `userId`, `userName`).
4. **Build the Text Pipeline:**
   - Connect the True branch of `Has Text?` to a **Code** node (`Extract Transcript (Text)`) to extract text transcript and sender metadata.
5. **Add Classification & Escalation Evaluation:**
   - Connect both `Extract Transcript (Voice)` and `Extract Transcript (Text)` outputs to an **HTTP Request** node (`Classify Intent & Sentiment`). Set POST method to `https://api.groq.com/openai/v1/chat/completions`, model `llama-3.1-8b-instant`, `temperature: 0`, and `response_format` JSON object. Authenticate with `groqApi`. Enable retry on fail and error continuation.
   - Connect success output to a **Code** node (`Parse Classification JSON`) to strip markdown backticks, parse JSON, and merge with metadata.
   - Connect to a **Google Sheets** node (`Get row(s) in sheet`). Set operation to lookup rows, selecting document ID, sheet "Escalations", and lookup value `={{ $json.chatId }}` on column `chatId`. Always output data.
   - Connect to a **Code** node (`Evaluate Escalation Lock`) to calculate whether an escalation occurred within the last 30 minutes (`30 * 60 * 1000` ms).
   - Connect to an **If** node (`Already Escalated?`) checking `={{ $json.isLocked }}` (strict boolean true).
6. **Build Escalation & Hand-off Path:**
   - Connect True output of `Already Escalated?` to a **Telegram** node (`Holding Reply`) sending a holding notice to `={{ $json.chatId }}`.
   - Connect False output of `Already Escalated?` to an **If** node (`Escalation Check`) checking if sentiment equals `angry`, `frustrated`, or `security_risk` is `true`.
   - Connect True output of `Escalation Check` to a **Google Sheets** node (`Set Escalation Lock`). Operation: `appendOrUpdate`, matching columns: `chatId`, updating `escalatedAt` with `={{ $now.toISO() }}`.
   - Connect to a **Telegram** node (`Notify Human Agent`) to alert the internal team (HTML parse mode, dynamic chat ID).
   - Connect to an **HTTP Request** node (`Generate Voice Reply (TTS)`). POST to `https://api.groq.com/openai/v1/audio/speech` using model `canopylabs/orpheus-v1-english`, voice `troy`, and WAV response format. Authenticate with `groqApi`. Enable error continuation.
   - Connect success output to a **Telegram** node (`Send Voice Reply to User`) with operation `sendAudio` and binary data enabled.
   - Connect output to `Log Interaction`.
7. **Build Intent Routing & AI Replies:**
   - Connect False output of `Escalation Check` to a **Switch** node (`Route by Intent`). Define rules for `faq`, `transaction_issue` / `account_kyc`, `wallet_security` / `trading_swap`, with fallback output `security_trading`.
   - Connect FAQ rule to an **HTTP Request** node (`Fast Model Reply (FAQ)`). POST to Groq chat completions using `llama-3.1-8b-instant`, max tokens 200, system prompt with FAQ rules, user content `={{ $('Parse Classification JSON').item.json.transcript }}`. Authenticate with `groqApi`. Enable retry on fail and error continuation.
   - Connect Transaction/KYC rule to an **HTTP Request** node (`Contextual Reply (Transaction/KYC)`). Use model `llama-3.3-70b-versatile`, max tokens 300. Authenticate with `groqApi`. Enable retry on fail and error continuation.
   - Connect Security/Trading rules and fallback output to an **HTTP Request** node (`Strong Model Reply (Security/Trading)`). Use model `llama-3.3-70b-versatile`, max tokens 400. Authenticate with `groqApi`. Enable retry on fail and error continuation.
   - Connect success outputs of all three AI reply nodes to a **Code** node (`Extract Reply Text`) to extract choice content.
   - Connect to a **Telegram** node (`Send Reply to User`) sending `={{ $json.replyText }}` to `={{ $json.chatId }}`.
   - Connect to a **Code** node (`Prepare Log Data`) to build the log object.
   - Connect to a **Google Sheets** node (`Log Interaction`). Operation `append`, sheet "Interactions", mapping all payload fields.
8. **Build Error Handling Pipeline:**
   - Connect error outputs from `Get Voice File`, `Fix Audio Filename`, `Transcribe (Whisper)`, `Classify Intent & Sentiment`, `Fast Model Reply (FAQ)`, `Contextual Reply (Transaction/KYC)`, `Strong Model Reply (Security/Trading)`, and `Generate Voice Reply (TTS)` to a **Code** node (`Handle API Error`).
   - Connect output of `Handle API Error` simultaneously to:
     - A **Slack** node (`Alert Slack (Error)`). Use OAuth2 authentication, select channel `all-soclear-consult`, and map failure details.
     - A **Google Sheets** node (`Log Error to Sheet`). Operation `append`, sheet "Errors", mapping error metadata.
     - A **Telegram** node (`Fallback Reply to User`) sending a polite error apology to `={{ $json.chatId }}`.
9. **Credentials Setup:**
   - Configure **Telegram Bot** credentials for bot interactions and notifications.
   - Configure **Groq API** credential (`groqApi`) for Whisper, Chat Completions, and TTS requests.
   - Configure **Google Sheets OAuth2** credentials referencing spreadsheet document ID `1519n4v45cPvcRwfxHc8kAuVt8ldZTVA3rX-V9jGuSow` with tabs for Interactions, Escalations, and Errors.
   - Configure **Slack OAuth2** credentials for error alerting.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Demo Video Walkthrough | Watch the complete workflow walkthrough on YouTube: [https://youtu.be/1KO_LBOGkVQ](https://youtu.be/1KO_LBOGkVQ) |
| Workflow Template Reference | Template ID: `17522`, Instance ID: `f63656ebba1c247b4cd5eb4513ebf0f3238c944efd076c05ff1db77e94c4c903` |