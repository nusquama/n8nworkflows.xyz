AI Website Chatbot with CRM Lead Collection using GPT and Google Sheets

https://n8nworkflows.xyz/workflows/ai-website-chatbot-with-crm-lead-collection-using-gpt-and-google-sheets-7227


# AI Website Chatbot with CRM Lead Collection using GPT and Google Sheets

### 1. Workflow Overview

This workflow implements an **AI-powered website chatbot integrated with CRM lead collection**, designed to engage website visitors, gather their contact details and automation needs, and store these leads in a Google Sheets-based CRM. It uses GPT-based AI agents for conversational interactions and data extraction, integrating AI with automated spreadsheet updates.

The workflow is logically divided into two main blocks:

- **1.1 Sub-Workflow: CRM Lead Processing**  
  A dedicated sub-workflow triggered by the main chatbot workflow. It receives raw lead data, cleans and formats it into structured JSON, appends it to a Google Sheet acting as a CRM, and sends confirmation feedback.

- **1.2 Main Workflow: Website Chatbot with AI Agent**  
  This is the primary interactive chatbot workflow. It listens to chat messages, runs the AI agent configured with a system prompt describing the company services and lead capture goals, maintains conversation context, and invokes the CRM sub-workflow to save leads.

---

### 2. Block-by-Block Analysis

#### 2.1 Sub-Workflow: CRM Lead Processing

**Overview:**  
This sub-workflow handles incoming lead data from the chatbot, extracts structured information (email and description), stores it in a Google Sheets CRM, and returns a confirmation message.

**Nodes Involved:**  
- When Executed by Another Workflow  
- OpenAI Chat Model4  
- Convert Conversation  
- Structured Output Parser  
- Append row in sheet (Google Sheets)  
- Code  

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point for this sub-workflow, triggered by the main workflow’s Tool Workflow node.  
  - *Configuration:* Passes through input data unchanged.  
  - *Connections:* Outputs to "Convert Conversation" node.  
  - *Failure cases:* Workflow trigger failure rare; ensure input data is valid.  

- **OpenAI Chat Model4**  
  - *Type:* AI Language Model (OpenAI GPT)  
  - *Role:* Provides AI processing for the Convert Conversation node.  
  - *Config:* Uses model "gpt-4o-mini" with default options.  
  - *Credentials:* OpenAI API credential configured.  
  - *Connections:* Connected to "Convert Conversation" via ai_languageModel.  
  - *Failure cases:* API key issues, quota limits, or network errors can cause failures.  

- **Convert Conversation**  
  - *Type:* Langchain Agent Node  
  - *Role:* Formats incoming raw text query into clean JSON containing "email" and "description".  
  - *Config:* Uses a system message instructing the agent to output only JSON with keys "email" and "description".  
  - *Key Expression:* `=query {{ $json.query }}` — extracts the query field from input JSON.  
  - *Connections:* Outputs main to "Append row in sheet" and ai_outputParser to "Structured Output Parser".  
  - *Failure cases:* Incorrect input format, incomplete parsing, or unexpected AI output could cause failures.  

- **Structured Output Parser**  
  - *Type:* Output Parser Structured  
  - *Role:* Enforces the JSON schema for output (email, description).  
  - *Config:* Example JSON schema provided to validate AI output.  
  - *Connections:* Feeds parsed output back to "Convert Conversation" node (for validation or further processing).  
  - *Failure cases:* Parsing errors if AI output deviates from schema.  

- **Append row in sheet (Google Sheets)**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends the structured lead data (email, description) as a new row into a configured Google Sheet.  
  - *Config:*  
    - Operation: Append  
    - Document ID: Google Sheet ID for the CRM spreadsheet  
    - Sheet Name: gid=0 (default tab)  
    - Columns mapped to JSON fields `output.email` and `output.description`  
  - *Credentials:* Google Sheets OAuth2 account configured.  
  - *Connections:* Outputs main to "Code" node.  
  - *Failure cases:* Authentication errors, API limits, incorrect sheet setup, or invalid data could cause failure.  

- **Code**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Returns a simple confirmation message `"Thanks for the info, we will be in touch soon"`.  
  - *Config:* Hardcoded JSON response with text field.  
  - *Connections:* Endpoint of the sub-workflow chain.  
  - *Failure cases:* Minimal risk; code execution errors possible if modified.  

---

#### 2.2 Main Workflow: Website Chatbot with AI Agent

**Overview:**  
This workflow listens for chat messages from website visitors, runs an AI agent configured with a system prompt to engage visitors, ask about their automation needs, collect name/email, and upon gathering data, invokes the CRM sub-workflow to save leads.

**Nodes Involved:**  
- When chat message received  
- Website Chatbot (Agent Node)  
- OpenAI Chat Model3  
- Simple Memory3  
- crm tool (Tool Workflow)  

**Node Details:**

- **When chat message received**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Entry trigger for the chatbot workflow when a visitor sends a chat message.  
  - *Config:* Default webhook with unique webhookId.  
  - *Connections:* Outputs main to "Website Chatbot" node.  
  - *Failure cases:* Webhook misconfiguration or network issues could prevent trigger.  

- **Website Chatbot (Agent Node)**  
  - *Type:* Langchain Agent  
  - *Role:* Main AI conversational agent handling visitor messages.  
  - *Config:* System message defines the agent’s role:  
    - Introduce consulting firm services briefly  
    - Ask visitor about processes to automate  
    - Collect visitor name and email  
    - Confirm info and thank visitor on collecting email  
    - Pass collected info to CRM tool sub-workflow  
  - *Connections:*  
    - ai_languageModel from "OpenAI Chat Model3"  
    - ai_memory from "Simple Memory3"  
    - ai_tool to "crm tool" (sub-workflow)  
  - *Failure cases:* AI response errors, incomplete data collection, or failure to trigger sub-workflow.  

- **OpenAI Chat Model3**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Provides GPT-4o-mini model for the chatbot agent.  
  - *Config:* Model set to "gpt-4o-mini", default options.  
  - *Credentials:* OpenAI API key configured.  
  - *Connections:* ai_languageModel to "Website Chatbot".  
  - *Failure cases:* API quota, auth failure, or network issues.  

- **Simple Memory3**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains short-term conversation context to enable coherent chatbot replies.  
  - *Config:* Default buffer window memory.  
  - *Connections:* ai_memory to "Website Chatbot".  
  - *Failure cases:* Memory overflow unlikely; improper context handling possible if misconfigured.  

- **crm tool (Tool Workflow)**  
  - *Type:* Langchain Tool Workflow  
  - *Role:* Invokes the CRM sub-workflow to process and store collected lead info.  
  - *Config:*  
    - Source set to "parameter"  
    - Sub-workflow JSON embedded directly, containing the CRM lead processing logic described above.  
    - Description: "crm tool to store lead information" (helps AI agent know when to call).  
  - *Connections:* ai_tool to "Website Chatbot".  
  - *Failure cases:* Sub-workflow errors propagate here; incorrect or incomplete parameter passing may cause failure.  

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                                     | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                              |
|---------------------------|---------------------------------|----------------------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger          | Starts main chatbot workflow on visitor message    |                                | Website Chatbot               |                                                                                                        |
| Website Chatbot            | Langchain Agent                 | Main AI conversational agent with system prompt   | When chat message received, OpenAI Chat Model3, Simple Memory3, crm tool |                               | "Create the Main Workflow (Website Chatbot)" note explains purpose and setup                           |
| OpenAI Chat Model3         | Langchain OpenAI Chat Model    | Provides GPT model to chatbot agent                 |                                | Website Chatbot (ai_languageModel) |                                                                                                        |
| Simple Memory3             | Langchain Memory Buffer Window | Maintains short-term chat context                    |                                | Website Chatbot (ai_memory)   |                                                                                                        |
| crm tool                  | Langchain Tool Workflow         | Calls CRM sub-workflow to store lead data          |                                | Website Chatbot (ai_tool)     | "Create the Main Workflow (Website Chatbot)" note explains how to connect sub-workflow                  |
| When Executed by Another Workflow | Execute Workflow Trigger  | Entry point for CRM sub-workflow                     |                                | Convert Conversation          | "Create the Sub-Workflow (CRM Tool)" note details this sub-workflow                                    |
| OpenAI Chat Model4         | Langchain OpenAI Chat Model    | Provides GPT model for parsing conversation data    |                                | Convert Conversation (ai_languageModel) |                                                                                                        |
| Convert Conversation       | Langchain Agent                | Extracts and formats lead info into JSON            | When Executed by Another Workflow, OpenAI Chat Model4, Structured Output Parser | Append row in sheet           | "Create the Sub-Workflow (CRM Tool)" note details node role                                            |
| Structured Output Parser   | Output Parser Structured       | Validates AI output matches JSON schema             |                                | Convert Conversation (ai_outputParser) |                                                                                                        |
| Append row in sheet        | Google Sheets                  | Appends lead data to CRM Google Sheet                | Convert Conversation            | Code                         | "Create the Sub-Workflow (CRM Tool)" note details Google Sheets setup                                  |
| Code                      | Code Node                     | Sends confirmation message after data append        | Append row in sheet            |                               |                                                                                                        |
| Sticky Note3               | Sticky Note                   | Instructions for main workflow creation              |                                |                               | Contains detailed instructions for main workflow creation                                             |
| Sticky Note5               | Sticky Note                   | Instructions for sub-workflow creation               |                                |                               | Contains detailed instructions for sub-workflow creation                                              |
| Sticky Note17              | Sticky Note                   | Contact info for support and customization           |                                |                               | Contains author contact email and LinkedIn link                                                      |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create the Sub-Workflow (CRM Tool)**

1. Create a new workflow named e.g., "CRM Lead Processing".
2. Add node "When Executed by Another Workflow" (Execute Workflow Trigger). Set input source to "passthrough".
3. Add "OpenAI Chat Model" node:
   - Set model to "gpt-4o-mini".
   - Configure OpenAI API credentials.
4. Add "Convert Conversation" node (Langchain Agent):
   - Set prompt type "define".
   - Set text expression: `=query {{ $json.query }}`.
   - System message: instruct agent to output clean JSON with keys `email` and `description`.
   - Enable output parser.
5. Connect "When Executed by Another Workflow" main output to "Convert Conversation" main input.
6. Connect "OpenAI Chat Model" ai_languageModel output to "Convert Conversation" ai_languageModel input.
7. Add "Structured Output Parser":
   - Set example JSON schema with fields `email` and `description`.
8. Connect "Structured Output Parser" ai_outputParser output to "Convert Conversation" ai_outputParser input.
9. Add "Append row in sheet" node (Google Sheets):
   - Operation: append
   - Document ID: Use your Google Sheet ID (e.g., copy from https://docs.google.com/spreadsheets/d/1Lr54_U6khrE8jjU-rocf0N1Phe7ZIovLppTjBzIraZk)
   - Sheet Name: Use the correct sheet tab (e.g., gid=0)
   - Columns: Map "email" to `={{ $json.output.email }}`, "description" to `={{ $json.output.description }}`
   - Connect Google Sheets OAuth2 credentials.
10. Connect "Convert Conversation" main output to "Append row in sheet" main input.
11. Add a "Code" node:
    - JavaScript code:  
      ```
      return [{ json: { text: "Thanks for the info, we will be in touch soon" } }];
      ```
12. Connect "Append row in sheet" main output to "Code" main input.

**Step 2: Create the Main Workflow (Website Chatbot)**

1. Create a new workflow named e.g., "AI Website Chatbot".
2. Add "When chat message received" node (Langchain Chat Trigger). Configure your chatbot integration webhook.
3. Add "OpenAI Chat Model" node:
   - Model: "gpt-4o-mini"
   - Configure OpenAI API credentials.
4. Add "Simple Memory" node (Langchain Memory Buffer Window), default settings.
5. Add "Website Chatbot" node (Langchain Agent):
   - System message:  
     ```
     You are the first point of contact for visitors on our website. We are a consulting firm that helps companies automate their internal processes using n8n, an open-source workflow automation platform.

     Keep answers brief.

     Our services include:

     Designing and implementing automations using n8n

     Replacing manual work with fully automated workflows

     Training teams to manage and scale automations in-house

     Your primary goals are:

     Briefly explain what we do in a helpful, conversational tone.

     Ask the visitor what kinds of processes they’re hoping to automate or what challenges they’re facing.

     Collect their name and email address so someone from our team can follow up.

     Be friendly, curious, and professional. If the visitor shares their contact information, confirm it and thank them — and pass the details to our CRM workflow.

     After you have the email address, and what the user needs. send all the info all together to the crm tool.
     ```
6. Add a "Tool Workflow" node:
   - Source: Parameter
   - Paste the entire JSON of the CRM sub-workflow created in Step 1 (or select the saved workflow version).
   - Description: "crm tool to store lead information".
7. Connect nodes as follows:
   - "When chat message received" main output → "Website Chatbot" main input
   - "OpenAI Chat Model" ai_languageModel output → "Website Chatbot" ai_languageModel input
   - "Simple Memory" ai_memory output → "Website Chatbot" ai_memory input
   - "Tool Workflow" ai_tool output → "Website Chatbot" ai_tool input
8. Configure credentials:
   - OpenAI API credentials on both OpenAI nodes.
   - Google Sheets OAuth2 credentials on the Google Sheets node inside the sub-workflow.
9. Activate both workflows.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                               |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Copy this Google Sheet to use as CRM: https://docs.google.com/spreadsheets/d/1Lr54_U6khrE8jjU-rocf0N1Phe7ZIovLppTjBzIraZk/edit?usp=drivesdk | Required for Google Sheets CRM integration                     |
| Enable Google Sheets API in your Google Cloud Console and link credentials for n8n Google Sheets OAuth2 | Setup prerequisite for Google Sheets node                      |
| Contact for help or customization: robert@ynteractive.com                                           | Author contact                                                 |
| LinkedIn profile: https://www.linkedin.com/in/robert-breen-29429625/                                | Author LinkedIn                                                |
| Sticky notes in workflow provide detailed step-by-step setup instructions for both workflows         | Refer to "Sticky Note3" and "Sticky Note5" in the workflow UI |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. It complies with all content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.