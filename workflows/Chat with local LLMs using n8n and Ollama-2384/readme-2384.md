Chat with local LLMs using n8n and Ollama

https://n8nworkflows.xyz/workflows/chat-with-local-llms-using-n8n-and-ollama-2384


# Chat with local LLMs using n8n and Ollama

### 1. Workflow Overview

This workflow enables real-time chat interactions with locally hosted Large Language Models (LLMs) using n8n integrated with Ollama, a tool for managing local LLMs. It is designed to facilitate private, cost-effective, and experimental AI conversations by routing user chat inputs to an Ollama server and returning AI-generated responses within n8n.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the user’s chat messages via a chat trigger node.
- **1.2 AI Processing (Chat LLM Chain):** Forwards the user input to the Ollama chat model and retrieves the AI-generated response.
- **1.3 Response Delivery:** Returns the AI response back to the chat interface (implicitly handled by the LangChain chat trigger node).

Additionally, there are sticky note nodes providing documentation and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from the user interface and initiates the workflow upon message reception.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  

  - **When chat message received**  
    - **Type:** `@n8n/n8n-nodes-langchain.chatTrigger`  
    - **Technical Role:** Acts as a webhook trigger specifically designed to start workflows upon receiving chat inputs in LangChain-enabled chat interfaces.  
    - **Configuration:** Default parameters with no additional options set; listens for chat messages without filtering.  
    - **Key Expressions/Variables:** None explicitly configured; standard input capture from chat.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Feeds into the “Chat LLM Chain” node.  
    - **Version Requirements:** Version 1.1 of the node type; ensure n8n supports this or higher for compatibility.  
    - **Potential Failure Points:**  
      - Webhook availability issues (e.g., port conflicts, network restrictions).  
      - Malformed or missing chat message payloads.  
      - If chat interface changes, trigger may fail to capture input correctly.

---

#### 2.2 AI Processing (Chat LLM Chain)

- **Overview:**  
  This block processes the chat input by sending it to the Ollama local LLM server via the Ollama Chat Model node within a LangChain chain, then collects the AI’s response.

- **Nodes Involved:**  
  - Chat LLM Chain  
  - Ollama Chat Model

- **Node Details:**  

  - **Chat LLM Chain**  
    - **Type:** `@n8n/n8n-nodes-langchain.chainLlm`  
    - **Technical Role:** Manages the LLM interaction chain; orchestrates sending input to the language model node and retrieving responses.  
    - **Configuration:** No additional parameters set; acts as a connector between chat trigger and LLM node.  
    - **Input Connections:** Receives input from “When chat message received” node.  
    - **Output Connections:** Sends output to the chat interface implicitly via LangChain trigger node (not explicitly shown).  
    - **Version Requirements:** Version 1.4.  
    - **Edge Cases:**  
      - Failure in chaining if the LLM node is unreachable or misconfigured.  
      - Timeouts or slow responses from Ollama server.  
      - Errors if output from Ollama is malformed or empty.

  - **Ollama Chat Model**  
    - **Type:** `@n8n/n8n-nodes-langchain.lmChatOllama`  
    - **Technical Role:** Connects to the local Ollama API to send prompts and receive AI-generated completions.  
    - **Configuration:** Uses credentials named “Local Ollama” linked to an Ollama API instance (default address: `http://localhost:11434`). No additional options set.  
    - **Input Connections:** Invoked by “Chat LLM Chain” node through the `ai_languageModel` connection.  
    - **Output Connections:** Returns AI responses to “Chat LLM Chain”.  
    - **Credentials:** Requires OAuth2 or API key credentials configured for local Ollama access.  
    - **Version Requirements:** Version 1.  
    - **Edge Cases:**  
      - Connection failure if Ollama server is not running or inaccessible.  
      - Authentication errors if credentials are invalid or missing.  
      - Network timeouts or server errors.  
      - Model response errors or empty replies.

---

#### 2.3 Response Delivery

- **Overview:**  
  The workflow implicitly returns the AI-generated response to the user chat interface by utilizing the LangChain chat trigger’s built-in response sending mechanism.

- **Nodes Involved:**  
  - None explicitly; the “When chat message received” node manages response delivery.

- **Node Details:**  
  - Response is automatically sent back through the webhook established by the chat trigger node once the AI response is received via the “Chat LLM Chain”.

- **Edge Cases:**  
  - Failures in returning the response if the original webhook context is lost or times out.  
  - If the AI response is empty or malformed, user receives no or invalid output.

---

#### 2.4 Documentation and Setup Notes

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  

  - **Sticky Note**  
    - Contains overview, purpose, usage scenarios, and setup instructions for the workflow.  
    - Positioned at workflow start for user reference.  

  - **Sticky Note1**  
    - Specific instructions regarding Ollama setup, including default localhost API address and Docker networking considerations.  
    - Important for ensuring connectivity between n8n and Ollama, especially in containerized environments.

---

### 3. Summary Table

| Node Name               | Node Type                                 | Functional Role            | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                      |
|-------------------------|-------------------------------------------|----------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger     | Input Reception (Trigger)   | —                           | Chat LLM Chain              | See Sticky Note: Workflow overview, usage, and setup steps.                                                                     |
| Chat LLM Chain           | @n8n/n8n-nodes-langchain.chainLlm         | AI Processing Chain         | When chat message received  | (Implicit response delivery) | See Sticky Note: Workflow overview, usage, and setup steps.                                                                     |
| Ollama Chat Model        | @n8n/n8n-nodes-langchain.lmChatOllama     | Connect to local Ollama LLM | Chat LLM Chain (ai_languageModel) | Chat LLM Chain              | See Sticky Note1: Ollama setup details including localhost address and Docker networking notes.                                 |
| Sticky Note              | n8n-nodes-base.stickyNote                  | Documentation               | —                           | —                           | Contains detailed workflow overview and usage instructions.                                                                      |
| Sticky Note1             | n8n-nodes-base.stickyNote                  | Documentation               | —                           | —                           | Contains Ollama setup instructions and Docker container networking advice.                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a `@n8n/n8n-nodes-langchain.chatTrigger` node.  
   - Name it **When chat message received**.  
   - Use default settings; no extra parameters needed.  
   - This node will act as the webhook listener for incoming chat inputs.

2. **Create the LLM Chain Node:**  
   - Add a `@n8n/n8n-nodes-langchain.chainLlm` node.  
   - Name it **Chat LLM Chain**.  
   - Leave parameters empty (default).  
   - Connect the output of **When chat message received** node to the **Chat LLM Chain** node input.

3. **Create the Ollama Chat Model Node:**  
   - Add a `@n8n/n8n-nodes-langchain.lmChatOllama` node.  
   - Name it **Ollama Chat Model**.  
   - Under credentials, configure and select credentials for “Local Ollama”:  
     - This should point to your local Ollama API endpoint, typically `http://localhost:11434`.  
     - Ensure Ollama is installed and running on your machine before executing the workflow.  
   - Leave options empty (default).  
   - Connect the **ai_languageModel** output from **Ollama Chat Model** node to the input of **Chat LLM Chain** node (specifically, the `ai_languageModel` connection).

4. **Connect Nodes:**  
   - Connect **When chat message received** → **Chat LLM Chain** (main output).  
   - Connect **Ollama Chat Model** → **Chat LLM Chain** (`ai_languageModel` output to input).

5. **Add Documentation Nodes:**  
   - Add a `n8n-nodes-base.stickyNote` node.  
   - Paste the workflow overview, usage scenarios, and setup instructions.  
   - Add a second sticky note with Ollama-specific setup guidance including Docker networking tips.

6. **Configure Credentials:**  
   - Create and configure a credential named “Local Ollama” in n8n.  
   - This should authenticate and authorize requests to your local Ollama instance. Usually, no authentication or a local API key depending on your Ollama setup.

7. **Save and Activate:**  
   - Save the workflow.  
   - Activate the workflow to start listening for chat messages and processing them through the local Ollama model.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Ensure Ollama is installed and running on your local machine before running the workflow.                            | https://ollama.com (official Ollama documentation and installation guides)                       |
| If running n8n in Docker, use `--net=host` to allow network access to the Ollama API server on localhost.            | Docker networking considerations for n8n and Ollama interoperability                            |
| Workflow enables private, cost-effective AI chat interactions without cloud API usage, ideal for confidential data. | Workflow design rationale and use case summary                                                  |
| This workflow leverages n8n LangChain integration nodes requiring n8n versions supporting LangChain nodes v1.1+.     | n8n version compatibility requirement                                                           |

---

This structured document provides a detailed, node-level technical reference enabling both humans and automation agents to understand, reproduce, and maintain the "Chat with local LLMs using n8n and Ollama" workflow effectively.