Create an AI content agent with Telegram, Gemini, and Blotato (no-code)

https://n8nworkflows.xyz/workflows/create-an-ai-content-agent-with-telegram--gemini--and-blotato--no-code--13409


# Create an AI content agent with Telegram, Gemini, and Blotato (no-code)

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Create an AI content agent with Telegram, Gemini, and Blotato (no-code)  
**Workflow name (in JSON):** AI Agent Content

**Purpose:**  
Turns a Telegram message (topic/content request) into an AI-orchestrated pipeline that can research a source, generate media assets (infographic or slideshow video), publish content to Instagram and LinkedIn via Blotato, and then send a confirmation back to Telegram.

**Target use cases:**
- One-message content operations (research → create visuals → publish)
- Multi-platform posting automation driven by conversational AI
- Async-ish orchestration where the agent decides which “tool” to call next (Blotato actions)

### Logical blocks
1.1 **Trigger & Input Capture (Telegram)**  
Receives incoming Telegram messages and passes user text/chat metadata downstream.

1.2 **AI Core Orchestration (LangChain Agent + Gemini + Memory)**  
Interprets user intent, maintains conversational context, and routes tasks by invoking “tools” (Blotato nodes).

1.3 **Research Engine (Blotato Sources)**  
Creates and retrieves structured research from a URL/source, returning data usable for content generation.

1.4 **Media Engine (Blotato Visual Generation)**  
Generates either an infographic or slideshow video using Blotato templates; retrieves final rendered asset.

1.5 **Distribution (Blotato Publishing)**  
Publishes the generated content to Instagram and LinkedIn.

1.6 **Response (Telegram)**  
Escapes the agent output for MarkdownV2 and sends it back to the originating Telegram chat.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Input Capture (Telegram)

**Overview:**  
This block starts the workflow when a Telegram bot receives a message. It provides the message text and chat ID used throughout the flow (especially for memory session scoping and responding).

**Nodes involved:**
- Telegram Bot Trigger

#### Node: Telegram Bot Trigger
- **Type / role:** `n8n-nodes-base.telegramTrigger` — webhook-based Telegram update trigger.
- **Configuration (interpreted):**
  - Listens to **updates: `message`** only (not callbacks, inline queries, etc.).
- **Key data produced:**
  - `message.text` (user’s request)
  - `message.chat.id` (used for memory session and response routing)
- **Connections:**
  - **Output (main)** → AI Content Orchestrator (main)
- **Credentials / requirements:**
  - Telegram API credential (“Telegram GiangxAI”) with bot token configured in n8n.
- **Common edge cases / failures:**
  - Bot token invalid/revoked → authentication errors.
  - Webhook not set / blocked by network → trigger never fires.
  - Non-text messages (photos, stickers) → `message.text` may be undefined (agent input expression can fail or become empty).

---

### 2.2 AI Core Orchestration (Agent + Gemini + Memory)

**Overview:**  
This is the “brain” of the workflow. It receives Telegram text, uses Gemini as the chat model, stores conversation history keyed by chat ID, and can invoke Blotato tool nodes to perform research, media generation, asset retrieval, and publishing.

**Nodes involved:**
- AI Content Orchestrator
- Gemini Chat Model
- Conversation Memory Store

#### Node: AI Content Orchestrator
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LangChain-style agent that can call tools and produce a final response.
- **Configuration (interpreted):**
  - **User input text:** `={{ $json.message.text }}`
  - **Prompt type:** “define” (custom prompt behavior)
  - **System message:** “You are a helpful assistant”
- **Tools available to the agent (via ai_tool connections):**
  - Source Collector, Source Retriever
  - Infographic Generator, Slideshow Video Generator, Visual Asset Retriever
  - Instagram Publisher, LinkedIn Publisher
- **Memory:**
  - Connected to Conversation Memory Store via **ai_memory**.
- **Language model:**
  - Connected to Gemini Chat Model via **ai_languageModel**.
- **Connections:**
  - **Input (main)** from Telegram Bot Trigger
  - **Output (main)** → Telegram Response Sender
  - **Tool outputs/inputs:** interacts with Blotato tool nodes through the agent tool interface (not standard main wiring).
- **Version-specific notes:**
  - Uses agent node **typeVersion 3**; tool/memory wiring relies on the newer LangChain integration ports (`ai_tool`, `ai_memory`, `ai_languageModel`).
- **Edge cases / failures:**
  - If `message.text` is missing, the agent may receive `null`/empty string.
  - Tool schema mismatch: `$fromAI()` fields used by tools must be provided by the agent; otherwise Blotato calls may fail due to missing required inputs.
  - Model/tool latency: long-running Blotato renders may exceed expected time if the agent doesn’t implement waiting/polling logic (no explicit Wait node exists in this workflow).

#### Node: Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides Gemini chat completion capability to the agent.
- **Configuration (interpreted):**
  - Default options (no explicit temperature/max tokens shown).
- **Connections:**
  - **Output (ai_languageModel)** → AI Content Orchestrator
- **Credentials / requirements:**
  - Google PaLM/Gemini credential (“api google”).
- **Edge cases / failures:**
  - Invalid API key / billing issues / quota exceeded.
  - Safety policy refusals from Gemini (agent may return blocked/empty content).
  - Timeouts on large context windows or long responses.

#### Node: Conversation Memory Store
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — sliding window chat memory.
- **Configuration (interpreted):**
  - **Session key (custom):** `={{ $json.message.chat.id }}`
  - **Window length:** 30 (keeps last ~30 turns/messages depending on implementation)
- **Connections:**
  - **Output (ai_memory)** → AI Content Orchestrator
- **Edge cases / failures:**
  - If incoming data lacks `message.chat.id`, session scoping breaks.
  - Memory growth is bounded by window size, but still may contain sensitive info—consider retention policy.

---

### 2.3 Research Engine (Blotato)

**Overview:**  
These nodes expose Blotato “source” operations as tools to the agent: create a source task from a URL and retrieve the structured result later by source ID.

**Nodes involved:**
- Source Collector
- Source Retriever

#### Node: Source Collector
- **Type / role:** `@blotato/n8n-nodes-blotato.blotatoTool` — creates a Blotato “source” job.
- **Configuration (interpreted):**
  - **Resource:** `source`
  - **Source URL:** `={{ $fromAI('URL', ``, 'string') }}`
    - The agent is expected to supply a URL field named **URL** when calling this tool.
- **Connections:**
  - Available to AI Content Orchestrator via **ai_tool**.
- **Credentials / requirements:**
  - Blotato API credential (“Blotato GiangxAI”).
- **Edge cases / failures:**
  - Missing/invalid URL (agent didn’t pass `URL`, or it’s malformed).
  - Source content behind paywalls / blocked robots → poor extraction or failures.
  - Asynchronous processing: creation may return a job/source ID before extraction completes.

#### Node: Source Retriever
- **Type / role:** `@blotato/n8n-nodes-blotato.blotatoTool` — fetches an existing source by ID.
- **Configuration (interpreted):**
  - **Resource:** `source`
  - **Operation:** `get`
  - **Source ID:** `={{ $fromAI('Source_ID', ``, 'string') }}`
- **Connections:**
  - Available to AI Content Orchestrator via **ai_tool**.
- **Edge cases / failures:**
  - Wrong/expired Source ID.
  - Retrieval before processing completion → incomplete/empty structured results.
  - Rate limits or transient 5xx from Blotato.

---

### 2.4 Media Engine (Blotato Visual Generation)

**Overview:**  
Enables the agent to generate visual content through predefined Blotato templates and then retrieve the rendered output by video ID.

**Nodes involved:**
- Infographic Generator
- Slideshow Video Generator
- Visual Asset Retriever

#### Node: Infographic Generator
- **Type / role:** `@blotato/n8n-nodes-blotato.blotatoTool` — generates a “video” resource using a template (used here for infographic generation).
- **Configuration (interpreted):**
  - **Resource:** `video`
  - **Prompt:** `={{ $fromAI('Prompt', ``, 'string') }}`
  - **Template:** ID `29ebb2bd-02b7-4317-8bb8-c30eb938e47c`  
    Name: “Generate an infographic carved into a wooden trail marker in a serene nature setting using AI image generation.”
  - **Template inputs:**
    - `description` = `$fromAI('Infographic_Description', ..., 'string')`
    - `footerText` = `$fromAI('Footer_CTA_Text', ..., 'string')`
- **Connections:**
  - Available to AI Content Orchestrator via **ai_tool**.
- **Edge cases / failures:**
  - Required inputs missing (depending on template validation on Blotato side).
  - Prompt/inputs too long or unsafe → generation failure.
  - Rendering time can be long; without explicit polling/wait nodes, agent must handle “not ready yet” states.

#### Node: Slideshow Video Generator
- **Type / role:** `@blotato/n8n-nodes-blotato.blotatoTool` — generates an image slideshow “video” using a template.
- **Configuration (interpreted):**
  - **Resource:** `video`
  - **Prompt:** `={{ $fromAI('Prompt', ``, 'string') }}`
  - **Template:** `/base/v2/image-slideshow/5903b592-1255-43b4-b9ac-f8ed7cbf6a5f/v1`  
    Name: “Image Slideshow with Text Overlays”
  - **Template inputs (agent-provided via `$fromAI`) include:**
    - `slides` (stringified JSON-like array): `Slides__e_g_____key____value____`
    - `textColor`: `Text_Color`
    - `slideDuration`: `Slide_Duration__seconds_`
    - `customTextPositionPercent` (**required**): `Custom_Text_Position______`
    - (Template supports other options like aspect ratio, transition, model, etc., but not mapped here.)
- **Connections:**
  - Available to AI Content Orchestrator via **ai_tool**.
- **Edge cases / failures:**
  - `customTextPositionPercent` is marked required by the template schema—if the agent omits it, the tool call may fail.
  - `slides` must be in the expected format (often JSON string); invalid formatting commonly breaks generation.

#### Node: Visual Asset Retriever
- **Type / role:** `@blotato/n8n-nodes-blotato.blotatoTool` — retrieves a generated “video” by ID (used to fetch final render URLs/status).
- **Configuration (interpreted):**
  - **Resource:** `video`
  - **Operation:** `get`
  - **Video ID:** `={{ $fromAI('Video_ID', ``, 'string') }}`
- **Connections:**
  - Available to AI Content Orchestrator via **ai_tool**.
- **Edge cases / failures:**
  - Video not ready yet (status still processing).
  - Wrong ID or permission errors.
  - Agent must decide when to retry/poll.

---

### 2.5 Distribution (Blotato Publishing)

**Overview:**  
These nodes publish the final text + media URLs to Instagram and LinkedIn through Blotato accounts, exposed as tools to the agent.

**Nodes involved:**
- Instagram Publisher
- LinkedIn Publisher

#### Node: Instagram Publisher
- **Type / role:** `@blotato/n8n-nodes-blotato.blotatoTool` — publishes content to an Instagram account via Blotato.
- **Configuration (interpreted):**
  - **Account:** `25299` (cached name: `giangxai.aff`)
  - **Post text:** `={{ $fromAI('Text', ``, 'string') }}`
  - **Media URLs:** `={{ $fromAI('Media_URLs', ``, 'string') }}`
- **Connections:**
  - Available to AI Content Orchestrator via **ai_tool**.
- **Edge cases / failures:**
  - Media URL invalid/unreachable or wrong format (some platforms require direct file URL, specific aspect ratio, etc.).
  - Instagram publishing constraints (caption length, hashtags, media type).
  - Account disconnected/expired inside Blotato.

#### Node: LinkedIn Publisher
- **Type / role:** `@blotato/n8n-nodes-blotato.blotatoTool` — publishes content to LinkedIn via Blotato.
- **Configuration (interpreted):**
  - **Platform:** `linkedin`
  - **Account:** `10190` (cached name: `Giang Vuong Thi`)
  - **Post text:** `$fromAI('Text', ...)`
  - **Media URLs:** `$fromAI('Media_URLs', ...)`
- **Connections:**
  - Available to AI Content Orchestrator via **ai_tool**.
- **Edge cases / failures:**
  - LinkedIn API/media requirements (image sizes, rate limits).
  - Account permissions (organization vs personal).
  - If the agent provides a single string for `Media_URLs` but Blotato expects an array/list format, posting can fail (depends on Blotato tool contract).

---

### 2.6 Response (Telegram)

**Overview:**  
Sends the agent’s final message back to the originating Telegram chat, escaping special characters to comply with MarkdownV2.

**Nodes involved:**
- Telegram Response Sender

#### Node: Telegram Response Sender
- **Type / role:** `n8n-nodes-base.telegram` — sends a Telegram message.
- **Configuration (interpreted):**
  - **Chat ID:** `={{ $('Telegram Bot Trigger').item.json.message.chat.id }}`
  - **Parse mode:** MarkdownV2
  - **Text:** Escapes MarkdownV2 special characters:
    - `={{ $json.output.replace(/([_*\[\]()~`>#+=|{}.!-])/g, '\\$1') }}`
    - Uses `$json.output` (the agent’s output field).
- **Connections:**
  - **Input (main)** from AI Content Orchestrator
  - No downstream nodes.
- **Edge cases / failures:**
  - If agent output is not in `$json.output` (different field name), message will be empty/fail.
  - MarkdownV2 escaping is broad; may still break if output includes unusual entities or already-escaped sequences.
  - Telegram rate limits and message length limits (long outputs may fail).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Bot Trigger | n8n-nodes-base.telegramTrigger | Entry point: receives Telegram messages | — | AI Content Orchestrator | ## TRIGGER\nReceives content requests from Telegram and starts the workflow. |
| AI Content Orchestrator | @n8n/n8n-nodes-langchain.agent | Central agent: intent analysis + tool routing + final response | Telegram Bot Trigger; (AI ports) Gemini Chat Model, Conversation Memory Store, multiple Blotato tools | Telegram Response Sender | ## AI CORE\nThe brain of the system. Understands user intent, decides whether to research or generate visuals, writes captions/scripts, and routes tasks to Blotato nodes. |
| Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for the agent (Gemini) | — | AI Content Orchestrator (ai_languageModel) | ## AI CORE\nThe brain of the system. Understands user intent, decides whether to research or generate visuals, writes captions/scripts, and routes tasks to Blotato nodes. |
| Conversation Memory Store | @n8n/n8n-nodes-langchain.memoryBufferWindow | Keeps chat context per Telegram chat ID | — | AI Content Orchestrator (ai_memory) | ## AI CORE\nThe brain of the system. Understands user intent, decides whether to research or generate visuals, writes captions/scripts, and routes tasks to Blotato nodes. |
| Source Collector | @blotato/n8n-nodes-blotato.blotatoTool | Tool: create Blotato source job from URL | AI Content Orchestrator (ai_tool call) | AI Content Orchestrator (tool result) | ## RESEARCH ENGINE (Blotato)\nCreates a research task in Blotato, extracts structured insights, and returns clean data ready for content generation. |
| Source Retriever | @blotato/n8n-nodes-blotato.blotatoTool | Tool: fetch Blotato source results by Source ID | AI Content Orchestrator (ai_tool call) | AI Content Orchestrator (tool result) | ## RESEARCH ENGINE (Blotato)\nCreates a research task in Blotato, extracts structured insights, and returns clean data ready for content generation. |
| Infographic Generator | @blotato/n8n-nodes-blotato.blotatoTool | Tool: generate infographic asset via template | AI Content Orchestrator (ai_tool call) | AI Content Orchestrator (tool result) | ## MEDIA ENGINE (Blotato)\nGenerates visual assets.Creates infographics or slideshow videos based on structured content. |
| Slideshow Video Generator | @blotato/n8n-nodes-blotato.blotatoTool | Tool: generate slideshow video via template | AI Content Orchestrator (ai_tool call) | AI Content Orchestrator (tool result) | ## MEDIA ENGINE (Blotato)\nGenerates visual assets.Creates infographics or slideshow videos based on structured content. |
| Visual Asset Retriever | @blotato/n8n-nodes-blotato.blotatoTool | Tool: retrieve generated video/asset by Video ID | AI Content Orchestrator (ai_tool call) | AI Content Orchestrator (tool result) | ## MEDIA ENGINE (Blotato)\nGenerates visual assets.Creatates infographics or slideshow videos based on structured content. |
| Instagram Publisher | @blotato/n8n-nodes-blotato.blotatoTool | Tool: publish to Instagram account via Blotato | AI Content Orchestrator (ai_tool call) | AI Content Orchestrator (tool result) | ## DISTRIBUTION (Blotato)\nPublishes generated content directly to Instagram and LinkedIn. |
| LinkedIn Publisher | @blotato/n8n-nodes-blotato.blotatoTool | Tool: publish to LinkedIn account via Blotato | AI Content Orchestrator (ai_tool call) | AI Content Orchestrator (tool result) | ## DISTRIBUTION (Blotato)\nPublishes generated content directly to Instagram and LinkedIn. |
| Telegram Response Sender | n8n-nodes-base.telegram | Sends final confirmation/result to Telegram | AI Content Orchestrator | — | ## RESPONSE\nSends confirmation back to Telegram. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block | — | — | (This node is itself the note)\n## AI CORE\nThe brain of the system. Understands user intent, decides whether to research or generate visuals, writes captions/scripts, and routes tasks to Blotato nodes. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment block | — | — | (This node is itself the note)\n## RESEARCH ENGINE (Blotato)\nCreates a research task in Blotato, extracts structured insights, and returns clean data ready for content generation. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment block | — | — | (This node is itself the note)\n## MEDIA ENGINE (Blotato)\nGenerates visual assets.Creates infographics or slideshow videos based on structured content. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment block | — | — | (This node is itself the note)\n## DISTRIBUTION (Blotato)\nPublishes generated content directly to Instagram and LinkedIn. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment block | — | — | (This node is itself the note)\n## RESPONSE\nSends confirmation back to Telegram. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment block | — | — | (This node is itself the note)\n## TRIGGER\nReceives content requests from Telegram and starts the workflow. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment block (setup/credits/links) | — | — | (This node is itself the note)\n# 🛠️ Workflow Setup Guide\nAuthor: [GiangxAI](https://www.youtube.com/@giangxai.official)\nSetup guide [n8n](https://n8n.partnerlinks.io/giangxai)\nBlotato: https://blotato.com/?ref=giang9s |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: **AI Agent Content** (or your preferred name).  
   - Ensure n8n has the LangChain agent nodes and the Blotato community/custom nodes installed (as applicable in your n8n environment).

2) **Add Telegram trigger**
   - Add node: **Telegram Trigger** (`telegramTrigger`)
   - Configure:
     - Updates: **message**
   - Create/select **Telegram API credential** with your bot token.
   - This node is the workflow entry.

3) **Add the AI agent (core orchestrator)**
   - Add node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Configure:
     - Text input: `{{ $json.message.text }}`
     - Prompt type: **Define**
     - System message: “You are a helpful assistant”
   - Connect: **Telegram Trigger → AI Agent** (main)

4) **Add Gemini model and connect it to the agent**
   - Add node: **Gemini Chat Model** (`lmChatGoogleGemini`)
   - Add/select **Google Gemini/PaLM credential** (API key).
   - Connect: **Gemini Chat Model → AI Agent** using the **ai_languageModel** connection type/port.

5) **Add conversation memory and connect it to the agent**
   - Add node: **Memory Buffer Window** (`memoryBufferWindow`)
   - Configure:
     - Session ID type: **Custom Key**
     - Session key: `{{ $json.message.chat.id }}`
     - Context window length: **30**
   - Connect: **Memory → AI Agent** using the **ai_memory** connection type/port.

6) **Add Blotato research tools**
   - Add node: **Blotato Tool** (rename to **Source Collector**)
     - Resource: **source**
     - Source URL: `{{ $fromAI('URL', '', 'string') }}`
     - Credentials: **Blotato API**
   - Add node: **Blotato Tool** (rename to **Source Retriever**)
     - Resource: **source**
     - Operation: **get**
     - Source ID: `{{ $fromAI('Source_ID', '', 'string') }}`
     - Credentials: **Blotato API**
   - Connect each of these nodes to the agent via **ai_tool** connections:
     - Source Collector → AI Agent (ai_tool)
     - Source Retriever → AI Agent (ai_tool)

7) **Add Blotato media generation tools**
   - Add node: **Blotato Tool** (rename to **Infographic Generator**)
     - Resource: **video**
     - Template: choose the infographic template (ID `29ebb2bd-02b7-4317-8bb8-c30eb938e47c`)
     - Prompt: `{{ $fromAI('Prompt', '', 'string') }}`
     - Template inputs:
       - Infographic Description: `{{ $fromAI('Infographic_Description', '', 'string') }}`
       - Footer CTA Text: `{{ $fromAI('Footer_CTA_Text', '', 'string') }}`
   - Add node: **Blotato Tool** (rename to **Slideshow Video Generator**)
     - Resource: **video**
     - Template: “Image Slideshow with Text Overlays” (path `/base/v2/image-slideshow/5903b592-1255-43b4-b9ac-f8ed7cbf6a5f/v1`)
     - Prompt: `{{ $fromAI('Prompt', '', 'string') }}`
     - Template inputs (as strings):
       - Slides: `{{ $fromAI('Slides__e_g_____key____value____', '', 'string') }}`
       - Text Color: `{{ $fromAI('Text_Color', '', 'string') }}`
       - Slide Duration: `{{ $fromAI('Slide_Duration__seconds_', '', 'string') }}`
       - Custom Text Position (%): `{{ $fromAI('Custom_Text_Position______', '', 'string') }}` (**required**)
   - Add node: **Blotato Tool** (rename to **Visual Asset Retriever**)
     - Resource: **video**
     - Operation: **get**
     - Video ID: `{{ $fromAI('Video_ID', '', 'string') }}`
   - Connect all three to the agent via **ai_tool** connections.

8) **Add Blotato publishing tools**
   - Add node: **Blotato Tool** (rename to **Instagram Publisher**)
     - Account: select your Instagram-connected Blotato account (in the JSON: `25299`)
     - Post text: `{{ $fromAI('Text', '', 'string') }}`
     - Media URLs: `{{ $fromAI('Media_URLs', '', 'string') }}`
   - Add node: **Blotato Tool** (rename to **LinkedIn Publisher**)
     - Platform: **linkedin**
     - Account: select your LinkedIn-connected Blotato account (in the JSON: `10190`)
     - Post text: `{{ $fromAI('Text', '', 'string') }}`
     - Media URLs: `{{ $fromAI('Media_URLs', '', 'string') }}`
   - Connect both to the agent via **ai_tool**.

9) **Add Telegram response sender**
   - Add node: **Telegram** (`n8n-nodes-base.telegram`) set to “Send Message”
   - Configure:
     - Chat ID: `{{ $('Telegram Bot Trigger').item.json.message.chat.id }}`
     - Parse mode: **MarkdownV2**
     - Text:
       - `{{ $json.output.replace(/([_*\[\]()~`>#+=|{}.!-])/g, '\\$1') }}`
   - Connect: **AI Agent → Telegram** (main)

10) **Credentials checklist**
   - Telegram: bot token configured and webhook reachable.
   - Google Gemini: valid API key + billing/quota OK.
   - Blotato: API key configured; accounts (Instagram/LinkedIn) connected inside Blotato and selectable in node.

11) **Operational constraint to validate (important)**
   - Because there is **no explicit Wait/poll loop node**, the **agent prompt/behavior must handle**:
     - creating a source job, then retrieving it later
     - generating media, then retrieving it later
   - If you see “not ready yet” results from Blotato, you may need to add explicit **Wait + Retry** logic (or instruct the agent to retry tool calls).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: GiangxAI | https://www.youtube.com/@giangxai.official |
| Setup guide link (n8n) | https://n8n.partnerlinks.io/giangxai |
| Blotato link | https://blotato.com/?ref=giang9s |
| Architecture described in notes: “Trigger → Analyze → Create → Wait → Retrieve → Publish → Confirm” | Refers to an async orchestration pattern; the current workflow relies on agent-driven tool calling (no explicit Wait node present). |