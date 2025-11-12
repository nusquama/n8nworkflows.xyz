Local Chatbot with Retrieval Augmented Generation (RAG)

https://n8nworkflows.xyz/workflows/local-chatbot-with-retrieval-augmented-generation--rag--5148


# Local Chatbot with Retrieval Augmented Generation (RAG)

### 1. Workflow Overview

The **Local Chatbot with Retrieval Augmented Generation (RAG)** workflow is designed to enable semantic search and conversational AI capabilities over user-submitted PDF documents. It supports two main use cases:

- **Data Ingestion:** Users upload PDF files via a form, which are processed, split into manageable chunks, embedded into vector representations, and stored in a local Qdrant vector database for semantic retrieval.
  
- **RAG Chatbot Interaction:** Users interact with a chatbot that leverages an AI agent connected to a language model and a semantic vector store retriever. The chatbot retrieves relevant document embeddings to generate context-aware, grounded responses.

The workflow is logically divided into two main blocks:

- **1.1 Data Ingestion Block:** Handles form submission, document loading, splitting, embedding, and vector storage.
- **1.2 RAG Chatbot Block:** Receives chat messages, invokes an AI agent with memory and language model, and uses the vector store as a retrieval tool to generate responses.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Data Ingestion Block

**Overview:**  
This block captures user-submitted PDF files through a form, loads the document content, splits it into text chunks, computes embeddings using a local Ollama service, and inserts these vectors into a local Qdrant vector database for future semantic retrieval.

**Nodes Involved:**  
- On form submission  
- Qdrant Vector Store (Insert mode)  
- Default Data Loader  
- Recursive Character Text Splitter  
- Embeddings Ollama  
- Sticky Note (Data Ingestion description)

**Node Details:**

- **On form submission**  
  - *Type & Role:* Form Trigger node; entry point for document ingestion triggered by user PDF upload.  
  - *Configuration:* Form titled "Add your file here" with one required file upload field accepting only `.pdf` files.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Sends uploaded file binary data downstream.  
  - *Edge Cases:* File upload failures, unsupported file formats, large file size issues.  
  - *Version:* 2.2.

- **Default Data Loader**  
  - *Type & Role:* Document loader; processes incoming binary PDF to extract text content.  
  - *Configuration:* Set to load data from binary input.  
  - *Inputs:* Receives binary PDF from "On form submission".  
  - *Outputs:* Outputs loaded document text for further processing.  
  - *Edge Cases:* PDF parsing errors, empty or corrupted files.  
  - *Version:* 1.

- **Recursive Character Text Splitter**  
  - *Type & Role:* Text splitter; breaks the loaded document text into chunks for efficient embedding.  
  - *Configuration:* Chunk size 200 characters, 50 characters overlap to maintain context continuity.  
  - *Inputs:* Receives document text from "Default Data Loader".  
  - *Outputs:* Outputs text chunks for embedding.  
  - *Edge Cases:* Very short documents may produce fewer chunks; overlapping could cause redundant data.  
  - *Version:* 1.

- **Embeddings Ollama**  
  - *Type & Role:* Embedding generator; converts text chunks into vector embeddings using Ollama local embedding model.  
  - *Configuration:* Uses model `"mxbai-embed-large:latest"` hosted locally.  
  - *Credentials:* Ollama API credentials configured for local Ollama service.  
  - *Inputs:* Receives text chunks from "Recursive Character Text Splitter".  
  - *Outputs:* Outputs vector embeddings for storage.  
  - *Edge Cases:* Model unavailability, API authentication failure, embedding latency.  
  - *Version:* 1.

- **Qdrant Vector Store**  
  - *Type & Role:* Vector database node; inserts embeddings into Qdrant collection for semantic search.  
  - *Configuration:* Mode set to `insert`, target collection named `"rag_collection"`.  
  - *Credentials:* Local Qdrant API credentials.  
  - *Inputs:* Receives embeddings from "Embeddings Ollama" and document data from "Default Data Loader".  
  - *Outputs:* Confirms insertion success.  
  - *Edge Cases:* Connection failure to Qdrant, malformed embeddings, collection not existing.  
  - *Version:* 1.2.

- **Sticky Note**  
  - *Type & Role:* Documentation node; visually annotates the data ingestion block with the note:  
    "**Data Ingestion**  
    Add data to the semantic database".  
  - *Inputs/Outputs:* None.

---

#### 2.2 RAG Chatbot Block

**Overview:**  
This block listens for incoming chat messages, uses an AI agent equipped with memory, a language model, and a retrieval tool connected to the Qdrant vector store to generate contextually relevant answers grounded in the ingested documents.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- Ollama Chat Model  
- Simple Memory  
- Qdrant Vector Store1 (Retriever mode)  
- Embeddings Ollama1  
- Sticky Note (RAG Chatbot description)

**Node Details:**

- **When chat message received**  
  - *Type & Role:* Chat Trigger node; entry point for initiating chatbot interaction on message reception.  
  - *Configuration:* Default options for receiving chat messages.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Outputs chat message data to "AI Agent".  
  - *Edge Cases:* Webhook connectivity issues, malformed chat messages.  
  - *Version:* 1.1.

- **AI Agent**  
  - *Type & Role:* Core conversational AI agent node; orchestrates language model, memory, and retrieval tool to answer queries.  
  - *Configuration:* System message instructs the agent as a helpful assistant with access to a semantic retrieval tool and to always provide arguments when executing the tool.  
  - *Inputs:* Receives chat messages from "When chat message received" and tool outputs from "Qdrant Vector Store1".  
  - *Outputs:* Produces chatbot responses.  
  - *Edge Cases:* Tool invocation errors, response generation timeouts, memory overflows.  
  - *Version:* 2.

- **Ollama Chat Model**  
  - *Type & Role:* Language model node; generates natural language responses.  
  - *Configuration:* Default settings using local Ollama chat model.  
  - *Credentials:* Uses same local Ollama API credentials as embeddings.  
  - *Inputs:* Feeds language model input from "AI Agent".  
  - *Outputs:* Provides language model output back to "AI Agent".  
  - *Edge Cases:* Model unavailability, latency, API failures.  
  - *Version:* 1.

- **Simple Memory**  
  - *Type & Role:* Conversational memory buffer to maintain context across chat turns.  
  - *Configuration:* Default windowed buffer memory with no additional params.  
  - *Inputs:* Stores conversation history for "AI Agent".  
  - *Outputs:* Supplies memory context to "AI Agent".  
  - *Edge Cases:* Memory size limits, loss of context on restart.  
  - *Version:* 1.3.

- **Qdrant Vector Store1**  
  - *Type & Role:* Vector database retriever; acts as a semantic search tool for the AI agent to query the document collection.  
  - *Configuration:* Mode set to `retrieve-as-tool` with tool name `"retriever"` and description. Uses same collection `"rag_collection"`.  
  - *Credentials:* Local Qdrant API.  
  - *Inputs:* Receives queries from "AI Agent".  
  - *Outputs:* Returns relevant document embeddings to "AI Agent".  
  - *Edge Cases:* Query failures, empty results, connection errors.  
  - *Version:* 1.2.

- **Embeddings Ollama1**  
  - *Type & Role:* Embeddings model used to compute vector queries compatible with Qdrant retriever.  
  - *Configuration:* Same embedding model and credentials as ingestion block.  
  - *Inputs:* Inputs from "Qdrant Vector Store1" for embedding queries.  
  - *Outputs:* Embeddings for retrieval.  
  - *Edge Cases:* Same as ingestion embeddings node.  
  - *Version:* 1.

- **Sticky Note1**  
  - *Type & Role:* Documentation node annotating the chatbot block:  
    "**RAG Chatbot**  
    Chat with your data".  
  - *Inputs/Outputs:* None.

---

### 3. Summary Table

| Node Name               | Node Type                                      | Functional Role                          | Input Node(s)              | Output Node(s)            | Sticky Note                                                         |
|-------------------------|------------------------------------------------|----------------------------------------|----------------------------|---------------------------|---------------------------------------------------------------------|
| On form submission      | Form Trigger                                   | Entry point for PDF upload              | None                       | Qdrant Vector Store        |                                                                     |
| Default Data Loader     | Document Default Data Loader                    | Loads PDF binary as text                 | On form submission          | Qdrant Vector Store        |                                                                     |
| Recursive Character Text Splitter | Text Splitter Recursive Character       | Splits document text into chunks         | Default Data Loader         | Embeddings Ollama          |                                                                     |
| Embeddings Ollama       | Embeddings Ollama                              | Generates vector embeddings (ingestion) | Recursive Character Text Splitter | Qdrant Vector Store        |                                                                     |
| Qdrant Vector Store     | Vector Store Qdrant                            | Inserts embeddings into Qdrant DB        | On form submission, Default Data Loader, Embeddings Ollama | None                      |                                                                     |
| Sticky Note             | Sticky Note                                    | Annotates Data Ingestion block           | None                       | None                      | **Data Ingestion** Add data to the semantic database                |
| When chat message received | Chat Trigger                                  | Entry point for chat messages            | None                       | AI Agent                   |                                                                     |
| AI Agent                | AI Agent                                       | Orchestrates chatbot response generation | When chat message received, Qdrant Vector Store1 | Ollama Chat Model, Simple Memory, Qdrant Vector Store1 |                                                                     |
| Ollama Chat Model       | LM Chat Ollama                                 | Generates language model responses       | AI Agent                   | AI Agent                   |                                                                     |
| Simple Memory           | Memory Buffer Window                           | Maintains conversational context         | AI Agent                   | AI Agent                   |                                                                     |
| Embeddings Ollama1      | Embeddings Ollama                              | Generates embeddings for retrieval queries| Qdrant Vector Store1       | Qdrant Vector Store1       |                                                                     |
| Qdrant Vector Store1    | Vector Store Qdrant                            | Retrieves relevant vectors as a tool     | AI Agent                   | AI Agent                   |                                                                     |
| Sticky Note1            | Sticky Note                                    | Annotates RAG Chatbot block               | None                       | None                      | **RAG Chatbot** Chat with your data                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named `"On form submission"`:  
   - Set form title to `"Add your file here"`.  
   - Add one required file upload field labeled `"File"`.  
   - Restrict accepted file types to `.pdf`.  
   - This node will trigger the workflow upon PDF upload.

2. **Add a Document Default Data Loader node** named `"Default Data Loader"`:  
   - Configure to load data from the binary input of the uploaded PDF (`On form submission`).  
   - Connect `"On form submission"` → `"Default Data Loader"` (binary data).

3. **Add a Recursive Character Text Splitter node** named `"Recursive Character Text Splitter"`:  
   - Set chunk size to `200` characters and overlap to `50` characters.  
   - Connect `"Default Data Loader"` → `"Recursive Character Text Splitter"`.

4. **Add an Embeddings Ollama node** named `"Embeddings Ollama"`:  
   - Set model to `"mxbai-embed-large:latest"`.  
   - Configure credentials for local Ollama API service.  
   - Connect `"Recursive Character Text Splitter"` → `"Embeddings Ollama"`.

5. **Add a Qdrant Vector Store node** named `"Qdrant Vector Store"`:  
   - Set mode to `"insert"`.  
   - Set the target collection to `"rag_collection"`.  
   - Use local Qdrant API credentials.  
   - Connect `"On form submission"` (main), `"Default Data Loader"` (ai_document), and `"Embeddings Ollama"` (ai_embedding) inputs appropriately into `"Qdrant Vector Store"`.

6. **Add a Sticky Note node** named `"Sticky Note"` with content:  
   ```
   ## Data Ingestion
   **Add data to the semantic database**
   ```  
   - Place near the ingestion nodes for clarity.

---

7. **Add a Chat Trigger node** named `"When chat message received"`:  
   - Default configuration to accept chat messages.  
   - This node is the entry point for chat interactions.

8. **Add an AI Agent node** named `"AI Agent"`:  
   - Set system message to:  
     `"You are a helpful assistant. You have access to a tool to retrieve data from a semantic database to answer questions. Always provide arguments when you execute the tool"`.  
   - Connect `"When chat message received"` → `"AI Agent"` (main input).

9. **Add an Ollama Chat Model node** named `"Ollama Chat Model"`:  
   - Use default settings.  
   - Configure Ollama API credentials (same as embedding node).  
   - Connect `"AI Agent"` (languageModel output) → `"Ollama Chat Model"` → `"AI Agent"` (languageModel input).

10. **Add a Simple Memory node** named `"Simple Memory"`:  
    - Use default buffer window memory to maintain conversation state.  
    - Connect `"AI Agent"` (ai_memory output) → `"Simple Memory"` → `"AI Agent"` (ai_memory input).

11. **Add an Embeddings Ollama node** named `"Embeddings Ollama1"`:  
    - Configure identically to the ingestion embedding node (`"mxbai-embed-large:latest"`).  
    - Connect to `"Qdrant Vector Store1"` as embedding input.

12. **Add a Qdrant Vector Store node** named `"Qdrant Vector Store1"`:  
    - Set mode to `"retrieve-as-tool"`.  
    - Set tool name to `"retriever"` with description `"Retrieve data from a semantic database to answer questions"`.  
    - Use collection `"rag_collection"`.  
    - Use local Qdrant API credentials.  
    - Connect `"Embeddings Ollama1"` (ai_embedding) → `"Qdrant Vector Store1"`.  
    - Connect `"Qdrant Vector Store1"` (ai_tool output) → `"AI Agent"` (ai_tool input).

13. **Connect all nodes to complete the flow:**  
    - `"When chat message received"` → `"AI Agent"` → `"Ollama Chat Model"`, `"Simple Memory"`, and `"Qdrant Vector Store1"` (retrieval tool).  
    - The AI Agent orchestrates chatbot responses using the language model, memory, and vector store retrieval.

14. **Add a Sticky Note node** named `"Sticky Note1"` with content:  
    ```
    ## RAG Chatbot
    **Chat with your data**
    ```  
    - Place near the chatbot interaction nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow uses local services for vector embeddings and language modeling powered by Ollama.           | Requires local Ollama API setup and credentials.                 |
| The vector database Qdrant is hosted locally and configured via API credentials.                           | Ensure Qdrant service is running with collection `rag_collection`. |
| The system message in the AI Agent encourages transparent tool use with arguments for retrieval operations.| Helps maintain explainability in chatbot responses.             |
| For PDF ingestion, only `.pdf` files are accepted, with chunk size and overlap parameters optimized for balance between context and performance. | Can be tuned to specific use cases or document sizes.            |