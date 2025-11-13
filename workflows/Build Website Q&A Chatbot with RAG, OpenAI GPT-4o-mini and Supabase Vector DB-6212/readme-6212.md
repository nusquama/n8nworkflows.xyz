Build Website Q&A Chatbot with RAG, OpenAI GPT-4o-mini and Supabase Vector DB

https://n8nworkflows.xyz/workflows/build-website-q-a-chatbot-with-rag--openai-gpt-4o-mini-and-supabase-vector-db-6212


# Build Website Q&A Chatbot with RAG, OpenAI GPT-4o-mini and Supabase Vector DB

### 1. Workflow Overview

This workflow builds a Retrieval-Augmented Generation (RAG) chatbot for a website using Supabase as a vector database, Cohere for embeddings, and OpenAI GPT-4o-mini as the language model. The chatbot enables users to submit questions, which are answered based strictly on content extracted and embedded from a target website. The workflow is structured around four main logical blocks:

- **1.1 Website Data Extraction:** Takes a user-provided website URL, scrapes its HTML content, extracts relevant text and metadata, and converts it into a structured JSON file for further processing.

- **1.2 Embedding Generation and Storage:** Processes the extracted website content by splitting it into manageable text chunks, generates vector embeddings for these chunks using Cohere embeddings, and inserts them into the Supabase vector store.

- **1.3 User Query Reception and Context Memory:** Listens for incoming chat messages (user questions), manages chat memory using Postgres to maintain conversational context, and prepares the question for vector-based retrieval.

- **1.4 Vector-Based Answer Retrieval and Response Generation:** Retrieves the most relevant document chunks from Supabase based on the user question embeddings, and uses an AI agent combining Cohere embeddings and OpenAI GPT-4o-mini to generate answers strictly grounded in the retrieved documents.

---

### 2. Block-by-Block Analysis

#### 2.1 Website Data Extraction

- **Overview:**  
  This block accepts a website URL from the user, performs HTTP scraping of the site, extracts key content and metadata using CSS selectors, and converts the extracted data into JSON format for downstream processing.

- **Nodes Involved:**  
  - Enter Website Url (Form Trigger)  
  - Website Data Scrapping (HTTP Request)  
  - HTML Extract (HTML Extract)  
  - Convert to File (Convert To File)  
  - Sticky Note (Website Data Extraction)

- **Node Details:**

  - **Enter Website Url**  
    - *Type:* Form Trigger  
    - *Role:* Captures user input of the website URL via a web form.  
    - *Configuration:* Single field "Website Url" with placeholder; triggers workflow on submission.  
    - *Connections:* Output → Website Data Scrapping  
    - *Edge Cases:* Invalid or unreachable URLs; ensure input validation or error handling on HTTP request node.  

  - **Website Data Scrapping**  
    - *Type:* HTTP Request  
    - *Role:* Fetches raw HTML content of the provided website URL.  
    - *Configuration:* URL dynamically set from form input; timeout 30s; max 5 redirects; 3 retry attempts on failure.  
    - *Connections:* Output → HTML Extract  
    - *Edge Cases:* Timeouts, HTTP errors, redirects loops, unavailable websites.  

  - **HTML Extract**  
    - *Type:* HTML Extract  
    - *Role:* Extracts specific elements from the HTML: titles (title, h1), content paragraphs and main article sections, meta description, and all links (href attributes).  
    - *Configuration:* CSS selectors target standard content tags.  
    - *Connections:* Output → Convert to File  
    - *Edge Cases:* Missing selectors, empty content, malformed HTML.  

  - **Convert to File**  
    - *Type:* Convert To File  
    - *Role:* Converts extracted HTML content into JSON file format for embedding.  
    - *Configuration:* Converts data to JSON file type.  
    - *Connections:* Output → Supabase Vector Store (for insertion)  
    - *Edge Cases:* Conversion failures if data is malformed.  

  - **Sticky Note**  
    - *Content:* "## Website Data Extraction" – clarifies this block's purpose visually.

---

#### 2.2 Embedding Generation and Storage

- **Overview:**  
  This block takes the JSON content file, splits the text into overlapping chunks to optimize embedding quality and context, generates vector embeddings using Cohere, then inserts these embeddings into the Supabase vector store in batches.

- **Nodes Involved:**  
  - Default Data Loader (Document Loader)  
  - Recursive Character Text Splitter (Text Splitter)  
  - Embeddings Cohere (Embeddings)  
  - Supabase Vector Store (Vector Store Insert)  
  - Sticky Note1 ("Generating Embeddings from Website Content")

- **Node Details:**

  - **Default Data Loader**  
    - *Type:* LangChain Document Default Data Loader  
    - *Role:* Loads the JSON file data for processing as documents.  
    - *Configuration:* Uses JSON loader with metadata injection (website_url from form input).  
    - *Connections:* Output → Supabase Vector Store (document input)  
    - *Edge Cases:* File reading errors, invalid JSON structure.  

  - **Recursive Character Text Splitter**  
    - *Type:* LangChain Text Splitter  
    - *Role:* Splits long text documents into chunks of 3000 characters with 500 characters overlap for embedding.  
    - *Configuration:* Default options, chunkSize=3000, chunkOverlap=500.  
    - *Connections:* Output → Default Data Loader  
    - *Edge Cases:* Very short documents, improper chunking.  

  - **Embeddings Cohere**  
    - *Type:* LangChain Embeddings (Cohere)  
    - *Role:* Generates vector embeddings from text chunks.  
    - *Configuration:* Uses Cohere API credentials.  
    - *Connections:* ai_embedding → Supabase Vector Store  
    - *Edge Cases:* API rate limits, invalid API keys, network errors.  

  - **Supabase Vector Store (Insert Mode)**  
    - *Type:* LangChain Vector Store (Supabase)  
    - *Role:* Inserts embeddings into the "documents" table in Supabase vector database.  
    - *Configuration:* Insert mode, batch size 100, credentials for Supabase API.  
    - *Connections:* Receives document, embedding, and text splitter inputs; no outputs (terminal insert).  
    - *Edge Cases:* Database connection failures, insertion errors, API limits.  

  - **Sticky Note1**  
    - *Content:* "## Generating Embeddings from Website Content"

---

#### 2.3 User Query Reception and Context Memory

- **Overview:**  
  Listens for incoming chat messages from users, manages chat conversational memory stored in Postgres to maintain context with a limited window, and prepares user queries for vector retrieval.

- **Nodes Involved:**  
  - When chat message received (Chat Trigger)  
  - Chat Memory (Postgres Chat Memory)  
  - Sticky Note2 ("User-Initiated Question")

- **Node Details:**

  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Webhook-based listener for incoming chat messages (user questions).  
    - *Configuration:* Public webhook mode; no authentication; triggers on message receipt.  
    - *Connections:* Output → Question & Answer Retrieve  
    - *Edge Cases:* Excessive message frequency, malformed inputs.  

  - **Chat Memory**  
    - *Type:* LangChain Memory (Postgres Chat Memory)  
    - *Role:* Stores and retrieves last 3 interactions to preserve conversational context.  
    - *Configuration:* Table "chat_memory" in Postgres; context window length = 3 messages.  
    - *Connections:* Memory interface with Question & Answer Retrieve node.  
    - *Edge Cases:* Database connectivity issues, memory overflow, data corruption.  

  - **Sticky Note2**  
    - *Content:* "## User-Initiated Question"

---

#### 2.4 Vector-Based Answer Retrieval and Response Generation

- **Overview:**  
  This block implements the RAG logic: it retrieves most relevant document chunks from Supabase based on the user's query embedding, then synthesizes an AI-generated answer strictly based on those documents using OpenAI GPT-4o-mini and Cohere embeddings.

- **Nodes Involved:**  
  - Question & Answer Retrieve (LangChain Agent)  
  - Data From Supabase Vector Store (Vector Store Retrieval)  
  - Embeddings With Cohere (Embeddings)  
  - OpenAI Chat Modell (Language Model)  
  - Sticky Note3 ("Vector-Based Answer Retrieval")

- **Node Details:**

  - **Question & Answer Retrieve**  
    - *Type:* LangChain Agent  
    - *Role:* Core AI agent that controls the question answering process.  
    - *Configuration:*  
      - System message instructs to respond strictly using Supabase vector store content.  
      - Steps: analyze question, retrieve relevant docs from Supabase, synthesize answer only from docs, fallback if no matches.  
      - Tools used: Supabase vector store and Cohere embeddings.  
    - *Connections:*  
      - Receives input from chat trigger.  
      - Uses Chat Memory, OpenAI Chat Model, Data From Supabase Vector Store, and Embeddings With Cohere as subcomponents.  
    - *Edge Cases:* No relevant documents found, inconsistent retrieval, OpenAI API failures.  

  - **Data From Supabase Vector Store**  
    - *Type:* LangChain Vector Store (Supabase)  
    - *Role:* Retrieves top matching document vectors from Supabase for a given query embedding.  
    - *Configuration:* Mode "retrieve-as-tool", no topK limit set, uses "documents" table, excludes document metadata from output, credentials provided.  
    - *Connections:* Output feeds into Question & Answer Retrieve agent as a tool.  
    - *Edge Cases:* Retrieval misses relevant docs, DB connectivity issues.  

  - **Embeddings With Cohere**  
    - *Type:* LangChain Embeddings (Cohere)  
    - *Role:* Generates embeddings for the user query to perform vector similarity search.  
    - *Configuration:* Uses Cohere API credentials.  
    - *Connections:* Feeds embeddings to Data From Supabase Vector Store.  
    - *Edge Cases:* API or network failures.  

  - **OpenAI Chat Modell**  
    - *Type:* LangChain Language Model (OpenAI GPT-4o-mini)  
    - *Role:* Generates final textual response based on retrieved documents and chat memory.  
    - *Configuration:* Model set to "gpt-4o-mini", uses OpenAI credentials.  
    - *Connections:* Output feeds into Question & Answer Retrieve node.  
    - *Edge Cases:* API rate limits, generation errors, hallucination risk if instructions are not strictly followed.  

  - **Sticky Note3**  
    - *Content:* "## Vector-Based Answer Retrieval"

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                       | Input Node(s)               | Output Node(s)                | Sticky Note                              |
|-----------------------------|--------------------------------------------|------------------------------------|-----------------------------|------------------------------|-----------------------------------------|
| Enter Website Url            | Form Trigger                               | User input of website URL           |                             | Website Data Scrapping        | ## Website Data Extraction              |
| Website Data Scrapping       | HTTP Request                              | Scrape website HTML content         | Enter Website Url            | HTML Extract                 | ## Website Data Extraction              |
| HTML Extract                | HTML Extract                              | Extract content & metadata          | Website Data Scrapping       | Convert to File              | ## Website Data Extraction              |
| Convert to File             | Convert To File                           | Convert extracted data to JSON file | HTML Extract                | Supabase Vector Store        | ## Generating Embeddings from Website Content |
| Default Data Loader         | Document Loader (LangChain)                | Load JSON data for embedding        | Recursive Character Text Splitter | Supabase Vector Store     | ## Generating Embeddings from Website Content |
| Recursive Character Text Splitter | Text Splitter (LangChain)                 | Split text into chunks for embedding |                             | Default Data Loader          | ## Generating Embeddings from Website Content |
| Embeddings Cohere           | Embeddings (LangChain - Cohere)            | Generate vector embeddings          |                             | Supabase Vector Store        | ## Generating Embeddings from Website Content |
| Supabase Vector Store       | Vector Store Insert (LangChain Supabase)   | Insert embeddings into DB           | Convert to File, Default Data Loader, Embeddings Cohere |                              | ## Generating Embeddings from Website Content |
| When chat message received  | Chat Trigger (LangChain)                    | Receive user chat message           |                             | Question & Answer Retrieve   | ## User-Initiated Question              |
| Chat Memory                | Memory (LangChain Postgres)                 | Store conversational context         |                             | Question & Answer Retrieve   | ## User-Initiated Question              |
| Question & Answer Retrieve  | Agent (LangChain)                           | Main RAG agent for answering        | When chat message received, Chat Memory, Data From Supabase Vector Store, Embeddings With Cohere, OpenAI Chat Modell |                              | ## Vector-Based Answer Retrieval        |
| Data From Supabase Vector Store | Vector Store Retrieval (LangChain Supabase) | Retrieve relevant docs from vector DB | Embeddings With Cohere       | Question & Answer Retrieve   | ## Vector-Based Answer Retrieval        |
| Embeddings With Cohere      | Embeddings (LangChain - Cohere)            | Embed user query for retrieval       |                             | Data From Supabase Vector Store | ## Vector-Based Answer Retrieval        |
| OpenAI Chat Modell          | Language Model (LangChain OpenAI)           | Generate final answer text           |                             | Question & Answer Retrieve   | ## Vector-Based Answer Retrieval        |
| Sticky Note                 | Sticky Note                               | Visual annotation                   |                             |                              | ## Website Data Extraction              |
| Sticky Note1                | Sticky Note                               | Visual annotation                   |                             |                              | ## Generating Embeddings from Website Content |
| Sticky Note2                | Sticky Note                               | Visual annotation                   |                             |                              | ## User-Initiated Question              |
| Sticky Note3                | Sticky Note                               | Visual annotation                   |                             |                              | ## Vector-Based Answer Retrieval        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Enter Website Url" node**  
   - Type: Form Trigger  
   - Configuration: Add single form field labeled "Website Url" with placeholder "Enter Website Url"  
   - Set webhook ID auto-generated  
   - No credentials needed  
   - Position appropriately (e.g., top-left)  

2. **Create "Website Data Scrapping" node**  
   - Type: HTTP Request  
   - Set URL to expression: `={{ $json["Website Url"] }}` to dynamically use form input  
   - Configure timeout: 30000 ms  
   - Enable max 5 redirects  
   - Set retry on fail with max 3 attempts  
   - Connect output of "Enter Website Url" to this node  

3. **Create "HTML Extract" node**  
   - Type: HTML Extract  
   - Add extraction rules:  
     - title: CSS selector `title, h1` (get text)  
     - content: CSS selector `p, article, .content, .post-content, main`  
     - meta_description: CSS selector `meta[name='description']` attribute `content`  
     - links: CSS selector `a[href]` attribute `href`  
   - Connect output of "Website Data Scrapping" to this node  

4. **Create "Convert to File" node**  
   - Type: Convert To File  
   - Operation: toJson  
   - Connect output of "HTML Extract" to this node  

5. **Create "Recursive Character Text Splitter" node**  
   - Type: LangChain Text Splitter (Recursive Character)  
   - Set chunkSize to 3000, chunkOverlap to 500  
   - Position downstream of "Convert to File" or as per your logical grouping  

6. **Create "Default Data Loader" node**  
   - Type: LangChain Document Default Data Loader  
   - Loader: JSON Loader  
   - Include metadata: add key `website_url` with value from expression: `={{ $('Enter Website Url').item.json['Website Url'] }}`  
   - Connect output of "Recursive Character Text Splitter" to this loader  

7. **Create "Embeddings Cohere" node**  
   - Type: LangChain Embeddings (Cohere)  
   - Set credentials to your Cohere API account  
   - Connect output of "Default Data Loader" to this node's ai_embedding input  

8. **Create "Supabase Vector Store" node (Insert mode)**  
   - Type: LangChain Vector Store Supabase  
   - Set mode to "insert"  
   - Table name: "documents"  
   - Set embedding batch size to 100  
   - Provide Supabase credentials  
   - Connect outputs from "Convert to File" (main), "Default Data Loader" (document), and "Embeddings Cohere" (embedding) to respective inputs  

9. **Create "When chat message received" node**  
   - Type: LangChain Chat Trigger  
   - Mode: webhook, public  
   - No authentication required  
   - Position at top-right or appropriate location  

10. **Create "Chat Memory" node**  
    - Type: LangChain Memory (Postgres Chat Memory)  
    - Table name: "chat_memory"  
    - Context window length: 3  
    - Provide Postgres credentials  

11. **Create "Embeddings With Cohere" node**  
    - Type: LangChain Embeddings (Cohere)  
    - Use the same Cohere credentials as before  

12. **Create "Data From Supabase Vector Store" node (Retrieve mode)**  
    - Type: LangChain Vector Store Supabase  
    - Mode: "retrieve-as-tool"  
    - Table name: "documents"  
    - Tool name: "documents_knowledge_base"  
    - Tool description: "work with documents data in Supabase vector store"  
    - Exclude document metadata  
    - Provide Supabase credentials  

13. **Create "OpenAI Chat Modell" node**  
    - Type: LangChain Language Model (OpenAI)  
    - Model: "gpt-4o-mini"  
    - Provide OpenAI API credentials  

14. **Create "Question & Answer Retrieve" node**  
    - Type: LangChain Agent  
    - Configure system message with instructions to:  
      - Respond strictly using Supabase vector store content  
      - Analyze user question, retrieve relevant docs, answer only from those docs  
      - Use Cohere embeddings and Supabase vector store as tools  
      - Fallback message if no matches  
    - Connect inputs:  
      - Main from "When chat message received"  
      - ai_memory from "Chat Memory"  
      - ai_embedding from "Embeddings With Cohere"  
      - ai_tool from "Data From Supabase Vector Store"  
      - ai_languageModel from "OpenAI Chat Modell"  

15. **Connect all nodes as per the logic described:**  
    - "Enter Website Url" → "Website Data Scrapping" → "HTML Extract" → "Convert to File" → "Supabase Vector Store" (insert)  
    - "Recursive Character Text Splitter" → "Default Data Loader" → "Embeddings Cohere" → "Supabase Vector Store" (insert)  
    - "When chat message received" → "Question & Answer Retrieve"  
    - "Chat Memory" → "Question & Answer Retrieve" (ai_memory)  
    - "Embeddings With Cohere" → "Data From Supabase Vector Store" → "Question & Answer Retrieve" (ai_tool)  
    - "OpenAI Chat Modell" → "Question & Answer Retrieve" (ai_languageModel)  

16. **Add Sticky Notes for clarity:**  
    - Website Data Extraction near first block  
    - Generating Embeddings from Website Content near embedding block  
    - User-Initiated Question near chat trigger and chat memory  
    - Vector-Based Answer Retrieval near question answering nodes  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                       |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| The system message in the "Question & Answer Retrieve" node is critical to avoid hallucinations by the AI.    | See system message content in node details section.  |
| Supabase vector store requires proper indexing and embedding dimension setup in your database.                | Refer to Supabase Vector DB documentation.            |
| Cohere API is used for embedding generation; ensure your API key has sufficient quota for batch embedding.   | https://cohere.ai/docs                                |
| OpenAI GPT-4o-mini model provides cost-effective chat generation for RAG workflows.                          | https://platform.openai.com/docs/models/gpt-4o-mini  |
| Postgres chat memory table must be created beforehand with appropriate schema for storing conversation data. | Refer to LangChain n8n memoryPostgresChat docs.      |
| The "Enter Website Url" form trigger exposes a public webhook; secure or restrict usage as needed.            | Implement external authentication if required.       |
| Retry and timeout settings on HTTP Request node help handle transient website scraping failures.              | Best practice for web scraping stability.            |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.