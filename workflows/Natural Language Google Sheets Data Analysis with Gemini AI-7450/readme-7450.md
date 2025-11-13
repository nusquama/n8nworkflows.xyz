Natural Language Google Sheets Data Analysis with Gemini AI

https://n8nworkflows.xyz/workflows/natural-language-google-sheets-data-analysis-with-gemini-ai-7450


# Natural Language Google Sheets Data Analysis with Gemini AI

### 1. Workflow Overview

This workflow, titled **Natural Language Google Sheets Data Analysis with Gemini AI**, enables users to query a Google Sheets dataset using natural language. It leverages Google Gemini AI for interpreting user questions and generating precise data aggregation queries, which are then executed on the spreadsheet data to produce summarized insights. The workflow is structured to:

- Parse natural language queries into structured aggregation instructions.
- Retrieve metadata about spreadsheet columns for accurate query formulation.
- Execute data aggregation operations (sum, average, count, unique count, min, max).
- Aggregate and format the results into a readable table.
- Provide an extendable sub-workflow for detailed data processing and output formatting.

Logical blocks:

- **1.1 Input Reception and AI Query Parsing**: Receives natural language input, manages conversational memory, and parses it into structured aggregation queries using Google Gemini AI.
- **1.2 Column Metadata Retrieval**: Fetches and profiles column information from the Google Sheet to guide query parsing and validation.
- **1.3 Data Summarization and Aggregation**: Executes aggregation operations based on AI-parsed instructions, including conditional routing for aggregation types.
- **1.4 Result Aggregation and Output Formatting**: Combines summarized data and formats it via AI for user-friendly presentation.
- **1.5 Sub-Workflow Execution**: Invokes a separate sub-workflow to process and summarize data further, promoting modularity.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and AI Query Parsing

- **Overview:**  
  This block manages user input and parses it into structured JSON instructions describing which column to analyze, what aggregation to perform, and the grouping level. It uses conversational memory to maintain context and the Google Gemini Chat model to interpret natural language questions into explicit commands.

- **Nodes Involved:**  
  - `Simple Memory`  
  - `Google Gemini Chat Model`  
  - `Structured Output Parser`  
  - `Talk to Data with Gemini`  

- **Node Details:**  

  - **Simple Memory**  
    - *Type:* Langchain memory buffer (sliding window)  
    - *Role:* Maintains short-term conversational context across queries.  
    - *Config:* Default, no parameters customized.  
    - *Connections:* Output feeds into `Talk to Data with Gemini` as AI memory.  
    - *Edge Cases:* Memory overflow or excessive accumulation could degrade response relevance.

  - **Google Gemini Chat Model**  
    - *Type:* Langchain Google Gemini AI chat model node  
    - *Role:* Core AI engine interpreting queries.  
    - *Config:* Uses Google PaLM API credential for authentication, no special options set.  
    - *Connections:* Linked as the language model in `Talk to Data with Gemini`.  
    - *Edge Cases:* API quota limits, auth failure, or network timeouts.

  - **Structured Output Parser**  
    - *Type:* Langchain structured output parser  
    - *Role:* Parses AI outputs into a JSON schema specifying:  
      ```json
      {
        "column": "customer",
        "aggregation": "sum",
        "level": "date, channel"
      }
      ```  
    - *Config:* Schema example provided to enforce output format.  
    - *Connections:* Output parser linked in `Talk to Data with Gemini`.  
    - *Edge Cases:* Parsing errors if AI output deviates from schema.

  - **Talk to Data with Gemini**  
    - *Type:* Langchain Agent  
    - *Role:* Orchestrates the AI conversation and tool usage; applies system prompt to constrain AI behavior to Google Sheets data queries only.  
    - *Config:*  
      - System prompt enforces use of column metadata tool before querying data.  
      - Restricts aggregation functions to sum, avg, count, countd, max, min.  
      - Outputs structured JSON with columns: `column`, `aggregation`, `level`.  
    - *Connections:*  
      - Inputs: `Simple Memory` (memory), `Get Column Info` (tool), `Google Gemini Chat Model` (language model), `Structured Output Parser` (output parser).  
      - Outputs go to `Execute Workflow - Summarize Data` node.  
    - *Edge Cases:*  
      - AI hallucinations or misunderstanding could cause invalid aggregations.  
      - Dependency on accurate column metadata is critical.

---

#### 1.2 Column Metadata Retrieval

- **Overview:**  
  This block extracts detailed metadata from the target Google Sheet’s column headers and data profiling to inform AI tools and ensure accurate query parsing.

- **Nodes Involved:**  
  - `Get Column Info`  

- **Node Details:**  

  - **Get Column Info**  
    - *Type:* Google Sheets Tool (specialized)  
    - *Role:* Extracts column names, inferred data types, human-readable descriptions, and profiling hints from a specified sheet tab.  
    - *Config:*  
      - Document ID set to sample marketing data sheet.  
      - Sheet name set to "Columns" tab (gid=467321788).  
      - Uses OAuth2 credential for Google Sheets access.  
    - *Connections:*  
      - Output is linked as a tool input for `Talk to Data with Gemini`.  
    - *Edge Cases:*  
      - Credential expiration or permission errors.  
      - Sheet structure changes breaking metadata extraction.

---

#### 1.3 Data Summarization and Aggregation

- **Overview:**  
  This block executes the actual data aggregation based on AI-provided instructions. It routes data through conditional nodes that select the correct aggregation operation and then aggregates the results.

- **Nodes Involved:**  
  - `Execute Workflow - Summarize Data` (Sub-workflow trigger)  

- **Node Details:**  

  - **Execute Workflow - Summarize Data**  
    - *Type:* Execute Workflow (sub-workflow invocation)  
    - *Role:* Runs a dedicated sub-workflow that performs data retrieval and summarization according to AI instructions.  
    - *Config:*  
      - Accepts parameters from the main workflow: aggregation type, column, and grouping level.  
      - Contains an embedded workflow JSON defining detailed data operations (see below).  
    - *Connections:*  
      - Input from `Talk to Data with Gemini`.  
      - No direct output connections shown here (handled inside sub-workflow).  
    - *Edge Cases:*  
      - Workflow invocation failures, parameter mismatches.  
      - Sub-workflow credential and configuration dependencies.

---

#### 1.3.1 Sub-Workflow: Data Processor

- **Overview:**  
  The sub-workflow executes the data query on the specified sheet, performs conditional aggregation routing, summarizes data accordingly, aggregates results, and formats output via AI.

- **Nodes Involved:**  
  - `When Executed by Another Workflow` (trigger)  
  - `Get Data` (Google Sheets data retrieval)  
  - `Type of Aggregation` (switch node to route aggregation)  
  - Multiple `Summarize` nodes for each aggregation type: Count Unique, Sum, Average, Min, Max, Count  
  - `Bring All Data together` (aggregate node)  
  - `Write into Table Output` (Langchain agent for formatting)  
  - `Google Gemini Chat Model1` (AI model for final formatting)  
  - `AI Agent1` (Langchain agent node for final table output)  

- **Node Details:**  

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point for the sub-workflow; receives parameters from the main workflow.  
    - *Connections:* Outputs to `Get Data`.

  - **Get Data**  
    - *Type:* Google Sheets  
    - *Role:* Retrieves rows from the target sheet tab "Data" (gid=365710158) of the sample marketing data sheet.  
    - *Config:* Uses OAuth2 credentials; no filters applied, fetches entire sheet content.  
    - *Connections:* Outputs to `Type of Aggregation`.

  - **Type of Aggregation**  
    - *Type:* Switch  
    - *Role:* Routes flow based on the aggregation type parsed by AI (`countd`, `sum`, `avg`, `min`, `max`, `count`).  
    - *Config:* Uses expressions to match aggregation type from workflow input parameters.  
    - *Connections:* Routes to respective `Summarize` nodes.

  - **Summarize Nodes (Count Unique, Sum, Average, Min, Max, Count)**  
    - *Type:* Summarize (n8n native)  
    - *Role:* Perform specific aggregation on the target column, grouped by levels specified by AI.  
    - *Config:*  
      - Field to summarize is dynamically set by expression referencing workflow input.  
      - Grouping fields set by AI-provided "level" parameter.  
    - *Connections:* All output to `Bring All Data together`.

  - **Bring All Data together**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all summarized data items into a single dataset for formatting.  
    - *Connections:* Outputs to `Write into Table Output`.

  - **Write into Table Output**  
    - *Type:* Langchain Agent  
    - *Role:* Uses AI to format the aggregated data into a readable table format.  
    - *Config:*  
      - System prompt instructs to write output as dimensions then metrics.  
      - Input text set to the aggregated data JSON.  
    - *Connections:*  
      - Linked to `Google Gemini Chat Model1` as language model.  
      - Outputs the final formatted result.

  - **Google Gemini Chat Model1**  
    - *Type:* Langchain Google Gemini AI Chat Model  
    - *Role:* Provides AI-powered text generation for final output formatting.  
    - *Config:* Uses same Google PaLM API credential.  
    - *Connections:* Feeds AI output to `Write into Table Output`.

  - **AI Agent1**  
    - *Type:* Langchain Agent  
    - *Role:* Final agent to generate the user-presentable table output from the aggregated data.  
    - *Config:* Receives text and system prompt from previous node.  
    - *Connections:* Final output node in sub-workflow.

- **Edge Cases:**  
  - Google Sheets API rate limits or failures.  
  - Mismatch in aggregation types or field names causing empty or erroneous summarizations.  
  - AI formatting failures or unexpected output errors.

---

#### 1.4 Result Aggregation and Output Formatting

- **Overview:**  
  This block collects all summarized data, aggregates it into a unified output, and uses AI to produce a clean table for user consumption.

- **Nodes Involved:**  
  - `Bring All Data together`  
  - `Write into Table Output`  
  - `Google Gemini Chat Model1`  
  - `AI Agent1`  

- **Node Details:**  
  Covered above as part of the sub-workflow.

---

#### 1.5 Informational Sticky Notes

- **Overview:**  
  Multiple sticky notes provide user instructions, setup guides, example queries, and contact info. These do not affect workflow execution but are crucial for user onboarding and customization.

- **Nodes Involved:**  
  - `Sticky Note`  
  - `Sticky Note1`  
  - `Sticky Note3`  
  - `Sticky Note4`  

- **Node Details:**  

  - **Sticky Note**  
    - Content: Contact info for support and customization requests.  
    - Position: Top left, visually separate.

  - **Sticky Note1**  
    - Content: Examples of aggregate-only pivot questions to guide users on query types supported.  
    - Includes metrics and dimension breakdown examples.

  - **Sticky Note3**  
    - Content: Setup instructions for the sub-workflow, including credential application and linking to main workflow.

  - **Sticky Note4**  
    - Content: Main workflow setup prerequisites, detailed instructions for configuring Google Gemini API and Google Sheets API, sample data usage, and triggering tips.  
    - Contains multiple helpful links:  
      - Google AI Studio: https://aistudio.google.com/  
      - Google Cloud Console: https://console.cloud.google.com/  
      - Sample Marketing Data sheet link: https://docs.google.com/spreadsheets/d/19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA/edit?usp=drivesdk  

- **Edge Cases:**  
  - User misconfiguration due to incomplete following of instructions.  
  - Credential expiration or incorrect API key setup.

---

### 3. Summary Table

| Node Name                   | Node Type                                       | Functional Role                        | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                     |
|-----------------------------|------------------------------------------------|-------------------------------------|----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note                                     | Support contact info                 |                                  |                                  | 📬 Need Help or Want to Customize This? Contact robert@ynteractive.com & LinkedIn profile       |
| Google Gemini Chat Model    | Langchain Google Gemini AI Chat Model           | AI language model for main query    |                                  | Talk to Data with Gemini          |                                                                                                |
| Get Column Info            | Google Sheets Tool                              | Extracts column metadata             |                                  | Talk to Data with Gemini          |                                                                                                |
| When Executed by Another Workflow | Execute Workflow Trigger                      | Entry trigger for sub-workflow      | Execute Workflow - Summarize Data | Get Data                         |                                                                                                |
| Structured Output Parser    | Langchain Structured Output Parser              | Parses AI output into JSON           |                                  | Talk to Data with Gemini          |                                                                                                |
| Google Gemini Chat Model1   | Langchain Google Gemini AI Chat Model           | AI model for final formatting        | Write into Table Output           | Write into Table Output           |                                                                                                |
| Simple Memory              | Langchain Memory Buffer Window                   | Maintains conversational context    |                                  | Talk to Data with Gemini          |                                                                                                |
| Sticky Note1                | Sticky Note                                     | Example aggregate queries            |                                  |                                  | 📊 Aggregate-Only Pivot Questions — One Metric at a Time (with detailed examples)               |
| Sticky Note3                | Sticky Note                                     | Sub-workflow setup instructions     |                                  |                                  | ⚙️ SUB-WORKFLOW - Data Processor setup guide                                                  |
| Sticky Note4                | Sticky Note                                     | Main workflow setup guide            |                                  |                                  | 📋 MAIN WORKFLOW - Query Parser prerequisites and setup instructions with links               |
| Talk to Data with Gemini    | Langchain Agent                                 | Parses user query into aggregation   | Simple Memory, Get Column Info, Google Gemini Chat Model, Structured Output Parser | Execute Workflow - Summarize Data |                                                                                                |
| Execute Workflow - Summarize Data | Execute Workflow                              | Runs sub-workflow to summarize data | Talk to Data with Gemini          |                                  |                                                                                                |
| Get Data                   | Google Sheets                                   | Retrieves data rows                  | When Executed by Another Workflow | Type of Aggregation              |                                                                                                |
| Type of Aggregation        | Switch                                          | Routes flow by aggregation type     | Get Data                         | Count Unique, Sum, Average, Min, Max, Count |                                                                                                |
| Count Unique               | Summarize                                       | Count distinct values aggregation   | Type of Aggregation              | Bring All Data together          |                                                                                                |
| Sum                        | Summarize                                       | Sum aggregation                     | Type of Aggregation              | Bring All Data together          |                                                                                                |
| Average                    | Summarize                                       | Average aggregation                 | Type of Aggregation              | Bring All Data together          |                                                                                                |
| Min                        | Summarize                                       | Minimum value aggregation           | Type of Aggregation              | Bring All Data together          |                                                                                                |
| Max                        | Summarize                                       | Maximum value aggregation           | Type of Aggregation              | Bring All Data together          |                                                                                                |
| Count                      | Summarize                                       | Count aggregation                   | Type of Aggregation              | Bring All Data together          |                                                                                                |
| Bring All Data together    | Aggregate                                       | Combines summarized data            | Count Unique, Sum, Average, Min, Max, Count | Write into Table Output          |                                                                                                |
| Write into Table Output    | Langchain Agent                                 | Formats aggregated data into table  | Bring All Data together, Google Gemini Chat Model1 |                              |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes** (optional for user guidance):  
   - Add notes with contact info, example queries, and setup instructions as per Sticky Note nodes content.

2. **Setup Credentials:**  
   - Create Google PaLM API credential for Google Gemini Chat Model nodes (use API key from Google AI Studio).  
   - Create Google Sheets OAuth2 credential (OAuth client ID from Google Cloud Console) for all Google Sheets nodes.

3. **Build Input Reception Block:**  
   - Add a **Simple Memory** node (Langchain memory buffer) with default parameters to hold conversation context.  
   - Add a **Google Gemini Chat Model** node configured with Google PaLM API credential.  
   - Add a **Structured Output Parser** node configured with a JSON schema example specifying keys: `column`, `aggregation`, `level`.  
   - Add a **Google Sheets Tool** node named `Get Column Info` configured to read the “Columns” sheet from your Google Sheet document to extract column metadata.  
   - Add a **Langchain Agent** node named `Talk to Data with Gemini` configured with:  
     - System message prompt enforcing: use column metadata tool first, limited aggregations, output JSON with `column`, `aggregation`, `level`.  
     - Connect `Simple Memory` as AI memory, `Get Column Info` as tool, `Google Gemini Chat Model` as language model, and `Structured Output Parser` as output parser.  
   - Connect `Talk to Data with Gemini` main output to the sub-workflow execution node.

4. **Create Sub-Workflow (Data Processor):**  
   - Add an **Execute Workflow Trigger** node `When Executed by Another Workflow` accepting input from main workflow.  
   - Add a **Google Sheets** node `Get Data` to read the “Data” tab from your Google Sheet with OAuth2 credentials.  
   - Add a **Switch** node `Type of Aggregation` evaluating the `aggregation` parameter passed from the main workflow, with cases: `countd`, `sum`, `avg`, `min`, `max`, `count`.  
   - Create separate **Summarize** nodes for each aggregation type:  
     - Configure each to summarize the specified column and group by the `level` parameter.  
   - Connect outputs of all Summarize nodes to an **Aggregate** node `Bring All Data together` to combine results.  
   - Add a **Langchain Agent** node `Write into Table Output` with:  
     - Input text bound to aggregated data.  
     - System message prompting to format output as dimensions then metrics in a table.  
     - Connect to a **Google Gemini Chat Model1** node for AI formatting.  
   - Connect `Google Gemini Chat Model1` back to `Write into Table Output` as language model.  
   - The final output from `Write into Table Output` is the sub-workflow’s result.

5. **Connect Main and Sub-Workflow:**  
   - In the main workflow, add an **Execute Workflow** node configured to call the sub-workflow.  
   - Pass AI-parsed parameters (`column`, `aggregation`, `level`) as inputs to the sub-workflow.

6. **Configure Triggers:**  
   - Add a manual or webhook trigger to start the main workflow.  
   - The user’s natural language query is input to the `Talk to Data with Gemini` node.

7. **Test and Validate:**  
   - Use sample marketing data or your own Google Sheet with similar schema.  
   - Test with example queries such as “Show total Spend by Channel.”  
   - Confirm outputs are structured and formatted tables.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| Contact for help or customization: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ | Sticky Note contact info                                                                       |
| Instructions to set up Google Gemini API: https://aistudio.google.com/                                         | Provided in Sticky Note4                                                                       |
| Instructions to set up Google Sheets API and OAuth2: https://console.cloud.google.com/                          | Provided in Sticky Note4                                                                       |
| Sample Marketing Data Google Sheet link: https://docs.google.com/spreadsheets/d/19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA/edit?usp=drivesdk | Used as default dataset                                                                        |
| Aggregate-only pivot question examples to guide user queries                                                   | Sticky Note1 content                                                                           |
| Sub-workflow configuration and linking instructions                                                           | Sticky Note3                                                                                   |

---

**Disclaimer:** The text and workflow details provided are extracted exclusively from an automated n8n workflow. The content respects all current content policies, contains no illegal or protected data, and uses only public and legal information.