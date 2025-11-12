🏠 Find your Home with Real Estate Agent and Bright Data

https://n8nworkflows.xyz/workflows/---find-your-home-with-real-estate-agent-and-bright-data-4872


# 🏠 Find your Home with Real Estate Agent and Bright Data

---

### 1. Workflow Overview

This workflow titled **"🏠 Find your Home with Real Estate Agent and Bright Data"** is designed to serve as an intelligent real estate assistant powered by AI and integrated with Bright Data’s marketplace dataset. It helps users find homes based on specific criteria by querying live real estate data and providing curated property listings.

**Target Use Cases:**
- Conversational real estate search through chat
- Filtering real estate listings by user-defined criteria (city, bedrooms, bathrooms, price)
- Presenting up-to-date property options with images and details
- Automated snapshot creation and retrieval from Bright Data

**Logical Blocks:**

- **1.1 Chat Input Reception and Memory Handling**  
  Receives user chat input, maintains conversational context, and routes queries to the AI agent.

- **1.2 AI Processing and Real Estate Querying**  
  Processes user input with OpenAI’s GPT-4o-mini model and a custom AI agent. It applies user filters and triggers Bright Data API calls to retrieve property snapshots.

- **1.3 Bright Data Integration and Snapshot Management**  
  Handles dataset filtering requests, waits for snapshot readiness, retrieves snapshot content, and feeds back data to the AI agent for final response generation.

- **1.4 Workflow Trigger and Control Logic**  
  Supports external workflow execution and controls asynchronous snapshot readiness with conditional checks and wait loops.

---

### 2. Block-by-Block Analysis

#### 1.1 Chat Input Reception and Memory Handling

**Overview:**  
This block initiates interaction by receiving chat messages, maintaining a sliding window memory buffer of the conversation, and forwarding data to the AI agent for processing.

**Nodes Involved:**  
- When chat message received  
- Simple Memory1  

**Node Details:**

- **When chat message received**  
  - *Type:* Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`)  
  - *Role:* Entry point for user chat messages  
  - *Configuration:* Public webhook; initial message greeting user and prompting for home search criteria  
  - *Key Variables:* `chatInput` (user's message)  
  - *Connections:* Outputs to "Real Estate AI Agent"  
  - *Edge Cases:* Webhook connectivity issues; malformed input messages  

- **Simple Memory1**  
  - *Type:* Memory Buffer Window (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)  
  - *Role:* Maintains last 30 conversational turns to provide context to the AI  
  - *Configuration:* Context window length set to 30  
  - *Connections:* Provides memory context to "Real Estate AI Agent"  
  - *Edge Cases:* Memory overflow if context too large; potential lag in real-time conversations  

---

#### 1.2 AI Processing and Real Estate Querying

**Overview:**  
This block uses OpenAI GPT-4o-mini and a specialized AI agent to interpret user queries, apply filters, and decide when to call Bright Data’s dataset filtering tools.

**Nodes Involved:**  
- OpenAI Chat Model  
- Real Estate AI Agent  

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* Language Model Chat (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)  
  - *Role:* Provides base language understanding with GPT-4o-mini  
  - *Configuration:* Model set to "gpt-4o-mini"; uses OpenAI credentials  
  - *Connections:* Feeds processed language input to "Real Estate AI Agent"  
  - *Edge Cases:* OpenAI API rate limits, authentication errors, model unavailability  

- **Real Estate AI Agent**  
  - *Type:* LangChain Agent (`@n8n/n8n-nodes-langchain.agent`)  
  - *Role:* Central AI logic interpreting user needs, generating filters, and orchestrating dataset queries  
  - *Configuration:*  
    - Max 10 iterations per query  
    - System message details real estate domain knowledge, filter requirements (city, bedrooms, bathrooms, price), and tool usage constraints  
    - Only one call to the "Filter Dataset" tool allowed per session  
    - Processes Bright Data snapshot content output into user-readable property listings with images  
  - *Key Expressions:*  
    - Uses `chatInput` (user message)  
    - Calls "Filter Dataset" and "Get Snapshot Content" tools as per user-provided filters  
  - *Connections:* Receives input from chat trigger, memory, and OpenAI model; outputs to Bright Data tool workflow nodes  
  - *Edge Cases:* Logic errors if user input incomplete; failure to generate or interpret filters; exceeding iteration limits  

---

#### 1.3 Bright Data Integration and Snapshot Management

**Overview:**  
This block manages calls to Bright Data’s marketplace dataset, creates filtered dataset snapshots, waits for snapshot readiness, and retrieves snapshot content for presentation.

**Nodes Involved:**  
- Filter Dataset  
- Get Snapshot Content  
- Recover Snapshot Content  
- Snapshot is ready?  
- Wait  

**Node Details:**

- **Filter Dataset**  
  - *Type:* Bright Data Tool (`n8n-nodes-brightdata.brightDataTool`)  
  - *Role:* Creates a snapshot filtered by user criteria (e.g., homeStatus=FOR_SALE, price, city, bedrooms, bathrooms)  
  - *Configuration:*  
    - Dataset ID dynamically set via AI-generated input  
    - Filters group dynamically constructed from AI input, always includes `homeStatus = FOR_SALE`  
    - Records limit set to 3 results  
    - Bright Data API credentials required  
  - *Connections:* Output connects to "Real Estate AI Agent" as tool response  
  - *Edge Cases:* API key invalid or expired; filters malformed; rate limits; snapshot creation delays  

- **Get Snapshot Content**  
  - *Type:* LangChain Tool Workflow (`@n8n/n8n-nodes-langchain.toolWorkflow`)  
  - *Role:* Invokes a sub-workflow (ID: Ky9jD15PJgT7PIgP) to process snapshot content and convert it to property listing format  
  - *Configuration:* Workflow ID hardcoded; no inputs mapped except empty object  
  - *Connections:* Output feeds back to "Real Estate AI Agent"  
  - *Edge Cases:* Sub-workflow ID mismatch; sub-workflow errors; data format inconsistencies  

- **Recover Snapshot Content**  
  - *Type:* Bright Data API (`n8n-nodes-brightdata.brightData`)  
  - *Role:* Queries Bright Data API to retrieve snapshot content using snapshot ID passed from external workflow or wait loop  
  - *Configuration:* Snapshot ID dynamically retrieved from external trigger data  
  - *Connections:* Output feeds "Snapshot is ready?" node  
  - *Edge Cases:* Snapshot ID invalid; API errors; data unavailability  

- **Snapshot is ready?**  
  - *Type:* If Node (`n8n-nodes-base.if`)  
  - *Role:* Checks if snapshot data contains valid property information or error (NodeApiError)  
  - *Configuration:* Evaluates existence of `items[0].abbreviatedAddress` and detects error strings  
  - *Connections:*  
    - True branch outputs (snapshot ready) - no further connection (end of wait loop)  
    - False branch triggers "Wait" node to retry after delay  
  - *Edge Cases:* False negatives if data format changes; infinite wait loops if snapshot never ready  

- **Wait**  
  - *Type:* Wait Node (`n8n-nodes-base.wait`)  
  - *Role:* Delays workflow to allow snapshot generation to complete before retrying content retrieval  
  - *Connections:* Loops back to "Recover Snapshot Content" node  
  - *Edge Cases:* Excessive delays; potential infinite loops if snapshot never becomes ready  

---

#### 1.4 Workflow Trigger and Control Logic

**Overview:**  
Handles execution triggers from external workflows or manual starts, supporting asynchronous snapshot retrieval and integration.

**Nodes Involved:**  
- When Executed by Another Workflow  

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger (`n8n-nodes-base.executeWorkflowTrigger`)  
  - *Role:* Entry point for external workflows triggering snapshot content retrieval  
  - *Configuration:* Input source set to passthrough  
  - *Connections:* Outputs directly to "Recover Snapshot Content" node  
  - *Edge Cases:* Missing or malformed external inputs; unauthorized workflow calls  

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                           | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                                                              |
|---------------------------|--------------------------------------|-----------------------------------------|----------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger                         | Entry point for user chat input         |                                  | Real Estate AI Agent            |                                                                                                                                          |
| Simple Memory1            | Memory Buffer Window                 | Maintains conversational context        |                                  | Real Estate AI Agent            |                                                                                                                                          |
| OpenAI Chat Model         | Language Model Chat (OpenAI GPT)    | Processes user language input            |                                  | Real Estate AI Agent            |                                                                                                                                          |
| Real Estate AI Agent      | LangChain Agent                     | Core AI logic and orchestration          | When chat message received, Simple Memory1, OpenAI Chat Model | Filter Dataset, Get Snapshot Content |                                                                                                                                          |
| Filter Dataset            | Bright Data Tool                    | Creates filtered real estate snapshot    | Real Estate AI Agent             | Real Estate AI Agent            | Add your Bright Data api key to "Filter Dataset" tool and "Recover Snapshot Content" node.                                               |
| Get Snapshot Content      | LangChain Tool Workflow             | Processes snapshot JSON into listing HTML | Real Estate AI Agent             | Real Estate AI Agent            | After pasting this workflow, update node "Get Snapshot Content" tool and add current Workflow ID.                                        |
| When Executed by Another Workflow | Execute Workflow Trigger          | External trigger for snapshot retrieval   |                                  | Recover Snapshot Content        |                                                                                                                                          |
| Recover Snapshot Content  | Bright Data API                    | Retrieves snapshot content by ID         | When Executed by Another Workflow, Wait | Snapshot is ready?             | Add your Bright Data api key to "Filter Dataset" tool and "Recover Snapshot Content" node.                                               |
| Snapshot is ready?        | If Node                            | Checks if snapshot content is ready       | Recover Snapshot Content         | Wait (if not ready)             |                                                                                                                                          |
| Wait                     | Wait Node                         | Delays before retrying snapshot retrieval | Snapshot is ready? (false branch) | Recover Snapshot Content        |                                                                                                                                          |
| Sticky Note1              | Sticky Note                       | Instructions and reminders for setup     |                                  |                               | # Real Estate AI Agent with Bright Data\n\n## TODO\n- After pasting this workflow, update node "Get Snapshot Content" tool and add current Workflow ID (for instance, if your workflow in n8n has the next url https://n8n-ai.cr.vps2.clients.killia.com/workflow/fjEIEQ1L6n2IKqlx your workflow Id is fjEIEQ1L6n2IKqlx).\n- Add your Bright Data api key to "Filter Dataset" tool and "Recover Snapshot Content" node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node:** *When chat message received*  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure as public webhook with welcome message:  
     `"Hi there! 👋\n\nMy name is Miquel and I am your personal Real Estate agent. I help you to find your new home.\n\nWhat are you looking for?"`  
   - Position: top-left for clarity  
   - Output: connect to "Real Estate AI Agent"  

2. **Create Node:** *Simple Memory1*  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set contextWindowLength to 30  
   - Connect output to "Real Estate AI Agent" as AI memory  

3. **Create Node:** *OpenAI Chat Model*  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Select model: "gpt-4o-mini"  
   - Add OpenAI API credentials (create in n8n credentials with your API key)  
   - Connect output to "Real Estate AI Agent" as AI language model  

4. **Create Node:** *Real Estate AI Agent*  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set input text to expression: `{{$json.chatInput}}`  
   - Set max iterations = 10  
   - Configure system message with detailed real estate instructions and filter rules:  
     - Use Zillow dataset metadata (city, bedrooms, bathrooms, price)  
     - Only call "Filter Dataset" if city, bedrooms, bathrooms, and price are provided  
     - Only one call to filter dataset allowed  
     - After filtering, call "Get Snapshot Content" with snapshot_id  
     - Format snapshot content into HTML listing with images  
   - Connect AI tool inputs to:  
     - "Filter Dataset" (as AI tool)  
     - "Get Snapshot Content" (as AI tool)  
   - Connect AI memory input from "Simple Memory1"  
   - Connect AI language model from "OpenAI Chat Model"  
   - Connect main input from "When chat message received"  

5. **Create Node:** *Filter Dataset*  
   - Type: `n8n-nodes-brightdata.brightDataTool`  
   - Set resource: "marketplaceDataset"  
   - Operation: "filterDataset"  
   - Dataset ID: set dynamically from AI input (expression mode)  
   - Filters group: set dynamically from AI input; always include filter `homeStatus = FOR_SALE`  
   - Records limit: 3  
   - Add Bright Data API credentials (create in n8n with your API key)  
   - Connect output to "Real Estate AI Agent" as AI tool response  

6. **Create Node:** *Get Snapshot Content*  
   - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Set workflowId to the ID of the sub-workflow that processes snapshot content (needs to be created or imported separately)  
   - Leave inputs empty or define according to sub-workflow requirements  
   - Connect output to "Real Estate AI Agent" as AI tool response  

7. **Create Node:** *When Executed by Another Workflow*  
   - Type: `n8n-nodes-base.executeWorkflowTrigger`  
   - Set input source to passthrough  
   - Connect output to "Recover Snapshot Content"  

8. **Create Node:** *Recover Snapshot Content*  
   - Type: `n8n-nodes-brightdata.brightData`  
   - Resource: "marketplaceDataset"  
   - Operation: "getSnapshotContent"  
   - Set snapshot_id from external workflow input (expression: `{{$json.query}}`)  
   - Add Bright Data API credentials  
   - Connect output to "Snapshot is ready?"  

9. **Create Node:** *Snapshot is ready?*  
   - Type: `n8n-nodes-base.if`  
   - Condition: Check if `items[0].abbreviatedAddress` exists and is not an error  
   - True branch: ends workflow or proceeds as needed  
   - False branch: connects to "Wait" node  

10. **Create Node:** *Wait*  
    - Type: `n8n-nodes-base.wait`  
    - Default wait time (use default or customize delay)  
    - Connect output back to "Recover Snapshot Content" to retry snapshot retrieval  

11. **Create Node:** *Sticky Note*  
    - Add notes for future users:  
      - Reminder to update "Get Snapshot Content" workflow ID after importing workflow  
      - Reminder to add Bright Data API keys to "Filter Dataset" and "Recover Snapshot Content" nodes  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| After pasting this workflow, update node "Get Snapshot Content" tool and add current Workflow ID (e.g., from https://n8n-ai.cr.vps2.clients.killia.com/workflow/fjEIEQ1L6n2IKqlx, workflow ID is fjEIEQ1L6n2IKqlx).                                                                                                      | Sticky Note in workflow                                                                                         |
| Add your Bright Data API key to "Filter Dataset" tool and "Recover Snapshot Content" node for API authentication and data access.                                                                                                                                                                                         | Sticky Note in workflow                                                                                         |
| This workflow combines OpenAI GPT-4o-mini with Bright Data marketplace datasets to provide filtered real estate listings based on user chat input with persistent conversational memory and asynchronous snapshot handling.                                                                                                | Workflow overview                                                                                               |
| The sub-workflow referenced in "Get Snapshot Content" node is critical for converting raw snapshot JSON into formatted HTML listings with images; ensure this sub-workflow is deployed and its ID is correctly set.                                                                                                       | Node description                                                                                                |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---