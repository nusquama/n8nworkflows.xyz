Generate short joke videos from Google Sheets with Google Gemini and Wavespeed AI

https://n8nworkflows.xyz/workflows/generate-short-joke-videos-from-google-sheets-with-google-gemini-and-wavespeed-ai-13040


# Generate short joke videos from Google Sheets with Google Gemini and Wavespeed AI

## 1. Workflow Overview

**Purpose:** Automatically generate short joke videos from ideas stored/managed in Google Sheets. It uses **Google Gemini** to (1) create a fresh joke/video concept and (2) expand it into detailed scene/video prompts, then calls **Wavespeed AI** (via HTTP) to generate an edited background and an **image-to-video** output. Finally, it saves the resulting video URL back to Google Sheets and publishes to social platforms via **Blotato** (TikTok, Instagram, YouTube, X), then marks the row as **DONE**.

**Target use cases:**
- Daily/periodic generation of short-form joke videos for social channels
- Keeping a log of used jokes/ideas to reduce repetition
- Hands-off publishing pipeline once credentials and sheet structure are in place

### 1.1 Triggers (Entry Points)
- Manual execution (for testing)
- Scheduled execution (for automation)

### 1.2 Idea Generation (Gemini Agent + Sheets tools)
- Gemini “Creative” agent produces an idea using two Google Sheets tools (jokes + past jokes) and a “think” tool to inject constraints/idea.

### 1.3 Persist Idea + Metadata (Google Sheets)
- Store idea, metadata, and status fields for downstream steps.

### 1.4 Prompt Expansion (Gemini Scripting Agent)
- Gemini “Scripting” agent produces structured prompts for scenes/video generation.

### 1.5 Scene Formatting
- Code node converts structured prompt output into the JSON/body expected by the edit/video generation APIs.

### 1.6 Video Generation Orchestration (Wavespeed via HTTP + Wait)
- Submit background edit job → wait → fetch edit result
- Submit Img2Vid job → wait → fetch video result

### 1.7 Save, Publish, and Finalize
- Save final URL to Google Sheets
- Fan-out to Blotato nodes (TikTok/Instagram/YouTube/X)
- Merge completion signals and update sheet status to “DONE”

---

## 2. Block-by-Block Analysis

### Block 1 — Workflow Start (Manual + Schedule)
**Overview:** Provides two ways to start the same pipeline: manually for testing or via a schedule for automation.  
**Nodes Involved:**  
- `When clicking ‘Execute workflow’`  
- `Schedule Trigger`

#### Node: When clicking ‘Execute workflow’
- **Type/role:** `Manual Trigger` — starts workflow on demand.
- **Config choices:** Default manual trigger (no parameters).
- **Connections:** Outputs to `Creative Video Idea`.
- **Edge cases:** None (only runs when user clicks execute).

#### Node: Schedule Trigger
- **Type/role:** `Schedule Trigger` — periodic automated runs.
- **Config choices:** Not shown in JSON (parameters empty). You must configure cadence (e.g., every day at 09:00).
- **Connections:** Outputs to `Creative Video Idea`.
- **Edge cases:** Misconfigured timezone/cadence; may run too often and exhaust API quotas.

---

### Block 2 — Creative Idea Generation (Gemini + Sheets Tools)
**Overview:** Uses a Gemini agent to generate a new joke-video idea while consulting Google Sheets (current joke pool + past jokes) to avoid repetition and meet constraints.  
**Nodes Involved:**  
- `Creative Video Idea` (Agent)  
- `Gemini Model (Creative)`  
- `Parse AI Output`  
- `Inject Idea`  
- `Google Sheet Jokes`  
- `Google Sheet Past Jokes`

#### Node: Gemini Model (Creative)
- **Type/role:** `lmChatGoogleGemini` — chat LLM backing the creative agent and structured parser.
- **Config choices:** Parameters are blank in JSON; typically includes:
  - Gemini model name (e.g., `gemini-1.5-pro` or `gemini-1.5-flash`)
  - Temperature tuned for creativity
  - Google/Gemini credentials
- **Connections:** Provides `ai_languageModel` to:
  - `Creative Video Idea`
  - `Parse AI Output`
- **Edge cases:** Invalid/expired credentials; model not available in region; safety filters blocking content; quota limits.

#### Node: Google Sheet Jokes
- **Type/role:** `googleSheetsTool` — LangChain tool wrapper to let the agent read/query jokes sheet.
- **Config choices:** Not present; usually configured with:
  - Spreadsheet ID, Sheet name/range
  - Read/list/search rows
  - Credentials (Google OAuth2 or service account)
- **Connections:** Exposed as `ai_tool` to `Creative Video Idea`.
- **Edge cases:** Permission issues; wrong range; API rate limits; empty sheet.

#### Node: Google Sheet Past Jokes
- **Type/role:** `googleSheetsTool` — tool to check previously used jokes/ideas.
- **Config choices:** Similar to above; typically points at a “Past”/history tab.
- **Connections:** Exposed as `ai_tool` to `Creative Video Idea`.
- **Edge cases:** Same as above; if history isn’t maintained, agent may repeat content.

#### Node: Inject Idea
- **Type/role:** `toolThink` — internal reasoning/tool instruction node used by the agent (commonly to enforce constraints like style, length, format).
- **Config choices:** Not shown; typically contains guidance like “Generate a short, family-friendly joke concept and return JSON matching schema…”.
- **Connections:** Exposed as `ai_tool` to `Creative Video Idea`.
- **Edge cases:** If instructions contradict parser schema, downstream parsing fails.

#### Node: Parse AI Output
- **Type/role:** `outputParserStructured` — validates/converts LLM output into a structured object for downstream use.
- **Config choices:** Not shown; normally contains a JSON schema (fields like `title`, `joke`, `visual_style`, `hashtags`, etc.).
- **Connections:** Provides `ai_outputParser` to `Creative Video Idea`.
- **Edge cases:** Most common failure point: LLM output not matching schema (missing fields, wrong types).

#### Node: Creative Video Idea
- **Type/role:** `LangChain Agent` — orchestrates Gemini + tools to produce a structured “video idea”.
- **Config choices (interpreted):**
  - Uses `Gemini Model (Creative)` as LLM
  - Uses tools: `Google Sheet Jokes`, `Google Sheet Past Jokes`, `Inject Idea`
  - Uses `Parse AI Output` to enforce structured result
- **Input/Output:** Triggered by Manual/Schedule; outputs structured idea to `Save Idea & Metadata`.
- **Edge cases:** Tool failures (Sheets), parser failures, or agent looping if tools return unexpected formats.

---

### Block 3 — Save Idea & Metadata (Google Sheets)
**Overview:** Stores the generated idea and any metadata needed for tracking and later updates (e.g., row ID, status, timestamps).  
**Nodes Involved:**  
- `Save Idea & Metadata`

#### Node: Save Idea & Metadata
- **Type/role:** `Google Sheets` — write/update a row with the new idea.
- **Config choices:** Parameters empty in JSON; typically:
  - Operation: Append row or Update row
  - Mappings from agent output (e.g., idea text, title, status = “PENDING/GENERATING”)
  - Store a key field (Row ID) for later updates (“DONE”)
- **Connections:** Outputs to `Detailed Video Prompts`.
- **Edge cases:** Wrong sheet/range; write permissions; missing required columns; row identifier not saved (breaks later updates).

---

### Block 4 — Detailed Prompt Generation (Gemini Scripting Agent)
**Overview:** Expands the saved idea into detailed, structured prompts suitable for video generation (scenes, image prompts, motion, camera directions, captions).  
**Nodes Involved:**  
- `Detailed Video Prompts` (Agent)  
- `Gemini Model (Scripting)`  
- `Refine and Validate Prompts`  
- `Parse Video Prompt`

#### Node: Gemini Model (Scripting)
- **Type/role:** `lmChatGoogleGemini` — LLM tuned for structured scripting.
- **Config choices:** Blank in JSON; usually lower temperature than creative step to ensure consistent structure.
- **Connections:** Provides `ai_languageModel` to `Detailed Video Prompts`.
- **Edge cases:** Same Gemini issues (auth/quota/safety), plus stricter formatting needs.

#### Node: Refine and Validate Prompts
- **Type/role:** `toolThink` — ensures prompts meet constraints (length, banned terms, required fields).
- **Config choices:** Not shown; typically enforces “return exactly N scenes”, “vertical 9:16”, “no copyrighted characters”, etc.
- **Connections:** Exposed as `ai_tool` to `Detailed Video Prompts`.
- **Edge cases:** Over-restrictive rules can cause agent to produce empty or repetitive output.

#### Node: Parse Video Prompt
- **Type/role:** `outputParserStructured` — validates the scripting output into a stable structure (e.g., array of scenes).
- **Config choices:** Not shown; typically schema like:
  - `scenes[]: { image_prompt, motion_prompt, duration_s, on_screen_text, sfx }`
- **Connections:** Provides `ai_outputParser` to `Detailed Video Prompts`.
- **Edge cases:** Schema mismatch is common; ensure the agent prompt and parser schema align.

#### Node: Detailed Video Prompts
- **Type/role:** `LangChain Agent` — turns idea into structured production prompts.
- **Connections:** Outputs to `Format Scene JSON`.
- **Edge cases:** Parser failures; inconsistent number of scenes; missing fields required by Wavespeed.

---

### Block 5 — Format & Prepare API Payloads
**Overview:** Converts structured scene data into the exact fields required by the downstream HTTP requests, then maps/sets fields used across later nodes.  
**Nodes Involved:**  
- `Format Scene JSON`  
- `Edit Fields`

#### Node: Format Scene JSON
- **Type/role:** `Code` — transforms agent output into a JSON payload for Wavespeed.
- **Config choices:** Code not included in JSON export (parameters empty). Typically:
  - Build a `scenes` array, join prompts, compute durations
  - Create request bodies for “background edit” and/or “img2vid”
  - Preserve identifiers from Sheets (row ID)
- **Connections:** Outputs to `Edit Fields`.
- **Edge cases:** Undefined paths if parser output differs; runtime JS errors; large payloads exceeding API limits.

#### Node: Edit Fields
- **Type/role:** `Set` — renames/selects fields for the next HTTP request.
- **Config choices:** Not shown; typically:
  - Keep only: `jobPayload`, `prompt`, `imageUrl`, `rowId`, etc.
  - Set constant fields like `aspect_ratio: "9:16"`
- **Connections:** Outputs to `Generate Background Edit`.
- **Edge cases:** Accidental field removal causes later HTTP nodes to miss required values.

---

### Block 6 — Wavespeed Background Edit Job (Async)
**Overview:** Submits a background edit request to Wavespeed, waits for completion, then fetches the result.  
**Nodes Involved:**  
- `Generate Background Edit`  
- `Wait for Edit`  
- `Get Edit Result`

#### Node: Generate Background Edit
- **Type/role:** `HTTP Request` — creates an edit job (async).
- **Config choices:** Parameters absent; typically:
  - Method: POST
  - URL: Wavespeed endpoint (edit)
  - Headers: Authorization (API key/bearer), Content-Type
  - Body: from `Edit Fields` / `Format Scene JSON`
- **Connections:** Outputs to `Wait for Edit`.
- **Edge cases:** 401/403 auth; 429 rate limit; 400 due to invalid body; timeouts.

#### Node: Wait for Edit
- **Type/role:** `Wait` — pauses workflow until a time passes or a webhook resumes execution.
- **Config choices:** Wait parameters empty; **webhookId is present**.
- **Connections:** Outputs to `Get Edit Result`.
- **Edge cases / important:** This workflow has **two Wait nodes sharing the same webhookId** (`e455bc41-...`). In n8n, each Wait node typically needs its own resume mechanism; reusing IDs can cause confusion or resume the wrong execution path depending on configuration. Validate your n8n version behavior and ensure unique resume webhooks if using webhook-based waits.

#### Node: Get Edit Result
- **Type/role:** `HTTP Request` — polls or fetches final edit asset using job ID from previous step.
- **Config choices:** Typically GET to `/result/{jobId}` or similar.
- **Connections:** Outputs to `Generate Video (Img2Vid)`.
- **Edge cases:** Job still processing; handle retries/backoff; missing jobId due to prior mapping errors.

---

### Block 7 — Wavespeed Img2Vid Job (Async)
**Overview:** Generates the final short video using the edited image/background result, waits for completion, then fetches the final URL.  
**Nodes Involved:**  
- `Generate Video (Img2Vid)`  
- `Wait for Video Gen`  
- `Get Video Result`

#### Node: Generate Video (Img2Vid)
- **Type/role:** `HTTP Request` — submits image-to-video generation job.
- **Config choices:** Typically POST with:
  - Input image from `Get Edit Result`
  - Motion prompt / style prompt
  - Duration, fps, aspect ratio
- **Connections:** Outputs to `Wait for Video Gen`.
- **Edge cases:** Same HTTP issues; invalid image URL; content moderation; long runtimes.

#### Node: Wait for Video Gen
- **Type/role:** `Wait` — pause until job completion.
- **Config choices:** Empty; shares the same webhookId as `Wait for Edit` in the export.
- **Connections:** Outputs to `Get Video Result`.
- **Edge cases:** Same webhookId concern; also overall execution timeout if waits are too long for your n8n setup.

#### Node: Get Video Result
- **Type/role:** `HTTP Request` — fetch final video URL/asset metadata.
- **Config choices:** Typically GET using job ID.
- **Connections:** Outputs to `Save Final URL`.
- **Edge cases:** Job not complete; transient 5xx; missing URL fields.

---

### Block 8 — Save URL, Publish to Socials, Mark Done
**Overview:** Saves the final video URL to Google Sheets, triggers posting to multiple platforms via Blotato, waits for all to complete (via Merge), then updates status to DONE.  
**Nodes Involved:**  
- `Save Final URL`  
- `Tiktok`  
- `Instagram`  
- `Youtube`  
- `Twitter (X)`  
- `Merge1`  
- `Update Status to "DONE"`

#### Node: Save Final URL
- **Type/role:** `Google Sheets` — update row with the final generated video URL and possibly other fields (thumbnail, duration).
- **Config choices:** Typically Update row by rowId saved earlier; write `final_url`.
- **Connections:** Fans out to `Tiktok`, `Instagram`, `Youtube`, `Twitter (X)`.
- **Edge cases:** If rowId not propagated, may update wrong row; sheet write conflicts.

#### Nodes: Tiktok / Instagram / Youtube / Twitter (X)
- **Type/role:** `@blotato/n8n-nodes-blotato.blotato` — publish content to each platform through Blotato.
- **Config choices:** Not shown; typically includes:
  - Blotato API key/credential
  - Platform selection + posting parameters (caption, hashtags, media URL)
  - Optional scheduling, privacy settings
- **Connections:** Each outputs to `Merge1` on a distinct input index.
- **Edge cases:** Platform posting failures; unsupported media format; caption length limits; rate limits; credential scope issues.

#### Node: Merge1
- **Type/role:** `Merge` — synchronizes multiple posting branches before final status update.
- **Config choices:** Not shown; but because it has 4 inputs connected, it likely uses a mode such as “Wait for all inputs”.
- **Connections:** Outputs to `Update Status to "DONE"`.
- **Edge cases:** If any branch errors or doesn’t emit an item, merge may not complete; consider “Continue On Fail” on posting nodes if you still want DONE.

#### Node: Update Status to "DONE"
- **Type/role:** `Google Sheets` — final update marking the content row as completed.
- **Config choices:** Update row field `status = DONE`.
- **Connections:** Terminal node.
- **Edge cases:** Same rowId propagation dependency.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Creative Video Idea |  |
| Schedule Trigger | Schedule Trigger | Automated entry point | — | Creative Video Idea |  |
| Gemini Model (Creative) | Google Gemini Chat Model | LLM for idea generation + parsing | — | Creative Video Idea; Parse AI Output |  |
| Google Sheet Past Jokes | Google Sheets Tool | Agent tool to query past jokes | — | Creative Video Idea (tool) |  |
| Google Sheet Jokes | Google Sheets Tool | Agent tool to query jokes pool | — | Creative Video Idea (tool) |  |
| Inject Idea | Tool (Think) | Constraint/idea injection tool | — | Creative Video Idea (tool) |  |
| Parse AI Output | Structured Output Parser | Validates creative idea schema | Gemini Model (Creative) (model) | Creative Video Idea (parser) |  |
| Creative Video Idea | LangChain Agent | Produces structured video idea | Manual Trigger; Schedule Trigger | Save Idea & Metadata |  |
| Save Idea & Metadata | Google Sheets | Persist idea + identifiers | Creative Video Idea | Detailed Video Prompts |  |
| Gemini Model (Scripting) | Google Gemini Chat Model | LLM for prompt scripting | — | Detailed Video Prompts |  |
| Refine and Validate Prompts | Tool (Think) | Enforces prompt constraints | — | Detailed Video Prompts (tool) |  |
| Parse Video Prompt | Structured Output Parser | Validates scene/prompt schema | — | Detailed Video Prompts (parser) |  |
| Detailed Video Prompts | LangChain Agent | Produces structured scene prompts | Save Idea & Metadata | Format Scene JSON |  |
| Format Scene JSON | Code | Build API-ready JSON payload | Detailed Video Prompts | Edit Fields |  |
| Edit Fields | Set | Normalize fields for HTTP calls | Format Scene JSON | Generate Background Edit |  |
| Generate Background Edit | HTTP Request | Submit background edit job | Edit Fields | Wait for Edit |  |
| Wait for Edit | Wait | Pause until edit completes | Generate Background Edit | Get Edit Result |  |
| Get Edit Result | HTTP Request | Fetch background edit output | Wait for Edit | Generate Video (Img2Vid) |  |
| Generate Video (Img2Vid) | HTTP Request | Submit img2vid job | Get Edit Result | Wait for Video Gen |  |
| Wait for Video Gen | Wait | Pause until video completes | Generate Video (Img2Vid) | Get Video Result |  |
| Get Video Result | HTTP Request | Fetch final video URL | Wait for Video Gen | Save Final URL |  |
| Save Final URL | Google Sheets | Save final asset URL | Get Video Result | Tiktok; Instagram; Youtube; Twitter (X) |  |
| Tiktok | Blotato | Publish to TikTok | Save Final URL | Merge1 |  |
| Instagram | Blotato | Publish to Instagram | Save Final URL | Merge1 |  |
| Youtube | Blotato | Publish to YouTube | Save Final URL | Merge1 |  |
| Twitter (X) | Blotato | Publish to X | Save Final URL | Merge1 |  |
| Merge1 | Merge | Wait for all posts before final update | Tiktok; Instagram; Youtube; Twitter (X) | Update Status to "DONE" |  |
| Update Status to "DONE" | Google Sheets | Mark content as completed | Merge1 | — |  |
| Sticky Note | Sticky Note | Comment container | — | — | *(empty)* |
| Sticky Note1 | Sticky Note | Comment container | — | — | *(empty)* |
| Sticky Note2 | Sticky Note | Comment container | — | — | *(empty)* |
| Sticky Note3 | Sticky Note | Comment container | — | — | *(empty)* |
| Sticky Note5 | Sticky Note | Comment container | — | — | *(empty)* |
| Sticky Note12 | Sticky Note | Comment container | — | — | *(empty)* |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add triggers**
   1) Add **Manual Trigger** node named `When clicking ‘Execute workflow’`.  
   2) Add **Schedule Trigger** node named `Schedule Trigger` and set your schedule (e.g., daily).  
   3) Connect both triggers to the same next node (`Creative Video Idea`).

3. **Add Gemini creative model**
   - Add **Google Gemini Chat Model** node named `Gemini Model (Creative)`.
   - Configure **Google/Gemini credentials**.
   - Choose a creative model and set temperature as desired.

4. **Add Google Sheets tools for the agent**
   1) Add **Google Sheets Tool** node `Google Sheet Jokes`:
      - Set credentials (Google OAuth2/service account).
      - Configure it to read/query the sheet containing joke candidates.
   2) Add **Google Sheets Tool** node `Google Sheet Past Jokes`:
      - Configure it to read/query a history sheet of already-used jokes/ideas.

5. **Add structured output parser for the idea**
   - Add **Structured Output Parser** node named `Parse AI Output`.
   - Define the schema you want the agent to return (e.g., `idea_title`, `joke_text`, `visual_style`, `caption`, `hashtags`, `language`, etc.).
   - Attach `Gemini Model (Creative)` as its language model input (n8n “ai_languageModel” connection).

6. **Add agent tool for constraints**
   - Add **Tool (Think)** node `Inject Idea` and put your constraints/instructions (family-friendly, short, unique vs. past jokes, output must match schema).

7. **Add the creative agent**
   - Add **Agent** node named `Creative Video Idea`.
   - Connect:
     - Triggers → `Creative Video Idea` (main)
     - `Gemini Model (Creative)` → `Creative Video Idea` (ai_languageModel)
     - Tools: `Google Sheet Jokes`, `Google Sheet Past Jokes`, `Inject Idea` → `Creative Video Idea` (ai_tool)
     - `Parse AI Output` → `Creative Video Idea` (ai_outputParser)
   - Ensure the agent prompt instructs it to use the tools and return output matching the parser schema.

8. **Persist the idea**
   - Add **Google Sheets** node `Save Idea & Metadata`.
   - Configure to **append** or **update** a row:
     - Write idea fields, status (e.g., `GENERATING`), timestamps.
     - Store a stable identifier for later updates (row number/id).
   - Connect `Creative Video Idea` → `Save Idea & Metadata`.

9. **Add Gemini scripting model**
   - Add **Google Gemini Chat Model** node `Gemini Model (Scripting)` with credentials.
   - Choose a model/temperature tuned for consistency.

10. **Add parser + constraint tool for video prompts**
   1) Add **Tool (Think)** node `Refine and Validate Prompts` with constraints (N scenes, 9:16, duration, safe content, avoid trademarks).
   2) Add **Structured Output Parser** node `Parse Video Prompt` with schema for scenes/prompts.

11. **Add the scripting agent**
   - Add **Agent** node `Detailed Video Prompts`.
   - Connect:
     - `Save Idea & Metadata` → `Detailed Video Prompts` (main)
     - `Gemini Model (Scripting)` → `Detailed Video Prompts` (ai_languageModel)
     - `Refine and Validate Prompts` → `Detailed Video Prompts` (ai_tool)
     - `Parse Video Prompt` → `Detailed Video Prompts` (ai_outputParser)

12. **Format payload for Wavespeed**
   1) Add **Code** node `Format Scene JSON`:
      - Transform `Detailed Video Prompts` output into the JSON bodies required by your Wavespeed endpoints.
      - Preserve `rowId` (or another key) from the Sheets step.
   2) Add **Set** node `Edit Fields` to keep only the fields needed by HTTP calls (e.g., `editBody`, `img2vidBody`, `rowId`).

13. **Configure Wavespeed API calls (Background Edit)**
   1) Add **HTTP Request** node `Generate Background Edit`:
      - POST to Wavespeed “edit” endpoint
      - Auth header (API key)
      - Body from `Edit Fields`
   2) Add **Wait** node `Wait for Edit`:
      - Use time-based wait or webhook-resume design.
   3) Add **HTTP Request** node `Get Edit Result`:
      - GET/poll using `jobId` from the edit submission response

14. **Configure Wavespeed API calls (Img2Vid)**
   1) Add **HTTP Request** node `Generate Video (Img2Vid)`:
      - POST to img2vid endpoint
      - Use output asset URL from `Get Edit Result`
   2) Add **Wait** node `Wait for Video Gen`
   3) Add **HTTP Request** node `Get Video Result` to retrieve final video URL

   **Important:** Ensure each Wait node is configured correctly; avoid ambiguous resume configuration. If using webhook resume, prefer unique resume webhooks per Wait node.

15. **Save final video URL**
   - Add **Google Sheets** node `Save Final URL`:
     - Update the same row using `rowId`
     - Write `final_video_url` and any additional metadata
   - Connect `Get Video Result` → `Save Final URL`.

16. **Publish via Blotato**
   - Add four **Blotato** nodes: `Tiktok`, `Instagram`, `Youtube`, `Twitter (X)`.
   - Configure Blotato credentials and each platform’s post settings (caption/hashtags/media URL).
   - Connect `Save Final URL` to each Blotato node.

17. **Merge completion and mark DONE**
   1) Add **Merge** node `Merge1` configured to wait for all inputs (or your preferred merge mode).
   2) Connect each Blotato node output to a different input of `Merge1`.
   3) Add **Google Sheets** node `Update Status to "DONE"` to update status field to DONE for the same rowId.
   4) Connect `Merge1` → `Update Status to "DONE"`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in the provided workflow export. | No additional embedded documentation was included. |
| Both `Wait for Edit` and `Wait for Video Gen` show the same webhookId in the export. Validate Wait/resume configuration to avoid incorrect resumes. | n8n Wait node behavior can vary by configuration/version; prefer distinct resume paths. |
| Several nodes have empty parameters in the export (HTTP Requests, Sheets nodes, Agents/Parsers). You must configure endpoints, schemas, and mappings to match your Sheets layout and Wavespeed/Blotato APIs. | Reproduction requires filling in credentials, sheet IDs/ranges, request URLs, and JSON schemas. |