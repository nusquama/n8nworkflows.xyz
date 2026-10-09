Summarize meeting transcripts into Google Docs with Claude via OpenRouter

https://n8nworkflows.xyz/workflows/summarize-meeting-transcripts-into-google-docs-with-claude-via-openrouter-20535


# Summarize meeting transcripts into Google Docs with Claude via OpenRouter

### 1. Workflow Overview

This workflow automates the ingestion, AI-powered summarization, and archiving of meeting transcripts into a central Google Document. It is designed for teams aiming to maintain a running, chronologically-ordered log of meeting minutes, action items, and owners without manual formatting. 

The workflow relies on a sequential, linear pipeline divided into three functional blocks:
- **1.1 Ingestion & Preparation Block:** Receives transcript files through an interactive web form, injects global configuration parameters, and performs rigorous client-side data validation and text sanitization.
- **1.2 AI Generation & Validation Block:** Compiles a context-aware system prompt, interacts with Claude via the OpenRouter API, and strictly validates the returned JSON structure against length, entity, and temporal constraints.
- **1.3 Document Archival & Response Block:** Exports the destination Google Doc as HTML, prepends the newly generated meeting minutes at the top of the file (separated from historical entries), uploads the revised document back to Google Drive, and renders a completion response screen for the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Ingestion & Preparation Block

##### Overview
This block serves as the user entry point, capturing uploaded raw transcripts via a web form, merging them with static team configurations, and sanitizing the text payload while determining metadata such as meeting dates, titles, and speakers.

##### Nodes Involved
- `Upload transcript via form`
- `Set configuration`
- `Prepare transcript and meeting details`

##### Node Details

###### Upload transcript via form
- **Type and Technical Role:** `n8n-nodes-base.formTrigger` (Webhook/Form Trigger)
- **Configuration Choices:** Configured as a Version 2.2 form trigger with a custom button label ("Create notes") and title ("Upload a meeting transcript"). Accepts a single file field restricted to `.txt` extensions (`multipleFiles: false`, `requiredField: true`), an optional date field (`Meeting date`), and an optional text field (`Meeting title`).
- **Key Expressions or Variables:** N/A (Triggers execution on form submission).
- **Input and Output Connections:** No input nodes; outputs to `Set configuration`.
- **Version-Specific Requirements:** Version 2.2+.
- **Edge Cases or Potential Failure Types:** Users uploading non-text formats or submitting empty files will fail subsequent validation checks.

###### Set configuration
- **Type and Technical Role:** `n8n-nodes-base.set` (Parameter Assignment)
- **Configuration Choices:** Defines global workflow variables including `notesDocId`, `organisationName`, `teamMembers`, `meetingDays`, `timezone`, `glossary`, `aiModel`, `maxSummaryWords`, and `maxTranscriptChars`.
- **Key Expressions or Variables:** Static string and numerical assignments acting as constant environment configurations for downstream code nodes.
- **Input and Output Connections:** Input from `Upload transcript via form`; output to `Prepare transcript and meeting details`.
- **Version-Specific Requirements:** Version 3.4+.
- **Edge Cases or Potential Failure Types:** Leaving `notesDocId` set to its placeholder value (`YOUR_...`) will trigger an explicit runtime exception in the subsequent code node.

###### Prepare transcript and meeting details
- **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration Choices:** Executes custom ES6 JavaScript to parse binary or combined-JSON file objects, enforce strict plain-text validation, compute timezones using `Intl.DateTimeFormat`, sanitize whitespace/byte-order marks, derive date sources (Form, Filename prefix, or Upload Date), and compute expected speaker label frequencies.
- **Key Expressions or Variables:** Reads variables from `$('Set configuration')` and `$('Upload transcript via form')`.
- **Input and Output Connections:** Input from `Set configuration`; output to `Build Claude request`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases or Potential Failure Types:** Throws errors for empty files, invalid MIME types, character length violations exceeding `maxTranscriptChars`, or invalid IANA timezones.

---

#### 2.2 AI Generation & Validation Block

##### Overview
This block constructs the prompt architecture containing metadata, system constraints, and transcript payloads, dispatches requests to Claude via OpenRouter, and enforces strict schema compliance on the returned JSON.

##### Nodes Involved
- `Build Claude request`
- `Ask Claude for notes via OpenRouter`
- `Validate Claude notes`

##### Node Details

###### Build Claude request
- **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration Choices:** Formulates a comprehensive system prompt enforcing precise extraction rules for agendas, topic bullets, and action items (including strict constraints on owners and due dates). Serializes the resulting payload into an OpenAI-compatible JSON structure.
- **Key Expressions or Variables:** Accesses dynamic fields from `$('Set configuration')` and the preceding code node (`$input.first().json`).
- **Input and Output Connections:** Input from `Prepare transcript and meeting details`; output to `Ask Claude for notes via OpenRouter`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases or Potential Failure Types:** Large transcripts may approach token context limits if truncation or limits are bypassed.

###### Ask Claude for notes via OpenRouter
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Integration)
- **Configuration Choices:** Issues a `POST` request to `https://openrouter.ai/api/v1/chat/completions` with a 300,000ms timeout. Configured with automatic retries (`maxTries: 3`, `waitBetweenTries: 5000ms`) and generic header authentication.
- **Key Expressions or Variables:** Body populated dynamically via `={{ $json.requestBody }}`.
- **Input and Output Connections:** Input from `Build Claude request`; output to `Validate Claude notes`.
- **Version-Specific Requirements:** Version 4.2+.
- **Edge Cases or Potential Failure Types:** Authentication failures, insufficient OpenRouter credits, API downtime, or request timeouts.

###### Validate Claude notes
- **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript Data Validation)
- **Configuration Choices:** Parses raw LLM output, strips markdown code-blocks, and validates array structures, word count bounds, agenda-to-topic parity, task sentence limits, valid team member owners, and real ISO date formatting.
- **Key Expressions or Variables:** References outputs from `$('Set configuration')`, `$('Prepare transcript and meeting details')`, and `$input.first().json`.
- **Input and Output Connections:** Input from `Ask Claude for notes via OpenRouter`; output to `Export notes Doc as HTML from Drive`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases or Potential Failure Types:** Truncated outputs due to token exhaustion (`finish_reason === 'length'`) or schema deviations throw detailed array-based error logs halting the workflow before file modifications occur.

---

#### 2.3 Document Archival & Response Block

##### Overview
This block manages the integration with Google Drive, exporting the destination document to HTML, injecting the newly compiled meeting minutes safely at the top of the body payload, updating the remote document, and rendering a success interface to the user.

##### Nodes Involved
- `Export notes Doc as HTML from Drive`
- `Prepend meeting to notes Doc HTML`
- `Upload updated notes Doc to Drive`
- `Show notes ready page`

##### Node Details

###### Export notes Doc as HTML from Drive
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (Google Drive API Integration)
- **Configuration Choices:** Calls the Google Drive REST APIv3 files export endpoint to convert the specified target document into `text/html`.
- **Key Expressions or Variables:** Dynamically interpolates the document ID via `={{ $('Set configuration').first().json.notesDocId }}`.
- **Input and Output Connections:** Input from `Validate Claude notes`; output to `Prepend meeting to notes Doc HTML`.
- **Version-Specific Requirements:** Version 4.2+.
- **Edge Cases or Potential Failure Types:** Permission errors, invalid document IDs, or OAuth2 token expirations.

###### Prepend meeting to notes Doc HTML
- **Type and Technical Role:** `n8n-nodes-base.code` (HTML Manipulation)
- **Configuration Choices:** Constructs a structured HTML string containing the meeting title, metadata badges, agenda lists, topic breakdown bullets, and a fully formatted action item table. Inserts this segment immediately after the `<body>` tag of the existing exported document, appending a horizontal rule separator (`<hr>`) if prior historical content exists.
- **Key Expressions or Variables:** Reads data from `Prepare transcript and meeting details`, `Validate Claude notes`, and the Drive export payload.
- **Input and Output Connections:** Input from `Export notes Doc as HTML from Drive`; output to `Upload updated notes Doc to Drive`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases or Potential Failure Types:** Malformed HTML structures or non-HTML exports trigger explicit exceptions protecting the target document from corruption.

###### Upload updated notes Doc to Drive
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (Google Drive API Integration)
- **Configuration Choices:** Sends an HTTP `PATCH` request to update the file content on Google Drive using raw `text/html; charset=UTF-8` payload with `uploadType=media` and shared drives support enabled (`supportsAllDrives=true`).
- **Key Expressions or Variables:** Body populated via `={{ $json.html }}`; file ID interpolated from configuration.
- **Input and Output Connections:** Input from `Prepend meeting to notes Doc HTML`; output to `Show notes ready page`.
- **Version-Specific Requirements:** Version 4.2+.
- **Edge Cases or Potential Failure Types:** Network interruptions or quota limits enforced by Google Drive API.

###### Show notes ready page
- **Type and Technical Role:** `n8n-nodes-base.form` (Form Completion Response)
- **Configuration Choices:** Form operation set to `completion` responding with custom HTML text displaying summary metrics (total action items, unassigned counts) and an outbound link to the modified Google Document.
- **Key Expressions or Variables:** Interpolates metadata properties from `$('Prepend meeting to notes Doc HTML')` and `$json.webViewLink`.
- **Input and Output Connections:** Input from `Upload updated notes Doc to Drive`; terminal node.
- **Version-Specific Requirements:** Version 1.
- **Edge Cases or Potential Failure Types:** None significant assuming upstream variables resolve correctly.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Upload transcript via form | `n8n-nodes-base.formTrigger` | Ingests plain-text transcript files and optional meeting metadata via web form. | None | Set configuration | Open the form URL (Test URL while building, Production URL once the workflow is active) and upload one .txt transcript. Date and title are optional. |
| Set configuration | `n8n-nodes-base.set` | Stores global environment configurations (Doc ID, team names, timezone, model). | Upload transcript via form | Prepare transcript and meeting details | Every value you need to change lives here: the notes Doc ID, the attendees, the meeting days, the glossary and the model. The uploaded file passes through untouched. |
| Prepare transcript and meeting details | `n8n-nodes-base.code` | Validates file extensions/sizes, cleans text, and derives calendar dates and titles. | Set configuration | Build Claude request | Rejects files that are not .txt, are empty or are too large. Cleans the text and works out the meeting date (form, then a YYYY-MM-DD filename prefix, then the upload date), the title and the next meeting date. |
| Build Claude request | `n8n-nodes-base.code` | Constructs system prompts and packages transcripts into OpenRouter API payloads. | Prepare transcript and meeting details | Ask Claude for notes via OpenRouter | Writes the note-taking instructions and puts the transcript in the request body. The rules here match the checks in Validate Claude notes. |
| Ask Claude for notes via OpenRouter | `n8n-nodes-base.httpRequest` | Dispatches chat completion queries to Claude via OpenRouter with automatic retries. | Build Claude request | Validate Claude notes | Header Auth credential with header name Authorization and value 'Bearer YOUR_OPENROUTER_KEY'. Retries up to 3 times if OpenRouter is briefly unavailable. |
| Validate Claude notes | `n8n-nodes-base.code` | Validates LLM-generated JSON schema compliance, word limits, and entities. | Ask Claude for notes via OpenRouter | Export notes Doc as HTML from Drive | Checks the summary word limit, that agenda and topics match, task length, owners and due dates. If anything fails it stops with a list of every problem and the Doc is not touched. |
| Export notes Doc as HTML from Drive | `n8n-nodes-base.httpRequest` | Exports the designated Google Doc as an HTML file from Google Drive. | Validate Claude notes | Prepend meeting to notes Doc HTML | Reads the current notes Doc as HTML so earlier meetings can be kept below the new one. |
| Prepend meeting to notes Doc HTML | `n8n-nodes-base.code` | Generates formatted HTML meeting notes and prepends them to historical data. | Export notes Doc as HTML from Drive | Upload updated notes Doc to Drive | Builds the title, date, attendees, source, summary and action item table, then inserts them at the top of the Doc, above a line separating earlier meetings. |
| Upload updated notes Doc to Drive | `n8n-nodes-base.httpRequest` | Replaces the remote Google Doc contents via a PATCH request with HTML media upload. | Prepend meeting to notes Doc HTML | Show notes ready page | Replaces the Doc's content with the updated HTML. Drive converts it into real Google Docs headings, lists and tables. |
| Show notes ready page | `n8n-nodes-base.form` | Renders a form completion view with operational metrics and a direct document link. | Upload updated notes Doc to Drive | None | The page the person who uploaded the transcript sees at the end, with a link to the Doc. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Form Trigger Node:**
   - Add a **Form Trigger** node named `Upload transcript via form`.
   - Configure options: Set button label to `Create notes`, form title to `Upload a meeting transcript`.
   - Define form fields: 
     - Field 1: Type `file`, Label `Transcript`, set `requiredField` to `true`, accept file types `.txt`, multiple files disabled.
     - Field 2: Type `date`, Label `Meeting date`.
     - Field 3: Type `text`, Label `Meeting title`, placeholder `Leave blank to use the file name`.
   - Set Response Mode to `lastNode`.

2. **Create Configuration Node:**
   - Add a **Set** node named `Set configuration`.
   - Add string/number assignments:
     - `notesDocId` (string): `YOUR_GOOGLE_DOC_ID`
     - `organisationName` (string): `Your Organisation`
     - `teamMembers` (string): `Person One, Person Two`
     - `meetingDays` (string): `Monday, Wednesday`
     - `timezone` (string): `Europe/London`
     - `glossary` (string): `Acme = the organisation's short name; Zoho = Zoho Projects`
     - `aiModel` (string): `anthropic/claude-sonnet-5.5`
     - `maxSummaryWords` (number): `500`
     - `maxTranscriptChars` (number): `200000`
   - **Connection:** Connect `Upload transcript via form` to `Set configuration`.

3. **Create Transcript Preparation Code Node:**
   - Add a **Code** node named `Prepare transcript and meeting details`.
   - Paste the JavaScript implementation provided in the JSON configuration for filtering binary files, sanitizing text, extracting dates, and compiling attendee lists.
   - **Connection:** Connect `Set configuration` to this node.

4. **Create AI Prompt Builder Code Node:**
   - Add a **Code** node named `Build Claude request`.
   - Paste the JavaScript implementation responsible for compiling the system prompt, setting parameters (`temperature: 0.2`, `max_tokens: 8000`, `response_format: json_object`), and structuring the OpenRouter API request body.
   - **Connection:** Connect `Prepare transcript and meeting details` to this node.

5. **Create OpenRouter HTTP Request Node:**
   - Add an **HTTP Request** node named `Ask Claude for notes via OpenRouter`.
   - Set Method to `POST`, URL to `https://openrouter.ai/api/v1/chat/completions`.
   - Configure Body: Specify JSON body using expression `={{ $json.requestBody }}`.
   - Configure Options: Set timeout to `300000ms`, enable `retryOnFail` with 3 max tries and 5000ms wait interval.
   - Configure Credentials: Add and select a **Header Auth** credential with header name `Authorization` and value `Bearer <your OpenRouter API key>`.
   - **Connection:** Connect `Build Claude request` to this node.

6. **Create Notes Validation Code Node:**
   - Add a **Code** node named `Validate Claude notes`.
   - Paste the JavaScript validation logic ensuring schema compliance, word bounds, team membership verification, and date validity.
   - **Connection:** Connect `Ask Claude for notes via OpenRouter` to this node.

7. **Create Google Drive Export HTTP Request Node:**
   - Add an **HTTP Request** node named `Export notes Doc as HTML from Drive`.
   - Set Method to `GET`, URL expression: `https://www.googleapis.com/drive/v3/files/{{ $('Set configuration').first().json.notesDocId }}/export`.
   - Configure Query Parameters: Add parameter `mimeType` with value `text/html`.
   - Configure Options: Set response format to text with output property name `data`.
   - Configure Credentials: Add and select a **Google Drive OAuth2 API** credential.
   - **Connection:** Connect `Validate Claude notes` to this node.

8. **Create HTML Prepend Code Node:**
   - Add a **Code** node named `Prepend meeting to notes Doc HTML`.
   - Paste the JavaScript code responsible for converting structured JSON notes into semantic HTML elements and prepending them to the existing document HTML.
   - **Connection:** Connect `Export notes Doc as HTML from Drive` to this node.

9. **Create Google Drive Upload HTTP Request Node:**
   - Add an **HTTP Request** node named `Upload updated notes Doc to Drive`.
   - Set Method to `PATCH`, URL expression: `https://www.googleapis.com/upload/drive/v3/files/{{ $('Set configuration').first().json.notesDocId }}`.
   - Configure Body: Set content type to Raw (`text/html; charset=UTF-8`), set body expression to `={{ $json.html }}`.
   - Configure Query Parameters: Add `uploadType` = `media`, `fields` = `id,name,webViewLink`, `supportsAllDrives` = `true`.
   - Configure Credentials: Use the same **Google Drive OAuth2 API** credential.
   - **Connection:** Connect `Prepend meeting to notes Doc HTML` to this node.

10. **Create Form Completion Response Node:**
    - Add a **Form** node named `Show notes ready page`.
    - Set operation to `completion`, response type to `showText`.
    - Provide the completion HTML template containing interpolated variables from `Prepend meeting to notes Doc HTML` and `$json.webViewLink`.
    - **Connection:** Connect `Upload updated notes Doc to Drive` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure the target Google Document is shared or accessible via the authenticated Google Drive OAuth2 account. | Google Drive Integration Setup |
| OpenRouter credits and API limits apply when processing large transcript bodies. Verify model slugs regularly against OpenRouter documentation. | OpenRouter API Configuration |