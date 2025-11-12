🤖 AI Powered RAG Chatbot for Your Docs + Google Drive + Gemini + Qdrant

https://n8nworkflows.xyz/workflows/---ai-powered-rag-chatbot-for-your-docs---google-drive---gemini---qdrant-2982


# 🤖 AI Powered RAG Chatbot for Your Docs + Google Drive + Gemini + Qdrant

### 1. Workflow Overview

This workflow implements a Retrieval-Augmented Generation (RAG) chatbot that integrates Google Drive document storage, Qdrant vector database, and Google Gemini AI to provide intelligent, context-aware conversational capabilities over user documents. It is designed for organizations needing to process large document repositories, extract metadata, store vector embeddings for efficient retrieval, and interact with users via a chat interface powered by advanced AI models.

The workflow is logically divided into three main blocks:

- **1.1 Document Processing & Storage**  
  Retrieves documents from a specified Google Drive folder, downloads and extracts their content, uses AI to extract metadata, splits text into chunks, generates embeddings, and stores vectors in Qdrant.

- **1.2 Vector Store Management & Deletion with Human Verification**  
  Manages vector data in Qdrant, including secure deletion of document vectors by file ID, with human-in-the-loop verification via Telegram notifications.

- **1.3 Intelligent Chat Interface**  
  Handles chat message reception, context retrieval from Qdrant, AI-powered response generation using Google Gemini and OpenAI models, chat history maintenance in Google Docs, and user response delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Processing & Storage

**Overview:**  
This block fetches documents from a Google Drive folder, downloads and extracts their text content, uses AI to extract rich metadata, splits the text into manageable chunks, generates vector embeddings, and inserts these vectors into the Qdrant vector store for later retrieval.

**Nodes Involved:**  
- Google Folder ID  
- Find File Ids in Google Drive Folder  
- Loop Over Items  
- Download File From Google Drive  
- Get File Contents  
- Extract Meta Data  
- Merge  
- Token Splitter  
- Data Loader  
- text-embeddings-3-large  
- Qdrant Vector Store  
- Wait  
- Send Completed Message  

**Node Details:**

- **Google Folder ID**  
  - Type: Set  
  - Role: Defines the Google Drive folder ID to process documents from.  
  - Configuration: Assigns a string variable `folder_id` with the target folder ID.  
  - Connections: Triggers "Find File Ids in Google Drive Folder".  
  - Edge cases: Folder ID must be valid and accessible; invalid ID causes no files found.

- **Find File Ids in Google Drive Folder**  
  - Type: Google Drive  
  - Role: Retrieves all file IDs from the specified Google Drive folder.  
  - Configuration: Uses OAuth2 credentials; filters by folder ID and drive "My Drive".  
  - Connections: Outputs file list to "File Id List" and "Qdrant Collection Name" and "Merge2".  
  - Edge cases: Permissions errors, empty folder, API rate limits.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes files one by one or in batches for downstream operations.  
  - Configuration: Default batch size; error handling continues on failure.  
  - Connections: Sends each file to "Download File From Google Drive" and "Send Completed Message".  
  - Edge cases: Large batch sizes may cause timeouts.

- **Download File From Google Drive**  
  - Type: Google Drive  
  - Role: Downloads the actual file content for each file ID.  
  - Configuration: Uses OAuth2 credentials; downloads file by ID.  
  - Connections: Outputs binary file data to "Get File Contents" and metadata to "Merge".  
  - Edge cases: File access denied, file deleted, network errors.

- **Get File Contents**  
  - Type: ExtractFromFile  
  - Role: Extracts text content from downloaded binary file.  
  - Configuration: Operation set to "text".  
  - Connections: Sends extracted text to "Extract Meta Data".  
  - Edge cases: Unsupported file formats, extraction failures.

- **Extract Meta Data**  
  - Type: Information Extractor (LangChain)  
  - Role: Uses AI to extract structured metadata from document text, including themes, topics, pain points, insights, conclusions, and keywords.  
  - Configuration: System prompt guides extraction; attributes defined explicitly.  
  - Connections: Outputs metadata JSON to "Merge".  
  - Edge cases: AI extraction may omit attributes if uncertain; prompt tuning may be needed.

- **Merge**  
  - Type: Merge  
  - Role: Combines metadata and file info for downstream processing.  
  - Configuration: Combine mode "combineAll".  
  - Connections: Outputs to "Token Splitter".  
  - Edge cases: Mismatched input lengths cause incomplete merges.

- **Token Splitter**  
  - Type: Text Splitter (Token-based)  
  - Role: Splits document text into chunks of 3000 tokens for embedding.  
  - Configuration: Chunk size 3000 tokens.  
  - Connections: Outputs chunks to "Data Loader".  
  - Edge cases: Very large documents may produce many chunks; chunk size tuning may be required.

- **Data Loader**  
  - Type: Document Data Loader (LangChain)  
  - Role: Prepares document chunks with metadata for vector embedding.  
  - Configuration: Metadata fields populated from extracted metadata and file info.  
  - Connections: Outputs to "Qdrant Vector Store".  
  - Edge cases: Missing metadata fields may reduce search quality.

- **text-embeddings-3-large**  
  - Type: OpenAI Embeddings  
  - Role: Generates vector embeddings for document chunks.  
  - Configuration: Model "text-embedding-3-large".  
  - Connections: Feeds embeddings to "Qdrant Vector Store".  
  - Edge cases: API key limits, rate limits, embedding failures.

- **Qdrant Vector Store**  
  - Type: Vector Store (Qdrant)  
  - Role: Inserts document vectors into Qdrant collection "nostr-damus-user-profiles".  
  - Configuration: Insert mode; uses Qdrant API credentials.  
  - Connections: Outputs to "Wait".  
  - Edge cases: Connection failures, collection not found, API errors.

- **Wait**  
  - Type: Wait  
  - Role: Introduces delay or webhook wait for asynchronous processing.  
  - Configuration: Webhook ID set; no parameters.  
  - Connections: Outputs to "Loop Over Items" to continue batch processing.  
  - Edge cases: Timeout or webhook not triggered.

- **Send Completed Message**  
  - Type: Telegram  
  - Role: Sends notification that Qdrant upsert completed.  
  - Configuration: Sends to configured Telegram chat ID; no attribution appended.  
  - Connections: Terminal node.  
  - Edge cases: Telegram API failures, invalid chat ID.

---

#### 2.2 Vector Store Management & Deletion with Human Verification

**Overview:**  
This block enables secure deletion of document vectors from Qdrant by file ID, requiring human approval via Telegram before proceeding. It ensures data integrity and prevents accidental data loss.

**Nodes Involved:**  
- Qdrant Collection Name  
- File Id List  
- Merge1  
- Confirm Qdrant Delete Points  
- If  
- Delete Qdrant Points by File ID  
- Merge2  
- Send Declined Message  

**Node Details:**

- **Qdrant Collection Name**  
  - Type: Set  
  - Role: Defines the Qdrant collection name for deletion operations.  
  - Configuration: Sets variable `qdrant_collection_name` to "nostr-damus-user-profiles".  
  - Connections: Outputs to "Merge1".  
  - Edge cases: Collection name must match existing Qdrant collection.

- **File Id List**  
  - Type: Summarize  
  - Role: Aggregates file IDs into a list for deletion.  
  - Configuration: Appends all file IDs from input.  
  - Connections: Outputs to "Merge1".  
  - Edge cases: Empty list means no deletion.

- **Merge1**  
  - Type: Merge  
  - Role: Combines collection name and file ID list for confirmation.  
  - Configuration: Combine by position.  
  - Connections: Outputs to "Confirm Qdrant Delete Points".  
  - Edge cases: Mismatched inputs cause incomplete data.

- **Confirm Qdrant Delete Points**  
  - Type: Telegram  
  - Role: Sends a warning message to Telegram chat requesting double approval to proceed with deletion.  
  - Configuration: Sends message with count of records to delete; waits for double approval with 15 minutes timeout.  
  - Connections: Outputs to "If" node.  
  - Edge cases: No approval or timeout cancels deletion.

- **If**  
  - Type: If  
  - Role: Checks if deletion was approved by user.  
  - Configuration: Condition on `data.approved` boolean true.  
  - Connections: If true, proceeds to "Delete Qdrant Points by File ID"; if false, to "Send Declined Message".  
  - Edge cases: Missing approval data causes false branch.

- **Delete Qdrant Points by File ID**  
  - Type: Code (JavaScript)  
  - Role: Executes custom code to delete points from Qdrant collection filtered by file IDs.  
  - Configuration: Uses LangChain QdrantVectorStore and OpenAIEmbeddings; connects to local Qdrant instance; deletes points matching metadata.file_id.  
  - Connections: Outputs to "Merge2".  
  - Edge cases: API errors, network failures, invalid file IDs, missing API keys.

- **Merge2**  
  - Type: Merge  
  - Role: Combines deletion results with file list for further processing.  
  - Configuration: Default merge.  
  - Connections: Outputs to "Loop Over Items" for further processing.  
  - Edge cases: Merge failures.

- **Send Declined Message**  
  - Type: Telegram  
  - Role: Notifies Telegram chat that deletion was declined.  
  - Configuration: Sends static text message; no attribution appended.  
  - Connections: Terminal node.  
  - Edge cases: Telegram API failures.

---

#### 2.3 Intelligent Chat Interface

**Overview:**  
This block manages chat message reception, context retrieval from Qdrant vector store, AI response generation using Google Gemini and OpenAI models, chat history updating in Google Docs, and user response delivery.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- Window Buffer Memory  
- Qdrant Vector Store Tool  
- Google Gemini Chat Model1  
- text-embeddings-3-large1  
- Update Chat History  
- Respond to User  

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (LangChain)  
  - Role: Entry point for incoming chat messages from users.  
  - Configuration: Default options; webhook ID assigned.  
  - Connections: Triggers "AI Agent".  
  - Edge cases: Webhook connectivity issues.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates AI response generation using tools and memory.  
  - Configuration: System message defines assistant role specialized in Nostr user profiles; uses "nostr_damus_user_profiles" tool for semantic search; concise, context-aware answers.  
  - Connections: Inputs from chat trigger and memory; outputs to "Update Chat History" and "Respond to User".  
  - Edge cases: Tool failures, memory overflow, prompt misalignment.

- **Window Buffer Memory**  
  - Type: Memory Buffer Window (LangChain)  
  - Role: Maintains conversational context window of last 40 messages for AI agent.  
  - Configuration: Context window length 40.  
  - Connections: Feeds memory to "AI Agent".  
  - Edge cases: Memory size limits, context truncation.

- **Qdrant Vector Store Tool**  
  - Type: Vector Store (Qdrant) as Tool (LangChain)  
  - Role: Provides semantic search capabilities over Qdrant collection "nostr-damus-user-profiles" as a tool for AI agent.  
  - Configuration: Retrieve mode "retrieve-as-tool"; topK=20; uses Qdrant API credentials.  
  - Connections: Used internally by "AI Agent".  
  - Edge cases: API failures, empty results.

- **Google Gemini Chat Model1**  
  - Type: Language Model Chat (Google Gemini)  
  - Role: Provides advanced chat completions with large max output tokens (8192).  
  - Configuration: Model "models/gemini-2.0-flash-exp"; uses Google Palm API credentials.  
  - Connections: Used by "AI Agent" as language model.  
  - Edge cases: API limits, latency.

- **text-embeddings-3-large1**  
  - Type: OpenAI Embeddings  
  - Role: Generates embeddings for chat inputs or context as needed.  
  - Configuration: Model "text-embedding-3-large"; uses OpenAI API credentials.  
  - Connections: Feeds embeddings to "Qdrant Vector Store Tool".  
  - Edge cases: API limits.

- **Update Chat History**  
  - Type: Google Docs  
  - Role: Appends chat input and AI response to a Google Docs document for persistent chat history.  
  - Configuration: Uses Google Docs OAuth2 credentials; updates document by inserting formatted text with timestamp, user input, and AI output.  
  - Connections: Outputs terminal.  
  - Edge cases: Document access errors, rate limits.

- **Respond to User**  
  - Type: Set  
  - Role: Prepares AI response text for delivery back to user.  
  - Configuration: Sets output field with AI-generated response.  
  - Connections: Terminal node.  
  - Edge cases: None significant.

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                                | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                          |
|--------------------------------|-----------------------------------|------------------------------------------------|--------------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| Data Loader                    | Document Data Loader (LangChain)  | Prepares document chunks with metadata         | Token Splitter                       | Qdrant Vector Store                   | ✨ Save Documents to Qdrant Vector Store                                                           |
| Token Splitter                 | Text Splitter (Token-based)       | Splits text into chunks                          | Merge                               | Data Loader                          |                                                                                                    |
| Qdrant Vector Store            | Vector Store (Qdrant)              | Inserts vectors into Qdrant                       | Data Loader                         | Wait                                | Perform Qdrant Vector Store Operations                                                             |
| Loop Over Items                | SplitInBatches                    | Batch processes files                            | Merge2, Wait                       | Send Completed Message, Download File From Google Drive |                                                                                                    |
| Wait                          | Wait                             | Waits for webhook or delay                        | Qdrant Vector Store                 | Loop Over Items                     |                                                                                                    |
| When clicking ‘Test workflow’ | Manual Trigger                   | Manual start trigger                             | None                               | Google Folder ID                    | 👍Start Here!                                                                                       |
| Google Gemini Chat Model       | LM Chat Google Gemini             | AI model for metadata extraction                  | Extract Meta Data                   | Extract Meta Data                   |                                                                                                    |
| Extract Meta Data             | Information Extractor (LangChain) | Extracts structured metadata from text           | Get File Contents                  | Merge                              | Extract Metadata for Qdrant Hybrid Search                                                          |
| Get File Contents             | ExtractFromFile                   | Extracts text from downloaded file                | Download File From Google Drive    | Extract Meta Data                   |                                                                                                    |
| Download File From Google Drive| Google Drive                     | Downloads files by ID                             | Loop Over Items                    | Get File Contents, Merge            | Google Drive                                                                                       |
| Find File Ids in Google Drive Folder | Google Drive               | Lists file IDs in folder                          | Google Folder ID                  | File Id List, Qdrant Collection Name, Merge2 | Google Drive                                                                                       |
| Google Folder ID              | Set                              | Sets Google Drive folder ID                       | When clicking ‘Test workflow’      | Find File Ids in Google Drive Folder |                                                                                                    |
| File Id List                 | Summarize                        | Aggregates file IDs for deletion                   | Find File Ids in Google Drive Folder | Merge1                            |                                                                                                    |
| Qdrant Collection Name        | Set                              | Sets Qdrant collection name for deletion          | Find File Ids in Google Drive Folder | Merge1                            | Prepare Qdrant Vector Store                                                                        |
| Merge                        | Merge                            | Combines metadata and file info                    | Extract Meta Data, Download File From Google Drive | Token Splitter               |                                                                                                    |
| Merge1                       | Merge                            | Combines collection name and file ID list          | File Id List, Qdrant Collection Name | Confirm Qdrant Delete Points       | Human In The Loop                                                                                   |
| Merge2                       | Merge                            | Combines deletion results with file list           | Delete Qdrant Points by File ID, Find File Ids in Google Drive Folder | Loop Over Items               |                                                                                                    |
| Confirm Qdrant Delete Points | Telegram                        | Sends deletion confirmation request with approval | Merge1                            | If                                 | Human In The Loop                                                                                   |
| If                           | If                              | Checks if deletion approved                         | Confirm Qdrant Delete Points       | Delete Qdrant Points by File ID, Send Declined Message |                                                                                                    |
| Delete Qdrant Points by File ID | Code (JavaScript)              | Deletes points from Qdrant by file ID               | If                               | Merge2                             | Delete From Qdrant Vector Store This operation can not be undone!!!                                |
| Send Declined Message         | Telegram                        | Notifies deletion declined                          | If                               | None                              |                                                                                                    |
| Send Completed Message        | Telegram                        | Notifies completion of vector store upsert          | Loop Over Items                   | None                              |                                                                                                    |
| When chat message received    | Chat Trigger (LangChain)         | Entry point for chat messages                       | None                             | AI Agent                          | 🗣️ Chat with Your Documents                                                                        |
| AI Agent                     | LangChain Agent                  | Orchestrates AI response generation                 | When chat message received, Window Buffer Memory, Qdrant Vector Store Tool, Google Gemini Chat Model1, text-embeddings-3-large1 | Update Chat History, Respond to User |                                                                                                    |
| Window Buffer Memory          | Memory Buffer Window (LangChain) | Maintains conversational context                    | None                             | AI Agent                         |                                                                                                    |
| Qdrant Vector Store Tool      | Vector Store (Qdrant) as Tool    | Provides semantic search tool for AI agent          | None                             | AI Agent                         |                                                                                                    |
| Google Gemini Chat Model1     | LM Chat Google Gemini             | AI model for chat completions                         | None                             | AI Agent                         |                                                                                                    |
| text-embeddings-3-large1      | OpenAI Embeddings                | Generates embeddings for chat context                 | None                             | Qdrant Vector Store Tool         |                                                                                                    |
| Update Chat History           | Google Docs                     | Appends chat history to Google Docs document          | AI Agent                         | None                              | Save Chat History                                                                                   |
| Respond to User               | Set                             | Prepares AI response for user delivery                 | AI Agent                         | None                              |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Set node named "Google Folder ID"**  
   - Assign a string variable `folder_id` with your target Google Drive folder ID.

2. **Add a Google Drive node "Find File Ids in Google Drive Folder"**  
   - Operation: List files in folder  
   - Filter by `folderId` set to `={{ $json.folder_id }}` and drive "My Drive"  
   - Connect "Google Folder ID" output to this node.  
   - Use Google Drive OAuth2 credentials.

3. **Add a Summarize node "File Id List"**  
   - Field to summarize: `id` with aggregation "append"  
   - Connect output of "Find File Ids in Google Drive Folder" to this node.

4. **Add a Set node "Qdrant Collection Name"**  
   - Assign string variable `qdrant_collection_name` with your Qdrant collection name (e.g., "nostr-damus-user-profiles").  
   - Connect output of "Find File Ids in Google Drive Folder" to this node.

5. **Add a Merge node "Merge1"**  
   - Mode: combineByPosition  
   - Connect outputs of "File Id List" and "Qdrant Collection Name" to this node.

6. **Add a Telegram node "Confirm Qdrant Delete Points"**  
   - Operation: sendAndWait with double approval  
   - Message: Warn user about deletion with count of records and collection name from inputs.  
   - Set approval timeout to 15 minutes.  
   - Connect "Merge1" output to this node.  
   - Use Telegram API credentials.

7. **Add an If node "If"**  
   - Condition: Check if `data.approved` is true.  
   - Connect "Confirm Qdrant Delete Points" output to this node.

8. **Add a Code node "Delete Qdrant Points by File ID"**  
   - Paste provided JavaScript code that deletes points from Qdrant filtered by file IDs.  
   - Configure OpenAI API key and Qdrant connection details inside the code.  
   - Connect "If" true output to this node.

9. **Add a Merge node "Merge2"**  
   - Connect "Delete Qdrant Points by File ID" and "Find File Ids in Google Drive Folder" outputs to this node.

10. **Add a SplitInBatches node "Loop Over Items"**  
    - Connect "Merge2" output to this node.  
    - Connect "Loop Over Items" output to "Download File From Google Drive" and "Send Completed Message".

11. **Add a Google Drive node "Download File From Google Drive"**  
    - Operation: download file by ID `={{ $json.id }}`  
    - Use Google Drive OAuth2 credentials.

12. **Add an ExtractFromFile node "Get File Contents"**  
    - Operation: extract text from binary file.  
    - Connect "Download File From Google Drive" output to this node.

13. **Add an Information Extractor node "Extract Meta Data"**  
    - Configure system prompt to extract metadata attributes: overarching_theme, recurring_topics, pain_points, analytical_insights, conclusion, keywords.  
    - Connect "Get File Contents" output to this node.

14. **Add a Merge node "Merge"**  
    - Mode: combineAll  
    - Connect "Extract Meta Data" and "Download File From Google Drive" outputs to this node.

15. **Add a Text Splitter node "Token Splitter"**  
    - Chunk size: 3000 tokens  
    - Connect "Merge" output to this node.

16. **Add a Document Data Loader node "Data Loader"**  
    - Set metadata fields from extracted metadata and file info (file_id, pubkey, overarching_theme, etc.)  
    - Data type: binary, specific field.  
    - Connect "Token Splitter" output to this node.

17. **Add an OpenAI Embeddings node "text-embeddings-3-large"**  
    - Model: text-embedding-3-large  
    - Connect "Data Loader" output to this node.  
    - Use OpenAI API credentials.

18. **Add a Qdrant Vector Store node "Qdrant Vector Store"**  
    - Mode: insert  
    - Collection: nostr-damus-user-profiles (or your collection)  
    - Connect "text-embeddings-3-large" output to this node.  
    - Use Qdrant API credentials.

19. **Add a Wait node "Wait"**  
    - Configure webhook ID for asynchronous continuation.  
    - Connect "Qdrant Vector Store" output to this node.

20. **Connect "Wait" output back to "Loop Over Items"**  
    - This enables batch processing loop.

21. **Add a Telegram node "Send Completed Message"**  
    - Message: "Qdrant vector store upsert completed"  
    - Connect "Loop Over Items" output to this node.  
    - Use Telegram API credentials.

22. **Add a Chat Trigger node "When chat message received"**  
    - Configure webhook ID.  
    - Entry point for chat messages.

23. **Add a Memory Buffer Window node "Window Buffer Memory"**  
    - Context window length: 40 messages.

24. **Add a Qdrant Vector Store Tool node "Qdrant Vector Store Tool"**  
    - Mode: retrieve-as-tool  
    - TopK: 20  
    - Collection: nostr-damus-user-profiles  
    - Use Qdrant API credentials.

25. **Add a Google Gemini Chat Model node "Google Gemini Chat Model1"**  
    - Model: models/gemini-2.0-flash-exp  
    - Max output tokens: 8192  
    - Use Google Palm API credentials.

26. **Add an OpenAI Embeddings node "text-embeddings-3-large1"**  
    - Model: text-embedding-3-large  
    - Use OpenAI API credentials.

27. **Add a LangChain Agent node "AI Agent"**  
    - System message: specialized assistant for Nostr user profiles with tool "nostr_damus_user_profiles"  
    - Connect inputs: chat trigger, memory buffer, vector store tool, Gemini chat model, embeddings.  
    - Outputs: "Update Chat History" and "Respond to User".

28. **Add a Google Docs node "Update Chat History"**  
    - Operation: update document by inserting formatted chat history text with timestamp, user input, and AI output.  
    - Use Google Docs OAuth2 credentials.

29. **Add a Set node "Respond to User"**  
    - Assign output field with AI response text.

30. **Connect "Update Chat History" and "Respond to User" outputs as terminal nodes.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini AI (PaLM API) for advanced chat completions and metadata extraction.                            | Requires Google Palm API credentials.                                                          |
| Qdrant vector store is used for efficient semantic search and storage of document embeddings.                                     | Requires Qdrant API credentials and a running Qdrant instance accessible at configured URL.    |
| Telegram bot integration provides human-in-the-loop verification for deletion and notifications on completion or decline.       | Requires Telegram bot API token and chat ID environment variable `TELEGRAM_CHAT_ID`.           |
| OpenAI API is used for generating embeddings and optionally for deletion code logic.                                              | Requires OpenAI API key configured in respective nodes.                                       |
| The workflow supports batch processing of documents for scalability.                                                              | Batch size can be adjusted in "Loop Over Items" node.                                         |
| Chat history is maintained in a Google Docs document for audit and reference.                                                     | Google Docs OAuth2 credentials required; document ID must be set in "Update Chat History".    |
| The deletion operation is irreversible and requires explicit double approval via Telegram to proceed.                            | See "Confirm Qdrant Delete Points" node configuration.                                        |
| The workflow includes sticky notes with detailed explanations and setup instructions for ease of maintenance and understanding. |                                                                                               |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.