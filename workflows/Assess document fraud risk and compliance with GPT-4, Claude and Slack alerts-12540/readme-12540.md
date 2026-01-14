Assess document fraud risk and compliance with GPT-4, Claude and Slack alerts

https://n8nworkflows.xyz/workflows/assess-document-fraud-risk-and-compliance-with-gpt-4--claude-and-slack-alerts-12540


# Assess document fraud risk and compliance with GPT-4, Claude and Slack alerts

## 1. Workflow Overview

**Workflow name (JSON):** *Autonomous AI-Driven Document Trust, Fraud Detection and Compliance Engine*  
**User-provided title:** *Assess document fraud risk and compliance with GPT-4, Claude and Slack alerts*  

**Purpose:**  
This workflow ingests an uploaded document via webhook, extracts its text (PDF / image OCR placeholder / Office extraction), runs multiple parallel AI analyses (metadata extraction, forgery indicators, semantic contradictions, historical pattern comparison via a vector store, and entity enrichment via a tool), computes an aggregated risk score, routes the case by risk level, stores results in Postgres, alerts teams in Slack for elevated risk, logs an audit trail, signs the result cryptographically, and returns a JSON response to the webhook caller.

**Primary use cases:**
- Fraud screening of inbound documents (invoices, contracts, IDs, certificates, etc.)
- Compliance pre-checks and escalation workflows
- Creating an auditable, signed decision record per document

### 1.1 Input Reception & Configuration
- Receive document over HTTP POST
- Set thresholds and shared configuration for the run
- Generate a unique SHA-256 hash for traceability/deduplication

### 1.2 Document Type Routing & Text Extraction
- Route by MIME type to PDF extraction, image OCR (placeholder), or Office extraction
- Merge extracted text into a single stream for downstream analysis

### 1.3 Parallel AI Analysis Pipelines (LangChain nodes)
- Metadata extraction (structured)
- Forgery indicator analysis (structured)
- Semantic contradiction detection (structured)
- Historical comparison using an in-memory vector store + embeddings + text splitting (structured)

### 1.4 Aggregation, Entity Enrichment, and Risk Scoring
- Merge the four AI analysis outputs
- Run entity enrichment agent (calls a custom code tool) and parse structured output
- Compute a weighted risk score + confidence + explainable risk factors

### 1.5 Risk Routing, Storage, Alerts, Audit, Proof, and Response
- Switch on risk thresholds (low/medium/high/critical)
- Store into Postgres tables
- Slack alerts for medium (human review) and critical (security alert)
- Wait-for-review step for medium/high handling
- Log audit trail
- Generate cryptographic signature
- Return final JSON response to webhook caller

---

## 2. Block-by-Block Analysis

### Block 2.1 — Ingestion, Run Configuration, and Hashing
**Overview:** Receives the document, sets global thresholds/config values, then generates a SHA-256 hash from the binary payload for tracking and storage.  
**Nodes involved:** Document Ingestion Webhook → Workflow Configuration → Generate Document Hash

#### Node: Document Ingestion Webhook
- **Type/role:** `Webhook` (entry point). Receives uploaded document via HTTP.
- **Key configuration:**
  - **Method:** POST
  - **Path:** `document-ingestion`
  - **Binary property:** expects file data in binary property named `data` (via `options.binaryPropertyName`)
  - **Response mode:** `lastNode` (response is produced by the last executed response node path; here “Return Processing Result”)
- **Inputs/outputs:** Entry node → outputs to **Workflow Configuration**
- **Failure/edge cases:**
  - Missing binary data property `data` will break downstream extraction/hash steps.
  - Large uploads may hit n8n body size limits or reverse proxy limits.
  - If caller doesn’t set/forward `mimeType` consistently, routing may fail (Switch fallback).

#### Node: Workflow Configuration
- **Type/role:** `Set` node. Defines workflow constants and keeps other incoming fields.
- **Key configuration choices (interpreted):**
  - Sets:
    - `riskThresholdLow = 30`
    - `riskThresholdHigh = 70`
    - `riskThresholdCritical = 90`
    - `maxWaitTimeHours = 48` (not currently used to configure the Wait node)
    - `vectorStoreMemoryId = "fraud_detection_docs"`
  - **Include other fields:** enabled, so original webhook JSON/binary remains available.
- **Inputs/outputs:** from Webhook → to **Generate Document Hash**
- **Failure/edge cases:** None typical; but missing or renamed config keys will break expressions later (threshold comparisons, vector store key).

#### Node: Generate Document Hash
- **Type/role:** `Crypto` node. Produces a SHA256 hash of the **binary** document.
- **Key configuration choices:**
  - Hash type: `SHA256`
  - `binaryData: true` (hash is derived from binary content)
  - Output property: `documentHash`
- **Inputs/outputs:** from Workflow Configuration → to **Route by Document Type**
- **Failure/edge cases:**
  - If binary is missing/empty, hashing may fail or produce unexpected output.
  - If binary property name isn’t what the node expects internally, confirm how the Crypto node selects binary (workflow sets webhook binary property name to `data`, but the crypto node is configured generically—test with your payload structure).

---

### Block 2.2 — Document Type Routing & Text Extraction
**Overview:** Routes documents by MIME type, extracts text via the appropriate method, then merges extracted text into one stream for analysis.  
**Nodes involved:** Route by Document Type → (Extract Text from PDF | OCR Processing for Images | Extract from Office Documents) → Merge Extracted Content

#### Node: Route by Document Type
- **Type/role:** `Switch` node. Branches based on `$json.mimeType`.
- **Key configuration choices:**
  - **PDF output:** if `$json.mimeType` contains `pdf`
  - **Image output:** if contains `image`
  - **Office output:** if contains `officedocument` OR `msword` OR `ms-excel`
  - **Fallback output:** `extra` (renamed to “extra” and used if no rule matches)
  - `ignoreCase: true`
- **Inputs/outputs:** from Generate Document Hash → to extraction nodes
- **Failure/edge cases:**
  - If `mimeType` is missing or non-standard, document goes to fallback (not connected), effectively dropping the execution.
  - Some Office MIME types (pptx, older xls) may not match; extend rules if needed.

#### Node: Extract Text from PDF
- **Type/role:** `Extract From File` node (PDF operation).
- **Key configuration:**
  - Operation: `pdf`
  - `joinPages: true` (returns one combined text)
- **Inputs/outputs:** from Switch “PDF” → to Merge Extracted Content (input 1)
- **Failure/edge cases:**
  - Scanned PDFs without embedded text will yield little/no text (should be routed to OCR in a more advanced design).
  - Password-protected PDFs may fail.

#### Node: OCR Processing for Images
- **Type/role:** `Code` node. Placeholder OCR extraction that simulates OCR output.
- **Key configuration/logic:**
  - Reads the first available binary property and returns:
    - `json.text` containing placeholder OCR text
    - metadata fields like `fileName`, `mimeType`, `fileSize`, `ocrProcessed`, `ocrTimestamp`
  - Uses placeholder endpoint/key strings:
    - `<__PLACEHOLDER_VALUE__OCR_API_ENDPOINT__>`
    - `<__PLACEHOLDER_VALUE__OCR_API_KEY__>`
- **Inputs/outputs:** from Switch “Image” → to Merge Extracted Content (input 2)
- **Failure/edge cases:**
  - Throws if no binary data exists.
  - Not a real OCR integration: in production you must replace with HTTP request to OCR provider and parse response.

#### Node: Extract from Office Documents
- **Type/role:** `Extract From File` node configured for `xlsx`.
- **Key configuration:**
  - Operation: `xlsx` (note: despite “Office” routing, it’s specifically configured for Excel)
- **Inputs/outputs:** from Switch “Office” → to Merge Extracted Content (input 3)
- **Failure/edge cases:**
  - Word/docx won’t be extracted correctly with `xlsx` operation. If you truly need doc/docx/pptx, add proper extraction operations or separate routing per subtype.

#### Node: Merge Extracted Content
- **Type/role:** `Merge` node combining text extraction outputs.
- **Key configuration:**
  - Mode: `combine`
  - Combine by: `position`
  - Number of inputs: 3
- **Inputs/outputs:** receives up to 3 extraction branches → outputs to **four** downstream branches in parallel:
  - Metadata Extraction Agent
  - Forgery Detection Agent
  - Semantic Contradiction Detector
  - Historical Document Vector Store
- **Failure/edge cases:**
  - If only one branch produces output, combine-by-position can yield missing items for other positions depending on how many items each branch outputs. Ensure each extraction path returns exactly one item for consistent merging.
  - Fallback (“extra”) from Switch is not connected; those documents never reach this merge.

---

### Block 2.3 — AI: Metadata Extraction (Structured)
**Overview:** Uses an OpenAI chat model to extract structured metadata fields from the extracted document text.  
**Nodes involved:** Metadata Extraction Agent + OpenAI Model - Metadata + Metadata Schema Parser

#### Node: Metadata Extraction Agent
- **Type/role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`). Orchestrates LLM call and structured parsing.
- **Key configuration:**
  - Input text: `{{ $json.text }}`
  - System message defines required fields (documentType, issuer, dates, entities, amounts, signatures, watermarks, etc.)
  - `hasOutputParser: true` (expects structured output via connected parser)
- **Connections:**
  - **Language model:** OpenAI Model - Metadata (ai_languageModel)
  - **Output parser:** Metadata Schema Parser (ai_outputParser)
  - **Main output:** to Merge Analysis Results (input 1)
- **Failure/edge cases:**
  - If `text` is empty, the model may hallucinate; enforce minimum length checks upstream.
  - Parser will fail if model returns invalid JSON for the schema.

#### Node: OpenAI Model - Metadata
- **Type/role:** OpenAI chat model connector (`lmChatOpenAi`).
- **Key configuration:**
  - Model: `gpt-4.1-mini`
- **Credentials:** `OpenAi account` required.
- **Failure/edge cases:** API quota, model access rights, latency/timeouts.

#### Node: Metadata Schema Parser
- **Type/role:** Structured output parser (`outputParserStructured`) with manual JSON schema.
- **Key configuration:**
  - Schema includes fields like `documentType`, `issuer`, `issueDate`, `expiryDate`, `entities[]`, etc.
  - Note: `amounts` is defined as an array of **numbers** only; if you expect currency strings, adjust schema accordingly.
- **Failure/edge cases:**
  - Strict schema can reject reasonable outputs (e.g., dates not in ISO format).
  - Consider adding required fields or loosening types for robustness.

---

### Block 2.4 — AI: Forgery Indicator Analysis (Structured)
**Overview:** Detects forgery/manipulation indicators from the text and returns an overall forgery risk score.  
**Nodes involved:** Forgery Detection Agent + OpenAI Model - Forgery + Forgery Analysis Schema

#### Node: Forgery Detection Agent
- **Type/role:** LangChain Agent for forensic analysis.
- **Key configuration:**
  - Input: `{{ $json.text }}`
  - System message enumerates forgery indicators and required structured fields.
  - `hasOutputParser: true`
- **Connections:**
  - Model: OpenAI Model - Forgery
  - Parser: Forgery Analysis Schema
  - Main output: Merge Analysis Results (input 2)
- **Failure/edge cases:**
  - Text-only analysis can’t detect visual tampering/fonts/watermarks reliably; results are heuristic.

#### Node: OpenAI Model - Forgery
- **Type/role:** OpenAI chat model connector.
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI credential required.

#### Node: Forgery Analysis Schema
- **Type/role:** Structured parser for forgery output.
- **Key config:** Requires `overallRiskScore`, `indicators[]`, `recommendation`.
- **Failure/edge cases:** Parser failures if LLM misses required fields.

---

### Block 2.5 — AI: Semantic Contradiction Detection (Structured)
**Overview:** Searches for internal contradictions (dates, numbers, entities, logical conflicts) and assigns a contradiction risk score.  
**Nodes involved:** Semantic Contradiction Detector + OpenAI Model - Semantic + Contradiction Schema Parser

#### Node: Semantic Contradiction Detector
- **Type/role:** LangChain Agent for semantic consistency checks.
- **Key configuration:**
  - Input: `{{ $json.text }}`
  - `hasOutputParser: true`
- **Connections:** Model + parser; main output to Merge Analysis Results (input 3)
- **Failure/edge cases:**
  - Requires sufficient context in extracted text; poor OCR/extraction causes false positives/negatives.

#### Node: OpenAI Model - Semantic
- **Type/role:** OpenAI chat connector (`gpt-4.1-mini`)

#### Node: Contradiction Schema Parser
- **Type/role:** Structured output parser
- **Schema:** Requires `overallRiskScore`, `contradictions[]`, `hasLogicalIssues`.

---

### Block 2.6 — Vector Store Insert + Historical Pattern Comparison (Structured)
**Overview:** Inserts the current document into an in-memory vector store (embedding + splitting), then uses an agent to compare the current document against stored historical patterns and outputs a historical risk score.  
**Nodes involved:** OpenAI Embeddings + Document Loader + Text Splitter + Historical Document Vector Store + Historical Pattern Comparison Agent + OpenAI Model - Pattern + Pattern Analysis Schema

#### Node: OpenAI Embeddings
- **Type/role:** Embedding generator (`embeddingsOpenAi`) for vectorization.
- **Credentials:** OpenAI credential required.
- **Connection:** feeds `ai_embedding` into Historical Document Vector Store.
- **Failure/edge cases:** Embedding model defaults may change; pin explicit model if you need determinism.

#### Node: Document Loader
- **Type/role:** LangChain document loader (`documentDefaultDataLoader`) to transform text into documents for embedding.
- **Key configuration:**
  - `jsonData: {{ $json.text }}` in expression mode
  - `textSplittingMode: custom` (expects external splitter connected)
- **Connections:**
  - Receives `ai_textSplitter` from Text Splitter
  - Outputs `ai_document` to Historical Document Vector Store
- **Failure/edge cases:** If `$json.text` is not a string, document creation fails.

#### Node: Text Splitter
- **Type/role:** Recursive character splitter.
- **Key configuration:** `chunkOverlap: 200` (chunk size not specified in node params—uses node default)
- **Connection:** provides splitter to Document Loader via `ai_textSplitter`.

#### Node: Historical Document Vector Store
- **Type/role:** In-memory vector store (`vectorStoreInMemory`) in **insert** mode.
- **Key configuration:**
  - `memoryKey` is dynamic: `{{ $('Workflow Configuration').item.json.vectorStoreMemoryId }}`
  - Mode: `insert` (stores the current document)
- **Main output:** goes to Historical Pattern Comparison Agent.
- **Failure/edge cases:**
  - In-memory store is ephemeral (lost on restart) and instance-scoped; not suitable for durable historical fraud memory.
  - Insert-only mode here suggests pattern comparison relies on the same memory key; but no explicit retrieval/search tool is wired into the pattern agent, so “comparison” may not actually query vectors unless the agent internally uses available tools (it doesn’t appear to have a retriever tool attached). Consider adding a vector store retriever tool for real similarity search.

#### Node: Historical Pattern Comparison Agent
- **Type/role:** LangChain Agent for pattern comparison.
- **Key configuration:** input `{{ $json.text }}`
- **Connections:** Model (OpenAI Model - Pattern) and parser (Pattern Analysis Schema)
- **Main output:** Merge Analysis Results (input 4)
- **Failure/edge cases:** Without an attached retrieval tool, the agent may produce generic “pattern” results.

#### Node: OpenAI Model - Pattern
- **Type/role:** OpenAI chat connector (`gpt-4.1-mini`)

#### Node: Pattern Analysis Schema
- **Type/role:** Structured parser for pattern output.
- **Schema:** `overallRiskScore`, `patterns[]`, `isKnownPattern`.

---

### Block 2.7 — Merge AI Results → Entity Enrichment (Tool + Agent)
**Overview:** Aggregates the four analysis outputs, then runs an enrichment agent that can call a custom tool to enrich entities and flag compliance issues.  
**Nodes involved:** Merge Analysis Results → Entity Enrichment Agent (+ OpenAI Model - Enrichment + Entity Enrichment Tool + Enrichment Schema Parser)

#### Node: Merge Analysis Results
- **Type/role:** `Merge` node combining 4 parallel analysis outputs.
- **Key configuration:**
  - Mode: `combine`
  - Combine by: `position`
  - Number of inputs: 4
- **Inputs:** from Metadata/Forgery/Semantic/Pattern agents.
- **Output:** to Entity Enrichment Agent.
- **Edge cases:**
  - If any upstream agent fails or outputs multiple items, position-based merging can misalign results.

#### Node: Entity Enrichment Agent
- **Type/role:** LangChain Agent that enriches entities using a tool.
- **Key configuration:**
  - Text input: `{{ JSON.stringify($json) }}` (passes merged analysis object as a string)
  - System message instructs tool usage and output structure.
  - `hasOutputParser: true`
- **Connections:**
  - Model: OpenAI Model - Enrichment
  - Tool: Entity Enrichment Tool
  - Parser: Enrichment Schema Parser
  - Main output: Calculate Risk Score
- **Edge cases:**
  - The merged input may not include a clean `entities` list unless earlier nodes place it consistently.
  - Tool call uses `$fromAI('query', ...)` and returns a formatted string; the agent must transform that string into structured schema—often fragile.

#### Node: Entity Enrichment Tool
- **Type/role:** LangChain code tool (`toolCode`) available to the enrichment agent.
- **Key behavior:**
  - Reads `query` from the agent: `const query = $fromAI('query', ...)`
  - Simulates enrichment data and returns a formatted text report.
  - Placeholders for internal/external APIs:
    - `<__PLACEHOLDER_VALUE__INTERNAL_DB_API_ENDPOINT__>`, etc.
- **Edge cases/failures:**
  - Returns error string if empty query; agent may still try to parse it.
  - Must be replaced with real HTTP calls for production.

#### Node: OpenAI Model - Enrichment
- **Type/role:** OpenAI chat connector (`gpt-4.1-mini`)

#### Node: Enrichment Schema Parser
- **Type/role:** Structured parser enforcing:
  - `enrichedEntities[]` and `overallEnrichmentScore`
- **Edge cases:** schema requires booleans/strings; tool returns freeform text, so model must map it correctly.

---

### Block 2.8 — Risk Scoring and Risk Routing
**Overview:** Computes weighted risk score, confidence interval, explainable risk factors, then routes to low/medium/high/critical outcomes.  
**Nodes involved:** Calculate Risk Score → Route by Risk Level

#### Node: Calculate Risk Score
- **Type/role:** `Code` node computing a multi-factor risk score.
- **Key logic highlights:**
  - Reads:
    - `metadataAnalysis`, `forgeryAnalysis`, `semanticAnalysis`, `patternAnalysis`, `enrichmentAnalysis`
  - Weights:
    - forgery 0.35, semantic 0.25, pattern 0.20, enrichment 0.15, metadata 0.05
  - Builds `riskFactors` list from high/critical indicators and patterns.
  - Produces:
    - `riskScore`, `riskLevel`, `confidence`, `confidenceInterval`, `recommendation`, `summary`, `componentScores`, and preserves original analyses.
- **Critical integration issue (important):**
  - The preceding merge/enrichment nodes do **not** explicitly map outputs into `metadataAnalysis`, `forgeryAnalysis`, etc. Unless the agent nodes output JSON with these exact keys, this code will default to `{}` and compute near-zero risk.
  - To make it work reliably, add a mapping step (Set node) after “Merge Analysis Results” (or after each agent) to place results into the expected keys.
- **Edge cases:**
  - Assumes `averageConfidence` exists; most parsers don’t define it. Defaults to 70.
  - Uses `metadataAnalysis.riskScore`, but metadata schema does not include `riskScore`.

#### Node: Route by Risk Level
- **Type/role:** `Switch` node routing by numeric thresholds from configuration.
- **Rules:**
  - Low Risk: `riskScore < riskThresholdLow` (30)
  - Medium Risk: `riskScore < riskThresholdHigh` (70)
  - High Risk: `riskScore < riskThresholdCritical` (90)
  - Fallback renamed: **Critical Risk**
- **Outputs:**
  - Low Risk → Store Low Risk Documents
  - Medium Risk → Notify Compliance Team
  - High Risk → Alert Security Team (note: despite label “High Risk”, it routes to security alert path; see below)
  - Critical Risk (fallback) → also goes to Slack alert path because only 3 outputs are connected; the fallback output is connected as “extra” in connections? In JSON, only three outputs are listed; fallback is labeled “Critical Risk” but not separately connected. This typically means “Critical Risk” uses the Switch’s fallback output index—verify in UI.
- **Edge cases:**
  - Ensure thresholds are consistent (low < high < critical).
  - Strict number validation enabled; if `riskScore` is a string, comparisons may fail.

---

### Block 2.9 — Storage, Human Review, Alerts, Audit Trail, Crypto Proof, Webhook Response
**Overview:** Stores outcomes in Postgres, triggers Slack notifications for review/escalation, optionally waits for human input, logs audit trail, signs the record, and returns the response.  
**Nodes involved:**  
- Low: Store Low Risk Documents → Log Audit Trail → Generate Cryptographic Proof → Prepare Response Data → Return Processing Result  
- Medium: Notify Compliance Team → Wait for Human Review → Store High Risk Documents → Log Audit Trail → … → Return  
- Critical: Alert Security Team → Store Critical Risk Documents → Log Audit Trail → … → Return

#### Node: Store Low Risk Documents
- **Type/role:** `Postgres` insert/upsert (depending on defaults; configured via column mapping).
- **Key configuration:**
  - Table: `<__PLACEHOLDER_VALUE__Low_Risk_Documents_Table__>` in `public`
  - Stores: `metadata`, `riskScore`, `timestamp`, `documentHash`
- **Edge cases:**
  - `metadata`/`timestamp` may not exist in current JSON (risk score node produces `summary.analysisTimestamp` but not `timestamp`).
  - Requires Postgres credentials (not shown in JSON; must be configured in n8n).
  - Table schema must match provided columns.

#### Node: Notify Compliance Team
- **Type/role:** `Slack` message (OAuth2).
- **Key configuration:**
  - Channel: `<__PLACEHOLDER_VALUE__Compliance_Channel_ID__>`
  - Text includes `documentHash` and `riskScore`; risk level hard-coded as “Medium”.
- **Edge cases:**
  - Message references “review link” but none is generated in workflow.
  - Slack OAuth2 scopes must allow posting to the channel.

#### Node: Wait for Human Review
- **Type/role:** `Wait` node, resume via webhook.
- **Key configuration:** `resume: webhook`
- **Behavior:** Pauses execution until an external call hits the generated wait-resume URL.
- **Edge cases:**
  - `maxWaitTimeHours` from configuration is not applied here; by default waits can remain pending until manual cleanup/timeout policies.
  - You must build an external review UI/process that calls the resume webhook with fields like `reviewDecision` and `reviewedBy` (expected by Store High Risk Documents).

#### Node: Store High Risk Documents
- **Type/role:** `Postgres` storage for reviewed cases.
- **Key configuration:**
  - Table: `<__PLACEHOLDER_VALUE__High_Risk_Documents_Table__>`
  - Stores: `riskScore`, `timestamp`, `reviewedBy`, `documentHash`, `reviewDecision`
  - Has a defined schema/matchingColumns list (suggesting upsert/matching behavior)
- **Edge cases:**
  - `timestamp`, `reviewDecision`, `reviewedBy` are not created by upstream nodes unless the wait resume payload includes them.
  - Consider also storing full analysis data for auditability.

#### Node: Alert Security Team
- **Type/role:** `Slack` message for critical security alert.
- **Key configuration:**
  - Channel: `<__PLACEHOLDER_VALUE__Security_Alert_Channel_ID__>`
  - mrkdwn + link_names enabled
- **Edge cases:** Same Slack credential/scopes concerns.

#### Node: Store Critical Risk Documents
- **Type/role:** `Postgres` storage for quarantined/critical cases.
- **Key configuration:**
  - Table: `<__PLACEHOLDER_VALUE__Critical_Risk_Documents_Table__>`
  - Stores: `riskScore`, `timestamp`, `documentHash`, `allAnalysisResults`
- **Edge cases:**
  - `allAnalysisResults` not produced by upstream nodes (workflow uses different property names like `metadataAnalysis`, etc.).
  - `timestamp` missing unless you set it.

#### Node: Log Audit Trail
- **Type/role:** `Postgres` audit logging.
- **Key configuration:**
  - Table: `<__PLACEHOLDER_VALUE__Audit_Trail_Table__>`
  - Stores: `userId`, `riskLevel`, `riskScore`, `documentHash`, `allAnalysisData`, `processingDuration`, `processingTimestamp`
- **Edge cases:**
  - Many fields are not set anywhere (`userId`, `allAnalysisData`, `processingDuration`, `processingTimestamp`).
  - Consider populating these in a dedicated Set node right before logging.

#### Node: Generate Cryptographic Proof
- **Type/role:** `Crypto` node signing the JSON payload.
- **Key configuration:**
  - Action: `sign`
  - Algorithm: `RSA-SHA256`
  - Encoding: base64
  - Signs: `{{ JSON.stringify($json) }}`
  - Private key: literal placeholder `YOUR_CREDENTIAL_HERE`
  - Output property: `cryptographicProof`
- **Edge cases:**
  - You must store the private key securely (prefer n8n credentials or environment variable injection, not hard-coded).
  - Invalid PEM format will fail signing.

#### Node: Prepare Response Data
- **Type/role:** `Set` node formatting the final webhook response.
- **Key configuration:**
  - Sets: `status`, `documentHash`, `riskScore`, `riskLevel`, `processingTimestamp`, `cryptographicProof`, `auditTrailId`
  - Uses `{{ $now.toISO() }}`
  - References `{{ $json.auditId }}` but no node sets `auditId` (Postgres node doesn’t return that by default unless configured).
- **Edge cases:** Missing `auditId` leads to null/empty `auditTrailId`.

#### Node: Return Processing Result
- **Type/role:** `Respond to Webhook` node.
- **Key configuration:**
  - HTTP 200
  - Respond with JSON
  - “Enable response output” enabled (returns data)
- **Edge cases:** If execution pauses at Wait node, webhook response may not be returned until resumed—this can cause client timeouts. Consider responding immediately and handling review asynchronously.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Document Ingestion Webhook | Webhook | Receives document upload (binary) | — | Workflow Configuration | ## How It Works<br>This workflow automates intelligent document analysis… via Gmail… (note: description doesn’t match actual Slack/Postgres implementation) |
| Workflow Configuration | Set | Defines thresholds + shared IDs | Document Ingestion Webhook | Generate Document Hash | ## Batch Document Ingestion & Classification<br>**Why:** Processes multiple document formats… |
| Generate Document Hash | Crypto | SHA-256 hash of binary for traceability | Workflow Configuration | Route by Document Type | ## Batch Document Ingestion & Classification<br>**Why:** Processes multiple document formats… |
| Route by Document Type | Switch | Branches extraction by MIME type | Generate Document Hash | Extract Text from PDF; OCR Processing for Images; Extract from Office Documents | ## Batch Document Ingestion & Classification<br>**Why:** Processes multiple document formats… |
| Extract Text from PDF | Extract From File | PDF text extraction | Route by Document Type | Merge Extracted Content | ## Batch Document Ingestion & Classification<br>**Why:** Processes multiple document formats… |
| OCR Processing for Images | Code | Placeholder OCR → text | Route by Document Type | Merge Extracted Content | ## Batch Document Ingestion & Classification<br>**Why:** Processes multiple document formats… |
| Extract from Office Documents | Extract From File | Office extraction (configured as XLSX) | Route by Document Type | Merge Extracted Content | ## Batch Document Ingestion & Classification<br>**Why:** Processes multiple document formats… |
| Merge Extracted Content | Merge | Combines extracted text streams | Extract Text from PDF; OCR Processing for Images; Extract from Office Documents | Metadata Extraction Agent; Forgery Detection Agent; Semantic Contradiction Detector; Historical Document Vector Store | ## Parallel Multi-Pipeline AI Analysis<br>**Why:** Routes documents through specialized AI processing streams… |
| Metadata Extraction Agent | LangChain Agent | Structured metadata extraction | Merge Extracted Content | Merge Analysis Results | ## Parallel Multi-Pipeline AI Analysis<br>**Why:** Routes documents through specialized AI processing streams… |
| OpenAI Model - Metadata | OpenAI Chat Model | LLM for metadata | (AI connection) | Metadata Extraction Agent | ## Setup Steps<br>…Add OpenAI API key with GPT-4 access… (model used is gpt-4.1-mini) |
| Metadata Schema Parser | Structured Output Parser | Enforces metadata schema | (AI connection) | Metadata Extraction Agent | ## Parallel Multi-Pipeline AI Analysis<br>… |
| Forgery Detection Agent | LangChain Agent | Structured forgery detection | Merge Extracted Content | Merge Analysis Results | ## Parallel Multi-Pipeline AI Analysis<br>… |
| OpenAI Model - Forgery | OpenAI Chat Model | LLM for forgery analysis | (AI connection) | Forgery Detection Agent | ## Setup Steps<br>…OpenAI… |
| Forgery Analysis Schema | Structured Output Parser | Enforces forgery schema | (AI connection) | Forgery Detection Agent | ## Parallel Multi-Pipeline AI Analysis<br>… |
| Semantic Contradiction Detector | LangChain Agent | Structured contradiction detection | Merge Extracted Content | Merge Analysis Results | ## Parallel Multi-Pipeline AI Analysis<br>… |
| OpenAI Model - Semantic | OpenAI Chat Model | LLM for semantic analysis | (AI connection) | Semantic Contradiction Detector | ## Setup Steps<br>…OpenAI… |
| Contradiction Schema Parser | Structured Output Parser | Enforces contradiction schema | (AI connection) | Semantic Contradiction Detector | ## Parallel Multi-Pipeline AI Analysis<br>… |
| OpenAI Embeddings | OpenAI Embeddings | Creates embeddings for vector store | (AI connection) | Historical Document Vector Store | ## Parallel Multi-Pipeline AI Analysis<br>… |
| Document Loader | Document Loader | Creates documents from extracted text | (implicit upstream via splitter) | Historical Document Vector Store | ## Parallel Multi-Pipeline AI Analysis<br>… |
| Text Splitter | Text Splitter | Splits text into chunks | — | Document Loader | ## Parallel Multi-Pipeline AI Analysis<br>… |
| Historical Document Vector Store | Vector Store (In-Memory) | Inserts doc vectors into memory | Merge Extracted Content; OpenAI Embeddings; Document Loader | Historical Pattern Comparison Agent | ## Parallel Multi-Pipeline AI Analysis<br>… |
| Historical Pattern Comparison Agent | LangChain Agent | Compares against historical patterns (conceptually) | Historical Document Vector Store | Merge Analysis Results | ## Parallel Multi-Pipeline AI Analysis<br>… |
| OpenAI Model - Pattern | OpenAI Chat Model | LLM for pattern analysis | (AI connection) | Historical Pattern Comparison Agent | ## Setup Steps<br>…OpenAI… |
| Pattern Analysis Schema | Structured Output Parser | Enforces pattern schema | (AI connection) | Historical Pattern Comparison Agent | ## Parallel Multi-Pipeline AI Analysis<br>… |
| Merge Analysis Results | Merge | Combines 4 analysis outputs | Metadata Extraction Agent; Forgery Detection Agent; Semantic Contradiction Detector; Historical Pattern Comparison Agent | Entity Enrichment Agent | ## Cross-Reference & Recommendation Synthesis<br>**Why:** Aggregates findings… |
| Entity Enrichment Tool | LangChain Tool (Code) | Enriches entities (simulated) | — (called by agent) | Entity Enrichment Agent | ## Cross-Reference & Recommendation Synthesis<br>… |
| Entity Enrichment Agent | LangChain Agent | Calls tool + returns structured entity enrichment | Merge Analysis Results | Calculate Risk Score | ## Cross-Reference & Recommendation Synthesis<br>… |
| OpenAI Model - Enrichment | OpenAI Chat Model | LLM for enrichment | (AI connection) | Entity Enrichment Agent | ## Setup Steps<br>…OpenAI… |
| Enrichment Schema Parser | Structured Output Parser | Enforces enrichment schema | (AI connection) | Entity Enrichment Agent | ## Cross-Reference & Recommendation Synthesis<br>… |
| Calculate Risk Score | Code | Weighted risk scoring + explanation | Entity Enrichment Agent | Route by Risk Level | ## Cross-Reference & Recommendation Synthesis<br>… |
| Route by Risk Level | Switch | Routes by thresholds | Calculate Risk Score | Store Low Risk Documents; Notify Compliance Team; Alert Security Team | ## Report Generation & Automated Distribution<br>**Why:** Compiles structured reports… (note: actual output is Slack+DB, not Gmail) |
| Store Low Risk Documents | Postgres | Stores low-risk results | Route by Risk Level | Log Audit Trail |  |
| Notify Compliance Team | Slack | Alerts compliance for review | Route by Risk Level | Wait for Human Review |  |
| Wait for Human Review | Wait | Pauses until review webhook resumes | Notify Compliance Team | Store High Risk Documents |  |
| Store High Risk Documents | Postgres | Stores reviewed medium-risk outcomes | Wait for Human Review | Log Audit Trail |  |
| Alert Security Team | Slack | Critical alert to security | Route by Risk Level | Store Critical Risk Documents |  |
| Store Critical Risk Documents | Postgres | Stores quarantined/critical cases | Alert Security Team | Log Audit Trail |  |
| Log Audit Trail | Postgres | Writes audit log | Store Low Risk Documents; Store High Risk Documents; Store Critical Risk Documents | Generate Cryptographic Proof |  |
| Generate Cryptographic Proof | Crypto | Signs final JSON payload | Log Audit Trail | Prepare Response Data |  |
| Prepare Response Data | Set | Shapes final response payload | Generate Cryptographic Proof | Return Processing Result |  |
| Return Processing Result | Respond to Webhook | Sends JSON response | Prepare Response Data | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (keep it inactive until credentials/tables are ready).

2. **Add “Document Ingestion Webhook” (Webhook node)**
   - Method: **POST**
   - Path: `document-ingestion`
   - Options → Binary Property Name: `data`
   - Response: **Last Node**

3. **Add “Workflow Configuration” (Set node)**
   - Add fields:
     - `riskThresholdLow` = 30 (Number)
     - `riskThresholdHigh` = 70 (Number)
     - `riskThresholdCritical` = 90 (Number)
     - `maxWaitTimeHours` = 48 (Number)
     - `vectorStoreMemoryId` = `fraud_detection_docs` (String)
   - Enable **Include Other Fields**.
   - Connect: Webhook → Workflow Configuration

4. **Add “Generate Document Hash” (Crypto node)**
   - Operation: **Hash**
   - Type: **SHA256**
   - Enable **Binary Data**
   - Output property name: `documentHash`
   - Connect: Workflow Configuration → Generate Document Hash

5. **Add “Route by Document Type” (Switch node)**
   - Value to evaluate: `{{ $json.mimeType }}`
   - Create rules:
     - PDF: contains `pdf`
     - Image: contains `image`
     - Office: contains `officedocument` OR `msword` OR `ms-excel`
   - Set fallback output (e.g., “extra”) and decide how to handle it (recommended: add an error response).
   - Connect: Generate Document Hash → Switch

6. **Add extraction nodes**
   - **Extract Text from PDF** (Extract From File)
     - Operation: `pdf`
     - Join pages: enabled
     - Connect from Switch “PDF”
   - **OCR Processing for Images** (Code)
     - Paste your OCR integration (replace placeholder code, or keep simulated behavior)
     - Connect from Switch “Image”
   - **Extract from Office Documents** (Extract From File)
     - If you need Word/PowerPoint too, create separate nodes; otherwise set operation `xlsx`
     - Connect from Switch “Office”

7. **Add “Merge Extracted Content” (Merge node)**
   - Mode: **Combine**
   - Combine by: **Position**
   - Number of inputs: 3
   - Connect each extraction node into one input of this merge.

8. **Add AI analysis agents + models + parsers (4 parallel branches)**
   - For each branch, add:
     - a **LangChain Agent** node
     - an **OpenAI Chat Model** node (`gpt-4.1-mini`)
     - a **Structured Output Parser** node with the corresponding schema
   - Configure each agent’s **text** to `{{ $json.text }}` and its system message as in the workflow.
   - Wire AI connections:
     - Model → Agent (ai_languageModel)
     - Parser → Agent (ai_outputParser)
   - Wire main connections:
     - Merge Extracted Content → each Agent

9. **Set up historical vector store insertion pipeline**
   - Add **OpenAI Embeddings** node (requires OpenAI credential)
   - Add **Text Splitter** (Recursive Character) with chunkOverlap 200
   - Add **Document Loader**:
     - jsonData: `{{ $json.text }}`
     - textSplittingMode: custom
     - connect Text Splitter to Document Loader via `ai_textSplitter`
   - Add **Vector Store In-Memory** node:
     - mode: insert
     - memoryKey: `{{ $('Workflow Configuration').item.json.vectorStoreMemoryId }}`
     - connect:
       - OpenAI Embeddings → Vector Store (ai_embedding)
       - Document Loader → Vector Store (ai_document)
       - Merge Extracted Content → Vector Store (main)
   - Connect Vector Store (main) → Historical Pattern Comparison Agent (main)

10. **Add “Merge Analysis Results” (Merge node)**
   - Mode: combine, by position, numberInputs: 4
   - Connect the four analysis agents’ **main outputs** into the four merge inputs.

11. **Add Entity Enrichment toolchain**
   - Add **Tool Code** node (“Entity Enrichment Tool”), implement real HTTP calls or keep placeholders.
   - Add **Entity Enrichment Agent**:
     - text: `{{ JSON.stringify($json) }}`
     - connect the tool to the agent (ai_tool)
     - connect OpenAI chat model (gpt-4.1-mini) (ai_languageModel)
     - connect Enrichment Schema Parser (ai_outputParser)
   - Connect: Merge Analysis Results → Entity Enrichment Agent

12. **Add “Calculate Risk Score” (Code node)**
   - Paste risk scoring code.
   - Connect: Entity Enrichment Agent → Calculate Risk Score
   - Recommended fix: insert a Set/Code node before this to map agent outputs into:
     - `metadataAnalysis`, `forgeryAnalysis`, `semanticAnalysis`, `patternAnalysis`, `enrichmentAnalysis`

13. **Add “Route by Risk Level” (Switch node)**
   - Compare `{{ $json.riskScore }}` to thresholds from Workflow Configuration.
   - Create outputs: Low, Medium, High; fallback renamed to Critical.
   - Connect: Calculate Risk Score → Route by Risk Level

14. **Add Slack notifications (OAuth2 credential required)**
   - “Notify Compliance Team” Slack node:
     - Post message to compliance channel ID placeholder
     - Connect from Route by Risk Level (Medium)
   - “Alert Security Team” Slack node:
     - Post message to security channel ID placeholder
     - Connect from Route by Risk Level (High/Critical path as desired)

15. **Add “Wait for Human Review” (Wait node)**
   - Mode: Resume via webhook
   - Connect: Notify Compliance Team → Wait
   - Ensure your external reviewer process calls the resume webhook with `reviewDecision`, `reviewedBy`, and any other needed fields.

16. **Add Postgres storage nodes (Postgres credential required)**
   - Create tables (or adjust mapping to your schema):
     - Low risk table
     - High risk reviewed table
     - Critical risk table
     - Audit trail table
   - Add nodes:
     - Store Low Risk Documents (connect from Switch low)
     - Store High Risk Documents (connect from Wait)
     - Store Critical Risk Documents (connect from Security Slack)
     - Log Audit Trail (connect from each store node)

17. **Add “Generate Cryptographic Proof” (Crypto node)**
   - Action: sign
   - Algorithm: RSA-SHA256
   - Value: `{{ JSON.stringify($json) }}`
   - Store private key securely (credential/env var)
   - Connect: Log Audit Trail → Generate Cryptographic Proof

18. **Add “Prepare Response Data” (Set node)**
   - Set output fields: status, documentHash, riskScore, riskLevel, processingTimestamp, cryptographicProof, auditTrailId
   - Ensure `auditTrailId` is actually available (configure Postgres node to return ID, or generate one).

19. **Add “Return Processing Result” (Respond to Webhook)**
   - Respond with JSON, code 200
   - Connect: Prepare Response Data → Respond to Webhook

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Prerequisites: NVIDIA NIM API access, OpenAI API key (GPT-4), Anthropic Claude API key …” | Sticky note content; however, this workflow JSON only uses OpenAI nodes + Slack/Postgres. No NVIDIA/Claude nodes are present. |
| “Setup Steps … Google Sheets … Gmail OAuth2 …” | Sticky note content does not match implemented nodes (no Google Sheets/Gmail nodes exist). |
| “How It Works … delivered via email … Gmail” | Sticky note description mismatches actual implementation (Slack + Postgres + webhook response). |
| Multiple nodes contain placeholder values for endpoints, API keys, Slack channel IDs, and Postgres table names. | Replace `<__PLACEHOLDER_VALUE__...__>` entries before production use. |
| Wait-for-review design can cause webhook caller timeouts if you truly wait before responding. | Consider asynchronous response pattern (respond immediately, continue processing separately). |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.