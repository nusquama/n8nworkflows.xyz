Create a Notion AI Assistant with Google Gemini for Managing Tasks & Content

https://n8nworkflows.xyz/workflows/create-a-notion-ai-assistant-with-google-gemini-for-managing-tasks---content-4857


# Create a Notion AI Assistant with Google Gemini for Managing Tasks & Content

---

## 1. Workflow Overview

This workflow implements an AI-powered assistant integrated with Notion for managing tasks and content via chat interactions. It is designed primarily for users who want to interact conversationally with their Notion workspace to retrieve information, create tasks, add content, or manage databases, leveraging Google Gemini AI for natural language understanding and generation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures chat messages from users to trigger the assistant.
- **1.2 AI Memory Management:** Maintains conversational context to provide coherent interactions.
- **1.3 AI Processing & Task Planning:** Uses Google Gemini and a custom AI agent to interpret user input, plan actions, and interact with Notion.
- **1.4 Notion Integration:** Lists available Notion tools and executes selected tools to perform operations in the Notion workspace.
- **1.5 Setup and Documentation:** Contains sticky notes providing setup instructions, disclaimers, and usage tips.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block receives chat messages from users and triggers the workflow.

**Nodes Involved:**  
- When chat message received

**Node Details:**  

- **When chat message received**  
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chatTrigger` — Listens for incoming chat messages via webhook.  
  - *Configuration:* File uploads are disabled (`allowFileUploads: false`).  
  - *Expressions/Variables:* None used; triggers on any received chat message.  
  - *Input:* Incoming HTTP webhook (chat message).  
  - *Output:* Sends message data to the AI Task Planner node.  
  - *Version:* 1.1  
  - *Potential Failures:* Webhook misconfiguration, network issues, malformed incoming data.  
  - *Sub-workflow:* None.

---

### 2.2 AI Memory Management

**Overview:**  
Maintains a sliding window memory buffer to preserve conversational context for the AI agent, enabling coherent multi-turn dialogue.

**Nodes Involved:**  
- Simple Memory

**Node Details:**  

- **Simple Memory**  
  - *Type & Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Implements a windowed conversational memory buffer.  
  - *Configuration:* Default settings with no specific parameters customized, meaning it keeps a limited recent conversation history for context.  
  - *Expressions/Variables:* None explicitly configured.  
  - *Input:* Receives memory updates from the AI Task Planner node (via the `ai_memory` connection).  
  - *Output:* Passes memory context back to the AI Task Planner for use during processing.  
  - *Version:* 1.3  
  - *Potential Failures:* Memory overflow if too large, loss of context if buffer windows too small, expression or data format errors.  
  - *Sub-workflow:* None.

---

### 2.3 AI Processing & Task Planning

**Overview:**  
This block handles interpreting user inputs using Google Gemini AI, planning tasks, and deciding which Notion tools to invoke.

**Nodes Involved:**  
- AI Task Planner  
- Google Gemini Chat Model

**Node Details:**  

- **Google Gemini Chat Model**  
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — Chat language model node using Google Gemini (PaLM) for natural language understanding and generation.  
  - *Configuration:* Uses model `"models/gemini-2.5-pro-preview-06-05"`; no additional options specified.  
  - *Expressions/Variables:* None beyond fixed model name.  
  - *Input:* Receives prompt and context from AI Task Planner node (via `ai_languageModel`).  
  - *Output:* Returns AI-generated chat responses back to AI Task Planner.  
  - *Version:* 1  
  - *Credentials:* Requires Google Palm API credentials to authenticate with Google Gemini.  
  - *Potential Failures:* API authentication errors, rate limits, network timeouts, invalid model name.  
  - *Sub-workflow:* None.

- **AI Task Planner**  
  - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` — Custom AI agent that orchestrates conversation, decides actions, and interfaces with Notion tools.  
  - *Configuration:*  
    - System message instructs the agent to act as a helpful assistant with access to the Notion workspace.  
    - Provides the task database ID and instructs the agent on how to retrieve database properties and avoid redundant questions.  
  - *Expressions/Variables:* System message is a multiline string with embedded variables (e.g., task database ID).  
  - *Input:* Receives chat input from "When chat message received" and memory context from "Simple Memory".  
  - *Output:* Sends AI decisions to "Notion - list tools", "Notion - execute tool", and memory buffer nodes.  
  - *Version:* 1.7  
  - *Potential Failures:* Expression evaluation errors, AI response parsing issues, credential/auth errors with Notion tools, logic errors in agent reasoning.  
  - *Sub-workflow:* None.

---

### 2.4 Notion Integration

**Overview:**  
This block interfaces with the Notion MCP server to list available Notion tools and execute AI-selected tools to manipulate the Notion workspace.

**Nodes Involved:**  
- Notion - list tools  
- Notion - execute tool

**Node Details:**  

- **Notion - list tools**  
  - *Type & Role:* `n8n-nodes-mcp.mcpClientTool` — Queries the Notion MCP server to retrieve the list of available tools (e.g., add content, create database).  
  - *Configuration:* No parameters; simply requests tool list.  
  - *Expressions/Variables:* None.  
  - *Input:* Called by AI Task Planner via `ai_tool` connection.  
  - *Output:* Returns tools list back to AI Task Planner for decision-making.  
  - *Version:* 1  
  - *Credentials:* Requires MCP Notion official credentials.  
  - *Potential Failures:* Authentication errors, connectivity issues with MCP server, malformed response.  
  - *Sub-workflow:* None.

- **Notion - execute tool**  
  - *Type & Role:* `n8n-nodes-mcp.mcpClientTool` — Executes a selected Notion tool with parameters specified by the AI agent.  
  - *Configuration:*  
    - `toolName` and `toolParameters` are dynamically set based on AI outputs using expressions:  
      - `toolName` extracted from AI response field `"tool"`.  
      - `toolParameters` extracted from AI response field `"tool_parameters"` parsed as JSON.  
  - *Expressions/Variables:*  
    - `toolName`: `={{ $fromAI("tool", "the tool selected") }}`  
    - `toolParameters`: `={{ $fromAI('tool_parameters', ``, 'json') }}`  
  - *Input:* Invoked by AI Task Planner via `ai_tool` connection.  
  - *Output:* Returns execution results to AI Task Planner.  
  - *Version:* 1  
  - *Credentials:* Requires MCP Notion official credentials.  
  - *Potential Failures:* Invalid tool names, incorrect parameters format, authentication failures, API errors from MCP server.  
  - *Sub-workflow:* None.

---

### 2.5 Setup and Documentation

**Overview:**  
Sticky notes provide important instructions, disclaimers, and tips for users and maintainers.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6

**Node Details:**  

- **Sticky Note**  
  - Provides a high-level description and visual example of the AI assistant connected to Notion, showing use cases like querying information, creating tasks, and managing databases.  
  - Includes an embedded image link illustrating the AI agent in action.

- **Sticky Note1**  
  - Contains a disclaimer stating that this template is only available for self-hosted use because it requires a community node.

- **Sticky Note2**  
  - Offers a tip to enrich the AI agent's context by completing the system message in the AI Task Planner node.

- **Sticky Note3**  
  - Setup step 1: Instruction to enter the chat model API key, noting Gemini as an example but allowing other models.

- **Sticky Note4**  
  - Setup step 2: Instructions for installing the `n8n-nodes-mcp` community node with links to the npm package and installation guide.

- **Sticky Note5**  
  - Setup step 3: Detailed instructions on configuring Notion MCP server credentials, including command-line arguments, environment variable handling, Docker-specific environment variable passing, and a link to the MCP server installation guide.

- **Sticky Note6**  
  - Setup step 4: Encourages users to open the chat interface and start interacting with the assistant, with a caution about the pages the AI can access.

All sticky notes are simple UI nodes with textual content and no input or output connections.

---

## 3. Summary Table

| Node Name                 | Node Type                               | Functional Role                                  | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                              |
|---------------------------|---------------------------------------|-------------------------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Receives chat messages and triggers workflow    | (trigger)                    | AI Task Planner             |                                                                                                        |
| Simple Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational memory context         | AI Task Planner (ai_memory)  | AI Task Planner             |                                                                                                        |
| Google Gemini Chat Model   | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Executes Google Gemini AI language model         | AI Task Planner (ai_languageModel) | AI Task Planner         | Setup step 1 - Enter your Chat model API key. Gemini is used here but other models can be used.         |
| AI Task Planner           | @n8n/n8n-nodes-langchain.agent         | Orchestrates AI reasoning and task planning     | When chat message received, Simple Memory, Google Gemini Chat Model, Notion tools | Simple Memory, Notion - list tools, Notion - execute tool | Tip: Give some context to your AI agent by completing the system message in this node.                  |
| Notion - list tools       | n8n-nodes-mcp.mcpClientTool            | Lists available Notion tools for AI interaction | AI Task Planner (ai_tool)    | AI Task Planner             | Setup step 2 - Install the n8n-nodes-mcp community node: https://www.npmjs.com/package/n8n-nodes-mcp    |
| Notion - execute tool     | n8n-nodes-mcp.mcpClientTool            | Executes selected Notion tool with parameters   | AI Task Planner (ai_tool)    | AI Task Planner             | Setup step 3 - Enter credentials for Notion MCP server with environment configuration details. See https://github.com/makenotion/notion-mcp-server |
| Sticky Note               | n8n-nodes-base.stickyNote              | Provides workflow description and visual example| None                        | None                       | Create an AI assistant connected to Notion with chat interaction and task management capabilities.      |
| Sticky Note1              | n8n-nodes-base.stickyNote              | Disclaimer about template availability           | None                        | None                       | This template is only available in self-hosted as it requires a community node.                         |
| Sticky Note2              | n8n-nodes-base.stickyNote              | Tip to enrich AI context                          | None                        | None                       | Tip: Give some context to your AI agent by completing the system message in the AI Task Planner node.   |
| Sticky Note3              | n8n-nodes-base.stickyNote              | Setup step 1 - Chat model API key entry          | None                        | None                       | Setup step 1 - Enter your Chat model API key. Gemini is used in this example but you can use the model of your choice. |
| Sticky Note4              | n8n-nodes-base.stickyNote              | Setup step 2 - Community node installation        | None                        | None                       | Setup step 2 - Install the [n8n-nodes-mcp community node](https://www.npmjs.com/package/n8n-nodes-mcp). [How to install a community node](https://docs.n8n.io/integrations/community-nodes/installation/). |
| Sticky Note5              | n8n-nodes-base.stickyNote              | Setup step 3 - Notion MCP server credentials     | None                        | None                       | Setup step 3 - Enter your credentials for the Notion MCP server. See installation guide: https://github.com/makenotion/notion-mcp-server |
| Sticky Note6              | n8n-nodes-base.stickyNote              | Setup step 4 - Start interacting with assistant  | None                        | None                       | Setup step 4 - Click on "open chat" below and start interacting with your assistant. Be cautious regarding pages accessible by the AI. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add node: `When chat message received` (`@n8n/n8n-nodes-langchain.chatTrigger`)  
   - Configure: Disable file uploads (`allowFileUploads: false`)  
   - Position at top-left for clarity.

2. **Create AI Memory Node**  
   - Add node: `Simple Memory` (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)  
   - Use default settings.  
   - Position below and to the right of AI Task Planner node (to be created next).

3. **Create AI Language Model Node**  
   - Add node: `Google Gemini Chat Model` (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)  
   - Set model name to `"models/gemini-2.5-pro-preview-06-05"`.  
   - Assign Google Palm API credentials (OAuth2 or API key) with proper permissions.  
   - Position to the right of Simple Memory node.

4. **Create AI Agent Node**  
   - Add node: `AI Task Planner` (`@n8n/n8n-nodes-langchain.agent`)  
   - Configure system message with instructions:  
     ```
     You are a helpful assistant. 

     You have access to my Notion workspace. You can retrieve the list of available Notion tools using the node “Notion - list tools”.

     Here is the ID of my task database:20d45c70c57381f09418d42c78ad360b.

     If you need to interact with a database, first use your tools to get its structure and properties. Never ask for the properties if you can obtain them by yourself.

     Also avoid to ask the exact names of pages to the user. Use the context you have to determine the pages that should be impacted.
     ```  
   - Position below the Trigger node but left of Simple Memory.

5. **Create Notion MCP Nodes**  
   - Add node: `Notion - list tools` (`n8n-nodes-mcp.mcpClientTool`)  
     - No parameters needed.  
     - Assign MCP Notion official credentials (token with correct scopes).  
     - Position to the right of AI Task Planner.
   - Add node: `Notion - execute tool` (`n8n-nodes-mcp.mcpClientTool`)  
     - Configure parameters:  
       - `toolName`: expression `={{ $fromAI("tool", "the tool selected") }}`  
       - `toolParameters`: expression `={{ $fromAI('tool_parameters', ``, 'json') }}`  
     - Assign same MCP Notion official credentials.  
     - Position to the right of Notion - list tools.

6. **Connect Nodes**  
   - Connect `When chat message received` main output to `AI Task Planner` main input.  
   - Connect `AI Task Planner` memory output (`ai_memory`) to `Simple Memory` input, and `Simple Memory` output back to `AI Task Planner` memory input.  
   - Connect `AI Task Planner` language model output (`ai_languageModel`) to `Google Gemini Chat Model` input and back.  
   - Connect `AI Task Planner` tool output (`ai_tool`) to both `Notion - list tools` and `Notion - execute tool` inputs, with their outputs returning back to `AI Task Planner` tool input.  

7. **Add Sticky Notes** (optional but recommended for clarity and documentation)  
   - Add sticky notes with the content and positions matching the original workflow for setup steps, disclaimers, and tips.  
   - Useful for end users and maintainers.

8. **Credential Setup**  
   - Configure Google Palm API credentials for Gemini node (API key or OAuth2).  
   - Configure MCP Notion credentials for MCP nodes using the official Notion MCP server token.  
   - Ensure environment variables and Docker commands are set as per Sticky Note5 instructions if using MCP server in Docker.

9. **Final Testing**  
   - Deploy the workflow.  
   - Send chat messages through the webhook endpoint exposed by `When chat message received`.  
   - Verify that the AI responds and can list and execute Notion tools correctly.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This template is only available in self-hosted mode as it requires the community node `n8n-nodes-mcp`.                                                                                                                             | See Sticky Note1                                                                                                  |
| Install the community node `n8n-nodes-mcp` via npm for Notion integration: https://www.npmjs.com/package/n8n-nodes-mcp                                                                                                              | Sticky Note4 and official n8n documentation on community node installation: https://docs.n8n.io/integrations/community-nodes/installation/ |
| Notion MCP server installation and environment variable handling guide: https://github.com/makenotion/notion-mcp-server                                                                                                             | Sticky Note5                                                                                                      |
| Google Gemini (PaLM) is used as the language model but other models compatible with LangChain nodes can be substituted.                                                                                                            | Sticky Note3                                                                                                      |
| Caution advised when granting AI assistant access to Notion pages to avoid unintended data exposure.                                                                                                                                | Sticky Note6                                                                                                      |
| Visual example of AI assistant interaction with Notion is available at: ![Your AI Agent in action](https://lh3.googleusercontent.com/d/18Io1JU1E1_Z0a0jMqwleRqInkYFllZkQ)                                                                 | Sticky Note                                                                                                       |

---

**Disclaimer:**  
The text provided is extracted exclusively from an n8n automated workflow. All processing respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---