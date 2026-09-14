Review Discord community applications with Tally, OpenAI and Discord

https://n8nworkflows.xyz/workflows/review-discord-community-applications-with-tally--openai-and-discord-17649


# Review Discord community applications with Tally, OpenAI and Discord

### 1. Workflow Overview

This workflow automates the collection, validation, AI-powered screening, routing, and auditing of Discord community membership applications. It removes manual overhead by processing form submissions through an AI evaluation model and delivering structured notifications to appropriate Discord channels while simultaneously logging results into a Google Sheets document.

The workflow logic is grouped into the following functional blocks:

- **1.1 Input Reception & Standardization:** Listens for incoming Tally form submissions, extracts the form data, and normalizes specific question answers into structured variables.
- **1.2 Validation Layer:** Inspects the normalized applicant payload to ensure that core required fields (such as email addresses and application content) are present before proceeding.
- **1.3 AI Review & Classification:** Passes the applicant payload to an OpenAI chat model configured with strict criteria to categorize submissions into “Approved”, “Needs Review”, or “Rejected” while generating reasoning, risk assessments, confidence scores, and summaries.
- **1.4 Decision Routing:** Uses conditional routing based on the AI’s status output to direct the application down the correct notification path.
- **1.5 Notification & Auditing:** Delivers richly formatted status updates to designated Discord channels based on the classification result, or triggers an error notification if critical data or processing fails, and persists all processed logs into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Standardization

**Overview:**  
This block triggers execution when a new form submission is received from Tally, flattening and mapping raw question payloads into clean, accessible data fields for subsequent nodes.

**Nodes Involved:**
- Receive Tally Submission
- Extract Applicant Information

**Node Details:**

- **Receive Tally Submission**
  - **Type and Technical Role:** `n8n-nodes-tallyforms.tallyTrigger` (Trigger Node)
  - **Configuration Choices:** Listens for webhook events emitted by Tally form submissions.
  - **Key Expressions or Variables:** None (Trigger event).
  - **Input and Output Connections:** Input: None (Webhook webhookId: `80817cf4-e9a9-4b7a-b91e-0c7e46d2c969`). Output: Connects to *Extract Applicant Information*.
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases or Potential Failure Types:** Webhook signature or configuration mismatch, network timeout from Tally.
  - **Sub-workflow Reference:** None.

- **Extract Applicant Information**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation Node)
  - **Configuration Choices:** Assigns specific Tally question IDs to semantic variable names (`name`, `email`, `content`, `grade`, `experience`) alongside metadata fields (`id`, `formId`, `formName`, `respondentId`, `createdAt`, `submissionPdfUrl`, `submissionPreviewUrl`).
  - **Key Expressions or Variables:** 
    - `name`: `={{ $json.question_888W7z.value || "" }}`
    - `email`: `={{ $json.question_zJJOPM.value || "" }}`
    - `content`: `={{ $json.question_YYYgNW.value || "" }}`
    - `grade`: `={{ $json.question_5vvQWZ.value || "" }}`
    - `experience`: `={{ $json.question_djjpBd.value || "" }}`
  - **Input and Output Connections:** Input: *Receive Tally Submission*. Output: Connects to *Validate Required Fields*.
  - **Version-Specific Requirements:** Type version 3.4.
  - **Edge Cases or Potential Failure Types:** Missing question keys in unexpected Tally form layouts will fall back to empty strings.
  - **Sub-workflow Reference:** None.

---

#### 2.2 Validation Layer

**Overview:**  
This block acts as a gatekeeper, verifying that essential identity and application data exist before incurring AI evaluation costs or triggering downstream actions.

**Nodes Involved:**
- Validate Required Fields

**Node Details:**

- **Validate Required Fields**
  - **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control Node)
  - **Configuration Choices:** Evaluates conditions using strict type validation with an AND combinator.
  - **Key Expressions or Variables:** 
    - Left value 1: `={{ $json.email }}`, Operator: `notEmpty`
    - Left value 2: `={{ $json.content }}`, Operator: `notEmpty`
  - **Input and Output Connections:** Input: *Extract Applicant Information*. Output (True/Valid): Connects to *Review Application with AI*. Output (False/Invalid): Connects to *Notify Processing Error*.
  - **Version-Specific Requirements:** Type version 2.3.
  - **Edge Cases or Potential Failure Types:** Evaluation failure if field properties are entirely undefined instead of empty strings.
  - **Sub-workflow Reference:** None.

---

#### 2.3 AI Review & Classification

**Overview:**  
This block submits standardized applicant data to an OpenAI language model using a dedicated system prompt and structural schema extractor to generate a standardized decision object.

**Nodes Involved:**
- Review Application with AI
- OpenAI Chat Model

**Node Details:**

- **Review Application with AI**
  - **Type and Technical Role:** `@n8n/n8n-nodes-langchain.informationExtractor` (AI / LangChain Node)
  - **Configuration Choices:** Uses a multi-line text template binding form information, applicant fields, and a comprehensive system prompt enforcing JSON-only output rules. Attributes extracted: `status`, `reason`, `confidence`, `risk`, and `summary`.
  - **Key Expressions or Variables:** Binds dynamic tokens for `formName`, `id`, `createdAt`, `submissionPdfUrl`, `name`, `email`, `grade`, `experience`, and `content`.
  - **Input and Output Connections:** Input (Main): *Validate Required Fields*. Input (AI Model): Connected from *OpenAI Chat Model*. Output: Connects to *Route Review Result1*.
  - **Version-Specific Requirements:** Type version 1.2.
  - **Edge Cases or Potential Failure Types:** API rate limits, model hallucinations failing to adhere to strict schema outputs, or invalid API credentials.
  - **Sub-workflow Reference:** None.

- **OpenAI Chat Model**
  - **Type and Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (LangChain Sub-Node)
  - **Configuration Choices:** Configures the base chat model parameters for OpenAI integration.
  - **Key Expressions or Variables:** None.
  - **Input and Output Connections:** Input: None. Output: Connects to the `ai_languageModel` input of *Review Application with AI*.
  - **Version-Specific Requirements:** Type version 1.3.
  - **Edge Cases or Potential Failure Types:** Authentication failures or quota exhaustion with OpenAI.
  - **Sub-workflow Reference:** None.

---

#### 2.4 Decision Routing

**Overview:**  
This block analyzes the structured status string returned by the AI review node and splits workflow execution into separate branches accordingly.

**Nodes Involved:**
- Route Review Result1

**Node Details:**

- **Route Review Result1**
  - **Type and Technical Role:** `n8n-nodes-base.switch` (Flow Control Node)
  - **Configuration Choices:** Evaluates the field `={{ $json.output.status }}` against four routes: `Approved`, `Needs Review`, `Rejected`, and a fallback `Unknown` route.
  - **Key Expressions or Variables:** `={{ $json.output.status }}`
  - **Input and Output Connections:** Input: *Review Application with AI*. Output branches:
    - Output 0 (Approved) -> *Notify Approved Application*
    - Output 1 (Needs Review) -> *Notify Manual Review*
    - Output 2 (Rejected) -> *Notify Rejected Application*
    - Output 3 (Fallback/Unknown) -> *Notify Processing Error*
  - **Version-Specific Requirements:** Type version 3.4.
  - **Edge Cases or Potential Failure Types:** Unexpected text case or unhandled status values falling through to the fallback output.
  - **Sub-workflow Reference:** None.

---

#### 2.5 Notification & Auditing

**Overview:**  
This block formats application results into rich Discord messages dispatched to corresponding channels based on decisions, while logging completed reviews to a centralized Google Sheets document.

**Nodes Involved:**
- Notify Approved Application
- Notify Manual Review
- Notify Rejected Application
- Notify Processing Error
- Save Review Log to Google Sheets

**Node Details:**

- **Notify Approved Application**
  - **Type and Technical Role:** `n8n-nodes-base.discord` (Action Node)
  - **Configuration Choices:** Posts a markdown-formatted notification highlighting successful approval.
  - **Key Expressions or Variables:** Binds properties using parent node references like `{{ $('Extract Applicant Information').item.json.id }}` and `{{ $('Review Application with AI').item.json.output.summary }}`.
  - **Input and Output Connections:** Input: *Route Review Result1* (Approved branch). Output: Connects to *Save Review Log to Google Sheets*.
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases or Potential Failure Types:** Discord webhook/bot rate limits, invalid channel IDs, or missing bot permissions.
  - **Sub-workflow Reference:** None.

- **Notify Manual Review**
  - **Type and Technical Role:** `n8n-nodes-base.discord` (Action Node)
  - **Configuration Choices:** Posts an alert message for applications requiring human moderation.
  - **Key Expressions or Variables:** Binds applicant data and AI evaluation reasons using node reference expressions.
  - **Input and Output Connections:** Input: *Route Review Result1* (Needs Review branch). Output: Connects to *Save Review Log to Google Sheets*.
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases or Potential Failure Types:** Bot permission errors or invalid target channel configuration.
  - **Sub-workflow Reference:** None.

- **Notify Rejected Application**
  - **Type and Technical Role:** `n8n-nodes-base.discord` (Action Node)
  - **Configuration Choices:** Posts rejection notices to the designated moderation or log channel.
  - **Key Expressions or Variables:** Binds applicant and AI assessment values using node references.
  - **Input and Output Connections:** Input: *Route Review Result1* (Rejected branch). Output: Connects to *Save Review Log to Google Sheets*.
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases or Potential Failure Types:** Discord API connectivity issues.
  - **Sub-workflow Reference:** None.

- **Notify Processing Error**
  - **Type and Technical Role:** `n8n-nodes-base.discord` (Action Node)
  - **Configuration Choices:** Posts a localized error alert (`【エラー / 要対応】`) when validation fails or unexpected states occur.
  - **Key Expressions or Variables:** `{{ $('Extract Applicant Information').item.json.name }}`, `{{ $('Extract Applicant Information').item.json.email }}`, `{{ $('Extract Applicant Information').item.json.content }}`
  - **Input and Output Connections:** Input: *Validate Required Fields* (False branch) or *Route Review Result1* (Fallback branch). Output: None (Terminal node).
  - **Version-Specific Requirements:** Type version 2.
  - **Edge Cases or Potential Failure Types:** Incorrect channel ID configuration.
  - **Sub-workflow Reference:** None.

- **Save Review Log to Google Sheets**
  - **Type and Technical Role:** `n8n-nodes-base.googleSheets` (Data Integration Node)
  - **Configuration Choices:** Appends rows containing application details and review results to a target spreadsheet document.
  - **Key Expressions or Variables:** Uses configured document and sheet identifiers.
  - **Input and Output Connections:** Input: Connected from *Notify Approved Application*, *Notify Manual Review*, and *Notify Rejected Application*. Output: None (Terminal node).
  - **Version-Specific Requirements:** Type version 4.7.
  - **Edge Cases or Potential Failure Types:** Google Sheets quota limitations, revoked OAuth2 tokens, or mismatched sheet column ranges.
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow documentation | None | None | AI-powered Discord Community Application Review<br><br>Overview<br><br>This workflow automatically reviews Discord community applications submitted through a Tally form using AI.<br>... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Step 1 documentation | None | None | Step 1<br>Receive Form Submission<br><br>The workflow starts when a user submits the Tally application form. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Step 2 documentation | None | None | Step 2<br>Extract Required Fields<br><br>This node organizes all submitted form data into a consistent structure so that the following nodes can easily access the applicant information. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Step 5 documentation | None | None | Step 5<br>Decision Routing<br><br>Routes the application according to the AI decision.<br><br>Possible outputs:<br><br>・Approved<br>・Needs Review<br>・Rejected<br>・Unknown/Error |
| Sticky Note4 | n8n-nodes-base.stickyNote | Step 4 documentation | None | None | Step 4<br>AI Application Review<br><br>The application is analyzed using OpenAI.<br><br>The AI returns:<br><br>・Status<br>・Summary<br>・Reason<br>・Confidence<br>・Risk Level |
| Sticky Note5 | n8n-nodes-base.stickyNote | Step 3 documentation | None | None | Step 3<br>Validate Required Information<br><br>Checks whether required fields such as name, email address and application content are present.<br><br>If required information is missing, the workflow immediately sends an error notification. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Step 6 documentation | None | None | Step 6<br>Discord Notification<br><br>Sends the review result to the corresponding Discord channel.<br><br>Each message includes:<br><br>・Applicant<br>・AI Summary<br>・Review Reason<br>・Confidence Score<br>・Risk Level |
| Sticky Note7 | n8n-nodes-base.stickyNote | Step 7 documentation | None | None | Step 7<br>Save Review Log<br><br>Stores every processed application inside Google Sheets.<br><br>This creates a searchable review history and supports future audits. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Error handling documentation | None | None | Error Handling<br><br>Applications with missing information or unexpected AI responses are redirected to a dedicated Discord error channel for manual review.<br><br>This prevents invalid applications from entering the normal review process. |
| Receive Tally Submission | n8n-nodes-tallyforms.tallyTrigger | Trigger workflow on form submission | None | Extract Applicant Information | |
| Extract Applicant Information | n8n-nodes-base.set | Standardize form payload fields | Receive Tally Submission | Validate Required Fields | |
| Validate Required Fields | n8n-nodes-base.if | Verify email and content presence | Extract Applicant Information | Review Application with AI, Notify Processing Error | |
| Review Application with AI | @n8n/n8n-nodes-langchain.informationExtractor | AI extraction of applicant evaluation | Validate Required Fields, OpenAI Chat Model | Route Review Result1 | |
| Route Review Result1 | n8n-nodes-base.switch | Route by AI status decision | Review Application with AI | Notify Approved Application, Notify Manual Review, Notify Rejected Application, Notify Processing Error | |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide LLM engine for AI review | None | Review Application with AI | |
| Notify Approved Application | n8n-nodes-base.discord | Post approved application notice | Route Review Result1 | Save Review Log to Google Sheets | |
| Notify Manual Review | n8n-nodes-base.discord | Post manual review notice | Route Review Result1 | Save Review Log to Google Sheets | |
| Notify Rejected Application | n8n-nodes-base.discord | Post rejection notice | Route Review Result1 | Save Review Log to Google Sheets | |
| Notify Processing Error | n8n-nodes-base.discord | Post error or fallback alert | Validate Required Fields, Route Review Result1 | None | |
| Save Review Log to Google Sheets | n8n-nodes-base.googleSheets | Append audit log entry | Notify Approved Application, Notify Manual Review, Notify Rejected Application | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Tally Forms Trigger** (`n8n-nodes-tallyforms.tallyTrigger`) named `Receive Tally Submission`. Connect your Tally credential.
2. **Add Data Extraction:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Extract Applicant Information`. Connect it to the output of `Receive Tally Submission`.
   - Configure assignments to map incoming Tally question IDs to standard fields (`name`, `email`, `content`, `grade`, `experience`) and metadata fields (`id`, `formId`, `formName`, `respondentId`, `createdAt`, `submissionPdfUrl`, `submissionPreviewUrl`).
3. **Add Validation Control Flow:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Validate Required Fields`. Connect its input to `Extract Applicant Information`.
   - Set conditions to verify that `email` and `content` are not empty using strict type validation.
4. **Configure AI Evaluation Components:**
   - Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Select your desired chat model and configure your OpenAI API credentials.
   - Add an **Information Extractor** node (`@n8n/n8n-nodes-langchain.informationExtractor`) named `Review Application with AI`.
   - Connect the OpenAI Chat Model output to the `ai_languageModel` input of the Information Extractor.
   - Connect the *true* output of `Validate Required Fields` to the main input of `Review Application with AI`.
   - Populate the text template with applicant fields and configure the system prompt and required attributes (`status`, `reason`, `confidence`, `risk`, `summary`) as specified in the analysis.
5. **Add Decision Routing:**
   - Add a **Switch** node (`n8n-nodes-base.switch`) named `Route Review Result1`. Connect its input to `Review Application with AI`.
   - Configure rules targeting `={{ $json.output.status }}` with three branches: `Approved`, `Needs Review`, and `Rejected`, setting a fallback route for `Unknown`.
6. **Set Up Discord Notifications:**
   - Add four **Discord** nodes (`n8n-nodes-base.discord`) named `Notify Approved Application`, `Notify Manual Review`, `Notify Rejected Application`, and `Notify Processing Error`. Connect your Discord Bot credentials.
   - Connect Switch output 0 (`Approved`) to `Notify Approved Application`.
   - Connect Switch output 1 (`Needs Review`) to `Notify Manual Review`.
   - Connect Switch output 2 (`Rejected`) to `Notify Rejected Application`.
   - Connect Switch fallback output (`Unknown`) and the *false* output of `Validate Required Fields` to `Notify Processing Error`.
   - Populate the message content fields for each node referencing the appropriate expression paths for applicant details and AI evaluations.
7. **Configure Google Sheets Auditing:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Save Review Log to Google Sheets`. Connect your Google Sheets credential.
   - Set the operation to `append` and specify your target document and sheet IDs.
   - Connect the output of `Notify Approved Application`, `Notify Manual Review`, and `Notify Rejected Application` into the input of `Save Review Log to Google Sheets`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| This template is designed for communities and organizations that receive membership applications and want to reduce manual review work. | Target Use Case & Audience |
| Requirements include Tally Forms, OpenAI API, Discord Bot, and Google Sheets integrations. | System Prerequisites |