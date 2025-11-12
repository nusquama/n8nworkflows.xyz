Create a Knowledge-Powered Chatbot with Claude, Supabase Vector DB & Postgres Memory

https://n8nworkflows.xyz/workflows/create-a-knowledge-powered-chatbot-with-claude--supabase-vector-db---postgres-memory-7381


# Create a Knowledge-Powered Chatbot with Claude, Supabase Vector DB & Postgres Memory

### 1. Workflow Overview

This workflow constructs an intelligent, knowledge-powered chatbot leveraging Claude (Anthropic) as the language model, Supabase as a vector database for knowledge retrieval, and PostgreSQL for conversation memory. It is designed for business, developer, and organizational use cases requiring customizable AI chatbots that access domain-specific knowledge bases and maintain conversational context.

The workflow is logically divided into these core blocks:

- **1.1 Input Reception:** Listens for incoming chat messages via an n8n Langchain Chat Trigger node.

- **1.2 AI Processing:** Uses an AI Agent node that integrates multiple components—language model, memory, and tools—to generate responses.

- **1.3 Language Model:** Employs Anthropic’s Claude model for natural language understanding and response generation.

- **1.4 Memory Management:** Utilizes PostgreSQL-based chat memory to store and recall conversation history, enabling context-aware interactions.

- **1.5 Knowledge Retrieval:** Connects to a Supabase vector store containing embedded documents for semantic search and relevant knowledge extraction.

- **1.6 Embeddings Generation:** Generates text embeddings using OpenAI embeddings to support similarity search in the Supabase vector database.

- **1.7 Documentation & Guidance:** Contains a large sticky note node providing detailed workflow description, setup instructions, customization options, use cases, and limitations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures incoming chat messages from any platform supporting webhook triggers and initiates the AI processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **Type and Role:** Langchain Chat Trigger node; listens for chat messages to start the workflow.  
  - **Configuration:** Default options; webhook ID assigned for external integration.  
  - **Expressions/Variables:** None explicitly configured; standard input reception.  
  - **Connections:** Output connects to the AI Agent node.  
  - **Version:** 1.3  
  - **Edge Cases:** Missing or malformed webhook calls; network timeouts; platform-specific webhook authentication or formatting issues.  
  - **Sub-workflows:** None.

#### 2.2 AI Processing

- **Overview:**  
  Central orchestrator that integrates the language model, memory, and external tools to process the user's query and produce a response.

- **Nodes Involved:**  
  - AI Agent

- **Node Details:**  
  - **Type and Role:** Langchain Agent node; manages AI interactions with modular components.  
  - **Configuration:** System message set to “You are a helpful assistant” defining the AI’s persona.  
  - **Expressions/Variables:** System message hardcoded; components connected via credentials and parameter links.  
  - **Connections:**  
    - Input from “When chat message received” node.  
    - Connected to “Anthropic Chat Model” as language model.  
    - Connected to “Postgres Chat Memory” as memory source.  
    - Connected to “Supabase Vector Store” as tool for knowledge retrieval.  
  - **Version:** 2.2  
  - **Edge Cases:**  
    - Improper credential configuration leading to auth failures.  
    - Component downtime or API rate limits.  
    - Unexpected or malformed input messages causing processing errors.  
  - **Sub-workflows:** None.

#### 2.3 Language Model

- **Overview:**  
  Executes the primary natural language processing and response generation using Anthropic's Claude model.

- **Nodes Involved:**  
  - Anthropic Chat Model

- **Node Details:**  
  - **Type and Role:** Langchain Anthropic Chat Model node; handles LLM operations.  
  - **Configuration:**  
    - Model selected: “claude-sonnet-4-20250514” (Claude 4 Sonnet) with caching enabled for efficiency.  
    - No additional options set.  
    - Credential: Anthropic API key configured.  
  - **Expressions/Variables:** Model selected via list mode expression to enable caching.  
  - **Connections:** Output connects to AI Agent node’s language model input.  
  - **Version:** 1.3  
  - **Edge Cases:**  
    - API authentication errors.  
    - API rate limits.  
    - Model unavailability or changes in API specification.  
  - **Sub-workflows:** None.

#### 2.4 Memory Management

- **Overview:**  
  Maintains chat history in PostgreSQL to provide context and traceability across conversations.

- **Nodes Involved:**  
  - Postgres Chat Memory

- **Node Details:**  
  - **Type and Role:** Langchain Postgres Chat Memory node; stores and retrieves conversation context.  
  - **Configuration:**  
    - Context window length set to 20 messages, balancing memory size and relevance.  
    - Credential: PostgreSQL database (n8n’s built-in or configured).  
  - **Expressions/Variables:** None special; static configuration.  
  - **Connections:** Output connected to AI Agent memory input.  
  - **Version:** 1.3  
  - **Edge Cases:**  
    - Database connection failure or misconfiguration.  
    - Data corruption or schema mismatch.  
    - Performance degradation with large context windows or high concurrency.  
  - **Sub-workflows:** None.

#### 2.5 Knowledge Retrieval

- **Overview:**  
  Provides semantic search functionality by querying a Supabase vector database with OpenAI-generated embeddings.

- **Nodes Involved:**  
  - Supabase Vector Store

- **Node Details:**  
  - **Type and Role:** Langchain Supabase Vector Store node; performs embedding-based document retrieval.  
  - **Configuration:**  
    - Mode set to “retrieve-as-tool” enabling automatic integration as a tool in the AI agent.  
    - Table name specified as “growth_ai_documents” where knowledge base documents are stored.  
    - Tool description set to “Database”.  
    - Credential: Supabase API credentials configured.  
  - **Expressions/Variables:** Table name selected via list expression for caching.  
  - **Connections:** Output connected as a tool input to AI Agent. Input depends on Embeddings OpenAI node.  
  - **Version:** 1.3  
  - **Edge Cases:**  
    - API key expiration or permission issues.  
    - Table or data schema changes causing retrieval errors.  
    - Network timeouts or slow queries affecting response time.  
  - **Sub-workflows:** None.

#### 2.6 Embeddings Generation

- **Overview:**  
  Converts text data into vector embeddings using OpenAI’s embeddings model for semantic similarity search.

- **Nodes Involved:**  
  - Embeddings OpenAI

- **Node Details:**  
  - **Type and Role:** Langchain OpenAI Embeddings node; generates vector representations of text.  
  - **Configuration:** Default options used; credentials for OpenAI API provided.  
  - **Expressions/Variables:** None special; static configuration.  
  - **Connections:** Output feeds into Supabase Vector Store node’s embedding input.  
  - **Version:** 1.2  
  - **Edge Cases:**  
    - API key invalid or rate limits exceeded.  
    - Network or API outages.  
    - Embedding model updates or deprecation.  
  - **Sub-workflows:** None.

#### 2.7 Documentation & Guidance

- **Overview:**  
  Provides comprehensive documentation and setup instructions via a sticky note node embedded in the workflow canvas.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Type and Role:** n8n Sticky Note node; purely informational, no data flow impact.  
  - **Configuration:** Large note describing use cases, technical architecture, setup steps, customization, and known limitations.  
  - **Expressions/Variables:** Static content.  
  - **Connections:** None.  
  - **Version:** 1  
  - **Edge Cases:** None (informational).  
  - **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                   | Input Node(s)             | Output Node(s)        | Sticky Note                                                                                       |
|------------------------|----------------------------------|---------------------------------|---------------------------|-----------------------|------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger            | Input reception                  |                           | AI Agent              |                                                                                                |
| AI Agent               | Langchain Agent                   | Central AI processing            | When chat message received |                         |                                                                                                |
| Anthropic Chat Model   | Langchain Anthropic Chat Model   | Language model                   |                           | AI Agent              |                                                                                                |
| Postgres Chat Memory   | Langchain Postgres Chat Memory   | Conversation memory              |                           | AI Agent              |                                                                                                |
| Supabase Vector Store  | Langchain Supabase Vector Store  | Knowledge retrieval tool         | Embeddings OpenAI          | AI Agent              |                                                                                                |
| Embeddings OpenAI      | Langchain OpenAI Embeddings      | Embeddings generation            |                           | Supabase Vector Store |                                                                                                |
| Sticky Note            | n8n Sticky Note                  | Documentation and workflow notes |                           |                       | # Intelligent chatbot with custom knowledge base<br>See setup instructions, use cases, and limitations in the note. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the workflow in n8n:

1. **Create the Trigger Node:**  
   - Add a **Langchain Chat Trigger** node named “When chat message received”.  
   - Use default settings (version 1.3).  
   - This node will listen for incoming chat messages via webhook.

2. **Add the AI Agent Node:**  
   - Add a **Langchain Agent** node named “AI Agent” (version 2.2).  
   - Set system message to: “You are a helpful assistant”.  
   - Connect the output of “When chat message received” node to the input of this node.

3. **Configure the Language Model:**  
   - Add a **Langchain Anthropic Chat Model** node named “Anthropic Chat Model” (version 1.3).  
   - Select the model “claude-sonnet-4-20250514” (Claude 4 Sonnet).  
   - Enable caching by selecting the model via list mode with cached result name.  
   - Configure Anthropic API credentials (OAuth or API key).  
   - Connect this node’s output to the AI Agent node’s language model input.

4. **Set Up Conversation Memory:**  
   - Add a **Langchain Postgres Chat Memory** node named “Postgres Chat Memory” (version 1.3).  
   - Set Context Window Length to 20 messages.  
   - Configure PostgreSQL credentials (can use n8n built-in PostgreSQL or external).  
   - Connect output to AI Agent node’s memory input.

5. **Configure Knowledge Retrieval:**  
   - Add a **Langchain Supabase Vector Store** node named “Supabase Vector Store” (version 1.3).  
   - Set mode to “retrieve-as-tool”.  
   - Enter your Supabase API credentials.  
   - Set the table name to your knowledge base table, e.g., “growth_ai_documents”.  
   - Set tool description to “Database” or customize as needed.  
   - Connect output as a tool input to AI Agent node.

6. **Add Embeddings Generation:**  
   - Add a **Langchain OpenAI Embeddings** node named “Embeddings OpenAI” (version 1.2).  
   - Configure OpenAI credentials.  
   - Connect this node’s output to the “Supabase Vector Store” node’s embedding input.

7. **Connect Nodes Appropriately:**  
   - Connect “When chat message received” → “AI Agent”  
   - Connect “Anthropic Chat Model” → “AI Agent” (language model input)  
   - Connect “Postgres Chat Memory” → “AI Agent” (memory input)  
   - Connect “Supabase Vector Store” → “AI Agent” (tool input)  
   - Connect “Embeddings OpenAI” → “Supabase Vector Store” (embedding input)

8. **Add Informational Sticky Note (Optional):**  
   - Add a **Sticky Note** node for documentation directly on the workflow canvas.  
   - Paste detailed instructions, use cases, and setup notes as needed.

9. **Credential Setup:**  
   - Ensure you have valid API keys and credentials added in n8n for:  
     - Anthropic API (Claude)  
     - OpenAI API (for embeddings)  
     - Supabase API (project and database access)  
     - PostgreSQL (conversation memory)  

10. **Test the Workflow:**  
    - Trigger the workflow by sending a chat message through the configured webhook.  
    - Verify AI Agent processes the message, accesses memory, retrieves knowledge from Supabase, and returns an answer.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow designed for businesses and developers needing customizable AI chatbots integrating personalized knowledge bases and memory.                                                                                                                                                                                                                                                       | General workflow purpose                                                                                 |
| Requires a Supabase project with vector database capable of storing embedded documents for semantic search.                                                                                                                                                                                                                                                                                     | Supabase vector DB setup                                                                                 |
| OpenAI embeddings configuration is necessary for vector similarity search functionality.                                                                                                                                                                                                                                                                                                        | Refer to Cole Medin’s tutorial on OpenAI embeddings and vectorization                                  |
| PostgreSQL is used for conversation memory to maintain context and history across sessions.                                                                                                                                                                                                                                                                                                    | n8n built-in PostgreSQL or external PostgreSQL instance                                                |
| Anthropic Claude model selected for advanced natural language generation; requires valid API key.                                                                                                                                                                                                                                                                                              | Anthropic API documentation                                                                             |
| Workflow can be customized for different platforms by replacing the chat trigger node with platform-specific webhook triggers (Slack, Teams, Discord, websites).                                                                                                                                                                                                                                | Platform-specific webhook configuration                                                                |
| The chatbot can be extended with tools for email, calendar, databases, and API integrations to perform actions beyond conversation.                                                                                                                                                                                                                                                            | Advanced action capabilities                                                                             |
| Watch Cole Medin’s tutorial video for comprehensive guidance on document vectorization and Supabase knowledge base construction: https://www.youtube.com/watch?v=XXXXXXX (replace XXXXXXX with actual video ID).                                                                                                                                                                                   | External tutorial link                                                                                   |
| Memory context window length can be adjusted to balance response relevance and resource usage.                                                                                                                                                                                                                                                                                                  | Configuration tip                                                                                       |
| Semantic search enables retrieval based on meaning rather than keywords, improving answer relevance.                                                                                                                                                                                                                                                                                            | Knowledge retrieval best practice                                                                       |
| Be aware of API rate limits and token usage costs for OpenAI and Anthropic services.                                                                                                                                                                                                                                                                                                            | Cost and rate limit considerations                                                                      |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, adhering strictly to current content policies with no illegal, offensive, or protected content. All data handled is legal and public.