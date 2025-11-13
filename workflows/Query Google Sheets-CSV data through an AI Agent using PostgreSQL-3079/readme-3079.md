Query Google Sheets/CSV data through an AI Agent using PostgreSQL

https://n8nworkflows.xyz/workflows/query-google-sheets-csv-data-through-an-ai-agent-using-postgresql-3079


# Query Google Sheets/CSV data through an AI Agent using PostgreSQL

### 1. Workflow Overview

This workflow enables querying structured financial or tabular data stored in Google Sheets or CSV files by leveraging an AI agent that dynamically generates optimized PostgreSQL queries from natural language questions. It is designed to overcome limitations of vector database approaches for numerical data by using PostgreSQL as a backend for efficient storage and querying.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Data Fetching**: Detects changes in Google Sheets via a trigger, fetches sheet data, and sets configuration parameters.
- **1.2 Database Table Management**: Checks if a corresponding PostgreSQL table exists, creates or drops tables dynamically based on the data schema inferred from the sheet.
- **1.3 Data Insertion**: Formats and inserts the fetched sheet data into the PostgreSQL table.
- **1.4 AI Agent Query Handling**: Listens for natural language queries, uses an AI agent to generate SQL queries based on the database schema, executes these queries, and returns results.
- **1.5 Schema Retrieval and Formatting**: Retrieves and formats the PostgreSQL database schema to assist the AI agent in query generation.
- **1.6 Sub-Workflows for Query Execution and Schema Retrieval**: Separate workflows invoked as tools by the AI agent to execute SQL queries and fetch database schema.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Fetching

- **Overview**: This block triggers when a specific Google Sheet file changes, sets key parameters such as the sheet URL and name, and fetches the sheet data for processing.
- **Nodes Involved**:  
  - Google Drive Trigger  
  - change_this (Set node)  
  - fetch sheet data (Google Sheets node)  

- **Node Details**:

  - **Google Drive Trigger**  
    - *Type*: Trigger node for Google Drive file changes  
    - *Configuration*: Watches a specific Google Sheet file by its ID for changes.  
    - *Credentials*: Uses Google Drive OAuth2 credentials.  
    - *Input/Output*: No input; outputs trigger data when the file changes.  
    - *Edge Cases*: Requires correct file ID and OAuth2 credentials; failure if file is inaccessible or permissions insufficient.

  - **change_this**  
    - *Type*: Set node  
    - *Configuration*: Sets two string parameters: `table_url` (Google Sheet URL) and `sheet_name` (sheet tab name, e.g., "product_list").  
    - *Input/Output*: Receives trigger data; outputs these parameters for downstream nodes.  
    - *Edge Cases*: Must update values to match user’s actual sheet URL and tab name.

  - **fetch sheet data**  
    - *Type*: Google Sheets node  
    - *Configuration*: Reads data from the specified sheet tab and document ID (taken from `change_this` node).  
    - *Credentials*: Uses Google Sheets OAuth2 credentials.  
    - *Input/Output*: Input from `table exists?` node; outputs all rows as JSON objects.  
    - *Edge Cases*: Requires correct sheet name and document ID; failure if sheet is private or credentials invalid.

---

#### 2.2 Database Table Management

- **Overview**: Checks if a PostgreSQL table corresponding to the sheet exists, drops it if necessary, and creates a new table based on the inferred schema from the sheet data.
- **Nodes Involved**:  
  - table exists? (PostgreSQL node)  
  - is not in database (If node)  
  - remove table (PostgreSQL node)  
  - create table query (Code node)  
  - create table (PostgreSQL node)  

- **Node Details**:

  - **table exists?**  
    - *Type*: PostgreSQL node  
    - *Configuration*: Executes a query to check if a table named `ai_table_<sheet_name>` exists in the database.  
    - *Input/Output*: Input from `change_this`; outputs boolean `exists`.  
    - *Edge Cases*: Requires valid PostgreSQL credentials; failure if connection issues or permission denied.

  - **is not in database**  
    - *Type*: If node  
    - *Configuration*: Checks if `table exists?` returned false (table does not exist).  
    - *Input/Output*: Input from `fetch sheet data`; outputs two branches: true (table does not exist) and false (table exists).  
    - *Edge Cases*: Expression evaluation errors if input missing.

  - **remove table**  
    - *Type*: PostgreSQL node  
    - *Configuration*: Drops the table `ai_table_<sheet_name>` if it exists, cascading to dependent objects.  
    - *Input/Output*: Triggered when table exists but needs replacement.  
    - *Edge Cases*: Requires proper permissions; failure if table locked or connection issues.

  - **create table query**  
    - *Type*: Code node (JavaScript)  
    - *Configuration*:  
      - Dynamically infers column names and types from sheet data.  
      - Detects currency columns by symbols and assigns `DECIMAL(15,2)`.  
      - Detects date columns in MM/DD/YYYY format and assigns `TIMESTAMP`.  
      - Generates a `CREATE TABLE IF NOT EXISTS` SQL statement with a UUID primary key column `ai_table_identifier`.  
      - Outputs the SQL query string and a column mapping object for later use.  
    - *Input/Output*: Input from `is not in database` (true branch); outputs SQL query and mapping.  
    - *Edge Cases*: Assumes consistent column headers; may misclassify types if data inconsistent.

  - **create table**  
    - *Type*: PostgreSQL node  
    - *Configuration*: Executes the SQL query generated by `create table query` to create the table.  
    - *Input/Output*: Input from `create table query`; outputs success or error.  
    - *Edge Cases*: Requires PostgreSQL permissions; failure if query malformed or connection issues.

---

#### 2.3 Data Insertion

- **Overview**: Formats the fetched sheet data according to inferred column types and inserts it into the PostgreSQL table.
- **Nodes Involved**:  
  - create insertion query (Code node)  
  - perform insertion (PostgreSQL node)  

- **Node Details**:

  - **create insertion query**  
    - *Type*: Code node (JavaScript)  
    - *Configuration*:  
      - Uses the column mapping from `create table query` to rename columns and infer types.  
      - Normalizes currency values by stripping symbols and converting to numbers.  
      - Parses date strings into ISO format.  
      - Converts percentages to decimals.  
      - Handles null, boolean, numeric, and text types appropriately.  
      - Constructs a parameterized SQL `INSERT INTO` statement with placeholders for all rows and columns.  
      - Outputs the query string and parameters array for PostgreSQL insertion.  
    - *Input/Output*: Input from `create table`; outputs query and parameters.  
    - *Edge Cases*: Data inconsistencies may cause null insertions; large datasets may hit parameter limits.

  - **perform insertion**  
    - *Type*: PostgreSQL node  
    - *Configuration*: Executes the parameterized insert query with provided parameters.  
    - *Input/Output*: Input from `create insertion query`; outputs success or error.  
    - *Edge Cases*: Requires PostgreSQL permissions; failure if parameter count exceeds limits or data type mismatches.

---

#### 2.4 AI Agent Query Handling

- **Overview**: Listens for natural language chat messages, uses an AI agent to generate SQL queries based on the database schema, executes those queries, and returns formatted results.
- **Nodes Involved**:  
  - When chat message received (Manual Chat Trigger)  
  - AI Agent With SQL Query Prompt (LangChain Agent node)  
  - execute_query_tool (Tool Workflow node)  
  - get_postgres_schema (Tool Workflow node)  
  - Google Gemini Chat Model (LLM node)  
  - sql query executor (PostgreSQL node)  
  - response output (Set node)  

- **Node Details**:

  - **When chat message received**  
    - *Type*: Manual Chat Trigger (LangChain)  
    - *Configuration*: Waits for user input messages to start the AI query process.  
    - *Input/Output*: No input; outputs chat message JSON.  
    - *Edge Cases*: Requires LangChain integration; failure if trigger misconfigured.

  - **AI Agent With SQL Query Prompt**  
    - *Type*: LangChain Agent node  
    - *Configuration*:  
      - System message defines role as a Database Query Assistant specializing in PostgreSQL.  
      - Uses two tools: `get_postgres_schema` and `execute_query_tool`.  
      - Implements a multi-step process: analyze question, fetch schema, build query, execute, and format results.  
      - Enforces best practices like parameterized queries and error handling.  
      - Uses Google Gemini Chat Model as the language model backend.  
    - *Input/Output*: Input from chat trigger and tool workflows; outputs SQL query JSON.  
    - *Edge Cases*: AI model errors, schema retrieval failures, or query execution errors.

  - **execute_query_tool**  
    - *Type*: Tool Workflow node  
    - *Configuration*: Invokes a separate workflow named `query_executer` to execute SQL queries.  
    - *Input/Output*: Receives SQL query JSON; outputs query results.  
    - *Edge Cases*: Requires the sub-workflow to be deployed and accessible.

  - **get_postgres_schema**  
    - *Type*: Tool Workflow node  
    - *Configuration*: Invokes a separate workflow named `get database schema` to retrieve the database schema as a formatted string.  
    - *Input/Output*: No input parameters; outputs schema string.  
    - *Edge Cases*: Requires the sub-workflow to be deployed and accessible.

  - **Google Gemini Chat Model**  
    - *Type*: LangChain LLM node  
    - *Configuration*: Uses Google Gemini 2.0 Flash model for natural language processing.  
    - *Credentials*: Google Palm API credentials required.  
    - *Input/Output*: Connected as AI language model for the agent node.  
    - *Edge Cases*: API quota limits, network errors.

  - **sql query executor**  
    - *Type*: PostgreSQL node  
    - *Configuration*: Executes the SQL query generated by the AI agent.  
    - *Input/Output*: Input from `Execute Workflow Trigger`; outputs query results.  
    - *Edge Cases*: Query syntax errors, permission issues.

  - **response output**  
    - *Type*: Set node  
    - *Configuration*: Sets the final response string to be returned to the user.  
    - *Input/Output*: Input from `sql query executor`; outputs formatted response.  
    - *Edge Cases*: None significant.

---

#### 2.5 Schema Retrieval and Formatting

- **Overview**: Retrieves the PostgreSQL database schema (tables and columns) and formats it into a readable string for the AI agent.
- **Nodes Involved**:  
  - schema finder (PostgreSQL node)  
  - schema to string (Code node)  

- **Node Details**:

  - **schema finder**  
    - *Type*: PostgreSQL node  
    - *Configuration*: Queries PostgreSQL system catalogs to get all tables and their columns with data types in the public schema.  
    - *Input/Output*: Input from `Execute Workflow Trigger`; outputs raw schema rows.  
    - *Edge Cases*: Requires read access to system catalogs.

  - **schema to string**  
    - *Type*: Code node (JavaScript)  
    - *Configuration*:  
      - Transforms raw schema rows into a formatted string describing each table and its columns with types.  
      - Output is a single string with tables separated by double newlines.  
    - *Input/Output*: Input from `schema finder`; outputs formatted schema string.  
    - *Edge Cases*: None significant.

---

#### 2.6 Sub-Workflows for Query Execution and Schema Retrieval

- **Overview**: Two separate workflows are referenced as tools by the AI agent to modularize query execution and schema retrieval.
- **Nodes Involved**:  
  - execute_query_tool (Tool Workflow node)  
  - get_postgres_schema (Tool Workflow node)  
  - Sticky Notes indicating these workflows:  
    - "Place this in a separate workflow named: query_executer"  
    - "place this in a separate workflow named: get database schema"  

- **Details**:

  - **query_executer**  
    - Expected to receive a JSON input with a `sql` string.  
    - Executes the SQL query against PostgreSQL and returns results.  
    - Used by the AI agent to run generated queries safely.

  - **get database schema**  
    - Expected to query the PostgreSQL schema and return a formatted string describing tables and columns.  
    - Used by the AI agent to understand database structure before query generation.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                                  | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                      |
|---------------------------|----------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Google Drive Trigger       | Google Drive Trigger              | Trigger on Google Sheet file changes            |                             | change_this                 |                                                                                                |
| change_this               | Set                              | Sets sheet URL and sheet name parameters        | Google Drive Trigger         | table exists?               |                                                                                                |
| table exists?             | PostgreSQL                       | Checks if PostgreSQL table for sheet exists     | change_this                 | fetch sheet data            |                                                                                                |
| fetch sheet data          | Google Sheets                    | Fetches data from specified Google Sheet tab    | table exists?               | is not in database          |                                                                                                |
| is not in database        | If                              | Branches based on table existence                | fetch sheet data            | create table query, remove table |                                                                                                |
| remove table              | PostgreSQL                      | Drops existing PostgreSQL table if present      | is not in database          | create table query          |                                                                                                |
| create table query        | Code                            | Infers schema and generates CREATE TABLE query  | is not in database          | create table                |                                                                                                |
| create table              | PostgreSQL                      | Executes CREATE TABLE query                       | create table query          | create insertion query      |                                                                                                |
| create insertion query    | Code                            | Formats data and generates INSERT query          | create table                | perform insertion           |                                                                                                |
| perform insertion         | PostgreSQL                      | Inserts formatted data into PostgreSQL table    | create insertion query      |                             |                                                                                                |
| When chat message received| Manual Chat Trigger (LangChain) | Receives natural language queries                |                             | AI Agent With SQL Query Prompt |                                                                                                |
| AI Agent With SQL Query Prompt | LangChain Agent              | Generates SQL queries from natural language      | When chat message received, execute_query_tool, get_postgres_schema, Google Gemini Chat Model | sql query executor          | Sticky Note: "## Use a powerful LLM to correctly build the SQL queries..."                      |
| execute_query_tool        | Tool Workflow                   | Executes SQL queries (sub-workflow)              | AI Agent With SQL Query Prompt | AI Agent With SQL Query Prompt | Sticky Note: "Place this in a separate workflow named: query_executer"                         |
| get_postgres_schema       | Tool Workflow                   | Retrieves database schema (sub-workflow)         | AI Agent With SQL Query Prompt | AI Agent With SQL Query Prompt | Sticky Note: "place this in a separate workflow named: get database schema"                    |
| Google Gemini Chat Model  | LangChain LLM                  | Provides AI language model for query generation  |                             | AI Agent With SQL Query Prompt |                                                                                                |
| Execute Workflow Trigger  | Execute Workflow Trigger        | Triggers execution of SQL query and schema fetch |                             | sql query executor, schema finder |                                                                                                |
| sql query executor        | PostgreSQL                      | Executes AI-generated SQL query                   | Execute Workflow Trigger    | response output             |                                                                                                |
| response output           | Set                            | Formats and sets the final response               | sql query executor          |                             |                                                                                                |
| schema finder             | PostgreSQL                      | Retrieves PostgreSQL schema info                   | Execute Workflow Trigger    | schema to string            |                                                                                                |
| schema to string          | Code                            | Formats schema info into readable string          | schema finder               | get_postgres_schema         |                                                                                                |
| Sticky Note               | Sticky Note                    | Notes for query_executer sub-workflow             |                             |                             | "Place this in a separate workflow named:\n### query_executer"                                |
| Sticky Note1              | Sticky Note                    | Notes for get database schema sub-workflow         |                             |                             | "place this in a separate workflow named: \n### get database schema"                          |
| Sticky Note2              | Sticky Note                    | General note on using LLM for SQL query building  |                             |                             | "## Use a powerful LLM to correctly build the SQL queries, which will be identified from the get schema tool and then executed by the execute query tool." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure to watch a specific Google Sheet file by its ID.  
   - Set credentials with Google Drive OAuth2.  
   - No input connections.

2. **Create Set Node ("change_this")**  
   - Type: Set  
   - Add two string fields:  
     - `table_url`: Set to your Google Sheet URL.  
     - `sheet_name`: Set to the specific sheet tab name (e.g., "product_list").  
   - Connect output of Google Drive Trigger to this node.

3. **Create PostgreSQL Node ("table exists?")**  
   - Type: PostgreSQL  
   - Operation: Execute Query  
   - Query:  
     ```sql
     SELECT EXISTS (
       SELECT 1 
       FROM information_schema.tables 
       WHERE table_name = 'ai_table_{{ $json.sheet_name }}'
     );
     ```  
   - Set PostgreSQL credentials.  
   - Connect output of `change_this` to this node.

4. **Create Google Sheets Node ("fetch sheet data")**  
   - Type: Google Sheets  
   - Configure to read data from the sheet tab named `={{ $json.sheet_name }}` and document ID from `={{ $json.table_url }}`.  
   - Set Google Sheets OAuth2 credentials.  
   - Connect output of `table exists?` to this node.

5. **Create If Node ("is not in database")**  
   - Type: If  
   - Condition: Check if `$('table exists?').item.json.exists` is false.  
   - Connect output of `fetch sheet data` to this node.

6. **Create PostgreSQL Node ("remove table")**  
   - Type: PostgreSQL  
   - Operation: Execute Query  
   - Query:  
     ```sql
     DROP TABLE IF EXISTS ai_table_{{ $json.sheet_name }} CASCADE;
     ```  
   - Set PostgreSQL credentials.  
   - Connect the false branch of `is not in database` to this node.

7. **Create Code Node ("create table query")**  
   - Type: Code  
   - Paste the provided JavaScript code that:  
     - Infers column types (currency, date, text).  
     - Generates a CREATE TABLE SQL statement with UUID primary key.  
     - Outputs `query` and `columnMapping`.  
   - Connect both true branch of `is not in database` and output of `remove table` to this node.

8. **Create PostgreSQL Node ("create table")**  
   - Type: PostgreSQL  
   - Operation: Execute Query  
   - Query: `{{ $json.query }}` (from previous node)  
   - Set PostgreSQL credentials.  
   - Connect output of `create table query` to this node.

9. **Create Code Node ("create insertion query")**  
   - Type: Code  
   - Paste the provided JavaScript code that:  
     - Formats data according to inferred types.  
     - Constructs parameterized INSERT INTO query and parameters array.  
   - Connect output of `create table` to this node.

10. **Create PostgreSQL Node ("perform insertion")**  
    - Type: PostgreSQL  
    - Operation: Execute Query  
    - Query: `{{$json.query}}`  
    - Query Replacement Parameters: `={{$json.parameters}}`  
    - Set PostgreSQL credentials.  
    - Connect output of `create insertion query` to this node.

11. **Create Manual Chat Trigger Node ("When chat message received")**  
    - Type: Manual Chat Trigger (LangChain)  
    - No special configuration.  
    - No input connections.

12. **Create LangChain Agent Node ("AI Agent With SQL Query Prompt")**  
    - Type: LangChain Agent  
    - Configure system message with detailed prompt defining role, tools, process, best practices, and response structure as provided.  
    - Add two tools:  
      - `get_postgres_schema` (Tool Workflow node)  
      - `execute_query_tool` (Tool Workflow node)  
    - Set Google Gemini Chat Model as AI language model.  
    - Connect output of `When chat message received` to this node.

13. **Create Tool Workflow Nodes**  
    - `execute_query_tool`: Configure to call a separate workflow named `query_executer` that executes SQL queries.  
    - `get_postgres_schema`: Configure to call a separate workflow named `get database schema` that returns formatted database schema.  
    - Connect these tool nodes as tools inside the agent node.

14. **Create Google Gemini Chat Model Node**  
    - Type: LangChain LLM  
    - Model Name: `models/gemini-2.0-flash`  
    - Set Google Palm API credentials.  
    - Connect as AI language model input to the agent node.

15. **Create Execute Workflow Trigger Node ("Execute Workflow Trigger")**  
    - Type: Execute Workflow Trigger  
    - No parameters.  
    - Connect output to two PostgreSQL nodes: `sql query executor` and `schema finder`.

16. **Create PostgreSQL Node ("sql query executor")**  
    - Type: PostgreSQL  
    - Operation: Execute Query  
    - Query: `{{ $json.query.sql }}`  
    - Set PostgreSQL credentials.  
    - Connect output of `Execute Workflow Trigger` to this node.

17. **Create Set Node ("response output")**  
    - Type: Set  
    - Assign field `response` with value `={{ $json }}` (the query result).  
    - Connect output of `sql query executor` to this node.

18. **Create PostgreSQL Node ("schema finder")**  
    - Type: PostgreSQL  
    - Operation: Execute Query  
    - Query:  
      ```sql
      SELECT 
          t.schemaname,
          t.tablename,
          c.column_name,
          c.data_type
      FROM 
          pg_catalog.pg_tables t
      JOIN 
          information_schema.columns c
          ON t.schemaname = c.table_schema
          AND t.tablename = c.table_name
      WHERE 
          t.schemaname = 'public'
      ORDER BY 
          t.tablename, c.ordinal_position;
      ```  
    - Set PostgreSQL credentials.  
    - Connect output of `Execute Workflow Trigger` to this node.

19. **Create Code Node ("schema to string")**  
    - Type: Code  
    - Paste provided JavaScript code that formats schema rows into a readable string.  
    - Connect output of `schema finder` to this node.

20. **Deploy and Configure Sub-Workflows**  
    - Create a workflow named `query_executer` that accepts a JSON input with a `sql` string, executes it on PostgreSQL, and returns results.  
    - Create a workflow named `get database schema` that runs the schema finder query and returns the formatted schema string.  
    - Ensure these workflows are accessible and linked correctly in the tool workflow nodes.

21. **Test the Workflow**  
    - Run the workflow manually to initialize the database table and insert data.  
    - Use the chat trigger to send natural language queries and verify AI-generated SQL queries execute correctly and return expected results.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Want to see it in action? Watch the full breakdown here: [📺 Video Link](https://www.youtube.com/watch?v=uj_XpLSMRmk) | Video demonstration of the workflow in operation.                                              |
| This workflow bypasses vector database inefficiencies by using PostgreSQL for numerical data queries.          | Conceptual advantage over vector DB approaches for financial/numerical data.                   |
| Ensure your Google Sheet or CSV has consistent column headers for smooth schema detection.                      | Best practice for data preparation.                                                           |
| Test with simple questions first to verify AI agent query generation.                                          | Recommended troubleshooting step.                                                             |
| Check out the [n8n Template Submission Guidelines](insert-link-here) for more best practices.                   | Additional resource for workflow development standards.                                       |
| Sticky Note: "## Use a powerful LLM to correctly build the SQL queries, which will be identified from the get schema tool and then executed by the execute query tool." | Emphasizes the AI agent’s role and tool integration.                                          |
| Sticky Note: "Place this in a separate workflow named:\n### query_executer"                                    | Instruction for modular sub-workflow handling SQL execution.                                  |
| Sticky Note: "place this in a separate workflow named: \n### get database schema"                              | Instruction for modular sub-workflow handling schema retrieval.                               |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Query Google Sheets/CSV data through an AI Agent using PostgreSQL" workflow in n8n. It covers all nodes, logic blocks, and integration points, enabling advanced users and AI agents to work effectively with the workflow.