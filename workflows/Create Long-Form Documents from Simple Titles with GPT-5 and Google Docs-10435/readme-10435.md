Create Long-Form Documents from Simple Titles with GPT-5 and Google Docs

https://n8nworkflows.xyz/workflows/create-long-form-documents-from-simple-titles-with-gpt-5-and-google-docs-10435


# Create Long-Form Documents from Simple Titles with GPT-5 and Google Docs

### 1. Workflow Overview

This workflow automates the creation of a long-form document on Google Docs, starting from a simple title and a desired word count. It is designed to generate structured content efficiently by leveraging advanced AI language models (GPT-5 and GPT-5-mini) and Google Docs API integration.

The workflow is logically divided into these functional blocks:

- **1.1 Input Reception:** Collects user input (title and word count) via a web form.
- **1.2 Document Initialization:** Creates a new Google Document based on the input title.
- **1.3 Content Planning:** Uses GPT-5-mini to generate a structured outline with sections and summaries.
- **1.4 Section Extraction and Looping:** Parses the outline, extracts sections, and iterates over them.
- **1.5 Section Content Generation:** For each section, GPT-5 generates detailed content constrained by the word count.
- **1.6 Document Updating:** Appends each generated section's content to the Google Document incrementally.
- **1.7 Memory Management:** Maintains a short memory buffer to provide context during section content generation.
- **1.8 User Feedback:** Displays the final Google Document link and confirms workflow completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures user input through a web form, specifically a document title and desired total word count for the generated document.

- **Nodes Involved:**  
  - `On form submission` (Form Trigger)  
  - `Form` (Form UI node)

- **Node Details:**  
  - **On form submission**  
    - Type: n8n form trigger node  
    - Role: Listens to form submissions with fields "Title" (string) and "Word Count" (number).  
    - Configuration: The form titled "Generate Long Content" requires both fields.  
    - Output: Emits JSON containing user inputs.  
    - Failure modes: Webhook not reachable, invalid input (missing required fields).  
    - Notes: Triggers the entire workflow start.

  - **Form**  
    - Type: n8n form node  
    - Role: Provides the UI for users to input data.  
    - Configuration: Custom CSS for branding and UI styling, completion message includes a dynamic Google Docs URL after generation.  
    - Input: Triggered by `CreateDocument` completion.  
    - Output: Provides user interface and feedback on the document creation status.

#### 2.2 Document Initialization

- **Overview:**  
  Initializes a new Google Document with the submitted title to serve as the output container for generated content.

- **Nodes Involved:**  
  - `CreateDocument`

- **Node Details:**  
  - **CreateDocument**  
    - Type: Google Docs node (create document)  
    - Role: Creates a blank Google Doc titled as per the user input.  
    - Configuration: Uses the form's Title field for document title, saves to default Google Drive folder.  
    - Credentials: Requires Google Docs OAuth2 credentials.  
    - Input: Triggered by form submission.  
    - Output: Returns document metadata including document ID and URL.  
    - Failure modes: Auth errors, API quota limits, network issues.

#### 2.3 Content Planning

- **Overview:**  
  Generates a detailed outline for the requested document using GPT-5-mini, returning structured sections with IDs, titles, and summaries.

- **Nodes Involved:**  
  - `gpt-5-mini` (Language Model)  
  - `Structured Output Parser`  
  - `ContentPlanner`

- **Node Details:**  
  - **gpt-5-mini**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Calls GPT-5-mini to generate the outline prompt.  
    - Configuration: Model set to "gpt-5-mini". No special options.  
    - Credentials: OpenAI API key.  
    - Input: Receives the document title from form submission.  
    - Output: Raw AI response.  
    - Potential issues: API rate limits, timeout (though none specified here).  

  - **Structured Output Parser**  
    - Type: LangChain structured output parser  
    - Role: Parses the raw AI response into a JSON structure with blogTitle and sections as per example schema.  
    - Configuration: JSON schema example provided for validation and parsing.  
    - Input: From `gpt-5-mini`.  
    - Output: Structured JSON object for next nodes.  
    - Failure modes: Parsing errors if AI output deviates from schema.

  - **ContentPlanner**  
    - Type: LangChain agent  
    - Role: Defines the prompt and system message for outline generation; uses structured output parser.  
    - Configuration: System prompt instructs generation of a full blog outline with sections and summaries, emphasizing clarity, SEO, and logical progression.  
    - Input: The title from the form.  
    - Output: Parsed structured outline JSON.  
    - Failure modes: AI misinterpretation, prompt failure.

#### 2.4 Section Extraction and Looping

- **Overview:**  
  Extracts the individual sections from the outline JSON and iterates over each section to generate content.

- **Nodes Involved:**  
  - `GetSections`  
  - `Split Out`  
  - `Loop Over Items`

- **Node Details:**  
  - **GetSections**  
    - Type: Set node  
    - Role: Extracts the "sections" array from the outline and generates a unique `documentId` for session context.  
    - Configuration: Assigns `sections` from the parsed output, creates random alphanumeric `documentId`.  
    - Input: From `ContentPlanner`.  
    - Output: JSON with sections array and documentId.  
    - Notes: `documentId` used for session tracking in memory node.

  - **Split Out**  
    - Type: Split Out node  
    - Role: Separates the "sections" array into individual items for batching.  
    - Configuration: Splits on the "sections" field, passing all other fields downstream.  
    - Input: From `GetSections`.  
    - Output: Multiple JSON items each representing one section.

  - **Loop Over Items**  
    - Type: Split In Batches node  
    - Role: Processes each section sequentially or in batches for content generation.  
    - Configuration: Default batch size (one at a time).  
    - Input: From `Split Out`.  
    - Output: Single section JSON per iteration.

#### 2.5 Section Content Generation

- **Overview:**  
  For each section, generates detailed Markdown content constrained by the word count per section, using GPT-5 and referencing a short memory buffer for context.

- **Nodes Involved:**  
  - `Simple Memory`  
  - `gpt-5` (Language Model)  
  - `ContentWriter`

- **Node Details:**  
  - **Simple Memory**  
    - Type: LangChain memory buffer window  
    - Role: Maintains a session-based short-term memory with a sliding window of 30 tokens for context continuity.  
    - Configuration: Uses `documentId` as session key, custom session ID type.  
    - Input: Receives section info from loop.  
    - Output: Provides memory context to the content writer.  
    - Failure modes: Missing or inconsistent session keys.

  - **gpt-5**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Generates the detailed section content.  
    - Configuration: Model set to "gpt-5" with a 10-minute timeout to handle longer generation.  
    - Credentials: OpenAI API key.  
    - Input: Prompted via `ContentWriter` agent.  
    - Failure modes: API timeout, rate limits, prompt errors.

  - **ContentWriter**  
    - Type: LangChain agent  
    - Role: Defines prompt for writing sections in Markdown with headings and word count limits.  
    - Configuration: System message instructs professional writing of Markdown sections, word count per section calculated dynamically by dividing total word count by section count.  
    - Input: Receives section JSON and memory context.  
    - Output: Detailed Markdown text for the section.  
    - Failure modes: Prompt misinterpretation, output formatting issues.

#### 2.6 Document Updating

- **Overview:**  
  Appends each generated section's content sequentially into the Google Document using its document ID.

- **Nodes Involved:**  
  - `UpdateDocument`

- **Node Details:**  
  - **UpdateDocument**  
    - Type: Google Docs node (update)  
    - Role: Inserts the generated content from each section into the Google Document.  
    - Configuration: Uses "insert" action to append text; document ID is passed from `CreateDocument`.  
    - Credentials: Google Docs OAuth2 credentials.  
    - Input: From `ContentWriter` output.  
    - Output: Updated document metadata.  
    - Failure modes: API rate limits, quota exceeded, auth failures.

#### 2.7 User Feedback

- **Overview:**  
  Provides user visibility on workflow progress and completion, including a link to the generated Google Document.

- **Nodes Involved:**  
  - `Form` (Completion message)

- **Node Details:**  
  - **Form** (completion)  
    - Type: UI node with user feedback  
    - Role: Shows the dynamic Google Docs link after document creation and content generation.  
    - Configuration: Completion message includes the document URL retrieved from `CreateDocument`.  
    - Failure modes: Link resolution errors if document ID not available.

#### 2.8 Documentation & Notes

- **Overview:**  
  Sticky Note node provides an extensive overview, setup instructions, customization tips, cost considerations, and workflow usage guidance.

- **Nodes Involved:**  
  - `Sticky Note4`

- **Node Details:**  
  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Documentation and user guidance inside the n8n editor.  
    - Content: Covers overview, setup steps, usage, customization, costs, and limitations.  
    - Positioned alongside nodes for easy reference.

---

### 3. Summary Table

| Node Name             | Node Type                             | Functional Role                   | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                   |
|-----------------------|-------------------------------------|---------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| On form submission     | n8n-nodes-base.formTrigger           | Collect user input               |                             | CreateDocument              |                                                                                               |
| CreateDocument        | n8n-nodes-base.googleDocs            | Create new Google Doc            | On form submission           | Form, ContentPlanner         |                                                                                               |
| Form                  | n8n-nodes-base.form                  | UI for input and completion link| CreateDocument               |                             | Completion message shows Google Docs link                                                    |
| gpt-5-mini            | @n8n/n8n-nodes-langchain.lmChatOpenAi| Generate document outline (AI)  |                             | Structured Output Parser     | Requires OpenAI API Key                                                                      |
| Structured Output Parser| @n8n/n8n-nodes-langchain.outputParserStructured | Parse AI output to JSON          | gpt-5-mini                   | ContentPlanner              |                                                                                               |
| ContentPlanner        | @n8n/n8n-nodes-langchain.agent       | Define outline prompt and parse | Structured Output Parser      | GetSections                 |                                                                                               |
| GetSections           | n8n-nodes-base.set                   | Extract sections & generate docId| ContentPlanner               | Split Out                   |                                                                                               |
| Split Out             | n8n-nodes-base.splitOut              | Split sections array             | GetSections                  | Loop Over Items             |                                                                                               |
| Loop Over Items       | n8n-nodes-base.splitInBatches        | Iterate over each section        | Split Out                   | Simple Memory (main 1), ContentWriter (main 2) |                                                                                               |
| Simple Memory         | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintain session context         | Loop Over Items              | ContentWriter               |                                                                                               |
| gpt-5                 | @n8n/n8n-nodes-langchain.lmChatOpenAi| Generate detailed section content|                             | ContentWriter               | Requires OpenAI API Key, 10-minute timeout                                                  |
| ContentWriter         | @n8n/n8n-nodes-langchain.agent       | Write section content in Markdown| Simple Memory, Loop Over Items | UpdateDocument            | Dynamic word count per section, professional writing prompt                                  |
| UpdateDocument        | n8n-nodes-base.googleDocs            | Append content to Google Doc     | ContentWriter                | Loop Over Items             | Requires Google Docs OAuth2 credentials                                                     |
| Sticky Note4          | n8n-nodes-base.stickyNote            | Documentation and guidance       |                             |                             | Contains detailed overview, setup, usage, customization, and cost information                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**  
   - Type: `Form Trigger`  
   - Name: `On form submission`  
   - Configure webhook with form titled "Generate Long Content"  
   - Fields:  
     - Title (string, required)  
     - Word Count (number, required)  

2. **Create Google Docs Node to Create Document:**  
   - Type: `Google Docs` (Create operation)  
   - Name: `CreateDocument`  
   - Title parameter: set to `={{ $('On form submission').item.json.Title }}`  
   - Folder ID: default (or specify folder)  
   - Credentials: Connect Google Docs OAuth2 credentials  
   - Connect output from `On form submission` to this node.

3. **Create Form UI Node for Feedback:**  
   - Type: `Form`  
   - Name: `Form`  
   - Configure custom CSS for styling (optional)  
   - Completion message:  
     `=https://docs.google.com/document/d/{{ $('CreateDocument').item.json.id }}`  
   - Connect output from `CreateDocument` to this node.

4. **Add GPT-5-mini Node for Outline Creation:**  
   - Type: `LangChain LM Chat OpenAI`  
   - Name: `gpt-5-mini`  
   - Model: `gpt-5-mini`  
   - Credentials: OpenAI API key  
   - Connect output from `On form submission` to this node.

5. **Add Structured Output Parser Node:**  
   - Type: `LangChain Output Parser Structured`  
   - Name: `Structured Output Parser`  
   - Provide JSON schema example for outline structure  
   - Connect output from `gpt-5-mini` to this node.

6. **Create LangChain Agent Node for Content Planning:**  
   - Type: `LangChain Agent`  
   - Name: `ContentPlanner`  
   - Prompt: Use the system message specifying role as content planner and instructions for outline generation  
   - Enable output parser usage  
   - Connect output from `Structured Output Parser` to this node.

7. **Create Set Node to Extract Sections and Generate Document ID:**  
   - Type: `Set`  
   - Name: `GetSections`  
   - Assignments:  
     - `sections` = `={{ $json.output.sections }}`  
     - `documentId` = `={{ Math.random().toString(36).substring(2, 10).toUpperCase() }}`  
   - Connect output from `ContentPlanner` to this node.

8. **Add Split Out Node to Separate Sections:**  
   - Type: `Split Out`  
   - Name: `Split Out`  
   - Field to split out: `sections`  
   - Include all other fields  
   - Connect output from `GetSections` to this node.

9. **Add Split In Batches Node for Section Looping:**  
   - Type: `Split In Batches`  
   - Name: `Loop Over Items`  
   - Default batch size (1)  
   - Connect output from `Split Out` to this node.

10. **Add Memory Buffer Window Node:**  
    - Type: `LangChain Memory Buffer Window`  
    - Name: `Simple Memory`  
    - Session key: `={{ $('GetSections').item.json.documentId }}`  
    - Session ID type: `customKey`  
    - Context window length: 30  
    - Connect `Loop Over Items` main output index 1 to this node.

11. **Add GPT-5 Node for Content Generation:**  
    - Type: `LangChain LM Chat OpenAI`  
    - Name: `gpt-5`  
    - Model: `gpt-5`  
    - Timeout: 600000 ms (10 minutes)  
    - Credentials: OpenAI API key  
    - Connect `Simple Memory` output and/or `Loop Over Items` main output index 2 to the next node.

12. **Add LangChain Agent Node for Writing Section Content:**  
    - Type: `LangChain Agent`  
    - Name: `ContentWriter`  
    - Prompt: System message instructs detailed Markdown section writing with word count per section calculated dynamically  
    - Disable passthrough binary images  
    - Connect output from `gpt-5` to this node.

13. **Add Google Docs Node to Update Document:**  
    - Type: `Google Docs` (Update operation)  
    - Name: `UpdateDocument`  
    - Operation: Update  
    - Action: Insert text from `ContentWriter` output  
    - Document URL: `={{ $('CreateDocument').item.json.id }}`  
    - Credentials: Google Docs OAuth2 credentials  
    - Connect output from `ContentWriter` to this node.

14. **Connect UpdateDocument output back to Loop Over Items:**  
    - This will allow looping through all sections sequentially.

15. **Add Sticky Note for Documentation (Optional):**  
    - Create a Sticky Note node with the overview, setup instructions, usage, customization, and limits as described.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow takes a title and automatically creates a detailed Google Document by building an outline, writing each section based on the word limit, and saving everything to Google Docs. Setup requires Google Docs OAuth and OpenAI API credentials. The form URL is public for user submissions. Customization options include changing AI models, prompts, document folder, and form styling. Costs scale with word count and number of sections, and API limits apply. The workflow handles Google Docs API write-rate limits by appending content one section at a time. | Internal Sticky Note within workflow editor.  |
| OpenAI API token usage influences cost and speed; consider adjusting word counts or batching for longer documents. The GPT-5 node has a 10-minute timeout to accommodate large generations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | OpenAI API best practices                      |
| Google Docs API quotas and rate limits require cautious document update pacing; this workflow appends section content sequentially to avoid quota issues.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Google Docs API documentation                  |
| The form node’s custom CSS can be modified to fit branding needs. The completion message dynamically generates the Google Docs URL to share with users after generation completes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | n8n Form node documentation                     |

---

This documentation provides a complete, detailed, and structured reference to understand, reproduce, and modify the “Create Long-Form Documents from Simple Titles with GPT-5 and Google Docs” workflow in n8n. It anticipates potential failure modes and integration considerations for robust operation.