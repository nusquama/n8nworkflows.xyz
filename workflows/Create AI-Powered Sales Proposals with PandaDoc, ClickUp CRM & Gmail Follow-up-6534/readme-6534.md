Create AI-Powered Sales Proposals with PandaDoc, ClickUp CRM & Gmail Follow-up

https://n8nworkflows.xyz/workflows/create-ai-powered-sales-proposals-with-pandadoc--clickup-crm---gmail-follow-up-6534


# Create AI-Powered Sales Proposals with PandaDoc, ClickUp CRM & Gmail Follow-up

### 1. Workflow Overview

This workflow automates the creation of customized sales proposals by integrating AI-generated content, document creation, CRM updates, and email follow-up. It is designed for agencies, freelancers, and sales teams who conduct sales calls and want to quickly turn call notes into professional proposals without manual formatting or data entry.

Logical blocks:

- **1.1 Input Reception:** Captures post-sales-call data via a Typeform form trigger.
- **1.2 AI Proposal Generation:** Uses OpenAI GPT-4o model to generate a structured, detailed sales proposal based on form inputs.
- **1.3 Proposal Document Creation:** Sends AI-generated proposal data to PandaDoc to create a customized proposal document.
- **1.4 CRM Update:** Updates the corresponding ClickUp lead/task with company name, proposal URL, and quote details.
- **1.5 Email Draft Creation:** Drafts a personalized Gmail email to the prospect with the proposal link ready for review and sending.
- **1.6 Informational Notes:** Sticky notes providing guidance, instructions, and credits related to the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives form responses immediately after a sales call to collect client details, pain points, solutions pitched, deliverables, timeline, pricing, and other relevant data.

**Nodes Involved:**  
- Sales Call Form (Typeform Trigger)

**Node Details:**

- **Sales Call Form**  
  - Type: Typeform Trigger  
  - Role: Receives webhook events from a Typeform form submission.  
  - Configuration: Set with form ID ("RADFsrT7") and webhook ID; configured to capture full form response without simplifying answers.  
  - Input: Webhook trigger from Typeform submission.  
  - Output: JSON containing detailed answers array and hidden fields (e.g., ClickUp task ID).  
  - Edge cases: Missing or incomplete form data could cause downstream failures; webhook connectivity issues could interrupt triggering.

---

#### 2.2 AI Proposal Generation

**Overview:**  
Generates a highly customized sales proposal in JSON format using OpenAI GPT-4o based on the form data.

**Nodes Involved:**  
- Generate Proposal Copy (OpenAI GPT-4o)

**Node Details:**

- **Generate Proposal Copy**  
  - Type: OpenAI (LangChain) node  
  - Role: Generates structured proposal copy with titles, problem/solution descriptions, scope items, deliverables, and timelines.  
  - Configuration: Uses "gpt-4o" model; system prompt sets assistant as intelligent writing assistant; detailed instructions include tone, format, content requirements, and JSON structure.  
  - Key expressions: Inputs form fields such as company description, problem, solution, deliverables, timeline, and dynamically builds prompt content for AI.  
  - Input: JSON from Typeform node with form answers.  
  - Output: JSON proposal object with fields like proposalTitle, problemTitle, problemText, solutionTitle, solutionText, scope1-4, deliverable1-3, timeline1-3.  
  - Edge cases: API rate limits; malformed prompt expressions; incomplete or malformed form data; AI generating incomplete JSON; network timeouts.  
  - Credentials: OpenAI API key configured.

---

#### 2.3 Proposal Document Creation

**Overview:**  
Creates a PandaDoc proposal document using the generated AI proposal data, populating a predefined template with dynamic tokens and recipient information.

**Nodes Involved:**  
- Create PandaDoc Proposal (HTTP Request)

**Node Details:**

- **Create PandaDoc Proposal**  
  - Type: HTTP Request  
  - Role: Sends a POST request to PandaDoc API to create a new document from a specific template.  
  - Configuration:  
    - URL: PandaDoc documents endpoint  
    - Method: POST  
    - Headers: Authorization with PandaDoc API key, Accept as JSON  
    - Body: JSON payload includes owner email, template UUID (needs to be replaced by user), document name (using company name from form), metadata (including ClickUp ID), tokens mapping AI-generated proposal fields to PandaDoc tokens, recipients array with client email and names, and pricing table with proposal title and price.  
  - Key expressions: Uses expressions to map data from AI node and form node, e.g. proposal content fields, company name, client emails, pricing number, and ClickUp IDs.  
  - Input: AI-generated proposal JSON and form data.  
  - Output: PandaDoc document creation response (including document ID).  
  - Edge cases: API key or template UUID missing or invalid; network errors; token fields mismatch; malformed JSON payload; API rate limits or authentication failures.  
  - Requires user to replace "<template id>" and "<your_api_key>" with actual PandaDoc credentials.

---

#### 2.4 CRM Update

**Overview:**  
Updates the corresponding ClickUp task/lead to reflect the new proposal details: company name, proposal URL, and quoted price.

**Nodes Involved:**  
- ClickUp (Update task)  
- Add Company Name (Set custom field)  
- Quote (Set custom field)  
- Proposal URL (Set custom field)

**Node Details:**

- **ClickUp**  
  - Type: ClickUp node (Update operation)  
  - Role: Updates task content with a concatenated summary of key sales call details (company summary, problem, solution, deliverables, timeline).  
  - Input: PandaDoc proposal creation output (to get ClickUp task ID) and form data.  
  - Output: Updated ClickUp task JSON, including custom fields.  
  - Credentials: OAuth2 ClickUp account.  
  - Edge cases: OAuth token expiration; invalid task ID; API errors.

- **Add Company Name**  
  - Type: ClickUp node (Set custom field)  
  - Role: Updates the custom field for company name on the ClickUp task.  
  - Input: Task ID from previous node, company name from form answers.  
  - Output: Confirmation of custom field update.

- **Quote**  
  - Type: ClickUp node (Set custom field)  
  - Role: Sets the quote/custom price field on the task using the price from the form response.  
  - Input: Task ID, price number from form answers.  
  - Output: Confirmation of update.

- **Proposal URL**  
  - Type: ClickUp node (Set custom field)  
  - Role: Sets a custom field with the URL link to the generated PandaDoc proposal.  
  - Input: Task ID, PandaDoc document ID to build URL.  
  - Output: Confirmation of update.

**Edge cases for this block:**  
- Invalid or missing custom field IDs may cause failures.  
- OAuth token refresh required.  
- API rate limits or network issues.  
- Mismatched field types (e.g., setting string in number field).

---

#### 2.5 Email Draft Creation

**Overview:**  
Creates a Gmail draft email addressed to the client, personalized with their first name and containing the proposal link for easy follow-up.

**Nodes Involved:**  
- Create a Draft (Gmail node)

**Node Details:**

- **Create a Draft**  
  - Type: Gmail node (Create draft)  
  - Role: Composes an HTML email draft with a personalized message thanking the client, referencing the proposal link, and prompting for feedback.  
  - Configuration:  
    - Recipient email from form answers.  
    - Subject: Fixed string "sent you the Proposal".  
    - Message body: Uses HTML formatting and expressions to insert client first name and proposal placeholders.  
  - Input: Form submission data and PandaDoc proposal link.  
  - Output: Gmail draft created in the user’s Gmail account.  
  - Credentials: Gmail OAuth2 account configured.  
  - Edge cases: OAuth token expiry; invalid recipient email; Gmail API quota limits.

---

#### 2.6 Informational Notes (Sticky Notes)

**Overview:**  
Provides visual documentation, guidance, credits, and instructions embedded inside the workflow for user reference.

**Nodes Involved:**  
- Sticky Note1  
- Sticky Note  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

**Node Details:**

- **Sticky Note1**: "Fill out quick form after call -> Generate Personalized PandaDoc Proposal"  
- **Sticky Note**: "Update Lead in CRM -> Add company name, proposal URL & quote fields"  
- **Sticky Note2**: "Create Email Draft"  
- **Sticky Note3**: Detailed overview of the entire workflow, use cases, setup instructions, and customization guidance.  
- **Sticky Note4**: Personal introduction and contact info of the workflow author Abdul, including website and email.

---

### 3. Summary Table

| Node Name              | Node Type                 | Functional Role                                  | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                              |
|------------------------|---------------------------|-------------------------------------------------|-----------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Sales Call Form         | Typeform Trigger          | Receive sales call form submission               | -                           | Generate Proposal Copy         | Fill out quick form after call -> Generate Personalized PandaDoc Proposal                                                  |
| Generate Proposal Copy  | OpenAI (LangChain)        | Generate AI-powered proposal JSON                 | Sales Call Form             | Create PandaDoc Proposal       | Fill out quick form after call -> Generate Personalized PandaDoc Proposal                                                  |
| Create PandaDoc Proposal| HTTP Request              | Create PandaDoc document from AI proposal data    | Generate Proposal Copy      | ClickUp                       | Fill out quick form after call -> Generate Personalized PandaDoc Proposal                                                  |
| ClickUp                | ClickUp API Update        | Update task content with sales call summary       | Create PandaDoc Proposal    | Add Company Name, Quote, Proposal URL | Update Lead in CRM -> Add company name, proposal URL & quote fields                                                      |
| Add Company Name        | ClickUp Set Custom Field  | Set company name custom field on ClickUp task     | ClickUp                    | -                             | Update Lead in CRM -> Add company name, proposal URL & quote fields                                                       |
| Quote                   | ClickUp Set Custom Field  | Set quote/price custom field on ClickUp task      | ClickUp                    | -                             | Update Lead in CRM -> Add company name, proposal URL & quote fields                                                       |
| Proposal URL            | ClickUp Set Custom Field  | Set proposal URL custom field on ClickUp task     | ClickUp                    | Create a Draft                | Update Lead in CRM -> Add company name, proposal URL & quote fields                                                       |
| Create a Draft          | Gmail Create Draft        | Draft personalized follow-up email to client      | Proposal URL               | -                             | Create Email Draft                                                                                                        |
| Sticky Note1            | StickyNote                | Instruction: Quick form to proposal generation     | -                           | -                             | Fill out quick form after call -> Generate Personalized PandaDoc Proposal                                                  |
| Sticky Note             | StickyNote                | Instruction: Update lead in CRM                    | -                           | -                             | Update Lead in CRM -> Add company name, proposal URL & quote fields                                                       |
| Sticky Note2            | StickyNote                | Instruction: Create email draft                    | -                           | -                             | Create Email Draft                                                                                                        |
| Sticky Note3            | StickyNote                | Full workflow overview, audience, and setup guide | -                           | -                             | See full detailed workflow overview and setup instructions                                                               |
| Sticky Note4            | StickyNote                | Author intro and contact info                       | -                           | -                             | Hey, I'm Abdul 👋 Build growth systems for consultants & agencies. Website and email contact info provided                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Typeform Trigger node** named "Sales Call Form"  
   - Set Form ID to your Typeform form that collects sales call data.  
   - Configure webhook ID or enable webhook in Typeform.  
   - Set "Only Answers" to false and "Simplify Answers" to false to get full raw data.

2. **Add an OpenAI LangChain node** named "Generate Proposal Copy"  
   - Select model "gpt-4o".  
   - Configure system prompt to instruct AI to generate a JSON proposal with fields: proposalTitle, problemTitle, problemText, solutionTitle, solutionText, 4 scope items, 3 deliverables, 3 timelines.  
   - Use form answers as variables in the prompt content.  
   - Connect output of "Sales Call Form" to this node.

3. **Add an HTTP Request node** named "Create PandaDoc Proposal"  
   - Set method to POST.  
   - URL: `https://api.pandadoc.com/public/v1/documents`  
   - Add HTTP headers:  
     - Authorization: `API-Key <your_api_key>` (replace with your PandaDoc API key)  
     - Accept: `application/json`  
   - Set Body Content-Type to JSON and configure body to map AI proposal JSON fields to PandaDoc tokens.  
   - Include recipient info from form answers.  
   - Include pricing table with proposal title and price from form answers.  
   - Connect output of "Generate Proposal Copy" to this node.

4. **Add a ClickUp node** named "ClickUp"  
   - Set operation to Update.  
   - Use ClickUp task ID from form hidden field.  
   - Update task content with a summary constructed from form answers (company description, problem, solution pitched, deliverables, timeline).  
   - Authenticate via OAuth2 with your ClickUp account.  
   - Connect output of "Create PandaDoc Proposal" to this node.

5. **Add three ClickUp nodes** for setting custom fields on the task:  
   - "Add Company Name": Set custom field for company name using form answer.  
   - "Quote": Set custom field for quote/price using form answer.  
   - "Proposal URL": Set custom field for proposal URL using PandaDoc document ID.  
   - Authenticate each node with the same ClickUp OAuth2 credentials.  
   - Connect "ClickUp" node output to these three nodes (in parallel or sequence).

6. **Add a Gmail node** named "Create a Draft"  
   - Operation: Create Draft.  
   - Set recipient email from form answers.  
   - Compose an HTML email including the client’s first name and proposal link.  
   - Authenticate with Gmail OAuth2 credentials.  
   - Connect output of "Proposal URL" to this node.

7. **Add Sticky Note nodes** as desired for internal documentation and guidance:  
   - Add notes for form usage, CRM update instructions, email draft creation, and full workflow overview.  
   - Include author contact details.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow automates personalized proposal creation and CRM/email integration post-sales call.                                     | Overall workflow purpose                             |
| Replace PandaDoc template UUID and API key placeholders with your actual credentials before use.                                 | PandaDoc API setup                                   |
| Requires ClickUp custom fields for company name, proposal URL, and quote; ensure fields exist and IDs are correct.               | ClickUp CRM setup                                   |
| Gmail OAuth2 credentials required to create draft emails.                                                                        | Gmail integration                                   |
| Workflow author: Abdul - builds growth systems for consultants & agencies. Visit https://www.builtbyabdul.com/ or email builtbyabdul@gmail.com | Author and support contact                         |
| Workflow uses GPT-4o model and LangChain integration for advanced AI text generation.                                             | AI integration details                              |
| Form can be replaced with any other tool that supports webhook responses (e.g., Tally).                                          | Customization flexibility                            |
| To customize proposal tone or content, modify the system prompt in the OpenAI node.                                              | AI prompt customization                             |
| Consider adding error handling or retry mechanisms for API nodes to handle network or auth failures.                             | Potential improvements                               |

---

*Disclaimer:* The text provided is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies and containing no illegal or offensive material. All data processed is legal and public.