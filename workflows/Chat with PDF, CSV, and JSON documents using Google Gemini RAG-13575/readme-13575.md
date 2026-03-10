Chat with PDF, CSV, and JSON documents using Google Gemini RAG

https://n8nworkflows.xyz/workflows/chat-with-pdf--csv--and-json-documents-using-google-gemini-rag-13575


# Chat with PDF, CSV, and JSON documents using Google Gemini RAG

# Workflow Reference: AI Multi-Format Document Q&A (PDF, CSV, JSON)

This document provides a technical breakdown of an n8n workflow designed to transform static documents (PDF, CSV, and JSON) into a searchable, AI-powered knowledge base using Retrieval-Augmented Generation (RAG) and Google Gemini.

---

### 1. Workflow Overview

The workflow is designed to bridge the gap between raw data files and conversational AI. It operates through two primary entry points: a file ingestion pipeline and a real-time chatbot interface.

**Logical Blocks:**
- **1.1 Document Ingestion:** Handles the upload, metadata enrichment, and indexing of files into an in-memory vector database.
- **1.2 Conversational Retrieval (RAG):** Manages the user interface, query processing, semantic search, and response generation using a specialized AI agent.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion
**Overview:** Processes uploaded files and converts them into searchable embeddings used for semantic retrieval.

*   **Document Upload Form (Trigger):**
    *   **Type:** Form Trigger
    *   **Configuration:** A public-facing form with a file upload field restricted to `.pdf`, `.csv`, and `.json`.
    *   **Role:** Acts as the entry point for populating the knowledge base.
*   **Add Metadata:**
    *   **Type:** Set Node
    *   **Configuration:** Extracts and assigns `filename`, `fileType` (mimetype), and `uploadDate` (submission timestamp) to the incoming file object.
    *   **Input:** Binary file data; **Output:** Structured metadata and binary content.
*   **Vector Store Insert:**
    *   **Type:** Vector Store (In-Memory)
    *   **Configuration:** Set to `insert` mode with a memory key of `Document_File`.
    *   **Sub-components:**
        *   **Document Loader:** Processes binary data using "Custom" splitting mode.
        *   **Token Splitter:** Uses `RecursiveCharacterTextSplitter` with a `200` chunk overlap to maintain context between segments.
        *   **Embeddings Google Gemini:** Uses `models/gemini-embedding-001` to vectorize text.
    *   **Failure Modes:** Large file timeouts or API rate limits on Gemini embeddings.

#### 2.2 Chat and AI Response
**Overview:** Retrieves relevant document context and generates grounded AI answers using a RAG-based agent.

*   **Chatbot Trigger:**
    *   **Type:** AI Chat Trigger
    *   **Configuration:** Configures the UI with the title "AI Document Knowledge Base Assistant" and a predefined welcome message.
*   **Knowledge Base Agent:**
    *   **Type:** AI Agent
    *   **Configuration:** Uses a comprehensive system prompt (10 rules) enforcing strict adherence to the knowledge base, prohibiting hallucinations, and requiring source citations.
    *   **Inputs:** User query from `Chatbot Trigger`.
*   **Vector Store Retrieve:**
    *   **Type:** Vector Store (In-Memory)
    *   **Configuration:** Mode set to `retrieve-as-tool`. `topK` is set to 5 (retrieves the 5 most relevant segments).
    *   **Role:** Functions as a "Search Tool" for the Agent to access the `Document_File` index.
*   **Google Gemini Chat Model:**
    *   **Type:** Google Gemini Chat Model
    *   **Role:** The "brain" providing reasoning and natural language synthesis.
*   **Chat Memory:**
    *   **Type:** Window Buffer Memory
    *   **Configuration:** Retains the last 10 messages to allow for contextual follow-up questions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Document Upload Form | Form Trigger | Entry point for files | None | Add Metadata | ## Document Ingestion |
| Add Metadata | Set | Data enrichment | Document Upload Form | Vector Store Insert | ## Document Ingestion |
| Vector Store Insert | In-Memory Vector Store | Data indexing | Add Metadata | None | ## Document Ingestion |
| Document Loader | Document Loader | File parsing | Vector Store Insert | Vector Store Insert | ## Document Ingestion |
| Token Splitter | Text Splitter | Text chunking | Document Loader | Document Loader | ## Document Ingestion |
| Embeddings Google Gemini | Gemini Embeddings | Vector generation | Vector Store Insert / Retrieve | Vector Store Insert / Retrieve | ## Document Ingestion |
| Chatbot Trigger | Chat Trigger | Chat UI Entry point | None | Knowledge Base Agent | ## Chat and AI response |
| Knowledge Base Agent | AI Agent | Logic & Reasoning | Chatbot Trigger | None | ## Chat and AI response |
| Google Gemini Chat Model | Gemini Chat Model | LLM Provider | Knowledge Base Agent | Knowledge Base Agent | ## Chat and AI response |
| Chat Memory | Window Buffer Memory | Conversation history | Knowledge Base Agent | Knowledge Base Agent | ## Chat and AI response |
| Vector Store Retrieve | In-Memory Vector Store | Search tool | Knowledge Base Agent | Knowledge Base Agent | ## Chat and AI response |

---

### 4. Reproducing the Workflow from Scratch

1.  **Ingestion Setup:**
    *   Add a **Form Trigger**. Configure a "File" field allowing `.pdf, .csv, .json`.
    *   Connect a **Set Node** (named "Add Metadata"). Map `{{ $json['Document File'][0].filename }}` and other metadata properties.
    *   Connect an **In-Memory Vector Store** node. Set Mode to "Insert". Provide a unique Name/ID for the memory key (e.g., `Document_File`).
    *   Attach a **Document Loader** (Binary mode) to the Vector Store.
    *   Attach a **Recursive Character Text Splitter** to the Document Loader (Overlap: 200).
    *   Attach **Google Gemini Embeddings** to the Vector Store.

2.  **Chat Interface Setup:**
    *   Add a **Chatbot Trigger**. Customize the title and greeting.
    *   Add an **AI Agent**. Set the Prompt Type to "Define". Use a system message that instructs the agent to search the tool first and only answer based on retrieved data.
    *   Connect the **Google Gemini Chat Model** to the Agent.
    *   Connect **Window Buffer Memory** to the Agent (Context limit: 10).
    *   Connect another **In-Memory Vector Store** node to the Agent's tool input. Set its mode to "Retrieve as Tool". Use the *same* memory key (`Document_File`) as the Ingestion node. Set `topK` to 5.
    *   Attach the same **Google Gemini Embeddings** to this Retrieval node.

3.  **Credential Configuration:**
    *   Configure a **Google Gemini API** credential and apply it to both the Chat Model and the Embeddings node.

4.  **Activation:**
    *   Save the workflow and toggle it to "Active". Upload a file via the form URL before testing the chat.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Persistence Warning** | Uses an in-memory vector store. Data resets when the workflow restarts. Replace with a persistent database (e.g., Pinecone, Supabase, Qdrant) for production use. |
| **Workflow Creator** | Developed by Md. Khalid Ali (AIN Consulting) |
| **Creator Website** | [https://ainconsulting.com](https://ainconsulting.com) |
| **RAG Methodology** | Demonstrates the core cycle: Ingest -> Embed -> Store -> Retrieve -> Generate. |