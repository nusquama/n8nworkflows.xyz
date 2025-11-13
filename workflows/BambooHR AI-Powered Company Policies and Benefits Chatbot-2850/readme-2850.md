BambooHR AI-Powered Company Policies and Benefits Chatbot

https://n8nworkflows.xyz/workflows/bamboohr-ai-powered-company-policies-and-benefits-chatbot-2850


# BambooHR AI-Powered Company Policies and Benefits Chatbot

### 1. Workflow Overview

This workflow implements an AI-powered chatbot designed to provide instant HR support by answering employee questions about company policies, benefits, and escalation contacts using BambooHR data. It automates retrieval and indexing of company HR documents, leverages AI to interpret employee queries, and identifies appropriate contacts for escalation.

The workflow is logically divided into the following blocks:

- **1.1 Company Policies Retrieval and Indexing**  
  Retrieves company policy documents from BambooHR, filters and downloads relevant PDFs, splits and embeds their content, and stores them in a vector database for semantic search.

- **1.2 Employee Query Reception and AI Processing**  
  Listens for employee chat input, uses AI to classify query type, searches the vector store for relevant documents, and processes the query with an AI agent that has access to company files and employee lookup tools.

- **1.3 Employee Lookup and Contact Identification**  
  Provides an optional tool to look up employee or department details from BambooHR, used by the AI agent to find contact persons for escalation when needed.

- **1.4 Contact Resolution Logic**  
  Implements logic to identify the most relevant contact person based on query context, including fallback steps to find senior employees or supervisors when direct contacts are missing.

---

### 2. Block-by-Block Analysis

#### 2.1 Company Policies Retrieval and Indexing

**Overview:**  
This block fetches all company files from BambooHR, filters to only include PDFs from the "Company Files" category, downloads them, splits their text content, generates embeddings, and inserts them into a Supabase vector store for semantic retrieval.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- GET all files (BambooHR)  
- Filter out files from undesired categories (Filter)  
- Split out individual files (SplitOut)  
- Filter out non-pdf files (Filter)  
- Download file from BambooHR (BambooHR)  
- Recursive Character Text Splitter (Text Splitter)  
- Default Data Loader (Document Loader)  
- Embeddings OpenAI (Embeddings)  
- Supabase Vector Store (Vector Store Insert)  
- Sticky Notes (contextual explanations)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the retrieval and indexing process.  
  - No parameters.  
  - Outputs to "GET all files".  
  - Edge cases: Manual trigger only; no automatic scheduling.

- **GET all files**  
  - Type: BambooHR API node  
  - Role: Retrieves all files from BambooHR with full metadata (simplify option off to retain categories).  
  - Parameters: `resource=file`, `operation=getAll`, `returnAll=true`, `simplifyOutput=false`.  
  - Outputs to "Filter out files from undesired categories".  
  - Edge cases: API rate limits, BambooHR auth errors.

- **Filter out files from undesired categories**  
  - Type: Filter node  
  - Role: Filters files to only those in the "Company Files" category.  
  - Condition: `$json.name == "Company Files"`.  
  - Outputs to "Split out individual files".  
  - Edge cases: No files matching category results in empty downstream data.

- **Split out individual files**  
  - Type: SplitOut node  
  - Role: Splits the array of files into individual items for processing.  
  - Field to split: `files`.  
  - Outputs to "Filter out non-pdf files".  
  - Edge cases: Empty input array.

- **Filter out non-pdf files**  
  - Type: Filter node  
  - Role: Keeps only files with `.pdf` extension in `originalFileName`.  
  - Condition: `$json.originalFileName.endsWith(".pdf") == true`.  
  - Outputs to "Download file from BambooHR".  
  - Edge cases: Files without extension or non-PDF files filtered out.

- **Download file from BambooHR**  
  - Type: BambooHR API node  
  - Role: Downloads the binary content of each PDF file by `fileId`.  
  - Parameters: `fileId` from input JSON `id`.  
  - Outputs to "Supabase Vector Store".  
  - Edge cases: Download failures, network issues, auth errors.

- **Recursive Character Text Splitter**  
  - Type: Text Splitter (LangChain)  
  - Role: Splits large text content into overlapping chunks (100 characters overlap) for embedding.  
  - Outputs to "Default Data Loader".  
  - Edge cases: Very large files may cause performance issues.

- **Default Data Loader**  
  - Type: Document Loader (LangChain)  
  - Role: Loads split text chunks as documents for embedding generation.  
  - Data type: binary (PDF content).  
  - Outputs to "Embeddings OpenAI".  
  - Edge cases: Parsing errors on malformed PDFs.

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings (LangChain)  
  - Role: Generates vector embeddings for document chunks.  
  - Credentials: OpenAI API key required.  
  - Outputs to "Supabase Vector Store".  
  - Edge cases: API rate limits, invalid credentials.

- **Supabase Vector Store**  
  - Type: Vector Store Insert (LangChain)  
  - Role: Inserts embeddings into Supabase vector database table `company_files`.  
  - Credentials: Supabase API key required.  
  - Mode: insert.  
  - Outputs: None (end of indexing chain).  
  - Edge cases: DB connection issues, insertion errors.

---

#### 2.2 Employee Query Reception and AI Processing

**Overview:**  
This block listens for employee chat input, classifies the query type, retrieves relevant documents from the vector store, and processes the query with an AI agent that can access company files and employee lookup tools.

**Nodes Involved:**  
- Employee initiates a conversation (Chat Trigger)  
- AI-Powered HR Benefits and Company Policies Chatbot (Execute Workflow Trigger)  
- Text Classifier (LangChain)  
- Supabase Vector Store Retrieval (Vector Store Query)  
- Vector Store Tool (LangChain Tool)  
- Window Buffer Memory (LangChain Memory)  
- HR AI Agent (LangChain Agent)  
- OpenAI Chat Model (multiple instances)  
- Embeddings OpenAI1 (Embeddings for retrieval)  
- Sticky Notes (guidelines and explanations)

**Node Details:**

- **Employee initiates a conversation**  
  - Type: Chat Trigger (LangChain)  
  - Role: Webhook entry point for employee chat queries.  
  - Webhook ID configured for external access.  
  - Outputs to "HR AI Agent".  
  - Edge cases: Webhook downtime, malformed input.

- **AI-Powered HR Benefits and Company Policies Chatbot**  
  - Type: Execute Workflow Trigger  
  - Role: Invokes the main AI processing workflow on incoming queries.  
  - Outputs to "Text Classifier".  
  - Edge cases: Workflow invocation failures.

- **Text Classifier**  
  - Type: Text Classifier (LangChain)  
  - Role: Classifies query input as either a "person" or "department" name.  
  - Input text: `{{$json.query.name}}` from the chat input.  
  - Categories: "person" and "department" with descriptions.  
  - Outputs to two parallel paths for employee or department lookup.  
  - Edge cases: Misclassification, ambiguous inputs.

- **Supabase Vector Store Retrieval**  
  - Type: Vector Store Query (LangChain)  
  - Role: Retrieves top 5 relevant documents from `company_files` vector store based on query embedding.  
  - Credentials: Supabase API key.  
  - Outputs to "Vector Store Tool".  
  - Edge cases: No relevant documents found.

- **Vector Store Tool**  
  - Type: LangChain Tool  
  - Role: Provides the AI agent access to the vector store for document retrieval.  
  - Description: Contains company handbook, 401k policies, benefits overview, expense policies.  
  - Outputs to "HR AI Agent".  
  - Edge cases: Tool invocation errors.

- **Window Buffer Memory**  
  - Type: LangChain Memory  
  - Role: Maintains conversational context window for the AI agent.  
  - Outputs to "HR AI Agent".  
  - Edge cases: Memory overflow or context loss.

- **HR AI Agent**  
  - Type: LangChain Agent  
  - Role: Central AI processing node that answers employee queries using tools and memory.  
  - System message defines role as helpful HR assistant with access to company files and employee lookup tool.  
  - Implements escalation guidelines for contact person retrieval.  
  - Credentials: OpenAI API key.  
  - Inputs: Chat trigger, vector store tool, employee lookup tool, memory.  
  - Outputs: AI-generated responses.  
  - Edge cases: AI model errors, tool invocation failures, incomplete data.

- **OpenAI Chat Model (multiple instances)**  
  - Type: Language Model (LangChain)  
  - Role: Used in various steps for classification, extraction, and response generation.  
  - Credentials: OpenAI API key.  
  - Edge cases: API limits, latency.

- **Embeddings OpenAI1**  
  - Type: Embeddings generation for retrieval queries.  
  - Credentials: OpenAI API key.

---

#### 2.3 Employee Lookup and Contact Identification

**Overview:**  
This block provides a tool to look up employee or department details from BambooHR, used by the AI agent to find contact persons or senior leaders for escalation.

**Nodes Involved:**  
- Employee Lookup Tool (LangChain Tool Workflow)  
- GET all employees (BambooHR)  
- Filter out other employees (Filter)  
- Stringify employee record for response (Set)  
- GET all employees (second path) (BambooHR)  
- Extract departments (Aggregate)  
- Ensure uniqueness in department list (Set)  
- Extract department (Information Extractor)  
- Retrieve all employees (BambooHR)  
- Filter out other departments (Filter)  
- Extract relevant employee fields (Aggregate)  
- Identify most senior employee (Chain LLM)  
- Format name for response (Set)  
- Auto-fixing Output Parser (Output Parser)  
- Structured Output Parser (Output Parser)  
- Sticky Notes (explanations and guidelines)

**Node Details:**

- **Employee Lookup Tool**  
  - Type: LangChain Tool Workflow  
  - Role: Allows AI agent to query employee or department details by name.  
  - Input schema: JSON with `name` field (employee or department).  
  - Outputs employee details or senior-most person in department.  
  - Edge cases: Requires exact name match; fallback logic if no match.

- **GET all employees**  
  - Type: BambooHR API node  
  - Role: Retrieves all employees for filtering by name.  
  - Parameters: `operation=getAll`, `returnAll=true`.  
  - Outputs to "Filter out other employees".  
  - Edge cases: Large data sets may cause inefficiency.

- **Filter out other employees**  
  - Type: Filter node  
  - Role: Filters employees to match the requested name from AI query.  
  - Condition: `$json.displayName == query.name`.  
  - Outputs to "Stringify employee record for response".  
  - Edge cases: Multiple employees with same name.

- **Stringify employee record for response**  
  - Type: Set node  
  - Role: Converts employee JSON record to string for AI response.  
  - Outputs: Final employee info.  
  - Edge cases: Empty input.

- **GET all employees (second path)**  
  - Type: BambooHR API node  
  - Role: Retrieves all employees for department extraction path.  
  - Outputs to "Extract departments".  
  - Edge cases: Same as first GET all employees.

- **Extract departments**  
  - Type: Aggregate node  
  - Role: Aggregates all department names from employee records.  
  - Outputs to "Ensure uniqueness in department list".  
  - Edge cases: Departments with inconsistent naming.

- **Ensure uniqueness in department list**  
  - Type: Set node  
  - Role: Removes duplicate departments and sets query string.  
  - Outputs to "Extract department".  
  - Edge cases: Empty department list.

- **Extract department**  
  - Type: Information Extractor (LangChain)  
  - Role: Extracts the most applicable department from the list based on query.  
  - Inputs: Query string and department list.  
  - Outputs to "Retrieve all employees".  
  - Edge cases: Ambiguous department names.

- **Retrieve all employees**  
  - Type: BambooHR API node  
  - Role: Retrieves all employees for filtering by department.  
  - Outputs to "Filter out other departments".  
  - Edge cases: Large data sets.

- **Filter out other departments**  
  - Type: Filter node  
  - Role: Filters employees to those in the extracted department.  
  - Outputs to "Extract relevant employee fields".  
  - Edge cases: Employees with missing department data.

- **Extract relevant employee fields**  
  - Type: Aggregate node  
  - Role: Aggregates specified fields (`id, displayName, jobTitle, workEmail`) for filtered employees.  
  - Outputs to "Identify most senior employee".  
  - Edge cases: Missing fields.

- **Identify most senior employee**  
  - Type: Chain LLM (LangChain)  
  - Role: Uses AI to determine the most senior employee from the list.  
  - Prompt includes employee list JSON.  
  - Outputs to "Format name for response".  
  - Edge cases: Ambiguous seniority.

- **Format name for response**  
  - Type: Set node  
  - Role: Formats the identified senior employee's name as a string response.  
  - Outputs: Final contact name.  
  - Edge cases: Empty output.

- **Auto-fixing Output Parser**  
  - Type: Output Parser (LangChain)  
  - Role: Parses and corrects AI output for senior employee identification.  
  - Outputs to "Identify most senior employee".  
  - Edge cases: Parsing errors.

- **Structured Output Parser**  
  - Type: Output Parser (LangChain)  
  - Role: Parses structured JSON output for employee lookup tool.  
  - Outputs to "Auto-fixing Output Parser".  
  - Edge cases: Invalid JSON.

---

#### 2.4 Contact Resolution Logic

**Overview:**  
This logic is embedded in the AI agent's system message and workflow design, guiding escalation contact retrieval: first searching company files, then using employee lookup tool for missing details or fallback to senior department contacts or supervisors.

**Nodes Involved:**  
- HR AI Agent (LangChain Agent)  
- Employee Lookup Tool (LangChain Tool Workflow)  
- Text Classifier (LangChain)  
- GET all employees and related filtering nodes  
- Sticky Notes with detailed operating guidelines

**Node Details:**

- The AI agent follows these steps when asked for a contact person:  
  1. Search company_files vector store for contact info.  
  2. If contact found but incomplete, use employee lookup tool to fill details.  
  3. If no contact found, use employee lookup tool with "HR" or relevant department to find senior person.  
  4. If still no contact, ask employee for their name, find their supervisor, and provide supervisor contact.  
- This logic is implemented through the combination of AI prompts, vector store queries, and employee lookup tool calls.  
- Edge cases include missing or outdated employee data, ambiguous queries, or API failures.

---

### 3. Summary Table

| Node Name                            | Node Type                                | Functional Role                                         | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                      |
|------------------------------------|----------------------------------------|---------------------------------------------------------|--------------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’       | Manual Trigger                         | Manual start of company files retrieval                  |                                      | GET all files                        |                                                                                                |
| GET all files                      | BambooHR API                          | Retrieve all company files                               | When clicking ‘Test workflow’         | Filter out files from undesired categories | Toggle **off** the _simplify_ option to ensure categories are retrieved as well                 |
| Filter out files from undesired categories | Filter                              | Filter files to "Company Files" category                 | GET all files                        | Split out individual files           |                                                                                                |
| Split out individual files          | SplitOut                              | Split files array into individual file items             | Filter out files from undesired categories | Filter out non-pdf files             |                                                                                                |
| Filter out non-pdf files            | Filter                                | Keep only PDF files                                      | Split out individual files           | Download file from BambooHR           |                                                                                                |
| Download file from BambooHR         | BambooHR API                          | Download PDF file content                                | Filter out non-pdf files             | Supabase Vector Store                |                                                                                                |
| Recursive Character Text Splitter   | Text Splitter (LangChain)             | Split text into overlapping chunks                       | Default Data Loader                  | Default Data Loader                  | ## STEP #1: Retrieve company policies and load them into a vector store                         |
| Default Data Loader                 | Document Loader (LangChain)            | Load document chunks for embedding                       | Recursive Character Text Splitter    | Embeddings OpenAI                   |                                                                                                |
| Embeddings OpenAI                  | OpenAI Embeddings (LangChain)          | Generate embeddings for document chunks                  | Default Data Loader                  | Supabase Vector Store                |                                                                                                |
| Supabase Vector Store              | Vector Store Insert (LangChain)         | Insert embeddings into vector database                   | Embeddings OpenAI, Download file from BambooHR |                                  |                                                                                                |
| Employee initiates a conversation   | Chat Trigger (LangChain)                | Receive employee chat queries                            |                                      | HR AI Agent                        |                                                                                                |
| AI-Powered HR Benefits and Company Policies Chatbot | Execute Workflow Trigger            | Trigger AI processing workflow                           | Employee initiates a conversation    | Text Classifier                    |                                                                                                |
| Text Classifier                   | Text Classifier (LangChain)              | Classify query as person or department                   | AI-Powered HR Benefits and Company Policies Chatbot | GET all employees, GET all employees (second path) |                                                                                                |
| GET all employees                 | BambooHR API                          | Retrieve all employees for person lookup                | Text Classifier                     | Filter out other employees          | ### GET singular employee by name path This path may be used multiple times by the HR AI Agent to look up the employee's details, and then to look up their supervisor's details. |
| Filter out other employees         | Filter                                | Filter employees matching requested name                | GET all employees                   | Stringify employee record for response |                                                                                                |
| Stringify employee record for response | Set                                  | Convert employee record to string for response          | Filter out other employees          |                                    |                                                                                                |
| GET all employees (second path)    | BambooHR API                          | Retrieve all employees for department lookup            | Text Classifier                     | Extract departments                |                                                                                                |
| Extract departments               | Aggregate                             | Aggregate department names from employee list           | GET all employees (second path)     | Ensure uniqueness in department list |                                                                                                |
| Ensure uniqueness in department list | Set                                  | Remove duplicate departments and set query string       | Extract departments                | Extract department                |                                                                                                |
| Extract department               | Information Extractor (LangChain)       | Extract most applicable department from list            | Ensure uniqueness in department list | Retrieve all employees             |                                                                                                |
| Retrieve all employees           | BambooHR API                          | Retrieve all employees for filtered department           | Extract department                | Filter out other departments       |                                                                                                |
| Filter out other departments       | Filter                                | Filter employees by department                            | Retrieve all employees             | Extract relevant employee fields   |                                                                                                |
| Extract relevant employee fields   | Aggregate                             | Aggregate key fields for department employees            | Filter out other departments       | Identify most senior employee      |                                                                                                |
| Identify most senior employee      | Chain LLM (LangChain)                  | Use AI to identify most senior employee                  | Extract relevant employee fields   | Format name for response           |                                                                                                |
| Format name for response           | Set                                  | Format senior employee name for response                 | Identify most senior employee      |                                    | ### Final node returns employee name The AI Agent can then call the employee lookup path to retrieve details, if requested. |
| Supabase Vector Store Retrieval    | Vector Store Query (LangChain)          | Retrieve relevant documents from vector store            | Embeddings OpenAI1                 | Vector Store Tool                 |                                                                                                |
| Vector Store Tool                | LangChain Tool                         | Provide vector store access to AI agent                  | Supabase Vector Store Retrieval    | HR AI Agent                      |                                                                                                |
| Window Buffer Memory             | LangChain Memory                      | Maintain conversation context for AI agent               |                                  | HR AI Agent                      |                                                                                                |
| HR AI Agent                     | LangChain Agent                       | Core AI agent answering queries using tools and memory  | Employee initiates a conversation, Vector Store Tool, Employee Lookup Tool, Window Buffer Memory |                                    |                                                                                                |
| Employee Lookup Tool             | LangChain Tool Workflow                | Tool to look up employee or department details           | HR AI Agent                      | HR AI Agent                      | This employee lookup tool gives the AI Benefits and Company Policies chatbot additional superpowers by allowing it to **search for an individual or a department to retrieve contact information from BambooHR**. |
| Auto-fixing Output Parser         | Output Parser (LangChain)               | Parse and correct AI output                               | Structured Output Parser           | Identify most senior employee      |                                                                                                |
| Structured Output Parser          | Output Parser (LangChain)               | Parse structured JSON output                              | OpenAI Chat Model5                | Auto-fixing Output Parser          |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - Purpose: Start company files retrieval manually.

2. **Create BambooHR Node to Get All Files**  
   - Name: `GET all files`  
   - Type: BambooHR API  
   - Resource: `file`  
   - Operation: `getAll`  
   - Return All: `true`  
   - Simplify Output: `false` (to retain categories)  
   - Credentials: BambooHR account  
   - Connect output from Manual Trigger.

3. **Create Filter Node to Keep "Company Files" Category**  
   - Name: `Filter out files from undesired categories`  
   - Condition: `$json.name == "Company Files"`  
   - Connect input from `GET all files`.

4. **Create SplitOut Node to Split Files Array**  
   - Name: `Split out individual files`  
   - Field to split out: `files`  
   - Connect input from filter node.

5. **Create Filter Node to Keep PDFs Only**  
   - Name: `Filter out non-pdf files`  
   - Condition: `$json.originalFileName.endsWith(".pdf") == true`  
   - Connect input from SplitOut node.

6. **Create BambooHR Node to Download Files**  
   - Name: `Download file from BambooHR`  
   - Resource: `file`  
   - Operation: `download`  
   - File ID: `{{$json.id}}`  
   - Credentials: BambooHR account  
   - Connect input from PDF filter node.

7. **Create Recursive Character Text Splitter Node**  
   - Name: `Recursive Character Text Splitter`  
   - Chunk Overlap: 100 characters  
   - Connect input from downloaded files.

8. **Create Default Data Loader Node**  
   - Name: `Default Data Loader`  
   - Data Type: binary  
   - Connect input from Text Splitter.

9. **Create OpenAI Embeddings Node**  
   - Name: `Embeddings OpenAI`  
   - Credentials: OpenAI API key  
   - Connect input from Data Loader.

10. **Create Supabase Vector Store Insert Node**  
    - Name: `Supabase Vector Store`  
    - Mode: insert  
    - Table Name: `company_files`  
    - Query Name: `match_files`  
    - Credentials: Supabase API key  
    - Connect input from Embeddings node and Download file node (for metadata).

11. **Create Chat Trigger Node**  
    - Name: `Employee initiates a conversation`  
    - Type: LangChain Chat Trigger  
    - Webhook ID: Generate new webhook  
    - Purpose: Receive employee queries.

12. **Create Execute Workflow Trigger Node**  
    - Name: `AI-Powered HR Benefits and Company Policies Chatbot`  
    - Connect input from Chat Trigger node.

13. **Create Text Classifier Node**  
    - Name: `Text Classifier`  
    - Input Text: `{{$json.query.name}}`  
    - Categories: "person" and "department" with descriptions  
    - Connect input from Execute Workflow Trigger.

14. **Create BambooHR Node to Get All Employees (Person Lookup Path)**  
    - Name: `GET all employees`  
    - Operation: `getAll`  
    - Return All: `true`  
    - Credentials: BambooHR account  
    - Connect input from Text Classifier (person path).

15. **Create Filter Node to Match Employee Name**  
    - Name: `Filter out other employees`  
    - Condition: `$json.displayName == {{$json.query.name}}`  
    - Connect input from GET all employees.

16. **Create Set Node to Stringify Employee Record**  
    - Name: `Stringify employee record for response`  
    - Assign `response` = `{{$json.toJsonString()}}`  
    - Connect input from Filter node.

17. **Create BambooHR Node to Get All Employees (Department Lookup Path)**  
    - Name: `GET all employees (second path)`  
    - Operation: `getAll`  
    - Return All: `true`  
    - Credentials: BambooHR account  
    - Connect input from Text Classifier (department path).

18. **Create Aggregate Node to Extract Departments**  
    - Name: `Extract departments`  
    - Field to aggregate: `department` (rename output to `departments`)  
    - Connect input from GET all employees (second path).

19. **Create Set Node to Ensure Unique Departments and Set Query**  
    - Name: `Ensure uniqueness in department list`  
    - Assign `departments` = `{{$json.departments.unique()}}`  
    - Assign `query` = `{{$('Text Classifier').item.json.query.name}}`  
    - Connect input from Aggregate node.

20. **Create Information Extractor Node to Extract Department**  
    - Name: `Extract department`  
    - Text: `{{$json.query}}`  
    - Attribute: `department` with description listing `{{$json.departments}}`  
    - Connect input from Set node.

21. **Create BambooHR Node to Retrieve All Employees**  
    - Name: `Retrieve all employees`  
    - Operation: `getAll`  
    - Return All: `true`  
    - Credentials: BambooHR account  
    - Connect input from Extract department.

22. **Create Filter Node to Filter Employees by Department**  
    - Name: `Filter out other departments`  
    - Condition: `$json.department == {{$('Extract department').item.json.output.department}}`  
    - Connect input from Retrieve all employees.

23. **Create Aggregate Node to Extract Relevant Employee Fields**  
    - Name: `Extract relevant employee fields`  
    - Fields: `id, displayName, jobTitle, workEmail`  
    - Aggregate: all item data  
    - Destination field: `department_employees`  
    - Connect input from Filter node.

24. **Create Chain LLM Node to Identify Most Senior Employee**  
    - Name: `Identify most senior employee`  
    - Prompt: "Who is the most senior employee from this list: {{ $json.department_employees.toJsonString() }}"  
    - Has output parser enabled  
    - Connect input from Aggregate node.

25. **Create Set Node to Format Name for Response**  
    - Name: `Format name for response`  
    - Assign `response` = `{{$json.output.name}}`  
    - Connect input from Chain LLM node.

26. **Create LangChain Tool Workflow Node for Employee Lookup Tool**  
    - Name: `Employee Lookup Tool`  
    - Workflow ID: This workflow's ID (self-reference)  
    - Description: Tool to retrieve employee or department details from BambooHR  
    - Input schema: JSON with `name` field  
    - Connect input from HR AI Agent node.

27. **Create LangChain Agent Node for HR AI Agent**  
    - Name: `HR AI Agent`  
    - System Message: Defines role as helpful HR assistant with access to vector store and employee lookup tool, including escalation guidelines.  
    - Credentials: OpenAI API key  
    - Connect inputs from Chat Trigger, Vector Store Tool, Employee Lookup Tool, Window Buffer Memory.

28. **Create Vector Store Tool Node**  
    - Name: `Vector Store Tool`  
    - Name: `company_files`  
    - Top K: 5  
    - Description: Retrieves info from company handbook, 401k policies, benefits overview, expense policies.  
    - Connect input from Supabase Vector Store Retrieval.

29. **Create Supabase Vector Store Retrieval Node**  
    - Name: `Supabase Vector Store Retrieval`  
    - Table Name: `company_files`  
    - Query Name: `match_files`  
    - Credentials: Supabase API key  
    - Connect input from Embeddings OpenAI1.

30. **Create Embeddings OpenAI1 Node**  
    - Name: `Embeddings OpenAI1`  
    - Credentials: OpenAI API key  
    - Connect input from HR AI Agent or query text.

31. **Create Window Buffer Memory Node**  
    - Name: `Window Buffer Memory`  
    - Connect input from HR AI Agent.

32. **Create Output Parser Nodes**  
    - Name: `Structured Output Parser` and `Auto-fixing Output Parser`  
    - Used in parsing AI outputs for employee lookup and senior employee identification.  
    - Connect accordingly to Chain LLM nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow outperforms BambooHR's native chatbot by providing superior specificity and appropriate escalations. | See images "BambooHR AI Chatbot.png", "BambooHR General benefits question.png", etc. (file IDs provided) |
| Estimated setup time is approximately 20 minutes, including BambooHR and AI credential configuration.           | Setup instructions in workflow description.                                                    |
| The employee lookup tool enhances chatbot capabilities by allowing exact or department-based contact retrieval. | Sticky Note4 content.                                                                           |
| BambooHR API does not support search by field for employees; filtering is done post-retrieval, which may be inefficient for large orgs. | Sticky Note6 content.                                                                           |
| For more information about the workflow creator Ludwig Gerdes, visit [LinkedIn](https://www.linkedin.com/in/ludwiggerdes). | Sticky Note10 content.                                                                          |

---

This document fully describes the "BambooHR AI-Powered Company Policies and Benefits Chatbot" workflow, enabling advanced users and AI agents to understand, reproduce, and modify the workflow with confidence.