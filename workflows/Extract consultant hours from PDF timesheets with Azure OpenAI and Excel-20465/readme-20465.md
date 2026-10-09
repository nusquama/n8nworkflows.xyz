Extract consultant hours from PDF timesheets with Azure OpenAI and Excel

https://n8nworkflows.xyz/workflows/extract-consultant-hours-from-pdf-timesheets-with-azure-openai-and-excel-20465


# Extract consultant hours from PDF timesheets with Azure OpenAI and Excel

### 1. Workflow Overview

This workflow automates the extraction, processing, and storage of consultant hours from PDF timesheets submitted via an n8n chat interface. Target use cases include project management, consulting operations, and financial controlling where timesheets are traditionally processed manually. 

The workflow is logically structured into four sequential blocks:
- **1.1 Input Reception & Normalization:** Receives chat inputs, handles multiple file attachments, and splits them into distinct items for parallel or sequential item processing.
- **1.2 AI-Powered Extraction & Parsing:** Extracts raw text from each PDF, passes it to an Azure OpenAI agent using a specialized system prompt, and strictly parses the returned string into validated JSON data.
- **1.3 Data Storage & Persistence:** Records the structured consultant hours simultaneously into a Microsoft 365 Excel workbook and an internal n8n Data Table.
- **1.4 Notification:** Dispatches a completion confirmation email via Microsoft Outlook once all database and spreadsheet updates conclude successfully.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Captures incoming messages containing PDF uploads from the chat interface and normalizes the payload structure so each uploaded PDF file is isolated into its own independent item.
- **Nodes Involved:** 
  - When chat message received
  - Get binaries out of chat

- **Node Details:**
  - **When chat message received**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chatTrigger` (Trigger node)
    - *Configuration:* Enables file uploads (`allowFileUploads: true`) via chat interface.
    - *Expressions/Variables:* None (entrypoint).
    - *Connections:* Outputs to `Get binaries out of chat`.
    - *Failure Types:* Webhook failure or payload size limitations if files exceed allowed limits.
  
  - **Get binaries out of chat**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Iterates through all binary properties attached to the incoming item, creating a new n8n item for each file while standardizing the binary property key to `data`.
    - *Expressions/Variables:* `$input.item.binary`
    - *Connections:* Input from `When chat message received`; Output to `Extract data from pdfs`.
    - *Failure Types:* JavaScript runtime exceptions if binary objects are malformed or empty.

---

#### 2.2 AI-Powered Extraction & Parsing
- **Overview:** Converts binary PDF files into plain text, evaluates the text via an Azure OpenAI agent instructed to extract specific time categories, and validates the output structure.
- **Nodes Involved:**
  - Extract data from pdfs
  - Timesheet Analyzer Agent
  - Companion Model
  - JSON Parser

- **Node Details:**
  - **Extract data from pdfs**
    - *Type and Technical Role:* `n8n-nodes-base.extractFromFile` (File processing)
    - *Configuration:* Operation set to `pdf`. Extracts text layers from binary inputs.
    - *Expressions/Variables:* None.
    - *Connections:* Input from `Get binaries out of chat`; Output to `Timesheet Analyzer Agent`.
    - *Failure Types:* Corrupted PDF format, password-protected files, or lack of a text layer (scanned PDFs lacking OCR will yield empty text).
  
  - **Timesheet Analyzer Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Uses a system prompt to define the role of a consulting timesheet expert. Enforces a strict JSON response schema covering consultant name, onsite hours, remote hours, and unknown hours.
    - *Expressions/Variables:* `={{ $json.text }}`
    - *Connections:* Input from `Extract data from pdfs` and `Companion Model`; Output to `JSON Parser`.
    - *Failure Types:* API rate limits, model hallucinations, or failure to strictly obey JSON formatting rules.
  
  - **Companion Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` (Language Model sub-node)
    - *Configuration:* Configured to use an Azure OpenAI chat model (`gpt-5` via credentials).
    - *Credentials:* `PPI Companion` (`azureOpenAiApi`)
    - *Connections:* Output to `Timesheet Analyzer Agent`.
    - *Failure Types:* Authentication failure, invalid API deployment names, or service downtime.
  
  - **JSON Parser**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Safely parses the raw text string returned by the AI agent into native JSON objects. Throws an explicit error halting the execution if the payload is not valid JSON. Unwraps single-element arrays automatically.
    - *Expressions/Variables:* `$input.all()`, `item.json.output`
    - *Connections:* Input from `Timesheet Analyzer Agent`; Output to `Append timesheet data to sheet`.
    - *Failure Types:* Throws explicit operational errors if the AI response contains markdown wrappers or invalid JSON structures.

---

#### 2.3 Data Storage & Persistence
- **Overview:** Records the validated consultant work hours into external and internal tracking repositories.
- **Nodes Involved:**
  - Append timesheet data to sheet
  - Insert time recordings

- **Node Details:**
  - **Append timesheet data to sheet**
    - *Type and Technical Role:* `n8n-nodes-base.microsoftExcel` (Spreadsheet integration)
    - *Configuration:* Resource set to `worksheet`, operation set to `append`. Maps fields (`name`, `onsite`, `remote`, `unknown`) to corresponding target columns in a specified Microsoft 365 Excel workbook.
    - *Expressions/Variables:* `={{ $('JSON Parser').item.json.name }}`, `={{ $('JSON Parser').item.json.onsite_hours }}` (and remote/unknown equivalents).
    - *Credentials:* Microsoft Excel OAuth2 API (`jWyHv7XTwwNHECmf`)
    - *Connections:* Input from `JSON Parser`; Output to `Insert time recordings`.
    - *Failure Types:* OAuth token expiration, missing worksheet columns, or locked Excel files.
  
  - **Insert time recordings**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Internal database storage)
    - *Configuration:* Appends records matching the schema (`name`, `onsite`, `remote`, `unknown`) to an internal n8n Data Table (`4eKyTPr6K3zHfN2A`).
    - *Expressions/Variables:* `={{ $json.name }}`, `={{ $json.onsite }}` (mapping parsed properties).
    - *Connections:* Input from `Append timesheet data to sheet`; Output to `Send a message with excel time sheet `.
    - *Failure Types:* Schema mismatches between payload fields and Data Table definitions.

---

#### 2.4 Notification
- **Overview:** Sends an email notification indicating that the batch timesheet processing has finished.
- **Nodes Involved:**
  - Send a message with excel time sheet 

- **Node Details:**
  - **Send a message with excel time sheet**
    - *Type and Technical Role:* `n8n-nodes-base.microsoftOutlook` (Email integration)
    - *Configuration:* Sends an email via Microsoft Outlook containing a static body content and configured recipient address.
    - *Expressions/Variables:* Standard parameters for subject and body.
    - *Credentials:* Microsoft Outlook OAuth2 API (`Yk7dnrJ8F2QJOKwT`)
    - *Connections:* Input from `Insert time recordings`.
    - *Failure Types:* Outlook OAuth credential revocation, invalid recipient email syntax, or mail server restrictions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When chat message received | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat trigger supporting file uploads | None | Get binaries out of chat | - When chat message received: Open the chat and upload one or more PDF timesheets. File uploads are enabled in the trigger options. |
| Get binaries out of chat | `n8n-nodes-base.code` | Splits multi-binary chat payloads into individual items | When chat message received | Extract data from pdfs | - Get binaries out of chat: All uploaded files arrive in a single item. This Code node splits each PDF into its own item under the binary key `data`. |
| Extract data from pdfs | `n8n-nodes-base.extractFromFile` | Extracts text layers from PDF binaries | Get binaries out of chat | Timesheet Analyzer Agent | - Extract data from pdfs: Converts each PDF into plain text, which is passed on to the AI Agent. |
| Timesheet Analyzer Agent | `@n8n/n8n-nodes-langchain.agent` | AI agent analyzing timesheet text and returning JSON | Extract data from pdfs, Companion Model | JSON Parser | AI Agent: Reads the timesheet text and classifies every entry as onsite, remote, or unknown. It returns a JSON object with the consultant's name and total hours per category. |
| Companion Model | `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` | Azure OpenAI language model provider | None | Timesheet Analyzer Agent | Companion: language model used by the agent. You can swap it for OpenAI, Anthropic, or another model. |
| JSON Parser | `n8n-nodes-base.code` | Parses and validates JSON output from the AI agent | Timesheet Analyzer Agent | Append timesheet data to sheet | JSON Parser: Parses the agent's text output into clean JSON fields. It stops the workflow with a clear error if the model returns invalid JSON. |
| Append timesheet data to sheet | `n8n-nodes-base.microsoftExcel` | Appends extracted hours to an Excel worksheet | JSON Parser | Insert time recordings | Append timesheet data to sheet: Adds one row per timesheet to your Excel workbook in Microsoft 365 (columns: name, onsite, remote, unknown). |
| Insert time recordings | `n8n-nodes-base.dataTable` | Saves extracted data into an n8n Data Table | Append timesheet data to sheet | Send a message with excel time sheet | Insert row: Saves the same result to an n8n Data Table. This keeps a history for reporting inside n8n. |
| Send a message with excel time sheet | `n8n-nodes-base.microsoftOutlook` | Sends an Outlook notification email upon completion | Insert time recordings | None | Sends an Outlook email once all timesheets are processed. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Trigger:** Place a **When chat message received** node (`@n8n/n8n-nodes-langchain.chatTrigger`). In its options, enable `Allow File Uploads`.
2. **Add Binary Splitting Logic:** Connect a **Code** node named `Get binaries out of chat` (`n8n-nodes-base.code`) to the trigger. Insert JavaScript code to iterate over `item.binary` properties and normalize each file into an independent item with a `data` binary key.
3. **Add PDF Extraction:** Connect an **Extract from File** node named `Extract data from pdfs` (`n8n-nodes-base.extractFromFile`). Set the operation to `pdf`.
4. **Configure the AI Agent & Language Model:** 
   - Add an **AI Agent** node named `Timesheet Analyzer Agent` (`@n8n/n8n-nodes-langchain.agent`), connected to receive text from the PDF extraction node.
   - Configure the Agent's system message to instruct the model to analyze timesheet texts, compute hours into `onsite_hours`, `remote_hours`, and `unknown_hours`, and return exclusively valid JSON. Set the prompt text parameter to `={{ $json.text }}`.
   - Attach an **Azure Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`) named `Companion Model` to the agent's model input. Configure credentials for Azure OpenAI (`azureOpenAiApi`) and select your target model deployment.
5. **Add JSON Validation:** Connect a **Code** node named `JSON Parser` (`n8n-nodes-base.code`) after the AI Agent. Use JavaScript to parse `item.json.output` (or fallback properties) via `JSON.parse()`, throwing an explicit Error if parsing fails.
6. **Configure Excel Storage:**
   - Connect a **Microsoft Excel** node named `Append timesheet data to sheet` (`n8n-nodes-base.microsoftExcel`).
   - Configure credentials (`microsoftExcelOAuth2Api`).
   - Set resource to `worksheet` and operation to `append`. Select your target workbook and worksheet.
   - Map values using expressions referencing the parser: `name` (`={{ $('JSON Parser').item.json.name }}`), `onsite` (`={{ $('JSON Parser').item.json.onsite_hours }}`), `remote` (`={{ $('JSON Parser').item.json.remote_hours }}`), and `unknown` (`={{ $('JSON Parser').item.json.unknown_hours }}`).
7. **Configure n8n Data Table Storage:**
   - Connect a **Data Table** node named `Insert time recordings` (`n8n-nodes-base.dataTable`).
   - Create or select an n8n Data Table containing columns: `name`, `onsite`, `remote`, `unknown`.
   - Map columns to respective values from the current item (`={{ $json.name }}`, etc.).
8. **Configure Outlook Notification:**
   - Connect a **Microsoft Outlook** node named `Send a message with excel time sheet` (`n8n-nodes-base.microsoftOutlook`).
   - Configure Microsoft Outlook OAuth2 credentials (`microsoftOutlookOAuth2Api`).
   - Set recipient address, subject, and body fields accordingly.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Extract consultant hours from PDF timesheets to Excel and an n8n Data Table. Review AI results before billing. | Workflow Overview / Disclaimer |
| Requires PDFs with a text layer. Scanned files require a preceding OCR step. | Technical Prerequisite |