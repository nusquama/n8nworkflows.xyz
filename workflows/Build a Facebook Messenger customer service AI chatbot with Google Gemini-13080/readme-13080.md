Build a Facebook Messenger customer service AI chatbot with Google Gemini

https://n8nworkflows.xyz/workflows/build-a-facebook-messenger-customer-service-ai-chatbot-with-google-gemini-13080


# Build a Facebook Messenger customer service AI chatbot with Google Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Build a Facebook Messenger customer service AI chatbot with Google Gemini  
**Purpose:** Receive Facebook Messenger events for a Facebook Page, validate and filter incoming user text messages, generate a contextual response using Google Gemini with short-term conversation memory, normalize the response for Messenger constraints, and send it back via Facebook Graph API. Optionally sends an image/video attachment if the AI output contains media URLs.

### 1.1 Webhook Reception & Validation
- Receives both **GET** (Webhook verification) and **POST** (Messenger events).
- Immediately responds with the verification challenge for GET, and acknowledges POST.
- Filters to only **text messages** and blocks **echo/self messages** (to prevent bot loops).

### 1.2 Context Extraction & Messenger Activity Indicators
- Extracts `user_id`, `page_id`, `message`, `timestamp`, and injects `page_access_token`.
- Sends **mark_seen** and **typing_on** actions to improve UX while AI responds.

### 1.3 AI Processing with Memory (Gemini + Buffer Window)
- Maintains a **per-user-per-page** short context window (last 10 messages).
- Uses a **LangChain Agent** with a custom system prompt (“Jenix” persona) and Gemini Flash model.

### 1.4 Response Formatting & Delivery (Text + Optional Media)
- Cleans markdown-like formatting and truncates to stay within Messenger limits.
- Sends the message back to Messenger.
- Checks AI output for `photo` or `video` URLs and sends attachments if present.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook & Validation

**Overview:** Handles Facebook webhook verification and message ingestion, then filters events to process only real user text messages (not bot echoes).

**Nodes involved:**
- Facebook Webhook
- Confirm Webhook
- Is message
- From user?

#### Node: Facebook Webhook
- **Type / Role:** `Webhook` trigger; entry point for Facebook events.
- **Configuration (interpreted):**
  - Path: `nguyenthieutoan-facebook-page-1234`
  - **Multiple methods enabled** (GET + POST)
  - Response mode: **Response Node** (delegates response to `Respond to Webhook`)
- **Key data used:** Expects Facebook payload in `body` and verification params in `query`.
- **Outputs / Connections:**
  - Output 1 → `Confirm Webhook` (used for verification / immediate response)
  - Output 2 → `Confirm Webhook` and → `Is message` (for POST events)
- **Failure / Edge cases:**
  - If Facebook can’t reach a public URL, verification and delivery fail.
  - Incorrect webhook path or missing FB subscription fields will prevent events.
  - If response isn’t returned quickly, Facebook may retry events.

#### Node: Confirm Webhook
- **Type / Role:** `Respond to Webhook`; returns response to Facebook.
- **Configuration (interpreted):**
  - HTTP 200
  - `Content-Type: text/plain`
  - Respond with text: `{{$json.query["hub.challenge"]}}`
- **Behavior:**
  - For **GET verification**, returns the `hub.challenge` Facebook expects.
  - For **POST events**, this expression may be empty; still returns 200 (ack).
- **Inputs / Outputs:**
  - Input from `Facebook Webhook`
  - No downstream nodes
- **Failure / Edge cases:**
  - If used for POST acknowledgements, returning empty body is usually fine (200 OK), but ensure Facebook accepts your response pattern.
  - If verification token validation is required, it’s not implemented here (Facebook commonly uses `hub.verify_token`).

#### Node: Is message
- **Type / Role:** `IF`; filters only text messages.
- **Condition:** checks existence of:
  - `{{$json.body.entry[0].messaging[0].message.text}}`
- **Outputs / Connections:**
  - True → `From user?`
  - False → stops (no further processing)
- **Failure / Edge cases:**
  - Many Messenger events are not text (postbacks, attachments, delivery, read receipts). Those will be ignored.
  - If payload arrays differ (multiple entries / messaging events), indexing `[0]` may miss additional messages.

#### Node: From user?
- **Type / Role:** `IF`; blocks echo/self messages.
- **Condition (as configured):**
  - `{{$json.body.entry[0].id}}` **equals** `{{$json.body.entry[0].messaging[0].recipient.id}}`
- **Interpretation / Caveat:**
  - This is intended to prevent bot echoes, but the configured comparison effectively checks if the **entry page id** equals the **recipient id** (often true). It does **not** explicitly check `message.is_echo`.
  - A more robust echo check is usually `messaging[0].message.is_echo !== true`.
- **Outputs / Connections:**
  - True → `Set Context`
  - False → stops
- **Failure / Edge cases:**
  - Risk of letting echo messages through (or blocking valid ones) depending on event structure.
  - If Facebook changes payload or additional messaging events exist, the condition may behave unexpectedly.

---

### Block 2 — AI Processing & Memory

**Overview:** Extracts key identifiers, sends “seen/typing” actions, and generates an answer using Gemini with a short conversation memory window.

**Nodes involved:**
- Set Context
- Seen
- Typing
- Conversation Memory
- Gemini Flash
- Process Merged Message

#### Node: Set Context
- **Type / Role:** `Set`; normalizes and stores core fields for downstream expressions.
- **Configuration (interpreted):**
  - `user_id` = sender id
  - `page_id` = recipient/page id
  - `message` = incoming text
  - `timestamp` = entry time
  - `page_access_token` = hardcoded placeholder string (`YOUR_PAGE_ACCESS_TOKEN_HERE (EEa...)`)
- **Outputs / Connections:**
  - → `Seen`
- **Key expressions:**
  - `{{$json.body.entry[0].messaging[0].sender.id}}`
  - `{{$json.body.entry[0].messaging[0].recipient.id}}`
  - `{{$json.body.entry[0].messaging[0].message.text}}`
- **Failure / Edge cases:**
  - Hardcoding `page_access_token` is risky; use credentials/environment variables if possible.
  - If incoming event doesn’t contain `message.text`, upstream filter should block it—but indexing issues can still throw expression errors in unusual payloads.

#### Node: Seen
- **Type / Role:** `HTTP Request`; sends Messenger “mark_seen”.
- **Configuration (interpreted):**
  - POST `https://graph.facebook.com/v24.0/{{page_id}}/messages`
  - Query: `access_token={{page_access_token}}`
  - JSON body includes `sender_action: "mark_seen"` and `recipient.id`
  - Timeout: 1000 ms
  - **onError:** continue (won’t stop the workflow)
- **Outputs / Connections:**
  - → `Typing`
- **Failure / Edge cases:**
  - Graph API auth errors (expired/invalid token, missing permissions).
  - Timeout is short; network latency may cause frequent timeouts (but errors won’t stop flow).

#### Node: Typing
- **Type / Role:** `HTTP Request`; sends Messenger “typing_on”.
- **Configuration (interpreted):**
  - Same endpoint and auth as `Seen`
  - JSON body includes `sender_action: "typing_on"`
  - Timeout: 1000 ms
  - **onError:** continue
- **Outputs / Connections:**
  - → `Process Merged Message`
- **Failure / Edge cases:** Same as `Seen`.

#### Node: Conversation Memory
- **Type / Role:** LangChain `Memory Buffer Window`; short-term memory for chat context.
- **Configuration (interpreted):**
  - Session key (custom): `{{user_id}} {{page_id}}` (concatenated)
  - Context window length: 10 messages
- **Connections (AI):**
  - Provides `ai_memory` input to `Process Merged Message`
- **Failure / Edge cases:**
  - Memory is ephemeral depending on n8n execution/memory settings; not a durable datastore.
  - If multiple concurrent executions for the same user occur, ordering/context may be inconsistent.

#### Node: Gemini Flash
- **Type / Role:** LangChain `lmChatGoogleGemini`; LLM provider for the agent.
- **Configuration (interpreted):**
  - Uses Google Gemini chat model defaults (node options empty).
  - Requires Google Gemini / PaLM API credentials (configured credential shown in workflow).
- **Connections (AI):**
  - Provides `ai_languageModel` input to `Process Merged Message`
- **Failure / Edge cases:**
  - Credential/auth failures, quota limits, model availability, safety filters.
  - Latency/timeouts depending on model and prompt size.

#### Node: Process Merged Message
- **Type / Role:** LangChain `Agent`; orchestrates prompt + memory + model to generate response.
- **Input text:** `{{$('Set Context').item.json.message}}`
- **System message:** A detailed persona and ruleset:
  - Persona: “Jenix” for Nguyễn Thiệu Toàn / GenStaff Company
  - Language mirroring (Vietnamese/English rules)
  - Output constraints: plain text, concise, under 1800 chars
  - Includes current timestamp: `Date/time: {{$now}}`
- **Connections:**
  - Main output → `Cut if reply more than 2000 characters`
  - AI inputs:
    - `ai_languageModel` from `Gemini Flash`
    - `ai_memory` from `Conversation Memory`
- **Failure / Edge cases:**
  - Agent output schema is not strictly enforced; downstream nodes assume certain fields exist (e.g., `.output`, `.output.photo`, `.output.video`).
  - If the agent returns non-text, empty output, or tool-like structures, formatting/switch logic may not behave as intended.

---

### Block 3 — Format & Send Response

**Overview:** Normalizes the AI response for Messenger (removes markdown, truncates), sends it as text, then optionally sends image/video attachments if URLs are present in the agent output.

**Nodes involved:**
- Cut if reply more than 2000 characters
- Send Text
- Switch
- Send image
- Send video

#### Node: Cut if reply more than 2000 characters
- **Type / Role:** `Code`; sanitizes and truncates AI output for Messenger.
- **Logic (interpreted):**
  - Reads AI text from `output` or `text`
  - Trims whitespace
  - Truncates to 1900 chars (adds `...`)
  - Removes markdown formatting:
    - Bold `**x**`, italic `*x*`
    - Code blocks and inline code backticks
  - Returns `{ page_message: clean_text }`
- **Connections:**
  - → `Send Text`
- **Failure / Edge cases:**
  - If upstream returns neither `output` nor `text`, sends empty string.
  - Regex stripping is “lossy” (may remove meaningful characters in some languages/formatting).
  - Does not split long replies into multiple messages; it truncates.

#### Node: Send Text
- **Type / Role:** `HTTP Request`; sends the text response to Messenger via Graph API.
- **Configuration (interpreted):**
  - POST `https://graph.facebook.com/v24.0/{{page_id}}/messages`
  - Query: `access_token={{page_access_token}}`
  - JSON body:
    - `recipient.id = user_id`
    - `messaging_type = "RESPONSE"`
    - `message.text = JSON.stringify($json.page_message)` (string-escapes content)
    - `metadata = "bot_rep"`
- **Connections:**
  - → `Switch`
- **Failure / Edge cases:**
  - Using `JSON.stringify` for `message.text` may cause quotes to be included in the final text depending on evaluation context. In many cases, you want the raw string (without extra quotes).
  - Graph API may reject messages that violate policy or formatting constraints.
  - Token permission issues: requires appropriate Messenger send permissions.

#### Node: Switch
- **Type / Role:** `Switch`; routes to optional media sending.
- **Rules (interpreted):**
  - Output “Image exist?” if `Process Merged Message.output.photo` contains `http`
  - Output “Video exist?” if `Process Merged Message.output.video` contains `http`
- **Connections:**
  - “Image exist?” → `Send image`
  - “Video exist?” → `Send video`
- **Failure / Edge cases:**
  - If `output.photo`/`output.video` are undefined, `contains` checks may evaluate false or error depending on n8n strictness; here it uses strict type validation, so undefined handling matters.
  - No default output path configured; if neither condition matches, the workflow ends after `Send Text`.

#### Node: Send image
- **Type / Role:** `HTTP Request`; sends an image attachment by URL.
- **Configuration (interpreted):**
  - POST Graph API `/messages`
  - Body: attachment type `image`, payload `url` from `Process Merged Message.output.photo`, `is_reusable: true`
- **Failure / Edge cases:**
  - URL must be publicly accessible by Facebook.
  - Unsupported or slow URLs may fail; file size/type restrictions apply.

#### Node: Send video
- **Type / Role:** `HTTP Request`; sends a video attachment by URL.
- **Configuration (interpreted):**
  - Same as image, but attachment type `video` and URL from `output.video`
- **Failure / Edge cases:**
  - Same as image; video constraints are stricter (size/format).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Main Overview | Sticky Note | Documentation / overview |  |  | ## Try It Out! … Author & Support … LICENCE … (includes links: https://nguyenthieutoan.com, https://n8n.io/creators/nguyenthieutoan) |
| Facebook Webhook | Webhook | Receives FB webhook GET/POST |  | Confirm Webhook; Is message | ## 1. Webhook & Validation … (link: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook) |
| Confirm Webhook | Respond to Webhook | Returns hub.challenge / acknowledges | Facebook Webhook |  | ## 1. Webhook & Validation … (link: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook) |
| Is message | IF | Filter only text messages | Facebook Webhook | From user? | ## 1. Webhook & Validation … |
| From user? | IF | Attempt to block bot echo / non-user events | Is message | Set Context | ## 1. Webhook & Validation … |
| Set Context | Set | Extracts ids, message, token | From user? | Seen | ## 2. AI Processing & Memory … (Gemini link + Memory link) / ## ⚠️ Important! Replace token … (link: https://developers.facebook.com) |
| Seen | HTTP Request | mark_seen sender action | Set Context | Typing | ## 2. AI Processing & Memory … |
| Typing | HTTP Request | typing_on sender action | Seen | Process Merged Message | ## 2. AI Processing & Memory … |
| Conversation Memory | LangChain Memory Buffer Window | Keeps last 10 turns per session key |  | Process Merged Message (ai_memory) | ## 2. AI Processing & Memory … (Memory link: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorybufferwindow/) |
| Gemini Flash | LangChain Chat Model (Google Gemini) | LLM for agent |  | Process Merged Message (ai_languageModel) | ## 2. AI Processing & Memory … (Gemini link: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.lmchatgooglegemini/) |
| Process Merged Message | LangChain Agent | Generates response using prompt + memory + LLM | Typing | Cut if reply more than 2000 characters | ## 2. AI Processing & Memory … (Customization note about system prompt) |
| Cut if reply more than 2000 characters | Code | Strip markdown, truncate, map to page_message | Process Merged Message | Send Text | ## 3. Format & Send Response … (link: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| Send Text | HTTP Request | Sends message text to Messenger | Cut if reply more than 2000 characters | Switch | ## 3. Format & Send Response … |
| Switch | Switch | Routes optional media sending | Send Text | Send image; Send video |  |
| Send image | HTTP Request | Sends image attachment by URL | Switch |  |  |
| Send video | HTTP Request | Sends video attachment by URL | Switch |  |
| Sticky Note - Section 1 | Sticky Note | Documentation: webhook/validation |  |  | ## 1. Webhook & Validation … (link: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook) |
| Sticky Note - Section 2 | Sticky Note | Documentation: AI + memory |  |  | ## 2. AI Processing & Memory … (links to Gemini + Memory) |
| Sticky Note - Section 3 | Sticky Note | Documentation: formatting + sending |  |  | ## 3. Format & Send Response … (link: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| Sticky Note | Sticky Note | Token warning |  |  | ## ⚠️ Important! … (link: https://developers.facebook.com) |
| Sticky Note1 | Sticky Note | Limitations + upgrade links |  |  | ## Limitations & When to Upgrade … (links: https://n8n.io/workflows/9192, https://n8n.io/workflows/11920) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add Webhook Trigger**
   - Node: **Webhook**
   - Method: enable **multiple methods** (GET and POST)
   - Path: set to something unique (e.g., `nguyenthieutoan-facebook-page-1234`)
   - Response mode: **Using “Respond to Webhook” node**

3. **Add Respond to Webhook**
   - Node: **Respond to Webhook**
   - Respond with: **Text**
   - Response body expression: `{{$json.query["hub.challenge"]}}`
   - Response code: **200**
   - Header: `Content-Type: text/plain`
   - Connect **Webhook → Respond to Webhook** (for GET verification and also to quickly ack POST if you keep the same structure).

4. **Add IF node to filter text messages**
   - Node: **IF**
   - Condition: “String exists”
   - Value: `{{$json.body.entry[0].messaging[0].message.text}}`
   - Connect **Webhook (POST output) → IF**

5. **Add IF node to block echo/self messages**
   - Node: **IF**
   - Configure as in the workflow (page id equals recipient id), or preferably improve:
     - Recommended robust check: ensure `message.is_echo` is not true.
   - Connect **Is message (true) → From user?**

6. **Add Set node for context**
   - Node: **Set**
   - Add fields:
     - `user_id` = `{{$json.body.entry[0].messaging[0].sender.id}}`
     - `page_id` = `{{$json.body.entry[0].messaging[0].recipient.id}}`
     - `message` = `{{$json.body.entry[0].messaging[0].message.text}}`
     - `timestamp` = `{{$json.body.entry[0].time}}`
     - `page_access_token` = your Page access token (prefer environment variable/credential)
   - Connect **From user? (true) → Set Context**

7. **Add HTTP Request “Seen”**
   - Node: **HTTP Request**
   - Method: POST
   - URL: `https://graph.facebook.com/v24.0/{{$('Set Context').item.json.page_id}}/messages`
   - Query parameter: `access_token = {{$('Set Context').item.json.page_access_token}}`
   - JSON body:
     - `recipient.id = {{$('Set Context').item.json.user_id}}`
     - `sender_action = "mark_seen"`
   - Options: Timeout 1000ms; Error handling: continue
   - Connect **Set Context → Seen**

8. **Add HTTP Request “Typing”**
   - Same endpoint/auth as Seen
   - JSON body: `sender_action = "typing_on"`
   - Timeout 1000ms; Error handling: continue
   - Connect **Seen → Typing**

9. **Add LangChain Memory Buffer Window**
   - Node: **Memory Buffer Window**
   - Session ID type: custom key
   - Session key expression: `{{$('Set Context').item.json.user_id}} {{$('Set Context').item.json.page_id}}`
   - Context window length: **10**

10. **Add Google Gemini Chat Model**
   - Node: **Google Gemini Chat Model** (Gemini Flash)
   - Create/attach credentials for Gemini API (Google PaLM/Gemini in n8n):
     - Add credential in n8n: API key
   - Keep default options unless you need temperature/max tokens adjustments.

11. **Add LangChain Agent**
   - Node: **AI Agent**
   - Text input: `{{$('Set Context').item.json.message}}`
   - Prompt type: define
   - System message: paste your persona/rules (use the provided “Jenix” prompt if desired)
   - Connect AI inputs:
     - **Gemini Flash** → Agent `ai_languageModel`
     - **Conversation Memory** → Agent `ai_memory`
   - Connect **Typing → Process Merged Message**

12. **Add Code node to normalize output**
   - Node: **Code**
   - Implement logic:
     - read AI response from `output` or `text`
     - trim, truncate to 1900, strip markdown
     - output `{ page_message: "..." }`
   - Connect **Agent → Code**

13. **Add HTTP Request “Send Text”**
   - Method: POST
   - URL: `https://graph.facebook.com/v24.0/{{$('Set Context').first().json.page_id}}/messages`
   - Query: `access_token = {{$('Set Context').first().json.page_access_token}}`
   - JSON body:
     - `recipient.id = {{$('Set Context').first().json.user_id}}`
     - `messaging_type = "RESPONSE"`
     - `message.text = {{$json.page_message}}` (recommended to avoid extra quoting)
     - `metadata = "bot_rep"`
   - Connect **Code → Send Text**

14. **Add Switch for optional media**
   - Node: **Switch**
   - Rule 1 (Image exist?): check if `{{$('Process Merged Message').item.json.output.photo}}` contains `http`
   - Rule 2 (Video exist?): check if `{{$('Process Merged Message').item.json.output.video}}` contains `http`
   - Connect **Send Text → Switch**

15. **Add HTTP Request “Send image”**
   - Same endpoint/auth as Send Text
   - JSON body: attachment type `image`, payload url from `output.photo`
   - Connect **Switch (Image exist?) → Send image**

16. **Add HTTP Request “Send video”**
   - Same endpoint/auth as Send Text
   - JSON body: attachment type `video`, payload url from `output.video`
   - Connect **Switch (Video exist?) → Send video**

17. **Facebook configuration**
   - In Facebook Developer settings:
     - Configure Webhook callback URL to your n8n webhook test/production URL.
     - Provide verify token (if you implement it).
     - Subscribe to `messages` events for your page.
   - Ensure your Page access token has required permissions to send messages.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Webhook node documentation | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook |
| HTTP Request node documentation | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest |
| Google Gemini Chat Model node documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.lmchatgooglegemini/ |
| Memory Buffer Window documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorybufferwindow/ |
| Replace `YOUR_PAGE_ACCESS_TOKEN_HERE` with your real Page Access Token | https://developers.facebook.com |
| Author: Nguyễn Thiệu Toàn (Jay Nguyen) – Website | https://nguyenthieutoan.com |
| Author – n8n creator profile | https://n8n.io/creators/nguyenthieutoan |
| Upgrade workflow: Smart message batching | https://n8n.io/workflows/9192 |
| Upgrade workflow: Smart human takeover | https://n8n.io/workflows/11920 |
| Limitations noted in workflow: no batching, no human takeover, basic memory, basic formatting/truncation | Included in “Limitations & When to Upgrade” sticky note content |

