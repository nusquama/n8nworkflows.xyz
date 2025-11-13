Build a GLPI Knowledge Base RAG Pipeline with Google Gemini and PostgreSQL

https://n8nworkflows.xyz/workflows/build-a-glpi-knowledge-base-rag-pipeline-with-google-gemini-and-postgresql-7171


# Build a GLPI Knowledge Base RAG Pipeline with Google Gemini and PostgreSQL

---

### 1. Workflow Overview

This workflow orchestrates a Retrieval-Augmented Generation (RAG) pipeline to build a knowledge base interface for GLPI (an IT asset management tool) using Google Gemini AI models and PostgreSQL vector stores. It is designed to receive chat messages, process them through AI language and embedding models, and query multiple vectorized knowledge stores to provide informed, context-aware responses.

The logical structure can be divided into the following blocks:

- **1.1 Input Reception:** Handles incoming chat messages to trigger the pipeline.
- **1.2 AI Language Model Processing:** Uses Google Gemini’s chat model to generate conversational responses.
- **1.3 Embeddings Generation:** Produces embeddings from the input to query vector stores.
- **1.4 Vector Store Querying:** Interfaces with PostgreSQL vector databases holding GLPI and Confluence knowledge bases.
- **1.5 AI Agent Coordination:** Manages memory and tool integration, combining embeddings, vector stores, and language models to generate final output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming chat messages via a webhook and initiates the AI processing pipeline.

**Nodes Involved:**  
- When chat message received  
- Sticky Note (comment related to this block)

**Node Details:**

- **When chat message received**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Entry point node that listens for chat messages on a webhook endpoint.  
  - *Configuration:* Uses a webhook ID to identify the endpoint; no additional parameters set.  
  - *Inputs:* External HTTP POST requests containing chat data.  
  - *Outputs:* Forwards the chat payload to the AI Agent node.  
  - *Potential Failures:* Webhook misconfiguration, invalid payload format, network issues.  
  - *Notes:* This node enables real-time interaction with the workflow.

- **Sticky Note**  
  - *Type:* n8n Sticky Note  
  - *Role:* Placeholder or commentary, no functional impact.  
  - *Content:* Empty in this workflow but positioned near the chat trigger node, likely for organizational clarity.

---

#### 2.2 AI Language Model Processing

**Overview:**  
This block uses Google Gemini’s chat model to generate responses based on input and contextual information.

**Nodes Involved:**  
- Google Gemini Chat Model  
- AI Agent  

**Node Details:**

- **Google Gemini Chat Model**  
  - *Type:* Langchain Google Gemini Language Model (Chat)  
  - *Role:* Core language model providing conversational AI capabilities.  
  - *Configuration:* Default parameters; relies on credentials for Google Gemini.  
  - *Inputs:* Chat messages and context forwarded from the AI Agent node.  
  - *Outputs:* Chat completions sent back to AI Agent for further processing or response dispatch.  
  - *Failure Modes:* API authentication errors, rate limits, latency/timeouts, malformed input data.  
  - *Version:* Type version 1.

- **AI Agent**  
  - *Type:* Langchain Agent Node  
  - *Role:* Central orchestrator combining memory, language model, and vector store tools to produce final AI responses.  
  - *Configuration:* No explicit parameters; configured via connected nodes for memory, language model, embeddings, and tools.  
  - *Inputs:* Receives initial trigger from "When chat message received" node, memory from "Simple Memory", language model from "Google Gemini Chat Model", and tools from vector stores.  
  - *Outputs:* Final AI-generated answers delivered downstream or back to chat interface.  
  - *Potential Issues:* Misconfiguration of connections, missing credentials, failure in upstream nodes affect output.  
  - *Version:* 1.8.

---

#### 2.3 Embeddings Generation

**Overview:**  
Generates vector embeddings from input texts using Google Gemini’s embedding model to query vector databases effectively.

**Nodes Involved:**  
- Embeddings Google Gemini1  
- Embeddings Google Gemini3  

**Node Details:**

- **Embeddings Google Gemini1**  
  - *Type:* Langchain Google Gemini Embeddings  
  - *Role:* Converts input text into vector embeddings for the "CONHECIMENTO_TI_GLPI" vector store.  
  - *Configuration:* Default setup, linked to the GLPI knowledge base vector store node.  
  - *Inputs:* Data from AI Agent or vector store query requests.  
  - *Outputs:* Embeddings passed to the "CONHECIMENTO_TI_GLPI" node.  
  - *Failure Modes:* API credential errors, input text encoding issues.

- **Embeddings Google Gemini3**  
  - *Type:* Langchain Google Gemini Embeddings  
  - *Role:* Generates embeddings for the Confluence knowledge base queries.  
  - *Configuration:* Similar to Embeddings Google Gemini1 but linked to "CONFLUENCE_TI_CONFLUENCE_SGU_GPL".  
  - *Inputs:* Text data from AI Agent or downstream logic.  
  - *Outputs:* Embeddings forwarded to "CONFLUENCE_TI_CONFLUENCE_SGU_GPL".  
  - *Failure Modes:* Same as above.

---

#### 2.4 Vector Store Querying

**Overview:**  
Queries PostgreSQL-based vector stores for relevant documents or knowledge snippets based on embeddings.

**Nodes Involved:**  
- CONHECIMENTO_TI_GLPI  
- CONFLUENCE_TI_CONFLUENCE_SGU_GPL  

**Node Details:**

- **CONHECIMENTO_TI_GLPI**  
  - *Type:* Langchain PostgreSQL Vector Store (PGVector)  
  - *Role:* Stores and retrieves GLPI knowledge base vectors.  
  - *Configuration:* Connected to "Embeddings Google Gemini1" for embedding input; configured with PostgreSQL credentials and schema details (not visible here).  
  - *Inputs:* Embeddings vectors.  
  - *Outputs:* Search results fed into AI Agent as a tool for knowledge retrieval.  
  - *Failure Modes:* Database connection issues, query errors, credential expiration.

- **CONFLUENCE_TI_CONFLUENCE_SGU_GPL**  
  - *Type:* Langchain PostgreSQL Vector Store (PGVector)  
  - *Role:* Stores and retrieves Confluence knowledge base vectors.  
  - *Configuration:* Connected to "Embeddings Google Gemini3"; no outputs configured in this workflow (empty output array).  
  - *Inputs:* Embeddings vectors.  
  - *Outputs:* None connected downstream here, potentially used internally by AI Agent.  
  - *Failure Modes:* Same as above.

---

#### 2.5 AI Agent Coordination and Memory Management

**Overview:**  
Manages chat memory and integrates tools (vector stores) and models to provide coherent AI responses.

**Nodes Involved:**  
- Simple Memory  
- AI Agent (already discussed)

**Node Details:**

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains conversational context with a sliding window memory buffer to retain recent interactions for coherent dialogue.  
  - *Configuration:* Default; no specific window size shown.  
  - *Inputs:* Likely connected to AI Agent to store and retrieve chat history.  
  - *Outputs:* Provides memory context back to AI Agent.  
  - *Failure Modes:* Memory overflow or mismanagement, context loss.

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                         | Input Node(s)                  | Output Node(s)                 | Sticky Note                         |
|-------------------------------|----------------------------------------|---------------------------------------|-------------------------------|-------------------------------|-----------------------------------|
| When chat message received     | Langchain Chat Trigger                  | Entry point webhook for chat messages | External HTTP requests         | AI Agent                      |                                   |
| Sticky Note                   | n8n Sticky Note                        | Comment / organization                 | None                          | None                          |                                   |
| Google Gemini Chat Model       | Langchain Google Gemini LM Chat        | Generates AI chat responses            | AI Agent                      | AI Agent                      |                                   |
| AI Agent                      | Langchain Agent                        | Orchestrates memory, tools, model     | When chat message received, Simple Memory, Google Gemini Chat Model, Vector Stores | None                          |                                   |
| Embeddings Google Gemini1      | Langchain Google Gemini Embeddings      | Generates embeddings for GLPI store    | AI Agent                      | CONHECIMENTO_TI_GLPI          |                                   |
| Embeddings Google Gemini3      | Langchain Google Gemini Embeddings      | Generates embeddings for Confluence    | AI Agent                      | CONFLUENCE_TI_CONFLUENCE_SGU_GPL |                                   |
| CONHECIMENTO_TI_GLPI           | Langchain PostgreSQL Vector Store (PGVector) | Vector store for GLPI knowledge base  | Embeddings Google Gemini1     | AI Agent                      |                                   |
| CONFLUENCE_TI_CONFLUENCE_SGU_GPL | Langchain PostgreSQL Vector Store (PGVector) | Vector store for Confluence knowledge | Embeddings Google Gemini3     | None                          |                                   |
| Simple Memory                 | Langchain Memory Buffer Window          | Maintains chat context memory          | AI Agent                      | AI Agent                      |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Trigger Node:**  
   - Type: Langchain Chat Trigger  
   - Configure webhook ID or let n8n generate one.  
   - Purpose: Receive chat messages to initiate the workflow.

2. **Add an AI Agent Node:**  
   - Type: Langchain Agent  
   - No special parameters; will connect to memory, tools, and language model nodes.  

3. **Add a Google Gemini Chat Model Node:**  
   - Type: Langchain Google Gemini LM Chat  
   - Configure with valid Google Gemini API credentials.  
   - Connect its output to the AI Agent node’s `ai_languageModel` input.

4. **Add Simple Memory Node:**  
   - Type: Langchain Memory Buffer Window  
   - Default settings suffice unless specific window size needed.  
   - Connect its output to the AI Agent node’s `ai_memory` input.

5. **Create Embeddings Nodes:**  
   - Add two Langchain Google Gemini Embeddings nodes named "Embeddings Google Gemini1" and "Embeddings Google Gemini3".  
   - Configure each with Google Gemini API credentials.

6. **Create PostgreSQL Vector Store Nodes:**  
   - Add two Langchain PostgreSQL Vector Store (PGVector) nodes named "CONHECIMENTO_TI_GLPI" and "CONFLUENCE_TI_CONFLUENCE_SGU_GPL".  
   - Provide PostgreSQL credentials and connection details (host, port, database, schema).  
   - Set each node to use the corresponding embeddings node as input:
     - "Embeddings Google Gemini1" → "CONHECIMENTO_TI_GLPI"  
     - "Embeddings Google Gemini3" → "CONFLUENCE_TI_CONFLUENCE_SGU_GPL"

7. **Connect Vector Stores to AI Agent:**  
   - Connect "CONHECIMENTO_TI_GLPI" node output to AI Agent’s `ai_tool` input.  
   - Connect "CONFLUENCE_TI_CONFLUENCE_SGU_GPL" node output to AI Agent’s `ai_tool` input as well (even if no direct output configured in original, for completeness).

8. **Wire the Trigger to AI Agent:**  
   - Connect "When chat message received" node’s output to AI Agent’s main input.

9. **Verify Credentials:**  
   - Ensure Google Gemini API credentials are correctly configured in all relevant nodes (Chat Model and Embeddings).  
   - Ensure PostgreSQL credentials and access rights are correct for vector store nodes.

10. **Deploy Workflow:**  
    - Activate the webhook URL from the trigger node to start receiving chat messages.  
    - Test the pipeline end-to-end using sample chat input.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                      |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| Google Gemini models are used for both embeddings and chat completions, ensuring consistency.   | Workflow uses Google Gemini API credentials; ensure access and quota management.    |
| Vector stores leverage PostgreSQL with PGVector extension for efficient similarity search.      | Requires PostgreSQL with PGVector installed and properly configured.                |
| The AI Agent node integrates multiple tools and memory to enable RAG-style question answering. | n8n Langchain agent documentation: https://docs.n8n.io/integrations/builtin/agents/ |
| Keep webhook endpoints secure and monitor usage to prevent unauthorized access.                  | Use n8n's built-in webhook security features or proxy with authentication.          |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---