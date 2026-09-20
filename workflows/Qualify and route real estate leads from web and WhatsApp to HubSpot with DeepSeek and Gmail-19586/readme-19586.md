Qualify and route real estate leads from web and WhatsApp to HubSpot with DeepSeek and Gmail

https://n8nworkflows.xyz/workflows/qualify-and-route-real-estate-leads-from-web-and-whatsapp-to-hubspot-with-deepseek-and-gmail-19586


# Qualify and route real estate leads from web and WhatsApp to HubSpot with DeepSeek and Gmail

### 1. Workflow Overview

This workflow is designed to automate real estate lead management and conversion. It ingests incoming leads from multiple channels, normalizes the data, screens for duplicates in HubSpot, qualifies the lead using DeepSeek AI, rotates lead assignment among available agents via Google Sheets (round-robin style), records the contact in HubSpot, and finally dispatches automated notifications to both the assigned sales agent and the lead.

The logic is divided into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming webhooks from web forms, WhatsApp, and Facebook Lead Ads, then standardizes the payload into a consistent format.
- **1.2 Duplicate Detection:** Queries HubSpot to determine if the lead already exists, flagging duplicates to halt redundant processing.
- **1.3 AI Qualification:** Passes lead details to a DeepSeek LLM via an AI Agent node to extract intent, priority, and budget signals, parsing the structured output.
- **1.4 Agent Rotation & Claiming:** Reads agent availability from Google Sheets, sorts by rotation history (round-robin), claims the slot for the selected agent, and handles fallback logic if no agent is available.
- **1.5 CRM Recording & Notifications:** Creates or updates the lead record inside HubSpot, emails the assigned agent with lead details, and sends a booking link (or general confirmation) to the lead.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Acts as the multi-channel gateway for the pipeline, standardizing divergent payloads (web forms, WhatsApp APIs, and Facebook Lead Ads) into a unified internal data schema.
- **Nodes Involved:** 
  - `Capture Webform Leads`
  - `Capture WhatsApp Leads`
  - `Capture Facebook Leads`
  - `Normalize Incoming Leads`
- **Node Details:**
  - **Capture Webform Leads**
    - Type: `n8n-nodes-base.webhook` (v2.1)
    - Role: Listens for incoming HTTP POST requests from website contact forms.
    - Config: Path `re-lead-webform`, method `POST`.
    - Connections: Output connects to `Normalize Incoming Leads`.
  - **Capture WhatsApp Leads**
    - Type: `n8n-nodes-base.webhook` (v2.1)
    - Role: Listens for incoming WhatsApp message webhooks.
    - Config: Path `re-lead-whatsapp`, method `POST`.
    - Connections: Output connects to `Normalize Incoming Leads`.
  - **Capture Facebook Leads**
    - Type: `n8n-nodes-base.webhook` (v2.1)
    - Role: Listens for Facebook Lead Ads webhook payloads.
    - Config: Path `re-lead-facebook`, method `POST`.
    - Connections: Output connects to `Normalize Incoming Leads`.
  - **Normalize Incoming Leads**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Evaluates payload structure to extract name, phone, email, message, and source, normalizing phone numbers to the last 10 digits.
    - Key Expressions: Evaluates `body.entry`, `body.messages`, or falls back to generic web form properties.
    - Connections: Inputs from all three webhooks; output connects to `Find Duplicate Leads in HubSpot`.
    - Edge Cases: Malformed payloads will result in empty string values for missing fields; phone regex cleanup handles non-numeric characters safely.

#### 2.2 Duplicate Detection
- **Overview:** Queries HubSpot CRM to prevent processing duplicate inquiries based on the normalized phone number.
- **Nodes Involved:**
  - `Find Duplicate Leads in HubSpot`
  - `Flag Duplicate Leads`
  - `Check for Duplicates`
  - `Log and Skip Duplicates`
- **Node Details:**
  - **Find Duplicate Leads in HubSpot**
    - Type: `n8n-nodes-base.hubspot` (v2.2)
    - Role: Searches HubSpot contacts matching the caller's normalized phone number.
    - Config: Operation `search`, authentication via `appToken`, limit `1`.
    - Key Expressions: Query set to `={{$json.normalizedPhone}}`.
    - Connections: Input from `Normalize Incoming Leads`; output connects to `Flag Duplicate Leads`.
    - Credentials: HubSpot Service Key account.
  - **Flag Duplicate Leads**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Compares search results to establish an `isDuplicate` boolean flag.
    - Connections: Input from HubSpot search; output connects to `Check for Duplicates`.
  - **Check for Duplicates**
    - Type: `n8n-nodes-base.if` (v2.3)
    - Role: Branching node that routes leads based on whether they already exist in the CRM.
    - Key Expressions: Evaluates `={{$json.isDuplicate}}` equals `true`.
    - Connections: True branch connects to `Log and Skip Duplicates`; false branch connects to `Qualify Leads with AI`.
  - **Log and Skip Duplicates**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Terminates the duplicate branch gracefully by outputting a skip log record.
    - Connections: Input from `Check for Duplicates` (True branch); terminal node.

#### 2.3 AI Qualification
- **Overview:** Leverages DeepSeek AI to evaluate lead viability, extracting priority, intent, lead type, and budget signals from unstructured text.
- **Nodes Involved:**
  - `Qualify Leads with AI`
  - `DeepSeek AI Chat Model`
  - `Extract AI Insights`
- **Node Details:**
  - **Qualify Leads with AI**
    - Type: `@n8n/n8n-nodes-langchain.agent` (v3.1)
    - Role: LangChain agent that analyzes lead data using a system prompt and scoring guide.
    - Config: Prompt defines JSON output format requirements (`leadType`, `priority`, `budgetSignal`, `intentSummary`).
    - Connections: Connected to `DeepSeek AI Chat Model` via AI language model connection; output connects to `Extract AI Insights`.
  - **DeepSeek AI Chat Model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatDeepSeek` (v1)
    - Role: Provides the DeepSeek chat model implementation to the LangChain agent.
    - Credentials: DeepSeek account.
  - **Extract AI Insights**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Cleans markdown wrappers from the LLM output, parses the JSON string, and merges insights with lead data.
    - Connections: Input from AI Agent; output connects to `Fetch Available Agents`.
    - Edge Cases: Includes a `try/catch` fallback block returning default `general` / `LOW` values if the AI output contains invalid JSON.

#### 2.4 Agent Rotation & Claiming
- **Overview:** Queries Google Sheets for active agents, identifies the user with the oldest assignment timestamp (round-robin), claims the slot, and prepares routing properties.
- **Nodes Involved:**
  - `Fetch Available Agents`
  - `Organize Agent Information`
  - `Determine Agent Availability`
  - `Create Agent Claim Data`
  - `Update Agent Slot in Sheets`
- **Node Details:**
  - **Fetch Available Agents**
    - Type: `n8n-nodes-base.googleSheets` (v4.7)
    - Role: Reads the "Agents" spreadsheet tab to fetch active agent records.
    - Config: Document ID and Sheet Name (`Agents`).
    - Credentials: Google Sheets account.
    - Connections: Output connects to `Organize Agent Information`.
  - **Organize Agent Information**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Filters valid rows, sorts agents ascending by `LastAssignedAt` timestamp, and selects the top candidate.
    - Connections: Input from Google Sheets; output connects to `Determine Agent Availability`.
  - **Determine Agent Availability**
    - Type: `n8n-nodes-base.if` (v2.3)
    - Role: Checks if a valid agent was returned from the sheet.
    - Key Expressions: Evaluates `={{$json.hasAgent}}` equals `true`.
    - Connections: True branch connects to `Create Agent Claim Data`; false branch bypasses claiming and connects directly to `Assemble Lead Data`.
  - **Create Agent Claim Data**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Prepares payload updating the selected agent's `LastAssignedAt` property with the current ISO timestamp.
    - Connections: Input from `Determine Agent Availability` (True branch); output connects to `Update Agent Slot in Sheets`.
  - **Update Agent Slot in Sheets**
    - Type: `n8n-nodes-base.googleSheets` (v4.7)
    - Role: Writes the updated `LastAssignedAt` value back to the Agents sheet row.
    - Config: Operation `update`, matching by column `Name`.
    - Credentials: Google Sheets account.
    - Connections: Output connects to `Assemble Lead Data`.

#### 2.5 CRM Recording & Notifications
- **Overview:** Creates or updates the contact record in HubSpot with standard fields and AI/assignment custom properties, then sends notification emails to both the assigned agent and the lead.
- **Nodes Involved:**
  - `Assemble Lead Data`
  - `Add Lead to HubSpot CRM`
  - `Compose Agent Notification`
  - `Email Assigned Agent`
  - `Draft Booking Email`
  - `Email Booking Link to Lead`
- **Node Details:**
  - **Assemble Lead Data**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Normalizes contact names, generates fallback placeholder emails if missing, and sets lead status strings based on agent availability.
    - Connections: Inputs merge from `Determine Agent Availability` (false branch) and `Update Agent Slot in Sheets`; output connects to `Add Lead to HubSpot CRM`.
  - **Add Lead to HubSpot CRM**
    - Type: `n8n-nodes-base.hubspot` (v2.2)
    - Role: Creates or updates the contact in HubSpot with standard fields and custom properties.
    - Config: Maps first name, last name, phone, email, and custom properties (`lead_type`, `priority`, `budget_signal`, `intent_summary`, `assigned_agent_name`, `assigned_agent_email`, `lead_status`).
    - Credentials: HubSpot Service Key account.
    - Connections: Output fans out to `Compose Agent Notification` and `Draft Booking Email`.
  - **Compose Agent Notification**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Drafts the internal email text body and subject line for the assigned agent containing lead summary details.
    - Connections: Input from HubSpot CRM creation; output connects to `Email Assigned Agent`.
  - **Email Assigned Agent**
    - Type: `n8n-nodes-base.gmail` (v2.2)
    - Role: Sends the qualification summary email to the assigned sales agent.
    - Config: `onError` set to `continueRegularOutput`.
    - Credentials: Gmail account.
    - Connections: Input from notification composer; terminal node.
  - **Draft Booking Email**
    - Type: `n8n-nodes-base.code` (v2)
    - Role: Formulates the response email to the lead, embedding the agent's booking link (`CalComLink`) if available or a fallback message if unassigned.
    - Connections: Input from HubSpot CRM creation; output connects to `Email Booking Link to Lead`.
  - **Email Booking Link to Lead**
    - Type: `n8n-nodes-base.gmail` (v2.2)
    - Role: Sends the booking link or general confirmation email to the lead.
    - Config: `onError` set to `continueRegularOutput`.
    - Credentials: Gmail account.
    - Connections: Input from booking email drafter; terminal node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Overview and setup documentation | None | None | ## Real Estate Lead Management & Conversion — Core Pipeline<br><br>### How it works<br><br>This workflow captures real estate leads from web forms, WhatsApp, and Facebook Lead Ads, normalizes the payload, and checks HubSpot for duplicates before continuing. New leads are qualified by an AI agent, matched to an available sales agent from Google Sheets, optionally claim that agent’s slot, and are then created in HubSpot. After CRM creation, it sends separate email notifications to the assigned agent and to the lead with booking details.<br><br>### Setup steps<br><br>- Configure the three webhook entry points for the web form, WhatsApp integration, and Facebook Lead Ads source.<br>- Connect HubSpot credentials and ensure the duplicate search and lead creation fields match your CRM contact or deal schema.<br>- Configure the DeepSeek chat model credentials used by the Lead Qualification Agent.<br>- Connect Google Sheets credentials and prepare the agent availability sheet with the columns expected by the search and claim steps.<br>- Connect Gmail credentials and verify sender settings, recipient fields, and email content generated by the message-building code nodes.<br><br>### Customization<br><br>Adjust the normalization code for each lead source, the duplicate-matching criteria, the AI qualification prompt/scoring logic, agent assignment rules, and the booking/notification email templates. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Capture and normalize documentation | None | None | ## Capture and normalize leads<br><br>Collects incoming leads from the three channel webhooks and converts them into a consistent lead payload for downstream processing. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Duplicate check documentation | None | None | ## Check for duplicates<br><br>Searches HubSpot for an existing lead, computes a duplicate flag, and routes the workflow based on whether the lead already exists. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Stop duplicate documentation | None | None | ## Stop duplicate leads<br><br>Logs duplicate leads and ends the duplicate branch without creating a new CRM record or sending follow-up messages. |
| Sticky Note4 | n8n-nodes-base.stickyNote | AI qualification documentation | None | None | ## Qualify with AI<br><br>Uses the AI qualification agent and its DeepSeek chat model to assess the lead, then parses the model output into structured data. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Agent lookup documentation | None | None | ## Find available agent<br><br>Looks up available real estate agents in Google Sheets, prepares assignment details, and checks whether a suitable agent was found. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Claim slot documentation | None | None | ## Claim agent slot<br><br>For leads with an available agent, builds the claim payload and updates Google Sheets to reserve that agent’s slot. |
| Sticky Note7 | n8n-nodes-base.stickyNote | CRM creation documentation | None | None | ## Create CRM lead<br><br>Prepares the final lead record, including assignment data when present, and creates the lead in HubSpot. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Agent email notification documentation | None | None | ## Notify assigned agent<br><br>Builds an internal notification message and emails the assigned agent about the new qualified lead. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Booking email documentation | None | None | ## Send booking email<br><br>Creates the lead-facing booking message and emails the lead with the booking link or next-step instructions. |
| Capture Webform Leads | n8n-nodes-base.webhook | Receive web form submissions | None | Normalize Incoming Leads | |
| Capture WhatsApp Leads | n8n-nodes-base.webhook | Receive WhatsApp message webhooks | None | Normalize Incoming Leads | |
| Capture Facebook Leads | n8n-nodes-base.webhook | Receive Facebook Lead Ads webhooks | None | Normalize Incoming Leads | |
| Normalize Incoming Leads | n8n-nodes-base.code | Normalize payloads and clean phone numbers | Capture Webform Leads, Capture WhatsApp Leads, Capture Facebook Leads | Find Duplicate Leads in HubSpot | |
| Find Duplicate Leads in HubSpot | n8n-nodes-base.hubspot | Search HubSpot contacts by phone number | Normalize Incoming Leads | Flag Duplicate Leads | |
| Flag Duplicate Leads | n8n-nodes-base.code | Determine duplicate existence flag | Find Duplicate Leads in HubSpot | Check for Duplicates | |
| Check for Duplicates | n8n-nodes-base.if | Route based on duplicate status | Flag Duplicate Leads | Log and Skip Duplicates, Qualify Leads with AI | |
| Log and Skip Duplicates | n8n-nodes-base.code | Terminate duplicate execution path | Check for Duplicates | None | |
| Qualify Leads with AI | @n8n/n8n-nodes-langchain.agent | AI Agent evaluating lead viability | Check for Duplicates | Extract AI Insights | |
| Extract AI Insights | n8n-nodes-base.code | Parse LLM output into structured data | Qualify Leads with AI | Fetch Available Agents | |
| Fetch Available Agents | n8n-nodes-base.googleSheets | Read active agents from spreadsheet | Extract AI Insights | Organize Agent Information | |
| Organize Agent Information | n8n-nodes-base.code | Sort agents by rotation and pick top match | Fetch Available Agents | Determine Agent Availability | |
| Determine Agent Availability | n8n-nodes-base.if | Check if an available agent exists | Organize Agent Information | Create Agent Claim Data, Assemble Lead Data | |
| Create Agent Claim Data | n8n-nodes-base.code | Build timestamp update payload | Determine Agent Availability | Update Agent Slot in Sheets | |
| Update Agent Slot in Sheets | n8n-nodes-base.googleSheets | Update agent `LastAssignedAt` timestamp | Create Agent Claim Data | Assemble Lead Data | |
| Assemble Lead Data | n8n-nodes-base.code | Build final contact record and status | Update Agent Slot in Sheets, Determine Agent Availability | Add Lead to HubSpot CRM | |
| Add Lead to HubSpot CRM | n8n-nodes-base.hubspot | Create contact in HubSpot with properties | Assemble Lead Data | Compose Agent Notification, Draft Booking Email | |
| Compose Agent Notification | n8n-nodes-base.code | Draft notification message for agent | Add Lead to HubSpot CRM | Email Assigned Agent | |
| Email Assigned Agent | n8n-nodes-base.gmail | Email assigned agent lead details | Compose Agent Notification | None | |
| DeepSeek AI Chat Model | @n8n/n8n-nodes-langchain.lmChatDeepSeek | DeepSeek LLM provider for AI agent | None | Qualify Leads with AI | |
| Draft Booking Email | n8n-nodes-base.code | Draft lead booking or confirmation email | Add Lead to HubSpot CRM | Email Booking Link to Lead | |
| Email Booking Link to Lead | n8n-nodes-base.gmail | Email booking link or notice to lead | Draft Booking Email | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually inside n8n:

1. **Create Webhook Entry Points:**
   - Add three **Webhook** nodes named `Capture Webform Leads`, `Capture WhatsApp Leads`, and `Capture Facebook Leads`. Set HTTP method to `POST` with respective paths: `re-lead-webform`, `re-lead-whatsapp`, and `re-lead-facebook`.
2. **Add Normalization Logic:**
   - Create a **Code** node named `Normalize Incoming Leads`. Connect all three webhooks to this node. Paste JavaScript parsing logic to check for Facebook (`body.entry`), WhatsApp (`body.messages`), or standard web form fields, and standardize phone numbers.
3. **Configure Duplicate Detection:**
   - Add a **HubSpot** node named `Find Duplicate Leads in HubSpot` (operation: `search`, limit `1`). Map query to `={{$json.normalizedPhone}}`. Connect credentials (`HubSpot Service Key account`).
   - Add a **Code** node named `Flag Duplicate Leads` to evaluate search results against whether a contact record ID exists.
   - Add an **If** node named `Check for Duplicates` to evaluate `={{$json.isDuplicate}}`.
   - Connect the True branch to a **Code** node named `Log and Skip Duplicates` to terminate duplicate processing.
4. **Set Up AI Qualification:**
   - Connect the False branch of `Check for Duplicates` to an **AI Agent** node named `Qualify Leads with AI` (promptType: define).
   - Add a **DeepSeek Chat Model** node and link it to the AI Agent via the `ai_languageModel` connection. Configure with DeepSeek API credentials.
   - Add a **Code** node named `Extract AI Insights` to clean and parse the LLM JSON response.
5. **Implement Agent Rotation & Sheets Claim:**
   - Add a **Google Sheets** node named `Fetch Available Agents` (operation: `read`, sheet: `Agents`). Connect Google Sheets OAuth2 credentials.
   - Add a **Code** node named `Organize Agent Information` to filter rows and sort ascending by `LastAssignedAt`.
   - Add an **If** node named `Determine Agent Availability` evaluating `={{$json.hasAgent}}`.
   - For the True branch, add a **Code** node named `Create Agent Claim Data` to prepare an updated ISO timestamp, then connect it to a **Google Sheets** node named `Update Agent Slot in Sheets` (operation: `update`, matching by column `Name`).
6. **Build CRM & Notification Pipeline:**
   - Merge the output of `Update Agent Slot in Sheets` and the False branch of `Determine Agent Availability` into a **Code** node named `Assemble Lead Data`.
   - Add a **HubSpot** node named `Add Lead to HubSpot CRM` to create the contact. Map standard fields and custom properties (`lead_type`, `priority`, `budget_signal`, `intent_summary`, `assigned_agent_name`, `assigned_agent_email`, `lead_status`).
   - Connect CRM creation output to two parallel branches:
     - **Branch A (Agent Notification):** Add a **Code** node (`Compose Agent Notification`) followed by a **Gmail** node (`Email Assigned Agent`) using Gmail OAuth2 credentials.
     - **Branch B (Lead Booking Email):** Add a **Code** node (`Draft Booking Email`) followed by a **Gmail** node (`Email Booking Link to Lead`) using Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets Template Reference | `https://docs.google.com/spreadsheets/d/1uqmbzBFlUT-REgEz-wcp2FdjSo-qBMNsR6Wqm58XjXM/edit` |
| HubSpot Custom Properties Requirement | Ensure properties `lead_type`, `priority`, `budget_signal`, `intent_summary`, `assigned_agent_name`, `assigned_agent_email`, and `lead_status` are created in HubSpot prior to execution. |