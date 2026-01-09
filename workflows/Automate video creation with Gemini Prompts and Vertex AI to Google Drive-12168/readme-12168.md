Automate video creation with Gemini Prompts and Vertex AI to Google Drive

https://n8nworkflows.xyz/workflows/automate-video-creation-with-gemini-prompts-and-vertex-ai-to-google-drive-12168


# Automate video creation with Gemini Prompts and Vertex AI to Google Drive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automate video creation with Gemini Prompts and Vertex AI to Google Drive  
**Purpose:** This workflow runs every 2 hours to (1) generate a video concept using Gemini, (2) turn that concept into a Veo 3-compatible prompt, (3) request a long-running Veo 3 video generation on Vertex AI, (4) poll until the video is ready, (5) convert the returned base64 payload into an MP4 binary, (6) upload it to Google Drive, and (7) log metadata/links into Google Sheets.

### Logical blocks
**1.1 Scheduled start & idea generation**  
Schedule trigger → Gemini-backed LLM chain that outputs a structured “video production concept” JSON.

**1.2 Structuring & logging the concept**  
Parse LLM output JSON → append/update a row in a Google Sheet (keyed by `id`).

**1.3 Veo 3 prompt engineering**  
Second LLM chain creates a Veo 3 prompt from the idea + environment → JS cleanup/normalization and derived fields.

**1.4 Vertex AI Veo 3 long-running generation**  
Set project/model/location/token → HTTP request to `predictLongRunning` → wait → polling via `fetchPredictOperation`.

**1.5 Completion loop, file conversion, Drive upload, final logging**  
IF “done?” → if not done wait & retry → if done convert base64 to binary MP4 → upload to Drive → update sheet with Drive link and status.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Scheduled start & idea generation
**Overview:** Starts every 2 hours and asks Gemini to generate a complete video production concept in strict JSON format.

**Nodes involved:**
- **Every 2 Hours1**
- **Google Gemini Chat Model**
- **Basic LLM Chain**

#### Node: Every 2 Hours1
- **Type / role:** Schedule Trigger; workflow entry point.
- **Configuration:** Interval rule: every **2 hours**.
- **Outputs:** Triggers **Basic LLM Chain**.
- **Edge cases:** Missed executions if n8n is down; timezone depends on instance settings.

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain Google Gemini chat model; provides the LLM backend for both LLM Chain nodes.
- **Configuration choices:** Uses **Google PaLM/Gemini API credentials** (`googlePalmApi`). No custom options set.
- **Connections:** Connected via `ai_languageModel` output to:
  - **Basic LLM Chain**
  - **Basic LLM Chain - Prompt**
- **Failure types:**
  - Invalid/expired API key, quota exhaustion, model/API downtime.
  - Content formatting issues (model returns non-JSON despite instruction).

#### Node: Basic LLM Chain
- **Type / role:** LangChain LLM Chain; generates the production concept as JSON.
- **Configuration choices:**
  - Prompt demands **ONLY valid JSON** with fields: `id, idea, caption, production, environment_pro, final_output`.
  - Prompt includes placeholder text `prompt_here` (not dynamically replaced in this workflow).
- **Key expressions/variables:** None (static prompt).
- **Input:** Triggered by schedule; uses Gemini model through `ai_languageModel`.
- **Output:** Expects `json.text` containing the model response.
- **Edge cases / failures:**
  - If Gemini returns markdown fenced code blocks or extra text, downstream JSON parsing can fail (a later Code node tries to strip ```json fences, but not other stray text).
  - `prompt_here` being literal may reduce usefulness unless manually edited or replaced with dynamic content.

---

### Block 1.2 — Structuring & logging the concept
**Overview:** Converts the LLM response string into a JSON object and appends/updates it in Google Sheets using `id` as the unique key.

**Nodes involved:**
- **Code in JavaScript**
- **Append or update row in sheet**

#### Node: Code in JavaScript
- **Type / role:** Code node; cleans and parses LLM output into structured JSON.
- **Configuration choices (interpreted):**
  - Reads `const text = $input.first().json.text;`
  - Removes markdown fences for JSON:
    - removes ```json and ```
  - `JSON.parse(cleanedText)` and returns as item JSON.
- **Inputs:** From **Basic LLM Chain**.
- **Outputs:** To **Append or update row in sheet**.
- **Potential failures:**
  - If the LLM output contains leading/trailing commentary (not fenced) or trailing commas, `JSON.parse` throws.
  - If `json.text` is missing (provider error), node throws.

#### Node: Append or update row in sheet
- **Type / role:** Google Sheets; persists the concept.
- **Operation:** `appendOrUpdate`
- **Matching columns:** `id` (acts as upsert key).
- **Mapped columns:**
  - `id, idea, caption, production, environment_pro, final_output` from current item JSON.
- **Document / sheet:** Spreadsheet “Video Prod Sheet - VEO3”, Sheet1.
- **Credentials:** Google Sheets OAuth2.
- **Outputs:** To **Basic LLM Chain - Prompt**.
- **Edge cases / failures:**
  - Permission/auth issues; sheet not shared with the credential account.
  - Column/schema mismatch: if the sheet columns differ, writes can fail or silently map incorrectly.
  - If `id` is empty/non-unique, updates may overwrite unintended rows or create duplicates depending on n8n behavior.

---

### Block 1.3 — Veo 3 prompt engineering
**Overview:** Uses the stored idea and environment fields to generate a Veo 3-friendly prompt, then cleans it and derives additional parameters (aspect ratio, duration).

**Nodes involved:**
- **Basic LLM Chain - Prompt**
- **Code in JavaScript1**

#### Node: Basic LLM Chain - Prompt
- **Type / role:** LangChain LLM Chain; transforms concept into a Veo 3 prompt.
- **Configuration choices:**
  - Prompt template uses expressions from incoming sheet row:
    - `{{ $json.idea }}`
    - `{{ $json.environment_pro }}`
  - Instruction: “You are a Veo 3 video prompt engineer. Give me a Veo3 prompt…”
- **Input:** From **Append or update row in sheet**.
- **Output:** Produces `json.text` (the prompt).
- **Failures:** Same LLM risks (quota, auth) and prompt quality variability.

#### Node: Code in JavaScript1
- **Type / role:** Code node; prompt cleanup + parameter extraction.
- **Configuration choices (interpreted):**
  - Reads `text = $input.first().json.text`
  - Removes unresolved template variables `{{ ... }}` via regex
  - Collapses whitespace
  - Attempts to read “previous data” from current flow (assumes sheet row data is still in item stream):
    - `const allData = $input.all(); const previousData = allData[0]?.json || {};`
  - Derives:
    - `aspect_ratio` default `9:16`, switches to `16:9` if final_output doesn’t include `9:16`
    - `duration` default `60s`, tries regex `(\d+)-(\d+)\s*seconds?` on `previousData.final_output`
  - Emits:
    - `id`, `veo3_prompt`, `aspect_ratio`, `duration`, `caption`, `timestamp`
- **Input:** From **Basic LLM Chain - Prompt**.
- **Output:** To **Setting**.
- **Critical edge case (data dependency):**
  - This node assumes the item still contains concept fields like `final_output` and `caption`. In practice, the incoming item at this point primarily contains the LLM output (`text`). Unless n8n merges prior fields, `previousData` may *not* include `final_output/caption/id`, causing defaults (and missing caption) to be used.
  - If you need guaranteed access to sheet data, use a Merge node or reference specific nodes via expressions (e.g. `$('Append or update row in sheet').item.json.caption`).

---

### Block 1.4 — Vertex AI Veo 3 long-running generation
**Overview:** Prepares required constants (project/model/location/token), calls Vertex AI’s long-running video generation endpoint, waits, then polls the operation until completed.

**Nodes involved:**
- **Setting**
- **Vertex AI-VEO**
- **Wait**
- **Vertex AI-fetch**

#### Node: Setting
- **Type / role:** Set node; centralizes configuration for Vertex calls.
- **Assignments:**
  - `PROJECT_ID`: `project-cd6b8b94-5c8c-49f8-9cf` (as provided)
  - `MODEL_VERSION`: `veo-3.0-generate-preview`
  - `LOCATION`: `us-central1`
  - `TEXT_PROMPT`: `={{ $json.veo3_prompt }}`
  - `ACCESS_TOKEN`: literal `Your_Access_Token` (must be replaced/refreshed)
  - `API_ENDPOINT`: `us-central1-aiplatform.googleapis.com`
- **Input:** From **Code in JavaScript1**.
- **Output:** To **Vertex AI-VEO**.
- **Edge cases:**
  - Hard-coded access token will expire (sticky note explicitly warns it changes hourly).
  - Incorrect project/location/model leads to 404/permission errors.

#### Node: Vertex AI-VEO
- **Type / role:** HTTP Request; starts long-running prediction (`predictLongRunning`).
- **Method/URL:** `POST https://{{ $json.API_ENDPOINT }}/v1/projects/{{ $json.PROJECT_ID }}/locations/{{ $json.LOCATION }}/publishers/google/models/{{ $json.MODEL_VERSION }}:predictLongRunning`
- **Headers:**
  - `Content-Type: application/json`
  - `Authorization: Bearer {{ $json.ACCESS_TOKEN }}`
- **Body (interpreted):**
  - Sends `instances: [{ prompt: TEXT_PROMPT }]`
  - Sends parameters (hard-coded here):
    - `aspectRatio: "9:16"`
    - `sampleCount: 1`
    - `durationSeconds: "8"` (note: string)
    - `personGeneration: "allow_all"`
    - `addWatermark: true`
    - `includeRaiReason: true`
    - `generateAudio: true`
  - Also includes an `endpoint` string that is **hard-coded** to `projects/n8n-project-440404/...` which may not match `PROJECT_ID`.
- **Output:** To **Wait**.
- **Failure types / integration issues:**
  - **Token expired** → 401.
  - **Mismatch between URL project and body endpoint** could cause unexpected errors or confusion.
  - Vertex response shape must include an operation name (later referenced as `$json.name` by fetch).
  - Duration/aspect ratio from earlier node are not actually used here (parameters are hard-coded).

#### Node: Wait
- **Type / role:** Wait node; delays before polling.
- **Configuration:** Wait **2 minutes**.
- **Input:** From **Vertex AI-VEO**.
- **Output:** To **Vertex AI-fetch**.
- **Edge cases:** Adds latency; may be too short/too long depending on generation time.

#### Node: Vertex AI-fetch
- **Type / role:** HTTP Request; polls the operation status and retrieves results.
- **Method/URL:**  
  `POST https://{{ $('Setting').item.json.API_ENDPOINT }}/v1/projects/{{ $('Setting').item.json.PROJECT_ID }}/locations/{{ $('Setting').item.json.LOCATION }}/publishers/google/models/{{ $('Setting').item.json.MODEL_VERSION }}:fetchPredictOperation`
- **Headers:** Bearer token from **Setting**.
- **Body:** `{ "operationName": "{{ $json.name }}" }`
  - Assumes incoming item has `name` from the long-running operation response.
- **Output:** To **Is Video Complete?1**
- **Failure types:**
  - If the operation response does not expose `name` at top-level, polling fails.
  - If Vertex returns different completion fields than expected, the IF node may never succeed.

---

### Block 1.5 — Completion loop, file conversion, Drive upload, final logging
**Overview:** Checks whether the Vertex operation is finished; if not, waits and retries. If complete, converts base64 to a binary MP4, uploads to Drive, then updates the sheet with the Drive link.

**Nodes involved:**
- **Is Video Complete?1**
- **Wait 1 Min & Retry1**
- **Convert to File**
- **Upload file**
- **Log Final Video in Sheet**

#### Node: Is Video Complete?1
- **Type / role:** IF node; branches on completion.
- **Condition:** `{{ $json.done }}` is boolean `true`.
- **True output:** **Convert to File**
- **False output:** **Wait 1 Min & Retry1**
- **Edge cases:**
  - If Vertex uses a different property (e.g., `done` nested), this will always be false.
  - Strict type validation can fail if `done` is string `"true"` rather than boolean.

#### Node: Wait 1 Min & Retry1
- **Type / role:** Wait node; implements polling loop delay.
- **Configuration:** Wait `amount: 60` (node UI likely interprets as seconds; unit not explicitly set here).
- **Output:** Back to **Vertex AI-fetch**.
- **Edge cases:** Potential infinite loop if operation never sets `done=true` (consider adding max retries).

#### Node: Convert to File
- **Type / role:** ConvertToFile; converts base64 payload to binary file.
- **Operation:** `toBinary`
- **Source property:** `response.videos[0].bytesBase64Encoded`
- **Input:** From IF(true) branch.
- **Output:** To **Upload file**
- **Failure types:**
  - If the fetch response schema differs (no `response.videos[0]...`), conversion fails.
  - Large payloads can hit memory limits.

#### Node: Upload file
- **Type / role:** Google Drive; uploads generated MP4 to a folder.
- **Destination:**
  - Drive: “My Drive”
  - Folder: `Archive` (folder ID `1ymC_Lf5Zi7wf3EsMIEV687PMclAnDlkV`)
- **Filename expression:**
  - `{{ $('Code in JavaScript').item.json.caption.replace(/\s+/g, '_') + '_' + $now.toMillis() }}`
  - Note: references caption from the **first concept JSON parse node** (not the Veo prompt). This works only if that node’s item is available in scope; in n8n it usually is, but can break if multi-item runs diverge.
- **Output:** To **Log Final Video in Sheet**
- **Failure types:**
  - Auth/permissions to folder.
  - Missing binary data (if Convert to File failed).
  - File name invalid characters (caption may contain characters Drive rejects).

#### Node: Log Final Video in Sheet
- **Type / role:** Google Sheets; updates existing row with production status and final Drive link.
- **Operation:** `update`
- **Matching columns:** `idea` (updates row where `idea` matches)
  - This is weaker than matching by `id`; duplicate ideas will cause wrong row updates.
- **Column mappings:**
  - `idea`: from `$('Append or update row in sheet').item.json.idea`
  - `production`: `"done"`
  - `final_output`: Drive `webViewLink`
  - `row_number`: `0` (likely ignored / readOnly; but it’s mapped)
- **Failure types:**
  - If `idea` differs slightly (whitespace changes), update won’t find the row.
  - If multiple rows share same idea, wrong row may be updated.
  - `row_number` is read-only per schema; setting it may be ignored or error depending on connector behavior/version.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every 2 Hours1 | scheduleTrigger | Start workflow on a 2-hour interval | — | Basic LLM Chain | ## 1. Generate ideas for the prompt |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM provider (Gemini) for both chains | — | Basic LLM Chain; Basic LLM Chain - Prompt | ## 1. Generate ideas for the prompt |
| Basic LLM Chain | chainLlm | Generate structured video concept JSON | Every 2 Hours1 (+ Gemini model) | Code in JavaScript | ## 1. Generate ideas for the prompt |
| Code in JavaScript | code | Strip markdown fences + JSON.parse LLM output | Basic LLM Chain | Append or update row in sheet | ## 1. Generate ideas for the prompt |
| Append or update row in sheet | googleSheets | Upsert concept row keyed by `id` | Code in JavaScript | Basic LLM Chain - Prompt | ## 1. Generate ideas for the prompt |
| Basic LLM Chain - Prompt | chainLlm | Generate Veo 3 prompt from idea + environment | Append or update row in sheet (+ Gemini model) | Code in JavaScript1 | ## 2. Generate Prompt for the video |
| Code in JavaScript1 | code | Clean prompt; derive aspect ratio/duration; package payload | Basic LLM Chain - Prompt | Setting | ## 2. Generate Prompt for the video |
| Setting | set | Store project/model/location/token and prompt | Code in JavaScript1 | Vertex AI-VEO | ## Change Access Token every hour or auto refreshes token using a Google cloud  service account |
| Vertex AI-VEO | httpRequest | Start Veo 3 long-running generation | Setting | Wait | ## 3. Generate video, convert to mp4, store and log |
| Wait | wait | Delay before first poll | Vertex AI-VEO | Vertex AI-fetch | ## 3. Generate video, convert to mp4, store and log |
| Vertex AI-fetch | httpRequest | Poll operation status and fetch result | Wait; Wait 1 Min & Retry1 | Is Video Complete?1 | ## 3. Generate video, convert to mp4, store and log |
| Is Video Complete?1 | if | Branch on `$json.done === true` | Vertex AI-fetch | Convert to File (true); Wait 1 Min & Retry1 (false) | ## 3. Generate video, convert to mp4, store and log |
| Wait 1 Min & Retry1 | wait | Polling loop delay | Is Video Complete?1 (false) | Vertex AI-fetch | ## 3. Generate video, convert to mp4, store and log |
| Convert to File | convertToFile | Base64 → binary MP4 | Is Video Complete?1 (true) | Upload file | ## 3. Generate video, convert to mp4, store and log |
| Upload file | googleDrive | Upload MP4 to Drive folder | Convert to File | Log Final Video in Sheet | ## 3. Generate video, convert to mp4, store and log |
| Log Final Video in Sheet | googleSheets | Update sheet row with Drive link and status | Upload file | — | ## 3. Generate video, convert to mp4, store and log |
| Sticky Note | stickyNote | Comment block header | — | — | ## 1. Generate ideas for the prompt |
| Sticky Note1 | stickyNote | Comment block header | — | — | ## 2. Generate Prompt for the video |
| Sticky Note2 | stickyNote | Comment block header | — | — | ## 3. Generate video, convert to mp4, store and log |
| Sticky Note3 | stickyNote | Token rotation warning | — | — | ## Change Access Token every hour or auto refreshes token using a Google cloud  service account |
| Sticky Note4 | stickyNote | Global workflow description + setup links | — | — | ## AI-Powered Video Generation Pipeline with Google Drive Storage… (includes links) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node **Schedule Trigger**.
   2. Set interval to **Every 2 hours**.

2) **Add Gemini model (LangChain)**
   1. Add node **Google Gemini Chat Model**.
   2. Configure credentials: **Gemini/PaLM API key** (n8n credential: `googlePalmApi`).

3) **Generate production concept (LLM chain)**
   1. Add node **Basic LLM Chain** (LangChain → LLM Chain).
   2. In the chain prompt, paste the “Generate a complete video production concept…” instruction requiring **ONLY valid JSON** with fields:
      - `id, idea, caption, production, environment_pro, final_output`
   3. Connect the **Google Gemini Chat Model** to this chain using the chain’s **AI Language Model** input.
   4. Connect **Schedule Trigger → Basic LLM Chain** (main connection).

4) **Parse concept JSON**
   1. Add node **Code** (JavaScript).
   2. Implement:
      - Read `$input.first().json.text`
      - Strip ```json fences
      - `JSON.parse()`
      - Return `{ json: parsedObject }`
   3. Connect **Basic LLM Chain → Code**.

5) **Upsert concept into Google Sheets**
   1. Add node **Google Sheets**.
   2. Credentials: **Google Sheets OAuth2**.
   3. Operation: **Append or update row**.
   4. Select your spreadsheet and sheet (create columns matching the schema):
      - `id, idea, caption, production, environment_pro, final_output`
   5. Set **Matching column** to `id`.
   6. Map each column to the incoming JSON fields.
   7. Connect **Code → Google Sheets**.

6) **Generate Veo 3 prompt (second LLM chain)**
   1. Add node **Basic LLM Chain - Prompt** (LLM Chain).
   2. Prompt should reference incoming fields:
      - Idea: `{{$json.idea}}`
      - Environment: `{{$json.environment_pro}}`
   3. Connect **Google Gemini Chat Model** to this chain via **AI Language Model** input.
   4. Connect **Google Sheets (upsert) → Basic LLM Chain - Prompt**.

7) **Clean prompt and package Veo request data**
   1. Add node **Code** (JavaScript) named similar to **Code in JavaScript1**.
   2. Clean the LLM text; remove unresolved `{{...}}`; normalize whitespace.
   3. Output JSON at least containing:
      - `veo3_prompt`
      - optionally `id`, `caption`, `aspect_ratio`, `duration`, `timestamp`
   4. Connect **Basic LLM Chain - Prompt → Code in JavaScript1**.
   5. (Recommended for correctness) If you need `caption/id/final_output`, reference them explicitly:
      - e.g. `$('Append or update row in sheet').item.json.caption`

8) **Set Vertex AI constants and token**
   1. Add node **Set** named **Setting**.
   2. Add fields:
      - `PROJECT_ID`
      - `MODEL_VERSION` (e.g. `veo-3.0-generate-preview`)
      - `LOCATION` (e.g. `us-central1`)
      - `API_ENDPOINT` (e.g. `us-central1-aiplatform.googleapis.com`)
      - `TEXT_PROMPT` = `{{$json.veo3_prompt}}`
      - `ACCESS_TOKEN` = a valid OAuth2 access token for Google Cloud / Vertex AI
   3. Connect **Code in JavaScript1 → Setting**.
   4. Credential note: this workflow uses a **manual bearer token**. To make it robust, replace with an automated token retrieval (service account JWT exchange or a dedicated auth workflow).

9) **Call Vertex AI Veo long-running endpoint**
   1. Add node **HTTP Request** named **Vertex AI-VEO**.
   2. Method: `POST`
   3. URL:
      - `https://{{$json.API_ENDPOINT}}/v1/projects/{{$json.PROJECT_ID}}/locations/{{$json.LOCATION}}/publishers/google/models/{{$json.MODEL_VERSION}}:predictLongRunning`
   4. Headers:
      - `Content-Type: application/json`
      - `Authorization: Bearer {{$json.ACCESS_TOKEN}}`
   5. JSON body:
      - `instances: [{ prompt: TEXT_PROMPT }]`
      - `parameters`: set duration/aspect ratio as desired
   6. Connect **Setting → Vertex AI-VEO**.

10) **Wait before polling**
   1. Add **Wait** node set to **2 minutes**.
   2. Connect **Vertex AI-VEO → Wait**.

11) **Fetch operation status**
   1. Add **HTTP Request** named **Vertex AI-fetch**.
   2. Method: `POST`
   3. URL:
      - `https://{{ $('Setting').item.json.API_ENDPOINT }}/v1/projects/{{ $('Setting').item.json.PROJECT_ID }}/locations/{{ $('Setting').item.json.LOCATION }}/publishers/google/models/{{ $('Setting').item.json.MODEL_VERSION }}:fetchPredictOperation`
   4. Headers include bearer token from **Setting**.
   5. Body JSON:
      - `{ "operationName": "{{ $json.name }}" }`
   6. Connect **Wait → Vertex AI-fetch**.

12) **Completion check + retry loop**
   1. Add **IF** node “Is Video Complete?”.
   2. Condition: boolean `{{$json.done}}` is true.
   3. Connect **Vertex AI-fetch → IF**.
   4. Add **Wait** node “Wait 1 Min & Retry” set to ~60 seconds.
   5. Connect IF **false → Wait 1 Min & Retry → Vertex AI-fetch** (loop).

13) **Convert base64 to binary MP4**
   1. Add **Convert to File** node.
   2. Operation: **toBinary**
   3. Source property: `response.videos[0].bytesBase64Encoded` (adjust to actual Vertex response)
   4. Connect IF **true → Convert to File**.

14) **Upload to Google Drive**
   1. Add **Google Drive** node “Upload file”.
   2. Credentials: Google Drive OAuth2.
   3. Choose destination folder (e.g. `Archive`).
   4. Name expression: build from caption + timestamp (ensure valid filename).
   5. Connect **Convert to File → Upload file**.

15) **Update Google Sheet with final link**
   1. Add **Google Sheets** node “Log Final Video in Sheet”.
   2. Operation: **Update**.
   3. Set a reliable matching key (recommended: `id`; the provided workflow matches by `idea`).
   4. Set `production = "done"` and `final_output = {{$json.webViewLink}}` from Drive node output.
   5. Connect **Upload file → Log Final Video in Sheet**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI-Powered Video Generation Pipeline with Google Drive Storage (overview of nodes and flow) | Sticky note content (global description) |
| Setup step: enable Vertex AI, obtain ACCESS TOKEN using `gcloud auth print-access-token` | Mentioned in sticky note |
| Enable Google Drive in Google Cloud and set up credential | Mentioned in sticky note |
| Get Gemini API and set up credential | https://aistudio.google.com/ |
| Google Cloud Vertex AI Studio link | https://console.cloud.google.com/vertex-ai/studio/ |
| Google Cloud Console link (Drive enablement mentioned) | https://console.cloud.google.com/ |
| Google Sheet template link | https://docs.google.com/spreadsheets/d/1575_YE8kQk92Xj2DTpx4feCYDXu4hZh8CVl57Un2l2k/edit?usp=sharing |
| ACCESS TOKEN changes hourly; change every hour or auto-refresh using a Google Cloud service account | Sticky Note: “Change Access Token every hour…” |

