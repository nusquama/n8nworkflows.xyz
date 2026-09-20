Serve a RAG chat backend for Next.js using OpenAI, Redis, and Pinecone

https://n8nworkflows.xyz/workflows/serve-a-rag-chat-backend-for-next-js-using-openai--redis--and-pinecone-19669


# Serve a RAG chat backend for Next.js using OpenAI, Redis, and Pinecone

### 1. Workflow Overview

This workflow functions as a secure backend API for a Next.js application, executing a Retrieval-Augmented Generation (RAG) chat agent. Next.js communicates by sending `POST` requests to a header-authenticated webhook, offloading heavy tasks such as request validation, Redis caching, vector retrieval via Pinecone, conversational LLM processing via OpenAI, and Postgres auditing to n8n. Additionally, it asynchronously triggers Incremental Static Regeneration (ISR) revalidation callbacks back to the Next.js application when content changes.

The workflow is categorized into the following logical blocks:
- **1.1 Input Reception & Configuration:** Receives incoming HTTP requests and establishes global configuration variables.
- **1.2 Request Validation:** Validates incoming payloads against structural rules and rejects malformed requests early.
- **1.3 Caching Layer:** Computes deterministic hashes to check for cached answers in Redis, bypassing heavy processing on repeat queries.
- **1.4 Retrieval & RAG Processing:** Embeds queries, fetches context from Pinecone, maintains conversation memory, and runs an OpenAI-powered RAG agent.
- **1.5 Output Processing & Logging:** Validates the agent's output, caches the result, logs turns to Postgres, and responds to the client.
- **1.6 Next.js ISR Revalidation:** Conditionally triggers out-of-band cache revalidation requests back to the Next.js frontend.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block serves as the entry point, receiving HTTP `POST` requests from the Next.js backend and defining default operational parameters.
- **Nodes Involved:** `Webhook - RAG Agent Request`, `Set - Config`
- **Node Details:**
  - **Webhook - RAG Agent Request**
    - *Type and technical role:* Webhook trigger (`n8n-nodes-base.webhook`). Listens for incoming HTTP `POST` requests.
    - *Configuration choices:* Configured for `POST` on path `rag-agent-query` with Header Authentication enabled (`x-api-key`).
    - *Input/Output:* No input connections; outputs to `Set - Config`.
    - *Failure types:* Authentication errors (`401 Unauthorized`) if the `x-api-key` header is missing or incorrect.
  - **Set - Config**
    - *Type and technical role:* Set node (`n8n-nodes-base.set`). Centralizes application configuration.
    - *Configuration choices:* Defines `vectorNamespaceDefault`, `allowedNamespaces`, `embeddingModel`, `chatModel`, `cacheTtlSeconds`, `maxQueryLength`, `topK`, `nextjsRevalidateUrl`, and `nextjsRevalidateSecret`.
    - *Input/Output:* Input from `Webhook - RAG Agent Request`; output to `Code - Validate Request Schema`.

#### 2.2 Request Validation
This block validates the incoming payload to ensure required fields are present and properly formatted before incurring LLM or database costs.
- **Nodes Involved:** `Code - Validate Request Schema`, `IF - Validation Passed`, `Respond - Validation Error (400)`
- **Node Details:**
  - **Code - Validate Request Schema**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Executes custom JavaScript to perform Zod-style validation.
    - *Configuration choices:* Checks that `query`, `conversationId`, and `tenantId` are non-empty strings, ensures `query` does not exceed `maxQueryLength`, and validates `namespace` against `allowedNamespaces`.
    - *Input/Output:* Input from `Set - Config`; output to `IF - Validation Passed`.
  - **IF - Validation Passed**
    - *Type and technical role:* IF node (`n8n-nodes-base.if`). Routes requests based on validation results.
    - *Configuration choices:* Evaluates `{{ $json.isValid === true }}`.
    - *Input/Output:* Input from `Code - Validate Request Schema`; outputs to `Code - Build Cache Key` (true branch) or `Respond - Validation Error (400)` (false branch).
  - **Respond - Validation Error (400)**
    - *Type and technical role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns a structured error response.
    - *Configuration choices:* Sets HTTP response code to `400` and returns a JSON payload containing validation error details.
    - *Input/Output:* Input from `IF - Validation Passed` (false branch); terminal node.

#### 2.3 Caching Layer
This block computes a deterministic cache key and queries Redis to return cached answers for identical requests.
- **Nodes Involved:** `Code - Build Cache Key`, `Redis - Check Query Cache`, `IF - Cache Hit`, `Respond - Success (Cached)`
- **Node Details:**
  - **Code - Build Cache Key**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Normalizes strings and computes hashes.
    - *Configuration choices:* Normalizes the query (lowercase, trimmed, collapsed whitespace) and creates a SHA-256 hash using tenant ID, namespace, and normalized query.
    - *Input/Output:* Input from `IF - Validation Passed`; output to `Redis - Check Query Cache`.
  - **Redis - Check Query Cache**
    - *Type and technical role:* Redis node (`n8n-nodes-base.redis`). Retrieves cached values.
    - *Configuration choices:* Uses `get` operation with the generated `cacheKey`. Set to continue on fail.
    - *Input/Output:* Input from `Code - Build Cache Key`; output to `IF - Cache Hit`.
  - **IF - Cache Hit**
    - *Type and technical role:* IF node (`n8n-nodes-base.if`). Checks if a valid Redis value was returned.
    - *Configuration choices:* Evaluates whether `{{ $json.value }}` exists.
    - *Input/Output:* Input from `Redis - Check Query Cache`; outputs to `Respond - Success (Cached)` (true branch) or `AI Agent - RAG Assistant` (false branch).
  - **Respond - Success (Cached)**
    - *Type and technical role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns the cached payload.
    - *Configuration choices:* Sets HTTP status `200` and appends `cacheHit: true` to the response envelope.
    - *Input/Output:* Input from `IF - Cache Hit` (true branch); terminal node.

#### 2.4 Retrieval & RAG Processing
On a cache miss, this block embeds the query, retrieves vector passages, and invokes an LLM agent equipped with conversational memory and retrieval tools.
- **Nodes Involved:** `Embeddings - OpenAI`, `Vector Store - Retrieve Context`, `Tool - Search Knowledge Base`, `OpenAI Chat Model`, `Window Buffer Memory`, `AI Agent - RAG Assistant`
- **Node Details:**
  - **Embeddings - OpenAI**
    - *Type and technical role:* LangChain Embedding model (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`). Generates vector embeddings for queries.
    - *Configuration choices:* Uses default settings with OpenAI credentials.
    - *Input/Output:* Connects to `Vector Store - Retrieve Context`.
  - **Vector Store - Retrieve Context**
    - *Type and technical role:* LangChain Pinecone vector store integration (`@n8n/n8n-nodes-langchain.vectorStorePinecone`). Searches the vector database.
    - *Configuration choices:* Scopes queries to the Pinecone namespace evaluated dynamically from `{{ $('Code - Build Cache Key').first().json.namespace }}`.
    - *Input/Output:* Connects to `Tool - Search Knowledge Base`.
  - **Tool - Search Knowledge Base**
    - *Type and technical role:* LangChain vector store tool (`@n8n/n8n-nodes-langchain.toolVectorStore`). Exposes retrieval as a callable agent tool.
    - *Configuration choices:* Sets `topK` dynamically from configuration.
    - *Input/Output:* Connects to `AI Agent - RAG Assistant`.
  - **OpenAI Chat Model**
    - *Type and technical role:* LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Provides the underlying LLM.
    - *Configuration choices:* Model name references `{{ $('Code - Build Cache Key').first().json.chatModel }}`.
    - *Input/Output:* Connects to `AI Agent - RAG Assistant`.
  - **Window Buffer Memory**
    - *Type and technical role:* LangChain Memory Buffer (`@n8n/n8n-nodes-langchain.memoryBufferWindow`). Maintains multi-turn chat history.
    - *Configuration choices:* Uses a custom session key mapped to `conversationId` with a context window length of `12`.
    - *Input/Output:* Connects to `AI Agent - RAG Assistant`.
  - **AI Agent - RAG Assistant**
    - *Type and technical role:* LangChain Agent (`@n8n/n8n-nodes-langchain.agent`). Orchestrates the RAG workflow.
    - *Configuration choices:* Defined system prompt enforces strict groundedness, source citation, and a structured JSON output format.
    - *Input/Output:* Inputs from memory, chat model, tool, and vector store; output to `Code - Parse & Validate Agent Output`.

#### 2.5 Output Processing & Logging
This block validates the model's output, caches the successful result, logs execution telemetry to Postgres, and responds to the client.
- **Nodes Involved:** `Code - Parse & Validate Agent Output`, `Redis - Write Query Cache`, `Postgres - Log Conversation Turn`, `Respond - Success (200)`
- **Node Details:**
  - **Code - Parse & Validate Agent Output**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Safely parses and validates agent JSON output.
    - *Configuration choices:* Enforces schema checks on `answer`, `citations`, `groundedness`, and `confidencePercent`. Falls back to a safe fallback shape if validation fails.
    - *Input/Output:* Input from `AI Agent - RAG Assistant`; output to `Redis - Write Query Cache`.
  - **Redis - Write Query Cache**
    - *Type and technical role:* Redis node (`n8n-nodes-base.redis`). Caches the processed answer.
    - *Configuration choices:* Uses `set` operation with TTL defined by `cacheTtlSeconds`. Set to continue on fail.
    - *Input/Output:* Input from `Code - Parse & Validate Agent Output`; output to `Postgres - Log Conversation Turn`.
  - **Postgres - Log Conversation Turn**
    - *Type and technical role:* Postgres node (`n8n-nodes-base.postgres`). Persists audit logs.
    - *Configuration choices:* Inserts conversation telemetry into the `rag_conversation_turns` table. Set to continue on fail.
    - *Input/Output:* Input from `Redis - Write Query Cache`; outputs to `Respond - Success (200)` and `IF - Content Changed Requires Revalidation`.
  - **Respond - Success (200)`
    - *Type and technical role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns the success envelope.
    - *Configuration choices:* Sets response code `200` and returns structured JSON data containing the answer, citations, groundedness, confidence, and cache status.
    - *Input/Output:* Input from `Postgres - Log Conversation Turn`; terminal node.

#### 2.6 Next.js ISR Revalidation
This block handles out-of-band ISR cache revalidation requests sent back to the Next.js application when responses are grounded.
- **Nodes Involved:** `IF - Content Changed Requires Revalidation`, `HTTP Request - Notify Next.js Revalidation`, `Code - Handle Revalidation Failure`, `NoOp - Revalidation Skipped`
- **Node Details:**
  - **IF - Content Changed Requires Revalidation**
    - *Type and technical role:* IF node (`n8n-nodes-base.if`). Determines if revalidation is needed.
    - *Configuration choices:* Evaluates whether `{{ $json.groundedness }}` equals `"grounded"`.
    - *Input/Output:* Input from `Postgres - Log Conversation Turn`; outputs to `HTTP Request - Notify Next.js Revalidation` (true branch) or `NoOp - Revalidation Skipped` (false branch).
  - **HTTP Request - Notify Next.js Revalidation**
    - *Type and technical role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Sends revalidation pings.
    - *Configuration choices:* Sends a `POST` request to `{{ $json.nextjsRevalidateUrl }}/api/revalidate` with a JSON body containing the ISR tag and shared secret. Timeout set to 5000ms. Set to continue on fail.
    - *Input/Output:* Input from `IF - Content Changed Requires Revalidation` (true branch); output to `Code - Handle Revalidation Failure`.
  - **Code - Handle Revalidation Failure**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Catches and logs callback failures.
    - *Configuration choices:* Checks for error properties and logs warnings non-fatally if the callback fails.
    - *Input/Output:* Input from `HTTP Request - Notify Next.js Revalidation`; terminal node.
  - **NoOp - Revalidation Skipped**
    - *Type and technical role:* No-Operation node (`n8n-nodes-base.noOp`). Terminal node for skipped revalidations.
    - *Configuration choices:* Performs no action, maintaining a complete execution audit trail.
    - *Input/Output:* Input from `IF - Content Changed Requires Revalidation` (false branch); terminal node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Workflow documentation | None | None | ## n8n as a Backend for Next.js — RAG Agent API... |
| Sticky Note - Trigger & Validation | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 1. Trigger, Config & Request Validation... |
| Sticky Note - Cache Layer | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 2. Cache Layer — Skip the Agent on Repeat Queries... |
| Sticky Note - Retrieval & RAG Agent | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 3. Retrieval & RAG Agent... |
| Sticky Note - Output Validation, Cache Write & Logging | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 4. Output Validation, Cache Write & Audit Log... |
| Sticky Note - Next.js ISR Revalidation Callback | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 5. n8n Calls Back Into Next.js — ISR Revalidation... |
| Webhook - RAG Agent Request | n8n-nodes-base.webhook | Receives incoming POST requests | None | Set - Config | ## n8n as a Backend for Next.js — RAG Agent API... |
| Set - Config | n8n-nodes-base.set | Centralizes workflow configuration | Webhook - RAG Agent Request | Code - Validate Request Schema | ## n8n as a Backend for Next.js — RAG Agent API... |
| Code - Validate Request Schema | n8n-nodes-base.code | Validates incoming request schema | Set - Config | IF - Validation Passed | ## 1. Trigger, Config & Request Validation... |
| IF - Validation Passed | n8n-nodes-base.if | Routes valid requests | Code - Validate Request Schema | Code - Build Cache Key, Respond - Validation Error (400) | ## 1. Trigger, Config & Request Validation... |
| Respond - Validation Error (400) | n8n-nodes-base.respondToWebhook | Returns 400 validation error | IF - Validation Passed | None | ## 1. Trigger, Config & Request Validation... |
| Code - Build Cache Key | n8n-nodes-base.code | Derives deterministic cache key | IF - Validation Passed | Redis - Check Query Cache | ## 2. Cache Layer — Skip the Agent on Repeat Queries... |
| Redis - Check Query Cache | n8n-nodes-base.redis | Looks up cache key in Redis | Code - Build Cache Key | IF - Cache Hit | ## 2. Cache Layer — Skip the Agent on Repeat Queries... |
| IF - Cache Hit | n8n-nodes-base.if | Branches on cache presence | Redis - Check Query Cache | Respond - Success (Cached), AI Agent - RAG Assistant | ## 2. Cache Layer — Skip the Agent on Repeat Queries... |
| Respond - Success (Cached) | n8n-nodes-base.respondToWebhook | Returns cached success response | IF - Cache Hit | None | ## 2. Cache Layer — Skip the Agent on Repeat Queries... |
| Tool - Search Knowledge Base | @n8n/n8n-nodes-langchain.toolVectorStore | Callable vector search tool | Vector Store - Retrieve Context | AI Agent - RAG Assistant | ## 3. Retrieval & RAG Agent... |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM backend | None | AI Agent - RAG Assistant | ## 3. Retrieval & RAG Agent... |
| Window Buffer Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Manages conversation memory | None | AI Agent - RAG Assistant | ## 3. Retrieval & RAG Agent... |
| AI Agent - RAG Assistant | @n8n/n8n-nodes-langchain.agent | Orchestrates RAG and reasoning | IF - Cache Hit, Tool - Search Knowledge Base, OpenAI Chat Model, Window Buffer Memory | Code - Parse & Validate Agent Output | ## 3. Retrieval & RAG Agent... |
| Code - Parse & Validate Agent Output | n8n-nodes-base.code | Validates agent JSON output | AI Agent - RAG Assistant | Redis - Write Query Cache | ## 4. Output Validation, Cache Write & Audit Log... |
| Redis - Write Query Cache | n8n-nodes-base.redis | Stores answer in Redis with TTL | Code - Parse & Validate Agent Output | Postgres - Log Conversation Turn | ## 4. Output Validation, Cache Write & Audit Log... |
| Postgres - Log Conversation Turn | n8n-nodes-base.postgres | Logs turn telemetry to Postgres | Redis - Write Query Cache | Respond - Success (200), IF - Content Changed Requires Revalidation | ## 4. Output Validation, Cache Write & Audit Log... |
| Respond - Success (200) | n8n-nodes-base.respondToWebhook | Returns success response to client | Postgres - Log Conversation Turn | None | ## 4. Output Validation, Cache Write & Audit Log... |
| IF - Content Changed Requires Revalidation | n8n-nodes-base.if | Checks if ISR revalidation is needed | Postgres - Log Conversation Turn | HTTP Request - Notify Next.js Revalidation, NoOp - Revalidation Skipped | ## 5. n8n Calls Back Into Next.js — ISR Revalidation... |
| HTTP Request - Notify Next.js Revalidation | n8n-nodes-base.httpRequest | POSTs revalidation tag to Next.js | IF - Content Changed Requires Revalidation | Code - Handle Revalidation Failure | ## 5. n8n Calls Back Into Next.js — ISR Revalidation... |
| Code - Handle Revalidation Failure | n8n-nodes-base.code | Handles revalidation callback errors | HTTP Request - Notify Next.js Revalidation | None | ## 5. n8n Calls Back Into Next.js — ISR Revalidation... |
| NoOp - Revalidation Skipped | n8n-nodes-base.noOp | Terminal node for skipped revalidation | IF - Content Changed Requires Revalidation | None | ## 5. n8n Calls Back Into Next.js — ISR Revalidation... |
| Embeddings - OpenAI | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates query vector embeddings | None | Vector Store - Retrieve Context | None |
| Vector Store - Retrieve Context | @n8n/n8n-nodes-langchain.vectorStorePinecone | Retrieves chunks from Pinecone | Embeddings - OpenAI | Tool - Search Knowledge Base | None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger Node:**
   - Add a **Webhook** node named `Webhook - RAG Agent Request`.
   - Set HTTP Method to `POST`, path to `rag-agent-query`, and Authentication to `Header Auth`. Configure the required header credential (`x-api-key`).

2. **Create the Configuration Node:**
   - Add a **Set** node named `Set - Config`.
   - Connect `Webhook - RAG Agent Request` to `Set - Config`.
   - Add string/number/array assignments for: `vectorNamespaceDefault`, `allowedNamespaces`, `embeddingModel`, `chatModel`, `cacheTtlSeconds`, `maxQueryLength`, `topK`, `nextjsRevalidateUrl`, and `nextjsRevalidateSecret`.

3. **Implement Request Validation:**
   - Add a **Code** node named `Code - Validate Request Schema` and connect `Set - Config` to it. Paste the Zod-style request validation JavaScript snippet.
   - Add an **IF** node named `IF - Validation Passed`. Connect validation code output to it, evaluating `{{ $json.isValid === true }}`.
   - Add a **Respond to Webhook** node named `Respond - Validation Error (400)`. Connect the false branch of the validation IF node, setting response code to `400` and returning the error JSON.

4. **Implement Caching Layer:**
   - Add a **Code** node named `Code - Build Cache Key` and connect the true branch of `IF - Validation Passed` to it. Include the hashing logic.
   - Add a **Redis** node named `Redis - Check Query Cache`. Set operation to `get`, key to `={{ $json.cacheKey }}`, and enable `Continue On Fail`. Connect cache key code node to it.
   - Add an **IF** node named `IF - Cache Hit`. Evaluate if `{{ $json.value }}` exists. Connect Redis check output to it.
   - Add a **Respond to Webhook** node named `Respond - Success (Cached)`. Connect the true branch of `IF - Cache Hit`, setting response code `200` and returning the cached response envelope.

5. **Configure RAG Agent and Sub-components:**
   - Add an **OpenAI Embeddings** node named `Embeddings - OpenAI`.
   - Add a **Pinecone Vector Store** node named `Vector Store - Retrieve Context`. Connect `Embeddings - OpenAI` to its embedding input, select your Pinecone index, and bind the namespace expression `={{ $('Code - Build Cache Key').first().json.namespace }}`.
   - Add a **Vector Store Tool** node named `Tool - Search Knowledge Base`. Connect `Vector Store - Retrieve Context` to its vector store input and set `topK` to `={{ $('Code - Build Cache Key').first().json.topK }}`.
   - Add an **OpenAI Chat Model** node named `OpenAI Chat Model`. Bind model configuration to `={{ $('Code - Build Cache Key').first().json.chatModel }}` and connect it to the agent language model input.
   - Add a **Window Buffer Memory** node named `Window Buffer Memory`. Set session key type to custom key `={{ $('Code - Build Cache Key').first().json.conversationId }}` with context window length `12`. Connect it to the agent memory input.
   - Add an **AI Agent** node named `AI Agent - RAG Assistant`. Set prompt type to define, using the system prompt instructing strict groundedness and JSON output formatting. Connect the false branch of `IF - Cache Hit`, chat model, memory, and tool nodes to the agent.

6. **Implement Output Parsing, Caching, and Logging:**
   - Add a **Code** node named `Code - Parse & Validate Agent Output`. Connect `AI Agent - RAG Assistant` to it and paste the output validation script.
   - Add a **Redis** node named `Redis - Write Query Cache`. Set operation to `set`, key to `={{ $json.cacheKey }}`, TTL to `={{ $json.cacheTtlSeconds }}`, value to the serialized output JSON, and enable `Continue On Fail`. Connect output validation code to it.
   - Add a **Postgres** node named `Postgres - Log Conversation Turn`. Set operation to insert into table `rag_conversation_turns` with columns matching session and telemetry fields. Enable `Continue On Fail`. Connect Redis write cache to it.
   - Add a **Respond to Webhook** node named `Respond - Success (200)`. Set response code `200` and return the success data envelope. Connect Postgres log node to it.

7. **Implement Next.js ISR Revalidation Callback:**
   - Add an **IF** node named `IF - Content Changed Requires Revalidation`. Connect Postgres log node to it and evaluate if `{{ $json.groundedness }}` equals `"grounded"`.
   - Add an **HTTP Request** node named `HTTP Request - Notify Next.js Revalidation`. Connect the true branch, setting method to `POST`, URL to `={{ $json.nextjsRevalidateUrl }}/api/revalidate`, timeout to 5000ms, and body payload containing tag and secret. Enable `Continue On Fail`.
   - Add a **Code** node named `Code - Handle Revalidation Failure`. Connect HTTP request node to it to log non-fatal warnings on failure.
   - Add a **NoOp** node named `NoOp - Revalidation Skipped`. Connect the false branch of the revalidation IF node to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| External Next.js API contract requiring x-api-key authentication and shared revalidation secret. | Setup instructions for Next.js Server Actions and Route Handlers. |
| Database schema dependency: rag_conversation_turns table. | Postgres schema initialization requirement. |