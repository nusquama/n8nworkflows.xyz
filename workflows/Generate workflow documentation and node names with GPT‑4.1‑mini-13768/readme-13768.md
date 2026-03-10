Generate workflow documentation and node names with GPT‑4.1‑mini

https://n8nworkflows.xyz/workflows/generate-workflow-documentation-and-node-names-with-gpt-4-1-mini-13768


# Generate workflow documentation and node names with GPT‑4.1‑mini

# AI-Powered n8n Workflow Documentation Generator

### 1. Workflow Overview
This workflow is a comprehensive automation suite designed to analyze, document, and manage n8n workflows using AI (GPT-4o/GPT-4o-mini). It allows users to interactively select a workflow from their instance and perform several high-level actions:
- **Node Renaming:** Suggests SEO-friendly and descriptive names for nodes.
- **Documentation Generation:** Creates structured Markdown documentation or Google Docs.
- **Logic Visualization:** Generates Mermaid.js flowcharts to represent the workflow structure.
- **Metadata Management:** Extracts and refines workflow descriptions and sticky note content.

The logic is divided into four main functional tracks:
1.  **Selection & Retrieval:** Fetching available workflows and focusing on a specific target.
2.  **Node Editing Track:** AI-driven renaming of nodes and updating the workflow JSON.
3.  **Documentation Track:** Processing workflow JSON to create human-readable docs via LLMs.
4.  **Sticky Note & Idea Track:** Managing internal notes and converting them into structured insights.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Workflow Selection
Retrieves the list of existing workflows from the n8n instance and provides a UI for the user to select one.
- **Nodes Involved:** `Form Trigger`, `Select Work Process`, `Get All Workflows`, `Aggregate`, `Select Workflow form Dynamic Workflow List`, `Code (Clean FlocFlow ID)`.
- **Node Details:**
    - **Form Trigger:** Entry point for the user via a web form.
    - **Get All Workflows / Aggregate:** Uses the n8n node to fetch IDs and names, then aggregates them into a list for selection.
    - **Select Workflow form Dynamic Workflow List:** A Form node where the user chooses the specific workflow to process.
    - **Code (Clean FlocFlow ID):** Extracts the specific ID and ensures it is ready for the retrieval node.

#### 2.2 Routing & Action Selection
Loads the full JSON of the selected workflow and routes the execution to the chosen task.
- **Nodes Involved:** `Get Specified Workflow`, `Switch`.
- **Node Details:**
    - **Get Specified Workflow:** Fetched via n8n node using the ID from the previous block.
    - **Switch:** A 4-way router that sends data to:
        1. Node Renaming/Editing logic.
        2. Workflow Name/SEO logic.
        3. Sticky Note processing.
        4. Documentation/Mermaid Chart generation.

#### 2.3 AI Node Editor (Renaming Logic)
Analyzes node types and their parameters to suggest more descriptive names, then creates a new workflow version.
- **Nodes Involved:** `Get Workflow Nodes Connections`, `AI Agent (node editor)`, `Prepare LLM Verify Data`, `Verify Update Nodes`, `Apply Update Nodes Connections`, `Assemble Update Workflow Object`, `Save Renamed Workflow`, `Create Workflow Update Links`, `Check Trigger Source Type`, `Display Update & Old Workflow Links`.
- **Node Details:**
    - **AI Agent (node editor):** Uses OpenAI and specific tools (`Think`, `Research`) to analyze current node names and suggest improvements.
    - **Verify Update Nodes:** A safety check to ensure the AI's suggestions are valid.
    - **Save Renamed Workflow:** Uses the n8n node to create a new workflow (or update) with the modified node names.
    - **Display Update & Old Workflow Links:** Provides the user with links to the original and the newly generated workflow.

#### 2.4 Documentation & Mermaid Track
Creates technical documentation and a visual representation of the workflow.
- **Nodes Involved:** `Get Single Workflow Using ID`, `Create a mermaid chart`, `Merge`, `Check Action`, `Basic LLM Chain`, `Merge1`, `Create DOCS`, `Check the action edit again`, `Edit Page`, `Workflow md content`, `Display Workflow Documentation`.
- **Node Details:**
    - **Create a mermaid chart:** Code node that parses the `connections` object into Mermaid.js syntax.
    - **Basic LLM Chain:** Takes the workflow JSON and the Mermaid chart to write a comprehensive documentation piece.
    - **Edit Page / Workflow md content:** Formats the final output for display or Google Docs integration.

#### 2.5 Sticky Note Analysis Track
Extracts and processes information contained within sticky notes.
- **Nodes Involved:** `Code (Separate notes Form Workflow)`, `Loop Over Items`, `Get Idea Form n8n Data Table`, `AI Agent (sticky-notes editor)`, `Display Workflow Description`.
- **Node Details:**
    - **Code (Separate notes Form Workflow):** Extracts `n8n-nodes-base.stickyNote` objects from the workflow JSON.
    - **AI Agent (sticky-notes editor):** Summarizes or expands on the content found in these notes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Form Trigger | Form Trigger | Start Workflow | None | Select Work Process |  |
| Select Work Process | Form | User Interaction | Form Trigger | Get All Workflows |  |
| Get All Workflows | n8n | Data Retrieval | Select Work Process | Aggregate |  |
| Aggregate | Aggregate | Data Processing | Get All Workflows | Select Workflow form Dynamic Workflow List |  |
| Select Workflow form Dynamic Workflow List | Form | User Interaction | Aggregate | Code (Clean FlocFlow ID) |  |
| Code (Clean FlocFlow ID) | Code | Data Formatting | Select Workflow form Dynamic Workflow List | Get Specified Workflow |  |
| Get Specified Workflow | n8n | Data Retrieval | Code (Clean FlocFlow ID) | Switch |  |
| Switch | Switch | Logic Routing | Get Specified Workflow | Multiple (4 paths) |  |
| AI Agent (node editor) | AI Agent | AI Processing | Get Workflow Nodes Connections | Prepare LLM Verify Data |  |
| AI Agent (workflow name editor) | AI Agent | AI Processing | Switch | Edit Name |  |
| AI Agent (sticky-notes editor) | AI Agent | AI Processing | Get Idea Form n8n Data Table | Loop Over Items |  |
| Basic LLM Chain | LLM Chain | AI Processing | Check Action | Merge1 |  |
| OpenAI | Chat OpenAI | AI Model Provider | AI Agents/Chains | None |  |
| Save Renamed Workflow | n8n | Data Storage | Assemble Update Workflow Object | Create Workflow Update Links |  |
| Display Workflow Documentation | Form | Output Display | Edit Page | None |  |
| Create a mermaid chart | Code | Visualization | Get Single Workflow Using ID | Merge |  |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a `Form Trigger` to start the process.
2.  **Workflow Fetching:**
    *   Add an `n8n` node (Operation: Get All) to list workflows.
    *   Add an `Aggregate` node to create a selection list.
    *   Add a `Form` node to let the user pick the workflow and the desired action (Rename, Document, etc.).
3.  **Data Retrieval:** Add an `n8n` node (Operation: Get) using the ID from the selection.
4.  **Routing:** Use a `Switch` node to branch based on the user's action choice.
5.  **Configure AI (OpenAI):** 
    *   Place a `Chat OpenAI` model node (GPT-4o recommended).
    *   Connect it to the various AI Agents and LLM Chains.
    *   Add `Memory Buffer Window` nodes to the agents for context retention.
6.  **Node Renaming Logic (Path 1):**
    *   Add a `Set` node to extract node connections.
    *   Add an `AI Agent` with a system prompt: "Analyze these n8n nodes and suggest SEO-friendly names."
    *   Add `Set` nodes to rebuild the workflow JSON with the new names.
    *   Add an `n8n` node (Operation: Create) to save the new workflow version.
7.  **Documentation Logic (Path 2):**
    *   Add a `Code` node to generate a Mermaid chart from the JSON connections.
    *   Add a `Basic LLM Chain` to synthesize the JSON and Chart into Markdown.
    *   Add a `Form` node to display the final text.
8.  **Output Display:** Configure final `Form` or `HTML` nodes to show the results (URLs for new workflows, Markdown text, or Google Doc links).
9.  **Credentials:** Ensure n8n credentials (API Key) and OpenAI credentials are configured and authorized.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Use GPT-4o-mini for cost efficiency in renaming, but GPT-4o for complex documentation. | Model Selection |
| The n8n node requires an API Key with "Workflow Read/Write" permissions. | Security Requirement |
| Mermaid.js charts can be rendered directly in GitHub or various Markdown viewers. | Visualization |