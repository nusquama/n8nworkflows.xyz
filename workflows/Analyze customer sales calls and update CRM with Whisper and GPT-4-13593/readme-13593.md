Analyze customer sales calls and update CRM with Whisper and GPT-4

https://n8nworkflows.xyz/workflows/analyze-customer-sales-calls-and-update-crm-with-whisper-and-gpt-4-13593


# Analyze customer sales calls and update CRM with Whisper and GPT-4

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow ingests a sales call recording + metadata, transcribes the audio with OpenAI Whisper, extracts structured “call intelligence” with GPT-4 (via the LangChain Agent node), enriches the analysis with HubSpot contact context, updates HubSpot CRM, emails a call summary to the sales team, writes an audit row to Google Sheets, and returns a structured confirmation to the uploader via webhook response.

**Target use cases:**
- Automating post-call admin (notes, next steps, risk, stage updates)
- Sales coaching (scorecard + objections + talk metrics)
- Compliance/audit trails (central log in Sheets)

### 1.1 Call Upload & Validation
Receives webhook payload (audio URL or binary audio) and validates required fields, formats, and constraints; normalizes metadata into a `callRecord`.

### 1.2 Transcription & Wait Buffer
Fetches the audio from `audioUrl`, transcribes it with Whisper, then waits briefly as a buffer before downstream processing.

### 1.3 GPT-4 Intelligence & CRM Context Enrichment
Structures transcript stats/chunks, fetches HubSpot contact context by email, then runs GPT-4 to produce strict JSON intelligence output and parses it into a unified object.

### 1.4 CRM Update, Summary & Audit
Updates HubSpot contact fields, logs a call activity in HubSpot, emails the intelligence summary, writes an audit row to Google Sheets, and returns a final JSON response to the webhook caller.

---

## 2. Block-by-Block Analysis

### Block 1 — Call Upload & Validation
**Overview:**  
Accepts inbound call upload requests and ensures metadata/audio are usable before spending API quota on transcription and AI analysis.

**Nodes Involved:**
- Sticky Note (global description)
- Sticky Note1 (block label)
- Receive Call Recording Upload
- Validate Call Metadata and Audio

#### Node: Sticky Note
- **Type / Role:** Sticky Note (documentation)
- **Configuration:** Contains end-to-end description, setup steps, sample payload, feature list.
- **Connections:** None (comment only)
- **Failure/Edge cases:** None

#### Node: Sticky Note1
- **Type / Role:** Sticky Note (block label)
- **Configuration:** “## 1. Call Upload & Validation”
- **Connections:** None

#### Node: Receive Call Recording Upload
- **Type / Role:** Webhook trigger (entry point)
- **Key config:**
  - **HTTP Method:** `POST`
  - **Path:** `call-upload`
  - **Response Mode:** `responseNode` (expects a Respond to Webhook node later)
  - **Webhook ID:** `call-analyzer-webhook`
- **Input/Output:**
  - **Output →** Validate Call Metadata and Audio
- **Version-specific:** Webhook node v2
- **Edge cases / failures:**
  - Missing binary file if caller intended file upload but only sent JSON
  - If caller sends multipart, the JSON may be under `body` (handled downstream)
  - If workflow errors before Respond node, caller may not receive a clean response

#### Node: Validate Call Metadata and Audio
- **Type / Role:** Code node (validation + normalization)
- **Key config choices:**
  - Reads payload from `$input.item.json.body || $input.item.json` to support both raw JSON and body-wrapped formats.
  - Enforces required metadata: `callId, repEmail, repName, contactEmail, contactName, companyName`
  - Validates email formats via regex
  - Requires either `audioUrl`/`audio_url` or binary attachment at `$input.item.binary.audio`
  - Rejects calls longer than **2 hours** (`durationSecs > 7200`)
  - Normalizes `dealStage` to lowercase with `_` and validates against an allowlist; otherwise sets `unknown`
  - Generates `processingId` like `PROC-<timestamp>-<rand>`
- **Outputs:**
  - Produces `{ callRecord: { ...normalized fields... } }` with fields like:
    - `callDurationMins`, `receivedAt`, `status: 'VALIDATED'`, `language` default `en`, etc.
- **Input/Output connections:**
  - **Input ←** Receive Call Recording Upload
  - **Output →** Fetch Audio File from URL
- **Edge cases / failures:**
  - Throws hard errors on missing fields/invalid emails/no audio/too-long call (stops workflow)
  - If binary audio is provided but no `audioUrl`, downstream “Fetch Audio File from URL” will receive `null` URL (a logic gap; see notes in reproduction section)
- **Version:** Code node v2

---

### Block 2 — Transcription & Wait Buffer
**Overview:**  
Retrieves the audio file (when `audioUrl` is provided), transcribes it via Whisper, then delays briefly to stabilize the flow.

**Nodes Involved:**
- Sticky Note2 (block label)
- Fetch Audio File from URL
- Transcribe Audio with OpenAI Whisper
- Wait — Transcription Buffer (15 sec)

#### Node: Sticky Note2
- **Type / Role:** Sticky Note (block label)
- **Content:** “## 2. Transcription & Wait Buffer”

#### Node: Fetch Audio File from URL
- **Type / Role:** HTTP Request (download audio as file)
- **Key config:**
  - **URL:** `{{ $json.callRecord.audioUrl }}`
  - **Response format:** file
  - **Timeout:** 30s
  - **Continue on Fail:** enabled
- **Input/Output connections:**
  - **Input ←** Validate Call Metadata and Audio
  - **Output →** Transcribe Audio with OpenAI Whisper
- **Edge cases / failures:**
  - `audioUrl` missing/invalid → request fails; because `continueOnFail=true`, it will pass an error payload forward, likely causing Whisper to fail later
  - 30s timeout may be too short for large files or slow storage endpoints
  - Auth-protected storage URLs (missing signed URL) will fail (403/401)

#### Node: Transcribe Audio with OpenAI Whisper
- **Type / Role:** HTTP Request (OpenAI transcription endpoint)
- **Key config:**
  - **POST** `https://api.openai.com/v1/audio/transcriptions`
  - **Body:** multipart form-data (but note: the JSON does not show the actual multipart fields such as `file`, `model`, `language`; this is critical to configure manually)
  - **Auth:** predefined credential type `openAiApi`
  - Adds header `Authorization: Bearer {{$credentials.apiKey}}`
  - **Timeout:** 120s
  - **Continue on Fail:** enabled
- **Input/Output connections:**
  - **Input ←** Fetch Audio File from URL (expects binary file output)
  - **Output →** Wait — Transcription Buffer (15 sec)
- **Version:** HTTP Request v4.2
- **Edge cases / failures:**
  - Missing required multipart fields (`model=whisper-1` and `file`) will cause 400 errors
  - Large audio may exceed OpenAI limits or take longer than 120s
  - If upstream download failed, Whisper will fail; downstream parser will throw

#### Node: Wait — Transcription Buffer (15 sec)
- **Type / Role:** Wait node (delay)
- **Key config:** waits **15 seconds**
- **Input/Output connections:**
  - **Input ←** Transcribe Audio with OpenAI Whisper
  - **Output →** Parse and Structure Transcript
- **Edge cases / failures:**
  - Not a true async “polling” confirmation; it just delays. If the transcription were async (it isn’t, for this endpoint), this would not guarantee completion.
- **Version:** Wait v1.1

---

### Block 3 — GPT-4 Intelligence & CRM Context Enrichment
**Overview:**  
Transforms Whisper output into transcript stats + chunking, enriches with HubSpot contact history, then prompts GPT-4 to return strict JSON intelligence suitable for CRM updates.

**Nodes Involved:**
- Sticky Note3 (block label)
- Parse and Structure Transcript
- Fetch CRM Contact History
- GPT-4 Call Intelligence Analysis
- GPT-4 Model
- Parse GPT-4 Intelligence Output

#### Node: Sticky Note3
- **Type / Role:** Sticky Note (block label)
- **Content:** “## 3. GPT-4 Intelligence & MCP Enrichment”
- **Note:** The workflow does not actually use MCP tooling; enrichment is via HubSpot API request.

#### Node: Parse and Structure Transcript
- **Type / Role:** Code node (Whisper result validation + metrics + chunking)
- **Key config/logic:**
  - Reads Whisper response from `$('Transcribe Audio with OpenAI Whisper').item.json`
  - Reads call metadata from `$('Validate Call Metadata and Audio').item.json.callRecord`
  - Fails if Whisper response missing or has `.error`
  - Fails if transcript text is empty
  - Calculates:
    - `wordCount`
    - `wordsPerMinute` (using `whisperResponse.duration` or call duration)
    - `estimatedSpeakerTurns` using gaps between segments > 1.5s
  - Chunks transcript to max 12,000 chars; passes **only first chunk** as `transcriptChunk`
- **Input/Output connections:**
  - **Input ←** Wait — Transcription Buffer
  - **Output →** Fetch CRM Contact History
- **Edge cases / failures:**
  - If Whisper returns no `segments`, speaker turn estimation becomes `0` (safe)
  - For long calls, only the first chunk is analyzed by GPT-4; later parts are ignored (may miss objections/next steps)
- **Version:** Code v2

#### Node: Fetch CRM Contact History
- **Type / Role:** HTTP Request (HubSpot contact search)
- **Key config:**
  - **POST** `https://api.hubapi.com/crm/v3/contacts/search`
  - JSON body filters by `email == {{ $json.callRecord.contactEmail }}`
  - Requests a set of properties (email/name/company/stage/notes fields, etc.)
  - Auth: `hubspotApi` predefined credential
  - **Timeout:** 10s
  - **Continue on Fail:** enabled
- **Input/Output connections:**
  - **Input ←** Parse and Structure Transcript
  - **Output →** GPT-4 Call Intelligence Analysis
- **Edge cases / failures:**
  - If auth missing/expired → 401; continues forward but GPT prompt may lack CRM context
  - If no results, later parsing sets `crmContactId` from fallback
  - HubSpot rate limits (429) possible

#### Node: GPT-4 Call Intelligence Analysis
- **Type / Role:** LangChain Agent node (generates structured intelligence)
- **Key config:**
  - Prompt includes call metadata, transcript chunk, and `crmContext` placeholder:
    - Uses `{{ $json.crmContext || 'No prior CRM history...' }}` but **no node sets `$json.crmContext`**. The actual HubSpot response is available in the item JSON, but not summarized into `crmContext` (likely a missing mapping step).
  - Requires **JSON-only** response with a specific schema (sentiment, intent, objections, buying signals, scorecard, CRM updates, follow-up email draft, etc.)
  - System message enforces “valid JSON only”
  - **Model is supplied via the separate “GPT-4 Model” node connection** (`ai_languageModel`)
- **Input/Output connections:**
  - **Main input ←** Fetch CRM Contact History
  - **AI language model input ←** GPT-4 Model
  - **Main output →** Parse GPT-4 Intelligence Output
- **Version:** 1.6
- **Edge cases / failures:**
  - Model may still return non-JSON (extra text, trailing commas) → downstream parser throws
  - Prompt references transcript quotes; if chunk misses key parts, output quality degrades
  - Token limit risk: prompt + transcript chunk may exceed context; model truncation could yield invalid JSON

#### Node: GPT-4 Model
- **Type / Role:** LangChain Chat Model (OpenAI)
- **Key config:**
  - **Model:** `gpt-4o`
  - **Temperature:** 0.2
  - **Max tokens:** 4000 (response budget)
  - Credential: OpenAI API
- **Connections:**
  - Outputs via `ai_languageModel` to GPT-4 Call Intelligence Analysis
- **Edge cases / failures:**
  - If max tokens too low for full JSON + quotes → truncated JSON → parse failure
  - Credential scope / key invalid → agent fails

#### Node: Parse GPT-4 Intelligence Output
- **Type / Role:** Code node (extracts text, strips fences, JSON.parse, merges context)
- **Key logic:**
  - Pulls AI text from multiple possible fields: `response/output/text` or `content[0].text/message.content`
  - Removes ```json fences if present
  - Parses JSON; throws with a 300-char raw preview on failure
  - Pulls:
    - `callData` from `Parse and Structure Transcript`
    - `crmResponse` from `Fetch CRM Contact History`
  - Determines `crmContactId` as:
    - first HubSpot search result id, else `callRecord.crmContactId`, else null
- **Input/Output connections:**
  - **Input ←** GPT-4 Call Intelligence Analysis
  - **Output →** Update CRM Contact and Deal AND Log Call Activity in CRM (parallel)
- **Edge cases / failures:**
  - Any slight JSON invalidity breaks the workflow
  - If HubSpot search failed but continued, `crmResponse` might be an error object; code safely attempts `results?.[0]`
- **Version:** Code v2

---

### Block 4 — CRM Update, Summary & Audit
**Overview:**  
Writes intelligence back to HubSpot (contact update + call activity), sends an email summary, appends an audit row to Google Sheets, and responds to the original webhook request.

**Nodes Involved:**
- Sticky Note4 (block label)
- Update CRM Contact and Deal
- Log Call Activity in CRM
- Send Call Intelligence Summary to Sales Team
- Write to Call Analytics Audit Log
- Build Final Processing Response
- Return Processing Confirmation

#### Node: Sticky Note4
- **Type / Role:** Sticky Note (block label)
- **Content:** “## 4. CRM Update, Summary & Audit”

#### Node: Update CRM Contact and Deal
- **Type / Role:** HTTP Request (HubSpot contact PATCH)
- **Key config:**
  - **PATCH** `https://api.hubapi.com/crm/v3/contacts/{{ $json.crmContactId || 'search' }}`
    - If `crmContactId` is null, it patches `/contacts/search` (invalid endpoint for PATCH) → guaranteed failure.
  - Writes multiple properties, including:
    - `hs_lead_status` set to `recommendedDealStage` (likely wrong semantic mapping; lead status ≠ deal stage)
    - `hubspot_owner_id` set to rep email (HubSpot owner id is usually a numeric ID)
    - Several custom-like fields: `call_sentiment_score`, `last_objection_type`, `buying_signal_count`, `deal_momentum`, etc.
- **Continue on Fail:** enabled
- **Input/Output connections:**
  - **Input ←** Parse GPT-4 Intelligence Output
  - **Output →** Send Call Intelligence Summary to Sales Team
- **Edge cases / failures:**
  - Missing/invalid `crmContactId` breaks update
  - Field names may not exist in the HubSpot portal → 400 validation errors
  - Type mismatches (strings vs numbers) can fail unless HubSpot coerces
  - Using rep email for `hubspot_owner_id` likely fails unless the portal uses email as owner id (unusual)

#### Node: Log Call Activity in CRM
- **Type / Role:** HTTP Request (HubSpot Calls object create)
- **Key config:**
  - **POST** `https://api.hubapi.com/crm/v3/objects/calls`
  - Creates call properties including title/body/duration/direction/disposition/status/timestamp
  - Associates call to contact via association type id `194` (HubSpot-defined)
- **Continue on Fail:** enabled
- **Input/Output connections:**
  - **Input ←** Parse GPT-4 Intelligence Output
  - **Output →** Write to Call Analytics Audit Log
- **Edge cases / failures:**
  - Association type id can vary by HubSpot object; if incorrect → association fails
  - Disposition logic uses sentiment to decide `connected` vs `left_voicemail` (may not match your portal’s allowed values)

#### Node: Send Call Intelligence Summary to Sales Team
- **Type / Role:** Email Send (SMTP)
- **Key config:**
  - Subject includes company name, sentiment, deal risk, callId
  - `toEmail` and `fromEmail` are set to expressions `"="` but **left empty** in the JSON (must be configured)
  - Credential: SMTP
- **Continue on Fail:** enabled
- **Input/Output connections:**
  - **Input ←** Update CRM Contact and Deal
  - **Output →** Build Final Processing Response
- **Edge cases / failures:**
  - Missing To/From leads to send failure
  - SMTP auth/relay restrictions
  - No email body is configured (only subject shown); likely sends blank email unless n8n defaults exist

#### Node: Write to Call Analytics Audit Log
- **Type / Role:** Google Sheets (append row)
- **Key config:**
  - Operation: `append`
  - `documentId` and `sheetName` are unresolved placeholders (`=`); schema mapping not defined
  - Credential: Google Sheets OAuth2
- **Continue on Fail:** enabled
- **Input/Output connections:**
  - **Input ←** Log Call Activity in CRM
  - **Output →** Build Final Processing Response
- **Edge cases / failures:**
  - Missing document/sheet configuration causes runtime failure
  - If you need consistent columns, you must define mapping schema

#### Node: Build Final Processing Response
- **Type / Role:** Code node (constructs webhook response payload)
- **Key output:**
  - Returns `success: true`, call identifiers, transcription stats, intelligence summary, scorecard, flags like `crmUpdated: true`, `summaryEmailSent: true`
  - Note: These flags are always `true` regardless of upstream failures because many nodes use `continueOnFail=true` and this node does not check their status.
- **Input/Output connections:**
  - **Input ←** Write to Call Analytics Audit Log OR Send Call Intelligence Summary to Sales Team (two possible incoming paths)
  - **Output →** Return Processing Confirmation
- **Edge cases / failures:**
  - If it runs from the email path, it may not have audit-log results (but it only uses `$input.item.json`, which still contains merged data from earlier)
  - Success may be reported even if CRM/email/sheets failed

#### Node: Return Processing Confirmation
- **Type / Role:** Respond to Webhook (final HTTP response)
- **Key config:**
  - Respond with JSON body: `JSON.stringify($json, null, 2)`
  - Response headers include:
    - `Content-Type: application/json`
    - `X-Processing-ID: {{ $json.processingId }}`
    - `X-Call-Sentiment: {{ $json.callIntelligence.sentiment }}`
- **Input/Output:**
  - **Input ←** Build Final Processing Response
  - Terminates webhook request
- **Edge cases / failures:**
  - If upstream errors before this node, webhook caller may not get a structured JSON response

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Global workflow description & setup notes |  |  | ## AI Customer Call Analyzer — Voice → Insights → CRM with GPT-4; Converts raw sales call recordings into structured CRM intelligence… (includes setup steps + sample payload + features) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Block label |  |  | ## 1. Call Upload & Validation |
| Sticky Note2 | n8n-nodes-base.stickyNote | Block label |  |  | ## 2. Transcription & Wait Buffer |
| Sticky Note3 | n8n-nodes-base.stickyNote | Block label |  |  | ## 3. GPT-4 Intelligence & MCP Enrichment |
| Sticky Note4 | n8n-nodes-base.stickyNote | Block label |  |  | ## 4. CRM Update, Summary & Audit |
| Receive Call Recording Upload | n8n-nodes-base.webhook | Entry point: receives upload request |  | Validate Call Metadata and Audio | ## 1. Call Upload & Validation |
| Validate Call Metadata and Audio | n8n-nodes-base.code | Validate/normalize metadata; build `callRecord` | Receive Call Recording Upload | Fetch Audio File from URL | ## 1. Call Upload & Validation |
| Fetch Audio File from URL | n8n-nodes-base.httpRequest | Download audio from provided URL | Validate Call Metadata and Audio | Transcribe Audio with OpenAI Whisper | ## 2. Transcription & Wait Buffer |
| Transcribe Audio with OpenAI Whisper | n8n-nodes-base.httpRequest | Call OpenAI Whisper transcription endpoint | Fetch Audio File from URL | Wait — Transcription Buffer (15 sec) | ## 2. Transcription & Wait Buffer |
| Wait — Transcription Buffer (15 sec) | n8n-nodes-base.wait | Delay before parsing transcript | Transcribe Audio with OpenAI Whisper | Parse and Structure Transcript | ## 2. Transcription & Wait Buffer |
| Parse and Structure Transcript | n8n-nodes-base.code | Validate Whisper output; stats; chunk transcript | Wait — Transcription Buffer (15 sec) | Fetch CRM Contact History | ## 3. GPT-4 Intelligence & MCP Enrichment |
| Fetch CRM Contact History | n8n-nodes-base.httpRequest | HubSpot search by contact email for context | Parse and Structure Transcript | GPT-4 Call Intelligence Analysis | ## 3. GPT-4 Intelligence & MCP Enrichment |
| GPT-4 Call Intelligence Analysis | @n8n/n8n-nodes-langchain.agent | GPT analysis prompt → structured JSON intelligence | Fetch CRM Contact History; (ai model) GPT-4 Model | Parse GPT-4 Intelligence Output | ## 3. GPT-4 Intelligence & MCP Enrichment |
| GPT-4 Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides `gpt-4o` model to agent |  | GPT-4 Call Intelligence Analysis | ## 3. GPT-4 Intelligence & MCP Enrichment |
| Parse GPT-4 Intelligence Output | n8n-nodes-base.code | Parse JSON, merge transcript + CRM data, pick contactId | GPT-4 Call Intelligence Analysis | Update CRM Contact and Deal; Log Call Activity in CRM | ## 4. CRM Update, Summary & Audit |
| Update CRM Contact and Deal | n8n-nodes-base.httpRequest | PATCH HubSpot contact properties | Parse GPT-4 Intelligence Output | Send Call Intelligence Summary to Sales Team | ## 4. CRM Update, Summary & Audit |
| Log Call Activity in CRM | n8n-nodes-base.httpRequest | Create HubSpot Calls object and associate to contact | Parse GPT-4 Intelligence Output | Write to Call Analytics Audit Log | ## 4. CRM Update, Summary & Audit |
| Send Call Intelligence Summary to Sales Team | n8n-nodes-base.emailSend | Email summary to rep/manager/team | Update CRM Contact and Deal | Build Final Processing Response | ## 4. CRM Update, Summary & Audit |
| Write to Call Analytics Audit Log | n8n-nodes-base.googleSheets | Append audit row to Google Sheets | Log Call Activity in CRM | Build Final Processing Response | ## 4. CRM Update, Summary & Audit |
| Build Final Processing Response | n8n-nodes-base.code | Build JSON response object for webhook | Send Call Intelligence Summary to Sales Team; Write to Call Analytics Audit Log | Return Processing Confirmation | ## 4. CRM Update, Summary & Audit |
| Return Processing Confirmation | n8n-nodes-base.respondToWebhook | Respond to webhook request with JSON + headers | Build Final Processing Response |  | ## 4. CRM Update, Summary & Audit |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: `AI Customer Call Analyzer — Voice to Insights to CRM with GPT-4`
- Set workflow setting `Execution Order` to `v1` (matches exported settings).

2) **Add Webhook trigger**
- Node: **Webhook** named `Receive Call Recording Upload`
- Method: `POST`
- Path: `call-upload`
- Response mode: **Respond to Webhook node**
- Save node to generate webhook URL for your uploader.

3) **Add validation Code node**
- Node: **Code** named `Validate Call Metadata and Audio`
- Mode: `Run once for each item`
- Paste logic implementing:
  - required fields check
  - email regex validation
  - audio presence check (`audioUrl` or `binary.audio`)
  - duration max 7200s
  - stage normalization
  - `callRecord` assembly with defaults and `processingId`
- Connect: Webhook → Code

4) **Add audio fetch HTTP node (for audioUrl path)**
- Node: **HTTP Request** named `Fetch Audio File from URL`
- URL: `{{ $json.callRecord.audioUrl }}`
- Response: **File**
- Timeout: 30000 ms
- Enable **Continue On Fail** (optional; matches JSON but consider disabling for stricter handling)
- Connect: Validate → Fetch

   **Important:** If you want to support *binary upload* (no URL), add an **IF** node before this step:
   - If `callRecord.audioUrl` exists → Fetch from URL
   - Else → skip fetch and pass binary forward
   (The provided workflow validates binary audio but does not route it correctly.)

5) **Add Whisper transcription HTTP node**
- Node: **HTTP Request** named `Transcribe Audio with OpenAI Whisper`
- Method: `POST`
- URL: `https://api.openai.com/v1/audio/transcriptions`
- Authentication: **OpenAI API** credential (create in n8n Credentials)
- Body type: **Multipart Form-Data**
- Add multipart fields (required by OpenAI):
  - `model`: `whisper-1`
  - `file`: map from the binary produced by “Fetch Audio…” (or from incoming binary)  
    - In n8n, set “Send Binary Data” / “Binary Property” depending on HTTP Request node UI version.
  - Optional: `language`: `{{ $json.callRecord.language }}`
- Timeout: 120000 ms
- Enable **Continue On Fail** (optional; matches JSON)
- Connect: Fetch Audio → Transcribe

6) **Add Wait node**
- Node: **Wait** named `Wait — Transcription Buffer (15 sec)`
- Amount: `15 seconds`
- Connect: Transcribe → Wait

7) **Add transcript parsing Code node**
- Node: **Code** named `Parse and Structure Transcript`
- Implement:
  - validate Whisper response (`.error`, `.text`)
  - compute word count, WPM, speaker-turn heuristic from `.segments`
  - chunk transcript to 12,000 chars and expose `transcriptChunk`
- Connect: Wait → Parse transcript

8) **Add HubSpot contact search**
- Create **HubSpot API** credential in n8n (Private App token or OAuth, depending on your setup).
- Node: **HTTP Request** named `Fetch CRM Contact History`
- Method: `POST`
- URL: `https://api.hubapi.com/crm/v3/contacts/search`
- Auth: HubSpot credential
- Body: JSON with filter on `email == {{ $json.callRecord.contactEmail }}`
- Timeout: 10000 ms
- Continue on Fail: optional
- Connect: Parse transcript → HubSpot search

9) **Add GPT model node**
- Node: **OpenAI Chat Model** named `GPT-4 Model`
- Model: `gpt-4o`
- Temperature: `0.2`
- Max output tokens: `4000`
- Credential: OpenAI API

10) **Add LangChain Agent node**
- Node: **AI Agent** named `GPT-4 Call Intelligence Analysis`
- Configure prompt (user message) that includes:
  - call metadata (from `callRecord`)
  - CRM history summary (either build a summary field first, or embed relevant parts from HubSpot response)
  - transcript chunk (`transcript.transcriptChunk`)
  - strict JSON schema requirement
- System message: enforce “valid JSON only”
- Connect:
  - Main: HubSpot search → Agent
  - AI language model: GPT-4 Model → Agent (AI connection)

11) **Add AI output parsing Code node**
- Node: **Code** named `Parse GPT-4 Intelligence Output`
- Logic:
  - extract AI text from agent output fields
  - strip JSON fences
  - `JSON.parse`
  - merge `callRecord`, `transcript`, parsed `intelligence`
  - derive `crmContactId` from HubSpot `results[0].id` fallback to provided `callRecord.crmContactId`
- Connect: Agent → Parse AI

12) **Add HubSpot contact update**
- Node: **HTTP Request** named `Update CRM Contact and Deal`
- Method: `PATCH`
- URL: `https://api.hubapi.com/crm/v3/contacts/{{ $json.crmContactId }}`
  - (Recommended: add an IF node to stop or branch if `crmContactId` is missing.)
- JSON body: map intelligence fields into HubSpot properties
- Auth: HubSpot credential
- Continue on Fail: optional
- Connect: Parse AI → Update

13) **Add HubSpot call activity create**
- Node: **HTTP Request** named `Log Call Activity in CRM`
- Method: `POST`
- URL: `https://api.hubapi.com/crm/v3/objects/calls`
- JSON body: set call properties + association to contact id
- Auth: HubSpot credential
- Continue on Fail: optional
- Connect: Parse AI → Log Call

14) **Add Email node**
- Create SMTP credential (or Gmail/Outlook nodes if preferred).
- Node: **Send Email** named `Send Call Intelligence Summary to Sales Team`
- Configure:
  - To: set to `{{ $json.callRecord.repEmail }}` and optionally manager email; or a fixed list
  - From: your sending address
  - Subject: similar to exported expression (company, sentiment, risk, call id)
  - Body: include `callSummary`, `nextSteps`, `followUpEmailDraft`, scorecard
- Connect: Update CRM → Email

15) **Add Google Sheets audit log**
- Create Google Sheets OAuth2 credential.
- Node: **Google Sheets** named `Write to Call Analytics Audit Log`
- Operation: **Append**
- Select Document and Sheet
- Define columns mapping (recommended):
  - `callId`, `processingId`, `repEmail`, `companyName`, `sentimentScore`, `dealRisk`, timestamps, etc.
- Connect: Log Call → Sheets

16) **Add final response builder**
- Node: **Code** named `Build Final Processing Response`
- Build a stable response object with:
  - identifiers, stats, summary intelligence
  - (Recommended) include real success flags by inspecting upstream node results instead of hardcoding `true`
- Connect: Email → Build Response
- Also connect: Sheets → Build Response (second input)

17) **Add Respond to Webhook**
- Node: **Respond to Webhook** named `Return Processing Confirmation`
- Respond with: JSON
- Body: `{{ JSON.stringify($json, null, 2) }}`
- Add headers:
  - `X-Processing-ID: {{ $json.processingId }}`
  - `X-Call-Sentiment: {{ $json.callIntelligence.sentiment }}`
- Connect: Build Response → Respond

18) **Credentials checklist**
- OpenAI API credential: used by Whisper HTTP node and GPT-4 Model node
- HubSpot API credential: used by both HubSpot HTTP nodes
- SMTP credential: used by email node
- Google Sheets OAuth2: used by Sheets node

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow description includes setup steps (OpenAI, HubSpot/Salesforce, Google Sheets, SMTP/Gmail), a sample upload payload, and feature list (Whisper transcription, GPT sentiment/intent, objections/signals, scorecard, audit trail). | From the main sticky note content in the workflow |
| The prompt references `crmContext`, but the workflow does not build a `crmContext` field from HubSpot results. Consider adding a Code node to summarize HubSpot search results into a short text block. | Affects GPT-4 Call Intelligence Analysis prompt |
| The validation allows binary audio, but the workflow only fetches audio from URL afterward. Add an IF branch to support true binary upload. | Between “Validate Call Metadata and Audio” and “Fetch Audio File from URL” |
| `PATCH /crm/v3/contacts/{{crmContactId || 'search'}}` is unsafe: if no contact is found, the endpoint becomes invalid. Add a “create contact” path or halt execution when `crmContactId` is missing. | Update CRM Contact and Deal node |
| HubSpot field mappings appear semantically inconsistent (`hs_lead_status` used for deal stage; `hubspot_owner_id` set to email). Adjust to your portal’s correct properties and data types. | Update CRM Contact and Deal node |