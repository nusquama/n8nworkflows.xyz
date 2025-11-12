Natural Language Task Management with Todoist and GPT-4o

https://n8nworkflows.xyz/workflows/natural-language-task-management-with-todoist-and-gpt-4o-4186


# Natural Language Task Management with Todoist and GPT-4o

### 1. Workflow Overview

This workflow, titled **"Natural Language Task Management with Todoist and GPT-4o"**, enables natural language interaction to manage Todoist tasks using advanced AI capabilities from OpenAI's GPT-4o model. It is designed for users who want to operate Todoist purely via conversational commands, automating creation, updating, querying, and organization of tasks, projects, sections, and labels without manual UI interaction.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Entry points for chat messages and external workflow calls trigger the process.
- **1.2 Orchestration & AI Processing:** An orchestrator node manages the conversation flow and memory, invoking specific Todoist actions.
- **1.3 Todoist Agent:** The core AI agent node interprets natural language commands and maps them to Todoist API operations.
- **1.4 Todoist API Operations:** Multiple HTTP Request Tool and Todoist Tool nodes perform specific CRUD actions on Todoist tasks, projects, sections, labels, and comments.
- **1.5 Memory Management:** A simple memory buffer stores conversation context.
- **1.6 Supporting Nodes:** Sticky notes provide documentation and usage instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Handles incoming requests either from chat interfaces or other workflows, triggering the workflow execution.

**Nodes Involved:**  
- When chat message received  
- When Executed by Another Workflow

**Node Details:**  

- **When chat message received**  
  - Type: Chat Trigger (Langchain)  
  - Role: Activates the workflow upon receiving a chat input (e.g., from Telegram, Slack).  
  - Config: Standard with webhookId for external chat platform integration.  
  - Inputs: External chat messages.  
  - Outputs: Connected to the Orchestrator node.  
  - Potential Failures: Webhook misconfiguration, missing credentials, network issues.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be triggered by other workflows, passing sessionID and chatInput parameters.  
  - Config: Defines expected input parameters: sessionID and chatInput.  
  - Inputs: Workflow calls with input parameters.  
  - Outputs: Connected to the Todoist Agent node.  
  - Potential Failures: Missing or malformed input parameters, workflow execution errors.

---

#### 1.2 Orchestration & AI Processing

**Overview:**  
Manages the conversational logic, maintaining context and calling the Todoist Agent with user input.

**Nodes Involved:**  
- Orchestrator  
- Simple Memory  
- OpenAI Chat Model1  
- Call Todoist Agent

**Node Details:**  

- **Orchestrator**  
  - Type: Langchain Agent  
  - Role: Central controller that processes chat input and decides next steps.  
  - Config: Minimal options, uses OpenAI GPT-4o as backend.  
  - Inputs: From "When chat message received".  
  - Outputs: Calls "Simple Memory" and "Call Todoist Agent".  
  - Failures: AI API errors, rate limits, memory overflow.

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Stores recent chat history to maintain conversational context.  
  - Config: Default buffer window size.  
  - Inputs: From Orchestrator.  
  - Outputs: Back to Orchestrator for contextual awareness.  
  - Failures: Memory corruption, data loss.

- **OpenAI Chat Model1**  
  - Type: Langchain OpenAI Chat Model Node  
  - Role: AI model instance (GPT-4o-latest) used by the Orchestrator.  
  - Config: Uses OpenAI API credentials.  
  - Inputs: Prompts from Orchestrator.  
  - Outputs: AI responses back to Orchestrator.  
  - Failures: API key invalid, quota exhausted, connectivity issues.

- **Call Todoist Agent**  
  - Type: Langchain Tool Workflow  
  - Role: Invokes the Todoist Agent sub-workflow with chatInput and sessionID parameters.  
  - Config: Calls the current workflow ID as a tool, passing relevant inputs.  
  - Inputs: From Orchestrator.  
  - Outputs: Depends on Todoist Agent output.  
  - Failures: Sub-workflow errors, parameter mismatch.

---

#### 1.3 Todoist Agent

**Overview:**  
Interprets natural language commands specifically for Todoist task management, deciding which API calls to make.

**Nodes Involved:**  
- Todoist Agent  
- OpenAI Chat Model  
- All Todoist API operation nodes (Tasks, Projects, Sections, Labels, Comments)

**Node Details:**  

- **Todoist Agent**  
  - Type: Langchain Agent (specialized)  
  - Role: Converts natural language input into Todoist actions, limiting up to two sequential steps.  
  - Config: Custom system message emphasizing privacy, language matching, and user-friendly responses.  
  - Inputs: From "When Executed by Another Workflow" or "Call Todoist Agent".  
  - Outputs: Routes to the appropriate Todoist Tool or HTTP Request nodes for API interaction.  
  - Failures: Expression evaluation errors, AI misinterpretation, incomplete user input.

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model Node  
  - Role: Provides AI capabilities for the Todoist Agent node.  
  - Config: Uses GPT-4o-latest with OpenAI credentials.  
  - Failures: API key/auth errors, rate limiting.

- **Todoist API Operation Nodes** (Examples below):  
  - **Get All Tasks:** Lists tasks via Todoist filter syntax with OAuth2 authentication.  
  - **Create a Task:** Creates tasks with multiple optional fields like labels, due dates, priority, and project.  
  - **Update a Task:** Updates task properties, requiring taskId.  
  - **Mark Task as Done, Reopen a Task, Move a Task:** Task state and organizational changes.  
  - **Get All Projects, Get a Project, Create/Update/Archive/Unarchive a Project:** Full project lifecycle management.  
  - **Get All Sections, Get a Section, Create/Update/Delete a Section:** Section management within projects.  
  - **Get All Labels, Get a Label, Create/Update/Delete a Label:** Label/tag management.  
  - **Add a Comment:** Adds text comments to tasks.  

Each node:  
  - Type: Mix of n8n Todoist Tool nodes and HTTP Request Tool nodes with OAuth2 credentials.  
  - Inputs: From Todoist Agent via AI tool interface.  
  - Outputs: Return concise, user-facing confirmation or data.  
  - Failures: OAuth token expiry, API rate limits, invalid parameters, network errors, expression evaluation issues.

---

#### 1.4 Memory Management

**Overview:**  
Maintains recent conversation history to enable contextual understanding across user interactions.

**Nodes Involved:**  
- Simple Memory (already covered in 1.2)

---

#### 1.5 Supporting Nodes

**Overview:**  
Provide documentation, instructions, and usage notes embedded in the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  

**Node Details:**  

- **Sticky Note**  
  - Type: Sticky Note Node  
  - Role: Detailed documentation on workflow purpose, setup instructions, quick start, customization ideas, and limitations.  
  - Content: Extensive markdown with credential setup, usage examples, and contact info.

- **Sticky Note1**  
  - Type: Sticky Note Node  
  - Role: Notes about the Orchestrator node as an example; suggests that in production, the Todoist Agent can be exposed directly as a tool without orchestration.

---

### 3. Summary Table

| Node Name                | Node Type                               | Functional Role                                  | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                     |
|--------------------------|---------------------------------------|-------------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger | Chat input trigger for user messages             | —                             | Orchestrator                   |                                                                                                                |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger | Triggered externally by other workflows          | —                             | Todoist Agent                  |                                                                                                                |
| Orchestrator             | @n8n/n8n-nodes-langchain.agent        | Conversational orchestration and AI processing   | When chat message received     | Simple Memory, Call Todoist Agent| *The Orchestrator is an example.* In production you can drop it and simply expose the **Todoist Agent** as a tool for any other agent workflow. |
| Simple Memory            | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational context                  | Orchestrator                  | Orchestrator                   |                                                                                                                |
| Call Todoist Agent       | @n8n/n8n-nodes-langchain.toolWorkflow | Invokes Todoist Agent sub-workflow with inputs   | Orchestrator                  | Todoist Agent                  |                                                                                                                |
| Todoist Agent            | @n8n/n8n-nodes-langchain.agent        | AI agent interpreting natural language commands  | When Executed by Another Workflow, Call Todoist Agent| Multiple Todoist API nodes   |                                                                                                                |
| OpenAI Chat Model        | @n8n/n8n-nodes-langchain.lmChatOpenAi | AI model for Todoist Agent                        | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| OpenAI Chat Model1       | @n8n/n8n-nodes-langchain.lmChatOpenAi | AI model for Orchestrator                         | Orchestrator                 | Orchestrator                  |                                                                                                                |
| Get All Tasks            | n8n-nodes-base.todoistTool            | List Todoist tasks via filter                     | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Get a Task               | n8n-nodes-base.todoistTool            | Retrieve single task details                       | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Create a Task            | n8n-nodes-base.todoistTool            | Create new task                                   | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Update a Task            | n8n-nodes-base.todoistTool            | Update existing task properties                   | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Mark Task as Done        | n8n-nodes-base.todoistTool            | Mark task completed                               | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Reopen a Task            | n8n-nodes-base.todoistTool            | Reopen completed task                             | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Move a Task              | n8n-nodes-base.todoistTool            | Move task between projects/sections or as subtask| Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Get All Projects         | n8n-nodes-base.httpRequestTool        | List all user projects                            | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Get a Project            | n8n-nodes-base.httpRequestTool        | Get single project details                        | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Create a Project         | n8n-nodes-base.httpRequestTool        | Create new project                               | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Update a Project         | n8n-nodes-base.httpRequestTool        | Update project properties                         | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Archive a Project        | n8n-nodes-base.httpRequestTool        | Archive a project                                | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Unarchive a Project      | n8n-nodes-base.httpRequestTool        | Unarchive a previously archived project          | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Get Archived Projects    | n8n-nodes-base.httpRequestTool        | List archived projects                            | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Get All Sections         | n8n-nodes-base.httpRequestTool        | List all sections                                | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Get a Section            | n8n-nodes-base.httpRequestTool        | Get section details                              | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Create a Section         | n8n-nodes-base.httpRequestTool        | Create new section                               | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Update a Section         | n8n-nodes-base.httpRequestTool        | Update section name                              | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Delete a Section         | n8n-nodes-base.httpRequestTool        | Delete a section                                 | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Get All Labels           | n8n-nodes-base.httpRequestTool        | List all labels                                  | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Get a Label              | n8n-nodes-base.httpRequestTool        | Get label details                                | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Create a Label           | n8n-nodes-base.httpRequestTool        | Create new label                                 | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Update a Label           | n8n-nodes-base.httpRequestTool        | Update label properties                          | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Delete a Label           | n8n-nodes-base.httpRequestTool        | Delete a label                                   | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Add a Comment            | n8n-nodes-base.httpRequestTool        | Add comment to task                              | Todoist Agent                 | Todoist Agent                  |                                                                                                                |
| Sticky Note              | n8n-nodes-base.stickyNote              | Documentation & usage instructions               | —                             | —                              | # 🪄 Ultimate Personal **Todoist Agent** … Extensive instructions, setup, and contact info.                    |
| Sticky Note1             | n8n-nodes-base.stickyNote              | Note on Orchestrator usage                        | —                             | —                              | *The Orchestrator is an example.* In production you can drop it and simply expose the **Todoist Agent** as a tool for any other agent workflow. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add **When chat message received** node (Langchain chatTrigger). Configure webhook to receive chat messages.
   - Add **When Executed by Another Workflow** node. Define input parameters `sessionID` and `chatInput`.

2. **Add Orchestrator Block:**

   - Add **Orchestrator** node (`@n8n/n8n-nodes-langchain.agent`), no special options needed.
   - Add **Simple Memory** node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`) connected to Orchestrator for context storage.
   - Add **OpenAI Chat Model1** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`), set model to `chatgpt-4o-latest`, attach OpenAI API credentials.
   - Connect Orchestrator to Simple Memory and OpenAI Chat Model1 accordingly.
   - Add **Call Todoist Agent** node (`@n8n/n8n-nodes-langchain.toolWorkflow`), configured to call this same workflow as a tool.
   - Set inputs for Call Todoist Agent: map `chatInput` and `sessionID` from Orchestrator.
   - Connect Orchestrator output to Call Todoist Agent.

3. **Build Todoist Agent Block:**

   - Add **Todoist Agent** node (`@n8n/n8n-nodes-langchain.agent`).
   - Configure system message to instruct AI on Todoist-specific behavior, emphasizing privacy, language matching, and action limits.
   - Connect **When Executed by Another Workflow** and **Call Todoist Agent** outputs to this node's input.
   - Add **OpenAI Chat Model** node for Todoist Agent, same model and credentials as above.
   - Connect Todoist Agent node to OpenAI Chat Model.

4. **Add Todoist API Nodes:**

   For each Todoist operation, add the corresponding node as below, configure OAuth2 credentials (Todoist OAuth2):

   - **Tasks:**
     - Get All Tasks (operation: getAll, filter dynamic from AI)
     - Get a Task (taskId from AI)
     - Create a Task (content mandatory, other optional from AI)
     - Update a Task (taskId required, update fields from AI)
     - Mark Task as Done (taskId required)
     - Reopen a Task (taskId required)
     - Move a Task (taskId and project/section/parent IDs from AI)
   
   - **Projects:**
     - Get All Projects (HTTP GET to `/rest/v2/projects`)
     - Get a Project (GET `/rest/v2/projects/{project_id}`)
     - Create a Project (POST with JSON body including name, description, color, parent_id, is_favorite)
     - Update a Project (POST to `/rest/v2/projects/{project_id}` with JSON body)
     - Archive a Project (POST `/rest/v2/projects/{project_id}/archive`)
     - Unarchive a Project (POST `/rest/v2/projects/{project_id}/unarchive`)
     - Get Archived Projects (GET `/api/v1/projects/archived`)

   - **Sections:**
     - Get All Sections (GET `/rest/v2/sections`)
     - Get a Section (GET `/rest/v2/sections/{section_id}`)
     - Create a Section (POST with name and project_id)
     - Update a Section (POST with new name)
     - Delete a Section (DELETE `/rest/v2/sections/{section_id}`)

   - **Labels:**
     - Get All Labels (GET `/rest/v2/labels`)
     - Get a Label (GET `/rest/v2/labels/{label_id}`)
     - Create a Label (POST with name, color, is_favorite)
     - Update a Label (POST with name, color, is_favorite)
     - Delete a Label (DELETE `/rest/v2/labels/{label_id}`)

   - **Comments:**
     - Add a Comment (POST `/rest/v2/comments` with content and task_id)

5. **Connect Todoist Agent output to each Todoist API node as an AI tool interface.**  
   This allows the agent to route commands dynamically.

6. **Add Supporting Sticky Notes:**

   - Add one large sticky note with detailed documentation and instructions (content as per original).
   - Add a smaller sticky note near Orchestrator explaining it is optional and can be replaced by exposing Todoist Agent directly.

7. **Credentials Setup:**

   - Configure **OpenAI API** credential with your OpenAI secret key.
   - Configure **Todoist OAuth2** credential by logging into Todoist via OAuth2 in n8n.

8. **Testing and Validation:**

   - Test by sending a natural language command via chat webhook or by triggering the workflow with sample inputs.
   - Verify that tasks/projects/labels are created, updated, or queried correctly in Todoist.
   - Monitor AI responses for accuracy and user-friendly confirmations.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| 🪄 Ultimate Personal **Todoist Agent**: Turns natural-language requests into perfectly-organized Todoist tasks automatically within n8n. Tested with GPT-4o-latest for optimal performance. No webhooks or extra secrets beyond OpenAI and Todoist OAuth2 credentials are needed.                                                                                                                                                                                                                                           | Main sticky note inside workflow.                                                                                       |
| Quick-start instructions: Import JSON, select credentials, test with sample message, swap trigger node for preferred chat platform (Telegram, Slack, etc.).                                                                                                                                                                                                                                                                                                                                                                  | Main sticky note inside workflow.                                                                                       |
| Customization ideas: integrate Notion or Google Sheets nodes for logging, swap chat trigger for speech-to-text (Whisper).                                                                                                                                                                                                                                                                                                                                                                                                    | Main sticky note inside workflow.                                                                                       |
| Limitations: no support yet for shared projects, file attachments in comments, or comment editing. Pull requests welcome.                                                                                                                                                                                                                                                                                                                                                                                                    | Main sticky note inside workflow.                                                                                       |
| The Orchestrator is an example node; in production, you can expose the Todoist Agent directly as a tool for other agent workflows.                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note1 near Orchestrator node.                                                                                   |
| OpenAI GPT-4o-latest model is used for best speed and accuracy.                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Configuration detail in OpenAI Chat Model nodes.                                                                        |
| Todoist API color options for projects and labels include berry_red, red, orange, yellow, olive_green, lime_green, green, mint_green, teal, sky_blue, light_blue, blue, grape, violet, lavender, magenta, salmon, charcoal, grey, taup. If user requests unsupported color, fallback to best match or omit.                                                                                                                                                                                                                         | Descriptions in Create/Update Project and Label nodes.                                                                 |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow built with n8n, adhering strictly to content policies, and contains no illegal, offensive, or protected material. All processed data are public and lawful.