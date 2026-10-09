Approve short-form AI videos for TikTok, Reels and Shorts with Gemini, Veo and Telegram

https://n8nworkflows.xyz/workflows/approve-short-form-ai-videos-for-tiktok--reels-and-shorts-with-gemini--veo-and-telegram-20504


# Approve short-form AI videos for TikTok, Reels and Shorts with Gemini, Veo and Telegram

### 1. Workflow Overview

This workflow automates the end-to-end creation, review, and multi-platform publishing of short-form AI videos. It monitors a Google Sheets document for new video ideas, uses Google Gemini to generate structured prompts and network-specific captions, creates vertical video clips via Google Veo 3.1, delivers them to Telegram for human approval, and publishes approved videos to TikTok, Instagram Reels, and YouTube Shorts via PostWire. 

The architecture is divided into six logical blocks:
- **1.1 Input Reception:** Monitors Google Sheets and filters raw rows to isolate unprocessed video concepts.
- **1.2 AI Content Planning:** Interacts with Google Gemini (`gemini-2.5-flash`) to generate a production-ready Veo prompt and metadata for multiple platforms.
- **1.3 Video Rendering and Upload Preparation:** Allocates an upload slot via PostWire, generates an 8-second 9:16 video using Google Veo 3.1 (`veo-3.1-fast-generate-preview`), uploads the binary data, and finalizes the media URL.
- **1.4 Telegram Review & Approval Gate:** Transmits the rendered video and formatted captions to a Telegram chat, pausing execution until the user responds with an interactive approval action.
- **1.5 Publishing & Success Handling:** Automatically pushes approved media to social channels via PostWire, verifies success, updates Google Sheets with live metrics, and broadcasts confirmation reports to Telegram.
- **1.6 Rejection & Failure Handling:** Manages declined ideas or publishing failures by logging appropriate error statuses, remediation hints, and timestamps back to Google Sheets and Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Detects newly inserted rows in a Google Sheets tracking table and filters out blank or previously processed items to maintain idempotency.
- **Nodes Involved:** `New idea in Google Sheets`, `Only new ideas`
- **Node Details:**
  - **New idea in Google Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheetsTrigger` (Polling Trigger)
    - *Configuration Choices:* Triggered on event `rowAdded` with a 1-minute poll cycle. Requires target Document ID and Sheet Name configuration.
    - *Key Expressions:* None (relies on native sheet indexing).
    - *Input / Output:* Input: None (Trigger) -> Output: `Only new ideas`.
    - *Edge Cases / Potential Failures:* API rate limits on Google Sheets; missing OAuth2 credentials.
  - **Only new ideas**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Data Filter)
    - *Configuration Choices:* Evaluates two strict conditions via v2 rule builder (`combinator: and`).
    - *Key Expressions:* 
      - Left Value 1: `={{ String($json.idea || '').trim() }}` (Operator: `notEmpty`)
      - Left Value 2: `={{ String($json.status || '').trim() }}` (Operator: `empty`)
    - *Input / Output:* Input: `New idea in Google Sheets` -> Output: `Plan the video with Gemini`.
    - *Edge Cases / Potential Failures:* Trimming prevents hidden whitespace from bypassing duplicate safety checks.

#### 2.2 AI Content Planning
- **Overview:** Sends the validated idea to Google Gemini to formulate structured screenplay directions (Veo prompt) and custom textual assets for social platforms.
- **Nodes Involved:** `Plan the video with Gemini`, `Read the plan`
- **Node Details:**
  - **Plan the video with Gemini**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.googleGemini` (AI Text Generation / Chat Model)
    - *Configuration Choices:* Model set to `models/gemini-2.5-flash`. Structured system message mandates strict JSON return keys (`veo_prompt`, `tiktok`, `instagram`, `youtube_title`, `youtube`, `linkedin`, `facebook`). JSON output mode enabled.
    - *Key Expressions:* Prompt content: `={{ 'Idea: ' + $json.idea }}`
    - *Input / Output:* Input: `Only new ideas` -> Output: `Read the plan`.
    - *Edge Cases / Potential Failures:* Model output parsing failures if the AI deviates from strict JSON schemas.
  - **Read the plan**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformer / Assignment)
    - *Configuration Choices:* Parses raw Gemini text output into structured objects and assigns operational workflow variables.
    - *Key Expressions:* 
      - `plan`: `={{ JSON.parse($('Plan the video with Gemini').item.json.content.parts.map(p => p.text || '').join('')) }}`
      - `idea`: `={{ String($('Only new ideas').item.json.idea).trim() }}`
      - `networks`: `={{ ['tiktok', 'instagram', 'youtube'] }}`
      - `telegram_chat_id`: `YOUR_CHAT_ID`
    - *Input / Output:* Input: `Plan the video with Gemini` -> Output: `Get an upload slot`.
    - *Edge Cases / Potential Failures:* Malformed JSON strings thrown by Gemini will trigger expression execution errors.

#### 2.3 Video Rendering and Upload Preparation
- **Overview:** Secures an ingestion slot via PostWire API, generates an 8-second 9:16 vertical video using Google Veo 3.1, uploads the generated media stream, and resolves a valid media URL.
- **Nodes Involved:** `Get an upload slot`, `Generate the clip with Veo 3.1`, `Upload the clip`, `Get the video link`
- **Node Details:**
  - **Get an upload slot**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration Choices:* POST request to `https://postwire.io/api/media/upload-url` using generic HTTP Header Authentication. Body specifies `{ content_type: 'video/mp4' }`. Never error flag active.
    - *Key Expressions:* JSON Body: `={{ JSON.stringify({ content_type: 'video/mp4' }) }}`
    - *Input / Output:* Input: `Read the plan` -> Output: `Generate the clip with Veo 3.1`.
    - *Edge Cases / Potential Failures:* Invalid PostWire API credentials; API downtime.
  - **Generate the clip with Veo 3.1**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.googleGemini` (AI Video Generation)
    - *Configuration Choices:* Model set to `models/veo-3.1-fast-generate-preview`. Resource set to `video`, operation to `generate`. Aspect ratio locked to `9:16`, duration set to `8` seconds.
    - *Key Expressions:* Prompt: `={{ $('Read the plan').item.json.plan.veo_prompt }}`
    - *Input / Output:* Input: `Get an upload slot` -> Output: `Upload the clip`.
    - *Edge Cases / Potential Failures:* Requires billing-enabled Google Gemini accounts; high execution costs; long generation timeouts.
  - **Upload the clip**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Binary HTTP Upload)
    - *Configuration Choices:* PUT request targeting the dynamic upload URL returned from previous steps. Content-type configured as binary data (`contentType: 'binaryData'`).
    - *Key Expressions:* URL: `={{ $('Get an upload slot').item.json.upload_url }}`
    - *Input / Output:* Input: `Generate the clip with Veo 3.1` -> Output: `Get the video link`.
    - *Edge Cases / Potential Failures:* Binary stream dropouts or storage quota exhaustion on PostWire.
  - **Get the video link**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Ingestion Verification)
    - *Configuration Choices:* GET request to `https://postwire.io/api/media/finalize` utilizing HTTP Header Auth with query parameters.
    - *Key Expressions:* Query Parameter `path`: `={{ $('Get an upload slot').item.json.path }}`
    - *Input / Output:* Input: `Upload the clip` -> Output: `Send the clip to Telegram`.
    - *Edge Cases / Potential Failures:* Premature finalization calls if upload processing has not completed.

#### 2.4 Telegram Review & Approval Gate
- **Overview:** Dispatches the rendered video file and platform-tailored captions to a designated Telegram chat, halting execution until the recipient interacts with inline approval buttons.
- **Nodes Involved:** `Send the clip to Telegram`, `Ask for approval in Telegram`, `Approved?`
- **Node Details:**
  - **Send the clip to Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Messaging API)
    - *Configuration Choices:* Sends a video via `sendVideo` resource using remote URLs. Error handling configured to continue regular output on failure. HTML parse mode enabled.
    - *Key Expressions:* 
      - File: `={{ $json.media_url }}`
      - Chat ID: `={{ $('Read the plan').item.json.telegram_chat_id }}`
      - Caption: Sanitized truncation logic combining TikTok, Instagram, and YouTube copy strings up to 900 characters.
    - *Input / Output:* Input: `Get the video link` -> Output: `Ask for approval in Telegram`.
    - *Edge Cases / Potential Failures:* Telegram file size limitations or unsupported video container constraints.
  - **Ask for approval in Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Interactive Workflow Gate)
    - *Configuration Choices:* Operation set to `sendAndWait` with response type `approval` (`double` type, custom labels: "Approve" and "Decline"). Webhook ID registered for callback tracking.
    - *Key Expressions:* Chat ID: `={{ $('Read the plan').item.json.telegram_chat_id }}`
    - *Input / Output:* Input: `Send the clip to Telegram` -> Output: `Approved?`.
    - *Edge Cases / Potential Failures:* Webhook timeouts if the user fails to respond within Telegram's active interaction window.
  - **Approved?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router)
    - *Configuration Choices:* Validates boolean payload returned from the Telegram approval action.
    - *Key Expressions:* Left Value: `={{ $json.data.approved }}` (Operator: `true`)
    - *Input / Output:* Input: `Ask for approval in Telegram` -> Outputs: True Branch (`Publish with PostWire`) / False Branch (`Mark the idea as declined`).

#### 2.5 Publishing & Success Handling
- **Overview:** Broadcasts approved media across targeted networks using customized parameters, verifies network-level delivery success, updates tracking sheets, and sends confirmation summaries.
- **Nodes Involved:** `Publish with PostWire`, `Every network published?`, `Log the links`, `Send the links to Telegram`
- **Node Details:**
  - **Publish with PostWire**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Multi-platform Dispatcher)
    - *Configuration Choices:* POST request to `https://postwire.io/api/post` using generic HTTP Header Auth. JSON body builds platform-specific mappings, AI generation flags, and idempotent keys.
    - *Key Expressions:* Comprehensive JSON body mapper handling platform arrays, individual text payloads, and title properties.
    - *Input / Output:* Input: `Approved?` (True branch) -> Output: `Every network published?`.
    - *Edge Cases / Potential Failures:* Network API restrictions or invalid platform auth tokens linked inside PostWire.
  - **Every network published?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Result Validator)
    - *Configuration Choices:* Evaluates whether the PostWire response array contains elements and every result object returns an `ok: true` status.
    - *Key Expressions:* Left Value: `={{ ($json.results || []).length > 0 && $json.results.every(r => r.ok) }}` (Operator: `true`)
    - *Input / Output:* Input: `Publish with PostWire` -> Outputs: True (`Log the links`) / False (`Log the failure`).
  - **Log the links**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Database Record Update)
    - *Configuration Choices:* Updates row matching configuration on column `idea`. Sets status to `posted` and compiles successful platform links.
    - *Key Expressions:* 
      - `status`: `posted`
      - `links`: `={{ ($json.results || []).filter(r => r.ok).map(r => r.platform + ' ' + (r.url || '')).join(' | ') }}`
    - *Input / Output:* Input: `Every network published?` (True) -> Output: `Send the links to Telegram`.
    - *Edge Cases / Potential Failures:* Schema mapping mismatches in Google Sheets.
  - **Send the links to Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Notification Dispatcher)
    - *Configuration Choices:* Sends plain text messaging with HTML formatting enabled.
    - *Key Expressions:* Text maps published network links, falling back to a processing status notice if URLs are delayed.
    - *Input / Output:* Input: `Log the links` -> Output: Terminal Node.

#### 2.6 Rejection & Failure Handling
- **Overview:** Captures user rejections from Telegram or partial publishing failures from PostWire, updating audit logs and alerting the operator with actionable remediation guidance.
- **Nodes Involved:** `Mark the idea as declined`, `Log the failure`, `Send the fix to Telegram`
- **Node Details:**
  - **Mark the idea as declined**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Database Record Update)
    - *Configuration Choices:* Updates target row based on `idea` matching. Sets status to `declined` and adds formatted timestamp notes.
    - *Key Expressions:* 
      - `status`: `declined`
      - `note`: `={{ 'Declined in Telegram on ' + $now.toFormat('yyyy-MM-dd HH:mm') }}`
    - *Input / Output:* Input: `Approved?` (False branch) -> Output: Terminal Node.
  - **Log the failure**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Database Record Update)
    - *Configuration Choices:* Updates row matching on `idea`. Dynamically calculates partial posting status and aggregates platform-specific fixes/errors.
    - *Key Expressions:* 
      - `status`: `={{ ($json.posted || 0) > 0 ? 'partly posted' : 'needs fix' }}`
      - `note`: Extracts exact error payloads or platform fixes.
    - *Input / Output:* Input: `Every network published?` (False) -> Output: `Send the fix to Telegram`.
  - **Send the fix to Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Error Alert Dispatcher)
    - *Configuration Choices:* Sends HTML-formatted messages outlining failed platforms and required corrective actions.
    - *Key Expressions:* Formats multi-line strings separating published vs rejected target endpoints.
    - *Input / Output:* Input: `Log the failure` -> Output: Terminal Node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **New idea in Google Sheets** | `n8n-nodes-base.googleSheetsTrigger` | Watches Google Sheets for newly added rows | None | Only new ideas | ## Approve AI videos in Telegram before they are posted<br>Add a video idea to Google Sheets and review the finished clip on your phone. Gemini plans it, Veo 3.1 renders it, and PostWire posts it to TikTok, Instagram Reels and YouTube Shorts only after you tap Approve in Telegram.<br><br>### How it works<br>1. **New idea in Google Sheets** fires when you add a row.<br>2. **Plan the video with Gemini** returns the Veo prompt, a caption for each network and the YouTube title.<br>3. **Generate the clip with Veo 3.1** renders 8 seconds in 9:16 and the file goes straight to a PostWire upload slot.<br>4. **Send the clip to Telegram** shows you the video with its captions, and **Ask for approval in Telegram** waits for your Approve or Decline.<br>5. On Approve, **Publish with PostWire** posts each network with its own text. The row and the chat get the links, or the exact fix for each network that refused. On Decline the row is marked declined.<br><br>### Setup<br>1. Create a free account at postwire.io, connect your networks and copy your API key. In n8n add a **Header Auth** credential (name `Authorization`, value `Bearer pw_live_…`) and select it in the three PostWire nodes.<br>2. Create a sheet with the columns idea, status, links and note, and pick it in the trigger and the three Sheets nodes.<br>3. Add a Google Gemini credential with billing enabled (about one dollar per 8-second Veo 3.1 Fast clip).<br>4. Create a bot with @BotFather, select its credential in the four Telegram nodes and type your chat ID in **Read the plan**.<br><br>### Customize<br>Edit the networks in Read the plan, or the video style in the Gemini system message. |
| **Only new ideas** | `n8n-nodes-base.filter` | Filters out rows lacking ideas or possessing existing statuses | New idea in Google Sheets | Plan the video with Gemini | ## Approve AI videos in Telegram before they are posted...<br>### 1. Idea from Google Sheets<br>A new row is the idea; rows without one, or already handled, are skipped. |
| **Plan the video with Gemini** | `@n8n/n8n-nodes-langchain.googleGemini` | Generates a Veo prompt and platform captions from an idea | Only new ideas | Read the plan | ## Approve AI videos in Telegram before they are posted...<br>### 2. Plan with Gemini<br>One Veo prompt and a caption for each network. |
| **Read the plan** | `n8n-nodes-base.set` | Parses Gemini output and sets workflow variables | Plan the video with Gemini | Get an upload slot | ## Approve AI videos in Telegram before they are posted...<br>### 2. Plan with Gemini<br>One Veo prompt and a caption for each network. |
| **Get an upload slot** | `n8n-nodes-base.httpRequest` | Requests an ingestion URL from PostWire | Read the plan | Generate the clip with Veo 3.1 | ## Approve AI videos in Telegram before they are posted...<br>### 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Generate the clip with Veo 3.1** | `@n8n/n8n-nodes-langchain.googleGemini` | Renders an 8-second vertical video via Veo | Get an upload slot | Upload the clip | ## Approve AI videos in Telegram before they are posted...<br>### 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Upload the clip** | `n8n-nodes-base.httpRequest` | Uploads binary video data to PostWire | Generate the clip with Veo 3.1 | Get the video link | ## Approve AI videos in Telegram before they are posted...<br>### 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Get the video link** | `n8n-nodes-base.httpRequest` | Finalizes upload and retrieves media URL | Upload the clip | Send the clip to Telegram | ## Approve AI videos in Telegram before they are posted...<br>### 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Send the clip to Telegram** | `n8n-nodes-base.telegram` | Delivers rendered video and captions to chat | Get the video link | Ask for approval in Telegram | ## Approve AI videos in Telegram before they are posted...<br>### 4. Review in Telegram<br>The clip and its captions arrive in your chat; the workflow waits for Approve or Decline. |
| **Ask for approval in Telegram** | `n8n-nodes-base.telegram` | Prompts user with interactive approval buttons | Send the clip to Telegram | Approved? | ## Approve AI videos in Telegram before they are posted...<br>### 4. Review in Telegram<br>The clip and its captions arrive in your chat; the workflow waits for Approve or Decline. |
| **Approved?** | `n8n-nodes-base.if` | Routes workflow based on user approval action | Ask for approval in Telegram | Publish with PostWire<br>Mark the idea as declined | ## Approve AI videos in Telegram before they are posted...<br>### 5. Publish, log and report<br>Approved clips go to every network; the row and the chat get the links or the fix. Declined ideas are marked in the sheet. |
| **Publish with PostWire** | `n8n-nodes-base.httpRequest` | Dispatches approved video to social platforms | Approved? | Every network published? | ## Approve AI videos in Telegram before they are posted...<br>### 5. Publish, log and report<br>Approved clips go to every network; the row and the chat get the links or the fix. Declined ideas are marked in the sheet. |
| **Every network published?** | `n8n-nodes-base.if` | Verifies complete multi-platform publishing success | Publish with PostWire | Log the links<br>Log the failure | ## Approve AI videos in Telegram before they are posted...<br>### 5. Publish, log and report<br>Approved clips go to every network; the row and the chat get the links or the fix. Declined ideas are marked in the sheet. |
| **Log the links** | `n8n-nodes-base.googleSheets` | Updates Google Sheets row with posted status and links | Every network published? | Send the links to Telegram | ## Approve AI videos in Telegram before they are posted...<br>### 5. Publish, log and report<br>Approved clips go to every network; the row and the chat get the links or the fix. Declined ideas are marked in the sheet. |
| **Log the failure** | `n8n-nodes-base.googleSheets` | Records publishing errors or partial statuses to sheet | Every network published? | Send the fix to Telegram | ## Approve AI videos in Telegram before they are posted...<br>### 5. Publish, log and report<br>Approved clips go to every network; the row and the chat get the links or the fix. Declined ideas are marked in the sheet. |
| **Send the links to Telegram** | `n8n-nodes-base.telegram` | Broadcasts success links to Telegram chat | Log the links | None | ## Approve AI videos in Telegram before they are posted...<br>### 5. Publish, log and report<br>Approved clips go to every network; the row and the chat get the links or the fix. Declined ideas are marked in the sheet. |
| **Send the fix to Telegram** | `n8n-nodes-base.telegram` | Reports publishing errors and required fixes to chat | Log the failure | None | ## Approve AI videos in Telegram before they are posted...<br>### 5. Publish, log and report<br>Approved clips go to every network; the row and the chat get the links or the fix. Declined ideas are marked in the sheet. |
| **Mark the idea as declined** | `n8n-nodes-base.googleSheets` | Updates row status to declined with timestamp note | Approved? | None | ## Approve AI videos in Telegram before they are posted...<br>### 5. Publish, log and report<br>Approved clips go to every network; the row and the chat get the links or the fix. Declined ideas are marked in the sheet. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheets Document:** Create a new sheet with four columns: `idea`, `status`, `links`, and `note`.
2. **Set up Credentials in n8n:**
   - **Google Sheets OAuth2:** Authorize access to your Google account.
   - **Google Gemini (LangChain):** Add your Gemini API credential with billing enabled.
   - **Telegram:** Create a bot via BotFather and add your Telegram bot token.
   - **HTTP Header Auth:** Add a generic Header Auth credential named `Authorization` with value `Bearer <your_postwire_api_key>` (referencing [PostWire](https://postwire.io)).
3. **Build Block 1 (Input Reception):**
   - Add a `Google Sheets Trigger` node (`New idea in Google Sheets`). Set event to `rowAdded`, poll every minute, and select your target document/sheet.
   - Add a `Filter` node (`Only new ideas`). Configure condition 1: `={{ String($json.idea || '').trim() }}` is not empty. Configure condition 2: `={{ String($json.status || '').trim() }}` is empty. Connect trigger output here.
4. **Build Block 2 (AI Content Planning):**
   - Add a `Google Gemini Chat Model` node (`Plan the video with Gemini`) using model `models/gemini-2.5-flash`. Enable JSON output. Set system message to plan a vertical 8-second video returning specific JSON keys (`veo_prompt`, `tiktok`, `instagram`, `youtube_title`, `youtube`, `linkedin`, `facebook`). Pass `={{ 'Idea: ' + $json.idea }}` as the user prompt. Connect filter output here.
   - Add a `Set` node (`Read the plan`). Assign variables: `plan` (JSON parsed Gemini output), `idea` (trimmed idea string), `networks` (`['tiktok', 'instagram', 'youtube']`), and `telegram_chat_id` (your Telegram numeric chat ID). Connect Gemini output here.
5. **Build Block 3 (Video Rendering and Upload):**
   - Add an `HTTP Request` node (`Get an upload slot`). Set method to `POST`, URL to `https://postwire.io/api/media/upload-url`, select HTTP Header Auth, send JSON body `{ content_type: 'video/mp4' }`. Enable "Never error". Connect `Read the plan` here.
   - Add a `Google Gemini` node (`Generate the clip with Veo 3.1`). Set resource to `video`, operation to `generate`, model to `models/veo-3.1-fast-generate-preview`, aspect ratio to `9:16`, duration to `8`. Pass `={{ $('Read the plan').item.json.plan.veo_prompt }}` as prompt. Connect upload slot output here.
   - Add an `HTTP Request` node (`Upload the clip`). Set method to `PUT`, URL to `={{ $('Get an upload slot').item.json.upload_url }}`, send body as binary data (`inputDataFieldName: 'data'`). Connect Veo node here.
   - Add an `HTTP Request` node (`Get the video link`). Set method to `GET`, URL to `https://postwire.io/api/media/finalize`, select HTTP Header Auth, add query parameter `path` set to `={{ $('Get an upload slot').item.json.path }}`. Connect upload node here.
6. **Build Block 4 (Telegram Review):**
   - Add a `Telegram` node (`Send the clip to Telegram`). Set resource to `message`, operation to `sendVideo`. Set file to `={{ $json.media_url }}`, chat ID to `={{ $('Read the plan').item.json.telegram_chat_id }}`, caption to sanitized platform summaries with HTML parse mode. Continue regular output on error. Connect video link node here.
   - Add a `Telegram` node (`Ask for approval in Telegram`). Set resource to `message`, operation to `sendAndWait`, response type to `approval` (double buttons: Approve / Decline). Connect previous Telegram node here.
   - Add an `If` node (`Approved?`). Set condition: `={{ $json.data.approved }}` is true. Connect approval node here.
7. **Build Block 5 (Publishing & Success Routing):**
   - From the True branch of `Approved?`, add an `HTTP Request` node (`Publish with PostWire`). Set method to `POST`, URL to `https://postwire.io/api/post`, select HTTP Header Auth, send JSON body mapping platforms, titles, and text strings.
   - Add an `If` node (`Every network published?`). Set condition verifying all results are ok (`={{ ($json.results || []).length > 0 && $json.results.every(r => r.ok) }}`). Connect PostWire node here.
   - From True branch, add a `Google Sheets` node (`Log the links`). Operation: `update`, match column `idea`, set status to `posted` and map success URLs. Connect to a `Telegram` node (`Send the links to Telegram`) sending success messages with HTML mode.
8. **Build Block 6 (Rejection & Failure Routing):**
   - From False branch of `Approved?`, add a `Google Sheets` node (`Mark the idea as declined`). Operation: `update`, match column `idea`, set status to `declined`, and set note timestamp using `$now.toFormat('yyyy-MM-dd HH:mm')`.
   - From False branch of `Every network published?`, add a `Google Sheets` node (`Log the failure`). Operation: `update`, match column `idea`, set status to `partly posted` or `needs fix`, and map error notes. Connect to a `Telegram` node (`Send the fix to Telegram`) broadcasting remediation steps.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create a free account at PostWire to connect social networks and acquire API keys. | [PostWire](https://postwire.io) |
| Estimate approximately one dollar per 8-second Veo 3.1 Fast clip when configuring billing-enabled Gemini accounts. | Google Gemini Billing Documentation |
| Create a custom Telegram bot utilizing BotFather to manage approval interactions. | Telegram BotFather |