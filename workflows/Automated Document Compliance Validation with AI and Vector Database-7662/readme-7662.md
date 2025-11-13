Automated Document Compliance Validation with AI and Vector Database

https://n8nworkflows.xyz/workflows/automated-document-compliance-validation-with-ai-and-vector-database-7662


# Automated Document Compliance Validation with AI and Vector Database

### 1. Workflow Overview

This workflow automates the validation of document compliance using AI and a vector database for semantic search. It targets organizations needing to audit, validate, and report on compliance of internal procedures against uploaded documents, such as policies, reports, or manuals. The workflow integrates document ingestion, preprocessing, embedding generation, vector storage, semantic retrieval, and AI-driven compliance analysis, culminating in structured compliance reports delivered via webhooks.

Logical blocks:

- **1.1 Document Upload & Fetch**: Receives uploaded audit documents via webhook, optionally fetching files from Microsoft Graph.
- **1.2 Document Preprocessing & Vector Management**: Deletes previous embeddings for the same document, extracts text from PDFs, splits text into chunks, generates embeddings, and inserts them into the Qdrant vector database.
- **1.3 Procedure Submission**: Receives the procedure to be validated along with metadata via webhook, formats the payload for AI processing.
- **1.4 AI Compliance Validation**: Uses the procedure input to generate search queries, retrieves relevant document chunks from Qdrant, and applies an AI agent (LLM) to analyze compliance, parsing outputs into structured JSON.
- **1.5 Return Results**: Sends the structured compliance report back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Document Upload & Fetch

- **Overview:** This block handles receiving audit documents uploaded via webhook and fetching the actual file content from Microsoft Graph if IDs are provided.
- **Nodes Involved:**  
  - Audit Document Upload (Webhook)  
  - Fetch Document (Microsoft Graph HTTP Request)

- **Node Details:**

  - **Audit Document Upload**  
    - Type: Webhook  
    - Role: Entry point for document upload requests (HTTP POST at path `/creatorhub/audit-document-upload`)  
    - Config: Listens for POST requests, responds with last node output  
    - Inputs: External HTTP request with JSON payload including SharePoint drive and document IDs  
    - Outputs: Passes payload downstream  
    - Edge cases: Missing or invalid SharePoint IDs, network errors fetching documents  

  - **Fetch Document (Microsoft Graph)**  
    - Type: HTTP Request  
    - Role: Downloads document content from Microsoft Graph given SharePoint Drive and Document ID  
    - Config: URL dynamically constructed from environment variable `GRAPH_BASE_URL` and incoming JSON parameters; expects binary content response  
    - Inputs: JSON body from webhook node  
    - Outputs: Binary document data  
    - Edge cases: Authentication/authorization failures with Graph API, file not found, timeout  

#### 1.2 Document Preprocessing & Vector Management

- **Overview:** Deletes old vector embeddings related to the uploaded document, extracts text from the PDF, splits text into chunks, generates embeddings, and inserts them into the Qdrant vector store.
- **Nodes Involved:**  
  - Delete Old Document Vectors (Code)  
  - Extract PDF Text  
  - Split Text into Chunks  
  - Load Document Metadata  
  - Generate Document Embeddings  
  - Insert Vectors into Qdrant

- **Node Details:**

  - **Delete Old Document Vectors**  
    - Type: Code (LangChain)  
    - Role: Connects to Qdrant and Ollama embedding services; deletes existing vectors tagged with the current document's file ID to avoid duplication  
    - Config: Uses environment variables for Ollama and Qdrant URLs, collection name, and embedding model; constructs a filter to delete vectors with matching metadata file_id  
    - Inputs: JSON including `spDocumentId` from webhook  
    - Outputs: Passes along input with added file_id metadata  
    - Edge cases: Qdrant API failures, network issues; non-fatal as process continues even if deletion fails  

  - **Extract PDF Text**  
    - Type: Extract From File  
    - Role: Extracts text content from the binary PDF data  
    - Config: Operates on binary property `data`, outputs extracted text  
    - Inputs: Binary PDF content from previous node  
    - Outputs: Text content for splitting  
    - Edge cases: Unsupported file formats, corrupt PDFs  

  - **Split Text into Chunks**  
    - Type: Recursive Character Text Splitter (LangChain)  
    - Role: Splits extracted text into overlapping chunks (overlap=10 characters) suitable for embedding generation  
    - Inputs: Extracted text  
    - Outputs: Text chunks with overlap  
    - Edge cases: Text too short or empty  

  - **Load Document Metadata**  
    - Type: Document Default Data Loader (LangChain)  
    - Role: Attaches metadata (documentId, documentName) to text chunks before embedding  
    - Config: Metadata values dynamically set from prior node outputs  
    - Inputs: Text chunks from splitter  
    - Outputs: Document chunks with metadata  
    - Edge cases: Missing metadata values  

  - **Generate Document Embeddings**  
    - Type: Embeddings Ollama (LangChain)  
    - Role: Generates vector embeddings for text chunks using Ollama model  
    - Config: Model name from environment or default `nomic-embed-text:latest`  
    - Credentials: Ollama API  
    - Inputs: Document chunks with metadata  
    - Outputs: Embeddings vectors  
    - Edge cases: API rate limits, model unavailability  

  - **Insert Vectors into Qdrant**  
    - Type: Vector Store Qdrant (LangChain)  
    - Role: Inserts newly generated vectors into the Qdrant collection  
    - Config: Insert mode, collection name from environment or default `audit-docs`  
    - Credentials: Qdrant API  
    - Inputs: Embeddings vectors  
    - Outputs: Confirmation of vector insertion  
    - Edge cases: Qdrant connectivity issues, collection not found  

#### 1.3 Procedure Submission

- **Overview:** Receives procedure-related information via webhook, formats it, and passes it downstream for AI compliance validation.
- **Nodes Involved:**  
  - Procedure Submission (Webhook)  
  - Format Procedure Payload (Code)

- **Node Details:**

  - **Procedure Submission**  
    - Type: Webhook  
    - Role: Accepts POST requests with JSON containing procedures, `spDocumentId`, and description  
    - Config: Path `/creatorhub/procedure-validate`, response mode set to respond with node output  
    - Inputs: JSON payload from external caller  
    - Outputs: Raw JSON to formatter node  
    - Edge cases: Invalid JSON, missing fields  

  - **Format Procedure Payload**  
    - Type: Code  
    - Role: Transforms incoming JSON to an array of individual procedure payloads for AI processing  
    - Config: Maps `procedures` array to separate JSON objects each containing `procedure`, `spDocumentId`, and `description`  
    - Inputs: JSON from webhook  
    - Outputs: Array of formatted procedure objects  
    - Edge cases: Empty or missing `procedures` array  

#### 1.4 AI Compliance Validation

- **Overview:** For each procedure, this block retrieves relevant document chunks from Qdrant, queries the AI agent to analyze compliance, and parses the AI output into structured JSON.
- **Nodes Involved:**  
  - AI Compliance Validator (LangChain Agent)  
  - Retrieve Relevant Document Chunks (Qdrant Vector Store)  
  - Generate Query Embeddings (Ollama Embeddings)  
  - Language Model (AI Agent) (Ollama Chat)  
  - Language Model (Structured Output) (Ollama Chat)  
  - Parse AI Response (Structured Output Parser)

- **Node Details:**

  - **AI Compliance Validator**  
    - Type: LangChain Agent  
    - Role: Core logic node defining the AI prompt, inputs, and output parsing for compliance validation  
    - Config: Detailed prompt embedding role, inputs (`procedure`, `spDocumentId`, `description`), task steps, output schema, and examples  
    - Inputs: Formatted procedures from previous step  
    - Outputs: Structured JSON compliance report  
    - Edge cases: Incomplete AI responses, prompt failures, API limits  
    - Note: Uses LLM + vector search results as tools internally  

  - **Retrieve Relevant Document Chunks**  
    - Type: Vector Store Qdrant (retrieve-as-tool mode)  
    - Role: Retrieves top 8 relevant text chunks filtered by `documentId` matching current procedure's `spDocumentId`  
    - Config: Uses JSON filter to restrict vector search to specific document  
    - Credentials: Qdrant API  
    - Inputs: Query embeddings from Generate Query Embeddings node  
    - Outputs: Relevant text chunks for AI analysis  
    - Edge cases: No matching vectors found, Qdrant errors  

  - **Generate Query Embeddings**  
    - Type: Embeddings Ollama  
    - Role: Generates embeddings for the procedure query text to perform vector similarity search  
    - Config: Model from environment variables  
    - Credentials: Ollama API  
    - Inputs: Procedure text from Format Procedure Payload (implicit via AI agent)  
    - Outputs: Query embeddings  
    - Edge cases: API failures  

  - **Language Model (AI Agent)**  
    - Type: LM Chat Ollama  
    - Role: Runs primary compliance AI analysis using the LLM model specified in environment (default `qwen2.5:7b`)  
    - Credentials: Ollama API  
    - Inputs: Procedures and retrieved document chunks via AI agent orchestration  
    - Outputs: Raw AI response text  
    - Edge cases: Model timeouts, generation errors  

  - **Language Model (Structured Output)**  
    - Type: LM Chat Ollama  
    - Role: Formats and structures AI raw output into JSON matching the compliance schema  
    - Credentials: Ollama API  
    - Inputs: AI agent's unstructured output  
    - Outputs: JSON string for parsing  
    - Edge cases: Parsing errors, incomplete output  

  - **Parse AI Response**  
    - Type: Output Parser Structured (LangChain)  
    - Role: Parses JSON string output into structured n8n JSON data  
    - Config: Schema validates presence of `confidenceLevel`, `summaryOfCompliance`, `summaryOfNonCompliance`, and `supportingTextCitations`  
    - Inputs: Output from structured LM node  
    - Outputs: Parsed structured compliance report  
    - Edge cases: Schema validation failures, auto-fix enabled to correct minor errors  

#### 1.5 Return Results

- **Overview:** Returns the structured compliance report to the original procedure submission webhook caller.
- **Nodes Involved:**  
  - Return Compliance Report (Respond to Webhook)

- **Node Details:**

  - **Return Compliance Report**  
    - Type: Respond to Webhook  
    - Role: Sends back all incoming items (compliance reports) as HTTP response  
    - Inputs: Parsed structured compliance JSON  
    - Outputs: HTTP response to external caller  
    - Edge cases: Webhook timeouts, large payload handling  

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                                  | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                           |
|----------------------------|------------------------------------|-------------------------------------------------|---------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------|
| Audit Document Upload       | Webhook                            | Entry point for audit document upload            | -                               | Fetch Document (Microsoft Graph)  | ### 1. Start: Upload Document * Via Webhook: Audit Document Upload * Accepts PDF/DOCX file * Optionally fetches from Microsoft Graph                  |
| Fetch Document (Microsoft Graph) | HTTP Request                    | Downloads document content from Microsoft Graph  | Audit Document Upload            | Delete Old Document Vectors       | ### 1. Start: Upload Document * Via Webhook: Audit Document Upload * Accepts PDF/DOCX file * Optionally fetches from Microsoft Graph                  |
| Delete Old Document Vectors | Code (LangChain)                   | Deletes old vector embeddings for uploaded file  | Fetch Document (Microsoft Graph) | Extract PDF Text                 | ### 2. Document Preprocessing * Clear Old Vectors (remove previous embeddings for same file) * Extract PDF Text * Split into chunks for embedding * Generate embeddings → Insert into Qdrant |
| Extract PDF Text            | Extract From File                  | Extracts text from PDF                            | Delete Old Document Vectors      | Insert Vectors into Qdrant        | ### 2. Document Preprocessing * Clear Old Vectors (remove previous embeddings for same file) * Extract PDF Text * Split into chunks for embedding * Generate embeddings → Insert into Qdrant |
| Split Text into Chunks      | Recursive Character Text Splitter | Splits text into overlapping chunks for embedding | Insert Vectors into Qdrant (via Load Document Metadata) | Load Document Metadata            | ### 2. Document Preprocessing * Clear Old Vectors (remove previous embeddings for same file) * Extract PDF Text * Split into chunks for embedding * Generate embeddings → Insert into Qdrant |
| Load Document Metadata      | Document Default Data Loader       | Attaches metadata to text chunks                  | Split Text into Chunks           | Insert Vectors into Qdrant        | ### 2. Document Preprocessing * Clear Old Vectors (remove previous embeddings for same file) * Extract PDF Text * Split into chunks for embedding * Generate embeddings → Insert into Qdrant |
| Generate Document Embeddings| Embeddings Ollama (LangChain)     | Generates vector embeddings for text chunks      | Load Document Metadata           | Insert Vectors into Qdrant        | ### 2. Document Preprocessing * Clear Old Vectors (remove previous embeddings for same file) * Extract PDF Text * Split into chunks for embedding * Generate embeddings → Insert into Qdrant |
| Insert Vectors into Qdrant  | Vector Store Qdrant (LangChain)   | Inserts embeddings into Qdrant collection         | Generate Document Embeddings, Load Document Metadata | Split Text into Chunks (for indexing) | ### 2. Document Preprocessing * Clear Old Vectors (remove previous embeddings for same file) * Extract PDF Text * Split into chunks for embedding * Generate embeddings → Insert into Qdrant |
| Procedure Submission        | Webhook                           | Accepts procedure validation requests             | -                               | Format Procedure Payload          | ### 3. Procedure Submission * Webhook: Procedure Submission * Accepts JSON payload (procedure, description, spDocumentId) * Payload formatted → passed to AI        |
| Format Procedure Payload    | Code                             | Formats procedure data array for AI processing    | Procedure Submission            | AI Compliance Validator           | ### 3. Procedure Submission * Webhook: Procedure Submission * Accepts JSON payload (procedure, description, spDocumentId) * Payload formatted → passed to AI        |
| AI Compliance Validator     | LangChain Agent                  | Orchestrates AI compliance analysis               | Format Procedure Payload        | Return Compliance Report          | ### 4. AI Compliance Validation * Retrieve Relevant Document Chunks from Qdrant * AI Compliance Validator uses LLM + embeddings * Output parsed & structured into JSON |
| Retrieve Relevant Document Chunks | Vector Store Qdrant (LangChain) | Retrieves relevant document chunks for AI query   | Generate Query Embeddings       | AI Compliance Validator           | ### 4. AI Compliance Validation * Retrieve Relevant Document Chunks from Qdrant * AI Compliance Validator uses LLM + embeddings * Output parsed & structured into JSON |
| Generate Query Embeddings   | Embeddings Ollama (LangChain)   | Generates embeddings for procedure query           | Format Procedure Payload        | Retrieve Relevant Document Chunks | ### 4. AI Compliance Validation * Retrieve Relevant Document Chunks from Qdrant * AI Compliance Validator uses LLM + embeddings * Output parsed & structured into JSON |
| Language Model (AI Agent)   | LM Chat Ollama (LangChain)       | Runs main AI compliance analysis                    | Generate Query Embeddings, Retrieve Relevant Document Chunks | AI Compliance Validator           | ### 4. AI Compliance Validation * Retrieve Relevant Document Chunks from Qdrant * AI Compliance Validator uses LLM + embeddings * Output parsed & structured into JSON |
| Language Model (Structured Output) | LM Chat Ollama (LangChain)  | Formats AI output into structured JSON              | AI Compliance Validator        | Parse AI Response                 | ### 4. AI Compliance Validation * Retrieve Relevant Document Chunks from Qdrant * AI Compliance Validator uses LLM + embeddings * Output parsed & structured into JSON |
| Parse AI Response           | Output Parser Structured (LangChain) | Parses and validates AI output JSON                  | Language Model (Structured Output) | AI Compliance Validator          | ### 4. AI Compliance Validation * Retrieve Relevant Document Chunks from Qdrant * AI Compliance Validator uses LLM + embeddings * Output parsed & structured into JSON |
| Return Compliance Report    | Respond to Webhook               | Returns compliance report to webhook caller         | AI Compliance Validator        | -                                | ### 5. Return Results * Structured compliance report returned to webhook caller                                               |
| Sticky Note                | Sticky Note                      | Notes annotations on workflow sections              | -                               | -                                | See node specific sticky notes above                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Audit Document Upload**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `creatorhub/audit-document-upload`  
   - Response Mode: Last Node  
   - Purpose: Receive audit document upload requests with JSON body containing `spDriveId` and `spDocumentId`.

2. **Create HTTP Request Node: Fetch Document (Microsoft Graph)**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `{{$env.GRAPH_BASE_URL || "https://graph.microsoft.com"}}/v1.0/drives/{{$json.body.spDriveId}}/items/{{$json.body.spDocumentId}}/content`  
   - Purpose: Download document binary content from Microsoft Graph using IDs from webhook.  
   - Connect Audit Document Upload → Fetch Document.

3. **Create Code Node: Delete Old Document Vectors**  
   - Type: Code (LangChain)  
   - Paste the provided JavaScript code that deletes existing vectors with metadata matching `spDocumentId`.  
   - Use environment variables: `OLLAMA_BASE_URL`, `QDRANT_BASE_URL`, `QDRANT_COLLECTION`, `OLLAMA_EMBED_MODEL`.  
   - Connect Fetch Document → Delete Old Document Vectors.

4. **Create Extract From File Node: Extract PDF Text**  
   - Type: Extract From File  
   - Operation: PDF  
   - Binary Property Name: `data` (from HTTP Request)  
   - Connect Delete Old Document Vectors → Extract PDF Text.

5. **Create Recursive Character Text Splitter Node: Split Text into Chunks**  
   - Type: Recursive Character Text Splitter (LangChain)  
   - Chunk Overlap: 10 characters  
   - Connect Extract PDF Text → Split Text into Chunks.

6. **Create Document Default Data Loader Node: Load Document Metadata**  
   - Type: Document Default Data Loader (LangChain)  
   - Metadata Values:  
     - `documentId` = `={{ $('Delete Old Document Vectors').item.json.body.spDocumentId }}`  
     - `documentName` = `={{ $('Delete Old Document Vectors').item.json.body.fileName }}` (if available)  
   - Text Splitting Mode: Custom  
   - Connect Split Text into Chunks → Load Document Metadata.

7. **Create Embeddings Ollama Node: Generate Document Embeddings**  
   - Type: Embeddings Ollama (LangChain)  
   - Model: `={{ $env.OLLAMA_EMBED_MODEL || "nomic-embed-text:latest" }}`  
   - Credentials: Ollama API  
   - Connect Load Document Metadata → Generate Document Embeddings.

8. **Create Vector Store Qdrant Node: Insert Vectors into Qdrant**  
   - Type: Vector Store Qdrant (LangChain)  
   - Mode: Insert  
   - Collection Name: `={{ $env.QDRANT_COLLECTION || "audit-docs" }}`  
   - Credentials: Qdrant API  
   - Connect Generate Document Embeddings → Insert Vectors into Qdrant.  
   - Also connect Load Document Metadata → Insert Vectors into Qdrant (as ai_document input).

9. **Create Webhook Node: Procedure Submission**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `creatorhub/procedure-validate`  
   - Response Mode: Response Node  
   - Purpose: Receive procedure JSON payload including `procedures` array, `spDocumentId`, and `description`.

10. **Create Code Node: Format Procedure Payload**  
    - Type: Code  
    - JavaScript: Extracts `procedures` array and maps each to separate JSON object with `procedure`, `spDocumentId`, and `description`.  
    - Connect Procedure Submission → Format Procedure Payload.

11. **Create LangChain Agent Node: AI Compliance Validator**  
    - Type: LangChain Agent  
    - Prompt: Use detailed prompt embedding role, tasks, examples as provided.  
    - Inputs: `procedure`, `spDocumentId`, `description`.  
    - Connect Format Procedure Payload → AI Compliance Validator.

12. **Create Embeddings Ollama Node: Generate Query Embeddings**  
    - Type: Embeddings Ollama (LangChain)  
    - Model: `={{ $env.OLLAMA_EMBED_MODEL || "nomic-embed-text:latest" }}`  
    - Credentials: Ollama API  
    - Connect Format Procedure Payload → Generate Query Embeddings.

13. **Create Vector Store Qdrant Node: Retrieve Relevant Document Chunks**  
    - Type: Vector Store Qdrant (LangChain)  
    - Mode: Retrieve-as-tool  
    - Top K: 8  
    - Search Filter JSON:  
      ```json
      {
        "should": [
          {
            "key": "metadata.documentId",
            "match": { "value": "{{ $('Format Procedure Payload').first().json.spDocumentId }}" }
          }
        ]
      }
      ```  
    - Credentials: Qdrant API  
    - Connect Generate Query Embeddings → Retrieve Relevant Document Chunks.

14. **Create LM Chat Ollama Node: Language Model (AI Agent)**  
    - Type: LM Chat Ollama  
    - Model: `={{ $env.OLLAMA_CHAT_MODEL || "qwen2.5:7b" }}`  
    - Options: numCtx=2048 tokens  
    - Credentials: Ollama API  
    - Connect Retrieve Relevant Document Chunks → AI Compliance Validator (ai_tool input) and connect AI Compliance Validator → Language Model (AI Agent) (ai_languageModel input).

15. **Create LM Chat Ollama Node: Language Model (Structured Output)**  
    - Type: LM Chat Ollama  
    - Model: `={{ $env.OLLAMA_CHAT_MODEL || "qwen2.5:7b" }}`  
    - Credentials: Ollama API  
    - Connect AI Compliance Validator → Language Model (Structured Output) (ai_languageModel input).

16. **Create Output Parser Structured Node: Parse AI Response**  
    - Type: Output Parser Structured (LangChain)  
    - Schema:  
      ```json
      {
        "type": "object",
        "required": ["confidenceLevel","summaryOfCompliance","summaryOfNonCompliance","supportingTextCitations"],
        "properties": {
          "procedure": {"type": "string"},
          "spDocumentId": {"type": "string"},
          "confidenceLevel": {"type": "integer"},
          "summaryOfCompliance": {"type": "string"},
          "summaryOfNonCompliance": {"type": "string"},
          "supportingTextCitations": {"type": "string"}
        }
      }
      ```  
    - Auto-fix: Enabled  
    - Connect Language Model (Structured Output) → Parse AI Response.

17. **Connect Parse AI Response → AI Compliance Validator (ai_outputParser input).**

18. **Create Respond to Webhook Node: Return Compliance Report**  
    - Type: Respond to Webhook  
    - Respond With: All Incoming Items  
    - Connect AI Compliance Validator → Return Compliance Report.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| The workflow uses Ollama API for both embeddings and language model chat completions, requiring valid API credentials configured within n8n.                   | Ollama API account credentials                                           |
| Qdrant vector store is used for vector insertion and retrieval; make sure Qdrant instance and collection (`audit-docs`) exist and are accessible.                | Qdrant vector database setup                                             |
| The AI Compliance Validator prompt is designed to produce structured JSON output with confidence scoring, compliance summaries, and citations for traceability. | Compliance analysis prompt embedded in AI Compliance Validator node     |
| Webhook endpoints `/creatorhub/audit-document-upload` and `/creatorhub/procedure-validate` are the external API interfaces for document upload and validation.  | Webhook endpoint paths                                                   |
| Sticky notes in the workflow provide high-level grouping and annotation to clarify functional blocks and flow.                                                 | Visible in n8n editor as workflow annotations                           |
| The workflow assumes PDF document format for text extraction; other formats like DOCX are accepted at upload but may require additional extraction logic.        | Extraction node configured for PDF only                                 |
| Use environment variables for all external service URLs and model names to simplify configuration and portability.                                             | Environment variables: `GRAPH_BASE_URL`, `OLLAMA_BASE_URL`, `QDRANT_BASE_URL`, `QDRANT_COLLECTION`, `OLLAMA_EMBED_MODEL`, `OLLAMA_CHAT_MODEL` |

---

*Disclaimer: The text provided is sourced exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*