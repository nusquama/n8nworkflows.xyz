Natural Language to SQL Converter for Google Sheets with GPT-4o

https://n8nworkflows.xyz/workflows/natural-language-to-sql-converter-for-google-sheets-with-gpt-4o-7451


# Natural Language to SQL Converter for Google Sheets with GPT-4o

---

### 1. Workflow Overview

This workflow enables users to convert natural language questions into SQL queries specifically tailored for Google Sheets, leveraging GPT-4o via n8n’s LangChain integration. It is designed to interact with a designated Google Sheet, dynamically retrieving column metadata to inform query construction and then executing the generated SQL query against the sheet data. The response is finally formatted into a clear table structure.

**Target Use Cases:**  
- Business analysts or marketers querying Google Sheets data without manually writing SQL  
- Automating data retrieval from Google Sheets based on conversational inputs  
- Integrations requiring on-the-fly SQL query generation for spreadsheet data  

**Logical Blocks:**

- **1.1 Input Reception:** Captures user natural language queries via a chat trigger node.  
- **1.2 Context & Column Metadata Acquisition:** Retrieves Google Sheets column info and manages conversational memory for contextual understanding.  
- **1.3 AI SQL Query Generation:** Uses GPT-4o with system prompts and structured output parsing to generate Google Sheets-compatible SQL queries using column letters.  
- **1.4 Data Retrieval:** Executes the generated SQL query URL against the Google Sheet to retrieve raw data.  
- **1.5 Output Formatting:** Processes and formats the retrieved data into a structured table output.  
- **1.6 Supporting Nodes:** Includes sticky notes for documentation and setup guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming chat messages that will be converted into SQL queries.  
- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point webhook listening for chat messages to trigger workflow  
    - Configuration: Default with webhook ID set; no additional parameters  
    - Inputs: External webhook call (chat message)  
    - Outputs: Connects to AI Agent3 for processing  
    - Failure cases: Network interruptions, webhook misconfiguration, malformed requests  
    - Version: 1.3

#### 1.2 Context & Column Metadata Acquisition

- **Overview:** Provides AI context by retrieving Google Sheets column metadata and maintaining conversation memory for multi-turn interactions.  
- **Nodes Involved:**  
  - Get Column Info2  
  - Simple Memory2

- **Node Details:**  
  - **Get Column Info2**  
    - Type: `n8n-nodes-base.googleSheetsTool`  
    - Role: Extracts column names, data types, and descriptions from a specified Google Sheet tab to inform AI about available columns and their letters  
    - Configuration:  
      - Google Sheet ID: `19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA`  
      - Sheet Name (tab): `Columns` (ID: `467321788`)  
      - OAuth2 credentials configured for Google Sheets API access  
    - Inputs: None (triggered within workflow)  
    - Outputs: Provides column metadata to AI Agent3 as an AI tool input  
    - Failure cases: Authentication errors, API rate limits, incorrect sheet ID or tab name, permission issues  
    - Version: 4.7

  - **Simple Memory2**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains conversational context window for multi-turn dialogue  
    - Configuration: Default buffer window with no custom parameters  
    - Inputs: Feeds context into AI Agent3’s memory  
    - Outputs: Feeds AI Agent3 node to improve contextual responses  
    - Failure cases: Memory overflow if excessively large conversation history, potential memory desync  
    - Version: 1.3

#### 1.3 AI SQL Query Generation

- **Overview:** Uses GPT-4o with a detailed system prompt to generate a Google Sheets-specific SQL query based on the user question and column metadata. The output is parsed into a structured JSON containing the query URL.  
- **Nodes Involved:**  
  - AI Agent3  
  - OpenAI Chat Model1  
  - Structured Output Parser1

- **Node Details:**  
  - **AI Agent3**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Main AI orchestrator node that combines language model, tools, memory, and output parsing  
    - Configuration:  
      - System message instructs to always use column letters, generate full Google Sheets query URLs, and output JSON with a `query` property  
      - Tools connected: Google Sheets column info (Get Column Info2) and memory (Simple Memory2)  
      - Language model: OpenAI Chat Model1 (GPT-4o)  
      - Output parser: Structured Output Parser1  
    - Inputs: Triggered by chat message, enriched with column data and memory  
    - Outputs: Produces JSON with the SQL query URL for execution  
    - Failure cases: Model API timeouts or errors, malformed AI output, parsing errors if AI output deviates from schema  
    - Version: 2.2

  - **OpenAI Chat Model1**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides GPT-4o language generation capabilities for AI Agent3  
    - Configuration: Model set to `gpt-4o`, no extra options  
    - Inputs: Receives prompt and context from AI Agent3  
    - Outputs: Generated text responses forwarded to AI Agent3  
    - Failure cases: API authentication errors, rate limits, network issues  
    - Version: 1.2

  - **Structured Output Parser1**  
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - Role: Parses AI response to ensure JSON structure with a `query` key  
    - Configuration: JSON schema example requires `{ "query": "sql query for google sheets" }`  
    - Inputs: Raw AI output from AI Agent3  
    - Outputs: Validated and parsed JSON passed back to AI Agent3  
    - Failure cases: Parsing failure if AI output is malformed or incomplete  
    - Version: 1.3

#### 1.4 Data Retrieval

- **Overview:** Executes the Google Sheets SQL query URL generated by the AI and retrieves the corresponding CSV data output.  
- **Nodes Involved:**  
  - Get Data from Google Sheet

- **Node Details:**  
  - **Get Data from Google Sheet**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Sends HTTP GET request to the Google Sheets query URL to fetch CSV data  
    - Configuration:  
      - URL dynamically set from AI Agent3 output’s `query` property (`{{ $json.output.query }}`)  
      - No additional query parameters or headers needed  
    - Inputs: Receives query URL from AI Agent3  
    - Outputs: Raw CSV data of query results forwarded to next node  
    - Failure cases: Invalid or malformed URL, permission denied (sheet not shared publicly), network timeouts  
    - Version: 4.2

#### 1.5 Output Formatting

- **Overview:** Formats the raw data retrieved into a clean table output, separating dimensions and metrics as requested.  
- **Nodes Involved:**  
  - Write into Table Output1  
  - OpenAI Chat Model2

- **Node Details:**  
  - **Write into Table Output1**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Processes the raw data string, formatting it into an organized table with dimensions and metrics columns  
    - Configuration:  
      - System prompt instructs to output results as dimensions first, then metrics  
      - Text input taken from the HTTP request node’s `data` property  
      - Uses OpenAI Chat Model2 as language model backend  
    - Inputs: Receives raw CSV data from HTTP request node  
    - Outputs: Final formatted table output  
    - Failure cases: Formatting errors if input data is malformed or empty, API errors  
    - Version: 2.2

  - **OpenAI Chat Model2**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: GPT-4.1-mini model used to format the output data into the requested table structure  
    - Configuration: Model set to `gpt-4.1-mini`  
    - Inputs: Data string from HTTP request node via Write into Table Output1 agent  
    - Outputs: Formatted textual table output  
    - Failure cases: API errors or throttling  
    - Version: 1.2

#### 1.6 Supporting Nodes (Documentation & Setup)

- **Overview:** Contains sticky notes offering setup instructions, contact info, and usage guidance for users or developers.  
- **Nodes Involved:**  
  - Sticky Note5  
  - Sticky Note12  
  - Sticky Note13

- **Node Details:**  
  - **Sticky Note5**  
    - Content: Contact information for help and customization requests with email and LinkedIn links  
    - Position: Prominent location for user reference  
  - **Sticky Note12 & Sticky Note13**  
    - Content: Detailed step-by-step setup guide covering OpenAI API keys, Google Sheets OAuth2 setup, sheet preparation, and system prompt customization  
    - Includes direct links to OpenAI platform, Google Cloud Console, and sample Google Sheet  
    - Key for onboarding and replicating the workflow  
  - Failure cases: None (documentation only)

---

### 3. Summary Table

| Node Name                 | Node Type                                          | Functional Role                          | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                 |
|---------------------------|---------------------------------------------------|----------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger              | Entry point to receive user queries    | External webhook             | AI Agent3                   |                                                                                             |
| AI Agent3                 | @n8n/n8n-nodes-langchain.agent                     | Generate SQL query using GPT-4o        | When chat message received, Get Column Info2, Simple Memory2, OpenAI Chat Model1, Structured Output Parser1 | Get Data from Google Sheet   |                                                                                             |
| Get Column Info2          | n8n-nodes-base.googleSheetsTool                     | Retrieve Google Sheets column metadata | AI Agent3 (ai_tool input)    | AI Agent3                   | Sticky Note12 & Sticky Note13 detail setup instructions for this node                       |
| Simple Memory2            | @n8n/n8n-nodes-langchain.memoryBufferWindow        | Maintain conversational context        |                             | AI Agent3                   |                                                                                             |
| OpenAI Chat Model1        | @n8n/n8n-nodes-langchain.lmChatOpenAi              | GPT-4o model for query generation      | AI Agent3 (ai_languageModel) | AI Agent3                   | Sticky Note12 & Sticky Note13 provide instructions for OpenAI API credential setup          |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured    | Parse AI output into JSON with query   | AI Agent3                   | AI Agent3                   |                                                                                             |
| Get Data from Google Sheet| n8n-nodes-base.httpRequest                           | Execute SQL query URL to fetch data    | AI Agent3                   | Write into Table Output1    | Sticky Note12 & Sticky Note13 mention sheet sharing and URL setup                          |
| Write into Table Output1  | @n8n/n8n-nodes-langchain.agent                      | Format retrieved data into table       | Get Data from Google Sheet   |                             |                                                                                             |
| OpenAI Chat Model2        | @n8n/n8n-nodes-langchain.lmChatOpenAi              | GPT-4.1-mini model for output formatting| Write into Table Output1     |                             | Sticky Note12 & Sticky Note13 provide OpenAI setup details                                 |
| Sticky Note5              | n8n-nodes-base.stickyNote                            | Contact info & support                  |                             |                             | 📬 Need Help or Want to Customize This? 📧 robert@ynteractive.com 🔗 LinkedIn profile link  |
| Sticky Note12             | n8n-nodes-base.stickyNote                            | Step-by-step setup instructions        |                             |                             | Detailed instructions for OpenAI & Google Sheets setup, sample sheet usage, prompt update  |
| Sticky Note13             | n8n-nodes-base.stickyNote                            | Google Sheet preparation & configuration|                             |                             | Sample data sheet usage, sheet permissions, and system prompt customization                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `@n8n/n8n-nodes-langchain.chatTrigger` node named `When chat message received`.  
   - Keep default settings; note webhook ID will be auto-generated.

2. **Set Up Google Sheets Column Metadata Node:**  
   - Add `n8n-nodes-base.googleSheetsTool` node named `Get Column Info2`.  
   - Configure:  
     - Document ID: Your Google Sheet ID (e.g. `19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA`)  
     - Sheet Name: Tab containing column metadata (e.g., "Columns")  
   - Create and assign Google Sheets OAuth2 credentials with appropriate scopes and authorized redirect URIs.

3. **Add Conversation Memory Node:**  
   - Add `@n8n/n8n-nodes-langchain.memoryBufferWindow` node named `Simple Memory2`.  
   - Use default settings.

4. **Add Language Model Node for GPT-4o:**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node named `OpenAI Chat Model1`.  
   - Select model `gpt-4o`.  
   - Configure OpenAI API credentials by creating new API key from OpenAI platform and entering into n8n.

5. **Add Structured Output Parser Node:**  
   - Add `@n8n/n8n-nodes-langchain.outputParserStructured` node named `Structured Output Parser1`.  
   - Enter JSON schema example: `{ "query": "sql query for google sheets" }`.

6. **Configure AI Agent Node:**  
   - Add `@n8n/n8n-nodes-langchain.agent` node named `AI Agent3`.  
   - Set system message to instruct AI to:  
     - Always use column letters for SQL queries  
     - Include full Google Sheets URL with gid and query string  
     - Example output format as JSON with `query` key  
   - Connect `Get Column Info2` as an AI tool input.  
   - Connect `Simple Memory2` as AI memory.  
   - Connect `OpenAI Chat Model1` as AI language model.  
   - Connect `Structured Output Parser1` as output parser.

7. **Create HTTP Request Node to Retrieve Data:**  
   - Add `n8n-nodes-base.httpRequest` node named `Get Data from Google Sheet`.  
   - Set URL to `={{ $json.output.query }}` (dynamic from AI Agent3 output).  
   - Leave other HTTP options default.

8. **Add Language Model Node for Formatting Output:**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node named `OpenAI Chat Model2`.  
   - Select model `gpt-4.1-mini`.  
   - Use same OpenAI API credentials as before.

9. **Add Formatting Agent Node:**  
   - Add `@n8n/n8n-nodes-langchain.agent` node named `Write into Table Output1`.  
   - Set prompt text to `={{ $json.data }}` to input raw CSV data.  
   - Set system message to "write this into one table. output as dimensions, then metrics".  
   - Connect `OpenAI Chat Model2` as language model.

10. **Connect Nodes in Order:**  
    - `When chat message received` → `AI Agent3`  
    - `Get Column Info2` → `AI Agent3` (ai_tool input)  
    - `Simple Memory2` → `AI Agent3` (ai_memory input)  
    - `OpenAI Chat Model1` → `AI Agent3` (ai_languageModel input)  
    - `Structured Output Parser1` → `AI Agent3` (ai_outputParser input)  
    - `AI Agent3` → `Get Data from Google Sheet`  
    - `Get Data from Google Sheet` → `Write into Table Output1`  
    - `OpenAI Chat Model2` → `Write into Table Output1`  

11. **Set Up Credentials:**  
    - OpenAI: Create API key from https://platform.openai.com/, add billing info, and configure in both OpenAI nodes.  
    - Google Sheets: Enable Google Sheets API in Google Cloud Console, create OAuth2 Client ID credentials, set redirect URIs as per n8n instructions, and assign to `Get Column Info2`.

12. **Prepare Google Sheet:**  
    - Use the sample sheet [Sample Marketing Data](https://docs.google.com/spreadsheets/d/19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA/edit?usp=drivesdk) or your own.  
    - Ensure a tab named "Columns" with column metadata exists.  
    - Share sheet with permission "Anyone with the link can view" to allow HTTP requests.  
    - Update system prompt URL and gid in `AI Agent3` if using a different sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| 📬 Need Help or Want to Customize This? Contact robert@ynteractive.com or visit LinkedIn profile.               | Sticky Note5                                                                                              |
| Step-by-step setup guide for OpenAI API keys, Google Sheets OAuth2 credentials, and sample sheet preparation.  | Sticky Note12 & Sticky Note13                                                                             |
| Sample Google Sheet used: [Sample Marketing Data](https://docs.google.com/spreadsheets/d/19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA/edit?usp=drivesdk) | Used for testing and reference sheet structure                                                           |
| OpenAI API platform for key creation and billing setup: https://platform.openai.com/                           | Setup resource                                                                                           |
| Google Cloud Console for enabling Sheets API and creating OAuth2 credentials: https://console.cloud.google.com/| Setup resource                                                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---