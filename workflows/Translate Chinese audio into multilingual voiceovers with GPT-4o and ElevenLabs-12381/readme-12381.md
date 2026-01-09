Translate Chinese audio into multilingual voiceovers with GPT-4o and ElevenLabs

https://n8nworkflows.xyz/workflows/translate-chinese-audio-into-multilingual-voiceovers-with-gpt-4o-and-elevenlabs-12381


# Translate Chinese audio into multilingual voiceovers with GPT-4o and ElevenLabs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow receives Chinese text via an HTTP webhook, translates it into multiple target languages using **GPT-4o** with structured output, generates **multilingual voiceovers** via **ElevenLabs Text-to-Speech**, enriches results with QA metrics/metadata, uploads audio to **Google Drive**, aggregates outputs, and produces a final summary/report. If QA fails, it sends a quality alert email via **Gmail**.

**Typical use cases:** multilingual localization for creators/educators, podcast narration in multiple languages, internal communications across global teams.

### 1.1 Ingestion & Configuration
Receives a POST request, normalizes inputs, injects required API/voice settings, and prepares a list of target languages for iteration.

### 1.2 Per-Language Translation (AI) + Structured Parsing
Iterates through languages; GPT-4o translates Chinese content into each language and returns structured JSON with `translatedText` and `targetLanguage`.

### 1.3 Voiceover Generation (ElevenLabs)
For each translation, generates an audio file (binary output) using ElevenLabs multilingual model and retains accompanying metadata.

### 1.4 Quality Assurance & Metadata Enrichment
Computes translation metrics (custom code), checks quality (IF), enriches audio metadata (custom code). If quality fails, triggers an alert email.

### 1.5 Storage, Aggregation, and Reporting
Uploads audio files to Google Drive, aggregates all per-language items into a combined structure, generates summary statistics, and prepares the final report returned to the webhook caller.

**Important consistency note:** Several sticky notes reference “NVIDIA Parakeet TDT” and “NVIDIA API credentials”, but the actual workflow implements translation with **OpenAI GPT-4o** and voice generation with **ElevenLabs** (no NVIDIA node/credential is present).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Ingestion & Triggering
**Overview:** Accepts an HTTP POST payload and initializes workflow fields used downstream (source text, target languages, ElevenLabs settings).

**Nodes involved:**
- Webhook Trigger
- Workflow Configuration
- Split Languages

#### Node: **Webhook Trigger**
- **Type / role:** `Webhook` — entry point to start the workflow from external systems.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: **/translate-audio**
  - Response mode: **Last node** (final node output becomes HTTP response)
- **Key inputs expected:** JSON body containing at least:
  - `chineseContent` (string)
  - `targetLanguages` (string or array-like; see Split node notes)
- **Connections:**
  - Output → **Workflow Configuration**
- **Failure/edge cases:**
  - Missing fields in request body will break later expressions (e.g., `$json.chineseContent`).
  - Large payloads may hit webhook/body limits (depends on n8n deployment).
- **Version notes:** typeVersion **2.1**.

#### Node: **Workflow Configuration**
- **Type / role:** `Set` — normalizes/sets working variables and injects API placeholders.
- **Configuration choices:**
  - Keeps all incoming fields (`includeOtherFields: true`)
  - Sets:
    - `chineseContent` = `{{$json.chineseContent}}`
    - `targetLanguages` = `{{$json.targetLanguages}}`
    - `elevenLabsApiKey` = placeholder value (should be replaced or moved to credentials/env)
    - `voiceId` = placeholder value
- **Connections:**
  - Input ← Webhook Trigger
  - Output → Split Languages
- **Failure/edge cases:**
  - If `targetLanguages` is a comma-separated string, later splitting may not behave as intended unless it is an array or split beforehand.
  - Storing secrets in Set nodes is risky (export logs, sharing workflow); prefer n8n credentials or environment variables.
- **Version notes:** typeVersion **3.4**.

#### Node: **Split Languages**
- **Type / role:** `Split Out` — iterates over each target language producing one item per language.
- **Configuration choices:**
  - Field to split: `targetLanguages`
  - Includes all other fields on each split item
- **Connections:**
  - Input ← Workflow Configuration
  - Output → Translation Agent
- **Failure/edge cases:**
  - `targetLanguages` should be an **array**. If it is a string, the node may split into characters or fail depending on actual input type.
  - Empty array → no downstream execution.
- **Version notes:** typeVersion **1**.

**Sticky note coverage (context):**
- “## Ingestion & Triggering … Webhook receives audio files and configuration …”  
  *Note: actual payload in this workflow is text (`chineseContent`) and languages; no audio ingestion node exists.*

---

### Block 2.2 — AI Translation Generation (GPT-4o + Structured Output)
**Overview:** For each language item, GPT-4o translates the Chinese content into the requested language and outputs a validated structured JSON object.

**Nodes involved:**
- Translation Agent
- OpenAI Chat Model
- Structured Output Parser

#### Node: **OpenAI Chat Model**
- **Type / role:** `LangChain Chat Model (OpenAI)` — provides the LLM used by the agent.
- **Configuration choices:**
  - Model: **gpt-4o**
  - No special options/tools configured.
- **Connections:**
  - Provides model to Translation Agent via `ai_languageModel`.
- **Failure/edge cases:**
  - Invalid/expired OpenAI credentials, quota exhaustion.
  - Model availability/permission issues.
  - Latency/timeouts for long text.
- **Version notes:** typeVersion **1.3**.
- **Credentials:** `openAiApi` (configured as “OpenAi account”).

#### Node: **Structured Output Parser**
- **Type / role:** `LangChain Structured Output Parser` — enforces a JSON schema for the agent’s output.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `translatedText` (string)
    - `targetLanguage` (string)
- **Connections:**
  - Supplies output parser to Translation Agent via `ai_outputParser`.
- **Failure/edge cases:**
  - If the model returns non-conforming JSON, parsing fails and stops the per-language run.
  - Schema currently does not mark fields as “required”; depending on parser behavior, missing fields may still pass or may degrade downstream nodes.
- **Version notes:** typeVersion **1.3**.

#### Node: **Translation Agent**
- **Type / role:** `LangChain Agent` — orchestrates prompt + model + parsing to produce structured translation output.
- **Configuration choices:**
  - Prompt text (per item):
    - Chinese content: `{{$json.chineseContent}}`
    - Target language: `{{$json.targetLanguages}}` (note: field name remains plural even when split)
  - System message: professional Chinese→multilingual translation guidelines; returns JSON format.
  - Output parser enabled.
- **Connections:**
  - Input ← Split Languages
  - `ai_languageModel` ← OpenAI Chat Model
  - `ai_outputParser` ← Structured Output Parser
  - Output → Generate Audio with ElevenLabs
- **Failure/edge cases:**
  - If Split Languages produces `targetLanguages` in unexpected type/value, the agent may translate incorrectly (e.g., to “French,Spanish” as one language).
  - Output parser failure stops the branch.
- **Version notes:** typeVersion **3.1**.

**Sticky note coverage (context):**
- “## AI-Driven Translation Generation … NVIDIA Parakeet TDT …”  
  *Mismatch: Translation is performed by OpenAI GPT-4o, not NVIDIA.*

---

### Block 2.3 — Audio Generation (ElevenLabs TTS)
**Overview:** Converts each translated text into an audio file using ElevenLabs multilingual model and outputs binary audio.

**Nodes involved:**
- Generate Audio with ElevenLabs
- Format Audio Result

#### Node: **Generate Audio with ElevenLabs**
- **Type / role:** `HTTP Request` — calls ElevenLabs Text-to-Speech API.
- **Configuration choices (interpreted):**
  - Method: **POST**
  - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ voiceId }}`
    - `voiceId` pulled from **Workflow Configuration**: `$('Workflow Configuration').item.json.voiceId`
  - Headers:
    - `xi-api-key`: from **Workflow Configuration** `elevenLabsApiKey`
    - `Content-Type: application/json`
  - Body (JSON):
    - `text`: `{{$json.translatedText}}`
    - `model_id`: `eleven_multilingual_v2`
    - `voice_settings`: stability/similarity/style/speaker_boost
  - Response: stored as **file/binary** under output property `audio`
- **Connections:**
  - Input ← Translation Agent
  - Output → Format Audio Result
- **Failure/edge cases:**
  - 401/403 if API key invalid.
  - 404/422 if `voiceId` invalid or body malformed.
  - Request failures for long text (may exceed limits); consider chunking.
  - Expression risk: body uses `{{ $json.translatedText }}` unquoted; if n8n doesn’t stringify properly, JSON can become invalid. Safer pattern is `"text": "={{ $json.translatedText }}"` (string expression).
- **Version notes:** typeVersion **4.3**.

#### Node: **Format Audio Result**
- **Type / role:** `Set` — standardizes per-language output fields and names the audio.
- **Configuration choices:**
  - `language` = `{{$json.targetLanguage}}`
  - `translatedText` = `{{$json.translatedText}}`
  - `audioFileName` = `{{$json.targetLanguage}}_audio.mp3`
  - `hasAudio` = `true`
  - Keeps other fields (including binary `audio`)
- **Connections:**
  - Input ← Generate Audio with ElevenLabs
  - Output → Calculate Translation Metrics
- **Failure/edge cases:**
  - If prior parser did not output `targetLanguage`, filename/language becomes empty.
- **Version notes:** typeVersion **3.4**.

---

### Block 2.4 — Quality Assurance, Metrics, and Branching
**Overview:** Computes custom metrics, checks whether translation meets quality expectations, then either enriches/continues or sends an alert email.

**Nodes involved:**
- Calculate Translation Metrics
- Check Translation Quality
- Enhance Audio Metadata
- Send Quality Alert Email

#### Node: **Calculate Translation Metrics**
- **Type / role:** `Code` — custom computation (implementation not provided in JSON).
- **Configuration choices:** Node exists but has no visible code in the exported JSON payload.
- **Connections:**
  - Input ← Format Audio Result
  - Output → Check Translation Quality
- **Failure/edge cases:**
  - If code is empty, it may output input unchanged (or fail depending on runtime).
  - If code references missing fields, it will throw and stop the item.
- **Version notes:** typeVersion **2**.
- **Action needed to reproduce:** you must implement the JS code (e.g., length ratios, language detection confidence, profanity checks, etc.).

#### Node: **Check Translation Quality**
- **Type / role:** `IF` — routes items based on quality thresholds.
- **Configuration choices:** Conditions are not shown (empty `options`), so actual rules are unknown from this export.
- **Connections:**
  - Input ← Calculate Translation Metrics
  - Output (true branch, index 0) → Enhance Audio Metadata
  - Output (false branch, index 1) → Send Quality Alert Email
- **Failure/edge cases:**
  - Without defined conditions, behavior may default or be incomplete.
  - If it expects computed metrics that are missing, it may route incorrectly.
- **Version notes:** typeVersion **2.3**.

#### Node: **Enhance Audio Metadata**
- **Type / role:** `Code` — enriches item metadata before storage (implementation not provided).
- **Connections:**
  - Input ← Check Translation Quality (pass branch)
  - Output → Upload to Google Drive
- **Failure/edge cases:**
  - Same as other Code node: missing code, missing expected fields, binary mishandling.
- **Version notes:** typeVersion **2**.

#### Node: **Send Quality Alert Email**
- **Type / role:** `Gmail` — notifies when quality checks fail.
- **Configuration choices:** Options exist but no email fields are visible in the export; message composition is unknown.
- **Connections:**
  - Input ← Check Translation Quality (fail branch)
  - No downstream connection (ends this branch).
- **Failure/edge cases:**
  - OAuth token expiry, insufficient Gmail scopes, “From” restrictions.
  - If subject/body/to are not configured, node will fail.
- **Version notes:** typeVersion **2.2**.
- **Credentials:** `gmailOAuth2` (“Gmail account”).

**Sticky note coverage (context):**
- “## Quality Assurance & Validation … OpenAI evaluates translation quality …”  
  *In the current graph, OpenAI is used for translation; there is no dedicated LLM evaluation node. QA appears intended to be implemented inside the Code/IF/Gmail steps.*

---

### Block 2.5 — Storage, Aggregation, and Reporting
**Overview:** Uploads validated audio to Google Drive, aggregates all language outputs into a consolidated structure, computes summary stats, and formats the final report.

**Nodes involved:**
- Upload to Google Drive
- Combine All Results
- Generate Summary Statistics
- Prepare Final Report

#### Node: **Upload to Google Drive**
- **Type / role:** `Google Drive` — stores generated audio assets.
- **Configuration choices:**
  - Drive: “My Drive”
  - Folder: root (`/ (Root folder)`)
  - (No file/binary mapping visible in export; typically you must select binary property `audio` and set file name.)
- **Connections:**
  - Input ← Enhance Audio Metadata
  - Output → Combine All Results
- **Failure/edge cases:**
  - If the node is not configured to use the binary `audio`, upload will fail or upload empty content.
  - OAuth issues, insufficient permissions, shared drive vs My Drive mismatches.
- **Version notes:** typeVersion **3**.
- **Credentials:** `googleDriveOAuth2Api` (“Google Drive account”).

#### Node: **Combine All Results**
- **Type / role:** `Aggregate` — combines per-language items after upload.
- **Configuration choices:**
  - Aggregation mode: **aggregateAllItemData**
  - Destination field name: `language` (this setting is unusual; typically destination is an array field—validate in n8n UI)
- **Connections:**
  - Input ← Upload to Google Drive
  - Output → Generate Summary Statistics
- **Failure/edge cases:**
  - Aggregation configuration may not produce the intended structure (depends on node behavior with `destinationFieldName`).
  - If no items reach this node (all failed QA), aggregation may output empty.
- **Version notes:** typeVersion **1**.

#### Node: **Generate Summary Statistics**
- **Type / role:** `Summarize` — computes summary fields across aggregated data.
- **Configuration choices:**
  - `fieldsToSummarize.values` contains an empty object in export, indicating it may not be fully configured.
- **Connections:**
  - Input ← Combine All Results
  - Output → Prepare Final Report
- **Failure/edge cases:**
  - If not configured, may output minimal/no statistics.
- **Version notes:** typeVersion **1.1**.

#### Node: **Prepare Final Report**
- **Type / role:** `Set` — final shaping of response returned by webhook.
- **Configuration choices:** Not defined in export (empty options), so output format is unknown.
- **Connections:**
  - Input ← Generate Summary Statistics
  - Output: none (last node; webhook responds with its output)
- **Failure/edge cases:**
  - If left unconfigured, it may pass-through or output empty depending on n8n behavior.
- **Version notes:** typeVersion **3.4**.

**Sticky note coverage (context):**
- “## Storage & Asset Management … Upload to Google Drive … metadata-rich filenames …”  
  *Actual upload naming depends on missing configuration/code; you should explicitly map `audioFileName` and binary `audio` in the Drive node.*

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | HTTP entry point | — | Workflow Configuration | ## Ingestion & Triggering<br>**What:** Webhook receives audio files and configuration<br>**Why:** Enables automated triggering from external applications without manual intervention |
| Workflow Configuration | n8n-nodes-base.set | Normalize inputs; inject API settings | Webhook Trigger | Split Languages | ## Ingestion & Triggering<br>**What:** Webhook receives audio files and configuration<br>**Why:** Enables automated triggering from external applications without manual intervention |
| Split Languages | n8n-nodes-base.splitOut | Per-language iteration | Workflow Configuration | Translation Agent | ## Ingestion & Triggering<br>**What:** Webhook receives audio files and configuration<br>**Why:** Enables automated triggering from external applications without manual intervention |
| Translation Agent | @n8n/n8n-nodes-langchain.agent | GPT translation orchestration | Split Languages | Generate Audio with ElevenLabs | ## AI-Driven Translation Generation<br>**What:** NVIDIA Parakeet TDT generates translations with structured output parsing<br>**Why:** Produces accurate translations using state-of-the-art neural models with consistent formatting |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider (GPT-4o) | — (AI connection) | Translation Agent (AI) | ## AI-Driven Translation Generation<br>**What:** NVIDIA Parakeet TDT generates translations with structured output parsing<br>**Why:** Produces accurate translations using state-of-the-art neural models with consistent formatting |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema | — (AI connection) | Translation Agent (AI) | ## AI-Driven Translation Generation<br>**What:** NVIDIA Parakeet TDT generates translations with structured output parsing<br>**Why:** Produces accurate translations using state-of-the-art neural models with consistent formatting |
| Generate Audio with ElevenLabs | n8n-nodes-base.httpRequest | TTS generation (binary audio) | Translation Agent | Format Audio Result | ## Setup Steps<br>1. Configure NVIDIA API credentials in the "Generate Audio with ElevenLabs"<br>2. Add OpenAI API key for quality evaluation in the "OpenAI Chat Model" node<br>3. Set up Google Drive OAuth connection and specify target folder ID for uploads<br>4. Configure Gmail SMTP credentials for notification delivery<br>5. Update webhook URL in source applications to trigger workflow<br>6. Customize target languages in "Split Languages" node if needed |
| Format Audio Result | n8n-nodes-base.set | Standardize output fields | Generate Audio with ElevenLabs | Calculate Translation Metrics |  |
| Calculate Translation Metrics | n8n-nodes-base.code | Compute QA metrics (custom) | Format Audio Result | Check Translation Quality | ## Quality Assurance & Validation<br>**What:** OpenAI evaluates translation quality against defined metrics<br>**Why:** Ensures output meets accuracy standards before delivery |
| Check Translation Quality | n8n-nodes-base.if | Pass/fail gating | Calculate Translation Metrics | Enhance Audio Metadata; Send Quality Alert Email | ## Quality Assurance & Validation<br>**What:** OpenAI evaluates translation quality against defined metrics<br>**Why:** Ensures output meets accuracy standards before delivery |
| Enhance Audio Metadata | n8n-nodes-base.code | Enrich metadata for storage/reporting | Check Translation Quality (pass) | Upload to Google Drive | ## Quality Assurance & Validation<br>**What:** OpenAI evaluates translation quality against defined metrics<br>**Why:** Ensures output meets accuracy standards before delivery |
| Send Quality Alert Email | n8n-nodes-base.gmail | Alert on QA failure | Check Translation Quality (fail) | — | ## Prerequisites<br>Active accounts: NVIDIA (build.nvidia.com), OpenAI, Google Drive, Gmail. API credentials for all services.<br>## Use Cases<br>International podcast distribution, e-learning course localization<br>## Customization<br>Modify target languages in Split node, adjust quality thresholds in OpenAI evaluation<br>## Benefits<br>Reduces translation time by 90%, eliminates manual quality checks through automated validation |
| Upload to Google Drive | n8n-nodes-base.googleDrive | Store audio assets | Enhance Audio Metadata | Combine All Results | ## Storage & Asset Management<br>**What:** Results upload to Google Drive with metadata-rich filenames<br>**Why:** Creates an organized, searchable archive accessible to team members |
| Combine All Results | n8n-nodes-base.aggregate | Aggregate per-language results | Upload to Google Drive | Generate Summary Statistics | ## Storage & Asset Management<br>**What:** Results upload to Google Drive with metadata-rich filenames<br>**Why:** Creates an organized, searchable archive accessible to team members |
| Generate Summary Statistics | n8n-nodes-base.summarize | Cross-language summarization | Combine All Results | Prepare Final Report |  |
| Prepare Final Report | n8n-nodes-base.set | Final response shaping | Generate Summary Statistics | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | ## How It Works<br>This workflow automates end-to-end audio translation with quality assurance... (contains NVIDIA/OpenAI/Drive/email description; partially mismatched to actual nodes) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | ## Setup Steps<br>... (mentions NVIDIA credentials in ElevenLabs node; Gmail SMTP though node is Gmail OAuth) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | ## Prerequisites / Use Cases / Customization / Benefits |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — | ## Quality Assurance & Validation |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment | — | — | ## AI-Driven Translation Generation |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment | — | — | ## Ingestion & Triggering |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment | — | — | ## Storage & Asset Management |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (ensure LangChain nodes are available in your n8n version).
2. **Add Webhook node**
   - Node type: *Webhook*
   - Method: **POST**
   - Path: `translate-audio`
   - Response mode: **Last node**
3. **Add Set node “Workflow Configuration”**
   - Keep other fields: **true**
   - Add fields:
     - `chineseContent` → expression: `{{$json.chineseContent}}`
     - `targetLanguages` → expression: `{{$json.targetLanguages}}`
     - `elevenLabsApiKey` → set from a secure source (preferred):
       - either use an environment variable via expression (e.g., `{{$env.ELEVENLABS_API_KEY}}`) or store in n8n credentials; avoid hardcoding
     - `voiceId` → expression or static value (your ElevenLabs voice ID)
4. **Connect:** Webhook Trigger → Workflow Configuration
5. **Add Split Out node “Split Languages”**
   - Field to split out: `targetLanguages`
   - Include all other fields: enabled
   - **Important:** Ensure webhook provides `targetLanguages` as an array, e.g. `["French","Spanish","Arabic"]`. If you receive a comma-separated string, add an extra Code/Set node before Split to convert it to an array.
6. **Connect:** Workflow Configuration → Split Languages
7. **Add OpenAI Chat Model node**
   - Node type: *OpenAI Chat Model (LangChain)*
   - Model: **gpt-4o**
   - Configure **OpenAI credentials** (API key).
8. **Add Structured Output Parser node**
   - Node type: *Structured Output Parser*
   - Schema (manual): object with `translatedText` (string) and `targetLanguage` (string).
9. **Add Agent node “Translation Agent”**
   - Node type: *AI Agent (LangChain)*
   - Prompt text (expression), for example:
     - `Chinese content to translate: {{ $json.chineseContent }}`
     - `Target language: {{ $json.targetLanguages }}`
   - System message: translator instructions; require JSON output matching schema.
   - Enable output parser.
10. **Connect AI inputs:**
    - OpenAI Chat Model → Translation Agent (AI language model connection)
    - Structured Output Parser → Translation Agent (AI output parser connection)
11. **Connect main flow:** Split Languages → Translation Agent
12. **Add HTTP Request node “Generate Audio with ElevenLabs”**
    - Method: **POST**
    - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $('Workflow Configuration').item.json.voiceId }}`
    - Headers:
      - `xi-api-key`: `={{ $('Workflow Configuration').item.json.elevenLabsApiKey }}`
      - `Content-Type`: `application/json`
    - Body: JSON including:
      - `text`: use a safe string expression (recommended): `"text": "={{ $json.translatedText }}"`
      - `model_id`: `eleven_multilingual_v2`
      - `voice_settings`: set as desired
    - Response: set to **File** and store as binary property name `audio`
13. **Connect:** Translation Agent → Generate Audio with ElevenLabs
14. **Add Set node “Format Audio Result”**
    - Keep other fields: **true**
    - Fields:
      - `language` = `{{$json.targetLanguage}}`
      - `translatedText` = `{{$json.translatedText}}`
      - `audioFileName` = `{{$json.targetLanguage}}_audio.mp3`
      - `hasAudio` = `true`
15. **Connect:** Generate Audio with ElevenLabs → Format Audio Result
16. **Add Code node “Calculate Translation Metrics”**
    - Implement your QA metrics in JS (required; not included in export). Example outputs you may compute:
      - `charCountSource`, `charCountTarget`, ratio thresholds
      - `containsUntranslatedChinese`, `emptyTranslation`, etc.
17. **Connect:** Format Audio Result → Calculate Translation Metrics
18. **Add IF node “Check Translation Quality”**
    - Define conditions based on metrics from the previous Code node (required; not included in export). Example:
      - Fail if `translatedText` is empty
      - Fail if target/source length ratio is outside bounds
19. **Connect:** Calculate Translation Metrics → Check Translation Quality
20. **Add Code node “Enhance Audio Metadata”** (pass branch)
    - Add fields for storage/reporting (required; not included in export), e.g. timestamps, normalized language codes, final filename patterns.
21. **Connect:** Check Translation Quality (true) → Enhance Audio Metadata
22. **Add Google Drive node “Upload to Google Drive”**
    - Operation: upload file (configure in UI)
    - Map:
      - **Binary property:** `audio`
      - **File name:** `audioFileName`
      - Folder: choose target folder (root or a dedicated folder)
    - Configure Google Drive OAuth2 credentials.
23. **Connect:** Enhance Audio Metadata → Upload to Google Drive
24. **Add Gmail node “Send Quality Alert Email”** (fail branch)
    - Configure recipient(s), subject, and body (required; not included in export)
    - Connect Gmail OAuth2 credentials (not SMTP).
25. **Connect:** Check Translation Quality (false) → Send Quality Alert Email
26. **Add Aggregate node “Combine All Results”**
    - Mode: aggregate all item data (ensure it produces a list/collection of results you expect)
27. **Connect:** Upload to Google Drive → Combine All Results
28. **Add Summarize node “Generate Summary Statistics”**
    - Configure summary fields (counts per language, failures, etc.) as needed.
29. **Connect:** Combine All Results → Generate Summary Statistics
30. **Add Set node “Prepare Final Report”**
    - Shape the final JSON returned to webhook caller (e.g., list of languages, Drive links/IDs, summary stats).
31. **Connect:** Generate Summary Statistics → Prepare Final Report
32. **Activate and test**
    - POST to the webhook with JSON similar to:
      - `chineseContent`: `"..."`,
      - `targetLanguages`: `["French","Spanish","Arabic"]`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes describe “audio files via webhook” and “NVIDIA Parakeet TDT translation”, but the implemented workflow uses **text input**, **OpenAI GPT-4o translation**, and **ElevenLabs TTS**. | Align documentation/notes with actual nodes to avoid operator error. |
| “Setup Steps” sticky note mentions “Gmail SMTP credentials”, but the workflow uses the **Gmail OAuth2** node/credential. | Ensure correct auth method/scopes for Gmail node. |
| Several nodes critical to QA/reporting are under-specified in the export: both **Code** nodes, **IF** conditions, Gmail message fields, Drive upload file mapping, Summarize fields, and final report shaping. | These must be explicitly implemented to reproduce exact behavior. |
| “Prerequisites” sticky note mentions NVIDIA account. | Not required unless you actually add NVIDIA translation; current workflow does not. |