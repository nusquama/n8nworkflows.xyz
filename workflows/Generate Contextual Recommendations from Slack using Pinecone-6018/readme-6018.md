Generate Contextual Recommendations from Slack using Pinecone

https://n8nworkflows.xyz/workflows/generate-contextual-recommendations-from-slack-using-pinecone-6018


# Generate Contextual Recommendations from Slack using Pinecone

### 1. Workflow Overview

This workflow, titled **"Generate Contextual Recommendations from Slack using Pinecone"**, is designed to process Slack-triggered inputs to generate contextual AI-powered recommendations by leveraging vector search technology (Pinecone), Azure OpenAI models, and additional AI tools for ranking and parsing. It integrates Slack, Google Drive/Sheets, Pinecone vector store, and AI services to deliver refined contextual responses back to Slack.

Logical blocks grouped by functional role and node dependencies:

- **1.1 Slack Input Reception**
  - Captures user messages or events from Slack to initiate the workflow.

- **1.2 Initial AI Processing and Document Retrieval**
  - Downloads and extracts data files from Google Drive.
  - Uses Azure OpenAI embeddings and Pinecone vector store to retrieve relevant context.
  - Employs a Cohere reranker to improve vector search results.
  - Applies an AI Agent with a chat model and output parser to interpret and structure the response.

- **1.3 Contextual Querying and Recommendation Generation**
  - Further vector store querying and embedding generation with Pinecone and Azure OpenAI.
  - Uses another Cohere reranker for refined results.
  - Invokes a second AI Agent for advanced processing with output parsers for structured and auto-fixing parsing.

- **1.4 Final Output Processing and Slack Response**
  - Edits and formats output fields.
  - Sends the final message back to Slack.

- **1.5 Miscellaneous Nodes**
  - Sticky notes used for documentation/commentary.
  - Google Sheets integration for additional data retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 Slack Input Reception

- **Overview:**  
  Listens to Slack events/messages to trigger the workflow and start processing user input.

- **Nodes Involved:**  
  - Slack Trigger

- **Node Details:**  
  - **Slack Trigger**  
    - Type: Slack Trigger node (Webhook based)  
    - Configuration: Listens to Slack events configured via webhookId; no additional parameters defined.  
    - Inputs: External Slack event webhook  
    - Outputs: Passes Slack event payload to the AI Agent node  
    - Edge Cases: Slack webhook misconfiguration, missing permissions, event format changes or rate limits.

---

#### 1.2 Initial AI Processing and Document Retrieval

- **Overview:**  
  Processes input by downloading relevant files from Google Drive, extracting text, embedding it using Azure OpenAI, querying Pinecone vector store for relevant documents, reranking those documents using Cohere, and running an AI Agent to generate initial contextual recommendations.

- **Nodes Involved:**  
  - AI Agent  
  - Download file  
  - Extract from File  
  - Embeddings Azure OpenAI2  
  - Pinecone Vector Store2  
  - Reranker Cohere1  
  - Azure OpenAI Chat Model1  
  - Structured Output Parser1  
  - Get row(s) in sheet in Google Sheets

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain AI Agent (v2)  
    - Role: Receives Slack Trigger output to coordinate downstream AI nodes.  
    - Connections: Output connected to “Download file” node.  
    - Failure Modes: Model API errors, timeout, input data format errors.

  - **Download file**  
    - Type: Google Drive node  
    - Role: Downloads a file from Google Drive (likely containing context or knowledge base).  
    - Inputs: Receives from AI Agent.  
    - Outputs: Passes file to “Extract from File”.  
    - Edge Cases: File not found, permissions error, network issues.

  - **Extract from File**  
    - Type: Extract Text node  
    - Role: Extracts raw content from the downloaded file.  
    - Inputs: File from Download file node.  
    - Outputs: Text content to AI Agent1.  
    - Edge Cases: Unsupported file formats, extraction failure.

  - **AI Agent1**  
    - Type: LangChain AI Agent (v2)  
    - Role: Processes extracted text to create embeddings and search Pinecone.  
    - Inputs: Extracted text.  
    - Outputs: Connects to Pinecone Vector Store5 node.  
    - Edge Cases: Embedding generation failure, API quota.

  - **Embeddings Azure OpenAI2**  
    - Type: Azure OpenAI Embeddings node  
    - Role: Converts text to vector embeddings for Pinecone.  
    - Inputs: Text from AI Agent1.  
    - Outputs: Vectors to Pinecone Vector Store2.  
    - Edge Cases: API errors, input size limits.

  - **Pinecone Vector Store2**  
    - Type: Pinecone Vector Store node  
    - Role: Performs vector similarity search using embeddings.  
    - Inputs: Embeddings from Embeddings Azure OpenAI2; also receives reranker input.  
    - Outputs: Passes results to AI Agent1.  
    - Edge Cases: Connection issues, invalid index, timeouts.

  - **Reranker Cohere1**  
    - Type: Cohere Reranker node  
    - Role: Reranks vector search results for improved relevance.  
    - Inputs: Receives vector search results and reranks them.  
    - Outputs: Feeds reranked results back to Pinecone Vector Store2.  
    - Edge Cases: API errors, reranking failures.

  - **Azure OpenAI Chat Model1**  
    - Type: Azure OpenAI Chat node  
    - Role: Language model for chat-based AI reasoning in AI Agent1.  
    - Inputs: Connected as language model for AI Agent1.  
    - Outputs: Provides chat completions to AI Agent1.  
    - Edge Cases: API errors, rate limits.

  - **Structured Output Parser1**  
    - Type: Structured output parser  
    - Role: Parses AI Agent1’s output into structured data formats.  
    - Inputs: Output from AI Agent1.  
    - Outputs: Structured data for downstream usage.  
    - Edge Cases: Parsing failures due to unexpected output format.

  - **Get row(s) in sheet in Google Sheets**  
    - Type: Google Sheets node  
    - Role: Retrieves data rows from Google Sheets, possibly for enrichment or validation.  
    - Inputs: Triggered by AI Agent1’s tool output.  
    - Outputs: Data fed back into AI Agent1.  
    - Edge Cases: Sheet access issues, incorrect ranges, API limits.

---

#### 1.3 Contextual Querying and Recommendation Generation

- **Overview:**  
  Performs refined vector searches and ranking, uses additional AI Agents with advanced parsing to generate final recommendations.

- **Nodes Involved:**  
  - Pinecone Vector Store5  
  - Embeddings Azure OpenAI5  
  - Reranker Cohere3  
  - AI Agent3  
  - Azure OpenAI Chat Model3  
  - Auto-fixing Output Parser  
  - Structured Output Parser2

- **Node Details:**  
  - **Pinecone Vector Store5**  
    - Type: Pinecone Vector Store node (v1.3)  
    - Role: Vector similarity search for contextual data.  
    - Inputs: Receives embeddings and reranker inputs.  
    - Outputs: Connects to AI Agent3.  
    - Edge Cases: Index availability, connection failures.

  - **Embeddings Azure OpenAI5**  
    - Type: Azure OpenAI Embeddings node  
    - Role: Generates embeddings for query or context.  
    - Inputs: Input text or documents for vectorization.  
    - Outputs: To Pinecone Vector Store5.  
    - Edge Cases: API limits, malformed inputs.

  - **Reranker Cohere3**  
    - Type: Cohere Reranker node  
    - Role: Reranks Pinecone search results for best matches.  
    - Inputs: Vector store results.  
    - Outputs: Back to Pinecone Vector Store5.  
    - Edge Cases: API failures, unexpected input.

  - **AI Agent3**  
    - Type: LangChain AI Agent (v2)  
    - Role: Processes reranked vector search results to generate refined recommendations.  
    - Inputs: From Pinecone Vector Store5.  
    - Outputs: To Edit Fields node for formatting.  
    - Edge Cases: Model timeouts, output format errors.

  - **Azure OpenAI Chat Model3**  
    - Type: Azure OpenAI Chat node  
    - Role: Chat model supporting AI Agent3.  
    - Inputs: Connected as language model for AI Agent3.  
    - Outputs: To AI Agent3 and Auto-fixing Output Parser.  
    - Edge Cases: API latency, quota.

  - **Auto-fixing Output Parser**  
    - Type: Output parser with auto-fixing  
    - Role: Parses AI output and attempts auto-corrections on malformed outputs.  
    - Inputs: AI Agent3 output and Azure Chat Model3 output.  
    - Outputs: To Structured Output Parser2 and AI Agent3.  
    - Edge Cases: Parsing failure, infinite loops in auto-fixing.

  - **Structured Output Parser2**  
    - Type: Structured output parser (v1.3)  
    - Role: Final structured parsing of output for consistent formatting.  
    - Inputs: From Auto-fixing Output Parser.  
    - Outputs: To Edit Fields for final adjustments.  
    - Edge Cases: Parsing format issues.

---

#### 1.4 Final Output Processing and Slack Response

- **Overview:**  
  Prepares the final message format and sends the recommendation back to Slack.

- **Nodes Involved:**  
  - Edit Fields  
  - Send a message

- **Node Details:**  
  - **Edit Fields**  
    - Type: Set node  
    - Role: Adjusts or reformats fields from AI Agent3 output before sending.  
    - Inputs: From AI Agent3.  
    - Outputs: To Slack message node.  
    - Edge Cases: Misconfiguration of fields, null values.

  - **Send a message**  
    - Type: Slack node  
    - Role: Sends the final AI-generated recommendation message back to Slack channel or user.  
    - Inputs: From Edit Fields.  
    - Outputs: Slack message posted.  
    - Edge Cases: Slack API limits, permission errors, message formatting.

---

#### 1.5 Miscellaneous Nodes

- **Sticky Notes**  
  - Used for annotating or documenting parts of the workflow. No configuration or connections.

---

### 3. Summary Table

| Node Name                      | Node Type                             | Functional Role                           | Input Node(s)             | Output Node(s)             | Sticky Note |
|--------------------------------|-------------------------------------|------------------------------------------|---------------------------|----------------------------|-------------|
| Slack Trigger                  | Slack Trigger                       | Trigger workflow on Slack event          | -                         | AI Agent                   |             |
| AI Agent                      | LangChain AI Agent (v2)             | Initial AI processing                     | Slack Trigger             | Download file              |             |
| Download file                 | Google Drive                       | Downloads context file from Drive        | AI Agent                  | Extract from File          |             |
| Extract from File             | Extract Text                       | Extracts text content from file           | Download file             | AI Agent1                  |             |
| AI Agent1                    | LangChain AI Agent (v2)             | Processes extracted text, embedding, search | Extract from File         | Pinecone Vector Store5     |             |
| Embeddings Azure OpenAI2      | Azure OpenAI Embeddings             | Converts text to vectors                   | AI Agent1                 | Pinecone Vector Store2     |             |
| Pinecone Vector Store2         | Pinecone Vector Store (v1.3)        | Vector similarity search                   | Embeddings Azure OpenAI2, Reranker Cohere1 | AI Agent1        |             |
| Reranker Cohere1              | Cohere Reranker                    | Re-ranks vector search results             | Pinecone Vector Store2    | Pinecone Vector Store2     |             |
| Azure OpenAI Chat Model1       | Azure OpenAI Chat                  | Language model for AI Agent1               | AI Agent1                 | AI Agent1                  |             |
| Structured Output Parser1      | Structured Output Parser (v1.3)    | Parses AI Agent1 output                     | AI Agent1                 | -                         |             |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool              | Retrieves sheet rows for enrichment         | AI Agent1                 | AI Agent1                  |             |
| Pinecone Vector Store5         | Pinecone Vector Store (v1.3)        | Vector search for refined context          | Embeddings Azure OpenAI5, Reranker Cohere3 | AI Agent3        |             |
| Embeddings Azure OpenAI5      | Azure OpenAI Embeddings             | Generates embeddings for Pinecone          | AI Agent1                 | Pinecone Vector Store5     |             |
| Reranker Cohere3              | Cohere Reranker                    | Re-ranks refined vector search results     | Pinecone Vector Store5    | Pinecone Vector Store5     |             |
| AI Agent3                    | LangChain AI Agent (v2)             | Generates refined recommendations           | Pinecone Vector Store5    | Edit Fields                |             |
| Azure OpenAI Chat Model3       | Azure OpenAI Chat                  | Language model for AI Agent3                | AI Agent3                 | AI Agent3, Auto-fixing Output Parser |             |
| Auto-fixing Output Parser      | Output Parser (auto-fixing)          | Parses and auto-corrects AI output          | Azure OpenAI Chat Model3  | Structured Output Parser2, AI Agent3 |             |
| Structured Output Parser2      | Structured Output Parser (v1.3)    | Final structured parsing of output          | Auto-fixing Output Parser | Edit Fields                |             |
| Edit Fields                  | Set                              | Formats final output for Slack message       | AI Agent3, Structured Output Parser2 | Send a message             |             |
| Send a message               | Slack                            | Sends final message to Slack                  | Edit Fields               | -                         |             |
| Sticky Note                  | Sticky Note                      | Documentation/annotation                      | -                         | -                         |             |
| Sticky Note1                 | Sticky Note                      | Documentation/annotation                      | -                         | -                         |             |
| Sticky Note3                 | Sticky Note                      | Documentation/annotation                      | -                         | -                         |             |
| Sticky Note4                 | Sticky Note                      | Documentation/annotation                      | -                         | -                         |             |
| Sticky Note5                 | Sticky Note                      | Documentation/annotation                      | -                         | -                         |             |
| Sticky Note6                 | Sticky Note                      | Documentation/annotation                      | -                         | -                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node**  
   - Type: Slack Trigger  
   - Configure webhook to listen for messages/events in the target Slack workspace.  
   - No special parameters.

2. **Create AI Agent Node ("AI Agent")**  
   - Type: LangChain AI Agent (v2)  
   - Connect input from Slack Trigger.  
   - Configure with Azure OpenAI Chat Model node and Structured Output Parser node (steps below).

3. **Create Azure OpenAI Chat Model Node ("Azure OpenAI Chat Model")**  
   - Type: Azure OpenAI Chat  
   - Configure with Azure OpenAI credentials, specify deployment/model.  
   - Connect output to AI Agent’s language model input.

4. **Create Structured Output Parser ("Structured Output Parser")**  
   - Type: Structured Output Parser (v1.3)  
   - Configure parsing schema to structure AI output.  
   - Connect output to AI Agent’s output parser input.

5. **Connect AI Agent node output to Google Drive "Download file" node**  
   - Type: Google Drive node  
   - Configure with Google Drive OAuth2 credentials.  
   - Setup to download a specific file or dynamically based on AI Agent output.

6. **Create "Extract from File" node**  
   - Type: Extract Text from File  
   - Connect input from Download file.  
   - Configure supported formats (PDF, DOCX, etc.).

7. **Create second AI Agent Node ("AI Agent1")**  
   - Type: LangChain AI Agent (v2)  
   - Connect input from Extract from File.  
   - Configure with:  
     - Azure OpenAI Chat Model1 (Azure OpenAI Chat node)  
     - Structured Output Parser1  
     - Embeddings Azure OpenAI2  
     - Pinecone Vector Store2  
     - Reranker Cohere1  
     - Google Sheets node

8. **Create Azure OpenAI Chat Model1 node**  
   - Configure with Azure OpenAI credentials.  
   - Connect to AI Agent1 language model input.

9. **Create Structured Output Parser1**  
   - Configure parsing format.  
   - Connect to AI Agent1 output parser input.

10. **Create Embeddings Azure OpenAI2 node**  
    - Configure with Azure OpenAI credentials for embeddings endpoint.  
    - Connect to AI Agent1 embedding input.

11. **Create Pinecone Vector Store2 node**  
    - Configure with Pinecone API key, environment, and index name.  
    - Connect to Embeddings Azure OpenAI2 output and reranker input.

12. **Create Reranker Cohere1 node**  
    - Configure with Cohere API key.  
    - Connect to Pinecone Vector Store2 reranker input.

13. **Create Google Sheets node ("Get row(s) in sheet in Google Sheets")**  
    - Configure with Google Sheets credentials.  
    - Set sheet name and data range to retrieve context rows.  
    - Connect to AI Agent1 tool input.

14. **Create third AI Agent Node ("AI Agent3")**  
    - Type: LangChain AI Agent (v2)  
    - Connect input from Pinecone Vector Store5 (step 15).  
    - Configure with Azure OpenAI Chat Model3, Auto-fixing Output Parser, Structured Output Parser2.

15. **Create Pinecone Vector Store5 node**  
    - Configure similar to Pinecone Vector Store2 but for refined searches.  
    - Connect inputs from Embeddings Azure OpenAI5, Reranker Cohere3.

16. **Create Embeddings Azure OpenAI5 node**  
    - Configure with Azure OpenAI credentials for embedding generation.  
    - Connect input from AI Agent1 or relevant text.

17. **Create Reranker Cohere3 node**  
    - Configure with Cohere API key.  
    - Connect input/output to Pinecone Vector Store5.

18. **Create Azure OpenAI Chat Model3 node**  
    - Configure with Azure OpenAI credentials.  
    - Connect as language model for AI Agent3 and to Auto-fixing Output Parser.

19. **Create Auto-fixing Output Parser**  
    - Configure to parse and auto-correct AI Agent3 output.  
    - Connect output to Structured Output Parser2 and AI Agent3.

20. **Create Structured Output Parser2**  
    - Configure final parsing schema.  
    - Connect output to Edit Fields.

21. **Create Edit Fields node**  
    - Type: Set node  
    - Configure fields to format final output (e.g., message text, blocks).  
    - Connect input from AI Agent3.

22. **Create Slack "Send a message" node**  
    - Configure with Slack OAuth2 credentials.  
    - Set destination channel or user dynamically.  
    - Connect input from Edit Fields.

23. **Add Sticky Notes as needed**  
    - For documentation or explanation inside the editor.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow integrates Slack, Google Drive, Google Sheets, Pinecone, Azure OpenAI, and Cohere services.         | Workflow integration overview                    |
| Pinecone Vector Store nodes require API keys and index configuration on the Pinecone platform.                   | Pinecone docs: https://www.pinecone.io/docs/    |
| Azure OpenAI nodes require Azure subscription and deployment of OpenAI models.                                  | Azure OpenAI docs: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/ |
| Cohere Reranker improves ranking of vector search results by semantic relevance.                                | Cohere docs: https://docs.cohere.ai/             |
| Slack Trigger and Send a Message nodes require Slack app credentials with correct event subscriptions and scopes.| Slack API docs: https://api.slack.com/           |
| Google Drive and Google Sheets nodes require OAuth2 credentials with appropriate scopes.                        | Google API docs: https://developers.google.com/drive and https://developers.google.com/sheets/api |
| Auto-fixing output parser attempts to correct malformed AI outputs to ensure structured data format compliance.  | LangChain output parsers documentation           |

---

**Disclaimer:**  
The text provided is based exclusively on a workflow automated using n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.