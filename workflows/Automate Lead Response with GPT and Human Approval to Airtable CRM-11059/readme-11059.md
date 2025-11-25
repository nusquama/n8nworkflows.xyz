Automate Lead Response with GPT and Human Approval to Airtable CRM

https://n8nworkflows.xyz/workflows/automate-lead-response-with-gpt-and-human-approval-to-airtable-crm-11059


# Automate Lead Response with GPT and Human Approval to Airtable CRM

### 1. Workflow Overview

This workflow automates the process of responding to new leads captured via an n8n Form. It incorporates AI-generated personalized email drafts, a human approval step for quality control, and automatic logging of leads and email statuses into Airtable as a CRM system. The workflow is designed to blend automation with human oversight, ensuring fast and consistent follow-ups while maintaining a professional and personalized touch.

**Target Use Cases:**  
- Small sales teams wanting AI assistance with drafting emails but keeping human approval.  
- Businesses receiving inbound inquiries needing quick, standardized responses.  
- Teams wanting seamless integration from lead intake via forms to CRM updates and email outreach without manual intervention.

**Logical Blocks:**

- **1.1 Lead Intake & Preparation:** Capture lead data from an n8n form and normalize it for further processing.  
- **1.2 AI Email Draft Generation:** Use AI (OpenAI GPT-4) to generate a personalized follow-up email draft based on the lead data.  
- **1.3 Human Approval Workflow:** Send the AI draft to the sales team via email with interactive approve/reject links; wait for human action.  
- **1.4 Conditional Email Sending & CRM Logging:** Based on approval, send the email to the lead and log/update lead status in Airtable accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Intake & Preparation

**Overview:**  
This block captures form submissions containing lead data, then extracts and normalizes the key fields into consistent variables for downstream use.

**Nodes Involved:**  
- On form submission  
- Workflow Configuration  
- Extract Lead Data  

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing form submissions with fields: Name, Email, Company, Phone, Message (all required).  
  - Configuration: Form titled "Leads" with described fields and description "We'll reach out to you soon!"  
  - Inputs: None (trigger)  
  - Outputs: Lead form data JSON  
  - Edge Cases: Missing or malformed form data; ensure required fields are filled.  
  - Webhook usage: Yes (webhook for form trigger).

- **Workflow Configuration**  
  - Type: Set  
  - Role: Holds configuration constants and placeholders for company details (name, email, phone, social URLs, Airtable base/table IDs).  
  - Configuration: Set multiple string variables with placeholders for customization.  
  - Inputs: Form data JSON from "On form submission"  
  - Outputs: Combined data with configuration variables for use downstream.  
  - Edge Cases: Placeholder values must be replaced before running; missing config could cause errors.

- **Extract Lead Data**  
  - Type: Set  
  - Role: Extracts and normalizes key lead fields into uniform variable names (leadName, leadEmail, etc.) from the incoming JSON.  
  - Configuration: Expressions extract fields from form submission JSON with fallback defaults (e.g., 'Unknown' for name).  
  - Inputs: Workflow Configuration output  
  - Outputs: Normalized lead data JSON  
  - Edge Cases: Missing fields in form data handled by default values; malformed JSON could break expressions.

---

#### 2.2 AI Email Draft Generation

**Overview:**  
Generates a personalized, professional follow-up email draft using AI, based on the normalized lead data.

**Nodes Involved:**  
- Generate Email Draft  
- OpenAI Chat Model  

**Node Details:**

- **Generate Email Draft**  
  - Type: Langchain Agent (AI agent node)  
  - Role: Prepares prompt and sends it to OpenAI GPT-4 for generating the email body text.  
  - Configuration: Prompt instructs the AI to write a warm, professional follow-up email using lead details (Name, Email, Company, Phone, Message). Returns only the email body text.  
  - Inputs: Extract Lead Data output  
  - Outputs: AI-generated email draft text  
  - Key Expressions: Uses placeholders like `{{ $json.Name }}`, etc., to inject lead data into prompt.  
  - Edge Cases: AI API rate limits or errors; incomplete or incorrect lead data reduces quality; prompt failures.  
  - Version: Requires Langchain integration with OpenAI GPT-4.

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Language model interface node configured to use GPT-4.1-mini model for the AI agent.  
  - Configuration: Model set to "gpt-4.1-mini"; linked to OpenAI credentials.  
  - Inputs: From Langchain agent node as language model provider.  
  - Outputs: Chat completions for draft generation  
  - Edge Cases: Authentication failures, exceeded usage quota, API timeouts.

---

#### 2.3 Human Approval Workflow

**Overview:**  
Sends the AI-generated email draft to the sales team for review and approval via email with embedded approve/reject links, then waits for human input.

**Nodes Involved:**  
- Send Approval Request Email  
- Wait for Approval  
- Check Approval Status  

**Node Details:**

- **Send Approval Request Email**  
  - Type: Gmail node (OAuth2)  
  - Role: Emails the draft to a configured sales team email for review.  
  - Configuration:  
    - To: Placeholder for sales approval email address.  
    - Subject: "New Lead Email Draft - Approval Required"  
    - HTML body: Includes lead details, AI draft, and interactive approve/reject links that trigger the webhook URL to resume the workflow.  
  - Inputs: Generated email draft and lead data  
  - Outputs: None (email sent)  
  - Edge Cases: Gmail OAuth2 credential issues; invalid recipient address; HTML rendering issues.

- **Wait for Approval**  
  - Type: Wait (webhook resume)  
  - Role: Pauses workflow execution waiting for an external webhook call triggered by user's approval or rejection click.  
  - Configuration: Webhook ID linked to approval links; timeout of 24 hours.  
  - Inputs: None (waits for webhook resume)  
  - Outputs: Resumed data containing the approval action query parameter.  
  - Edge Cases: Timeout if no approval/rejection; webhook URL must be production not test URL.

- **Check Approval Status**  
  - Type: If (conditional)  
  - Role: Evaluates the query parameter `action` from webhook resume to decide if the draft was approved or rejected.  
  - Configuration: Checks if `action == "approve"`  
  - Inputs: Output from Wait for Approval webhook  
  - Outputs: Two branches — approved (true) or rejected (false)  
  - Edge Cases: Unexpected or missing query parameters; case sensitivity issues.

---

#### 2.4 Conditional Email Sending & CRM Logging

**Overview:**  
Based on approval, either sends the approved email to the lead and logs the lead with "Email Sent - Approved" status, or logs the lead with a "Email Rejected - Needs Review" status without sending.

**Nodes Involved:**  
- Send Email to Lead  
- Create Lead in Airtable  
- Update Lead Status (Rejected)  

**Node Details:**

- **Send Email to Lead**  
  - Type: Gmail node (OAuth2)  
  - Role: Sends the approved AI-generated email to the lead’s email address.  
  - Configuration:  
    - To: Lead's email from form submission  
    - Subject: "Thank you for your interest!"  
    - HTML body: Styled email with the AI draft content, sender signature, and company branding pulled from configuration node.  
  - Inputs: Approved path from Check Approval Status node  
  - Outputs: Leads into Airtable creation node  
  - Edge Cases: Credential failures, invalid lead email, HTML rendering issues.

- **Create Lead in Airtable**  
  - Type: Airtable node (OAuth2)  
  - Role: Creates a new record in Airtable lead CRM with full lead details, email draft, and status "Email Sent - Approved".  
  - Configuration:  
    - Base and Table IDs from configuration  
    - Maps lead fields: Email, Notes (message), Phone, Status, First Name, Email Draft, Company Name  
  - Inputs: Output from Send Email to Lead  
  - Outputs: None  
  - Edge Cases: Authentication errors, invalid base/table IDs, field mapping mismatches.

- **Update Lead Status (Rejected)**  
  - Type: Airtable node (OAuth2)  
  - Role: Creates a record in Airtable with lead data but status "Email Rejected - Needs Review" if email draft rejected.  
  - Configuration: Same base/table and fields as Create Lead node, with adjusted status value.  
  - Inputs: Rejected path from Check Approval Status node  
  - Outputs: None  
  - Edge Cases: Same as above; ensures rejected leads are tracked for review.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                           | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                               |
|---------------------------|--------------------------------|-----------------------------------------|---------------------------|----------------------------|----------------------------------------------------------------------------------------------------------|
| On form submission        | Form Trigger                   | Entry: Captures lead form submissions   | None                      | Workflow Configuration      |                                                                                                          |
| Workflow Configuration    | Set                           | Stores company & Airtable config values | On form submission         | Extract Lead Data           |                                                                                                          |
| Extract Lead Data         | Set                           | Normalizes lead data fields              | Workflow Configuration     | Generate Email Draft        | 📝 Lead Intake & Preparation: normalizes form fields for AI accuracy                                     |
| Generate Email Draft      | Langchain Agent                | Generates AI email draft using GPT-4    | Extract Lead Data, OpenAI Chat Model | Send Approval Request Email | 🤖 Draft Generation & Human Approval: AI writes draft, sends for human review                            |
| OpenAI Chat Model         | Langchain LM Chat OpenAI       | Provides GPT-4 model for AI prompt       | N/A                       | Generate Email Draft        |                                                                                                          |
| Send Approval Request Email | Gmail OAuth2                  | Sends draft to sales for approval       | Generate Email Draft       | Wait for Approval           |                                                                                                          |
| Wait for Approval         | Wait (Webhook Resume)          | Pauses workflow until approval response | Send Approval Request Email | Check Approval Status       |                                                                                                          |
| Check Approval Status     | If                            | Checks if draft approved or rejected    | Wait for Approval          | Send Email to Lead, Update Lead Status (Rejected) |                                                                                                          |
| Send Email to Lead        | Gmail OAuth2                   | Sends approved email to lead             | Check Approval Status (approve) | Create Lead in Airtable    | 📊 Log Lead in Airtable & Send Final Email: email to lead, logs approved leads                           |
| Create Lead in Airtable   | Airtable OAuth2                | Logs approved lead with status           | Send Email to Lead         | None                       |                                                                                                          |
| Update Lead Status (Rejected) | Airtable OAuth2             | Logs rejected lead with status           | Check Approval Status (reject) | None                       |                                                                                                          |
| Sticky Note               | Sticky Note                   | Explanation and workflow purpose        | None                      | None                       | 📌 Purpose: Full explanation of workflow benefits and use cases                                         |
| Sticky Note1              | Sticky Note                   | Setup instructions                       | None                      | None                       | 🔧 Setup Instructions: credential and config setup guidance                                             |
| Sticky Note2              | Sticky Note                   | Troubleshooting tips                     | None                      | None                       | 🛠️ Troubleshooting: common issues and fixes                                                           |
| Sticky Note3              | Sticky Note                   | Lead intake explanation                  | None                      | None                       | 📝 Lead Intake & Preparation block explanation                                                         |
| Sticky Note4              | Sticky Note                   | Draft generation & approval explanation | None                      | None                       | 🤖 Draft Generation & Human Approval block explanation                                                 |
| Sticky Note5              | Sticky Note                   | Logging & email sending explanation      | None                      | None                       | 📊 Log Lead in Airtable & Send Final Email block explanation                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form titled "Leads" with these fields (all required):  
     - Name (text)  
     - Email (email)  
     - Company (text)  
     - Phone (text, placeholder "(222)222-2222")  
     - Message (text)  
   - Publish the form to get the webhook URL.

2. **Add "Workflow Configuration" Node (Set)**  
   - Add string variables to store configuration:  
     - airtableBaseId, airtableTableId (Airtable IDs)  
     - companyName, senderName, senderTitle  
     - companyEmail, companyPhone, companyAddress  
     - linkedinUrl, twitterUrl, facebookUrl, websiteUrl  
   - Use this node output to provide global config for the workflow.

3. **Add "Extract Lead Data" Node (Set)**  
   - Use expressions to extract and normalize form data fields:  
     - leadName = `={{ $json.body.name || 'Unknown' }}`  
     - leadEmail = `={{ $json.body.email || '' }}`  
     - leadCompany = `={{ $json.body.company || '' }}`  
     - leadPhone = `={{ $json.body.phone || '' }}`  
     - leadMessage = `={{ $json.body.message || '' }}`

4. **Add "OpenAI Chat Model" Node (Langchain LM Chat OpenAI)**  
   - Configure credentials with OpenAI API (requires GPT-4 access).  
   - Select model "gpt-4.1-mini" or appropriate GPT-4 model.

5. **Add "Generate Email Draft" Node (Langchain Agent)**  
   - Configure prompt to instruct AI to generate a warm, professional email using lead details (referencing normalized variables).  
   - Connect this node to use the "OpenAI Chat Model" as the language model provider.  
   - Output only the email body text.

6. **Add "Send Approval Request Email" Node (Gmail OAuth2)**  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient email to your sales approval address.  
   - Compose an HTML email that includes:  
     - Lead details from form submission  
     - AI-generated draft  
     - Approve and Reject buttons linking to the webhook URL that resumes the workflow with action query (`?action=approve` or `?action=reject`).

7. **Add "Wait for Approval" Node (Wait)**  
   - Configure to wait for webhook resume with the same webhook ID used in the approval email links.  
   - Set timeout to 24 hours or desired limit.

8. **Add "Check Approval Status" Node (If)**  
   - Condition: `={{ $('Wait for Approval').item.json.query.action }} == "approve"`  
   - True branch: approved path  
   - False branch: rejected path

9. **Add "Send Email to Lead" Node (Gmail OAuth2)**  
   - Configure Gmail credentials.  
   - Email to lead's email from form submission.  
   - Subject: "Thank you for your interest!"  
   - HTML body: include AI draft, personalized signature using company config variables.

10. **Add "Create Lead in Airtable" Node (Airtable OAuth2)**  
    - Configure Airtable OAuth2 credentials.  
    - Select base and table matching your CRM.  
    - Map fields:  
      - Email, Phone, Company Name, First Name, Notes (Message), Email Draft, Status ("Email Sent - Approved").  
    - Connect output from "Send Email to Lead".

11. **Add "Update Lead Status (Rejected)" Node (Airtable OAuth2)**  
    - Same Airtable base/table as above.  
    - Map same fields but set Status to "Email Rejected - Needs Review".  
    - Connect output from "Check Approval Status" false branch.

12. **Connect the nodes in order:**  
    - On form submission → Workflow Configuration → Extract Lead Data → Generate Email Draft → Send Approval Request Email → Wait for Approval → Check Approval Status  
    - Approved branch → Send Email to Lead → Create Lead in Airtable  
    - Rejected branch → Update Lead Status (Rejected)

13. **Test the workflow:**  
    - Submit test form data.  
    - Verify AI email draft generation.  
    - Confirm approval email receipt with working approve/reject links.  
    - Approve or reject and check that email is sent or lead status updated accordingly in Airtable.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow enables a polished lead-response system with AI drafting, human approval, CRM logging, and automated Gmail sending, ideal for teams without a full CRM system.                                                          | Sticky Note 📌 Purpose                                               |
| Setup requires connecting Gmail, Airtable, OpenAI credentials and configuring your Airtable base and table with specified fields. Replace placeholder values with actual company and credential details before use.                     | Sticky Note 🔧 Setup Instructions                                    |
| Common issues include approval link malfunctions (ensure production webhook URLs), email sending failures (check Gmail OAuth2), Airtable syncing problems (verify IDs and field names), and stuck waiting (ensure webhook triggers). | Sticky Note 🛠️ Troubleshooting                                      |
| The AI prompt uses direct lead data injection for personalized, relevant email drafting; modifying the prompt allows customization of email tone and content.                                                                        | Insights from Generate Email Draft prompt                            |
| The approval email includes HTML buttons linking to webhook resume URLs with query parameters controlling workflow path, requiring proper webhook deployment and URL exposure.                                                        | Send Approval Request Email node details                            |

---

**Disclaimer:**  
This documentation is based solely on an automated n8n workflow export. All processing complies with current content policies and handles only legal, public data. No illegal or offensive content is present.