Website Contact Form to Slack with Optional Email Confirmation

https://n8nworkflows.xyz/workflows/website-contact-form-to-slack-with-optional-email-confirmation-6987


# Website Contact Form to Slack with Optional Email Confirmation

### 1. Workflow Overview

This workflow automates the process of handling website contact form submissions by sending the collected data to a Slack channel and optionally sending a confirmation email to the submitter via either Gmail or Microsoft Outlook. It is designed for businesses or teams who want to centralize lead notifications in Slack while optionally acknowledging contacts through email.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures form submissions from a website contact form.
- **1.2 Notification Dispatch:** Sends the form data as a message to a specified Slack channel.
- **1.3 Optional Email Confirmation:** Sends a thank-you email to the submitter via Gmail or Microsoft Outlook.

An additional **Sticky Note** node provides setup instructions and contact information for support.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives user input data from a website contact form, capturing fields for Name, Email, and Phone via a webhook.

- **Nodes Involved:**  
  - Form Submission on Website

- **Node Details:**

  - **Form Submission on Website**  
    - **Type:** Form Trigger (Webhook Listener)  
    - **Role:** Listens for HTTP POST requests from the website form and extracts submitted data.  
    - **Configuration:**  
      - Form Title: "Contact Form"  
      - Fields captured: Name, Email, Phone  
      - Webhook ID provided for external form integration  
      - No authentication required (public webhook)  
    - **Expressions/Variables:** Captures `$json.Name`, `$json.Email`, `$json.Phone` from incoming requests.  
    - **Input Connections:** None (trigger node)  
    - **Output Connections:** To Slack node, Send Email - Outlook, Send Email - Gmail  
    - **Version:** 2.2  
    - **Edge Cases / Failure Modes:**  
      - Missing required fields if frontend form validation is bypassed  
      - Malformed or unexpected payloads  
      - Unavailable webhook if network issues occur  
    - **Sub-Workflow:** None

#### 2.2 Notification Dispatch

- **Overview:**  
  Posts a notification message to a Slack channel with the submitted contact details for team awareness.

- **Nodes Involved:**  
  - Slack

- **Node Details:**

  - **Slack**  
    - **Type:** Slack node (send message)  
    - **Role:** Sends a formatted message to a specific Slack channel containing the form submission information.  
    - **Configuration:**  
      - Channel: #leads (channel ID: C08T2J84F6C)  
      - Message Text: "You have a form submission with these details. Name: {{ $json.Name }} Email: {{ $json.Email }} Phone: {{ $json.Phone }}"  
      - Credentials: Slack Bot Token via OAuth2  
    - **Expressions/Variables:** Uses the JSON data from the form trigger node to fill in the message.  
    - **Input Connections:** From Form Submission on Website  
    - **Output Connections:** None (terminal node)  
    - **Version:** 2.3  
    - **Edge Cases / Failure Modes:**  
      - Slack API authentication failures (token expired/revoked)  
      - Slack channel not found or insufficient permissions  
      - Rate limiting by Slack API  
    - **Sub-Workflow:** None

#### 2.3 Optional Email Confirmation

- **Overview:**  
  Sends a personalized confirmation email to the submitter via either Gmail or Microsoft Outlook, depending on user preference or availability.

- **Nodes Involved:**  
  - Send Email - Gmail  
  - Send Email - Outlook

- **Node Details:**

  - **Send Email - Gmail**  
    - **Type:** Gmail node (send email)  
    - **Role:** Sends a thank-you email via Gmail API.  
    - **Configuration:**  
      - Recipient: Email from form submission (`$json.Email`)  
      - Subject: "Thank you for reaching out!"  
      - Message body: "Hi {{ $json.Name }}, Thank you for reaching out on our website. We'll be in touch soon!"  
      - Credentials: Gmail OAuth2  
    - **Input Connections:** From Form Submission on Website  
    - **Output Connections:** None  
    - **Version:** 2.1  
    - **Edge Cases / Failure Modes:**  
      - Gmail OAuth2 token expiration or revocation  
      - Recipient email invalid or blocked  
      - Gmail API rate limiting or outages  
    - **Sub-Workflow:** None

  - **Send Email - Outlook**  
    - **Type:** Microsoft Outlook node (send email)  
    - **Role:** Sends a thank-you email via Outlook API.  
    - **Configuration:**  
      - Recipient: Email from form submission (`$json.Email`)  
      - Subject: "Thank you for reaching out!"  
      - Body: "Hi {{ $json.Name }}, Thank you for reaching out on our website. We'll be in touch soon!"  
      - Credentials: Microsoft Outlook OAuth2  
    - **Input Connections:** From Form Submission on Website  
    - **Output Connections:** None  
    - **Version:** 2  
    - **Edge Cases / Failure Modes:**  
      - Outlook OAuth2 token expiration or revocation  
      - Invalid recipient email  
      - Microsoft API throttling or failures  
    - **Sub-Workflow:** None

#### 2.4 Support and Setup Instructions

- **Overview:**  
  Provides detailed instructions for setup, usage, and support contact information as a sticky note within the workflow canvas.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - **Type:** Sticky Note (documentation)  
    - **Role:** Offers a comprehensive step-by-step setup guide and contact details for workflow assistance.  
    - **Content Summary:**  
      - Contact info for Robert Breen (email and LinkedIn)  
      - Instructions on embedding the form using the webhook URL  
      - Explanation of Slack notifications and email confirmations using Gmail or Outlook APIs  
      - Highlights credential requirements for Slack, Gmail, and Outlook  
    - **Input/Output Connections:** None  
    - **Version:** 1  
    - **Edge Cases:** None (non-executable documentation node)  
    - **Sub-Workflow:** None

---

### 3. Summary Table

| Node Name              | Node Type                | Functional Role                | Input Node(s)                | Output Node(s)                         | Sticky Note                                                                                                  |
|------------------------|--------------------------|-------------------------------|-----------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Form Submission on Website | Form Trigger             | Capture website contact form data | None                        | Slack, Send Email - Outlook, Send Email - Gmail | See sticky note for detailed setup instructions and support contact                                           |
| Slack                  | Slack                    | Post notification to Slack channel | Form Submission on Website  | None                                 | See sticky note for detailed setup instructions and support contact                                           |
| Send Email - Outlook   | Microsoft Outlook        | Send confirmation email via Outlook | Form Submission on Website  | None                                 | See sticky note for detailed setup instructions and support contact                                           |
| Send Email - Gmail     | Gmail                    | Send confirmation email via Gmail | Form Submission on Website  | None                                 | See sticky note for detailed setup instructions and support contact                                           |
| Sticky Note            | Sticky Note              | Documentation and setup guide   | None                        | None                                 | Provides detailed setup steps, contact info, and credential requirements                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Submission Trigger Node:**  
   - Add a **Form Trigger** node and name it `Form Submission on Website`.  
   - Configure the form title as `"Contact Form"`.  
   - Add form fields: `Name`, `Email`, and `Phone`.  
   - Save and copy the webhook URL generated by this node.  
   - This webhook URL will be used as the form action URL on your website.

2. **Create the Slack Notification Node:**  
   - Add a **Slack** node named `Slack`.  
   - Set the operation to send a message to a channel.  
   - Select the Slack channel (e.g., #leads) by its channel ID, here `C08T2J84F6C`.  
   - In the message text parameter, enter:  
     `You have a form submission with these details. Name: {{ $json.Name }} Email: {{ $json.Email }} Phone: {{ $json.Phone }}`  
   - Set up Slack credentials with an OAuth2 Slack Bot Token having permission to post to the channel.  
   - Connect the output of `Form Submission on Website` node to this Slack node.

3. **Create Email Confirmation Nodes (Optional):**  
   You can choose to send confirmation emails via Gmail, Outlook, or both.

   - **For Gmail:**  
     - Add a **Gmail** node named `Send Email - Gmail`.  
     - Configure to send email with:  
       - Recipient: `={{ $json.Email }}`  
       - Subject: `"Thank you for reaching out!"`  
       - Message: `"Hi {{ $json.Name }}, Thank you for reaching out on our website. We'll be in touch soon!"`  
     - Set up Gmail OAuth2 credentials with send email scope.  
     - Connect the output of `Form Submission on Website` node to this node.

   - **For Outlook:**  
     - Add a **Microsoft Outlook** node named `Send Email - Outlook`.  
     - Configure to send email with:  
       - To Recipients: `={{ $json.Email }}`  
       - Subject: `"Thank you for reaching out!"`  
       - Body Content: `"Hi {{ $json.Name }}, \n\nThank you for reaching out on our website. We'll be in touch soon!"`  
     - Set up Microsoft Outlook OAuth2 credentials with email send permissions.  
     - Connect the output of `Form Submission on Website` node to this node.

4. **Add a Sticky Note for Documentation:**  
   - Add a **Sticky Note** node on the canvas.  
   - Enter content explaining the workflow purpose, setup steps, and contact info for support (see section 2.4).  
   - This node has no connections.

5. **Test the Workflow:**  
   - Submit test data through the webhook URL or embedded form.  
   - Verify Slack message appears in the channel.  
   - Confirm receipt of confirmation email via Gmail or Outlook.

6. **Deploy and Embed:**  
   - Embed the webhook URL in your website contact form’s action attribute or configure your no-code form builder to post data to this webhook URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                            |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Contact for additional help: **Robert Breen** — Email: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ | Support contact in sticky note                             |
| Step-by-step setup instructions are included in the workflow’s sticky note for easy reference                                    | Sticky Note node content                                   |
| Slack API requires a Bot Token with chat:write permissions to post messages to channels                                           | Slack API documentation                                    |
| Gmail and Microsoft Outlook nodes require OAuth2 credentials with email send scopes                                               | Gmail API and Microsoft Graph API documentation            |
| Webhook URL must be kept private to avoid spam or unauthorized submissions                                                        | Security best practice for webhook usage                    |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.