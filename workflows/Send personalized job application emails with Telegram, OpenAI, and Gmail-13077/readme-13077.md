Send personalized job application emails with Telegram, OpenAI, and Gmail

https://n8nworkflows.xyz/workflows/send-personalized-job-application-emails-with-telegram--openai--and-gmail-13077


# Send personalized job application emails with Telegram, OpenAI, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send personalized job application emails with Telegram, OpenAI, and Gmail  
**Workflow name (in JSON):** Send job applications with Telegram, OpenAI, and Gmail  
**Purpose:** Automate drafting and sending tailored job application emails from a *job posting screenshot* received in Telegram. The workflow uses OpenAI (Vision + a stored resume file) to extract job details and generate a personalized email draft, stores the draft in Redis, asks the user for confirmation in Telegram, then sends the email via Gmail with the resume attached from Google Drive (immediate + scheduled send paths).

### Logical blocks
1. **1.1 Telegram Intake & Routing**  
   Receives Telegram updates (message with photo or callback button click) and routes accordingly.
2. **1.2 AI Request Preparation & Vision+Resume Analysis**  
   Builds an OpenAI Responses API payload including the image, caption, and a resume file reference; calls OpenAI and parses strict JSON output.
3. **1.3 Draft Persistence & Telegram Confirmation**  
   Stores the draft in Redis keyed by job title (normalized), then sends a preview message to Telegram with an inline “Send The Email” button.
4. **1.4 Confirmation Handling, Resume Download, and Sending (Immediate + Scheduled)**  
   On callback: retrieves the stored draft, downloads resume from Drive, optionally waits 10 seconds, then sends the email via Gmail and notifies the user.

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Intake & Routing

**Overview:** Listens to Telegram for either (a) a job post screenshot message or (b) a callback query from the approval button. Routes the execution to the AI analysis path or the sending path.

**Nodes involved:**
- Telegram Trigger
- Route by message type
- Show typing indicator

#### Node: Telegram Trigger
- **Type / Role:** `telegramTrigger` — entry point webhook for Telegram updates.
- **Configuration (interpreted):**
  - Listens to `message` and `callback_query` updates.
  - `download: true` to automatically download media and include it in `binary`.
- **Key data used downstream:**
  - For images: `json.message.photo` and downloaded `binary` content.
  - For callbacks: `json.callback_query...`
- **Outputs / Connections:**
  - Output → **Route by message type**
- **Version requirements:** typeVersion `1.2`.
- **Failure/edge cases:**
  - Missing webhook configuration in n8n/Telegram credentials can prevent updates.
  - If Telegram sends a message without a photo, the AI path will fail later due to missing binary.
  - Large photos may hit download/size limits depending on n8n and Telegram API behavior.

#### Node: Route by message type
- **Type / Role:** `switch` — routes execution based on the incoming update shape.
- **Configuration (interpreted):**
  - Output **Job_Post_Image** when `{{$json.message?.photo}}` is a **non-empty array**.
  - Output **Sending** when `{{$json.callback_query?.message}}` is a **non-empty object**.
- **Outputs / Connections:**
  - **Job_Post_Image** → **Show typing indicator**
  - **Sending** → **Retrieve draft from Redis**
- **Version requirements:** typeVersion `3.2`.
- **Failure/edge cases:**
  - If Telegram payload formats differ (e.g., other update types), neither condition matches and nothing happens.
  - If the message is a document/image not in `message.photo`, it won’t match the image route.

#### Node: Show typing indicator
- **Type / Role:** `telegram` — sends chat action (“typing”/activity indicator).
- **Configuration (interpreted):**
  - Operation: `sendChatAction`
  - `chatId: {{$json.message.chat.id}}`
- **Outputs / Connections:**
  - Output → **Build AI request payload**
- **Version requirements:** typeVersion `1.2`.
- **Failure/edge cases:**
  - If triggered from a callback path (it isn’t, by design), `message.chat.id` would be missing.
  - Telegram credential/token issues produce auth failures.

---

### 2.2 AI Request Preparation & Vision+Resume Analysis

**Overview:** Converts the Telegram image to a base64 data URL, incorporates optional caption guidance, references a resume stored in OpenAI Files, and calls OpenAI Responses API with a strict JSON schema. Parses the model output into structured fields.

**Nodes involved:**
- Build AI request payload
- Call OpenAI API
- Parse AI response

#### Node: Build AI request payload
- **Type / Role:** `code` — constructs the OpenAI `/v1/responses` request body.
- **Configuration (interpreted):**
  - Reads the first incoming Telegram item: `$('Telegram Trigger').first()`.
  - Finds the **first binary property** and builds `data:<mime>;base64,<data>` for `input_image`.
  - Reads caption from:
    - `inputItem.json.message.caption` or `inputItem.json.caption`, default `""`.
  - Reads optional `insights` from `inputItem.json.insights`.
  - Hardcoded model: `gpt-5.2`.
  - Hardcoded OpenAI resume file id: `resumeFileId = "YOUR_OPENAI_FILE_ID_HERE"` and validates it starts with `file-`.
  - Uses **json_schema strict mode** with schema `job_apply_payload_v1`:
    - `job_fields` object (title, company, location, seniority, employment_type, language, requirements[], nice_to_have[], salary_range, posted_date, recruiter_email)
    - `email_subject`, `email_body`
  - Includes system instructions:
    - Prioritize caption > image > resume (including recruiter email override).
    - Pick only one job offer (data-related).
    - Output plain text email, ~120–180 words unless overridden by insights.
- **Key expressions/variables:**
  - Uses `$('Telegram Trigger').first()` and binary extraction via `Object.keys(inputItem.binary||{})[0]`.
- **Outputs / Connections:**
  - Output `{ json: { payload } }` → **Call OpenAI API**
- **Version requirements:** Code node typeVersion `2`.
- **Failure/edge cases:**
  - Throws error if no binary image found.
  - Throws error if resume file id not updated to a valid `file-...`.
  - If Telegram provides multiple binaries, it always uses the first key (may not be the intended image).
  - Caption conflicts: caption wins over insights by instruction, but if caption is absent, insights may be ignored by model.

#### Node: Call OpenAI API
- **Type / Role:** `httpRequest` — calls OpenAI Responses API directly.
- **Configuration (interpreted):**
  - POST `https://api.openai.com/v1/responses`
  - Auth: predefined credential type `openAiApi`
  - Sends JSON body: `={{ $json.payload }}`
  - Header: `Content-Type: application/json`
- **Inputs / Outputs:**
  - Input from **Build AI request payload**
  - Output → **Parse AI response**
- **Version requirements:** HTTP Request typeVersion `4.2` (notable because auth + body handling differs across versions).
- **Failure/edge cases:**
  - OpenAI credential missing/invalid (401).
  - Model name mismatch or access issues.
  - Payload too large (image size) leading to 413/400.
  - Timeouts/retries not configured (node `options` empty).

#### Node: Parse AI response
- **Type / Role:** `code` — parses model response text into JSON.
- **Configuration (interpreted):**
  - Reads: `$json.output?.[0]?.content?.[0]?.text || '{}'`
  - `JSON.parse(raw)`; throws a detailed error including the raw text if parsing fails.
- **Outputs / Connections:**
  - Output structured JSON → **Store draft in Redis**
- **Version requirements:** typeVersion `2`.
- **Failure/edge cases:**
  - If OpenAI returns a different structure (API changes or error payload), `raw` may be `'{}'` and downstream nodes may fail due to missing fields.
  - If model outputs invalid JSON despite schema, parsing throws and stops workflow.

---

### 2.3 Draft Persistence & Telegram Confirmation

**Overview:** Persists the generated draft in Redis for 15 minutes, then sends a formatted preview to Telegram with an inline approval button whose callback data is the Redis key.

**Nodes involved:**
- Store draft in Redis
- Send preview to Telegram

#### Node: Store draft in Redis
- **Type / Role:** `redis` — stores the draft for later retrieval on user confirmation.
- **Configuration (interpreted):**
  - Operation: `SET`
  - Key: `{{$json.job_fields.title.replaceAll(" ","")}}`
  - Value: JSON string of `{ Offer, Subject, Recruiter, Email }`
  - TTL: 900 seconds (15 minutes), `expire: true`
- **Inputs / Outputs:**
  - Input from **Parse AI response**
  - Output → **Send preview to Telegram**
- **Version requirements:** typeVersion `1`.
- **Failure/edge cases:**
  - `replaceAll` requires a string; if `title` is empty, key becomes `""` (high collision risk).
  - Two different jobs with same title (or same title after removing spaces) overwrite each other.
  - Redis connection/auth failure will prevent confirmations from working later.
  - If the user clicks after TTL expiry, retrieval fails.

#### Node: Send preview to Telegram
- **Type / Role:** `telegram` — sends the draft preview and an inline keyboard for approval.
- **Configuration (interpreted):**
  - Sends message to: `chatId = {{ $('Telegram Trigger').item.json.message.chat.id }}`
  - Message includes offer, subject, recruiter email, and draft body.
  - Inline keyboard button:
    - Text: “Send The Email”
    - `callback_data`: `{{ $('Parse AI response').item.json.job_fields.title.replaceAll(" ","") }}`
  - Additional fields: `parse_mode: HTML`, `appendAttribution: false`
- **Inputs / Outputs:**
  - Input from **Store draft in Redis**
  - No downstream node from this path (waits for callback trigger later).
- **Version requirements:** typeVersion `1.2`.
- **Failure/edge cases:**
  - Message template uses “*bold*” markers but parse mode is HTML; formatting may not render as intended.
  - If recruiter_email is empty, user may approve sending to an empty address later (Gmail send will fail).
  - If chatId expression is evaluated outside the message route, it would break—but routing prevents that.

---

### 2.4 Confirmation Handling, Resume Download, and Sending (Immediate + Scheduled)

**Overview:** On callback, retrieves the stored draft from Redis, parses it, downloads the resume from Google Drive into binary, then sends the email via Gmail. It also includes a 10-second scheduling path and user status updates in Telegram.

**Nodes involved:**
- Retrieve draft from Redis
- Parse draft data
- Download resume from Drive
- Notify immediate send
- Update Telegram status
- Prepare scheduled send
- Wait before scheduled send
- Download resume for scheduled send
- Send email via Gmail
- Notify scheduled send complete

#### Node: Retrieve draft from Redis
- **Type / Role:** `redis` — fetches the draft using callback_data as key.
- **Configuration (interpreted):**
  - Operation: `GET`
  - Key: `{{$json.callback_query.message.reply_markup.inline_keyboard[0][0].callback_data}}`
- **Inputs / Outputs:**
  - Input from **Route by message type** (Sending path)
  - Output → **Parse draft data**
- **Version requirements:** typeVersion `1`.
- **Failure/edge cases:**
  - If Telegram callback payload changes or inline keyboard structure differs, the key expression breaks.
  - If draft expired (TTL), Redis returns null/empty; downstream JSON.parse will throw.

#### Node: Parse draft data
- **Type / Role:** `code` — converts stored string back into usable fields.
- **Configuration (interpreted):**
  - Parses `i.json.value || i.json.propertyName` (fallback suggests node output differences).
  - Outputs: `{ offer, subject, recruiter, email }`
- **Inputs / Outputs:**
  - Input from **Retrieve draft from Redis**
  - Output → **Download resume from Drive**
- **Version requirements:** typeVersion `2`.
- **Failure/edge cases:**
  - If Redis returned empty, JSON.parse fails.
  - If stored structure changes, field mapping breaks silently (empty values).

#### Node: Download resume from Drive
- **Type / Role:** `googleDrive` — downloads resume PDF for attachment.
- **Configuration (interpreted):**
  - Operation: `download`
  - File ID must be set to your resume PDF (`YOUR_GOOGLE_DRIVE_FILE_ID` placeholder).
  - Output binary property name: `Resume`
  - File name: `Your_Resume.pdf`
- **Inputs / Outputs:**
  - Input from **Parse draft data**
  - Outputs to **Notify immediate send** and **Prepare scheduled send** (fan-out)
- **Version requirements:** typeVersion `3`.
- **Failure/edge cases:**
  - Missing/invalid Drive credentials or file permissions.
  - Wrong file id → 404.
  - Large files may exceed memory limits in n8n binary handling.

#### Node: Notify immediate send
- **Type / Role:** `telegram` — informs user email “was sent” (note: it triggers before the actual Gmail send in this design).
- **Configuration (interpreted):**
  - Text: `✅ Congratulations, your Email to {{ $json.recruiter }} was sent...`
  - Chat ID uses callback context: `{{ $('Telegram Trigger').item.json.callback_query.message.chat.id }}`
- **Inputs / Outputs:**
  - Input from **Download resume from Drive**
  - Output → **Update Telegram status**
- **Version requirements:** typeVersion `1.2`.
- **Failure/edge cases:**
  - This message is logically premature unless Gmail send occurs elsewhere; users may be misled if Gmail fails.
  - If callback_query not present (should be), chatId expression fails.

#### Node: Update Telegram status
- **Type / Role:** `telegram` — announces scheduled send in 10 seconds.
- **Configuration (interpreted):**
  - Sends offer and recruiter from **Parse draft data** via item referencing.
  - Chat ID from callback query.
  - parse_mode HTML.
- **Inputs / Outputs:**
  - Input from **Notify immediate send**
  - Output → (none other than message)
- **Version requirements:** typeVersion `1.2`.
- **Failure/edge cases:**
  - Same HTML vs markdown formatting mismatch risk.

#### Node: Prepare scheduled send
- **Type / Role:** `code` — pass-through to preserve items for wait scheduling branch.
- **Configuration:** `return $input.all();`
- **Inputs / Outputs:**
  - Input from **Download resume from Drive**
  - Output → **Wait before scheduled send**
- **Version requirements:** typeVersion `2`.
- **Failure/edge cases:** Minimal; only fails if input empty.

#### Node: Wait before scheduled send
- **Type / Role:** `wait` — delays processing for 10 seconds.
- **Configuration:** `amount: 10` (seconds by default in Wait node UI).
- **Inputs / Outputs:**
  - Input from **Prepare scheduled send**
  - Output → **Download resume for scheduled send**
- **Version requirements:** typeVersion `1.1`.
- **Failure/edge cases:**
  - Wait nodes depend on n8n execution persistence; if using queue mode or pruning executions, could be interrupted.
  - If workflow is deactivated mid-wait, behavior depends on n8n settings.

#### Node: Download resume for scheduled send
- **Type / Role:** `googleDrive` — downloads the resume again for the scheduled branch.
- **Configuration:** same as “Download resume from Drive” but separate node.
- **Inputs / Outputs:**
  - Input from **Wait before scheduled send**
  - Output → **Send email via Gmail**
- **Version requirements:** typeVersion `3`.
- **Failure/edge cases:** same as the first download node.

#### Node: Send email via Gmail
- **Type / Role:** `gmail` — sends the application email.
- **Configuration (interpreted):**
  - To: `{{$json.recruiter}}`
  - Subject: `{{$json.subject}}`
  - Body: `{{$json.email}}`
  - Email type: `text`
  - Credential: Gmail OAuth2 (“Gmail account ylaf”)
  - **Important:** No attachment field is configured, despite downloading resume into binary `Resume`.
- **Inputs / Outputs:**
  - Input from **Download resume for scheduled send**
  - Output → **Notify scheduled send complete**
- **Version requirements:** typeVersion `2.1`.
- **Failure/edge cases:**
  - If recruiter is empty/invalid → Gmail API error.
  - OAuth token expiry/refresh issues.
  - Missing attachment configuration means resume will **not** be attached unless added in node options.

#### Node: Notify scheduled send complete
- **Type / Role:** `telegram` — confirms completion after Gmail send node.
- **Configuration (interpreted):**
  - Message uses recruiter value via: `{{ $('Download resume for scheduled send').item.json.recruiter }}`
  - Chat ID from callback query.
- **Inputs / Outputs:**
  - Input from **Send email via Gmail**
  - End.
- **Version requirements:** typeVersion `1.2`.
- **Failure/edge cases:**
  - Referencing `Download resume for scheduled send` by name is brittle if node renamed.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / setup notes | — | — | ## How it works … Setup steps (includes OpenAI Files + Drive file_id + Telegram bot + creds) |
| Sticky Note1 | Sticky Note | Documentation for routing block | — | — | ## Input routing … routes image vs callback |
| Sticky Note2 | Sticky Note | Documentation for AI block | — | — | ## AI analysis and email generation … |
| Sticky Note3 | Sticky Note | Documentation for confirmation block | — | — | ## Draft storage and user confirmation … |
| Telegram Trigger | Telegram Trigger | Entry point for Telegram updates | — | Route by message type | ## Input routing … routes image vs callback |
| Route by message type | Switch | Branching: image vs callback | Telegram Trigger | Show typing indicator; Retrieve draft from Redis | ## Input routing … routes image vs callback |
| Show typing indicator | Telegram | UX feedback (chat action) | Route by message type | Build AI request payload | ## AI analysis and email generation … |
| Build AI request payload | Code | Build OpenAI Responses payload (image+resume+schema) | Show typing indicator | Call OpenAI API | ## AI analysis and email generation … |
| Call OpenAI API | HTTP Request | Call OpenAI `/v1/responses` | Build AI request payload | Parse AI response | ## AI analysis and email generation … |
| Parse AI response | Code | Parse strict JSON from model output | Call OpenAI API | Store draft in Redis | ## AI analysis and email generation … |
| Store draft in Redis | Redis | Persist draft for confirmation (TTL) | Parse AI response | Send preview to Telegram | ## Draft storage and user confirmation … |
| Send preview to Telegram | Telegram | Send draft + approval inline button | Store draft in Redis | — | ## Draft storage and user confirmation … |
| Retrieve draft from Redis | Redis | Fetch stored draft on approval click | Route by message type | Parse draft data |  |
| Parse draft data | Code | Parse stored JSON into fields | Retrieve draft from Redis | Download resume from Drive |  |
| Download resume from Drive | Google Drive | Download resume (binary “Resume”) | Parse draft data | Notify immediate send; Prepare scheduled send |  |
| Notify immediate send | Telegram | User notification (immediate path) | Download resume from Drive | Update Telegram status |  |
| Update Telegram status | Telegram | Announces scheduled send in 10s | Notify immediate send | — |  |
| Prepare scheduled send | Code | Pass-through before wait | Download resume from Drive | Wait before scheduled send |  |
| Wait before scheduled send | Wait | Delay execution 10 seconds | Prepare scheduled send | Download resume for scheduled send |  |
| Download resume for scheduled send | Google Drive | Download resume again for send branch | Wait before scheduled send | Send email via Gmail |  |
| Send email via Gmail | Gmail | Send application email | Download resume for scheduled send | Notify scheduled send complete |  |
| Notify scheduled send complete | Telegram | Final confirmation after send | Send email via Gmail | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it: *Send job applications with Telegram, OpenAI, and Gmail*.

2. **Add credentials (required):**
   - **Telegram API**: create a bot via `@BotFather`, copy token, add Telegram credentials in n8n.
   - **OpenAI API**: add OpenAI credential (API key) compatible with HTTP Request node “openAiApi”.
   - **Redis**: add Redis host/port/password (as applicable).
   - **Google Drive OAuth2**: connect to the Drive account that stores your resume PDF.
   - **Gmail OAuth2**: connect to the Gmail account used to send applications.

3. **Upload resume to OpenAI Files API** (outside n8n) and copy the resulting `file-...` id.

4. **Upload resume PDF to Google Drive** and copy its file id.

5. **Node: Telegram Trigger**
   - Add **Telegram Trigger** node.
   - Updates: enable `message` and `callback_query`.
   - Additional fields: enable **Download** = true.
   - This becomes the workflow entry node.

6. **Node: Route by message type (Switch)**
   - Add a **Switch** node.
   - Rule/output 1 (rename output to `Job_Post_Image`): condition `{{ $json.message?.photo }}` → “array not empty”.
   - Rule/output 2 (rename output to `Sending`): condition `{{ $json.callback_query?.message }}` → “object not empty”.
   - Connect **Telegram Trigger → Route by message type**.

7. **Job image branch: Show typing indicator**
   - Add **Telegram** node, operation **sendChatAction**.
   - Chat ID: `{{ $json.message.chat.id }}`
   - Connect `Route by message type (Job_Post_Image) → Show typing indicator`.

8. **Node: Build AI request payload (Code)**
   - Add **Code** node and paste the logic that:
     - Extracts first binary image and builds a data URL.
     - Reads caption and insights.
     - Sets model to `gpt-5.2`.
     - Sets `resumeFileId` to your OpenAI `file-...`.
     - Builds `/v1/responses` payload with `input_image`, `input_file`, and strict JSON schema.
   - Connect **Show typing indicator → Build AI request payload**.

9. **Node: Call OpenAI API (HTTP Request)**
   - Add **HTTP Request** node.
   - Method: POST
   - URL: `https://api.openai.com/v1/responses`
   - Authentication: **Predefined credential type** = OpenAI (`openAiApi`).
   - Send body as JSON: `{{ $json.payload }}`
   - Header: `Content-Type: application/json`
   - Connect **Build AI request payload → Call OpenAI API**.

10. **Node: Parse AI response (Code)**
    - Add **Code** node that reads `output[0].content[0].text`, parses JSON, throws on parse errors.
    - Connect **Call OpenAI API → Parse AI response**.

11. **Node: Store draft in Redis**
    - Add **Redis** node, operation **Set**.
    - Key: `{{ $json.job_fields.title.replaceAll(" ","") }}`
    - Value: stringify `{ Offer, Subject, Recruiter, Email }` from parsed AI output.
    - Enable expire/TTL: 900 seconds.
    - Connect **Parse AI response → Store draft in Redis**.

12. **Node: Send preview to Telegram**
    - Add **Telegram** node (send message).
    - Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
    - Message text: include offer, subject, recruiter, email body.
    - Inline keyboard with one button:
      - Text: “Send The Email”
      - callback_data: `{{ $('Parse AI response').item.json.job_fields.title.replaceAll(" ","") }}`
    - Connect **Store draft in Redis → Send preview to Telegram**.

13. **Callback branch: Retrieve draft from Redis**
    - Add **Redis** node, operation **Get**.
    - Key: `{{ $json.callback_query.message.reply_markup.inline_keyboard[0][0].callback_data }}`
    - Connect `Route by message type (Sending) → Retrieve draft from Redis`.

14. **Node: Parse draft data (Code)**
    - Add **Code** node to `JSON.parse` the redis value and output `{offer, subject, recruiter, email}`.
    - Connect **Retrieve draft from Redis → Parse draft data**.

15. **Node: Download resume from Drive**
    - Add **Google Drive** node, operation **Download**.
    - File ID: your Drive resume file id.
    - Options:
      - File name: `Your_Resume.pdf`
      - Binary property name: `Resume`
    - Connect **Parse draft data → Download resume from Drive**.

16. **Immediate notification (as in the workflow)**
    - Add **Telegram** node “Notify immediate send”.
    - Chat ID: `{{ $('Telegram Trigger').item.json.callback_query.message.chat.id }}`
    - Text: success message referencing `{{ $json.recruiter }}`.
    - Connect **Download resume from Drive → Notify immediate send**.

17. **Scheduled status update**
    - Add **Telegram** node “Update Telegram status”.
    - Chat ID: callback chat id expression above.
    - Text: “scheduled to send in 10s” and include offer/recruiter.
    - Connect **Notify immediate send → Update Telegram status**.

18. **Scheduled send chain**
    - Add **Code** node “Prepare scheduled send” (pass-through).
      - Code: `return $input.all();`
    - Add **Wait** node “Wait before scheduled send” with amount = 10.
    - Add **Google Drive** node “Download resume for scheduled send” (download same file id; binary property `Resume`).
    - Add **Gmail** node “Send email via Gmail”:
      - To: `{{ $json.recruiter }}`
      - Subject: `{{ $json.subject }}`
      - Message: `{{ $json.email }}`
      - Type: Text
      - Select Gmail OAuth2 credential.
      - (If you want the resume attached, configure the node’s attachments to use binary property `Resume`.)
    - Add **Telegram** node “Notify scheduled send complete”:
      - Chat ID: callback chat id expression.
      - Text confirms the send.
    - Connect:
      - **Download resume from Drive → Prepare scheduled send → Wait → Download resume for scheduled send → Send email via Gmail → Notify scheduled send complete**

19. **Activate** the workflow, then send a job posting screenshot to your Telegram bot.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Upload your resume to OpenAI Files API and note the file_id” | Required for the OpenAI `input_file` reference used in the AI request payload |
| “Upload your resume to Google Drive and note the file_id” | Required for both Google Drive download nodes |
| “Create a Telegram bot via @BotFather and save the token” | Telegram Trigger + Telegram nodes require this token |
| “Configure credentials in n8n: Telegram, OpenAI, Redis, Gmail, Google Drive” | Credentials are mandatory for the workflow to run end-to-end |
| “Update the PayloadForReply node with your OpenAI resume file_id” | In this workflow that node is named **Build AI request payload** |
| “Update both Download Resume nodes with your Google Drive file_id” | Nodes: **Download resume from Drive** and **Download resume for scheduled send** |
| “Activate the workflow and send a job post screenshot to your bot” | Operational instruction (how to use) |

