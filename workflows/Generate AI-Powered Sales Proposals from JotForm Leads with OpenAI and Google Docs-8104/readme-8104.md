Generate AI-Powered Sales Proposals from JotForm Leads with OpenAI and Google Docs

https://n8nworkflows.xyz/workflows/generate-ai-powered-sales-proposals-from-jotform-leads-with-openai-and-google-docs-8104


# Generate AI-Powered Sales Proposals from JotForm Leads with OpenAI and Google Docs

### 1. Workflow Overview

This workflow automates the generation and delivery of AI-powered sales proposals based on leads captured via JotForm. It targets sales teams and business developers who want to streamline proposal creation by leveraging AI analysis of sales call transcripts and prospect data. The workflow’s logical structure can be divided into these functional blocks:

- **1.1 Lead Capture and Template Preparation:** Triggered by a new JotForm submission, it copies a Google Drive proposal template and obtains an audio call recording linked in the form.
- **1.2 Audio Transcription:** The audio call recording is transcribed to text using OpenAI’s speech-to-text capabilities.
- **1.3 AI Proposal Generation:** Using the prospect’s details and the transcribed call text, OpenAI’s GPT-4 model generates a detailed, structured sales proposal in JSON format.
- **1.4 Proposal Document Editing:** The proposal JSON data is merged into a Google Docs template by replacing placeholders.
- **1.5 PDF Conversion and Email Delivery:** The updated Google Docs document is converted to a PDF and emailed to the prospect.

This logical separation clarifies the workflow’s data flow from lead acquisition through AI processing to final document delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Capture and Template Preparation

- **Overview:** Captures new lead submissions from JotForm, copies a Google Drive proposal template file for each lead, and downloads an audio call recording linked in the form to prepare for transcription.
- **Nodes Involved:** JotForm Trigger, Google Drive, Google Drive1

**Node Details:**

- **JotForm Trigger**
  - *Type:* Trigger node for JotForm submissions.
  - *Configuration:* Listens to form ID `251206359432049`.
  - *Inputs:* External webhook from JotForm on form submission.
  - *Outputs:* JSON with form fields such as First Name, Last Name, Company Name, Email, and Call Link.
  - *Failure modes:* Webhook connectivity issues, invalid form ID, or missing expected fields.
  - *Notes:* Requires JotForm API credentials and correct form ID.

- **Google Drive**
  - *Type:* Google Drive operation for file copying.
  - *Configuration:* Copies a Google Docs template file identified by file ID `1DSHUhq_DoM80cM7LZ5iZs6UGoFb3ZHsLpU3mZDuQwuQ`.
  - *Filename:* Named dynamically as `{Company Name} | Ai Proposal`.
  - *Inputs:* Triggered by JotForm Trigger output.
  - *Outputs:* JSON including the new file’s ID.
  - *Failure modes:* OAuth credential expiration, missing permissions, file not found.
  - *Notes:* Requires Google Drive OAuth2 credentials with edit access.

- **Google Drive1**
  - *Type:* Google Drive file download.
  - *Configuration:* Downloads the audio file for transcription using URL from the form field `Call Link`.
  - *Inputs:* Output of Google Drive node.
  - *Outputs:* Binary audio data.
  - *Failure modes:* Invalid or inaccessible audio URL, network timeouts.
  - *Notes:* Ensures the audio file is accessible for transcription.

---

#### 2.2 Audio Transcription

- **Overview:** Transcribes the audio call recording into text using OpenAI’s speech-to-text capability.
- **Nodes Involved:** OpenAI

**Node Details:**

- **OpenAI**
  - *Type:* OpenAI audio transcription operation.
  - *Configuration:* Resource set to `audio`, operation set to `transcribe`.
  - *Inputs:* Binary audio data from Google Drive1 node.
  - *Outputs:* JSON with transcribed text under `text` field.
  - *Failure modes:* API quota limits, audio format unsupported, transcription errors.
  - *Notes:* Requires OpenAI API key configured under credentials.

---

#### 2.3 AI Proposal Generation

- **Overview:** Uses GPT-4 to analyze prospect details and the transcribed sales call to generate a detailed sales proposal as a JSON object.
- **Nodes Involved:** OpenAI1

**Node Details:**

- **OpenAI1**
  - *Type:* OpenAI Chat Completion (GPT-4).
  - *Configuration:* Model set to `gpt-4.1-mini`. Input prompt includes prospect details (First Name, Last Name, Company Name) and the sales call transcript.
  - *Prompt:* Detailed system instructions specifying output format (JSON) and proposal content requirements including problem statements, solution, pricing, and upsell.
  - *Inputs:* Text transcription from OpenAI node, plus JotForm data.
  - *Outputs:* JSON with structured proposal fields (problem statement, AI system name, pricing, upsell, etc.).
  - *Failure modes:* API rate limits, malformed prompt expressions, unexpected output format.
  - *Notes:* Strict adherence to JSON output is enforced for downstream processing.

---

#### 2.4 Proposal Document Editing

- **Overview:** Updates a Google Docs template by replacing placeholders with the generated proposal data.
- **Nodes Involved:** Google Docs

**Node Details:**

- **Google Docs**
  - *Type:* Google Docs update operation.
  - *Configuration:* Uses the copied Google Docs file ID from the Google Drive node.
  - *Replacements:* Maps template placeholders (e.g. `{{problemStatement}}`) to corresponding JSON fields from OpenAI1 output.
  - *Inputs:* Proposal JSON from OpenAI1 and file ID from Google Drive.
  - *Outputs:* Updated Google Docs document metadata.
  - *Failure modes:* Missing placeholders, insufficient permissions, API errors.
  - *Notes:* Requires Google Docs OAuth2 credentials with edit rights.

---

#### 2.5 PDF Conversion and Email Delivery

- **Overview:** Converts the updated Google Docs to PDF format and sends it as an email attachment to the prospect.
- **Nodes Involved:** Google Drive2, Gmail

**Node Details:**

- **Google Drive2**
  - *Type:* Google Drive file download with conversion.
  - *Configuration:* Downloads the updated Google Docs file and converts it to PDF (`docsToFormat` set to `application/pdf`).
  - *Inputs:* Document ID from Google Docs node.
  - *Outputs:* Binary PDF data labeled as `proposal`.
  - *Failure modes:* API rate limits, conversion failures.
  - *Notes:* Uses same Google Drive credentials as previous nodes.

- **Gmail**
  - *Type:* Email sending node using Gmail.
  - *Configuration:* Drafts an email with subject including `{Company Name}` and AI system name, body personalized with prospect’s first name and proposal context.
  - *Attachments:* Includes the generated PDF proposal.
  - *Inputs:* Binary PDF from Google Drive2, email address from JotForm Trigger.
  - *Outputs:* Email draft sent via Gmail API.
  - *Failure modes:* OAuth token expiry, email delivery errors, invalid email addresses.
  - *Notes:* Requires Gmail OAuth2 credentials configured.

---

### 3. Summary Table

| Node Name      | Node Type                          | Functional Role                  | Input Node(s)      | Output Node(s)    | Sticky Note                                                                                                          |
|----------------|----------------------------------|--------------------------------|--------------------|-------------------|----------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger | n8n-nodes-base.jotFormTrigger    | Lead submission trigger         | (None, Trigger)    | Google Drive      | See detailed connection and setup instructions in Sticky Note1.                                                     |
| Google Drive   | n8n-nodes-base.googleDrive       | Copy proposal template          | JotForm Trigger    | Google Drive1     |                                                                                                                      |
| Google Drive1  | n8n-nodes-base.googleDrive       | Download audio call recording   | Google Drive       | OpenAI            |                                                                                                                      |
| OpenAI         | @n8n/n8n-nodes-langchain.openAi | Transcribe audio to text        | Google Drive1      | OpenAI1           |                                                                                                                      |
| OpenAI1        | @n8n/n8n-nodes-langchain.openAi | Generate AI sales proposal JSON | OpenAI             | Google Docs       |                                                                                                                      |
| Google Docs    | n8n-nodes-base.googleDocs        | Update proposal document        | OpenAI1            | Google Drive2     |                                                                                                                      |
| Google Drive2  | n8n-nodes-base.googleDrive       | Convert Google Doc to PDF       | Google Docs        | Gmail             |                                                                                                                      |
| Gmail          | n8n-nodes-base.gmail             | Email proposal to prospect      | Google Drive2      | (None)            |                                                                                                                      |
| Sticky Note1   | n8n-nodes-base.stickyNote        | Documentation & setup guide     | (None)             | (None)            | ⚙️ Proposal Generator Template (Automates proposal creation from JotForm submissions). See setup steps and links.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a JotForm Trigger Node:**
   - Node Type: `JotForm Trigger`
   - Configure with your JotForm API credentials.
   - Set `form` parameter to your form ID (e.g. `251206359432049`).
   - Connect this node as the starting trigger.

2. **Add Google Drive Node to Copy Template:**
   - Node Type: `Google Drive`
   - Set operation to `copy`.
   - Provide the template file ID (Google Docs template).
   - Set the filename dynamically as `={{ $json["Company Name"] }} | Ai Proposal`.
   - Connect output of JotForm Trigger to this node.
   - Use Google Drive OAuth2 credentials with edit permissions.

3. **Add Google Drive Node to Download Audio:**
   - Node Type: `Google Drive`
   - Set operation to `download`.
   - Configure `fileId` mode as URL, value to `={{ $('JotForm Trigger').item.json['Call Link'] }}`.
   - Connect output of Google Drive copy node to this node.
   - Use same Google Drive credentials.

4. **Add OpenAI Node for Transcription:**
   - Node Type: `@n8n/n8n-nodes-langchain.openAi`
   - Set resource to `audio` and operation to `transcribe`.
   - Connect output of Google Drive download node to this node.
   - Use OpenAI API credentials.

5. **Add OpenAI Node for Proposal Generation:**
   - Node Type: `@n8n/n8n-nodes-langchain.openAi`
   - Set model to `gpt-4.1-mini` or preferred GPT-4 variant.
   - Prepare prompt messages:
     - User message includes prospect details from JotForm and transcribed text from previous node.
     - System message includes detailed SOP and instructions to output proposal JSON only.
   - Connect transcription output to this node.
   - Use the same OpenAI credentials.

6. **Add Google Docs Node to Update Proposal Document:**
   - Node Type: `Google Docs`
   - Set operation to `update`.
   - Provide the copied Google Docs file ID from Google Drive copy node.
   - Configure placeholder replacements mapping each JSON key from the OpenAI1 output to matching placeholders in the Google Docs template.
   - Connect OpenAI1 output to this node.
   - Use Google Docs OAuth2 credentials with edit rights.

7. **Add Google Drive Node to Convert Google Doc to PDF:**
   - Node Type: `Google Drive`
   - Set operation to `download`.
   - Set `fileId` to the updated Google Docs file ID.
   - Enable Google file conversion from Docs to PDF.
   - Connect Google Docs node output to this node.
   - Use Google Drive credentials.

8. **Add Gmail Node to Send Email:**
   - Node Type: `Gmail`
   - Configure to send an email draft with dynamic subject: `={{ $('JotForm Trigger').item.json['Company Name'] }} | {{ $('OpenAI1').item.json.choices[0].message.content.aiSystem }} Proposal`.
   - Body text personalized with the prospect’s first name and proposal context.
   - Attach the PDF from Google Drive2 node.
   - Send to the email address from JotForm submission.
   - Use Gmail OAuth2 credentials.

9. **Add a Sticky Note Node (Optional):**
   - Add a sticky note summarizing the setup instructions, credentials requirements, and node interconnections.
   - Position it clearly for users referencing the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Setup requires API keys and OAuth2 credentials for JotForm, Google Drive, Google Docs, OpenAI, and Gmail.            | Refer to sticky note in workflow for detailed credential setup instructions.                            |
| The OpenAI prompt includes a detailed SOP to ensure JSON-only output for structured proposal generation.             | Ensures downstream nodes can reliably parse and inject data into Google Docs.                          |
| The Google Docs template must include placeholders matching the keys in the AI JSON output for seamless replacement. | Placeholders look like `{{problemStatement}}`, `{{aiSystem}}`, etc.                                     |
| The email node sends a draft email with the proposal PDF attached to the prospect’s email from the form submission.   | Uses Gmail OAuth2 credentials; ensure correct scopes are enabled for attachment sending.               |
| Workflow author credits: LeeWei                                                                                      | See sticky note for author and additional setup tips.                                                  |
| JotForm and Google Drive links for API key generation and OAuth setup are provided in the sticky note content.       | [JotForm](https://www.jotform.com/), [Google Drive](https://www.google.com/drive/), [OpenAI](https://platform.openai.com/) |

---

**Disclaimer:** The content provided is exclusively derived from an n8n automation workflow. All data processed and generated respects applicable content policies and contains no illegal, offensive, or protected materials.