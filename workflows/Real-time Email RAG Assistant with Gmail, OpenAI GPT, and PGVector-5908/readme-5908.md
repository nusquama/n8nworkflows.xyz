Real-time Email RAG Assistant with Gmail, OpenAI GPT, and PGVector

https://n8nworkflows.xyz/workflows/real-time-email-rag-assistant-with-gmail--openai-gpt--and-pgvector-5908


# Real-time Email RAG Assistant with Gmail, OpenAI GPT, and PGVector

### 1. Workflow Overview

This workflow implements a Real-time Email Retrieval-Augmented Generation (RAG) assistant, designed to automatically ingest new Gmail emails into a vector database and enable intelligent conversational search through an AI chat interface. It targets use cases involving enhanced email management, semantic search, and AI-driven email query answering.

The workflow is logically divided into two main blocks:

- **1.1 Email Ingestion & Vectorization:** Watches for new emails, retrieves full email data, processes and splits the email content, generates embeddings with OpenAI, and stores them in a PostgreSQL database using the PGVector extension.

- **1.2 Conversational RAG Agent:** Listens for incoming chat queries, performs vector search on stored email embeddings to retrieve relevant documents, and uses an OpenAI chat model combined with the retrieval results to generate helpful AI responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Ingestion & Vectorization

- **Overview:**  
  This block automates the ingestion of new Gmail emails. It retrieves detailed email content, formats it for semantic processing, splits long text into manageable chunks, generates vector embeddings via OpenAI, and stores them into a PostgreSQL vector store for later retrieval.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Get Mail Data  
  - Enhanced Default Data Loader  
  - Recursive Character Text Splitter2  
  - Embeddings OpenAI3  
  - Postgres PGVector Store3  
  - Sticky Note2 (Documentation)

- **Node Details:**

  1. **Gmail Trigger**  
     - Type: Trigger node for Gmail  
     - Role: Watches for new incoming emails, polling every minute  
     - Configuration: No filters set, polls every minute to catch new emails promptly  
     - Inputs: None (trigger)  
     - Outputs: Passes new email metadata (including ID) downstream  
     - Edge Cases: Gmail API rate limits, authentication expiry, no new emails found  
     - Version: 1.2

  2. **Get Mail Data**  
     - Type: Gmail API node  
     - Role: Retrieves full details of the email using the ID received from the trigger  
     - Configuration: Operation “get” with messageId dynamically set from trigger data  
     - Inputs: Email metadata from Gmail Trigger  
     - Outputs: Full email data including headers, body, sender, recipients  
     - Edge Cases: Email deleted before retrieval, API errors, incomplete data  
     - Version: 2.1

  3. **Enhanced Default Data Loader**  
     - Type: Document preparation node (Langchain)  
     - Role: Formats the email content into a structured JSON string with metadata for embedding  
     - Configuration:  
       - Metadata includes `data_source` set to "gmail" and `created_at` with the email date  
       - JSON body includes date, from/to addresses, subject, and plain text body of email  
     - Inputs: Full email JSON from Get Mail Data  
     - Outputs: Structured document ready for text splitting and embedding  
     - Edge Cases: Missing fields in email JSON, formatting expression failures  
     - Version: 1

  4. **Recursive Character Text Splitter2**  
     - Type: Text splitter node (Langchain)  
     - Role: Splits the document text into chunks (max 2000 characters with 200 overlap) to optimize embedding generation and retrieval  
     - Configuration: Markdown split code, chunk size 2000, overlap 200  
     - Inputs: Document from Enhanced Default Data Loader  
     - Outputs: Array of text chunks for embedding  
     - Edge Cases: Very short emails (may produce only one chunk), malformed markdown  
     - Version: 1

  5. **Embeddings OpenAI3**  
     - Type: OpenAI embeddings node (Langchain)  
     - Role: Generates vector embeddings for each text chunk using OpenAI's API  
     - Configuration: Default options; uses configured OpenAI API credentials  
     - Inputs: Text chunks from Recursive Character Text Splitter2  
     - Outputs: Embeddings vectors for each chunk  
     - Edge Cases: API rate limits, network errors, invalid API key  
     - Version: 1.1

  6. **Postgres PGVector Store3**  
     - Type: Vector store node for PostgreSQL PGVector (Langchain)  
     - Role: Inserts the generated embeddings along with metadata into the `emails_vector_history` table  
     - Configuration: Mode “insert”, table name `emails_vector_history`  
     - Inputs: Embeddings from Embeddings OpenAI3, document metadata from Enhanced Default Data Loader  
     - Outputs: Confirmation of successful insertion  
     - Edge Cases: Database connection failure, SQL errors, duplicate entries  
     - Version: 1.1

  7. **Sticky Note2**  
     - Type: Documentation node  
     - Content: "Watch Trigger (Email) - New Email\nGet new Email -> Extract the text -> Add to Vector Store"  
     - Purpose: Provides a visual summary of this block’s process

---

#### 2.2 Conversational RAG Agent

- **Overview:**  
  This block handles incoming chat messages, performs semantic search over the stored email embeddings, and replies with AI-generated answers based on retrieved documents using a RAG strategy.

- **Nodes Involved:**  
  - When chat message received  
  - New RAG Agent  
  - Postgres PGVector Store  
  - Embeddings OpenAI  
  - OpenAI Chat Model  
  - Sticky Note (Documentation)

- **Node Details:**

  1. **When chat message received**  
     - Type: Chat trigger node (Langchain)  
     - Role: Listens for incoming chat messages to start the RAG query process  
     - Configuration: Default, webhook ID configured to receive messages  
     - Inputs: External chat client or system  
     - Outputs: Chat messages forwarded to RAG agent  
     - Edge Cases: Webhook connectivity, malformed chat input  
     - Version: 1.1

  2. **New RAG Agent**  
     - Type: Langchain Agent node  
     - Role: Orchestrates retrieval-augmented generation by combining vector search results and language model response  
     - Configuration:  
       - System message instructs the assistant to fetch data from RAG and answer helpfully  
       - Injects current date for time-aware responses  
     - Inputs: Chat messages, vector search tool, language model  
     - Outputs: AI-generated chat responses  
     - Edge Cases: Model timeout, empty retrieval results, ambiguous queries  
     - Version: 1.7

  3. **Postgres PGVector Store**  
     - Type: Vector store node for PostgreSQL PGVector (Langchain)  
     - Role: Performs vector similarity search to retrieve top 5 relevant email chunks based on query embeddings  
     - Configuration:  
       - Mode: “retrieve-as-tool” with tool name `emails_vector_search`  
       - Tool description guides the agent to include time frames in queries for accurate results  
       - Table name: `emails_vector_history`  
       - TopK: 5 results returned  
     - Inputs: Query embeddings from Embeddings OpenAI  
     - Outputs: Relevant email documents as retrieval results  
     - Edge Cases: Empty vector store, database connection issues  
     - Version: 1.1

  4. **Embeddings OpenAI**  
     - Type: OpenAI embeddings node (Langchain)  
     - Role: Converts incoming chat message text into embeddings used for vector search  
     - Configuration: Default, with OpenAI API credentials  
     - Inputs: Chat message text from When chat message received  
     - Outputs: Query embeddings for vector store search  
     - Edge Cases: API failure, malformed input  
     - Version: 1.1

  5. **OpenAI Chat Model**  
     - Type: OpenAI chat completion node (Langchain)  
     - Role: Generates the final natural language response based on the RAG agent’s processing  
     - Configuration: Default options, uses OpenAI API credentials  
     - Inputs: Language model input from New RAG Agent  
     - Outputs: AI-generated chat reply  
     - Edge Cases: API rate limits, model errors, incomplete responses  
     - Version: 1

  6. **Sticky Note**  
     - Type: Documentation node  
     - Content: "RAG AI Agent\n\nThis powerful n8n workflow creates an intelligent email search and query system using RAG (Retrieval-Augmented Generation) technology. The system automatically ingests Gmail emails into a vector database and provides conversational AI-powered search capabilities through a chat interface"  
     - Purpose: Provides a high-level explanation of the workflow’s AI assistant capabilities

---

### 3. Summary Table

| Node Name                   | Node Type                            | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                  |
|-----------------------------|------------------------------------|----------------------------------------|------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Gmail Trigger               | gmailTrigger                       | Watches for new Gmail emails            | —                      | Get Mail Data            | Watch Trigger (Email) - New Email<br>Get new Email -> Extract the text -> Add to Vector Store                 |
| Get Mail Data               | gmail                             | Retrieves full email data                | Gmail Trigger           | Postgres PGVector Store3 | Watch Trigger (Email) - New Email<br>Get new Email -> Extract the text -> Add to Vector Store                 |
| Enhanced Default Data Loader| documentDefaultDataLoader          | Formats email content for embedding     | Recursive Character Text Splitter2 (reverse connection, see later) | Recursive Character Text Splitter2 | Watch Trigger (Email) - New Email<br>Get new Email -> Extract the text -> Add to Vector Store                 |
| Recursive Character Text Splitter2 | textSplitterRecursiveCharacterTextSplitter | Splits email text into chunks            | Enhanced Default Data Loader | Embeddings OpenAI3       | Watch Trigger (Email) - New Email<br>Get new Email -> Extract the text -> Add to Vector Store                 |
| Embeddings OpenAI3          | embeddingsOpenAi                  | Generates vector embeddings from chunks | Recursive Character Text Splitter2 | Postgres PGVector Store3 | Watch Trigger (Email) - New Email<br>Get new Email -> Extract the text -> Add to Vector Store                 |
| Postgres PGVector Store3    | vectorStorePGVector               | Inserts embeddings into vector DB       | Embeddings OpenAI3      | —                       | Watch Trigger (Email) - New Email<br>Get new Email -> Extract the text -> Add to Vector Store                 |
| When chat message received  | chatTrigger                      | Receives incoming chat messages         | —                      | New RAG Agent            | RAG AI Agent<br>This powerful n8n workflow creates an intelligent email search and query system using RAG tech|
| New RAG Agent               | agent                            | Coordinates retrieval and AI response   | When chat message received | OpenAI Chat Model, Postgres PGVector Store | RAG AI Agent<br>This powerful n8n workflow creates an intelligent email search and query system using RAG tech|
| Postgres PGVector Store     | vectorStorePGVector               | Retrieves relevant embeddings for query | Embeddings OpenAI       | New RAG Agent            | RAG AI Agent<br>This powerful n8n workflow creates an intelligent email search and query system using RAG tech|
| Embeddings OpenAI           | embeddingsOpenAi                  | Creates embeddings from chat queries    | When chat message received | Postgres PGVector Store  | RAG AI Agent<br>This powerful n8n workflow creates an intelligent email search and query system using RAG tech|
| OpenAI Chat Model           | lmChatOpenAi                     | Generates AI chat responses              | New RAG Agent           | New RAG Agent (output)   | RAG AI Agent<br>This powerful n8n workflow creates an intelligent email search and query system using RAG tech|
| Sticky Note2                | stickyNote                      | Documentation for email ingestion block | —                      | —                       | Watch Trigger (Email) - New Email<br>Get new Email -> Extract the text -> Add to Vector Store                 |
| Sticky Note                 | stickyNote                      | Documentation for RAG agent block       | —                      | —                       | RAG AI Agent<br>This powerful n8n workflow creates an intelligent email search and query system using RAG tech|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Gmail Trigger node**  
   - Set to poll every minute with no filters to watch for new incoming emails.

2. **Create a Gmail node ("Get Mail Data")**  
   - Operation: `get`  
   - Message ID: Set to `={{ $json.id }}` from the Gmail Trigger output to fetch full email details.

3. **Create an Enhanced Default Data Loader node**  
   - Input: Use the full email JSON from "Get Mail Data"  
   - Configure JSON data as an expression that formats the email details including date, from/to addresses, subject, and plain text body.  
   - Add metadata fields:  
     - `data_source` = "gmail"  
     - `created_at` = `={{ $json.date }}`  

4. **Create a Recursive Character Text Splitter node ("Recursive Character Text Splitter2")**  
   - Set split code to "markdown"  
   - Chunk size: 2000 characters  
   - Chunk overlap: 200 characters  
   - Input: Output of Enhanced Default Data Loader

5. **Create an OpenAI Embeddings node ("Embeddings OpenAI3")**  
   - Use OpenAI API credentials  
   - Input: Text chunks from the splitter node

6. **Create a PostgreSQL PGVector Store node ("Postgres PGVector Store3")**  
   - Mode: `insert`  
   - Table name: `emails_vector_history`  
   - Input: Embeddings from OpenAI Embeddings node and metadata from Enhanced Default Data Loader

7. **Connect nodes in order:**  
   Gmail Trigger → Get Mail Data → Postgres PGVector Store3  
   Enhanced Default Data Loader ← Recursive Character Text Splitter2 ← Embeddings OpenAI3 → Postgres PGVector Store3

8. **Create a Chat Trigger node ("When chat message received")**  
   - Configure webhook to receive chat messages

9. **Create an OpenAI Embeddings node ("Embeddings OpenAI") for chat queries**  
   - Use OpenAI API credentials  
   - Input: Incoming chat message text

10. **Create a PostgreSQL PGVector Store node ("Postgres PGVector Store")**  
    - Mode: `retrieve-as-tool`  
    - Table name: `emails_vector_history`  
    - Tool name: `emails_vector_search`  
    - Tool description: Include instructions to always specify time frames in queries  
    - TopK: 5 for top results  
    - Input: Embeddings from chat embeddings node

11. **Create a Langchain Agent node ("New RAG Agent")**  
    - System message:  
      ```
      You are a helpful assistant that will get data from RAG and send a good response to user.
      Today date is this if user ask for dated or latest data: {{ $now }}
      ```  
    - Connect inputs from chat trigger, Postgres PGVector Store (vector search), and OpenAI Chat Model

12. **Create an OpenAI Chat Model node ("OpenAI Chat Model")**  
    - Use OpenAI API credentials  
    - Input: Language model input from the New RAG Agent node

13. **Connect nodes in order:**  
    When chat message received → New RAG Agent → OpenAI Chat Model  
    Embeddings OpenAI (chat) → Postgres PGVector Store → New RAG Agent  
    OpenAI Chat Model → New RAG Agent (output)

14. **Add Sticky Note nodes** for documentation and workflow clarity as per the original setup.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                   | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses PGVector extension in PostgreSQL for efficient vector similarity search. Make sure PGVector is installed and configured in your PostgreSQL instance.                      | PGVector official: https://github.com/pgvector/pgvector                                          |
| The OpenAI API credentials must have access to both embeddings and chat completion models. Ensure API keys have sufficient quota and rate limits.                                             | OpenAI API documentation: https://platform.openai.com/docs/api-reference                        |
| The system message in the RAG agent dynamically inserts the current date ({{ $now }}) to improve time-aware answers.                                                                           | Dynamic variables in n8n expressions: https://docs.n8n.io/nodes/expressions/                      |
| For security, the Gmail OAuth credentials must be set with read access and token refresh enabled to avoid interruptions.                                                                        | Gmail API OAuth2 setup: https://developers.google.com/gmail/api/auth/about-auth                 |
| The workflow is designed to be extensible: additional vector databases or AI models may be integrated by replacing or augmenting the vector store and LLM nodes respectively.                  | n8n Langchain nodes documentation: https://docs.n8n.io/integrations/builtin/ai/langchain/       |
| RAG workflows can be sensitive to data volume and latency; monitor API usage and database performance to maintain responsiveness.                                                              | n8n workflow performance tuning: https://docs.n8n.io/integrations/performance/                   |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.