Generate workflows from natural language using GPT-4o Mini and n8nBuilder API

https://n8nworkflows.xyz/workflows/generate-workflows-from-natural-language-using-gpt-4o-mini-and-n8nbuilder-api-12064


# Generate workflows from natural language using GPT-4o Mini and n8nBuilder API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

This workflow generates complete **n8n workflow JSON** from a **natural-language description** by calling the **n8nBuilder API** (`https://api.n8nbuilder.dev/api/generate`). It provides **three entry modes**:

1. **Form Mode**: a public form where users enter API token, email, and request text; the workflow displays a success/error completion page with the generated JSON.
2. **AI Chat Mode**: a chat interface powered by **GPT-4o Mini** that forwards user requests to an HTTP “tool” node (n8nBuilder API) and returns the generated workflow JSON.
3. **Webhook Mode**: a programmatic HTTP endpoint (`POST /webhook/generate-workflow`) that returns JSON success/error responses.

### 1.1 Input Reception (3 entry points)
- Form Trigger (interactive UI)
- Chat Trigger (conversational interface)
- Webhook Trigger (API integration)

### 1.2 Workflow Generation (n8nBuilder API call)
- HTTP requests to n8nBuilder with user token (`x-user-token`) and payload (`email`, `query`)

### 1.3 Response Normalization + Validation
- `Set` nodes normalize output and add timestamps/flags
- `IF` nodes validate “looks like JSON” by checking output length

### 1.4 Output Delivery
- Form completion pages (success vs error)
- Webhook HTTP responses (200 vs 400)
- Chat output (via agent main output)

---

## 2. Block-by-Block Analysis

### Block A — Form Mode: Collect input and generate workflow
**Overview:** Presents a form UI, sends inputs to n8nBuilder API, then routes to success/error completion pages based on a simple validation flag.

**Nodes involved:**
- `Workflow Generator Form`
- `Call n8nBuilder API`
- `Process API Response`
- `Handle API Errors`
- `Show Results`
- `Show Error Message`

#### Node: Workflow Generator Form
- **Type / role:** `n8n-nodes-base.formTrigger` — Entry point (Form Mode).
- **Configuration (interpreted):**
  - Form title: “n8n Workflow Generator”
  - Description: “Generate complete n8n workflows… using the n8nBuilder API”
  - Button label: “Generate Workflow”
  - Attribution disabled.
  - Fields (all required):
    - `api_token` (text): “n8nBuilder API Token”
    - `email` (email): “Your Email Address”
    - `query` (textarea): “Workflow Description”
- **Inputs/Outputs:**
  - **Outputs:** Main → `Call n8nBuilder API`
  - Produces JSON containing `api_token`, `email`, `query` at top-level (`$json.*`).
- **Edge cases / failures:**
  - Users submit invalid email format (form validation).
  - Missing/blank required fields (blocked by form).
- **Version notes:** typeVersion `2.4`.

#### Node: Call n8nBuilder API
- **Type / role:** `n8n-nodes-base.httpRequest` — Calls n8nBuilder generation endpoint.
- **Configuration:**
  - POST `https://api.n8nbuilder.dev/api/generate`
  - Timeout: 60s
  - Headers:
    - `Content-Type: application/json`
    - `x-user-token: {{$json.api_token}}`
  - Body parameters:
    - `email: {{$json.email}}`
    - `query: {{$json.query}}`
- **Inputs/Outputs:**
  - **Input:** From `Workflow Generator Form`
  - **Output:** Main → `Process API Response`
  - Expects response containing `output` (string) from API.
- **Edge cases / failures:**
  - 401/403 if token invalid.
  - 429 if quota/limits reached.
  - 5xx / network timeout.
  - Response schema differs (missing `output`).
- **Version notes:** typeVersion `4.2`.

#### Node: Process API Response
- **Type / role:** `n8n-nodes-base.set` — Normalizes response and adds metadata.
- **Configuration:**
  - Sets:
    - `output` = `{{$json.output}}`
    - `timestamp` = `{{$now.toISO()}}`
    - `query_description` = `{{$('Workflow Generator Form').item.json.query}}`
    - `has_valid_json` (boolean) = `{{$json.output && $json.output.length > 10}}`
      - This is a **heuristic**: checks only length, not JSON validity.
- **Inputs/Outputs:**
  - Input from `Call n8nBuilder API`
  - Output → `Handle API Errors`
- **Edge cases / failures:**
  - If API returns object/array in `output` instead of string, length check may behave unexpectedly.
  - If node name changes, expression `$('Workflow Generator Form')...` will break.
- **Version notes:** typeVersion `3.4`.

#### Node: Handle API Errors
- **Type / role:** `n8n-nodes-base.if` — Branching based on `has_valid_json`.
- **Configuration:**
  - Condition: `{{$json.has_valid_json}} == true`
- **Inputs/Outputs:**
  - True branch → `Show Results`
  - False branch → `Show Error Message`
- **Edge cases / failures:**
  - False positives: long error strings pass length check.
  - False negatives: short but valid JSON fails.
- **Version notes:** typeVersion `2`.

#### Node: Show Results
- **Type / role:** `n8n-nodes-base.form` — Form completion page (success).
- **Configuration:**
  - Operation: `completion`
  - Title: “Workflow Generated Successfully!”
  - Message includes:
    - `{{ $json.timestamp }}`
    - `{{ $json.query_description }}`
    - JSON output embedded in a code block: `{{ $json.output }}`
    - Link: https://community.n8n.io
- **Inputs/Outputs:**
  - Input from `Handle API Errors` (true)
  - No downstream nodes.
- **Edge cases / failures:**
  - If `output` contains invalid/unescaped content, display may be messy (but usually fine as plain text).
- **Version notes:** typeVersion `2.4`.

#### Node: Show Error Message
- **Type / role:** `n8n-nodes-base.form` — Form completion page (error).
- **Configuration:**
  - Operation: `completion`
  - Title: “Generation Failed”
  - Message lists likely causes and links:
    - Dashboard: https://n8nbuilder.dev/dashboard
    - Support: https://n8nbuilder.dev/support
  - Shows response snippet: `{{ $json.output || 'No response from API' }}`
- **Inputs/Outputs:**
  - Input from `Handle API Errors` (false)
- **Edge cases / failures:**
  - If the HTTP request node hard-fails (execution stops), this node may never run unless you configure error handling in n8n (workflow-level or node-level).
- **Version notes:** typeVersion `2.4`.

---

### Block B — AI Chat Mode: Conversational generation via agent + tool
**Overview:** Accepts chat messages, uses GPT-4o Mini with an agent that is instructed to call the “Generate Workflow Tool”, then returns the tool output as the chat response.

**Nodes involved:**
- `When chat message received`
- `GPT-4o Mini Model`
- `Generate Workflow Tool`
- `n8nBuilder Chat Assistant`
- `Extract Generated Workflow`

#### Node: When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Entry point for chat.
- **Configuration:**
  - Default options (no custom parameters set)
- **Inputs/Outputs:**
  - Output → `n8nBuilder Chat Assistant`
- **Edge cases / failures:**
  - Chat channel availability depends on n8n chat feature configuration.
- **Version notes:** typeVersion `1.4`.

#### Node: GPT-4o Mini Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Language model used by the agent.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.1` (low variance, more deterministic)
  - Built-in tools: none enabled here (tooling is provided via the separate HTTP Request Tool node).
  - **Credentials:** Requires OpenAI credentials configured in n8n.
- **Connections:**
  - AI Language Model output → `n8nBuilder Chat Assistant` (connection type `ai_languageModel`)
- **Edge cases / failures:**
  - Missing/invalid OpenAI API key.
  - Model not available for the credential/region.
  - Rate limits/timeouts.
- **Version notes:** typeVersion `1.3`.

#### Node: Generate Workflow Tool
- **Type / role:** `n8n-nodes-base.httpRequestTool` — A “tool” callable by the agent (LangChain tool interface) to hit the n8nBuilder API.
- **Configuration:**
  - POST `https://api.n8nbuilder.dev/api/generate`
  - Timeout: 60s
  - Headers:
    - `x-user-token: YOUR_USER_TOKEN` (placeholder; must be replaced)
    - `Content-Type: application/json`
  - Body:
    - `email: YOUR_EMAIL_ADRESS` (placeholder; must be replaced)
    - `query: {{$fromAI('parameters1_Value', ``, 'string')}}`
      - Pulls the tool argument from the agent/tool invocation.
- **Connections:**
  - Tool output → `n8nBuilder Chat Assistant` (connection type `ai_tool`)
- **Edge cases / failures:**
  - Placeholders not replaced → authentication failure.
  - If the agent passes an empty query, API may return an error or low-quality result.
  - API schema changes (expects different fields).
- **Version notes:** typeVersion `4.3`.

#### Node: n8nBuilder Chat Assistant
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates model + tool calls.
- **Configuration:**
  - System message (key behavior):
    - “Your top priority is to forward request to ‘Generate Workflow Tool’ tool.”
    - “After gathering the response from the tool, simply return it as a response.”
- **Connections:**
  - Main input from `When chat message received`
  - AI model input from `GPT-4o Mini Model`
  - Tool input from `Generate Workflow Tool`
  - Main output → `Extract Generated Workflow`
- **Edge cases / failures:**
  - If the agent does not call the tool (prompt drift), output may be missing `output`.
  - Tool errors may surface as agent text instead of structured `output`.
- **Version notes:** typeVersion `3.1`.

#### Node: Extract Generated Workflow
- **Type / role:** `n8n-nodes-base.set` — Extracts the `output` field for downstream use.
- **Configuration:**
  - Sets `output` = `{{$json.output}}`
- **Connections:**
  - Input from `n8nBuilder Chat Assistant`
  - No downstream nodes shown (acts as terminal/placeholder for expansion).
- **Edge cases / failures:**
  - If agent returns plain text without `output`, this will set `output` to empty/undefined.
- **Version notes:** typeVersion `3.4`.

---

### Block C — Webhook Mode: Programmatic endpoint with JSON response
**Overview:** Exposes a webhook endpoint that accepts token/email/query in the request body, calls n8nBuilder API, validates response, and returns a JSON success or error payload.

**Nodes involved:**
- `Webhook Trigger`
- `Call n8nBuilder API (Webhook)`
- `Process Webhook Response`
- `Check Webhook Response`
- `Return Success Response`
- `Return Error Response`

#### Node: Webhook Trigger
- **Type / role:** `n8n-nodes-base.webhook` — Entry point for Webhook Mode.
- **Configuration:**
  - Path: `generate-workflow`
  - Method: `POST`
  - Response mode: `responseNode` (workflow must end with Respond to Webhook node)
- **Inputs/Outputs:**
  - Output → `Call n8nBuilder API (Webhook)`
  - Incoming payload expected in `$json.body`.
- **Edge cases / failures:**
  - If callers send non-JSON or wrong content-type, `$json.body.*` may be missing.
  - If no Respond node runs (e.g., crash), caller gets timeout.
- **Version notes:** typeVersion `2`.

#### Node: Call n8nBuilder API (Webhook)
- **Type / role:** `n8n-nodes-base.httpRequest` — Calls n8nBuilder using body parameters from the incoming webhook.
- **Configuration:**
  - POST `https://api.n8nbuilder.dev/api/generate`
  - Timeout: 60s
  - Headers:
    - `Content-Type: application/json`
    - `x-user-token: {{$json.body.api_token}}`
  - Body:
    - `email: {{$json.body.email}}`
    - `query: {{$json.body.query}}`
- **Inputs/Outputs:**
  - Input from `Webhook Trigger`
  - Output → `Process Webhook Response`
- **Edge cases / failures:**
  - Missing body fields leads to invalid API request.
  - Auth/quota/network errors as in Form Mode.
- **Version notes:** typeVersion `4.2`.

#### Node: Process Webhook Response
- **Type / role:** `n8n-nodes-base.set` — Normalizes response for downstream branching and responding.
- **Configuration:**
  - `success` (boolean) = `{{$json.output && $json.output.length > 10}}`
  - `workflow_json` (string) = `{{$json.output}}`
  - `timestamp` = `{{$now.toISO()}}`
- **Inputs/Outputs:**
  - Output → `Check Webhook Response`
- **Edge cases / failures:**
  - Same “length heuristic” limitations as Form Mode.
- **Version notes:** typeVersion `3.4`.

#### Node: Check Webhook Response
- **Type / role:** `n8n-nodes-base.if` — Branches to success/error responders.
- **Configuration:** `{{$json.success}} == true`
- **Outputs:**
  - True → `Return Success Response`
  - False → `Return Error Response`
- **Version notes:** typeVersion `2`.

#### Node: Return Success Response
- **Type / role:** `n8n-nodes-base.respondToWebhook` — Returns HTTP 200 JSON payload.
- **Configuration:**
  - Response code: 200
  - Respond with: JSON
  - Body (expression):
    ```js
    {
      "success": true,
      "timestamp": "{{ $json.timestamp }}",
      "workflow": {{ $json.workflow_json }}
    }
    ```
  - Note: `workflow` is injected **without quotes**, assuming `workflow_json` is valid JSON text.
- **Edge cases / failures:**
  - If `workflow_json` is a string containing JSON (not parsed), direct injection can produce invalid JSON response.
  - If `workflow_json` includes leading/trailing text, response breaks.
- **Version notes:** typeVersion `1.1`.

#### Node: Return Error Response
- **Type / role:** `n8n-nodes-base.respondToWebhook` — Returns HTTP 400 JSON payload.
- **Configuration:**
  - Response code: 400
  - JSON body includes fixed error message and timestamp.
- **Edge cases / failures:**
  - If timestamp is missing (upstream failure), response may render with empty value.
- **Version notes:** typeVersion `1.1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point | — | n8nBuilder Chat Assistant | AI Chat Mode\n\nConversational workflow generation using OpenAI.\nConfigure OpenAI credentials and update YOUR_EMAIL_ADRESS / YOUR_USER_TOKEN in the tool node. |
| n8nBuilder Chat Assistant | @n8n/n8n-nodes-langchain.agent | Agent that calls tool and returns result | When chat message received | Extract Generated Workflow | AI Chat Mode\n\nConversational workflow generation using OpenAI.\nConfigure OpenAI credentials and update YOUR_EMAIL_ADRESS / YOUR_USER_TOKEN in the tool node. |
| GPT-4o Mini Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backing the agent | — (AI connection) | n8nBuilder Chat Assistant (ai_languageModel) | AI Chat Mode\n\nConversational workflow generation using OpenAI.\nConfigure OpenAI credentials and update YOUR_EMAIL_ADRESS / YOUR_USER_TOKEN in the tool node. |
| Generate Workflow Tool | n8n-nodes-base.httpRequestTool | Tool callable by agent to generate workflow via API | — (AI tool connection) | n8nBuilder Chat Assistant (ai_tool) | AI Chat Mode\n\nConversational workflow generation using OpenAI.\nConfigure OpenAI credentials and update YOUR_EMAIL_ADRESS / YOUR_USER_TOKEN in the tool node. |
| Extract Generated Workflow | n8n-nodes-base.set | Extract `output` from agent result | n8nBuilder Chat Assistant | — | AI Chat Mode\n\nConversational workflow generation using OpenAI.\nConfigure OpenAI credentials and update YOUR_EMAIL_ADRESS / YOUR_USER_TOKEN in the tool node. |
| Workflow Generator Form | n8n-nodes-base.formTrigger | Form entry point | — | Call n8nBuilder API | Form Mode\n\nSimple form-based workflow generation.\nUsers enter API token, email, and workflow description. |
| Call n8nBuilder API | n8n-nodes-base.httpRequest | Calls generation API from form inputs | Workflow Generator Form | Process API Response | Form Mode\n\nSimple form-based workflow generation.\nUsers enter API token, email, and workflow description. |
| Process API Response | n8n-nodes-base.set | Adds timestamp, query echo, validity flag | Call n8nBuilder API | Handle API Errors | Response and Error Handling\n\nValidates API response and displays success or error message to the user. |
| Handle API Errors | n8n-nodes-base.if | Routes to success/error form completion | Process API Response | Show Results; Show Error Message | Response and Error Handling\n\nValidates API response and displays success or error message to the user. |
| Show Results | n8n-nodes-base.form | Success completion page with JSON output | Handle API Errors (true) | — | Response and Error Handling\n\nValidates API response and displays success or error message to the user. |
| Show Error Message | n8n-nodes-base.form | Error completion page with troubleshooting | Handle API Errors (false) | — | Response and Error Handling\n\nValidates API response and displays success or error message to the user. |
| Webhook Trigger | n8n-nodes-base.webhook | Webhook entry point | — | Call n8nBuilder API (Webhook) | Webhook Mode\n\nProgrammatic API integration. POST to /webhook/generate-workflow with JSON body: { "api_token": "...", "email": "...", "query": "..." } |
| Call n8nBuilder API (Webhook) | n8n-nodes-base.httpRequest | Calls generation API from webhook body | Webhook Trigger | Process Webhook Response | Webhook Mode\n\nProgrammatic API integration. POST to /webhook/generate-workflow with JSON body: { "api_token": "...", "email": "...", "query": "..." } |
| Process Webhook Response | n8n-nodes-base.set | Normalizes API response for webhook | Call n8nBuilder API (Webhook) | Check Webhook Response | Response and Error Handling\n\nValidates API response and displays success or error message to the user. |
| Check Webhook Response | n8n-nodes-base.if | Routes to webhook success/error response | Process Webhook Response | Return Success Response; Return Error Response | Response and Error Handling\n\nValidates API response and displays success or error message to the user. |
| Return Success Response | n8n-nodes-base.respondToWebhook | Returns 200 with generated workflow | Check Webhook Response (true) | — | Response and Error Handling\n\nValidates API response and displays success or error message to the user. |
| Return Error Response | n8n-nodes-base.respondToWebhook | Returns 400 with error message | Check Webhook Response (false) | — | Response and Error Handling\n\nValidates API response and displays success or error message to the user. |
| n8nBuilder Workflow Generator | n8n-nodes-base.stickyNote | Documentation / usage notes | — | — |  |
| AI Chat Mode | n8n-nodes-base.stickyNote | Documentation / usage notes | — | — |  |
| Form Mode | n8n-nodes-base.stickyNote | Documentation / usage notes | — | — |  |
| Response and Error Handling | n8n-nodes-base.stickyNote | Documentation / usage notes | — | — |  |
| Advanced Options | n8n-nodes-base.stickyNote | Documentation / links | — | — |  |
| Webhook Mode | n8n-nodes-base.stickyNote | Documentation / usage notes | — | — |  |
| Response and Error Handling1 | n8n-nodes-base.stickyNote | Documentation / usage notes | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

### A. Form Mode (UI)
1. **Create node:** *Form Trigger* (`Form Trigger`)
   - Title: `n8n Workflow Generator`
   - Description: `Generate complete n8n workflows from natural language descriptions using the n8nBuilder API`
   - Button label: `Generate Workflow`
   - Fields (required):
     - `api_token` (text)
     - `email` (email)
     - `query` (textarea)
2. **Create node:** *HTTP Request* (`Call n8nBuilder API`)
   - Method: POST
   - URL: `https://api.n8nbuilder.dev/api/generate`
   - Timeout: `60000`
   - Headers:
     - `Content-Type` = `application/json`
     - `x-user-token` = expression `{{$json.api_token}}`
   - Body fields:
     - `email` = `{{$json.email}}`
     - `query` = `{{$json.query}}`
3. **Create node:** *Set* (`Process API Response`)
   - Add fields:
     - `output` (string) = `{{$json.output}}`
     - `timestamp` (string) = `{{$now.toISO()}}`
     - `query_description` (string) = `{{$('Workflow Generator Form').item.json.query}}` (adjust node name if different)
     - `has_valid_json` (boolean) = `{{$json.output && $json.output.length > 10}}`
4. **Create node:** *IF* (`Handle API Errors`)
   - Boolean condition: `{{$json.has_valid_json}}` equals `true`
5. **Create node:** *Form* (`Show Results`) with **operation = Completion**
   - Completion title: `Workflow Generated Successfully!`
   - Completion message: include `{{$json.timestamp}}`, `{{$json.query_description}}`, and code block with `{{$json.output}}`
6. **Create node:** *Form* (`Show Error Message`) with **operation = Completion**
   - Completion title: `Generation Failed`
   - Completion message: include troubleshooting and show `{{$json.output || 'No response from API'}}`
7. **Connect nodes:**
   - Form Trigger → HTTP Request → Set → IF
   - IF(true) → Show Results
   - IF(false) → Show Error Message

### B. AI Chat Mode (OpenAI + Agent + Tool)
8. **Create node:** *Chat Trigger* (`When chat message received`)
9. **Create node:** *OpenAI Chat Model* (`GPT-4o Mini Model`)
   - Model: `gpt-4o-mini`
   - Temperature: `0.1`
   - **Credentials:** configure OpenAI API key in n8n credentials and select it here.
10. **Create node:** *HTTP Request Tool* (`Generate Workflow Tool`)
    - Method: POST
    - URL: `https://api.n8nbuilder.dev/api/generate`
    - Timeout: `60000`
    - Headers:
      - `Content-Type` = `application/json`
      - `x-user-token` = replace with your token (or use a credential/variable strategy)
    - Body:
      - `email` = replace with your email
      - `query` = expression `{{$fromAI('parameters1_Value', '', 'string')}}`
11. **Create node:** *AI Agent* (`n8nBuilder Chat Assistant`)
    - System message:
      - Instruct it to call “Generate Workflow Tool” and return the tool response directly.
12. **Create node:** *Set* (`Extract Generated Workflow`)
    - `output` = `{{$json.output}}`
13. **Connect nodes:**
    - Chat Trigger (main) → Agent (main)
    - OpenAI Model → Agent (AI language model connection)
    - HTTP Request Tool → Agent (AI tool connection)
    - Agent (main) → Extract Generated Workflow

### C. Webhook Mode (API endpoint)
14. **Create node:** *Webhook* (`Webhook Trigger`)
    - HTTP Method: POST
    - Path: `generate-workflow`
    - Response mode: `Response node`
15. **Create node:** *HTTP Request* (`Call n8nBuilder API (Webhook)`)
    - POST `https://api.n8nbuilder.dev/api/generate`
    - Headers:
      - `Content-Type: application/json`
      - `x-user-token` = `{{$json.body.api_token}}`
    - Body:
      - `email` = `{{$json.body.email}}`
      - `query` = `{{$json.body.query}}`
16. **Create node:** *Set* (`Process Webhook Response`)
    - `success` = `{{$json.output && $json.output.length > 10}}`
    - `workflow_json` = `{{$json.output}}`
    - `timestamp` = `{{$now.toISO()}}`
17. **Create node:** *IF* (`Check Webhook Response`)
    - Condition: `{{$json.success}} == true`
18. **Create node:** *Respond to Webhook* (`Return Success Response`)
    - Response code: 200
    - Respond with: JSON
    - Body:
      - `success: true`
      - `timestamp: {{$json.timestamp}}`
      - `workflow: {{$json.workflow_json}}` (be careful: requires valid JSON, not just a string)
19. **Create node:** *Respond to Webhook* (`Return Error Response`)
    - Response code: 400
    - JSON body with `success: false`, error message, `timestamp`.
20. **Connect nodes:**
    - Webhook → HTTP Request → Set → IF
    - IF(true) → Return Success Response
    - IF(false) → Return Error Response

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get API token from n8nbuilder.dev and provide it via Form/Webhook; for Chat Mode replace placeholders in the tool node. | n8nBuilder Workflow Generator sticky note |
| Community node available: `npm install n8n-nodes-n8nbuilder` | Mentioned under “Advanced Options” |
| GitHub: `github.com/mbakgun/n8n-nodes-n8nbuilder` | “Advanced Options” sticky note |
| Dashboard (token/quota): https://n8nbuilder.dev/dashboard | Used in error completion message |
| Support: https://n8nbuilder.dev/support | Used in error completion message |
| n8n Community: https://community.n8n.io | Linked in success completion message |

