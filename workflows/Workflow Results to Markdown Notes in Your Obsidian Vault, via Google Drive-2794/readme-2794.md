Workflow Results to Markdown Notes in Your Obsidian Vault, via Google Drive

https://n8nworkflows.xyz/workflows/workflow-results-to-markdown-notes-in-your-obsidian-vault--via-google-drive-2794


# Workflow Results to Markdown Notes in Your Obsidian Vault, via Google Drive

### 1. Workflow Overview

This workflow automates the conversion of any n8n workflow output into Markdown notes that are saved in a Google Drive folder synchronized with an Obsidian Vault via a symbolic link. It enables real-time creation and updating of Markdown notes in Obsidian, leveraging Google Drive as the intermediary storage and sync mechanism.

**Target Use Cases:**
- Automatically generate Markdown notes from diverse data sources such as RSS feeds, YouTube transcripts, or Slack messages.
- Use AI agents to format notes according to knowledge management methodologies (e.g., Zettelkasten), generate YAML frontmatter metadata, and suggest relevant tags.
- Manage attachments by saving binary files to Google Drive alongside notes.

**Logical Blocks:**

- **1.1 Input Reception:** Receives workflow outputs from any other n8n workflow via an Execute Workflow Trigger node.
- **1.2 Binary Attachment Handling:** Checks for and saves any binary attachments to Google Drive.
- **1.3 AI-Assisted Note Composition:** Optionally uses AI agents to generate structured Zettelkasten notes and YAML frontmatter from raw input data.
- **1.4 Data Restructuring:** Extracts and organizes note components (title, content, frontmatter, references) for saving.
- **1.5 Markdown Note Saving:** Saves the composed Markdown note as a `.md` file in the designated Google Drive folder.
- **1.6 Setup and Integration Instructions:** Provides sticky notes with detailed setup instructions for folder creation, Google Drive configuration, and symlink creation to integrate with Obsidian Vault.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives the output data from any other n8n workflow, acting as the entry point for note creation.

- **Nodes Involved:**  
  - Receive results from any workflow

- **Node Details:**  
  - **Receive results from any workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry trigger node that accepts data from other workflows.  
    - Configuration: Default, no parameters set.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to the conditional check for binary attachments and directly to the Markdown file saving node.  
    - Edge Cases: If no data is received, downstream nodes will not execute.  
    - Version: 1

#### 1.2 Binary Attachment Handling

- **Overview:**  
  Checks if the incoming data contains binary attachments and saves them to Google Drive if present.

- **Nodes Involved:**  
  - If the input has binary attachment  
  - Save attachment

- **Node Details:**  
  - **If the input has binary attachment**  
    - Type: If  
    - Role: Conditional node that checks for the existence of a binary property in the input JSON.  
    - Configuration: Condition tests if `$json["binary"]` exists.  
    - Inputs: From "Receive results from any workflow"  
    - Outputs: True branch to "Save attachment", false branch ignored.  
    - Edge Cases: If binary data is malformed or missing, attachment saving is skipped.  
    - Version: 2.2

  - **Save attachment**  
    - Type: Google Drive  
    - Role: Saves binary data as files in a specified Google Drive folder.  
    - Configuration:  
      - Operation: createFromBinary (implied by inputDataFieldName)  
      - Folder ID: Points to a specific Google Drive folder (ID: 15dvUtfSjaCCXmnOVeIUfeyRd_raI3PnQ)  
      - Drive: "My Drive"  
      - Input Data Field: "data" (binary content)  
    - Inputs: From "If the input has binary attachment" (true branch)  
    - Outputs: None (end node)  
    - Edge Cases: Google Drive API errors (auth, quota, network), invalid binary data.  
    - Version: 3

#### 1.3 AI-Assisted Note Composition (Optional)

- **Overview:**  
  Uses AI agents to generate a structured Zettelkasten note and YAML frontmatter metadata from the raw input JSON data.

- **Nodes Involved:**  
  - Write Zettlekasten note from input1  
  - Structured Output Parser  
  - OpenAI Chat Model  
  - Write YAML Frontmatter  
  - Structured Output Parser1  
  - OpenAI Chat Model1  
  - Sticky Note3 (instructional)

- **Node Details:**  
  - **Write Zettlekasten note from input1**  
    - Type: LangChain Agent (AI agent)  
    - Role: Converts raw JSON input into a structured Zettelkasten note in JSON format.  
    - Configuration:  
      - System message instructs the AI to extract key insights, create a unique ID, title, content, tags, and references following Zettelkasten principles.  
      - Input text: JSON stringified input data.  
      - Output parser enabled for structured JSON output.  
    - Inputs: From "Receive results from any workflow" (via conditional or direct connection)  
    - Outputs: To "Write YAML Frontmatter"  
    - Edge Cases: AI model errors, malformed input JSON, API rate limits.  
    - Version: 1.7

  - **Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses AI output into a defined schema with "title" and "content" fields.  
    - Configuration: Manual schema specifying required fields.  
    - Inputs: From "Write Zettlekasten note from input1" (ai_outputParser)  
    - Outputs: To "Write Zettlekasten note from input1" (ai_languageModel) — this connection is part of the AI node's internal flow.  
    - Edge Cases: Parsing errors if AI output deviates from schema.  
    - Version: 1.2

  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides the language model backend for the AI agent nodes.  
    - Configuration: Uses default OpenAI API settings with linked credentials.  
    - Inputs: From "Structured Output Parser" (ai_languageModel)  
    - Outputs: To "Write Zettlekasten note from input1" (ai_outputParser)  
    - Edge Cases: API key invalid, rate limits, network errors.  
    - Version: 1

  - **Write YAML Frontmatter**  
    - Type: LangChain Agent  
    - Role: Generates YAML frontmatter metadata for the note based on the AI-generated content.  
    - Configuration:  
      - System message instructs generation of YAML frontmatter including title, date, tags, aliases, status, and source.  
      - Input text: The "content" field from the AI-generated note.  
      - Output parser enabled for structured output.  
    - Inputs: From "Write Zettlekasten note from input1" (main output)  
    - Outputs: To "Restructure JSON"  
    - Edge Cases: AI output parsing errors, API failures.  
    - Version: 1.7

  - **Structured Output Parser1**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses the YAML frontmatter generated by the AI agent.  
    - Configuration: Example JSON schema with "frontmatter" field.  
    - Inputs: From "Write YAML Frontmatter" (ai_outputParser)  
    - Outputs: To "Write YAML Frontmatter" (ai_languageModel) — internal AI flow.  
    - Edge Cases: Parsing errors if AI output is malformed.  
    - Version: 1.2

  - **OpenAI Chat Model1**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Language model backend for the YAML frontmatter generation.  
    - Configuration: Uses same OpenAI credentials as above.  
    - Inputs: From "Structured Output Parser1" (ai_languageModel)  
    - Outputs: To "Write YAML Frontmatter" (ai_outputParser)  
    - Edge Cases: Same as above.  
    - Version: 1

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Provides user instructions about the optional AI agent usage for note composition.  
    - Content: Explains how to insert AI agents between the webhook and Google Drive node for enhanced note formatting.  
    - Position: Informational only, no connections.

#### 1.4 Data Restructuring

- **Overview:**  
  Extracts and organizes the AI-generated note components into JSON fields for saving.

- **Nodes Involved:**  
  - Restructure JSON

- **Node Details:**  
  - **Restructure JSON**  
    - Type: Set  
    - Role: Assigns extracted fields from AI output to standardized JSON keys: title, content, frontmatter, references.  
    - Configuration:  
      - title: From AI note output title  
      - content: From AI note output content  
      - frontmatter: From AI-generated frontmatter  
      - references: From AI note output references  
    - Inputs: From "Write YAML Frontmatter"  
    - Outputs: To "Save Markdown file"  
    - Edge Cases: Missing fields in AI output may cause empty values.  
    - Version: 3.4

#### 1.5 Markdown Note Saving

- **Overview:**  
  Saves the final Markdown note file to the designated Google Drive folder, making it accessible in Obsidian via sync.

- **Nodes Involved:**  
  - Save Markdown file

- **Node Details:**  
  - **Save Markdown file**  
    - Type: Google Drive  
    - Role: Creates or updates a Markdown `.md` file in Google Drive with the note content.  
    - Configuration:  
      - Filename: `{{ $json.title }}.md` (dynamic from note title)  
      - Content: Combines frontmatter and content in Markdown format:  
        ```
        ---
        {{ $json.frontmatter }}
        ---
        {{ $json.content }}
        ```  
      - Folder ID: Points to the same Google Drive folder as attachments (ID: 15dvUtfSjaCCXmnOVeIUfeyRd_raI3PnQ)  
      - Drive: "My Drive"  
      - Operation: createFromText  
    - Inputs: From "Receive results from any workflow" (directly) and from "Restructure JSON" (after AI processing)  
    - Outputs: None (end node)  
    - Edge Cases: Google Drive API errors, filename conflicts, invalid content.  
    - Version: 3

#### 1.6 Setup and Integration Instructions

- **Overview:**  
  Provides detailed user instructions on how to configure Google Drive, create the necessary folders, and establish a symbolic link to the Obsidian Vault.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**  
  - **Sticky Note1**  
    - Content: Explains the workflow purpose and instructs users to send outputs to the Execute Workflow Trigger node.  
  - **Sticky Note**  
    - Content: Step-by-step setup instructions for creating the Google Drive folder, configuring the Google Drive node, and linking the folder to Obsidian Vault.  
  - **Sticky Note2**  
    - Content: Detailed instructions on creating a symbolic link (symlink) between the Google Drive folder and Obsidian Vault folder on Windows using the `mklink` command.  
  - All are informational only, no connections.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                               | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                  |
|--------------------------------|----------------------------------|-----------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Receive results from any workflow | Execute Workflow Trigger          | Entry point receiving workflow outputs        | None                             | If the input has binary attachment, Save Markdown file | Sticky Note1: Explains sending outputs to this trigger node                                  |
| If the input has binary attachment | If                              | Checks for binary attachments                   | Receive results from any workflow | Save attachment                  |                                                                                              |
| Save attachment                | Google Drive                     | Saves binary attachments to Google Drive       | If the input has binary attachment | None                            |                                                                                              |
| Write Zettlekasten note from input1 | LangChain Agent                 | AI agent generating structured Zettelkasten note | Receive results from any workflow | Write YAML Frontmatter           | Sticky Note3: Optional AI usage instructions                                                |
| Structured Output Parser       | LangChain Output Parser Structured | Parses AI note JSON output                      | Write Zettlekasten note from input1 | Write Zettlekasten note from input1 (internal AI flow) |                                                                                              |
| OpenAI Chat Model              | LangChain LM Chat OpenAI         | Provides OpenAI language model backend          | Structured Output Parser          | Write Zettlekasten note from input1 (internal AI flow) |                                                                                              |
| Write YAML Frontmatter         | LangChain Agent                 | AI agent generating YAML frontmatter metadata  | Write Zettlekasten note from input1 | Restructure JSON               |                                                                                              |
| Structured Output Parser1      | LangChain Output Parser Structured | Parses YAML frontmatter output                  | Write YAML Frontmatter            | Write YAML Frontmatter (internal AI flow) |                                                                                              |
| OpenAI Chat Model1             | LangChain LM Chat OpenAI         | OpenAI backend for YAML frontmatter generation | Structured Output Parser1         | Write YAML Frontmatter (internal AI flow) |                                                                                              |
| Restructure JSON              | Set                              | Extracts and organizes note components          | Write YAML Frontmatter            | Save Markdown file              |                                                                                              |
| Save Markdown file             | Google Drive                     | Saves final Markdown note file to Google Drive | Receive results from any workflow, Restructure JSON | None                            | Sticky Note: Setup instructions for Google Drive node configuration and filename formatting  |
| Sticky Note                   | Sticky Note                      | Setup instructions for Google Drive folder and Obsidian symlink | None                             | None                             | See content for detailed setup steps                                                        |
| Sticky Note1                  | Sticky Note                      | Workflow overview and trigger usage instructions | None                             | None                             | See content for workflow purpose and trigger usage                                          |
| Sticky Note2                  | Sticky Note                      | Instructions for creating symlink between Google Drive and Obsidian Vault | None                             | None                             | See content for Windows mklink command instructions                                         |
| Sticky Note3                  | Sticky Note                      | Optional AI agent usage instructions            | None                             | None                             | See content for AI agent integration details                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Execute Workflow Trigger Node**  
   - Type: Execute Workflow Trigger  
   - Name: "Receive results from any workflow"  
   - No special parameters. This node will receive data from other workflows.

2. **Add an If Node to Check for Binary Attachments**  
   - Type: If  
   - Name: "If the input has binary attachment"  
   - Condition: Check if `$json["binary"]` exists (operator: string exists)  
   - Connect input from "Receive results from any workflow".

3. **Add Google Drive Node to Save Attachments**  
   - Type: Google Drive  
   - Name: "Save attachment"  
   - Operation: createFromBinary (default for binary input)  
   - Drive ID: Select "My Drive"  
   - Folder ID: Set to your designated Google Drive folder ID (e.g., "15dvUtfSjaCCXmnOVeIUfeyRd_raI3PnQ")  
   - Input Data Field Name: "data" (binary field)  
   - Connect input from the true output of the If node.

4. **Add Google Drive Node to Save Markdown Notes**  
   - Type: Google Drive  
   - Name: "Save Markdown file"  
   - Operation: createFromText  
   - Drive ID: "My Drive"  
   - Folder ID: Same as attachment folder  
   - Name: Set expression to `={{ $json.title }}.md`  
   - Content: Set expression to:  
     ```
     =---
     {{ $json.frontmatter }}
     ---
     {{ $json.content }}
     ```  
   - Connect input from both:  
     - Directly from "Receive results from any workflow" (for non-AI workflows)  
     - From "Restructure JSON" node (for AI-processed workflows)

5. **(Optional) Add AI Agent Node to Generate Zettelkasten Note**  
   - Type: LangChain Agent  
   - Name: "Write Zettlekasten note from input1"  
   - Credentials: OpenAI API credentials configured  
   - Parameters:  
     - Text: `={{ JSON.stringify($json) }}`  
     - System Message: Instruct AI to create a Zettelkasten note with unique ID, title, content, tags, references in JSON format.  
     - Enable output parser with schema for title and content.  
   - Connect input from "Receive results from any workflow".

6. **Add LangChain Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Name: "Structured Output Parser"  
   - Schema: Manual with fields "title" and "content"  
   - Connect input from "Write Zettlekasten note from input1" (ai_outputParser).

7. **Add OpenAI Chat Model Node**  
   - Type: LangChain LM Chat OpenAI  
   - Name: "OpenAI Chat Model"  
   - Credentials: Same OpenAI API credentials  
   - Connect input from "Structured Output Parser" (ai_languageModel).

8. **Add AI Agent Node to Generate YAML Frontmatter**  
   - Type: LangChain Agent  
   - Name: "Write YAML Frontmatter"  
   - Credentials: OpenAI API credentials  
   - Parameters:  
     - Text: `={{ $json.output.content }}` (content from previous AI output)  
     - System Message: Instruct AI to generate YAML frontmatter with title, date, tags, aliases, status, source.  
     - Enable output parser.  
   - Connect input from "Write Zettlekasten note from input1" (main output).

9. **Add LangChain Output Parser Node for Frontmatter**  
   - Type: LangChain Output Parser Structured  
   - Name: "Structured Output Parser1"  
   - Example Schema: JSON with "frontmatter" field  
   - Connect input from "Write YAML Frontmatter" (ai_outputParser).

10. **Add OpenAI Chat Model Node for Frontmatter**  
    - Type: LangChain LM Chat OpenAI  
    - Name: "OpenAI Chat Model1"  
    - Credentials: OpenAI API credentials  
    - Connect input from "Structured Output Parser1" (ai_languageModel).

11. **Add Set Node to Restructure JSON**  
    - Type: Set  
    - Name: "Restructure JSON"  
    - Assignments:  
      - title: `={{ $('Write Zettlekasten note from input1').item.json.output.title }}`  
      - content: `={{ $('Write Zettlekasten note from input1').item.json.output.content }}`  
      - frontmatter: `={{ $json.output.frontmatter }}`  
      - references: `={{ $('Write Zettlekasten note from input1').item.json.output.references }}`  
    - Connect input from "Write YAML Frontmatter".

12. **Connect "Restructure JSON" Output to "Save Markdown file" Input**  
    - This completes the AI-assisted note creation path.

13. **Add Sticky Notes for Setup Instructions**  
    - Create three Sticky Note nodes with the following content:  
      - Sticky Note1: Workflow overview and trigger usage instructions.  
      - Sticky Note: Google Drive folder creation and node configuration instructions.  
      - Sticky Note2: Symlink creation instructions for Windows using `mklink`.  
      - Sticky Note3: Optional AI agent usage instructions.

14. **Configure Credentials**  
    - Google Drive OAuth2 API credentials must be set up and linked to Google Drive nodes.  
    - OpenAI API credentials must be configured and linked to LangChain AI nodes.

15. **Folder Setup Outside n8n**  
    - Create a folder in Google Drive Desktop app to sync with your local machine.  
    - Create a symbolic link from this folder to a folder inside your Obsidian Vault using the Windows command prompt:  
      ```
      mklink /D "C:\Path\To\Obsidian\Vault\TargetFolder" "C:\Path\To\GoogleDrive\SourceFolder"
      ```  
    - This ensures notes saved in Google Drive appear instantly in Obsidian.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automatically creates and updates notes in your Obsidian Vault in real-time from n8n workflow results.         | Sticky Note1 content                                                                                |
| Create a folder in your Google Drive Desktop app and configure the Google Drive node to save Markdown notes there.           | Sticky Note content                                                                                 |
| Establish a symbolic link between the Google Drive folder and a folder in your Obsidian Vault to enable instant syncing.    | Sticky Note2 content with Windows `mklink` command instructions                                   |
| Optional: Use AI agents to compose notes in Zettelkasten format, generate YAML frontmatter, and suggest tags.               | Sticky Note3 content                                                                                |
| OpenAI API credentials are required for AI agent nodes; Google Drive OAuth2 credentials are required for Google Drive nodes.| Credential setup instructions implicit in node configurations                                     |
| Example use cases include converting RSS feeds, YouTube transcripts, or Slack messages into Obsidian notes.                 | Workflow description                                                                                |

---

This documentation provides a complete, structured reference for understanding, reproducing, and modifying the "Workflow Results to Markdown Notes in Your Obsidian Vault, via Google Drive" n8n workflow. It covers all nodes, their roles, configurations, and integration points, enabling both human users and AI agents to work effectively with this workflow.