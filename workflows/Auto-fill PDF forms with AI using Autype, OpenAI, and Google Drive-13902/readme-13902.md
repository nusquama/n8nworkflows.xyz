Auto-fill PDF forms with AI using Autype, OpenAI, and Google Drive

https://n8nworkflows.xyz/workflows/auto-fill-pdf-forms-with-ai-using-autype--openai--and-google-drive-13902


# Auto-fill PDF forms with AI using Autype, OpenAI, and Google Drive

# 1. Workflow Overview

This workflow automates the end-to-end filling of a PDF form using AI. It downloads a fillable PDF from a public URL, uploads it to Autype, extracts the form field metadata, asks an OpenAI-powered AI Agent to map known applicant data to those fields, validates the AI output, fills and flattens the PDF via Autype, and finally saves the completed document to Google Drive.

Typical use cases include repetitive completion of structured PDF forms such as registrations, compliance forms, declarations, and application documents where the form layout may vary but field names and types can be inspected programmatically.

## 1.1 Input Reception and PDF Acquisition

The workflow starts manually and fetches a sample fillable PDF from a Google Drive direct-download URL.

## 1.2 PDF Upload and Form Field Extraction

The downloaded binary PDF is uploaded to Autype, which then runs a form-field extraction operation. Because the extraction appears to be asynchronous, the workflow waits briefly and then checks the Autype job status.

## 1.3 AI-Based Field Mapping

The extracted field metadata is inserted directly into an AI Agent prompt together with hardcoded applicant data. The AI is instructed to return a JSON object that maps PDF field names to values, respecting field types and available options.

## 1.4 AI Output Validation and Fill Payload Preparation

A Code node parses the AI response, attempts recovery if the model wrapped JSON in Markdown fences, and builds the payload required by Autype’s form-filling operation.

## 1.5 PDF Filling and Delivery

Autype fills the PDF form, flattens it to make the result non-editable, returns the output file, and the workflow uploads the finished PDF to Google Drive.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and PDF Acquisition

### Overview

This block triggers the workflow manually and downloads the source PDF form as a binary file. It provides the raw input document that all downstream operations depend on.

### Nodes Involved

- Run Workflow
- Download PDF Form

### Node Details

#### Run Workflow
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point used for testing or on-demand execution.
- **Configuration choices:**  
  No parameters are set. It simply starts the flow when the user clicks execute.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Download PDF Form`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None beyond normal manual execution behavior.
- **Sub-workflow reference:**  
  Not a sub-workflow node.

#### Download PDF Form
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the PDF form file from a public URL.
- **Configuration choices:**  
  - HTTP URL points to a Google Drive direct-download endpoint:
    `https://drive.google.com/uc?export=download&id=19FEbkEjFA1H8jFcqHcDb5yGrJoiOe7-u`
  - Response is configured as a **file**, so binary data is emitted rather than JSON/text.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Run Workflow`
  - Output: `Upload PDF Form`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - URL no longer publicly accessible
  - HTTP 403/404 from Google Drive
  - Download returns an HTML interstitial instead of the PDF
  - Large-file or timeout issues
  - Binary property naming assumptions downstream if node defaults are altered
- **Sub-workflow reference:**  
  Not a sub-workflow node.

---

## 2.2 PDF Upload and Form Field Extraction

### Overview

This block sends the PDF to Autype and starts the form-field extraction process. Because the extraction returns a job-like result rather than immediate field metadata, the workflow waits and then checks the tool job status.

### Nodes Involved

- Upload PDF Form
- Get Form Fields
- Wait
- Get tool job status

### Node Details

#### Upload PDF Form
- **Type and technical role:** `n8n-nodes-autype.autype`  
  Uploads a file into Autype so document tools can operate on it.
- **Configuration choices:**  
  - Resource: `file`
  - Uses Autype API credentials
- **Key expressions or variables used:**  
  None visible in parameters.
- **Input and output connections:**  
  - Input: `Download PDF Form`
  - Output: `Get Form Fields`
- **Version-specific requirements:**  
  Type version `1`. Requires the community package `n8n-nodes-autype`, which is typically only available on self-hosted n8n.
- **Edge cases or potential failure types:**  
  - Missing or invalid Autype API key
  - Uploaded input not recognized as a PDF
  - Binary input missing because upstream response format changed
  - API quota, upload size, or service availability issues
- **Sub-workflow reference:**  
  Not a sub-workflow node.

#### Get Form Fields
- **Type and technical role:** `n8n-nodes-autype.autype`  
  Calls Autype document tools to inspect the uploaded PDF and extract field metadata.
- **Configuration choices:**  
  - Resource: `documentTools`
  - Operation: `formFields`
  - `fileId` is read from the previous node output using:
    `={{ $json.id }}`
- **Key expressions or variables used:**  
  - `$json.id` from `Upload PDF Form`
- **Input and output connections:**  
  - Input: `Upload PDF Form`
  - Output: `Wait`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - If Autype returns a job object rather than final metadata, downstream nodes must not assume fields are already available
  - Invalid or expired file ID
  - Unsupported PDF form structure
  - No fillable fields found
- **Sub-workflow reference:**  
  Not a sub-workflow node.

#### Wait
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution to give the asynchronous Autype job time to complete.
- **Configuration choices:**  
  - Amount: `10`
  - No unit is shown in JSON; in practice this depends on node defaults/UI interpretation and should be verified during rebuild.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Get Form Fields`
  - Output: `Get tool job status`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Wait period too short, causing status checks before the job completes
  - Wait node configuration ambiguity if unit is not explicitly chosen
- **Sub-workflow reference:**  
  Not a sub-workflow node.

#### Get tool job status
- **Type and technical role:** `n8n-nodes-autype.autype`  
  Retrieves the result/status of the asynchronous Autype document-tools job.
- **Configuration choices:**  
  - Resource: `documentTools`
  - Operation: `getJobStatus`
  - `jobId` comes from the `Get Form Fields` node:
    `={{ $('Get Form Fields').item.json.id }}`
- **Key expressions or variables used:**  
  - `$('Get Form Fields').item.json.id`
- **Input and output connections:**  
  - Input: `Wait`
  - Output: `AI Agent`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Job still processing when checked
  - Job failed in Autype
  - Expression fails if `Get Form Fields` did not return `id`
  - Returned payload shape may differ from expected structure used later in the AI prompt
- **Sub-workflow reference:**  
  Not a sub-workflow node.

---

## 2.3 AI-Based Field Mapping

### Overview

This block uses an LLM through n8n’s LangChain AI Agent to map extracted PDF fields to applicant values. The form metadata is embedded directly into the prompt, and the model is instructed to output only a JSON object.

### Nodes Involved

- OpenAI Chat Model
- AI Agent

### Node Details

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the AI Agent with an OpenAI chat model.
- **Configuration choices:**  
  - Model selected: `gpt-5-mini`
  - No extra options are set
  - Uses OpenAI API credentials
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none through main flow
  - AI output connection: connected to `AI Agent` via `ai_languageModel`
- **Version-specific requirements:**  
  Type version `1.2`. Model availability depends on the connected OpenAI account and the n8n/OpenAI integration version.
- **Edge cases or potential failure types:**  
  - Invalid OpenAI credentials
  - Model not available for the account
  - Rate limits or token limits
  - Prompt/response mismatch if model ignores JSON-only instruction
- **Sub-workflow reference:**  
  Not a sub-workflow node.

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Executes a prompt-driven reasoning step using the connected OpenAI chat model and emits the generated mapping.
- **Configuration choices:**  
  - Prompt type: `define`
  - Max iterations: `3`
  - The user message contains:
    - A task description: fill out a PDF registration form
    - Hardcoded applicant data:
      - Full Name: Maria Schmidt
      - ID Number: 1234560001
      - Gender: Female
      - Married: yes
      - City: Berlin
      - Preferred Language: German
      - Notes including `{{ $now.format('yyyy-MM-dd') }}`
    - A dynamic list of form fields generated from extracted metadata:
      ```js
      {{ $json.metadata.fields.map(f => '- "' + f.name + '" (' + f.type + ')' + (f.options && f.options.length > 0 ? ' [options: ' + f.options.join(', ') + ']' : '') + (f.isReadOnly ? ' [read-only, skip]' : '')).join('\n') }}
      ```
    - A final instruction: map each field name to the correct value
  - System message instructs the model to:
    - Return a JSON object
    - Use exact option strings for dropdown/radio fields
    - Use booleans for checkboxes
    - Skip read-only fields
- **Key expressions or variables used:**  
  - `$now.format('yyyy-MM-dd')`
  - `$json.metadata.fields.map(...)`
- **Input and output connections:**  
  - Main input: `Get tool job status`
  - Main output: `Prepare Fill Data`
  - AI language model input: `OpenAI Chat Model`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - `metadata.fields` missing or shaped differently than expected
  - Expression error if `metadata` is undefined
  - Model returns prose or Markdown instead of raw JSON
  - Model chooses invalid dropdown/radio option labels
  - Token growth if the form contains many fields/options
- **Sub-workflow reference:**  
  Not a sub-workflow node.

---

## 2.4 AI Output Validation and Fill Payload Preparation

### Overview

This block converts the AI output into a valid JSON string for Autype’s fill operation and retrieves the original Autype file ID from the upload step. It acts as a guardrail between AI-generated content and the final document write operation.

### Nodes Involved

- Prepare Fill Data

### Node Details

#### Prepare Fill Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Validates and normalizes the AI response, then packages the fill payload and the PDF file ID together.
- **Configuration choices:**  
  JavaScript code performs the following:
  1. Reads `raw = $json.output`
  2. Reads `fileId` from `Upload PDF Form` using:
     `$('Upload PDF Form').first().json.id`
  3. Attempts to parse the AI output as JSON
  4. If parsing fails, removes surrounding Markdown code fences and retries
  5. Returns:
     - `fields`: `JSON.stringify(fields)`
     - `fileId`
- **Key expressions or variables used:**  
  - `$json.output`
  - `$('Upload PDF Form').first().json.id`
- **Input and output connections:**  
  - Input: `AI Agent`
  - Output: `Fill PDF Form`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - `output` missing from AI Agent result
  - AI returns invalid JSON that cannot be repaired
  - Cross-node reference to `Upload PDF Form` fails if node name changes
  - Returned structure from AI is not a flat object mapping field names to values
- **Sub-workflow reference:**  
  Not a sub-workflow node.

---

## 2.5 PDF Filling and Delivery

### Overview

This block submits the validated field mapping to Autype to fill the PDF, flattens the document to prevent editing, and stores the finished file in Google Drive.

### Nodes Involved

- Fill PDF Form
- Save Filled PDF to Drive

### Node Details

#### Fill PDF Form
- **Type and technical role:** `n8n-nodes-autype.autype`  
  Executes the actual PDF form fill using the prepared field-value JSON.
- **Configuration choices:**  
  - Resource: `documentTools`
  - Operation: `fillForm`
  - `fileId`: `={{ $json.fileId }}`
  - `fields`: `={{ $json.fields }}`
  - `flatten`: `true`
  - `downloadOutput`: `true` so the filled PDF is returned as a file for the next node
- **Key expressions or variables used:**  
  - `$json.fileId`
  - `$json.fields`
- **Input and output connections:**  
  - Input: `Prepare Fill Data`
  - Output: `Save Filled PDF to Drive`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Invalid JSON string in `fields`
  - Field names do not match actual PDF field names
  - Unsupported value types for specific field kinds
  - Autype fill or rendering failure
  - Flattening may remove interactivity irreversibly
- **Sub-workflow reference:**  
  Not a sub-workflow node.

#### Save Filled PDF to Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the completed PDF to a chosen Google Drive folder.
- **Configuration choices:**  
  - File name:
    `=filled-form-{{ $now.format('yyyy-MM-dd') }}.pdf`
  - Drive: `My Drive`
  - Folder ID: manually supplied as `YOUR_FOLDER_ID`
  - Uses Google Drive OAuth2 credentials
- **Key expressions or variables used:**  
  - `$now.format('yyyy-MM-dd')`
- **Input and output connections:**  
  - Input: `Fill PDF Form`
  - Output: none
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - Missing/invalid Google Drive OAuth2 credentials
  - Folder ID invalid or inaccessible
  - Binary file property mismatch
  - Upload permissions or quota issues
- **Sub-workflow reference:**  
  Not a sub-workflow node.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Workflow | Manual Trigger | Manual start of the workflow |  | Download PDF Form | ## Auto-Fill PDF Forms with AI Using Autype and OpenAI<br>### This workflow downloads a fillable PDF form from a URL, extracts all form fields using Autype, sends the field list to an AI Agent (OpenAI) together with applicant data, and uses the structured AI response to fill the form automatically. The filled PDF is flattened (non-editable) and saved to Google Drive.<br><br>**The sample form has these fields:**<br>- FullName (text)<br>- ID (text)<br>- Gender (radio: Male / Female)<br>- Married (checkbox: true / false)<br>- City (dropdown: e.g. New York, Berlin, ...)<br>- Language (optionlist: e.g. English, German, ...)<br>- Notes (text)<br><br>**Use Case:** A company that regularly fills the same government or compliance PDF forms -- permit applications, registrations, insurance declarations. The AI reads whatever fields the form has and maps the right values automatically.<br><br>Click "Test Workflow" to run it once with the sample form.<br><br>### How it works<br>1. **Download PDF Form** -- Fetches the fillable PDF from a URL.<br>2. **Upload PDF Form** -- Uploads it to Autype Tools.<br>3. **Get Form Fields** -- Autype extracts field names, types, options, read-only flags as metadata.<br>4. **AI Agent** -- Receives the field list and applicant data as a prompt. The system message instructs the AI to return raw JSON only.<br>5. **Prepare Fill Data** -- Parses and validates the AI JSON output, pairs it with the file ID.<br>6. **Fill PDF Form** -- Autype fills all fields and flattens them (non-editable).<br>7. **Save Filled PDF to Drive** -- Uploads the completed PDF to Google Drive.<br><br>### Requirements<br>* **Autype account** -- [app.autype.com](https://app.autype.com) > Settings > API Keys<br>* **OpenAI account** -- API key from [platform.openai.com](https://platform.openai.com)<br>* **Google Drive** -- OAuth2 credential in n8n (optional)<br>* **n8n-nodes-autype** -- Install via Settings > Community Nodes<br><br>> Community node: requires self-hosted n8n. Not available on n8n Cloud. |
| Download PDF Form | HTTP Request | Download source PDF as binary file | Run Workflow | Upload PDF Form | ### 1. Download & Extract Form Fields<br>Downloads the PDF form from a URL and uploads it to Autype. The Get Form Fields operation returns all field names, types, current values, and options as metadata (no output file). The field info flows directly into the AI Agent prompt via an expression. |
| Upload PDF Form | Autype | Upload PDF into Autype | Download PDF Form | Get Form Fields | ### 1. Download & Extract Form Fields<br>Downloads the PDF form from a URL and uploads it to Autype. The Get Form Fields operation returns all field names, types, current values, and options as metadata (no output file). The field info flows directly into the AI Agent prompt via an expression. |
| Get Form Fields | Autype | Start or request Autype form field extraction | Upload PDF Form | Wait | ### 1. Download & Extract Form Fields<br>Downloads the PDF form from a URL and uploads it to Autype. The Get Form Fields operation returns all field names, types, current values, and options as metadata (no output file). The field info flows directly into the AI Agent prompt via an expression. |
| Wait | Wait | Pause before polling Autype job status | Get Form Fields | Get tool job status | ### 1. Download & Extract Form Fields<br>Downloads the PDF form from a URL and uploads it to Autype. The Get Form Fields operation returns all field names, types, current values, and options as metadata (no output file). The field info flows directly into the AI Agent prompt via an expression. |
| Get tool job status | Autype | Retrieve extracted field metadata after async processing | Wait | AI Agent | ### 1. Download & Extract Form Fields<br>Downloads the PDF form from a URL and uploads it to Autype. The Get Form Fields operation returns all field names, types, current values, and options as metadata (no output file). The field info flows directly into the AI Agent prompt via an expression. |
| AI Agent | AI Agent | Generate field-to-value JSON mapping from applicant data and field metadata | Get tool job status | Prepare Fill Data | ### 2. AI Agent Fills Fields<br>The AI Agent node receives the form fields and applicant data directly in its prompt (no separate Code node needed). The system message instructs the AI to return only a raw JSON object. The Prepare Fill Data code node validates and parses the response. Edit the applicant data directly in the AI Agent prompt. |
| OpenAI Chat Model | OpenAI Chat Model | LLM backend for the AI Agent |  | AI Agent | ### 2. AI Agent Fills Fields<br>The AI Agent node receives the form fields and applicant data directly in its prompt (no separate Code node needed). The system message instructs the AI to return only a raw JSON object. The Prepare Fill Data code node validates and parses the response. Edit the applicant data directly in the AI Agent prompt. |
| Prepare Fill Data | Code | Parse/validate AI JSON and prepare Autype fill payload | AI Agent | Fill PDF Form | ### 2. AI Agent Fills Fields<br>The AI Agent node receives the form fields and applicant data directly in its prompt (no separate Code node needed). The system message instructs the AI to return only a raw JSON object. The Prepare Fill Data code node validates and parses the response. Edit the applicant data directly in the AI Agent prompt.<br>### 3. Fill Form & Save<br>The Prepare Fill Data code node parses and validates the AI JSON, then passes it to the Autype Fill Form operation. Fields are flattened (non-editable). The result is saved to Google Drive. Replace the Drive node with Email, S3, Slack, or any other output. |
| Fill PDF Form | Autype | Fill and flatten the PDF form, returning the completed file | Prepare Fill Data | Save Filled PDF to Drive | ### 3. Fill Form & Save<br>The Prepare Fill Data code node parses and validates the AI JSON, then passes it to the Autype Fill Form operation. Fields are flattened (non-editable). The result is saved to Google Drive. Replace the Drive node with Email, S3, Slack, or any other output. |
| Save Filled PDF to Drive | Google Drive | Upload completed PDF to Google Drive | Fill PDF Form |  | ### 3. Fill Form & Save<br>The Prepare Fill Data code node parses and validates the AI JSON, then passes it to the Autype Fill Form operation. Fields are flattened (non-editable). The result is saved to Google Drive. Replace the Drive node with Email, S3, Slack, or any other output. |
| Sticky Note | Sticky Note | Canvas documentation and requirements |  |  | ## Auto-Fill PDF Forms with AI Using Autype and OpenAI<br>### This workflow downloads a fillable PDF form from a URL, extracts all form fields using Autype, sends the field list to an AI Agent (OpenAI) together with applicant data, and uses the structured AI response to fill the form automatically. The filled PDF is flattened (non-editable) and saved to Google Drive.<br><br>**The sample form has these fields:**<br>- FullName (text)<br>- ID (text)<br>- Gender (radio: Male / Female)<br>- Married (checkbox: true / false)<br>- City (dropdown: e.g. New York, Berlin, ...)<br>- Language (optionlist: e.g. English, German, ...)<br>- Notes (text)<br><br>**Use Case:** A company that regularly fills the same government or compliance PDF forms -- permit applications, registrations, insurance declarations. The AI reads whatever fields the form has and maps the right values automatically.<br><br>Click "Test Workflow" to run it once with the sample form.<br><br>### How it works<br>1. **Download PDF Form** -- Fetches the fillable PDF from a URL.<br>2. **Upload PDF Form** -- Uploads it to Autype Tools.<br>3. **Get Form Fields** -- Autype extracts field names, types, options, read-only flags as metadata.<br>4. **AI Agent** -- Receives the field list and applicant data as a prompt. The system message instructs the AI to return raw JSON only.<br>5. **Prepare Fill Data** -- Parses and validates the AI JSON output, pairs it with the file ID.<br>6. **Fill PDF Form** -- Autype fills all fields and flattens them (non-editable).<br>7. **Save Filled PDF to Drive** -- Uploads the completed PDF to Google Drive.<br><br>### Requirements<br>* **Autype account** -- [app.autype.com](https://app.autype.com) > Settings > API Keys<br>* **OpenAI account** -- API key from [platform.openai.com](https://platform.openai.com)<br>* **Google Drive** -- OAuth2 credential in n8n (optional)<br>* **n8n-nodes-autype** -- Install via Settings > Community Nodes<br><br>> Community node: requires self-hosted n8n. Not available on n8n Cloud. |
| Sticky Note1 | Sticky Note | Canvas annotation for extraction block |  |  | ### 1. Download & Extract Form Fields<br>Downloads the PDF form from a URL and uploads it to Autype. The Get Form Fields operation returns all field names, types, current values, and options as metadata (no output file). The field info flows directly into the AI Agent prompt via an expression. |
| Sticky Note2 | Sticky Note | Canvas annotation for AI block |  |  | ### 2. AI Agent Fills Fields<br>The AI Agent node receives the form fields and applicant data directly in its prompt (no separate Code node needed). The system message instructs the AI to return only a raw JSON object. The Prepare Fill Data code node validates and parses the response. Edit the applicant data directly in the AI Agent prompt. |
| Sticky Note3 | Sticky Note | Canvas annotation for fill/save block |  |  | ### 3. Fill Form & Save<br>The Prepare Fill Data code node parses and validates the AI JSON, then passes it to the Autype Fill Form operation. Fields are flattened (non-editable). The result is saved to Google Drive. Replace the Drive node with Email, S3, Slack, or any other output. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Auto-fill PDF forms with AI using Autype and OpenAI`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name it: `Run Workflow`
   - No additional configuration required.

3. **Add an HTTP Request node to download the PDF**
   - Node type: `HTTP Request`
   - Name it: `Download PDF Form`
   - Set the URL to the source PDF, for example:
     `https://drive.google.com/uc?export=download&id=19FEbkEjFA1H8jFcqHcDb5yGrJoiOe7-u`
   - In response settings, choose **File** as the response format.
   - Keep the binary output available for the next node.
   - Connect `Run Workflow -> Download PDF Form`.

4. **Install the Autype community node if not already installed**
   - In self-hosted n8n, go to **Settings > Community Nodes**
   - Install `n8n-nodes-autype`
   - This workflow depends on that package.
   - Note: community nodes are generally unavailable on n8n Cloud.

5. **Create Autype credentials**
   - In n8n credentials, create an `Autype API` credential.
   - Use your API key from:
     [https://app.autype.com](https://app.autype.com)
   - Save the credential with a recognizable name, for example `Autype account`.

6. **Add an Autype node to upload the PDF**
   - Node type: `Autype`
   - Name it: `Upload PDF Form`
   - Resource: `file`
   - Use the Autype credential created above.
   - Connect `Download PDF Form -> Upload PDF Form`.
   - This node should receive the binary PDF from the HTTP Request node.

7. **Add an Autype node to request form field extraction**
   - Node type: `Autype`
   - Name it: `Get Form Fields`
   - Resource: `documentTools`
   - Operation: `formFields`
   - Set `fileId` to this expression:
     `={{ $json.id }}`
   - Connect `Upload PDF Form -> Get Form Fields`.

8. **Add a Wait node**
   - Node type: `Wait`
   - Name it: `Wait`
   - Set the wait amount to `10`
   - Verify the unit in the UI and explicitly choose an appropriate one, typically seconds.
   - Connect `Get Form Fields -> Wait`.

9. **Add an Autype node to retrieve the job result**
   - Node type: `Autype`
   - Name it: `Get tool job status`
   - Resource: `documentTools`
   - Operation: `getJobStatus`
   - Set `jobId` to:
     `={{ $('Get Form Fields').item.json.id }}`
   - Connect `Wait -> Get tool job status`.

10. **Create OpenAI credentials**
    - In n8n credentials, create an `OpenAI API` credential.
    - Use your API key from:
      [https://platform.openai.com](https://platform.openai.com)
    - Save it, for example, as `OpenAI account`.

11. **Add an OpenAI Chat Model node**
    - Node type: `OpenAI Chat Model`
    - Name it: `OpenAI Chat Model`
    - Select model: `gpt-5-mini`
    - Attach your OpenAI credential.
    - Do not connect it via the normal main connection.

12. **Add an AI Agent node**
    - Node type: `AI Agent`
    - Name it: `AI Agent`
    - Set **Prompt Type** to `Define`.
    - Set **Max Iterations** to `3`.
    - Set the main prompt text to:

      ```text
      Fill out a PDF registration form.

      Person/company data:
      - Full Name: Maria Schmidt
      - ID Number: 1234560001
      - Gender: Female
      - Married: yes
      - City: Berlin
      - Preferred Language: German
      - Notes: Submitted via automated workflow on {{ $now.format('yyyy-MM-dd') }}

      The PDF form has these fields (from Autype Get Form Fields):

      {{ $json.metadata.fields.map(f => '- "' + f.name + '" (' + f.type + ')' + (f.options && f.options.length > 0 ? ' [options: ' + f.options.join(', ') + ']' : '') + (f.isReadOnly ? ' [read-only, skip]' : '')).join('\n') }}

      Map each field name to the correct value from the data above.
      ```

    - Set the **System Message** to:

      ```text
      You are a precise form-filling assistant. You receive applicant data and a list of PDF form fields with their types and allowed options. Return a JSON object mapping each field name to its value. Use exact option strings for dropdowns and radio groups. Use booleans for checkboxes. Skip read-only fields.
      ```

    - Connect the `OpenAI Chat Model` node to the `AI Agent` using the AI language model connector.
    - Connect `Get tool job status -> AI Agent`.

13. **Add a Code node to validate the AI output**
    - Node type: `Code`
    - Name it: `Prepare Fill Data`
    - Use JavaScript.
    - Paste this code:

      ```javascript
      const raw = $json.output;
      const fileId = $('Upload PDF Form').first().json.id;

      // The AI Agent returns a JSON string — parse and re-stringify to validate
      let fields;
      try {
        fields = typeof raw === 'string' ? JSON.parse(raw) : raw;
      } catch (e) {
        // Strip markdown fences if present and retry
        let cleaned = raw.replace(/^```(?:json)?\