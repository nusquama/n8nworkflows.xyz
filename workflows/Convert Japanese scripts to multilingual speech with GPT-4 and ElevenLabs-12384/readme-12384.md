Convert Japanese scripts to multilingual speech with GPT-4 and ElevenLabs

https://n8nworkflows.xyz/workflows/convert-japanese-scripts-to-multilingual-speech-with-gpt-4-and-elevenlabs-12384


# Convert Japanese scripts to multilingual speech with GPT-4 and ElevenLabs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow takes a Japanese script, translates it into multiple target languages using GPT-4-class models (via n8n LangChain nodes), then generates speech audio for the translated text using the ElevenLabs Text-to-Speech API. It validates the returned audio binary and outputs a standardized “success” or “failed” result.

**Primary use cases:** enterprise localization, multilingual customer communications, content publishing pipelines producing multilingual voiceovers.

### 1.1 Input & Configuration
Collects the Japanese source text, list of target languages, and ElevenLabs synthesis parameters (voice ID, model, stability, similarity boost, API key).

### 1.2 AI Translation Orchestration (Multi-language)
An “orchestrator” agent is instructed to call a dedicated translation tool **for each target language** and return structured results.

### 1.3 Translation Execution (Per-language tool call)
A translation agent tool uses an OpenAI chat model and a structured output parser to produce natural translations plus notes/adaptations.

### 1.4 TTS Request Preparation & Audio Generation
Selects a translation result (currently the **first** translation only), constructs the ElevenLabs request payload, and calls ElevenLabs to produce audio (binary file response).

### 1.5 Audio Validation, Branching & Output Standardization
Validates audio presence and file size bounds, then routes to either a standardized success output (with naming/metadata) or a failure output (with diagnostics).

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Workflow Configuration
**Overview:** Initializes the workflow with source text, languages, and ElevenLabs synthesis parameters. This block is the single entry point and parameter source for later expressions.

**Nodes involved:**
- Manual Trigger
- Workflow Configuration

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual start for testing and ad-hoc runs.
- **Configuration choices:** No parameters.
- **Connections:**
  - **Output →** Workflow Configuration
- **Potential failures:** None (only starts execution).
- **Version notes:** TypeVersion 1.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — defines workflow input variables and defaults.
- **Key fields set:**
  - `japaneseScript` (string placeholder)
  - `targetLanguages` (string: `"English,Spanish,French,German"`)
  - `voiceId` (string placeholder)
  - `elevenLabsApiKey` (string placeholder)
  - `voiceStability` (number: `0.5`)
  - `voiceSimilarityBoost` (number: `0.75`)
  - `modelId` (string: `"eleven_multilingual_v2"`)
- **Notable behavior:** `includeOtherFields: true` (preserves inbound fields).
- **Connections:**
  - **Input ←** Manual Trigger
  - **Output →** Translation Orchestrator Agent
- **Potential failures / edge cases:**
  - Missing placeholders not replaced → downstream translation/TTS will fail.
  - `targetLanguages` is a comma-separated string; orchestration reliability depends on the agent correctly splitting it.
- **Version notes:** TypeVersion 3.4.

**Sticky notes relevant to this block:**
- “## Orchestrator analyzes content and selects translation strategy …”
- “## How It Works …”
- “## Setup Steps …”
- “## Prerequisites …”

---

### Block 2 — AI Translation Orchestration (Multi-language)
**Overview:** Uses an orchestrator agent to coordinate translations by calling a translation tool once per target language. Produces a structured list of translations.

**Nodes involved:**
- Translation Orchestrator Agent
- OpenAI Chat Model - Orchestrator
- Structured Output Parser - Orchestrator
- Translation Agent Tool (as an AI tool connection)

#### Node: Translation Orchestrator Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — central agent that plans and calls tools.
- **Prompting / configuration:**
  - **User text (expression):**
    - `Japanese script: {{ $json.japaneseScript }}`
    - `Target languages: {{ $json.targetLanguages }}`
  - **System message:** instructs orchestrator to call the Translation Agent Tool for each target language, preserve tone/nuance, and return structured output.
  - `promptType: define`
  - `hasOutputParser: true` (expects the structured parser to enforce JSON schema)
- **AI connections:**
  - **Language model:** OpenAI Chat Model - Orchestrator
  - **Output parser:** Structured Output Parser - Orchestrator
  - **Tool:** Translation Agent Tool via `ai_tool` connection
- **Main connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Prepare Translation Request
- **Potential failures / edge cases:**
  - Orchestrator may not correctly iterate all languages if the comma-separated string is interpreted poorly.
  - Output might not match schema → parser failure.
  - Token limits for long scripts or many languages.
- **Version notes:** TypeVersion 3.1.

#### Node: OpenAI Chat Model - Orchestrator
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM for orchestrator.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Credentials: `OpenAi account` (OpenAI API)
- **Connections:**
  - **ai_languageModel →** Translation Orchestrator Agent
- **Potential failures:**
  - Invalid/expired OpenAI credentials, rate limits, model availability.
- **Version notes:** TypeVersion 1.3.

#### Node: Structured Output Parser - Orchestrator
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema for orchestrator output.
- **Schema (interpreted):**
  - Object containing:
    - `translations`: array of `{ language: string, translatedText: string }`
    - `contextNotes`: string
    - `culturalAdaptations`: array of strings
- **Connections:**
  - **ai_outputParser →** Translation Orchestrator Agent
- **Potential failures:**
  - Model returns non-JSON or missing keys → parser error.
- **Version notes:** TypeVersion 1.3.

**Sticky notes relevant to this block:**
- “## Orchestrator analyzes content and selects translation strategy …”
- “## Translation agent converts text with cultural context …” (contextually overlaps the translation logic)
- “## Setup Steps …”
- “## How It Works …”

---

### Block 3 — Translation Execution Tool (Per-language)
**Overview:** Implements the actual translation capability as a tool callable by the orchestrator. It uses a dedicated OpenAI model connection and returns structured translation output including notes.

**Nodes involved:**
- Translation Agent Tool
- OpenAI Chat Model - Translation
- Structured Output Parser - Translation

#### Node: Translation Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool callable by an agent.
- **Input mapping (expression):**
  - `Japanese text: {{ $fromAI("japaneseText") }}`
  - `Target language: {{ $fromAI("targetLanguage") }}`
  - These values are expected to be provided by the orchestrator when it calls the tool.
- **System message:** expert Japanese translator; preserve tone/formality; cultural adaptations; provide context notes; return structured format.
- **Tool description:** “Translates Japanese text to a target language with cultural context awareness”
- **AI connections:**
  - **Language model:** OpenAI Chat Model - Translation
  - **Output parser:** Structured Output Parser - Translation
- **Connections:**
  - Connected to orchestrator via `ai_tool` (the tool is *invoked* by Translation Orchestrator Agent).
- **Potential failures / edge cases:**
  - If orchestrator calls the tool without `japaneseText` / `targetLanguage`, `$fromAI()` resolves empty → poor output or parser failure.
  - Output schema mismatch → parser failure.
- **Version notes:** TypeVersion 3.

#### Node: OpenAI Chat Model - Translation
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — translation LLM.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Credentials: `OpenAi account`
- **Connections:**
  - **ai_languageModel →** Translation Agent Tool
- **Potential failures:** same as other OpenAI model node (auth/rate/model).
- **Version notes:** TypeVersion 1.3.

#### Node: Structured Output Parser - Translation
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces schema for each translation.
- **Schema (interpreted):**
  - Object containing:
    - `language` (string)
    - `translatedText` (string)
    - `contextNotes` (string)
    - `culturalAdaptations` (array of strings)
- **Connections:**
  - **ai_outputParser →** Translation Agent Tool
- **Potential failures:** model output not matching schema.
- **Version notes:** TypeVersion 1.3.

**Sticky notes relevant to this block:**
- “## Translation agent converts text with cultural context …”
- “## Setup Steps …”
- “## How It Works …”

---

### Block 4 — TTS Preparation & ElevenLabs Synthesis
**Overview:** Takes the orchestrator’s structured output, selects the first translation entry, merges in ElevenLabs parameters, and requests audio generation from ElevenLabs (binary output).

**Nodes involved:**
- Prepare Translation Request
- ElevenLabs Text-to-Speech

#### Node: Prepare Translation Request
- **Type / role:** `n8n-nodes-base.set` — prepares a clean payload for TTS.
- **Key expressions / variables:**
  - `translatedText = {{ $json.translations[0].translatedText }}`
  - `language = {{ $json.translations[0].language }}`
  - Pulls configuration from **Workflow Configuration** node:
    - `voiceId = {{ $('Workflow Configuration').first().json.voiceId }}`
    - `elevenLabsApiKey = {{ $('Workflow Configuration').first().json.elevenLabsApiKey }}`
    - `voiceSettings = {{ { "stability": $('Workflow Configuration').first().json.voiceStability, "similarity_boost": $('Workflow Configuration').first().json.voiceSimilarityBoost } }}`
    - `modelId = {{ $('Workflow Configuration').first().json.modelId }}`
- **Connections:**
  - **Input ←** Translation Orchestrator Agent
  - **Output →** ElevenLabs Text-to-Speech
- **Critical edge case:**  
  - This node uses `translations[0]` only. Even if the orchestrator returns multiple translations, only the first language gets synthesized. To synthesize *all* languages, you would need to split the array into items (e.g., Item Lists / Split Out, or a Code node) and loop to ElevenLabs.
- **Version notes:** TypeVersion 3.4.

#### Node: ElevenLabs Text-to-Speech
- **Type / role:** `n8n-nodes-base.httpRequest` — calls ElevenLabs REST API to generate audio.
- **Request configuration:**
  - Method: `POST`
  - URL (expression): `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.voiceId }}`
  - Headers:
    - `xi-api-key: {{ $json.elevenLabsApiKey }}`
    - `Content-Type: application/json`
  - Body (JSON):
    - `text`: from `translatedText`
    - `model_id`: from `modelId` (default `eleven_multilingual_v2`)
    - `voice_settings`: `{ stability, similarity_boost }`
  - Response handling:
    - Response format: **file**
    - Binary property name: `audioData`
- **Connections:**
  - **Input ←** Prepare Translation Request
  - **Output →** Audio Quality Validation
- **Potential failures / edge cases:**
  - 401/403 if API key invalid or subscription lacks access.
  - Invalid `voiceId` → 404 or API error.
  - Payload formatting issues (ensure `translatedText` is serialized as a JSON string; malformed JSON can happen if text isn’t quoted correctly by expression evaluation).
  - Large text length may exceed ElevenLabs limits.
- **Version notes:** TypeVersion 4.3.

**Sticky notes relevant to this block:**
- “## ElevenLabs generates professional audio with optimized parameters …”
- “## Setup Steps …”
- “## Prerequisites …”

---

### Block 5 — Audio Validation, Routing, and Standardized Output
**Overview:** Ensures audio binary exists and is within size limits, then branches into success metadata output or a failure diagnostic payload.

**Nodes involved:**
- Audio Quality Validation
- Check Audio Quality
- Standardize Audio Output
- Handle Quality Failure

#### Node: Audio Quality Validation
- **Type / role:** `n8n-nodes-base.code` — validates binary audio and adds metadata.
- **Logic summary (JS):**
  - Reads `item.binary.audioData`.
  - If missing: returns `{ isValid:false, error:'No audio data received', fileSize:0 }`.
  - Computes byte size from base64: `Buffer.from(audioData.data, 'base64').length`.
  - Valid range:
    - min: `1000` bytes (1KB)
    - max: `50 * 1024 * 1024` bytes (50MB)
  - Returns JSON fields:
    - `isValid`, `fileSize`, `fileSizeKB`, `mimeType`, `fileName`, `error` (if invalid)
  - Passes through binary `audioData`.
- **Connections:**
  - **Input ←** ElevenLabs Text-to-Speech
  - **Output →** Check Audio Quality
- **Potential failures / edge cases:**
  - If ElevenLabs returns non-base64 binary structure, size calc may be wrong.
  - If `audioData.data` is undefined, fileSize becomes 0 and fails validation.
- **Version notes:** TypeVersion 2.

#### Node: Check Audio Quality
- **Type / role:** `n8n-nodes-base.if` — routes based on validation result.
- **Condition:**
  - `{{ $('Audio Quality Validation').item.json.isValid }}` is `true`
  - Uses “loose” type validation.
- **Connections:**
  - **True →** Standardize Audio Output
  - **False →** Handle Quality Failure
- **Potential failures / edge cases:**
  - If validation node output changes shape, expression could fail or resolve null.
- **Version notes:** TypeVersion 2.3.

#### Node: Standardize Audio Output
- **Type / role:** `n8n-nodes-base.set` — formats successful output.
- **Fields set:**
  - `status = "success"`
  - `audioFileName = {{ $json.language }}_{{ $now.toFormat("yyyyMMdd_HHmmss") }}.mp3`
  - `audioSizeKB = {{ $json.fileSizeKB }}`
  - `audioMimeType = {{ $json.mimeType }}`
  - `deliveryFormat = "binary"`
  - `timestamp = {{ $now.toISO() }}`
- **Connections:**
  - **Input ←** Check Audio Quality (true branch)
  - **Output:** end of workflow
- **Edge cases:**
  - `language` must exist in incoming JSON; if missing, filename becomes `_timestamp.mp3`.
- **Version notes:** TypeVersion 3.4.

#### Node: Handle Quality Failure
- **Type / role:** `n8n-nodes-base.set` — formats failure output.
- **Fields set:**
  - `status = "failed"`
  - `errorMessage = {{ $json.error }}`
  - `fileSize = {{ $json.fileSize }}`
  - `timestamp = {{ $now.toISO() }}`
- **Connections:**
  - **Input ←** Check Audio Quality (false branch)
  - **Output:** end of workflow
- **Version notes:** TypeVersion 3.4.

**Sticky notes relevant to this block:**
- “## Quality validator assesses audio against standards and Output …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual entry point | — | Workflow Configuration | ## Orchestrator analyzes content and selects translation strategy\n**Why**: Optimizes approach based on content complexity, domain, and language pairs for superior results |
| Workflow Configuration | n8n-nodes-base.set | Defines source text, target languages, ElevenLabs params | Manual Trigger | Translation Orchestrator Agent | ## Prerequisites\nOpenAI API access with GPT-4 capabilities, active ElevenLabs subscription. |
| Translation Orchestrator Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates per-language translation tool calls; structured output | Workflow Configuration | Prepare Translation Request | ## Orchestrator analyzes content and selects translation strategy\n**Why**: Optimizes approach based on content complexity, domain, and language pairs for superior results |
| Translation Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Tool: translate JA→target language with notes/adaptations | (Called by Orchestrator) | (Returns to Orchestrator) | ## Translation agent converts text with cultural context\n**Why**: Delivers accurate, natural translations appropriate for target audience expectations and regional nuances |
| OpenAI Chat Model - Orchestrator | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for orchestrator | — | Translation Orchestrator Agent (ai_languageModel) | ## Setup Steps\n1. Configure OpenAI API key in "Translation Orchestrator" \n2. Set up ElevenLabs credentials in "Text-to-Speech" \n3. Define source and target languages in "Workflow Configuration" \n4. Customize orchestration logic based on content types and complexity\n5. Set quality thresholds in "Audio Quality Validation" matching output |
| OpenAI Chat Model - Translation | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for translation tool | — | Translation Agent Tool (ai_languageModel) | ## Translation agent converts text with cultural context\n**Why**: Delivers accurate, natural translations appropriate for target audience expectations and regional nuances |
| Structured Output Parser - Orchestrator | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces orchestrator JSON schema | — | Translation Orchestrator Agent (ai_outputParser) | ## Orchestrator analyzes content and selects translation strategy\n**Why**: Optimizes approach based on content complexity, domain, and language pairs for superior results |
| Structured Output Parser - Translation | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces per-translation JSON schema | — | Translation Agent Tool (ai_outputParser) | ## Translation agent converts text with cultural context\n**Why**: Delivers accurate, natural translations appropriate for target audience expectations and regional nuances |
| Prepare Translation Request | n8n-nodes-base.set | Selects translation[0], merges ElevenLabs config | Translation Orchestrator Agent | ElevenLabs Text-to-Speech | ## ElevenLabs generates professional audio with optimized parameters\n**Why**: Creates broadcast-quality speech with proper pronunciation and natural intonation |
| ElevenLabs Text-to-Speech | n8n-nodes-base.httpRequest | Calls ElevenLabs TTS API, returns binary audio | Prepare Translation Request | Audio Quality Validation | ## ElevenLabs generates professional audio with optimized parameters\n**Why**: Creates broadcast-quality speech with proper pronunciation and natural intonation |
| Audio Quality Validation | n8n-nodes-base.code | Validates presence/size of audio binary | ElevenLabs Text-to-Speech | Check Audio Quality | ## Quality validator assesses audio against standards and Output\n**Why**: Ensures output meets publication requirements before delivery, preventing costly quality issues |
| Check Audio Quality | n8n-nodes-base.if | Branches success/failure based on validation | Audio Quality Validation | Standardize Audio Output; Handle Quality Failure | ## Quality validator assesses audio against standards and Output\n**Why**: Ensures output meets publication requirements before delivery, preventing costly quality issues |
| Standardize Audio Output | n8n-nodes-base.set | Success output formatting + filename | Check Audio Quality (true) | — | ## Quality validator assesses audio against standards and Output\n**Why**: Ensures output meets publication requirements before delivery, preventing costly quality issues |
| Handle Quality Failure | n8n-nodes-base.set | Failure output formatting + diagnostics | Check Audio Quality (false) | — | ## Quality validator assesses audio against standards and Output\n**Why**: Ensures output meets publication requirements before delivery, preventing costly quality issues |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | ## Prerequisites\nOpenAI API access with GPT-4 capabilities, active ElevenLabs subscription.\n## Use Cases\nEnterprise content localization, multilingual customer communications\n## Customization\nAdd language-specific translation agents, modify orchestration routing logic\n## Benefits\nDelivers consistent translation quality through intelligent routing |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | ## Setup Steps\n1. Configure OpenAI API key in "Translation Orchestrator" \n2. Set up ElevenLabs credentials in "Text-to-Speech" \n3. Define source and target languages in "Workflow Configuration" \n4. Customize orchestration logic based on content types and complexity\n5. Set quality thresholds in "Audio Quality Validation" matching output  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | ## How It Works\nThis workflow provides enterprise-grade translation and text-to-speech automation for international communication teams, content publishers, and localization services. It addresses producing high-quality multilingual audio content with consistent accuracy and natural delivery at scale. An AI orchestrator analyzes source content to determine optimal translation strategy, selecting specialized agents based on content type, complexity, and target languages. The translation agent processes text with contextual awareness, generating structured output that feeds into ElevenLabs' neural text-to-speech engine. Each audio file undergoes automated quality validation checking pronunciation accuracy, natural flow, and technical specifications. High-quality outputs proceed to standardized formatting for delivery, while failures trigger dedicated error handling with diagnostic reporting, ensuring reliable production of professional multilingual audio assets. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — | ## Quality validator assesses audio against standards and Output\n**Why**: Ensures output meets publication requirements before delivery, preventing costly quality issues |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment | — | — | ## Translation agent converts text with cultural context\n**Why**: Delivers accurate, natural translations appropriate for target audience expectations and regional nuances |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment | — | — | ## Orchestrator analyzes content and selects translation strategy\n**Why**: Optimizes approach based on content complexity, domain, and language pairs for superior results |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment | — | — | ## ElevenLabs generates professional audio with optimized parameters\n**Why**: Creates broadcast-quality speech with proper pronunciation and natural intonation |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Japanese Script to Multilingual Speech Synthesis with AI Translation* (or your preferred name).

2. **Add “Manual Trigger”**
   - Node type: **Manual Trigger**
   - Connect: Manual Trigger → (next step)

3. **Add “Workflow Configuration” (Set node)**
   - Node type: **Set**
   - Add fields:
     - `japaneseScript` (String) — put your Japanese input (or placeholder)
     - `targetLanguages` (String) — e.g. `English,Spanish,French,German`
     - `voiceId` (String) — your ElevenLabs voice ID
     - `elevenLabsApiKey` (String) — your ElevenLabs API key
     - `voiceStability` (Number) — `0.5`
     - `voiceSimilarityBoost` (Number) — `0.75`
     - `modelId` (String) — `eleven_multilingual_v2`
   - Enable **Include Other Fields**.
   - Connect: Manual Trigger → Workflow Configuration → Translation Orchestrator Agent

4. **Add “Translation Orchestrator Agent”**
   - Node type: **AI Agent** (LangChain Agent in n8n)
   - User text:
     - `Japanese script: {{ $json.japaneseScript }}`
     - `Target languages: {{ $json.targetLanguages }}`
   - System message: paste the orchestrator instructions (call tool per language, return structured output).
   - Enable/attach an **Output Parser** (next step).
   - Connect main: Workflow Configuration → Translation Orchestrator Agent

5. **Add “OpenAI Chat Model - Orchestrator”**
   - Node type: **OpenAI Chat Model** (LangChain)
   - Model: `gpt-4.1-mini` (or equivalent)
   - Credentials:
     - Create/choose **OpenAI API** credential in n8n.
   - Connect: OpenAI Chat Model - Orchestrator (ai_languageModel) → Translation Orchestrator Agent

6. **Add “Structured Output Parser - Orchestrator”**
   - Node type: **Structured Output Parser**
   - Schema: object with:
     - `translations[]` items `{ language, translatedText }`
     - `contextNotes` string
     - `culturalAdaptations[]` string
   - Connect: Structured Output Parser - Orchestrator (ai_outputParser) → Translation Orchestrator Agent

7. **Add “Translation Agent Tool”**
   - Node type: **Agent Tool**
   - Tool description: translation tool (as in workflow)
   - Tool input text (expressions):
     - `Japanese text: {{ $fromAI("japaneseText") }}`
     - `Target language: {{ $fromAI("targetLanguage") }}`
   - System message: translator instructions (preserve tone, cultural notes, structured output).
   - Connect the tool to orchestrator:
     - Translation Agent Tool (ai_tool) → Translation Orchestrator Agent

8. **Add “OpenAI Chat Model - Translation”**
   - Node type: **OpenAI Chat Model**
   - Model: `gpt-4.1-mini`
   - Credentials: same OpenAI credential (or another).
   - Connect: OpenAI Chat Model - Translation (ai_languageModel) → Translation Agent Tool

9. **Add “Structured Output Parser - Translation”**
   - Node type: **Structured Output Parser**
   - Schema: object with:
     - `language` string
     - `translatedText` string
     - `contextNotes` string
     - `culturalAdaptations[]` string
   - Connect: Structured Output Parser - Translation (ai_outputParser) → Translation Agent Tool

10. **Add “Prepare Translation Request” (Set node)**
    - Node type: **Set**
    - Add fields (expressions):
      - `translatedText = {{ $json.translations[0].translatedText }}`
      - `language = {{ $json.translations[0].language }}`
      - `voiceId = {{ $('Workflow Configuration').first().json.voiceId }}`
      - `elevenLabsApiKey = {{ $('Workflow Configuration').first().json.elevenLabsApiKey }}`
      - `voiceSettings = {{ { "stability": $('Workflow Configuration').first().json.voiceStability, "similarity_boost": $('Workflow Configuration').first().json.voiceSimilarityBoost } }}`
      - `modelId = {{ $('Workflow Configuration').first().json.modelId }}`
    - Enable **Include Other Fields**.
    - Connect: Translation Orchestrator Agent → Prepare Translation Request → ElevenLabs Text-to-Speech

11. **Add “ElevenLabs Text-to-Speech” (HTTP Request)**
    - Node type: **HTTP Request**
    - Method: `POST`
    - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.voiceId }}`
    - Send headers: enabled
      - `xi-api-key: {{ $json.elevenLabsApiKey }}`
      - `Content-Type: application/json`
    - Send body: JSON
      - Body:
        - `text`: `{{ $json.translatedText }}`
        - `model_id`: `{{ $json.modelId }}`
        - `voice_settings`: `{{ $json.voiceSettings }}`
    - Response: set **Response Format = File**, Output property name `audioData`
    - Connect: ElevenLabs Text-to-Speech → Audio Quality Validation

12. **Add “Audio Quality Validation” (Code node)**
    - Node type: **Code**
    - Mode: **Run once for each item**
    - Paste the validation logic:
      - verify `binary.audioData`
      - compute decoded size
      - validate 1KB–50MB
      - return `json.isValid` and diagnostic fields while preserving `binary.audioData`
    - Connect: Audio Quality Validation → Check Audio Quality

13. **Add “Check Audio Quality” (IF node)**
    - Node type: **IF**
    - Condition: Boolean **true** on
      - `{{ $('Audio Quality Validation').item.json.isValid }}`
    - Connect:
      - True → Standardize Audio Output
      - False → Handle Quality Failure

14. **Add “Standardize Audio Output” (Set node)**
    - Node type: **Set**
    - Fields:
      - `status = "success"`
      - `audioFileName = {{ $json.language }}_{{ $now.toFormat("yyyyMMdd_HHmmss") }}.mp3`
      - `audioSizeKB = {{ $json.fileSizeKB }}`
      - `audioMimeType = {{ $json.mimeType }}`
      - `deliveryFormat = "binary"`
      - `timestamp = {{ $now.toISO() }}`
    - End node (no further connection required).

15. **Add “Handle Quality Failure” (Set node)**
    - Node type: **Set**
    - Fields:
      - `status = "failed"`
      - `errorMessage = {{ $json.error }}`
      - `fileSize = {{ $json.fileSize }}`
      - `timestamp = {{ $now.toISO() }}`
    - End node.

**Credential notes**
- **OpenAI credential:** required by both OpenAI Chat Model nodes.
- **ElevenLabs:** this workflow uses a raw API key in a Set node + HTTP header (not an n8n credential object). For production, consider moving the key to n8n credentials or environment variables to avoid storing secrets in workflow data.

**Sub-workflows:** None (no Execute Workflow node present). The “Translation Agent Tool” is an internal tool node invoked by the orchestrator, not a separate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI API access with GPT-4 capabilities, active ElevenLabs subscription. | Prerequisites (sticky note) |
| Use cases: Enterprise content localization, multilingual customer communications | Prerequisites / Use cases (sticky note) |
| Customization: Add language-specific translation agents, modify orchestration routing logic | Customization (sticky note) |
| Benefits: Delivers consistent translation quality through intelligent routing | Benefits (sticky note) |
| Setup steps include configuring OpenAI, ElevenLabs, languages, orchestration logic, and quality thresholds | Setup Steps (sticky note) |
| “How it works” overview describing orchestrator, translation agent, ElevenLabs TTS, and quality validation | How It Works (sticky note) |

