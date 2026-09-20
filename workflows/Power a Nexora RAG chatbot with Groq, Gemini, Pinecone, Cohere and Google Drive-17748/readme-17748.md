Power a Nexora RAG chatbot with Groq, Gemini, Pinecone, Cohere and Google Drive

https://n8nworkflows.xyz/workflows/power-a-nexora-rag-chatbot-with-groq--gemini--pinecone--cohere-and-google-drive-17748


# Power a Nexora RAG chatbot with Groq, Gemini, Pinecone, Cohere and Google Drive

### 1. Workflow Overview

This workflow implements a complete Retrieval-Augmented Generation (RAG) chatbot and automated knowledge base maintenance pipeline. It serves two distinct functional roles: handling real-time customer and internal inquiries via a chat interface grounded in verified company documents, and automatically updating the vector database when new files are uploaded to cloud storage. 

The architecture is divided into two primary logical blocks:
- **1.1 Chatbot Orchestration & Retrieval Stack:** Manages incoming chat interactions through a public webhook, uses an advanced Large Language Model (Groq Llama) with conversation memory, and fetches relevant contextual data from a Pinecone vector store enhanced by Google Gemini embeddings and Cohere reranking.
- **1.2 Automated Google Drive Ingestion Pipeline:** Monitors a designated Google Drive folder for new files, downloads and processes the documents, chunks the content using recursive character splitting, embeds the text via Google Gemini, and upserts the vectors into the same Pinecone index for immediate retrieval availability.

---

### 2. Block-by-Block Analysis

#### 2.1 Chatbot Orchestration & Retrieval Stack
**Overview:** This block captures incoming user messages, processes them through an AI agent equipped with conversational memory, and queries a vector knowledge base using semantic search and reranking to deliver strictly grounded, accurate responses.

**Nodes Involved:**
- `When Chat Message Received`
- `Chatbot Response Agent`
- `Groq Chat Model Interaction`
- `Memory Buffer Window`
- `Main Pinecone Vector Store`
- `Google Gemini Embeddings 1`
- `Cohere Reranker`

**Node Details:**
- **When Chat Message Received**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chatTrigger` (Chat Trigger). Acts as the primary entry point for user chat sessions via a public webhook.
  - *Configuration:* Set to public webhook mode.
  - *Connections:* Outputs to `Chatbot Response Agent` (Main).
  - *Edge Cases/Failure Types:* Webhook connectivity drops or unauthenticated external client requests.
- **Chatbot Response Agent**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent). Orchestrates the conversation, enforces persona and strict RAG grounding guidelines, and utilizes attached tools and memory.
  - *Configuration:* Configured with an extensive system prompt defining identity as "Nexora Assistant," response constraints, grounding rules, tone, and scope boundaries.
  - *Connections:* Inputs from `When Chat Message Received`. Receives AI tools, language model, memory, and vector retrieval configurations via sub-connections.
  - *Edge Cases/Failure Types:* Hallucination risks mitigated by prompt rules; failure if connected language model or tools time out.
- **Groq Chat Model Interaction**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (Groq Chat Model). Provides high-speed inference capabilities to the AI Agent.
  - *Configuration:* Model set to `llama-3.3-70b-versatile`.
  - *Connections:* Outputs to `Chatbot Response Agent` (`ai_languageModel`).
  - *Credentials Required:* Groq API credentials.
  - *Edge Cases/Failure Types:* API rate limits, service outages, or invalid API keys.
- **Memory Buffer Window**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Window Buffer Memory). Maintains short-term conversational context across turns.
  - *Connections:* Connects to `Chatbot Response Agent`.
- **Main Pinecone Vector Store**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.vectorStorePinecone` (Pinecone Vector Store). Acts as a retrieval tool for the AI Agent to fetch relevant company knowledge chunks.
  - *Configuration:* Mode set to `retrieve-as-tool`, linked to Pinecone index `erhan8n` with reranking enabled.
  - *Connections:* Receives embedding from `Google Gemini Embeddings 1` and reranker from `Cohere Reranker`. Outputs to `Chatbot Response Agent` (`ai_tool`).
  - *Credentials Required:* Pinecone API credentials.
  - *Edge Cases/Failure Types:* Dimensionality mismatches between query vectors and stored vectors.
- **Google Gemini Embeddings 1**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini` (Google Gemini Embeddings). Transforms text queries into vector embeddings for semantic search.
  - *Connections:* Outputs to `Main Pinecone Vector Store` (`ai_embedding`).
  - *Credentials Required:* Google Gemini (PaLM) API credentials.
  - *Edge Cases/Failure Types:* API quota exhaustion or authentication failures.
- **Cohere Reranker**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.rerankerCohere` (Cohere Reranker). Reranks retrieved Pinecone chunks to prioritize the most relevant semantic matches.
  - *Connections:* Outputs to `Main Pinecone Vector Store` (`ai_reranker`).
  - *Credentials Required:* Cohere API credentials.
  - *Edge Cases/Failure Types:* Service unavailability or payload limits exceeded.

---

#### 2.2 Automated Google Drive Ingestion Pipeline
**Overview:** This block automates the synchronization of new knowledge base documents by polling a Google Drive folder, downloading added files, processing their content into manageable text chunks, generating vector embeddings, and upserting them into the vector database.

**Nodes Involved:**
- `When File Added to Drive`
- `Download File from Drive`
- `Secondary Pinecone Vector Store`
- `Generate Google Gemini Embeddings`
- `Load Default Data Format`
- `Split Text by Character`

**Node Details:**
- **When File Added to Drive**
  - *Type & Role:* `n8n-nodes-base.googleDriveTrigger` (Google Drive Trigger). Monitors a specific Google Drive directory for newly created files.
  - *Configuration:* Trigger event set to `fileCreated` on specific folder (`Erha File`, ID: `1dWj0JtJhRtnAOuc-SyJv0DS5CR-x9V6V`) polling every minute.
  - *Connections:* Outputs to `Download File from Drive` (Main).
  - *Credentials Required:* Google Drive OAuth2 credentials.
  - *Edge Cases/Failure Types:* Polling delays, permission revocation on the watched folder.
- **Download File from Drive**
  - *Type & Role:* `n8n-nodes-base.googleDrive` (Google Drive). Downloads the binary file content of the newly detected document.
  - *Configuration:* Operation set to `download`, targeting file ID `={{ $json.id }}`.
  - *Connections:* Inputs from `When File Added to Drive`. Outputs to `Secondary Pinecone Vector Store` (Main).
  - *Credentials Required:* Google Drive OAuth2 credentials.
  - *Edge Cases/Failure Types:* File deletion before download completion, unsupported file formats.
- **Secondary Pinecone Vector Store**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.vectorStorePinecone` (Pinecone Vector Store). Inserts processed document chunks and their vector embeddings into the knowledge base index.
  - *Configuration:* Mode set to `insert`, targeting Pinecone index `erhan8n`.
  - *Connections:* Receives binary file data from `Download File from Drive`, document loader from `Load Default Data Format`, and embeddings from `Generate Google Gemini Embeddings`.
  - *Credentials Required:* Pinecone API credentials.
  - *Edge Cases/Failure Types:* Upsert failures due to network timeouts or index capacity limits.
- **Generate Google Gemini Embeddings**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini` (Google Gemini Embeddings). Generates vector representations for document text chunks.
  - *Configuration:* Model set to `models/gemini-embedding-2`.
  - *Connections:* Outputs to `Secondary Pinecone Vector Store` (`ai_embedding`).
  - *Credentials Required:* Google Gemini (PaLM) API credentials.
- **Load Default Data Format**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` (Default Document Loader). Parses binary document data into text format and attaches metadata.
  - *Configuration:* Configured for binary data input with metadata values capturing `={{ $json.name }}` as file name.
  - *Connections:* Receives text splitter from `Split Text by Character`. Outputs to `Secondary Pinecone Vector Store` (`ai_document`).
- **Split Text by Character**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter` (Recursive Character Text Splitter). Breaks large text documents into smaller chunks for precise embedding.
  - *Configuration:* Chunk size set to `500` characters with a chunk overlap of `200` characters.
  - *Connections:* Outputs to `Load Default Data Format` (`ai_textSplitter`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Setup Guide | None | None | ## Erha Chatbot Final<br><br>### How it works<br><br>This workflow powers a retrieval-augmented chatbot and maintains its knowledge base from Google Drive files. New or changed Drive files are downloaded, split, embedded with Google Gemini, and stored in Pinecone. Chat messages are handled by an AI Agent using a Groq chat model, short-term memory, Pinecone retrieval, Gemini embeddings, and Cohere reranking to generate grounded responses.<br><br>### Setup steps<br><br>- Connect credentials for Google Drive, Pinecone, Google Gemini embeddings, Groq, and Cohere.<br>- Configure the Google Drive Trigger to watch the intended folder or file events.<br>- Set the Pinecone index, namespace, and vector dimensions to match the Gemini embedding model used in both ingestion and retrieval.<br>- Configure the AI Agent prompt, memory window size, retrieval settings, and Groq model according to the chatbot’s requirements.<br><br>### Customization<br><br>Adjust the text splitter chunk size and overlap for your documents, tune Pinecone retrieval and Cohere reranking parameters, and update the AI Agent instructions to fit Erha’s tone, scope, and escalation rules. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Grouping Label | None | None | ## Chat intake and agent<br><br>Receives user chat messages and routes them into the AI Agent, which orchestrates the chatbot response using connected model, memory, and retrieval tools nearby on the canvas. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Grouping Label | None | None | ## Retrieval and reranking<br><br>Provides the chat agent’s knowledge retrieval stack, including Pinecone lookup, Gemini embeddings for query vectors, and Cohere reranking to prioritize the most relevant retrieved content. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Grouping Label | None | None | ## Drive file intake<br><br>Watches Google Drive for new or updated files and downloads the file content so it can be added to the vector knowledge base. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Grouping Label | None | None | ## Embed and index documents<br><br>Transforms downloaded documents into searchable vector content by loading file data, splitting text into chunks, embedding it with Google Gemini, and storing the results in Pinecone. |
| When Chat Message Received | @n8n/n8n-nodes-langchain.chatTrigger | Chat Webhook Trigger | None | Chatbot Response Agent | ## Chat intake and agent<br><br>Receives user chat messages and routes them into the AI Agent, which orchestrates the chatbot response using connected model, memory, and retrieval tools nearby on the canvas. |
| Chatbot Response Agent | @n8n/n8n-nodes-langchain.agent | AI Agent Orchestrator | When Chat Message Received | None | ## Chat intake and agent<br><br>Receives user chat messages and routes them into the AI Agent, which orchestrates the chatbot response using connected model, memory, and retrieval tools nearby on the canvas. |
| Groq Chat Model Interaction | @n8n/n8n-nodes-langchain.lmChatGroq | LLM Provider (Groq) | None | Chatbot Response Agent | ## Chat intake and agent<br><br>Receives user chat messages and routes them into the AI Agent, which orchestrates the chatbot response using connected model, memory, and retrieval tools nearby on the canvas. |
| Main Pinecone Vector Store | @n8n/n8n-nodes-langchain.vectorStorePinecone | Vector Knowledge Retrieval Tool | Google Gemini Embeddings 1, Cohere Reranker | Chatbot Response Agent | ## Retrieval and reranking<br><br>Provides the chat agent’s knowledge retrieval stack, including Pinecone lookup, Gemini embeddings for query vectors, and Cohere reranking to prioritize the most relevant retrieved content. |
| Cohere Reranker | @n8n/n8n-nodes-langchain.rerankerCohere | Search Results Reranking | None | Main Pinecone Vector Store | ## Retrieval and reranking<br><br>Provides the chat agent’s knowledge retrieval stack, including Pinecone lookup, Gemini embeddings for query vectors, and Cohere reranking to prioritize the most relevant retrieved content. |
| When File Added to Drive | n8n-nodes-base.googleDriveTrigger | Google Drive Folder Watcher | None | Download File from Drive | ## Drive file intake<br><br>Watches Google Drive for new or updated files and downloads the file content so it can be added to the vector knowledge base. |
| Download File from Drive | n8n-nodes-base.googleDrive | File Downloader | When File Added to Drive | Secondary Pinecone Vector Store | ## Drive file intake<br><br>Watches Google Drive for new or updated files and downloads the file content so it can be added to the vector knowledge base. |
| Load Default Data Format | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Document Parser & Metadata Loader | Split Text by Character | Secondary Pinecone Vector Store | ## Embed and index documents<br><br>Transforms downloaded documents into searchable vector content by loading file data, splitting text into chunks, embedding it with Google Gemini, and storing the results in Pinecone. |
| Split Text by Character | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Text Chunking Utility | None | Load Default Data Format | ## Embed and index documents<br><br>Transforms downloaded documents into searchable vector content by loading file data, splitting text into chunks, embedding it with Google Gemini, and storing the results in Pinecone. |
| Generate Google Gemini Embeddings | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Vector Embedding Generator (Ingestion) | None | Secondary Pinecone Vector Store | ## Embed and index documents<br><br>Transforms downloaded documents into searchable vector content by loading file data, splitting text into chunks, embedding it with Google Gemini, and storing the results in Pinecone. |
| Secondary Pinecone Vector Store | @n8n/n8n-nodes-langchain.vectorStorePinecone | Vector Database Upsert Node | Download File from Drive, Load Default Data Format, Generate Google Gemini Embeddings | None | ## Embed and index documents<br><br>Transforms downloaded documents into searchable vector content by loading file data, splitting text into chunks, embedding it with Google Gemini, and storing the results in Pinecone. |
| Google Gemini Embeddings 1 | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Vector Embedding Generator (Retrieval) | None | Main Pinecone Vector Store | ## Retrieval and reranking<br><br>Provides the chat agent’s knowledge retrieval stack, including Pinecone lookup, Gemini embeddings for query vectors, and Cohere reranking to prioritize the most relevant retrieved content. |
| Memory Buffer Window | @n8n/n8n-nodes-langchain.memoryBufferWindow | Short-term Conversation Memory | None | Chatbot Response Agent | ## Chat intake and agent<br><br>Receives user chat messages and routes them into the AI Agent, which orchestrates the chatbot response using connected model, memory, and retrieval tools nearby on the canvas. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create Chat Intake & Agent:**
   - Add a **Chat Trigger** (`When Chat Message Received`) node. Configure mode as webhook (public).
   - Add an **AI Agent** (`Chatbot Response Agent`) node. Set the system message prompt to define the Nexora Assistant identity, guardrails, and grounding rules. Connect `When Chat Message Received` (Main) to `Chatbot Response Agent` (Main).
   - Add a **Groq Chat Model** (`Groq Chat Model Interaction`) node. Set the model to `llama-3.3-70b-versatile`. Configure Groq API credentials. Connect its output to the `ai_languageModel` input of the `Chatbot Response Agent`.
   - Add a **Window Buffer Memory** (`Memory Buffer Window`) node. Connect it to the `ai_memory` input of the `Chatbot Response Agent`.

2. **Set Up Knowledge Retrieval & Reranking:**
   - Add a **Pinecone Vector Store** (`Main Pinecone Vector Store`) node. Set mode to `retrieve-as-tool`, select index `erhan8n`, enable reranking, and fill in the tool description. Configure Pinecone API credentials. Connect its output to the `ai_tool` input of the `Chatbot Response Agent`.
   - Add a **Google Gemini Embeddings** (`Google Gemini Embeddings 1`) node. Configure Google Gemini API credentials. Connect its output to the `ai_embedding` input of `Main Pinecone Vector Store`.
   - Add a **Cohere Reranker** (`Cohere Reranker`) node. Configure Cohere API credentials. Connect its output to the `ai_reranker` input of `Main Pinecone Vector Store`.

3. **Set Up Google Drive Ingestion Trigger:**
   - Add a **Google Drive Trigger** (`When File Added to Drive`) node. Set event to `fileCreated`, trigger on specific folder, select the target folder (`Erha File`, ID: `1dWj0JtJhRtnAOuc-SyJv0DS5CR-x9V6V`), and set polling frequency to every minute. Configure Google Drive OAuth2 credentials.
   - Add a **Google Drive** (`Download File from Drive`) node. Set operation to `download` with file ID `={{ $json.id }}`. Configure Google Drive OAuth2 credentials. Connect `When File Added to Drive` (Main) to `Download File from Drive` (Main).

4. **Set Up Document Processing & Indexing:**
   - Add a **Pinecone Vector Store** (`Secondary Pinecone Vector Store`) node. Set mode to `insert`, select index `erhan8n`. Configure Pinecone API credentials.
   - Add a **Google Gemini Embeddings** (`Generate Google Gemini Embeddings`) node with model `models/gemini-embedding-2`. Configure Google Gemini API credentials. Connect its output to the `ai_embedding` input of `Secondary Pinecone Vector Store`.
   - Add a **Recursive Character Text Splitter** (`Split Text by Character`) node. Set chunk size to `500` and chunk overlap to `200`.
   - Add a **Default Data Loader** (`Load Default Data Format`) node. Set data type to binary, configure metadata mapping `file` to `={{ $json.name }}`. Connect `Split Text by Character` (`ai_textSplitter`) to `Load Default Data Format`, and connect `Load Default Data Format` (`ai_document`) to `Secondary Pinecone Vector Store` (`ai_document`).
   - Connect `Download File from Drive` (Main) to `Secondary Pinecone Vector Store` (Main).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure the vector dimensions of the Pinecone index (`erhan8n`) match the output dimensions of the Google Gemini embedding models used in both ingestion and retrieval pipelines. | Vector Database Configuration |
| The Google Drive watcher polls every minute; ensure the authenticated Google Drive account has read permissions for the target folder (`Erha File`). | Google Drive Integration |