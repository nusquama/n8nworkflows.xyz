Scrape and ingest web pages into a Pinecone RAG stack with Firecrawl and OpenAI

https://n8nworkflows.xyz/workflows/scrape-and-ingest-web-pages-into-a-pinecone-rag-stack-with-firecrawl-and-openai-13964


# Scrape and ingest web pages into a Pinecone RAG stack with Firecrawl and OpenAI

## 1. Workflow Overview

This workflow serves two related purposes:

1. **Ingestion pipeline:** accept a URL through an HTTP webhook, validate and normalize it, scrape the page content with Firecrawl, generate OpenAI embeddings, and store the resulting vectors in a Pinecone index.
2. **RAG chat interface:** expose a chat entry point that lets users query the indexed knowledge base using an agent backed by an OpenRouter chat model, Pinecone retrieval, OpenAI embeddings, and Cohere reranking.

The workflow is organized into the following logical blocks.

### 1.1 Ingestion Entry and URL Validation
Receives a POST request containing a URL, checks whether the input is present and valid, and either forwards a normalized URL for scraping or returns a validation error response.

### 1.2 Web Scraping and Vector Ingestion
Uses Firecrawl to scrape the target page into structured content, transforms that content into documents, generates embeddings, and inserts them into Pinecone.

### 1.3 Ingestion Response Handling
Returns the final HTTP response to the original webhook caller after ingestion completes, or a 422 response when validation fails.

### 1.4 Chat-Based Retrieval Interface
Provides a second entry point through an n8n chat trigger, then routes the user query into an agent that can retrieve knowledge from Pinecone.

### 1.5 LLM, Memory, Retrieval, Embeddings, and Reranking
Supplies the chat agent with its required supporting components: OpenRouter as the language model, buffer memory for conversation continuity, Pinecone retrieval as a tool, OpenAI embeddings for query vectorization, and Cohere reranking for retrieval quality improvement.

---

## 2. Block-by-Block Analysis

## 2.1 Ingestion Entry and URL Validation

**Overview:**  
This block accepts a URL via webhook and ensures it is present and syntactically acceptable before any external API call is made. It is the main safeguard against invalid ingestion requests.

**Nodes Involved:**  
- Receive URL
- Validate and normalize URL
- Return URL validation error

### Node Details

#### Receive URL
- **Type and technical role:** `n8n-nodes-base.webhook`  
  HTTP entry point for ingestion requests.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Response mode: `responseNode`, meaning a dedicated Respond to Webhook node must send the HTTP response.
  - Webhook path is a generated UUID-like path.
- **Key expressions or variables used:**  
  None internally; downstream nodes read `body.url`.
- **Input and output connections:**  
  - No input
  - Outputs to: `Validate and normalize URL`
- **Version-specific requirements:**  
  Uses webhook node version `2.1`.
- **Edge cases or potential failure types:**  
  - If the workflow is inactive, production webhook calls will fail.
  - If the caller sends malformed JSON or omits `url`, downstream validation will reject it.
  - Since response mode is handled by another node, missing response execution could leave requests hanging.
- **Sub-workflow reference:**  
  None

#### Validate and normalize URL
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript validation and normalization layer.
- **Configuration choices:**  
  - `onError` is set to `continueErrorOutput`, allowing execution to continue even if the code throws an error.
  - Reads the first incoming item’s request body.
  - Normalizes the URL by stripping protocol and path, validating only the domain, then rebuilding a normalized `https://domain` URL.
- **Key expressions or variables used:**  
  - `const body = $input.first().json.body;`
  - `const raw = body?.url?.trim();`
  - Domain extraction:
    - remove protocol with `^https?://`
    - remove path with `/.*$`
  - Validation regex:
    - `/^[a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?(\.[a-zA-Z]{2,})+$/`
- **Input and output connections:**  
  - Input from: `Receive URL`
  - Outputs to:
    - `Scrape page with Firecrawl`
    - `Return URL validation error`
- **Version-specific requirements:**  
  Uses code node version `2`.
- **Edge cases or potential failure types:**  
  - Missing `url` returns a JSON object with `status: 422`, but does not throw.
  - Invalid domains throw an error.
  - The regex accepts domains but does not validate full URLs with paths, query strings, IP addresses, localhost, or uncommon TLD patterns.
  - Any submitted path is discarded intentionally.
  - Because both downstream nodes are connected from the main output, response behavior depends on execution semantics and error routing; this deserves testing after import.
- **Sub-workflow reference:**  
  None

#### Return URL validation error
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends an HTTP 422 response back to the ingestion caller.
- **Configuration choices:**  
  - Response code: `422`
  - Response key set from `={{ $json.error }}`
- **Key expressions or variables used:**  
  - `{{$json.error}}`
- **Input and output connections:**  
  - Input from: `Validate and normalize URL`
  - No output
- **Version-specific requirements:**  
  Uses Respond to Webhook version `1.5`.
- **Edge cases or potential failure types:**  
  - The code node returns `message` for missing URL, but this response node reads `error`; this mismatch may produce an empty or incorrect response body.
  - If the code node succeeds, this node may still be reachable depending on flow behavior and should be tested.
  - If the webhook execution has already been answered elsewhere, n8n may reject a second response.
- **Sub-workflow reference:**  
  None

---

## 2.2 Web Scraping and Vector Ingestion

**Overview:**  
This block takes a normalized URL, scrapes the page with Firecrawl, prepares the scraped content as documents with metadata, generates embeddings through OpenAI, and inserts everything into Pinecone.

**Nodes Involved:**  
- Scrape page with Firecrawl
- Generate OpenAI embeddings
- Load scraped content
- Store embeddings in Pinecone

### Node Details

#### Scrape page with Firecrawl
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawl`  
  External scraping connector that fetches the target page and converts it into clean content.
- **Configuration choices:**  
  - Operation: `scrape`
  - URL sourced from the validation node
  - Scrape options request output in the default format configuration, which in this context is intended to produce markdown.
- **Key expressions or variables used:**  
  - `={{ $('Validate and normalize URL').item.json.url }}`
- **Input and output connections:**  
  - Input from: `Validate and normalize URL`
  - Output to: `Store embeddings in Pinecone`
- **Version-specific requirements:**  
  Uses Firecrawl node version `1`.
- **Edge cases or potential failure types:**  
  - Firecrawl authentication failure
  - Unreachable site, rate limiting, blocked scraping, or timeout
  - Target page may return empty or partial content
  - Some sites may require JavaScript rendering or anti-bot handling beyond this configuration
- **Sub-workflow reference:**  
  None

#### Generate OpenAI embeddings
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  Embedding model provider used by the Pinecone vector store node during ingestion.
- **Configuration choices:**  
  - No explicit model override is shown, so behavior depends on node defaults or account/node version defaults.
  - Sticky note indicates Pinecone must be configured for `text-embedding-3-small` with 1536 dimensions.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - No main input
  - AI embedding output to: `Store embeddings in Pinecone`
- **Version-specific requirements:**  
  Uses embeddings node version `1.2`.
- **Edge cases or potential failure types:**  
  - OpenAI credential errors
  - Model/dimension mismatch with Pinecone index
  - Token or content-size limitations if documents are very large and chunking is not handled elsewhere
- **Sub-workflow reference:**  
  None

#### Load scraped content
- **Type and technical role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
  Converts the incoming scrape result into LangChain-compatible document objects and enriches them with metadata.
- **Configuration choices:**  
  - Adds metadata field `url`
  - URL metadata references the normalized URL from the validation step
- **Key expressions or variables used:**  
  - `={{ $('Validate and normalize URL').item.json.url }}`
- **Input and output connections:**  
  - No main input shown in classic form; it acts as an AI document provider
  - AI document output to: `Store embeddings in Pinecone`
- **Version-specific requirements:**  
  Uses document loader version `1.1`.
- **Edge cases or potential failure types:**  
  - If Firecrawl output shape is incompatible or empty, document extraction may fail or produce no documents.
  - Metadata references the validated base URL, not necessarily the final redirected URL.
- **Sub-workflow reference:**  
  None

#### Store embeddings in Pinecone
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  Central vector store node that inserts documents and embeddings into Pinecone.
- **Configuration choices:**  
  - Mode: `insert`
  - Target Pinecone index: `firecrawl`
  - Receives:
    - main input from scrape result
    - document input from the loader
    - embedding model from OpenAI embeddings node
- **Key expressions or variables used:**  
  None directly in the node config beyond selected resource list values.
- **Input and output connections:**  
  - Main input from: `Scrape page with Firecrawl`
  - AI document input from: `Load scraped content`
  - AI embedding input from: `Generate OpenAI embeddings`
  - Main output to: `Return ingestion result`
- **Version-specific requirements:**  
  Uses Pinecone vector store node version `1.3`.
- **Edge cases or potential failure types:**  
  - Pinecone auth or index-not-found errors
  - Index dimension mismatch
  - Region/environment mismatch in credentials
  - Duplicate content ingestion if same page is sent repeatedly and no deduplication strategy exists
  - If scrape content is too large and no chunking occurs, insert behavior may be suboptimal
- **Sub-workflow reference:**  
  None

---

## 2.3 Ingestion Response Handling

**Overview:**  
This block sends the HTTP response for successful ingestion. It closes the webhook request lifecycle after Pinecone insertion completes.

**Nodes Involved:**  
- Return ingestion result

### Node Details

#### Return ingestion result
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends a success response to the original ingestion webhook caller.
- **Configuration choices:**  
  - HTTP status code: `200`
  - Response type: JSON
  - Executes once
  - Response body contains a message with the number of items processed
- **Key expressions or variables used:**  
  - `{{$input.all().length}}`
  - Response body text: `"Added {{$input.all().length}} items to Supabase"`
- **Input and output connections:**  
  - Input from: `Store embeddings in Pinecone`
  - No output
- **Version-specific requirements:**  
  Uses Respond to Webhook version `1.5`.
- **Edge cases or potential failure types:**  
  - The message says **Supabase**, but the workflow actually stores data in **Pinecone**. This is a documentation/output bug and should be corrected.
  - If no items are inserted, response may still claim success with a low count.
  - If upstream fails and no error handler exists, the caller may get a generic execution failure instead.
- **Sub-workflow reference:**  
  None

---

## 2.4 Chat-Based Retrieval Interface

**Overview:**  
This block provides an interactive chat entry point. It forwards incoming user messages into a tool-using AI agent connected to the indexed knowledge base.

**Nodes Involved:**  
- Receive chat message
- Answer query from knowledge base

### Node Details

#### Receive chat message
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Entry point for chat conversations.
- **Configuration choices:**  
  - Uses default options
  - Generates a separate webhook/chat endpoint for interactive use
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - No input
  - Main output to: `Answer query from knowledge base`
- **Version-specific requirements:**  
  Uses chat trigger version `1.4`.
- **Edge cases or potential failure types:**  
  - Workflow must be active for production chat endpoint use.
  - If chat payload format differs from what the trigger expects, execution may fail.
- **Sub-workflow reference:**  
  None

#### Answer query from knowledge base
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent that handles user questions and can call the Pinecone retrieval tool.
- **Configuration choices:**  
  - Uses default options
  - Wired to:
    - language model
    - memory
    - retrieval tool
- **Key expressions or variables used:**  
  None explicitly configured.
- **Input and output connections:**  
  - Main input from: `Receive chat message`
  - AI language model input from: `OpenRouter LLM`
  - AI memory input from: `Chat memory`
  - AI tool input from: `Retrieve documents from Pinecone`
- **Version-specific requirements:**  
  Uses agent node version `3.1`.
- **Edge cases or potential failure types:**  
  - If the retrieval tool is unavailable, the agent may answer poorly or fail.
  - LLM auth/model issues from OpenRouter can stop the chain.
  - Poor retrieval quality may occur if the Pinecone index is empty or embeddings are inconsistent with ingestion.
- **Sub-workflow reference:**  
  None

---

## 2.5 LLM, Memory, Retrieval, Embeddings, and Reranking

**Overview:**  
This support block equips the chat agent with retrieval-augmented generation capabilities. It provides the language model, short-term memory, query embedding, vector search, and reranking.

**Nodes Involved:**  
- OpenRouter LLM
- Chat memory
- Retrieve documents from Pinecone
- Generate OpenAI embeddings1
- Rerank results with Cohere

### Node Details

#### OpenRouter LLM
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Chat model used by the agent for final answer generation.
- **Configuration choices:**  
  - Model: `anthropic/claude-sonnet-4.6`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output to: `Answer query from knowledge base`
- **Version-specific requirements:**  
  Uses node version `1`.
- **Edge cases or potential failure types:**  
  - OpenRouter credential issues
  - Model availability changes
  - Provider-side rate limits or latency
- **Sub-workflow reference:**  
  None

#### Chat memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Maintains recent conversation state for the agent.
- **Configuration choices:**  
  - Defaults are used; no explicit memory window parameters are shown.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI memory output to: `Answer query from knowledge base`
- **Version-specific requirements:**  
  Uses node version `1.3`.
- **Edge cases or potential failure types:**  
  - Default memory window size may be too small or too large depending on use case.
  - If session handling is not configured externally, conversation continuity may vary by runtime context.
- **Sub-workflow reference:**  
  None

#### Retrieve documents from Pinecone
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  Exposes Pinecone search as a callable tool for the agent.
- **Configuration choices:**  
  - Mode: `retrieve-as-tool`
  - Pinecone index: `firecrawl`
  - Tool description: `Retrieve data for the AI Agent.`
  - Reranking enabled
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - AI embedding input from: `Generate OpenAI embeddings1`
  - AI reranker input from: `Rerank results with Cohere`
  - AI tool output to: `Answer query from knowledge base`
- **Version-specific requirements:**  
  Uses Pinecone vector store version `1.3`.
- **Edge cases or potential failure types:**  
  - Pinecone auth/index errors
  - Empty index leads to irrelevant or no results
  - Query embedding model must remain compatible with indexed vectors
  - Tool description is minimal; richer guidance can improve agent tool usage behavior
- **Sub-workflow reference:**  
  None

#### Generate OpenAI embeddings1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  Embeds the user query before retrieval from Pinecone.
- **Configuration choices:**  
  - Default options only
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI embedding output to: `Retrieve documents from Pinecone`
- **Version-specific requirements:**  
  Uses embeddings node version `1.2`.
- **Edge cases or potential failure types:**  
  - Must match Pinecone vector dimension expectations
  - Credential and rate-limit failures are possible
- **Sub-workflow reference:**  
  None

#### Rerank results with Cohere
- **Type and technical role:** `@n8n/n8n-nodes-langchain.rerankerCohere`  
  Reranks the retrieved Pinecone results before they are returned to the agent.
- **Configuration choices:**  
  - Default settings
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI reranker output to: `Retrieve documents from Pinecone`
- **Version-specific requirements:**  
  Uses reranker node version `1`.
- **Edge cases or potential failure types:**  
  - Cohere auth or quota errors
  - Added latency on retrieval
  - If result set is very small, reranking may have limited impact
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | ## How it works |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 1. A webhook receives a URL via POST request |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 2. The URL is validated and normalized, returning a 422 error if invalid |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 3. Firecrawl scrapes the page and converts it to clean markdown |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 4. OpenAI generates 1536-dimensional vector embeddings from the content |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 5. The content and embeddings are stored in Pinecone |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | ## Setup steps |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 1. Create a Pinecone index with the settings from the "Pinecone setup" sticky |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 2. Add your Firecrawl API key |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 3. Add your OpenAI API key (for embeddings) |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 4. Add your OpenRouter API key (for the chat agent) |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 5. Add your Cohere API key (for reranking) |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for overall workflow behavior and setup |  |  | 6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | ## How it works |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 1. A webhook receives a URL via POST request |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 2. The URL is validated and normalized, returning a 422 error if invalid |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 3. Firecrawl scrapes the page and converts it to clean markdown |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 4. OpenAI generates 1536-dimensional vector embeddings from the content |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 5. The content and embeddings are stored in Pinecone |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | ## Setup steps |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 1. Create a Pinecone index with the settings from the "Pinecone setup" sticky |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 2. Add your Firecrawl API key |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 3. Add your OpenAI API key (for embeddings) |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 4. Add your OpenRouter API key (for the chat agent) |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 5. Add your Cohere API key (for reranking) |
| Receive URL | n8n-nodes-base.webhook | Receives POST ingestion requests |  | Validate and normalize URL | 6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | ## How it works |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 1. A webhook receives a URL via POST request |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 2. The URL is validated and normalized, returning a 422 error if invalid |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 3. Firecrawl scrapes the page and converts it to clean markdown |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 4. OpenAI generates 1536-dimensional vector embeddings from the content |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 5. The content and embeddings are stored in Pinecone |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | ## Setup steps |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 1. Create a Pinecone index with the settings from the "Pinecone setup" sticky |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 2. Add your Firecrawl API key |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 3. Add your OpenAI API key (for embeddings) |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 4. Add your OpenRouter API key (for the chat agent) |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 5. Add your Cohere API key (for reranking) |
| Validate and normalize URL | n8n-nodes-base.code | Validates request payload and normalizes domain into https URL | Receive URL | Scrape page with Firecrawl; Return URL validation error | 6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | ## How it works |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 1. A webhook receives a URL via POST request |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 2. The URL is validated and normalized, returning a 422 error if invalid |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 3. Firecrawl scrapes the page and converts it to clean markdown |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 4. OpenAI generates 1536-dimensional vector embeddings from the content |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 5. The content and embeddings are stored in Pinecone |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | ## Setup steps |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 1. Create a Pinecone index with the settings from the "Pinecone setup" sticky |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 2. Add your Firecrawl API key |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 3. Add your OpenAI API key (for embeddings) |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 4. Add your OpenRouter API key (for the chat agent) |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 5. Add your Cohere API key (for reranking) |
| Scrape page with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes and converts page content for ingestion | Validate and normalize URL | Store embeddings in Pinecone | 6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Return URL validation error | n8n-nodes-base.respondToWebhook | Returns 422 response for invalid input | Validate and normalize URL |  |  |
| Store embeddings in Pinecone | @n8n/n8n-nodes-langchain.vectorStorePinecone | Inserts documents and vectors into Pinecone | Scrape page with Firecrawl; Load scraped content; Generate OpenAI embeddings | Return ingestion result |  |
| Return ingestion result | n8n-nodes-base.respondToWebhook | Returns success response after ingestion | Store embeddings in Pinecone |  |  |
| Generate OpenAI embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Provides embedding model for ingestion |  | Store embeddings in Pinecone |  |
| Load scraped content | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Converts scrape output into documents with metadata |  | Store embeddings in Pinecone |  |
| Receive chat message | @n8n/n8n-nodes-langchain.chatTrigger | Receives user chat requests for RAG querying |  | Answer query from knowledge base |  |
| Answer query from knowledge base | @n8n/n8n-nodes-langchain.agent | Runs the chat agent with tool access | Receive chat message; OpenRouter LLM; Chat memory; Retrieve documents from Pinecone |  |  |
| OpenRouter LLM | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Chat model used by the RAG agent |  | Answer query from knowledge base |  |
| Chat memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains short conversation history |  | Answer query from knowledge base |  |
| Retrieve documents from Pinecone | @n8n/n8n-nodes-langchain.vectorStorePinecone | Retrieval tool for knowledge base search | Generate OpenAI embeddings1; Rerank results with Cohere | Answer query from knowledge base |  |
| Generate OpenAI embeddings1 | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embeds user queries for retrieval |  | Retrieve documents from Pinecone |  |
| Rerank results with Cohere | @n8n/n8n-nodes-langchain.rerankerCohere | Reranks retrieved documents |  | Retrieve documents from Pinecone |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | ## Pinecone setup |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | Your Pinecone index must use 1536 dimensions to match the `text-embedding-3-small` OpenAI model. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | 1. Go to your Pinecone console and open your index settings |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | 2. Select text-embedding-3-small as the embedding model |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | 3. Confirm these settings: |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | Setting \| Value |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | Modality \| Text |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | Vector type \| Dense |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | Dimension \| 1536 |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Pinecone configuration |  |  | Metric \| cosine |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Scrape and ingest web content into Pinecone with Firecrawl`.

2. **Add a Webhook node** named `Receive URL`.
   - Node type: `Webhook`
   - HTTP method: `POST`
   - Response mode: `Using Respond to Webhook node`
   - Leave options default unless you need authentication or custom response settings.
   - This node will be the ingestion entry point.

3. **Add a Code node** named `Validate and normalize URL`.
   - Connect `Receive URL` → `Validate and normalize URL`.
   - Set error handling to continue so that invalid input can be routed to a response node.
   - Paste logic equivalent to:
     - read `body.url`
     - trim the string
     - if missing, return status `422` and an explanatory message
     - strip protocol and path
     - validate domain format
     - if invalid, throw an error
     - if valid, return:
       - `status: 200`
       - `domain`
       - `url: https://<domain>`

4. **Use this JavaScript behavior in the Code node:**
   - Read from: first input item, request body
   - Normalize:
     - `https://example.com/path?q=1` → `https://example.com`
     - `firecrawl.dev` → `https://firecrawl.dev`
   - Domain regex should only accept standard hostnames.

5. **Add a Respond to Webhook node** named `Return URL validation error`.
   - Connect it from `Validate and normalize URL`.
   - Response code: `422`
   - Configure it to return an error field or message field from the incoming JSON.
   - Recommended improvement: return `{{$json.message || $json.error || 'Invalid URL'}}`
   - This avoids the mismatch present in the source workflow.

6. **Add the Firecrawl node** named `Scrape page with Firecrawl`.
   - Connect `Validate and normalize URL` → `Scrape page with Firecrawl`.
   - Node type: Firecrawl
   - Operation: `scrape`
   - URL field expression:
     - `{{$('Validate and normalize URL').item.json.url}}`
   - Keep scrape format set to markdown/default text output.
   - Create and attach a **Firecrawl API credential**.

7. **Create Firecrawl credentials.**
   - In n8n credentials, add your Firecrawl API key.
   - Assign it to the Firecrawl node.
   - Test against a known public page.

8. **Add an OpenAI Embeddings node** named `Generate OpenAI embeddings`.
   - This node will provide embeddings for ingestion into Pinecone.
   - Use OpenAI credentials.
   - If the model can be chosen explicitly, set it to `text-embedding-3-small` to match the stated Pinecone dimension requirement.

9. **Create OpenAI credentials.**
   - Add an OpenAI API credential in n8n.
   - Attach it to both embeddings nodes used in this workflow.

10. **Add a Document Default Data Loader node** named `Load scraped content`.
    - Configure metadata to include:
      - `url` = `{{$('Validate and normalize URL').item.json.url}}`
    - This node converts scrape content into documents suitable for vector storage.

11. **Add a Pinecone Vector Store node** named `Store embeddings in Pinecone`.
    - Mode: `insert`
    - Select your Pinecone index: `firecrawl` or another equivalent index
    - Connect:
      - main input from `Scrape page with Firecrawl`
      - AI embedding input from `Generate OpenAI embeddings`
      - AI document input from `Load scraped content`

12. **Create Pinecone credentials.**
    - Add a Pinecone API credential in n8n.
    - Ensure region/environment matches your Pinecone project.
    - Attach credentials to the Pinecone node.

13. **Prepare the Pinecone index before running ingestion.**
    - Create an index named `firecrawl` or update the workflow to your preferred index name.
    - Required characteristics:
      - Modality: Text
      - Vector type: Dense
      - Dimension: 1536
      - Metric: cosine
    - This is required if using `text-embedding-3-small`.

14. **Add a Respond to Webhook node** named `Return ingestion result`.
    - Connect `Store embeddings in Pinecone` → `Return ingestion result`.
    - Response code: `200`
    - Respond with JSON.
    - Recommended response body:
      - `{"message":"Added {{$input.all().length}} items to Pinecone"}`
    - This corrects the inaccurate “Supabase” message from the source workflow.

15. **Add a Chat Trigger node** named `Receive chat message`.
    - This creates the chat entry point for querying the knowledge base.
    - Keep default settings unless you need session customization.

16. **Add an AI Agent node** named `Answer query from knowledge base`.
    - Connect `Receive chat message` → `Answer query from knowledge base`.
    - Leave default options unless you want a custom system prompt.
    - This node will orchestrate retrieval and answer generation.

17. **Add an OpenRouter Chat Model node** named `OpenRouter LLM`.
    - Connect its AI language model output to `Answer query from knowledge base`.
    - Set model to:
      - `anthropic/claude-sonnet-4.6`
    - Create and attach an OpenRouter API credential.

18. **Create OpenRouter credentials.**
    - Add your OpenRouter API key in n8n.
    - Attach it to the `OpenRouter LLM` node.

19. **Add a Buffer Window Memory node** named `Chat memory`.
    - Connect its AI memory output to `Answer query from knowledge base`.
    - Defaults are acceptable initially.
    - Optionally tune the memory window later based on chat length.

20. **Add a second OpenAI Embeddings node** named `Generate OpenAI embeddings1`.
    - This node is used for retrieval query embeddings.
    - Attach the same OpenAI credentials.
    - Prefer the same embedding model family as ingestion for consistency.

21. **Add a Cohere Reranker node** named `Rerank results with Cohere`.
    - Create and attach a Cohere API credential.
    - Keep default settings unless you need a specific reranking model or custom parameters.

22. **Create Cohere credentials.**
    - Add a Cohere API key in n8n.
    - Attach it to the reranker node.

23. **Add a second Pinecone Vector Store node** named `Retrieve documents from Pinecone`.
    - Mode: `retrieve as tool`
    - Select the same Pinecone index used for ingestion
    - Enable reranking
    - Tool description:
      - `Retrieve data for the AI Agent.`
    - Connect:
      - AI embedding input from `Generate OpenAI embeddings1`
      - AI reranker input from `Rerank results with Cohere`
      - AI tool output to `Answer query from knowledge base`

24. **Confirm all connections.**
    - Ingestion branch:
      - `Receive URL` → `Validate and normalize URL`
      - `Validate and normalize URL` → `Scrape page with Firecrawl`
      - `Validate and normalize URL` → `Return URL validation error`
      - `Scrape page with Firecrawl` → `Store embeddings in Pinecone`
      - `Generate OpenAI embeddings` → `Store embeddings in Pinecone` (AI embedding)
      - `Load scraped content` → `Store embeddings in Pinecone` (AI document)
      - `Store embeddings in Pinecone` → `Return ingestion result`
    - Chat branch:
      - `Receive chat message` → `Answer query from knowledge base`
      - `OpenRouter LLM` → `Answer query from knowledge base` (AI language model)
      - `Chat memory` → `Answer query from knowledge base` (AI memory)
      - `Generate OpenAI embeddings1` → `Retrieve documents from Pinecone` (AI embedding)
      - `Rerank results with Cohere` → `Retrieve documents from Pinecone` (AI reranker)
      - `Retrieve documents from Pinecone` → `Answer query from knowledge base` (AI tool)

25. **Optionally add sticky notes** for operational clarity.
   Include:
   - high-level workflow steps
   - required credentials
   - Pinecone index settings

26. **Test the ingestion webhook in manual mode.**
   - Example POST body:
     ```json
     {
       "url": "firecrawl.dev"
     }
     ```
   - Also test:
     ```json
     {
       "url": "https://example.com/docs"
     }
     ```
   - Confirm that the code normalizes these into a base HTTPS domain.

27. **Validate error handling.**
   - Send an empty body or invalid domain.
   - Confirm the caller receives HTTP 422.
   - If not, adjust the code node’s error path or the response node field mapping.

28. **Test Pinecone insertion.**
   - After a successful call, verify vectors/documents exist in the selected Pinecone index.
   - Check metadata contains the `url` field.

29. **Test the chat branch.**
   - Use the chat trigger interface or endpoint.
   - Ask a question related to the scraped content.
   - Confirm the agent retrieves relevant content and produces an answer.

30. **Activate the workflow** once both branches work in test mode.

### Expected Inputs and Outputs

#### Ingestion webhook input
- Method: `POST`
- JSON body:
  ```json
  {
    "url": "https://example.com"
  }
  ```

#### Ingestion success output
- HTTP 200
- JSON message confirming processed item count

#### Ingestion error output
- HTTP 422
- JSON error/message indicating missing or invalid URL

#### Chat input
- User message through the Chat Trigger interface

#### Chat output
- Natural-language answer grounded in retrieved Pinecone documents

### Recommended Improvements While Rebuilding
- Fix `Return URL validation error` to use `message` as well as `error`.
- Fix success response text from “Supabase” to “Pinecone”.
- Add chunking before embeddings if you expect long pages.
- Add deduplication using URL hash or canonical URL metadata.
- Add explicit model selection for both embedding nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Overall workflow note: A webhook receives a URL, validates it, scrapes content with Firecrawl, creates OpenAI embeddings, stores them in Pinecone, and exposes a RAG chat agent with Cohere reranking. | Workflow purpose |
| Setup note: Required credentials are Firecrawl, OpenAI, OpenRouter, Cohere, and Pinecone. | Environment setup |
| Pinecone requirement: index must use 1536 dimensions to match `text-embedding-3-small`. | Pinecone configuration |
| Pinecone settings: Modality = Text, Vector type = Dense, Dimension = 1536, Metric = cosine. | Pinecone configuration |
| Example ingestion call: `{"url": "https://example.com"}` | Webhook test payload |

### Additional Implementation Notes
- The workflow contains **two entry points**:
  1. `Receive URL` for ingestion
  2. `Receive chat message` for querying
- There are **no sub-workflow invocation nodes** in this workflow.
- The pinned sample input on the webhook node shows a real test case using:
  - `firecrawl.dev`
- The workflow is currently **inactive** in the provided JSON, so production endpoints would not respond until activated.
- The ingestion branch and chat branch both rely on the **same Pinecone index**, which is essential for the RAG loop to work correctly.