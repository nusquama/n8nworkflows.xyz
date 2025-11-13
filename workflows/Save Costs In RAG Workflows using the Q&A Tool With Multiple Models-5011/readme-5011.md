Save Costs In RAG Workflows using the Q&A Tool With Multiple Models

https://n8nworkflows.xyz/workflows/save-costs-in-rag-workflows-using-the-q-a-tool-with-multiple-models-5011


# Save Costs In RAG Workflows using the Q&A Tool With Multiple Models

### 1. Workflow Overview

This n8n workflow is designed to optimize costs in Retrieval-Augmented Generation (RAG) workflows by leveraging a Question & Answer (Q&A) tool that uses multiple AI language models with different cost profiles. The workflow enables users to upload documents, embed and store their data in-memory as vectors, and then interactively query this data using a chat interface powered by both an expensive, high-quality AI model and a cheaper, faster alternative. The logical structure includes two main functional blocks:

- **1.1 Data Upload & Vector Storage ("Load Data" flow):** Handles user file uploads, processes documents for embedding generation, and inserts vectorized data into an in-memory vector store.
- **1.2 Chat-Based Retrieval & Answering ("Retriever" flow):** Listens for chat messages, queries the vector store for relevant data, and answers questions by selecting between an expensive and a cheap AI model via a Q&A tool.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Upload & Vector Storage ("Load Data" flow)

- **Overview:**  
  This block allows users to upload PDF or CSV files, loads the binary document data, generates embeddings using OpenAI, and stores the resulting vectors in an in-memory vector store for quick retrieval.

- **Nodes Involved:**  
  - Upload your file here  
  - Default Data Loader  
  - Embeddings OpenAI  
  - Insert Data to Store  
  - Sticky Note  
  - Sticky Note1  

- **Node Details:**

  1. **Upload your file here**  
     - Type: Form Trigger  
     - Role: Entry point for file upload via a web form  
     - Config: Accepts PDF and CSV files only; single or multiple uploads allowed; form titled "Upload your data to test RAG"  
     - Inputs: HTTP webhook trigger  
     - Outputs: Binary file data forwarded downstream  
     - Edge Cases: File type validation failure, large file uploads causing timeouts  

  2. **Default Data Loader**  
     - Type: Document Data Loader (LangChain)  
     - Role: Processes uploaded binary documents into a suitable format for embedding generation  
     - Config: Data type set to "binary" for document input  
     - Inputs: Binary data from upload node  
     - Outputs: Structured document objects  
     - Edge Cases: Unsupported file content, corrupted files  

  3. **Embeddings OpenAI**  
     - Type: OpenAI Embeddings node (LangChain)  
     - Role: Converts documents into vector embeddings for semantic search  
     - Config: Uses OpenAI API credentials; no extra options configured  
     - Inputs: Document objects from loader  
     - Outputs: Vectors representing document semantics  
     - Edge Cases: API authentication errors, rate limiting, embedding generation errors  

  4. **Insert Data to Store**  
     - Type: Vector Store In-Memory (LangChain)  
     - Role: Inserts embeddings into an in-memory vector store under key `vector_store_key`  
     - Config: Mode set to "insert"; memory key explicitly set to `vector_store_key` to allow retrieval later  
     - Inputs: Embeddings and documents  
     - Outputs: Confirmation of insert  
     - Edge Cases: Memory overflow if too much data; concurrency issues if multiple inserts happen simultaneously  

  5. **Sticky Note**  
     - Type: Sticky note  
     - Role: Provides user guidance and documentation summary of how to load data and use the retriever flow  
     - Content Highlights: Explains two main flows, quick start instructions, and links to official RAG docs [https://docs.n8n.io/advanced-ai/rag-in-n8n/]  

  6. **Sticky Note1**  
     - Type: Sticky note  
     - Role: Label for the "Load Data" flow block for visual clarity  

---

#### 1.2 Chat-Based Retrieval & Answering ("Retriever" flow)

- **Overview:**  
  This block listens for incoming chat messages, queries the vector store for relevant context, and uses a Q&A tool that intelligently switches between a cheap and an expensive AI model to answer questions based on the stored data.

- **Nodes Involved:**  
  - When chat message received  
  - AI Agent  
  - Expensive model  
  - Cheap Model  
  - Query Data Tool  
  - Answer questions with a vector store  
  - Sticky Note2  

- **Node Details:**

  1. **When chat message received**  
     - Type: LangChain Chat Trigger  
     - Role: Webhook entry point to receive chat messages for question answering  
     - Config: Default options; webhook ID configured for external invocation  
     - Inputs: Incoming chat messages via HTTP webhook  
     - Outputs: Chat message data forwarded downstream  
     - Edge Cases: Webhook timeout, malformed messages  

  2. **AI Agent**  
     - Type: LangChain Agent  
     - Role: Orchestrates the Q&A process using tools and language models  
     - Config: No options set; uses linked tools and models as inputs  
     - Inputs: Chat messages from trigger, AI tools and language models from other nodes  
     - Outputs: Final answer to chat user  
     - Edge Cases: Agent logic errors, tool invocation failures  

  3. **Expensive model**  
     - Type: LangChain OpenAI Chat Model  
     - Role: Provides high-quality, expensive AI responses (GPT-4.1)  
     - Config: Model explicitly set to "gpt-4.1"  
     - Credentials: OpenAI API key  
     - Inputs: Used by AI Agent as one possible language model  
     - Outputs: Language model completions  
     - Edge Cases: API rate limits, authentication failures, latency  

  4. **Cheap Model**  
     - Type: LangChain OpenAI Chat Model  
     - Role: Provides cheaper, faster AI responses (GPT-4.1-mini)  
     - Config: Model explicitly set to "gpt-4.1-mini"  
     - Credentials: OpenAI API key  
     - Inputs: Used by Q&A tool for less expensive answers  
     - Outputs: Language model completions  
     - Edge Cases: Same as Expensive Model but generally faster and cheaper  

  5. **Query Data Tool**  
     - Type: Vector Store In-Memory (LangChain)  
     - Role: Retrieves relevant vectors from the in-memory store keyed by `vector_store_key`  
     - Config: References the same memory key used for insertion  
     - Inputs: Triggered by AI Agent or Q&A tool to fetch context  
     - Outputs: Retrieved relevant vectors/documents  
     - Edge Cases: Empty or missing vector store, memory expiration  

  6. **Answer questions with a vector store**  
     - Type: LangChain Tool Vector Store  
     - Role: Q&A tool that uses retrieved vectors as context to answer questions  
     - Config: Description notes the data contains custom knowledge  
     - Inputs: Vector store outputs and cheap model completions  
     - Outputs: Answers fed to AI Agent  
     - Edge Cases: Tool errors if vector store empty or malformed data  

  7. **Sticky Note2**  
     - Type: Sticky note  
     - Role: Visual label for the "Retriever" flow block  

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                                  | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                        |
|----------------------------|----------------------------------|-------------------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------|
| Upload your file here       | Form Trigger                     | Capture user file uploads                        | (Webhook)                  | Insert Data to Store        |                                                                                                                    |
| Default Data Loader         | Document Data Loader (LangChain) | Process uploaded files into documents           | Insert Data to Store        | Embeddings OpenAI           |                                                                                                                    |
| Embeddings OpenAI           | OpenAI Embeddings (LangChain)    | Generate vector embeddings from documents       | Default Data Loader         | Insert Data to Store, Query Data Tool |                                                                                                                    |
| Insert Data to Store        | Vector Store In-Memory (LangChain)| Insert embeddings into in-memory vector store   | Upload your file here, Default Data Loader, Embeddings OpenAI |                             |                                                                                                                    |
| Sticky Note                 | Sticky Note                     | Readme and usage instructions                    |                            |                            | "### Readme\nLoad your data into a vector database with the 📚 **Load Data** flow, and then use your data as chat context with the 🐕 **Retriever** flow.\n\n**Quick start**\n1. Click on the `Execute Workflow` button to run the 📚 **Load Data** flow.\n2. Click on `Open Chat` button to run the 🐕 **Retriever** flow. Then ask a question about content from your document(s)\n\nFor more info, check our [docs on RAG in n8n](https://docs.n8n.io/advanced-ai/rag-in-n8n/)" |
| Sticky Note1                | Sticky Note                     | Label for Load Data flow                         |                            |                            | "### 📚 Load Data Flow"                                                                                            |
| When chat message received  | LangChain Chat Trigger           | Receive chat input messages                      | (Webhook)                  | AI Agent                   |                                                                                                                    |
| AI Agent                   | LangChain Agent                  | Orchestrate Q&A using tools and models          | When chat message received, Expensive model, Cheap Model, Answer questions with a vector store |                            |                                                                                                                    |
| Expensive model             | LangChain OpenAI Chat Model      | High-cost, high-quality AI language model       |                            | AI Agent                   |                                                                                                                    |
| Cheap Model                 | LangChain OpenAI Chat Model      | Lower-cost, smaller AI model                      |                            | Answer questions with a vector store |                                                                                                                    |
| Query Data Tool             | Vector Store In-Memory (LangChain)| Retrieve vectors from in-memory store            | Embeddings OpenAI          | Answer questions with a vector store |                                                                                                                    |
| Answer questions with a vector store | LangChain Tool Vector Store | Q&A tool using vector store context              | Query Data Tool, Cheap Model | AI Agent                   | "The data contains custom knowledge"                                                                               |
| Sticky Note2                | Sticky Note                     | Label for Retriever flow                         |                            |                            | "### 🐕 2. Retriever Flow"                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Upload your file here"):**  
   - Node Type: Form Trigger  
   - Configure form titled "Upload your data to test RAG"  
   - Add file upload field accepting `.pdf, .csv` files, required  
   - Save and note webhook URL  

2. **Add Document Data Loader Node ("Default Data Loader"):**  
   - Node Type: LangChain Document Default Data Loader  
   - Set `dataType` parameter to "binary"  
   - Connect input from Form Trigger node  

3. **Add OpenAI Embeddings Node ("Embeddings OpenAI"):**  
   - Node Type: LangChain Embeddings OpenAI  
   - Authenticate with OpenAI API credential configured in n8n  
   - Connect input from Document Data Loader node  

4. **Add Vector Store In-Memory Node ("Insert Data to Store"):**  
   - Node Type: LangChain Vector Store In-Memory  
   - Set mode to "insert"  
   - Set memory key to string `vector_store_key` (list mode)  
   - Connect inputs from Form Trigger, Document Data Loader, and Embeddings OpenAI nodes  
   - This stores the vectors for retrieval  

5. **Add Sticky Notes ("Sticky Note" and "Sticky Note1"):**  
   - Add two Sticky Note nodes  
   - First: Content with Readme instructions and link to [RAG docs](https://docs.n8n.io/advanced-ai/rag-in-n8n/)  
   - Second: Label "📚 Load Data Flow"  

6. **Add LangChain Chat Trigger Node ("When chat message received"):**  
   - Node Type: LangChain Chat Trigger  
   - Configure webhook to receive chat messages  

7. **Add LangChain Agent Node ("AI Agent"):**  
   - Node Type: LangChain Agent  
   - Connect input from Chat Trigger node  

8. **Add Expensive AI Language Model Node ("Expensive model"):**  
   - Node Type: LangChain OpenAI Chat Model  
   - Model parameter: "gpt-4.1"  
   - Connect to AI Agent node as language model input  
   - Authenticate with OpenAI API  

9. **Add Cheap AI Language Model Node ("Cheap Model"):**  
   - Node Type: LangChain OpenAI Chat Model  
   - Model parameter: "gpt-4.1-mini"  
   - Connect to Q&A tool (see next step)  
   - Authenticate with OpenAI API  

10. **Add Vector Store In-Memory Node ("Query Data Tool"):**  
    - Node Type: LangChain Vector Store In-Memory  
    - Set memory key to `vector_store_key` (same as insertion)  
    - Connect input from Embeddings OpenAI node (for embedding reference)  

11. **Add Q&A Tool Node ("Answer questions with a vector store"):**  
    - Node Type: LangChain Tool Vector Store  
    - Description: "The data contains custom knowledge"  
    - Connect vector store input from Query Data Tool node  
    - Connect AI language model input from Cheap Model node  
    - Connect output to AI Agent node as a tool  

12. **Connect AI Agent node outputs to final response:**  
    - Ensure AI Agent node outputs final chat response  

13. **Add Sticky Note ("Sticky Note2"):**  
    - Label "🐕 2. Retriever Flow"  

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                   |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow provides a cost-optimized approach to RAG by combining a cheap and expensive language model via a vector store Q&A tool. | Workflow purpose                                  |
| For detailed explanation of RAG integration in n8n, see the official docs: [https://docs.n8n.io/advanced-ai/rag-in-n8n/]          | Official documentation link                       |
| Quick start instructions are embedded inside the workflow as Sticky Notes for user guidance.                                       | Embedded documentation inside workflow            |

---

*Disclaimer:* The provided text stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.