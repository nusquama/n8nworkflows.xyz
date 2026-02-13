Create TikTok videos with GPT-4o, Digital Twins, Veo 3 and Telegram

https://n8nworkflows.xyz/workflows/create-tiktok-videos-with-gpt-4o--digital-twins--veo-3-and-telegram-13097


# Create TikTok videos with GPT-4o, Digital Twins, Veo 3 and Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow generates TikTok-style short videos that “resonate” with a defined target audience by combining: (1) a Telegram message as the creative direction, (2) GPT‑4o agent research + scriptwriting, (3) Digital Twin feedback testing via OriginalVoices, and (4) Veo 3 video generation via fal.ai queue—then returns the result back to Telegram.

**Typical use cases:**
- Rapid TikTok concepting for a product feature, launch, or offer.
- Audience-calibrated hooks and scripts validated with “Digital Twins”.
- Automated short-form video generation from a winning script.

### Logical Blocks
**1.1 Telegram Input Reception**  
Receives a user message in Telegram (topic/direction) that triggers the entire workflow.

**1.2 Product/Brand Configuration**  
Defines static product metadata (name, description, target audience, tone) used in the agent prompt.

**1.3 AI Research, Scriptwriting, and Digital Twin Testing**  
A LangChain Agent (GPT‑4o) uses two tools (Apify and OriginalVoices MCP) to:
- Scrape TikTok trends
- Query Digital Twins for perception and script testing
- Produce a strict JSON output containing scripts, winners, and a detailed video prompt

**1.4 Output Parsing and Video Generation (Veo 3 via fal.ai Queue)**  
Parses the agent’s JSON output, submits a Veo 3 generation request, waits, then polls request status.

**1.5 Telegram Result Delivery**  
Sends “video ready” details (hook, caption, hashtags, and video URL if present) back to Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Input Reception

**Overview:**  
This block starts the workflow when a Telegram user sends a message to the bot. The incoming message text becomes the “video topic/direction” for the agent.

**Nodes Involved:**
- Telegram Trigger

#### Node: Telegram Trigger
- **Type / role:** `n8n-nodes-base.telegramTrigger` — webhook-like trigger for Telegram updates.
- **Configuration choices:**
  - Listens to update type: **message**
- **Key expressions / variables used elsewhere:**
  - `$('Telegram Trigger').item.json.message.text` (topic/direction)
  - `$('Telegram Trigger').item.json.message.chat.id` (reply chat id)
- **Input / output connections:**
  - **Output:** to **Workflow Configuration**
- **Version notes:** `typeVersion 1.1` (standard Telegram Trigger behavior).
- **Potential failures / edge cases:**
  - Telegram credentials not configured / bot not started by the user.
  - Messages without `.text` (stickers, photos, voice notes) will break the agent prompt expectations unless handled.
  - If bot privacy settings prevent receiving messages in groups, triggers may not fire as expected.

**Sticky note coverage:**  
“### 1. Trigger”

---

### 2.2 Product/Brand Configuration

**Overview:**  
Stores product metadata that is injected into the agent prompt (name/description/audience/tone). This allows reusable, consistent positioning across runs.

**Nodes Involved:**
- Workflow Configuration (Set)

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — creates fields used downstream.
- **Configuration choices (interpreted):**
  - Sets four string fields:
    - `productName` = “Your Product Name”
    - `productDescription` = brief description placeholder
    - `targetAudience` = “US, 18-35, interested in tech and productivity”
    - `tone` = “casual, energetic, Gen-Z friendly”
- **Key expressions / variables used elsewhere:**
  - Referenced in the agent prompt:
    - `$('Workflow Configuration').item.json.productName`, etc.
- **Input / output connections:**
  - **Input:** from **Telegram Trigger**
  - **Output:** to **Research, Write & Test**
- **Version notes:** `typeVersion 3.4` (Set node with “assignments” structure).
- **Potential failures / edge cases:**
  - Mis-typed field names will cause empty prompt variables and reduce output quality.
  - If multiple items ever flow in, Set node behavior can replicate values per item; current design assumes single item per run.

**Sticky note coverage:**  
“### 2. Configuration  
Edit with your product details”

---

### 2.3 AI Research, Scriptwriting, and Digital Twin Testing

**Overview:**  
A GPT‑4o agent orchestrates research and audience testing using two MCP tools: Apify for TikTok trend scraping and OriginalVoices for Digital Twin feedback. It must output **raw JSON only** with scripts, winners, and a detailed `videoPrompt`.

**Nodes Involved:**
- Research, Write & Test (Agent)
- OpenAI Chat Model
- Apify MCP (MCP Tool)
- OriginalVoices Digital Twins (MCP Tool)

#### Node: Research, Write & Test
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LangChain agent that can call tools and produce a final response.
- **Configuration choices (interpreted):**
  - Prompt is “define”-style with strict multi-step instructions:
    1. Use **Apify** to find 5–10 trending TikTok videos (formats/hooks/sounds/hashtags/styles).
    2. Use **ask_twins** (via OriginalVoices MCP tool) for product perception.
    3. Write 3 scripts (hook + full script + visual direction).
    4. Test hooks and authenticity with **two** ask_twins calls.
    5. Output **ONLY raw JSON** matching a specified schema including:
       - `trends[]`, `productInsights`, `scripts[]`, `scriptTestFeedback`, `bestScript`, `videoPrompt` (100–150 words), `caption`, `hashtags[]`.
  - The prompt injects dynamic fields from:
    - Telegram message text
    - Workflow Configuration fields
- **Key expressions / variables:**
  - `{{ $('Telegram Trigger').item.json.message.text }}`
  - `{{ $('Workflow Configuration').item.json.productName }}` etc.
- **Input / output connections:**
  - **Input:** from **Workflow Configuration**
  - **AI language model input:** via **OpenAI Chat Model** connection (`ai_languageModel`)
  - **AI tools:** via **Apify MCP** and **OriginalVoices Digital Twins** (`ai_tool`)
  - **Main output:** to **Parse JSON**
- **Version notes:** `typeVersion 3.1` (LangChain agent node; behavior can vary by n8n/LangChain package versions).
- **Potential failures / edge cases:**
  - Model returns non-JSON, trailing commentary, or markdown code fences (partly mitigated by Parse JSON node).
  - Tool call failures (Apify/OriginalVoices auth, rate limits, timeouts) may degrade output or cause agent failure.
  - The prompt demands parallel research; actual tool invocation order is agent-determined and not guaranteed to be parallel.
  - If Telegram message is empty/non-text, topic injection becomes blank and output quality suffers.

**Sticky note coverage:**  
“### 3. Research, Write & Test  
Connect Apify + OriginalVoices credentials”

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM for the agent.
- **Configuration choices:**
  - Model: **gpt-4o**
- **Input / output connections:**
  - **Output (AI language model):** to **Research, Write & Test**
- **Version notes:** `typeVersion 1.3`.
- **Potential failures / edge cases:**
  - Missing/invalid OpenAI credentials.
  - Model availability changes; “gpt-4o” must exist in your OpenAI account/region.
  - Rate limiting or token limits (long tool outputs + long prompt can overflow context).

#### Node: Apify MCP
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — exposes Apify MCP tools to the agent for scraping/research.
- **Configuration choices:**
  - Endpoint: `https://mcp.apify.com/sse`
  - Transport: **SSE**
  - Auth: **Header Auth** (generic header-based)
  - Header expected (per sticky note): `Authorization: Bearer YOUR_APIFY_TOKEN`
- **Input / output connections:**
  - **Tool connection:** to **Research, Write & Test**
- **Version notes:** `typeVersion 1.2`.
- **Potential failures / edge cases:**
  - Wrong token format (must include `Bearer` prefix).
  - SSE connectivity issues (firewalls/proxies) can prevent tool use.
  - Apify actor/tool limits, quotas, or anti-scraping restrictions affecting TikTok data.

#### Node: OriginalVoices Digital Twins
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — provides the `ask_twins` capability (Digital Twin feedback).
- **Configuration choices:**
  - Endpoint: `https://api.originalvoices.ai/mcp`
  - Auth: **Header Auth**
  - Header expected (per sticky note): `X-Api-Key: YOUR_API_KEY`
- **Input / output connections:**
  - **Tool connection:** to **Research, Write & Test**
- **Version notes:** `typeVersion 1.2`.
- **Potential failures / edge cases:**
  - Invalid API key / quota limitations.
  - The prompt emphasizes a strict schema (`audience` string, `questions` array). If agent sends mismatched parameters, tool calls can fail.
  - Latency/timeouts if twin calls are slow.

---

### 2.4 Output Parsing and Video Generation (Veo 3 via fal.ai Queue)

**Overview:**  
This block converts the agent’s final text output into structured JSON, submits a Veo 3.1 generation job to fal.ai’s queue endpoint, waits 60 seconds, then polls the job status for the video URL.

**Nodes Involved:**
- Parse JSON (Code)
- Generate Video (Veo 3) (HTTP Request)
- Wait for Video (Wait)
- Poll Video Status (HTTP Request)

#### Node: Parse JSON
- **Type / role:** `n8n-nodes-base.code` — sanitizes and parses the agent response into JSON fields for downstream use.
- **Configuration choices (interpreted):**
  - Reads: `$input.first().json.output`
  - If it’s a string, strips markdown fences:
    - removes ```json and ``` markers
  - Parses JSON via `JSON.parse`
  - Returns: `{ json: parsedObject }`
- **Key expressions / variables:**
  - Expects the agent output stored under `output` (typical for some agent node outputs).
- **Input / output connections:**
  - **Input:** from **Research, Write & Test**
  - **Output:** to **Generate Video (Veo 3)**
- **Version notes:** `typeVersion 2`.
- **Potential failures / edge cases:**
  - If the agent output is not valid JSON (missing commas/quotes), `JSON.parse` throws and the workflow fails.
  - If the agent returns JSON already as an object but not under `.output`, this code won’t find it.
  - If the agent returns additional text outside JSON (not fenced), stripping won’t help.

**Sticky note coverage:**  
“### 4. Generate Video”

#### Node: Generate Video (Veo 3)
- **Type / role:** `n8n-nodes-base.httpRequest` — submits a video generation request.
- **Configuration choices (interpreted):**
  - Method: **POST**
  - URL: `https://queue.fal.run/fal-ai/veo3.1`
  - Body: JSON
    - `prompt`: `{{ $json.videoPrompt }}`
    - `aspect_ratio`: `9:16`
    - `duration`: `8s`
  - Response format: JSON
  - Authentication: **Generic credential → HTTP Header Auth**
    - Header expected (per sticky note): `Authorization: Key YOUR_FAL_API_KEY`
- **Input / output connections:**
  - **Input:** from **Parse JSON**
  - **Output:** to **Wait for Video**
- **Version notes:** `typeVersion 4.2` (HTTP Request node).
- **Potential failures / edge cases:**
  - 401/403 if fal.ai key is missing/invalid or header format is wrong (`Key ...`).
  - Request may succeed but return a queued `request_id` only; workflow assumes downstream poll uses `$json.request_id`.
  - Prompt length/format issues: Veo may reject malformed prompts or enforce policy constraints.

#### Node: Wait for Video
- **Type / role:** `n8n-nodes-base.wait` — delays execution before polling.
- **Configuration choices:**
  - Wait duration: **60 seconds**
- **Input / output connections:**
  - **Input:** from **Generate Video (Veo 3)**
  - **Output:** to **Poll Video Status**
- **Version notes:** `typeVersion 1.1`.
- **Potential failures / edge cases:**
  - Video generation might take longer than 60 seconds; a single wait + single poll may return “in progress” with no final URL.

#### Node: Poll Video Status
- **Type / role:** `n8n-nodes-base.httpRequest` — checks the queued request status.
- **Configuration choices (interpreted):**
  - Method: default GET (not explicitly set; HTTP Request defaults apply)
  - URL (expression):  
    `https://queue.fal.run/fal-ai/veo3.1/requests/{{ $json.request_id }}`
  - Response format: JSON
  - Authentication: same **HTTP Header Auth** as Veo request (fal key)
- **Input / output connections:**
  - **Input:** from **Wait for Video**
  - **Output:** to **Confirm via Telegram**
- **Version notes:** `typeVersion 4.2`.
- **Potential failures / edge cases:**
  - If `request_id` is absent or nested differently, the URL expression becomes invalid.
  - The poll response schema may differ (e.g., `status`, `output`, `video.url`). The Telegram node tries several paths to find a URL.
  - If still processing, poll may not include a final URL; current workflow does not loop/retry.

---

### 2.5 Telegram Result Delivery

**Overview:**  
Sends back the winning hook plus a ready-to-post caption and hashtags, and includes the video URL when available.

**Nodes Involved:**
- Confirm via Telegram

#### Node: Confirm via Telegram
- **Type / role:** `n8n-nodes-base.telegram` — sends a Telegram message.
- **Configuration choices (interpreted):**
  - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
  - Text includes:
    - `bestScript.hook` (from Parse JSON node’s output)
    - `caption` and `hashtags` formatted as `#tag`
    - Video URL fallback resolution:
      - `{{ $json.video?.url || $json.output?.video?.url || 'Video generation in progress' }}`
- **Input / output connections:**
  - **Input:** from **Poll Video Status**
  - No downstream nodes.
- **Version notes:** `typeVersion 1.2`.
- **Potential failures / edge cases:**
  - If Parse JSON fails earlier, expressions referencing it will error.
  - If poll response doesn’t contain `video.url` or `output.video.url`, user will get “Video generation in progress”.
  - Telegram message length limits could be reached if caption/hashtags are too long (the prompt requests <150 chars caption; hashtags can still extend total length).

**Sticky note coverage:**  
“### 5. Report back to Telegram”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / canvas annotation |  |  | ## TikTok Videos That Resonate With Your Target Audience… Setup steps incl. credentials (Telegram/OpenAI/Apify/OriginalVoices/fal), customization notes, contact: Discord `vedad27`, email `vedad@originalvoices.ai` |
| Sticky Note Config | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ### 1. Trigger |
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Entry point: receives Telegram message | — | Workflow Configuration | ### 1. Trigger |
| Sticky Note Trigger | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ### 2. Configuration \nEdit with your product details |
| Workflow Configuration | n8n-nodes-base.set | Defines product/audience/tone variables | Telegram Trigger | Research, Write & Test | ### 2. Configuration \nEdit with your product details |
| Sticky Note Agent | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ### 3. Research, Write & Test \nConnect Apify + OriginalVoices credentials |
| Research, Write & Test | @n8n/n8n-nodes-langchain.agent | Agent: trend research + scriptwriting + twin testing | Workflow Configuration | Parse JSON | ### 3. Research, Write & Test \nConnect Apify + OriginalVoices credentials |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for agent (gpt-4o) | — (AI model link) | Research, Write & Test | ### 3. Research, Write & Test \nConnect Apify + OriginalVoices credentials |
| Apify MCP | @n8n/n8n-nodes-langchain.mcpClientTool | MCP tool for TikTok trend scraping | — (tool link) | Research, Write & Test | ### 3. Research, Write & Test \nConnect Apify + OriginalVoices credentials |
| OriginalVoices Digital Twins | @n8n/n8n-nodes-langchain.mcpClientTool | MCP tool for Digital Twin feedback (`ask_twins`) | — (tool link) | Research, Write & Test | ### 3. Research, Write & Test \nConnect Apify + OriginalVoices credentials |
| Parse JSON | n8n-nodes-base.code | Parses/cleans agent output into JSON | Research, Write & Test | Generate Video (Veo 3) | ### 4. Generate Video |
| Sticky Note Video | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ### 4. Generate Video |
| Generate Video (Veo 3) | n8n-nodes-base.httpRequest | Submit Veo 3 job to fal queue | Parse JSON | Wait for Video | ### 4. Generate Video |
| Wait for Video | n8n-nodes-base.wait | Delay before polling generation status | Generate Video (Veo 3) | Poll Video Status | ### 4. Generate Video |
| Poll Video Status | n8n-nodes-base.httpRequest | Poll fal queue for result URL | Wait for Video | Confirm via Telegram | ### 4. Generate Video |
| Sticky Note Report | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ### 5. Report back to Telegram |
| Confirm via Telegram | n8n-nodes-base.telegram | Sends hook/caption/video URL back to user | Poll Video Status | — | ### 5. Report back to Telegram |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **TikTok Videos That Resonate With Your Target Audience**
   - (Optional) Add tags: Content Creation, AI, TikTok, Marketing

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Updates: **message**
   - Credentials: connect your Telegram bot token (create via **@BotFather**)
   - This is the workflow entry node.

3. **Add Workflow Configuration (Set)**
   - Node: **Set** (rename to “Workflow Configuration”)
   - Add fields (string):
     - `productName`
     - `productDescription`
     - `targetAudience`
     - `tone`
   - Set your real values (not placeholders).
   - Connect: **Telegram Trigger → Workflow Configuration**

4. **Add the LangChain Agent**
   - Node: **AI Agent** (LangChain) (rename to “Research, Write & Test”)
   - Prompt mode: **Define**
   - Paste the full instruction prompt (ensure it demands **ONLY raw JSON** output and includes:
     - Telegram message topic: `{{ $('Telegram Trigger').item.json.message.text }}`
     - Product fields from Set node
     - Steps for Apify trends + ask_twins perception + 3 scripts + 2 ask_twins tests
     - Output schema including `videoPrompt`, `caption`, `hashtags`)
   - Connect: **Workflow Configuration → Research, Write & Test**

5. **Add and connect the OpenAI Chat Model**
   - Node: **OpenAI Chat Model**
   - Model: **gpt-4o**
   - Credentials: OpenAI API key in n8n credentials
   - Connect as the agent’s **Language Model** (AI connection):  
     **OpenAI Chat Model → Research, Write & Test (ai_languageModel)**

6. **Add Apify MCP tool**
   - Node: **MCP Client Tool** (rename “Apify MCP”)
   - Endpoint URL: `https://mcp.apify.com/sse`
   - Transport: **SSE**
   - Authentication: **Header Auth**
     - Header: `Authorization`
     - Value: `Bearer YOUR_APIFY_TOKEN`
   - Connect as agent **Tool**:  
     **Apify MCP → Research, Write & Test (ai_tool)**

7. **Add OriginalVoices MCP tool**
   - Node: **MCP Client Tool** (rename “OriginalVoices Digital Twins”)
   - Endpoint URL: `https://api.originalvoices.ai/mcp`
   - Authentication: **Header Auth**
     - Header: `X-Api-Key`
     - Value: `YOUR_API_KEY`
   - Connect as agent **Tool**:  
     **OriginalVoices Digital Twins → Research, Write & Test (ai_tool)**

8. **Add Parse JSON (Code)**
   - Node: **Code** (rename “Parse JSON”)
   - JavaScript logic:
     - Read the agent output field (commonly `$input.first().json.output`)
     - Strip code fences if present
     - `JSON.parse()` into an object and return it as `item.json`
   - Connect: **Research, Write & Test → Parse JSON**

9. **Add Generate Video (Veo 3) HTTP Request**
   - Node: **HTTP Request** (rename “Generate Video (Veo 3)”)
   - Method: **POST**
   - URL: `https://queue.fal.run/fal-ai/veo3.1`
   - Send body: **JSON**
   - JSON body:
     - `prompt`: from parsed output `{{$json.videoPrompt}}`
     - `aspect_ratio`: `"9:16"`
     - `duration`: `"8s"`
   - Authentication: **HTTP Header Auth** credential
     - Header: `Authorization`
     - Value: `Key YOUR_FAL_API_KEY`
   - Response: JSON
   - Connect: **Parse JSON → Generate Video (Veo 3)**

10. **Add Wait node**
   - Node: **Wait** (rename “Wait for Video”)
   - Wait for: **60 seconds**
   - Connect: **Generate Video (Veo 3) → Wait for Video**

11. **Add Poll Video Status HTTP Request**
   - Node: **HTTP Request** (rename “Poll Video Status”)
   - URL (expression):  
     `https://queue.fal.run/fal-ai/veo3.1/requests/{{ $json.request_id }}`
   - Authentication: same **fal** header auth (`Authorization: Key ...`)
   - Response: JSON
   - Connect: **Wait for Video → Poll Video Status**

12. **Add Telegram “Send Message” node**
   - Node: **Telegram** (send message) (rename “Confirm via Telegram”)
   - Chat ID expression:  
     `={{ $('Telegram Trigger').item.json.message.chat.id }}`
   - Message text expression includes:
     - Hook: `{{ $('Parse JSON').item.json.bestScript.hook }}`
     - Caption + hashtags: `{{ $('Parse JSON').item.json.caption }} {{ $('Parse JSON').item.json.hashtags.map(h => '#' + h).join(' ') }}`
     - Video URL fallback: `{{ $json.video?.url || $json.output?.video?.url || 'Video generation in progress' }}`
   - Connect: **Poll Video Status → Confirm via Telegram**

13. **Activate**
   - Ensure all credentials are set (Telegram, OpenAI, Apify header auth, OriginalVoices header auth, fal header auth).
   - Activate the workflow and send a text message to your bot with a TikTok concept direction.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “TikTok Videos That Resonate With Your Target Audience… Setup: Telegram Bot (@BotFather), OpenAI API, Apify header auth (`Authorization: Bearer …`), OriginalVoices header auth (`X-Api-Key: …`), fal header auth (`Authorization: Key …`). Customize audience/trends/scripts/video style. Need help: Discord `vedad27` or email `vedad@originalvoices.ai`.” | Included in the main sticky note on the canvas (workflow overview + setup guidance). |