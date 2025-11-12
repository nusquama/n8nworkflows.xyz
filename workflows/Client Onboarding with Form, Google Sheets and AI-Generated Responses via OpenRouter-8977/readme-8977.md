Client Onboarding with Form, Google Sheets and AI-Generated Responses via OpenRouter

https://n8nworkflows.xyz/workflows/client-onboarding-with-form--google-sheets-and-ai-generated-responses-via-openrouter-8977


# Client Onboarding with Form, Google Sheets and AI-Generated Responses via OpenRouter

---

## 1. Workflow Overview

This workflow automates client onboarding by collecting detailed client information via a customized public web form, logging the data into Google Sheets, generating an AI-powered client summary, and sending a personalized welcome email. It is designed for businesses seeking to streamline lead capture, triage, and initial client engagement using AI-enhanced insights and automated communication.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** A public-facing form captures comprehensive client data including contact details, company info, and project goals.

- **1.2 Data Logging:** The submitted client data is appended or updated in a Google Sheets spreadsheet for record-keeping.

- **1.3 AI Client Data Summarization:** An AI agent processes the raw client data to create a concise, factual summary for quick human triage.

- **1.4 AI-Generated Welcome Email:** Another AI agent drafts a personalized welcome email referencing both the client data and the AI-generated summary, integrating knowledge from a Pinecone vector store.

- **1.5 Communication Dispatch:** The workflow sends the email via Gmail and notifies an internal Telegram chat about the new client submission.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block triggers the workflow on submission of a public lead form that collects rich client data with validation, custom styles, and a user-friendly interface.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - *Type & Role:* Form Trigger node; serves as the webhook endpoint and user interface for client data collection.  
  - *Configuration:*  
    - Custom HTML, CSS, and JavaScript create a branded, accessible form with fields for first/last name, email, role, company info, website, phone, company size, revenue, budget, and project goals.  
    - Validation logic for email and phone formats, inline feedback, and submission control.  
    - Form Title: "Get Started with Gurey Ai"  
    - Form Description: "Speak to an expert and start a plan for your project."  
    - Webhook path: "gurey_ai.com"  
    - Bot submissions ignored to prevent spam.  
    - Responds with a confirmation text: "We have received your Form| Thanks from Gurey Ai."  
  - *Key Expressions:* Uses form field names to map inputs for downstream nodes.  
  - *Inputs:* External HTTP POST on form submission.  
  - *Outputs:* Emits the form data as JSON.  
  - *Potential Failures:*  
    - Network issues causing webhook unavailability.  
    - Malformed or incomplete form submissions despite client-side validation.  
    - Bot submissions (ignored).  
  - *Credential Requirements:* None.

---

### 2.2 Data Logging

**Overview:**  
Logs all submitted client data into a Google Sheets document, appending or updating rows to maintain a structured client database.

**Nodes Involved:**  
- Log client data  
- Execution Data (for error status capture)

**Node Details:**  

- **Log client data**  
  - *Type & Role:* Google Sheets node; appends or updates client data rows.  
  - *Configuration:*  
    - Document ID and Sheet name point to the "Form Clients" Google Sheet.  
    - Mapping of all form fields (e.g., First Name, Last Name, Email, Role, Company Size, Revenue, Budget, Website, Phone, Project Goals) plus submission date.  
    - Append or update operation with matching on "First Name" column to avoid duplicates.  
  - *Inputs:* JSON from form submission node.  
  - *Outputs:* Outputs the written row data for downstream use.  
  - *Credential Requirements:* Google Sheets OAuth2 credentials.  
  - *Potential Failures:*  
    - Authentication failures with Google API.  
    - Sheet ID or permissions issues.  
    - Network timeouts or API rate limits.  

- **Execution Data**  
  - *Type & Role:* Execution Data node; used here to mark workflow status as "Failed" in case of errors.  
  - *Configuration:* Sets a key "status" to value "Failed".  
  - *Inputs:* None connected explicitly; likely used for error handling/logging.  
  - *Outputs:* None used downstream.  

---

### 2.3 AI Client Data Summarization

**Overview:**  
Processes the raw client data by invoking an AI agent configured as a Client Data Summarization Agent. It generates a concise, factual summary of the client’s details.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model  
- Structured Output Parser

**Node Details:**  

- **AI Agent**  
  - *Type & Role:* LangChain Agent node; uses LLM to summarize client data.  
  - *Configuration:*  
    - Text prompt injects all relevant form fields into a template instructing the agent to produce a concise, neutral client summary based solely on provided data.  
    - System message guides the agent on style, boundaries, and output format.  
    - Has an output parser enabled to ensure structured JSON output.  
  - *Inputs:* JSON from "Log client data" node (form fields).  
  - *Outputs:* JSON containing the client summary under key "client summary".  
  - *Credential Requirements:* OpenRouter API credentials.  
  - *Potential Failures:*  
    - API authentication or rate limit issues.  
    - LLM timeouts or malformed responses.  
    - Parsing errors if output deviates from expected schema.

- **OpenRouter Chat Model**  
  - *Type & Role:* Language Model node; provides LLM backend for the AI Agent.  
  - *Configuration:* Uses default OpenRouter model with API credentials.  
  - *Inputs:* Connected as AI language model for the AI Agent.  
  - *Outputs:* Raw LLM responses.  
  - *Credential Requirements:* OpenRouter API key.

- **Structured Output Parser**  
  - *Type & Role:* Output parser node; ensures AI Agent output conforms to a JSON schema.  
  - *Configuration:* JSON schema example requires a "client summary" string field.  
  - *Inputs:* Raw LLM output from OpenRouter Chat Model.  
  - *Outputs:* Parsed, validated JSON.  
  - *Potential Failures:* Parsing errors if AI output is malformed.  
  - *Auto-fix:* Enabled to attempt minor corrections.

---

### 2.4 AI-Generated Welcome Email

**Overview:**  
Generates a personalized welcome email for the client, referencing their data and the AI-generated summary. This agent uses the Pinecone vector store as a knowledge base to enrich and contextualize the email content.

**Nodes Involved:**  
- Email Agent  
- Pinecone Vector Store  
- Embeddings OpenAI  
- OpenRouter Chat Model1  
- Structured Output Parser1

**Node Details:**  

- **Email Agent**  
  - *Type & Role:* LangChain Agent node; drafts a welcome email including subject and body.  
  - *Configuration:*  
    - Text prompt includes client data and the client summary from the previous AI Agent.  
    - System message instructs a warm, professional tone, referencing client goals and knowledge base content.  
    - Output parser enabled with schema requiring "subject" and "body" fields.  
  - *Inputs:*  
    - Client data from "Log client data" node.  
    - Client summary from "AI Agent".  
    - Knowledge retrieved from Pinecone vector store for content enrichment.  
  - *Outputs:* JSON with email "subject" and "body".  
  - *Credential Requirements:* OpenRouter API key for LLM, Pinecone API key for vector store.  
  - *Potential Failures:*  
    - API auth or timeouts.  
    - Parsing errors on structured output.  
    - Missing client data leading to fallback placeholders in email.

- **Pinecone Vector Store**  
  - *Type & Role:* Vector store node; retrieves relevant documents from a Pinecone namespace "Email Automation" to support the Email Agent.  
  - *Configuration:*  
    - Uses Pinecone index "databases".  
    - Retrieval mode as a tool for the agent's knowledge base.  
  - *Inputs:* Embeddings from OpenAI Embeddings node.  
  - *Outputs:* Contextual knowledge snippets.  
  - *Credential Requirements:* Pinecone API key.  
  - *Potential Failures:* Network or authentication issues.

- **Embeddings OpenAI**  
  - *Type & Role:* Embeddings node; transforms query text into vector embeddings for Pinecone retrieval.  
  - *Configuration:* 512 dimensions.  
  - *Inputs:* Text queries from Email Agent.  
  - *Outputs:* Embeddings for vector store search.  
  - *Credential Requirements:* OpenAI API key.

- **OpenRouter Chat Model1**  
  - *Type & Role:* Language Model node; LLM backend for Email Agent, using Anthropic Claude 3.5 Sonnet model via OpenRouter.  
  - *Credential Requirements:* OpenRouter API key.

- **Structured Output Parser1**  
  - *Type & Role:* Output parser for Email Agent’s response.  
  - *Schema:* JSON with "subject" and "body" fields for email content.  
  - *Auto-fix:* Enabled.

---

### 2.5 Communication Dispatch

**Overview:**  
Sends the generated welcome email via Gmail and notifies a Telegram chat about the new client submission.

**Nodes Involved:**  
- Send a message (Gmail)  
- Send a text message (Telegram)

**Node Details:**  

- **Send a message (Gmail)**  
  - *Type & Role:* Gmail node; sends email to the client or internal email address.  
  - *Configuration:*  
    - Recipient email is statically set to "{Your email}" (placeholder, should be replaced).  
    - Subject and message body populated from Email Agent output.  
    - Email type: plain text.  
  - *Inputs:* JSON from Email Agent.  
  - *Outputs:* Confirmation of email sent.  
  - *Credential Requirements:* Gmail OAuth2 credentials.  
  - *Potential Failures:*  
    - Authentication errors.  
    - Invalid recipient address.  
    - Gmail API rate limits.

- **Send a text message (Telegram)**  
  - *Type & Role:* Telegram node; sends a notification to internal team chat.  
  - *Configuration:*  
    - Static chat ID "6158704034".  
    - Message includes links to Google Sheet and workflow URL for quick access.  
    - Attribution disabled.  
  - *Inputs:* Triggered after email sent.  
  - *Outputs:* Confirmation of message sent.  
  - *Credential Requirements:* Telegram Bot API credentials.  
  - *Potential Failures:*  
    - Invalid chat ID.  
    - Bot permissions.  
    - Network issues.

---

## 3. Summary Table

| Node Name              | Node Type                                  | Functional Role                        | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                    |
|------------------------|--------------------------------------------|-------------------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger                              | Client data input via public form    | —                      | Log client data          | Public-facing lead form with custom HTML/CSS/JS.                                                               |
| Log client data        | Google Sheets                            | Append/update client data in sheet   | On form submission      | AI Agent                 | Logs client data securely into Google Sheets.                                                                 |
| Execution Data         | Execution Data                           | Record error status                  | —                      | —                       | Marks workflow status as "Failed" for error handling.                                                         |
| AI Agent               | LangChain Agent                         | Summarize client data with AI        | Log client data         | Log summary into sheet   | Summarizes client info into concise JSON format.                                                              |
| OpenRouter Chat Model  | LangChain LLM                           | Provides LLM backend for AI Agent    | —                      | AI Agent, Structured Output Parser | Requires OpenRouter API credentials.                                                                            |
| Structured Output Parser| Output Parser                           | Validates AI summary output           | OpenRouter Chat Model   | AI Agent                 | Ensures client summary JSON format.                                                                            |
| Log summary into sheet | Google Sheets                            | Updates sheet with AI summary         | AI Agent                | Email Agent              | Updates Google Sheet with client summary.                                                                      |
| Email Agent            | LangChain Agent                         | Draft personalized welcome email     | Log summary into sheet, Pinecone Vector Store | Send a message           | Generates email subject and body using client data & knowledge base.                                          |
| Pinecone Vector Store  | Vector Store                            | Provides knowledge base for Email Agent | Embeddings OpenAI       | Email Agent              | Retrieves contextual info from Pinecone namespace "Email Automation".                                          |
| Embeddings OpenAI      | Embeddings                             | Creates embeddings for Pinecone      | Email Agent             | Pinecone Vector Store    | Uses OpenAI for vector embedding generation.                                                                   |
| OpenRouter Chat Model1 | LangChain LLM                          | LLM backend for Email Agent (Claude) | —                      | Email Agent, Structured Output Parser1 | Requires OpenRouter API credentials.                                                                            |
| Structured Output Parser1| Output Parser                         | Validates email JSON output           | OpenRouter Chat Model1  | Email Agent              | Ensures email subject/body JSON format.                                                                        |
| Send a message         | Gmail                                  | Sends welcome email                   | Email Agent             | Send a text message      | Sends email via Gmail OAuth2.                                                                                   |
| Send a text message    | Telegram                               | Notifies internal team of new client | Send a message          | —                       | Sends Telegram notification with sheet and workflow links.                                                    |
| Sticky Note            | Sticky Note                           | Documentation & branding              | —                      | —                       | Full workflow description and YouTube channel link.                                                           |
| Sticky Note2           | Sticky Note                           | Documentation & branding              | —                      | —                       | YouTube subscription reminder.                                                                                 |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add "On form submission" (Form Trigger) node:**
   - Set webhook path to "gurey_ai.com".
   - Paste the provided custom HTML, CSS, and JavaScript to create the branded form.
   - Configure form fields to include:
     - First Name (required)
     - Last Name
     - Email (type email, required)
     - Your Role within Organization (required)
     - Company Name (required)
     - Website (optional)
     - Phone Number (optional)
     - Company Size (dropdown, required)
     - Annual Revenue (dropdown, required)
     - Project Budget (dropdown, required)
     - Project Goals (textarea, required)
   - Enable "Ignore Bots".
   - Set confirmation text to "We have received your Form| Thanks from Gurey Ai".
   - No credentials needed.

3. **Add "Log client data" (Google Sheets) node:**
   - Connect input from "On form submission" node.
   - Operation: Append or Update.
   - Set Google Sheet document ID and sheet name ("Form Clients", gid=0).
   - Map form fields to columns accordingly, including submission date.
   - Set matching column as "First Name".
   - Use Google Sheets OAuth2 API credentials.

4. **Add "AI Agent" (LangChain Agent) node:**
   - Connect input from "Log client data".
   - Configure prompt to inject all client data fields with the provided system message defining the summarization task.
   - Enable output parser.
   - Use "OpenRouter Chat Model" as AI language model.

5. **Add "OpenRouter Chat Model" node:**
   - Configure with OpenRouter API credentials.
   - No additional parameters needed.
   - Connect as AI language model for "AI Agent".

6. **Add "Structured Output Parser" node:**
   - Configure with JSON schema expecting "client summary" string.
   - Enable auto-fix.
   - Connect input from "OpenRouter Chat Model".
   - Connect output to "AI Agent".

7. **Add "Log summary into sheet" (Google Sheets) node:**
   - Connect input from "AI Agent".
   - Operation: Update.
   - Use same Google Sheet and sheet as "Log client data".
   - Update "Summarization" column matching on "First Name".
   - Use Google Sheets OAuth2 credentials.

8. **Add "Email Agent" (LangChain Agent) node:**
   - Connect input from "Log summary into sheet".
   - Configure prompt to include client data and client summary.
   - Use system message instructing to create a warm, professional personalized welcome email.
   - Enable output parser expecting JSON with "subject" and "body".
   - Use "OpenRouter Chat Model1" as AI language model.
   - Use "Pinecone Vector Store" as knowledge base tool.

9. **Add "Pinecone Vector Store" node:**
   - Connect input from "Embeddings OpenAI".
   - Configure with Pinecone index "databases" and namespace "Email Automation".
   - Use Pinecone API credentials.
   - Set mode to "retrieve-as-tool".

10. **Add "Embeddings OpenAI" node:**
    - Connect input from "Email Agent".
    - Configure dimension size 512.
    - Use OpenAI API credentials.

11. **Add "OpenRouter Chat Model1" node:**
    - Configure with OpenRouter API credentials.
    - Use Anthropic/Claude 3.5 Sonnet model.

12. **Add "Structured Output Parser1" node:**
    - Configure JSON schema expecting "subject" and "body".
    - Enable auto-fix.
    - Connect input from "OpenRouter Chat Model1".
    - Connect output to "Email Agent".

13. **Add "Send a message" (Gmail) node:**
    - Connect input from "Email Agent".
    - Set recipient email to intended address (replace placeholder "{Your email}").
    - Use email subject and body fields from Email Agent output.
    - Use Gmail OAuth2 credentials.

14. **Add "Send a text message" (Telegram) node:**
    - Connect input from "Send a message" node.
    - Configure chat ID to internal Telegram group or user.
    - Message text includes links to the Google Sheet and workflow URL.
    - Use Telegram Bot API credentials.

15. **Add Sticky Note nodes as documentation (optional):**
    - Copy content from provided sticky notes for internal documentation and branding.

16. **Test the workflow end-to-end:**
    - Submit the form.
    - Verify data logs into Google Sheets.
    - Confirm AI summary is generated and saved.
    - Confirm personalized email is composed and sent.
    - Confirm Telegram notification is received.

17. **Set error handling as needed:**
    - Use Execution Data node or n8n's native error handling for logging failures.

---

## 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is branded as "Gurey AI Partnership Form & Client Triage Workflow".                                | Workflow title and branding.                                                                   |
| Book a call with Gurey Osman: [Calendly Link](https://calendly.com/gureyosman2008/30min)                         | For direct partnership inquiries.                                                              |
| Workflow overview video and updates available on YouTube: [Gurey Osman YouTube Channel](https://www.youtube.com/@gureyosman06) | For learning more about n8n workflows and AI integrations.                                     |
| Google Sheets OAuth2 API Credentials setup guide: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleSheets/#credentials | Official documentation for Google Sheets API setup.                                           |
| OpenRouter API key setup: https://openrouter.ai/keys                                                             | For access to large language models used in this workflow.                                     |
| Pinecone vector store used as a knowledge base for contextual email generation.                                  | Pinecone documentation: https://www.pinecone.io/docs/                                           |
| Ensure to replace placeholder email "{Your email}" with actual recipient address before deployment.              | Important for correct email delivery.                                                          |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---