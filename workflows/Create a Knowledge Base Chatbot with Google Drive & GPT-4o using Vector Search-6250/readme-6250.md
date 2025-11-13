Create a Knowledge Base Chatbot with Google Drive & GPT-4o using Vector Search

https://n8nworkflows.xyz/workflows/create-a-knowledge-base-chatbot-with-google-drive---gpt-4o-using-vector-search-6250


# Create a Knowledge Base Chatbot with Google Drive & GPT-4o using Vector Search

### 1. Workflow Overview

This workflow implements a **Knowledge Base Chatbot** leveraging documents stored in **Google Drive** and OpenAI’s **GPT-4o-mini** model with vector search capabilities. It enables users to query a chatbot whose knowledge is dynamically built from files (PDFs, text, etc.) uploaded or updated in a specified Google Drive folder. The workflow consists of two main logical blocks:

- **1.1 File Upload and Knowledge Preparation:**  
  Watches a specific Google Drive folder for new or updated files, downloads these files, processes their content by splitting into manageable chunks, generates vector embeddings using OpenAI, and stores these embeddings in an in-memory vector store for fast retrieval.

- **1.2 AI Chatbot for Question Answering:**  
  Exposes a webhook for incoming user queries, authenticates requests, retrieves contextually relevant information from the vector store based on the user’s question, uses OpenAI GPT-4o-mini to generate answers grounded strictly in the document content, and sends the responses back to the user’s chat platform.

Additional supporting nodes include detailed configuration and usage sticky notes, token authentication for security, and conversation memory for maintaining context.

---

### 2. Block-by-Block Analysis

#### 2.1 File Upload and Knowledge Preparation

**Overview:**  
This block monitors a specific Google Drive folder for new or updated files, downloads their content, processes and splits text into chunks, generates embeddings, and stores them in a vector store to create a searchable knowledge base.

**Nodes Involved:**  
- File created in the Folder (Google Drive Trigger)  
- File updated in the Folder (Google Drive Trigger)  
- Search Files in your Google Drive Folder (Google Drive node)  
- Download Files (Google Drive node)  
- Embeddings OpenAI (OpenAI Embeddings node)  
- Default Data Loader (Langchain Document Loader)  
- Recursive Character Text Splitter (Langchain Text Splitter)  
- Simple Vector Store (Langchain Vector Store In-Memory)  

**Node Details:**

- **File created in the Folder**  
  - Type: Google Drive Trigger  
  - Role: Triggers workflow when a new file is created in the specified Google Drive folder (`folderToWatch` set to folder ID `1Ve5ZLtQwpK5Hq4YxZTNRduRgPlSMZ1hJ`).  
  - Configuration: Polls every minute to detect new files.  
  - Input: None (trigger node)  
  - Output: File metadata of the newly created file.  
  - Edge cases: Delay in Google Drive propagating file creation events; possible missing events if polling interval is too large.

- **File updated in the Folder**  
  - Type: Google Drive Trigger  
  - Role: Triggers workflow when an existing file is updated in the same Google Drive folder.  
  - Configuration: Polls every minute.  
  - Input: None (trigger node)  
  - Output: File metadata of the updated file.  
  - Edge cases: Same as created trigger; potential double processing if file is frequently updated.

- **Search Files in your Google Drive Folder**  
  - Type: Google Drive  
  - Role: Searches for files within the specified Drive folder to retrieve metadata necessary for downloading.  
  - Configuration: Filters by `driveId` ("My Drive") and `folderId` (same folder ID as triggers).  
  - Input: Trigger data (file metadata from the triggers)  
  - Output: List of files matching folder filter.  
  - Edge cases: Permission issues, empty folder, multiple files with similar names.

- **Download Files**  
  - Type: Google Drive  
  - Role: Downloads the binary content of the file identified by `fileId` from the search node.  
  - Configuration: Uses file ID passed dynamically from previous node (`{{$json["id"]}}`).  
  - Input: File metadata (including file ID)  
  - Output: Binary content of the file.  
  - Edge cases: Download failure due to permissions, large file size timeouts, unsupported file formats.

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings (Langchain)  
  - Role: Converts document text chunks into vector embeddings using OpenAI API.  
  - Configuration: Uses OpenAI API credentials with default options; no special parameters changed.  
  - Input: Document chunks from data loader.  
  - Output: Vector embeddings for each chunk.  
  - Edge cases: Rate limits, invalid API key, unsupported languages or characters.

- **Default Data Loader**  
  - Type: Langchain Document Loader  
  - Role: Loads the downloaded document in binary mode, preparing it for text splitting.  
  - Configuration: `dataType` set to binary with `binaryMode` targeting a specific field (contains file content).  
  - Input: Binary file content from Download Files node.  
  - Output: Document content ready for splitting.  
  - Edge cases: Unsupported file types, corrupt files.

- **Recursive Character Text Splitter**  
  - Type: Langchain Text Splitter  
  - Role: Splits large document text recursively into overlapping chunks to improve embedding quality.  
  - Configuration: `chunkOverlap` set to 100 characters (overlap between chunks to preserve context).  
  - Input: Document content from Default Data Loader.  
  - Output: List of text chunks for embedding.  
  - Edge cases: Very large documents might cause memory issues; improper splitting if input is not text.

- **Simple Vector Store**  
  - Type: Langchain Vector Store In-Memory  
  - Role: Stores the generated vector embeddings in an in-memory store with a specific memory key (`vector_store_key`).  
  - Configuration: Insert mode to add new embeddings.  
  - Input: Embeddings from Embeddings OpenAI and documents from Default Data Loader.  
  - Output: Confirmation of storage; vector index updated.  
  - Edge cases: Memory limits for large data; data loss on workflow restart as it is in-memory.

---

#### 2.2 AI Chatbot for Question Answering

**Overview:**  
This block receives user questions via webhook, authenticates the request, retrieves relevant document chunks using vector search, generates an AI answer constrained to document content, and sends the response back to the user’s chat platform.

**Nodes Involved:**  
- Webhook  
- Edit Fields  
- AI Agent (Langchain Agent)  
- Window Buffer Memory (Langchain Memory Buffer)  
- Simple Vector Store2 (Langchain Vector Store In-Memory)  
- Embeddings OpenAI1  
- OpenAI Chat Model1  
- Answer questions with a vector store (Langchain Tool Vector Store)  
- Is AI Agent output exist? (Code node)  
- Token Authentication (HTTP Request)  
- Send to Chat App (HTTP Request)  

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Entry point for external systems to send user questions to the chatbot.  
  - Configuration: HTTP POST method, unique webhook path `bfb0e32d-659b-4fc5-a7a3-695c55137855`.  
  - Input: User query payload (expected JSON with chat message content and metadata).  
  - Output: JSON data passed downstream.  
  - Edge cases: Unauthorized requests, invalid payload format, webhook URL exposure.

- **Edit Fields**  
  - Type: Set node  
  - Role: Extracts and maps the user’s question and session ID from incoming webhook JSON.  
  - Configuration: Assigns `chatInput` from `body.Data.ChatMessage.Content`, and `sessionId` from `body.Data.ChatMessage.RoomId`.  
  - Input: Webhook JSON.  
  - Output: Simplified JSON with `chatInput` and `sessionId`.  
  - Edge cases: Missing fields in input JSON, malformed data.

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Core AI logic that answers the user’s question using only the knowledge base documents retrieved.  
  - Configuration:  
    - System message instructs the agent to be factual, friendly, and answer strictly based on document content.  
    - Uses GPT-4o-mini model.  
  - Input: Processed user question and retrieved context from vector store and memory.  
  - Output: AI-generated answer text.  
  - Edge cases: API limits, hallucination if context is insufficient, model unavailability.

- **Window Buffer Memory**  
  - Type: Langchain Memory Buffer (Window)  
  - Role: Maintains conversation history (context window) to provide memory of previous messages for coherent multi-turn dialogue.  
  - Configuration: Default settings (window memory) with no additional parameters specified.  
  - Input: Prior conversation data and current input.  
  - Output: Contextual memory passed to AI agent.  
  - Edge cases: Memory size limits, losing older context, session mismatches.

- **Simple Vector Store2**  
  - Type: Langchain Vector Store In-Memory  
  - Role: Retrieves relevant document embeddings for the user’s question from the stored knowledge base.  
  - Configuration: Uses the same memory key `vector_store_key` as the first vector store for consistency.  
  - Input: Embeddings generated from the question (`Embeddings OpenAI1`).  
  - Output: Matching document chunks for AI agent.  
  - Edge cases: No relevant documents found, memory loss on restart.

- **Embeddings OpenAI1**  
  - Type: OpenAI Embeddings  
  - Role: Converts the user's question into vector embeddings for similarity search.  
  - Configuration: Default OpenAI options and credentials.  
  - Input: User question text (`chatInput`).  
  - Output: Vector representation of the question.  
  - Edge cases: API quota exceeded, malformed input.

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Model  
  - Role: Generates natural language responses from the retrieved document context.  
  - Configuration: Uses GPT-4o-mini model with default options.  
  - Input: Document chunks from vector store search results.  
  - Output: AI-generated text response.  
  - Edge cases: API errors, rate limits, hallucinations if context is poor.

- **Answer questions with a vector store**  
  - Type: Langchain Tool Vector Store  
  - Role: Performs the semantic search on the vector store to find document chunks relevant to the user question.  
  - Configuration: Default settings; uses `Simple Vector Store2` as vector source.  
  - Input: Question embeddings from `Embeddings OpenAI1`.  
  - Output: Relevant document text snippets.  
  - Edge cases: No results found, incorrect matches.

- **Is AI Agent output exist?**  
  - Type: Code node (JavaScript)  
  - Role: Validates the AI Agent’s output; if no output present, returns a fallback message.  
  - Configuration: Returns `"Sorry, I didn’t understand your message. Please try again or send a text."` if no AI output.  
  - Input: Output from AI Agent node.  
  - Output: Validated or fallback response content.  
  - Edge cases: Code errors, unexpected input data structure.

- **Token Authentication**  
  - Type: HTTP Request  
  - Role: Authenticates incoming requests by validating tokens with an external authentication service.  
  - Configuration:  
    - POST method with form-urlencoded body including `grant_type`, `client_id`, `client_secret`.  
    - Sends `Ocp-Apim-Subscription-Key` header.  
    - URL and credentials placeholders require user customization.  
  - Input: Message content from `Is AI Agent output exist?`.  
  - Output: Access token or authentication status.  
  - Edge cases: Invalid credentials, network errors, token expiry.

- **Send to Chat App**  
  - Type: HTTP Request  
  - Role: Sends the AI-generated answer back to the user’s chat application or front-end interface.  
  - Configuration:  
    - POST method with JSON body including `roomId`, `content`, `type`, `platform`, and `companyId` extracted from webhook and validation nodes.  
    - Sends authorization header with bearer token and subscription key.  
    - Target URL is a placeholder, must be configured.  
  - Input: Authenticated token and AI answer content.  
  - Output: Confirmation or status from chat app endpoint.  
  - Edge cases: Endpoint unavailability, authorization failures, malformed request.

---

### 3. Summary Table

| Node Name                          | Node Type                            | Functional Role                                   | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                   |
|-----------------------------------|------------------------------------|--------------------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------|
| File created in the Folder         | Google Drive Trigger               | Triggers on new file creation in folder          | None                             | Search Files in your Google Drive Folder | Part 1: File Upload and Knowledge Preparation                                                 |
| File updated in the Folder         | Google Drive Trigger               | Triggers on file update in folder                  | None                             | Search Files in your Google Drive Folder | Part 1: File Upload and Knowledge Preparation                                                 |
| Search Files in your Google Drive Folder | Google Drive                     | Searches files in target folder                    | File created/updated in the Folder | Download Files                     | Part 1: File Upload and Knowledge Preparation                                                 |
| Download Files                    | Google Drive                      | Downloads file content                             | Search Files in your Google Drive Folder | Simple Vector Store               | Part 1: File Upload and Knowledge Preparation                                                 |
| Embeddings OpenAI                | OpenAI Embeddings (Langchain)      | Creates vector embeddings from document chunks    | Default Data Loader               | Simple Vector Store               | Part 1: File Upload and Knowledge Preparation                                                 |
| Default Data Loader              | Langchain Document Loader           | Loads document content into chunks                 | Recursive Character Text Splitter | Simple Vector Store               | Part 1: File Upload and Knowledge Preparation                                                 |
| Recursive Character Text Splitter | Langchain Text Splitter             | Splits documents into overlapping chunks           | Default Data Loader              | Default Data Loader               | Part 1: File Upload and Knowledge Preparation                                                 |
| Simple Vector Store             | Langchain Vector Store In-Memory    | Stores document vector embeddings                   | Download Files, Embeddings OpenAI, Default Data Loader | None                            | Part 1: File Upload and Knowledge Preparation                                                 |
| Webhook                         | Webhook                            | Receives incoming user questions                    | None                            | Edit Fields                      | Part 2: AI Chatbot for Question Answering                                                    |
| Edit Fields                     | Set                               | Extracts user question and session ID               | Webhook                       | AI Agent                        | Part 2: AI Chatbot for Question Answering                                                    |
| AI Agent                       | Langchain Agent                    | Generates AI responses based on document context    | Edit Fields, Window Buffer Memory, Answer questions with a vector store | Is AI Agent output exist?       | Part 2: AI Chatbot for Question Answering                                                    |
| Window Buffer Memory           | Langchain Memory Buffer            | Maintains conversation memory                        | None (used by AI Agent)          | AI Agent                        | Part 2: AI Chatbot for Question Answering                                                    |
| Simple Vector Store2           | Langchain Vector Store In-Memory   | Retrieves document embeddings for question search   | Embeddings OpenAI1               | Answer questions with a vector store | Part 2: AI Chatbot for Question Answering                                                    |
| Embeddings OpenAI1             | OpenAI Embeddings (Langchain)      | Converts user question to embeddings                 | Edit Fields                    | Simple Vector Store2             | Part 2: AI Chatbot for Question Answering                                                    |
| OpenAI Chat Model1             | OpenAI Chat Model (Langchain)      | Generates AI text response from retrieved documents | Answer questions with a vector store | AI Agent                      | Part 2: AI Chatbot for Question Answering                                                    |
| Answer questions with a vector store | Langchain Tool Vector Store       | Performs vector search to find relevant document chunks | Simple Vector Store2, Embeddings OpenAI1 | AI Agent                      | Part 2: AI Chatbot for Question Answering                                                    |
| Is AI Agent output exist?      | Code                              | Validates AI output or returns fallback message     | AI Agent                        | Token Authentication            | Part 2: AI Chatbot for Question Answering                                                    |
| Token Authentication          | HTTP Request                      | Authenticates incoming request token                | Is AI Agent output exist?        | Send to Chat App                | Part 2: AI Chatbot for Question Answering                                                    |
| Send to Chat App               | HTTP Request                      | Sends AI response back to chat application          | Token Authentication            | None                            | Part 2: AI Chatbot for Question Answering                                                    |
| Sticky Note                   | Sticky Note                       | Configuration guide after importing the template    | None                            | None                            | See detailed setup instructions for Google Drive and OpenAI credentials                      |
| Sticky Note1                  | Sticky Note                       | Detailed Google Drive OAuth2 connection setup       | None                            | None                            | Step-by-step OAuth2 credential setup for Google Drive                                        |
| Sticky Note2                  | Sticky Note                       | General workflow description and use case overview  | None                            | None                            | Explains workflow purpose and capabilities                                                  |
| Sticky Note3                  | Sticky Note                       | Part 1 logic summary table                           | None                            | None                            | Summarizes file upload and knowledge base preparation nodes                                 |
| Sticky Note4                  | Sticky Note                       | Part 2 logic summary table                           | None                            | None                            | Summarizes AI chatbot question answering nodes                                             |
| Sticky Note5                  | Sticky Note                       | Webhook URL customization and token authentication tips | None                            | None                            | Guidance on webhook URL and token authentication customization                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Triggers:**  
   - Create two **Google Drive Trigger** nodes:  
     - One set to trigger on **fileCreated** in the target folder (ID `1Ve5ZLtQwpK5Hq4YxZTNRduRgPlSMZ1hJ`).  
     - One set to trigger on **fileUpdated** in the same folder.  
   - Configure both to poll every minute.

2. **Search Files in Google Drive Folder:**  
   - Add a **Google Drive** node configured to search files within the folder ID specified above (`1Ve5ZLtQwpK5Hq4YxZTNRduRgPlSMZ1hJ`).  
   - Connect both triggers to this node’s input.

3. **Download Files:**  
   - Add a **Google Drive** node with operation **download**.  
   - Configure the file ID dynamically with expression `{{$json["id"]}}` to use the ID from the previous search node.  
   - Connect the search node to this download node.

4. **Recursive Character Text Splitter:**  
   - Add a **Langchain Recursive Character Text Splitter** node.  
   - Set `chunkOverlap` to 100 characters.  
   - Connect the output of a document loader (next step) to this node’s input.

5. **Default Data Loader:**  
   - Add a **Langchain Default Data Loader** node.  
   - Set `dataType` to `binary` and `binaryMode` to `specificField` (default).  
   - Connect Recursive Character Text Splitter to this node.

6. **Embeddings OpenAI:**  
   - Add a **Langchain Embeddings OpenAI** node.  
   - Use your OpenAI API credential.  
   - Connect Default Data Loader to this node’s input.

7. **Simple Vector Store:**  
   - Add a **Langchain Vector Store In-Memory** node.  
   - Set mode to `insert`.  
   - Use memory key `vector_store_key`.  
   - Connect the Embeddings OpenAI node to this node’s `ai_embedding` input.  
   - Connect the Default Data Loader node to this node’s `ai_document` input.  
   - Connect Download Files node main output to the vector store main input to associate files with embeddings.

8. **Add Webhook for Incoming Questions:**  
   - Add a **Webhook** node. Set HTTP method to POST and assign a unique path (e.g., `/bfb0e32d-659b-4fc5-a7a3-695c55137855`).  
   - This will be the chatbot’s endpoint.

9. **Edit Fields:**  
   - Add a **Set** node after Webhook.  
   - Map the incoming JSON fields:  
     - `chatInput` = `{{$json["body"]["Data"]["ChatMessage"]["Content"]}}`  
     - `sessionId` = `{{$json["body"]["Data"]["ChatMessage"]["RoomId"]}}`

10. **Window Buffer Memory:**  
    - Add a **Langchain Memory Buffer Window** node to store conversation context.  
    - Connect it as a memory input for the AI Agent.

11. **Embeddings OpenAI1 (for questions):**  
    - Add another **Langchain Embeddings OpenAI** node for question embeddings.  
    - Use the same OpenAI credential.  
    - Connect `chatInput` from Edit Fields node as input.

12. **Simple Vector Store2 (for retrieval):**  
    - Add a second **Langchain Vector Store In-Memory** node.  
    - Use the same memory key `vector_store_key`.  
    - Connect Embeddings OpenAI1 node to its `ai_embedding` input.

13. **Answer questions with a vector store:**  
    - Add a **Langchain Tool Vector Store** node configured to search the vector store for relevant document chunks.  
    - Connect Simple Vector Store2 to its vector store input.  
    - Connect OpenAI Chat Model1 (next step) to its `ai_languageModel` input.

14. **OpenAI Chat Model1:**  
    - Add **Langchain OpenAI Chat Model** node with model `gpt-4o-mini`.  
    - Connect output of vector store search node to this node.

15. **AI Agent:**  
    - Add a **Langchain Agent** node.  
    - Configure system message instructing factual, document-based responses in a friendly tone.  
    - Use model `gpt-4o-mini`.  
    - Connect Edit Fields node main output, Window Buffer Memory (`ai_memory`), and Answer questions with a vector store (`ai_tool`) to AI Agent inputs.

16. **Is AI Agent output exist? (Validation):**  
    - Add a **Code** node to check if AI Agent output exists or fallback.  
    - Paste JS code to return fallback message if no output.

17. **Token Authentication:**  
    - Add an **HTTP Request** node to authenticate incoming requests.  
    - Configure POST method with form-urlencoded body for client credentials grant.  
    - Set header with your subscription key.  
    - Use your authentication service URL and credentials.

18. **Send to Chat App:**  
    - Add another **HTTP Request** node to send answers back to chat platform.  
    - POST method with JSON body containing `roomId`, `content`, `type`, `platform`, and `companyId`.  
    - Use Authorization header with bearer token from authentication node.  
    - Configure the target chat app URL.

19. **Connect Nodes:**  
    - Connect Webhook > Edit Fields > AI Agent > Is AI Agent output exist? > Token Authentication > Send to Chat App.

20. **Credentials Setup:**  
    - Create Google Drive OAuth2 credentials in n8n (per Sticky Note1 instructions).  
    - Create OpenAI API credentials with your API key.  
    - Use these credentials in all Google Drive nodes and OpenAI nodes respectively.

21. **Other Settings:**  
    - Customize webhook URLs, token authentication URLs, and chat app URLs according to your environment.  
    - Ensure folder ID in Google Drive nodes matches your target folder.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| ## Google Drive Connection Setup Guide for OAuth2 Credentials in n8n (Detailed step-by-step instructions including Google Cloud project creation, API enabling, OAuth consent screen configuration, credential creation, and adding redirect URIs)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note1 (Positioned near Google Drive nodes)                                                           |
| ## Configuration Guide After Importing the Template (Includes how to set Google Drive credentials, specify files/folders, connect OpenAI credentials, and use them in nodes)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note (Positioned top right)                                                                           |
| ## Simple AI Chatbot Using Google Drive Knowledge in n8n (Overview explaining the workflow’s purpose: automatic file updates, knowledge base creation, AI chatbot responding only based on file content)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note2 (Positioned near top center)                                                                    |
| ### Part 1 Summary Table (Detailed explanation of each node involved in file upload and knowledge preparation)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note3 (Positioned near Part 1 nodes)                                                                   |
| ### Part 2 Summary Table (Detailed explanation of nodes involved in AI question answering, webhook usage, authentication, and sending responses)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note4 (Positioned near Part 2 nodes)                                                                   |
| ### Webhook URL Customization & Token Authentication Tips (Instructions on customizing webhook path, securing requests, and configuring response endpoints)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note5 (Positioned near webhook and authentication nodes)                                              |
| If 403:access denied error occurs during Google Drive OAuth2 setup, ensure your email is added as a test user in the OAuth consent screen in Google Cloud Console.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note1 content                                                                                           |
| Recommended best practice: Use environment variables or `Set` nodes to manage endpoint URLs and keys to simplify maintenance and improve security.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note5 and Sticky Note                                                                                   |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, adhering strictly to applicable content policies without any illegal, offensive, or protected elements. All data handled is legal and public.