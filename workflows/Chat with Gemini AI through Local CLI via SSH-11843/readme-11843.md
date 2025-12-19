Chat with Gemini AI through Local CLI via SSH

https://n8nworkflows.xyz/workflows/chat-with-gemini-ai-through-local-cli-via-ssh-11843


# Chat with Gemini AI through Local CLI via SSH

---

### 1. Workflow Overview

This workflow implements a custom AI chat interface powered by Google Gemini AI, integrated via a local CLI executed over SSH. Its main purpose is to enable conversational interactions with Gemini AI through a secure command-line interface hosted on a server accessible by n8n. The architecture is split into a primary chat handling workflow and a sub-workflow acting as a CLI tool executor.

The workflow is logically divided into the following blocks:

- **1.1 Chat Input Reception:** Captures incoming chat messages via a webhook trigger designed for streaming sessions and session memory.
- **1.2 AI Conversation Memory:** Maintains conversational context using a simple memory buffer window to hold recent dialogue.
- **1.3 AI Agent Processing:** Handles the chat input by invoking an AI agent configured to use the Gemini CLI tool before formulating a final answer.
- **1.4 Gemini CLI Tool Sub-Workflow:** Executes the Gemini CLI commands via SSH on a local or remote machine, returning the CLI response to the main workflow.
- **1.5 Output Handling and No-Op:** Finalizes processing with a no-operation node to complete the flow gracefully.
- **1.6 Documentation & Instructions:** Sticky notes provide detailed setup, configuration, and operational guidelines.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Chat Input Reception

- **Overview:**  
  This block initiates the workflow by receiving chat messages from users via an HTTP webhook. It supports streaming responses and maintains session data in memory.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* Langchain Chat Trigger (Webhook)  
    - *Configuration:*  
      - Mode: Webhook  
      - Public access enabled (no authentication)  
      - Response mode: Streaming, allowing partial output as the AI generates it  
      - Session memory: Loads previous session data from memory to maintain context  
    - *Key expressions:* None, uses default webhook payload for chat input  
    - *Input connections:* None (entry point)  
    - *Output connections:* Main output connected to "AI Agent" node to process input  
    - *Potential failures:*  
      - Network or webhook delivery issues  
      - Session memory load failures if corrupted or missing  
    - *Version:* 1.3  

#### Block 1.2: AI Conversation Memory

- **Overview:**  
  Maintains a windowed memory buffer of recent conversation turns to provide context continuity for the AI agent.

- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**  
  - **Simple Memory**  
    - *Type:* Langchain Memory Buffer Window  
    - *Configuration:* Default settings, maintains recent chat history in memory buffer  
    - *Key expressions:* None  
    - *Input connections:* Receives AI memory input from "When chat message received" node  
    - *Output connections:* None explicitly connected (used internally for memory)  
    - *Potential failures:* Memory buffer overflow or corruption, if very large sessions  
    - *Version:* 1.3  

#### Block 1.3: AI Agent Processing

- **Overview:**  
  Processes the chat input using a Langchain AI agent that is explicitly instructed to use the Gemini CLI tool before providing a final response.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Call 'Gemini CLI'  
  - No Operation, do nothing

- **Node Details:**  
  - **AI Agent**  
    - *Type:* Langchain Agent  
    - *Configuration:*  
      - Input text sourced from chat input JSON field  
      - System message instructs agent to always use the Gemini CLI tool before answering  
      - Prompt type: Custom defined  
      - Tools: Gemini CLI (via sub-workflow) and Google Gemini Chat Model (LLM)  
    - *Key expressions:* `{{$json.chatInput}}` for user input  
    - *Input connections:* Main input from "When chat message received" node  
    - *Output connections:* Main output connected to "No Operation, do nothing" node  
    - *Potential failures:*  
      - Agent misinterpretation if system prompt is altered incorrectly  
      - Tool invocation failures if sub-workflow unavailable  
      - Model API rate limits or network failures  
    - *Version:* 2.2  

  - **Google Gemini Chat Model**  
    - *Type:* Langchain Google Gemini Chat LLM  
    - *Configuration:* Default options  
    - *Input connections:* AI languageModel input from "AI Agent" node  
    - *Output connections:* AI languageModel output to "AI Agent" node  
    - *Potential failures:* Authentication errors with Google account, API limits, network issues  
    - *Version:* 1  

  - **Call 'Gemini CLI'**  
    - *Type:* Langchain Tool Workflow (Sub-workflow call)  
    - *Configuration:*  
      - Calls a separate sub-workflow named "Gemini CLI" via its workflow ID  
      - Maps input prompt from `{{$json.chatInput}}` to sub-workflow parameter "prompt"  
      - Described as a CLI tool interface  
    - *Input connections:* AI tool input from "AI Agent" node  
    - *Output connections:* AI tool output to "AI Agent" node  
    - *Potential failures:*  
      - Sub-workflow missing or misconfigured  
      - Parameter mapping errors  
    - *Version:* 2.2  

  - **No Operation, do nothing**  
    - *Type:* NoOp (no operation)  
    - *Configuration:* Default, acts as a sink node  
    - *Input connections:* Main input from "AI Agent" node  
    - *Output connections:* None  
    - *Potential failures:* None, purely structural  

#### Block 1.4: Gemini CLI Tool Sub-Workflow

- **Overview:**  
  Separate workflow that executes the Gemini CLI command over SSH and returns the output. This is intended to be copied to a separate workflow and linked back as a tool.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Execute a command (SSH)  
  - Edit Fields  
  - Sticky Note1 (instructional)

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Configuration:* Accepts inputs with parameter "prompt"  
    - *Input connections:* None (trigger entry)  
    - *Output connections:* Main output to "Execute a command" node  
    - *Potential failures:* Input parameter missing or malformed  
    - *Version:* 1.1  

  - **Execute a command**  
    - *Type:* SSH Node  
    - *Configuration:*  
      - Executes command: `gemini "{{ $json.prompt }}"`  
      - Requires SSH credentials configured in n8n (host, user, key/pass)  
    - *Input connections:* Main input from "When Executed by Another Workflow" node  
    - *Output connections:* Main output to "Edit Fields" node  
    - *Potential failures:*  
      - SSH connection failures (network, auth, host unreachable)  
      - Command execution errors (gemini CLI not installed, invalid command)  
      - Timeout or resource constraints  
    - *Version:* 1  

  - **Edit Fields**  
    - *Type:* Set Node  
    - *Configuration:* Assigns the SSH execution standard output (`$json.stdout`) to a new field "answer"  
    - *Input connections:* Main input from "Execute a command" node  
    - *Output connections:* None (output back to calling workflow)  
    - *Potential failures:* Missing or empty stdout, JSON parsing errors  
    - *Version:* 3.4  

  - **Sticky Note1**  
    - *Purpose:* Instructional note explaining the requirement to move these nodes to a separate workflow and link it as a tool in the main workflow.  
    - *Content Highlights:*  
      - Copy nodes to new workflow  
      - Execute gemini CLI command via SSH  
      - Return LLM response message  

#### Block 1.5: Output Handling and End of Flow

- **Overview:**  
  This block finalizes the conversation processing by connecting the AI Agent output to a no-operation node, effectively marking the end of the workflow execution.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**  
  See above in Block 1.3.

#### Block 1.6: Documentation & Instructions

- **Overview:**  
  Several sticky notes provide comprehensive setup instructions, configuration notes, and contact information.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**  
  - **Sticky Note**  
    - Detailed prerequisites for environment setup including NodeJS version, Gemini CLI installation, SSH access  
    - Instructions for separating the sub-workflow and linking it as a tool  
    - Contact information and website for support  

  - **Sticky Note2**  
    - Explains AI Agent configuration, emphasizing the system prompt to mandate CLI tool usage  
    - Notes on extensibility by adding more tools  

  - **Sticky Note3**  
    - Describes the chat interface trigger node’s customizable and embeddable nature  

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                       | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                                      |
|-----------------------------|---------------------------------------------|------------------------------------|-----------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------|
| When chat message received  | Langchain Chat Trigger (Webhook)            | Chat message reception trigger     | None                        | AI Agent                       | See Sticky Note3: Chat Interface Trigger customizable and embeddable                                            |
| Simple Memory               | Langchain Memory Buffer Window               | Conversation memory buffer         | When chat message received (ai_memory) | None                          |                                                                                                                  |
| Google Gemini Chat Model    | Langchain LLM Chat Model                      | AI language model                  | AI Agent (ai_languageModel)  | AI Agent                       |                                                                                                                  |
| Call 'Gemini CLI'           | Langchain Tool Workflow (Sub-workflow call) | CLI tool invocation                | AI Agent (ai_tool)           | AI Agent                       | See Sticky Note2: AI Agent Configuration notes about CLI tool and system prompt                                 |
| AI Agent                   | Langchain Agent                              | Process chat input using AI agent | When chat message received   | No Operation, do nothing        | See Sticky Note2                                                                                                 |
| No Operation, do nothing    | NoOp Node                                   | Final output node                  | AI Agent                    | None                          |                                                                                                                  |
| Sticky Note                 | Sticky Note                                 | Workflow setup and usage instructions | None                        | None                          | See content in section 1.6                                                                                        |
| Execute a command           | SSH Node                                    | Run Gemini CLI command via SSH    | When Executed by Another Workflow | Edit Fields                  |                                                                                                                  |
| When Executed by Another Workflow | Execute Workflow Trigger               | Sub-workflow entry trigger        | None                        | Execute a command               | See Sticky Note1: Instructions to move to separate workflow                                                     |
| Edit Fields                 | Set Node                                    | Assign SSH output to "answer" field | Execute a command            | None                          |                                                                                                                  |
| Sticky Note1                | Sticky Note                                 | Instructions for sub-workflow setup | None                        | None                          | See section 1.4                                                                                                  |
| Sticky Note2                | Sticky Note                                 | AI Agent configuration notes      | None                        | None                          | See section 1.3                                                                                                  |
| Sticky Note3                | Sticky Note                                 | Chat interface trigger notes      | None                        | None                          | See section 1.1                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Main Workflow**

   1.1. Add a **Langchain Chat Trigger** node named **"When chat message received"**  
       - Set mode to "webhook"  
       - Enable public access  
       - Configure response mode to "streaming"  
       - Enable loading previous session from memory

   1.2. Add a **Langchain Memory Buffer Window** node named **"Simple Memory"**  
       - Use default settings  
       - Connect its **ai_memory** input to the **When chat message received** node's **ai_memory** output

   1.3. Add a **Langchain Google Gemini Chat Model** node named **"Google Gemini Chat Model"**  
       - Use default options

   1.4. Add a **Langchain Agent** node named **"AI Agent"**  
       - Set the input text expression to `{{$json.chatInput}}`  
       - Under options, set the system message:  
         ```  
         Every time you MUST use the CLI tool before you provide final answer!  
         ```  
       - Set prompt type to "define" (custom)  
       - Connect **When chat message received** main output to **AI Agent** main input  
       - Connect **Google Gemini Chat Model** output to **AI Agent** AI languageModel input  
       - Connect **Call 'Gemini CLI'** output to **AI Agent** AI tool input  

   1.5. Add a **Langchain Tool Workflow** node named **"Call 'Gemini CLI'"**  
       - Select the sub-workflow you will create in step 2  
       - Map workflow input "prompt" to `{{$json.chatInput}}`  
       - Connect output to **AI Agent** AI tool input

   1.6. Add a **No Operation** node named **"No Operation, do nothing"**  
       - Connect **AI Agent** main output to this node

2. **Create the Sub-Workflow (Gemini CLI Executor)**

   2.1. Add an **Execute Workflow Trigger** node named **"When Executed by Another Workflow"**  
       - Define workflow input parameter "prompt" (string)

   2.2. Add an **SSH** node named **"Execute a command"**  
       - Command: `gemini "{{ $json.prompt }}"`  
       - Configure SSH credentials (host, username, authentication method) for target machine running Gemini CLI  
       - Connect **When Executed by Another Workflow** main output to this node

   2.3. Add a **Set** node named **"Edit Fields"**  
       - Assign a new field named "answer" with value `{{$json.stdout}}` (SSH command output)  
       - Connect **Execute a command** main output to this node

3. **Configure Credentials**

   - SSH Node: Add or update SSH credentials with valid host, username, and authentication (password or private key) for the Gemini CLI host machine.
   - Google Gemini Chat Model node: Ensure Google authentication credentials are configured and valid for API access.

4. **Connect Sub-Workflow to Main Workflow**

   - In the main workflow, open the **Call 'Gemini CLI'** node and select the sub-workflow you created.
   - Save both workflows.

5. **Test the Workflow**

   - Trigger the **When chat message received** webhook with a chat input.
   - Verify the AI Agent invokes the Gemini CLI via SSH and returns output.
   - Check logs for SSH connection and command execution errors if any.

6. **Optional: Add Sticky Notes**

   - Add sticky notes to document setup instructions, agent configuration, chat trigger info, and sub-workflow usage following the original content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                | Context or Link                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Connect Gemini CLI with your n8n workflow: Requires n8n self-hosted, NodeJS >= 20.x, Gemini CLI installed globally via npm, and SSH access set up. Separate the sub-workflow and link it as a tool. Contact Alex at alex@elitiv.com or visit https://www.elitiv.com for support. | Setup instructions in Sticky Note                  |
| AI Agent is configured to always use the Gemini CLI tool before answering, customizable system prompt. Extensible with additional tools like Google Search or Wikipedia integration.                                                                                      | AI Agent configuration notes in Sticky Note2       |
| Chat Interface Trigger node is customizable and embeddable for integration into websites or applications.                                                                                                                                                                   | Chat interface notes in Sticky Note3                |
| Gemini CLI command executed on remote host over SSH: `gemini "{{ $json.prompt }}"`. Ensure Gemini CLI is installed on the host and accessible via SSH user configured in n8n.                                                                                             | Sub-workflow command node details                   |

---

**Disclaimer:**  
The text provided derives exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---