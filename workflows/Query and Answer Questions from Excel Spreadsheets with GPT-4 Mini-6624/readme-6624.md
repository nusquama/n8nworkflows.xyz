Query and Answer Questions from Excel Spreadsheets with GPT-4 Mini

https://n8nworkflows.xyz/workflows/query-and-answer-questions-from-excel-spreadsheets-with-gpt-4-mini-6624


# Query and Answer Questions from Excel Spreadsheets with GPT-4 Mini

### 1. Workflow Overview

This n8n workflow enables users to query Excel spreadsheets interactively using GPT-4 Mini through a chat interface. It accepts chat inputs, processes them with AI agents that include memory and language models, and fetches relevant data from Excel files to answer questions intelligently. The workflow is designed for scenarios such as customer support, data analysis, or knowledge retrieval where users need to extract insights from spreadsheets conversationally.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Captures user chat input via a webhook-based chat trigger.
- **1.2 AI Processing:** Uses a smart AI agent that integrates a language model, memory of the chat history, and AI tool capabilities.
- **1.3 Data Retrieval:** Retrieves data dynamically from Microsoft Excel spreadsheets to support AI responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by capturing user messages or questions through a chat interface. It serves as the entry point for the conversational interaction.

- **Nodes Involved:**  
  - Start Chat Conversation

- **Node Details:**  
  - **Start Chat Conversation**  
    - *Type:* `chatTrigger` (Langchain Chat Trigger)  
    - *Role:* Listens for incoming chat messages and triggers the workflow.  
    - *Configuration:* Configured with a webhook ID to receive chat input events. No additional parameters specified.  
    - *Input:* External chat messages (via webhook).  
    - *Output:* Passes chat input to the Smart AI Agent node.  
    - *Version:* 1.1  
    - *Potential Failures:* Webhook connectivity issues, malformed input, or missing chat context.

#### 2.2 AI Processing

- **Overview:**  
  This block processes the user query using a smart AI agent that incorporates an OpenAI GPT-4 Mini language model and maintains conversation context via memory. It also connects to an Excel data retrieval tool as an AI tool for data-driven responses.

- **Nodes Involved:**  
  - Smart AI Agent  
  - OpenAI Chat Model  
  - Remember Chat History

- **Node Details:**  
  - **Smart AI Agent**  
    - *Type:* `agent` (Langchain Agent Node)  
    - *Role:* Orchestrates the AI response by integrating language model output, chat memory, and external AI tools (Excel).  
    - *Configuration:* No additional parameters shown; uses linked AI tool, memory, and language model.  
    - *Input:* Takes chat input from Start Chat Conversation.  
    - *Output:* Generates final AI answers sent back into the conversation flow.  
    - *Version:* 1.7  
    - *Edge Cases:* AI model timeout, tool invocation errors, memory overflow, or unsupported queries.  
    - *Sub-workflow:* Not specified.

  - **OpenAI Chat Model**  
    - *Type:* `lmChatOpenAi` (Langchain OpenAI Chat Model)  
    - *Role:* Provides GPT-4 Mini language model capabilities for generating natural language responses.  
    - *Configuration:* Uses default model settings; credential configuration (OpenAI API key) required externally.  
    - *Input:* Receives prompt context from Smart AI Agent.  
    - *Output:* Returns generated text to Smart AI Agent.  
    - *Version:* 1.2  
    - *Potential Failures:* API authentication errors, rate limits, response timeouts.

  - **Remember Chat History**  
    - *Type:* `memoryBufferWindow` (Langchain Memory Buffer)  
    - *Role:* Stores and manages recent chat messages to maintain conversation context for the AI.  
    - *Configuration:* Default buffer window size assumed; no parameters shown.  
    - *Input:* Chat messages from Start Chat Conversation or Smart AI Agent.  
    - *Output:* Provides memory context to Smart AI Agent.  
    - *Version:* 1.3  
    - *Edge Cases:* Memory size limits, loss of context on restart.

#### 2.3 Data Retrieval

- **Overview:**  
  This block integrates Microsoft Excel as an AI tool allowing the AI agent to query spreadsheet data dynamically during conversations.

- **Nodes Involved:**  
  - Get Excel Data

- **Node Details:**  
  - **Get Excel Data**  
    - *Type:* `microsoftExcelTool` (Microsoft Excel Tool Node)  
    - *Role:* Provides access to Excel spreadsheet data as an AI tool for the Smart AI Agent.  
    - *Configuration:* No specific parameters shown; assumes configured credentials for Microsoft Excel API.  
    - *Input:* Invoked by Smart AI Agent as an AI tool during processing.  
    - *Output:* Returns spreadsheet data to the Smart AI Agent to inform AI responses.  
    - *Version:* 2.1  
    - *Edge Cases:* Authentication failures, file access permission issues, unsupported Excel formats, API rate limits.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                         | Input Node(s)           | Output Node(s)         | Sticky Note |
|-----------------------|----------------------------------|---------------------------------------|------------------------|------------------------|-------------|
| Start Chat Conversation | Langchain Chat Trigger (chatTrigger) | Entry point for user chat input       | -                      | Smart AI Agent          |             |
| Smart AI Agent         | Langchain Agent (agent)           | Processes queries using AI and tools  | Start Chat Conversation | -                      |             |
| OpenAI Chat Model      | Langchain OpenAI Chat Model (lmChatOpenAi) | Provides GPT-4 Mini model responses   | Smart AI Agent (ai_languageModel) | Smart AI Agent          |             |
| Remember Chat History  | Langchain Memory Buffer (memoryBufferWindow) | Maintains chat history context        | -                      | Smart AI Agent          |             |
| Get Excel Data         | Microsoft Excel Tool (microsoftExcelTool) | Retrieves Excel spreadsheet data      | -                      | Smart AI Agent (ai_tool)|             |
| Sticky Note1           | Sticky Note                      | -                                     | -                      | -                      |             |
| Sticky Note2           | Sticky Note                      | -                                     | -                      | -                      |             |
| Sticky Note3           | Sticky Note                      | -                                     | -                      | -                      |             |
| Sticky Note            | Sticky Note                      | -                                     | -                      | -                      |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Start Chat Conversation node:**  
   - Type: Langchain Chat Trigger (`chatTrigger`)  
   - Configure webhook with a unique ID (or generate automatically).  
   - No additional parameters needed.

2. **Create the OpenAI Chat Model node:**  
   - Type: Langchain OpenAI Chat Model (`lmChatOpenAi`)  
   - Configure with your OpenAI API credentials.  
   - Use default model settings for GPT-4 Mini or specify if needed.

3. **Create the Remember Chat History node:**  
   - Type: Langchain Memory Buffer (`memoryBufferWindow`)  
   - Use default buffer size or configure window length to store recent messages.

4. **Create the Get Excel Data node:**  
   - Type: Microsoft Excel Tool (`microsoftExcelTool`)  
   - Configure with Microsoft Excel API credentials (OAuth2 recommended).  
   - Specify Excel file and sheet access as per your requirements.

5. **Create the Smart AI Agent node:**  
   - Type: Langchain Agent (`agent`)  
   - Set up inputs to accept:  
     - AI Language Model input from the OpenAI Chat Model node  
     - AI Memory input from the Remember Chat History node  
     - AI Tool input from the Get Excel Data node  
   - Connect the main trigger input from the Start Chat Conversation node.

6. **Connect nodes:**  
   - Connect Start Chat Conversation → Smart AI Agent (main input).  
   - Connect OpenAI Chat Model → Smart AI Agent (ai_languageModel input).  
   - Connect Remember Chat History → Smart AI Agent (ai_memory input).  
   - Connect Get Excel Data → Smart AI Agent (ai_tool input).

7. **Verify credentials:**  
   - Ensure OpenAI API key is set up in n8n credentials.  
   - Ensure Microsoft Excel OAuth2 credentials are configured and authorized.

8. **Test workflow:**  
   - Start the webhook and send sample chat messages.  
   - Confirm Excel data is queried correctly and responses are generated.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow uses Langchain nodes to combine AI language models, memory, and external tools.   | Langchain integration documentation: https://docs.n8n.io/integrations/langchain/                   |
| Microsoft Excel Tool node requires OAuth2 credentials and appropriate file permissions.         | Microsoft Graph API documentation: https://docs.microsoft.com/en-us/graph/api/resources/excel     |
| OpenAI Chat Model node requires valid OpenAI API key with GPT-4 Mini access.                    | OpenAI API docs: https://platform.openai.com/docs/api-reference/chat/create                        |
| Workflow designed for conversational querying of structured data in Excel spreadsheets.         | Useful for support desks, data analysis, and knowledge base querying.                              |
| No sticky notes with content were present in the workflow JSON; adjust accordingly if added.    |                                                                                                   |

---

*Disclaimer:* The provided text is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.