Create AI product review videos with Gemini, Veo 3, Blotato and Google Sheets

https://n8nworkflows.xyz/workflows/create-ai-product-review-videos-with-gemini--veo-3--blotato-and-google-sheets-12929


# Create AI product review videos with Gemini, Veo 3, Blotato and Google Sheets

## 1. Workflow Overview

**Workflow name:** Auto-Generate AI Product Review Videos  
**Stated title:** Create AI product review videos with Gemini, Veo 3, Blotato and Google Sheets

**Purpose:**  
Collect a product image + short description via an n8n Form, use **Google Gemini** to (1) analyze the product image, (2) generate an **UGC-style image prompt** and a **3-scene continuous UGC video prompt sequence**, then call external APIs to generate an image and three video clips, **merge** the clips into one final video, **publish** to Facebook via **Blotato**, and **log** success/error to **Google Sheets**.

### 1.1 Logical Blocks
1. **Product Input & Image Understanding**: form intake + Gemini image analysis.  
2. **Image Prompt Generation (Structured)**: agent generates strict JSON `{ image_prompt }`.  
3. **Image Rendering & Polling**: call image generation API, wait, fetch generated image URL(s).  
4. **Video Prompt Generation (Structured)**: agent generates strict JSON for title/description + 3 scene prompts; split into per-scene items.  
5. **Video Rendering Loop & Status Polling**: iterate scenes, generate video, poll until ready; aggregate resulting clip URLs.  
6. **Video Merge & Download**: concatenate clips via external endpoint; download final video file.  
7. **Publish & Log**: upload to Blotato, post to Facebook; log posted/error status to Sheets.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Product Input & Image Understanding
**Overview:** Receives a product image and a short description, then uses Gemini multimodal analysis to extract visible packaging/context text for downstream prompt-building.

**Nodes involved:**
- **Product Image & Info Input**
- **Analyze Product Image**

#### Node: Product Image & Info Input
- **Type / role:** `Form Trigger` — workflow entrypoint (web form upload).
- **Configuration choices:**
  - Form title: **“UCG VIDEO”**
  - Fields:
    - **Image** (file upload; becomes binary property named `Image`)
    - **Video review description** (text)
  - Description: “Enter product images to generate content”
- **Inputs/outputs:**
  - **No input** (trigger).
  - Output goes to **Analyze Product Image**.
- **Edge cases / failures:**
  - Missing file upload or wrong field name will break downstream `binaryPropertyName: Image`.
  - Large files can exceed n8n limits depending on instance config.

#### Node: Analyze Product Image
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` (Gemini multimodal “analyze image” operation).
- **Configuration choices:**
  - Resource: **image**, Operation: **analyze**
  - Input type: **binary**
  - Binary property name: **Image**
  - Model: `models/gemini-3-pro-preview`
- **Credentials:** Google PaLM/Gemini API credential `api google`
- **Inputs/outputs:**
  - Input: binary image from the Form trigger.
  - Output: text description under `content.parts[0].text` (used later by the prompt agent).
- **Edge cases / failures:**
  - Auth/quota errors from Gemini.
  - If Gemini returns unexpected structure, expressions referencing `content.parts[0].text` may fail.

---

### Block 2.2 — Image Prompt Generation (Structured)
**Overview:** Uses a LangChain Agent with strict system rules (UGC realism + “don’t invent text”) to produce a single JSON object `{ "image_prompt": "..." }`.

**Nodes involved:**
- **Google Gemini Chat Model**
- **Structured Output Parser1**
- **AI Prompt Agent**

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini` — LLM provider for the agent.
- **Configuration choices:**
  - Model: `models/gemini-3-pro-preview`
- **Credentials:** `api google`
- **Connections:**
  - Supplies language model to **AI Prompt Agent**
  - Also connected to **Structured Output Parser1** as its LLM (for auto-fix).
- **Edge cases / failures:**
  - Model availability changes (preview models can be deprecated).
  - Rate limits.

#### Node: Structured Output Parser1
- **Type / role:** `outputParserStructured` — enforces/repairs JSON output.
- **Configuration choices:**
  - `autoFix: true` (tries to repair invalid JSON using the connected model)
  - JSON schema example:
    ```json
    { "image_prompt": "string" }
    ```
- **Connections:**
  - Used as output parser for **AI Prompt Agent**.
- **Edge cases / failures:**
  - If the model returns non-repairable output, parsing fails.
  - Overly permissive schema example: only checks shape, not content constraints.

#### Node: AI Prompt Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates the image generation prompt.
- **Configuration choices (interpreted):**
  - Prompt includes:
    - User description from the form:
      - Expression: `{{ $('Product Image & Info Input').item.json['Mô tả video review'] }}`
      - **Risk:** field label mismatch (form shows “Video review description”, but expression uses Vietnamese key).
    - Gemini image analysis text:
      - `{{ $json.content.parts[0].text }}`
  - System message (Vietnamese): strict UGC style rules; must preserve all visible text 100%; output must be **exactly one JSON object** with key `image_prompt`; max 120 words; no advertising tone.
  - `hasOutputParser: true` to enforce Structured Output Parser.
- **Inputs/outputs:**
  - Input: output of **Analyze Product Image**
  - Output: `output.image_prompt` used by **Generate Images**
- **Edge cases / failures:**
  - **Field name mismatch** likely causes empty user description.
  - If Gemini analysis is empty, prompt quality degrades.
  - Output parser may reject if agent adds extra text.

---

### Block 2.3 — Image Rendering & Polling
**Overview:** Calls an external image generation endpoint (geminigen.ai) with the prompt, waits, then polls job history to retrieve generated image URLs.

**Nodes involved:**
- **Generate Images**
- **Wait3**
- **Get Images**

#### Node: Generate Images
- **Type / role:** `HTTP Request` — submits an image generation job.
- **Configuration choices:**
  - Method: **POST**
  - URL: `https://api.geminigen.ai/uapi/v1/generate_image`
  - Body type: `multipart-form-data`
  - Parameters:
    - `prompt` = `{{ $json.output.image_prompt }}`
    - `model` = `imagen-pro`
    - `aspect_ratio` = `9:16`
    - `style` = `Photorealistic`
  - Auth: `httpHeaderAuth` credential **Vbee API**
- **Inputs/outputs:**
  - Input: JSON from AI Prompt Agent (must contain `output.image_prompt`).
  - Output: expected to include a `uuid` for polling (used by Wait/Get).
- **Edge cases / failures:**
  - API key invalid/expired.
  - Multipart formatting issues.
  - Provider may return different fields; workflow assumes `uuid` exists.

#### Node: Wait3
- **Type / role:** `Wait` — delays before polling image result.
- **Configuration choices:** Wait **15 seconds**.
- **Inputs/outputs:** passes through prior JSON (incl. uuid) to Get Images.
- **Edge cases:**
  - If rendering takes longer than 15s, polling may return “processing” or empty results (no retry loop here).

#### Node: Get Images
- **Type / role:** `HTTP Request` — polls job history for the generated image.
- **Configuration choices:**
  - URL: `https://api.geminigen.ai/uapi/v1/history/{{ $json.uuid }}`
  - Auth: `httpHeaderAuth` **Vbee API**
- **Outputs:**
  - Expected to contain `generated_image[0].image_url`, referenced later by video generation.
- **Edge cases / failures:**
  - If job not ready, `generated_image` may be missing/empty → later expression fails.
  - No explicit status check/retry for images.

---

### Block 2.4 — Video Prompt Generation (Structured) + Scene Splitting
**Overview:** Generates a continuous 3-scene UGC video prompt sequence (with Vietnamese spoken dialogue embedded), parses into structured JSON, then splits into individual scene items for looping video generation.

**Nodes involved:**
- **Google Gemini Chat Model1**
- **Video Prompt Parser**
- **Create Video Prompt**
- **Split Video Prompts**

#### Node: Google Gemini Chat Model1
- **Type / role:** `lmChatGoogleGemini` — LLM provider for video prompt agent and parser auto-fix.
- **Configuration:**
  - Model: `models/gemini-3-pro-preview`
- **Credentials:** `api google`
- **Connections:** feeds **Create Video Prompt** and **Video Prompt Parser**.
- **Edge cases:** same as other Gemini model node.

#### Node: Video Prompt Parser
- **Type / role:** `outputParserStructured` — attempts to enforce a JSON structure for video prompts.
- **Configuration choices:**
  - `autoFix: true`
  - Schema example includes `video_title`, `video_description`, and a nested `scenes` object, but the agent’s contract outputs `scene_1/2/3` at the top level.
  - **Important mismatch:** parser example and agent’s required output structure do not align.
- **Connections:** set as the output parser for **Create Video Prompt**.
- **Edge cases / failures:**
  - Because of structure mismatch, auto-fix may rewrite keys unexpectedly, or parsing can fail, or downstream Split node may not find `output.scenes`.

#### Node: Create Video Prompt
- **Type / role:** `agent` — writes Vietnamese title/description and 3 continuous prompts with dialogue.
- **Configuration choices:**
  - User text: `{{ $json.input_text }}` (but upstream does not set `input_text`; likely empty unless set elsewhere).
  - System message (English) defines strict continuity rules and JSON-only output contract:
    - Must output JSON with keys: `video_title`, `video_description`, `scene_1.video_prompt`, `scene_2.video_prompt`, `scene_3.video_prompt`
    - Dialogue **must be Vietnamese** and embedded in quotes in the prompt.
- **Inputs/outputs:**
  - Input comes from **Get Images** (so it can be considered “reference image context” in narrative, but no actual image passed here unless included in text).
  - Output is parsed into `output` by the structured parser.
- **Edge cases / failures:**
  - `input_text` missing reduces personalization.
  - Parser mismatch may break `Split Video Prompts`.

#### Node: Split Video Prompts
- **Type / role:** `Split Out` — converts scenes collection into per-item scene prompts.
- **Configuration choices:**
  - Field to split: `output.scenes`
- **Inputs/outputs:**
  - Input: parsed JSON from Create Video Prompt.
  - Output: multiple items each containing a `video_prompt` (as expected by later nodes).
- **Edge cases / failures:**
  - If actual output is `output.scene_1`/`scene_2`/`scene_3` (not `output.scenes`), split will yield **0 items** or error.
  - Pinned data suggests items contain `video_prompt` directly, indicating manual adjustments or parser auto-fix behavior.

---

### Block 2.5 — Video Rendering Loop & Status Polling
**Overview:** Iterates through each scene prompt, requests video generation (Veo 3 fast), polls until completion, then aggregates final clip URLs across scenes.

**Nodes involved:**
- **Loop Video Prompts**
- **Generate Videos**
- **Wait1**
- **Get Videos**
- **If1**
- **Aggregate Videos**

#### Node: Loop Video Prompts
- **Type / role:** `Split In Batches` — batching/loop control over per-scene items.
- **Configuration choices:** default options (batch size not explicitly set; n8n defaults apply).
- **Connections:**
  - Main output 0 → **Generate Videos**
  - Main output 1 → **Aggregate Videos** (called when batching completes)
- **Edge cases / failures:**
  - If no items arrive from Split, loop ends immediately → aggregation runs with empty set.

#### Node: Generate Videos
- **Type / role:** `HTTP Request` — submits Veo video generation job per scene.
- **Configuration choices:**
  - POST `https://api.geminigen.ai/uapi/v1/video-gen/veo`
  - Multipart params:
    - `prompt` = `{{ $json.video_prompt }}`
    - `model` = `veo-3-fast`
    - `resolution` = `720p`
    - `aspect_ratio` = `9:16`
    - `file_urls` = `{{ $('Get Images').item.json.generated_image[0].image_url }}`
      - Uses the generated image as reference.
  - Auth: `httpHeaderAuth` **Vbee API**
- **Outputs:** expected `uuid` for polling.
- **Edge cases / failures:**
  - If `generated_image[0]` missing → expression error.
  - Provider may require multiple file URLs or different field naming.

#### Node: Wait1
- **Type / role:** `Wait` — delays between status checks.
- **Configuration:** 10 seconds.
- **Edge cases:** for long renders, repeated loop will continue polling.

#### Node: Get Videos
- **Type / role:** `HTTP Request` — polls video render status.
- **Configuration choices:**
  - URL: `https://api.geminigen.ai/uapi/v1/history/{{ $json.uuid }}`
  - Auth type set to header auth, but **credentials are not attached in JSON** for this node.
- **Edge cases / failures:**
  - If auth is required (likely), this node will fail unless n8n inherits credentials (it won’t by default). You likely must set the **Vbee API** credential here.
  - Response shape must include `status` and `generated_video[0].video_url`.

#### Node: If1
- **Type / role:** `If` — checks whether the video is finished.
- **Condition:** `{{ $json.status }}` **equals** `2` (numeric).
  - Assumes provider uses status code `2` as “completed”.
- **Connections:**
  - True → **Loop Video Prompts** (continue to next scene / next batch)
  - False → **Wait1** (poll again)
- **Edge cases:**
  - Status might be string or different numeric codes; strict validation is enabled.

#### Node: Aggregate Videos
- **Type / role:** `Aggregate` — collects video URLs from each completed scene into an array.
- **Configuration choices:**
  - Aggregates field: `generated_video[0].video_url`
  - Output becomes something like `video_url: [url1, url2, ...]`
- **Inputs/outputs:**
  - Input from Loop’s “done” output.
  - Output goes to JS code to reformat for merge endpoint.
- **Edge cases:**
  - Missing `generated_video[0].video_url` will produce empty entries.

---

### Block 2.6 — Video Merge & Download
**Overview:** Formats the aggregated URLs, calls an external concatenation service, then downloads the merged video file.

**Nodes involved:**
- **Code in JavaScript**
- **Merge Videos**
- **Download Video**

#### Node: Code in JavaScript
- **Type / role:** `Code` — transforms aggregated URLs into the merge API payload.
- **Behavior (interpreted):**
  - Reads first aggregated item: `const item = $input.first();`
  - Expects `item.json.video_url` to be an array of URLs.
  - Builds:
    - `video_urls: [{ video_url: url1 }, ...]`
    - Adds constant `id: "2323"`
  - Outputs `output` as a **stringified JSON**: `output: JSON.stringify(finalObject)`
- **Connections:** → **Merge Videos**
- **Edge cases / failures:**
  - If `video_url` isn’t an array, `.map()` fails.
  - Stringifying JSON is required because Merge Videos sends `jsonBody` from `{{ $json.output }}`; if you later change Merge Videos to accept an object, remove stringify.

#### Node: Merge Videos
- **Type / role:** `HTTP Request` — concatenates the videos.
- **Configuration choices:**
  - POST to `https://no-code-architects-toolkit-.../v1/video/concatenate`
  - Body: `specifyBody: json`, `jsonBody = {{ $json.output }}`
  - Auth: `httpHeaderAuth` (credential not shown; likely required depending on service)
  - Redirect options enabled
- **Output:** expected to include `response` holding a downloadable URL.
- **Edge cases / failures:**
  - If service expects JSON object but receives JSON string, it may fail unless server accepts raw string.
  - Endpoint availability / latency issues.

#### Node: Download Video
- **Type / role:** `HTTP Request` — downloads the merged video from URL.
- **Configuration choices:**
  - URL: `{{ $json.response }}`
  - Auth: uses **Vbee API** header auth (may be unnecessary if URL is public; may break if it rejects unknown headers).
- **Connections:** → **Upload media**
- **Edge cases:**
  - Not configured explicitly to “Download as binary” in the shown parameters; Blotato upload expects `useBinaryData: true`. Ensure this node is set to **Return Binary Data** (otherwise Upload media won’t have binary).

---

### Block 2.7 — Publish & Log
**Overview:** Uploads the final video to Blotato, publishes a Facebook post with caption + media, then logs the outcome to Google Sheets.

**Nodes involved:**
- **Upload media**
- **Create post Facebook**
- **Log Success**
- **Log Error**

#### Node: Upload media
- **Type / role:** `@blotato/n8n-nodes-blotato.blotato` — uploads video asset to Blotato media library.
- **Configuration choices:**
  - Resource: **media**
  - `useBinaryData: true` (expects binary from previous node)
- **Credentials:** `Blotato GiangxAI`
- **Output:** expected to include a hosted `url` used for posting.
- **Edge cases:**
  - If binary not present, upload fails.
  - Blotato API limits / supported formats.

#### Node: Create post Facebook
- **Type / role:** Blotato node — creates/publishes a Facebook post.
- **Configuration choices:**
  - Platform: **facebook**
  - Account selected: “Giang VT”
  - Page selected: ID `688227101036478`
  - Text: `{{ $('Product Image & Info Input').item.json['Mô tả'] }}`
    - **Risk:** field mismatch again (form label differs).
  - Media URL: `{{ $json.url }}` (from Upload media)
  - `onError: continueErrorOutput` enabled: node produces both success and error outputs.
- **Connections:**
  - Output 0 → **Log Success**
  - Output 1 → **Log Error**
- **Edge cases:**
  - Permission issues with Facebook page.
  - Post text expression may be empty due to wrong field key.

#### Node: Log Success
- **Type / role:** `Google Sheets` — append row for successful post.
- **Configuration choices:**
  - Operation: **append**
  - Spreadsheet: “Video Review” (ID `1gzD2jSl...`)
  - Sheet tab: “Trang tính1” (gid=0)
  - Writes column `Status = "Posted"` (other columns exist but not mapped here).
- **Credentials:** not shown in JSON, but required in n8n for Google Sheets.
- **Edge cases:**
  - Missing Google auth / revoked token.
  - Sheet columns mismatch with schema mapping.

#### Node: Log Error
- **Type / role:** `Google Sheets` — append row for failed post.
- **Configuration choices:** same spreadsheet/tab, sets `Status = "Error"`.
- **Edge cases:** same as Log Success.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Product Image & Info Input | Form Trigger | Collect image + description (entrypoint) | — | Analyze Product Image | ## Product input / Submit a product image and basic product information. The workflow will generate, merge, and publish an AI product review video automatically. |
| Analyze Product Image | Google Gemini (image analyze) | Extract visible product context from image | Product Image & Info Input | AI Prompt Agent | ## Image Prompt Generation / Analyze the product image to extract visual context and generate structured image prompts. These prompts are reused later to ensure visual consistency across generated scenes. |
| Google Gemini Chat Model | Gemini Chat Model | LLM for image prompt agent + parser autofix | — | AI Prompt Agent; Structured Output Parser1 | ## Image Prompt Generation / Analyze the product image to extract visual context and generate structured image prompts. These prompts are reused later to ensure visual consistency across generated scenes. |
| Structured Output Parser1 | Structured Output Parser | Enforce `{image_prompt}` JSON | Google Gemini Chat Model (as LLM) | AI Prompt Agent (as output parser) | ## Image Prompt Generation / Analyze the product image to extract visual context and generate structured image prompts. These prompts are reused later to ensure visual consistency across generated scenes. |
| AI Prompt Agent | LangChain Agent | Generate UGC image prompt JSON | Analyze Product Image | Generate Images | ## Image Prompt Generation / Analyze the product image to extract visual context and generate structured image prompts. These prompts are reused later to ensure visual consistency across generated scenes. |
| Generate Images | HTTP Request | Submit image generation job | AI Prompt Agent | Wait3 | ## Image Generation / Generate AI-based product images using the structured image prompts. These images are used as visual references for the video generation stage. |
| Wait3 | Wait | Delay before polling image job | Generate Images | Get Images | ## Image Generation / Generate AI-based product images using the structured image prompts. These images are used as visual references for the video generation stage. |
| Get Images | HTTP Request | Poll image generation result | Wait3 | Create Video Prompt | ## Image Generation / Generate AI-based product images using the structured image prompts. These images are used as visual references for the video generation stage. |
| Google Gemini Chat Model1 | Gemini Chat Model | LLM for video prompt agent + parser autofix | — | Create Video Prompt; Video Prompt Parser | ## Video Prompt Generation / Generate structured video prompts for each scene based on the product review script. Each prompt represents a short segment of the final product review video. |
| Video Prompt Parser | Structured Output Parser | Enforce structured JSON for video prompts | Google Gemini Chat Model1 (as LLM) | Create Video Prompt (as output parser) | ## Video Prompt Generation / Generate structured video prompts for each scene based on the product review script. Each prompt represents a short segment of the final product review video. |
| Create Video Prompt | LangChain Agent | Create title/description + 3 continuous scene prompts | Get Images | Split Video Prompts | ## Video Prompt Generation / Generate structured video prompts for each scene based on the product review script. Each prompt represents a short segment of the final product review video. |
| Split Video Prompts | Split Out | Split prompts into per-scene items | Create Video Prompt | Loop Video Prompts | ## Video Generation / Generate short video scenes from the video prompts and monitor rendering status. Each scene is processed independently before being merged into the final video. |
| Loop Video Prompts | Split In Batches | Iterate scene prompts; trigger aggregation at end | Split Video Prompts; If1 (true) | Generate Videos; Aggregate Videos | ## Video Generation / Generate short video scenes from the video prompts and monitor rendering status. Each scene is processed independently before being merged into the final video. |
| Generate Videos | HTTP Request | Submit Veo video generation job per scene | Loop Video Prompts | Wait1 | ## Video Generation / Generate short video scenes from the video prompts and monitor rendering status. Each scene is processed independently before being merged into the final video. |
| Wait1 | Wait | Delay before polling video job | Generate Videos; If1 (false) | Get Videos | ## Video Generation / Generate short video scenes from the video prompts and monitor rendering status. Each scene is processed independently before being merged into the final video. |
| Get Videos | HTTP Request | Poll video generation result | Wait1 | If1 | ## Video Generation / Generate short video scenes from the video prompts and monitor rendering status. Each scene is processed independently before being merged into the final video. |
| If1 | If | Check render completion (`status == 2`) | Get Videos | Loop Video Prompts (true); Wait1 (false) | ## Video Generation / Generate short video scenes from the video prompts and monitor rendering status. Each scene is processed independently before being merged into the final video. |
| Aggregate Videos | Aggregate | Collect final clip URLs | Loop Video Prompts (done output) | Code in JavaScript | ## Video Merge / Merge all generated video scenes into a single final product review video. This step prepares the video for publishing without manual editing. |
| Code in JavaScript | Code | Build merge payload from URLs | Aggregate Videos | Merge Videos | ## Video Merge / Merge all generated video scenes into a single final product review video. This step prepares the video for publishing without manual editing. |
| Merge Videos | HTTP Request | Concatenate clips via external service | Code in JavaScript | (none connected) | ## Video Merge / Merge all generated video scenes into a single final product review video. This step prepares the video for publishing without manual editing. |
| Download Video | HTTP Request | Download merged video file | Merge Videos | Upload media | ## Publish & Log / Publish the final video to social media platforms. Publishing results, including success and error states, are logged to Google Sheets for tracking. |
| Upload media | Blotato | Upload video file to Blotato | Download Video | Create post Facebook | ## Publish & Log / Publish the final video to social media platforms. Publishing results, including success and error states, are logged to Google Sheets for tracking. |
| Create post Facebook | Blotato | Publish Facebook post with uploaded media | Upload media | Log Success; Log Error | ## Publish & Log / Publish the final video to social media platforms. Publishing results, including success and error states, are logged to Google Sheets for tracking. |
| Log Success | Google Sheets | Append “Posted” row | Create post Facebook (success output) | — | ## Publish & Log / Publish the final video to social media platforms. Publishing results, including success and error states, are logged to Google Sheets for tracking. |
| Log Error | Google Sheets | Append “Error” row | Create post Facebook (error output) | — | ## Publish & Log / Publish the final video to social media platforms. Publishing results, including success and error states, are logged to Google Sheets for tracking. |
| Sticky Note | Sticky Note | Comment | — | — | ## Video Generation / Generate short video scenes from the video prompts and monitor rendering status. Each scene is processed independently before being merged into the final video. |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Video Merge / Merge all generated video scenes into a single final product review video. This step prepares the video for publishing without manual editing. |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Video Prompt Generation / Generate structured video prompts for each scene based on the product review script. Each prompt represents a short segment of the final product review video. |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Image Generation / Generate AI-based product images using the structured image prompts. These images are used as visual references for the video generation stage. |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Image Prompt Generation / Analyze the product image to extract visual context and generate structured image prompts. These prompts are reused later to ensure visual consistency across generated scenes. |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Publish & Log / Publish the final video to social media platforms. Publishing results, including success and error states, are logged to Google Sheets for tracking. |
| Sticky Note6 | Sticky Note | Comment | — | — | ## Product input / Submit a product image and basic product information. The workflow will generate, merge, and publish an AI product review video automatically. |
| Sticky Note7 | Sticky Note | Comment / setup guide | — | — | ## 🛠️ Workflow Setup Guide / Author: [GiangxAI](https://www.youtube.com/@giangxai.official) / You can track publishing results using the sample [Google Sheet](https://docs.google.com/spreadsheets/d/1gzD2jSlznYENE_IwwW0o4Qe0ZbQTnkj3wi8dKixmEv0/edit?usp=sharing) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger**
   1. Add node: **Form Trigger**
   2. Title: `UCG VIDEO`
   3. Add fields:
      - File field label: `Image`
      - Text field label: `Video review description`
   4. Ensure the file field produces binary property named **`Image`**.

2. **Analyze the uploaded image with Gemini**
   1. Add node: **Google Gemini** (LangChain) → resource **Image**, operation **Analyze**
   2. Input type: **Binary**
   3. Binary property: `Image`
   4. Model: `models/gemini-3-pro-preview`
   5. Configure **Google Gemini/PaLM credential**.

3. **Create image prompt agent (structured JSON)**
   1. Add node: **Google Gemini Chat Model** with model `models/gemini-3-pro-preview`
   2. Add node: **Structured Output Parser**
      - Enable **Auto-fix**
      - Schema example: `{ "image_prompt": "string" }`
   3. Add node: **AI Agent**
      - System message: (UGC image prompt builder rules; enforce JSON-only `{image_prompt}`)
      - User message should reference:
        - Form text field (use the exact form key, e.g. `{{ $('Product Image & Info Input').item.json['Video review description'] }}`)
        - Gemini image analysis `{{ $json.content.parts[0].text }}`
      - Attach the **Chat Model** as `ai_languageModel`
      - Attach the **Structured Output Parser** as `ai_outputParser`
   4. Connect: Form Trigger → Analyze Image → AI Agent

4. **Generate image via external API and poll**
   1. Add **HTTP Request** “Generate Images”
      - POST `https://api.geminigen.ai/uapi/v1/generate_image`
      - Content type: **multipart/form-data**
      - Fields: `prompt={{$json.output.image_prompt}}`, `model=imagen-pro`, `aspect_ratio=9:16`, `style=Photorealistic`
      - Auth: **Header Auth** credential (API key for provider)
   2. Add **Wait** node (15s)
   3. Add **HTTP Request** “Get Images”
      - GET `https://api.geminigen.ai/uapi/v1/history/{{ $json.uuid }}`
      - Same header auth credential
   4. Connect: AI Agent → Generate Images → Wait → Get Images

5. **Generate structured 3-scene video prompts**
   1. Add **Google Gemini Chat Model** (second instance) for video prompting.
   2. Add **Structured Output Parser** for video JSON.
      - Make schema match your agent output. Recommended:
        - Either output `{ scenes: { scene_1:{video_prompt}, ... } }` and split `output.scenes`
        - Or keep `scene_1/2/3` at top level and use a Code node to convert to an array before looping.
   3. Add **AI Agent** “Create Video Prompt”
      - System message: continuity rules + Vietnamese dialogue + JSON-only output.
      - User text: pass form description (ensure correct key).
   4. Connect: Get Images → Create Video Prompt

6. **Split prompts into per-scene items**
   1. Add **Split Out** node
      - Field to split: whichever structure you chose (e.g. `output.scenes`)
   2. Connect: Create Video Prompt → Split Out

7. **Loop scenes, generate video, poll until done**
   1. Add **Split In Batches** node (Loop)
   2. Add **HTTP Request** “Generate Videos”
      - POST `https://api.geminigen.ai/uapi/v1/video-gen/veo`
      - Multipart fields:
        - `prompt={{$json.video_prompt}}`
        - `model=veo-3-fast`, `resolution=720p`, `aspect_ratio=9:16`
        - `file_urls={{ $('Get Images').item.json.generated_image[0].image_url }}`
      - Header auth credential
   3. Add **Wait** (10s)
   4. Add **HTTP Request** “Get Videos”
      - GET `https://api.geminigen.ai/uapi/v1/history/{{ $json.uuid }}`
      - **Set the same header auth credential**
   5. Add **If** node
      - Condition: `{{$json.status}}` equals number `2`
      - True: back to **Split In Batches** to continue
      - False: to **Wait** to poll again
   6. Wire:
      - Split Out → Split In Batches
      - Split In Batches (item) → Generate Videos → Wait → Get Videos → If
      - If false → Wait (poll loop)
      - If true → Split In Batches (next item)

8. **Aggregate clip URLs**
   1. Add **Aggregate** node
      - Aggregate field: `generated_video[0].video_url`
   2. Connect: Split In Batches “done” output → Aggregate

9. **Build merge payload + merge**
   1. Add **Code** node to transform aggregated array into the merge service format:
      - `video_urls: [{video_url: ...}, ...]`
   2. Add **HTTP Request** “Merge Videos”
      - POST your concatenate endpoint
      - Send JSON body (either object or string, depending on API expectation)
      - Configure auth if required
   3. Connect: Aggregate → Code → Merge Videos

10. **Download merged video as binary**
   1. Add **HTTP Request** “Download Video”
      - URL: `{{$json.response}}` (or correct field)
      - Enable **Return Binary Data** (critical)
      - Set binary property name (e.g. `data`)
   2. Connect: Merge Videos → Download Video

11. **Upload to Blotato and publish to Facebook**
   1. Add **Blotato** node “Upload media”
      - Resource: `media`
      - Use binary data: true (select the binary property from Download Video)
   2. Add **Blotato** node “Create post Facebook”
      - Platform: facebook
      - Select account + page
      - Text: map from form field (ensure correct key)
      - Media URL: `{{$json.url}}` (from upload)
      - Enable “Continue on Fail” / error output if you want split success/error logging
   3. Connect: Download Video → Upload media → Create post Facebook

12. **Log results to Google Sheets**
   1. Add **Google Sheets** “Log Success”
      - Operation: Append
      - Spreadsheet: your logging sheet
      - Set `Status = Posted` (map other columns if needed: title/description/link)
   2. Add **Google Sheets** “Log Error”
      - Append with `Status = Error`
   3. Connect Create post Facebook:
      - Success output → Log Success
      - Error output → Log Error
   4. Configure **Google Sheets OAuth2** credentials.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: GiangxAI | https://www.youtube.com/@giangxai.official |
| Sample Google Sheet for tracking publishing results | https://docs.google.com/spreadsheets/d/1gzD2jSlznYENE_IwwW0o4Qe0ZbQTnkj3wi8dKixmEv0/edit?usp=sharing |
| Setup guide highlights: form input, add Gemini/OpenRouter creds, configure Veo3 provider keys, verify merge endpoint, add Blotato creds, connect Google Sheets | From the workflow’s setup sticky note |

**High-impact implementation warnings (from the JSON as provided):**
- Several expressions reference Vietnamese field names (`'Mô tả video review'`, `'Mô tả'`) that do **not** match the shown Form field labels; correct these to your actual form keys.
- “Get Videos” HTTP node is configured for header auth but does **not** show credentials attached—add the provider credential.
- Video prompt parser schema example does not match the agent’s required JSON structure; align them to avoid split/loop failures.
- Ensure “Download Video” returns **binary data**, otherwise Blotato media upload will fail.

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.