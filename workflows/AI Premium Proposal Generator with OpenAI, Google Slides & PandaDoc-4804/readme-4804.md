AI Premium Proposal Generator with OpenAI, Google Slides & PandaDoc

https://n8nworkflows.xyz/workflows/ai-premium-proposal-generator-with-openai--google-slides---pandadoc-4804


# AI Premium Proposal Generator with OpenAI, Google Slides & PandaDoc

### 1. Workflow Overview

This workflow, titled **AI Premium Proposal Generator with OpenAI, Google Slides & PandaDoc**, is designed to automate the creation of professional business proposals during or immediately after live sales calls. It captures essential deal data from a form, leverages AI to generate tailored proposal content, and produces polished documents either via a free Google Slides template or a premium PandaDoc integration with payment options. The workflow is structured into four main logical blocks:

- **1.1 Input Reception:** Captures sales call data through a web form trigger.
- **1.2 AI Content Generation:** Uses OpenAI’s GPT-4 model to create detailed proposal content in JSON format based on the form inputs.
- **1.3 Document Generation:** Two parallel paths generate proposals:
  - **Google Slides Path:** Copies a slides template, replaces placeholders with AI-generated text, then emails the proposal link.
  - **PandaDoc Path:** Creates a PandaDoc document with embedded payment/pricing info and sends it via email.
- **1.4 Notification & Delivery:** Sends the generated proposal links/documents to clients through Gmail.

This segmentation supports a “freemium” model allowing a free option (Google Slides) and a paid option (PandaDoc) with invoice/payment integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Captures detailed sales call information via a web form designed for quick completion during calls. Fields include company info, problem, solution, scope, cost, and timeline expectations.

**Nodes Involved:**  
- On form submission  
- On form submission1  

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing sales call data for the Google Slides path.  
  - *Configuration:* Form titled "Sales Call Logging Form" with required fields: First Name, Last Name, Company Name, Email, Website, Problem, Solution, Scope, Cost, How soon?  
  - *Inputs:* HTTP webhook POST from form submission  
  - *Outputs:* Triggers downstream AI content generation for Google Slides  
  - *Potential Failures:* Missing required fields; malformed inputs; network/webhook failures.  

- **On form submission1**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing sales call data for the PandaDoc path.  
  - *Configuration:* Identical form and fields as above; separate webhook ID to isolate flows.  
  - *Inputs:* HTTP webhook POST from form submission  
  - *Outputs:* Triggers AI content generation for PandaDoc path  
  - *Potential Failures:* Same as above  

---

#### 2.2 AI Content Generation

**Overview:**  
Processes form inputs with OpenAI GPT-4 to generate comprehensive proposal content structured as JSON. Output includes proposal titles, problem summaries, solutions, scopes, milestones, and timelines, following a strict template and tone rules.

**Nodes Involved:**  
- OpenAI  
- OpenAI1  

**Node Details:**

- **OpenAI**  
  - *Type:* OpenAI GPT-4 (LangChain integration)  
  - *Role:* Generates proposal JSON for the Google Slides path.  
  - *Configuration:*  
    - Model: GPT-4o  
    - System prompt sets tone as helpful and professional writing assistant.  
    - Instructions specify spartan, casual, professional tone; all fields must be filled; concise descriptions.  
    - Input JSON assembled from form data (companyName, problem, solution, scope, currentDate, howSoon, depositCost).  
  - *Expressions:* Uses template expressions to map form fields to prompt variables.  
  - *Inputs:* From "On form submission" node (form data)  
  - *Outputs:* JSON object with proposal content fields for Google Slides replacement.  
  - *Potential Failures:* API errors; rate limits; malformed JSON output; incomplete fields if AI fails; credential issues.  

- **OpenAI1**  
  - *Type:* OpenAI GPT-4 (LangChain)  
  - *Role:* Generates proposal JSON for the PandaDoc path with slightly different field structure and numbering in text fields.  
  - *Configuration:*  
    - Model: GPT-4o  
    - System prompt similar to OpenAI node but with additional instructions for numbering problem summaries and solutions.  
    - Returns JSON with fields suited for PandaDoc tokens and pricing integration.  
  - *Expressions:* Same templating for form data inputs.  
  - *Inputs:* From "On form submission1" node (form data)  
  - *Outputs:* JSON object for PandaDoc document creation  
  - *Potential Failures:* Same as OpenAI node above  

---

#### 2.3 Document Generation

**Overview:**  
Generates the final proposal documents using either Google Slides or PandaDoc, populating templates with AI-generated content.

**Nodes Involved:**  
- Google Drive  
- Replace Text  
- HTTP Request  

**Node Details:**

- **Google Drive**  
  - *Type:* Google Drive  
  - *Role:* Copies a Google Slides template file to a new file named after the proposal title.  
  - *Configuration:*  
    - Operation: copy  
    - fileId: ID of the Slides template  
    - Copy name: dynamic, from AI-generated proposalTitle  
    - Copy does not require writer permission.  
  - *Inputs:* From OpenAI node (Google Slides path)  
  - *Outputs:* Passes new file ID (presentationId) to "Replace Text" node  
  - *Potential Failures:* Permission denied; invalid template ID; quota exceeded; network errors  
  - *Version:* v3  

- **Replace Text**  
  - *Type:* Google Slides  
  - *Role:* Replaces text placeholders in the copied Slides file with AI-generated proposal content.  
  - *Configuration:*  
    - Operation: replaceText  
    - presentationId: from Google Drive copy  
    - Multiple text replacements mapped from OpenAI JSON fields (e.g., {{proposalTitle}} replaced with AI content)  
    - Includes static cost field "$1,850"  
  - *Inputs:* From Google Drive node  
  - *Outputs:* Passes presentationId and updated slides info to Gmail node  
  - *Potential Failures:* Invalid presentation ID; exceeded API quotas; text replacement errors; missing fields in AI output  
  - *Version:* v2  

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Creates a PandaDoc document via API with embedded client and proposal data, including payment/pricing info.  
  - *Configuration:*  
    - POST to PandaDoc documents endpoint  
    - JSON body includes: proposal title, tokens with client and sender info (emails, names), multiple scopes, timelines, problem and solution texts, pricing table with cost extracted and converted to number.  
    - Template UUID specified for PandaDoc template  
    - Headers include API key authorization and accept JSON; API key placeholder must be replaced.  
  - *Inputs:* From OpenAI1 node (PandaDoc path)  
  - *Outputs:* Document creation response passed to Gmail1 for emailing  
  - *Potential Failures:* Authorization errors if API key invalid; template UUID errors; malformed JSON; network issues; rate limiting  
  - *Version:* v4.2  

---

#### 2.4 Notification & Delivery

**Overview:**  
Sends emails to clients with links to the generated proposals, using Gmail OAuth2 credentials.

**Nodes Involved:**  
- Gmail  
- Gmail1  

**Node Details:**

- **Gmail**  
  - *Type:* Gmail  
  - *Role:* Sends email with Google Slides proposal link to client.  
  - *Configuration:*  
    - To: email from form submission  
    - Subject: "Re: Proposal for LeftClick" (hardcoded, could be dynamic)  
    - Message body includes personalized text with link to Google Slides presentation (editable URL includes presentationId).  
    - Email type: plain text  
    - Attribution disabled to avoid appended signatures from n8n  
  - *Inputs:* From Replace Text node  
  - *Outputs:* None (end of Google Slides path)  
  - *Potential Failures:* Invalid email; Gmail OAuth2 token expiration; quota limits; network errors  
  - *Version:* v2.1  

- **Gmail1**  
  - *Type:* Gmail  
  - *Role:* Sends email with PandaDoc proposal link to client.  
  - *Configuration:*  
    - To: email from form submission1  
    - Subject and message similar to Gmail node, includes PandaDoc document link  
    - Email type: plain text  
    - Attribution disabled  
  - *Inputs:* From HTTP Request node (PandaDoc creation)  
  - *Outputs:* None (end of PandaDoc path)  
  - *Potential Failures:* Same as Gmail node  

---

### 3. Summary Table

| Node Name          | Node Type                  | Functional Role                           | Input Node(s)      | Output Node(s)     | Sticky Note                                                                                                                      |
|--------------------|----------------------------|-----------------------------------------|--------------------|--------------------|---------------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger               | Capture sales call data (Google Slides) | (Webhook)          | OpenAI             | 🚀 STEP 1: Sales Call Form - Capture deal data for AI processing                                                                 |
| OpenAI             | OpenAI GPT-4 LangChain    | Generate proposal JSON for Slides       | On form submission | Google Drive       | 🧠 STEP 2: AI Content Generation - Converts input to 20+ professional proposal sections                                          |
| Google Drive       | Google Drive               | Copy Slides template for proposal       | OpenAI             | Replace Text       | 📄 STEP 3: Document Generation - Google Slides path                                                                               |
| Replace Text       | Google Slides              | Replace placeholders in Slides template | Google Drive       | Gmail              | 📄 STEP 3: Document Generation - Google Slides path                                                                               |
| Gmail              | Gmail                      | Send proposal email with Slides link    | Replace Text       | -                  | 📄 STEP 3: Document Generation - Google Slides path                                                                               |
| On form submission1 | Form Trigger               | Capture sales call data (PandaDoc)      | (Webhook)          | OpenAI1            | 🚀 STEP 1: Sales Call Form - Capture deal data for AI processing                                                                 |
| OpenAI1            | OpenAI GPT-4 LangChain    | Generate proposal JSON for PandaDoc     | On form submission1 | HTTP Request       | 🧠 STEP 2: AI Content Generation - Converts input to 20+ professional proposal sections                                          |
| HTTP Request       | HTTP Request (PandaDoc API) | Create PandaDoc proposal with pricing   | OpenAI1            | Gmail1             | 📄 STEP 3: Document Generation - PandaDoc path with payment integration                                                          |
| Gmail1             | Gmail                      | Send proposal email with PandaDoc link  | HTTP Request       | -                  | 📄 STEP 3: Document Generation - PandaDoc path                                                                                   |
| sticky-note-1      | Sticky Note                | Informational - Sales Call Form overview| -                  | -                  | 🚀 STEP 1: Sales Call Form - Captures essential deal information in under 30 seconds                                              |
| sticky-note-2      | Sticky Note                | Informational - AI Content Generation   | -                  | -                  | 🧠 STEP 2: AI Content Generation - Transforms inputs into professional proposal content                                           |
| sticky-note-3      | Sticky Note                | Informational - Document Generation     | -                  | -                  | 📄 STEP 3: Document Generation - Google Slides and PandaDoc paths                                                                 |
| sticky-note-4      | Sticky Note                | Informational - Workflow Advantages     | -                  | -                  | ⚡ WORKFLOW ADVANTAGES - Speed, Quality, Dual Options, Revenue Impact                                                             |
| Sticky Note        | Sticky Note                | Informational - Google Slides flow intro| -                  | -                  | ## Google Slides AI Proposal Generator - This flow generates proposals using the free Google Slides solution                     |
| Sticky Note1       | Sticky Note                | Informational - PandaDoc flow intro     | -                  | -                  | ## PandaDoc AI Proposal Generator - This flow generates proposals using the paid PandaDoc solution, and sends invoice alongside  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node (On form submission) for Google Slides Path:**  
   - Type: Form Trigger  
   - Form Title: "Sales Call Logging Form"  
   - Add fields: First Name (required), Last Name (required), Company Name (required), Email (email), Website (required), Problem (textarea, required), Solution (textarea, required), Scope (textarea, required), Cost (required), How soon? (required)  
   - Save webhook URL for form integration.

2. **Create OpenAI Node (OpenAI) for Google Slides:**  
   - Type: OpenAI GPT-4 (LangChain)  
   - Model: GPT-4o  
   - System prompt: "You are a helpful, intelligent writing assistant."  
   - Messages: Include instructions to generate JSON proposal with all required fields, spartan tone, concise descriptions, use form data as input variables mapped via expressions.  
   - JSON output enabled.

3. **Create Google Drive Node:**  
   - Type: Google Drive  
   - Operation: Copy  
   - File ID: Google Slides proposal template ID  
   - Name: Use expression to name copied file with proposalTitle from OpenAI output  
   - Set copyRequiresWriterPermission to false  
   - Connect OpenAI → Google Drive

4. **Create Google Slides Replace Text Node:**  
   - Type: Google Slides  
   - Operation: replaceText  
   - Presentation ID: from Google Drive copy output  
   - Add replacement pairs for all placeholders (e.g., {{proposalTitle}} replaced by OpenAI JSON content proposalTitle)  
   - Set static cost value ("$1,850") in corresponding placeholder  
   - Connect Google Drive → Replace Text

5. **Create Gmail Node:**  
   - Type: Gmail  
   - Credentials: OAuth2 for Gmail account  
   - To: Email from form submission  
   - Subject: "Re: Proposal for LeftClick" (adjust as needed)  
   - Message: Include personalized message with link to Google Slides presentation (using presentationId)  
   - Disable attribution  
   - Connect Replace Text → Gmail

6. **Create Form Trigger Node (On form submission1) for PandaDoc Path:**  
   - Duplicate form trigger with same fields and settings for PandaDoc flow  
   - Save webhook URL separately.

7. **Create OpenAI Node (OpenAI1) for PandaDoc:**  
   - Similar setup as OpenAI, but with prompt adjusted for PandaDoc JSON format  
   - Include numbered problem summaries and solutions in text fields  
   - Connect On form submission1 → OpenAI1

8. **Create HTTP Request Node for PandaDoc Document Creation:**  
   - Method: POST  
   - URL: https://api.pandadoc.com/public/v1/documents/  
   - Headers: Authorization "API-Key {yourApiKeyGoesHere}", accept application/json  
   - Body: JSON template with proposal data tokens, recipients, pricing tables, and PandaDoc template UUID  
   - Use expressions to fill tokens from OpenAI1 output and form data  
   - Connect OpenAI1 → HTTP Request

9. **Create Gmail Node (Gmail1) for PandaDoc Email:**  
   - Credentials: same Gmail OAuth2 account  
   - To: Email from form submission1  
   - Subject and message similar to Google Slides Gmail node but adjusted for PandaDoc link  
   - Disable attribution  
   - Connect HTTP Request → Gmail1

10. **Add Sticky Notes:**  
    - Add descriptive sticky notes matching content from original workflow for clarity and documentation.

11. **Set Credentials:**  
    - Configure OpenAI API key credentials for both OpenAI nodes  
    - Configure Google Drive OAuth2 credentials for drive and slides nodes  
    - Configure Gmail OAuth2 credentials for Gmail nodes  
    - Configure PandaDoc API key in HTTP Request node header (replace placeholder)

12. **Test Workflow:**  
    - Submit test data through both form webhooks  
    - Check AI-generated JSON outputs for completeness  
    - Verify Google Slides copied and populated correctly  
    - Verify PandaDoc document creation and pricing table correctness  
    - Confirm emails sent with correct links and content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow enables lightning-fast proposal generation (under 30 seconds) during live sales calls.                                                  | Sticky Note 4 - Workflow Advantages                                                             |
| Dual proposal generation options: free Google Slides or premium PandaDoc with payment integration.                                              | Sticky Note 3 - Document Generation explanation                                                 |
| PandaDoc requires an API key and a pre-created template UUID to function properly.                                                              | HTTP Request node in PandaDoc path                                                              |
| AI prompts enforce concise, professional, and fully filled JSON outputs to ensure seamless template population.                                | OpenAI and OpenAI1 node system messages                                                         |
| Gmail OAuth2 credentials must have Send Email permission and be configured properly to avoid delivery issues.                                   | Gmail and Gmail1 nodes                                                                           |
| Google Drive OAuth2 credentials must have access to the Slides template file and permission to copy and edit.                                   | Google Drive and Google Slides Replace Text nodes                                               |
| Form Trigger nodes require embedding the webhook URL in the sales call logging form used by sales reps.                                         | On form submission and On form submission1 nodes                                                |
| Example pricing range per implementation: $1,500-$5,000, depending on client scope and pricing selected.                                        | Sticky Note 4 - Workflow Advantages                                                             |
| The workflow is tagged as part of the “N8N Course” and can be used as a reference for integrating AI with document generation and emailing.     | Workflow metadata                                                                               |

---

**Disclaimer:** The provided text is extracted solely from an automated n8n workflow. All data is legal, public, and compliant with content policies. No illegal or offensive material is included.