🤖🧠 AI Agent Chatbot + LONG TERM Memory + Note Storage + Telegram

https://n8nworkflows.xyz/workflows/-----ai-agent-chatbot---long-term-memory---note-storage---telegram-2872


# 🤖🧠 AI Agent Chatbot + LONG TERM Memory + Note Storage + Telegram

### 1. Workflow Overview

This workflow implements an AI-powered chatbot with advanced long-term memory and note storage capabilities, integrated with Telegram for real-time communication and Google Docs for persistent data storage. It is designed to maintain conversational context, intelligently manage user memories and notes, and provide personalized, context-aware responses.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages via a LangChain chat trigger node.
- **1.2 Memory Retrieval:** Fetches long-term memories and notes from Google Docs to provide context.
- **1.3 Memory Aggregation and Merging:** Combines retrieved memories and notes into a unified dataset.
- **1.4 AI Agent Processing:** Uses a custom AI agent with integrated tools and memory buffers to generate responses, manage memory, and handle notes.
- **1.5 Memory and Note Storage:** Saves updated long-term memories and notes back to Google Docs.
- **1.6 Response Delivery:** Sends the AI-generated response back to the user via Telegram and sets output for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from users, triggering the workflow to start processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point for chat messages, triggers workflow on new user input.  
    - Configuration: Default options enabled, webhook ID assigned for external integration.  
    - Inputs: External chat messages (e.g., from Telegram or other LangChain-compatible sources).  
    - Outputs: Passes message data downstream for memory retrieval.  
    - Edge Cases: Failure if webhook is unreachable or message format is invalid.  
    - Version: 1.1

---

#### 2.2 Memory Retrieval

- **Overview:**  
  Retrieves stored long-term memories and notes from Google Docs to provide context for the AI agent.

- **Nodes Involved:**  
  - Retrieve Long Term Memories  
  - Retrieve Notes  
  - Merge  
  - Aggregate  
  - Sticky Note7 (comment)  
  - Sticky Note11 (comment)

- **Node Details:**  
  - **Retrieve Long Term Memories**  
    - Type: `n8n-nodes-base.googleDocs`  
    - Role: Fetches the document containing long-term memories.  
    - Configuration: Operation set to "get", document URL specified (Google Doc ID placeholder).  
    - Credentials: Google Docs OAuth2 credentials required.  
    - Outputs: Memory data JSON.  
    - Edge Cases: Document access errors, permission issues, or empty documents.  
    - Version: 2  
    - Sticky Note7: "Retrieve Long Term Memories\nGoogle Docs"  

  - **Retrieve Notes**  
    - Type: `n8n-nodes-base.googleDocs`  
    - Role: Fetches the document containing user notes.  
    - Configuration: Operation "get", document URL specified (Google Doc ID placeholder).  
    - Credentials: Google Docs OAuth2 credentials required.  
    - On Error: Continues workflow even if retrieval fails.  
    - Outputs: Notes data JSON.  
    - Edge Cases: Similar to Retrieve Long Term Memories.  
    - Version: 2  
    - Sticky Note11: "Retrieve Notes\nGoogle Docs"  

  - **Merge**  
    - Type: `n8n-nodes-base.merge`  
    - Role: Combines outputs from memory and notes retrieval nodes into a single stream.  
    - Inputs: Two inputs from Retrieve Long Term Memories and Retrieve Notes.  
    - Outputs: Merged data for aggregation.  
    - Version: 3  

  - **Aggregate**  
    - Type: `n8n-nodes-base.aggregate`  
    - Role: Aggregates all merged item data into a single dataset for AI agent consumption.  
    - Configuration: Aggregate all item data.  
    - Inputs: Output from Merge node.  
    - Outputs: Aggregated memory and note data.  
    - Version: 1  

---

#### 2.3 AI Agent Processing

- **Overview:**  
  Processes the user input along with retrieved memories and notes using a custom AI agent that manages memory, notes, and generates responses.

- **Nodes Involved:**  
  - AI Tools Agent  
  - Window Buffer Memory  
  - gpt-4o-mini  
  - DeepSeek-V3 Chat  
  - Sticky Note (LLM)  
  - Sticky Note2 (AI Agent description)  
  - Sticky Note (AI Tools Agent instructions)

- **Node Details:**  
  - **AI Tools Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Core AI agent node that processes input, manages memory, notes, and generates responses.  
    - Configuration:  
      - Text input from the original chat message.  
      - System message defines role, rules, tools (Save Memory, Save Note), memory and note usage guidelines, fallback responses, and privacy rules.  
      - Integrates with AI language models and memory nodes via ai_languageModel and ai_memory connections.  
    - Inputs: Aggregated memory and notes, current chat input, window buffer memory, language models.  
    - Outputs: AI-generated response and tool outputs (memory/note data).  
    - Edge Cases: Expression errors in system message, API failures, memory inconsistencies.  
    - Version: 1.7  

  - **Window Buffer Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains short-term conversational context with a sliding window of recent messages.  
    - Configuration:  
      - Session key derived from the chat session ID.  
      - Context window length set to 50 messages.  
    - Inputs: Session ID from chat trigger.  
    - Outputs: Contextual memory for AI Tools Agent.  
    - Edge Cases: Session ID missing or malformed.  
    - Version: 1.3  

  - **gpt-4o-mini**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Primary language model for general AI interactions.  
    - Configuration: Default GPT-4o-mini model with OpenAI API credentials.  
    - Inputs: Connected as ai_languageModel to AI Tools Agent.  
    - Outputs: Language model responses.  
    - Edge Cases: API rate limits, authentication errors.  
    - Version: 1.1  

  - **DeepSeek-V3 Chat**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Specialized AI model for enhanced chat processing.  
    - Configuration: Model set to "deepseek-chat" with separate OpenAI credentials.  
    - Inputs: Not connected downstream (standalone or reserved for future use).  
    - Outputs: None connected.  
    - Edge Cases: API errors, unused node may cause confusion.  
    - Version: 1.1  

---

#### 2.4 Memory and Note Storage

- **Overview:**  
  Saves updated long-term memories and notes back to Google Docs based on AI agent outputs.

- **Nodes Involved:**  
  - Save Long Term Memories  
  - Save Notes  
  - Sticky Note9 (Save Long Term Memories comment)  
  - Sticky Note10 (Save Notes comment)  
  - Sticky Note8 (Save To Current Chat Memory comment)

- **Node Details:**  
  - **Save Long Term Memories**  
    - Type: `n8n-nodes-base.googleDocsTool`  
    - Role: Inserts new memory entries into the long-term memory Google Doc.  
    - Configuration:  
      - Action inserts JSON with memory content from AI output and current timestamp.  
      - Operation set to "update".  
      - Document URL specified (Google Doc ID placeholder).  
    - Credentials: Google Docs OAuth2 credentials.  
    - Inputs: AI Tools Agent tool output (memory).  
    - Edge Cases: Document update failures, malformed AI output.  
    - Version: 2  
    - Sticky Note9: "Save Long Term Memories\nGoogle Docs"  

  - **Save Notes**  
    - Type: `n8n-nodes-base.googleDocsTool`  
    - Role: Inserts new note entries into the notes Google Doc.  
    - Configuration:  
      - Action inserts JSON with note content from AI output and current timestamp.  
      - Operation set to "update".  
      - Document URL specified (Google Doc ID placeholder).  
    - Credentials: Google Docs OAuth2 credentials.  
    - Inputs: AI Tools Agent tool output (note).  
    - Edge Cases: Similar to Save Long Term Memories.  
    - Version: 2  
    - Sticky Note10: "Save Notes\nGoogle Docs"  

---

#### 2.5 Response Delivery

- **Overview:**  
  Sends the AI-generated response back to the user via Telegram and prepares output data for other uses.

- **Nodes Involved:**  
  - Telegram Response  
  - Chat Response  
  - Sticky Note1 (Telegram optional comment)

- **Node Details:**  
  - **Telegram Response**  
    - Type: `n8n-nodes-base.telegram`  
    - Role: Sends the AI-generated text response to the user's Telegram chat.  
    - Configuration:  
      - Text set to AI output.  
      - Chat ID hardcoded as "1234567891" (should be dynamic in production).  
      - Parse mode set to HTML, attribution disabled.  
    - Credentials: Telegram API OAuth2 credentials.  
    - Inputs: AI Tools Agent main output.  
    - Edge Cases: Invalid chat ID, Telegram API errors, message formatting issues.  
    - Version: 1.2  
    - Sticky Note1: "Telegram \n(Optional)"  

  - **Chat Response**  
    - Type: `n8n-nodes-base.set`  
    - Role: Sets the final output string from AI response for downstream use or logging.  
    - Configuration: Assigns `output` field from AI output JSON.  
    - Inputs: AI Tools Agent main output.  
    - Outputs: Final output data.  
    - Version: 3.4  

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                              | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                   |
|----------------------------|--------------------------------------|----------------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point for chat messages                 | -                                | Retrieve Long Term Memories, Retrieve Notes |                                                                                              |
| Retrieve Long Term Memories | n8n-nodes-base.googleDocs             | Fetch long-term memories from Google Docs    | When chat message received        | Merge                            | Retrieve Long Term Memories<br>Google Docs                                                   |
| Retrieve Notes             | n8n-nodes-base.googleDocs             | Fetch user notes from Google Docs             | When chat message received        | Merge                            | Retrieve Notes<br>Google Docs                                                                |
| Merge                      | n8n-nodes-base.merge                  | Combine memories and notes streams            | Retrieve Long Term Memories, Retrieve Notes | Aggregate                       |                                                                                              |
| Aggregate                  | n8n-nodes-base.aggregate              | Aggregate merged data into single dataset     | Merge                            | AI Tools Agent                   |                                                                                              |
| AI Tools Agent             | @n8n/n8n-nodes-langchain.agent        | Core AI agent processing input and memory     | Aggregate, Window Buffer Memory, gpt-4o-mini | Telegram Response, Chat Response, Save Long Term Memories, Save Notes |                                                                                              |
| Window Buffer Memory       | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintain short-term conversational context   | When chat message received        | AI Tools Agent                   |                                                                                              |
| gpt-4o-mini                | @n8n/n8n-nodes-langchain.lmChatOpenAi | Primary language model for AI responses       | AI Tools Agent (ai_languageModel) | AI Tools Agent                   |                                                                                              |
| DeepSeek-V3 Chat           | @n8n/n8n-nodes-langchain.lmChatOpenAi | Specialized chat model (unused output)        | -                                | -                               |                                                                                              |
| Save Long Term Memories    | n8n-nodes-base.googleDocsTool         | Save updated memories to Google Docs          | AI Tools Agent (ai_tool)          | AI Tools Agent                   | Save Long Term Memories<br>Google Docs                                                      |
| Save Notes                 | n8n-nodes-base.googleDocsTool         | Save updated notes to Google Docs              | AI Tools Agent (ai_tool)          | AI Tools Agent                   | Save Notes<br>Google Docs                                                                   |
| Telegram Response          | n8n-nodes-base.telegram                | Send AI response back to Telegram user        | AI Tools Agent                   | -                               | Telegram<br>(Optional)                                                                       |
| Chat Response              | n8n-nodes-base.set                    | Set final output string from AI response      | AI Tools Agent                   | -                               |                                                                                              |
| Sticky Note7               | n8n-nodes-base.stickyNote             | Comment: Retrieve Long Term Memories           | -                                | -                               | Retrieve Long Term Memories<br>Google Docs                                                  |
| Sticky Note8               | n8n-nodes-base.stickyNote             | Comment: Save To Current Chat Memory (Optional) | -                                | -                               | Save To Current Chat Memory (Optional)                                                      |
| Sticky Note9               | n8n-nodes-base.stickyNote             | Comment: Save Long Term Memories               | -                                | -                               | Save Long Term Memories<br>Google Docs                                                     |
| Sticky Note10              | n8n-nodes-base.stickyNote             | Comment: Save Notes                             | -                                | -                               | Save Notes<br>Google Docs                                                                  |
| Sticky Note11              | n8n-nodes-base.stickyNote             | Comment: Retrieve Notes                         | -                                | -                               | Retrieve Notes<br>Google Docs                                                              |
| Sticky Note1               | n8n-nodes-base.stickyNote             | Comment: Telegram (Optional)                    | -                                | -                               | Telegram (Optional)                                                                        |
| Sticky Note                | n8n-nodes-base.stickyNote             | Comment: LLM                                    | -                                | -                               | LLM                                                                                        |
| Sticky Note2               | n8n-nodes-base.stickyNote             | Comment: AI AGENT with Long Term Memory & Note Storage | -                                | -                               | AI AGENT with Long Term Memory & Note Storage                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a `@n8n/n8n-nodes-langchain.chatTrigger` node named "When chat message received".  
   - Configure webhook ID (auto-generated) to receive chat messages.  
   - Leave options as default.

2. **Set Up Google Docs Credentials**  
   - Configure Google OAuth2 credentials in n8n with appropriate scopes for reading and updating Google Docs.

3. **Add Google Docs Nodes for Memory Retrieval**  
   - Add a `googleDocs` node named "Retrieve Long Term Memories".  
   - Set operation to "get".  
   - Provide the Google Doc URL or ID for long-term memories.  
   - Connect input from "When chat message received".  
   - Enable "Always Output Data" to ensure output even if empty.

4. **Add Google Docs Node for Notes Retrieval**  
   - Add another `googleDocs` node named "Retrieve Notes".  
   - Set operation to "get".  
   - Provide the Google Doc URL or ID for notes.  
   - Connect input from "When chat message received".  
   - Set "On Error" to "continueRegularOutput" to avoid workflow failure on errors.  
   - Enable "Always Output Data".

5. **Merge Retrieved Data**  
   - Add a `merge` node named "Merge".  
   - Connect "Retrieve Long Term Memories" to input 1 and "Retrieve Notes" to input 2.  
   - Use default merge settings.

6. **Aggregate Merged Data**  
   - Add an `aggregate` node named "Aggregate".  
   - Set aggregation to "aggregateAllItemData".  
   - Connect input from "Merge".

7. **Add Window Buffer Memory Node**  
   - Add `@n8n/n8n-nodes-langchain.memoryBufferWindow` node named "Window Buffer Memory".  
   - Set `sessionKey` to `={{ $('When chat message received').item.json.sessionId }}`.  
   - Set `sessionIdType` to "customKey".  
   - Set `contextWindowLength` to 50.

8. **Add Primary Language Model Node**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node named "gpt-4o-mini".  
   - Use OpenAI API credentials configured in n8n.  
   - Use default GPT-4o-mini model settings.

9. **Add Secondary Language Model Node (Optional)**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node named "DeepSeek-V3 Chat".  
   - Set model to "deepseek-chat".  
   - Use separate OpenAI credentials if applicable.

10. **Add AI Agent Node**  
    - Add `@n8n/n8n-nodes-langchain.agent` node named "AI Tools Agent".  
    - Set `text` parameter to `={{ $('When chat message received').item.json.chatInput }}`.  
    - Paste the detailed system message defining role, rules, tools, memories, notes, and fallback responses.  
    - Connect inputs:  
      - `ai_languageModel` from "gpt-4o-mini".  
      - `ai_memory` from "Window Buffer Memory".  
      - `main` from "Aggregate".  
    - Connect outputs to "Telegram Response", "Chat Response", "Save Long Term Memories", and "Save Notes".

11. **Add Google Docs Tool Node to Save Memories**  
    - Add `googleDocsTool` node named "Save Long Term Memories".  
    - Set operation to "update".  
    - Provide Google Doc URL for memories.  
    - Configure action to insert JSON with keys:  
      ```json
      {
        "memory": "{{ $fromAI('memory') }}",
        "date": "{{ $now }}"
      }
      ```  
    - Connect input from AI Tools Agent's `ai_tool` output.

12. **Add Google Docs Tool Node to Save Notes**  
    - Add `googleDocsTool` node named "Save Notes".  
    - Set operation to "update".  
    - Provide Google Doc URL for notes.  
    - Configure action to insert JSON with keys:  
      ```json
      {
        "note": "{{ $fromAI('memory') }}",
        "date": "{{ $now }}"
      }
      ```  
    - Connect input from AI Tools Agent's `ai_tool` output.

13. **Add Telegram Node for Response Delivery**  
    - Add `telegram` node named "Telegram Response".  
    - Set `text` to `={{ $json.output }}`.  
    - Set `chatId` to the dynamic chat ID from the incoming message (replace hardcoded "1234567891" with dynamic expression).  
    - Set parse mode to HTML and disable attribution.  
    - Connect input from AI Tools Agent main output.

14. **Add Set Node for Final Output**  
    - Add `set` node named "Chat Response".  
    - Assign field `output` with value `={{ $json.output }}` from AI Tools Agent.  
    - Connect input from AI Tools Agent main output.

15. **Add Sticky Notes (Optional)**  
    - Add sticky notes to document the purpose of each block and node as per the original workflow for clarity.

16. **Test and Validate**  
    - Test the workflow end-to-end with sample Telegram messages.  
    - Verify memory and notes are correctly retrieved and saved in Google Docs.  
    - Confirm AI responses are sent back to Telegram.  
    - Handle errors such as API failures, invalid credentials, or missing documents.

---

This completes the detailed analysis and reconstruction guide for the "🤖🧠 AI Agent Chatbot + LONG TERM Memory + Note Storage + Telegram" workflow.