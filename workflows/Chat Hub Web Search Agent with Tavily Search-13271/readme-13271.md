Chat Hub Web Search Agent with Tavily Search

https://n8nworkflows.xyz/workflows/chat-hub-web-search-agent-with-tavily-search-13271


# Chat Hub Web Search Agent with Tavily Search

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow creates a **Chat Hub “Search Agent”** that answers user questions in n8n Chat Hub. For any factual/verifiable question, it is instructed to **use Tavily Search first**, then respond with **mandatory inline citations**.

**Target use cases:**
- Quick fact lookup with sources (prices, specs, dates, policies, availability)
- Lightweight research and synthesis with verifiable citations
- A base agent you can extend with more tools and models

### Logical Blocks
**1.1 Chat Input Reception (Chat Hub Trigger)**  
Receives chat messages from n8n Chat Hub and streams responses back.

**1.2 Agent Orchestration (AI Agent + model + memory)**  
Routes the user message into a LangChain-style agent, using an LLM (OpenRouter Gemini) and a short window memory.

**1.3 Web Search Tooling (Tavily Tool)**  
Provides the agent with a single “Search” tool for web queries (advanced depth).

---

## 2. Block-by-Block Analysis

### 2.1 Chat Input Reception (Chat Hub Trigger)

**Overview:**  
Starts the workflow when a user sends a message in n8n Chat Hub. Configured for **streaming** responses, enabling the agent to reply progressively.

**Nodes involved:**
- When chat message received in Chat Hub

#### Node: **When chat message received in Chat Hub**
- **Type / Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Chat Hub entry point (trigger).
- **Key configuration:**
  - **Response mode:** `streaming` (the agent can stream partial outputs).
  - **Agent visibility:** `availableInChat: true`
  - **Agent name:** “Search Agent”
  - **Agent description:** “An agent that can search quickly… as well as do deep research”
  - **Agent icon:** emoji 🌎 (display-only)
- **Key data produced (typical):**
  - `chatInput` (the user’s message), used downstream by the agent.
- **Connections:**
  - **Main output →** AI Agent (main input)
- **Failure/edge cases:**
  - If Chat Hub is not enabled/accessible, the trigger won’t receive messages.
  - Streaming requires compatible Chat Hub frontend behavior; if disabled, responses may appear only when complete.
- **Version notes:** Node typeVersion `1.4` — ensure your n8n instance supports Chat Hub trigger nodes at this version range.

---

### 2.2 Agent Orchestration (AI Agent + LLM + Memory)

**Overview:**  
Takes the incoming chat message, applies a strict system prompt (search-first + citations), uses an OpenRouter chat model to reason, and uses memory to keep short conversational context.

**Nodes involved:**
- AI Agent
- Chat Model
- Simple Memory

#### Node: **AI Agent**
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — central agent that plans, calls tools, and generates the final response.
- **Key configuration choices:**
  - **Prompt type:** `define` (explicitly defined system message and input text)
  - **User text input expression:**
    - `={{ $('When chat message received in Chat Hub').item.json.chatInput }}`
    - This pulls the incoming Chat Hub message (`chatInput`) from the trigger node.
  - **System message (major behaviors):**
    - “Search-before-answering” for factual/verifiable questions
    - Mandatory inline citations format: `... [_[domain.com](https://full-url)_]`
    - Avoids unsourced claims; asks clarifying questions only when blocked
    - Formatting guidance for Chat Hub (Markdown, links, images)
  - **Max iterations:** `50` (caps tool/LLM loops; helps prevent runaway loops but still fairly high)
- **Connections:**
  - **Main input ←** When chat message received in Chat Hub
  - **ai_languageModel ←** Chat Model
  - **ai_memory ←** Simple Memory
  - **ai_tool ←** Search (Tavily)
- **Failure/edge cases:**
  - If the upstream trigger payload changes or `chatInput` is absent, the expression can fail or produce empty input.
  - Long iterative loops: with `maxIterations=50`, the agent could still be slow or costly if it repeatedly searches.
  - The system prompt enforces citations; if the model fails to comply, outputs may not meet requirements (you may need additional output validation if strict compliance is critical).
- **Version notes:** typeVersion `3.1` — ensure your n8n LangChain agent node matches this major version because parameters and tool wiring can differ.

#### Node: **Chat Model**
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — LLM provider via OpenRouter.
- **Key configuration:**
  - **Model:** `google/gemini-3-flash-preview`
  - **Temperature:** `0.1` (biases toward deterministic/consistent outputs; good for citation discipline)
- **Credentials:**
  - **OpenRouter API credential** required (configured as “n8n-general-use-mcgarrigle” in the workflow).
- **Connections:**
  - **ai_languageModel →** AI Agent
- **Failure/edge cases:**
  - Invalid/expired OpenRouter key, quota limits, model deprecation/renaming, or OpenRouter outages.
  - “Preview” models can change behavior; may affect citation compliance or tool usage.
- **Version notes:** typeVersion `1` — confirm OpenRouter node availability in your n8n build.

#### Node: **Simple Memory**
- **Type / Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term conversation memory.
- **Key configuration:**
  - Uses default settings (no explicit window size shown in parameters; defaults depend on node implementation/version).
- **Connections:**
  - **ai_memory →** AI Agent
- **Failure/edge cases:**
  - Memory window may be too small/large depending on defaults; can lead to forgetting context or higher token usage.
  - If multiple chat sessions share the same workflow improperly, context leakage could occur (depends on how n8n scopes Chat Hub sessions; typically scoped per session).
- **Version notes:** typeVersion `1.3` — default behavior may differ from earlier versions.

---

### 2.3 Web Search Tooling (Tavily)

**Overview:**  
Provides the agent a single web search tool (Tavily) with **advanced** search depth, used to gather sources for factual claims.

**Nodes involved:**
- Search

#### Node: **Search**
- **Type / Role:** `@tavily/n8n-nodes-tavily.tavilyTool` — tool node callable by the agent.
- **Key configuration:**
  - **Query expression:**
    - `={{ $fromAI('Query', ``, 'string') }}`
    - This indicates the query text is supplied by the agent at runtime (LangChain tool call). If the agent does not provide “Query”, it falls back to an empty string.
  - **Search depth:** `advanced` (more thorough; potentially slower/costlier than basic)
- **Credentials:**
  - Tavily API credential required (configured as “Tavily account” in the workflow).
- **Connections:**
  - **ai_tool →** AI Agent
- **Failure/edge cases:**
  - Empty query (if the agent calls the tool incorrectly) may produce poor/no results.
  - Tavily API rate limits, invalid key, network timeouts.
  - Search results quality varies; the agent must still choose primary sources.
- **Version notes:** typeVersion `1` — ensure the Tavily community/partner node is installed and matches.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received in Chat Hub | @n8n/n8n-nodes-langchain.chatTrigger | Entry point from Chat Hub; streams responses | — | AI Agent | ## Chat in Chat Hub; Go to `{your-n8n-url}/home/chat` to use this agent! |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates reasoning + tool calls + final response | When chat message received in Chat Hub | (Tool/model/memory channels) | ## Just a Regular Ol Agent; You can build up from here! Add any other tool you want or add to the system prompt to fit your needs! |
| Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM backend via OpenRouter (Gemini) | — (wired via ai_languageModel) | AI Agent | ## Replace Me; If you want, replace me with the provider and model you prefer!; Maybe pick one out on our [AI Benchmark](https://n8n.io/ai-benchmark) |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Short-term conversation context | — (wired via ai_memory) | AI Agent |  |
| Search | @tavily/n8n-nodes-tavily.tavilyTool | Web search tool for factual verification | — (called by AI Agent) | AI Agent | Quickly returns results with Tavily Search |
| Sticky Note | n8n-nodes-base.stickyNote | Annotation | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Annotation | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Annotation | — | — |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Annotation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add Trigger Node**
   - Add node: **Chat Trigger** (name it: *When chat message received in Chat Hub*).
   - Configure:
     - **Response Mode:** `streaming`
     - **Agent Name:** `Search Agent`
     - **Agent Description:** `An agent that can search quickly for fast answers as well as do deep research`
     - **Agent Icon:** emoji 🌎
     - Ensure **Available in Chat** is enabled.

3. **Add AI Agent Node**
   - Add node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`).
   - Configure:
     - **Prompt type:** `define`
     - **Text:** set expression to:
       - `{{ $('When chat message received in Chat Hub').item.json.chatInput }}`
     - **Options → Max Iterations:** `50`
     - **System message:** paste the system message from the workflow (search-first rules + citations format + formatting rules).

4. **Connect Trigger to Agent**
   - Connect: **When chat message received in Chat Hub (main output)** → **AI Agent (main input)**.

5. **Add Chat Model (OpenRouter)**
   - Add node: **OpenRouter Chat Model** (`lmChatOpenRouter`) named *Chat Model*.
   - Configure:
     - **Model:** `google/gemini-3-flash-preview`
     - **Temperature:** `0.1`
   - Credentials:
     - Create/select **OpenRouter API** credentials (API key from OpenRouter).
   - Connect via the AI wiring:
     - **Chat Model (ai_languageModel output)** → **AI Agent (ai_languageModel input)**.

6. **Add Memory**
   - Add node: **Simple Memory** (`memoryBufferWindow`).
   - Leave defaults (or adjust window size if your node exposes it).
   - Connect:
     - **Simple Memory (ai_memory output)** → **AI Agent (ai_memory input)**.

7. **Add Tavily Search Tool**
   - Install/enable Tavily node package if required (`@tavily/n8n-nodes-tavily`).
   - Add node: **Tavily Tool** named *Search*.
   - Configure:
     - **Query:** use expression `{{ $fromAI('Query', '', 'string') }}`
     - **Options → search_depth:** `advanced`
   - Credentials:
     - Create/select **Tavily API** credentials (Tavily API key).
   - Connect:
     - **Search (ai_tool output)** → **AI Agent (ai_tool input)**.

8. **(Optional) Add Sticky Notes**
   - Add sticky note near the trigger:
     - `## Chat in Chat Hub\nGo to {your-n8n-url}/home/chat to use this agent!`
   - Add sticky note near the agent:
     - `## Just a Regular Ol Agent\nYou can build up from here!...`
   - Add sticky note near Chat Model:
     - `## Replace Me ... [AI Benchmark](https://n8n.io/ai-benchmark)`
   - Add sticky note near Search:
     - `Quickly returns results with Tavily Search`

9. **Test in Chat Hub**
   - Open: `{your-n8n-url}/home/chat`
   - Select **Search Agent**
   - Ask a factual question and confirm:
     - The agent runs Tavily searches when needed
     - Responses include inline citations in the required format

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Go to `{your-n8n-url}/home/chat` to use this agent! | Chat Hub usage note |
| Maybe pick one out on our AI Benchmark | https://n8n.io/ai-benchmark |
| Agent is designed to “search-before-answering” with mandatory citations | Implemented in the AI Agent system message |
| Chat Hub does not auto-render images; include full URLs in markdown | Implemented in the AI Agent system message |