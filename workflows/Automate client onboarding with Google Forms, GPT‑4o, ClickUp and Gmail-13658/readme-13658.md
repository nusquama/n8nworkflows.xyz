Automate client onboarding with Google Forms, GPT‑4o, ClickUp and Gmail

https://n8nworkflows.xyz/workflows/automate-client-onboarding-with-google-forms--gpt-4o--clickup-and-gmail-13658


# Automate client onboarding with Google Forms, GPT‑4o, ClickUp and Gmail

# Workflow Reference: Client Onboarding Automation

This document provides a technical breakdown of the **Client Onboarding Automation** workflow. This system automates the transition from a signed proposal to an active project workspace by integrating form intake, AI-driven task generation, project management setup, and client communication.

---

### 1. Workflow Overview

The workflow automates the repetitive administrative tasks associated with onboarding a new client. It moves from data collection to workspace creation and project scoping in a single automated sequence.

**Logical Blocks:**
- **1.1 Trigger & Intake:** Captures client data via an n8n Form and extracts text from uploaded PDF proposals.
- **1.2 Workspace Provisioning:** Automatically creates dedicated folders in Google Drive and project structures (Folders/Lists) in ClickUp.
- **1.3 AI Task Scoping:** Uses GPT-4o-mini to analyze the proposal text and generate 20–30 granular onboarding tasks.
- **1.4 Project Population:** Iterates through the AI-generated tasks to create individual entries in ClickUp.
- **1.5 Client Delivery:** Sends a personalized welcome email containing links to the newly created resources.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Intake
**Overview:** This block handles the initial entry of client information and the conversion of binary PDF data into a searchable text format.
- **Nodes Involved:** `Receive Onboarding Form`, `Extract PDF Content`, `Set Project Information`.
- **Node Details:**
    - **Receive Onboarding Form:** (Form Trigger) Collects Name, Email, Company, Website, and a PDF file.
    - **Extract PDF Content:** (Extract from File) Specifically configured for PDF operations to pull text from the "Proposal/Scope Document" binary property.
    - **Set Project Information:** (Set) Maps the extracted text to a clean variable `projectInformation` for later AI consumption.

#### 2.2 Workspace Provisioning
**Overview:** Sets up the digital infrastructure for the client.
- **Nodes Involved:** `Create Google Drive Folder`, `Set Drive Folder ID`, `Create ClickUp Folder`, `Create ClickUp List`.
- **Node Details:**
    - **Create Google Drive Folder:** Creates a folder named `[Company Name] | Client Folder` inside a specified parent directory.
    - **Create ClickUp Folder/List:** Creates a hierarchical structure in ClickUp. Note: The Folder node has "Continue Regular Output" enabled for error handling.
    - **Configuration:** Requires predefined Team IDs and Space IDs in ClickUp parameters.

#### 2.3 AI Task Generation
**Overview:** The intelligence core of the workflow. It transforms the raw proposal text into a structured JSON list of tasks.
- **Nodes Involved:** `Generate Onboarding Tasks`, `OpenAI Chat Model`, `Parse Task JSON Output`.
- **Node Details:**
    - **Generate Onboarding Tasks:** (AI Agent) Uses a detailed System Prompt to define the SOP (20-30 tasks, US Central Time, weekend-aware due dates).
    - **OpenAI Chat Model:** Configured to use `gpt-4o-mini`.
    - **Parse Task JSON Output:** (Structured Output Parser) Enforces a schema containing `title`, `description`, and `due_date`.

#### 2.4 Project Population & Notification
**Overview:** Executes the creation of tasks in ClickUp and notifies the client once the environment is ready.
- **Nodes Involved:** `Split Tasks Array`, `Loop Through Tasks`, `Create ClickUp Task`, `Send Welcome Email`.
- **Node Details:**
    - **Loop Through Tasks:** (Split in Batches) Processes each AI-generated task one by one.
    - **Create ClickUp Task:** Maps AI output to ClickUp fields. *Requires manual update of `assignees` IDs.*
    - **Send Welcome Email:** (Gmail) Triggered only after the loop finishes. It uses expressions to include the Google Drive folder link and client name.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Onboarding Form | Form Trigger | Data Entry | (None) | Extract PDF Content | 1. Trigger & Document Intake |
| Extract PDF Content | ExtractFromFile | OCR/Text Extraction | Receive Onboarding Form | Set Project Information | 1. Trigger & Document Intake |
| Set Project Information | Set | Data Normalization | Extract PDF Content | Create Google Drive Folder | 1. Trigger & Document Intake |
| Create Google Drive Folder| Google Drive | Cloud Storage Setup | Set Project Information | Set Drive Folder ID | 2. Workspace Setup |
| Set Drive Folder ID | Set | Variable Storage | Create Google Drive Folder| Create ClickUp Folder | 2. Workspace Setup |
| Create ClickUp Folder | ClickUp | PM Infrastructure | Set Drive Folder ID | Create ClickUp List | 2. Workspace Setup |
| Create ClickUp List | ClickUp | PM Infrastructure | Create ClickUp Folder | Generate Onboarding Tasks| 2. Workspace Setup |
| Generate Onboarding Tasks | AI Agent | Logic & Scoping | Create ClickUp List | Split Tasks Array | 3. AI Task Generation |
| OpenAI Chat Model | ChatOpenAi | LLM Provider | (None) | Generate Onboarding Tasks| 3. AI Task Generation |
| Parse Task JSON Output | OutputParser | Data Structuring | (None) | Generate Onboarding Tasks| 3. AI Task Generation |
| Split Tasks Array | SplitOut | Data Formatting | Generate Onboarding Tasks | Loop Through Tasks | 3. AI Task Generation |
| Loop Through Tasks | SplitInBatches | Iterator | Split Tasks Array, Create ClickUp Task | Send Welcome Email, Create ClickUp Task | 3. AI Task Generation |
| Create ClickUp Task | ClickUp | Task Creation | Loop Through Tasks | Loop Through Tasks | 3. AI Task Generation |
| Send Welcome Email | Gmail | Client Communication | Loop Through Tasks | (None) | 4. Client Notification |

---

### 4. Reproducing the Workflow from Scratch

1.  **Form Setup:** Create a `n8n Form Trigger`. Add fields: Name (String), Email (String), Company Name (String), Website (String), and Proposal (File).
2.  **Text Extraction:** Add an `Extract From File` node. Set operation to "PDF". Set the "Property Name" to the name of the file field from the form.
3.  **Cloud Storage:** Add a `Google Drive` node. Select "Create a Folder". Use an expression for the name: `{{ $json.companyName }} | Client Folder`.
4.  **PM Structure:**
    *   Add a `ClickUp` node to "Create a Folder".
    *   Add a second `ClickUp` node to "Create a List" inside that folder.
    *   *Note:* You must retrieve your ClickUp `Team ID` and `Space ID` from your ClickUp URL or API settings.
5.  **AI Intelligence:** 
    *   Add an `AI Agent` node. 
    *   Attach an `OpenAI Chat Model` (set to `gpt-4o-mini`).
    *   Attach a `Structured Output Parser`. Define a schema with an array of objects containing `title`, `description`, and `due_date`.
    *   Paste the SOP/System Message into the Agent to ensure it knows how to calculate dates and task density.
6.  **Looping Logic:**
    *   Add a `Split Out` node to target the `output.tasks` array from the AI.
    *   Add a `Split in Batches` node.
    *   Inside the loop, add a `ClickUp` node to "Create a Task". Map the `title`, `description`, and `due_date` from the current batch item.
7.  **Final Step:** Add a `Gmail` node connected to the "done" output of the loop. Draft the email body using expressions to pull the Google Drive link and Client Name.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Contact & Feedback** | [Start the conversation here](https://tally.so/r/EkKGgB) |
| **Project Creator** | [Milo Bravo | LinkedIn](https://linkedin.com/in/MiloBravo/) |
| **AI Model Requirement** | Requires OpenAI API key with access to GPT-4o-mini. |
| **ClickUp Requirement** | User must manually find and input Space/Team IDs and Assignee IDs. |