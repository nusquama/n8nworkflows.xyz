Generate visual resumes from Telegram using Google Gemini and BrowserAct

https://n8nworkflows.xyz/workflows/generate-visual-resumes-from-telegram-using-google-gemini-and-browseract-12346


# Generate visual resumes from Telegram using Google Gemini and BrowserAct

## 1. Workflow Overview

**Workflow name:** *Generate visual resumes from Telegram inputs using Google Gemini & BrowserAct*  
**Purpose:** Accept a Telegram message containing (or requesting) resume creation, classify the intent, collect a library of resume template “design inspirations” via BrowserAct, reverse-engineer their visuals using Gemini (image analysis), store those design descriptions in Google Sheets, then use Gemini to (1) choose the best-matching design for the candidate and produce a structured “visual blueprint” and (2) generate a final resume image from a highly detailed prompt, which is sent back to the Telegram user.

**Primary use cases**
- Users paste resume text (or structured resume content) into Telegram and receive a professionally styled resume image.
- Workflow maintains a dynamic “design knowledge base” (Google Sheet) generated from external templates gathered by BrowserAct.

### 1.1 Input Reception (Telegram)
Receives Telegram messages and kicks off AI-based classification.

### 1.2 Intent Classification & Routing
Gemini + structured output parser forces a JSON classification: `Resume` vs `Chat` vs `NoData`, then a Switch routes execution.

### 1.3 Template Harvesting (BrowserAct) + Sheet Reset
If resume intent, BrowserAct workflow runs to retrieve template artifacts (e.g., SVG resume templates), then Google Sheet is cleared and re-populated.

### 1.4 Visual Decomposition (Convert + Gemini Vision) → Knowledge Base
Each template file is downloaded, converted to JPG, analyzed by Gemini (image), and appended to Google Sheets as “Resume Details”.

### 1.5 Candidate-to-Design Matching → Visual Blueprint
The sheet’s design descriptions are aggregated; Gemini Agent (“Resume Writer”) selects the best design and outputs a strict JSON “visual blueprint”.

### 1.6 Prompt Engineering → Image Generation → Delivery
Another Gemini Agent converts the blueprint into a detailed image prompt; Gemini image model generates the resume image and the bot sends it back to the user.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Telegram)

**Overview:** Listens for incoming Telegram messages and provides the raw user text to the AI classifier.  
**Nodes involved:** `User Sends Message to Bot`

#### Node: User Sends Message to Bot
- **Type / role:** `telegramTrigger` — entry point receiving Telegram updates.
- **Config choices:** Listens to `message` updates only.
- **Inputs/outputs:**  
  - **Output →** `Validate user Input`
- **Edge cases / failures:**
  - Telegram credential revoked/expired.
  - Bot not added to chat / missing permissions.
  - Non-text messages (photos/files) will still arrive as updates; downstream assumes `message.text` exists.

---

### Block 2 — Intent Classification & Routing

**Overview:** Uses Gemini (LLM) with a strict structured parser to classify user input into one of three types and routes the workflow accordingly.  
**Nodes involved:** `Validate user Input`, `Google Gemini`, `Structured Output`, `Check For Input Type`

#### Node: Google Gemini
- **Type / role:** `lmChatGoogleGemini` — language model backing the classifier agent.
- **Config choices:** Default options; uses Google Gemini (PaLM) credential.
- **Connections:**  
  - Provides **AI language model** input to `Validate user Input` and `Structured Output`.
- **Edge cases:**
  - API quota, permission, invalid model access.
  - Latency/timeouts.

#### Node: Structured Output
- **Type / role:** `outputParserStructured` — forces classifier output into JSON.
- **Config choices:**  
  - `autoFix: true` to attempt to repair malformed JSON.
  - Schema example: `{"Type":"Resume","Category":"...","Resume":"..."}`
- **Connections:**  
  - **ai_outputParser →** `Validate user Input`
- **Edge cases:**
  - If the model returns non-JSON and autoFix fails, the agent node fails.
  - Schema example is minimal; fields like spacing (`" Resume"`) in prompt rules can cause inconsistent keys.

#### Node: Validate user Input
- **Type / role:** `langchain.agent` — classifies user text into `{Type, Category, Resume}`.
- **Config choices (interpreted):**
  - **Input text:** `{{$json.message.text}}` from the Telegram trigger.
  - **System message:** classification rules:
    - `Type=Resume` only when resume + skills exist; else `NoData`.
    - `Type=Chat` for general conversation.
    - Output must be raw JSON only (no markdown).
  - **Has output parser:** yes (Structured Output).
- **Outputs/variables:**
  - Produces `json.output` containing parsed JSON fields like `output.Type`, `output.Resume`, `output.Category`.
- **Connections:**  
  - **Main →** `Check For Input Type`
- **Edge cases:**
  - Telegram messages without `.text` (voice note, attachment) → expression evaluates to `undefined`, classification may fail or misroute.
  - Prompt contains inconsistent spacing in keys (`" Resume"` vs `"Resume"`) which can create hard-to-handle output keys.

#### Node: Check For Input Type
- **Type / role:** `switch` — routes based on `{{$json.output.Type}}`.
- **Config choices:**
  - Rule 1: equals `"Resume"` → Resume path.
  - Rule 2: equals `"Chat"` → Chat path.
  - Rule 3: equals `"NoData"` → Chat path (to ask for more info).
- **Connections:**
  - **Resume output →** `Run Resume Workflow` and `Notify User` (two parallel outputs on same branch)
  - **Chat output →** `Chatting With User`
  - **NoData output →** `Chatting With User`
- **Edge cases:**
  - If `output.Type` missing or misspelled → no rule matches → workflow stops at switch with no next node.
  - Case sensitivity is enabled; `"resume"` will not match.

---

### Block 3 — User Communication (Immediate Acknowledgement + Chat Replies)

**Overview:** For resume requests, immediately acknowledges the user; for chat/insufficient data, generates a reply via Gemini and sends it.  
**Nodes involved:** `Notify User`, `Chatting With User`, `Google Gemini1`, `Answer the User`

#### Node: Notify User
- **Type / role:** `telegram` — sends acknowledgement message.
- **Config choices:**
  - Text: `"Ok, I will do it please give me a moment."`
  - `parse_mode: HTML`
  - Chat ID expression references Telegram Trigger: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
- **Connections:** None (terminal on that branch).
- **Edge cases:**
  - Chat ID expression can fail if trigger item context differs (e.g., multiple items or different execution path).
  - Telegram API failures.

#### Node: Google Gemini1
- **Type / role:** `lmChatGoogleGemini` — LLM backing the chat agent.
- **Connections:** Provides AI model input to `Chatting With User`.
- **Edge cases:** Same as other Gemini nodes (quota/timeouts).

#### Node: Chatting With User
- **Type / role:** `langchain.agent` — generates a single text reply.
- **Config choices:**
  - Text includes: `Input type : {{$json.output.Type}} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
  - System guidance:
    - If `NoData`: ask user to provide resume with education, skills, name.
    - If `Chat`: produce a normal response.
    - Output raw text only; no markdown/code fences.
- **Connections:**  
  - **Main →** `Answer the User`
- **Edge cases:**
  - If Telegram message text missing, prompt contains `undefined`.
  - Without an output parser, the model may still add formatting; Telegram node will send it as-is.

#### Node: Answer the User
- **Type / role:** `telegram` — sends the chat agent output.
- **Config choices:**
  - Text: `={{ $json.output }}`
  - Chat ID from trigger.
  - `parse_mode: HTML`
- **Edge cases:**
  - If `Chatting With User` outputs under a different key (e.g., `text`), this expression sends blank.
  - HTML parse_mode can break if the model output contains invalid HTML-like characters.

---

### Block 4 — Template Harvesting (BrowserAct) + Preprocessing

**Overview:** Runs a BrowserAct workflow (external automation) that returns a JSON string representing multiple template items, clears the Google Sheet, parses the JSON string into individual items, then starts a batch loop.  
**Nodes involved:** `Run Resume Workflow`, `Clear sheet`, `Splitting BrowserAct Items`, `Loop Over Items`

#### Node: Run Resume Workflow
- **Type / role:** `browserAct` — runs a BrowserAct WORKFLOW to fetch/generate template data.
- **Config choices:**
  - `type: WORKFLOW`
  - `workflowId: 70253132134943736`
  - `timeout: 7200` seconds (2 hours)
  - Workflow config mapping includes `input-JobHero` (removed) and matches columns including `input-Category` (note: schema shows only JobHero with `removed:true`; matchingColumns lists both).
- **Connections:**  
  - **Main →** `Clear sheet`
- **Edge cases:**
  - BrowserAct auth failures / invalid workflow ID.
  - BrowserAct output shape mismatch; downstream expects `$input.first().json.output.string`.
  - Long runtime; n8n execution time limits may still apply depending on instance settings.

#### Node: Clear sheet
- **Type / role:** `googleSheets` (operation: clear) — wipes the target sheet before inserting new “Resume Details”.
- **Config choices:**
  - Document ID and Sheet ID are placeholders using `parameter.SpreadSheetID` / `parameter.SheetID`.
- **Connections:**  
  - **Main →** `Splitting BrowserAct Items`
- **Edge cases:**
  - Wrong sheet/document IDs, missing permissions, OAuth token expired.
  - Clearing the entire sheet removes prior knowledge base; if later steps fail, sheet stays empty.

#### Node: Splitting BrowserAct Items
- **Type / role:** `code` — parses BrowserAct JSON string output into multiple n8n items.
- **Config choices (interpreted):**
  - Reads: `$input.first().json.output.string`
  - Validates presence, JSON parseability, and ensures it’s an array.
  - Returns `[{json: item}, ...]` to “split” into items.
- **Connections:**  
  - **Main →** `Loop Over Items`
- **Edge cases:**
  - If BrowserAct returns a different path (e.g., `output` not containing `string`), node throws an error and stops execution.
  - Malformed JSON string.
  - Parsed data not an array.

#### Node: Loop Over Items
- **Type / role:** `splitInBatches` — iterates over template items (and also triggers post-loop steps due to dual outputs).
- **Config choices:** Default options (batch size defaults to 1 in UI unless changed; not explicitly set here).
- **Connections:**
  - **Output 1 →** `Get Saved Resume` (this is unusual: it runs while looping unless controlled carefully)
  - **Output 2 →** `Download Resume`
- **Important behavior note:** With `SplitInBatches`, output 0 is the current batch; output 1 is “no items left”. In this workflow, `Get Saved Resume` is connected to output index **0** (first output) per connections, and `Download Resume` to output index **1** (second output). That implies:
  - It may try to **get sheet data during each batch** (if connected to main index 0), and only download when “done” (if connected to main index 1), or the reverse depending on how n8n indexes are interpreted in the exported connections.
- **Edge cases:**
  - Miswired outputs can invert the intended logic (download vs finalize).
  - If batch loop never advances correctly, can stall.

---

### Block 5 — Visual Decomposition of Templates (Download → Convert → Analyze → Save)

**Overview:** For each template item, downloads a resume file (expected to be SVG via URL), converts to JPG, uses Gemini image analysis to describe the design, then appends “Resume Details” to Google Sheets.  
**Nodes involved:** `Download Resume`, `SVG To JPG`, `Analyze Resume image`, `Save Data`

#### Node: Download Resume
- **Type / role:** `httpRequest` — downloads the template file from a URL.
- **Config choices:**
  - URL: `={{ $json.Resume }}` (expects each split item has a `Resume` field containing a direct download URL)
- **Connections:**  
  - **Main →** `SVG To JPG`
- **Edge cases:**
  - Missing `Resume` field, invalid URL, 403/404.
  - Large files/timeouts.
  - Response is not SVG or not convertible.

#### Node: SVG To JPG
- **Type / role:** `cloudConvert` — converts downloaded file to JPG.
- **Config choices:** output format `jpg`.
- **Connections:**  
  - **Main →** `Analyze Resume image`
- **Edge cases:**
  - CloudConvert credential expired, quota exceeded.
  - Input binary not correctly passed from HTTP Request (common n8n binary-property mismatch).
  - Unsupported input format.

#### Node: Analyze Resume image
- **Type / role:** `langchain.googleGemini` (image analyze) — vision model extracts design attributes.
- **Config choices:**
  - Operation: analyze image from **binary** input.
  - Model: `models/gemini-3-pro-preview`
  - Extensive instruction prompt focusing on layout, colors, typography, and unique features.
- **Connections:**  
  - **Main →** `Save Data`
- **Edge cases:**
  - If binary property name differs from what this node expects, analysis fails.
  - Vision model access restrictions.
  - Output size limits.

#### Node: Save Data
- **Type / role:** `googleSheets` append — stores Gemini’s design description as a row in the sheet.
- **Config choices:**
  - Appends column `Resume Details` with value: `={{ $json.content.parts[0].text }}`
- **Connections:**  
  - **Main →** `Loop Over Items` (to continue looping)
- **Edge cases:**
  - If Gemini node output structure differs (no `content.parts[0].text`), append writes blank or errors.
  - Sheet missing header `Resume Details`.

---

### Block 6 — Build Design Knowledge Base Input (Read + Aggregate)

**Overview:** Reads the saved “Resume Details” rows from Google Sheets and aggregates them into one payload for the design-selection agent.  
**Nodes involved:** `Get Saved Resume`, `Aggregate Google Sheet Data`

#### Node: Get Saved Resume
- **Type / role:** `googleSheets` read/get — fetches stored rows.
- **Config choices:** Document ID and Sheet ID placeholders.
- **Connections:**  
  - **Main →** `Aggregate Google Sheet Data`
- **Edge cases:**
  - Empty sheet (if prior steps failed or cleared).
  - Permissions/token expiry.

#### Node: Aggregate Google Sheet Data
- **Type / role:** `aggregate` — combines all items into one.
- **Config choices:** `aggregateAllItemData` (collects all rows into a single item, typically under `data`).
- **Connections:**  
  - **Main →** `Resume Writer`
- **Edge cases:**
  - Large sheets → big payload to LLM.
  - Output field naming (`data`) must match what downstream expects.

---

### Block 7 — Candidate-to-Design Matching → Visual Blueprint (Structured JSON)

**Overview:** Uses the aggregated template descriptions as “Design Knowledge Base” and the user’s resume + category to select the best template and output a strict JSON visual blueprint.  
**Nodes involved:** `Resume Writer`, `Google Gemini2`, `Structured Output1`

#### Node: Google Gemini2
- **Type / role:** `lmChatGoogleGemini` — LLM backing the blueprint agent.
- **Connections:** Provides model to `Resume Writer` + `Structured Output1`.
- **Edge cases:** quota/timeouts.

#### Node: Structured Output1
- **Type / role:** `outputParserStructured` — validates and repairs the “visual blueprint” JSON.
- **Config choices:** `autoFix: true`, schema example matches the large “Resume.visual_blueprint” structure.
- **Connections:**  
  - **ai_outputParser →** `Resume Writer`
- **Edge cases:**
  - If the model invents hex codes not present in “Resume Details”, the prompt says not to hallucinate but parser cannot enforce truthfulness.

#### Node: Resume Writer
- **Type / role:** `langchain.agent` — selects design row and outputs blueprint JSON.
- **Config choices (interpreted):**
  - Prompt includes:
    - `Template Examples : {{ $json.data[0]["Resume Details"] }}`
    - `User Resume : {{ $('Validate user Input').first().json.output.Resume }}`
    - `Category : {{ $('Validate user Input').first().json.output.Category }}`
  - System message defines:
    - Industry matching rules (Healthcare/Education/HR → soft; Engineering/Data → structural; Law/Finance → classic).
    - Must extract precise hex codes / fonts / layout rules from template text.
    - Output strictly the defined JSON schema.
  - Has structured output parser.
- **Connections:**  
  - **Main →** `Resume Visualization`
- **Edge cases:**
  - Uses only `data[0]["Resume Details"]` (first row) as “Template Examples”, even though the sheet likely has many rows—this limits matching quality and may be unintended.
  - If `Validate user Input` output keys differ (e.g., `" Resume "`), expressions resolve to null.
  - Large system prompt may increase cost/latency.

---

### Block 8 — Prompt Engineering → Image Generation → Telegram Delivery

**Overview:** Converts the blueprint JSON into a single massive image-generation prompt, generates a resume image using Gemini image model, and sends it as a Telegram photo.  
**Nodes involved:** `Resume Visualization`, `Google Gemini3`, `Structured Output2`, `Generate Resume`, `Send Resume Back`

#### Node: Google Gemini3
- **Type / role:** `lmChatGoogleGemini` — LLM backing the prompt-specialist agent.
- **Connections:** Provides model to `Resume Visualization` + `Structured Output2`.

#### Node: Structured Output2
- **Type / role:** `outputParserStructured` — enforces `{ "Image_Prompt": "..." }`.
- **Config choices:** `autoFix: true`
- **Connections:**  
  - **ai_outputParser →** `Resume Visualization`
- **Edge cases:** If prompt becomes extremely long, the model may truncate; parser won’t prevent truncation.

#### Node: Resume Visualization
- **Type / role:** `langchain.agent` — generates an image prompt JSON for “Nano Banana Pro”.
- **Config choices:**
  - Input text: `={{ $json.output.Resume }}` (expects prior node output contains `output.Resume`, i.e., the blueprint JSON)
  - System message: detailed requirements (canvas, coordinates, hex accuracy, typography, graphical details).
  - Has structured output parser.
- **Connections:**  
  - **Main →** `Generate Resume`
- **Edge cases:**
  - If `Resume Writer` outputs under different key structure, `$json.output.Resume` may be wrong.
  - Prompt may include forbidden/unsupported instructions for the image model (depends on model constraints).

#### Node: Generate Resume
- **Type / role:** `langchain.googleGemini` (image generate) — generates final resume image binary.
- **Config choices:**
  - Model: `models/gemini-3-pro-image-preview` (cached name: “Nano Banana Pro”)
  - Prompt: `={{ $json.output.Image_Prompt }}`
  - Output binary property: `data`
- **Connections:**  
  - **Main →** `Send Resume Back`
- **Edge cases:**
  - Model access, content filtering, quota.
  - If output binary property is not `data`, Telegram node won’t find it.

#### Node: Send Resume Back
- **Type / role:** `telegram` — sends generated image to the user.
- **Config choices:**
  - Operation: `sendPhoto`
  - `binaryData: true` (expects binary in the incoming item)
  - Chat ID expression references trigger chat.
- **Edge cases:**
  - Telegram photo size limits.
  - If binary property isn’t the default expected by Telegram node, you may need to set “Binary Property” (not shown in parameters).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point (Telegram inbound message) | — | Validate user Input | ### 🤖 Step 1: Input Analysis & Routing |
| Validate user Input | langchain.agent | Classify user message into Resume/Chat/NoData (JSON) | User Sends Message to Bot; Google Gemini (AI) + Structured Output (parser) | Check For Input Type | ### 🤖 Step 1: Input Analysis & Routing |
| Google Gemini | lmChatGoogleGemini | LLM for classification agent | — | Validate user Input; Structured Output | ### 🤖 Step 1: Input Analysis & Routing |
| Structured Output | outputParserStructured | Enforce classifier JSON schema | Google Gemini (AI) | Validate user Input | ### 🤖 Step 1: Input Analysis & Routing |
| Check For Input Type | switch | Route flow based on `output.Type` | Validate user Input | Run Resume Workflow; Notify User; Chatting With User | ### 🤖 Step 1: Input Analysis & Routing |
| Notify User | telegram | Acknowledge resume request | Check For Input Type | — | ### 🕷️ Step 2: BrowserAct Automation & Data Prep |
| Run Resume Workflow | browserAct | Run BrowserAct workflow to fetch template data | Check For Input Type | Clear sheet | ### 🕷️ Step 2: BrowserAct Automation & Data Prep |
| Clear sheet | googleSheets | Clear sheet before inserting new template analyses | Run Resume Workflow | Splitting BrowserAct Items | ### 🕷️ Step 2: BrowserAct Automation & Data Prep |
| Splitting BrowserAct Items | code | Parse BrowserAct JSON string into multiple items | Clear sheet | Loop Over Items | ### 🕷️ Step 2: BrowserAct Automation & Data Prep |
| Loop Over Items | splitInBatches | Iterate through template items (batch loop) | Splitting BrowserAct Items; Save Data | Get Saved Resume; Download Resume | ### 👁️ Step 3: Visual Decomposition |
| Download Resume | httpRequest | Download template file from URL | Loop Over Items | SVG To JPG | ### 👁️ Step 3: Visual Decomposition |
| SVG To JPG | cloudConvert | Convert SVG (or other) to JPG | Download Resume | Analyze Resume image | ### 👁️ Step 3: Visual Decomposition |
| Analyze Resume image | langchain.googleGemini (image analyze) | Extract visual design description from image | SVG To JPG | Save Data | ### 👁️ Step 3: Visual Decomposition |
| Save Data | googleSheets | Append “Resume Details” row | Analyze Resume image | Loop Over Items | ### 👁️ Step 3: Visual Decomposition |
| Get Saved Resume | googleSheets | Read stored “Resume Details” rows | Loop Over Items | Aggregate Google Sheet Data | ### 🎨 Step 4: AI Design Generation |
| Aggregate Google Sheet Data | aggregate | Combine all sheet rows into single payload | Get Saved Resume | Resume Writer | ### 🎨 Step 4: AI Design Generation |
| Resume Writer | langchain.agent | Choose best design + output visual blueprint JSON | Aggregate Google Sheet Data; Google Gemini2 (AI) + Structured Output1 (parser) | Resume Visualization | ### 🎨 Step 4: AI Design Generation |
| Google Gemini2 | lmChatGoogleGemini | LLM for blueprint agent | — | Resume Writer; Structured Output1 | ### 🎨 Step 4: AI Design Generation |
| Structured Output1 | outputParserStructured | Enforce blueprint JSON schema | Google Gemini2 (AI) | Resume Writer | ### 🎨 Step 4: AI Design Generation |
| Resume Visualization | langchain.agent | Convert blueprint JSON → detailed image prompt JSON | Resume Writer; Google Gemini3 (AI) + Structured Output2 (parser) | Generate Resume | ### 🎨 Step 4: AI Design Generation |
| Google Gemini3 | lmChatGoogleGemini | LLM for prompt-specialist agent | — | Resume Visualization; Structured Output2 | ### 🎨 Step 4: AI Design Generation |
| Structured Output2 | outputParserStructured | Enforce `{Image_Prompt}` JSON | Google Gemini3 (AI) | Resume Visualization | ### 🎨 Step 4: AI Design Generation |
| Generate Resume | langchain.googleGemini (image generate) | Generate resume image binary | Resume Visualization | Send Resume Back | ### 🎨 Step 4: AI Design Generation |
| Send Resume Back | telegram | Send generated image to user | Generate Resume | — | ### 🎨 Step 4: AI Design Generation |
| Chatting With User | langchain.agent | Generate text response for Chat/NoData | Check For Input Type; Google Gemini1 (AI) | Answer the User | ### 🤖 Step 1: Input Analysis & Routing |
| Google Gemini1 | lmChatGoogleGemini | LLM for chat agent | — | Chatting With User | ### 🤖 Step 1: Input Analysis & Routing |
| Answer the User | telegram | Send chat agent response | Chatting With User | — | ### 🤖 Step 1: Input Analysis & Routing |
| Documentation | stickyNote | Comment block | — | — | ## ⚡ Workflow Overview & Setup + links to https://docs.browseract.com |
| Step 1 Explanation | stickyNote | Comment block | — | — | ### 🤖 Step 1: Input Analysis & Routing |
| Step 2 Explanation | stickyNote | Comment block | — | — | ### 🕷️ Step 2: BrowserAct Automation & Data Prep |
| Step 3 Explanation | stickyNote | Comment block | — | — | ### 👁️ Step 3: Visual Decomposition |
| Step 4 Explanation | stickyNote | Comment block | — | — | ### 🎨 Step 4: AI Design Generation |
| Sticky Note | stickyNote | Comment/link block | — | — | @[youtube](TnObYpgHXSs) |
| Sticky Note1 | stickyNote | Comment block | — | — | ### Google Sheet Headers: Resume Details |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Trigger**
   - Add node: **Telegram Trigger**
   - Updates: **message**
   - Credential: connect your **Telegram bot** credential.

2) **Add Gemini LLM (classifier model provider)**
   - Add node: **Google Gemini Chat Model** (`lmChatGoogleGemini`)
   - Credential: **Google Gemini (PaLM) API**
   - Leave options default.

3) **Add Structured Output Parser (classifier)**
   - Add node: **Structured Output Parser**
   - Enable **Auto-fix**
   - Schema example: `{"Type":"Resume","Category":"extracted_Category","Resume":"User_Resume"}`

4) **Add Agent: “Validate user Input”**
   - Add node: **AI Agent (LangChain)**
   - Text: `{{$json.message.text}}`
   - System message: paste the classification rules (Resume/Chat/NoData) and require raw JSON only.
   - Connect:
     - **AI Language Model** input from **Google Gemini Chat Model**
     - **Output Parser** from **Structured Output Parser**
   - Connect **Telegram Trigger → Validate user Input**

5) **Add Switch: “Check For Input Type”**
   - Add node: **Switch**
   - Add rules on `{{$json.output.Type}}`:
     - equals `Resume`
     - equals `Chat`
     - equals `NoData`
   - Connect **Validate user Input → Switch**

6) **Resume branch: Notify user**
   - Add node: **Telegram** (“Notify User”)
   - Operation: sendMessage
   - Text: `Ok, I will do it please give me a moment.`
   - Chat ID expression: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
   - Connect from Switch “Resume” output to this node (parallel is OK).

7) **Resume branch: Run BrowserAct workflow**
   - Add node: **BrowserAct**
   - Type: `WORKFLOW`
   - Workflow ID: `70253132134943736` (replace with yours)
   - Timeout: `7200`
   - Credential: BrowserAct API key
   - Ensure your BrowserAct account contains the template/workflow referenced (sticky note says **AI Resume Replicant**).
   - Connect Switch “Resume” output → BrowserAct.

8) **Clear Google Sheet**
   - Add node: **Google Sheets**
   - Operation: **Clear**
   - Select Document + Sheet (create a sheet beforehand).
   - Credential: Google Sheets OAuth2
   - Connect BrowserAct → Clear sheet.

9) **Parse BrowserAct output into items**
   - Add node: **Code**
   - Use code that:
     - Reads `$input.first().json.output.string`
     - `JSON.parse()` it
     - Ensures it’s an array
     - Returns `parsedData.map(x => ({json: x}))`
   - Connect Clear sheet → Code.

10) **Loop over parsed items**
   - Add node: **Split In Batches** (“Loop Over Items”)
   - Keep default batch size (or set as needed).
   - Connect Code → Split In Batches.

11) **Download template resume**
   - Add node: **HTTP Request**
   - URL: `{{$json.Resume}}`
   - Ensure it downloads as binary (configure “Response Format”/“Download” accordingly in n8n UI if needed).
   - Connect Split In Batches (batch output) → HTTP Request.

12) **Convert to JPG**
   - Add node: **CloudConvert**
   - Output format: **jpg**
   - Credential: CloudConvert OAuth2
   - Connect HTTP Request → CloudConvert.

13) **Analyze image with Gemini (vision)**
   - Add node: **Google Gemini (Image Analyze)** (`@n8n/n8n-nodes-langchain.googleGemini`)
   - Resource: **image**
   - Operation: **analyze**
   - Input: **binary**
   - Model: `models/gemini-3-pro-preview`
   - Prompt: paste the “Creative Director Expert” instruction text.
   - Connect CloudConvert → Analyze.

14) **Append analysis to Google Sheet**
   - Add node: **Google Sheets**
   - Operation: **Append**
   - Map column **Resume Details** to: `{{$json.content.parts[0].text}}`
   - Connect Analyze → Save Data.

15) **Loop continuation**
   - Connect Save Data → Split In Batches (to continue).
   - Ensure the “when done” output of Split In Batches goes to the next stage (see step 16). Verify in your n8n UI which output is “done” vs “next batch”.

16) **Read saved template descriptions**
   - Add node: **Google Sheets** (“Get Saved Resume”)
   - Operation: **Get Many / Read** (default read)
   - Select same Document + Sheet
   - Connect Split In Batches **done** output → Get Saved Resume.

17) **Aggregate rows**
   - Add node: **Aggregate**
   - Mode: **Aggregate all item data**
   - Connect Get Saved Resume → Aggregate.

18) **Generate visual blueprint (Resume Writer)**
   - Add node: **Gemini Chat Model** (second instance) for this agent if desired (or reuse).
   - Add node: **Structured Output Parser** with the full blueprint schema.
   - Add node: **AI Agent** (“Resume Writer”)
     - Prompt includes:
       - Template examples from aggregated sheet data (ideally all rows; current workflow uses `data[0]`)
       - User resume + category from `Validate user Input`
     - System message: “Senior Resume Architect…” (design matching + strict JSON)
     - Attach model + parser
   - Connect Aggregate → Resume Writer.

19) **Convert blueprint → image prompt**
   - Add node: **Gemini Chat Model** (third instance) for prompt generation.
   - Add node: **Structured Output Parser** with schema `{ "Image_Prompt": "..." }`
   - Add node: **AI Agent** (“Resume Visualization”)
     - Input: blueprint JSON
     - System message: “Nano Banana Pro Prompt Specialist…”
     - Attach model + parser
   - Connect Resume Writer → Resume Visualization.

20) **Generate resume image**
   - Add node: **Google Gemini (Image Generate)** (`@n8n/n8n-nodes-langchain.googleGemini`)
   - Resource: **image**
   - Model: `models/gemini-3-pro-image-preview`
   - Prompt: `{{$json.output.Image_Prompt}}`
   - Set binary output property to `data`
   - Connect Resume Visualization → Generate Resume.

21) **Send image back to Telegram**
   - Add node: **Telegram** (sendPhoto)
   - Enable **Binary Data**
   - Set Chat ID expression to trigger chat id
   - Ensure the node is configured to use the binary property produced by the previous node (often `data`)
   - Connect Generate Resume → Send Resume Back.

22) **Chat/NoData branch**
   - Add node: **Gemini Chat Model** for chat.
   - Add node: **AI Agent** (“Chatting With User”) with system message rules for Chat/NoData.
   - Add node: **Telegram** (“Answer the User”) sending `{{$json.output}}` to chat ID.
   - Connect Switch Chat output → Chat agent → Telegram send message.
   - Connect Switch NoData output → Chat agent → Telegram send message.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Workflow Overview & Setup” requirements: Credentials needed: Telegram, Google Gemini (PaLM), Google Sheets, CloudConvert, BrowserAct. Mandatory BrowserAct template: **AI Resume Replicant**. | Internal sticky note (“Documentation”) |
| BrowserAct help links: How to find API key & workflow ID; connect n8n; customize templates. | https://docs.browseract.com |
| YouTube reference | @[youtube](TnObYpgHXSs) |
| Google Sheet must have header column: `Resume Details` | Sticky note “Google Sheet Headers” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.