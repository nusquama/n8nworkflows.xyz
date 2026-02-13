Automate WhatsApp customer support with GPT‑4, RAG, text, voice, image and docs

https://n8nworkflows.xyz/workflows/automate-whatsapp-customer-support-with-gpt-4--rag--text--voice--image-and-docs-13173


# Automate WhatsApp customer support with GPT‑4, RAG, text, voice, image and docs

## 1. Workflow Overview

**Purpose:**  
This workflow implements an **AI-powered WhatsApp customer support bot** using **Rapiwa** as the WhatsApp gateway and **OpenAI (GPT‑4 class chat model + audio/image capabilities)** via n8n’s LangChain nodes. It supports **text**, **voice/audio**, **images**, and **documents (PDF/XLS/XLSX)**. It also adds **RAG-like retrieval** through a “Research” agent tool that can query company/product/service data (Google Sheets) and documentation/support resources (Google Docs + HTTP tools).

**Target use cases:**
- Automated first-line customer support over WhatsApp
- Multimodal support: interpret and respond to voice notes, images, and attached documents
- Retrieval of product/service/company info from internal sources (Sheets/Docs) and external docs endpoints

### 1.1 Entry & Message Type Routing
Receives WhatsApp events from Rapiwa and routes them by message type (status, reactions, text, audio, image, document).

### 1.2 Status/Reaction Logging
Stores delivery/read/reaction signals into Google Sheets.

### 1.3 Text Support (LLM + Memory + Research Tool)
Generates a text reply using an agent with conversation memory and tool-based retrieval.

### 1.4 Voice/Audio Support (download → transcribe → agent → summarize → reply)
Downloads the audio, transcribes/analyzes it, generates a response, then summarizes to keep replies concise, and sends back via WhatsApp.

### 1.5 Image Support (download → vision analysis → prompt formatting → agent → reply)
Fetches the image, runs vision analysis, formats results into a prompt, generates a response, and sends back.

### 1.6 Document Support (download → type detect → extract → prompt → agent → reply)
Downloads an attached document, detects file type, extracts text/structured content from PDF/XLS/XLSX, builds a document prompt, generates a response, and replies. Unsupported types trigger an error reply.

### 1.7 Shared AI Infrastructure (OpenAI model, Memory, Think tool, Research tool + connectors)
All agent nodes share the same OpenAI chat model connection and memory window; agents can call a “Research” tool that in turn can call multiple Google Sheets/Docs/HTTP tools.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Entry & Message Type Routing
**Overview:** Receives inbound WhatsApp webhook events from Rapiwa and routes them into the correct processing pipeline based on the message category (status, reaction, text, audio, image, document).  
**Nodes involved:** `Rapiwa Trigger`, `Route Types`

#### Node: Rapiwa Trigger
- **Type / role:** `n8n-nodes-rapiwa.rapiwaTrigger` — webhook trigger for WhatsApp events.
- **Configuration (interpreted):** Uses a Rapiwa webhook endpoint (`webhookId` present). Receives payloads for message events (text/media/status/reaction).
- **Inputs/Outputs:** Entry node → outputs to `Route Types`.
- **Potential failures/edge cases:**
  - Rapiwa credentials/webhook misconfiguration (no events received)
  - Unexpected payload shape (downstream expressions may break)

#### Node: Route Types
- **Type / role:** `Switch` — branches based on inbound message type.
- **Configuration (interpreted):** Has 6 outgoing branches connected to:
  1) `Status`  
  2) `Reaction`  
  3) `Generates AI responses, maintains context, retrieves info` (text)  
  4) `Downloads audio from message URL` (audio)  
  5) `Fetches image data via HTTP request` (image)  
  6) `Download Document` (document)
- **Inputs/Outputs:** Input from `Rapiwa Trigger`; multiple outputs as above.
- **Potential failures/edge cases:**
  - If a new/unknown type arrives, it may not match any case → message not handled (silent drop unless default route exists; not visible in JSON)

---

### Block 2.2 — Status/Reaction Logging
**Overview:** Captures WhatsApp message status updates and reactions and logs them to Google Sheets.  
**Nodes involved:** `Status`, `Reaction`, `Save Message status in sheet`

#### Node: Status
- **Type / role:** `NoOp` — placeholder branch endpoint for status messages.
- **Configuration:** No operation; used for flow clarity.
- **Connections:** `Route Types` → `Status` → `Save Message status in sheet`
- **Edge cases:** None (but if payload differs, sheet mapping in the next node can fail).

#### Node: Reaction
- **Type / role:** `NoOp` — placeholder branch endpoint for reactions.
- **Connections:** `Route Types` → `Reaction` → `Save Message status in sheet`

#### Node: Save Message status in sheet
- **Type / role:** `Google Sheets` — writes status/reaction data.
- **Configuration (interpreted):** Likely “Append” or “Update” row operation; exact sheet ID/range not provided in JSON.
- **Inputs/Outputs:** From `Status` and `Reaction`; no downstream nodes.
- **Potential failures/edge cases:**
  - Google OAuth/Service Account permission errors
  - Sheet/range misconfiguration
  - Payload fields missing → mapping errors

---

### Block 2.3 — Text Support (Agent + Memory + Research)
**Overview:** For text messages, an AI agent generates replies while maintaining context and retrieving information via tools (Sheets/Docs/HTTP).  
**Nodes involved:** `Generates AI responses, maintains context, retrieves info`, `Memory`, `Think`, `Research`, `OpenAI`, plus all tools connected to `Research` (covered in Block 2.7), and `Rapiwa (text replay)`.

#### Node: Generates AI responses, maintains context, retrieves info
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — main conversational agent for text.
- **Configuration (interpreted):**
  - Uses `OpenAI` as chat LLM (connected via `ai_languageModel`)
  - Uses `Memory` (connected via `ai_memory`) to persist short conversation history (buffer window)
  - Has access to tools: `Think` and `Research` (connected via `ai_tool`)
- **Inputs/Outputs:** Input from `Route Types` (text branch) → output to `Rapiwa (text replay)`.
- **Key variables/expressions:** Not visible (node params omitted), but typically pulls inbound message text, sender id, and conversation id from trigger payload.
- **Potential failures/edge cases:**
  - Missing sender identifier breaks memory threading
  - Token limits if memory window not bounded or prompt too large
  - Tool calling failures (HTTP errors, Sheets errors) leading to incomplete answers

#### Node: Memory
- **Type / role:** `memoryBufferWindow` — maintains recent conversation turns.
- **Configuration (interpreted):** Buffer window size configured in node (not shown); shared across multiple agents.
- **Connections:** Provides memory to 4 agent nodes:
  - `Generates AI responses...`
  - `Processes the image analysis...`
  - `Analyzes the document...`
  - `Processes audio...`
- **Edge cases:**
  - If conversation key is not stable per user/chat, memory can mix users
  - In stateless execution environments, memory persistence depends on n8n execution storage configuration

#### Node: Think
- **Type / role:** `toolThink` — internal reasoning tool for agents (helps structure thoughts/tool usage).
- **Connections:** Available as a tool to multiple agents (text/image/audio).
- **Edge cases:** Minimal; but if agents overuse it, cost/latency increases.

#### Node: Rapiwa (text replay)
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — sends reply back to WhatsApp.
- **Configuration (interpreted):** Sends the agent’s generated text to the originating WhatsApp user/chat.
- **Inputs/Outputs:** From text agent; terminal.
- **Potential failures/edge cases:**
  - Rapiwa auth or send-message failures
  - Message formatting constraints (length, unsupported characters)

---

### Block 2.4 — Voice/Audio Support
**Overview:** Downloads voice note audio, transcribes it with OpenAI, generates an answer via an agent, then summarizes and sends a concise WhatsApp reply.  
**Nodes involved:** `Downloads audio from message URL`, `Transcribes and analyzes`, `Processes audio and generates a response`, `Summarizes audio for concise replies`, `Rapiwa (audio/voice replay)`, plus `OpenAI`, `Memory`, `Think`, `Research`.

#### Node: Downloads audio from message URL
- **Type / role:** `HTTP Request` — fetches binary audio from the URL in the incoming message.
- **Configuration (interpreted):** Likely uses “Download” / “Response: File” to get binary data; URL taken from webhook payload.
- **Outputs:** Sends to:
  - `Transcribes and analyzes`
  - `Processes audio and generates a response` (parallel feed)
- **Edge cases:**
  - Expired media URLs
  - Large file size → timeouts / memory constraints
  - Missing auth headers if the media URL requires authorization

#### Node: Transcribes and analyzes
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI operation for audio transcription/analysis.
- **Configuration (interpreted):** Uses OpenAI audio endpoint (e.g., transcription) with the downloaded binary.
- **Connections:** Input from audio download → output to `Processes audio and generates a response`.
- **Edge cases:**
  - Unsupported audio codec/format
  - Audio too long → model limits
  - OpenAI rate limits/timeouts

#### Node: Processes audio and generates a response
- **Type / role:** `LangChain Agent` — agent that takes transcription + context and drafts a response.
- **Configuration:** Uses `OpenAI` model + `Memory` + tools (`Think`, `Research`).
- **Connections:** Inputs from `Downloads audio...` and `Transcribes...` (agent can combine both) → output to `Summarizes audio for concise replies`.
- **Edge cases:**
  - If transcription fails/empty, agent may hallucinate or respond generically
  - Tool failures (same as text)

#### Node: Summarizes audio for concise replies
- **Type / role:** `LangChain Agent` — “post-processing” agent to compress the answer.
- **Configuration:** Uses `OpenAI` model; has `Research` tool connection too (per workflow connections).
- **Connections:** From audio agent → to `Rapiwa (audio/voice replay)`.
- **Edge cases:** Over-summarization can remove critical steps; needs careful prompt.

#### Node: Rapiwa (audio/voice replay)
- **Type / role:** Sends final text back to WhatsApp.
- **Connections:** From summarizer; terminal.

---

### Block 2.5 — Image Support
**Overview:** Downloads an image, analyzes it with an OpenAI vision-capable node, formats the analysis into a prompt, generates a response with an agent, and replies on WhatsApp.  
**Nodes involved:** `Fetches image data via HTTP request`, `Analyzes the image content`, `Formats the image analysis into a structured prompt`, `Processes the image analysis and generates a response`, `Rapiwa (image replay)`, plus `OpenAI`, `Memory`, `Think`, `Research`.

#### Node: Fetches image data via HTTP request
- **Type / role:** `HTTP Request` — downloads image binary.
- **Edge cases:** Same as audio download (expired links, auth headers, large images).

#### Node: Analyzes the image content
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — vision analysis (image-to-text).
- **Configuration (interpreted):** Uses an OpenAI vision-capable model/operation; input is binary image.
- **Connections:** From image fetch → output to `Formats the image analysis into a structured prompt`.
- **Edge cases:** Unsupported image formats, oversized images, rate limits.

#### Node: Formats the image analysis into a structured prompt
- **Type / role:** `Set` — builds a clean prompt object/fields for the downstream agent.
- **Configuration:** Not shown; typically maps “vision result” into a `prompt` or `input` field for the agent.
- **Connections:** From image analysis → to image agent.
- **Edge cases:** If vision node output schema changes, mapping breaks.

#### Node: Processes the image analysis and generates a response
- **Type / role:** `LangChain Agent` — produces the final reply based on analysis + context + retrieval.
- **Configuration:** Uses `OpenAI`, `Memory`, tools (`Think`, `Research`).
- **Connections:** From formatted prompt → to `Rapiwa (image replay)`.

#### Node: Rapiwa (image replay)
- **Type / role:** Sends final reply.
- **Edge cases:** Message length constraints.

---

### Block 2.6 — Document Support (PDF/XLS/XLSX + Unsupported Types)
**Overview:** Downloads a document, detects/branches on file extension, extracts text/structured data depending on type, maps it into a prompt, generates a response, and replies. Unsupported types trigger an error message.  
**Nodes involved:** `Download Document`, `Map file extensions`, `Route Document Types`, `Extract from PDF`, `Extract from XLS`, `Extract from XLSX`, `Map JSON`, `Map document prompt`, `Analyzes the document and generates a response`, `Rapiwa (document replay)`, `Rapiwa (Sends an error for unsupported file types.)`

#### Node: Download Document
- **Type / role:** `HTTP Request` — downloads attached document binary.
- **Configuration:** Likely “Download file” mode; URL from inbound payload.
- **Connections:** To `Map file extensions`.
- **Edge cases:** Same as other media downloads.

#### Node: Map file extensions
- **Type / role:** `Code` — extracts filename/extension and normalizes it for routing.
- **Configuration:** Custom JS (not included) presumably sets a field like `extension` or `mime`.
- **Connections:** To `Route Document Types`.
- **Edge cases:**
  - Missing filename/extension in payload
  - Uppercase extensions (PDF vs pdf) if not normalized
  - MIME/type mismatches

#### Node: Route Document Types
- **Type / role:** `Switch` — routes based on extension/type.
- **Connections (as wired):**
  - Multiple outputs go directly to `Map document prompt` (likely for simple text-based docs)
  - `Extract from PDF` branch
  - `Map JSON` branch (used for spreadsheet extraction normalization)
  - `Extract from XLS` branch
  - `Extract from XLSX` branch
  - `Rapiwa (Sends an error for unsupported file types.)` default/unsupported branch
- **Edge cases:** If extension mapping fails, document may go to wrong branch or unsupported.

#### Node: Extract from PDF
- **Type / role:** `Extract From File` — extracts text from PDF binary.
- **Connections:** To `Map document prompt`.
- **Edge cases:** Scanned PDFs without OCR may yield empty text.

#### Node: Extract from XLS / Extract from XLSX
- **Type / role:** `Extract From File` — extracts spreadsheet content.
- **Connections:** Both go to `Map JSON`.
- **Edge cases:** Complex spreadsheets may extract poorly; large files may time out.

#### Node: Map JSON
- **Type / role:** `Set` — normalizes extracted spreadsheet output into JSON fields.
- **Connections:** To `Map document prompt`.

#### Node: Map document prompt
- **Type / role:** `Set` — constructs the final prompt payload for document analysis.
- **Connections:** To `Analyzes the document and generates a response`.
- **Edge cases:** Prompt can become huge for large documents → token limit failures.

#### Node: Analyzes the document and generates a response
- **Type / role:** `LangChain Agent` — generates final answer based on extracted content + context + retrieval.
- **Configuration:** Uses `OpenAI`, `Memory`, tools (`Research` is connected).
- **Connections:** To `Rapiwa (document replay)`.

#### Node: Rapiwa (document replay)
- **Type / role:** Sends response back to WhatsApp.

#### Node: Rapiwa (Sends an error for unsupported file types.)
- **Type / role:** Sends an error message when document type is unsupported.
- **Edge cases:** Ensure it uses correct chat/user id even when routing fails.

---

### Block 2.7 — RAG / Retrieval Tooling (“Research” and its tools)
**Overview:** Provides a tool-based retrieval layer that agents can call to fetch information from internal sources (Sheets/Docs) and external documentation endpoints via HTTP tools.  
**Nodes involved:** `Research`, `Support Desk`, `salebot`, `delix`, `socialvibe`, `faculty`, `yoori`, `meetair`, `oxoo`, `flixoo`, `Log Customer Issues`, `Read Service`, `Read Product`, `Read Company Information`, `Company documentation`, `Docs (retrieves company documentation)`

#### Node: Research
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool wrapper that the agents can call (function/tool calling).
- **Configuration (interpreted):** Uses `OpenAI` as language model for tool reasoning; can call multiple connected tools.
- **Connections:**
  - Provided as `ai_tool` to multiple agents: text/image/audio/document/summarizer.
  - Calls out to the tools listed below (each connected to `Research` as `ai_tool`).
- **Edge cases:**
  - Tool selection ambiguity: overlapping tools can cause inconsistent retrieval
  - Tool latency: multiple HTTP/Sheets calls may slow replies
  - If tools require auth and credentials are missing, the agent may fail mid-run

#### Node: Support Desk (notes include links)
- **Type / role:** `httpRequestTool` — callable HTTP tool for documentation/support endpoints.
- **Notes (sticky content on node):**  
  https://docs.salebot.app/  
  https://docs.delix.cloud/  
  https://socialvibe.spagreen.net/docs/  
  https://faculty.spagreen.net/docs/  
  https://docs.spagreen.net/docs/yoori/get-started/introduction  
  https://meetair.spagreen.net/docs/  
  https://oxoo.spagreen.net/documentation/android/  
  https://docs.flixoo.app/
- **Edge cases:** robots/anti-bot protections; HTML parsing; endpoint downtime.

#### Nodes: salebot / delix / socialvibe / faculty / yoori / meetair / oxoo / flixoo
- **Type / role:** `httpRequestTool` — specialized HTTP tools (likely each targets a specific product’s docs endpoint).
- **Connections:** Each is callable by `Research`.
- **Edge cases:** Same as above; also requires consistent output format for the agent.

#### Nodes: Read Product / Read Service / Read Company Information / Log Customer Issues
- **Type / role:** `googleSheetsTool` — callable tools to read/write structured business data.
- **Connections:** Callable by `Research`.
- **Edge cases:** Permission errors; rate limits; inconsistent sheet schema.

#### Nodes: Company documentation / Docs (retrieves company documentation)
- **Type / role:** `googleDocsTool` — callable tools to retrieve text from Google Docs.
- **Connections:** Callable by `Research`.
- **Edge cases:** Doc permissions; large docs; formatting noise.

---

### Block 2.8 — Shared Model Provider
**Overview:** Central OpenAI chat model used by all agents and the Research tool.  
**Nodes involved:** `OpenAI`

#### Node: OpenAI
- **Type / role:** `lmChatOpenAi` — shared chat language model configuration.
- **Connections:** Supplies `ai_languageModel` to:
  - `Research`
  - `Generates AI responses...`
  - `Processes audio...`
  - `Summarizes audio...`
  - `Processes the image analysis...`
  - `Analyzes the document...`
- **Edge cases:**
  - Model mismatch: vision/audio nodes use `openAi` nodes separately, but agents use chat model; keep models consistent
  - Rate limits across multiple modalities
  - If using GPT‑4-class model, cost and latency considerations

---

### Block 2.9 — Sticky Notes (present but empty)
**Overview:** Multiple sticky note nodes exist but their `content` is empty, so they do not add documentation context.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`, `Sticky Note5`, `Sticky Note6`, `Sticky Note7`, `Sticky Note9`, `Sticky Note10`, `Sticky Note11`, `Sticky Note12`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Rapiwa Trigger | rapiwaTrigger | Entry point (WhatsApp webhook) | — | Route Types |  |
| Route Types | Switch | Route events by message type | Rapiwa Trigger | Status; Reaction; Generates AI responses, maintains context, retrieves info; Downloads audio from message URL; Fetches image data via HTTP request; Download Document |  |
| Status | NoOp | Status branch placeholder | Route Types | Save Message status in sheet |  |
| Reaction | NoOp | Reaction branch placeholder | Route Types | Save Message status in sheet |  |
| Save Message status in sheet | Google Sheets | Log status/reaction events | Status; Reaction | — |  |
| Generates AI responses, maintains context, retrieves info | LangChain Agent | Text chatbot agent | Route Types | Rapiwa (text replay) |  |
| Rapiwa (text replay) | rapiwa | Send text reply | Generates AI responses, maintains context, retrieves info | — |  |
| Downloads audio from message URL | HTTP Request | Download audio binary | Route Types | Transcribes and analyzes; Processes audio and generates a response |  |
| Transcribes and analyzes | OpenAI (LangChain) | Audio transcription/analysis | Downloads audio from message URL | Processes audio and generates a response |  |
| Processes audio and generates a response | LangChain Agent | Produce answer from audio transcript | Downloads audio from message URL; Transcribes and analyzes | Summarizes audio for concise replies |  |
| Summarizes audio for concise replies | LangChain Agent | Compress final audio response | Processes audio and generates a response | Rapiwa (audio/voice replay) |  |
| Rapiwa (audio/voice replay) | rapiwa | Send audio-derived text reply | Summarizes audio for concise replies | — |  |
| Fetches image data via HTTP request | HTTP Request | Download image binary | Route Types | Analyzes the image content |  |
| Analyzes the image content | OpenAI (LangChain) | Vision analysis | Fetches image data via HTTP request | Formats the image analysis into a structured prompt |  |
| Formats the image analysis into a structured prompt | Set | Normalize vision output to prompt | Analyzes the image content | Processes the image analysis and generates a response |  |
| Processes the image analysis and generates a response | LangChain Agent | Answer using image analysis | Formats the image analysis into a structured prompt | Rapiwa (image replay) |  |
| Rapiwa (image replay) | rapiwa | Send image-derived reply | Processes the image analysis and generates a response | — |  |
| Download Document | HTTP Request | Download document binary | Route Types | Map file extensions |  |
| Map file extensions | Code | Determine/normalize extension/type | Download Document | Route Document Types |  |
| Route Document Types | Switch | Route by document type | Map file extensions | Map document prompt; Extract from PDF; Map JSON; Extract from XLS; Extract from XLSX; Rapiwa (Sends an error for unsupported file types.) |  |
| Extract from PDF | Extract From File | Extract text from PDF | Route Document Types | Map document prompt |  |
| Extract from XLS | Extract From File | Extract data from XLS | Route Document Types | Map JSON |  |
| Extract from XLSX | Extract From File | Extract data from XLSX | Route Document Types | Map JSON |  |
| Map JSON | Set | Normalize spreadsheet extraction | Extract from XLS; Extract from XLSX; Route Document Types | Map document prompt |  |
| Map document prompt | Set | Build document analysis prompt | Route Document Types; Extract from PDF; Map JSON | Analyzes the document and generates a response |  |
| Analyzes the document and generates a response | LangChain Agent | Respond based on document content | Map document prompt | Rapiwa (document replay) |  |
| Rapiwa (document replay) | rapiwa | Send document-derived reply | Analyzes the document and generates a response | — |  |
| Rapiwa (Sends an error for unsupported file types.) | rapiwa | Send unsupported-type error | Route Document Types | — |  |
| Memory | Memory Buffer Window | Shared conversation context | — | (memory to multiple agents) |  |
| Think | toolThink | Internal reasoning tool | — | (tool to agents) |  |
| OpenAI | lmChatOpenAi | Shared chat model | — | Research; multiple agents |  |
| Research | agentTool | Tool-based retrieval orchestrator | — | (tool to agents) |  |
| Support Desk | httpRequestTool | HTTP retrieval tool (docs hub) | — | Research | https://docs.salebot.app/ \nhttps://docs.delix.cloud/ \nhttps://socialvibe.spagreen.net/docs/\nhttps://faculty.spagreen.net/docs/\nhttps://docs.spagreen.net/docs/yoori/get-started/introduction\nhttps://meetair.spagreen.net/docs/\nhttps://oxoo.spagreen.net/documentation/android/\nhttps://docs.flixoo.app/ |
| salebot | httpRequestTool | Product docs HTTP tool | — | Research |  |
| delix | httpRequestTool | Product docs HTTP tool | — | Research |  |
| socialvibe | httpRequestTool | Product docs HTTP tool | — | Research |  |
| faculty | httpRequestTool | Product docs HTTP tool | — | Research |  |
| yoori | httpRequestTool | Product docs HTTP tool | — | Research |  |
| meetair | httpRequestTool | Product docs HTTP tool | — | Research |  |
| oxoo | httpRequestTool | Product docs HTTP tool | — | Research |  |
| flixoo | httpRequestTool | Product docs HTTP tool | — | Research |  |
| Read Product | Google Sheets Tool | Retrieve product info | — | Research |  |
| Read Service | Google Sheets Tool | Retrieve service info | — | Research |  |
| Read Company Information | Google Sheets Tool | Retrieve company info | — | Research |  |
| Log Customer Issues | Google Sheets Tool | Log issues to a sheet | — | Research |  |
| Company documentation | Google Docs Tool | Retrieve internal docs | — | Research |  |
| Docs (retrieves company documentation) | Google Docs Tool | Retrieve internal docs | — | Research |  |
| Sticky Note | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note1 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note2 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note3 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note4 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note5 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note6 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note7 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note9 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note10 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note11 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note12 | Sticky Note | Comment container (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node **Rapiwa Trigger** (Rapiwa → Trigger).
   2. Configure Rapiwa credentials and webhook in Rapiwa dashboard; paste/connect in node.
   3. Verify you receive a payload containing: message type, sender/chat id, and (if media) media URL.

2) **Add message router**
   1. Add **Switch** node named **Route Types**.
   2. Create cases for:
      - `status` → output 1
      - `reaction` → output 2
      - `text` → output 3
      - `audio/voice` → output 4
      - `image` → output 5
      - `document` → output 6
   3. Connect: `Rapiwa Trigger` → `Route Types`.

3) **Status/Reaction logging**
   1. Add two **NoOp** nodes: **Status**, **Reaction**.
   2. Connect `Route Types` outputs (status/reaction) to these NoOps.
   3. Add **Google Sheets** node **Save Message status in sheet**.
   4. Configure Google Sheets credentials (OAuth2 or Service Account).
   5. Choose operation (commonly **Append row**) and map fields from the trigger payload (message id, status, timestamp, reaction, sender).
   6. Connect `Status` → `Save Message status in sheet` and `Reaction` → `Save Message status in sheet`.

4) **Shared AI components**
   1. Add **OpenAI Chat Model** node (**lmChatOpenAi**) named **OpenAI**.
      - Set model (GPT‑4 class) and API key credential.
   2. Add **Memory Buffer Window** node named **Memory**.
      - Configure window size and ensure the conversation/session key uses the WhatsApp chat id or sender id.
   3. Add **Tool: Think** node named **Think**.
   4. Add **Agent Tool** node named **Research**.
      - Connect **OpenAI** → `Research` via **ai_languageModel**.

5) **Create retrieval tools for “Research”**
   1. Add **Google Sheets Tool** nodes:
      - **Read Product**, **Read Service**, **Read Company Information**, **Log Customer Issues**
      - Configure each with the appropriate spreadsheet and operations (read/query/append).
   2. Add **Google Docs Tool** nodes:
      - **Company documentation**, **Docs (retrieves company documentation)**
      - Configure doc IDs and “get content” operations.
   3. Add **HTTP Request Tool** nodes:
      - **Support Desk**, **salebot**, **delix**, **socialvibe**, **faculty**, **yoori**, **meetair**, **oxoo**, **flixoo**
      - Configure base URLs and any headers/auth needed.
      - Put the provided links into **Support Desk** notes/description if desired.
   4. Connect each of these tools to **Research** as **ai_tool**.

6) **Text agent + reply**
   1. Add **LangChain Agent** named **Generates AI responses, maintains context, retrieves info**.
   2. Connect:
      - **OpenAI** → agent (`ai_languageModel`)
      - **Memory** → agent (`ai_memory`)
      - **Think** → agent (`ai_tool`)
      - **Research** → agent (`ai_tool`)
   3. In agent prompt/system instructions, ensure it:
      - Answers in the desired brand tone
      - Uses `Research` when user asks product/company questions
      - Asks clarifying questions when uncertain
   4. Add **Rapiwa** node **Rapiwa (text replay)** to send a text message back.
      - Map recipient/chat id from trigger payload
      - Map message body from agent output
   5. Connect: `Route Types` (text) → text agent → `Rapiwa (text replay)`.

7) **Audio pipeline**
   1. Add **HTTP Request** node **Downloads audio from message URL**:
      - Set URL from incoming payload’s media URL field
      - Enable binary download (store as binary property, e.g., `data`)
   2. Add **OpenAI** node **Transcribes and analyzes** (LangChain OpenAI):
      - Configure transcription operation and use the binary audio as input
   3. Add **LangChain Agent** **Processes audio and generates a response**:
      - Connect OpenAI model, Memory, Think, Research
      - Feed transcript text into agent prompt
   4. Add **LangChain Agent** **Summarizes audio for concise replies**:
      - Uses OpenAI model (optionally no tools, but in this workflow it’s connected to Research)
   5. Add **Rapiwa** node **Rapiwa (audio/voice replay)** to send final text.
   6. Connect: `Route Types` (audio) → download → transcribe → audio agent → summarize → rapiwa reply.  
      (Optionally also connect download → audio agent if you want metadata/filename available, as in the provided workflow.)

8) **Image pipeline**
   1. Add **HTTP Request** node **Fetches image data via HTTP request** (binary download).
   2. Add **OpenAI** node **Analyzes the image content** (vision).
   3. Add **Set** node **Formats the image analysis into a structured prompt**:
      - Map vision description + any detected text into a single `prompt` field.
   4. Add **LangChain Agent** **Processes the image analysis and generates a response**:
      - Connect OpenAI model, Memory, Think, Research
   5. Add **Rapiwa** node **Rapiwa (image replay)** to send final text.
   6. Connect: `Route Types` (image) → fetch → analyze → set → agent → reply.

9) **Document pipeline**
   1. Add **HTTP Request** node **Download Document** (binary download).
   2. Add **Code** node **Map file extensions**:
      - Parse filename/URL to get extension (lowercased).
      - Output a field like `docExt` for routing.
   3. Add **Switch** node **Route Document Types**:
      - Cases: `pdf`, `xls`, `xlsx` (and optionally doc/txt/csv if you support them)
      - Default route → unsupported
   4. Add extraction nodes:
      - **Extract from PDF** (Extract From File)
      - **Extract from XLS** (Extract From File)
      - **Extract from XLSX** (Extract From File)
   5. Add **Set** node **Map JSON** to normalize spreadsheet extraction into a textual/JSON summary.
   6. Add **Set** node **Map document prompt** to build the agent input (document text + user question + metadata).
   7. Add **LangChain Agent** **Analyzes the document and generates a response**:
      - Connect OpenAI model + Memory + Research tool
   8. Add **Rapiwa** node **Rapiwa (document replay)**.
   9. Add **Rapiwa** node **Rapiwa (Sends an error for unsupported file types.)** for the default branch.
   10. Connect the chain: document download → map extension → switch → extract/map → prompt → agent → reply.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| https://docs.salebot.app/ | Support documentation endpoint (listed in “Support Desk”) |
| https://docs.delix.cloud/ | Support documentation endpoint (listed in “Support Desk”) |
| https://socialvibe.spagreen.net/docs/ | Support documentation endpoint (listed in “Support Desk”) |
| https://faculty.spagreen.net/docs/ | Support documentation endpoint (listed in “Support Desk”) |
| https://docs.spagreen.net/docs/yoori/get-started/introduction | Support documentation endpoint (listed in “Support Desk”) |
| https://meetair.spagreen.net/docs/ | Support documentation endpoint (listed in “Support Desk”) |
| https://oxoo.spagreen.net/documentation/android/ | Support documentation endpoint (listed in “Support Desk”) |
| https://docs.flixoo.app/ | Support documentation endpoint (listed in “Support Desk”) |

**Disclaimer (provided by user/developer):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.