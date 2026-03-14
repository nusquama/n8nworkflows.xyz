Mask PII in documents for GDPR-safe AI processing with Postgres and Claude

https://n8nworkflows.xyz/workflows/mask-pii-in-documents-for-gdpr-safe-ai-processing-with-postgres-and-claude-13941


# Mask PII in documents for GDPR-safe AI processing with Postgres and Claude

# 1. Workflow Overview

This workflow implements a GDPR-oriented document-processing pipeline for PDF uploads. It extracts text from an uploaded document, detects personally identifiable information (PII), replaces detected values with tokens, stores originals in a Postgres vault, sends only masked text to an AI model, and optionally restores approved values afterward. It also writes compliance events to an audit log.

The intended use cases include:
- Safe AI summarization or extraction from documents containing personal data
- Privacy-preserving preprocessing before sending data to Claude
- Token-based de-identification with controlled re-injection
- Auditability for compliance programs

## 1.1 Document Intake and Baseline Configuration
The workflow starts from a webhook that accepts uploaded files. A Set node then adds runtime configuration such as `documentId`, confidence threshold, and Postgres table names.

## 1.2 OCR Text Extraction
The uploaded PDF is passed to an OCR/text extraction step that produces text used by all downstream PII detectors.

## 1.3 PII Detection Layer
Multiple parallel detection branches scan the extracted text:
- Regex/code-based email detection
- Regex/code-based phone detection
- Regex/code-based ID number detection
- AI-based physical address detection using a LangChain agent with Ollama

## 1.4 Detection Merge and Consolidation
All detection outputs are merged, then a code node attempts to resolve overlaps, remove duplicates, and produce a consolidated PII map.

## 1.5 Tokenization and Vault Storage
Detected PII values are converted into placeholder tokens, and original values are prepared for insertion into a Postgres vault table.

## 1.6 Masked Text Generation and Safety Gate
The workflow builds a masked version of the OCR text and checks whether masking succeeded. If masking fails, AI processing is blocked and an alert is sent.

## 1.7 AI Processing on Masked Data
If masking is successful, the masked text is sent to a Claude-based AI agent for structured document analysis. A structured output parser constrains the response format.

## 1.8 Controlled Re-Injection
The workflow analyzes the AI output for tokens, decides whether certain fields may be restored, queries the vault for original values, and attempts to restore approved values only.

## 1.9 Audit Logging
The final restoration output is written to a Postgres audit table for traceability.

---

# 2. Block-by-Block Analysis

## 2.1 Document Intake and Baseline Configuration

### Overview
This block receives the uploaded PDF through an HTTP webhook and initializes workflow-level settings used later for vault and audit storage. It also creates a timestamp-based document identifier.

### Nodes Involved
- Document Upload Webhook
- Workflow Configuration

### Node Details

#### Document Upload Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point for inbound HTTP document uploads.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `gdpr-document-upload`
  - Response mode: `lastNode`
  - Raw body enabled
- **Key expressions or variables used:**
  - None in configuration
- **Input and output connections:**
  - Input: none
  - Output: `Workflow Configuration`
- **Version-specific requirements:**
  - Type version `2.1`
- **Edge cases or potential failure types:**
  - Invalid upload format
  - Missing binary file payload
  - If OCR expects binary and webhook payload is malformed, downstream extraction fails
  - Response may hang if downstream nodes fail without error handling
- **Sub-workflow reference:** none

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`; enriches the incoming item with workflow variables.
- **Configuration choices:**
  - Adds:
    - `documentId = {{$now.toISO()}}`
    - `confidenceThreshold = 0.8`
    - `vaultTable = "pii_vault"`
    - `auditTable = "pii_audit_log"`
  - Includes other incoming fields
- **Key expressions or variables used:**
  - `$now.toISO()`
- **Input and output connections:**
  - Input: `Document Upload Webhook`
  - Output: `OCR Extract Text`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - Timestamp-based `documentId` may not be unique enough under high concurrency compared to UUIDs
  - If later nodes expect different field names, data mismatch can occur
- **Sub-workflow reference:** none

---

## 2.2 OCR Text Extraction

### Overview
This block extracts text from the uploaded PDF and preserves source data. The extracted text becomes the canonical input for all PII detection branches.

### Nodes Involved
- OCR Extract Text

### Node Details

#### OCR Extract Text
- **Type and technical role:** `n8n-nodes-base.extractFromFile`; PDF text extraction/OCR-style extraction from uploaded file content.
- **Configuration choices:**
  - Operation: `pdf`
  - Keep source: `both`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Input: `Workflow Configuration`
  - Outputs to:
    - `Email Detector`
    - `Phone Detector`
    - `ID Number Detector`
    - `Address Detector AI`
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - Fails if no binary PDF is present
  - OCR/text extraction quality may be poor for scanned or image-heavy PDFs
  - Large PDFs may increase runtime or memory use
  - Extracted text field naming consistency matters because code nodes use `text` or `extractedText`
- **Sub-workflow reference:** none

---

## 2.3 PII Detection Layer

### Overview
This block runs multiple detectors in parallel on the OCR output. Three are regex-based code nodes, and one is an AI-powered address detector using Ollama plus a structured output parser.

### Nodes Involved
- Email Detector
- Phone Detector
- ID Number Detector
- Address Detector AI
- Address Output Parser
- Ollama Chat Model

### Node Details

#### Email Detector
- **Type and technical role:** `n8n-nodes-base.code`; regex-based email extraction from OCR text.
- **Configuration choices:**
  - Uses a global email regex
  - Emits objects with:
    - `value`
    - `type = "email"`
    - `start_pos`
    - `end_pos`
    - `confidence = 1.0`
  - Returns payload under `detections`
- **Key expressions or variables used:**
  - `item.json.text`
- **Input and output connections:**
  - Input: `OCR Extract Text`
  - Output: `Merge PII Detections`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Can miss unusual but valid email formats
  - Can detect OCR-corrupted strings as emails
  - Uses `start_pos/end_pos`, but downstream consolidation expects `start/end`
- **Sub-workflow reference:** none

#### Phone Detector
- **Type and technical role:** `n8n-nodes-base.code`; regex-based phone detection with several common formats.
- **Configuration choices:**
  - Uses multiple regex patterns for international and domestic formats
  - Deduplicates values with `Set`
  - Outputs under `detected_pii`, not `detections`
  - Includes `start_pos` and `end_pos`
- **Key expressions or variables used:**
  - `item.json.text`
- **Input and output connections:**
  - Input: `OCR Extract Text`
  - Output: `Merge PII Detections`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Output schema is inconsistent with other detectors
  - Because downstream merger/consolidator reads `detections`, phone matches may be ignored
  - OCR spacing/noise may break patterns
  - Possible false positives on numeric strings
- **Sub-workflow reference:** none

#### ID Number Detector
- **Type and technical role:** `n8n-nodes-base.code`; regex-based detection of SSN, PAN, driver’s license, bank account numbers, and IBAN.
- **Configuration choices:**
  - Reads `extractedText` or `text`
  - Emits array under `detections`
  - Uses subtype values such as `ssn`, `pan`, `drivers_license`, `bank_account`, `iban`
- **Key expressions or variables used:**
  - `item.json.extractedText || item.json.text || ''`
- **Input and output connections:**
  - Input: `OCR Extract Text`
  - Output: `Merge PII Detections`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - High false-positive risk for generic long numbers
  - Validation is minimal
  - Contains an odd SSN exclusion check against `'+1234567890'`, likely unintended
  - Uses `start_pos/end_pos`, but downstream consolidation expects `start/end`
- **Sub-workflow reference:** none

#### Address Detector AI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; LLM agent used to detect physical addresses in text.
- **Configuration choices:**
  - Input text: `={{ $json.text }}`
  - Prompt type: defined directly
  - Uses a system message instructing the model to detect physical addresses and return positions plus confidence
  - Structured output parser enabled
- **Key expressions or variables used:**
  - `$json.text`
- **Input and output connections:**
  - Main input: `OCR Extract Text`
  - AI language model input: `Ollama Chat Model`
  - AI output parser input: `Address Output Parser`
  - Main output: `Merge PII Detections`
- **Version-specific requirements:**
  - Type version `3`
  - Requires LangChain-compatible n8n installation
- **Edge cases or potential failure types:**
  - LLM may hallucinate addresses or positions
  - Output schema may not match what downstream consolidation expects
  - If `text` is empty, output quality collapses
  - Depends on reachable Ollama instance and model availability
- **Sub-workflow reference:** none

#### Address Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces JSON schema for address detections.
- **Configuration choices:**
  - Manual JSON schema with root property `addresses`
  - Each address object includes:
    - `value`
    - `type` enum `address`
    - `start_pos`
    - `end_pos`
    - `confidence`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Output parser connection into `Address Detector AI`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - If model output cannot be parsed, the agent node fails
  - Schema root uses `addresses`, not `detections`, causing downstream mismatch unless normalized
- **Sub-workflow reference:** none

#### Ollama Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOllama`; local LLM backend for address detection.
- **Configuration choices:**
  - No explicit model shown in parameters
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Output to `Address Detector AI` as language model
- **Version-specific requirements:**
  - Type version `1`
  - Requires a running Ollama service accessible to n8n
- **Edge cases or potential failure types:**
  - Missing Ollama endpoint/model
  - Latency or model startup delays
  - Local model quality may be inconsistent
- **Sub-workflow reference:** none

---

## 2.4 Detection Merge and Consolidation

### Overview
This block gathers outputs from the parallel detectors and tries to build a single conflict-resolved PII list. It is meant to remove duplicates and overlapping detections before tokenization.

### Nodes Involved
- Merge PII Detections
- PII Consolidation & Conflict Resolver

### Node Details

#### Merge PII Detections
- **Type and technical role:** `n8n-nodes-base.merge`; combines multiple detector outputs.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Inputs from:
    - `Email Detector`
    - `Phone Detector`
    - `ID Number Detector`
    - `Address Detector AI`
  - Output: `PII Consolidation & Conflict Resolver`
- **Version-specific requirements:**
  - Type version `3.2`
- **Edge cases or potential failure types:**
  - Multi-input semantics depend on n8n merge behavior; misalignment can produce unexpected payload structure
  - Since detector outputs are structurally inconsistent, combined result may not be usable as intended
- **Sub-workflow reference:** none

#### PII Consolidation & Conflict Resolver
- **Type and technical role:** `n8n-nodes-base.code`; intended to normalize all detections, resolve overlaps, deduplicate, and emit a final PII map.
- **Configuration choices:**
  - Reads all incoming items
  - Collects only `item.json.detections`
  - Sorts on `a.start - b.start`
  - Resolves overlap by confidence, then span length
  - Deduplicates by `type|value|start|end`
  - Creates `piiMap` with ids like `pii_1`
- **Key expressions or variables used:**
  - `$input.all()`
  - `$input.first().json.extractedText || $input.first().json.text || ''`
- **Input and output connections:**
  - Input: `Merge PII Detections`
  - Output: `Tokenization & Vault Storage`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Major schema mismatch:
    - detectors emit `start_pos/end_pos`, not `start/end`
    - phone detector emits `detected_pii`, not `detections`
    - address parser root is `addresses`, not `detections`
  - As written, overlap logic may fail or produce incorrect sort order
  - `originalText` may not survive merge reliably
- **Sub-workflow reference:** none

---

## 2.5 Tokenization and Vault Storage

### Overview
This block generates token placeholders for detected PII values and prepares the records to be stored in a Postgres vault. It is the core privacy-preserving step before AI access.

### Nodes Involved
- Tokenization & Vault Storage
- Store Tokens in Vault

### Node Details

#### Tokenization & Vault Storage
- **Type and technical role:** `n8n-nodes-base.code`; generates random token strings and vault records, and also builds a masked text copy.
- **Configuration choices:**
  - Expects `item.json.consolidatedPII`
  - Generates tokens like `<<TYPE_AB12>>`
  - Builds:
    - `vaultRecords`
    - `tokenMap`
    - `maskedText`
    - `documentId`
- **Key expressions or variables used:**
  - `item.json.consolidatedPII || []`
  - `item.json.originalText || ''`
- **Input and output connections:**
  - Input: `PII Consolidation & Conflict Resolver`
  - Output: `Store Tokens in Vault`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Critical field mismatch: previous node outputs `piiMap`, not `consolidatedPII`
  - As configured, likely generates zero tokens unless corrected
  - Random 4-hex token suffix has collision risk in larger volumes
  - Replacing by raw value can unintentionally replace repeated context outside intended positions
- **Sub-workflow reference:** none

#### Store Tokens in Vault
- **Type and technical role:** `n8n-nodes-base.postgres`; inserts token/original-value pairs into Postgres vault table.
- **Configuration choices:**
  - Schema: `public`
  - Table name from `Workflow Configuration.vaultTable`
  - Column mapping:
    - `type = {{$json.type}}`
    - `token = "YOUR_CREDENTIAL_HERE"` ← placeholder/static value
    - `created_at`
    - `document_id`
    - `original_value`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.vaultTable }}`
  - Several `{{$json...}}` mappings
- **Input and output connections:**
  - Input: `Tokenization & Vault Storage`
  - Output: `Generate Masked Text`
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires Postgres credentials
- **Edge cases or potential failure types:**
  - Appears misconfigured:
    - token column is hardcoded to `"YOUR_CREDENTIAL_HERE"`
    - likely not iterating correctly over `vaultRecords`
  - If table schema differs from mapping, inserts fail
  - Missing insert/operation specifics may depend on default behavior
- **Sub-workflow reference:** none

---

## 2.6 Masked Text Generation and Safety Gate

### Overview
This block attempts to create the final masked text used for AI processing and verifies whether masking was successful. If not, the workflow stops AI exposure and sends an alert.

### Nodes Involved
- Generate Masked Text
- Masking Success Check
- Block AI Processing
- Send Alert Notification

### Node Details

#### Generate Masked Text
- **Type and technical role:** `n8n-nodes-base.code`; replaces original PII values in OCR text using vault records.
- **Configuration choices:**
  - Reads OCR text directly from `OCR Extract Text`
  - Reads tokenized records from `Store Tokens in Vault`
  - Replaces each `original_value` with `token`
  - Returns:
    - `masked_text`
    - `original_text`
    - `token_count`
    - `masking_success`
    - `replacements`
- **Key expressions or variables used:**
  - `$('OCR Extract Text').first().json`
  - `$('Store Tokens in Vault').all()`
- **Input and output connections:**
  - Input: `Store Tokens in Vault`
  - Output: `Masking Success Check`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - If vault inserts do not return `token` and `original_value`, masking fails
  - If Postgres node output differs from expected insert-return format, no replacements occur
  - Raw string replacement can replace unintended duplicate values
- **Sub-workflow reference:** none

#### Masking Success Check
- **Type and technical role:** `n8n-nodes-base.if`; safety gate before AI processing.
- **Configuration choices:**
  - Condition checks whether `$('Generate Masked Text').item.json.masking_success` equals `true`
- **Key expressions or variables used:**
  - `={{ $('Generate Masked Text').item.json.masking_success }}`
- **Input and output connections:**
  - Input: `Generate Masked Text`
  - True output: `AI Processing (Masked Data)`
  - False output: `Block AI Processing`
- **Version-specific requirements:**
  - Type version `2.3`
- **Edge cases or potential failure types:**
  - If expression fails or item shape is unexpected, routing may misbehave
  - If zero PII is found, current logic may treat masking as failure depending on earlier output
- **Sub-workflow reference:** none

#### Block AI Processing
- **Type and technical role:** `n8n-nodes-base.set`; creates a blocked-status payload when masking is unsafe.
- **Configuration choices:**
  - Sets:
    - `error = "Masking failed - AI processing blocked"`
    - `status = "BLOCKED"`
    - `requires_manual_review = true`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Input: `Masking Success Check` false branch
  - Output: `Send Alert Notification`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - Does not preserve original fields unless default include behavior does so; downstream alert expects fields not guaranteed to exist
- **Sub-workflow reference:** none

#### Send Alert Notification
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends failure notification to an external endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL placeholder must be replaced
  - Sends body with:
    - `error_details = {{$json.error_details}}`
    - `document_id = {{$json.document_id}}`
    - `timestamp = {{$now.toISO()}}`
- **Key expressions or variables used:**
  - `$json.error_details`
  - `$json.document_id`
  - `$now.toISO()`
- **Input and output connections:**
  - Input: `Block AI Processing`
  - Output: none
- **Version-specific requirements:**
  - Type version `4.3`
- **Edge cases or potential failure types:**
  - Placeholder URL causes immediate failure until configured
  - `error_details` and `document_id` are not set by the previous node as currently written
  - External webhook/network/auth errors possible
- **Sub-workflow reference:** none

---

## 2.7 AI Processing on Masked Data

### Overview
This block sends the masked document to Claude for structured analysis. A structured parser constrains the result format so downstream logic can inspect it programmatically.

### Nodes Involved
- AI Processing (Masked Data)
- AI Processing Model
- AI Output Parser

### Node Details

#### AI Processing (Masked Data)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI agent that processes only masked text.
- **Configuration choices:**
  - Input text: `={{ $json.masked_text }}`
  - System message instructs the model to preserve tokens like `<<EMAIL_7F3A>>`
  - Structured output parser enabled
- **Key expressions or variables used:**
  - `$json.masked_text`
- **Input and output connections:**
  - Main input: `Masking Success Check` true branch
  - AI language model input: `AI Processing Model`
  - AI output parser input: `AI Output Parser`
  - Main output: `Re-Injection Controller`
- **Version-specific requirements:**
  - Type version `3`
  - Requires LangChain nodes support
- **Edge cases or potential failure types:**
  - Model may alter token formatting despite prompt
  - If masked text is empty, output utility drops
  - Anthropic credential or model access errors possible
- **Sub-workflow reference:** none

#### AI Processing Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Claude chat model backend.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Output to `AI Processing (Masked Data)` as language model
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires Anthropic API credentials
  - Model availability depends on current n8n/Anthropic support
- **Edge cases or potential failure types:**
  - Model name may not exist in all environments
  - Rate limits, quota errors, auth issues
- **Sub-workflow reference:** none

#### AI Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates structured AI output.
- **Configuration choices:**
  - Manual schema with fields:
    - `documentType` required
    - `summary` required
    - optional `keyEntities`, `dates`, `amounts`, `processedData`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Output parser connection into `AI Processing (Masked Data)`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - If model response is not valid against schema, agent step fails
  - Numeric coercion in `amounts.amount` may fail if model returns strings
- **Sub-workflow reference:** none

---

## 2.8 Controlled Re-Injection

### Overview
This block identifies whether masked tokens in the AI output should be restored, queries the vault, and attempts to replace authorized tokens with original PII values.

### Nodes Involved
- Re-Injection Controller
- Retrieve Original Values
- Restore Original PII

### Node Details

#### Re-Injection Controller
- **Type and technical role:** `n8n-nodes-base.code`; analyzes AI output, extracts token references, and prepares re-injection metadata.
- **Configuration choices:**
  - Reads `aiOutput || json`
  - Looks for field permissions marked `restore` or `unmask`
  - Recursively scans objects for token pattern:
    - `TOKEN_([A-Z_]+)_([a-f0-9-]+)`
- **Key expressions or variables used:**
  - `$execution.id`
- **Input and output connections:**
  - Input: `AI Processing (Masked Data)`
  - Output: `Retrieve Original Values`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Token pattern does not match earlier token format `<<TYPE_HASH>>`
  - `fieldPermissions` is never set upstream
  - Likely finds zero tokens even when AI preserved placeholders
- **Sub-workflow reference:** none

#### Retrieve Original Values
- **Type and technical role:** `n8n-nodes-base.postgres`; selects original values from the vault by token.
- **Configuration choices:**
  - Operation: `select`
  - Return all: `true`
  - Table from `Workflow Configuration.vaultTable`
  - Where clause:
    - column `token`
    - value `={{ $('Re-Injection Controller').item.json.token }}`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.vaultTable }}`
  - `={{ $('Re-Injection Controller').item.json.token }}`
- **Input and output connections:**
  - Input: `Re-Injection Controller`
  - Output: `Restore Original PII`
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires Postgres credentials
- **Edge cases or potential failure types:**
  - Re-Injection Controller does not output a top-level `token` field
  - Query likely returns nothing
  - If multiple tokens should be fetched, current configuration does not iterate them cleanly
- **Sub-workflow reference:** none

#### Restore Original PII
- **Type and technical role:** `n8n-nodes-base.code`; replaces tokens in approved fields with original values from vault data.
- **Configuration choices:**
  - Reads first input item as AI output
  - Builds token map from `Retrieve Original Values`
  - Replaces tokens matching regex `\[TOKEN_[A-Z0-9]+\]`
  - Restores only if `allowed_for_reinjection` is truthy
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('Retrieve Original Values').all()`
- **Input and output connections:**
  - Input: `Retrieve Original Values`
  - Output: `Store Audit Log`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Token regex does not match token format used earlier
  - Reads `pii_type` and `allowed_for_reinjection`, which are not inserted in vault storage
  - Likely performs no actual restoration
- **Sub-workflow reference:** none

---

## 2.9 Audit Logging

### Overview
This block writes compliance-related metadata to Postgres. It is intended to track masking and re-injection events.

### Nodes Involved
- Store Audit Log

### Node Details

#### Store Audit Log
- **Type and technical role:** `n8n-nodes-base.postgres`; inserts audit information into Postgres.
- **Configuration choices:**
  - Table from `Workflow Configuration.auditTable`
  - Schema: `public`
  - Mapped columns:
    - `actor = "system"`
    - `timestamp = {{$json.timestamp}}`
    - `document_id = {{$json.document_id}}`
    - `token_count = {{$json.token_count}}`
    - `pii_types_detected = {{$json.pii_types_detected}}`
    - `ai_access_confirmed = true`
    - `re_injection_events = {{$json.re_injection_events}}`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.auditTable }}`
- **Input and output connections:**
  - Input: `Restore Original PII`
  - Output: none
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires Postgres credentials
- **Edge cases or potential failure types:**
  - Upstream node does not currently provide several mapped fields
  - Insert may fail if columns do not exist or types differ
  - Audit record quality is incomplete unless data mapping is corrected
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Document Upload Webhook | n8n-nodes-base.webhook | HTTP entry point for document upload |  | Workflow Configuration | ## Document Upload<br>A webhook receives uploaded documents.<br>This entry point triggers the workflow and passes the file to the OCR step for text extraction. |
| Workflow Configuration | n8n-nodes-base.set | Adds runtime config values and table names | Document Upload Webhook | OCR Extract Text | ## GDPR-Safe AI Document Processing<br><br>This workflow processes uploaded documents while protecting sensitive personal data. When a PDF is uploaded, OCR extracts the text and multiple detectors identify Personally Identifiable Information (PII) such as emails, phone numbers, ID numbers, and addresses.<br><br>Detected PII is consolidated and replaced with secure tokens while the original values are stored in a Postgres vault. The AI model only processes the masked version of the document, ensuring sensitive information is never exposed.<br><br>If required, a controlled re-injection mechanism can restore original values from the vault. All masking, AI access, and restoration events are recorded in an audit log.<br><br>Setup<br><br>Configure Postgres credentials.<br><br>Create pii_vault and pii_audit_log tables.<br><br>Connect an AI model.<br><br>Send documents to the webhook. |
| OCR Extract Text | n8n-nodes-base.extractFromFile | Extracts text from uploaded PDF | Workflow Configuration | Email Detector, Phone Detector, ID Number Detector, Address Detector AI | ## OCR Text Extraction<br><br>Extracts text from uploaded PDF files. |
| Email Detector | n8n-nodes-base.code | Detects email addresses via regex | OCR Extract Text | Merge PII Detections | ## PII Detection Layer<br><br>Multiple detectors scan the document to identify sensitive information such as emails, phone numbers, ID numbers, and physical addresses. |
| Phone Detector | n8n-nodes-base.code | Detects phone numbers via regex | OCR Extract Text | Merge PII Detections | ## PII Detection Layer<br><br>Multiple detectors scan the document to identify sensitive information such as emails, phone numbers, ID numbers, and physical addresses. |
| ID Number Detector | n8n-nodes-base.code | Detects ID-like numeric/alphanumeric patterns | OCR Extract Text | Merge PII Detections | ## PII Detection Layer<br><br>Multiple detectors scan the document to identify sensitive information such as emails, phone numbers, ID numbers, and physical addresses. |
| Address Detector AI | @n8n/n8n-nodes-langchain.agent | Uses AI to detect physical addresses | OCR Extract Text, Ollama Chat Model, Address Output Parser | Merge PII Detections | ## Address Detection (AI) local ollama<br><br>An AI model analyzes the OCR text to detect physical addresses that are harder to capture with regex patterns. |
| Address Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured address output schema |  | Address Detector AI | ## Address Detection (AI) local ollama<br><br>An AI model analyzes the OCR text to detect physical addresses that are harder to capture with regex patterns. |
| Ollama Chat Model | @n8n/n8n-nodes-langchain.lmChatOllama | Local LLM backend for address detection |  | Address Detector AI | ## Address Detection (AI) local ollama<br><br>An AI model analyzes the OCR text to detect physical addresses that are harder to capture with regex patterns. |
| Merge PII Detections | n8n-nodes-base.merge | Combines detector outputs | Email Detector, Phone Detector, ID Number Detector, Address Detector AI | PII Consolidation & Conflict Resolver | ## Merge Detection Results<br><br>All detection outputs are merged into a single dataset. |
| PII Consolidation & Conflict Resolver | n8n-nodes-base.code | Resolves overlaps and deduplicates detections | Merge PII Detections | Tokenization & Vault Storage | ## IResolve Overlapping Detections<br><br>Overlapping or duplicate PII detections are resolved. |
| Tokenization & Vault Storage | n8n-nodes-base.code | Generates placeholder tokens and vault records | PII Consolidation & Conflict Resolver | Store Tokens in Vault | ##Tokenization & Vault Storage<br><br>Each detected PII value is replaced with a secure token such as:<br><br><<EMAIL_AB12>><br>The original values are stored securely in a Postgres vault table. |
| Store Tokens in Vault | n8n-nodes-base.postgres | Inserts token records into Postgres vault | Tokenization & Vault Storage | Generate Masked Text | ##Tokenization & Vault Storage<br><br>Each detected PII value is replaced with a secure token such as:<br><br><<EMAIL_AB12>><br>The original values are stored securely in a Postgres vault table. |
| Generate Masked Text | n8n-nodes-base.code | Replaces original PII in text with tokens | Store Tokens in Vault | Masking Success Check | ##Tokenization & Vault Storage<br><br>Each detected PII value is replaced with a secure token such as:<br><br><<EMAIL_AB12>><br>The original values are stored securely in a Postgres vault table. |
| Masking Success Check | n8n-nodes-base.if | Prevents AI processing if masking failed | Generate Masked Text | AI Processing (Masked Data), Block AI Processing | ## IMasking Safety Check<br><br>Before AI processing, the workflow verifies that masking was successful.<br><br>If masking fails, AI processing is blocked to prevent accidental exposure of sensitive information. |
| Block AI Processing | n8n-nodes-base.set | Produces blocked-status payload | Masking Success Check | Send Alert Notification | ## IMasking Safety Check<br><br>Before AI processing, the workflow verifies that masking was successful.<br><br>If masking fails, AI processing is blocked to prevent accidental exposure of sensitive information. |
| Send Alert Notification | n8n-nodes-base.httpRequest | Sends external alert on masking failure | Block AI Processing |  | ## IMasking Safety Check<br><br>Before AI processing, the workflow verifies that masking was successful.<br><br>If masking fails, AI processing is blocked to prevent accidental exposure of sensitive information. |
| AI Processing (Masked Data) | @n8n/n8n-nodes-langchain.agent | Runs masked document analysis via LLM | Masking Success Check, AI Processing Model, AI Output Parser | Re-Injection Controller | ## AI Processing (Masked Data)<br><br>The masked document is sent to an AI model for analysis.<br><br>Since sensitive data is replaced with tokens, the AI can safely summarize or extract structured information. |
| AI Processing Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backend |  | AI Processing (Masked Data) | ## AI Processing (Masked Data)<br><br>The masked document is sent to an AI model for analysis.<br><br>Since sensitive data is replaced with tokens, the AI can safely summarize or extract structured information. |
| AI Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured AI extraction schema |  | AI Processing (Masked Data) | ## AI Processing (Masked Data)<br><br>The masked document is sent to an AI model for analysis.<br><br>Since sensitive data is replaced with tokens, the AI can safely summarize or extract structured information. |
| Re-Injection Controller | n8n-nodes-base.code | Determines whether tokens should be restored | AI Processing (Masked Data) | Retrieve Original Values | ## PII Re-Injection Controller<br><br>Analyzes AI output to determine whether specific tokens should be replaced with original values.<br><br>Restoration follows defined permissions to control where sensitive data can appear. |
| Retrieve Original Values | n8n-nodes-base.postgres | Queries vault for original token values | Re-Injection Controller | Restore Original PII | ## Restore Original Values<br><br>Original PII values are retrieved from the vault and restored only in approved fields.<br><br>This ensures controlled access to sensitive data. |
| Restore Original PII | n8n-nodes-base.code | Replaces approved tokens with original values | Retrieve Original Values | Store Audit Log | ## Restore Original Values<br><br>Original PII values are retrieved from the vault and restored only in approved fields.<br><br>This ensures controlled access to sensitive data. |
| Store Audit Log | n8n-nodes-base.postgres | Inserts compliance record into audit table | Restore Original PII |  | ## Compliance Audit Log<br><br>All detection, masking, AI access, and restoration events are recorded in a Postgres audit table.<br><br>This provides traceability and supports privacy compliance requirements. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note11 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note12 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note13 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |
| Sticky Note14 | n8n-nodes-base.stickyNote | Documentation/annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence. It follows the current workflow closely, while also pointing out places where the original JSON is internally inconsistent and should be corrected during recreation.

## Prerequisites
1. Prepare an n8n instance with:
   - Postgres access
   - Anthropic credentials
   - Ollama available for local address detection
   - LangChain nodes enabled
2. Create two Postgres tables:
   - `pii_vault`
   - `pii_audit_log`

### Suggested `pii_vault` columns
- `token` text
- `original_value` text
- `type` text
- `document_id` text
- `created_at` timestamp
- optionally `allowed_for_reinjection` boolean
- optionally `pii_type` text

### Suggested `pii_audit_log` columns
- `actor` text
- `timestamp` timestamp
- `document_id` text
- `token_count` integer
- `pii_types_detected` jsonb or text
- `ai_access_confirmed` boolean
- `re_injection_events` jsonb or text

## Build Steps

1. **Create a Webhook node**
   - Name: `Document Upload Webhook`
   - Type: Webhook
   - Method: `POST`
   - Path: `gdpr-document-upload`
   - Response mode: `Last Node`
   - Enable raw body
   - Ensure your upload format provides a binary PDF consumable by the next node

2. **Create a Set node**
   - Name: `Workflow Configuration`
   - Add fields:
     - `documentId` as expression `{{$now.toISO()}}`
     - `confidenceThreshold` as number `0.8`
     - `vaultTable` as string `pii_vault`
     - `auditTable` as string `pii_audit_log`
   - Keep incoming fields

3. **Connect `Document Upload Webhook` → `Workflow Configuration`**

4. **Create an Extract From File node**
   - Name: `OCR Extract Text`
   - Operation: `PDF`
   - Keep source: `both`
   - Confirm it reads the uploaded PDF binary

5. **Connect `Workflow Configuration` → `OCR Extract Text`**

### Build the PII detectors

6. **Create a Code node**
   - Name: `Email Detector`
   - Paste the email detection script
   - Recommended correction: standardize output to `detections` and fields `start`/`end` rather than `start_pos`/`end_pos`

7. **Create a Code node**
   - Name: `Phone Detector`
   - Paste the phone detection script
   - Recommended correction: change output property from `detected_pii` to `detections`
   - Also normalize `start_pos/end_pos` to `start/end`

8. **Create a Code node**
   - Name: `ID Number Detector`
   - Paste the ID detection script
   - Recommended correction: normalize `start_pos/end_pos` to `start/end`

9. **Create an Ollama Chat Model node**
   - Name: `Ollama Chat Model`
   - Configure the local Ollama endpoint and model
   - Pick an instruction-following model appropriate for extraction

10. **Create a Structured Output Parser node**
    - Name: `Address Output Parser`
    - Use a manual schema with an array of address objects
    - Recommended correction: return under `detections` instead of `addresses`, or add a normalization step later

11. **Create an AI Agent node**
    - Name: `Address Detector AI`
    - Input text: `{{$json.text}}`
    - Enable output parser
    - System message: instruct model to detect physical addresses with positions and confidence
    - Attach:
      - `Ollama Chat Model` as language model
      - `Address Output Parser` as parser

12. **Connect `OCR Extract Text` to all four detector branches**
    - `OCR Extract Text` → `Email Detector`
    - `OCR Extract Text` → `Phone Detector`
    - `OCR Extract Text` → `ID Number Detector`
    - `OCR Extract Text` → `Address Detector AI`

13. **Connect `Ollama Chat Model` → `Address Detector AI`**
14. **Connect `Address Output Parser` → `Address Detector AI` via parser connection**

### Merge and consolidate detections

15. **Create a Merge node**
    - Name: `Merge PII Detections`
    - Mode: `Combine`
    - Combine by: `Combine All`

16. **Connect detector outputs into `Merge PII Detections`**
    - Email
    - Phone
    - ID number
    - Address AI

17. **Create a Code node**
    - Name: `PII Consolidation & Conflict Resolver`
    - Paste the consolidation script
    - Required correction:
      - make it read all supported formats
      - map `start_pos/end_pos` to `start/end`
      - include address outputs
      - preserve `originalText`
   - Best target output:
     - `consolidatedPII`
     - `originalText`
     - `documentId`

18. **Connect `Merge PII Detections` → `PII Consolidation & Conflict Resolver`**

### Tokenize and store vault entries

19. **Create a Code node**
    - Name: `Tokenization & Vault Storage`
    - Paste the tokenization script
    - Required correction:
      - make it read `piiMap` or rename prior output to `consolidatedPII`
      - output one item per vault record if you want direct Postgres inserts
   - Recommended token format: keep `<<TYPE_HASH>>`

20. **Connect `PII Consolidation & Conflict Resolver` → `Tokenization & Vault Storage`**

21. **Create a Postgres node**
    - Name: `Store Tokens in Vault`
    - Credentials: your Postgres credential
    - Operation: insert/upsert according to your schema
    - Schema: `public`
    - Table: `{{$('Workflow Configuration').first().json.vaultTable}}`
    - Map columns:
      - `token` → token from tokenization node
      - `original_value` → original detected value
      - `type` → PII type
      - `document_id` → document ID
      - `created_at` → created timestamp
    - Required correction:
      - replace hardcoded `"YOUR_CREDENTIAL_HERE"` with the actual token expression

22. **Connect `Tokenization & Vault Storage` → `Store Tokens in Vault`**

### Generate masked text and enforce safety

23. **Create a Code node**
    - Name: `Generate Masked Text`
    - Paste the masking script
    - Ensure the Postgres node returns inserted rows with `token` and `original_value`, or read from the tokenization node directly instead

24. **Connect `Store Tokens in Vault` → `Generate Masked Text`**

25. **Create an IF node**
    - Name: `Masking Success Check`
    - Condition:
      - left value: `{{$('Generate Masked Text').item.json.masking_success}}`
      - operation: equals
      - right value: `true`

26. **Connect `Generate Masked Text` → `Masking Success Check`**

27. **Create a Set node**
    - Name: `Block AI Processing`
    - Add:
      - `error = "Masking failed - AI processing blocked"`
      - `status = "BLOCKED"`
      - `requires_manual_review = true`
    - Recommended addition:
      - pass through `documentId` and detailed failure reason

28. **Create an HTTP Request node**
    - Name: `Send Alert Notification`
    - Method: `POST`
    - Replace placeholder URL with your incident endpoint
    - Send body:
      - `error_details`
      - `document_id`
      - `timestamp = {{$now.toISO()}}`
    - Recommended correction:
      - point body expressions to actual fields present from previous node

29. **Connect `Masking Success Check` false branch → `Block AI Processing`**
30. **Connect `Block AI Processing` → `Send Alert Notification`**

### Masked AI processing with Claude

31. **Create an Anthropic Chat Model node**
    - Name: `AI Processing Model`
    - Credentials: Anthropic
    - Model: `claude-sonnet-4-5-20250929`
    - If unavailable, choose a currently supported Claude model in your environment

32. **Create a Structured Output Parser node**
    - Name: `AI Output Parser`
    - Configure manual schema with:
      - `documentType`
      - `summary`
      - optional `keyEntities`
      - optional `dates`
      - optional `amounts`
      - optional `processedData`

33. **Create an AI Agent node**
    - Name: `AI Processing (Masked Data)`
    - Text input: `{{$json.masked_text}}`
    - Enable output parser
    - Add system prompt instructing the model to preserve tokens exactly
    - Attach `AI Processing Model`
    - Attach `AI Output Parser`

34. **Connect `Masking Success Check` true branch → `AI Processing (Masked Data)`**
35. **Connect `AI Processing Model` → `AI Processing (Masked Data)`**
36. **Connect `AI Output Parser` → `AI Processing (Masked Data)`**

### Controlled re-injection

37. **Create a Code node**
    - Name: `Re-Injection Controller`
    - Paste the provided script
    - Required correction:
      - token regex must match the token format you actually use
      - if using `<<EMAIL_AB12>>`, update matching accordingly
      - define or inject `fieldPermissions` if re-injection policy is required

38. **Connect `AI Processing (Masked Data)` → `Re-Injection Controller`**

39. **Create a Postgres node**
    - Name: `Retrieve Original Values`
    - Operation: `Select`
    - Schema: `public`
    - Table: `{{$('Workflow Configuration').first().json.vaultTable}}`
    - Query by `token`
    - Required correction:
      - iterate over extracted tokens, not `$('Re-Injection Controller').item.json.token` unless that field is explicitly created

40. **Connect `Re-Injection Controller` → `Retrieve Original Values`**

41. **Create a Code node**
    - Name: `Restore Original PII`
    - Paste the restore script
    - Required correction:
      - token regex must match actual token syntax
      - if approval flags are needed, store `allowed_for_reinjection` in vault or policy config
      - ensure the node receives both AI output and vault query results in a compatible structure

42. **Connect `Retrieve Original Values` → `Restore Original PII`**

### Audit logging

43. **Create a Postgres node**
    - Name: `Store Audit Log`
    - Credentials: Postgres
    - Schema: `public`
    - Table: `{{$('Workflow Configuration').first().json.auditTable}}`
    - Map:
      - `actor = "system"`
      - `timestamp`
      - `document_id`
      - `token_count`
      - `pii_types_detected`
      - `ai_access_confirmed = true`
      - `re_injection_events`
    - Required correction:
      - ensure prior node outputs these fields or derive them before insertion

44. **Connect `Restore Original PII` → `Store Audit Log`**

### Add the documentation notes

45. **Create Sticky Notes** matching the original sections:
    - GDPR-Safe AI Document Processing
    - Document Upload
    - OCR Text Extraction
    - PII Detection Layer
    - Address Detection (AI) local ollama
    - Merge Detection Results
    - IResolve Overlapping Detections
    - Tokenization & Vault Storage
    - IMasking Safety Check
    - AI Processing (Masked Data)
    - PII Re-Injection Controller
    - Restore Original Values
    - Compliance Audit Log

## Important implementation corrections
To make this workflow actually operate end-to-end, apply these fixes during reproduction:

1. **Normalize all detector outputs**
   - Use the same output key, preferably `detections`
   - Use the same positional fields, preferably `start` and `end`

2. **Align consolidation output with tokenization input**
   - Either:
     - change consolidation output to `consolidatedPII`
   - or:
     - change tokenization node to read `piiMap`

3. **Fix vault insertion**
   - Map `token` to the actual token value
   - Ensure one database row is produced per vault record

4. **Unify token format everywhere**
   - Current workflow uses conflicting formats:
     - `<<TYPE_HASH>>`
     - `TOKEN_TYPE_ID`
     - `[TOKEN_X]`
   - Pick one and update:
     - token generation
     - AI prompt examples
     - re-injection token extraction
     - restore regex
     - Postgres lookup

5. **Provide field-level re-injection policy**
   - `fieldPermissions` is referenced but never defined
   - Add a Set or Code node before re-injection if this policy matters

6. **Ensure audit fields exist**
   - Add a transformation node before `Store Audit Log` if needed

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| GDPR-Safe AI Document Processing: This workflow processes uploaded documents while protecting sensitive personal data. When a PDF is uploaded, OCR extracts the text and multiple detectors identify Personally Identifiable Information (PII) such as emails, phone numbers, ID numbers, and addresses. Detected PII is consolidated and replaced with secure tokens while the original values are stored in a Postgres vault. The AI model only processes the masked version of the document, ensuring sensitive information is never exposed. If required, a controlled re-injection mechanism can restore original values from the vault. All masking, AI access, and restoration events are recorded in an audit log. | Workflow purpose |
| Setup: Configure Postgres credentials. Create `pii_vault` and `pii_audit_log` tables. Connect an AI model. Send documents to the webhook. | Deployment/setup |
| Address Detection (AI) local ollama: An AI model analyzes the OCR text to detect physical addresses that are harder to capture with regex patterns. | Local Ollama integration |
| Example token shown in note: `<<EMAIL_AB12>>` | Tokenization convention shown in design notes |
| The workflow contains no sub-workflow nodes and only one explicit entry point: `Document Upload Webhook`. | Architecture note |
| The current JSON is conceptually strong but operationally inconsistent; normalization of detector outputs and token formats is required for reliable production use. | Implementation note |