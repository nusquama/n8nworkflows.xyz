Answer internal policy questions from Google Drive with OpenAI and Qdrant

https://n8nworkflows.xyz/workflows/answer-internal-policy-questions-from-google-drive-with-openai-and-qdrant-19671


# Answer internal policy questions from Google Drive with OpenAI and Qdrant

### 1. Workflow Overview

This workflow synchronizes internal company documentation from a monitored Google Drive folder into a Qdrant vector database and serves as an intelligent support agent answering user questions via n8n Chat or Slack. It features automated file ingestion, type-based text extraction, vector synchronization, and an AI-driven retrieval-augmented generation (RAG) agent equipped with conversation memory and source citation formatting.

The logic is grouped into the following functional blocks:
- **1.1 File Intake and Merging:** Watches Google Drive for newly created or updated files, downloads them, and consolidates the streams.
- **1.2 MIME-Type Routing and Text Extraction:** Evaluates incoming file formats (PDF, Google Docs, TXT, DOCX) and executes format-specific text extraction (including a cloud conversion pattern for DOCX files).
- **1.3 Vector Synchronization and Qdrant Ingestion:** Removes stale vector embeddings matching the target file ID and indexes newly chunked and embedded document text into Qdrant.
- **1.4 Question Intake and Normalization:** Receives messages from Slack or the n8n Chat interface, filters out system/bot events, and standardizes payload fields.
- **1.5 AI Retrieval and Response Generation:** Processes questions through an OpenAI-powered AI Agent using windowed conversation memory and a Qdrant knowledge-base tool.
- **1.6 Output Formatting and Delivery:** Formats document source citations into markdown or Slack link markup and conditionally routes responses back to Slack channels.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: File Intake and Merging
- **Overview:** Monitors a designated Google Drive folder for file additions or modifications, pulling down binary content for processing.
- **Nodes Involved:** 
  - `When File Created in Drive`
  - `Download Created File`
  - `When File Updated in Drive`
  - `Download Updated File`
  - `Merge Files for Processing`

- **Node Details:**
  - **When File Created in Drive**
    - *Type and Technical Role:* `n8n-nodes-base.googleDriveTrigger` — Triggers execution every minute when a file is created inside the configured target folder.
    - *Configuration Choices:* Watches folder ID `1OMm12f0vxmfuLjM0D0Fq2bFzR20qP37m` on a 1-minute polling interval.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Download Created File`.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* API rate limits, revoked Google OAuth credentials, or folder permission losses.
  - **Download Created File**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` — Downloads binary data of the created file.
    - *Configuration Choices:* Operation set to `download` using item ID expression; converts Google native docs to plain text.
    - *KeyExpressions/Variables:* `={{ $json.id }}`
    - *Input/Output:* Input from `When File Created in Drive`; output connected to `Merge Files for Processing` (index 0).
    - *Version Requirements:* Type Version 3.
    - *Edge Cases/Failures:* Large file download timeouts or missing storage permissions.
  - **When File Updated in Drive**
    - *Type and Technical Role:* `n8n-nodes-base.googleDriveTrigger` — Triggers every minute when an existing file in the target folder is modified.
    - *Configuration Choices:* Watches folder ID `1OMm12f0vxmfuLjM0D0Fq2bFzR20qP37m` on a 1-minute polling interval for `fileUpdated` events.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Download Updated File`.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Rapid successive updates causing overlapping workflow executions.
  - **Download Updated File**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` — Downloads binary content for updated files.
    - *Configuration Choices:* Operation set to `download` with Google file conversion options enabled.
    - *Key Expressions/Variables:* `={{ $json.id }}`
    - *Input/Output:* Input from `When File Updated in Drive`; output connected to `Merge Files for Processing` (index 1).
    - *Version Requirements:* Type Version 3.
    - *Edge Cases/Failures:* File deleted between trigger event and download execution.
  - **Merge Files for Processing**
    - *Type and Technical Role:* `n8n-nodes-base.merge` — Combines the created and updated file streams into a single output pipeline.
    - *Configuration Choices:* Default append/merge mode.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs from `Download Created File` and `Download Updated File`; output connected to `Route by File Type`.
    - *Version Requirements:* Type Version 3.
    - *Edge Cases/Failures:* Unhandled MIME streams passing through if upstream filters fail.

---

#### Block 1.2: MIME-Type Routing and Text Extraction
- **Overview:** Inspects document MIME types and directs files through dedicated extraction paths or temporary cloud conversion routines.
- **Nodes Involved:**
  - `Route by File Type`
  - `Extract Text from PDF`
  - `Extract Text from Google Doc`
  - `Extract Text from TXT`
  - `Convert DOCX to Google Doc`
  - `Export GDoc to Text`
  - `Remove Temporary Google Doc`
  - `Text Extraction Output`

- **Node Details:**
  - **Route by File Type**
    - *Type and Technical Role:* `n8n-nodes-base.switch` — Routes items conditionally based on document MIME type.
    - *Configuration Choices:* Evaluates `$binary.data.mimeType` or `$json.mimeType` against PDF (`application/pdf`), Google Doc (`application/vnd.google-apps.document`), DOCX (`application/vnd.openxmlformats-officedocument.wordprocessingml.document`), and TXT (`text/plain`).
    - *Key Expressions/Variables:* `={{ $binary.data.mimeType }}`, `={{ $json.mimeType }}`
    - *Input/Output:* Input from `Merge Files for Processing`; outputs connect to respective extraction nodes.
    - *Version Requirements:* Type Version 3.
    - *Edge Cases/Failures:* Unsupported MIME types result in unrouted items dropped by the switch.
  - **Extract Text from PDF**
    - *Type and Technical Role:* `n8n-nodes-base.extractFromFile` — Extracts text strings from PDF binary payloads.
    - *Configuration Choices:* Operation set to `pdf`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Input from `Route by File Type`; output connected to `Text Extraction Output`.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Scanned PDFs lacking an OCR text layer yield empty output.
  - **Extract Text from Google Doc**
    - *Type and Technical Role:* `n8n-nodes-base.extractFromFile` — Extracts text from native Google document formats.
    - *Configuration Choices:* Operation set to `text`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Input from `Route by File Type`; output connected to `Text Extraction Output`.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Permission restrictions on shared Google workspace files.
  - **Extract Text from TXT**
    - *Type and Technical Role:* `n8n-nodes-base.extractFromFile` — Reads plain text file contents.
    - *Configuration Choices:* Operation set to `text`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Input from `Route by File Type`; output connected to `Text Extraction Output`.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Character encoding mismatches (e.g., non-UTF-8 files).
  - **Convert DOCX to Google Doc**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Makes a REST call to Google Drive API to copy a DOCX file and convert it into a temporary Google Doc.
    - *Configuration Choices:* POST request to `https://www.googleapis.com/drive/v3/files/{fileId}/copy` with a JSON body setting `mimeType` to `application/vnd.google-apps.document`.
    - *Key Expressions/Variables:* `={{ "https://www.googleapis.com/drive/v3/files/" + ($('When File Created in Drive').isExecuted ? $('When File Created in Drive').item.json.id : $('When File Updated in Drive').item.json.id) + "/copy" }}`
    - *Input/Output:* Input from `Route by File Type`; output connected to `Export GDoc to Text`.
    - *Version Requirements:* Type Version 4.5.
    - *Edge Cases/Failures:* API quota exhaustion or invalid OAuth scopes (`drive.file`).
  - **Export GDoc to Text**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Exports the temporary Google Doc as plain text.
    - *Configuration Choices:* GET request to Google Drive API export endpoint with response format set to text.
    - *Key Expressions/Variables:* `={{ "https://www.googleapis.com/drive/v3/files/" + $('Convert DOCX to Google Doc').item.json.id + "/export?mimeType=text/plain" }}`
    - *Input/Output:* Input from `Convert DOCX to Google Doc`; outputs connected to `Remove Temporary Google Doc` and `Text Extraction Output`.
    - *Version Requirements:* Type Version 4.5.
    - *Edge Cases/Failures:* Export timeout on exceptionally large documents.
  - **Remove Temporary Google Doc**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Deletes the temporary Google Doc created during conversion to prevent clutter.
    - *Configuration Choices:* DELETE request to Google Drive API file endpoint.
    - *Key Expressions/Variables:* `={{ "https://www.googleapis.com/drive/v3/files/" + $('Convert DOCX to Google Doc').item.json.id }}`
    - *Input/Output:* Input from `Export GDoc to Text`; no subsequent downstream connections.
    - *Version Requirements:* Type Version 4.5.
    - *Edge Cases/Failures:* Fails silently if the file was already removed, though execution continues.
  - **Text Extraction Output**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` — Acts as a consolidation point for all extracted text paths before vector operations.
    - *Configuration Choices:* Pass-through node.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs from PDF, Google Doc, TXT, and DOCX export paths; output connected to `Delete Vectors by File ID`.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* None.

---

#### Block 1.3: Vector Synchronization and Qdrant Ingestion
- **Overview:** Purges outdated vector records belonging to the updated file ID from Qdrant, splits the fresh text into chunks, computes embeddings, and stores them in the vector database.
- **Nodes Involved:**
  - `Delete Vectors by File ID`
  - `Insert into Qdrant Vector Store`
  - `Generate OpenAI Embeddings`
  - `Load Default Data`
  - `Split Text Recursively`

- **Node Details:**
  - **Delete Vectors by File ID**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Sends a delete-by-filter payload to Qdrant REST API to remove existing vector points associated with the document's `file_id`.
    - *Configuration Choices:* POST request to Qdrant points delete endpoint using pre-defined Qdrant REST API credentials.
    - *Key Expressions/Variables:* Evaluates trigger execution state to extract the correct file ID: `={{ JSON.stringify({ filter: { must: [{ key: "metadata.file_id", match: { value: $('When File Created in Drive').isExecuted ? $('When File Created in Drive').item.json.id : $('When File Updated in Drive').item.json.id } }] } }) }}`
    - *Input/Output:* Input from `Text Extraction Output`; output connected to `Insert into Qdrant Vector Store`.
    - *Version Requirements:* Type Version 4.5.
    - *Edge Cases/Failures:* Missing payload index on `metadata.file_id` in Qdrant will cause query rejections.
  - **Insert into Qdrant Vector Store**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.vectorStoreQdrant` — Ingests text chunks and embeddings into the Qdrant vector database collection.
    - *Configuration Choices:* Mode set to `insert` on collection `company_docs`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs from `Delete Vectors by File ID`, `Generate OpenAI Embeddings`, `Load Default Data`; no downstream connections.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Qdrant connection timeouts or schema validation errors.
  - **Generate OpenAI Embeddings**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — Computes vector embeddings for document chunks during ingestion.
    - *Configuration Choices:* Model set to `text-embedding-3-small`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Insert into Qdrant Vector Store` via AI embedding connection.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* OpenAI API rate limits or quota exceeded errors.
  - **Load Default Data**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` — Loads extracted text data and binds metadata fields (`file_id`, `file_name`, `drive_url`, `last_modified`) to vector payloads.
    - *Configuration Choices:* JSON mode set to expression data pulling from `Text Extraction Output`.
    - *Key Expressions/Variables:* Dynamic bindings referencing file metadata attributes from Drive triggers.
    - *Input/Output:* Input from `Split Text Recursively` (text splitter connection); output connected to `Insert into Qdrant Vector Store`.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Missing text attributes resulting in empty document payloads.
  - **Split Text Recursively**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter` — Splits long documents into manageable chunks.
    - *Configuration Choices:* Chunk overlap configured to 200 characters.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Load Default Data` via AI text splitter connection.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Extremely large files causing memory spikes during recursive splitting.

---

#### Block 1.4: Question Intake and Normalization
- **Overview:** Ingests user prompts from Slack and the n8n Chat interface, filters out automated messages, and standardizes disparate payloads into a unified format.
- **Nodes Involved:**
  - `Chat Message Trigger`
  - `When Slack Event Occurs`
  - `Filter Slack Messages`
  - `Normalize Message Data`
  - `Merge Chat and Slack Data`

- **Node Details:**
  - **Chat Message Trigger**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chatTrigger` — Provides an embedded public web chat interface for user queries.
    - *Configuration Choices:* Public webhook enabled with custom greeting initial message.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Normalize Message Data`.
    - *Version Requirements:* Type Version 1.1.
    - *Edge Cases/Failures:* Public endpoint exposure without rate limiting.
  - **When Slack Event Occurs**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.slackTrigger` — Listens for Slack events (`app_mention` and `message`) in a designated channel.
    - *Configuration Choices:* Target channel configured to `C0C0UEMPD5E`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Filter Slack Messages`.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Webhook verification failures or incorrect Slack app subscription scopes.
  - **Filter Slack Messages**
    - *Type and Technical Role:* `n8n-nodes-base.filter` — Filters out bot messages, message edits, and deletions from Slack event streams.
    - *Configuration Choices:* Condition expression checks that `!$json.bot_id` and subtypes do not match bot messages or edits.
    - *Key Expressions/Variables:* `={{ !$json.bot_id && $json.subtype !== "bot_message" && $json.subtype !== "message_changed" && $json.subtype !== "message_deleted" }}`
    - *Input/Output:* Input from `When Slack Event Occurs`; output connected to `Normalize Message Data`.
    - *Version Requirements:* Type Version 2.3.
    - *Edge Cases/Failures:* Unhandled message subtype events passing through if Slack API introduces new types.
  - **Normalize Message Data**
    - *Type and Technical Role:* `n8n-nodes-base.set` — Standardizes chat and Slack inputs into unified attributes (`userInput`, `sessionId`, `channel`, `threadTs`, `origin`).
    - *Configuration Choices:* Assignments mapping input properties conditionally based on source channel.
    - *Key Expressions/Variables:* `={{ $json.chatInput || $json.text }}`, `={{ $json.sessionId || ($json.user + "_" + $json.channel) }}`, `={{ $json.chatInput ? "chat" : "slack" }}`
    - *Input/Output:* Inputs from `Chat Message Trigger` and `Filter Slack Messages`; output connected to `Merge Chat and Slack Data`.
    - *Version Requirements:* Type Version 3.4.
    - *Edge Cases/Failures:* Missing session identifiers resulting in fallback key concatenation errors.
  - **Merge Chat and Slack Data**
    - *Type and Technical Role:* `n8n-nodes-base.merge` — Consolidates normalized message streams from chat and Slack into a single pipeline.
    - *Configuration Choices:* Default append/merge mode.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs from `Normalize Message Data` nodes; output connected to `Support AI Agent`.
    - *Version Requirements:* Type Version 3.
    - *Edge Cases/Failures:* Stream ordering issues under high concurrent load.

---

#### Block 1.5: AI Retrieval and Response Generation
- **Overview:** Evaluates user queries using an OpenAI chat model backed by windowed conversation memory and a Qdrant vector retrieval tool constrained by system instructions.
- **Nodes Involved:**
  - `Support AI Agent`
  - `Buffer Memory Storage`
  - `OpenAI GPT-4 Chat Model`
  - `Retrieve from Qdrant Vector Store`
  - `Query OpenAI Embeddings`

- **Node Details:**
  - **Support AI Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` — Core LangChain agent orchestrating tools, memory, and LLM reasoning to answer user prompts.
    - *Configuration Choices:* Prompt type set to define; includes extensive system instructions defining IntuzBot's identity, greeting behavior, fallback responses, and citation formats.
    - *Key Expressions/Variables:* `={{ $json.userInput }}`
    - *Input/Output:* Inputs from `Merge Chat and Slack Data`, `Buffer Memory Storage`, `OpenAI GPT-4 Chat Model`, and `Retrieve from Qdrant Vector Store`; output connected to `Format Citations for Messaging`.
    - *Version Requirements:* Type Version 1.7.
    - *Edge Cases/Failures:* Token limit overflows or prompt injection attempts bypassing system constraints.
  - **Buffer Memory Storage**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Maintains conversation history per session.
    - *Configuration Choices:* Context window length set to 12 messages; session key configured to custom session ID.
    - *Key Expressions/Variables:* `={{ $json.sessionId }}`
    - *Input/Output:* Output connected to `Support AI Agent` via AI memory connection.
    - *Version Requirements:* Type Version 1.3.
    - *Edge Cases/Failures:* Memory store desynchronization if session IDs collide.
  - **OpenAI GPT-4 Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Language model powering the AI agent's reasoning.
    - *Configuration Choices:* Model configured to `gpt-4o-mini`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Support AI Agent` via AI language model connection.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* OpenAI API outages or rate limiting.
  - **Retrieve from Qdrant Vector Store**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.vectorStoreQdrant` — Acts as an agent tool querying the Qdrant vector collection for relevant context.
    - *Configuration Choices:* Mode set to `retrieve-as-tool` with top-K set to 5; tool name `company_knowledge_base`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs from `Support AI Agent` and `Query OpenAI Embeddings`; output connected to `Support AI Agent` via AI tool connection.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* Poor embedding matches returning irrelevant context chunks.
  - **Query OpenAI Embeddings**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — Computes embeddings for search queries sent to Qdrant.
    - *Configuration Choices:* Model set to `text-embedding-3-small`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Retrieve from Qdrant Vector Store` via AI embedding connection.
    - *Version Requirements:* Type Version 1.
    - *Edge Cases/Failures:* API connectivity failures during vector query generation.

---

#### Block 1.6: Output Formatting and Delivery
- **Overview:** Parses AI-generated text, reformats citation syntax into Slack-compatible markup, and conditionally posts messages back to Slack threads.
- **Nodes Involved:**
  - `Format Citations for Messaging`
  - `Check If Origin Is Slack`
  - `Post Message to Slack`

- **Node Details:**
  - **Format Citations for Messaging**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript code node that transforms standard markdown citation links into Slack-compatible hyperlink formatting.
    - *Configuration Choices:* Custom regex replacement matching `[Source: Name](URL)` and converting it to `<URL|Name>`.
    - *Key Expressions/Variables:* Evaluates `$json.output || $json.text`.
    - *Input/Output:* Input from `Support AI Agent`; output connected to `Check If Origin Is Slack`.
    - *Version Requirements:* Type Version 2.
    - *Edge Cases/Failures:* Malformed citation markdown failing regex matching.
  - **Check If Origin Is Slack**
    - *Type and Technical Role:* `n8n-nodes-base.if` — Conditional branch checking whether the request originated from Slack.
    - *Configuration Choices:* Evaluates whether `$json.origin` equals `slack`.
    - *Key Expressions/Variables:* `={{ $json.origin }}`
    - *Input/Output:* Input from `Format Citations for Messaging`; output connected to `Post Message to Slack`.
    - *Version Requirements:* Type Version 2.
    - *Edge Cases/Failures:* Origin metadata missing or mistyped.
  - **Post Message to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` — Posts formatted responses back to the originating Slack channel and thread.
    - *Configuration Choices:* Operation set to post message to channel using dynamic channel ID and thread timestamp reference.
    - *Key Expressions/Variables:* `={{ $json.formatted }}`, channel ID from Slack trigger, thread timestamp `={{ $('When Slack Event Occurs').item.json.event_ts }}`.
    - *Input/Output:* Input from `Check If Origin Is Slack`; no downstream connections.
    - *Version Requirements:* Type Version 2.2.
    - *Edge Cases/Failures:* Bot lacking chat write permissions or expired Slack OAuth tokens.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation and setup instructions | None | None | ## Self-Updating Internal Document Support Bot<br><br>### How it works<br><br>This workflow keeps an internal support knowledge base updated from Google Drive and answers user questions from chat or Slack. New or updated Drive files are downloaded, routed by file type, converted or extracted to text, then re-indexed into Qdrant with OpenAI embeddings. Incoming chat and Slack questions are normalized, answered by an AI agent using Qdrant retrieval, and Slack-originated requests are posted back to Slack with formatted citations.<br><br>### Setup steps<br><br>- Configure Google Drive OAuth credentials and set the Drive trigger folders/events for created and updated files.<br>- Configure the HTTP request nodes that call Google Drive APIs for DOCX conversion/export/deletion, including OAuth scopes for Drive file copy, export, and delete operations.<br>- Configure Qdrant credentials, collection name, and the delete-by-file_id endpoint so old vectors are removed before re-inserting updated content.<br>- Configure OpenAI credentials for both ingestion embeddings, query embeddings, and the chat model used by the AI Agent.<br>- Configure Slack trigger and Slack post-message credentials, channels, bot permissions, and filtering rules to ignore irrelevant events or bot messages.<br><br>### Customization<br><br>Adjust supported MIME-type routes, text splitter chunk size/overlap, Qdrant metadata fields, retrieval settings, Slack citation formatting, and the AI Agent prompt to match internal documentation and response style. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Intake documentation | None | None | ## Created file intake<br><br>Watches Google Drive for newly created files and downloads the new file content for ingestion. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Intake documentation | None | None | ## Updated file intake<br><br>Watches Google Drive for file updates and downloads the modified file content so existing knowledge can be refreshed. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Routing documentation | None | None | ## Merge and route files<br><br>Combines created and updated file streams, then branches processing based on each file's MIME type. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Extraction documentation | None | None | ## Direct text extraction<br><br>Extracts text directly from supported PDF, Google Doc, and TXT inputs, then funnels the results through a shared extracted-text pass-through node. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Conversion documentation | None | None | ## DOCX conversion path<br><br>Handles DOCX-like files by copying them to a temporary Google Doc, exporting that document as text, and deleting the temporary document afterward. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Sync documentation | None | None | ## Remove stale vectors<br><br>Deletes previously stored Qdrant vectors for the current file_id before inserting the refreshed document content. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Ingestion documentation | None | None | ## Insert document vectors<br><br>Chunks extracted documents, creates OpenAI embeddings, loads document data, and inserts the new vectors into Qdrant. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Slack intake documentation | None | None | ## Slack question intake<br><br>Receives Slack events and filters them before passing valid user messages into the shared question pipeline. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Chat intake documentation | None | None | ## Chat question intake<br><br>Receives direct chat messages from the built-in chat interface as a second user-input channel. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Normalization documentation | None | None | ## Normalize incoming questions<br><br>Standardizes chat and Slack inputs into common fields such as userInput, sessionId, channel, and threadTs, then merges the input streams for the AI agent. |
| Sticky Note11 | n8n-nodes-base.stickyNote | AI generation documentation | None | None | ## Generate grounded answer<br><br>Uses an AI Agent with an OpenAI chat model, window memory, Qdrant retrieval tool, and query embeddings to answer questions from the indexed internal documents. |
| Sticky Note12 | n8n-nodes-base.stickyNote | Formatting documentation | None | None | ## Format and reply<br><br>Formats the agent response with citations, checks whether the request originated in Slack, and posts the final answer back to Slack when appropriate. |
| When File Created in Drive | n8n-nodes-base.googleDriveTrigger | Triggers on created Drive files | None | Download Created File | |
| Download Created File | n8n-nodes-base.googleDrive | Downloads created file binary | When File Created in Drive | Merge Files for Processing | |
| When File Updated in Drive | n8n-nodes-base.googleDriveTrigger | Triggers on updated Drive files | None | Download Updated File | |
| Download Updated File | n8n-nodes-base.googleDrive | Downloads updated file binary | When File Updated in Drive | Merge Files for Processing | |
| Merge Files for Processing | n8n-nodes-base.merge | Merges created and updated file streams | Download Created File, Download Updated File | Route by File Type | |
| Route by File Type | n8n-nodes-base.switch | Routes files by MIME type | Merge Files for Processing | Extract Text from PDF, Extract Text from Google Doc, Convert DOCX to Google Doc, Extract Text from TXT | |
| Extract Text from PDF | n8n-nodes-base.extractFromFile | Extracts text from PDF files | Route by File Type | Text Extraction Output | |
| Text Extraction Output | n8n-nodes-base.noOp | Consolidates extracted text streams | Extract Text from PDF, Extract Text from Google Doc, Extract Text from TXT, Export GDoc to Text | Delete Vectors by File ID | |
| Delete Vectors by File ID | n8n-nodes-base.httpRequest | Deletes old Qdrant vectors by file ID | Text Extraction Output | Insert into Qdrant Vector Store | |
| Insert into Qdrant Vector Store | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Inserts text vectors into Qdrant | Delete Vectors by File ID, Generate OpenAI Embeddings, Load Default Data | None | |
| Generate OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates chunk embeddings for ingestion | None | Insert into Qdrant Vector Store | |
| Load Default Data | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads document data and metadata | Split Text Recursively | Insert into Qdrant Vector Store | |
| Split Text Recursively | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits document text into chunks | None | Load Default Data | |
| Extract Text from Google Doc | n8n-nodes-base.extractFromFile | Extracts text from Google Docs | Route by File Type | Text Extraction Output | |
| Extract Text from TXT | n8n-nodes-base.extractFromFile | Extracts text from TXT files | Route by File Type | Text Extraction Output | |
| Chat Message Trigger | @n8n/n8n-nodes-langchain.chatTrigger | Web chat user input trigger | None | Normalize Message Data | |
| Merge Chat and Slack Data | n8n-nodes-base.merge | Merges normalized chat and Slack streams | Normalize Message Data | Support AI Agent | |
| Support AI Agent | @n8n/n8n-nodes-langchain.agent | Core AI agent handling reasoning and tools | Merge Chat and Slack Data, Buffer Memory Storage, OpenAI GPT-4 Chat Model, Retrieve from Qdrant Vector Store | Format Citations for Messaging | |
| Buffer Memory Storage | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation memory | None | Support AI Agent | |
| OpenAI GPT-4 Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM chat model for agent | None | Support AI Agent | |
| Retrieve from Qdrant Vector Store | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Qdrant retrieval tool for agent | Query OpenAI Embeddings | Support AI Agent | |
| Query OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates query embeddings | None | Retrieve from Qdrant Vector Store | |
| Format Citations for Messaging | n8n-nodes-base.code | Formats citation links for Slack/Chat | Support AI Agent | Check If Origin Is Slack | |
| Check If Origin Is Slack | n8n-nodes-base.if | Routes response if origin is Slack | Format Citations for Messaging | Post Message to Slack | |
| Post Message to Slack | n8n-nodes-base.slack | Posts message back to Slack channel | Check If Origin Is Slack | None | |
| Normalize Message Data | n8n-nodes-base.set | Standardizes chat and Slack message fields | Chat Message Trigger, Filter Slack Messages | Merge Chat and Slack Data | |
| When Slack Event Occurs | @n8n/n8n-nodes-langchain.slackTrigger | Triggers on Slack events | None | Filter Slack Messages | |
| Filter Slack Messages | n8n-nodes-base.filter | Filters out bot events from Slack | When Slack Event Occurs | Normalize Message Data | |
| Convert DOCX to Google Doc | n8n-nodes-base.httpRequest | Copies DOCX to temporary Google Doc | Route by File Type | Export GDoc to Text | |
| Export GDoc to Text | n8n-nodes-base.httpRequest | Exports temporary Google Doc as text | Convert DOCX to Google Doc | Remove Temporary Google Doc, Text Extraction Output | |
| Remove Temporary Google Doc | n8n-nodes-base.httpRequest | Deletes temporary Google Doc | Export GDoc to Text | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step numbered guide to rebuild the entire workflow manually in n8n.

#### Step 1: Set Up File Intake Triggers
1. Add a **Google Drive Trigger** node named `When File Created in Drive`.
   - Set **Event** to `File Created`.
   - Set **Trigger On** to `Specific Folder`, pointing to your target folder (e.g., folder ID `1OMm12f0vxmfuLjM0D0Fq2bFzR20qP37m`).
   - Configure polling time to every minute.
   - Configure **Google Drive OAuth2 API** credentials.
2. Add a **Google Drive** node named `Download Created File`.
   - Set **Operation** to `Download`.
   - Set **File ID** to `={{ $json.id }}`.
   - Set binary property name to `data` and enable Google file conversion to `text/plain`.
   - Connect `When File Created in Drive` output to `Download Created File`.
3. Add a **Google Drive Trigger** node named `When File Updated in Drive`.
   - Set **Event** to `File Updated`, targeting the same folder ID and polling interval.
4. Add a **Google Drive** node named `Download Updated File`.
   - Set **Operation** to `Download`, **File ID** to `={{ $json.id }}`, with file conversion enabled.
   - Connect `When File Updated in Drive` output to `Download Updated File`.
5. Add a **Merge** node named `Merge Files for Processing`.
   - Connect `Download Created File` to input index 0 and `Download Updated File` to input index 1.

#### Step 2: Configure MIME Routing and Text Extraction
1. Add a **Switch** node named `Route by File Type`.
   - Connect `Merge Files for Processing` output to this node.
   - Create 4 output rules:
     - Output 0 (PDF): Left value `={{ $binary.data.mimeType }}`, operator `equals`, right value `application/pdf`.
     - Output 1 (GoogleDoc): Left value `={{ $json.mimeType }}`, operator `equals`, right value `application/vnd.google-apps.document`.
     - Output 2 (DOCX): Left value `={{ $binary.data.mimeType }}`, operator `equals`, right value `application/vnd.openxmlformats-officedocument.wordprocessingml.document`.
     - Output 3 (TXT): Left value `={{ $binary.data.mimeType }}`, operator `equals`, right value `text/plain`.
2. Add an **Extract From File** node named `Extract Text from PDF`.
   - Set operation to `PDF`. Connect Switch Output 0 here.
3. Add an **Extract From File** node named `Extract Text from Google Doc`.
   - Set operation to `Text`. Connect Switch Output 1 here.
4. Add an **Extract From File** node named `Extract Text from TXT`.
   - Set operation to `Text`. Connect Switch Output 3 here.
5. Set up the DOCX Conversion pipeline (Switch Output 2):
   - Add an **HTTP Request** node named `Convert DOCX to Google Doc`. Set method to `POST`, URL to `={{ "https://www.googleapis.com/drive/v3/files/" + ($('When File Created in Drive').isExecuted ? $('When File Created in Drive').item.json.id : $('When File Updated in Drive').item.json.id) + "/copy" }}`, body type to JSON with `={{ JSON.stringify({ name: "n8n-temp-docx-" + $execution.id, mimeType: "application/vnd.google-apps.document" }) }}`, using Google Drive OAuth2 credentials.
   - Add an **HTTP Request** node named `Export GDoc to Text`. Set method to `GET`, URL to `={{ "https://www.googleapis.com/drive/v3/files/" + $('Convert DOCX to Google Doc').item.json.id + "/export?mimeType=text/plain" }}`, response format to text.
   - Add an **HTTP Request** node named `Remove Temporary Google Doc`. Set method to `DELETE`, URL to `={{ "https://www.googleapis.com/drive/v3/files/" + $('Convert DOCX to Google Doc').item.json.id }}`.
   - Connect `Convert DOCX to Google Doc` -> `Export GDoc to Text`. Connect `Export GDoc to Text` outputs to both `Remove Temporary Google Doc` and `Text Extraction Output`.
6. Add a **NoOp** node named `Text Extraction Output`.
   - Connect outputs from `Extract Text from PDF`, `Extract Text from Google Doc`, `Extract Text from TXT`, and `Export GDoc to Text` into this node.

#### Step 3: Implement Qdrant Synchronization and Vector Storage
1. Add an **HTTP Request** node named `Delete Vectors by File ID`.
   - Set method to `POST`, URL to your Qdrant cluster points delete endpoint (`https://<cluster-url>:6333/collections/company_docs/points/delete?wait=true`), specify JSON body with filter expression checking `metadata.file_id`. Use Qdrant REST API credentials.
   - Connect `Text Extraction Output` to this node.
2. Add a **Vector Store Qdrant** node named `Insert into Qdrant Vector Store`.
   - Set mode to `insert`, collection name to `company_docs`.
   - Connect `Delete Vectors by File ID` to main input.
3. Attach AI sub-nodes to `Insert into Qdrant Vector Store`:
   - Add **Embeddings OpenAI** (`Generate OpenAI Embeddings`) with model `text-embedding-3-small` connected to the embedding input.
   - Add **Document Default Data Loader** (`Load Default Data`) configured with metadata values (`file_id`, `file_name`, `drive_url`, `last_modified`) pulling expressions from Drive triggers, and jsonData set to `={{ $('Text Extraction Output').item.json.text }}`. Connect to document input.
   - Add **Text Splitter Recursive Character** (`Split Text Recursively`) with chunk overlap 200 connected to the text splitter input of `Load Default Data`.

#### Step 4: Set Up Chat and Slack Message Triggers
1. Add a **Chat Trigger** node named `Chat Message Trigger`. Set public to true with initial greeting message.
2. Add a **Slack Trigger** node named `When Slack Event Occurs`. Trigger on `app_mention` and `message` for target channel ID. Configure Slack OAuth2 credentials.
3. Add a **Filter** node named `Filter Slack Messages`. Set condition to evaluate `!$json.bot_id && $json.subtype !== "bot_message" && $json.subtype !== "message_changed" && $json.subtype !== "message_deleted"`. Connect `When Slack Event Occurs` here.
4. Add a **Set** node named `Normalize Message Data`. Configure assignments for `userInput`, `sessionId`, `channel`, `threadTs`, and `origin`. Connect both `Chat Message Trigger` and `Filter Slack Messages` outputs here.
5. Add a **Merge** node named `Merge Chat and Slack Data` to combine normalized inputs.

#### Step 5: Configure the AI Support Agent
1. Add an **AI Agent** node named `Support AI Agent`.
   - Set text to `={{ $json.userInput }}`.
   - Configure system message prompt defining IntuzBot's identity, guidelines, fallback behavior, and citation requirements.
   - Connect `Merge Chat and Slack Data` to main input.
2. Attach AI sub-nodes:
   - Add **LM Chat OpenAI** (`OpenAI GPT-4 Chat Model`) with model `gpt-4o-mini` connected to language model input.
   - Add **Memory Buffer Window** (`Buffer Memory Storage`) with session key `={{ $json.sessionId }}` and context window length 12 connected to memory input.
   - Add **Vector Store Qdrant** (`Retrieve from Qdrant Vector Store`) set to `retrieve-as-tool` mode, topK 5, tool name `company_knowledge_base`, connected to tool input.
   - Add **Embeddings OpenAI** (`Query OpenAI Embeddings`) with model `text-embedding-3-small` connected to embedding input of the Qdrant retrieval tool.

#### Step 6: Format Citations and Reply
1. Add a **Code** node named `Format Citations for Messaging`.
   - Insert JavaScript code mapping markdown citation links (`[Source: Name](URL)`) to Slack link markup (`<URL|Name>`).
   - Connect `Support AI Agent` output here.
2. Add an **If** node named `Check If Origin Is Slack`.
   - Set condition to check if `={{ $json.origin }}` equals `slack`. Connect `Format Citations for Messaging` here.
3. Add a **Slack** node named `Post Message to Slack`.
   - Set operation to post message, channel select to channel ID `={{ $('When Slack Event Occurs').item.json.channel }}`, text to `={{ $json.formatted }}`, and thread timestamp to `={{ $('When Slack Event Occurs').item.json.event_ts }}` under other options.
   - Connect true output of `Check If Origin Is Slack` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official Website Template | [Intuz n8n Workflow Automation Templates](https://www.intuz.com/n8n-workflow-automation-templates/) |
| Support Email | getstarted@intuz.com |
| Company LinkedIn Page | [Intuz LinkedIn](https://www.linkedin.com/company/intuz) |
| Partner Links | [n8n Partner Links - Intuz](https://n8n.partnerlinks.io/intuz) |
| Custom Workflow Automation Requests | [Intuz Get Started](https://www.intuz.com/get-started/) |