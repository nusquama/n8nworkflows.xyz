Route secure AI customer support chats with OpenAI and SoterAI

https://n8nworkflows.xyz/workflows/route-secure-ai-customer-support-chats-with-openai-and-soterai-20480


# Route secure AI customer support chats with OpenAI and SoterAI

### 1. Workflow Overview

This workflow implements an autonomous enterprise AI customer support copilot protected end-to-end by SoterAI. Its primary purpose is to safely handle customer inquiries regarding order statuses, tracking, shipping, and refunds while defending against prompt injection attacks, persona overrides, and sensitive data leakage (DLP). 

The application logic is partitioned into five functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming messages from multiple channels (interactive chat, REST webhook, or manual tests) and normalizes the payload into a consistent structure containing message text, session IDs, and customer metadata.
- **1.2 Input Security & Threat Screening:** Evaluates messages via SoterAI Input Threat Shield to block prompt injections and enforce strict business topic scopes, providing a dual-branch routing mechanism (safe vs. flagged).
- **1.3 AI Processing & Tool Execution:** Leverages an OpenAI-backed LangChain agent equipped with multi-turn memory, an ERP order lookup simulator, and a calculation tool to generate contextual support answers.
- **1.4 Output DLP & Sanitization:** Scans the generated LLM response using a SoterAI Output Guardrail to redact sensitive data (PII, credentials, tokens) before delivering it to the end user.
- **1.5 SecOps Incident Telemetry:** Intercepts security threats, formats incident payloads, dispatches alerts to external webhooks, and returns polite refusal messages to blocked users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** This block establishes multiple entry points for customer inquiries and standardizes the incoming payload variables into a single schema for downstream nodes.
- **Nodes Involved:** 
  - `Chat Trigger (Interactive UI)`
  - `Production Webhook Trigger`
  - `Manual Test Trigger`
  - `Unified Intake & Normalizer`
- **Node Details:**
  - **Chat Trigger (Interactive UI)**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.chatTrigger` (Trigger). Acts as an interactive UI chat widget entry point.
    - *Configuration:* Default options enabled; binds a webhook ID for chat testing.
    - *Input/Output:* Output connects to `Unified Intake & Normalizer`.
    - *Edge Cases:* Unhandled UI connection drops; handled via fallback triggers.
  - **Production Webhook Trigger**
    - *Type and Role:* `n8n-nodes-base.webhook` (Trigger). Listens for external POST requests at `/webhook/soterai-secure-agent`.
    - *Configuration:* Method: `POST`, Response Mode: `responseNode`.
    - *Input/Output:* Output connects to `Unified Intake & Normalizer`.
    - *Edge Cases:* Malformed JSON payloads or missing headers.
  - **Manual Test Trigger**
    - *Type and Role:* `n8n-nodes-base.manualTrigger` (Trigger). Allows manual execution inside the n8n editor using test data.
    - *Configuration:* Default.
    - *Input/Output:* Output connects to `Unified Intake & Normalizer`.
  - **Unified Intake & Normalizer**
    - *Type and Role:* `n8n-nodes-base.code` (Data transformation). Extracts and normalizes message text, session IDs, and customer emails regardless of source channel.
    - *Configuration:* JavaScript code parsing properties (`customerMessage`, `inputText`, `chatInput`, `body.message`, etc.).
    - *Key Expressions:* Dynamic fallback extraction across incoming JSON properties.
    - *Input/Output:* Inputs from all three triggers; output connects to `🛡️ SoterAI Input Threat Shield`.
    - *Edge Cases:* Empty input bodies default to a fallback test string (`Hi, where is my order #INV-8842?...`).

#### 2.2 Input Security & Threat Screening
- **Overview:** Acts as a zero-trust firewall inspecting incoming user messages to neutralize prompt injections, jailbreaks, and off-topic requests before any LLM compute occurs.
- **Nodes Involved:**
  - `🛡️ SoterAI Input Threat Shield`
- **Node Details:**
  - **🛡️ SoterAI Input Threat Shield**
    - *Type and Role:* `n8n-nodes-soterai.soterGuard` (Security guardrail). Intercepts inbound prompts for threats.
    - *Configuration:* Operation: `inputGuard`, Resource: `guardrail`, Sensitivity: `BALANCED`, Detection Engine: `AUTO`, Action on Threat: `BLOCK`. Enforces topic allowances (`orders, tracking, billing, returns`) and customized refusal messages for prompt injections, off-topic prompts, and sensitive data attempts.
    - *Key Expressions:* `={{ $json.sessionId }}` and `={{ $json.customerMessage }}`.
    - *Input/Output:* Input from `Unified Intake & Normalizer`. Dual outputs: Output 0 (Safe) connects to `🧠 Autonomous Support Copilot`; Output 1 (Threat/Flagged) connects to `🚨 SecOps Threat Incident Formatter`.
    - *Credentials:* SoterAI Production Cloud (`soterApi`).
    - *Edge Cases:* Request timeout (3000ms); handled by local fallback configurations (`neverDowngradeToLocal: false`).

#### 2.3 AI Processing & Tool Execution
- **Overview:** Processes safe customer support queries using an autonomous LLM agent equipped with conversational memory and live mock ERP tools.
- **Nodes Involved:**
  - `🧠 Autonomous Support Copilot`
  - `OpenAI Chat Model`
  - `Conversation Memory`
  - `Order Lookup Tool`
  - `Refund & Policy Calculator`
- **Node Details:**
  - **🧠 Autonomous Support Copilot**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent). Core orchestrator for reasoning and tool calling.
    - *Configuration:* Prompt Type: `define`, Error Handling: `continueRegularOutput`, System message defining an enterprise e-commerce support persona.
    - *Key Expressions:* `={{ $json.outputText || $json.rawInput || $('Unified Intake & Normalizer').item.json.customerMessage }}`.
    - *Input/Output:* Receives prompt input from `🛡️ SoterAI Input Threat Shield` (Output 0). Connected to Language Model, Memory, and Tools via LangChain sub-connections. Output connects to `🔒 SoterAI Output Guardrail (DLP)`.
  - **OpenAI Chat Model**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model provider).
    - *Configuration:* Model: `gpt-4o-mini`, Temperature: `0.2`.
    - *Credentials:* OpenAI API (`openAiApi`).
  - **Conversation Memory**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Memory manager). Maintains chat history.
    - *Configuration:* Session Key: `={{ $('Unified Intake & Normalizer').item.json.sessionId }}`, Context Window Length: `10`.
  - **Order Lookup Tool**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.toolCode` (Custom tool). Simulates database lookups for orders (e.g., `INV-8842`, `ORD-1029`).
    - *Configuration:* JavaScript routine matching regular expressions against query strings to query a mock database dictionary.
  - **Refund & Policy Calculator**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.toolCalculator` (Utility tool). Evaluates mathematical expressions for discounts or refund windows.

#### 2.4 Output DLP & Sanitization
- **Overview:** Intercepts outgoing AI responses to perform data loss prevention (DLP), scanning for sensitive PII, phone numbers, API tokens, or secrets before delivery.
- **Nodes Involved:**
  - `🔒 SoterAI Output Guardrail (DLP)`
  - `Deliver Response to Customer`
  - `Deliver Chat / Web Response`
  - `🚨 SecOps Output DLP Leak Alert`
- **Node Details:**
  - **🔒 SoterAI Output Guardrail (DLP)**
    - *Type and Role:* `n8n-nodes-soterai.soterGuard` (Security guardrail). Scans outbound text for secrets and PII.
    - *Configuration:* Operation: `outputGuard`, Resource: `guardrail`, Sensitivity: `BALANCED`, Action on Threat: `REDACT`.
    - *Key Expressions:* `={{ $('Unified Intake & Normalizer').item.json.sessionId }}` and `={{ $json.output || $json.text || $json.aiReply || ... }}`.
    - *Input/Output:* Input from `🧠 Autonomous Support Copilot`. Output 0 (Clean/Redacted) connects to `Deliver Response to Customer` and `Deliver Chat / Web Response`. Output 1 (Leak Detected) connects to `🚨 SecOps Output DLP Leak Alert`.
    - *Credentials:* SoterAI Production Cloud (`soterApi`).
  - **Deliver Response to Customer**
    - *Type and Role:* `n8n-nodes-base.respondToWebhook` (Webhook response). Returns JSON payload to webhook callers.
    - *Configuration:* Respond with JSON, constructing an object containing success status, reply text, and security metadata.
  - **Deliver Chat / Web Response**
    - *Type and Role:* `n8n-nodes-base.set` (Data formatter). Prepares structured response properties for chat interfaces.
    - *Configuration:* Assignments mapping `customerReply`, `securityStatus`, and `riskScore`.
  - **🚨 SecOps Output DLP Leak Alert**
    - *Type and Role:* `n8n-nodes-base.set` (Logging/Telemetry). Formats alerts when sensitive data leakage attempts are intercepted in AI outputs.
    - *Configuration:* Assignments for `alert`, `redactedText`, and `findingsCount`.

#### 2.5 SecOps Incident Telemetry
- **Overview:** Handles malicious prompts blocked at the input stage by formatting structured incident payloads, alerting security endpoints, and returning safe refusal messages.
- **Nodes Involved:**
  - `🚨 SecOps Threat Incident Formatter`
  - `Dispatch SecOps Alert (Slack/Webhook)`
  - `Send Security Rejection to Customer`
- **Node Details:**
  - **🚨 SecOps Threat Incident Formatter**
    - *Type and Role:* `n8n-nodes-base.set` (Data transformer). Converts threat metadata into a standardized incident schema.
    - *Configuration:* Assignments mapping incident type (`PROMPT_INJECTION_OR_SECURITY_THREAT`), severity (`HIGH`), risk score, reason, and timestamps.
  - **Dispatch SecOps Alert (Slack/Webhook)**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (HTTP client). Dispatches incident notifications to external monitoring systems or webhooks.
    - *Configuration:* URL: `https://httpbin.org/post` (placeholder for Slack/Teams/SIEM), Method: `POST`, JSON payload body.
    - *Input/Output:* Input from `🚨 SecOps Threat Incident Formatter`.
    - *Edge Cases:* Network timeouts or endpoint rejection; mitigated via standard error flows.
  - **Send Security Rejection to Customer**
    - *Type and Role:* `n8n-nodes-base.respondToWebhook` (Webhook response). Returns a safe rejection message to blocked callers.
    - *Configuration:* Respond with JSON containing `success: false`, user notification message, risk scores, and execution ID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Chat Trigger (Interactive UI) | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat widget UI entry point | None | Unified Intake & Normalizer | 📥 Stage 1: Ingestion & Normalizer |
| Production Webhook Trigger | `n8n-nodes-base.webhook` | External webhook endpoint | None | Unified Intake & Normalizer | 📥 Stage 1: Ingestion & Normalizer |
| Manual Test Trigger | `n8n-nodes-base.manualTrigger` | Manual test runner | None | Unified Intake & Normalizer | 📥 Stage 1: Ingestion & Normalizer |
| Unified Intake & Normalizer | `n8n-nodes-base.code` | Normalizes payload variables | Chat Trigger, Production Webhook, Manual Test Trigger | 🛡️ SoterAI Input Threat Shield | 📥 Stage 1: Ingestion & Normalizer |
| 🛡️ SoterAI Input Threat Shield | `n8n-nodes-soterai.soterGuard` | Zero-trust input prompt firewall | Unified Intake & Normalizer | Autonomous Support Copilot, SecOps Threat Incident Formatter | 🛡️ Stage 2: SoterAI Input Threat Shield |
| 🧠 Autonomous Support Copilot | `@n8n/n8n-nodes-langchain.agent` | Autonomous LLM agent | SoterAI Input Threat Shield | 🔒 SoterAI Output Guardrail (DLP) | 🧠 Stage 3: Autonomous AI Copilot & Verified Tools |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM provider (`gpt-4o-mini`) | None | Autonomous Support Copilot | 🧠 Stage 3: Autonomous AI Copilot & Verified Tools |
| Conversation Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Multi-turn chat memory | None | Autonomous Support Copilot | 🧠 Stage 3: Autonomous AI Copilot & Verified Tools |
| Order Lookup Tool | `@n8n/n8n-nodes-langchain.toolCode` | Simulated ERP order lookup tool | None | Autonomous Support Copilot | 🧠 Stage 3: Autonomous AI Copilot & Verified Tools |
| Refund & Policy Calculator | `@n8n/n8n-nodes-langchain.toolCalculator` | Calculation utility tool | None | Autonomous Support Copilot | 🧠 Stage 3: Autonomous AI Copilot & Verified Tools |
| 🔒 SoterAI Output Guardrail (DLP) | `n8n-nodes-soterai.soterGuard` | Outbound DLP secret/PII scanner | Autonomous Support Copilot | Deliver Response to Customer, Deliver Chat / Web Response, SecOps Output DLP Leak Alert | 🔒 Stage 4: Real-Time Output DLP & Delivery |
| Deliver Response to Customer | `n8n-nodes-base.respondToWebhook` | Returns sanitized JSON response | SoterAI Output Guardrail (DLP) | None | 🔒 Stage 4: Real-Time Output DLP & Delivery |
| Deliver Chat / Web Response | `n8n-nodes-base.set` | Formats chat response properties | SoterAI Output Guardrail (DLP) | None | 🔒 Stage 4: Real-Time Output DLP & Delivery |
| 🚨 SecOps Threat Incident Formatter | `n8n-nodes-base.set` | Formats security incident data | SoterAI Input Threat Shield | Dispatch SecOps Alert, Send Security Rejection to Customer | 🚨 Stage 5: Automated SecOps Incident Telemetry & Threat Escalation (Triggered on Attack) |
| Dispatch SecOps Alert (Slack/Webhook) | `n8n-nodes-base.httpRequest` | Sends incident payload to SIEM/Slack | SecOps Threat Incident Formatter | None | 🚨 Stage 5: Automated SecOps Incident Telemetry & Threat Escalation (Triggered on Attack) |
| Send Security Rejection to Customer | `n8n-nodes-base.respondToWebhook` | Returns refusal notice to blocked user | SecOps Threat Incident Formatter | None | 🚨 Stage 5: Automated SecOps Incident Telemetry & Threat Escalation (Triggered on Attack) |
| 🚨 SecOps Output DLP Leak Alert | `n8n-nodes-base.set` | Logs AI output DLP leak attempts | SoterAI Output Guardrail (DLP) | None | 🔒 Stage 4: Real-Time Output DLP & Delivery |

---

### 4. Reproducing the Workflow from Scratch

1. **Install Community Nodes:** Navigate to n8n Settings > Community Nodes and install `n8n-nodes-soterai`.
2. **Create Ingestion Triggers:**
   - Add a **Chat Trigger (Interactive UI)** node (`@n8n/n8n-nodes-langchain.chatTrigger`).
   - Add a **Webhook** node (`n8n-nodes-base.webhook`), set HTTP Method to `POST`, and Path to `soterai-secure-agent`.
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`).
3. **Add the Normalizer:**
   - Create a **Code** node (`n8n-nodes-base.code`) named `Unified Intake & Normalizer`.
   - Paste JavaScript normalization logic to extract message text, session ID, email, and channel metadata from input payloads.
   - Connect all three triggers into this normalizer node.
4. **Configure Input Security:**
   - Add a **SoterAI** node (`n8n-nodes-soterai.soterGuard`) named `🛡️ SoterAI Input Threat Shield`.
   - Set Operation to `inputGuard`, Resource to `guardrail`, Sensitivity to `BALANCED`, and Action on Threat to `BLOCK`.
   - Configure allowed topics (`orders, tracking, billing, returns`) and custom user refusal messages.
   - Connect `Unified Intake & Normalizer` to this node.
   - Configure SoterAI API credentials (`soterApi`).
5. **Configure AI Copilot & Sub-Nodes:**
   - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`) named `🧠 Autonomous Support Copilot`. Set prompt type to define with a customer support system prompt.
   - Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`), select model `gpt-4o-mini`, set temperature to `0.2`, and add OpenAI API credentials (`openAiApi`). Connect its output to the AI Agent (`ai_languageModel`).
   - Add a **Window Buffer Memory** node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`), set context length to `10`, and bind session key to `={{ $('Unified Intake & Normalizer').item.json.sessionId }}`. Connect to the AI Agent (`ai_memory`).
   - Add a **Code Tool** node (`@n8n/n8n-nodes-langchain.toolCode`) named `Order Lookup Tool` with mock order lookup logic for IDs like `INV-8842` and `ORD-1029`. Connect to the AI Agent (`ai_tool`).
   - Add a **Calculator Tool** node (`@n8n/n8n-nodes-langchain.toolCalculator`) named `Refund & Policy Calculator`. Connect to the AI Agent (`ai_tool`).
   - Connect Safe Output (Output 0) of `🛡️ SoterAI Input Threat Shield` to `🧠 Autonomous Support Copilot`.
6. **Configure Output DLP & Delivery:**
   - Add a second **SoterAI** node (`n8n-nodes-soterai.soterGuard`) named `🔒 SoterAI Output Guardrail (DLP)`.
   - Set Operation to `outputGuard`, Resource to `guardrail`, and Action on Threat to `REDACT`. Connect `🧠 Autonomous Support Copilot` to this node.
   - Connect Output 0 (Clean/Redacted) to:
     - A **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`) named `Deliver Response to Customer`.
     - A **Set** node (`n8n-nodes-base.set`) named `Deliver Chat / Web Response`.
   - Connect Output 1 (Threat/Leak) to a **Set** node named `🚨 SecOps Output DLP Leak Alert`.
7. **Configure SecOps Incident Response:**
   - Connect Threat Output (Output 1) of `🛡️ SoterAI Input Threat Shield` to a **Set** node (`n8n-nodes-base.set`) named `🚨 SecOps Threat Incident Formatter`.
   - Connect this formatter node to:
     - An **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Dispatch SecOps Alert (Slack/Webhook)` configured as a POST request to your alert endpoint (e.g., Slack/Teams/SIEM webhook).
     - A **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`) named `Send Security Rejection to Customer` returning blocked incident responses.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Production-grade autonomous AI customer support copilot protected end-to-end by SoterAI (`n8n-nodes-soterai` v3) & LangChain. | Architecture Overview |
| OWASP LLM01 Prompt Injection Defense • OWASP LLM06 Sensitive Data Redaction • Dual-Branch Security Routing • Verified ERP Tools • Multi-Turn Memory. | Compliance & Security Features |
| Tested with 100% pass rate in real Docker production environments across prompt injection attacks, legitimate customer support queries, and sensitive data leakage scenarios. | Field Testing & Reliability |