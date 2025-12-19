AI-Powered Cover Letter Generator with Resume Matching & Google Docs

https://n8nworkflows.xyz/workflows/ai-powered-cover-letter-generator-with-resume-matching---google-docs-11840


# AI-Powered Cover Letter Generator with Resume Matching & Google Docs

### 1. Workflow Overview

This workflow automates the creation of a personalized cover letter by combining a user-provided job description with a predefined resume. It leverages a large language model (LLM) agent to generate a tailored cover letter formatted as structured JSON, then creates and populates a Google Document with the generated text for easy download and further editing.

The workflow supports two primary modes of input: manual form submission (for end-users to provide a job description) and execution as a sub-workflow where both job description and resume can be provided programmatically.

**Logical Blocks:**

- **1.1 Input Reception:** Accepts the job description via a form trigger or sub-workflow input and sets a static resume.
- **1.2 AI Processing:** Uses an LLM chat model agent to generate a personalized cover letter JSON output based on the resume and job description.
- **1.3 Output Validation:** Parses and validates the LLM output to enforce a strict JSON schema.
- **1.4 Document Creation:** Creates a new Google Document with a standardized title reflecting the company, job title, and current date.
- **1.5 Document Population:** Inserts the generated cover letter text into the created Google Doc.
- **1.6 User Delivery:** Redirects the user to the Google Doc for download or further editing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives the job description either via a manual form submission (interactive) or as inputs when triggered by another workflow. It also sets a static, hardcoded resume that the AI agent will use during text generation.

- **Nodes Involved:**  
  - On form submission  
  - When Executed by Another Workflow  
  - Configuration

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger (Webhook-based)  
    - Role: Captures user input of the job description text through a web form titled "Job Description".  
    - Configuration: Single textarea field labeled "jobDescription".  
    - Inputs: HTTP request from user form submission.  
    - Outputs: Passes `jobDescription` field downstream.  
    - Failure modes: Network/webhook errors, missing form input.  
    - Notes: Only required user input when running manually.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Enables this workflow to be called as a sub-workflow with parameters passed programmatically.  
    - Configuration: Accepts workflow inputs `jobDescription` and `resume`.  
    - Inputs: External workflow execution trigger with inputs.  
    - Outputs: Passes inputs downstream exactly.  
    - Failure modes: Invalid or missing inputs, execution permission errors.

  - **Configuration**  
    - Type: Set Node  
    - Role: Injects a static resume text into the workflow for use by the AI agent.  
    - Configuration: Assigns a multiline string containing a detailed resume for "Joel Gamble".  
    - Inputs: Receives `jobDescription` from form or sub-workflow trigger.  
    - Outputs: Passes both `jobDescription` and static `resume` downstream.  
    - Failure modes: None typical; manual update required to change resume content.

---

#### 2.2 AI Processing

- **Overview:**  
  This block runs a language model agent that takes the job description and resume, analyzes them, and generates a personalized cover letter in a strict JSON format.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - Write Cover Letter  
  - Structured Output Parser

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: Language Model Chat Node (OpenRouter API)  
    - Role: Provides the underlying LLM engine (GPT-5-mini) to the agent node.  
    - Configuration: Uses the "openai/gpt-5-mini" model via OpenRouter API credentials.  
    - Inputs: Receives prompt and context from the agent node.  
    - Outputs: Returns raw LLM completions.  
    - Failure modes: API key or quota issues, timeout, network errors.

  - **Write Cover Letter**  
    - Type: LangChain Agent Node  
    - Role: Builds an LLM prompt incorporating the static resume and job description; instructs the model to output a JSON object with fields: `jobTitle`, `company`, and `coverLetter`.  
    - Configuration:  
      - System prompt specifies tone ("laid-back conversational manner"), output format (JSON), and content requirements (relevancy, length, no fluff).  
      - User prompt includes the full resume and job description dynamically.  
    - Inputs: Receives `resume` from Configuration node and `jobDescription` from form or sub-workflow.  
    - Outputs: Generates structured JSON object with cover letter content.  
    - Failure modes: Model hallucination, malformed output (mitigated by parser), API failures.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Validates and enforces the JSON schema expected from the LLM output to prevent malformed downstream data.  
    - Configuration: Uses a JSON schema example defining required fields (`jobTitle`, `company`, `coverLetter`).  
    - Inputs: Receives raw LLM output from Write Cover Letter.  
    - Outputs: Passes parsed and validated JSON object downstream.  
    - Failure modes: Parsing errors if output deviates from schema; workflow should handle or log such exceptions.

---

#### 2.3 Document Creation & Population

- **Overview:**  
  This block creates a new Google Document titled with the company, job title, and current date, then populates it with the generated cover letter text.

- **Nodes Involved:**  
  - Create Cover Letter  
  - Populate Cover Letter

- **Node Details:**

  - **Create Cover Letter**  
    - Type: Google Docs Create Document Node  
    - Role: Creates a new Google Doc with a dynamic title to store the cover letter.  
    - Configuration:  
      - Title format: `{company} - {jobTitle} - {YYYY-MM-DD}`  
      - Uses Google Docs OAuth2 credentials for access.  
      - Folder ID specified to save document in a particular Google Drive folder.  
    - Inputs: Receives parsed JSON output with `company` and `jobTitle`.  
    - Outputs: Returns document metadata including document ID and URL.  
    - Failure modes: Credential expiration, permission errors, API rate limits.

  - **Populate Cover Letter**  
    - Type: Google Docs Update Document Node  
    - Role: Inserts the generated cover letter text into the newly created Google Doc.  
    - Configuration:  
      - Operation: Update document  
      - Action: Insert text (the `coverLetter` field) at the start or default location.  
      - Document URL dynamically set from previous node output.  
      - Uses same Google Docs OAuth2 credentials.  
    - Inputs: Receives document ID from Create Cover Letter, and cover letter text from parsed JSON.  
    - Outputs: Confirms document update success.  
    - Failure modes: Document access revoked, network issues, invalid document ID.

---

#### 2.4 User Delivery

- **Overview:**  
  This final block redirects the user to the completed Google Doc to download or edit the cover letter.

- **Nodes Involved:**  
  - Download Cover Letter

- **Node Details:**

  - **Download Cover Letter**  
    - Type: Form Node (Completion Operation)  
    - Role: Completes the workflow by redirecting the user to the Google Doc URL for direct access.  
    - Configuration:  
      - Operation: Completion  
      - Redirect URL constructed dynamically: `https://docs.google.com/document/d/{documentId}`  
      - Responds with HTTP redirect to the user.  
    - Inputs: Receives document ID from Populate Cover Letter node.  
    - Outputs: HTTP redirect response to user.  
    - Failure modes: Invalid document ID, redirect errors, user browser issues.

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                     | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                        |
|--------------------------|----------------------------------|-----------------------------------|-----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger                     | Accepts job description input     | (Webhook trigger)            | Configuration                 | This workflow generates a personalized cover letter using a provided **job description** and a **preconfigured resume**, then creates and populates a Google Doc for immediate download. It supports both manual form submission and execution as a sub-workflow. |
| Configuration            | Set                              | Injects static resume text        | On form submission           | Write Cover Letter             | See above                                                                                                         |
| When Executed by Another Workflow | Execute Workflow Trigger      | Accepts inputs for sub-workflow   | External trigger             | Write Cover Letter             | See above                                                                                                         |
| OpenRouter Chat Model    | Language Model Chat (OpenRouter) | Provides LLM engine               | Write Cover Letter (agent)   | Write Cover Letter (agent)     |                                                                                                                  |
| Write Cover Letter       | LangChain Agent                  | Generates cover letter JSON       | Configuration, When Executed by Another Workflow | Create Cover Letter           |                                                                                                                  |
| Structured Output Parser | LangChain Output Parser          | Validates LLM JSON output         | Write Cover Letter           | Create Cover Letter            |                                                                                                                  |
| Create Cover Letter      | Google Docs Create Document Node | Creates new Google Doc            | Structured Output Parser     | Populate Cover Letter          |                                                                                                                  |
| Populate Cover Letter    | Google Docs Update Document Node | Inserts cover letter into document | Create Cover Letter          | Download Cover Letter          |                                                                                                                  |
| Download Cover Letter    | Form Completion Node             | Redirects user to Google Doc      | Populate Cover Letter        | (End)                        |                                                                                                                  |
| Sticky Note             | Sticky Note                     | Workflow description and notes    | N/A                         | N/A                           | # Cover Letter Generator<br>This workflow generates a personalized cover letter using a provided **job description** and a **preconfigured resume**, then creates and populates a Google Doc for immediate download. It supports both manual form submission and execution as a sub-workflow.<br>... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Node Name: `On form submission`  
   - Type: Form Trigger  
   - Configuration:  
     - Form Title: "Job Description"  
     - Add one textarea field labeled `jobDescription`  
   - Position: Start of workflow  
   - Purpose: To capture user input of the job description.

2. **Create an Execute Workflow Trigger Node**  
   - Node Name: `When Executed by Another Workflow`  
   - Type: Execute Workflow Trigger  
   - Configuration: Accept input parameters `jobDescription` and `resume`  
   - Purpose: To allow sub-workflow execution with inputs.

3. **Create a Set Node**  
   - Node Name: `Configuration`  
   - Type: Set  
   - Configuration:  
     - Assign a string variable named `resume` containing the full static resume text (as provided).  
     - Pass through `jobDescription` unchanged.  
   - Input connections: Connect output of `On form submission` and `When Executed by Another Workflow` to this node (merge or parallel as needed).  
   - Purpose: To inject the static resume for use by the LLM agent.

4. **Create an OpenRouter Chat Model Node**  
   - Node Name: `OpenRouter Chat Model`  
   - Type: LLM Chat Node  
   - Configuration:  
     - Model: `openai/gpt-5-mini`  
     - Credentials: Add and configure OpenRouter API credentials with valid API key.  
   - Purpose: Provide the LLM engine for the subsequent agent node.

5. **Create a LangChain Agent Node**  
   - Node Name: `Write Cover Letter`  
   - Type: LangChain Agent  
   - Configuration:  
     - Prompt Type: Define (custom)  
     - System Message: Use the provided detailed system prompt explaining role, task, format, and requirements for the cover letter generation.  
     - Text Input: Pass in the `resume` and `jobDescription` variables from `Configuration` and form inputs.  
     - Output Parser: Enable and link to the next node (`Structured Output Parser`).  
   - Connect the `OpenRouter Chat Model` as the AI language model input.  
   - Purpose: Generate a structured JSON cover letter.

6. **Create a LangChain Structured Output Parser Node**  
   - Node Name: `Structured Output Parser`  
   - Type: Output Parser  
   - Configuration:  
     - Define JSON schema with required fields: `jobTitle`, `company`, `coverLetter`.  
     - Use the example JSON provided to ensure correct parsing.  
   - Connect input from `Write Cover Letter` node’s output parser.  
   - Purpose: Validate and parse the LLM output before downstream use.

7. **Create a Google Docs Create Document Node**  
   - Node Name: `Create Cover Letter`  
   - Type: Google Docs Create Document  
   - Configuration:  
     - Title: Use an expression combining `{{ $json.output.company }} - {{ $json.output.jobTitle }} - {{ $now.format('yyyy-MM-dd') }}`  
     - Folder ID: Set to target Google Drive folder ID for document storage.  
     - Credentials: Configure Google Docs OAuth2 with appropriate scopes and account.  
   - Connect input from `Structured Output Parser` node.  
   - Purpose: Create a new Google Document with a standardized title.

8. **Create a Google Docs Update Document Node**  
   - Node Name: `Populate Cover Letter`  
   - Type: Google Docs Update Document  
   - Configuration:  
     - Operation: Update document  
     - Action: Insert text  
     - Text: Use expression `={{ $('Write Cover Letter').item.json.output.coverLetter }}` to insert the generated cover letter.  
     - Document URL: Use the document ID from the `Create Cover Letter` node output.  
     - Credentials: Use the same Google Docs OAuth2 credentials.  
   - Connect input from `Create Cover Letter`.  
   - Purpose: Insert the generated cover letter text into the document.

9. **Create a Form Completion Node**  
   - Node Name: `Download Cover Letter`  
   - Type: Form Node (Completion Operation)  
   - Configuration:  
     - Operation: Completion  
     - Redirect URL: Construct redirect link `=https://docs.google.com/document/d/{{ $json.documentId }}` to automatically open the created Google Doc.  
     - Respond With: Redirect  
   - Connect input from `Populate Cover Letter`.  
   - Purpose: Redirect the user to the Google Doc for review and download.

10. **Connect Nodes According to Logic:**  
    - `On form submission` → `Configuration`  
    - `When Executed by Another Workflow` → `Write Cover Letter` (alternative path for sub-workflow)  
    - `Configuration` → `Write Cover Letter`  
    - `Write Cover Letter` → `Structured Output Parser`  
    - `Structured Output Parser` → `Create Cover Letter`  
    - `Create Cover Letter` → `Populate Cover Letter`  
    - `Populate Cover Letter` → `Download Cover Letter`

11. **Set Credentials:**  
    - Setup `OpenRouter API` credentials with a valid key for the OpenRouter Chat Model node.  
    - Setup `Google Docs OAuth2` credentials with permissions to create and edit Google Docs in the specified folder.

12. **Test Workflow:**  
    - Submit a job description via the form or trigger via sub-workflow with inputs.  
    - Verify that a Google Doc is created with the expected title and populated cover letter content.  
    - Confirm redirection to the document URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow generates a personalized cover letter using a provided **job description** and a **preconfigured resume**, then creates and populates a Google Doc for immediate download. It supports both manual form submission and execution as a sub-workflow. The cover letter generation leverages an LLM agent with an expert system prompt designed to produce concise, casual yet professional letters strictly formatted as JSON.                                                                                                                                                | Workflow purpose and design notes                                                                |
| The resume text is hardcoded in the `Configuration` node. Update this node’s content to change the applicant’s resume for future cover letters.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Static resume injection explanation                                                             |
| The Google Docs nodes require OAuth2 credentials with scopes for document creation and editing. Ensure the configured Google account has access to the target folder specified by the Folder ID.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Google Docs OAuth2 credential requirements                                                      |
| The OpenRouter Chat Model node uses the "openai/gpt-5-mini" model via OpenRouter API. Valid API access and sufficient quota are required to avoid runtime errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | OpenRouter API integration details                                                              |
| The structured output parser enforces a strict JSON schema to ensure downstream nodes receive valid data, preventing workflow failure due to malformed LLM responses.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Output validation importance                                                                     |
| Redirecting the user to the Google Doc URL allows immediate review, editing, or downloading of the generated cover letter through a familiar Google Docs interface.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Final user delivery method                                                                       |
| For advanced customization, consider parameterizing the resume input to accept dynamic resumes per user or integrating additional resume sources.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Suggestions for workflow enhancement                                                           |
| Official n8n documentation for Google Docs nodes and LangChain integrations can assist with troubleshooting and extending functionality.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | https://docs.n8n.io/nodes/n8n-nodes-base.googledocs/ and https://docs.n8n.io/nodes/ai-nodes/       |

---

**Disclaimer:** The provided text and workflow are generated exclusively using n8n automation tools, adhering to all relevant content policies. No illegal, offensive, or protected content is included. All handled data is legal and publicly accessible.