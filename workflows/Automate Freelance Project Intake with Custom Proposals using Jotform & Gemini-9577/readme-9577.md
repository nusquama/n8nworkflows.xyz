Automate Freelance Project Intake with Custom Proposals using Jotform & Gemini

https://n8nworkflows.xyz/workflows/automate-freelance-project-intake-with-custom-proposals-using-jotform---gemini-9577


# Automate Freelance Project Intake with Custom Proposals using Jotform & Gemini

### 1. Workflow Overview

This workflow automates the intake of freelance project inquiries submitted via a JotForm form. Its primary purpose is to analyze incoming client project requests, generate customized freelance proposals using AI, log the client and proposal details into a Google Sheet CRM, and then send a tailored proposal email to the client automatically.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Captures new project inquiries submitted through a JotForm form.
- **1.2 AI Processing:** Uses an AI agent with LangChain and Google Gemini integration to analyze the inquiry, reference an internal Google Document containing services/pricing, and generate a structured JSON proposal and email.
- **1.3 Data Logging:** Logs or updates the client’s project details and AI-generated proposal in a Google Sheet CRM.
- **1.4 Decision & Email Dispatch:** Checks the proposal’s alignment status and conditionally sends the customized proposal email to the client via Gmail.
- **1.5 Informational Overlay:** A Sticky Note explains the workflow’s overall logic for maintainers or users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new submissions on a specific JotForm and triggers the workflow with the submitted data.

- **Nodes Involved:**  
  - JotForm Trigger

- **Node Details:**

  - **JotForm Trigger**  
    - *Type & Role:* Trigger node for JotForm form submissions, starts workflow on new form entry.  
    - *Configuration:* Connected to form ID `252853173448059` with authenticated JotForm API credentials.  
    - *Key Expressions:* None (direct trigger).  
    - *Connections:* Outputs data to "AI Agent".  
    - *Version:* 1 (basic trigger functionality).  
    - *Edge Cases:* Possible failures include webhook setup errors, API authentication issues, or missing form data.  
    - *Sub-Workflow:* None.

#### 2.2 AI Processing

- **Overview:**  
  This block processes the client’s inquiry with an AI agent that fetches internal service/pricing data from a Google Doc, analyzes the request, and produces a structured JSON proposal including a project assessment and a personalized email.

- **Nodes Involved:**  
  - My Freelance Document  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - AI Agent

- **Node Details:**

  - **My Freelance Document**  
    - *Type & Role:* Google Docs Tool node to retrieve the latest internal freelancing services and pricing document.  
    - *Configuration:* Retrieves contents from a specified Google Doc URL using a Google Service Account credential.  
    - *Key Expressions:* None; static document fetch.  
    - *Connections:* Feeds data as a tool input to "AI Agent".  
    - *Version:* 2.  
    - *Edge Cases:* Document inaccessible or credential failures trigger AI agent to output an error message and halt.  
    - *Sub-Workflow:* None.

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model node providing Google Gemini (PaLM) AI capabilities.  
    - *Configuration:* Uses Google PaLM API credentials to enable AI language model features for the "AI Agent".  
    - *Key Expressions:* None; used as an AI language model reference.  
    - *Connections:* Connected as AI language model input to "AI Agent".  
    - *Version:* 1.  
    - *Edge Cases:* API quota limits, authentication errors, or network timeouts may cause failures.  
    - *Sub-Workflow:* None.

  - **Structured Output Parser**  
    - *Type & Role:* LangChain output parser node to enforce the AI agent’s output structure as a valid JSON object matching a strict schema.  
    - *Configuration:* Uses a JSON schema example defining required fields such as "project_type", "confidence", "summary", etc.  
    - *Key Expressions:* None; validates AI output format.  
    - *Connections:* Connected as AI output parser input to "AI Agent".  
    - *Version:* 1.3.  
    - *Edge Cases:* Parsing fails if AI output is malformed or non-JSON.  
    - *Sub-Workflow:* None.

  - **AI Agent**  
    - *Type & Role:* LangChain Agent node that orchestrates AI processing by combining input from the JotForm submission, the internal Google Doc tool, the Gemini language model, and the structured output parser.  
    - *Configuration:*  
      - Prompt defines role as “AI Freelance Proposal Generator” with detailed instructions and output specification.  
      - Enforces calling the Google Doc tool first, then analyzing client data, and generating a JSON proposal.  
      - Key variables extracted from the JotForm JSON (e.g., client name, email, budget, project description).  
    - *Key Expressions:* Uses advanced templating to inject form responses into the prompt.  
    - *Connections:* Receives input from "JotForm Trigger"; uses AI tool ("My Freelance Document"), AI language model ("Google Gemini Chat Model"), and AI output parser ("Structured Output Parser"). Outputs proposal JSON to "Append or update row in sheet".  
    - *Version:* 2.2.  
    - *Edge Cases:*  
      - Failure to access Google Doc results in immediate error output.  
      - AI generation failures or timeouts can disrupt proposal creation.  
      - Parsing errors if AI output is invalid.  
    - *Sub-Workflow:* None.

#### 2.3 Data Logging

- **Overview:**  
  This block records or updates the client’s inquiry and AI-generated proposal details in a Google Sheet acting as a CRM.

- **Nodes Involved:**  
  - Append or update row in sheet

- **Node Details:**

  - **Append or update row in sheet**  
    - *Type & Role:* Google Sheets node to append a new row or update an existing one based on the client's email as a unique key.  
    - *Configuration:*  
      - Spreadsheet ID: `1gg-xBRO93bPg41PDqOx_V_OtokWljHAf0IgIWsWgUls`  
      - Sheet Name: `Sheet1` (gid=0)  
      - Columns mapped include client full name, email, phone, project summary, proposal confidence, project type, budget, email subject, and AI-generated email body.  
      - Matching column: "Email" to update existing records if any.  
    - *Key Expressions:* Pulls client data from JotForm and proposal data from AI Agent JSON output.  
    - *Connections:* Outputs to "If" node for conditional processing.  
    - *Version:* 4.7.  
    - *Edge Cases:*  
      - Authentication errors with Google Sheets OAuth2.  
      - Sheet or columns missing or renamed causing mapping failures.  
      - Network or quota issues.  
    - *Sub-Workflow:* None.

#### 2.4 Decision & Email Dispatch

- **Overview:**  
  Evaluates the AI-generated project type to determine if the proposal should be sent. If aligned or partially aligned, sends the proposal email via Gmail.

- **Nodes Involved:**  
  - If  
  - Send a message

- **Node Details:**

  - **If**  
    - *Type & Role:* Conditional node to check if the project type is "aligned" or "partially_aligned".  
    - *Configuration:*  
      - Case-insensitive string check on `project_type` field from AI output JSON.  
      - Conditions: `project_type == "aligned"` OR `project_type == "partially_aligned"`.  
    - *Key Expressions:* `={{ $json.project_type }}`  
    - *Connections:* On true, outputs to "Send a message". On false, workflow ends (no email sent).  
    - *Version:* 2.2.  
    - *Edge Cases:* Missing or malformed `project_type` field may cause false negatives.  
    - *Sub-Workflow:* None.

  - **Send a message**  
    - *Type & Role:* Gmail node to send the AI-generated proposal email to the client.  
    - *Configuration:*  
      - Recipient dynamically set to client email from Google Sheet data.  
      - Email subject and HTML body taken from AI output fields.  
      - Attribution disabled.  
      - Uses Gmail OAuth2 credentials.  
    - *Key Expressions:*  
      - `sendTo = {{ $json.Email }}`  
      - `subject = {{ $json['Email Subject'] }}`  
      - `message = {{ $json['AI generated Email body'] }}`  
    - *Connections:* None (final node).  
    - *Version:* 2.1.  
    - *Edge Cases:*  
      - Gmail API quota or authentication errors.  
      - Invalid or missing email addresses.  
    - *Sub-Workflow:* None.

#### 2.5 Informational Overlay

- **Overview:**  
  A Sticky Note node providing a textual summary of the workflow’s operation for users or maintainers.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - *Type & Role:* Documentation node within n8n for visual explanation.  
    - *Configuration:* Contains a multi-line markdown note explaining the workflow steps and logic.  
    - *Connections:* None.  
    - *Version:* 1.  
    - *Edge Cases:* None.  
    - *Sub-Workflow:* None.

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                                | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                       |
|---------------------------|-------------------------------------|-----------------------------------------------|----------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| JotForm Trigger           | n8n-nodes-base.jotFormTrigger       | Receive project inquiry submissions           | (trigger)            | AI Agent                 | This is how this works: <br> * Client submits inquiry via JotForm, triggering automation.       |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent      | Analyze inquiry, generate proposal & email    | JotForm Trigger      | Append or update row in sheet | * AI references services doc, generates structured JSON proposal and email.                  |
| My Freelance Document     | n8n-nodes-base.googleDocsTool       | Retrieve internal service/pricing document     | (used by AI Agent)   | AI Agent (tool input)    | * Internal Google Doc is the single source of truth for proposals.                              |
| Google Gemini Chat Model  | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provide AI language model capability           | (used by AI Agent)   | AI Agent (language model input) | * AI language model enabling proposal generation.                                            |
| Structured Output Parser  | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce valid JSON output format                 | (used by AI Agent)   | AI Agent (output parser input)   | * Validates AI output matches expected JSON schema.                                          |
| Append or update row in sheet | n8n-nodes-base.googleSheets          | Log/update client and proposal data in CRM     | AI Agent             | If                       | * Logs client inquiry and AI proposal in Google Sheets CRM.                                    |
| If                        | n8n-nodes-base.if                   | Check project alignment to decide on email send | Append or update row in sheet | Send a message       | * Sends email only if project is aligned or partially aligned.                                 |
| Send a message             | n8n-nodes-base.gmail                | Email the proposal to the client                | If                   | (end)                    | * Sends personalized proposal email via Gmail.                                                |
| Sticky Note                | n8n-nodes-base.stickyNote           | Workflow explanation for users                  | None                 | None                     | See first row sticky note content.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Type: `JotForm Trigger`  
   - Set `Form` field to the form ID `252853173448059`  
   - Set credentials with your JotForm API account  
   - Position it as the workflow start trigger

2. **Add AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configure the prompt with the detailed instructions specifying:  
     - Role as AI Freelance Proposal Generator  
     - Require calling internal "My Freelance Document" tool first  
     - Input client form fields from the trigger (map fields such as "Explain Your Requirement in depth", client name, email, phone, company)  
     - Define output as a single JSON object with fields: project_type, confidence, summary, budget, proposal_quality, next_action, email_subject, email_template  
   - Connect input from "JotForm Trigger"  
   - Set up AI tools: connect to "My Freelance Document" (tool), "Google Gemini Chat Model" (AI language model), and "Structured Output Parser" (output parser) nodes

3. **Create My Freelance Document Node**  
   - Type: `Google Docs Tool`  
   - Set operation to `get` to fetch document contents  
   - Provide the Google Doc URL containing your freelance service catalog and pricing  
   - Authenticate with a Google Service Account credential  
   - Connect output as AI tool input to the AI Agent

4. **Create Google Gemini Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Use Google PaLM API credentials  
   - Connect output as AI language model input to AI Agent

5. **Create Structured Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Provide a JSON schema example matching the expected AI JSON output structure  
   - Connect output as AI output parser input to AI Agent

6. **Add Google Sheets Node "Append or update row in sheet"**  
   - Type: `Google Sheets`  
   - Set operation to `appendOrUpdate`  
   - Set spreadsheet ID to your Google Sheet CRM ID (e.g., `1gg-xBRO93bPg41PDqOx_V_OtokWljHAf0IgIWsWgUls`)  
   - Set Sheet Name to the relevant sheet (e.g., `Sheet1`)  
   - Define columns mapping:  
     - Email (match column)  
     - Full Name (concatenate first and last from JotForm)  
     - Phone number  
     - Requirement (client project description)  
     - project_type, confidence, summary, budget, email subject, AI generated Email body (from AI Agent output JSON)  
   - Authenticate with Google Sheets OAuth2 credentials  
   - Connect input from AI Agent node’s main output

7. **Add If Node**  
   - Type: `If`  
   - Configure condition to check if `project_type` equals `aligned` or `partially_aligned` (case insensitive)  
   - Connect input from Google Sheets node  
   - On true, connect output to "Send a message" node

8. **Add Gmail Node "Send a message"**  
   - Type: `Gmail`  
   - Set "Send To" dynamically to the client’s email from the Google Sheets data  
   - Set "Subject" dynamically to AI-generated email subject  
   - Set "Message" dynamically to AI-generated email body (HTML allowed)  
   - Disable attribution append  
   - Authenticate with Gmail OAuth2 credentials  
   - Connect input from If node (true branch)

9. **Add Sticky Note Node (Optional)**  
   - Type: `Sticky Note`  
   - Add workflow explanation content for maintainers or users  
   - Position it visually for clarity in the editor

10. **Test Workflow End-to-End**  
    - Submit a test inquiry via JotForm  
    - Confirm AI generates valid proposal JSON  
    - Confirm data logs correctly to Google Sheets  
    - Confirm conditional email sends when project_type is aligned or partially aligned  
    - Monitor for errors in authentication, API limits, or AI generation  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The AI Agent is designed to strictly output a single valid JSON object; it uses `'NAN'` string for missing or inapplicable fields instead of empty strings.      | Ensures downstream nodes can reliably parse outputs without errors.                                         |
| The internal Google Doc is the single source of truth for services, deliverables, and pricing; it must be kept up to date for accurate proposals.                | [Google Docs URL] (configured per user’s environment)                                                        |
| The workflow leverages Google Gemini (PaLM) through LangChain integration for advanced natural language understanding and generation.                          | Requires Google PaLM API access and quota.                                                                  |
| Proposal sending is gated by project alignment status to avoid sending irrelevant proposals to clients.                                                         | Helps maintain professionalism and reduces manual review workload.                                          |
| Sticky Note included for quick understanding by new maintainers; recommended to keep updated with any workflow changes.                                         | Internal documentation best practice.                                                                        |
| Credentials used: JotForm API key, Google Service Account for Docs, Google OAuth2 for Sheets, Gmail OAuth2 for sending emails, Google PaLM API key for AI model.  | Ensure all credentials have appropriate permissions and are secured.                                        |