Digest and analyse YouTube videos with Apify and Gemini Pro

https://n8nworkflows.xyz/workflows/digest-and-analyse-youtube-videos-with-apify-and-gemini-pro-13675


# Digest and analyse YouTube videos with Apify and Gemini Pro

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Digest and analyse YouTube videos with Apify and Gemini Pro — Workflow Reference (n8n)

Workflow name in JSON: **Video Digestion Workflow**  
Stated title: **Digest and analyse YouTube videos with Apify and Gemini Pro**

---

## 1. Workflow Overview

This workflow ingests a **YouTube video URL** via an n8n **Form Trigger**, uses **Apify** to (1) produce a downloadable MP4 link and (2) fetch the **YouTube transcript + metadata**, then performs **video-level visual analysis** using **Google Gemini (video analyze)** to extract many short “B-roll” clips with timestamps and cropping metadata (webcam overlay detection). The Gemini output is then parsed, validated, filtered to keep only usable screen content clips, and combined with transcript + metadata. Next, an **OpenAI (LangChain) node** generates structured JSON metadata for repurposing/SEO. Finally, the workflow **calls a separate sub-workflow** (“Short Creation (HUMAN)”) asynchronously to proceed with shorts creation.

### 1.1 Input Reception (Stage 1)
- Collect the YouTube URL from a form submission.

### 1.2 Video Acquisition via Apify (Stage 1 continuation)
- Run an Apify actor to download the video and get a direct MP4 URL.
- Run an Apify actor to obtain transcript and full video metadata.

### 1.3 Consolidation / Normalization
- A Code node merges the Form + Apify outputs into a single normalized “video data” object (URLs, IDs, transcript text, segments, metadata).

### 1.4 Visual Analysis (Stage 2B)
- Gemini analyzes the entire video and outputs many 3-second clips with strict schema including webcam overlay detection and cropping instructions.

### 1.5 Clip Parsing, Filtering, and Categorization
- A Code node parses Gemini’s JSON response, validates, categorizes clips, filters out unusable clips, and outputs structured `key_moments` plus analysis statistics.

### 1.6 Transcript + Visual Context Metadata (AI analysis)
- OpenAI node generates a structured JSON overview for the video (summary, audience, takeaways, SEO keywords, suggested titles, etc.) using transcript + key moments.

### 1.7 Prepare Payload & Trigger Downstream Sub-workflow
- A Set node assembles final fields.
- Execute Workflow node triggers “Short Creation (HUMAN)” asynchronously.

---

## 2. Block-by-Block Analysis

### Block 1 — Documentation / Operator Notes
**Overview:** Non-executing sticky notes explaining stages and providing links/context.  
**Nodes involved:**  
- Setup Instructions (Sticky Note)  
- Stage 1 (Sticky Note)  
- Stage 2A - Transcript (Sticky Note)  
- Stage 2B - Visual1 (Sticky Note)  
- Sticky Note (tutorial link)

**Node details (Sticky Notes)**
- **Type/role:** `n8n-nodes-base.stickyNote` — visual documentation only; no runtime effect.
- **Failure modes:** None (not executed).

---

### Block 2 — Stage 1: YouTube URL Input
**Overview:** Receives the YouTube URL from a user through an n8n hosted form trigger.  
**Nodes involved:**  
- YouTube URL Form

#### Node: YouTube URL Form
- **Type:** `n8n-nodes-base.formTrigger` (typeVersion 2.1)
- **Technical role:** Entry point. Creates a webhook endpoint and emits submitted fields as JSON.
- **Configuration choices:**
  - Path/Webhook ID: `youtube-form-trigger` (path is the same)
  - Form title: “YouTube Video Processor”
  - Field: “YouTube URL” (required)
  - Description: “Submit a YouTube video URL for automated shorts creation”
- **Key variables/fields:**
  - Output field name: `$json['YouTube URL']`
- **Connections:**
  - **Out →** Apify YouTube Downloader
- **Potential failures / edge cases:**
  - Invalid YouTube URL format (not validated here; downstream may fail).
  - If form path conflicts with another webhook path, trigger won’t register.
  - If running behind reverse proxy, webhook URL configuration must be correct.

---

### Block 3 — Stage 1: Apify Video Download + Transcript Extraction
**Overview:** Uses Apify actors to produce a downloadable MP4 URL and fetch transcript + metadata.  
**Nodes involved:**  
- Apify YouTube Downloader  
- Apify YouTube Transcript

#### Node: Apify YouTube Downloader
- **Type:** `@apify/n8n-nodes-apify.apify` (typeVersion 1)
- **Technical role:** Runs Apify actor **epctex/youtube-video-downloader** and returns dataset items.
- **Configuration choices (interpreted):**
  - Operation: “Run actor and get dataset”
  - Actor: “Youtube Video Downloader (epctex/youtube-video-downloader)”
  - Input body highlights:
    - `startUrls`: `[ "{{ $json['YouTube URL'] }}" ]` (from the Form Trigger output)
    - Proxy: Apify proxy enabled, group `RESIDENTIAL`
    - `quality`: `"720"`
    - `maxRequestRetries`: `2`
    - `includeFailedVideos`: `false`
    - `useFfmpeg`: `false`
- **Credentials:** Apify API credential (“Adams Apify Account”)
- **Connections:**
  - **In ←** YouTube URL Form
  - **Out →** Apify YouTube Transcript
- **Potential failures / edge cases:**
  - Apify auth issues (invalid token, workspace access).
  - Actor run failures due to YouTube restrictions, region locks, rate limits.
  - Dataset empty (no `downloadUrl` returned); will cause downstream Gemini video analysis to fail.
  - Proxy group availability/quotas.
  - Video not available at 720p.

#### Node: Apify YouTube Transcript
- **Type:** `@apify/n8n-nodes-apify.apify` (typeVersion 1)
- **Technical role:** Runs Apify actor **starvibe/youtube-video-transcript** and returns transcript segments + metadata.
- **Configuration choices:**
  - Operation: “Run actor and get dataset”
  - Actor: “YouTube Video Transcript (starvibe/youtube-video-transcript)”
  - Input body highlights:
    - `youtube_url`: `{{ $('YouTube URL Form').item.json['YouTube URL'] }}`
    - `language`: `"en"`
    - `include_transcript_text`: `true`
- **Credentials:** Apify API credential (“Adams Apify Account”)
- **Connections:**
  - **In ←** Apify YouTube Downloader
  - **Out →** Set Video Data
- **Potential failures / edge cases:**
  - Transcript disabled/unavailable; actor may fail or return no transcript array.
  - Auto-generated transcript only; may affect accuracy.
  - Actor dataset shape differences (workflow code assumes `transcript` is an array of `{text,start,duration}` but pinned data shows `{text,start,end}`).
  - Language mismatch (video not English).

---

### Block 4 — Consolidate Apify Outputs into a Single “Video Data” Object
**Overview:** Normalizes the downloader and transcript responses into one consistent object for later AI steps.  
**Nodes involved:**  
- Set Video Data

#### Node: Set Video Data
- **Type:** `n8n-nodes-base.code` (typeVersion 2)
- **Technical role:** JavaScript transform; merges data from:
  - Form trigger (`YouTube URL Form`)
  - Downloader (`Apify YouTube Downloader`)
  - Transcript dataset item (current input)
- **Configuration choices / logic:**
  - Reads:
    - `formData = $('YouTube URL Form').first().json`
    - `downloaderData = $('Apify YouTube Downloader').first().json`
    - `transcriptData = $input.first().json`
  - Extracts:
    - `video_url` from `downloaderData.downloadUrl` (MP4 link)
    - `youtube_url` from downloader `sourceUrl` or form URL
    - `video_id` from transcript message regex or URL `v=` parameter
    - Metadata: title, duration seconds, channel info, thumbnail, publish date, description, counts
  - Builds transcript:
    - `transcriptSegments` from `transcriptData.transcript`
    - Each segment includes formatted timestamps
    - `transcript_text` is concatenated segment text
  - Returns one item with a rich `json` payload including `segments` and `transcript_text`.
- **Connections:**
  - **In ←** Apify YouTube Transcript
  - **Out →** Analyse Video
- **Potential failures / edge cases:**
  - If downloader dataset lacks `downloadUrl`, `video_url` becomes empty → Gemini “video analyze” likely fails.
  - Segment parsing assumes `duration` exists; if actor returns `end` instead, `end` is computed incorrectly (end becomes `start + 0`).
  - `transcriptData.description` may be undefined; downstream prompt references it.
  - Regex extraction for video ID may fail if transcript message format changes.
- **Version-specific notes:**
  - Uses Code node v2 (supports modern JS + `$()` item selection helpers).
  - Relies on `.first()` which fails if node produced zero items.

---

### Block 5 — Stage 2B: Visual Analysis of the Video (Gemini)
**Overview:** Gemini analyzes the full video and produces a large JSON array of 3-second clips, including webcam overlay detection and crop metadata compatible with Creatomate.  
**Nodes involved:**  
- Analyse Video

#### Node: Analyse Video
- **Type:** `@n8n/n8n-nodes-langchain.googleGemini` (typeVersion 1.1)
- **Technical role:** Calls Google Gemini with `resource: video`, `operation: analyze`.
- **Configuration choices:**
  - Model: `models/gemini-pro-latest`
  - Video input: `videoUrls = {{ $json.video_url }}`
  - Prompt: Very strict instruction set requiring:
    - Full-video coverage
    - Many 3-second clips (50–250+ depending on length)
    - JSON array output only
    - Fields including `person_visible`, `has_usable_screen_content`, `buffer_check`, and optional `webcam_region`, `safe_crop_zone`, `creatomate_crop`
- **Credentials:** Google PaLM / Gemini API credential (“n8n-gmail adam gemini key”)
- **Connections:**
  - **In ←** Set Video Data
  - **Out →** Parse Key Actions
- **Potential failures / edge cases:**
  - Video URL not publicly accessible to Gemini (signed URLs expiring, blocked download, unsupported format).
  - Token/time limits: analyzing long videos may time out or truncate output.
  - Model may return invalid JSON or wrap in markdown; downstream parser tries to strip markdown but may still fail.
  - Output may be extremely large; n8n item size limits or memory constraints can be hit.
  - The prompt demands “NO OVERLAPPING” and “minimum 1 second gap” which the model may violate; downstream does not enforce non-overlap.
- **Integration note:**
  - Output is expected to be a JSON array of clips; later Code node attempts multiple response shapes.

---

### Block 6 — Parse, Validate, Filter, and Categorize Visual Clips
**Overview:** Converts Gemini output into an internal structure; filters unusable “talking head only” clips and incomplete webcam overlay clips; preserves clean screen and croppable webcam overlays; emits stats.  
**Nodes involved:**  
- Parse Key Actions

#### Node: Parse Key Actions
- **Type:** `n8n-nodes-base.code` (typeVersion 2)
- **Technical role:** Robust parser + validator + categorizer for Gemini results.
- **Configuration choices / logic:**
  - Inputs:
    - `geminiOutput = $input.first().json`
    - `videoData = $('Set Video Data').first().json`
  - Attempts to extract response text from multiple possible Gemini response structures:
    - array forms, `.content.parts[0].text`, `.output[...]`, `.text`, string
  - Removes markdown fences like ```json
  - `JSON.parse(responseText)` → expects an array of clips
  - Validations:
    - If parse fails → throws `FATAL: Failed to parse Gemini response ...`
    - If array length = 0 → throws fatal error about no usable B-roll
  - Categorization rules:
    - Clean screen: `person_visible !== true`
    - Webcam overlay usable: `person_visible === true && has_usable_screen_content === true && safe_crop_zone !== null && creatomate_crop !== null`
    - Talking head: `person_visible === true && has_usable_screen_content === false/undefined` (filtered out)
    - Incomplete webcam data: missing crop data (filtered out)
  - Builds `processedClips` with `_processing` metadata:
    - `requires_crop`, `crop_strategy`, `content_preserved_percent`, clip index
  - Final validation:
    - If after filtering no clips remain → throws fatal with counts breakdown
  - Outputs:
    - `key_moments` (usable clips only)
    - `visual_analysis` object with counts, unique apps, positions, total action time, etc.
    - Re-attaches transcript + metadata from `videoData`
- **Connections:**
  - **In ←** Analyse Video
  - **Out →** AI Section Analyzer1
- **Potential failures / edge cases:**
  - Gemini returns JSON but not an array (object) → fatal.
  - Gemini returns huge payload; parsing may exceed memory limits.
  - Clips missing required fields; code does not deeply validate per-field types, so downstream might break later.
  - Clean screen criteria is `person_visible !== true` (so `undefined` counts as clean). If Gemini omits the field, it will be treated as clean screen even if not.
  - No enforcement that `end_seconds = start_seconds + 3` (only assumed).
- **Version-specific notes:**
  - Code node v2 and use of `$()` cross-node access.

---

### Block 7 — Transcript + Key Moments → Structured Video Metadata (OpenAI)
**Overview:** Generates a structured “video overview” object for SEO and repurposing using transcript and visual key moments.  
**Nodes involved:**  
- AI Section Analyzer1

#### Node: AI Section Analyzer1
- **Type:** `@n8n/n8n-nodes-langchain.openAi` (typeVersion 2.1)
- **Technical role:** Calls OpenAI model with enforced JSON Schema output.
- **Configuration choices:**
  - Model: `gpt-5.2`
  - Output format: `json_schema` (strict schema provided in node)
  - System prompt: defines brand voice for `@adamfreelances` (technical, educational, anti-guru) and requires **JSON only**.
  - User prompt includes:
    - Transcript: `{{ $json.transcript_text }}`
    - Key moments JSON: `{{ $json.key_moments.toJsonString() }}`
    - Duration, segment count, video name
    - Video description from: `{{ $('Set Video Data').item.json.description }}`
- **Connections:**
  - **In ←** Parse Key Actions
  - **Out →** Edit Fields
- **Potential failures / edge cases:**
  - If transcript text is extremely long, model context window may overflow → truncation or errors.
  - If `key_moments` is large (hundreds of clips), prompt size can explode.
  - If `$('Set Video Data').item.json.description` is undefined, prompt still renders but with blank value.
  - Schema mismatch: model must match required schema exactly; otherwise node may error.
- **Credentials:** OpenAI credential (“APG n8n”)

---

### Block 8 — Final Payload Shaping
**Overview:** Collects and reshapes fields from prior steps into a final payload to hand off to a downstream workflow.  
**Nodes involved:**  
- Edit Fields

#### Node: Edit Fields
- **Type:** `n8n-nodes-base.set` (typeVersion 3.4)
- **Technical role:** Constructs a flattened/selected output object for downstream processing.
- **Configuration choices:**
  - Assigns:
    - `youtube_url`, `video_id`, `video_name`, `video_url`, `videoDescription`, `channel_title`
    - `transcript_text`, `segments`, `key_moments`, `visual_analysis`
    - `visual_analysis.video_duration_seconds` (duplicated extraction)
    - `output[0].content[0].text.video_overview` (extracts OpenAI response subtree)
- **Key expressions / variables:**
  - Many fields read from `$('Parse Key Actions').item.json...`
  - The `video_overview` extraction: `{{ $json.output[0].content[0].text.video_overview }}`
    - This assumes the OpenAI node output has an `output[0].content[0].text.video_overview` structure.
- **Connections:**
  - **In ←** AI Section Analyzer1
  - **Out →** Call 'Shorts Creation'1
- **Potential failures / edge cases:**
  - If OpenAI node output shape differs (common across model/providers), `$json.output[0]...` may be undefined and set will error or set null.
  - Potential duplication/confusion: `visual_analysis.video_duration_seconds` is both inside `visual_analysis` and separately assigned.

---

### Block 9 — Trigger Sub-workflow: Shorts Creation
**Overview:** Invokes an external workflow that likely creates YouTube Shorts assets based on the extracted moments and metadata.  
**Nodes involved:**  
- Call 'Shorts Creation'1

#### Node: Call 'Shorts Creation'1
- **Type:** `n8n-nodes-base.executeWorkflow` (typeVersion 1.3)
- **Technical role:** Executes another n8n workflow by ID.
- **Configuration choices:**
  - Target workflow: `Short Creation (HUMAN)` (workflowId `nEzi6P6Sf5b5Fepd`)
  - `waitForSubWorkflow: false` (fire-and-forget / async)
  - No defined workflow inputs mapping (empty schema/mapping)
- **Connections:**
  - **In ←** Edit Fields
  - **Out:** none (end of workflow)
- **Potential failures / edge cases:**
  - Target workflow missing, renamed, or access restricted.
  - If the sub-workflow expects inputs but none are passed, it may fail immediately.
  - Async execution means this parent workflow won’t know if downstream succeeded.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Setup Instructions | n8n-nodes-base.stickyNote | Documentation |  |  | ## YouTube Shorts Automation v7.0 / ### @adamfreelances / APIFY YOUTUBE APPROACH … |
| Stage 1 | n8n-nodes-base.stickyNote | Documentation |  |  | ## Stage 1: YouTube URL Input … |
| Stage 2A - Transcript | n8n-nodes-base.stickyNote | Documentation |  |  | ## Stage 2A: Transcript … |
| YouTube URL Form | n8n-nodes-base.formTrigger | Entry point: collect YouTube URL |  | Apify YouTube Downloader | ## Stage 1: YouTube URL Input … |
| Apify YouTube Downloader | @apify/n8n-nodes-apify.apify | Download video → MP4 URL | YouTube URL Form | Apify YouTube Transcript | ## Stage 1: YouTube URL Input … |
| Apify YouTube Transcript | @apify/n8n-nodes-apify.apify | Fetch transcript + metadata | Apify YouTube Downloader | Set Video Data | ## Stage 2A: Transcript … |
| Set Video Data | n8n-nodes-base.code | Merge/normalize video + transcript data | Apify YouTube Transcript | Analyse Video |  |
| Stage 2B - Visual1 | n8n-nodes-base.stickyNote | Documentation |  |  | ## Stage 2B: Visual Analysis … |
| Analyse Video | @n8n/n8n-nodes-langchain.googleGemini | Visual clip extraction + crop metadata | Set Video Data | Parse Key Actions | ## Stage 2B: Visual Analysis … |
| Parse Key Actions | n8n-nodes-base.code | Parse/validate/filter Gemini clips | Analyse Video | AI Section Analyzer1 |  |
| AI Section Analyzer1 | @n8n/n8n-nodes-langchain.openAi | Transcript + moments → structured metadata | Parse Key Actions | Edit Fields |  |
| Edit Fields | n8n-nodes-base.set | Build final payload for downstream | AI Section Analyzer1 | Call 'Shorts Creation'1 |  |
| Call 'Shorts Creation'1 | n8n-nodes-base.executeWorkflow | Trigger downstream shorts workflow (async) | Edit Fields |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation (external link) |  |  | ## Watch the tutorial / https://youtu.be/Gg_bNn-NeI8 / @[youtube](Gg_bNn-NeI8) |

---

## 4. Reproducing the Workflow from Scratch (Manual Steps)

1. **Create a new workflow**
   - Name it something like “Video Digestion Workflow”.

2. **Add Form Trigger**
   - Node: **Form Trigger**
   - Path: `youtube-form-trigger`
   - Form title: “YouTube Video Processor”
   - Description: “Submit a YouTube video URL for automated shorts creation”
   - Add one required field:
     - Label: “YouTube URL”
     - Placeholder: `https://www.youtube.com/watch?v=...`

3. **Add Apify node: YouTube Downloader**
   - Node: **Apify**
   - Operation: **Run actor and get dataset**
   - Actor: **epctex/youtube-video-downloader**
   - Input (Custom Body) set to a JSON expression that includes:
     - `startUrls`: `[ "{{ $json['YouTube URL'] }}" ]`
     - `quality`: `"720"`
     - Apify proxy enabled with group `RESIDENTIAL`
     - `maxRequestRetries`: `2`, `includeFailedVideos`: `false`, `useFfmpeg`: `false`
   - Credentials: configure **Apify API token** in n8n Credentials and select it here.
   - Connect: **Form Trigger → Apify YouTube Downloader**

4. **Add Apify node: YouTube Transcript**
   - Node: **Apify**
   - Operation: **Run actor and get dataset**
   - Actor: **starvibe/youtube-video-transcript**
   - Input body should include:
     - `include_transcript_text`: `true`
     - `language`: `"en"`
     - `youtube_url`: `{{ $('YouTube URL Form').item.json['YouTube URL'] }}`
   - Use same Apify credentials.
   - Connect: **Apify YouTube Downloader → Apify YouTube Transcript**

5. **Add Code node: “Set Video Data”**
   - Node: **Code**
   - Paste logic that:
     - Reads from `YouTube URL Form` and `Apify YouTube Downloader`
     - Uses current input as transcript dataset item
     - Outputs a single JSON item containing:
       - `video_url` (download URL), `youtube_url`, `video_id`, `video_name`, `duration`, `description`, etc.
       - `segments` array + concatenated `transcript_text`
   - Connect: **Apify YouTube Transcript → Set Video Data**

6. **Add Google Gemini node: “Analyse Video”**
   - Node: **Google Gemini (LangChain)**
   - Resource: **video**
   - Operation: **analyze**
   - Model: `models/gemini-pro-latest`
   - Video URLs: `{{ $json.video_url }}`
   - Prompt: paste your long clip-extraction prompt (the workflow’s prompt is extremely strict and Creatomate-oriented).
   - Credentials: configure **Google Gemini/PaLM API key** credential in n8n and select it.
   - Connect: **Set Video Data → Analyse Video**

7. **Add Code node: “Parse Key Actions”**
   - Node: **Code**
   - Implement logic to:
     - Extract text from Gemini response
     - Strip markdown fences
     - `JSON.parse()` into an array
     - Filter clips into:
       - Clean screen clips
       - Webcam overlay clips with valid crop data
       - Drop talking-head-only and incomplete webcam entries
     - Output:
       - `key_moments` and `visual_analysis` stats
       - Include transcript + metadata from “Set Video Data”
   - Connect: **Analyse Video → Parse Key Actions**

8. **Add OpenAI node: “AI Section Analyzer1”**
   - Node: **OpenAI (LangChain)**
   - Model: `gpt-5.2` (or your available equivalent)
   - Configure output format to **JSON Schema** and paste the schema (video_overview object with required fields).
   - System message: enforce brand voice and “JSON only”.
   - User message: include transcript, key moments, and metadata via expressions.
   - Credentials: configure **OpenAI API credential** and select it.
   - Connect: **Parse Key Actions → AI Section Analyzer1**

9. **Add Set node: “Edit Fields”**
   - Node: **Set**
   - Add assignments to forward/reshape:
     - Fields from Parse Key Actions: `youtube_url`, `video_id`, `video_name`, `video_url`, `videoDescription`, `channel_title`, `transcript_text`, `segments`, `key_moments`, `visual_analysis`
     - Extract the OpenAI output’s `video_overview` object from the OpenAI node output shape you receive in your n8n version (adjust expression accordingly).
   - Connect: **AI Section Analyzer1 → Edit Fields**

10. **Add Execute Workflow node: “Call 'Shorts Creation'1”**
   - Node: **Execute Workflow**
   - Select workflow: **Short Creation (HUMAN)** (create/import it separately)
   - Set **Wait for sub-workflow** = `false` for async
   - (Optional but recommended) Define workflow inputs mapping to pass the payload to the sub-workflow explicitly.
   - Connect: **Edit Fields → Call 'Shorts Creation'1**

11. **(Optional) Add Sticky Notes**
   - Add sticky notes for Stage 1/2 documentation and include:
     - Tutorial link: `https://youtu.be/Gg_bNn-NeI8`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Watch the tutorial: https://youtu.be/Gg_bNn-NeI8 | Workflow sticky note (“Watch the tutorial”) |
| “YouTube Shorts Automation v7.0” / “@adamfreelances” / APIFY YOUTUBE APPROACH summary | Setup Instructions sticky note |
| Stage notes describing Stage 1 (URL input) + Stage 2A (Transcript) + Stage 2B (Visual Analysis) | Sticky notes in canvas |

