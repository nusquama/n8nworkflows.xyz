Create TikTok Shop product videos with Seedance 2.5, OpenAI, and SocialRobot

https://n8nworkflows.xyz/workflows/create-tiktok-shop-product-videos-with-seedance-2-5--openai--and-socialrobot-20932


# Create TikTok Shop product videos with Seedance 2.5, OpenAI, and SocialRobot

### 1. Workflow Overview

This workflow automates the creation of 15-second vertical TikTok Shop product videos using generative AI and publishes them directly to TikTok as inbox drafts. Target use cases include e-commerce content generation, automated affiliate marketing, and rapid social media asset production. 

The workflow groups its execution into six distinct logical blocks:
- **1.1 Input Reception & Configuration:** Ingests product details and optional reference photos via an n8n Form trigger, establishes global video rendering parameters, and evaluates whether image uploads require staging.
- **1.2 Media Preparation:** Iterates through user-uploaded product photos, securely transfers them to Replicate file storage, and aggregates public URLs for reference-based video generation.
- **1.3 AI Script & Prompt Generation:** Utilizes OpenAI to write timed production scripts, multi-beat video prompts, voiceover text, and platform-optimized captions constrained strictly by verified product facts.
- **1.4 Video Generation & Polling:** Submits the rendering job to ByteDance Seedance 2.5 via Replicate, polling asynchronously at configured intervals until completion, failure, or timeout.
- **1.5 Validation & Quality Control:** Downloads the generated MP4/MOV artifact, programmatically inspects headers for valid duration and file size constraints, and routes invalid media to error handling.
- **1.6 Distribution & Error Handling:** Delivers the verified asset to TikTok as an inbox draft via the SocialRobot community node (with optional cross-posting to Instagram Reels) or catches failures to broadcast descriptive error messages.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration

**Overview:** 
This block initializes the execution flow by capturing user inputs from a web form, setting global configuration variables for the video generation pipeline, and branching based on the presence of uploaded imagery.

**Nodes Involved:**
- `Product Video Form`
- `Set Video Settings`
- `Has Product Photos?`

**Node Details:**

- **Product Video Form**
  - **Type & Role:** `n8n-nodes-base.formTrigger` — Webhook-based form trigger that halts workflow execution until user submission.
  - **Configuration Choices:** Configured to collect fields: `Product Name` (string, required), `Selling Points` (textarea, required), `Offer or Call to Action` (string, optional), `Video Language` (dropdown: English/Spanish, required), and `Product Photos` (multiple files, optional, limited to `.jpg`, `.jpeg`, `.png`, `.webp`). Response mode is set to respond immediately on receipt with a custom submission confirmation text.
  - **Key Expressions:** None (Entry point).
  - **Connections:** Input: None (Trigger); Output: `Set Video Settings`.
  - **Version Requirements:** Version 2.3.
  - **Edge Cases & Failure Types:** Form validation errors if required fields are missing; webhook timeout if downstream processing hangs before responding.
  - **Sub-workflow Reference:** None.

- **Set Video Settings**
  - **Type & Role:** `n8n-nodes-base.set` — Parameter assignment node that acts as a central configuration store for rendering, cost estimation, and validation rules.
  - **Configuration Choices:** Assigns global constants including `model` (`bytedance/seedance-2.5`), `resolution` (`720p`), `duration_seconds` (`15`), `aspect_ratio` (`9:16`), `generate_audio` (`true`), `price_per_second_usd` (`0.2312`), `max_photos` (`4`), `poll_every_seconds` (`20`), `timeout_minutes` (`30`), `max_file_mb` (`50`), `min_seconds` (`3`), `max_seconds` (`60`), `brand_voice`, and `extra_hashtags`.
  - **Key Expressions:** Static assignments.
  - **Connections:** Input: `Product Video Form`; Output: `Has Product Photos?`.
  - **Version Requirements:** Version 3.4.
  - **Edge Cases & Failure Types:** Type mismatch errors if downstream nodes expect numbers and configuration returns strings.
  - **Sub-workflow Reference:** None.

- **Has Product Photos?**
  - **Type & Role:** `n8n-nodes-base.if` — Conditional branching node that checks whether the user submitted binary image attachments.
  - **Configuration Choices:** Evaluates if the filtered count of binary items starting with mime type `image/` is greater than 0 using loose type validation.
  - **Key Expressions:** `={{ Object.values($('Product Video Form').first().binary || {}).filter(b => String(b.mimeType || '').startsWith('image/')).length }}`
  - **Connections:** Input: `Set Video Settings`; Output (True): `Prepare Product Photos`; Output (False): `Collect Photo URLs`.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases & Failure Types:** Missing binary object properties if form data is malformed.
  - **Sub-workflow Reference:** None.

---

#### Block 1.2: Media Preparation

**Overview:** 
This block handles auxiliary product images by normalizing binary attachments, uploading them to Replicate’s transient file storage, and compiling their secure URLs into a unified array for AI model reference.

**Nodes Involved:**
- `Prepare Product Photos`
- `Upload Photo to Replicate`
- `Collect Photo URLs`

**Node Details:**

- **Prepare Product Photos**
  - **Type & Role:** `n8n-nodes-base.code` — JavaScript execution node that maps binary form attachments into individual items for batch processing.
  - **Configuration Choices:** Reads binary fields from the form, filters for valid image types, slices array by `max_photos` limit, and outputs structured items containing file names and binary data.
  - **Key Expressions:** References `$('Set Video Settings').first().json.max_photos` and `$('Product Video Form').first()`.
  - **Connections:** Input: `Has Product Photos?` (True); Output: `Upload Photo to Replicate`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Throws execution errors if binary property indices are out of bounds.
  - **Sub-workflow Reference:** None.

- **Upload Photo to Replicate**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — HTTP client node executing multipart form uploads to Replicate's storage API.
  - **Configuration Choices:** Method `POST`, URL `https://api.replicate.com/v1/files`, content type `multipart-form-data`, input data field `photo`. Retry on fail enabled (3 attempts with 3000ms delay).
  - **Key Expressions:** None.
  - **Connections:** Input: `Prepare Product Photos`; Output: `Collect Photo URLs`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** HTTP 401/403 authentication failures, network timeouts, or rate limits. Requires valid Bearer Token.
  - **Sub-workflow Reference:** None.

- **Collect Photo URLs**
  - **Type & Role:** `n8n-nodes-base.code` — JavaScript execution node that aggregates uploaded asset URLs or handles zero-photo fallback states.
  - **Configuration Choices:** Safely checks if `Upload Photo to Replicate` executed; extracts `urls.get` properties, filtering out empty values. Returns an object containing `photo_urls` array and `photo_count`.
  - **Key Expressions:** Uses `try/catch` block wrapping `$('Upload Photo to Replicate').isExecuted`.
  - **Connections:** Input: `Upload Photo to Replicate` (and `Has Product Photos?` False branch); Output: `Write Script and Prompt`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Undefined property errors if Replicate's API response structure changes.
  - **Sub-workflow Reference:** None.

---

#### Block 1.3: AI Script & Prompt Generation

**Overview:** 
This block connects to OpenAI to generate a production-ready script, timed visual prompts, voiceover text, and TikTok captions based strictly on verified product selling points.

**Nodes Involved:**
- `Write Script and Prompt`
- `OpenAI GPT-6 Luna`
- `Script Format`

**Node Details:**

- **Write Script & Prompt**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.chainLlm` — LangChain LLM Chain node that orchestrates structured prompt generation.
  - **Configuration Choices:** Prompt defines strict rules: zero claims beyond provided selling points, no fake testimonials, no real people, and mandatory integration of reference photo tags (`[Image1]` etc.).
  - **Key Expressions:** Dynamic template expressions pulling product name, selling points, offer details, language, brand voice, reference photo count, and duration from preceding nodes.
  - **Connections:** Input: `Collect Photo URLs`; Output: `Build Seedance Request`; Connected to `OpenAI GPT-6 Luna` and `Script Format`.
  - **Version Requirements:** Version 1.7.
  - **Edge Cases & Failure Types:** LLM hallucination, token limit exhaustion, or refusal to generate due to safety filters.
  - **Sub-workflow Reference:** None.

- **OpenAI GPT-6 Luna**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Language model model provider integration.
  - **Configuration Choices:** Configured to use model `gpt-6-luna`.
  - **Key Expressions:** None.
  - **Connections:** Connected to `Write Script and Prompt` (AI Language Model link).
  - **Version Requirements:** Version 1.2.
  - **Edge Cases & Failure Types:** API authentication errors, insufficient quota, or upstream OpenAI outages.
  - **Sub-workflow Reference:** None.

- **Script Format**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Structured output parser enforcing JSON adherence.
  - **Configuration Choices:** Enforces schema returning `seedance_prompt`, `hook`, `voiceover`, and `caption`.
  - **Key Expressions:** JSON schema example configuration.
  - **Connections:** Connected to `Write Script and Prompt` (AI Output Parser link).
  - **Version Requirements:** Version 1.3.
  - **Edge Cases & Failure Types:** Parsing failures if LLM response violates the JSON schema structure.
  - **Sub-workflow Reference:** None.

---

#### Block 1.4: Video Generation & Polling

**Overview:** 
This block builds the payload for ByteDance Seedance 2.5, submits the prediction job to Replicate, and implements an asynchronous polling loop to monitor job progress.

**Nodes Involved:**
- `Build Seedance Request`
- `Submit Seedance Job`
- `Wait Between Checks`
- `Check Seedance Job`
- `Route by Job Status`
- `Cancel Timed Out Job`

**Node Details:**

- **Build Seedance Request**
  - **Type & Role:** `n8n-nodes-base.code` — JavaScript execution node structuring the API payload for Replicate.
  - **Configuration Choices:** Extracts prompt, duration, resolution, aspect ratio, audio generation flags, and attaches `reference_images` array if photos exist.
  - **Key Expressions:** Reads JSON output from `Write Script and Prompt` and `Collect Photo URLs`.
  - **Connections:** Input: `Write Script and Prompt`; Output: `Submit Seedance Job`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Type coercion issues with boolean or numeric input parameters.
  - **Sub-workflow Reference:** None.

- **Submit Seedance Job**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — HTTP client node initiating asynchronous video rendering.
  - **Configuration Choices:** Method `POST`, URL dynamically evaluates model endpoint, sends JSON body stringified from preceding step. Authenticated via HTTP Bearer Auth.
  - **Key Expressions:** `=https://api.replicate.com/v1/models/{{ $json.model }}/predictions` and `={{ JSON.stringify($json.body) }}`.
  - **Connections:** Input: `Build Seedance Request`; Output: `Wait Between Checks`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** API rejection due to invalid input schemas or concurrency limits.
  - **Sub-workflow Reference:** None.

- **Wait Between Checks**
  - **Type & Role:** `n8n-nodes-base.wait` — Pause execution node to respect API rate limits during video generation.
  - **Configuration Choices:** Unit set to seconds, amount dynamically read from video settings (`poll_every_seconds`).
  - **Key Expressions:** `={{ $('Set Video Settings').first().json.poll_every_seconds }}`.
  - **Connections:** Input: `Submit Seedance Job` and `Route by Job Status` (Still Running branch); Output: `Check Seedance Job`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** Webhook registration failures on self-hosted instances with improper public URL configurations.
  - **Sub-workflow Reference:** None.

- **Check Seedance Job**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — HTTP client node polling Replicate for prediction status.
  - **Configuration Choices:** Method `GET`, URL queries prediction ID. Retry on fail enabled (3 attempts with 5000ms delay). Authenticated via HTTP Bearer Auth.
  - **Key Expressions:** `=https://api.replicate.com/v1/predictions/{{ $json.id }}`.
  - **Connections:** Input: `Wait Between Checks`; Output: `Route by Job Status`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Temporary network drops during status checks.
  - **Sub-workflow Reference:** None.

- **Route by Job Status**
  - **Type & Role:** `n8n-nodes-base.switch` — Multi-path routing node evaluating prediction state and elapsed time.
  - **Configuration Choices:** Four evaluation rules:
    1. *Succeeded:* `status === 'succeeded'`
    2. *Failed:* Checks if status matches `['failed', 'canceled', 'aborted']`
    3. *Timed Out:* Calculates minutes elapsed since `created_at` against `timeout_minutes` setting.
    4. *Still Running:* Fallback output.
  - **Key Expressions:** Complex conditional expressions validating date timestamps and array inclusions.
  - **Connections:** Input: `Check Seedance Job`; Output (Succeeded): `Download Video`; Output (Failed): `Describe Failure`; Output (Timed Out): `Cancel Timed Out Job`; Output (Still Running): `Wait Between Checks`.
  - **Version Requirements:** Version 3.2.
  - **Edge Cases & Failure Types:** Unhandled status strings returned by upstream API changes.
  - **Sub-workflow Reference:** None.

- **Cancel Timed Out Job**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — HTTP client node aborting stuck Replicate jobs.
  - **Configuration Choices:** Method `POST`, URL dynamically populated from Replicate's cancellation URL property. Authenticated via HTTP Bearer Auth.
  - **Key Expressions:** `={{ $json.urls.cancel }}`.
  - **Connections:** Input: `Route by Job Status` (Timed Out); Output: `Describe Failure`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Failure to reach Replicate API during cancellation.
  - **Sub-workflow Reference:** None.

---

#### Block 1.5: Validation & Quality Control

**Overview:** 
This block downloads the rendered video file from Replicate storage, inspects container headers to verify exact duration and file size limits, and enforces quality gates before social publishing.

**Nodes Involved:**
- `Download Video`
- `Check Size and Duration`
- `Video Passes Checks`

**Node Details:**

- **Download Video**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — HTTP client node fetching the binary video payload.
  - **Configuration Choices:** Response format set to file, outputting to binary property `data`. Retry on fail enabled (3 attempts).
  - **Key Expressions:** `={{ Array.isArray($json.output) ? $json.output[0] : $json.output }}`.
  - **Connections:** Input: `Route by Job Status` (Succeeded); Output: `Check Size and Duration`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Broken CDN links or expired Replicate output URLs.
  - **Sub-workflow Reference:** None.

- **Check Size and Duration**
  - **Type & Role:** `n8n-nodes-base.code` — JavaScript execution node analyzing binary MP4/MOV header structures (`mvhd` box parsing).
  - **Configuration Choices:** Computes file size in megabytes, parses binary buffer offsets to extract media scale and duration units, validates against MIME types, maximum file size, and minimum/maximum allowed seconds.
  - **Key Expressions:** Utilizes helper `this.helpers.getBinaryDataBuffer(0, 'data')`.
  - **Connections:** Input: `Download Video`; Output: `Video Passes Checks`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Corrupted video headers failing byte-offset parsing.
  - **Sub-workflow Reference:** None.

- **Video Passes Checks**
  - **Type & Role:** `n8n-nodes-base.if` — Conditional branching node checking validation success flags.
  - **Configuration Choices:** Evaluates boolean property `ok`.
  - **Key Expressions:** `={{ $json.ok }}`.
  - **Connections:** Input: `Check Size and Duration`; Output (True): `Save TikTok Draft` and `Optional Instagram Reel`; Output (False): `Describe Failure`.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases & Failure Types:** Uncaught validation arrays.
  - **Sub-workflow Reference:** None.

---

#### Block 1.6: Distribution & Error Handling

**Overview:** 
This block uploads the verified video to TikTok as an inbox draft via SocialRobot, optionally cross-posts to Instagram Reels, summarizes execution metadata, or catches failures to trigger error propagation.

**Nodes Involved:**
- `Save TikTok Draft`
- `Summarize Draft`
- `Optional Instagram Reel`
- `Describe Failure`
- `Stop With Error`

**Node Details:**

- **Save TikTok Draft**
  - **Type & Role:** `@socialrobot-io/n8n-nodes-socialrobot.socialRobot` — Community node integration for TikTok automation.
  - **Configuration Choices:** Resource `tiktok`, operation `create`, post mode `UPLOAD` (Send to Inbox / Draft), publish mode `NOW`, media type `VIDEO`, source binary property `data`. Error handling configured to continue on error output.
  - **Key Expressions:** `={{ $('Write Script and Prompt').first().json.output.caption }}`.
  - **Connections:** Input: `Video Passes Checks` (True); Output Main: `Summarize Draft`; Output Error: `Describe Failure`.
  - **Version Requirements:** Community node version 3.1.1+.
  - **Edge Cases & Failure Types:** Invalid SocialRobot API keys, disconnected TikTok accounts, or API rate limiting.
  - **Sub-workflow Reference:** None.

- **Summarize Draft**
  - **Type & Role:** `n8n-nodes-base.set` — Final transformation node compiling run metadata.
  - **Configuration Choices:** Assigns summary metrics including TikTok post ID, status, product name, language, video duration, file size, estimated cost, status notes, and next-step instructions.
  - **Key Expressions:** Dynamic property mapping referencing form inputs, video analysis nodes, and settings.
  - **Connections:** Input: `Save TikTok Draft`; Output: None (Terminal node).
  - **Version Requirements:** Version 3.4.
  - **Edge Cases & Failure Types:** None.
  - **Sub-workflow Reference:** None.

- **Optional Instagram Reel**
  - **Type & Role:** `@socialrobot-io/n8n-nodes-socialrobot.socialRobot` — Community node integration for Instagram automation.
  - **Configuration Choices:** Resource `instagram`, operation `create`, media type `VIDEO`, source binary property `data`, publish mode `NOW`, `isReel` true, `isAiGenerated` true. Node is disabled by default.
  - **Key Expressions:** Caption mapped from OpenAI output.
  - **Connections:** Input: `Video Passes Checks` (True); Output: None.
  - **Version Requirements:** Community node version 3.1.1+.
  - **Edge Cases & Failure Types:** Instagram API restrictions or missing account selections.
  - **Sub-workflow Reference:** None.

- **Describe Failure**
  - **Type & Role:** `n8n-nodes-base.code` — JavaScript execution node formatting readable error diagnostics.
  - **Configuration Choices:** Inspects execution state across cancellation jobs, job failure objects, SocialRobot errors, and validation problem arrays to construct a consolidated error message.
  - **Key Expressions:** Evaluates `.isExecuted` flags on upstream nodes.
  - **Connections:** Input: `Route by Job Status` (Failed), `Cancel Timed Out Job`, `Video Passes Checks` (False), and `Save TikTok Draft` (Error branch); Output: `Stop With Error`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Missing error properties in unexpected upstream exceptions.
  - **Sub-workflow Reference:** None.

- **Stop With Error**
  - **Type & Role:** `n8n-nodes-base.stopAndError` — Workflow termination node that halts execution and reports failure.
  - **Configuration Choices:** Sets custom error message based on formatted failure description.
  - **Key Expressions:** `={{ $json.message }}`.
  - **Connections:** Input: `Describe Failure`; Output: None (Terminal node).
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failure Types:** None.
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Template Overview | n8n-nodes-base.stickyNote | Documentation overview and setup guide | None | None | ## Create TikTok Shop product videos with Seedance 2.5<br><br>Turn product photos and a few true selling points into a 15-second vertical AI product video, then send it to your TikTok inbox as a draft with SocialRobot.<br><br>### How it works<br>1. A form collects 0-4 optional product photos, selling points and the language (English or Spanish).<br>2. Any photos go to Replicate (skipped when none are uploaded). OpenAI writes a timed script, the Seedance prompt and the caption.<br>3. Seedance 2.5 renders a 9:16 720p video with sound. The workflow polls until it is done, failed or timed out.<br>4. File size and duration are checked, then SocialRobot sends the video to your TikTok inbox.<br>5. Any failure stops the run with a readable message.<br><br>### Setup steps<br>- [ ] Self-hosted n8n with the SocialRobot community node 3.1.1+ (npm latest 3.1.2) and your SocialRobot API key<br>- [ ] Connect TikTok in SocialRobot and pick the account in **Save TikTok Draft**<br>- [ ] Add a Replicate API token (Bearer Auth) to the three Replicate nodes<br>- [ ] Add an OpenAI credential<br>- [ ] Check model, length and limits in **Set Video Settings**<br><br>### Customization<br>- Switch to Seedance 2.0 fast or another provider in the video section.<br>- Enable **Optional Instagram Reel** (disabled by default) and pick an account to cross-post.<br>- SocialRobot does not publish to YouTube.<br><br>About $3.47 per 15-second video on Replicate at 720p (list price, Oct 2026). |
| Section Product Brief | n8n-nodes-base.stickyNote | Section grouping note for form and prep | None | None | ## Collect the product brief<br>The form takes 0-4 optional product photos, selling points and the language. **Has Product Photos?** sends photos to Replicate file storage so Seedance can use them as references; with no photos it skips straight to a text-only prompt. |
| Section Script | n8n-nodes-base.stickyNote | Section grouping note for OpenAI script generation | None | None | ## Write the script<br>OpenAI writes a timed hook, demo and call to action, the Seedance prompt and the caption. It only uses your selling points: no fake reviews, no real people. |
| Section Seedance | n8n-nodes-base.stickyNote | Section grouping note for Seedance rendering | None | None | ## Generate with Seedance 2.5<br>Submits the Replicate job, checks every 20 s and gives up after 30 min. For Seedance 2.0 fast change `model` in Set Video Settings. For fal, WaveSpeed or BytePlus swap the URLs (see description). |
| Section Checks | n8n-nodes-base.stickyNote | Section grouping note for validation | None | None | ## Check the video<br>Downloads the result (Replicate deletes outputs after an hour), then checks file size and real MP4 duration against your limits. |
| Section TikTok Draft | n8n-nodes-base.stickyNote | Section grouping note for publishing | None | None | ## Save the TikTok draft<br>Publish Now with Post Mode Send to Inbox (Draft) puts the video in your TikTok inbox. Nothing goes public until you post it in the app. Optional: enable the Instagram Reel node (already connected, disabled by default) and pick an account. |
| Section Errors | n8n-nodes-base.stickyNote | Section grouping note for error handling | None | None | ## Handle errors<br>Failed or timed-out jobs, files that fail the checks and SocialRobot errors end here with a readable message. Add an error workflow in the settings for alerts. |
| Warning Finish in TikTok | n8n-nodes-base.stickyNote | Publishing guidance note | None | None | ## Finish in the TikTok app<br>If you enable the Instagram Reel node, reels are labeled as AI-generated automatically. The SocialRobot node does not send the TikTok AI label for inbox drafts. In TikTok, open the inbox notification, turn on the 'AI-generated content' label, add sound or a product link, then post. |
| Product Video Form | n8n-nodes-base.formTrigger | Ingests product brief and optional images | None | Set Video Settings | |
| Set Video Settings | n8n-nodes-base.set | Assigns global configuration constants | Product Video Form | Has Product Photos? | |
| Prepare Product Photos | n8n-nodes-base.code | Prepares binary form files for upload | Has Product Photos? | Upload Photo to Replicate | |
| Upload Photo to Replicate | n8n-nodes-base.httpRequest | Uploads photos to Replicate file storage | Prepare Product Photos | Collect Photo URLs | |
| Collect Photo URLs | n8n-nodes-base.code | Aggregates file URLs or handles zero-photo state | Upload Photo to Replicate, Has Product Photos? | Write Script and Prompt | |
| Write Script and Prompt | @n8n/n8n-nodes-langchain.chainLlm | Orchestrates AI script and prompt generation | Collect Photo URLs | Build Seedance Request | |
| OpenAI GPT-6 Luna | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM chat capabilities | None | Write Script and Prompt | |
| Script Format | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON output structure for script | None | Write Script and Prompt | |
| Build Seedance Request | n8n-nodes-base.code | Constructs Replicate API input payload | Write Script and Prompt | Submit Seedance Job | |
| Submit Seedance Job | n8n-nodes-base.httpRequest | Submits rendering job to Replicate | Build Seedance Request | Wait Between Checks | |
| Wait Between Checks | n8n-nodes-base.wait | Pauses execution between polling requests | Submit Seedance Job, Route by Job Status | Check Seedance Job | |
| Check Seedance Job | n8n-nodes-base.httpRequest | Polls prediction status from Replicate | Wait Between Checks | Route by Job Status | |
| Route by Job Status | n8n-nodes-base.switch | Routes execution based on job state and timer | Check Seedance Job | Download Video, Describe Failure, Cancel Timed Out Job, Wait Between Checks | |
| Download Video | n8n-nodes-base.httpRequest | Downloads rendered video binary from Replicate | Route by Job Status | Check Size and Duration | |
| Check Size and Duration | n8n-nodes-base.code | Inspects MP4/MOV headers for size and duration | Download Video | Video Passes Checks | |
| Video Passes Checks | n8n-nodes-base.if | Evaluates validation success flag | Check Size and Duration | Save TikTok Draft, Optional Instagram Reel, Describe Failure | |
| Save TikTok Draft | @socialrobot-io/n8n-nodes-socialrobot.socialRobot | Uploads video to TikTok inbox as draft | Video Passes Checks | Summarize Draft, Describe Failure | |
| Summarize Draft | n8n-nodes-base.set | Compiles final run summary metadata | Save TikTok Draft | None | |
| Optional Instagram Reel | @socialrobot-io/n8n-nodes-socialrobot.socialRobot | Cross-posts video as Instagram Reel | Video Passes Checks | None | |
| Cancel Timed Out Job | n8n-nodes-base.httpRequest | Cancels timed-out Replicate jobs | Route by Job Status | Describe Failure | |
| Describe Failure | n8n-nodes-base.code | Formats readable error messages | Route by Job Status, Cancel Timed Out Job, Video Passes Checks, Save TikTok Draft | Stop With Error | |
| Stop With Error | n8n-nodes-base.stopAndError | Terminates execution with error alert | Describe Failure | None | |
| Has Product Photos? | n8n-nodes-base.if | Checks if user uploaded images | Set Video Settings | Prepare Product Photos, Collect Photo URLs | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**
   - Create a new **Form Trigger** node named `Product Video Form`.
   - Set form title to `Create TikTok Shop product video` and description to `Upload up to 4 clear product photos (optional) and list a few true selling points. The finished 15-second video is sent to your TikTok inbox as a draft.`
   - Add form fields: `Product Name` (Text, required), `Selling Points` (Textarea, required), `Offer or Call to Action` (Text, optional), `Video Language` (Dropdown with options `English` and `Spanish`, required), and `Product Photos` (File upload, multiple files enabled, optional, accepted types `.jpg, .jpeg, .png, .wp`). Set response mode to *On Received*.

2. **Create Settings Set Node:**
   - Create a **Set** node named `Set Video Settings`.
   - Add string assignments: `model` = `bytedance/seedance-2.5`, `resolution` = `720p`, `aspect_ratio` = `9:16`, `brand_voice` = `Upbeat, honest and specific. No hype words.`, `extra_hashtags` = `#tiktokshop`.
   - Add number assignments: `duration_seconds` = `15`, `price_per_second_usd` = `0.2312`, `max_photos` = `4`, `poll_every_seconds` = `20`, `timeout_minutes` = `30`, `max_file_mb` = `50`, `min_seconds` = `3`, `max_seconds` = `60`.
   - Add boolean assignment: `generate_audio` = `true`.
   - Connect `Product Video Form` output to `Set Video Settings`.

3. **Create Photo Branching & Preparation Nodes:**
   - Create an **If** node named `Has Product Photos?`. Configure condition to check if the filtered count of binary items starting with `image/` from the form is greater than 0. Connect `Set Video Settings` to `Has Product Photos?`.
   - Create a **Code** node named `Prepare Product Photos` with JavaScript parsing binary attachments and slicing by `max_photos`. Connect `HasProduct Photos?` (True branch) to this node.
   - Create an **HTTP Request** node named `Upload Photo to Replicate`. Set method to `POST`, URL to `https://api.replicate.com/v1/files`, content type to `multipart-form-data`, input field name to `photo`. Configure Generic Credential Type: HTTP Bearer Auth. Enable retry on fail. Connect `Prepare Product Photos` to this node.
   - Create a **Code** node named `Collect Photo URLs` to extract `urls.get` values or handle zero uploads. Connect both `Upload Photo to Replicate` and `Has Product Photos?` (False branch) to this node.

4. **Create AI Script Generation Nodes:**
   - Create an **OpenAI Chat Model** node named `OpenAI GPT-6 Luna` configured with model `gpt-6-luna`. Add an OpenAI credential.
   - Create a **Structured Output Parser** node named `Script Format` with a JSON schema defining `seedance_prompt`, `hook`, `voiceover`, and `caption`.
   - Create a **Basic LLM Chain** node named `Write Script and Prompt`. Configure prompt instructions to build timed beats, strict factual constraints, and double-quoted spoken lines. Connect `OpenAI GPT-6 Luna` to the AI Language Model input and `Script Format` to the AI Output Parser input. Connect `Collect Photo URLs` to the main input.

5. **Create Seedance Execution & Polling Nodes:**
   - Create a **Code** node named `Build Seedance Request` to assemble the JSON payload with prompt, duration, resolution, aspect ratio, audio flag, and reference images. Connect `Write Script and Prompt` to this node.
   - Create an **HTTP Request** node named `Submit Seedance Job`. Set method to `POST`, URL to `=https://api.replicate.com/v1/models/{{ $json.model }}/predictions`, specify body as JSON using `={{ JSON.stringify($json.body) }}`. Configure HTTP Bearer Auth. Connect `Build Seedance Request` to this node.
   - Create a **Wait** node named `Wait Between Checks`. Set unit to seconds and amount to `={{ $('Set Video Settings').first().json.poll_every_seconds }}`. Connect `Submit Seedance Job` and the fallback branch of the subsequent switch to this node.
   - Create an **HTTP Request** node named `Check Seedance Job`. Set method to `GET`, URL to `=https://api.replicate.com/v1/predictions/{{ $json.id }}`. Configure HTTP Bearer Auth and retry on fail. Connect `Wait Between Checks` to this node.
   - Create a **Switch** node named `Route by Job Status`. Configure rules for Succeeded (`status === 'succeeded'`), Failed (`['failed', 'canceled', 'aborted'].includes($json.status)`), and Timed Out (elapsed time > `timeout_minutes`). Set fallback output to "Still Running". Connect `Check Seedance Job` to this node.
   - Create an **HTTP Request** node named `Cancel Timed Out Job`. Set method to `POST`, URL to `={{ $json.urls.cancel }}` with HTTP Bearer Auth. Connect the Timed Out branch of `Route by Job Status` to this node.

6. **Create Validation & Quality Control Nodes:**
   - Create an **HTTP Request** node named `Download Video`. Set method to `GET`, URL to `={{ Array.isArray($json.output) ? $json.output[0] : $json.output }}`, response format to File under binary property `data`. Enable retry on fail. Connect the Succeeded branch of `Route by Job Status` to this node.
   - Create a **Code** node named `Check Size and Duration` with JavaScript parsing binary buffers for `mvhd` box headers, checking file size against `max_file_mb`, and validating duration limits. Connect `Download Video` to this node.
   - Create an **If** node named `Video Passes Checks`. Configure condition to check `={{ $json.ok }}`. Connect `Check Size and Duration` to this node.

7. **Create Distribution & Error Handling Nodes:**
   - Create a **SocialRobot** community node named `Save TikTok Draft`. Set resource to `tiktok`, operation to `create`, post mode to `UPLOAD` (Send to Inbox / Draft), publish mode to `NOW`, media type to `VIDEO`, binary property to `data`. Configure SocialRobot API credentials and select the TikTok account. Enable "Continue Node on Error". Connect the True branch of `Video Passes Checks` to this node.
   - Create a **Set** node named `Summarize Draft` to map final execution metrics. Connect the main output of `Save TikTok Draft` to this node.
   - Create a **SocialRobot** community node named `Optional Instagram Reel` (disabled by default). Set resource to `instagram`, operation to `create`, media type to `VIDEO`, binary property to `data`, publish mode to `NOW`, `isReel` to true, and `isAiGenerated` to true. Connect the True branch of `Video Passes Checks` to this node.
   - Create a **Code** node named `Describe Failure` to aggregate error strings from cancelled jobs, failed statuses, SocialRobot errors, or validation problems. Connect the Failed branch of `Route by Job Status`, the output of `Cancel Timed Out Job`, the False branch of `Video Passes Checks`, and the error output of `Save TikTok Draft` to this node.
   - Create a **Stop and Error** node named `Stop With Error` with error message `={{ $json.message }}`. Connect `Describe Failure` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Self-Hosted n8n Requirement | Requires self-hosted n8n instance to support community nodes. |
| SocialRobot Community Node Version | Requires SocialRobot community node version 3.1.1+ (npm latest 3.1.2). |
| Replicate API Pricing | Approximately $3.47 per 15-second video generated using ByteDance Seedance 2.5 at 720p (list price estimate). |
| Platform Publishing Notice | SocialRobot delivers TikTok videos to the mobile app inbox as drafts (`UPLOAD` mode); final AI-generated labels, sounds, and product tags must be applied inside the TikTok mobile app. |