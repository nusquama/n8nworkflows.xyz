Send Personalized HTML Welcome Emails to New Xero Contacts via SMTP

https://n8nworkflows.xyz/workflows/send-personalized-html-welcome-emails-to-new-xero-contacts-via-smtp-7165


# Send Personalized HTML Welcome Emails to New Xero Contacts via SMTP

### 1. Workflow Overview

This workflow automates sending personalized HTML welcome emails via SMTP to new contacts created in Xero accounting software. It listens for webhook events from Xero when a new contact is added, verifies if the contact is genuinely new, fetches the full contact details, builds a customized HTML email tailored to that contact, and then sends the email using an SMTP email node.

Logical blocks include:

- **1.1 Input Reception:** Receiving webhook events from Xero for new contacts.
- **1.2 New Contact Verification:** Determining if the webhook event corresponds to a new contact creation.
- **1.3 Data Enrichment:** Fetching full contact details from Xero for personalization.
- **1.4 Email Construction:** Building the personalized HTML email content using a code node.
- **1.5 Email Dispatch:** Sending the constructed email via SMTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives webhook POST requests triggered by new contact events in Xero.

- **Nodes Involved:**  
  - New Contact in Xero

- **Node Details:**  
  - **New Contact in Xero**  
    - Type: Webhook  
    - Role: Entry point for the workflow, listens for incoming HTTP requests from Xero when a contact is created.  
    - Configuration: Uses a unique webhook ID; no additional parameters configured.  
    - Expressions/Variables: Captures incoming webhook payload (contact info).  
    - Input: External HTTP POST from Xero.  
    - Output: Passes webhook data to the next node "Is it a NEW Contact?"  
    - Version: 2  
    - Potential Failures: Invalid or missing webhook payload; misconfiguration of webhook URL in Xero; network issues.  

---

#### 2.2 New Contact Verification

- **Overview:**  
Determines whether the incoming webhook event corresponds to a genuinely new contact or an update to an existing one, thus avoiding duplicate welcome emails.

- **Nodes Involved:**  
  - Is it a NEW Contact?

- **Node Details:**  
  - **Is it a NEW Contact?**  
    - Type: If (conditional logic)  
    - Role: Checks condition(s) to decide workflow path.  
    - Configuration: Conditions likely check webhook payload fields to confirm new contact status (e.g., creation timestamp or a flag). Exact logic not shown in JSON but implied.  
    - Expressions: Uses data from "New Contact in Xero" node to evaluate conditions.  
    - Input: Receives webhook payload from "New Contact in Xero".  
    - Output:  
       - True branch: Leads to "Fetch Full Contact Details from Xero" (for new contacts only).  
       - False branch: Ends workflow or ignores non-new contacts.  
    - Version: 2.2  
    - Edge Cases: False positives/negatives if webhook data is incomplete; missing fields; if Xero sends update events.  

---

#### 2.3 Data Enrichment

- **Overview:**  
Retrieves detailed contact information from Xero using the contact ID from the webhook to enrich the email content.

- **Nodes Involved:**  
  - Fetch Full Contact Details from Xero

- **Node Details:**  
  - **Fetch Full Contact Details from Xero**  
    - Type: Xero node  
    - Role: Queries Xero API to obtain full contact details by contact ID.  
    - Configuration: Uses credentials to authenticate with Xero API; configured to fetch contact details by ID obtained from webhook.  
    - Expressions: Uses contact ID from previous node's data.  
    - Input: Triggered only on True branch from "Is it a NEW Contact?".  
    - Output: Passes detailed contact info to "Build Personalized HTML Email".  
    - Version: 1  
    - Potential Failures: API auth errors; rate limits; missing contact; network timeouts.  

---

#### 2.4 Email Construction

- **Overview:**  
Constructs a personalized HTML email body tailored to the new contact's details.

- **Nodes Involved:**  
  - Build Personalized HTML Email

- **Node Details:**  
  - **Build Personalized HTML Email**  
    - Type: Code (JavaScript)  
    - Role: Dynamically generates HTML email content using contact data.  
    - Configuration: Contains custom JS code to format contact details into an HTML template.  
    - Expressions/Variables: Uses data from "Fetch Full Contact Details from Xero" (e.g., contact name, company).  
    - Input: Receives enriched contact details.  
    - Output: Passes constructed email content (HTML) to "Send Personalized Welcome Email".  
    - Version: 2  
    - Edge Cases: Code errors; missing contact fields; improper HTML escaping; undefined variables.  

---

#### 2.5 Email Dispatch

- **Overview:**  
Sends the personalized HTML welcome email via SMTP to the new contact.

- **Nodes Involved:**  
  - Send Personalized Welcome Email

- **Node Details:**  
  - **Send Personalized Welcome Email**  
    - Type: EmailSend (SMTP)  
    - Role: Sends an email via SMTP server using the constructed HTML content.  
    - Configuration: SMTP credentials set up (not shown in JSON); email recipient, subject, and HTML body dynamically set from previous node output.  
    - Expressions: Likely uses email address from contact details and HTML from the code node.  
    - Input: Receives HTML email content and recipient info from "Build Personalized HTML Email".  
    - Output: Workflow ends after email sent.  
    - Version: 2.1  
    - Potential Failures: SMTP auth errors; invalid email addresses; email sending limits; network issues.  

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                         | Input Node(s)             | Output Node(s)                       | Sticky Note |
|-------------------------------|--------------------|---------------------------------------|---------------------------|------------------------------------|-------------|
| New Contact in Xero            | Webhook            | Entry point: receive webhook from Xero| (external)                | Is it a NEW Contact?                |             |
| Is it a NEW Contact?           | If                 | Check if contact is newly created     | New Contact in Xero       | Fetch Full Contact Details from Xero (true branch) |             |
| Fetch Full Contact Details from Xero | Xero API node   | Retrieve full contact details          | Is it a NEW Contact?      | Build Personalized HTML Email      |             |
| Build Personalized HTML Email  | Code (JavaScript)  | Build personalized HTML email content | Fetch Full Contact Details from Xero | Send Personalized Welcome Email |             |
| Send Personalized Welcome Email| EmailSend (SMTP)   | Send the welcome email via SMTP        | Build Personalized HTML Email | (end)                            |             |
| Sticky Note                   | Sticky Note        | (No content)                          |                           |                                    |             |
| Sticky Note1                  | Sticky Note        | (No content)                          |                           |                                    |             |
| Sticky Note2                  | Sticky Note        | (No content)                          |                           |                                    |             |
| Sticky Note3                  | Sticky Note        | (No content)                          |                           |                                    |             |
| Sticky Note4                  | Sticky Note        | (No content)                          |                           |                                    |             |
| Sticky Note5                  | Sticky Note        | (No content)                          |                           |                                    |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "New Contact in Xero"**  
   - Type: Webhook  
   - Purpose: Receive HTTP POST from Xero when a new contact is created.  
   - Setup: Copy webhook URL generated by n8n and configure it in Xero to send contact creation events.  
   - Parameters: Default webhook settings; no extra config needed.  

2. **Create If Node: "Is it a NEW Contact?"**  
   - Type: If  
   - Purpose: Check if the incoming webhook payload corresponds to a new contact.  
   - Set condition(s) based on webhook data, e.g.:  
     - Field indicating creation event or timestamp presence.  
   - Connect "New Contact in Xero" node output to this node input.  

3. **Create Xero Node: "Fetch Full Contact Details from Xero"**  
   - Type: Xero  
   - Purpose: Retrieve full contact details using contact ID from webhook.  
   - Credentials: Configure Xero OAuth2 credentials in n8n.  
   - Parameters: Set operation to fetch contact by ID; map contact ID from "Is it a NEW Contact?" output.  
   - Connect True branch of "Is it a NEW Contact?" to this node.  

4. **Create Code Node: "Build Personalized HTML Email"**  
   - Type: Code (JavaScript)  
   - Purpose: Generate personalized HTML email body using contact details.  
   - Code Sample (conceptual):  
     ```javascript
     const contact = items[0].json;
     return [{
       json: {
         html: `<html><body><h1>Welcome, ${contact.Name}!</h1><p>Thank you for joining us.</p></body></html>`,
         email: contact.EmailAddress
       }
     }];
     ```  
   - Connect output of "Fetch Full Contact Details from Xero" to this node.  

5. **Create EmailSend Node: "Send Personalized Welcome Email"**  
   - Type: EmailSend (SMTP)  
   - Purpose: Send the email via SMTP.  
   - Credentials: Configure SMTP credentials (e.g., Gmail, Outlook, or other SMTP server).  
   - Parameters:  
     - Recipient Email: Use expression to get from code node output (e.g., `{{$json["email"]}}`).  
     - Subject: e.g. "Welcome to [Your Company Name]"  
     - HTML Body: Use expression to get from code node output (e.g., `{{$json["html"]}}`).  
   - Connect output of "Build Personalized HTML Email" to this node.  

6. **Workflow Activation**  
   - Activate the workflow to listen for incoming webhooks and process them in real-time.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                   |
|------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow requires proper configuration of the Xero webhook to send new contact creation events to the n8n webhook URL. | Xero API and webhook configuration documentation |
| SMTP credentials must be securely stored in n8n credentials manager and tested before use.     | n8n SMTP node documentation                       |
| Code node HTML generation must sanitize inputs to avoid injection and formatting errors.       | Best practices for email HTML content             |
| Handle Xero API rate limits and webhook retries to ensure robust processing in production.     | Xero API rate limits docs                          |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.