E-Commerce Assistant for Shopify & WooCommerce with GPT-4o, Gemini & RAG

https://n8nworkflows.xyz/workflows/e-commerce-assistant-for-shopify---woocommerce-with-gpt-4o--gemini---rag-6100


# E-Commerce Assistant for Shopify & WooCommerce with GPT-4o, Gemini & RAG

---

### 1. Workflow Overview

This workflow is a **multi-platform E-Commerce conversational assistant** designed to support customer queries related to Shopify and WooCommerce stores, leveraging advanced AI models including GPT-4o-mini and Google Gemini, along with Retrieval-Augmented Generation (RAG) techniques for general knowledge queries.

**Target Use Cases:**
- Customer support for Shopify and WooCommerce e-commerce platforms.
- Answering general knowledge questions outside of e-commerce context.
- Providing product and order information via API integrations.
- Routing user queries intelligently based on platform context.

**Logical Blocks:**

- **1.1 Trigger & Initial Query Routing:** Receives user chat messages and classifies them into Shopify, WooCommerce, or General queries using an AI routing agent.
- **1.2 General Queries Processing (RAG):** Handles non-e-commerce queries with a knowledge base retrieval tool (Pinecone vector store) combined with embeddings and GPT-4o-mini.
- **1.3 Shopify Assistant:** Uses Google Gemini chat model and Shopify API tools (product listing, order info, GraphQL) to process Shopify-specific queries.
- **1.4 WooCommerce Assistant:** Uses Google Gemini chat model and WooCommerce API tools (order details, product listing) to handle WooCommerce-specific questions.
- **1.5 Memory Management:** Maintains session-based conversational memory buffers for the agents to provide context-aware responses.
- **1.6 Final Response Merge:** Consolidates output from the three branches (Shopify, WooCommerce, General) into a single response stream.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger & Initial Query Routing

**Overview:**  
Receives chat messages from users and routes queries to the appropriate e-commerce platform agent or general knowledge agent.

**Nodes Involved:**  
- When chat message received  
- Simple Memory  
- Router Model (GPT-4o-mini)  
- Initial Query Router (Agent)  
- Check if Its General query (IF Node)  
- Route To Shopify or WooCommerce (IF Node)

**Node Details:**

- **When chat message received**  
  - Type: `chatTrigger` from LangChain  
  - Role: Entry trigger node activated by user chat messages.  
  - Configuration: Exposes webhook to receive chat input and sessionId.  
  - Outputs: `chatInput` (user message), `sessionId` (unique conversation ID).  
  - Failures: Webhook downtime or invalid input format.  

- **Simple Memory**  
  - Type: `memoryBufferWindow`  
  - Role: Holds conversational memory for routing agent, enabling context retention.  
  - Configuration: Default buffer window, no custom session key.  
  - Inputs/Outputs: Connected from trigger to Router Model.  

- **Router Model**  
  - Type: `lmChatOpenAi` (OpenAI GPT-4o-mini)  
  - Role: Language model tasked with classifying user queries strictly into "SHOPIFY", "WOOCOMMERCE", or "None of them".  
  - Configuration: Uses GPT-4o-mini model with a custom prompt to enforce one-word classification.  
  - Key Expression: `User query: {{ $json.chatInput }}`  
  - Inputs: Memory buffer output  
  - Outputs: Classification string for routing.  
  - Failures: Model quota limits, malformed prompt issues.

- **Initial Query Router (Agent)**  
  - Type: `agent` (LangChain)  
  - Role: Implements the routing logic using the Router Model's output to determine query topic.  
  - Configuration: Prompt enforces strict one-word output routing.  
  - Inputs: Routed from Router Model; uses the chat input.  
  - Outputs: Single word output: `SHOPIFY`, `WOOCOMMERCE`, or `None of them`.  
  - Failures: Agent execution errors, prompt parsing issues.

- **Check if Its General query (IF Node)**  
  - Type: Conditional node  
  - Role: Branches workflow based on routing output.  
  - Condition: Checks if output is `SHOPIFY` or `WOOCOMMERCE`.  
  - Outputs:  
      - True: Continue to Shopify or WooCommerce routing  
      - False: Send to General Queries agent  
  - Failures: Expression evaluation errors.

- **Route To Shopify or WooCommerce (IF Node)**  
  - Type: Conditional node  
  - Role: Further routes based on exact string `SHOPIFY` or else defaults to WooCommerce.  
  - Condition: Output equals `SHOPIFY`.  
  - Outputs:  
      - True: Shopify Assistant Agent path  
      - False: WooCommerce Assistant Agent path  
  - Failures: Expression mismatches.

---

#### 2.2 General Queries Processing (RAG)

**Overview:**  
Handles queries unrelated to Shopify or WooCommerce, using a retrieval-augmented generation approach with Pinecone vector store and AWS Bedrock embeddings.

**Nodes Involved:**  
- General Queries (Agent)  
- Pinecone Vector Store  
- Embeddings AWS Bedrock  
- Memory RAG  
- GPT-4o-mini  
- Merge

**Node Details:**

- **General Queries (Agent)**  
  - Type: LangChain Agent  
  - Role: Answers general knowledge questions using retrieved context.  
  - Configuration: Uses GPT-4o-mini language model and Pinecone vector store as retrieval tool.  
  - Input: User query from chat trigger, context from Pinecone.  
  - Output: Final helpful answer text.  
  - Failures: Retrieval errors, model failures.

- **Pinecone Vector Store**  
  - Type: Vector database tool  
  - Role: Retrieves top 5 relevant documents based on query embedding.  
  - Configuration: Pinecone index "example-01" used, retrieves topK=5 items.  
  - Inputs: Vector embeddings from AWS Bedrock.  
  - Output: Context chunks for agent.  
  - Failures: API connection issues, index misconfiguration.

- **Embeddings AWS Bedrock**  
  - Type: Embeddings node  
  - Role: Converts user query text into vector embeddings using Amazon Titan Embed Text V2.  
  - Configuration: Model set to "amazon.titan-embed-text-v2:0".  
  - Inputs: User query text.  
  - Outputs: Vector embeddings.  
  - Failures: AWS credential errors, API limits.

- **Memory RAG**  
  - Type: Memory buffer window  
  - Role: Maintains conversation memory keyed by sessionId for context continuity.  
  - Configuration: Uses sessionId from chat trigger as custom session key.  
  - Inputs: User query and prior conversation context.  
  - Outputs: Context for agent.  
  - Failures: Missing or invalid sessionId.

- **GPT-4o-mini**  
  - Type: Language model node  
  - Role: Underlying LLM used by General Queries agent.  
  - Configuration: GPT-4o-mini model from OpenAI with default options.  
  - Inputs: Context + user query.  
  - Outputs: Generated textual answers.  
  - Failures: API quota limits, rate limits.

- **Merge**  
  - Type: Merge node  
  - Role: Combines outputs from all branches (General, Shopify, WooCommerce) into a single stream.  
  - Configuration: Number inputs = 3, merging by data index.  
  - Inputs: Output from General Queries (index 1).  
  - Outputs: Consolidated final response stream.  
  - Failures: Data loss if inputs missing.

---

#### 2.3 Shopify Assistant

**Overview:**  
Manages Shopify-related queries using Google Gemini chat model and Shopify API tools (REST and GraphQL).

**Nodes Involved:**  
- Shopify Assistant Agent (LangChain agent)  
- Shopify Chat Model (Google Gemini)  
- Get Order info (Shopify API node)  
- Fetch All Products (Shopify API node)  
- GraphQL (Shopify GraphQL API node)  
- Shopify Memory  
- Merge

**Node Details:**

- **Shopify Assistant Agent**  
  - Type: LangChain Agent  
  - Role: Main AI agent for Shopify queries, orchestrating tool usage and responses.  
  - Configuration:  
    - Instructions in prompt to extract order IDs from user text.  
    - Tools available: Get Order info, Fetch All Products, GraphQL.  
    - Rules: Use GraphQL for single product info, Fetch All Products only for bulk info.  
  - Inputs: User chat input, Shopify Memory.  
  - Outputs: Final answer or tool command.  
  - Failures: Incorrect tool usage, API errors.

- **Shopify Chat Model**  
  - Type: Language model node using Google Gemini  
  - Role: Provides conversational AI capability for Shopify agent.  
  - Configuration: Uses Google Palm API credentials.  
  - Inputs: Text prompts from agent.  
  - Outputs: Generated messages.  
  - Failures: API throttling, network.

- **Get Order info**  
  - Type: Shopify API node  
  - Role: Fetches order details by order ID from Shopify store.  
  - Configuration: Operation "get" on resource "order", orderId extracted from AI override.  
  - Credentials: Shopify Access Token (OAuth or API token).  
  - Inputs: Order ID from user query.  
  - Outputs: Order details JSON.  
  - Failures: Invalid order ID, auth errors.

- **Fetch All Products**  
  - Type: Shopify API node  
  - Role: Retrieves a limited list of products (limit: 25).  
  - Configuration: Operation "getAll" on resource "product", does not return all.  
  - Credentials: Shopify Access Token.  
  - Inputs: Triggered by Shopify Assistant Agent tool request.  
  - Outputs: Product list.  
  - Failures: API rate limits.

- **GraphQL**  
  - Type: GraphQL API node  
  - Role: Executes custom Shopify GraphQL queries for granular data.  
  - Configuration: Queries and variables dynamically generated by AI override inputs.  
  - Endpoint: Shopify admin GraphQL endpoint.  
  - Authentication: Header Auth credentials.  
  - Inputs: Query, Variables, Operation Name from AI override.  
  - Outputs: Query results.  
  - Failures: Query syntax errors, auth failures.

- **Shopify Memory**  
  - Type: Memory buffer window  
  - Role: Stores Shopify conversation history keyed by sessionId for context.  
  - Configuration: Uses sessionId from chat trigger as key.  
  - Inputs: Conversation context.  
  - Outputs: Context for agent.  
  - Failures: Missing sessionId.

- **Merge**  
  - Role: Consolidates Shopify output to main merge node for final response.

---

#### 2.4 WooCommerce Assistant

**Overview:**  
Handles WooCommerce-related queries using Google Gemini chat model and WooCommerce REST API tools.

**Nodes Involved:**  
- WooCommerce Assistant Agent (LangChain agent)  
- WooCommerce Chat Model (Google Gemini)  
- Fetch All Products2 (WooCommerce API)  
- Fetch Order Details (WooCommerce API)  
- WooCommerce Memory  
- Merge

**Node Details:**

- **WooCommerce Assistant Agent**  
  - Type: LangChain Agent  
  - Role: AI assistant managing WooCommerce queries.  
  - Configuration: Uses Google Gemini chat model and WooCommerce tools.  
  - Inputs: User chat input, WooCommerce Memory.  
  - Outputs: Text answer or tool commands.  
  - Failures: API errors, tool misuse.

- **WooCommerce Chat Model**  
  - Type: Language model node using Google Gemini  
  - Role: Conversational AI for WooCommerce assistant.  
  - Configuration: Google Palm API credentials.  
  - Inputs: Agent prompts.  
  - Outputs: Generated chat messages.  
  - Failures: API limits.

- **Fetch All Products2**  
  - Type: WooCommerce API node  
  - Role: Retrieves all WooCommerce products.  
  - Configuration: Operation "getAll" on products resource.  
  - Credentials: WooCommerce API credentials.  
  - Inputs: Triggered by WooCommerce Assistant Agent.  
  - Outputs: Product list JSON.  
  - Failures: Auth errors, API downtime.

- **Fetch Order Details**  
  - Type: WooCommerce API node  
  - Role: Retrieves order details by order ID.  
  - Configuration: Resource "order", operation "get", orderId from AI override.  
  - Credentials: WooCommerce API credentials.  
  - Inputs: Order ID extracted from user message.  
  - Outputs: Order detail JSON.  
  - Failures: Invalid orderId, auth failure.

- **WooCommerce Memory**  
  - Type: Memory buffer window  
  - Role: Maintains WooCommerce conversation context per session.  
  - Configuration: Session key from chat trigger sessionId.  
  - Inputs: Conversation data.  
  - Outputs: Context for agent.  
  - Failures: Missing sessionId.

- **Merge**  
  - Consolidates WooCommerce output to main merge node.

---

#### 2.5 Final Response Merge

**Overview:**  
Combines the outputs from Shopify Assistant, WooCommerce Assistant, and General Queries into one unified response stream for downstream usage (e.g., user interface display).

**Nodes Involved:**  
- Merge

**Node Details:**

- **Merge**  
  - Type: Merge node  
  - Role: Merges three input streams (Shopify, WooCommerce, General) into a single output.  
  - Configuration: Number of inputs = 3, merging by data index.  
  - Inputs: Final text responses from three branches.  
  - Outputs: Consolidated final response JSON.  
  - Failures: Missing inputs if any branch fails, incomplete data.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                         | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                  |
|----------------------------|--------------------------------------|---------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| When chat message received  | LangChain Chat Trigger                | Entry trigger for user chat message   |                              | Initial Query Router           | Trigger: Activates workflow on new user message; outputs chatInput and sessionId            |
| Simple Memory              | Memory Buffer Window                  | Maintains memory for routing agent    | When chat message received    | Router Model                  |                                                                                              |
| Router Model               | LLM Chat (OpenAI GPT-4o-mini)        | Classifies query intent                | Simple Memory                | Initial Query Router           |                                                                                              |
| Initial Query Router       | LangChain Agent                      | Routes query to Shopify, WooCommerce, or General | Router Model                 | Check if Its General query     | Initial Routing: Outputs single word for routing: SHOPIFY, WOOCOMMERCE, None of them          |
| Check if Its General query | IF Node                             | Branches to General or E-commerce agents | Initial Query Router           | Route To Shopify or WooCommerce, General Queries | Checks if query is SHOPIFY or WOOCOMMERCE; else routes to General Queries                   |
| Route To Shopify or WooCommerce | IF Node                         | Routes to Shopify or WooCommerce agent| Check if Its General query    | Shopify Assistant Agent, WooCommerce Assistant Agent | Routes SHOPIFY to Shopify Assistant; else WooCommerce Assistant                              |
| General Queries            | LangChain Agent                      | Answers general knowledge questions   | Check if Its General query    | Merge                        | General Queries Agent & Tools using RAG with Pinecone vector store                          |
| Pinecone Vector Store      | Vector Store (Pinecone)              | Retrieves relevant documents for RAG  | Embeddings AWS Bedrock        | General Queries               | Pinecone Vector Store (Tool) searches knowledge base for relevant documents                 |
| Embeddings AWS Bedrock     | Embeddings Node (AWS Bedrock)        | Generates vector embeddings for query |                             | Pinecone Vector Store         |                                                                                              |
| Memory RAG                 | Memory Buffer Window                  | Stores conversation context for RAG   | When chat message received    | General Queries               |                                                                                              |
| GPT-4o-mini                | LLM Chat (OpenAI GPT-4o-mini)        | Language model for General Queries    |                             | General Queries               |                                                                                              |
| Shopify Assistant Agent    | LangChain Agent                      | Handles Shopify-specific queries      | Route To Shopify or WooCommerce | Merge                        | Shopify Agent & Tools: uses tools Get Order info, Fetch All Products, GraphQL               |
| Shopify Chat Model         | LLM Chat (Google Gemini)             | Chat model for Shopify assistant      | Shopify Assistant Agent       | Shopify Assistant Agent       |                                                                                              |
| Get Order info             | Shopify API Node                    | Fetch Shopify order details by ID     | Shopify Assistant Agent       | Shopify Assistant Agent       |                                                                                              |
| Fetch All Products         | Shopify API Node                    | Fetch Shopify product list             | Shopify Assistant Agent       | Shopify Assistant Agent       |                                                                                              |
| GraphQL                   | GraphQL API Node                    | Runs Shopify GraphQL queries           | Shopify Assistant Agent       | Shopify Assistant Agent       |                                                                                              |
| Shopify Memory            | Memory Buffer Window                 | Stores Shopify conversation context   | When chat message received    | Shopify Assistant Agent       |                                                                                              |
| WooCommerce Assistant Agent | LangChain Agent                   | Handles WooCommerce-specific queries  | Route To Shopify or WooCommerce | Merge                        | WooCommerce Agent & Tools: uses Fetch Order Details, Fetch All Products2                    |
| WooCommerce Chat Model    | LLM Chat (Google Gemini)             | Chat model for WooCommerce assistant  | WooCommerce Assistant Agent   | WooCommerce Assistant Agent   |                                                                                              |
| Fetch All Products2       | WooCommerce API Node                | Fetch WooCommerce product list         | WooCommerce Assistant Agent   | WooCommerce Assistant Agent   |                                                                                              |
| Fetch Order Details       | WooCommerce API Node                | Fetch WooCommerce order details by ID | WooCommerce Assistant Agent   | WooCommerce Assistant Agent   |                                                                                              |
| WooCommerce Memory        | Memory Buffer Window                | Stores WooCommerce conversation context| When chat message received    | WooCommerce Assistant Agent   |                                                                                              |
| Merge                     | Merge Node                        | Combines final responses into one     | Shopify Assistant Agent, WooCommerce Assistant Agent, General Queries |                               | Merge: consolidates Shopify, WooCommerce, General responses into a single output             |
| Sticky Note (multiple)    | Sticky Note                        | Documentation comments                 |                              |                               | See detailed sticky notes in the workflow for full explanations and links                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Node Type: LangChain Chat Trigger (`chatTrigger`)  
   - Configuration: Set webhook to receive user chat messages, output user message as `chatInput` and session as `sessionId`.

2. **Add Simple Memory Node**  
   - Type: Memory Buffer Window (`memoryBufferWindow`)  
   - Configuration: Use default buffer window; no custom session key.

3. **Add Router Model Node**  
   - Type: LLM Chat OpenAI (`lmChatOpenAi`)  
   - Credentials: OpenAI API with GPT-4o-mini access.  
   - Parameters: Use GPT-4o-mini model; prompt to classify user query strictly as "SHOPIFY", "WOOCOMMERCE", or "None of them" with one word output.  
   - Connect: Input from Simple Memory; output to Initial Query Router.

4. **Add Initial Query Router Agent Node**  
   - Type: LangChain Agent  
   - Parameters: Use prompt to route queries based on Router Model output.  
   - Connect: Input from Router Model; output to Check if Its General query node.

5. **Add Check if Its General Query IF Node**  
   - Type: IF Node  
   - Condition: Check if routing output equals "SHOPIFY" or "WOOCOMMERCE".  
   - True Output: Route To Shopify or WooCommerce IF Node.  
   - False Output: General Queries Agent node.

6. **Add Route To Shopify or WooCommerce IF Node**  
   - Type: IF Node  
   - Condition: Check if routing output equals "SHOPIFY".  
   - True Output: Shopify Assistant Agent node.  
   - False Output: WooCommerce Assistant Agent node.

7. **Add General Queries Agent Node**  
   - Type: LangChain Agent  
   - Credentials: OpenAI API with GPT-4o-mini.  
   - Parameters: Use RAG approach with Pinecone vector store for retrieval.  
   - Connect: Input from Check if Its General query false output.

8. **Add Embeddings AWS Bedrock Node**  
   - Type: Embeddings Node  
   - Credentials: AWS account with Bedrock API access.  
   - Parameters: Use model `amazon.titan-embed-text-v2:0`.  
   - Connect: Input from General Queries node's user query.

9. **Add Pinecone Vector Store Node**  
   - Type: Vector Store (Pinecone)  
   - Credentials: Pinecone API account.  
   - Parameters: Use index "example-01" with topK=5.  
   - Connect: Input from Embeddings AWS Bedrock node.

10. **Add Memory RAG Node**  
    - Type: Memory Buffer Window  
    - Parameters: Use sessionId from chat trigger as custom key.  
    - Connect: Input from When chat message received; output to General Queries agent.

11. **Add Shopify Assistant Agent Node**  
    - Type: LangChain Agent  
    - Credentials: Google Gemini API credentials.  
    - Parameters:  
      - Prompt instructing how to extract order IDs and use Shopify tools.  
      - Tools: Get Order info, Fetch All Products, GraphQL.  
    - Connect: Input from Route To Shopify or WooCommerce IF Node.

12. **Add Shopify Chat Model Node**  
    - Type: LLM Chat Google Gemini  
    - Credentials: Google Palm API.  
    - Connect: Linked internally to Shopify Assistant Agent.

13. **Add Get Order info Node**  
    - Type: Shopify API node  
    - Credentials: Shopify Access Token.  
    - Parameters: Operation "get" on "order" resource; orderId from AI override variable.  
    - Connect: Used as a tool by Shopify Assistant Agent.

14. **Add Fetch All Products Node**  
    - Type: Shopify API node  
    - Credentials: Shopify Access Token.  
    - Parameters: Operation "getAll" on "product" resource, limit 25.  
    - Connect: Tool for Shopify Assistant Agent.

15. **Add GraphQL Node**  
    - Type: GraphQL API node  
    - Credentials: Header Auth for Shopify admin API.  
    - Parameters: Dynamic query, variables, operation name from AI override.  
    - Connect: Tool for Shopify Assistant Agent.

16. **Add Shopify Memory Node**  
    - Type: Memory Buffer Window  
    - Parameters: Use sessionId as custom key.  
    - Connect: Input from chat trigger; output to Shopify Assistant Agent.

17. **Add WooCommerce Assistant Agent Node**  
    - Type: LangChain Agent  
    - Credentials: Google Gemini API.  
    - Parameters: Prompt to handle WooCommerce queries.  
    - Connect: Input from Route To Shopify or WooCommerce IF Node.

18. **Add WooCommerce Chat Model Node**  
    - Type: LLM Chat Google Gemini  
    - Credentials: Google Palm API.  
    - Connect: Internally linked to WooCommerce Assistant Agent.

19. **Add Fetch All Products2 Node**  
    - Type: WooCommerce API node  
    - Credentials: WooCommerce API credentials.  
    - Parameters: Operation "getAll" on products.  
    - Connect: Tool for WooCommerce Assistant Agent.

20. **Add Fetch Order Details Node**  
    - Type: WooCommerce API node  
    - Credentials: WooCommerce API credentials.  
    - Parameters: Operation "get" on "order" resource, orderId from AI override.  
    - Connect: Tool for WooCommerce Assistant Agent.

21. **Add WooCommerce Memory Node**  
    - Type: Memory Buffer Window  
    - Parameters: Use sessionId as custom key.  
    - Connect: Input from chat trigger; output to WooCommerce Assistant Agent.

22. **Add Merge Node**  
    - Type: Merge  
    - Parameters: Number inputs = 3 (General Queries, Shopify Agent, WooCommerce Agent)  
    - Connect: Inputs from all three final agent nodes; output is consolidated final answer.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses **GPT-4o-mini** and **Google Gemini** models for AI chat capabilities.           | Requires OpenAI and Google Palm API credentials respectively.                                   |
| Shopify API nodes require valid **Shopify Access Token** with appropriate permissions.              | Token stored in n8n credentials as `shopifyAccessTokenApi`.                                    |
| WooCommerce API nodes require WooCommerce REST API credentials.                                    | Credentials stored as `wooCommerceApi`.                                                         |
| Retrieval-Augmented Generation implemented via **Pinecone Vector Store** with AWS Bedrock embeddings.| Pinecone index named `example-01` and AWS Bedrock embedding model `amazon.titan-embed-text-v2:0`.|
| The workflow implements **session-based conversational memory** for context continuity.            | Session ID from chat trigger used as memory key for LangChain memory buffer nodes.              |
| Sticky notes in the workflow provide detailed explanations per block and node.                     | Refer to sticky notes for inline documentation inside n8n editor.                              |
| GraphQL node dynamically executes queries generated by AI for Shopify data retrieval.               | Requires Header Authentication with Shopify admin API token.                                   |
| The workflow is designed for modular extensibility to add other e-commerce platforms or tools.     | Routing agent can be updated with additional classification outputs and corresponding agents.  |

---

**Disclaimer:** The provided description is generated solely based on the n8n workflow JSON definition. The workflow complies with all content policies and handles only legal, public data interactions.