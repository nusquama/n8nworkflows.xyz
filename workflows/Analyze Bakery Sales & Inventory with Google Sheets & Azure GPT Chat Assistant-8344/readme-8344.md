Analyze Bakery Sales & Inventory with Google Sheets & Azure GPT Chat Assistant

https://n8nworkflows.xyz/workflows/analyze-bakery-sales---inventory-with-google-sheets---azure-gpt-chat-assistant-8344


# Analyze Bakery Sales & Inventory with Google Sheets & Azure GPT Chat Assistant

### 1. Workflow Overview

This workflow, titled **"Analyze Bakery Sales & Inventory with Google Sheets & Azure GPT Chat Assistant"**, is designed to provide an interactive AI-powered assistant for analyzing bakery sales and inventory data stored in Google Sheets. It enables users to send chat queries about bakery data, which the AI processes and answers concisely with actionable insights.

**Target Use Cases:**  
- Bakery managers or analysts querying sales and stock data interactively.  
- Real-time AI-driven data analysis and reporting without manual spreadsheet inspection.  
- Handling conversational context for ongoing discussions about bakery performance.

**Logical Blocks and Their Roles:**  

- **1.1 Input Reception**  
  - Node: *When chat message received*  
  - Role: Entry point triggered by user chat messages.

- **1.2 AI Processing Core**  
  - Nodes: *AI Agent*, *Azure OpenAI Chat Model*, *Simple Memory*  
  - Role: Processes user queries, maintains conversational memory, and generates AI-driven responses with a professional, concise data analysis focus.

- **1.3 Data Retrieval**  
  - Node: *Retrieve bakery data*  
  - Role: Connects to Google Sheets to fetch structured bakery sales and inventory data for analysis.

- **1.4 Documentation & Contextual Notes**  
  - Nodes: Multiple *Sticky Note* nodes  
  - Role: Provide descriptive documentation within the workflow for clarity and maintenance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from users and triggers the workflow when a message is received.

- **Nodes Involved:**  
  - *When chat message received*

- **Node Details:**  

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Technical Role: Webhook-based trigger node that activates the workflow on receiving a chat message.  
    - Configuration: Default parameters, no filters or options set.  
    - Expressions/Variables: None.  
    - Input: External chat message (webhook trigger).  
    - Output: Passes chat message data to the *AI Agent*.  
    - Version-Specific: Uses typeVersion 1.3, compatible with current LangChain node triggers.  
    - Edge Cases / Potential Failures: Webhook connectivity issues, malformed message payloads, or unauthorized access could cause failures.  
    - Sub-Workflow: None.

---

#### 1.2 AI Processing Core

- **Overview:**  
  This block interprets and processes user queries, maintains conversational context, and generates responses using an Azure GPT-based language model. It ensures professional, clear, and concise data analysis tailored to bakery sales and inventory.

- **Nodes Involved:**  
  - *AI Agent*  
  - *Simple Memory*  
  - *Azure OpenAI Chat Model*

- **Node Details:**  

  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Technical Role: Central AI engine coordinating reasoning and data analysis responses.  
    - Configuration:  
      - System message defines assistant behavior (professional data analyst specialized in Excel datasets).  
      - Instructions emphasize concise, actionable insights, no assumptions, and plain English.  
    - Expressions/Variables: System message hardcoded in parameters.  
    - Input: Receives chat trigger output and memory context.  
    - Output: Sends processed AI responses downstream.  
    - Version-Specific: Uses typeVersion 2.2, supporting advanced LangChain agent features.  
    - Edge Cases / Failures: Invalid or ambiguous user queries, failure to retrieve data, or logic errors in analysis.  
    - Sub-Workflow: None.

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Technical Role: Maintains conversation history to provide context across chat interactions.  
    - Configuration: Default buffer window parameters (not explicitly set here).  
    - Expressions/Variables: None.  
    - Input: Connected from *AI Agent* output (ai_memory).  
    - Output: Feeds memory context back to *AI Agent*.  
    - Version-Specific: typeVersion 1.3 compatible with LangChain memory buffer nodes.  
    - Edge Cases / Failures: Memory overflow if conversation is too long, or memory desynchronization.  
    - Sub-Workflow: None.

  - **Azure OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`  
    - Technical Role: Provides GPT-based language understanding and response generation backend.  
    - Configuration:  
      - Model: "gpt-5-mini" (an Azure-hosted GPT model variant).  
      - No extra options set, uses default parameters.  
    - Expressions/Variables: None.  
    - Input: Receives prompts from *AI Agent*.  
    - Output: Returns AI-generated text to *AI Agent*.  
    - Credentials: Uses Azure OpenAI API credential for authentication.  
    - Version-Specific: typeVersion 1, compatible with Azure OpenAI API in n8n.  
    - Edge Cases / Failures: API authentication errors, rate limits, network issues, or model unavailability.  
    - Sub-Workflow: None.

---

#### 1.3 Data Retrieval

- **Overview:**  
  This block fetches structured bakery sales and inventory data from a specific Google Sheets document, enabling the AI agent to perform data-driven analysis.

- **Nodes Involved:**  
  - *Retrieve bakery data*

- **Node Details:**  

  - **Retrieve bakery data**  
    - Type: `n8n-nodes-base.googleSheetsTool`  
    - Technical Role: Connects to Google Sheets, reads data from a specified sheet and document.  
    - Configuration:  
      - Document ID: `1dCCQzjoDZak-mQD1iyGd5aHKGFeh15fsBPUIoTgAYGw` (Bakery data spreadsheet)  
      - Sheet Name: "Full Month" (sheet ID 764145761)  
      - No filters or ranges specified, assumes full sheet read.  
    - Expressions/Variables: Document and sheet IDs configured via list mode with cached URLs for reference.  
    - Input: None (called as AI tool by *AI Agent*).  
    - Output: Provides structured bakery data in JSON format to *AI Agent*.  
    - Credentials: Uses Google Sheets OAuth2 credentials for API access.  
    - Version-Specific: typeVersion 4.7, latest stable Google Sheets node with advanced options.  
    - Edge Cases / Failures: OAuth token expiry, API quotas, spreadsheet access permissions, empty or malformed data.  
    - Sub-Workflow: None.

---

#### 1.4 Documentation & Contextual Notes

- **Overview:**  
  These nodes provide embedded documentation inside the workflow canvas explaining each primary node’s purpose and role, improving maintainability and onboarding.

- **Nodes Involved:**  
  - *Sticky Note*  
  - *Sticky Note1*  
  - *Sticky Note2*  
  - *Sticky Note3*  
  - *Sticky Note4*

- **Node Details:**  

  - All Sticky Notes are of type `n8n-nodes-base.stickyNote`, purely informational with no inputs or outputs.  
  - They include descriptive markdown content outlining workflow structure, node purposes, and links to the bakery data sheet.  
  - Positioned near relevant nodes for contextual clarity.  
  - No version requirements or failure modes.  
  - Content includes:  
    - Workflow title and description  
    - Node-specific purposes and instructions  
    - Link to the Google Sheets bakery dataset: https://docs.google.com/spreadsheets/d/1dCCQzjoDZak-mQD1iyGd5aHKGFeh15fsBPUIoTgAYGw/edit?usp=drivesdk

---

### 3. Summary Table

| Node Name                 | Node Type                                    | Functional Role                   | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                              |
|---------------------------|----------------------------------------------|---------------------------------|------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger         | Entry point for chat messages    | External webhook       | AI Agent                | ## Workflow: **Data Analytics for Bakery** - Node 1: Entry point triggered by user chat messages.                         |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent                | Core AI reasoning and response  | When chat message received, Simple Memory, Retrieve bakery data, Azure OpenAI Chat Model | Simple Memory           | Node 2: Central AI engine, professional concise data analysis assistant.                                                 |
| Simple Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow  | Maintains conversational memory | AI Agent                | AI Agent                | Node 3: Stores short-term conversation context for continuity.                                                           |
| Retrieve bakery data      | n8n-nodes-base.googleSheetsTool               | Fetches bakery data from Sheets | None                    | AI Agent                | Node 4: Connects to Google Sheets bakery dataset. Dataset link: https://docs.google.com/spreadsheets/d/1dCCQzjoDZak-mQD1iyGd5aHKGFeh15fsBPUIoTgAYGw/edit?usp=drivesdk |
| Azure OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi   | Provides GPT language model     | AI Agent                | AI Agent                | Node 5: GPT-based language backend ensuring professional tone and style.                                                 |
| Sticky Note               | n8n-nodes-base.stickyNote                      | Documentation                   | None                    | None                    | ## Workflow: **Data Analytics for Bakery** - Node 1: Entry point triggered by user chat messages.                         |
| Sticky Note1              | n8n-nodes-base.stickyNote                      | Documentation                   | None                    | None                    | Node 2: Central AI engine, professional concise data analysis assistant.                                                 |
| Sticky Note2              | n8n-nodes-base.stickyNote                      | Documentation                   | None                    | None                    | Node 3: Stores short-term conversation context for continuity.                                                           |
| Sticky Note3              | n8n-nodes-base.stickyNote                      | Documentation                   | None                    | None                    | Node 4: Connects to Google Sheets bakery dataset. Dataset link: https://docs.google.com/spreadsheets/d/1dCCQzjoDZak-mQD1iyGd5aHKGFeh15fsBPUIoTgAYGw/edit?usp=drivesdk |
| Sticky Note4              | n8n-nodes-base.stickyNote                      | Documentation                   | None                    | None                    | Node 5: GPT-based language backend ensuring professional tone and style.                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add a **When chat message received** node (`@n8n/n8n-nodes-langchain.chatTrigger`)  
   - Leave default parameters; this node acts as the webhook trigger for incoming chat messages.

2. **Add AI Agent Node**  
   - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`)  
   - Configure the `systemMessage` with the provided prompt specifying the assistant as a professional data analyst specialized in Excel datasets, emphasizing concise and actionable responses.  
   - Set typeVersion to 2.2.

3. **Add Simple Memory Node**  
   - Add a **Simple Memory** node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)  
   - Use default buffer window settings to maintain conversation context.  
   - Set typeVersion to 1.3.

4. **Add Google Sheets Data Retrieval Node**  
   - Add a **Google Sheets Tool** node (`n8n-nodes-base.googleSheetsTool`)  
   - Configure with:  
     - Document ID: `1dCCQzjoDZak-mQD1iyGd5aHKGFeh15fsBPUIoTgAYGw`  
     - Sheet Name: "Full Month" (Sheet ID 764145761)  
   - Authenticate using a configured **Google Sheets OAuth2** credential with read access.  
   - Set typeVersion to 4.7.

5. **Add Azure OpenAI Chat Model Node**  
   - Add **Azure OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`)  
   - Set the model to `"gpt-5-mini"` or equivalent Azure GPT model.  
   - Authenticate with valid **Azure OpenAI API** credentials.  
   - Set typeVersion to 1.

6. **Connect Nodes**  
   - Connect **When chat message received** → **AI Agent** (main output to main input).  
   - Connect **Simple Memory** output (`ai_memory`) → **AI Agent** input (`ai_memory`).  
   - Connect **Retrieve bakery data** output (`ai_tool`) → **AI Agent** input (`ai_tool`).  
   - Connect **Azure OpenAI Chat Model** output (`ai_languageModel`) → **AI Agent** input (`ai_languageModel`).  
   - Connect **AI Agent** output to **Simple Memory** input (`ai_memory`) to close the memory loop.

7. **Add Sticky Notes for Documentation** (Optional but recommended)  
   - Add sticky notes near each major node describing their purpose as per the content in the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Dataset for bakery sales and inventory used in this workflow is publicly accessible here: [Bakery Data Sheet](https://docs.google.com/spreadsheets/d/1dCCQzjoDZak-mQD1iyGd5aHKGFeh15fsBPUIoTgAYGw/edit?usp=drivesdk) | Google Sheets bakery dataset                                                                                      |
| The AI Agent uses a system prompt to enforce a professional, concise, and user-focused data analysis style without guessing or overextending beyond user queries. | Workflow’s AI interaction design                                                                                  |
| Azure OpenAI Chat Model node requires Azure OpenAI API credentials with access to GPT models; ensure valid subscription and quota.                              | Azure OpenAI API documentation: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/                |
| Google Sheets OAuth2 credentials must have permission to read the specified spreadsheet; ensure sharing and API scopes are correctly configured.                | Google Sheets API docs: https://developers.google.com/sheets/api/guides/authorizing                                   |

---

This completes the comprehensive technical reference document for the workflow **"Analyze Bakery Sales & Inventory with Google Sheets & Azure GPT Chat Assistant"**. It supports detailed understanding, maintenance, and manual reconstruction without external dependencies.