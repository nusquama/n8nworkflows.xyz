Create YouTube thumbnails and quote cards from Drive videos with OpenAI and Canvora

https://n8nworkflows.xyz/workflows/create-youtube-thumbnails-and-quote-cards-from-drive-videos-with-openai-and-canvora-17758


# Create YouTube thumbnails and quote cards from Drive videos with OpenAI and Canvora

### 1. Workflow Overview

This workflow automates the generation of YouTube thumbnails and pull-quote graphics from raw video files. When a new video is added to a designated Google Drive folder, the workflow extracts its audio, transcribes it via OpenAI, uses a language model to derive a concise headline and key quotes, submits this structured concept to the Canvora API for visual asset generation, polls for completion, and finally compiles the resulting graphics into a review-ready Gmail draft.

The logical execution is grouped into five functional blocks:
- **1.1 Input Reception & Audio Transcription:** Triggers on new Drive files, downloads the video, and performs audio-to-text transcription using OpenAI.
- **1.2 Concept Extraction & Parsing:** Analyzes the transcript with an AI model to extract a punchy headline and pull quotes, then sanitizes and formats the raw response.
- **1.3 Visual Generation Dispatch:** Submits the structured concept payload to the Canvora API to generate YouTube thumbnail and quote card visuals.
- **1.4 Polling Loop & Status Validation:** Implements a polling loop with a 30-second delay to check generation status until success, failure, or timeout limits are reached.
- **1.5 Deliverable Assembly & Review:** Assembles the generated visual asset URLs into an HTML email structure and creates a draft in Gmail for final human review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Audio Transcription
- **Overview:** This block monitors a targeted Google Drive folder for incoming video assets, retrieves the binary file data, and transcribes the audio track into text using OpenAI's transcription service.
- **Nodes Involved:**
  - `When Video Added to Drive`
  - `Download Video`
  - `Transcribe Audio`
- **Node Details:**
  - **When Video Added to Drive**
    - *Type & Technical Role:* `n8n-nodes-base.googleDriveTrigger` (Event trigger). Polls a specific Google Drive folder every minute for newly created files.
    - *Configuration:* Event set to `fileCreated`, trigger mode set to `specificFolder`.
    - *Input/Output:* No incoming connections; outputs new file object metadata downstream to `Download Video`.
    - *Edge Cases/Failure Types:* Google Drive API rate limits, revoked authentication tokens, or missing folder IDs.
  - **Download Video**
    - *Type & Technical Role:* `n8n-nodes-base.googleDrive` (Action). Downloads the binary video content using the file ID from the trigger node.
    - *Configuration:* Operation set to `download`, file ID evaluated via expression `={{ $json.id }}`.
    - *Input/Output:* Input from `When Video Added to Drive`; outputs binary file data to `Transcribe Audio`.
    - *Edge Cases/Failure Types:* Large file timeouts, network drops, or insufficient storage/permissions.
  - **Transcribe Audio**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI LangChain / Audio). Transcribes binary audio input to text.
    - *Configuration:* Resource set to `audio`, operation set to `transcribe`. Requires active OpenAI credentials.
    - *Input/Output:* Input from `Download Video`; outputs transcription text object to `Extract Thumbnail + Quotes`.
    - *Edge Cases/Failure Types:* Unsupported audio formats, excessive file sizes exceeding OpenAI limits, or API key exhaustion.

#### 2.2 Concept Extraction & Parsing
- **Overview:** This block processes the raw transcript to extract structured design concepts (a punchy headline and three pull-quotes) via an LLM, then normalizes the output text via custom JavaScript.
- **Nodes Involved:**
  - `Extract Thumbnail + Quotes`
  - `Parse Concept`
- **Node Details:**
  - **Extract Thumbnail + Quotes**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI LangChain / Chat Model). Executes a chat prompt requesting strict JSON containing a thumbnail headline and three quotes.
    - *Configuration:* Model set to `gpt-5-mini`. Prompt references transcript text via expression: `={{ From this video transcript, return STRICT JSON only: {"headline": "...", "quotes": [...]}. Transcript: {{ $('Transcribe Audio').item.json.text }} }}`.
    - *Input/Output:* Input from `Transcribe Audio`; outputs LLM message payload to `Parse Concept`.
    - *Edge Cases/Failure Types:* Malformed JSON responses from the LLM model, rate limits, or context window limits.
  - **Parse Concept**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation). Safely parses the LLM output, handles regex fallbacks if the response contains markdown or non-strict wrappers, and builds a consolidated string format.
    - *Configuration:* Custom JavaScript snippet parsing `.message?.content` or `.text`, extracting JSON objects, and formatting quotes.
    - *Input/Output:* Input from `Extract Thumbnail + Quotes`; outputs sanitized `{ headline, content }` JSON to `Post to Canvora`.
    - *Edge Cases/Failure Types:* Script execution errors if text structure is entirely unanticipated by regex fallback logic.

#### 2.3 Visual Generation Dispatch
- **Overview:** Submits the parsed text concept and formatting preferences to the Canvora API to trigger asynchronous rendering of the visual assets.
- **Nodes Involved:**
  - `Post to Canvora`
- **Node Details:**
  - **Post to Canvora**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API integration). Sends a POST request to dispatch the generation job.
    - *Configuration:* URL `https://api.canvora.ai/api/generations`, method `POST`, authentication set to generic HTTP Header Auth (`X-API-Key`). Request body built via JSON stringification including `inputType: 'text'`, input content, title prefix, and `outputFormats: ['youtube_thumbnail', 'quote_card']`.
    - *Input/Output:* Input from `Parse Concept`; outputs generation task confirmation object (containing generation ID) to `Wait 30s`.
    - *Edge Cases/Failure Types:* Invalid API keys, HTTP 4xx/5xx responses from Canvora, or invalid payload schemas.

#### 2.4 Polling Loop & Status Validation
- **Overview:** Implements a delay-and-check loop to wait for the asynchronous graphic generation process to finalize on the Canvora backend.
- **Nodes Involved:**
  - `Wait 30s`
  - `Fetch Generation Status`
  - `If Generation Done`
  - `If Timed Out`
  - `If Generation Succeeded`
  - `Generation Failed (credits auto-refunded)`
- **Node Details:**
  - **Wait 30s**
    - *Type & Technical Role:* `n8n-nodes-base.wait` (Flow control). Pauses workflow execution for 30 seconds between polling requests.
    - *Configuration:* Amount set to `30` seconds.
    - *Input/Output:* Inputs from `Post to Canvora` and retry loop from `If Timed Out`; outputs to `Fetch Generation Status`.
  - **Fetch Generation Status**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API integration). Queries the current job state from Canvora.
    - *Configuration:* URL uses expression `=https://api.canvora.ai/api/generations/{{ $('Post to Canvora').item.json.generation.id }}` with HTTP Header Auth.
    - *Input/Output:* Input from `Wait 30s`; outputs generation status object to `If Generation Done`.
  - **If Generation Done**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional router). Evaluates whether the generation task has reached a terminal status (`completed`, `failed`, `partial`, or `cancelled`).
    - *Configuration:* Left value expression checks status inclusion: `={{ ['completed','failed','partial','cancelled'].includes($json.generation.status) }}`.
    - *Input/Output:* Input from `Fetch Generation Status`. True branch leads to `If Generation Succeeded`; false branch leads to `If Timed Out`.
  - **If Timed Out**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional router). Determines if the polling loop has exceeded maximum iteration limits.
    - *Configuration:* Left value expression checks run index: `={{ $runIndex >= 20 }}` (allowing up to 20 retry attempts, roughly 10 minutes total).
    - *Input/Output:* Input from `If Generation Done` (false branch). True branch routes to `Generation Failed`; false branch loops back to `Wait 30s`.
  - **If Generation Succeeded**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional router). Checks whether the completed generation status represents a successful outcome.
    - *Configuration:* Left value expression: `={{ ['completed','partial'].includes($json.generation.status) }}`.
    - *Input/Output:* Input from `If Generation Done` (true branch). True branch leads to `Build Review Email`; false branch leads to `Generation Failed`.
  - **Generation Failed (credits auto-refunded)**
    - *Type & Technical Role:* `n8n-nodes-base.noOp` (Terminal / Marker). Acts as a placeholder or endpoint for failed or timed-out generation processes where credits are automatically returned.
    - *Configuration:* Default empty parameters.
    - *Input/Output:* Inputs from `If Timed Out` (true branch) and `If Generation Succeeded` (false branch); no outputs.

#### 2.5 Deliverable Assembly & Review
- **Overview:** Formats the generated asset image URLs into an HTML review layout and creates a corresponding draft message in Gmail.
- **Nodes Involved:**
  - `Build Review Email`
  - `Create Gmail Draft`
- **Node Details:**
  - **Build Review Email**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation). Iterates through generation output arrays, mapping format titles and file URLs into an HTML email template containing review image tags and an edit link.
    - *Configuration:* JavaScript mapping outputs to construct HTML string and subject line.
    - *Input/Output:* Input from `If Generation Succeeded` (true branch); outputs `{ subject, html }` object to `Create Gmail Draft`.
  - **Create Gmail Draft**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email integration). Creates an email draft inside the connected Gmail account for human review before publishing.
    - *Configuration:* Resource set to `draft`, message body set via expression `={{ $json.html }}`, subject set via `={{ $json.subject }}`. Requires Gmail OAuth2 credentials.
    - *Input/Output:* Input from `Build Review Email`; terminal node with no downstream connections.
    - *Edge Cases/Failure Types:* Gmail API authorization expiration, scope limitations, or quota errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **When Video Added to Drive** | `n8n-nodes-base.googleDriveTrigger` | Triggers workflow on new video creation in a specific folder. | None | Download Video | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft<br><br>### How it works<br>Drop a video into a watched Google Drive folder. The workflow transcribes it,<br>an AI model pulls a punchy **thumbnail headline** and three **pull-quotes**,<br>and Canvora designs a **YouTube thumbnail** plus **quote cards** — on brand.<br>The finished visuals are assembled into a **Gmail draft** for a quick review<br>before you publish (no auto-send, no Slack).<br><br>### Setup steps<br>- Create a free Canvora account at canvora.ai — free credits, no card — and<br>  copy an API key from Integrations → API Keys (all plans).<br>- In each Canvora HTTP Request node, add a **Header Auth** credential:<br>  name `X-API-Key`, value your `vd_...` key.<br>- Connect your Google Drive and Gmail credentials.<br>- Add your OpenAI credential (used for transcription + concept extraction).<br>- In the trigger, pick the Drive folder you'll drop videos into.<br><br>### Customization<br>- Add `instagram_reel` or `twitter_post` to `outputFormats` for more clip<br>  art (100+ formats: GET https://api.canvora.ai/api/formats).<br>- Pass a `brandId` (GET /api/brands) to lock visuals to your brand kit.<br>- Change "Create Gmail Draft" to "send" to skip review, or upload the images<br>  to Drive instead.<br>- Cost: 2 visuals = 20 credits per video.<br>---<br>## Transcribe the video<br><br>When a video lands in the watched Drive folder, it's downloaded, transcribed, and an AI model pulls a headline + pull-quotes. |
| **Download Video** | `n8n-nodes-base.googleDrive` | Downloads the binary video file from Google Drive. | When Video Added to Drive | Transcribe Audio | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Transcribe the video<br><br>When a video lands in the watched Drive folder, it's downloaded, transcribed, and an AI model pulls a headline + pull-quotes. |
| **Transcribe Audio** | `@n8n/n8n-nodes-langchain.openAi` | Transcribes the downloaded video's audio track into text. | Download Video | Extract Thumbnail + Quotes | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Transcribe the video<br><br>When a video lands in the watched Drive folder, it's downloaded, transcribed, and an AI model pulls a headline + pull-quotes. |
| **Extract Thumbnail + Quotes** | `@n8n/n8n-nodes-langchain.openAi` | Extracts structured headline and quote data from the transcript using LLM. | Transcribe Audio | Parse Concept | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Transcribe the video<br><br>When a video lands in the watched Drive folder, it's downloaded, transcribed, and an AI model pulls a headline + pull-quotes. |
| **Parse Concept** | `n8n-nodes-base.code` | Parses, sanitizes, and formats the raw LLM output into a unified content payload. | Extract Thumbnail + Quotes | Post to Canvora | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft... |
| **Post to Canvora** | `n8n-nodes-base.httpRequest` | Submits the concept payload to the Canvora API to trigger graphic generation. | Parse Concept | Wait 30s | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Design the visuals<br><br>Sends the concept to Canvora, which designs a YouTube thumbnail and quote cards. |
| **Wait 30s** | `n8n-nodes-base.wait` | Pauses execution for 30 seconds before polling generation status. | Post to Canvora, If Timed Out | Fetch Generation Status | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Poll generation status<br><br>Waits between checks, fetches the state, and loops until the visuals finish or it times out. |
| **Fetch Generation Status** | `n8n-nodes-base.httpRequest` | Queries the current generation status from the Canvora API. | Wait 30s | If Generation Done | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Poll generation status<br><br>Waits between checks, fetches the state, and loops until the visuals finish or it times out. |
| **If Generation Done** | `n8n-nodes-base.if` | Checks if the generation task has reached a terminal state. | Fetch Generation Status | If Generation Succeeded, If Timed Out | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Poll generation status<br><br>Waits between checks, fetches the state, and loops until the visuals finish or it times out. |
| **If Timed Out** | `n8n-nodes-base.if` | Verifies whether the polling loop has exceeded 20 retry attempts. | If Generation Done | Generation Failed (credits auto-refunded), Wait 30s | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Poll generation status<br><br>Waits between checks, fetches the state, and loops until the visuals finish or it times out. |
| **If Generation Succeeded** | `n8n-nodes-base.if` | Validates if the completed task succeeded or failed. | If Generation Done | Build Review Email, Generation Failed (credits auto-refunded) | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Deliver for review<br><br>On success, assembles the visuals into a Gmail draft for a quick review before publishing. |
| **Build Review Email** | `n8n-nodes-base.code` | Constructs HTML review markup combining asset image previews and edit links. | If Generation Succeeded | Create Gmail Draft | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Deliver for review<br><br>On success, assembles the visuals into a Gmail draft for a quick review before publishing. |
| **Create Gmail Draft** | `n8n-nodes-base.gmail` | Creates a Gmail draft message containing the generated review package. | Build Review Email | None | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Deliver for review<br><br>On success, assembles the visuals into a Gmail draft for a quick review before publishing. |
| **Generation Failed (credits auto-refunded)** | `n8n-nodes-base.noOp` | Terminal handling node for failed, partial, or timed-out visual generations. | If Timed Out, If Generation Succeeded | None | ## New Video → Thumbnail + Quote Cards → Drive + Gmail Draft...<br>---<br>## Handle failure<br><br>If generation fails or times out, credits are refunded automatically. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create the Trigger Node:**
   - Add a **Google Drive Trigger** node (`When Video Added to Drive`).
   - Configure event to `fileCreated`, polling interval to every minute, and select a target folder to watch.
   - Set required Google Drive credentials.

2. **Add Video Download Node:**
   - Add a **Google Drive** node (`Download Video`).
   - Set operation to `download` and configure the file ID parameter using the expression: `={{ $json.id }}`.
   - Connect `When Video Added to Drive` main output to `Download Video`.

3. **Add Audio Transcription Node:**
   - Add an **OpenAI** node (`Transcribe Audio`) from the LangChain integration category.
   - Set resource to `audio`, operation to `transcribe`. Configure OpenAI credentials.
   - Connect `Download Video` main output to `Transcribe Audio`.

4. **Add Concept Extraction Node:**
   - Add a second **OpenAI** chat node (`Extract Thumbnail + Quotes`).
   - Set model to `gpt-5-mini`. Populate the prompt field with: 
     `=From this video transcript, return STRICT JSON only: {"headline": "a punchy 4-7 word thumbnail headline", "quotes": ["quote 1", "quote 2", "quote 3"]}. Transcript: {{ $('Transcribe Audio').item.json.text }}`
   - Connect `Transcribe Audio` main output to `Extract Thumbnail + Quotes`.

5. **Add Concept Parser Code Node:**
   - Add a **Code** node (`Parse Concept`).
   - Insert the JavaScript snippet to safely parse the JSON output from the LLM and format it into a combined string object (`headline` and `content`).
   - Connect `Extract Thumbnail + Quotes` main output to `Parse Concept`.

6. **Add Canvora API Dispatch Node:**
   - Add an **HTTP Request** node (`Post to Canvora`).
   - Set method to `POST`, URL to `https://api.canvora.ai/api/generations`.
   - Configure generic HTTP Header Authentication with header name `X-API-Key` and your Canvora API key.
   - Enable JSON body specification, setting the body to:
     `={{ JSON.stringify({ inputType: 'text', inputContent: $('Parse Concept').item.json.content, title: 'Video kit: ' + $('Parse Concept').item.json.headline, outputFormats: ['youtube_thumbnail', 'quote_card'] }) }}`
   - Connect `Parse Concept` main output to `Post to Canvora`.

7. **Add Polling Loop Infrastructure:**
   - Add a **Wait** node (`Wait 30s`) with an amount of `30` seconds. Connect `Post to Canvora` output to it.
   - Add an **HTTP Request** node (`Fetch Generation Status`). Set method to `GET`, URL to `=https://api.canvora.ai/api/generations/{{ $('Post to Canvora').item.json.generation.id }}` using HTTP Header Auth. Connect `Wait 30s` output to it.
   - Add an **If** node (`If Generation Done`). Set condition to check if `{{ ['completed','failed','partial','cancelled'].includes($json.generation.status) }}` is true. Connect `Fetch Generation Status` output to it.

8. **Add Timeout and Success Condition Routers:**
   - From the `false` output of `If Generation Done`, connect to an **If** node (`If Timed Out`). Set condition to check if `={{ $runIndex >= 20 }}` is true.
     - If true (timeout reached), connect to a **No-Op** node (`Generation Failed (credits auto-refunded)`).
     - If false (retry limit not reached), connect back to the input of `Wait 30s`.
   - From the `true` output of `If Generation Done`, connect to an **If** node (`If Generation Succeeded`). Set condition to check if `={{ ['completed','partial'].includes($json.generation.status) }}` is true.
     - If false, connect to `Generation Failed (credits auto-refunded)`.

9. **Add Review Compilation and Email Delivery Nodes:**
   - From the `true` output of `If Generation Succeeded`, connect to a **Code** node (`Build Review Email`). Insert JavaScript that compiles the output image URLs and formats an HTML string.
   - Add a **Gmail** node (`Create Gmail Draft`). Set resource to `draft`, subject to `={{ $json.subject }}`, and message body to `={{ $json.html }}`. Configure Gmail OAuth2 credentials.
   - Connect `Build Review Email` output to `Create Gmail Draft`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Canvora platform for agentic visual generation | [canvora.ai/for-agents](https://canvora.ai/for-agents) |
| Canvora API format endpoint reference | [Canvora Formats API](https://api.canvora.ai/api/formats) |
| Canvora API brand kit reference | [Canvora Brands API](https://api.canvora.ai/api/brands) |